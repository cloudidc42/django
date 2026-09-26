# Part 059: Django Testing เบื้องต้น: unittest

> **ขั้นตอนที่ 581-590 ของหลักสูตร** | Phase 7: Testing & Quality Assurance (Part แรกของ Phase นี้)
>
> ตลอด 58 Part ที่ผ่านมา คุณสร้างแอปบล็อกที่มี Model, View, Form, API, Frontend
> integration และระบบอัปโหลดไฟล์ครบถ้วน แต่มีสิ่งหนึ่งที่หลักสูตรนี้จงใจ**ยังไม่แตะเลย**
> จนถึงตอนนี้ นั่นคือ **Automated Testing** — ทุกครั้งที่คุณตรวจสอบว่าโค้ดทำงานถูกต้อง
> คุณต้องเปิด `python manage.py shell` มาลองเรียกฟังก์ชันเอง หรือเปิดเบราว์เซอร์เข้า
> เว็บไซต์แล้วคลิกทดสอบด้วยมือ (**manual testing**) วิธีนี้ใช้ได้ตอนโปรเจกต์เล็ก แต่เมื่อ
> `Post` model จาก Part 011 มี field และ method เพิ่มขึ้นเรื่อย ๆ (`increment_view_count()`,
> `is_free`, `publish()`, การสร้าง `slug`/`excerpt` อัตโนมัติใน `save()`) การตรวจสอบด้วยมือ
> ทุกครั้งที่แก้โค้ดจะกลายเป็นเรื่องที่ทำไม่ทันและพลาดง่ายมาก **Phase 7 ทั้ง Phase**
> (Part 059-065) จะสอนให้คุณเขียน test ที่รันอัตโนมัติแทนมือคุณ เริ่มจาก Part นี้ที่จะพา
> ไปรู้จัก `unittest` — รากฐานที่ Django ใช้สร้างระบบ testing ทั้งหมด, `TestCase` ของ
> Django, การรัน test, Test Database ที่แยกจากฐานข้อมูลจริงเสมอ และปิดท้ายด้วยการเขียน
> test suite แรกของคุณสำหรับ `Post` model จริง ๆ

---

## สารบัญของ Part นี้

- ขั้นตอนที่ 581: `TestCase` class เบื้องต้น — `setUp()`/`tearDown()`
- ขั้นตอนที่ 582: Assert methods ที่ใช้บ่อย — `assertEqual`, `assertTrue`, `assertFalse`, `assertRaises`, `assertIsNone`
- ขั้นตอนที่ 583: Django สร้างและทำลาย Test Database อย่างไร (แยกจากฐานข้อมูลจริงเสมอ)
- ขั้นตอนที่ 584: การรัน Test — `python manage.py test`, test discovery, รัน test เฉพาะไฟล์/คลาส/method
- ขั้นตอนที่ 585: `TestCase` vs `SimpleTestCase` vs `TransactionTestCase` — ตารางเปรียบเทียบเมื่อไหร่ใช้อะไร
- ขั้นตอนที่ 586: Test Isolation — ทำไมแต่ละ test ถูกครอบด้วย transaction ที่ rollback อัตโนมัติ
- ขั้นตอนที่ 587: การจัดระเบียบไฟล์ Test — `tests.py` เดี่ยว vs แพ็กเกจ `tests/` แยกตามหมวด
- ขั้นตอนที่ 588: `setUpClass()`/`tearDownClass()` สำหรับ setup ที่ใช้ทรัพยากรหนัก ทำครั้งเดียวต่อคลาส
- ขั้นตอนที่ 589: การข้าม Test — `@skip`, `@skipIf`, `@skipUnless`
- ขั้นตอนที่ 590: สรุปและแบบฝึกหัด — เขียน test suite แรกสำหรับ `Post` model

---

## ขั้นตอนที่ 581: `TestCase` class เบื้องต้น — `setUp()`/`tearDown()`

### 581.1 ทำไมต้องมี Automated Testing

ก่อนเข้าโค้ด ต้องเข้าใจก่อนว่าทำไมนักพัฒนามืออาชีพถึงให้ความสำคัญกับ testing มาก
ลองเปรียบเทียบสองแนวทาง:

| แนวทาง | วิธีทำ | ปัญหา |
|---|---|---|
| **Manual Testing** | เปิดเบราว์เซอร์/shell แล้วลองเองทุกครั้งที่แก้โค้ด | ช้า, ลืมเช็คบางเคส, ไม่มีบันทึกว่าเช็คอะไรไปแล้ว, พังซ้ำโดยไม่รู้ตัว (regression) |
| **Automated Testing** | เขียนโค้ดที่ตรวจสอบโค้ดอีกทีหนึ่ง แล้วรันด้วยคำสั่งเดียว | ต้องลงทุนเวลาเขียนตอนแรก แต่รันซ้ำได้ไม่จำกัดครั้ง ฟรี และไม่มีวันลืมเช็ค |

ยิ่งโปรเจกต์โตขึ้น (แบบ `Post` model ของเราที่ตอนนี้มี field เกือบ 10 ตัว และ method
ทางธุรกิจอีกหลายตัว) ค่าใช้จ่ายของ manual testing จะสูงขึ้นเรื่อย ๆ ในขณะที่ automated
test ยังคงรันเร็วเท่าเดิมทุกครั้ง นี่คือเหตุผลที่ทีมงานระดับโลกทุกทีมถือว่า test คือ
ส่วนหนึ่งของโค้ด ไม่ใช่ "งานเสริม" ที่ทำเมื่อมีเวลาว่าง

### 581.2 `unittest`: รากฐานของ Python (ไม่ใช่ของ Django)

**`unittest`** คือโมดูลมาตรฐาน (standard library) ที่มากับ Python เองตั้งแต่ต้น
ไม่เกี่ยวอะไรกับ Django โดยตรง แนวคิดหลักของมันคือ **xUnit pattern** ซึ่งเป็นรูปแบบ
testing ที่ใช้กันทั่วทั้งวงการ (JUnit ของ Java, RSpec แนวคิดคล้ายกันของ Ruby ก็มาจาก
รากเดียวกัน):

```python
# ตัวอย่าง unittest แบบล้วน ๆ ไม่เกี่ยว Django เลย (ไว้เทียบให้เห็นภาพ)
import unittest


class MathTest(unittest.TestCase):
    def test_addition(self):
        self.assertEqual(1 + 1, 2)

    def test_subtraction(self):
        self.assertEqual(5 - 3, 2)


if __name__ == '__main__':
    unittest.main()
```

**กฎพื้นฐานของ `unittest` ที่ Django สืบทอดมาทั้งหมด**:

1. Test ทุกตัวต้องอยู่ใน class ที่สืบทอดจาก `TestCase`
2. ชื่อ method ของ test **ต้องขึ้นต้นด้วย `test_`** เสมอ (ไม่เช่นนั้น test runner
   จะไม่รู้จักและไม่รันให้)
3. ในแต่ละ test method ใช้ **assert method** (เช่น `self.assertEqual(...)`) เพื่อ
   ตรวจสอบว่าผลลัพธ์ตรงกับที่คาดหวังหรือไม่
4. ถ้า assert ล้มเหลว test นั้นจะถูกรายงานว่า **FAIL** ถ้าโค้ดที่ทดสอบ raise exception
   ที่ไม่คาดคิด test นั้นจะถูกรายงานว่า **ERROR** (คนละสถานะกับ FAIL)

### 581.3 `django.test.TestCase`: `unittest.TestCase` เวอร์ชันที่ "รู้จัก Django"

Django ไม่ได้สร้างระบบ testing ของตัวเองขึ้นมาใหม่ทั้งหมด แต่**ต่อยอด**จาก `unittest`
โดยมี `django.test.TestCase` เป็นตัวหลักที่ใช้บ่อยที่สุด:

```python
# django/test/testcases.py (แนวคิดโดยสรุป ไม่ใช่ซอร์สจริงทั้งหมด)
from unittest import TestCase as UnitTestTestCase


class TestCase(TransactionTestCase):
    """เพิ่มความสามารถเรื่อง Database transaction/rollback เข้าไปจาก unittest.TestCase เดิม"""
```

`django.test.TestCase` เพิ่มความสามารถสำคัญที่ `unittest.TestCase` เปล่า ๆ ไม่มี:

- เชื่อมกับ **Test Database** อัตโนมัติ (รายละเอียดเต็มในขั้นตอนที่ 583)
- ครอบทุก test method ด้วย **database transaction ที่ rollback อัตโนมัติ** (ขั้นตอนที่ 586)
- มี `self.client` (Django Test Client) สำหรับจำลอง HTTP request ไปยัง view (จะใช้จริง
  ใน Part 060)
- มี assert method เพิ่มเติมเฉพาะของ Django เช่น `assertQuerySetEqual`,
  `assertContains`, `assertRedirects`, `assertFormError` (Part 060)

โครงสร้างไฟล์ทดสอบพื้นฐานของ Django หน้าตาแบบนี้:

```python
# blog/tests.py
from django.test import TestCase
from blog.models import Post


class PostModelTest(TestCase):
    def test_str_returns_title(self):
        post = Post.objects.create(title="บทความทดสอบ", content="เนื้อหาบางส่วน")
        self.assertEqual(str(post), "บทความทดสอบ")
```

รันด้วยคำสั่ง (รายละเอียดเต็มในขั้นตอนที่ 584):

```bash
python manage.py test blog
```

### 581.4 `setUp()`: เตรียมข้อมูลก่อน test แต่ละตัว

ปัญหาที่พบทันทีเมื่อมี test หลายตัวในคลาสเดียวกัน คือแต่ละ test มักต้องการข้อมูล
ตั้งต้นชุดเดียวกัน ถ้าเขียนสร้างข้อมูลซ้ำในทุก method จะขัดกับหลักการ **DRY**
ที่เรียนมาตั้งแต่ Part 001 `setUp()` แก้ปัญหานี้:

```python
from django.test import TestCase
from blog.models import Post


class PostModelTest(TestCase):
    def setUp(self):
        """
        Django/unittest จะเรียก setUp() นี้ให้อัตโนมัติ 'ก่อน' test method ทุกตัว
        ในคลาสนี้ทำงาน — เรียกใหม่ทุกครั้ง ไม่ใช่เรียกครั้งเดียวแล้วใช้ร่วมกัน
        """
        self.post = Post.objects.create(
            title="เรียนรู้ Django Testing",
            content="เนื้อหาต้นฉบับสำหรับทดสอบ" * 20,
        )

    def test_str_returns_title(self):
        self.assertEqual(str(self.post), "เรียนรู้ Django Testing")

    def test_slug_generated_automatically(self):
        self.assertTrue(self.post.slug)  # slug ต้องไม่ว่างเปล่า (สร้างจาก save() อัตโนมัติ)

    def test_default_is_not_published(self):
        self.assertFalse(self.post.is_published)
```

**จุดสำคัญที่สุดของ `setUp()`**: มันถูกเรียก**ใหม่ทั้งหมด**ก่อน**แต่ละ** test method
ไม่ใช่แค่ครั้งเดียวตอนต้นคลาส หมายความว่าถ้าคลาสนี้มี 3 test methods `setUp()`
จะถูกเรียกทั้งหมด 3 ครั้ง (ครั้งละ 1 ก่อนแต่ละ method) การทำแบบนี้ทำให้แต่ละ test
เริ่มต้นด้วยข้อมูลที่ **สดใหม่และเหมือนกันทุกครั้ง** ไม่มี test ไหนได้รับผลกระทบจาก
การเปลี่ยนแปลงข้อมูลของ test ก่อนหน้า (แนวคิดนี้จะเจาะลึกเต็มรูปแบบในขั้นตอนที่ 586)

### 581.5 `tearDown()`: ทำความสะอาดหลัง test แต่ละตัว

```python
from django.test import TestCase
from blog.models import Post


class PostFileHandlingTest(TestCase):
    def setUp(self):
        self.post = Post.objects.create(title="ทดสอบไฟล์", content="เนื้อหา")

    def tearDown(self):
        """
        เรียก 'หลัง' test method ทุกตัวเสมอ ไม่ว่า test นั้นจะผ่านหรือ FAIL/ERROR ก็ตาม
        ใช้สำหรับปิดทรัพยากรที่เปิดไว้ เช่น ไฟล์ ปิด connection ภายนอก ลบไฟล์ชั่วคราว
        """
        # ตัวอย่าง: ถ้า test นี้สร้างไฟล์จริงไว้ที่ MEDIA_ROOT ต้องลบทิ้งเอง
        if self.post.cover_image:
            self.post.cover_image.delete(save=False)
```

**ในทางปฏิบัติ** สำหรับข้อมูลในฐานข้อมูล (เช่น `Post` object ที่สร้างใน `setUp()`)
คุณ**ไม่จำเป็นต้องเขียน `tearDown()` เพื่อลบเอง** เพราะ `django.test.TestCase` จัดการ
ให้อัตโนมัติผ่านกลไก transaction rollback (ขั้นตอนที่ 586) — `tearDown()` จึงมีประโยชน์
จริง ๆ กับทรัพยากร**นอกฐานข้อมูล** เท่านั้น เช่น ไฟล์บนดิสก์, การเชื่อมต่อ API ภายนอก,
mock ที่ patch ไว้แบบ manual (ไม่ผ่าน decorator/context manager)

### 581.6 ลำดับการทำงานเต็มรูปแบบของ 1 Test Class

```
สำหรับคลาสที่มี 3 test methods:

setUpClass()                    ← เรียกครั้งเดียว ก่อนทุก test ในคลาส (ขั้นตอนที่ 588)
    │
    ├── setUp()                 ← ก่อน test_a
    ├── test_a()
    ├── tearDown()               ← หลัง test_a
    │
    ├── setUp()                 ← ก่อน test_b
    ├── test_b()
    ├── tearDown()               ← หลัง test_b
    │
    ├── setUp()                 ← ก่อน test_c
    ├── test_c()
    ├── tearDown()               ← หลัง test_c
    │
tearDownClass()                 ← เรียกครั้งเดียว หลังทุก test ในคลาสเสร็จหมด
```

ตารางนี้จะช่วยให้เข้าใจภาพรวมก่อนไปเรียน `setUpClass()`/`tearDownClass()` แบบเต็ม
รูปแบบในขั้นตอนที่ 588

---

## ขั้นตอนที่ 582: Assert methods ที่ใช้บ่อย

### 582.1 `assertEqual()` / `assertNotEqual()`: หัวใจของ assert เกือบทุกคำสั่ง

```python
from django.test import TestCase
from blog.models import Post


class AssertBasicsTest(TestCase):
    def test_assert_equal(self):
        post = Post.objects.create(title="Django คือดี", content="เนื้อหา")
        self.assertEqual(post.title, "Django คือดี")
        self.assertEqual(1 + 1, 2)
        self.assertEqual([1, 2, 3], [1, 2, 3])   # เทียบ list ได้ตรง ๆ

    def test_assert_not_equal(self):
        post = Post.objects.create(title="A", content="เนื้อหา")
        self.assertNotEqual(post.title, "B")
```

**ทำไมไม่เขียน `assert a == b` ธรรมดาแบบ Python เฉย ๆ?** เพราะ `assertEqual()` ให้
**ข้อความ error ที่อ่านง่ายกว่ามาก** เมื่อ test ล้มเหลว ลองเทียบ:

```
# ใช้ assert ธรรมดา
AssertionError

# ใช้ self.assertEqual(post.title, "Django คือดี")
AssertionError: 'Django คือด' != 'Django คือดี'
- Django คือด
+ Django คือดี
?           +
```

ข้อความที่สองบอกทันทีว่าค่าไหนผิด ผิดตรงไหน ประหยัดเวลา debug ได้มาก — นี่คือเหตุผล
ที่หลักสูตรนี้ (และทีมงานมืออาชีพทุกทีม) **ห้ามใช้ `assert` ธรรมดาในไฟล์ test เด็ดขาด**
ให้ใช้ assert method ของ `unittest`/Django เสมอ

### 582.2 `assertTrue()` / `assertFalse()`: ตรวจสอบค่าความจริง

```python
class AssertBooleanTest(TestCase):
    def test_assert_true(self):
        post = Post.objects.create(title="A", content="X" * 500)
        self.assertTrue(post.slug)          # slug ต้องไม่ใช่ค่า falsy ('' หรือ None)
        self.assertTrue(len(post.excerpt) > 0)

    def test_assert_false(self):
        post = Post.objects.create(title="A", content="เนื้อหา")
        self.assertFalse(post.is_published)  # ค่า default ต้องเป็น False
        self.assertFalse(post.is_premium)
```

**ข้อควรระวังสำคัญ**: `assertTrue(a == b)` กับ `assertEqual(a, b)` **ให้ผลลัพธ์เหมือนกัน
ตอน test ผ่าน แต่ต่างกันมากตอน test ล้มเหลว**:

```python
# ❌ ไม่แนะนำ: ล้มเหลวแล้วบอกแค่ "False is not true" ไม่รู้ค่าจริงคืออะไร
self.assertTrue(post.title == "Django คือดี")

# ✅ แนะนำ: ล้มเหลวแล้วบอกค่าทั้งสองฝั่งชัดเจน
self.assertEqual(post.title, "Django คือดี")
```

**กฎเหล็ก**: ใช้ `assertTrue`/`assertFalse` เฉพาะกับค่าที่เป็น boolean โดยธรรมชาติ
(เช่น `post.is_published`, `os.path.exists(...)`) ถ้าเป็นการเทียบค่าสองค่า ให้ใช้
`assertEqual` เสมอ

### 582.3 `assertIsNone()` / `assertIsNotNone()`

```python
class AssertNoneTest(TestCase):
    def test_optional_field_defaults_to_none(self):
        post = Post.objects.create(title="A", content="B")
        # สมมติมี field ที่ optional และยังไม่ตั้งค่า
        self.assertIsNone(post.pk is None and None)  # ตัวอย่างสาธิตรูปแบบ ไม่ใช่การใช้งานจริง

    def test_saved_post_has_primary_key(self):
        post = Post.objects.create(title="A", content="B")
        self.assertIsNotNone(post.pk)   # หลัง save() แล้ว pk ต้องไม่ใช่ None
        self.assertIsNotNone(post.created_at)
```

`assertIsNone(x)` เทียบเท่ากับ `assertEqual(x, None)` แต่**อ่านเจตนาชัดเจนกว่า**
และสำคัญกว่านั้น: `assertIsNone`/`assertIsNotNone` ใช้ตัวดำเนินการ `is`/`is not`
ภายใน ไม่ใช่ `==` ซึ่งถูกต้องตามหลักการเปรียบเทียบกับ `None` ใน Python (PEP 8 แนะนำ
`is None` เสมอ ไม่ใช่ `== None`)

### 582.4 `assertRaises()`: ตรวจสอบว่า exception ถูก raise จริง

```python
from django.core.exceptions import ValidationError
from django.db.utils import IntegrityError
from django.test import TestCase
from blog.models import Post


class AssertRaisesTest(TestCase):
    def test_duplicate_slug_raises_integrity_error(self):
        Post.objects.create(title="A", slug="same-slug", content="X")
        with self.assertRaises(IntegrityError):
            # unique=True บน slug ต้องทำให้ database ปฏิเสธแถวที่สอง
            Post.objects.create(title="B", slug="same-slug", content="Y")

    def test_blank_title_fails_full_clean(self):
        post = Post(title="", content="เนื้อหา")
        with self.assertRaises(ValidationError):
            post.full_clean()   # title ไม่มี blank=True จึงต้องไม่ผ่าน validation
```

`assertRaises` ใช้เป็น **context manager** (คู่กับ `with`) เสมอในโค้ดยุคปัจจุบัน
โค้ดภายใน `with` block ต้อง raise exception ตามชนิดที่ระบุ ไม่เช่นนั้น test จะ FAIL
ทันที (ถ้าไม่มี exception เกิดขึ้นเลย ก็ถือว่า FAIL เพราะไม่เป็นไปตามที่คาดหวัง)

ยังมี `assertRaisesMessage()` ที่ตรวจสอบทั้งชนิด exception **และ** ข้อความ error
พร้อมกันในคำสั่งเดียว:

```python
def test_validation_error_message(self):
    post = Post(title="", content="เนื้อหา")
    with self.assertRaisesMessage(ValidationError, 'This field cannot be blank'):
        post.full_clean()
```

### 582.5 Assert methods เพิ่มเติมที่ใช้บ่อยไม่แพ้กัน

| Assert Method | ตรวจสอบว่า | ตัวอย่าง |
|---|---|---|
| `assertIn(a, b)` | `a` อยู่ใน `b` | `self.assertIn('django', post.content.lower())` |
| `assertNotIn(a, b)` | `a` ไม่อยู่ใน `b` | `self.assertNotIn('DRAFT', post.get_content_type_display())` |
| `assertGreater(a, b)` | `a > b` | `self.assertGreater(post.view_count, 0)` |
| `assertGreaterEqual(a, b)` | `a >= b` | `self.assertGreaterEqual(len(post.excerpt), 0)` |
| `assertLess(a, b)` / `assertLessEqual(a, b)` | `a < b` / `a <= b` | `self.assertLess(post.reading_time_minutes, 60)` |
| `assertAlmostEqual(a, b)` | `a` ใกล้เคียง `b` (ใช้กับ float ที่มีความคลาดเคลื่อน) | `self.assertAlmostEqual(0.1 + 0.2, 0.3, places=5)` |
| `assertIsInstance(a, cls)` | `a` เป็น instance ของ `cls` | `self.assertIsInstance(post.price, Decimal)` |
| `assertListEqual(a, b)` | list สองอันเหมือนกันทุกตำแหน่ง | `self.assertListEqual(list(qs), [post1, post2])` |
| `assertQuerySetEqual(qs, values)` | QuerySet ตรงกับค่าที่คาดหวัง (Django-specific) | `self.assertQuerySetEqual(Post.objects.all(), [post], transform=lambda p: p)` |
| `assertCountEqual(a, b)` | สอง list มีสมาชิกเหมือนกัน **ไม่สนลำดับ** | `self.assertCountEqual([1, 2, 3], [3, 1, 2])` |

```python
from decimal import Decimal
from django.test import TestCase
from blog.models import Post


class AssertMoreTest(TestCase):
    def test_price_is_decimal_type(self):
        post = Post.objects.create(
            title="พรีเมียม", content="เนื้อหา", is_premium=True, price=Decimal('99.00')
        )
        self.assertIsInstance(post.price, Decimal)
        self.assertGreater(post.price, 0)

    def test_queryset_matches_expected_posts(self):
        p1 = Post.objects.create(title="A", content="1")
        p2 = Post.objects.create(title="B", content="2")
        self.assertQuerySetEqual(
            Post.objects.order_by('title'),
            [p1, p2],
        )
```

> **หมายเหตุเรื่องชื่อ**: ก่อน Django 4.1 assert method นี้ชื่อ `assertQuerysetEqual`
> (ตัว s ใน "set" เป็นพิมพ์เล็ก) ตั้งแต่ Django 4.1 เปลี่ยนเป็น `assertQuerySetEqual`
> (S ใหญ่) และตั้งแต่ **Django 5.0 ชื่อเดิมถูกลบทิ้งแล้ว** ใช้ไม่ได้อีกต่อไป — หลักสูตรนี้
> ใช้ Django 5.x จึงต้องใช้ชื่อใหม่เสมอ ถ้าเจอโค้ดเก่าที่ error ว่า `AttributeError` เรื่องนี้
> คือสาเหตุที่พบบ่อยที่สุด

Assert method ที่เกี่ยวกับการทดสอบ **View/HTTP Response** เช่น `assertContains()`,
`assertRedirects()`, `assertTemplateUsed()`, `assertFormError()` ต้องใช้คู่กับ
`self.client` ซึ่งเป็นเนื้อหาหลักของ **Part 060 (Testing Views, Models และ Forms)**
Part นี้ตั้งใจโฟกัสที่ assert method ระดับ Python/Model ล้วน ๆ ก่อน

---

## ขั้นตอนที่ 583: Django สร้างและทำลาย Test Database อย่างไร

### 583.1 หลักการที่สำคัญที่สุด: Test Database แยกขาดจากฐานข้อมูลจริงเสมอ

นี่คือกฎเหล็กที่ทำให้ automated testing ปลอดภัย 100%:

> **ทุกครั้งที่รัน `python manage.py test` Django จะสร้างฐานข้อมูลใหม่ทั้งฐานขึ้นมา
> ใช้เฉพาะระหว่างทดสอบ แล้ว "ทำลายทิ้ง" ทันทีที่ test จบ — ฐานข้อมูล development
> หรือ production ของคุณจะไม่ถูกแตะต้องแม้แต่บรรทัดเดียว**

ลองดูลำดับเหตุการณ์เต็มรูปแบบเมื่อรัน `python manage.py test`:

```
1. Django อ่านค่า DATABASES ใน settings.py
2. สร้างฐานข้อมูลใหม่ชื่อ "test_" + ชื่อฐานข้อมูลเดิม
   (เช่น NAME='myproject' → สร้าง 'test_myproject')
3. รัน migration ทั้งหมดของทุกแอปลงบนฐานข้อมูล test ที่สร้างขึ้นใหม่นี้
   (เหมือนกับตอนคุณรัน `python manage.py migrate` ครั้งแรกบนเครื่องใหม่)
4. เริ่มรัน test ทั้งหมดโดยใช้ฐานข้อมูล test นี้
5. เมื่อ test ครบทุกตัวแล้ว Django จะ "DROP DATABASE" ฐานข้อมูล test ทิ้งทันที
   (ยกเว้นระบุ --keepdb จะไม่ลบ — ดูขั้นตอนที่ 583.4)
```

### 583.2 ตัวอย่าง Output จริงตอนรัน Test

```
$ python manage.py test blog
Creating test database for alias 'default'...
System check identified no issues (0 silenced).
...
----------------------------------------------------------------------
Ran 3 tests in 0.042s

OK
Destroying test database for alias 'default'...
```

สังเกตข้อความ **"Creating test database"** ที่ต้นและ **"Destroying test database"**
ที่ท้าย — นี่คือหลักฐานตรง ๆ ว่า Django สร้างและทำลายฐานข้อมูลจริงทุกครั้งที่รันคำสั่งนี้
ไม่ใช่แค่คำเปรียบเปรย

### 583.3 SQLite vs PostgreSQL/MySQL: ความแตกต่างของ Test Database

| ประเด็น | SQLite | PostgreSQL / MySQL |
|---|---|---|
| ตำแหน่งฐานข้อมูล test | สร้างเป็น **in-memory database** (`:memory:`) โดยค่าเริ่มต้น | สร้างเป็นฐานข้อมูลจริงบนเซิร์ฟเวอร์ ชื่อ `test_<ชื่อฐานข้อมูลเดิม>` |
| ความเร็ว | เร็วมาก (ไม่มีการเขียนดิสก์เลย) | ช้ากว่า SQLite แต่ใกล้เคียง production มากกว่า |
| สิทธิ์ผู้ใช้ที่ต้องมี | ไม่ต้องมีสิทธิ์พิเศษ | ผู้ใช้ในฐานข้อมูลต้องมีสิทธิ์ **`CREATEDB`** (Postgres) หรือ `CREATE`/`DROP` (MySQL) |
| เหมาะกับ | รันบนเครื่อง dev / CI ที่ต้องการความเร็ว | รันเมื่อต้องการพฤติกรรมที่ตรงกับ production 100% (คำแนะนำของหลักสูตรนี้) |

หลักสูตรนี้แนะนำให้ **ใช้ฐานข้อมูลชนิดเดียวกับ production เสมอเมื่อรัน test** (เช่น
ถ้า production ใช้ PostgreSQL ให้ test บน PostgreSQL ด้วย) เพราะพฤติกรรมบางอย่างต่างกัน
ระหว่างฐานข้อมูล เช่น การจัดการ `UNIQUE constraint`, `CASE SENSITIVITY` ของการค้นหา
ข้อความ, หรือชนิดข้อมูล — test ที่ผ่านบน SQLite อาจ FAIL จริงบน PostgreSQL ก็ได้

### 583.4 ตั้งค่า Test Database เพิ่มเติมใน `settings.py`

```python
# settings.py
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.postgresql',
        'NAME': 'myproject',
        'USER': 'myproject_user',
        'PASSWORD': 'secret',
        'HOST': 'localhost',
        'PORT': '5432',
        'TEST': {
            # กำหนดชื่อฐานข้อมูล test เอง แทนชื่อ default ที่ Django สร้างให้อัตโนมัติ
            'NAME': 'myproject_test_db',
        },
    }
}
```

คำสั่งที่ควรรู้คู่กับการจัดการ test database:

```bash
# --keepdb: ไม่ลบฐานข้อมูล test ทิ้งหลังรันเสร็จ (เก็บไว้ใช้รอบถัดไป)
python manage.py test --keepdb

# ผลลัพธ์ที่ต่างออกไปเมื่อใช้ --keepdb (ครั้งที่สองเป็นต้นไปจะเร็วขึ้นมาก
# เพราะข้าม migration ที่ไม่เปลี่ยนแปลง):
# "Using existing test database for alias 'default'..."
```

**เหตุผลที่ควรใช้ `--keepdb` ระหว่างพัฒนา**: การสร้างฐานข้อมูลใหม่และรัน migration
ทั้งหมดใหม่ทุกครั้งใช้เวลา โดยเฉพาะโปรเจกต์ที่มี migration หลายร้อยไฟล์ `--keepdb`
จะรัน migration เฉพาะที่เปลี่ยนแปลงจริง ๆ เท่านั้น ทำให้ loop การเขียน-รัน test เร็วขึ้น
มาก แต่ **ในระบบ CI/CD (Part 088) ควรสร้างฐานข้อมูลใหม่ทุกครั้งเสมอ** (ไม่ใช้
`--keepdb`) เพื่อยืนยันว่า migration ทั้งหมดยังรันได้สำเร็จจากศูนย์จริง

### 583.5 ทำไมต้องแยกฐานข้อมูล test ออกจากฐานข้อมูลจริงเด็ดขาด

ลองจินตนาการว่า Django ไม่แยกฐานข้อมูล test ออกมา แล้วรัน test ตรงบนฐานข้อมูลจริง:

```python
def test_delete_removes_post(self):
    post = Post.objects.create(title="ทดสอบลบ", content="X")
    post.delete()
    self.assertEqual(Post.objects.count(), 0)  # ถ้ารันบน DB จริง ← ลบข้อมูลจริงหมดเลย!
```

ถ้า test แบบนี้รันบนฐานข้อมูล production จริง จะเกิดหายนะทันที (`Post.objects.count()`
ที่คาดหวังว่าเป็น 0 จะลบทุกอย่างในตารางจริงทิ้งเพื่อให้ assertion ผ่าน) การที่ Django
**บังคับ**สร้างฐานข้อมูลแยกต่างหากทุกครั้ง (ไม่มีทางปิดพฤติกรรมนี้ได้เลยในโหมด `test`)
คือ **safety net ระดับ framework** ที่ป้องกันความผิดพลาดร้ายแรงแบบนี้ไม่ให้เกิดขึ้นได้
ไม่ว่านักพัฒนาจะเขียน test พลาดแค่ไหนก็ตาม

---

## ขั้นตอนที่ 584: การรัน Test — `python manage.py test`, test discovery, รัน test เฉพาะไฟล์/คลาส/method

### 584.1 คำสั่งพื้นฐานที่สุด

```bash
# รัน test ทั้งหมดในทุกแอปของโปรเจกต์
python manage.py test

# รัน test เฉพาะแอป blog
python manage.py test blog

# รัน test เฉพาะคลาส
python manage.py test blog.tests.PostModelTest

# รัน test เฉพาะ method เดียว
python manage.py test blog.tests.PostModelTest.test_str_returns_title
```

รูปแบบ label ที่ใช้กับ `manage.py test` เขียนเป็น **Python import path** คั่นด้วยจุด
ไม่ใช่ path ของไฟล์ (ไม่ใช่ `blog/tests.py::PostModelTest` แบบที่บาง test runner อื่นใช้)

### 584.2 Test Discovery: Django หา Test เจอได้อย่างไร

Django ใช้กลไก **test discovery** ของ `unittest` (ผ่าน `unittest.defaultTestLoader.discover()`)
เพื่อค้นหาไฟล์ test โดยอัตโนมัติ กฎการค้นหามีดังนี้:

1. เริ่มค้นจาก **root directory ของโปรเจกต์** (หรือแอปที่ระบุ ถ้าใส่ label)
2. ไล่เข้าไปทุกโฟลเดอร์ที่เป็น **Python package** (มีไฟล์ `__init__.py`)
3. หาไฟล์ที่ชื่อตรงกับ pattern `test*.py` (ค่าเริ่มต้น) — เช่น `tests.py`,
   `test_models.py`, `test_views.py`, `testing_utils.py` ก็นับด้วย (ขึ้นต้นด้วย
   `test` เฉย ๆ ก็พอ)
4. ในแต่ละไฟล์ หา class ที่สืบทอดจาก `unittest.TestCase` (รวมถึง `django.test.TestCase`
   และตัวแปรอื่น ๆ ที่จะเรียนในขั้นตอนที่ 585)
5. ใน class นั้น หา method ที่ชื่อขึ้นต้นด้วย `test_`

```bash
# เปลี่ยน pattern การค้นหาไฟล์ (ค่า default คือ test*.py)
python manage.py test --pattern="check_*.py"
```

### 584.3 Verbosity: ควบคุมปริมาณ output

```bash
python manage.py test -v 0   # เงียบสุด แสดงแค่ผลรวมสุดท้าย
python manage.py test -v 1   # ค่า default: จุด (.) ต่อ test ที่ผ่าน, F ต่อ test ที่ FAIL
python manage.py test -v 2   # แสดงชื่อ test method ทุกตัวที่รัน พร้อมสถานะ
python manage.py test -v 3   # verbose สุด รวมถึง SQL query และรายละเอียด database setup
```

ตัวอย่าง output ที่ `-v 2`:

```
test_default_is_not_published (blog.tests.PostModelTest) ... ok
test_slug_generated_automatically (blog.tests.PostModelTest) ... ok
test_str_returns_title (blog.tests.PostModelTest) ... ok
```

### 584.4 Flag ที่ใช้บ่อยในงานจริง

| Flag | ความหมาย |
|---|---|
| `--failfast` | หยุดรันทันทีที่เจอ test แรกที่ FAIL (ไม่ต้องรอจนครบทุกตัว) เหมาะกับตอน debug |
| `--keepdb` | ไม่ลบฐานข้อมูล test ทิ้งหลังรันเสร็จ (ขั้นตอนที่ 583.4) |
| `--parallel` | รัน test หลายตัวพร้อมกันด้วยหลาย process เพื่อความเร็ว (Django จะสร้างฐานข้อมูล test แยกต่อ process ให้อัตโนมัติ) |
| `--parallel 4` | ระบุจำนวน process ที่ต้องการใช้ชัดเจน (ค่า default ถ้าไม่ระบุตัวเลขคือเท่าจำนวน CPU core) |
| `--reverse` | รัน test ย้อนลำดับ (ใช้ตรวจว่า test มีการพึ่งพาลำดับกันโดยไม่ตั้งใจหรือไม่ — ถ้า reverse แล้ว FAIL แปลว่า test isolation มีปัญหา) |
| `--tag=slow` | รันเฉพาะ test ที่ติด tag ที่กำหนด (ต้องแปะ `@tag('slow')` ไว้ที่ test method/class ก่อน) |
| `--exclude-tag=slow` | รันทุกตัว **ยกเว้น** test ที่ติด tag ที่กำหนด |
| `-k KEYWORD` | รันเฉพาะ test ที่ชื่อ method/class ตรงกับ substring ที่กำหนด (ตั้งแต่ Django 4.0) |
| `--debug-sql` | แสดง SQL query ทั้งหมดที่ query set รันระหว่าง test ที่ FAIL — มีประโยชน์มากตอน debug query ผิด |

```bash
# ตัวอย่างการใช้ -k: รันทุก test ที่ชื่อมีคำว่า "slug" อยู่ ไม่ว่าจะอยู่คลาสไหน
python manage.py test -k slug

# ตัวอย่างการใช้ tag
python manage.py test --tag=slow --parallel
```

```python
from django.test import TestCase, tag


class PostModelTest(TestCase):
    @tag('fast')
    def test_str_returns_title(self):
        ...

    @tag('slow', 'integration')
    def test_heavy_report_generation(self):
        ...
```

### 584.5 ตัวอย่างการอ่านผลลัพธ์เมื่อ Test ล้มเหลว

```
$ python manage.py test blog
Creating test database for alias 'default'...
System check identified no issues (0 silenced).
F..
======================================================================
FAIL: test_default_reading_time_is_one_minute (blog.tests.PostModelTest)
----------------------------------------------------------------------
Traceback (most recent call last):
  File "/app/blog/tests.py", line 15, in test_default_reading_time_is_one_minute
    self.assertEqual(post.reading_time_minutes, 2)
AssertionError: 1 != 2

----------------------------------------------------------------------
Ran 3 tests in 0.031s

FAILED (failures=1)
Destroying test database for alias 'default'...
```

สังเกต 3 ส่วนสำคัญ: **`F..`** (test แรก FAIL ตัวที่สองสามผ่าน), **Traceback** ที่บอก
บรรทัดและไฟล์ชัดเจน, และสรุปท้าย **`FAILED (failures=1)`** — Django คืนค่า exit code
ที่ไม่ใช่ 0 เมื่อมี test ล้มเหลว ซึ่งเป็นสิ่งสำคัญมากสำหรับ CI/CD pipeline (Part 088)
ที่ใช้ exit code ตัดสินว่าควรหยุด deploy หรือไม่

---

## ขั้นตอนที่ 585: `TestCase` vs `SimpleTestCase` vs `TransactionTestCase` — ตารางเปรียบเทียบเมื่อไหร่ใช้อะไร

### 585.1 ทำไม Django มี TestCase หลายแบบ

Django ให้ TestCase มาหลายชนิดเพราะ**ไม่ใช่ทุก test ต้องการความสามารถเดียวกัน**
บาง test ไม่แตะฐานข้อมูลเลย (เช่น test ฟังก์ชัน utility ล้วน ๆ) บาง test ต้องการ
database transaction ปกติ และบาง test ต้องการทดสอบพฤติกรรม transaction เองโดยตรง
— การเลือกชนิดที่เหมาะสมช่วยให้ test **เร็วขึ้น** และ**สื่อความหมายชัดเจนขึ้น**ว่า
test นั้นทำอะไรได้บ้าง

### 585.2 ตารางเปรียบเทียบเต็มรูปแบบ

| คุณสมบัติ | `SimpleTestCase` | `TestCase` | `TransactionTestCase` |
|---|---|---|---|
| Import จาก | `django.test` | `django.test` | `django.test` |
| เข้าถึงฐานข้อมูลได้ | ❌ ค่า default (ห้ามเด็ดขาด จะ raise error ถ้าพยายาม query) | ✅ ได้เต็มที่ | ✅ ได้เต็มที่ |
| กลไก isolation ระหว่าง test | ไม่มี (ไม่มี DB ให้ isolate) | ครอบด้วย **transaction + rollback** หลังแต่ละ test (เร็วมาก) | **TRUNCATE ตาราง** ทั้งหมดแล้วโหลด fixture ใหม่หลังแต่ละ test (ช้ากว่า) |
| ความเร็ว | เร็วที่สุด | เร็ว (เร็วกว่า TransactionTestCase มาก) | ช้าที่สุดในสามแบบ |
| ทดสอบ `transaction.on_commit()` ได้จริงไหม | ไม่เกี่ยวข้อง | ❌ ไม่ commit จริง callback จึงไม่ถูกเรียก (ยกเว้นใช้ `captureOnCommitCallbacks`) | ✅ ได้ (มี commit จริงเกิดขึ้น) |
| ใช้ `self.client` (Test Client) ได้ไหม | ✅ ได้ (แต่ view ที่แตะ DB จะ error ถ้าไม่ตั้ง `databases`) | ✅ ได้เต็มที่ | ✅ ได้เต็มที่ |
| ใช้เมื่อไหร่ | ทดสอบ form validation ล้วน ๆ, template tag, utility function, regex, serializer ที่ไม่แตะ DB | **ค่า default ที่ควรใช้เกือบทุกกรณี** ทดสอบ Model, View, Form ที่ต้องมีข้อมูลในฐานข้อมูล | ทดสอบ threading, multi-database transaction, LiveServerTestCase (Selenium, Part 064), หรือพฤติกรรมที่ต้องพึ่ง commit จริง |

### 585.3 ตัวอย่าง `SimpleTestCase`: ทดสอบฟังก์ชันที่ไม่แตะฐานข้อมูล

```python
from django.test import SimpleTestCase
from django.utils.text import slugify


class SlugifyHelperTest(SimpleTestCase):
    def test_thai_and_english_mixed_slugify(self):
        # slugify ไม่แตะฐานข้อมูลเลย เหมาะกับ SimpleTestCase ที่เร็วกว่า
        result = slugify("Hello World 2026")
        self.assertEqual(result, "hello-world-2026")

    def test_empty_string_slugify(self):
        self.assertEqual(slugify(""), "")
```

ถ้าพยายาม query ฐานข้อมูลใน `SimpleTestCase` โดยไม่ได้เปิดสิทธิ์ไว้ จะได้ error ทันที:

```python
class BrokenSimpleTest(SimpleTestCase):
    def test_this_will_error(self):
        from blog.models import Post
        Post.objects.create(title="A", content="B")   # ❌ raise AssertionError
```

```
AssertionError: Database queries to 'default' are not allowed in SimpleTestCase
subclasses. Either subclass TestCase or TransactionTestCase to ensure proper test
isolation or add 'default' to BrokenSimpleTest.databases to silence this failure.
```

ถ้ามีเหตุผลจำเป็นต้องแตะฐานข้อมูลใน `SimpleTestCase` (พบน้อยมาก) เปิดสิทธิ์ได้ด้วย:

```python
class RareCaseTest(SimpleTestCase):
    databases = {'default'}   # เปิดสิทธิ์เข้าถึงฐานข้อมูล default ชั่วคราว

    def test_read_only_check(self):
        from blog.models import Post
        self.assertEqual(Post.objects.count(), 0)
```

### 585.4 ตัวอย่าง `TestCase`: ตัวเลือกมาตรฐานสำหรับ 95% ของงานจริง

```python
from django.test import TestCase
from blog.models import Post


class PostModelStandardTest(TestCase):
    def test_create_and_retrieve_post(self):
        Post.objects.create(title="มาตรฐาน", content="เนื้อหา")
        self.assertEqual(Post.objects.count(), 1)
```

นี่คือคลาสที่ใช้มาตลอดทั้งขั้นตอนที่ 581-584 แล้ว และเป็นคลาสที่คุณจะใช้ **มากที่สุด
ตลอด Phase 7** เพราะ blog application ของเราแตะฐานข้อมูลแทบทุกฟีเจอร์

### 585.5 ตัวอย่าง `TransactionTestCase`: เมื่อต้องทดสอบ transaction เอง

```python
from django.db import transaction
from django.test import TransactionTestCase
from blog.models import Post


class PostTransactionBehaviorTest(TransactionTestCase):
    def test_rollback_on_exception_inside_atomic_block(self):
        with self.assertRaises(ValueError):
            with transaction.atomic():
                Post.objects.create(title="จะถูก rollback", content="X")
                raise ValueError("จำลอง error ระหว่างทำ transaction")

        # เพราะ atomic() rollback ทั้งหมดเมื่อเกิด exception ภายใน จึงไม่มี Post ถูกบันทึกจริง
        self.assertEqual(Post.objects.count(), 0)
```

**ทำไมต้องใช้ `TransactionTestCase` แทน `TestCase` ธรรมดาในตัวอย่างนี้?** เพราะ
`django.test.TestCase` ครอบทุก test ด้วย transaction ของตัวเองอยู่แล้ว (ขั้นตอนที่ 586)
ทำให้ `transaction.atomic()` ที่เขียนซ้อนเข้าไปข้างในกลายเป็นแค่ **savepoint** ซ้อนกัน
ไม่ใช่ transaction จริงที่ commit/rollback อย่างสมบูรณ์ — พฤติกรรมบางอย่างของ
transaction จริง (โดยเฉพาะที่เกี่ยวกับ `on_commit()` หรือพฤติกรรมข้าม connection)
จึงทดสอบได้แม่นยำเฉพาะใน `TransactionTestCase` เท่านั้น

### 585.6 ทางเลือกสมัยใหม่: `captureOnCommitCallbacks` (Django 3.2+)

ก่อน Django 3.2 การทดสอบ `transaction.on_commit()` แทบทุกครั้งบังคับให้ต้องใช้
`TransactionTestCase` ที่ช้ากว่า ปัจจุบันมีทางลัดที่เร็วกว่ามาก:

```python
from django.db import transaction
from django.test import TestCase   # ใช้ TestCase ธรรมดาได้เลย ไม่ต้อง TransactionTestCase
from blog.models import Post


class OnCommitCallbackTest(TestCase):
    def test_on_commit_callback_runs(self):
        with self.captureOnCommitCallbacks(execute=True) as callbacks:
            with transaction.atomic():
                post = Post.objects.create(title="A", content="B")
                transaction.on_commit(lambda: post.increment_view_count())

        self.assertEqual(len(callbacks), 1)
        post.refresh_from_db()
        self.assertEqual(post.view_count, 1)
```

`captureOnCommitCallbacks(execute=True)` จำลองว่า transaction ได้ commit จริงแล้ว
บังคับให้ callback ที่ลงทะเบียนด้วย `transaction.on_commit()` ถูกเรียกทันที ทำให้
ทดสอบได้เร็วเท่า `TestCase` ปกติ โดยไม่ต้องเสียความเร็วไปกับ `TransactionTestCase`

---

## ขั้นตอนที่ 586: Test Isolation — ทำไมแต่ละ test ถูกครอบด้วย transaction ที่ rollback อัตโนมัติ

### 586.1 ปัญหาที่ Test Isolation แก้ไข

ลองจินตนาการว่า test ทั้งหมดในคลาสเดียวกัน**ใช้ฐานข้อมูลร่วมกันแบบไม่มีการทำความสะอาด**:

```python
class BadIsolationExample(TestCase):
    def test_a_creates_post(self):
        Post.objects.create(title="โพสต์ A", content="X")
        self.assertEqual(Post.objects.count(), 1)   # ผ่าน ถ้ารันตัวแรก

    def test_b_creates_another_post(self):
        Post.objects.create(title="โพสต์ B", content="Y")
        self.assertEqual(Post.objects.count(), 1)   # ควรผ่าน... แต่ถ้า test_a รันก่อน
                                                       # จะมี 2 แถวในฐานข้อมูล ทำให้ FAIL!
```

ถ้าไม่มีกลไก isolation ผลลัพธ์ของ `test_b` จะ**ขึ้นอยู่กับว่า `test_a` รันมาก่อนหรือไม่**
— test ที่ผลลัพธ์เปลี่ยนไปตามลำดับการรัน (**order-dependent test**) ถือเป็นหนึ่งใน
ปัญหาที่ร้ายแรงที่สุดของชุด test เพราะทำให้ผลลัพธ์**ไม่น่าเชื่อถือ** (flaky)

### 586.2 วิธีที่ `django.test.TestCase` แก้ปัญหานี้: Transaction + Rollback

`django.test.TestCase` แก้ปัญหานี้ด้วยกลไกที่ชาญฉลาดมาก:

```
1. ก่อนเริ่ม test method แรกในคลาส:
   → เปิด database transaction ระดับคลาส (class-level atomic block)

2. ก่อนเริ่ม test method แต่ละตัว:
   → สร้าง "savepoint" (จุดบันทึกภายใน transaction เดียวกัน)

3. test method ทำงาน สร้าง/แก้ไข/ลบข้อมูลตามปกติ

4. เมื่อ test method จบ (ไม่ว่าจะผ่านหรือ FAIL):
   → "rollback ไปที่ savepoint" ที่สร้างไว้ในข้อ 2
   → ข้อมูลทั้งหมดที่ test นั้นสร้าง/แก้ไขจะถูกลบล้างกลับสู่สภาพก่อนหน้าทันที

5. เมื่อ test method ทุกตัวในคลาสรันครบแล้ว:
   → rollback transaction ระดับคลาสทั้งหมด (ข้อมูลกลับสู่สภาพก่อนเริ่มคลาสนี้)
```

ผลลัพธ์คือ **ทุก test method เริ่มต้นด้วยฐานข้อมูลที่สะอาดเหมือนกันเป๊ะทุกครั้ง**
ไม่ว่าจะรันตามลำดับไหนก็ตาม (ทดสอบได้ด้วย `--reverse` จากขั้นตอนที่ 584.4):

```python
class GoodIsolationExample(TestCase):
    def test_a_creates_post(self):
        Post.objects.create(title="โพสต์ A", content="X")
        self.assertEqual(Post.objects.count(), 1)   # ผ่านเสมอ

    def test_b_creates_another_post(self):
        Post.objects.create(title="โพสต์ B", content="Y")
        self.assertEqual(Post.objects.count(), 1)   # ผ่านเสมอเช่นกัน! เพราะ test_a
                                                       # ถูก rollback ไปแล้วก่อน test_b เริ่ม
```

### 586.3 ทำไมใช้ Transaction Rollback แทนการลบข้อมูลด้วยมือ

| วิธี | ความเร็ว | ความน่าเชื่อถือ |
|---|---|---|
| Rollback transaction (สิ่งที่ `TestCase` ทำ) | **เร็วมาก** (ไม่มีการเขียนดิสก์จริงถาวร ฐานข้อมูลจำแค่ "ยกเลิกสิ่งที่ทำไป") | สมบูรณ์แบบ 100% — คืนสภาพทุกตารางแม่นยำ ไม่มีข้อมูลตกค้าง |
| ลบข้อมูลด้วยมือใน `tearDown()` | ช้ากว่า และเสี่ยงลืมลบบาง table | เสี่ยงพลาด ถ้าลืมลบ table ใด table หนึ่งจะมีข้อมูลตกค้างกระทบ test อื่น |
| TRUNCATE ทุกตาราง (สิ่งที่ `TransactionTestCase` ทำ) | ช้าที่สุด (ต้องรีเซ็ต auto-increment sequence ด้วย) | สมบูรณ์เช่นกัน แต่ช้ากว่า rollback มาก |

นี่คือเหตุผลที่คำแนะนำมาตรฐานคือ **ใช้ `TestCase` เป็นค่าเริ่มต้นเสมอ** และใช้
`TransactionTestCase` เฉพาะเมื่อมีเหตุผลจำเพาะเจาะจงตามขั้นตอนที่ 585.5 เท่านั้น

### 586.4 `setUpTestData()`: ข้อมูลที่สร้างครั้งเดียวต่อคลาส ไม่ใช่ต่อ test

Django มี classmethod พิเศษชื่อ `setUpTestData()` ที่**มีเฉพาะใน `TestCase`**
(ไม่มีใน `unittest.TestCase` ธรรมดา) ออกแบบมาเพื่อเพิ่มประสิทธิภาพเมื่อหลาย test
ในคลาสเดียวกันต้องการข้อมูลตั้งต้นชุดเดียวกันที่**ไม่ถูกแก้ไข**:

```python
from django.test import TestCase
from blog.models import Post


class PostQueryTest(TestCase):
    @classmethod
    def setUpTestData(cls):
        """
        ทำงานเพียง 'ครั้งเดียว' สำหรับทั้งคลาส (ไม่ใช่ทุก test แบบ setUp())
        ข้อมูลที่สร้างที่นี่ถูกครอบด้วย transaction ระดับคลาส แล้วแต่ละ test method
        จะได้ savepoint ของตัวเอง — ถ้า test ไหนแก้ไขข้อมูลนี้ การแก้ไขจะถูก
        rollback หลัง test นั้นจบ ทำให้ test ถัดไปยังเห็นข้อมูลต้นฉบับเหมือนเดิม
        """
        cls.published_post = Post.objects.create(
            title="เผยแพร่แล้ว", content="X", is_published=True
        )
        cls.draft_post = Post.objects.create(
            title="ฉบับร่าง", content="Y", is_published=False
        )

    def test_published_posts_count(self):
        self.assertEqual(Post.objects.filter(is_published=True).count(), 1)

    def test_can_mutate_without_affecting_other_tests(self):
        # แก้ title ของ object ที่มาจาก setUpTestData — การแก้นี้จะถูก rollback
        # อัตโนมัติหลัง test นี้จบ ไม่กระทบ test อื่นในคลาสเดียวกัน
        self.published_post.title = "ชื่อใหม่ชั่วคราว"
        self.published_post.save()
        self.assertEqual(self.published_post.title, "ชื่อใหม่ชั่วคราว")

    def test_original_title_unaffected_by_previous_test(self):
        # แม้ test ก่อนหน้าจะแก้ title ไปแล้ว แต่ object สดใหม่จาก DB ยังเป็นค่าเดิม
        fresh = Post.objects.get(pk=self.published_post.pk)
        self.assertEqual(fresh.title, "เผยแพร่แล้ว")
```

**เปรียบเทียบ `setUp()` กับ `setUpTestData()`**:

| | `setUp()` | `setUpTestData()` |
|---|---|---|
| เรียกกี่ครั้ง | ทุก test method (instance method) | ครั้งเดียวต่อคลาส (classmethod) |
| ความเร็วเมื่อมี test เยอะ | ช้ากว่า (สร้างข้อมูลซ้ำทุกครั้ง) | **เร็วกว่ามาก** (สร้างครั้งเดียว ใช้ savepoint แยกแค่ต่อ test) |
| มีใน `TestCase` เท่านั้นหรือ | มีในทุก TestCase (มาจาก unittest) | **มีเฉพาะ `django.test.TestCase`** |
| เหมาะกับ | ข้อมูลที่แต่ละ test ต้องการค่าต่างกัน หรือ logic setup ซับซ้อน | ข้อมูลตั้งต้นที่เหมือนกันทุก test ในคลาส (ทั่วไปคือ 80% ของกรณีใช้งาน) |

**คำแนะนำของหลักสูตรนี้**: ใช้ `setUpTestData()` เป็นค่าเริ่มต้นเมื่อข้อมูลตั้งต้น
เหมือนกันทุก test ในคลาส สลับไปใช้ `setUp()` เฉพาะเมื่อแต่ละ test ต้องการข้อมูล
ที่แตกต่างกันจริง ๆ หรือ logic การสร้างข้อมูลซับซ้อนเกินกว่าจะใช้ร่วมกันได้

---

## ขั้นตอนที่ 587: การจัดระเบียบไฟล์ Test — `tests.py` เดี่ยว vs แพ็กเกจ `tests/` แยกตามหมวด

### 587.1 จุดเริ่มต้น: `tests.py` ไฟล์เดียว

เมื่อรัน `python manage.py startapp blog` (ตั้งแต่ Part 004) Django สร้างไฟล์
`blog/tests.py` เปล่า ๆ ให้อัตโนมัติ:

```python
# blog/tests.py (ไฟล์เริ่มต้นที่ Django สร้างให้)
from django.test import TestCase

# Create your tests here.
```

สำหรับแอปขนาดเล็กที่มี model 1-2 ตัว การเขียน test ทั้งหมดในไฟล์เดียวนี้ก็เพียงพอ
และเป็นวิธีที่หลักสูตรนี้ใช้มาตลอด Part 001-058 (แม้ยังไม่ได้เขียน test จริงจัง)

### 587.2 เมื่อไหร่ที่ `tests.py` เดี่ยวเริ่มมีปัญหา

เมื่อแอป `blog` ของเราตอนนี้มี `Post`, `Category` (จาก Part 012), Form (Part 025-026),
View (Part 007, 022-024), API Serializer (Part 040-041) และจะมี test ของทั้งหมดนี้
เพิ่มเข้ามาใน Phase 7 ไฟล์ `tests.py` เดี่ยวจะกลายเป็นไฟล์**ยาวหลายพันบรรทัด**
ซึ่งมีปัญหาชัดเจน:

- หา test ที่เกี่ยวกับ model เจอยาก ต้อง scroll ผ่าน test ของ view/form ก่อน
- Merge conflict บ่อยเวลาทำงานเป็นทีม (หลายคนแก้ไฟล์เดียวกันพร้อมกัน)
- รัน test เฉพาะหมวดหมู่ทำได้ยาก (ต้องรู้ชื่อคลาสแม่นยำ)

### 587.3 วิธีแก้: แปลง `tests.py` เป็นแพ็กเกจ `tests/`

```
blog/
├── __init__.py
├── models.py
├── views.py
├── forms.py
├── tests/                      ← เปลี่ยนจากไฟล์ tests.py เป็นโฟลเดอร์
│   ├── __init__.py             ← ทำให้โฟลเดอร์นี้เป็น Python package (จำเป็น!)
│   ├── test_models.py          ← test ของ Post, Category model
│   ├── test_views.py           ← test ของ FBV/CBV ทั้งหมด (Part 060)
│   ├── test_forms.py           ← test ของ ModelForm/Form (Part 060)
│   └── test_serializers.py     ← test ของ DRF serializer (Part 061-062)
```

**ขั้นตอนแปลงจริง**:

```bash
# 1. ลบไฟล์ tests.py เดิม (backup เนื้อหาไว้ก่อนถ้ามี)
rm blog/tests.py

# 2. สร้างโฟลเดอร์ tests/ แทน
mkdir blog/tests

# 3. สร้างไฟล์ __init__.py เปล่า ๆ เพื่อให้เป็น package
touch blog/tests/__init__.py

# 4. สร้างไฟล์ test แยกตามหมวด
touch blog/tests/test_models.py
touch blog/tests/test_views.py
```

**สำคัญมาก**: ไฟล์ในโฟลเดอร์ `tests/` ต้องขึ้นต้นด้วย `test_` (ตรงกับ pattern
`test*.py` ที่ test discovery มองหา ตามขั้นตอนที่ 584.2) ถ้าตั้งชื่อผิด เช่น
`models_test.py` (สลับคำ) Django จะ**หาไม่เจอและไม่รันให้เงียบ ๆ** โดยไม่มี error
เตือนเลย — นี่คือกับดักที่พบบ่อยมากตอนย้ายจากไฟล์เดียวมาเป็นแพ็กเกจ

### 587.4 ตัวอย่างเนื้อหาไฟล์ที่แยกแล้ว

```python
# blog/tests/test_models.py
from django.test import TestCase
from blog.models import Post


class PostModelTest(TestCase):
    def test_str_returns_title(self):
        post = Post.objects.create(title="ทดสอบ", content="เนื้อหา")
        self.assertEqual(str(post), "ทดสอบ")
```

```python
# blog/tests/test_views.py  (ตัวอย่างล่วงหน้า — เนื้อหาเต็มอยู่ใน Part 060)
from django.test import TestCase
from django.urls import reverse
from blog.models import Post


class PostListViewTest(TestCase):
    def test_list_view_returns_200(self):
        response = self.client.get(reverse('blog:list'))
        self.assertEqual(response.status_code, 200)
```

รันแยกเฉพาะหมวดได้ทันทีด้วย import path ปกติ:

```bash
python manage.py test blog.tests.test_models
python manage.py test blog.tests.test_views
python manage.py test blog.tests.test_models.PostModelTest.test_str_returns_title
```

### 587.5 อนุสัญญาการตั้งชื่อ (Naming Convention) มาตรฐานของหลักสูตรนี้

| ประเภทของสิ่งที่ทดสอบ | ชื่อไฟล์ | ชื่อคลาส |
|---|---|---|
| Model | `test_models.py` | `<ModelName>ModelTest` เช่น `PostModelTest` |
| View | `test_views.py` | `<ViewName>ViewTest` เช่น `PostListViewTest`, `PostDetailViewTest` |
| Form | `test_forms.py` | `<FormName>FormTest` เช่น `PostFormTest` |
| Serializer (DRF) | `test_serializers.py` | `<SerializerName>SerializerTest` |
| Utility function | `test_utils.py` | `<FunctionName>Test` หรือจัดกลุ่มตามไฟล์ต้นฉบับ |
| Management command | `test_commands.py` | `<CommandName>CommandTest` |

การตั้งชื่อที่สม่ำเสมอแบบนี้ทำให้ทุกคนในทีมเดาได้ทันทีว่า test ของสิ่งที่กำลังหาอยู่
ไฟล์ไหน โดยไม่ต้องเปิดค้นหา — เป็นรายละเอียดเล็ก ๆ ที่ส่งผลใหญ่ต่อความเร็วในการทำงาน
ร่วมกันของทีม

---

## ขั้นตอนที่ 588: `setUpClass()`/`tearDownClass()` สำหรับ setup ที่ใช้ทรัพยากรหนัก ทำครั้งเดียวต่อคลาส

### 588.1 ความแตกต่างจาก `setUpTestData()`

ขั้นตอนที่ 586.4 แนะนำ `setUpTestData()` ซึ่งเป็นวิธี "มาตรฐาน" สำหรับสร้าง**ข้อมูลใน
ฐานข้อมูล** เพียงครั้งเดียวต่อคลาส แต่ `setUpClass()`/`tearDownClass()` เป็น method
ระดับที่**ต่ำกว่าและกว้างกว่า** — มาจาก `unittest` ตรง ๆ (ไม่ใช่ของ Django) และใช้ได้
กับ **ทุก TestCase ชนิด** รวมถึง `SimpleTestCase` ที่ไม่มีฐานข้อมูลด้วยซ้ำ

| | `setUpTestData()` | `setUpClass()` |
|---|---|---|
| มาจาก | Django (`django.test.TestCase` เท่านั้น) | Python `unittest` (ทุก TestCase) |
| เหมาะกับ | สร้าง object ในฐานข้อมูล | ทรัพยากรอะไรก็ได้ที่ **ไม่ใช่ฐานข้อมูล** เช่น โหลดไฟล์ขนาดใหญ่, เริ่ม mock server, compile regex ที่ซับซ้อน, เชื่อมต่อ cache client |
| ต้องเรียก `super()` เองไหม | ไม่ต้อง | **ต้องเรียกเสมอ** (`super().setUpClass()`) ไม่เช่นนั้นกลไก transaction ของ Django จะพัง |

### 588.2 ตัวอย่างการใช้งานจริง: โหลดไฟล์ตัวอย่างขนาดใหญ่ครั้งเดียว

```python
# blog/tests/test_import.py
import json
from pathlib import Path
from django.test import TestCase

FIXTURE_DIR = Path(__file__).resolve().parent / 'fixtures'


class BulkPostImportTest(TestCase):
    @classmethod
    def setUpClass(cls):
        super().setUpClass()   # ต้องเรียกก่อนเสมอ! ไม่เช่นนั้น TestCase จะทำงานผิดพลาด
        # โหลดไฟล์ JSON ขนาดใหญ่ (สมมติมี 5,000 รายการ) เพียงครั้งเดียว
        # แทนที่จะโหลดซ้ำทุก test method ซึ่งจะช้ามาก
        with open(FIXTURE_DIR / 'sample_posts_bulk.json', encoding='utf-8') as f:
            cls.raw_post_data = json.load(f)

    @classmethod
    def tearDownClass(cls):
        # คืนหน่วยความจำที่ใช้เก็บข้อมูลขนาดใหญ่ ก่อนเรียก super() เสมอ
        del cls.raw_post_data
        super().tearDownClass()

    def test_fixture_has_expected_count(self):
        self.assertEqual(len(self.raw_post_data), 5000)

    def test_first_item_has_required_keys(self):
        first_item = self.raw_post_data[0]
        self.assertIn('title', first_item)
        self.assertIn('content', first_item)
```

**ลำดับการเรียก `super()` ที่ต้องจำให้แม่น**:

- ใน `setUpClass()`: เรียก `super().setUpClass()` **ก่อน** ทำ logic ของตัวเอง
- ใน `tearDownClass()`: เรียก `super().tearDownClass()` **หลัง** ทำ logic ของตัวเอง
  (ทำความสะอาดของตัวเองก่อน แล้วค่อยปล่อยให้ parent class ทำความสะอาดของมัน)

ถ้าลืมเรียก `super()` ใน `django.test.TestCase` transaction ระดับคลาสที่ Django
ใช้ห่อทุก test เอาไว้จะไม่ถูกเปิด/ปิดอย่างถูกต้อง อาจทำให้ test ถัดไปในชุดทดสอบ
ทั้งหมดพังแบบหาสาเหตุยากมาก

### 588.3 ตัวอย่างการใช้ร่วมกับ `unittest.mock.patch` ระดับคลาส

```python
from unittest.mock import patch
from django.test import TestCase
from blog.models import Post


class PostExternalServiceTest(TestCase):
    @classmethod
    def setUpClass(cls):
        super().setUpClass()
        # เปิด patch ไว้ตลอดทั้งคลาส แทนที่จะเปิด-ปิดทุก test method
        cls.notify_patcher = patch('blog.services.notify_subscribers')
        cls.mock_notify = cls.notify_patcher.start()

    @classmethod
    def tearDownClass(cls):
        cls.notify_patcher.stop()   # ต้องหยุด patch เองก่อนเรียก super()
        super().tearDownClass()

    def test_publish_does_not_actually_call_external_service(self):
        post = Post.objects.create(title="A", content="B")
        post.publish()
        self.mock_notify.assert_not_called()   # ยังไม่เรียกเพราะยังไม่ได้ผูก logic นี้จริง
```

> **หมายเหตุ**: การทดสอบด้วย `unittest.mock` แบบเจาะลึก (mock ฟังก์ชัน, mock API
> ภายนอก, `MagicMock`, `patch.object`) เป็นเนื้อหาหลักของ **Part 062 (Test Coverage
> และ Mocking)** ตัวอย่างข้างต้นเป็นเพียงการแสดงให้เห็นว่า `setUpClass()`
> เหมาะกับการเปิด/ปิด patch ระดับคลาสได้เช่นกัน

### 588.4 เมื่อไหร่ควรใช้ `setUpClass()` แทน `setUpTestData()`

ใช้ **`setUpTestData()`** เมื่อ:
- สร้าง object ของ Django model ที่ต้องบันทึกลงฐานข้อมูล
- อยู่ใน `django.test.TestCase` เท่านั้น (ไม่ใช่ `SimpleTestCase`)

ใช้ **`setUpClass()`** เมื่อ:
- ทรัพยากรที่ต้องเตรียมไม่ใช่ฐานข้อมูล (ไฟล์, mock, การเชื่อมต่อ, การคำนวณหนัก)
- ต้องการให้ใช้ได้กับทุก TestCase ชนิด รวมถึง `SimpleTestCase`
- ทำงานร่วมกับ library ภายนอกที่ไม่รู้จัก Django (เช่น เปิด mock HTTP server ด้วย
  library `responses` หรือ `httpretty`)

ทั้งสองใช้**ร่วมกันในคลาสเดียวได้**ตามปกติ ไม่ขัดแย้งกัน

---

## ขั้นตอนที่ 589: การข้าม Test — `@skip`, `@skipIf`, `@skipUnless`

### 589.1 ทำไมบางครั้งต้อง "ข้าม" Test แทนที่จะลบทิ้ง

มีสถานการณ์จริงที่ test ไม่ควรถูกลบทิ้ง แต่ก็ไม่ควรถูกรันในบางสภาพแวดล้อม เช่น:

- Test ที่ต้องพึ่งพาฐานข้อมูลชนิดเฉพาะ (เช่น PostgreSQL full-text search) แต่ทีม
  บางคนพัฒนาบนเครื่องที่ใช้ SQLite
- Test ของฟีเจอร์ที่ยังเขียนไม่เสร็จ (placeholder ไว้ก่อน)
- Test ที่ต้องพึ่งพา library ภายนอกที่ติดตั้งเฉพาะบางสภาพแวดล้อม
- Test ที่รู้อยู่แล้วว่าจะ FAIL เพราะ bug ที่ยังไม่ได้แก้ (documented known issue)

Python `unittest` มี decorator 3 ตัวสำหรับจัดการสถานการณ์เหล่านี้

### 589.2 `@skip`: ข้ามแบบไม่มีเงื่อนไข

```python
from unittest import skip
from django.test import TestCase
from blog.models import Post


class PostFeatureTest(TestCase):
    @skip("รอ Part 061 ที่จะเชื่อม pytest-django เข้ามาแทน unittest runner")
    def test_not_yet_ready_feature(self):
        # เขียนไว้ล่วงหน้า แต่ยังไม่พร้อมรัน
        self.fail("ยังไม่ได้ implement")
```

ผลลัพธ์ตอนรัน:

```
test_not_yet_ready_feature (blog.tests.PostFeatureTest) ... skipped
'รอ Part 061 ที่จะเชื่อม pytest-django เข้ามาแทน unittest runner'
```

**ข้อสำคัญ**: `@skip` ต้อง**ระบุเหตุผลเสมอ** (เป็น string บังคับ) เพื่อให้คนอื่นที่มา
อ่าน test log เข้าใจทันทีว่าทำไม test นี้ถึงไม่ถูกรัน ไม่ใช่ปล่อยให้เดา

### 589.3 `@skipIf` / `@skipUnless`: ข้ามแบบมีเงื่อนไข

```python
import django
from unittest import skipIf, skipUnless
from django.conf import settings
from django.db import connection
from django.test import TestCase
from blog.models import Post


class PostDatabaseSpecificTest(TestCase):
    @skipUnless(connection.vendor == 'postgresql', "ต้องใช้ PostgreSQL เท่านั้น")
    def test_full_text_search_only_works_on_postgresql(self):
        # สมมติใช้ฟีเจอร์ full-text search ของ PostgreSQL โดยเฉพาะ (Part 070)
        Post.objects.create(title="Django คือดี", content="เนื้อหา")
        results = Post.objects.filter(title__search="Django")
        self.assertEqual(results.count(), 1)

    @skipIf(connection.vendor == 'sqlite', "SQLite ไม่รองรับ concurrent write ที่ทดสอบตรงนี้")
    def test_concurrent_write_behavior(self):
        pass

    @skipUnless(django.VERSION >= (5, 0), "ต้องใช้ Django 5.0 ขึ้นไป")
    def test_feature_available_since_django_5(self):
        pass

    @skipIf(not getattr(settings, 'ENABLE_PREMIUM_FEATURES', False), "ฟีเจอร์พรีเมียมปิดอยู่")
    def test_premium_pricing_logic(self):
        post = Post.objects.create(title="A", content="B", is_premium=True, price=99)
        self.assertFalse(post.is_free)
```

**ความต่างระหว่างสองตัว**:

| Decorator | ข้าม test เมื่อ |
|---|---|
| `@skipIf(condition, reason)` | `condition` เป็น **True** |
| `@skipUnless(condition, reason)` | `condition` เป็น **False** (คือรันเฉพาะเมื่อเงื่อนไขเป็นจริง) |

จำง่าย ๆ: **"skip if bad"** และ **"skip unless good"**

### 589.4 `self.skipTest()`: ข้ามจากภายใน test method โดยตรง

บางครั้งเงื่อนไขในการข้ามซับซ้อนเกินกว่าจะเขียนเป็น decorator บรรทัดเดียว
เรียก `self.skipTest()` จากภายในตัว method ได้เลย:

```python
class PostConditionalTest(TestCase):
    def test_image_processing_requires_pillow(self):
        try:
            import PIL  # noqa: F401
        except ImportError:
            self.skipTest("ต้องติดตั้ง Pillow ก่อนถึงจะรัน test นี้ได้")

        # โค้ดที่เหลือรันเฉพาะเมื่อมี Pillow ติดตั้งอยู่จริง
        post = Post.objects.create(title="A", content="B")
        self.assertTrue(hasattr(post, 'cover_image'))
```

### 589.5 `@expectedFailure`: บันทึกว่า test นี้รู้อยู่แล้วว่าจะ FAIL

```python
from unittest import expectedFailure
from django.test import TestCase


class KnownBugTest(TestCase):
    @expectedFailure
    def test_reading_time_calculation_has_known_rounding_bug(self):
        # Bug tracked in: internal-issue-tracker#4821
        post = Post.objects.create(title="A", content="X" * 1000, reading_time_minutes=5)
        # สมมติมี bug ที่คำนวณผิดไป 1 นาทีเสมอ (ตัวอย่างสาธิต)
        self.assertEqual(post.reading_time_label, "อ่านประมาณ 4 นาที")  # ค่าจริงคือ 5
```

ถ้า test ที่แปะ `@expectedFailure` **FAIL จริงตามคาด** ผลลัพธ์รวมจะรายงานเป็น
`expected failures=1` (ไม่นับเป็น FAIL ของทั้งชุด) แต่ถ้า test นี้กลับ**ผ่าน**โดยไม่คาดคิด
(เช่น มีคนแก้ bug ไปแล้วแต่ลืมลบ decorator) จะรายงานเป็น **`unexpected successes=1`**
ซึ่งเป็นสัญญาณเตือนให้กลับมาลบ `@expectedFailure` ออกและปรับ assertion ให้ถูกต้อง

### 589.6 ตารางสรุปทุก Decorator ในขั้นตอนนี้

| Decorator | Import จาก | ใช้เมื่อ |
|---|---|---|
| `@skip(reason)` | `unittest` | ข้ามไม่มีเงื่อนไข ต้องระบุเหตุผลเสมอ |
| `@skipIf(condition, reason)` | `unittest` | ข้ามเมื่อเงื่อนไขเป็นจริง |
| `@skipUnless(condition, reason)` | `unittest` | รันเฉพาะเมื่อเงื่อนไขเป็นจริง |
| `self.skipTest(reason)` | เรียกผ่าน `self` ใน test method | เงื่อนไขซับซ้อน ต้องคำนวณระหว่าง test ทำงาน |
| `@expectedFailure` | `unittest` | บันทึก known bug ที่ยังไม่ได้แก้ไว้อย่างมีสติ |
| `@tag(*tags)` | `django.test` | ไม่ใช่การข้าม แต่ใช้กรองรันเฉพาะกลุ่ม (ขั้นตอนที่ 584.4) |

---

## ขั้นตอนที่ 590: สรุปและแบบฝึกหัด

### 590.1 สรุปสิ่งที่ได้เรียนรู้ใน Part นี้

- ✅ เข้าใจว่า `unittest` เป็นรากฐานของ Python เอง และ `django.test.TestCase`
  ต่อยอดมาโดยเพิ่มความสามารถเรื่องฐานข้อมูล
- ✅ ใช้ `setUp()`/`tearDown()` เตรียมและทำความสะอาดข้อมูลก่อน-หลัง test แต่ละตัว
- ✅ ใช้ assert method หลักได้ครบ: `assertEqual`, `assertTrue`/`assertFalse`,
  `assertIsNone`/`assertIsNotNone`, `assertRaises`, และรู้จักตัวเสริมอีกหลายตัว
- ✅ เข้าใจว่า Django สร้างและทำลาย Test Database แยกจากฐานข้อมูลจริงเสมอทุกครั้งที่
  รัน `python manage.py test`
- ✅ รัน test ได้หลายระดับ: ทั้งโปรเจกต์, เฉพาะแอป, เฉพาะคลาส, เฉพาะ method พร้อม
  flag ที่จำเป็น (`--keepdb`, `--failfast`, `--parallel`, `-k`)
- ✅ แยกแยะ `TestCase`, `SimpleTestCase`, `TransactionTestCase` ได้ว่าใช้เมื่อไหร่
- ✅ เข้าใจกลไก Transaction Rollback ที่ทำให้แต่ละ test เป็นอิสระจากกัน (Test Isolation)
  และรู้จัก `setUpTestData()` เพื่อเพิ่มประสิทธิภาพ
- ✅ จัดระเบียบไฟล์ test จาก `tests.py` เดี่ยวเป็นแพ็กเกจ `tests/` แยกตามหมวดหมู่
- ✅ ใช้ `setUpClass()`/`tearDownClass()` สำหรับทรัพยากรหนักที่ควรเตรียมครั้งเดียว
- ✅ ข้าม test ได้อย่างเหมาะสมด้วย `@skip`, `@skipIf`, `@skipUnless`, `@expectedFailure`
- ✅ เขียน test suite แรกที่ครอบคลุม field validation, `__str__`, และ custom method
  ทั้งหมดของ `Post` model

### 590.2 Test Suite เต็มรูปแบบสำหรับ `Post` Model

ก่อนเขียน test ทบทวน `Post` model เวอร์ชันสมบูรณ์จาก Part 011 อีกครั้ง (โมเดลนี้คือ
สิ่งที่เราจะทดสอบทั้งหมดในหัวข้อนี้):

```python
# blog/models.py
from django.db import models
from django.urls import reverse
from django.utils.text import slugify


class Post(models.Model):
    class ContentType(models.TextChoices):
        ARTICLE = 'ARTICLE', 'บทความ'
        TUTORIAL = 'TUTORIAL', 'สอนการใช้งาน'
        NEWS = 'NEWS', 'ข่าวสาร'
        ANNOUNCEMENT = 'ANNOUNCEMENT', 'ประกาศ'

    title = models.CharField(max_length=200, verbose_name='หัวข้อบทความ')
    slug = models.SlugField(max_length=220, unique=True, blank=True)
    content = models.TextField(verbose_name='เนื้อหา')
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)
    is_published = models.BooleanField(default=False, verbose_name='เผยแพร่แล้ว')

    excerpt = models.CharField(max_length=300, blank=True, verbose_name='สรุปย่อ')
    content_type = models.CharField(
        max_length=20, choices=ContentType.choices, default=ContentType.ARTICLE,
    )
    is_premium = models.BooleanField(default=False)
    price = models.DecimalField(max_digits=6, decimal_places=2, default=0)
    view_count = models.PositiveIntegerField(default=0, editable=False)
    reading_time_minutes = models.PositiveSmallIntegerField(default=1)

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

    def get_absolute_url(self):
        return reverse('blog:detail', kwargs={'slug': self.slug})

    def increment_view_count(self):
        self.view_count += 1
        self.save(update_fields=['view_count'])

    @property
    def is_free(self):
        return not self.is_premium or self.price == 0

    @property
    def reading_time_label(self):
        if self.reading_time_minutes <= 1:
            return 'อ่านไม่ถึง 1 นาที'
        return f'อ่านประมาณ {self.reading_time_minutes} นาที'

    def publish(self):
        self.is_published = True
        self.save(update_fields=['is_published', 'updated_at'])
```

```python
# blog/urls.py
from django.urls import path
from . import views

app_name = 'blog'

urlpatterns = [
    path('', views.post_list, name='list'),
    path('<slug:slug>/', views.post_detail, name='detail'),
]
```

```python
# config/urls.py (ทบทวนจาก Part 006: blog.urls ถูก include ไว้ใต้ prefix 'posts/')
from django.contrib import admin
from django.urls import path, include

urlpatterns = [
    path('admin/', admin.site.urls),
    path('posts/', include('blog.urls')),
]
```

ตอนนี้เขียน test suite เต็มรูปแบบไว้ที่ `blog/tests/test_models.py` (ตามโครงสร้าง
ไฟล์แบบแพ็กเกจจากขั้นตอนที่ 587):

```python
# blog/tests/test_models.py
from decimal import Decimal

from django.core.exceptions import ValidationError
from django.db.utils import IntegrityError
from django.test import TestCase
from django.utils.text import slugify

from blog.models import Post


class PostFieldValidationTest(TestCase):
    """ทดสอบเรื่อง field, การ validate ข้อมูล, และ constraint ระดับฐานข้อมูล"""

    def test_title_is_required(self):
        post = Post(title="", content="เนื้อหาบางส่วน")
        with self.assertRaises(ValidationError):
            post.full_clean()

    def test_content_is_required(self):
        post = Post(title="หัวข้อ", content="")
        with self.assertRaises(ValidationError):
            post.full_clean()

    def test_excerpt_can_be_blank_at_form_level(self):
        # excerpt มี blank=True จึงต้องผ่าน full_clean() แม้ไม่กรอก
        # (save() จะเติมให้อัตโนมัติภายหลัง แต่ full_clean() ไม่เรียก save())
        post = Post(title="หัวข้อ", content="เนื้อหา" * 10)
        try:
            post.full_clean()
        except ValidationError as exc:
            self.fail(f"full_clean() ไม่ควร raise error สำหรับ excerpt ที่ว่างเปล่า: {exc}")

    def test_slug_must_be_unique(self):
        Post.objects.create(title="หัวข้อแรก", slug="same-slug", content="X")
        with self.assertRaises(IntegrityError):
            Post.objects.create(title="หัวข้อสอง", slug="same-slug", content="Y")

    def test_default_field_values(self):
        post = Post.objects.create(title="ค่าเริ่มต้น", content="X")
        self.assertFalse(post.is_published)
        self.assertFalse(post.is_premium)
        self.assertEqual(post.price, Decimal('0'))
        self.assertEqual(post.view_count, 0)
        self.assertEqual(post.reading_time_minutes, 1)
        self.assertEqual(post.content_type, Post.ContentType.ARTICLE)

    def test_price_field_returns_decimal_type(self):
        post = Post.objects.create(
            title="พรีเมียม", content="X", is_premium=True, price=Decimal('149.50')
        )
        self.assertIsInstance(post.price, Decimal)
        self.assertEqual(post.price, Decimal('149.50'))

    def test_view_count_cannot_be_negative_at_database_level(self):
        # PositiveIntegerField validate ผ่าน full_clean() ไม่ใช่ผ่าน save() ตรง ๆ
        post = Post(title="A", content="B", view_count=-1)
        with self.assertRaises(ValidationError):
            post.full_clean()


class PostSaveBehaviorTest(TestCase):
    """ทดสอบ logic พิเศษที่เขียนไว้ใน save() ที่ override เอง"""

    def test_slug_is_auto_generated_from_title_when_blank(self):
        post = Post.objects.create(title="เรียนรู้ Django Testing วันนี้", content="X")
        self.assertEqual(post.slug, slugify("เรียนรู้ Django Testing วันนี้"))

    def test_explicit_slug_is_not_overwritten(self):
        post = Post.objects.create(title="หัวข้อใด ๆ", slug="my-custom-slug", content="X")
        self.assertEqual(post.slug, "my-custom-slug")

    def test_excerpt_is_generated_from_content_when_blank(self):
        long_content = "คำ " * 200   # เนื้อหายาวเกิน 300 ตัวอักษรแน่นอน
        post = Post.objects.create(title="A", content=long_content)
        self.assertTrue(post.excerpt)
        self.assertLessEqual(len(post.excerpt), 300)
        self.assertTrue(post.excerpt.endswith('...'))

    def test_explicit_excerpt_is_not_overwritten(self):
        post = Post.objects.create(
            title="A", content="เนื้อหายาว" * 50, excerpt="สรุปย่อที่เขียนเอง"
        )
        self.assertEqual(post.excerpt, "สรุปย่อที่เขียนเอง")


class PostStringRepresentationTest(TestCase):
    """ทดสอบ __str__ ตามกฎเหล็กของหลักสูตรที่ทุก model ต้องมี"""

    def test_str_returns_title(self):
        post = Post.objects.create(title="Django คือดีที่สุด", content="X")
        self.assertEqual(str(post), "Django คือดีที่สุด")

    def test_str_used_in_f_string(self):
        post = Post.objects.create(title="ทดสอบ f-string", content="X")
        self.assertEqual(f"บทความ: {post}", "บทความ: ทดสอบ f-string")


class PostCustomMethodTest(TestCase):
    """ทดสอบ custom business-logic method ทั้งหมดจากขั้นตอนที่ 106 ของ Part 011"""

    @classmethod
    def setUpTestData(cls):
        cls.free_post = Post.objects.create(
            title="บทความฟรี", content="X", is_premium=False, price=Decimal('0'),
        )
        cls.premium_post = Post.objects.create(
            title="บทความพรีเมียม", content="Y",
            is_premium=True, price=Decimal('99.00'),
        )

    def test_get_absolute_url_returns_correct_path(self):
        # blog.urls ถูก include ไว้ใต้ prefix 'posts/' ใน config/urls.py (ทบทวนจาก Part 006)
        expected = f"/posts/{self.free_post.slug}/"
        self.assertEqual(self.free_post.get_absolute_url(), expected)

    def test_increment_view_count_adds_one(self):
        self.assertEqual(self.free_post.view_count, 0)
        self.free_post.increment_view_count()
        self.assertEqual(self.free_post.view_count, 1)

    def test_increment_view_count_persists_to_database(self):
        self.free_post.increment_view_count()
        reloaded = Post.objects.get(pk=self.free_post.pk)
        self.assertEqual(reloaded.view_count, 1)

    def test_increment_view_count_called_multiple_times(self):
        for _ in range(5):
            self.free_post.increment_view_count()
        self.assertEqual(self.free_post.view_count, 5)

    def test_is_free_true_for_non_premium_post(self):
        self.assertTrue(self.free_post.is_free)

    def test_is_free_false_for_premium_post_with_price(self):
        self.assertFalse(self.premium_post.is_free)

    def test_is_free_true_for_premium_post_with_zero_price(self):
        zero_price_premium = Post.objects.create(
            title="พรีเมียมแต่ฟรี", content="Z", is_premium=True, price=Decimal('0'),
        )
        self.assertTrue(zero_price_premium.is_free)

    def test_reading_time_label_for_one_minute_or_less(self):
        post = Post.objects.create(title="A", content="B", reading_time_minutes=1)
        self.assertEqual(post.reading_time_label, "อ่านไม่ถึง 1 นาที")

    def test_reading_time_label_for_multiple_minutes(self):
        post = Post.objects.create(title="A", content="B", reading_time_minutes=7)
        self.assertEqual(post.reading_time_label, "อ่านประมาณ 7 นาที")

    def test_publish_sets_is_published_true(self):
        self.assertFalse(self.free_post.is_published)
        self.free_post.publish()
        self.assertTrue(self.free_post.is_published)

    def test_publish_persists_to_database(self):
        self.free_post.publish()
        reloaded = Post.objects.get(pk=self.free_post.pk)
        self.assertTrue(reloaded.is_published)


class PostMetaOptionsTest(TestCase):
    """ทดสอบพฤติกรรมที่มาจาก Meta class เช่น ordering"""

    def test_posts_are_ordered_by_created_at_descending(self):
        older = Post.objects.create(title="เก่ากว่า", content="X")
        newer = Post.objects.create(title="ใหม่กว่า", content="Y")
        posts = list(Post.objects.all())
        self.assertEqual(posts, [newer, older])   # newer ต้องมาก่อนตาม ordering = ['-created_at']
```

รันทดสอบไฟล์นี้:

```bash
python manage.py test blog.tests.test_models -v 2
```

ผลลัพธ์ที่ควรได้ (ย่อ):

```
Creating test database for alias 'default'...
test_content_is_required (blog.tests.test_models.PostFieldValidationTest) ... ok
test_default_field_values (blog.tests.test_models.PostFieldValidationTest) ... ok
...
test_str_returns_title (blog.tests.test_models.PostStringRepresentationTest) ... ok
...
test_publish_sets_is_published_true (blog.tests.test_models.PostCustomMethodTest) ... ok
...
----------------------------------------------------------------------
Ran 21 tests in 0.312s

OK
Destroying test database for alias 'default'...
```

Test suite นี้ครอบคลุมครบ 3 หัวข้อที่โจทย์กำหนด: **field validation** (`PostFieldValidationTest`,
`PostSaveBehaviorTest`), **`__str__`** (`PostStringRepresentationTest`), และ
**custom methods** (`PostCustomMethodTest`, `PostMetaOptionsTest`) — และใช้เทคนิค
ทั้งหมดที่เรียนมาใน Part นี้: `setUpTestData()`, assert method หลากหลายชนิด,
`assertRaises` กับทั้ง `ValidationError` และ `IntegrityError`

### 590.3 Checklist ก่อนไป Part ถัดไป

- [ ] เข้าใจความแตกต่างระหว่าง `unittest` (มาตรฐาน Python) กับ `django.test.TestCase`
      (ส่วนขยายของ Django)
- [ ] เขียน `setUp()` และอธิบายได้ว่าทำไมมันถูกเรียกก่อน**ทุก** test method ไม่ใช่
      แค่ครั้งเดียว
- [ ] ใช้ `assertEqual`, `assertTrue`/`assertFalse`, `assertIsNone`/`assertIsNotNone`,
      `assertRaises` ได้คล่องโดยไม่ต้องเปิดเอกสารดู
- [ ] อธิบายได้ว่าทำไม Django ต้องสร้างและทำลาย Test Database ทุกครั้ง แทนที่จะใช้
      ฐานข้อมูล dev ตรง ๆ
- [ ] รัน `python manage.py test` ได้ทั้งแบบทั้งโปรเจกต์, เฉพาะแอป, เฉพาะคลาส,
      เฉพาะ method
- [ ] อธิบายได้ว่าเมื่อไหร่ควรใช้ `TestCase`, `SimpleTestCase`, `TransactionTestCase`
      แต่ละแบบ
- [ ] อธิบายกลไก transaction rollback ที่ทำให้แต่ละ test isolate จากกันได้ด้วยคำพูด
      ตัวเอง
- [ ] แปลง `tests.py` เดี่ยวเป็นแพ็กเกจ `tests/` ได้ พร้อมตั้งชื่อไฟล์ถูกต้องตาม
      convention
- [ ] เขียนและรัน test suite เต็มรูปแบบสำหรับ `Post` model ได้สำเร็จ ครบทั้ง 21 test
      ผ่านหมด (สีเขียว/`OK`)

### 590.4 แบบฝึกหัดท้ายบท

**แบบฝึกหัดที่ 1**: เขียน test เพิ่มในคลาส `PostFieldValidationTest` ที่ชื่อ
`test_title_max_length_is_200_characters` เพื่อยืนยันว่าถ้าสร้าง `Post` ที่มี `title`
ยาวเกิน 200 ตัวอักษร `full_clean()` ต้อง raise `ValidationError` (คำใบ้: สร้าง string
ยาว 201 ตัวอักษรด้วย `"ก" * 201`)

**แบบฝึกหัดที่ 2**: เพิ่ม `Category` model ง่าย ๆ ตามที่ออกแบบไว้ในแบบฝึกหัดที่ 2
ของ Part 011 (มี `name`, `slug`, `__str__`, `get_absolute_url`) แล้วเขียนไฟล์
`blog/tests/test_models.py` เพิ่มคลาส `CategoryModelTest` ที่ทดสอบ `__str__`,
การสร้าง slug อัตโนมัติ, และ `unique=True` ของทั้ง `name` และ `slug`

**แบบฝึกหัดที่ 3**: เขียน test ที่พิสูจน์ **Test Isolation** ด้วยตัวเอง: สร้างคลาส
`PostIsolationProofTest(TestCase)` ที่มี 2 test methods โดยแต่ละตัวสร้าง `Post`
คนละ 3 รายการ แล้ว assert ว่า `Post.objects.count()` เท่ากับ 3 เสมอในทั้งสอง
methods จากนั้นรันด้วย `python manage.py test --reverse` เพื่อยืนยันว่าผลลัพธ์
เหมือนเดิมไม่ว่าจะรันย้อนลำดับหรือไม่ (ถ้า test ทั้งสองผ่านทั้งสองทิศทาง แปลว่า
isolation ทำงานถูกต้อง)

**แบบฝึกหัดที่ 4 (ขั้นสูง)**: เขียนคลาส `PostConditionalSkipTest(TestCase)` ที่มี
test สำหรับตรวจสอบว่า field `content_type` ใช้ค่าจาก `Post.ContentType` ได้ถูกต้อง
ทุกตัวเลือก (`ARTICLE`, `TUTORIAL`, `NEWS`, `ANNOUNCEMENT`) โดยใช้ **`subTest()`**
ของ `unittest` (ค้นหาเอกสารเพิ่มเติมว่า `subTest()` คืออะไร — มันช่วยให้ loop ทดสอบ
หลายค่าในหนึ่ง test method โดยยังเห็นว่าค่าไหน FAIL บ้างแยกกันชัดเจน) แล้วเพิ่ม
`@skipUnless` ตรวจสอบว่า `django.VERSION >= (5, 0)` ก่อนรัน test นี้เสมอ

### 590.5 คำถามที่พบบ่อย (FAQ)

**Q: ต้องเขียน test ให้ครอบคลุม 100% ของโค้ดเลยหรือไม่?**
A: ไม่จำเป็นต้อง 100% เสมอไป (แม้จะเป็นเป้าหมายที่ดี) สิ่งสำคัญกว่าคือทดสอบ **business
logic ที่ซับซ้อนและมีความเสี่ยงสูง** ก่อน เช่น การคำนวณราคา, การ validate ข้อมูล,
custom method ต่าง ๆ — ส่วนโค้ดง่าย ๆ ที่ Django สร้างให้อัตโนมัติ (เช่น `CharField`
ธรรมดาที่ไม่มี logic พิเศษ) มีความสำคัญในการทดสอบน้อยกว่า เราจะเรียนเรื่องการวัด
**Test Coverage** อย่างเป็นระบบใน Part 062

**Q: ทำไม test ที่เขียนในขั้นตอนที่ 590.2 ถึงไม่ต้องลบข้อมูลเองหลังแต่ละ test?**
A: เพราะ `django.test.TestCase` ครอบทุก test method ด้วย transaction ที่ rollback
อัตโนมัติหลังจบ (อธิบายละเอียดในขั้นตอนที่ 586) ข้อมูลทุกอย่างที่สร้างในแต่ละ test
(รวมถึงที่มาจาก `setUpTestData()`) จะถูกล้างกลับสู่สภาพก่อนหน้าโดยอัตโนมัติ ไม่ต้อง
เขียน `tearDown()` เพื่อลบเอง

**Q: `manage.py test` กับ `pytest` ต่างกันอย่างไร ทำไม Part 061 ถึงจะสอน pytest-django
เพิ่มอีก ทั้งที่เราเพิ่งเรียน `unittest` ไป?**
A: `manage.py test` ใช้ `unittest` runner ที่มากับ Django โดยตรง เขียนง่าย ไม่ต้อง
ติดตั้งอะไรเพิ่ม เหมาะกับการเรียนรู้พื้นฐาน (จุดประสงค์ของ Part นี้) ส่วน `pytest`
เป็นเครื่องมือ third-party ที่ได้รับความนิยมสูงมากในวงการ Python เพราะมี syntax ที่
กระชับกว่า (ไม่ต้องมี class), fixture system ที่ยืดหยุ่นกว่า, และ plugin ecosystem
ขนาดใหญ่ **สิ่งสำคัญที่ต้องเข้าใจ**: test ที่เขียนด้วย `unittest.TestCase`/
`django.test.TestCase` แบบใน Part นี้**ยังคงรันได้ปกติแม้เปลี่ยนไปใช้ pytest แล้ว**
เพราะ pytest รองรับการรัน `unittest`-style test ได้อยู่แล้วโดยไม่ต้องแก้โค้ดเลย
ความรู้ใน Part นี้จึงไม่มีวันล้าสมัยแม้จะย้ายไปใช้เครื่องมืออื่นในอนาคต

**Q: ถ้า test FAIL ใน CI/CD แต่รันบนเครื่องตัวเองผ่านปกติ ควรสงสัยอะไรก่อน?**
A: อันดับแรกให้สงสัยเรื่อง **Test Isolation** (ขั้นตอนที่ 586) — ลองรันด้วย
`python manage.py test --reverse` และ `python manage.py test --parallel` บนเครื่อง
ตัวเองดูว่ายัง PASS อยู่ไหม ถ้า FAIL แปลว่ามี test ที่แอบพึ่งพาลำดับการรันหรือ
สถานะที่ค้างจาก test อื่นโดยไม่ตั้งใจ อันดับสองให้ตรวจสอบว่าฐานข้อมูลที่ใช้ใน CI/CD
เป็นชนิดเดียวกับที่ใช้บนเครื่อง dev หรือไม่ (ขั้นตอนที่ 583.3) เพราะพฤติกรรมบางอย่าง
ต่างกันระหว่าง SQLite และ PostgreSQL/MySQL จริง

### 590.6 เตรียมตัวสำหรับ Part ถัดไป

**Part 060: Testing Views, Models และ Forms** จะพา test suite ของเราก้าวออกจาก
การทดสอบ model ล้วน ๆ ไปสู่การทดสอบ **HTTP layer เต็มรูปแบบ** ด้วย **Django Test
Client** (`self.client`) เราจะ:

- เรียนรู้ `self.client.get()`, `self.client.post()` เพื่อจำลอง HTTP request จริง
  โดยไม่ต้องเปิดเบราว์เซอร์
- ใช้ assert method เฉพาะของ View ที่เก็บไว้ไม่พูดถึงใน Part นี้: `assertContains()`,
  `assertRedirects()`, `assertTemplateUsed()`, `assertFormError()`
- ทดสอบ `PostListView`/`PostDetailView` (Part 022), `PostCreateView`/`PostUpdateView`
  (Part 023) ให้ครบทุก HTTP method และทุกเงื่อนไข permission (Part 033, 038)
- ทดสอบ `PostForm`/`ModelForm` (Part 025-026) ทั้งกรณีข้อมูลถูกต้องและข้อมูลผิดพลาด
- เชื่อมโยงทุกอย่างเข้ากับ `Post` model และ business logic ที่ทดสอบไว้แล้วใน Part นี้
  ให้เห็นภาพรวมว่า Model + View + Form ทำงานร่วมกันถูกต้องจริงแบบ end-to-end

Test suite ของคุณจะเริ่มมีความมั่นใจมากขึ้นเรื่อย ๆ ตลอด Phase 7 — เตรียม `blog/tests/`
ที่สร้างไว้ใน Part นี้ให้พร้อม แล้วไปทดสอบ View กันต่อเลย!
