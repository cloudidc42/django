# Part 062: Test Coverage และ Mocking

> **ขั้นตอนที่ 611-620 ของหลักสูตร** | Phase 7: Testing & Quality Assurance
>
> เป้าหมายของ Part นี้: รู้ว่า test suite ของคุณ "ครอบคลุม" โค้ดจริงมากแค่ไหนด้วย
> `coverage.py`, ตั้งค่า `.coveragerc` ให้รายงานที่ได้มีความหมาย, ผูก coverage
> เข้ากับ CI แบบมืออาชีพโดยไม่ตกหลุมพราง "coverage สูง = test ดี", และเรียนรู้
> `unittest.mock` อย่างลึกซึ้ง — `Mock`, `MagicMock`, `patch`, spy pattern —
> เพื่อแยกโค้ดของคุณออกจากบริการภายนอกที่ควบคุมไม่ได้ เช่น การส่งอีเมลจริงหรือ
> การเรียก Have I Been Pwned API จาก Part 035 เมื่อจบ Part นี้ คุณจะเขียน test
> ที่รันเร็ว เชื่อถือได้ ไม่พึ่งพาเครือข่ายภายนอก และยังคงทดสอบ "พฤติกรรมจริง"
> ของระบบได้อย่างมีความหมาย ไม่ใช่แค่ทำให้ตัวเลข coverage สวยงามบนหน้าจอ

---

## สารบัญของ Part นี้

- ขั้นตอนที่ 611: ติดตั้งและใช้ `coverage.py` วัด Test Coverage ของโปรเจกต์
- ขั้นตอนที่ 612: อ่าน Coverage Report, ตั้งค่า `.coveragerc` (exclude migrations, `__init__.py`)
- ขั้นตอนที่ 613: ตั้งเป้าหมาย Coverage ใน CI (`--fail-under=80`) และข้อควรระวังว่า coverage สูงไม่ได้แปลว่า test ดี
- ขั้นตอนที่ 614: `unittest.mock` เบื้องต้น — `Mock`, `MagicMock`
- ขั้นตอนที่ 615: `patch` decorator/context manager — วิธี mock ฟังก์ชัน/method ที่ถูกต้อง ("patch where it's used")
- ขั้นตอนที่ 616: Mock การเรียก External API จริง (เช่น การส่งอีเมล, การเรียก Have I Been Pwned จาก Part 035)
- ขั้นตอนที่ 617: Mock เวลา ด้วย `freezegun` (ทดสอบ logic ที่ผูกกับวันที่/เวลา)
- ขั้นตอนที่ 618: Spy pattern — `assert_called_with()`, `assert_called_once()`, `call_count`
- ขั้นตอนที่ 619: ข้อควรระวังการ Mock มากเกินไป (test ที่ mock ทุกอย่างจนไม่ทดสอบอะไรจริง — "test smell")
- ขั้นตอนที่ 620: สรุปและแบบฝึกหัด — เขียน test ที่ mock การส่งอีเมลและการเรียก API ภายนอกจริง

---

## ขั้นตอนที่ 611: ติดตั้งและใช้ `coverage.py` วัด Test Coverage ของโปรเจกต์

### 611.1 Test Coverage คืออะไร และทำไมต้องวัด

ตั้งแต่ Part 059-061 คุณเขียน test ไปแล้วจำนวนมากทั้งด้วย `unittest`/Django
`TestCase` และ `pytest-django` แต่คำถามที่ยังไม่มีคำตอบชัดเจนคือ **"โค้ดกี่
เปอร์เซ็นต์ของโปรเจกต์ที่ test suite เคยรันผ่านจริง ๆ บ้าง"**

**Test Coverage** คือตัวชี้วัดที่บอกว่า เมื่อรัน test suite ทั้งหมดแล้ว บรรทัดโค้ด
(statement), เงื่อนไข (branch), หรือฟังก์ชันใดบ้างที่ **ถูกรันจริงอย่างน้อยหนึ่งครั้ง**
ระหว่างการทดสอบ ส่วนที่ไม่เคยถูกรันเลยเรียกว่า **uncovered code** ซึ่งอาจซ่อนบั๊ก
ที่ไม่มีใครเคยทดสอบมาก่อนเลย

**เครื่องมือมาตรฐานของวงการ Python** สำหรับวัด coverage คือแพ็กเกจ
[`coverage.py`](https://coverage.readthedocs.io/) ซึ่งทำงานโดยการ "แอบดู"
ทุกบรรทัดที่ interpreter รันระหว่าง test แล้วสรุปเป็นรายงาน

> **ข้อควรจำใจไว้ตั้งแต่ต้น Part นี้**: coverage บอกได้แค่ว่า "บรรทัดนี้ถูกรัน
> หรือไม่" มันไม่รู้เลยว่า test ที่รันผ่านบรรทัดนั้น **ตรวจสอบผลลัพธ์ถูกต้อง
> หรือไม่** — ประเด็นนี้จะอธิบายละเอียดในขั้นตอนที่ 613

### 611.2 ติดตั้ง `coverage.py` และ `pytest-cov`

โปรเจกต์ `blog` ของเราใช้ `pytest-django` เป็นตัวรัน test หลักตั้งแต่ Part 061
ดังนั้นแนะนำให้ติดตั้งทั้ง `coverage` และปลั๊กอิน `pytest-cov` ที่เชื่อม
`coverage.py` เข้ากับ `pytest` โดยตรง:

```bash
pip install coverage pytest-cov
pip freeze > requirements-dev.txt
```

ตรวจสอบเวอร์ชัน:

```bash
coverage --version
pytest --version
```

### 611.3 วิธีที่ 1: รันผ่าน `coverage` ตรง ๆ (ใช้ได้กับ `pytest` หรือ `manage.py test`)

```bash
# ห่อคำสั่งรัน test ปกติด้วย "coverage run"
coverage run -m pytest

# หรือถ้ายังใช้ manage.py test อยู่ (Part 059-060)
coverage run manage.py test
```

`coverage.py` จะบันทึกข้อมูลไว้ในไฟล์ `.coverage` (ไฟล์ binary ที่ไม่ควร commit
เข้า Git — ต้องเพิ่มใน `.gitignore`) จากนั้นดูรายงานสรุปด้วย:

```bash
coverage report
```

ตัวอย่างผลลัพธ์:

```
Name                              Stmts   Miss  Cover
-----------------------------------------------------
accounts/models.py                   28      2    93%
accounts/services.py                 12      4    67%
accounts/validators.py               45      6    87%
accounts/views.py                    64     18    72%
blog/models.py                       35      0   100%
blog/views.py                        88     22    75%
config/settings.py                   40      0   100%
-----------------------------------------------------
TOTAL                                312     52    83%
```

### 611.4 วิธีที่ 2: ใช้ `pytest-cov` (แนะนำสำหรับโปรเจกต์นี้)

`pytest-cov` ทำให้ไม่ต้องพิมพ์ `coverage run` แยกต่างหาก และรองรับ flag เพิ่มเติม
ที่มีประโยชน์มาก:

```bash
pytest --cov=.
```

แสดงผลแบบละเอียดกว่าเดิมพร้อมบอกว่า "บรรทัดไหนที่ยังไม่ถูกทดสอบ" ด้วย
`--cov-report=term-missing`:

```bash
pytest --cov=. --cov-report=term-missing
```

```
Name                              Stmts   Miss  Cover   Missing
---------------------------------------------------------------
accounts/services.py                 12      4    67%   18-21
accounts/views.py                    64     18    72%   45-50, 88-95
blog/views.py                        88     22    75%   102-110, 140-152
---------------------------------------------------------------
TOTAL                                312     52    83%
```

คอลัมน์ `Missing` บอกเลขบรรทัดที่ยังไม่มี test เคยรันผ่านเลย — เป็นจุดเริ่มต้น
ที่ดีที่สุดในการตัดสินใจว่าจะเขียน test เพิ่มตรงไหนก่อน

### 611.5 บันทึกค่าเริ่มต้นไว้ใน `pyproject.toml` ของโปรเจกต์

เพื่อไม่ต้องพิมพ์ flag ยาว ๆ ทุกครั้ง ให้ตั้งค่า `addopts` ต่อจาก config ของ
`pytest-django` ที่สร้างไว้ตั้งแต่ Part 061:

```toml
# pyproject.toml
[tool.pytest.ini_options]
DJANGO_SETTINGS_MODULE = "config.settings"
python_files = ["tests.py", "test_*.py", "*_tests.py"]
addopts = "--cov=. --cov-report=term-missing"
```

ตอนนี้พิมพ์แค่ `pytest` เฉย ๆ ก็จะได้รายงาน coverage ติดมาด้วยทุกครั้ง

### 611.6 สร้างรายงาน HTML ที่อ่านง่ายกว่า Terminal

Terminal report เหมาะกับการดูภาพรวมเร็ว ๆ แต่เมื่อต้องไล่ดูว่า "บรรทัดไหนกันแน่"
ที่ยังไม่ถูกทดสอบ รายงานแบบ HTML สะดวกกว่ามาก:

```bash
coverage html
# หรือถ้าใช้ pytest-cov
pytest --cov=. --cov-report=html
```

คำสั่งนี้สร้างโฟลเดอร์ `htmlcov/` ขึ้นมา เปิดไฟล์ `htmlcov/index.html` ด้วยเบราว์เซอร์
จะเห็นตารางไฟล์ทั้งหมดพร้อมเปอร์เซ็นต์ และเมื่อคลิกเข้าไปในไฟล์ใดไฟล์หนึ่ง จะเห็น
โค้ดจริงพร้อมพื้นหลังสี **เขียว = ถูกทดสอบ**, **แดง = ไม่เคยถูกรันเลย**,
**เหลือง = branch ถูกรันแค่บางทาง** (จะอธิบายละเอียดในขั้นตอนที่ 612.2)

```bash
# .gitignore (เพิ่มต่อจากที่มีอยู่แล้วตั้งแต่ Part 001)
.coverage
htmlcov/
.pytest_cache/
```

---

## ขั้นตอนที่ 612: อ่าน Coverage Report, ตั้งค่า `.coveragerc` (exclude migrations, `__init__.py`)

### 612.1 อ่านตัวเลขในรายงานให้เข้าใจจริง

| คอลัมน์ | ความหมาย |
|---|---|
| `Stmts` | จำนวน "statement" (บรรทัดที่รันได้จริง ไม่นับ comment/blank line) ทั้งหมดในไฟล์ |
| `Miss` | จำนวน statement ที่ **ไม่เคย** ถูกรันเลยระหว่าง test |
| `Cover` | เปอร์เซ็นต์ `(Stmts - Miss) / Stmts * 100` |
| `Missing` | เลขบรรทัด (หรือช่วงบรรทัด) ที่ยังไม่ถูกทดสอบ |
| `Branch` (ถ้าเปิด `--branch`) | จำนวนจุดแตกกิ่ง (if/else, for, while) ทั้งหมด |
| `BrPart` | จำนวนกิ่งที่ถูกรัน "แค่บางทาง" เช่น `if` เคยเป็นจริงแต่ไม่เคยเป็นเท็จเลย |

### 612.2 Statement Coverage vs Branch Coverage — ทำไมแค่ Statement ไม่พอ

ปัญหาคลาสสิกของ statement coverage คือมันหลอกตาได้ง่าย ลองดูฟังก์ชันนี้:

```python
# accounts/services.py
def get_discount_rate(user):
    if user.is_premium:
        return 0.20
    return 0.0
```

ถ้า test เรียก `get_discount_rate(premium_user)` เพียงครั้งเดียว **ทุกบรรทัด**
ในฟังก์ชันนี้ถูกรันครบ (statement coverage = 100%) แต่ path ที่ `user.is_premium`
เป็น `False` **ไม่เคยถูกทดสอบเลย** — ถ้ามีบั๊กในเงื่อนไขนั้นก็จะไม่มีใครรู้

เปิด **branch coverage** เพื่อจับปัญหานี้:

```bash
coverage run --branch -m pytest
coverage report -m
```

```toml
# pyproject.toml
[tool.pytest.ini_options]
addopts = "--cov=. --cov-branch --cov-report=term-missing"
```

ตอนนี้ถ้า test ยังไม่เคยเรียกด้วย `user.is_premium == False` รายงานจะแจ้งเตือน
บรรทัด `if user.is_premium:` ว่าเป็น **partial branch** ทันที แม้ statement
coverage จะเป็น 100% ก็ตาม

### 612.3 สร้างไฟล์ `.coveragerc` เพื่อกรอง Noise ออกจากรายงาน

ถ้าไม่ตั้งค่าอะไรเลย รายงาน coverage จะรวมไฟล์ที่ **ไม่มีความหมายที่จะวัด** เข้าไปด้วย
เช่น migration files (สร้างอัตโนมัติ ไม่ควรต้องเขียน test แยกเพื่อ "ทดสอบ" มัน),
`__init__.py` ที่มักว่างเปล่า, หรือไฟล์ config ที่ import ตอน startup เท่านั้น
ทำให้ตัวเลข coverage รวมดูต่ำกว่าความเป็นจริงโดยไม่มีประโยชน์

สร้างไฟล์ `.coveragerc` ที่ root ของโปรเจกต์ (ระดับเดียวกับ `manage.py`):

```ini
; .coveragerc
[run]
source = .
branch = True
omit =
    */migrations/*
    */tests/*
    */test_*.py
    manage.py
    config/asgi.py
    config/wsgi.py
    venv/*
    */__init__.py
    */admin.py
    */apps.py

[report]
show_missing = True
skip_covered = False
precision = 1
exclude_lines =
    pragma: no cover
    def __repr__
    def __str__
    raise NotImplementedError
    if TYPE_CHECKING:
    if __name__ == .__main__.:
    class .*\bProtocol\):
    @(abc\.)?abstractmethod

[html]
directory = htmlcov
```

อธิบายทีละส่วน:

| Section | Key | ความหมาย |
|---|---|---|
| `[run]` | `source` | โฟลเดอร์/แพ็กเกจที่ต้องการวัด coverage (ปกติคือ root โปรเจกต์) |
| `[run]` | `branch` | เปิด branch coverage เป็นค่าเริ่มต้นโดยไม่ต้องพิมพ์ `--branch` ทุกครั้ง |
| `[run]` | `omit` | รายการ pattern ของไฟล์ที่ **ไม่นับ** เข้ารายงานเลย |
| `[report]` | `show_missing` | แสดงคอลัมน์ `Missing` เสมอเมื่อรัน `coverage report` |
| `[report]` | `exclude_lines` | pattern regex ของ**บรรทัดโค้ด**ที่ไม่นับว่าต้องถูกทดสอบ แม้อยู่ในไฟล์ที่วัด |

### 612.4 ทำไมต้อง `omit` migrations และ `__init__.py`

- **`*/migrations/*`**: ไฟล์ migration ถูกสร้างอัตโนมัติจาก `makemigrations`
  (Part 013) มันคือ "คำสั่งเปลี่ยนแปลงฐานข้อมูล" ไม่ใช่ business logic ที่ควร
  ต้องมี unit test แยก — การเขียน test ให้ migration แทบไม่มีประโยชน์และเสียเวลา
- **`*/__init__.py`**: ส่วนใหญ่เป็นไฟล์ว่างเปล่าหรือมีแค่ import สำหรับจัดระเบียบ
  package ไม่มี logic ที่ต้องทดสอบ
- **`manage.py`, `config/asgi.py`, `config/wsgi.py`**: เป็นไฟล์ entry point ที่รัน
  ตอน startup เท่านั้น ไม่มี business logic ให้ทดสอบเชิงหน่วย (unit) การทดสอบว่า
  "แอปรันได้จริง" ควรเป็นหน้าที่ของ integration/smoke test แทน (Part 064)
- **`*/admin.py`, `*/apps.py`**: ส่วนใหญ่เป็นการลงทะเบียน config เชิงประกาศ
  (declarative) ล้วน ๆ — ถ้ามี logic ซับซ้อนจริงจังใน `admin.py` (เช่น custom
  `get_queryset()`) ทีมส่วนใหญ่จะแยกออกไปเป็นฟังก์ชันแยกในไฟล์อื่นแล้วทดสอบตรงนั้น
  แทนเพื่อไม่ต้องพึ่ง Django Admin ทั้งระบบ

### 612.5 ใช้ `# pragma: no cover` สำหรับบรรทัดที่ตั้งใจไม่ทดสอบ

บางบรรทัดในโค้ด "ควร" ไม่ถูกทดสอบด้วยเหตุผลที่ดี เช่น branch ป้องกันเหตุการณ์ที่
แทบเป็นไปไม่ได้ในทางปฏิบัติ:

```python
# blog/services.py
def get_thumbnail_url(post):
    if post.cover_image:
        return post.cover_image.url
    else:  # pragma: no cover
        # เกิดขึ้นได้ทางทฤษฎีเท่านั้น เพราะ form บังคับ cover_image เสมอ
        # แต่เก็บ branch นี้ไว้เผื่อมีการเรียกจาก management command ในอนาคต
        return settings.DEFAULT_THUMBNAIL_URL
```

`coverage.py` จะข้ามบรรทัดที่มี comment `# pragma: no cover` (หรือทั้ง block
ถ้าอยู่ท้ายบรรทัด `if`/`def`) ไปจากการคำนวณเปอร์เซ็นต์ทันที **ใช้อย่างระมัดระวัง**
— นี่ไม่ใช่เครื่องมือ "แต่งตัวเลขให้สวย" แต่เป็นการสื่อสารกับทีมว่า "จุดนี้ตั้งใจ
ไม่ทดสอบ เพราะเหตุผลที่ระบุไว้ชัดเจน"

### 612.6 ตั้งค่าแบบเดียวกันใน `pyproject.toml` (ทางเลือกสมัยใหม่)

ถ้าต้องการรวม config ทุกอย่างไว้ในไฟล์เดียวแทนการมีทั้ง `.coveragerc` และ
`pyproject.toml` แยกกัน `coverage.py` ตั้งแต่เวอร์ชัน 5 ขึ้นไปอ่าน section
`[tool.coverage.*]` จาก `pyproject.toml` ได้โดยตรง (มีผลเทียบเท่ากับ `.coveragerc`
ทุกประการ — เลือกใช้แบบใดแบบหนึ่งเท่านั้น ห้ามมีทั้งสองไฟล์พร้อมกันเพราะจะสับสน
ว่าตัวไหนมีผลจริง):

```toml
# pyproject.toml
[tool.coverage.run]
source = ["."]
branch = true
omit = [
    "*/migrations/*",
    "*/tests/*",
    "manage.py",
    "config/asgi.py",
    "config/wsgi.py",
    "*/__init__.py",
]

[tool.coverage.report]
show_missing = true
precision = 1
exclude_lines = [
    "pragma: no cover",
    "raise NotImplementedError",
    "if TYPE_CHECKING:",
]

[tool.coverage.html]
directory = "htmlcov"
```

---

## ขั้นตอนที่ 613: ตั้งเป้าหมาย Coverage ใน CI (`--fail-under=80`) และข้อควรระวังว่า coverage สูงไม่ได้แปลว่า test ดี

### 613.1 บังคับเปอร์เซ็นต์ขั้นต่ำด้วย `--cov-fail-under`

เพื่อป้องกันไม่ให้ coverage ของโปรเจกต์**ลดลง**เรื่อย ๆ เมื่อทีมงานเพิ่มขึ้นและ
บางคนลืมเขียน test ให้โค้ดใหม่ สามารถบังคับให้ CI **fail ทันที** ถ้า coverage
รวมต่ำกว่าเกณฑ์ที่กำหนด:

```bash
pytest --cov=. --cov-report=term-missing --cov-fail-under=80
```

ถ้า coverage รวมต่ำกว่า 80% คำสั่งนี้จะจบด้วย exit code ที่ไม่ใช่ 0 ทันที
(แม้ทุก test จะผ่านหมดก็ตาม) ทำให้ CI pipeline หยุดและแจ้งเตือนทีม

หรือกำหนดผ่าน `.coveragerc`/`pyproject.toml` แทนเพื่อไม่ต้องพิมพ์ flag ทุกครั้ง:

```ini
; .coveragerc
[report]
fail_under = 80
```

### 613.2 ตัวอย่าง GitHub Actions Workflow ที่บังคับ Coverage Gate

```yaml
# .github/workflows/tests.yml
name: Tests

on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: "3.12"

      - name: Install dependencies
        run: |
          python -m pip install --upgrade pip
          pip install -r requirements.txt
          pip install -r requirements-dev.txt

      - name: Run tests with coverage gate
        run: |
          pytest --cov=. --cov-report=term-missing --cov-report=xml --cov-fail-under=80
        env:
          DJANGO_SETTINGS_MODULE: config.settings
          SECRET_KEY: test-secret-key-not-for-production
```

> **หมายเหตุ**: การตั้งค่า CI/CD แบบเต็มรูปแบบ รวมถึง matrix testing หลายเวอร์ชัน
> Python/Django, caching dependency, และการ deploy อัตโนมัติ จะอยู่ใน
> **Part 088: CI/CD ด้วย GitHub Actions** เนื้อหาใน Part นี้แสดงแค่ส่วนที่
> เกี่ยวกับ coverage gate เท่านั้น

### 613.3 ควรตั้งเป้าไว้ที่กี่เปอร์เซ็นต์

| ช่วง Coverage | ความหมายในทางปฏิบัติ |
|---|---|
| ต่ำกว่า 50% | เสี่ยงสูงมาก โค้ดส่วนใหญ่ไม่มีใครรู้ว่าพังหรือไม่เมื่อแก้ไข |
| 50-70% | จุดเริ่มต้นที่ยอมรับได้สำหรับโปรเจกต์เก่าที่เพิ่งเริ่มเขียน test ย้อนหลัง |
| 70-85% | ระดับที่ทีม production ส่วนใหญ่ใช้เป็นเกณฑ์ CI จริง (เช่น `--fail-under=80` ในตัวอย่างข้างต้น) |
| 85-95% | ดีมาก เหมาะกับ core business logic หรือ library ที่คนอื่นนำไปใช้ต่อ |
| 100% | ไม่ใช่เป้าหมายที่สมเหตุสมผลเสมอไป — ดูขั้นตอนที่ 613.4-613.5 |

**ไม่มีตัวเลขวิเศษ**ที่ใช้ได้กับทุกโปรเจกต์ สิ่งสำคัญกว่าตัวเลขคือ **โค้ดส่วนที่
สำคัญที่สุด** (payment logic, authentication, permission checks) ต้องมี coverage
สูงเสมอ แม้ตัวเลขรวมทั้งโปรเจกต์จะยังไม่ถึง 100% ก็ตาม

### 613.4 กับดักอันตรายที่สุดของ Part นี้: Coverage สูง ≠ Test ดี

นี่คือประเด็นที่**สำคัญที่สุดใน Part นี้** และเป็นความเข้าใจผิดที่พบบ่อยมาก
แม้ในทีมที่มีประสบการณ์ ลองดูตัวอย่างนี้:

```python
# accounts/services.py
def calculate_shipping_cost(weight_kg, is_express):
    if weight_kg <= 0:
        raise ValueError("น้ำหนักต้องมากกว่า 0")
    base_cost = weight_kg * 15
    if is_express:
        base_cost *= 2
    return round(base_cost, 2)
```

```python
# accounts/tests.py — test ที่ทำให้ coverage เป็น 100% แต่ "ไม่ได้ทดสอบอะไรจริง"
def test_calculate_shipping_cost():
    calculate_shipping_cost(5, True)
    calculate_shipping_cost(5, False)
    # ไม่มี assert แม้แต่บรรทัดเดียว!
    # coverage นับว่าทุกบรรทัดถูกรันแล้ว = 100% แต่ test นี้จะ "ผ่าน" เสมอ
    # ไม่ว่าฟังก์ชันจะคำนวณผิดขนาดไหนก็ตาม (เช่น ลืม *2 ตอน is_express)
```

test ข้างต้น**ไม่มีค่าอะไรเลย**ในเชิงคุณภาพ แม้ coverage report จะบอกว่า
`calculate_shipping_cost` ถูกทดสอบครบ 100% ก็ตาม เพราะไม่มีการตรวจสอบว่า
**ผลลัพธ์ที่ได้ถูกต้องหรือไม่** ถ้ามีคนมาแก้บั๊ก (เช่น ลบ `base_cost *= 2` ทิ้งโดย
ไม่ได้ตั้งใจ) test นี้ก็ยังคง "ผ่าน" เหมือนเดิม

เวอร์ชันที่ถูกต้อง:

```python
# accounts/tests.py
import pytest
from accounts.services import calculate_shipping_cost


def test_calculate_shipping_cost_standard():
    assert calculate_shipping_cost(5, is_express=False) == 75.0


def test_calculate_shipping_cost_express_doubles_price():
    assert calculate_shipping_cost(5, is_express=True) == 150.0


def test_calculate_shipping_cost_rejects_zero_weight():
    with pytest.raises(ValueError, match="น้ำหนักต้องมากกว่า 0"):
        calculate_shipping_cost(0, is_express=False)


def test_calculate_shipping_cost_rejects_negative_weight():
    with pytest.raises(ValueError):
        calculate_shipping_cost(-1, is_express=False)
```

Coverage ของทั้งสองเวอร์ชันเท่ากันทุกประการ (100% เหมือนกัน) แต่เวอร์ชันที่สอง
**จะจับบั๊กได้จริง** ถ้ามีใครแก้โค้ดผิดพลาด ส่วนเวอร์ชันแรกจะไม่มีทางจับอะไรได้เลย

### 613.5 Test Smell อื่น ๆ ที่ทำให้ Coverage สูงแต่ไร้ประโยชน์

| Test Smell | ลักษณะ | ทำไมอันตราย |
|---|---|---|
| **Assertion-free test** | เรียกฟังก์ชันแต่ไม่มี `assert` เลย (ตัวอย่าง 613.4) | ผ่านเสมอไม่ว่าผลลัพธ์จะผิดแค่ไหน |
| **Testing the mock, not the code** | mock ทุกอย่างจนเหลือแค่ตรวจว่า mock ถูกเรียก (ขั้นตอนที่ 619) | ไม่ได้ยืนยันพฤติกรรมจริงของระบบ |
| **Happy-path only** | ทดสอบแค่กรณีที่ input ถูกต้องเสมอ ไม่เคยทดสอบ edge case/error | บั๊กที่เกิดกับ input แปลก ๆ จะหลุดรอดไปถึง production |
| **Tautological assertion** | `assert result == result` หรือ assert ค่าที่ copy มาจากโค้ดจริงเป๊ะ ๆ โดยไม่ได้คำนวณอิสระ | ไม่มีทางจับบั๊กได้เลยเพราะ assert สะท้อน implementation ไม่ใช่ requirement |

### 613.6 เกริ่นนำ Mutation Testing: เครื่องมือตรวจสอบว่า Test "จับบั๊กได้จริง" หรือไม่

ถ้าอยากรู้ว่า test suite ที่มีอยู่ "จับบั๊กได้จริง" แค่ไหน (ไม่ใช่แค่รันผ่าน
บรรทัดโค้ด) มีเทคนิคชื่อ **Mutation Testing**: เครื่องมือจะ "แก้โค้ดให้ผิด
โดยตั้งใจ" ทีละจุดเล็ก ๆ (เช่น เปลี่ยน `>` เป็น `>=`, เปลี่ยน `+` เป็น `-`)
เรียกว่า **mutant** แล้วรัน test suite ใหม่ ถ้า test suite ยัง "ผ่าน" ทั้งที่
โค้ดผิดไปแล้ว แปลว่า test นั้น**ไม่ได้ทดสอบจุดนั้นจริง** (mutant "รอดชีวิต" —
survived mutant)

```bash
pip install mutmut
mutmut run --paths-to-mutate=accounts/services.py
mutmut results
```

Mutation testing ใช้เวลารันนานกว่า coverage ธรรมดามาก (ต้องรัน test suite ซ้ำ
หลายร้อยหลายพันครั้ง) จึงไม่เหมาะรันทุกครั้งใน CI ปกติ แต่เป็นเครื่องมือที่
ยอดเยี่ยมสำหรับตรวจสุขภาพ test suite เป็นระยะ ๆ เพื่อยืนยันว่า coverage สูง
ที่มีอยู่นั้น "มีความหมายจริง" ไม่ใช่แค่ assertion-free test ตามขั้นตอนที่ 613.4

---

## ขั้นตอนที่ 614: `unittest.mock` เบื้องต้น — `Mock`, `MagicMock`

### 614.1 ทำไมต้อง Mock

ระบบจริงมักมี **dependency ภายนอก** ที่เราไม่อยากให้ test ต้องพึ่งพาโดยตรง เช่น:

- การส่งอีเมลจริง (ช้า, ต้องมี SMTP server, อาจโดน rate limit)
- การเรียก HTTP API ภายนอก เช่น Have I Been Pwned จาก Part 035 (ต้องมีอินเทอร์เน็ต,
  อาจล่ม, ทำให้ test suite ช้าและไม่เสถียร — **flaky test**)
- นาฬิกาของระบบ (`timezone.now()`) ที่เปลี่ยนค่าไปเรื่อย ๆ ทำให้ test ที่ผูกกับเวลา
  ผลลัพธ์ไม่คงที่
- Payment gateway, SMS gateway, third-party service ใด ๆ ที่มีค่าใช้จ่ายจริงต่อ
  การเรียกแต่ละครั้ง

**Mocking** คือเทคนิคการ "แทนที่" object หรือฟังก์ชันจริงด้วย object ปลอมที่เรา
ควบคุมพฤติกรรมได้เต็มที่ ทำให้ test **เร็ว, เสถียร (deterministic), และไม่มี
side effect ต่อโลกภายนอก**

Python มีโมดูล `unittest.mock` ในตัว (standard library ตั้งแต่ Python 3.3)
ไม่ต้องติดตั้งอะไรเพิ่มเพื่อใช้งานพื้นฐาน

### 614.2 `Mock`: Object ปลอมที่รับทุกอย่างได้

```python
from unittest.mock import Mock

mock_obj = Mock()

# เรียก attribute หรือ method อะไรก็ได้ Mock จะ "สร้างมันขึ้นมาเอง" อัตโนมัติ
mock_obj.some_method()
mock_obj.some_attribute
mock_obj.another.deeply.nested.chain()   # ทำงานได้แม้ไม่เคยประกาศไว้ล่วงหน้า

# ตรวจสอบว่าเคยถูกเรียกหรือไม่
print(mock_obj.some_method.called)          # True
print(mock_obj.some_method.call_count)      # 1
```

### 614.3 กำหนดค่าที่ Mock ต้อง Return

```python
from unittest.mock import Mock

mock_calculator = Mock()
mock_calculator.calculate_total.return_value = 999.99

result = mock_calculator.calculate_total(items=[1, 2, 3])
assert result == 999.99   # ไม่สนใจ argument ที่ส่งเข้าไปเลย เพราะเรากำหนด return_value ตายตัว
```

### 614.4 `side_effect`: จำลองพฤติกรรมที่ซับซ้อนกว่า Return ค่าคงที่

`side_effect` ยืดหยุ่นกว่า `return_value` มาก รองรับ 3 รูปแบบ:

```python
from unittest.mock import Mock

# 1) เป็นฟังก์ชัน — Mock จะเรียกฟังก์ชันนี้แทน แล้วคืนค่าที่ฟังก์ชัน return
mock_fn = Mock(side_effect=lambda x: x * 2)
assert mock_fn(5) == 10

# 2) เป็น Exception — Mock จะ raise exception นั้นทันทีเมื่อถูกเรียก
mock_fn = Mock(side_effect=ConnectionError("เชื่อมต่อ API ไม่สำเร็จ"))
try:
    mock_fn()
except ConnectionError as e:
    print(str(e))   # "เชื่อมต่อ API ไม่สำเร็จ"

# 3) เป็น iterable — คืนค่าถัดไปในลิสต์ทุกครั้งที่ถูกเรียก (มีประโยชน์มากเวลาจำลอง
#    การเรียกซ้ำหลายครั้งที่ได้ผลลัพธ์ต่างกัน เช่น retry logic)
mock_fn = Mock(side_effect=[1, 2, ValueError("ครั้งที่ 3 ล้มเหลว")])
assert mock_fn() == 1
assert mock_fn() == 2
try:
    mock_fn()
except ValueError as e:
    print(str(e))   # "ครั้งที่ 3 ล้มเหลว"
```

### 614.5 `MagicMock`: `Mock` เวอร์ชันที่รองรับ Magic Method

`Mock` พื้นฐาน**ไม่รองรับ** magic method (dunder methods) เช่น `__len__`,
`__iter__`, `__eq__` — ถ้าเรียกจะเกิด `TypeError`:

```python
from unittest.mock import Mock

m = Mock()
len(m)   # TypeError: object of type 'Mock' has no len()
```

`MagicMock` แก้ปัญหานี้โดย config magic method ทั้งหมดให้อัตโนมัติ:

```python
from unittest.mock import MagicMock

m = MagicMock()
print(len(m))          # 0 (ค่าเริ่มต้น)
print(list(m))         # [] (ค่าเริ่มต้น เพราะ __iter__ ถูกตั้งไว้)
print(bool(m))         # True

m.__len__.return_value = 5
print(len(m))          # 5
```

**เมื่อไหร่ควรใช้ตัวไหน**: `MagicMock` เป็นค่าเริ่มต้นที่ปลอดภัยกว่าเมื่อไม่แน่ใจ
(ในความเป็นจริง `patch()` ในขั้นตอนที่ 615 ใช้ `MagicMock` เป็นค่าเริ่มต้นเสมอ)
ใช้ `Mock` ธรรมดาเฉพาะเมื่อรู้แน่ชัดว่า object ที่จำลองไม่มี magic method
เกี่ยวข้องเลย เพื่อความชัดเจนของโค้ด

### 614.6 `spec` และ `spec_set`: ป้องกัน Mock "โกหก" ว่ามี Method ที่ไม่มีจริง

ปัญหาใหญ่ของ `Mock`/`MagicMock` เปล่า ๆ คือมันรับทุก attribute แม้จะสะกดผิดหรือ
method นั้นไม่มีอยู่จริงใน class ต้นฉบับเลย ทำให้ test "ผ่าน" ทั้งที่โค้ดจริงจะ
พังทันทีถ้ารันจริง:

```python
from unittest.mock import Mock
from accounts.models import CustomUser

# ไม่มี spec — พิมพ์ method ผิดก็ไม่มีใครเตือน
mock_user = Mock()
mock_user.usernaem   # พิมพ์ผิด! แต่ Mock ยอมให้ผ่านเฉย ๆ ไม่ error

# มี spec — Mock จะ "เลียนแบบ" interface ของ CustomUser จริง
mock_user = Mock(spec=CustomUser)
mock_user.username   # ผ่าน เพราะ CustomUser มี attribute นี้จริง
mock_user.usernaem   # AttributeError ทันที! เพราะ CustomUser ไม่มี attribute นี้
```

`spec_set` เข้มงวดกว่า `spec` อีกขั้น: `spec` อนุญาตให้**อ่าน**เฉพาะ attribute
ที่มีจริง แต่ยัง**ตั้งค่า** attribute ใหม่ที่ไม่มีจริงได้ ส่วน `spec_set` ห้าม
ทั้งอ่านและเขียน attribute ที่ไม่มีอยู่ใน class ต้นฉบับ:

```python
mock_user = Mock(spec_set=CustomUser)
mock_user.username = "somchai"   # ผ่าน เพราะ CustomUser มี field นี้จริง
mock_user.some_typo_field = "x"  # AttributeError ทันที แม้จะเป็นการ "ตั้งค่าใหม่" ก็ตาม
```

> **คำแนะนำระดับมืออาชีพ**: ใน production test suite ควรใช้ `spec`/`spec_set`
> (หรือ `autospec=True` ที่จะเห็นในขั้นตอนที่ 618.4) แทบทุกครั้งที่ mock class
> หรือ object ที่มี interface ชัดเจน เพราะช่วยจับบั๊กจากการพิมพ์ผิดหรือ API
> เปลี่ยนแปลงได้ตั้งแต่ตอนรัน test แทนที่จะไปพังตอน production

---

## ขั้นตอนที่ 615: `patch` decorator/context manager — วิธี mock ฟังก์ชัน/method ที่ถูกต้อง ("patch where it's used")

### 615.1 `patch` คืออะไร

`Mock`/`MagicMock` ในขั้นตอนที่ 614 คือ object ปลอมที่เราสร้างเอง แต่ในสถานการณ์
จริง โค้ดที่เราอยากทดสอบมักเรียกใช้ฟังก์ชัน/class จาก **โมดูลอื่น** โดยตรง เช่น
`send_mail()` จาก `django.core.mail` หรือ `requests.get()` จาก package `requests`
เราไม่สามารถ "ส่ง Mock เข้าไปแทน" ได้ตรง ๆ เพราะโค้ดไม่ได้รับมันเป็น parameter

`unittest.mock.patch` แก้ปัญหานี้โดย **"สลับ" object ที่ path ที่ระบุ ให้กลาย
เป็น `MagicMock` ชั่วคราว เฉพาะระหว่างที่ test กำลังรันอยู่** แล้ว**คืนค่าเดิม
กลับอัตโนมัติ**เมื่อ test จบ (ไม่ว่าจะผ่านหรือ fail ก็ตาม)

### 615.2 ใช้เป็น Decorator

```python
# accounts/services.py
from django.core.mail import send_mail
from django.conf import settings


def send_welcome_email(user):
    send_mail(
        subject="ยินดีต้อนรับสู่ MyBlog",
        message=f"สวัสดีคุณ {user.username} ขอบคุณที่สมัครสมาชิก",
        from_email=settings.DEFAULT_FROM_EMAIL,
        recipient_list=[user.email],
    )
```

```python
# accounts/tests.py
from unittest.mock import patch
from accounts.services import send_welcome_email


class FakeUser:
    username = "somchai"
    email = "somchai@example.com"


@patch("accounts.services.send_mail")
def test_send_welcome_email_calls_send_mail(mock_send_mail):
    send_welcome_email(FakeUser())

    mock_send_mail.assert_called_once()
```

สังเกตว่าฟังก์ชัน test รับ argument ชื่อ `mock_send_mail` เพิ่มมาโดยอัตโนมัติ —
`patch` จะ**ส่ง Mock ที่มันสร้างเข้ามาเป็น argument ตัวสุดท้าย**ของฟังก์ชัน
เสมอ ทำให้เราตรวจสอบพฤติกรรมของมันต่อได้ในเนื้อ test

### 615.3 ใช้เป็น Context Manager

เมื่ออยากจำกัดขอบเขตของการ patch ให้แคบกว่าทั้งฟังก์ชัน test ใช้ `with` แทน:

```python
from unittest.mock import patch
from accounts.services import send_welcome_email


def test_send_welcome_email_with_context_manager():
    with patch("accounts.services.send_mail") as mock_send_mail:
        send_welcome_email(FakeUser())
        mock_send_mail.assert_called_once()

    # ออกจาก with block แล้ว send_mail กลับเป็นของจริงทันที
```

### 615.4 Patch หลายจุดพร้อมกัน (Stacking Decorators)

```python
@patch("accounts.services.send_mail")
@patch("accounts.services.log_email_sent")
def test_send_welcome_email_logs_and_sends(mock_log, mock_send_mail):
    # กฎสำคัญ: ลำดับ argument จะ "สวนทาง" กับลำดับ decorator เสมอ
    # decorator ที่อยู่ "ใกล้ฟังก์ชันที่สุด" (bottom) จะกลายเป็น argument "แรกสุด"
    send_welcome_email(FakeUser())
    mock_send_mail.assert_called_once()
    mock_log.assert_called_once()
```

หรือใช้หลาย context manager ซ้อนกัน (อ่านง่ายกว่าเมื่อมีมากกว่า 2-3 ตัว
ด้วย parenthesized context managers ของ Python 3.10+):

```python
def test_send_welcome_email_with_multiple_context_managers():
    with (
        patch("accounts.services.send_mail") as mock_send_mail,
        patch("accounts.services.log_email_sent") as mock_log,
    ):
        send_welcome_email(FakeUser())
        mock_send_mail.assert_called_once()
        mock_log.assert_called_once()
```

### 615.5 `patch.object`: Patch Method ของ Class ที่มีอยู่แล้วโดยตรง

เมื่อต้องการ patch แค่ method เดียวของ class ที่ import มาแล้ว (ไม่ต้องพิมพ์
dotted path เต็ม) ใช้ `patch.object` สะดวกกว่า:

```python
from unittest.mock import patch
from accounts.services import EmailService


@patch.object(EmailService, "send")
def test_email_service_send_is_called(mock_send):
    service = EmailService()
    service.notify_user(FakeUser())
    mock_send.assert_called_once()
```

### 615.6 หัวใจสำคัญที่สุดของ Part นี้: "Patch Where It's Used" ไม่ใช่ "Patch Where It's Defined"

นี่คือแหล่งความสับสนอันดับหนึ่งของทุกคนที่เริ่มใช้ `unittest.mock` แม้แต่คน
ที่เขียน Python มานานก็ยังพลาดบ่อย ลองดูตัวอย่างที่ **ผิด**:

```python
# accounts/services.py
from django.core.mail import send_mail   # import เข้ามาผูกกับชื่อ send_mail ใน "โมดูลนี้"


def send_welcome_email(user):
    send_mail(subject="...", message="...", from_email="...", recipient_list=[user.email])
```

```python
# ❌ ผิด! patch ที่ "ต้นทาง" ของฟังก์ชัน ไม่ใช่ที่ที่มันถูกเรียกใช้จริง
@patch("django.core.mail.send_mail")
def test_send_welcome_email_wrong_patch_target(mock_send_mail):
    send_welcome_email(FakeUser())
    mock_send_mail.assert_called_once()   # ❌ FAIL! Mock ไม่เคยถูกเรียกเลย
```

**ทำไม test นี้ fail ทั้งที่ path ที่ patch ก็ดู "ถูกต้อง"**: เมื่อ Python รัน
`from django.core.mail import send_mail` ที่ด้านบนของ `accounts/services.py`
มันจะสร้าง**ชื่อใหม่**ชื่อ `send_mail` ขึ้นมาใน **namespace ของ
`accounts.services`** ที่ชี้ไปยังฟังก์ชันเดียวกัน — จากจุดนี้เป็นต้นไป
`accounts.services.send_mail` และ `django.core.mail.send_mail` เป็น
**ชื่อสองชื่อที่ชี้ไปที่ object เดียวกัน แต่แยกอิสระจากกัน**

เมื่อเรา `patch("django.core.mail.send_mail")` เราแค่เปลี่ยนว่า **ชื่อ
`send_mail` ในโมดูล `django.core.mail`** ชี้ไปที่ไหน — แต่โค้ดใน
`accounts/services.py` ไม่ได้มองหาชื่อนั้นในโมดูล `django.core.mail` อีกต่อไป
แล้ว มันใช้ชื่อ `send_mail` **ที่ผูกไว้ใน namespace ของตัวเอง**ตั้งแต่ตอน
import ซึ่งยังคงชี้ไปที่ฟังก์ชันจริง ไม่ได้ถูกแตะต้องเลย

**กฎทอง**: **ต้อง patch ที่จุดที่ชื่อนั้นถูกใช้งาน (consumer) ไม่ใช่จุดที่มัน
ถูกประกาศไว้ครั้งแรก (origin)** ดังนั้นวิธีที่ถูกต้องคือ:

```python
# ✅ ถูกต้อง! patch ที่ "จุดที่ accounts.services เรียกใช้" ไม่ใช่ที่ django.core.mail
@patch("accounts.services.send_mail")
def test_send_welcome_email_correct_patch_target(mock_send_mail):
    send_welcome_email(FakeUser())
    mock_send_mail.assert_called_once()   # ✅ PASS
```

### 615.7 ตารางสรุป: จะรู้ได้อย่างไรว่าต้อง Patch Path ไหน

| รูปแบบการ import ในไฟล์ที่ถูกทดสอบ | ต้อง patch path ไหน |
|---|---|
| `from django.core.mail import send_mail` แล้วเรียก `send_mail(...)` | `"accounts.services.send_mail"` (path ของไฟล์ที่เรียกใช้) |
| `from django.core import mail` แล้วเรียก `mail.send_mail(...)` | `"accounts.services.mail.send_mail"` หรือ `"django.core.mail.send_mail"` ก็ได้ทั้งคู่ (เพราะเข้าถึงผ่าน attribute ของโมดูล ไม่ได้ copy ชื่อมาตรง ๆ) |
| `import requests` แล้วเรียก `requests.get(...)` | `"accounts.validators.requests.get"` (path ของไฟล์ที่เรียกใช้ ตามด้วย `.requests.get`) |

**หลักจำง่าย ๆ**: ถ้าไฟล์ที่ทดสอบใช้ `from X import Y` แล้วเรียก `Y(...)` ตรง ๆ
ต้อง patch `"<โมดูลของไฟล์ที่ทดสอบ>.Y"` เสมอ ถ้าใช้ `import X` แล้วเรียก
`X.Y(...)` จะ patch ที่ `"X.Y"` (ต้นทาง) หรือ `"<โมดูลที่ทดสอบ>.X.Y"` ก็ได้ผล
เหมือนกัน เพราะไม่ได้มีการ copy ชื่อ `Y` ออกมาต่างหาก แต่เข้าถึงผ่าน attribute
ของโมดูล `X` ทุกครั้งที่เรียก

---

## ขั้นตอนที่ 616: Mock การเรียก External API จริง (เช่น การส่งอีเมล, การเรียก Have I Been Pwned จาก Part 035)

### 616.1 ทางเลือกที่ 1 สำหรับทดสอบอีเมล: `django.core.mail.outbox` (ไม่ต้อง Mock เอง)

Django มีกลไก **fake email backend สำหรับ test** ในตัวอยู่แล้ว — เมื่อรัน test
ด้วย `django.test.TestCase` (หรือ `pytest-django` ที่ผูก settings เดียวกัน)
Django จะสลับ `EMAIL_BACKEND` เป็น
`django.core.mail.backends.locmem.EmailBackend` ให้อัตโนมัติ ซึ่งเก็บอีเมลที่
"ส่งออก" ทั้งหมดไว้ในลิสต์ `django.core.mail.outbox` แทนที่จะส่งจริง:

```python
# accounts/tests.py
import pytest
from django.core import mail
from accounts.services import send_welcome_email


class FakeUser:
    username = "somchai"
    email = "somchai@example.com"


@pytest.mark.django_db
def test_send_welcome_email_via_outbox():
    send_welcome_email(FakeUser())

    assert len(mail.outbox) == 1
    sent_email = mail.outbox[0]
    assert sent_email.subject == "ยินดีต้อนรับสู่ MyBlog"
    assert "somchai@example.com" in sent_email.to
```

วิธีนี้เหมาะสำหรับ **integration test ระดับกลาง** ที่ต้องการยืนยันว่า Django
mail framework ทำงานถูกต้องจริง ๆ (subject, body, recipient ครบถ้วน) โดยไม่ต้อง
เขียน mock เอง — แต่มันยัง**ไม่ใช่** external API ตัวจริง (ไม่ได้ต่อ SMTP server
จริง) จึงยังปลอดภัยและเร็วสำหรับ test suite

### 616.2 ทางเลือกที่ 2: `patch` ตรง ๆ เมื่อต้องการ Unit Test แบบแยกส่วนอย่างเข้มงวด

เมื่อต้องการทดสอบแค่ "ฟังก์ชันของเราเรียก `send_mail` ด้วย argument ที่ถูกต้อง
หรือไม่" โดยไม่สนใจว่า Django mail framework ทำงานถูกต้องหรือเปล่า (เพราะนั่น
เป็นความรับผิดชอบของ Django เอง ไม่ใช่โค้ดของเรา) ให้ใช้ `patch` ตามหลัก
"patch where it's used" จากขั้นตอนที่ 615.6:

```python
# accounts/tests.py
from unittest.mock import patch
from accounts.services import send_welcome_email


@patch("accounts.services.send_mail")
def test_send_welcome_email_calls_send_mail_with_correct_arguments(mock_send_mail):
    user = FakeUser()
    send_welcome_email(user)

    mock_send_mail.assert_called_once_with(
        subject="ยินดีต้อนรับสู่ MyBlog",
        message="สวัสดีคุณ somchai ขอบคุณที่สมัครสมาชิก",
        from_email="noreply@myblog.example.com",
        recipient_list=["somchai@example.com"],
    )
```

test นี้รันเร็วกว่า (ไม่ต้องแตะฐานข้อมูลหรือ mail framework เลย) และแยกความ
รับผิดชอบชัดเจน: **เราทดสอบแค่ logic ของเราเอง** ไม่ใช่ทดสอบว่า Django ส่งอีเมล
ได้จริงหรือไม่ (นั่นเป็นสิ่งที่ Django's own test suite รับผิดชอบอยู่แล้ว)

### 616.3 Mock การเรียก HTTP API ภายนอกจริง: `PwnedPasswordValidator` จาก Part 035

Part 035 ขั้นตอนที่ 349 เขียน `PwnedPasswordValidator` ที่เรียก Have I Been
Pwned API ผ่าน `requests.get()` จริง — นี่คือตัวอย่างที่สมบูรณ์แบบของ external
dependency ที่**ต้อง**ถูก mock ใน test เพราะ:

- ไม่อยากให้ test suite ต้องพึ่งพาอินเทอร์เน็ตหรือ HIBP ยังทำงานปกติอยู่หรือไม่
- ไม่อยากให้ test ช้าลงเพราะรอ network round-trip ทุกครั้งที่รัน
- ต้องการควบคุมผลลัพธ์ที่แน่นอน (เช่น จำลองว่า "รหัสผ่านนี้เคยรั่วไหลแน่นอน")
  ซึ่งทำไม่ได้ถ้าพึ่งพา API จริงที่ข้อมูลเปลี่ยนแปลงได้ตลอดเวลา

ทบทวนโค้ดจาก Part 035 (`accounts/validators.py`):

```python
# accounts/validators.py (จาก Part 035 ขั้นตอนที่ 349.3)
import hashlib
import logging

import requests
from django.core.exceptions import ValidationError
from django.utils.translation import gettext as _

logger = logging.getLogger(__name__)

HIBP_API_URL = "https://api.pwnedpasswords.com/range/{prefix}"


class PwnedPasswordValidator:
    def __init__(self, timeout=2, fail_open=True):
        self.timeout = timeout
        self.fail_open = fail_open

    def validate(self, password, user=None):
        sha1_hash = hashlib.sha1(password.encode("utf-8")).hexdigest().upper()
        prefix, suffix = sha1_hash[:5], sha1_hash[5:]

        try:
            response = requests.get(
                HIBP_API_URL.format(prefix=prefix),
                headers={"User-Agent": "MyBlog-Django-PwnedPasswordValidator"},
                timeout=self.timeout,
            )
            response.raise_for_status()
        except requests.RequestException as exc:
            logger.warning("ไม่สามารถเชื่อมต่อ HIBP API ได้: %s", exc)
            if self.fail_open:
                return
            raise ValidationError(
                _("ไม่สามารถตรวจสอบความปลอดภัยของรหัสผ่านได้ในขณะนี้ กรุณาลองใหม่อีกครั้ง"),
                code="password_pwned_check_failed",
            )

        for line in response.text.splitlines():
            candidate_suffix, count = line.split(":")
            if candidate_suffix == suffix:
                raise ValidationError(
                    _("รหัสผ่านนี้เคยปรากฏในฐานข้อมูลรหัสผ่านที่รั่วไหลมาแล้ว "
                      "อย่างน้อย %(count)s ครั้ง กรุณาเลือกรหัสผ่านอื่นเพื่อความปลอดภัย"),
                    code="password_pwned",
                    params={"count": count},
                )

    def get_help_text(self):
        return _("รหัสผ่านของคุณจะถูกตรวจสอบว่าเคยรั่วไหลในเหตุการณ์ข้อมูลรั่วไหลที่รู้จักหรือไม่")
```

สังเกตว่าไฟล์นี้เขียน `import requests` แล้วเรียก `requests.get(...)` (ไม่ใช่
`from requests import get`) ดังนั้นตามตารางในขั้นตอนที่ 615.7 เรา patch ที่
`"accounts.validators.requests.get"` ได้ (หรือ `"requests.get"` ตรง ๆ ก็ได้ผล
เหมือนกัน เพราะเป็นการ `import X` ไม่ใช่ `from X import Y`)

### 616.4 เขียน Test ครบทุก Path ของ `PwnedPasswordValidator`

```python
# accounts/tests.py
from unittest.mock import Mock, patch

import pytest
import requests
from django.core.exceptions import ValidationError

from accounts.validators import PwnedPasswordValidator


class TestPwnedPasswordValidator:
    def setup_method(self):
        self.validator = PwnedPasswordValidator()

    @patch("accounts.validators.requests.get")
    def test_password_found_in_breach_raises_validation_error(self, mock_get):
        # sha1("password123") = CBFDAC6008F9CAB4083784CBD1874F76618D2A97
        # prefix = "CBFDA", suffix = "C6008F9CAB4083784CBD1874F76618D2A97"
        fake_response = Mock()
        fake_response.text = (
            "C6008F9CAB4083784CBD1874F76618D2A97:2680482\n"
            "OTHERSUFFIXTHATDOESNOTMATCH00000000:15\n"
        )
        fake_response.raise_for_status.return_value = None
        mock_get.return_value = fake_response

        with pytest.raises(ValidationError) as exc_info:
            self.validator.validate("password123")

        assert "เคยปรากฏในฐานข้อมูลรหัสผ่านที่รั่วไหล" in str(exc_info.value)
        mock_get.assert_called_once()
        # ตรวจสอบด้วยว่าเราส่งแค่ prefix 5 ตัวอักษร ไม่ส่งรหัสผ่านเต็มออกไป (k-Anonymity)
        called_url = mock_get.call_args.args[0]
        assert called_url.endswith("/range/CBFDA")

    @patch("accounts.validators.requests.get")
    def test_password_not_found_in_breach_passes(self, mock_get):
        fake_response = Mock()
        fake_response.text = "SOMEUNRELATEDSUFFIX0000000000000000:1\n"
        fake_response.raise_for_status.return_value = None
        mock_get.return_value = fake_response

        # ไม่ raise = ผ่าน
        self.validator.validate("a-very-unique-password-nobody-uses-2026")

    @patch("accounts.validators.requests.get")
    def test_api_timeout_fails_open_by_default(self, mock_get):
        mock_get.side_effect = requests.exceptions.Timeout("เชื่อมต่อ HIBP ไม่ทัน")

        # fail_open=True (ค่าเริ่มต้น) ต้อง "ปล่อยผ่าน" ไม่บล็อกผู้ใช้
        self.validator.validate("any-password")   # ไม่ raise

    @patch("accounts.validators.requests.get")
    def test_api_timeout_fails_closed_when_configured(self, mock_get):
        strict_validator = PwnedPasswordValidator(fail_open=False)
        mock_get.side_effect = requests.exceptions.Timeout("เชื่อมต่อ HIBP ไม่ทัน")

        with pytest.raises(ValidationError, match="ไม่สามารถตรวจสอบความปลอดภัย"):
            strict_validator.validate("any-password")
```

**สังเกตสิ่งสำคัญ**: test ทั้ง 4 case นี้รันเร็วมาก (มิลลิวินาที) ไม่ต้องต่อ
อินเทอร์เน็ตจริงเลย และครอบคลุมทุก branch สำคัญของฟังก์ชัน — พบแล้ว, ไม่พบ,
API ล้มเหลวแบบ fail-open, API ล้มเหลวแบบ fail-closed — ซึ่งเป็นสิ่งที่แทบ
เป็นไปไม่ได้เลยถ้าต้องพึ่งพา HIBP API จริงในทุกครั้งที่รัน test (จะจำลองกรณี
"API ล้มเหลว" ได้อย่างไรถ้าไม่ mock?)

### 616.5 เปรียบเทียบ: Mock อีเมล vs Mock HTTP API — เหมือนกันแค่ไหน

| ประเด็น | Mock `send_mail` (616.2) | Mock `requests.get` (616.4) |
|---|---|---|
| Dependency ที่ถูกแทนที่ | ฟังก์ชันของ Django framework | ฟังก์ชันของ third-party library |
| ทางเลือกที่ไม่ต้อง mock เอง | มี (`mail.outbox` จาก locmem backend) | ไม่มีในตัว Django — ต้อง mock เองเสมอสำหรับ HTTP call ทั่วไป |
| สิ่งที่ต้องตรวจสอบหลัง mock | argument ที่ส่งเข้า `send_mail` ถูกต้อง | URL/headers ที่ส่งไปถูกต้อง และ logic ที่ประมวลผล response กลับมาถูกต้อง |
| ความเสี่ยงถ้าไม่ mock | test ช้าลงเล็กน้อย (แต่ locmem backend ก็ไม่ได้ส่งจริงอยู่แล้ว) | test suite ต้องพึ่งอินเทอร์เน็ต, ช้ามาก, ไม่เสถียร (flaky), และอาจโดน rate limit จาก HIBP จริง |

---

## ขั้นตอนที่ 617: Mock เวลา ด้วย `freezegun` (ทดสอบ logic ที่ผูกกับวันที่/เวลา)

### 617.1 ปัญหาของการทดสอบ Logic ที่ผูกกับเวลา

ลองดูฟังก์ชันตรวจสอบว่าโปรโมชันยังใช้งานได้อยู่หรือไม่:

```python
# blog/models.py
from django.db import models
from django.utils import timezone


class Promotion(models.Model):
    code = models.CharField(max_length=20)
    starts_at = models.DateTimeField()
    ends_at = models.DateTimeField()

    def is_active(self):
        now = timezone.now()
        return self.starts_at <= now <= self.ends_at
```

ถ้าเขียน test แบบตรงไปตรงมาโดยใช้วันที่ปัจจุบันจริง ๆ test นี้จะ**ผลลัพธ์ไม่คงที่**
เมื่อเวลาผ่านไป — ถ้า `ends_at` ถูก hardcode เป็นวันที่ในอดีตของตอนที่เขียน test
มันจะ fail ทันทีเมื่อรันในวันข้างหน้า (test ที่ผลลัพธ์เปลี่ยนไปตามเวลาแบบนี้
เรียกว่า **time-bomb test** — เป็น test smell ที่อันตรายมาก เพราะจะพังเองในอนาคต
โดยไม่มีใครแก้โค้ดเลย)

### 617.2 ติดตั้งและใช้ `freezegun`

**`freezegun`** คือ library ยอดนิยมที่ "หยุดเวลา" ของทั้งระบบ (`datetime.now()`,
`django.utils.timezone.now()`, `time.time()`) ให้คงที่ตามที่เรากำหนดระหว่าง
test เท่านั้น:

```bash
pip install freezegun
```

```python
# blog/tests.py
from datetime import timedelta

import pytest
from django.utils import timezone
from freezegun import freeze_time

from blog.models import Promotion


@pytest.mark.django_db
@freeze_time("2026-06-15 12:00:00")
def test_promotion_is_active_during_valid_period():
    promo = Promotion.objects.create(
        code="SUMMER2026",
        starts_at=timezone.now() - timedelta(days=1),
        ends_at=timezone.now() + timedelta(days=1),
    )
    assert promo.is_active() is True


@pytest.mark.django_db
@freeze_time("2026-06-15 12:00:00")
def test_promotion_is_not_active_before_start():
    promo = Promotion.objects.create(
        code="WINTER2026",
        starts_at=timezone.now() + timedelta(days=1),   # เริ่มพรุ่งนี้
        ends_at=timezone.now() + timedelta(days=10),
    )
    assert promo.is_active() is False


@pytest.mark.django_db
@freeze_time("2026-06-15 12:00:00")
def test_promotion_is_not_active_after_end():
    promo = Promotion.objects.create(
        code="EXPIRED2026",
        starts_at=timezone.now() - timedelta(days=10),
        ends_at=timezone.now() - timedelta(days=1),   # จบไปแล้วเมื่อวาน
    )
    assert promo.is_active() is False
```

test ทั้งสามตัวนี้จะได้ผลลัพธ์**เหมือนเดิมทุกครั้งไม่ว่าจะรันวันไหน** เพราะ
`@freeze_time("2026-06-15 12:00:00")` บังคับให้ `timezone.now()` คืนค่าเวลานั้น
เสมอตลอดการรันฟังก์ชัน test นั้น ๆ

### 617.3 ใช้เป็น Context Manager และการ "เดินเวลาไปข้างหน้า"

บางสถานการณ์ต้องการจำลอง "เวลาผ่านไป" ระหว่าง test เดียวกัน เช่น ทดสอบว่า
token หมดอายุหลังผ่านไปตามระยะเวลาที่กำหนดหรือไม่:

```python
# accounts/tests.py
from freezegun import freeze_time
from django.contrib.auth.tokens import default_token_generator
from django.test import override_settings

from accounts.models import CustomUser


@pytest.mark.django_db
def test_password_reset_token_expires_after_timeout():
    user = CustomUser.objects.create_user(
        username="somchai", email="somchai@example.com", password="OldPass123!"
    )

    with freeze_time("2026-01-01 09:00:00") as frozen_time:
        token = default_token_generator.make_token(user)
        assert default_token_generator.check_token(user, token) is True

        # เดินเวลาไปข้างหน้า 2 วัน (มากกว่า PASSWORD_RESET_TIMEOUT = 1 วันจาก Part 035)
        frozen_time.tick(delta=timedelta(days=2))

        with override_settings(PASSWORD_RESET_TIMEOUT=60 * 60 * 24):
            assert default_token_generator.check_token(user, token) is False
```

`frozen_time.tick(delta=...)` (หรือ `frozen_time.move_to("2026-01-03")`)
ใช้เดินเวลาไปข้างหน้า/ถอยหลังจากจุดที่ freeze ไว้ตอนแรก โดยไม่ต้องออกจาก
context manager แล้วเข้าใหม่

### 617.4 `freeze_time` ครอบคลุมโมดูลไหนบ้าง

`freezegun` แพตช์ที่ระดับลึกกว่า `unittest.mock.patch` ทั่วไปมาก มันแทนที่
`datetime.datetime.now`, `datetime.date.today`, `time.time` และอื่น ๆ ในทุกโมดูล
ที่ import `datetime`/`time` เข้าไปโดยตรง (ไม่ติดปัญหา "patch where it's used"
จากขั้นตอนที่ 615.6 เหมือน mock ทั่วไป เพราะมันแพตช์ที่ตัว C-level function ของ
`datetime` module เอง) ทำให้ `django.utils.timezone.now()` (ซึ่งเรียก
`datetime.now()` ภายใน) ได้รับผลกระทบไปด้วยโดยอัตโนมัติ นี่คือเหตุผลที่
`freezegun` สะดวกกว่าการ `patch("django.utils.timezone.now")` เอง — ไม่ต้อง
กังวลว่าจะ patch จุดไหน เพราะมันครอบคลุมทั้งระบบให้แล้ว

### 617.5 ทำไมไม่ใช้ `unittest.mock.patch` กับ `timezone.now` ตรง ๆ

ทำได้เหมือนกัน แต่มีข้อเสียชัดเจนเมื่อเทียบกับ `freezegun`:

```python
# ทำได้ แต่ยุ่งยากกว่ามาก
from unittest.mock import patch
from datetime import datetime

@patch("django.utils.timezone.now")
def test_with_manual_patch(mock_now):
    mock_now.return_value = datetime(2026, 6, 15, 12, 0, 0, tzinfo=timezone.utc)
    # ปัญหา: ทุกที่ในโค้ดที่เรียก timezone.now() โดยตรง (import คนละจุด) ต้อง
    # patch แยกทีละจุดตามกฎ "patch where it's used" — ถ้ามี 5 ไฟล์เรียก
    # timezone.now() ต้อง patch ครบทั้ง 5 จุด! ในขณะที่ freezegun แก้ปัญหานี้
    # ให้ครบในบรรทัดเดียว
    ...
```

**สรุป**: ใช้ `freezegun` สำหรับทุกกรณีที่ต้องควบคุมเวลา ใช้
`patch("...timezone.now")` เฉพาะเมื่อไม่อยากเพิ่ม dependency ใหม่ให้โปรเจกต์
และมีจุดที่เรียก `timezone.now()` แค่จุดเดียวเท่านั้น

---

## ขั้นตอนที่ 618: Spy pattern — `assert_called_with()`, `assert_called_once()`, `call_count`

### 618.1 Stub, Mock และ Spy ต่างกันอย่างไร

คำศัพท์ทั้งสามมักถูกใช้ปนกันในบทสนทนาทั่วไป แต่ในทาง Test Double มีความหมาย
เฉพาะที่ต่างกัน:

| ประเภท | หน้าที่หลัก | ตัวอย่างในบทนี้ |
|---|---|---|
| **Stub** | คืนค่าปลอมที่กำหนดไว้ล่วงหน้า เพื่อให้โค้ดที่ทดสอบทำงานต่อได้ | `mock_get.return_value = fake_response` (ขั้นตอนที่ 616.4) |
| **Mock** (ในความหมายแคบ) | Stub ที่**ยัง**ตรวจสอบด้วยว่าถูกเรียกอย่างไรบ้าง | เกือบทุกตัวอย่างใน Part นี้ที่ใช้ `Mock()`/`MagicMock()` |
| **Spy** | ห่อ object **จริง** ไว้ แล้วบันทึกว่ามันถูกเรียกอย่างไรบ้าง โดยยังให้ทำงานจริงตามปกติ | `patch.object(obj, "method", wraps=obj.method)` |

ในทางปฏิบัติ เมื่อพูดถึง "spy pattern" ใน context ของ `unittest.mock` เรามักหมาย
ถึงการใช้ `Mock`/`patch` เพื่อ**สังเกตการณ์** (observe) ว่าฟังก์ชันถูกเรียก
อย่างไรบ้าง มากกว่าการควบคุม return value ของมัน — เมธอด `assert_called_*`
ทั้งหมดคือเครื่องมือหลักของรูปแบบนี้

### 618.2 เมธอด `assert_called_*` ที่ใช้บ่อยที่สุด

```python
from unittest.mock import Mock

mock_notify = Mock()

mock_notify("somchai@example.com", subject="แจ้งเตือน")

# ถูกเรียกอย่างน้อย 1 ครั้งหรือไม่ (ไม่สนใจจำนวนครั้ง)
mock_notify.assert_called()

# ถูกเรียก "เพียงครั้งเดียว" เท่านั้นหรือไม่ (ไม่สนใจ argument)
mock_notify.assert_called_once()

# ถูกเรียกด้วย argument ชุดนี้ "ครั้งล่าสุด" หรือไม่
mock_notify.assert_called_with("somchai@example.com", subject="แจ้งเตือน")

# ถูกเรียก "เพียงครั้งเดียว" และด้วย argument ชุดนี้เป๊ะ ๆ (เข้มงวดที่สุด แนะนำให้ใช้เป็นค่าเริ่มต้น)
mock_notify.assert_called_once_with("somchai@example.com", subject="แจ้งเตือน")

# ไม่เคยถูกเรียกเลย
mock_other = Mock()
mock_other.assert_not_called()

# ถูกเรียกด้วย argument ชุดนี้ "ครั้งใดครั้งหนึ่ง" ก็ได้ (ไม่จำเป็นต้องเป็นครั้งล่าสุด)
mock_notify.assert_any_call("somchai@example.com", subject="แจ้งเตือน")
```

**ข้อควรระวัง**: ทุก `assert_called*` method จะ**ไม่ error เลยแม้พิมพ์ชื่อผิด**
เช่น `mock.asert_called_once()` (สะกดผิด `assert`) จะไม่ error แต่จะสร้าง
attribute ใหม่ที่ชื่อ `asert_called_once` แล้วเรียกมันเฉย ๆ โดยไม่ตรวจสอบอะไร
เลย — test จะ "ผ่าน" เสมอทั้งที่ไม่ได้ assert อะไรจริง! นี่คือเหตุผลสำคัญที่ควร
ใช้ `autospec=True` (ขั้นตอนที่ 618.4) เพื่อป้องกันปัญหานี้

### 618.3 ตรวจสอบรายละเอียดของการเรียกด้วย `call_count`, `call_args`, `call_args_list`

```python
from unittest.mock import Mock

mock_send = Mock()
mock_send("user1@example.com")
mock_send("user2@example.com", priority="high")
mock_send("user3@example.com")

print(mock_send.call_count)          # 3

print(mock_send.call_args)           # เก็บเฉพาะ "การเรียกครั้งล่าสุด"
                                      # call('user3@example.com')

print(mock_send.call_args.args)      # ('user3@example.com',)
print(mock_send.call_args.kwargs)    # {}

print(mock_send.call_args_list)      # เก็บ "ทุกครั้ง" ที่เคยถูกเรียก ตามลำดับ
# [call('user1@example.com'),
#  call('user2@example.com', priority='high'),
#  call('user3@example.com')]

# ตรวจสอบว่าการเรียกครั้งที่ 2 (index 1) มี kwarg priority="high" หรือไม่
assert mock_send.call_args_list[1].kwargs == {"priority": "high"}
```

### 618.4 `autospec=True`: Spy ที่ปลอดภัยกว่า Mock ธรรมดา

`autospec=True` ที่ใช้กับ `patch()` จะทำสิ่งเดียวกับ `spec` ในขั้นตอนที่ 614.6
โดยอัตโนมัติ **รวมถึงตรวจสอบ signature ของ argument ด้วย**:

```python
from unittest.mock import patch


@patch("accounts.services.send_mail", autospec=True)
def test_send_welcome_email_with_autospec(mock_send_mail):
    send_welcome_email(FakeUser())

    # ถ้าโค้ดจริงเรียก send_mail() ด้วยจำนวน argument ผิด หรือชื่อ keyword
    # argument ที่ไม่มีอยู่จริงใน signature ของ django.core.mail.send_mail
    # test นี้จะ TypeError ทันที แม้ mock_send_mail.assert_called_once() จะผ่าน
    mock_send_mail.assert_called_once()
```

โดยไม่มี `autospec=True` ถ้าโค้ดจริงเผลอเรียก
`send_mail(subjcet="...")` (พิมพ์ผิด) test จะยังคง "ผ่าน" เพราะ `Mock` เปล่า
ยอมรับ keyword argument อะไรก็ได้ — `autospec=True` จับความผิดพลาดแบบนี้ได้
ทันทีตั้งแต่รัน test แทนที่จะไปพังตอน production จริง

### 618.5 ตัวอย่างสมบูรณ์: Spy บน Django Signal

ทดสอบว่าเมื่อสมัครสมาชิกสำเร็จ signal `post_save` เรียก
`send_welcome_email` ด้วย user ที่ถูกต้องหรือไม่ (สมมติว่ามี signal handler
ต่อจาก Part 032):

```python
# accounts/signals.py
from django.db.models.signals import post_save
from django.dispatch import receiver

from accounts.models import CustomUser
from accounts.services import send_welcome_email


@receiver(post_save, sender=CustomUser)
def send_welcome_email_on_signup(sender, instance, created, **kwargs):
    if created:
        send_welcome_email(instance)
```

```python
# accounts/tests.py
from unittest.mock import patch

import pytest
from accounts.models import CustomUser


@pytest.mark.django_db
@patch("accounts.signals.send_welcome_email", autospec=True)
def test_signup_triggers_welcome_email_exactly_once(mock_send_welcome_email):
    user = CustomUser.objects.create_user(
        username="somchai", email="somchai@example.com", password="SecurePass123!"
    )

    mock_send_welcome_email.assert_called_once_with(user)

    # แก้ไข user คนเดิม (ไม่ใช่การสร้างใหม่) ต้อง "ไม่" ส่งอีเมลต้อนรับซ้ำ
    user.first_name = "สมชาย"
    user.save()

    mock_send_welcome_email.assert_called_once()   # ยังคงเป็น 1 ครั้งเท่านั้น ไม่ใช่ 2
```

test นี้ใช้ spy pattern อย่างสมบูรณ์: ตรวจสอบว่าฟังก์ชันถูกเรียก **ถูกต้อง
กี่ครั้ง** และ **ด้วย argument อะไร** โดยไม่ต้องสนใจว่า `send_welcome_email`
ภายในทำงานอย่างไร (นั่นเป็นหน้าที่ของ test ในขั้นตอนที่ 616 ที่ทดสอบแยกไว้แล้ว)

---

## ขั้นตอนที่ 619: ข้อควรระวังการ Mock มากเกินไป (test ที่ mock ทุกอย่างจนไม่ทดสอบอะไรจริง — "test smell")

### 619.1 เมื่อ Mock กลายเป็นดาบสองคม

Mocking เป็นเครื่องมือที่ทรงพลัง แต่ **ใช้มากเกินไปจะทำให้ test สูญเสีย
ความหมายไปเลย** ลองดูตัวอย่างที่ over-mock อย่างรุนแรง:

```python
# ❌ ตัวอย่างของ Over-Mocking ที่ไม่ควรเขียนแบบนี้
from unittest.mock import Mock, patch


@patch("blog.views.Post.objects")
@patch("blog.views.render")
def test_post_list_view_over_mocked(mock_render, mock_post_objects):
    mock_queryset = Mock()
    mock_queryset.all.return_value = mock_queryset
    mock_queryset.order_by.return_value = [Mock(), Mock()]
    mock_post_objects.all.return_value = mock_queryset
    mock_render.return_value = Mock(status_code=200)

    from blog.views import post_list
    request = Mock()
    response = post_list(request)

    mock_post_objects.all.assert_called_once()
    mock_render.assert_called_once()
    assert response.status_code == 200
```

test นี้ **ไม่ได้ทดสอบอะไรจริงเลย** มัน mock ทุกอย่างจนเหลือแค่การตรวจสอบว่า
"เรียก method ที่ชื่อถูกต้องหรือเปล่า" — ถ้ามีบั๊กจริงในการ query ข้อมูล
(เช่น ลืม filter เฉพาะ post ที่ `published=True`) หรือ template ที่ render
ออกมาผิด test นี้ก็ยัง**ผ่านเหมือนเดิมทุกประการ** เพราะไม่มีจุดไหนแตะฐานข้อมูล
หรือ template จริงเลยแม้แต่นิดเดียว

### 619.2 อาการของ "Test Smell" จากการ Mock มากเกินไป

| อาการ | ตัวอย่าง | ปัญหา |
|---|---|---|
| **Mock สิ่งที่ไม่ใช่ external boundary** | mock `Post.objects` (Django ORM) แทนที่จะใช้ test database จริง | ORM ของ Django ผ่านการทดสอบมาอย่างดีแล้ว การ mock มันทำให้ test ไม่ยืนยันว่า query ของเราถูกต้องจริงหรือไม่ |
| **Mock ฟังก์ชันที่กำลังทดสอบเอง (โดยไม่ตั้งใจ)** | mock helper function ภายในของฟังก์ชันที่กำลังทดสอบ | test แค่ตรวจว่า "เรียก helper แล้ว" ไม่ได้ตรวจว่า "ผลลัพธ์สุดท้ายถูกต้อง" |
| **Assertion ตรวจแค่ mock ถูกเรียก ไม่ตรวจผลลัพธ์จริง** | `mock_render.assert_called_once()` แต่ไม่เคย assert เนื้อหาที่ render ออกมา | บั๊กด้าน business logic หลุดรอดไปได้ง่ายมาก |
| **Test พังทุกครั้งที่ Refactor แม้พฤติกรรมไม่เปลี่ยน** | เปลี่ยนจาก `Post.objects.filter().order_by()` เป็น `Post.objects.order_by().filter()` (ผลลัพธ์เหมือนเดิม) แต่ test ที่ mock เจาะจงลำดับ method พัง | test ผูกติดกับ **วิธีการ implement** ไม่ใช่ **พฤติกรรมที่สังเกตได้จากภายนอก** — เปราะบางเกินไป |
| **จำนวนบรรทัด mock setup มากกว่าโค้ดจริงที่ทดสอบ** | ต้อง mock 5 object ซ้อนกันเพื่อทดสอบฟังก์ชัน 3 บรรทัด | สัญญาณชัดเจนว่า test นี้ควรใช้ของจริงแทน (เช่น test database) |

### 619.3 หลักการ: Mock เฉพาะที่ "System Boundary" เท่านั้น

**กฎทองของการ mock ที่ดี**: ให้ mock เฉพาะจุดที่โค้ดของเรา**ข้ามพรมแดนออกไป
สู่ระบบภายนอกที่เราไม่ได้เขียนเอง** (external boundary) เท่านั้น ส่วน logic
ภายในระบบของเราเอง (models, business logic, การคำนวณ) ควรทดสอบด้วยของจริง
เสมอเมื่อเป็นไปได้

| ควร Mock (System Boundary) | ไม่ควร Mock (Internal Collaborator) |
|---|---|
| การเรียก HTTP API ภายนอก (`requests.get`) | Django ORM (`Model.objects.filter()`) — ใช้ test database จริงแทน (Part 060-061) |
| การส่งอีเมลจริง/SMS/Push Notification | ฟังก์ชัน/method ภายในแอปของเราเอง (business logic) |
| นาฬิกาของระบบ (`timezone.now()`) — ใช้ `freezegun` แทนที่จะ mock ตรง ๆ | Django template rendering (`render()`) — เว้นแต่จะทดสอบแค่ view logic แยกจาก template โดยเฉพาะ |
| Payment gateway, third-party SDK | Django Form validation — ทดสอบด้วยข้อมูลจริงที่ผ่าน/ไม่ผ่าน validator |
| File system operations ที่ควบคุมไม่ได้ (เช่น cloud storage) | Serializer/Form ของโปรเจกต์ตัวเอง |
| ระบบ Queue/Message Broker ภายนอก (Celery broker, Redis) | Custom exception class ของโปรเจกต์ตัวเอง |

### 619.4 Refactor ตัวอย่างจาก 619.1 ให้เป็น Test ที่มีความหมายจริง

```python
# ✅ เวอร์ชันที่ดีกว่า: ใช้ test database จริงผ่าน pytest-django (Part 061)
import pytest
from django.urls import reverse

from blog.models import Post


@pytest.mark.django_db
def test_post_list_view_shows_only_published_posts(client):
    Post.objects.create(title="บทความที่เผยแพร่แล้ว", published=True)
    Post.objects.create(title="บทความฉบับร่าง", published=False)

    response = client.get(reverse("blog:post-list"))

    assert response.status_code == 200
    titles = [post.title for post in response.context["posts"]]
    assert "บทความที่เผยแพร่แล้ว" in titles
    assert "บทความฉบับร่าง" not in titles
```

test เวอร์ชันนี้**สั้นกว่า**, **อ่านง่ายกว่า**, และที่สำคัญที่สุดคือ **ทดสอบ
พฤติกรรมจริงที่ผู้ใช้สังเกตเห็นได้** (เห็นบทความที่เผยแพร่แล้ว ไม่เห็นฉบับร่าง)
แทนที่จะทดสอบว่า "เรียก method ชื่ออะไรบ้าง" — ถ้ามีบั๊กในการ filter จริง
test นี้จะจับได้ทันที ต่างจากเวอร์ชัน mock ทั้งหมดที่จับไม่ได้เลย

### 619.5 Checklist ก่อนตัดสินใจ Mock อะไรสักอย่าง

ก่อนเขียน `patch(...)` ทุกครั้ง ให้ถามตัวเองตามลำดับนี้:

1. **สิ่งที่กำลังจะ mock เป็นของภายนอกระบบจริง ๆ หรือไม่** (network, เวลา,
   บริการที่มีค่าใช้จ่าย/ผลข้างเคียงจริง)? ถ้าใช่ → mock ได้เลย
2. **ถ้าไม่ mock แล้ว test จะช้าเกินไปหรือไม่เสถียร (flaky) หรือไม่**? เช่น
   ต้องรอ network round-trip จริง → พิจารณา mock
3. **ถ้าใช้ของจริงแทน (เช่น test database) test จะซับซ้อนเกินความจำเป็น
   หรือไม่**? ถ้าของจริงตั้งค่าง่ายอยู่แล้ว (เช่น Django test database ที่
   `pytest-django` เตรียมให้ฟรี) **ไม่ต้อง mock**
4. **หลัง mock แล้ว test ยังคง assert ผลลัพธ์ที่มีความหมาย (ไม่ใช่แค่ตรวจว่า
   mock ถูกเรียก) หรือไม่**? ถ้า assertion ทั้งหมดเป็นแค่ `assert_called*`
   โดยไม่มีการตรวจสอบผลลัพธ์ปลายทางเลย ควรพิจารณาออกแบบ test ใหม่

> **สรุปสั้น ๆ ที่ควรจำไปตลอด**: **"Mock ที่ขอบ ไม่ mock ตรงกลาง"**
> (Mock at the boundaries, not in the middle) — ยิ่ง mock น้อยเท่าที่จำเป็น
> test ก็ยิ่งน่าเชื่อถือมากขึ้นเท่านั้น

---

## ขั้นตอนที่ 620: สรุปและแบบฝึกหัด

### 620.1 สรุปสิ่งที่ได้เรียนรู้ใน Part นี้

- ติดตั้งและใช้ `coverage.py`/`pytest-cov` วัด test coverage ของโปรเจกต์ทั้งแบบ
  terminal report และ HTML report ที่ไล่ดูเป็นบรรทัดได้
- ตั้งค่า `.coveragerc` (หรือ `[tool.coverage.*]` ใน `pyproject.toml`) เพื่อ
  `omit` ไฟล์ที่ไม่มีความหมายให้วัด เช่น migrations, `__init__.py`, entry point
  และเข้าใจความแตกต่างระหว่าง statement coverage กับ branch coverage
- ผูก coverage gate เข้ากับ CI ด้วย `--cov-fail-under=80` และเข้าใจอย่างลึกซึ้ง
  ว่า **coverage สูงไม่ได้แปลว่า test ดี** — assertion-free test สามารถทำให้
  coverage 100% ได้โดยไม่จับบั๊กอะไรเลยแม้แต่ตัวเดียว พร้อมรู้จัก mutation
  testing เป็นเครื่องมือตรวจสุขภาพ test suite เพิ่มเติม
- ใช้ `unittest.mock.Mock`/`MagicMock` สร้าง test double, กำหนด
  `return_value`/`side_effect`, และใช้ `spec`/`spec_set` ป้องกัน mock ที่
  "โกหก" ว่ามี interface ที่ไม่มีจริง
- ใช้ `patch` ทั้งแบบ decorator และ context manager พร้อมเข้าใจหลักการสำคัญ
  ที่สุดคือ **"patch where it's used, not where it's defined"**
- Mock การเรียก external API จริง ทั้งการส่งอีเมล (`send_mail`) และการเรียก
  Have I Been Pwned API จาก Part 035 (`requests.get`) ให้ครอบคลุมทุก branch
  รวมถึงกรณี API ล้มเหลว
- ใช้ `freezegun` ควบคุมเวลาของระบบระหว่าง test เพื่อทดสอบ logic ที่ผูกกับ
  วันที่/เวลาโดยไม่ต้องพึ่งเวลาปัจจุบันจริง และไม่เจอปัญหา time-bomb test
- ใช้ spy pattern ผ่าน `assert_called_with()`, `assert_called_once()`,
  `call_count`, `call_args_list` และ `autospec=True` เพื่อตรวจสอบพฤติกรรมของ
  โค้ดอย่างแม่นยำและปลอดภัยจากการพิมพ์ผิด
- ตระหนักถึงอันตรายของการ mock มากเกินไป และยึดหลัก **"mock ที่ขอบ ไม่ mock
  ตรงกลาง"** เพื่อให้ test ยังคงทดสอบพฤติกรรมจริงของระบบ ไม่ใช่แค่ทดสอบว่า
  mock ถูกเรียกถูกต้องหรือไม่

### 620.2 Checklist ก่อนไป Part ถัดไป

- [ ] รัน `pytest --cov=. --cov-report=term-missing` แล้วอ่านรายงานได้ว่าไฟล์
      ไหน/บรรทัดไหนยังไม่มี test คลุม
- [ ] สร้าง `.coveragerc` ที่ `omit` migrations, `__init__.py`, และไฟล์ entry
      point ของโปรเจกต์ `blog` สำเร็จ
- [ ] อธิบายความแตกต่างระหว่าง statement coverage กับ branch coverage ได้
      พร้อมยกตัวอย่างโค้ดที่ statement coverage 100% แต่ branch coverage ไม่ครบ
- [ ] ตั้ง `--cov-fail-under=80` ใน CI workflow และอธิบายได้ว่าทำไม coverage
      สูงไม่ได้รับประกันว่า test มีคุณภาพ
- [ ] เขียน `Mock`/`MagicMock` พร้อมกำหนด `return_value` และ `side_effect` ได้
- [ ] ใช้ `patch` decorator/context manager ได้ถูกต้องตามหลัก "patch where
      it's used" ไม่ใช่ "patch where it's defined"
- [ ] เขียน test mock การส่งอีเมล (`send_mail`) และการเรียก HIBP API
      (`requests.get`) ครอบคลุมทั้งกรณีสำเร็จและ API ล้มเหลวได้
- [ ] ใช้ `freezegun` ทดสอบ logic ที่ผูกกับวันที่/เวลาได้ โดยไม่ต้องพึ่งวันที่
      ปัจจุบันจริง
- [ ] ใช้ `assert_called_once_with()`, `call_count`, `call_args_list` ตรวจสอบ
      พฤติกรรมของ mock ได้อย่างละเอียด
- [ ] อธิบายได้ว่าเมื่อไหร่ "ควร" และ "ไม่ควร" mock อะไรสักอย่าง พร้อมยกตัวอย่าง
      test ที่ over-mock จนไม่มีความหมาย

### 620.3 แบบฝึกหัดท้ายบท

**แบบฝึกหัดที่ 1**: เพิ่ม `pytest-cov` เข้าไปในโปรเจกต์ `blog` ของคุณ สร้างไฟล์
`.coveragerc` ที่ `omit` migrations, `__init__.py`, `manage.py`,
`config/asgi.py`, `config/wsgi.py` ตามขั้นตอนที่ 612 แล้วรัน
`pytest --cov=. --cov-report=html` ดูรายงาน HTML ที่ได้ เลือกไฟล์ที่มี coverage
ต่ำที่สุด 1 ไฟล์ แล้วเขียน test เพิ่มจนกว่า coverage ของไฟล์นั้นจะเกิน 80%
(ห้ามใช้ assertion-free test ตามที่เตือนไว้ในขั้นตอนที่ 613.4)

**แบบฝึกหัดที่ 2**: เขียนฟังก์ชัน `notify_admin_of_new_signup(user)` ใน
`accounts/services.py` ที่เรียก `send_mail` ส่งอีเมลแจ้งแอดมินทุกครั้งที่มีคน
สมัครสมาชิกใหม่ จากนั้นเขียน test 2 แบบ: (1) แบบใช้ `django.core.mail.outbox`
ตามขั้นตอนที่ 616.1 และ (2) แบบใช้ `@patch("accounts.services.send_mail")`
ตามขั้นตอนที่ 616.2 พร้อม `assert_called_once_with()` ตรวจสอบ argument
ครบถ้วน แล้วเขียนอธิบายสั้น ๆ ว่า test สองแบบนี้ต่างกันอย่างไรและควรเลือกใช้
แบบไหนเมื่อไหร่

**แบบฝึกหัดที่ 3**: เขียน test ครบทุก branch ของ `PwnedPasswordValidator` จาก
Part 035 โดย mock `requests.get` ให้ครอบคลุม 4 กรณีตามขั้นตอนที่ 616.4: พบว่า
รหัสผ่านเคยรั่วไหล, ไม่พบ, API timeout แบบ `fail_open=True`, และ API timeout
แบบ `fail_open=False` จากนั้นเพิ่ม test เคสที่ 5: จำลองว่า API ตอบกลับด้วย
`status_code` 500 (ใช้ `response.raise_for_status.side_effect =
requests.HTTPError("Server Error")`) แล้วตรวจสอบว่า validator จัดการ error
กรณีนี้ได้ถูกต้องเหมือนกรณี timeout หรือไม่

**แบบฝึกหัดที่ 4 (ขั้นสูง)**: สร้างโมเดล `Coupon` ที่มี field `code`,
`discount_percent`, `valid_from`, `valid_until` และ method `is_redeemable()`
ที่คืนค่า `True` เฉพาะเมื่อเวลาปัจจุบันอยู่ในช่วง `valid_from`-`valid_until`
เท่านั้น เขียน test ด้วย `freezegun` ให้ครบ 3 กรณี (ก่อนเริ่ม, ระหว่างใช้งาน,
หลังหมดอายุ) ตามขั้นตอนที่ 617 จากนั้นค้นหาโค้ด test ใด ๆ ในโปรเจกต์ `blog`
ของคุณเองที่ over-mock ตามอาการในตารางขั้นตอนที่ 619.2 (หรือจงใจเขียนตัวอย่าง
ที่ over-mock ขึ้นมาเองถ้ายังไม่มี) แล้ว refactor ให้ใช้ test database จริง
ผ่าน `@pytest.mark.django_db` แทน พร้อมอธิบายว่า test เวอร์ชันใหม่ตรวจจับบั๊ก
ได้ดีกว่าเวอร์ชันเดิมอย่างไร

### 620.4 คำถามที่พบบ่อย (FAQ)

**Q: Coverage 100% แปลว่าโปรเจกต์ไม่มีบั๊กเลยใช่ไหม?**
A: ไม่ใช่เด็ดขาด ตามที่อธิบายละเอียดในขั้นตอนที่ 613.4 coverage บอกได้แค่ว่า
"บรรทัดนี้เคยถูกรัน" ไม่ได้บอกว่า "ผลลัพธ์ถูกต้อง" test ที่ไม่มี assertion
เลยก็ทำให้ coverage เป็น 100% ได้ทั้งที่ไม่จับบั๊กอะไรเลยแม้แต่ตัวเดียว
coverage เป็นเครื่องมือช่วย**หาจุดที่ยังไม่มี test เลย** ไม่ใช่เครื่องมือวัด
คุณภาพของ test ที่มีอยู่

**Q: ควรใช้ `unittest.mock` หรือปลั๊กอิน `pytest-mock` (fixture `mocker`)?**
A: ทั้งสองใช้ `unittest.mock` เป็นเครื่องยนต์เบื้องหลังเหมือนกันทุกประการ
`pytest-mock` แค่ห่อ API ให้เข้ากับสไตล์ fixture ของ `pytest` มากขึ้น (เช่น
`mocker.patch(...)` แทน `@patch(...)`) และ**auto-cleanup ให้อัตโนมัติ**โดย
ไม่ต้องพึ่ง decorator/context manager หลักการ "patch where it's used" ยังคง
ใช้เหมือนกันทุกประการไม่ว่าจะเลือกใช้ตัวไหน — Part นี้เลือกสอน `unittest.mock`
ตรง ๆ ก่อนเพราะเป็นพื้นฐานที่ทุกเครื่องมืออื่นสร้างต่อยอดมาจากมัน เมื่อเข้าใจ
แล้วจะเปลี่ยนไปใช้ `pytest-mock` ทีหลังก็ทำได้ทันทีโดยแทบไม่ต้องเรียนรู้อะไรใหม่

**Q: ทำไม `patch("module.func")` กับ `patch("other_module.func")` (คนละที่)
ถึงให้ผลต่างกัน ทั้งที่เป็นฟังก์ชันเดียวกัน?**
A: เพราะ `patch` ไม่ได้แก้ไข "ฟังก์ชันต้นฉบับ" แต่แก้ไข **"ชื่อที่ผูกไว้ใน
namespace ของโมดูลที่ระบุ"** เท่านั้น ถ้าไฟล์ `A.py` ทำ
`from B import func` มันจะสร้างชื่อ `func` ขึ้นใหม่ในโมดูล `A` เอง (คนละชื่อ
กับ `B.func` แม้จะชี้ไปที่ object เดียวกันตอน import) การ `patch("A.func")`
กับ `patch("B.func")` จึงเป็นการแก้ไข "ชื่อ" คนละตัวกัน มีแค่ `patch("A.func")`
เท่านั้นที่ส่งผลต่อโค้ดในไฟล์ `A.py` — รายละเอียดเต็มอยู่ในขั้นตอนที่ 615.6

**Q: `freezegun` ทำให้ test ช้าลงมากไหม เพราะต้อง patch ทั้งระบบเวลา?**
A: แทบไม่มีผลกระทบที่สังเกตได้ในทางปฏิบัติ `freeze_time` เพิ่ม overhead
ระดับไมโครวินาทีต่อการเรียกแต่ละครั้งเท่านั้น เทียบกับประโยชน์ที่ได้ (test
ที่ผลลัพธ์คงที่ 100% ไม่เป็น time-bomb test ตามขั้นตอนที่ 617.1) ถือว่าคุ้มค่า
มากเสมอสำหรับ logic ใด ๆ ที่เกี่ยวข้องกับวันที่/เวลา

**Q: ถ้าเขียน test แบบ mock ทุกอย่าง (over-mock) ไปแล้วจำนวนมาก ควรลบทิ้ง
ทั้งหมดเลยไหม?**
A: ไม่จำเป็นต้องลบทิ้งทันที แต่ควรจัดลำดับความสำคัญ refactor ตาม**ความเสี่ยง**
ของแต่ละส่วน — เริ่มจาก business logic ที่สำคัญที่สุด (เช่น การคำนวณเงิน,
permission check, authentication) ให้ใช้ test database จริงก่อน ส่วนพื้นที่ที่
ความเสี่ยงต่ำกว่าอาจปล่อยไว้ก่อนได้ ที่สำคัญคือ**ตั้งแต่วันนี้เป็นต้นไป** ให้ยึด
หลักการ "mock ที่ขอบ ไม่ mock ตรงกลาง" จากขั้นตอนที่ 619.3 เมื่อเขียน test ใหม่
ทุกครั้ง เพื่อไม่ให้ปัญหาสะสมเพิ่มขึ้นอีก

---

## เตรียมตัวสำหรับ Part ถัดไป

**Part 063: Factory Boy และ Test Data Generation** จะพาคุณไปแก้ปัญหาที่คุณ
น่าจะเริ่มสังเกตเห็นแล้วจากการเขียน test จำนวนมากใน Part นี้และ Part ก่อนหน้า
นั่นคือการสร้างข้อมูลทดสอบ (`CustomUser.objects.create_user(...)`,
`Post.objects.create(...)`) ที่ต้องพิมพ์ field ซ้ำ ๆ ในหลายสิบ test จนโค้ด
test เยิ่นเย้อและบำรุงรักษายาก คุณจะได้เรียนรู้ **Factory Boy** library ที่
ช่วยสร้าง object จำลองแบบสมจริงได้ในบรรทัดเดียว ผสานกับ `Faker` เพื่อสุ่ม
ข้อมูล (ชื่อ, อีเมล, วันที่) ที่สมจริงแบบไม่ต้อง hardcode ทุกครั้ง, การสร้าง
`SubFactory` สำหรับ foreign key relationship, `Sequence` สำหรับข้อมูลที่ต้อง
ไม่ซ้ำกัน (เช่น username), และเทคนิค `Trait` สำหรับสร้าง object ที่มีเงื่อนไข
พิเศษ (เช่น "user ที่เป็น premium" หรือ "post ที่เผยแพร่แล้ว")

เนื่องจาก Factory Boy จะถูกใช้ร่วมกับเทคนิค mocking ที่เพิ่งเรียนไปใน Part นี้
บ่อยมาก (เช่น สร้าง user ปลอมด้วย factory แล้ว mock การส่งอีเมลต้อนรับตาม
ขั้นตอนที่ 616 เพื่อไม่ให้ test ช้าลงจากการสร้างข้อมูลจำนวนมาก) ให้ทบทวน
`unittest.mock`, `patch`, และหลักการ "mock ที่ขอบ ไม่ mock ตรงกลาง" จาก
Part นี้ให้แม่นก่อนไปต่อ เพราะทั้งสองเรื่องจะทำงานเสริมกันตลอดในทุก Part
ของ Phase Testing ที่เหลือ
