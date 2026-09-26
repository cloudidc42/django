# Part 065: Continuous Testing และ Code Quality Tools

> **ขั้นตอนที่ 641-650 ของหลักสูตร** | Phase 7: Testing & Quality Assurance (Part สุดท้ายของ Phase นี้)
>
> ตลอด Part 059-064 คุณเรียนรู้วิธี**เขียน test** ให้ครอบคลุมและเชื่อถือได้ — ตั้งแต่
> `unittest`/`TestCase` พื้นฐาน, `pytest-django`, การวัด coverage, การ mock, การสร้าง
> ข้อมูลทดสอบด้วย Factory Boy ไปจนถึง integration test ผ่านเบราว์เซอร์จริงด้วย Selenium
> แต่การมี test suite ที่ดีเพียงอย่างเดียวยังไม่พอสำหรับทีมมืออาชีพ เพราะยังมีคำถามอีก
> ชั้นหนึ่งที่ test suite ตอบไม่ได้: โค้ดของคุณ**สะอาด**แค่ไหน (linting), **type ถูกต้อง**
> หรือไม่ (type checking), มี**ช่องโหว่ความปลอดภัย**ที่มองไม่เห็นด้วยตาหรือไม่ (security
> scanning) และที่สำคัญที่สุดคือ — test suite ที่ผ่านหมด 100% นั้น **ตรวจจับบั๊กได้จริง
> หรือแค่วิ่งผ่านเฉย ๆ** (mutation testing) Part นี้จะพาคุณประกอบเครื่องมือทั้งหมดเข้าด้วย
> กันเป็น **Quality Gate** อัตโนมัติที่รันทุกครั้งก่อน commit ด้วย `pre-commit`, มาตรฐาน
> เดียวกันทั้งทีมผ่าน `Makefile`/`tox`, และปิดท้ายด้วยการสรุปภาพรวมทั้ง **Phase 7** ก่อน
> ก้าวเข้าสู่ **Phase 8: Performance & Caching**

---

## สารบัญของ Part นี้

- ขั้นตอนที่ 641: Pre-commit Hooks ด้วย framework `pre-commit` — รัน check อัตโนมัติก่อน commit ทุกครั้ง
- ขั้นตอนที่ 642: Linting ด้วย Ruff — เร็วกว่า Flake8/Pylint มาก ตั้งค่ากฎที่เหมาะกับ Django
- ขั้นตอนที่ 643: Type Checking ด้วย `mypy` + `django-stubs` สำหรับ Django โดยเฉพาะ
- ขั้นตอนที่ 644: Code Formatting อัตโนมัติด้วย `ruff format`
- ขั้นตอนที่ 645: ทบทวน django-debug-toolbar เป็นเครื่องมือ quality ระหว่างพัฒนา
- ขั้นตอนที่ 646: Static Analysis ด้านความปลอดภัยด้วย `bandit`
- ขั้นตอนที่ 647: Test-Driven Development (TDD) — Red-Green-Refactor cycle
- ขั้นตอนที่ 648: Mutation Testing ด้วย `mutmut` — ทดสอบว่าคุณภาพ test สูงจริงหรือแค่ coverage สูง
- ขั้นตอนที่ 649: Makefile/tox มาตรฐานสำหรับคำสั่งพัฒนาที่ทีมใช้ร่วมกัน
- ขั้นตอนที่ 650: สรุป Phase 7 ทั้งหมด (Part 059-065) + Quiz + แบบฝึกหัดใหญ่ปิดท้าย Phase + คำนำสู่ Phase 8

---

## ขั้นตอนที่ 641: Pre-commit Hooks ด้วย framework `pre-commit`

### 641.1 ปัญหาที่ pre-commit แก้: "ลืมรัน check ก่อน push"

ตลอด Phase 7 คุณมีเครื่องมือตรวจสอบคุณภาพโค้ดมากมาย (test, coverage, และในอีกไม่กี่
ขั้นตอนข้างหน้าคือ linter, type checker, security scanner) แต่ปัญหาคลาสสิกของทีมพัฒนา
ทุกทีมคือ: **เครื่องมือเหล่านี้มีประโยชน์ก็ต่อเมื่อมีคนรันมันจริง ๆ** ในความเป็นจริง
นักพัฒนามักลืมรัน `ruff check` หรือ `pytest` ก่อน `git commit` โดยเฉพาะตอนรีบ ทำให้
โค้ดที่มีปัญหาเล็ดลอดเข้า repository และไปพังที่ CI/CD pipeline แทน ซึ่งช้ากว่าและ
เสียเวลารอผลมากกว่าการจับได้ตั้งแต่บนเครื่องตัวเอง

**Git hooks** คือกลไกของ Git ที่ให้รันสคริปต์อัตโนมัติ ณ จุดต่าง ๆ ของ workflow เช่น
ก่อน commit (`pre-commit`), ก่อน push (`pre-push`) แต่การเขียน hook สคริปต์เองมีปัญหา:
ต้องเขียนเองทุกโปรเจกต์ ไม่มีเวอร์ชัน ไม่แชร์ระหว่างทีมง่าย ๆ (เพราะโฟลเดอร์ `.git/hooks/`
ไม่ถูก commit ขึ้น Git) — framework **[`pre-commit`](https://pre-commit.com/)** (เขียน
ด้วย Python โดย Anthony Sottile) แก้ปัญหานี้ทั้งหมดด้วยไฟล์ config ที่ commit ได้จริง

### 641.2 ติดตั้ง `pre-commit`

```bash
pip install pre-commit

# ตรวจสอบเวอร์ชัน
pre-commit --version
```

เพิ่มเข้า `requirements/dev.txt` (ตามโครงสร้างที่วางไว้ตั้งแต่ Part 003):

```
# requirements/dev.txt (เพิ่มส่วนนี้)
pre-commit==4.0.1
```

### 641.3 สร้างไฟล์ `.pre-commit-config.yaml`

ไฟล์นี้ต้องอยู่ที่ **root ของโปรเจกต์** (ระดับเดียวกับ `manage.py`) และเป็นไฟล์ที่
**ต้อง commit ขึ้น Git เสมอ** เพราะเป็นสัญญาร่วมของทั้งทีมว่าจะรัน check อะไรบ้าง:

```yaml
# .pre-commit-config.yaml
repos:
  # --- Hooks พื้นฐานที่ pre-commit ดูแลเอง ---
  - repo: https://github.com/pre-commit/pre-commit-hooks
    rev: v5.0.0
    hooks:
      - id: trailing-whitespace       # ลบช่องว่างท้ายบรรทัดที่ไม่จำเป็น
      - id: end-of-file-fixer         # บังคับให้ทุกไฟล์จบด้วย newline เดียว
      - id: check-yaml                # ตรวจสอบว่าไฟล์ .yml/.yaml ไม่ syntax error
      - id: check-json                # ตรวจสอบไฟล์ .json
      - id: check-toml                # ตรวจสอบไฟล์ .toml (เช่น pyproject.toml)
      - id: check-added-large-files   # กันไม่ให้ commit ไฟล์ใหญ่เกิน 500KB โดยไม่ตั้งใจ
        args: ["--maxkb=500"]
      - id: check-merge-conflict      # กันไม่ให้ commit เครื่องหมาย <<<<<<< ค้างอยู่
      - id: detect-private-key        # ตรวจจับ private key ที่หลุดเข้ามาในโค้ด
      - id: mixed-line-ending         # บังคับ line ending ให้เป็นมาตรฐานเดียวกัน (LF)
        args: ["--fix=lf"]

  # --- Ruff: linter + formatter (รายละเอียดในขั้นตอนที่ 642, 644) ---
  - repo: https://github.com/astral-sh/ruff-pre-commit
    rev: v0.7.4
    hooks:
      - id: ruff
        args: ["--fix", "--exit-non-zero-on-fix"]
      - id: ruff-format

  # --- Bandit: security static analysis (รายละเอียดในขั้นตอนที่ 646) ---
  - repo: https://github.com/PyCQA/bandit
    rev: 1.8.0
    hooks:
      - id: bandit
        args: ["-c", "pyproject.toml"]
        additional_dependencies: ["bandit[toml]"]
        exclude: ^.*/(migrations|tests)/.*$

  # --- mypy: type checking ต้องใช้ local hook เพราะต้องเห็น django-stubs
  #     และ dependency ทั้งหมดของโปรเจกต์ (รายละเอียดในขั้นตอนที่ 643) ---
  - repo: local
    hooks:
      - id: mypy
        name: mypy (type checking)
        entry: mypy
        language: system
        types: [python]
        pass_filenames: false
        args: ["blog/", "config/"]
        exclude: ^.*/migrations/.*$
```

### 641.4 ติดตั้ง Hook เข้ากับ Git และรันครั้งแรก

คำสั่งเดียวที่มือใหม่ลืมบ่อยที่สุด: การมีไฟล์ `.pre-commit-config.yaml` **ไม่ได้แปลว่า
Git hook ถูกเปิดใช้งานอัตโนมัติ** ต้องรันคำสั่ง `pre-commit install` เพื่อ "เสียบ" hook
เข้ากับ `.git/hooks/pre-commit` ของ repository นั้นก่อนเสมอ (ทำครั้งเดียวต่อเครื่อง
ต่อ repository ที่ clone มาใหม่):

```bash
# เสียบ hook เข้ากับ .git/hooks/pre-commit ของโปรเจกต์นี้
pre-commit install

# ผลลัพธ์ที่คาดหวัง:
# pre-commit installed at .git/hooks/pre-commit

# ทดสอบรัน hook ทั้งหมดกับทุกไฟล์ในโปรเจกต์ (ไม่ต้องรอ commit จริง)
pre-commit run --all-files
```

ตัวอย่างผลลัพธ์เมื่อรัน `pre-commit run --all-files` ครั้งแรก:

```
trim trailing whitespace.................................................Passed
fix end of files.........................................................Passed
check yaml................................................................Passed
check json.................................................................Passed
check toml.................................................................Passed
check for added large files..............................................Passed
check for merge conflicts.................................................Passed
detect private key........................................................Passed
mixed line ending.........................................................Passed
ruff......................................................................Failed
- hook id: ruff
- files were modified by this hook

blog/models.py:12:1: F401 [*] `django.utils.timezone` imported but unused
Found 1 error (1 fixed, 0 remaining).

ruff-format................................................................Passed
bandit......................................................................Passed
mypy (type checking).......................................................Passed
```

จากนี้ไป **ทุกครั้งที่คุณรัน `git commit`** Git จะรัน hook ทั้งหมดนี้อัตโนมัติก่อนยอมให้
commit สำเร็จ ถ้า hook ไหนแก้ไฟล์ให้ (เช่น `ruff --fix` หรือ `end-of-file-fixer`)
commit จะถูก**ยกเลิกครั้งแรก** เพื่อให้คุณตรวจสอบการแก้ไขนั้นก่อน แล้ว `git add` ไฟล์
ที่ถูกแก้ซ้ำแล้ว commit อีกครั้งจึงจะผ่าน — นี่คือพฤติกรรมที่ตั้งใจ ไม่ใช่บั๊ก

### 641.5 การข้าม Hook (และทำไมควรใช้อย่างระมัดระวังที่สุด)

```bash
# ข้าม pre-commit hook ทั้งหมดสำหรับ commit นี้ครั้งเดียว (ใช้เฉพาะกรณีฉุกเฉินจริง ๆ)
git commit -m "fix: hotfix urgent" --no-verify

# ข้าม hook เฉพาะบางตัว (ระบุผ่าน environment variable)
SKIP=mypy git commit -m "wip: กำลังปรับ type hints อยู่ ยังไม่เสร็จ"
```

**กฎของทีมมืออาชีพ**: `--no-verify` ควรใช้เป็น**ข้อยกเว้น**เท่านั้น (เช่น hotfix
production ที่ CI จะตรวจสอบซ้ำอยู่แล้วก่อน merge) ไม่ใช่พฤติกรรมปกติ เพราะ pre-commit
ที่ถูกข้ามบ่อย ๆ เท่ากับไม่มี quality gate เลย ทีมที่จริงจังมักตั้งค่า CI (ขั้นตอนที่ 649
และ Part 088) ให้รัน `pre-commit run --all-files` ซ้ำอีกครั้งบนเซิร์ฟเวอร์เสมอ เพื่อ
ป้องกันกรณีที่มีคน `--no-verify` หลุดเข้า main branch มาได้

### 641.6 อัปเดตเวอร์ชันของ Hook ให้ทันสมัยอยู่เสมอ

```bash
# อัปเดตทุก hook ในไฟล์ config ให้เป็นเวอร์ชันล่าสุดที่มี tag บน GitHub
pre-commit autoupdate

# รันซ้ำเพื่อให้แน่ใจว่าเวอร์ชันใหม่ยังทำงานถูกต้องกับโปรเจกต์
pre-commit run --all-files
```

ควรรัน `pre-commit autoupdate` เป็นประจำ (เช่น เดือนละครั้ง หรือใช้ Dependabot/Renovate
สร้าง pull request อัตโนมัติเมื่อมีเวอร์ชันใหม่ — รายละเอียดเรื่อง dependency automation
จะอยู่ใน Part 088) เพื่อให้ได้กฎ lint และ security check ล่าสุดเสมอ

---

## ขั้นตอนที่ 642: Linting ด้วย Ruff

### 642.1 ทำไมต้องมี Linter และทำไมต้อง Ruff

**Linter** คือเครื่องมือที่วิเคราะห์โค้ดแบบ static (ไม่ต้องรันโปรแกรมจริง) เพื่อหา
ปัญหาเชิงสไตล์ (style), ข้อผิดพลาดที่มักเกิดจริง (เช่น import ที่ไม่ได้ใช้, ตัวแปรที่
ประกาศซ้ำ), และแนวปฏิบัติที่ไม่ดี ก่อนหน้านี้วงการ Python ใช้ **Flake8** และ **Pylint**
เป็นมาตรฐาน แต่ทั้งสองตัวเขียนด้วย Python ล้วน ทำให้ช้าเมื่อโปรเจกต์ใหญ่ขึ้น

**[Ruff](https://docs.astral.sh/ruff/)** จาก **Astral** (บริษัทเดียวกับที่สร้าง `uv`
ที่คุณติดตั้งไปแล้วใน Part 003) เขียนด้วย **Rust** และรวมกฎของ Flake8 กว่า 800+ กฎ
บวกกับ plugin ยอดนิยมกว่า 50 ตัว (isort, pyupgrade, flake8-bugbear, flake8-django
ฯลฯ) ไว้ในไบนารีเดียว โดยไม่ต้องติดตั้ง plugin แยก:

| คุณสมบัติ | Flake8 | Pylint | Ruff |
|---|---|---|---|
| ภาษาที่เขียน | Python | Python | Rust |
| ความเร็ว (โปรเจกต์ ~50,000 บรรทัด) | ~5-10 วินาที | ~30-60 วินาที | ~0.1-0.3 วินาที |
| จำนวนกฎในตัว | น้อย (ต้องใช้ plugin เพิ่ม) | เยอะมาก | เยอะมาก (รวม plugin ดังเข้ามาแล้ว) |
| Auto-fix ในตัว | ❌ (ต้องพึ่ง autopep8/autoflake) | ❌ | ✅ (`--fix`) |
| รวม formatter ในตัว | ❌ | ❌ | ✅ (`ruff format`, ขั้นตอนที่ 644) |
| รองรับ Django-specific rules | ผ่าน plugin `flake8-django` | ผ่าน plugin แยก | ✅ ในตัว (`DJ` rules) |
| Config file | `.flake8`, `setup.cfg` | `.pylintrc` | `pyproject.toml` (ไฟล์เดียวกับที่ใช้ทั้งโปรเจกต์) |

ด้วยความเร็วระดับนี้ Ruff จึงรันเป็น pre-commit hook ได้แบบไม่รู้สึกหน่วงเลย (เทียบกับ
Pylint ที่หลายทีมเลี่ยงใช้เป็น pre-commit hook เพราะช้าเกินไปสำหรับทุก commit)

### 642.2 ติดตั้ง Ruff

```bash
pip install ruff

# ตรวจสอบเวอร์ชัน
ruff --version
```

### 642.3 ตั้งค่า Ruff ใน `pyproject.toml`

```toml
# pyproject.toml
[tool.ruff]
line-length = 100
target-version = "py312"
exclude = [
    "migrations",
    ".venv",
    "venv",
    "staticfiles",
]

[tool.ruff.lint]
select = [
    "E",    # pycodestyle errors
    "W",    # pycodestyle warnings
    "F",    # Pyflakes (import ไม่ใช้, ตัวแปรไม่ใช้, ฯลฯ)
    "I",    # isort (จัดเรียง import อัตโนมัติ)
    "N",    # pep8-naming (ชื่อ class/function ตามธรรมเนียม PEP 8)
    "UP",   # pyupgrade (แนะนำ syntax ใหม่ของ Python ที่ทันสมัยกว่า)
    "B",    # flake8-bugbear (จับ bug pattern ที่พบบ่อย)
    "A",    # flake8-builtins (เตือนเมื่อตั้งชื่อตัวแปรทับ builtin เช่น `id`, `list`)
    "C4",   # flake8-comprehensions (แนะนำวิธีเขียน comprehension ที่ดีกว่า)
    "DJ",   # flake8-django (กฎเฉพาะของ Django — ดูรายละเอียดด้านล่าง)
    "SIM",  # flake8-simplify (แนะนำวิธีเขียน logic ให้กระชับขึ้น)
    "RUF",  # กฎเฉพาะของ Ruff เอง
]
ignore = [
    "E501",  # บรรทัดยาวเกิน — ให้ ruff format เป็นคนจัดการแทน ไม่ต้องเตือนซ้ำ
]

[tool.ruff.lint.per-file-ignores]
# ไฟล์ migration ถูก Django generate อัตโนมัติ ไม่ควรบังคับ style เดียวกับโค้ดที่เขียนเอง
"*/migrations/*.py" = ["E501", "N806"]
# ไฟล์ทดสอบอนุญาตให้ import ที่ไม่ได้ใช้โดยตรง (fixture) และ assert แบบง่าย
"*/tests/*.py" = ["S101"]
"conftest.py" = ["S101"]

[tool.ruff.lint.isort]
known-first-party = ["blog", "config"]
```

### 642.4 ตัวอย่างกฎ `DJ` (flake8-django) ที่สำคัญที่สุดสำหรับ Django

| Rule | ชื่อ | ปัญหาที่จับ |
|---|---|---|
| `DJ001` | Avoid nullable string field | `CharField`/`TextField` ที่ตั้ง `null=True` (ควรใช้ `blank=True` แทน เพราะ Django แนะนำไม่ให้สตริงมีค่าว่างสองแบบคือ `NULL` และ `""`) |
| `DJ006` | Avoid `exclude` in ModelForm | `ModelForm.Meta.exclude` เสี่ยงต่อ mass assignment เมื่อเพิ่ม field ใหม่ในอนาคตแล้วลืมอัปเดต `exclude` |
| `DJ007` | Avoid `fields = "__all__"` | เช่นเดียวกับ `exclude` — ควรระบุ `fields` แบบเจาะจงเสมอ |
| `DJ008` | Model missing `__str__` | Model ที่ไม่มี `__str__()` จะแสดงเป็น `Post object (1)` ใน Django Admin ซึ่งใช้งานยาก |
| `DJ012` | Model field/method order | แนะนำให้เรียง field, `Meta`, `__str__`, method อื่น ๆ ตามธรรมเนียมที่อ่านง่าย |

ตัวอย่างโค้ดที่ Ruff จะเตือน (และวิธีแก้):

```python
# blog/models.py — ❌ ก่อนแก้ (Ruff เตือน DJ001, DJ008)
class Post(models.Model):
    title = models.CharField(max_length=200, null=True)  # DJ001
    slug = models.SlugField(unique=True)
    # ... ไม่มี __str__() ...  # DJ008
```

```python
# blog/models.py — ✅ หลังแก้
class Post(models.Model):
    title = models.CharField(max_length=200, blank=True)  # ใช้ blank=True แทน null=True
    slug = models.SlugField(unique=True)

    def __str__(self) -> str:
        return self.title
```

### 642.5 คำสั่งรัน Ruff ที่ใช้บ่อยที่สุด

```bash
# ตรวจสอบทั้งโปรเจกต์ (แสดงปัญหา ไม่แก้ให้)
ruff check .

# ตรวจสอบและแก้ไขปัญหาที่แก้ได้อัตโนมัติทันที (เช่น import ไม่เรียง, import ไม่ใช้)
ruff check . --fix

# ตรวจสอบเฉพาะไฟล์หรือโฟลเดอร์เดียว
ruff check blog/

# แสดงรายละเอียดกฎแต่ละตัวว่าหมายถึงอะไร (สำหรับกฎที่ยังไม่คุ้น)
ruff rule DJ001

# ดูรายการกฎทั้งหมดที่เปิดใช้งานอยู่ในโปรเจกต์นี้
ruff check . --show-settings | head -30
```

ตัวอย่างผลลัพธ์เมื่อพบปัญหา:

```
blog/forms.py:8:9: DJ007 Avoid using `fields = "__all__"` in ModelForm
  |
8 |         fields = "__all__"
  |         ^^^^^^^^^^^^^^^^^^ DJ007
  |
  = help: Replace `"__all__"` with a list of fields

Found 1 error.
```

### 642.6 เชื่อม Ruff เข้ากับ VS Code (ต่อยอดจาก Part 001)

ใน Part 001 ขั้นตอนที่ 5 คุณติดตั้ง Extension **Ruff** (โดย Astral) ไปแล้ว และตั้ง
`editor.defaultFormatter` เป็น `charliermarsh.ruff` ไว้แล้ว ทำให้ทุกครั้งที่ save ไฟล์
VS Code จะรัน Ruff format และแสดงเส้นใต้สีแดง/เหลืองใต้โค้ดที่ผิดกฎทันทีโดยไม่ต้อง
รอ `pre-commit` เลย — นี่คือเหตุผลที่หลักสูตรนี้แนะนำ Ruff ตั้งแต่ Part แรก เพื่อให้คุณ
คุ้นชินกับ feedback แบบทันทีมาตลอดทาง

---

## ขั้นตอนที่ 643: Type Checking ด้วย `mypy` + `django-stubs`

### 643.1 ทำไม Type Checking ถึงสำคัญ และทำไม Django ต้องมี stub แยก

Python เป็นภาษา **dynamically typed** โดยธรรมชาติ Type Hints (ที่ทบทวนไปใน Part 002)
ช่วยให้ editor และเครื่องมือ static analysis เข้าใจว่าตัวแปรแต่ละตัวควรเป็นชนิดอะไร
แต่การมี type hint เฉย ๆ ไม่ได้แปลว่ามันถูก**ตรวจสอบจริง** — **`mypy`** คือเครื่องมือ
ที่อ่าน type hint ทั้งหมดในโค้ด แล้วตรวจสอบว่าตรรกะสอดคล้องกันหรือไม่ โดยไม่ต้องรัน
โปรแกรมจริงเลย (คล้าย compiler ของภาษาที่ type แน่นอย่าง Java/TypeScript)

ปัญหาคือ Django ใช้ **metaclass และ dynamic attribute** จำนวนมาก (เช่น
`Post.objects.filter(...)` ที่ `objects` ถูกสร้างขึ้นเองแบบไดนามิกโดย Django ORM)
ทำให้ mypy ธรรมดาไม่เข้าใจโครงสร้างเหล่านี้เลย จึงต้องใช้ **[`django-stubs`](https://github.com/typeddjango/django-stubs)**
ซึ่งเป็นชุด "type stub" (ไฟล์ `.pyi` ที่บอก type ของโค้ด Django โดยไม่ต้องแก้ source
จริง) พร้อม mypy plugin ที่สอน mypy ให้เข้าใจ Django ORM, `Model.Meta`, และ QuerySet
ได้อย่างถูกต้อง

### 643.2 ติดตั้ง mypy และ django-stubs

```bash
pip install mypy django-stubs django-stubs-ext
```

เพิ่มเข้า `requirements/dev.txt`:

```
# requirements/dev.txt (เพิ่มส่วนนี้)
mypy==1.13.0
django-stubs==5.1.1
django-stubs-ext==5.1.1
```

### 643.3 ตั้งค่าใน `pyproject.toml`

```toml
# pyproject.toml
[tool.mypy]
python_version = "3.12"
plugins = ["mypy_django_plugin.main"]
mypy_path = "."
exclude = [
    "migrations/",
    ".venv/",
]
# เข้มงวดระดับกลาง — ยังไม่ถึง --strict ทั้งหมดเพื่อไม่ให้เรียนรู้ยากเกินไปในช่วงแรก
check_untyped_defs = true
disallow_untyped_defs = false
warn_unused_ignores = true
warn_redundant_casts = true
no_implicit_optional = true

[tool.django-stubs]
django_settings_module = "config.settings"
```

**สังเกต**: `django_settings_module` ต้องชี้ไปที่ settings module จริงของโปรเจกต์
(ตัวเดียวกับที่ตั้งใน `pytest.ini` ตอน Part 061 ขั้นตอนที่ 602) เพราะ `django-stubs`
plugin ต้อง import settings เพื่อรู้ว่าโปรเจกต์มี app และ model อะไรบ้างระหว่างตรวจสอบ

### 643.4 ตัวอย่างการเขียน Type Hint ใน Django ที่ mypy ตรวจสอบได้

```python
# blog/models.py
from django.db import models
from django.urls import reverse


class Post(models.Model):
    title = models.CharField(max_length=200)
    slug = models.SlugField(unique=True)
    content = models.TextField()
    view_count = models.PositiveIntegerField(default=0)
    is_free = models.BooleanField(default=True)

    class Meta:
        ordering = ["-id"]

    def __str__(self) -> str:
        return self.title

    def get_absolute_url(self) -> str:
        return reverse("blog:post_detail", kwargs={"slug": self.slug})

    def increment_view_count(self) -> None:
        self.view_count += 1
        self.save(update_fields=["view_count"])

    @property
    def reading_time_minutes(self) -> int:
        word_count = len(self.content.split())
        return max(1, round(word_count / 200))
```

```python
# blog/views.py
from django.http import HttpRequest, HttpResponse
from django.shortcuts import get_object_or_404, render

from blog.models import Post


def post_detail(request: HttpRequest, slug: str) -> HttpResponse:
    post = get_object_or_404(Post, slug=slug)
    post.increment_view_count()
    return render(request, "blog/post_detail.html", {"post": post})
```

รันตรวจสอบ:

```bash
mypy blog/
```

### 643.5 ตัวอย่าง Error ที่ mypy จับได้จริง และวิธีแก้

| Error ที่ mypy แสดง | สาเหตุ | วิธีแก้ |
|---|---|---|
| `error: Argument "slug" to "get_object_or_404" has incompatible type "int"; expected "str"` | ส่ง type ผิดเข้าฟังก์ชัน | แก้ให้ตรงกับ signature จริง |
| `error: "Post" has no attribute "publish_date"` | พิมพ์ชื่อ field ผิด หรือ field ถูกลบไปแล้วแต่โค้ดยังอ้างอยู่ | ตรวจสอบชื่อ field ใน `models.py` ให้ตรงกัน |
| `error: Item "None" of "Optional[Post]" has no attribute "title"` | ใช้ `Post.objects.filter(...).first()` (คืนค่า `Post \| None`) แล้วเรียก attribute ทันทีโดยไม่เช็ค `None` ก่อน | เพิ่ม `if post is not None:` หรือใช้ `get_object_or_404` แทน |
| `error: Need type annotation for "posts"` | ประกาศ list/dict ว่างโดยไม่มี type hint กำกับ | เขียน `posts: list[Post] = []` |

ตัวอย่างบั๊กจริงที่ mypy ช่วยจับได้ก่อนขึ้น production:

```python
# ❌ โค้ดที่ผ่าน test ทุกตัวได้ (เพราะ test มักมีข้อมูลอยู่เสมอ) แต่มีบั๊กแฝง
def get_featured_post() -> Post:
    return Post.objects.filter(is_featured=True).first()
    # mypy เตือนทันที: Incompatible return value type
    # (got "Post | None", expected "Post")
    # เพราะถ้าไม่มีบทความ featured เลย .first() จะคืน None แต่ signature สัญญาว่าคืน Post เสมอ
```

```python
# ✅ แก้ไขให้ signature ตรงกับความเป็นจริง
def get_featured_post() -> Post | None:
    return Post.objects.filter(is_featured=True).first()
```

### 643.6 การรัน mypy เป็น pre-commit hook แบบ `local`

สังเกตจากขั้นตอนที่ 641.3 ว่า hook ของ mypy ใช้ `repo: local` แทนที่จะดึงจาก
`https://github.com/pre-commit/mirrors-mypy` แบบมาตรฐาน เหตุผลคือ **mypy ต้อง
เห็น dependency ทั้งหมดของโปรเจกต์จริง ๆ** (Django, django-stubs, DRF, ฯลฯ) เพื่อ
resolve type ให้ถูกต้อง การใช้ mirror repo ของ pre-commit ต้อง maintain รายชื่อ
`additional_dependencies` แยกให้ตรงกับ `requirements/dev.txt` ทุกครั้งซึ่งเสี่ยงหลุด
sync ได้ง่าย การใช้ `language: system` (เรียก `mypy` ที่ติดตั้งอยู่ใน venv ปัจจุบัน
ของคุณโดยตรง) จึงปลอดภัยและตรงกับความจริงมากกว่าในโปรเจกต์ Django

---

## ขั้นตอนที่ 644: Code Formatting อัตโนมัติด้วย `ruff format`

### 644.1 Formatter คืออะไร ต่างจาก Linter อย่างไร

**Linter** (ขั้นตอนที่ 642) บอกว่าโค้ด "มีปัญหาเชิงตรรกะหรือสไตล์" ตรงไหน ส่วน
**Formatter** จัดการเรื่อง**หน้าตา**ของโค้ดล้วน ๆ (การเว้นวรรค, ความยาวบรรทัด,
เครื่องหมายคำพูดเดี่ยว/คู่, การจัดวง เล็บ) โดยไม่แตะตรรกะเลย เป้าหมายคือทำให้โค้ด
ทั้งทีม**หน้าตาเหมือนกันทุกไฟล์** โดยไม่ต้องเถียงกันเรื่องสไตล์ส่วนตัวอีกต่อไป
(ปรากฏการณ์ที่เรียกว่า "bikeshedding")

Python มี formatter ยอดนิยมสองตัว: **[Black](https://black.readthedocs.io/)**
(มาตรฐานดั้งเดิมตั้งแต่ปี 2018 มีปรัชญา "opinionated, ไม่มี config ให้เถียง") และ
**`ruff format`** ที่มาทีหลังแต่ให้ผลลัพธ์**เข้ากันได้กับ Black ประมาณ 99.9%**
เพราะ Ruff ตั้งใจออกแบบให้ migrate จาก Black มาได้แทบไม่ต้องเปลี่ยนอะไร:

| คุณสมบัติ | Black | `ruff format` |
|---|---|---|
| ภาษาที่เขียน | Python | Rust |
| ความเร็ว | มาตรฐาน | เร็วกว่า 20-30 เท่า |
| ความเข้ากันได้ของผลลัพธ์ | มาตรฐานอ้างอิง | เข้ากันได้ ~99.9% กับ Black |
| ต้องติดตั้งเพิ่มจาก Ruff หรือไม่ | ต้องติดตั้งแยก (`pip install black`) | ไม่ต้อง — มากับ `ruff` ตัวเดียวกับ linter |
| เหมาะกับหลักสูตรนี้ | ทางเลือกได้ | ✅ แนะนำ เพราะใช้ไบนารีเดียวกับ linter ที่ติดตั้งไปแล้ว |

เนื่องจากคุณติดตั้ง Ruff ไปแล้วในขั้นตอนที่ 642 (สำหรับ lint) หลักสูตรนี้จึงแนะนำให้ใช้
`ruff format` แทน Black เพื่อลดจำนวนเครื่องมือที่ต้องดูแล — **แต่ถ้าทีมของคุณใช้ Black
มาก่อนแล้ว ก็สามารถสลับไปใช้ Black แทนได้ทันทีโดยหลักการเดียวกันทุกประการ**

### 644.2 ตั้งค่า `ruff format` (ใช้ `[tool.ruff]` เดียวกับขั้นตอนที่ 642)

```toml
# pyproject.toml — เพิ่มส่วนนี้ต่อจาก [tool.ruff.lint] ที่ตั้งไว้แล้ว
[tool.ruff.format]
quote-style = "double"          # ใช้เครื่องหมายคำพูดคู่เสมอ (เหมือน Black)
indent-style = "space"          # เว้นวรรคด้วย space ไม่ใช่ tab
skip-magic-trailing-comma = false
line-ending = "lf"              # บังคับ Unix line ending เสมอ ป้องกันปัญหาข้าม OS
```

### 644.3 คำสั่งใช้งาน

```bash
# ฟอร์แมตทั้งโปรเจกต์ (แก้ไฟล์จริงทันที)
ruff format .

# ฟอร์แมตเฉพาะโฟลเดอร์เดียว
ruff format blog/

# ตรวจสอบว่าโค้ด "ฟอร์แมตถูกต้องแล้วหรือยัง" โดยไม่แก้ไฟล์ (ใช้ใน CI)
ruff format --check .

# แสดง diff ว่าถ้าฟอร์แมตจะเปลี่ยนอะไรบ้าง โดยยังไม่แก้จริง
ruff format --diff blog/models.py
```

ตัวอย่างผลลัพธ์ `ruff format --check .` เมื่อมีไฟล์ที่ยังไม่ผ่านฟอร์แมต (ใช้ใน CI
เพื่อ**ปฏิเสธ** pull request ที่ไม่ได้รัน format มาก่อน):

```
Would reformat: blog/views.py
Would reformat: blog/forms.py
2 files would be reformatted, 14 files already formatted
```

### 644.4 Workflow ที่แนะนำ: format ก่อน lint เสมอ

ลำดับที่ถูกต้องของ `.pre-commit-config.yaml` ในขั้นตอนที่ 641.3 คือให้ `ruff` (lint
พร้อม `--fix`) รันก่อน `ruff-format` เสมอ เพราะการแก้ lint บางอย่าง (เช่น เรียง
import ใหม่) อาจทำให้การเว้นบรรทัดเปลี่ยนไป การรัน format ทีหลังจึงรับประกันว่าผลลัพธ์
สุดท้ายสะอาดทั้งสองด้าน:

```
1. ruff check --fix   → แก้ปัญหาเชิงตรรกะ/import ก่อน
2. ruff format        → จัดหน้าตาโค้ดให้เรียบร้อยสุดท้าย
```

### 644.5 ตั้งค่า VS Code ให้ format on save (ต่อยอด Part 001)

การตั้งค่า `.vscode/settings.json` ที่ทำไว้ตั้งแต่ Part 001 ขั้นตอนที่ 5.3
(`"editor.formatOnSave": true` และ `"editor.defaultFormatter": "charliermarsh.ruff"`)
ครอบคลุมทั้ง lint auto-fix และ format อยู่แล้ว ทำให้ในทางปฏิบัติคุณแทบไม่ต้องพิมพ์
`ruff format .` ด้วยมือเลยตลอดการพัฒนาประจำวัน — คำสั่งนี้จะมีประโยชน์จริง ๆ ตอนรัน
ใน CI/CD และตอนต้องฟอร์แมตทั้งโปรเจกต์ครั้งแรกหลัง clone หรือหลัง merge จากหลายคน

---

## ขั้นตอนที่ 645: ทบทวน django-debug-toolbar เป็นเครื่องมือ Quality ระหว่างพัฒนา

### 645.1 เชื่อมโยงกับสิ่งที่ติดตั้งไปแล้วใน Part 005

ใน **Part 005 (Django Apps และการจัดระเบียบโค้ด)** คุณติดตั้ง
**`django-debug-toolbar`** ไปแล้วเพื่อดู SQL query, template context, และ request
timing ระหว่างพัฒนา ในขั้นตอนนี้เราจะ**ทบทวนและยกระดับ**การใช้เครื่องมือนี้ให้เป็นส่วน
หนึ่งของ "quality workflow" อย่างเป็นระบบ ไม่ใช่แค่เปิดดูเล่นเฉย ๆ เพราะมันคือเครื่องมือ
quality ประเภทที่**ต่างจากทุกตัวที่เรียนมาใน Part นี้**: ตัวอื่นทั้งหมด (Ruff, mypy,
Bandit, pytest) เป็น **automated check** ที่รันแบบ headless ไม่ต้องมีคนดู ในขณะที่
Debug Toolbar เป็นเครื่องมือ **manual, interactive inspection** ที่ต้องอาศัยสายตาคน
มองระหว่างพัฒนาจริง — ทั้งสองแนวทางเสริมกัน ไม่ได้แทนที่กัน

### 645.2 ทบทวนการตั้งค่าให้ครบถ้วน

```python
# config/settings/dev.py
from .base import *  # noqa: F401,F403

DEBUG = True

INSTALLED_APPS += ["debug_toolbar"]

MIDDLEWARE = [
    "debug_toolbar.middleware.DebugToolbarMiddleware",
    *MIDDLEWARE,
]

INTERNAL_IPS = ["127.0.0.1"]

DEBUG_TOOLBAR_CONFIG = {
    "SHOW_TOOLBAR_CALLBACK": lambda request: DEBUG,
    "SHOW_COLLAPSED": True,  # ย่อ panel ไว้ก่อน ไม่บังหน้าจอตอนเริ่มพัฒนา
}
```

```python
# config/urls.py
from django.conf import settings

urlpatterns = [
    # ... urlpatterns เดิม ...
]

if settings.DEBUG:
    import debug_toolbar

    urlpatterns += [
        path("__debug__/", include(debug_toolbar.urls)),
    ]
```

**กฎเหล็กด้านความปลอดภัย**: `debug_toolbar` ต้องถูกเพิ่มเข้า `INSTALLED_APPS` และ
`urlpatterns` เฉพาะเมื่อ `settings.DEBUG is True` เท่านั้น (ตามโค้ดด้านบน) — การลืม
guard นี้แล้วปล่อยให้ Debug Toolbar โผล่บน production คือช่องโหว่ความปลอดภัยจริงที่
เปิดเผยข้อมูล SQL query, environment variable, และ stack trace ให้ผู้ไม่หวังดีเห็นได้
โดยตรง (Bandit ในขั้นตอนที่ 646 จะช่วยเตือนกรณีคล้ายกันนี้ด้วย)

### 645.3 Panel ที่มีประโยชน์ที่สุดสำหรับงาน Quality

| Panel | ใช้ตรวจอะไร | ประโยชน์ด้าน Quality |
|---|---|---|
| **SQL** | รายการ query ทั้งหมดที่เกิดขึ้นในหน้านั้น พร้อมเวลาที่ใช้ | ตรวจจับปัญหา **N+1 query** ก่อนที่ Part 067 (Query Optimization) จะเจาะลึกวิธีแก้ |
| **Templates** | template ที่ถูก render และ context variable ทั้งหมด | ตรวจสอบว่า view ส่ง context เกินความจำเป็นหรือไม่ |
| **Cache** | การเรียก cache framework (hit/miss) | เตรียมความพร้อมก่อนเรียน Caching ใน Part 068-069 |
| **Signals** | signal ทั้งหมดที่ยิงระหว่าง request | ช่วย debug พฤติกรรมที่ signal ทำให้ query เพิ่มขึ้นโดยไม่รู้ตัว |
| **Profiling** | เวลาที่แต่ละฟังก์ชันใช้ (ต้องเปิดเพิ่มใน config) | หาจุดคอขวดเบื้องต้นก่อนใช้เครื่องมือ profiling เต็มรูปแบบใน Part 066 |

### 645.4 ตัวอย่างการใช้ SQL Panel จับปัญหา N+1 Query จริง

```python
# blog/views.py — ❌ มีปัญหา N+1 query
def post_list(request: HttpRequest) -> HttpResponse:
    posts = Post.objects.filter(is_published=True)
    return render(request, "blog/post_list.html", {"posts": posts})
```

```html
<!-- blog/templates/blog/post_list.html -->
{% for post in posts %}
    <h2>{{ post.title }}</h2>
    <p>โดย {{ post.author.get_full_name }}</p>  {# เข้าถึง post.author ทุกรอบ #}
{% endfor %}
```

เปิด Debug Toolbar แล้วดู **SQL panel** จะเห็นทันทีว่าถ้ามี 20 บทความ จะเกิด query
ทั้งหมด **21 ครั้ง** (1 ครั้งสำหรับดึงบทความทั้งหมด + 20 ครั้งสำหรับดึง `author` ของ
แต่ละบทความแยกกัน) — นี่คือสัญญาณคลาสสิกของ N+1 query ที่ Debug Toolbar ช่วยให้
"เห็นด้วยตา" ได้ทันทีโดยไม่ต้องเดา วิธีแก้ (ใช้ `select_related`) จะเจาะลึกเต็มรูปแบบ
ใน Part 067 แต่หลักการเบื้องต้นคือ:

```python
# ✅ แก้ไขด้วย select_related — ลด query เหลือ 1 ครั้งเดียว
def post_list(request: HttpRequest) -> HttpResponse:
    posts = Post.objects.filter(is_published=True).select_related("author")
    return render(request, "blog/post_list.html", {"posts": posts})
```

### 645.5 Debug Toolbar vs Automated Test: ใช้เมื่อไหร่

| สถานการณ์ | เครื่องมือที่เหมาะสม |
|---|---|
| ต้องการรู้ทันทีว่าหน้านี้มี query กี่ครั้งระหว่างเขียนโค้ด | Debug Toolbar (มองด้วยตา) |
| ต้องการป้องกันไม่ให้ N+1 query กลับมาอีกในอนาคตหลังแก้แล้ว | Automated test ที่ assert จำนวน query (`assertNumQueries`, เรียนไปแล้วใน Part 060) |
| ต้องการตรวจสอบ query ของทุกหน้าใน CI แบบอัตโนมัติ | Test + coverage (Part 062) ไม่ใช่ Debug Toolbar ซึ่งเป็นเครื่องมือ manual เท่านั้น |

นี่คือบทเรียนสำคัญ: **Debug Toolbar ช่วยให้คุณ "ค้นพบ" ปัญหาได้เร็ว แต่ automated
test ต่างหากที่ "ป้องกัน" ไม่ให้ปัญหากลับมาซ้ำ** ทั้งสองเครื่องมือจึงควรอยู่คู่กันเสมอ
ไม่ใช่เลือกใช้อย่างใดอย่างหนึ่ง

---

## ขั้นตอนที่ 646: Static Analysis ด้านความปลอดภัยด้วย `bandit`

### 646.1 Bandit คืออะไร และต่างจาก Ruff/mypy อย่างไร

**[Bandit](https://bandit.readthedocs.io/)** คือเครื่องมือ static analysis ที่พัฒนา
โดยทีม OpenStack Security แล้วส่งต่อให้ **PyCQA** ดูแล ออกแบบมาเพื่อสแกนหา
**รูปแบบโค้ดที่เสี่ยงต่อช่องโหว่ความปลอดภัย** โดยเฉพาะ (ต่างจาก Ruff ที่เน้นสไตล์/
ตรรกะทั่วไป และ mypy ที่เน้น type) เช่น การใช้ `eval()`, hardcoded password,
SQL query ที่สร้างด้วย string concatenation (เสี่ยง SQL Injection), การใช้
`pickle` กับข้อมูลที่ไม่น่าเชื่อถือ

Bandit ทำงานโดยแปลงโค้ด Python เป็น **AST (Abstract Syntax Tree)** แล้วจับคู่กับ
รูปแบบที่รู้จักว่าเสี่ยง คล้ายกับที่ Ruff ทำ แต่เจาะจงเฉพาะมุมมองความปลอดภัย

### 646.2 ติดตั้ง Bandit

```bash
pip install "bandit[toml]"
```

`[toml]` extra จำเป็นเพื่อให้ Bandit อ่าน config จาก `pyproject.toml` ได้ (ไม่งั้น
ต้องใช้ไฟล์ `.bandit` แยกแบบเก่า)

### 646.3 ตั้งค่าใน `pyproject.toml`

```toml
# pyproject.toml
[tool.bandit]
exclude_dirs = ["migrations", "tests", "venv", ".venv"]
skips = [
    "B101",  # assert used — ยอมรับได้เพราะ Django test/assert ใช้เป็นปกติในไฟล์ที่ไม่ได้ exclude
]

[tool.bandit.assert_used]
skips = ["*/tests/*.py", "*/test_*.py"]
```

### 646.4 รันตรวจสอบและตัวอย่างช่องโหว่จริงที่ Bandit จับได้

```bash
# สแกนทั้งโปรเจกต์ (แสดงผลระดับความรุนแรงสูง-กลาง-ต่ำ)
bandit -r blog/ config/ -x blog/migrations,blog/tests

# แสดงเฉพาะปัญหาระดับ Medium ขึ้นไป
bandit -r blog/ -ll

# สร้างรายงานแบบ JSON (ใช้เชื่อมกับ CI dashboard)
bandit -r blog/ -f json -o bandit-report.json
```

ตัวอย่างที่ 1 — **Hardcoded Secret** (ตรวจพบบ่อยที่สุดในโปรเจกต์จริง):

```python
# ❌ ก่อนแก้ — Bandit เตือนทันที: B105 hardcoded_password_string
def send_admin_notification(message: str) -> None:
    api_key = "sk_live_51H8xJ2KZv..."   # อันตรายมาก! เผลอ commit ขึ้น Git
    requests.post("https://api.example.com/notify", json={"key": api_key, "message": message})
```

```python
# ✅ หลังแก้ — อ่านจาก environment variable แทน (ตามหลักการ Part 010)
import environ

env = environ.Env()


def send_admin_notification(message: str) -> None:
    api_key = env("NOTIFICATION_API_KEY")
    requests.post("https://api.example.com/notify", json={"key": api_key, "message": message})
```

ตัวอย่างที่ 2 — **SQL Injection ผ่าน raw query**:

```python
# ❌ ก่อนแก้ — Bandit เตือน B608 hardcoded_sql_expressions
def search_posts_by_title(title: str) -> list[Post]:
    query = f"SELECT * FROM blog_post WHERE title LIKE '%{title}%'"
    return list(Post.objects.raw(query))
    # ถ้า title = "'; DROP TABLE blog_post; --" จะเกิด SQL Injection ทันที
```

```python
# ✅ หลังแก้ — ใช้ Django ORM parameterized query ที่ป้องกัน SQL Injection โดยอัตโนมัติ
def search_posts_by_title(title: str) -> list[Post]:
    return list(Post.objects.filter(title__icontains=title))
```

ตัวอย่างที่ 3 — **การใช้ `eval()`**:

```python
# ❌ Bandit เตือน B307 eval — อันตรายมากถ้ารับ input จากผู้ใช้
def calculate_discount(formula: str, price: float) -> float:
    return eval(formula.replace("price", str(price)))
```

```python
# ✅ ใช้ตรรกะที่ควบคุมได้แทนการรันโค้ดแบบไดนามิก
DISCOUNT_RULES = {
    "student": 0.15,
    "member": 0.10,
}


def calculate_discount(rule_name: str, price: float) -> float:
    discount_rate = DISCOUNT_RULES.get(rule_name, 0.0)
    return price * (1 - discount_rate)
```

### 646.5 กฎ Bandit ที่พบบ่อยที่สุดในโปรเจกต์ Django

| ID | ชื่อ | คำอธิบายสั้น |
|---|---|---|
| `B105`/`B106`/`B107` | Hardcoded password/secret | พบ string ที่ดูเหมือน password/API key ฝังตรงในโค้ด |
| `B201` | `flask_debug_true` (เทียบเท่า Django คือ `DEBUG=True`) | ตรวจสอบเพิ่มเติมด้วย `manage.py check --deploy` จาก Part 010 |
| `B307` | `eval` | ใช้ `eval()` กับข้อมูลที่อาจมาจากผู้ใช้ |
| `B301` | `pickle` | `pickle.loads()` กับข้อมูลที่ไม่น่าเชื่อถือ (deserialize ได้ arbitrary code) |
| `B608` | `hardcoded_sql_expressions` | สร้าง SQL query ด้วย string formatting/concatenation |
| `B324` | `hashlib` insecure hash | ใช้ MD5/SHA1 สำหรับงานด้าน security (ควรใช้ SHA-256 ขึ้นไป หรือ `django.contrib.auth.hashers`) |
| `B113` | `request_without_timeout` | เรียก `requests.get()`/`requests.post()` โดยไม่ตั้ง `timeout` เสี่ยง hang ค้าง |

### 646.6 Bandit vs Ruff's `S` rules: เลือกใช้ตัวไหน

ที่จริง Ruff เองก็มีชุดกฎ `S` (flake8-bandit) ที่ port มาจาก Bandit บางส่วน คำถาม
คือทำไม Part นี้ยังแนะนำให้ติดตั้ง Bandit แยกต่างหาก:

| ประเด็น | Ruff `S` rules | Bandit เต็มรูปแบบ |
|---|---|---|
| ความครอบคลุมของกฎ | ครอบคลุมบางส่วน (พอร์ตมาไม่ครบ 100%) | ครอบคลุมที่สุด เป็นต้นฉบับ |
| ความเร็ว | เร็วมาก (รวมกับ lint อื่นในตัวเดียว) | ช้ากว่า Ruff แต่ยังเร็วพอสำหรับ pre-commit |
| รายงานแยกสำหรับทีม Security | ปนกับผล lint ทั่วไป | มีรายงานเฉพาะทาง (JSON/HTML) ส่งให้ทีม security ตรวจสอบแยกได้ |
| การยอมรับในวงการ (compliance/audit) | ยังใหม่ | เป็นมาตรฐานที่ auditor คุ้นเคยมานาน |

หลักสูตรนี้แนะนำให้ **ใช้ทั้งคู่**: เปิด Ruff `S` (ถ้าต้องการ) เพื่อ feedback เร็วระหว่าง
เขียนโค้ด และยังคงรัน Bandit แยกใน pre-commit + CI เพื่อรายงานที่ละเอียดและเป็น
มาตรฐานที่ทีม security ยอมรับสำหรับการ audit ก่อน deploy จริง

---

## ขั้นตอนที่ 647: Test-Driven Development (TDD) — Red-Green-Refactor Cycle

### 647.1 TDD คืออะไร

**Test-Driven Development (TDD)** คือแนวทางการเขียนโค้ดที่กลับลำดับปกติ: แทนที่จะ
เขียนโค้ดฟีเจอร์ก่อนแล้วค่อยเขียน test ตามทีหลัง (ที่คุณทำมาตลอด Part 059-064) TDD
ให้คุณ**เขียน test ก่อนที่โค้ดฟีเจอร์นั้นจะมีอยู่จริงด้วยซ้ำ** แนวคิดนี้เกิดจาก
Kent Beck ผู้บุกเบิก Extreme Programming (XP) และสรุปเป็นวงจร 3 ขั้นตอนที่เรียกว่า
**Red-Green-Refactor**:

```
        ┌─────────────────────────────────────────────────┐
        │                                                   │
        ▼                                                   │
   ┌─────────┐        ┌──────────┐        ┌──────────────┐  │
   │   RED   │ ─────> │  GREEN   │ ─────> │   REFACTOR    │──┘
   │ เขียน test │      │ เขียนโค้ด  │       │ ปรับโค้ดให้ดีขึ้น │
   │ ที่ล้มเหลว │      │ ให้ผ่านแบบ │       │ โดย test ยัง    │
   │ (ยังไม่มี  │      │ ง่ายที่สุด │       │ ต้องผ่านเหมือนเดิม │
   │  โค้ดจริง)  │      │ เท่าที่จะ  │       │              │
   │           │      │ ทำได้     │       │              │
   └─────────┘        └──────────┘        └──────────────┘
```

| ขั้นตอน | สิ่งที่ทำ | เป้าหมาย |
|---|---|---|
| **🔴 Red** | เขียน test สำหรับพฤติกรรมที่ยังไม่มีโค้ดรองรับ แล้วรันดูว่า**ล้มเหลว** | ยืนยันว่า test นี้ตรวจจับความไม่มีอยู่ของฟีเจอร์ได้จริง (ไม่ใช่ test ที่ผ่านอยู่แล้วโดยไม่ได้ตรวจอะไร) |
| **🟢 Green** | เขียนโค้ด**น้อยที่สุดเท่าที่จำเป็น**เพื่อให้ test ผ่าน ไม่ต้องสวยงามตอนนี้ | ยืนยันว่าโค้ดทำงานถูกต้องตามที่ test คาดหวัง |
| **🔵 Refactor** | ปรับปรุงโครงสร้างโค้ดให้สะอาด อ่านง่าย ไม่ซ้ำซ้อน โดย**รัน test ซ้ำทุกครั้ง**เพื่อยืนยันว่ายังผ่านเหมือนเดิม | ได้โค้ดคุณภาพดีโดยไม่กลัวว่าจะพังพฤติกรรมเดิม เพราะมี test คุ้มกันอยู่ |

### 647.2 ตัวอย่างจริง: พัฒนาฟีเจอร์ `reading_time_minutes` ด้วย TDD

สมมติทีมต้องการฟีเจอร์ใหม่: แสดง **"เวลาที่ใช้อ่านโดยประมาณ"** บนหน้าบทความ
(สมมติฐาน: อ่านได้ประมาณ 200 คำต่อนาที) มาเดินตามวงจร TDD ทีละขั้น:

**🔴 ขั้นที่ 1: เขียน test ก่อน (ยังไม่มี `reading_time_minutes` อยู่เลย)**

```python
# blog/tests/test_models.py
import pytest

from blog.models import Post


@pytest.mark.django_db
class TestPostReadingTime:
    def test_reading_time_for_short_post(self):
        post = Post.objects.create(
            title="บทความสั้น",
            slug="short-post",
            content=" ".join(["คำ"] * 100),  # เนื้อหา 100 คำ
        )
        assert post.reading_time_minutes == 1  # 100 คำ / 200 คำต่อนาที ปัดขึ้นเป็น 1 นาทีขั้นต่ำ

    def test_reading_time_for_long_post(self):
        post = Post.objects.create(
            title="บทความยาว",
            slug="long-post",
            content=" ".join(["คำ"] * 1000),  # เนื้อหา 1000 คำ
        )
        assert post.reading_time_minutes == 5  # 1000 / 200 = 5 นาทีพอดี
```

รันดูก่อน:

```bash
pytest blog/tests/test_models.py::TestPostReadingTime -v
```

```
FAILED blog/tests/test_models.py::TestPostReadingTime::test_reading_time_for_short_post
AttributeError: 'Post' object has no attribute 'reading_time_minutes'
FAILED blog/tests/test_models.py::TestPostReadingTime::test_reading_time_for_long_post
AttributeError: 'Post' object has no attribute 'reading_time_minutes'
```

**นี่คือขั้น Red ที่ถูกต้อง** — test ล้มเหลวด้วยเหตุผลที่คาดหวังพอดี (attribute
ยังไม่มีอยู่จริง) ถ้า test ผ่านตั้งแต่ขั้นนี้ แปลว่า test เขียนผิดหรือไม่ได้ตรวจอะไรเลย

**🟢 ขั้นที่ 2: เขียนโค้ดน้อยที่สุดให้ผ่าน**

```python
# blog/models.py
class Post(models.Model):
    # ... field เดิม ...

    @property
    def reading_time_minutes(self) -> int:
        word_count = len(self.content.split())
        return max(1, round(word_count / 200))
```

รันซ้ำ:

```bash
pytest blog/tests/test_models.py::TestPostReadingTime -v
```

```
PASSED blog/tests/test_models.py::TestPostReadingTime::test_reading_time_for_short_post
PASSED blog/tests/test_models.py::TestPostReadingTime::test_reading_time_for_long_post
```

**🔵 ขั้นที่ 3: Refactor**

ในตัวอย่างนี้โค้ดยังกระชับพออยู่แล้ว แต่สมมติทีมตัดสินใจว่าค่า "200 คำต่อนาที" ควร
ปรับได้จาก settings แทนที่จะ hardcode ไว้ในโมเดล — นี่คือจังหวะทำ refactor:

```python
# config/settings/base.py
BLOG_READING_SPEED_WPM = 200  # คำต่อนาที ใช้คำนวณเวลาอ่านโดยประมาณ
```

```python
# blog/models.py
from django.conf import settings


class Post(models.Model):
    # ... field เดิม ...

    @property
    def reading_time_minutes(self) -> int:
        word_count = len(self.content.split())
        wpm = getattr(settings, "BLOG_READING_SPEED_WPM", 200)
        return max(1, round(word_count / wpm))
```

รัน test ซ้ำอีกครั้งเพื่อยืนยันว่า refactor ไม่ได้ทำให้พฤติกรรมเปลี่ยน:

```bash
pytest blog/tests/test_models.py::TestPostReadingTime -v
# PASSED ทั้งสองเทสต์เหมือนเดิม — refactor ปลอดภัย
```

### 647.3 ข้อดีและข้อจำกัดของ TDD

| ข้อดี | ข้อจำกัด |
|---|---|
| ได้ test coverage สูงโดยธรรมชาติ (ทุกฟีเจอร์มี test มาก่อนเสมอ) | ช้ากว่าตอนเริ่มเขียนฟีเจอร์ใหม่ ๆ ในระยะสั้น |
| บังคับให้คิด "ผลลัพธ์ที่ต้องการ" ก่อนลงมือเขียนโค้ด ลดการออกแบบที่หลงทาง | ไม่เหมาะกับงาน exploratory/prototype ที่ยังไม่รู้ requirement ชัดเจน (เช่น ทดลอง UI) |
| Refactor ได้อย่างมั่นใจเพราะมี test คุ้มกันตลอด | ต้องมีวินัยสูง ทีมที่ไม่ชินอาจเขียน test ผิวเผินแค่ให้ผ่านขั้น Red-Green โดยไม่ได้คิดเคส edge case จริง |
| Design ที่ได้มักมีการแยกส่วน (decoupled) ดีกว่า เพราะโค้ดที่ test ยากมักบอกใบ้ว่าออกแบบไม่ดี | Legacy code ที่ไม่มี test มาก่อนเลย ทำ TDD ต่อยอดยากกว่าการเขียนใหม่ตั้งแต่ต้น |

**คำแนะนำเชิงปฏิบัติ**: หลักสูตรนี้ไม่ได้บังคับให้คุณทำ TDD 100% ของเวลาการพัฒนา
แต่แนะนำให้ใช้ TDD กับ**ตรรกะทางธุรกิจที่ซับซ้อนหรือมีความเสี่ยงสูง** (เช่น การคำนวณ
ราคา, การตรวจสอบสิทธิ์, business rule ที่ซับซ้อน) ซึ่งเป็นจุดที่ผลตอบแทนของ TDD
สูงที่สุด ส่วนงาน UI ที่ยังไม่นิ่งหรือ prototype เร็ว ๆ การเขียน test ตามทีหลัง (ที่
เรียกว่า "test-after") ก็ยังใช้ได้ดีเช่นกัน

---

## ขั้นตอนที่ 648: Mutation Testing ด้วย `mutmut`

### 648.1 ปัญหาที่ Coverage สูงแต่ไม่ได้แปลว่า Test ดี

ใน **Part 062 ขั้นตอนที่ 613** คุณเรียนไปแล้วว่า **coverage สูงไม่ได้แปลว่า test
ดี** เพราะ coverage บอกแค่ว่า "บรรทัดนี้ถูกรันหรือไม่" แต่ไม่รู้ว่า test ตรวจสอบผลลัพธ์
ถูกต้องจริงหรือเปล่า ลองดูตัวอย่างสุดโต่งนี้:

```python
# blog/models.py
def calculate_discount_price(price: float, discount_percent: float) -> float:
    return price - (price * discount_percent / 100)
```

```python
# tests/test_models.py — ❌ test ที่ coverage 100% แต่ไม่ตรวจอะไรเลยจริง ๆ
def test_calculate_discount_price():
    result = calculate_discount_price(100, 10)
    assert result is not None   # ผ่านเสมอไม่ว่าฟังก์ชันจะคำนวณถูกหรือผิด!
```

test ข้างต้นทำให้บรรทัด `return price - ...` ถูกรัน (coverage = 100%) แต่ต่อให้
สลับเครื่องหมาย `-` เป็น `+` โดยไม่ตั้งใจ (บั๊กร้ายแรง) test นี้ก็ยัง**ผ่านเหมือนเดิม**
เพราะไม่เคย assert ค่าที่ถูกต้องจริง ๆ เลย — นี่คือช่องว่างที่ coverage มองไม่เห็น
และเป็นเหตุผลที่ต้องมีเครื่องมือชั้นที่ลึกกว่านั้นอีกชั้น

### 648.2 Mutation Testing คืออะไร

**Mutation Testing** แก้ปัญหานี้ด้วยแนวคิดที่ฉลาดมาก: เครื่องมือจะ**จงใจแก้โค้ดของ
คุณให้ผิดเล็กน้อย** (เรียกการแก้นี้ว่า "mutant") เช่น เปลี่ยน `-` เป็น `+`, เปลี่ยน
`>` เป็น `>=`, เปลี่ยน `True` เป็น `False`, ลบเงื่อนไข `if` ทิ้ง แล้วรัน test suite
ทั้งหมดกับโค้ดที่ถูกแก้นั้น

- ถ้า **test suite ล้มเหลว** (จับความผิดปกติได้) → mutant นั้นถูกเรียกว่า **"killed"**
  (test ทำหน้าที่ถูกต้อง)
- ถ้า **test suite ยังผ่านหมด** แม้โค้ดจะถูกแก้ให้ผิดไปแล้ว → mutant นั้นถูกเรียกว่า
  **"survived"** (สัญญาณอันตราย: แปลว่า test ไม่ได้ตรวจสอบพฤติกรรมนี้จริง ๆ)

**Mutation Score** คือเปอร์เซ็นต์ของ mutant ที่ถูก "killed" เทียบกับทั้งหมด ยิ่งสูง
ยิ่งแปลว่า test suite ของคุณตรวจจับบั๊กได้ไวจริง ไม่ใช่แค่ "วิ่งผ่านโค้ด" เฉย ๆ

### 648.3 ติดตั้งและตั้งค่า `mutmut`

```bash
pip install mutmut
```

```toml
# pyproject.toml
[tool.mutmut]
paths_to_mutate = "blog/"
runner = "python -m pytest -x -q"
tests_dir = "blog/tests/"
```

### 648.4 รัน Mutation Testing

```bash
# รัน mutmut (จะสร้าง mutant ทีละตัว แล้วรัน test suite ซ้ำสำหรับแต่ละตัว — ใช้เวลานาน)
mutmut run

# ดูสรุปผลรวม
mutmut results
```

ตัวอย่างผลลัพธ์:

```
- 45 killed mutants 🎉 (66%)
- 18 survived mutants 🙁 (26%)
- 6 skipped mutants (timeout, 8%)

69 total mutants
```

ดู mutant ที่รอดตัวหนึ่งโดยเฉพาะ (เพื่อดูว่า test ควรเพิ่มอะไร):

```bash
mutmut show 12
```

```diff
--- blog/models.py
+++ blog/models.py
@@ -45,7 +45,7 @@
     @property
     def reading_time_minutes(self) -> int:
         word_count = len(self.content.split())
         wpm = getattr(settings, "BLOG_READING_SPEED_WPM", 200)
-        return max(1, round(word_count / wpm))
+        return max(1, round(word_count * wpm))
```

### 648.5 ตัวอย่าง: แก้ Test ให้ "ฆ่า" Mutant ที่รอดตัวนี้

Mutant ด้านบนเปลี่ยน `/` เป็น `*` แล้ว test suite เดิมจากขั้นตอนที่ 647.2 ยังผ่าน
ทั้งหมด — เป็นไปได้อย่างไร? ตรวจสอบดูจะพบว่า test เดิมทดสอบแค่กรณี "จำนวนคำที่พอดี"
เท่านั้น (100 คำ → 1 นาที, 1000 คำ → 5 นาที) แต่บังเอิญว่าเลข 100 และ 200 ให้ผลลัพธ์
ที่ดูสมเหตุสมผลได้ทั้งสองทาง (`max(1, round(100/200))` = 1 และ `max(1, round(100*200))`
ก็ยัง `max(1, ...)` ทำให้ค่าดูไม่ต่างในเทสต์ที่เขียนไว้ตอนแรก เนื่องจาก edge case
ไม่ครอบคลุมพอ) วิธีแก้คือเพิ่ม test ที่แยกแยะสองพฤติกรรมนี้ได้ชัดเจนกว่า:

```python
# blog/tests/test_models.py — เพิ่มเทสต์ใหม่ที่แยกแยะ / กับ * ได้ชัดเจน
def test_reading_time_scales_inversely_with_reading_speed(settings):
    settings.BLOG_READING_SPEED_WPM = 100  # อ่านช้าลง ควรใช้เวลานานขึ้น ไม่ใช่สั้นลง
    post = Post.objects.create(
        title="ทดสอบความเร็วอ่าน",
        slug="reading-speed-test",
        content=" ".join(["คำ"] * 1000),
    )
    assert post.reading_time_minutes == 10  # 1000/100 = 10 (ถ้าใช้ * แทน / จะได้ 100,000 ผิดชัดเจน)
```

รัน `mutmut run` ซ้ำ mutant ตัวนี้จะถูก "killed" ทันที เพราะ test ใหม่จับความแตกต่าง
ระหว่างการหารกับการคูณได้ชัดเจนแล้ว

### 648.6 ข้อควรระวัง: Mutation Testing ไม่ควรรันทุก Commit

| ประเด็น | รายละเอียด |
|---|---|
| ความเร็ว | ต้องรัน test suite ใหม่**ทุก mutant** อาจใช้เวลาหลักนาทีถึงหลักชั่วโมงในโปรเจกต์ใหญ่ |
| ความถี่ที่แนะนำ | รันเป็น **scheduled job รายสัปดาห์** หรือก่อน release ใหญ่ ไม่ใช่ pre-commit hook ทุกครั้ง |
| ขอบเขตที่ควร mutate | จำกัดเฉพาะโมดูลที่มี business logic สำคัญ (`paths_to_mutate`) ไม่ต้อง mutate ทั้งโปรเจกต์ |
| การตีความผล | mutant ที่รอดไม่ได้แปลว่าต้อง "ฆ่า" ให้หมดทุกตัวเสมอไป บาง mutant อาจ equivalent กับโค้ดเดิม (ให้ผลลัพธ์เหมือนกันทุกกรณีจริง) ต้องใช้วิจารณญาณประกอบ |

หลักสูตรนี้แนะนำให้ใช้ mutation testing เป็นเครื่องมือ**ตรวจสุขภาพ test suite เป็น
ระยะ** ไม่ใช่ gate ที่บล็อกทุก commit เหมือน Ruff/mypy/pytest ปกติ (รายละเอียดการตั้ง
schedule ด้วย GitHub Actions cron จะอยู่ใน Part 088)

---

## ขั้นตอนที่ 649: Makefile/tox มาตรฐานสำหรับคำสั่งพัฒนาที่ทีมใช้ร่วมกัน

### 649.1 ปัญหาที่เกิดเมื่อทีมโตขึ้น: "คำสั่งอะไรนะที่ต้องรัน"

ถึงจุดนี้โปรเจกต์ `blog` ของคุณมีคำสั่งที่ต้องจำมากมาย: `ruff check .`, `ruff format .`,
`mypy blog/`, `bandit -r blog/ -x ...`, `pytest --cov=blog --cov-fail-under=80`,
`pre-commit run --all-files` — สมาชิกใหม่ในทีมที่เพิ่ง clone โปรเจกต์จะจำคำสั่งเหล่านี้
ไม่ได้ทั้งหมด และแต่ละคนอาจพิมพ์ flag ผิดเพี้ยนกันไปคนละแบบ **`Makefile`** (มาตรฐาน
Unix ที่มีมาตั้งแต่ยุค 1970) และ **`tox`** (มาตรฐานของวงการ Python สำหรับทดสอบข้าม
หลาย environment) ช่วยแก้ปัญหานี้โดยการ "ห่อ" คำสั่งยาว ๆ ให้เหลือแค่คำสั่งสั้น
เดียวที่ทุกคนพิมพ์เหมือนกันได้เป๊ะ

### 649.2 สร้าง `Makefile` มาตรฐานของโปรเจกต์

```makefile
# Makefile
.PHONY: install migrate run shell test test-cov lint format typecheck security mutation quality clean help

help:  ## แสดงรายการคำสั่งทั้งหมดพร้อมคำอธิบาย
	@grep -E '^[a-zA-Z_-]+:.*?## .*$$' $(MAKEFILE_LIST) | sort | awk 'BEGIN {FS = ":.*?## "}; {printf "\033[36m%-15s\033[0m %s\n", $$1, $$2}'

install:  ## ติดตั้ง dependency ทั้งหมดและเสียบ pre-commit hook
	pip install -r requirements/dev.txt
	pre-commit install

migrate:  ## รัน database migration
	python manage.py migrate

run:  ## รัน development server
	python manage.py runserver

shell:  ## เปิด Django shell (ใช้ shell_plus จาก django-extensions ถ้ามี)
	python manage.py shell_plus

test:  ## รัน test suite ทั้งหมดด้วย pytest
	pytest

test-cov:  ## รัน test พร้อมวัด coverage และบังคับขั้นต่ำ 80%
	pytest --cov=blog --cov-report=term-missing --cov-report=html --cov-fail-under=80

lint:  ## ตรวจสอบโค้ดด้วย Ruff (ไม่แก้ไฟล์)
	ruff check .

lint-fix:  ## ตรวจสอบและแก้ไขปัญหาที่แก้ได้อัตโนมัติ
	ruff check . --fix

format:  ## จัดฟอร์แมตโค้ดทั้งโปรเจกต์
	ruff format .

format-check:  ## ตรวจสอบว่าโค้ดฟอร์แมตถูกต้องแล้ว (ใช้ใน CI ไม่แก้ไฟล์)
	ruff format --check .

typecheck:  ## ตรวจสอบ type ด้วย mypy + django-stubs
	mypy blog/ config/

security:  ## สแกนช่องโหว่ความปลอดภัยด้วย Bandit
	bandit -r blog/ config/ -x blog/migrations,blog/tests

mutation:  ## รัน mutation testing (ใช้เวลานาน แนะนำรันเฉพาะก่อน release)
	mutmut run
	mutmut results

quality: lint format-check typecheck security test-cov  ## รัน quality gate ทั้งหมด (ใช้ก่อน push/ใน CI)
	@echo "✅ ผ่านทุก quality check แล้ว พร้อม push/merge"

clean:  ## ลบไฟล์ cache/ชั่วคราวทั้งหมด
	find . -type d -name "__pycache__" -exec rm -rf {} + 2>/dev/null || true
	rm -rf .pytest_cache .mypy_cache .ruff_cache .mutmut-cache htmlcov .coverage
```

ทดสอบใช้งาน:

```bash
make help       # ดูรายการคำสั่งทั้งหมดพร้อมคำอธิบาย
make install    # ติดตั้งครั้งแรกหลัง clone โปรเจกต์
make quality    # รัน quality gate ทั้งหมดก่อน push
```

ตัวอย่างผลลัพธ์ `make help`:

```
format         จัดฟอร์แมตโค้ดทั้งโปรเจกต์
format-check   ตรวจสอบว่าโค้ดฟอร์แมตถูกต้องแล้ว (ใช้ใน CI ไม่แก้ไฟล์)
install        ติดตั้ง dependency ทั้งหมดและเสียบ pre-commit hook
lint           ตรวจสอบโค้ดด้วย Ruff (ไม่แก้ไฟล์)
lint-fix       ตรวจสอบและแก้ไขปัญหาที่แก้ได้อัตโนมัติ
migrate        รัน database migration
mutation       รัน mutation testing (ใช้เวลานาน แนะนำรันเฉพาะก่อน release)
quality        รัน quality gate ทั้งหมด (ใช้ก่อน push/ใน CI)
run            รัน development server
security       สแกนช่องโหว่ความปลอดภัยด้วย Bandit
shell          เปิด Django shell (ใช้ shell_plus จาก django-extensions ถ้ามี)
test           รัน test suite ทั้งหมดด้วย pytest
test-cov       รัน test พร้อมวัด coverage และบังคับขั้นต่ำ 80%
typecheck      ตรวจสอบ type ด้วย mypy + django-stubs
```

### 649.3 `tox`: ทดสอบข้ามหลายเวอร์ชันของ Python/Django

ในขณะที่ `Makefile` เหมาะกับ "คำสั่งที่รันบนเครื่องปัจจุบัน" **`tox`** ตอบโจทย์ที่
`Makefile` ทำไม่ได้ดี: การทดสอบโปรเจกต์กับ**หลายเวอร์ชันของ Python และ Django พร้อม
กัน** โดย tox จะสร้าง virtual environment แยกของตัวเองสำหรับแต่ละ combination
อัตโนมัติ ซึ่งสำคัญมากสำหรับโปรเจกต์ที่เป็น library หรือทีมที่ต้อง support LTS
หลายเวอร์ชัน (ตามที่เรียนเรื่อง Django LTS ใน Part 001 ขั้นตอนที่ 9.4):

```bash
pip install tox
```

```ini
; tox.ini
[tox]
envlist = py{310,311,312}-django{42,51}
isolated_build = True

[testenv]
deps =
    -r requirements/test.txt
    django42: django>=4.2,<4.3
    django51: django>=5.1,<5.2
setenv =
    DJANGO_SETTINGS_MODULE = config.settings
commands =
    ruff check .
    ruff format --check .
    mypy blog/ config/
    bandit -r blog/ config/ -x blog/migrations,blog/tests
    pytest --cov=blog --cov-fail-under=80
```

รันทดสอบทุก combination:

```bash
# รันทุก environment ที่ประกาศไว้ (py310-django42, py310-django51, py311-django42, ...)
tox

# รันเฉพาะ environment เดียว
tox -e py312-django51

# รันแบบ parallel เพื่อความเร็ว (tox 4+)
tox run-parallel
```

ตัวอย่างผลลัพธ์:

```
py310-django42: OK (12.4 seconds)
py310-django51: OK (13.1 seconds)
py311-django42: OK (11.8 seconds)
py311-django51: OK (12.0 seconds)
py312-django42: OK (10.9 seconds)
py312-django51: OK (11.2 seconds)

  py310-django42: commands succeeded
  py310-django51: commands succeeded
  py311-django42: commands succeeded
  py311-django51: commands succeeded
  py312-django42: commands succeeded
  py312-django51: commands succeeded
  congratulations :)
```

### 649.4 ตารางเปรียบเทียบ Makefile vs tox: ใช้เมื่อไหร่

| คุณสมบัติ | `Makefile` | `tox` |
|---|---|---|
| จุดประสงค์หลัก | ย่อคำสั่งยาวให้จำง่าย รันบน environment ปัจจุบัน | ทดสอบข้ามหลายเวอร์ชัน Python/Django ใน environment แยกกัน |
| สร้าง venv ให้อัตโนมัติหรือไม่ | ❌ (ใช้ venv ที่ activate อยู่) | ✅ (สร้างแยกต่อ combination) |
| ความเร็วในการรันประจำวัน | เร็ว (ไม่มี overhead) | ช้ากว่า (ต้องสร้าง/sync environment ก่อน) |
| เหมาะกับ | คำสั่งที่ใช้บ่อยระหว่างพัฒนา (`make test`, `make lint`) | CI matrix testing, library ที่ต้อง support หลายเวอร์ชัน |
| ใช้ร่วมกันได้หรือไม่ | ✅ ใช้คู่กันได้ปกติ — Makefile เรียก `tox` ก็ได้ (`make ci: tox`) |

### 649.5 เชื่อมเข้ากับ CI/CD (ตัวอย่างเบื้องต้น ก่อนเจาะลึกใน Part 088)

```yaml
# .github/workflows/quality.yml (ตัวอย่างเบื้องต้น — เจาะลึกเต็มรูปแบบใน Part 088)
name: Quality Gate

on: [push, pull_request]

jobs:
  quality:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: "3.12"
      - run: pip install -r requirements/dev.txt
      - run: make quality
```

ด้วย `Makefile` ที่สร้างไว้ ไฟล์ CI config จึงสั้นและอ่านง่ายมาก เพราะไม่ต้องเขียนทุก
คำสั่งซ้ำในทั้งเครื่อง dev และ CI server — แก้ที่ `Makefile` ที่เดียว ทั้งสองฝั่งก็
sync กันเสมอโดยอัตโนมัติ นี่คือหลักการ **"single source of truth"** ที่ทีมมืออาชีพ
ยึดถือ

---

## ขั้นตอนที่ 650: สรุป Phase 7 ทั้งหมด (Part 059-065)

ยินดีด้วย! คุณเดินทางมาถึงขั้นตอนที่ **650 จาก 1000 ขั้นตอน** ของหลักสูตรทั้งหมด และ
ปิดฉาก **Phase 7: Testing & Quality Assurance** อย่างสมบูรณ์แล้ว ก่อนจะก้าวเข้าสู่
Phase 8 มาทบทวนภาพรวมทุกอย่างที่ผ่านมา ทดสอบความเข้าใจด้วย quiz และปิดท้ายด้วย
แบบฝึกหัดใหญ่ที่รวบยอดทุกทักษะทั้ง 7 Part

### 650.1 ทบทวนภาพรวม Part 059-065

| Part | หัวข้อหลัก | สิ่งที่ได้เรียนรู้สำคัญที่สุด |
|---|---|---|
| **059** | Django Testing เบื้องต้น: unittest | `TestCase`, `setUp()`/`tearDown()`, assert methods, Test Database ที่แยกจากจริงเสมอ, Test Isolation ด้วย transaction rollback |
| **060** | Testing Views, Models และ Forms | ทดสอบ view ด้วย Django test `Client`, ทดสอบ model method/validation, ทดสอบ form ทั้งกรณี valid/invalid |
| **061** | pytest-django และ Fixtures | ติดตั้ง `pytest-django`, `pytest.ini`/`pyproject.toml`, `conftest.py`, fixture แบบ dependency injection ที่กระชับกว่า `setUp()` |
| **062** | Test Coverage และ Mocking | `coverage.py`, `.coveragerc`, ข้อควรระวัง "coverage สูงไม่ได้แปลว่า test ดี", `unittest.mock` (`Mock`, `MagicMock`, `patch`), `freezegun` |
| **063** | Factory Boy และ Test Data Generation | สร้างข้อมูลทดสอบสมจริงด้วย `Factory`, `Faker`, `SubFactory`, `Sequence` แทนการเขียน `Post.objects.create(...)` ซ้ำ ๆ ด้วยมือ |
| **064** | Integration Testing และ Selenium | ทดสอบผ่านเบราว์เซอร์จริงด้วย `LiveServerTestCase` + Selenium WebDriver, Page Object Pattern, ทดสอบ JavaScript/HTMX ที่ unit test แตะไม่ถึง |
| **065** | Continuous Testing และ Code Quality Tools | `pre-commit`, Ruff (lint + format), `mypy`+`django-stubs`, Bandit, TDD, Mutation Testing, Makefile/tox — ประกอบทุกอย่างเป็น Quality Gate เดียว |

### 650.2 แผนภาพ Quality Gate แบบสมบูรณ์ที่คุณสร้างได้แล้วตอนนี้

```
        นักพัฒนาแก้โค้ด
              │
              ▼
   ┌─────────────────────┐
   │  git commit          │
   └──────────┬───────────┘
              │  pre-commit hook ทำงานอัตโนมัติ (ขั้นตอนที่ 641)
              ▼
   ┌─────────────────────────────────────────────────┐
   │ 1. trailing-whitespace, check-yaml, ฯลฯ           │
   │ 2. ruff check --fix      (lint, ขั้นตอนที่ 642)     │
   │ 3. ruff format           (format, ขั้นตอนที่ 644)   │
   │ 4. bandit                (security, ขั้นตอนที่ 646) │
   │ 5. mypy                  (type check, ขั้นตอนที่ 643)│
   └──────────┬────────────────────────────────────────┘
              │ ผ่านทั้งหมด → commit สำเร็จ
              ▼
   ┌─────────────────────┐
   │  git push             │
   └──────────┬───────────┘
              │
              ▼
   ┌─────────────────────────────────────────────────┐
   │ CI/CD (GitHub Actions — เจาะลึกใน Part 088)        │
   │ make quality:                                     │
   │   lint → format-check → typecheck → security      │
   │   → test-cov (unittest/pytest จาก Part 059-063,   │
   │      Selenium จาก Part 064, coverage ≥ 80%)        │
   │ (mutation testing รันแยกเป็น schedule รายสัปดาห์)   │
   └──────────┬────────────────────────────────────────┘
              │ ผ่านทั้งหมด
              ▼
        Merge เข้า main / Deploy
```

แผนภาพนี้คือสิ่งที่แยก**ทีมมือสมัครเล่น**ออกจาก**ทีมระดับมืออาชีพ**อย่างชัดเจน:
ทีมมือสมัครเล่นพึ่งพา "ความจำและวินัยส่วนตัว" ของแต่ละคน ในขณะที่ทีมมืออาชีพสร้าง
**ระบบอัตโนมัติที่บังคับคุณภาพโดยไม่ต้องพึ่งความจำใคร** — บั๊กและช่องโหว่ที่จะหลุด
ไปถึง production ต้องผ่านด่านทั้งหมดนี้ก่อนเสมอ

### 650.3 แบบทดสอบความเข้าใจรวม (Quiz)

ลองตอบคำถามต่อไปนี้ด้วยตัวเองก่อนเปิดดูเฉลย เพื่อประเมินว่าคุณพร้อมสำหรับ Phase 8
หรือยัง:

**1.** อธิบายความแตกต่างระหว่าง `TestCase` ของ Django (unittest-based จาก Part 059)
กับการใช้ fixture ของ `pytest-django` (Part 061) — แบบไหนเหมาะกับสถานการณ์ไหน

**2.** เพราะเหตุใด "Test Coverage 100%" จึงไม่ได้รับประกันว่าโค้ดไม่มีบั๊ก ยกตัวอย่าง
ประกอบ

**3.** `unittest.mock.patch` ควร patch ที่ไหน ("patch where it's used" จาก Part 062)
อธิบายพร้อมเหตุผล

**4.** Factory Boy (Part 063) ต่างจากการเขียน `Model.objects.create(...)` ตรง ๆ
อย่างไร และช่วยแก้ปัญหาอะไรเมื่อโปรเจกต์มี field บังคับจำนวนมาก

**5.** ทำไม integration test ด้วย Selenium (Part 064) ถึงจำเป็น ทั้งที่มี unit test
และ `pytest-django` ครอบคลุมอยู่แล้ว

**6.** อธิบายว่า `pre-commit` framework แก้ปัญหาอะไร และทำไมการมีไฟล์
`.pre-commit-config.yaml` เพียงอย่างเดียวยังไม่พอ ต้องทำอะไรเพิ่มอีกขั้นหนึ่ง

**7.** ทำไม Ruff ถึงเร็วกว่า Flake8/Pylint มาก และ rule กลุ่ม `DJ` ใน Ruff คืออะไร

**8.** ทำไม mypy ธรรมดา (ไม่มี `django-stubs`) ถึงไม่สามารถตรวจสอบโค้ด Django ORM
ได้อย่างถูกต้อง

**9.** Bandit กับ Ruff's `S` rules ต่างกันอย่างไร และทำไมหลักสูตรนี้แนะนำให้ใช้
ทั้งสองอย่างร่วมกัน

**10.** อธิบายวงจร Red-Green-Refactor ของ TDD และบอกว่าขั้นตอนไหนที่มือใหม่มักข้าม
ไปโดยไม่ตั้งใจ พร้อมผลเสียที่ตามมา

**11. (ขั้นสูง)** Mutation Testing ต่างจาก Coverage อย่างไร และคำว่า "mutant survived"
หมายถึงอะไร เป็นสัญญาณที่ดีหรือไม่ดี

**12. (ขั้นสูง)** อธิบายว่าทำไมหลักสูตรนี้แนะนำให้รัน `mutmut` เป็น scheduled job
รายสัปดาห์ แทนที่จะใส่เป็น pre-commit hook เหมือน Ruff/mypy

**13. (ขั้นสูง)** `Makefile` กับ `tox` มีจุดประสงค์ต่างกันอย่างไร และเพราะเหตุใดโปรเจกต์
มืออาชีพจำนวนมากจึงใช้ทั้งสองเครื่องมือร่วมกันแทนที่จะเลือกอย่างใดอย่างหนึ่ง

<details>
<summary>คลิกเพื่อดูเฉลยแบบย่อ</summary>

1. `TestCase` เหมาะกับทีมที่คุ้นเคยกับ OOP/inheritance และต้องการ setup ที่ใช้ร่วมกัน
   ผ่าน class hierarchy ส่วน pytest fixture เหมาะกับการ compose ข้อมูลทดสอบแบบยืดหยุ่น
   กว่า ลด boilerplate และรองรับ parametrize ได้ดีกว่ามาก ทั้งสองใช้ร่วมกันในโปรเจกต์
   เดียวได้ (pytest รัน `TestCase` เดิมได้ปกติ)
2. เพราะ coverage วัดแค่ "บรรทัดถูกรันหรือไม่" ไม่ได้ตรวจว่า assertion ตรวจสอบผลลัพธ์
   ถูกต้องจริงหรือเปล่า ตัวอย่างเช่น test ที่เขียน `assert result is not None` ผ่านได้
   เสมอไม่ว่าฟังก์ชันจะคำนวณถูกหรือผิด (ดูตัวอย่างเต็มในขั้นตอนที่ 648.1)
3. ต้อง patch ที่ "จุดที่ object ถูกเรียกใช้" ไม่ใช่ "จุดที่ object ถูกนิยาม" เพราะ Python
   import แบบ bind ชื่อเข้า namespace ปลายทาง การ patch ผิด namespace จะไม่มีผลกับ
   โค้ดที่รันจริง
4. Factory Boy สร้างข้อมูลทดสอบแบบมี default ค่าที่สมเหตุสมผลให้อัตโนมัติผ่าน `Faker`
   ทำให้ไม่ต้องระบุทุก field ที่ไม่เกี่ยวกับสิ่งที่กำลังทดสอบ ลด boilerplate มาก
   โดยเฉพาะเมื่อ model มี field บังคับ (`null=False`) จำนวนมาก
5. เพราะ unit test/pytest-django ไม่ได้รัน JavaScript จริงในเบราว์เซอร์ พฤติกรรมที่
   พึ่งพา client-side interaction (เช่น HTMX partial update, Alpine.js, form validation
   ฝั่ง JS) ต้องทดสอบผ่านเบราว์เซอร์จริงเท่านั้นถึงจะมั่นใจได้
6. `pre-commit` แก้ปัญหาการลืมรัน check ก่อน commit ด้วยไฟล์ config ที่แชร์ได้ทั้งทีม
   แต่การมีไฟล์เพียงอย่างเดียวไม่พอ เพราะยังไม่ได้ "เสียบ" เข้ากับ `.git/hooks/`
   ของแต่ละเครื่อง ต้องรัน `pre-commit install` เพิ่มอีกขั้นหนึ่งเสมอหลัง clone
7. Ruff เขียนด้วย Rust (ต่างจาก Flake8/Pylint ที่เขียนด้วย Python) ทำให้เร็วกว่ามาก
   rule กลุ่ม `DJ` มาจาก plugin `flake8-django` ที่ตรวจจับปัญหาเฉพาะของ Django เช่น
   `null=True` บน string field หรือ model ที่ไม่มี `__str__()`
8. เพราะ Django ใช้ metaclass และ dynamic attribute จำนวนมาก (เช่น `Model.objects`)
   ที่ mypy ธรรมดาไม่รู้จักโครงสร้างเหล่านี้ ต้องอาศัย stub file และ plugin ของ
   `django-stubs` เพื่อ "สอน" mypy ให้เข้าใจ Django ORM
9. Ruff's `S` rules พอร์ตกฎของ Bandit มาบางส่วนและรวมอยู่กับผลลัพธ์ lint ทั่วไป
   ในขณะที่ Bandit เต็มรูปแบบครอบคลุมกฎมากกว่าและให้รายงานเฉพาะทางที่ทีม security
   คุ้นเคย หลักสูตรแนะนำใช้ร่วมกันเพื่อความเร็ว (Ruff) และความครอบคลุม (Bandit)
10. Red (เขียน test ที่ล้มเหลวก่อน) → Green (เขียนโค้ดน้อยที่สุดให้ผ่าน) → Refactor
    (ปรับปรุงโค้ดโดย test ยังผ่าน) มือใหม่มักข้ามขั้น Red คือเขียนโค้ดกับ test พร้อมกัน
    หรือเขียนโค้ดก่อนแล้วค่อยเขียน test ตาม ทำให้ไม่มั่นใจว่า test ตรวจจับความผิดพลาด
    ได้จริงหรือแค่ผ่านเฉย ๆ
11. Coverage วัดแค่ว่าโค้ดถูกรันหรือไม่ ส่วน Mutation Testing วัดว่า test "ตรวจจับ
    ความผิดพลาด" ได้จริงหรือไม่ โดยจงใจแก้โค้ดให้ผิดแล้วดูว่า test ล้มเหลวตามหรือไม่
    "mutant survived" หมายถึง test suite ไม่จับความผิดปกตินั้นได้ ซึ่งเป็นสัญญาณไม่ดี
    ที่บอกว่าจุดนั้นของโค้ดยังทดสอบไม่ครอบคลุมพอ
12. เพราะต้องรัน test suite ทั้งหมดซ้ำสำหรับทุก mutant ที่สร้างขึ้น ใช้เวลานานมาก
    (หลักนาทีถึงหลักชั่วโมงในโปรเจกต์ใหญ่) ไม่เหมาะกับ feedback loop เร็ว ๆ แบบ
    pre-commit ที่ต้องการผลลัพธ์ภายในไม่กี่วินาที
13. `Makefile` ย่อคำสั่งยาวให้จำง่ายและรันบน environment ปัจจุบันเครื่องเดียว ส่วน
    `tox` สร้าง environment แยกเพื่อทดสอบข้ามหลายเวอร์ชัน Python/Django พร้อมกัน
    ทีมมืออาชีพมักใช้ `Makefile` เป็นทางลัดประจำวัน และให้ `Makefile` เรียก `tox`
    หรือ CI เรียก `make quality` โดยตรงเพื่อให้คำสั่งทั้งสองที่ sync กันเสมอ

</details>

### 650.4 แบบฝึกหัดใหญ่ปิดท้าย Phase 7: Quality Gate ที่สมบูรณ์สำหรับ Blog Project

นี่คือแบบฝึกหัดที่รวบยอดทุกทักษะจาก Part 059-065 เข้าด้วยกัน เป้าหมายคือทำให้
โปรเจกต์ `blog` ของคุณมี **CI-ready Quality Gate ที่สมบูรณ์** พร้อมใช้งานจริงก่อน
เข้าสู่ Phase 8

**ข้อกำหนดด้าน Pre-commit และ Tooling:**

1. สร้างไฟล์ `.pre-commit-config.yaml` ที่มี hook ครบทั้ง 4 กลุ่มตามขั้นตอนที่ 641.3:
   pre-commit-hooks พื้นฐาน, Ruff (lint + format), Bandit, และ mypy (local hook)
2. รัน `pre-commit install` แล้วยืนยันว่า `git commit` ปกติจะรัน hook อัตโนมัติจริง
   (ลองแก้โค้ดให้มี import ที่ไม่ใช้ แล้ว commit ดู ต้องเห็น Ruff บล็อกและแก้ให้อัตโนมัติ)
3. ตั้งค่า `pyproject.toml` ให้มีครบทุก section: `[tool.ruff]`, `[tool.ruff.lint]`,
   `[tool.ruff.format]`, `[tool.mypy]`, `[tool.django-stubs]`, `[tool.bandit]`

**ข้อกำหนดด้าน Type Safety:**

4. เพิ่ม type hint ให้ครบทุก view function และ model method ใน app `blog`
   (parameter และ return type) แล้วรัน `mypy blog/` จนไม่มี error เหลือ
5. หาก mypy เจอ error ที่แก้ไม่ได้ทันที ให้ใช้ `# type: ignore[error-code]` พร้อม
   comment อธิบายเหตุผลกำกับเสมอ (ห้าม ignore แบบไม่มีเหตุผล)

**ข้อกำหนดด้าน Security:**

6. รัน `bandit -r blog/ config/ -x blog/migrations,blog/tests` แล้วแก้ไขทุกปัญหา
   ระดับ Medium ขึ้นไปให้หมด
7. ตรวจสอบว่าไม่มี hardcoded secret ใด ๆ หลงเหลืออยู่ในโค้ด (ทุกค่าที่ควรเป็นความลับ
   ต้องอ่านจาก environment variable ตามหลักการ Part 010)

**ข้อกำหนดด้าน TDD และ Mutation Testing:**

8. เลือก business logic หนึ่งอย่างในแอป `blog` ที่ยังไม่มี (เช่น
   `calculate_reading_time`, `is_post_editable_by(user)`, หรือ `generate_excerpt`)
   แล้วพัฒนาด้วยวงจร **Red-Green-Refactor** ตามขั้นตอนที่ 647.2 อย่างเคร่งครัด —
   บันทึกผลการรัน test ทั้ง 3 ขั้นตอนไว้เป็นหลักฐาน (screenshot หรือ log)
9. รัน `mutmut run` เฉพาะกับโมดูลที่เพิ่งเขียนใน TDD ข้อ 8 แล้วดู mutation score
   ถ้ามี mutant รอด ให้เพิ่ม test จนกว่า mutation score ของโมดูลนั้นจะถึง 100%

**ข้อกำหนดด้าน Makefile/tox และ CI:**

10. สร้าง `Makefile` ที่มีคำสั่งครบตามขั้นตอนที่ 649.2 อย่างน้อย:
    `install`, `test`, `test-cov`, `lint`, `format`, `typecheck`, `security`, `quality`
11. รัน `make quality` แล้วต้องผ่านทุกขั้นตอนโดยไม่มี error (lint, format-check,
    typecheck, security, test-cov ที่ coverage ไม่ต่ำกว่า 80%)
12. สร้างไฟล์ `.github/workflows/quality.yml` ตามตัวอย่างในขั้นตอนที่ 649.5 ที่เรียก
    `make quality` (แม้ยังไม่ได้เรียนเรื่อง GitHub Actions เต็มรูปแบบจนถึง Part 088
    ก็ให้ลองสร้างไฟล์นี้ไว้ก่อนเป็นการเตรียมตัว)

**เกณฑ์ตรวจสอบความสำเร็จ (Checklist ปิดท้าย Phase 7):**

- [ ] `pre-commit run --all-files` ผ่านทุก hook โดยไม่มี error เหลือ
- [ ] `git commit` ปกติ (ไม่ใช้ `--no-verify`) รัน hook อัตโนมัติและบล็อก commit ที่มี
      ปัญหาได้จริง
- [ ] `mypy blog/ config/` ผ่านโดยไม่มี error (หรือมี `# type: ignore` ที่มีเหตุผล
      กำกับทุกจุด)
- [ ] `bandit -r blog/ config/ -x blog/migrations,blog/tests` ไม่มีปัญหาระดับ Medium
      ขึ้นไปเหลืออยู่
- [ ] มี business logic อย่างน้อย 1 ฟังก์ชัน/method ที่พัฒนาด้วย TDD ครบวงจร
      Red-Green-Refactor และมี mutation score 100% เฉพาะจุดนั้น
- [ ] `make quality` รันผ่านทั้งหมดในคำสั่งเดียว
- [ ] ไฟล์ `.github/workflows/quality.yml` ถูกสร้างและ push ขึ้น GitHub แล้ว
- [ ] Coverage รวมทั้งโปรเจกต์ (`pytest --cov=blog --cov-fail-under=80`) ไม่ต่ำกว่า 80%

**ระดับขั้นสูง (โบนัส)**: ตั้งค่า `tox.ini` ให้ทดสอบโปรเจกต์กับ Python 3.11 และ 3.12
พร้อมกัน (ตามขั้นตอนที่ 649.3) แล้วรัน `tox run-parallel` ยืนยันว่าโปรเจกต์ทำงานได้
ถูกต้องทั้งสองเวอร์ชัน — นี่คือมาตรฐานที่ library หรือโปรเจกต์ open source ระดับโลก
ใช้กันจริงก่อนปล่อยเวอร์ชันใหม่ทุกครั้ง

### 650.5 คำนำสู่ Phase 8: Performance & Caching

ตลอด Phase 7 คุณสร้างเกราะป้องกันคุณภาพโค้ดให้แน่นหนาแล้ว: มี test ครอบคลุม, มี
linter/type checker/security scanner คอยเฝ้าระวัง, และมี Quality Gate อัตโนมัติที่
ไม่พึ่งพาความจำของใครคนใดคนหนึ่ง แต่คำถามถัดไปที่ทีมมืออาชีพต้องตอบคือ: **"โปรเจกต์
ที่ถูกต้องและมีคุณภาพนี้ เร็วพอสำหรับผู้ใช้จริงหรือยัง?"** — โค้ดที่ผ่าน test ทุกตัว
และไม่มีช่องโหว่เลย ก็ยังสามารถ**ช้าจนผู้ใช้เลิกใช้งาน**ได้เช่นกัน

**Phase 8: Performance & Caching (Part 066-072 | ขั้นตอนที่ 651-720)** จะพาคุณ:

- **Part 066**: **Django Performance Profiling** — ใช้เครื่องมือ profiling วัดว่า
  ส่วนไหนของโค้ดกินเวลามากที่สุดจริง ๆ (ต่อยอดจาก Debug Toolbar Panel "Profiling"
  ที่แตะไปแล้วในขั้นตอนที่ 645.3 ของ Part นี้)
- **Part 067**: **Query Optimization** — แก้ปัญหา N+1 query ด้วย `select_related`/
  `prefetch_related` อย่างเป็นระบบ (ต่อยอดโดยตรงจากตัวอย่างในขั้นตอนที่ 645.4)
- **Part 068-069**: **Django Caching Framework** และ **Redis** — ลดภาระฐานข้อมูล
  ด้วยการ cache ผลลัพธ์ที่คำนวณซ้ำบ่อย
- **Part 070**: **Database Indexing** — เข้าใจว่าฐานข้อมูลค้นหาข้อมูลอย่างไรในระดับ
  ลึก และออกแบบ index ให้เหมาะสม
- **Part 071-072**: **Pagination** สำหรับข้อมูลจำนวนมาก และ **Load Testing** เพื่อ
  จำลองผู้ใช้จำนวนมากเข้าระบบพร้อมกันจริง

สิ่งสำคัญคือ **ทุกเครื่องมือที่เรียนใน Phase 7 จะยังคงใช้งานควบคู่ไปตลอด Phase 8**:
เมื่อคุณ optimize query ใน Part 067 คุณจะเขียน test (Part 059-061) เพื่อยืนยันว่า
จำนวน query ลดลงจริงด้วย `assertNumQueries`, เมื่อเพิ่ม caching ใน Part 068-069
Bandit และ mypy (ขั้นตอนที่ 643, 646) จะยังคอยตรวจสอบว่า cache key หรือ serialization
ที่เพิ่มเข้ามาไม่เปิดช่องโหว่ใหม่ และ `make quality` จากขั้นตอนที่ 649 จะยังเป็นคำสั่ง
เดียวที่คุณรันก่อน push ทุกครั้งไม่เปลี่ยนแปลง — Phase 8 คือการนำ "โค้ดที่ถูกต้องและ
มีคุณภาพ" จาก Phase 7 มาทำให้ "เร็วพอสำหรับการใช้งานจริงระดับโลก" โดยไม่ยอมเสีย
คุณภาพที่สร้างมาทั้งหมดไปแม้แต่น้อย

เตรียม `blog` project ของคุณให้พร้อม (ตรวจสอบว่า `make quality` ผ่านหมดแล้ว), เตรียม
ติดตั้ง PostgreSQL ให้มีข้อมูลตัวอย่างจำนวนมากพอสำหรับทดสอบ performance จริง (Factory
Boy จาก Part 063 จะมีประโยชน์มากในการสร้างข้อมูลทดสอบหลักพัน-หมื่นแถว) แล้วไปพบกัน
ที่ Part 066 เพื่อเจาะลึกโลกของ **Performance & Caching** กัน!

ยินดีด้วยอีกครั้งที่จบ Phase 7 แล้ว! 🎉
