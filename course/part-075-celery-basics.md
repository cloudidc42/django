# Part 075: Celery เบื้องต้น: Background Tasks

> **ขั้นตอนที่ 741-750 ของหลักสูตร** | Phase 9: Async, Celery และ Channels
>
> Part 071 ขั้นตอนที่ 708 เกริ่นไว้แล้วว่า Streaming และ `.iterator()` ช่วยแก้ปัญหา
> **memory** ของงานหนักได้ แต่ไม่ได้แก้ปัญหา **เวลารอของผู้ใช้** — Part นี้คือ Part
> ที่เจาะลึกเต็มรูปแบบตามที่สัญญาไว้ เราจะติดตั้ง **Celery** และเชื่อมกับ **Redis**
> ที่ตั้งค่าไว้แล้วอย่างละเอียดตั้งแต่ Part 069 ให้ทำหน้าที่เป็น **Message Broker**
> เขียน Task แรกด้วย `@shared_task` เรียกใช้ผ่าน `.delay()`, รัน Celery Worker แยก
> process ออกจาก Django, ตั้งค่า Result Backend เพื่อติดตามสถานะงาน, ทำความเข้าใจ
> กลไก Retry แบบ exponential backoff, เจาะลึกว่าทำไม Celery ถึงเปลี่ยน default
> serializer จาก pickle เป็น JSON เพื่อความปลอดภัย, เรียนรู้กฎเหล็กข้อสำคัญที่สุด
> ข้อหนึ่งของ Celery คือ **ห้ามส่ง Model instance เข้า Task ตรง ๆ**, และปิดท้าย
> ด้วยการจัดการ Error อย่างเป็นระบบ เมื่อจบ Part นี้ คุณจะแปลงระบบส่งอีเมลทั้งหมด
> ของ Blog (Password Reset, การแจ้งเตือนคอมเมนต์ใหม่) ให้กลายเป็น Background Task
> ที่ทนทาน มี retry อัตโนมัติ และไม่บล็อก request cycle อีกต่อไป

---

## สารบัญของ Part นี้

- ขั้นตอนที่ 741: ทำไมต้องมี Background Task — ปัญหาของงานที่บล็อก Request Cycle
- ขั้นตอนที่ 742: ติดตั้ง Celery และตั้งค่า Broker ด้วย Redis (ต่อยอดจาก Part 069)
- ขั้นตอนที่ 743: เขียน Task แรกด้วย `@shared_task` เรียกใช้ผ่าน `.delay()`
- ขั้นตอนที่ 744: การรัน Celery Worker — `celery -A config worker`, `--concurrency`
- ขั้นตอนที่ 745: Result Backend — เก็บผลลัพธ์ของ Task, `task.status`, `task.get()`
- ขั้นตอนที่ 746: Task Retry — `autoretry_for`, `max_retries`, `retry_backoff`
- ขั้นตอนที่ 747: Task Serialization — JSON เทียบกับ Pickle และทำไม Celery เปลี่ยน default
- ขั้นตอนที่ 748: ส่ง Argument ที่ซับซ้อนเข้า Task อย่างถูกวิธี — ส่ง Primary Key ไม่ใช่ Model
- ขั้นตอนที่ 749: จัดการ Error ใน Task — `on_failure` Hook และ Logging
- ขั้นตอนที่ 750: สรุปและแบบฝึกหัด — แปลงระบบอีเมลของ Blog เป็น Celery Task

---

## ขั้นตอนที่ 741: ทำไมต้องมี Background Task — ปัญหาของงานที่บล็อก Request Cycle

### 741.1 ทวนวงจร Request-Response ปกติของ Django

ทวนจาก Part 001 ขั้นตอนที่ 3: เมื่อ browser ส่ง request มา Django จะรัน View หนึ่ง
ฟังก์ชันจาก **ต้นจนจบ** แล้วค่อยส่ง `HttpResponse` กลับไป — ทุกบรรทัดโค้ดภายใน View
ทำงานแบบ **synchronous** (เรียงลำดับ ทีละบรรทัด) และ **request จะไม่ได้รับคำตอบ
จนกว่า View จะ return** ไม่ว่า View นั้นจะใช้เวลาสั้นแค่ 5 มิลลิวินาที หรือยาวถึง
30 วินาทีก็ตาม

```
Browser ──> View เริ่มทำงาน ──> ... โค้ดทำงานไปเรื่อย ๆ ... ──> View return ──> Browser ได้ Response
            │                                                  │
            └──────────────── ผู้ใช้เห็นหน้าเว็บ "กำลังโหลด" ตลอดช่วงนี้ ────────────────┘
```

### 741.2 ตัวอย่างปัญหาจริง: การส่งอีเมลใน View

สถานการณ์คลาสสิกที่สุดที่ทุกทีมพัฒนาเว็บต้องเจอ: ผู้ใช้กด "สมัครสมาชิก" แล้วระบบ
ต้องส่งอีเมลยืนยันตัวตน:

```python
# accounts/views.py — เวอร์ชันที่ "ทำงานได้" แต่มีปัญหาซ่อนอยู่
from django.core.mail import send_mail
from django.shortcuts import redirect
from django.conf import settings
from .forms import SignUpForm


def signup_view(request):
    if request.method == 'POST':
        form = SignUpForm(request.POST)
        if form.is_valid():
            user = form.save()

            # ปัญหาอยู่ตรงนี้: send_mail() เชื่อมต่อ SMTP server จริง ผ่าน network
            # ใช้เวลาไม่แน่นอน อาจ 200ms ในวันปกติ หรือ 8-15 วินาทีถ้า SMTP server
            # ของผู้ให้บริการช้าหรือ network มีปัญหาชั่วคราว
            send_mail(
                subject='ยืนยันการสมัครสมาชิก',
                message=f'สวัสดีคุณ {user.username} กรุณายืนยันอีเมลของคุณ...',
                from_email=settings.DEFAULT_FROM_EMAIL,
                recipient_list=[user.email],
            )

            return redirect('accounts:signup_success')
    else:
        form = SignUpForm()
    return render(request, 'accounts/signup.html', {'form': form})
```

**ผู้ใช้ต้องรอ SMTP server ตอบกลับก่อน ถึงจะเห็นหน้า "สมัครสำเร็จ"** ทั้งที่การส่ง
อีเมลไม่ใช่สิ่งที่ผู้ใช้ต้อง**รอ**เห็นผลจริง ๆ เลย — ผู้ใช้แค่ต้องการรู้ว่าสมัคร
สำเร็จแล้วเท่านั้น อีเมลจะไปถึงกล่องจดหมายช้าหรือเร็วอีก 1-2 วินาทีแทบไม่มีผลต่อ
ประสบการณ์การใช้งานเลย

### 741.3 Timeline เปรียบเทียบ: บล็อก vs ไม่บล็อก

```
แบบบล็อก (ปัจจุบัน):
Request ──> save user (20ms) ──> send_mail() รอ SMTP (200ms - 15,000ms) ──> Response
            └──────────────────── ผู้ใช้รอทั้งหมดนี้ ─────────────────────┘

แบบ Background Task (เป้าหมายของ Part นี้):
Request ──> save user (20ms) ──> ส่งงานเข้าคิว (2ms) ──> Response ทันที (~22ms)
                                        │
                                        └──> Celery Worker (process แยกต่างหาก)
                                             หยิบงานจากคิวไปส่งอีเมลจริง
                                             (ไม่กระทบเวลาตอบสนองของ Request เลย)
```

ผู้ใช้ได้รับ response ใน **22 มิลลิวินาที** แทนที่จะรอสูงสุดถึง **15 วินาที** — ต่างกัน
เกือบ 700 เท่าในกรณีเลวร้ายที่สุด และที่สำคัญกว่านั้นคือ **เวลาตอบสนองไม่ผันผวนตาม
สุขภาพของ SMTP server ภายนอกอีกต่อไป**

### 741.4 งานประเภทไหนบ้างที่ควรเป็น Background Task

| ลักษณะงาน | ตัวอย่าง | เหมาะเป็น Background Task ไหม |
|---|---|---|
| เรียก service ภายนอกผ่าน network | ส่งอีเมล, ส่ง SMS, เรียก Payment Gateway, เรียก Push Notification | ✅ เหมาะมาก — เวลาตอบสนองของ service ภายนอกควบคุมไม่ได้ |
| ประมวลผลข้อมูลหนัก/ใช้เวลานาน | สร้างรายงาน PDF, resize รูปภาพจำนวนมาก, export CSV ขนาดใหญ่ (ทวนจาก Part 071 ขั้นตอนที่ 705-707) | ✅ เหมาะมาก — งานใช้ CPU/IO นาน ไม่ควรผูกกับอายุของ request |
| งานที่ต้องรันตามตารางเวลา | ลบข้อมูลเก่าทุกเที่ยงคืน, สรุปยอดขายรายวัน, sync ยอด view จาก Redis กลับ DB (ทวนจาก Part 069 ขั้นตอนที่ 682.3) | ✅ เหมาะ — แต่ต้องใช้ Celery Beat (เรียนเต็มใน Part 076) |
| งานที่ผู้ใช้ต้อง**เห็นผลทันที** เพื่อตัดสินใจขั้นต่อไป | ตรวจสอบ username ซ้ำตอนสมัคร, validate ข้อมูลฟอร์ม, login | ❌ ไม่เหมาะ — ผู้ใช้ต้องรอผลลัพธ์ก่อนไปขั้นตอนถัดไปอยู่แล้ว |
| Query ฐานข้อมูลธรรมดาที่เร็วอยู่แล้ว | ดึงรายการ Post มาแสดง, บันทึกฟอร์มเดี่ยว ๆ | ❌ ไม่จำเป็น — เพิ่มความซับซ้อนโดยไม่ได้ประโยชน์ |

**หลักการตัดสินใจ**: ถามตัวเองว่า *"ผู้ใช้จำเป็นต้องเห็นผลลัพธ์ของงานนี้ก่อนถึงจะ
ไปขั้นตอนถัดไปได้หรือไม่"* ถ้าคำตอบคือ **ไม่จำเป็น** และงานนั้นใช้เวลานานหรือขึ้นกับ
ปัจจัยภายนอกที่ควบคุมไม่ได้ นั่นคือผู้สมัครที่ดีสำหรับ Background Task

### 741.5 ทำไมไม่ใช้ `threading.Thread` ธรรมดาแทน Celery

มือใหม่หลายคนคิดจะแก้ปัญหานี้ง่าย ๆ ด้วย Python thread ในตัว:

```python
# วิธีที่ "ดูเหมือนจะได้ผล" แต่มีปัญหาซ่อนอยู่มากในระดับ production
import threading

def signup_view(request):
    # ...
    threading.Thread(target=send_mail, args=(...)).start()
    return redirect('accounts:signup_success')  # return ทันทีไม่รอ thread
```

| ประเด็น | `threading.Thread` ธรรมดา | Celery + Redis Broker |
|---|---|---|
| ถ้า worker process/thread ล่มระหว่างทำงาน | งานหายไปเลย ไม่มีทางรู้ ไม่มีทาง retry | Broker เก็บงานไว้จนกว่าจะมี worker มารับและยืนยันว่าทำสำเร็จ (`acknowledgment`) |
| ถ้า deploy โค้ดใหม่ / restart server ขณะ thread กำลังทำงาน | Thread ถูกฆ่ากลางคัน งานค้างไม่สมบูรณ์ | Worker แยก process ต่างหาก จัดการ graceful shutdown ได้ (รอ task ปัจจุบันจบก่อน) |
| กระจายงานไปทำที่เครื่องอื่นได้ไหม | ❌ ทำได้แค่ในเครื่อง/process เดียวกับ Django | ✅ Worker รันที่เครื่องไหนก็ได้ ตราบใดที่เชื่อมต่อ Broker เดียวกัน — scale แยกจาก web server ได้อิสระ |
| จำกัดจำนวนงานที่ทำพร้อมกัน (concurrency control) | ทำเองทั้งหมด เสี่ยงสร้าง thread ไม่จำกัดจนเครื่องล่ม | มีในตัว (`--concurrency`, ขั้นตอนที่ 744) |
| Retry อัตโนมัติเมื่อล้มเหลว | ต้องเขียนเอง | มีในตัว (ขั้นตอนที่ 746) |
| ติดตามสถานะงาน (สำเร็จ/ล้มเหลว/กำลังทำ) | ต้องเขียนระบบเก็บสถานะเอง | มี Result Backend ในตัว (ขั้นตอนที่ 745) |
| ผลกระทบจาก Python GIL ต่องานที่ใช้ CPU หนัก | Thread ทั้งหมดแย่ง GIL กัน ไม่ได้ประโยชน์จาก multi-core จริง | Worker แบบ `prefork` (ค่าเริ่มต้น) ใช้หลาย **process** จริง ไม่ติด GIL |

**สรุป**: `threading.Thread` อาจพอใช้ได้สำหรับ script ทดลองเล่นเล็ก ๆ แต่สำหรับระบบ
production ที่ต้องทนทานต่อความล้มเหลว ต้องปรับสเกลได้ และต้องติดตามสถานะงานได้
**Celery คือมาตรฐานอุตสาหกรรม** ที่แก้ปัญหาทั้งหมดนี้ให้แล้ว

### 741.6 ตารางสรุปขั้นตอนที่ 741

| หัวข้อ | สรุป |
|---|---|
| ปัญหาหลัก | View ทำงานแบบ synchronous — ผู้ใช้ต้องรอทุกบรรทัดโค้ดจบก่อนได้ response |
| ตัวอย่างคลาสสิก | ส่งอีเมลใน View ทำให้เวลาตอบสนองผันผวนตาม SMTP server ภายนอก |
| งานที่เหมาะเป็น Background Task | เรียก service ภายนอก, ประมวลผลหนัก, งานตามตารางเวลา |
| งานที่ไม่เหมาะ | งานที่ผู้ใช้ต้องเห็นผลก่อนไปขั้นตอนถัดไป |
| ทำไมไม่ใช้ `threading.Thread` | ไม่ทนต่อ crash, ไม่มี retry, กระจายงานข้ามเครื่องไม่ได้, ติด GIL |
| ทางออก | Celery — Distributed Task Queue มาตรฐานของ Python/Django |

---

## ขั้นตอนที่ 742: ติดตั้ง Celery และตั้งค่า Broker ด้วย Redis

### 742.1 สถาปัตยกรรมของ Celery โดยรวม

ก่อนติดตั้งอะไร ต้องเข้าใจภาพรวมของส่วนประกอบทั้งหมดก่อน:

```
┌─────────────┐   1. .delay()    ┌─────────────┐   3. ดึงงานไปทำ   ┌──────────────┐
│ Django View │ ───────────────> │   Broker    │ <──────────────── │ Celery Worker │
│ (Producer)  │   ส่งงานเข้าคิว   │   (Redis)   │                    │  (Process     │
└─────────────┘                  └─────────────┘                    │   แยกต่างหาก) │
       │                                 ▲                          └──────┬───────┘
       │ 2. คืน AsyncResult (task_id)     │ 4. เขียนผลลัพธ์กลับ                 │ ประมวลผล task จริง
       ▼                                 │                                  │
┌─────────────┐                  ┌──────┴──────┐                          │
│  ตรวจสอบผล   │ <─────────────── │Result Backend│ <─────────────────────────┘
│(task.status) │   5. อ่านผล      │   (Redis)    │
└─────────────┘                  └─────────────┘
```

| ส่วนประกอบ | หน้าที่ | ในหลักสูตรนี้ใช้ |
|---|---|---|
| **Broker (Message Broker)** | คิวกลางที่รับงานจาก Django แล้วรอให้ Worker มาหยิบไปทำ | Redis (ต่อยอดจาก Part 069 ที่ตั้งไว้แล้ว) |
| **Worker** | Process แยกต่างหากจาก Django ที่คอยหยิบงานจาก Broker มาประมวลผลจริง | `celery -A config worker` (ขั้นตอนที่ 744) |
| **Result Backend** | ที่เก็บสถานะและผลลัพธ์ของแต่ละ task | Redis เช่นกัน (คนละ DB index จาก Broker, ขั้นตอนที่ 745) |
| **Task** | ฟังก์ชัน Python ที่ถูกลงทะเบียนให้ Celery รู้จักว่าเรียกผ่านคิวได้ | `@shared_task` (ขั้นตอนที่ 743) |

**จุดสำคัญที่ต้องเข้าใจตั้งแต่ต้น**: Celery Worker คือ **process ของ Python ที่แยก
ออกจาก process ของ Django web server โดยสิ้นเชิง** ต้องสั่งรันแยกต่างหาก (ขั้นตอน
ที่ 744) และเมื่อ deploy จริงมักรันคนละเครื่อง/คนละ container กับ web server ด้วยซ้ำ
— นี่คือสิ่งที่ทำให้ Celery ปรับสเกลได้อิสระจากเว็บเซิร์ฟเวอร์

### 742.2 ทำไมใช้ Redis เป็น Broker แทนที่จะติดตั้งใหม่

Celery รองรับ Broker หลายแบบ ที่นิยมที่สุดคือ **Redis** และ **RabbitMQ** (เจาะลึก
ความแตกต่างเต็มรูปแบบใน Part 077) แต่ Part นี้เลือก Redis เพราะ **โปรเจกต์ของเรามี
Redis พร้อมใช้งานอยู่แล้วตั้งแต่ Part 069** — ไม่ต้องติดตั้ง service ใหม่เพิ่ม
ไม่ต้องเรียนรู้ระบบใหม่ แค่ต่อยอดจากสิ่งที่มีอยู่:

```bash
# ตรวจสอบว่า Redis ยังทำงานอยู่ (ทวนจาก Part 069 ขั้นตอนที่ 681.2)
redis-cli ping
# ผลลัพธ์ที่คาดหวัง: PONG
```

ถ้ายังไม่มี Redis ในเครื่อง ให้ย้อนกลับไปติดตั้งตาม Part 069 ขั้นตอนที่ 681.2 ก่อน
(ผ่าน `apt install redis-server`, `brew install redis`, หรือ Docker)

### 742.3 ติดตั้ง Celery ผ่าน pip

```bash
# activate venv ก่อนเสมอ (ทวนจาก Part 001 ขั้นตอนที่ 8)
pip install celery
pip freeze | grep -i celery >> requirements.txt
```

**ข้อสังเกตสำคัญ**: เราไม่ต้องติดตั้ง package เพิ่มสำหรับ "Celery คุยกับ Redis"
เพราะ `redis` และ `hiredis` ถูกติดตั้งไว้แล้วตั้งแต่ Part 069 ขั้นตอนที่ 681.3 —
Celery ใช้ library `redis` ตัวเดียวกันนี้ในการเชื่อมต่อ Broker ผ่าน transport
`redis://` เบื้องหลัง (ผ่าน library ตัวกลางชื่อ `kombu` ที่ Celery ใช้จัดการ
transport หลายชนิด)

ตรวจสอบเวอร์ชัน:

```bash
celery --version
# ผลลัพธ์ที่คาดหวัง เช่น: celery 5.4.0
```

### 742.4 สร้างไฟล์ `config/celery.py`

โครงสร้างมาตรฐานของ Celery ในโปรเจกต์ Django คือสร้างไฟล์ `celery.py` ไว้ **ข้าง ๆ**
`settings.py` ในโฟลเดอร์ config ของโปรเจกต์ (โฟลเดอร์เดียวกับที่มี `settings.py`,
`urls.py`, `wsgi.py`):

```python
# config/celery.py
import os

from celery import Celery

# บอก Celery ว่า Django settings module อยู่ที่ไหน ต้องตั้งก่อนสร้าง Celery app
# เพื่อให้ Celery รู้จัก Django apps และเชื่อมต่อฐานข้อมูลได้ถูกต้องเมื่อ Worker เริ่มทำงาน
os.environ.setdefault('DJANGO_SETTINGS_MODULE', 'config.settings')

app = Celery('config')

# อ่านค่า config ทั้งหมดที่ขึ้นต้นด้วย CELERY_ จาก Django settings.py โดยอัตโนมัติ
# namespace='CELERY' หมายถึง: ตัวแปรใน settings.py ต้องขึ้นต้นด้วย CELERY_ เท่านั้น
# (เช่น CELERY_BROKER_URL ไม่ใช่ BROKER_URL เฉย ๆ) เพื่อไม่ให้ปนกับ setting อื่นของ Django
app.config_from_object('django.conf:settings', namespace='CELERY')

# ค้นหาไฟล์ tasks.py ในทุก app ที่อยู่ใน INSTALLED_APPS โดยอัตโนมัติ
# ไม่ต้อง import task ทีละไฟล์เอง — แค่สร้างไฟล์ชื่อ tasks.py ในแอปไหน Celery ก็เจอเอง
app.autodiscover_tasks()


@app.task(bind=True, ignore_result=True)
def debug_task(self):
    """Task ทดสอบง่าย ๆ สำหรับตรวจสอบว่า Celery ทำงานถูกต้อง (ใช้ในขั้นตอนที่ 744)"""
    print(f'Request: {self.request!r}')
```

### 742.5 เชื่อม Celery App เข้ากับ Django ผ่าน `config/__init__.py`

ขั้นตอนที่มือใหม่มักลืม แต่ **สำคัญมาก** — ต้อง import `celery_app` ในไฟล์
`__init__.py` ของโปรเจกต์ เพื่อให้ Celery app ถูกโหลดขึ้นมาทุกครั้งที่ Django
เริ่มทำงาน (ทั้งตอนรัน `runserver` และตอนรัน `celery worker`):

```python
# config/__init__.py
from .celery import app as celery_app

__all__ = ('celery_app',)
```

**ทำไมต้องทำแบบนี้**: `@shared_task` (ที่จะเรียนในขั้นตอนที่ 743) ต้องรู้ว่ามี
Celery app instance ไหนที่ "ผูก" อยู่ด้วย ถ้าไม่ import `celery_app` ไว้ใน
`__init__.py` การเรียก `.delay()` จากที่อื่นในโปรเจกต์อาจหาไม่เจอว่า Celery app
ตัวไหนควรรับงานนี้ไป

### 742.6 ตั้งค่า `CELERY_*` ใน `settings.py`

```python
# config/settings.py
# ทวนจาก Part 069 ขั้นตอนที่ 681.5 — ตัวแปรนี้มีอยู่แล้วในโปรเจกต์
REDIS_HOST = os.environ.get('REDIS_HOST', '127.0.0.1')
REDIS_PORT = os.environ.get('REDIS_PORT', '6379')

# ── Celery Configuration ──────────────────────────────────────────
# ใช้ Redis DB index 4 สำหรับ Broker — แยกจาก DB 1 (page cache), DB 2 (throttle),
# DB 3 (sessions) ที่ใช้ไปแล้วใน Part 069 ขั้นตอนที่ 681.5 เพื่อไม่ให้ FLUSHDB
# ของ cache กระทบคิวงานที่ยังรอ Worker อยู่ โดยไม่ตั้งใจ
CELERY_BROKER_URL = f'redis://{REDIS_HOST}:{REDIS_PORT}/4'

# Result Backend ใช้ DB index 5 แยกออกไปอีก (รายละเอียดเหตุผลในขั้นตอนที่ 745)
CELERY_RESULT_BACKEND = f'redis://{REDIS_HOST}:{REDIS_PORT}/5'

# บังคับใช้ JSON แทน pickle ทั้งการส่งงานเข้าคิวและการเก็บผลลัพธ์
# (เหตุผลเชิงความปลอดภัยแบบละเอียดอยู่ในขั้นตอนที่ 747)
CELERY_ACCEPT_CONTENT = ['json']
CELERY_TASK_SERIALIZER = 'json'
CELERY_RESULT_SERIALIZER = 'json'

# ให้ Celery ใช้ timezone เดียวกับ Django เสมอ ป้องกันความสับสนเรื่องเวลา
# โดยเฉพาะเมื่อถึง Celery Beat (periodic task) ใน Part 076
CELERY_TIMEZONE = TIME_ZONE
CELERY_ENABLE_UTC = True

# บันทึกสถานะ 'STARTED' ทันทีที่ Worker เริ่มทำงาน (ไม่ใช่แค่ PENDING/SUCCESS/FAILURE)
# มีประโยชน์มากตอน debug ว่า task ค้างอยู่ที่ขั้นไหน
CELERY_TASK_TRACK_STARTED = True

# ผลลัพธ์ใน Result Backend จะถูกลบทิ้งอัตโนมัติหลังผ่านไป 1 ชั่วโมง
# ป้องกัน Redis บวมด้วยผลลัพธ์เก่าที่ไม่มีใครมาอ่านแล้ว
CELERY_RESULT_EXPIRES = 3600

# จำกัดเวลาสูงสุดที่ 1 task รันได้ (วินาที) — ป้องกัน task ที่ค้างตลอดกาลจาก
# การครอง worker process ไปตลอดกาล (soft limit ยิง exception ให้ task จัดการเอง
# ก่อน hard limit จะบังคับฆ่า process ทิ้ง)
CELERY_TASK_SOFT_TIME_LIMIT = 300
CELERY_TASK_TIME_LIMIT = 360
```

### 742.7 ตารางสรุป `CELERY_BROKER_URL` เทียบกับ `CACHES` ที่ตั้งไว้ใน Part 069

| Redis DB Index | ใช้ทำอะไร | ตั้งไว้ใน Part |
|---|---|---|
| 0 | ค่าเริ่มต้นของ Redis (ไม่ได้ใช้ในโปรเจกต์นี้โดยเจาะจง) | — |
| 1 | Page/Query Cache (`CACHES['default']`) | Part 069 ขั้นตอนที่ 681.5 |
| 2 | Throttle Counters (`CACHES['throttle']`) | Part 069 ขั้นตอนที่ 681.5 |
| 3 | Session Storage (`CACHES['sessions']`) | Part 069 ขั้นตอนที่ 684.2 |
| 4 | **Celery Broker** (`CELERY_BROKER_URL`) | Part นี้ (075) |
| 5 | **Celery Result Backend** (`CELERY_RESULT_BACKEND`) | Part นี้ (075) |

การแยก DB index ยังคงหลักการเดิมจาก Part 069 ขั้นตอนที่ 681.5 ทุกประการ: ทำให้
`redis-cli -n 4 FLUSHDB` (ล้างคิวงานทั้งหมด — บางครั้งจำเป็นตอน debug) ไม่กระทบ
cache, throttle counter, หรือ session ของผู้ใช้ที่กำลังใช้งานอยู่เลย

### 742.8 ทดสอบว่า Celery เชื่อมต่อ Redis ได้จริง

```bash
python manage.py shell
```

```python
>>> from config.celery import app
>>> app.control.inspect().ping()
# ถ้ายังไม่ได้รัน worker (ขั้นตอนที่ 744) จะได้ None เพราะยังไม่มี worker ให้ตอบกลับ
# นี่เป็นเรื่องปกติ ณ จุดนี้ — เราจะรัน worker จริงในขั้นตอนถัดไป
```

```bash
# ตรวจสอบว่า Redis DB index 4 (broker) ว่างเปล่าอยู่ ก่อนเริ่มส่งงานจริง
redis-cli -n 4 dbsize
# (integer) 0
```

### 742.9 ตารางสรุปขั้นตอนที่ 742

| หัวข้อ | สรุป |
|---|---|
| ติดตั้ง | `pip install celery` (ไม่ต้องติดตั้ง `redis` เพิ่ม เพราะมีจาก Part 069 แล้ว) |
| ไฟล์ที่ต้องสร้าง | `config/celery.py` (สร้าง app instance) + แก้ `config/__init__.py` (import `celery_app`) |
| Broker | Redis DB index 4 (`CELERY_BROKER_URL`) |
| Result Backend | Redis DB index 5 (`CELERY_RESULT_BACKEND`) |
| Serializer | บังคับ JSON ทั้งหมด (`CELERY_ACCEPT_CONTENT`, `CELERY_TASK_SERIALIZER`, `CELERY_RESULT_SERIALIZER`) |
| `autodiscover_tasks()` | Celery หา `tasks.py` ในทุก app ของ `INSTALLED_APPS` ให้อัตโนมัติ |

---

## ขั้นตอนที่ 743: เขียน Task แรกด้วย `@shared_task` เรียกใช้ผ่าน `.delay()`

### 743.1 ทำไมใช้ `@shared_task` แทน `@app.task`

Celery มี decorator สองแบบสำหรับประกาศ task:

```python
# แบบที่ 1: @app.task — ผูกกับ Celery app instance ตรง ๆ
from config.celery import app

@app.task
def my_task():
    ...

# แบบที่ 2: @shared_task — ไม่ผูกกับ app instance ใด ๆ ตอนประกาศ
from celery import shared_task

@shared_task
def my_task():
    ...
```

| ประเด็น | `@app.task` | `@shared_task` |
|---|---|---|
| ต้อง import Celery app instance ไหม | ✅ ต้อง import `config.celery.app` เข้ามาในทุกไฟล์ที่เขียน task | ❌ ไม่ต้อง import อะไรนอกจาก `celery` เอง |
| ใช้ซ้ำใน reusable app (แอปที่แชร์ข้ามหลายโปรเจกต์) ได้ไหม | ❌ ยาก — แอปนั้นต้องรู้จัก Celery app instance เฉพาะของโปรเจกต์นี้เท่านั้น | ✅ ได้ — `@shared_task` จะไปผูกกับ app instance ที่ "current" อยู่ตอนรันจริง ไม่สนใจว่าอยู่โปรเจกต์ไหน |
| ความเสี่ยงเรื่อง Circular Import | สูงกว่า (ไฟล์ tasks.py ต้อง import จาก config ซึ่งอาจ import กลับมาที่ app อีกที) | ต่ำกว่ามาก |
| คำแนะนำอย่างเป็นทางการของ Celery | ใช้เมื่อรู้แน่ชัดว่ามี Celery app เดียวตลอดไป | **แนะนำเป็นค่าเริ่มต้นสำหรับโค้ดในแอป Django ทุกกรณี** |

**กฎของหลักสูตรนี้**: ใช้ `@shared_task` เสมอสำหรับ task ที่เขียนในแอปของ Django
(`blog/tasks.py`, `accounts/tasks.py`) เพราะเขียนง่ายกว่า เสี่ยง circular import
น้อยกว่า และเป็นแนวทางที่เอกสารทางการของ Celery แนะนำสำหรับโครงสร้างแบบนี้

### 743.2 Convention: ไฟล์ `tasks.py` ในแต่ละแอป

ทวนจากขั้นตอนที่ 742.4 — `app.autodiscover_tasks()` จะสแกนหาไฟล์ชื่อ `tasks.py`
ในทุกแอปที่อยู่ใน `INSTALLED_APPS` โดยอัตโนมัติ ดังนั้น convention มาตรฐานคือสร้าง
ไฟล์ `tasks.py` ไว้ในแต่ละแอป (เหมือนที่มี `views.py`, `models.py` อยู่แล้ว):

```python
# blog/tasks.py
import logging

from celery import shared_task
from django.conf import settings
from django.core.mail import send_mail

logger = logging.getLogger(__name__)


@shared_task
def send_welcome_email_task(user_email, username):
    """
    ส่งอีเมลต้อนรับสมาชิกใหม่แบบ background — ไม่บล็อก request cycle
    ของ view สมัครสมาชิกอีกต่อไป (ทวนปัญหาจากขั้นตอนที่ 741.2)
    """
    logger.info('กำลังส่งอีเมลต้อนรับไปยัง %s', user_email)
    send_mail(
        subject='ยินดีต้อนรับสู่ Blog ของเรา',
        message=f'สวัสดีคุณ {username}\n\nขอบคุณที่สมัครสมาชิกกับเรา',
        from_email=settings.DEFAULT_FROM_EMAIL,
        recipient_list=[user_email],
    )
    logger.info('ส่งอีเมลต้อนรับไปยัง %s สำเร็จ', user_email)
```

### 743.3 เรียกใช้ Task ผ่าน `.delay()`

`@shared_task` เติม method พิเศษให้ฟังก์ชันโดยอัตโนมัติ ที่สำคัญที่สุดคือ
`.delay()` — เรียกแล้วงานจะถูกส่งเข้าคิวทันที **ไม่รอผลลัพธ์**:

```python
# accounts/views.py — เวอร์ชันที่แก้ปัญหาจากขั้นตอนที่ 741.2 แล้ว
from django.shortcuts import redirect, render
from blog.tasks import send_welcome_email_task
from .forms import SignUpForm


def signup_view(request):
    if request.method == 'POST':
        form = SignUpForm(request.POST)
        if form.is_valid():
            user = form.save()

            # เปลี่ยนจากการเรียก send_mail() ตรง ๆ เป็นส่งเข้าคิวแทน
            # View return ทันทีโดยไม่รอ SMTP server เลย
            send_welcome_email_task.delay(user.email, user.username)

            return redirect('accounts:signup_success')
    else:
        form = SignUpForm()
    return render(request, 'accounts/signup.html', {'form': form})
```

**สิ่งที่เปลี่ยนไปมีแค่บรรทัดเดียว**: จาก `send_mail(...)` เป็น
`send_welcome_email_task.delay(user.email, user.username)` — โครงสร้างของ View
ที่เหลือไม่ต้องแก้อะไรเลย

### 743.4 `.delay()` คืนค่าอะไร

```python
>>> from blog.tasks import send_welcome_email_task
>>> result = send_welcome_email_task.delay('test@example.com', 'ทดสอบ')
>>> result
<AsyncResult: 3f2504e0-4f89-11d3-9a0c-0305e82c3301>
>>> result.id
'3f2504e0-4f89-11d3-9a0c-0305e82c3301'
```

`.delay()` คืนค่าเป็น **`AsyncResult`** object ทันที (ไม่รอ task ทำงานเสร็จ) โดยมี
`.id` เป็น UUID เฉพาะของงานนี้ — เก็บ `.id` นี้ไว้ได้ถ้าต้องการมาตรวจสอบสถานะทีหลัง
(รายละเอียดเต็มเรื่อง `AsyncResult` อยู่ในขั้นตอนที่ 745)

### 743.5 `.delay()` เทียบกับ `.apply_async()`

```python
# .delay() คือ shortcut ของ .apply_async() แบบไม่มี option พิเศษ
send_welcome_email_task.delay(user.email, user.username)

# เทียบเท่ากับ
send_welcome_email_task.apply_async(args=[user.email, user.username])

# .apply_async() ให้ควบคุมรายละเอียดเพิ่มเติมได้ เช่น หน่วงเวลาก่อนเริ่มทำงาน
send_welcome_email_task.apply_async(
    args=[user.email, user.username],
    countdown=60,          # รอ 60 วินาทีก่อนเริ่มทำงานจริง
)
```

| ประเด็น | `.delay(*args, **kwargs)` | `.apply_async(args=[...], kwargs={...}, **options)` |
|---|---|---|
| ความง่ายในการเขียน | ✅ สั้น กระชับ อ่านง่าย | ต้องห่อ args เป็น list เสมอ |
| ตั้งค่า `countdown`/`eta` (หน่วงเวลา), `queue` (เลือกคิวเฉพาะ), `expires` (หมดอายุ) | ❌ ทำไม่ได้ | ✅ ทำได้ครบ |
| ใช้บ่อยแค่ไหนในโค้ดจริง | บ่อยที่สุด (90% ของการเรียก task ทั่วไป) | ใช้เมื่อต้องการ option พิเศษเท่านั้น |

**คำแนะนำ**: ใช้ `.delay()` เป็นค่าเริ่มต้นเสมอสำหรับ Part นี้ ส่วน option ขั้นสูง
ของ `.apply_async()` เช่นการต่อ task เป็นลำดับ (chain), รันขนาน (group), หรือ
กำหนดตารางเวลาซ้ำ ๆ (Celery Beat) จะเจาะลึกเต็มรูปแบบใน **Part 076**

### 743.6 ทดสอบ Task ผ่าน Django Shell ก่อนเชื่อม View จริง

```bash
python manage.py shell
```

```python
>>> from blog.tasks import send_welcome_email_task
>>> send_welcome_email_task.delay('someone@example.com', 'คุณทดสอบ')
<AsyncResult: 8a1c3f2e-...>
```

ณ จุดนี้ถ้ายังไม่ได้เปิด Celery Worker (ขั้นตอนที่ 744) งานจะ**ถูกส่งเข้าคิวรอไว้
เฉย ๆ ใน Redis** โดยยังไม่มีใครมาทำ ตรวจสอบได้ว่างานเข้าคิวจริง:

```bash
redis-cli -n 4 llen celery
# (integer) 1   ← มี 1 งานรอคิวอยู่ (ชื่อคิวเริ่มต้นคือ "celery")
```

### 743.7 ตารางสรุปขั้นตอนที่ 743

| หัวข้อ | สรุป |
|---|---|
| Decorator ที่ใช้ | `@shared_task` (ไม่ผูกกับ app instance ตรง ๆ เหมาะกับโค้ดในแอป Django) |
| ไฟล์ที่เก็บ task | `<app_name>/tasks.py` — Celery หาเจอเองผ่าน `autodiscover_tasks()` |
| เรียกใช้แบบพื้นฐาน | `.delay(*args, **kwargs)` |
| เรียกใช้แบบมี option | `.apply_async(args=[...], countdown=..., queue=...)` |
| ค่าที่ได้กลับมา | `AsyncResult` object พร้อม `.id` — ไม่ block รอผลลัพธ์ |
| ก่อนมี Worker | งานเข้าคิวรอเฉย ๆ ใน Redis จนกว่าจะมี Worker มาหยิบไปทำ |

---

## ขั้นตอนที่ 744: การรัน Celery Worker

### 744.1 คำสั่งพื้นฐานในการรัน Worker

เปิด **terminal ใหม่แยกต่างหาก** จาก terminal ที่รัน `python manage.py runserver`
(ต้องรันคู่กันตลอดเวลาที่พัฒนา):

```bash
# activate venv ก่อนเสมอ (terminal ใหม่ = venv ใหม่ ต้อง activate ใหม่ทุกครั้ง)
source venv/bin/activate

# รัน Celery Worker โดยชี้ไปที่ Celery app ของโปรเจกต์ (-A ย่อจาก --app)
celery -A config worker --loglevel=info
```

ผลลัพธ์ที่ควรเห็นเมื่อ Worker เริ่มทำงานสำเร็จ:

```
 -------------- celery@my-laptop v5.4.0
--- ***** -----
-- ******* ---- Linux-6.8.0-x86_64
- *** --- * ---
- ** ---------- [config]
- ** ---------- .> app:         config:0x7f8a1c2b3d40
- ** ---------- .> transport:   redis://127.0.0.1:6379/4
- ** ---------- .> results:     redis://127.0.0.1:6379/5
- *** --- * --- .> concurrency: 8 (prefork)
-- ******* ---- .> task events: OFF (enable -E to monitor tasks in this worker)
--- ***** -----
 -------------- [queues]
                .> celery           exchange=celery(direct) key=celery

[tasks]
  . blog.tasks.send_welcome_email_task
  . config.celery.debug_task

[2026-09-26 10:00:00,123: INFO/MainProcess] Connected to redis://127.0.0.1:6379/4
[2026-09-26 10:00:00,145: INFO/MainProcess] celery@my-laptop ready.
```

สังเกตส่วน `[tasks]` — Celery แสดงรายการ task ทั้งหมดที่ `autodiscover_tasks()`
เจอ (ทวนจากขั้นตอนที่ 742.4) ถ้า task ที่เขียนไว้ไม่ปรากฏในรายการนี้ แปลว่ามีปัญหา
เรื่อง import หรือชื่อไฟล์ไม่ใช่ `tasks.py` — ต้องแก้ก่อนงานจะถูกประมวลผลได้

### 744.2 ทดสอบว่า Worker หยิบงานที่ค้างอยู่ไปทำจริง

ถ้าในขั้นตอนที่ 743.6 คุณส่งงานเข้าคิวไว้ก่อนเปิด Worker งานนั้นจะถูกหยิบไปทำทันที
ที่ Worker เริ่มทำงาน (log จะแสดงทันที):

```
[2026-09-26 10:00:01,001: INFO/ForkPoolWorker-1] Task blog.tasks.send_welcome_email_task[8a1c3f2e-...] received
[2026-09-26 10:00:01,205: INFO/ForkPoolWorker-1] Task blog.tasks.send_welcome_email_task[8a1c3f2e-...] succeeded in 0.198s: None
```

ตรวจสอบว่าคิวว่างแล้วหลัง Worker หยิบงานไปทำ:

```bash
redis-cli -n 4 llen celery
# (integer) 0   ← งานถูกหยิบไปทำหมดแล้ว
```

### 744.3 `--concurrency`: จำนวน Worker Process ที่ทำงานพร้อมกัน

```bash
# กำหนดจำนวน child process ที่ทำงานพร้อมกันชัดเจน (ค่าเริ่มต้น = จำนวน CPU core)
celery -A config worker --loglevel=info --concurrency=4
```

| ค่า `--concurrency` | ความหมาย | เหมาะกับสถานการณ์ไหน |
|---|---|---|
| ไม่ระบุ (ค่าเริ่มต้น) | เท่ากับจำนวน CPU core ของเครื่อง (ตรวจสอบด้วย `nproc`) | กรณีทั่วไปที่งานส่วนใหญ่เป็น CPU-bound |
| ตัวเลขต่ำ (เช่น 2-4) | จำกัด resource การใช้งานให้ไม่เยอะเกินไป | เครื่อง production ที่ต้อง share CPU กับ web server |
| ตัวเลขสูง (เช่น 20-50) | ทำงานพร้อมกันได้มาก แต่แต่ละ process ใช้ CPU น้อยมาก | งานที่ IO-bound เป็นหลัก (เช่น รอ network เยอะ อย่างการส่งอีเมล) และใช้ pool แบบ `gevent`/`eventlet` แทน `prefork` |

**คำแนะนำเริ่มต้น**: ปล่อยให้ Celery ใช้ค่า default (เท่าจำนวน core) ในเครื่อง
พัฒนา แล้วค่อยปรับจูนตามผลการวัดจริงเมื่อ deploy ขึ้น production (เทคนิค Load
Testing ที่ใช้วัดค่าที่เหมาะสมได้ ทวนจาก Part 072)

### 744.4 `--pool`: ประเภทของ Worker Pool

```bash
celery -A config worker --loglevel=info --pool=prefork    # ค่าเริ่มต้น (Linux/macOS)
celery -A config worker --loglevel=info --pool=solo       # จำเป็นสำหรับ Windows
celery -A config worker --loglevel=info --pool=threads    # ใช้ thread แทน process
```

| Pool Type | กลไกภายใน | เหมาะกับ |
|---|---|---|
| `prefork` (ค่าเริ่มต้น) | สร้าง **child process** จริงหลายตัวด้วย `os.fork()` — ได้ประโยชน์จาก multi-core เต็มที่ ไม่ติด GIL | งานทั่วไป โดยเฉพาะที่ใช้ CPU (image processing, PDF generation) — **ใช้ค่านี้เป็นค่าเริ่มต้นเสมอในหลักสูตรนี้** |
| `solo` | รันใน process เดียว ไม่มี concurrency เลย | **Windows เท่านั้น** เพราะ `os.fork()` ใช้ไม่ได้บน Windows |
| `threads` | ใช้ thread ภายใน process เดียว | งาน IO-bound ล้วน ๆ ที่ต้องการ concurrency สูงโดยไม่อยากเปลือง memory จากการ fork หลาย process |
| `gevent`/`eventlet` | Coroutine-based concurrency (ต้องติดตั้ง package เพิ่ม) | งาน IO-bound จำนวนมากมาก (หลักพัน task พร้อมกัน) เช่น ยิง HTTP request จำนวนมาก |

**คำเตือนสำคัญสำหรับผู้ใช้ Windows**: ถ้ารัน `celery -A config worker` แบบ
ค่าเริ่มต้นบน Windows จะเจอ error เกี่ยวกับ `fork` ทันที **ต้องใส่ `--pool=solo`
เสมอ** (ทวนแนวคิดจาก Part 001 ขั้นตอนที่ 6.3 ที่แนะนำให้ผู้ใช้ Windows ติดตั้ง
WSL2 — ถ้าใช้ WSL2 จะรันแบบ Linux ปกติได้โดยไม่ต้องใส่ flag พิเศษนี้เลย)

### 744.5 การรัน Worker ระหว่างพัฒนา: ต้องมี 2 Terminal เสมอ

```
Terminal 1:                          Terminal 2:
$ source venv/bin/activate           $ source venv/bin/activate
$ python manage.py runserver         $ celery -A config worker --loglevel=info
```

**นี่คือความแตกต่างสำคัญที่ต้องจำให้ขึ้นใจจาก Part นี้เป็นต้นไป**: ตั้งแต่ตอนนี้
ทุกครั้งที่พัฒนาฟีเจอร์ที่เกี่ยวข้องกับ Celery ต้องเปิด **สอง terminal พร้อมกันเสมอ**
ถ้าลืมเปิด Worker แล้วเรียก `.delay()` งานจะแค่**ค้างอยู่ในคิวเงียบ ๆ** ไม่มี error
ใด ๆ ปรากฏใน Django เลย (เพราะ `.delay()` แค่ส่งงานเข้า Redis สำเร็จ ไม่ได้รอดูว่า
มีใครมาทำหรือไม่) — เป็นสาเหตุอันดับหนึ่งที่มือใหม่สับสนว่า "ทำไมอีเมลไม่ถูกส่ง"

### 744.6 `--loglevel`: ระดับความละเอียดของ Log

```bash
celery -A config worker --loglevel=debug     # ละเอียดที่สุด เห็นทุกขั้นตอนภายใน
celery -A config worker --loglevel=info      # แนะนำสำหรับพัฒนาปกติ (ค่าที่ใช้ตลอด Part นี้)
celery -A config worker --loglevel=warning   # เห็นเฉพาะปัญหาที่อาจเกิดขึ้น
celery -A config worker --loglevel=error     # เห็นเฉพาะ error จริง ๆ (แนะนำสำหรับ production)
```

### 744.7 ตารางสรุปขั้นตอนที่ 744

| หัวข้อ | สรุป |
|---|---|
| คำสั่งพื้นฐาน | `celery -A config worker --loglevel=info` |
| ต้องรันคู่กับ | `python manage.py runserver` เสมอ (คนละ terminal) |
| `--concurrency` | จำนวน worker process พร้อมกัน (ค่าเริ่มต้น = จำนวน CPU core) |
| `--pool=prefork` | ค่าเริ่มต้นบน Linux/macOS ใช้ multi-process จริง ไม่ติด GIL |
| `--pool=solo` | จำเป็นสำหรับ Windows (ไม่มี `os.fork()`) |
| อาการทั่วไปเมื่อลืมเปิด Worker | งานค้างในคิวเงียบ ๆ ไม่มี error ปรากฏฝั่ง Django |

---

## ขั้นตอนที่ 745: Result Backend — เก็บผลลัพธ์ของ Task

### 745.1 Broker กับ Result Backend ต่างกันอย่างไร

ทวนจากขั้นตอนที่ 742.1: **Broker** ทำหน้าที่แค่ "ส่งงานจาก Django ไปยัง Worker"
เป็นทางเดียว — มันไม่รู้ด้วยซ้ำว่างานที่ส่งไปสำเร็จหรือล้มเหลว ถ้าต้องการรู้
**สถานะและผลลัพธ์**ของงานที่ทำไปแล้ว ต้องมี **Result Backend** แยกต่างหาก
(ในโปรเจกต์นี้ตั้งไว้ที่ Redis DB index 5 ตามขั้นตอนที่ 742.6)

```
Django View ──.delay()──> Broker (DB 4) ──> Worker ทำงาน ──> เขียนผลลัพธ์ ──> Result Backend (DB 5)
                                                                                    ▲
Django ตรวจสอบสถานะทีหลัง ─────────────────────────────────────────────────────────┘
```

### 745.2 `AsyncResult`: Object ที่ใช้ตรวจสอบสถานะ

```python
>>> from blog.tasks import send_welcome_email_task
>>> result = send_welcome_email_task.delay('test@example.com', 'ทดสอบ')
>>> result.id
'3f2504e0-4f89-11d3-9a0c-0305e82c3301'

>>> result.status
'PENDING'   # ยังไม่มี worker หยิบไปทำ (หรือกำลังรออยู่)

# ... รอสักครู่ให้ worker ทำงานเสร็จ ...

>>> result.status
'SUCCESS'
```

ถ้าต้องการดึง `AsyncResult` กลับมาจาก `task_id` ที่เก็บไว้ก่อนหน้า (เช่น เก็บไว้ใน
ฐานข้อมูลหรือ session เพื่อให้ผู้ใช้กลับมาเช็คสถานะทีหลัง):

```python
>>> from celery.result import AsyncResult
>>> result = AsyncResult('3f2504e0-4f89-11d3-9a0c-0305e82c3301')
>>> result.status
'SUCCESS'
```

### 745.3 ตารางสถานะทั้งหมดของ Task

| สถานะ (`result.status`) | ความหมาย |
|---|---|
| `PENDING` | ยังไม่มี worker รับงานไปทำ (หรือ task_id ไม่มีอยู่จริงเลย — Celery แยกสองกรณีนี้ไม่ได้ด้วยตัวเอง) |
| `STARTED` | Worker เริ่มทำงานแล้ว (ปรากฏเฉพาะเมื่อตั้ง `CELERY_TASK_TRACK_STARTED = True` ตามขั้นตอนที่ 742.6) |
| `RETRY` | Task ล้มเหลวและกำลังจะลองใหม่ (เจาะลึกในขั้นตอนที่ 746) |
| `SUCCESS` | ทำงานสำเร็จ — `result.result` จะมีค่าที่ task `return` กลับมา |
| `FAILURE` | ทำงานล้มเหลวและไม่ retry ต่อแล้ว — `result.result` จะเป็น exception object ที่เกิดขึ้น |

### 745.4 `.get()`: รอผลลัพธ์แบบ Synchronous (และทำไมต้องระวังมาก)

```python
>>> result = send_welcome_email_task.delay('test@example.com', 'ทดสอบ')
>>> value = result.get(timeout=10)   # บล็อกรอจนกว่า task จะเสร็จ หรือ timeout
```

**คำเตือนที่สำคัญที่สุดของขั้นตอนนี้**: **ห้ามเรียก `.get()` ใน Django View
โดยเด็ดขาด** ถ้าเผลอเขียนแบบนี้:

```python
# ผิดมาก — ทำลายเป้าหมายทั้งหมดของ Part นี้!
def signup_view(request):
    # ...
    result = send_welcome_email_task.delay(user.email, user.username)
    result.get(timeout=10)   # ❌ View กลับมา "บล็อกรอ" เหมือนเดิมทุกประการ!
    return redirect('accounts:signup_success')
```

การเรียก `.get()` ทันทีหลัง `.delay()` ทำให้ View **กลับไปรอ** จนกว่า Worker จะทำ
งานเสร็จอีกครั้ง เหมือนไม่ได้ย้ายไป background เลย (ย้อนกลับไปปัญหาในขั้นตอนที่
741.2 ทันที) — `.get()` มีประโยชน์เฉพาะในบริบทที่ **ตั้งใจรอผลลัพธ์จริง ๆ** เท่านั้น
เช่น script เบื้องหลังอีกตัวที่ต้องรอผลของ task ก่อนไปขั้นตอนถัดไป หรือใน unit test

### 745.5 วิธีที่ถูกต้อง: ให้ Frontend Poll สถานะแทน

```python
# blog/views.py
from django.http import JsonResponse
from celery.result import AsyncResult


def task_status_view(request, task_id):
    """API endpoint ให้ frontend เรียกถามสถานะ task เป็นระยะ (polling)"""
    result = AsyncResult(task_id)
    return JsonResponse({
        'task_id': task_id,
        'status': result.status,
        'ready': result.ready(),   # True เมื่อ SUCCESS หรือ FAILURE (จบแล้วไม่ว่าผลจะเป็นอย่างไร)
    })
```

```python
# blog/urls.py
path('tasks/<str:task_id>/status/', views.task_status_view, name='task_status'),
```

Flow ที่ถูกต้อง: View ส่งงานเข้าคิวแล้ว **return ทันที** พร้อม `task_id` กลับไปให้
JavaScript ฝั่ง frontend เก็บไว้ จากนั้น frontend เรียก endpoint นี้ซ้ำ ๆ ทุก 1-2
วินาที (หรือใช้ WebSocket ผ่าน Django Channels ที่เรียนไปแล้วใน Part 074 สำหรับ
งานที่ต้องการอัปเดตสถานะแบบ real-time จริง ๆ) จนกว่า `ready` จะเป็น `true`

### 745.6 `ignore_result=True`: ปิด Result Backend เฉพาะ Task ที่ไม่ต้องใช้

Task จำนวนมาก (เช่น การส่งอีเมลต้อนรับ) **ไม่มีใครสนใจผลลัพธ์เลย** — เขียนผลลัพธ์
เก็บใน Redis DB 5 ไปโดยเปล่าประโยชน์ เปลืองทั้ง memory และเวลาที่ Worker ต้องใช้
เขียนกลับ:

```python
@shared_task(ignore_result=True)
def send_welcome_email_task(user_email, username):
    """ไม่มีใคร .get() ผลลัพธ์นี้เลย — ปิดการเขียนผลลัพธ์เพื่อประหยัด resource"""
    # ... โค้ดเดิมจากขั้นตอนที่ 743.2 ...
```

**คำแนะนำระดับมืออาชีพ**: ตั้ง `ignore_result=True` เป็นค่าเริ่มต้นสำหรับทุก task
ที่ไม่ต้องมีใครมาตรวจสอบผลลัพธ์ทีหลัง (เช่น task แจ้งเตือน, task ส่งอีเมลทั่วไป)
และเปิด result backend ไว้เฉพาะ task ที่มี flow แบบขั้นตอนที่ 745.5 (frontend
ต้องมาถามสถานะจริง ๆ) เท่านั้น

### 745.7 ตารางสรุปขั้นตอนที่ 745

| หัวข้อ | สรุป |
|---|---|
| Result Backend | เก็บสถานะ/ผลลัพธ์ของ task แยกจาก Broker (Redis DB index 5 ในโปรเจกต์นี้) |
| ตรวจสอบสถานะ | `result.status` — `PENDING`/`STARTED`/`RETRY`/`SUCCESS`/`FAILURE` |
| รอผลลัพธ์แบบ block | `result.get(timeout=...)` — **ห้ามใช้ใน View เด็ดขาด** |
| วิธีที่ถูกต้อง | ให้ frontend poll ผ่าน endpoint ที่เช็ค `AsyncResult(task_id).status` |
| ประหยัด resource | ใส่ `ignore_result=True` ให้ task ที่ไม่มีใครตรวจผลลัพธ์ |

---

## ขั้นตอนที่ 746: Task Retry — จัดการความล้มเหลวแบบอัตโนมัติ

### 746.1 ทำไม Task ถึงล้มเหลวได้บ่อยกว่าที่คิด

Task จำนวนมาก (โดยเฉพาะที่เรียก service ภายนอกตามตารางในขั้นตอนที่ 741.4) ล้มเหลว
ได้จากสาเหตุที่ **ไม่ใช่ bug ในโค้ดเลย** เช่น SMTP server ปิดปรับปรุงชั่วคราว,
network กระตุกเสี้ยววินาที, external API rate limit ชั่วคราว — ปัญหาเหล่านี้
**มักหายไปเองถ้าลองใหม่อีกครั้งหลังรอสักครู่**

### 746.2 `autoretry_for`: Retry อัตโนมัติเมื่อเจอ Exception ที่กำหนด

```python
# blog/tasks.py
import logging
import smtplib

from celery import shared_task
from django.conf import settings
from django.core.mail import send_mail

logger = logging.getLogger(__name__)


@shared_task(
    bind=True,
    autoretry_for=(smtplib.SMTPException, ConnectionError, TimeoutError),
    max_retries=5,
    retry_backoff=True,
    retry_backoff_max=600,
    retry_jitter=True,
)
def send_welcome_email_task(self, user_email, username):
    """
    ส่งอีเมลต้อนรับ พร้อม retry อัตโนมัติสูงสุด 5 ครั้งเมื่อเจอปัญหาเครือข่าย/SMTP
    ที่มักเป็นปัญหาชั่วคราว ไม่ใช่ bug ของโค้ดเอง
    """
    logger.info('กำลังส่งอีเมลต้อนรับไปยัง %s (ครั้งที่ %d)', user_email, self.request.retries + 1)
    send_mail(
        subject='ยินดีต้อนรับสู่ Blog ของเรา',
        message=f'สวัสดีคุณ {username}\n\nขอบคุณที่สมัครสมาชิกกับเรา',
        from_email=settings.DEFAULT_FROM_EMAIL,
        recipient_list=[user_email],
    )
```

### 746.3 อธิบายแต่ละ Parameter

| Parameter | ความหมาย |
|---|---|
| `bind=True` | ทำให้ task รับ `self` เป็น argument แรก เพื่อเข้าถึง `self.request` (ดู retry count ปัจจุบัน) และเรียก `self.retry()` เองได้ (ขั้นตอนที่ 746.5) |
| `autoretry_for=(ExceptionA, ExceptionB, ...)` | ถ้า task โยน exception ประเภทใดประเภทหนึ่งในนี้ Celery จะ retry ให้อัตโนมัติ โดยไม่ต้องเขียน `try/except` เอง |
| `max_retries=5` | ลองซ้ำได้สูงสุด 5 ครั้ง หลังจากนั้นถ้ายังล้มเหลวอยู่ จะกลายเป็นสถานะ `FAILURE` ถาวร |
| `retry_backoff=True` | เปิดใช้ **Exponential Backoff** — เวลารอก่อน retry แต่ละครั้งจะเพิ่มขึ้นเป็นทวีคูณ แทนที่จะรอเวลาเท่ากันทุกครั้ง |
| `retry_backoff_max=600` | จำกัดเพดานสูงสุดของเวลารอไม่ให้เกิน 600 วินาที (10 นาที) แม้จะ retry ไปหลายรอบแล้ว |
| `retry_jitter=True` | เพิ่มความสุ่มเล็กน้อยในเวลารอ ป้องกัน task จำนวนมากที่ล้มเหลวพร้อมกัน (เช่น SMTP server ล่มพร้อมกันหมด) มา retry พร้อมกันเป๊ะอีกครั้ง (แนวคิดเดียวกับ Cache Stampede ที่ทวนจาก Part 069 ขั้นตอนที่ 683.1) |

### 746.4 คำนวณเวลารอจริงของ Exponential Backoff

สูตรของ Celery: เวลารอ (วินาที) ก่อน retry ครั้งที่ N คือประมาณ `2^N` วินาที
(ปรับด้วย jitter แบบสุ่มถ้าเปิดไว้):

| Retry ครั้งที่ | เวลารอโดยประมาณ (ไม่มี jitter) | เวลารอจริง (มี `retry_jitter=True`) |
|---|---|---|
| 1 | 1 วินาที | สุ่มระหว่าง 0-1 วินาที |
| 2 | 2 วินาที | สุ่มระหว่าง 0-2 วินาที |
| 3 | 4 วินาที | สุ่มระหว่าง 0-4 วินาที |
| 4 | 8 วินาที | สุ่มระหว่าง 0-8 วินาที |
| 5 | 16 วินาที | สุ่มระหว่าง 0-16 วินาที |

**ทำไมต้องเพิ่มเวลารอแบบทวีคูณแทนที่จะรอเท่ากันทุกครั้ง**: ถ้า SMTP server กำลัง
มีปัญหาหนัก การ retry ถี่ ๆ ทุก 1 วินาทีรัว ๆ 5 ครั้งจะยิ่งซ้ำเติมปัญหา (คล้ายกับการ
DDoS ตัวเอง) แต่การเว้นระยะให้นานขึ้นเรื่อย ๆ ให้เวลา server ฟื้นตัวจริง ๆ ก่อนลอง
ใหม่ ซึ่งเป็นแนวปฏิบัติมาตรฐานของระบบ distributed ทั่วไป

### 746.5 Retry แบบ Manual ด้วย `self.retry()` (เมื่อ `autoretry_for` ไม่พอ)

บางครั้งการตัดสินใจว่าจะ retry หรือไม่ ซับซ้อนกว่าแค่ "เจอ exception ประเภทนี้ก็
retry" เช่น ต้องเช็ค HTTP status code ที่ได้กลับมาก่อนตัดสินใจ:

```python
# blog/tasks.py
import requests
from celery import shared_task


@shared_task(bind=True, max_retries=3)
def notify_external_webhook_task(self, webhook_url, payload):
    """ยิง webhook ไปยัง service ภายนอก retry เองเมื่อได้ status code ที่บ่งบอกปัญหาชั่วคราว"""
    try:
        response = requests.post(webhook_url, json=payload, timeout=5)
        response.raise_for_status()
    except requests.exceptions.RequestException as exc:
        # 5xx = ปัญหาฝั่ง server ปลายทาง (มักเป็นชั่วคราว) → ควร retry
        # 4xx = ปัญหาฝั่งเรา (เช่น payload ผิดรูปแบบ) → retry ไปก็ไม่มีทางสำเร็จ ไม่ควร retry
        if isinstance(exc, requests.exceptions.HTTPError) and 400 <= exc.response.status_code < 500:
            raise  # ปล่อยให้ล้มเหลวถาวรทันที ไม่ retry
        # กรณีอื่น (timeout, connection error, 5xx) ให้ retry แบบ exponential backoff เอง
        raise self.retry(exc=exc, countdown=2 ** self.request.retries)
```

**จุดสำคัญ**: `self.retry()` ต้องมี `raise` นำหน้าเสมอ เพราะ `self.retry()`
ทำงานโดยการโยน exception พิเศษ (`Retry`) ออกไปภายใน — ถ้าลืม `raise` โค้ดหลังจาก
บรรทัดนั้นจะยังทำงานต่อไปทั้งที่ตั้งใจจะหยุดแล้ว

### 746.6 ตารางเปรียบเทียบ `autoretry_for` กับ `self.retry()`

| ประเด็น | `autoretry_for` | `self.retry()` แบบ manual |
|---|---|---|
| ความซับซ้อนในการเขียน | ต่ำมาก — แค่ระบุใน decorator | สูงกว่า — ต้องเขียน `try/except` เอง |
| ควบคุมเงื่อนไขว่าจะ retry เมื่อไหร่ | ควบคุมได้แค่ "ชนิดของ exception" | ควบคุมได้ละเอียดทุกกรณี (เช่น เช็ค status code, เนื้อหา response) |
| เหมาะกับ | Task ทั่วไปที่ error ชนิดไหนก็ควร retry เหมือนกันหมด | Task ที่ต้องแยกแยะว่า error แบบไหนควร retry แบบไหนไม่ควร |

### 746.7 ตารางสรุปขั้นตอนที่ 746

| หัวข้อ | สรุป |
|---|---|
| `autoretry_for` | Retry อัตโนมัติเมื่อเจอ exception ที่ระบุไว้ |
| `max_retries` | จำนวนครั้งสูงสุดที่ลองซ้ำได้ ก่อนกลายเป็น `FAILURE` ถาวร |
| `retry_backoff` | เปิด Exponential Backoff — เวลารอเพิ่มขึ้นเป็นทวีคูณทุกครั้งที่ retry |
| `retry_jitter` | สุ่มเวลารอเล็กน้อย ป้องกัน retry พร้อมกันจำนวนมาก (thundering herd) |
| `self.retry()` | Retry แบบ manual เมื่อ logic การตัดสินใจซับซ้อนกว่าแค่ชนิด exception |

---

## ขั้นตอนที่ 747: Task Serialization — JSON เทียบกับ Pickle

### 747.1 Serialization คืออะไร และทำไม Task ต้องมีมัน

เมื่อเรียก `.delay(arg1, arg2)` ค่า `arg1`, `arg2` ต้องถูกแปลงเป็น **string/bytes**
ก่อน เพื่อส่งผ่าน Redis (ซึ่งเก็บได้แค่ string/bytes เท่านั้น — ทวนแนวคิด Redis
data structure จาก Part 069 ขั้นตอนที่ 682.1) แล้ว Worker ที่ฝั่งรับต้องแปลงกลับ
เป็น Python object อีกครั้งก่อนเรียกใช้ฟังก์ชันจริง กระบวนการแปลงไป-กลับนี้เรียกว่า
**Serialization/Deserialization** และ Celery รองรับหลายรูปแบบ ที่สำคัญที่สุดคือ
**JSON** และ **Pickle**

### 747.2 ทำไม Celery เปลี่ยน Default จาก Pickle เป็น JSON (ตั้งแต่เวอร์ชัน 4.0)

Celery เวอร์ชันเก่า (ก่อน 4.0) ใช้ **Pickle** เป็น serializer เริ่มต้น เพราะ Pickle
แปลง Python object ได้แทบทุกชนิดโดยไม่ต้องเขียนโค้ดแปลงเอง แต่มีปัญหาด้านความ
ปลอดภัยร้ายแรงซ่อนอยู่:

```python
import pickle

# Pickle สามารถ deserialize เป็นการ "รันโค้ดใด ๆ ก็ได้" ถ้าข้อมูลถูกปลอมแปลง
# (นี่ไม่ใช่ตัวอย่างสมมติ — เป็นช่องโหว่ด้านความปลอดภัยที่มีการรายงานจริงในอุตสาหกรรม)
class Malicious:
    def __reduce__(self):
        import os
        return (os.system, ('echo "โดนแฮกแล้ว!" > /tmp/hacked.txt',))

payload = pickle.dumps(Malicious())
# เมื่อฝั่งรับเรียก pickle.loads(payload) โค้ดใน __reduce__ จะถูกรันทันที
# โดยไม่ต้องมี "การเรียกฟังก์ชันที่เป็นอันตราย" จากฝั่งเราเลยแม้แต่น้อย
```

**เงื่อนไขที่ทำให้อันตรายนี้เกิดขึ้นจริง**: ถ้า Broker (Redis ในกรณีนี้) ถูกเข้าถึง
โดยไม่ได้รับอนุญาต (เช่น ตั้งค่า Redis ไม่มี password, เปิด port สู่ internet
โดยไม่ได้ตั้งใจ) ผู้โจมตีสามารถ**แทรกข้อความปลอมที่เป็น pickle ที่เป็นอันตราย**
เข้าไปในคิวโดยตรง เมื่อ Worker หยิบมา deserialize ด้วย `pickle.loads()` โค้ดของ
ผู้โจมตีจะถูกรันทันทีบนเครื่อง Worker — เรียกว่าเป็นช่องโหว่ประเภท **Insecure
Deserialization** (อยู่ในรายการ OWASP Top 10 มาอย่างต่อเนื่อง)

ทีมพัฒนา Celery ตัดสินใจเปลี่ยน **default เป็น JSON ตั้งแต่เวอร์ชัน 4.0 เป็นต้นมา**
เพราะ **JSON parser ไม่มีความสามารถรันโค้ดได้เลยไม่ว่ากรณีใด** — JSON แปลงได้แค่
`dict`, `list`, `str`, `int`, `float`, `bool`, `None` เท่านั้น ไม่มีทางที่การ parse
JSON จะกลายเป็นการรันคำสั่งระบบได้

### 747.3 ตารางเปรียบเทียบ JSON กับ Pickle

| ประเด็น | JSON | Pickle |
|---|---|---|
| ความปลอดภัยเมื่อ deserialize ข้อมูลที่ไม่น่าเชื่อถือ | ✅ ปลอดภัย — parse ได้แค่ data structure พื้นฐาน | ❌ อันตรายมาก — อาจรันโค้ดใด ๆ ก็ได้ (Insecure Deserialization) |
| รองรับชนิดข้อมูล Python พื้นฐาน (`dict`, `list`, `str`, `int`) | ✅ รองรับ | ✅ รองรับ |
| รองรับ `datetime`, `Decimal` โดยตรง | ❌ ไม่รองรับ — ต้องแปลงเป็น string/number เองก่อนส่ง | ✅ รองรับตรง ๆ โดยไม่ต้องแปลง |
| รองรับ Custom Class/Model instance โดยตรง | ❌ ไม่รองรับเลย (ทวนเหตุผลเชิงลึกในขั้นตอนที่ 748) | ✅ รองรับ (แต่ไม่ควรทำ ตามเหตุผลในขั้นตอนที่ 748 อยู่ดี) |
| ทำงานข้ามภาษาโปรแกรมมิ่งอื่นได้ไหม (เช่น Worker ที่เขียนด้วยภาษาอื่น) | ✅ ได้ — JSON เป็นมาตรฐานสากล | ❌ ผูกกับ Python เท่านั้น |
| Default ของ Celory ตั้งแต่เวอร์ชันไหน | 4.0 เป็นต้นมา (2016) | ก่อน 4.0 |
| คำแนะนำของหลักสูตรนี้ | **ใช้เสมอ** | หลีกเลี่ยงโดยสิ้นเชิง เว้นแต่มีเหตุผลเฉพาะเจาะจงจริง ๆ และควบคุม Broker ได้ 100% |

### 747.4 ตั้งค่าบังคับ JSON อย่างชัดเจน (ทวนจากขั้นตอนที่ 742.6)

```python
# config/settings.py
CELERY_ACCEPT_CONTENT = ['json']   # Worker จะ "ปฏิเสธ" งานที่ serialize มาแบบอื่นทันที
CELERY_TASK_SERIALIZER = 'json'
CELERY_RESULT_SERIALIZER = 'json'
```

`CELERY_ACCEPT_CONTENT = ['json']` คือบรรทัดที่สำคัญที่สุด — มันบอก Worker ว่า
**ห้ามรับ/ประมวลผลข้อความที่ serialize มาด้วยวิธีอื่นเด็ดขาด** แม้ว่าจะมีใครพยายาม
ส่ง pickle payload เข้ามาในคิว (ไม่ว่าจะตั้งใจหรือมี bug ที่ไหนสักแห่งในระบบ)
Worker จะปฏิเสธทันทีโดยไม่พยายาม deserialize เลย นี่คือเกราะป้องกันชั้นสุดท้ายที่
สำคัญมาก

### 747.5 ข้อจำกัดของ JSON: ต้องแปลง `datetime`/`Decimal` เองก่อนส่ง

```python
# ผิด — datetime ไม่ใช่ชนิดข้อมูลที่ JSON serialize ได้ตรง ๆ
from django.utils import timezone

@shared_task
def process_event_task(occurred_at):
    ...

process_event_task.delay(timezone.now())
# TypeError: Object of type datetime is not JSON serializable
```

```python
# ถูก — แปลงเป็น ISO 8601 string ก่อนส่งเข้า task เสมอ
from django.utils import timezone
from django.utils.dateparse import parse_datetime

@shared_task
def process_event_task(occurred_at_iso):
    occurred_at = parse_datetime(occurred_at_iso)   # แปลงกลับเป็น datetime ในฝั่ง Worker
    # ... ใช้งาน occurred_at ต่อตามปกติ ...

process_event_task.delay(timezone.now().isoformat())
```

เช่นเดียวกันกับ `Decimal` (ใช้บ่อยกับราคาสินค้า, จำนวนเงิน — ทวนจาก Part 001
ขั้นตอนที่ 3.3 ที่ `Product.price` เป็น `DecimalField`):

```python
# ผิด
@shared_task
def process_payment_task(amount):
    ...

process_payment_task.delay(Decimal('199.50'))  # TypeError เช่นกัน

# ถูก — แปลงเป็น str ก่อนส่ง แล้วแปลงกลับเป็น Decimal ในฝั่ง Worker
from decimal import Decimal

@shared_task
def process_payment_task(amount_str):
    amount = Decimal(amount_str)
    ...

process_payment_task.delay(str(Decimal('199.50')))
```

**หลักการทอง**: ก่อนเรียก `.delay()` ให้ถามตัวเองเสมอว่า *"argument ที่กำลังจะส่ง
เข้าไปนี้ ถ้าเอาไปทำ `json.dumps()` ตรง ๆ จะ error ไหม"* ถ้าใช่ ต้องแปลงเป็น
string/number/dict/list พื้นฐานก่อนส่งเสมอ

### 747.6 ตารางสรุปขั้นตอนที่ 747

| หัวข้อ | สรุป |
|---|---|
| Default serializer ปัจจุบัน | JSON (ตั้งแต่ Celery 4.0) |
| ทำไมเลิกใช้ Pickle เป็น default | ป้องกัน Insecure Deserialization — ผู้โจมตีอาจรันโค้ดผ่าน Broker ที่ถูกเข้าถึงโดยไม่ได้รับอนุญาต |
| `CELERY_ACCEPT_CONTENT` | ต้องตั้งเป็น `['json']` เสมอ เพื่อให้ Worker ปฏิเสธ payload ที่ไม่ใช่ JSON โดยอัตโนมัติ |
| ข้อจำกัดของ JSON | ไม่รองรับ `datetime`, `Decimal`, custom object ตรง ๆ — ต้องแปลงเป็น string ก่อนส่งเสมอ |

---

## ขั้นตอนที่ 748: ส่ง Argument ที่ซับซ้อนเข้า Task อย่างถูกวิธี

### 748.1 กฎเหล็กที่สำคัญที่สุดข้อหนึ่งของ Celery: ห้ามส่ง Model Instance ตรง ๆ

```python
# blog/views.py — ผิดมาก แม้จะดูเหมือนสะดวกที่สุด
from blog.tasks import notify_new_comment_task
from .models import Comment


def add_comment_view(request, post_id):
    # ...
    comment = Comment.objects.create(post_id=post_id, author=request.user, body=body)

    # ❌ ส่ง Model instance ตรง ๆ เข้า task — ห้ามทำแบบนี้เด็ดขาด
    notify_new_comment_task.delay(comment)
    # ...
```

### 748.2 ทำไมถึงห้าม: มีปัญหาซ้อนกันถึง 3 ชั้น

**ปัญหาที่ 1 — Serialize ไม่ได้เลยตั้งแต่แรก**: ทวนจากขั้นตอนที่ 747.5 — JSON
serialize ได้แค่ data structure พื้นฐาน `Comment` เป็น Django Model instance ที่
ซับซ้อนกว่านั้นมาก (มี `_state`, related manager, methods ต่าง ๆ) การพยายามส่งเข้า
task ด้วย `CELERY_TASK_SERIALIZER = 'json'` จะทำให้เกิด `TypeError` ทันทีตั้งแต่
ตอนเรียก `.delay()`

```
TypeError: Object of type Comment is not JSON serializable
```

**ปัญหาที่ 2 — แม้จะสมมติว่าใช้ Pickle ได้ ข้อมูลก็จะ "เก่า" (Stale Data)**:
สมมติย้อนกลับไปใช้ Pickle (ซึ่งไม่ควรทำตามขั้นตอนที่ 747 อยู่แล้ว) — ค่าที่ถูก
serialize เก็บไว้คือ **สถานะของ object ณ วินาทีที่เรียก `.delay()`** ถ้า Worker
หยิบงานไปทำช้ากว่านั้น 30 วินาที (เช่นคิวมีงานค้างอยู่เยอะ) และระหว่างนั้นมีคนอื่น
แก้ไข comment นั้นไปแล้ว (เช่น ผู้ดูแลระบบลบคอมเมนต์เพราะไม่เหมาะสม) Task จะยังคง
ทำงานกับข้อมูล**เวอร์ชันเก่า**ที่ไม่ตรงกับความเป็นจริงในฐานข้อมูลอีกต่อไป

**ปัญหาที่ 3 — เปลืองพื้นที่ Broker โดยไม่จำเป็น**: Model instance ทั้งก้อนมีขนาด
ใหญ่กว่า primary key เพียงตัวเดียวมาก (โดยเฉพาะถ้ามี field ขนาดใหญ่อย่าง
`TextField`) ทำให้ Redis Broker ต้องเก็บข้อมูลซ้ำซ้อนกับที่มีอยู่แล้วในฐานข้อมูล
โดยไม่จำเป็น

### 748.3 วิธีที่ถูกต้อง: ส่ง Primary Key แล้ว Query ใหม่ในฝั่ง Worker

```python
# blog/tasks.py
import logging

from celery import shared_task
from django.conf import settings
from django.core.mail import send_mail

from .models import Comment

logger = logging.getLogger(__name__)


@shared_task(bind=True, autoretry_for=(ConnectionError,), max_retries=3, retry_backoff=True)
def notify_new_comment_task(self, comment_id):
    """
    รับแค่ comment_id (int ธรรมดา — JSON serialize ได้ทันที) แล้ว query
    ข้อมูลล่าสุดจากฐานข้อมูลเองในฝั่ง Worker เสมอ รับประกันว่าได้ข้อมูลปัจจุบัน
    ที่สุด ไม่ใช่ snapshot เก่าตอนที่ view เรียก .delay()
    """
    try:
        comment = Comment.objects.select_related('post', 'author', 'post__author').get(
            id=comment_id
        )
    except Comment.DoesNotExist:
        # กรณีที่ comment ถูกลบไปแล้วก่อน worker จะหยิบงานมาทำ (เช่นโดน spam filter
        # ลบทิ้งอัตโนมัติ) — ไม่ใช่ error ที่ต้อง retry เพราะ retry ไปก็ไม่มีทางเจอ
        # ให้ log ไว้เฉย ๆ แล้วจบ task อย่างเงียบ ๆ
        logger.warning('ไม่พบ Comment id=%s อาจถูกลบไปแล้วก่อน worker จะทำงาน', comment_id)
        return

    post_author_email = comment.post.author.email
    send_mail(
        subject=f'มีคอมเมนต์ใหม่ในบทความ "{comment.post.title}"',
        message=f'{comment.author.username} เขียนว่า: {comment.body}',
        from_email=settings.DEFAULT_FROM_EMAIL,
        recipient_list=[post_author_email],
    )
    logger.info('แจ้งเตือนคอมเมนต์ id=%s สำเร็จ', comment_id)
```

```python
# blog/views.py — เรียกใช้แบบถูกต้อง
from blog.tasks import notify_new_comment_task
from .models import Comment


def add_comment_view(request, post_id):
    # ...
    comment = Comment.objects.create(post_id=post_id, author=request.user, body=body)

    # ✅ ส่งแค่ primary key (int ธรรมดา) — JSON serialize ได้ทันที ไม่มีปัญหาข้อมูลเก่า
    notify_new_comment_task.delay(comment.id)
    # ...
```

### 748.4 ตารางเปรียบเทียบ: ส่ง Model Instance vs ส่ง Primary Key

| ประเด็น | ส่ง Model Instance ตรง ๆ (❌) | ส่ง Primary Key (✅) |
|---|---|---|
| JSON Serialize ได้ไหม | ❌ ไม่ได้เลย — `TypeError` ทันที | ✅ ได้ (`int`/`str` เป็นชนิดพื้นฐาน) |
| ข้อมูลที่ Worker ใช้เป็นข้อมูลล่าสุดเสมอไหม | ❌ ไม่ใช่ — เป็น snapshot ตอนเรียก `.delay()` | ✅ ใช่ — query ใหม่ทุกครั้งตอน Worker เริ่มทำงานจริง |
| จัดการกรณี object ถูกลบไปแล้วได้ไหม | ❌ ไม่รู้เลยว่าถูกลบไปแล้ว (ใช้ snapshot เก่าทำงานต่อ) | ✅ เจอ `DoesNotExist` ชัดเจน จัดการได้ตรงจุด |
| ขนาดข้อมูลใน Broker | ใหญ่ (ทั้ง object) | เล็กมาก (แค่ตัวเลขเดียว) |
| หลักการที่ยึดตาม | — | เดียวกับ REST API ที่ส่ง ID ผ่าน URL แล้วค่อย query (ทวนแนวคิดจาก Phase 5) |

### 748.5 กรณีที่ต้องส่งข้อมูลหลายชิ้น: ใช้ ID หลายตัว ไม่ใช่ Object หลายตัว

```python
# ✅ ถูกต้อง — ส่งเป็น list ของ id ธรรมดา
@shared_task
def send_bulk_notification_task(post_id, user_ids):
    post = Post.objects.get(id=post_id)
    users = User.objects.filter(id__in=user_ids)
    for user in users:
        # ... ส่งแจ้งเตือนแต่ละคน ...
        pass

send_bulk_notification_task.delay(post.id, list(user.id for user in target_users))
```

### 748.6 ตารางสรุปขั้นตอนที่ 748

| หัวข้อ | สรุป |
|---|---|
| กฎเหล็ก | ห้ามส่ง Django Model instance เข้า `.delay()`/`.apply_async()` โดยเด็ดขาด |
| วิธีที่ถูกต้อง | ส่ง primary key (หรือ list ของ primary key) แล้ว query ใหม่ในฝั่ง Worker เสมอ |
| ประโยชน์ที่ได้ | Serialize ได้จริงด้วย JSON, ข้อมูลล่าสุดเสมอ, ไม่มีปัญหา stale data, ขนาดเล็ก |
| ต้องจัดการเพิ่ม | `DoesNotExist` — object อาจถูกลบไปแล้วระหว่างที่งานรอคิวอยู่ |

---

## ขั้นตอนที่ 749: จัดการ Error ใน Task — `on_failure` Hook และ Logging

### 749.1 ทำไมการจัดการ Error ใน Task ถึงสำคัญกว่าใน View ปกติ

เมื่อ View เกิด error ผู้ใช้จะเห็นหน้า 500 ทันที (หรือ Django Debug Page ตอนพัฒนา)
— **มีคนรู้ทันทีว่าเกิดปัญหา** แต่เมื่อ Task ที่รันใน background เกิด error
**ไม่มีใครเห็นเลยถ้าไม่ได้ตั้งระบบแจ้งเตือนไว้** ผู้ใช้ที่สมัครสมาชิกอาจไม่รู้เลย
ว่าอีเมลต้อนรับของตัวเองไม่เคยถูกส่งออกไป เพราะ Task ล้มเหลวเงียบ ๆ อยู่เบื้องหลัง
— นี่คือเหตุผลที่การจัดการ Error อย่างเป็นระบบใน Celery Task จึงสำคัญไม่แพ้การ
เขียน logic ของ task เอง

### 749.2 `on_failure` Hook: ทำงานอัตโนมัติทุกครั้งที่ Task ล้มเหลวถาวร

Celery มี concept ของ **Custom Task Class** ที่ override behavior บางอย่างได้
`on_failure` คือ method ที่ถูกเรียกอัตโนมัติทุกครั้งที่ task ล้มเหลว**หลัง retry
ครบจำนวนสูงสุดแล้ว** (หรือไม่มีการตั้ง retry เลยตั้งแต่ต้น):

```python
# core/celery_tasks.py
import logging

from celery import Task

logger = logging.getLogger(__name__)


class LoggingTask(Task):
    """
    Base Task class ที่ log ทุกครั้งที่ task ล้มเหลวถาวร (หลัง retry หมดจำนวนแล้ว)
    นำไปใช้ร่วมกับ shared_task ทุกตัวที่ต้องการติดตาม failure อย่างเป็นระบบ
    """

    def on_failure(self, exc, task_id, args, kwargs, einfo):
        logger.error(
            'Task ล้มเหลวถาวร: name=%s task_id=%s args=%s kwargs=%s error=%s',
            self.name, task_id, args, kwargs, exc,
            exc_info=einfo,   # แนบ traceback เต็มรูปแบบเข้า log
        )
        # จุดนี้คือที่ที่ควรเชื่อมต่อระบบแจ้งเตือนภายนอก เช่น Sentry, Slack webhook
        # หรือส่งอีเมลแจ้งทีม dev — รายละเอียดการ monitor แบบเต็มรูปแบบเรียนใน Part 079
        super().on_failure(exc, task_id, args, kwargs, einfo)
```

```python
# blog/tasks.py
from celery import shared_task
from core.celery_tasks import LoggingTask


@shared_task(
    base=LoggingTask,
    bind=True,
    autoretry_for=(ConnectionError,),
    max_retries=3,
    retry_backoff=True,
)
def send_welcome_email_task(self, user_email, username):
    # ... โค้ดเดิมจากขั้นตอนที่ 746.2 ...
    pass
```

**จุดสำคัญ**: `on_failure` จะถูกเรียก **หลังจาก** retry ครบทุกครั้งแล้วเท่านั้น
ถ้า task retry สำเร็จในครั้งที่ 2 จาก `max_retries=3` `on_failure` จะไม่ถูกเรียก
เลย — มันคือ "ทางออกสุดท้าย" ก่อนที่ task จะถูกทิ้งเป็น `FAILURE` ถาวรจริง ๆ

### 749.3 `on_retry` และ `on_success`: Hook อื่น ๆ ที่มีประโยชน์

```python
class LoggingTask(Task):
    def on_failure(self, exc, task_id, args, kwargs, einfo):
        logger.error('Task ล้มเหลวถาวร: %s [%s] — %s', self.name, task_id, exc)
        super().on_failure(exc, task_id, args, kwargs, einfo)

    def on_retry(self, exc, task_id, args, kwargs, einfo):
        logger.warning(
            'Task กำลัง retry: %s [%s] ครั้งที่ %d — สาเหตุ: %s',
            self.name, task_id, self.request.retries + 1, exc,
        )
        super().on_retry(exc, task_id, args, kwargs, einfo)

    def on_success(self, retval, task_id, args, kwargs):
        logger.info('Task สำเร็จ: %s [%s]', self.name, task_id)
        super().on_success(retval, task_id, args, kwargs)
```

| Hook | ถูกเรียกเมื่อไหร่ |
|---|---|
| `on_success(retval, task_id, args, kwargs)` | Task ทำงานสำเร็จ (return ปกติ ไม่มี exception) |
| `on_failure(exc, task_id, args, kwargs, einfo)` | Task ล้มเหลวถาวร (หลัง retry ครบจำนวนแล้ว หรือ exception ที่ไม่ได้อยู่ใน `autoretry_for`) |
| `on_retry(exc, task_id, args, kwargs, einfo)` | Task กำลังจะ retry (ยังไม่ถือว่าล้มเหลวถาวร) |

### 749.4 Logging ภายใน Task: ใช้ `logging` แทน `print()` เสมอ

```python
# ผิด — print() ไปโผล่ที่ stdout ของ worker process เท่านั้น ไม่มีระดับความสำคัญ
# ไม่มี timestamp มาตรฐาน ไม่ถูกส่งต่อไปยังระบบ log aggregation ใด ๆ
@shared_task
def bad_example_task():
    print('เริ่มทำงาน')
    print('เกิดปัญหา!')

# ถูก — ใช้ logging module มาตรฐานของ Python/Django
import logging
logger = logging.getLogger(__name__)

@shared_task
def good_example_task():
    logger.info('เริ่มทำงาน')
    logger.error('เกิดปัญหา!')
```

`logging.getLogger(__name__)` ทำให้ log message ที่ออกมาจาก task มี **ชื่อ module
ต้นทาง** (เช่น `blog.tasks`) ติดไปด้วยเสมอ ทำให้ตามหาต้นตอได้ง่ายเมื่อระบบมี task
จำนวนมาก และยังเชื่อมกับระบบ `LOGGING` configuration มาตรฐานของ Django ได้ทันที
(ส่งต่อไปยังไฟล์ log, console, หรือระบบ log aggregation ภายนอกได้ตามที่ตั้งค่าไว้)

### 749.5 ตั้งค่า Logger เฉพาะสำหรับ Celery ใน `settings.py`

```python
# config/settings.py
LOGGING = {
    'version': 1,
    'disable_existing_loggers': False,
    'formatters': {
        'verbose': {
            'format': '{asctime} [{levelname}] {name}: {message}',
            'style': '{',
        },
    },
    'handlers': {
        'console': {
            'class': 'logging.StreamHandler',
            'formatter': 'verbose',
        },
    },
    'loggers': {
        'celery': {
            'handlers': ['console'],
            'level': 'INFO',
            'propagate': False,
        },
        'blog.tasks': {
            'handlers': ['console'],
            'level': 'INFO',
            'propagate': False,
        },
    },
}
```

### 749.6 ตารางสรุปขั้นตอนที่ 749

| หัวข้อ | สรุป |
|---|---|
| ทำไมสำคัญ | Task ที่ล้มเหลวใน background ไม่มีใครเห็นทันทีเหมือน error ใน View |
| `on_failure` | Hook ที่ถูกเรียกอัตโนมัติเมื่อ task ล้มเหลวถาวร (หลัง retry ครบแล้ว) |
| `on_retry` / `on_success` | Hook เสริมสำหรับติดตามสถานะระหว่างทางและตอนสำเร็จ |
| Logging | ใช้ `logging.getLogger(__name__)` เสมอ ห้ามใช้ `print()` ใน task |
| ขั้นต่อไป | เชื่อมต่อ `on_failure` เข้ากับระบบแจ้งเตือนจริง (Sentry/Slack) และ Dashboard ติดตามสถานะ (Flower) ใน Part 079 |

---

## ขั้นตอนที่ 750: สรุปและแบบฝึกหัด — แปลงระบบอีเมลของ Blog เป็น Celery Task

### 750.1 ประกอบทุกอย่างเข้าด้วยกัน: `config/celery.py` และ `config/settings.py` ฉบับสมบูรณ์

```python
# config/celery.py (ฉบับสมบูรณ์ของ Part นี้)
import os

from celery import Celery

os.environ.setdefault('DJANGO_SETTINGS_MODULE', 'config.settings')

app = Celery('config')
app.config_from_object('django.conf:settings', namespace='CELERY')
app.autodiscover_tasks()


@app.task(bind=True, ignore_result=True)
def debug_task(self):
    print(f'Request: {self.request!r}')
```

```python
# config/__init__.py (ฉบับสมบูรณ์)
from .celery import app as celery_app

__all__ = ('celery_app',)
```

```python
# config/settings.py — ส่วน Celery (ฉบับสมบูรณ์รวมทุกขั้นตอนของ Part นี้)
REDIS_HOST = os.environ.get('REDIS_HOST', '127.0.0.1')
REDIS_PORT = os.environ.get('REDIS_PORT', '6379')

CELERY_BROKER_URL = f'redis://{REDIS_HOST}:{REDIS_PORT}/4'
CELERY_RESULT_BACKEND = f'redis://{REDIS_HOST}:{REDIS_PORT}/5'
CELERY_ACCEPT_CONTENT = ['json']
CELERY_TASK_SERIALIZER = 'json'
CELERY_RESULT_SERIALIZER = 'json'
CELERY_TIMEZONE = TIME_ZONE
CELERY_ENABLE_UTC = True
CELERY_TASK_TRACK_STARTED = True
CELERY_RESULT_EXPIRES = 3600
CELERY_TASK_SOFT_TIME_LIMIT = 300
CELERY_TASK_TIME_LIMIT = 360
```

### 750.2 Base Task Class สำหรับใช้ร่วมกันทั้งโปรเจกต์

```python
# core/celery_tasks.py (ฉบับสมบูรณ์)
import logging

from celery import Task

logger = logging.getLogger(__name__)


class LoggingTask(Task):
    """Base Task ที่ log สถานะสำคัญของทุก task ที่ใช้ base=LoggingTask"""

    def on_failure(self, exc, task_id, args, kwargs, einfo):
        logger.error(
            'Task ล้มเหลวถาวร: name=%s task_id=%s args=%s kwargs=%s error=%s',
            self.name, task_id, args, kwargs, exc, exc_info=einfo,
        )
        super().on_failure(exc, task_id, args, kwargs, einfo)

    def on_retry(self, exc, task_id, args, kwargs, einfo):
        logger.warning(
            'Task กำลัง retry: name=%s task_id=%s ครั้งที่ %d เหตุผล=%s',
            self.name, task_id, self.request.retries + 1, exc,
        )
        super().on_retry(exc, task_id, args, kwargs, einfo)

    def on_success(self, retval, task_id, args, kwargs):
        logger.info('Task สำเร็จ: name=%s task_id=%s', self.name, task_id)
        super().on_success(retval, task_id, args, kwargs)
```

### 750.3 แปลง Password Reset ให้เป็น Celery Task

```python
# accounts/tasks.py
import logging
import smtplib

from celery import shared_task
from django.conf import settings
from django.contrib.auth import get_user_model
from django.contrib.auth.tokens import default_token_generator
from django.core.mail import send_mail
from django.template.loader import render_to_string
from django.utils.encoding import force_bytes
from django.utils.http import urlsafe_base64_encode

from core.celery_tasks import LoggingTask

logger = logging.getLogger(__name__)
User = get_user_model()


@shared_task(
    base=LoggingTask,
    bind=True,
    autoretry_for=(smtplib.SMTPException, ConnectionError, TimeoutError),
    max_retries=5,
    retry_backoff=True,
    retry_backoff_max=600,
    retry_jitter=True,
    ignore_result=True,
)
def send_password_reset_email_task(self, user_id, protocol, domain):
    """
    ส่งอีเมล reset password แบบ background
    รับแค่ user_id (primary key) ตามกฎเหล็กในขั้นตอนที่ 748 — query User ใหม่
    เสมอในฝั่ง worker เพื่อให้ได้ข้อมูลล่าสุด (เช่น อีเมลที่อาจถูกเปลี่ยนไปแล้ว)
    """
    try:
        user = User.objects.get(id=user_id, is_active=True)
    except User.DoesNotExist:
        logger.warning('ไม่พบผู้ใช้ id=%s หรือถูกปิดใช้งานแล้ว — ยกเลิกการส่งอีเมล reset', user_id)
        return

    uid = urlsafe_base64_encode(force_bytes(user.pk))
    token = default_token_generator.make_token(user)
    reset_url = f'{protocol}://{domain}/accounts/reset/{uid}/{token}/'

    message = render_to_string('accounts/password_reset_email.txt', {
        'user': user,
        'reset_url': reset_url,
    })

    logger.info('กำลังส่งอีเมล reset password ไปยัง user_id=%s', user_id)
    send_mail(
        subject='คำขอตั้งรหัสผ่านใหม่',
        message=message,
        from_email=settings.DEFAULT_FROM_EMAIL,
        recipient_list=[user.email],
    )
```

```python
# accounts/views.py
from django.contrib.auth import get_user_model
from django.contrib.auth.forms import PasswordResetForm
from django.shortcuts import redirect, render

from .tasks import send_password_reset_email_task

User = get_user_model()


def password_reset_request_view(request):
    if request.method == 'POST':
        form = PasswordResetForm(request.POST)
        if form.is_valid():
            email = form.cleaned_data['email']
            user = User.objects.filter(email=email, is_active=True).first()

            # หมายเหตุด้านความปลอดภัย: ตอบกลับ "ส่งลิงก์แล้ว" เสมอไม่ว่าจะเจอ
            # อีเมลนี้ในระบบหรือไม่ ป้องกันผู้ไม่หวังดีใช้ฟอร์มนี้เช็คว่าอีเมล
            # ไหนมีบัญชีอยู่ในระบบบ้าง (User Enumeration — เรียนเต็มใน Phase 10)
            if user is not None:
                send_password_reset_email_task.delay(
                    user.id, request.scheme, request.get_host()
                )
            return redirect('accounts:password_reset_done')
    else:
        form = PasswordResetForm()
    return render(request, 'accounts/password_reset_form.html', {'form': form})
```

### 750.4 แปลง Comment Notification ให้เป็น Celery Task

```python
# blog/tasks.py (ฉบับสมบูรณ์รวมทุกขั้นตอนของ Part นี้)
import logging
import smtplib

from celery import shared_task
from django.conf import settings
from django.core.mail import send_mail

from core.celery_tasks import LoggingTask
from .models import Comment

logger = logging.getLogger(__name__)


@shared_task(
    base=LoggingTask,
    bind=True,
    autoretry_for=(smtplib.SMTPException, ConnectionError, TimeoutError),
    max_retries=5,
    retry_backoff=True,
    retry_backoff_max=600,
    retry_jitter=True,
    ignore_result=True,
)
def notify_new_comment_task(self, comment_id):
    """แจ้งเตือนเจ้าของบทความเมื่อมีคอมเมนต์ใหม่ (ทวนโครงสร้างจากขั้นตอนที่ 748.3)"""
    try:
        comment = Comment.objects.select_related('post', 'author', 'post__author').get(
            id=comment_id
        )
    except Comment.DoesNotExist:
        logger.warning('ไม่พบ Comment id=%s อาจถูกลบไปแล้วก่อน worker จะทำงาน', comment_id)
        return

    if not comment.post.author.email:
        logger.info('เจ้าของบทความ id=%s ไม่มีอีเมล ข้ามการแจ้งเตือน', comment.post.id)
        return

    send_mail(
        subject=f'มีคอมเมนต์ใหม่ในบทความ "{comment.post.title}"',
        message=f'{comment.author.username} เขียนว่า:\n\n{comment.body}',
        from_email=settings.DEFAULT_FROM_EMAIL,
        recipient_list=[comment.post.author.email],
    )
    logger.info('แจ้งเตือนคอมเมนต์ id=%s ไปยัง %s สำเร็จ', comment_id, comment.post.author.email)
```

```python
# blog/views.py (ส่วนที่เกี่ยวข้อง)
from django.contrib.auth.decorators import login_required
from django.shortcuts import get_object_or_404, redirect

from .forms import CommentForm
from .models import Post
from .tasks import notify_new_comment_task


@login_required
def add_comment_view(request, post_id):
    post = get_object_or_404(Post, id=post_id, is_published=True)
    if request.method == 'POST':
        form = CommentForm(request.POST)
        if form.is_valid():
            comment = form.save(commit=False)
            comment.post = post
            comment.author = request.user
            comment.save()

            # ส่งแค่ primary key เข้าคิว — View return ทันทีโดยไม่รอส่งอีเมลเลย
            notify_new_comment_task.delay(comment.id)

            return redirect('blog:detail', slug=post.slug)
    else:
        form = CommentForm()
    return render(request, 'blog/add_comment.html', {'form': form, 'post': post})
```

### 750.5 คำสั่งรันระบบทั้งหมดพร้อมกันสำหรับพัฒนา (ทวนจากขั้นตอนที่ 744.5)

```bash
# Terminal 1: Django development server
source venv/bin/activate
python manage.py runserver

# Terminal 2: Celery Worker
source venv/bin/activate
celery -A config worker --loglevel=info

# Terminal 3 (ทางเลือก): ตรวจสอบ Redis โดยตรงระหว่างพัฒนา
redis-cli -n 4 monitor   # ดูคำสั่งที่ Broker ได้รับแบบ real-time
```

### 750.6 สรุปสิ่งที่ได้เรียนรู้ใน Part นี้

- ✅ เข้าใจปัญหาของงานที่บล็อก request cycle และรู้ว่างานประเภทไหนควรเป็น
  background task
- ✅ ติดตั้ง Celery และตั้งค่า Redis เป็น Broker ต่อยอดจากโครงสร้างที่มีอยู่แล้ว
  จาก Part 069 โดยแยก DB index ชัดเจน (Broker = DB 4, Result Backend = DB 5)
- ✅ เขียน Task แรกด้วย `@shared_task` และเรียกใช้ผ่าน `.delay()`
- ✅ รัน Celery Worker แยก process และเข้าใจ `--concurrency`, `--pool`
- ✅ ใช้ Result Backend ติดตามสถานะ task ผ่าน `AsyncResult` และรู้ว่าทำไม
  `.get()` ห้ามใช้ใน View
- ✅ ตั้งค่า Retry อัตโนมัติด้วย `autoretry_for`, `max_retries`, `retry_backoff`,
  `retry_jitter`
- ✅ เข้าใจว่าทำไม Celery เปลี่ยน default serializer จาก Pickle เป็น JSON
  เพื่อความปลอดภัย และรู้ข้อจำกัดของ JSON กับ `datetime`/`Decimal`
- ✅ เรียนรู้กฎเหล็ก: ห้ามส่ง Model instance เข้า task ตรง ๆ ต้องส่ง primary key
  แทนเสมอ
- ✅ จัดการ Error อย่างเป็นระบบด้วย `on_failure` hook และ logging ที่ถูกต้อง
- ✅ แปลงระบบส่งอีเมลของ Blog (password reset, comment notification) ทั้งหมด
  ให้เป็น Celery Task ที่ทนทานและไม่บล็อก request cycle

### 750.7 Checklist ก่อนไป Part ถัดไป

- [ ] รัน `pip install celery` และเห็น `celery` ใน `requirements.txt`
- [ ] สร้าง `config/celery.py` และแก้ `config/__init__.py` ให้ import `celery_app`
- [ ] ตั้งค่า `CELERY_BROKER_URL`/`CELERY_RESULT_BACKEND` ชี้ไป Redis DB index
      4 และ 5 ตามลำดับ
- [ ] รัน `celery -A config worker --loglevel=info` แล้วเห็น worker พร้อมทำงาน
      (`celery@... ready.`)
- [ ] เขียน task ทดสอบด้วย `@shared_task` แล้วเรียกผ่าน `.delay()` จาก shell
      สำเร็จ
- [ ] ตรวจสอบสถานะ task ผ่าน `AsyncResult(task_id).status` ได้ถูกต้อง
- [ ] ทดสอบ `autoretry_for` โดยจำลอง exception แล้วเห็น log แสดงการ retry
- [ ] ยืนยันว่า `CELERY_ACCEPT_CONTENT = ['json']` ถูกตั้งไว้ในโปรเจกต์แล้ว
- [ ] แปลงอย่างน้อย 1 task ในโปรเจกต์ให้ส่ง primary key แทน Model instance
- [ ] เพิ่ม `LoggingTask` เป็น base class และเห็น log เมื่อ task ล้มเหลว/สำเร็จ

### 750.8 แบบฝึกหัดท้ายบท

**แบบฝึกหัดที่ 1**: เขียน `@shared_task` ชื่อ `send_export_ready_email_task` ที่รับ
`user_id` และ `file_path` (ผลลัพธ์จาก Streaming CSV Export ที่เขียนไว้ใน Part 071
ขั้นตอนที่ 705) แล้วส่งอีเมลแจ้งผู้ใช้ว่าไฟล์ export พร้อมดาวน์โหลดแล้ว พร้อม
`autoretry_for` และ `LoggingTask` เหมือนตัวอย่างในขั้นตอนที่ 750.3-750.4

**แบบฝึกหัดที่ 2**: จำลองสถานการณ์ SMTP server ล่ม (แก้ `EMAIL_HOST` ใน
`settings.py` ชั่วคราวให้ชี้ไปที่ host ที่ไม่มีอยู่จริง เช่น `nonexistent.invalid`)
แล้วรัน task ส่งอีเมลจริง สังเกต log ของ Worker ว่าเกิด retry ตามลำดับเวลาแบบ
exponential backoff หรือไม่ (ทวนสูตรจากขั้นตอนที่ 746.4) บันทึกเวลาที่ retry
แต่ละครั้งเกิดขึ้นจริงเทียบกับตารางทฤษฎี

**แบบฝึกหัดที่ 3**: เขียนโค้ดที่ **จงใจทำผิด** ตามขั้นตอนที่ 748.1 (ส่ง Model
instance ของ `Post` เข้า task ตรง ๆ) แล้วสังเกต error message ที่ Celery แสดงออกมา
เมื่อพยายาม `.delay()` จากนั้นแก้ไขให้ถูกต้องตามขั้นตอนที่ 748.3 และเขียนอธิบาย
ด้วยคำพูดตัวเอง (ในไฟล์ `notes.md`) ว่า error message ที่เห็นเกี่ยวข้องกับหัวข้อ
Serialization ในขั้นตอนที่ 747 อย่างไร

**แบบฝึกหัดที่ 4 (ขั้นสูง)**: สร้าง view `task_status_view` ตามขั้นตอนที่ 745.5
แล้วเขียนหน้า HTML ง่าย ๆ ที่ใช้ JavaScript `fetch()` เรียก endpoint นี้ทุก 2 วินาที
(polling) เพื่อแสดงสถานะการส่งอีเมล export จากแบบฝึกหัดที่ 1 แบบ real-time บนหน้า
เว็บ (เช่น "กำลังส่ง..." → "ส่งสำเร็จแล้ว") โดยไม่ต้อง refresh หน้าเว็บเอง

### 750.9 คำถามที่พบบ่อย (FAQ)

**Q: ถ้าลืมเปิด Celery Worker แล้วเรียก `.delay()` จะเกิด error ไหม?**
A: ไม่เกิด error ใด ๆ เลยฝั่ง Django — `.delay()` แค่ส่งงานเข้า Redis สำเร็จ
(ทวนจากขั้นตอนที่ 744.5) งานจะรอคิวอยู่เงียบ ๆ จนกว่าจะมี Worker มาหยิบไปทำ
ถ้าสงสัยว่างานหายไปไหน ให้เช็คด้วย `redis-cli -n 4 llen celery` เพื่อดูว่ามีงาน
ค้างอยู่ในคิวกี่ชิ้น

**Q: จำเป็นต้องมี Result Backend เสมอไหม ถ้าไม่สนใจผลลัพธ์เลย?**
A: ไม่จำเป็น — ถ้าทุก task ในโปรเจกต์ตั้ง `ignore_result=True` (ทวนจากขั้นตอนที่
745.6) จะไม่มีการเขียนอะไรลง Result Backend เลย แต่หลักสูตรนี้ยังคงแนะนำให้ตั้งค่า
`CELERY_RESULT_BACKEND` ไว้เสมอ เผื่ออนาคตมี task ที่ต้องการติดตามสถานะแบบ
ขั้นตอนที่ 745.5

**Q: ทำไม Celery Worker ต้องรันแยก process จาก Django ทำไมไม่รวมกันไปเลย?**
A: เพราะ web server กับ background job มีลักษณะการใช้ resource ต่างกันมาก — web
server ต้องตอบสนองเร็วต่อ request จำนวนมากพร้อมกัน ในขณะที่ Worker อาจต้องใช้
เวลานานกับงานหนักบางชิ้น การแยก process ทำให้ปรับสเกลแต่ละส่วนได้อิสระจากกัน
(เช่น เพิ่มจำนวน Worker โดยไม่ต้องเพิ่ม web server, หรือ deploy Worker ไปคนละ
เครื่องกับ web server เลยก็ได้) ทวนหลักการนี้จากขั้นตอนที่ 742.1

**Q: ควรใช้ Redis หรือ RabbitMQ เป็น Broker ในโปรเจกต์ production จริง?**
A: ทั้งคู่ใช้งานได้ดีและเป็นที่นิยมทั้งคู่ — หลักสูตรนี้เลือก Redis ใน Part นี้
เพราะมีอยู่แล้วในโปรเจกต์จาก Part 069 ทำให้เรียนรู้ได้เร็วโดยไม่ต้องติดตั้งระบบ
ใหม่ แต่ RabbitMQ มีจุดเด่นด้าน message durability และ routing ที่ซับซ้อนกว่า
ในบางสถานการณ์ — จะเปรียบเทียบรายละเอียดทั้งสองแบบอย่างลึกซึ้งใน **Part 077**

**Q: `@shared_task` กับ `@app.task` เลือกผิดจะมีปัญหาจริงจังไหม?**
A: สำหรับโปรเจกต์เดี่ยว ๆ ที่ไม่ได้แชร์แอปข้ามหลายโปรเจกต์ ความแตกต่างในทางปฏิบัติ
มีน้อยมาก แต่หลักสูตรนี้แนะนำ `@shared_task` เป็นมาตรฐานเสมอ (ทวนเหตุผลจาก
ขั้นตอนที่ 743.1) เพราะเขียนง่ายกว่า เสี่ยง circular import น้อยกว่า และเป็น
แนวทางที่เอกสารทางการแนะนำสำหรับโค้ดในแอป Django

---

## เตรียมตัวสำหรับ Part ถัดไป

**Part 076: Celery ขั้นสูง — Periodic Tasks, Chains, Chords** จะพาคุณต่อยอดจาก
พื้นฐานที่แน่นแล้วใน Part นี้ ไปสู่การใช้งาน Celery ระดับที่ซับซ้อนขึ้น: ตั้งเวลา
ให้ task รันซ้ำอัตโนมัติตามตารางเวลาด้วย **Celery Beat** (เช่น sync ยอด view จาก
Redis Hash กลับฐานข้อมูลทุก 5 นาที ตามที่ Part 069 ขั้นตอนที่ 682.3 เกริ่นไว้, หรือ
รีเซ็ต trending score ทุกสัปดาห์ตามขั้นตอนที่ 682.4), ต่อ task หลายตัวให้ทำงาน
**เรียงลำดับกัน** ด้วย **Chain** (เช่น resize รูปภาพเสร็จแล้วค่อยส่งอีเมลแจ้ง),
รัน task หลายตัว**พร้อมกัน**แล้วรอให้ครบทุกตัวก่อนไปขั้นตอนถัดไปด้วย **Chord**
(เช่น ประมวลผลไฟล์หลายส่วนพร้อมกันแล้วค่อยรวมผลลัพธ์) และเทคนิคการออกแบบ workflow
ของ task ที่ซับซ้อนอย่างเป็นระบบ

เตรียม Celery Worker ที่รันได้แล้วจาก Part นี้ให้พร้อม เพราะ Part ถัดไปจะใช้
โครงสร้างเดิมทั้งหมดต่อยอดทันทีโดยไม่ต้องติดตั้งใหม่!
