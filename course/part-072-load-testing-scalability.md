# Part 072: Load Testing และ Scalability Testing

> **ขั้นตอนที่ 711-720 ของหลักสูตร** | Phase 8: Performance & Caching (Part สุดท้ายของ Phase)
>
> เป้าหมายของ Part นี้: ปิดท้าย Phase 8 ด้วยทักษะที่ตอบคำถามสำคัญที่สุดก่อนระบบขึ้น
> Production จริง — **"ระบบของเรารองรับผู้ใช้พร้อมกันได้กี่คนกันแน่?"** คุณจะเรียนรู้
> การเขียน Load Test Script ด้วย **Locust** (Python) และเปรียบเทียบกับ **k6**
> (JavaScript) ทั้งสองเครื่องมือ, วิธีหา **Breaking Point** ของระบบผ่านตัวเลข
> requests/sec และ response time percentiles (p50/p95/p99), การคำนวณขนาด
> **Database Connection Pool** ที่เหมาะสมด้วย PgBouncer, สูตรคำนวณจำนวน **Gunicorn
> Worker** ที่เหมาะสมทั้งแบบ sync และ async, แนวคิด **Horizontal vs Vertical
> Scaling**, และที่สำคัญที่สุดคือการ**วัดผลจริง**ว่า Caching ที่ทำมาใน Part 068-069
> ช่วยเพิ่ม throughput ได้มากแค่ไหนด้วยตัวเลขก่อน/หลังที่จับต้องได้ ปิดท้ายด้วยการ
> **สรุปภาพรวมทั้ง Phase 8** (Part 066-072) พร้อม Quiz 12 ข้อและแบบฝึกหัดใหญ่ที่ให้คุณ
> ทำ **Performance Audit เต็มรูปแบบ** ของโปรเจกต์ Blog ตั้งแต่ profile → optimize
> query → เพิ่ม caching → วัดผลด้วย load test ก่อน/หลัง ก่อนจะแนะนำ Phase 9
> (Async, Celery, Channels) ที่รอคุณอยู่ใน Part 073

---

## สารบัญของ Part นี้

- ขั้นตอนที่ 711: ทำไม Load Testing สำคัญก่อนขึ้น Production จริง
- ขั้นตอนที่ 712: ติดตั้งและเขียน Load Test Script ด้วย Locust
- ขั้นตอนที่ 713: `k6` เป็นทางเลือก (เขียนด้วย JavaScript) เปรียบเทียบกับ Locust
- ขั้นตอนที่ 714: การหาจุดแตกหัก (Breaking Point) ของระบบ — requests/sec, response time percentiles (p50/p95/p99)
- ขั้นตอนที่ 715: ขนาด Database Connection Pool ที่เหมาะสมภายใต้ load สูง
- ขั้นตอนที่ 716: การปรับจูน Gunicorn Worker — sync vs async worker, สูตรคำนวณจำนวน worker ที่เหมาะสม
- ขั้นตอนที่ 717: แนวคิด Horizontal Scaling vs Vertical Scaling
- ขั้นตอนที่ 718: เปรียบเทียบผล Load Test ก่อน/หลังเปิดใช้ Caching
- ขั้นตอนที่ 719: เกริ่นการทำ Continuous Performance Testing ใน CI
- ขั้นตอนที่ 720: สรุป Phase 8 ทั้งหมด (Part 066-072) + Quiz + แบบฝึกหัดใหญ่ปิดท้าย Phase

---

## ขั้นตอนที่ 711: ทำไม Load Testing สำคัญก่อนขึ้น Production จริง

### 711.1 ตลอด Phase 8 เราวัด "เร็วแค่ไหน" แต่ยังไม่เคยวัด "รับได้กี่คนพร้อมกัน"

Part 066-071 สอนให้คุณทำให้ **1 request เดียว** เร็วขึ้น: profiling หา bottleneck
(066), ลด N+1 query ด้วย `select_related`/`prefetch_related` (067), cache
ผลลัพธ์ที่คำนวณซ้ำบ่อย (068-069), เพิ่ม index ให้ query เร็วขึ้น (070), และแบ่งหน้า
ข้อมูลก้อนใหญ่ (071) — ทุกเทคนิคเหล่านี้ตอบคำถามว่า **"ใช้เวลากี่มิลลิวินาที"**

แต่คำถามที่สำคัญไม่แพ้กันและไม่เคยถูกตอบมาก่อนคือ **"ถ้ามี 500 คนกดเข้าเว็บพร้อมกัน
ระบบยังตอบสนองได้ดีอยู่ไหม?"** นี่คือคำถามที่ **Load Testing** ตอบ และเป็นคำถามที่
unit test/integration test จาก Phase 7 (Part 059-065) **ตอบไม่ได้เลย** เพราะ test
เหล่านั้นรันทีละ request เดียวเสมอ

### 711.2 ตารางแยกประเภทการทดสอบด้าน Performance

| ประเภท | คำถามที่ตอบ | วิธีทดสอบ | ใช้เมื่อไหร่ |
|---|---|---|---|
| **Load Testing** | ระบบรองรับ traffic ระดับที่คาดหวังได้จริงไหม (เช่น 200 concurrent users) | จำลอง traffic ที่ระดับเป้าหมายคงที่ ดูว่า response time/error rate ยังอยู่ในเกณฑ์ที่ยอมรับได้ | ก่อน deploy ทุกครั้งที่คาดว่า traffic จะเพิ่ม |
| **Stress Testing** | ระบบพังที่จุดไหน (Breaking Point) | เพิ่ม traffic ขึ้นเรื่อย ๆ จนระบบเริ่ม error/ช้าผิดปกติ | วางแผน capacity, หา bottleneck ตัวจริง |
| **Spike Testing** | ระบบรับมือกับ traffic ที่พุ่งขึ้นกะทันหันได้ไหม (เช่น โพสต์ไวรัลถูกแชร์) | ยิง traffic จากศูนย์ขึ้นไปสูงมากในเวลาสั้น ๆ แล้วลดกลับ | เตรียมรับ flash sale, ข่าวไวรัล, promotion |
| **Soak Testing (Endurance)** | ระบบมี memory leak หรือ resource ค่อย ๆ หมดไปเมื่อรันนาน ๆ ไหม | ยิง traffic ระดับปานกลางต่อเนื่องหลายชั่วโมงถึงหลายวัน | ก่อนขึ้น production ระบบที่ต้องรันตลอด 24/7 |

Part นี้จะเน้น **Load Testing** และ **Stress Testing** เป็นหลัก (ขั้นตอนที่ 712-714)
เพราะเป็นสองแบบที่ทีมพัฒนาส่วนใหญ่ต้องทำก่อน deploy ทุกครั้งที่มีการเปลี่ยนแปลงสำคัญ

### 711.3 ตัวอย่างความเสียหายจริงเมื่อไม่ทำ Load Testing

ในโลกจริง ระบบที่ผ่าน unit test 100% และทำงานถูกต้องสมบูรณ์แบบตอน demo ให้ทีมดู
สามารถล่มได้ทันทีเมื่อเจอ traffic จริง ตัวอย่างสถานการณ์ที่พบบ่อยในระบบ e-commerce/
ตั๋วจองที่พัก:

- **เปิดขายตั๋วคอนเสิร์ต**: ทุกคนกดปุ่ม "ซื้อ" พร้อมกันในวินาทีเดียวกัน ถ้าไม่เคย
  load test มาก่อน ระบบอาจ hang เพราะ database connection หมด (ขั้อ 715) หรือ
  Gunicorn worker ไม่พอ (ขั้อ 716) ทำให้ request ค้างอยู่ใน queue จนหมด timeout
- **โพสต์กลายเป็นไวรัล**: หน้ารายละเอียดบทความหนึ่งถูกแชร์กระจายในโซเชียล ทำให้
  traffic เพิ่มขึ้น 50 เท่าในเวลาไม่กี่นาที ถ้าไม่มี caching (Part 068) รองรับ
  ฐานข้อมูลจะถูก query ซ้ำ ๆ จนล่มทั้งระบบ ทั้งที่หน้าอื่นของเว็บไซต์ไม่มีปัญหาเลย
- **Batch job รันพร้อม peak hour**: งาน background (เช่น ส่งอีเมลสรุปรายวัน) ที่ไป
  แย่ง database connection pool ในช่วงเวลาที่ user ใช้งานเยอะที่สุดพอดี ทำให้ทั้ง
  งาน background และ request ของ user ช้าลงพร้อมกัน

ทุกกรณีข้างต้น**ตรวจจับไม่ได้เลย**ด้วย unit test หรือแม้แต่การทดสอบด้วยมือคนเดียว
บนเครื่อง dev — ต้องอาศัย **การจำลอง load จริง** เท่านั้นถึงจะเห็นปัญหาก่อนที่ user
จริงจะเจอ

### 711.4 Metric หลักที่ต้องวัดเสมอเมื่อทำ Load Test

| Metric | ความหมาย | ทำไมสำคัญ |
|---|---|---|
| **Throughput (RPS/TPS)** | จำนวน Request/Transaction ที่ระบบประมวลผลสำเร็จต่อวินาที | บอกว่าระบบ "รับงานได้เร็วแค่ไหน" ในภาพรวม |
| **Response Time (Latency)** | เวลาที่ใช้ตั้งแต่ส่ง request จนได้ response กลับ | ผู้ใช้รู้สึกได้โดยตรง — ทวนจากขั้อ 714 ว่าทำไมต้องดูเป็น percentile ไม่ใช่ค่าเฉลี่ย |
| **Error Rate** | สัดส่วน request ที่ล้มเหลว (5xx, timeout, connection refused) | ตัวชี้วัดที่ตรงไปตรงมาที่สุดว่าระบบ "พัง" หรือยัง |
| **Concurrency (Virtual Users)** | จำนวนผู้ใช้ที่ "กำลังทำงานพร้อมกัน" ในการทดสอบ | ตัวแปรที่เราค่อย ๆ เพิ่มขึ้นเพื่อหา breaking point |
| **Resource Saturation** | CPU/Memory/DB connections ของ server ที่ใช้ไป (%) | บอกว่า bottleneck ที่แท้จริงอยู่ตรงไหน (app server, database, network) |

### 711.5 Load Testing ต้องรันบน Environment ที่ใกล้เคียง Production ที่สุด

ข้อผิดพลาดที่พบบ่อยที่สุดของทีมที่เพิ่งเริ่มทำ load testing คือรันทดสอบบนเครื่อง
`runserver` ของ Django เอง (development server) ซึ่ง**ไม่ได้ออกแบบมาให้รองรับ
concurrent request หลายพันตัวเลย** และให้ผลลัพธ์ที่บิดเบือนจนใช้อ้างอิงไม่ได้

**กฎเหล็กของ Load Testing**: ต้องทดสอบผ่าน **staging environment** ที่ใช้ Gunicorn/
uWSGI จริง (ทวนจากขั้อ 716), เชื่อมกับ PostgreSQL จริง (ไม่ใช่ SQLite), ผ่าน Nginx
reverse proxy จริง, และควรมีสเปกเครื่องใกล้เคียง production ให้มากที่สุด ไม่เช่นนั้น
ตัวเลขที่วัดได้จะไม่มีความหมายใด ๆ เมื่อเทียบกับสิ่งที่จะเกิดขึ้นจริง

---

## ขั้นตอนที่ 712: ติดตั้งและเขียน Load Test Script ด้วย Locust

### 712.1 Locust คืออะไร

**Locust** คือเครื่องมือ Load Testing แบบ Open Source ที่เขียน test script ด้วย
**Python ล้วน** (ต่างจากเครื่องมือรุ่นเก่าอย่าง JMeter ที่ใช้ XML/GUI) จุดเด่นคือ:

- เขียน scenario เป็นโค้ด Python ทำให้ใช้ logic ซับซ้อนได้ (if/else, loop, สุ่มค่า)
- รองรับการรันแบบ **Distributed** (master แจกงานให้ worker หลายเครื่อง) เพื่อจำลอง
  traffic ระดับหลักแสน request ได้
- มี Web UI แบบ real-time ให้ดูกราฟระหว่างทดสอบ
- ใช้โมเดล **Greenlet** (ผ่าน `gevent`) ทำให้ virtual user นับพันรันได้ในเครื่องเดียว
  โดยไม่ต้องใช้ thread จริงจำนวนมาก

### 712.2 ติดตั้ง Locust

```bash
pip install locust
locust --version
```

เพิ่มใน `requirements-dev.txt` (แยกจาก `requirements.txt` ของ production เพราะ
Locust เป็นเครื่องมือทดสอบ ไม่ใช่ dependency ของแอปจริง):

```
# requirements-dev.txt
locust==2.31.0
```

### 712.3 โครงสร้าง Locust Test พื้นฐาน

```python
# loadtests/locustfile.py
from locust import HttpUser, task, between


class BlogVisitorUser(HttpUser):
    """จำลองผู้เข้าชมเว็บบล็อกทั่วไปที่ยังไม่ login"""

    # เวลาสุ่มพักระหว่างแต่ละ task (จำลองพฤติกรรมคนอ่านจริง ไม่ใช่ยิงรัว ๆ ไม่หยุด)
    wait_time = between(1, 5)

    @task(10)
    def view_post_list(self):
        """หน้ารายการบทความ — เข้าบ่อยที่สุด ให้ weight สูงสุด"""
        self.client.get("/blog/", name="/blog/ [list]")

    @task(6)
    def view_post_detail(self):
        """หน้ารายละเอียดบทความ — สุ่มเลือก slug จากชุดที่เตรียมไว้ล่วงหน้า"""
        slug = self.random_slug()
        self.client.get(f"/blog/{slug}/", name="/blog/[slug]/ [detail]")

    @task(2)
    def view_category_filter(self):
        """หน้ากรองตามหมวดหมู่ — ใช้บ่อยน้อยกว่าหน้าแรก"""
        self.client.get("/blog/?category=django", name="/blog/?category=... [filter]")

    def random_slug(self):
        import random
        # slug ตัวอย่างที่มีอยู่จริงในฐานข้อมูลทดสอบ (seed ไว้ล่วงหน้าก่อนรัน load test)
        slugs = ["django-tips", "python-async-basics", "postgresql-indexing-101"]
        return random.choice(slugs)
```

จุดสำคัญของโค้ดข้างต้น: `name="..."` ใน `self.client.get()` ใช้**รวมสถิติของ URL
ที่มีตัวแปร** (เช่น `/blog/django-tips/`, `/blog/python-async-basics/`) เข้าเป็นแถว
เดียวในรายงานผล — ถ้าไม่ใส่ `name` Locust จะแยกสถิติของแต่ละ slug ออกจากกันจนตาราง
ผลลัพธ์อ่านไม่รู้เรื่องเลย (เป็นกับดักคลาสสิกของมือใหม่ Locust)

### 712.4 จำลอง User ที่ Login และมี Session

```python
# loadtests/locustfile.py (ต่อจากเดิม)
from locust import HttpUser, task, between, SequentialTaskSet


class AuthenticatedAuthorFlow(SequentialTaskSet):
    """
    SequentialTaskSet บังคับให้ task รันตามลำดับที่เขียนไว้เป๊ะ ๆ ต่างจาก @task ปกติ
    ที่ Locust จะสุ่มเลือกลำดับเอง — ใช้เมื่อ scenario มีลำดับขั้นตอนที่ต้องพึ่งพากัน
    เช่น ต้อง login ก่อนถึงจะโพสต์คอมเมนต์ได้
    """

    def on_start(self):
        """รันครั้งเดียวตอนเริ่ม TaskSet นี้ — ใช้ login เพื่อได้ session cookie"""
        response = self.client.get("/accounts/login/")
        csrf_token = response.cookies.get("csrftoken")

        self.client.post(
            "/accounts/login/",
            data={
                "username": "loadtest_author",
                "password": "LoadTest123!",
                "csrfmiddlewaretoken": csrf_token,
            },
            headers={"Referer": self.client.base_url + "/accounts/login/"},
        )

    @task
    def view_dashboard(self):
        self.client.get("/dashboard/", name="/dashboard/")

    @task
    def post_comment(self):
        csrf_token = self.client.cookies.get("csrftoken")
        self.client.post(
            "/blog/django-tips/comments/",
            data={
                "text": "ความคิดเห็นทดสอบจาก Load Test",
                "csrfmiddlewaretoken": csrf_token,
            },
            name="/blog/[slug]/comments/ [POST]",
        )

    def on_stop(self):
        """รันครั้งเดียวตอน TaskSet นี้จบ — logout ให้เรียบร้อย"""
        self.client.get("/accounts/logout/")


class AuthenticatedAuthorUser(HttpUser):
    tasks = [AuthenticatedAuthorFlow]
    wait_time = between(2, 8)
```

**เรื่อง CSRF ที่ต้องจัดการเสมอ**: `HttpUser` ของ Locust ใช้ `requests.Session`
ภายใน ซึ่ง**เก็บ cookie ให้อัตโนมัติ**ข้ามแต่ละ request เหมือน browser จริง (ทวน
แนวคิดเดียวกับ `client.login()` ของ Django test client จาก Part 050 ขั้อ 492.4)
แต่ Locust **ไม่รู้จัก** Django CSRF token โดยอัตโนมัติ ต้องดึงจาก cookie เอง
แล้วแนบไปกับทุก POST request ด้วยมือเสมอ — ถ้าลืมขั้นตอนนี้ ทุก POST จะได้
`403 Forbidden` และผลการทดสอบจะผิดเพี้ยนไปทั้งหมด

### 712.5 ให้ Locust หลาย User Class ทำงานพร้อมกันในสัดส่วนที่ต้องการ

```python
# loadtests/locustfile.py (ต่อจากเดิม)
class BlogVisitorUser(HttpUser):
    weight = 9   # 90% ของ virtual user ทั้งหมดเป็นผู้เข้าชมทั่วไป
    wait_time = between(1, 5)
    # ... (task เดิมจากขั้อ 712.3)


class AuthenticatedAuthorUser(HttpUser):
    weight = 1   # 10% เป็นผู้เขียนที่ login แล้ว
    tasks = [AuthenticatedAuthorFlow]
    wait_time = between(2, 8)
```

`weight` กำหนดสัดส่วนว่า Locust จะสร้าง instance ของ User class ไหนมากน้อยแค่ไหน
เมื่อรันพร้อมกันหลาย class — การผสมสัดส่วนแบบนี้สำคัญมากเพื่อให้ traffic pattern
ที่ทดสอบ**ใกล้เคียงพฤติกรรมจริง** (ผู้เข้าชมทั่วไปย่อมเยอะกว่าผู้เขียนบทความเสมอ)

### 712.6 รัน Locust แบบ Web UI (สำหรับ Explore แบบ Interactive)

```bash
locust -f loadtests/locustfile.py --host=https://staging.myblog.example.com
```

เปิดเบราว์เซอร์ไปที่ `http://localhost:8089` แล้วกรอกจำนวน **Number of users**
และ **Spawn rate** (จำนวน user ใหม่ที่เพิ่มต่อวินาที) แล้วกด Start — จะเห็นกราฟ
RPS, Response Time, และ Number of Users แบบ real-time

### 712.7 รันแบบ Headless (สำหรับใส่ใน Script/CI)

```bash
locust -f loadtests/locustfile.py \
    --host=https://staging.myblog.example.com \
    --users 200 \
    --spawn-rate 10 \
    --run-time 5m \
    --headless \
    --csv=loadtests/results/baseline \
    --html=loadtests/results/baseline_report.html
```

| Flag | ความหมาย |
|---|---|
| `--users` | จำนวน Virtual User สูงสุดที่จะค่อย ๆ เพิ่มขึ้นไปถึง |
| `--spawn-rate` | อัตราเพิ่ม user ใหม่ (users/second) — ค่อย ๆ ramp up ไม่ใช่ยิงพรวดเดียว |
| `--run-time` | ระยะเวลาทดสอบทั้งหมด (`5m` = 5 นาที, `1h` = 1 ชั่วโมง) |
| `--headless` | ไม่เปิด Web UI รันจบแล้วออกจากโปรแกรมทันที เหมาะกับ CI |
| `--csv=PREFIX` | บันทึกผลสถิติเป็นไฟล์ CSV (`_stats.csv`, `_failures.csv`, `_stats_history.csv`) |
| `--html=FILE` | สร้างรายงานสรุปเป็นไฟล์ HTML พร้อมกราฟ |

### 712.8 ตารางสรุปขั้นตอนที่ 712

| ประเด็น | สรุป |
|---|---|
| Locust เขียนด้วยภาษาอะไร | Python ล้วน — ใช้ logic ซับซ้อนได้เต็มที่ |
| หน่วยพื้นฐานของ scenario | `HttpUser` (พฤติกรรม 1 ประเภทผู้ใช้) + `@task` (การกระทำแต่ละอย่าง) |
| ควบคุมลำดับ task แบบเคร่งครัด | ใช้ `SequentialTaskSet` แทน `@task` ปกติที่สุ่มลำดับ |
| จัดการ CSRF | ต้องดึงจาก cookie เองแล้วแนบกับทุก POST ด้วยมือ |
| ผสมสัดส่วน user หลายแบบ | ใช้ `weight` ในแต่ละ `HttpUser` class |
| รันแบบ CI-friendly | `--headless --csv=... --html=...` |

---

## ขั้นตอนที่ 713: `k6` เป็นทางเลือก (เขียนด้วย JavaScript) เปรียบเทียบกับ Locust

### 713.1 k6 คืออะไร

**k6** คือเครื่องมือ Load Testing จาก Grafana Labs เขียน engine ด้วยภาษา **Go**
(เร็วและใช้ resource น้อยกว่า) แต่ script ที่ผู้ใช้เขียนเป็น **JavaScript**
(ES2015+) จุดเด่นที่ทำให้ k6 แตกต่างจาก Locust อย่างชัดเจนคือ **Threshold** —
ความสามารถกำหนดเงื่อนไข pass/fail ไว้ใน script ตรง ๆ ทำให้ผสานเข้ากับ CI ได้ง่าย
มาก (เชื่อมกับขั้อ 719)

### 713.2 ติดตั้ง k6

```bash
# macOS
brew install k6

# Ubuntu/Debian
sudo gpg -k
sudo gpg --no-default-keyring --keyring /usr/share/keyrings/k6-archive-keyring.gpg \
    --keyserver hkp://keyserver.ubuntu.com:80 --recv-keys C5AD17C747E3415A3642D57D77C6C491D6AC1D69
echo "deb [signed-by=/usr/share/keyrings/k6-archive-keyring.gpg] https://dl.k6.io/deb stable main" \
    | sudo tee /etc/apt/sources.list.d/k6.list
sudo apt-get update
sudo apt-get install k6

# หรือใช้ Docker โดยไม่ต้องติดตั้งอะไรเลย
docker run --rm -i grafana/k6 run - <script.js
```

### 713.3 เขียน Scenario เดียวกันกับขั้อ 712 ด้วย k6

```javascript
// loadtests/blog_flow.js
import http from 'k6/http';
import { check, sleep } from 'k6';

// ค่า BASE_URL รับผ่าน environment variable เพื่อสลับ target ได้ (staging/production-like)
const BASE_URL = __ENV.BASE_URL || 'https://staging.myblog.example.com';

const SLUGS = ['django-tips', 'python-async-basics', 'postgresql-indexing-101'];

export const options = {
    // จำลองการ ramp-up แบบเดียวกับ Locust: ค่อย ๆ เพิ่ม virtual user (VU)
    stages: [
        { duration: '1m', target: 50 },    // ramp-up ไปถึง 50 VU ใน 1 นาที
        { duration: '3m', target: 200 },   // ramp-up ต่อไปถึง 200 VU ใน 3 นาที
        { duration: '5m', target: 200 },   // คงที่ 200 VU เป็นเวลา 5 นาที (steady state)
        { duration: '1m', target: 0 },     // ramp-down กลับสู่ 0
    ],
    // Threshold — เงื่อนไข pass/fail ที่ทำให้ k6 คืน exit code ไม่เป็น 0 เมื่อไม่ผ่าน
    thresholds: {
        http_req_duration: ['p(95)<400', 'p(99)<800'],  // p95 ต้องไม่เกิน 400ms, p99 ไม่เกิน 800ms
        http_req_failed: ['rate<0.01'],                   // error rate ต้องน้อยกว่า 1%
    },
};

function randomSlug() {
    return SLUGS[Math.floor(Math.random() * SLUGS.length)];
}

export default function () {
    // 1. เข้าหน้ารายการบทความ
    const listResponse = http.get(`${BASE_URL}/blog/`, { tags: { name: 'blog_list' } });
    check(listResponse, {
        'list status is 200': (r) => r.status === 200,
    });
    sleep(Math.random() * 3 + 1);   // จำลอง "เวลาอ่าน" ก่อนกดต่อไป (คล้าย wait_time ของ Locust)

    // 2. เข้าหน้ารายละเอียดบทความแบบสุ่ม
    const slug = randomSlug();
    const detailResponse = http.get(`${BASE_URL}/blog/${slug}/`, { tags: { name: 'blog_detail' } });
    check(detailResponse, {
        'detail status is 200': (r) => r.status === 200,
        'detail body contains title': (r) => r.body.includes('<h1'),
    });
    sleep(Math.random() * 4 + 1);

    // 3. บางครั้งกรองตามหมวดหมู่
    if (Math.random() < 0.2) {
        const filterResponse = http.get(`${BASE_URL}/blog/?category=django`, {
            tags: { name: 'blog_filter' },
        });
        check(filterResponse, { 'filter status is 200': (r) => r.status === 200 });
        sleep(1);
    }
}
```

### 713.4 รัน k6

```bash
k6 run loadtests/blog_flow.js

# กำหนด target แบบไม่ต้องแก้โค้ด (override ผ่าน environment variable)
k6 run -e BASE_URL=https://staging.myblog.example.com loadtests/blog_flow.js

# ส่งออกผลลัพธ์เป็น JSON เพื่อวิเคราะห์ต่อ หรือส่งเข้า time-series database (InfluxDB/Prometheus)
k6 run --out json=loadtests/results/k6_result.json loadtests/blog_flow.js
```

เมื่อรันจบ k6 จะพิมพ์สรุปผลออกทาง terminal ทันที รวมถึงบอกชัดเจนว่า **threshold
ผ่านหรือไม่ผ่าน** (exit code เป็น 0 เมื่อผ่านทุก threshold, ไม่เป็น 0 เมื่อมี
threshold ใดไม่ผ่าน) — คุณสมบัตินี้คือหัวใจสำคัญที่ทำให้ k6 เหมาะกับการฝังใน CI
Pipeline (ขั้อ 719) มากกว่า Locust ซึ่งไม่มี concept ของ threshold ในตัวเอง

### 713.5 ตารางเปรียบเทียบ Locust vs k6 แบบละเอียด

| ประเด็น | Locust | k6 |
|---|---|---|
| ภาษาที่เขียน Script | Python | JavaScript (ES2015+) |
| Engine เขียนด้วย | Python (ใช้ gevent สำหรับ concurrency) | Go (compiled, ใช้ resource ต่อ VU น้อยกว่ามาก) |
| Web UI แบบ real-time | ✅ มี (มาให้ในตัว) | ❌ ไม่มี (มีแต่ output เป็น terminal/JSON — ต้องต่อ Grafana เองถ้าอยากได้กราฟ) |
| Threshold (pass/fail อัตโนมัติ) | ❌ ไม่มีในตัว (ต้องเขียน logic เช็คเอง) | ✅ มีในตัว ผ่าน `options.thresholds` |
| Distributed Mode (หลายเครื่อง) | ✅ มี (`--master`/`--worker`) | ✅ มี (k6 Cloud หรือ Operator บน Kubernetes) |
| การจำลอง Concurrency | Greenlet (เบา ใช้ RAM น้อยต่อ VU) | Goroutine (เบายิ่งกว่า รองรับ VU จำนวนมากกว่าต่อเครื่องเดียวกัน) |
| เหมาะกับทีมที่ถนัด | Python/Django (เขียน scenario ซับซ้อนด้วย logic ที่คุ้นเคย) | Frontend/DevOps ที่ถนัด JavaScript, หรือทีมที่เน้นผสาน CI |
| Protocol ที่รองรับ | HTTP/HTTPS, WebSocket (ผ่าน plugin), gRPC (ผ่าน plugin) | HTTP/HTTPS, WebSocket, gRPC ในตัว, Browser testing (k6 browser) |
| License | Open Source (MIT) ฟรีทั้งหมด | Open Source (AGPL) ฟรี ส่วน Cloud/Distributed ขนาดใหญ่เสียเงิน |

### 713.6 คำแนะนำเชิงปฏิบัติ: เลือกใช้ตัวไหนเมื่อไหร่

- ใช้ **Locust** เมื่อทีมถนัด Python อยู่แล้ว (ทีม Django ส่วนใหญ่), ต้องการ
  scenario ที่มี logic ซับซ้อนมาก (เช่น ต้องเรียก API ภายนอกเพื่อสุ่มข้อมูลก่อน
  ยิง load), หรือต้องการดูกราฟ real-time ระหว่างทดสอบแบบ interactive
- ใช้ **k6** เมื่อต้องการผสานเข้ากับ CI Pipeline อย่างเข้มงวด (threshold ทำให้
  build fail อัตโนมัติ), ต้องการทดสอบด้วย VU จำนวนมากบนเครื่องเดียว (k6 ใช้
  resource ต่อ VU น้อยกว่ามาก), หรือทีมมีสมาชิกที่ถนัด JavaScript อยู่แล้ว

หลักสูตรนี้จะใช้ **Locust เป็นหลัก** ในแบบฝึกหัดเพราะสอดคล้องกับ ecosystem ของ
Django ที่เรียนมาตลอดหลักสูตร แต่แนะนำให้คุณลองทั้งสองตัวเพื่อรู้จักเครื่องมือทั้ง
สองฝั่ง เพราะในงานจริงบางทีมใช้ k6 เป็นมาตรฐานของทั้งองค์กรอยู่แล้ว

### 713.7 ตารางสรุปขั้นตอนที่ 713

| ประเด็น | สรุป |
|---|---|
| k6 เขียนด้วยภาษาอะไร | JavaScript (engine เป็น Go) |
| จุดเด่นที่ทำให้ k6 เข้ากับ CI ได้ดี | `options.thresholds` — exit code ไม่เป็น 0 เมื่อไม่ผ่านเกณฑ์ |
| ข้อจำกัดหลักเทียบกับ Locust | ไม่มี Web UI แบบ real-time ในตัว |
| คำแนะนำของหลักสูตร | ใช้ Locust เป็นหลัก แต่รู้จัก k6 ไว้สำหรับกรณีที่ต้องผสาน CI เข้มงวด |

---

## ขั้นตอนที่ 714: การหาจุดแตกหัก (Breaking Point) ของระบบ — Requests/sec, Response Time Percentiles

### 714.1 Breaking Point คืออะไร

**Breaking Point** คือระดับ concurrency (จำนวน virtual user พร้อมกัน) ที่ระบบเริ่ม
**ไม่สามารถรักษาคุณภาพบริการไว้ได้อีกต่อไป** สัญญาณที่บ่งบอกว่าถึงจุดนี้แล้วมีสอง
แบบหลัก:

1. **Error rate พุ่งขึ้นอย่างชัดเจน** (เช่น จากใกล้ 0% กระโดดไปเป็น 5%+ ทันที) —
   มักเกิดจาก connection pool เต็ม (ขั้อ 715), worker ไม่พอ (ขั้อ 716), หรือ
   request timeout
2. **Response time เพิ่มขึ้นแบบไม่เป็นเส้นตรง (non-linear)** — จากที่เพิ่ม
   concurrency ทีละนิดแล้ว latency ขึ้นทีละนิด กลายเป็นเพิ่มนิดเดียวแล้ว latency
   พุ่งกระฉูด (เพราะ request เริ่มต้องรอคิวยาวขึ้นเรื่อย ๆ จนควบคุมไม่ได้)

### 714.2 ทำไมค่าเฉลี่ย (Average) หลอกลวงได้ — ต้องดู Percentile

สมมติมี 100 requests ที่ response time ดังนี้: 95 requests ใช้เวลา **50ms**
(เร็วมาก) และ 5 requests ใช้เวลา **5,000ms** (ช้ามาก เพราะไปติด lock หรือ query
หนักบางตัว)

```
ค่าเฉลี่ย (Average) = ((95 × 50) + (5 × 5000)) / 100 = (4,750 + 25,000) / 100 = 297.5ms
```

ตัวเลข **297.5ms ดูเหมือนโอเค** แต่ความจริง**มีผู้ใช้ 5% ที่รอนานถึง 5 วินาที**
ซึ่งในระบบที่มี traffic วันละหลักแสน request หมายถึงคนนับพันคนที่เจอประสบการณ์
แย่มากทุกวัน — ค่าเฉลี่ยไม่เคยบอกเรื่องนี้ได้เลย นี่คือเหตุผลที่วงการ Performance
Engineering ใช้ **Percentile** แทนค่าเฉลี่ยเสมอ

| Percentile | ความหมาย | ใช้ตัดสินใจอะไร |
|---|---|---|
| **p50 (Median)** | 50% ของ request เร็วกว่าค่านี้ | ประสบการณ์ของผู้ใช้ "ทั่วไป" ส่วนใหญ่ |
| **p95** | 95% ของ request เร็วกว่าค่านี้ (มีแค่ 5% ที่ช้ากว่า) | มาตรฐานอุตสาหกรรมที่ใช้ตั้ง SLA บ่อยที่สุด |
| **p99** | 99% ของ request เร็วกว่าค่านี้ (มีแค่ 1% ที่ช้ากว่า) | จับ "หางที่แย่ที่สุด" (long tail) ที่ค่าเฉลี่ยซ่อนไว้ |
| **p99.9** | 99.9% ของ request เร็วกว่าค่านี้ | ใช้กับระบบขนาดใหญ่มากที่แม้ 0.1% ก็หมายถึงคนหลายพันคน |

**กฎที่ควรจำ**: อย่าใช้ Average เป็นเกณฑ์ตัดสินใจเรื่อง Performance เด็ดขาด ให้ใช้
p95 เป็นอย่างน้อยเสมอ (ทีมระดับโลกส่วนใหญ่ตั้ง SLA ด้วย p95 หรือ p99)

### 714.3 ออกแบบ Custom Load Shape เพื่อหา Breaking Point อัตโนมัติด้วย Locust

การหา breaking point ต้องเพิ่ม concurrency แบบ **step (ขั้นบันได)** แล้ววัดผล
แต่ละขั้นแยกกัน แทนที่จะ ramp-up ต่อเนื่องแบบ smooth เหมือนขั้อ 712 — Locust รองรับ
ผ่านคลาส `LoadTestShape`:

```python
# loadtests/breaking_point_shape.py
from locust import LoadTestShape


class StepLoadShape(LoadTestShape):
    """
    เพิ่ม concurrency ทีละขั้น (step) ขั้นละ 2 นาที เพื่อดูว่า metric แย่ลงตรงขั้นไหน
    รูปแบบ: 50 -> 100 -> 150 -> 200 -> 250 -> 300 users ทุก 2 นาที
    """

    step_time = 120          # แต่ละขั้นนานกี่วินาที
    step_users = 50          # เพิ่ม user ทีละกี่คนในแต่ละขั้น
    max_users = 300          # เพดานสูงสุดที่จะทดสอบ
    spawn_rate = 20          # อัตราเพิ่ม user ภายในแต่ละขั้น

    def tick(self):
        run_time = self.get_run_time()
        current_step = run_time // self.step_time

        target_users = self.step_users * (current_step + 1)
        if target_users > self.max_users:
            return None   # คืน None เพื่อบอก Locust ว่าทดสอบจบแล้ว

        return (target_users, self.spawn_rate)
```

รันโดยรวม shape นี้เข้ากับ `locustfile.py` เดิม:

```bash
locust -f loadtests/locustfile.py -f loadtests/breaking_point_shape.py \
    --host=https://staging.myblog.example.com \
    --headless \
    --csv=loadtests/results/breaking_point
```

### 714.4 อ่านผลลัพธ์เพื่อระบุ Breaking Point

หลังรันจบ เปิดไฟล์ `breaking_point_stats_history.csv` ที่ Locust สร้างให้ (บันทึก
สถิติทุก ๆ ไม่กี่วินาทีตลอดการทดสอบ) แล้วสร้างตารางสรุปตามแต่ละ step:

| Concurrent Users | RPS | p50 (ms) | p95 (ms) | p99 (ms) | Error Rate |
|---|---|---|---|---|---|
| 50 | 48.2 | 45 | 90 | 140 | 0.0% |
| 100 | 95.1 | 52 | 110 | 180 | 0.0% |
| 150 | 141.3 | 68 | 165 | 260 | 0.1% |
| 200 | 178.6 | 95 | 310 | 520 | 0.3% |
| **250** | **190.2** | **220** | **1,850** | **3,200** | **4.8%** |
| 300 | 165.4 | 890 | 4,600 | 8,100 | 22.7% |

จากตารางตัวอย่างนี้ **จุดแตกหักอยู่ที่ประมาณ 200-250 concurrent users**:

- ที่ 150 → 200 users: p95 เพิ่มจาก 165ms เป็น 310ms (เพิ่มขึ้น ~2 เท่า) — เริ่ม
  เห็นสัญญาณเตือน แต่ยังพอรับได้
- ที่ 200 → 250 users: p95 กระโดดจาก 310ms เป็น 1,850ms (เพิ่มขึ้น ~6 เท่า!) และ
  error rate เริ่มขยับจาก 0.3% เป็น 4.8% — **นี่คือ breaking point**
- ที่ 300 users: RPS จริงกลับ**ลดลง**จาก 190 เหลือ 165 ทั้งที่ยิง user เข้ามาเยอะ
  กว่าเดิม — สัญญาณคลาสสิกของระบบที่ **saturated เต็มที่แล้ว** (คิวยาวจนแทบไม่มี
  งานไหนเสร็จทันเวลาเลย)

### 714.5 หาสาเหตุของ Breaking Point ด้วยการดู Resource ควบคู่กัน

ตัวเลขจาก Locust/k6 บอกแค่ **อาการ** แต่บอกไม่ได้ว่า**สาเหตุ**อยู่ตรงไหน ต้อง
monitor resource ของ server คู่ขนานไปด้วยเสมอระหว่างรัน load test (ผ่านเครื่องมือ
เช่น `htop`, `django-silk` จาก Part 066, หรือ Grafana/Prometheus ถ้ามี):

| อาการที่สังเกตพร้อม Breaking Point | สาเหตุที่เป็นไปได้มากที่สุด | ไปแก้ที่ขั้นตอนไหน |
|---|---|---|
| CPU ของ app server ขึ้นไป 100% | Worker ไม่พอ หรือ code มี computation หนักที่ยังไม่ optimize | ขั้อ 716 (Gunicorn worker), Part 066 (profiling) |
| Database connection error ("too many connections") | Connection pool เล็กเกินไป | ขั้อ 715 |
| Memory ของ app server เพิ่มขึ้นเรื่อย ๆ ไม่ลด | Memory leak ใน worker process ที่รันนาน | ขั้อ 716 (`max_requests` restart worker) |
| CPU ของ database server พุ่งสูง แต่ app server ยังว่าง | Query หนักที่ยังไม่มี index หรือ N+1 query หลุดรอด | Part 067, Part 070 |
| Network I/O เต็มก่อน CPU/Memory จะเต็ม | Bandwidth ระหว่าง server หรือ payload response ใหญ่เกินจำเป็น | ตรวจสอบ serializer, compression (gzip) |

### 714.6 ตารางสรุปขั้นตอนที่ 714

| ประเด็น | สรุป |
|---|---|
| Breaking Point | จุดที่ error rate พุ่งขึ้นชัดเจน หรือ latency เพิ่มแบบไม่เป็นเส้นตรง |
| ทำไมห้ามใช้ Average | ค่าเฉลี่ยซ่อน "หางที่แย่ที่สุด" ที่ผู้ใช้บางส่วนเจอจริง ๆ ได้ |
| Percentile มาตรฐานที่ใช้ตั้ง SLA | p95 เป็นอย่างน้อย, ระบบใหญ่มักดู p99 ด้วย |
| วิธีหา Breaking Point อัตโนมัติ | เพิ่ม concurrency แบบ step ด้วย `LoadTestShape` แล้วเทียบ metric แต่ละ step |
| หาสาเหตุที่แท้จริง | ต้อง monitor resource (CPU/Memory/DB connections) คู่ขนานกับผลจาก load test เสมอ |

---

## ขั้นตอนที่ 715: ขนาด Database Connection Pool ที่เหมาะสมภายใต้ Load สูง

### 715.1 ทำไม Database Connection ถึงเป็นทรัพยากรที่มีราคาแพง

การเปิด connection ใหม่ไปยัง PostgreSQL แต่ละครั้งมีต้นทุนที่มองไม่เห็นแต่มีจริง:
TCP handshake, การ authenticate, และที่สำคัญที่สุดคือ **PostgreSQL สร้าง process
ใหม่ (ไม่ใช่ thread) สำหรับทุก connection** ซึ่งกิน memory ประมาณ **5-10MB ต่อ
connection** โดยประมาณ — ถ้ามี 1,000 concurrent request แต่ละอันเปิด connection
ใหม่ของตัวเอง จะกิน memory ฝั่ง database ไปถึง 5-10GB แค่สำหรับ "การมีอยู่ของ
connection" เฉย ๆ ยังไม่นับการประมวลผล query จริง

### 715.2 ทวน `CONN_MAX_AGE` ของ Django (Persistent Connections)

Django มีการตั้งค่านี้มาให้พื้นฐานอยู่แล้ว:

```python
# config/settings.py
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.postgresql',
        'NAME': 'blogdb',
        'USER': 'blog_app',
        'PASSWORD': env('DB_PASSWORD'),
        'HOST': 'db.internal',
        'PORT': '5432',
        'CONN_MAX_AGE': 60,        # เก็บ connection ไว้ใช้ซ้ำนาน 60 วินาที แทนที่จะเปิด/ปิดทุก request
        'CONN_HEALTH_CHECKS': True,  # เช็คว่า connection ที่จะใช้ซ้ำยัง "มีชีวิต" อยู่ก่อนใช้จริง
    }
}
```

`CONN_MAX_AGE=60` ทำให้แต่ละ Gunicorn worker **เก็บ connection ของตัวเองไว้ใช้ซ้ำ**
นานสูงสุด 60 วินาที แทนที่จะเปิด-ปิด connection ใหม่ทุก request (ค่า default คือ
`0` ซึ่งหมายถึงปิดทันทีหลังจบทุก request — เหมาะกับ dev เท่านั้น) แต่ข้อจำกัดสำคัญ
คือ **จำนวน connection สูงสุดที่เปิดพร้อมกันยังคงเท่ากับจำนวน Gunicorn worker ทั้ง
หมด** (ทวนจากขั้อ 716) ถ้ามี 20 worker ก็มีอย่างน้อย 20 connection เปิดค้างไว้เสมอ
— และถ้า scale เป็นหลาย server พร้อมกัน จำนวน connection รวมจะยิ่งทวีคูณ จนอาจเกิน
`max_connections` ของ PostgreSQL ได้ง่าย ๆ (ค่า default ของ PostgreSQL คือ 100)

### 715.3 แก้ปัญหาด้วย PgBouncer (Connection Pooler)

**PgBouncer** คือ lightweight connection pooler ที่ทำหน้าที่เป็น "ตัวกลาง" ระหว่าง
Django กับ PostgreSQL — Django worker ทุกตัว connect เข้า PgBouncer แทนที่จะ
connect ตรงเข้า PostgreSQL แล้ว PgBouncer จะ **ใช้ connection จริงจำนวนน้อย
ร่วมกัน (pool)** แจกจ่ายให้ worker หลายร้อยตัวสลับกันใช้

```
┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│ Gunicorn      │     │              │     │              │
│ Worker 1-20   │────▶│  PgBouncer    │────▶│  PostgreSQL   │
│ (200 conn)    │     │  (pool: 20)  │     │  (20 conn)    │
└──────────────┘     └──────────────┘     └──────────────┘
```

ติดตั้งและตั้งค่า PgBouncer:

```bash
sudo apt install pgbouncer
```

```ini
; /etc/pgbouncer/pgbouncer.ini
[databases]
blogdb = host=127.0.0.1 port=5432 dbname=blogdb

[pgbouncer]
listen_addr = 0.0.0.0
listen_port = 6432
auth_type = scram-sha-256
auth_file = /etc/pgbouncer/userlist.txt

; transaction pooling: คืน connection กลับ pool ทันทีที่ transaction จบ (ไม่ใช่รอจน
; client ปิด connection) — เหมาะกับ Django ที่สุด เพราะให้ throughput สูงสุด
pool_mode = transaction

; จำนวน client connection สูงสุดที่ PgBouncer เองรับได้ (จาก Django worker ทุกตัวรวมกัน)
max_client_conn = 1000

; จำนวน connection จริงที่เปิดไปยัง PostgreSQL ต่อ database (ตัวเลขสำคัญที่สุด)
default_pool_size = 20

; connection สำรองสำหรับ query ที่ค้างนาน ไม่ให้แย่ง pool หลักจนหมด
reserve_pool_size = 5
reserve_pool_timeout = 3
```

แล้วเปลี่ยน `HOST`/`PORT` ใน Django ให้ชี้ไปที่ PgBouncer แทน PostgreSQL ตรง ๆ:

```python
# config/settings.py
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.postgresql',
        'NAME': 'blogdb',
        'HOST': 'pgbouncer.internal',   # ชี้ไปที่ PgBouncer แทน PostgreSQL โดยตรง
        'PORT': '6432',                  # port ของ PgBouncer ไม่ใช่ 5432
        # เมื่อใช้ pool_mode = transaction ต้องปิด CONN_MAX_AGE ของ Django เอง
        # (ให้ PgBouncer เป็นคนจัดการ pooling แทนทั้งหมด ไม่ทับซ้อนกัน)
        'CONN_MAX_AGE': 0,
    }
}
```

**ข้อควรระวังสำคัญ**: เมื่อใช้ `pool_mode = transaction` ของ PgBouncer **ห้ามใช้
ฟีเจอร์ที่พึ่งพา session state ข้าม transaction** เช่น `SET search_path`,
`LISTEN/NOTIFY`, หรือ prepared statement บางรูปแบบ เพราะ connection จริงอาจถูก
สลับไปให้ client อื่นใช้ทันทีที่ transaction จบ — Django ORM ปกติทำงานร่วมกับ
`transaction` mode ได้ดีอยู่แล้วในกรณีส่วนใหญ่ แต่ควรทดสอบให้แน่ใจก่อนใช้จริง

### 715.4 สูตรคำนวณขนาด Pool เบื้องต้น

ไม่มีตัวเลขที่ถูกต้องตายตัวสำหรับทุกระบบ แต่มีสูตรเริ่มต้นที่ใช้กันแพร่หลายในวงการ
(ดัดแปลงจากแนวทางของ PostgreSQL wiki สำหรับประมาณการ concurrency ที่เหมาะสม):

```
pool_size ≈ ((จำนวน CPU core ของ database server × 2) + จำนวน effective spindle)
```

สำหรับ SSD/NVMe สมัยใหม่ (ไม่มี spindle แบบจานหมุน) ให้ประมาณ `effective_spindle`
เป็น 1 เสมอ ตัวอย่างเช่น database server ที่มี 8 core:

```
pool_size ≈ (8 × 2) + 1 = 17  →  ปัดเป็น 20 เพื่อความปลอดภัย
```

**ตัวเลขนี้เป็นแค่จุดเริ่มต้น ไม่ใช่คำตอบสุดท้าย** — ค่าที่ถูกต้องจริงต้องมาจากการ
**load test จริงแล้ววัดผล** ตามตารางด้านล่าง

### 715.5 ทดสอบผลกระทบของขนาด Pool ต่าง ๆ ด้วย Locust

รัน `StepLoadShape` จากขั้อ 714 ซ้ำหลายรอบ โดยเปลี่ยนแค่ค่า `default_pool_size`
ใน `pgbouncer.ini` แล้ว restart PgBouncer ก่อนรันแต่ละรอบ (`sudo systemctl restart
pgbouncer`) ที่ concurrency คงที่ 200 users:

| `default_pool_size` | RPS ที่ 200 users | p95 (ms) | Error Rate | หมายเหตุ |
|---|---|---|---|---|
| 5 | 82.1 | 1,450 | 3.2% | Pool เล็กเกินไป — request ต้องรอคิวแย่ง connection นาน |
| 10 | 145.6 | 480 | 0.4% | ดีขึ้นชัดเจน แต่ยังมี queueing บ้าง |
| **20** | **178.6** | **310** | **0.3%** | **จุดสมดุลที่ดี — เพิ่ม pool ต่อไปแทบไม่ช่วยอะไรเพิ่ม** |
| 40 | 181.2 | 295 | 0.3% | แทบไม่ต่างจาก 20 — เพิ่ม pool เกินความจำเป็นของ CPU database |
| 80 | 162.8 | 510 | 1.1% | **แย่ลง!** — connection เยอะเกินทำให้ PostgreSQL เสีย overhead จาก context switching ระหว่าง process |

ผลลัพธ์ตัวอย่างนี้แสดงหลักการสำคัญ: **Pool ที่ใหญ่กว่าไม่ได้แปลว่าดีกว่าเสมอไป**
เพราะ PostgreSQL ใช้โมเดล 1 process ต่อ 1 connection การมี connection มากเกินกว่า
ที่ CPU core จะประมวลผลพร้อมกันได้จริง ทำให้เกิด **context switching overhead**
ที่ทำให้ throughput รวมลดลงแทนที่จะเพิ่มขึ้น — นี่คือเหตุผลที่ต้องหาตัวเลขที่
เหมาะสมด้วยการทดลองจริง ไม่ใช่ตั้งเดาสูง ๆ ไว้ก่อนแล้วคิดว่าปลอดภัย

### 715.6 ตารางสรุปขั้นตอนที่ 715

| ประเด็น | สรุป |
|---|---|
| ทำไม DB connection แพง | PostgreSQL ใช้ 1 process ต่อ 1 connection กิน memory ~5-10MB/connection |
| `CONN_MAX_AGE` ของ Django | ลดการเปิด/ปิด connection บ่อย แต่ยังจำกัดที่ 1 connection ต่อ 1 worker |
| PgBouncer ทำหน้าที่อะไร | รวม connection จาก worker จำนวนมากให้ใช้ pool ขนาดเล็กร่วมกัน |
| `pool_mode` ที่แนะนำสำหรับ Django | `transaction` — throughput สูงสุด แต่ระวังฟีเจอร์ที่พึ่ง session state |
| สูตรเริ่มต้นคำนวณ pool size | `(CPU core × 2) + 1` แล้วปรับจากผล load test จริง |
| กับดักสำคัญที่สุด | Pool ใหญ่เกินไปทำให้ performance **แย่ลง** ไม่ใช่ดีขึ้น เพราะ context switching |

---

## ขั้นตอนที่ 716: การปรับจูน Gunicorn Worker — Sync vs Async Worker, สูตรคำนวณจำนวน Worker ที่เหมาะสม

### 716.1 ทวนสถาปัตยกรรมของ Gunicorn

Gunicorn (Green Unicorn) คือ WSGI HTTP Server ที่ทำหน้าที่รับ HTTP request แล้ว
ส่งต่อให้ Django ประมวลผล — Gunicorn มี process หลักชื่อ **Master** ที่คอยจัดการ
process ลูกที่เรียกว่า **Worker** ซึ่งเป็นตัวที่**รัน Django แอปจริง**และประมวลผล
request แต่ละตัว

```
                  ┌────────────────┐
   Requests ────▶ │  Gunicorn       │
                  │  Master Process │
                  └────────┬────────┘
                           │ กระจายงานให้ Worker
        ┌──────────────────┼──────────────────┐
        ▼                  ▼                  ▼
┌───────────────┐  ┌───────────────┐  ┌───────────────┐
│  Worker 1      │  │  Worker 2      │  │  Worker 3      │
│  (Django App)  │  │  (Django App)  │  │  (Django App)  │
└───────────────┘  └───────────────┘  └───────────────┘
```

### 716.2 Worker Class: Sync vs Async — ความแตกต่างที่ชี้ชะตา Performance

| Worker Class | โมเดล Concurrency | 1 Worker รับกี่ Request พร้อมกัน | เหมาะกับงานประเภทไหน |
|---|---|---|---|
| `sync` (default) | 1 process = 1 request ในเวลาเดียว (blocking) | 1 | CPU-bound work, งานทั่วไปที่ไม่มี I/O รอนาน |
| `gthread` | 1 process แบ่งเป็นหลาย thread | เท่ากับจำนวน `--threads` | Mixed workload ที่มีทั้ง CPU และ I/O เล็กน้อย |
| `gevent` | Greenlet (cooperative, ต้อง monkey-patch I/O) | หลักร้อยถึงหลักพัน | I/O-bound หนัก (เรียก external API, DB query ที่รอนาน) |
| `eventlet` | คล้าย gevent แต่ implementation ต่างกัน | หลักร้อยถึงหลักพัน | I/O-bound หนัก (ทางเลือกแทน gevent) |
| `uvicorn.workers.UvicornWorker` | ASGI async แท้ (ไม่ต้อง monkey-patch) | หลักพันขึ้นไปสำหรับ async view | Django Async Views (Part 073), WebSocket ผ่าน Channels |

**หลักการสำคัญที่ต้องเข้าใจ**: `sync` worker แต่ละตัว**ประมวลผลได้ทีละ 1 request
เท่านั้น** ถ้า request นั้นกำลังรอ database หรือ external API (I/O-bound) worker
ตัวนั้นจะ**ถูกบล็อกทั้งหมด**ไม่สามารถรับ request อื่นได้เลยจนกว่าจะเสร็จ — นี่คือ
เหตุผลที่ระบบที่ต้องรอ I/O เยอะ (เช่น เรียก payment gateway ภายนอก) มักได้ประโยชน์
มหาศาลจากการเปลี่ยนไปใช้ `gevent`/`eventlet` หรือ async view (จะเจาะลึกเต็มรูปแบบ
ใน Part 073 เมื่อเริ่ม Phase 9)

### 716.3 สูตรคำนวณจำนวน Worker มาตรฐาน

สูตรอย่างเป็นทางการจากเอกสารของ Gunicorn เอง สำหรับ `sync` worker (CPU-bound
workload ทั่วไป):

```
workers = (2 × จำนวน CPU core) + 1
```

```python
# ตัวอย่าง: server มี 4 CPU core
workers = (2 × 4) + 1 = 9
```

เหตุผลของสูตรนี้: ตัวเลข `2×core` มาจากการที่ระบบส่วนใหญ่มี I/O บางส่วนปะปนอยู่
เสมอ (แม้จะเป็น CPU-bound เป็นหลัก) ทำให้จำนวน worker ที่เหมาะสมมากกว่าจำนวน core
ที่มีจริงเล็กน้อยเพื่อใช้ core ได้เต็มประสิทธิภาพระหว่างที่ worker บางตัวรอ I/O
ส่วน `+1` คือ worker สำรองเผื่อกรณีฉุกเฉิน

**ข้อควรระวัง**: สูตรนี้เป็นแค่**จุดเริ่มต้น** ไม่ใช่กฎตายตัว — ระบบที่มี I/O
เยอะ (query หนัก, เรียก API ภายนอกบ่อย) อาจต้องการ worker มากกว่านี้ (หรือควรใช้
`gthread`/`gevent` แทนการเพิ่ม `sync` worker เรื่อย ๆ) ส่วนระบบที่ทำ computation
หนักมาก (CPU-bound แท้ ๆ) อาจต้องการ worker **น้อยกว่า** สูตรนี้เพราะ worker ที่
เยอะเกินจำนวน core จริงจะแย่ง CPU กันเองจนช้าลง

### 716.4 ไฟล์ตั้งค่า `gunicorn.conf.py` ฉบับสมบูรณ์

```python
# gunicorn.conf.py
import multiprocessing

bind = "0.0.0.0:8000"

# สูตรมาตรฐาน (2 × core) + 1 — ปรับตัวเลขสุดท้ายจากผล load test จริงเสมอ
workers = multiprocessing.cpu_count() * 2 + 1
worker_class = "sync"

# ใช้คู่กับ worker_class = "gthread" เท่านั้น (ไม่มีผลกับ sync/gevent)
threads = 2

# Timeout: ถ้า worker ไม่ตอบสนองเกินเวลานี้ (วินาที) Gunicorn จะ kill แล้วสร้างใหม่
timeout = 30
graceful_timeout = 30

# ป้องกัน Memory Leak: restart worker อัตโนมัติหลังประมวลผลครบจำนวน request ที่กำหนด
max_requests = 1000
max_requests_jitter = 50   # สุ่ม +/- ค่านี้ เพื่อไม่ให้ worker ทุกตัว restart พร้อมกันเป๊ะ

# จำนวน connection ที่รอในคิวก่อนถูกปฏิเสธ (สำหรับตอน traffic พุ่งกะทันหัน)
backlog = 2048

# Keep-alive: ระยะเวลาที่ worker คง connection ไว้รอ request ถัดไปจาก client เดียวกัน
keepalive = 5

# Logging
accesslog = "-"     # พิมพ์ access log ออก stdout (เหมาะกับ container ที่ log รวมศูนย์)
errorlog = "-"
loglevel = "info"
```

รันด้วย:

```bash
gunicorn config.wsgi:application -c gunicorn.conf.py
```

### 716.5 `max_requests` — ทำไมต้อง Restart Worker เป็นระยะ

Python ไม่ garbage collect ทุก object ได้สมบูรณ์แบบเสมอไป โดยเฉพาะเมื่อมี
3rd-party library บางตัวที่มี memory leak เล็กน้อยสะสมไปเรื่อย ๆ ทีละ request —
`max_requests = 1000` บอก Gunicorn ให้ **kill worker แล้วสร้างตัวใหม่ทดแทน**
โดยอัตโนมัติหลังประมวลผลครบ 1,000 request ป้องกันไม่ให้ memory ของ worker บวมขึ้น
เรื่อย ๆ จนกิน RAM ทั้งเครื่องเมื่อรันต่อเนื่องเป็นวัน ๆ (เชื่อมโยงกับ **Soak
Testing** จากขั้อ 711.2 ที่ตรวจจับปัญหานี้ได้ก่อน production)

`max_requests_jitter = 50` เพิ่มความสุ่มเข้าไป (แต่ละ worker restart ที่ตัวเลข
สุ่มระหว่าง 950-1050 request) เพื่อป้องกันไม่ให้ worker ทุกตัวใน server เดียวกัน
restart **พร้อมกันเป๊ะ** ซึ่งจะทำให้ capacity ของ server หายไปชั่วขณะทั้งหมดพร้อม
กัน

### 716.6 ทดลองเปรียบเทียบ Sync vs Gevent Worker ด้วย Load Test จริง

สมมติ endpoint หนึ่งต้องเรียก external API (เช่น ตรวจสอบสถานะการชำระเงิน) ที่ใช้
เวลาตอบกลับ 200ms โดยเฉลี่ย — ทดสอบด้วย Locust ที่ concurrency คงที่ 100 users:

```bash
pip install gevent
```

```python
# gunicorn.conf.py (ปรับสำหรับทดสอบ gevent)
workers = 5
worker_class = "gevent"
worker_connections = 1000   # จำนวน greenlet สูงสุดต่อ worker (เฉพาะ gevent/eventlet)
```

| Worker Class | จำนวน Worker | RPS ที่ 100 users | p95 (ms) | หมายเหตุ |
|---|---|---|---|---|
| `sync` | 9 (ตามสูตร) | 42.1 | 2,380 | Worker ถูกบล็อกรอ external API ทีละตัว คิวยาวมาก |
| `gthread` (threads=4) | 9 | 118.6 | 850 | ดีขึ้นมากเพราะแต่ละ worker รับได้ 4 request พร้อมกัน |
| `gevent` | 5 | 312.4 | 220 | **ดีที่สุด** — greenlet นับพันตัวรอ I/O พร้อมกันได้โดยแทบไม่เสีย overhead |

ผลลัพธ์นี้แสดงให้เห็นชัดเจนว่า **สำหรับงานที่ I/O-bound หนัก (รอ network มาก)**
การเปลี่ยน `worker_class` มีผลกระทบต่อ throughput **มากกว่า**การเพิ่มจำนวน
`sync` worker เข้าไปเรื่อย ๆ อย่างเทียบกันไม่ได้เลย — นี่คือเหตุผลที่ Part 073
(Async Views และ ASGI) และการใช้ `UvicornWorker` จะกลายเป็นเครื่องมือสำคัญมากขึ้น
เมื่อระบบมี I/O-bound workload เยอะขึ้นเรื่อย ๆ ตามที่หลักสูตรจะพาไปต่อใน Phase 9

### 716.7 ตารางสรุปขั้นตอนที่ 716

| ประเด็น | สรุป |
|---|---|
| สูตรคำนวณ Worker มาตรฐาน | `(2 × CPU core) + 1` — จุดเริ่มต้น ไม่ใช่คำตอบสุดท้าย |
| `sync` worker เหมาะกับ | CPU-bound work, ไม่มี I/O รอนาน |
| `gevent`/`eventlet` เหมาะกับ | I/O-bound หนัก (เรียก external API, query ที่รอนาน) |
| `max_requests` มีไว้ทำไม | ป้องกัน memory leak สะสมจนกิน RAM หมดเมื่อรันนาน |
| วิธีหาค่าที่เหมาะสมจริง | Load test เปรียบเทียบ worker class/จำนวนต่าง ๆ แล้ววัดผล ไม่ใช่ใช้สูตรเดียวตายตัว |

---

## ขั้นตอนที่ 717: แนวคิด Horizontal Scaling vs Vertical Scaling

### 717.1 นิยามพื้นฐาน

| แนวทาง | ความหมาย | เปรียบเทียบง่าย ๆ |
|---|---|---|
| **Vertical Scaling (Scale Up)** | เพิ่มทรัพยากรให้ server เครื่องเดิม (CPU/RAM/SSD ที่แรงขึ้น) | อัปเกรดรถคันเดิมให้เครื่องยนต์แรงขึ้น |
| **Horizontal Scaling (Scale Out)** | เพิ่มจำนวน server แล้วกระจาย traffic ไปหลายเครื่อง | ซื้อรถเพิ่มอีกคันมาวิ่งคู่กัน |

### 717.2 ตารางเปรียบเทียบข้อดี-ข้อเสีย

| ประเด็น | Vertical Scaling | Horizontal Scaling |
|---|---|---|
| ความยากในการทำ | ง่ายมาก — แค่เปลี่ยนสเปกเครื่อง (หรือ resize instance บน Cloud) ไม่ต้องแก้โค้ด | ต้องออกแบบแอปให้ **stateless** ตั้งแต่แรก + ต้องมี Load Balancer |
| เพดานสูงสุด | มีจำกัด (เครื่องที่แรงที่สุดในตลาดก็ยังมีเพดาน) | แทบไม่มีเพดาน (เพิ่ม server ได้เรื่อย ๆ ตามต้องการ) |
| Downtime ตอน scale | มักต้อง restart/reboot เครื่อง (มี downtime) | เพิ่ม server ใหม่เข้า pool ได้โดยไม่กระทบ server เดิมเลย (zero-downtime) |
| Single Point of Failure | ✅ มี — ถ้าเครื่องเดียวนั้นล่ม ระบบทั้งหมดล่มตาม | ❌ ไม่มี (ถ้า 1 เครื่องล่ม เครื่องอื่นยังรับ traffic ต่อได้) |
| ต้นทุนเมื่อ scale ต่อเนื่อง | เพิ่มขึ้นแบบ**ไม่เป็นเส้นตรง** (เครื่องแรงมาก ๆ แพงกว่าสัดส่วน) | เพิ่มขึ้นแบบเป็นเส้นตรงมากกว่า (เครื่องขนาดกลางจำนวนมาก) |
| ความซับซ้อนของระบบ | ต่ำ | สูงกว่า (ต้องจัดการ session, cache แชร์ข้าม server, distributed lock) |

### 717.3 เงื่อนไขสำคัญที่ต้องมีก่อนทำ Horizontal Scaling: แอปต้อง Stateless

ปัญหาที่พบบ่อยที่สุดเมื่อทีมพยายาม horizontal scale โดยไม่เตรียมตัวมาก่อนคือ
**Session ผูกกับเครื่อง**: ถ้า user login เข้า server A แล้ว Load Balancer ส่ง
request ถัดไปของ user คนเดียวกันไปที่ server B ที่ไม่รู้จัก session นั้นเลย ผู้ใช้
จะถูก logout โดยไม่ทราบสาเหตุ

**ทางแก้ที่ถูกต้อง** (ทวนจาก Part 069 ที่ตั้งค่า Redis ไว้แล้ว): เก็บ **session
และ cache ไว้ที่ Redis ส่วนกลาง** ที่ทุก server เข้าถึงร่วมกันได้ แทนที่จะเก็บไว้
ใน memory ของแต่ละเครื่อง (`LocMemCache` ที่ Part 068 ข้อ 671.4 เตือนไว้แล้วว่า
**ไม่ sync กันข้าม process/server**):

```python
# config/settings.py
SESSION_ENGINE = "django.contrib.sessions.backends.cache"
SESSION_CACHE_ALIAS = "default"

CACHES = {
    "default": {
        "BACKEND": "django.core.cache.backends.redis.RedisCache",
        "LOCATION": "redis://redis.internal:6379/1",   # Redis กลางที่ทุก server เห็นร่วมกัน
    }
}
```

รวมถึงไฟล์ที่ผู้ใช้อัปโหลด (Media files จาก Part 058) ก็ต้องย้ายไปเก็บที่
**Object Storage กลาง** (เช่น AWS S3, MinIO) แทนการเก็บไว้ใน disk ของเครื่องใด
เครื่องหนึ่ง ไม่เช่นนั้นไฟล์ที่อัปโหลดผ่าน server A จะหาไม่เจอเมื่อมีคนขอดูผ่าน
server B

### 717.4 ตัวอย่างการตั้งค่า Load Balancer ด้วย Nginx

```nginx
# /etc/nginx/sites-available/blog
upstream django_backend {
    least_conn;   # ส่ง request ไปยัง server ที่มี connection ค้างอยู่น้อยที่สุด

    server 10.0.1.11:8000 max_fails=3 fail_timeout=30s;
    server 10.0.1.12:8000 max_fails=3 fail_timeout=30s;
    server 10.0.1.13:8000 max_fails=3 fail_timeout=30s;
}

server {
    listen 80;
    server_name myblog.example.com;

    location / {
        proxy_pass http://django_backend;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

`least_conn` เป็นหนึ่งใน load balancing algorithm ของ Nginx ที่เหมาะกับ Django
มากกว่า `round-robin` (ค่า default) เมื่อ request บางตัวใช้เวลานานกว่าตัวอื่น
มาก เพราะจะไม่ส่ง request ใหม่ไปทับเครื่องที่กำลังยุ่งอยู่แล้ว `max_fails`/
`fail_timeout` ทำให้ Nginx หยุดส่ง traffic ไปยัง server ที่เพิ่ง fail ไปชั่วคราว
โดยอัตโนมัติ (basic health check)

### 717.5 การ Scale ฐานข้อมูล: Vertical ก่อน แล้วค่อย Horizontal

ฐานข้อมูลมีความซับซ้อนกว่า app server เพราะ**เก็บ state จริง** (ข้อมูล) ไม่ใช่
stateless เหมือน Django app — แนวทางมาตรฐานของอุตสาหกรรมคือ:

1. **Vertical scale ฐานข้อมูลก่อนเสมอ** (เพิ่ม CPU/RAM/ใช้ SSD เร็วขึ้น) เพราะทำ
   ได้ง่ายและไม่ต้องแก้สถาปัตยกรรมแอปเลย
2. เมื่อ vertical scale ถึงเพดานแล้ว (หรือแพงเกินคุ้ม) ค่อยไปทาง **Read
   Replica** — สร้างฐานข้อมูลสำเนาที่รับเฉพาะ query อ่าน (`SELECT`) แยกออกจาก
   ฐานข้อมูลหลักที่รับเฉพาะ query เขียน (`INSERT`/`UPDATE`/`DELETE`) เหมาะกับ
   ระบบที่มีอัตราส่วนอ่าน:เขียน สูงมาก (บล็อกทั่วไปมักอ่านมากกว่าเขียนหลายสิบเท่า)
3. ขั้นสุดท้ายที่ซับซ้อนที่สุดคือ **Sharding** (แบ่งข้อมูลออกเป็นหลายฐานข้อมูล
   ตาม key บางอย่าง เช่น user_id) ซึ่งเพิ่มความซับซ้อนมหาศาลและควรทำเมื่อจำเป็น
   จริง ๆ เท่านั้น (หัวข้อนี้อยู่นอกขอบเขตของ Part นี้ จะกล่าวถึงเพิ่มเติมเมื่อถึง
   Phase ที่ว่าด้วยสถาปัตยกรรมระดับองค์กร)

### 717.6 ตารางสรุปการตัดสินใจ: เมื่อไหร่ควรใช้แบบไหน

| สถานการณ์ | แนวทางที่แนะนำ |
|---|---|
| Traffic ยังน้อย ทีมเล็ก ต้องการความเรียบง่าย | Vertical scaling — เพิ่มสเปก server เดียวไปก่อน |
| ต้องการ High Availability (ห้ามมี downtime) | Horizontal scaling — จำเป็นเพราะ vertical ไม่มี redundancy |
| Traffic เพิ่มแบบคาดเดาได้ยาก/ผันผวนสูง (เช่น flash sale) | Horizontal scaling + Auto Scaling (เพิ่ม/ลด server อัตโนมัติตาม load) |
| ฐานข้อมูลเป็นคอขวด และอ่านมากกว่าเขียนเยอะ | Read Replica (horizontal เฉพาะฝั่งอ่าน) |
| App server เป็นคอขวดจาก CPU/Memory ไม่พอ | ลอง Vertical ก่อน (ง่ายกว่า) ถ้ายังไม่พอค่อย Horizontal |

### 717.7 ตารางสรุปขั้นตอนที่ 717

| ประเด็น | สรุป |
|---|---|
| Vertical Scaling | เพิ่มสเปกเครื่องเดิม — ง่ายแต่มีเพดานและมี single point of failure |
| Horizontal Scaling | เพิ่มจำนวนเครื่อง — ไม่มีเพดาน แต่ต้องออกแบบแอปให้ stateless ก่อน |
| เงื่อนไขบังคับก่อน Horizontal Scale | Session และ Cache ต้องอยู่ที่ Redis กลาง, Media files ต้องอยู่ Object Storage กลาง |
| ลำดับการ Scale ฐานข้อมูล | Vertical ก่อน → Read Replica → Sharding (ซับซ้อนสุด ใช้เมื่อจำเป็นจริง) |

---

## ขั้นตอนที่ 718: เปรียบเทียบผล Load Test ก่อน/หลังเปิดใช้ Caching

### 718.1 ทวนสิ่งที่ Part 068-069 สร้างไว้

Part 068 สอน `@cache_page`, `{% cache %}` fragment caching, และ Low-level Cache
API ส่วน Part 069 เจาะลึก Redis เป็น production backend พร้อม cache stampede
protection — ทั้งหมดนี้เป็น**การคาดเดาอย่างมีหลักการ**ว่าจะช่วยเพิ่ม performance
แต่ตลอดสอง Part นั้น**ยังไม่เคยวัดผลจริงด้วยตัวเลข**เลยว่า caching ช่วยได้มาก
แค่ไหนภายใต้ load จริง — นี่คือสิ่งที่ขั้อนี้จะทำ

### 718.2 ออกแบบการทดลองอย่างเป็นวิทยาศาสตร์

หลักการสำคัญของการวัดผล "ก่อน/หลัง" คือต้อง **เปลี่ยนแค่ตัวแปรเดียว** (caching)
โดยควบคุมทุกอย่างอื่นให้เหมือนกันทุกประการ:

| ตัวแปร | ต้องเหมือนกันทุกรอบ |
|---|---|
| Locust script (`locustfile.py`) | ใช้ไฟล์เดียวกันเป๊ะทั้งสองรอบ |
| Load Shape (จำนวน user, spawn rate, ระยะเวลา) | ใช้ `StepLoadShape` เดียวกันจากขั้อ 714 |
| ข้อมูลในฐานข้อมูล | Seed ข้อมูลชุดเดียวกันก่อนแต่ละรอบ (จำนวน post, comment เท่ากัน) |
| สเปกเครื่อง Staging | เครื่องเดียวกัน ไม่เปลี่ยนระหว่างสองรอบ |
| จำนวน Gunicorn Worker | ค่าที่ tune ไว้แล้วจากขั้อ 716 ใช้เหมือนกันทั้งสองรอบ |
| **ตัวแปรเดียวที่เปลี่ยน** | เปิด/ปิด caching (`@cache_page` + `{% cache %}` + Redis) เท่านั้น |

```bash
# รอบที่ 1: ปิด caching (checkout branch ก่อนทำ Part 068)
git checkout before-caching
locust -f loadtests/locustfile.py -f loadtests/breaking_point_shape.py \
    --host=https://staging.myblog.example.com --headless \
    --csv=loadtests/results/no_cache

# รอบที่ 2: เปิด caching (checkout branch หลังทำ Part 068-069)
git checkout after-caching
locust -f loadtests/locustfile.py -f loadtests/breaking_point_shape.py \
    --host=https://staging.myblog.example.com --headless \
    --csv=loadtests/results/with_cache
```

### 718.3 ตัวอย่างผลลัพธ์เปรียบเทียบ (ที่ 200 concurrent users คงที่)

| Metric | ก่อนเปิด Caching | หลังเปิด Caching | ผลต่าง |
|---|---|---|---|
| RPS | 178.6 | 612.3 | **เพิ่มขึ้น 3.4 เท่า** |
| p50 Response Time | 95ms | 18ms | เร็วขึ้น 5.3 เท่า |
| p95 Response Time | 310ms | 42ms | เร็วขึ้น 7.4 เท่า |
| p99 Response Time | 520ms | 85ms | เร็วขึ้น 6.1 เท่า |
| Error Rate | 0.3% | 0.0% | ดีขึ้น |
| Query Count ต่อ Request (หน้ารายการ) | 14 queries (วัดด้วย `django-silk` จาก Part 066) | 0-1 query (เฉพาะตอน cache miss) | ลด load ฐานข้อมูลมหาศาล |
| CPU ของ Database Server ที่ 200 users | 78% | 12% | ลดลงอย่างมีนัยสำคัญ |

ตัวเลขเหล่านี้ (เป็นตัวอย่างสมมติที่สมจริง) แสดงให้เห็นสิ่งที่ Part 068 ข้อ 671.1
พูดไว้ด้วยหลักการ: **"การอ่านจาก memory เร็วกว่าการไปแตะฐานข้อมูลหลักสิบถึงหลัก
พันเท่า"** — ตอนนี้คุณเห็นตัวเลขจริงที่พิสูจน์คำกล่าวนั้นแล้ว

### 718.4 อธิบายว่าทำไม Caching ถึงเพิ่ม Breaking Point ได้มาก

Bottleneck หลักของระบบก่อนเปิด caching คือ **ฐานข้อมูล** (CPU 78% ที่ 200 users)
— เมื่อเปิด caching ส่วนใหญ่ของ request (cache hit) **ไม่แตะฐานข้อมูลเลย** ทำให้
CPU ฝั่งฐานข้อมูลเหลือเยอะขึ้นมาก, connection pool ที่เคยตึงจากขั้อ 715 ก็ว่างลง
ให้ request ส่วนน้อยที่จำเป็นต้อง query จริง (cache miss) ได้ resource เต็มที่ —
ผลลัพธ์คือ**ทั้งระบบรองรับ concurrency ได้สูงขึ้นมาก**ในสเปกเครื่องเท่าเดิม

### 718.5 ข้อควรระวัง: Load Test ต้องออกแบบให้ "สมจริง" ไม่ใช่ "เข้าข้าง Cache" เกินไป

กับดักสำคัญที่ทำให้ตัวเลข "หลังเปิด caching" ดูดีเกินจริงคือ **การทดสอบด้วย
scenario ที่ทุก virtual user เข้าหน้าเดียวกันซ้ำ ๆ** (เช่น ทุกคนเข้า `/blog/`
เฉย ๆ ไม่มีอะไรอื่น) เพราะแบบนี้ **cache hit rate จะเกือบ 100%** ตั้งแต่ request
ที่สองเป็นต้นไป ซึ่งไม่สะท้อนพฤติกรรมจริงที่ผู้ใช้เข้าหลายหน้าที่ต่างกัน (แต่ละ
slug มี cache key ของตัวเอง ทวนจาก Part 068 ข้อ 672.3)

**วิธีทดสอบที่สมจริงกว่า**: ใช้ `locustfile.py` ที่มีการสุ่มเข้าหลาย slug/หลาย
หน้าตามขั้อ 712.3 อยู่แล้ว (ไม่ใช่ hardcode URL เดียว) และควรมีสัดส่วน
**cache miss ปนอยู่เสมอ** (เช่น เข้าหน้าที่ query string ต่างกันไปเรื่อย ๆ ตาม
`?page=`) เพื่อให้ตัวเลขที่วัดได้ใกล้เคียงสิ่งที่จะเกิดขึ้นจริงเมื่อ user จริง
หลายพันคนเข้าเว็บพร้อมกันในหัวข้อที่แตกต่างกัน

### 718.6 ตารางสรุปขั้นตอนที่ 718

| ประเด็น | สรุป |
|---|---|
| หลักการทดลองที่ถูกต้อง | เปลี่ยนแค่ตัวแปรเดียว (caching) ควบคุมทุกอย่างอื่นให้เหมือนกัน |
| ผลลัพธ์ที่คาดหวังจาก Caching | เพิ่ม RPS, ลด latency percentile ทุกระดับ, ลดภาระฐานข้อมูลอย่างมาก |
| ทำไม Breaking Point สูงขึ้น | Bottleneck (ฐานข้อมูล) ถูกตัดออกจาก request ส่วนใหญ่ |
| กับดักที่ต้องระวัง | Scenario ที่ทุก user เข้าหน้าเดียวกันซ้ำจะให้ตัวเลข cache hit ที่ optimistic เกินจริง |

---

## ขั้นตอนที่ 719: เกริ่นการทำ Continuous Performance Testing ใน CI

### 719.1 ทำไม Load Test ควรรันอัตโนมัติ ไม่ใช่รันด้วยมือเป็นครั้งคราว

ทีมส่วนใหญ่ทำ load test แค่**ครั้งเดียว**ก่อน launch แล้วไม่เคยรันซ้ำอีกเลย — แต่
โค้ดที่เพิ่มเข้ามาทีหลัง (feature ใหม่, dependency ใหม่) อาจทำให้ performance
**เสื่อมถอยลงทีละนิด (Performance Regression)** โดยไม่มีใครรู้ตัว จนกว่าจะสายเกินไป
แนวคิด **Continuous Performance Testing** คือการรัน load test แบบเบา ๆ
**อัตโนมัติทุกครั้ง** ที่มีการเปลี่ยนแปลงโค้ดสำคัญ เพื่อจับ regression ตั้งแต่เนิ่น ๆ
เหมือนที่ Part 065 (Continuous Testing และ Code Quality Tools) สอนเรื่องรัน
unit test อัตโนมัติทุก commit

### 719.2 ตัวอย่าง GitHub Actions Workflow ที่รัน k6 อัตโนมัติ

k6 เหมาะกับงานนี้มากที่สุด (ทวนจากขั้อ 713.5) เพราะมี **Threshold** ในตัวที่ทำให้
CI รู้ได้ทันทีว่า performance "ผ่าน" หรือ "ไม่ผ่าน" โดยไม่ต้องเขียน logic เพิ่ม:

```yaml
# .github/workflows/performance-test.yml
name: Continuous Performance Testing

on:
  pull_request:
    branches: [main]
  schedule:
    - cron: "0 3 * * *"   # รันซ้ำทุกวันตอนตี 3 เพื่อจับ regression ที่ไม่ได้มาจาก PR

jobs:
  load-test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Deploy to ephemeral staging environment
        run: |
          # deploy โค้ดปัจจุบันไปยัง staging ชั่วคราวสำหรับทดสอบเท่านั้น
          ./scripts/deploy_ephemeral_staging.sh

      - name: Run k6 load test
        uses: grafana/k6-action@v0.3.1
        with:
          filename: loadtests/blog_flow.js
        env:
          BASE_URL: https://ephemeral-staging.myblog.example.com

      - name: Tear down ephemeral staging environment
        if: always()
        run: ./scripts/teardown_ephemeral_staging.sh
```

เพราะ `options.thresholds` ในไฟล์ `loadtests/blog_flow.js` (ขั้อ 713.3) กำหนด
`p(95)<400` และ `rate<0.01` ไว้แล้ว — ถ้า pull request ไหนทำให้ p95 แย่ลงเกิน 400ms
หรือ error rate เกิน 1% ขั้นตอน `Run k6 load test` จะ **fail ทันที** ทำให้ GitHub
บล็อกการ merge PR นั้นโดยอัตโนมัติ เหมือนกับที่ unit test fail บล็อกการ merge

### 719.3 ทำไมเรื่องนี้ยังไม่เจาะลึกเต็มรูปแบบใน Part นี้

การผสาน Load Testing เข้ากับ CI Pipeline อย่างสมบูรณ์ต้องอาศัยความรู้เรื่อง
**GitHub Actions ขั้นสูง** (matrix build, secret management, self-hosted
runner สำหรับงานหนัก, การ deploy ephemeral environment อัตโนมัติ) ซึ่งเป็นเนื้อหา
หลักของ **Part 088: CI/CD ด้วย GitHub Actions** ในภายหลัง — Part นั้นจะพากลับมา
เจาะลึกการตั้งค่า workflow นี้แบบเต็มรูปแบบ รวมถึงการเก็บผลลัพธ์ประวัติ (trend
over time) เพื่อดู graph ว่า performance ของระบบดีขึ้นหรือแย่ลงตามเวลา และการ
แจ้งเตือนทีมอัตโนมัติเมื่อ threshold ไม่ผ่าน ตอนนี้ขอให้คุณเข้าใจแค่ **แนวคิด**
ว่า Load Testing ไม่ควรเป็นกิจกรรมที่ทำครั้งเดียวจบ แต่ควรเป็นส่วนหนึ่งของ
pipeline ที่รันซ้ำอย่างสม่ำเสมอ

---

## ขั้นตอนที่ 720: สรุป Phase 8 ทั้งหมด (Part 066-072)

### 720.1 ภาพรวม Phase 8: จากการวัดผล สู่การปรับแต่ง สู่การพิสูจน์ด้วยตัวเลข

Phase 8 พาคุณเดินทางผ่าน 7 Part ที่ประกอบกันเป็น**วงจรสมบูรณ์ของ Performance
Engineering**:

```
Part 066 (Profiling) ──▶ หาว่า "ช้าตรงไหน" ด้วยเครื่องมือวัดผลจริง
        │
        ▼
Part 067 (Query Optimization) ──▶ แก้ N+1 query ด้วย select_related/prefetch_related
        │
        ▼
Part 068-069 (Caching + Redis) ──▶ เก็บผลลัพธ์ที่คำนวณแล้วไว้ใช้ซ้ำแทนคำนวณใหม่
        │
        ▼
Part 070 (Database Indexing) ──▶ ทำให้ query ที่ยังต้องแตะฐานข้อมูลจริงเร็วที่สุด
        │
        ▼
Part 071 (Pagination) ──▶ ไม่โหลดข้อมูลทั้งหมดทีเดียวเมื่อ dataset ใหญ่มาก
        │
        ▼
Part 072 (Load Testing) ──▶ พิสูจน์ด้วยตัวเลขจริงว่าทุกอย่างที่ทำมาช่วยได้จริงแค่ไหน
```

### 720.2 ตารางทบทวนสาระสำคัญของแต่ละ Part

| Part | หัวข้อหลัก | เครื่องมือ/เทคนิคสำคัญที่สุด | คำถามที่ตอบได้ |
|---|---|---|---|
| **066** | Performance Profiling | `django-silk`, `django-debug-toolbar`, `cProfile` | "โค้ดส่วนไหนช้าที่สุด และช้าเพราะอะไร" |
| **067** | Query Optimization | `select_related`, `prefetch_related`, `annotate`, `only`/`defer` | "ทำไม query เดียวกันถึงยิง SQL หลายสิบครั้ง (N+1)" |
| **068** | Caching Framework เบื้องต้น | `@cache_page`, `{% cache %}`, Low-level Cache API, Signals invalidation | "จะเก็บผลลัพธ์ที่คำนวณแล้วไว้ใช้ซ้ำได้อย่างไรโดยไม่แสดงข้อมูลเก่าผิดเวลา" |
| **069** | Redis Caching ขั้นสูง | Redis persistence, cache stampede protection ด้วย distributed lock | "จะทำ caching ให้ปลอดภัยและเสถียรในระดับ production ได้อย่างไร" |
| **070** | Database Indexing และ Query Analysis | `db_index=True`, composite index, `EXPLAIN ANALYZE` | "ทำไม query ที่ optimize แล้วยังช้าอยู่ดี และจะอ่าน query plan อย่างไร" |
| **071** | Pagination และ Large Dataset Handling | `PageNumberPagination`, Cursor/Keyset Pagination | "จะแสดงข้อมูลนับล้าน record โดยไม่โหลดทั้งหมดพร้อมกันได้อย่างไร" |
| **072** | Load Testing และ Scalability Testing | Locust, k6, PgBouncer, Gunicorn tuning, Horizontal/Vertical Scaling | "ระบบรองรับผู้ใช้พร้อมกันได้กี่คนกันแน่ และจะพิสูจน์ด้วยตัวเลขได้อย่างไร" |

### 720.3 หลักการที่ยึดโยงทั้ง Phase 8 เข้าด้วยกัน

ตลอด 7 Part ของ Phase นี้ มีหลักการร่วมกันอยู่เบื้องหลังเสมอ ไม่ว่าจะมองจากมุมไหน:

1. **วัดก่อนเสมอ อย่าเดา (Measure, Don't Guess)** — Part 066 สอนหลักการนี้ตั้งแต่
   ต้น และ Part 072 (Part นี้) ปิดท้าย Phase ด้วยหลักการเดียวกัน: อย่าเชื่อว่า
   การ optimize ใด ๆ ได้ผลจนกว่าจะมีตัวเลขจากการวัดจริงยืนยัน
2. **Bottleneck ย้ายที่เสมอเมื่อแก้จุดหนึ่งสำเร็จ** — แก้ N+1 query (067) แล้ว
   bottleneck ย้ายไปที่ query ที่ไม่มี index (070); เพิ่ม caching (068-069) แล้ว
   bottleneck ย้ายไปที่ connection pool ไม่พอ (715) หรือ worker ไม่พอ (716) —
   Performance Engineering เป็นกระบวนการที่ต่อเนื่อง ไม่มีจุดจบ "เสร็จสมบูรณ์"
   ตายตัว
3. **Trade-off มีอยู่เสมอ** — Caching แลกกับความเสี่ยงข้อมูลเก่า (Cache
   Invalidation จาก 068), Pagination แบบ Cursor แลกกับความสามารถกระโดดไปหน้า
   กลาง ๆ ไม่ได้ (071), Connection Pool ใหญ่แลกกับ context switching overhead
   (715) — ไม่มีคำตอบที่ถูกเสมอไปในทุกสถานการณ์ ต้องเข้าใจ trade-off แล้วเลือก
   ให้เหมาะกับระบบของตัวเอง

### 720.4 Quiz ทบทวน Phase 8 (12 ข้อ พร้อมเฉลย)

**คำถามที่ 1**: เครื่องมือใดใน Part 066 ใช้แสดงจำนวน SQL query ที่เกิดขึ้นจริงใน
แต่ละหน้าเว็บระหว่างพัฒนา พร้อม stack trace ของแต่ละ query?

> **เฉลย**: `django-debug-toolbar` (แสดงผลผ่าน panel บนหน้าเว็บโดยตรง) และ
> `django-silk` (เก็บข้อมูลลงฐานข้อมูลเพื่อดูย้อนหลังและวิเคราะห์เชิงลึกกว่า)

**คำถามที่ 2**: `select_related()` และ `prefetch_related()` ต่างกันอย่างไร และ
ควรใช้กับความสัมพันธ์แบบไหน?

> **เฉลย**: `select_related()` ใช้ SQL `JOIN` ทำงานกับความสัมพันธ์ ForeignKey/
> OneToOne (ฝั่ง "1") เพราะ join แล้วได้ผลลัพธ์แถวเดียวต่อ object หลัก ส่วน
> `prefetch_related()` ยิง query แยกต่างหากแล้วนำมาประกอบกันใน Python ใช้กับ
> ManyToMany หรือ reverse ForeignKey (ฝั่ง "หลาย") ที่ join ตรงจะทำให้แถวข้อมูล
> ซ้ำซ้อนกัน

**คำถามที่ 3**: ถ้าลืมใส่ `format="json"` (หรือใช้ `TEST_REQUEST_DEFAULT_FORMAT`)
ในการเก็บ cache ด้วย `{% cache %}` ทำไมทุกคนถึงเห็นข้อมูลเดียวกันแม้เนื้อหาควรต่าง
กันตาม category?

> **เฉลย**: เพราะลืมใส่ `vary_on` argument (เช่น `category.slug`) ให้ `{% cache
> %}` tag ทำให้ Django คำนวณ cache key เดียวกันสำหรับทุกค่าของ category — ต้อง
> ใส่ตัวแปรที่ทำให้เนื้อหาต่างกันเป็น `vary_on` เสมอ (ทวนจาก Part 068 ข้อ 673.5)

**คำถามที่ 4**: เพราะเหตุใด `cache.get_or_set('key', _fetch(), timeout=300)`
(เรียก `_fetch()` ทันที) จึงเป็นข้อผิดพลาดที่พบบ่อย เทียบกับ `cache.get_or_set
('key', _fetch, timeout=300)`?

> **เฉลย**: การเรียก `_fetch()` ทันทีทำให้ query/computation เกิดขึ้น**ทุกครั้ง**
> ไม่ว่า cache จะ hit หรือ miss เพราะ Python ประเมินค่า argument ก่อนส่งเข้า
> ฟังก์ชันเสมอ ในขณะที่ส่ง `_fetch` (ไม่มีวงเล็บ) จะถูกเรียกก็ต่อเมื่อ cache
> miss เท่านั้น ทำให้ประโยชน์ของ caching หายไปครึ่งหนึ่งถ้าเขียนผิดแบบแรก

**คำถามที่ 5**: `CONN_MAX_AGE` ของ Django กับ PgBouncer แก้ปัญหาเดียวกันหรือไม่
อย่างไร?

> **เฉลย**: ไม่เหมือนกันทั้งหมด `CONN_MAX_AGE` ลดการเปิด/ปิด connection บ่อย
> **ต่อ 1 worker** แต่จำนวน connection สูงสุดยังเท่ากับจำนวน worker ทั้งหมดอยู่ดี
> ส่วน PgBouncer ทำให้ worker **จำนวนมาก** (หลักร้อย-พัน) ใช้ connection จริง
> **จำนวนน้อย** (หลักสิบ) ร่วมกันผ่าน pool — PgBouncer แก้ปัญหาที่ `CONN_MAX_AGE`
> แก้ไม่ได้เมื่อ scale ไปหลาย server/worker จำนวนมาก

**คำถามที่ 6**: ทำไมการดูค่าเฉลี่ย (Average) ของ Response Time ถึงอันตราย ควรดู
อะไรแทน?

> **เฉลย**: ค่าเฉลี่ยถูกดึงลงได้ง่ายด้วย request ส่วนใหญ่ที่เร็ว จนซ่อน request
> ส่วนน้อยที่ช้ามาก (long tail) ไว้ ควรดู **Percentile** แทน (p95 เป็นอย่างน้อย,
> ระบบใหญ่ควรดู p99 ด้วย) เพราะสะท้อนประสบการณ์ของผู้ใช้กลุ่มที่แย่ที่สุดได้จริง

**คำถามที่ 7**: สูตรคำนวณจำนวน Gunicorn Worker มาตรฐานคืออะไร และเมื่อไหร่ที่
สูตรนี้ใช้ไม่ได้ผลดี?

> **เฉลย**: `(2 × จำนวน CPU core) + 1` ใช้ได้ผลดีกับ workload แบบ `sync` worker
> ที่เป็น CPU-bound เป็นหลัก แต่ใช้ไม่ได้ผลดีกับงานที่ I/O-bound หนัก (รอ
> external API/database นาน) ซึ่งควรเปลี่ยนไปใช้ `gevent`/`eventlet` worker
> class แทนการเพิ่มจำนวน `sync` worker เรื่อย ๆ

**คำถามที่ 8**: ทำไม Connection Pool ที่ใหญ่เกินไปถึงทำให้ Performance แย่ลง
แทนที่จะดีขึ้น?

> **เฉลย**: เพราะ PostgreSQL ใช้โมเดล 1 process ต่อ 1 connection การมี
> connection พร้อมกันมากเกินกว่าจำนวน CPU core ที่ประมวลผลได้จริง ทำให้เกิด
> **context switching overhead** ระหว่าง process จำนวนมาก ซึ่งกิน CPU ไปกับการ
> สลับงานแทนที่จะเอาไปประมวลผล query จริง ทำให้ throughput รวมลดลง

**คำถามที่ 9**: ก่อนทำ Horizontal Scaling แอป Django ต้องมีคุณสมบัติอะไรเป็น
พื้นฐาน และทำไม?

> **เฉลย**: แอปต้องเป็น **Stateless** คือไม่เก็บ session หรือไฟล์อัปโหลดไว้ใน
> memory/disk ของเครื่องใดเครื่องหนึ่งโดยเฉพาะ ต้องย้าย session/cache ไปที่
> Redis กลาง และย้าย media files ไปที่ Object Storage กลาง ไม่เช่นนั้น request
> ที่ Load Balancer ส่งไปคนละเครื่องกันของ user คนเดียวกันจะเห็นข้อมูลไม่ตรงกัน

**คำถามที่ 10**: อะไรคือความแตกต่างหลักที่ทำให้ k6 เหมาะกับการฝังใน CI Pipeline
มากกว่า Locust?

> **เฉลย**: k6 มี `options.thresholds` ในตัวที่กำหนดเงื่อนไข pass/fail ได้ตรง ๆ
> ใน script (เช่น `p(95)<400`) ทำให้ exit code ของ k6 บอกผลลัพธ์อัตโนมัติว่า
> "ผ่านเกณฑ์ performance หรือไม่" โดยไม่ต้องเขียน logic เช็คเพิ่มเอง ในขณะที่
> Locust ไม่มี concept นี้ในตัว ต้องเขียนโค้ดแยกไปอ่านผลลัพธ์และตัดสินเองอีกที

**คำถามที่ 11**: เมื่อรัน Load Test แล้วพบว่าเปิด caching ทำให้ RPS เพิ่มขึ้น
มหาศาลเกือบ 100% ตั้งแต่ user คนที่สอง ควรสงสัยอะไรเกี่ยวกับการออกแบบ test
scenario?

> **เฉลย**: ควรสงสัยว่า test scenario อาจให้ virtual user ทุกคนเข้าหน้าเดียวกัน
> ซ้ำ ๆ (เช่น URL เดียวไม่มีการสุ่ม) ทำให้ cache hit rate สูงเกินจริงเมื่อเทียบกับ
> พฤติกรรมผู้ใช้จริงที่เข้าหลายหน้าที่แตกต่างกัน ควรออกแบบ scenario ให้มีความ
> หลากหลายของ URL/query string เพื่อให้สัดส่วน cache hit/miss สมจริงกว่านี้

**คำถามที่ 12**: Read Replica ของฐานข้อมูลช่วยแก้ปัญหาอะไร และไม่เหมาะกับระบบ
แบบไหน?

> **เฉลย**: Read Replica ช่วยกระจาย query ประเภทอ่าน (`SELECT`) ออกจากฐานข้อมูล
> หลักไปยังฐานข้อมูลสำเนา ทำให้รองรับ traffic อ่านได้มากขึ้นโดยไม่เพิ่มภาระ
> ฐานข้อมูลหลักที่รับผิดชอบงานเขียน เหมาะกับระบบที่อัตราส่วนอ่าน:เขียนสูงมาก
> (เช่น บล็อก, เว็บข่าว) แต่ไม่ช่วยอะไรกับระบบที่มีสัดส่วนงานเขียนเยอะพอ ๆ กับ
> อ่าน (เช่น ระบบซื้อขายหุ้นแบบ real-time) เพราะงานเขียนยังต้องผ่านฐานข้อมูลหลัก
> เพียงตัวเดียวอยู่ดี

### 720.5 แบบฝึกหัดใหญ่ปิดท้าย Phase 8: Performance Audit เต็มรูปแบบของ Blog Project

รวบยอดทุกทักษะจาก Part 066-072 ให้คุณทำ **Performance Audit แบบมืออาชีพ**
ให้กับโปรเจกต์ Blog ที่สร้างมาตลอดหลักสูตร ทำตามลำดับขั้นตอนต่อไปนี้ และเก็บ
ผลลัพธ์ของแต่ละขั้นเป็นรายงานสั้น ๆ (ไฟล์ `PERFORMANCE_AUDIT.md` ในโปรเจกต์):

**ขั้นที่ 1 — Baseline Profiling (ทวน Part 066)**

ติดตั้ง `django-silk` เปิด profiling บนหน้า `/blog/` (list) และ `/blog/<slug>/`
(detail) บันทึกจำนวน query, เวลารวมที่ใช้ query, และเวลา render ทั้งหมดของแต่ละ
หน้า **ก่อนแก้ไขอะไรเลย**

**ขั้นที่ 2 — แก้ N+1 Query (ทวน Part 067)**

หาจุดที่มี N+1 query ในหน้ารายการบทความ (มักเกิดจากการเข้าถึง
`post.category.name` หรือ `post.comments.count()` ใน loop ของ template โดยไม่มี
`select_related`/`prefetch_related`) แก้ไขแล้ววัดผลซ้ำด้วย `django-silk`
เปรียบเทียบจำนวน query ก่อน/หลัง

**ขั้นที่ 3 — เพิ่ม Database Index (ทวน Part 070)**

ใช้ `EXPLAIN ANALYZE` ตรวจสอบ query ที่ยัง query ฐานข้อมูลอยู่ (หลัง cache miss)
ว่ามี Sequential Scan บน field ที่ใช้ filter/order บ่อยหรือไม่ (เช่น
`is_published`, `created_at`, `category_id`) เพิ่ม index ที่เหมาะสมแล้ววัดผลซ้ำ

**ขั้นที่ 4 — เพิ่ม Caching ครบ 3 ระดับ (ทวน Part 068-069)**

ใส่ `@cache_page` ให้หน้ารายการบทความ, `{% cache %}` ให้ sidebar หมวดหมู่ (พร้อม
`vary_on` ที่ถูกต้อง), และ Low-level Cache API สำหรับค่าที่คำนวณหนัก (เช่น
จำนวนบทความทั้งหมด) ตั้งค่า Redis เป็น backend และเขียน Django Signal
invalidate cache อัตโนมัติเมื่อมีการแก้ไข `Post`/`Category`

**ขั้นที่ 5 — ตรวจสอบ Pagination (ทวน Part 071)**

ยืนยันว่าหน้ารายการบทความและ API endpoint ใช้ pagination ที่เหมาะสมกับขนาด
dataset (PageNumberPagination สำหรับหน้าเว็บทั่วไป, Cursor Pagination สำหรับ
API ที่ dataset ใหญ่มากหรือมีการเขียนข้อมูลใหม่บ่อย)

**ขั้นที่ 6 — เขียน Locust Script ที่สมจริง (ทวน ขั้อ 712)**

เขียน `locustfile.py` ที่จำลอง traffic ผสมทั้งผู้เข้าชมทั่วไป (weight สูง) และ
ผู้เขียนที่ login แล้ว (weight ต่ำ) พร้อมสุ่มเข้าหลาย URL ที่หลากหลายตามขั้อ
718.5

**ขั้นที่ 7 — รัน Load Test ก่อน/หลัง แล้วหา Breaking Point (ทวน ขั้อ 714, 718)**

รัน `StepLoadShape` เปรียบเทียบระหว่าง branch ก่อนทำ audit ทั้งหมด กับ branch
หลังทำครบทุกขั้น บันทึกตาราง RPS/p50/p95/p99/Error Rate ที่แต่ละระดับ
concurrency แล้วระบุว่า Breaking Point ขยับไปที่ระดับใด

**ขั้นที่ 8 — ปรับจูน Gunicorn และ Connection Pool (ทวน ขั้อ 715-716)**

ทดลองปรับจำนวน Gunicorn worker และขนาด PgBouncer pool ตามสูตรเริ่มต้นในขั้อ
715.4 และ 716.3 แล้วทดสอบซ้ำเพื่อหาค่าที่เหมาะสมที่สุดสำหรับสเปกเครื่อง staging
ที่มีอยู่

**ขั้นที่ 9 — เขียนรายงานสรุป**

สรุปในไฟล์ `PERFORMANCE_AUDIT.md` ว่า: (1) Bottleneck เดิมคืออะไร (2) แก้ไปกี่จุด
ด้วยเทคนิคอะไรบ้าง (3) ตัวเลข RPS/p95/Breaking Point ก่อน-หลังต่างกันเท่าไหร่
(4) มี trade-off อะไรที่ต้องยอมรับบ้าง (เช่น ความสด/เก่าของข้อมูลที่ cache ไว้)

**เกณฑ์ความสำเร็จ**: รายงานควรแสดงให้เห็นว่าระบบรองรับ concurrent user เพิ่มขึ้น
อย่างมีนัยสำคัญ (เป้าหมายอ้างอิง: อย่างน้อย 2 เท่า) โดยที่ p95 Response Time ที่
ระดับ concurrency เดิมลดลงอย่างชัดเจน และไม่มี error rate เพิ่มขึ้นจากการ
เปลี่ยนแปลงใด ๆ ที่ทำไป

### 720.6 เตรียมตัวสำหรับ Phase 9: Async, Celery และ Channels

Phase 8 ปิดฉากลงด้วยระบบ Blog ที่**เร็วและรองรับ concurrent user ได้สูงกว่าเดิม
มาก** แต่มีข้อจำกัดหนึ่งที่ยังไม่ได้แตะเลยตลอด 72 Part ที่ผ่านมา: **ทุก request
ยังคงเป็นแบบ synchronous** — เมื่อ view ต้องทำงานที่ใช้เวลานาน (ส่งอีเมล, เรียก
API ภายนอก, ประมวลผลไฟล์ขนาดใหญ่, สร้างรายงาน PDF) ผู้ใช้ต้อง**รอจนกว่างานนั้น
จะเสร็จ**ก่อนถึงจะได้ response กลับมา ทั้งที่งานเหล่านี้ไม่จำเป็นต้องทำให้เสร็จ
ทันทีต่อหน้าผู้ใช้เลย

**Phase 9: Async, Celery และ Channels (Part 073-079)** จะพาคุณแก้ข้อจำกัดนี้
อย่างเป็นระบบ:

- **Part 073: Async Views และ ASGI เบื้องต้น** — เปลี่ยน view จาก synchronous
  เป็น `async def` เพื่อจัดการ I/O-bound work หลายงานพร้อมกันโดยไม่บล็อก worker
  (ทวนแนวคิดที่ขั้อ 716.6 ของ Part นี้เกริ่นไว้ว่า `gevent` ช่วยได้แค่ไหน — Part
  073 จะพาไปถึงทางออกที่เป็นมาตรฐานสมัยใหม่กว่าคือ ASGI แท้)
- **Part 074: Django Channels ขั้นสูง** — WebSocket สำหรับ real-time feature
  (แชท, notification สด ๆ)
- **Part 075-076: Celery** — ย้ายงานหนักที่ **ไม่จำเป็นต้องรอ** ออกจาก request-
  response cycle ไปทำเป็น Background Task แยกต่างหาก (ส่งอีเมล, สร้างรายงาน,
  ประมวลผลรูปภาพ) พร้อม Periodic Tasks และ Task Chains
- **Part 077: Message Queue ด้วย RabbitMQ/Redis** — โครงสร้างพื้นฐานที่ Celery
  ใช้ส่งงานระหว่าง process
- **Part 078-079: Real-time Notification และ Background Job Monitoring** —
  ประกอบทุกอย่างเข้าด้วยกันเป็นระบบแจ้งเตือนแบบ real-time พร้อมเครื่องมือ monitor
  งาน background ด้วย Flower

สิ่งที่คุณเรียนรู้ใน Phase 8 (โดยเฉพาะ Redis จาก Part 069) จะเป็นรากฐานสำคัญของ
Phase 9 ทันที เพราะ **Redis คือ message broker มาตรฐานที่ Celery ใช้บ่อยที่สุด**
— เท่ากับว่าคุณได้ตั้งค่าโครงสร้างพื้นฐานสำคัญไว้ล่วงหน้าแล้วโดยไม่รู้ตัว พร้อม
สำหรับ Part 073 ที่กำลังจะเริ่มต้นทันที

---

## เตรียมตัวสำหรับ Part ถัดไป

**Part 073: Async Views และ ASGI เบื้องต้น** จะเริ่มต้น Phase 9 ด้วยการอธิบาย
ความแตกต่างระหว่าง WSGI ที่ใช้มาตลอดหลักสูตรกับ ASGI, วิธีเขียน `async def` view
ตัวแรก, และเมื่อไหร่ที่ async view ให้ประโยชน์จริง (I/O-bound) เทียบกับตอนที่ไม่
ช่วยอะไรเลย (CPU-bound) — เตรียมทบทวนพื้นฐาน `async`/`await` ของ Python จาก
Part 002 ไว้ให้แม่นก่อนไปต่อ
