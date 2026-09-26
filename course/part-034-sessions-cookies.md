# Part 034: Django Sessions และ Cookies

> **ขั้นตอนที่ 331-340 ของหลักสูตร** | Phase 4: Authentication, Users และ Permissions
>
> เป้าหมายของ Part นี้: เข้าใจกลไกเบื้องหลังของ **Django Session Framework** อย่างถ่องแท้
> ตั้งแต่การทำงานของ `SessionMiddleware` และ cookie `sessionid`, Session Backend ทั้ง 4
> แบบที่ Django รองรับ, การใช้งาน `request.session` ในโค้ดจริง, การตั้งค่าอายุและความ
> ปลอดภัยของ cookie อย่างมืออาชีพ, ไปจนถึงการนำ Session มาสร้างระบบ "ตะกร้าสินค้า" สำหรับ
> ผู้ใช้ที่ยังไม่ login และการป้องกัน Session Hijacking ด้วย `cycle_key()` เมื่อจบ Part นี้
> คุณจะเข้าใจว่าทำไม Django ถึง "จำ" คุณได้ทุกครั้งที่กดรีเฟรชหน้าเว็บ ทั้งที่ HTTP เป็น
> โปรโตคอลที่ไม่มีสถานะ (stateless) โดยธรรมชาติ

---

## สารบัญของ Part นี้

- ขั้นตอนที่ 331: กลไกการทำงานของ Django Session — `SessionMiddleware`, cookie `sessionid` ทำงานอย่างไรเบื้องหลัง
- ขั้นตอนที่ 332: Session Backend ที่ Django รองรับ — database (default), cache, cached_db, signed cookies พร้อมตารางเปรียบเทียบข้อดีข้อเสีย
- ขั้นตอนที่ 333: การใช้งาน `request.session` — `.get()`, `[]=`, `.pop()`, `.flush()`, `.set_expiry()`
- ขั้นตอนที่ 334: ตั้งค่าอายุ Session — `SESSION_COOKIE_AGE`, `SESSION_EXPIRE_AT_BROWSER_CLOSE`, `SESSION_SAVE_EVERY_REQUEST`
- ขั้นตอนที่ 335: ความปลอดภัยของ Cookie — `SESSION_COOKIE_SECURE`, `SESSION_COOKIE_HTTPONLY`, `SESSION_COOKIE_SAMESITE`
- ขั้นตอนที่ 336: ตัวอย่างจริง — ใช้ Session ทำระบบ "ตะกร้าสินค้า" สำหรับผู้ใช้ที่ยังไม่ login
- ขั้นตอนที่ 337: ทบทวนความสัมพันธ์ระหว่าง Session กับ CSRF Cookie (เชื่อมกับ Part 008)
- ขั้นตอนที่ 338: ความเสี่ยง Session Hijacking และแนวทางป้องกันด้วย `cycle_key()`
- ขั้นตอนที่ 339: คำสั่ง `clearsessions` สำหรับล้าง session ที่หมดอายุ และการตั้ง cron job
- ขั้นตอนที่ 340: สรุปและแบบฝึกหัด — ตะกร้าสินค้าแบบ session-based พร้อม regenerate session ID ตอน login

---

## ขั้นตอนที่ 331: กลไกการทำงานของ Django Session

### 331.1 ปัญหาที่ Session แก้: HTTP เป็นโปรโตคอลที่ไม่มีสถานะ (Stateless)

HTTP ถูกออกแบบให้แต่ละ request **ไม่เกี่ยวข้องกัน** เซิร์ฟเวอร์ไม่มีทาง "จำ" ได้เองว่า
request ที่เพิ่งเข้ามาเป็นคนเดิมกับ request ก่อนหน้าหรือไม่ ปัญหานี้ชัดเจนมากในเว็บ
แอปพลิเคชันจริง เช่น:

- ผู้ใช้ login สำเร็จในหน้าแรก แต่พอกดลิงก์ไปหน้าอื่น เซิร์ฟเวอร์ต้อง "รู้" ว่านี่คือคนที่
  เพิ่ง login ไปแล้ว ไม่ใช่ผู้ใช้นิรนาม
- ผู้ใช้เพิ่มสินค้าลงตะกร้า แล้วไปดูสินค้าชิ้นอื่นต่อ ตะกร้าต้องไม่หายไป

**Session** คือกลไกที่แก้ปัญหานี้ โดยให้เซิร์ฟเวอร์เก็บข้อมูลของผู้ใช้แต่ละคนไว้ฝั่งตัวเอง
(server-side) แล้วมอบ "กุญแจ" สั้น ๆ หนึ่งดอกให้เบราว์เซอร์เก็บไว้ผ่าน **cookie** เพื่อใช้
อ้างอิงกลับมาหาข้อมูลชุดนั้นในทุก request ถัดไป

### 331.2 `SessionMiddleware`: หัวใจของระบบ Session ทั้งหมด

Django ใช้ **Middleware** (จะเรียนเจาะลึกเต็มรูปแบบใน Phase 5) เป็นกลไกที่แทรกโค้ดเข้าไป
ทำงานก่อนและหลัง view ทุกครั้ง `SessionMiddleware` คือ middleware ที่รับผิดชอบระบบ
Session ทั้งหมด และถูกเปิดใช้งานมาให้อัตโนมัติแล้วตั้งแต่ `startproject` (ทบทวนจาก
Part 004):

```python
# config/settings.py
MIDDLEWARE = [
    'django.middleware.security.SecurityMiddleware',
    'django.contrib.sessions.middleware.SessionMiddleware',   # ← ตัวนี้
    'django.middleware.common.CommonMiddleware',
    'django.middleware.csrf.CsrfViewMiddleware',
    'django.contrib.auth.middleware.AuthenticationMiddleware',
    'django.contrib.messages.middleware.MessageMiddleware',
    'django.middleware.clickjacking.XFrameOptionsMiddleware',
]
```

> **ทำไมต้องอยู่เกือบบนสุดของลิสต์**: Middleware ทำงานตามลำดับ **บนลงล่างตอนขาเข้า
> (request)** และ **ล่างขึ้นบนตอนขาออก (response)** — กฎทองข้อเดิมเรื่อง "ลำดับสำคัญ
> เสมอ" ที่เจอมาตั้งแต่ Part 006 (URL patterns) และ Part 008 (Template search order)
> `SessionMiddleware` ต้องทำงาน**ก่อน** `AuthenticationMiddleware` เสมอ เพราะระบบ
> Authentication (Part 031) ต้องใช้ session ในการเก็บว่า "ผู้ใช้คนไหน login อยู่"

`SessionMiddleware` ทำหน้าที่ 2 จังหวะ:

| จังหวะ | สิ่งที่ทำ |
|---|---|
| **ขาเข้า (`process_request`)** | อ่าน cookie ชื่อ `sessionid` จาก request → ใช้ค่านั้นเป็น "กุญแจ" ไปโหลดข้อมูล session จาก backend (เช่น ฐานข้อมูล) มาผูกไว้ที่ `request.session` ให้ view เรียกใช้ได้ |
| **ขาออก (`process_response`)** | ตรวจสอบว่า session ถูกแก้ไข (`request.session.modified`) หรือมีการตั้งค่าที่ต้อง sync หรือไม่ → ถ้าใช่ จะบันทึกข้อมูลลง backend และส่ง header `Set-Cookie: sessionid=...` กลับไปให้เบราว์เซอร์ |

### 331.3 แผนภาพการทำงานแบบละเอียด: Request แรกเทียบกับ Request ถัดไป

```
=== Request แรกของผู้ใช้ (ยังไม่เคยมี cookie sessionid) ===

Browser ─────────── HTTP Request (ไม่มี cookie sessionid) ──────────> Django
                                                                          │
                                                          SessionMiddleware
                                                          (process_request):
                                                          ไม่พบ sessionid ในคุกกี้
                                                          → เตรียม session ว่างเปล่าไว้ก่อน
                                                                          │
                                                                          ▼
                                                      View: request.session['cart'] = [1, 2]
                                                                          │
                                                          SessionMiddleware
                                                          (process_response):
                                                          เห็นว่า session.modified == True
                                                          → สร้าง session key ใหม่ (สุ่ม 32 ตัวอักษร)
                                                          → บันทึกข้อมูลลง backend (เช่น django_session)
                                                          → แนบ Set-Cookie: sessionid=8f14e45f...
Browser <────────── HTTP Response (Set-Cookie: sessionid=8f14e45f...) ── Django


=== Request ถัดไป (เบราว์เซอร์แนบ cookie มาด้วยอัตโนมัติ) ===

Browser ─── Cookie: sessionid=8f14e45f... ───> Django
                                                    │
                                    SessionMiddleware (process_request):
                                    พบ sessionid ในคุกกี้
                                    → query backend ด้วยกุญแจนี้ → โหลดข้อมูลกลับมา
                                    → ผูกไว้ที่ request.session
                                                    │
                                                    ▼
                                View: request.session['cart']  →  ได้ [1, 2] เหมือนเดิม!
```

สังเกตว่า **ข้อมูลจริงไม่เคยถูกส่งไปกลับระหว่างเบราว์เซอร์กับเซิร์ฟเวอร์เลย** สิ่งที่ส่งไป
มีแค่ "กุญแจ" (session key) 32 ตัวอักษรแบบสุ่มเท่านั้น ข้อมูลจริงทั้งหมดอยู่ฝั่งเซิร์ฟเวอร์
เสมอ (ยกเว้น backend แบบ `signed_cookies` ที่จะเรียนในขั้นตอนที่ 332 ซึ่งเป็นข้อยกเว้น)

### 331.4 ตรวจสอบ cookie `sessionid` ด้วยตาตัวเอง

ลองรันโปรเจกต์ Django แล้วเปิด DevTools ของเบราว์เซอร์ (กด F12) ไปที่แท็บ
**Application → Cookies** (Chrome) หรือ **Storage → Cookies** (Firefox) คุณจะเห็น
cookie ชื่อ `sessionid` ที่มีค่าเป็นสตริงยาว ๆ แบบนี้:

```
Name: sessionid
Value: 8f14e45fceea167a5a36dedd4bea2543
Domain: localhost
Path: /
Expires: Session (หรือวันที่ ถ้าตั้ง SESSION_COOKIE_AGE)
HttpOnly: ✓
Secure: (ขึ้นกับ SESSION_COOKIE_SECURE)
SameSite: Lax
```

ค่านี้คือ **session key** ที่ Django ใช้อ้างอิงไปหาข้อมูลจริงในฐานข้อมูล (หรือ backend อื่น
ที่เลือกใช้) เราจะเจาะลึกความหมายของ `HttpOnly`, `Secure`, `SameSite` ในขั้นตอนที่ 335

### 331.5 ดูข้อมูล Session จริงในฐานข้อมูล (Default: Database Backend)

เมื่อยังไม่ได้เปลี่ยนค่า `SESSION_ENGINE` ใน `settings.py` Django จะใช้
**Database Backend** เป็นค่าเริ่มต้น ซึ่งเก็บข้อมูลไว้ในตาราง `django_session` ที่ถูกสร้าง
ให้อัตโนมัติผ่าน `django.contrib.sessions` (อยู่ใน `INSTALLED_APPS` มาตั้งแต่
`startproject`) ลองเปิดดูด้วย Django shell:

```bash
python manage.py shell
```

```python
>>> from django.contrib.sessions.models import Session
>>> Session.objects.all()
<QuerySet [<Session: 8f14e45fceea167a5a36dedd4bea2543>]>

>>> s = Session.objects.first()
>>> s.session_key
'8f14e45fceea167a5a36dedd4bea2543'
>>> s.expire_date
datetime.datetime(2026, 10, 10, 3, 12, 0, tzinfo=datetime.timezone.utc)
>>> s.get_decoded()
{'cart': [1, 2], '_auth_user_id': '7', '_auth_user_backend': 'django.contrib.auth.backends.ModelBackend'}
```

สังเกต 2 อย่างสำคัญ:

1. คอลัมน์ `session_data` ในตารางจริง ๆ **ไม่ได้เก็บเป็น plain text** แต่ถูก
   serialize (โดยค่าเริ่มต้นด้วย JSON) แล้ว **encode/sign** ด้วย `SECRET_KEY` ของโปรเจกต์
   ก่อนเก็บลงฐานข้อมูล ต้องเรียกผ่าน `.get_decoded()` เท่านั้นถึงจะอ่านค่าจริงได้
2. คีย์ `_auth_user_id` และ `_auth_user_backend` คือสิ่งที่ Django Authentication
   System (Part 031) ใช้เก็บว่า "ผู้ใช้คนไหนกำลัง login อยู่" — นี่คือหลักฐานว่าระบบ
   Authentication ทั้งหมดของ Django **สร้างอยู่บนรากฐานของ Session Framework**
   ที่กำลังเรียนใน Part นี้นั่นเอง

---

## ขั้นตอนที่ 332: Session Backend ที่ Django รองรับ

### 332.1 `SESSION_ENGINE`: สลับที่เก็บข้อมูล Session ได้โดยไม่ต้องแก้โค้ด View

Django ออกแบบระบบ Session ให้แยก "ตรรกะการใช้งาน" (`request.session`) ออกจาก
"ที่เก็บข้อมูลจริง" อย่างชัดเจน — ตัวอย่างคลาสสิกของหลักการ **abstraction** ทำให้คุณ
สลับ backend ได้เพียงแก้ค่า setting เดียว โดยที่โค้ด view ที่เขียนไว้ **ไม่ต้องแก้แม้แต่
บรรทัดเดียว**:

```python
# config/settings.py
SESSION_ENGINE = 'django.contrib.sessions.backends.db'   # ค่าเริ่มต้น
```

Django มาพร้อม 4 backend หลักให้เลือกใช้:

| ค่า `SESSION_ENGINE` | เก็บข้อมูลจริงที่ไหน |
|---|---|
| `django.contrib.sessions.backends.db` | ตาราง `django_session` ในฐานข้อมูลหลัก (ค่าเริ่มต้น) |
| `django.contrib.sessions.backends.cache` | Cache backend ที่ตั้งไว้ใน `CACHES` (เช่น Redis, Memcached) |
| `django.contrib.sessions.backends.cached_db` | ทั้งฐานข้อมูล**และ**cache ควบคู่กัน (write-through) |
| `django.contrib.sessions.backends.signed_cookies` | เก็บข้อมูลทั้งหมดในตัว cookie เอง (ไม่มี server-side storage) |

(ยังมี `django.contrib.sessions.backends.file` สำหรับเก็บเป็นไฟล์บน disk แต่ไม่นิยม
ใช้ในงาน production จริงเพราะสเกลข้าม server หลายเครื่องไม่ได้ จึงไม่ลงรายละเอียดใน
หลักสูตรนี้)

### 332.2 Database Backend (ค่าเริ่มต้น): เข้าใจง่าย ปลอดภัย แต่ช้ากว่า

```python
# config/settings.py
SESSION_ENGINE = 'django.contrib.sessions.backends.db'
```

ทุกครั้งที่ session ถูกอ่านหรือเขียน Django จะยิง query ไปที่ตาราง `django_session`
โดยตรง เหมาะกับโปรเจกต์ขนาดเล็กถึงกลาง หรือช่วงเริ่มพัฒนา เพราะ:

- ไม่ต้องติดตั้งอะไรเพิ่มเติม (ใช้ฐานข้อมูลที่มีอยู่แล้ว)
- ข้อมูล persistent เต็มรูปแบบ — เซิร์ฟเวอร์ restart ข้อมูล session ไม่หาย
- Debug ง่าย เพราะดูข้อมูลผ่าน Django Admin หรือ shell ได้ตรง ๆ

ข้อเสียคือทุก request ที่แตะ session (แทบทุก request ในเว็บที่มี login) จะเพิ่ม query
ไปที่ฐานข้อมูลหลัก 1 ครั้งเสมอ ซึ่งเป็นภาระเพิ่มเติมเมื่อ traffic สูงมาก

### 332.3 Cache Backend: เร็วที่สุด แต่เสี่ยงข้อมูลหายถ้า Cache ล่ม

```python
# config/settings.py
SESSION_ENGINE = 'django.contrib.sessions.backends.cache'
SESSION_CACHE_ALIAS = 'default'   # ชี้ไปยัง alias ใน CACHES ที่ต้องตั้งไว้ก่อน

CACHES = {
    'default': {
        'BACKEND': 'django.core.cache.backends.redis.RedisCache',
        'LOCATION': 'redis://127.0.0.1:6379/1',
    }
}
```

Session ทั้งหมดจะถูกเก็บใน **memory-based cache** อย่าง Redis หรือ Memcached
(เจาะลึกเรื่อง Caching Framework เต็มรูปแบบใน Phase 9) ทำให้อ่าน/เขียนเร็วกว่า
database backend มาก แต่มีความเสี่ยงสำคัญ: **ถ้า cache server ล่มหรือรีสตาร์ทโดยไม่มี
persistence (เช่น Memcached ที่ default ไม่บันทึกลง disk) ข้อมูล session ของผู้ใช้
ทุกคนจะหายทันที** ผู้ใช้ที่ login อยู่จะถูกเด้งออกจากระบบทั้งหมด

### 332.4 Cached_db Backend: จุดสมดุลระหว่างความเร็วกับความปลอดภัยของข้อมูล

```python
# config/settings.py
SESSION_ENGINE = 'django.contrib.sessions.backends.cached_db'
SESSION_CACHE_ALIAS = 'default'
```

`cached_db` ทำงานแบบ **write-through cache**: เขียนข้อมูลลงทั้ง cache และฐานข้อมูล
พร้อมกันทุกครั้ง แต่**อ่าน**จาก cache ก่อนเสมอ (เร็ว) ถ้า cache ไม่มีข้อมูล (เช่น cache
เพิ่งถูกล้าง) จะ fallback ไปอ่านจากฐานข้อมูลแทนโดยอัตโนมัติ ทำให้ได้ทั้งความเร็วของ
cache และความทนทานของฐานข้อมูลไปพร้อมกัน — **นี่คือ backend ที่แนะนำที่สุดสำหรับ
โปรเจกต์ production ระดับกลางถึงใหญ่ที่มี Redis อยู่แล้วในระบบ**

### 332.5 Signed Cookies Backend: ไม่มี Server-Side Storage เลย

```python
# config/settings.py
SESSION_ENGINE = 'django.contrib.sessions.backends.signed_cookies'
```

backend นี้แตกต่างจาก 3 แบบข้างต้นโดยสิ้นเชิง: **ข้อมูล session ทั้งหมดถูกเก็บอยู่ใน
ตัว cookie เอง** ไม่มีการเก็บอะไรไว้ฝั่งเซิร์ฟเวอร์เลย โดย Django จะ serialize ข้อมูล
เป็น JSON แล้ว **เซ็นชื่อ (sign)** ด้วย `SECRET_KEY` ก่อนส่งไปเป็นค่า cookie

> **จุดที่เข้าใจผิดบ่อยที่สุด**: "signed" ไม่ใช่ "encrypted" — ข้อมูลใน cookie
> **อ่านได้** โดยใครก็ตามที่เปิด DevTools ดู (แค่ decode base64) เพียงแต่ **แก้ไขค่า
> ไม่ได้** เพราะถ้าแก้แม้แต่ตัวอักษรเดียว ลายเซ็นจะไม่ตรงกับข้อมูล Django จะปฏิเสธและ
> ถือว่า session นั้นไม่ถูกต้องทันที **ห้ามเก็บข้อมูลสำคัญหรือเป็นความลับ** (เช่น
> role ผู้ใช้แบบดิบ, ข้อมูลบัตรเครดิต) ลงใน backend นี้เด็ดขาด

ข้อจำกัดสำคัญอีกข้อคือ **ขนาด**: เบราว์เซอร์ส่วนใหญ่จำกัดขนาด cookie ต่อโดเมนไว้ที่
ประมาณ **4KB** ถ้าข้อมูล session ใหญ่เกินนี้ (เช่น ตะกร้าสินค้าที่มีของหลายสิบชิ้น)
จะถูกตัดหรือ error ทันที และเพราะข้อมูลถูกส่งไปกลับทุก request จึงกิน bandwidth
มากกว่า backend อื่นที่ส่งแค่ session key สั้น ๆ

### 332.6 ตารางเปรียบเทียบข้อดี-ข้อเสียทั้ง 4 Backend

| ประเด็น | `db` (default) | `cache` | `cached_db` | `signed_cookies` |
|---|---|---|---|---|
| ที่เก็บข้อมูลจริง | ฐานข้อมูลหลัก | Cache (Redis/Memcached) | ทั้ง cache และฐานข้อมูล | ตัว cookie เอง |
| ความเร็วในการอ่าน/เขียน | ปานกลาง | เร็วมาก | เร็วมาก (อ่านจาก cache) | เร็ว (ไม่มี round-trip เพิ่ม) |
| ข้อมูลทนทานเมื่อ server รีสตาร์ท | ✅ ทนทาน | ❌ เสี่ยงหายถ้า cache ไม่มี persistence | ✅ ทนทาน (fallback ไปฐานข้อมูล) | ✅ ทนทาน (อยู่ที่เบราว์เซอร์) |
| ต้องติดตั้งอะไรเพิ่ม | ไม่ต้อง | ต้องมี Redis/Memcached | ต้องมี Redis/Memcached | ไม่ต้อง |
| ภาระต่อฐานข้อมูลหลัก | สูง (query ทุก request) | ไม่มี | ต่ำ (เขียนเบื้องหลัง อ่านจาก cache) | ไม่มี |
| ขนาดข้อมูลสูงสุดที่เก็บได้ | ไม่จำกัด (ตามฐานข้อมูล) | ไม่จำกัด (ตาม cache) | ไม่จำกัด | ~4KB (ข้อจำกัดของ cookie) |
| ความปลอดภัยของข้อมูล | สูง (อยู่ฝั่งเซิร์ฟเวอร์เท่านั้น) | สูง | สูง | ปานกลาง (ผู้ใช้อ่านเนื้อหาได้ แก้ไม่ได้) |
| Server ต้องเป็น stateless เต็มรูปแบบ (สเกลหลายเครื่องได้ทันที) | ต้องใช้ฐานข้อมูลกลางร่วมกัน | ต้องใช้ cache กลางร่วมกัน | ต้องใช้ทั้งคู่ร่วมกัน | ✅ ได้ทันที (ไม่มี state ฝั่งเซิร์ฟเวอร์เลย) |
| แนะนำสำหรับ | โปรเจกต์เล็ก-กลาง, ช่วงพัฒนา | งานที่ต้องการความเร็วสูงสุด ยอมรับความเสี่ยง | Production ระดับกลาง-ใหญ่ที่มี Redis อยู่แล้ว | API/Microservice ที่ต้องการ stateless เต็มรูปแบบ ข้อมูล session เล็กมาก |

### 332.7 คำแนะนำระดับมืออาชีพในการเลือก Backend

- **ช่วงเริ่มพัฒนา/โปรเจกต์เรียนรู้ (หลักสูตรนี้)**: ใช้ `db` (ค่าเริ่มต้น) ไปก่อน
  เพราะเข้าใจง่าย debug สะดวก ไม่ต้องติดตั้งอะไรเพิ่ม
- **Production ที่มี Redis อยู่แล้ว (เช่น ใช้ทำ caching หรือ Celery broker)**:
  เปลี่ยนเป็น `cached_db` แทบจะทันทีที่มี Redis พร้อมใช้งาน เพราะได้ประโยชน์เต็มที่
  โดยความเสี่ยงต่ำมาก (เราจะติดตั้ง Redis จริงใน Phase 9)
- **ระบบที่ต้องการ scale แนวนอนแบบ stateless เต็มรูปแบบ (เช่น serverless functions)**:
  พิจารณา `signed_cookies` แต่ต้องเก็บเฉพาะข้อมูลเล็ก ๆ ที่ไม่ลับ
- **หลีกเลี่ยง `cache` เดี่ยว ๆ (ไม่มี `_db`) ใน production** เว้นแต่จะยอมรับความเสี่ยง
  ที่ผู้ใช้ทุกคนอาจถูกเด้งออกจากระบบพร้อมกันถ้า cache ล่ม

---

## ขั้นตอนที่ 333: การใช้งาน `request.session`

### 333.1 `request.session` มีหน้าตาเหมือน Python Dictionary

Django ออกแบบ `request.session` ให้ใช้งานเหมือน `dict` ทุกประการ (จริง ๆ แล้วมัน
implement `MutableMapping` ของ Python) ทำให้นักพัฒนาที่คุ้นเคย dict อยู่แล้วใช้งานได้
ทันทีโดยไม่ต้องเรียนรู้ API ใหม่:

```python
# views.py
def some_view(request):
    # เขียนค่า (เหมือน dict ทุกประการ)
    request.session['favorite_color'] = 'blue'
    request.session['visit_count'] = 1

    # อ่านค่าแบบตรง (ระวัง KeyError ถ้าไม่มี key นี้)
    color = request.session['favorite_color']

    # อ่านค่าแบบปลอดภัยด้วย .get() (แนะนำเสมอ)
    count = request.session.get('visit_count', 0)

    # ตรวจสอบว่ามี key นี้อยู่หรือไม่
    if 'favorite_color' in request.session:
        ...

    # ลบค่าออกแบบปลอดภัย (ไม่ error ถ้าไม่มี key)
    request.session.pop('favorite_color', None)

    # ลบค่าออกแบบตรง (ระวัง KeyError ถ้าไม่มี key)
    del request.session['visit_count']

    # ดู key ทั้งหมดที่มีอยู่ใน session
    all_keys = list(request.session.keys())

    return render(request, 'some_template.html')
```

### 333.2 ตัวอย่างจริง: นับจำนวนครั้งที่ผู้ใช้เข้าเยี่ยมชมหน้าเว็บ

```python
# views.py
def homepage(request):
    visit_count = request.session.get('visit_count', 0)
    visit_count += 1
    request.session['visit_count'] = visit_count

    return render(request, 'core/home.html', {'visit_count': visit_count})
```

```html
<!-- templates/core/home.html -->
<p>คุณเข้าชมหน้านี้มาแล้ว {{ visit_count }} ครั้ง</p>
```

ลอง refresh หน้าเว็บซ้ำ ๆ จะเห็นตัวเลขเพิ่มขึ้นเรื่อย ๆ ทั้งที่แต่ละ request เป็นคนละ
request กันโดยสิ้นเชิงในมุมมองของ HTTP — นี่คือพลังของ Session ที่ทำให้แอปพลิเคชัน
"จำ" สถานะข้ามหลาย request ได้

### 333.3 `.flush()`: ล้างข้อมูล Session ทั้งหมดและสร้าง Key ใหม่

```python
def custom_logout_view(request):
    request.session.flush()
    return redirect('home')
```

`.flush()` ทำ 2 อย่างพร้อมกัน:

1. ลบข้อมูล session **ทั้งหมด**ออกจาก backend (เช่น ลบแถวในตาราง `django_session`)
2. สร้าง **session key ใหม่** ที่ไม่เกี่ยวข้องกับ key เดิมเลย (คนละค่ากับก่อน flush)

นี่คือเมธอดที่ Django ใช้ภายใน `django.contrib.auth.logout()` เอง (จะเรียนใน
Part 031) เพราะเมื่อผู้ใช้ logout ต้องแน่ใจว่า **ไม่มีทางเดาหรือใช้ session key เดิม
กลับมาแอบอ้างเป็นผู้ใช้คนนั้นได้อีก** — ต่างจาก `.clear()` ที่แค่ล้างข้อมูลในหน่วยความจำ
แต่ session key เดิมยังคงอยู่จนกว่าจะมีการบันทึกรอบถัดไป

### 333.4 `.set_expiry()`: กำหนดอายุของ Session เฉพาะรายครั้ง

```python
# หมดอายุเมื่อปิดเบราว์เซอร์ (ส่ง 0)
request.session.set_expiry(0)

# หมดอายุใน 1800 วินาที (30 นาที) นับจากตอนนี้
request.session.set_expiry(1800)

# หมดอายุ ณ วันเวลาที่กำหนดแน่นอน
from datetime import timedelta
from django.utils import timezone
request.session.set_expiry(timezone.now() + timedelta(days=7))

# กลับไปใช้ค่า default จาก SESSION_COOKIE_AGE ใน settings.py
request.session.set_expiry(None)
```

`.set_expiry()` มีประโยชน์มากเมื่อต้องการทำฟีเจอร์ **"จดจำฉันไว้" (Remember Me)**
แบบละเอียด — เช่น ถ้าผู้ใช้ติ๊กช่อง "จดจำฉันไว้ 30 วัน" ให้เรียก
`request.session.set_expiry(60 * 60 * 24 * 30)` แต่ถ้าไม่ติ๊ก ให้เรียก
`request.session.set_expiry(0)` เพื่อให้ session หมดอายุทันทีที่ปิดเบราว์เซอร์
(รายละเอียดฟีเจอร์นี้เต็มรูปแบบจะอยู่ใน Part 035)

เมธอดที่เกี่ยวข้องสำหรับตรวจสอบค่าปัจจุบัน:

```python
request.session.get_expiry_age()               # จำนวนวินาทีที่เหลือก่อนหมดอายุ
request.session.get_expiry_date()               # วันเวลาที่จะหมดอายุ (datetime object)
request.session.get_expire_at_browser_close()   # True/False
```

### 333.5 ข้อควรระวัง: การแก้ไข Nested Object ไม่ถูกตรวจจับอัตโนมัติ

จุดที่มือใหม่พลาดบ่อยที่สุดเกี่ยวกับ `request.session` คือการแก้ไข **ข้อมูลที่ซ้อนอยู่
ข้างใน** (เช่น list หรือ dict ที่เก็บอยู่ในค่าของ session):

```python
# ❌ กรณีที่ Django "ไม่รู้" ว่า session ถูกแก้ไข
def add_item_wrong(request):
    if 'items' not in request.session:
        request.session['items'] = []

    request.session['items'].append('new_item')   # แก้ list ที่อยู่ข้างในโดยตรง
    # SessionMiddleware จะไม่บันทึกการเปลี่ยนแปลงนี้ลง backend!
    # เพราะ __setitem__ ของ request.session ไม่ได้ถูกเรียกตรง ๆ
    return redirect('cart_detail')
```

Django ตรวจจับการเปลี่ยนแปลงผ่านการเรียก `request.session['key'] = value` เท่านั้น
(ซึ่งจะตั้ง flag ภายในชื่อ `modified = True` ให้อัตโนมัติ) แต่การเรียก
`.append()`, `.update()`, หรือแก้ dict ข้างในโดยตรง **ไม่ผ่าน `__setitem__` ของ
session เอง** จึงไม่ถูกตรวจจับ วิธีแก้คือตั้งค่า `request.session.modified = True`
ด้วยตัวเองเสมอเมื่อแก้ไขข้อมูลแบบซ้อน:

```python
# ✅ วิธีที่ถูกต้อง
def add_item_correct(request):
    if 'items' not in request.session:
        request.session['items'] = []

    request.session['items'].append('new_item')
    request.session.modified = True   # ← บอก Django ตรง ๆ ว่ามีการแก้ไขเกิดขึ้น
    return redirect('cart_detail')
```

จุดนี้สำคัญมากและจะกลับมาใช้จริงในขั้นตอนที่ 336 ตอนสร้างระบบตะกร้าสินค้า เพราะ
โครงสร้างตะกร้า (dict ซ้อน dict) เป็น nested object แบบเป๊ะ ๆ ที่ต้องระวังเรื่องนี้

---

## ขั้นตอนที่ 334: ตั้งค่าอายุ Session

### 334.1 `SESSION_COOKIE_AGE`: อายุของ Session เป็นวินาที

```python
# config/settings.py
SESSION_COOKIE_AGE = 1209600   # ค่าเริ่มต้นของ Django = 2 สัปดาห์ (14 * 24 * 60 * 60)
```

นี่คือค่า **default** ที่ใช้เมื่อไม่มีการเรียก `.set_expiry()` เจาะจงในโค้ด กำหนดว่า
cookie `sessionid` จะมีอายุกี่วินาทีนับจากตอนที่ถูกสร้าง/อัปเดตล่าสุด ตัวอย่างค่าที่
พบบ่อยในโปรเจกต์จริง:

| ค่า (วินาที) | เท่ากับ | สถานการณ์ที่เหมาะสม |
|---|---|---|
| `1800` | 30 นาที | ระบบธนาคาร/การเงินที่ต้องการความปลอดภัยสูง |
| `3600` | 1 ชั่วโมง | แดชบอร์ดผู้ดูแลระบบ (admin panel) |
| `86400` | 1 วัน | เว็บแอปทั่วไป |
| `1209600` | 2 สัปดาห์ (ค่าเริ่มต้น Django) | เว็บทั่วไปที่เน้นความสะดวกผู้ใช้ |
| `2592000` | 30 วัน | ฟีเจอร์ "จดจำฉันไว้" |

### 334.2 `SESSION_EXPIRE_AT_BROWSER_CLOSE`: หมดอายุทันทีที่ปิดเบราว์เซอร์

```python
# config/settings.py
SESSION_EXPIRE_AT_BROWSER_CLOSE = False   # ค่าเริ่มต้น
```

เมื่อตั้งเป็น `True` Django จะส่ง cookie แบบ **session cookie** (ไม่มีวันที่ `Expires`
กำกับ) ซึ่งเป็นชนิด cookie ที่เบราว์เซอร์**ลบทิ้งอัตโนมัติทันทีที่ปิดหน้าต่างเบราว์เซอร์
ทั้งหมด** (ไม่ใช่แค่ปิดแท็บ) ค่านี้จะ **override** ค่า `SESSION_COOKIE_AGE` เสมอ
(ทำให้ `SESSION_COOKIE_AGE` แทบไม่มีผลถ้าตั้งค่านี้เป็น `True`)

> **ข้อควรระวัง**: เบราว์เซอร์สมัยใหม่หลายตัว (โดยเฉพาะ Chrome ที่เปิดใช้ฟีเจอร์
> "Continue where you left off" หรือการกู้คืนแท็บอัตโนมัติ) **อาจไม่ลบ session
> cookie จริง ๆ** เมื่อปิดเบราว์เซอร์ เพราะเบราว์เซอร์คืนสถานะแท็บทั้งหมดกลับมารวมถึง
> cookie ด้วย ดังนั้นห้ามพึ่งพาค่านี้เป็นมาตรการความปลอดภัย**เพียงอย่างเดียว** ควรใช้
> ร่วมกับ `SESSION_COOKIE_AGE` ที่สั้นพอสมควรเสมอเป็นเกราะป้องกันชั้นที่สอง

### 334.3 `SESSION_SAVE_EVERY_REQUEST`: ต่ออายุ Session ทุกครั้งที่มี Activity

```python
# config/settings.py
SESSION_SAVE_EVERY_REQUEST = False   # ค่าเริ่มต้น
```

โดยค่าเริ่มต้น (`False`) Django จะบันทึก session ลง backend **เฉพาะตอนที่ข้อมูลใน
session ถูกแก้ไข** (`request.session.modified == True`) เท่านั้น หมายความว่าถ้าผู้ใช้
แค่เปิดดูหน้าเว็บโดยไม่มีการเขียนอะไรลง session เลย อายุของ session จะ**ไม่ถูกต่อ**
แม้ผู้ใช้จะยังคง active อยู่ตลอดก็ตาม — เมื่อถึงเวลาตาม `SESSION_COOKIE_AGE`
session จะหมดอายุแม้ผู้ใช้กำลังใช้งานอยู่

ถ้าตั้งเป็น `True` Django จะบันทึก (และต่ออายุ cookie) **ทุกครั้งที่มี request เข้ามา**
ไม่ว่าข้อมูลจะถูกแก้ไขหรือไม่ ทำให้ได้พฤติกรรมแบบ **sliding expiration** (นับอายุใหม่
ทุกครั้งที่มี activity) ซึ่งเป็นพฤติกรรมที่ผู้ใช้ทั่วไปคาดหวัง (เช่น "ตราบใดที่ฉันยังใช้งาน
เว็บอยู่ ไม่ควรถูกเด้งออก") แต่แลกมาด้วย **query/write เพิ่มขึ้นทุก request** ซึ่งเป็น
ภาระต่อฐานข้อมูลหรือ cache backend มากขึ้นตามสัดส่วน traffic

### 334.4 ตารางสรุปพฤติกรรมเมื่อผสมค่าต่าง ๆ กัน

| `SESSION_EXPIRE_AT_BROWSER_CLOSE` | `SESSION_SAVE_EVERY_REQUEST` | พฤติกรรมที่ได้ |
|---|---|---|
| `False` | `False` (ค่าเริ่มต้นทั้งคู่) | Session หมดอายุตาม `SESSION_COOKIE_AGE` นับจากครั้งล่าสุดที่มีการ**แก้ไข**ข้อมูล ไม่ใช่แค่เข้าดู |
| `False` | `True` | Session หมดอายุตาม `SESSION_COOKIE_AGE` แต่นับใหม่ทุกครั้งที่มี request (sliding expiration) |
| `True` | `False`/`True` | Session หมดอายุทันทีที่ปิดเบราว์เซอร์ ไม่ว่าจะตั้งค่าที่สองอย่างไร (ค่านี้มีอำนาจเหนือกว่า) |

### 334.5 ตัวอย่างการตั้งค่าจริง: ระบบที่ต้องการ Sliding Expiration 1 ชั่วโมง

สถานการณ์ทั่วไปในงานจริง: "ให้ผู้ใช้อยู่ในระบบได้นานเท่าที่ยังใช้งานต่อเนื่อง แต่ถ้าไม่มี
กิจกรรมใด ๆ นาน 1 ชั่วโมง ให้ตัดออกจากระบบเพื่อความปลอดภัย" ตั้งค่าได้ดังนี้:

```python
# config/settings.py
SESSION_COOKIE_AGE = 3600            # 1 ชั่วโมง
SESSION_SAVE_EVERY_REQUEST = True    # นับอายุใหม่ทุก request ที่มี activity
SESSION_EXPIRE_AT_BROWSER_CLOSE = False
```

ด้วยค่านี้ ตราบใดที่ผู้ใช้กด refresh หรือคลิกลิงก์ในเว็บไซต์อย่างน้อยทุก ๆ ไม่เกิน 1
ชั่วโมง session จะไม่มีวันหมดอายุ แต่ถ้าปล่อยทิ้งไว้เกิน 1 ชั่วโมงโดยไม่มี request ใด ๆ
เข้ามาเลย session จะหมดอายุทันที

---

## ขั้นตอนที่ 335: ความปลอดภัยของ Cookie

### 335.1 `SESSION_COOKIE_SECURE`: บังคับส่งผ่าน HTTPS เท่านั้น

```python
# config/settings.py
SESSION_COOKIE_SECURE = True   # ค่าเริ่มต้นคือ False
```

เมื่อตั้งเป็น `True` เบราว์เซอร์จะแนบ cookie `sessionid` ไปกับ request **เฉพาะที่ส่งผ่าน
HTTPS เท่านั้น** ถ้ามีใครพยายามเข้าเว็บผ่าน `http://` (ไม่เข้ารหัส) เบราว์เซอร์จะไม่ส่ง
cookie นี้ไปด้วยเลย ป้องกันไม่ให้ session key รั่วไหลผ่านการดักฟัง network แบบ
plain-text (เช่น บน Wi-Fi สาธารณะที่ไม่ปลอดภัย)

> **กฎเหล็กสำหรับ Production**: **ต้องตั้ง `SESSION_COOKIE_SECURE = True` เสมอ**
> เมื่อ deploy จริงที่มี HTTPS (ซึ่งควรมีเสมอในปี 2026) แต่ **ห้ามตั้งเป็น `True`
> ระหว่างพัฒนาในเครื่อง** (`DEBUG = True`, รันด้วย `runserver` ที่เป็น HTTP ธรรมดา)
> เพราะจะทำให้ login ไม่ติดเลยแม้แต่ครั้งเดียว (cookie จะไม่ถูกส่งกลับมาเนื่องจากเป็น
> HTTP) วิธีจัดการที่ถูกต้องคือผูกค่านี้ไว้กับตัวแปรสภาพแวดล้อม ซึ่งจะเจาะลึกเรื่อง
> การแยกค่า config ตาม environment แบบเต็มรูปแบบใน Part 010 ที่ผ่านมาแล้ว:

```python
# config/settings.py
import os

SESSION_COOKIE_SECURE = os.environ.get('DJANGO_ENV') == 'production'
```

### 335.2 `SESSION_COOKIE_HTTPONLY`: ปิดกั้นไม่ให้ JavaScript อ่าน Cookie ได้

```python
# config/settings.py
SESSION_COOKIE_HTTPONLY = True   # ค่าเริ่มต้นของ Django คือ True อยู่แล้ว
```

`HttpOnly` เป็น flag ระดับเบราว์เซอร์ (ไม่ใช่กลไกเฉพาะของ Django) ที่บอกเบราว์เซอร์ว่า
**ห้ามให้ JavaScript ฝั่ง client เข้าถึง cookie นี้ผ่าน `document.cookie` โดยเด็ดขาด**
ลองเปิด Console ของเบราว์เซอร์แล้วพิมพ์ `document.cookie` — cookie ที่มี `HttpOnly`
จะไม่ปรากฏในผลลัพธ์เลย

นี่คือมาตรการป้องกันชั้นสำคัญต่อ **XSS (Cross-Site Scripting)** ที่เรียนไปแล้วใน
Part 008 (ขั้นตอนที่ 78): ถึงแม้ผู้โจมตีจะสามารถแทรกโค้ด JavaScript อันตรายเข้าไปใน
หน้าเว็บได้สำเร็จ (ผ่านช่องโหว่อื่น) โค้ดนั้นก็**ไม่สามารถอ่านค่า `sessionid` ไปขโมยส่ง
ให้เซิร์ฟเวอร์ของผู้ไม่หวังดีได้** เพราะ `HttpOnly` ปิดกั้นไว้ตั้งแต่ระดับเบราว์เซอร์
Django ตั้งค่านี้เป็น `True` มาให้ตั้งแต่ต้น**ไม่ควรเปลี่ยนเป็น `False` เว้นแต่มีเหตุผล
จำเป็นจริง ๆ** (แทบไม่มีเหตุผลที่ดีเลยในการปิดมัน)

### 335.3 `SESSION_COOKIE_SAMESITE`: ป้องกัน Cross-Site Request

```python
# config/settings.py
SESSION_COOKIE_SAMESITE = 'Lax'   # ค่าเริ่มต้นของ Django
```

`SameSite` ควบคุมว่า cookie นี้จะถูกส่งไปพร้อม request ที่มาจาก**เว็บไซต์อื่น**
(cross-site request) หรือไม่ — เป็นกลไกป้องกันการโจมตีแบบ **CSRF** อีกชั้นหนึ่งที่ทำงาน
ระดับเบราว์เซอร์ (แยกจากกลไก CSRF token ของ Django ที่เรียนใน Part 008 แต่เสริมกัน)
มี 3 ค่าให้เลือก:

| ค่า | พฤติกรรม | ตัวอย่างสถานการณ์ |
|---|---|---|
| `'Strict'` | cookie จะถูกส่งไปก็ต่อเมื่อ request มาจากโดเมนเดียวกัน**เท่านั้น** แม้แต่การคลิกลิงก์จากเว็บอื่นมาที่เว็บเราก็ไม่ส่ง cookie ไปด้วยในรอบแรก | ระบบธนาคาร/การเงินที่ต้องการความปลอดภัยสูงสุด แต่ผู้ใช้จะดูเหมือน "ยังไม่ login" ทันทีที่คลิกลิงก์จากอีเมลหรือเว็บอื่นเข้ามา |
| `'Lax'` (ค่าเริ่มต้น) | ส่ง cookie ไปกับการ navigate ปกติข้ามเว็บไซต์ (เช่น คลิกลิงก์ `<a href>`, พิมพ์ URL เอง — ใช้ method GET) แต่**ไม่ส่ง**กับ request ที่มาจาก form POST ข้ามโดเมน หรือ `<img>`/`<iframe>` ที่ฝังจากเว็บอื่น | เว็บแอปทั่วไป — สมดุลระหว่างความปลอดภัยกับ user experience ที่ดี |
| `'None'` | ส่ง cookie ไปทุกกรณี ไม่ว่าจะเป็น cross-site แบบไหนก็ตาม (แต่**ต้องคู่กับ `SESSION_COOKIE_SECURE = True` เสมอ** ไม่เช่นนั้นเบราว์เซอร์สมัยใหม่จะปฏิเสธ cookie นี้ทันที) | ระบบที่ต้องฝัง iframe ข้ามโดเมน, Single Sign-On (SSO) ข้ามหลายโดเมนย่อย |

> **คำแนะนำของหลักสูตรนี้**: ใช้ค่าเริ่มต้น `'Lax'` สำหรับโปรเจกต์ทั่วไปเกือบทั้งหมด
> เพราะสมดุลที่สุด เปลี่ยนเป็น `'Strict'` เฉพาะระบบที่อ่อนไหวสูงมาก (เช่น
> internet banking) และใช้ `'None'` เฉพาะกรณีจำเป็นทางสถาปัตยกรรมจริง ๆ เท่านั้น
> (เช่น ระบบ SSO ที่จะเรียนใน Phase 4 ตอนท้าย)

### 335.4 Setting อื่น ๆ ที่เกี่ยวกับ Cookie ที่ควรรู้จัก

| Setting | ค่าเริ่มต้น | ความหมาย |
|---|---|---|
| `SESSION_COOKIE_NAME` | `'sessionid'` | ชื่อของ cookie — เปลี่ยนได้เพื่อลดโอกาสถูกสแกนหาช่องโหว่แบบเดารูปแบบมาตรฐานของ Django |
| `SESSION_COOKIE_PATH` | `'/'` | path ของเว็บไซต์ที่ cookie นี้จะถูกส่งไปด้วย ปกติใช้ `/` (ทั้งเว็บไซต์) เสมอ |
| `SESSION_COOKIE_DOMAIN` | `None` | โดเมนที่ cookie ใช้ได้ — ตั้งเป็น `.example.com` (มีจุดนำหน้า) ถ้าต้องการให้ cookie ใช้ร่วมกันได้ระหว่าง subdomain เช่น `shop.example.com` กับ `blog.example.com` |

### 335.5 ตัวอย่าง `settings.py` ที่พร้อมสำหรับ Production

```python
# config/settings.py — ส่วนของการตั้งค่า Session สำหรับ Production
import os

DJANGO_ENV = os.environ.get('DJANGO_ENV', 'development')
IS_PRODUCTION = DJANGO_ENV == 'production'

# --- Session Backend ---
SESSION_ENGINE = 'django.contrib.sessions.backends.cached_db'
SESSION_CACHE_ALIAS = 'default'

# --- อายุของ Session ---
SESSION_COOKIE_AGE = 60 * 60 * 4          # 4 ชั่วโมง
SESSION_SAVE_EVERY_REQUEST = True          # sliding expiration
SESSION_EXPIRE_AT_BROWSER_CLOSE = False

# --- ความปลอดภัยของ Cookie ---
SESSION_COOKIE_SECURE = IS_PRODUCTION      # True เฉพาะตอน production ที่มี HTTPS
SESSION_COOKIE_HTTPONLY = True             # ปิดกั้น JavaScript เสมอ (ค่าเริ่มต้นอยู่แล้ว)
SESSION_COOKIE_SAMESITE = 'Lax'
SESSION_COOKIE_NAME = 'myapp_sessionid'    # เปลี่ยนชื่อจากค่า default ของ Django
SESSION_COOKIE_PATH = '/'
```

ตั้งค่าลักษณะนี้ทำให้โปรเจกต์เดียวกัน **ทำงานถูกต้องได้ทั้งตอนพัฒนาในเครื่อง (HTTP)
และตอน deploy จริง (HTTPS)** โดยไม่ต้องแก้โค้ดสลับไปมาด้วยมือ — สอดคล้องกับหลักการ
แยก configuration ตาม environment ที่เรียนไปแล้วใน Part 010

---

## ขั้นตอนที่ 336: ตัวอย่างจริง — ตะกร้าสินค้าด้วย Session สำหรับผู้ใช้ที่ยังไม่ Login

### 336.1 ทำไมต้องใช้ Session สำหรับตะกร้าสินค้า

ระบบ e-commerce เกือบทุกระบบต้องรองรับกรณี **ผู้ใช้ยังไม่ login แต่อยากหยิบสินค้าใส่
ตะกร้าไปก่อน** แล้วค่อย login ตอนจะชำระเงิน การเก็บตะกร้าไว้ในฐานข้อมูลผูกกับ user
account (ForeignKey ไปยัง `User`) ทำไม่ได้ในกรณีนี้เพราะยังไม่มี user คนไหนผูกอยู่เลย
**Session คือคำตอบที่เหมาะสมที่สุด** เพราะ:

- ไม่ต้องมี user account ก็ใช้งานได้ (ผูกกับ browser ผ่าน cookie แทน)
- ข้อมูลอยู่ได้ข้ามหลาย request/หลายหน้า ตราบใดที่ session ยังไม่หมดอายุ
- เมื่อผู้ใช้ login ภายหลัง สามารถ "โอนย้าย" ตะกร้าจาก session ไปผูกกับ user
  account ถาวรในฐานข้อมูลได้ (เทคนิคนี้จะกล่าวถึงในแบบฝึกหัดท้ายบท)

### 336.2 เตรียม Model สินค้า (สมมติว่ามีแอป `shop` อยู่แล้ว)

```python
# shop/models.py
from django.db import models


class Product(models.Model):
    name = models.CharField(max_length=200)
    price = models.DecimalField(max_digits=10, decimal_places=2)
    stock = models.PositiveIntegerField(default=0)

    def __str__(self):
        return self.name
```

### 336.3 ออกแบบโครงสร้างข้อมูลตะกร้าใน Session

เก็บตะกร้าเป็น **dict ซ้อน dict** โดยใช้ `product_id` (แปลงเป็น string เพราะ Django
serialize session เป็น JSON ซึ่งบังคับให้ key ของ object ต้องเป็น string เสมอ) เป็นคีย์:

```python
{
    "3": {"quantity": 2, "price": "590.00"},
    "7": {"quantity": 1, "price": "1290.00"},
}
```

> **ทำไมต้องเก็บ `price` ซ้ำไว้ในตะกร้าด้วย ทั้งที่ query จาก `Product.price` ก็ได้**:
> เพื่อ "freeze" ราคา ณ เวลาที่หยิบใส่ตะกร้า ถ้าเจ้าของร้านเปลี่ยนราคาสินค้าในระหว่างที่
> ลูกค้ากำลังช้อปปิ้งอยู่ ราคาที่ลูกค้าเห็นในตะกร้าจะไม่กระโดดเปลี่ยนโดยไม่รู้ตัว —
> เป็นแนวปฏิบัติมาตรฐานของระบบ e-commerce ทุกระบบ

### 336.4 สร้างคลาส `Cart` เป็นตัวจัดการ Session (คั่นกลางระหว่าง View กับ Session)

```python
# cart/cart.py
from decimal import Decimal

from shop.models import Product

CART_SESSION_KEY = 'cart'


class Cart:
    """
    ตัวจัดการตะกร้าสินค้าที่ผูกอยู่กับ request.session
    ออกแบบให้ view เรียกใช้ได้สะดวกเหมือนใช้ object ทั่วไป
    โดยไม่ต้องยุ่งกับรายละเอียดของ session dict โดยตรง
    """

    def __init__(self, request):
        self.session = request.session
        cart = self.session.get(CART_SESSION_KEY)
        if cart is None:
            # ยังไม่เคยมีตะกร้ามาก่อน สร้างตะกร้าว่างแล้วบันทึกลง session ทันที
            cart = self.session[CART_SESSION_KEY] = {}
        self.cart = cart

    def add(self, product, quantity=1, override_quantity=False):
        """เพิ่มสินค้าลงตะกร้า หรืออัปเดตจำนวนถ้ามีอยู่แล้ว"""
        product_id = str(product.id)
        if product_id not in self.cart:
            self.cart[product_id] = {
                'quantity': 0,
                'price': str(product.price),   # เก็บเป็น string เพราะ Decimal ไม่ใช่ JSON-serializable
            }

        if override_quantity:
            self.cart[product_id]['quantity'] = quantity
        else:
            self.cart[product_id]['quantity'] += quantity

        self.save()

    def remove(self, product):
        """ลบสินค้าออกจากตะกร้าทั้งรายการ"""
        product_id = str(product.id)
        if product_id in self.cart:
            del self.cart[product_id]
            self.save()

    def save(self):
        """
        บังคับให้ Django รู้ว่า session ถูกแก้ไข
        จำเป็นเสมอเพราะเราแก้ไข dict ที่ซ้อนอยู่ข้างใน (nested object)
        ไม่ใช่การเรียก session['key'] = value ตรง ๆ (ทบทวนขั้นตอนที่ 333.5)
        """
        self.session.modified = True

    def clear(self):
        """ล้างตะกร้าทั้งหมด เช่น หลังชำระเงินสำเร็จ"""
        del self.session[CART_SESSION_KEY]
        self.save()

    def __iter__(self):
        """
        ให้ Cart object วนลูปได้ตรง ๆ ใน template ด้วย {% for item in cart %}
        พร้อมแนบ Product object และคำนวณราคารวมต่อรายการให้เสร็จสรรพ
        """
        product_ids = self.cart.keys()
        products = Product.objects.filter(id__in=product_ids)
        cart = self.cart.copy()

        for product in products:
            cart[str(product.id)]['product'] = product

        for item in cart.values():
            item['price'] = Decimal(item['price'])
            item['total_price'] = item['price'] * item['quantity']
            yield item

    def __len__(self):
        """ให้เรียก len(cart) เพื่อนับจำนวนชิ้นสินค้ารวมทั้งหมดได้"""
        return sum(item['quantity'] for item in self.cart.values())

    def get_total_price(self):
        """ราคารวมทั้งตะกร้า"""
        return sum(
            Decimal(item['price']) * item['quantity']
            for item in self.cart.values()
        )
```

### 336.5 Context Processor: ให้ทุกหน้าเห็นจำนวนสินค้าในตะกร้าได้ (navbar)

```python
# cart/context_processors.py
from .cart import Cart


def cart(request):
    return {'cart': Cart(request)}
```

```python
# config/settings.py
TEMPLATES = [
    {
        # ...
        'OPTIONS': {
            'context_processors': [
                # ... context processors เดิม
                'cart.context_processors.cart',   # ← เพิ่มบรรทัดนี้
            ],
        },
    },
]
```

(ทบทวน Context Processors แบบเต็มรูปแบบได้ที่ Part 008 ขั้นตอนที่ 79 และ Part 030)

### 336.6 Views: เพิ่ม/ลบ/แสดงตะกร้า

```python
# cart/views.py
from django.shortcuts import get_object_or_404, redirect, render
from django.views.decorators.http import require_POST

from shop.models import Product
from .cart import Cart


@require_POST
def cart_add(request, product_id):
    cart = Cart(request)
    product = get_object_or_404(Product, id=product_id)
    quantity = int(request.POST.get('quantity', 1))
    cart.add(product=product, quantity=quantity)
    return redirect('cart:cart_detail')


@require_POST
def cart_remove(request, product_id):
    cart = Cart(request)
    product = get_object_or_404(Product, id=product_id)
    cart.remove(product)
    return redirect('cart:cart_detail')


def cart_detail(request):
    cart = Cart(request)
    return render(request, 'cart/detail.html', {'cart': cart})
```

```python
# cart/urls.py
from django.urls import path
from . import views

app_name = 'cart'

urlpatterns = [
    path('', views.cart_detail, name='cart_detail'),
    path('add/<int:product_id>/', views.cart_add, name='cart_add'),
    path('remove/<int:product_id>/', views.cart_remove, name='cart_remove'),
]
```

```python
# config/urls.py
from django.urls import include, path

urlpatterns = [
    # ...
    path('cart/', include('cart.urls', namespace='cart')),
]
```

### 336.7 Template: แสดงรายการสินค้าในตะกร้า

```html
<!-- cart/templates/cart/detail.html -->
{% extends 'base.html' %}

{% block content %}
<h1>ตะกร้าสินค้าของคุณ</h1>

<table>
    <thead>
        <tr>
            <th>สินค้า</th>
            <th>ราคาต่อชิ้น</th>
            <th>จำนวน</th>
            <th>รวม</th>
            <th></th>
        </tr>
    </thead>
    <tbody>
        {% for item in cart %}
        <tr>
            <td>{{ item.product.name }}</td>
            <td>{{ item.price }} บาท</td>
            <td>{{ item.quantity }}</td>
            <td>{{ item.total_price }} บาท</td>
            <td>
                <form method="post" action="{% url 'cart:cart_remove' item.product.id %}">
                    {% csrf_token %}
                    <button type="submit">ลบ</button>
                </form>
            </td>
        </tr>
        {% empty %}
        <tr><td colspan="5">ยังไม่มีสินค้าในตะกร้า</td></tr>
        {% endfor %}
    </tbody>
</table>

<p><strong>ยอดรวมทั้งหมด: {{ cart.get_total_price }} บาท</strong></p>
{% endblock %}
```

```html
<!-- ตัวอย่างปุ่ม "เพิ่มลงตะกร้า" ในหน้า shop/detail.html -->
<form method="post" action="{% url 'cart:cart_add' product.id %}">
    {% csrf_token %}
    <input type="number" name="quantity" value="1" min="1">
    <button type="submit">เพิ่มลงตะกร้า</button>
</form>
```

สังเกตว่าทุกฟอร์ม `method="post"` มี `{% csrf_token %}` กำกับเสมอ ตามกฎเหล็กที่
เรียนไปแล้วใน Part 008 — เรื่องนี้จะเชื่อมโยงกับ Session โดยตรงในขั้นตอนถัดไป

### 336.8 ทดสอบด้วยตาตัวเอง: ตะกร้าอยู่ที่ไหน

ลองเพิ่มสินค้าลงตะกร้าโดย**ไม่ login** แล้วเปิด DevTools → Application → Cookies
คุณจะเห็นว่ามี cookie `sessionid` ถูกสร้างขึ้น (ถ้ายังไม่มีมาก่อน) และถ้าเปิด Django
shell ตรวจดูตาราง `django_session` (กรณีใช้ backend เริ่มต้น) จะเห็นข้อมูล
`{'cart': {'3': {'quantity': 2, 'price': '590.00'}}}` ถูกเก็บไว้จริง — พิสูจน์ว่า
ตะกร้าทั้งหมดทำงานอยู่บนกลไก Session ที่เรียนมาตั้งแต่ขั้นตอนที่ 331 โดยไม่ต้องมี user
account ใด ๆ เกี่ยวข้องเลย

---

## ขั้นตอนที่ 337: ทบทวนความสัมพันธ์ระหว่าง Session กับ CSRF Cookie

### 337.1 ทบทวนสั้น ๆ: `{% csrf_token %}` จาก Part 008

ใน Part 008 (ขั้นตอนที่ 78.5) เราเรียนไปแล้วว่า `{% csrf_token %}` ทำหน้าที่แทรก
hidden input ที่มี token สุ่มลงในทุกฟอร์ม `method="post"` เพื่อป้องกันการโจมตีแบบ
**CSRF (Cross-Site Request Forgery)** — ถ้ายังไม่คุ้นเคย แนะนำย้อนกลับไปทบทวนก่อน
เพราะ Part นี้จะพูดถึงเฉพาะ**ความสัมพันธ์กับ Session** เท่านั้น ไม่ลงรายละเอียดกลไก
CSRF ซ้ำอีกครั้ง

### 337.2 CSRF Token ผูกกับ Session อย่างไร: ขึ้นกับ `CSRF_USE_SESSIONS`

Django มี setting ชื่อ `CSRF_USE_SESSIONS` ที่กำหนดว่า **CSRF secret** จะถูกเก็บไว้
ที่ไหน:

```python
# config/settings.py
CSRF_USE_SESSIONS = False   # ค่าเริ่มต้นของ Django
```

| ค่า `CSRF_USE_SESSIONS` | CSRF secret ถูกเก็บที่ไหน |
|---|---|
| `False` (ค่าเริ่มต้น) | เก็บใน cookie แยกต่างหากชื่อ `csrftoken` (ไม่ผูกกับ `sessionid` เลย) |
| `True` | เก็บไว้**ใน**ข้อมูล session เอง (ผูกกับ `sessionid` โดยตรง) ไม่มีการสร้าง cookie `csrftoken` แยกอีกต่อไป |

โดยค่าเริ่มต้น (`False`) CSRF secret กับ Session key จึงเป็น**คนละ cookie กันโดย
สิ้นเชิง** — คุณจะเห็น 2 cookies แยกกันใน DevTools: `sessionid` และ `csrftoken`
ซึ่งเป็นเหตุผลที่แม้แต่หน้าที่ผู้ใช้**ยังไม่ login เลย** (เช่น หน้า login เอง หรือ
ฟอร์มติดต่อทั่วไป) ก็ยังมี CSRF protection ทำงานได้ปกติ เพราะ `csrftoken` cookie
ถูกสร้างขึ้นทันทีที่ template render ที่มี `{% csrf_token %}` ถูกเรียก โดยไม่ต้องรอให้
มี session ที่จริงจังอะไรมาก่อน

### 337.3 ทำไม Session หมดอายุระหว่างกรอกฟอร์ม ถึงทำให้ได้ HTTP 403

สถานการณ์ที่พบบ่อยในงานจริง: ผู้ใช้เปิดหน้าฟอร์มค้างไว้นานมาก (เช่น เขียนคอมเมนต์ยาว ๆ)
จน session หมดอายุระหว่างนั้น พอกด submit กลับได้ `HTTP 403 Forbidden` พร้อมข้อความ
"CSRF verification failed" ทั้งที่ผู้ใช้ไม่ได้ทำอะไรผิด

ถ้าโปรเจกต์ตั้ง `CSRF_USE_SESSIONS = True` เหตุผลชัดเจนมาก: CSRF secret ที่ฝังอยู่ใน
hidden input ของฟอร์ม (ที่ render ไว้ตอนโหลดหน้าครั้งแรก) จะไม่ตรงกับ secret ใหม่
อีกต่อไป เพราะ session เดิมถูกลบไปแล้วเมื่อหมดอายุ (ถ้าใช้ backend แบบ `db` เมื่อ
session หมดอายุ ข้อมูลจะถูกทำเครื่องหมายว่าใช้ไม่ได้ และค่า secret ที่เคยผูกไว้ก็หายไป
ด้วย) เมื่อ submit ฟอร์ม token ที่ส่งมาจึงตรวจสอบไม่ผ่าน

**แนวทางแก้ปัญหาระดับ UX ที่ทีมมืออาชีพใช้กัน**: ตั้ง `SESSION_COOKIE_AGE` ให้นานพอ
สมควรสำหรับฟอร์มที่ใช้เวลากรอกนาน หรือใช้ JavaScript แจ้งเตือนผู้ใช้ก่อน session
ใกล้หมดอายุ (เทคนิค "session timeout warning modal" ซึ่งเป็นแนวทางระดับ frontend
ที่อยู่นอกขอบเขตของ Part นี้) — ประเด็นสำคัญที่ต้องจำคือ **Session และ CSRF เป็นกลไก
คนละชั้นที่ทำงานประกอบกัน แต่ Session ที่หมดอายุสามารถทำให้ CSRF verification ล้มเหลว
ตามไปด้วยได้** ถ้าตั้งค่า `CSRF_USE_SESSIONS = True`

---

## ขั้นตอนที่ 338: ความเสี่ยง Session Hijacking และแนวทางป้องกัน

### 338.1 Session Hijacking คืออะไร

**Session Hijacking** คือการที่ผู้ไม่หวังดีขโมยหรือเดา **session key** ของผู้ใช้ที่
login อยู่แล้ว แล้วนำ key นั้นไปใช้แอบอ้างเป็นผู้ใช้คนนั้นได้ทันที โดย**ไม่ต้องรู้
username/password เลยแม้แต่น้อย** เพราะระบบตรวจสอบแค่ว่า session key ที่ส่งมาถูกต้อง
หรือไม่เท่านั้น ช่องทางหลักที่ผู้โจมตีใช้ขโมย session key มีดังนี้:

| ช่องทางการโจมตี | อธิบาย | มาตรการป้องกันหลัก |
|---|---|---|
| **XSS (Cross-Site Scripting)** | แทรก JavaScript อันตรายเพื่ออ่าน `document.cookie` แล้วส่งค่าไปให้เซิร์ฟเวอร์ของผู้โจมตี | `SESSION_COOKIE_HTTPONLY = True` (ขั้นตอนที่ 335.2) — ปิดกั้นไม่ให้ JS อ่าน cookie ได้เลย |
| **Network Sniffing** | ดักฟัง traffic ที่ไม่เข้ารหัส (เช่น Wi-Fi สาธารณะ) เพื่อขโมย cookie ที่ส่งผ่าน HTTP ธรรมดา | `SESSION_COOKIE_SECURE = True` (ขั้นตอนที่ 335.1) บังคับใช้ HTTPS เท่านั้น |
| **Session Fixation** | ผู้โจมตีกำหนด session key ที่ตัวเองรู้ล่วงหน้าให้เหยื่อใช้ (เช่น ส่งลิงก์ที่มี `?sessionid=xxx` หรือฝัง cookie ไว้ก่อน) แล้วรอให้เหยื่อ login ด้วย session key นั้น จากนั้นผู้โจมตีก็ใช้ key เดียวกันเข้าระบบในฐานะเหยื่อได้ทันที | **Regenerate session key ทุกครั้งที่มีการเปลี่ยนระดับสิทธิ์ (privilege change)** เช่น หลัง login สำเร็จ ด้วย `cycle_key()` — หัวข้อหลักของขั้นตอนนี้ |
| **Cross-Site Request Forgery ร่วมกับ Session** | แม้ไม่ได้ขโมย session key โดยตรง แต่หลอกให้เบราว์เซอร์ของเหยื่อ (ที่มี session ถูกต้องอยู่) ส่ง request อันตรายแทน | CSRF token (Part 008) + `SESSION_COOKIE_SAMESITE` (ขั้นตอนที่ 335.3) |

### 338.2 Session Fixation คือช่องโหว่ที่ Part นี้ต้องเจาะลึกที่สุด

ลองจินตนาการสถานการณ์นี้: เว็บไซต์ A ไม่มีมาตรการป้องกันใด ๆ — ผู้โจมตีเปิดเว็บ A
ด้วยตัวเอง ได้ session key มา `abc123` จากนั้นส่งลิงก์ (หรือหลอกฝัง cookie ผ่านช่องทาง
อื่น) ให้เหยื่อใช้ session key `abc123` เดียวกันนี้ เมื่อเหยื่อ **login สำเร็จ** ด้วย
username/password ของตัวเอง แต่ระบบไม่ได้เปลี่ยน session key เลย — ข้อมูลผู้ใช้ที่
login แล้วจึงถูกผูกเข้ากับ session key `abc123` ตัวเดิม ซึ่งผู้โจมตี**รู้ค่าอยู่แล้วตั้งแต่
ต้น** ผู้โจมตีจึงสามารถใช้ key เดียวกันนี้เข้าเว็บไซต์ A แล้วพบว่าตัวเองกลายเป็นผู้ใช้ที่
login สำเร็จไปแล้วโดยอัตโนมัติ

### 338.3 `cycle_key()`: สร้าง Session Key ใหม่โดยรักษาข้อมูลเดิมไว้

```python
request.session.cycle_key()
```

`cycle_key()` ต่างจาก `flush()` ตรงที่ **ไม่ลบข้อมูลใด ๆ ในตัว session** เพียงแค่
สร้าง **session key ใหม่** ให้แทนที่ key เดิม แล้วย้ายข้อมูลทั้งหมดไปผูกกับ key ใหม่นี้
ทำให้ key เดิมที่อาจถูกผู้โจมตีล่วงรู้มาก่อน **ใช้งานไม่ได้อีกต่อไปทันที** ในขณะที่ผู้ใช้
ตัวจริงยังคงข้อมูล session เดิมทั้งหมดไว้ (เช่น ตะกร้าสินค้าที่เพิ่งหยิบไว้ก่อน login
จะไม่หายไปไหน)

| เมธอด | ลบข้อมูล session? | สร้าง key ใหม่? | ใช้เมื่อไหร่ |
|---|---|---|---|
| `.flush()` | ✅ ลบทั้งหมด | ✅ | ตอน logout — ไม่ต้องการให้เหลือร่องรอยข้อมูลใด ๆ ของผู้ใช้เดิมอีก |
| `.cycle_key()` | ❌ เก็บข้อมูลเดิมไว้ | ✅ | ตอน login สำเร็จ หรือมีการเปลี่ยนระดับสิทธิ์ — ต้องการกัน session fixation แต่ไม่อยากทำข้อมูลที่มีอยู่ก่อนหน้าหาย |

### 338.4 ข่าวดี: Django's `login()` เรียก `cycle_key()`/`flush()` ให้อัตโนมัติอยู่แล้ว

ถ้าคุณใช้ `django.contrib.auth.login()` มาตรฐานตามที่จะเรียนใน Part 031 (แทนที่จะ
เขียนกลไก login เองทั้งหมดโดยไม่ผ่านฟังก์ชันนี้) **Django จัดการเรื่อง session
fixation ให้อัตโนมัติอยู่แล้ว**:

```python
# views.py
from django.contrib.auth import authenticate, login as auth_login
from django.shortcuts import render, redirect


def custom_login_view(request):
    if request.method == 'POST':
        username = request.POST.get('username')
        password = request.POST.get('password')
        user = authenticate(request, username=username, password=password)

        if user is not None:
            auth_login(request, user)
            # ภายในฟังก์ชัน auth_login() ด้านบนนี้ Django จะตรวจสอบ:
            # - ถ้า session เดิมยังไม่เคย login เป็นใครเลย → เรียก cycle_key()
            #   (สร้าง key ใหม่ แต่เก็บข้อมูลเดิม เช่น ตะกร้าสินค้า ไว้ครบ)
            # - ถ้า session เดิมผูกกับ user คนละคนอยู่แล้ว → เรียก flush() แทน
            #   (ป้องกันการใช้ session ต่อจากผู้ใช้คนก่อนหน้าโดยไม่ตั้งใจ)
            return redirect('home')

    return render(request, 'accounts/login.html')
```

นี่คือเหตุผลสำคัญที่หลักสูตรนี้ (และ Django เอง) **แนะนำให้ใช้
`django.contrib.auth.login()` เสมอ แทนการเขียนกลไก set session เองทั้งหมด** —
มาตรการป้องกัน session fixation ถูกฝังมาให้ "ฟรี" ในบรรทัดเดียว

### 338.5 เมื่อต้องเรียก `cycle_key()` ด้วยตัวเองอย่างชัดเจน

แม้ `login()` จะจัดการให้อัตโนมัติ แต่มีบางสถานการณ์ที่คุณต้อง **เรียก `cycle_key()`
เองอย่างชัดเจน** เพราะไม่ได้เรียก `login()` ซ้ำ ทั้งที่มีการเปลี่ยนระดับสิทธิ์ของผู้ใช้
เกิดขึ้นจริง เช่น:

- หลังผู้ใช้ยืนยันตัวตนขั้นที่สองสำเร็จ (Two-Factor Authentication — จะเรียนเต็ม
  รูปแบบใน Part 037)
- หลังผู้ใช้เปลี่ยนรหัสผ่านสำเร็จขณะที่ยัง login ค้างอยู่ (Part 035)
- หลัง admin ยกระดับสิทธิ์ผู้ใช้ (เช่น เปลี่ยนจาก user ธรรมดาเป็น staff) ในระบบที่
  session ผูกกับข้อมูลสิทธิ์แบบ cache ไว้

```python
# accounts/views.py — ตัวอย่างหลังยืนยัน OTP สำเร็จในระบบ 2FA
from django.contrib.auth.decorators import login_required
from django.shortcuts import redirect, render


@login_required
def verify_otp_view(request):
    if request.method == 'POST':
        otp_code = request.POST.get('otp_code')

        if verify_otp_for_user(request.user, otp_code):   # ฟังก์ชันสมมติ
            request.session['otp_verified'] = True
            request.session.cycle_key()   # regenerate key หลังยกระดับสิทธิ์การยืนยันตัวตน
            return redirect('dashboard')

    return render(request, 'accounts/verify_otp.html')
```

### 338.6 มาตรการเสริมอื่น ๆ ที่ควรทำควบคู่กันเสมอ

- **บังคับ HTTPS ทั้งเว็บไซต์เสมอใน production** ผ่าน `SECURE_SSL_REDIRECT = True`
  ควบคู่กับ `SESSION_COOKIE_SECURE = True` (จะเจาะลึก security settings ทั้งชุดใน
  Phase 10)
- **ตั้งอายุ Session ให้สั้นพอสมควร** โดยเฉพาะระบบที่อ่อนไหวสูง (ทบทวนขั้นตอนที่ 334)
  ยิ่ง session มีอายุยืนยาว ยิ่งมีหน้าต่างเวลาให้ผู้โจมตีใช้ประโยชน์จาก key ที่ขโมยได้
  นานขึ้นตามไปด้วย
- **แจ้ง logout ทุกอุปกรณ์เมื่อผู้ใช้เปลี่ยนรหัสผ่าน** ซึ่งทำได้ผ่านกลไก
  `get_session_auth_hash()` ที่จะเรียนเต็มรูปแบบใน Part 035
- **Monitor และ Log พฤติกรรมผิดปกติ** เช่น session เดียวถูกใช้จาก IP address หรือ
  User-Agent ที่เปลี่ยนไปกะทันหัน (เทคนิคระดับสูงที่จะกล่าวถึงใน Phase 10 เรื่อง
  Security Hardening)

---

## ขั้นตอนที่ 339: คำสั่ง `clearsessions` และการตั้ง Cron Job

### 339.1 ปัญหา: ทำไม Session ที่หมดอายุแล้วยังค้างอยู่ในฐานข้อมูล

เมื่อใช้ **Database Backend** (ค่าเริ่มต้น) การที่ session "หมดอายุ" หมายถึง Django
จะ**ไม่ยอมรับ**ค่านั้นเป็น session ที่ใช้งานได้อีกต่อไป (ตรวจสอบจากคอลัมน์
`expire_date` เทียบกับเวลาปัจจุบัน) แต่ **แถวข้อมูลในตาราง `django_session` ยัง
ไม่ถูกลบออกไปไหน** Django ไม่มีกลไก background process ที่คอยลบข้อมูลหมดอายุให้
อัตโนมัติ เพราะการรัน background job เป็นความรับผิดชอบของ deployment/infrastructure
ไม่ใช่ของตัว web framework โดยตรง

ถ้าปล่อยไว้นานเป็นเดือนเป็นปีโดยไม่จัดการ ตาราง `django_session` จะมีข้อมูลขยะสะสม
เพิ่มขึ้นเรื่อย ๆ (โดยเฉพาะเว็บที่มี traffic สูง มีผู้ใช้ไม่ login จำนวนมากที่แต่ละคนสร้าง
session ใหม่ทุกครั้ง) ส่งผลเสียต่อ:

- ขนาดฐานข้อมูลที่โตขึ้นโดยไม่จำเป็น
- ความเร็วของ query ที่เกี่ยวข้องกับตาราง `django_session` (แม้จะมี index บน
  `session_key` และ `expire_date` อยู่แล้ว แต่ตารางที่เล็กกว่าย่อมเร็วกว่าเสมอ)

### 339.2 คำสั่ง `clearsessions`: ลบ Session ที่หมดอายุออกจากฐานข้อมูล

Django มี management command สำเร็จรูปมาให้แก้ปัญหานี้โดยเฉพาะ:

```bash
python manage.py clearsessions
```

คำสั่งนี้จะลบทุกแถวในตาราง `django_session` ที่มีค่า `expire_date` **น้อยกว่าเวลา
ปัจจุบัน** (คือหมดอายุไปแล้วจริง ๆ) ออกทั้งหมด โดยไม่กระทบ session ที่ยัง active อยู่
เลยแม้แต่น้อย ปลอดภัยที่จะรันได้ตลอดเวลาโดยไม่ต้องปิดเว็บไซต์หรือหยุดผู้ใช้งาน

> **หมายเหตุสำคัญเรื่อง Backend อื่น**: คำสั่งนี้มีผลจริงเฉพาะ backend ที่เก็บข้อมูลลง
> ฐานข้อมูล คือ `db` และ `cached_db` เท่านั้น สำหรับ `cache` backend เดี่ยว ๆ
> ข้อมูลจะหมดอายุและถูกลบออกจาก cache เองโดยอัตโนมัติตามกลไก TTL ของ Redis/
> Memcached อยู่แล้ว ส่วน `signed_cookies` ไม่มี server-side storage ให้ต้องเคลียร์
> เลยตั้งแต่ต้น คำสั่ง `clearsessions` จึงไม่มีผลอะไรกับ 2 backend หลังนี้

### 339.3 ตั้งเป็น Cron Job ให้รันอัตโนมัติสม่ำเสมอ

เอกสารทางการของ Django แนะนำให้รันคำสั่งนี้เป็นประจำผ่าน cron job (บน Linux/macOS)
เพราะเป็นงานที่ **ไม่จำเป็นต้องมีคนคอยรันด้วยมือ** และควรเกิดขึ้นสม่ำเสมอในช่วงเวลาที่
traffic ต่ำ (เช่น ตอนกลางดึก) เพื่อไม่ให้กระทบ performance ของระบบ:

```bash
# แก้ไข crontab ด้วยคำสั่ง
crontab -e
```

```cron
# รันทุกวันตอนตี 3 (03:00)
0 3 * * * cd /path/to/django-mastery-course && /path/to/django-mastery-course/venv/bin/python manage.py clearsessions >> /var/log/django_clearsessions.log 2>&1
```

อธิบายรูปแบบ cron expression `0 3 * * *`:

| ตำแหน่ง | ค่า | ความหมาย |
|---|---|---|
| นาที | `0` | นาทีที่ 0 |
| ชั่วโมง | `3` | ชั่วโมงที่ 3 (ตี 3) |
| วันที่ในเดือน | `*` | ทุกวัน |
| เดือน | `*` | ทุกเดือน |
| วันในสัปดาห์ | `*` | ทุกวันในสัปดาห์ |

### 339.4 ทางเลือกอื่นสำหรับ Production ระดับมืออาชีพ

ในระบบ production จริงที่ deploy บน Linux server สมัยใหม่ มีทางเลือกอื่นนอกเหนือจาก
`crontab` ดั้งเดิมที่นิยมใช้กัน:

- **systemd timer**: ทางเลือกที่ทันสมัยกว่า cron บน Linux distro ยุคใหม่ ให้ log และ
  การจัดการ error ที่ดีกว่า (จะเจาะลึกเรื่อง systemd เต็มรูปแบบใน Phase 11 — DevOps)
- **django-crontab** หรือ **django-celery-beat**: package ที่ให้ตั้ง scheduled
  task ผ่านโค้ด Python/Django settings โดยตรง แทนที่จะไปแก้ crontab ของระบบปฏิบัติการ
  เหมาะกับทีมที่ต้องการเก็บ configuration ทั้งหมดไว้ใน version control ของโปรเจกต์เอง
  (จะเจาะลึก Celery แบบเต็มรูปแบบใน Phase 9)
- **Managed platform's scheduled jobs**: ถ้า deploy บนแพลตฟอร์มอย่าง Heroku,
  Railway, หรือ AWS จะมีฟีเจอร์ scheduled job/cron ในตัวแพลตฟอร์มเองให้ตั้งค่าผ่าน
  dashboard โดยไม่ต้องยุ่งกับ crontab ของเซิร์ฟเวอร์โดยตรง

**คำแนะนำของหลักสูตรนี้**: สำหรับตอนนี้ให้เข้าใจหลักการและใช้ `crontab` แบบพื้นฐานไป
ก่อน เมื่อถึง Phase 11 (DevOps) เราจะกลับมาตั้งค่าเรื่องนี้อย่างเป็นระบบมากขึ้นในบริบท
ของการ deploy จริงบน production server

---

## ขั้นตอนที่ 340: สรุปและแบบฝึกหัด

### 340.1 สรุปสิ่งที่ได้เรียนรู้ใน Part นี้

- ✅ เข้าใจว่าทำไมต้องมี Session และ `SessionMiddleware` ทำงานอย่างไรเบื้องหลัง
  ทั้งขาเข้า (`process_request`) และขาออก (`process_response`)
- ✅ รู้จัก Session Backend ทั้ง 4 แบบ (`db`, `cache`, `cached_db`,
  `signed_cookies`) พร้อมข้อดี-ข้อเสียและวิธีเลือกใช้ให้เหมาะกับสถานการณ์จริง
- ✅ ใช้งาน `request.session` ได้ครบทุกเมธอดหลัก (`.get()`, `[]=`, `.pop()`,
  `.flush()`, `.set_expiry()`) และเข้าใจข้อควรระวังเรื่อง nested object กับ
  `session.modified`
- ✅ ตั้งค่าอายุ Session ได้ตามต้องการด้วย `SESSION_COOKIE_AGE`,
  `SESSION_EXPIRE_AT_BROWSER_CLOSE`, `SESSION_SAVE_EVERY_REQUEST`
- ✅ ตั้งค่าความปลอดภัยของ Cookie ครบทุกตัว (`SESSION_COOKIE_SECURE`,
  `SESSION_COOKIE_HTTPONLY`, `SESSION_COOKIE_SAMESITE`) พร้อมเข้าใจเหตุผลของแต่ละค่า
- ✅ สร้างระบบตะกร้าสินค้าแบบ session-based ที่ใช้งานได้จริงสำหรับผู้ใช้ที่ยังไม่ login
- ✅ เข้าใจความสัมพันธ์ระหว่าง Session กับ CSRF Cookie ผ่าน `CSRF_USE_SESSIONS`
- ✅ เข้าใจความเสี่ยง Session Hijacking โดยเฉพาะ Session Fixation และวิธีป้องกันด้วย
  `cycle_key()` (พร้อมรู้ว่า `django.contrib.auth.login()` จัดการให้อัตโนมัติแล้ว)
- ✅ ใช้คำสั่ง `clearsessions` และตั้ง cron job ให้ล้าง session หมดอายุอัตโนมัติ

### 340.2 Checklist ก่อนไป Part ถัดไป

- [ ] เปิด DevTools ดู cookie `sessionid` ในเบราว์เซอร์ของคุณได้ และอธิบายได้ว่า
      `HttpOnly`, `Secure`, `SameSite` แต่ละอันหมายถึงอะไร
- [ ] เปิด Django shell แล้ว query ตาราง `django_session` ด้วย
      `Session.objects.all()` และถอดรหัสข้อมูลด้วย `.get_decoded()` สำเร็จ
- [ ] เขียนโค้ดทดสอบ `request.session.get()`, `[]=`, `.pop()`, `.flush()`,
      `.set_expiry()` ครบทุกตัวในโปรเจกต์ทดลองของตัวเอง
- [ ] ตั้งค่า `SESSION_COOKIE_AGE`, `SESSION_SAVE_EVERY_REQUEST`,
      `SESSION_EXPIRE_AT_BROWSER_CLOSE` แล้วสังเกตพฤติกรรมที่เปลี่ยนไปจริงในเบราว์เซอร์
- [ ] สร้างแอป `cart` พร้อมคลาส `Cart`, views, urls, templates ตามขั้นตอนที่ 336
      ครบถ้วน และทดสอบเพิ่ม/ลบสินค้าได้จริงโดยไม่ต้อง login
- [ ] รันคำสั่ง `python manage.py clearsessions` สำเร็จอย่างน้อย 1 ครั้ง
- [ ] อธิบายความแตกต่างระหว่าง `.flush()` กับ `.cycle_key()` ได้ด้วยคำพูดของตัวเอง

### 340.3 แบบฝึกหัดท้ายบท

**แบบฝึกหัดที่ 1 (ภาคปฏิบัติหลัก)**: ต่อยอดจากขั้นตอนที่ 336 ให้สมบูรณ์เป็นระบบ
ตะกร้าสินค้าเต็มรูปแบบ โดยต้องมีฟีเจอร์เพิ่มเติมดังนี้: (1) view สำหรับอัปเดตจำนวน
สินค้าในตะกร้าโดยไม่ต้องลบแล้วเพิ่มใหม่ (ใช้ `override_quantity=True` ที่เตรียมไว้ใน
เมธอด `add()` แล้ว), (2) แสดงจำนวนสินค้ารวมในตะกร้า (`len(cart)`) ที่ navbar ของทุก
หน้าผ่าน context processor ที่สร้างไว้ในขั้นตอนที่ 336.5, (3) ทดสอบว่าเมื่อปิดเบราว์เซอร์
ทั้งหมดแล้วเปิดใหม่ ตะกร้ายังคงอยู่หรือไม่ (ขึ้นกับค่า `SESSION_EXPIRE_AT_BROWSER_CLOSE`
ที่ตั้งไว้) แล้วอธิบายผลลัพธ์ที่เห็น

**แบบฝึกหัดที่ 2 (Session Hijacking)**: เขียน custom login view ของตัวเอง (ไม่ใช้
`LoginView` สำเร็จรูป) ที่เรียก `authenticate()` และ `django.contrib.auth.login()`
ตามปกติ จากนั้นเปิด DevTools สังเกตค่า cookie `sessionid` **ก่อน** login และ
**หลัง** login สำเร็จ แล้วบันทึกผลว่าค่าเปลี่ยนไปหรือไม่ (ควรพบว่าเปลี่ยน เพราะ Django
เรียก `cycle_key()`/`flush()` ให้อัตโนมัติตามที่เรียนในขั้นตอนที่ 338.4) — ถ้าอยาก
เห็นความแตกต่างชัดเจนขึ้น ลองปิดการเรียก `auth_login()` แล้วเซ็ต
`request.session['_auth_user_id']` ด้วยมือแทนโดยไม่เรียก `cycle_key()` แล้วสังเกตว่า
session key ไม่เปลี่ยนเลย เพื่อเข้าใจว่าทำไมเราถึงต้องใช้ฟังก์ชันมาตรฐานของ Django เสมอ
(ทำในโปรเจกต์ทดลองเท่านั้น ห้ามใช้แพทเทิร์นนี้ในระบบจริงเด็ดขาด)

**แบบฝึกหัดที่ 3 (Session Backend)**: ติดตั้ง Redis บนเครื่องของคุณ (หรือรันผ่าน
Docker ด้วยคำสั่ง `docker run -d -p 6379:6379 redis`) แล้วเปลี่ยน `SESSION_ENGINE`
ของโปรเจกต์ทดลองเป็น `cached_db` ตามขั้นตอนที่ 332.4 ทดสอบว่าระบบ login และตะกร้า
สินค้าที่สร้างไว้ในแบบฝึกหัดที่ 1 ยังทำงานถูกต้องเหมือนเดิมหรือไม่ (ควรทำงานเหมือนเดิม
ทุกประการ เพราะโค้ด view ไม่ต้องแก้เลยแม้แต่บรรทัดเดียว) แล้วลองปิด Redis ระหว่างที่
เว็บไซต์กำลังรันอยู่ สังเกตว่าเกิดอะไรขึ้นกับผู้ใช้ที่ login ค้างอยู่

**แบบฝึกหัดที่ 4 (ขั้นสูง — โอนย้ายตะกร้าจาก Session ไปสู่ User Account)**: ออกแบบและ
implement ฟังก์ชัน `merge_session_cart_to_user(request, user)` ที่ทำงานทันทีหลัง
ผู้ใช้ login สำเร็จ (เรียกใน view หรือผ่าน Django Signal `user_logged_in` ที่จะเรียน
เต็มรูปแบบใน Phase 2) โดยให้ดึงข้อมูลตะกร้าจาก `request.session['cart']` ไปบันทึกลง
Model `CartItem` ที่ผูกกับ `user` แบบถาวรในฐานข้อมูล (ต้องออกแบบ Model เอง) จากนั้น
ล้างตะกร้าใน session ทิ้งด้วย `cart.clear()` เพื่อไม่ให้ข้อมูลซ้ำซ้อนกัน 2 ที่ — โจทย์นี้
จำลองสถานการณ์จริงของระบบ e-commerce เกือบทุกระบบที่ต้อง "รวม" ตะกร้าของผู้ใช้ที่
ช้อปปิ้งไว้ก่อน login เข้ากับบัญชีถาวรของตัวเอง

### 340.4 คำถามที่พบบ่อย (FAQ)

**Q: Session กับ Cookie เป็นสิ่งเดียวกันหรือไม่?**
A: ไม่ใช่สิ่งเดียวกัน **Cookie** คือกลไกระดับ HTTP ทั่วไปที่ใช้เก็บข้อมูลเล็ก ๆ ฝั่ง
เบราว์เซอร์ ส่วน **Session** คือกลไกระดับแอปพลิเคชันที่ Django สร้างขึ้นโดยใช้ cookie
เป็นเพียง "พาหะ" ในการส่ง session key ไปกลับเท่านั้น ข้อมูลจริงของ session (ยกเว้น
backend แบบ `signed_cookies`) ไม่ได้อยู่ใน cookie เลยแม้แต่น้อย

**Q: ถ้าผู้ใช้ปิดการรับ Cookie ในเบราว์เซอร์ ระบบ Session จะยังทำงานได้ไหม?**
A: ไม่ได้ เพราะ Django ใช้ cookie เป็นวิธีมาตรฐานเพียงวิธีเดียวในการส่ง session key
ไปกลับ (Django รุ่นเก่ามาก ๆ เคยรองรับการส่ง session key ผ่าน URL parameter แต่ถูก
ตัดออกไปนานแล้วเพราะมีความเสี่ยงด้านความปลอดภัยสูงมาก — session key จะรั่วไหลผ่าน
browser history, server log, และ Referer header ได้ง่าย) ถ้าผู้ใช้ปิด cookie ทั้งหมด
ระบบที่ต้องพึ่ง session (login, ตะกร้าสินค้า) จะใช้งานไม่ได้เลย

**Q: ควรเก็บข้อมูลอะไรบ้างใน Session และไม่ควรเก็บอะไร?**
A: ควรเก็บเฉพาะข้อมูล**ชั่วคราวและมีขนาดเล็ก** ที่เกี่ยวข้องกับสถานะของผู้ใช้คนนั้น
ในช่วงเวลาสั้น ๆ เช่น ตะกร้าสินค้าก่อน login, ภาษาที่เลือกใช้งาน, ขั้นตอนของฟอร์มแบบ
multi-step ไม่ควรเก็บข้อมูลที่ควรอยู่ในฐานข้อมูลถาวรอยู่แล้ว (เช่น ประวัติการสั่งซื้อ
ทั้งหมด) หรือข้อมูลขนาดใหญ่มาก (โดยเฉพาะถ้าใช้ backend แบบ `signed_cookies` ที่มี
ข้อจำกัด ~4KB ตามขั้นตอนที่ 332.5)

**Q: ทำไม `SESSION_COOKIE_HTTPONLY` ค่าเริ่มต้นถึงเป็น `True` อยู่แล้ว แต่
`SESSION_COOKIE_SECURE` ถึงเป็น `False`?**
A: `HttpOnly` ไม่มีข้อเสียใด ๆ เลยในการเปิดใช้งานเสมอ (แทบไม่มีเหตุผลที่ JavaScript
ฝั่ง client จำเป็นต้องอ่านค่า `sessionid` โดยตรง) Django จึงตั้งเป็น `True` ให้ตั้งแต่
ต้น แต่ `Secure` ถ้าตั้งเป็น `True` ตั้งแต่เริ่มต้น จะทำให้นักพัฒนาที่พัฒนาในเครื่องผ่าน
HTTP ธรรมดา (`runserver`) **login ไม่ติดเลยแม้แต่ครั้งเดียว** เพราะ cookie จะไม่ถูกส่ง
กลับมา Django จึงปล่อยเป็น `False` ให้นักพัฒนาไปตั้งเป็น `True` เองตอน deploy จริงที่มี
HTTPS แล้ว (ตามตัวอย่างในขั้นตอนที่ 335.5)

**Q: ทำไมไม่ใช้ `localStorage` หรือ `sessionStorage` ของเบราว์เซอร์แทน Session ของ
Django ไปเลย?**
A: `localStorage`/`sessionStorage` เป็นกลไกฝั่ง**เบราว์เซอร์ล้วน ๆ** ที่ JavaScript
เข้าถึงได้โดยตรง ทำให้เสี่ยงต่อ XSS สูงกว่ามาก (ไม่มีกลไกแบบ `HttpOnly` มาป้องกัน) และ
เซิร์ฟเวอร์ไม่สามารถเชื่อถือหรือตรวจสอบข้อมูลจากฝั่ง client ได้เลยหากไม่มีการยืนยันซ้ำ
เพิ่มเติม เหมาะกับข้อมูล UI preference เล็ก ๆ น้อย ๆ ที่ไม่กระทบความปลอดภัย (เช่น
theme สี dark/light mode) มากกว่าข้อมูลที่เกี่ยวข้องกับการยืนยันตัวตนหรือธุรกรรมสำคัญ
ซึ่งควรฝากไว้กับ Session Framework ฝั่งเซิร์ฟเวอร์เสมอ

**Q: ระบบ REST API (เช่นที่จะเรียนใน Phase 6) ยังใช้ Session แบบนี้อยู่ไหม?**
A: โดยทั่วไปแล้ว REST API มักออกแบบให้เป็น **stateless เต็มรูปแบบ** และใช้กลไก
**Token-based Authentication** (เช่น JWT) แทน Session-based เพราะรองรับการสเกล
ข้ามหลายเซิร์ฟเวอร์ได้ง่ายกว่าโดยไม่ต้องกังวลเรื่อง session affinity อย่างไรก็ตาม
Django REST Framework ก็ยังรองรับ `SessionAuthentication` ได้เช่นกัน (มักใช้คู่กับ
Django Admin หรือ Browsable API) รายละเอียดเต็มรูปแบบจะอยู่ใน Phase 6 (Django
REST Framework)

---

## เตรียมตัวสำหรับ Part ถัดไป

**Part 035: Password Management, Reset และ Security** จะพาไปเจาะลึกเรื่องรหัสผ่าน
อย่างเต็มรูปแบบ ตั้งแต่ระบบ hashing รหัสผ่านของ Django (`PBKDF2`, `Argon2`),
`PasswordValidator` สำหรับบังคับความซับซ้อนของรหัสผ่าน, ระบบ "ลืมรหัสผ่าน" ที่ส่ง
ลิงก์รีเซ็ตทางอีเมลด้วย `PasswordResetView`, ไปจนถึง `get_session_auth_hash()` ที่ใช้
บังคับ logout ผู้ใช้จากทุกอุปกรณ์โดยอัตโนมัติทันทีที่เปลี่ยนรหัสผ่าน — ซึ่งเป็นหัวข้อที่
เชื่อมโยงโดยตรงกับกลไก Session ที่คุณเพิ่งเรียนจบไปใน Part นี้ โดยเฉพาะเรื่องการ
regenerate session และการจัดการ session หลายตัวของผู้ใช้คนเดียวกันพร้อมกัน

เตรียมโปรเจกต์ทดลองที่มีระบบ login พื้นฐาน (จาก Part 031) และแอป `cart` ที่สร้างไว้ใน
Part นี้ให้พร้อม เพราะ Part 035 จะใช้ทั้งสองอย่างต่อยอดทันที
