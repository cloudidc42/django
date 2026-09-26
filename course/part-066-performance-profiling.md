# Part 066: Django Performance Profiling

> **ขั้นตอนที่ 651-660 ของหลักสูตร** | Phase 8: Performance & Caching (Part แรกของ Phase นี้)
>
> ตลอด Phase 7 คุณสร้าง **Quality Gate** ที่รับประกันว่าโค้ดถูกต้อง มี test ครอบคลุม
> ปลอดภัย และสะอาดตามมาตรฐานทีมมืออาชีพ แต่คำถามที่ Quality Gate ตอบไม่ได้เลยคือ
> "โค้ดที่ถูกต้องนี้ **เร็วพอ** สำหรับผู้ใช้จริงหรือไม่?" Part นี้เปิด **Phase 8:
> Performance & Caching** ด้วยการวาง**รากฐานการวัดผล**ก่อนที่จะไปเรียนวิธี "แก้"
> ปัญหาความช้าใน Part 067-072 คุณจะได้ทบทวน `django-debug-toolbar` (ติดตั้งไปแล้วตั้งแต่
> Part 005) ให้ลึกกว่าเดิม รู้จัก `django-silk` สำหรับเก็บข้อมูล performance สะสม
> ใช้ `cProfile` และ `py-spy` วิเคราะห์โค้ด Python ระดับฟังก์ชัน เข้าใจภาพรวม APM
> สำหรับ production จริง เขียน middleware วัดเวลาเอง และปิดท้ายด้วยการตั้ง
> **Performance Budget** ที่เป็นรูปธรรม — ทั้งหมดนี้ยึดหลักการเดียว: **"Measure first,
> optimize second"** อย่าเดา อย่าเชื่อสัญชาตญาณ ให้เครื่องมือบอกความจริง

---

## สารบัญของ Part นี้

- ขั้นตอนที่ 651: ทำไม Performance สำคัญ — Perceived vs Actual Performance และผลกระทบต่อ SEO/Conversion Rate
- ขั้นตอนที่ 652: ทบทวน django-debug-toolbar เจาะลึกกว่าเดิม — SQL Panel, Timing Panel, Cache Panel
- ขั้นตอนที่ 653: `django-silk` — Profiling ที่ใกล้เคียง Production มากกว่า Debug Toolbar
- ขั้นตอนที่ 654: Python Profiling พื้นฐานด้วย `cProfile`
- ขั้นตอนที่ 655: `py-spy` — Sampling Profiler สำหรับ Process ที่กำลังรันอยู่โดยไม่แก้โค้ด
- ขั้นตอนที่ 656: วิธีค้นหา N+1 Query Problem อย่างเป็นระบบ (เกริ่นก่อนเข้า Part 067 เต็มรูปแบบ)
- ขั้นตอนที่ 657: ภาพรวม APM Tools — Sentry Performance, New Relic, Datadog สำหรับ Monitor Production จริง
- ขั้นตอนที่ 658: วัดเวลาแต่ละขั้นตอนของ Request/Response Cycle ด้วย Custom Middleware
- ขั้นตอนที่ 659: การตั้ง Performance Budget/SLO
- ขั้นตอนที่ 660: สรุปและแบบฝึกหัด — ใช้เครื่องมือทั้งหมด Profile หน้า Blog List หา Bottleneck จริง

---

## ขั้นตอนที่ 651: ทำไม Performance สำคัญ — Perceived Performance vs Actual Performance และผลกระทบต่อ SEO/Conversion Rate

### 651.1 Actual Performance คืออะไร

**Actual Performance** คือเวลาที่วัดได้จริงด้วยนาฬิกา (wall-clock time) ตั้งแต่ผู้ใช้
กดคลิกจนได้รับผลลัพธ์ครบถ้วน แบ่งเป็นช่วงเวลาที่วัดได้แยกกันชัดเจน:

```
Browser                                          Server (Django)
   │                                                    │
   │──── 1. DNS Lookup ────────────────────────────────>│
   │──── 2. TCP + TLS Handshake ────────────────────────>│
   │──── 3. HTTP Request ส่งออกไป ───────────────────────>│
   │                                                    │ 4. Django ประมวลผล
   │                                                    │    (Middleware → View →
   │                                                    │     ORM Query → Template
   │                                                    │     Render) ← ส่วนนี้คือ
   │                                                    │    "TTFB" (Time To First Byte)
   │<──── 5. Response กลับมา (HTML/JSON) ─────────────────│
   │──── 6. Browser Parse HTML/CSS/JS ────────────────────
   │──── 7. Browser Render + JavaScript ทำงาน ────────────
   │
   ▼ ผู้ใช้เห็นหน้าเว็บสมบูรณ์
```

**Django ควบคุมได้เต็มที่แค่ขั้นตอนที่ 4** เท่านั้น (เวลาที่ server ใช้ประมวลผล
request หนึ่งครั้ง) ส่วนขั้นตอน 1-3 ขึ้นกับเครือข่าย/CDN และขั้นตอน 6-7 ขึ้นกับ
ฝั่ง frontend — Phase 8 ทั้งหมดของหลักสูตรนี้เจาะจงไปที่การทำให้ **ขั้นตอนที่ 4
เร็วที่สุดเท่าที่จะทำได้** เพราะนั่นคืองานของ Backend Developer

### 651.2 Perceived Performance คืออะไร และทำไมสำคัญพอ ๆ กับ Actual Performance

**Perceived Performance** คือ "ความรู้สึก" ของผู้ใช้ว่าเว็บเร็วหรือช้า ซึ่งไม่ตรงกับ
ตัวเลขจริงเสมอไป ตัวอย่างเช่น หน้าที่ใช้เวลา 3 วินาทีแต่แสดง **skeleton screen**
(โครงหน้าจอเปล่า ๆ) ทันทีอาจรู้สึกเร็วกว่าหน้าที่ใช้เวลาแค่ 1.5 วินาทีแต่จอขาวเปล่า
จนกว่าทุกอย่างจะโหลดเสร็จ

| แนวคิด | วัดด้วยอะไร | ตัวอย่างเทคนิคที่ปรับปรุงได้ |
|---|---|---|
| **Actual Performance** | นาฬิกาจับเวลาจริง (ms) | Query optimization, caching, indexing (Phase 8 ทั้งหมด) |
| **Perceived Performance** | ความรู้สึกของผู้ใช้ (subjective) | Progressive rendering, skeleton screen, optimistic UI, loading spinner ที่เหมาะสม |

หลักสูตรนี้เน้น **Actual Performance** เป็นหลัก เพราะเป็นสิ่งที่ Backend/Django
ควบคุมได้โดยตรงและวัดผลได้เป็นตัวเลข แต่ควรรู้ไว้ว่าทั้งสองแนวคิดต้องทำงานร่วมกัน
ในโปรเจกต์จริง — server เร็วแต่ frontend design แย่ก็ยังทำให้ผู้ใช้รู้สึกว่าเว็บช้าได้

### 651.3 ตัวเลขวิจัยที่พิสูจน์ผลกระทบของ Performance ต่อธุรกิจจริง

Performance ไม่ใช่แค่เรื่อง "เทคนิคเก๋ ๆ" แต่ส่งผลต่อรายได้และการเติบโตธุรกิจโดยตรง
มีงานวิจัยและข้อมูลจากบริษัทระดับโลกจำนวนมากที่ยืนยันเรื่องนี้:

| แหล่งข้อมูล | ผลกระทบที่พบ |
|---|---|
| **Amazon** | ทุก ๆ 100ms ที่หน้าเว็บช้าลง ยอดขายลดลงประมาณ 1% |
| **Google** | หน้าเว็บที่โหลดช้าลง 500ms ทำให้ traffic การค้นหาลดลงประมาณ 20% |
| **Google/SOASTA Research** | ความน่าจะเป็นที่ผู้ใช้จะ bounce (ออกจากเว็บทันที) เพิ่มขึ้น 32% เมื่อเวลาโหลดหน้าเว็บเพิ่มจาก 1 วินาทีเป็น 3 วินาที และเพิ่มขึ้นถึง 90% เมื่อเพิ่มเป็น 5 วินาที |
| **Akamai/Aberdeen Group** | ทุก ๆ 1 วินาทีที่หน้าเว็บช้าลง อัตรา conversion (การซื้อ/สมัครสมาชิก) ลดลงประมาณ 7% |
| **Walmart** | ทุก ๆ 100ms ที่เร็วขึ้น conversion rate เพิ่มขึ้นสูงสุดถึง 1% |
| **BBC** | เสียผู้ใช้เพิ่มอีก 10% ทุก ๆ 1 วินาทีที่หน้าเว็บใช้เวลาโหลดนานขึ้น |

**บทเรียนสำคัญ**: ตัวเลข millisecond ที่ดูเหมือนเล็กน้อยในสายตานักพัฒนา แปลเป็น
**เงินจริง** และ **ผู้ใช้จริง** ที่หายไปในสเกลใหญ่เสมอ นี่คือเหตุผลที่บริษัทระดับโลก
ลงทุนกับทีม Performance Engineering โดยเฉพาะ

### 651.4 Performance กับ SEO: Core Web Vitals

ตั้งแต่ปี 2021 Google ใช้ **Core Web Vitals** เป็นหนึ่งใน ranking factor อย่างเป็น
ทางการ (เรียกว่า "Page Experience Update") หมายความว่าเว็บที่ช้าจะ**ถูกจัดอันดับต่ำ
กว่าในผลการค้นหา** แม้เนื้อหาจะดีเท่ากันก็ตาม

| Metric | วัดอะไร | เกณฑ์ดี (Good) | เกณฑ์ต้องปรับปรุง | เกณฑ์แย่ (Poor) |
|---|---|---|---|---|
| **LCP** (Largest Contentful Paint) | เวลาที่ element ใหญ่ที่สุดในหน้าจอ render เสร็จ | ≤ 2.5 วินาที | 2.5-4.0 วินาที | > 4.0 วินาที |
| **INP** (Interaction to Next Paint) | เวลาตอบสนองหลังผู้ใช้โต้ตอบ (คลิก/พิมพ์) — แทนที่ FID ตั้งแต่มี.ค. 2024 | ≤ 200ms | 200-500ms | > 500ms |
| **CLS** (Cumulative Layout Shift) | ความเลื่อนของ layout ที่ไม่คาดคิดระหว่างโหลด | ≤ 0.1 | 0.1-0.25 | > 0.25 |

Django ส่งผลโดยตรงต่อ **LCP** ผ่านเวลา **TTFB (Time To First Byte)** — ถ้า Django
ใช้เวลานานในการประมวลผล view ก่อนส่ง byte แรกกลับมา ทุก metric ข้างต้นก็จะช้าตามไป
โดยอัตโนมัติ เพราะ browser ยังเริ่ม render อะไรไม่ได้เลยจนกว่าจะได้รับ response แรก

Google แนะนำ TTFB ที่ดีไว้ที่ **≤ 800ms** (และยิ่งต่ำยิ่งดี ระดับมืออาชีพมักตั้งเป้า
ไว้ที่ 100-200ms สำหรับหน้าที่ cache แล้ว) — นี่คือตัวเลขที่ Phase 8 ทั้งหมดของ
หลักสูตรนี้จะพาคุณไล่ตามให้ถึง

### 651.5 หลักการทองคำ: "Measure, Don't Guess"

Donald Knuth นักวิทยาศาสตร์คอมพิวเตอร์ชื่อดังเคยกล่าวไว้ว่า:

> "Premature optimization is the root of all evil"
> (การ optimize ก่อนเวลาอันควรคือรากเหง้าของความชั่วร้ายทั้งปวง)

ความหมายไม่ใช่ "ห้าม optimize" แต่คือ **ห้าม optimize โดยไม่มีข้อมูล** นักพัฒนา
มือใหม่มักเดาว่าจุดไหนช้า (เช่น "ORM ต้องช้าแน่ ๆ" หรือ "Python วน loop ช้า")
แล้วเสียเวลาไป optimize จุดที่ไม่ได้เป็นปัญหาจริง ในขณะที่จุดที่ทำให้หน้าเว็บช้าจริง
อาจเป็นเรื่องคาดไม่ถึง เช่น การเรียก API ภายนอกที่ timeout นาน หรือ query เดียว
ที่ไม่มี index

**Part นี้ทั้ง Part จึงไม่มีการ "แก้ปัญหา" performance เลยแม้แต่บรรทัดเดียว** —
มีแต่การ**วัด**และ**ค้นหา**ว่าปัญหาอยู่ตรงไหน เพราะการแก้ปัญหาจริง (query
optimization, caching, indexing) คืองานของ Part 067-072 ที่จะตามมา หลักการที่ต้อง
จำให้ขึ้นใจตลอด Phase 8 คือ:

```
1. Measure  (วัดค่าปัจจุบันด้วยเครื่องมือ — Part 066)
2. Identify (หาว่า bottleneck คืออะไร — Part 066)
3. Fix      (แก้เฉพาะจุดที่วัดแล้วว่าเป็นปัญหาจริง — Part 067-071)
4. Measure again (วัดซ้ำเพื่อยืนยันว่าดีขึ้นจริง ไม่ใช่แค่ความรู้สึก)
5. Set a budget (ตั้งงบประมาณเพื่อป้องกันไม่ให้แย่ลงอีกในอนาคต — Part 066 ขั้นตอนที่ 659)
```

---

## ขั้นตอนที่ 652: ทบทวน django-debug-toolbar เจาะลึกกว่าเดิม — SQL Panel, Timing Panel, Cache Panel

### 652.1 ทบทวนภาพรวม Panel ทั้งหมดของ Debug Toolbar

ใน Part 005 คุณติดตั้ง `django-debug-toolbar` และใน Part 065 ขั้นตอนที่ 645
คุณทบทวนการตั้งค่าและใช้งานเบื้องต้นไปแล้ว ในขั้นตอนนี้เราจะ**เจาะลึก 3 panel ที่
สำคัญที่สุดสำหรับงาน performance** โดยเฉพาะ ทบทวน panel เริ่มต้นทั้งหมดที่มากับ
Debug Toolbar:

| Panel | ข้อมูลที่แสดง |
|---|---|
| Versions | เวอร์ชัน Django, Python, package ที่ติดตั้ง |
| Timer | เวลารวมของ request (browser timing + CPU time) |
| Settings | ค่า settings ทั้งหมดของโปรเจกต์ |
| Headers | HTTP request/response headers |
| Request | GET/POST/COOKIES/session data |
| **SQL** | รายการ query ทั้งหมด พร้อมเวลาที่ใช้ (เจาะลึกในขั้นตอนที่ 652.2) |
| Static files | ไฟล์ static ที่ถูกใช้ |
| Templates | template ที่ render และ context ที่ส่งเข้าไป |
| **Cache** | การเรียก cache framework (เจาะลึกในขั้นตอนที่ 652.4) |
| Signals | signal ทั้งหมดที่ยิงระหว่าง request |
| Redirects | ติดตาม HTTP redirect chain |
| Profiling | เวลาที่แต่ละฟังก์ชันใช้ (ไม่เปิดโดยค่าเริ่มต้น เพราะทำให้ช้าลงมาก) |

### 652.2 SQL Panel เจาะลึก: อ่านอย่างไรให้เจอปัญหาจริง

SQL Panel คือ panel ที่นักพัฒนา Django มืออาชีพเปิดดูบ่อยที่สุด เพราะปัญหา
performance ของ Django ส่วนใหญ่ (>80% ในโปรเจกต์ทั่วไป) มาจากการจัดการฐานข้อมูล
ที่ไม่เหมาะสม ไม่ใช่จาก Python logic เอง

ตัวอย่างโค้ดที่มีปัญหา (จงใจใส่ query ซ้ำเพื่อสาธิต):

```python
# blog/views.py
from django.shortcuts import render
from blog.models import Post


def post_list(request):
    posts = Post.objects.filter(is_published=True)[:20]
    return render(request, "blog/post_list.html", {"posts": posts})
```

```html
<!-- blog/templates/blog/post_list.html -->
{% for post in posts %}
    <article>
        <h2>{{ post.title }}</h2>
        <p>โดย {{ post.author.get_full_name }} | หมวดหมู่: {{ post.category.name }}</p>
        <p>ความคิดเห็น {{ post.comments.count }} รายการ</p>
    </article>
{% endfor %}
```

เมื่อเปิด SQL Panel ในหน้านี้ จะเห็นข้อมูลที่สำคัญ 4 อย่าง:

1. **จำนวน query รวม** แสดงเป็นตัวเลขใหญ่บนแท็บ (เช่น "SQL queries (61)")
2. **แถวสีเดียวกัน (highlight)**: Debug Toolbar จัดกลุ่ม query ที่มีโครงสร้างเหมือนกัน
   (ต่างกันแค่ค่า parameter) ด้วยแถบสีเดียวกันอัตโนมัติ — ถ้าเห็นแถบสีซ้ำ ๆ กันเป็น
   จำนวนมาก **นั่นคือสัญญาณ N+1 Query ชัดเจนที่สุด**
3. **เวลาแต่ละ query**: คอลัมน์ "Time" แสดงเวลาเป็น ms ของแต่ละ query แยกกัน
   ช่วยหาว่า query ไหนช้าผิดปกติ (ไม่ใช่แค่จำนวนเยอะ)
4. **ปุ่ม "Explain"**: กดดู query execution plan ของฐานข้อมูลได้ทันทีโดยไม่ต้องไปเปิด
   `psql` หรือเขียน `EXPLAIN ANALYZE` เอง (รายละเอียดเรื่อง execution plan จะเจาะลึก
   เต็มรูปแบบใน Part 070: Database Indexing)

ตัวอย่างที่คาดว่าจะเห็นจากโค้ดด้านบน (20 บทความ):

```
SQL queries (61)
  1 query: SELECT * FROM blog_post WHERE is_published = true LIMIT 20     [2.1ms]
  20 queries (สีเดียวกัน): SELECT * FROM auth_user WHERE id = %s          [0.4ms x 20]
  20 queries (สีเดียวกัน): SELECT * FROM blog_category WHERE id = %s     [0.3ms x 20]
  20 queries (สีเดียวกัน): SELECT COUNT(*) FROM blog_comment WHERE post_id = %s [0.5ms x 20]
```

รวม 61 query สำหรับหน้าที่ควรใช้แค่ **1 query** — นี่คือปัญหา N+1 แบบซ้อนกันถึง
3 ชั้น (author, category, comments count) ซึ่งจะเจาะลึกวิธีแก้เต็มรูปแบบใน Part 067
แต่ในขั้นตอนนี้ สิ่งสำคัญคือ**เห็นและนับปัญหาได้ก่อน**

### 652.3 Timing Panel: แยกเวลาแต่ละช่วงของ Request

Timing Panel แสดงกราฟแท่งแนวนอนแบ่งเวลาทั้งหมดของ request ออกเป็นช่วง ๆ:

```
Total: 245ms
├── Python (CPU) time:  180ms  ████████████████████████████░░░░░░░░
├── SQL time:             52ms  █████████░░░░░░░░░░░░░░░░░░░░░░░░░░░
└── Browser processing:   13ms  ██░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░
```

การอ่าน Timing Panel ช่วยตอบคำถามสำคัญตั้งแต่แรกว่า **ปัญหาอยู่ที่ Python logic
หรืออยู่ที่ฐานข้อมูล** ถ้า SQL time สูงเทียบกับ Total → ไปโฟกัสที่ SQL Panel และ
Part 067 (Query Optimization) แต่ถ้า Python time สูงและ SQL time ต่ำ → ปัญหาน่าจะ
อยู่ที่ logic การประมวลผลในฝั่ง Python (เช่น loop ที่หนัก, การคำนวณซับซ้อน,
การเรียก external API) ซึ่งต้องใช้ `cProfile` (ขั้นตอนที่ 654) เจาะลึกต่อ

### 652.4 Cache Panel: ตรวจสอบ Cache Hit/Miss แบบเรียลไทม์

Cache Panel เป็น panel ที่มักถูกมองข้าม เพราะช่วง Part 001-064 โปรเจกต์ยังไม่ได้ใช้
caching อย่างจริงจัง (จะเรียนเต็มรูปแบบใน Part 068-069) แต่ panel นี้พร้อมใช้งาน
ตั้งแต่ตอนนี้ เพื่อให้คุณคุ้นเคยไว้ก่อน

ตั้งค่า cache backend สำหรับ development ก่อน (ใช้ local memory cache ที่ไม่ต้อง
ติดตั้งอะไรเพิ่ม):

```python
# config/settings/dev.py
CACHES = {
    "default": {
        "BACKEND": "django.core.cache.backends.locmem.LocMemCache",
        "LOCATION": "unique-snowflake",
    }
}
```

ตัวอย่างโค้ดที่เรียก cache framework (ทดลองก่อนเรียนเต็มรูปแบบใน Part 068):

```python
# blog/views.py
from django.core.cache import cache
from django.shortcuts import render
from blog.models import Post


def homepage_stats(request):
    total_posts = cache.get("total_posts_count")
    if total_posts is None:
        total_posts = Post.objects.filter(is_published=True).count()
        cache.set("total_posts_count", total_posts, timeout=300)

    return render(request, "blog/home.html", {"total_posts": total_posts})
```

เปิด Cache Panel จะเห็นรายละเอียดทุกครั้งที่โค้ดเรียก `cache.get()`/`cache.set()`:

| Call | Time | Args | Hit/Miss |
|---|---|---|---|
| `get` | 0.02ms | `total_posts_count` | **Miss** (ครั้งแรก) |
| `set` | 0.05ms | `total_posts_count`, timeout=300 | - |
| `get` | 0.01ms | `total_posts_count` | **Hit** (รีเฟรชครั้งถัดไป) |

รวมถึงสรุปด้านบน panel: **"Total calls: 3, Total time: 0.08ms, Hits: 1, Misses: 1,
Sets: 1"** — ตัวเลข **Hit Rate** นี้จะกลายเป็นตัวชี้วัดสำคัญมากเมื่อไปถึง Part 068-069
ที่ต้องตัดสินใจว่า cache strategy ที่ออกแบบไว้ได้ผลจริงหรือไม่

### 652.5 เปิดใช้งาน Profiling Panel (ปิดโดยค่าเริ่มต้น)

Debug Toolbar มี Profiling Panel ในตัวที่แสดงเวลาแต่ละฟังก์ชัน คล้าย `cProfile`
แต่ปิดไว้โดยค่าเริ่มต้นเพราะทำให้ request ช้าลงอย่างเห็นได้ชัด (instrumentation
overhead) เปิดใช้เฉพาะตอนต้องการเจาะลึกจริง ๆ:

```python
# config/settings/dev.py
DEBUG_TOOLBAR_PANELS = [
    "debug_toolbar.panels.versions.VersionsPanel",
    "debug_toolbar.panels.timer.TimerPanel",
    "debug_toolbar.panels.settings.SettingsPanel",
    "debug_toolbar.panels.headers.HeadersPanel",
    "debug_toolbar.panels.request.RequestPanel",
    "debug_toolbar.panels.sql.SQLPanel",
    "debug_toolbar.panels.staticfiles.StaticFilesPanel",
    "debug_toolbar.panels.templates.TemplatesPanel",
    "debug_toolbar.panels.cache.CachePanel",
    "debug_toolbar.panels.signals.SignalsPanel",
    "debug_toolbar.panels.redirects.RedirectsPanel",
    "debug_toolbar.panels.profiling.ProfilingPanel",  # เพิ่มเข้ามาเฉพาะตอนต้องการ
]
```

### 652.6 ข้อจำกัดของ Debug Toolbar ที่ทำให้ต้องมีเครื่องมือเพิ่ม

Debug Toolbar ยอดเยี่ยมสำหรับการดู request เดียว ณ ขณะนั้น แต่มีข้อจำกัด 3 อย่างที่
ทำให้ไม่พอสำหรับงาน performance ระดับมืออาชีพ:

1. **ไม่เก็บประวัติ**: ปิดหน้าเว็บแล้วข้อมูลหายทันที ไม่สามารถเปรียบเทียบ "เมื่อวาน
   หน้านี้ใช้เวลาเท่าไหร่" ได้
2. **ใช้ได้เฉพาะตอน `DEBUG=True`**: ไม่สามารถใช้ตรวจสอบ API endpoint ที่เรียกจาก
   mobile app หรือ background job ที่ไม่มี browser มาเรนเดอร์ toolbar
3. **ดูได้ทีละ request**: ไม่สามารถหาคำตอบเช่น "endpoint ไหนที่ถูกเรียกบ่อยที่สุดและ
   ช้าที่สุดโดยรวมตลอดทั้งวัน" ได้เลย

ข้อจำกัดทั้ง 3 ข้อนี้คือเหตุผลที่ทีมมืออาชีพเสริมด้วย **`django-silk`** ซึ่งเราจะ
เรียนรู้ในขั้นตอนถัดไป

---

## ขั้นตอนที่ 653: `django-silk` — Profiling ที่ใกล้เคียง Production มากกว่า Debug Toolbar

### 653.1 django-silk คืออะไร และต่างจาก Debug Toolbar อย่างไร

**[django-silk](https://github.com/jazzband/django-silk)** คือ package ที่ดูแลโดย
Jazzband ทำหน้าที่คล้าย Debug Toolbar แต่ **บันทึกข้อมูลทุก request ลงฐานข้อมูล
จริง** ทำให้ดูย้อนหลังได้ เปรียบเทียบข้าม request ได้ และเรียงลำดับหา request ที่ช้า
ที่สุดในช่วงเวลาที่ผ่านมาได้ทั้งหมด

| คุณสมบัติ | django-debug-toolbar | django-silk |
|---|---|---|
| เก็บข้อมูลที่ไหน | หน่วยความจำชั่วคราว (ต่อ request) | ฐานข้อมูลจริง (ตาราง `silk_request`) |
| ดูย้อนหลังได้ | ❌ (หายเมื่อปิดหน้า) | ✅ ดูประวัติทุก request ที่ผ่านมาได้ |
| เปรียบเทียบหลาย request | ❌ | ✅ เรียงลำดับตามเวลา/จำนวน query ได้ |
| ใช้กับ API/non-HTML view | ยากกว่า (toolbar ฝังใน HTML) | ✅ ใช้ได้ปกติ (เก็บผ่าน middleware) |
| Overhead | ต่ำ-ปานกลาง | ปานกลาง-สูงกว่า (เขียนลง DB ทุก request) |
| ใช้ใน production ได้ไหม | ไม่ควร (ต้อง `DEBUG=True`) | ได้ในระดับ staging/production แบบจำกัด (มี sampling) |

### 653.2 ติดตั้งและตั้งค่า django-silk

```bash
pip install django-silk
```

เพิ่มเข้า `requirements/dev.txt`:

```
# requirements/dev.txt
django-silk==5.3.1
```

```python
# config/settings/dev.py
INSTALLED_APPS += [
    "silk",
]

MIDDLEWARE = [
    "silk.middleware.SilkyMiddleware",  # ควรอยู่ต้น ๆ ของ MIDDLEWARE เพื่อจับเวลาทั้งหมด
    *MIDDLEWARE,
]

# ตั้งค่าพื้นฐานของ Silk
SILKY_PYTHON_PROFILER = True          # เปิด cProfile อัตโนมัติสำหรับทุก request
SILKY_PYTHON_PROFILER_BINARY = False  # เก็บผลลัพธ์เป็นข้อความอ่านง่าย ไม่ใช่ binary
SILKY_MAX_RECORDED_REQUESTS = 10_000  # เก็บสูงสุดกี่ request ก่อนเริ่มลบของเก่า
SILKY_AUTHENTICATION = True           # บังคับ login ก่อนเข้าดู /silk/
SILKY_AUTHORISATION = True            # ตรวจสอบสิทธิ์เพิ่มเติมด้วย has_permission ด้านล่าง
```

```python
# config/urls.py
from django.conf import settings

urlpatterns = [
    # ... urlpatterns เดิม ...
]

if settings.DEBUG:
    urlpatterns += [
        path("silk/", include("silk.urls", namespace="silk")),
    ]
```

รัน migration ให้ Silk สร้างตารางเก็บข้อมูล:

```bash
python manage.py migrate silk
```

**กฎเหล็กด้านความปลอดภัย**: เช่นเดียวกับ Debug Toolbar, Silk เก็บข้อมูล SQL query
และ request/response body เต็มรูปแบบ ซึ่งอาจมีข้อมูลอ่อนไหว (password ใน POST body,
token ใน header) — ต้องเปิดเฉพาะใน `DEBUG=True` หรือ staging environment ที่จำกัด
สิทธิ์เข้าถึงเท่านั้น และ `SILKY_AUTHENTICATION`/`SILKY_AUTHORISATION` ต้องเปิดเสมอ
เพื่อบังคับให้เฉพาะ staff เท่านั้นที่เข้าดูได้:

```python
# config/settings/staging.py (ถ้าต้องการเปิด Silk บน staging แบบจำกัดสิทธิ์)
def silky_permissions(user):
    return user.is_staff

SILKY_PERMISSIONS = silky_permissions
```

### 653.3 ใช้งาน Silk UI ดูประวัติ Request

เข้า `http://localhost:8000/silk/` จะเห็นตารางรายการ request ทั้งหมดที่เคยเกิดขึ้น
พร้อมคอลัมน์ที่สำคัญ:

| Path | Method | Status | Time (ms) | Num SQL Queries | Time on SQL (ms) |
|---|---|---|---|---|---|
| `/blog/` | GET | 200 | 245.3 | 61 | 52.1 |
| `/blog/post/django-tips/` | GET | 200 | 38.2 | 4 | 6.5 |
| `/api/posts/` | GET | 200 | 512.7 | 143 | 480.2 |

สังเกตว่า **สามารถกดหัวคอลัมน์เพื่อเรียงลำดับ** หาได้ทันทีว่า endpoint ไหนช้าที่สุด
โดยรวม (ในตัวอย่างคือ `/api/posts/` ที่มี query สูงถึง 143 ครั้ง!) นี่คือความสามารถ
ที่ Debug Toolbar ทำไม่ได้เลยเพราะดูได้ทีละ request เท่านั้น กดเข้าไปในแต่ละแถวจะ
เห็นรายละเอียดเดียวกับ SQL Panel ของ Debug Toolbar (query ทีละบรรทัด พร้อม stack
trace ว่า query นั้นถูกเรียกจากบรรทัดไหนของโค้ด)

### 653.4 Silk Profiling ด้วย Decorator และ Context Manager

นอกจากติดตาม request อัตโนมัติ Silk ยังมี decorator `@silk_profile` สำหรับ profile
ฟังก์ชันหรือ code block เฉพาะจุดที่ไม่ใช่ทั้ง request เช่น background task หรือ
management command:

```python
# blog/services.py
from silk.profiling.profiler import silk_profile


@silk_profile(name="Generate Monthly Report")
def generate_monthly_report(year: int, month: int) -> dict:
    """สร้างรายงานสรุปยอดบทความและยอดวิวประจำเดือน"""
    from blog.models import Post

    posts = Post.objects.filter(
        published_at__year=year, published_at__month=month
    )
    return {
        "total_posts": posts.count(),
        "total_views": sum(p.view_count for p in posts),
    }
```

หรือใช้เป็น context manager กับเฉพาะบางส่วนของฟังก์ชันที่ต้องการ:

```python
from silk.profiling.profiler import silk_profile


def complex_view_logic(request):
    # ส่วนที่ไม่ต้องการวัด
    user = request.user

    with silk_profile(name="Heavy Calculation Block"):
        result = _run_expensive_calculation(user)

    return result
```

ผลลัพธ์จะปรากฏใน Silk UI ภายใต้แท็บ **"Profiling"** แยกต่างหากจากรายการ request
ปกติ พร้อมกราฟเวลาสะสมของแต่ละครั้งที่ฟังก์ชันนี้ถูกเรียก

### 653.5 การจัดการข้อมูลเก่า (Data Retention)

เนื่องจาก Silk เขียนข้อมูลลงฐานข้อมูลจริงทุก request ตารางจะโตขึ้นเรื่อย ๆ ถ้าไม่มี
การล้างข้อมูลเก่า Silk มี management command สำหรับจัดการเรื่องนี้:

```bash
# ลบข้อมูล request ที่เก่ากว่า 7 วัน
python manage.py silk_clear_request_log

# กำหนดจำนวนวันเอง (แก้ค่า SILKY_MAX_RECORDED_REQUESTS_CHECK_PERCENT
# หรือรันคำสั่งนี้ผ่าน cron/Celery Beat เป็นประจำ)
```

ทีมมืออาชีพมักตั้ง **cron job รายวัน** หรือ **Celery Beat task** (จะเรียนเรื่อง
Celery ใน Phase ถัดไป) ให้รันคำสั่งนี้อัตโนมัติ เพื่อไม่ให้ตาราง `silk_request` และ
`silk_sqlquery` โตจนกระทบ performance ของฐานข้อมูลเอง — เป็นเรื่องน่าขันที่เครื่องมือ
วัด performance กลับทำให้ performance แย่ลงถ้าไม่จัดการดี ๆ

### 653.6 ตัวอย่าง: ใช้ Silk วิเคราะห์ปัญหา N+1 ในหน้า Blog List

ย้อนกลับไปที่ตัวอย่างในขั้นตอนที่ 652.2 (หน้า `post_list` ที่มี 61 query) ลองเข้า
`/silk/` แล้วกดดูรายละเอียด request นั้น จะเห็นข้อมูลเพิ่มเติมที่ Debug Toolbar
ให้ไม่ได้:

- **กราฟเปรียบเทียบ** กับค่าเฉลี่ยของ endpoint เดียวกันใน 24 ชั่วโมงที่ผ่านมา
  (เช่น "endpoint นี้เฉลี่ย 58 query, ช้าลงเรื่อย ๆ ทุกวันเพราะจำนวนบทความเพิ่มขึ้น" —
  สัญญาณเตือนว่า N+1 จะยิ่งแย่ลงเมื่อข้อมูลโต)
- **Stack trace ของแต่ละ query**: บอกชัดเจนว่า query `SELECT * FROM auth_user
  WHERE id = %s` ถูกเรียกจากบรรทัดไหนใน template (ผ่าน `post.author.get_full_name`)
  ทำให้หาต้นตอได้เร็วกว่าการเดา

ข้อมูลนี้คือสิ่งที่จะนำไปใช้แก้ไขจริงด้วย `select_related()`/`prefetch_related()`
ใน **Part 067: Query Optimization** — ในขั้นตอนนี้เป้าหมายคือ**ค้นหาและวัดปริมาณ
ปัญหาให้ชัดเจนก่อน**

---

## ขั้นตอนที่ 654: Python Profiling พื้นฐานด้วย `cProfile`

### 654.1 cProfile คืออะไร (Deterministic Profiler)

**`cProfile`** คือ profiler ที่มากับ Python มาตรฐาน (standard library) ไม่ต้อง
ติดตั้งอะไรเพิ่ม ทำงานแบบ **deterministic profiling** คือ**ดักจับทุกการเรียกฟังก์ชัน
จริง ๆ** (instrumentation) แล้วนับเวลาที่ใช้ในแต่ละฟังก์ชันอย่างแม่นยำ 100%
ต่างจาก SQL Panel ที่เห็นแค่ query, `cProfile` เห็นลึกถึงระดับ**ฟังก์ชัน Python
ล้วน ๆ** แม้จะไม่มี query เกี่ยวข้องเลยก็ตาม (เช่น loop คำนวณหนัก, string
processing, การ serialize JSON ขนาดใหญ่)

### 654.2 ใช้ cProfile กับฟังก์ชันเดี่ยว ๆ อย่างง่ายที่สุด

```python
import cProfile
import pstats

from blog.services import generate_monthly_report

profiler = cProfile.Profile()
profiler.enable()

generate_monthly_report(2026, 9)

profiler.disable()
stats = pstats.Stats(profiler)
stats.sort_stats("cumulative")
stats.print_stats(15)  # แสดง 15 อันดับแรกที่ใช้เวลารวมมากที่สุด
```

### 654.3 สร้าง Management Command สำหรับ Profile โค้ดที่ต้องการ

วิธีที่สะดวกกว่าการเขียนสคริปต์แยกทุกครั้งคือสร้าง management command ที่ profile
คำสั่งใดก็ได้:

```python
# blog/management/commands/profile_report.py
import cProfile
import pstats
from io import StringIO

from django.core.management.base import BaseCommand

from blog.services import generate_monthly_report


class Command(BaseCommand):
    help = "Profile ฟังก์ชัน generate_monthly_report ด้วย cProfile"

    def add_arguments(self, parser):
        parser.add_argument("--year", type=int, default=2026)
        parser.add_argument("--month", type=int, default=9)
        parser.add_argument(
            "--output", type=str, default=None,
            help="บันทึกผลลัพธ์เป็นไฟล์ .prof สำหรับเปิดด้วย snakeviz",
        )

    def handle(self, *args, **options):
        profiler = cProfile.Profile()
        profiler.enable()

        result = generate_monthly_report(options["year"], options["month"])

        profiler.disable()

        if options["output"]:
            profiler.dump_stats(options["output"])
            self.stdout.write(self.style.SUCCESS(f"บันทึกผลลัพธ์ที่ {options['output']}"))

        stream = StringIO()
        stats = pstats.Stats(profiler, stream=stream)
        stats.sort_stats("cumulative")
        stats.print_stats(20)
        self.stdout.write(stream.getvalue())
        self.stdout.write(self.style.SUCCESS(f"ผลลัพธ์: {result}"))
```

รันคำสั่ง:

```bash
python manage.py profile_report --year 2026 --month 9 --output report.prof
```

### 654.4 อ่านผลลัพธ์ pstats: คอลัมน์แต่ละตัวหมายถึงอะไร

ตัวอย่างผลลัพธ์:

```
         184261 function calls (181022 primitive calls) in 0.842 seconds

   Ordered by: cumulative time
   List reduced from 1204 to 20 due to restriction <20>

   ncalls  tottime  percall  cumtime  percall filename:lineno(function)
        1    0.001    0.001    0.842    0.842 services.py:8(generate_monthly_report)
      156    0.023    0.000    0.612    0.004 query.py:412(filter)
    12480    0.089    0.000    0.412    0.000 base.py:340(__init__)
      156    0.301    0.002    0.301    0.002 {method 'execute' of 'sqlite3.Cursor'}
    12480    0.156    0.000    0.156    0.000 fields.py:88(to_python)
```

| คอลัมน์ | ความหมาย |
|---|---|
| `ncalls` | จำนวนครั้งที่ฟังก์ชันนี้ถูกเรียก |
| `tottime` | เวลารวมที่ใช้**ในฟังก์ชันนี้เองเท่านั้น** (ไม่รวมฟังก์ชันที่มันเรียกต่อ) |
| `percall` (ตัวแรก) | `tottime` หารด้วย `ncalls` |
| `cumtime` | เวลารวมที่ใช้ในฟังก์ชันนี้**รวมถึงฟังก์ชันย่อยที่มันเรียกทั้งหมด** |
| `percall` (ตัวที่สอง) | `cumtime` หารด้วย `ncalls` |

**หลักการอ่าน**: เรียงตาม `cumulative` (ค่าเริ่มต้น) เพื่อหา "จุดเริ่มต้น" ของปัญหา
ก่อน (ฟังก์ชันระดับบนสุดที่ห่อฟังก์ชันอื่นไว้) แล้วค่อยไล่ดู `tottime` เพื่อหาว่า
**ฟังก์ชันไหนที่กินเวลาในตัวมันเองมากที่สุดจริง ๆ** (ไม่ใช่แค่เพราะเรียกฟังก์ชันอื่น
ที่ช้าต่อ) จากตัวอย่างข้างต้น จะเห็นว่า `{method 'execute' of 'sqlite3.Cursor'}`
มี `tottime` สูงสุด (0.301s) หมายความว่าเวลาส่วนใหญ่หมดไปกับการรัน SQL query จริง
ไม่ใช่ Python logic — ควรไปทาง Query Optimization (Part 067) มากกว่าไปแก้ Python code

### 654.5 Visualize ผลลัพธ์ด้วย snakeviz (Flame Graph แบบ Interactive)

การอ่านตัวเลขดิบยากต่อการเห็นภาพรวม **[snakeviz](https://jiffyclub.github.io/snakeviz/)**
แปลงไฟล์ `.prof` เป็นกราฟ interactive ที่เปิดในเบราว์เซอร์:

```bash
pip install snakeviz

# เปิดไฟล์ .prof ที่บันทึกไว้จากขั้นตอนที่ 654.3
snakeviz report.prof
```

snakeviz จะเปิดเบราว์เซอร์อัตโนมัติแสดง **Icicle chart** (ค่าเริ่มต้น) หรือ **Sunburst
chart**: แต่ละแท่ง/ส่วนโค้งแทนฟังก์ชันหนึ่งตัว **ความกว้าง = สัดส่วนเวลาที่ใช้**
ยิ่งแท่งกว้างเท่าไหร่ ยิ่งเป็นจุดที่ควรสนใจมากเท่านั้น สามารถคลิกซูมเข้าไปดูฟังก์ชัน
ย่อยที่ซ้อนอยู่ข้างในได้ทันที ทำให้เห็นภาพรวมได้เร็วกว่าการไล่อ่านตัวเลขทีละบรรทัด
มาก โดยเฉพาะเมื่อโค้ดมีฟังก์ชันซ้อนกันหลายสิบชั้น

### 654.6 ข้อจำกัดของ cProfile: Overhead สูงเกินกว่าจะใช้ใน Production

`cProfile` ดักจับทุกการเรียกฟังก์ชัน (instrumentation) ทำให้**เพิ่ม overhead ให้
โค้ดช้าลง 2-5 เท่า** จากปกติ — ยอมรับได้ในเครื่อง dev หรือรัน management command
แยกต่างหาก แต่**ห้ามเปิดใช้กับทุก request บน production เด็ดขาด** เพราะจะทำให้
ผู้ใช้จริงได้รับประสบการณ์ที่แย่ลงมากระหว่างที่กำลัง profile

นี่คือเหตุผลที่เมื่อต้องวิเคราะห์ process ที่รันอยู่จริงบน production เราต้องใช้
เครื่องมือคนละแบบที่ไม่มี instrumentation overhead — นั่นคือ **`py-spy`** ซึ่งเป็น
หัวข้อของขั้นตอนถัดไป

---

## ขั้นตอนที่ 655: `py-spy` — Sampling Profiler สำหรับ Process ที่กำลังรันอยู่โดยไม่ต้องแก้โค้ด

### 655.1 Sampling Profiler ต่างจาก Deterministic Profiler อย่างไร

**[py-spy](https://github.com/benfred/py-spy)** เขียนด้วย **Rust** โดย Ben Frederickson
ทำงานแบบ **sampling profiler**: แทนที่จะดักจับทุกการเรียกฟังก์ชัน (แบบ `cProfile`)
มันจะ**"แอบดู" stack trace ของ process เป็นระยะ ๆ** (เช่น ทุก 100 ครั้งต่อวินาที)
โดยไม่แตะต้องโค้ดที่กำลังรันอยู่เลย เพราะมันอ่านหน่วยความจำของ process โดยตรงจาก
ภายนอก (คล้ายกับที่ debugger ทำ) ไม่ต้อง import library ใด ๆ เข้าไปในโค้ด

| คุณสมบัติ | cProfile (Deterministic) | py-spy (Sampling) |
|---|---|---|
| วิธีทำงาน | Instrument ทุกการเรียกฟังก์ชัน | สุ่มดู stack trace เป็นระยะ |
| Overhead | สูงมาก (2-5 เท่า) | ต่ำมาก (~1-2%) |
| ต้องแก้โค้ดไหม | ต้อง import และเรียก `.enable()` | **ไม่ต้องแก้โค้ดแม้แต่บรรทัดเดียว** |
| แนบเข้า process ที่รันอยู่แล้วได้ไหม | ❌ ต้องเริ่มโค้ดใหม่พร้อม profiler | ✅ แนบเข้า PID ที่กำลังรันอยู่ได้ทันที |
| ความแม่นยำ | 100% แม่นยำ (นับทุกครั้งจริง) | เป็นค่าประมาณทางสถิติ (แม่นยำพอสำหรับหาจุดที่หนักที่สุด) |
| เหมาะกับ | Dev/staging, วิเคราะห์ฟังก์ชันเดี่ยว | **Production จริงที่กำลังมีผู้ใช้งานอยู่** |

### 655.2 ติดตั้ง py-spy

```bash
pip install py-spy

# ตรวจสอบเวอร์ชัน
py-spy --version
```

**ข้อควรระวังเรื่องสิทธิ์**: เนื่องจาก py-spy ต้องอ่านหน่วยความจำของ process อื่น
บน Linux อาจต้องรันด้วย `sudo` หรือปรับค่า `ptrace_scope`:

```bash
# Linux: อนุญาตให้ py-spy แนบเข้า process ได้โดยไม่ต้อง sudo (ชั่วคราว)
sudo sysctl -w kernel.yama.ptrace_scope=0

# หรือรันด้วย sudo ตรง ๆ ทุกครั้ง
sudo py-spy top --pid 12345

# ใน Docker container ต้องเพิ่ม capability นี้ตอนรัน container
docker run --cap-add SYS_PTRACE ...
```

### 655.3 `py-spy top` — ดูฟังก์ชันที่กินเวลาแบบเรียลไทม์ (เหมือนคำสั่ง `top`)

หา PID ของ Django process ก่อน (เช่น gunicorn worker หรือ `runserver`):

```bash
ps aux | grep manage.py
# หรือถ้ารันด้วย gunicorn
ps aux | grep gunicorn
```

```bash
py-spy top --pid 12345
```

ผลลัพธ์จะรีเฟรชแบบเรียลไทม์คล้ายคำสั่ง `top` ของ Linux:

```
Total Samples 1400
GIL: 45.00%, Active: 89.00%, Threads: 4

  %Own   %Total  OwnTime  TotalTime  Function (filename)
 32.00%  38.00%   2.1s      2.5s     get_full_name (django/contrib/auth/models.py)
 21.00%  55.00%   1.4s      3.6s     post_list (blog/views.py)
 15.00%  15.00%   1.0s      1.0s     execute (django/db/backends/utils.py)
  8.00%   8.00%   0.5s      0.5s     render (django/template/base.py)
```

คอลัมน์ `%Own` บอกสัดส่วนเวลาที่ฟังก์ชันนั้นใช้เอง ส่วน `%Total` รวมฟังก์ชันย่อยที่
มันเรียกด้วย — ใช้แนวคิดเดียวกับ `tottime`/`cumtime` ของ `cProfile` แต่วัดจาก process
จริงที่รันอยู่ ไม่ใช่จากการรันแยกต่างหาก

### 655.4 `py-spy dump` — Snapshot Stack Trace ทันที (Debug Process ที่ค้าง)

เมื่อ Django worker (เช่น gunicorn) **ค้าง (hang)** ไม่ตอบสนอง คำสั่งที่มีประโยชน์
ที่สุดคือ `py-spy dump` เพราะมันแสดง stack trace ปัจจุบันของทุก thread ทันที
โดยไม่ต้อง restart หรือรอ timeout:

```bash
py-spy dump --pid 12345
```

```
Process 12345: gunicorn: worker [config.wsgi]
Python v3.12.4

Thread 12345 (idle): "MainThread"
    _wait_for_tstate_lock (threading.py:1170)
    join (threading.py:1119)
    run_gunicorn_worker (gunicorn/workers/sync.py:45)

Thread 12346 (active): "Thread-1"
    execute (django/db/backends/utils.py:88)
    _fetch_all (django/db/models/query.py:1567)
    post_detail (blog/views.py:34)
```

จาก stack trace นี้ เห็นชัดว่า thread กำลังค้างอยู่ที่การรัน query ใน `post_detail`
— สถานการณ์จริงที่พบบ่อยคือ query ที่ lock ตารางไว้นาน หรือ deadlock ระหว่าง
transaction ซึ่งการใช้ `py-spy dump` ช่วยวินิจฉัยได้ภายในไม่กี่วินาที **โดยไม่ต้อง
หยุดหรือ restart process เลย**

### 655.5 `py-spy record` — สร้าง Flame Graph โดยไม่ต้องแก้โค้ด

```bash
# แนบเข้า process ที่รันอยู่แล้ว เก็บข้อมูล 30 วินาที
py-spy record -o profile.svg --pid 12345 --duration 30

# หรือรันคำสั่งใหม่พร้อม profile ไปด้วยเลย (เหมาะกับ dev/testing)
py-spy record -o profile.svg -- python manage.py runserver
```

ผลลัพธ์คือไฟล์ `profile.svg` ที่เปิดด้วยเบราว์เซอร์ได้ทันที แสดงเป็น **Flame Graph**:

```
แกน X (แนวนอน) = สัดส่วนเวลาที่ใช้ (ยิ่งกว้าง ยิ่งกินเวลามาก)
แกน Y (แนวตั้ง) = ความลึกของ call stack (ฟังก์ชันที่อยู่ข้างบนคือถูกเรียกจากฟังก์ชันข้างล่าง)

┌──────────────────────────────────────────────┐
│              get_full_name (แคบ)               │  ← ลึกสุด กินเวลาน้อย
├────────────────────────┬──────────────────────┤
│      _fetch_all         │   render_template    │  ← เรียกจากชั้นบน
├──────────────────────────────────────────────┤
│              post_list (view function)         │  ← กว้างที่สุด = เวลาส่วนใหญ่
└──────────────────────────────────────────────┘
```

การอ่าน Flame Graph ให้มองหา **"ที่ราบกว้าง" (plateau)** บนกราฟ — นั่นคือฟังก์ชันที่
กินเวลามากที่สุดในสัดส่วนรวม ต่างจาก icicle chart ของ snakeviz ตรงที่ Flame Graph
เหมาะกับการดูภาพรวมของ**ทั้ง process** มากกว่าฟังก์ชันเดียว

### 655.6 Use Case จริง: Profile Gunicorn Worker บน Production โดยไม่กระทบผู้ใช้

สถานการณ์จริงที่ py-spy มีค่ามากที่สุดคือ: เว็บ production ช้าลงกะทันหันแต่ไม่มี
error log อะไรเลย ทีมต้องหาสาเหตุ**โดยไม่หยุดให้บริการ**

```bash
# 1. หา PID ของ gunicorn worker ที่กำลังรับ traffic จริงบนเซิร์ฟเวอร์
ps aux | grep "gunicorn: worker"

# 2. Record 20 วินาที ระหว่างที่ traffic กำลังเข้ามาปกติ
sudo py-spy record -o /tmp/prod-profile.svg --pid 45231 --duration 20

# 3. ดาวน์โหลดไฟล์ svg กลับมาเปิดในเครื่อง local
scp user@production-server:/tmp/prod-profile.svg ./
```

ข้อดีที่ทำให้ปลอดภัยสำหรับ production คือ:

- **Overhead ต่ำมาก** (~1-2%) แทบไม่กระทบผู้ใช้จริงระหว่าง record
- **ไม่ต้อง restart worker** — worker เดิมยังรับ traffic ต่อเนื่องปกติทุกประการ
- **ไม่ต้อง deploy โค้ดใหม่** — ไม่มีความเสี่ยงเรื่อง deployment ผิดพลาดแทรกซ้อน

**ข้อควรระวัง**: ควรทดสอบคำสั่งนี้บน staging ก่อนเสมอเพื่อความคุ้นเคย และจำกัด
`--duration` ให้สั้น (10-30 วินาที) เพื่อไม่ให้ไฟล์ผลลัพธ์ใหญ่เกินความจำเป็น
py-spy เหมาะที่สุดสำหรับการ "จับภาพช่วงเวลาสั้น ๆ ที่กำลังมีปัญหา" ไม่ใช่การ monitor
ต่อเนื่องตลอดเวลา (สำหรับงานนั้นควรใช้ APM ในขั้นตอนที่ 657 แทน)

---

## ขั้นตอนที่ 656: วิธีค้นหา N+1 Query Problem อย่างเป็นระบบ

### 656.1 ทบทวน: N+1 Query Problem คืออะไร

ใน Part 065 ขั้นตอนที่ 645.4 คุณเห็นตัวอย่าง N+1 query เบื้องต้นแล้ว สรุปสั้น ๆ อีก
ครั้ง: **N+1 Query Problem** คือรูปแบบที่โค้ดยิง query 1 ครั้งเพื่อดึงรายการหลัก
(เช่น บทความ 20 รายการ) แล้วยิง query **เพิ่มอีก N ครั้ง** (N = จำนวนรายการ) เพื่อ
ดึงข้อมูลที่เกี่ยวข้องของแต่ละรายการแยกกัน รวมเป็น **N+1 query** ทั้งที่ควรทำได้ด้วย
query เดียวหรือสองครั้ง

ปัญหานี้เป็น**สาเหตุอันดับหนึ่ง**ของความช้าในแอป Django ทั่วโลก เพราะมันซ่อนตัวได้
ดีมากตอน development (ข้อมูลทดสอบมีน้อย จึง N ยังเล็ก ไม่รู้สึกช้า) แต่จะ**ระเบิด**
ทันทีที่ข้อมูล production โตขึ้นเป็นหลักพันหลักหมื่นแถว

### 656.2 Checklist เป็นระบบสำหรับตรวจจับ N+1 (ก่อนเรียนวิธีแก้เต็มรูปแบบใน Part 067)

ทีมมืออาชีพไม่รอให้ผู้ใช้แจ้งว่าเว็บช้า แต่มีกระบวนการตรวจจับ N+1 อย่างเป็นระบบ
4 ชั้น ที่เสริมกันตลอดวงจรพัฒนา:

```
ชั้นที่ 1: ระหว่างพัฒนา (มองด้วยตา)
  → Debug Toolbar SQL Panel (ขั้นตอนที่ 652.2): ดูจำนวน query + สีซ้ำของแถว

ชั้นที่ 2: ก่อน merge/push (อัตโนมัติแต่ยังต้องมีคนดูผล)
  → django-silk (ขั้นตอนที่ 653): ตรวจสอบ endpoint ที่ query สูงผิดปกติเทียบค่าเฉลี่ย

ชั้นที่ 3: ระหว่างพัฒนาแบบ real-time (แจ้งเตือนทันทีที่เขียนโค้ด)
  → django-nplusone middleware: raise exception ทันทีที่ตรวจพบรูปแบบ N+1

ชั้นที่ 4: ใน CI/CD (ป้องกันไม่ให้กลับมาซ้ำในอนาคต)
  → assertNumQueries ใน automated test (ทบทวนจาก Part 060)
```

### 656.3 ติดตั้งและใช้ `nplusone` — Middleware ตรวจจับ N+1 อัตโนมัติ

**[nplusone](https://github.com/jmcarp/nplusone)** คือ library ที่ hook เข้ากับ
Django ORM โดยตรง และ**แจ้งเตือนทันที**เมื่อพบรูปแบบการเข้าถึง related field
ที่ไม่ได้ใช้ `select_related`/`prefetch_related` ไว้ล่วงหน้า

```bash
pip install nplusone
```

```python
# config/settings/dev.py
INSTALLED_APPS += ["nplusone.ext.django"]

MIDDLEWARE += ["nplusone.ext.django.NPlusOneMiddleware"]

# โหมด dev: log คำเตือนออกมาเฉย ๆ ไม่รบกวนการทำงาน
NPLUSONE_LOGGER = logging.getLogger("nplusone")
NPLUSONE_LOG_LEVEL = logging.WARNING

# โหมด test: บังคับให้ raise exception ทันที (เข้มงวดกว่า เหมาะกับ CI)
NPLUSONE_RAISE = True
```

```python
# config/settings/test.py
from .base import *  # noqa: F401,F403

NPLUSONE_RAISE = True  # ทำให้ test ล้มเหลวทันทีถ้าเจอ N+1 ใหม่
```

ตัวอย่างผลลัพธ์เมื่อ nplusone ตรวจพบปัญหา (log หรือ exception แล้วแต่การตั้งค่า):

```
nplusone.core.exceptions.NPlusOneError: Potential n+1 query detected
    on `Post.author`
```

ข้อความนี้ชี้เป้าได้ตรงจุดทันทีว่า field ไหนของ model ไหนที่เป็นต้นเหตุ ทำให้ไม่ต้อง
เดาจากการอ่าน SQL panel เอง

### 656.4 เขียน Test ป้องกัน N+1 กลับมาอีกในอนาคต (ทบทวนจาก Part 060)

การตรวจจับด้วยตาหรือ middleware ยังไม่พอ เพราะไม่มีอะไรห้ามไม่ให้นักพัฒนาคนอื่นเขียน
โค้ดที่มี N+1 กลับเข้ามาใหม่ในอนาคต วิธีป้องกันถาวรคือเขียน **automated test** ที่
บังคับจำนวน query ตายตัวด้วย `assertNumQueries` (เรียนไปแล้วใน Part 060):

```python
# blog/tests/test_views.py
from django.test import TestCase
from django.urls import reverse

from blog.tests.factories import PostFactory


class PostListViewQueryCountTests(TestCase):
    def setUp(self):
        PostFactory.create_batch(20, is_published=True)

    def test_post_list_uses_constant_number_of_queries(self):
        """
        หน้า post_list ต้องใช้ query คงที่ไม่ว่าจะมีบทความกี่รายการ
        (ป้องกัน N+1 กลับมาในอนาคต — ถ้าใครเผลอลบ select_related() ออก
        test นี้จะ fail ทันทีใน CI ก่อนขึ้น production)
        """
        url = reverse("blog:post_list")

        # ตัวเลข 4 มาจาก: 1 query หลัก + select_related author/category
        # (รวมอยู่ใน query เดียวกันแล้ว) + 1 query สำหรับ comment count
        # แบบ annotate (ไม่ใช่ query แยกต่อบทความ)
        with self.assertNumQueries(4):
            self.client.get(url)
```

**สังเกต**: test นี้เขียนขึ้นเพื่อรอ "จำนวนที่ควรจะเป็น**หลังแก้ไข**" ไว้ล่วงหน้า
ก่อนที่จะไปแก้จริงใน Part 067 — นี่คือเทคนิคที่เรียกว่า **"Write the test first,
then make it pass"** (แนวคิดเดียวกับ TDD ที่เรียนใน Part 065 ขั้นตอนที่ 647)
ที่ใช้ได้ดีมากกับงาน performance ด้วยเช่นกัน

### 656.5 Walkthrough แบบครบวงจร: สืบสวนหน้า Blog List ด้วยเครื่องมือทั้งหมด

รวบยอดทุกเครื่องมือที่เรียนมาในขั้นตอนก่อนหน้า มาใช้สืบสวนปัญหาจริงเป็นขั้นตอน:

```
ขั้นที่ 1: เปิด Debug Toolbar → SQL Panel
  → เห็น "SQL queries (61)" พร้อมแถวสีซ้ำจำนวนมาก

ขั้นที่ 2: เปิด /silk/ → กดเรียงตาม "Num SQL Queries"
  → ยืนยันว่า /blog/ เป็น endpoint ที่ query สูงที่สุดในระบบ และแนวโน้มเพิ่มขึ้น
    ตามจำนวนบทความที่โตขึ้นทุกวัน (ดูกราฟ trend ของ Silk)

ขั้นที่ 3: เปิดใช้งาน nplusone middleware
  → เห็น log แจ้งชัดเจน 3 จุด: Post.author, Post.category, Post.comments

ขั้นที่ 4: เขียน test ด้วย assertNumQueries กำหนดเป้าหมาย (เช่น ควรเหลือ 1-2 query)
  → test นี้ fail ทันทีเพราะยังไม่ได้แก้ไข ("Red" ตามหลัก TDD)

สรุปผลการสืบสวน: พบ N+1 3 ชั้นซ้อนกันในหน้าเดียว จากทั้งหมด 61 query
ควรลดเหลือได้ไม่เกิน 2-3 query ด้วยเทคนิค select_related + prefetch_related + annotate
```

**นี่คือจุดที่ Part 066 จบหน้าที่ของมันพอดี** — คุณได้**ค้นพบและวัดปริมาณ**ปัญหาอย่าง
เป็นระบบครบทั้ง 4 ชั้นแล้ว ส่วนวิธี**แก้ไขจริง**ด้วย `select_related()`,
`prefetch_related()`, `Prefetch()` object, `only()`, `defer()`, และ `annotate()`
คือเนื้อหาเต็มรูปแบบทั้ง Part ของ **Part 067: Query Optimization** ที่จะตามมาทันที

---

## ขั้นตอนที่ 657: ภาพรวม APM Tools — Sentry Performance, New Relic, Datadog สำหรับ Monitor Production จริง

### 657.1 ทำไมเครื่องมือใน Dev (Debug Toolbar/Silk) ไม่พอสำหรับ Production

เครื่องมือทั้งหมดที่เรียนมาก่อนหน้านี้ (Debug Toolbar, Silk, cProfile, py-spy) ล้วน
เป็นเครื่องมือที่ต้อง**มีคนไปเปิดดูเอง** ณ เวลาใดเวลาหนึ่ง ในขณะที่ production จริง
ต้องการระบบที่:

- **Monitor 24/7 อัตโนมัติ** โดยไม่มีคนต้องนั่งเฝ้า
- **แจ้งเตือน (Alert)** ทันทีเมื่อ performance ตกลงผิดปกติ (เช่น p95 latency
  เกินเป้าที่ตั้งไว้)
- **เชื่อมโยง Error กับ Performance**: เมื่อเกิด error ต้องรู้ทันทีว่า request นั้น
  ช้าด้วยหรือไม่ เกิดที่ query ไหน
- **Distributed Tracing**: ติดตาม request หนึ่งตัวข้ามหลาย service (Django →
  database → Redis → external API) เพื่อหาว่าจุดไหนในสาย pipeline ที่ช้าที่สุด
- **เก็บข้อมูลระยะยาว**: ดู trend performance ย้อนหลังเป็นสัปดาห์/เดือน เพื่อเห็น
  แนวโน้มการเสื่อมสภาพ (performance regression) ก่อนที่ผู้ใช้จะบ่น

ระบบประเภทนี้เรียกว่า **APM (Application Performance Monitoring)**

### 657.2 ตารางเปรียบเทียบ APM Tools ยอดนิยม

| เครื่องมือ | โมเดล | จุดเด่น | เหมาะกับ |
|---|---|---|---|
| **Sentry Performance** | SaaS (มี free tier) | ผสาน Error Tracking + Performance ในตัวเดียว, setup ง่ายที่สุดสำหรับ Django | ทีมเล็ก-กลาง ที่ต้องการเริ่มต้นเร็ว |
| **New Relic** | SaaS (มี free tier จำกัด) | Dashboard ละเอียดมาก, APM ครบวงจร มีมานาน | องค์กรขนาดกลาง-ใหญ่ |
| **Datadog** | SaaS (enterprise-focused) | ครอบคลุมทั้ง Infrastructure + APM + Log ในแพลตฟอร์มเดียว | องค์กรขนาดใหญ่ที่มีหลาย service |
| **OpenTelemetry + Grafana/Prometheus** | Open Source (self-hosted) | ควบคุมข้อมูลเองทั้งหมด ไม่มีค่าใช้จ่าย SaaS | ทีมที่มีความสามารถดูแล infrastructure เอง |

### 657.3 ติดตั้ง Sentry Performance Monitoring สำหรับ Django

Sentry เป็นตัวเลือกที่แนะนำที่สุดสำหรับเริ่มต้น เพราะ setup ง่ายและผสาน Error
Tracking (ถ้าเคยติดตั้งไว้แล้วจากบทเรียนก่อนหน้า) เข้ากับ Performance ในหน้าจอ
เดียวกัน:

```bash
pip install --upgrade sentry-sdk
```

```python
# config/settings/production.py
import sentry_sdk
from sentry_sdk.integrations.django import DjangoIntegration

sentry_sdk.init(
    dsn=env("SENTRY_DSN"),
    integrations=[DjangoIntegration()],
    # เปิด Performance Monitoring: เก็บ trace 20% ของ request ทั้งหมด
    # (ไม่เก็บ 100% เพื่อลด overhead และค่าใช้จ่าย)
    traces_sample_rate=0.2,
    # เก็บ code-level profiling ควบคู่กับ trace (feature ใหม่ของ Sentry)
    profiles_sample_rate=0.2,
    send_default_pii=False,  # ไม่ส่งข้อมูลส่วนบุคคลของผู้ใช้ไปยัง Sentry
    environment=env("ENVIRONMENT", default="production"),
)
```

### 657.4 อ่านผลลัพธ์ Sentry Performance: Transaction และ Waterfall View

เมื่อติดตั้งแล้ว Sentry จะแบ่ง request แต่ละครั้งเป็น **Transaction** และแสดงเป็น
**Waterfall View** (คล้าย Timing Panel ของ Debug Toolbar แต่เก็บสะสมและดูย้อนหลัง
ได้) โดยแบ่งเป็น **Span** ย่อย ๆ:

```
Transaction: GET /blog/                                Total: 850ms
├── Span: django.middleware                              12ms
├── Span: django.view (post_list)                        720ms
│   ├── Span: db.query (SELECT ... FROM blog_post)        45ms
│   ├── Span: db.query (SELECT ... FROM auth_user) x20   380ms  ← ปัญหา N+1!
│   └── Span: django.template.render                     180ms
└── Span: django.response                                 18ms
```

หน้า Dashboard ของ Sentry ยังแสดงกราฟ **p50/p75/p95/p99** ของแต่ละ endpoint
ย้อนหลังได้เป็นสัปดาห์ ทำให้เห็น**แนวโน้ม** ไม่ใช่แค่ snapshot เดียว — ตัวอย่างเช่น
เห็นได้ทันทีว่า endpoint `/blog/` ค่า p95 latency เพิ่มขึ้นทุกสัปดาห์ตามจำนวนบทความ
ที่โตขึ้น (สอดคล้องกับที่สังเกตด้วย Silk ในขั้นตอนที่ 653.6) นี่คือสัญญาณเตือนล่วงหน้า
ที่ APM ให้ได้ดีกว่าเครื่องมือ dev เพียงอย่างเดียว

### 657.5 New Relic และ Datadog: แนวคิด Agent-Based Auto-Instrumentation

New Relic และ Datadog ใช้แนวคิดคล้ายกัน คือติดตั้ง **agent** ที่ inject
instrumentation เข้าไปในโค้ดโดยอัตโนมัติโดยแทบไม่ต้องแก้โค้ดเลย:

```bash
# New Relic: ติดตั้งและรัน Django ผ่าน wrapper ของ agent
pip install newrelic
newrelic-admin generate-config <LICENSE_KEY> newrelic.ini
NEW_RELIC_CONFIG_FILE=newrelic.ini newrelic-admin run-program \
    gunicorn config.wsgi:application

# Datadog: ใช้ ddtrace รันคู่กับ gunicorn ในลักษณะเดียวกัน
pip install ddtrace
DD_SERVICE=django-blog DD_ENV=production ddtrace-run \
    gunicorn config.wsgi:application
```

แนวคิด **auto-instrumentation** แบบนี้สะดวกมากในระดับองค์กร เพราะทีม infrastructure
สามารถเปิดใช้ monitoring ให้ทุกโปรเจกต์ได้โดยแทบไม่ต้องขอให้นักพัฒนาแก้โค้ดแอปเลย
แต่ก็มาพร้อมค่าใช้จ่ายที่สูงกว่า Sentry อย่างชัดเจนเมื่อ traffic เยอะขึ้น
(มักคิดราคาตาม host หรือปริมาณข้อมูลที่ ingest)

### 657.6 เมื่อไหร่ควรใช้ APM เต็มรูปแบบ เทียบกับเครื่องมือที่เรียนมาก่อนหน้า

| สถานการณ์ | เครื่องมือที่เหมาะสม |
|---|---|
| กำลังพัฒนาฟีเจอร์ใหม่ ยังไม่ deploy | Debug Toolbar |
| ต้องการดูภาพรวม query ของทั้งแอประหว่างพัฒนา | django-silk |
| ต้องการวิเคราะห์ฟังก์ชัน Python เฉพาะจุดแบบละเอียด | cProfile + snakeviz |
| Production ช้ากะทันหัน ต้อง debug โดยไม่หยุดระบบ | py-spy |
| ต้องการ monitor 24/7 พร้อมแจ้งเตือนอัตโนมัติ | **APM (Sentry/New Relic/Datadog)** |
| ทีมมีงบจำกัด ต้องการควบคุมข้อมูลเอง | OpenTelemetry + self-hosted Grafana |

ทีมมืออาชีพระดับโลกมักใช้**ทั้งหมดร่วมกัน** ไม่ใช่เลือกอย่างใดอย่างหนึ่ง: Debug
Toolbar/Silk สำหรับ dev, py-spy สำหรับ incident response แบบเฉพาะกิจ, และ APM
สำหรับ monitor ต่อเนื่องตลอดเวลาบน production

---

## ขั้นตอนที่ 658: วัดเวลาแต่ละขั้นตอนของ Request/Response Cycle ด้วย Custom Middleware

### 658.1 ทำไมต้องเขียน Middleware วัดเวลาเอง ทั้งที่มีเครื่องมือสำเร็จรูปแล้ว

Debug Toolbar, Silk และ APM ล้วนเป็นเครื่องมือสำเร็จรูปที่ทรงพลัง แต่บางครั้งทีม
ต้องการ**ควบคุมแบบเฉพาะเจาะจง** ที่เครื่องมือสำเร็จรูปให้ไม่ได้ เช่น:

- Log เฉพาะ request ที่ช้าเกินเกณฑ์ที่กำหนด ลงไฟล์ log ของทีมเอง (ไม่ผ่าน SaaS
  ภายนอก ด้วยเหตุผลด้านข้อมูลอ่อนไหวหรือ compliance)
- ส่ง header พิเศษกลับไปให้ frontend อ่านเวลาแบบ real-time (Server-Timing)
- Overhead ต่ำที่สุดเท่าที่จะทำได้ เพราะรันทุก request บน production จริง

### 658.2 Middleware วัดเวลารวมของ Request แบบพื้นฐาน

```python
# config/middleware/timing.py
import logging
import time

logger = logging.getLogger("performance")

SLOW_REQUEST_THRESHOLD_MS = 500  # ถือว่า "ช้า" ถ้าเกิน 500ms


class RequestTimingMiddleware:
    """วัดเวลารวมของแต่ละ request และ log เฉพาะที่ช้าเกินเกณฑ์"""

    def __init__(self, get_response):
        self.get_response = get_response

    def __call__(self, request):
        start_time = time.perf_counter()

        response = self.get_response(request)

        duration_ms = (time.perf_counter() - start_time) * 1000

        if duration_ms > SLOW_REQUEST_THRESHOLD_MS:
            logger.warning(
                "Slow request detected: %s %s took %.1fms",
                request.method,
                request.path,
                duration_ms,
                extra={
                    "path": request.path,
                    "method": request.method,
                    "duration_ms": round(duration_ms, 1),
                    "status_code": response.status_code,
                },
            )

        return response
```

```python
# config/settings/base.py
MIDDLEWARE = [
    "config.middleware.timing.RequestTimingMiddleware",  # อยู่บนสุดเพื่อจับเวลาทั้งหมด
    "django.middleware.security.SecurityMiddleware",
    # ... middleware อื่น ๆ ตามเดิม ...
]
```

**สังเกตตำแหน่ง**: การวาง middleware ไว้**บนสุด**ของ `MIDDLEWARE` list สำคัญมาก
เพราะ Django เรียก middleware เรียงจากบนลงล่างตอนรับ request และเรียงกลับจากล่าง
ขึ้นบนตอนส่ง response กลับ — การอยู่บนสุดทำให้ middleware นี้จับเวลา**ทั้งหมด**
รวมถึง middleware อื่น ๆ ทุกตัวที่อยู่ถัดไปด้วย

### 658.3 แยกเวลาแต่ละ Phase ละเอียดขึ้น: DB Time vs View Time

การรู้แค่เวลารวมยังไม่พอสำหรับวินิจฉัยปัญหา ปรับ middleware ให้แยกเวลาที่ใช้ไปกับ
ฐานข้อมูลออกจากเวลาที่เหลือ (Python logic + template render) โดยใช้
`django.db.connection.queries`:

```python
# config/middleware/timing.py
import logging
import time

from django.conf import settings
from django.db import connection, reset_queries

logger = logging.getLogger("performance")

SLOW_REQUEST_THRESHOLD_MS = 500


class RequestTimingMiddleware:
    def __init__(self, get_response):
        self.get_response = get_response

    def __call__(self, request):
        if settings.DEBUG:
            reset_queries()  # เคลียร์ query log ของ request ก่อนหน้า

        start_time = time.perf_counter()
        response = self.get_response(request)
        duration_ms = (time.perf_counter() - start_time) * 1000

        db_time_ms = 0.0
        query_count = 0
        if settings.DEBUG:
            # connection.queries เก็บเวลาแต่ละ query เฉพาะตอน DEBUG=True เท่านั้น
            query_count = len(connection.queries)
            db_time_ms = sum(float(q["time"]) for q in connection.queries) * 1000

        python_time_ms = duration_ms - db_time_ms

        if duration_ms > SLOW_REQUEST_THRESHOLD_MS:
            logger.warning(
                "Slow request: %s %s | total=%.1fms db=%.1fms (%d queries) python=%.1fms",
                request.method,
                request.path,
                duration_ms,
                db_time_ms,
                query_count,
                python_time_ms,
                extra={
                    "path": request.path,
                    "method": request.method,
                    "duration_ms": round(duration_ms, 1),
                    "db_time_ms": round(db_time_ms, 1),
                    "query_count": query_count,
                    "python_time_ms": round(python_time_ms, 1),
                },
            )

        return response
```

**หมายเหตุสำคัญ**: `connection.queries` เก็บข้อมูลเฉพาะตอน `settings.DEBUG = True`
เท่านั้น (เพื่อไม่ให้กิน memory บน production) ถ้าต้องการแยก DB time บน production
จริง ต้องใช้ Database instrumentation ของ APM (ขั้นตอนที่ 657) แทน ซึ่งออกแบบมาให้
เก็บข้อมูลนี้อย่างมีประสิทธิภาพโดยเฉพาะ

### 658.4 เพิ่ม Server-Timing Header ให้ Browser DevTools อ่านได้

**Server-Timing** คือ HTTP header มาตรฐาน (W3C) ที่ browser DevTools รู้จักและแสดง
ผลในแท็บ **Network → Timing** โดยอัตโนมัติ ทำให้ frontend developer เห็นว่าเวลา
ฝั่ง server แบ่งเป็นส่วนไหนบ้าง โดยไม่ต้องเปิด backend tool แยก:

```python
# config/middleware/timing.py (เพิ่มเข้าไปจากขั้นตอนที่ 658.3)
class RequestTimingMiddleware:
    def __init__(self, get_response):
        self.get_response = get_response

    def __call__(self, request):
        # ... โค้ดวัดเวลาเดิมจาก 658.3 ...
        response = self.get_response(request)
        # ... คำนวณ duration_ms, db_time_ms, python_time_ms เดิม ...

        response["Server-Timing"] = (
            f"db;dur={db_time_ms:.1f}, "
            f"app;dur={python_time_ms:.1f}, "
            f"total;dur={duration_ms:.1f}"
        )
        return response
```

เมื่อเปิด Chrome DevTools → Network → คลิก request → แท็บ Timing จะเห็นแถบ
"Server Timing" แสดงค่า `db`, `app`, `total` ที่ backend ส่งมา ทำให้การสื่อสาร
ระหว่างทีม frontend และ backend เรื่อง performance เป็นรูปธรรมมากขึ้น ไม่ต้องเดา
กันว่า "ช้าเพราะ server หรือช้าเพราะ network"

### 658.5 Log แบบมีโครงสร้าง (Structured Logging) เพื่อส่งเข้า Log Aggregator

ในทีมที่มีระบบรวม log จากหลายเซิร์ฟเวอร์ (เช่น ELK Stack, Loki, CloudWatch Logs)
การ log เป็นข้อความธรรมดาทำให้ค้นหา/กรองข้อมูลยาก ควร log เป็น **JSON structured
log** แทน:

```python
# config/settings/production.py
LOGGING = {
    "version": 1,
    "disable_existing_loggers": False,
    "formatters": {
        "json": {
            "()": "pythonjsonlogger.jsonlogger.JsonFormatter",
            "format": "%(asctime)s %(levelname)s %(name)s %(message)s",
        },
    },
    "handlers": {
        "console": {
            "class": "logging.StreamHandler",
            "formatter": "json",
        },
    },
    "loggers": {
        "performance": {
            "handlers": ["console"],
            "level": "WARNING",
            "propagate": False,
        },
    },
}
```

```bash
pip install python-json-logger
```

ด้วยการตั้งค่านี้ log ที่ middleware สร้างจากขั้นตอนที่ 658.2-658.3 (ผ่านพารามิเตอร์
`extra={...}`) จะถูกแปลงเป็น JSON บรรทัดเดียวโดยอัตโนมัติ เช่น:

```json
{"asctime": "2026-09-26 10:15:32", "levelname": "WARNING", "name": "performance", "message": "Slow request: GET /blog/ | total=850.2ms db=612.1ms (61 queries) python=238.1ms", "path": "/blog/", "method": "GET", "duration_ms": 850.2, "db_time_ms": 612.1, "query_count": 61, "python_time_ms": 238.1}
```

รูปแบบนี้นำเข้า Elasticsearch/Loki/CloudWatch ได้ทันทีโดยไม่ต้องเขียน parser เอง
ทำให้สร้าง dashboard และ alert (เช่น "แจ้งเตือนถ้ามี slow request เกิน 10 ครั้งใน
5 นาที") ได้ง่ายกว่าการ grep ข้อความ log แบบเดิมมาก

### 658.6 จาก Custom Middleware สู่การ Integrate กับระบบ Monitoring ภายนอก

Custom middleware ที่เขียนเองในขั้นตอนนี้เหมาะสำหรับ**เก็บข้อมูลดิบ** แต่การนำไป
สร้าง dashboard สวยงามพร้อมกราฟ trend/alert เต็มรูปแบบ ควรส่งต่อให้เครื่องมือ
เฉพาะทางจัดการ — Prometheus (ผ่าน `django-prometheus` package) สามารถ export
metric เช่น `django_http_requests_latency_seconds` ให้ Grafana สร้างกราฟและตั้ง
alert ได้โดยตรง ซึ่งเป็นแนวทางที่ทีม DevOps/SRE ระดับมืออาชีพนิยมใช้ควบคู่กับ
APM (รายละเอียดเชิงลึกเรื่อง monitoring stack แบบเต็มรูปแบบจะอยู่ใน Phase DevOps
ของหลักสูตรนี้)

---

## ขั้นตอนที่ 659: การตั้ง Performance Budget และ SLO

### 659.1 Performance Budget คืออะไร

**Performance Budget** คือ**เกณฑ์ตัวเลขที่ตกลงกันไว้ล่วงหน้า** ว่าแต่ละหน้า/endpoint
"ต้องเร็วแค่ไหน" เปรียบเสมือนงบประมาณเงินที่ห้ามใช้เกิน — ถ้าไม่มี budget ทีมจะไม่มี
ทางรู้ได้เลยว่า "เร็วพอหรือยัง" เพราะ "เร็ว" เป็นคำที่ตีความต่างกันได้ไม่จำกัด
Performance Budget เปลี่ยนคำถามเชิงความรู้สึก ("เว็บช้าไปไหม?") ให้เป็นคำถามที่
ตรวจสอบได้อัตโนมัติ ("TTFB เกิน 200ms หรือไม่?")

### 659.2 SLI, SLO, SLA: นิยามที่ต้องแยกให้ออก

แนวคิดนี้มาจากวงการ **Site Reliability Engineering (SRE)** ของ Google และเป็น
ศัพท์มาตรฐานที่ทีมวิศวกรรมระดับโลกใช้ร่วมกัน:

| คำศัพท์ | ความหมาย | ตัวอย่าง |
|---|---|---|
| **SLI** (Service Level Indicator) | **ตัวชี้วัด**ที่วัดได้จริง | "p95 latency ของหน้า `/blog/`" |
| **SLO** (Service Level Objective) | **เป้าหมาย**ภายในของทีมสำหรับ SLI นั้น | "p95 latency ของหน้า `/blog/` ต้อง ≤ 200ms" |
| **SLA** (Service Level Agreement) | **สัญญา**กับลูกค้าภายนอก (มักมีบทลงโทษถ้าไม่ทำตาม) | "รับประกัน uptime 99.9% มิฉะนั้นคืนเงินตามสัดส่วน" |

Performance Budget ที่เราจะตั้งในขั้นตอนนี้คือระดับ **SLO** (เป้าหมายภายในทีม)
ไม่จำเป็นต้องเป็น SLA ที่ผูกพันทางกฎหมายกับลูกค้าเสมอไป

### 659.3 ทำไมต้องดู Percentile (p50/p95/p99) ไม่ใช่แค่ค่าเฉลี่ย

ข้อผิดพลาดคลาสสิกของทีมที่เพิ่งเริ่มวัด performance คือดูแค่ **ค่าเฉลี่ย (average)**
ซึ่งซ่อนปัญหาที่ผู้ใช้บางกลุ่มเจอจริงได้ง่ายมาก ตัวอย่าง:

```
ตัวอย่างข้อมูล response time ของ 100 request:
  95 request ใช้เวลา 100ms
  5 request ใช้เวลา 5,000ms (เพราะ N+1 query ตอนมี cache miss)

ค่าเฉลี่ย (average) = (95×100 + 5×5000) / 100 = 345ms
  → ดูเหมือนโอเค ไม่มีใครตกใจ

แต่ค่า p95 (95th percentile) = 100ms
ค่า p99 (99th percentile) = 5,000ms
  → เผยความจริงว่า 1 ใน 100 คน ("unlucky user") เจอความช้าขนาด 5 วินาทีเต็ม ๆ
```

| Percentile | ความหมาย | ใช้ทำอะไร |
|---|---|---|
| **p50** (median) | ครึ่งหนึ่งของผู้ใช้เจอเวลาน้อยกว่านี้ | ดูภาพรวม "ทั่วไป" ของระบบ |
| **p95** | 95% ของผู้ใช้เจอเวลาน้อยกว่านี้ | **มาตรฐานที่นิยมใช้ตั้ง SLO มากที่สุด** |
| **p99** | 99% ของผู้ใช้เจอเวลาน้อยกว่านี้ | จับ "worst case" ที่พบได้แต่ไม่บ่อย (tail latency) |

ทีมมืออาชีพเกือบทั้งหมดตั้ง SLO บนฐาน **p95** เป็นค่าเริ่มต้น (สมดุลระหว่างความ
เข้มงวดกับความเป็นไปได้จริง) และเสริมด้วย p99 สำหรับ endpoint ที่สำคัญมาก ๆ

### 659.4 ตัวอย่าง Performance Budget สำหรับเว็บบล็อกจริง

| หน้า/Endpoint | Metric | Budget (p95) | เหตุผล |
|---|---|---|---|
| หน้าแรก (`/`) | TTFB | ≤ 200ms | หน้าที่มี traffic สูงสุด ต้อง cache ได้ดี |
| รายการบทความ (`/blog/`) | TTFB | ≤ 250ms | มี pagination และ query หลายตาราง |
| รายละเอียดบทความ (`/blog/<slug>/`) | TTFB | ≤ 150ms | ควร cache ได้ง่ายเพราะเนื้อหาเปลี่ยนไม่บ่อย |
| ค้นหา (`/search/`) | TTFB | ≤ 400ms | query ซับซ้อนกว่า ยอมรับได้ที่ช้ากว่าเล็กน้อย |
| API endpoint (`/api/posts/`) | TTFB | ≤ 300ms | ใช้โดย mobile app ที่ network อาจช้าอยู่แล้ว |
| หน้า Admin (`/admin/`) | TTFB | ไม่กำหนด (best-effort) | ใช้โดยทีมภายในเท่านั้น ไม่กระทบผู้ใช้จริง |
| **จำนวน SQL query ต่อ request** | Query count | ≤ 10 queries | ป้องกัน N+1 กลับมาแบบครอบคลุมทุกหน้า |
| **Core Web Vitals: LCP** | Frontend metric | ≤ 2.5 วินาที | ตามเกณฑ์ Google (ขั้นตอนที่ 651.4) |

### 659.5 บังคับ Performance Budget อัตโนมัติด้วย pytest

การตั้ง budget เฉย ๆ โดยไม่มีการตรวจสอบอัตโนมัติจะกลายเป็นแค่เอกสารที่ไม่มีใครดูอีก
เลยหลังจากสัปดาห์แรก ต้องเขียนเป็น **test ที่รันใน CI ทุกครั้ง**:

```python
# blog/tests/test_performance_budget.py
import time

from django.test import TestCase
from django.urls import reverse

from blog.tests.factories import PostFactory

# กำหนด budget เป็นค่าคงที่ที่แก้ไขง่าย อ้างอิงตารางในขั้นตอนที่ 659.4
PERFORMANCE_BUDGETS_MS = {
    "blog:home": 200,
    "blog:post_list": 250,
    "blog:post_detail": 150,
    "blog:search": 400,
}


class PerformanceBudgetTests(TestCase):
    """
    Test เหล่านี้เป็น 'canary' ที่ป้องกันไม่ให้ performance แย่ลงโดยไม่รู้ตัว
    หมายเหตุ: นี่คือค่าประมาณจาก test environment ไม่ใช่ตัวเลข production จริง
    (ใน CI ที่ resource จำกัด ตัวเลขจะสูงกว่าปกติ ควรปรับ threshold ให้เผื่อไว้)
    """

    @classmethod
    def setUpTestData(cls):
        PostFactory.create_batch(50, is_published=True)

    def _assert_within_budget(self, url_name, url_kwargs=None):
        url = reverse(url_name, kwargs=url_kwargs or {})
        budget_ms = PERFORMANCE_BUDGETS_MS[url_name]

        start = time.perf_counter()
        response = self.client.get(url)
        duration_ms = (time.perf_counter() - start) * 1000

        self.assertEqual(response.status_code, 200)
        self.assertLess(
            duration_ms,
            budget_ms,
            msg=(
                f"{url_name} ใช้เวลา {duration_ms:.1f}ms "
                f"เกิน budget ที่ตั้งไว้ {budget_ms}ms — ตรวจสอบ N+1 query "
                f"หรือ logic ที่เพิ่งแก้ไขล่าสุด"
            ),
        )

    def test_home_within_budget(self):
        self._assert_within_budget("blog:home")

    def test_post_list_within_budget(self):
        self._assert_within_budget("blog:post_list")

    def test_post_detail_within_budget(self):
        post = PostFactory(slug="test-post", is_published=True)
        self._assert_within_budget("blog:post_detail", {"slug": post.slug})
```

**ข้อควรระวังสำคัญ**: เวลาที่วัดได้ใน CI environment (ซึ่งมัก resource จำกัดกว่า
เครื่อง dev หรือ production จริง) จะมีความแปรปรวนสูงกว่าปกติ ทีมมืออาชีพมักตั้ง
threshold ใน test แบบนี้ให้**หลวมกว่า budget จริง 1.5-2 เท่า** เพื่อไม่ให้ CI fail
แบบ flaky (สุ่มไม่แน่นอน) จากความผันผวนของ resource เพียงอย่างเดียว — จุดประสงค์
หลักคือจับ**การถดถอยที่ชัดเจน** (regression) ไม่ใช่บังคับตัวเลขที่แม่นยำ 100%
ระดับ production

### 659.6 มองไปข้างหน้า: Load Testing เพื่อยืนยัน Budget ภายใต้ Traffic สูง

Test ในขั้นตอนที่ 659.5 ยืนยัน budget ได้แค่ตอน**ไม่มี concurrent user** เท่านั้น
คำถามที่สำคัญไม่แพ้กันคือ "budget นี้ยังคงอยู่หรือไม่เมื่อมีผู้ใช้ 1,000 คนเข้าพร้อม
กัน?" คำตอบนั้นต้องใช้เครื่องมือ **Load Testing** เช่น Locust หรือ k6 ที่จำลอง
traffic จำนวนมากพร้อมกันจริง — นี่คือเนื้อหาเต็มรูปแบบของ **Part 072: Load Testing
และ Scalability Testing** ที่จะปิดท้าย Phase 8 ทั้งหมด ในตอนนี้ให้จำไว้เป็นแนวคิด
ก่อนว่า Performance Budget ที่ตั้งในขั้นตอนนี้จะถูกนำไปทดสอบซ้ำภายใต้ load จริง
อีกครั้งเมื่อถึง Part 072

---

## ขั้นตอนที่ 660: สรุปและแบบฝึกหัด

### 660.1 ตารางสรุปเครื่องมือทั้งหมดใน Part นี้

| เครื่องมือ | ใช้เมื่อไหร่ | Overhead | เหมาะกับ Environment |
|---|---|---|---|
| **django-debug-toolbar** | ระหว่างเขียนโค้ด ดูทันทีทีละหน้า | ต่ำ-ปานกลาง | Development เท่านั้น |
| **django-silk** | ต้องการดูประวัติย้อนหลัง เปรียบเทียบข้าม request | ปานกลาง | Development, Staging (จำกัดสิทธิ์) |
| **cProfile + snakeviz** | วิเคราะห์ฟังก์ชัน Python เฉพาะจุดแบบละเอียดที่สุด | สูง (2-5 เท่า) | Development, สคริปต์แยกต่างหาก |
| **py-spy** | Debug production ที่กำลังรันจริงโดยไม่แก้โค้ด/ไม่ restart | ต่ำมาก (~1-2%) | Production (ใช้ชั่วคราวเฉพาะกิจ) |
| **nplusone** | ตรวจจับ N+1 แบบ real-time ระหว่างพัฒนา/test | ต่ำ | Development, Test/CI |
| **assertNumQueries** | ป้องกัน N+1 กลับมาซ้ำถาวร | ไม่มี (รันใน test) | CI/CD |
| **APM (Sentry/New Relic/Datadog)** | Monitor 24/7 พร้อม alert อัตโนมัติ | ต่ำ (sampled) | Production |
| **Custom Middleware** | ควบคุมเฉพาะทาง, ส่ง Server-Timing, log แบบกำหนดเอง | ต่ำมาก | ทุก Environment |
| **Performance Budget Test** | ป้องกัน performance ถดถอยแบบอัตโนมัติใน CI | ไม่มี (รันใน test) | CI/CD |

### 660.2 Workflow แนะนำ: ลำดับการใช้เครื่องมือเมื่อเจอปัญหา Performance

```
1. สงสัยว่าหน้าไหนช้า?
   → เปิด Debug Toolbar ดูทันที (ระหว่าง dev)
   → หรือเปิด /silk/ เรียงตาม Time/Num Queries (ถ้าอยากดูภาพรวมทั้งระบบ)

2. รู้แล้วว่าหน้าไหนช้า แต่ไม่รู้ว่าช้าที่ SQL หรือ Python?
   → ดู Timing Panel: SQL time สูง → ไปข้อ 3a, Python time สูง → ไปข้อ 3b

3a. SQL time สูง
   → ตรวจสอบ SQL Panel หา query ซ้ำ (N+1) → ยืนยันด้วย nplusone middleware
   → เขียน assertNumQueries กำหนดเป้าหมาย → แก้ไขจริงใน Part 067

3b. Python time สูง (ไม่ใช่ query)
   → รัน cProfile ผ่าน management command → เปิดด้วย snakeviz หาฟังก์ชันที่หนักสุด

4. Production ช้ากะทันหันตอนกลางคืน ไม่มีใครแก้โค้ดเลย?
   → py-spy dump/record แนบเข้า process ทันที ไม่ต้อง deploy ใหม่
   → ดู APM dashboard (Sentry/New Relic) เทียบ trend กับเมื่อวาน

5. แก้ปัญหาเสร็จแล้ว จะป้องกันไม่ให้กลับมาอีกได้อย่างไร?
   → ตั้ง Performance Budget เป็น test ใน CI (ขั้นตอนที่ 659.5)
   → ตั้ง Alert ใน APM สำหรับ threshold เดียวกัน (ขั้นตอนที่ 657)
```

### 660.3 สรุปสิ่งที่ได้เรียนรู้ใน Part นี้

- ✅ เข้าใจความแตกต่างระหว่าง Perceived Performance และ Actual Performance และ
  ผลกระทบของ performance ต่อ SEO (Core Web Vitals) และ conversion rate จริง
  ด้วยตัวเลขจากงานวิจัย
- ✅ ทบทวน django-debug-toolbar เจาะลึก 3 panel สำคัญ: SQL, Timing, Cache
- ✅ ติดตั้งและใช้ django-silk เพื่อเก็บประวัติ performance ย้อนหลังที่ Debug
  Toolbar ทำไม่ได้
- ✅ ใช้ `cProfile` + `pstats` + `snakeviz` วิเคราะห์ฟังก์ชัน Python ระดับลึก
- ✅ ใช้ `py-spy` (`top`, `dump`, `record`) profile process ที่รันอยู่จริงโดยไม่ต้อง
  แก้โค้ดหรือหยุดระบบ — เหมาะกับ incident บน production
- ✅ ค้นหา N+1 Query Problem อย่างเป็นระบบ 4 ชั้น: Debug Toolbar → Silk →
  nplusone → assertNumQueries
- ✅ เข้าใจภาพรวม APM Tools (Sentry Performance, New Relic, Datadog) และเมื่อไหร่
  ควรใช้แทนเครื่องมือ dev
- ✅ เขียน Custom Middleware วัดเวลาแต่ละ phase ของ request พร้อม Server-Timing
  header และ structured logging
- ✅ เข้าใจ SLI/SLO/SLA และตั้ง Performance Budget ที่บังคับด้วย automated test
  ได้จริง

### 660.4 Checklist ก่อนไป Part ถัดไป

- [ ] ติดตั้ง `django-silk` ในโปรเจกต์ `blog` สำเร็จ และเข้าดู `/silk/` ได้
- [ ] เปิด SQL Panel ของ Debug Toolbar แล้วนับจำนวน query ของหน้า post list ได้
- [ ] เขียน management command ที่ profile ฟังก์ชันหนึ่งด้วย `cProfile` สำเร็จ และ
      เปิดผลลัพธ์ด้วย `snakeviz` ได้
- [ ] ติดตั้ง `py-spy` และทดลองรัน `py-spy top --pid <PID>` กับ `python manage.py
      runserver` ที่รันอยู่สำเร็จ
- [ ] ติดตั้ง `nplusone` middleware และเห็นคำเตือน N+1 อย่างน้อย 1 จุดในโปรเจกต์
- [ ] เขียน test ด้วย `assertNumQueries` อย่างน้อย 1 test ที่ระบุจำนวน query เป้าหมาย
- [ ] เขียน Custom Middleware วัดเวลา request พร้อมเพิ่ม `Server-Timing` header
- [ ] มีตาราง Performance Budget อย่างน้อย 3 endpoint พร้อม test ที่บังคับ budget
      นั้นใน `blog/tests/test_performance_budget.py`

### 660.5 แบบฝึกหัดท้ายบท

**แบบฝึกหัดที่ 1**: ติดตั้ง `django-silk` เข้ากับโปรเจกต์ `blog` ของคุณให้ครบตาม
ขั้นตอนที่ 653.2 แล้วเปิดใช้งานหน้าเว็บอย่างน้อย 20 ครั้งผ่าน endpoint ต่าง ๆ กัน
(home, post list, post detail, search) จากนั้นเข้า `/silk/` เรียงลำดับตาม "Time"
และ "Num SQL Queries" แล้วบันทึกลง `notes.md` ว่า endpoint ไหนช้าที่สุด และมีจำนวน
query เท่าไหร่

**แบบฝึกหัดที่ 2**: เขียน management command ชื่อ `profile_post_list` ที่ใช้
`cProfile` วัดเวลาของฟังก์ชัน view `post_list` (เรียกผ่าน Django test client หรือ
เรียกฟังก์ชันตรง ๆ ก็ได้) บันทึกผลลัพธ์เป็นไฟล์ `.prof` แล้วเปิดด้วย `snakeviz`
สรุปเป็นข้อความว่าฟังก์ชันไหน 3 อันดับแรกที่ใช้เวลา `tottime` มากที่สุด

**แบบฝึกหัดที่ 3**: รัน Django development server แล้วใช้ `py-spy record -o
profile.svg --pid <PID> --duration 15` ระหว่างที่คุณเข้าเว็บและคลิกไปมาหลาย ๆ
หน้าพร้อมกัน เปิดไฟล์ `profile.svg` ที่ได้ในเบราว์เซอร์ แล้วอธิบายด้วยคำพูดของ
ตัวเองว่า flame graph ที่เห็นบอกอะไรบ้าง (แท่งไหนกว้างที่สุด และคาดว่าทำไม)

**แบบฝึกหัดที่ 4 (โจทย์ใหญ่รวบยอด)**: ทำการสืบสวนแบบครบวงจรกับหน้า Blog List
ของโปรเจกต์คุณเอง โดยใช้เครื่องมือ**ทั้งหมด**ที่เรียนใน Part นี้ตามลำดับ workflow
ในขั้นตอนที่ 660.2:
1. เปิด Debug Toolbar หาจำนวน query และดู pattern สีซ้ำ
2. เปิด Silk ยืนยันว่า endpoint นี้ติด 3 อันดับที่ช้าที่สุดในระบบหรือไม่
3. ติดตั้ง `nplusone` แล้วดูว่าแจ้งเตือน N+1 กี่จุด และเป็น field ไหนบ้าง
4. เขียน test ด้วย `assertNumQueries` ระบุจำนวน query **เป้าหมาย** ที่ควรจะเป็น
   หลังแก้ไข (test นี้ควร fail ก่อน เพราะยังไม่ได้แก้จริง — เป็น "Red" ของ TDD)
5. กำหนด Performance Budget (p95 TTFB) สำหรับหน้านี้ในตาราง แล้วเขียน
   `PerformanceBudgetTests` ตามขั้นตอนที่ 659.5
6. เขียนรายงานสรุป 1 หน้าใน `notes.md` ระบุ: bottleneck ที่แท้จริงคืออะไร (ควรเป็น
   N+1 query), มีกี่จุด, และตัวเลข query count ปัจจุบันเทียบกับเป้าหมายที่ตั้งไว้
   (**ยังไม่ต้องแก้ไขจริง** — การแก้ไขคือเนื้อหาของ Part 067)

### 660.6 คำถามที่พบบ่อย (FAQ)

**Q: ต้องติดตั้งทั้ง django-debug-toolbar และ django-silk พร้อมกันไหม หรือเลือก
อย่างใดอย่างหนึ่งพอ?**
A: แนะนำให้ติดตั้งทั้งคู่ เพราะใช้งานคนละจุดประสงค์ Debug Toolbar เหมาะกับการดู
รายละเอียดของหน้าที่กำลังเปิดอยู่ตรงหน้าทันที ส่วน Silk เหมาะกับการดูภาพรวมและ
ประวัติย้อนหลัง — ทีมมืออาชีพจำนวนมากใช้ทั้งสองตัวควบคู่กันตลอดการพัฒนา

**Q: py-spy ปลอดภัยพอที่จะรันบน production จริงหรือไม่?**
A: ปลอดภัยกว่า `cProfile` มาก เพราะ overhead ต่ำกว่ามาก (sampling ไม่ใช่
instrumentation) แต่ควรใช้แบบ**เฉพาะกิจ** (record ช่วงสั้น ๆ 10-30 วินาทีตอนมี
ปัญหาจริง) ไม่ใช่เปิดทิ้งไว้ตลอดเวลา และควรทดสอบบน staging ให้คุ้นเคยกับคำสั่งก่อน
นำไปใช้กับ production จริงเสมอ

**Q: ทำไม assertNumQueries ถึงสำคัญ ทั้งที่มี Debug Toolbar และ Silk ช่วยดูแล้ว?**
A: เพราะ Debug Toolbar/Silk เป็นเครื่องมือ**manual** ที่ต้องมีคนไปเปิดดูเอง ในขณะที่
`assertNumQueries` เป็น**automated check** ที่รันทุกครั้งใน CI โดยไม่ต้องมีใครจำได้
ว่าต้องไปเปิดดู — นี่คือหลักการเดียวกับที่ Part 065 สอนเรื่อง pre-commit hook:
เครื่องมือที่ต้องอาศัยความจำคนจะถูกลืมในที่สุด ส่วนเครื่องมือที่บังคับอัตโนมัติจะ
ทำงานเสมอ

**Q: ควรตั้ง Performance Budget ที่เข้มงวดแค่ไหน?**
A: เริ่มจากตัวเลขที่**วัดได้จริงในปัจจุบัน + เผื่อ margin เล็กน้อย** ไม่ใช่ตัวเลข
ในฝันที่ทำไม่ได้จริง เช่น ถ้าตอนนี้หน้า `/blog/` ใช้เวลาเฉลี่ย 300ms ให้ตั้ง budget
ไว้ที่ 350-400ms ก่อน แล้วค่อย ๆ ลดตัวเลขลงทีละน้อยหลังจากทำ Query Optimization
(Part 067) และ Caching (Part 068-069) สำเร็จแล้ว การตั้ง budget ที่เข้มงวดเกินจริง
ตั้งแต่แรกจะทำให้ CI fail ตลอดเวลาจนทีมเริ่มเพิกเฉยต่อมัน (เหมือนสัญญาณเตือนไฟไหม้
ที่ดังบ่อยเกินจนคนเลิกสนใจ)

**Q: ทำไมไม่สอน APM (Sentry/New Relic/Datadog) แบบลงมือทำละเอียดเหมือนเครื่องมืออื่น
ในบทนี้?**
A: เพราะ APM เป็นบริการ SaaS ภายนอกที่ต้องสมัครบัญชีและมีค่าใช้จ่ายจริงเมื่อ scale
ขึ้น หลักสูตรนี้จึงให้**ภาพรวมแนวคิดและโค้ดตั้งค่าเบื้องต้น**พอให้เริ่มต้นได้ ส่วน
รายละเอียดเชิงลึกเรื่อง monitoring stack แบบเต็มรูปแบบ (รวมถึง self-hosted ด้วย
Prometheus/Grafana) จะอยู่ใน Phase DevOps ของหลักสูตรที่เจาะจงเรื่อง infrastructure
โดยเฉพาะ

**Q: ต้องใช้เครื่องมือทั้งหมดในบทนี้พร้อมกันทุกครั้งไหม?**
A: ไม่จำเป็น เลือกใช้ตาม workflow ในขั้นตอนที่ 660.2 ตามสถานการณ์ที่เจอจริง —
เครื่องมือทั้งหมดในบทนี้คือ "กล่องเครื่องมือ" (toolbox) ที่หยิบมาใช้ตามความเหมาะสม
ไม่ใช่ checklist ที่ต้องทำครบทุกข้อทุกครั้งที่ตรวจสอบ performance

---

## เตรียมตัวสำหรับ Part ถัดไป

คุณเพิ่งค้นพบและวัดปริมาณปัญหา **N+1 Query** ในหน้า Blog List อย่างเป็นระบบ
ครบทั้ง 4 ชั้น (Debug Toolbar → Silk → nplusone → assertNumQueries) แต่ยังไม่ได้
**แก้ไข**อะไรเลยแม้แต่บรรทัดเดียว — นั่นคือหน้าที่ของ **Part 067: Query
Optimization: select_related, prefetch_related** ที่จะพาคุณเจาะลึกเทคนิคทั้งหมด
ที่ Django ORM มีให้เพื่อลดจำนวน query ให้เหลือน้อยที่สุด ได้แก่ `select_related()`
สำหรับความสัมพันธ์ ForeignKey/OneToOne, `prefetch_related()` และ `Prefetch()`
object สำหรับความสัมพันธ์ ManyToMany/reverse ForeignKey, `only()`/`defer()`
สำหรับจำกัด field ที่ดึงมา, และ `annotate()` สำหรับคำนวณค่าสรุป (เช่น comment
count) ในระดับฐานข้อมูลแทนการ loop นับใน Python

เตรียม test ที่คุณเขียนไว้ในแบบฝึกหัดที่ 4 (ขั้นตอนที่ 660.5) ให้พร้อม เพราะ
Part 067 จะพาคุณทำให้ test เหล่านั้นเปลี่ยนจาก **"Red" เป็น "Green"** ทีละจุดด้วย
เทคนิคการ optimize query อย่างเป็นระบบ แล้วไปพบกันที่ Part 067!
