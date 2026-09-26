# Part 078: Real-time Notification System

> **ขั้นตอนที่ 771-780 ของหลักสูตร** | Phase 9: Async, Celery และ Channels
>
> นี่คือ Part สรุปยอดของ Phase 9 — โปรเจกต์แคปสโตน (capstone) ที่ประกอบ **ทุกเทคนิค**
> ที่เรียนมาตลอด Phase นี้เข้าเป็นฟีเจอร์เดียวที่ใช้งานได้จริงในระดับมืออาชีพ:
> `post_save` Signal จาก Part 019 จุดชนวนเหตุการณ์, Celery Task จาก Part 075-076
> รับงานไปทำเบื้องหลังโดยไม่บล็อก request, Channel Layer และ Consumer จาก Part 057
> และ 074 ส่งข้อความไปหา browser ที่เชื่อมต่ออยู่แบบ real-time, ส่วน HTMX และ
> Alpine.js จาก Phase 6 (Part 053-054) ทำหน้าที่แสดงผล badge นับจำนวนที่ยังไม่ได้
> อ่านแบบ reactive โดยไม่ต้อง reload หน้า เราจะสร้างระบบแจ้งเตือนที่สมบูรณ์ครบวงจร:
> ตั้งแต่ Model ที่ยืดหยุ่นด้วย Generic Foreign Key, การส่งแบบ real-time ผ่าน
> WebSocket, Email Digest รายวันด้วย Celery Beat, การเกริ่น Web Push Notification
> สำหรับแจ้งเตือนแม้ปิดแท็บ, ระบบให้ผู้ใช้เลือก opt-out เป็นรายประเภท, ไปจนถึงการ
> เขียนเทสต์ครอบคลุมทั้ง pipeline ตั้งแต่ signal ยันหน้าจอผู้ใช้

---

## สารบัญของ Part นี้

- ขั้นตอนที่ 771: ออกแบบสถาปัตยกรรมระบบแจ้งเตือนแบบ Real-time ทั้งระบบ
- ขั้นตอนที่ 772: ออกแบบ `Notification` Model ด้วย Generic Foreign Key
- ขั้นตอนที่ 773: Trigger การแจ้งเตือนผ่าน Signal แล้วส่งต่อเป็น Celery Task
- ขั้นตอนที่ 774: ส่งการแจ้งเตือนแบบ Real-time ผ่าน Channels WebSocket
- ขั้นตอนที่ 775: UI Badge/Counter แจ้งเตือนด้วย Alpine.js และ HTMX
- ขั้นตอนที่ 776: Email Digest — รวมการแจ้งเตือนส่งสรุปด้วย Celery Beat
- ขั้นตอนที่ 777: เกริ่น Web Push Notification สำหรับแจ้งเตือนแม้ปิดแท็บ
- ขั้นตอนที่ 778: ระบบตั้งค่าการแจ้งเตือนของผู้ใช้ (Opt-out รายประเภท)
- ขั้นตอนที่ 779: การเขียนเทสต์สำหรับ Pipeline การแจ้งเตือนทั้งระบบ
- ขั้นตอนที่ 780: สรุปและแบบฝึกหัด — ระบบแจ้งเตือนเต็มรูปแบบ

---

## ขั้นตอนที่ 771: ออกแบบสถาปัตยกรรมระบบแจ้งเตือนแบบ Real-time ทั้งระบบ

### 771.1 โจทย์ของ Part นี้

สถานการณ์เป้าหมาย: คุณ (`somchai`) เขียนโพสต์ไว้ในบล็อก แล้วมีคนอื่น (`malee`) เข้ามา
แสดงความคิดเห็น สิ่งที่ควรเกิดขึ้นทันทีคือ:

1. แถวข้อมูล `Notification` ใหม่ถูกสร้างในฐานข้อมูล บอกว่า "malee แสดงความคิดเห็นบน
   โพสต์ของคุณ"
2. ถ้า `somchai` เปิดเว็บอยู่ตอนนั้นพอดี (มี WebSocket connection ค้างอยู่) ตัวเลข
   badge มุมขวาบนของหน้าเว็บต้องเด้งขึ้นทันที **โดยไม่ต้อง refresh หน้า**
3. ถ้า `somchai` ไม่ได้เปิดเว็บอยู่ ให้สะสมการแจ้งเตือนไว้ แล้วส่งสรุปเป็นอีเมล
   ตอนเช้าวันถัดไป (Email Digest)
4. ถ้า `somchai` ปิดแท็บเบราว์เซอร์ไปเลยแต่เปิด permission ไว้ ให้ยิง
   Web Push Notification ไปโผล่บนหน้าจอได้แม้ไม่มีแท็บเปิดอยู่

**ข้อจำกัดสำคัญที่ต้องยึดตลอดทั้ง Part**: การสร้าง Comment ต้อง**ตอบสนองเร็วเท่าเดิม**
ไม่ว่าจะมีผู้รับการแจ้งเตือนกี่คน ไม่ว่า Redis หรือ SMTP จะช้าแค่ไหนในขณะนั้น — ทวนกฎ
เหล็กจาก Part 075 ข้อ 741: **งานที่ผู้ใช้ไม่จำเป็นต้องเห็นผลทันทีต้องออกจาก request
cycle เสมอ**

### 771.2 ทำไมต้องมี Celery คั่นกลางระหว่าง Signal กับ WebSocket

มือใหม่หลายคนพอเรียน Channels (Part 057) จบ มักเขียนโค้ดแบบลัดขั้นตอนนี้ในหัว
`signals.py`:

```python
# ❌ อย่าทำแบบนี้ — เรียก async channel layer ตรงจาก signal แบบ sync
@receiver(post_save, sender=Comment)
def notify_wrong_way(sender, instance, created, **kwargs):
    if not created:
        return
    Notification.objects.create(...)          # เขียน DB ตรงใน request cycle
    channel_layer = get_channel_layer()
    async_to_sync(channel_layer.group_send)(...)  # เรียก Redis ตรงใน request cycle
```

โค้ดนี้ **ทำงานได้** แต่ทำลายเป้าหมายของ Part 075 ทั้งหมด: ทั้งการเขียน `Notification`
ลงฐานข้อมูล และการเรียก `group_send()` ไปยัง Redis (Channel Layer) ต่างก็เป็น I/O ที่
เกิดขึ้น**ระหว่าง request** ของคนที่กำลังโพสต์คอมเมนต์อยู่ ถ้า Redis ช้าเพียงชั่วครู่
(เช่น กำลังถูก query หนักจากกลุ่มแชทอื่น) ผู้ใช้ที่แค่อยากโพสต์คอมเมนต์จะต้องรอไปด้วย
โดยไม่มีเหตุผล

```
ตารางเปรียบเทียบ
┌──────────────────────────────┬───────────────────────┬────────────────────────┐
│ ประเด็น                       │ Signal → ทำตรงทันที    │ Signal → Celery Task    │
├──────────────────────────────┼───────────────────────┼────────────────────────┤
│ Request ของคนคอมเมนต์บล็อกไหม │ บล็อก (รอ DB+Redis)   │ ไม่บล็อก (ส่งเข้าคิว 2ms) │
│ ถ้า Redis/SMTP ล่มชั่วคราว     │ Comment ทั้งก้อนพัง    │ Retry อัตโนมัติ (Part076) │
│ Recipient หลายคน (เช่น mention หลายคน) │ ยิ่งช้าตามจำนวนคน │ กระจายงานหลาย task ขนานได้ │
└──────────────────────────────┴───────────────────────┴────────────────────────┘
```

**สถาปัตยกรรมที่ถูกต้องของ Part นี้**: Signal ทำหน้าที่แค่ **"สังเกตเห็นเหตุการณ์แล้ว
โยนงานเข้าคิวทันที"** (`.delay()` ใช้เวลาแค่ 1-2 มิลลิวินาที) ส่วนงานหนักทั้งหมด (เขียน
DB, เรียก Channel Layer, เช็ค preference, ส่งอีเมล) ย้ายไปทำใน Celery Worker ทั้งหมด

### 771.3 แผนภาพสถาปัตยกรรมเต็มรูปแบบ

```
┌─────────────┐  post_save   ┌──────────────┐  .delay()   ┌───────────────┐
│   Comment    │ ────────────>│ blog/signals │────────────>│ Celery Broker │
│  ถูกสร้าง     │  (Part 019)  │     .py      │  (Part 075) │  (Redis DB 4) │
└─────────────┘              └──────────────┘             └───────┬───────┘
                                                                   │ Worker หยิบงาน
                                                                   ▼
                                                     ┌─────────────────────────┐
                                                     │  Celery Task:            │
                                                     │  process_new_comment_    │
                                                     │  notification            │
                                                     │  1. เช็ค preference      │
                                                     │     (ขั้นตอนที่ 778)      │
                                                     │  2. สร้าง Notification    │
                                                     │     (ขั้นตอนที่ 772)      │
                                                     └────────────┬─────────────┘
                                                                  │
                              ┌───────────────────────────────────┼───────────────────────────┐
                              ▼                                   ▼                           ▼
                    ┌───────────────────┐             ┌─────────────────────┐    ┌──────────────────────┐
                    │ Channel Layer      │             │ (สะสมไว้รอ)          │    │ Web Push (ถ้าเปิดไว้)  │
                    │ group_send()       │             │ Celery Beat ทุกเช้า   │    │ pywebpush             │
                    │ (ขั้นตอนที่ 774)     │             │ ส่ง Email Digest      │    │ (ขั้นตอนที่ 777)        │
                    └─────────┬──────────┘             │ (ขั้นตอนที่ 776)      │    └──────────────────────┘
                              │ ถ้า user online          └──────────────────────┘
                              ▼
                    ┌───────────────────┐
                    │ WebSocket Consumer │
                    │ (realtime app)     │
                    └─────────┬──────────┘
                              │ ส่งผ่าน connection ที่เปิดค้างอยู่
                              ▼
                    ┌───────────────────┐
                    │ Browser: Alpine.js │
                    │ อัปเดต Badge ทันที  │
                    │ (ขั้นตอนที่ 775)     │
                    └───────────────────┘
```

จุดสำคัญที่สุดของแผนภาพนี้คือ **Celery Task เป็นจุดตัดสินใจกลาง (single decision
point)** — มันตัดสินใจว่าผู้รับควรได้รับการแจ้งเตือนหรือไม่ (เช็ค preference จาก
ขั้นตอนที่ 778) แล้วค่อยกระจายออกไปตามช่องทางที่เหมาะสม ไม่ใช่ให้แต่ละช่องทางไป
เช็คเงื่อนไขซ้ำกันเอง

### 771.4 App ใหม่ที่ต้องสร้าง: `notifications`

```bash
python manage.py startapp notifications
```

```python
# config/settings.py
INSTALLED_APPS = [
    # ...
    'blog',
    'accounts',
    'realtime',          # จาก Part 057/074 — เก็บ Consumer และ routing ของ Channels
    'notifications',     # ใหม่ใน Part นี้
]
```

`notifications` เป็นแอปที่**ไม่รู้จัก** `blog` หรือ `accounts` โดยตรง (import จาก
`blog.models` เท่าที่จำเป็นเท่านั้นภายใน task) ส่วน `realtime` (แอปที่เก็บ Consumer
มาตั้งแต่ Part 057/074) จะเพิ่ม Consumer ตัวใหม่สำหรับรับการแจ้งเตือนส่วนตัว โดยไม่
ต้องสร้างแอปแยกซ้ำซ้อน — ทวนหลักการแยกความรับผิดชอบ (separation of concerns) ที่
Part 019 ข้อ 184.1 วางไว้: `notifications` ดูแลเรื่อง "ข้อมูลและ logic การแจ้งเตือน"
ส่วน `realtime` ดูแลเรื่อง "ท่อส่งข้อความแบบ real-time" เท่านั้น

### 771.5 ตารางสรุปขั้นตอนที่ 771

| หัวข้อ | สรุป |
|---|---|
| ปัญหาหลัก | ต้องแจ้งเตือนผู้ใช้ทันทีโดยไม่บล็อก request ของคนที่ทำให้เกิดเหตุการณ์ |
| จุดตัดสินใจกลาง | Celery Task เดียวที่เช็ค preference แล้วกระจายไปยัง 3 ช่องทาง |
| ช่องทางที่ 1 | Channel Layer → WebSocket → Badge (real-time ตอน online) |
| ช่องทางที่ 2 | Celery Beat → Email Digest (สรุปตอน offline) |
| ช่องทางที่ 3 | Web Push API (แจ้งเตือนแม้ปิดแท็บ) |
| แอปใหม่ | `notifications` (data/logic) ทำงานร่วมกับ `realtime` (transport) ที่มีอยู่แล้ว |

---

## ขั้นตอนที่ 772: ออกแบบ `Notification` Model ด้วย Generic Foreign Key

### 772.1 ทำไมต้องใช้ Generic Foreign Key

การแจ้งเตือนแต่ละอันมัก "ชี้ไปหา" object ที่ต่างชนิดกัน: บางอันชี้ไปหา `Comment`
บางอันชี้ไปหา `Post` (เช่นมีคนกดถูกใจ) และในอนาคตอาจมี `Follow`, `Order`, หรือ model
อื่นที่ยังไม่ถูกสร้างขึ้นเลยด้วยซ้ำ ถ้าออกแบบด้วย `ForeignKey` ธรรมดา จะต้องมีคอลัมน์
แยกสำหรับทุกชนิด object ที่เป็นไปได้:

```python
# ❌ ออกแบบผิด — ต้องเพิ่มคอลัมน์ทุกครั้งที่มี object ชนิดใหม่ที่ต้องแจ้งเตือนได้
class Notification(models.Model):
    related_comment = models.ForeignKey(Comment, null=True, blank=True, ...)
    related_post = models.ForeignKey(Post, null=True, blank=True, ...)
    related_order = models.ForeignKey(Order, null=True, blank=True, ...)
    # เพิ่มเรื่อย ๆ ไม่มีที่สิ้นสุดทุกครั้งที่มีฟีเจอร์ใหม่ต้องแจ้งเตือน
```

Django มีเฟรมเวิร์กมาตรฐานสำหรับปัญหานี้อยู่แล้วคือ **Content Types Framework**
(`django.contrib.contenttypes` — ติดตั้งมาใน `INSTALLED_APPS` ตั้งแต่ Django สร้าง
โปรเจกต์ให้ครั้งแรกที่ Part 004) ซึ่งเก็บ "รายชื่อโมเดลทั้งหมดในระบบ" ไว้ในตาราง
`django_content_type` ทำให้เขียนความสัมพันธ์แบบ **"ชี้ไปที่ object ชนิดไหนก็ได้"**
ผ่าน **`GenericForeignKey`** ได้โดยใช้แค่ 2 คอลัมน์เสมอ ไม่ว่าจะมีโมเดลกี่ชนิดในระบบ

### 772.2 กลไกเบื้องหลัง `GenericForeignKey`

| คอลัมน์ | หน้าที่ |
|---|---|
| `target_content_type` (`ForeignKey` ไปยัง `ContentType`) | บอกว่า object ที่ชี้ไปเป็น **โมเดลชนิดไหน** (เช่น `blog.Comment` หรือ `blog.Post`) |
| `target_object_id` (`PositiveIntegerField`) | บอกว่าเป็น **แถวไหน** ของโมเดลนั้น (คือค่า primary key) |
| `target` (`GenericForeignKey`) | field เสมือนที่รวม 2 คอลัมน์ข้างบนเข้าด้วยกัน ใช้งานเหมือน `ForeignKey` ปกติตอนเขียนโค้ด (`notification.target`) แต่**ไม่ได้สร้างคอลัมน์เพิ่ม**ในฐานข้อมูล |

```
Notification row ตัวอย่าง:
┌────┬───────────┬──────────────────────┬─────────────────┬──────────────┐
│ id │ recipient │ target_content_type   │ target_object_id │ notification │
│    │           │ (ชี้ไป django_content │                  │ _type        │
│    │           │ _type row ของ Comment)│                  │              │
├────┼───────────┼──────────────────────┼─────────────────┼──────────────┤
│ 1  │ somchai   │ blog | comment        │ 42               │ new_comment  │
│ 2  │ somchai   │ blog | post           │ 7                 │ post_liked   │
└────┴───────────┴──────────────────────┴─────────────────┴──────────────┘
```

### 772.3 เขียน Model เต็มรูปแบบ

```python
# notifications/models.py
from django.conf import settings
from django.contrib.contenttypes.fields import GenericForeignKey
from django.contrib.contenttypes.models import ContentType
from django.db import models
from django.urls import reverse
from django.utils import timezone


class NotificationType(models.TextChoices):
    """
    ประเภทของการแจ้งเตือนทั้งหมดในระบบ — รวมศูนย์ไว้ที่เดียวเพื่อให้ signal,
    task, form ตั้งค่า opt-out (ขั้นตอนที่ 778), และ template ใช้ค่าเดียวกันเสมอ
    """
    NEW_COMMENT = 'new_comment', 'มีความคิดเห็นใหม่บนโพสต์ของคุณ'
    COMMENT_REPLY = 'comment_reply', 'มีคนตอบกลับความคิดเห็นของคุณ'
    POST_LIKED = 'post_liked', 'มีคนถูกใจโพสต์ของคุณ'


class Notification(models.Model):
    # ผู้รับ — เจ้าของ badge ที่จะเห็นการแจ้งเตือนนี้
    recipient = models.ForeignKey(
        settings.AUTH_USER_MODEL,
        on_delete=models.CASCADE,
        related_name='notifications',
    )
    # ผู้กระทำ — คนที่ทำให้เกิดเหตุการณ์ (อาจเป็น None ถ้า user ถูกลบไปแล้วภายหลัง)
    actor = models.ForeignKey(
        settings.AUTH_USER_MODEL,
        on_delete=models.SET_NULL,
        null=True,
        blank=True,
        related_name='+',   # ไม่ต้องการ reverse accessor ย้อนกลับ (ทวนจาก Part 012)
    )

    notification_type = models.CharField(max_length=20, choices=NotificationType.choices)
    verb = models.CharField(max_length=255)  # ข้อความสำเร็จรูปที่ render ไว้แล้ว เช่น "malee แสดงความคิดเห็น..."

    # Generic Foreign Key — ชี้ไปหา object ที่เกี่ยวข้อง (Comment/Post/อื่น ๆ ในอนาคต)
    target_content_type = models.ForeignKey(
        ContentType, on_delete=models.CASCADE, null=True, blank=True,
    )
    target_object_id = models.PositiveIntegerField(null=True, blank=True)
    target = GenericForeignKey('target_content_type', 'target_object_id')

    is_read = models.BooleanField(default=False)
    created_at = models.DateTimeField(auto_now_add=True)
    read_at = models.DateTimeField(null=True, blank=True)

    # ทวนแนวคิดจาก Part 076: ป้องกันการส่ง Email Digest ซ้ำสำหรับแจ้งเตือนเดิม
    digest_sent_at = models.DateTimeField(null=True, blank=True)

    class Meta:
        ordering = ['-created_at']
        indexes = [
            # ทวนหลักการจาก Part 070: index ที่ตรงกับ query ที่ใช้บ่อยที่สุด —
            # "ดึงรายการที่ยังไม่อ่านของ user คนหนึ่ง เรียงจากใหม่ไปเก่า"
            models.Index(
                fields=['recipient', 'is_read', '-created_at'],
                name='notif_recipient_unread_idx',
            ),
            # ใช้กับ query ของ Celery Beat digest (ขั้นตอนที่ 776):
            # "หาแจ้งเตือนที่ยังไม่เคยถูกส่งอีเมลสรุป"
            models.Index(
                fields=['recipient', 'digest_sent_at'],
                name='notif_recipient_digest_idx',
            ),
        ]

    def __str__(self):
        return f'[{self.get_notification_type_display()}] -> {self.recipient}'

    def mark_as_read(self):
        if not self.is_read:
            self.is_read = True
            self.read_at = timezone.now()
            self.save(update_fields=['is_read', 'read_at'])

    def get_target_url(self):
        """คำนวณ URL ที่ควรพาผู้ใช้ไปเมื่อกดที่การแจ้งเตือนนี้"""
        if self.target is None:
            return reverse('notifications:list')

        # Comment ไม่มีหน้าของตัวเอง ต้องพาไปที่โพสต์แล้ว anchor ไปที่ id ของคอมเมนต์
        target_model_name = self.target_content_type.model
        if target_model_name == 'comment':
            return f'{self.target.post.get_absolute_url()}#comment-{self.target.pk}'
        if target_model_name == 'post':
            return self.target.get_absolute_url()
        return reverse('notifications:list')
```

### 772.4 ทำไมเลือก `on_delete=models.SET_NULL` ให้ `actor` แต่ `CASCADE` ให้ `recipient`

ทวนตารางตัดสินใจจาก Part 012: **`recipient`** ใช้ `CASCADE` เพราะถ้า user ที่เป็น
เจ้าของการแจ้งเตือนถูกลบทั้งบัญชี ก็ไม่มีเหตุผลจะเก็บการแจ้งเตือนที่ไม่มีใครอ่านไว้
อีกต่อไป แต่ **`actor`** ใช้ `SET_NULL` เพราะถ้าคนที่ **ทำให้เกิด**การแจ้งเตือน (เช่น
คนที่มาคอมเมนต์) ลบบัญชีตัวเองไปภายหลัง **ประวัติการแจ้งเตือนของผู้รับควรยังอยู่**
(เพียงแต่ไม่รู้ว่า "ใคร" เป็นคนทำแล้ว) — การลบทิ้งทั้งแถวจะทำให้ผู้รับเสียประวัติการ
แจ้งเตือนของตัวเองไปอย่างไม่สมเหตุสมผล

### 772.5 Migration และทดสอบเบื้องต้นใน Shell

```bash
python manage.py makemigrations notifications
python manage.py migrate
```

```bash
python manage.py shell
```

```python
>>> from django.contrib.auth import get_user_model
>>> from django.contrib.contenttypes.models import ContentType
>>> from blog.models import Post
>>> from notifications.models import Notification, NotificationType
>>> User = get_user_model()
>>> somchai = User.objects.get(username='somchai')
>>> malee = User.objects.get(username='malee')
>>> post = Post.objects.filter(author=somchai).first()
>>> notification = Notification.objects.create(
...     recipient=somchai,
...     actor=malee,
...     notification_type=NotificationType.NEW_COMMENT,
...     verb=f'{malee.username} แสดงความคิดเห็นบนโพสต์ "{post.title}" ของคุณ',
...     target_content_type=ContentType.objects.get_for_model(post),
...     target_object_id=post.pk,
... )
>>> notification.target
<Post: หัวข้อโพสต์ตัวอย่าง>
>>> notification.get_target_url()
'/blog/หัวข้อโพสต์ตัวอย่าง/'
```

`ContentType.objects.get_for_model(post)` คือวิธีมาตรฐานในการหาแถว `ContentType`
ที่ตรงกับโมเดลของ instance ที่ส่งเข้าไป (Django cache ผลลัพธ์นี้ไว้ในหน่วยความจำให้
อัตโนมัติ ไม่ query ฐานข้อมูลซ้ำทุกครั้งที่เรียก)

### 772.6 ตารางสรุปขั้นตอนที่ 772

| หัวข้อ | สรุป |
|---|---|
| ปัญหาที่แก้ | Notification ต้องชี้ไปหา object ต่างชนิดกันได้โดยไม่เพิ่มคอลัมน์ทุกครั้ง |
| กลไกที่ใช้ | `GenericForeignKey` จาก `django.contrib.contenttypes` (2 คอลัมน์: content type + object id) |
| Field สำคัญ | `recipient` (CASCADE), `actor` (SET_NULL), `is_read`, `digest_sent_at` |
| Index ที่เพิ่ม | `(recipient, is_read, -created_at)` และ `(recipient, digest_sent_at)` ตรงกับ query จริงที่จะใช้ |
| Helper method | `mark_as_read()`, `get_target_url()` |

---

## ขั้นตอนที่ 773: Trigger การแจ้งเตือนผ่าน Signal แล้วส่งต่อเป็น Celery Task

### 773.1 เขียน Signal ใน `blog/signals.py`

ทวนโครงสร้างมาตรฐานจาก Part 019 ข้อ 184.3: เขียน receiver ไว้ใน `signals.py`
ของแอปที่เป็น**เจ้าของเหตุการณ์** (`blog` เป็นเจ้าของ `Comment` model) แล้วเชื่อมผ่าน
`AppConfig.ready()` — **ไม่ใช่เขียนใน `notifications` app** แม้ปลายทางจะเป็นแอปนั้น
ก็ตาม เพราะ `blog` คือแอปที่รู้จักเหตุการณ์ "มี Comment ใหม่" โดยตรง:

```python
# blog/signals.py (เพิ่มต่อจาก receiver อื่น ๆ ที่มีอยู่แล้วในแอปนี้)
from django.db.models.signals import post_save
from django.dispatch import receiver

from .models import Comment
from notifications.tasks import process_new_comment_notification


@receiver(post_save, sender=Comment, dispatch_uid='blog_trigger_comment_notification')
def trigger_comment_notification(sender, instance, created, **kwargs):
    """
    ทวนกฎเหล็กจาก Part 075 ข้อ 748: ห้ามส่ง Model instance เข้า Celery Task ตรง ๆ
    ส่งแค่ instance.pk (primary key) เท่านั้น เพราะ instance ที่ serialize เป็น JSON
    ไม่ได้ (Part 075 ข้อ 747) และข้อมูลอาจเก่าไปแล้วตอนที่ Worker หยิบงานไปทำจริง
    """
    if not created:
        return  # ไม่แจ้งเตือนตอนแก้ไขคอมเมนต์เดิม ทวนกฎ `if created:` จาก Part 019 ข้อ 183.3
    process_new_comment_notification.delay(instance.pk)
```

จุดสำคัญที่ต้องย้ำอีกครั้งแม้จะเรียนมาแล้วจาก Part 019 และ 075: `if created:` ป้องกัน
ไม่ให้ทุกครั้งที่มีคนแก้ไขคอมเมนต์เดิม (เช่นแก้คำผิด) สร้างการแจ้งเตือนซ้ำ และการส่ง
เฉพาะ `instance.pk` แทน `instance` ทั้งก้อน ทำให้ Celery รับส่งงานผ่าน JSON serializer
(ตั้งค่าไว้ตั้งแต่ Part 075 ข้อ 742.6) ได้โดยไม่มีปัญหา

### 773.2 เชื่อม Signal ใน `blog/apps.py`

```python
# blog/apps.py
from django.apps import AppConfig


class BlogConfig(AppConfig):
    default_auto_field = 'django.db.models.BigAutoField'
    name = 'blog'

    def ready(self):
        import blog.signals  # noqa: F401
```

### 773.3 เขียน Celery Task: `process_new_comment_notification`

Task นี้ทำ 3 อย่างตามลำดับ: (1) โหลดข้อมูลที่จำเป็นจาก `comment_id` ที่ได้รับมา
(2) ตัดสินใจว่าใครควรเป็นผู้รับ และ (3) สร้างแถว `Notification` — **ยังไม่แตะเรื่อง
WebSocket ในขั้นตอนนี้** (เพิ่มในขั้นตอนที่ 774 ถัดไป) เพื่อให้เห็นภาพทีละชั้นชัดเจน:

```python
# notifications/tasks.py
import logging

from celery import shared_task
from django.apps import apps
from django.contrib.contenttypes.models import ContentType

from .models import Notification, NotificationType
from .services import user_wants_realtime_notification

logger = logging.getLogger(__name__)


@shared_task(ignore_result=True)
def process_new_comment_notification(comment_id):
    """
    รับ comment_id (ไม่ใช่ Comment instance ตามกฎ Part 075 ข้อ 748) แล้วตัดสินใจว่า
    ควรสร้าง Notification ให้ใคร — เผื่อกรณีที่เป็นการตอบกลับ (reply) หรือคอมเมนต์
    บนโพสต์โดยตรง
    """
    Comment = apps.get_model('blog', 'Comment')
    try:
        comment = Comment.objects.select_related(
            'post', 'post__author', 'author', 'parent', 'parent__author',
        ).get(pk=comment_id)
    except Comment.DoesNotExist:
        # ทวนกฎจาก Part 075 ข้อ 749: comment อาจถูกลบไปแล้วก่อน Worker จะหยิบงาน
        # ไปทำ (เช่นโดน moderator ลบทันทีเพราะเป็นสแปม) — ไม่ใช่ bug ต้อง log ไว้เฉย ๆ
        logger.info('process_new_comment_notification: comment %s ไม่พบแล้ว', comment_id)
        return

    if comment.parent_id and comment.parent.author_id:
        # กรณีตอบกลับคอมเมนต์คนอื่น -> แจ้งเตือนเจ้าของคอมเมนต์แม่
        recipient = comment.parent.author
        notification_type = NotificationType.COMMENT_REPLY
        verb = f'{comment.author} ตอบกลับความคิดเห็นของคุณในโพสต์ "{comment.post.title}"'
        target = comment
    else:
        # กรณีคอมเมนต์บนโพสต์โดยตรง -> แจ้งเตือนเจ้าของโพสต์
        recipient = comment.post.author
        notification_type = NotificationType.NEW_COMMENT
        verb = f'{comment.author} แสดงความคิดเห็นบนโพสต์ "{comment.post.title}" ของคุณ'
        target = comment

    if recipient_id_matches_actor(recipient, comment.author):
        return  # ไม่แจ้งเตือนตัวเองเวลาคอมเมนต์/ตอบกลับคอมเมนต์ของตัวเอง

    if not user_wants_realtime_notification(recipient, notification_type):
        logger.info(
            'ผู้ใช้ %s ปิดการแจ้งเตือนประเภท %s ไว้ ข้ามการสร้าง Notification',
            recipient, notification_type,
        )
        return

    notification = Notification.objects.create(
        recipient=recipient,
        actor=comment.author,
        notification_type=notification_type,
        verb=verb,
        target_content_type=ContentType.objects.get_for_model(target),
        target_object_id=target.pk,
    )
    logger.info('สร้าง Notification #%s ให้ %s สำเร็จ', notification.pk, recipient)
    return notification.pk


def recipient_id_matches_actor(recipient, actor):
    return recipient.pk == actor.pk
```

### 773.4 ทำไมต้องเช็ค "ไม่แจ้งเตือนตัวเอง" ก่อนเช็ค preference เสมอ

ลำดับการเช็คในโค้ดข้างบนมีความหมาย: เช็ค **self-notification** ก่อนเสมอ แล้วค่อยเช็ค
**preference** เพราะเป็นเงื่อนไขที่ไม่เกี่ยวข้องกันคนละมิติ — การคอมเมนต์บนโพสต์ของ
ตัวเองไม่ควรสร้าง `Notification` เลยไม่ว่าจะตั้งค่า preference เป็นอย่างไร ในขณะที่
preference (ขั้นตอนที่ 778) ใช้ตัดสินใจเฉพาะกรณีที่ recipient เป็นคนละคนกับ actor
แล้วเท่านั้น สลับลำดับกันแล้วผลลัพธ์ทางตรรกะจะเหมือนกัน แต่การเช็คสิ่งที่ **ไม่ต้อง
query ฐานข้อมูล** (`recipient.pk == actor.pk`) ก่อนสิ่งที่ **ต้อง query** (อ่านค่า
`recipient.notification_preference`) ประหยัด query โดยไม่จำเป็นในกรณีคอมเมนต์ตัวเอง
ซึ่งเกิดขึ้นบ่อยกว่าที่คิด (เช่น เจ้าของบล็อกตอบคำถามในคอมเมนต์ของตัวเอง)

### 773.5 ทดสอบว่า Task ถูกเรียกจริงจากการสร้าง Comment

```bash
python manage.py runserver          # Terminal 1
celery -A config worker --loglevel=info   # Terminal 2 (ทวนจาก Part 075 ข้อ 744)
```

```python
python manage.py shell
```

```python
>>> from blog.models import Comment, Post
>>> post = Post.objects.filter(author__username='somchai').first()
>>> Comment.objects.create(post=post, author_id=2, content='บทความดีมากเลยครับ')
<Comment: ...>
```

ดู log ฝั่ง Terminal 2 ควรเห็น:

```
[INFO/ForkPoolWorker-1] Task notifications.tasks.process_new_comment_notification[...] received
[INFO/ForkPoolWorker-1] สร้าง Notification #1 ให้ somchai สำเร็จ
[INFO/ForkPoolWorker-1] Task ... succeeded in 0.045s
```

และ request ที่สร้าง Comment (ถ้าทำผ่าน view จริงแทน shell) จะได้ response กลับทันที
โดยไม่ต้องรอขั้นตอนทั้งหมดนี้ทำงานเสร็จก่อนเลย — นี่คือผลลัพธ์ที่ต้องการตามข้อ 771.2

### 773.6 ตารางสรุปขั้นตอนที่ 773

| หัวข้อ | สรุป |
|---|---|
| Signal อยู่ที่ | `blog/signals.py` (แอปเจ้าของเหตุการณ์ ทวน Part 019) เชื่อมผ่าน `blog/apps.py` |
| ส่งอะไรเข้า Task | `comment.pk` เท่านั้น ไม่ใช่ instance (ทวนกฎ Part 075 ข้อ 748) |
| Task ตัดสินใจ | ผู้รับคือใคร (parent author หรือ post author), ไม่แจ้งเตือนตัวเอง, เช็ค preference |
| ผลลัพธ์ของขั้นตอนนี้ | มีแถว `Notification` ในฐานข้อมูล — ยังไม่ส่ง real-time (ต่อในขั้นตอนที่ 774) |

---

## ขั้นตอนที่ 774: ส่งการแจ้งเตือนแบบ Real-time ผ่าน Channels WebSocket

### 774.1 เพิ่ม Helper `push_notification_realtime`

แยกโค้ดที่คุยกับ Channel Layer ไว้เป็นฟังก์ชันของตัวเอง เพื่อให้ทั้ง Celery Task
(ขั้นตอนนี้) และเทสต์ (ขั้นตอนที่ 779) เรียกใช้ตรง ๆ ได้โดยไม่ต้องผ่าน task wrapper:

```python
# notifications/realtime.py
from asgiref.sync import async_to_sync
from channels.layers import get_channel_layer


def notification_group_name(user_id):
    """ชื่อกลุ่มเฉพาะของผู้ใช้แต่ละคน — ทวน convention จาก Part 074 ข้อ 731.6
    (ต้องมี prefix ชัดเจนเพื่อไม่ให้ชนกับกลุ่มประเภทอื่น เช่น chat_ หรือ system_)"""
    return f'user_notifications_{user_id}'


def push_notification_realtime(notification):
    """
    เรียกจาก Celery Task (sync context) จึงต้องห่อ group_send (async) ด้วย
    async_to_sync — เทคนิคเดียวกับที่ Part 074 ข้อ 733 ใช้เรียก Channel Layer
    จากโค้ด synchronous
    """
    channel_layer = get_channel_layer()
    async_to_sync(channel_layer.group_send)(
        notification_group_name(notification.recipient_id),
        {
            'type': 'notification_message',  # ต้องตรงกับชื่อ method ใน Consumer เป๊ะ
            'notification': {
                'id': notification.pk,
                'notification_type': notification.notification_type,
                'verb': notification.verb,
                'created_at': notification.created_at.isoformat(),
                'target_url': notification.get_target_url(),
            },
        },
    )
```

**ทวนกลไกสำคัญจาก Part 057 ข้อ 565**: `type: 'notification_message'` ในดิกชันนารี
ที่ส่งเข้า `group_send()` คือชื่อ **method** ที่ Consumer ทุกตัวในกลุ่มนั้นต้องมี —
Channels แปลง underscore (`notification_message`) เป็นชื่อ method ตรงตัว ถ้าตั้งชื่อ
ผิดหรือ Consumer ไม่มี method นี้ Channels จะโยน error ทันทีตอน dispatch

### 774.2 เชื่อม Helper เข้ากับ Task จากขั้นตอนที่ 773

```python
# notifications/tasks.py (แก้ไขจากขั้นตอนที่ 773.3)
from .realtime import push_notification_realtime


@shared_task(ignore_result=True)
def process_new_comment_notification(comment_id):
    # ... โค้ดเดิมทั้งหมดจากขั้นตอนที่ 773.3 จนถึงการสร้าง notification ...

    notification = Notification.objects.create(
        recipient=recipient,
        actor=comment.author,
        notification_type=notification_type,
        verb=verb,
        target_content_type=ContentType.objects.get_for_model(target),
        target_object_id=target.pk,
    )
    logger.info('สร้าง Notification #%s ให้ %s สำเร็จ', notification.pk, recipient)

    push_notification_realtime(notification)   # เพิ่มบรรทัดนี้ — ส่ง real-time ทันที

    return notification.pk
```

จุดสำคัญ: `push_notification_realtime()` ทำงาน**ต่อจาก** `Notification.objects.create()`
สำเร็จแล้วเท่านั้น — ถ้าสลับลำดับ (ส่ง WebSocket ก่อนเขียน DB) แล้ว browser กด "mark
as read" กลับมาทันทีที่ได้รับ ข้อความ อาจเจอ race condition ที่ endpoint หา
`Notification` ที่ยังไม่ถูกเขียนลง DB จริงไม่เจอ

### 774.3 เขียน Consumer ต่อยอดจาก `AuthenticatedGroupConsumer` (Part 074 ข้อ 732.3)

เพิ่ม Consumer ใหม่ในไฟล์ `realtime/consumers.py` ที่มีอยู่แล้วจาก Part 057/074 —
ใช้ base class `AuthenticatedGroupConsumer` ที่สร้างไว้แล้ว ทำให้ไม่ต้องเขียนโค้ด
ตรวจสิทธิ์หรือจัดการ `joined_groups`/`disconnect()` ซ้ำเลยแม้แต่บรรทัดเดียว:

```python
# realtime/consumers.py (เพิ่มต่อท้ายไฟล์เดิมจาก Part 074)
from notifications.realtime import notification_group_name


class UserNotificationConsumer(AuthenticatedGroupConsumer):
    """
    Consumer ส่วนตัวของผู้ใช้แต่ละคน — ต่างจาก StaffNotificationConsumer ของ
    Part 074 ข้อ 732.4 (ที่แจ้งเตือนทีมงานทุกคนพร้อมกันผ่านกลุ่มเดียว) ตัวนี้ทุกคน
    เข้ากลุ่มของตัวเองเท่านั้น (`user_notifications_<id>`) จึงไม่มีทางเห็นแจ้งเตือน
    ของคนอื่นได้แม้แต่ทางทฤษฎี เพราะแต่ละคนอยู่คนละกลุ่มกันเสมอ
    """

    async def on_authorized_connect(self):
        user = self.scope['user']
        await self.join_group(notification_group_name(user.id))
        await self.send_json({'event': 'connected'})

    async def notification_message(self, event):
        """method นี้ถูกเรียกโดย Channels เมื่อมีข้อความ type='notification_message'
        เข้ามาในกลุ่มที่ instance นี้เป็นสมาชิกอยู่ (ทวนกลไกจาก Part 057 ข้อ 565.2)"""
        await self.send_json({'event': 'notification', 'notification': event['notification']})

    async def receive_json(self, content, **kwargs):
        action = content.get('action')
        if action == 'mark_read':
            await self._mark_read(content.get('notification_id'))
        elif action == 'mark_all_read':
            await self._mark_all_read()

    async def _mark_read(self, notification_id):
        from channels.db import database_sync_to_async
        from notifications.models import Notification

        updated_count = await database_sync_to_async(
            Notification.objects.filter(
                pk=notification_id, recipient=self.scope['user'],
            ).update
        )(is_read=True)
        await self.send_json({'event': 'marked_read', 'notification_id': notification_id, 'ok': bool(updated_count)})

    async def _mark_all_read(self):
        from channels.db import database_sync_to_async
        from notifications.models import Notification

        updated_count = await database_sync_to_async(
            Notification.objects.filter(recipient=self.scope['user'], is_read=False).update
        )(is_read=True)
        await self.send_json({'event': 'marked_all_read', 'count': updated_count})
```

**ทำไม `.update()` แทน loop `.save()` ทีละแถว**: ทวนหลักการจาก Part 013/014 — การ
`mark_all_read` อาจต้องอัปเดตหลายสิบแถวพร้อมกัน `.filter(...).update(is_read=True)`
ยิง SQL `UPDATE` เพียง**คำสั่งเดียว**ครอบคลุมทุกแถวที่ตรงเงื่อนไข เร็วกว่าการ
`.save()` ทีละ object มาก และไม่ยิง `post_save` signal ซ้ำ (ซึ่งในกรณีนี้ไม่มีใคร
ฟัง `post_save` ของ `Notification` อยู่แล้ว แต่เป็นพฤติกรรมที่ต้องรู้ไว้เสมอ)

### 774.4 เพิ่ม Route ใน `realtime/routing.py`

```python
# realtime/routing.py (เพิ่มต่อจาก path เดิมของ Part 057/074)
from django.urls import re_path

from . import consumers

websocket_urlpatterns = [
    # ... path เดิมของ Part 057/074 เช่น chat, echo ...
    re_path(r'^ws/notifications/mine/$', consumers.UserNotificationConsumer.as_asgi()),
]
```

ตั้งใจใช้ path `ws/notifications/mine/` แยกจาก `ws/notifications/` เดิมที่ Part
057/074 ใช้สำหรับแจ้งเตือนทีมงาน (staff) เรื่องคอมเมนต์ใหม่ทั้งระบบ — สอง Consumer นี้
**ทำหน้าที่ต่างกันโดยสิ้นเชิง** (staff เห็นทุกคอมเมนต์ vs. เจ้าของโพสต์เห็นเฉพาะของ
ตัวเอง) จึงต้องแยก path ให้ชัดเจน ไม่ใช้ path ซ้ำกัน

### 774.5 ทดสอบด้วยมือผ่าน Browser DevTools Console

```javascript
// เปิดหน้าเว็บที่ล็อกอินเป็น somchai แล้วเปิด DevTools Console
const ws = new WebSocket(`ws://${location.host}/ws/notifications/mine/`);
ws.onmessage = (event) => console.log('ได้รับ:', JSON.parse(event.data));
ws.onopen = () => console.log('เชื่อมต่อสำเร็จ');
```

จากนั้นเปิด terminal อีกอันสร้าง Comment บนโพสต์ของ `somchai` ผ่าน shell ตามข้อ
773.5 — Console ของ browser ควรแสดงข้อความทันที:

```javascript
ได้รับ: {event: "notification", notification: {id: 3, notification_type: "new_comment", verb: "malee แสดงความคิดเห็น...", ...}}
```

### 774.6 ตารางสรุปขั้นตอนที่ 774

| หัวข้อ | สรุป |
|---|---|
| Helper ใหม่ | `notifications/realtime.py` — `push_notification_realtime()` ห่อด้วย `async_to_sync` |
| Consumer ใหม่ | `UserNotificationConsumer` ใน `realtime/consumers.py` ต่อยอดจาก `AuthenticatedGroupConsumer` (Part 074) |
| ชื่อกลุ่ม | `user_notifications_<user_id>` — คนละกลุ่มต่อคน แยกจากกลุ่ม staff เดิม |
| Path ใหม่ | `ws/notifications/mine/` แยกจาก `ws/notifications/` ของ Part 057/074 |
| Action ที่รองรับจาก client | `mark_read`, `mark_all_read` ผ่าน `.update()` (ไม่ loop `.save()`) |

---

## ขั้นตอนที่ 775: UI Badge/Counter แจ้งเตือนด้วย Alpine.js และ HTMX

### 775.1 แบ่งหน้าที่ระหว่าง Alpine.js กับ HTMX ให้ชัดเจน

ทวนเครื่องมือจาก Phase 6: **HTMX** (Part 053) เก่งเรื่อง "ขอ HTML fragment จาก
server แล้วแทรกเข้าไปในหน้า" ส่วน **Alpine.js** (Part 054) เก่งเรื่อง "state ฝั่ง
client ที่ต้องอัปเดตแบบ reactive โดยไม่ขอ server ทุกครั้ง" — badge แจ้งเตือนต้องการ
ทั้งสองอย่างพร้อมกันคนละจุด:

| ส่วนของ UI | เครื่องมือที่เหมาะ | เหตุผล |
|---|---|---|
| ตัวเลขนับที่ต้องกระพริบทันทีที่มี WebSocket message เข้ามา | **Alpine.js** | เป็น state ในหน่วยความจำของ browser ล้วน ๆ ไม่ต้องขอ server ใหม่ทุกครั้งที่ตัวเลขเปลี่ยน |
| รายการแจ้งเตือนแบบละเอียดในกล่อง dropdown เมื่อกดเปิด | **HTMX** | เป็น HTML fragment ที่ render ฝั่ง server ได้ง่ายกว่า (ใช้ template ภาษาเดียวกับหน้าเว็บทั้งหมด) |
| ปุ่ม "ทำเครื่องหมายว่าอ่านแล้วทั้งหมด" | **HTMX** | เป็น action แบบครั้งเดียว (POST แล้วรับ HTML ใหม่กลับมา) ไม่ต้องมี state ซับซ้อน |

### 775.2 Context Processor: ส่งจำนวนที่ยังไม่อ่านให้ทุกหน้า

ก่อน WebSocket เชื่อมต่อสำเร็จ (เช่น ตอนโหลดหน้าเว็บครั้งแรก) badge ต้องมีค่าเริ่มต้น
ที่ถูกต้องอยู่แล้ว — ใช้ **context processor** (เทคนิคจาก Part 030) ส่งค่าเริ่มต้นนี้
เข้าไปในทุก template โดยไม่ต้องเขียนซ้ำในทุก view:

```python
# notifications/context_processors.py
def unread_notification_count(request):
    if not request.user.is_authenticated:
        return {'unread_notification_count': 0}
    return {
        'unread_notification_count': request.user.notifications.filter(is_read=False).count(),
    }
```

```python
# config/settings.py
TEMPLATES = [
    {
        # ...
        'OPTIONS': {
            'context_processors': [
                # ... context processor เดิมจาก Part 030 ...
                'notifications.context_processors.unread_notification_count',
            ],
        },
    },
]
```

### 775.3 Template: Badge + Alpine.js Component

```html
<!-- templates/notifications/_bell.html -->
<div
    x-data="notificationBell({{ unread_notification_count }})"
    x-init="connect()"
    class="notification-bell"
>
    <button @click="open = !open" class="bell-button" aria-label="การแจ้งเตือน">
        🔔
        <span x-show="count > 0" x-text="count" x-cloak class="badge-count"></span>
    </button>

    <div
        x-show="open"
        @click.outside="open = false"
        x-cloak
        class="notification-dropdown"
        hx-get="{% url 'notifications:dropdown_list' %}"
        hx-trigger="intersect once"
        hx-target="this"
        hx-swap="innerHTML"
    >
        กำลังโหลด...
    </div>
</div>

<script src="{% static 'js/notifications.js' %}"></script>
```

จุดที่น่าสังเกต: `hx-trigger="intersect once"` ทำให้ HTMX ยิง request ขอรายการ
แจ้งเตือนแบบละเอียด **เฉพาะตอนที่ dropdown ปรากฏบนจอครั้งแรก** (`intersect` มาจาก
Intersection Observer API) ไม่ใช่ทุกครั้งที่หน้าเว็บโหลด — ประหยัด query ฐานข้อมูล
สำหรับผู้ใช้ที่ไม่เคยกดเปิด dropdown เลย

### 775.4 Alpine.js Component: เชื่อม WebSocket เข้ากับ State

ใช้ `ResilientWebSocket` ที่เขียนไว้แล้วใน Part 074 ข้อ 734.4 (`static/js/
websocket-client.js`) แทนที่จะเปิด `new WebSocket()` ตรง ๆ เพื่อได้ประโยชน์จาก
exponential backoff + jitter ที่ทำไว้แล้วโดยไม่ต้องเขียนซ้ำ:

```javascript
// static/js/notifications.js
document.addEventListener('alpine:init', () => {
    Alpine.data('notificationBell', (initialCount) => ({
        count: initialCount,
        open: false,
        socket: null,

        connect() {
            this.socket = new ResilientWebSocket(
                `${location.protocol === 'https:' ? 'wss' : 'ws'}://${location.host}/ws/notifications/mine/`,
            );

            this.socket.onmessage = (event) => {
                const data = JSON.parse(event.data);
                if (data.event === 'notification') {
                    this.count += 1;
                    this.showToast(data.notification);
                } else if (data.event === 'marked_all_read') {
                    this.count = 0;
                }
            };

            this.socket.connect();
        },

        showToast(notification) {
            // แสดง toast แจ้งเตือนสั้น ๆ มุมจอ — ใช้ library toast ใดก็ได้ที่โปรเจกต์เลือกใช้
            const toast = document.createElement('div');
            toast.className = 'notification-toast';
            toast.textContent = notification.verb;
            toast.onclick = () => { window.location.href = notification.target_url; };
            document.body.appendChild(toast);
            setTimeout(() => toast.remove(), 5000);
        },

        markAllRead() {
            if (this.socket && this.socket.socket && this.socket.socket.readyState === WebSocket.OPEN) {
                this.socket.socket.send(JSON.stringify({ action: 'mark_all_read' }));
            }
        },
    }));
});
```

**สังเกตว่า `count` เพิ่มขึ้นทันทีที่ได้รับ WebSocket message โดยไม่ต้องขอ server
ใหม่เลย** — นี่คือจุดแข็งของ Alpine.js ที่ทำให้ badge รู้สึก "สด" ทันทีจริง ๆ
ต่างจากการใช้ `hx-trigger="every 10s"` ของ HTMX มา poll ถามทุก 10 วินาที ซึ่งมี
ความหน่วงเฉลี่ยครึ่งหนึ่งของ interval เสมอ (5 วินาทีโดยเฉลี่ยในตัวอย่างนี้)

### 775.5 Template: รายการ Dropdown (HTMX Fragment)

```html
<!-- templates/notifications/_dropdown_list.html -->
<div class="notification-list">
    {% for notification in notifications %}
        <a
            href="{{ notification.get_target_url }}"
            class="notification-item {% if not notification.is_read %}unread{% endif %}"
        >
            <p>{{ notification.verb }}</p>
            <time>{{ notification.created_at|timesince }} ที่แล้ว</time>
        </a>
    {% empty %}
        <p class="empty-state">ยังไม่มีการแจ้งเตือน</p>
    {% endfor %}

    {% if notifications %}
        <button
            hx-post="{% url 'notifications:mark_all_read' %}"
            hx-target="closest .notification-list"
            hx-swap="outerHTML"
            @click="count = 0"
        >
            ทำเครื่องหมายว่าอ่านแล้วทั้งหมด
        </button>
    {% endif %}
</div>
```

### 775.6 View ที่รองรับ Fragment ทั้งสองตัว

```python
# notifications/views.py
from django.contrib.auth.decorators import login_required
from django.shortcuts import render
from django.views.decorators.http import require_POST


@login_required
def dropdown_list(request):
    notifications = request.user.notifications.select_related(
        'actor', 'target_content_type',
    )[:10]
    return render(request, 'notifications/_dropdown_list.html', {'notifications': notifications})


@login_required
@require_POST
def mark_all_read(request):
    request.user.notifications.filter(is_read=False).update(is_read=True)
    return dropdown_list(request)   # คืน fragment เดิม แต่ตอนนี้ทุกอันกลายเป็น "อ่านแล้ว"
```

```python
# notifications/urls.py
from django.urls import path

from . import views

app_name = 'notifications'
urlpatterns = [
    path('dropdown/', views.dropdown_list, name='dropdown_list'),
    path('mark-all-read/', views.mark_all_read, name='mark_all_read'),
]
```

`@click="count = 0"` ในข้อ 775.5 ทำงาน**คู่ขนาน**กับ `hx-post` — HTMX จัดการเรื่อง
"อัปเดต HTML ของรายการ" ส่วน Alpine (`@click`) จัดการเรื่อง "อัปเดตตัวเลข badge"
พร้อมกันในคลิกเดียว โดยไม่ต้องรอ HTMX request เสร็จก่อนถึงจะอัปเดตตัวเลขได้ (ให้
ความรู้สึกตอบสนองทันที) นี่คือตัวอย่างที่ชัดเจนที่สุดของการใช้ HTMX และ Alpine.js
**ร่วมกัน** แทนที่จะเลือกใช้แค่ตัวใดตัวหนึ่ง

### 775.7 ตารางสรุปขั้นตอนที่ 775

| หัวข้อ | สรุป |
|---|---|
| ค่าเริ่มต้นก่อน WebSocket เชื่อมต่อ | Context processor คำนวณ `unread_notification_count` ให้ทุกหน้า |
| ตัวเลข badge แบบ real-time | Alpine.js state (`count`) อัปเดตทันทีที่ WebSocket ส่งข้อความมา |
| รายการละเอียดใน dropdown | HTMX fragment โหลดแบบ lazy (`hx-trigger="intersect once"`) |
| ปุ่ม mark-all-read | HTMX อัปเดต HTML + Alpine อัปเดตตัวเลข badge พร้อมกันในคลิกเดียว |
| ที่มาของ `ResilientWebSocket` | ใช้ซ้ำจาก Part 074 ข้อ 734.4 ไม่ต้องเขียน reconnect logic ใหม่ |

---

## ขั้นตอนที่ 776: Email Digest — รวมการแจ้งเตือนส่งสรุปด้วย Celery Beat

### 776.1 ทำไมต้องมี Digest แทนที่จะส่งอีเมลทันทีทุกครั้ง

ถ้าส่งอีเมลทันทีทุกครั้งที่มีการแจ้งเตือน (แบบเดียวกับ real-time WebSocket) ผู้ใช้ที่
โพสต์ยอดนิยมจะได้รับอีเมลนับสิบฉบับต่อวัน ซึ่งสร้างความรำคาญมากกว่าประโยชน์
(ปรากฏการณ์ที่เรียกว่า **notification fatigue**) แนวทางที่ระบบระดับมืออาชีพใช้กันคือ
**สะสมการแจ้งเตือนไว้ แล้วส่งเป็นอีเมลสรุปครั้งเดียวตามรอบเวลา** (เช่น วันละครั้ง)
ทวนจาก Part 076 ข้อ 751.2: นี่คืองานที่เหมาะกับ **Celery Beat** เพราะเป็นส่วนหนึ่งของ
business logic โดยตรง ไม่ใช่งานดูแลระบบเซิร์ฟเวอร์แบบที่ cron ของ OS เหมาะกว่า

### 776.2 Query หาแจ้งเตือนที่ต้องรวมในรอบ Digest ถัดไป

```python
# notifications/services.py
from django.contrib.auth import get_user_model
from django.db.models import Count

from .models import Notification

User = get_user_model()


def get_users_with_pending_digest():
    """
    หา user ทุกคนที่มีแจ้งเตือนค้างส่ง digest อย่างน้อย 1 รายการ — ใช้ index
    (recipient, digest_sent_at) ที่ออกแบบไว้ตั้งแต่ขั้นตอนที่ 772.3
    """
    return (
        User.objects.filter(
            notifications__digest_sent_at__isnull=True,
            notification_preference__email_digest_enabled=True,
        )
        .annotate(pending_count=Count('notifications'))
        .distinct()
    )


def get_pending_digest_notifications(user):
    return user.notifications.filter(digest_sent_at__isnull=True).select_related(
        'actor', 'target_content_type',
    )
```

### 776.3 Template อีเมลสรุป

```html
<!-- templates/notifications/email/digest.html -->
<h2>สรุปการแจ้งเตือนของคุณ — {{ notifications|length }} รายการ</h2>
<ul>
    {% for notification in notifications %}
        <li>
            {{ notification.verb }}
            <br>
            <small>{{ notification.created_at|date:"d/m/Y H:i" }}</small>
        </li>
    {% endfor %}
</ul>
<p>
    <a href="{{ site_url }}{% url 'notifications:list' %}">ดูการแจ้งเตือนทั้งหมด</a>
    |
    <a href="{{ site_url }}{% url 'notifications:preferences' %}">ตั้งค่าการแจ้งเตือน</a>
</p>
```

### 776.4 Celery Task: `send_notification_digest`

```python
# notifications/tasks.py (เพิ่มต่อจาก task เดิม)
from django.conf import settings
from django.core.mail import send_mail
from django.template.loader import render_to_string
from django.utils import timezone
from django.utils.html import strip_tags

from .services import get_pending_digest_notifications, get_users_with_pending_digest


@shared_task(ignore_result=True)
def send_notification_digest():
    """
    ทวนโครงสร้างจาก Part 076 ข้อ 751 — task นี้ถูกเรียกโดย Celery Beat เท่านั้น
    ไม่มีที่ไหนในโค้ดเรียก .delay() ตรง ๆ (ยกเว้นตอนเทส) เพราะเป็นงานตามตารางเวลา
    """
    sent_count = 0
    for user in get_users_with_pending_digest():
        pending = list(get_pending_digest_notifications(user))
        if not pending:
            continue

        html_message = render_to_string('notifications/email/digest.html', {
            'notifications': pending,
            'site_url': settings.SITE_URL,
        })
        send_mail(
            subject=f'สรุปการแจ้งเตือน {len(pending)} รายการจาก Blog ของเรา',
            message=strip_tags(html_message),
            html_message=html_message,
            from_email=settings.DEFAULT_FROM_EMAIL,
            recipient_list=[user.email],
        )

        # ทำเครื่องหมายว่า "ถูกส่งไปแล้ว" ป้องกันการส่งซ้ำในรอบถัดไป (ทวน
        # แนวคิด idempotency จาก Part 076 ข้อ 758)
        notification_ids = [n.pk for n in pending]
        Notification.objects.filter(pk__in=notification_ids).update(digest_sent_at=timezone.now())
        sent_count += 1

    return {'digests_sent': sent_count}
```

**จุดสำคัญเรื่อง idempotency**: ถ้า task นี้ถูกเรียกซ้ำโดยไม่ได้ตั้งใจ (เช่น Celery
Beat schedule ผิดพลาดยิงซ้ำ) ผู้ใช้จะ**ไม่ได้รับอีเมลซ้ำ** เพราะรอบที่สองจะหาแจ้งเตือน
ที่ `digest_sent_at__isnull=True` ไม่เจอเลย (ทุกอันถูกทำเครื่องหมายไปแล้วในรอบแรก)
— ทวนหลักการ idempotency ที่ Part 076 ข้อ 758 เน้นย้ำว่าสำคัญที่สุดสำหรับ task ที่มี
ผลข้างเคียงกับโลกภายนอก (ในที่นี้คือการส่งอีเมลจริง)

### 776.5 ตั้งตารางเวลาใน `CELERY_BEAT_SCHEDULE`

```python
# config/settings.py
from celery.schedules import crontab

CELERY_BEAT_SCHEDULE = {
    # ... schedule เดิมจาก Part 076 ...
    'send-notification-digest-daily': {
        'task': 'notifications.tasks.send_notification_digest',
        'schedule': crontab(hour=7, minute=0),   # ทุกวันเวลา 07:00 ตามเวลาเซิร์ฟเวอร์
    },
}
```

```bash
# รัน Celery Beat คู่กับ Worker (Terminal ที่ 3 — ทวนจาก Part 076 ข้อ 751.4)
celery -A config beat --loglevel=info
```

### 776.6 ตารางเปรียบเทียบ: ความถี่ Digest ที่เลือกได้

| ความถี่ | `crontab(...)` | เหมาะกับ |
|---|---|---|
| ทันที (ไม่ต้อง digest) | ไม่ใช้ Beat เลย ส่งจาก task ตรง ๆ | แพลตฟอร์มที่ notification สำคัญมาก (เช่น แจ้งเตือนความปลอดภัย) |
| รายชั่วโมง | `crontab(minute=0)` | แพลตฟอร์มที่มี traffic สูง ต้องการความสดพอสมควร |
| **รายวัน (เลือกใช้ใน Part นี้)** | `crontab(hour=7, minute=0)` | บล็อกทั่วไปที่ไม่ได้เร่งด่วน เหมาะกับอ่านตอนเช้า |
| รายสัปดาห์ | `crontab(hour=7, minute=0, day_of_week=1)` | สรุปภาพรวมสำหรับแพลตฟอร์มที่ traffic ต่ำ |

### 776.7 ตารางสรุปขั้นตอนที่ 776

| หัวข้อ | สรุป |
|---|---|
| ปัญหาที่แก้ | ป้องกัน notification fatigue จากการส่งอีเมลทันทีทุกครั้ง |
| กลไก | `digest_sent_at` field ทำเครื่องหมายว่า "ถูกรวมส่งไปแล้ว" |
| Task | `send_notification_digest` เรียกจาก Celery Beat เท่านั้น |
| Schedule | `crontab(hour=7, minute=0)` — ทุกวันตอนเช้า |
| Idempotency | เรียกซ้ำได้อย่างปลอดภัย เพราะ query ใหม่จะไม่เจอแจ้งเตือนที่ส่งไปแล้ว |

---

## ขั้นตอนที่ 777: เกริ่น Web Push Notification สำหรับแจ้งเตือนแม้ปิดแท็บ

### 777.1 ข้อจำกัดของ WebSocket ที่ Web Push แก้ได้

WebSocket (ขั้นตอนที่ 774) ทำงานได้ดีเยี่ยม **ตราบใดที่แท็บเปิดอยู่** — ทันทีที่ผู้ใช้
ปิดแท็บหรือปิดเบราว์เซอร์ connection ก็ตายไปด้วย ไม่มีทางส่งอะไรถึงเขาได้อีกจนกว่าจะ
เปิดหน้าเว็บใหม่ **Web Push API** ของเบราว์เซอร์แก้ปัญหานี้ได้ เพราะทำงานผ่าน
**Service Worker** ซึ่งเป็นสคริปต์ที่เบราว์เซอร์รันแยกต่างหากจากหน้าเว็บ **แม้ไม่มี
แท็บเปิดอยู่เลยก็ยังรับ push message ได้** (ตราบใดที่เบราว์เซอร์ยังเปิดอยู่ในเครื่อง
หรือแม้แต่บางระบบปฏิบัติการยังส่งต่อได้แม้เบราว์เซอร์ปิดสนิท)

```
ตารางเปรียบเทียบ 3 ช่องทางที่ Part นี้สร้างขึ้น
┌──────────────────┬──────────────────┬──────────────────┬──────────────────────┐
│ ช่องทาง            │ ต้องเปิดแท็บไหม    │ ความเร็ว          │ ใช้ตอนไหน              │
├──────────────────┼──────────────────┼──────────────────┼──────────────────────┤
│ WebSocket (774)   │ ต้อง              │ ทันที (< 1 วิ)     │ ผู้ใช้ online อยู่พอดี   │
│ Web Push (777)    │ ไม่ต้อง (Service   │ ทันที (< 1 วิ)     │ ผู้ใช้ปิดแท็บ/เบราว์เซอร์ │
│                    │ Worker ทำงานเบื้องหลัง)│               │ แต่เปิด permission ไว้ │
│ Email Digest (776)│ ไม่ต้อง            │ ล่าช้าถึง 1 วัน    │ สรุปภาพรวมไม่เร่งด่วน  │
└──────────────────┴──────────────────┴──────────────────┴──────────────────────┘
```

> **ขอบเขตของขั้นตอนนี้**: Web Push เป็นหัวข้อที่ลึกพอจะเป็น Part เต็มได้เอง (มีเรื่อง
> VAPID key rotation, payload encryption ตามสเปก, การจัดการ subscription หมดอายุ
> ข้าม browser vendor ที่พฤติกรรมต่างกัน) Part นี้จะสร้าง **เวอร์ชันที่ทำงานได้จริง
> ครบ flow** ตั้งแต่ subscribe ถึงส่งจริง แต่ไม่ลงลึกเรื่อง edge case ระดับ production
> เต็มรูปแบบ

### 777.2 ติดตั้ง `pywebpush` และสร้าง VAPID Keys

```bash
pip install pywebpush
pip freeze | grep -i pywebpush >> requirements.txt
```

**VAPID** (Voluntary Application Server Identification) คือกลไกที่ push service
ของแต่ละเบราว์เซอร์ (Google, Mozilla, ...) ใช้ยืนยันว่าเซิร์ฟเวอร์ของเราเป็นผู้ส่ง
จริง ต้องสร้างคู่กุญแจ (public/private key) ไว้ **ครั้งเดียวต่อโปรเจกต์**:

```bash
python -c "from py_vapid import Vapid02; v = Vapid02(); v.generate_keys(); print(v.private_pem()); print(v.public_key)"
```

```python
# config/settings.py
VAPID_PRIVATE_KEY = os.environ.get('VAPID_PRIVATE_KEY')
VAPID_PUBLIC_KEY = os.environ.get('VAPID_PUBLIC_KEY')
VAPID_CLAIM_EMAIL = 'mailto:admin@example.com'
```

**ห้ามใส่ private key ลง Git โดยเด็ดขาด** — เก็บใน environment variable เสมอ (ทวน
หลักการจาก Part 009 เรื่อง `.env` และ `.gitignore`)

### 777.3 Model เก็บ Push Subscription

```python
# notifications/models.py (เพิ่มต่อจาก Notification)
class PushSubscription(models.Model):
    """
    เก็บข้อมูลที่เบราว์เซอร์คืนมาตอน subscribe — endpoint คือ URL เฉพาะของ
    push service ผู้ให้บริการนั้น ๆ (Google FCM, Mozilla, ...) ที่เราต้องยิง
    ข้อความไปหา ส่วน p256dh/auth คือ public key สำหรับเข้ารหัส payload
    """
    user = models.ForeignKey(
        settings.AUTH_USER_MODEL, on_delete=models.CASCADE, related_name='push_subscriptions',
    )
    endpoint = models.URLField(max_length=500, unique=True)
    p256dh = models.CharField(max_length=255)
    auth = models.CharField(max_length=255)
    created_at = models.DateTimeField(auto_now_add=True)
```

```bash
python manage.py makemigrations notifications
python manage.py migrate
```

### 777.4 Service Worker: รับ Push Event

Service Worker ต้องถูก serve จาก **root scope** (`/sw.js`) ไม่ใช่จาก `/static/...`
เพราะ scope ของ Service Worker ครอบคลุมเฉพาะ path ที่มันถูก serve มาหรือลึกกว่านั้น
เท่านั้น — ถ้า serve จาก `/static/js/sw.js` มันจะควบคุมได้แค่หน้าที่อยู่ใต้
`/static/js/` ซึ่งไม่มีประโยชน์เลย จึงต้องเปิด view เฉพาะให้ Django serve ไฟล์นี้
จาก root:

```python
# notifications/views.py (เพิ่มต่อจากเดิม)
from django.views.generic import TemplateView


class ServiceWorkerView(TemplateView):
    template_name = 'notifications/sw.js'
    content_type = 'application/javascript'
```

```python
# config/urls.py
from notifications.views import ServiceWorkerView

urlpatterns = [
    # ...
    path('sw.js', ServiceWorkerView.as_view(), name='service_worker'),
]
```

```javascript
// templates/notifications/sw.js
self.addEventListener('push', (event) => {
    const data = event.data.json();
    event.waitUntil(
        self.registration.showNotification(data.title, {
            body: data.body,
            icon: '/static/img/notification-icon.png',
            data: { url: data.url },
        }),
    );
});

self.addEventListener('notificationclick', (event) => {
    event.notification.close();
    event.waitUntil(clients.openWindow(event.notification.data.url));
});
```

### 777.5 JavaScript ฝั่ง Client: ขอ Permission และ Subscribe

```javascript
// static/js/push.js
async function subscribeToPush(vapidPublicKey) {
    if (!('serviceWorker' in navigator) || !('PushManager' in window)) {
        console.warn('เบราว์เซอร์นี้ไม่รองรับ Web Push');
        return;
    }

    const registration = await navigator.serviceWorker.register('/sw.js');
    const permission = await Notification.requestPermission();
    if (permission !== 'granted') return;

    const subscription = await registration.pushManager.subscribe({
        userVisibleOnly: true,
        applicationServerKey: urlBase64ToUint8Array(vapidPublicKey),
    });

    await fetch('/notifications/push/subscribe/', {
        method: 'POST',
        headers: {
            'Content-Type': 'application/json',
            'X-CSRFToken': getCookie('csrftoken'),   // ทวนจาก Part 053 ข้อ 522.2
        },
        body: JSON.stringify(subscription.toJSON()),
    });
}

function urlBase64ToUint8Array(base64String) {
    const padding = '='.repeat((4 - (base64String.length % 4)) % 4);
    const base64 = (base64String + padding).replace(/-/g, '+').replace(/_/g, '/');
    const rawData = window.atob(base64);
    return Uint8Array.from([...rawData].map((char) => char.charCodeAt(0)));
}
```

### 777.6 View บันทึก Subscription

```python
# notifications/views.py (เพิ่มต่อ)
import json

from django.http import JsonResponse
from django.views.decorators.csrf import csrf_protect

from .models import PushSubscription


@login_required
@require_POST
@csrf_protect
def push_subscribe(request):
    data = json.loads(request.body)
    PushSubscription.objects.update_or_create(
        endpoint=data['endpoint'],
        defaults={
            'user': request.user,
            'p256dh': data['keys']['p256dh'],
            'auth': data['keys']['auth'],
        },
    )
    return JsonResponse({'status': 'subscribed'})
```

### 777.7 Celery Task: ส่ง Web Push จริง

```python
# notifications/tasks.py (เพิ่มต่อ)
import json

from django.conf import settings
from pywebpush import WebPushException, webpush

from .models import PushSubscription


@shared_task(ignore_result=True)
def send_web_push(notification_id):
    try:
        notification = Notification.objects.select_related('recipient').get(pk=notification_id)
    except Notification.DoesNotExist:
        return

    payload = json.dumps({
        'title': 'มีการแจ้งเตือนใหม่',
        'body': notification.verb,
        'url': notification.get_target_url(),
    })

    for subscription in notification.recipient.push_subscriptions.all():
        try:
            webpush(
                subscription_info={
                    'endpoint': subscription.endpoint,
                    'keys': {'p256dh': subscription.p256dh, 'auth': subscription.auth},
                },
                data=payload,
                vapid_private_key=settings.VAPID_PRIVATE_KEY,
                vapid_claims={'sub': settings.VAPID_CLAIM_EMAIL},
            )
        except WebPushException as exc:
            if exc.response is not None and exc.response.status_code == 410:
                # 410 Gone หมายถึง subscription หมดอายุถาวรแล้ว (ผู้ใช้ถอน permission
                # หรือเปลี่ยนเบราว์เซอร์) — ลบทิ้งเพื่อไม่ให้พยายามส่งซ้ำไปเรื่อย ๆ
                subscription.delete()
            else:
                logger.warning('ส่ง Web Push ไม่สำเร็จ: %s', exc)
```

เรียก task นี้จากจุดเดียวกับที่เรียก `push_notification_realtime()` ในขั้นตอนที่
774.2 (เพิ่มเงื่อนไขว่าผู้ใช้เปิด `web_push_enabled` ไว้หรือไม่ — รายละเอียดเต็มอยู่
ในขั้นตอนที่ 778 ถัดไป)

### 777.8 ตารางสรุปขั้นตอนที่ 777

| หัวข้อ | สรุป |
|---|---|
| เครื่องมือหลัก | Web Push API + Service Worker (ฝั่ง browser), `pywebpush` (ฝั่ง server) |
| กุญแจที่ต้องสร้าง | คู่ VAPID key (public เก็บใน JS, private เก็บใน env var เท่านั้น) |
| Model ใหม่ | `PushSubscription` (endpoint + p256dh + auth ต่อผู้ใช้แต่ละคน) |
| จุดที่ต้อง serve จาก root | `/sw.js` (ไม่ใช่ `/static/...`) เพราะ Service Worker scope |
| การจัดการ subscription หมดอายุ | ตรวจ HTTP 410 จาก push service แล้วลบ `PushSubscription` ทิ้ง |

---

## ขั้นตอนที่ 778: ระบบตั้งค่าการแจ้งเตือนของผู้ใช้ (Opt-out รายประเภท)

### 778.1 ออกแบบ Model การตั้งค่า

ทวนแนวคิด auto-create จาก Part 019 ข้อ 183: ทุก user ควรมี `NotificationPreference`
ติดตัวมาอัตโนมัติทันทีที่สมัครสมาชิก เหมือนที่ `Profile` ถูกสร้างอัตโนมัติ:

```python
# notifications/models.py (เพิ่มต่อ)
class NotificationPreference(models.Model):
    """
    ใช้ muted_types (JSONField เก็บ list ของ notification_type ที่ผู้ใช้ปิดไว้)
    แทนการสร้างคอลัมน์ Boolean แยกทีละประเภท — เพิ่มประเภทแจ้งเตือนใหม่ในอนาคต
    (ขั้นตอนที่ 772.3 อาจเพิ่ม NotificationType ใหม่ได้เรื่อย ๆ) โดยไม่ต้องทำ
    migration เพิ่มคอลัมน์ทุกครั้ง
    """
    user = models.OneToOneField(
        settings.AUTH_USER_MODEL, on_delete=models.CASCADE, related_name='notification_preference',
    )
    muted_types = models.JSONField(default=list, blank=True)
    email_digest_enabled = models.BooleanField(default=True)
    web_push_enabled = models.BooleanField(default=False)

    def wants_realtime(self, notification_type):
        return notification_type not in self.muted_types

    def wants_email_digest(self, notification_type):
        return self.email_digest_enabled and notification_type not in self.muted_types

    def wants_web_push(self, notification_type):
        return self.web_push_enabled and notification_type not in self.muted_types
```

```bash
python manage.py makemigrations notifications
python manage.py migrate
```

### 778.2 Auto-create ด้วย Signal (ทวนแบบเป๊ะจาก Part 019 ข้อ 183.4)

```python
# accounts/signals.py (เพิ่มต่อจาก create_profile/save_profile ที่มีอยู่แล้ว)
from notifications.models import NotificationPreference


@receiver(post_save, sender=settings.AUTH_USER_MODEL, dispatch_uid='accounts_create_notification_preference')
def create_notification_preference(sender, instance, created, **kwargs):
    if created:
        NotificationPreference.objects.create(user=instance)
```

รูปแบบนี้เหมือนกับ `create_profile` ทุกประการ — เพิ่ม receiver ใหม่ในไฟล์
`signals.py` เดิมของ `accounts` โดยไม่ต้องแก้อะไรใน `AppConfig.ready()` เพิ่มเติม
(import `accounts.signals` ครั้งเดียวครอบคลุมทุก receiver ในไฟล์นั้นอยู่แล้ว)

### 778.3 Helper สำหรับ Task ตรวจสอบ Preference (เติมจากขั้นตอนที่ 773)

```python
# notifications/services.py (เพิ่มต่อ)
def user_wants_realtime_notification(user, notification_type):
    preference, _ = NotificationPreference.objects.get_or_create(user=user)
    return preference.wants_realtime(notification_type)


def user_wants_web_push(user, notification_type):
    preference, _ = NotificationPreference.objects.get_or_create(user=user)
    return preference.wants_web_push(notification_type)
```

**ทำไมใช้ `get_or_create` แทนที่จะเชื่อ signal เพียงอย่างเดียว**: แม้ signal ใน
ข้อ 778.2 จะสร้าง `NotificationPreference` ให้ทุก user ใหม่แล้วก็ตาม แต่ user ที่
สมัครไว้ **ก่อน** ที่ฟีเจอร์นี้ถูก deploy จะไม่มี `NotificationPreference` อยู่เลย
`get_or_create` เป็นเกราะป้องกันชั้นที่สองที่รับประกันว่าโค้ดจะไม่พังด้วย
`RelatedObjectDoesNotExist` (ปัญหาเดียวกับที่ Part 019 ข้อ 181.1 อธิบายไว้) ไม่ว่า
user คนนั้นจะสมัครมาตั้งแต่เมื่อไหร่

### 778.4 Form ให้ผู้ใช้ตั้งค่าเอง

```python
# notifications/forms.py
from django import forms

from .models import NotificationPreference, NotificationType


class NotificationPreferenceForm(forms.ModelForm):
    enabled_types = forms.MultipleChoiceField(
        choices=NotificationType.choices,
        widget=forms.CheckboxSelectMultiple,
        required=False,
        label='แจ้งเตือนแบบ Real-time และ Web Push สำหรับประเภท',
    )

    class Meta:
        model = NotificationPreference
        fields = ['email_digest_enabled', 'web_push_enabled']

    def __init__(self, *args, **kwargs):
        super().__init__(*args, **kwargs)
        all_types = {choice_value for choice_value, _ in NotificationType.choices}
        muted = set(self.instance.muted_types or [])
        # checkbox แสดง "ประเภทที่เปิดอยู่" ไม่ใช่ muted_types ตรง ๆ เพราะ UX ที่ดี
        # ควรให้ผู้ใช้ติ๊กสิ่งที่ "อยากได้" มากกว่าติ๊กสิ่งที่ "อยากปิด"
        self.fields['enabled_types'].initial = list(all_types - muted)

    def save(self, commit=True):
        instance = super().save(commit=False)
        all_types = {choice_value for choice_value, _ in NotificationType.choices}
        enabled = set(self.cleaned_data['enabled_types'])
        instance.muted_types = list(all_types - enabled)
        if commit:
            instance.save()
        return instance
```

### 778.5 View และ Template

```python
# notifications/views.py (เพิ่มต่อ)
from django.contrib import messages
from django.shortcuts import redirect

from .forms import NotificationPreferenceForm
from .models import NotificationPreference


@login_required
def preferences(request):
    preference, _ = NotificationPreference.objects.get_or_create(user=request.user)
    if request.method == 'POST':
        form = NotificationPreferenceForm(request.POST, instance=preference)
        if form.is_valid():
            form.save()
            messages.success(request, 'บันทึกการตั้งค่าการแจ้งเตือนแล้ว')
            return redirect('notifications:preferences')
    else:
        form = NotificationPreferenceForm(instance=preference)
    return render(request, 'notifications/preferences.html', {'form': form})
```

```html
<!-- templates/notifications/preferences.html -->
{% extends "base.html" %}
{% block content %}
<h1>ตั้งค่าการแจ้งเตือน</h1>
<form method="post">
    {% csrf_token %}
    <fieldset>
        <legend>เปิดรับแจ้งเตือนประเภทไหนบ้าง (Real-time + Web Push)</legend>
        {{ form.enabled_types }}
    </fieldset>
    <label>{{ form.email_digest_enabled }} รับอีเมลสรุปรายวัน</label>
    <label>{{ form.web_push_enabled }} เปิดใช้ Web Push Notification</label>
    <button type="submit">บันทึก</button>
</form>
{% endblock %}
```

```python
# notifications/urls.py (เพิ่มต่อ)
urlpatterns += [
    path('preferences/', views.preferences, name='preferences'),
    path('push/subscribe/', views.push_subscribe, name='push_subscribe'),
]
```

### 778.6 ตารางสรุปเมทริกซ์ Preference x Channel

| `notification_type` | อยู่ใน `muted_types` | `email_digest_enabled` | ผลลัพธ์ real-time | ผลลัพธ์ email digest | ผลลัพธ์ web push |
|---|---|---|---|---|---|
| `new_comment` | ไม่อยู่ | `True` | ✅ ส่ง | ✅ รวมใน digest | ✅ ส่ง (ถ้า `web_push_enabled=True`) |
| `new_comment` | **อยู่** | `True` | ❌ ข้าม | ❌ ข้าม | ❌ ข้าม |
| `comment_reply` | ไม่อยู่ | **`False`** | ✅ ส่ง | ❌ ข้าม (ปิด digest ทั้งระบบ) | ✅ ส่ง (ถ้าเปิด) |

จุดสำคัญ: `muted_types` มีผลกับ**ทุกช่องทางพร้อมกัน** (ปิดประเภทนั้นคือปิดสนิท
ไม่ว่าจะช่องทางไหน) ในขณะที่ `email_digest_enabled`/`web_push_enabled` เป็น
**สวิตช์แยกต่างหากทับซ้อนอีกชั้น** เฉพาะช่องทางนั้น ๆ — นี่คือดีไซน์ที่ให้ผู้ใช้
ควบคุมได้ทั้ง "ประเภทไหน" และ "ช่องทางไหน" โดยไม่ต้องมี checkbox นับสิบช่องที่
สร้างความสับสน (2 มิติคูณกัน ไม่ใช่ 1 checkbox ต่อ 1 คู่ประเภท×ช่องทาง)

### 778.7 ตารางสรุปขั้นตอนที่ 778

| หัวข้อ | สรุป |
|---|---|
| Model | `NotificationPreference` — `muted_types` (JSONField) + สวิตช์ราย channel |
| Auto-create | Signal `post_save` บน User (รูปแบบเดียวกับ `create_profile` ของ Part 019) |
| Fallback | `get_or_create` ในทุกจุดที่อ่าน preference กันกรณี user เก่าก่อน deploy ฟีเจอร์นี้ |
| Form | Checkbox แสดง "ประเภทที่เปิดอยู่" (invert จาก `muted_types` ตอน render/save) |
| จุดที่ต้องเช็ค | ทุก task ที่จะกระจายแจ้งเตือน (773, 776, 777) ต้องเช็คก่อนส่งเสมอ |

---

## ขั้นตอนที่ 779: การเขียนเทสต์สำหรับ Pipeline การแจ้งเตือนทั้งระบบ

### 779.1 วางกลยุทธ์การเทส: แยกเทสตามชั้นของ Pipeline

Pipeline ทั้งหมดมี 4 ชั้น แต่ละชั้นเทสด้วยเทคนิคที่ต่างกัน ทวนจากที่เรียนมาตลอด
Phase 8-9:

| ชั้น | เทคนิคที่ใช้ | ทวนจาก |
|---|---|---|
| Signal ยิง Task ถูกต้อง | Mock `.delay()` ด้วย `mocker.patch` | Part 059 (unittest) |
| Task สร้าง Notification ถูกต้อง | `CELERY_TASK_ALWAYS_EAGER = True` + `pytest.mark.django_db` | Part 076 ข้อ 759 |
| WebSocket ส่งถึงจริง | `WebsocketCommunicator` + `InMemoryChannelLayer` | Part 057 ข้อ 568 |
| Preference กันการแจ้งเตือนได้จริง | เทส task โดยตรงพร้อม preference ต่างค่ากัน | Part 076 |

### 779.2 ตั้งค่าเทสให้ Celery รันแบบ Eager และ Channel Layer เป็น In-memory

```python
# conftest.py (เพิ่มต่อจาก fixture เดิมของ Part 076 ข้อ 759.3)
import pytest


@pytest.fixture(autouse=True)
def celery_eager_mode(settings):
    settings.CELERY_TASK_ALWAYS_EAGER = True
    settings.CELERY_TASK_EAGER_PROPAGATES = True


@pytest.fixture(autouse=True)
def in_memory_channel_layer(settings):
    """
    ทวนจาก Part 057 ข้อ 568.2: ใช้ InMemoryChannelLayer แทน Redis จริงในเทส
    เพราะเร็วกว่ามาก และไม่ต้องพึ่งพา Redis service ที่ต้องรันอยู่ตอนรันเทส
    (สำคัญมากสำหรับ CI ที่อาจไม่มี Redis ให้ใช้)
    """
    settings.CHANNEL_LAYERS = {
        'default': {'BACKEND': 'channels.layers.InMemoryChannelLayer'},
    }
```

### 779.3 เทสชั้นที่ 1: Signal เรียก Task ถูกต้อง

```python
# blog/tests/test_notification_signal.py
import pytest

from blog.models import Comment, Post


@pytest.mark.django_db
def test_creating_comment_triggers_notification_task(mocker, django_user_model):
    """
    เทสแค่ว่า signal เรียก task ด้วย argument ที่ถูกต้อง — ไม่สนใจว่าข้างใน task
    ทำอะไรบ้าง (นั่นคือหน้าที่ของเทสชั้นที่ 2) การแยกแบบนี้ทำให้เทสแต่ละตัว
    ล้มเหลวด้วยเหตุผลเดียว ไม่ปนกัน ทวนหลักการ Unit Test จาก Part 059
    """
    mock_task = mocker.patch('blog.signals.process_new_comment_notification.delay')
    author = django_user_model.objects.create_user(username='somchai', password='pass12345')
    post = Post.objects.create(author=author, title='โพสต์ทดสอบ', content='...', is_published=True)

    comment = Comment.objects.create(post=post, author=author, content='คอมเมนต์ทดสอบ')

    mock_task.assert_called_once_with(comment.pk)


@pytest.mark.django_db
def test_editing_comment_does_not_trigger_notification_task(mocker, django_user_model):
    """ทวนกฎ `if created:` จาก Part 019 ข้อ 183.3 — แก้ไขคอมเมนต์เดิมต้องไม่ยิง task ซ้ำ"""
    author = django_user_model.objects.create_user(username='somchai', password='pass12345')
    post = Post.objects.create(author=author, title='โพสต์ทดสอบ', content='...', is_published=True)
    comment = Comment.objects.create(post=post, author=author, content='ต้นฉบับ')

    mock_task = mocker.patch('blog.signals.process_new_comment_notification.delay')
    comment.content = 'แก้ไขแล้ว'
    comment.save()

    mock_task.assert_not_called()
```

### 779.4 เทสชั้นที่ 2: Task สร้าง Notification และเช็ค Preference ถูกต้อง

```python
# notifications/tests/test_tasks.py
import pytest

from blog.models import Comment, Post
from notifications.models import Notification, NotificationPreference, NotificationType
from notifications.tasks import process_new_comment_notification


@pytest.mark.django_db
def test_task_creates_notification_for_post_author(django_user_model):
    author = django_user_model.objects.create_user(username='somchai', password='pass12345')
    commenter = django_user_model.objects.create_user(username='malee', password='pass12345')
    post = Post.objects.create(author=author, title='โพสต์ทดสอบ', content='...', is_published=True)
    comment = Comment.objects.create(post=post, author=commenter, content='เยี่ยมมาก!')

    process_new_comment_notification(comment.pk)   # eager mode เรียกตรงได้เลย (ทวนจาก Part 076 ข้อ 759.4)

    notification = Notification.objects.get(recipient=author)
    assert notification.notification_type == NotificationType.NEW_COMMENT
    assert notification.actor == commenter
    assert notification.target == comment


@pytest.mark.django_db
def test_task_skips_self_comment(django_user_model):
    author = django_user_model.objects.create_user(username='somchai', password='pass12345')
    post = Post.objects.create(author=author, title='โพสต์ทดสอบ', content='...', is_published=True)
    comment = Comment.objects.create(post=post, author=author, content='ตอบคำถามในโพสต์ตัวเอง')

    process_new_comment_notification(comment.pk)

    assert Notification.objects.filter(recipient=author).count() == 0


@pytest.mark.django_db
def test_task_respects_muted_notification_type(django_user_model):
    """หัวใจของขั้นตอนที่ 778: ผู้ใช้ที่ mute ประเภทนี้ไว้ต้องไม่ได้รับ Notification เลย"""
    author = django_user_model.objects.create_user(username='somchai', password='pass12345')
    commenter = django_user_model.objects.create_user(username='malee', password='pass12345')
    NotificationPreference.objects.filter(user=author).update(
        muted_types=[NotificationType.NEW_COMMENT],
    )
    post = Post.objects.create(author=author, title='โพสต์ทดสอบ', content='...', is_published=True)
    comment = Comment.objects.create(post=post, author=commenter, content='คอมเมนต์')

    process_new_comment_notification(comment.pk)

    assert Notification.objects.filter(recipient=author).count() == 0


@pytest.mark.django_db
def test_task_handles_deleted_comment_gracefully(django_user_model):
    """ทวน Part 075 ข้อ 749: comment อาจถูกลบไปแล้วก่อน worker หยิบงาน — ต้องไม่ crash"""
    process_new_comment_notification(999999)   # comment_id ที่ไม่มีอยู่จริง
    # ไม่มี exception เกิดขึ้น = ผ่านเทส
```

### 779.5 เทสชั้นที่ 3: WebSocket ได้รับข้อความจริง

```python
# notifications/tests/test_consumers.py
import json

import pytest
from channels.db import database_sync_to_async
from channels.testing import WebsocketCommunicator

from blog.models import Comment, Post
from config.asgi import application
from notifications.tasks import process_new_comment_notification


@pytest.mark.asyncio
@pytest.mark.django_db(transaction=True)
async def test_recipient_receives_realtime_notification():
    from django.contrib.auth import get_user_model

    User = get_user_model()
    author = await database_sync_to_async(User.objects.create_user)(
        username='somchai', password='pass12345',
    )
    commenter = await database_sync_to_async(User.objects.create_user)(
        username='malee', password='pass12345',
    )
    post = await database_sync_to_async(Post.objects.create)(
        author=author, title='โพสต์ทดสอบ', content='...', is_published=True,
    )

    communicator = WebsocketCommunicator(application, '/ws/notifications/mine/')
    communicator.scope['user'] = author
    connected, _ = await communicator.connect()
    assert connected is True

    # skip ข้อความ 'connected' แรกที่ on_authorized_connect ส่งมา (ขั้นตอนที่ 774.3)
    await communicator.receive_from(timeout=1)

    comment = await database_sync_to_async(Comment.objects.create)(
        post=post, author=commenter, content='เยี่ยมมากครับ!',
    )
    await database_sync_to_async(process_new_comment_notification)(comment.pk)

    response = json.loads(await communicator.receive_from(timeout=2))
    assert response['event'] == 'notification'
    assert 'malee' in response['notification']['verb']

    await communicator.disconnect()


@pytest.mark.asyncio
@pytest.mark.django_db(transaction=True)
async def test_user_only_receives_own_notifications():
    """ทวนความปลอดภัยจาก Part 074 ข้อ 731.6: ห้ามข้ามกลุ่มไปเห็นแจ้งเตือนของคนอื่นเด็ดขาด"""
    from django.contrib.auth import get_user_model

    User = get_user_model()
    somchai = await database_sync_to_async(User.objects.create_user)(
        username='somchai', password='pass12345',
    )
    other_user = await database_sync_to_async(User.objects.create_user)(
        username='niran', password='pass12345',
    )

    communicator = WebsocketCommunicator(application, '/ws/notifications/mine/')
    communicator.scope['user'] = other_user   # niran เชื่อมต่ออยู่ ไม่ใช่ somchai
    connected, _ = await communicator.connect()
    assert connected is True
    await communicator.receive_from(timeout=1)  # ข้อความ 'connected'

    from notifications.realtime import push_notification_realtime
    from notifications.models import Notification, NotificationType

    notification = await database_sync_to_async(Notification.objects.create)(
        recipient=somchai, notification_type=NotificationType.NEW_COMMENT, verb='ทดสอบ',
    )
    await database_sync_to_async(push_notification_realtime)(notification)

    # niran ต้องไม่ได้รับอะไรเลย เพราะแจ้งเตือนนี้ส่งเข้ากลุ่มของ somchai เท่านั้น
    with pytest.raises(TimeoutError, match='.*'):
        import asyncio
        await asyncio.wait_for(communicator.receive_from(timeout=1), timeout=1.5)

    await communicator.disconnect()
```

### 779.6 เทสชั้นที่ 4: Email Digest ไม่ส่งซ้ำ (Idempotency)

```python
# notifications/tests/test_digest.py
import pytest
from django.core import mail

from notifications.models import Notification, NotificationType
from notifications.tasks import send_notification_digest


@pytest.mark.django_db
def test_digest_sends_once_and_marks_notifications(django_user_model):
    recipient = django_user_model.objects.create_user(
        username='somchai', password='pass12345', email='somchai@example.com',
    )
    Notification.objects.create(
        recipient=recipient, notification_type=NotificationType.NEW_COMMENT, verb='ทดสอบ 1',
    )
    Notification.objects.create(
        recipient=recipient, notification_type=NotificationType.NEW_COMMENT, verb='ทดสอบ 2',
    )

    result = send_notification_digest()

    assert result == {'digests_sent': 1}
    assert len(mail.outbox) == 1
    assert mail.outbox[0].to == ['somchai@example.com']
    assert Notification.objects.filter(recipient=recipient, digest_sent_at__isnull=True).count() == 0


@pytest.mark.django_db
def test_digest_does_not_resend_already_sent_notifications(django_user_model):
    """หัวใจของขั้นตอนที่ 776.4: เรียกซ้ำ 2 ครั้งต้องส่งอีเมลแค่ครั้งเดียว"""
    recipient = django_user_model.objects.create_user(
        username='somchai', password='pass12345', email='somchai@example.com',
    )
    Notification.objects.create(
        recipient=recipient, notification_type=NotificationType.NEW_COMMENT, verb='ทดสอบ',
    )

    send_notification_digest()
    send_notification_digest()   # เรียกซ้ำรอบสอง จำลอง Beat schedule ผิดพลาด

    assert len(mail.outbox) == 1   # ยังคงส่งแค่ฉบับเดียว ไม่ใช่ 2 ฉบับ


@pytest.mark.django_db
def test_digest_skips_users_who_disabled_it(django_user_model):
    from notifications.models import NotificationPreference

    recipient = django_user_model.objects.create_user(
        username='somchai', password='pass12345', email='somchai@example.com',
    )
    NotificationPreference.objects.filter(user=recipient).update(email_digest_enabled=False)
    Notification.objects.create(
        recipient=recipient, notification_type=NotificationType.NEW_COMMENT, verb='ทดสอบ',
    )

    send_notification_digest()

    assert len(mail.outbox) == 0
```

### 779.7 ตารางสรุปขั้นตอนที่ 779

| หัวข้อ | สรุป |
|---|---|
| Fixture หลัก | `celery_eager_mode` (Part 076) + `in_memory_channel_layer` (Part 057) ตั้ง `autouse=True` |
| เทสชั้น Signal | Mock `.delay()` ยืนยันว่าถูกเรียกด้วย `comment.pk` ที่ถูกต้อง |
| เทสชั้น Task | เรียก task ตรง ๆ (eager mode) ยืนยัน Notification/self-skip/preference/edge case |
| เทสชั้น WebSocket | `WebsocketCommunicator` ยืนยันว่าได้รับข้อความจริง และไม่รั่วไหลข้ามผู้ใช้ |
| เทสชั้น Digest | ยืนยัน idempotency (เรียกซ้ำไม่ส่งอีเมลซ้ำ) และเคารพ preference |

---

## ขั้นตอนที่ 780: สรุปและแบบฝึกหัด — ระบบแจ้งเตือนเต็มรูปแบบ

### 780.1 สรุปสิ่งที่ได้เรียนรู้ใน Part นี้

- ✅ ออกแบบสถาปัตยกรรม Real-time Notification โดยใช้ Celery เป็นจุดตัดสินใจกลาง
  ระหว่าง Signal กับช่องทางส่งออกทั้งสาม (WebSocket, Email, Web Push)
- ✅ ออกแบบ `Notification` Model ด้วย `GenericForeignKey` เพื่อชี้ไปหา object
  ต่างชนิดกันได้โดยไม่ต้องเพิ่มคอลัมน์ทุกครั้งที่มีฟีเจอร์ใหม่
- ✅ เชื่อม `post_save` Signal เข้ากับ Celery Task ตามกฎ "ส่งแค่ primary key"
  จาก Part 075 และป้องกันการแจ้งเตือนตัวเอง/คอมเมนต์ที่ถูกลบไปแล้ว
- ✅ ส่งการแจ้งเตือนแบบ real-time ผ่าน Channel Layer ด้วย `async_to_sync` และ
  Consumer ที่ต่อยอดจาก base class ของ Part 074
- ✅ สร้าง Badge/Counter แบบ reactive ด้วย Alpine.js ผสานกับ HTMX fragment
  สำหรับรายการละเอียดและปุ่ม action
- ✅ ตั้งเวลา Email Digest รายวันด้วย Celery Beat พร้อมกลไก idempotency ป้องกัน
  การส่งซ้ำ
- ✅ เข้าใจภาพรวมของ Web Push API, Service Worker, และ VAPID keys พร้อมโค้ด
  ที่ใช้งานได้จริงตั้งแต่ subscribe ถึงส่ง
- ✅ ออกแบบระบบ Opt-out รายประเภทด้วย `NotificationPreference` ที่ทำงานร่วมกับ
  ทุกช่องทางส่งออก
- ✅ เขียนเทสต์ครอบคลุมทั้ง pipeline ตั้งแต่ signal, task, WebSocket, ไปจนถึง
  email digest โดยแยกเทสตามชั้นความรับผิดชอบ

### 780.2 Checklist ก่อนไป Part ถัดไป

- [ ] สร้างแอป `notifications` พร้อม `Notification`, `NotificationPreference`,
      `PushSubscription` model และรัน migration สำเร็จ
- [ ] `blog/signals.py` เรียก `process_new_comment_notification.delay(comment.pk)`
      เมื่อมี Comment ใหม่ และไม่เรียกตอนแก้ไขคอมเมนต์เดิม
- [ ] Task สร้าง `Notification` ถูกต้อง แยกกรณี comment ปกติ vs. reply ได้
- [ ] `UserNotificationConsumer` เพิ่มใน `realtime/consumers.py` และ route
      `ws/notifications/mine/` ทำงานได้จริงเมื่อทดสอบผ่าน browser console
- [ ] Badge บน navbar อัปเดตตัวเลขทันทีเมื่อได้รับ WebSocket message โดยไม่ reload
- [ ] Celery Beat ส่ง Email Digest ตามเวลาที่ตั้งไว้ และไม่ส่งซ้ำเมื่อรันซ้ำ
- [ ] หน้า `/notifications/preferences/` บันทึกการ mute ประเภทแจ้งเตือนได้จริง
      และ task เคารพการตั้งค่านั้น
- [ ] เทสต์ทั้ง 4 ชั้น (signal, task, consumer, digest) ผ่านครบทุกเคส

### 780.3 แบบฝึกหัดท้ายบท

**แบบฝึกหัดที่ 1**: เพิ่มประเภทการแจ้งเตือนใหม่ `POST_LIKED` ให้ทำงานเต็ม pipeline
— สร้าง Signal บน model `Like` (ถ้ายังไม่มีในโปรเจกต์ของคุณ ให้สร้าง model ง่าย ๆ
`Like(user, post, created_at)` ที่มี `unique_together` ป้องกันกดถูกใจซ้ำ) เชื่อมเข้า
Celery Task ใหม่ที่สร้าง `Notification` ประเภท `post_liked` และส่งผ่านทั้ง 3 ช่องทาง
เหมือนกับ `new_comment`

**แบบฝึกหัดที่ 2**: ปัจจุบัน `mark_all_read` ผ่าน WebSocket (ขั้นตอนที่ 774.3) และ
ผ่าน HTMX endpoint (ขั้นตอนที่ 775.6) เป็นคนละ code path ที่ทำสิ่งเดียวกัน ให้รีแฟก
เตอร์ (refactor) ทั้งสองจุดให้เรียกฟังก์ชันร่วมกันเพียงจุดเดียวใน `notifications/
services.py` ตามหลัก DRY จาก Part 001 ข้อ 3.4 แล้วเขียนเทสต์ยืนยันว่าทั้งสอง entry
point ให้ผลลัพธ์เหมือนกัน

**แบบฝึกหัดที่ 3**: เพิ่ม Rate Limiting ให้ WebSocket action `mark_read`/
`mark_all_read` โดยนำ `RateLimitMixin` จาก Part 074 ข้อ 732.5 มาผสมกับ
`UserNotificationConsumer` ป้องกันผู้ใช้ (หรือบั๊กฝั่ง client) ยิง request มาถี่
เกินไป

**แบบฝึกหัดที่ 4 (ขั้นสูง)**: เขียน management command `send_test_notification
--username=<name> --type=<notification_type>` ที่สร้าง `Notification` ทดสอบและ
ยิงผ่านทั้ง 3 ช่องทางทันที (ไม่ต้องรอ Celery Beat) ใช้สำหรับทีม QA ทดสอบ UI badge
และ Web Push โดยไม่ต้องสร้าง Comment จริงทุกครั้ง — ออกแบบให้ปฏิเสธการทำงานถ้ารันบน
`DEBUG = False` (ทวนความระมัดระวังเรื่อง production safety)

### 780.4 คำถามที่พบบ่อย (FAQ)

**Q: ทำไมไม่ใช้ package สำเร็จรูปอย่าง `django-notifications-hq` แทนที่จะเขียนเอง
ทั้งหมด?**
A: `django-notifications-hq` และ package คล้ายกันเป็นตัวเลือกที่ดีสำหรับโปรเจกต์
จริงที่ต้องการความเร็วในการพัฒนา และมีฟีเจอร์ Generic Foreign Key แบบเดียวกับที่
Part นี้สอนอยู่แล้ว แต่หลักสูตรนี้เลือกเขียนเองทั้งหมดเพราะเป้าหมายคือให้คุณเข้าใจ
**กลไกเบื้องหลัง** ของการประกอบ Signal + Celery + Channels + Generic Foreign Key
เข้าด้วยกัน ซึ่งเป็นทักษะที่ถ่ายทอดไปใช้กับปัญหาอื่นที่ไม่มี package สำเร็จรูปรองรับ
ได้ ในงานจริงเมื่อเข้าใจกลไกนี้ดีแล้ว การเลือกใช้ package สำเร็จรูปหรือเขียนเองเป็น
การตัดสินใจทางวิศวกรรมที่ทำได้อย่างมีข้อมูลรองรับ

**Q: ถ้าผู้ใช้เปิดหลายแท็บพร้อมกัน (หรือหลายอุปกรณ์) badge จะอัปเดตครบทุกที่ไหม?**
A: ครบ เพราะการออกแบบกลุ่ม `user_notifications_<user_id>` ในขั้นตอนที่ 774.1 ไม่ได้
ผูกกับ connection เส้นใดเส้นหนึ่ง — ทุกแท็บ/อุปกรณ์ที่ล็อกอินเป็น user คนเดียวกันจะ
`group_add` เข้ากลุ่มเดียวกันหมด (คนละ `channel_name` แต่กลุ่มเดียวกัน) เมื่อมีการ
`group_send` เข้ากลุ่มนี้ ทุก connection ที่เป็นสมาชิกจะได้รับข้อความพร้อมกันทั้งหมด
ทวนกลไกนี้จาก Part 057 ข้อ 565

**Q: ถ้า Celery Worker ล่มไปพอดีตอนที่มีคนคอมเมนต์ การแจ้งเตือนจะหายไปเลยไหม?**
A: ไม่หาย เพราะ Broker (Redis) เก็บงานที่ยังไม่ถูกยืนยันว่าทำสำเร็จไว้เสมอ (ทวนจาก
Part 075 ข้อ 741.5) เมื่อ Worker กลับมาทำงานใหม่จะหยิบงานที่ค้างอยู่ไปทำต่อทันที
ผู้ใช้อาจได้รับการแจ้งเตือนช้ากว่าปกติเล็กน้อย (ไม่ real-time 100% ในเหตุการณ์ที่
Worker ล่มพอดี) แต่**จะไม่หายไปเฉย ๆ อย่างเงียบ ๆ** ซึ่งต่างจากการเขียนโค้ดแจ้งเตือน
ด้วย `threading.Thread` ธรรมดาตามที่ Part 075 ข้อ 741.5 เตือนไว้

**Q: ควรเก็บ Notification เก่าไว้ตลอดไปไหม หรือควรลบทิ้งบ้าง?**
A: ไม่ควรเก็บตลอดไป ในทางปฏิบัติควรมี Celery Beat task อีกตัวที่ลบ `Notification`
ที่เก่ากว่า 90-180 วันทิ้งเป็นระยะ (รูปแบบเดียวกับ periodic task อื่นที่ Part 076
ข้อ 751 สอนไว้) เพื่อไม่ให้ตาราง `notifications_notification` โตจนกระทบ
performance ของ query `unread_notification_count` ที่ต้องรันแทบทุก request
(ทวนความสำคัญของการดูแลขนาดตารางจาก Part 070)

---

## เตรียมตัวสำหรับ Part ถัดไป

**Part 079: Background Job Monitoring: Flower, Django-RQ** ซึ่งเป็น Part สุดท้าย
ของ Phase 9 จะพาไปดูว่า Celery Task ทั้งหมดที่สร้างมาตลอด Part 075-078
(รวมถึง Celery Task ของระบบแจ้งเตือนใน Part นี้) ทำงานอยู่หรือไม่ ล้มเหลวบ่อยแค่ไหน
ผ่านเครื่องมือ Flower และเปรียบเทียบกับทางเลือกอย่าง django-RQ ก่อนที่ Phase 9
จะปิดท้ายด้วยการสรุปภาพรวมทั้งหมด แล้วเข้าสู่ Phase 10 (Security)

ระบบแจ้งเตือนที่คุณสร้างเสร็จใน Part นี้จะยังคงถูกใช้อ้างอิงต่อไปใน Part 079
โดยเฉพาะตอนที่หลักสูตรพูดถึง Monitoring (เพื่อติดตามว่า Celery Task ของระบบแจ้งเตือน
ล้มเหลวบ่อยแค่ไหนผ่าน Flower) — เตรียมโปรเจกต์ของคุณให้พร้อม แล้วไปต่อกันเลย!
