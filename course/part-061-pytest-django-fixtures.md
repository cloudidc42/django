# Part 061: pytest-django และ Fixtures

> **ขั้นตอนที่ 601-610 ของหลักสูตร** | Phase 7: Testing & Quality Assurance
>
> เป้าหมายของ Part นี้: ย้ายทักษะการเขียน test ที่คุณสร้างไว้ใน Part 059-060 ด้วย
> `unittest.TestCase` ของ Django มาสู่โลกของ **pytest** ซึ่งเป็นเครื่องมือทดสอบที่ทีม
> มืออาชีพจำนวนมากทั่วโลกเลือกใช้ในปี 2025-2026 คุณจะเข้าใจว่าทำไม pytest ถึงได้รับ
> ความนิยมสูงกว่า unittest ในโปรเจกต์ใหม่ ๆ ติดตั้งและตั้งค่า `pytest-django` อย่าง
> ถูกต้อง เขียนและแชร์ fixture ด้วย `conftest.py` ใช้ `@pytest.mark.django_db` เพื่อ
> เข้าถึงฐานข้อมูล ใช้ `@pytest.mark.parametrize` เพื่อลดโค้ดซ้ำซ้อน และสุดท้ายจะแปลง
> test suite ของแอป `blog` ทั้งหมดจาก TestCase style ให้เป็น pytest function-style
> อย่างสมบูรณ์ พร้อมรู้จัก plugin ที่ขาดไม่ได้อย่าง `pytest-cov`, `pytest-xdist` และ
> `pytest-mock`

---

## สารบัญของ Part นี้

- ขั้นตอนที่ 601: ทำไมทีมมืออาชีพจำนวนมากเลือก pytest แทน unittest
- ขั้นตอนที่ 602: ติดตั้ง `pytest-django`, ตั้งค่า `pytest.ini`/`pyproject.toml` และ `conftest.py`
- ขั้นตอนที่ 603: Fixture เบื้องต้นด้วย `@pytest.fixture`
- ขั้นตอนที่ 604: `db` fixture และ `@pytest.mark.django_db` — การเข้าถึงฐานข้อมูลใน pytest
- ขั้นตอนที่ 605: `@pytest.mark.parametrize` — รัน test เดียวกันหลายชุดข้อมูล
- ขั้นตอนที่ 606: `conftest.py` สำหรับแชร์ fixture ข้ามไฟล์ test หลายไฟล์
- ขั้นตอนที่ 607: `client` fixture ของ pytest-django (เทียบกับ `self.client` ใน unittest style)
- ขั้นตอนที่ 608: การย้าย test แบบ `TestCase` (unittest style) จาก Part 060 ให้เป็น pytest function-style
- ขั้นตอนที่ 609: ภาพรวม pytest plugin ที่มีประโยชน์ (`pytest-cov`, `pytest-xdist`, `pytest-mock`)
- ขั้นตอนที่ 610: สรุปและแบบฝึกหัด — แปลง test suite ทั้งหมดจาก Part 060 ให้เป็น pytest style

---

## ขั้นตอนที่ 601: ทำไมทีมมืออาชีพจำนวนมากเลือก pytest แทน unittest

### 601.1 ทบทวนสิ่งที่คุณเขียนไว้ใน Part 059-060

ใน Part 059-060 คุณเขียน test ของแอป `blog` โดยใช้ `django.test.TestCase` ซึ่งสืบทอด
มาจาก `unittest.TestCase` ของ Python มาตรฐาน หน้าตาของไฟล์ `blog/tests.py` ที่คุณ
น่าจะมีอยู่ในมือตอนนี้ประมาณนี้:

```python
# blog/tests.py (สไตล์ unittest จาก Part 059-060)
from django.test import TestCase
from django.urls import reverse

from .forms import PostForm
from .models import Post


class PostModelTest(TestCase):
    def setUp(self):
        self.post = Post.objects.create(
            title="ทดสอบ Django ฉบับสมบูรณ์",
            content="เนื้อหาบทความทดสอบ",
            content_type=Post.ContentType.ARTICLE,
            is_published=True,
        )

    def test_str_representation(self):
        self.assertEqual(str(self.post), "ทดสอบ Django ฉบับสมบูรณ์")

    def test_slug_auto_generated(self):
        self.assertTrue(self.post.slug)

    def test_default_content_type_is_article(self):
        post = Post.objects.create(title="เริ่มต้น", content="...")
        self.assertEqual(post.content_type, Post.ContentType.ARTICLE)


class PostListViewTest(TestCase):
    def setUp(self):
        self.published = Post.objects.create(
            title="เผยแพร่แล้ว", content="...", is_published=True,
        )
        self.draft = Post.objects.create(
            title="ฉบับร่าง", content="...", is_published=False,
        )

    def test_list_view_status_code(self):
        response = self.client.get(reverse("blog:list"))
        self.assertEqual(response.status_code, 200)

    def test_list_view_shows_only_published_posts(self):
        response = self.client.get(reverse("blog:list"))
        self.assertContains(response, self.published.title)
        self.assertNotContains(response, self.draft.title)


class PostFormTest(TestCase):
    def test_form_invalid_when_title_too_short(self):
        form = PostForm(data={
            "title": "สั้น",
            "content": "เนื้อหา",
            "content_type": Post.ContentType.ARTICLE,
        })
        self.assertFalse(form.is_valid())
        self.assertIn("title", form.errors)
```

โค้ดนี้ **ไม่ได้ผิดอะไรเลย** และยังใช้งานได้ดีในหลายทีม แต่ Part นี้จะแสดงให้เห็นว่า
เมื่อ codebase โตขึ้น (สมมติว่าคุณมีหลายร้อย test เหมือนโปรเจกต์ระดับ Instagram หรือ
Mozilla) รูปแบบ function-based ของ pytest จะดูแลรักษาง่ายกว่ามาก

### 601.2 ปรัชญาของ pytest: "test คือฟังก์ชันธรรมดา ไม่ใช่ method ใน class"

**pytest** คือ testing framework ของ Python ที่ไม่ได้มาพร้อม Python (ต้องติดตั้งเพิ่ม)
แต่กลายเป็นเครื่องมือทดสอบที่มีคนดาวน์โหลดจาก PyPI มากที่สุดในโลก Python เพราะแนวคิด
หลักคือ:

- **Test ไม่จำเป็นต้องอยู่ใน class** — เขียนเป็นฟังก์ชันธรรมดาที่ขึ้นต้นด้วย `test_`
  ก็พอ ไม่ต้องสืบทอดจาก `TestCase` เสมอไป
- **ใช้ `assert` ธรรมดาของ Python** — ไม่ต้องจำชื่อ method อย่าง `assertEqual`,
  `assertTrue`, `assertIn` อีกต่อไป
- **Dependency Injection ผ่าน Fixture** — แทนที่จะใช้ `setUp()`/`tearDown()` คุณขอ
  "ของที่ต้องใช้" ผ่าน parameter ของฟังก์ชัน แล้ว pytest จะฉีด (inject) ให้อัตโนมัติ
- **Plugin ecosystem ขนาดใหญ่มาก** — มี plugin กว่า 1,300+ ตัวบน PyPI ครอบคลุมตั้งแต่
  coverage, mocking, parallel testing ไปจนถึง Django/Flask/FastAPI integration

### 601.3 ตารางเปรียบเทียบ syntax แบบละเอียด

| หัวข้อ | `unittest` (Django `TestCase`) | `pytest` |
|---|---|---|
| การประกาศ test | ต้องอยู่ใน class ที่สืบทอด `TestCase` | เขียนเป็นฟังก์ชันธรรมดาได้เลย (class ก็ยังใช้ได้ถ้าต้องการ) |
| การเปรียบเทียบค่า | `self.assertEqual(a, b)` | `assert a == b` |
| ตรวจสอบค่า True/False | `self.assertTrue(x)` | `assert x` |
| ตรวจสอบว่ามีค่าอยู่ใน list | `self.assertIn(a, b)` | `assert a in b` |
| ตรวจสอบว่า raise exception | `with self.assertRaises(ValueError):` | `with pytest.raises(ValueError):` |
| Setup ก่อน test | `def setUp(self):` | `@pytest.fixture` function |
| Teardown หลัง test | `def tearDown(self):` | ส่วนหลัง `yield` ใน fixture |
| Setup ระดับ class (ครั้งเดียว) | `setUpTestData(cls)` classmethod | fixture scope `"class"` หรือ `"module"` |
| รัน test เดียว | `python manage.py test blog.tests.PostModelTest.test_str_representation` | `pytest blog/tests/test_models.py::test_str_representation` |
| รัน test หลายชุดข้อมูล (data-driven) | ต้องเขียน loop เองใน `setUp` หรือใช้ library เสริม | `@pytest.mark.parametrize` ในตัว |
| ข้าม test แบบมีเงื่อนไข | `@unittest.skipIf(condition, "เหตุผล")` | `@pytest.mark.skipif(condition, reason="เหตุผล")` |
| แสดงผล diff ตอน assert ล้มเหลว | ข้อความสั้น ต้องอ่าน traceback เอง | แสดง diff ละเอียดของ assert แบบอัตโนมัติ (assertion rewriting) |
| การเข้าถึงฐานข้อมูล Django | ได้อัตโนมัติเมื่ออยู่ใน `TestCase` | ต้องขอผ่าน fixture `db` หรือ marker `@pytest.mark.django_db` |
| Test client | `self.client` (attribute มากับ class) | fixture `client` (รับผ่าน parameter) |

### 601.4 ตารางเปรียบเทียบ ecosystem และ plugin

| ความต้องการ | ฝั่ง unittest/Django | ฝั่ง pytest |
|---|---|---|
| วัด code coverage | `coverage run manage.py test` | `pytest-cov` (ผนวกเข้ากับ pytest โดยตรง) |
| Mock object | `unittest.mock` (built-in) | `unittest.mock` ใช้ได้เหมือนกัน + `pytest-mock` ทำให้ syntax สั้นลงผ่าน fixture `mocker` |
| รัน test แบบขนาน (parallel) | `python manage.py test --parallel` | `pytest-xdist` (`pytest -n auto`) ยืดหยุ่นกว่าและใช้ได้นอก Django ด้วย |
| สร้างข้อมูลทดสอบ (test data factory) | เขียน helper function เอง หรือ fixtures (JSON) | `factory_boy` + `pytest-factoryboy` (จะเรียนใน Part 063) |
| แสดงผลลัพธ์ test สวยงามอ่านง่าย | ข้อความ dot/E ธรรมดา | `pytest-sugar` แสดง progress bar และสีสัน |
| สุ่มลำดับ test เพื่อจับ test ที่พึ่งพากันผิด ๆ | ไม่มีในตัว | `pytest-randomly` |
| ตั้งค่า environment variable ให้ test | ต้องเขียน `setUp`/`tearDown` เอง | `pytest-env` |
| จำนวน plugin บน PyPI | ผูกกับ Django/unittest เท่านั้น | มากกว่า 1,300 plugin ครอบคลุมแทบทุก framework |

### 601.5 pytest ไม่ได้ลอยมาจากไหน — และใครใช้งานจริงบ้าง

pytest ถูกพัฒนาโดย Holger Krekel เริ่มต้นในชื่อ `py.test` ตั้งแต่ปี 2004 และกลายเป็น
โปรเจกต์อิสระที่ได้รับการดูแลจากชุมชนขนาดใหญ่ในเวลาต่อมา ปัจจุบัน pytest ถูกใช้เป็น
มาตรฐานการทดสอบใน:

- **โปรเจกต์ open source ขนาดใหญ่จำนวนมากในระบบนิเวศ Python** เช่น `pandas`,
  `requests`, `Flask`, `scikit-learn` และ `black` ล้วนใช้ pytest เป็น test runner หลัก
- **ทีม backend ที่ใช้ Django ในระดับ production** จำนวนมากเปลี่ยนจาก
  `manage.py test` มาเป็น `pytest` + `pytest-django` เพราะได้ syntax ที่กระชับกว่า
  โดยไม่ต้องแลกกับความสามารถของ Django test client
- **เครื่องมือ CI/CD สมัยใหม่** (GitHub Actions, GitLab CI) รองรับ pytest ออกรายงาน
  แบบ JUnit XML ได้ในตัว ทำให้ dashboard แสดงผลได้ตรงมาตรฐานเดียวกับภาษาอื่น

### 601.6 ข้อควรระวัง: ไม่ต้องทิ้ง `TestCase` ทั้งหมดในคราวเดียว

ข่าวดีคือ **pytest รัน `unittest.TestCase` (รวมถึง Django `TestCase`) ได้โดยไม่ต้อง
แก้โค้ดแม้แต่บรรทัดเดียว** เพราะ pytest ออกแบบมาให้ discover และรัน test ทุกรูปแบบที่
Python รู้จัก ดังนั้นการย้ายจาก unittest ไป pytest **ทำแบบค่อยเป็นค่อยไปได้เต็มที่**
ไม่จำเป็นต้อง rewrite ทั้ง codebase ในวันเดียว ซึ่งเป็นแนวทางที่ทีมมืออาชีพเลือกใช้จริง
(migrate ทีละไฟล์ ทีละ sprint) — Part นี้จะสอนทั้งวิธีตั้งค่าให้ pytest รัน test เก่าได้
และวิธีแปลง test ใหม่ให้เป็น pytest-native style

---

## ขั้นตอนที่ 602: ติดตั้ง `pytest-django`, ตั้งค่า `pytest.ini`/`pyproject.toml` และ `conftest.py`

### 602.1 ติดตั้งแพ็กเกจที่จำเป็น

ตามหลักการจาก Part 003 เรื่องการจัดการ dependency: เครื่องมือทดสอบเป็น
**development dependency** ไม่ใช่สิ่งที่ต้องติดตั้งบน production server จึงควรแยกไฟล์
`requirements-dev.txt` ออกจาก `requirements.txt` หลัก

```bash
# ตรวจสอบว่า venv ถูก activate อยู่ก่อนเสมอ
source venv/bin/activate

# ติดตั้ง pytest และตัวเชื่อมกับ Django
pip install "pytest~=8.3" "pytest-django~=4.9"
```

บันทึกลงไฟล์ dev dependency:

```bash
# requirements-dev.txt
-r requirements.txt

pytest~=8.3
pytest-django~=4.9
```

```bash
pip install -r requirements-dev.txt
```

### 602.2 ตั้งค่าผ่าน `pytest.ini` (แนวทางที่พบบ่อยที่สุด)

สร้างไฟล์ `pytest.ini` ไว้ที่ **root ของโปรเจกต์** (ระดับเดียวกับ `manage.py`):

```ini
; pytest.ini
[pytest]
DJANGO_SETTINGS_MODULE = config.settings
python_files = tests.py test_*.py *_tests.py
testpaths = blog
addopts = --reuse-db -v
```

อธิบายแต่ละบรรทัด:

| ค่า config | ความหมาย |
|---|---|
| `DJANGO_SETTINGS_MODULE` | บอก `pytest-django` ว่า Django settings module ของโปรเจกต์อยู่ที่ไหน (สำคัญที่สุด ถ้าไม่ตั้งจะ import Model ไม่ได้เลย) |
| `python_files` | pattern ของไฟล์ที่ pytest จะถือว่าเป็นไฟล์ test (ค่า default ของ pytest คือ `test_*.py *_test.py` เท่านั้น เพิ่ม `tests.py` เพื่อให้เข้ากับไฟล์เดิมจาก Part 059-060) |
| `testpaths` | โฟลเดอร์ที่จะให้ pytest ค้นหา test โดยไม่ต้องพิมพ์ path เอง (เพิ่มชื่อแอปอื่น ๆ คั่นด้วยช่องว่างเมื่อมีหลายแอป) |
| `addopts` | ตัวเลือกที่จะถูกเติมให้ทุกครั้งที่รัน `pytest` เฉย ๆ (`--reuse-db` ใช้ฐานข้อมูลทดสอบเดิมแทนสร้างใหม่ทุกครั้งเพื่อความเร็ว, `-v` แสดงผลละเอียด) |

### 602.3 ทางเลือก: ตั้งค่าผ่าน `pyproject.toml`

ถ้าโปรเจกต์ของคุณใช้ `pyproject.toml` เป็นศูนย์กลาง config อยู่แล้ว (ตามที่แนะนำใน
Part 003 ขั้นตอนที่ 25 เรื่อง Poetry) สามารถย้าย config เดียวกันมาไว้ที่นี่แทนได้
โดยไม่ต้องมีไฟล์ `pytest.ini` แยก (ห้ามมีทั้งสองไฟล์พร้อมกัน `pytest.ini` จะถูกใช้ก่อนเสมอ):

```toml
# pyproject.toml
[tool.pytest.ini_options]
DJANGO_SETTINGS_MODULE = "config.settings"
python_files = ["tests.py", "test_*.py", "*_tests.py"]
testpaths = ["blog"]
addopts = "--reuse-db -v"
```

### 602.4 `conftest.py` — จุดเริ่มต้นของ fixture ที่แชร์ได้ทั้งโปรเจกต์

`conftest.py` คือไฟล์พิเศษที่ pytest จะโหลดอัตโนมัติ **โดยไม่ต้อง import** วางไว้ที่
root ของโปรเจกต์เพื่อให้ทุกไฟล์ test มองเห็น fixture ที่ประกาศไว้ในนี้ (รายละเอียด
เชิงลึกเรื่องการแชร์ fixture ข้ามหลายไฟล์อยู่ในขั้นตอนที่ 606):

```python
# conftest.py (วางไว้ที่ root โปรเจกต์ ระดับเดียวกับ manage.py)
import pytest


@pytest.fixture
def sample_post(db):
    """Fixture พื้นฐานสำหรับสร้าง Post หนึ่งรายการที่เผยแพร่แล้ว"""
    from blog.models import Post

    return Post.objects.create(
        title="บทความตัวอย่างสำหรับทดสอบ",
        content="เนื้อหาบทความตัวอย่าง",
        content_type=Post.ContentType.ARTICLE,
        is_published=True,
    )
```

### 602.5 ตรวจสอบว่าทุกอย่างทำงานถูกต้อง

```bash
# ตรวจสอบเวอร์ชัน
pytest --version
# pytest 8.3.x
# pytest-django-4.9.x

# รัน test ทั้งหมดในโปรเจกต์
pytest

# รันเฉพาะแอป blog
pytest blog/

# รันไฟล์เดียว
pytest blog/tests.py

# รัน test function/method เดียว
pytest blog/tests.py::PostModelTest::test_str_representation
```

ถ้าเห็นข้อความประมาณนี้ แสดงว่าตั้งค่าสำเร็จ:

```
============================= test session starts ==============================
platform linux -- Python 3.12.4, pytest-8.3.3, pluggy-1.5.0
django settings: config.settings (from ini file)
rootdir: /home/user/django-mastery-course
configfile: pytest.ini
testpaths: blog
plugins: django-4.9.0
collected 7 items

blog/tests.py::PostModelTest::test_str_representation PASSED             [ 14%]
blog/tests.py::PostModelTest::test_slug_auto_generated PASSED            [ 28%]
...
============================== 7 passed in 0.42s ===============================
```

สังเกตว่า **test เก่าที่เขียนแบบ `TestCase` ยังรันผ่านได้ทุกตัว** โดยไม่ต้องแก้อะไร
เลย — นี่คือสิ่งที่ยืนยันคำกล่าวในขั้นตอนที่ 601.6

### 602.6 ตารางเปรียบเทียบคำสั่งรัน test

| งานที่ต้องการทำ | `manage.py test` | `pytest` |
|---|---|---|
| รันทั้งหมด | `python manage.py test` | `pytest` |
| รันเฉพาะแอป | `python manage.py test blog` | `pytest blog/` |
| รัน class เดียว | `python manage.py test blog.tests.PostModelTest` | `pytest blog/tests.py::PostModelTest` |
| รัน test เดียว | `python manage.py test blog.tests.PostModelTest.test_str_representation` | `pytest blog/tests.py::PostModelTest::test_str_representation` |
| หยุดทันทีเมื่อเจอ fail ตัวแรก | `--failfast` | `-x` |
| แสดงผลละเอียด | `--verbosity 2` | `-v` |
| ใช้ฐานข้อมูลเดิมซ้ำ (เร็วขึ้น) | `--keepdb` | `--reuse-db` |
| รันแบบขนาน | `--parallel` | `-n auto` (ต้องติดตั้ง `pytest-xdist`) |

---

## ขั้นตอนที่ 603: Fixture เบื้องต้นด้วย `@pytest.fixture`

### 603.1 Fixture คืออะไร

**Fixture** คือฟังก์ชันที่ pytest จะเรียกให้ก่อนที่ test function จะรัน แล้วนำค่าที่
`return` (หรือ `yield`) กลับมา ไปใส่ให้ parameter ของ test function ที่ชื่อตรงกับชื่อ
fixture นั้น กลไกนี้เรียกว่า **Dependency Injection** — test ไม่ต้องรู้ว่าของถูกสร้าง
ขึ้นมาอย่างไร รู้แค่ว่า "ขอของชิ้นนี้มาใช้" ก็พอ

```python
# ไม่เกี่ยวกับ Django เลย — เข้าใจแนวคิดพื้นฐานก่อน
import pytest


@pytest.fixture
def sample_numbers():
    return [1, 2, 3, 4, 5]


def test_sum_of_numbers(sample_numbers):
    assert sum(sample_numbers) == 15


def test_max_of_numbers(sample_numbers):
    assert max(sample_numbers) == 5
```

สังเกตว่า `sample_numbers` ถูกใช้ซ้ำได้ในหลาย test function โดยไม่ต้องเขียน
`setUp()` เหมือน unittest และแต่ละ test ยังได้ค่า **สดใหม่** ทุกครั้ง (fixture ถูกเรียก
ใหม่ทุกครั้งที่ test เริ่ม ตาม scope เริ่มต้น)

### 603.2 fixture ที่ใช้ fixture อื่นเป็น input

Fixture หนึ่งตัวสามารถ "ขอ" fixture ตัวอื่นเป็น parameter ได้เช่นกัน ทำให้สร้าง
ห่วงโซ่การเตรียมข้อมูล (composition) ได้อย่างเป็นระบบ:

```python
@pytest.fixture
def sample_numbers():
    return [1, 2, 3, 4, 5]


@pytest.fixture
def doubled_numbers(sample_numbers):
    return [n * 2 for n in sample_numbers]


def test_doubled_numbers(doubled_numbers):
    assert doubled_numbers == [2, 4, 6, 8, 10]
```

### 603.3 ตาราง scope ของ fixture

Fixture มี parameter `scope` กำหนดว่า pytest จะสร้างค่าใหม่บ่อยแค่ไหน ยิ่ง scope
กว้าง ยิ่งเร็ว แต่ต้องระวังเรื่อง state รั่วไหลระหว่าง test:

| scope | สร้างใหม่เมื่อไหร่ | เหมาะกับ |
|---|---|---|
| `"function"` (ค่า default) | ทุก test function | ข้อมูลที่ต้องสดใหม่เสมอ ไม่อยากให้ test มีผลต่อกัน |
| `"class"` | ครั้งเดียวต่อ test class | ข้อมูลที่ใช้ร่วมกันได้ในกลุ่ม test ที่เกี่ยวข้องกัน |
| `"module"` | ครั้งเดียวต่อไฟล์ test | การตั้งค่าที่หนักและไม่เปลี่ยนแปลงระหว่าง test ในไฟล์เดียวกัน |
| `"package"` | ครั้งเดียวต่อแพ็กเกจ (โฟลเดอร์ที่มี `__init__.py`) | ใช้น้อย เหมาะกับ config ระดับกลุ่มแอป |
| `"session"` | ครั้งเดียวตลอดการรัน pytest ทั้งหมด | การเชื่อมต่อภายนอกที่มีค่าใช้จ่ายสูงมาก เช่น browser instance ของ Selenium |

```python
@pytest.fixture(scope="module")
def expensive_config():
    print("\n[สร้าง config ที่หนักหน่วง — ครั้งเดียวต่อไฟล์]")
    return {"api_key": "dummy-key", "timeout": 30}
```

**ข้อควรระวังสำคัญ**: fixture ที่แตะฐานข้อมูล Django (`db`, `django_db_setup`) ควร
ระวังการใช้ scope กว้างกว่า `"function"` เพราะ Django จะ rollback transaction ของแต่ละ
test อัตโนมัติ (อธิบายละเอียดในขั้นตอนที่ 604) การผสม scope กว้างกับ database fixture
ต้องเข้าใจกลไก transaction ให้ดีก่อน ไม่เช่นนั้นข้อมูลจาก test หนึ่งอาจรั่วไปอีก test

### 603.4 yield fixture: จัดการ setup และ teardown ในฟังก์ชันเดียว

ถ้า fixture ต้อง "ทำความสะอาด" หลัง test จบ (เทียบเท่า `tearDown()` ของ unittest)
ให้ใช้ `yield` แทน `return` — โค้ดก่อน `yield` คือ setup โค้ดหลัง `yield` คือ teardown:

```python
import pytest


@pytest.fixture
def temp_media_root(tmp_path, settings):
    """เปลี่ยน MEDIA_ROOT ไปยังโฟลเดอร์ชั่วคราวระหว่างการทดสอบ แล้วคืนค่าเดิมทีหลัง"""
    original_media_root = settings.MEDIA_ROOT
    settings.MEDIA_ROOT = tmp_path

    yield tmp_path

    settings.MEDIA_ROOT = original_media_root
```

`tmp_path` และ `settings` ในตัวอย่างข้างบนคือ **fixture ที่มากับ pytest/pytest-django
โดยตรง** (`tmp_path` สร้างโฟลเดอร์ชั่วคราวให้อัตโนมัติ, `settings` ทำให้แก้ไข Django
settings ชั่วคราวได้และคืนค่าเดิมให้เองเมื่อ test จบ) นี่คือตัวอย่างที่แสดงให้เห็นพลัง
ของการที่ fixture เรียกใช้ fixture อื่นได้อย่างอิสระ

### 603.5 ตัวอย่างจริง: fixture สำหรับแอป `blog`

```python
# blog/conftest.py (fixture เฉพาะแอป blog — รายละเอียดเรื่อง conftest หลายระดับ
# อยู่ในขั้นตอนที่ 606)
import pytest

from blog.models import Post


@pytest.fixture
def post_data():
    """ข้อมูลดิบสำหรับสร้าง Post — ยังไม่บันทึกลงฐานข้อมูล"""
    return {
        "title": "บทความทดสอบมาตรฐาน",
        "content": "เนื้อหาทดสอบที่มีความยาวเพียงพอสำหรับการทดสอบ",
        "content_type": Post.ContentType.ARTICLE,
        "is_published": True,
    }


@pytest.fixture
def published_post(db, post_data):
    """Post ที่บันทึกลงฐานข้อมูลจริงและเผยแพร่แล้ว"""
    return Post.objects.create(**post_data)


@pytest.fixture
def draft_post(db, post_data):
    """Post ที่บันทึกลงฐานข้อมูลจริงแต่ยังเป็นฉบับร่าง"""
    data = {**post_data, "title": "ฉบับร่างสำหรับทดสอบ", "is_published": False}
    return Post.objects.create(**data)
```

ตัวอย่างนี้แสดงรูปแบบที่ทีมมืออาชีพนิยมใช้: แยก **ข้อมูลดิบ** (`post_data`) ออกจาก
**object ที่บันทึกจริง** (`published_post`, `draft_post`) เพื่อให้ทดสอบ form validation
ด้วยข้อมูลดิบได้ โดยไม่ต้องสร้างและลบ object ในฐานข้อมูลจริงเมื่อไม่จำเป็น

---

## ขั้นตอนที่ 604: `db` fixture และ `@pytest.mark.django_db` — การเข้าถึงฐานข้อมูลใน pytest

### 604.1 ทำไม pytest ธรรมดาแตะฐานข้อมูล Django ไม่ได้

นี่คือความแตกต่างสำคัญที่สุดระหว่าง Django `TestCase` กับ pytest ธรรมดา: เมื่อคุณ
สืบทอดจาก `django.test.TestCase` Django จะจัดการเปิด transaction ฐานข้อมูลทดสอบและ
`rollback` ให้อัตโนมัติหลัง test จบทุกครั้ง แต่ pytest **ไม่รู้จัก Django** โดยธรรมชาติ
ถ้าคุณลองรันโค้ดนี้ตรง ๆ:

```python
# ผิด! จะได้ error: "Database access not allowed"
def test_create_post_without_marker():
    from blog.models import Post
    Post.objects.create(title="ทดสอบ", content="เนื้อหา")
```

จะได้ error:

```
RuntimeError: Database access not allowed, use the "django_db" mark, or the
"db" or "transactional_db" fixtures to enable it.
```

นี่คือ **feature ไม่ใช่ bug** — `pytest-django` ป้องกันไม่ให้ test ที่ไม่ตั้งใจแตะ
ฐานข้อมูลไปแตะโดยไม่รู้ตัว (ซึ่งจะทำให้ test ช้าลงและอาจมีผลข้างเคียงที่ไม่คาดคิด)
คุณต้อง "ขออนุญาต" อย่างชัดเจนก่อนเสมอ

### 604.2 วิธีที่ 1: `@pytest.mark.django_db` (มาร์กที่ระดับฟังก์ชัน)

```python
import pytest

from blog.models import Post


@pytest.mark.django_db
def test_create_post():
    post = Post.objects.create(title="ทดสอบ", content="เนื้อหา")
    assert Post.objects.count() == 1
    assert post.pk is not None
```

### 604.3 วิธีที่ 2: fixture `db` (ผ่าน parameter ของฟังก์ชัน)

```python
def test_create_post_with_db_fixture(db):
    post = Post.objects.create(title="ทดสอบ", content="เนื้อหา")
    assert post.pk is not None
```

ทั้งสองวิธีทำงาน**เหมือนกันทุกประการ** — เลือกใช้ตามความสะดวก แต่ทีมส่วนใหญ่นิยม
`@pytest.mark.django_db` เพราะอ่านง่ายกว่าตอนสแกนโค้ดผ่าน ๆ (เห็น decorator ด้านบน
ฟังก์ชันชัดเจนกว่าต้องไล่ดู parameter list) ส่วน fixture `db` มักใช้เมื่อ fixture ตัวอื่น
ต้องพึ่งพาฐานข้อมูลด้วย (อย่างที่เห็นใน `published_post(db, post_data)` ในขั้นตอนก่อนหน้า)

### 604.4 `transactional_db` — เมื่อไหร่ต้องใช้ transaction จริง

โดยปกติ `django_db`/`db` จะห่อ test ด้วย transaction แล้ว rollback ทันทีหลังจบ ทำให้
เร็วมาก แต่บาง feature ของ Django (เช่น `transaction.on_commit()`, การทดสอบ
`select_for_update()` ข้าม thread, หรือ raw SQL ที่ผูกกับ transaction commit จริง)
ต้องการ transaction ที่ commit ได้จริง ให้ใช้ `transactional_db` หรือส่ง
`transaction=True` ให้ marker:

```python
import pytest
from django.db import transaction

from blog.models import Post


@pytest.mark.django_db(transaction=True)
def test_on_commit_hook_runs_after_real_commit(django_capture_on_commit_callbacks):
    with django_capture_on_commit_callbacks(execute=True) as callbacks:
        with transaction.atomic():
            Post.objects.create(title="ทดสอบ on_commit", content="เนื้อหา")

    # callbacks ที่ลงทะเบียนผ่าน transaction.on_commit() จะถูกเรียกจริงที่นี่
    assert isinstance(callbacks, list)
```

`django_capture_on_commit_callbacks` เป็น fixture ของ `pytest-django` ที่ห่อ
helper ตัวเดียวกับที่ Django ใช้ใน `TestCase.captureOnCommitCallbacks()` มาให้แบบ
fixture-native — สะดวกกว่าการเขียน context manager เอง

### 604.5 `reset_sequences` — เมื่อต้องการให้ primary key เริ่มจาก 1 เสมอ

```python
@pytest.mark.django_db(reset_sequences=True)
def test_post_pk_starts_from_one():
    post = Post.objects.create(title="ทดสอบ", content="เนื้อหา")
    assert post.pk == 1
```

ค่า default คือ `reset_sequences=False` เพื่อความเร็ว เปิดใช้เฉพาะเมื่อ test ของคุณ
ต้องพึ่งพาค่า primary key ที่แน่นอน (ซึ่งโดยทั่วไปถือเป็น anti-pattern — ควรออกแบบ
test ให้ไม่ผูกกับค่า pk ตายตัวถ้าเป็นไปได้)

### 604.6 ตัวเลือก command line ที่เกี่ยวกับฐานข้อมูลทดสอบ

| flag | ความหมาย |
|---|---|
| `--reuse-db` | ใช้ฐานข้อมูลทดสอบเดิมที่มีอยู่แล้วแทนการสร้างใหม่ (เร็วขึ้นมากในการรันซ้ำระหว่างพัฒนา) |
| `--create-db` | บังคับสร้างฐานข้อมูลทดสอบใหม่ แม้จะมี `--reuse-db` ใน `addopts` อยู่ก็ตาม (ใช้เมื่อเพิ่ง `makemigrations` ใหม่) |
| `--no-migrations` | ข้ามการรัน migration ทั้งหมด แล้วสร้างตารางจาก model ปัจจุบันตรง ๆ (เร็วขึ้นมากสำหรับโปรเจกต์ที่มี migration เยอะ แต่จะไม่ทดสอบว่า migration ทำงานถูกต้อง) |
| `--ds=config.settings` | ระบุ `DJANGO_SETTINGS_MODULE` ทาง command line แทนการตั้งค่าใน `pytest.ini` |

```bash
# ระหว่างพัฒนา ใช้ --reuse-db เพื่อความเร็ว
pytest --reuse-db

# หลังเพิ่ม migration ใหม่ ต้องบังคับสร้างฐานข้อมูลใหม่หนึ่งครั้ง
pytest --create-db

# ก่อน push ขึ้น CI ควรรันแบบสร้างฐานข้อมูลใหม่เสมอเพื่อความชัวร์ (CI ไม่ต้องพึ่ง --reuse-db)
pytest
```

### 604.7 ตัวอย่างจริง: ทดสอบ `Post` model แบบครบวงจร

```python
# blog/tests/test_models.py
import pytest

from blog.models import Post


@pytest.fixture
def post(db):
    return Post.objects.create(
        title="ทดสอบ Django ฉบับสมบูรณ์",
        content="เนื้อหาบทความทดสอบ",
        content_type=Post.ContentType.ARTICLE,
        is_published=True,
    )


def test_str_representation(post):
    assert str(post) == "ทดสอบ Django ฉบับสมบูรณ์"


def test_slug_auto_generated(post):
    assert post.slug != ""


def test_default_content_type_is_article(db):
    post = Post.objects.create(title="เริ่มต้น", content="...")
    assert post.content_type == Post.ContentType.ARTICLE


def test_ordering_shows_newest_post_first(db):
    older = Post.objects.create(title="เก่ากว่า", content="...")
    newer = Post.objects.create(title="ใหม่กว่า", content="...")

    posts = list(Post.objects.all())

    assert posts[0] == newer
    assert posts[1] == older
```

โมเดล `Post` ที่ใช้ในตัวอย่างทั้งหมดของ Part นี้อ้างอิงจาก Part 011 พร้อมปรับ
`slug` ให้รองรับภาษาไทยด้วย `allow_unicode=True` (จุดที่มือใหม่มักพลาด เพราะ
`slugify()` ค่าเริ่มต้นจะตัดตัวอักษรที่ไม่ใช่ ASCII ทิ้งหมด ทำให้ slug ของบทความ
ภาษาไทยว่างเปล่า):

```python
# blog/models.py (ส่วนที่เกี่ยวข้องกับ slug)
from django.db import models
from django.urls import reverse
from django.utils.text import slugify


class Post(models.Model):
    class ContentType(models.TextChoices):
        ARTICLE = "ARTICLE", "บทความ"
        TUTORIAL = "TUTORIAL", "สอนการใช้งาน"
        NEWS = "NEWS", "ข่าวสาร"

    title = models.CharField(max_length=200)
    slug = models.SlugField(max_length=220, unique=True, blank=True, allow_unicode=True)
    content = models.TextField()
    excerpt = models.CharField(max_length=300, blank=True)
    content_type = models.CharField(
        max_length=20, choices=ContentType.choices, default=ContentType.ARTICLE,
    )
    is_published = models.BooleanField(default=False)
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)

    class Meta:
        ordering = ["-created_at"]

    def __str__(self):
        return self.title

    def save(self, *args, **kwargs):
        if not self.slug:
            self.slug = slugify(self.title, allow_unicode=True)
        super().save(*args, **kwargs)

    def get_absolute_url(self):
        return reverse("blog:detail", kwargs={"slug": self.slug})
```

---

## ขั้นตอนที่ 605: `@pytest.mark.parametrize` — รัน test เดียวกันหลายชุดข้อมูล

### 605.1 ปัญหาของการเขียน test ซ้ำ ๆ ใน unittest

ลองดูโค้ดสไตล์ unittest ที่ทดสอบ validation หลายกรณีของ `PostForm`:

```python
# แบบ unittest — ต้องเขียน method แยกทุกกรณี (ซ้ำซ้อนมาก)
class PostFormTest(TestCase):
    def test_form_valid_with_long_title(self):
        form = PostForm(data={"title": "หัวข้อความยาวพอสมควร", "content": "...",
                               "content_type": Post.ContentType.ARTICLE})
        self.assertTrue(form.is_valid())

    def test_form_invalid_with_short_title(self):
        form = PostForm(data={"title": "สั้น", "content": "...",
                               "content_type": Post.ContentType.ARTICLE})
        self.assertFalse(form.is_valid())

    def test_form_invalid_with_empty_title(self):
        form = PostForm(data={"title": "", "content": "...",
                               "content_type": Post.ContentType.ARTICLE})
        self.assertFalse(form.is_valid())
```

โครงสร้างซ้ำ 3 รอบ ต่างกันแค่ input กับผลลัพธ์ที่คาดหวัง — นี่คือจุดที่
`@pytest.mark.parametrize` แก้ปัญหาได้อย่างสวยงาม

### 605.2 syntax พื้นฐาน

```python
import pytest


@pytest.mark.parametrize("value,expected", [
    (1, 1),
    (2, 4),
    (3, 9),
])
def test_square(value, expected):
    assert value ** 2 == expected
```

pytest จะรัน test นี้ **3 รอบ** โดยแทนค่า `value, expected` จากแต่ละ tuple ในลิสต์
ผลลัพธ์ตอนรันจะแสดงแยกแต่ละชุดข้อมูลชัดเจน:

```
test_square[1-1] PASSED
test_square[2-4] PASSED
test_square[3-9] PASSED
```

### 605.3 ตั้งชื่อ test case ด้วย `ids` ให้อ่านง่ายขึ้น

ชื่อ default อย่าง `test_square[1-1]` อ่านยากเมื่อข้อมูลซับซ้อนขึ้น ใช้ `ids` เพื่อ
ตั้งชื่อที่สื่อความหมาย:

```python
@pytest.mark.parametrize(
    "title,should_have_error",
    [
        ("หัวข้อความยาวพอสมควร", False),
        ("สั้น", True),
        ("", True),
    ],
    ids=["valid-title", "too-short", "empty-title"],
)
def test_post_form_title_validation(title, should_have_error):
    from blog.forms import PostForm
    from blog.models import Post

    form = PostForm(data={
        "title": title,
        "content": "เนื้อหาบทความ",
        "content_type": Post.ContentType.ARTICLE,
    })
    form.is_valid()

    assert ("title" in form.errors) is should_have_error
```

ตอนรันจะได้ผลลัพธ์อ่านง่ายขึ้นมาก:

```
test_post_form_title_validation[valid-title] PASSED
test_post_form_title_validation[too-short] PASSED
test_post_form_title_validation[empty-title] PASSED
```

### 605.4 รวม `parametrize` เข้ากับ fixture และ `django_db`

```python
import pytest

from blog.models import Post


@pytest.mark.django_db
@pytest.mark.parametrize(
    "content_type",
    [Post.ContentType.ARTICLE, Post.ContentType.TUTORIAL, Post.ContentType.NEWS],
)
def test_post_saves_with_every_content_type(content_type):
    post = Post.objects.create(
        title="ทดสอบประเภทเนื้อหา", content="เนื้อหา", content_type=content_type,
    )
    assert post.content_type == content_type
```

### 605.5 Stack หลาย `parametrize` เพื่อทดสอบแบบ matrix

การวาง `@pytest.mark.parametrize` ซ้อนกันหลายชั้นจะทำให้ pytest รันทุก
**combination** ที่เป็นไปได้ (คูณกันแบบ cartesian product) — ใช้เมื่อต้องการทดสอบ
ความสัมพันธ์ของตัวแปรสองชุดพร้อมกัน:

```python
@pytest.mark.django_db
@pytest.mark.parametrize("content_type", [Post.ContentType.ARTICLE, Post.ContentType.NEWS])
@pytest.mark.parametrize("is_published", [True, False])
def test_post_creation_matrix(content_type, is_published):
    post = Post.objects.create(
        title="ทดสอบ matrix",
        content="เนื้อหา",
        content_type=content_type,
        is_published=is_published,
    )
    assert post.content_type == content_type
    assert post.is_published == is_published
```

โค้ดข้างบนจะรันทั้งหมด **4 test case** (2 ค่าของ `content_type` × 2 ค่าของ
`is_published`) จากการเขียนฟังก์ชันเดียว

### 605.6 ตารางสรุปเมื่อไหร่ควรใช้ parametrize

| สถานการณ์ | ควรใช้ parametrize หรือไม่ |
|---|---|
| ทดสอบ validation logic เดียวกันด้วย input หลายแบบ | ✅ ใช้แน่นอน |
| ทดสอบ edge case ของตัวเลข (0, ค่าติดลบ, ค่ามาก ๆ) | ✅ ใช้แน่นอน |
| ทดสอบ HTTP status code ของหลาย URL ที่มี logic คล้ายกัน | ✅ เหมาะมาก |
| Business logic ของแต่ละกรณีต่างกันโดยสิ้นเชิง (ไม่ใช่แค่ input/output ต่างกัน) | ❌ แยกเป็นฟังก์ชัน test คนละตัวจะอ่านง่ายกว่า |
| ต้องการ error message เฉพาะเจาะจงมากในแต่ละกรณี | ⚠️ ใช้ได้ แต่ใส่ `ids` ให้ครบเสมอเพื่อ debug ง่าย |

---

## ขั้นตอนที่ 606: `conftest.py` สำหรับแชร์ fixture ข้ามไฟล์ test หลายไฟล์

### 606.1 ปัญหา: fixture ซ้ำกันหลายไฟล์

เมื่อโปรเจกต์โตขึ้น คุณจะมีไฟล์ test หลายไฟล์ที่ต้องการ fixture เดียวกัน เช่น
`published_post` ที่ทั้ง `test_models.py`, `test_views.py`, และ `test_forms.py` ต่าง
ก็ต้องใช้ ถ้าประกาศซ้ำในแต่ละไฟล์จะผิดหลัก DRY ที่เรียนมาตั้งแต่ Part 001

### 606.2 `conftest.py` ทำงานอย่างไร

pytest จะสแกนหาไฟล์ชื่อ `conftest.py` ใน **ทุกโฟลเดอร์ตั้งแต่ root ลงไปจนถึงโฟลเดอร์
ที่มีไฟล์ test นั้นอยู่** โดยอัตโนมัติ **ไม่ต้อง `import` เอง** — fixture ทุกตัวที่
ประกาศไว้ใน `conftest.py` จะ "มองเห็น" ได้จากไฟล์ test ทุกไฟล์ในโฟลเดอร์นั้นและ
โฟลเดอร์ย่อยทั้งหมด

### 606.3 โครงสร้างโฟลเดอร์แบบ conftest หลายระดับ

```
django-mastery-course/
├── conftest.py                  # fixture ที่ใช้ได้ "ทั้งโปรเจกต์" ทุกแอป
├── pytest.ini
├── manage.py
├── config/
│   └── settings.py
└── blog/
    ├── conftest.py               # fixture ที่ใช้ได้เฉพาะ "แอป blog" เท่านั้น
    ├── models.py
    ├── forms.py
    ├── views.py
    └── tests/
        ├── __init__.py
        ├── test_models.py        # มองเห็น fixture จาก conftest.py ทั้ง 2 ระดับ
        ├── test_views.py
        └── test_forms.py
```

fixture ที่ประกาศใน `blog/conftest.py` จะ **มองไม่เห็นจากแอปอื่น** (เช่น แอป
`accounts` หรือ `shop` ที่จะเจอในบท frontend/e-commerce ภายหลัง) แต่ fixture ที่
ประกาศใน `conftest.py` ที่ root จะมองเห็นได้จาก**ทุกแอป** นี่คือหลักการ "ยิ่งอยู่ใกล้
root ยิ่งใช้ร่วมกันได้กว้าง"

### 606.4 fixture ระดับ root (ใช้ร่วมกันได้ทั้งโปรเจกต์)

```python
# conftest.py (root)
import pytest


@pytest.fixture
def staff_user(django_user_model, db):
    """ผู้ใช้ staff ทั่วไป — แอปไหนก็ใช้ได้ ไม่ใช่แค่ blog"""
    return django_user_model.objects.create_user(
        username="staffuser", password="testpass123", is_staff=True,
    )


@pytest.fixture
def regular_user(django_user_model, db):
    """ผู้ใช้ทั่วไปที่ไม่มีสิทธิ์พิเศษ"""
    return django_user_model.objects.create_user(
        username="regularuser", password="testpass123",
    )
```

`django_user_model` คือ fixture ที่ `pytest-django` มีให้อัตโนมัติ ชี้ไปที่ Custom
User Model ของโปรเจกต์เสมอ (เทียบเท่า `get_user_model()` ที่เรียนใน Part 032) จึงใช้
ได้แม้คุณจะเปลี่ยนไปใช้ custom user model แล้วก็ตาม

### 606.5 fixture ระดับแอป (เฉพาะ `blog`)

```python
# blog/conftest.py
import pytest

from blog.models import Post


@pytest.fixture
def post_data():
    return {
        "title": "บทความทดสอบมาตรฐาน",
        "content": "เนื้อหาทดสอบที่มีความยาวเพียงพอ",
        "content_type": Post.ContentType.ARTICLE,
        "is_published": True,
    }


@pytest.fixture
def published_post(db, post_data):
    return Post.objects.create(**post_data)


@pytest.fixture
def draft_post(db, post_data):
    data = {**post_data, "title": "ฉบับร่างสำหรับทดสอบ", "is_published": False}
    return Post.objects.create(**data)


@pytest.fixture(autouse=True)
def _clear_post_cache():
    """ตัวอย่าง autouse fixture — รันอัตโนมัติทุก test ในแอป blog โดยไม่ต้องขอ"""
    from django.core.cache import cache

    cache.clear()
    yield
    cache.clear()
```

### 606.6 `autouse=True` คืออะไร และควรใช้เมื่อไหร่

fixture ที่ใส่ `autouse=True` จะถูกเรียก **ทุก test ในขอบเขตนั้นโดยอัตโนมัติ** โดย
ไม่ต้องระบุชื่อใน parameter ของ test เลย เหมาะกับงานที่ต้อง "ทำเสมอ" เช่น เคลียร์
cache ก่อน-หลัง test, รีเซ็ต environment variable, หรือปิดการเชื่อมต่อ external API
จริงระหว่างทดสอบ — ใช้ให้น้อยและเจาะจง เพราะถ้าใช้พร่ำเพรื่อจะทำให้ test อ่านยาก
(เห็น test ทำงานแปลก ๆ โดยไม่รู้ว่ามาจาก autouse fixture ที่ไหน)

---

## ขั้นตอนที่ 607: `client` fixture ของ pytest-django

### 607.1 `self.client` ใน unittest เทียบกับ `client` fixture ใน pytest

ใน Django `TestCase` คุณเข้าถึง test client ผ่าน `self.client` ที่ framework เตรียม
ไว้ให้อัตโนมัติเมื่อ class เริ่มทำงาน ส่วนใน pytest คุณ "ขอ" client ผ่าน parameter
ชื่อ `client` เหมือน fixture ทั่วไป — เบื้องหลังเป็น `django.test.Client` ตัวเดียวกัน
ทุกประการ ไม่มีอะไรต่างกันในเชิงพฤติกรรม

```python
# unittest style
class PostListViewTest(TestCase):
    def test_list_view_status_code(self):
        response = self.client.get(reverse("blog:list"))
        self.assertEqual(response.status_code, 200)
```

```python
# pytest style
from django.urls import reverse


def test_list_view_status_code(client):
    response = client.get(reverse("blog:list"))
    assert response.status_code == 200
```

### 607.2 ตัวอย่างการใช้ `client` ทดสอบ view ของแอป `blog`

```python
# blog/tests/test_views.py
import pytest
from django.urls import reverse


@pytest.mark.django_db
def test_list_view_shows_only_published_posts(client, published_post, draft_post):
    response = client.get(reverse("blog:list"))
    content = response.content.decode()

    assert response.status_code == 200
    assert published_post.title in content
    assert draft_post.title not in content


@pytest.mark.django_db
def test_detail_view_returns_404_for_unknown_slug(client):
    response = client.get(reverse("blog:detail", kwargs={"slug": "not-exist"}))
    assert response.status_code == 404


@pytest.mark.django_db
def test_detail_view_returns_200_for_existing_post(client, published_post):
    url = reverse("blog:detail", kwargs={"slug": published_post.slug})
    response = client.get(url)

    assert response.status_code == 200
    assert response.context["post"] == published_post
```

สังเกตว่า `published_post` และ `draft_post` มาจาก `blog/conftest.py` ที่เขียนไว้ใน
ขั้นตอนที่ 606 — pytest หาให้อัตโนมัติโดยไม่ต้อง import

### 607.3 `admin_client` และ `admin_user` — ทดสอบหน้า admin แบบสั้นที่สุด

`pytest-django` มี fixture พิเศษที่สร้าง superuser และ login ให้เสร็จสรรพในตัวเดียว:

```python
@pytest.mark.django_db
def test_admin_can_access_post_changelist(admin_client):
    response = admin_client.get("/admin/blog/post/")
    assert response.status_code == 200


def test_admin_user_fixture_is_staff_and_superuser(admin_user):
    assert admin_user.is_staff is True
    assert admin_user.is_superuser is True
```

`admin_client` คือ `client` ที่ login เป็น superuser ให้แล้วโดยอัตโนมัติ (สร้าง user
ชื่อ `admin` รหัสผ่าน `password` ให้เสร็จ) เทียบเท่ากับที่คุณต้องเขียนเองแบบนี้ใน
unittest:

```python
# สิ่งที่ admin_client ทำให้อัตโนมัติ ถ้าเขียนเองแบบ unittest จะยาวกว่านี้มาก
class AdminAccessTest(TestCase):
    def setUp(self):
        self.admin = User.objects.create_superuser(
            username="admin", email="admin@example.com", password="password",
        )
        self.client.login(username="admin", password="password")

    def test_admin_can_access_post_changelist(self):
        response = self.client.get("/admin/blog/post/")
        self.assertEqual(response.status_code, 200)
```

### 607.4 `rf` fixture (RequestFactory) — ทดสอบ view function/class โดยตรงโดยไม่ผ่าน URL routing

```python
from blog.views import PostDetailView


def test_post_detail_view_with_request_factory(rf, db, published_post):
    request = rf.get(f"/blog/{published_post.slug}/")
    response = PostDetailView.as_view()(request, slug=published_post.slug)

    assert response.status_code == 200
```

`rf` เหมาะกับกรณีที่ต้องการทดสอบ **เฉพาะ view** โดยไม่สนใจว่า `urls.py` ตั้งค่าไว้
ถูกหรือไม่ (แยกความรับผิดชอบของ test ให้ชัดเจน — การทดสอบ URL routing ควรอยู่คนละ
test กับการทดสอบ view logic)

### 607.5 ตารางสรุป fixture ที่ pytest-django เตรียมไว้ให้

| fixture | หน้าที่ |
|---|---|
| `client` | `django.test.Client` เปล่า ๆ เทียบเท่า `self.client` |
| `admin_client` | `client` ที่ login เป็น superuser ให้แล้ว |
| `admin_user` | instance ของ superuser (สร้างให้อัตโนมัติ ใช้ร่วมกับ `client.force_login()` เองก็ได้) |
| `django_user_model` | คลาส User model ปัจจุบันของโปรเจกต์ (รองรับ custom user model) |
| `rf` | `django.test.RequestFactory` สำหรับสร้าง request object โดยไม่ผ่าน URL dispatcher |
| `db` | เปิดการเข้าถึงฐานข้อมูล (transaction, rollback อัตโนมัติ) |
| `transactional_db` | เปิดการเข้าถึงฐานข้อมูลแบบ transaction จริง (commit ได้จริง) |
| `settings` | แก้ไข Django settings ชั่วคราวระหว่าง test แล้วคืนค่าเดิมให้อัตโนมัติ |
| `mailoutbox` | รายการอีเมลที่ถูกส่งระหว่าง test (เทียบเท่า `django.core.mail.outbox`) |
| `live_server` | รัน Django development server จริงสำหรับทดสอบด้วย Selenium (จะใช้ใน Part 064) |

---

## ขั้นตอนที่ 608: การย้าย test แบบ `TestCase` จาก Part 060 ให้เป็น pytest function-style

### 608.1 หลักการแปลงทีละส่วน

| ส่วนของ unittest | แปลงเป็น pytest อย่างไร |
|---|---|
| `class XxxTest(TestCase):` | ลบ class ทิ้ง เปลี่ยนแต่ละ method เป็นฟังก์ชันระดับ module |
| `def setUp(self): self.x = ...` | เปลี่ยนเป็น `@pytest.fixture def x(): return ...` |
| `def tearDown(self): ...` | ย้ายมาไว้หลัง `yield` ใน fixture เดียวกัน |
| `self.assertEqual(a, b)` | `assert a == b` |
| `self.assertTrue(x)` / `self.assertFalse(x)` | `assert x` / `assert not x` |
| `self.assertIn(a, b)` | `assert a in b` |
| `self.assertContains(response, text)` | `assert text in response.content.decode()` |
| `self.client` | parameter `client` |
| `with self.assertRaises(ValueError):` | `with pytest.raises(ValueError):` |
| ต้องเข้าถึง DB เพราะสืบทอดจาก `TestCase` โดยอัตโนมัติ | ต้องเติม `@pytest.mark.django_db` หรือ parameter `db` เอง |

### 608.2 ตัวอย่างเต็ม: Model test (Before/After)

```python
# ก่อน — blog/tests.py (unittest, จาก Part 059-060)
from django.test import TestCase

from blog.models import Post


class PostModelTest(TestCase):
    def setUp(self):
        self.post = Post.objects.create(
            title="ทดสอบ Django ฉบับสมบูรณ์",
            content="เนื้อหาบทความทดสอบ",
            is_published=True,
        )

    def test_str_representation(self):
        self.assertEqual(str(self.post), "ทดสอบ Django ฉบับสมบูรณ์")

    def test_slug_auto_generated(self):
        self.assertTrue(self.post.slug)

    def test_default_content_type_is_article(self):
        self.assertEqual(self.post.content_type, Post.ContentType.ARTICLE)
```

```python
# หลัง — blog/tests/test_models.py (pytest)
import pytest

from blog.models import Post


@pytest.fixture
def post(db):
    return Post.objects.create(
        title="ทดสอบ Django ฉบับสมบูรณ์",
        content="เนื้อหาบทความทดสอบ",
        is_published=True,
    )


def test_str_representation(post):
    assert str(post) == "ทดสอบ Django ฉบับสมบูรณ์"


def test_slug_auto_generated(post):
    assert post.slug != ""


def test_default_content_type_is_article(post):
    assert post.content_type == Post.ContentType.ARTICLE
```

### 608.3 ตัวอย่างเต็ม: View test (Before/After)

```python
# ก่อน — blog/tests.py (unittest)
from django.test import TestCase
from django.urls import reverse

from blog.models import Post


class PostListViewTest(TestCase):
    def setUp(self):
        self.published = Post.objects.create(
            title="เผยแพร่แล้ว", content="...", is_published=True,
        )
        self.draft = Post.objects.create(
            title="ฉบับร่าง", content="...", is_published=False,
        )

    def test_list_view_status_code(self):
        response = self.client.get(reverse("blog:list"))
        self.assertEqual(response.status_code, 200)

    def test_list_view_shows_only_published_posts(self):
        response = self.client.get(reverse("blog:list"))
        self.assertContains(response, self.published.title)
        self.assertNotContains(response, self.draft.title)
```

```python
# หลัง — blog/tests/test_views.py (pytest)
import pytest
from django.urls import reverse

from blog.models import Post


@pytest.fixture
def published(db):
    return Post.objects.create(title="เผยแพร่แล้ว", content="...", is_published=True)


@pytest.fixture
def draft(db):
    return Post.objects.create(title="ฉบับร่าง", content="...", is_published=False)


def test_list_view_status_code(client):
    response = client.get(reverse("blog:list"))
    assert response.status_code == 200


def test_list_view_shows_only_published_posts(client, published, draft):
    response = client.get(reverse("blog:list"))
    content = response.content.decode()

    assert published.title in content
    assert draft.title not in content
```

### 608.4 ตัวอย่างเต็ม: Form test (Before/After พร้อม parametrize)

```python
# ก่อน — blog/tests.py (unittest, ต้องเขียนแยก method ทุกกรณี)
from django.test import TestCase

from blog.forms import PostForm
from blog.models import Post


class PostFormTest(TestCase):
    def test_form_valid_with_long_title(self):
        form = PostForm(data={
            "title": "หัวข้อความยาวพอสมควร",
            "content": "เนื้อหา",
            "content_type": Post.ContentType.ARTICLE,
        })
        self.assertTrue(form.is_valid())

    def test_form_invalid_with_short_title(self):
        form = PostForm(data={
            "title": "สั้น",
            "content": "เนื้อหา",
            "content_type": Post.ContentType.ARTICLE,
        })
        self.assertFalse(form.is_valid())
        self.assertIn("title", form.errors)
```

```python
# หลัง — blog/tests/test_forms.py (pytest + parametrize รวมทั้งสอง test เดิมเป็นตัวเดียว)
import pytest

from blog.forms import PostForm
from blog.models import Post


@pytest.mark.parametrize(
    "title,should_have_error",
    [
        ("หัวข้อความยาวพอสมควร", False),
        ("สั้น", True),
        ("", True),
    ],
    ids=["valid-title", "too-short", "empty-title"],
)
def test_post_form_title_validation(title, should_have_error):
    form = PostForm(data={
        "title": title,
        "content": "เนื้อหาบทความ",
        "content_type": Post.ContentType.ARTICLE,
    })
    form.is_valid()

    assert ("title" in form.errors) is should_have_error


def test_form_valid_with_all_required_fields():
    form = PostForm(data={
        "title": "หัวข้อทดสอบที่ถูกต้อง",
        "slug": "test-slug",
        "content": "เนื้อหา",
        "content_type": Post.ContentType.ARTICLE,
        "is_published": True,
    })
    assert form.is_valid(), form.errors
```

สังเกตว่า test 2 ตัวจากเวอร์ชัน unittest (และอีก 1 กรณีที่ยังไม่ได้เขียน คือ
empty title) รวมกันเหลือ **1 ฟังก์ชันเดียว** ที่ครอบคลุม 3 สถานการณ์ในเวอร์ชัน pytest

### 608.5 กรณีที่ยังไม่จำเป็นต้องแปลงทันที

ไม่ใช่ทุก test ที่ต้องรีบแปลงเป็น pytest style คุณสามารถคง `TestCase` ไว้ได้เมื่อ:

- **ใช้ `setUpTestData(cls)`** เพื่อสร้างข้อมูลที่หนักและ **ใช้ร่วมกันแบบ read-only**
  ระหว่างทุก test ใน class (Django ห่อด้วย transaction savepoint ให้อัตโนมัติ
  ป้องกัน test หนึ่งแก้ไขข้อมูลแล้วกระทบ test อื่นในกลุ่มเดียวกัน) — การแปลงเป็น
  fixture scope `"class"` ทำได้แต่ต้องระวังเรื่อง mutable state มากกว่า
- **ทดสอบผ่าน Selenium ด้วย `LiveServerTestCase`** ซึ่งจะเรียนใน Part 064 —
  โครงสร้าง class-based ยังเหมาะกับการจัดการ browser driver lifecycle
- **ทีมยังอยู่ระหว่างเปลี่ยนผ่าน** และต้องการลดความเสี่ยงโดยแปลงทีละไฟล์ตามที่กล่าว
  ไว้ในขั้นตอนที่ 601.6

---

## ขั้นตอนที่ 609: ภาพรวม pytest plugin ที่มีประโยชน์

### 609.1 `pytest-cov` — วัด code coverage ในตัวเดียวกับการรัน test

```bash
pip install "pytest-cov~=5.0"
```

```bash
# รัน test พร้อมวัด coverage ของแอป blog เฉพาะ แสดงบรรทัดที่ยังไม่ถูกทดสอบ
pytest --cov=blog --cov-report=term-missing
```

ผลลัพธ์ตัวอย่าง:

```
Name                   Stmts   Miss  Cover   Missing
----------------------------------------------------
blog/models.py            28      2    93%   45-46
blog/forms.py              9      0   100%
blog/views.py              14      3    79%   22-24
----------------------------------------------------
TOTAL                      51      5    90%
```

Part 062 จะเจาะลึกเรื่อง coverage แบบเต็มรูปแบบ (รวมถึงการตั้งเป้า coverage
threshold และไฟล์ `.coveragerc`) — ในที่นี้ให้รู้จักไว้ก่อนว่ามันผนวกเข้ากับ pytest
ได้อย่างไร้รอยต่อ

### 609.2 `pytest-xdist` — รัน test แบบขนานหลาย process

```bash
pip install "pytest-xdist~=3.6"
```

```bash
# รันด้วยจำนวน worker เท่ากับจำนวน CPU core ที่มี
pytest -n auto

# รันด้วย 4 worker แบบเจาะจง
pytest -n 4
```

`pytest-django` ออกแบบมาให้ทำงานร่วมกับ `pytest-xdist` ได้ทันที โดยจะสร้าง
ฐานข้อมูลทดสอบแยกให้แต่ละ worker โดยอัตโนมัติ (ต่อท้ายชื่อฐานข้อมูลด้วย `_gw0`,
`_gw1`, ...) ทำให้ test ที่ใช้เวลารัน 2 นาทีบนเครื่องเดียว อาจเหลือ 30 วินาทีบนเครื่อง
8 core — ยิ่งมี test เยอะเท่าไหร่ ยิ่งคุ้มค่ามากขึ้นเท่านั้น

**ข้อควรระวัง**: test ที่มี **side effect ร่วมกัน** (เช่น เขียนไฟล์ไปยัง path ตายตัว
เดียวกัน หรือพึ่งพาลำดับการรันที่แน่นอน) อาจพังเมื่อรันแบบขนาน เพราะแต่ละ worker
ทำงานพร้อมกันโดยไม่รู้จักกัน จึงเป็นเหตุผลสำคัญที่ต้องเขียน test ให้ **independent**
จากกันเสมอ (หลักการเดียวกับที่เน้นย้ำเรื่อง fixture scope ในขั้นตอนที่ 603.3)

### 609.3 `pytest-mock` — mocking ที่ integrate เข้ากับระบบ fixture

```bash
pip install "pytest-mock~=3.14"
```

`pytest-mock` ห่อ `unittest.mock` มาตรฐานให้ใช้งานผ่าน fixture ชื่อ `mocker` แทนการ
เขียน `@patch` decorator หรือ `with patch(...):` เอง:

```python
# สมมติว่า blog/services.py มีฟังก์ชัน send_notification_email
def test_publish_post_sends_notification(mocker, published_post):
    mock_send = mocker.patch("blog.services.send_notification_email")

    from blog.services import publish_post
    publish_post(published_post)

    mock_send.assert_called_once_with(published_post)
```

ข้อดีของ `mocker` เทียบกับ `unittest.mock.patch` ตรง ๆ คือ pytest จะ **cleanup ให้
อัตโนมัติ** หลัง test จบเสมอ ไม่ต้องกังวลเรื่องลืม `.stop()` เหมือนเขียน patch เอง
Part 062 จะสอนเรื่อง Mocking แบบละเอียดกว่านี้มาก ทั้ง `unittest.mock` ล้วน ๆ และ
`pytest-mock` ควบคู่กัน

### 609.4 ตารางสรุป plugin ที่ควรรู้จักเพิ่มเติม

| Plugin | หน้าที่ | จะเรียนละเอียดที่ Part |
|---|---|---|
| `pytest-django` | เชื่อม pytest เข้ากับ Django (ติดตั้งไปแล้วในขั้นตอนที่ 602) | Part นี้ |
| `pytest-cov` | วัด code coverage | Part 062 |
| `pytest-xdist` | รัน test แบบขนาน | Part นี้ (609.2) |
| `pytest-mock` | mocking ผ่าน fixture `mocker` | Part 062 |
| `pytest-factoryboy` | แปลง `factory_boy` factory ให้กลายเป็น fixture อัตโนมัติ | Part 063 |
| `pytest-sugar` | แสดงผลลัพธ์ progress bar และสีสันสวยงามระหว่างรัน | ใช้ได้ทันทีหลังติดตั้ง ไม่ต้อง config |
| `pytest-randomly` | สุ่มลำดับการรัน test ทุกครั้ง เพื่อจับ test ที่แอบพึ่งพาลำดับกัน | ใช้ได้ทันทีหลังติดตั้ง |
| `pytest-env` | ตั้งค่า environment variable ให้ test ผ่าน config file | ใช้เมื่อจัดการ secret/config หลาย environment |

### 609.5 ตัวอย่าง config รวม plugin ทั้งหมดใน `pyproject.toml`

```toml
# pyproject.toml
[tool.pytest.ini_options]
DJANGO_SETTINGS_MODULE = "config.settings"
python_files = ["tests.py", "test_*.py", "*_tests.py"]
testpaths = ["blog"]
addopts = "--reuse-db -v --cov=blog --cov-report=term-missing"

[tool.coverage.run]
source = ["blog"]
omit = ["*/migrations/*", "*/tests/*"]
```

```bash
# หลังตั้งค่านี้ พิมพ์ pytest เฉย ๆ ก็จะรัน test พร้อมวัด coverage ให้ทันที
pytest

# เพิ่มการรันขนานเมื่อต้องการความเร็ว
pytest -n auto
```

---

## ขั้นตอนที่ 610: สรุปและแบบฝึกหัด

### 610.1 สรุปสิ่งที่ได้เรียนรู้ใน Part นี้

- ✅ เข้าใจความแตกต่างเชิงปรัชญาระหว่าง `unittest` (class-based) กับ `pytest`
  (function-based + fixture) และรู้ว่าทั้งสองแบบอยู่ร่วมกันในโปรเจกต์เดียวได้
- ✅ ติดตั้ง `pytest` และ `pytest-django` ตั้งค่า `pytest.ini`/`pyproject.toml`
  และเข้าใจความหมายของทุกค่าที่ตั้ง
- ✅ เขียน fixture พื้นฐานด้วย `@pytest.fixture` เข้าใจเรื่อง scope และ
  yield fixture สำหรับ setup/teardown
- ✅ เข้าใจกลไก database isolation ของ `pytest-django` ผ่าน `db` fixture และ
  `@pytest.mark.django_db` รวมถึง `transactional_db` และ `reset_sequences`
- ✅ ใช้ `@pytest.mark.parametrize` ลดโค้ดซ้ำซ้อน ทั้งแบบพารามิเตอร์เดียวและ
  แบบ matrix (stack หลาย parametrize)
- ✅ จัดระเบียบ fixture ด้วย `conftest.py` หลายระดับ (root vs. ต่อแอป) และรู้จัก
  `autouse=True`
- ✅ ใช้ fixture พิเศษของ pytest-django: `client`, `admin_client`, `admin_user`,
  `rf`, `django_user_model`
- ✅ แปลง test suite ของแอป `blog` จาก TestCase style เป็น pytest function-style
  ครบทั้ง model, view, และ form test
- ✅ รู้จัก plugin สำคัญ `pytest-cov`, `pytest-xdist`, `pytest-mock` และ plugin
  เสริมอื่น ๆ ที่ทีมมืออาชีพใช้จริง

### 610.2 Checklist ก่อนไป Part ถัดไป

- [ ] ติดตั้ง `pytest` และ `pytest-django` ใน venv สำเร็จ และรัน `pytest --version` ได้
- [ ] มีไฟล์ `pytest.ini` หรือ `[tool.pytest.ini_options]` ใน `pyproject.toml` ที่ตั้งค่า
      `DJANGO_SETTINGS_MODULE` ถูกต้อง
- [ ] รัน `pytest` แล้ว test เดิมจาก Part 059-060 (สไตล์ `TestCase`) ผ่านทั้งหมดโดยไม่
      ต้องแก้โค้ด
- [ ] เขียน fixture อย่างน้อย 3 ตัวใน `conftest.py` (root และ/หรือระดับแอป)
- [ ] เขียน test ที่ใช้ `@pytest.mark.django_db` และ test ที่ใช้ fixture `db` ได้ทั้ง
      สองแบบ
- [ ] เขียน `@pytest.mark.parametrize` อย่างน้อย 1 จุดพร้อม `ids` ที่อ่านง่าย
- [ ] แปลง `PostModelTest`, `PostListViewTest`, `PostFormTest` จาก Part 060 ให้เป็น
      pytest function-style ครบทั้ง 3 กลุ่ม
- [ ] ติดตั้งและลองใช้ `pytest-cov` (`pytest --cov=blog`) และ `pytest-xdist`
      (`pytest -n auto`) สำเร็จอย่างน้อยครั้งละหนึ่งคำสั่ง

### 610.3 แบบฝึกหัดท้ายบท

**แบบฝึกหัดที่ 1**: เปิดไฟล์ `blog/tests.py` ที่คุณเขียนไว้ใน Part 059-060 (หรือ
ไฟล์ตัวอย่างในขั้นตอนที่ 601.1 ของ Part นี้ถ้ายังไม่มี) แปลง `PostModelTest` ทั้ง
class ให้เป็นไฟล์ `blog/tests/test_models.py` แบบ pytest function-style โดยต้องมี
fixture อย่างน้อย 1 ตัว และใช้ `@pytest.mark.parametrize` อย่างน้อย 1 จุด (เช่น
ทดสอบว่าทุกค่าใน `Post.ContentType` บันทึกลงฐานข้อมูลได้ถูกต้อง)

**แบบฝึกหัดที่ 2**: สร้างไฟล์ `pytest.ini` (หรือปรับ `pyproject.toml`) ให้โปรเจกต์
ของคุณเอง ตั้งค่า `addopts` ให้รวม `--reuse-db -v` แล้วรัน `pytest` ยืนยันว่า test
ทั้งหมดผ่าน จากนั้นลองลบ `DJANGO_SETTINGS_MODULE` ออกชั่วคราวเพื่อดู error message
ที่เกิดขึ้น แล้วอธิบายด้วยคำพูดตัวเองว่า error นั้นบอกอะไร ก่อนจะใส่ค่ากลับคืน

**แบบฝึกหัดที่ 3**: เขียน `conftest.py` สองระดับให้โปรเจกต์ของคุณ — ระดับ root
ให้มี fixture `staff_user`/`regular_user` (ใช้ `django_user_model`) และระดับแอป
`blog` ให้มี fixture `published_post`/`draft_post` พร้อม autouse fixture อย่างน้อย
1 ตัว (เช่น เคลียร์ cache ก่อน-หลังทุก test) แล้วเขียน test ยืนยันว่า fixture ทั้งหมด
ทำงานถูกต้องผ่าน `client` fixture

**แบบฝึกหัดที่ 4 (ขั้นสูง)**: ติดตั้ง `pytest-cov` และ `pytest-xdist` แล้วรัน
`pytest --cov=blog --cov-report=term-missing -n auto` บันทึกผลลัพธ์ coverage
percentage ที่ได้ ระบุว่าไฟล์ไหนมี coverage ต่ำที่สุดในแอป `blog` และอธิบายว่า
เพราะเหตุใด (เช่น branch ของ error handling ที่ยังไม่มี test ครอบคลุม) พร้อมเขียน
test เพิ่มอย่างน้อย 1 ตัวเพื่อเพิ่ม coverage ของไฟล์นั้น

### 610.4 คำถามที่พบบ่อย (FAQ)

**Q: ต้องลบไฟล์ `blog/tests.py` แบบ unittest ทิ้งทันทีหรือไม่?**
A: ไม่จำเป็นเลย อย่างที่อธิบายในขั้นตอนที่ 601.6 pytest รัน `unittest.TestCase`
ได้โดยตรงอยู่แล้ว ทีมมืออาชีพจำนวนมากปล่อยให้ test เก่าอยู่ในรูปแบบ `TestCase`
ต่อไปเรื่อย ๆ และเขียน test ใหม่เป็น pytest style เท่านั้น ค่อย ๆ แปลงของเก่าเมื่อ
มีเวลาหรือเมื่อต้องแก้ไฟล์นั้นอยู่แล้ว (ไม่ต้อง refactor เพื่อ refactor)

**Q: ใช้ `python manage.py test` ต่อไปได้ไหม หลังติดตั้ง pytest แล้ว?**
A: ได้ครับ/ค่ะ ทั้งสองคำสั่งอยู่ร่วมกันได้ในโปรเจกต์เดียวกันโดยไม่ชนกัน แต่เมื่อ
ทีมตัดสินใจใช้ pytest เป็นหลักแล้ว ควรกำหนดให้ CI/CD pipeline (จะเรียนใน Part 088)
รันผ่าน `pytest` เท่านั้น เพื่อให้ผลลัพธ์ coverage และ report เป็นมาตรฐานเดียวกัน
ทั้งทีม ไม่ปะปนกันระหว่างสอง test runner

**Q: ทำไม test ที่ใช้ `client` fixture บางตัวยังต้องใส่ `@pytest.mark.django_db`
อยู่ ทั้งที่ `client` ก็ยิงไป view ที่แตะฐานข้อมูลอยู่แล้ว?**
A: fixture `client` เองไม่ได้เปิดสิทธิ์เข้าถึงฐานข้อมูลให้กับ **ตัว test function**
โดยอัตโนมัติ มันแค่ทำให้ยิง HTTP request ได้ ถ้า test ของคุณสร้างข้อมูลด้วย
`Model.objects.create(...)` ก่อนเรียก `client.get(...)` (เหมือนตัวอย่างในขั้นตอนที่
607.2) ตัว test function เองก็ยังต้องขอสิทธิ์ผ่าน `db`/`django_db` เช่นเดิม ส่วนถ้า
view ที่ถูกเรียกไปแตะฐานข้อมูลแต่ตัว test เองไม่ได้แตะโดยตรง (ไม่มีการสร้าง object
ก่อน) จะไม่มี error เพราะ `pytest-django` เปิด database access ให้ทั้ง request-response
cycle เมื่อเห็น marker `django_db` อยู่แล้วในระดับ test

**Q: ควรใช้ `--no-migrations` เป็นค่า default ใน `addopts` เลยหรือไม่ เพื่อความเร็ว?**
A: ไม่แนะนำให้ตั้งเป็นค่า default ถาวร เพราะจะทำให้คุณพลาดจับบั๊กที่เกิดจาก
migration เขียนผิด (เช่น ลืมใส่ default value ให้ field ใหม่) ควรใช้ `--reuse-db`
เป็นค่า default เพื่อความเร็วระหว่างพัฒนา แล้วรันแบบเต็ม (ไม่มี flag พิเศษ หรือ
`--create-db`) อย่างน้อยก่อน commit และบน CI ทุกครั้งเสมอ

**Q: pytest-xdist ทำให้ test ที่เคยผ่านกลับ fail แบบสุ่ม ๆ ควรทำอย่างไร?**
A: นี่คือสัญญาณว่า test ของคุณมี **hidden dependency** ระหว่างกัน (เช่น ใช้
ตัวแปร global ร่วมกัน, เขียนไฟล์ไป path เดียวกัน, หรือพึ่งพาลำดับการรัน) ให้ลองรัน
`pytest -p no:randomly` เทียบกับ `pytest-randomly` เปิดใช้งาน เพื่อยืนยันว่าปัญหา
มาจากลำดับจริงหรือไม่ แล้วแก้ที่ root cause (ทำให้แต่ละ test สร้างข้อมูลของตัวเอง
ผ่าน fixture เสมอ ไม่แชร์ state ข้าม test) — อย่าปิด `-n auto` ทิ้งเพื่อหนีปัญหา
เพราะบั๊กแบบนี้จะโผล่มาอีกในระบบ production ที่รันแบบ concurrent จริง

---

## เตรียมตัวสำหรับ Part ถัดไป

**Part 062: Test Coverage และ Mocking** จะพาคุณเจาะลึกสองเรื่องที่แตะไว้เพียงผิวเผิน
ใน Part นี้: การตั้งเป้า **code coverage** อย่างมีความหมาย (ไม่ใช่ไล่ตัวเลข 100%
แบบไม่มีเหตุผล) การตั้งค่า `.coveragerc`/`[tool.coverage.*]` แบบมืออาชีพ และการ
**Mock** external dependency อย่างถูกต้อง ทั้งการเรียก API ภายนอก, การส่งอีเมล, และ
การจำลองเวลา (`freezegun`) เพื่อทดสอบ logic ที่ขึ้นกับวันที่/เวลาได้อย่างแม่นยำและ
รวดเร็ว โดยจะต่อยอดจาก fixture และ `pytest-mock` ที่คุณเพิ่งรู้จักในขั้นตอนที่ 609
ของ Part นี้โดยตรง

เตรียม `blog/services.py` ที่มีฟังก์ชันเรียก external service (เช่น ส่งอีเมลแจ้งเตือน
เมื่อมีบทความใหม่) ไว้ให้พร้อม เพราะเราจะใช้มันเป็นกรณีศึกษาหลักตลอด Part 062
