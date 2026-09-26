# Part 057: WebSockets เบื้องต้นด้วย Django Channels

> **ขั้นตอนที่ 561-570 ของหลักสูตร** | Phase 6: Frontend Integration
>
> Part 052 สอนให้คุณดึงข้อมูลด้วย `fetch()` + JSON เมื่อผู้ใช้ **สั่งการเอง**
> (กดปุ่ม, พิมพ์ค้นหา) Part 053 ต่อยอดด้วย HTMX ที่ทำสิ่งเดียวกันโดยไม่ต้อง
> เขียน JavaScript เอง แต่ทั้งสองแนวทางมีข้อจำกัดร่วมกันข้อหนึ่งที่ยังไม่ถูก
> แก้: **ทุกการสื่อสารต้องเริ่มจากฝั่ง client เสมอ** — server ไม่มีทาง
> "บอกก่อน" ว่ามีอะไรใหม่เกิดขึ้น เว้นแต่ client จะถามซ้ำ ๆ เป็นระยะ (polling)
> Part นี้จะแนะนำ **WebSocket** และ **Django Channels** ซึ่งเปิดทางให้
> server ส่งข้อมูลเข้าหา client ได้เองทันทีที่มีเหตุการณ์เกิดขึ้น โดยไม่ต้อง
> รอให้ client มาถามก่อนเลยแม้แต่ครั้งเดียว คุณจะได้เรียนรู้ตั้งแต่แนวคิด
> WebSocket เทียบกับ HTTP request/response ธรรมดา ติดตั้ง Django Channels
> บนพื้นฐาน ASGI ที่ปูไว้ตั้งแต่ Part 004 เขียน Consumer ทั้งแบบ synchronous
> (`WebsocketConsumer`) และ asynchronous (`AsyncWebsocketConsumer`) ใช้
> Channel Layer บน Redis เพื่อกระจายข้อความข้าม connection ผูก Signal จาก
> Part 019 เข้ากับ Channel Layer เพื่อสร้างระบบแจ้งเตือนคอมเมนต์ใหม่แบบ
> real-time จริง ยืนยันตัวตนผู้ใช้ใน Consumer เขียนเทสต์ด้วย
> `WebsocketCommunicator` และปิดท้ายด้วยข้อควรพิจารณาตอน deploy — Part นี้
> เน้นปูพื้นฐานให้แน่นก่อน ส่วน Groups ขั้นสูง การจัดการ scale หลาย process
> และรูปแบบ Consumer ที่ซับซ้อนกว่านี้จะเจาะลึกเต็มรูปแบบใน **Part 074**

---

## สารบัญของ Part นี้

- ขั้นตอนที่ 561: แนวคิด WebSocket เทียบกับ HTTP Request/Response ธรรมดา
- ขั้นตอนที่ 562: ติดตั้ง Django Channels และตั้งค่า ASGI ต่อยอดจาก Part 004
- ขั้นตอนที่ 563: `WebsocketConsumer` เบื้องต้น (Synchronous) — Consumer แรกที่ Echo ข้อความกลับ
- ขั้นตอนที่ 564: `AsyncWebsocketConsumer` — เหตุผลที่ควรใช้ Async ในงาน Real-time
- ขั้นตอนที่ 565: Channel Layers ด้วย Redis — ส่งข้อความข้าม Consumer ด้วย `group_send`
- ขั้นตอนที่ 566: สร้างระบบแจ้งเตือนคอมเมนต์ใหม่แบบ Real-time จริง (ผูกกับ Signal จาก Part 019)
- ขั้นตอนที่ 567: การยืนยันตัวตนใน Consumer (`self.scope['user']`, `AuthMiddlewareStack`)
- ขั้นตอนที่ 568: การเขียน Test สำหรับ WebSocket Consumer (`WebsocketCommunicator`)
- ขั้นตอนที่ 569: ข้อควรพิจารณาการ Deploy Channels (Daphne/Uvicorn แทน Gunicorn ธรรมดา)
- ขั้นตอนที่ 570: สรุปและแบบฝึกหัด — ระบบแจ้งเตือนคอมเมนต์ใหม่แบบ Real-time เต็มรูปแบบ

---

## ขั้นตอนที่ 561: แนวคิด WebSocket เทียบกับ HTTP Request/Response ธรรมดา

### 561.1 ทบทวนข้อจำกัดของ HTTP ที่ Part 052-053 ยังแก้ไม่ได้

ทุก Part ที่ผ่านมาในหลักสูตรนี้ ไม่ว่าจะเป็น View ธรรมดา (Part 004), REST API
(Phase 5), Fetch API (Part 052) หรือ HTMX (Part 053) ล้วนสร้างอยู่บนโมเดล
การสื่อสารแบบเดียวกันทั้งหมดคือ **HTTP Request/Response**:

```
Browser (Client)                          Django (Server)
      │                                          │
      │  1. ส่ง Request (client เป็นฝ่ายเริ่มเสมอ)   │
      │ ───────────────────────────────────────> │
      │                                          │  2. ประมวลผล
      │  3. ส่ง Response กลับ แล้ว "ปิด" การเชื่อมต่อ │
      │ <─────────────────────────────────────── │
      │                                          │
   (เงียบ — server บอกอะไร client ไม่ได้เลย
    จนกว่า client จะถามใหม่)
```

ลักษณะสำคัญของ HTTP ที่เป็นทั้งจุดแข็งและข้อจำกัด:

- **Client เป็นฝ่ายเริ่มการสื่อสารเสมอ** — server ไม่มีช่องทาง "พูดก่อน" ได้
  เลยไม่ว่าจะออกแบบ View ฉลาดแค่ไหนก็ตาม
- **Stateless และเป็น request-response แบบ 1 ต่อ 1** — request หนึ่งได้
  response หนึ่ง แล้วการเชื่อมต่อ TCP มักถูกปิด (หรือ keep-alive รอ request
  ถัดไปช่วงสั้น ๆ) ไม่มี "การสนทนาต่อเนื่อง"

ตัวอย่างเดิมของ Part 053 ข้อ 527 (Infinite Scroll ด้วย `hx-trigger="revealed"`)
และปุ่มถูกใจข้อ 522.4 ล้วนทำงานได้ดีเพราะ**ผู้ใช้เป็นคนกดเอง** — แต่ลอง
จินตนาการโจทย์ใหม่: "แจ้งเตือนแอดมินทันทีที่มีคนโพสต์คอมเมนต์ใหม่ โดยแอดมิน
ไม่ต้องกดอะไรเลย และไม่ต้องรีเฟรชหน้า" โจทย์นี้ทำไม่ได้ด้วย HTTP ธรรมดา
เพราะไม่มีทางที่ server จะ "ส่ง response" ไปหา browser ที่ไม่ได้ส่ง request
มาก่อน

### 561.2 ทางแก้แบบเดิม: Polling (ถามซ้ำเป็นระยะ)

ก่อนจะรู้จัก WebSocket หลายทีมแก้ปัญหานี้ด้วยเทคนิค **Polling** — ให้ client
ยิง request ถามซ้ำ ๆ ทุก ๆ 2-5 วินาทีว่า "มีอะไรใหม่ไหม" ซึ่ง HTMX ทำได้ง่าย
มากด้วย `hx-trigger="every 3s"` (ต่อยอดจากตารางใน Part 053 ข้อ 523.1):

```html
<!-- ตัวอย่าง Polling ด้วย HTMX — ใช้งานได้ แต่มีต้นทุนแฝง -->
<div hx-get="{% url 'blog:unread_comment_count' %}"
     hx-trigger="every 3s"
     hx-swap="innerHTML">
    0
</div>
```

Polling ใช้งานได้จริงและง่ายมาก แต่มีต้นทุนที่ต้องเข้าใจให้ชัดก่อนตัดสินใจ
เลือกใช้:

| ประเด็น | Polling (Part 053 ข้อ 523.1) | WebSocket (Part นี้) |
|---|---|---|
| ใครเป็นฝ่ายเริ่มส่งข้อมูล | Client ถามซ้ำเป็นระยะเสมอ | **Server ส่งได้เองทันทีที่มีเหตุการณ์** |
| Latency (ความหน่วงกว่าจะเห็นข้อมูลใหม่) | สูงสุดเท่ากับรอบเวลา poll (เช่น 3 วินาที) | เกือบเป็นศูนย์ (ทันทีที่ server ส่ง) |
| จำนวน Request ต่อผู้ใช้ 1 คนใน 1 นาที | สูงมาก (20 request/นาที ถ้า poll ทุก 3 วินาที) แม้ไม่มีอะไรใหม่เลย | 1 การเชื่อมต่อค้างไว้ ไม่มี request ซ้ำ |
| ภาระเซิร์ฟเวอร์เมื่อผู้ใช้เพิ่มขึ้น | เพิ่มเป็นเส้นตรงตามจำนวนผู้ใช้ × ความถี่ poll | เพิ่มตามจำนวนผู้ใช้ (connection ค้าง) แต่ไม่มี request ซ้ำซ้อน |
| ความซับซ้อนในการเขียนโค้ด | ต่ำมาก (attribute เดียวจบ) | สูงกว่า ต้องตั้งค่า ASGI, Consumer, Channel Layer |
| เหมาะกับงานที่ความถี่การเปลี่ยนแปลงต่ำ | ✅ เหมาะมาก (เช็ค stock สินค้าทุก 30 วินาทีก็พอ) | ใช้ได้แต่ "ฆ่ายุงด้วยปืนใหญ่" ถ้าข้อมูลเปลี่ยนไม่บ่อย |
| เหมาะกับ Chat / Live Notification / Multiplayer | ❌ Latency สูงเกินไป รู้สึก "ไม่จริง" | ✅ นี่คืองานที่ WebSocketถูกออกแบบมาให้ทำโดยเฉพาะ |

### 561.3 WebSocket คืออะไร

**WebSocket** คือโปรโตคอลการสื่อสารที่เปิด **การเชื่อมต่อ TCP เส้นเดียวค้างไว้
ตลอดเวลา (persistent connection)** ระหว่าง client กับ server โดยทั้งสองฝั่ง
สามารถ**ส่งข้อความหากันได้ทุกเมื่อโดยไม่ต้องรอกัน** — เรียกว่าเป็น
**full-duplex** (สื่อสารได้สองทางพร้อมกัน ต่างจาก HTTP ที่เป็น half-duplex
คือต้องผลัดกันพูด)

```
HTTP (Half-duplex, connection ใหม่ทุกครั้ง)          WebSocket (Full-duplex, connection เดียวค้างไว้)
──────────────────────────────────────           ──────────────────────────────────────
Browser ──req──> Server                            Browser ══════ handshake ══════> Server
Browser <──res── Server  (ปิด/รอใหม่)                        (เชื่อมต่อค้างไว้)
Browser ──req──> Server  (connection ใหม่)          Browser ──msg──> Server  (เมื่อไหร่ก็ได้)
Browser <──res── Server                            Browser <──msg── Server  (เมื่อไหร่ก็ได้ ไม่ต้องรอถาม)
   ...ทำซ้ำไปเรื่อย ๆ ถ้าอยากได้ข้อมูลใหม่                   ...ส่งกลับไปมาได้ตลอดอายุ connection เดียว
```

### 561.4 Handshake: WebSocket "เกิด" มาจาก HTTP Request ธรรมดา

จุดที่มือใหม่มักแปลกใจคือ WebSocket **ไม่ได้เริ่มต้นจากอากาศ** — การเชื่อมต่อ
ทุกครั้งเริ่มจาก HTTP request ปกติที่มี header พิเศษขอ "อัปเกรด" โปรโตคอล
ก่อนเสมอ เรียกขั้นตอนนี้ว่า **Handshake**:

```
1. Browser ส่ง HTTP Request ปกติ พร้อม header พิเศษ:

   GET /ws/notifications/ HTTP/1.1
   Host: example.com
   Upgrade: websocket
   Connection: Upgrade
   Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==
   Sec-WebSocket-Version: 13

2. ถ้า server รองรับ WebSocket จะตอบกลับด้วย status พิเศษ:

   HTTP/1.1 101 Switching Protocols
   Upgrade: websocket
   Connection: Upgrade
   Sec-WebSocket-Accept: s3pPLMBiTxaQ9kYGzzhZRbK+xOo=

3. หลังจากนี้ connection TCP เส้นเดิมจะ "เปลี่ยนโหมด" กลายเป็น WebSocket
   ทั้งสองฝั่งส่งข้อความ (frame) หากันได้อิสระจนกว่าฝ่ายใดฝ่ายหนึ่งจะปิด
```

นี่คือเหตุผลที่ Part นี้ต้องมาหลัง Part 004 (ที่แนะนำ WSGI/ASGI) เพราะ **Django
ธรรมดาที่รันด้วย WSGI ไม่มีทางจัดการ handshake แบบนี้ได้เลย** — WSGI ถูก
ออกแบบมาให้รับ request หนึ่ง แล้วคืน response หนึ่งแล้วจบ ไม่มีแนวคิดเรื่อง
"เชื่อมต่อค้างไว้แล้วส่งข้อความหลายรอบ" อยู่ในสเปกเลยแม้แต่น้อย

### 561.5 ทำไม Django (ที่ออกแบบมาให้เป็น Synchronous) ถึงต้องมี "โปรโตคอลใหม่"

Django ดั้งเดิมสร้างมาบนแนวคิด **1 request เข้า → 1 thread/process ประมวลผล
→ 1 response ออก → จบ** ซึ่งตรงกับสเปกของ WSGI เป๊ะ (ทบทวน Part 004 ข้อ
32.5) แต่ WebSocket ต้องการโมเดลที่ต่างไปโดยสิ้นเชิง: **1 connection คงอยู่
เป็นเวลานาน (อาจเป็นชั่วโมง) และต้องรับ-ส่งข้อความได้หลายครั้งระหว่างทาง**
ถ้าใช้โมเดล thread-per-request แบบ WSGI ตรง ๆ การเปิดค้างไว้แบบนี้จะกิน
thread ของเซิร์ฟเวอร์ไปเรื่อย ๆ จนหมดอย่างรวดเร็วเมื่อมีผู้ใช้หลักร้อยคน
เชื่อมต่อพร้อมกัน

**ASGI (Asynchronous Server Gateway Interface)** ที่ Part 004 ข้อ 32.6 แนะนำ
ไว้แบบผิวเผิน คือคำตอบของปัญหานี้ — ออกแบบมาให้รองรับทั้ง HTTP แบบเดิม
**และ** protocol ที่เป็น long-lived connection อย่าง WebSocket ในตัวเดียวกัน
โดยใช้ event loop (คล้ายกับที่ Node.js ใช้) แทนโมเดล thread-per-request
**Django Channels** คือ package ทางการที่ต่อยอดจาก ASGI ของ Django เพื่อทำให้
เขียน WebSocket consumer ได้ในรูปแบบที่คุ้นเคยคล้าย View — Part นี้ทั้งหมด
คือการเรียนรู้วิธีใช้ Channels

### 561.6 ตารางสรุป: WebSocket เทียบกับเทคนิค Real-time อื่นที่มีในเว็บ

| เทคนิค | ทิศทางการสื่อสาร | ใครเริ่ม | ใช้งานง่ายแค่ไหน | เหมาะกับ |
|---|---|---|---|---|
| HTTP Polling (Part 053 ข้อ 523.1) | Client → Server → Client | Client เท่านั้น | ง่ายที่สุด (attribute เดียว) | ข้อมูลเปลี่ยนไม่บ่อย, ไม่ต้อง real-time จริง |
| Server-Sent Events (SSE) | **Server → Client ทางเดียว** | Client เปิด แล้ว server ส่งได้เอง | ปานกลาง (`EventSource` built-in ของ browser) | Feed แจ้งเตือนทางเดียว, live score, log streaming |
| **WebSocket** | **Client ↔ Server สองทาง** | ทั้งสองฝ่ายส่งได้ตลอดเวลา | สูงกว่า (ต้องมี Channels/ASGI) | Chat, Multiplayer, Collaborative editing, Dashboard สองทาง |

> **ข้อสังเกตสำหรับหลักสูตรนี้**: ปุ่มถูกใจใน Part 053 ข้อ 522.4 **ไม่จำเป็น
> ต้องใช้ WebSocket** เพราะผู้ใช้เป็นคนกดเอง (HTMX ก็เพียงพอและง่ายกว่ามาก)
> แต่ "แจ้งเตือนคอมเมนต์ใหม่แบบไม่ต้องกดอะไรเลย" ในขั้นตอนที่ 566 คืองานที่
> WebSocket เหมาะสมที่สุด เพราะต้องให้ **server เป็นฝ่ายเริ่มพูดก่อน** — นี่
> คือหลักการเลือกเครื่องมือที่มืออาชีพต้องแยกแยะให้ออก ไม่ใช่ใช้ WebSocket
> กับทุกอย่างเพียงเพราะมันดูล้ำสมัยกว่า

---

## ขั้นตอนที่ 562: ติดตั้ง Django Channels และตั้งค่า ASGI ต่อยอดจาก Part 004

### 562.1 ทบทวนไฟล์ `config/asgi.py` จาก Part 004

เปิดไฟล์ `config/asgi.py` ที่ Django สร้างให้ตั้งแต่ Part 004 ข้อ 32.6 — ไฟล์นี้
ถูกสร้างมาตั้งแต่ `startproject` แต่**ไม่เคยถูกใช้งานจริงเลยจนถึงตอนนี้**
เพราะ `manage.py runserver` ที่ผ่านมาทั้งหมดใช้โหมด WSGI เป็นค่าเริ่มต้น และ
`settings.py` มีแค่ `WSGI_APPLICATION` ไม่มี `ASGI_APPLICATION`:

```python
# config/asgi.py (จาก Part 004 — ยังไม่มีการแก้ไข)
import os

from django.core.asgi import get_asgi_application

os.environ.setdefault("DJANGO_SETTINGS_MODULE", "config.settings")

application = get_asgi_application()
```

`get_asgi_application()` คืน ASGI callable ที่รองรับแค่ **HTTP** เท่านั้น
(เทียบเท่าความสามารถของ `get_wsgi_application()` แต่พูดภาษา ASGI) — ยังไม่
รู้จัก WebSocket เลยแม้แต่น้อย Part นี้คือจุดที่เราจะ**อัปเกรด**ไฟล์นี้ให้
รองรับทั้งสองโปรโตคอลพร้อมกัน

### 562.2 ติดตั้ง Package ที่จำเป็น

```bash
# ตรวจสอบให้แน่ใจว่า (venv) ยัง active อยู่ก่อนเสมอ (ทบทวนกฎเหล็กจาก Part 001)
pip install "channels>=4.1,<5" "daphne>=4.1,<5" "channels_redis>=4.2,<5"

# บันทึก dependency ใหม่ลง requirements.txt ทันที (ทบทวนกฎจาก Part 001 ข้อ 9.3)
pip freeze > requirements.txt
```

| Package | หน้าที่ |
|---|---|
| `channels` | Core library ที่เพิ่มความสามารถ WebSocket (และ protocol อื่น) ให้ Django ผ่าน ASGI |
| `daphne` | ASGI server อ้างอิงของทีม Django เอง ใช้รัน `runserver` แบบรองรับ WebSocket ตอนพัฒนา และ deploy จริงได้ด้วย (ขั้นตอนที่ 569) |
| `channels_redis` | Channel Layer backend ที่ใช้ Redis เป็นตัวกลางส่งข้อความข้าม process (ขั้นตอนที่ 565) |

### 562.3 เพิ่ม `INSTALLED_APPS` — ลำดับสำคัญมาก

```python
# config/settings.py
INSTALLED_APPS = [
    "daphne",                          # ← ต้องอยู่บนสุด ก่อน django.contrib.staticfiles เสมอ
    "django.contrib.admin",
    "django.contrib.auth",
    "django.contrib.contenttypes",
    "django.contrib.sessions",
    "django.contrib.messages",
    "django.contrib.staticfiles",
    "channels",                        # ← เพิ่มเข้ามาใหม่
    # ...แอปของโปรเจกต์เอง เช่น "blog", "accounts" ตามเดิม
    "realtime",                        # ← แอปใหม่ที่จะสร้างในข้อถัดไป
]
```

**เหตุผลที่ `"daphne"` ต้องอยู่บนสุด**: เมื่อ Django โหลด app registry มันจะ
เช็คว่ามี app `daphne` อยู่ใน `INSTALLED_APPS` หรือไม่ ถ้ามี Django จะ
**แทนที่ command `runserver` เดิม (ที่ใช้ WSGI dev server) ด้วยเวอร์ชันของ
Daphne โดยอัตโนมัติ** ทำให้ `python manage.py runserver` ตอนพัฒนารองรับทั้ง
HTTP และ WebSocket ในพอร์ตเดียวกันทันที โดยไม่ต้องรันเซิร์ฟเวอร์แยกสองตัว
ต้องอยู่ **ก่อน** `django.contrib.staticfiles` ไม่เช่นนั้นการแทนที่ command
จะไม่เกิดขึ้น (นี่คือกฎที่เอกสารทางการของ Channels ระบุไว้ชัดเจน)

### 562.4 สร้างแอป `realtime` สำหรับเก็บ Consumer และ Routing

ต่อยอดจากรูปแบบ Project vs App ที่เรียนใน Part 004 ข้อ 37 — เราจะสร้างแอปใหม่
ชื่อ `realtime` ขึ้นมาเก็บโค้ดที่เกี่ยวกับ WebSocket โดยเฉพาะ แยกออกจาก `blog`
เพื่อให้ชัดเจนว่านี่คือ "ชั้นการสื่อสารแบบ real-time" ไม่ใช่ business logic
ของบล็อก:

```bash
python manage.py startapp realtime
```

ลบไฟล์ `realtime/models.py`, `realtime/admin.py`, `realtime/migrations/`
ทิ้งได้เลยถ้าไม่มี Model ของตัวเอง (แอปนี้จะไม่เก็บข้อมูลลงฐานข้อมูลโดยตรง
เพียงแค่ทำหน้าที่เป็นชั้นสื่อสาร real-time เท่านั้น)

### 562.5 ตั้งค่า `ASGI_APPLICATION`

```python
# config/settings.py
ASGI_APPLICATION = "config.asgi.application"
```

นี่คือ setting คู่ขนานกับ `WSGI_APPLICATION` ที่ Part 004 ข้อ 33.7 อธิบายไว้ —
บอก Channels ว่าตัวแปร ASGI callable หลักของโปรเจกต์อยู่ที่ไหน

### 562.6 อัปเกรด `config/asgi.py` ให้รองรับทั้ง HTTP และ WebSocket

```python
# config/asgi.py (เวอร์ชันอัปเกรดสำหรับ Channels)
import os

from django.core.asgi import get_asgi_application

os.environ.setdefault("DJANGO_SETTINGS_MODULE", "config.settings")

# สำคัญมาก: ต้องเรียก get_asgi_application() ก่อน import อะไรก็ตามที่แตะ
# Model หรือ App Registry (เช่น routing.py, consumers.py) เพราะฟังก์ชันนี้
# เป็นตัวที่เรียก django.setup() ให้ครบก่อน ถ้า import routing.py มาก่อน
# บรรทัดนี้ จะเจอ AppRegistryNotReady error ทันที
django_asgi_app = get_asgi_application()

from channels.auth import AuthMiddlewareStack           # noqa: E402
from channels.routing import ProtocolTypeRouter, URLRouter  # noqa: E402

import realtime.routing  # noqa: E402

application = ProtocolTypeRouter({
    # request HTTP ธรรมดาทั้งหมดยังคงไปที่ Django app เดิมทุกประการ
    # ไม่มีอะไรเปลี่ยนแปลงสำหรับ View, Template, DRF ที่เขียนมาตั้งแต่ Phase 1-5
    "http": django_asgi_app,

    # request ที่เป็น WebSocket handshake เท่านั้นที่จะถูกส่งเข้าทางนี้
    "websocket": AuthMiddlewareStack(
        URLRouter(realtime.routing.websocket_urlpatterns)
    ),
})
```

อธิบายโครงสร้างนี้ทีละชั้น:

| ชั้น | หน้าที่ |
|---|---|
| `ProtocolTypeRouter` | ประตูแรกสุด แยก request ตาม**ชนิดโปรโตคอล** (`http` หรือ `websocket`) ก่อนส่งต่อ |
| `django_asgi_app` (`"http"`) | HTTP ทุกเส้นทางยังคงวิ่งผ่าน Django ตามปกติ 100% — View, URLconf, Middleware ทั้งหมดจาก Phase 1-6 ทำงานเหมือนเดิมทุกประการ ไม่ต้องแก้อะไรเลย |
| `AuthMiddlewareStack` (`"websocket"`) | เติม `scope['user']` ให้ WebSocket consumer เข้าถึงผู้ใช้ที่ล็อกอินอยู่ได้ (เจาะลึกในขั้นตอนที่ 567) |
| `URLRouter` | ตัวจับคู่ URL สำหรับ WebSocket โดยเฉพาะ ทำหน้าที่คล้าย `ROOT_URLCONF` ของ Part 004 ข้อ 33.7 แต่แยกเป็นคนละระบบกันโดยสิ้นเชิง |

### 562.7 สร้างไฟล์ `realtime/routing.py` (ยังว่างเปล่าไปก่อน)

```python
# realtime/routing.py
from django.urls import path

websocket_urlpatterns = [
    # จะเริ่มเพิ่ม path จริงในขั้นตอนที่ 563 เป็นต้นไป
]
```

ต้องมีไฟล์นี้ก่อน เพราะ `config/asgi.py` ในข้อ 562.6 `import realtime.routing`
ไว้แล้ว — ถ้าไม่มีไฟล์นี้ `runserver` จะล้มเหลวทันทีตั้งแต่ตอนเริ่มด้วย
`ModuleNotFoundError`

### 562.8 ทดสอบว่า Daphne เข้ามาแทนที่ `runserver` สำเร็จ

```bash
python manage.py runserver
```

สังเกต output บรรทัดแรก — ถ้าเห็นข้อความประมาณนี้ แสดงว่าตั้งค่าสำเร็จ:

```
Watching for file changes with StatReloader
Performing system checks...

System check identified no issues (0 silenced).
January 15, 2026 - 10:02:31
Django version 5.1.2, using settings 'config.settings'
Starting ASGI/Daphne version 4.1.2 development server at http://127.0.0.1:8000/
Quit the server with CONTROL-C.
```

ข้อความ **"Starting ASGI/Daphne version ... development server"** (แทนที่จะ
เป็น "Starting development server" แบบ WSGI เดิมของ Part 004 ข้อ 34.1) คือ
หลักฐานว่า `INSTALLED_APPS` และ `ASGI_APPLICATION` ถูกตั้งค่าถูกต้องแล้ว
ลองเปิด `http://127.0.0.1:8000/` — ทุกหน้าเว็บที่เขียนมาตั้งแต่ Phase 1-6
ยังคงทำงานเหมือนเดิมทุกประการ เพราะ HTTP request ทั้งหมดยังคงถูกส่งเข้า
`django_asgi_app` ตามที่ตั้งค่าไว้ในข้อ 562.6

### 562.9 โครงสร้างไฟล์ทั้งหมดหลังขั้นตอนนี้

```
django-mastery-course/
├── config/
│   ├── asgi.py          ← แก้ไข (ProtocolTypeRouter)
│   ├── settings.py      ← แก้ไข (INSTALLED_APPS, ASGI_APPLICATION)
│   ├── urls.py
│   └── wsgi.py          ← ไม่แตะ (ยังใช้สำหรับ deploy แบบ HTTP-only ถ้าต้องการ)
├── blog/
│   └── ...              ← ไม่แตะเลยในขั้นตอนนี้
├── realtime/
│   ├── __init__.py
│   ├── apps.py
│   ├── consumers.py     ← จะเขียนในขั้นตอนที่ 563
│   ├── routing.py       ← สร้างแล้ว (ยังว่าง)
│   └── tests/           ← จะเขียนในขั้นตอนที่ 568
└── requirements.txt      ← อัปเดตแล้ว
```

---

## ขั้นตอนที่ 563: `WebsocketConsumer` เบื้องต้น (Synchronous)

### 563.1 Consumer คืออะไร — เทียบกับ View

ใน Django ปกติ **View** คือฟังก์ชัน (หรือ class) ที่รับ `HttpRequest` หนึ่งตัว
แล้วคืน `HttpResponse` หนึ่งตัว จบในครั้งเดียว (ทบทวน Part 001 ข้อ 3.3)
**Consumer** คือแนวคิดคู่ขนานกันสำหรับ WebSocket — แต่แทนที่จะเป็น
"รับ 1 คืน 1 แล้วจบ" Consumer คือ**object ที่มีอายุยืนยาวตลอดการเชื่อมต่อ**
และมี method หลักสามตัวที่ Channels เรียกให้อัตโนมัติตามเหตุการณ์:

| Method | ถูกเรียกเมื่อไหร่ | เทียบเท่า View |
|---|---|---|
| `connect()` | ทันทีที่ client เชื่อมต่อเข้ามา (หลัง handshake ข้อ 561.4 สำเร็จ) | จุดเริ่มต้นของ View function |
| `receive()` | ทุกครั้งที่ client ส่งข้อความมา (เกิดขึ้นได้**หลายครั้ง**ตลอดอายุ connection) | ไม่มีเทียบเท่าตรง ๆ ใน View ปกติ (View รับ input แค่ครั้งเดียวตอนเริ่ม) |
| `disconnect()` | เมื่อ connection ถูกปิด (client ปิดเอง, เน็ตหลุด, หรือ server สั่งปิด) | ไม่มีเทียบเท่า (View ไม่มีแนวคิด "จบการเชื่อมต่อ" แยกจาก return) |

### 563.2 เขียน Consumer แรก: Echo Server

Consumer ที่ง่ายที่สุดเท่าที่จะเป็นไปได้คือ **Echo Server** — ส่งข้อความ
อะไรมา ก็ส่งข้อความเดิมกลับไปทันที เหมาะที่สุดสำหรับทำความเข้าใจวงจรชีวิต
ของ Consumer ก่อนไปทำอะไรซับซ้อนกว่านี้:

```python
# realtime/consumers.py
from channels.generic.websocket import WebsocketConsumer


class EchoConsumer(WebsocketConsumer):
    """Consumer สาธิตแบบ synchronous ที่ง่ายที่สุด: ส่งอะไรมา ส่งกลับไปเหมือนเดิม"""

    def connect(self):
        # ต้องเรียก self.accept() เสมอ ไม่เช่นนั้น Channels จะปฏิเสธ handshake
        # โดยอัตโนมัติ (เทียบเท่าการคืน HTTP 403 ใน View ปกติ)
        self.accept()

    def disconnect(self, close_code):
        # ไม่มีอะไรต้องทำความสะอาดสำหรับ Consumer ง่าย ๆ ตัวนี้
        # (Consumer ที่ join กลุ่มไว้ต้อง group_discard ที่นี่ — ดูขั้นตอนที่ 565)
        pass

    def receive(self, text_data=None, bytes_data=None):
        # text_data คือข้อความแบบ string (กรณีส่งเป็น text frame)
        # bytes_data คือข้อมูลแบบ binary (กรณีส่งเป็น binary frame เช่น ไฟล์)
        self.send(text_data=text_data)
```

`WebsocketConsumer` (ไม่มีคำว่า `Async` นำหน้า) คือ base class แบบ
**synchronous** — เขียน method ธรรมดา (`def` ไม่ใช่ `async def`) เหมือน View
ทั่วไปที่คุ้นเคยมาตั้งแต่ Part 004 ทำให้เป็นจุดเริ่มต้นที่เข้าใจง่ายที่สุด
สำหรับผู้ที่ยังไม่คุ้นเคยกับ `async`/`await` ของ Python

### 563.3 เชื่อม Consumer เข้ากับ URL ผ่าน `routing.py`

```python
# realtime/routing.py
from django.urls import path

from . import consumers

websocket_urlpatterns = [
    path("ws/echo/", consumers.EchoConsumer.as_asgi()),
]
```

สังเกตว่าใช้ `path()` ตัวเดียวกับที่เรียนใน Part 004 ข้อ 32.4 ได้เลย (Channels
รองรับทั้ง `path()` และ `re_path()`) จุดที่ต่างจาก View คือ **`.as_asgi()`**
แทนที่จะเป็น `.as_view()` ของ Class-Based View — นี่คือ classmethod ที่แปลง
Consumer class ให้กลายเป็น ASGI application ที่ `URLRouter` เรียกใช้ได้

> **ข้อควรสังเกต**: URL ของ Consumer มักขึ้นต้นด้วย `ws/` เป็นธรรมเนียม
> (ไม่ใช่กฎบังคับ) เพื่อแยกให้เห็นชัดเจนจาก URL ของ View ปกติเวลาอ่านโค้ด
> หรือดู log ของ reverse proxy (ขั้นตอนที่ 569)

### 563.4 ทดสอบด้วย Browser Console — ไม่ต้องเขียน Frontend เต็มรูปแบบก่อน

รัน `python manage.py runserver` แล้วเปิด Developer Console ของเบราว์เซอร์
(`F12`) บนหน้าใดก็ได้ของเว็บไซต์ แล้วรันโค้ด JavaScript นี้ทีละบรรทัด:

```javascript
// เปิดการเชื่อมต่อ WebSocket ไปยัง Consumer ที่เพิ่งเขียน
const socket = new WebSocket('ws://127.0.0.1:8000/ws/echo/');

socket.onopen = () => {
    console.log('เชื่อมต่อสำเร็จ!');
    socket.send('สวัสดี Django Channels');
};

socket.onmessage = (event) => {
    console.log('ได้รับข้อความกลับ:', event.data);
    // ควรเห็น: "ได้รับข้อความกลับ: สวัสดี Django Channels"
};

socket.onclose = (event) => {
    console.log('การเชื่อมต่อถูกปิด รหัส:', event.code);
};
```

ถ้าเห็นข้อความ `ได้รับข้อความกลับ: สวัสดี Django Channels` ใน console แสดงว่า
วงจร `connect()` → `receive()` → `send()` ทำงานถูกต้องครบวงจรแล้ว —
Consumer แรกของคุณทำงานได้จริง!

### 563.5 ข้อจำกัดของ `WebsocketConsumer` แบบ Synchronous

`WebsocketConsumer` ทำงานได้ถูกต้อง แต่มีข้อจำกัดสำคัญที่ต้องเข้าใจก่อนใช้
งานจริง: **แต่ละ connection ที่เข้ามาจะถูกรันบน thread แยกต่างหากจาก
thread pool ของ ASGI server** (Daphne จัดสรร thread pool ไว้จำนวนจำกัดสำหรับ
รัน sync consumer โดยเฉพาะ ผ่านกลไกของ `asgiref`) เมื่อมีผู้ใช้เชื่อมต่อ
WebSocket พร้อมกันหลักพันคน จำนวน thread ที่ต้องใช้ก็เพิ่มตามไปด้วยเป็น
เส้นตรง ซึ่งเป็นข้อจำกัดเดียวกับที่ WSGI มีมาแต่เดิม (ทบทวนข้อ 561.5) —
นี่คือเหตุผลที่ขั้นตอนถัดไปจะแนะนำ `AsyncWebsocketConsumer` ซึ่งแก้ปัญหานี้
ได้อย่างตรงจุด

| ประเด็น | `WebsocketConsumer` (Sync) | `AsyncWebsocketConsumer` (ขั้นตอนที่ 564) |
|---|---|---|
| เขียน method แบบ | `def` ธรรมดา | `async def` |
| ใช้ทรัพยากรต่อ 1 connection | 1 thread จาก thread pool ที่มีจำกัด | 1 coroutine บน event loop เดียว (เบากว่ามาก) |
| เรียก Django ORM ได้ตรง ๆ ไหม | ✅ ได้ตรง ๆ (ทำงานแบบ sync อยู่แล้ว) | ❌ ต้องห่อด้วย `database_sync_to_async` (ขั้นตอนที่ 564) |
| เหมาะกับ Connection จำนวนมาก | ไม่เหมาะ (thread หมดเร็ว) | เหมาะมาก (ออกแบบมาสำหรับสิ่งนี้โดยเฉพาะ) |

---

## ขั้นตอนที่ 564: `AsyncWebsocketConsumer` — เหตุผลที่ควรใช้ Async ในงาน Real-time

### 564.1 ทำไม "Async" ถึงเหมาะกับ WebSocket เป็นพิเศษ

ลักษณะเฉพาะของ connection แบบ WebSocket คือ **ส่วนใหญ่ของเวลามันจะ "ว่าง"
(idle)** — connection เปิดค้างไว้เป็นนาทีหรือชั่วโมง แต่ข้อความจริง ๆ อาจถูก
ส่งเพียงไม่กี่ครั้งต่อนาที การจัดสรร 1 thread ของระบบปฏิบัติการให้กับแต่ละ
connection ที่ "ว่าง" แบบนี้ (ตามที่ `WebsocketConsumer` ทำในขั้นตอนที่ 563)
จึงสิ้นเปลืองมาก เพราะ thread คือทรัพยากรราคาแพง (แต่ละ thread กิน memory
หลัก KB และมี context-switching overhead)

**Async/await** (จาก module `asyncio` ของ Python ที่แนะนำให้ทบทวนใน Part
002) ใช้ **event loop เดียว** จัดการ connection ที่ "ว่าง" หลายพันตัวพร้อมกัน
ได้ โดยสลับไปทำงานอื่นทันทีเมื่อ connection ใดยังไม่มีอะไรให้ทำ (กำลังรอ
`await`) แล้วค่อยกลับมาทำงานต่อเมื่อมีข้อมูลเข้ามาจริง — นี่คือสาเหตุที่
เอกสารทางการของ Channels แนะนำให้ **ใช้ `AsyncWebsocketConsumer` เป็นค่า
เริ่มต้นสำหรับ Consumer ใหม่ทุกตัวในงานจริง** เว้นแต่มีเหตุผลเฉพาะที่ต้องใช้
sync (เช่น เรียก library ภายนอกที่รองรับแค่ sync)

### 564.2 เขียน Consumer เดิมใหม่แบบ Async

```python
# realtime/consumers.py
from channels.generic.websocket import AsyncWebsocketConsumer, WebsocketConsumer


class EchoConsumer(WebsocketConsumer):
    """เวอร์ชัน Sync จากขั้นตอนที่ 563 — เก็บไว้เพื่อเทียบเคียง"""

    def connect(self):
        self.accept()

    def disconnect(self, close_code):
        pass

    def receive(self, text_data=None, bytes_data=None):
        self.send(text_data=text_data)


class AsyncEchoConsumer(AsyncWebsocketConsumer):
    """เวอร์ชัน Async — ทำงานเหมือนกันทุกประการ ต่างแค่ async/await"""

    async def connect(self):
        await self.accept()

    async def disconnect(self, close_code):
        pass

    async def receive(self, text_data=None, bytes_data=None):
        await self.send(text_data=text_data)
```

โครงสร้างเหมือนกันทุกประการกับข้อ 563.2 เพียงแค่เติม `async` หน้า `def`
ทุกตัว และ `await` หน้าทุก method ของ Channels ที่เป็น coroutine
(`self.accept()`, `self.send()`) — นี่คือความตั้งใจของทีม Channels ที่ทำให้
API ทั้งสองฝั่งดูคล้ายกันที่สุดเท่าที่จะทำได้ เพื่อลด learning curve

### 564.3 ปัญหาที่ต้องระวัง: Django ORM เป็น Synchronous โดยธรรมชาติ

ปัญหาใหญ่ที่สุดของการย้ายมาใช้ `AsyncWebsocketConsumer` คือ **Django ORM
(QuerySet, `.save()`, `.filter()` ฯลฯ) ยังคงเป็นโค้ด synchronous ล้วน ๆ**
(ทบทวน Part 011-015) ถ้าเรียกมันตรง ๆ จากใน `async def` จะเจอ exception
ทันที:

```python
# ❌ ผิด — จะได้ SynchronousOnlyOperation exception ทันที
class BadAsyncConsumer(AsyncWebsocketConsumer):
    async def receive(self, text_data=None, bytes_data=None):
        count = Post.objects.count()  # เรียก ORM ตรง ๆ จาก async context ไม่ได้!
        await self.send(text_data=str(count))
```

ทางแก้คือห่อโค้ดที่เรียก ORM ด้วย **`database_sync_to_async`** จาก
`channels.db` ซึ่งจะรันฟังก์ชันนั้นในเธรดแยกที่ไม่มี event loop ทำงานอยู่
(ผ่านกลไกเดียวกับที่ `sync_to_async` ของ `asgiref` ใช้) แล้วส่งผลลัพธ์กลับมา
ให้ async context ต่อได้อย่างปลอดภัย:

```python
# realtime/consumers.py
from channels.db import database_sync_to_async
from channels.generic.websocket import AsyncWebsocketConsumer

from blog.models import Post


class AsyncEchoConsumer(AsyncWebsocketConsumer):
    async def connect(self):
        await self.accept()

    async def receive(self, text_data=None, bytes_data=None):
        # ✅ ถูกต้อง — ห่อการเรียก ORM ด้วย database_sync_to_async เสมอ
        post_count = await database_sync_to_async(Post.objects.count)()
        await self.send(
            text_data=f"มีบทความทั้งหมด {post_count} เรื่อง (คุณพิมพ์ว่า: {text_data})"
        )
```

`database_sync_to_async` รับฟังก์ชัน (ไม่ใช่ผลลัพธ์) เป็นพารามิเตอร์ แล้ว
คืนฟังก์ชัน async ตัวใหม่ที่เรียกได้ด้วย `await` — สังเกตวงเล็บสองชั้น:
`database_sync_to_async(Post.objects.count)()` วงเล็บแรกคือการ "ห่อ" ฟังก์ชัน
`Post.objects.count` วงเล็บที่สองคือการ "เรียก" ฟังก์ชันที่ห่อแล้วนั้นจริง ๆ

### 564.4 ตารางสรุปกฎการเรียก Code แบบ Sync จาก Async Consumer

| สถานการณ์ | วิธีจัดการที่ถูกต้อง |
|---|---|
| เรียก Django ORM (QuerySet, `.save()`, signal ที่ trigger query) | ห่อด้วย `channels.db.database_sync_to_async` |
| เรียกฟังก์ชัน sync อื่นที่ไม่เกี่ยวกับฐานข้อมูล (เช่น เขียนไฟล์, เรียก library sync) | ห่อด้วย `asgiref.sync.sync_to_async` (ตัวทั่วไปกว่า `database_sync_to_async`) |
| เรียก Channel Layer (`group_send`, `group_add`) | ไม่ต้องห่อ — API ของ `channel_layer` เป็น async-native อยู่แล้ว (ขั้นตอนที่ 565) |
| เรียก `self.accept()`, `self.send()`, `self.close()` ของ Consumer เอง | ไม่ต้องห่อ — เป็น method ของ `AsyncWebsocketConsumer` ที่เป็น async อยู่แล้ว |

> **กฎเหล็กของขั้นตอนนี้**: ถ้าเห็น error `SynchronousOnlyOperation` หรือ
> `You cannot call this from an async context` ขึ้นตอนรัน Consumer แบบ
> Async ให้สงสัยไว้ก่อนเลยว่ามีการเรียก Django ORM (หรือโค้ด sync อื่น) ตรง ๆ
> โดยไม่ได้ห่อด้วย `database_sync_to_async`/`sync_to_async`

---

## ขั้นตอนที่ 565: Channel Layers ด้วย Redis — ส่งข้อความข้าม Consumer

### 565.1 ปัญหา: Consumer แต่ละตัว "คุยกันเองไม่ได้"

Consumer ที่เขียนมาจนถึงตอนนี้ (Echo) ทำงานแบบ "ตัวใครตัวมัน" — client A
เชื่อมต่อเข้ามาก็ได้ instance ของ Consumer 1 ตัว client B เชื่อมต่อเข้ามาก็
ได้อีก instance หนึ่งแยกกันอิสระ **ไม่มีทางที่ Consumer ของ client A จะส่ง
ข้อความไปหา client B ได้โดยตรง** ทั้งที่นี่คือความสามารถพื้นฐานที่สุดที่งาน
real-time เกือบทุกชนิดต้องการ (ห้องแชทที่มีคนมากกว่า 1 คน, แจ้งเตือนที่ต้อง
กระจายไปหาผู้ใช้หลายคนพร้อมกัน)

**Channel Layer** คือกลไกกลางที่ Channels เตรียมไว้แก้ปัญหานี้โดยเฉพาะ —
เปรียบเสมือน "กระดานประกาศ" ที่ทุก Consumer เข้าถึงร่วมกันได้ ไม่ว่าจะรันอยู่
บน process หรือเครื่องเดียวกันหรือไม่ก็ตาม

```
Consumer A (client A)          Channel Layer (Redis)          Consumer B (client B)
       │                              │                              │
       │  group_add("chat_room1")    │                              │
       │ ───────────────────────────>│                              │
       │                              │   group_add("chat_room1")   │
       │                              │<─────────────────────────── │
       │                              │                              │
       │  group_send("chat_room1", msg)                              │
       │ ───────────────────────────>│                              │
       │                              │──── กระจายไปทุก channel ────>│
       │                              │      ที่ join กลุ่มนี้ไว้      │
       │  (ตัวเองก็ได้รับด้วยถ้า join ไว้)  │                        ▼
       ▼                              │                    ได้รับ msg ทาง WebSocket
```

### 565.2 ติดตั้งและรัน Redis

Redis คือฐานข้อมูล in-memory ความเร็วสูงที่ `channels_redis` ใช้เป็น backend
เริ่มต้นสำหรับ Channel Layer (ติดตั้ง `channels_redis` ไว้แล้วในขั้นตอนที่
562.2):

```bash
# วิธีที่แนะนำที่สุดสำหรับเครื่อง dev: รันผ่าน Docker (ไม่ต้องติดตั้งอะไรลงเครื่องจริง)
docker run -d --name redis-dev -p 6379:6379 redis:7-alpine

# หรือติดตั้งลงเครื่องโดยตรง
# macOS
brew install redis
brew services start redis

# Ubuntu/Debian
sudo apt install redis-server -y
sudo systemctl enable --now redis-server

# ทดสอบว่า Redis ทำงานอยู่จริง
redis-cli ping
# ควรได้ผลลัพธ์: PONG
```

> **หมายเหตุ**: หลักสูตรนี้จะสอนการรัน Redis ผ่าน Docker Compose ร่วมกับ
> PostgreSQL อย่างเป็นระบบใน **Part 087** — ตอนนี้ใช้วิธีติดตั้งตรงหรือ
> `docker run` แบบง่ายไปก่อนเพื่อโฟกัสที่ Channels

### 565.3 ตั้งค่า `CHANNEL_LAYERS`

```python
# config/settings.py
CHANNEL_LAYERS = {
    "default": {
        "BACKEND": "channels_redis.core.RedisChannelLayer",
        "CONFIG": {
            "hosts": [("127.0.0.1", 6379)],
        },
    },
}
```

โครงสร้างนี้จงใจให้คล้ายกับ `DATABASES` และ `CACHES` ที่เรียนมาก่อนหน้านี้ —
คีย์ `"default"` คือชื่อของ Channel Layer หลัก (รองรับหลายตัวพร้อมกันได้เช่น
เดียวกับ multiple databases) `BACKEND` ระบุ class ที่ใช้เชื่อม `CONFIG`
ระบุพารามิเตอร์เชื่อมต่อ (`hosts` คือ list ของ `(host, port)` ของ Redis
รองรับหลาย instance เพื่อกระจายโหลดในงานจริง)

### 565.4 สาม Method หลักของ Channel Layer

| Method | หน้าที่ |
|---|---|
| `group_add(group_name, channel_name)` | ให้ channel ของ consumer ปัจจุบัน (`self.channel_name`) "เข้าร่วม" กลุ่มที่ชื่อ `group_name` |
| `group_discard(group_name, channel_name)` | ให้ channel ปัจจุบัน "ออกจาก" กลุ่ม (ต้องเรียกใน `disconnect()` เสมอ ไม่เช่นนั้นกลุ่มจะเต็มไปด้วย channel ที่ตายไปแล้ว) |
| `group_send(group_name, message_dict)` | ส่ง `message_dict` ไปหา**ทุก channel**ที่อยู่ในกลุ่มนั้น ณ ขณะนั้น |

จุดสำคัญที่มือใหม่มักพลาด: `message_dict` ที่ส่งให้ `group_send` **ต้องมีคีย์
ชื่อ `"type"` เสมอ** ค่าของ `"type"` คือชื่อของ **method ที่จะถูกเรียกใน
Consumer ผู้รับ** (Channels แปลง `.` เป็น `_` ให้อัตโนมัติถ้ามี แต่ธรรมเนียม
ที่นิยมที่สุดคือตั้งชื่อแบบ `snake_case` ตรง ๆ ไปเลย)

### 565.5 ตัวอย่างเต็ม: ห้องแชทที่มีหลายคนพูดคุยกันได้จริง

```python
# realtime/consumers.py (เพิ่มเข้าไป)
import json

from channels.generic.websocket import AsyncWebsocketConsumer


class ChatConsumer(AsyncWebsocketConsumer):
    async def connect(self):
        # ดึงค่า room_name จาก URL (เช่น /ws/chat/general/ -> room_name = "general")
        self.room_name = self.scope["url_route"]["kwargs"]["room_name"]
        self.room_group_name = f"chat_{self.room_name}"

        await self.channel_layer.group_add(self.room_group_name, self.channel_name)
        await self.accept()

    async def disconnect(self, close_code):
        # ต้องออกจากกลุ่มเสมอ ไม่เช่นนั้น Redis จะสะสม channel ที่ตายแล้วไปเรื่อย ๆ
        await self.channel_layer.group_discard(self.room_group_name, self.channel_name)

    async def receive(self, text_data=None, bytes_data=None):
        data = json.loads(text_data)

        # กระจายข้อความไปให้ "ทุกคนในห้อง" รวมถึงตัวเองด้วย ผ่าน Channel Layer
        # ไม่ใช่ self.send() ตรง ๆ (ซึ่งจะส่งกลับหาแค่ตัวเองเท่านั้น)
        await self.channel_layer.group_send(
            self.room_group_name,
            {
                "type": "chat_message",       # ← จะไปเรียก method ชื่อ chat_message ด้านล่าง
                "message": data["message"],
                "sender": data.get("sender", "ไม่ระบุชื่อ"),
            },
        )

    async def chat_message(self, event):
        """ถูกเรียกโดย Channel Layer เมื่อมีข้อความใหม่เข้ากลุ่มนี้ (ไม่ได้ถูกเรียกโดย client โดยตรง)"""
        await self.send(text_data=json.dumps({
            "message": event["message"],
            "sender": event["sender"],
        }))
```

```python
# realtime/routing.py (อัปเดต)
from django.urls import path

from . import consumers

websocket_urlpatterns = [
    path("ws/echo/", consumers.EchoConsumer.as_asgi()),
    path("ws/echo-async/", consumers.AsyncEchoConsumer.as_asgi()),
    path("ws/chat/<str:room_name>/", consumers.ChatConsumer.as_asgi()),
]
```

`self.scope["url_route"]["kwargs"]["room_name"]` คือวิธีที่ Consumer อ่านค่า
ที่ capture มาจาก URL pattern (`<str:room_name>`) — เทียบเท่ากับพารามิเตอร์
`room_name` ที่ View ฟังก์ชันปกติรับเข้ามาโดยตรง (ทบทวน Part 004 ข้อ 32.4)
เพียงแต่ Consumer ต้องขุดค่าออกมาจาก `self.scope` เอง

### 565.6 ทดสอบห้องแชทด้วย Browser Console สองแท็บ

เปิดเบราว์เซอร์ 2 แท็บ (หรือ 2 หน้าต่าง) แล้วรันโค้ดนี้ในแต่ละแท็บ (เปลี่ยน
`sender` ให้ต่างกัน):

```javascript
// แท็บที่ 1
const socket1 = new WebSocket('ws://127.0.0.1:8000/ws/chat/general/');
socket1.onmessage = (e) => console.log('แท็บ 1 ได้รับ:', JSON.parse(e.data));
socket1.onopen = () => socket1.send(JSON.stringify({message: 'สวัสดีทุกคน', sender: 'คนที่ 1'}));
```

```javascript
// แท็บที่ 2 (เปิดคนละแท็บ)
const socket2 = new WebSocket('ws://127.0.0.1:8000/ws/chat/general/');
socket2.onmessage = (e) => console.log('แท็บ 2 ได้รับ:', JSON.parse(e.data));
```

เมื่อแท็บ 1 ส่งข้อความ ทั้งแท็บ 1 และแท็บ 2 ควรเห็นข้อความ **"สวัสดีทุกคน"**
ปรากฏใน console พร้อมกัน — นี่คือหลักฐานว่า Channel Layer กระจายข้อความข้าม
connection ที่แยกกันอิสระได้สำเร็จ (ถ้าทดสอบใน 2 browser process หรือ 2
เครื่องคนละเครื่องที่ชี้ไปที่ Redis ตัวเดียวกัน ก็จะได้ผลลัพธ์เดียวกันทุก
ประการ — นี่คือพลังที่แท้จริงของ Channel Layer เมื่อ deploy จริงบนหลาย
process)

---

## ขั้นตอนที่ 566: สร้างระบบแจ้งเตือนคอมเมนต์ใหม่แบบ Real-time จริง

### 566.1 เป้าหมาย: ผูก Signal จาก Part 019 เข้ากับ Channel Layer

ตอนนี้เรามีชิ้นส่วนครบสำหรับแก้โจทย์ที่ตั้งไว้ในขั้นตอนที่ 561.1: **"แจ้งเตือน
แอดมินทันทีที่มีคนโพสต์คอมเมนต์ใหม่ โดยแอดมินไม่ต้องกดอะไรเลย"** ใช้ Model
`Comment` จาก Part 026 ข้อ 251 ตรง ๆ (ไม่ต้องแก้ Model แม้แต่บรรทัดเดียว)
และใช้แนวคิด **Signal** จาก Part 019 ข้อ 183 ที่เคยใช้ `post_save` สร้าง
`Profile` อัตโนมัติให้ `User` — คราวนี้จะใช้ `post_save` ของ `Comment` แทน
เพื่อส่งข้อความเข้า Channel Layer ทันทีที่มีการบันทึกคอมเมนต์ใหม่:

```
Browser (ผู้อ่านทั่วไป)                Django View                Channel Layer            Browser (แอดมิน)
      │                                    │                          │                          │
      │  POST คอมเมนต์ใหม่ (HTTP ปกติ         │                          │                          │
      │  หรือผ่าน HTMX แบบ Part 053)          │                          │                          │
      │ ──────────────────────────────────>│                          │                          │
      │                                    │  Comment.objects.create() │                          │
      │                                    │  ──> post_save signal ยิง │                          │
      │                                    │  ──> group_send("notifications", ...) ─────────────> │
      │                                    │                          │  (ผลักเข้าหา WebSocket    │
      │                                    │                          │   ของแอดมินที่เปิดค้างไว้)  │
      │  <── HTTP Response ปกติ (200) ──────│                          │                          ▼
      │                                    │                          │              เห็นแจ้งเตือนทันที
      │                                    │                          │              โดยไม่ต้องกดอะไรเลย
```

จุดสำคัญที่ควรสังเกต: **ผู้อ่านที่โพสต์คอมเมนต์ไม่ได้ยุ่งเกี่ยวกับ WebSocket
เลยแม้แต่น้อย** — เขาโพสต์ผ่าน HTTP ธรรมดา (จะเป็น View ปกติจาก Part 026
หรือผ่าน HTMX จาก Part 053 ก็ได้ผลลัพธ์เดียวกัน) WebSocket มีไว้สำหรับ**ฝั่ง
แอดมินที่รอรับการแจ้งเตือน**เท่านั้น — นี่คือรูปแบบการออกแบบที่พบบ่อยที่สุด
ในงานจริง: ไม่ใช่ทุก endpoint ต้องกลายเป็น WebSocket ทั้งหมด แต่เลือกใช้
เฉพาะจุดที่ต้องการให้ server เป็นฝ่ายเริ่มพูดก่อน

### 566.2 เขียน Signal Receiver ต่อยอดจากรูปแบบ Part 019

ต่อยอดโครงสร้างไฟล์จาก Part 019 ข้อ 184.3 (`signals.py` แยกไฟล์ + เชื่อมใน
`AppConfig.ready()`) มาใช้กับแอป `blog`:

```python
# blog/signals.py
from asgiref.sync import async_to_sync
from channels.layers import get_channel_layer
from django.db.models.signals import post_save
from django.dispatch import receiver

from .models import Comment


@receiver(post_save, sender=Comment, dispatch_uid="blog_notify_new_comment")
def notify_new_comment(sender, instance, created, **kwargs):
    """ทุกครั้งที่มี Comment ใหม่ถูกสร้าง ให้กระจายข้อความแจ้งเตือนผ่าน Channel Layer"""
    if not created:
        return  # แจ้งเตือนเฉพาะตอนสร้างใหม่เท่านั้น ไม่ใช่ทุกครั้งที่แก้ไข (ทบทวนกฎเหล็ก Part 019 ข้อ 183.3)

    channel_layer = get_channel_layer()
    if channel_layer is None:
        return  # เผื่อกรณี CHANNEL_LAYERS ยังไม่ถูกตั้งค่า (เช่น รัน management command บางตัวแยกต่างหาก)

    # signal receiver เป็นฟังก์ชัน synchronous ธรรมดา (ถูกเรียกจาก Comment.save()
    # ซึ่งเป็นโค้ด sync) แต่ channel_layer.group_send() เป็น coroutine (async)
    # จึงต้องแปลงด้วย async_to_sync ก่อนเรียกใช้ — ตรงข้ามกับ database_sync_to_async
    # ในขั้นตอนที่ 564 ที่แปลงจาก sync ไปหา async
    async_to_sync(channel_layer.group_send)(
        "notifications",
        {
            "type": "notify",
            "comment_id": instance.pk,
            "post_id": instance.post_id,
            "post_title": instance.post.title,
            "post_slug": instance.post.slug,
            "author": instance.author,
            "text_preview": instance.text[:80],
            "created_at": instance.created_at.isoformat(),
        },
    )
```

เชื่อม signal นี้ใน `AppConfig.ready()` ของแอป `blog` ตามรูปแบบเดียวกับ Part
019 ข้อ 184.4 เป๊ะ:

```python
# blog/apps.py
from django.apps import AppConfig


class BlogConfig(AppConfig):
    default_auto_field = "django.db.models.BigAutoField"
    name = "blog"

    def ready(self):
        import blog.signals  # noqa: F401
```

### 566.3 เขียน `NotificationConsumer`

```python
# realtime/consumers.py (เพิ่มเข้าไป)
import json

from channels.generic.websocket import AsyncWebsocketConsumer


class NotificationConsumer(AsyncWebsocketConsumer):
    GROUP_NAME = "notifications"

    async def connect(self):
        # TODO: ยังไม่มีการตรวจสอบสิทธิ์เลยในเวอร์ชันนี้ — ทุกคนที่เชื่อมต่อเข้ามา
        # จะได้รับการแจ้งเตือนหมด ซึ่งไม่ถูกต้อง เราจะแก้ในขั้นตอนที่ 567
        await self.channel_layer.group_add(self.GROUP_NAME, self.channel_name)
        await self.accept()

    async def disconnect(self, close_code):
        await self.channel_layer.group_discard(self.GROUP_NAME, self.channel_name)

    async def notify(self, event):
        """ถูกเรียกโดย Channel Layer ทุกครั้งที่ signal ในข้อ 566.2 ทำงาน"""
        payload = {key: value for key, value in event.items() if key != "type"}
        await self.send(text_data=json.dumps(payload))
```

```python
# realtime/routing.py (อัปเดต)
from django.urls import path

from . import consumers

websocket_urlpatterns = [
    path("ws/echo/", consumers.EchoConsumer.as_asgi()),
    path("ws/echo-async/", consumers.AsyncEchoConsumer.as_asgi()),
    path("ws/chat/<str:room_name>/", consumers.ChatConsumer.as_asgi()),
    path("ws/notifications/", consumers.NotificationConsumer.as_asgi()),
]
```

### 566.4 ทดสอบด้วยมือก่อนต่อ Frontend เต็มรูปแบบ

เปิด 2 แท็บ: แท็บแรกเชื่อมต่อ WebSocket ค้างไว้ (จำลองแอดมินที่เปิดหน้า
dashboard ทิ้งไว้) แท็บที่สองสร้างคอมเมนต์ใหม่ผ่าน Django shell (จำลองผู้อ่าน
ทั่วไปโพสต์คอมเมนต์):

```javascript
// แท็บ 1 (เบราว์เซอร์): จำลองแอดมินที่เปิดหน้าทิ้งไว้
const notifSocket = new WebSocket('ws://127.0.0.1:8000/ws/notifications/');
notifSocket.onmessage = (event) => {
    console.log('🔔 แจ้งเตือนใหม่:', JSON.parse(event.data));
};
```

```bash
# terminal อีกหน้าต่างหนึ่ง: จำลองผู้อ่านโพสต์คอมเมนต์
python manage.py shell
```

```python
>>> from blog.models import Post, Comment
>>> post = Post.objects.first()
>>> Comment.objects.create(post=post, author="ผู้อ่านทดสอบ", text="เยี่ยมมากเลยบทความนี้!")
```

ทันทีที่บรรทัด `Comment.objects.create(...)` รันเสร็จ ควรเห็นข้อความ
**"🔔 แจ้งเตือนใหม่: {...}"** ปรากฏใน console ของแท็บเบราว์เซอร์ทันที **โดย
ที่แท็บนั้นไม่ได้กดอะไรเลย ไม่ได้ยิง request ใด ๆ เพิ่มเติม** — นี่คือหัวใจ
ของสิ่งที่ Part นี้ทั้งหมดพยายามสอน: server เป็นฝ่ายเริ่มพูดก่อนได้จริง

---

## ขั้นตอนที่ 567: การยืนยันตัวตนใน Consumer

### 567.1 ปัญหาความปลอดภัยของขั้นตอนที่ 566: ใครก็เชื่อมต่อรับแจ้งเตือนได้

`NotificationConsumer` ในขั้นตอนที่ 566.3 มีช่องโหว่ร้ายแรง: **ใครก็ตามที่
รู้ URL `/ws/notifications/` เชื่อมต่อเข้ามารับการแจ้งเตือนคอมเมนต์ใหม่ทั้ง
ระบบได้ทันที** โดยไม่ต้องล็อกอินด้วยซ้ำ ทั้งที่ควรเป็นข้อมูลสำหรับ staff
เท่านั้น — Part นี้จึงต้องแก้ปัญหาการยืนยันตัวตนก่อนเอาไปใช้งานจริง

### 567.2 `self.scope` คือ "Request" เวอร์ชัน WebSocket

ทุก Consumer มี attribute ชื่อ **`self.scope`** ซึ่งเป็น `dict` ที่บรรจุข้อมูล
เกี่ยวกับ connection นั้น — เทียบเท่ากับ `request` ที่ View ธรรมดารับเข้ามา
(ทบทวน Part 004) เพียงแต่อยู่ในรูปแบบ dictionary แทนที่จะเป็น object:

| คีย์ใน `self.scope` | เทียบเท่าใน `request` ของ View ปกติ |
|---|---|
| `scope["user"]` | `request.user` (ต้องมี `AuthMiddlewareStack` ก่อนถึงจะมีคีย์นี้) |
| `scope["session"]` | `request.session` |
| `scope["cookies"]` | `request.COOKIES` |
| `scope["headers"]` | `request.headers` (แต่เป็น list ของ tuple คู่ bytes ไม่ใช่ dict) |
| `scope["path"]` | `request.path` |
| `scope["query_string"]` | `request.META["QUERY_STRING"]` (เป็น bytes) |
| `scope["url_route"]["kwargs"]` | พารามิเตอร์ที่ View ฟังก์ชันรับเข้ามาโดยตรง (เช่น `slug`) |

### 567.3 `AuthMiddlewareStack` ทำงานอย่างไร

ใน `config/asgi.py` ข้อ 562.6 เราห่อ `URLRouter` ไว้ด้วย `AuthMiddlewareStack`
ไปแล้ว — Middleware ตัวนี้ทำหน้าที่คล้าย `AuthenticationMiddleware` ของ
Django ปกติ (ทบทวน Part 004 ข้อ 33.6) ทุกประการ: **มันอ่าน session cookie
ที่เบราว์เซอร์แนบมาตอน handshake โดยอัตโนมัติ** (เบราว์เซอร์แนบ cookie มา
กับทุก request รวมถึง WebSocket handshake ด้วย ตราบใดที่เป็น same-origin)
แล้วค้นหา session ที่ตรงกันในฐานข้อมูล เพื่อเติมค่าที่ถูกต้องลงใน
`scope["user"]` ให้อัตโนมัติ — ถ้าไม่มี session ที่ล็อกอินอยู่ จะได้
`AnonymousUser` กลับมา (เหมือนกับพฤติกรรมของ `request.user` ตอนไม่ได้
ล็อกอิน)

### 567.4 เขียน Consumer เวอร์ชันปลอดภัย: ตรวจสิทธิ์ก่อน `accept()`

```python
# realtime/consumers.py (เวอร์ชันสมบูรณ์ของ NotificationConsumer)
import json

from channels.generic.websocket import AsyncWebsocketConsumer


class NotificationConsumer(AsyncWebsocketConsumer):
    GROUP_NAME = "notifications"

    async def connect(self):
        user = self.scope["user"]

        if not user.is_authenticated or not user.is_staff:
            # ปฏิเสธการเชื่อมต่อ — ใช้ close code ในช่วง 4000-4999 ซึ่งสเปกของ
            # WebSocket จองไว้ให้แอปพลิเคชันกำหนดความหมายเองได้อย่างอิสระ
            # (ต่างจาก code มาตรฐานอย่าง 1000 = ปิดปกติ, 1006 = การเชื่อมต่อหลุด)
            await self.close(code=4001)
            return

        await self.channel_layer.group_add(self.GROUP_NAME, self.channel_name)
        await self.accept()

    async def disconnect(self, close_code):
        # ไม่ต้องกังวลเรื่องเรียก group_discard สำหรับ connection ที่ถูกปฏิเสธไปแล้ว
        # (ไม่เคย group_add เข้าไปตั้งแต่แรก) — group_discard บน channel ที่ไม่เคย
        # อยู่ในกลุ่มจะไม่ error แต่อย่างใด Channels จัดการให้อย่างปลอดภัย
        await self.channel_layer.group_discard(self.GROUP_NAME, self.channel_name)

    async def notify(self, event):
        payload = {key: value for key, value in event.items() if key != "type"}
        await self.send(text_data=json.dumps(payload))
```

**ข้อควรระวังสำคัญ**: ต้องเรียก `await self.close(code=...)` **ก่อน** ที่จะ
`return` ออกจาก `connect()` และ**ต้องไม่เรียก `self.accept()`** ในกรณีที่
ปฏิเสธ — ถ้าเผลอเรียก `accept()` ไปแล้วค่อย `close()` client จะเห็นว่า
เชื่อมต่อสำเร็จแวบหนึ่งก่อนถูกตัดทันที ซึ่งสร้างความสับสนโดยไม่จำเป็น

### 567.5 ข้อจำกัดสำคัญ: `AuthMiddlewareStack` ใช้ได้กับ Session Auth เท่านั้น

`AuthMiddlewareStack` ทำงานได้ดีเยี่ยมสำหรับเว็บแอปที่ใช้ **Session-based
Authentication** (ล็อกอินผ่านฟอร์ม ใช้ cookie เก็บ session ตามที่เรียนมา
ตลอดหลักสูตรฝั่ง Template) แต่**ไม่ทำงานกับ Token-based Authentication**
ของ DRF (JWT, Token Authentication จาก Phase 5) เพราะ WebSocket handshake
มาตรฐานของเบราว์เซอร์ **ไม่รองรับการแนบ custom header อย่าง
`Authorization: Bearer ...`** ได้แบบ `fetch()` ปกติ

ทางแก้ที่นิยมในงานจริงสำหรับ client ที่ต้องใช้ token (เช่น mobile app หรือ
SPA ที่ใช้ JWT) คือแนบ token มาทาง **query string** แทน แล้วเขียน Custom
Middleware ของตัวเองมาแทนที่/ต่อยอด `AuthMiddlewareStack` เพื่ออ่านค่านั้น
ออกมา validate เอง (แนวคิดคล้าย Custom Middleware ของ Django ปกติที่จะเรียน
ใน Phase 8) — รายละเอียดเชิงลึกของ pattern นี้อยู่นอกขอบเขตของ Part
เบื้องต้นนี้ และจะกลับมาเจาะลึกใน **Part 074** ที่ว่าด้วย Consumer และ
Groups ขั้นสูง สำหรับตอนนี้ขอให้เข้าใจหลักการก่อนว่า **`scope['user']` มา
จากที่ไหน และทำไมถึงมีข้อจำกัดนี้**

---

## ขั้นตอนที่ 568: การเขียน Test สำหรับ WebSocket Consumer

### 568.1 ทำไมต้องใช้ `pytest` และ `pytest-asyncio` สำหรับเทสต์ Consumer

Consumer แบบ Async ต้องถูกเรียกจาก**เทสต์ที่เป็น async function เช่นกัน**
ซึ่ง `unittest.TestCase` ของ Django (ที่ใช้มาตลอดหลักสูตรใน Phase ก่อนหน้า)
ไม่รองรับ `async def test_...` ได้อย่างเป็นธรรมชาติ — เอกสารทางการของ
Channels จึงแนะนำให้ใช้ **`pytest`** ร่วมกับ **`pytest-django`** และ
**`pytest-asyncio`** สำหรับเทสต์ WebSocket Consumer โดยเฉพาะ (เทสต์ View/
Model ปกติที่เขียนด้วย `TestCase` ในหลักสูตรนี้จนถึงตอนนี้ยังใช้ต่อได้ตาม
เดิมทุกประการ — ใช้ `pytest` เฉพาะสำหรับกลุ่มเทสต์ที่เกี่ยวกับ Channels
เท่านั้น)

```bash
pip install pytest pytest-django pytest-asyncio
pip freeze > requirements.txt
```

```ini
# pytest.ini (สร้างที่ root ของโปรเจกต์ ระดับเดียวกับ manage.py)
[pytest]
DJANGO_SETTINGS_MODULE = config.settings
asyncio_mode = auto
python_files = test_*.py
```

`asyncio_mode = auto` บอก `pytest-asyncio` ให้รันทุกฟังก์ชันเทสต์ที่เขียน
เป็น `async def` โดยอัตโนมัติ โดยไม่ต้องแปะ decorator `@pytest.mark.asyncio`
ซ้ำทุกฟังก์ชัน (แต่หลักสูตรนี้จะยังคงแปะ decorator ไว้ให้เห็นชัดเจนในตัวอย่าง
เพื่อความเข้าใจง่าย)

### 568.2 ใช้ Channel Layer แบบ In-Memory ตอนเทสต์ (ไม่ต้องพึ่ง Redis จริง)

การรันเทสต์ไม่ควรพึ่งพา Redis จริงที่ตั้งค่าไว้ในขั้นตอนที่ 565 (จะทำให้
เทสต์ช้าและเปราะบาง ถ้า Redis ไม่ได้รันอยู่เทสต์จะพังทั้งหมด) Channels มี
Backend สำรองชื่อ **`InMemoryChannelLayer`** ที่ทำงานในหน่วยความจำล้วน ๆ
เหมาะสำหรับการเทสต์โดยเฉพาะ:

```python
# conftest.py (สร้างที่ root ของโปรเจกต์)
import pytest


@pytest.fixture(autouse=True)
def use_in_memory_channel_layer(settings):
    """เทสต์ทุกตัวในโปรเจกต์จะใช้ Channel Layer แบบในหน่วยความจำแทน Redis จริงเสมอ"""
    settings.CHANNEL_LAYERS = {
        "default": {"BACKEND": "channels.layers.InMemoryChannelLayer"},
    }
```

`autouse=True` ทำให้ fixture นี้ทำงานกับ**ทุกเทสต์**โดยอัตโนมัติโดยไม่ต้อง
ระบุชื่อ fixture ในทุกฟังก์ชันเทสต์ — เทคนิคเดียวกับที่ pytest แนะนำสำหรับ
การตั้งค่าที่ต้องการให้ "ใช้เสมอ" ทั่วทั้ง test suite

### 568.3 เทสต์ Consumer พื้นฐาน: `WebsocketCommunicator`

```python
# realtime/tests/test_echo_consumer.py
import pytest
from channels.testing import WebsocketCommunicator

from config.asgi import application


@pytest.mark.asyncio
async def test_echo_consumer_replies_with_same_message():
    communicator = WebsocketCommunicator(application, "/ws/echo-async/")
    connected, subprotocol = await communicator.connect()
    assert connected is True

    await communicator.send_to(text_data="สวัสดี Channels")
    response = await communicator.receive_from()
    assert response == "สวัสดี Channels"

    await communicator.disconnect()
```

`WebsocketCommunicator` คือเทียบเท่า `Client` ของ Django test framework
(`django.test.Client`) สำหรับ WebSocket โดยเฉพาะ — สร้างขึ้นด้วย ASGI
application ทั้งชุด (`config.asgi.application` ที่เราตั้งค่าไว้ในขั้นตอนที่
562) กับ path ที่ต้องการทดสอบ แล้วมี method ให้ `connect()`, `send_to()`,
`receive_from()`, `disconnect()` จำลองพฤติกรรมของเบราว์เซอร์ได้ครบวงจร
โดยไม่ต้องเปิดเบราว์เซอร์จริงเลย

### 568.4 เทสต์การกระจายข้อความข้าม Connection ด้วย Group

```python
# realtime/tests/test_chat_consumer.py
import json

import pytest
from channels.testing import WebsocketCommunicator

from config.asgi import application


@pytest.mark.asyncio
async def test_message_broadcasts_to_everyone_in_same_room():
    communicator_a = WebsocketCommunicator(application, "/ws/chat/general/")
    communicator_b = WebsocketCommunicator(application, "/ws/chat/general/")

    connected_a, _ = await communicator_a.connect()
    connected_b, _ = await communicator_b.connect()
    assert connected_a and connected_b

    await communicator_a.send_to(text_data=json.dumps({
        "message": "สวัสดีทุกคน",
        "sender": "คนที่ 1",
    }))

    # ทั้งคู่ต้องได้รับข้อความ เพราะ group_send() กระจายให้ทุก channel ในกลุ่ม
    # รวมถึงผู้ส่งเองด้วย (ทบทวนขั้นตอนที่ 565.5)
    response_a = json.loads(await communicator_a.receive_from())
    response_b = json.loads(await communicator_b.receive_from())

    assert response_a["message"] == "สวัสดีทุกคน"
    assert response_b["message"] == "สวัสดีทุกคน"

    await communicator_a.disconnect()
    await communicator_b.disconnect()


@pytest.mark.asyncio
async def test_different_rooms_do_not_leak_messages_to_each_other():
    communicator_general = WebsocketCommunicator(application, "/ws/chat/general/")
    communicator_random = WebsocketCommunicator(application, "/ws/chat/random/")

    await communicator_general.connect()
    await communicator_random.connect()

    await communicator_general.send_to(text_data=json.dumps({"message": "ข้อความห้อง general"}))

    # ห้อง general ต้องได้รับข้อความของตัวเอง
    response = json.loads(await communicator_general.receive_from())
    assert response["message"] == "ข้อความห้อง general"

    # ห้อง random ต้อง "ไม่ได้รับอะไรเลย" ภายในเวลาที่กำหนด (timeout)
    with pytest.raises(TimeoutError):
        await communicator_random.receive_from(timeout=0.5)

    await communicator_general.disconnect()
    await communicator_random.disconnect()
```

เทสต์ที่สองแสดงเทคนิคสำคัญ: การยืนยันว่า "ไม่ควรได้รับข้อความ" ทำผ่านการ
`await` พร้อม `timeout` สั้น ๆ แล้วคาดหวังว่าจะเจอ `TimeoutError` — ถ้าห้อง
`random` ดันได้รับข้อความของห้อง `general` มาด้วย (บั๊กเรื่อง group name
ไม่แยกกันอย่างถูกต้อง) เทสต์นี้จะ fail ทันทีเพราะ `receive_from()` จะคืน
ค่าได้สำเร็จโดยไม่ timeout

### 568.5 เทสต์ Consumer ที่มีการยืนยันตัวตน (จำลอง `scope["user"]` โดยตรง)

ในเทสต์ ไม่จำเป็นต้องพึ่งพา cookie/session จริงเพื่อจำลองผู้ใช้ที่ล็อกอินอยู่
— สามารถกำหนดค่า `scope["user"]` ให้ `WebsocketCommunicator` ได้โดยตรงก่อน
เรียก `connect()`:

```python
# realtime/tests/test_notification_consumer.py
import json

import pytest
from channels.db import database_sync_to_async
from channels.testing import WebsocketCommunicator
from django.contrib.auth import get_user_model
from django.contrib.auth.models import AnonymousUser

from blog.models import Comment, Post
from config.asgi import application

User = get_user_model()


@pytest.mark.asyncio
@pytest.mark.django_db(transaction=True)
async def test_anonymous_user_is_rejected():
    communicator = WebsocketCommunicator(application, "/ws/notifications/")
    communicator.scope["user"] = AnonymousUser()

    connected, subprotocol = await communicator.connect()
    assert connected is False  # ต้องถูกปฏิเสธตามที่ตั้งไว้ในขั้นตอนที่ 567.4


@pytest.mark.asyncio
@pytest.mark.django_db(transaction=True)
async def test_non_staff_user_is_rejected():
    regular_user = await database_sync_to_async(User.objects.create_user)(
        username="reader", password="testpass123", is_staff=False,
    )
    communicator = WebsocketCommunicator(application, "/ws/notifications/")
    communicator.scope["user"] = regular_user

    connected, _ = await communicator.connect()
    assert connected is False


@pytest.mark.asyncio
@pytest.mark.django_db(transaction=True)
async def test_new_comment_broadcasts_to_connected_staff():
    staff_user = await database_sync_to_async(User.objects.create_user)(
        username="editor", password="testpass123", is_staff=True,
    )
    post = await database_sync_to_async(Post.objects.create)(
        title="ทดสอบ Django Channels", slug="test-django-channels",
        content="เนื้อหาสำหรับทดสอบ", is_published=True,
    )

    communicator = WebsocketCommunicator(application, "/ws/notifications/")
    communicator.scope["user"] = staff_user
    connected, _ = await communicator.connect()
    assert connected is True

    # การสร้าง Comment จะไปทริกเกอร์ signal ในขั้นตอนที่ 566.2 โดยอัตโนมัติ
    await database_sync_to_async(Comment.objects.create)(
        post=post, author="ผู้อ่านทดสอบ", text="เยี่ยมมากเลยบทความนี้!",
    )

    response = json.loads(await communicator.receive_from(timeout=2))
    assert response["post_title"] == "ทดสอบ Django Channels"
    assert response["author"] == "ผู้อ่านทดสอบ"

    await communicator.disconnect()
```

`@pytest.mark.django_db(transaction=True)` จำเป็นสำหรับเทสต์ที่มีการเรียก
โค้ดข้าม thread ผ่าน `database_sync_to_async` (การสร้าง `User`, `Post`,
`Comment` แต่ละครั้งรันอยู่คนละ thread จาก event loop หลัก) — ถ้าใช้แค่
`@pytest.mark.django_db` ธรรมดา อาจเจอปัญหาข้อมูลมองไม่เห็นข้ามเธรดเพราะ
Django ครอบทุกเทสต์ไว้ใน transaction เดียวที่ยังไม่ commit จริง

---

## ขั้นตอนที่ 569: ข้อควรพิจารณาการ Deploy Channels

### 569.1 `runserver` + Daphne เหมาะสำหรับ Development เท่านั้น

เช่นเดียวกับที่ Part 004 ข้อ 34.5 เตือนไว้ว่า `runserver` แบบ WSGI ไม่ได้
ออกแบบมาสำหรับ production — Daphne ที่เข้ามาแทนที่ `runserver` ใน
ขั้นตอนที่ 562.8 ก็ยังคง **เป็นแค่ dev server** เหมือนเดิม (มันแค่เพิ่ม
ความสามารถ WebSocket เข้าไปในโหมดพัฒนาเท่านั้น) production ต้องรัน ASGI
server แยกต่างหากเสมอ

### 569.2 ทำไม Gunicorn ธรรมดา (แบบ Sync Worker) ใช้กับ Channels ไม่ได้

Gunicorn ที่หลักสูตรนี้จะแนะนำให้ใช้ deploy จริงใน **Part 089** โดยค่า
เริ่มต้นใช้ **sync worker** ที่พูดภาษา **WSGI เท่านั้น** — มันไม่รู้จักแนวคิด
"อัปเกรดเป็น WebSocket" ตามที่อธิบายในขั้นตอนที่ 561.4 เลยแม้แต่น้อย ถ้า
พยายามเชื่อมต่อ WebSocket เข้าไปยัง Gunicorn sync worker ธรรมดา จะได้ error
ทันทีเพราะมันไม่มีกลไกจัดการ HTTP `Upgrade` header

| ประเภทเซิร์ฟเวอร์ | พูดภาษาอะไร | รองรับ WebSocket ไหม | ใช้ตอนไหน |
|---|---|---|---|
| Gunicorn (sync worker, ค่าเริ่มต้น) | WSGI เท่านั้น | ❌ ไม่รองรับเลย | โปรเจกต์ที่ไม่มี WebSocket/Channels (ส่วนใหญ่ของหลักสูตรจนถึง Part 056) |
| Gunicorn + `uvicorn.workers.UvicornWorker` | ASGI (ผ่าน Uvicorn ในฐานะ worker class) | ✅ รองรับ | ต้องการความเสถียรของ process management ของ Gunicorn ผสานกับความเร็วของ Uvicorn |
| **Daphne** | ASGI (ตัวอ้างอิงของทีม Django) | ✅ รองรับ | ทีมที่อยากใช้เครื่องมือทางการของ Django โดยตรง ไม่ต้องผสมสองเครื่องมือ |
| **Uvicorn** (รันตรง ๆ ไม่ผ่าน Gunicorn) | ASGI | ✅ รองรับ | ต้องการประสิทธิภาพสูงสุด นิยมคู่กับ `--workers` สำหรับ multi-process |

### 569.3 ตัวอย่างคำสั่งรัน Production ASGI Server

```bash
# ตัวเลือกที่ 1: Daphne ตรง ๆ
daphne -b 0.0.0.0 -p 8001 config.asgi:application

# ตัวเลือกที่ 2: Uvicorn ตรง ๆ พร้อมหลาย worker process
uvicorn config.asgi:application --host 0.0.0.0 --port 8001 --workers 4

# ตัวเลือกที่ 3: Gunicorn คุม process + Uvicorn worker class (นิยมมากในทีมที่คุ้นเคย
# กับ Gunicorn อยู่แล้วจาก Part 089 และอยากได้ process management ที่แข็งแรงกว่า)
pip install uvicorn-worker
gunicorn config.asgi:application -k uvicorn_worker.UvicornWorker --workers 4 --bind 0.0.0.0:8001
```

### 569.4 Channel Layer (Redis) ต้องเป็น Service กลางที่ทุก Process มองเห็นร่วมกัน

เมื่อ deploy จริงด้วยหลาย worker process (`--workers 4` ในตัวอย่างข้างต้น)
**การเชื่อมต่อ WebSocket ของผู้ใช้แต่ละคนอาจตกไปอยู่คนละ process กัน** —
ถ้าแอดมิน A เชื่อมต่ออยู่ที่ process #1 แต่คอมเมนต์ใหม่ถูกสร้างโดย request
ที่ไปตกที่ process #3 การแจ้งเตือนจะยังคงไปถึงแอดมิน A ได้อย่างถูกต้อง
**ก็ต่อเมื่อทุก process ชี้ไปที่ Redis instance เดียวกัน** ตามที่ตั้งค่าไว้ใน
`CHANNEL_LAYERS` (ขั้นตอนที่ 565.3) — Redis คือ "จุดนัดพบกลาง" ที่ทำให้หลาย
process คุยกันได้ ถ้าลืมข้อนี้แล้วแต่ละ process ใช้ Redis คนละตัว (หรือ
เผลอใช้ `InMemoryChannelLayer` ใน production) การแจ้งเตือนจะไปถึงเฉพาะ
connection ที่บังเอิญอยู่ process เดียวกับที่สร้างคอมเมนต์เท่านั้น ซึ่งเป็น
บั๊กที่พบบ่อยมากตอน deploy ครั้งแรก

### 569.5 Reverse Proxy (Nginx) ต้อง Forward Header สำหรับ WebSocket โดยเฉพาะ

Nginx ที่จะเรียนตั้งค่าอย่างเป็นระบบใน **Part 089** ต้องมี config เพิ่มเติม
เฉพาะสำหรับเส้นทาง WebSocket เพื่อส่งต่อ header `Upgrade`/`Connection` ที่
กล่าวถึงในขั้นตอนที่ 561.4 ให้ถูกต้อง:

```nginx
# ตัวอย่างส่วนที่เกี่ยวกับ WebSocket ใน nginx.conf (เจาะลึกเต็มรูปแบบใน Part 089)
upstream django_asgi {
    server 127.0.0.1:8001;
}

server {
    listen 80;
    server_name example.com;

    location /ws/ {
        proxy_pass http://django_asgi;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        # WebSocket เปิดค้างไว้นาน ต้องเพิ่ม timeout ให้มากกว่าค่าเริ่มต้น
        # (default ของ nginx คือ 60 วินาที ซึ่งสั้นเกินไปสำหรับ WebSocket)
        proxy_read_timeout 86400;
    }

    location / {
        proxy_pass http://django_asgi;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```

บรรทัดที่พลาดบ่อยที่สุดคือ `proxy_set_header Connection "upgrade";` — ถ้า
ลืมบรรทัดนี้ WebSocket handshake จะล้มเหลวทันทีตอน deploy จริง แม้ว่าจะ
ทำงานได้ปกติดีตอนทดสอบบนเครื่อง dev (เพราะตอน dev ไม่มี reverse proxy คั่น
กลาง เบราว์เซอร์คุยกับ Daphne โดยตรง)

### 569.6 ตารางสรุปสิ่งที่เปลี่ยนไปเมื่อ Deploy โปรเจกต์ที่มี Channels

| องค์ประกอบ | โปรเจกต์ Django ปกติ (Part 089) | โปรเจกต์ที่มี Channels (Part นี้) |
|---|---|---|
| Application Server | Gunicorn (sync worker) | Daphne / Uvicorn / Gunicorn+Uvicorn worker |
| Service เสริมที่ต้องรันคู่กัน | PostgreSQL | PostgreSQL **+ Redis** (สำหรับ Channel Layer) |
| Nginx config | Forward header ปกติ | ต้องเพิ่ม `Upgrade`/`Connection` header + timeout ยาวขึ้น |
| การเพิ่ม worker process เพื่อ scale | ไม่กระทบ logic ของแอป | ต้องมั่นใจว่าทุก process ใช้ Redis ตัวเดียวกัน (ข้อ 569.4) |

> **ขอบเขตของ Part นี้**: รายละเอียดเชิงลึกเรื่อง Docker Compose สำหรับรัน
> Django + Redis คู่กัน (Part 087), การตั้งค่า Nginx แบบเต็มรูปแบบ (Part
> 089), และการ scale Channels ข้ามหลายเครื่องพร้อมจัดการ Consumer/Group
> ขั้นสูง (Part 074) จะถูกเจาะลึกในแต่ละ Part นั้น ๆ โดยเฉพาะ — Part นี้ให้
> แค่ภาพรวมที่เพียงพอสำหรับเข้าใจว่า "อะไรเปลี่ยนไปบ้าง" เมื่อเทียบกับ
> deploy โปรเจกต์ที่ไม่มี WebSocket

---

## ขั้นตอนที่ 570: สรุปและแบบฝึกหัด

### 570.1 Capstone: ระบบแจ้งเตือนคอมเมนต์ใหม่แบบ Real-time เต็มรูปแบบ

นำทุกเทคนิคจากขั้นตอนที่ 561-569 มาประกอบกันเป็นระบบแจ้งเตือนที่ใช้งานได้
จริงบน `base.html` — กระดิ่งแจ้งเตือนที่แสดงจำนวนที่ยังไม่อ่าน พร้อม toast
เด้งขึ้นทันทีที่มีคอมเมนต์ใหม่ และเชื่อมต่อใหม่อัตโนมัติถ้าหลุด

**Backend ทั้งหมด (สรุปรวมจากขั้นตอนที่ 562-567 — ไม่มีโค้ดใหม่เพิ่ม):**

```
config/
├── settings.py       ← INSTALLED_APPS (daphne, channels), ASGI_APPLICATION, CHANNEL_LAYERS
└── asgi.py           ← ProtocolTypeRouter + AuthMiddlewareStack + URLRouter

blog/
├── apps.py           ← ready() import blog.signals
└── signals.py         ← notify_new_comment() ผูกกับ post_save ของ Comment

realtime/
├── consumers.py       ← NotificationConsumer (มีการเช็ค is_staff)
├── routing.py          ← websocket_urlpatterns
└── tests/              ← test_notification_consumer.py จากขั้นตอนที่ 568
```

**Frontend ใหม่ทั้งหมดที่ต้องเพิ่ม (ส่วนที่ยังไม่เคยเขียนใน Part นี้):**

```html
<!-- templates/base.html (เพิ่มก่อนปิด </body> — เฉพาะส่วนของ staff เท่านั้น) -->
{% if request.user.is_staff %}
<div id="notification-bell" class="notification-bell" title="การแจ้งเตือน">
    🔔 <span id="notification-count">0</span>
</div>
<div id="notification-toast-container"></div>

<script>
(function () {
    let socket = null;
    let unreadCount = 0;
    let reconnectDelay = 1000;  // เริ่มที่ 1 วินาที แล้วเพิ่มเป็นเท่าตัวทุกครั้งที่หลุด

    function connect() {
        const protocol = window.location.protocol === 'https:' ? 'wss' : 'ws';
        socket = new WebSocket(`${protocol}://${window.location.host}/ws/notifications/`);

        socket.onopen = () => {
            reconnectDelay = 1000;  // เชื่อมต่อสำเร็จแล้ว รีเซ็ตเวลาหน่วงสำหรับครั้งถัดไปที่หลุด
        };

        socket.onmessage = (event) => {
            const data = JSON.parse(event.data);
            unreadCount += 1;
            document.getElementById('notification-count').textContent = unreadCount;
            showToast(`ความคิดเห็นใหม่จาก ${data.author} บน "${data.post_title}"`);
        };

        socket.onclose = () => {
            // การเชื่อมต่อหลุด (server รีสตาร์ท, เครือข่ายมีปัญหา ฯลฯ)
            // ลองเชื่อมใหม่แบบ exponential backoff เพื่อไม่ให้ยิง request รัวเกินไป
            // ถ้า server ยังไม่พร้อม (สูงสุดไม่เกิน 30 วินาทีต่อครั้ง)
            setTimeout(connect, reconnectDelay);
            reconnectDelay = Math.min(reconnectDelay * 2, 30000);
        };
    }

    function showToast(message) {
        const toast = document.createElement('div');
        toast.className = 'notification-toast';
        toast.textContent = message;
        document.getElementById('notification-toast-container').appendChild(toast);
        setTimeout(() => toast.remove(), 6000);
    }

    document.getElementById('notification-bell').addEventListener('click', () => {
        unreadCount = 0;
        document.getElementById('notification-count').textContent = unreadCount;
    });

    connect();
})();
</script>
{% endif %}
```

```css
/* static/css/notifications.css */
.notification-bell {
    position: fixed;
    top: 16px;
    right: 16px;
    cursor: pointer;
    font-size: 1.2rem;
    z-index: 1000;
}

.notification-toast {
    background: #1f2937;
    color: #fff;
    padding: 10px 16px;
    border-radius: 6px;
    margin-top: 8px;
    max-width: 320px;
    box-shadow: 0 2px 8px rgba(0, 0, 0, 0.2);
}

#notification-toast-container {
    position: fixed;
    top: 56px;
    right: 16px;
    z-index: 1000;
}
```

ผลลัพธ์สุดท้าย: ผู้อ่านโพสต์คอมเมนต์ผ่านหน้าเว็บปกติ (ไม่ว่าจะเป็น View
ธรรมดาจาก Part 026 หรือฟอร์ม HTMX จาก Part 053) แอดมินที่เปิดหน้าใด ๆ ของ
เว็บไซต์ทิ้งไว้ (ตราบใดที่ `base.html` ถูก extend) จะเห็นตัวเลขที่กระดิ่ง
เพิ่มขึ้นและ toast เด้งขึ้นมาทันที **โดยไม่ต้องกดอะไรเลยแม้แต่ครั้งเดียว**
และถ้าอินเทอร์เน็ตหลุดชั่วคราว สคริปต์จะพยายามเชื่อมต่อใหม่ให้อัตโนมัติ

### 570.2 สรุปสิ่งที่ได้เรียนรู้ใน Part นี้

- ✅ เข้าใจความแตกต่างพื้นฐานระหว่าง HTTP Request/Response กับ WebSocket:
  ใครเป็นฝ่ายเริ่มสื่อสาร, Handshake ทำงานอย่างไร, และทำไม Polling ถึงไม่ใช่
  คำตอบสุดท้ายสำหรับงานที่ต้องการ latency ต่ำจริง ๆ
- ✅ ติดตั้ง Django Channels บนพื้นฐาน ASGI ที่ Part 004 ปูไว้ตั้งแต่ต้น
  ตั้งค่า `INSTALLED_APPS` (`daphne` ต้องอยู่บนสุด), `ASGI_APPLICATION`,
  และอัปเกรด `config/asgi.py` เป็น `ProtocolTypeRouter`
- ✅ เขียน Consumer แบบ Synchronous (`WebsocketConsumer`) และเข้าใจวงจร
  ชีวิต `connect()` → `receive()` → `disconnect()`
- ✅ เขียน Consumer แบบ Asynchronous (`AsyncWebsocketConsumer`) และเข้าใจ
  ว่าทำไมมันเหมาะกับ WebSocket มากกว่า พร้อมรู้จัก `database_sync_to_async`
  สำหรับเรียก Django ORM จาก async context อย่างปลอดภัย
- ✅ ใช้ Channel Layer บน Redis เพื่อกระจายข้อความข้าม Consumer ด้วย
  `group_add`, `group_send`, `group_discard`
- ✅ ผูก Django Signal (`post_save`) จาก Part 019 เข้ากับ Channel Layer
  ด้วย `async_to_sync` เพื่อสร้างระบบแจ้งเตือนคอมเมนต์ใหม่แบบ real-time
  ที่ทำงานจริง
- ✅ ยืนยันตัวตนผู้ใช้ใน Consumer ผ่าน `self.scope['user']` และ
  `AuthMiddlewareStack` พร้อมรู้ข้อจำกัดเรื่อง Token-based Authentication
- ✅ เขียนเทสต์สำหรับ WebSocket Consumer ด้วย `pytest` +
  `channels.testing.WebsocketCommunicator` รวมถึงการจำลอง Group Broadcast
  และการจำลอง `scope['user']` โดยไม่ต้องพึ่ง session จริง
- ✅ เข้าใจข้อควรพิจารณาตอน deploy: ทำไม Gunicorn sync worker ธรรมดาใช้กับ
  Channels ไม่ได้ ต้องใช้ Daphne/Uvicorn แทน และทำไม Redis ต้องเป็น service
  กลางที่ทุก process มองเห็นร่วมกัน

### 570.3 Checklist ก่อนไป Part ถัดไป

- [ ] ติดตั้ง `channels`, `daphne`, `channels_redis` สำเร็จ และเห็นข้อความ
      "Starting ASGI/Daphne" ตอนรัน `runserver`
- [ ] ตั้งค่า `config/asgi.py` เป็น `ProtocolTypeRouter` ที่แยก `http` และ
      `websocket` ได้ถูกต้อง โดยที่หน้าเว็บ HTTP เดิมทั้งหมดยังทำงานปกติ
- [ ] เขียน `EchoConsumer` แบบ sync และทดสอบผ่าน browser console สำเร็จ
- [ ] เขียน `AsyncEchoConsumer` แบบ async และอธิบายได้ว่าทำไมต้องใช้
      `database_sync_to_async` เมื่อต้องเรียก Django ORM
- [ ] ติดตั้ง Redis และตั้งค่า `CHANNEL_LAYERS` สำเร็จ ทดสอบห้องแชทด้วย
      2 แท็บเบราว์เซอร์แล้วเห็นข้อความกระจายถึงกันจริง
- [ ] เขียน Signal ที่ผูก `post_save` ของ `Comment` เข้ากับ
      `channel_layer.group_send()` สำเร็จ และทดสอบผ่าน Django shell
- [ ] ใส่การตรวจสอบ `self.scope['user'].is_staff` ใน `NotificationConsumer`
      และทดสอบว่า user ที่ไม่ใช่ staff ถูกปฏิเสธการเชื่อมต่อจริง
- [ ] ติดตั้ง `pytest`/`pytest-django`/`pytest-asyncio` และเขียนเทสต์ด้วย
      `WebsocketCommunicator` ผ่านอย่างน้อย 3 เคสตามขั้นตอนที่ 568
- [ ] ต่อกระดิ่งแจ้งเตือนเข้ากับ `base.html` ตามข้อ 570.1 และทดสอบผ่าน
      เบราว์เซอร์จริงว่าเห็นการแจ้งเตือนทันทีที่มีคอมเมนต์ใหม่

### 570.4 แบบฝึกหัดท้ายบท

**แบบฝึกหัดที่ 1**: ปรับปรุง `AsyncEchoConsumer` จากขั้นตอนที่ 564 ให้แทนที่
จะ echo ข้อความกลับตรง ๆ ให้แปลงข้อความที่ได้รับเป็นตัวพิมพ์ใหญ่ทั้งหมด (ถ้า
เป็นภาษาอังกฤษ) และนับจำนวนตัวอักษรแนบไปด้วยในรูปแบบ JSON เช่น ส่ง `"hello"`
ไป ต้องได้รับ `{"transformed": "HELLO", "length": 5}` กลับมา

**แบบฝึกหัดที่ 2**: ขยาย `ChatConsumer` จากขั้นตอนที่ 565 ให้มีฟีเจอร์
**"แสดงจำนวนผู้ใช้ที่ออนไลน์อยู่ในห้อง"** โดยใช้ Redis (`channels_redis`
รองรับการนับ channel ในกลุ่มผ่าน library เสริม หรือจะเก็บตัวนับเองด้วย
Django cache framework ที่เรียนใน Part 001 ข้อ 116 ก็ได้) ทุกครั้งที่มีคน
เข้า/ออกห้อง ให้ `group_send` ข้อความประเภท `user_count_update` ไปบอกทุกคน
ในห้องว่าตอนนี้มีกี่คนออนไลน์อยู่

**แบบฝึกหัดที่ 3**: เพิ่มปุ่ม **"ทำเครื่องหมายว่าอ่านแล้ว" (Mark as Read)**
ให้ระบบแจ้งเตือนในข้อ 570.1 โดยสร้าง Model `Notification` ใหม่ที่เก็บ
ประวัติการแจ้งเตือนแต่ละรายการลงฐานข้อมูลจริง (ไม่ใช่แค่ push ผ่าน
WebSocket แล้วหายไปถ้าไม่ได้เปิดหน้าทัน) พร้อม field `is_read` แล้วเขียน
View ธรรมดา (ไม่ต้องใช้ WebSocket) สำหรับดึงประวัติแจ้งเตือนทั้งหมดมาแสดงใน
หน้า `/notifications/` และปุ่มเปลี่ยนสถานะเป็นอ่านแล้ว

**แบบฝึกหัดที่ 4 (ขั้นสูง — Capstone เต็มรูปแบบ)**: ประกอบระบบแจ้งเตือน
คอมเมนต์ใหม่แบบ real-time ให้สมบูรณ์ที่สุดโดยรวมทุกอย่างเข้าด้วยกัน: (1)
Signal ที่ผูกกับ `post_save` ของ `Comment` ตามขั้นตอนที่ 566, (2)
`NotificationConsumer` ที่ตรวจสอบสิทธิ์ staff ตามขั้นตอนที่ 567, (3) เก็บ
ประวัติแจ้งเตือนลงฐานข้อมูลจริงตามแบบฝึกหัดที่ 3, (4) หน้า frontend ตามข้อ
570.1 ที่แสดงกระดิ่งพร้อมจำนวนที่ยังไม่อ่านซึ่ง**คำนวณจากฐานข้อมูลจริงตอน
โหลดหน้าแรก** (ไม่ใช่เริ่มที่ 0 เสมอเหมือนตัวอย่างในข้อ 570.1) แล้วค่อยบวก
เพิ่มแบบ real-time เมื่อมีคอมเมนต์ใหม่เข้ามาระหว่างที่เปิดหน้าค้างไว้ และ (5)
เขียนเทสต์ครอบคลุมทั้งระบบตามรูปแบบขั้นตอนที่ 568 อย่างน้อย 6 เคส (รวมเคส
ที่ผู้ใช้ทั่วไปพยายามเชื่อมต่อแล้วถูกปฏิเสธ)

### 570.5 คำถามที่พบบ่อย (FAQ)

**Q: ควรใช้ WebSocket แทน HTMX Polling (Part 053) กับทุกฟีเจอร์ที่ต้องการ
ความสด (real-time-ish) เลยหรือไม่?**
A: ไม่ควร — ให้เลือกตามความถี่ของการเปลี่ยนแปลงข้อมูลและความคาดหวังของ
ผู้ใช้ ตามตารางในข้อ 561.6 งานที่ผู้ใช้เป็นคนกดเอง (ปุ่มถูกใจ, ค้นหา,
infinite scroll) HTMX ยังคงเป็นตัวเลือกที่ง่ายกว่าและเพียงพอเสมอ WebSocket
คุ้มค่ากับความซับซ้อนที่เพิ่มขึ้นก็ต่อเมื่อ**server ต้องเป็นฝ่ายเริ่มพูดก่อน
จริง ๆ** โดยที่ผู้ใช้ไม่ได้ทำอะไรเลย เช่น แจ้งเตือน, ห้องแชท, แดชบอร์ดที่
ต้องอัปเดตแบบวินาทีต่อวินาที

**Q: ทำไมตอนทดสอบใน Part นี้ต้องเปิด browser console เขียน JavaScript เอง
ทั้งที่ Part 052-053 มี HTMX/Fetch ช่วยให้เขียนน้อยลง?**
A: เพราะ **HTMX ไม่รองรับ WebSocket โดยตรง** (HTMX ทำงานบนโมเดล
request-response ของ HTTP เท่านั้น แม้จะมี extension เสริมชื่อ
`htmx-ws` ที่เพิ่มการรองรับ WebSocket แบบพื้นฐานให้ HTMX ได้ก็ตาม)
หลักสูตรนี้เลือกสอน WebSocket API ดิบของเบราว์เซอร์ (`new WebSocket(...)`)
ก่อน เพื่อให้เข้าใจกลไกเบื้องหลังอย่างแท้จริง ก่อนจะเลือกใช้ library ช่วยใน
โปรเจกต์จริงถ้าต้องการ

**Q: ถ้าไม่มี Redis ใช้ `InMemoryChannelLayer` ใน production ได้ไหม
เพื่อประหยัดค่าใช้จ่าย?**
A: ใช้ได้เฉพาะกรณีที่ deploy ด้วย **1 process เท่านั้น** เพราะ
`InMemoryChannelLayer` เก็บข้อมูลกลุ่มไว้ใน memory ของ process นั้น ๆ
เพียงลำพัง ไม่ได้แชร์ข้ามกันตามที่อธิบายในขั้นตอนที่ 569.4 — ถ้า deploy
ด้วยมากกว่า 1 worker process (ซึ่งเป็นมาตรฐานของงานจริงเพื่อรองรับโหลด)
การแจ้งเตือนจะไปถึงเฉพาะ connection ที่บังเอิญอยู่ process เดียวกับที่
สร้างเหตุการณ์เท่านั้น ทำให้ระบบทำงาน "บางครั้งได้ บางครั้งไม่ได้" ซึ่ง
เป็นบั๊กที่ตามหาสาเหตุยากมากถ้าไม่รู้ที่มา ควรใช้ Redis เสมอสำหรับ
production ที่มีมากกว่า 1 process

**Q: signal receiver ในขั้นตอนที่ 566.2 ทำให้ View ที่สร้างคอมเมนต์ทำงาน
ช้าลงหรือไม่ เพราะต้องรอส่งข้อความผ่าน Redis ก่อน?**
A: มีผลกระทบเล็กน้อยจริง เพราะ `async_to_sync(channel_layer.group_send)`
เป็นการเรียกแบบ **synchronous** (รอผลลัพธ์) ภายใน request-response cycle
ปกติ ทำให้ View ต้องรอจนกว่าข้อความจะถูกส่งเข้า Redis สำเร็จก่อนถึงจะคืน
response ได้ ในงานที่ต้องการ throughput สูงมาก ๆ ทีมมืออาชีพมักย้ายงาน
ประเภทนี้ไปทำแบบ **asynchronous ผ่าน Task Queue** (เช่น Celery ที่จะเรียน
ใน Phase 9) แทนการเรียกตรงจาก signal เพื่อไม่ให้ผู้ใช้ที่โพสต์คอมเมนต์
ต้องรอกระบวนการแจ้งเตือนที่ไม่เกี่ยวกับเขาโดยตรง — แต่สำหรับสเกลของ
หลักสูตรนี้และเว็บส่วนใหญ่ ความหน่วงที่เพิ่มขึ้นนี้เล็กน้อยมากจนไม่มีนัยสำคัญ

---

## เตรียมตัวสำหรับ Part ถัดไป

**Part 058: File Upload, Image Processing และ Media Handling** จะปิดท้าย
Phase 6 ด้วยเรื่องที่ต่างไปจาก Part นี้โดยสิ้นเชิง — การจัดการไฟล์ที่ผู้ใช้
อัปโหลดเข้ามา (`ImageField`/`FileField` ที่เกริ่นไว้สั้น ๆ ตั้งแต่ Part 026
ข้อ 143 กับ `PostImage`) อย่างเจาะลึก: การตั้งค่า `MEDIA_ROOT`/`MEDIA_URL`
อย่างถูกต้อง การ validate ชนิดและขนาดไฟล์ก่อนบันทึก การประมวลผลรูปภาพด้วย
`Pillow` (resize, crop, สร้าง thumbnail อัตโนมัติ), การอัปโหลดไฟล์ผ่าน
`multipart/form-data` พร้อม progress bar (ที่ข้อ 526.2 ของ Part 053 เคย
เตือนไว้ว่าต้องปิด `hx-boost` สำหรับฟอร์มประเภทนี้) และการจัดเก็บไฟล์บน
Cloud Storage (S3) แทน local disk เมื่อ deploy จริง เตรียม virtual
environment ให้พร้อมสำหรับติดตั้ง `Pillow` เพิ่ม และลองนึกดูว่าตอนนี้คุณมี
ระบบแจ้งเตือนคอมเมนต์แบบ real-time แล้ว — Part ถัดไปจะทำให้คอมเมนต์แนบ
รูปภาพได้ด้วย น่าสนใจว่าจะผสานทั้งสองเรื่องเข้าด้วยกันได้อย่างไร!
