# Part 039: บทนำสู่ Django REST Framework

> **ขั้นตอนที่ 381-390 ของหลักสูตร** | Phase 5: Django REST Framework & API (ตอนที่ 1)
>
> เป้าหมายของ Part นี้: เข้าใจว่า REST คืออะไรในเชิงหลักการ ทำไมเว็บแอปสมัยใหม่ต้องมี API
> แยกออกจากหน้าเว็บ HTML ที่คุณสร้างมาตลอด Phase 1-4 จากนั้นติดตั้ง Django REST Framework
> (DRF) ลงในโปรเจกต์ `secureblog` ของคุณ ทำความรู้จักกับ `Request`/`Response` object ที่ DRF
> ใช้แทน `HttpRequest`/`HttpResponse` ของ Django เอง เข้าใจกลไก Content Negotiation และ
> Browsable API ที่เป็นจุดขายเด่นของ DRF เห็นภาพรวมของ `REST_FRAMEWORK` settings dict
> เป็นครั้งแรก และปิดท้ายด้วยการเขียน `APIView` แรกในชีวิตที่ endpoint `/api/ping/` คืนค่า
> JSON สถานะระบบ พร้อมใช้ `rest_framework.status` แทนเลข HTTP status ตรง ๆ เมื่อจบ Part นี้
> คุณจะมีรากฐานที่แน่นพอจะเริ่มเขียน Serializer จริงจังใน Part 040 ถัดไป

---

## สารบัญของ Part นี้

- ขั้นตอนที่ 381: REST คืออะไร — หลักการ REST และทำไมต้องมี API แยกจากหน้าเว็บ HTML
- ขั้นตอนที่ 382: ติดตั้ง Django REST Framework
- ขั้นตอนที่ 383: DRF `Request`/`Response` object เทียบกับ Django `HttpRequest`/`HttpResponse`
- ขั้นตอนที่ 384: Browsable API — ทดสอบ API ผ่านเบราว์เซอร์ได้ทันที
- ขั้นตอนที่ 385: ภาพรวม `REST_FRAMEWORK` settings dict ใน settings.py
- ขั้นตอนที่ 386: Content Negotiation — JSONRenderer, BrowsableAPIRenderer และ renderer อื่น ๆ
- ขั้นตอนที่ 387: เขียน `APIView` แรกแบบง่ายที่สุด คืนค่า JSON ตรง ๆ
- ขั้นตอนที่ 388: `rest_framework.status` module — ใช้ constant แทนเลข HTTP status ตรง ๆ
- ขั้นตอนที่ 389: เปรียบเทียบ DRF กับ `JsonResponse` ธรรมดาของ Django เอง
- ขั้นตอนที่ 390: สรุปและแบบฝึกหัด — สร้าง endpoint แรก `/api/ping/`

---

## ขั้นตอนที่ 381: REST คืออะไร — หลักการ REST และทำไมต้องมี API แยกจากหน้าเว็บ HTML

### 381.1 ทบทวนสิ่งที่คุณสร้างมาตลอด Phase 1-4

ก่อนเริ่ม Phase 5 ขอให้ทบทวนสิ่งที่คุณทำมาตลอด 38 Part ที่ผ่านมาก่อน: ทุก view ที่คุณเขียน
ไม่ว่าจะเป็น FBV (Part 007) หรือ CBV (Part 021-024) ล้วนจบลงด้วยการคืนค่า `HttpResponse`
ที่มี **HTML เต็มหน้า** ฝังอยู่ข้างใน เบราว์เซอร์รับ HTML นี้มาแล้ว render เป็นหน้าเว็บให้
ผู้ใช้เห็นโดยตรง นี่คือรูปแบบที่เรียกว่า **Server-Side Rendering (SSR)** หรือบางครั้งเรียก
**Server-Rendered Web Application**

```
Browser ──GET /posts/hello-django/──> Django View
Browser <──HTML เต็มหน้า (มี <html><head><body>...)── Django View
```

รูปแบบนี้ใช้ได้ดีมาตลอดเมื่อ "ผู้บริโภค" ข้อมูลของคุณมีแค่เบราว์เซอร์เพียงอย่างเดียว แต่โลก
ของซอฟต์แวร์ปี 2026 มี "ผู้บริโภค" ข้อมูลหลากหลายกว่านั้นมาก และนี่คือจุดเริ่มต้นของ Phase 5

### 381.2 REST คืออะไรกันแน่

**REST (Representational State Transfer)** ไม่ใช่ framework, ไม่ใช่ library, และไม่ใช่
protocol แต่เป็น **สถาปัตยกรรมรูปแบบหนึ่ง (architectural style)** สำหรับออกแบบระบบเครือข่าย
ที่ถูกเสนอโดย **Roy Fielding** ในวิทยานิพนธ์ปริญญาเอกปี 2000 ชื่อ *"Architectural Styles
and the Design of Network-based Software Architectures"* Fielding เป็นหนึ่งในผู้ร่วมเขียน
มาตรฐาน HTTP/1.1 เอง ดังนั้น REST จึงถูกออกแบบมาให้เข้ากันได้อย่างเป็นธรรมชาติกับ HTTP ที่
เว็บใช้งานอยู่แล้ว

**API ที่ออกแบบตามหลัก REST เรียกว่า RESTful API** — เป็นวิธีสื่อสารระหว่างระบบสองระบบ
(ปกติคือ client กับ server) โดยส่งข้อมูลกันด้วยรูปแบบที่มีโครงสร้างชัดเจน (ส่วนใหญ่คือ JSON
ในปัจจุบัน) แทนที่จะส่ง HTML ที่ผสมทั้งข้อมูลและการแสดงผลเข้าด้วยกัน

### 381.3 หลักการ 6 ข้อของ REST (REST Architectural Constraints)

Fielding นิยาม REST ด้วยข้อจำกัด (constraints) 6 ข้อ ที่ระบบใดก็ตามต้องทำตามทั้งหมดถึงจะ
เรียกว่า "RESTful" อย่างแท้จริง:

| ข้อจำกัด | ความหมาย | ตัวอย่างในบริบท Django |
|---|---|---|
| **1. Client-Server** | แยก client (ผู้ขอข้อมูล) กับ server (ผู้ให้ข้อมูล) ออกจากกันชัดเจน พัฒนาแยกกันได้อิสระ | Frontend React กับ Django backend แยก repository กันได้สนิท |
| **2. Statelessness** | แต่ละ request ต้องมีข้อมูลครบถ้วนในตัวเอง server **ห้ามจำ** สถานะของ client ไว้ระหว่าง request | ทุก request ต้องแนบ token/credential ของตัวเองมาเสมอ ไม่พึ่งพา session ที่ server จำไว้จาก request ก่อนหน้า |
| **3. Cacheable** | Response ต้องระบุชัดเจนว่า cache ได้หรือไม่ เพื่อลดจำนวน request ที่ไม่จำเป็น | ใช้ HTTP header `Cache-Control`, `ETag` (จะเจาะลึกใน Phase 8 เรื่อง Caching) |
| **4. Uniform Interface** | ใช้อินเทอร์เฟซเดียวกันทั้งระบบ: URL แทน "ทรัพยากร (resource)", HTTP verb แทน "การกระทำ" | `/api/posts/5/` คือทรัพยากร, `DELETE` คือการกระทำ ไม่ใช่ `/api/deletePost?id=5` |
| **5. Layered System** | Client ไม่จำเป็นต้องรู้ว่าคุยกับ server ตัวจริงโดยตรง หรือผ่าน proxy/load balancer/cache กี่ชั้น | Nginx reverse proxy หน้า Django, CDN คั่นหน้า static/media (Phase 11) |
| **6. Code on Demand (ไม่บังคับ)** | Server ส่งโค้ดที่ execute ได้กลับไปให้ client รันเองได้ (ไม่บังคับ เป็นข้อเดียวที่ optional) | เช่นส่ง JavaScript กลับไปให้ browser รัน (พบน้อยในบริบท API-only) |

**ข้อจำกัดที่สำคัญที่สุดสำหรับหลักสูตรนี้คือข้อ 2 (Statelessness) และข้อ 4 (Uniform
Interface)** เพราะเป็นสองข้อที่ส่งผลโดยตรงต่อวิธีที่เราจะออกแบบ URL, View, และระบบ
Authentication ตลอดทั้ง Phase 5

### 381.4 เจาะลึก Statelessness: ทำไมสำคัญมาก

**Statelessness** หมายความว่า server **ไม่เก็บสถานะการสนทนา (conversation state)** ของ
client ไว้ระหว่าง request แต่ละครั้ง ทุก request ต้องพกพาข้อมูลทุกอย่างที่จำเป็นต่อการ
ประมวลผลมาในตัวมันเองอย่างสมบูรณ์

เปรียบเทียบกับสิ่งที่คุณเรียนใน Part 034 (Django Sessions และ Cookies):

```
❌ แบบมี state (Session-based, เหมาะกับเว็บ HTML แบบดั้งเดิม):
   1. ผู้ใช้ login → server สร้าง session เก็บไว้ที่ฝั่ง server (หรือ database)
   2. server ส่ง session ID กลับไปเป็น cookie
   3. Request ถัดไปทุกครั้ง browser แนบ cookie นั้นมาอัตโนมัติ
   4. server เปิด session เดิมขึ้นมาดูว่า "ผู้ใช้คนนี้คือใคร"
      → server ต้อง "จำ" ว่า session ID นี้ผูกกับ user คนไหน (มี state ฝั่ง server)

✅ แบบ Stateless (เหมาะกับ REST API):
   1. ผู้ใช้ login → server ออก token (เช่น JWT) ให้ครั้งเดียว
   2. Client เก็บ token นั้นไว้เอง (ไม่ใช่ server เก็บ)
   3. Request ถัดไปทุกครั้ง client แนบ token มาใน header เอง
      (Authorization: Bearer <token>)
   4. server ตรวจสอบ token ที่แนบมา "ในทุก request" โดยไม่ต้องเปิดดู state เดิมที่เก็บไว้
      → server ไม่ต้อง "จำ" อะไรเกี่ยวกับ client เลยระหว่าง request
```

ข้อดีของ Statelessness:

- **Scale ได้ง่ายกว่ามาก**: เพราะ server แต่ละตัวไม่ต้องแชร์ state ระหว่างกัน จะเพิ่มจำนวน
  server (horizontal scaling) กี่ตัวก็ได้ โดยไม่ต้องกังวลว่า request ที่ 2 ของผู้ใช้คนเดิม
  จะไปตกที่ server คนละตัวกับ request แรก
- **ทดสอบง่ายกว่า**: แต่ละ request ทดสอบแยกได้อิสระ ไม่ต้องจำลองลำดับ state ก่อนหน้า
- **Debug ง่ายกว่า**: อ่าน request เดียวก็รู้ทุกอย่างที่จำเป็น ไม่ต้องไล่ดูประวัติ session

เราจะกลับมาเจาะลึกเรื่อง Token Authentication และ JWT (ซึ่งเป็นการนำหลัก Statelessness มา
ใช้จริง) ใน **Part 046** แต่ตอนนี้ขอให้เข้าใจหลักการนี้ให้แม่นก่อน เพราะมันจะอธิบายว่าทำไม
DRF ถึงไม่แนะนำให้พึ่งพา Session Authentication เป็นค่าเริ่มต้นสำหรับ API ที่ให้บริการ
mobile app หรือ frontend แยก

### 381.5 เจาะลึก Uniform Interface: Resource-Based URL

หลักการ **Resource-Based URL** หมายความว่า URL ควรแทน **"สิ่งที่เป็นทรัพยากร (noun)"**
ไม่ใช่ **"การกระทำ (verb)"** เพราะ "การกระทำ" ถูกสื่อผ่าน **HTTP method** อยู่แล้ว

```
❌ RPC-style (Remote Procedure Call) — URL บอกการกระทำ (พบบ่อยใน API รุ่นเก่า):
   GET  /getAllPosts
   GET  /getPostById?id=5
   POST /createNewPost
   POST /updatePost?id=5
   POST /deletePost?id=5

✅ REST-style — URL บอกทรัพยากร, HTTP method บอกการกระทำ:
   GET    /api/posts/          → ดึงรายการ Post ทั้งหมด
   POST   /api/posts/          → สร้าง Post ใหม่
   GET    /api/posts/5/        → ดึง Post ที่ id=5
   PUT    /api/posts/5/        → แก้ไข Post ที่ id=5 (ทั้ง object)
   PATCH  /api/posts/5/        → แก้ไข Post ที่ id=5 (บางส่วน)
   DELETE /api/posts/5/        → ลบ Post ที่ id=5
```

สังเกตว่าแบบ REST-style **URL เดียวกัน (`/api/posts/` หรือ `/api/posts/5/`) ทำหน้าที่ต่างกัน
ตาม HTTP method ที่ใช้เรียก** นี่คือแก่นของ Uniform Interface — ไม่ต้องมี URL แยกสำหรับทุก
การกระทำเหมือนแบบ RPC-style ที่ URL เพิ่มขึ้นเรื่อย ๆ ตามจำนวนฟีเจอร์

### 381.6 ตาราง HTTP Verb และความหมายมาตรฐานใน REST

| HTTP Method | ความหมาย (Semantic) | Idempotent? | Safe? | เทียบเท่า CRUD |
|---|---|---|---|---|
| `GET` | ดึงข้อมูล ไม่มีผลข้างเคียงต่อฐานข้อมูล | ✅ ใช่ | ✅ ใช่ | Read |
| `POST` | สร้างทรัพยากรใหม่ (หรือ trigger การกระทำที่ไม่ idempotent) | ❌ ไม่ | ❌ ไม่ | Create |
| `PUT` | แทนที่ทรัพยากรทั้งหมดด้วยข้อมูลใหม่ (ต้องส่งครบทุก field) | ✅ ใช่ | ❌ ไม่ | Update (เต็มรูปแบบ) |
| `PATCH` | แก้ไขทรัพยากรบางส่วน (ส่งเฉพาะ field ที่เปลี่ยน) | ⚠️ ขึ้นกับการ implement | ❌ ไม่ | Update (บางส่วน) |
| `DELETE` | ลบทรัพยากร | ✅ ใช่ | ❌ ไม่ | Delete |

**Idempotent** หมายถึง "เรียกซ้ำกี่ครั้งก็ได้ผลลัพธ์สุดท้ายเหมือนเดิม" เช่น `DELETE
/api/posts/5/` เรียกครั้งแรก Post หายไป เรียกซ้ำครั้งที่สองก็ยังคือ "Post ไม่มีอยู่" (แม้จะ
ได้ 404 แทน 204 ก็ตาม) เทียบกับ `POST /api/posts/` ที่เรียกซ้ำ 3 ครั้งจะได้ Post ใหม่ 3
records — นี่คือเหตุผลที่ `POST` ไม่ idempotent

**Safe** หมายถึง "ไม่เปลี่ยนแปลงสถานะของระบบเลย" มีแค่ `GET`, `HEAD`, `OPTIONS` เท่านั้นที่
เป็น safe method — นี่คือเหตุผลเชิงหลักการที่ **ห้ามใช้ `GET` สำหรับการกระทำที่แก้ไขข้อมูล**
แม้จะเขียนโค้ดให้ทำได้ทางเทคนิคก็ตาม (ทบทวนแนวคิดนี้จาก Part 007 ขั้นตอนที่ 64 ที่เคยเตือน
เรื่อง CSRF และ `GET` สำหรับแก้ไขข้อมูล)

### 381.7 ทำไมต้องมี API แยกจากหน้าเว็บ HTML

คำถามสำคัญที่สุดของขั้นตอนนี้: ในเมื่อ Phase 1-4 สร้างเว็บที่ทำงานได้สมบูรณ์แบบอยู่แล้ว
(ทั้ง authentication, permission, form) ทำไมต้องเสียเวลาสร้าง API แยกอีกชุด?

| สถานการณ์ | ทำไม HTML render ธรรมดาไม่พอ |
|---|---|
| **Mobile App (iOS/Android)** | แอปมือถือไม่มี "เบราว์เซอร์" render HTML ภายในตัว ต้องการข้อมูลดิบ (JSON) มาแสดงผลด้วย UI native ของตัวเอง |
| **Single-Page Application (React, Vue, Angular)** | Frontend ต้องการแค่ข้อมูล ไม่ต้องการ HTML เต็มหน้าที่ผสมการแสดงผลมาด้วย เพราะ Frontend จัดการ render เอง |
| **Third-Party Integration** | บริษัทอื่นต้องการดึงข้อมูลจากระบบคุณไปใช้ในระบบของเขาเอง (เช่น partner เชื่อม catalog สินค้า) ต้องการ JSON ไม่ใช่ HTML |
| **Microservices Architecture** | ระบบภายในองค์กรหลายทีมต้องคุยกันเอง (service-to-service) ไม่มี "ผู้ใช้" มานั่งดูหน้าเว็บเลย |
| **IoT / Smart Device** | อุปกรณ์ฝังตัว (embedded device) ส่งและรับข้อมูลแบบเบา ๆ ไม่มีความสามารถ render HTML |
| **Testing และ Automation** | สคริปต์ทดสอบอัตโนมัติดึงข้อมูลตรง ๆ จาก endpoint ง่ายกว่า parse HTML |
| **Multiple Frontend สำหรับระบบเดียว** | เว็บ + มือถือ + Smart TV app ใช้ backend เดียวกัน แต่ frontend คนละแบบ ต้องมี API กลางให้ทุกฝั่งเรียก |

ข้อสังเกตสำคัญ: **HTML คือรูปแบบการแสดงผล (presentation format) ที่ผูกติดกับเบราว์เซอร์
เท่านั้น** ในขณะที่ **JSON คือรูปแบบข้อมูล (data format) ที่เป็นกลาง** ไม่ผูกกับภาษาหรือ
platform ใดเลย — นี่คือเหตุผลเชิงสถาปัตยกรรมที่แท้จริงว่าทำไมโลกซอฟต์แวร์ถึงย้ายไปทาง
**"Backend เป็น API + Frontend แยกอิสระ"** มากขึ้นเรื่อย ๆ ตั้งแต่ยุค 2010 เป็นต้นมา

### 381.8 ภาพรวมสถาปัตยกรรมที่กำลังจะเปลี่ยนไป

```
สถาปัตยกรรมเดิม (Phase 1-4): Server-Rendered Monolith
┌──────────┐   HTTP    ┌─────────────────────────────┐    SQL    ┌──────────┐
│ Browser  │ <───────> │ Django (View + Template)      │ <───────> │ Database │
└──────────┘   HTML    └─────────────────────────────┘           └──────────┘


สถาปัตยกรรมใหม่ (Phase 5 เป็นต้นไป): API-First Backend
┌──────────┐                                          ┌──────────┐
│ Browser  │──┐                                    ┌──│ Mobile   │
│ (React)  │  │   HTTP (JSON)   ┌────────────────┐  │  │ App      │
└──────────┘  ├────────────────>│ Django + DRF   │<─┤  └──────────┘
┌──────────┐  │                 │ (API Views)     │  │  ┌──────────┐
│ Partner  │──┘                 └────────┬────────┘  └──│ Admin    │
│ System   │                             │ SQL           │ Panel    │
└──────────┘                             ▼               └──────────┘
                                    ┌──────────┐
                                    │ Database │
                                    └──────────┘
```

สังเกตว่า **Django Admin ที่คุณสร้างมาตั้งแต่ Part 003 ยังคงใช้ HTML rendering แบบเดิมได้
ตามปกติ** — เราไม่ได้ทิ้ง SSR ไปทั้งหมด แต่จะ **เพิ่ม** ชั้น API เข้ามาให้บริการ client
ประเภทอื่นควบคู่กันไป โปรเจกต์จริงจำนวนมากใช้ทั้งสองแบบผสมกันในระบบเดียว (Django Admin +
เว็บสาธารณะแบบ SSR + API สำหรับ mobile app)

### 381.9 REST ไม่ใช่มาตรฐานตายตัว แต่เป็นแนวปฏิบัติ

ข้อควรรู้เชิงลึกสำหรับความเป็นมืออาชีพ: **ไม่มี "REST compliance checker" อย่างเป็นทางการ**
API ส่วนใหญ่ที่เรียกตัวเองว่า "RESTful" ในโลกจริง (รวมถึง API ของ GitHub, Twitter/X, Stripe)
ไม่ได้ทำตามข้อจำกัดทั้ง 6 ข้อของ Fielding อย่างเคร่งครัด 100% (โดยเฉพาะข้อ HATEOAS ซึ่งเป็น
ส่วนย่อยของ Uniform Interface ที่มักถูกมองข้าม) สิ่งที่สำคัญกว่าในทางปฏิบัติคือการยึดหลัก
**Resource-based URL + HTTP verb มีความหมายตรงตามมาตรฐาน + Stateless** ให้สม่ำเสมอทั้งระบบ
ซึ่งเป็นสามหลักการที่ DRF ทั้ง framework ถูกออกแบบมาให้ทำตามโดยอัตโนมัติ

### 381.10 สรุปแนวคิดของขั้นตอนนี้

- REST คือสถาปัตยกรรมรูปแบบหนึ่ง เสนอโดย Roy Fielding ปี 2000 ไม่ใช่ framework หรือ protocol
- หลักการ 6 ข้อของ REST: Client-Server, Statelessness, Cacheable, Uniform Interface,
  Layered System, Code on Demand (optional)
- **Statelessness**: server ไม่จำสถานะ client ระหว่าง request แต่ละครั้งต้องพกข้อมูลครบ
- **Uniform Interface**: URL แทนทรัพยากร (noun), HTTP method แทนการกระทำ (verb)
- API แยกจาก HTML render จำเป็นเมื่อมี client หลายประเภท: mobile app, SPA, partner system,
  microservices, IoT
- Django Admin และเว็บ SSR ที่เรียนมาตลอด Phase 1-4 ยังใช้งานคู่กับ API ได้ ไม่ต้องทิ้งไป

---

## ขั้นตอนที่ 382: ติดตั้ง Django REST Framework

### 382.1 Django REST Framework คืออะไร

**Django REST Framework (DRF)** คือ third-party package ที่ได้รับความนิยมสูงสุดสำหรับสร้าง
RESTful API บน Django เขียนโดย Tom Christie เปิดตัวครั้งแรกในปี 2011 และกลายเป็นมาตรฐาน
โดยพฤตินัย (de facto standard) ของวงการ Django ในปัจจุบัน แม้แต่เอกสารทางการของ Django เอง
ก็แนะนำ DRF เมื่อพูดถึงการสร้าง API

DRF ไม่ได้เพิ่ม "ความสามารถใหม่" ที่ Django ทำไม่ได้เลย (Django เปล่า ๆ ก็สร้าง JSON API ได้
ด้วย `JsonResponse` อย่างที่เห็นใน Part 007) แต่ DRF ให้ **เครื่องมือระดับสูง** ที่ลดโค้ดซ้ำ
ซ้อนลงมหาศาลสำหรับงานที่พบบ่อยในการสร้าง API: serialization, validation, authentication,
permission, pagination, filtering, browsable UI สำหรับทดสอบ, และ API documentation

### 382.2 ตรวจสอบเวอร์ชัน Django และ Python ที่รองรับ

ก่อนติดตั้ง ควรตรวจสอบตาราง compatibility ระหว่าง DRF, Django, และ Python เสมอ (DRF ออก
เวอร์ชันใหม่ตามหลัง Django release เสมอ):

| DRF Version | รองรับ Django | รองรับ Python |
|---|---|---|
| 3.15.x (ล่าสุด ณ ที่เขียนหลักสูตรนี้) | 4.2, 5.0, 5.1 | 3.8 - 3.12 |
| 3.14.x | 3.2, 4.0, 4.1, 4.2 | 3.6 - 3.11 |

หลักสูตรนี้ใช้ **Django 5.1.x** (ตาม Part 001) คู่กับ **DRF 3.15+** ซึ่งเป็นคู่เวอร์ชันล่าสุด
ที่รองรับกันอย่างสมบูรณ์

### 382.3 ติดตั้งผ่าน pip

Activate virtual environment ของโปรเจกต์ `secureblog` ให้เรียบร้อยก่อนเสมอ (ทบทวนกฎเหล็ก
จาก Part 001 ขั้นตอนที่ 8: ห้ามติดตั้ง package แบบ global เด็ดขาด):

```bash
# ตรวจสอบว่า venv ถูก activate อยู่ (ต้องเห็น (venv) หน้า prompt)
source venv/bin/activate    # macOS/Linux
# venv\Scripts\activate      # Windows

# ติดตั้ง Django REST Framework
pip install djangorestframework

# ตรวจสอบเวอร์ชันที่ติดตั้ง
python -m pip show djangorestframework
```

ผลลัพธ์ที่คาดหวังจาก `pip show`:

```
Name: djangorestframework
Version: 3.15.2
Summary: Web APIs for Django, made easy.
Home-page: https://www.django-rest-framework.org/
Author: Tom Christie
License: BSD
Requires: django
Required-by:
```

### 382.4 เพิ่มเข้า `INSTALLED_APPS`

DRF เป็น Django app ปกติเหมือน app อื่น ๆ ที่คุณเคยเพิ่มมาตลอดหลักสูตร (ทบทวนแนวคิดจาก
Part 005) เปิดไฟล์ `secureblog/settings.py` แล้วเพิ่ม `'rest_framework'` เข้า
`INSTALLED_APPS`:

```python
# secureblog/settings.py

INSTALLED_APPS = [
    'django.contrib.admin',
    'django.contrib.auth',
    'django.contrib.contenttypes',
    'django.contrib.sessions',
    'django.contrib.messages',
    'django.contrib.staticfiles',
    # Third-party apps
    'guardian',              # ติดตั้งแล้วใน Part 038 (django-guardian)
    'rest_framework',        # <-- เพิ่มบรรทัดนี้
    # Local apps
    'blog',
    'accounts',
]
```

**ข้อควรระวัง**: ต้องเป็น `'rest_framework'` (ตัวพิมพ์เล็กทั้งหมด มี underscore) ไม่ใช่
`'djangorestframework'` หรือ `'django_rest_framework'` — ชื่อ package ที่ติดตั้งผ่าน pip
(`djangorestframework`) กับชื่อ Python app ที่ import จริง (`rest_framework`) **ไม่ตรงกัน**
นี่คือจุดพลาดที่มือใหม่เจอบ่อยที่สุดตอนติดตั้ง DRF ครั้งแรก

### 382.5 เพิ่ม URL สำหรับ Browsable API Login/Logout (เตรียมไว้สำหรับขั้นตอนที่ 384)

DRF มี URL สำเร็จรูปสำหรับหน้า login/logout ของ Browsable API (จะเห็นผลจริงในขั้นตอนที่ 384)
เพิ่มเข้า root `urls.py` ของโปรเจกต์ไว้ตั้งแต่ตอนนี้:

```python
# secureblog/urls.py
from django.contrib import admin
from django.urls import path, include

urlpatterns = [
    path('admin/', admin.site.urls),
    path('accounts/', include('accounts.urls')),
    path('api/', include('blog.api_urls')),          # จะสร้างจริงในขั้นตอนที่ 387
    path('api-auth/', include('rest_framework.urls')),  # login/logout ของ Browsable API
]
```

### 382.6 ตรวจสอบว่าติดตั้งสำเร็จด้วย Python shell

```bash
python manage.py shell
```

```python
>>> import rest_framework
>>> rest_framework.VERSION
'3.15.2'

>>> from rest_framework.views import APIView
>>> from rest_framework.response import Response
>>> APIView
<class 'rest_framework.views.APIView'>
```

หากไม่มี error ใด ๆ แสดงว่า DRF พร้อมใช้งานแล้ว

### 382.7 อัปเดต `requirements.txt`

ทบทวนวินัยจาก Part 001 ขั้นตอนที่ 9.3: ทุกครั้งที่ติดตั้ง package ใหม่ ต้องอัปเดต
`requirements.txt` ทันที:

```bash
pip freeze > requirements.txt
cat requirements.txt
```

```
asgiref==3.8.1
Django==5.1.4
django-guardian==2.4.0
djangorestframework==3.15.2
sqlparse==0.5.1
```

### 382.8 ตารางสรุปสิ่งที่ทำในขั้นตอนนี้

| ขั้นตอน | คำสั่ง/การเปลี่ยนแปลง | จุดประสงค์ |
|---|---|---|
| 1. ติดตั้ง package | `pip install djangorestframework` | เพิ่มไลบรารี DRF เข้า venv |
| 2. เพิ่ม `INSTALLED_APPS` | `'rest_framework'` | ให้ Django รู้จัก app ของ DRF (templates, static ของ Browsable API) |
| 3. เพิ่ม URL สำหรับ auth | `path('api-auth/', include('rest_framework.urls'))` | เตรียม login/logout สำหรับ Browsable API |
| 4. ตรวจสอบการติดตั้ง | `python -c "import rest_framework; print(rest_framework.VERSION)"` | ยืนยันว่าติดตั้งสำเร็จและ import ได้ |
| 5. อัปเดต dependency file | `pip freeze > requirements.txt` | บันทึก dependency ให้ทีม/อนาคตติดตั้งตามได้ |

---

## ขั้นตอนที่ 383: DRF `Request`/`Response` object เทียบกับ Django `HttpRequest`/`HttpResponse` ธรรมดา

### 383.1 ปัญหาของ `HttpRequest`/`HttpResponse` ธรรมดาเมื่อสร้าง API

ทบทวนจาก Part 007: `HttpRequest` (`request` object ที่ Django ส่งเข้า view ทุกตัว) มี
`request.GET` (query string), `request.POST` (form-encoded body), และ `request.body` (raw
bytes) แยกกันตามประเภทของข้อมูลที่ส่งเข้ามา ปัญหาคือเมื่อสร้าง API ที่ต้องรับข้อมูลได้ทั้ง
JSON, form data, และ multipart (file upload) พร้อมกัน คุณต้อง parse `request.body` เองด้วย
`json.loads()` ทุกครั้งที่ client ส่ง JSON มา:

```python
# แบบ Django ธรรมดา — ต้อง parse JSON เองทุกครั้ง
import json
from django.http import JsonResponse

def create_post_plain(request):
    if request.method == 'POST':
        try:
            data = json.loads(request.body)   # ต้อง parse เอง
        except json.JSONDecodeError:
            return JsonResponse({'error': 'invalid JSON'}, status=400)
        title = data.get('title')
        # ... ทำงานต่อ
        return JsonResponse({'title': title}, status=201)
```

DRF แก้ปัญหานี้ด้วยการห่อ `HttpRequest` เดิมไว้ข้างในคลาส **`Request`** ของตัวเอง ที่ทำ
**parsing อัตโนมัติ** ตาม `Content-Type` header ที่ client ส่งมา

### 383.2 `rest_framework.request.Request`: ห่อ (wrap) ไม่ใช่แทนที่

จุดสำคัญที่ต้องเข้าใจ: **DRF `Request` ไม่ได้แทนที่ `HttpRequest` ของ Django แต่ "ห่อ"
(compose) มันไว้ข้างใน** — ทุก attribute เดิมของ `HttpRequest` (เช่น `request.user`,
`request.META`, `request.session`) ยังคงเรียกใช้ได้ปกติทุกประการผ่าน DRF `Request` เพราะ
DRF `Request` ส่งต่อ (delegate) ไปยัง `HttpRequest` ต้นฉบับที่เก็บไว้ใน `request._request`

```python
# แนวคิดจากซอร์สโค้ดจริงของ rest_framework.request.Request (ย่อ)
class Request:
    def __init__(self, request, parsers=None, authenticators=None, ...):
        self._request = request          # HttpRequest ต้นฉบับของ Django ถูกเก็บไว้ตรงนี้
        self.parsers = parsers or ()
        self.authenticators = authenticators or ()
        # ...

    def __getattr__(self, attr):
        # attribute ไหนที่ Request ไม่มีเอง จะ "ส่งต่อ" ไปหา self._request อัตโนมัติ
        try:
            return getattr(self._request, attr)
        except AttributeError:
            return self.__getattribute__(attr)

    @property
    def data(self):
        # คืนข้อมูลที่ parse แล้วจาก body ไม่ว่าจะเป็น JSON, form, multipart
        if not hasattr(self, '_full_data'):
            self._parse()
        return self._full_data
```

นี่คือเหตุผลที่โค้ดแบบนี้ทำงานได้ปกติแม้อยู่ใน DRF view: `request.user`,
`request.session`, `request.META['REMOTE_ADDR']` — ทุกอย่างที่คุณคุ้นเคยจาก Phase 3-4 ยัง
ใช้ได้เหมือนเดิมทุกประการ

### 383.3 ตารางเปรียบเทียบ attribute ที่สำคัญ

| ต้องการข้อมูล | Django `HttpRequest` | DRF `Request` | หมายเหตุ |
|---|---|---|---|
| ข้อมูลจาก body (parsed) | `request.POST` (form-encoded เท่านั้น) หรือ `json.loads(request.body)` เอง | **`request.data`** | `request.data` parse ให้อัตโนมัติไม่ว่าจะเป็น JSON, form, multipart |
| Query string parameters | `request.GET` | **`request.query_params`** | ทำงานเหมือนกันทุกประการ DRF แค่เปลี่ยนชื่อให้สื่อความหมายชัดกว่า (`request.GET` ทำให้สับสนว่าเกี่ยวกับ HTTP GET method) |
| ไฟล์ที่อัปโหลด | `request.FILES` | `request.FILES` (เหมือนเดิม) | ไม่เปลี่ยน ยังใช้ชื่อเดิม |
| ผู้ใช้ปัจจุบัน | `request.user` | `request.user` (เหมือนเดิม) | เหมือนกัน แต่ DRF เติม authentication scheme เพิ่มเติมได้ (Part 045) |
| HTTP method | `request.method` | `request.method` (เหมือนเดิม) | เหมือนกันทุกประการ |
| Raw bytes ของ body | `request.body` | `request.body` (เหมือนเดิม, ยังเรียกได้) | DRF เก็บ raw ไว้เผื่อ parser เอง ก่อนแปลงเป็น `.data` |
| Content negotiation | ไม่มีในตัว ต้องเขียนเอง | `request.accepted_renderer`, `request.accepted_media_type` | DRF คำนวณให้อัตโนมัติจาก header `Accept` |

### 383.4 ทดลอง `request.data` กับ Content-Type ต่าง ๆ

```python
# blog/api_views.py (ตัวอย่างสาธิต — จะเขียน APIView จริงจังในขั้นตอนที่ 387)
from rest_framework.views import APIView
from rest_framework.response import Response


class EchoDataView(APIView):
    """สาธิตว่า request.data parse ข้อมูลให้อัตโนมัติไม่ว่า Content-Type จะเป็นอะไร"""

    def post(self, request):
        return Response({
            'received_data': request.data,
            'content_type_ที่ client ส่งมา': request.content_type,
            'query_params': request.query_params,
        })
```

ทดสอบด้วย `curl` ส่งข้อมูลสามแบบที่ต่างกัน:

```bash
# แบบที่ 1: ส่งเป็น JSON
curl -X POST http://127.0.0.1:8000/api/echo/ \
  -H "Content-Type: application/json" \
  -d '{"title": "สวัสดี Django REST Framework"}'

# แบบที่ 2: ส่งเป็น form-encoded (เหมือนฟอร์ม HTML ธรรมดา)
curl -X POST http://127.0.0.1:8000/api/echo/ \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "title=สวัสดีแบบฟอร์ม"

# แบบที่ 3: ส่งเป็น multipart (เหมือนอัปโหลดไฟล์)
curl -X POST http://127.0.0.1:8000/api/echo/ \
  -F "title=สวัสดีแบบมัลติพาร์ต"
```

ทั้งสามแบบจะได้ `request.data['title']` ออกมาถูกต้องเหมือนกันทุกครั้ง แม้ว่า `Content-Type`
จะต่างกันโดยสิ้นเชิง — นี่คือสิ่งที่ `request.POST` ของ Django ธรรมดา**ทำไม่ได้เอง** (เพราะ
`request.POST` เข้าใจแค่ `application/x-www-form-urlencoded` และ `multipart/form-data`
เท่านั้น ไม่รู้จัก `application/json` เลย)

### 383.5 `rest_framework.response.Response`: Content Negotiation อัตโนมัติสำหรับ output

เช่นเดียวกับฝั่ง input (`Request`) DRF มีคลาสสำหรับฝั่ง output ชื่อ **`Response`** ที่แตกต่าง
จาก `JsonResponse`/`HttpResponse` ของ Django ตรงที่ **ไม่ต้องระบุ format ตายตัวตอนสร้าง**
DRF จะเลือก format ที่เหมาะสม (JSON, Browsable HTML, XML ถ้าติดตั้ง renderer เพิ่ม ฯลฯ) ให้
อัตโนมัติตาม header `Accept` ที่ client ส่งมา (จะเจาะลึกกลไกนี้เต็มรูปแบบในขั้นตอนที่ 386)

```python
# เปรียบเทียบโค้ดฝั่ง output
from django.http import JsonResponse
from rest_framework.response import Response

# Django ธรรมดา — คืน JSON เสมอ ไม่ว่า client จะขอ format ไหน
def django_view(request):
    return JsonResponse({'status': 'ok'})

# DRF — คืน "ข้อมูล Python" ธรรมดา แล้วให้ Response ตัดสินใจแปลงเป็น format
#        ที่เหมาะสมกับ client เอง (JSON API ปกติ, หรือหน้า HTML สวย ๆ ถ้าเปิดผ่านเบราว์เซอร์)
class StatusView(APIView):
    def get(self, request):
        return Response({'status': 'ok'})
```

### 383.6 ตารางเปรียบเทียบ `Response` กับ `JsonResponse`/`HttpResponse`

| คุณสมบัติ | `HttpResponse`/`JsonResponse` (Django) | `Response` (DRF) |
|---|---|---|
| รับข้อมูลชนิดไหนตอนสร้าง | `HttpResponse`: string/bytes, `JsonResponse`: dict (serialize เป็น JSON string ทันที) | Python data structure ดิบ (dict, list) ยังไม่ถูกแปลงเป็น string จนกว่า renderer จะทำงาน |
| เลือก format output | ตายตัว (`JsonResponse` = JSON เสมอ) | เลือกอัตโนมัติจาก `Accept` header ผ่าน Content Negotiation |
| ใช้กับ `status=` ตัวเลขได้ไหม | ✅ ได้ (`status=404`) | ✅ ได้ และแนะนำให้ใช้ `rest_framework.status` แทนตัวเลข (ขั้นตอนที่ 388) |
| แสดงผลใน Browsable API ได้ไหม | ❌ ไม่ได้ (ไม่รู้จัก renderer ของ DRF) | ✅ ได้โดยอัตโนมัติ (ขั้นตอนที่ 384) |
| ต้องใช้กับ view ประเภทไหน | View ใดก็ได้ของ Django (FBV/CBV ธรรมดา) | ต้องใช้ภายใน DRF view เท่านั้น (`APIView`, `@api_view`, ViewSet) |

**ข้อควรระวังสำคัญ**: `Response` ของ DRF **ใช้ได้เฉพาะภายใน DRF view เท่านั้น** (view ที่รับ
DRF `Request` เข้ามา) ถ้านำ `Response` ไปใช้ใน Django FBV/CBV ธรรมดาที่ไม่ได้ผ่าน DRF
`Request` จะเกิด error เพราะกลไก content negotiation ต้องพึ่งข้อมูลที่ DRF แนบไว้กับ
`request` ตอน dispatch (จะเห็นรายละเอียดนี้ในขั้นตอนที่ 387 เมื่อเจาะลึก `APIView`)

### 383.7 สรุปแนวคิดของขั้นตอนนี้

- DRF `Request` **ห่อ** (ไม่ใช่แทนที่) `HttpRequest` เดิม — attribute เดิมทั้งหมดยังใช้ได้
- `request.data` แทนที่ `request.POST`/`json.loads(request.body)` โดย parse อัตโนมัติทุก
  `Content-Type` (JSON, form, multipart)
- `request.query_params` ทำงานเหมือน `request.GET` ทุกประการ เปลี่ยนแค่ชื่อให้สื่อความหมาย
- DRF `Response` รับ Python data structure ดิบ แล้วให้ renderer เลือก format output ตาม
  content negotiation แทนที่จะ hardcode เป็น JSON เสมอ
- `Response` ใช้ได้เฉพาะภายใน DRF view เท่านั้น

---

## ขั้นตอนที่ 384: Browsable API — ทดสอบ API ผ่านเบราว์เซอร์ได้ทันที

### 384.1 ปัญหาที่ Browsable API แก้: ทดสอบ JSON API แบบเดิมยุ่งยากแค่ไหน

ก่อนมี DRF การทดสอบ JSON API ต้องพึ่งเครื่องมือภายนอกเสมอ เช่น `curl`, Postman, หรือเขียน
สคริปต์ Python เอง เพราะเบราว์เซอร์เปิด URL ของ JSON API ตรง ๆ จะเห็นแค่ข้อความ JSON ดิบ ๆ
ไม่มี format สวยงาม ไม่มีฟอร์มให้กรอกทดสอบ `POST`/`PUT`/`DELETE`

```
เปิด http://127.0.0.1:8000/api/posts/ ด้วยเบราว์เซอร์แบบ JsonResponse ธรรมดา:

[{"id": 1, "title": "Hello Django", "is_published": true}, {"id": 2, ...}]

→ เห็นแค่ text ดิบ ไม่มีปุ่มกด ไม่มีฟอร์ม ต้องเปิด Postman แยกถ้าจะทดสอบ POST
```

### 384.2 Browsable API คืออะไร

**Browsable API** คือฟีเจอร์เด่นที่สุดอย่างหนึ่งของ DRF: เมื่อเปิด endpoint API ของ DRF
ผ่าน**เบราว์เซอร์โดยตรง** (ไม่ใช่ผ่าน `curl`/Postman/JavaScript `fetch`) DRF จะตรวจจับได้ว่า
เบราว์เซอร์ส่ง header `Accept: text/html` มา (ค่า default ของทุกเบราว์เซอร์เมื่อพิมพ์ URL ใน
address bar) แล้วเลือก render เป็น **หน้าเว็บ HTML ที่ใช้งานได้จริง** แทน JSON ดิบ พร้อม:

- แสดงข้อมูล JSON แบบจัดรูปแบบสวยงาม พร้อม syntax highlighting
- ปุ่มกดสำหรับ `GET`, `OPTIONS` และฟอร์มสำหรับ `POST`/`PUT`/`PATCH`/`DELETE` (ถ้า view
  รองรับ method นั้น และผู้ใช้มีสิทธิ์)
- แสดงชื่อ view, docstring ของ view เป็นคำอธิบาย
- ปุ่ม Login/Logout (จาก URL ที่เพิ่มไว้ในขั้นตอนที่ 382.5)
- แสดง HTTP status code, response header ทั้งหมดที่ส่งกลับจริง

นี่คือกลไกเดียวกับที่อธิบายไว้ในขั้นตอนที่ 383.5-383.6: **Content Negotiation** ทำงานทั้งสอง
ทิศทาง — ทั้งฝั่ง `Request` (parse input) และฝั่ง `Response` (เลือก format output) โดย
Browsable API คือผลลัพธ์ที่มองเห็นได้ของ Content Negotiation ฝั่ง output เมื่อ client คือ
เบราว์เซอร์

### 384.3 ทดลองใช้งานจริง (เมื่อเขียน `APIView` เสร็จในขั้นตอนที่ 387)

```bash
python manage.py runserver
```

จากนั้นเปิดเบราว์เซอร์ไปที่ `http://127.0.0.1:8000/api/ping/` จะเห็นหน้าตาประมาณนี้ (อธิบาย
เป็นโครงสร้าง เพราะเป็นหน้าเว็บ ไม่ใช่โค้ด):

```
┌─────────────────────────────────────────────────────────────┐
│ Django REST framework                     [Login] (มุมขวาบน)  │
├─────────────────────────────────────────────────────────────┤
│ Ping                                                          │
│ ping/                                                          │
│                                                                │
│ GET /api/ping/                                                │
│                                                                │
│ HTTP 200 OK                                                    │
│ Allow: GET, HEAD, OPTIONS                                      │
│ Content-Type: application/json                                 │
│ Vary: Accept                                                    │
│                                                                │
│ {                                                              │
│     "status": "ok",                                             │
│     "message": "pong",                                          │
│     "django_version": "5.1.4"                                   │
│ }                                                              │
│                                                                │
│ [GET] [OPTIONS]                                                 │
└─────────────────────────────────────────────────────────────┘
```

เทียบกับการเปิด URL เดียวกันด้วย `curl -H "Accept: application/json"` จะได้ **JSON ดิบล้วน
ๆ** โดยไม่มี HTML ห่อเลย — นี่คือ Content Negotiation ที่ทำงานจริงตาม header `Accept`

### 384.4 ทำไม Browsable API ถึงมีประโยชน์มากในทางปฏิบัติ

| ประโยชน์ | รายละเอียด |
|---|---|
| **ลดเวลา setup เครื่องมือทดสอบ** | ไม่ต้องเปิด Postman/Insomnia แยกระหว่างพัฒนา เปิดเบราว์เซอร์อย่างเดียวพอ |
| **สื่อสารกับทีม Frontend ง่ายขึ้น** | ส่งลิงก์ endpoint ให้ทีม Frontend เปิดดูโครงสร้างข้อมูลจริงได้ทันที ไม่ต้องอธิบายปากเปล่า |
| **Self-documenting ระดับหนึ่ง** | เห็น HTTP method ที่รองรับ, field ที่ต้องกรอก (ผ่านฟอร์ม auto-generate จาก Serializer ที่จะเรียน Part 040) |
| **Debug ง่ายขึ้น** | เห็น HTTP header เต็มรูปแบบ, error message ที่จัดรูปแบบสวยงามอ่านง่าย |
| **ใช้ทดสอบ Authentication/Permission ได้จริง** | ปุ่ม Login ทำให้ทดสอบ endpoint ที่ต้อง login ได้โดยไม่ต้องเขียนโค้ดแยก |

### 384.5 ปิด/จำกัด Browsable API ในกรณีที่ต้องการ

แม้ Browsable API จะมีประโยชน์มากตอนพัฒนา แต่บาง endpoint (เช่น endpoint ที่ return ข้อมูล
จำนวนมหาศาล หรือ endpoint สาธารณะที่ไม่อยากให้คนทั่วไปคลิกเล่นฟอร์มได้) อาจต้องการปิดไว้
ทำได้โดยกำหนด `renderer_classes` เฉพาะ view นั้น ให้เหลือแค่ `JSONRenderer`:

```python
from rest_framework.views import APIView
from rest_framework.renderers import JSONRenderer
from rest_framework.response import Response


class PublicHeavyDataView(APIView):
    """Endpoint นี้ปิด Browsable API ไว้ เพราะข้อมูลใหญ่มากและเป็น public-facing"""
    renderer_classes = [JSONRenderer]   # ไม่มี BrowsableAPIRenderer ในรายการ

    def get(self, request):
        return Response({'data': list(range(10000))})
```

เมื่อเปิด `PublicHeavyDataView` ด้วยเบราว์เซอร์จะได้ JSON ดิบเสมอ ไม่มีหน้า HTML สวยงามห่อให้
อีกต่อไป (เจาะลึกกลไก `renderer_classes` เต็มรูปแบบในขั้นตอนที่ 386)

### 384.6 สรุปแนวคิดของขั้นตอนนี้

- Browsable API คือการที่ DRF render หน้า HTML ให้ทดสอบ API ได้เมื่อเปิดผ่านเบราว์เซอร์
- กลไกเบื้องหลังคือ Content Negotiation ตาม header `Accept` ที่ client ส่งมา
- เบราว์เซอร์ส่ง `Accept: text/html` โดย default → ได้ HTML สวยงาม
- `curl`/JavaScript ที่ระบุ `Accept: application/json` ชัดเจน → ได้ JSON ดิบ
- ปิด Browsable API เฉพาะ view ได้ด้วยการกำหนด `renderer_classes = [JSONRenderer]`

---

## ขั้นตอนที่ 385: ภาพรวม `REST_FRAMEWORK` settings dict ใน settings.py

### 385.1 DRF settings ทำงานอย่างไร

DRF อ่านค่า configuration ทั้งหมดจาก dict ตัวเดียวชื่อ **`REST_FRAMEWORK`** ใน
`settings.py` ของโปรเจกต์ (คล้ายกับที่ Django เองมี setting เดี่ยว ๆ กระจายหลายตัว แต่ DRF
รวมทุกอย่างไว้ใน namespace เดียว) ถ้าไม่ตั้งค่า `REST_FRAMEWORK` เลย DRF จะใช้ **ค่า default
ที่กำหนดไว้ในตัวเอง** (นิยามอยู่ที่ `rest_framework/settings.py` ในซอร์สโค้ดของ DRF)

```python
# แนวคิดการทำงานของ DRF settings (ย่อจากกลไกจริง)
from rest_framework.settings import api_settings

# ทุกครั้งที่ DRF ต้องการค่า setting ตัวไหน จะดึงผ่าน api_settings
# api_settings จะเช็คก่อนว่า REST_FRAMEWORK ใน settings.py ของโปรเจกต์กำหนดค่านั้นไว้ไหม
# ถ้าไม่มี จะ fallback ไปใช้ค่า DEFAULTS ที่ DRF เตรียมไว้ให้เอง
print(api_settings.DEFAULT_PERMISSION_CLASSES)
# ['rest_framework.permissions.AllowAny']   ← ค่า default ถ้าไม่ตั้งเอง
```

### 385.2 โครงสร้างของ `REST_FRAMEWORK` dict ใน `settings.py`

```python
# secureblog/settings.py

REST_FRAMEWORK = {
    # ---- Permission: ใครมีสิทธิ์เรียก API ได้บ้าง (เจาะลึกเต็มรูปแบบใน Part 045) ----
    'DEFAULT_PERMISSION_CLASSES': [
        'rest_framework.permissions.IsAuthenticatedOrReadOnly',
    ],

    # ---- Authentication: DRF รู้จัก "ผู้ใช้" จาก request ได้อย่างไร (Part 045-046) ----
    'DEFAULT_AUTHENTICATION_CLASSES': [
        'rest_framework.authentication.SessionAuthentication',
        'rest_framework.authentication.TokenAuthentication',
    ],

    # ---- Renderer: format ที่ Response แปลงข้อมูลออกไปได้ (ขั้นตอนที่ 386) ----
    'DEFAULT_RENDERER_CLASSES': [
        'rest_framework.renderers.JSONRenderer',
        'rest_framework.renderers.BrowsableAPIRenderer',
    ],

    # ---- Parser: format ที่ Request รับข้อมูลเข้ามาได้ (ขั้นตอนที่ 383) ----
    'DEFAULT_PARSER_CLASSES': [
        'rest_framework.parsers.JSONParser',
        'rest_framework.parsers.FormParser',
        'rest_framework.parsers.MultiPartParser',
    ],

    # ---- Pagination: แบ่งหน้าผลลัพธ์อัตโนมัติ (เจาะลึกใน Part 047) ----
    'DEFAULT_PAGINATION_CLASS': 'rest_framework.pagination.PageNumberPagination',
    'PAGE_SIZE': 20,

    # ---- Throttling: จำกัดจำนวน request ต่อช่วงเวลา (เจาะลึกใน Part 048) ----
    'DEFAULT_THROTTLE_CLASSES': [
        'rest_framework.throttling.AnonRateThrottle',
        'rest_framework.throttling.UserRateThrottle',
    ],
    'DEFAULT_THROTTLE_RATES': {
        'anon': '100/day',
        'user': '1000/day',
    },

    # ---- Filtering: ค้นหา/กรอง/เรียงลำดับผลลัพธ์ (เจาะลึกใน Part 047) ----
    'DEFAULT_FILTER_BACKENDS': [
        'django_filters.rest_framework.DjangoFilterBackend',
    ],

    # ---- Schema: สร้าง API documentation อัตโนมัติ (เจาะลึกใน Part 049) ----
    'DEFAULT_SCHEMA_CLASS': 'rest_framework.schemas.coreapi.AutoSchema',
}
```

**สำหรับ Part นี้ เราจะยังไม่ตั้งค่าอะไรมากไปกว่าที่จำเป็นสำหรับ endpoint `/api/ping/`** —
ตารางและโค้ดด้านบนคือ**แผนที่รวม**ของทั้ง Phase 5 ให้เห็นภาพว่าแต่ละ key จะถูกเจาะลึกใน Part
ไหนบ้าง อย่าเพิ่งกังวลถ้ายังไม่เข้าใจทุกบรรทัด

### 385.3 ตารางสรุป key สำคัญของ `REST_FRAMEWORK` และ Part ที่จะเจาะลึก

| Key | หน้าที่ | Part ที่เจาะลึก |
|---|---|---|
| `DEFAULT_PERMISSION_CLASSES` | ควบคุมว่าใครเรียก API ได้บ้างเป็นค่าเริ่มต้นทั้งระบบ | **Part 045** |
| `DEFAULT_AUTHENTICATION_CLASSES` | วิธีที่ DRF ระบุตัวตนผู้ใช้จาก request (Session, Token, JWT) | **Part 045-046** |
| `DEFAULT_RENDERER_CLASSES` | รูปแบบ output ที่ `Response` แปลงข้อมูลออกไปได้ | **ขั้นตอนที่ 386 (Part นี้)** |
| `DEFAULT_PARSER_CLASSES` | รูปแบบ input ที่ `request.data` parse เข้ามาได้ | ขั้นตอนที่ 383 (Part นี้) |
| `DEFAULT_PAGINATION_CLASS` / `PAGE_SIZE` | แบ่งหน้าผลลัพธ์รายการยาว ๆ อัตโนมัติ | **Part 047** |
| `DEFAULT_THROTTLE_CLASSES` / `DEFAULT_THROTTLE_RATES` | จำกัดจำนวน request ต่อผู้ใช้/ต่อ IP ในช่วงเวลาหนึ่ง | **Part 048** |
| `DEFAULT_FILTER_BACKENDS` | เปิดใช้การค้นหา/กรอง/เรียงลำดับผ่าน query parameter | **Part 047** |
| `DEFAULT_SCHEMA_CLASS` / drf-spectacular | สร้างเอกสาร API (OpenAPI/Swagger) อัตโนมัติจากโค้ด | **Part 049** |
| `DEFAULT_VERSIONING_CLASS` | จัดการ API versioning (`/api/v1/`, `/api/v2/`) | **Part 048** |
| `EXCEPTION_HANDLER` | กำหนด error response format แบบ custom ทั้งระบบ | **Part 043** |
| `TEST_REQUEST_DEFAULT_FORMAT` | format เริ่มต้นตอนเขียน test เรียก API | **Part 050** |

### 385.4 ตั้งค่าเริ่มต้นแบบเรียบง่ายสำหรับ Part นี้

เนื่องจาก Part นี้ยังไม่ได้เรียนเรื่อง Permission/Authentication ของ DRF อย่างจริงจัง (รอ
Part 045) ให้ตั้งค่าเพียงเท่าที่จำเป็นสำหรับ endpoint `/api/ping/` ที่จะสร้างในขั้นตอนที่
387-390:

```python
# secureblog/settings.py

REST_FRAMEWORK = {
    'DEFAULT_RENDERER_CLASSES': [
        'rest_framework.renderers.JSONRenderer',
        'rest_framework.renderers.BrowsableAPIRenderer',
    ],
}
```

การไม่ตั้งค่า `DEFAULT_PERMISSION_CLASSES` เลย หมายความว่า DRF จะใช้ค่า default ของตัวเองคือ
`AllowAny` (ทุกคนเรียก API ได้โดยไม่ต้อง login) ซึ่งเหมาะสำหรับ endpoint สาธารณะอย่าง
`/api/ping/` ที่เราจะสร้าง แต่ **ไม่เหมาะกับ endpoint ที่จัดการข้อมูลจริงของ blog** เราจะ
กลับมาตั้งค่านี้อย่างรัดกุมใน Part 045

### 385.5 ตรวจสอบค่า settings ที่มีผลจริงผ่าน Python shell

```python
>>> from rest_framework.settings import api_settings
>>> api_settings.DEFAULT_RENDERER_CLASSES
[<class 'rest_framework.renderers.JSONRenderer'>,
 <class 'rest_framework.renderers.BrowsableAPIRenderer'>]

>>> api_settings.DEFAULT_PERMISSION_CLASSES
[<class 'rest_framework.permissions.AllowAny'>]

>>> api_settings.PAGE_SIZE
None
```

สังเกตว่า `api_settings` คืน **class จริง** (ไม่ใช่ string path) แม้ว่าใน `settings.py` เรา
เขียนเป็น string เช่น `'rest_framework.renderers.JSONRenderer'` — DRF ทำการ import string
นั้นให้กลายเป็น class จริงโดยอัตโนมัติเบื้องหลัง (คล้ายกับกลไกที่ Django ใช้ resolve
`MIDDLEWARE` list เป็น class จริงตอน server เริ่มทำงาน)

### 385.6 สรุปแนวคิดของขั้นตอนนี้

- DRF ทุก configuration รวมอยู่ใน dict เดียวชื่อ `REST_FRAMEWORK` ใน `settings.py`
- ไม่ตั้งค่าอะไรเลย = ใช้ค่า default ของ DRF เอง (ผ่าน `rest_framework.settings.api_settings`)
- key สำคัญที่จะเจาะลึกตลอด Phase 5: permission, authentication, renderer, parser,
  pagination, throttling, filtering, schema
- Part นี้ตั้งค่าเพียง `DEFAULT_RENDERER_CLASSES` เพื่อเตรียมพร้อมสำหรับ endpoint แรก

---

## ขั้นตอนที่ 386: Content Negotiation — JSONRenderer, BrowsableAPIRenderer และ renderer อื่น ๆ

### 386.1 Content Negotiation คืออะไรกันแน่ (นิยามให้ชัดเจน)

**Content Negotiation** คือกลไกที่ server เลือก **รูปแบบการนำเสนอข้อมูล (representation)**
ที่เหมาะสมที่สุดให้ client โดยอิงจาก header `Accept` ที่ client ส่งมาในแต่ละ request นี่คือ
คำว่า **"Representational"** ใน "**Representational** State Transfer" ที่อธิบายไว้ในขั้นตอน
ที่ 381 — REST ไม่ได้บังคับว่าทรัพยากรต้องแทนด้วย JSON เสมอไป ทรัพยากรเดียวกันสามารถมีได้
หลาย "representation" (JSON, XML, HTML, CSV ฯลฯ) และ client เลือกได้ว่าต้องการ
representation แบบไหนผ่าน header `Accept`

```
Client ส่ง:  Accept: application/json         → Server ตอบ JSON
Client ส่ง:  Accept: text/html                 → Server ตอบ HTML (Browsable API)
Client ส่ง:  Accept: application/xml           → Server ตอบ XML (ถ้าติดตั้ง XML renderer)
Client ไม่ส่ง Accept header เลย                → Server ใช้ renderer ตัวแรกใน DEFAULT_RENDERER_CLASSES
```

### 386.2 Renderer ที่มากับ DRF ในตัว (built-in renderers)

| Renderer class | Media type | หน้าที่ |
|---|---|---|
| `JSONRenderer` | `application/json` | แปลง Python data structure เป็น JSON string — renderer หลักที่ใช้เกือบทุก API |
| `BrowsableAPIRenderer` | `text/html` | render หน้า HTML แบบ interactive สำหรับทดสอบผ่านเบราว์เซอร์ (ขั้นตอนที่ 384) |
| `TemplateHTMLRenderer` | `text/html` | render ผ่าน Django template ปกติ (`.html`) เหมาะกับ hybrid app ที่ผสม HTML กับ DRF |
| `StaticHTMLRenderer` | `text/html` | คืน HTML string ตรง ๆ ที่ view เตรียมไว้เอง โดยไม่ผ่าน template engine |
| `MultiPartRenderer` | `multipart/form-data` | ใช้เป็น renderer ฝั่ง test client เท่านั้น ไม่ค่อยใช้ใน production view |

**ข้อสังเกต**: DRF **ไม่มี XML renderer มาให้ในตัว** ต้องติดตั้ง package แยก เช่น
`djangorestframework-xml` ถ้าต้องการรองรับ XML — สะท้อนว่า JSON คือ format หลักที่ DRF
(และอุตสาหกรรมปัจจุบัน) ให้ความสำคัญที่สุด

### 386.3 กำหนด renderer เฉพาะ view

Renderer กำหนดได้ทั้งระดับ project (ผ่าน `DEFAULT_RENDERER_CLASSES` ในขั้นตอนที่ 385) และ
ระดับ view เดี่ยว ๆ ผ่าน class attribute `renderer_classes`:

```python
from rest_framework.views import APIView
from rest_framework.response import Response
from rest_framework.renderers import JSONRenderer, BrowsableAPIRenderer


class PostStatsView(APIView):
    """Endpoint นี้เปิดทั้ง JSON และ Browsable API (ค่าเดียวกับ default ของโปรเจกต์)"""
    renderer_classes = [JSONRenderer, BrowsableAPIRenderer]

    def get(self, request):
        return Response({'total_posts': 42, 'total_published': 30})


class InternalMetricsView(APIView):
    """Endpoint สำหรับระบบภายในเรียกกันเอง (service-to-service) ปิด Browsable API"""
    renderer_classes = [JSONRenderer]

    def get(self, request):
        return Response({'cpu_usage': 0.42, 'memory_usage': 0.68})
```

**ลำดับใน list มีความหมาย**: renderer ตัวแรกใน `renderer_classes` (หรือ
`DEFAULT_RENDERER_CLASSES`) คือ renderer ที่ถูกใช้เมื่อ client **ไม่ได้ระบุ** `Accept`
header ที่ตรงกับตัวไหนเลย (fallback renderer)

### 386.4 กลไกภายใน: DRF เลือก renderer อย่างไรเมื่อมี request เข้ามา

```
1. Client ส่ง request มาพร้อม header Accept (เช่น "application/json" หรือ "text/html")
2. DRF ดู renderer_classes ของ view (หรือ DEFAULT_RENDERER_CLASSES ถ้า view ไม่ได้กำหนดเอง)
3. DRF จับคู่ Accept header กับ media_type ของแต่ละ renderer ใน list
   → เจอตัวที่ตรงที่สุด (exact match ก่อน, wildcard */* เป็นตัวสุดท้าย)
4. ถ้าไม่เจอตัวไหนตรงเลย และไม่มี */* → ตอบกลับด้วย HTTP 406 Not Acceptable
5. Renderer ที่เลือกได้จะถูกเก็บไว้ที่ request.accepted_renderer
6. เมื่อ view คืนค่า Response(data) กลับมา DRF จะเรียก
   request.accepted_renderer.render(data) เพื่อแปลงเป็น bytes จริงก่อนส่งออก
```

ทดลองดู `request.accepted_renderer` และ `request.accepted_media_type` ผ่านโค้ดจริง:

```python
class DebugRendererView(APIView):
    def get(self, request):
        return Response({
            'accepted_media_type': request.accepted_media_type,
            'accepted_renderer_class': request.accepted_renderer.__class__.__name__,
        })
```

```bash
curl -H "Accept: application/json" http://127.0.0.1:8000/api/debug-renderer/
# {"accepted_media_type": "application/json", "accepted_renderer_class": "JSONRenderer"}

curl -H "Accept: text/html" http://127.0.0.1:8000/api/debug-renderer/
# ได้หน้า Browsable API กลับมา (เพราะ Accept ตรงกับ BrowsableAPIRenderer)
```

### 386.5 URL format suffix: อีกวิธีหนึ่งในการระบุ format ที่ต้องการ

นอกจาก header `Accept` แล้ว DRF ยังรองรับการระบุ format ผ่าน **URL suffix** เช่น
`/api/ping.json` หรือ query parameter `?format=json` ซึ่งสะดวกมากเวลาทดสอบผ่านเบราว์เซอร์
โดยไม่ต้องตั้งค่า header เอง (เบราว์เซอร์กำหนด `Accept` header เองไม่ได้ง่าย ๆ ผ่าน address
bar) เปิดใช้งานผ่าน `format_suffix_patterns`:

```python
# blog/api_urls.py
from django.urls import path
from rest_framework.urlpatterns import format_suffix_patterns
from . import api_views

urlpatterns = [
    path('ping/', api_views.PingView.as_view(), name='api-ping'),
]

urlpatterns = format_suffix_patterns(urlpatterns)
```

เมื่อเปิดใช้ `format_suffix_patterns` แล้ว จะสามารถเข้าถึง endpoint ได้หลายรูปแบบ:

```
http://127.0.0.1:8000/api/ping/          → ใช้ Content Negotiation ปกติจาก Accept header
http://127.0.0.1:8000/api/ping.json      → บังคับ JSON เสมอ ไม่สนใจ Accept header
http://127.0.0.1:8000/api/ping/?format=json   → บังคับ JSON เสมอ ผ่าน query parameter
```

### 386.6 ตารางสรุป Content Negotiation

| แนวคิด | อธิบาย |
|---|---|
| `Accept` header | วิธีมาตรฐานของ HTTP ที่ client บอก server ว่าต้องการ representation แบบไหน |
| `renderer_classes` | list ของ renderer ที่ view นั้นรองรับ เรียงตามลำดับความสำคัญ |
| `request.accepted_renderer` | renderer ที่ DRF เลือกให้ใช้จริงสำหรับ request นี้ |
| `request.accepted_media_type` | media type string ที่ตรงกับ renderer ที่เลือก |
| `format_suffix_patterns` | เปิดให้ระบุ format ผ่าน URL suffix (`.json`) หรือ `?format=` แทน header |
| HTTP 406 Not Acceptable | เกิดเมื่อไม่มี renderer ตัวไหนรองรับ `Accept` ที่ client ขอมาเลย |

---

## ขั้นตอนที่ 387: เขียน `APIView` แรกแบบง่ายที่สุด คืนค่า JSON ตรง ๆ

### 387.1 `APIView` คืออะไร — เชื่อมโยงกับ Part 021

ทบทวนจาก Part 021: CBV ทุกตัวของ Django สืบทอดจาก `django.views.generic.base.View` ซึ่งมี
`as_view()`, `setup()`, และ `dispatch()` เป็นแกนหลัก **`rest_framework.views.APIView` คือ
subclass ของ `django.views.View` ตัวเดียวกันนี้เอง** — DRF ไม่ได้สร้างระบบ CBV ใหม่ทั้งหมด
แต่ **ต่อยอด** จากกลไกเดิมของ Django ที่คุณเข้าใจอย่างละเอียดมาแล้วจาก Part 021

```python
# แนวคิดจากซอร์สโค้ดจริงของ rest_framework.views.APIView (ย่อ)
from django.views.generic import View


class APIView(View):
    renderer_classes = api_settings.DEFAULT_RENDERER_CLASSES
    parser_classes = api_settings.DEFAULT_PARSER_CLASSES
    authentication_classes = api_settings.DEFAULT_AUTHENTICATION_CLASSES
    permission_classes = api_settings.DEFAULT_PERMISSION_CLASSES
    throttle_classes = api_settings.DEFAULT_THROTTLE_CLASSES

    @classmethod
    def as_view(cls, **initkwargs):
        view = super().as_view(**initkwargs)
        view.cls = cls
        view.csrf_exempt = True     # DRF จัดการ CSRF ด้วยกลไกของตัวเอง ไม่ใช่ของ Django
        return view

    def dispatch(self, request, *args, **kwargs):
        # (1) แปลง HttpRequest ของ Django ให้กลายเป็น DRF Request (ขั้นตอนที่ 383)
        self.request = self.initialize_request(request, *args, **kwargs)

        try:
            # (2) เรียก initial() เพื่อทำ content negotiation, authentication,
            #     permission check, throttle check ก่อนเรียก handler จริง
            self.initial(self.request, *args, **kwargs)

            # (3) หา handler ตาม HTTP method เหมือน View.dispatch() เดิมทุกประการ
            #     (getattr(self, request.method.lower(), self.http_method_not_allowed))
            if request.method.lower() in self.http_method_names:
                handler = getattr(self, request.method.lower(),
                                   self.http_method_not_allowed)
            else:
                handler = self.http_method_not_allowed
            response = handler(self.request, *args, **kwargs)

        except Exception as exc:
            # (4) DRF มี exception handler กลางที่แปลง Exception เป็น Response ที่เหมาะสม
            #     (เจาะลึกเต็มรูปแบบใน Part 043)
            response = self.handle_exception(exc)

        # (5) เตรียม Response ให้พร้อม render (เลือก renderer ที่เหมาะสม)
        self.response = self.finalize_response(request, response, *args, **kwargs)
        return self.response
```

สังเกตว่า **โครงสร้างพื้นฐานเหมือนกับ `View.dispatch()` ที่เรียนใน Part 021 ขั้นตอนที่ 203
ทุกประการ**: หา handler ตาม `request.method.lower()` ผ่าน `getattr()` เหมือนกันเป๊ะ ๆ สิ่งที่
`APIView` เพิ่มเข้ามาคือ **ขั้นตอนก่อนและหลัง** การเรียก handler: แปลง request เป็น DRF
`Request` ก่อน (`initialize_request`), เช็ค authentication/permission/throttle ก่อนเรียก
handler (`initial`), และแปลง response ให้พร้อม render หลังจากได้ผลลัพธ์แล้ว
(`finalize_response`)

### 387.2 ตารางเปรียบเทียบ `View` (Part 021) กับ `APIView` (Part นี้)

| กลไก | `django.views.View` (Part 021) | `rest_framework.views.APIView` |
|---|---|---|
| Base class | (ไม่มี, เป็นรากที่สุด) | สืบทอดจาก `django.views.View` โดยตรง |
| `as_view()` | คืนฟังก์ชัน `view()` | คืนฟังก์ชัน `view()` เหมือนกัน แต่เพิ่ม `csrf_exempt = True` |
| `dispatch()` | หา handler ตาม `request.method` แล้วเรียกทันที | เพิ่มขั้นตอน `initial()` (auth/permission/throttle check) ก่อนเรียก handler |
| `request` ที่ handler ได้รับ | `django.http.HttpRequest` | `rest_framework.request.Request` (ห่อ `HttpRequest` เดิม — ขั้นตอนที่ 383) |
| ค่าที่ handler ต้องคืน | `HttpResponse` หรือ subclass | `rest_framework.response.Response` (แนะนำ, แต่ `HttpResponse` ก็ยังใช้ได้) |
| Error handling | ไม่มีในตัว ต้องเขียน `try/except` เอง | มี `handle_exception()` กลาง แปลง Exception เป็น Response มาตรฐาน (Part 043) |
| Content Negotiation | ไม่มี | มีในตัวผ่าน `finalize_response()` (ขั้นตอนที่ 386) |

### 387.3 เขียน `APIView` แรก: `PingView`

มาเขียน `APIView` ที่ง่ายที่สุดเท่าที่จะเป็นไปได้ — endpoint ตรวจสอบว่า API ทำงานอยู่หรือไม่
(pattern มาตรฐานที่ทุกระบบ production ต้องมี เรียกว่า **health check endpoint**):

```python
# blog/api_views.py
import django
from rest_framework.views import APIView
from rest_framework.response import Response


class PingView(APIView):
    """
    Health-check endpoint ง่ายที่สุด — ใช้ตรวจสอบว่า API ทำงานปกติหรือไม่
    ไม่ต้องการ authentication ใด ๆ (สำหรับ Load Balancer/Monitoring system เรียกตรวจสอบ)
    """

    def get(self, request):
        return Response({
            'status': 'ok',
            'message': 'pong',
            'django_version': django.get_version(),
        })
```

เชื่อมเข้า `urls.py`:

```python
# blog/api_urls.py
from django.urls import path
from . import api_views

app_name = 'blog_api'

urlpatterns = [
    path('ping/', api_views.PingView.as_view(), name='ping'),
]
```

```python
# secureblog/urls.py
from django.contrib import admin
from django.urls import path, include

urlpatterns = [
    path('admin/', admin.site.urls),
    path('accounts/', include('accounts.urls')),
    path('api/', include('blog.api_urls')),
    path('api-auth/', include('rest_framework.urls')),
]
```

### 387.4 ทดสอบด้วย `runserver`

```bash
python manage.py runserver
```

**ทดสอบผ่านเบราว์เซอร์** (เห็น Browsable API ตามที่เรียนในขั้นตอนที่ 384):

```
http://127.0.0.1:8000/api/ping/
```

**ทดสอบผ่าน `curl`**:

```bash
curl http://127.0.0.1:8000/api/ping/
```

```json
{"status":"ok","message":"pong","django_version":"5.1.4"}
```

**ทดสอบผ่าน Python shell ด้วย `RequestFactory` (เทคนิคเดียวกับ Part 021 ขั้นตอนที่ 203.3)**:

```python
>>> from django.test import RequestFactory
>>> from blog.api_views import PingView
>>> factory = RequestFactory()

>>> request = factory.get('/api/ping/')
>>> view = PingView.as_view()
>>> response = view(request)
>>> response.status_code
200
>>> response.data
{'status': 'ok', 'message': 'pong', 'django_version': '5.1.4'}
```

สังเกตว่า `response.data` คือ attribute พิเศษของ DRF `Response` ที่เก็บ **ข้อมูลดิบก่อน
render** ไว้ (ต่างจาก `HttpResponse.content` ที่เป็น bytes ที่ render แล้ว) มีประโยชน์มากตอน
เขียน automated test ใน Part 050 เพราะเทียบค่าได้ตรง ๆ โดยไม่ต้อง `json.loads()` เอง

### 387.5 ทดสอบเมื่อเรียกด้วย method ที่ไม่รองรับ

```python
>>> request = factory.post('/api/ping/')
>>> response = view(request)
>>> response.status_code
405
>>> response.data
{'detail': ErrorDetail(string='Method "POST" not allowed.', code='method_not_allowed')}
```

สังเกตว่าพฤติกรรมนี้**เหมือนกับ `View` ธรรมดาทุกประการ** ตามที่อธิบายไว้ในขั้นตอนที่ 387.1-
387.2 (405 อัตโนมัติเมื่อไม่มี handler สำหรับ method นั้น) เพียงแต่ DRF ให้ error message มา
เป็น JSON ที่มีโครงสร้างชัดเจน (`{'detail': ...}`) แทนที่จะเป็น plain text ธรรมดาแบบ
`HttpResponseNotAllowed` ของ Django ดั้งเดิม

### 387.6 สรุปแนวคิดของขั้นตอนนี้

- `APIView` สืบทอดจาก `django.views.View` ตัวเดียวกับที่เรียนใน Part 021 ไม่ใช่ระบบใหม่แยก
- `dispatch()` ของ `APIView` เพิ่มขั้นตอน `initial()` (auth/permission/throttle) และ
  `finalize_response()` (content negotiation) ครอบ handler เดิม
- Handler (`get`, `post`, ...) ของ `APIView` รับ DRF `Request` และควรคืน DRF `Response`
- `PingView` คือตัวอย่าง health-check endpoint แบบ minimal ที่สุด: import `django`,
  return `Response({...})`
- Method ที่ไม่มี handler นิยามไว้ ยังคงได้ 405 อัตโนมัติเหมือน `View` ธรรมดา

---

## ขั้นตอนที่ 388: `rest_framework.status` module — ใช้ constant แทนเลข HTTP status ตรง ๆ

### 388.1 ปัญหาของการ hardcode ตัวเลข HTTP status

โค้ดที่เขียน `status=404`, `status=201`, `status=400` ตรง ๆ นั้น **ทำงานได้ถูกต้อง** แต่มี
ปัญหาด้าน **ความอ่านง่าย (readability)**: ผู้อ่านโค้ดที่ไม่ได้จำเลข HTTP status ทั้งหมดได้
ต้องเปิดหาความหมายของตัวเลขทุกครั้ง และเสี่ยงพิมพ์เลขผิด (เช่น พิมพ์ `403` ทั้งที่ตั้งใจจะ
เขียน `404`) โดยไม่มี IDE หรือ linter เตือนให้เลย เพราะทั้งคู่เป็นแค่ `int` ธรรมดา

### 388.2 `rest_framework.status`: module รวม constant ทั้งหมดของ HTTP status code

DRF มี module ชื่อ `rest_framework.status` ที่รวม **HTTP status code ทุกตัวตามมาตรฐาน RFC**
ไว้เป็น constant ชื่อที่สื่อความหมาย นำเข้าใช้แทนตัวเลขตรง ๆ ได้ทันที:

```python
from rest_framework import status

status.HTTP_200_OK                    # 200
status.HTTP_201_CREATED               # 201
status.HTTP_204_NO_CONTENT            # 204
status.HTTP_400_BAD_REQUEST           # 400
status.HTTP_401_UNAUTHORIZED          # 401
status.HTTP_403_FORBIDDEN             # 403
status.HTTP_404_NOT_FOUND             # 404
status.HTTP_405_METHOD_NOT_ALLOWED    # 405
status.HTTP_500_INTERNAL_SERVER_ERROR # 500
```

### 388.3 ตาราง HTTP status code ที่ใช้บ่อยที่สุดใน REST API

| Constant | ตัวเลข | หมวดหมู่ | เมื่อไหร่ควรใช้ |
|---|---|---|---|
| `status.HTTP_200_OK` | 200 | Success | `GET`/`PUT`/`PATCH` สำเร็จ และมีข้อมูลส่งกลับ |
| `status.HTTP_201_CREATED` | 201 | Success | `POST` สร้างทรัพยากรใหม่สำเร็จ |
| `status.HTTP_204_NO_CONTENT` | 204 | Success | `DELETE` สำเร็จ (ไม่มี body ส่งกลับ) |
| `status.HTTP_400_BAD_REQUEST` | 400 | Client Error | ข้อมูลที่ client ส่งมาไม่ผ่าน validation |
| `status.HTTP_401_UNAUTHORIZED` | 401 | Client Error | ยังไม่ได้ authenticate เลย (ไม่รู้ว่าเป็นใคร) |
| `status.HTTP_403_FORBIDDEN` | 403 | Client Error | authenticate แล้ว แต่ไม่มีสิทธิ์ทำสิ่งนี้ |
| `status.HTTP_404_NOT_FOUND` | 404 | Client Error | ไม่พบทรัพยากรที่ระบุ |
| `status.HTTP_405_METHOD_NOT_ALLOWED` | 405 | Client Error | เรียกด้วย HTTP method ที่ endpoint นี้ไม่รองรับ |
| `status.HTTP_429_TOO_MANY_REQUESTS` | 429 | Client Error | เกิน throttle limit (Part 048) |
| `status.HTTP_500_INTERNAL_SERVER_ERROR` | 500 | Server Error | เกิด exception ที่ไม่คาดคิดฝั่ง server |

**ข้อสังเกตเรื่อง 401 vs 403 ที่มือใหม่มักสับสน**: `401 Unauthorized` ที่จริงควรอ่านว่า
"**ยัง Unauthenticated**" (มาตรฐาน HTTP ตั้งชื่อคลาดเคลื่อนตั้งแต่ต้น) หมายถึง server **ไม่
รู้ว่า client เป็นใครเลย** ในขณะที่ `403 Forbidden` หมายถึง server **รู้แล้วว่า client เป็น
ใคร แต่คนนั้นไม่มีสิทธิ์** — เราจะกลับมาเจาะลึกความแตกต่างนี้อย่างละเอียดพร้อมโค้ดจริงใน
**Part 045** เมื่อเรียนเรื่อง Permission Classes ของ DRF

### 388.4 ใช้งานจริงใน `APIView`

```python
from rest_framework.views import APIView
from rest_framework.response import Response
from rest_framework import status


class PostCreateDemoView(APIView):
    """สาธิตการใช้ status constant แทนตัวเลข (ยังไม่มี Serializer จริง — เรียน Part 040)"""

    def post(self, request):
        title = request.data.get('title')

        if not title:
            return Response(
                {'error': 'ต้องระบุ title'},
                status=status.HTTP_400_BAD_REQUEST,
            )

        # (จำลองการสร้างสำเร็จ ในความเป็นจริงจะบันทึกลงฐานข้อมูลผ่าน Serializer)
        return Response(
            {'title': title, 'created': True},
            status=status.HTTP_201_CREATED,
        )

    def delete(self, request, pk):
        # (จำลองการลบสำเร็จ)
        return Response(status=status.HTTP_204_NO_CONTENT)
```

### 388.5 เปรียบเทียบโค้ดก่อน-หลังใช้ `status` module

```python
# ❌ ก่อน: hardcode ตัวเลข อ่านยาก เสี่ยงพิมพ์ผิด
def create_post(request):
    if not request.data.get('title'):
        return Response({'error': 'ต้องระบุ title'}, status=400)
    return Response({'created': True}, status=201)


# ✅ หลัง: ใช้ constant สื่อความหมายชัดเจน, IDE autocomplete ช่วยได้, พิมพ์ผิดจะ error ทันที
from rest_framework import status

def create_post(request):
    if not request.data.get('title'):
        return Response({'error': 'ต้องระบุ title'}, status=status.HTTP_400_BAD_REQUEST)
    return Response({'created': True}, status=status.HTTP_201_CREATED)
```

ข้อดีเพิ่มเติมของการใช้ constant: ถ้าพิมพ์ชื่อ constant ผิด (เช่น `status.HTTP_40_BAD_REQUST`)
Python จะ raise `AttributeError` ทันทีตอนรันโค้ด ในขณะที่พิมพ์ตัวเลขผิด (`status=402` แทนที่
จะเป็น `400`) โปรแกรมจะรันผ่านไปเรียบร้อยโดยไม่มี error ใด ๆ เตือน แต่ client จะได้รับ status
code ที่ผิดความหมายไปเงียบ ๆ ซึ่ง debug ยากกว่ามาก

### 388.6 ตารางหมวดหมู่ HTTP status code ทั้งหมด (ภาพรวมมาตรฐาน)

| ช่วงตัวเลข | หมวดหมู่ | ความหมาย |
|---|---|---|
| 1xx | Informational | ระหว่างดำเนินการ (พบน้อยมากในการเขียน API ทั่วไป) |
| 2xx | Success | คำขอสำเร็จ |
| 3xx | Redirection | ต้องไปที่ URL อื่นต่อ |
| 4xx | Client Error | client ทำผิดพลาด (ส่งข้อมูลผิด, ไม่มีสิทธิ์, ไม่มีทรัพยากร) |
| 5xx | Server Error | server เกิดข้อผิดพลาดที่ไม่ใช่ความผิดของ client |

`rest_framework.status` มี constant ครบทุกช่วง รวมถึงมี helper function ที่มีประโยชน์มาก:

```python
from rest_framework import status

status.is_success(200)          # True  (2xx)
status.is_client_error(404)     # True  (4xx)
status.is_server_error(500)     # True  (5xx)
status.is_redirect(301)         # True  (3xx)
```

### 388.7 สรุปแนวคิดของขั้นตอนนี้

- `rest_framework.status` รวม HTTP status code มาตรฐานทุกตัวเป็น constant ที่สื่อความหมาย
- ใช้ `status.HTTP_XXX_NAME` แทนการ hardcode ตัวเลขเสมอ เพื่อความอ่านง่ายและป้องกันการพิมพ์ผิด
- 401 = "ยังไม่รู้ว่าเป็นใคร" (Unauthenticated), 403 = "รู้แล้วแต่ไม่มีสิทธิ์" (Forbidden)
- มี helper function `is_success()`, `is_client_error()`, `is_server_error()` ให้ใช้ตรวจสอบ
  หมวดหมู่ของ status code แบบ dynamic ได้ด้วย

---

## ขั้นตอนที่ 389: เปรียบเทียบ DRF กับ `JsonResponse` ธรรมดาของ Django เอง

### 389.1 ทบทวน: Django เปล่า ๆ ก็สร้าง JSON API ได้

สิ่งสำคัญที่ต้องเข้าใจให้ชัดเจนคือ **DRF ไม่ใช่สิ่งเดียวที่ทำให้ Django ส่ง JSON ได้** ทบทวน
จาก Part 007: `django.http.JsonResponse` มีมาให้ใน Django core อยู่แล้วตั้งแต่ต้น สร้าง JSON
API แบบง่าย ๆ ได้โดยไม่ต้องติดตั้งอะไรเพิ่มเลย:

```python
# blog/views.py — API endpoint แบบ Django ธรรมดาล้วน ๆ ไม่มี DRF เกี่ยวข้องเลย
import json
from django.http import JsonResponse, HttpResponseBadRequest
from django.views import View
from django.views.decorators.csrf import csrf_exempt
from django.utils.decorators import method_decorator
from .models import Post


@method_decorator(csrf_exempt, name='dispatch')
class PlainPostListView(View):
    def get(self, request):
        posts = Post.objects.filter(is_published=True).values('id', 'title', 'slug')
        return JsonResponse(list(posts), safe=False)

    def post(self, request):
        try:
            data = json.loads(request.body)
        except json.JSONDecodeError:
            return HttpResponseBadRequest(
                json.dumps({'error': 'invalid JSON'}),
                content_type='application/json',
            )

        title = data.get('title')
        if not title:
            return JsonResponse({'error': 'ต้องระบุ title'}, status=400)

        post = Post.objects.create(title=title, slug=title.lower().replace(' ', '-'))
        return JsonResponse({'id': post.id, 'title': post.title}, status=201)
```

เทียบกับเวอร์ชัน DRF (ยังไม่ใช้ Serializer เต็มรูปแบบ เพื่อให้เทียบกันตรง ๆ — Serializer จริง
จะเรียนใน Part 040):

```python
# blog/api_views.py — API endpoint แบบ DRF
from rest_framework.views import APIView
from rest_framework.response import Response
from rest_framework import status
from .models import Post


class PostListAPIView(APIView):
    def get(self, request):
        posts = Post.objects.filter(is_published=True).values('id', 'title', 'slug')
        return Response(list(posts))

    def post(self, request):
        title = request.data.get('title')
        if not title:
            return Response(
                {'error': 'ต้องระบุ title'},
                status=status.HTTP_400_BAD_REQUEST,
            )

        post = Post.objects.create(title=title, slug=title.lower().replace(' ', '-'))
        return Response({'id': post.id, 'title': post.title}, status=status.HTTP_201_CREATED)
```

### 389.2 ตารางเปรียบเทียบโดยละเอียด

| ประเด็น | `JsonResponse` (Django เปล่า) | DRF (`APIView` + `Response`) |
|---|---|---|
| Parse JSON จาก body | ต้อง `json.loads(request.body)` เอง พร้อม `try/except` เอง | `request.data` parse ให้อัตโนมัติ ไม่ว่า Content-Type จะเป็น JSON/form/multipart |
| CSRF สำหรับ POST | ต้อง `@csrf_exempt` เอง (หรือจัดการ CSRF token เองถ้าไม่ exempt) | DRF จัดการ CSRF ตามกลไก authentication ของตัวเอง (เจาะลึก Part 045) |
| Content Negotiation | ไม่มี ต้อง hardcode format เดียว | มีในตัว เลือก JSON/HTML/format อื่นตาม `Accept` header อัตโนมัติ |
| Browsable API สำหรับทดสอบ | ❌ ไม่มี ต้องพึ่ง Postman/curl เท่านั้น | ✅ มีในตัวทันทีไม่ต้องตั้งค่าเพิ่ม (ขั้นตอนที่ 384) |
| Validation ข้อมูล input | เขียน `if`/`try-except` ตรวจสอบเองทุกฟิลด์ | มี Serializer จัดการ validation อย่างเป็นระบบ (Part 040) |
| แปลง Model → JSON ที่ซับซ้อน (nested relations) | เขียน dict/list comprehension เองทั้งหมด | Serializer + nested serializer จัดการให้ (Part 041) |
| Authentication หลายแบบพร้อมกัน (Session, Token, JWT) | ต้องเขียน middleware/decorator เองทั้งหมด | มี `authentication_classes` สลับ/ผสมได้ทันที (Part 045-046) |
| Pagination อัตโนมัติ | เขียน slice queryset + คำนวณ next/prev URL เอง | มี `DEFAULT_PAGINATION_CLASS` ให้ใช้ทันที (Part 047) |
| Throttling (rate limit) | ไม่มีในตัว ต้องเขียนเองหรือหา package แยก | มี `throttle_classes` ให้ใช้ทันที (Part 048) |
| API Documentation อัตโนมัติ | ไม่มี ต้องเขียนเอกสารแยกมือ | สร้างได้อัตโนมัติจาก Serializer/View ผ่าน `drf-spectacular` (Part 049) |
| ขนาดโค้ดสำหรับ API เล็ก ๆ (เช่น health check) | สั้นกว่าเล็กน้อย ไม่ต้อง import อะไรเพิ่ม | เพิ่ม dependency และโค้ดเล็กน้อยกว่า แต่แลกกับ consistency ทั้งระบบ |
| Learning curve | ต่ำมาก (ใช้ความรู้ Django ปกติที่มีอยู่แล้ว) | ต้องเรียนรู้แนวคิดเพิ่ม (Serializer, ViewSet, Permission class ของ DRF เอง) |

### 389.3 เมื่อไหร่ควรใช้ `JsonResponse` ธรรมดา เมื่อไหร่ควรใช้ DRF

| สถานการณ์ | คำแนะนำ | เหตุผล |
|---|---|---|
| Endpoint เดี่ยว ๆ ง่าย ๆ เช่น health check, webhook receiver | `JsonResponse` พอเพียง | ไม่คุ้มที่จะดึง DRF ทั้ง framework มาใช้กับ endpoint เดียว |
| ระบบ API ที่มีหลาย endpoint, ต้อง CRUD ข้อมูลจริง | **DRF** | ลดโค้ดซ้ำซ้อนมหาศาลเมื่อมี endpoint จำนวนมาก (Serializer, ViewSet, Router) |
| ต้องการ Browsable API ให้ทีมอื่นทดสอบง่าย | **DRF** | เป็นฟีเจอร์ที่ DRF มีให้ฟรีทันที |
| ต้องรองรับ authentication หลายแบบ (JWT, Token, Session) | **DRF** | ระบบ `authentication_classes`/`permission_classes` ออกแบบมาสำหรับสถานการณ์นี้โดยเฉพาะ |
| Public webhook ที่รับ payload จาก third-party (เช่น Stripe, GitHub) แบบ format ตายตัว | `JsonResponse` หรือ `HttpResponse` เปล่า ๆ มักเพียงพอ | Payload มาจากภายนอกตามสเปกตายตัวอยู่แล้ว ไม่ต้องการ content negotiation |
| โปรเจกต์เล็กมาก, ทีมคนเดียว, ไม่มีแผนขยาย API | เริ่มจาก `JsonResponse` ก็ได้ ค่อยย้ายมา DRF ทีหลังถ้าจำเป็น | หลีกเลี่ยงความซับซ้อนที่ไม่จำเป็นในช่วงแรก (YAGNI principle) |

**ข้อสรุปเชิงปฏิบัติสำหรับหลักสูตรนี้**: เนื่องจากโปรเจกต์ `secureblog` ของเราจะมี endpoint
จำนวนมากตลอด Phase 5 (Post, Category, Tag, Comment ทั้งหมดต้องมี API) และต้องผสานกับระบบ
Authentication/Permission ที่ซับซ้อนจาก Phase 4 (Part 031-038) การใช้ **DRF เต็มรูปแบบคือ
ตัวเลือกที่เหมาะสมที่สุด** — แต่ความรู้เรื่อง `JsonResponse` จาก Part 007 จะยังมีประโยชน์เสมอ
สำหรับ endpoint พิเศษที่ไม่เข้ากับรูปแบบมาตรฐานของ DRF (เช่น webhook receiver ในอนาคต)

### 389.4 สรุปแนวคิดของขั้นตอนนี้

- `JsonResponse` ของ Django เปล่า ๆ สร้าง JSON API ได้จริง ไม่จำเป็นต้องมี DRF เสมอไป
- DRF ให้ประโยชน์มากที่สุดเมื่อระบบมี endpoint จำนวนมากที่ต้องการความสม่ำเสมอ (validation,
  authentication, pagination, documentation) ร่วมกัน
- สำหรับ endpoint เดี่ยว ๆ ง่าย ๆ หรือ webhook เฉพาะทาง `JsonResponse` ยังคงเป็นตัวเลือกที่ดี
- โปรเจกต์ `secureblog` ของหลักสูตรนี้เลือกใช้ DRF เต็มรูปแบบตลอด Phase 5 เพราะขนาดและความ
  ซับซ้อนของระบบ API ที่จะสร้าง

---

## ขั้นตอนที่ 390: สรุปและแบบฝึกหัด — สร้าง endpoint แรก `/api/ping/`

### 390.1 สร้าง endpoint `/api/ping/` แบบสมบูรณ์: รวมทุกอย่างที่เรียนใน Part นี้

มาประกอบทุกแนวคิดของ Part นี้เข้าด้วยกัน — ปรับปรุง `PingView` จากขั้นตอนที่ 387 ให้เป็น
health-check endpoint แบบมืออาชีพที่ตรวจสอบสถานะระบบจริง ไม่ใช่แค่คืนข้อความ `"pong"` เฉย ๆ

```python
# blog/api_views.py
import django
from django.db import connection
from django.utils import timezone
from rest_framework.views import APIView
from rest_framework.response import Response
from rest_framework import status


class PingView(APIView):
    """
    Health-check endpoint สำหรับตรวจสอบสถานะระบบโดยรวม
    ใช้โดย Load Balancer, Monitoring system (เช่น UptimeRobot, Kubernetes liveness probe)
    เพื่อตรวจสอบว่าระบบยังทำงานปกติอยู่หรือไม่ ไม่ต้องการ authentication เพราะระบบ
    ตรวจสอบภายนอกส่วนใหญ่ไม่มีบัญชีผู้ใช้ในระบบ
    """

    def get(self, request):
        db_status = self._check_database()

        overall_status = 'ok' if db_status == 'ok' else 'degraded'
        http_status = (
            status.HTTP_200_OK
            if overall_status == 'ok'
            else status.HTTP_503_SERVICE_UNAVAILABLE
        )

        return Response(
            {
                'status': overall_status,
                'message': 'pong',
                'timestamp': timezone.now().isoformat(),
                'django_version': django.get_version(),
                'checks': {
                    'database': db_status,
                },
            },
            status=http_status,
        )

    def _check_database(self):
        """ตรวจสอบว่าเชื่อมต่อฐานข้อมูลได้จริงหรือไม่ ด้วยการ query ที่เบาที่สุด"""
        try:
            with connection.cursor() as cursor:
                cursor.execute('SELECT 1')
            return 'ok'
        except Exception:
            return 'error'
```

ผลลัพธ์เมื่อเรียกผ่าน `curl`:

```bash
curl -i http://127.0.0.1:8000/api/ping/
```

```
HTTP/1.1 200 OK
Content-Type: application/json
Vary: Accept
Allow: GET, HEAD, OPTIONS

{
    "status": "ok",
    "message": "pong",
    "timestamp": "2026-09-26T10:30:00.123456+00:00",
    "django_version": "5.1.4",
    "checks": {
        "database": "ok"
    }
}
```

### 390.2 สรุปสิ่งที่ได้เรียนรู้ใน Part นี้

- ✅ เข้าใจหลักการ REST 6 ข้อ โดยเฉพาะ Statelessness และ Uniform Interface
  (Resource-based URL + HTTP verb มีความหมายมาตรฐาน)
- ✅ เข้าใจเหตุผลเชิงสถาปัตยกรรมว่าทำไมต้องมี API แยกจากหน้าเว็บ HTML: mobile app, SPA
  frontend, third-party integration, microservices
- ✅ ติดตั้ง Django REST Framework และเพิ่มเข้า `INSTALLED_APPS` สำเร็จ
- ✅ เข้าใจว่า DRF `Request` ห่อ (ไม่ใช่แทนที่) `HttpRequest` เดิม และ `request.data`/
  `request.query_params` ทำงานอย่างไร
- ✅ เข้าใจว่า DRF `Response` ใช้ Content Negotiation เลือก format output อัตโนมัติแทนการ
  hardcode เป็น JSON เสมอ
- ✅ เข้าใจ Browsable API และกลไก Content Negotiation ที่อยู่เบื้องหลัง
- ✅ เห็นภาพรวมของ `REST_FRAMEWORK` settings dict และ key สำคัญที่จะเจาะลึกตลอด Phase 5
- ✅ เข้าใจ renderer ต่าง ๆ ของ DRF (`JSONRenderer`, `BrowsableAPIRenderer`) และวิธีกำหนด
  เฉพาะ view
- ✅ เข้าใจว่า `APIView` สืบทอดจาก `django.views.View` ที่เรียนใน Part 021 และเขียน
  `APIView` แรกได้เอง
- ✅ ใช้ `rest_framework.status` แทนตัวเลข HTTP status ตรง ๆ ได้อย่างถูกต้อง
- ✅ เข้าใจข้อดี-ข้อเสียของ DRF เทียบกับ `JsonResponse` ธรรมดา และรู้ว่าเมื่อไหร่ควรเลือกแบบไหน
- ✅ สร้าง health-check endpoint `/api/ping/` แบบสมบูรณ์ที่ตรวจสอบสถานะฐานข้อมูลจริง

### 390.3 Checklist ก่อนไป Part ถัดไป

- [ ] อธิบายหลักการ REST 6 ข้อได้ด้วยคำพูดของตัวเอง โดยเฉพาะ Statelessness
- [ ] อธิบายได้ว่าทำไม URL แบบ `/api/posts/5/` ถึงดีกว่า `/getPostById?id=5`
- [ ] รัน `pip install djangorestframework` และเพิ่ม `'rest_framework'` ใน
  `INSTALLED_APPS` สำเร็จ
- [ ] อธิบายความแตกต่างระหว่าง `request.data` กับ `request.POST` ได้
- [ ] เปิด endpoint ผ่านเบราว์เซอร์แล้วเห็นหน้า Browsable API จริง
- [ ] อธิบายได้ว่า `APIView` สืบทอดมาจาก class ไหนของ Django และมี `dispatch()` ต่างจาก
  `View` ธรรมดาอย่างไร
- [ ] เขียน `APIView` ที่มี `get()` คืน `Response` สำเร็จอย่างน้อย 1 ตัว
- [ ] ใช้ `rest_framework.status` แทนตัวเลข status ได้ในโค้ดของตัวเอง
- [ ] สร้าง endpoint `/api/ping/` ที่ตรวจสอบสถานะฐานข้อมูลจริงตามขั้นตอนที่ 390.1 สำเร็จ
- [ ] ทดสอบ `/api/ping/` ทั้งผ่านเบราว์เซอร์และ `curl` แล้วได้ผลลัพธ์ตรงกับที่คาดหวัง

### 390.4 แบบฝึกหัดท้ายบท

**แบบฝึกหัดที่ 1 (พื้นฐาน)**: เขียน `APIView` ชื่อ `ServerTimeView` ที่มี `get()` คืนค่า
เวลาปัจจุบันของ server ในรูปแบบ JSON `{"server_time": "...", "timezone": "..."}` โดยใช้
`django.utils.timezone.now()` และ `settings.TIME_ZONE` แล้วเชื่อมเข้า `blog/api_urls.py` ที่
path `/api/server-time/` ทดสอบผ่านทั้งเบราว์เซอร์ (ดู Browsable API) และ `curl`

**แบบฝึกหัดที่ 2 (ประยุกต์)**: เขียน `APIView` ชื่อ `EchoHeadersView` ที่มี `get()` อ่านค่า
จาก `request.META` แล้วคืนค่า User-Agent และ IP address ของผู้เรียก (`REMOTE_ADDR`) กลับไป
เป็น JSON จากนั้นทดลองเรียกด้วย `curl -H "User-Agent: MyTestClient/1.0"` แล้วยืนยันว่าค่าที่
ได้กลับมาตรงกับที่ส่งไป (ทบทวนการอ่าน `request.META` จาก Part 007)

**แบบฝึกหัดที่ 3 (Content Negotiation)**: เขียน `APIView` ชื่อ `RestrictedPingView` ที่
ทำงานเหมือน `PingView` ทุกประการ แต่กำหนด `renderer_classes = [JSONRenderer]` เท่านั้น (ไม่มี
`BrowsableAPIRenderer`) แล้วพิสูจน์ด้วยการเปิดผ่านเบราว์เซอร์ว่าได้ JSON ดิบแทนหน้า HTML สวย
งาม จากนั้นอธิบายในไฟล์ `notes.md` ว่าทำไมถึงเป็นเช่นนั้น โดยอ้างอิงกลไก Content Negotiation
จากขั้นตอนที่ 386

**แบบฝึกหัดที่ 4 (ขั้นสูง)**: ขยาย `PingView` จากขั้นตอนที่ 390.1 ให้ตรวจสอบเพิ่มอีก 1 อย่าง
คือจำนวน record ทั้งหมดในตาราง `Post` (`Post.objects.count()`) แล้วเพิ่มเข้า `checks` dict
เป็น `{"database": "ok", "post_count_reachable": true}` โดยจับ Exception ถ้า query ล้มเหลว
ให้ `overall_status` เปลี่ยนเป็น `"degraded"` และคืน `status.HTTP_503_SERVICE_UNAVAILABLE`
เหมือนกับตรรกะเดิมของ `_check_database()` (จำลองสถานการณ์นี้ได้โดยลองเปลี่ยนชื่อ field ผิด
ชั่วคราวเพื่อบังคับให้เกิด Exception แล้วดูว่า endpoint ตอบสนองถูกต้องหรือไม่)

### 390.5 คำถามที่พบบ่อย (FAQ)

**Q: จำเป็นต้องรู้ Generic CBV จาก Part 021-024 ก่อนเรียน DRF หรือไม่?**
A: จำเป็นมาก โดยเฉพาะกลไก `as_view()`, `dispatch()`, และแนวคิด mixin จาก Part 021 และ
Part 024 เพราะ `APIView` และ Generic API View ของ DRF (ที่จะเรียนใน Part 043) ต่อยอดจาก
แนวคิดเดียวกันทั้งหมด ถ้ายังไม่แม่นเรื่อง `dispatch()` และ mixin แนะนำให้กลับไปทบทวน
Part 021 ก่อนไปต่อ Part 040

**Q: DRF เป็น package ที่ Django ทีมงานหลักดูแลเองหรือเป็น third-party?**
A: เป็น **third-party package** ที่ไม่ได้อยู่ใน Django core แต่ได้รับการยอมรับอย่างกว้างขวาง
จนกลายเป็นมาตรฐานโดยพฤตินัยของวงการ Django เอกสารทางการของ Django เองก็แนะนำ DRF เมื่อพูด
ถึงการสร้าง API และบริษัทระดับโลกจำนวนมาก (รวมถึง Instagram ในบางส่วน) ใช้ DRF ในระบบจริง

**Q: ทำไม endpoint `/api/ping/` ในขั้นตอนที่ 390.1 ไม่ต้องมี authentication เลย ในเมื่อ
Phase 4 ทั้ง Phase สอนเรื่อง security?**
A: Health-check endpoint เป็นกรณีพิเศษที่**จงใจ**เปิดสาธารณะ เพราะระบบภายนอกที่เรียก (Load
Balancer, Kubernetes, Monitoring service) มักไม่มีบัญชีผู้ใช้ในระบบของเราเลย และต้องเรียก
บ่อยมาก (ทุกไม่กี่วินาที) การบังคับ authentication จะทำให้ระบบ monitoring ทำงานไม่ได้ อย่างไร
ก็ตามควรระวังไม่ให้ endpoint นี้เปิดเผยข้อมูลอ่อนไหว (เช่น ไม่ควรคืน stack trace เต็มรูปแบบ
เมื่อเกิด error) — เราจะเรียนเรื่องการกำหนด permission แบบละเอียดต่าง endpoint ใน Part 045

**Q: ควรใช้ `Response()` หรือ `JsonResponse()` ภายใน `APIView` ดี ในเมื่อทั้งคู่ใช้ได้จริง?**
A: ควรใช้ `Response()` เสมอภายใน `APIView`/`ViewSet` เพราะเป็นสิ่งเดียวที่ทำงานร่วมกับ
Content Negotiation, Browsable API, และ Exception Handler ของ DRF ได้อย่างสมบูรณ์
`JsonResponse` ยังคง "ทำงานได้" ทางเทคนิคถ้าคุณ return มันจาก handler ของ `APIView`
(เพราะมันก็เป็น `HttpResponse` subclass เหมือนกัน) แต่จะไม่ได้ประโยชน์จาก renderer ของ DRF
เลย (Browsable API จะไม่แสดงผล, format จะ hardcode เป็น JSON เสมอ) — ในทางปฏิบัติแทบไม่มี
เหตุผลที่ดีจะทำแบบนั้น

### เตรียมตัวสำหรับ Part ถัดไป

**Part 040: Serializers เบื้องต้น** จะแก้ปัญหาที่คุณเจอไปแล้วในขั้นตอนที่ 389.1 ของ Part นี้:
การแปลง Model instance เป็น JSON ด้วยมือ (เขียน `.values()` หรือ dict comprehension เอง) ทำ
ได้สำหรับกรณีง่าย ๆ แต่จะยุ่งยากมากขึ้นเรื่อย ๆ เมื่อต้อง validate ข้อมูล input, จัดการ
relationship ระหว่าง Model (เช่น `Post` กับ `Category`/`Tag`/`Comment`), หรือแปลงกลับจาก
JSON เป็น Model instance ตอนสร้าง/แก้ไขข้อมูล

Part 040 จะแนะนำ **`Serializer`** class ของ DRF ซึ่งทำหน้าที่คล้ายกับ `Form`/`ModelForm` ที่
คุณเรียนไปแล้วใน Part 025-029 แต่ออกแบบมาสำหรับแปลงข้อมูลไป-กลับระหว่าง Python object กับ
JSON แทนที่จะเป็น HTML form คุณจะได้เขียน `PostSerializer` ตัวแรก เข้าใจ field types ของ
Serializer, `validate_<field>()`, `is_valid()`, และ `.save()` ซึ่งจะกลายเป็นรากฐานสำคัญที่สุด
ของ DRF ตลอด Phase 5 ที่เหลือ

เตรียมเปิด `blog/models.py` (ทบทวน field ของ `Post`, `Category`, `Tag`, `Comment`) และ
`blog/api_views.py` ที่สร้างไว้ใน Part นี้ให้พร้อม เพราะ Part 040 จะกลับมาเขียน
`PostListAPIView` ใหม่ทั้งหมดด้วย Serializer แทนโค้ด manual ที่เห็นในขั้นตอนที่ 389.1!
