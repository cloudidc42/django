# Part 079: Background Job Monitoring: Flower, Django-RQ

> **ขั้นตอนที่ 781-790 ของหลักสูตร** | Phase 9: Async, Celery และ Channels (Part สุดท้ายของ Phase)
>
> เป้าหมายของ Part นี้: ตลอด Part 075-078 คุณสร้างระบบ background task ที่ซับซ้อนขึ้น
> เรื่อย ๆ — Celery worker, Celery Beat, Chain/Group/Chord, Message Queue, และระบบแจ้งเตือน
> real-time — แต่ทุกอย่างที่ผ่านมา คุณ "มองไม่เห็น" ว่างานที่ยิงเข้าคิวไปแล้วเกิดอะไรขึ้น
> จริง ๆ นอกจากอ่าน log ทีละบรรทัด Part นี้จะเปิด **หน้าต่างสู่ห้องเครื่อง** ของระบบ
> background job ทั้งหมด ด้วย **Flower** (Web UI สำหรับ Celery) และแนะนำ **django-rq**
> ทางเลือกที่เรียบง่ายกว่า Celery พร้อม **RQ Dashboard** ของมันเอง จากนั้นจะเชื่อมต่อ
> การแจ้งเตือนความล้มเหลวเข้ากับ Sentry, วางหลัก Logging ที่ถูกต้องสำหรับ background
> task, และปิดท้ายด้วยการ shutdown worker อย่างปลอดภัยตอน deploy — ก่อนจะสรุปภาพรวม
> ทั้ง Phase 9 อย่างละเอียด พร้อม quiz และแบบฝึกหัดใหญ่ปิดท้าย ก่อนเข้าสู่ Phase 10:
> Security ใน Part 080

---

## สารบัญของ Part นี้

- ขั้นตอนที่ 781: `Flower` — Web UI สำหรับ Monitor Celery
- ขั้นตอนที่ 782: ติดตั้งและตั้งค่า Flower พร้อม Authentication
- ขั้นตอนที่ 783: อ่าน Dashboard ของ Flower — ประวัติ Task, สถานะ Worker, กราฟต่าง ๆ
- ขั้นตอนที่ 784: `django-rq` เป็นทางเลือกแทน Celery
- ขั้นตอนที่ 785: `RQ Dashboard` สำหรับ Monitor django-rq
- ขั้นตอนที่ 786: ตารางเปรียบเทียบ Celery vs django-RQ — เมื่อไหร่ควรเลือกอะไร
- ขั้นตอนที่ 787: การแจ้งเตือนเมื่อ Background Job ล้มเหลว (เชื่อม Sentry)
- ขั้นตอนที่ 788: Best Practice การทำ Logging สำหรับ Background Task
- ขั้นตอนที่ 789: การ Shutdown Worker อย่างสวยงาม (Graceful Shutdown) และข้อควรพิจารณาตอน Deploy
- ขั้นตอนที่ 790: สรุป Phase 9 ทั้งหมด (Part 073-079) + Quiz + แบบฝึกหัดใหญ่ + คำนำสู่ Phase 10

---

## ขั้นตอนที่ 781: `Flower` — Web UI สำหรับ Monitor Celery

### 781.1 ปัญหา: ระบบ Celery ที่ซับซ้อนขึ้นเรื่อย ๆ แต่มองไม่เห็นอะไรเลย

ทวนความจำจาก Part 075-076: ตอนนี้ระบบของคุณมี Celery worker หลายกลุ่มแยกตาม queue
(`high_priority`, `default`, `low_priority` จากขั้นตอนที่ 756.3), มี Celery Beat ยิง
periodic task, มี chain/group/chord ที่ทำงานซับซ้อนหลายขั้นตอน และมี message broker
(Redis หรือ RabbitMQ จาก Part 077) ที่รับ-ส่งงานตลอดเวลา

คำถามที่เกิดขึ้นทุกวันในทีมที่ดูแลระบบนี้คือ:

- "ตอนนี้มี worker ทำงานอยู่กี่ตัว? ตัวไหนหายไปหรือเปล่า?"
- "Task `send_password_reset_email` ที่ผู้ใช้บ่นว่าไม่ได้รับอีเมล มันรันไปหรือยัง? ล้มเหลว
  ตรงไหน?"
- "คิว `low_priority` มีงานค้างอยู่กี่งาน? งานสะสมเพิ่มขึ้นเรื่อย ๆ ไหม (แปลว่า worker
  ไม่พอ)?"
- "Task ไหนใช้เวลารันนานผิดปกติเมื่อเทียบกับค่าเฉลี่ย?"

การหาคำตอบเหล่านี้จาก **log file ล้วน ๆ** ทำได้ แต่ช้ามากและไม่เหมาะกับการดูแบบ
เรียลไทม์ นี่คือปัญหาที่ **Flower** ถูกสร้างมาแก้โดยเฉพาะ

### 781.2 Flower คืออะไร

**Flower** คือเครื่องมือ Web-based monitoring และ administration สำหรับ Celery cluster
พัฒนาโดยชุมชน Celery เอง (ปัจจุบันอยู่ภายใต้องค์กร Celery Project บน GitHub) ทำงานโดย
**ฟังสัญญาณ (event) ที่ Celery worker ส่งออกมา** ผ่าน broker เดียวกับที่ worker ใช้
แล้วนำมาแสดงผลเป็นหน้าเว็บที่อ่านง่าย

```
┌─────────────┐  events (task-started,        ┌─────────────┐
│   Celery    │  task-succeeded, task-failed,  │             │
│   Worker    │ ──────────────────────────────>│    Redis /  │
│  (worker A) │                                 │   RabbitMQ  │
└─────────────┘                                 │  (Broker)   │
┌─────────────┐                                 │             │
│   Celery    │ ──────────────────────────────> │             │
│   Worker    │                                 └──────┬──────┘
│  (worker B) │                                        │ subscribe events
└─────────────┘                                        ▼
                                                  ┌─────────────┐
                                                  │   Flower    │
                                                  │ (Web UI :5555)│
                                                  └──────┬──────┘
                                                         │ HTTP
                                                         ▼
                                                  เบราว์เซอร์ของคุณ
```

จุดสำคัญที่ต้องเข้าใจ: **Flower ไม่ได้ควบคุม worker โดยตรง** มันแค่ฟัง event ที่ worker
กระจายออกมาผ่าน broker (แบบเดียวกับที่ Celery ใช้ยิงงานเข้าคิว) และสามารถส่งคำสั่งควบคุม
บางอย่างกลับไป (ผ่านกลไก **remote control** ของ Celery) เช่น revoke task, shutdown worker
— รายละเอียดจะอยู่ในขั้นตอนที่ 783.5

### 781.3 ฟีเจอร์หลักของ Flower

| ฟีเจอร์ | รายละเอียด |
|---|---|
| Real-time Worker Monitoring | เห็นว่า worker ตัวไหน online/offline, concurrency, load average |
| Task History | ดูประวัติ task ย้อนหลัง (state, args, kwargs, runtime, exception) |
| Graphs | กราฟจำนวน task สำเร็จ/ล้มเหลวต่อวินาที, กราฟ worker online ตามเวลา |
| Remote Control | Revoke/Terminate task, restart pool, ปรับ rate limit จาก UI โดยไม่ต้อง SSH เข้าเครื่อง |
| REST API | `/api/tasks`, `/api/workers` ฯลฯ — ใช้เขียนสคริปต์ตรวจสอบสุขภาพระบบอัตโนมัติได้ |
| Broker Monitoring | ดูความยาวคิวได้ (สมบูรณ์ที่สุดเมื่อใช้ RabbitMQ ที่มี Management API) |
| Authentication | Basic Auth หรือ OAuth2 (Google) ป้องกันการเข้าถึงโดยไม่ได้รับอนุญาต |

### 781.4 ทำไม Flower ถึงจำเป็นแม้จะมี APM อย่าง Sentry แล้ว (ทวนจาก Part 066)

Part 066 ขั้นตอนที่ 657 สอน APM Tools อย่าง Sentry Performance ไปแล้ว หลายคนอาจสงสัยว่า
ถ้ามี Sentry แล้วยังต้องมี Flower อีกทำไม คำตอบคือทั้งสองเครื่องมือ **ตอบคำถามคนละแบบ**:

| ประเด็น | Flower | Sentry/APM |
|---|---|---|
| จุดโฟกัส | สถานะ **ปัจจุบัน** ของ worker/คิว/task แบบเรียลไทม์ | Error tracking + performance trend ระยะยาว |
| ควบคุม worker ได้ไหม | ✅ Revoke/restart pool ได้จาก UI โดยตรง | ❌ ดูได้อย่างเดียว ควบคุมไม่ได้ |
| ต้องติดตั้งเพิ่มในโค้ดแอปไหม | ไม่ต้องแก้โค้ด task เลย แค่รันกระบวนการแยก | ต้องติดตั้ง SDK และครอบ integration (ขั้นตอนที่ 787) |
| เก็บข้อมูลย้อนหลังนานแค่ไหน | จำกัดตามหน่วยความจำ/`--persistent` (ไม่เหมาะเก็บนานเป็นเดือน) | เก็บยาวนาน ค้นหา query ย้อนหลังได้สะดวกกว่ามาก |
| เหมาะกับ | Debug เฉพาะหน้า, ดูสุขภาพ worker วันต่อวัน | วิเคราะห์แนวโน้ม, แจ้งเตือนอัตโนมัติ, สืบสวน incident |

**สรุปหลักการ**: ทีมมืออาชีพใช้ **ทั้งสองร่วมกัน** — Flower สำหรับดูว่า "ตอนนี้ระบบเป็น
อย่างไร" แบบเรียลไทม์ และ Sentry สำหรับ "เกิดอะไรขึ้นเมื่อคืนนี้ และมันเกิดขึ้นบ่อยแค่ไหน"

### 781.5 ตารางสรุปขั้นตอนที่ 781

| หัวข้อ | สรุป |
|---|---|
| Flower คืออะไร | Web UI ที่ฟัง event จาก Celery worker ผ่าน broker แล้วแสดงผลแบบเรียลไทม์ |
| ควบคุม worker ได้จริงไหม | ได้ผ่านกลไก remote control ของ Celery (revoke, restart pool, rate limit) |
| ใช้แทน Sentry ได้ไหม | ไม่ควรใช้แทนกัน — Flower ดู "ตอนนี้", Sentry ดู "แนวโน้มและ error ย้อนหลัง" |
| ต้องแก้โค้ด task ไหม | ไม่ต้อง — รันเป็นกระบวนการแยกต่างหาก |

---

## ขั้นตอนที่ 782: ติดตั้งและตั้งค่า Flower พร้อม Authentication

### 782.1 ติดตั้ง Flower

```bash
pip install flower
pip freeze | grep flower >> requirements.txt
```

### 782.2 รัน Flower ครั้งแรก

```bash
celery -A config flower --port=5555
```

เปิดเบราว์เซอร์ไปที่ `http://localhost:5555` จะเห็น Dashboard ของ Flower ทันที **แต่
ตอนนี้ยังไม่มี authentication ใด ๆ เลย — ใครก็ตามที่เข้าถึง port 5555 ได้จะเห็นข้อมูล
ทั้งหมด รวมถึง argument ของ task ที่อาจมีข้อมูลอ่อนไหว เช่น email, user ID, หรือแม้แต่
token บางประเภทที่ส่งผ่าน argument โดยไม่ระวัง**

> **คำเตือนสำคัญที่สุดของ Part นี้**: **ห้าม deploy Flower ขึ้น production โดยไม่มี
> authentication เด็ดขาด** นี่คือความผิดพลาดด้าน security ที่พบบ่อยมากในทีมที่เพิ่งเริ่ม
> ใช้ Celery — เปิด Flower ให้เข้าถึงได้จาก public internet โดยไม่ป้องกันอะไรเลย ทำให้
> ใครก็ได้เห็นโครงสร้างงานภายในระบบ, revoke task ของคนอื่น, หรือแม้แต่ shutdown worker
> ทั้งคลัสเตอร์ได้จาก UI โดยตรง

### 782.3 เปิด Basic Authentication

วิธีที่ง่ายและเพียงพอสำหรับทีมขนาดเล็ก-กลาง:

```bash
celery -A config flower \
    --port=5555 \
    --basic-auth=admin:S3cur3P@ssw0rd,ops:An0therP@ss
```

รูปแบบคือ `username:password` คั่นด้วย comma สำหรับหลายบัญชี **ห้าม hardcode รหัสผ่าน
ในคำสั่งตรง ๆ แบบนี้ในสภาพแวดล้อมจริง** ให้อ่านจาก environment variable แทนเสมอ:

```bash
# .env (ไม่ commit เข้า git — ทวนหลักการจาก Part 010)
FLOWER_BASIC_AUTH=admin:S3cur3P@ssw0rd,ops:An0therP@ss
```

```bash
celery -A config flower --port=5555 --basic-auth="$FLOWER_BASIC_AUTH"
```

### 782.4 เปิด OAuth2 ด้วย Google (แนะนำสำหรับองค์กรที่ใช้ Google Workspace)

สำหรับทีมที่มีขนาดใหญ่ขึ้นและใช้ Google Workspace อยู่แล้ว การผูก Flower เข้ากับ
OAuth2 ของ Google ปลอดภัยกว่า Basic Auth มาก เพราะไม่ต้องจัดการรหัสผ่านแยกต่างหาก และ
ใช้ประโยชน์จากระบบ 2FA ของ Google ที่มีอยู่แล้ว:

```bash
celery -A config flower \
    --port=5555 \
    --auth=".*@yourcompany\.com" \
    --oauth2_key="$GOOGLE_OAUTH2_KEY" \
    --oauth2_secret="$GOOGLE_OAUTH2_SECRET" \
    --oauth2_redirect_uri="https://flower.yourcompany.com/login"
```

`--auth` รับ regular expression ของอีเมลที่อนุญาต ทำให้จำกัดการเข้าถึงเฉพาะคนในองค์กร
เท่านั้น แม้จะ login ผ่าน Google ได้ก็ตาม

### 782.5 ตั้งค่าถาวรด้วยไฟล์ config แทนการพิมพ์ flag ยาว ๆ ทุกครั้ง

```python
# flowerconfig.py (วางไว้ที่ root ของโปรเจกต์)
import os

port = 5555
basic_auth = [os.environ["FLOWER_BASIC_AUTH"]]
persistent = True
db = "flower.db"          # เก็บ task history ลงไฟล์ ไม่หายเมื่อ restart Flower
max_tasks = 10000         # จำกัดจำนวน task ที่เก็บใน memory/db ไม่ให้บวมไม่จำกัด
broker_api = os.environ.get("CELERY_BROKER_URL")
```

```bash
celery -A config flower --conf=flowerconfig.py
```

> **ทำไมต้องตั้ง `persistent=True`**: โดย default Flower เก็บ task history ไว้ใน
> หน่วยความจำเท่านั้น ถ้า restart กระบวนการ Flower (เช่นตอน deploy) ประวัติ task ทั้งหมด
> จะหายทันที การเปิด `persistent=True` พร้อม `db=` จะบันทึกลงไฟล์ (ใช้ `shelve` ของ Python
> เบื้องหลัง) ทำให้ดูประวัติย้อนหลังได้แม้ restart Flower ไปแล้ว

### 782.6 Deploy Flower อย่างปลอดภัยด้วย Docker Compose + Nginx Reverse Proxy

ในระบบจริง ไม่ควรเปิด port 5555 ออกสู่ internet โดยตรงแม้จะมี auth แล้วก็ตาม ควรวาง
ไว้หลัง reverse proxy ที่บังคับ HTTPS และจำกัด IP ได้อีกชั้น:

```yaml
# docker-compose.yml (ส่วนที่เกี่ยวข้อง)
services:
  flower:
    build: .
    command: >
      celery -A config flower --conf=flowerconfig.py
    env_file: .env
    depends_on:
      - redis
    expose:
      - "5555"     # expose ให้เฉพาะ network ภายใน ไม่ publish ports ออกนอกเครื่อง
    networks:
      - internal
```

```nginx
# /etc/nginx/sites-available/flower.conf
server {
    listen 443 ssl;
    server_name flower.yourcompany.com;

    ssl_certificate     /etc/letsencrypt/live/flower.yourcompany.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/flower.yourcompany.com/privkey.pem;

    # จำกัดเฉพาะ IP ของ office/VPN เท่านั้น เป็นเกราะป้องกันชั้นที่สอง
    allow 203.0.113.0/24;
    deny all;

    location / {
        proxy_pass http://flower:5555;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";   # Flower ใช้ WebSocket สำหรับ real-time update
    }
}
```

> **สังเกต `Upgrade`/`Connection` header**: Flower ใช้ WebSocket ในการอัปเดตหน้า
> Dashboard แบบเรียลไทม์โดยไม่ต้อง refresh หน้าเอง (คล้ายหลักการเดียวกับ Django Channels
> ที่เรียนใน Part 057/074) การตั้งค่า Nginx เป็น reverse proxy สำหรับ WebSocket ต้องเปิด
> header เหล่านี้เสมอ ไม่เช่นนั้น Dashboard จะค้างไม่อัปเดตอัตโนมัติ

### 782.7 ตารางสรุปขั้นตอนที่ 782

| หัวข้อ | สรุป |
|---|---|
| ติดตั้ง | `pip install flower` |
| รันพื้นฐาน | `celery -A config flower --port=5555` |
| Auth ระดับพื้นฐาน | `--basic-auth=user:pass` อ่านจาก environment variable เสมอ |
| Auth ระดับองค์กร | OAuth2 ผ่าน Google (`--auth`, `--oauth2_key`, `--oauth2_secret`) |
| เก็บ history ข้าม restart | `persistent=True` + `db=flower.db` ในไฟล์ config |
| Deploy จริง | อยู่หลัง reverse proxy + HTTPS + จำกัด IP เสมอ ห้าม expose ตรงสู่ internet |
| กฎเหล็ก | ห้ามรัน Flower โดยไม่มี authentication เด็ดขาดในทุกสภาพแวดล้อมที่เข้าถึงได้จากนอกเครื่อง |

---

## ขั้นตอนที่ 783: อ่าน Dashboard ของ Flower — ประวัติ Task, สถานะ Worker, กราฟต่าง ๆ

### 783.1 เปิดใช้งาน Event ให้ครบก่อนใช้งาน Flower เต็มรูปแบบ

Flower ต้องพึ่ง event ที่ worker กระจายออกมา ตรวจสอบให้แน่ใจว่า worker รันด้วย flag
`-E` (เปิดส่ง event) เสมอ — ถ้าลืม flag นี้ Flower จะเห็นแค่ worker online/offline
แต่จะไม่เห็นรายละเอียด task เลย:

```bash
celery -A config worker -Q high_priority --concurrency=8 -E --hostname=worker-urgent@%h
```

หรือกำหนดถาวรใน settings แทนการพึ่ง flag ทุกครั้ง:

```python
# config/settings.py
CELERY_WORKER_SEND_TASK_EVENTS = True
CELERY_TASK_SEND_SENT_EVENT = True   # ให้เห็นสถานะ PENDING ตั้งแต่ตอนยิงงาน ไม่ใช่แค่ตอน worker รับ
```

### 783.2 แท็บ Workers — สถานะของ Worker แต่ละตัว

หน้าแรกที่ Flower แสดงคือ **Workers tab** ซึ่งแสดงตารางของ worker ทุกตัวที่เชื่อมต่อ
เข้ากับ broker เดียวกัน (รวมทุกกลุ่มที่แยกคิวไว้จากขั้นตอนที่ 756.3):

| คอลัมน์ | ความหมาย |
|---|---|
| Worker Name | ชื่อ worker เช่น `worker-urgent@web-01` (มาจาก `--hostname`) |
| Status | `Online` (สีเขียว) / `Offline` (สีแดง — ไม่ได้รับ heartbeat เกิน timeout) |
| Active | จำนวน task ที่กำลังรันอยู่ตอนนี้ |
| Processed | จำนวน task ทั้งหมดที่ประมวลผลไปแล้วตั้งแต่ worker เริ่มทำงาน |
| Load Average | ค่า load เฉลี่ยของเครื่องที่ worker รันอยู่ (1/5/15 นาที เหมือนคำสั่ง `uptime`) |
| Pool | ประเภท pool (`prefork`, `eventlet`, `gevent`) และจำนวน concurrency |

คลิกที่ชื่อ worker เพื่อดูรายละเอียดเพิ่มเติม เช่น registered tasks ทั้งหมดที่ worker
ตัวนั้นรู้จัก, active queues ที่กำลังฟังอยู่, และค่า config ที่ worker บูตขึ้นมาด้วย
(มีประโยชน์มากเวลา debug ปัญหา "ทำไม config ที่แก้ไม่มีผล" — อาจเป็นเพราะ worker ยังไม่
ถูก restart)

### 783.3 แท็บ Tasks — ประวัติ Task ทั้งหมด

แท็บนี้คือหัวใจของ Flower แสดงตาราง task ทุกตัวที่เคยยิงเข้าระบบ พร้อม filter/search:

| คอลัมน์ | ความหมาย |
|---|---|
| Name | ชื่อ task เต็ม เช่น `blog.tasks.render_report_pdf` |
| UUID | Task ID เฉพาะของการรันครั้งนี้ (ใช้ค้นหาข้าม log ได้ — ทวนหลักการ correlation ID ในขั้นตอนที่ 788) |
| State | `PENDING`, `RECEIVED`, `STARTED`, `SUCCESS`, `FAILURE`, `RETRY`, `REVOKED` |
| Args / Kwargs | argument ที่ส่งเข้า task (ระวังข้อมูลอ่อนไหวตามที่เตือนในขั้นตอนที่ 782.2) |
| Result | ผลลัพธ์ที่ task คืนค่ามา (ถ้ามี result backend) |
| Received / Started / Succeeded | timestamp ของแต่ละ state |
| Runtime | เวลาที่ใช้รันจริง (วินาที) — ใช้เปรียบเทียบหา task ที่ช้าผิดปกติ |
| Worker | worker ตัวไหนเป็นคนรัน task นี้ |

คลิกที่ task ใดตัวหนึ่งเพื่อดูรายละเอียดเต็ม รวมถึง **traceback แบบเต็ม** ถ้า task
ล้มเหลว (เหมือนกับที่เห็นใน terminal log แต่ดูผ่าน UI สะดวกกว่ามาก) และจำนวนครั้งที่
retry ไปแล้วถ้า task นั้นตั้ง `autoretry_for` ไว้ (ทวนจาก Part 075 ขั้นตอนที่ 746)

### 783.4 แท็บ Monitor — กราฟภาพรวมระบบ

แท็บนี้แสดงกราฟเรียลไทม์ 2 ส่วนหลัก:

1. **Task Rate**: จำนวน task ต่อวินาทีที่ succeeded/failed/retried แยกสี ใช้ดู
   แนวโน้มโหลดของระบบ เช่น เห็นยอด spike ตอนที่ Beat ยิง `send_weekly_digest_email`
   ทุกวันจันทร์ 7 โมงเช้า (ทวนจาก Part 076 ขั้นตอนที่ 751)
2. **Number of Workers**: กราฟจำนวน worker online ตามเวลา ใช้สังเกตว่า worker
   crash/restart บ่อยผิดปกติหรือไม่ (ถ้ากราฟมีรอยหยักลง-ขึ้นถี่ ๆ แปลว่ามีปัญหา
   worker ไม่เสถียร ต้องตรวจสอบ memory leak หรือ `--max-tasks-per-child`)

### 783.5 Remote Control — ควบคุม Worker จาก UI โดยตรง

Flower อาศัยกลไก **remote control** ของ Celery (คำสั่งเดียวกับที่ใช้ผ่าน
`celery -A config control ...` ใน CLI) ทำให้ทำสิ่งต่อไปนี้ได้จากปุ่มใน UI โดยไม่ต้อง
SSH เข้าเครื่อง:

| การกระทำ | ใช้เมื่อ |
|---|---|
| **Revoke** | ยกเลิก task ที่ยังไม่เริ่มรัน (อยู่ใน state `PENDING`) — เช่น ยิงอีเมลผิดคนไปแล้วอยากยกเลิกก่อนส่งจริง |
| **Terminate** | หยุด task ที่ **กำลังรันอยู่** ทันที (ส่ง signal ไปขัดจังหวะ) — ใช้ระวัง เพราะอาจทำให้ task ค้างครึ่งทาง ถ้าไม่ idempotent (ทวนขั้นตอนที่ 758) |
| **Restart Pool** | สั่งให้ worker restart child process ทั้งหมด (มีประโยชน์เมื่อสงสัยว่า worker มี memory leak สะสม) |
| **Rate Limit** | ปรับ rate limit ของ task ชนิดหนึ่งแบบ real-time เช่น จำกัด `send_password_reset_email` ไม่ให้เกิน 10 ครั้ง/นาที ระหว่างเกิดเหตุการณ์ผิดปกติ (ถูกโจมตี) โดยไม่ต้อง deploy โค้ดใหม่ |
| **Shutdown** | สั่งปิด worker ตัวนั้นทั้งหมด (ใช้ระวังมาก โดยเฉพาะใน production ที่มี auto-restart จาก process manager) |

```bash
# ตัวอย่างคำสั่งเดียวกันแบบ CLI (ไม่ผ่าน UI) — Flower เรียกกลไกเบื้องหลังแบบนี้เหมือนกัน
celery -A config control revoke <task-id>
celery -A config control rate_limit blog.tasks.render_report_pdf 10/m
```

### 783.6 ใช้ REST API ของ Flower เขียน Health Check Script

Flower เปิด REST API ให้เรียกได้โดยไม่ต้องเปิดเบราว์เซอร์ ใช้ประโยชน์ในการเขียน
สคริปต์ตรวจสุขภาพระบบอัตโนมัติ (เช่นเรียกจาก Kubernetes readiness probe หรือ cron
ตรวจสอบภายนอก):

```python
# scripts/check_celery_health.py
import sys
import requests

FLOWER_URL = "http://flower:5555"
AUTH = ("admin", "S3cur3P@ssw0rd")


def check_workers_online(min_workers=2):
    resp = requests.get(f"{FLOWER_URL}/api/workers", auth=AUTH, timeout=5)
    resp.raise_for_status()
    workers = resp.json()
    online = [w for w, info in workers.items() if info.get("status")]

    if len(online) < min_workers:
        print(f"เตือน: มี worker online แค่ {len(online)} ตัว (ต้องการอย่างน้อย {min_workers})")
        return False
    print(f"ปกติ: worker online {len(online)} ตัว")
    return True


if __name__ == "__main__":
    sys.exit(0 if check_workers_online() else 1)
```

### 783.7 ตารางสรุปขั้นตอนที่ 783

| หัวข้อ | สรุป |
|---|---|
| ก่อนใช้งาน | ต้องเปิด `-E` หรือ `CELERY_WORKER_SEND_TASK_EVENTS = True` ที่ worker เสมอ |
| แท็บ Workers | สถานะ online/offline, active, processed, load average ของแต่ละ worker |
| แท็บ Tasks | ประวัติ task ทุกตัว ค้นหา/filter ตาม state, name, worker ได้ |
| แท็บ Monitor | กราฟ task rate และจำนวน worker online ตามเวลา |
| Remote Control | Revoke, Terminate, Restart Pool, Rate Limit, Shutdown — ทำได้จาก UI โดยตรง |
| REST API | `/api/workers`, `/api/tasks` — ใช้เขียน health check script อัตโนมัติได้ |

---

## ขั้นตอนที่ 784: `django-rq` เป็นทางเลือกแทน Celery

### 784.1 ทำไมต้องรู้จักทางเลือกอื่นทั้งที่เพิ่งลงทุนเรียน Celery มาเต็ม 3 Part

Celery เป็นเครื่องมือที่ทรงพลังมาก แต่ **ความทรงพลังมาพร้อมความซับซ้อน**: ต้องมี
broker แยก, ต้องเข้าใจ serialization, result backend, routing, chain/group/chord,
retry policy ฯลฯ สำหรับหลายทีม (โดยเฉพาะทีมเล็ก หรือโปรเจกต์ที่ต้องการแค่ "รันงาน
เบื้องหลังง่าย ๆ ไม่กี่แบบ") ความซับซ้อนนี้อาจเกินความจำเป็น

**RQ (Redis Queue)** คือไลบรารีคิวงานของ Python ที่ออกแบบมาให้ **เรียบง่ายที่สุดเท่าที่
จะทำได้** โดยใช้ **Redis อย่างเดียว** เป็นทั้ง broker และที่เก็บผลลัพธ์ ไม่มีแนวคิดเรื่อง
broker หลายชนิด, ไม่มี AMQP, ไม่มี serializer ให้เลือก — เขียนฟังก์ชัน Python ธรรมดา
แล้วส่งเข้าคิวได้เลย

**`django-rq`** คือ package ที่ห่อ RQ เข้ากับ Django ให้สะดวกยิ่งขึ้น เพิ่ม
management command, การตั้งค่าผ่าน `settings.py`, และหน้า dashboard ในตัว

### 784.2 ติดตั้งและตั้งค่า

```bash
pip install django-rq
pip freeze | grep -E "^(rq|django-rq)" >> requirements.txt
```

```python
# config/settings.py
INSTALLED_APPS = [
    ...
    'django_rq',
]

RQ_QUEUES = {
    'high': {
        'HOST': 'localhost',
        'PORT': 6379,
        'DB': 6,                    # แยก Redis DB index จาก Celery (DB 4-5) ทวนหลักการ Part 069
        'DEFAULT_TIMEOUT': 500,
    },
    'default': {
        'HOST': 'localhost',
        'PORT': 6379,
        'DB': 6,
        'DEFAULT_TIMEOUT': 360,
    },
    'low': {
        'HOST': 'localhost',
        'PORT': 6379,
        'DB': 6,
        'DEFAULT_TIMEOUT': 360,
    },
}
```

> **สังเกต DB index อีกครั้ง**: ทวนหลักการแยก Redis DB index จาก Part 069 ขั้นตอนที่
> 681.5 และ Part 076 ขั้นตอนที่ 751.1 — ถ้าใช้ RQ คู่กับ Celery ในระบบเดียวกัน (เช่น
> ระหว่างค่อย ๆ ย้ายจาก Celery มา RQ) ต้องแยก DB index ให้ชัดเจนเสมอ (Celery ใช้ DB 4-5,
> RQ ใช้ DB 6 ในตัวอย่างนี้) ไม่เช่นนั้นข้อมูลจะปนกันจนตรวจสอบไม่ออกว่างานไหนเป็นของระบบไหน

### 784.3 เขียน Job แรกด้วย django-rq

ต่างจาก Celery ที่ต้อง `@shared_task` และมี `config/celery.py` แยก RQ ไม่บังคับ
โครงสร้างพิเศษใด ๆ — **ฟังก์ชัน Python ธรรมดาก็เป็น job ได้ทันที**:

```python
# blog/jobs.py
import logging

from django.core.mail import send_mail

logger = logging.getLogger(__name__)


def send_comment_notification_email(comment_id):
    """Job ธรรมดา — ไม่ต้องมี decorator พิเศษใด ๆ เพื่อให้เป็น RQ job ได้"""
    from .models import Comment

    try:
        comment = Comment.objects.select_related('post', 'post__author').get(pk=comment_id)
    except Comment.DoesNotExist:
        logger.warning('Comment id=%s ถูกลบไปก่อนที่ job จะรัน ข้าม', comment_id)
        return

    send_mail(
        subject=f'มีความคิดเห็นใหม่ในโพสต์ "{comment.post.title}"',
        message=f'{comment.author_name}: {comment.content}',
        from_email='noreply@example.com',
        recipient_list=[comment.post.author.email],
    )
    logger.info('ส่งอีเมลแจ้งเตือนคอมเมนต์ id=%s สำเร็จ', comment_id)
```

ยิง job เข้าคิวจาก view หรือ signal ด้วย `django_rq.enqueue()`:

```python
# blog/views.py
import django_rq

from .jobs import send_comment_notification_email


def add_comment(request, post_id):
    # ... บันทึกคอมเมนต์ตามปกติ ...
    comment = form.save()

    queue = django_rq.get_queue('default')
    queue.enqueue(send_comment_notification_email, comment.id)

    return redirect('post_detail', pk=post_id)
```

### 784.4 ทางเลือกที่กระชับกว่า: `@job` Decorator

`django-rq` มี decorator `@job` ให้เรียกแบบสั้นคล้าย `.delay()` ของ Celery ถ้าต้องการ
ความคุ้นเคยแบบเดียวกัน:

```python
# accounts/jobs.py
from django_rq import job


@job('high')   # ระบุชื่อคิวตรง ๆ ที่ decorator ได้เลย
def send_password_reset_email(user_id):
    from django.contrib.auth import get_user_model
    from django.core.mail import send_mail

    User = get_user_model()
    user = User.objects.get(pk=user_id)
    send_mail(
        subject='รีเซ็ตรหัสผ่านของคุณ',
        message='คลิกลิงก์เพื่อรีเซ็ตรหัสผ่าน: https://example.com/reset/...',
        from_email='noreply@example.com',
        recipient_list=[user.email],
    )
```

```python
# accounts/views.py
from .jobs import send_password_reset_email

send_password_reset_email.delay(user.id)   # เหมือน .delay() ของ Celery ทุกประการ
```

### 784.5 รัน RQ Worker

```bash
# รัน worker ที่ฟังคิว high ก่อน แล้วค่อย default แล้วค่อย low (ลำดับ = priority)
python manage.py rqworker high default low
```

**ข้อสังเกตสำคัญเรื่อง priority ของ RQ**: RQ ใช้หลักการง่ายมาก — **worker จะเช็คคิว
ตามลำดับที่ระบุใน argument เสมอ** ถ้าคิว `high` มีงานค้างอยู่ worker จะหยิบจากคิวนั้น
ก่อนเสมอ ไม่สนใจคิว `default`/`low` เลยจนกว่า `high` จะว่าง — นี่คือกลไก priority
queue ที่ **เข้าใจง่ายกว่า** priority field ของ Celery/RabbitMQ ที่กล่าวถึงในขั้นตอนที่
756.4 มาก แลกมาด้วยความยืดหยุ่นที่น้อยกว่า (ปรับ priority runtime ไม่ได้ ต้องกำหนดตอน
สั่งรัน worker เท่านั้น)

### 784.6 การจัดตารางเวลา (Scheduling) ด้วย `rq-scheduler`

RQ เองไม่มีแนวคิดแบบ Celery Beat ในตัว ต้องติดตั้ง package เสริม `rq-scheduler`:

```bash
pip install rq-scheduler
```

```python
# blog/services/schedule_jobs.py
from datetime import datetime, timedelta

from django_rq import get_scheduler

from blog.jobs import sync_post_view_stats_rq


def schedule_recurring_jobs():
    scheduler = get_scheduler('default')

    # ล้าง schedule เก่าก่อน (ป้องกันสร้างซ้ำถ้ารันฟังก์ชันนี้หลายครั้ง)
    for job in scheduler.get_jobs():
        if job.func_name == 'blog.jobs.sync_post_view_stats_rq':
            scheduler.cancel(job)

    scheduler.schedule(
        scheduled_time=datetime.utcnow(),
        func=sync_post_view_stats_rq,
        interval=300,     # ทุก 300 วินาที เทียบเท่ากับ CELERY_BEAT_SCHEDULE ในขั้นตอนที่ 751.3
        repeat=None,       # None = รันซ้ำไม่มีที่สิ้นสุด
    )
```

```bash
# ต้องรัน scheduler เป็นกระบวนการแยกต่างหาก คล้าย celery beat
python manage.py rqscheduler
```

### 784.7 Job Dependency: ทางเลือกที่เรียบง่ายกว่า Chain

RQ รองรับการต่อ job แบบพื้นฐานผ่าน argument `depends_on` — ไม่ทรงพลังเท่า `chain()`
ของ Celery (ไม่มี Group/Chord ในตัว) แต่เพียงพอสำหรับ pipeline ง่าย ๆ 2-3 ขั้นตอน:

```python
queue = django_rq.get_queue('default')

job1 = queue.enqueue(gather_report_data_rq)
job2 = queue.enqueue(render_report_pdf_rq, depends_on=job1)
job3 = queue.enqueue(email_report_to_admin_rq, depends_on=job2)
```

> **ข้อจำกัดที่ต้องรู้**: `depends_on` ของ RQ **ไม่ส่งผลลัพธ์ของ job ก่อนหน้าให้ job
> ถัดไปโดยอัตโนมัติ** เหมือนที่ `chain()` ของ Celery ทำ (ทวนขั้นตอนที่ 753.3) ต้องดึงผล
> ลัพธ์เองผ่าน `job1.result` ภายใน job ถัดไป หรือบันทึกผลลัพธ์ลงฐานข้อมูล/cache แล้วให้
> job ถัดไปอ่านเอา — นี่คือหนึ่งในความแตกต่างสำคัญที่ทำให้ RQ "เรียบง่ายกว่าแต่ก็มีน้ำหนัก
> ต้องเขียนโค้ดเชื่อมเองมากกว่า" เมื่อ pipeline ซับซ้อนขึ้น

### 784.8 ตารางสรุปขั้นตอนที่ 784

| หัวข้อ | สรุป |
|---|---|
| RQ คืออะไร | ไลบรารีคิวงานที่ใช้ Redis อย่างเดียว เรียบง่ายกว่า Celery มาก |
| Job คืออะไร | ฟังก์ชัน Python ธรรมดา ไม่ต้องมี decorator พิเศษก็ใช้ได้ |
| ยิง job เข้าคิว | `django_rq.get_queue('name').enqueue(func, args)` หรือ `@job` + `.delay()` |
| รัน worker | `python manage.py rqworker high default low` (ลำดับ = priority) |
| Scheduling | ต้องใช้ package เสริม `rq-scheduler` (ไม่มี Beat ในตัว) |
| Workflow | มีแค่ `depends_on` พื้นฐาน ไม่มี chain/group/chord ในตัว |

---

## ขั้นตอนที่ 785: `RQ Dashboard` สำหรับ Monitor django-rq

### 785.1 Dashboard ในตัวของ django-rq

`django-rq` มาพร้อม Dashboard ในตัวโดยไม่ต้องติดตั้งอะไรเพิ่ม แค่เพิ่ม URL:

```python
# config/urls.py
from django.contrib import admin
from django.contrib.admin.views.decorators import staff_member_required
from django.urls import include, path

urlpatterns = [
    path('admin/', admin.site.urls),
    path('django-rq/', include('django_rq.urls')),
]
```

**สำคัญมาก**: `django_rq.urls` **ไม่มีการป้องกันในตัวโดย default** ต้องครอบด้วย
`staff_member_required` เองเสมอ (คล้ายหลักการเดียวกับที่ Flower ต้องมี auth ในขั้นตอนที่
782.2):

```python
# config/urls.py — วิธีที่ถูกต้อง: ครอบ decorator ก่อนแล้วค่อย include
from django.contrib.admin.views.decorators import staff_member_required
from django.urls import include, path
from django.views.decorators.cache import never_cache

urlpatterns = [
    path(
        'django-rq/',
        staff_member_required(never_cache(include('django_rq.urls'))),
    ),
]
```

> **หมายเหตุเชิงเทคนิค**: การครอบ `include()` ตรง ๆ แบบนี้ใช้ได้เพราะ `django_rq.urls`
> คืนค่าเป็น URLconf resolver ธรรมดา ถ้าเวอร์ชัน Django ของคุณไม่รองรับรูปแบบนี้ ทางเลือก
> ที่ปลอดภัยกว่าคือใช้ Nginx จำกัด path `/django-rq/` ให้เฉพาะ IP ภายในองค์กรเหมือนที่ทำกับ
> Flower ในขั้นตอนที่ 782.6 เป็นการป้องกันซ้อนอีกชั้น

เข้าถึงได้ที่ `/django-rq/` (ต้อง login เป็น staff ก่อน) จะเห็น:

| ส่วน | รายละเอียด |
|---|---|
| Queues | รายชื่อคิวทั้งหมด (`high`, `default`, `low`) พร้อมจำนวนงานที่ค้างอยู่ในแต่ละคิว |
| Workers | worker ที่กำลังทำงานอยู่ พร้อม job ปัจจุบันที่กำลังรัน |
| Jobs by Status | แยกตาม `Queued`, `Started`, `Finished`, `Failed`, `Deferred`, `Scheduled` |
| Requeue | ปุ่ม requeue job ที่ล้มเหลว กลับเข้าคิวใหม่ได้ทันทีจาก UI |

### 785.2 เพิ่มลิงก์เข้า Django Admin โดยตรง

```python
# config/settings.py
RQ_SHOW_ADMIN_LINK = True
```

เมื่อเปิดค่านี้ Django Admin index page (`/admin/`) จะมีลิงก์ "Django RQ" ปรากฏขึ้นเอง
ให้คลิกเข้าไปดู Dashboard ได้โดยไม่ต้องจำ URL แยก — สะดวกมากสำหรับทีมที่ใช้ Django
Admin เป็นศูนย์กลางการจัดการระบบอยู่แล้ว

### 785.3 ทางเลือกแบบ Standalone: `rq-dashboard`

ถ้าไม่ได้ใช้ Django หรือต้องการ Dashboard ที่แยกอิสระจากแอป Django (เช่น ทีม
infrastructure อยากมี dashboard กลางสำหรับดู RQ ของหลายโปรเจกต์พร้อมกัน) มี package
แยกต่างหากชื่อ **`rq-dashboard`** (สร้างด้วย Flask ไม่เกี่ยวกับ Django เลย):

```bash
pip install rq-dashboard
rq-dashboard --redis-url redis://localhost:6379/6 --port 9181
```

เข้าถึงได้ที่ `http://localhost:9181` แสดงข้อมูลชุดเดียวกับ Dashboard ของ `django-rq`
(เพราะอ่านจาก Redis เดียวกัน) แต่**ไม่มีระบบ authentication ในตัวเลย** ต้องวางหลัง
reverse proxy ที่มี auth เสมอเหมือนกับ Flower

### 785.4 ตารางเปรียบเทียบวิธี Monitor RQ ทั้งสองแบบ

| ประเด็น | `django-rq` Dashboard ในตัว | `rq-dashboard` (Standalone) |
|---|---|---|
| ต้องติดตั้งเพิ่มไหม | ไม่ต้อง (มากับ django-rq อยู่แล้ว) | ต้องติดตั้งแยก |
| Authentication | ใช้ระบบ login ของ Django (`staff_member_required`) ได้ทันที | ไม่มีในตัว ต้องพึ่ง reverse proxy |
| ผูกกับ Django Admin | ✅ ผ่าน `RQ_SHOW_ADMIN_LINK` | ❌ ไม่เกี่ยวข้องกับ Django เลย |
| เหมาะกับ | โปรเจกต์ Django ทั่วไปที่ใช้ django-rq อยู่แล้ว | ทีมที่ต้องการ dashboard กลางข้ามหลายโปรเจกต์/ภาษา |

### 785.5 ตารางสรุปขั้นตอนที่ 785

| หัวข้อ | สรุป |
|---|---|
| Dashboard ในตัว | `django_rq.urls` — ต้องครอบ `staff_member_required` เองเสมอ |
| ลิงก์จาก Admin | `RQ_SHOW_ADMIN_LINK = True` |
| Standalone | `rq-dashboard` (Flask) — ไม่มี auth ในตัว ต้องพึ่ง reverse proxy |
| ข้อมูลที่แสดง | คิว, worker, job แยกตามสถานะ, ปุ่ม requeue job ที่ล้มเหลว |

---

## ขั้นตอนที่ 786: ตารางเปรียบเทียบ Celery vs django-RQ — เมื่อไหร่ควรเลือกอะไร

### 786.1 ตารางเปรียบเทียบแบบละเอียด

| มิติ | Celery | django-RQ |
|---|---|---|
| Broker ที่รองรับ | Redis, RabbitMQ, Amazon SQS และอื่น ๆ อีกมาก | **Redis เท่านั้น** |
| ความซับซ้อนในการติดตั้ง | สูงกว่า (ต้องมี `celery.py`, ตั้ง serializer, result backend แยก) | ต่ำมาก (แค่ `RQ_QUEUES` ใน settings) |
| การตั้งเวลา (Scheduling) | มี Celery Beat ในตัว + `django-celery-beat` จัดการผ่าน Admin | ต้องพึ่ง package เสริม `rq-scheduler` |
| Workflow ซับซ้อน (Chain/Group/Chord) | ✅ มีครบ รองรับ pipeline ที่ซับซ้อนมาก (Part 076) | ⚠️ มีแค่ `depends_on` พื้นฐาน ไม่มี Group/Chord ในตัว |
| Task Routing/Priority | ยืดหยุ่นสูง แยกคิวได้ไม่จำกัด ปรับ runtime ได้ผ่าน remote control | เรียบง่าย: ลำดับคิวที่ส่งให้ worker ตอนสั่งรัน กำหนดล่วงหน้าเท่านั้น |
| Retry Policy | ละเอียดมาก (`autoretry_for`, backoff, jitter, max_retries ต่อ exception) | มีพื้นฐาน (`@job(retry=Retry(max=3))`) แต่ยืดหยุ่นน้อยกว่า |
| Monitoring | Flower — ฟีเจอร์ครบ, remote control เต็มรูปแบบ | RQ Dashboard/`django-rq` — เรียบง่าย อ่านง่าย แต่ควบคุมได้น้อยกว่า |
| Serialization | เลือกได้ (JSON แนะนำ, มี pickle แต่ไม่ปลอดภัย — ทวน Part 076 ขั้นตอนที่ 747) | ใช้ pickle เป็นค่าเริ่มต้นภายใน (ต้องระวังเรื่อง security เมื่อ argument มาจากภายนอก) |
| Concurrency Model | Prefork, eventlet, gevent, threads — เลือกได้ตามงาน | Fork process ต่อ job (เรียบง่ายตรงไปตรงมา) |
| Learning Curve | สูงกว่าอย่างชัดเจน | ต่ำมาก เขียนฟังก์ชัน Python ธรรมดาแล้วใช้ได้เลย |
| Community/Ecosystem | ใหญ่กว่ามาก ใช้ในระบบระดับ enterprise ทั่วโลก | เล็กกว่า แต่เพียงพอสำหรับ use case ทั่วไป |
| Overhead ต่อ Job | สูงกว่าเล็กน้อยจาก feature ที่มากกว่า | ต่ำกว่า เพราะเรียบง่ายกว่ามาก |

### 786.2 เกณฑ์การตัดสินใจแบบใช้งานได้จริง

```
เริ่มต้น: มีแค่ Redis อยู่แล้ว และงานเบื้องหลังไม่ซับซ้อนมาก (ส่งอีเมล, resize รูป)?
    │
    ├─ ใช่ → เริ่มด้วย django-rq (เรียบง่าย ตั้งค่าเร็ว ทีมเล็กดูแลง่าย)
    │
    └─ ไม่ใช่ → พิจารณาต่อ:
         │
         ├─ ต้องการ pipeline ซับซ้อน (chain/group/chord), routing หลายระดับ,
         │  retry policy ละเอียด, หรือมีแผนใช้ RabbitMQ/SQS ในอนาคต?
         │      │
         │      └─ ใช่ → ใช้ Celery ตั้งแต่แรก (ทวนทั้งหมดที่เรียนใน Part 075-078)
         │
         └─ ทีมมีขนาดใหญ่ ต้องการ monitoring/control ระดับสูงสุด (Flower) และ
            อยากได้ ecosystem ที่ community ใหญ่ที่สุดรองรับระยะยาว?
                │
                └─ ใช่ → Celery
```

**คำแนะนำระดับมืออาชีพของหลักสูตรนี้**: หลายทีมในโลกจริงเริ่มต้นด้วย django-rq
เพราะเรียบง่ายและ ship ฟีเจอร์ได้เร็ว แล้ว **ย้ายมา Celery ภายหลัง** เมื่อความต้องการ
ของระบบซับซ้อนขึ้น (ต้องการ chain/chord จริงจัง, ต้องการ routing หลายระดับ, หรือทีม
โตขึ้นจนต้องการ observability ระดับ Flower) การเริ่มด้วยเครื่องมือที่ตรงกับความซับซ้อน
ของปัญหา ณ ตอนนั้น **ดีกว่าการใช้เครื่องมือที่ทรงพลังเกินความจำเป็นตั้งแต่วันแรก** เสมอ
— นี่คือหลักการ "YAGNI" (You Aren't Gonna Need It) ที่ใช้ได้กับการเลือกเครื่องมือด้วย
ไม่ใช่แค่การออกแบบโค้ด

### 786.3 ตารางสรุปขั้นตอนที่ 786

| หัวข้อ | สรุป |
|---|---|
| เลือก django-RQ เมื่อ | ระบบเล็ก-กลาง, มี Redis อยู่แล้ว, งานไม่ซับซ้อน, ต้องการเริ่มเร็ว |
| เลือก Celery เมื่อ | ต้องการ chain/group/chord, routing ซับซ้อน, retry ละเอียด, หรือใช้ RabbitMQ/SQS |
| ย้ายจาก RQ ไป Celery ได้ไหม | ได้ และเป็นเส้นทางที่พบบ่อยในทีมจริงเมื่อระบบโตขึ้น |
| หลักการเลือกเครื่องมือ | ใช้เครื่องมือที่ตรงกับความซับซ้อนของปัญหา ณ ตอนนี้ ไม่ใช่ที่ทรงพลังที่สุดเสมอไป |

---

## ขั้นตอนที่ 787: การแจ้งเตือนเมื่อ Background Job ล้มเหลว (เชื่อม Sentry)

### 787.1 ทวนปัญหา: Flower/RQ Dashboard บอกแค่ "ตอนนี้" ไม่มีใครมาดูตลอดเวลา

Flower และ RQ Dashboard ยอดเยี่ยมสำหรับดูสถานะแบบเรียลไทม์ **แต่ไม่มีใครนั่งจ้องหน้าจอ
ตลอด 24 ชั่วโมง** ถ้า task ล้มเหลวตอนตี 3 และไม่มีใครเห็น Dashboard ในตอนนั้น ปัญหาจะ
ถูกค้นพบก็ต่อเมื่อผู้ใช้มาร้องเรียนในวันถัดไป — สายเกินไปสำหรับหลายกรณี (เช่น การชำระเงิน
ล้มเหลวเงียบ ๆ) ทางแก้คือเชื่อม background job เข้ากับ **Sentry** (ที่ติดตั้งไปแล้วใน
Part 066 ขั้นตอนที่ 657) เพื่อให้ error ทุกตัวถูกจับและแจ้งเตือนทันทีโดยอัตโนมัติ

### 787.2 ติดตั้ง Sentry Integration สำหรับทั้ง Celery และ django-RQ

```bash
pip install --upgrade sentry-sdk
```

```python
# config/settings/production.py
import sentry_sdk
from sentry_sdk.integrations.django import DjangoIntegration
from sentry_sdk.integrations.celery import CeleryIntegration
from sentry_sdk.integrations.rq import RqIntegration

sentry_sdk.init(
    dsn=env("SENTRY_DSN"),
    integrations=[
        DjangoIntegration(),
        # monitor_beat_tasks=True เปิด Sentry Crons — เฝ้าดูว่า periodic task
        # ที่มี @monitor decorator (ขั้นตอนที่ 787.4) รันตรงเวลาหรือไม่
        CeleryIntegration(monitor_beat_tasks=True),
        RqIntegration(),
    ],
    traces_sample_rate=0.2,
    send_default_pii=False,
    environment=env("ENVIRONMENT", default="production"),
)
```

เพียงเท่านี้ **Sentry จะจับ exception ที่เกิดขึ้นใน Celery task และ RQ job โดย
อัตโนมัติทันที** ไม่ต้องเขียนโค้ดครอบ `try/except` เพิ่มเลย — เหมือนกับที่
`DjangoIntegration()` จับ exception จาก view โดยอัตโนมัติที่เรียนมาก่อนหน้านี้

### 787.3 เพิ่ม Context เฉพาะทางด้วย `on_failure` Hook (ทวนจาก Part 075 ขั้นตอนที่ 749)

แม้ Sentry จะจับ exception อัตโนมัติแล้ว บางครั้งอยากแนบข้อมูลเพิ่มเติมที่เฉพาะเจาะจง
กับ business logic เข้าไปด้วย เช่น "task นี้เกี่ยวกับคำสั่งซื้อไหน" เพื่อค้นหาได้ง่ายขึ้น
ใน Sentry:

```python
# config/celery.py หรือ common/tasks.py
import logging

from celery import Task
import sentry_sdk

logger = logging.getLogger(__name__)


class ObservableTask(Task):
    """Base Task class ที่แนบ context เพิ่มเติมให้ Sentry ทุกครั้งที่ task ล้มเหลว"""

    def on_failure(self, exc, task_id, args, kwargs, einfo):
        sentry_sdk.set_context("celery_task", {
            "task_id": task_id,
            "task_name": self.name,
            "args": args,
            "kwargs": kwargs,
            "retries": self.request.retries,
        })
        sentry_sdk.set_tag("celery_queue", self.request.delivery_info.get("routing_key", "unknown"))
        sentry_sdk.capture_exception(exc)

        logger.error(
            'Task ล้มเหลว: %s (id=%s, retries=%d) — %s',
            self.name, task_id, self.request.retries, exc,
        )
        super().on_failure(exc, task_id, args, kwargs, einfo)
```

```python
# payments/tasks.py
from celery import shared_task

from common.tasks import ObservableTask


@shared_task(base=ObservableTask, bind=True, name='payments.tasks.charge_customer_payment')
def charge_customer_payment(self, order_id, amount):
    ...
```

### 787.4 Sentry Crons: เฝ้าดู Periodic Task ว่า "ไม่ได้รัน" ด้วย ไม่ใช่แค่ "รันแล้วพัง"

จุดที่หลายทีมมองข้ามคือ **Celery Beat ที่หยุดทำงานเงียบ ๆ** (เช่น process ตายไปแต่
ไม่มีใครสังเกต) จะไม่มี exception ให้ Sentry จับเลย เพราะ **ไม่มีอะไรรันเลย** — เงียบ
สนิท ไม่ error ไม่ log ทางแก้คือ **Sentry Crons** ซึ่งใช้หลักการ "check-in": task ต้อง
รายงานตัวเข้า Sentry ทุกครั้งที่รัน ถ้า Sentry ไม่เห็นการรายงานตัวตามเวลาที่คาดไว้
จะแจ้งเตือนทันที (ตรงข้ามกับการรอ error):

```python
# blog/tasks.py
from celery import shared_task
from sentry_sdk.crons import monitor

from common.tasks import ObservableTask


@shared_task(base=ObservableTask, name='blog.tasks.sync_post_view_stats')
@monitor(monitor_slug='sync-post-view-stats')   # ต้องสร้าง monitor slug นี้ใน Sentry dashboard ก่อน
def sync_post_view_stats():
    ...
```

เมื่อตั้งค่านี้ Sentry จะรู้ schedule ที่คาดไว้ (ทุก 5 นาที ตาม `CELERY_BEAT_SCHEDULE`
ในขั้นตอนที่ 751.5) และแจ้งเตือนทันทีถ้า task ไม่รันตามเวลาที่คาด — ครอบคลุมทั้งกรณี
"รันแล้วพัง" (จับผ่าน `CeleryIntegration` ปกติ) **และ** "ไม่รันเลย" (จับผ่าน Sentry
Crons) ซึ่งเป็นสถานการณ์ที่อันตรายกว่าและตรวจจับยากกว่ามาก

### 787.5 แจ้งเตือนช่องทางเสริม: ส่งเข้า Slack สำหรับ Task สำคัญที่สุด

สำหรับ task ที่สำคัญที่สุดในระบบ (เช่น การชำระเงิน) การรอดู Sentry dashboard อาจช้า
เกินไป ควรส่งแจ้งเตือนเข้า Slack ทันทีด้วย เพื่อให้ทีมเห็นในช่องทางที่เปิดอยู่ตลอดเวลา:

```python
# common/alerts.py
import logging

import requests
from django.conf import settings

logger = logging.getLogger(__name__)


def notify_slack_critical_failure(task_name, task_id, exc):
    """ส่งแจ้งเตือนเข้า Slack เฉพาะ task ที่สำคัญมากจริง ๆ เท่านั้น ไม่ใช้กับทุก task"""
    webhook_url = settings.SLACK_ALERTS_WEBHOOK_URL
    if not webhook_url:
        return

    try:
        requests.post(
            webhook_url,
            json={
                "text": (
                    f":rotating_light: *Background Task ล้มเหลว*\n"
                    f"Task: `{task_name}`\nTask ID: `{task_id}`\nError: `{exc}`"
                )
            },
            timeout=5,
        )
    except requests.RequestException:
        # ห้ามให้การส่ง Slack ล้มเหลวไปทำให้ task หลักพังซ้ำซ้อน — log ไว้เฉย ๆ พอ
        logger.exception('ส่งแจ้งเตือน Slack ไม่สำเร็จ สำหรับ task %s', task_name)
```

```python
# payments/tasks.py
class PaymentTask(ObservableTask):
    def on_failure(self, exc, task_id, args, kwargs, einfo):
        super().on_failure(exc, task_id, args, kwargs, einfo)
        notify_slack_critical_failure(self.name, task_id, exc)
```

> **หลักการสำคัญ**: การแจ้งเตือนเสริม (Slack) ต้องไม่ทำให้ระบบหลักพังซ้อน ถ้าการยิง
> webhook ล้มเหลว (เช่น Slack ล่ม) ต้อง **จับ exception ไว้เอง** และ log ไว้เฉย ๆ ห้าม
> ปล่อยให้ error หลุดออกไปจน `on_failure` เองพังซ้ำ ไม่เช่นนั้นจะเกิด error ซ้อน error
> ที่วุ่นวายกว่าเดิม

### 787.6 ตารางสรุปขั้นตอนที่ 787

| หัวข้อ | สรุป |
|---|---|
| จับ error อัตโนมัติ | `CeleryIntegration()` และ `RqIntegration()` ใน `sentry_sdk.init()` |
| เพิ่ม context เฉพาะทาง | Custom `Task` class override `on_failure` แนบ context ก่อน `capture_exception` |
| ตรวจจับ "ไม่รันเลย" | Sentry Crons ผ่าน `@monitor(monitor_slug=...)` — ต่างจาก error tracking ปกติ |
| แจ้งเตือนเร่งด่วนสุด | ส่งเข้า Slack เฉพาะ task ที่สำคัญมากจริง ๆ และห้ามให้ Slack ล้มเหลวไปพัง task หลัก |

---

## ขั้นตอนที่ 788: Best Practice การทำ Logging สำหรับ Background Task

### 788.1 ทำไม Logging ของ Background Task ต่างจาก Logging ของ Web View

ใน Django view ปกติ เรารู้ context ครบ: request, user, path — แต่ **background task
ไม่มี request object** และรันแยกกระบวนการจาก web server เมื่อผู้ใช้รายงานปัญหาว่า
"อีเมลไม่มา" การไล่ log เพื่อหาว่า task ตัวไหนที่เกี่ยวข้องกับ request ของผู้ใช้คนนั้น
เป็นเรื่องยากมาก **ถ้าไม่วางโครงสร้าง log ให้เชื่อมกันได้ตั้งแต่แรก**

### 788.2 หลักการที่ 1: ผูก Task ID เข้ากับทุกบรรทัด Log โดยอัตโนมัติ

ใช้ `logging.Filter` แนบ `task_id`/`task_name` เข้ากับทุก log record โดยอัตโนมัติ
โดยไม่ต้องพิมพ์ซ้ำในทุกจุดที่เขียน `logger.info(...)`:

```python
# common/logging_filters.py
import logging

from celery._state import get_current_task


class CeleryTaskContextFilter(logging.Filter):
    """แนบ task_id/task_name เข้าทุก log record อัตโนมัติเมื่อรันอยู่ภายใน Celery task"""

    def filter(self, record):
        task = get_current_task()
        if task and task.request and task.request.id:
            record.task_id = task.request.id
            record.task_name = task.name
        else:
            record.task_id = '-'
            record.task_name = '-'
        return True
```

```python
# config/settings.py
LOGGING = {
    'version': 1,
    'disable_existing_loggers': False,
    'filters': {
        'celery_task_context': {
            '()': 'common.logging_filters.CeleryTaskContextFilter',
        },
    },
    'formatters': {
        'task_verbose': {
            'format': '[{asctime}] {levelname} task={task_name} id={task_id} {message}',
            'style': '{',
        },
    },
    'handlers': {
        'console': {
            'class': 'logging.StreamHandler',
            'filters': ['celery_task_context'],
            'formatter': 'task_verbose',
        },
    },
    'loggers': {
        'blog': {'handlers': ['console'], 'level': 'INFO'},
        'accounts': {'handlers': ['console'], 'level': 'INFO'},
        'payments': {'handlers': ['console'], 'level': 'INFO'},
    },
}
```

ผลลัพธ์: ทุกบรรทัด log จาก task จะมีรูปแบบ
`[2026-09-26 09:00:01] INFO task=blog.tasks.render_report_pdf id=a1b2c3d4 ...`
โดยที่โค้ดใน task ไม่ต้องพิมพ์ `task_id` เองเลยแม้แต่ครั้งเดียว — ทำให้ค้นหา log
ทั้งหมดที่เกี่ยวกับ task ตัวหนึ่งได้ด้วยการ `grep` หา `id=a1b2c3d4` ตัวเดียว (ตัว UUID
เดียวกับที่เห็นในคอลัมน์ UUID ของ Flower ในขั้นตอนที่ 783.3 — เชื่อมกันได้ทันที)

### 788.3 หลักการที่ 2: ผูก Request ตั้งต้นเข้ากับ Task ด้วย Correlation ID

ปัญหาต่อมาคือ: task ที่ยิงจาก view ไหน ของ request ไหน ทวนจาก Part 066 (ที่มักมีการ
ใช้ request ID สำหรับ trace ข้าม middleware/log) หลักการเดียวกันนี้ต้อง **ส่งต่อไปยัง
background task ด้วย**:

```python
# common/middleware.py
import uuid

import threading

_request_id_ctx = threading.local()


class RequestIDMiddleware:
    """สร้าง request ID เฉพาะของแต่ละ request แล้วเก็บไว้ให้ view เข้าถึงได้"""

    def __init__(self, get_response):
        self.get_response = get_response

    def __call__(self, request):
        request.request_id = request.META.get('X-Request-ID', str(uuid.uuid4()))
        _request_id_ctx.value = request.request_id
        response = self.get_response(request)
        response['X-Request-ID'] = request.request_id
        return response


def get_current_request_id():
    return getattr(_request_id_ctx, 'value', None)
```

```python
# blog/views.py
from blog.services.reports import trigger_weekly_report_pipeline
from common.middleware import get_current_request_id


def generate_report_view(request):
    request_id = get_current_request_id()
    trigger_weekly_report_pipeline(category_id=None, request_id=request_id)
    return redirect('report_pending')
```

```python
# blog/tasks.py
@shared_task(name='blog.tasks.gather_report_data')
def gather_report_data(category_id=None, request_id=None):
    logger.info('เริ่มรวบรวมข้อมูลรายงาน (มาจาก request_id=%s)', request_id)
    ...
```

ตอนนี้เมื่อไล่ log จาก request_id ตัวเดียว จะเห็นทั้ง log ฝั่ง web view **และ** log
ของทุก task ในทุกขั้นตอนของ chain ที่ตามมา — เชื่อมโยงจากต้นทางถึงปลายทางได้ครบวงจร
ซึ่งเป็นสิ่งที่ **จำเป็นมากเมื่อใช้ centralized logging** เช่น ELK Stack, Grafana Loki
หรือ CloudWatch Logs Insights ที่รองรับการ query ด้วย correlation ID

### 788.4 หลักการที่ 3: อย่า Log ข้อมูลอ่อนไหว (PII) ลงไปตรง ๆ

ทวนจากคำเตือนเรื่อง Flower ในขั้นตอนที่ 782.2 — argument ของ task มักมีข้อมูลอ่อนไหว
เช่น email, บัตรเครดิต, token หลักการเดียวกันนี้ใช้กับ log ด้วย:

```python
# ผิด — log เบอร์บัตรเครดิตเต็มใบตรง ๆ
logger.info('กำลังชำระเงินด้วยบัตร %s', card_number)

# ถูก — mask ข้อมูลอ่อนไหวก่อน log เสมอ
logger.info('กำลังชำระเงินด้วยบัตรลงท้าย %s', card_number[-4:])
```

```python
# common/logging_utils.py
def mask_email(email):
    """แปลง user@example.com เป็น u***@example.com สำหรับ log"""
    local, _, domain = email.partition('@')
    if len(local) <= 1:
        return f'*@{domain}'
    return f'{local[0]}{"*" * (len(local) - 1)}@{domain}'
```

### 788.5 หลักการที่ 4: เลือก Log Level ให้เหมาะสมกับสถานการณ์

| สถานการณ์ | Level ที่แนะนำ |
|---|---|
| Task เริ่มทำงาน / ทำงานเสร็จสำเร็จ | `INFO` |
| Task retry (ยังไม่ fail ถาวร) | `WARNING` |
| Task fail ถาวรหลัง retry ครบทุกครั้งแล้ว | `ERROR` (Sentry จะจับอัตโนมัติผ่าน `on_failure` ในขั้นตอนที่ 787) |
| ข้อมูล debug ระดับละเอียด (เช่น query ที่รันแต่ละขั้น) | `DEBUG` (ปิดใน production ปกติ) |
| เหตุการณ์ผิดปกติที่ยังไม่ถือว่าเป็น error (เช่น record หาไม่เจอเพราะถูกลบไปแล้ว) | `WARNING` |

**หลักการสำคัญ**: อย่า log ทุกอย่างเป็น `ERROR` เพียงเพราะ "อยากให้เห็นชัด" เพราะจะทำให้
Sentry แจ้งเตือน false alarm บ่อยเกินไปจนทีมเริ่มเพิกเฉยต่อ alert จริง (ทวนหลักการ
"สัญญาณเตือนไฟไหม้ที่ดังบ่อยเกินไป" จาก Part 066 ขั้นตอนที่ 659)

### 788.6 หลักการที่ 5: Logging ของ django-RQ ใช้หลักการเดียวกัน

```python
# common/logging_filters.py
import logging

from rq import get_current_job


class RQJobContextFilter(logging.Filter):
    def filter(self, record):
        job = get_current_job()
        if job:
            record.task_id = job.id
            record.task_name = job.func_name
        else:
            record.task_id = '-'
            record.task_name = '-'
        return True
```

ใช้ formatter/handler เดียวกันกับที่ตั้งไว้ในขั้นตอนที่ 788.2 ได้เลย ทำให้ไม่ว่าระบบ
จะใช้ Celery หรือ django-RQ (หรือทั้งสองพร้อมกันระหว่างช่วงเปลี่ยนผ่าน) รูปแบบ log
ที่ออกมาจะสอดคล้องกัน อ่านและค้นหาได้ในมาตรฐานเดียว

### 788.7 ตารางสรุปขั้นตอนที่ 788

| หลักการ | สรุป |
|---|---|
| ผูก Task ID อัตโนมัติ | `logging.Filter` อ่านจาก `celery._state.get_current_task()` / `rq.get_current_job()` |
| Correlation ID ข้าม Request → Task | ส่ง `request_id` เป็น argument ของ task ตั้งแต่ตอนยิงงาน |
| ห้าม log PII ตรง ๆ | Mask ข้อมูลอ่อนไหวก่อน log เสมอ (เบอร์บัตร, email) |
| เลือก Log Level ให้เหมาะสม | INFO สำหรับปกติ, WARNING สำหรับ retry, ERROR สำหรับ fail ถาวรเท่านั้น |
| ใช้กับทั้ง Celery และ RQ | หลักการเดียวกัน ต่างกันแค่ฟังก์ชันดึง context ปัจจุบัน |

---

## ขั้นตอนที่ 789: การ Shutdown Worker อย่างสวยงาม (Graceful Shutdown) และข้อควรพิจารณาตอน Deploy

### 789.1 ปัญหา: Deploy โค้ดใหม่ระหว่างที่ Worker กำลังทำงานอยู่

ทุกครั้งที่ deploy โค้ดเวอร์ชันใหม่ ต้อง restart worker process เพื่อให้โค้ดใหม่มีผล
แต่ ณ วินาทีนั้น **อาจมี task กำลังรันอยู่ครึ่งทาง** เช่น กำลังตัดเงินลูกค้าอยู่พอดี
ถ้า worker ถูกฆ่าทิ้งทันที (SIGKILL) โดยไม่ทันให้ทำงานเสร็จ อาจเกิดผลลัพธ์ที่แย่ที่สุด:
เงินถูกตัดจากบัตรไปแล้วแต่ระบบไม่บันทึกว่าสำเร็จ (เชื่อมโยงกับปัญหา idempotency ที่
เรียนใน Part 076 ขั้นตอนที่ 758 โดยตรง) การ shutdown worker อย่างถูกวิธีจึงสำคัญ
พอ ๆ กับการออกแบบ task ให้ idempotent

### 789.2 สาม Level ของการ Shutdown Celery Worker

| วิธี | Signal | พฤติกรรม |
|---|---|---|
| **Warm Shutdown** (แนะนำเสมอ) | `SIGTERM` (ครั้งแรก) | Worker หยุด**รับ**งานใหม่ทันที แต่ปล่อยให้ task ที่กำลังรันอยู่ทำงานจนเสร็จก่อนค่อยปิดตัว |
| **Cold Shutdown** | `SIGTERM`/`SIGQUIT` (ครั้งที่สองติดกัน) | ปิดทันที ไม่รอ task ที่กำลังรันอยู่ให้เสร็จ — เสี่ยงงานค้างครึ่งทาง |
| **Force Kill** | `SIGKILL` | ฆ่าทันทีไม่มีการ cleanup ใด ๆ เลย ใช้เฉพาะกรณีฉุกเฉินที่ worker ค้างจริง ๆ เท่านั้น |

```bash
# วิธีทดสอบ warm shutdown ด้วยมือ
kill -TERM <celery_worker_pid>

# ห้ามส่ง SIGTERM ซ้ำสองครั้งโดยไม่ตั้งใจ (เช่น กด Ctrl+C สองครั้งใน terminal)
# เพราะครั้งที่สองจะกลายเป็น cold shutdown ทันที
```

> **กฎเหล็ก**: **การ deploy ทุกครั้งต้องใช้ warm shutdown (SIGTERM เพียงครั้งเดียว)
> เสมอ** และตั้ง timeout ในการรอให้นานพอสำหรับ task ที่ใช้เวลานานที่สุดในระบบ (ทวน
> `soft_time_limit`/`time_limit` จาก Part 076 ขั้นตอนที่ 757 — timeout ตอน deploy ควร
> นานกว่าค่า `time_limit` สูงสุดของ task ทุกตัวเสมอ)

### 789.3 systemd: ตั้งค่า Timeout ให้ถูกต้อง

```ini
# /etc/systemd/system/celery-worker.service
[Unit]
Description=Celery Worker
After=network.target redis.service

[Service]
Type=forking
User=celery
Group=celery
EnvironmentFile=/etc/celery/celery.conf
WorkingDirectory=/opt/myapp
ExecStart=/opt/myapp/venv/bin/celery multi start worker1 \
    -A config --pidfile=/var/run/celery/%n.pid \
    --logfile=/var/log/celery/%n.log
ExecStop=/opt/myapp/venv/bin/celery multi stopwait worker1 \
    --pidfile=/var/run/celery/%n.pid
ExecReload=/opt/myapp/venv/bin/celery multi restart worker1 \
    -A config --pidfile=/var/run/celery/%n.pid

# ให้เวลา worker ทำงานที่ค้างอยู่ให้เสร็จก่อนถูกบังคับปิด — ต้องนานกว่า time_limit สูงสุด
TimeoutStopSec=300
KillSignal=SIGTERM

[Install]
WantedBy=multi-user.target
```

สังเกตการใช้ `celery multi stopwait` แทน `celery multi stop` — คำสั่ง `stopwait`
**รอ (wait)** จนกว่า worker จะปิดตัวสมบูรณ์ (warm shutdown เสร็จสิ้น) ก่อนคืนค่ากลับ
ให้ systemd ต่างจาก `stop` ธรรมดาที่ส่ง signal แล้วคืนค่าทันทีโดยไม่รอ

### 789.4 Kubernetes: `terminationGracePeriodSeconds` และ `preStop` Hook

```yaml
# k8s/celery-worker-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: celery-worker-default
spec:
  replicas: 4
  template:
    spec:
      # ต้องนานกว่า time_limit ของ task ที่ยาวที่สุด + เวลา buffer เผื่อ
      terminationGracePeriodSeconds: 180
      containers:
        - name: celery-worker
          image: myapp:latest
          command: ["celery", "-A", "config", "worker", "-Q", "default", "--concurrency=4"]
          lifecycle:
            preStop:
              exec:
                # หน่วงเวลาสั้น ๆ ก่อนส่ง SIGTERM จริง เพื่อให้ Service/Load Balancer
                # หยุดส่ง traffic ใหม่เข้ามาก่อน (สำคัญกว่าสำหรับ web pod แต่ทำไว้ก็ไม่เสียหาย)
                command: ["sh", "-c", "sleep 5"]
      restartPolicy: Always
```

**คำเตือนสำคัญ**: ถ้า `terminationGracePeriodSeconds` สั้นเกินไป (ค่า default ของ
Kubernetes คือ 30 วินาที) และ task ในระบบใช้เวลารันนานกว่านั้น Kubernetes จะส่ง
**SIGKILL บังคับ** ทันทีที่ครบเวลา แม้ worker จะกำลัง warm shutdown อยู่ก็ตาม — ทำให้
งานที่กำลังรันอยู่ถูกฆ่ากลางทางอยู่ดี **ต้องตั้งค่านี้ให้นานกว่า `time_limit` สูงสุด
ของทุก task ในระบบเสมอ** ไม่มีข้อยกเว้น

### 789.5 Celery Beat: ข้อควรระวังเพิ่มเติมตอน Restart

ทวนกฎเหล็กจาก Part 076 ขั้นตอนที่ 751.4: **ห้ามรัน Beat มากกว่า 1 instance พร้อมกัน**
ซึ่งหมายความว่า **Beat ต้อง deploy เป็น Deployment ที่มี replica ตายตัว = 1 เสมอ ไม่ใช่
strategy แบบ rolling update ที่มีบางจังหวะที่ instance เก่ากับใหม่รันซ้อนกันชั่วคราว**
(ซึ่งเป็นพฤติกรรมปกติของ Kubernetes Deployment ทั่วไป):

```yaml
# k8s/celery-beat-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: celery-beat
spec:
  replicas: 1
  strategy:
    type: Recreate   # สำคัญมาก: บังคับให้ pod เก่าตายก่อน แล้วค่อยสร้าง pod ใหม่
                      # ต่างจาก RollingUpdate (default) ที่จะมี pod เก่า+ใหม่ซ้อนกันชั่วคราว
  template:
    spec:
      terminationGracePeriodSeconds: 30   # Beat เองไม่ค่อยมีงานค้าง ไม่ต้องรอนานเท่า worker
      containers:
        - name: celery-beat
          image: myapp:latest
          command: ["celery", "-A", "config", "beat", "--loglevel=info"]
```

`strategy.type: Recreate` คือกุญแจสำคัญ — บังคับให้ Kubernetes ฆ่า pod เก่าให้ตาย
สมบูรณ์ก่อน แล้วค่อยสร้าง pod ใหม่ขึ้นมา รับประกันว่าจะไม่มีช่วงเวลาใดที่ Beat 2
instance รันพร้อมกันแม้แต่วินาทีเดียว (ต่างจาก worker ที่ใช้ `RollingUpdate` ได้ตามปกติ
เพราะ worker หลายตัวรันพร้อมกันได้อยู่แล้วโดยไม่มีปัญหา)

### 789.6 Graceful Shutdown ของ django-RQ Worker

RQ ใช้หลักการ signal เดียวกันกับ Celery:

| Signal | พฤติกรรม |
|---|---|
| `SIGTERM`/`SIGINT` (ครั้งแรก) | Warm shutdown — ทำ job ปัจจุบันให้เสร็จก่อน แล้วค่อยปิดตัว ไม่รับ job ใหม่ |
| `SIGTERM`/`SIGINT` (ครั้งที่สองติดกัน) | Cold shutdown — พยายามหยุด job ปัจจุบันทันที |

```bash
# ทดสอบ warm shutdown ของ RQ worker
kill -TERM <rq_worker_pid>
```

การตั้งค่า `terminationGracePeriodSeconds`/`TimeoutStopSec` ใช้หลักการเดียวกันกับ
Celery ทุกประการ — ต้องนานกว่า `DEFAULT_TIMEOUT` ที่ตั้งไว้ใน `RQ_QUEUES` (ขั้นตอนที่
784.2) เสมอ

### 789.7 Health Check ที่ถูกต้องสำหรับ Worker: อย่าใช้ Queue Depth เป็น Liveness Probe

ข้อผิดพลาดที่พบบ่อย: ใช้ "จำนวนงานค้างในคิว" เป็นตัวตัดสิน liveness probe ของ worker
— นี่คือความเข้าใจผิด เพราะ **worker ที่ตายไปแล้วกับ worker ที่แค่กำลังยุ่งอยู่กับงาน
หนัก ทำให้คิวมีงานค้างเหมือนกันทั้งคู่** วิธีที่ถูกต้องคือเช็ค **heartbeat ของ worker
เอง**:

```bash
# ใช้เป็น liveness probe script — เช็คว่า worker ตอบสนองจริง ไม่ใช่แค่ดูความยาวคิว
celery -A config inspect ping --destination=celery@$(hostname) --timeout=5
```

```yaml
# k8s/celery-worker-deployment.yaml (เพิ่มเติมจากขั้นตอนที่ 789.4)
livenessProbe:
  exec:
    command:
      - sh
      - -c
      - "celery -A config inspect ping --destination=celery@$HOSTNAME --timeout=5"
  initialDelaySeconds: 30
  periodSeconds: 60
  timeoutSeconds: 10
```

### 789.8 Checklist สำหรับการ Deploy Background Job System อย่างปลอดภัย

| ข้อ | รายละเอียด |
|---|---|
| ✅ | ใช้ warm shutdown (`SIGTERM` ครั้งเดียว) เสมอตอน deploy — ไม่เคยใช้ `SIGKILL` โดยตรง |
| ✅ | `terminationGracePeriodSeconds`/`TimeoutStopSec` นานกว่า `time_limit` สูงสุดของทุก task |
| ✅ | Celery Beat ใช้ `strategy.type: Recreate` เสมอ ไม่ใช้ `RollingUpdate` |
| ✅ | Liveness probe เช็ค heartbeat (`inspect ping`) ไม่ใช่ความยาวคิว |
| ✅ | Task ที่สำคัญออกแบบให้ idempotent (ทวน Part 076 ขั้นตอนที่ 758) เผื่อ cold shutdown เกิดขึ้นโดยไม่ตั้งใจ |
| ✅ | Flower/RQ Dashboard เฝ้าดูช่วง deploy จริงเพื่อยืนยันว่า worker กลับมา online ครบ |
| ✅ | Sentry Crons (ขั้นตอนที่ 787.4) ยืนยันว่า Beat schedule ยังรันต่อเนื่องหลัง deploy |

### 789.9 ตารางสรุปขั้นตอนที่ 789

| หัวข้อ | สรุป |
|---|---|
| Warm vs Cold Shutdown | `SIGTERM` ครั้งแรก = รอ task เสร็จก่อน, ครั้งที่สอง = ฆ่าทันที |
| systemd | ใช้ `celery multi stopwait` + `TimeoutStopSec` นานเพียงพอ |
| Kubernetes Worker | `terminationGracePeriodSeconds` นานกว่า `time_limit` สูงสุดเสมอ |
| Kubernetes Beat | `strategy.type: Recreate` เพื่อไม่ให้มี Beat 2 instance ซ้อนกัน |
| Liveness Probe ที่ถูกต้อง | `celery inspect ping` (heartbeat) ไม่ใช่ความยาวคิว |
| RQ Worker | ใช้หลักการ signal เดียวกับ Celery ทุกประการ |

---

## ขั้นตอนที่ 790: สรุป Phase 9 ทั้งหมด (Part 073-079)

Part นี้เป็น Part สุดท้ายของ **Phase 9: Async, Celery และ Channels** ก่อนจะข้ามไป
Phase 10 มาทบทวนภาพรวมทั้งหมดที่คุณสร้างมาตลอด 7 Part (70 ขั้นตอน) อย่างละเอียด

### 790.1 ตารางภาพรวม Phase 9 ทั้งหมด

| Part | ขั้นตอน | หัวข้อหลัก | สิ่งที่ได้สร้างขึ้น |
|---|---|---|---|
| **073** | 721-730 | Async Views และ ASGI เบื้องต้น | Async view ด้วย `async def`, เข้าใจ I/O-bound vs CPU-bound, Async ORM, `sync_to_async`/`async_to_sync`, เรียก external API ด้วย `httpx`, เปรียบเทียบ Daphne/Uvicorn/Hypercorn |
| **074** | 731-740 | Django Channels ขั้นสูง | Consumer แบบ inheritance/mixin, `channels_redis` ใน production, reconnect strategy, Presence tracking, rate limiting WebSocket, scale ข้ามหลาย server, ระบบแชท real-time เต็มรูปแบบ |
| **075** | 741-750 | Celery เบื้องต้น | ติดตั้ง Celery + Redis broker, `@shared_task`, `.delay()`, result backend, retry, serialization (JSON), error handling, แปลงระบบอีเมลเป็น Celery task |
| **076** | 751-760 | Celery ขั้นสูง | Celery Beat, `django-celery-beat`, Chain/Group/Chord, task routing/queue, time limit, idempotency, testing ด้วย `CELERY_TASK_ALWAYS_EAGER`, pipeline รายงานรายสัปดาห์ |
| **077** | 761-770 | Message Queue ด้วย RabbitMQ/Redis | เจาะลึก AMQP protocol, เปรียบเทียบ RabbitMQ vs Redis เป็น broker อย่างละเอียด, สลับ broker โดยไม่กระทบ business logic, Dead Letter Queue, ออกแบบระบบให้ทนทานเมื่อ broker ล่มชั่วคราว |
| **078** | 771-780 | Real-time Notification System | Notification model, ส่งแจ้งเตือนแบบ real-time ผ่าน Channels, ประกอบ Celery chord เข้ากับ Channels layer (ตามที่ Part 076 ขั้นตอนที่ 760.5 กล่าวถึง), สถานะอ่าน/ยังไม่อ่าน, การตั้งค่าความถี่การแจ้งเตือนของผู้ใช้, fallback เป็นอีเมลสรุปเมื่อผู้ใช้ offline |
| **079** | 781-790 | Background Job Monitoring | Flower + authentication, อ่าน Dashboard, django-rq ทางเลือกที่เรียบง่ายกว่า, RQ Dashboard, เปรียบเทียบ Celery vs RQ, เชื่อม Sentry, logging best practice, graceful shutdown |

### 790.2 สถาปัตยกรรมรวมของระบบ Async ทั้งหมดใน Phase 9

หลังจบ Phase 9 ระบบของคุณมีสถาปัตยกรรมโดยรวมดังนี้ (ทุกกล่องคือสิ่งที่คุณสร้างขึ้นเอง
ตลอด Part 073-079):

```
                         ┌───────────────────────┐
                         │   Browser (Client)     │
                         └───────────┬────────────┘
                    HTTP             │             WebSocket
              ┌────────────────────┐ │ ┌──────────────────────────┐
              ▼                    │ │ ▼                          │
      ┌───────────────┐            │ │  ┌──────────────────┐      │
      │  Django (WSGI/ │◄───────────┘ └─►│ Django Channels   │      │
      │  ASGI, Part073)│  Async View      │ Consumers (074)   │      │
      └───────┬────────┘                 └─────────┬──────────┘      │
              │                                     │ Group send      │
              │ .delay() / enqueue()                ▼                 │
              ▼                            ┌──────────────────┐       │
      ┌───────────────┐    events         │  channels_redis   │       │
      │  Celery /      │──────────────────►│  Channel Layer     │───────┘
      │  django-RQ     │                   └──────────────────┘
      │  (075-076, 784)│
      └───────┬────────┘
              │ broker (Redis/RabbitMQ, Part 077)
              ▼
      ┌───────────────┐        ┌──────────────────┐
      │  Worker Pool   │───────►│ Notification     │  (Part 078)
      │  (แยกคิวตาม    │        │ System (DB +     │
      │  ความสำคัญ)    │        │ Channels fan-out)│
      └───────┬────────┘        └──────────────────┘
              │
              │ monitor
              ▼
      ┌───────────────┐        ┌──────────────────┐
      │  Flower /      │───────►│  Sentry (Errors +│  (Part 079)
      │  RQ Dashboard  │        │  Crons + Slack)  │
      └───────────────┘        └──────────────────┘
```

ทุกกล่องในแผนภาพนี้คือสิ่งที่คุณสามารถอธิบายได้แล้วว่าทำงานอย่างไร ตั้งแต่ต้นทาง
(request จาก browser) จนถึงปลายทาง (การแจ้งเตือน error กลับไปหาทีม dev) — นี่คือ
ภาพรวมที่ทีม backend ระดับมืออาชีพต้องเข้าใจครบทุกจุดในแผนภาพ ไม่ใช่แค่บางส่วน

### 790.3 Quiz ทบทวน Phase 9 (12 ข้อ พร้อมเฉลย)

**คำถามที่ 1**: Async view ใน Django ช่วยเพิ่มประสิทธิภาพได้ดีที่สุดในสถานการณ์แบบไหน?

> **เฉลย**: งานแบบ **I/O-bound** เช่น เรียก API ภายนอก, รอฐานข้อมูล/เครือข่าย เพราะ
> ระหว่างรอ I/O เสร็จ event loop ไปประมวลผล request อื่นได้ก่อน ต่างจากงานแบบ
> **CPU-bound** (คำนวณหนัก) ที่ async ไม่ช่วยอะไรเลยเพราะ GIL ยังบล็อกอยู่ดี (ทวนขั้นตอนที่
> 722 และ 729)

**คำถามที่ 2**: ทำไม Django Channels ต้องใช้ `channels_redis` เป็น Channel Layer ใน
production แทนที่จะใช้ `InMemoryChannelLayer`?

> **เฉลย**: `InMemoryChannelLayer` เก็บข้อมูลไว้ในหน่วยความจำของกระบวนการเดียว ทำให้
> **ใช้ scale ข้ามหลาย server ไม่ได้** (ผู้ใช้ 2 คนที่เชื่อมต่อกับ server คนละตัวจะคุยกัน
> ไม่ได้) `channels_redis` ใช้ Redis เป็นตัวกลางร่วม ทำให้ทุก server เห็น group/message
> เดียวกัน (ทวนขั้นตอนที่ 733 และ 737)

**คำถามที่ 3**: `.delay()` ของ Celery กับการเรียกฟังก์ชันตรง ๆ ต่างกันอย่างไร และทำไม
ต้องใช้ `.delay()` สำหรับงานหนัก?

> **เฉลย**: `.delay()` ยิงงานเข้าคิวแล้วคืนค่าทันที (non-blocking) ให้ worker แยก
> กระบวนการไปทำภายหลัง ต่างจากเรียกฟังก์ชันตรง ๆ ที่ **บล็อก request-response cycle**
> จนกว่างานจะเสร็จ — สำหรับงานหนัก (ส่งอีเมล, สร้าง PDF) การบล็อกแบบนี้ทำให้ผู้ใช้ต้องรอ
> นานเกินไปและเสี่ยง timeout (ทวนขั้นตอนที่ 741 และ 743)

**คำถามที่ 4**: เพราะเหตุใด Celery Beat จึงห้ามรันมากกว่า 1 instance พร้อมกันเด็ดขาด?

> **เฉลย**: เพราะ Beat แต่ละ instance จะยิง schedule เดียวกันเข้าคิวซ้ำ ทำให้ periodic
> task ถูกรัน **ซ้ำหลายเท่าตามจำนวน instance** เช่น อีเมลสรุปรายสัปดาห์จะถูกส่งซ้ำ 2-3
> ฉบับต่อผู้ใช้หนึ่งคน (ทวนขั้นตอนที่ 751.4 และ 789.5)

**คำถามที่ 5**: จงอธิบายความแตกต่างระหว่าง Chain, Group และ Chord ด้วยประโยคสั้น ๆ

> **เฉลย**: **Chain** รันตามลำดับ ผลของ A ป้อนเข้า B; **Group** รันพร้อมกันแบบขนานโดย
> ไม่ขึ้นต่อกัน; **Chord** คือ Group ที่รอทุกตัวเสร็จก่อนแล้วส่งผลรวมเข้า callback ตัวเดียว
> โดยไม่ต้อง block รอ (ทวนขั้นตอนที่ 753-755)

**คำถามที่ 6**: ทำไม Redis จึงไม่เหมาะกับการทำ Message Priority แบบละเอียดในคิวเดียว
เมื่อเทียบกับ RabbitMQ?

> **เฉลย**: Redis รองรับ priority field แบบจำกัดมาก ไม่แม่นยำเท่า RabbitMQ ที่รองรับ
> เต็มรูปแบบผ่าน `x-max-priority` ของ AMQP โดยตรง คำแนะนำทั่วไปคือแยกเป็นคนละคิวแทนการ
> พึ่ง priority ในคิวเดียวเมื่อใช้ Redis (ทวนขั้นตอนที่ 756.4 และ Part 077)

**คำถามที่ 7**: ระบบ Notification แบบ Real-time (Part 078) ทำงานร่วมกับ Celery และ
Channels อย่างไรเมื่อผู้ใช้ offline อยู่?

> **เฉลย**: เมื่อผู้ใช้ online ระบบส่ง notification ผ่าน Channels layer ทันทีแบบ
> real-time แต่เมื่อผู้ใช้ offline ระบบต้องมีกลไก fallback เช่น บันทึกลงฐานข้อมูลรอไว้
> ให้เห็นเมื่อ login กลับมา หรือส่งอีเมลสรุปผ่าน Celery task แทน เพื่อไม่ให้แจ้งเตือน
> สำคัญหายไปเฉย ๆ

**คำถามที่ 8**: ทำไมต้องเปิด `-E` (หรือ `CELERY_WORKER_SEND_TASK_EVENTS`) ก่อนใช้งาน
Flower ให้เต็มรูปแบบ?

> **เฉลย**: Flower ทำงานโดยฟัง **event** ที่ worker กระจายออกมาผ่าน broker ถ้าไม่เปิด
> การส่ง event ไว้ Flower จะเห็นแค่ worker online/offline แต่จะไม่เห็นรายละเอียด task
> เลย (ทวนขั้นตอนที่ 783.1)

**คำถามที่ 9**: จงเปรียบเทียบข้อดี-ข้อเสียของ django-RQ เทียบกับ Celery ในแง่ความซับซ้อน
กับความสามารถของ workflow

> **เฉลย**: django-RQ เรียบง่ายกว่ามาก เขียนฟังก์ชัน Python ธรรมดาแล้วใช้ได้เลย ใช้
> Redis อย่างเดียว แต่ workflow มีจำกัดมาก (แค่ `depends_on` พื้นฐาน ไม่มี Group/Chord)
> และไม่มี Beat ในตัว (ต้องพึ่ง `rq-scheduler`) ส่วน Celery ซับซ้อนกว่าในการติดตั้งแต่
> มี workflow ที่ทรงพลังครบ รองรับ broker หลายชนิด และ routing/retry ที่ละเอียดกว่ามาก
> (ทวนขั้นตอนที่ 786)

**คำถามที่ 10**: Sentry Crons ต่างจากการที่ `CeleryIntegration()` จับ exception ปกติ
อย่างไร และทำไมทั้งสองจำเป็นต้องมีคู่กัน?

> **เฉลย**: `CeleryIntegration()` จับ exception เมื่อ task **รันแล้วเกิด error**
> แต่ Sentry Crons ใช้หลักการ check-in เพื่อตรวจจับสถานการณ์ที่ **task ไม่รันเลย**
> (เช่น Beat process ตายไปเงียบ ๆ) ซึ่งไม่มี exception ให้จับ ทั้งสองต้องมีคู่กันเพื่อ
> ครอบคลุมทั้ง "รันแล้วพัง" และ "ไม่รันเลย" (ทวนขั้นตอนที่ 787.4)

**คำถามที่ 11**: ทำไมการใช้ความยาวคิว (queue depth) เป็น liveness probe ของ Celery
worker จึงเป็นวิธีที่ผิด?

> **เฉลย**: เพราะ worker ที่ตายไปแล้วกับ worker ที่แค่กำลังยุ่งอยู่กับงานหนัก ทำให้
> คิวมีงานค้างเหมือนกันทั้งคู่ วิธีที่ถูกต้องคือเช็ค heartbeat ของ worker เองผ่าน
> `celery inspect ping` (ทวนขั้นตอนที่ 789.7)

**คำถามที่ 12**: จงอธิบายว่าทำไม `terminationGracePeriodSeconds` ของ Kubernetes
Deployment สำหรับ Celery worker จึงต้องตั้งค่าให้นานกว่า `time_limit` สูงสุดของ task
ในระบบเสมอ

> **เฉลย**: เพราะเมื่อ Kubernetes สั่งปิด pod มันจะส่ง `SIGTERM` (warm shutdown) แล้วรอ
> ตาม `terminationGracePeriodSeconds` ก่อนจะบังคับส่ง `SIGKILL` ถ้าเวลานี้สั้นกว่า
> เวลาที่ task ที่ยาวที่สุดต้องใช้ในการทำงานให้เสร็จ Kubernetes จะฆ่า worker กลางทาง
> ทั้งที่ยังพยายาม shutdown อย่างสวยงามอยู่ (ทวนขั้นตอนที่ 789.4)

### 790.4 Checklist ก่อนไป Phase ถัดไป

- [ ] อธิบายความแตกต่างระหว่าง sync view, async view, และ WebSocket consumer ได้ด้วยคำพูดตัวเอง
- [ ] เขียน Celery task พร้อม retry, idempotency, และ time limit ได้ครบ
- [ ] ประกอบ chain/group/chord อย่างน้อยหนึ่ง pipeline ที่ใช้งานได้จริง
- [ ] อธิบายได้ว่าทำไมต้องแยก queue ตามความสำคัญ และรู้ข้อจำกัดของ priority ใน Redis
- [ ] ติดตั้ง Flower พร้อม authentication และอ่าน Dashboard เพื่อ debug task ที่ล้มเหลวได้
- [ ] เขียน background job อย่างน้อยหนึ่งตัวด้วย django-rq และอธิบายได้ว่าต่างจาก Celery อย่างไร
- [ ] เชื่อมต่อ Sentry เข้ากับทั้ง Celery และ RQ พร้อมทดสอบว่า error/Sentry Crons ทำงานจริง
- [ ] ตั้งค่า logging ที่ผูก task ID และ correlation ID เข้ากับทุก log record อัตโนมัติ
- [ ] อธิบาย warm/cold shutdown ได้ และตั้งค่า deployment (systemd/Kubernetes) อย่างปลอดภัย

### 790.5 แบบฝึกหัดใหญ่ปิดท้าย Phase 9: ระบบ "ประมวลผลคำสั่งซื้อ" แบบครบวงจร

สร้างระบบจำลอง **e-commerce order processing pipeline** ที่ประกอบทุกเทคนิคจาก Phase 9
ทั้งหมดเข้าด้วยกัน โดยมีข้อกำหนดดังนี้:

**ส่วนที่ 1 — Pipeline หลัก (ทวน Part 075-076)**:
สร้าง Celery chain 4 ขั้นตอนสำหรับคำสั่งซื้อใหม่:
`validate_order` → `charge_payment` (ต้อง idempotent 100% ตามหลักการขั้นตอนที่ 758
เพราะเกี่ยวกับเงินจริง) → `reserve_inventory` → `send_order_confirmation`
ถ้าขั้นตอนใดล้มเหลว ต้องมี `link_error` แจ้งเตือนทันที และต้อง**คืนเงิน (refund)
อัตโนมัติ**ถ้าขั้นตอนหลังจาก charge_payment ล้มเหลว (เช่น สินค้าหมดสต๊อกหลังตัดเงินไปแล้ว)

**ส่วนที่ 2 — Routing และ Queue (ทวนขั้นตอนที่ 756)**:
แยก `charge_payment` ไปอยู่ queue `critical` (concurrency สูง, worker แยกเฉพาะ)
และ `send_order_confirmation` ไปอยู่ queue `low_priority`

**ส่วนที่ 3 — Real-time Notification (ทวน Part 078)**:
เมื่อ order สำเร็จ ส่ง notification แบบ real-time ผ่าน Channels ไปยังหน้าเว็บของลูกค้า
ทันที (ไม่ต้อง refresh หน้าเอง) พร้อมบันทึกลงฐานข้อมูล notification สำหรับกรณีลูกค้า
offline

**ส่วนที่ 4 — Monitoring (ทวน Part 079)**:
ติดตั้ง Flower พร้อม authentication เชื่อม Sentry พร้อม Sentry Crons สำหรับ periodic
task ที่ตรวจสอบคำสั่งซื้อที่ค้างเกิน 10 นาที (`check_stuck_orders`) และตั้ง logging
ที่ผูก `order_id` เข้ากับทุก log record ของ pipeline นี้โดยอัตโนมัติ

**ส่วนที่ 5 — Deployment (ทวนขั้นตอนที่ 789)**:
เขียนไฟล์ Kubernetes manifest (หรือ systemd unit ถ้าไม่ได้ใช้ K8s) ที่ตั้งค่า graceful
shutdown อย่างถูกต้องสำหรับ worker คิว `critical` โดยเฉพาะ พร้อมอธิบายเป็นลายลักษณ์
อักษรว่าทำไม `terminationGracePeriodSeconds` ที่เลือกถึงเพียงพอ

**เกณฑ์ความสำเร็จ**: เขียนเทสยืนยันว่า (1) เรียก `charge_payment` ซ้ำ 5 ครั้งแล้วเงิน
ถูกตัดครั้งเดียว (2) ถ้า `reserve_inventory` ล้มเหลว เงินที่ตัดไปแล้วต้องถูกคืนอัตโนมัติ
(3) notification ถูกส่งผ่าน Channels จริงเมื่อ order สำเร็จ และ (4) log ทุกบรรทัดของ
pipeline หนึ่ง order มี `order_id` เดียวกันปรากฏครบทุกจุด ค้นหาด้วย `grep` ได้ในคำสั่ง
เดียว

นี่คือแบบฝึกหัดที่รวมทุกสิ่งที่เรียนมาตลอด Phase 9 เข้าด้วยกันเป็นระบบเดียวที่ใกล้เคียง
กับสิ่งที่ทีม backend มืออาชีพต้องสร้างจริงในระบบ e-commerce ระดับโลก

### 790.6 คำถามที่พบบ่อยส่งท้าย Phase 9

**Q: จำเป็นต้องเรียนทั้ง Celery และ django-RQ จริง ๆ ไหม ไม่เลือกเรียนอย่างใดอย่างหนึ่ง
พอไหม?**
A: สำหรับงานจริง คุณอาจใช้แค่ตัวใดตัวหนึ่งในโปรเจกต์หนึ่ง ๆ แต่การเข้าใจทั้งคู่สำคัญมาก
เพราะ (1) โปรเจกต์ที่คุณเข้าร่วมในอนาคตอาจเลือกใช้ตัวใดตัวหนึ่งไปแล้ว และ (2) การเข้าใจ
ทางเลือกทั้งสองแบบช่วยให้คุณ**ตัดสินใจเลือกเครื่องมือได้อย่างมีเหตุผล** แทนที่จะใช้
Celery ทุกครั้งเพียงเพราะ "เคยเรียนมา" ทั้งที่งานนั้นเรียบง่ายพอที่ django-rq จะเพียงพอ

**Q: ทำไมหลักสูตรนี้ไม่สอน AWS SQS หรือ Google Cloud Tasks ทั้งที่เป็น managed service
ที่ไม่ต้องดูแล broker เอง?**
A: หลักการของ Celery/RQ ที่เรียนไปสามารถต่อยอดไปใช้กับ managed queue service ได้ไม่ยาก
(Celery รองรับ SQS เป็น broker ได้โดยตรง) แนวคิดเรื่อง idempotency, routing,
monitoring, graceful shutdown ที่เรียนมาทั้งหมดใช้ได้เหมือนกันไม่ว่าจะรัน broker เอง
หรือใช้ managed service เราจะพูดถึงการเลือก managed service เทียบกับ self-hosted
อย่างละเอียดใน Phase DevOps

**Q: Phase 9 ดูซับซ้อนมาก จำเป็นต้องใช้ทุกเทคนิคในทุกโปรเจกต์จริงหรือไม่?**
A: ไม่จำเป็น โปรเจกต์เล็กอาจใช้แค่ Celery พื้นฐาน (Part 075) ไม่ต้องมี Chord หรือ
Message Queue ขั้นสูงเลยก็ได้ หลักการสำคัญที่สุดที่ควรติดตัวไปเสมอไม่ว่าโปรเจกต์จะใหญ่
แค่ไหนคือ **idempotency** (ขั้นตอนที่ 758) และ **graceful shutdown** (ขั้นตอนที่ 789)
เพราะสองเรื่องนี้ป้องกันความเสียหายร้ายแรงที่สุดได้แม้ในระบบที่เรียบง่ายที่สุด

**Q: Phase 10 ที่กำลังจะเริ่มเกี่ยวข้องกับ Background Job ที่เรียนมาไหม?**
A: เกี่ยวข้องโดยตรง — ตัวอย่างเช่น การ validate/sanitize argument ที่ส่งเข้า Celery
task ก็เป็นส่วนหนึ่งของ input validation (Part 081), การเก็บ `SENTRY_DSN`,
`CELERY_BROKER_URL` และ credential อื่น ๆ อย่างปลอดภัยคือหัวข้อของ Part 084 Secrets
Management โดยตรง และการจำกัด rate ของ task ที่เรียก API ภายนอก (ทวนขั้นตอนที่ 756)
เชื่อมโยงกับ Rate Limiting ใน Part 083 อย่างแนบแน่น

---

## เตรียมตัวสำหรับ Part ถัดไป

Phase 9 จบลงแล้วอย่างสมบูรณ์! คุณสร้างระบบ asynchronous เต็มรูปแบบตั้งแต่ async view,
WebSocket real-time, background task queue ที่ซับซ้อน, ไปจนถึงการ monitor และ deploy
อย่างปลอดภัย — ทักษะชุดนี้คือสิ่งที่แยกนักพัฒนา Django มือใหม่ออกจากวิศวกร backend
ระดับมืออาชีพที่สร้างระบบรองรับ traffic จริงได้

**Part 080: Django Security Best Practices เบื้องต้น** จะเริ่มต้น **Phase 10: Security**
โดยพาไปทบทวนภาพรวมความปลอดภัยของ Django ทั้งระบบ ตั้งแต่ `SECRET_KEY`, `DEBUG=False`
ใน production, `ALLOWED_HOSTS`, การอัปเดต dependency ให้ปลอดภัยจาก CVE ที่รู้จัก, ไปจนถึง
Django Security Checklist อย่างเป็นทางการ ก่อนจะเจาะลึกแต่ละหัวข้อในรายละเอียด — CSRF/XSS/
SQL Injection (Part 081), Security Headers/HTTPS (Part 082), Rate Limiting/Brute Force
Protection (Part 083 ซึ่งต่อยอดจากหลักการ rate limit ที่เรียนใน Phase นี้โดยตรง),
Secrets Management (Part 084 ซึ่งครอบคลุมการเก็บ `SENTRY_DSN`/`CELERY_BROKER_URL` ที่คุณ
ใช้มาตลอด Phase 9 อย่างปลอดภัย), และปิดท้ายด้วย Security Auditing/Penetration Testing
เบื้องต้น (Part 085)

เตรียมทบทวนระบบ Django ที่คุณสร้างมาตั้งแต่ Part 001 ทั้งหมดไว้ให้พร้อม เพราะ Phase 10
จะพาคุณย้อนกลับไปตรวจสอบทุกจุดของระบบด้วยมุมมองของแฮกเกอร์ ก่อนจะเสริมเกราะป้องกันให้
แน่นหนาระดับที่ใช้งานจริงในระบบระดับโลกได้!
