# Part 003: Virtual Environment, pip และการจัดการแพ็กเกจ

> **ขั้นตอนที่ 21-30 ของหลักสูตร** | Phase 1: รากฐาน Python & Django
>
> เป้าหมายของ Part นี้: เจาะลึกเครื่องมือจัดการแพ็กเกจ (package manager) ของ Python
> ตั้งแต่ `pip` แบบดั้งเดิมไปจนถึงเครื่องมือยุคใหม่อย่าง `pip-tools`, `uv` และ `Poetry`
> เมื่อจบ Part นี้ คุณจะสามารถจัดการ dependency ของโปรเจกต์ Django ได้อย่างมืออาชีพ
> แยก environment สำหรับ dev/staging/production ได้ถูกต้อง แก้ปัญหา dependency conflict
> เป็น และตรวจสอบช่องโหว่ความปลอดภัยของ package ที่ใช้งานได้ นี่คือทักษะที่แยก
> นักพัฒนามือใหม่ที่ "ติดตั้งแล้วจบ" ออกจากมืออาชีพที่ดูแล dependency ได้อย่างเป็นระบบ
> ตลอดอายุของโปรเจกต์

---

## สารบัญของ Part นี้

- ขั้นตอนที่ 21: pip เจาะลึก (install, uninstall, list, show, git/local path, version specifiers)
- ขั้นตอนที่ 22: requirements.txt แบบมืออาชีพ (pinning และแยกตาม environment)
- ขั้นตอนที่ 23: pip-tools (pip-compile, pip-sync) เพื่อ lock dependency แบบ reproducible
- ขั้นตอนที่ 24: uv — package manager ยุคใหม่จาก Astral ที่เร็วกว่า pip มาก
- ขั้นตอนที่ 25: Poetry และ pyproject.toml (dependency management + packaging)
- ขั้นตอนที่ 26: การจัดการหลาย environment ต่อโปรเจกต์ (dev/staging/production)
- ขั้นตอนที่ 27: ติดตั้ง package จาก private index หรือจาก Git repository โดยตรง
- ขั้นตอนที่ 28: การแก้ปัญหา Dependency Conflict และ `pip check`
- ขั้นตอนที่ 29: Security scanning ของ dependency ด้วย pip-audit และ safety
- ขั้นตอนที่ 30: สรุป Best Practice Checklist และแบบฝึกหัด

---

## ขั้นตอนที่ 21: pip เจาะลึก

### 21.1 pip คืออะไรกันแน่

`pip` (ย่อมาจาก **"Pip Installs Packages"** — ใช่ครับ มันเป็นคำย่อแบบ recursive)
คือ **package installer มาตรฐาน** ของ Python ที่มากับ Python เองตั้งแต่เวอร์ชัน 3.4
เป็นต้นมา หน้าที่หลักของ pip คือดาวน์โหลด ติดตั้ง อัปเดต และถอดถอน package จาก
**PyPI (Python Package Index)** ซึ่งเป็นคลัง package กลางของ Python ที่ https://pypi.org

ก่อนไปต่อ ให้ตรวจสอบว่า pip พร้อมใช้งานและอยู่ใน virtual environment ที่ถูกต้อง
(สมมติว่าคุณ activate `venv` จาก Part 001 ไว้แล้ว):

```bash
# ตรวจสอบเวอร์ชัน pip และตำแหน่งที่ pip ทำงานอยู่
pip --version
# ผลลัพธ์ควรมีลักษณะ:
# pip 24.2 from /path/to/django-mastery-course/venv/lib/python3.12/site-packages/pip (python 3.12)

# สำคัญมาก: ตรวจสอบว่า pip ชี้ไปที่ venv ไม่ใช่ระบบ (global)
which pip      # macOS/Linux
where pip      # Windows

# อัปเดต pip ให้เป็นเวอร์ชันล่าสุดเสมอก่อนเริ่มงาน
python -m pip install --upgrade pip
```

**ทำไมต้องใช้ `python -m pip` แทน `pip` ตรง ๆ?** เพราะ `python -m pip` รับประกันว่า
pip ที่ถูกเรียกใช้เป็นของ interpreter Python ตัวที่กำลัง active อยู่จริง ๆ ป้องกันปัญหา
"pip ติดตั้ง package ไปอีกที่หนึ่ง แต่ python รันจากอีกที่หนึ่ง" ซึ่งเป็นบั๊กคลาสสิก
ที่มือใหม่เจอบ่อยมาก โดยเฉพาะเมื่อเครื่องมี Python หลายเวอร์ชัน

### 21.2 คำสั่ง install พื้นฐานและตัวเลือกที่สำคัญ

```bash
# ติดตั้ง package เวอร์ชันล่าสุด
pip install requests

# ติดตั้งหลาย package พร้อมกัน
pip install requests django pillow

# ติดตั้งเวอร์ชันเจาะจง
pip install django==5.1.2

# ติดตั้งแบบระบุช่วงเวอร์ชัน (version specifier)
pip install "django>=5.1,<5.2"

# อัปเกรด package ที่ติดตั้งอยู่แล้วให้เป็นเวอร์ชันล่าสุด
pip install --upgrade django
pip install -U django          # -U คือ shorthand ของ --upgrade

# ติดตั้งโดยไม่ยุ่งกับ dependency ของ package นั้น (ใช้เมื่อรู้ว่าไม่จำเป็น)
pip install --no-deps some-package

# ติดตั้งแบบไม่ใช้ cache (บังคับดาวน์โหลดใหม่)
pip install --no-cache-dir django

# ติดตั้งแบบ dry-run เพื่อดูว่าจะเกิดอะไรขึ้นโดยไม่ติดตั้งจริง (pip 22.2+)
pip install --dry-run django

# ติดตั้ง pre-release version (alpha, beta, rc)
pip install --pre django
```

### 21.3 Version Specifiers แบบละเอียด

การระบุเวอร์ชันของ package อย่างถูกต้องเป็นทักษะสำคัญมาก เพราะมีผลต่อความเสถียร
ของโปรเจกต์ในระยะยาว ตารางต่อไปนี้สรุป operator ที่ pip รองรับตามมาตรฐาน
[PEP 440](https://peps.python.org/pep-0440/):

| Operator | ความหมาย | ตัวอย่าง | คำอธิบาย |
|---|---|---|---|
| `==` | เท่ากับเป๊ะ | `django==5.1.2` | ติดตั้งเวอร์ชันนี้เท่านั้น (exact pin) |
| `!=` | ไม่เท่ากับ | `django!=5.0.0` | ติดตั้งเวอร์ชันไหนก็ได้ยกเว้นเวอร์ชันนี้ |
| `>=` | มากกว่าหรือเท่ากับ | `django>=5.1` | อย่างน้อยเวอร์ชันนี้ |
| `<=` | น้อยกว่าหรือเท่ากับ | `django<=5.1.9` | ไม่เกินเวอร์ชันนี้ |
| `>` , `<` | มากกว่า / น้อยกว่า | `django<5.2` | ไม่รวมเวอร์ชันที่ระบุ |
| `~=` | Compatible release | `django~=5.1.2` | เทียบเท่า `>=5.1.2, ==5.1.*` (ปลอดภัยจาก breaking change) |
| `===` | Arbitrary equality | `django===5.1.2+local` | เทียบ string ตรง ๆ (ใช้น้อยมาก) |
| (ไม่ระบุ) | ล่าสุดเสมอ | `django` | อันตรายสำหรับ production เพราะควบคุมไม่ได้ |

ตัวอย่างการรวม specifier หลายตัว:

```bash
# ต้อง Django ระหว่าง 5.1 (รวม) ถึงก่อน 5.2
pip install "django>=5.1,<5.2"

# ~= คือตัวที่มืออาชีพนิยมใช้มากที่สุดสำหรับ patch-level safety
# django~=5.1.2 หมายความว่า >=5.1.2 และ <5.2.0
# กล่าวคือ รับ patch version ใหม่ (5.1.3, 5.1.4, ...) แต่ไม่รับ minor version ใหม่ (5.2.0)
pip install "django~=5.1.2"
```

### 21.4 คำสั่งดูข้อมูล package: list, show, freeze

```bash
# แสดงรายการ package ทั้งหมดที่ติดตั้งใน environment ปัจจุบัน
pip list

# แสดงเฉพาะ package ที่ล้าสมัย (มีเวอร์ชันใหม่กว่าบน PyPI)
pip list --outdated

# แสดงรายละเอียดของ package หนึ่งตัว (เวอร์ชัน, ตำแหน่งติดตั้ง, dependency)
pip show django

# ผลลัพธ์ตัวอย่าง:
# Name: Django
# Version: 5.1.2
# Summary: A high-level Python web framework...
# Home-page: https://www.djangoproject.com/
# Location: /path/to/venv/lib/python3.12/site-packages
# Requires: asgiref, sqlparse
# Required-by:

# แสดงในรูปแบบที่ใช้บันทึกลง requirements.txt ได้ (pinned เป๊ะทุกตัว)
pip freeze

# แสดง dependency tree แบบเห็นความสัมพันธ์ชัดเจน (ต้องติดตั้งเพิ่ม)
pip install pipdeptree
pipdeptree
```

ตัวอย่างผลลัพธ์ของ `pipdeptree` ที่ช่วยให้เห็นว่าใครพึ่งพาใคร:

```
Django==5.1.2
├── asgiref==3.8.1 [required: >=3.8.1,<4, installed: 3.8.1]
└── sqlparse==0.5.1 [required: >=0.3.1, installed: 0.5.1]
djangorestframework==3.15.2
└── Django==5.1.2 [required: >=4.2, installed: 5.1.2]
```

### 21.5 คำสั่ง uninstall

```bash
# ถอดถอน package เดียว (จะถาม confirm ก่อนลบ)
pip uninstall requests

# ถอดถอนแบบไม่ต้องยืนยัน (ใช้ระวังใน script)
pip uninstall -y requests

# ถอดถอนหลาย package พร้อมกัน
pip uninstall -y requests pillow

# ถอดถอนทุก package ตามรายการในไฟล์ (มักใช้ล้าง environment ก่อนติดตั้งใหม่)
pip uninstall -y -r requirements.txt
```

**ข้อควรระวัง**: `pip uninstall` จะลบเฉพาะ package ที่ระบุเท่านั้น ไม่ได้ลบ
dependency ที่ package นั้นดึงมาด้วยอัตโนมัติ (ต่างจาก `apt remove --autoremove`
ใน Linux) ถ้าต้องการล้าง environment ทั้งหมดให้สะอาด วิธีที่ปลอดภัยที่สุดคือ
ลบโฟลเดอร์ venv ทิ้งแล้วสร้างใหม่:

```bash
deactivate
rm -rf venv
python3 -m venv venv
source venv/bin/activate
pip install -r requirements.txt
```

### 21.6 ติดตั้ง package จาก Local Path

บางครั้งคุณต้องการติดตั้ง package ที่ยังไม่ได้ publish ขึ้น PyPI เช่น library
ภายในบริษัท หรือกำลังพัฒนา package ของตัวเองอยู่:

```bash
# ติดตั้งจากโฟลเดอร์ local (ต้องมีไฟล์ pyproject.toml หรือ setup.py ในโฟลเดอร์นั้น)
pip install /path/to/my-local-package

# ติดตั้งแบบ "editable mode" — เหมาะกับตอนพัฒนา package เอง
# การแก้โค้ดใน source จะมีผลทันทีโดยไม่ต้อง reinstall ใหม่
pip install -e /path/to/my-local-package

# ตัวอย่างจริง: กำลังพัฒนา internal package "company-utils" อยู่ใน sibling folder
pip install -e ../company-utils

# ติดตั้งจากไฟล์ archive (.tar.gz, .whl) โดยตรง
pip install ./dist/mypackage-1.0.0-py3-none-any.whl
```

`-e` (editable install) เป็นเทคนิคที่ทีมพัฒนา library ภายในองค์กรใช้กันเป็น
มาตรฐาน เพราะทำให้ทดสอบการเปลี่ยนแปลงของ shared library ร่วมกับโปรเจกต์หลัก
ได้ทันทีโดยไม่ต้อง publish ขึ้น PyPI ก่อน

### 21.7 ติดตั้ง package จาก Git Repository โดยตรง

```bash
# ติดตั้งจาก branch ล่าสุดของ default branch
pip install git+https://github.com/django/django.git

# ระบุ branch เฉพาะ
pip install git+https://github.com/django/django.git@stable/5.1.x

# ระบุ tag เฉพาะ (แนะนำมากกว่า branch เพื่อความ reproducible)
pip install git+https://github.com/django/django.git@5.1.2

# ระบุ commit hash เฉพาะเจาะจง (แม่นยำที่สุด)
pip install git+https://github.com/django/django.git@a1b2c3d4

# ติดตั้งผ่าน SSH (เหมาะกับ private repository ที่ตั้งค่า SSH key ไว้แล้ว)
pip install git+ssh://git@github.com/mycompany/internal-lib.git@main

# ระบุ subdirectory ถ้า package ไม่ได้อยู่ที่ root ของ repo
pip install "git+https://github.com/mycompany/monorepo.git@main#subdirectory=packages/mylib"
```

รูปแบบนี้มีประโยชน์มากเมื่อต้องการใช้ fix หรือ feature ที่ยังไม่ถูก release อย่างเป็น
ทางการบน PyPI เช่น กำลังรอ pull request ของตัวเองถูก merge เข้า upstream project

### 21.8 การตรวจสอบไฟล์ที่ pip ติดตั้งจริง

```bash
# ดูรายชื่อไฟล์ทั้งหมดที่ package นี้ติดตั้งลงเครื่อง
pip show -f django | head -30

# ตรวจสอบว่า package ไหนขึ้นต้นด้วยชื่อที่กำหนด (ใช้ค้นหา)
pip list | grep -i django
```

---

## ขั้นตอนที่ 22: requirements.txt แบบมืออาชีพ

### 22.1 ปัญหาของ `pip freeze` แบบตรงไปตรงมา

ใน Part 001 เราใช้ `pip freeze > requirements.txt` ซึ่งใช้ได้ในระดับเริ่มต้น
แต่มีปัญหาสำคัญเมื่อโปรเจกต์เติบโตขึ้น:

1. **ปนกันหมดระหว่าง dependency ตรง ๆ กับ dependency ของ dependency**:
   คุณสั่ง `pip install django` แต่ `pip freeze` จะบันทึกทั้ง `Django`, `asgiref`,
   `sqlparse` รวมกันหมด ทำให้ไม่รู้ว่าอันไหนคือสิ่งที่คุณตั้งใจติดตั้งจริง ๆ
2. **ไม่แยกตาม environment**: เครื่องมือทดสอบอย่าง `pytest` หรือ linter อย่าง
   `ruff` ไม่ควรถูกติดตั้งบน production server แต่ `pip freeze` จะรวมทุกอย่าง
   ไว้ในไฟล์เดียว
3. **Comment และ source ต้นทางหายไป**: ไม่มีทางรู้ว่าทำไมถึงต้องใช้เวอร์ชันนี้

### 22.2 โครงสร้าง requirements แบบมืออาชีพ: แยกเป็นหลายไฟล์

มาตรฐานที่ทีมพัฒนา Django มืออาชีพใช้กันทั่วไปคือการสร้างโฟลเดอร์ `requirements/`
แล้วแยกไฟล์ตาม environment โดยใช้กลไก **`-r` (include)** ของ pip เพื่อลดการซ้ำซ้อน:

```
django-mastery-course/
├── requirements/
│   ├── base.txt          # dependency ที่ทุก environment ต้องใช้
│   ├── dev.txt           # base.txt + เครื่องมือพัฒนา (debug toolbar, ipython)
│   ├── test.txt          # base.txt + เครื่องมือทดสอบ (pytest, factory-boy)
│   └── prod.txt          # base.txt + เครื่องมือ production (gunicorn, sentry)
├── manage.py
└── ...
```

**`requirements/base.txt`** — dependency หลักที่จำเป็นทุกที่ไม่ว่าจะรันที่ไหน:

```
# requirements/base.txt
# --- Core framework ---
Django==5.1.2
djangorestframework==3.15.2

# --- Database ---
psycopg[binary]==3.2.3

# --- Environment & configuration ---
python-decouple==3.8
dj-database-url==2.3.0

# --- Image handling ---
Pillow==11.0.0
```

**`requirements/dev.txt`** — สำหรับเครื่องนักพัฒนาแต่ละคน:

```
# requirements/dev.txt
-r base.txt

# --- Debugging & development tools ---
django-debug-toolbar==4.4.6
ipython==8.29.0
django-extensions==3.2.3

# --- Code quality ---
ruff==0.7.4
mypy==1.13.0
django-stubs==5.1.1

# --- Testing (นักพัฒนามักรันเทสต์บนเครื่องตัวเองด้วย) ---
-r test.txt
```

**`requirements/test.txt`** — สำหรับ CI/CD และการรันเทสต์:

```
# requirements/test.txt
-r base.txt

pytest==8.3.3
pytest-django==4.9.0
pytest-cov==6.0.0
factory-boy==3.3.1
```

**`requirements/prod.txt`** — สำหรับ production server เท่านั้น เน้นความ
เรียบง่ายและปลอดภัย ไม่มี debugging tools ปะปน:

```
# requirements/prod.txt
-r base.txt

# --- WSGI server ---
gunicorn==23.0.0

# --- Static files ---
whitenoise==6.8.2

# --- Error monitoring ---
sentry-sdk==2.18.0

# --- Object storage (สำหรับ media files บน production) ---
django-storages[s3]==1.14.4
```

### 22.3 การติดตั้งตาม environment

```bash
# บนเครื่อง dev
pip install -r requirements/dev.txt

# ใน CI pipeline (รันเทสต์เท่านั้น ไม่ต้องมี dev tools)
pip install -r requirements/test.txt

# บน production server
pip install -r requirements/prod.txt
```

### 22.4 กฎการ Pin เวอร์ชันอย่างมืออาชีพ

| ระดับความเข้มงวด | รูปแบบ | เหมาะกับ |
|---|---|---|
| **Exact pin** (แนะนำสำหรับ production) | `Django==5.1.2` | Production, CI — ต้องการผลลัพธ์เดิมทุกครั้ง (reproducible build) |
| **Compatible release** | `Django~=5.1.2` | ไลบรารีที่ต้องการ security patch อัตโนมัติแต่ไม่เปลี่ยน API |
| **Range** | `Django>=5.1,<5.2` | เมื่อพัฒนา library ที่ต้องรองรับหลายเวอร์ชันของ dependency |
| **ไม่ pin เลย** | `Django` | **ห้ามใช้ใน production เด็ดขาด** — เสี่ยงต่อ breaking change แบบไม่รู้ตัว |

**กฎเหล็ก**: ไฟล์ `requirements/prod.txt` (หรือไฟล์ lock ที่จะพูดถึงใน
ขั้นตอนที่ 23) **ต้อง exact-pin ทุก package รวมถึง transitive dependency ด้วย**
เพื่อให้มั่นใจว่า `pip install` ในวันนี้ กับอีก 6 เดือนข้างหน้า ให้ผลลัพธ์เหมือนกัน
เป๊ะทุกประการ — นี่คือหลักการที่เรียกว่า **reproducible build**

### 22.5 คอมเมนต์และการจัดกลุ่มที่ดี

การเขียน `requirements.txt` ที่ดีไม่ใช่แค่รายการเวอร์ชัน แต่ควรมีคอมเมนต์อธิบาย
ว่าทำไมถึงต้องมี package แต่ละตัว โดยเฉพาะตัวที่ไม่ชัดเจนในตัวเอง:

```
# requirements/base.txt

Django==5.1.2

# ใช้ psycopg (v3) แทน psycopg2 เพราะรองรับ async และเป็นเวอร์ชันที่ Django
# แนะนำอย่างเป็นทางการตั้งแต่ Django 4.2 เป็นต้นไป
psycopg[binary]==3.2.3

# ใช้จัดการ environment variables — ดู Part 010 (Django Settings)
python-decouple==3.8

# Pillow จำเป็นสำหรับ ImageField ใน models.py (ดู Part 058)
Pillow==11.0.0
```

---

## ขั้นตอนที่ 23: pip-tools เพื่อ Lock Dependency แบบ Reproducible

### 23.1 ปัญหาที่ pip-tools แก้

การเขียน `requirements/base.txt` ด้วยมือแล้ว exact-pin ทุกตัวเองมีปัญหา:
เวลาคุณเขียนแค่ `Django==5.1.2` คุณต้องไปหาเองว่า `asgiref` และ `sqlparse`
(dependency ของ Django) ควร pin เป็นเวอร์ชันอะไรด้วย ยิ่งโปรเจกต์มี dependency
เยอะ ยิ่งจัดการด้วยมือยากขึ้นเรื่อย ๆ

**pip-tools** แก้ปัญหานี้ด้วยแนวคิด **แยกไฟล์ "สิ่งที่ต้องการ" (`.in`) ออกจากไฟล์
"สิ่งที่ล็อกไว้" (`.txt`)**:

```
requirements/base.in   →  (คุณเขียนเอง) "ฉันต้องการ Django และ psycopg"
requirements/base.txt  →  (สร้างอัตโนมัติ) รายการ exact-pin ของทุก package
                           รวม dependency ทั้งหมดแบบ resolve แล้ว
```

### 23.2 ติดตั้ง pip-tools

```bash
pip install pip-tools

# ตรวจสอบว่าติดตั้งสำเร็จ
pip-compile --version
pip-sync --version
```

### 23.3 เขียนไฟล์ `.in` (input file)

```
# requirements/base.in
django>=5.1,<5.2
djangorestframework
psycopg[binary]
python-decouple
Pillow
```

```
# requirements/dev.in
-c base.txt

-r base.in
django-debug-toolbar
ipython
django-extensions
ruff
mypy
django-stubs
```

```
# requirements/test.in
-c base.txt

-r base.in
pytest
pytest-django
pytest-cov
factory-boy
```

```
# requirements/prod.in
-r base.in
gunicorn
whitenoise
sentry-sdk
django-storages[s3]
```

**หมายเหตุเรื่อง `-c base.txt`**: นี่คือ **constraint file** ที่บอกให้
`pip-compile` ใช้เวอร์ชันเดียวกับที่ `base.txt` ล็อกไว้แล้วสำหรับ dependency
ที่ซ้ำกัน (เช่น Django ที่ทั้ง dev และ test ต้องใช้ตัวเดียวกับ base) ป้องกันไม่ให้
แต่ละไฟล์ resolve เวอร์ชันต่างกันโดยไม่ตั้งใจ

### 23.4 คำสั่ง pip-compile: สร้างไฟล์ lock

```bash
# compile base.in -> base.txt (resolve dependency ทั้งหมดและ pin ให้)
pip-compile requirements/base.in --output-file requirements/base.txt

# compile ไฟล์อื่น ๆ ตามลำดับ (ต้อง compile base ก่อนเสมอเพราะมีการอ้างอิงถึง)
pip-compile requirements/dev.in --output-file requirements/dev.txt
pip-compile requirements/test.in --output-file requirements/test.txt
pip-compile requirements/prod.in --output-file requirements/prod.txt

# เพิ่ม hash ของแต่ละ package เพื่อความปลอดภัยสูงสุด (ป้องกัน supply-chain attack)
pip-compile --generate-hashes requirements/prod.in --output-file requirements/prod.txt

# อัปเกรด package ทั้งหมดให้เป็นเวอร์ชันล่าสุดที่ยัง compatible กัน
pip-compile --upgrade requirements/base.in --output-file requirements/base.txt

# อัปเกรดเฉพาะ package เดียว
pip-compile --upgrade-package django requirements/base.in --output-file requirements/base.txt
```

ผลลัพธ์ของ `requirements/base.txt` ที่ pip-compile สร้างจะมีลักษณะดังนี้
(สังเกตว่ามันบอกด้วยว่า package แต่ละตัวมาจาก dependency ของอะไร — traceability
ที่ `pip freeze` ให้ไม่ได้):

```
#
# This file is autogenerated by pip-compile with Python 3.12
# by the following command:
#
#    pip-compile requirements/base.in --output-file requirements/base.txt
#
asgiref==3.8.1
    # via django
django==5.1.2
    # via -r requirements/base.in
djangorestframework==3.15.2
    # via -r requirements/base.in
pillow==11.0.0
    # via -r requirements/base.in
psycopg==3.2.3
    # via -r requirements/base.in
python-decouple==3.8
    # via -r requirements/base.in
sqlparse==0.5.1
    # via django
```

### 23.5 คำสั่ง pip-sync: ทำให้ environment ตรงกับไฟล์ lock เป๊ะ

`pip install -r requirements.txt` จะ **เพิ่ม** package ตามไฟล์ แต่ไม่ลบ package
ที่ไม่ได้อยู่ในไฟล์ออก ทำให้ environment ค่อย ๆ "สกปรก" จาก package เก่าที่
ไม่ได้ใช้แล้ว `pip-sync` แก้ปัญหานี้โดยทำให้ environment ตรงกับไฟล์ `.txt`
**แบบเป๊ะ 100%** (ติดตั้งของที่ขาด และ **ถอดถอน** ของที่ไม่มีในไฟล์ออกด้วย):

```bash
# ทำให้ venv ตรงกับ dev.txt เป๊ะ (ลบ package ที่ไม่อยู่ในไฟล์ออกอัตโนมัติ)
pip-sync requirements/dev.txt

# sync ได้หลายไฟล์พร้อมกัน (รวม base + test)
pip-sync requirements/base.txt requirements/test.txt
```

**คำเตือน**: `pip-sync` เป็นคำสั่งที่ทำลายล้าง (destructive) — มันจะถอดถอน
ทุก package ที่ไม่ได้อยู่ใน `.txt` ที่ระบุ ห้ามรันโดยไม่ตรวจสอบก่อนว่าไฟล์ครบถ้วน

### 23.6 Workflow การทำงานประจำวันด้วย pip-tools

```bash
# 1. ต้องการเพิ่ม package ใหม่ -> แก้ไฟล์ .in
echo "django-filter" >> requirements/base.in

# 2. compile ใหม่ทุกไฟล์ที่เกี่ยวข้อง (เรียงจาก base ไปยังไฟล์ที่ constraint จาก base)
pip-compile requirements/base.in --output-file requirements/base.txt
pip-compile requirements/dev.in --output-file requirements/dev.txt

# 3. sync environment ให้ตรงกับไฟล์ล่าสุด
pip-sync requirements/dev.txt

# 4. commit ทั้งไฟล์ .in และ .txt ลง Git เสมอ (ทั้งคู่สำคัญ!)
git add requirements/
git commit -m "Add django-filter dependency"
```

เพื่อความสะดวก ทีมมืออาชีพมักเขียน `Makefile` ครอบคำสั่งเหล่านี้:

```makefile
# Makefile
.PHONY: compile sync upgrade

compile:
	pip-compile requirements/base.in --output-file requirements/base.txt
	pip-compile requirements/dev.in --output-file requirements/dev.txt
	pip-compile requirements/test.in --output-file requirements/test.txt
	pip-compile requirements/prod.in --output-file requirements/prod.txt

sync:
	pip-sync requirements/dev.txt

upgrade:
	pip-compile --upgrade requirements/base.in --output-file requirements/base.txt
```

จากนั้นทีมงานเพียงพิมพ์ `make compile` และ `make sync` โดยไม่ต้องจำคำสั่งยาว ๆ

---

## ขั้นตอนที่ 24: uv — Package Manager ยุคใหม่จาก Astral

### 24.1 uv คืออะไร และทำไมถึงเร็วกว่า pip มาก

**uv** คือเครื่องมือจัดการ Python package และ virtual environment ที่พัฒนาโดย
**Astral** (บริษัทเดียวกับที่สร้าง `Ruff` — linter สุดเร็วที่เราติดตั้งใน VS Code
ใน Part 001) เขียนด้วยภาษา **Rust** ทำให้เร็วกว่า pip **10-100 เท่า** ในการ
resolve และติดตั้ง dependency

เหตุผลหลักที่ uv เร็วมาก:

1. **เขียนด้วย Rust** ไม่ใช่ Python (ไม่มี overhead ของ Python interpreter)
2. **Dependency resolver แบบ parallel** ดาวน์โหลดและ resolve หลาย package
   พร้อมกันจริง ๆ
3. **Global cache แบบ hard-link**: package ที่เคยดาวน์โหลดแล้วในโปรเจกต์หนึ่ง
   จะถูก hard-link (ไม่ใช่ copy) ไปยังโปรเจกต์อื่นที่ใช้ package เดียวกัน
   ทำให้แทบไม่เปลืองพื้นที่ดิสก์และติดตั้งซ้ำเร็วมาก
4. **รวมฟังก์ชันของหลายเครื่องมือไว้ในตัวเดียว**: แทนที่ pip + venv + pip-tools
   + pyenv (บางส่วน) ด้วยไบนารีเดียว

ในปี 2025-2026 ทีมพัฒนา Python จำนวนมาก (รวมถึงทีมที่ดูแล Django เอง) เริ่มย้าย
CI/CD pipeline มาใช้ uv เพราะลดเวลา build จากหลักนาทีเหลือหลักวินาที

### 24.2 การติดตั้ง uv

```bash
# macOS / Linux
curl -LsSf https://astral.sh/uv/install.sh | sh

# Windows PowerShell
powershell -ExecutionPolicy ByPass -c "irm https://astral.sh/uv/install.ps1 | iex"

# หรือติดตั้งผ่าน pip ก็ได้ (ช้ากว่าเล็กน้อยแต่สะดวกถ้าไม่อยากรัน installer script)
pip install uv

# หรือผ่าน Homebrew บน macOS
brew install uv

# ตรวจสอบการติดตั้ง
uv --version
```

### 24.3 สร้าง Virtual Environment ด้วย uv

```bash
# สร้าง venv ด้วย uv (เร็วกว่า python -m venv มาก เพราะ uv ไม่ต้อง copy ไฟล์เยอะ)
uv venv

# ผลลัพธ์: สร้างโฟลเดอร์ .venv (ชื่อ default ของ uv ต่างจาก venv ของ Python มาตรฐาน)
# .venv/
#     bin/
#         python
#         activate

# ระบุ Python เวอร์ชันที่ต้องการ (uv จะดาวน์โหลด Python เวอร์ชันนั้นให้อัตโนมัติ
# ถ้ายังไม่มีในเครื่อง — ไม่ต้องใช้ pyenv แยกต่างหากอีกต่อไป!)
uv venv --python 3.12

# activate เหมือน venv ปกติทุกประการ
source .venv/bin/activate       # macOS/Linux
.venv\Scripts\activate          # Windows
```

### 24.4 ติดตั้ง package ด้วย uv pip

uv ออกแบบให้ใช้ interface ที่คุ้นเคยกับ pip เดิมได้เลย เพียงเติม `uv` นำหน้า:

```bash
# ติดตั้ง package (syntax เหมือน pip ทุกอย่าง)
uv pip install django

# ติดตั้งจากไฟล์ requirements.txt
uv pip install -r requirements/dev.txt

# ติดตั้งแบบระบุเวอร์ชัน
uv pip install "django>=5.1,<5.2"

# แสดงรายการ package ที่ติดตั้ง
uv pip list

# แสดงรายละเอียด package
uv pip show django

# freeze รายการ package
uv pip freeze > requirements.txt

# compile .in -> .txt (uv มี pip-compile ในตัว เร็วกว่า pip-tools มาก!)
uv pip compile requirements/base.in -o requirements/base.txt

# sync environment ให้ตรงกับไฟล์ (แทนที่ pip-sync)
uv pip sync requirements/base.txt

# ถอดถอน package
uv pip uninstall django
```

### 24.5 uv project mode: pyproject.toml + uv.lock

นอกจากโหมด "pip-compatible" ข้างต้น uv ยังมีโหมดจัดการโปรเจกต์แบบสมบูรณ์
คล้าย Poetry ที่จะพูดถึงในขั้นตอนถัดไป แต่เร็วกว่ามาก:

```bash
# สร้างโปรเจกต์ Django ใหม่ด้วย uv (สร้าง pyproject.toml, .venv, uv.lock อัตโนมัติ)
uv init django-mastery-course
cd django-mastery-course

# เพิ่ม dependency เข้าโปรเจกต์ (uv จะ resolve, ติดตั้ง, และอัปเดต uv.lock ให้ทันที)
uv add django
uv add "djangorestframework>=3.15"

# เพิ่ม dependency เฉพาะกลุ่ม dev (คล้าย extras)
uv add --dev pytest pytest-django ruff mypy

# ลบ dependency ออก
uv remove django-debug-toolbar

# รันคำสั่งภายใน environment ของโปรเจกต์ โดยไม่ต้อง activate เอง
uv run python manage.py runserver
uv run python manage.py migrate
uv run pytest

# sync environment ให้ตรงกับ uv.lock เป๊ะ (เหมือน pip-sync แต่เร็วกว่ามาก)
uv sync

# sync เฉพาะ production dependency (ไม่รวม dev group)
uv sync --no-dev
```

### 24.6 ไฟล์ pyproject.toml ที่ uv สร้างให้

```toml
# pyproject.toml
[project]
name = "django-mastery-course"
version = "0.1.0"
description = "Django learning project"
requires-python = ">=3.12"
dependencies = [
    "django>=5.1,<5.2",
    "djangorestframework>=3.15",
]

[dependency-groups]
dev = [
    "pytest>=8.3.3",
    "pytest-django>=4.9.0",
    "ruff>=0.7.4",
    "mypy>=1.13.0",
]

[tool.uv]
package = false
```

### 24.7 uv.lock: ไฟล์ล็อกแบบ Cross-Platform

`uv.lock` คือไฟล์ที่ uv สร้างอัตโนมัติเมื่อรัน `uv add`/`uv sync` บันทึกเวอร์ชัน
เป๊ะของทุก dependency (รวม transitive dependency) พร้อม hash สำหรับตรวจสอบ
ความถูกต้อง และรองรับหลาย platform ในไฟล์เดียว (ต่างจาก `requirements.txt`
ที่ pin เฉพาะ platform ที่ compile) **ต้อง commit ไฟล์นี้ลง Git เสมอ**
เพื่อให้ทุกคนในทีมและ CI/CD ใช้เวอร์ชัน dependency ชุดเดียวกันเป๊ะ

```bash
# ตรวจสอบว่า uv.lock ตรงกับ pyproject.toml หรือไม่ (ใช้ใน CI)
uv lock --check

# อัปเดตทุก dependency ให้เป็นเวอร์ชันล่าสุดที่ compatible กัน
uv lock --upgrade

# อัปเดตเฉพาะ package เดียว
uv lock --upgrade-package django
```

### 24.8 ตารางเปรียบเทียบความเร็ว pip vs uv

| การทำงาน | pip | uv | เร็วขึ้นประมาณ |
|---|---|---|---|
| สร้าง virtual environment | ~2-3 วินาที | ~0.02-0.1 วินาที | 20-100 เท่า |
| ติดตั้ง Django + dependency (cold cache) | ~3-5 วินาที | ~0.3-0.6 วินาที | 8-10 เท่า |
| ติดตั้งซ้ำ (warm cache) | ~2-3 วินาที | ~0.05 วินาที | 40-60 เท่า |
| Resolve dependency ที่ซับซ้อน (Django + DRF + Celery ฯลฯ) | หลักสิบวินาที | หลักวินาที | 10-20 เท่า |

ในโปรเจกต์จริงที่มี dependency หลายสิบตัว ความแตกต่างนี้มีผลชัดเจนมากต่อเวลา
build ของ CI/CD pipeline ซึ่งอาจรันหลายสิบครั้งต่อวัน

---

## ขั้นตอนที่ 25: Poetry และ pyproject.toml

### 25.1 Poetry คืออะไร

**Poetry** คือเครื่องมือจัดการ dependency และ packaging ของ Python ที่ได้รับ
ความนิยมมาตั้งแต่ปี 2018 ก่อนที่ uv จะถือกำเนิด จุดเด่นของ Poetry คือรวม
การจัดการ **dependency, virtual environment, การ build package, และการ publish
ขึ้น PyPI** ไว้ในเครื่องมือเดียว โดยใช้ไฟล์ `pyproject.toml` เป็นศูนย์กลาง
(มาตรฐานที่ Python กำหนดใน PEP 518/621)

แม้ uv จะเร็วกว่ามากในปี 2025-2026 แต่ Poetry ยังถูกใช้งานอย่างแพร่หลายในทีม
จำนวนมาก โดยเฉพาะทีมที่ต้อง publish library ขึ้น PyPI จริงจัง เพราะ Poetry
มีเครื่องมือ publish ที่สมบูรณ์และเป็นที่ยอมรับมานาน

### 25.2 การติดตั้ง Poetry

```bash
# วิธีที่แนะนำอย่างเป็นทางการ (ติดตั้งแยกจาก Python environment ของโปรเจกต์)
curl -sSL https://install.python-poetry.org | python3 -

# macOS ผ่าน Homebrew
brew install poetry

# ตรวจสอบการติดตั้ง
poetry --version

# ตั้งค่าให้ Poetry สร้าง virtual environment ไว้ในโฟลเดอร์โปรเจกต์เอง
# (ค่า default ของ Poetry คือเก็บ venv รวมไว้ที่อื่นในเครื่อง ซึ่งหลายคนไม่ชอบ)
poetry config virtualenvs.in-project true
```

### 25.3 เริ่มต้นโปรเจกต์ใหม่ด้วย Poetry

```bash
# สร้างโปรเจกต์ใหม่ทั้งโฟลเดอร์
poetry new django-mastery-course

# หรือถ้ามีโฟลเดอร์โปรเจกต์อยู่แล้ว ให้ init แทน
cd django-mastery-course
poetry init
```

`poetry init` จะถามคำถามทีละข้อ (ชื่อโปรเจกต์, เวอร์ชัน, description, Python
version ที่รองรับ) แล้วสร้างไฟล์ `pyproject.toml` ให้อัตโนมัติ

### 25.4 โครงสร้าง pyproject.toml แบบ Poetry

```toml
# pyproject.toml
[tool.poetry]
name = "django-mastery-course"
version = "0.1.0"
description = "โปรเจกต์เรียนรู้ Django อย่างเป็นระบบ"
authors = ["Your Name <you@example.com>"]
readme = "README.md"
package-mode = false   # true เมื่อจะ publish เป็น library, false สำหรับ Django project ทั่วไป

[tool.poetry.dependencies]
python = "^3.12"
django = "^5.1"
djangorestframework = "^3.15"
psycopg = {extras = ["binary"], version = "^3.2"}
python-decouple = "^3.8"
pillow = "^11.0"

[tool.poetry.group.dev.dependencies]
django-debug-toolbar = "^4.4"
ipython = "^8.29"
ruff = "^0.7"
mypy = "^1.13"
django-stubs = "^5.1"

[tool.poetry.group.test.dependencies]
pytest = "^8.3"
pytest-django = "^4.9"
pytest-cov = "^6.0"
factory-boy = "^3.3"

[build-system]
requires = ["poetry-core"]
build-backend = "poetry.core.masonry.api"
```

**หมายเหตุเรื่อง `^` (caret)**: Poetry ใช้สัญลักษณ์ `^5.1` ซึ่งหมายถึง
"ยอมรับเวอร์ชันใด ๆ ที่ `>=5.1.0, <6.0.0`" (ไม่ทำให้ major version เปลี่ยน)
ต่างจาก `~=5.1` ของ pip ที่ล็อกแน่นกว่า (ล็อกที่ minor version) ผู้เริ่มต้นมัก
สับสนสอง operator นี้ ต้องระวังให้ดี

| Syntax | ระบบ | ความหมาย |
|---|---|---|
| `^5.1.2` | Poetry (Caret) | `>=5.1.2, <6.0.0` |
| `~5.1.2` | Poetry (Tilde) | `>=5.1.2, <5.2.0` |
| `~=5.1.2` | pip/PEP 440 | `>=5.1.2, <5.2.0` (เทียบเท่า Tilde ของ Poetry) |

### 25.5 คำสั่งจัดการ dependency ของ Poetry

```bash
# เพิ่ม dependency ใหม่ (จะ resolve, ติดตั้ง, และอัปเดต pyproject.toml + poetry.lock)
poetry add django

# เพิ่มเข้ากลุ่มเฉพาะ (dev, test)
poetry add --group dev ruff mypy
poetry add --group test pytest pytest-django

# ระบุเวอร์ชันเจาะจง
poetry add "django@^5.1"

# ลบ dependency
poetry remove django-debug-toolbar

# ติดตั้งทุก dependency ตาม pyproject.toml + poetry.lock (ใช้ตอน clone โปรเจกต์ใหม่)
poetry install

# ติดตั้งเฉพาะกลุ่ม production (ไม่รวม dev/test) — ใช้บน production server
poetry install --only main

# ติดตั้งรวม dev แต่ไม่รวม test
poetry install --with dev --without test

# อัปเดต dependency ทั้งหมดให้เป็นเวอร์ชันล่าสุดที่ยัง compatible
poetry update

# อัปเดตเฉพาะ package เดียว
poetry update django
```

### 25.6 การรันคำสั่งภายใน Poetry environment

```bash
# วิธีที่ 1: ใช้ poetry run (แนะนำ ไม่ต้อง activate เอง)
poetry run python manage.py runserver
poetry run python manage.py migrate
poetry run pytest

# วิธีที่ 2: เข้าไปใน shell ของ virtual environment โดยตรง
poetry env activate     # Poetry 2.0+ แสดงคำสั่ง activate ให้คัดลอกไปรัน
# หรือเวอร์ชันเก่ากว่า:
poetry shell

# ตรวจสอบว่า virtual environment อยู่ที่ไหน
poetry env info
poetry env list
```

### 25.7 poetry.lock: ไฟล์ล็อกของ Poetry

เช่นเดียวกับ `uv.lock`, ไฟล์ `poetry.lock` บันทึกเวอร์ชันเป๊ะของทุก dependency
(รวม transitive) พร้อม hash สำหรับความปลอดภัย **ต้อง commit ลง Git เสมอ**:

```bash
# ตรวจสอบว่า poetry.lock สอดคล้องกับ pyproject.toml หรือไม่ (รันใน CI ก่อน install)
poetry check --lock

# export poetry.lock ออกมาเป็น requirements.txt แบบดั้งเดิม
# (มีประโยชน์เมื่อ deploy ไปยัง platform ที่รองรับแค่ requirements.txt)
poetry self add poetry-plugin-export
poetry export -f requirements.txt --output requirements.txt --without-hashes
poetry export -f requirements.txt --output requirements/prod.txt --only main
```

### 25.8 ตารางเปรียบเทียบ pip+pip-tools vs uv vs Poetry

| คุณสมบัติ | pip + pip-tools | uv | Poetry |
|---|---|---|---|
| ความเร็ว | ปานกลาง | เร็วที่สุด (Rust) | ปานกลาง-ช้า (Python) |
| จัดการ venv ในตัว | ❌ (ต้องใช้ `venv` แยก) | ✅ | ✅ |
| Lock file reproducible | ✅ (ผ่าน pip-compile) | ✅ (`uv.lock`) | ✅ (`poetry.lock`) |
| จัดการ Python version (คล้าย pyenv) | ❌ | ✅ | ❌ |
| Build/Publish package ขึ้น PyPI | ❌ (ต้องใช้ `build`/`twine`) | ✅ (`uv build`, `uv publish`) | ✅ (เป็นจุดแข็งดั้งเดิม) |
| Ecosystem/ความนิยมปี 2026 | มาตรฐานดั้งเดิม ใช้ได้ทุกที่ | เติบโตเร็วมาก กลายเป็นตัวเลือกยอดนิยม | ยังนิยมมาก โดยเฉพาะทีมที่ publish library |
| ความง่ายสำหรับมือใหม่ | ง่ายที่สุด (เข้าใจ pip ตรง ๆ) | ง่าย (syntax คล้าย pip) | ต้องเรียนรู้แนวคิดใหม่บ้าง |
| เหมาะกับหลักสูตรนี้ | ✅ ใช้เป็นฐานตลอดหลักสูตร | แนะนำเป็นทางเลือกเร็ว | ทางเลือกเพิ่มเติม |

**คำแนะนำของหลักสูตรนี้**: เราจะยึด `pip` + `requirements/*.txt` เป็นมาตรฐาน
หลักตลอดหลักสูตร เพราะเข้าใจง่ายที่สุดและใช้ได้กับทุก deployment platform
แต่แนะนำให้คุณลองใช้ `uv` ควบคู่ไปด้วย เนื่องจากในปี 2025-2026 ทีมงานระดับ
มืออาชีพจำนวนมากได้เปลี่ยนมาใช้ uv เป็นเครื่องมือหลักแล้ว และรูปแบบคำสั่งก็
คล้ายกับ pip มากจนสลับไปมาได้ไม่ยาก

---

## ขั้นตอนที่ 26: การจัดการหลาย Environment ต่อโปรเจกต์

### 26.1 ทำไมต้องแยก Environment

โปรเจกต์ Django ระดับมืออาชีพแทบทุกโปรเจกต์ต้องรันในหลายสภาพแวดล้อมที่มี
ความต้องการต่างกัน:

| Environment | จุดประสงค์ | ตัวอย่าง package พิเศษ |
|---|---|---|
| **Local / Development** | นักพัฒนาเขียนโค้ดบนเครื่องตัวเอง | `django-debug-toolbar`, `ipython`, hot-reload tools |
| **Test / CI** | รันอัตโนมัติเมื่อ push โค้ด | `pytest`, `pytest-cov`, `factory-boy` |
| **Staging** | ทดสอบก่อนขึ้น production จริง (เหมือน production ทุกอย่าง) | เหมือน production แต่อาจมี logging เพิ่ม |
| **Production** | ระบบจริงที่ผู้ใช้เข้าถึง | `gunicorn`, `sentry-sdk`, `whitenoise` |

การใช้ dependency ชุดเดียวกันทุก environment มีความเสี่ยง เช่น หากติดตั้ง
`django-debug-toolbar` บน production โดยไม่ได้ตั้งใจ อาจเปิดช่องให้ผู้ไม่หวังดี
เห็นข้อมูล internal ของระบบผ่าน debug panel — เป็นช่องโหว่ความปลอดภัยจริงที่
เคยเกิดขึ้นในหลายบริษัท

### 26.2 โครงสร้างไฟล์แนะนำแบบเต็ม (ใช้ pip-tools)

```
django-mastery-course/
├── requirements/
│   ├── base.in
│   ├── base.txt
│   ├── dev.in
│   ├── dev.txt
│   ├── test.in
│   ├── test.txt
│   ├── staging.in
│   ├── staging.txt
│   ├── prod.in
│   └── prod.txt
├── config/
│   └── settings/
│       ├── __init__.py
│       ├── base.py
│       ├── dev.py
│       ├── staging.py
│       └── prod.py
├── .env.example
├── .env                    # ไม่ commit ลง Git (มีอยู่ใน .gitignore)
└── manage.py
```

**`requirements/staging.in`** ตัวอย่าง — staging ควรใกล้เคียง production มาก
ที่สุด แต่อาจเพิ่มเครื่องมือ debug บางส่วนที่ยอมรับความเสี่ยงได้:

```
# requirements/staging.in
-r prod.in

# เพิ่ม logging/debugging tools ที่ยอมรับได้ใน staging แต่ไม่ควรมีใน prod จริง
django-silk
```

### 26.3 เชื่อมโยง requirements กับ Django Settings

การแยก requirements ควรไปคู่กับการแยก Django settings (จะเจาะลึกใน Part 010)
ตัวอย่างการเชื่อมโยงเบื้องต้น:

```python
# config/settings/base.py
import os
from pathlib import Path

BASE_DIR = Path(__file__).resolve().parent.parent.parent

INSTALLED_APPS = [
    "django.contrib.admin",
    "django.contrib.auth",
    "django.contrib.contenttypes",
    "django.contrib.sessions",
    "django.contrib.messages",
    "django.contrib.staticfiles",
    "rest_framework",
]
```

```python
# config/settings/dev.py
from .base import *  # noqa: F401,F403

DEBUG = True
ALLOWED_HOSTS = ["localhost", "127.0.0.1"]

# ต้องติดตั้งจาก requirements/dev.txt เท่านั้น
INSTALLED_APPS += ["debug_toolbar", "django_extensions"]
MIDDLEWARE = ["debug_toolbar.middleware.DebugToolbarMiddleware", *MIDDLEWARE]
INTERNAL_IPS = ["127.0.0.1"]
```

```python
# config/settings/prod.py
from .base import *  # noqa: F401,F403
import os

DEBUG = False
ALLOWED_HOSTS = os.environ["ALLOWED_HOSTS"].split(",")
SECURE_SSL_REDIRECT = True
SESSION_COOKIE_SECURE = True
CSRF_COOKIE_SECURE = True
```

การรันจะระบุ settings module ที่ต้องการผ่าน environment variable:

```bash
# รัน development server
DJANGO_SETTINGS_MODULE=config.settings.dev python manage.py runserver

# รัน production (ผ่าน gunicorn)
DJANGO_SETTINGS_MODULE=config.settings.prod gunicorn config.wsgi:application
```

### 26.4 การจัดการ Environment ด้วย uv (แบบ dependency-groups)

uv รองรับแนวคิด "extras" และ "dependency-groups" ที่ทำให้จัดการหลาย
environment ในไฟล์ `pyproject.toml` เดียวได้:

```toml
[project]
name = "django-mastery-course"
dependencies = [
    "django>=5.1,<5.2",
    "psycopg[binary]>=3.2",
]

[project.optional-dependencies]
prod = [
    "gunicorn>=23.0",
    "whitenoise>=6.8",
    "sentry-sdk>=2.18",
]

[dependency-groups]
dev = ["django-debug-toolbar>=4.4", "ipython>=8.29", "ruff>=0.7"]
test = ["pytest>=8.3", "pytest-django>=4.9", "factory-boy>=3.3"]
```

```bash
# ติดตั้งพร้อม extras (prod)
uv sync --extra prod

# ติดตั้งเฉพาะ dependency-group test (สำหรับ CI)
uv sync --only-group test

# ติดตั้งแบบรวม dev group เข้าไปด้วย (ค่า default ของ uv sync)
uv sync
```

### 26.5 Environment Variables กับ Package ที่ต่างกันตาม Environment

บางครั้งการเลือก package ก็ขึ้นอยู่กับ environment เช่น ใช้ SQLite ตอน dev
แต่ใช้ PostgreSQL ตอน production ไฟล์ `.env` (ที่จะเจาะลึกใน Part 010) ช่วยให้
สลับได้โดยไม่ต้องแก้โค้ด:

```bash
# .env.example (commit ไฟล์นี้ลง Git เป็นตัวอย่าง)
DJANGO_SETTINGS_MODULE=config.settings.dev
DATABASE_URL=sqlite:///db.sqlite3
SECRET_KEY=change-me-in-production
DEBUG=True
```

```bash
# .env (ไม่ commit — อยู่ใน .gitignore)
DJANGO_SETTINGS_MODULE=config.settings.prod
DATABASE_URL=postgres://user:password@localhost:5432/mydb
SECRET_KEY=super-secret-random-key-generated-properly
DEBUG=False
```

---

## ขั้นตอนที่ 27: ติดตั้งจาก Private Index หรือจาก Git Repository โดยตรง

### 27.1 ทำไมต้องใช้ Private Package Index

บริษัทขนาดกลางถึงใหญ่มักมี package ภายในที่ไม่ต้องการเผยแพร่สู่สาธารณะบน
PyPI เช่น shared library ระหว่างทีม, internal SDK, หรือ business logic ที่
เป็นความลับทางการค้า จึงต้องใช้ **Private Package Index** เช่น:

- **AWS CodeArtifact**
- **Google Artifact Registry**
- **Azure Artifacts**
- **JFrog Artifactory**
- **GitLab Package Registry**
- **devpi** (self-hosted, open source)

### 27.2 ติดตั้งจาก Private Index ด้วย pip

```bash
# ระบุ index URL ตรง ๆ ในคำสั่ง (index-url จะแทนที่ PyPI ทั้งหมด)
pip install --index-url https://username:password@pypi.mycompany.com/simple/ internal-package

# ใช้ --extra-index-url เพื่อ "เพิ่ม" index อีกแหล่งควบคู่กับ PyPI หลัก
# (แนะนำมากกว่า --index-url เพราะยังหา public package ได้ปกติ)
pip install --extra-index-url https://pypi.mycompany.com/simple/ internal-package django

# ตั้งค่าถาวรผ่านไฟล์ pip.conf เพื่อไม่ต้องพิมพ์ทุกครั้ง
```

**ไฟล์ `pip.conf`** (Linux/macOS: `~/.pip/pip.conf` หรือ `~/.config/pip/pip.conf`,
Windows: `%APPDATA%\pip\pip.ini`):

```ini
[global]
extra-index-url = https://pypi.mycompany.com/simple/
trusted-host = pypi.mycompany.com
```

หรือตั้งค่าเฉพาะโปรเจกต์ผ่านไฟล์ `requirements/base.txt` เอง (ระบุที่บรรทัดบนสุด):

```
# requirements/base.txt
--extra-index-url https://pypi.mycompany.com/simple/

django==5.1.2
internal-analytics-sdk==2.3.0
```

### 27.3 การจัดการ Credential อย่างปลอดภัย

**ห้าม hardcode username/password ลงในไฟล์ requirements.txt ที่ commit ลง Git
เด็ดขาด** วิธีที่ปลอดภัยกว่าคือใช้ environment variable:

```bash
# ตั้งค่า environment variable (ใน CI/CD secret หรือ .env ที่ไม่ commit)
export PIP_EXTRA_INDEX_URL="https://${PYPI_USER}:${PYPI_TOKEN}@pypi.mycompany.com/simple/"

# pip จะอ่าน PIP_EXTRA_INDEX_URL อัตโนมัติโดยไม่ต้องระบุใน command
pip install internal-package
```

หรือใช้ไฟล์ `.netrc` เพื่อเก็บ credential แยกจาก config หลัก:

```
# ~/.netrc (ต้องตั้ง permission เป็น 600 เท่านั้น: chmod 600 ~/.netrc)
machine pypi.mycompany.com
login myusername
password mytoken
```

### 27.4 ใช้ Poetry กับ Private Index

```toml
# pyproject.toml
[[tool.poetry.source]]
name = "mycompany"
url = "https://pypi.mycompany.com/simple/"
priority = "supplemental"
```

```bash
# ตั้งค่า credential ผ่าน environment variable (ชื่อตัวแปรอิงจากชื่อ source)
export POETRY_HTTP_BASIC_MYCOMPANY_USERNAME=myusername
export POETRY_HTTP_BASIC_MYCOMPANY_PASSWORD=mytoken

poetry add internal-analytics-sdk --source mycompany
```

### 27.5 ใช้ uv กับ Private Index

```toml
# pyproject.toml
[[tool.uv.index]]
name = "mycompany"
url = "https://pypi.mycompany.com/simple/"
```

```bash
# ตั้งค่า credential ผ่าน environment variable
export UV_INDEX_MYCOMPANY_USERNAME=myusername
export UV_INDEX_MYCOMPANY_PASSWORD=mytoken

uv add internal-analytics-sdk
```

### 27.6 การ Pin package จาก Git ให้ Reproducible อย่างสมบูรณ์

ทวนจากขั้นตอนที่ 21 — เมื่อติดตั้งจาก Git repository ควร pin ที่ **commit hash**
เสมอสำหรับ production เพราะ branch/tag อาจถูกแก้ไขทับ (force push) ในภายหลัง
ทำให้ build ไม่ reproducible อีกต่อไป:

```
# requirements/base.txt
# ปลอดภัยที่สุด: pin ที่ commit hash เฉพาะเจาะจง
git+https://github.com/mycompany/internal-lib.git@a1b2c3d4e5f67890abcdef1234567890abcdef12#egg=internal-lib

# เสี่ยงกว่า: pin ที่ tag (แต่ tag ถูกลบ/ย้ายได้ในทางเทคนิค แม้จะผิดธรรมเนียม)
git+https://github.com/mycompany/internal-lib.git@v2.3.0#egg=internal-lib

# อันตรายที่สุด: pin ที่ branch (เปลี่ยนแปลงได้ตลอดเวลา ห้ามใช้ใน production)
git+https://github.com/mycompany/internal-lib.git@main#egg=internal-lib
```

---

## ขั้นตอนที่ 28: การแก้ปัญหา Dependency Conflict

### 28.1 Dependency Conflict คืออะไร

**Dependency Conflict** เกิดขึ้นเมื่อ package สองตัว (หรือมากกว่า) ในโปรเจกต์
เดียวกัน ต้องการ dependency ตัวเดียวกันแต่คนละเวอร์ชันที่ไม่ compatible กัน
ตัวอย่างสถานการณ์จริง:

```
Package A ต้องการ requests>=2.31,<3.0
Package B ต้องการ requests==2.28.0

pip ไม่สามารถติดตั้ง requests เวอร์ชันเดียวที่ตอบโจทย์ทั้งสองได้
```

### 28.2 ตัวอย่าง Error message ที่จะเจอ

```
ERROR: Cannot install package-a==1.0 and package-b==2.0 because these
package versions have conflicting dependencies.

The conflict is caused by:
    package-a 1.0 depends on requests<3.0,>=2.31
    package-b 2.0 depends on requests==2.28.0

To fix this you could try to:
1. loosen the range of package versions you've specified
2. remove package versions to allow pip to attempt to solve the dependency conflict
```

### 28.3 pip check: ตรวจสอบ Conflict ใน Environment ที่มีอยู่

```bash
# ตรวจสอบว่า environment ปัจจุบันมี dependency ขัดแย้งกันหรือไม่
pip check

# ผลลัพธ์เมื่อไม่มีปัญหา
# No broken requirements found.

# ผลลัพธ์เมื่อมีปัญหา
# package-b 2.0 has requirement requests==2.28.0, but you have requests 2.31.0.
```

**ข้อควรรู้**: `pip check` ตรวจสอบเฉพาะ environment ที่ติดตั้งไปแล้วเท่านั้น
ไม่ได้ป้องกันไม่ให้เกิด conflict ตั้งแต่แรก (pip เวอร์ชันเก่าอาจติดตั้งทับกัน
เงียบ ๆ โดยไม่เตือน) ควรรัน `pip check` เป็นขั้นตอนหนึ่งใน CI pipeline เสมอ

### 28.4 ขั้นตอนการวินิจฉัยปัญหาอย่างเป็นระบบ

```bash
# ขั้นที่ 1: ดูว่าใครต้องการ dependency ตัวที่ขัดแย้งกันบ้าง
pip install pipdeptree
pipdeptree --reverse --packages requests

# ผลลัพธ์จะแสดง "ใครเรียกใช้ requests บ้าง" แบบย้อนกลับ
# requests==2.31.0
# ├── package-a==1.0 [requires: requests>=2.31,<3.0]
# └── package-b==2.0 [requires: requests==2.28.0]

# ขั้นที่ 2: ตรวจสอบว่า package-b เวอร์ชันใหม่กว่ารองรับ requests เวอร์ชันใหม่หรือไม่
pip index versions package-b

# ขั้นที่ 3: ลองอัปเดต package-b เป็นเวอร์ชันที่รองรับ requests รุ่นใหม่กว่า
pip install "package-b>=2.1"
```

### 28.5 กลยุทธ์แก้ปัญหา Dependency Conflict

| กลยุทธ์ | เมื่อไหร่ควรใช้ | ตัวอย่าง |
|---|---|---|
| **อัปเกรด package ที่ล็อกเวอร์ชันเก่าเกินไป** | เมื่อมีเวอร์ชันใหม่ที่แก้ปัญหาแล้ว | `pip install "package-b>=2.1"` |
| **ลด constraint ของตัวเอง** | เมื่อคุณ pin เวอร์ชันแน่นเกินความจำเป็น | เปลี่ยน `requests==2.28.0` เป็น `requests>=2.28,<3.0` |
| **ใช้ dependency resolver ของ uv/Poetry** | เมื่อ conflict ซับซ้อนหลายชั้น | resolver ของทั้งคู่ฉลาดกว่า pip แบบดั้งเดิม และให้ error message ชัดเจนกว่า |
| **แยก virtual environment** | เมื่อ package สองตัวขัดแย้งกันจริง ๆ และต้องใช้ทั้งคู่ | รัน script เป็น subprocess คนละ venv |
| **มองหา package ทดแทน** | เมื่อ package หนึ่งเลิกดูแลแล้ว (unmaintained) | เปลี่ยนจาก `psycopg2` เป็น `psycopg` (v3) |

### 28.6 ใช้ uv/Poetry Resolver ที่ฉลาดกว่า pip

pip แบบดั้งเดิม (ก่อนเวอร์ชัน 20.3) ใช้ resolver แบบ "legacy" ที่ติดตั้งทับกัน
โดยไม่ตรวจสอบ conflict อย่างละเอียด ปัจจุบัน pip ใช้ resolver ใหม่ที่ดีขึ้นมาก
แต่ uv และ Poetry มี resolver ที่ครบถ้วนกว่าและ**รายงาน error ได้ชัดเจนกว่า
มาก** เมื่อเจอ conflict:

```bash
# uv จะแสดง dependency chain ที่ขัดแย้งกันแบบละเอียด ระบุ root cause ชัดเจน
uv pip install package-a package-b

# ตัวอย่าง error message ของ uv ที่ชัดเจนกว่า pip มาก:
# × No solution found when resolving dependencies:
# ╰─▶ Because package-b==2.0 depends on requests==2.28.0 and package-a==1.0
#     depends on requests>=2.31, we can conclude that package-a==1.0 and
#     package-b==2.0 are incompatible.
```

### 28.7 การป้องกัน Conflict ตั้งแต่ต้น

1. **ใช้ lock file เสมอ** (pip-tools/`uv.lock`/`poetry.lock`) เพื่อให้ resolver
   ตรวจสอบความเข้ากันได้ล่วงหน้าก่อนที่จะติดตั้งจริง
2. **รัน `pip check` หรือเทียบเท่าใน CI ทุกครั้ง** ก่อน merge โค้ด
3. **อัปเดต dependency สม่ำเสมอ** (เช่นทุกเดือน) แทนที่จะปล่อยค้างนาน ๆ แล้ว
   ต้องอัปเกรดทีเดียวหลายเวอร์ชัน ซึ่งเสี่ยง conflict สูงกว่ามาก
4. **หลีกเลี่ยงการ pin แน่นเกินจำเป็น** สำหรับ library ที่ไม่ใช่ตัวหลักของ
   ระบบ ให้ใช้ range แบบ `~=` แทน `==` เพื่อให้ resolver มีพื้นที่ปรับตัว

---

## ขั้นตอนที่ 29: Security Scanning ของ Dependency

### 29.1 ทำไมต้องสแกนความปลอดภัยของ Dependency

รายงานด้านความปลอดภัยหลายฉบับชี้ตรงกันว่า **ช่องโหว่ส่วนใหญ่ในแอปพลิเคชัน
สมัยใหม่ไม่ได้มาจากโค้ดที่เราเขียนเอง แต่มาจาก third-party dependency** ที่เรา
ติดตั้งมาใช้ Django project ทั่วไปมี dependency (รวม transitive) หลายสิบถึง
หลายร้อยตัว การตรวจสอบด้วยมือทุกตัวเป็นไปไม่ได้ในทางปฏิบัติ จึงต้องใช้
เครื่องมือสแกนอัตโนมัติ

### 29.2 pip-audit: เครื่องมือสแกนอย่างเป็นทางการที่ดูแลโดย PyPA

**pip-audit** พัฒนาโดย Python Packaging Authority (PyPA) ตรวจสอบ dependency
เทียบกับฐานข้อมูลช่องโหว่ **OSV (Open Source Vulnerabilities)** ของ Google

```bash
# ติดตั้ง
pip install pip-audit

# สแกน environment ปัจจุบันที่ activate อยู่
pip-audit

# สแกนจากไฟล์ requirements โดยตรง (ไม่ต้องติดตั้งจริงก่อน)
pip-audit -r requirements/prod.txt

# สแกนและพยายามแก้ไขอัตโนมัติ (อัปเกรด package ที่มีช่องโหว่เป็นเวอร์ชันที่ปลอดภัย)
pip-audit --fix

# ส่งออกผลลัพธ์เป็น JSON (ใช้ต่อใน CI pipeline หรือ dashboard)
pip-audit -r requirements/prod.txt --format json --output audit-report.json

# ระบุ vulnerability ที่รับทราบแล้วและต้องการยกเว้นชั่วคราว
pip-audit -r requirements/prod.txt --ignore-vuln GHSA-xxxx-xxxx-xxxx
```

ตัวอย่างผลลัพธ์เมื่อพบช่องโหว่:

```
Found 2 known vulnerabilities in 1 package
Name    Version ID                  Fix Versions
------- ------- -------------------  ------------
pillow  10.0.0  GHSA-8vj2-vxx3-667w  10.0.1
pillow  10.0.0  PYSEC-2023-175       10.3.0
```

### 29.3 Safety: ทางเลือกเชิงพาณิชย์ที่ใช้ฐานข้อมูล Curated

**Safety** เป็นอีกเครื่องมือยอดนิยม ใช้ฐานข้อมูลช่องโหว่ของตัวเอง
(Safety DB) ที่ทีมงานคัดกรองเพิ่มเติมนอกเหนือจากแหล่งสาธารณะ

```bash
# ติดตั้ง
pip install safety

# สแกน environment ปัจจุบัน
safety check

# สแกนจากไฟล์ requirements
safety check -r requirements/prod.txt

# แสดงผลแบบละเอียดพร้อมคำแนะนำการแก้ไข
safety check -r requirements/prod.txt --full-report

# ส่งออกผลลัพธ์เป็น JSON
safety check -r requirements/prod.txt --json --output safety-report.json

# เวอร์ชันใหม่ของ safety ใช้คำสั่ง scan แทน check
safety scan
```

### 29.4 เปรียบเทียบ pip-audit vs safety

| คุณสมบัติ | pip-audit | safety |
|---|---|---|
| ผู้พัฒนา/ดูแล | PyPA (ทีมงานอย่างเป็นทางการของ Python) | Safety Cybersecurity (บริษัทเอกชน) |
| ฐานข้อมูลช่องโหว่ | OSV (Open Source Vulnerabilities โดย Google) | Safety DB (มีทั้งฟรีและ commercial tier) |
| ราคา | ฟรี 100% (open source) | ฟรีสำหรับใช้พื้นฐาน, มี paid tier สำหรับ database ที่ครบถ้วนกว่า |
| ความเร็วอัปเดตฐานข้อมูล | รวดเร็ว เพราะ sync จาก OSV โดยตรง | รวดเร็วเช่นกัน โดยเฉพาะ tier ที่เสียเงิน |
| แนะนำสำหรับ | โปรเจกต์ทั่วไป, open source, งบจำกัด | ทีมองค์กรที่ต้องการ compliance report ที่ครบถ้วน |

**คำแนะนำ**: ใช้ทั้งสองตัวควบคู่กันใน CI pipeline เพราะแต่ละตัวใช้ฐานข้อมูล
คนละแหล่ง การใช้ทั้งคู่ช่วยเพิ่มโอกาสตรวจพบช่องโหว่ได้ครอบคลุมมากขึ้น

### 29.5 รวม Security Scanning เข้ากับ GitHub Actions (ตัวอย่างเบื้องต้น)

เราจะเรียนเรื่อง CI/CD อย่างละเอียดใน Part 088 แต่ขอยกตัวอย่างเบื้องต้นให้เห็น
ภาพว่าการสแกนความปลอดภัยควรเป็นส่วนหนึ่งของ pipeline อัตโนมัติ:

```yaml
# .github/workflows/security-audit.yml
name: Dependency Security Audit

on:
  push:
    branches: [main]
  schedule:
    - cron: "0 6 * * 1"   # รันทุกวันจันทร์ เวลา 06:00 UTC

jobs:
  audit:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: "3.12"

      - name: Install audit tools
        run: pip install pip-audit safety

      - name: Run pip-audit
        run: pip-audit -r requirements/prod.txt

      - name: Run safety check
        run: safety check -r requirements/prod.txt
```

การตั้ง schedule ให้รันทุกสัปดาห์สำคัญมาก เพราะช่องโหว่ใหม่ถูกค้นพบตลอดเวลา
แม้คุณจะไม่ได้แก้โค้ดเลยก็ตาม dependency เดิมที่เคยปลอดภัยอาจถูกประกาศว่ามี
ช่องโหว่ในภายหลังได้เสมอ

### 29.6 uv audit และแนวทางในอนาคต

ปัจจุบัน uv ยังไม่มีคำสั่ง audit ในตัวโดยตรง แต่สามารถใช้ pip-audit ร่วมกับ
environment ที่ uv สร้างได้ตามปกติ:

```bash
# สแกน environment ที่จัดการโดย uv โดยใช้ pip-audit
uv pip install pip-audit
uv run pip-audit

# หรือ export dependency ออกมาจาก uv.lock แล้วสแกน
uv export --format requirements-txt -o /tmp/requirements-export.txt
pip-audit -r /tmp/requirements-export.txt
```

### 29.7 นโยบายจัดการช่องโหว่ที่พบ

เมื่อพบช่องโหว่ ให้ปฏิบัติตามลำดับความสำคัญนี้:

1. **Critical/High severity ที่มี patch แล้ว**: อัปเดตทันทีภายใน 24-48 ชั่วโมง
2. **Critical/High severity ที่ยังไม่มี patch**: ประเมินว่าโค้ด path ที่มี
   ช่องโหว่ถูกใช้งานจริงหรือไม่ พิจารณา workaround ชั่วคราว หรือเปลี่ยน
   package ทดแทน
3. **Medium/Low severity**: วางแผนอัปเดตในรอบ maintenance ปกติ (เช่น sprint
   ถัดไป)
4. **บันทึกทุกครั้งที่ตัดสินใจ "ยอมรับความเสี่ยง" (accept risk)** พร้อมเหตุผล
   และวันที่จะทบทวนใหม่ ไม่ใช่แค่เพิกเฉยไปเฉย ๆ

---

## ขั้นตอนที่ 30: สรุป Best Practice Checklist และแบบฝึกหัด

### 30.1 สรุปสิ่งที่ได้เรียนรู้ใน Part นี้

- ✅ ใช้ `pip` เจาะลึก: install/uninstall/list/show, version specifiers ตาม
  PEP 440, ติดตั้งจาก local path (`-e`) และจาก Git repository
- ✅ จัดโครงสร้าง `requirements/base.txt`, `dev.txt`, `test.txt`, `prod.txt`
  แยกตาม environment อย่างมืออาชีพ พร้อม pin เวอร์ชันอย่างถูกต้อง
- ✅ ใช้ `pip-tools` (`pip-compile`, `pip-sync`) เพื่อแยกไฟล์ `.in`
  (สิ่งที่ต้องการ) ออกจาก `.txt` (สิ่งที่ล็อกแล้ว) และรักษา reproducible build
- ✅ ติดตั้งและใช้งาน `uv` เครื่องมือยุคใหม่จาก Astral ที่เร็วกว่า pip
  10-100 เท่า ทั้งโหมด pip-compatible และโหมด project (`uv add`, `uv sync`,
  `uv.lock`)
- ✅ ใช้ `Poetry` และ `pyproject.toml` จัดการ dependency + packaging ในตัวเดียว
  พร้อมเข้าใจความแตกต่างของ `^` และ `~` ใน version constraint
- ✅ จัดการหลาย environment (dev/test/staging/production) ต่อโปรเจกต์เดียว
  อย่างเป็นระบบ เชื่อมโยงกับ Django settings และ environment variables
- ✅ ติดตั้ง package จาก private index และจาก Git repository พร้อมจัดการ
  credential อย่างปลอดภัย
- ✅ วินิจฉัยและแก้ปัญหา Dependency Conflict ด้วย `pip check`, `pipdeptree`
  และ resolver ที่ดีกว่าของ uv/Poetry
- ✅ สแกนช่องโหว่ความปลอดภัยของ dependency ด้วย `pip-audit` และ `safety`
  พร้อมรวมเข้ากับ CI pipeline

### 30.2 Best Practice Checklist สำหรับจัดการ Dependency มืออาชีพ

- [ ] ไม่เคยติดตั้ง package แบบ global — ใช้ virtual environment เสมอ
- [ ] แยก `requirements/` เป็นอย่างน้อย `base`, `dev`, `prod` (เพิ่ม `test`,
      `staging` ตามความจำเป็น)
- [ ] Production ใช้ exact pin (`==`) ทุก package รวม transitive dependency
- [ ] มีไฟล์ lock (`pip-tools` output, `uv.lock`, หรือ `poetry.lock`) และ
      commit ลง Git เสมอ ทั้งไฟล์ `.in`/`pyproject.toml` และไฟล์ lock
- [ ] ไม่ hardcode credential ของ private index ลงในไฟล์ที่ commit ลง Git
- [ ] รัน `pip check` (หรือเทียบเท่า) ใน CI pipeline ก่อน merge ทุกครั้ง
- [ ] รัน security scan (`pip-audit`/`safety`) อัตโนมัติทุกสัปดาห์ ไม่ใช่
      แค่ตอน deploy
- [ ] Package จาก Git ที่ใช้ใน production ต้อง pin ที่ commit hash ไม่ใช่
      branch
- [ ] อัปเดต dependency สม่ำเสมอ (เช่นรายเดือน) แทนปล่อยค้างจนต้องอัปเกรด
      ครั้งใหญ่
- [ ] ทดลองใช้ `uv` เพื่อลดเวลา build ของ CI/CD pipeline

### 30.3 แบบฝึกหัดท้ายบท

**แบบฝึกหัดที่ 1**: สร้างโฟลเดอร์ `requirements/` ในโปรเจกต์
`django-mastery-course` จาก Part 001 พร้อมไฟล์ `base.in`, `dev.in`, `test.in`,
`prod.in` แล้วใช้ `pip-compile` สร้างไฟล์ `.txt` ที่สอดคล้องกันทั้งหมด
จากนั้นทดสอบ `pip-sync requirements/dev.txt` ว่าใช้งานได้จริง

**แบบฝึกหัดที่ 2**: ติดตั้ง `uv` แล้วสร้างโปรเจกต์ทดสอบใหม่ด้วย `uv init`
เพิ่ม dependency `django` และ `djangorestframework` ด้วย `uv add` จากนั้น
วัดเวลาที่ใช้เปรียบเทียบกับการสร้าง venv + ติดตั้งด้วย `pip install` แบบเดิม
บันทึกผลเป็นตัวเลขวินาทีทั้งสองวิธี

**แบบฝึกหัดที่ 3**: จำลองสถานการณ์ dependency conflict โดยตั้งใจติดตั้ง
สอง package ที่ต้องการ `requests` คนละเวอร์ชันกัน (ค้นหาตัวอย่างจาก PyPI
ที่ยังมีปัญหานี้อยู่จริง หรือสร้าง local package สองตัวขึ้นมาเองเพื่อจำลอง)
แล้วใช้ `pipdeptree --reverse` วินิจฉัยปัญหา พร้อมเขียนอธิบายว่าคุณจะแก้
ปัญหานี้อย่างไรในสถานการณ์จริง

**แบบฝึกหัดที่ 4 (ขั้นสูง)**: รัน `pip-audit -r requirements/prod.txt` กับ
ไฟล์ requirements ที่มี package เวอร์ชันเก่า (เช่นลองใส่ `django==3.2.0`
หรือ `pillow==9.0.0` ที่ทราบกันดีว่ามีช่องโหว่ในอดีต) บันทึกผลลัพธ์ที่ได้
และเขียนแผนการแก้ไข (remediation plan) ว่าจะอัปเดต package ไหนเป็นเวอร์ชัน
อะไรบ้าง

### 30.4 คำถามที่พบบ่อย (FAQ)

**Q: ควรใช้ pip, uv หรือ Poetry ดีในการเรียนหลักสูตรนี้?**
A: หลักสูตรนี้จะใช้ `pip` + `requirements/*.txt` เป็นมาตรฐานหลักในทุก Part
ถัดไป เพราะเข้าใจง่ายและใช้ได้กับทุก deployment platform โดยไม่มีข้อจำกัด
แต่แนะนำให้คุณฝึกใช้ `uv` ควบคู่ไปด้วยเป็นทางเลือก เพราะเป็นทิศทางที่
อุตสาหกรรมกำลังมุ่งไปอย่างชัดเจนในปี 2025-2026 ส่วน Poetry เหมาะมากถ้า
คุณวางแผนจะพัฒนา library ของตัวเองเพื่อ publish ขึ้น PyPI ในอนาคต

**Q: จำเป็นต้องใช้ pip-tools ถ้าจะใช้ uv อยู่แล้วหรือไม่?**
A: ไม่จำเป็น เพราะ uv มี `uv pip compile` ที่ทำหน้าที่เหมือน `pip-compile`
ในตัวอยู่แล้ว และเร็วกว่ามาก อย่างไรก็ตาม การเข้าใจแนวคิดของ pip-tools
ยังสำคัญ เพราะหลายโปรเจกต์ในโลกจริง (โดยเฉพาะโปรเจกต์เก่า) ยังใช้ pip-tools
อยู่ และแนวคิด `.in` vs `.txt` เป็นพื้นฐานที่ uv/Poetry ก็ใช้หลักการเดียวกัน

**Q: ทำไม requirements.txt ของ production ต้อง exact pin ทุกตัว แม้แต่
transitive dependency?**
A: เพราะถ้าไม่ pin transitive dependency ให้ครบ การ `pip install` ในวันที่
ต่างกันอาจได้ package เวอร์ชันต่างกัน (เนื่องจากมี package ใหม่ถูก publish
ขึ้น PyPI ระหว่างนั้น) ทำให้ production build ของวันนี้กับพรุ่งนี้ไม่เหมือนกัน
ซึ่งเป็นสาเหตุของบั๊กประเภท "งานบนเครื่องฉันรันได้ แต่บน server ไม่ได้"
ที่พบบ่อยมาก การมีไฟล์ lock ที่ pin ทุกอย่างคือทางแก้ที่ยั่งยืนที่สุด

**Q: ควร commit ไฟล์ lock (uv.lock, poetry.lock, requirements.txt ที่
compile แล้ว) ลง Git หรือไม่?**
A: **ต้อง commit เสมอ** ไม่ว่าจะใช้เครื่องมือไหน ไฟล์ lock คือสิ่งที่ทำให้
ทุกคนในทีม (และ CI/CD) ได้ dependency ชุดเดียวกันเป๊ะ การไม่ commit ไฟล์นี้
เท่ากับสูญเสียประโยชน์หลักของการมี lock file ไปเลย

**Q: pip-audit กับ safety ต่างกันแค่ไหน ต้องใช้ทั้งคู่จริงหรือ?**
A: ทั้งสองใช้ฐานข้อมูลช่องโหว่คนละแหล่ง (`pip-audit` ใช้ OSV ของ Google,
`safety` ใช้ฐานข้อมูลของตัวเอง) ในทางปฏิบัติ การใช้ทั้งคู่ไม่ได้เสียเวลา
มากนักและช่วยเพิ่มโอกาสตรวจพบช่องโหว่ได้มากขึ้น สำหรับโปรเจกต์ขนาดเล็ก
การใช้ `pip-audit` เพียงตัวเดียวก็เพียงพอ เพราะเป็นเครื่องมือฟรีที่ดูแล
โดยทีมงานอย่างเป็นทางการของ Python

---

## เตรียมตัวสำหรับ Part ถัดไป

**Part 004: สร้างโปรเจกต์ Django แรกและทำความเข้าใจโครงสร้าง** จะพาคุณใช้
`django-admin startproject` สร้างโปรเจกต์ Django จริงเป็นครั้งแรก โดยนำ
ความรู้เรื่อง virtual environment และ requirements ที่เพิ่งเรียนใน Part นี้
มาใช้ตั้งแต่ก้าวแรก คุณจะได้เจาะลึกไฟล์ทุกไฟล์ที่ Django สร้างให้อัตโนมัติ
(`manage.py`, `settings.py`, `urls.py`, `wsgi.py`, `asgi.py`) เข้าใจว่าแต่ละ
ไฟล์ทำหน้าที่อะไร และเริ่มวางโครงสร้างโปรเจกต์ที่จะใช้ต่อเนื่องไปตลอด
หลักสูตรที่เหลือ

เตรียม virtual environment และไฟล์ `requirements/` ที่คุณสร้างใน Part นี้
ไว้ให้พร้อม เพราะเราจะเริ่มติดตั้ง Django เวอร์ชันที่ pin ไว้อย่างเป็นทางการ
และสร้างโปรเจกต์จริงตัวแรกกันในบทถัดไป!
