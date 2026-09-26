# Part 076: Celery ขั้นสูง: Periodic Tasks, Chains, Chords

> **ขั้นตอนที่ 751-760 ของหลักสูตร** | Phase 9: Async, Celery และ Channels
>
> เป้าหมายของ Part นี้: ต่อยอดจาก Celery พื้นฐาน (Part 075) ที่คุณรู้จัก `@shared_task`,
> การรัน worker, และการยิงงานด้วย `.delay()` แล้ว เข้าสู่การใช้งาน Celery แบบมืออาชีพเต็ม
> รูปแบบ — ตั้งเวลารัน task อัตโนมัติด้วย **Celery Beat** (แทนที่ cron แบบเก่าตามที่ Part
> 069 ขั้นตอนที่ 682.3 เกริ่นไว้), จัดการ schedule ผ่านฐานข้อมูลด้วย `django-celery-beat`
> เพื่อแก้ไขได้จาก Admin โดยไม่ต้อง deploy ใหม่, เชื่อม task หลายตัวเข้าด้วยกันด้วย
> **Chain**, รันแบบขนานด้วย **Group**, รวมผลลัพธ์ทั้งหมดเข้า callback เดียวด้วย
> **Chord**, จัดสรร task ไปคนละ queue ตามความสำคัญ, ป้องกัน task ค้างด้วย time limit,
> ออกแบบ task ให้ retry ซ้ำได้อย่างปลอดภัย (idempotent), และเขียน test สำหรับ Celery
> task อย่างถูกต้อง เมื่อจบ Part นี้ คุณจะสร้าง pipeline ประมวลผลข้อมูลแบบขนานและมีลำดับ
> ขั้นตอนที่ซับซ้อนได้ พร้อมระบบตั้งเวลาที่ทีมที่ไม่ใช่โปรแกรมเมอร์ก็แก้ไขได้เอง

---

## สารบัญของ Part นี้

- ขั้นตอนที่ 751: Celery Beat — ตั้งเวลารัน Task แบบ cron (periodic tasks)
- ขั้นตอนที่ 752: `django-celery-beat` — จัดการ schedule ผ่านฐานข้อมูล (แก้ผ่าน Admin ได้โดยไม่ต้อง deploy ใหม่)
- ขั้นตอนที่ 753: Task Chain — รัน task ต่อกันเป็นลำดับ (`chain()`, ผลลัพธ์ task แรกส่งต่อให้ task ถัดไป)
- ขั้นตอนที่ 754: Task Group — รันหลาย task พร้อมกันแบบขนาน (`group()`)
- ขั้นตอนที่ 755: Chord — รูปแบบ group + callback (รอทุก task ใน group เสร็จก่อนแล้วค่อยรัน callback)
- ขั้นตอนที่ 756: Task Routing — ส่ง task ไปคนละ queue ตามความสำคัญ, priority queue
- ขั้นตอนที่ 757: Time Limit ของ Task — `soft_time_limit`/`time_limit` ป้องกัน task ค้าง
- ขั้นตอนที่ 758: Idempotency ใน Task — ออกแบบ task ให้ retry ซ้ำได้อย่างปลอดภัยโดยไม่เกิดผลข้างเคียงซ้ำ
- ขั้นตอนที่ 759: การเขียน Test สำหรับ Celery Task — `CELERY_TASK_ALWAYS_EAGER=True`
- ขั้นตอนที่ 760: สรุปและแบบฝึกหัด — สร้าง pipeline สร้างรายงานสถิติบล็อกรายสัปดาห์ด้วย Chain/Chord

---

## ขั้นตอนที่ 751: Celery Beat — ตั้งเวลารัน Task แบบ cron (periodic tasks)

### 751.1 ทวนสถานะจาก Part 075

ใน Part 075 คุณได้ติดตั้ง Celery พื้นฐานแล้ว มี `config/celery.py` ที่สร้าง Celery app,
เชื่อม Redis เป็น broker, เขียน task ด้วย `@shared_task` และยิงงานแบบ **สั่งครั้งเดียว
เมื่อมีเหตุการณ์เกิดขึ้น** เช่น ผู้ใช้กด submit ฟอร์มแล้วค่อยยิง `send_welcome_email.delay(user.id)`

โครงสร้างที่ควรมีอยู่แล้วจาก Part 075:

```python
# config/celery.py
import os

from celery import Celery

os.environ.setdefault('DJANGO_SETTINGS_MODULE', 'config.settings')

app = Celery('config')
app.config_from_object('django.conf:settings', namespace='CELERY')
app.autodiscover_tasks()
```

```python
# config/__init__.py
from .celery import app as celery_app

__all__ = ('celery_app',)
```

```python
# config/settings.py
CELERY_BROKER_URL = os.environ.get('CELERY_BROKER_URL', 'redis://127.0.0.1:6379/4')
CELERY_RESULT_BACKEND = os.environ.get('CELERY_RESULT_BACKEND', 'redis://127.0.0.1:6379/5')
CELERY_ACCEPT_CONTENT = ['json']
CELERY_TASK_SERIALIZER = 'json'
CELERY_RESULT_SERIALIZER = 'json'
CELERY_TIMEZONE = 'Asia/Bangkok'
```

> **สังเกต DB index**: ทวนหลักการแยก Redis DB index จาก Part 069 ขั้นตอนที่ 681.5 —
> เราใช้ DB 4 สำหรับ broker (คิวงาน) และ DB 5 สำหรับ result backend (เก็บผลลัพธ์) แยก
> จาก DB 1-3 ที่ใช้เป็น page cache/throttle/session ไปแล้ว เพื่อไม่ให้ `FLUSHDB` หรือการ
> monitor ส่วนหนึ่งไปกระทบอีกส่วนหนึ่งโดยไม่ตั้งใจ

ส่วนที่ Part นี้จะเพิ่มเข้ามาคือ: **จะเกิดอะไรขึ้นถ้าเราต้องการให้ task รันเอง
"ตามเวลา" โดยไม่มีผู้ใช้คนไหนมากระตุ้น?** เช่น "ทุกวันเที่ยงคืน ให้สรุปยอดขายของวันนั้น",
"ทุก 5 นาที ให้ sync ยอด view จาก Redis Hash กลับเข้าฐานข้อมูลจริง" (ตามที่ Part 069
ขั้นตอนที่ 682.3 ทิ้งท้ายไว้ว่าจะมาเรียนใน Part นี้), หรือ "ทุกวันจันทร์เช้า ให้ส่งอีเมล
สรุปสถิติบล็อกประจำสัปดาห์" — นี่คือหน้าที่ของ **Celery Beat**

### 751.2 Celery Beat คืออะไร และทำไมไม่ควรใช้ cron ธรรมดา

**Celery Beat** คือ scheduler กระบวนการแยกต่างหาก (separate process) ที่มีหน้าที่เดียว:
**อ่านตาราง schedule แล้วยิง task เข้าคิวตามเวลาที่กำหนด** มันไม่ได้รันงานเอง — แค่ส่ง
งานเข้าคิวให้ Celery worker ที่มีอยู่แล้วไปหยิบไปทำงานตามปกติ

หลายคนสงสัยว่า "ทำไมไม่ใช้ `cron` ของระบบปฏิบัติการเลย ก็ตั้งเวลารันสคริปต์ Python ได้
เหมือนกัน?" คำตอบคือ cron ใช้ได้สำหรับระบบเล็ก ๆ แต่มีข้อจำกัดสำคัญเมื่อระบบโตขึ้น:

| ประเด็น | `cron` ของระบบปฏิบัติการ | Celery Beat |
|---|---|---|
| รันบนเครื่องไหน | ต้องรันบนเครื่องใดเครื่องหนึ่งเจาะจง (ปกติคือ server หลัก) | รันบนเครื่องไหนก็ได้ในระบบ แค่ยิงเข้าคิวกลาง |
| ถ้าเครื่องที่รัน cron ล่ม | Task ทั้งหมดไม่รันจนกว่าจะแก้เครื่องนั้น (single point of failure) | ตั้ง Beat สำรอง หรือใช้ `django-celery-beat` กับ leader election ได้ |
| Retry เมื่องานล้มเหลว | ต้องเขียนเอง (ปกติไม่มี retryในตัว) | ใช้กลไก retry ของ Celery ที่มีอยู่แล้วได้ทันที (`autoretry_for`, `max_retries`) |
| เชื่อมกับโค้ด Django | ต้องเรียก `python manage.py <command>` แยกกระบวนการ เสีย overhead boot Django ทุกครั้ง | Task รันในบริบท Celery worker ที่ boot Django ไว้แล้วล่วงหน้า เร็วกว่า |
| Concurrency / Queue | รันตรงในเครื่องนั้น ไม่ผ่านคิว ถ้างานหนักจะแย่ง resource กับเว็บแอปโดยตรง | งานเข้าคิว กระจายไปยัง worker หลายเครื่องได้ ไม่กระทบเว็บแอปหลัก |
| แก้ schedule โดยไม่ downtime | ต้อง SSH เข้าเครื่อง แก้ crontab เอง | แก้ผ่าน Django Admin ได้ถ้าใช้ `django-celery-beat` (ขั้นตอนที่ 752) |
| Log และ Monitoring รวมศูนย์ | กระจัดกระจายตามแต่ละเครื่อง | รวมอยู่ใน Celery logging/Flower (Part 079) เหมือน task อื่น ๆ |

**สรุปหลักการ**: cron เหมาะกับ script เดี่ยว ๆ ที่ไม่เกี่ยวกับแอปหลัก (เช่น backup
ฐานข้อมูลทั้งเครื่อง) ส่วน Celery Beat เหมาะกับงานที่เป็นส่วนหนึ่งของ business logic
ของแอป Django และต้องการความน่าเชื่อถือ/retry/scale เหมือน task อื่น ๆ ในระบบ

### 751.3 ติดตั้งและตั้งค่า Celery Beat แบบพื้นฐาน (ไฟล์ config ในโค้ด)

Celery Beat มาพร้อมกับ Celery อยู่แล้ว ไม่ต้องติดตั้งเพิ่ม (แพ็กเกจ `django-celery-beat`
ในขั้นตอนที่ 752 เป็นส่วนเสริมสำหรับเก็บ schedule ในฐานข้อมูลเท่านั้น) กำหนด schedule
เริ่มต้นแบบ **hardcode ในไฟล์ settings** ก่อน (ยังไม่ผ่านฐานข้อมูล) เพื่อให้เข้าใจกลไก
พื้นฐานก่อน:

```python
# config/settings.py
from celery.schedules import crontab

CELERY_BEAT_SCHEDULE = {
    'sync-post-view-stats-every-5-minutes': {
        'task': 'blog.tasks.sync_post_view_stats',
        'schedule': 300.0,   # หน่วยเป็นวินาที: ทุก 300 วินาที (5 นาที)
    },
    'send-weekly-digest-every-monday-7am': {
        'task': 'blog.tasks.send_weekly_digest_email',
        'schedule': crontab(hour=7, minute=0, day_of_week=1),   # จันทร์ = 1
    },
    'cleanup-expired-tokens-daily-midnight': {
        'task': 'accounts.tasks.cleanup_expired_tokens',
        'schedule': crontab(hour=0, minute=0),   # ทุกวันเที่ยงคืน
    },
}
```

รูปแบบ `schedule` มี 2 แบบหลัก:

| รูปแบบ | ตัวอย่าง | ความหมาย |
|---|---|---|
| ตัวเลข (วินาที) หรือ `timedelta` | `300.0` หรือ `timedelta(minutes=5)` | รันซ้ำทุก ๆ ช่วงเวลาคงที่ นับจากตอนที่ Beat เริ่มทำงาน |
| `crontab(...)` | `crontab(hour=7, minute=0, day_of_week=1)` | รันตามเวลานาฬิกาจริงแบบ cron — เที่ยงตรง ไม่ใช่นับถอยหลังจากตอนสตาร์ท |

พารามิเตอร์ของ `crontab()` ทั้งหมด (ค่า default คือ `*` = ทุกค่า):

```python
crontab(
    minute='*',       # 0-59
    hour='*',         # 0-23
    day_of_week='*',  # 0-6 (อาทิตย์=0) หรือ 1-7 (จันทร์=1) — Celery รองรับทั้งคู่
    day_of_month='*', # 1-31
    month_of_year='*',# 1-12
)

# ตัวอย่างที่ใช้บ่อย
crontab(minute=0, hour='*/2')            # ทุก 2 ชั่วโมง ณ นาที 0
crontab(minute='*/15')                   # ทุก 15 นาที
crontab(hour=9, minute=0, day_of_month=1)  # วันที่ 1 ของทุกเดือน เวลา 09:00
```

### 751.4 การรัน Celery Beat คู่กับ Worker

Beat และ Worker เป็นคนละกระบวนการ ต้องรันแยกกันเสมอในระบบ production:

```bash
# Terminal 1: รัน Worker (หยิบงานจากคิวไปทำจริง — เหมือน Part 075)
celery -A config worker --loglevel=info

# Terminal 2: รัน Beat (อ่าน schedule แล้วยิงงานเข้าคิวตามเวลา)
celery -A config beat --loglevel=info
```

เมื่อ Beat ยิง task เข้าคิวตามเวลาแล้ว งานจะไปรอที่คิว Redis เหมือน task ปกติทุกประการ
Worker จะหยิบไปทำ **Beat ไม่ได้รันโค้ดเอง มันแค่ "กดปุ่มส่งงาน" ตามเวลาเท่านั้น**

> **คำเตือนสำคัญที่สุดของ Celery Beat**: **ห้ามรัน Beat มากกว่า 1 instance พร้อมกัน
> เด็ดขาด** ถ้ารัน Beat 2 ตัวพร้อมกัน (เช่น deploy ผิดพลาดทำให้มี process ซ้ำ) ทุก
> schedule จะถูกยิงเข้าคิว **ซ้ำสองเท่า** เช่น อีเมลสรุปรายสัปดาห์จะถูกส่งซ้ำ 2 ฉบับ
> ให้ผู้ใช้ทุกคน วิธีป้องกันเวลา deploy บน container orchestration (Kubernetes) คือ
> ตั้งค่าให้ Beat เป็น **Deployment ที่มี replica ตายตัวเท่ากับ 1 เสมอ** (ไม่ใช่
> autoscale เหมือน worker) — เราจะเจาะลึกการ deploy Celery ใน Phase DevOps

สำหรับเครื่องพัฒนา (development) เท่านั้น สามารถรวม worker + beat ในกระบวนการเดียวด้วย
flag `-B` เพื่อความสะดวก:

```bash
# ใช้เฉพาะตอนพัฒนา ห้ามใช้ใน production เด็ดขาด (เพราะ scale worker แยกจาก beat ไม่ได้)
celery -A config worker -B --loglevel=info
```

### 751.5 ตัวอย่างจริง: Sync ยอด View จาก Redis Hash กลับเข้าฐานข้อมูล (ต่อจาก Part 069)

นี่คือ task ที่ Part 069 ขั้นตอนที่ 682.3 สัญญาไว้ว่าจะมาสอนใน Part นี้ ทวนความจำ:
เราเก็บยอด view/like ของแต่ละโพสต์ไว้ใน Redis Hash เพื่อรับ traffic เขียนหนัก ๆ โดยไม่ยิง
`UPDATE` เข้า PostgreSQL ทุกครั้ง แต่ข้อมูลใน Redis เป็นแค่ **cache ชั่วคราว** สุดท้ายต้อง
sync กลับเข้าฐานข้อมูลจริงเป็นระยะ เพื่อไม่ให้ข้อมูลหายถ้า Redis restart (แม้จะตั้ง
`appendonly yes` ไว้แล้วก็ตาม ฐานข้อมูลหลักคือแหล่งความจริงสุดท้ายเสมอ — source of truth):

```python
# blog/tasks.py
import logging

from celery import shared_task
from django.db import transaction
from django_redis import get_redis_connection

from .models import Post

logger = logging.getLogger(__name__)

STATS_KEY_PATTERN = 'post_stats:*'


@shared_task(name='blog.tasks.sync_post_view_stats')
def sync_post_view_stats():
    """
    รันทุก 5 นาทีผ่าน Celery Beat — อ่านยอด views/likes ทั้งหมดจาก Redis Hash
    (ที่ Part 069 ขั้นตอนที่ 682.3 เขียนด้วย HINCRBY) แล้ว sync กลับเข้า PostgreSQL
    ใช้ bulk_update เพื่อลดจำนวนรอบ query ให้เหลือน้อยที่สุดแม้จะมีหลายร้อยโพสต์
    """
    con = get_redis_connection("default")
    updated_count = 0
    posts_to_update = []

    # SCAN แทน KEYS เสมอใน production เพราะ KEYS บล็อก Redis ทั้งตัวจนกว่าจะสแกนเสร็จ
    for raw_key in con.scan_iter(match=STATS_KEY_PATTERN, count=100):
        key = raw_key.decode('utf-8')
        post_id = int(key.split(':')[1])
        raw = con.hgetall(key)

        views = int(raw.get(b'views', 0))
        likes = int(raw.get(b'likes', 0))

        try:
            post = Post.objects.get(pk=post_id)
        except Post.DoesNotExist:
            logger.warning('Post id=%s มีใน Redis stats แต่ไม่มีในฐานข้อมูลแล้ว ข้าม', post_id)
            continue

        post.views_count = views
        post.likes_count = likes
        posts_to_update.append(post)
        updated_count += 1

    if posts_to_update:
        with transaction.atomic():
            Post.objects.bulk_update(posts_to_update, ['views_count', 'likes_count'])

    logger.info('sync_post_view_stats: sync สำเร็จ %d โพสต์', updated_count)
    return {'synced_posts': updated_count}
```

เพิ่มเข้า schedule:

```python
# config/settings.py
CELERY_BEAT_SCHEDULE = {
    'sync-post-view-stats-every-5-minutes': {
        'task': 'blog.tasks.sync_post_view_stats',
        'schedule': 300.0,
    },
}
```

> **ทำไมต้อง sync ทุก 5 นาที ไม่ใช่ทุกวินาที**: ยิ่ง sync ถี่ ยิ่งใกล้เคียงข้อมูลจริงมาก
> ขึ้น แต่ก็สร้างภาระ query ฐานข้อมูลมากขึ้นตามไปด้วย (bulk_update ทุกโพสต์ที่มีการ
> เปลี่ยนแปลง) 5 นาทีเป็นจุดสมดุลที่ดีสำหรับสถิติที่ไม่จำเป็นต้อง real-time เป๊ะ (ต่างจาก
> ยอดเงินในบัญชีที่ต้องแม่นยำทันที) ถ้าธุรกิจต้องการความสดใหม่กว่านี้ ให้ปรับตัวเลขนี้ได้
> ตามความเหมาะสม โดยเฝ้าดู query load ของฐานข้อมูลควบคู่ไปด้วย

### 751.6 ตารางสรุปขั้นตอนที่ 751

| หัวข้อ | สรุป |
|---|---|
| Celery Beat คืออะไร | กระบวนการแยกที่อ่าน schedule แล้วยิง task เข้าคิวตามเวลา ไม่ได้รันงานเอง |
| ทำไมดีกว่า cron | Retry ในตัว, ทำงานผ่านคิวกลาง, scale ได้, log รวมศูนย์, แก้ schedule ได้โดยไม่ downtime (ขั้นตอนที่ 752) |
| ตั้ง schedule แบบพื้นฐาน | `CELERY_BEAT_SCHEDULE` dict ใน settings.py |
| รูปแบบเวลา | ตัวเลข/`timedelta` (นับถอยหลังจากสตาร์ท) หรือ `crontab()` (ตามนาฬิกาจริง) |
| คำสั่งรัน | `celery -A config beat` (แยกจาก `celery -A config worker` เสมอใน production) |
| กฎเหล็ก | ห้ามรัน Beat มากกว่า 1 instance พร้อมกันเด็ดขาด |

---

## ขั้นตอนที่ 752: `django-celery-beat` — จัดการ schedule ผ่านฐานข้อมูล

### 752.1 ปัญหาของการ hardcode schedule ในไฟล์ settings

วิธีในขั้นตอนที่ 751 ใช้งานได้ดี แต่มีข้อจำกัดสำคัญ: **ทุกครั้งที่ต้องการแก้เวลา** (เช่น
เปลี่ยนจากส่งอีเมลสรุปทุกวันจันทร์ เป็นทุกวันศุกร์) **ต้องแก้โค้ดแล้ว deploy ใหม่ทั้งระบบ**
ในทีมจริง ผู้ที่อยากปรับเวลาส่งอีเมลอาจเป็นทีมการตลาดที่ไม่ได้เขียนโค้ด หรือแม้แต่
โปรแกรมเมอร์เองก็ไม่อยากต้อง deploy ทั้งระบบแค่เพื่อเปลี่ยนตัวเลขนาทีตัวเดียว

**`django-celery-beat`** แก้ปัญหานี้โดยเก็บ schedule ทั้งหมดไว้ใน **ฐานข้อมูล** แทนไฟล์
โค้ด แล้วให้ Beat อ่านจากฐานข้อมูลแทน ทำให้แก้ไขผ่าน **Django Admin** ได้ทันที โดยไม่
ต้อง deploy ใหม่เลย

### 752.2 ติดตั้งและตั้งค่า

```bash
pip install django-celery-beat
pip freeze | grep celery-beat >> requirements.txt
```

```python
# config/settings.py
INSTALLED_APPS = [
    ...
    'django_celery_beat',
]

CELERY_BEAT_SCHEDULER = 'django_celery_beat.schedulers:DatabaseScheduler'
```

```bash
python manage.py migrate django_celery_beat
```

Migration นี้จะสร้างตารางสำหรับเก็บ schedule 5 ตาราง:

| ตาราง | ใช้เก็บ |
|---|---|
| `PeriodicTask` | รายการ task ทั้งหมดที่ต้องรันตามเวลา (ชื่อ task, argument, เปิด/ปิด) |
| `IntervalSchedule` | schedule แบบ "ทุก N วินาที/นาที/ชั่วโมง/วัน" |
| `CrontabSchedule` | schedule แบบ cron (นาที, ชั่วโมง, วันในสัปดาห์ ฯลฯ) |
| `ClockedSchedule` | schedule แบบ "รันครั้งเดียวที่เวลานี้เวลาเดียว" (one-off) |
| `SolarSchedule` | schedule ตามตำแหน่งดวงอาทิตย์ (พระอาทิตย์ขึ้น/ตก ตามพิกัด GPS — ใช้น้อยมากในเว็บทั่วไป) |

รันคำสั่งเดิมเหมือนขั้นตอนที่ 751.4 ได้เลย (`celery -A config beat`) — ตอนนี้มันจะอ่าน
schedule จากฐานข้อมูลแทนไฟล์ `settings.py` โดยอัตโนมัติ

### 752.3 สร้าง Periodic Task ผ่าน Django Admin

`django-celery-beat` ลงทะเบียน `ModelAdmin` ให้อัตโนมัติเมื่อเพิ่มใน `INSTALLED_APPS`
เข้า `/admin/django_celery_beat/periodictask/` แล้วกด "Add periodic task":

1. **Name**: ชื่อสำหรับแสดงผล (ไม่ใช่ชื่อ task จริง) เช่น "ส่งสรุปสถิติบล็อกรายสัปดาห์"
2. **Task (registered)**: เลือกจาก dropdown ที่ Celery รู้จัก autodiscover มาให้แล้ว
   เช่น `blog.tasks.send_weekly_digest_email`
3. **Interval Schedule** หรือ **Crontab Schedule**: เลือกอย่างใดอย่างหนึ่ง (สร้างใหม่ได้
   จากหน้าเดียวกัน หรือเลือกจากที่มีอยู่แล้วมาใช้ซ้ำ)
4. **Enabled**: ติ๊กเพื่อเปิดใช้งาน (ปิดโดยไม่ต้องลบก็ได้ — สะดวกมากเวลา debug)
5. **Arguments (JSON)**: argument แบบ positional ที่ส่งให้ task เช่น `[1, "weekly"]`
6. **Keyword Arguments (JSON)**: argument แบบ keyword เช่น `{"category_id": 3}`

### 752.4 สร้าง Periodic Task ผ่านโค้ด (แนะนำสำหรับค่าเริ่มต้นของระบบ)

แม้จุดเด่นคือแก้ผ่าน Admin ได้ แต่ **ค่าเริ่มต้นของระบบ** (default schedule ที่ควรมีอยู่
ตั้งแต่ deploy ครั้งแรก) ควรสร้างผ่านโค้ดใน **data migration** เพื่อให้ทุก environment
(dev, staging, production) มี schedule เริ่มต้นเหมือนกันแน่นอน โดยทีมยังสามารถเข้าไปปรับ
เวลาที่แน่นอนผ่าน Admin ได้อีกทีภายหลัง:

```python
# blog/migrations/0009_create_default_periodic_tasks.py
import json

from django.db import migrations


def create_periodic_tasks(apps, schema_editor):
    IntervalSchedule = apps.get_model('django_celery_beat', 'IntervalSchedule')
    CrontabSchedule = apps.get_model('django_celery_beat', 'CrontabSchedule')
    PeriodicTask = apps.get_model('django_celery_beat', 'PeriodicTask')

    every_5_minutes, _ = IntervalSchedule.objects.get_or_create(
        every=5, period='minutes',
    )
    PeriodicTask.objects.get_or_create(
        name='sync-post-view-stats-every-5-minutes',
        defaults={
            'task': 'blog.tasks.sync_post_view_stats',
            'interval': every_5_minutes,
            'enabled': True,
        },
    )

    monday_7am, _ = CrontabSchedule.objects.get_or_create(
        minute='0', hour='7', day_of_week='1',
        day_of_month='*', month_of_year='*',
    )
    PeriodicTask.objects.get_or_create(
        name='send-weekly-digest-every-monday-7am',
        defaults={
            'task': 'blog.tasks.send_weekly_digest_email',
            'crontab': monday_7am,
            'enabled': True,
            'kwargs': json.dumps({'category_id': None}),
        },
    )


def remove_periodic_tasks(apps, schema_editor):
    PeriodicTask = apps.get_model('django_celery_beat', 'PeriodicTask')
    PeriodicTask.objects.filter(
        name__in=[
            'sync-post-view-stats-every-5-minutes',
            'send-weekly-digest-every-monday-7am',
        ]
    ).delete()


class Migration(migrations.Migration):
    dependencies = [
        ('blog', '0008_previous_migration'),
        ('django_celery_beat', '0018_improve_crontab_helptext'),
    ]

    operations = [
        migrations.RunPython(create_periodic_tasks, remove_periodic_tasks),
    ]
```

> **ทำไมใช้ `get_or_create` แทน `create`**: migration นี้อาจถูกรันซ้ำได้ในบาง workflow
> (เช่น ทดสอบ migration แบบ fake หรือ rebuild ฐานข้อมูล test) การใช้ `get_or_create`
> ทำให้ migration idempotent — รันกี่ครั้งก็ได้ผลลัพธ์เหมือนเดิม ไม่สร้างข้อมูลซ้ำ (หลักการ
> เดียวกับที่จะเจาะลึกเรื่อง idempotency ของ task เองในขั้นตอนที่ 758)

### 752.5 One-off Task ด้วย `ClockedSchedule`

บางครั้งต้องการรัน task **ครั้งเดียวที่เวลาที่แน่นอนในอนาคต** ไม่ใช่ซ้ำ ๆ ตามรอบ เช่น
"ส่งอีเมลแจ้งเตือนโปรโมชั่นวันที่ 1 มกราคม เวลา 09:00 น. ครั้งเดียว":

```python
# blog/services/schedule_oneoff.py
from django.utils import timezone
from django_celery_beat.models import ClockedSchedule, PeriodicTask
import json


def schedule_promotion_email(run_at, promotion_id):
    """สร้าง one-off task ที่รันครั้งเดียวตามเวลาที่กำหนด แล้วปิดตัวเองอัตโนมัติ"""
    clocked, _ = ClockedSchedule.objects.get_or_create(clocked_time=run_at)

    PeriodicTask.objects.create(
        name=f'send-promotion-email-{promotion_id}-{run_at.isoformat()}',
        task='blog.tasks.send_promotion_email',
        clocked=clocked,
        one_off=True,   # สำคัญมาก: ปิดตัวเอง (enabled=False) หลังรันสำเร็จ 1 ครั้ง
        enabled=True,
        kwargs=json.dumps({'promotion_id': promotion_id}),
    )
```

```python
>>> from django.utils import timezone
>>> from datetime import timedelta
>>> from blog.services.schedule_oneoff import schedule_promotion_email
>>> schedule_promotion_email(timezone.now() + timedelta(days=1), promotion_id=42)
```

`one_off=True` คือ flag สำคัญ — ถ้าไม่ตั้งค่านี้ Celery Beat จะพยายามรัน task ซ้ำทุกครั้ง
ที่เช็ค schedule หลังจากเวลานั้นผ่านไปแล้ว (เพราะ `ClockedSchedule` ที่ตรงกับเวลาปัจจุบัน
"ยัง due อยู่เรื่อย ๆ" ถ้าไม่มีกลไกปิดตัวเอง)

### 752.6 DatabaseScheduler ตรวจจับการเปลี่ยนแปลงอย่างไร (ไม่ต้อง restart Beat)

คำถามที่พบบ่อยที่สุด: "ถ้าแก้ schedule ผ่าน Admin ต้อง restart กระบวนการ `celery beat`
ไหม?" **คำตอบคือไม่ต้อง** `DatabaseScheduler` จะ poll ฐานข้อมูลเป็นระยะ (ค่า default
ทุก 5 วินาที ควบคุมด้วย `beat_max_loop_interval`) เพื่อเช็คว่ามีการเปลี่ยนแปลงใน
`PeriodicTask` table หรือไม่ (เช็คผ่าน timestamp `last_update`) ถ้ามีการเปลี่ยนแปลง
มันจะโหลด schedule ใหม่เข้าหน่วยความจำทันทีโดยไม่ต้อง restart process เลย — นี่คือ
จุดขายหลักของ `django-celery-beat` เมื่อเทียบกับ schedule แบบ hardcode ในขั้นตอนที่ 751
ที่ต้อง restart Beat ทุกครั้งที่แก้ไข

### 752.7 ตารางสรุปขั้นตอนที่ 752

| หัวข้อ | สรุป |
|---|---|
| ปัญหาที่แก้ | เปลี่ยน schedule โดยไม่ต้อง deploy ใหม่ |
| ติดตั้ง | `pip install django-celery-beat` + เพิ่ม app + `CELERY_BEAT_SCHEDULER` + migrate |
| จัดการผ่าน | Django Admin (`PeriodicTask`, `IntervalSchedule`, `CrontabSchedule`) |
| ค่าเริ่มต้นของระบบ | สร้างผ่าน data migration ด้วย `get_or_create` (idempotent) |
| รันครั้งเดียว | `ClockedSchedule` + `one_off=True` |
| ต้อง restart Beat ไหมเมื่อแก้ schedule | ไม่ต้อง — `DatabaseScheduler` poll DB อัตโนมัติทุก ~5 วินาที |

---

## ขั้นตอนที่ 753: Task Chain — รัน Task ต่อกันเป็นลำดับ

### 753.1 ปัญหา: งานที่ต้องทำตามลำดับ โดยผลลัพธ์ของขั้นก่อนหน้าป้อนเข้าขั้นถัดไป

สมมติว่าต้องการสร้างรายงาน PDF สรุปสถิติบล็อกแล้วส่งอีเมล กระบวนการมี 3 ขั้นตอนที่
**ต้องทำตามลำดับ** เพราะขั้นถัดไปต้องใช้ผลลัพธ์จากขั้นก่อนหน้า:

```
1. รวบรวมข้อมูลสถิติจากฐานข้อมูล (gather_report_data)
        │ ผลลัพธ์ = dict ข้อมูลดิบ
        ▼
2. สร้างไฟล์ PDF จากข้อมูลนั้น (render_report_pdf)
        │ ผลลัพธ์ = path ของไฟล์ PDF ที่สร้างเสร็จ
        ▼
3. ส่งอีเมลแนบไฟล์ PDF (email_report_to_admin)
```

วิธีที่ **ผิด** ที่มือใหม่มักทำ คือเรียก `.delay()` ทีละตัวแล้วรอผลด้วย `.get()` ใน
โค้ด Python ปกติ (ซึ่งจะคุยรายละเอียดปัญหาเต็ม ๆ ในขั้นตอนที่ 754.3):

```python
# อย่าทำแบบนี้! .get() บล็อกจนกว่า task จะเสร็จ เสีย concurrency ของ Celery ไปเปล่า ๆ
data_result = gather_report_data.delay()
data = data_result.get()   # บล็อกรอ

pdf_result = render_report_pdf.delay(data)
pdf_path = pdf_result.get()   # บล็อกรอ

email_report_to_admin.delay(pdf_path)
```

วิธีที่ถูกต้องคือใช้ **`chain()`** ของ Celery ซึ่งให้ Celery เป็นผู้จัดการลำดับการรันเอง
โดยไม่ต้องมี process ไหนนั่งบล็อกรอเลย

### 753.2 Signature: หน่วยพื้นฐานของ Chain/Group/Chord

ก่อนใช้ `chain()` ต้องรู้จัก **Signature** ก่อน — Signature คือการ "ห่อ" task พร้อม
argument ไว้ล่วงหน้า โดยยังไม่รันทันที (ต่างจาก `.delay()` ที่รันทันที):

```python
from celery import signature

# 3 วิธีสร้าง signature ที่เทียบเท่ากัน
sig1 = gather_report_data.s(category_id=3)
sig2 = gather_report_data.signature(kwargs={'category_id': 3})
sig3 = signature('blog.tasks.gather_report_data', kwargs={'category_id': 3})
```

`.s()` คือ shorthand ของ `.signature()` ที่ใช้บ่อยที่สุด มีตัวแปรสำคัญอีกตัวคือ `.si()`
(immutable signature) ซึ่งจะอธิบายในหัวข้อถัดไป

### 753.3 ตัวอย่างจริง: Chain สร้างและส่งรายงาน PDF

```python
# blog/tasks.py
import io
import logging

from celery import shared_task, chain
from django.core.mail import EmailMessage
from django.utils import timezone

from .models import Post, Category

logger = logging.getLogger(__name__)


@shared_task(name='blog.tasks.gather_report_data')
def gather_report_data(category_id=None):
    """ขั้นที่ 1: รวบรวมข้อมูลดิบจากฐานข้อมูล คืนค่าเป็น dict ที่ serialize เป็น JSON ได้"""
    qs = Post.objects.filter(is_published=True)
    if category_id:
        qs = qs.filter(category_id=category_id)

    return {
        'total_posts': qs.count(),
        'total_views': sum(qs.values_list('views_count', flat=True)),
        'top_posts': list(
            qs.order_by('-views_count').values('title', 'views_count')[:5]
        ),
        'generated_at': timezone.now().isoformat(),
    }


@shared_task(name='blog.tasks.render_report_pdf')
def render_report_pdf(report_data):
    """
    ขั้นที่ 2: รับผลลัพธ์จาก gather_report_data มาสร้าง PDF
    Celery ส่งผลลัพธ์ของ task ก่อนหน้าเป็น argument ตัวแรกให้อัตโนมัติเมื่อใช้ chain()
    """
    from reportlab.lib.pagesizes import A4
    from reportlab.pdfgen import canvas

    buffer = io.BytesIO()
    pdf = canvas.Canvas(buffer, pagesize=A4)
    pdf.drawString(50, 800, f"รายงานสถิติบล็อก — สร้างเมื่อ {report_data['generated_at']}")
    pdf.drawString(50, 780, f"จำนวนโพสต์ทั้งหมด: {report_data['total_posts']}")
    pdf.drawString(50, 760, f"ยอด view รวม: {report_data['total_views']}")

    y = 720
    for post in report_data['top_posts']:
        pdf.drawString(50, y, f"- {post['title']} ({post['views_count']} views)")
        y -= 20

    pdf.save()
    buffer.seek(0)

    from django.core.files.storage import default_storage
    filename = f"reports/weekly_report_{timezone.now():%Y%m%d_%H%M%S}.pdf"
    saved_path = default_storage.save(filename, buffer)
    return saved_path


@shared_task(name='blog.tasks.email_report_to_admin')
def email_report_to_admin(pdf_path, recipient='admin@example.com'):
    """ขั้นที่ 3: รับ path ของ PDF จากขั้นก่อนหน้า มาแนบไฟล์แล้วส่งอีเมล"""
    from django.core.files.storage import default_storage

    email = EmailMessage(
        subject='รายงานสถิติบล็อกประจำสัปดาห์',
        body='กรุณาดูรายงานที่แนบมาด้วยไฟล์นี้',
        to=[recipient],
    )
    with default_storage.open(pdf_path, 'rb') as f:
        email.attach('weekly_report.pdf', f.read(), 'application/pdf')
    email.send()

    logger.info('ส่งรายงาน PDF ไปที่ %s สำเร็จ (%s)', recipient, pdf_path)
    return {'sent_to': recipient, 'pdf_path': pdf_path}
```

เชื่อม 3 task เข้าด้วยกันด้วย `chain()`:

```python
# blog/services/reports.py
from celery import chain

from blog.tasks import gather_report_data, render_report_pdf, email_report_to_admin


def trigger_weekly_report_pipeline(category_id=None, recipient='admin@example.com'):
    """
    สร้าง chain: gather → render → email
    ผลลัพธ์ของแต่ละ task จะถูกส่งเป็น argument แรกให้ task ถัดไปโดยอัตโนมัติ
    """
    pipeline = chain(
        gather_report_data.s(category_id=category_id),
        render_report_pdf.s(),
        email_report_to_admin.s(recipient=recipient),
    )
    async_result = pipeline.apply_async()
    return async_result.id
```

หรือเขียนด้วย operator `|` (pipe) ซึ่งอ่านง่ายและเป็นที่นิยมมากกว่าในโค้ดจริง —
เทียบเท่ากับการเรียก `chain()` ตรง ๆ ทุกประการ:

```python
def trigger_weekly_report_pipeline_v2(category_id=None, recipient='admin@example.com'):
    pipeline = (
        gather_report_data.s(category_id=category_id)
        | render_report_pdf.s()
        | email_report_to_admin.s(recipient=recipient)
    )
    return pipeline.apply_async().id
```

**สิ่งที่เกิดขึ้นเบื้องหลัง**: Worker หยิบ `gather_report_data` มารันก่อน เมื่อเสร็จแล้ว
Celery จะเอาผลลัพธ์ (dict) ไปต่อท้าย argument ของ `render_report_pdf` โดยอัตโนมัติแล้ว
ยิงเข้าคิวต่อทันที (ไม่ต้องมีใครนั่งรอ) เมื่อ `render_report_pdf` เสร็จ ผลลัพธ์ (path
ของ PDF) จะถูกส่งต่อให้ `email_report_to_admin` ในลักษณะเดียวกัน — **ทั้งกระบวนการ
ไม่มี process ไหนต้องบล็อกรอเลยแม้แต่วินาทีเดียว**

### 753.4 `.s()` vs `.si()`: Mutable กับ Immutable Signature

ปกติ Celery จะ "เติม" ผลลัพธ์ของ task ก่อนหน้าเป็น argument ตัวแรกให้ task ถัดไปเสมอ
(เรียกว่า mutable signature — พฤติกรรมปกติของ `.s()`) แต่บางครั้งเราไม่ต้องการแบบนั้น
เช่น task ที่ไม่ได้สนใจผลลัพธ์ก่อนหน้าเลย (แค่ต้องการรันต่อท้ายตามลำดับเฉย ๆ):

```python
from celery import shared_task


@shared_task
def log_pipeline_step(step_name):
    logger.info('Pipeline step เสร็จแล้ว: %s', step_name)


# ถ้าใช้ .s() ธรรมดา — ผลลัพธ์จาก render_report_pdf (path ของ PDF) จะถูกยัดเป็น
# argument ตัวแรกของ log_pipeline_step โดยไม่ตั้งใจ ทำให้ step_name กลายเป็น path แทน
pipeline_wrong = render_report_pdf.s() | log_pipeline_step.s('pdf_rendered')  # ผิด!

# ใช้ .si() (immutable) เพื่อ "ล็อก" argument ไว้ตายตัว ไม่รับผลลัพธ์จาก task ก่อนหน้ามาแทรก
pipeline_correct = render_report_pdf.s() | log_pipeline_step.si('pdf_rendered')  # ถูกต้อง
```

| Signature | ผลลัพธ์จาก task ก่อนหน้า | ใช้เมื่อ |
|---|---|---|
| `.s()` (mutable) | ถูกส่งเป็น argument แรกให้ task ถัดไปอัตโนมัติ | ต้องการส่งต่อผลลัพธ์ (กรณีปกติส่วนใหญ่) |
| `.si()` (immutable) | ถูกทิ้งไป ไม่ส่งต่อ ใช้ argument ที่กำหนดไว้ตายตัวเท่านั้น | task ไม่สนใจผลลัพธ์ก่อนหน้า เช่น ส่ง log/notification ธรรมดา |

### 753.5 การจัดการ Error ใน Chain ด้วย `link_error`

ถ้า task ใดใน chain ล้มเหลว **task ที่เหลือใน chain จะไม่ถูกรันต่อทันที** (chain หยุด
กลางทาง) เพื่อดักจับความล้มเหลวนี้และแจ้งเตือน ใช้ `link_error()`:

```python
@shared_task(name='blog.tasks.notify_pipeline_failure')
def notify_pipeline_failure(request, exc, traceback):
    """
    Celery เรียก error callback พร้อม argument พิเศษ 3 ตัวเสมอ:
    request (ข้อมูล task ที่ fail), exc (exception object), traceback (string)
    """
    logger.error(
        'Pipeline สร้างรายงานล้มเหลว: task=%s exc=%s',
        request.task, exc,
    )
    # ส่งแจ้งเตือนไปยัง Slack/อีเมลของทีม dev ในสถานการณ์จริง


def trigger_weekly_report_pipeline_with_error_handling(category_id=None):
    pipeline = (
        gather_report_data.s(category_id=category_id)
        | render_report_pdf.s()
        | email_report_to_admin.s()
    )
    pipeline.apply_async(link_error=notify_pipeline_failure.s())
```

### 753.6 ตารางสรุปขั้นตอนที่ 753

| หัวข้อ | สรุป |
|---|---|
| Chain คืออะไร | รัน task ต่อกันเป็นลำดับ โดย Celery ส่งผลลัพธ์ของ task ก่อนหน้าให้ task ถัดไปอัตโนมัติ |
| สร้างด้วย | `chain(a.s(), b.s(), c.s())` หรือเขียนสั้นด้วย `a.s() \| b.s() \| c.s()` |
| `.s()` vs `.si()` | mutable (รับผลลัพธ์ก่อนหน้า) vs immutable (ไม่รับ ใช้ argument ตายตัว) |
| จัดการ error | `.apply_async(link_error=callback.s())` — chain หยุดถ้า task ใดล้มเหลว |
| ข้อดีเทียบกับเรียก `.delay()` + `.get()` เอง | ไม่มี process ไหนบล็อกรอผลลัพธ์เลย ทุกอย่างทำงานผ่านคิวล้วน ๆ |

---

## ขั้นตอนที่ 754: Task Group — รันหลาย Task พร้อมกันแบบขนาน

### 754.1 ปัญหา: งานที่เป็นอิสระจากกัน ไม่จำเป็นต้องรอทีละตัว

ต่างจาก Chain ที่งานต้องรัน**ตามลำดับ**เพราะขึ้นต่อกัน บางครั้งเรามีงานหลายชิ้นที่
**เป็นอิสระจากกันโดยสิ้นเชิง** เช่น ต้องการดึงสถิติของ 5 หมวดหมู่บล็อกพร้อมกัน แต่ละ
หมวดหมู่ไม่เกี่ยวข้องกัน ถ้ารันทีละตัวตามลำดับ (เหมือน chain) จะเสียเวลารวมเป็นผลรวมของ
ทุกตัว แต่ถ้ารัน **พร้อมกัน** เวลารวมจะเท่ากับตัวที่ช้าที่สุดเพียงตัวเดียว

```
รันทีละตัว (sequential): 3s + 3s + 3s + 3s + 3s = 15 วินาที
รันพร้อมกัน (parallel):   max(3s, 3s, 3s, 3s, 3s) = 3 วินาที (ถ้ามี worker ว่างพอ)
```

**`group()`** คือเครื่องมือของ Celery สำหรับสถานการณ์นี้

### 754.2 ตัวอย่างจริง: ดึงสถิติหลายหมวดหมู่พร้อมกัน

```python
# blog/tasks.py
@shared_task(name='blog.tasks.gather_category_stats')
def gather_category_stats(category_id):
    """คำนวณสถิติของหมวดหมู่เดียว — เป็นอิสระจากหมวดหมู่อื่นโดยสิ้นเชิง"""
    category = Category.objects.get(pk=category_id)
    posts = Post.objects.filter(category=category, is_published=True)

    return {
        'category_id': category_id,
        'category_name': category.name,
        'post_count': posts.count(),
        'total_views': sum(posts.values_list('views_count', flat=True)),
    }
```

```python
# blog/services/reports.py
from celery import group

from blog.tasks import gather_category_stats


def gather_all_category_stats_parallel():
    """
    รัน gather_category_stats สำหรับทุกหมวดหมู่พร้อมกัน แทนที่จะวน loop เรียกทีละตัว
    เหมาะมากเมื่อมีหมวดหมู่จำนวนมาก และแต่ละงานใช้เวลา query/คำนวณพอสมควร
    """
    category_ids = Category.objects.values_list('id', flat=True)

    job = group(gather_category_stats.s(cid) for cid in category_ids)
    group_result = job.apply_async()
    return group_result
```

### 754.3 การรอผลลัพธ์: `.get()` และทำไมต้องระวังการใช้ในเว็บ request

`GroupResult` ที่ได้กลับมามีเมธอด `.get()` ให้รอผลลัพธ์ทั้งหมด:

```python
>>> group_result = gather_all_category_stats_parallel()
>>> group_result.ready()      # เช็คว่าทุก task ใน group เสร็จหรือยัง (ไม่บล็อก)
False
>>> results = group_result.get(timeout=30)   # รอจนเสร็จทั้งหมด หรือ timeout 30 วินาที
>>> results
[{'category_id': 1, 'category_name': 'Python', ...}, {'category_id': 2, ...}, ...]
```

**คำเตือนสำคัญ**: **ห้ามเรียก `.get()` ภายใน Django view หรือภายใน Celery task อื่น
โดยไม่ระวัง** เพราะ:

1. **ใน Django view**: `.get()` บล็อก request-response cycle ทำให้ผู้ใช้ต้องรอจนกว่า
   task ทั้งหมดจะเสร็จ — ขัดกับเจตนารมณ์ของ Celery ที่ต้องการให้ view ตอบกลับเร็ว
   แล้วให้งานหนักไปทำเบื้องหลัง ถ้าต้องรอผลจริง ๆ ควรใช้ AJAX polling หรือ WebSocket
   (Part 057/078) แจ้งผลลัพธ์กลับไปแทน ไม่ใช่บล็อก request ตรง ๆ
2. **ใน Celery task อื่น**: ถ้า task A เรียก `.get()` รอผลของ group ที่ยิงจากภายใน task A
   เอง และ worker pool มี concurrency จำกัด (เช่น worker เดียว, concurrency=1) จะเกิด
   **deadlock** ทันที เพราะ worker ตัวเดียวกันไปนั่งรอผลของงานที่ต้องใช้ worker (ตัวเดียว
   กันนั้นแหละ) มารันให้ ทำให้ทุกอย่างค้างตลอดกาล เอกสารทางการของ Celery เตือนเรื่องนี้
   อย่างชัดเจนว่า **"never call result.get() synchronously in a task"**

วิธีที่ถูกต้องเมื่อต้องการ "รอผลลัพธ์ของ group แล้วทำอะไรต่อ" คือใช้ **Chord** ซึ่งเป็น
หัวข้อของขั้นตอนที่ 755 — Chord ทำสิ่งนี้ได้โดยไม่ต้อง block เลย

### 754.4 รวม Chain กับ Group: รัน Group แล้วต่อด้วยขั้นตอนถัดไป (ที่ยังไม่ใช่ callback รวมผล)

Chain และ Group รวมกันได้ เช่น รัน group ของการเตรียมข้อมูลหลายส่วนพร้อมกัน แล้วค่อยส่ง
"สัญญาณเสร็จแล้ว" ต่อไปทำอย่างอื่นที่ไม่ต้องพึ่งผลลัพธ์ตรง ๆ (ถ้าต้อง**รวมผลลัพธ์**เข้า
task เดียว ต้องใช้ Chord ในขั้นตอนที่ 755 ไม่ใช่แบบนี้):

```python
from celery import chain, group

pipeline = chain(
    group(
        gather_category_stats.s(1),
        gather_category_stats.s(2),
        gather_category_stats.s(3),
    ),
    log_pipeline_step.si('all_category_stats_gathered'),   # ใช้ .si() เพราะไม่ต้องรับผล list
)
pipeline.apply_async()
```

### 754.5 ตารางสรุปขั้นตอนที่ 754

| หัวข้อ | สรุป |
|---|---|
| Group คืออะไร | รันหลาย task พร้อมกันแบบขนาน โดยไม่มีลำดับก่อนหลัง (ต่างจาก Chain) |
| สร้างด้วย | `group(task.s(x) for x in items)` |
| รอผลลัพธ์ | `group_result.get(timeout=...)` — **ห้ามใช้ในเว็บ view หรือใน task อื่นแบบไม่ระวัง (เสี่ยง deadlock)** |
| ถ้าต้องรวมผลลัพธ์เข้า callback | ใช้ **Chord** (ขั้นตอนที่ 755) แทนการ `.get()` เอง |
| รวมกับ Chain ได้ | ใช้ `.si()` สำหรับขั้นตอนที่ตามหลัง group แต่ไม่ต้องรับผลลัพธ์แบบ list |

---

## ขั้นตอนที่ 755: Chord — รูปแบบ Group + Callback

### 755.1 Chord คืออะไร

**Chord** คือรูปแบบผสมระหว่าง Group กับ Callback: **รันหลาย task พร้อมกันแบบ Group
ก่อน แล้วเมื่อ "ทุกตัว" ใน group เสร็จสมบูรณ์ ให้รวมผลลัพธ์ทั้งหมดเป็น list ส่งเข้า
callback task ตัวเดียว** — โดยที่ **ไม่มี process ไหนต้อง `.get()` บล็อกรอเลย**
(แก้ปัญหาที่พูดถึงในขั้นตอนที่ 754.3 ได้อย่างสมบูรณ์)

```
                    ┌─> gather_category_stats(1) ─┐
                    │                              │
รัน group พร้อมกัน ──┼─> gather_category_stats(2) ─┼──> รอทุกตัวเสร็จ ──> combine_stats(results)
                    │                              │      (callback)
                    └─> gather_category_stats(3) ─┘
```

### 755.2 ตัวอย่างจริง: รวมสถิติทุกหมวดหมู่เป็นรายงานเดียว

```python
# blog/tasks.py
@shared_task(name='blog.tasks.combine_category_stats')
def combine_category_stats(results):
    """
    Callback ของ chord — รับ list ผลลัพธ์จาก gather_category_stats ทุกตัวที่รันเสร็จแล้ว
    Celery ส่ง list นี้เป็น argument แรกให้อัตโนมัติ เรียงตามลำดับที่ประกาศใน group
    """
    total_posts = sum(r['post_count'] for r in results)
    total_views = sum(r['total_views'] for r in results)

    report = {
        'categories': results,
        'total_posts': total_posts,
        'total_views': total_views,
        'generated_at': timezone.now().isoformat(),
    }

    logger.info(
        'สร้างรายงานรวมสำเร็จ: %d หมวดหมู่, %d โพสต์, %d views รวม',
        len(results), total_posts, total_views,
    )
    return report
```

```python
# blog/services/reports.py
from celery import chord

from blog.tasks import gather_category_stats, combine_category_stats


def build_full_category_report():
    """
    Chord: รัน gather_category_stats ของทุกหมวดหมู่แบบขนาน
    เมื่อครบทุกตัวแล้ว ค่อยรวมผลด้วย combine_category_stats แบบไม่ต้อง .get() รอเลย
    """
    category_ids = list(Category.objects.values_list('id', flat=True))

    header = [gather_category_stats.s(cid) for cid in category_ids]
    callback = combine_category_stats.s()

    result = chord(header)(callback)
    return result.id
```

หรือเขียนด้วย syntax ทางเลือกที่อ่านง่ายกว่าในบางกรณี:

```python
def build_full_category_report_v2():
    category_ids = list(Category.objects.values_list('id', flat=True))
    workflow = chord(
        (gather_category_stats.s(cid) for cid in category_ids),
        combine_category_stats.s(),
    )
    return workflow.apply_async().id
```

### 755.3 ข้อกำหนดสำคัญ: Chord ต้องมี Result Backend

Chord จำเป็นต้องมี **result backend** (ที่เก็บผลลัพธ์ของแต่ละ task) เพื่อให้ Celery
รู้ว่า "task ไหนใน group เสร็จแล้วบ้าง" และรอจนครบก่อนค่อยยิง callback ถ้าไม่ได้ตั้งค่า
`CELERY_RESULT_BACKEND` ไว้ (ทวนจากขั้นตอนที่ 751.1 — เราตั้งเป็น Redis DB 5 ไว้แล้ว)
chord จะ **ไม่ทำงาน** และ Celery จะแจ้ง error ทันทีตอนยิงงาน:

```python
# config/settings.py — ต้องมีบรรทัดนี้เสมอเมื่อจะใช้ chord
CELERY_RESULT_BACKEND = 'redis://127.0.0.1:6379/5'
```

> **หมายเหตุเชิงเทคนิค**: เมื่อใช้ Redis เป็น result backend, Celery ใช้กลไกภายในที่
> เรียกว่า **chord unlock task** ซึ่งจะพยายามเช็คทุก ๆ ช่วงเวลาสั้น ๆ ว่าสมาชิกทุกตัว
> ใน group เสร็จหรือยัง (คล้าย polling แบบเบา ๆ) ต่างจาก backend บางตัว (เช่น RabbitMQ
> ล้วน ๆ) ที่ไม่รองรับ chord ได้ดีเท่า Redis — **นี่คืออีกเหตุผลที่หลักสูตรนี้เลือก Redis
> เป็นทั้ง broker และ result backend ตั้งแต่ Part 075**

### 755.4 การจัดการ Error ใน Chord: `link_error` ระดับ Chord

ถ้า task ใดใน group ของ chord ล้มเหลว โดย default แล้ว **callback จะไม่ถูกเรียกเลย**
และ error จะถูกเก็บไว้ใน result backend ให้ตรวจสอบภายหลัง เพื่อดักจับสถานการณ์นี้อย่าง
ชัดเจน:

```python
@shared_task(name='blog.tasks.notify_chord_failure')
def notify_chord_failure(request, exc, traceback):
    logger.error('Chord ล้มเหลว: task=%s exc=%s', request.task, exc)


def build_full_category_report_safe():
    category_ids = list(Category.objects.values_list('id', flat=True))
    header = [gather_category_stats.s(cid) for cid in category_ids]
    callback = combine_category_stats.s()

    # ผูก error callback เข้ากับทุก task ใน header (แต่ละตัวใน group)
    header_with_error_handling = [
        sig.on_error(notify_chord_failure.s()) for sig in header
    ]

    result = chord(header_with_error_handling)(callback)
    return result.id
```

### 755.5 ตารางเปรียบเทียบ Chain vs Group vs Chord

| รูปแบบ | ลักษณะการทำงาน | ใช้เมื่อ |
|---|---|---|
| **Chain** | A → B → C ตามลำดับ ผลของ A ป้อนเข้า B, ผลของ B ป้อนเข้า C | งานที่แต่ละขั้นขึ้นต่อกัน ต้องรอผลลัพธ์ก่อนหน้าก่อนเริ่มขั้นถัดไป |
| **Group** | A, B, C รันพร้อมกันทั้งหมด ไม่มีใครรอใคร | งานที่เป็นอิสระจากกันโดยสิ้นเชิง ต้องการความเร็วจากการขนาน |
| **Chord** | A, B, C รันพร้อมกัน (เหมือน Group) แล้วรอครบทุกตัวก่อนส่งผลรวมเข้า D (callback) | ต้องการทั้งความเร็วจากการขนาน **และ** ต้องรวมผลลัพธ์ทั้งหมดเข้าเป็นก้อนเดียวในขั้นตอนถัดไป |

### 755.6 ตารางสรุปขั้นตอนที่ 755

| หัวข้อ | สรุป |
|---|---|
| Chord คืออะไร | Group + Callback ที่รอทุก task เสร็จก่อนรวมผลเข้า task เดียว โดยไม่ block |
| สร้างด้วย | `chord(header)(callback)` หรือ `chord(header, callback).apply_async()` |
| ข้อกำหนด | ต้องมี `CELERY_RESULT_BACKEND` ตั้งค่าไว้เสมอ (แนะนำ Redis) |
| Callback รับอะไร | `list` ของผลลัพธ์จากทุก task ใน header ตามลำดับที่ประกาศ |
| Error handling | ใช้ `.on_error()` กับแต่ละ signature ใน header เพื่อดักจับความล้มเหลวเฉพาะจุด |

---

## ขั้นตอนที่ 756: Task Routing — ส่ง Task ไปคนละ Queue ตามความสำคัญ

### 756.1 ปัญหา: Task สำคัญติดคิวรอ Task ที่ไม่สำคัญ

โดย default แล้ว **ทุก task ที่ยิงเข้า Celery จะไปอยู่ในคิวเดียวกันหมด** (คิวชื่อ
`celery` เป็นค่าเริ่มต้น) worker จะหยิบงานตามลำดับ FIFO (先入先出) โดยประมาณ นี่คือปัญหา
ใหญ่เมื่อระบบมีทั้ง **task เร่งด่วน** (เช่น ส่งอีเมลรีเซ็ตรหัสผ่าน — ผู้ใช้รอผลอยู่หน้าจอ)
และ **task ที่ไม่เร่งด่วน** (เช่น สร้างรายงานสถิติรายสัปดาห์ ใช้เวลาหลายนาที) ปนกันอยู่
ในคิวเดียว:

```
สถานการณ์ปัญหา: คิวเดียวรวมทุกอย่าง
[weekly_report] [weekly_report] [password_reset_email] [weekly_report] ...
                                        ↑
                          ผู้ใช้ที่รอรีเซ็ตรหัสผ่านต้องรอ
                          จนกว่า weekly_report 2 งานก่อนหน้าจะทำเสร็จก่อน!
```

ทางแก้คือ **แยกคิว (queue) ตามความสำคัญ** แล้วให้ worker แต่ละกลุ่มรับผิดชอบคนละคิว

### 756.2 กำหนด Queue ให้ Task ด้วย `CELERY_TASK_ROUTES`

```python
# config/settings.py
CELERY_TASK_ROUTES = {
    'accounts.tasks.send_password_reset_email': {'queue': 'high_priority'},
    'accounts.tasks.send_login_otp': {'queue': 'high_priority'},
    'blog.tasks.sync_post_view_stats': {'queue': 'default'},
    'blog.tasks.gather_report_data': {'queue': 'low_priority'},
    'blog.tasks.render_report_pdf': {'queue': 'low_priority'},
    'blog.tasks.email_report_to_admin': {'queue': 'low_priority'},
}

# ประกาศคิวทั้งหมดที่มีในระบบให้ Celery รู้จักล่วงหน้า (ไม่บังคับแต่แนะนำให้ทำเสมอ)
CELERY_TASK_QUEUES = {
    'high_priority': {'exchange': 'high_priority', 'routing_key': 'high_priority'},
    'default': {'exchange': 'default', 'routing_key': 'default'},
    'low_priority': {'exchange': 'low_priority', 'routing_key': 'low_priority'},
}
CELERY_TASK_DEFAULT_QUEUE = 'default'
```

หรือกำหนด queue ตรงที่ตัว decorator ของ task เอง (สะดวกกว่าเมื่อ routing ไม่ซับซ้อนมาก
และอยากเห็น queue ของ task นั้นชัดเจนในที่เดียวกับตัวโค้ด):

```python
# accounts/tasks.py
from celery import shared_task


@shared_task(name='accounts.tasks.send_password_reset_email', queue='high_priority')
def send_password_reset_email(user_id):
    ...
```

### 756.3 รัน Worker แยกตาม Queue

จุดสำคัญของการแยกคิวคือต้อง**รัน worker แยกกลุ่มตามคิว** ไม่เช่นนั้นการแยกคิวจะไม่มี
ประโยชน์อะไรเลย (worker กลุ่มเดียวก็ยังหยิบงานทุกคิวปนกันอยู่ดี):

```bash
# Worker กลุ่มที่ 1: รับผิดชอบเฉพาะงานเร่งด่วน ใช้ concurrency สูง ตอบสนองไว
celery -A config worker -Q high_priority --concurrency=8 --hostname=worker-urgent@%h

# Worker กลุ่มที่ 2: รับผิดชอบงานทั่วไป
celery -A config worker -Q default --concurrency=4 --hostname=worker-default@%h

# Worker กลุ่มที่ 3: รับผิดชอบงานหนักที่ไม่เร่งด่วน ใช้ concurrency ต่ำกว่าเพื่อไม่แย่ง CPU
celery -A config worker -Q low_priority --concurrency=2 --hostname=worker-batch@%h
```

ด้วยการตั้งค่านี้ แม้ `low_priority` queue จะมีงาน 100 รายการค้างอยู่ งาน
`high_priority` ที่เพิ่งเข้ามาจะถูกหยิบไปทำโดย `worker-urgent` ทันที ไม่ต้องรอคิวอื่น
เลยแม้แต่วินาทีเดียว เพราะเป็นคนละ worker กันโดยสิ้นเชิง

### 756.4 เรื่อง "Priority Queue" ใน Redis: ข้อจำกัดที่ต้องรู้

หลายคนเข้าใจผิดว่า Celery มี "priority ระดับ message" ที่ทำให้ message ความสำคัญสูง
แซงคิวไปก่อนได้ในคิวเดียวกัน (คล้าย priority queue ใน data structure) — ความจริงคือ
**Redis broker รองรับ priority ได้แบบจำกัดมาก** ต่างจาก RabbitMQ ที่รองรับเต็มรูปแบบ
กว่า (จะเจาะลึก RabbitMQ ใน Part 077):

| Broker | รองรับ Message Priority ในคิวเดียว | คำแนะนำ |
|---|---|---|
| **Redis** | ⚠️ รองรับแบบจำกัด (ผ่าน `priority` field แต่ไม่แม่นยำเท่า RabbitMQ และมีข้อจำกัดเรื่องจำนวนระดับ priority) | **แนะนำให้แยกเป็นคนละคิวไปเลยแทนการพึ่ง priority ในคิวเดียว** เหมือนที่ทำในขั้นตอนที่ 756.2-756.3 |
| **RabbitMQ** | ✅ รองรับเต็มรูปแบบผ่าน `x-max-priority` ของ AMQP โดยตรง | ใช้ priority field ได้จริงถ้าจำเป็นต้องมี priority ละเอียดกว่าแค่ 2-3 ระดับ |

**คำแนะนำระดับมืออาชีพของหลักสูตรนี้**: สำหรับ 90% ของระบบทั่วไป **การแยกคิวตาม
ความสำคัญ (เหมือนขั้นตอนที่ 756.2-756.3) เพียงพอและเข้าใจง่ายกว่ามาก** เมื่อเทียบกับ
การพยายามตั้งค่า priority ละเอียดในคิวเดียว ควรใช้ priority field ของ RabbitMQ ก็ต่อเมื่อ
มีความต้องการที่ซับซ้อนจริง ๆ เช่น "priority มี 10 ระดับ และเปลี่ยนแปลงตาม business
logic แบบไดนามิก" ซึ่งพบได้น้อยกว่าการแยกคิวแบบตายตัวมาก

### 756.5 ตารางสรุปขั้นตอนที่ 756

| หัวข้อ | สรุป |
|---|---|
| ปัญหาที่แก้ | Task เร่งด่วนติดรอ task ที่ไม่เร่งด่วนในคิวเดียวกัน |
| กำหนด queue | `CELERY_TASK_ROUTES` ใน settings หรือ `queue=` ที่ decorator |
| รัน worker แยกคิว | `celery -A config worker -Q <queue_name>` — ต้องรันแยกกลุ่มจริง ๆ ไม่ใช่แค่ตั้งค่า route |
| Priority ใน Redis | รองรับจำกัด — แนะนำแยกคิวแทนพึ่ง priority field ในคิวเดียว |
| Priority เต็มรูปแบบ | ต้องใช้ RabbitMQ (Part 077) |

---

## ขั้นตอนที่ 757: Time Limit ของ Task — ป้องกัน Task ค้าง

### 757.1 ปัญหา: Task ที่ค้างตลอดกาล

Task ที่เรียก external API (เช่น payment gateway, third-party service) มีความเสี่ยงที่
"ค้าง" (hang) ได้เสมอ ถ้า API นั้นไม่ตอบกลับเลยและไม่มี timeout ป้องกันไว้ที่ระดับ
HTTP client เอง worker process ที่รัน task นั้นจะถูก **จองตลอดไป** ไม่สามารถไปหยิบ
งานอื่นมาทำต่อได้ ยิ่งเกิดซ้ำ ๆ ยิ่งทำให้ worker ทั้งหมดถูกจองจนไม่เหลือ capacity ประมวลผล
งานใหม่เลย (คล้ายปัญหา thread pool exhaustion ในโปรแกรมทั่วไป)

Celery มีกลไกป้องกันระดับ **worker** ที่ไม่ต้องพึ่ง timeout ของ HTTP client เพียงอย่าง
เดียว นั่นคือ **Time Limit**

### 757.2 `soft_time_limit` vs `time_limit`

| พารามิเตอร์ | พฤติกรรมเมื่อครบเวลา | ดักจับได้ในโค้ดไหม |
|---|---|---|
| `soft_time_limit` | Celery ส่ง exception `SoftTimeLimitExceeded` เข้าไปใน task ที่กำลังรันอยู่ | ✅ ดักจับได้ด้วย `try/except` — ใช้ทำ cleanup ก่อนจบงานอย่างสุภาพ |
| `time_limit` (hard limit) | Celery **ฆ่า worker process ทันที** (`SIGKILL`) โดยไม่ให้โอกาส cleanup เลย | ❌ ดักจับไม่ได้ — เป็นทางเลือกสุดท้ายเมื่อ soft limit ไม่ได้ผล |

**แนวทางที่แนะนำ**: ตั้งทั้งคู่เสมอ โดยให้ `time_limit` มากกว่า `soft_time_limit`
เล็กน้อย (เช่น 10-30 วินาที) เพื่อให้ task มีโอกาส cleanup ตัวเองก่อน แล้วค่อยถูกบังคับ
ฆ่าถ้า cleanup เองก็ยังค้างอยู่:

```python
# blog/tasks.py
from celery import shared_task
from celery.exceptions import SoftTimeLimitExceeded

import logging
import requests

logger = logging.getLogger(__name__)


@shared_task(
    bind=True,
    soft_time_limit=25,   # แจ้งเตือนแบบสุภาพหลัง 25 วินาที
    time_limit=35,        # ฆ่าจริงถ้ายังไม่จบภายใน 35 วินาที
)
def fetch_external_seo_score(self, url):
    """เรียก external API วิเคราะห์ SEO ของ URL — API ภายนอกอาจค้างได้เสมอ"""
    try:
        response = requests.get(
            'https://seo-analyzer.example.com/api/score',
            params={'url': url},
            timeout=20,   # timeout ระดับ HTTP client เอง (ควรตั้งคู่กับ soft_time_limit เสมอ)
        )
        response.raise_for_status()
        return response.json()

    except SoftTimeLimitExceeded:
        # มีโอกาส cleanup ก่อนถูกฆ่าจริง เช่น บันทึก log ว่างานนี้ timeout ไว้ตรวจสอบภายหลัง
        logger.error('fetch_external_seo_score timeout สำหรับ url=%s', url)
        raise   # ควร re-raise เสมอ เพื่อให้ Celery ทำเครื่องหมาย task นี้ว่า failed จริง ๆ

    except requests.RequestException as exc:
        logger.warning('fetch_external_seo_score ล้มเหลว url=%s exc=%s', url, exc)
        raise self.retry(exc=exc, countdown=60, max_retries=3)
```

### 757.3 ตั้งค่า Time Limit เป็นค่า Default ของทั้งระบบ

นอกจากตั้งเฉพาะ task เป็นราย ๆ ยังตั้งเป็นค่า default ของ worker ทั้งหมดได้ (task ที่
ไม่ได้ระบุค่าเฉพาะของตัวเองจะใช้ค่านี้แทน):

```python
# config/settings.py
CELERY_TASK_SOFT_TIME_LIMIT = 60    # ค่า default: 60 วินาที สำหรับ task ที่ไม่ได้ระบุเอง
CELERY_TASK_TIME_LIMIT = 90         # ค่า default: 90 วินาที
```

> **ข้อควรระวัง**: `time_limit` ที่ตั้งไว้สั้นเกินไปสำหรับ task ที่ทำงานหนักโดยธรรมชาติ
> (เช่น สร้างรายงาน PDF ขนาดใหญ่ที่ใช้เวลาปกติ 2 นาที) จะทำให้ task นั้นถูกฆ่ากลางคัน
> ทุกครั้งทั้งที่ไม่ได้ค้างจริง ควรตั้งค่า `soft_time_limit`/`time_limit` **เฉพาะ task**
> ที่ต้องการ override ให้ต่างจากค่า default เสมอ โดยพิจารณาจากเวลาทำงานปกติของ task
> นั้นบวก buffer ที่เหมาะสม ไม่ใช่ตั้งค่าเดียวกันหมดทั้งระบบแบบไม่ได้คิด

### 757.4 ตารางสรุปขั้นตอนที่ 757

| หัวข้อ | สรุป |
|---|---|
| ปัญหาที่แก้ | Task ที่เรียก external service ค้างตลอดกาล จองพื้นที่ worker ไม่ให้ทำงานอื่น |
| `soft_time_limit` | ส่ง `SoftTimeLimitExceeded` เข้า task — ดักจับได้เพื่อ cleanup อย่างสุภาพ |
| `time_limit` | ฆ่า worker process ทันที (`SIGKILL`) — ดักจับไม่ได้ ใช้เป็นทางเลือกสุดท้าย |
| ตั้งแบบเฉพาะ task | `@shared_task(soft_time_limit=25, time_limit=35)` |
| ตั้งแบบ default ทั้งระบบ | `CELERY_TASK_SOFT_TIME_LIMIT` / `CELERY_TASK_TIME_LIMIT` ใน settings |
| หลักการตั้งเวลา | ให้ `time_limit` มากกว่า `soft_time_limit` เล็กน้อยเสมอ เพื่อเปิดโอกาส cleanup ก่อน |

---

## ขั้นตอนที่ 758: Idempotency ใน Task — Retry ซ้ำได้อย่างปลอดภัย

### 758.1 ทำไม Task ต้องรันซ้ำได้โดยไม่เกิดผลข้างเคียงซ้ำ

Celery รับประกันการส่งงานแบบ **at-least-once delivery** (ส่งอย่างน้อย 1 ครั้ง) ไม่ใช่
**exactly-once** (ส่งพอดี 1 ครั้ง) หมายความว่า **มีสถานการณ์ที่ task เดียวกันถูกรันซ้ำ
มากกว่า 1 ครั้งได้จริงในชีวิตจริง** เช่น:

- Worker crash **หลัง**รันงานเสร็จ (เช่น ส่งอีเมลไปแล้ว) แต่**ก่อน**ส่ง acknowledgment
  กลับไปที่ broker ว่า "ทำเสร็จแล้ว" — เมื่อ worker ใหม่มาแทนที่ broker จะคิดว่างานนี้
  ยังไม่เสร็จ แล้วส่งให้ทำซ้ำอีกครั้ง
- Task เรียก `self.retry()` เองเพราะเจอ exception ชั่วคราว (เช่น network error) แต่จริง
  ๆ แล้วผลข้างเคียงบางส่วนได้เกิดขึ้นไปแล้วก่อนจุดที่ error (เช่น บันทึกข้อมูลบางส่วนลง
  DB สำเร็จแล้ว แต่ API เรียก third-party ล้มเหลว)
- ตั้งค่า `acks_late=True` (ทวนหัวข้อ 758.4) ซึ่งจงใจทำให้เกิดการรันซ้ำได้ในบางกรณี
  เพื่อแลกกับความปลอดภัยเมื่อ worker ล่มกลางงาน

**หลักการสำคัญที่สุดของ Part นี้**: **ออกแบบทุก task ให้ "Idempotent"** คือรันกี่ครั้ง
ก็ได้ ผลลัพธ์สุดท้ายต้องเหมือนเดิมเสมอ ไม่เกิดผลข้างเคียงซ้ำซ้อน (เช่น ไม่ส่งอีเมลซ้ำ,
ไม่หักเงินซ้ำ, ไม่สร้าง record ซ้ำ)

### 758.2 ตัวอย่างปัญหา: Task ที่ไม่ Idempotent

```python
# ตัวอย่างที่ผิด — ถ้า task นี้ถูกรันซ้ำ (เช่นเพราะ worker crash หลังส่งอีเมลสำเร็จ)
# ผู้ใช้จะได้รับใบแจ้งหนี้ซ้ำ 2 ฉบับ!
@shared_task(bind=True, max_retries=3)
def send_invoice_email_bad(self, order_id):
    order = Order.objects.get(pk=order_id)
    email = EmailMessage(
        subject=f'ใบแจ้งหนี้คำสั่งซื้อ #{order.id}',
        body='...',
        to=[order.customer_email],
    )
    email.send()   # ถ้า worker crash ตรงนี้พอดี (ส่งสำเร็จแล้วแต่ยังไม่ ack)
                    # Celery จะส่งงานนี้ให้รันซ้ำ → ส่งอีเมลซ้ำอีกฉบับ
```

### 758.3 ทางแก้: Idempotency Key ผ่าน Unique Constraint ในฐานข้อมูล

วิธีที่แข็งแรงที่สุด (robust ที่สุด) คือใช้ **unique constraint ของฐานข้อมูล** เป็น
กลไกป้องกันการทำงานซ้ำ ไม่ใช่พึ่งตรรกะ if-check ในโค้ด Python เพียงอย่างเดียว (เพราะ
if-check ธรรมดายังมี race condition ได้เหมือนที่ Part 069 ขั้นตอนที่ 685.1 อธิบายไว้):

```python
# blog/models.py
from django.db import models


class EmailLog(models.Model):
    """
    บันทึกทุกครั้งที่ระบบพยายามส่งอีเมลประเภทหนึ่ง ๆ ให้ order หนึ่ง ๆ
    unique_together ป้องกันการส่งอีเมลประเภทเดียวกันซ้ำสำหรับ order เดียวกัน
    ที่ระดับฐานข้อมูล — แม้จะมี 2 process พยายามส่งพร้อมกันเป๊ะ ๆ ก็ตาม
    """
    order = models.ForeignKey('shop.Order', on_delete=models.CASCADE)
    email_type = models.CharField(max_length=50)   # เช่น 'invoice', 'shipping_confirmation'
    sent_at = models.DateTimeField(auto_now_add=True)

    class Meta:
        unique_together = [('order', 'email_type')]
```

```python
# blog/tasks.py
from django.db import IntegrityError
from django.core.mail import EmailMessage

from .models import EmailLog, Order


@shared_task(bind=True, max_retries=3, autoretry_for=(Exception,), retry_backoff=True)
def send_invoice_email(self, order_id):
    """
    Idempotent เต็มรูปแบบ: สร้าง EmailLog ก่อนส่งจริงเสมอ
    ถ้า record มีอยู่แล้ว (จาก retry ครั้งก่อน) unique_together จะ raise IntegrityError
    ทำให้รู้ทันทีว่า "เคยส่งไปแล้ว" แล้วข้ามการส่งซ้ำอย่างปลอดภัย
    """
    try:
        EmailLog.objects.create(order_id=order_id, email_type='invoice')
    except IntegrityError:
        logger.info('อีเมลใบแจ้งหนี้สำหรับ order_id=%s เคยถูกส่งไปแล้ว ข้ามการส่งซ้ำ', order_id)
        return {'status': 'skipped_duplicate', 'order_id': order_id}

    order = Order.objects.get(pk=order_id)
    email = EmailMessage(
        subject=f'ใบแจ้งหนี้คำสั่งซื้อ #{order.id}',
        body='...',
        to=[order.customer_email],
    )
    email.send()

    return {'status': 'sent', 'order_id': order_id}
```

**หลักการสำคัญของแพทเทิร์นนี้**: เราสร้าง `EmailLog` record **ก่อน**ที่จะทำผลข้างเคียง
จริง (ส่งอีเมล) เสมอ ไม่ใช่สร้างหลังส่งสำเร็จ เพราะถ้าสร้างทีหลัง ยังมีช่องว่างที่
worker อาจ crash ระหว่างส่งอีเมลสำเร็จแล้วแต่ยังไม่ทันบันทึก log — ทำให้ retry รอบถัดไป
มองไม่เห็นว่าเคยส่งไปแล้ว และส่งซ้ำอยู่ดี

### 758.4 `acks_late` และ `worker_prefetch_multiplier`: ความสัมพันธ์กับ At-least-once Delivery

```python
# config/settings.py
CELERY_TASK_ACKS_LATE = True          # ack หลังงานเสร็จ ไม่ใช่ทันทีที่หยิบงานมา
CELERY_WORKER_PREFETCH_MULTIPLIER = 1  # หยิบงานทีละ 1 ต่อ worker process ไม่ prefetch ล่วงหน้า
```

| การตั้งค่า | พฤติกรรม | ข้อดี | ข้อเสีย |
|---|---|---|---|
| `acks_late=False` (ค่า default ดั้งเดิม) | Ack ทันทีที่ worker **หยิบ** งานมา (ก่อนรันเสร็จด้วยซ้ำ) | Throughput สูงกว่าเล็กน้อย | ถ้า worker crash **ระหว่าง**ทำงาน งานนั้นจะหายไปเลย ไม่มีใครรันซ้ำให้ |
| `acks_late=True` | Ack **หลัง**งานรันเสร็จสมบูรณ์เท่านั้น | ถ้า worker crash กลางงาน broker จะส่งงานให้ worker ตัวอื่นรันใหม่ ไม่มีงานไหนหายไปเงียบ ๆ | เพิ่มโอกาสที่ task จะถูกรันซ้ำ (เช่นเดียวกับสถานการณ์ในขั้นตอนที่ 758.1) — **จึงต้องคู่กับการออกแบบ idempotent เสมอ** |

**คำแนะนำระดับมืออาชีพ**: ตั้ง `CELERY_TASK_ACKS_LATE = True` สำหรับ task ที่ **สำคัญ
และรับผลข้างเคียงซ้ำไม่ได้ถ้าไม่มีการป้องกัน** (เช่น ส่งอีเมล, เรียก payment API) ควบคู่
กับการออกแบบ idempotent ตามขั้นตอนที่ 758.3 เสมอ — ทั้งสองสิ่งนี้ **ต้องมาคู่กัน**
`acks_late` เพียงอย่างเดียวโดยไม่มี idempotency จะทำให้เกิดปัญหาส่งซ้ำ ส่วน idempotency
เพียงอย่างเดียวโดยไม่มี `acks_late` ก็ยังเสี่ยงงานหายไปเงียบ ๆ เมื่อ worker crash

### 758.5 ตัวอย่าง Idempotency สำหรับ Task ที่ไม่มี "การส่ง" แต่มีการ "คำนวณ/บันทึก"

Idempotency ไม่ได้จำกัดแค่การส่งอีเมล ใช้ได้กับทุก task ที่มีผลข้างเคียง เช่น task
`sync_post_view_stats` จากขั้นตอนที่ 751.5 ก็ **เป็น idempotent อยู่แล้วโดยธรรมชาติ**
เพราะมันแค่ "ตั้งค่า" (`post.views_count = views`) ไม่ใช่ "บวกเพิ่ม" (`post.views_count
+= views`) — รันกี่ครั้งซ้อนกันก็ได้ค่าสุดท้ายเหมือนเดิมเสมอ ตราบใดที่ข้อมูลต้นทางใน
Redis ไม่เปลี่ยน นี่คือหลักการออกแบบที่ดี: **เลือกใช้ operation แบบ "set ค่าสุดท้าย"
แทน "เพิ่มค่าทับไปเรื่อย ๆ" เมื่อทำได้ เพราะ set ซ้ำกี่ครั้งก็ได้ผลเหมือนเดิม แต่ increment
ซ้ำจะได้ผลลัพธ์ผิดเพี้ยนทุกครั้งที่รันซ้ำ**

### 758.6 ตารางสรุปขั้นตอนที่ 758

| หัวข้อ | สรุป |
|---|---|
| ทำไมต้อง idempotent | Celery รับประกันแค่ at-least-once ไม่ใช่ exactly-once — task อาจถูกรันซ้ำได้จริง |
| ทางแก้ที่แข็งแรงที่สุด | Idempotency key ผ่าน unique constraint ของฐานข้อมูล ไม่ใช่ if-check ธรรมดา |
| `acks_late=True` | ป้องกันงานหายเมื่อ worker crash กลางงาน แต่เพิ่มโอกาสรันซ้ำ — ต้องคู่กับ idempotency เสมอ |
| หลักการออกแบบ | เลือก "set ค่าสุดท้าย" แทน "increment ทับไปเรื่อย ๆ" เมื่อทำได้ |
| จุดที่ต้องระวังที่สุด | สร้าง idempotency record **ก่อน** ทำผลข้างเคียงจริงเสมอ ไม่ใช่หลังทำสำเร็จ |

---

## ขั้นตอนที่ 759: การเขียน Test สำหรับ Celery Task

### 759.1 ปัญหาของการเทส Task แบบไม่ตั้งค่าอะไรเลย

ถ้าเรียก `some_task.delay()` ในเทสโดยไม่ตั้งค่าอะไรเป็นพิเศษ Celery จะพยายาม **ยิงงาน
เข้า broker จริง** (Redis) แล้วรอ worker จริงมาหยิบไปทำ — ซึ่งเทสจะไม่รู้ผลลัพธ์ทันที
(async) ทำให้เทสไม่แน่นอน (flaky) หรือต้องรัน worker คู่ไปกับเทสเสมอ ซึ่งช้าและซับซ้อน
โดยไม่จำเป็น

### 759.2 `CELERY_TASK_ALWAYS_EAGER`: รัน Task แบบ Synchronous ทันทีในเทส

Celery มีโหมด **eager** ที่ทำให้ `.delay()`/`.apply_async()` **รันโค้ดของ task ทันที
แบบ synchronous ในกระบวนการเดียวกับที่เรียก** โดยไม่ผ่าน broker/worker จริงเลย เหมาะ
สำหรับเทสเป็นอย่างยิ่ง:

```python
# config/settings.py (หรือไฟล์ settings/test.py แยกต่างหากถ้ามี)
CELERY_TASK_ALWAYS_EAGER = True
CELERY_TASK_EAGER_PROPAGATES = True   # ให้ exception ใน task โผล่ขึ้นมาที่เทสจริง ๆ (สำคัญมาก!)
```

> **ทำไมต้องตั้ง `CELERY_TASK_EAGER_PROPAGATES = True` คู่กันเสมอ**: ถ้าไม่ตั้งค่านี้
> เมื่อ task เกิด exception ในโหมด eager ผลลัพธ์จะถูกเก็บไว้ในสถานะ `FAILURE` เงียบ ๆ
> โดยไม่ raise exception ออกมาให้เทสเห็น ทำให้เทสอาจ "ผ่าน" ทั้งที่ task ข้างในพังจริง ๆ
> — นี่คือกับดักที่พบบ่อยที่สุดเวลาเทส Celery task

### 759.3 ตั้งค่าเฉพาะสภาพแวดล้อมเทสด้วย `pytest-django`

แนะนำให้ตั้งค่านี้เฉพาะตอนรันเทสเท่านั้น (ไม่ปนกับ development/production settings)
โดยใช้ fixture ของ `pytest-django` (ทวนจาก Part 061):

```python
# conftest.py
import pytest


@pytest.fixture(autouse=True)
def celery_eager_mode(settings):
    """บังคับให้ทุกเทสรัน Celery task แบบ eager (synchronous) โดยอัตโนมัติ"""
    settings.CELERY_TASK_ALWAYS_EAGER = True
    settings.CELERY_TASK_EAGER_PROPAGATES = True
```

### 759.4 ตัวอย่างเทสจริง: Task เดี่ยว, Chain, และ Idempotency

```python
# blog/tests/test_tasks.py
import pytest
from django.core import mail

from blog.models import Post, Category, EmailLog
from blog.tasks import sync_post_view_stats, send_invoice_email
from shop.models import Order


@pytest.mark.django_db
def test_sync_post_view_stats_updates_post_from_redis(mocker):
    """เทส task เดี่ยว — mock Redis connection เพื่อไม่ต้องพึ่ง Redis จริงระหว่างเทส"""
    post = Post.objects.create(title='Test Post', views_count=0, likes_count=0)

    fake_redis = mocker.MagicMock()
    fake_redis.scan_iter.return_value = [f'post_stats:{post.id}'.encode()]
    fake_redis.hgetall.return_value = {b'views': b'150', b'likes': b'12'}
    mocker.patch('blog.tasks.get_redis_connection', return_value=fake_redis)

    result = sync_post_view_stats()   # เรียกตรง ๆ ได้เลยเพราะ eager mode ทำให้เป็น sync function

    post.refresh_from_db()
    assert post.views_count == 150
    assert post.likes_count == 12
    assert result == {'synced_posts': 1}


@pytest.mark.django_db
def test_send_invoice_email_sends_once(django_user_model):
    """เทสว่าอีเมลถูกส่งจริง และมีการสร้าง EmailLog เพื่อป้องกันการส่งซ้ำ"""
    order = Order.objects.create(customer_email='customer@example.com', total=999)

    result = send_invoice_email(order.id)

    assert result['status'] == 'sent'
    assert len(mail.outbox) == 1
    assert mail.outbox[0].to == ['customer@example.com']
    assert EmailLog.objects.filter(order=order, email_type='invoice').exists()


@pytest.mark.django_db
def test_send_invoice_email_is_idempotent_on_retry(django_user_model):
    """
    หัวใจของขั้นตอนที่ 758: เรียกซ้ำ 2 ครั้งด้วย order เดิม
    ต้องส่งอีเมลแค่ครั้งเดียว ไม่ใช่ 2 ครั้ง แม้ task จะถูกเรียกซ้ำ (จำลอง retry)
    """
    order = Order.objects.create(customer_email='customer@example.com', total=999)

    send_invoice_email(order.id)
    result_second_call = send_invoice_email(order.id)   # จำลองการรันซ้ำ

    assert result_second_call['status'] == 'skipped_duplicate'
    assert len(mail.outbox) == 1   # ยืนยันว่าส่งแค่ครั้งเดียวจริง ๆ แม้เรียก task 2 ครั้ง
    assert EmailLog.objects.filter(order=order, email_type='invoice').count() == 1
```

### 759.5 ข้อควรระวัง: Chord ในโหมด Eager มีข้อจำกัด

**ข้อควรระวังสำคัญที่ต้องรู้**: แม้ `chain()` และ `group()` จะทำงานถูกต้องตรงตาม
พฤติกรรมจริงในโหมด eager แต่ **`chord()` ในโหมด eager มีพฤติกรรมที่ต่างจากการรันจริง
ผ่าน broker เล็กน้อย** — ในโหมด eager, Celery จะรัน header ทุกตัวแบบ synchronous
เรียงตามลำดับ (ไม่ได้ขนานจริง เพราะไม่มี worker หลายตัว) แล้วค่อยเรียก callback ทันที
ซึ่ง**ผลลัพธ์สุดท้ายถูกต้องเหมือนกัน** แต่ **ไม่ได้ทดสอบพฤติกรรม concurrency/race
condition จริงของ chord** ดังนั้นเทส chord ในโหมด eager ควรใช้ยืนยันแค่ **"logic ของ
callback ถูกต้องเมื่อได้รับผลลัพธ์ที่คาดหวัง"** เท่านั้น ส่วนพฤติกรรมด้าน concurrency
จริง (เช่น การรอครบทุก task ก่อน callback เมื่อรันแบบขนานจริงบน worker หลายตัว) ควร
ทดสอบแยกต่างหากด้วย **integration test ที่รัน worker จริง** (มักทำใน CI แยก stage
หรือทดสอบบน staging environment ก่อน deploy production):

```python
# blog/tests/test_reports.py
import pytest

from blog.models import Category
from blog.services.reports import build_full_category_report


@pytest.mark.django_db
def test_build_full_category_report_combines_all_categories():
    """
    เทส logic ของ chord ในโหมด eager — ยืนยันว่า callback รวมผลถูกต้อง
    เมื่อได้รับผลลัพธ์จากทุกหมวดหมู่ (ไม่ได้ยืนยัน concurrency จริง)
    """
    cat1 = Category.objects.create(name='Python')
    cat2 = Category.objects.create(name='Django')

    async_result = build_full_category_report()
    report = async_result  # ในโหมด eager .apply_async() คืนค่า EagerResult ที่ .get() ได้ทันที

    # ในโหมด eager สามารถเรียก .get() ได้อย่างปลอดภัย เพราะไม่มีการ block รอ worker จริง
    from celery.result import AsyncResult
    if hasattr(async_result, 'get'):
        report = async_result.get()

    assert len(report['categories']) == 2
```

### 759.6 ตารางสรุปขั้นตอนที่ 759

| หัวข้อ | สรุป |
|---|---|
| ปัญหาที่แก้ | เทสไม่ควรพึ่ง broker/worker จริง เพราะทำให้ flaky และช้า |
| ตั้งค่าหลัก | `CELERY_TASK_ALWAYS_EAGER = True` + `CELERY_TASK_EAGER_PROPAGATES = True` (คู่กันเสมอ) |
| ตั้งใน pytest | fixture `autouse=True` ที่ override `settings` |
| เทส idempotency | เรียก task ซ้ำ 2 ครั้งในเทสเดียว ยืนยันผลข้างเคียงเกิดแค่ครั้งเดียว |
| ข้อควรระวัง | `chord()` ในโหมด eager ไม่ได้ทดสอบ concurrency จริง — ต้องมี integration test แยกสำหรับพฤติกรรมนั้น |

---

## ขั้นตอนที่ 760: สรุปและแบบฝึกหัด

### 760.1 Case Study เต็มรูปแบบ: Pipeline สร้างรายงานสถิติบล็อกรายสัปดาห์

มารวมทุกอย่างที่เรียนมาใน Part นี้เข้าด้วยกันเป็นระบบเดียว: **ทุกวันจันทร์เวลา 07:00 น.
ให้ระบบรวบรวมสถิติของทุกหมวดหมู่แบบขนาน (Group) รวมผลเป็นรายงานเดียว (Chord), สร้าง
ไฟล์ PDF และส่งอีเมลตามลำดับ (Chain), โดยงานทั้งหมดอยู่ใน queue ที่ไม่เร่งด่วน (Task
Routing), มี time limit ป้องกันค้าง, ออกแบบให้ idempotent, และตั้งเวลาผ่าน Admin
(`django-celery-beat`)**

```python
# blog/tasks.py — รวม task ทั้งหมดของ pipeline นี้ไว้ในที่เดียว
import logging

from celery import shared_task
from celery.exceptions import SoftTimeLimitExceeded
from django.core.mail import EmailMessage
from django.db import IntegrityError, transaction
from django.utils import timezone

from .models import Category, Post, WeeklyReport

logger = logging.getLogger(__name__)


@shared_task(
    name='blog.tasks.gather_category_stats_v2',
    queue='low_priority',
    soft_time_limit=20,
    time_limit=30,
    bind=True,
    autoretry_for=(Exception,),
    retry_backoff=True,
    max_retries=3,
)
def gather_category_stats_v2(self, category_id):
    try:
        category = Category.objects.get(pk=category_id)
        posts = Post.objects.filter(category=category, is_published=True)
        return {
            'category_id': category_id,
            'category_name': category.name,
            'post_count': posts.count(),
            'total_views': sum(posts.values_list('views_count', flat=True)),
        }
    except SoftTimeLimitExceeded:
        logger.error('gather_category_stats_v2 timeout category_id=%s', category_id)
        raise


@shared_task(name='blog.tasks.build_weekly_report', queue='low_priority')
def build_weekly_report(category_results, week_number):
    """
    Callback ของ chord — idempotent ผ่าน get_or_create บน (week_number)
    รันซ้ำกี่ครั้งก็ได้รายงานสัปดาห์เดียวกันเสมอ ไม่สร้างซ้ำ
    """
    report, created = WeeklyReport.objects.get_or_create(
        week_number=week_number,
        defaults={
            'data': {
                'categories': category_results,
                'total_posts': sum(r['post_count'] for r in category_results),
                'total_views': sum(r['total_views'] for r in category_results),
            },
            'generated_at': timezone.now(),
        },
    )
    if not created:
        logger.info('WeeklyReport week=%s มีอยู่แล้ว ไม่สร้างซ้ำ', week_number)

    return report.id


@shared_task(name='blog.tasks.render_and_send_weekly_report', queue='low_priority')
def render_and_send_weekly_report(report_id):
    """สร้าง PDF จาก WeeklyReport แล้วส่งอีเมล — idempotent ผ่าน EmailLog เหมือนขั้นตอนที่ 758.3"""
    from .models import EmailLog

    try:
        EmailLog.objects.create(
            order_id=None, email_type=f'weekly_report_{report_id}',
        )
    except IntegrityError:
        logger.info('รายงานสัปดาห์ id=%s เคยถูกส่งไปแล้ว ข้าม', report_id)
        return {'status': 'skipped_duplicate'}

    report = WeeklyReport.objects.get(pk=report_id)
    email = EmailMessage(
        subject=f'รายงานสถิติบล็อกประจำสัปดาห์ที่ {report.week_number}',
        body=f"จำนวนโพสต์: {report.data['total_posts']}, ยอด view รวม: {report.data['total_views']}",
        to=['admin@example.com'],
    )
    email.send()
    return {'status': 'sent', 'report_id': report_id}


@shared_task(name='blog.tasks.notify_weekly_pipeline_failure', queue='low_priority')
def notify_weekly_pipeline_failure(request, exc, traceback):
    logger.error('Weekly report pipeline ล้มเหลว: task=%s exc=%s', request.task, exc)
```

ประกอบเป็น pipeline เดียวด้วย Chord + Chain:

```python
# blog/services/weekly_pipeline.py
from celery import chain, chord

from blog.tasks import (
    gather_category_stats_v2,
    build_weekly_report,
    render_and_send_weekly_report,
    notify_weekly_pipeline_failure,
)
from blog.models import Category


def trigger_weekly_report_pipeline(week_number):
    """
    โครงสร้างเต็มของ pipeline:
    Chord(Group ของ gather_category_stats_v2 ทุกหมวดหมู่ -> build_weekly_report)
      | render_and_send_weekly_report
    เขียนด้วย chord แล้วต่อท้ายด้วย chain โดยตรงผ่าน operator |
    """
    category_ids = list(Category.objects.values_list('id', flat=True))

    header = [
        gather_category_stats_v2.s(cid).on_error(notify_weekly_pipeline_failure.s())
        for cid in category_ids
    ]
    callback = build_weekly_report.s(week_number=week_number)

    workflow = chord(header, callback) | render_and_send_weekly_report.s()
    workflow.apply_async(link_error=notify_weekly_pipeline_failure.s())
```

ตั้ง schedule ผ่าน `django-celery-beat` (ทวนขั้นตอนที่ 752.4) ให้รันทุกวันจันทร์ 07:00:

```python
# blog/migrations/0010_schedule_weekly_report_pipeline.py
from django.db import migrations
import json


def create_schedule(apps, schema_editor):
    CrontabSchedule = apps.get_model('django_celery_beat', 'CrontabSchedule')
    PeriodicTask = apps.get_model('django_celery_beat', 'PeriodicTask')

    monday_7am, _ = CrontabSchedule.objects.get_or_create(
        minute='0', hour='7', day_of_week='1', day_of_month='*', month_of_year='*',
    )
    PeriodicTask.objects.get_or_create(
        name='trigger-weekly-report-pipeline',
        defaults={
            'task': 'blog.tasks.trigger_weekly_report_pipeline_entrypoint',
            'crontab': monday_7am,
            'enabled': True,
        },
    )


class Migration(migrations.Migration):
    dependencies = [('blog', '0009_create_default_periodic_tasks')]
    operations = [migrations.RunPython(create_schedule, migrations.RunPython.noop)]
```

```python
# blog/tasks.py — entrypoint แยกต่างหาก เพราะ Beat ยิงได้แค่ task เดียว ไม่ใช่ pipeline object โดยตรง
from django.utils import timezone


@shared_task(name='blog.tasks.trigger_weekly_report_pipeline_entrypoint')
def trigger_weekly_report_pipeline_entrypoint():
    """Wrapper ธรรมดาให้ Celery Beat เรียกได้ — ข้างในค่อยประกอบ chord/chain จริง"""
    from blog.services.weekly_pipeline import trigger_weekly_report_pipeline

    week_number = timezone.now().isocalendar()[1]
    trigger_weekly_report_pipeline(week_number)
```

> **ทำไมต้องมี "entrypoint task" แยก**: `django-celery-beat` เก็บแค่ **ชื่อ task เดียว**
> ต่อ 1 schedule เท่านั้น มันไม่รู้จักและยิง `chain()`/`chord()` object ตรง ๆ ไม่ได้
> ทางแก้มาตรฐานคือสร้าง task ธรรมดาสั้น ๆ (entrypoint) ให้ Beat เรียก แล้วให้ task นั้น
> เป็นคนประกอบและยิง chain/chord ที่ซับซ้อนอีกทีจากข้างใน

### 760.2 สรุปสิ่งที่ได้เรียนรู้ใน Part นี้

- ✅ เข้าใจ Celery Beat และทำไมดีกว่า cron แบบดั้งเดิมสำหรับงานที่ผูกกับแอป Django
- ✅ ตั้งค่า schedule แบบ hardcode ด้วย `CELERY_BEAT_SCHEDULE` และแบบผ่านฐานข้อมูลด้วย
  `django-celery-beat` ที่แก้ไขผ่าน Admin ได้โดยไม่ต้อง deploy ใหม่
- ✅ สร้าง periodic task ผ่านทั้ง Admin UI และ data migration (พร้อมหลักการ idempotent
  migration ด้วย `get_or_create`)
- ✅ ใช้ `chain()` เชื่อม task ที่ต้องรันตามลำดับ พร้อมเข้าใจ `.s()` vs `.si()`
- ✅ ใช้ `group()` รันงานที่เป็นอิสระจากกันแบบขนาน และรู้ว่าทำไมไม่ควร `.get()` แบบไม่ระวัง
- ✅ ใช้ `chord()` รวม group + callback โดยไม่ต้อง block รอผลลัพธ์เลย
- ✅ แยก task ไปคนละ queue ตามความสำคัญด้วย `CELERY_TASK_ROUTES` และเข้าใจข้อจำกัดของ
  priority queue บน Redis broker
- ✅ ป้องกัน task ค้างด้วย `soft_time_limit`/`time_limit`
- ✅ ออกแบบ task ให้ idempotent ด้วย unique constraint ระดับฐานข้อมูล เพื่อรองรับ
  at-least-once delivery ของ Celery อย่างปลอดภัย
- ✅ เขียนเทส Celery task ด้วย `CELERY_TASK_ALWAYS_EAGER` และรู้ข้อจำกัดของการเทส chord
  ในโหมด eager
- ✅ ประกอบทุกเทคนิคเข้าเป็น pipeline สร้างรายงานสถิติบล็อกรายสัปดาห์ที่ใช้งานได้จริง

### 760.3 Checklist ก่อนไป Part ถัดไป

- [ ] รัน `celery -A config beat --loglevel=info` แล้วเห็น log ยิง task ตามเวลาที่ตั้งไว้
- [ ] ติดตั้ง `django-celery-beat`, migrate, และสร้าง periodic task ผ่าน Django Admin สำเร็จ
- [ ] เขียน `chain()` อย่างน้อย 1 pipeline ที่ผลลัพธ์ของ task หนึ่งส่งต่อให้อีก task หนึ่ง
- [ ] เขียน `group()` รันหลาย task พร้อมกัน และตรวจสอบด้วย `GroupResult.get()` (นอก view)
- [ ] เขียน `chord()` ที่มี callback รวมผลลัพธ์จาก group ได้ถูกต้อง
- [ ] ตั้งค่า `CELERY_TASK_ROUTES` แยก queue อย่างน้อย 2 queue และรัน worker แยกกัน
- [ ] เขียน task ที่มี `soft_time_limit`/`time_limit` และทดสอบว่า `SoftTimeLimitExceeded` ถูก raise จริง
- [ ] เขียน task ที่ idempotent ด้วย unique constraint และเทสยืนยันว่าเรียกซ้ำแล้วไม่เกิดผลข้างเคียงซ้ำ
- [ ] ตั้ง `CELERY_TASK_ALWAYS_EAGER = True` ในสภาพแวดล้อมเทส และเทส task ผ่านสำเร็จ
- [ ] รัน pipeline สร้างรายงานรายสัปดาห์เต็มรูปแบบจากขั้นตอนที่ 760.1 ได้จริงบนเครื่องของคุณ

### 760.4 แบบฝึกหัดท้ายบท

**แบบฝึกหัดที่ 1**: สร้าง periodic task ใหม่ชื่อ `cleanup_old_notifications` ที่ลบ
notification ที่มีอายุเกิน 30 วันออกจากฐานข้อมูล ตั้งให้รันทุกวันเวลาตี 3 ผ่าน
`django-celery-beat` ทั้งแบบสร้างผ่าน Admin และแบบสร้างผ่าน data migration แล้วเทียบว่า
ทั้งสองวิธีให้ผลลัพธ์เหมือนกันหรือไม่เมื่อเช็คในตาราง `PeriodicTask`

**แบบฝึกหัดที่ 2**: เขียน chain 4 ขั้นตอนสำหรับกระบวนการ "ประมวลผลรูปภาพที่ผู้ใช้อัปโหลด":
resize รูปภาพ → สร้าง thumbnail 3 ขนาด → อัปโหลดขึ้น cloud storage → อัปเดต URL ใน Model
ทดสอบว่าถ้า resize ล้มเหลว (ใส่ path ไฟล์ผิด) ขั้นตอนที่เหลือใน chain จะไม่ถูกรันต่อจริง
และ error callback ที่คุณเขียนถูกเรียกอย่างถูกต้อง

**แบบฝึกหัดที่ 3**: สร้าง chord ที่ดึงข้อมูลราคาสินค้าจาก 5 marketplace ภายนอกพร้อมกัน
(mock ด้วย `time.sleep()` สุ่มค่าเพื่อจำลอง latency ของแต่ละที่) แล้ว callback หาราคาต่ำ
สุดพร้อมชื่อร้าน วัดเวลาที่ใช้ทั้ง pipeline เทียบกับถ้าเรียกทีละ marketplace ตามลำดับ
(เขียนทั้งสองแบบแล้ว benchmark เทียบกันจริง ด้วย `time.perf_counter()`)

**แบบฝึกหัดที่ 4 (ขั้นสูง)**: ออกแบบ task `charge_customer_payment(order_id, amount)`
ให้เป็น idempotent อย่างสมบูรณ์ โดยจำลองว่าเรียก payment gateway ภายนอกที่บางครั้ง
"ตอบกลับช้าจนเหมือนล้มเหลว แต่จริง ๆ หักเงินสำเร็จไปแล้ว" (จำลองด้วยการสุ่ม exception
หลัง sleep) ออกแบบให้ระบบไม่หักเงินซ้ำแม้ retry กี่ครั้งก็ตาม โดยใช้ทั้ง idempotency key
ระดับฐานข้อมูล **และ** `acks_late=True` ร่วมกัน แล้วเขียนเทสยืนยันด้วยการเรียก task
ซ้ำ 5 ครั้งติดกันในเทสเดียว ตรวจสอบว่ายอดเงินถูกหักแค่ครั้งเดียว

### 760.5 คำถามที่พบบ่อย (FAQ)

**Q: ต้องใช้ `django-celery-beat` เสมอไหม หรือใช้ `CELERY_BEAT_SCHEDULE` แบบ hardcode
พอสำหรับโปรเจกต์เล็ก ๆ?**
A: สำหรับโปรเจกต์ส่วนตัวหรือ side project ที่ schedule ไม่ค่อยเปลี่ยน `CELERY_BEAT_SCHEDULE`
แบบ hardcode ก็เพียงพอและเรียบง่ายกว่า แต่ทันทีที่มีทีมงานที่ไม่ใช่โปรแกรมเมอร์ต้อง
ปรับเวลา หรือ schedule เปลี่ยนบ่อย `django-celery-beat` คุ้มค่ากับความซับซ้อนที่เพิ่มขึ้น
เล็กน้อยแน่นอน

**Q: ทำไม chord ของฉันไม่ยอมรัน callback เลย ทั้งที่ทุก task ใน group ดูเหมือนจะเสร็จแล้ว?**
A: สาเหตุที่พบบ่อยที่สุดคือลืมตั้งค่า `CELERY_RESULT_BACKEND` (ทวนขั้นตอนที่ 755.3)
เพราะ chord ต้องพึ่ง result backend ในการติดตามว่า task ไหนเสร็จแล้วบ้าง สาเหตุรองลงมา
คือมี task ตัวใดตัวหนึ่งใน group ล้มเหลวแบบเงียบ ๆ (silent failure) ทำให้ chord รอ
"ตลอดกาล" เพราะ backend คิดว่ายังไม่ครบ — ควรตรวจสอบ log ของทุก task ใน header อย่าง
ละเอียดเสมอเมื่อ chord ไม่ทำงานตามคาด

**Q: ควรตั้ง `acks_late=True` เป็นค่า default ของทุก task เลยหรือไม่?**
A: ไม่ควรตั้งเป็น default แบบเหมารวมทุก task โดยไม่คิด เพราะ task ที่ **ไม่ idempotent**
(และไม่มีแผนจะทำให้เป็น) จะเสี่ยงเกิดผลข้างเคียงซ้ำมากขึ้นถ้าเปิด `acks_late` คำแนะนำคือ
เปิดเฉพาะ task ที่ (1) สำคัญมากจนรับไม่ได้ถ้างานหายไปเงียบ ๆ **และ** (2) ได้ออกแบบให้
idempotent ตามขั้นตอนที่ 758 แล้วเรียบร้อย ทั้งสองเงื่อนไขต้องมาคู่กันเสมอ

**Q: Chain/Group/Chord ใช้กับ Django Channels (WebSocket) ที่เรียนใน Part 057/074
ร่วมกันได้ไหม?**
A: ได้ และเป็นแพทเทิร์นที่พบบ่อยมากในระบบจริง เช่น ใช้ chord รวบรวมผลการประมวลผลหนัก ๆ
แบบขนาน แล้วให้ callback สุดท้ายส่ง message ผ่าน Channels layer ไปแจ้งผู้ใช้แบบ real-time
ว่า "งานเสร็จแล้ว" แทนที่จะให้ผู้ใช้ต้อง refresh หน้าเว็บเองหรือ poll ผลลัพธ์ซ้ำ ๆ — เรา
จะสร้างระบบแจ้งเตือนแบบ real-time เต็มรูปแบบนี้ใน Part 078

---

## เตรียมตัวสำหรับ Part ถัดไป

**Part 077: Message Queue ด้วย RabbitMQ/Redis** จะพาไปเจาะลึกเบื้องหลังของ message
broker ที่ Celery ใช้งานอยู่ ตั้งแต่ทำความเข้าใจ AMQP protocol ของ RabbitMQ เปรียบเทียบ
RabbitMQ กับ Redis ในฐานะ broker อย่างละเอียด (จุดที่ Redis ทำได้จำกัดอย่าง priority
queue ที่เกริ่นไว้ในขั้นตอนที่ 756.4 จะได้คำตอบเต็มที่นี่), การสลับ broker จาก Redis
ไปเป็น RabbitMQ ในโปรเจกต์ที่มีอยู่แล้วโดยไม่กระทบ business logic ของ task เลย, Dead
Letter Queue สำหรับเก็บงานที่ล้มเหลวซ้ำ ๆ, และการออกแบบระบบให้ทนทานเมื่อ broker
ล่มชั่วคราว เตรียม Celery worker และ Beat ที่ทำงานได้แล้วจาก Part นี้ไว้ให้พร้อม
เพราะเราจะต่อยอดจากโค้ดชุดเดียวกันนี้ทันที!
