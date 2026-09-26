# Part 073: Async Views และ ASGI เบื้องต้น

> **ขั้นตอนที่ 721-730 ของหลักสูตร** | Phase 9: Async, Celery และ Channels (Part แรกของ Phase)
>
> Part 057 พาคุณตั้งค่า ASGI เพื่อรองรับ WebSocket ผ่าน Django Channels ไปแล้ว
> แต่ตอนนั้นเราโฟกัสที่ **WebSocket โดยเฉพาะ** และ HTTP request ทั้งหมดยังคงถูก
> ส่งเข้า `django_asgi_app` แบบเดิมทุกประการ ไม่มี View ไหนถูกเขียนเป็น `async def`
> เลยแม้แต่ตัวเดียว Part นี้คือจุดที่เราจะกลับมาที่ **HTTP View ธรรมดา** อีกครั้ง
> แล้วถามคำถามใหม่ว่า "ถ้า View ของเราต้องรอ I/O นาน ๆ เช่น เรียก API ภายนอก
> จะเขียนเป็น `async def` เพื่อให้เร็วขึ้นได้ไหม" คุณจะได้เขียน Async View แรก
> ด้วยตัวเอง เข้าใจว่าเมื่อไหร่มันช่วยจริงและเมื่อไหร่มันไม่ช่วยเลย ใช้ Async ORM
> ของ Django 5.x (`aget`, `acreate`, `afilter`, `async for`) เชื่อมโค้ด sync กับ
> async เข้าด้วยกันด้วย `sync_to_async`/`async_to_sync` เรียก External API แบบ
> concurrent ด้วย `httpx`, เปรียบเทียบ ASGI server สามค่าย (Daphne, Uvicorn,
> Hypercorn), เขียน Middleware ที่รองรับทั้งสองโหมด, เขียนเทสต์สำหรับ Async View
> และปิดท้ายด้วยข้อจำกัดที่สำคัญที่สุดที่มือใหม่มักเข้าใจผิด: **Async ไม่ได้ทำให้
> Python เร็วขึ้นเสมอไป** เพราะ GIL ยังคงอยู่ที่นั่นเสมอ Part นี้ปูพื้นฐานให้แน่น
> ก่อนที่ Part 074 จะพาไปเจาะลึก Consumer และ Groups ขั้นสูงของ Channels ต่อ

---

## สารบัญของ Part นี้

- ขั้นตอนที่ 721: เขียน Async View แรกด้วย `async def` ใน Django
- ขั้นตอนที่ 722: เมื่อไหร่ Async View ถึงช่วยจริง — I/O-bound เทียบกับ CPU-bound
- ขั้นตอนที่ 723: Async ORM Support ใน Django 5.x — `aget()`, `acreate()`, `afilter()`, `async for`
- ขั้นตอนที่ 724: `sync_to_async`/`async_to_sync` — เชื่อมโค้ด sync กับ async เข้าด้วยกัน
- ขั้นตอนที่ 725: เรียก External Async API จาก Django ด้วย `httpx` async client
- ขั้นตอนที่ 726: ASGI Server เปรียบเทียบ — Daphne vs Uvicorn vs Hypercorn
- ขั้นตอนที่ 727: Middleware ที่รองรับ Async (`async_capable`/`sync_capable`)
- ขั้นตอนที่ 728: การเขียน Test สำหรับ Async View (`async def test_...`, `AsyncClient`)
- ขั้นตอนที่ 729: ข้อจำกัดของ Async ใน Python — GIL และทำไมไม่ช่วยงาน CPU-bound
- ขั้นตอนที่ 730: สรุปและแบบฝึกหัด

---

## ขั้นตอนที่ 721: เขียน Async View แรกด้วย `async def` ใน Django

### 721.1 ทบทวน: View ปกติที่เขียนมาตั้งแต่ Part 004 คือ Synchronous ล้วน ๆ

ทุก View ที่คุณเขียนมาตลอดหลักสูตรนี้ ไม่ว่าจะเป็น function-based view (Part 004),
class-based view (Phase 3) หรือ DRF ViewSet (Phase 5) ล้วนเป็นโค้ด **synchronous**
ทั้งหมด — เขียนด้วย `def` ธรรมดา และเมื่อ View ต้องรออะไรสักอย่าง (เช่น query
ฐานข้อมูล) **thread ที่กำลังรัน View นั้นจะหยุดรอเฉย ๆ จนกว่าจะได้คำตอบ** ก่อนจะ
ทำงานบรรทัดถัดไปต่อ:

```python
# blog/views.py (ทบทวนรูปแบบเดิมจาก Part 004)
from django.shortcuts import render

from .models import Post


def post_list(request):
    posts = Post.objects.filter(is_published=True)  # thread หยุดรอจน query เสร็จ
    return render(request, "blog/post_list.html", {"posts": posts})
```

Django ตั้งแต่เวอร์ชัน **3.1** เป็นต้นมา (และเสถียรเต็มรูปแบบตั้งแต่ 4.1) เปิดทางให้
เขียน View เป็น **`async def`** ได้โดยตรง โดยไม่ต้องพึ่ง Channels เลยแม้แต่น้อย —
Channels (Part 057) กับ Async View (Part นี้) เป็นคนละเรื่องกัน แม้จะยืนอยู่บน
รากฐาน ASGI เดียวกันก็ตาม

### 721.2 เขียน Async View แรก: เปลี่ยน `def` เป็น `async def`

```python
# blog/views.py
import asyncio

from django.http import HttpResponse


async def async_ping_view(request):
    """Async View ที่ง่ายที่สุดเท่าที่จะเป็นไปได้ — ไม่มีอะไรให้ await จริง ๆ ด้วยซ้ำ"""
    await asyncio.sleep(0)  # ตัวอย่างจุดที่ "ยกการควบคุมกลับให้ event loop" ได้
    return HttpResponse("pong (async)")
```

```python
# blog/urls.py
from django.urls import path

from . import views

app_name = "blog"

urlpatterns = [
    # ... path เดิมจาก Part ก่อนหน้า ...
    path("async-ping/", views.async_ping_view, name="async_ping"),
]
```

สังเกตว่า **`urls.py` ไม่ต้องแก้อะไรเป็นพิเศษเลย** — `path()` รับได้ทั้งฟังก์ชัน
`def` ธรรมดาและ `async def` โดย Django จะตรวจจับความแตกต่างให้อัตโนมัติเบื้องหลัง
รันเซิร์ฟเวอร์แล้วเข้า `http://127.0.0.1:8000/blog/async-ping/` จะเห็นข้อความ
`pong (async)` เหมือนกับ View ปกติทุกประการจากมุมมองของ browser

### 721.3 Django รู้ได้อย่างไรว่า View ไหนเป็น Async

เบื้องหลัง Django ใช้ `asyncio.iscoroutinefunction()` (ผ่าน helper ของ
`asgiref`) ตรวจสอบทุก View ตอนโหลด URLconf — ถ้าพบว่าเป็น coroutine function
Django จะจัดการเรียกมันด้วยกลไกที่ต่างจาก View ปกติ ตารางเปรียบเทียบจุดต่าง:

| ประเด็น | Sync View (`def`) | Async View (`async def`) |
|---|---|---|
| วิธีนิยาม | `def my_view(request):` | `async def my_view(request):` |
| การ return | `return HttpResponse(...)` | `return HttpResponse(...)` (เหมือนเดิมทุกประการ) |
| การเรียกใน Middleware chain | เรียกตรง ๆ | Django ต้อง `await` มัน |
| รันได้บน WSGI server ปกติหรือไม่ | ✅ ได้ตามปกติ | ✅ ได้ — Django ห่อด้วย `async_to_sync` ให้อัตโนมัติ แต่ **เสียประโยชน์ของ async ไปเกือบหมด** (ขั้นตอนที่ 722) |
| รันได้บน ASGI server หรือไม่ | ✅ ได้ (ห่อด้วย `sync_to_async` ให้อัตโนมัติ) | ✅ ได้เต็มประสิทธิภาพ |
| เหมาะกับงานแบบไหน | งานทั่วไป, งานที่ใช้ ORM หนัก | งานที่ต้องรอ I/O ภายนอกนาน ๆ (ขั้นตอนที่ 722) |

**ข้อสังเกตสำคัญที่มือใหม่มักพลาด**: การเปลี่ยน `def` เป็น `async def` เฉย ๆ
**ไม่ได้ทำให้ View เร็วขึ้นโดยอัตโนมัติ** ถ้าข้างในยังคงเป็นโค้ดที่รอแบบ blocking
อยู่ดี (เช่น เรียก `requests.get()` ธรรมดาที่เป็น sync library) การ async
จะช่วยจริงก็ต่อเมื่อทุกจุดที่ "รอ" ภายใน View ใช้ `await` กับของที่เป็น async
แท้ ๆ เท่านั้น — นี่คือสิ่งที่ขั้นตอนที่ 722 จะอธิบายอย่างละเอียด

### 721.4 Class-Based View แบบ Async

Django รองรับ CBV ที่เป็น async ด้วยเช่นกัน ตั้งแต่ Django 4.1 method ของ
`View` (เช่น `get`, `post`) สามารถเป็น `async def` ได้โดยตรง:

```python
# blog/views.py
from django.http import JsonResponse
from django.views import View


class AsyncStatusView(View):
    """CBV ที่ทุก HTTP method เป็น async — Django ตรวจจับให้อัตโนมัติจาก __init__"""

    async def get(self, request, *args, **kwargs):
        return JsonResponse({"status": "ok", "mode": "async"})
```

```python
# blog/urls.py
urlpatterns = [
    # ...
    path("async-status/", views.AsyncStatusView.as_view(), name="async_status"),
]
```

**กฎสำคัญ**: ใน CBV เดียวกัน **ห้ามผสม** `def get` กับ `async def post` —
Django กำหนดว่า **ทุก HTTP method handler ใน class เดียวกันต้องเป็นแบบเดียวกัน
หมด** (ทั้ง sync หรือทั้ง async) ไม่เช่นนั้นจะเจอ `ImproperlyConfigured` ตอน
`as_view()` ถูกเรียก

### 721.5 ทดสอบด้วย `curl` และดู Header ที่ต่างออกไป

```bash
curl -i http://127.0.0.1:8000/blog/async-ping/
```

```
HTTP/1.1 200 OK
Content-Type: text/plain; charset=utf-8
Content-Length: 15

pong (async)
```

จากมุมมองของ HTTP protocol แล้ว **ไม่มีอะไรต่างจาก sync view เลยแม้แต่นิดเดียว**
— async view ไม่ได้เปลี่ยนโปรโตคอล ไม่ได้เปลี่ยน response format ไม่ต้องเปลี่ยน
โค้ดฝั่ง frontend เลยแม้แต่บรรทัดเดียว สิ่งที่เปลี่ยนไปคือ **วิธีที่ Django รัน
โค้ดของคุณอยู่เบื้องหลังเท่านั้น**

---

## ขั้นตอนที่ 722: เมื่อไหร่ Async View ถึงช่วยจริง — I/O-bound เทียบกับ CPU-bound

### 722.1 สองประเภทของงานที่ View มักต้องทำ

ก่อนจะตัดสินใจว่า View ไหนควรเป็น async ต้องแยกให้ออกก่อนว่างานที่ View นั้นทำ
เป็นประเภทไหนใน 2 ประเภทนี้:

| ประเภทงาน | ความหมาย | ตัวอย่าง | Async ช่วยไหม |
|---|---|---|---|
| **I/O-bound** | เวลาส่วนใหญ่หมดไปกับการ**รอ**สิ่งภายนอกตอบกลับ (CPU ว่างอยู่ระหว่างรอ) | เรียก external API, รอ network, รอ disk I/O, รอ DB query ผ่าน network | ✅ **ช่วยได้มาก** — event loop สลับไปทำ request อื่นระหว่างรอได้ |
| **CPU-bound** | เวลาส่วนใหญ่หมดไปกับการ**คำนวณจริง** (CPU ทำงานเต็มที่ตลอด) | ย่อรูปภาพ, สร้าง PDF, เข้ารหัส/ถอดรหัสหนัก ๆ, loop คำนวณจำนวนมาก | ❌ **ไม่ช่วยเลย** (เจาะลึกเหตุผลในขั้นตอนที่ 729) |

### 722.2 ภาพเปรียบเทียบ: เซิร์ฟเวอร์กำลังทำอะไรระหว่าง "รอ"

```
Sync View เรียก External API (blocking):
Thread 1: [ส่ง request]───(ว่าง แต่ thread ถูกจองไว้ ทำอะไรไม่ได้)───[ได้ response]─▶ ทำงานต่อ
Thread 2: ต้องรอ Thread 1 ว่างก่อน ถ้า thread pool เต็มก็ต้องต่อคิว

Async View เรียก External API (non-blocking):
Task 1:   [ส่ง request]──await──┐
                                  ├─ event loop ว่าง ไปรัน Task 2, 3, 4 ต่อระหว่างรอ
Task 2:   [ส่ง request]──await──┤
                                  ▼
Task 1:  ◀── ได้ response กลับมา ── ทำงานต่อจากจุดเดิม (ไม่เสีย thread ไปเลย)
```

นี่คือเหตุผลที่ข้อ 722.1 สรุปว่า async ช่วยงาน I/O-bound ได้มาก — ระหว่างที่
View กำลัง "รอ" คำตอบจากภายนอก event loop เดียวสามารถสลับไปประมวลผล request
ของผู้ใช้คนอื่นได้ทันที โดยไม่ต้องจองทรัพยากร thread ทิ้งไว้เฉย ๆ เหมือนโมเดล
synchronous แบบเดิม

### 722.3 ตัวอย่างจริง: เรียก 3 External API พร้อมกันแบบ Concurrent

สมมติ View ต้องเรียก 3 endpoint ภายนอกที่ไม่เกี่ยวข้องกัน (เช่น ดึงจำนวน stars
ของ 3 repository บน GitHub มาแสดงในหน้าเดียว) ถ้าเขียนแบบ sync จะต้องรอทีละตัว
เรียงกันไป (**sequential**) แต่ถ้าเขียนแบบ async จะเรียกพร้อมกันได้ด้วย
`asyncio.gather()` (**concurrent**):

```python
# blog/services/github.py
import httpx

REPOS = ["django/django", "encode/httpx", "celery/celery"]


async def fetch_repo_stars(client: httpx.AsyncClient, repo_full_name: str) -> dict:
    response = await client.get(f"https://api.github.com/repos/{repo_full_name}")
    response.raise_for_status()
    data = response.json()
    return {"repo": repo_full_name, "stars": data["stargazers_count"]}


async def fetch_all_repo_stars() -> list[dict]:
    async with httpx.AsyncClient(timeout=5.0) as client:
        # asyncio.gather ส่งทั้ง 3 request "พร้อมกัน" ไม่ต้องรอตัวก่อนหน้าเสร็จก่อน
        results = await asyncio.gather(
            *(fetch_repo_stars(client, repo) for repo in REPOS)
        )
    return results
```

```python
# blog/views.py
import asyncio

from django.http import JsonResponse

from .services.github import fetch_all_repo_stars


async def project_showcase(request):
    repo_stats = await fetch_all_repo_stars()
    return JsonResponse({"repos": repo_stats})
```

ถ้าแต่ละ request ใช้เวลาประมาณ 300ms เท่ากัน:

| วิธีเรียก | เวลารวมโดยประมาณ | เหตุผล |
|---|---|---|
| Sync แบบเรียงลำดับ (`requests.get()` ทีละตัว) | ~900ms (300 × 3) | ต้องรอตัวก่อนหน้าตอบกลับก่อนถึงจะยิงตัวถัดไป |
| Async แบบ `asyncio.gather()` | ~300ms (เท่ากับตัวที่ช้าที่สุดตัวเดียว) | ทั้ง 3 request "รอ" พร้อมกันบน event loop เดียว |

นี่คือ**ตัวอย่างที่ชัดเจนที่สุด**ของสถานการณ์ที่ async view คุ้มค่าอย่างแท้จริง
— ยิ่งจำนวน external call ที่ต้องรอมากเท่าไหร่ ส่วนต่างของเวลาก็ยิ่งมากขึ้น
แบบทวีคูณเมื่อเทียบกับ sync

### 722.4 กรณีที่ Async "ไม่ช่วย" หรือ "ช่วยน้อยมาก" ที่ต้องระวัง

- **View ที่ทำแค่ query ฐานข้อมูลธรรมดา 1-2 ครั้งแล้ว render template**: ถ้า
  database server อยู่ในเครือข่ายเดียวกัน (low latency) ประโยชน์ของ async
  แทบไม่ต่างจาก sync เลย เพราะเวลาที่ "รอ" สั้นมากอยู่แล้ว
- **View ที่ประมวลผลข้อมูลหนัก ๆ ใน Python เอง** (loop คำนวณ, string processing
  ขนาดใหญ่): เป็นงาน CPU-bound เต็มตัว — async ไม่ช่วยเลย (ขั้นตอนที่ 729)
- **View ที่ใช้ library ภายนอกที่ยังไม่รองรับ async** (เช่น เรียก `requests`
  แบบ sync ธรรมดาอยู่ใน `async def`): จะ **บล็อก event loop ทั้งหมด** ระหว่าง
  รอ กลายเป็นแย่กว่า sync view เดิมเสียอีก เพราะทำให้ request อื่นทั้งหมดที่
  ใช้ event loop เดียวกันต้องรอไปด้วย

### 722.5 กฎการตัดสินใจอย่างง่าย

```
View ของคุณต้องรอ I/O ภายนอก (API, network, disk) นานพอสมควรหรือไม่?
        │
        ├── ไม่ต้องรอเลย/รอสั้นมาก (query DB local ธรรมดา) → เขียนแบบ sync ตามเดิม พอแล้ว
        │
        └── ต้องรอ I/O ภายนอกนาน หรือต้องรอหลายอย่างพร้อมกัน
                │
                ├── มี library รองรับ async จริง (httpx, asyncpg, aioredis) → เขียนเป็น async def ได้ประโยชน์เต็มที่
                │
                └── มีแต่ library sync (requests, psycopg2 sync mode) → ใช้ sync_to_async ห่อไว้ (ขั้นตอนที่ 724)
                    หรือคงเป็น sync view ไปก่อนจนกว่าจะมี async client ให้ใช้
```

---

## ขั้นตอนที่ 723: Async ORM Support ใน Django 5.x

### 723.1 ภาพรวม: Django เพิ่ม Async ORM มาแบบค่อยเป็นค่อยไป

Django เริ่มเพิ่มเมธอดฝั่ง async ให้กับ QuerySet มาตั้งแต่ Django 4.1 และ
เพิ่มขึ้นเรื่อย ๆ จนครบเกือบทุกเมธอดหลักใน Django 5.x — หลักการตั้งชื่อคือ
**เติมตัวอักษร `a` นำหน้าชื่อเมธอด sync เดิม** (a แทนคำว่า "async"):

| เมธอด Sync (คุ้นเคยจาก Phase 2) | เมธอด Async คู่กัน | ใช้งานอย่างไร |
|---|---|---|
| `Post.objects.get(pk=1)` | `await Post.objects.aget(pk=1)` | ดึง 1 object |
| `Post.objects.create(title="...")` | `await Post.objects.acreate(title="...")` | สร้าง object ใหม่ |
| `Post.objects.get_or_create(...)` | `await Post.objects.aget_or_create(...)` | ดึงหรือสร้างถ้ายังไม่มี |
| `Post.objects.update_or_create(...)` | `await Post.objects.aupdate_or_create(...)` | อัปเดตหรือสร้าง |
| `post.save()` | `await post.asave()` | บันทึก instance |
| `post.delete()` | `await post.adelete()` | ลบ instance |
| `Post.objects.filter(...).delete()` | `await Post.objects.filter(...).adelete()` | ลบทั้ง QuerySet |
| `Post.objects.filter(...).update(...)` | `await Post.objects.filter(...).aupdate(...)` | อัปเดตทั้ง QuerySet |
| `Post.objects.count()` | `await Post.objects.acount()` | นับจำนวน |
| `Post.objects.exists()` | `await Post.objects.aexists()` | เช็คว่ามีอยู่ไหม |
| `Post.objects.first()` / `.last()` | `await Post.objects.afirst()` / `.alast()` | ตัวแรก/ตัวสุดท้าย |
| `Post.objects.latest("field")` | `await Post.objects.alatest("field")` | ล่าสุดตาม field |
| `Post.objects.aggregate(...)` | `await Post.objects.aaggregate(...)` | คำนวณรวม เช่น `Sum`, `Avg` |
| `Post.objects.bulk_create([...])` | `await Post.objects.abulk_create([...])` | สร้างหลาย object พร้อมกัน |
| `Post.objects.bulk_update([...])` | `await Post.objects.abulk_update([...])` | อัปเดตหลาย object พร้อมกัน |

**สิ่งที่ยังไม่รองรับโดยตรงในการ query แบบซับซ้อน**: `filter()`, `exclude()`,
`annotate()`, `order_by()` ยังคงเป็น sync เหมือนเดิม เพราะเมธอดเหล่านี้ **ไม่ได้
แตะฐานข้อมูลจริง** — มันแค่สร้าง QuerySet object ที่ยัง "lazy" อยู่ (ทบทวน
concept lazy evaluation จาก Part 011) การ query จริงจะเกิดขึ้นก็ต่อเมื่อคุณ
เรียกเมธอดที่ลงท้ายด้วย `a` (หรือ iterate มัน) เท่านั้น

### 723.2 เขียน Async View ที่ใช้ Async ORM เต็มรูปแบบ

```python
# blog/views.py
from django.http import JsonResponse

from .models import Post


async def async_post_detail(request, slug):
    try:
        post = await Post.objects.aget(slug=slug, is_published=True)
    except Post.DoesNotExist:
        return JsonResponse({"error": "ไม่พบบทความนี้"}, status=404)

    return JsonResponse({
        "title": post.title,
        "slug": post.slug,
        "published_at": post.published_at.isoformat() if post.published_at else None,
    })


async def async_publish_stats(request):
    total_published = await Post.objects.filter(is_published=True).acount()
    latest_post = await Post.objects.filter(is_published=True).alatest("published_at")

    return JsonResponse({
        "total_published": total_published,
        "latest_title": latest_post.title,
    })
```

### 723.3 `async for` — Iterate QuerySet แบบ Async

การ loop QuerySet ปกติด้วย `for post in Post.objects.all():` **ใช้ไม่ได้**
ใน async context (Django จะโยน `SynchronousOnlyOperation` ทันที) ต้องใช้
`async for` คู่กับ `.aiterator()` แทน:

```python
# blog/views.py
async def async_post_titles(request):
    titles = []
    async for post in Post.objects.filter(is_published=True).aiterator():
        titles.append(post.title)

    return JsonResponse({"titles": titles})
```

`.aiterator()` มีประโยชน์เพิ่มเติมเช่นเดียวกับ `.iterator()` ฝั่ง sync (ทบทวน
Part 071 เรื่อง Large Dataset Handling) — คือไม่โหลดผลลัพธ์ทั้งหมดเข้า memory
พร้อมกัน แต่ดึงมาทีละ chunk จากฐานข้อมูลแทน เหมาะมากเมื่อ QuerySet มีจำนวนแถว
มาก

### 723.4 ความจริงที่ต้องรู้: Async ORM ยังคงวิ่งผ่าน Thread Pool อยู่ดี

**ข้อควรระวังสำคัญที่สุดของขั้นตอนนี้**: ทีมงาน Django เองยอมรับตรง ๆ ว่า
เมธอด `a*` เหล่านี้ **ไม่ได้ทำให้ query ฐานข้อมูลกลายเป็น non-blocking I/O
จริง ๆ** เบื้องหลังมันยังคงเรียก driver ฐานข้อมูลแบบ synchronous เดิม (เช่น
`psycopg2`/`psycopg`) เพียงแต่ **ห่อการเรียกนั้นด้วย `sync_to_async`**
(ขั้นตอนที่ 724) ให้อัตโนมัติ แล้วรันในเธรดแยกจาก thread pool เพื่อไม่ให้
บล็อก event loop หลัก

```
await Post.objects.aget(pk=1)
        │
        ▼
   sync_to_async(  # เกิดขึ้นเบื้องหลังอัตโนมัติ ไม่ต้องเขียนเอง
       Post.objects.get
   )(pk=1)
        │
        ▼
   รันบนเธรดแยกใน thread pool ── query จริงผ่าน psycopg2 (sync driver)
```

**ความหมายเชิงปฏิบัติ**:

| สิ่งที่ Async ORM ให้ | สิ่งที่ Async ORM **ไม่ได้** ให้ |
|---|---|
| Syntax ที่สอดคล้องกัน — เขียน `await` ได้ทั้งไฟล์โดยไม่ต้องสลับ context | ความเร็วของ query เดี่ยว ๆ **ไม่ได้เร็วขึ้น** เมื่อเทียบกับ sync ORM |
| ไม่บล็อก event loop ระหว่างรอ query (request อื่นทำงานต่อได้) | Concurrency ที่แท้จริงระดับ database driver (ต้องรอ driver แบบ `asyncpg` ในอนาคต) |
| ผสมกับการเรียก external API แบบ async อื่น ๆ ในฟังก์ชันเดียวกันได้ลื่นไหล | ประโยชน์เมื่อ View มี query DB เพียงอย่างเดียวไม่มี I/O ภายนอกอื่นเลย (แทบไม่ต่างจาก sync) |

พูดง่าย ๆ คือ: **ใช้ Async ORM เพื่อไม่ให้ query DB บล็อก event loop ในระหว่าง
ที่ View เดียวกันกำลังทำ async I/O อื่นอยู่ด้วย** ไม่ใช่เพื่อหวังว่า query
เดียวจะเร็วขึ้นในตัวมันเอง

---

## ขั้นตอนที่ 724: `sync_to_async`/`async_to_sync` — เชื่อมโค้ด sync กับ async

### 724.1 ทำไมต้องมีสะพานเชื่อมสองทาง

โลกจริงเต็มไปด้วยโค้ดผสม — บาง library รองรับ async บาง library รองรับแค่
sync และโค้ดเก่าในโปรเจกต์ของคุณเองก็อาจเป็น sync ทั้งหมด package
**`asgiref`** (ที่ Django ใช้เป็นแกนกลางของ ASGI มาตั้งแต่ Part 004) มีฟังก์ชัน
สองตัวที่เป็นสะพานเชื่อมระหว่างสองโลกนี้:

| ฟังก์ชัน | ใช้เมื่อไหร่ | ทิศทาง |
|---|---|---|
| `sync_to_async(fn)` | มีฟังก์ชัน **sync** อยู่ในมือ อยากเรียกจาก **async context** | Sync → Async |
| `async_to_sync(fn)` | มีฟังก์ชัน **async** อยู่ในมือ อยากเรียกจาก **sync context** | Async → Sync |

### 724.2 `sync_to_async` — เรียกโค้ด Sync จาก Async View

```python
# blog/services/reports.py
import time


def generate_pdf_report_sync(post_id: int) -> bytes:
    """ฟังก์ชัน sync ที่มาจาก library เก่าที่ยังไม่รองรับ async (สมมติ)"""
    time.sleep(2)  # จำลองงานที่ใช้เวลานาน (I/O หรือเรียก library ภายนอกแบบ sync)
    return b"%PDF-1.4 fake pdf content"
```

```python
# blog/views.py
from asgiref.sync import sync_to_async
from django.http import HttpResponse

from .services.reports import generate_pdf_report_sync


async def async_download_report(request, post_id):
    # ห่อฟังก์ชัน sync ด้วย sync_to_async ก่อน แล้วค่อย await
    pdf_bytes = await sync_to_async(generate_pdf_report_sync)(post_id)
    return HttpResponse(pdf_bytes, content_type="application/pdf")
```

**พารามิเตอร์ `thread_sensitive` ที่ต้องเข้าใจ**:

```python
# thread_sensitive=True (ค่าเริ่มต้น) — รันบนเธรดเดียวกับที่เรียก (thread pool ของ Django)
# ปลอดภัยกว่าสำหรับโค้ดที่แตะ Django ORM หรือ state ที่ผูกกับเธรด
await sync_to_async(generate_pdf_report_sync, thread_sensitive=True)(post_id)

# thread_sensitive=False — รันบน thread pool แยกต่างหาก (asgiref จัดสรรให้)
# เหมาะกับงานที่ไม่แตะ ORM และอยากให้รันขนานกันได้จริงหลายเธรด
await sync_to_async(generate_pdf_report_sync, thread_sensitive=False)(post_id)
```

| ค่า `thread_sensitive` | ใช้เมื่อไหร่ |
|---|---|
| `True` (ค่าเริ่มต้น) | โค้ดที่เรียก Django ORM, หรือใช้ resource ที่ผูกกับ thread เดียว (เช่น connection ฐานข้อมูลบางชนิด) |
| `False` | โค้ดคำนวณล้วน ๆ ที่ไม่แตะ Django, ไฟล์, หรือ library ที่ไม่ thread-safe ข้าม request |

### 724.3 `async_to_sync` — เรียกโค้ด Async จาก Sync Context

สถานการณ์กลับกัน: คุณมีฟังก์ชัน sync (เช่น Django Signal handler ที่ทบทวนจาก
Part 019 หรือ management command) แต่ต้องเรียกฟังก์ชันที่เป็น async (เช่น
`channel_layer.group_send()` ที่เจอไปแล้วใน Part 057 ข้อ 565.4):

```python
# blog/signals.py
from asgiref.sync import async_to_sync
from channels.layers import get_channel_layer
from django.db.models.signals import post_save
from django.dispatch import receiver

from .models import Post


@receiver(post_save, sender=Post)
def notify_new_post(sender, instance, created, **kwargs):
    """Signal handler เป็น sync โดยธรรมชาติ (Django signal ไม่รองรับ async handler โดยตรง)
    แต่ channel_layer.group_send() เป็น coroutine — ต้องห่อด้วย async_to_sync"""
    if not created:
        return

    channel_layer = get_channel_layer()
    async_to_sync(channel_layer.group_send)(
        "posts_feed",
        {"type": "post_published", "title": instance.title},
    )
```

`async_to_sync` ทำงานโดย**สร้าง event loop ใหม่ชั่วคราวขึ้นมารันฟังก์ชัน async
นั้นจนจบ แล้วคืนผลลัพธ์กลับมาแบบ blocking** — เหมาะกับจุดที่ยังไงก็ต้องรอผล
อยู่แล้วในโค้ด sync (เช่น signal handler ที่ Django เรียกแบบ synchronous เสมอ)

### 724.4 ตารางสรุปกฎการเลือกใช้

| สถานการณ์ | เครื่องมือที่ใช้ |
|---|---|
| อยู่ใน `async def` view แล้วต้องเรียก Django ORM ตรง ๆ (ไม่ใช้ `a*` methods) | `sync_to_async(thread_sensitive=True)` หรือใช้ Async ORM ตรง ๆ (ขั้นตอนที่ 723) แทน |
| อยู่ใน `async def` view แล้วต้องเรียก library sync ทั่วไป (ไม่ใช่ ORM) | `sync_to_async(thread_sensitive=False)` |
| อยู่ใน sync signal handler / management command แล้วต้องเรียก async API (Channel Layer, async library) | `async_to_sync` |
| อยู่ใน `async def` แล้วต้องเรียก async function อื่นตรง ๆ | ไม่ต้องห่ออะไรเลย — `await` ตรง ๆ ได้เลย |
| อยู่ใน sync `def` view แล้วต้องเรียก sync function อื่น | ไม่ต้องห่ออะไรเลย — เรียกตรง ๆ เหมือนเดิม |

> **คำเตือนระดับมืออาชีพ**: การสลับไปมาระหว่าง sync/async บ่อยเกินไปใน View
> เดียว (ห่อแล้วห่ออีกหลายชั้น) มีต้นทุนแฝงด้าน performance จากการสลับเธรด/
> event loop ถ้าพบว่า View หนึ่งต้องห่อ `sync_to_async` มากกว่า 2-3 จุด
> ให้กลับไปทบทวนว่า **View นี้ควรเป็น sync view ธรรมดาไปเลยหรือไม่** เพราะ
> อาจไม่มี I/O-bound มากพอที่จะคุ้มกับความซับซ้อนที่เพิ่มขึ้น (ทบทวนกฎการ
> ตัดสินใจจากขั้นตอนที่ 722.5)

---

## ขั้นตอนที่ 725: เรียก External Async API จาก Django ด้วย `httpx`

### 725.1 ทำไมต้อง `httpx` ไม่ใช่ `requests`

Library `requests` ที่คุ้นเคยกันดี **เป็น synchronous ล้วน ๆ และไม่มีแผนจะ
รองรับ async ในอนาคต** (ตามที่ทีมพัฒนา `requests` ประกาศไว้อย่างเป็นทางการ)
**`httpx`** คือ HTTP client รุ่นใหม่ที่ออกแบบ API ให้คล้าย `requests` มาก
ที่สุดเท่าที่จะทำได้ (ทำให้ migrate ง่าย) แต่รองรับทั้งโหมด sync และ async
ในตัวเดียวกัน:

```bash
pip install httpx
pip freeze > requirements.txt
```

| Library | Sync | Async | API คล้าย requests |
|---|---|---|---|
| `requests` | ✅ | ❌ | (เป็นต้นแบบ) |
| `httpx` | ✅ | ✅ | ✅ มาก |
| `aiohttp` | ❌ | ✅ | ต่างพอสมควร (API เป็นของตัวเอง) |

### 725.2 `httpx.AsyncClient` เบื้องต้น

```python
# blog/services/currency.py
import httpx

EXCHANGE_API_URL = "https://api.exchangerate-api.com/v4/latest/USD"


async def fetch_usd_thb_rate() -> float:
    async with httpx.AsyncClient(timeout=5.0) as client:
        response = await client.get(EXCHANGE_API_URL)
        response.raise_for_status()  # โยน httpx.HTTPStatusError ถ้า status ไม่ใช่ 2xx
        data = response.json()
        return data["rates"]["THB"]
```

`async with httpx.AsyncClient() as client:` สำคัญมาก — เป็น **async context
manager** ที่ดูแลการเปิด/ปิด connection pool ให้อัตโนมัติ (เทียบเท่ากับการใช้
`with requests.Session() as session:` ฝั่ง sync) การสร้าง `AsyncClient` ใหม่
ทุกครั้งที่เรียกฟังก์ชันมีต้นทุนสูงกว่าที่ควร ถ้า View ต้องเรียก external API
บ่อยมาก ควรพิจารณาสร้าง client เดียวใช้ซ้ำตลอดอายุแอป (ดูข้อ 725.5)

### 725.3 View ที่เรียก External API พร้อม Error Handling ครบถ้วน

```python
# blog/views.py
import httpx
from django.http import JsonResponse

from .services.currency import fetch_usd_thb_rate


async def async_currency_widget(request):
    try:
        rate = await fetch_usd_thb_rate()
    except httpx.TimeoutException:
        return JsonResponse(
            {"error": "External API ตอบกลับช้าเกินไป กรุณาลองใหม่"}, status=504
        )
    except httpx.HTTPStatusError as exc:
        return JsonResponse(
            {"error": f"External API คืน error: {exc.response.status_code}"},
            status=502,
        )
    except httpx.RequestError:
        return JsonResponse(
            {"error": "ไม่สามารถเชื่อมต่อ External API ได้ในขณะนี้"}, status=503
        )

    return JsonResponse({"usd_to_thb": rate})
```

ตารางสรุป exception หลักของ `httpx` ที่ต้องดักจับ:

| Exception | เกิดเมื่อไหร่ |
|---|---|
| `httpx.TimeoutException` | รอ response นานเกิน `timeout` ที่กำหนด |
| `httpx.HTTPStatusError` | ได้ response กลับมาแล้ว แต่ status code เป็น 4xx/5xx (ต้องเรียก `.raise_for_status()` เองก่อนถึงจะโยน) |
| `httpx.ConnectError` | เชื่อมต่อ server ปลายทางไม่ได้เลย (DNS ผิด, server ล่ม) |
| `httpx.RequestError` | คลาสแม่ของ error เกี่ยวกับ request ทั้งหมด (ใช้ดักรวมกรณีอื่นที่ไม่ได้ระบุเจาะจง) |

### 725.4 เรียกหลาย API พร้อม Timeout และ Fallback ค่า Default

รูปแบบที่พบบ่อยในงานจริง: ถ้า external API ล่ม **ไม่ควรทำให้หน้าเว็บทั้งหน้า
พังไปด้วย** — ควรมีค่า fallback ให้แสดงแทน:

```python
# blog/services/currency.py (เพิ่มเข้าไป)
import httpx


async def fetch_usd_thb_rate_with_fallback() -> float:
    try:
        async with httpx.AsyncClient(timeout=3.0) as client:
            response = await client.get(EXCHANGE_API_URL)
            response.raise_for_status()
            return response.json()["rates"]["THB"]
    except httpx.RequestError:
        # ไม่สามารถติดต่อ API ได้ → คืนค่า fallback แทนที่จะทำให้ View ล้ม
        return 36.5  # ค่าประมาณล่าสุดที่เคยบันทึกไว้ (ในงานจริงควรดึงจาก cache)
```

หลักการนี้ควบคู่กับ Caching Framework ที่เรียนไปแล้วใน Part 068-069 —
ในงานจริงควรแคชผลลัพธ์จาก external API ไว้ระยะสั้น ๆ (เช่น 5 นาที) เพื่อไม่
ต้องยิง request ออกไปทุกครั้งที่มีคนเข้าเว็บ ลดทั้ง latency และความเสี่ยงที่
จะโดน rate limit จาก API ภายนอก

### 725.5 ใช้ Client เดียวซ้ำได้ทั้งแอป (Connection Pooling)

```python
# blog/services/http_client.py
import httpx

# สร้าง client เดียวไว้ระดับ module — ใช้ซ้ำได้ตลอดอายุ process
# เพื่อได้ประโยชน์จาก connection pooling / keep-alive เต็มที่
_shared_client: httpx.AsyncClient | None = None


def get_shared_client() -> httpx.AsyncClient:
    global _shared_client
    if _shared_client is None:
        _shared_client = httpx.AsyncClient(timeout=5.0)
    return _shared_client
```

```python
# blog/views.py
from .services.http_client import get_shared_client


async def async_project_showcase_v2(request):
    client = get_shared_client()
    response = await client.get("https://api.github.com/repos/django/django")
    data = response.json()
    return JsonResponse({"stars": data["stargazers_count"]})
```

การสร้าง `AsyncClient` ใหม่ทุก request มีต้นทุนจากการเปิด TCP connection ใหม่
ทุกครั้ง ในขณะที่การใช้ client เดียวซ้ำช่วยให้ httpx นำ connection เดิมกลับมา
ใช้ (keep-alive) ได้เมื่อเรียก host เดิมซ้ำ ๆ — งานจริงระดับ production มักตั้ง
client แบบนี้ไว้ใน `AppConfig.ready()` หรือใช้ dependency injection pattern
ที่ซับซ้อนกว่านี้ แต่หลักการพื้นฐานเหมือนกัน

---

## ขั้นตอนที่ 726: ASGI Server เปรียบเทียบ — Daphne vs Uvicorn vs Hypercorn

### 726.1 ทบทวน: ASGI Server คืออะไร

ทบทวนจาก Part 004 ข้อ 32.6 และ Part 057 ข้อ 561.5 — **ASGI server** คือ
โปรแกรมที่ทำหน้าที่รับ HTTP/WebSocket connection จริงจากเครือข่าย แล้วส่งต่อ
เข้าสู่ `application` callable ใน `config/asgi.py` ของ Django (เทียบเท่า
บทบาทของ Gunicorn ในโลก WSGI) การจะได้ประโยชน์เต็มที่จาก Async View ที่เขียน
มาในขั้นตอนก่อนหน้า **จำเป็นต้องรันด้วย ASGI server จริง ไม่ใช่ `runserver`
แบบ WSGI เดิม**

### 726.2 ตารางเปรียบเทียบ 3 ค่ายหลัก

| คุณสมบัติ | **Daphne** | **Uvicorn** | **Hypercorn** |
|---|---|---|---|
| ผู้พัฒนา | ทีม Django เอง (Django Software Foundation) | ทีม Encode (ผู้สร้าง `httpx`, `starlette`) | ทีม Pallets (ผู้สร้าง Flask/Quart) |
| จุดเด่นดั้งเดิม | ASGI server ตัวแรกของโลก สร้างมาคู่กับ Channels | เน้นความเร็วสูงสุด ใช้ `uvloop` เป็น event loop | รองรับมาตรฐานหลากหลายที่สุด |
| รองรับ WebSocket | ✅ (เป็นจุดแข็งดั้งเดิม) | ✅ | ✅ |
| รองรับ HTTP/2 | ✅ | ✅ (ผ่าน `uvicorn[standard]`) | ✅ (รองรับดีที่สุดในสาม) |
| รองรับ HTTP/3 | ❌ | ⚠️ อยู่ระหว่างพัฒนา | ✅ |
| ความเร็ว (benchmark ทั่วไปสำหรับ HTTP ธรรมดา) | ปานกลาง | **เร็วที่สุด** (ด้วย `uvloop` + `httptools`) | เร็ว (ใกล้เคียง Uvicorn) |
| การใช้งานร่วมกับ Django Channels | เป็น server อ้างอิงหลักที่เอกสาร Channels แนะนำ | ใช้ได้เช่นกัน ตั้งแต่ Channels 3.0+ | ใช้ได้เช่นกัน |
| รูปแบบ process/worker | Twisted-based reactor | `asyncio`/`uvloop` + รองรับหลาย worker ผ่าน Gunicorn | `asyncio`/`trio`/`uvloop` เลือกได้ |
| เหมาะกับ | โปรเจกต์ที่ใช้ Channels เป็นหลัก (WebSocket เยอะ) | โปรเจกต์ Async View ทั่วไปที่เน้นความเร็ว HTTP | โปรเจกต์ที่ต้องการ HTTP/3 หรือความยืดหยุ่นสูงสุด |

### 726.3 ติดตั้งและรันแต่ละตัว

```bash
# Daphne (ติดตั้งไปแล้วตั้งแต่ Part 057 สำหรับ Channels)
pip install daphne
daphne -b 0.0.0.0 -p 8000 config.asgi:application

# Uvicorn
pip install "uvicorn[standard]"
uvicorn config.asgi:application --host 0.0.0.0 --port 8000 --reload

# Hypercorn
pip install hypercorn
hypercorn config.asgi:application --bind 0.0.0.0:8000
```

`uvicorn[standard]` ติดตั้ง dependency เสริม (`uvloop`, `httptools`,
`websockets`) ที่ทำให้ Uvicorn เร็วขึ้นอย่างมีนัยสำคัญเมื่อเทียบกับการติดตั้ง
`uvicorn` เปล่า ๆ — แนะนำให้ติดตั้งแบบมี `[standard]` เสมอสำหรับ production

### 726.4 รันหลาย Worker Process ด้วย Gunicorn + Uvicorn Worker Class

ในโลก production จริง มักไม่รัน `uvicorn` ตรง ๆ เดี่ยว ๆ แต่ใช้ **Gunicorn**
(ที่คุ้นเคยจาก Phase 8/11 ฝั่ง WSGI) เป็นตัวจัดการ process หลายตัว โดยสลับ
worker class ไปใช้ของ Uvicorn แทน — ได้ทั้งความเสถียรของ Gunicorn (process
management, graceful restart) และความเร็วของ Uvicorn (async event loop):

```bash
pip install gunicorn "uvicorn[standard]"

gunicorn config.asgi:application \
    --worker-class uvicorn.workers.UvicornWorker \
    --workers 4 \
    --bind 0.0.0.0:8000
```

รูปแบบนี้เป็นที่นิยมที่สุดในทีมมืออาชีพจำนวนมากสำหรับ deploy Django ที่มี
Async View โดยไม่ได้ใช้ WebSocket หนักมาก (ถ้าใช้ Channels/WebSocket เป็นหลัก
เอกสารทางการยังคงแนะนำ Daphne ตรง ๆ ตามที่ Part 057 ข้อ 569 จะกล่าวถึง)

### 726.5 คำแนะนำสำหรับหลักสูตรนี้

| สถานการณ์ | Server ที่แนะนำ |
|---|---|
| กำลังพัฒนาและทดสอบ Async View บนเครื่อง | `python manage.py runserver` (ใช้ได้ปกติ Django จะจัดการให้อัตโนมัติ) หรือ `uvicorn config.asgi:application --reload` เพื่อทดสอบสภาพแวดล้อมจริงกว่า |
| โปรเจกต์ที่มี Channels/WebSocket เป็นส่วนสำคัญ | Daphne (ตามที่ตั้งค่าไว้แล้วใน Part 057) |
| โปรเจกต์ที่เน้น Async View + REST API เป็นหลัก ไม่มี WebSocket | Gunicorn + `UvicornWorker` |
| ต้องการ HTTP/3 หรือ feature ทดลองใหม่ล่าสุด | Hypercorn |

เราจะกลับมาเจาะลึกเรื่องการ deploy ASGI server จริงบน production พร้อม
reverse proxy (Nginx) และ process manager อีกครั้งใน Phase 11 (DevOps)

---

## ขั้นตอนที่ 727: Middleware ที่รองรับ Async

### 727.1 ทบทวน `RequestTimingMiddleware` จาก Part 066

Part 066 ข้อ 658.2 พาคุณเขียน `RequestTimingMiddleware` แบบ synchronous ไว้
ที่ `config/middleware/timing.py` — ทีนี้ลองสมมติว่าโปรเจกต์เริ่มมี Async View
จำนวนมากขึ้นเรื่อย ๆ (จากขั้นตอนที่ 721-725) คำถามคือ: **Middleware ตัวเดิม
จะยังทำงานถูกต้องไหมเมื่อ View ที่มันครอบเป็น `async def`?**

คำตอบคือ **ทำงานได้** แต่มีต้นทุนแฝง — Django ต้องคอย **"ปรับโหมด" (adapt)**
ระหว่าง sync middleware กับ async view ให้คุยกันรู้เรื่อง ทุกครั้งที่ request
ผ่าน middleware sync ตัวหนึ่งแล้วต้องส่งต่อไปยัง view/middleware ที่เป็น async
Django จะเรียก `sync_to_async`/`async_to_sync` ให้อัตโนมัติเบื้องหลัง ซึ่งมี
overhead จากการสลับเธรดทุกจุดที่ต้องแปลงโหมด

### 727.2 attribute `async_capable` และ `sync_capable`

ทุก Middleware class ใน Django มี class attribute สองตัวที่บอก Django ว่า
middleware ตัวนี้รองรับโหมดไหนบ้าง:

```python
class MyMiddleware:
    async_capable = True   # รองรับการทำงานในสาย async ได้
    sync_capable = True    # รองรับการทำงานในสาย sync ได้
```

| ค่า | ความหมาย |
|---|---|
| `sync_capable = True, async_capable = False` | ค่าเริ่มต้นของ middleware แบบเก่า (ที่สืบทอดจาก `MiddlewareMixin` รุ่นก่อน Django 4.1) — ทำงานได้เฉพาะในสาย sync Django จะ**ห่อให้อัตโนมัติ**เมื่ออยู่ในสาย async แต่เสียค่า adapt ทุกครั้ง |
| `sync_capable = False, async_capable = True` | ทำงานได้เฉพาะสาย async เท่านั้น (พบน้อยในทางปฏิบัติ) |
| `sync_capable = True, async_capable = True` | **แนะนำที่สุด** — middleware ตรวจสอบเองว่าอยู่ในสายไหน แล้วเลือกทำงานแบบที่เหมาะสม ไม่มีค่า adapt เลย |

ตั้งแต่ Django 4.1 เป็นต้นมา `django.utils.deprecation.MiddlewareMixin`
(base class ที่ middleware ส่วนใหญ่ที่เขียนแบบเก่าสืบทอดมา) ถูกอัปเดตให้
รองรับทั้งสองโหมดโดยอัตโนมัติแล้ว แต่ middleware ที่เขียนแบบ **function-based**
หรือเขียน `__call__` เองตรง ๆ (แบบ `RequestTimingMiddleware` ของ Part 066)
ยังคงต้องประกาศ attribute และเขียนโค้ดรองรับทั้งสองโหมดด้วยตัวเอง

### 727.3 อัปเกรด `RequestTimingMiddleware` ให้รองรับทั้ง Sync และ Async

```python
# config/middleware/timing.py (อัปเกรดจาก Part 066 ข้อ 658.2 ให้รองรับ Async)
import logging
import time

from asgiref.sync import iscoroutinefunction, markcoroutinefunction

logger = logging.getLogger("performance")

SLOW_REQUEST_THRESHOLD_MS = 500


class RequestTimingMiddleware:
    """เวอร์ชันที่รองรับทั้ง Sync View (Part 066) และ Async View (Part 073)"""

    async_capable = True
    sync_capable = True

    def __init__(self, get_response):
        self.get_response = get_response
        # ตรวจสอบว่า get_response (view/middleware ถัดไปในสาย) เป็น coroutine หรือไม่
        self._is_coroutine = iscoroutinefunction(self.get_response)
        if self._is_coroutine:
            # บอก Django ว่า instance นี้ (ไม่ใช่แค่ method) ควรถูกเรียกแบบ async
            markcoroutinefunction(self)

    def __call__(self, request):
        if self._is_coroutine:
            return self.__acall__(request)
        return self._sync_call(request)

    def _sync_call(self, request):
        start_time = time.perf_counter()
        response = self.get_response(request)
        self._log_if_slow(request, response, start_time)
        return response

    async def __acall__(self, request):
        start_time = time.perf_counter()
        response = await self.get_response(request)
        self._log_if_slow(request, response, start_time)
        return response

    def _log_if_slow(self, request, response, start_time):
        duration_ms = (time.perf_counter() - start_time) * 1000
        if duration_ms > SLOW_REQUEST_THRESHOLD_MS:
            logger.warning(
                "Slow request: %s %s took %.1fms",
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
```

โครงสร้างนี้คือ**รูปแบบมาตรฐานที่เอกสารทางการของ Django แนะนำ**สำหรับเขียน
middleware ที่รองรับทั้งสองโหมด — จุดสำคัญคือ:

1. ตรวจสอบตอน `__init__` **ครั้งเดียว** ว่า `get_response` เป็น async หรือไม่
   (ไม่ต้องเช็คซ้ำทุก request เพราะ chain ของ middleware ถูกกำหนดตายตัวตอน
   Django เริ่มทำงาน)
2. เรียก `markcoroutinefunction(self)` เพื่อบอก Django ว่า **instance นี้**
   ควรถูก treat เป็น coroutine เมื่ออยู่ในสาย async
3. แยกโค้ด logic จริง (`_log_if_slow`) ออกมาเป็นเมธอดกลางที่ทั้งสองเวอร์ชัน
   เรียกร่วมกันได้ ลดการเขียนโค้ดซ้ำซ้อน

`MIDDLEWARE` ใน `settings.py` **ไม่ต้องแก้ไขอะไรเลย** ยังคงอยู่ตำแหน่งเดิม
จาก Part 066:

```python
# config/settings.py (ไม่เปลี่ยนแปลงจาก Part 066)
MIDDLEWARE = [
    "config.middleware.timing.RequestTimingMiddleware",
    "django.middleware.security.SecurityMiddleware",
    # ... middleware อื่น ๆ ตามเดิม ...
]
```

### 727.4 ตารางสรุปสถานะ Async Support ของ Built-in Middleware

ตั้งแต่ Django 4.1 เป็นต้นมา middleware ที่มากับ Django เองเกือบทั้งหมดถูก
อัปเดตให้รองรับทั้งสองโหมดแล้ว:

| Middleware | รองรับ Async |
|---|---|
| `SecurityMiddleware` | ✅ |
| `SessionMiddleware` | ✅ |
| `CommonMiddleware` | ✅ |
| `CsrfViewMiddleware` | ✅ |
| `AuthenticationMiddleware` | ✅ |
| `MessageMiddleware` | ✅ |
| `XFrameOptionsMiddleware` | ✅ |
| `GZipMiddleware` | ✅ |
| `LocaleMiddleware` | ✅ |

ด้วยเหตุนี้ ถ้าโปรเจกต์ของคุณใช้แค่ middleware built-in ทั้งหมด (ไม่มี
custom middleware แบบเก่าที่ยังไม่รองรับ async) การเพิ่ม Async View เข้าไป
จะแทบไม่มี overhead จากการ adapt เลย — ปัญหาจะเกิดก็ต่อเมื่อมี **third-party
middleware เก่า** หรือ **custom middleware ที่ยังไม่ได้อัปเกรด** (แบบก่อน
727.3) ปะปนอยู่ในสาย

### 727.5 ตรวจสอบว่า Middleware สาย async มีค่า Adapt กี่จุด

```bash
python manage.py check --deploy
```

Django จะไม่ฟ้อง warning เรื่องนี้โดยตรง แต่คุณสามารถสังเกตได้จาก log ระดับ
`DEBUG` ของ `django.request` หรือใช้ APM (Part 066 ข้อ 657) วัดเวลาที่หายไป
ระหว่าง middleware แต่ละชั้นเพื่อหาว่าจุดไหนมี adapt overhead สูงผิดปกติ

---

## ขั้นตอนที่ 728: การเขียน Test สำหรับ Async View

### 728.1 `AsyncClient` — คู่หูของ `Client` ฝั่ง Async

Django มี `django.test.AsyncClient` เป็นเวอร์ชัน async ของ `Client` ที่ใช้
เขียนเทสต์มาตลอดหลักสูตร (ทบทวน Phase 4) ตั้งแต่ **Django 4.2** เป็นต้นมา
`TestCase` ทุกตัวมี attribute **`self.async_client`** ให้ใช้ได้ทันทีโดยไม่
ต้อง import หรือสร้างเองเลย:

```python
# blog/tests/test_async_views.py
from django.test import TestCase

from blog.models import Post


class AsyncViewTests(TestCase):
    async def test_async_ping_returns_200(self):
        response = await self.async_client.get("/blog/async-ping/")

        self.assertEqual(response.status_code, 200)
        self.assertEqual(response.content, b"pong (async)")
```

สังเกตสองจุดสำคัญ:

1. `async def test_...` — Django รองรับการนิยาม test method เป็น coroutine
   ได้โดยตรงตั้งแต่ Django 4.1 โดยไม่ต้องใช้ `IsolatedAsyncioTestCase` จาก
   `unittest` เลย ยังคงสืบทอดจาก `django.test.TestCase` ตามปกติ
2. `await self.async_client.get(...)` — ต้อง `await` เสมอเพราะ `AsyncClient`
   คืน coroutine ไม่ใช่ response object ตรง ๆ แบบ `Client` ปกติ

### 728.2 เทสต์ Async View ที่ใช้ Async ORM ร่วมด้วย

```python
# blog/tests/test_async_views.py (เพิ่มเข้าไป)
class AsyncOrmViewTests(TestCase):
    async def test_async_post_detail_found(self):
        post = await Post.objects.acreate(
            title="ทดสอบ Async View",
            slug="test-async-view",
            content="เนื้อหาทดสอบ",
            is_published=True,
        )

        response = await self.async_client.get(f"/blog/async/{post.slug}/")

        self.assertEqual(response.status_code, 200)
        self.assertEqual(response.json()["title"], "ทดสอบ Async View")

    async def test_async_post_detail_not_found(self):
        response = await self.async_client.get("/blog/async/no-such-slug/")

        self.assertEqual(response.status_code, 404)
```

**ทำไมใช้ `await Post.objects.acreate(...)` แทน `Post.objects.create(...)`
ธรรมดา**: เมื่ออยู่ใน `async def test_...` โค้ดของคุณกำลังรันอยู่บน event
loop จริงที่ Django สร้างขึ้นเพื่อรัน test method นั้น การเรียก ORM แบบ sync
ตรง ๆ ในบริบทนี้มีความเสี่ยงจะชนกับกฎ `SynchronousOnlyOperation` เช่นเดียวกับ
ในขั้นตอนที่ 723 — การใช้เมธอด `a*` ให้สม่ำเสมอทั้งใน view และใน test จึง
ปลอดภัยและสอดคล้องกันที่สุด

### 728.3 เทสต์ Middleware แบบ Async ที่เขียนไว้ในขั้นตอนที่ 727

```python
# config/tests/test_timing_middleware.py
from django.test import TestCase


class TimingMiddlewareAsyncTests(TestCase):
    async def test_async_view_gets_timed_without_error(self):
        # แค่ยืนยันว่า middleware ที่รองรับ async (ข้อ 727.3) ไม่ทำให้ request ล้ม
        response = await self.async_client.get("/blog/async-ping/")

        self.assertEqual(response.status_code, 200)
```

เทสต์นี้อาจดูเหมือนไม่ได้ตรวจสอบอะไรมาก แต่มีคุณค่ามาก: **ถ้า middleware
ยังไม่ได้อัปเกรดให้รองรับ async อย่างถูกต้อง (ลืมตั้ง `async_capable` หรือ
ลืมเรียก `markcoroutinefunction`) เทสต์นี้จะช่วยจับ regression ได้ทันที**
ก่อนที่จะไปพังตอน production จริง

### 728.4 เทสต์การเรียก External API แบบ Mock (ไม่ยิง Request จริงตอนรันเทสต์)

การเทสต์ View ที่เรียก `httpx` จริงเป็นความคิดที่ไม่ดี (เทสต์จะช้า ไม่เสถียร
ขึ้นกับ network และ external service อาจล่มระหว่างรัน CI) ต้อง **mock**
การเรียก `httpx.AsyncClient.get` แทน ด้วย `unittest.mock.patch`:

```python
# blog/tests/test_async_views.py (เพิ่มเข้าไป)
from unittest.mock import AsyncMock, patch


class AsyncExternalApiViewTests(TestCase):
    @patch("blog.services.currency.httpx.AsyncClient.get", new_callable=AsyncMock)
    async def test_currency_widget_success(self, mock_get):
        mock_get.return_value.raise_for_status.return_value = None
        mock_get.return_value.json.return_value = {"rates": {"THB": 36.25}}

        response = await self.async_client.get("/blog/currency/")

        self.assertEqual(response.status_code, 200)
        self.assertEqual(response.json()["usd_to_thb"], 36.25)

    @patch(
        "blog.services.currency.httpx.AsyncClient.get",
        new_callable=AsyncMock,
        side_effect=__import__("httpx").TimeoutException("timeout"),
    )
    async def test_currency_widget_timeout(self, mock_get):
        response = await self.async_client.get("/blog/currency/")

        self.assertEqual(response.status_code, 504)
```

จุดสำคัญคือใช้ **`AsyncMock`** (ไม่ใช่ `MagicMock` ธรรมดา) เพราะเมธอดที่ถูก
mock (`AsyncClient.get`) เป็น coroutine — ถ้าใช้ `MagicMock` ธรรมดาไปแทนที่
โค้ดที่ `await` ผลลัพธ์จาก mock จะพังทันทีเพราะ `MagicMock` ไม่ใช่ awaitable

### 728.5 รันเทสต์และตรวจสอบผล

```bash
python manage.py test blog.tests.test_async_views
```

```
Found 5 test(s).
Creating test database for alias 'default'...
System check identified no issues (0 silenced).
.....
----------------------------------------------------------------------
Ran 5 tests in 0.842s

OK
```

Django test runner จัดการรัน async test method ให้อัตโนมัติโดยไม่ต้อง
ตั้งค่าอะไรเพิ่มเติมเลย — เขียนปนกันระหว่าง `def test_...` (sync) และ
`async def test_...` (async) ใน `TestCase` เดียวกันได้ตามปกติ

---

## ขั้นตอนที่ 729: ข้อจำกัดของ Async ใน Python — GIL

### 729.1 GIL คืออะไร

**GIL (Global Interpreter Lock)** คือกลไกภายในของ **CPython** (ตัวแปลภาษา
Python มาตรฐานที่เกือบทุกคนใช้) ที่บังคับว่า **ในเวลาใดเวลาหนึ่ง มีเพียง
เธรดเดียวเท่านั้นที่สามารถรัน Python bytecode ได้** ไม่ว่าเครื่องนั้นจะมี CPU
core กี่ตัวก็ตาม GIL มีมาตั้งแต่ยุคแรกของ Python เพื่อทำให้การจัดการ memory
ภายใน (reference counting) ปลอดภัยจาก race condition โดยไม่ต้องใส่ lock
ละเอียดยิบทุกจุด (ซึ่งจะทำให้โค้ด single-thread ช้าลงมาก)

```
ระบบที่มี 4 CPU core รัน Python threads 4 ตัวพร้อมกัน:

Core 1: [Thread A กำลังถือ GIL รัน bytecode] ← มีแค่เธรดนี้ทำงานได้จริง ณ ขณะนี้
Core 2: [ว่าง — Thread B รอ GIL อยู่]
Core 3: [ว่าง — Thread C รอ GIL อยู่]
Core 4: [ว่าง — Thread D รอ GIL อยู่]

(สลับกันถือ GIL ทีละเธรด ไม่ใช่รันขนานกันจริงในระดับ bytecode)
```

### 729.2 แล้ว `asyncio` หลบ GIL ได้อย่างไร — คำตอบคือ "ไม่ได้หลบ"

จุดที่มือใหม่เข้าใจผิดบ่อยที่สุด: **`asyncio` ไม่ได้ทำให้ Python หลุดพ้นจาก
GIL** — `asyncio` ยังคงรันอยู่บน **เธรดเดียว** เสมอ (single-threaded event
loop) สิ่งที่มันทำคือ **cooperative multitasking**: เมื่อโค้ดเจอ `await` ที่
กำลังรอ I/O (เช่น รอ response จาก network) โค้ดจะ **สมัครใจคืนการควบคุมกลับ
ให้ event loop** เพื่อให้ event loop ไปรัน coroutine ตัวอื่นที่พร้อมทำงานต่อ

```
GIL ล็อกไว้ที่ "ระดับ interpreter" (จำกัดจำนวนเธรดที่รัน bytecode พร้อมกัน)
asyncio ทำงานที่ "ระดับแอปพลิเคชัน" (สลับงานบนเธรดเดียวระหว่างจุดที่รอ I/O)

              GIL: อนุญาตแค่ 1 เธรดรัน bytecode ตลอดเวลา
                              │
        ┌─────────────────────┴─────────────────────┐
        │                                             │
  Threading (หลายเธรดแย่งกันถือ GIL)      asyncio (เธรดเดียว สลับ task กันเอง
  → ยังคงถูกจำกัดโดย GIL อยู่ดี              โดยไม่ต้องแย่ง GIL ข้ามเธรดเลย)
```

นี่คือเหตุผลที่ async ช่วยงาน **I/O-bound** ได้ดีกว่า threading แบบเดิมด้วย
ซ้ำในหลายกรณี (ไม่มี overhead จากการแย่ง lock ข้ามเธรด) แต่ **ไม่ได้แปลว่า
async ทำให้ Python มี "การประมวลผลแบบขนานจริง" (true parallelism)** เพิ่มขึ้น
มาแม้แต่นิดเดียว — มันแค่จัดตารางงานรอ I/O ให้ฉลาดขึ้นบนเธรดเดียวเท่านั้น

### 729.3 ทำไม CPU-bound Task ถึง "พัง" ทั้ง Event Loop

เพราะ event loop ทำงานบน**เธรดเดียว** ถ้า coroutine ตัวใดตัวหนึ่งเริ่มทำงาน
คำนวณหนัก ๆ โดยไม่มีจุด `await` เลยระหว่างทาง **event loop จะไม่มีโอกาสสลับ
ไปทำ coroutine อื่นได้เลย** ทุก request ที่กำลังรออยู่ (แม้จะเป็น I/O-bound
ล้วน ๆ) จะถูกแช่แข็งไปด้วยจนกว่างานคำนวณนั้นจะเสร็จ:

```python
# blog/views.py
# ❌ ตัวอย่างที่ผิดพลาดร้ายแรง — CPU-bound task ใน Async View
async def async_bad_prime_check(request):
    def is_prime(n):
        if n < 2:
            return False
        for i in range(2, int(n**0.5) + 1):
            if n % i == 0:
                return False
        return True

    # คำนวณเลขเฉพาะตัวใหญ่มาก — ใช้เวลานานและไม่มี await เลยตลอดทาง
    result = is_prime(999999999999999989)  # ค้างทั้ง event loop หลายวินาที!

    return JsonResponse({"is_prime": result})
```

ระหว่างที่ `is_prime()` กำลังคำนวณอยู่ (สมมติใช้เวลา 3 วินาที) **request อื่น
ทุกตัวที่วิ่งอยู่บน event loop เดียวกันจะค้างสนิททั้งหมด** แม้ request เหล่านั้น
จะเป็น I/O-bound ล้วน ๆ ที่ปกติควรทำงานได้ลื่นไหลก็ตาม — นี่แย่กว่า sync view
เดิมเสียอีก เพราะ sync view (ที่รันบน thread pool หลายเธรด) อย่างน้อยยังมี
เธรดอื่นว่างให้ request อื่นใช้งานต่อได้ระหว่างรอ

### 729.4 ทางแก้ที่ถูกต้องสำหรับ CPU-bound Task ใน Async View

| ทางเลือก | วิธีใช้ | เหมาะกับ |
|---|---|---|
| `sync_to_async(fn, thread_sensitive=False)` | ย้ายงานคำนวณไปรันบนเธรดแยก (ทบทวนขั้นตอนที่ 724) | งานคำนวณที่ไม่นานมาก (หลักวินาที) และไม่ต้องการรอผลจากคนอื่นด้วย |
| `loop.run_in_executor()` | ย้ายไปรันบน `ThreadPoolExecutor`/`ProcessPoolExecutor` โดยตรง | ต้องการควบคุม executor เอง เช่นจำกัดจำนวนเธรด |
| **ส่งเป็น Background Task ด้วย Celery** | ให้ View แค่ "จอง" งานแล้วคืน response ทันที ส่วนงานหนักไปทำใน worker แยก process ทั้งหมด | งานที่ใช้เวลานาน (นาทีขึ้นไป) หรือไม่จำเป็นต้องรอผลทันที (เจาะลึกเต็มรูปแบบใน **Part 075**) |
| ใช้ `multiprocessing` ตรง ๆ | รันบนหลาย process จริง (หลบ GIL ได้เพราะแต่ละ process มี interpreter ของตัวเอง) | งานคำนวณหนักที่ต้องการ true parallelism และมี CPU core เหลือเฟือ |

```python
# blog/views.py (แก้ไขให้ปลอดภัย)
from asgiref.sync import sync_to_async


def is_prime_sync(n: int) -> bool:
    if n < 2:
        return False
    for i in range(2, int(n**0.5) + 1):
        if n % i == 0:
            return False
    return True


async def async_prime_check_fixed(request):
    # ย้ายงาน CPU-bound ไปรันบนเธรดแยก ไม่บล็อก event loop หลัก
    result = await sync_to_async(is_prime_sync, thread_sensitive=False)(
        999999999999999989
    )
    return JsonResponse({"is_prime": result})
```

**ข้อควรระวัง**: วิธีนี้แก้ปัญหา "event loop ค้าง" ได้ แต่**ไม่ได้ทำให้งาน
คำนวณนั้นเร็วขึ้นเลย** เพราะ GIL ยังคงจำกัดให้มีแค่เธรดเดียวรัน bytecode ได้
ในเวลาเดียวกันอยู่ดี (ทบทวนข้อ 729.1) มันแค่ย้ายภาระไปอยู่ในเธรดที่ไม่ใช่
event loop หลัก ทำให้ request อื่นยังคงตอบสนองได้ตามปกติเท่านั้น ถ้าต้องการ
ความเร็วในการคำนวณจริง ต้องใช้ multiprocessing หรือ Celery worker แยก process
เท่านั้น

### 729.5 ตารางสรุปใหญ่: Async, Threading, Multiprocessing กับ GIL

| แนวทาง | หลบ GIL ได้ไหม | เหมาะกับ I/O-bound | เหมาะกับ CPU-bound |
|---|---|---|---|
| Synchronous ปกติ (thread-per-request แบบ WSGI) | ไม่เกี่ยวข้อง (1 คำสั่งต่อครั้งต่อเธรด) | ปานกลาง (เสียเธรดระหว่างรอ) | ปานกลาง (เธรดอื่นยังทำงานได้ระหว่างรอ GIL) |
| `asyncio`/Async View | ❌ ไม่หลบ (ยังเป็นเธรดเดียว) | ✅ ดีมาก | ❌ แย่มาก (บล็อกทุกอย่าง) |
| `threading` (multi-thread ปกติ) | ❌ ไม่หลบ (แย่ง GIL กัน) | ✅ ดี | ❌ ไม่ช่วย (bytecode ยังรันทีละเธรด) |
| `multiprocessing` | ✅ หลบได้ (แต่ละ process มี GIL ของตัวเอง) | ใช้ได้แต่ overhead สูงเกินจำเป็น | ✅ ดีมาก |
| Celery worker (แยก process) | ✅ หลบได้ | ✅ ดี (เหมาะกับงานที่ไม่ต้องรอผลทันที) | ✅ ดีมาก |

> **หมายเหตุถึงอนาคต**: ชุมชน Python กำลังพัฒนา **PEP 703 (free-threaded
> CPython)** ที่เปิดให้ปิด GIL ได้ในบาง build ทดลอง (เริ่มมีให้ทดลองใช้ตั้งแต่
> Python 3.13 เป็นต้นมา) แต่ ณ ต้นปี 2026 ยังอยู่ในสถานะทดลอง (experimental)
> ไม่ใช่ default ของ CPython และ ecosystem ส่วนใหญ่ (รวม Django และ library
> จำนวนมาก) ยังไม่ได้ทดสอบรองรับอย่างเต็มรูปแบบ หลักการในขั้นตอนนี้ยังคงเป็น
> ความจริงสำหรับ CPython แบบมาตรฐานที่ใช้งานจริงในปัจจุบัน

---

## ขั้นตอนที่ 730: สรุปและแบบฝึกหัด

### 730.1 สรุปสิ่งที่ได้เรียนรู้ใน Part นี้

- ✅ เขียน Async View แรกด้วย `async def` ทั้งแบบ function-based และ class-based
- ✅ เข้าใจกฎการตัดสินใจว่าเมื่อไหร่ควรใช้ async — งาน I/O-bound ได้ประโยชน์
  ชัดเจน ส่วนงาน CPU-bound ไม่ได้ประโยชน์เลย
- ✅ ใช้ Async ORM ของ Django 5.x (`aget`, `acreate`, `afilter`+`async for`,
  `aiterator`) พร้อมเข้าใจว่าเบื้องหลังยังวิ่งผ่าน thread pool อยู่ดี
- ✅ เชื่อมโค้ด sync กับ async ด้วย `sync_to_async`/`async_to_sync` ได้อย่าง
  ถูกต้องตามสถานการณ์
- ✅ เรียก External API แบบ concurrent ด้วย `httpx.AsyncClient` และ
  `asyncio.gather()` พร้อม error handling ครบถ้วน
- ✅ เปรียบเทียบ ASGI server สามค่าย (Daphne, Uvicorn, Hypercorn) และเลือก
  ใช้ให้เหมาะกับสถานการณ์
- ✅ อัปเกรด Middleware เดิมจาก Part 066 ให้รองรับทั้ง Sync และ Async ด้วย
  `async_capable`/`sync_capable`
- ✅ เขียนเทสต์สำหรับ Async View ด้วย `async def test_...` และ
  `self.async_client` รวมถึง mock การเรียก external API ด้วย `AsyncMock`
- ✅ เข้าใจ GIL อย่างถ่องแท้ และรู้ว่าทำไม async ไม่ช่วยงาน CPU-bound พร้อม
  ทางแก้ที่ถูกต้อง (`sync_to_async` แบบไม่ sensitive, multiprocessing, Celery)

### 730.2 Checklist ก่อนไป Part ถัดไป

- [ ] เขียนและทดสอบ Async View อย่างน้อย 1 ตัวที่ทำงานได้จริงบนเครื่องของคุณ
- [ ] เขียน service function ที่เรียก external API 2-3 endpoint พร้อมกันด้วย
      `asyncio.gather()` แล้ววัดเวลาว่าเร็วกว่า sequential จริง
- [ ] แปลง View ที่ใช้ Django ORM ให้ใช้ `aget()`/`afilter()`+`async for` ได้
      โดยไม่มี `SynchronousOnlyOperation`
- [ ] ลองรันโปรเจกต์ด้วย Uvicorn (`uvicorn config.asgi:application --reload`)
      แทน `runserver` อย่างน้อยหนึ่งครั้ง
- [ ] อัปเกรด custom middleware อย่างน้อย 1 ตัวในโปรเจกต์ให้รองรับ async ตาม
      รูปแบบข้อ 727.3
- [ ] เขียนเทสต์ async อย่างน้อย 3 เคส (สำเร็จ, error, timeout) พร้อม mock
      external API ด้วย `AsyncMock`
- [ ] อธิบายได้ด้วยคำพูดตัวเองว่าทำไม CPU-bound task ใน `async def` ถึงทำให้
      ทั้งเซิร์ฟเวอร์ค้าง โดยอ้างอิงถึง GIL และ event loop

### 730.3 แบบฝึกหัดท้ายบท

**แบบฝึกหัดที่ 1**: เขียน Async View ชื่อ `async_multi_source_dashboard` ที่
เรียก external API อย่างน้อย 3 แหล่งพร้อมกันด้วย `asyncio.gather()` (เช่น
GitHub API, Exchange Rate API, และ API สาธารณะอื่นที่คุณเลือกเอง) แล้วรวม
ผลลัพธ์ทั้งหมดเป็น JSON response เดียว ต้องมี error handling ที่ทำให้ถ้า
1 ใน 3 แหล่งล่ม แหล่งอื่นยังคงแสดงผลได้ตามปกติ (ใช้ `asyncio.gather(...,
return_exceptions=True)` ช่วย)

**แบบฝึกหัดที่ 2**: หยิบ Model ใดก็ได้ในโปรเจกต์ Blog ของคุณ เขียน Async View
ที่ทำ CRUD ครบ 4 อย่าง (Create, Read, Update, Delete) โดยใช้ Async ORM
ทั้งหมด (`acreate`, `aget`, `asave`, `adelete`) แล้วเขียนเทสต์ครบทั้ง 4
เมธอดด้วย `async def test_...` และ `self.async_client`

**แบบฝึกหัดที่ 3**: เขียน Custom Middleware ใหม่ (ไม่ใช่ตัวจับเวลา) ที่รองรับ
ทั้ง Sync และ Async ตามรูปแบบข้อ 727.3 เช่น middleware ที่เติม request ID
แบบสุ่มลงใน header `X-Request-ID` ของทุก response แล้วเขียนเทสต์ยืนยันว่า
ทำงานถูกต้องทั้งกับ Sync View และ Async View ในโปรเจกต์เดียวกัน

**แบบฝึกหัดที่ 4 (บังคับ — สรุปของ Part นี้)**: เลือก View ที่มีอยู่แล้วใน
โปรเจกต์ของคุณที่เรียก external API แบบ synchronous (ด้วย `requests` หรือ
`httpx` sync client) แปลงให้เป็น Async View เต็มรูปแบบด้วย `httpx.AsyncClient`
จากนั้นเขียนสคริปต์ benchmark ง่าย ๆ (ใช้ `time.perf_counter()` หรือเครื่องมือ
อย่าง `locust`/`k6` ที่เรียนจาก Part 072) ยิง request ซ้ำ 50-100 ครั้งแบบ
concurrent ไปยังทั้งเวอร์ชัน sync และ async แล้วบันทึกผลเปรียบเทียบ:

- Requests per second (RPS) ของแต่ละเวอร์ชัน
- Response time เฉลี่ยและ p95
- จำนวน worker/thread ที่ใช้ในแต่ละเวอร์ชัน

เขียนสรุปผลเป็นตารางเปรียบเทียบ พร้อมอธิบายว่าผลลัพธ์ที่ได้สอดคล้องกับ
หลักการ I/O-bound vs CPU-bound ในขั้นตอนที่ 722 หรือไม่ อย่างไร

### 730.4 คำถามที่พบบ่อย (FAQ)

**Q: ควรเปลี่ยน View ทั้งหมดในโปรเจกต์เป็น `async def` เลยไหม เพื่อให้ "ทันสมัย"?**
A: ไม่ควรอย่างยิ่ง ทบทวนกฎการตัดสินใจจากขั้นตอนที่ 722.5 — View ที่ทำแค่
query ฐานข้อมูล local ธรรมดาแล้ว render template แทบไม่ได้ประโยชน์จาก async
เลย แถมยังเพิ่มความซับซ้อนโดยไม่จำเป็น (ต้องระวัง `SynchronousOnlyOperation`
ทุกจุดที่แตะ ORM) ใช้ async เฉพาะ View ที่มี I/O-bound work ชัดเจนเท่านั้น

**Q: ถ้าใช้ Async View แล้ว จำเป็นต้องเปลี่ยนไปใช้ ASGI server (Uvicorn/Daphne)
ทันทีไหม หรือยังใช้ Gunicorn + WSGI เดิมได้?**
A: Async View ยังคงรันได้บน WSGI server ปกติ (Django ห่อด้วย `async_to_sync`
ให้อัตโนมัติตามตารางในข้อ 721.3) แต่จะ**เสียประโยชน์ของ async ไปเกือบทั้งหมด**
เพราะสุดท้ายก็ยังคงบล็อกเธรดเหมือน sync view อยู่ดี ถ้าต้องการประโยชน์เต็มที่
จำเป็นต้องรันด้วย ASGI server (ขั้นตอนที่ 726) เท่านั้น

**Q: `httpx` กับ `aiohttp` ต่างกันอย่างไร ทำไม Part นี้เลือก `httpx`?**
A: ทั้งคู่เป็น async HTTP client ที่ใช้งานได้ดีทั้งคู่ แต่ `httpx` มี API ที่
ออกแบบให้คล้าย `requests` มากที่สุด (ทีมงานตั้งใจให้ migrate ง่าย) และรองรับ
ทั้งโหมด sync และ async ในไลบรารีเดียวกัน ในขณะที่ `aiohttp` รองรับแค่ async
อย่างเดียวและมี API เป็นของตัวเองที่ต่างจาก `requests` พอสมควร หลักสูตรนี้
เลือก `httpx` เพื่อความคุ้นเคยและความยืดหยุ่นที่มากกว่า

**Q: Async ORM ของ Django จะ "เร็วขึ้นจริง" ในอนาคตไหม เมื่อมี async database
driver รองรับเต็มรูปแบบ?**
A: มีความเป็นไปได้สูง ทีมงาน Django กำลังพัฒนาไปในทิศทางนั้น (มีการพูดคุย
เรื่องรองรับ driver อย่าง `psycopg` โหมด async เต็มรูปแบบมากขึ้นเรื่อย ๆ ใน
แต่ละเวอร์ชัน) แต่ ณ Django 5.x ที่หลักสูตรนี้ใช้ ข้อเท็จจริงในข้อ 723.4 ยังคง
เป็นความจริงอยู่ — Async ORM ให้ความสะดวกด้าน syntax และป้องกันการบล็อก event
loop เป็นหลัก ไม่ใช่ความเร็วของ query เดี่ยว ๆ ที่เพิ่มขึ้น

**Q: ทำไม Signal handler ใน Django ถึงยังเป็น sync เสมอ ทั้งที่ตอนนี้มี async
ORM แล้ว?**
A: Django Signals (ทบทวน Part 019) ยังไม่รองรับ async receiver โดยตรงใน
เวอร์ชันปัจจุบันของหลักสูตรนี้ — ถ้า signal handler ต้องเรียกโค้ด async (เช่น
`channel_layer.group_send()` ในข้อ 724.3) ต้องห่อด้วย `async_to_sync` เสมอ
ตามรูปแบบที่แสดงไว้ นี่คือจุดที่สะพานเชื่อม sync/async ยังจำเป็นอยู่มากใน
โค้ด Django จริง

---

## เตรียมตัวสำหรับ Part ถัดไป

**Part 074: Django Channels ขั้นสูง: Consumers และ Groups** จะพาคุณกลับไปที่
Django Channels ที่ปูพื้นไว้ใน Part 057 อีกครั้ง แต่คราวนี้เจาะลึกแบบเต็ม
รูปแบบ — ใช้ความรู้เรื่อง `async def`, `sync_to_async`/`async_to_sync`, และ
`database_sync_to_async` ที่เพิ่งเรียนใน Part นี้มาเขียน Consumer ที่ซับซ้อน
ขึ้น จัดการ Groups หลายแบบพร้อมกัน (ห้องแชทหลายห้อง, การแจ้งเตือนแบบ
per-user), จัดการ reconnect และ connection lifecycle อย่างมืออาชีพ ก่อนที่
Part 075 จะเปลี่ยนโฟกัสไปที่ **Celery** สำหรับงานเบื้องหลังที่ไม่ต้องการ
คำตอบทันที (ต่างจาก Async View ใน​Part นี้ที่ยังคงต้องรอผลก่อนตอบ response
กลับไปเสมอ) เตรียมทบทวนเรื่อง Channel Layer และ Redis จาก Part 057 ข้อ
565 ไว้ให้แม่น เพราะ Part 074 จะต่อยอดจากจุดนั้นทันที
