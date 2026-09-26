# Part 019: Signals และ Django Lifecycle Hooks

> **ขั้นตอนที่ 181-190 ของหลักสูตร** | Phase 2: Models, ORM และ Admin
>
> เป้าหมายของ Part นี้: เข้าใจแนวคิด **Signals** ของ Django อย่างถ่องแท้ ตั้งแต่หลักการ
> Observer Pattern เบื้องหลัง ไปจนถึงการใช้งานจริงเพื่อ **แก้ปัญหาที่ค้างมาตั้งแต่ Part 012**
> — `User` ใหม่ที่ไม่มี `Profile` จนทำให้ `user.profile` โยน `RelatedObjectDoesNotExist`
> คุณจะสร้างระบบ auto-create `Profile` ด้วย `post_save`, เรียนรู้ตำแหน่งไฟล์ `signals.py`
> ที่ถูกต้องและวิธีเชื่อมผ่าน `AppConfig.ready()`, ใช้ `pre_delete`/`post_delete` ลบไฟล์
> avatar ออกจาก storage จริง, ใช้ `m2m_changed` เพื่อ log การเปลี่ยนแปลง tag, สร้าง
> Custom Signal ของตัวเอง, และเข้าใจข้อควรระวังที่ทำให้ทีมงานมืออาชีพจำนวนมากใช้ signals
> อย่างระมัดระวัง เมื่อจบ Part นี้ โปรเจกต์บล็อกของคุณจะไม่มี `User` ตัวไหนที่ขาด `Profile`
> อีกต่อไป และไฟล์ avatar เก่าจะไม่ค้างอยู่ใน storage เมื่อ Profile ถูกลบ

---

## สารบัญของ Part นี้

- ขั้นตอนที่ 181: Signals คืออะไร แนวคิด Observer Pattern และ Built-in Signals หลัก
- ขั้นตอนที่ 182: เชื่อม Signal Receiver ด้วย `@receiver` เทียบกับ `signal.connect()`
- ขั้นตอนที่ 183: แก้ปัญหาจาก Part 012 จริง — Auto-create `Profile` ด้วย `post_save`
- ขั้นตอนที่ 184: ตำแหน่งไฟล์ `signals.py` ที่ถูกต้อง และการเชื่อมใน `AppConfig.ready()`
- ขั้นตอนที่ 185: `pre_delete`/`post_delete` — ลบไฟล์ avatar ออกจาก Storage จริง
- ขั้นตอนที่ 186: `m2m_changed` — Log การเพิ่ม/ลบ Tag ของ `Post`
- ขั้นตอนที่ 187: สร้าง Custom Signal ของตัวเอง (`django.dispatch.Signal`)
- ขั้นตอนที่ 188: ข้อควรระวังของ Signals — Performance, Hidden Side Effects, ทางเลือก
- ขั้นตอนที่ 189: Signal อื่น ๆ ที่ควรรู้: `request_started`, `request_finished`, `got_request_exception`
- ขั้นตอนที่ 190: สรุปและแบบฝึกหัด

---

## ขั้นตอนที่ 181: Signals คืออะไร แนวคิด Observer Pattern และ Built-in Signals หลัก

### 181.1 ปัญหาที่ Signals แก้ไข

ลองจินตนาการสถานการณ์นี้: ทุกครั้งที่มี `User` ใหม่ถูกสร้างในระบบ คุณต้องการให้เกิดสิ่ง
ต่าง ๆ ตามมาโดยอัตโนมัติ เช่น สร้าง `Profile` ให้, ส่งอีเมลต้อนรับ, บันทึก log, และสร้าง
"กระเป๋าเงิน" เริ่มต้นให้ในระบบ e-commerce

วิธีที่ตรงไปตรงมาที่สุดคือเขียนโค้ดเหล่านี้ต่อท้ายทุกจุดที่สร้าง `User`:

```python
# ไม่แนะนำ — ต้องเขียนซ้ำทุกที่ที่สร้าง User
def register_view(request):
    user = User.objects.create_user(username=..., password=...)
    Profile.objects.create(user=user)          # ต้องจำให้เขียนบรรทัดนี้
    send_welcome_email(user)                    # ต้องจำให้เขียนบรรทัดนี้
    Wallet.objects.create(user=user, balance=0)  # ต้องจำให้เขียนบรรทัดนี้
    ...
```

ปัญหาคือ `User` อาจถูกสร้างได้จาก**หลายจุด**ในระบบ: หน้าสมัครสมาชิกปกติ, Django Admin,
คำสั่ง `createsuperuser`, `python manage.py shell`, data migration, หรือ management
command ที่ import ผู้ใช้จำนวนมาก — ถ้าลืมเขียนโค้ดข้างต้นไว้ในจุดใดจุดหนึ่ง ระบบจะมี
`User` ที่ขาด `Profile` ทันที (นี่คือปัญหาที่เราเจอจริงใน Part 012 ขั้นตอนที่ 112.5!)

**Signals** คือกลไกของ Django ที่ให้คุณ "แปะ" โค้ดที่ต้องการให้ทำงานเข้ากับ **เหตุการณ์**
(event) แทนที่จะเขียนซ้ำที่ **ทุกจุดเรียกใช้งาน** ไม่ว่า `User` จะถูกสร้างจากที่ไหนก็ตาม
โค้ดที่แปะไว้จะทำงานเสมอ เพราะมันถูกเชื่อมกับ "เหตุการณ์การบันทึก" ไม่ใช่กับ "จุดเรียก"

### 181.2 แนวคิด Observer Pattern เบื้องหลัง Signals

Signals ของ Django สร้างขึ้นจากแนวคิดการออกแบบซอฟต์แวร์ที่เรียกว่า
**Observer Pattern** (บางครั้งเรียก Publish-Subscribe หรือ Pub/Sub) ซึ่งประกอบด้วย
3 องค์ประกอบหลัก:

| องค์ประกอบ | บทบาท | ตัวอย่างใน Django |
|---|---|---|
| **Signal** | ตัวกลางที่ "ประกาศ" ว่ามีเหตุการณ์ประเภทนี้อยู่ | `post_save`, `pre_delete` |
| **Sender** | ผู้ส่งสัญญาณเมื่อเหตุการณ์เกิดขึ้นจริง | `User`, `Profile`, `Post` (โมเดลใด ๆ) |
| **Receiver** | ฟังก์ชันที่ "สมัครรับฟัง" สัญญาณนั้น แล้วทำงานเมื่อถูกเรียก | ฟังก์ชัน `create_profile(...)` ที่คุณเขียนเอง |

```
┌──────────┐     save()      ┌───────────────┐    ประกาศสัญญาณ    ┌──────────────────┐
│  Sender  │ ───────────────>│  Django ORM   │ ──────────────────>│   Signal Dispatcher │
│ (User)   │   ถูกบันทึก      │  (model save) │   post_save.send() │  (django.dispatch)   │
└──────────┘                 └───────────────┘                    └─────────┬────────────┘
                                                                             │ แจ้งทุก receiver
                                                    ┌────────────────────────┼────────────────────────┐
                                                    ▼                        ▼                        ▼
                                          create_profile()          send_welcome_email()      log_new_user()
                                          (receiver 1)               (receiver 2)               (receiver 3)
```

จุดสำคัญของ Observer Pattern คือ **Sender ไม่จำเป็นต้องรู้จัก Receiver เลย** — `User`
model ไม่มีโค้ดสักบรรทัดที่รู้ว่า `Profile` จะถูกสร้างตามหลัง มันแค่ "ประกาศ" ว่า
"ฉันเพิ่งถูกบันทึก" แล้วปล่อยให้ระบบ dispatcher จัดการแจ้งทุกคนที่สนใจฟังเอง สิ่งนี้ทำให้
โค้ดของ `User` (ซึ่งเราแก้ไม่ได้ เพราะเป็นส่วนหนึ่งของ Django เอง) และโค้ดของ `Profile`
(ที่เราเขียนเอง) **แยกออกจากกันอย่างสมบูรณ์ (decoupled)**

### 181.3 รายการ Built-in Signals หลักที่เกี่ยวกับ Model

Django มี built-in signals อยู่หลายกลุ่ม กลุ่มที่ใช้บ่อยที่สุดคือ **Model signals**
ซึ่งอยู่ใน `django.db.models.signals`:

| Signal | เกิดขึ้นเมื่อไหร่ | Arguments สำคัญที่ได้รับ |
|---|---|---|
| `pre_init` | ก่อน `Model.__init__()` เริ่มทำงาน (ก่อนสร้าง instance ในหน่วยความจำ) | `sender`, `args`, `kwargs` |
| `post_init` | หลัง `Model.__init__()` ทำงานเสร็จ (instance ถูกสร้างในหน่วยความจำแล้ว แต่ยังไม่บันทึกลง DB) | `sender`, `instance` |
| `pre_save` | ก่อนเรียก `save()` ลงฐานข้อมูล (ก่อน INSERT/UPDATE) | `sender`, `instance`, `raw`, `using`, `update_fields` |
| `post_save` | หลังบันทึกลงฐานข้อมูลสำเร็จ (หลัง INSERT/UPDATE) | `sender`, `instance`, `created`, `raw`, `using`, `update_fields` |
| `pre_delete` | ก่อนลบ instance ออกจากฐานข้อมูล | `sender`, `instance`, `using` |
| `post_delete` | หลังลบ instance ออกจากฐานข้อมูลสำเร็จ | `sender`, `instance`, `using` |
| `m2m_changed` | เมื่อความสัมพันธ์ `ManyToManyField` เปลี่ยนแปลง (`add`, `remove`, `clear`, `set`) | `sender`, `instance`, `action`, `reverse`, `model`, `pk_set`, `using` |

นอกจากนี้ยังมี **request/response signals** ที่ทำงานระดับ HTTP request ทั้งก้อน
(เราจะเจาะลึกในขั้นตอนที่ 189) และ **migration signals** อย่าง `pre_migrate`/
`post_migrate` ที่จะได้เจอในหัวข้อ deployment ช่วง Phase 11

### 181.4 อธิบายความแตกต่างที่มือใหม่มักสับสน: `pre_save` vs `post_save`

จุดที่ต้องเข้าใจให้แม่นคือ **timing** ของแต่ละ signal เทียบกับการเขียนลงฐานข้อมูลจริง:

```
คำสั่ง user.save() ถูกเรียก
         │
         ▼
┌─────────────────┐
│   pre_save       │  ← instance.pk อาจยังเป็น None (ถ้าเป็นการสร้างใหม่)
│   ส่งสัญญาณ       │     ยังไม่มีข้อมูลใน DB
└────────┬─────────┘
         │
         ▼
┌─────────────────┐
│  INSERT/UPDATE   │  ← Django สั่ง SQL จริงไปที่ฐานข้อมูล
│  ลงฐานข้อมูลจริง   │
└────────┬─────────┘
         │
         ▼
┌─────────────────┐
│   post_save      │  ← instance.pk มีค่าแน่นอนแล้ว (ถ้าสร้างใหม่จะได้ pk จาก DB)
│   ส่งสัญญาณ       │     ข้อมูลถูกบันทึกจริงแล้ว 100%
└──────────────────┘
```

**กฎที่ต้องจำ**: ถ้าต้องการ "แก้ไขค่าของ instance ก่อนบันทึก" ให้ใช้ `pre_save` (เช่น
เติม slug อัตโนมัติ — แม้ในทางปฏิบัติมักเขียนใน `save()` ของ model เองมากกว่า) แต่ถ้า
ต้องการ "ทำอะไรบางอย่างกับ object อื่นหลังจากที่ instance นี้ถูกบันทึกลง DB จริงแล้ว"
(เช่น สร้าง `Profile` ให้ `User` ที่เพิ่งถูกสร้าง) ต้องใช้ **`post_save` เท่านั้น** เพราะ
ต้องมั่นใจว่า `user.pk` มีค่าแล้วก่อนจะเอาไปสร้าง `Profile` ที่อ้างอิงกลับมา

---

## ขั้นตอนที่ 182: เชื่อม Signal Receiver ด้วย `@receiver` เทียบกับ `signal.connect()`

### 182.1 วิธีที่ 1: `signal.connect()` แบบ Manual

วิธีดั้งเดิมที่สุดในการเชื่อม receiver function เข้ากับ signal คือเรียก `.connect()`
บน signal object โดยตรง:

```python
# accounts/signals.py
from django.contrib.auth.models import User
from django.db.models.signals import post_save


def log_user_created(sender, instance, created, **kwargs):
    if created:
        print(f'มีผู้ใช้ใหม่: {instance.username}')


post_save.connect(log_user_created, sender=User)
```

พารามิเตอร์สำคัญของ `.connect()`:

| พารามิเตอร์ | ความหมาย |
|---|---|
| `receiver` | ฟังก์ชัน (หรือ callable) ที่จะถูกเรียกเมื่อ signal ถูกส่ง |
| `sender` | จำกัดให้ทำงานเฉพาะเมื่อ sender ตรงกับ model ที่ระบุ (ถ้าไม่ระบุ = รับฟังทุก sender) |
| `weak` | ใช้ weak reference หรือไม่ (default `True` — ป้องกัน memory leak แต่ต้องระวังถ้า receiver เป็น local function ที่ถูก garbage collect) |
| `dispatch_uid` | รหัสเฉพาะป้องกันการเชื่อม receiver ตัวเดียวกันซ้ำสองครั้งโดยไม่ตั้งใจ |

### 182.2 วิธีที่ 2: `@receiver` Decorator (แนะนำ — เป็นมาตรฐานที่ใช้กันจริง)

Django มี decorator ชื่อ `@receiver` ใน `django.dispatch` ที่ทำสิ่งเดียวกันแต่**อ่านง่าย
กว่ามาก** และเป็นรูปแบบที่โปรเจกต์ระดับมืออาชีพส่วนใหญ่เลือกใช้:

```python
# accounts/signals.py
from django.contrib.auth.models import User
from django.db.models.signals import post_save
from django.dispatch import receiver


@receiver(post_save, sender=User)
def log_user_created(sender, instance, created, **kwargs):
    if created:
        print(f'มีผู้ใช้ใหม่: {instance.username}')
```

`@receiver` รับพารามิเตอร์ตัวแรกเป็น signal (หรือ **list ของ signal หลายตัว** ก็ได้ ถ้า
ต้องการให้ฟังก์ชันเดียวรับฟังหลายสัญญาณพร้อมกัน) และรับ keyword argument เดียวกับ
`.connect()` ทั้งหมด (`sender`, `weak`, `dispatch_uid`)

```python
# รับฟังได้หลาย signal ด้วยฟังก์ชันเดียว
@receiver([post_save, post_delete], sender=Profile)
def clear_profile_cache(sender, instance, **kwargs):
    cache.delete(f'profile:{instance.pk}')
```

### 182.3 ตารางเปรียบเทียบ `@receiver` กับ `.connect()`

| ประเด็น | `@receiver` decorator | `.connect()` แบบ manual |
|---|---|---|
| ความอ่านง่าย | สูง — เห็นความสัมพันธ์ signal-receiver ตรงจุดประกาศฟังก์ชันทันที | ต้องเลื่อนตาไปดูบรรทัดที่เรียก `.connect()` แยกต่างหาก |
| รองรับหลาย signal พร้อมกัน | ✅ ส่งเป็น list ได้เลย | ต้องเขียน `.connect()` แยกทีละ signal |
| นิยมใช้ในโปรเจกต์จริง | ✅ เป็นมาตรฐานโดยพฤตินัย | ใช้เมื่อต้องเชื่อม/ยกเลิกการเชื่อมแบบ dynamic ตอน runtime |
| การยกเลิกการเชื่อม (`disconnect`) | ทำได้ แต่ต้องเก็บ reference ฟังก์ชันไว้ก่อน | ตรงไปตรงมากว่า เพราะเรียก `.connect()`/`.disconnect()` คู่กันได้ชัดเจน |

> **คำแนะนำของหลักสูตรนี้**: ใช้ `@receiver` เป็นค่าเริ่มต้นเสมอ เก็บ `.connect()` แบบ
> manual ไว้สำหรับกรณีพิเศษเท่านั้น เช่น ต้องการเชื่อม/ยกเลิกการเชื่อม signal แบบมีเงื่อนไข
> ตอน runtime (เช่น ปิด signal ชั่วคราวระหว่างรัน data migration ขนาดใหญ่)

### 182.4 `dispatch_uid`: ป้องกันปัญหา Receiver ถูกเรียกซ้ำ

ปัญหาที่พบบ่อยมากในโปรเจกต์จริงคือ receiver ถูกเรียก**สองครั้งขึ้นไป**ต่อหนึ่งเหตุการณ์
สาเหตุมักมาจากไฟล์ที่มีการเชื่อม signal ถูก import ซ้ำมากกว่าหนึ่งครั้ง (เราจะเจาะลึก
สาเหตุที่แท้จริงและวิธีป้องกันแบบถูกต้องในขั้นตอนที่ 184) การใส่ `dispatch_uid` เป็น
เกราะป้องกันชั้นที่สองที่ทำให้ Django รู้ว่า "receiver ที่มี uid นี้เชื่อมไปแล้ว ไม่ต้อง
เชื่อมซ้ำ":

```python
@receiver(post_save, sender=User, dispatch_uid='accounts_create_profile')
def create_profile(sender, instance, created, **kwargs):
    ...
```

`dispatch_uid` ควรเป็น string ที่ไม่ซ้ำกันทั้งโปรเจกต์ นิยมตั้งชื่อตามรูปแบบ
`'<app_name>_<action>'` เพื่อให้อ่านง่ายและไม่ชนกัน

---

## ขั้นตอนที่ 183: แก้ปัญหาจาก Part 012 จริง — Auto-create `Profile` ด้วย `post_save`

### 183.1 ทบทวนปัญหาที่ค้างมาจาก Part 012

จำได้ไหมว่าใน Part 012 ขั้นตอนที่ 112.5 เราเจอปัญหานี้:

```python
>>> user2 = User.objects.create_user(username='anon', password='pass123456')
>>> user2.profile
Traceback (most recent call last):
    ...
accounts.models.RelatedObjectDoesNotExist: User has no profile.
```

เพราะ `Profile` ถูกสร้างแยกต่างหากด้วยมือ ถ้าใครลืมสร้างให้ `User` ใหม่ (ซึ่งเกิดขึ้นบ่อย
มากในทางปฏิบัติ เพราะ `User` อาจถูกสร้างจากหลายจุด: หน้าสมัครสมาชิก, Django Admin,
`createsuperuser`, หรือ shell) ระบบจะพังทันทีที่มีโค้ดส่วนไหนเข้าถึง `user.profile` วันนี้
เราจะแก้ปัญหานี้ให้ถาวรด้วย `post_save` signal

### 183.2 ทำไมต้องเป็น `post_save` ไม่ใช่ `pre_save`

การสร้าง `Profile` ต้องมี `user` (instance ของ `User`) ที่มี **`pk` แล้ว** เพราะ
`Profile.user` เป็น `OneToOneField` ที่อ้างอิงกลับไปหา `User` ด้วย `user_id` — ถ้า `User`
ยังไม่ถูกบันทึกลงฐานข้อมูล (`pk` ยังเป็น `None`) เราจะสร้าง `Profile` ที่อ้างอิงไม่ได้เลย
นี่คือเหตุผลที่ **ต้องใช้ `post_save`** ซึ่งรับประกันว่า `instance.pk` มีค่าแน่นอนแล้ว

### 183.3 ทำความเข้าใจ `created` flag ให้แม่น — จุดที่สำคัญที่สุดของขั้นตอนนี้

`post_save` ส่ง keyword argument ชื่อ **`created`** มาด้วยเสมอ ซึ่งเป็น `boolean`:

| ค่า `created` | ความหมาย |
|---|---|
| `True` | นี่คือการ **INSERT แถวใหม่** ลงฐานข้อมูล (object เพิ่งถูกสร้างครั้งแรก) |
| `False` | นี่คือการ **UPDATE แถวเดิม** ที่มีอยู่แล้ว |

**นี่คือจุดที่มือใหม่พลาดบ่อยที่สุด**: ถ้าลืมเช็ค `created` และเขียนโค้ดสร้าง `Profile`
แบบไม่มีเงื่อนไข จะเกิดข้อผิดพลาดร้ายแรงทันทีที่มีใครแก้ไข `User` ที่มีอยู่แล้ว (เช่น
เปลี่ยน email) เพราะ `post_save` จะถูกส่งสัญญาณทุกครั้งที่ `save()` ถูกเรียก **ไม่ว่าจะ
เป็นการสร้างใหม่หรืออัปเดตก็ตาม**:

```python
# ❌ ผิด — จะพยายามสร้าง Profile ซ้ำทุกครั้งที่ User ถูก save() แม้เป็นแค่การอัปเดต
@receiver(post_save, sender=User)
def create_profile_wrong(sender, instance, **kwargs):
    Profile.objects.create(user=instance)
    # เมื่อ user ที่มี Profile อยู่แล้วถูกแก้ไขและ save() อีกครั้ง
    # โค้ดนี้จะพยายาม INSERT Profile ซ้ำ แล้วชน unique constraint ของ OneToOneField
    # -> django.db.utils.IntegrityError: UNIQUE constraint failed: accounts_profile.user_id
```

```python
# ✅ ถูกต้อง — สร้าง Profile เฉพาะตอนที่เป็นการสร้าง User ใหม่เท่านั้น
@receiver(post_save, sender=User)
def create_profile_correct(sender, instance, created, **kwargs):
    if created:
        Profile.objects.create(user=instance)
```

> **กฎเหล็กของขั้นตอนนี้**: ทุกครั้งที่เขียน `post_save` receiver ที่ทำงานกับการ "สร้าง
> ครั้งแรก" เท่านั้น **ต้องเช็ค `if created:` เสมอ** ไม่มีข้อยกเว้น

### 183.4 เขียน Receiver แบบสมบูรณ์ — Auto-create และ Auto-save `Profile`

ในทางปฏิบัติ เรามักต้องการทำ 2 อย่างพร้อมกัน: (1) สร้าง `Profile` เมื่อ `User` ใหม่ถูก
สร้าง และ (2) บันทึก `Profile` ที่มีอยู่แล้วทุกครั้งที่ `User` ถูกอัปเดต (เผื่อกรณีมี
signal อื่นที่แก้ไข `user.profile` ระหว่างทาง หรือใช้ pattern `save()` ต่อเนื่องกัน):

```python
# accounts/signals.py
from django.conf import settings
from django.db.models.signals import post_save
from django.dispatch import receiver

from .models import Profile


@receiver(post_save, sender=settings.AUTH_USER_MODEL, dispatch_uid='accounts_create_profile')
def create_profile(sender, instance, created, **kwargs):
    """สร้าง Profile ให้อัตโนมัติทุกครั้งที่มี User ใหม่ถูกสร้าง"""
    if created:
        Profile.objects.create(user=instance)


@receiver(post_save, sender=settings.AUTH_USER_MODEL, dispatch_uid='accounts_save_profile')
def save_profile(sender, instance, **kwargs):
    """บันทึก Profile ที่มีอยู่ทุกครั้งที่ User ถูกอัปเดต (กันเผื่อ Profile หลุด sync)"""
    if hasattr(instance, 'profile'):
        instance.profile.save()
```

สังเกตว่าเราใช้ `sender=settings.AUTH_USER_MODEL` (string `'auth.User'`) แทนที่จะ
`import User` ตรง ๆ — ด้วยเหตุผลเดียวกับที่อธิบายไว้ใน Part 012 ขั้นตอนที่ 112.3: ถ้า
โปรเจกต์นี้เปลี่ยนไปใช้ Custom User Model ในอนาคต (Part 032) โค้ดส่วนนี้จะยังทำงานถูกต้อง
ทันทีโดยไม่ต้องแก้ไข **แต่มีข้อควรระวัง**: พารามิเตอร์ `sender=` ของ `@receiver` ยอมรับ
ทั้ง string แบบ `'app_label.ModelName'` และ class จริง แต่ **`settings.AUTH_USER_MODEL`
คืนค่าเป็น string ในรูปแบบนั้นอยู่แล้ว** จึงส่งตรงเข้า `sender=` ได้เลยโดยไม่ต้องแปลง

### 183.5 ทดสอบว่าปัญหาถูกแก้แล้วจริง

```bash
python manage.py shell
```

```python
>>> from django.contrib.auth.models import User
>>> user3 = User.objects.create_user(username='malee', password='securepass456')
>>> user3.profile
<Profile: โปรไฟล์ของ malee>
>>> user3.profile.bio
''
```

ไม่มี `RelatedObjectDoesNotExist` อีกต่อไป! ทุก `User` ที่ถูกสร้างจากนี้ไป (ไม่ว่าจะจาก
shell, หน้าสมัครสมาชิก, หรือ Django Admin) จะมี `Profile` ติดตัวมาโดยอัตโนมัติเสมอ

### 183.6 กรณี `createsuperuser` — ทดสอบเส้นทางที่มักถูกลืม

```bash
python manage.py createsuperuser --username admin --email admin@example.com
```

```python
>>> from django.contrib.auth.models import User
>>> admin = User.objects.get(username='admin')
>>> admin.profile
<Profile: โปรไฟล์ของ admin>
```

นี่คือพลังที่แท้จริงของ signals: `createsuperuser` เป็นคำสั่งของ Django เอง เราไม่มีทาง
ไปแก้โค้ดภายในของมันเพื่อเพิ่มบรรทัด `Profile.objects.create(...)` ได้เลย แต่เพราะ
`createsuperuser` ก็เรียก `User.objects.create_superuser()` ซึ่งสุดท้ายก็ผ่าน `save()`
เหมือนกัน `post_save` signal จึงทำงานให้เราโดยอัตโนมัติ **โดยไม่ต้องแก้โค้ดของ Django
เลยแม้แต่บรรทัดเดียว**

---

## ขั้นตอนที่ 184: ตำแหน่งไฟล์ `signals.py` ที่ถูกต้อง และการเชื่อมใน `AppConfig.ready()`

### 184.1 ทำไมห้ามเชื่อม Signal ตรง ๆ ใน `models.py`

หลายคนอาจคิดว่าเขียน receiver function ไว้ท้าย `models.py` เลยก็น่าจะได้ เช่น:

```python
# accounts/models.py — ❌ ไม่แนะนำ
from django.db.models.signals import post_save
from django.dispatch import receiver


class Profile(models.Model):
    ...


@receiver(post_save, sender=settings.AUTH_USER_MODEL)
def create_profile(sender, instance, created, **kwargs):
    if created:
        Profile.objects.create(user=instance)
```

วิธีนี้ **ใช้งานได้จริง** ในกรณีง่าย ๆ แต่ Django ไม่แนะนำด้วยเหตุผลสำคัญ 2 ข้อ:

1. **ความสับสนเรื่องหน้าที่ของไฟล์ (Separation of Concerns)**: `models.py` ควรมีหน้าที่
   นิยาม **โครงสร้างข้อมูล** เท่านั้น ส่วน "พฤติกรรมที่เกิดขึ้นเมื่อมีเหตุการณ์" เป็นคนละ
   concern กัน การแยกไฟล์ทำให้โค้ดอ่านง่ายขึ้นมากเมื่อโปรเจกต์ใหญ่ขึ้น
2. **ความเสี่ยงเรื่อง Double-Registration**: นี่คือปัญหาทางเทคนิคที่ร้ายแรงกว่า และเป็น
   เหตุผลหลักที่ Django กำหนดรูปแบบมาตรฐานไว้ในเอกสารทางการ

### 184.2 ปัญหา Double-Registration คืออะไร

เมื่อ Django โหลดแอปตอนเริ่มระบบ (`AppConfig.ready()` — จะอธิบายละเอียดถัดไป) โมดูล
`models.py` ของทุกแอปจะถูก import เข้ามาโดยอัตโนมัติเป็นส่วนหนึ่งของกระบวนการโหลด
app registry **แต่ `models.py` ก็อาจถูก import ซ้ำได้อีกจากที่อื่น** เช่น จากไฟล์
`admin.py`, จาก management command, หรือจากการ `import` ข้ามแอประหว่างกัน

ถ้า decorator `@receiver` อยู่ใน `models.py` โดยตรง ทุกครั้งที่โมดูลนี้ถูก import ซ้ำใน
บริบทที่ Python **ยังไม่เคย cache โมดูลนี้ไว้** (เช่น กรณีพิเศษบางอย่างของ testing
framework ที่ reload โมดูล หรือการตั้งค่า autoreload บางรูปแบบ) `.connect()` จะถูกเรียก
ซ้ำสอง (หรือมากกว่า) ครั้ง ทำให้ receiver function เดียวกันถูกเชื่อมเข้ากับ signal
**หลายชุด** — ผลลัพธ์คือ `create_profile()` อาจถูกเรียก 2 ครั้งต่อ `User` หนึ่งคน (แม้จะ
มี `if created:` ป้องกันไม่ให้สร้าง `Profile` ซ้ำ แต่ในกรณี signal อื่นที่ไม่มีการป้องกัน
เช่น "ส่งอีเมลต้อนรับ" ผู้ใช้จะได้รับอีเมลซ้ำ 2 ฉบับ!)

Django ปกติจะ import `models.py` เพียงครั้งเดียวต่อการรันโปรแกรมหนึ่งครั้งอยู่แล้ว (ด้วย
กลไก module caching ของ Python เอง) ทำให้ปัญหานี้**ไม่ค่อยเกิดในกรณีทั่วไป** แต่ Django
เอกสารทางการยังคงแนะนำรูปแบบที่ปลอดภัยกว่าเสมอ เพราะ:

- โปรเจกต์ที่ใช้ **testing framework บางตัว** หรือ **hot-reload บางรูปแบบ** อาจ import
  ซ้ำได้ในสถานการณ์ที่คาดไม่ถึง
- `dispatch_uid` (จากขั้นตอนที่ 182.4) ช่วยป้องกันปัญหานี้ได้ระดับหนึ่ง แต่ **การป้องกัน
  ที่ถูกต้องตั้งแต่ต้นทางคือแยกไฟล์และเชื่อมใน `ready()` เพียงจุดเดียว** ดีกว่าการแก้ปัญหา
  ปลายทางด้วย `dispatch_uid` เสมอ

### 184.3 โครงสร้างไฟล์มาตรฐานที่ Django แนะนำ

```
accounts/
├── __init__.py
├── apps.py          ← เชื่อม signals ที่นี่ ใน AppConfig.ready()
├── models.py         ← นิยาม Profile model เท่านั้น ไม่มี signal
├── signals.py        ← เขียน receiver functions ทั้งหมดไว้ที่นี่
├── admin.py
└── migrations/
```

### 184.4 เขียน `signals.py`

```python
# accounts/signals.py
from django.conf import settings
from django.db.models.signals import post_save
from django.dispatch import receiver

from .models import Profile


@receiver(post_save, sender=settings.AUTH_USER_MODEL, dispatch_uid='accounts_create_profile')
def create_profile(sender, instance, created, **kwargs):
    if created:
        Profile.objects.create(user=instance)


@receiver(post_save, sender=settings.AUTH_USER_MODEL, dispatch_uid='accounts_save_profile')
def save_profile(sender, instance, **kwargs):
    if hasattr(instance, 'profile'):
        instance.profile.save()
```

### 184.5 เชื่อมใน `apps.py` ผ่าน `AppConfig.ready()`

```python
# accounts/apps.py
from django.apps import AppConfig


class AccountsConfig(AppConfig):
    default_auto_field = 'django.db.models.BigAutoField'
    name = 'accounts'

    def ready(self):
        import accounts.signals  # noqa: F401
```

**อธิบายทีละส่วน:**

- `ready()` คือเมธอดของ `AppConfig` ที่ Django เรียก **หลังจาก** app registry โหลด
  โมเดลของทุกแอปเสร็จสมบูรณ์แล้ว (ระบบรับประกันว่าตอนนี้ `Model` class ทั้งหมดพร้อมใช้
  งานแล้ว 100%) นี่คือจุดที่ **ปลอดภัยที่สุด** ในการเชื่อม signal
- `import accounts.signals` ภายใน `ready()` ทำให้ decorator `@receiver` ในไฟล์นั้นถูก
  ประมวลผลและเชื่อมกับ signal จริง ๆ ครั้งเดียว ตอนแอปเริ่มทำงาน
- คอมเมนต์ `# noqa: F401` บอก linter (เช่น Ruff, Flake8) ว่า "การ import นี้ตั้งใจ ไม่ได้
  ลืมลบ" เพราะปกติ linter จะเตือนว่า "imported but unused" เนื่องจากเราไม่ได้เรียกใช้
  ชื่อ `accounts.signals` ที่ไหนต่อ — แต่ผลข้างเคียงของการ import (การรัน `@receiver`
  decorator) คือสิ่งที่เราต้องการจริง ๆ

### 184.6 จุดที่มือใหม่พลาดบ่อยที่สุด: ลืมตั้ง `default_app_config` หรือใส่ `AppConfig` ผิดที่

Django ตั้งแต่เวอร์ชัน 3.2 เป็นต้นมา **ตรวจจับ `AppConfig` ในไฟล์ `apps.py` ให้อัตโนมัติ**
โดยไม่ต้องประกาศ `default_app_config` ใน `__init__.py` แบบรุ่นเก่าอีกต่อไป แต่ยังมีจุดที่
ต้องระวัง 2 อย่าง:

1. **ต้องแน่ใจว่า `INSTALLED_APPS` ชี้ไปที่แอปโดยใช้ชื่อโมดูล** (เช่น `'accounts'`) ไม่ใช่
   ชี้ไปที่ `AppConfig` class โดยตรงแบบเก่า — ถ้าใช้ชื่อโมดูลปกติ Django จะหา `apps.py`
   และเรียก `AppConfig` ตัวแรกที่เจอให้เองอัตโนมัติ
2. **ถ้ามีการ custom `default_auto_field` หรือตั้งชื่อ class ของ `AppConfig` เอง** (เช่น
   `AccountsConfig` แทนชื่อ default `AccountsConfig` ที่ `startapp` สร้างให้อยู่แล้ว)
   ต้องแน่ใจว่า `ready()` ถูกเพิ่มเข้าไปใน class เดียวกับที่ Django กำลังใช้งานจริง
   ตรวจสอบได้ด้วยคำสั่ง:

```bash
python manage.py shell -c "from django.apps import apps; print(apps.get_app_config('accounts').__class__)"
```

```
<class 'accounts.apps.AccountsConfig'>
```

### 184.7 ทดสอบว่า Signal ถูกเชื่อมจริงหลังแยกไฟล์

```bash
python manage.py shell
```

```python
>>> from django.contrib.auth.models import User
>>> user4 = User.objects.create_user(username='ประสิทธิ์', password='pass789012')
>>> user4.profile
<Profile: โปรไฟล์ของ ประสิทธิ์>
```

ทำงานเหมือนเดิมทุกประการ แต่ตอนนี้โค้ดมีโครงสร้างที่ปลอดภัยและเป็นมาตรฐานที่ทีมงาน
มืออาชีพทั่วโลกใช้กัน

### 184.8 ตารางสรุป: สามรูปแบบการเชื่อม Signal เทียบกัน

| รูปแบบ | ความปลอดภัยจาก Double-Registration | แนะนำใช้เมื่อไหร่ |
|---|---|---|
| เชื่อมตรงใน `models.py` | ⚠️ เสี่ยงในบางสถานการณ์ | ไม่แนะนำ แม้จะใช้งานได้ในกรณีทั่วไป |
| แยกไฟล์ `signals.py` + import ตรงใน `models.py` ท้ายไฟล์ | ⚠️ ยังเสี่ยงเหมือนเดิม เพราะจุด import ยังอยู่ใน `models.py` | ไม่แนะนำ |
| แยกไฟล์ `signals.py` + import ใน `AppConfig.ready()` | ✅ ปลอดภัยที่สุด ตามเอกสารทางการของ Django | **มาตรฐานที่ควรใช้เสมอ** |

---

## ขั้นตอนที่ 185: `pre_delete`/`post_delete` — ลบไฟล์ Avatar ออกจาก Storage จริง

### 185.1 ปัญหาไฟล์กำพร้า (Orphaned Files)

`Profile.avatar` เป็น `ImageField` ที่เก็บไฟล์จริงไว้ในโฟลเดอร์ `media/avatars/` (ตั้งค่า
ไว้ตั้งแต่ Part 009) แต่มีเรื่องที่มือใหม่มักไม่รู้: **การลบ `Profile` object ออกจาก
ฐานข้อมูล ไม่ได้ลบไฟล์รูปภาพจริงออกจาก storage ให้อัตโนมัติ**

```python
>>> profile = Profile.objects.get(user__username='malee')
>>> profile.avatar
<ImageFieldFile: avatars/malee_photo.jpg>
>>> profile.delete()
(1, {'accounts.Profile': 1})
```

หลัง `delete()` แถวในฐานข้อมูลหายไปแล้ว แต่ไฟล์ `media/avatars/malee_photo.jpg`
**ยังคงอยู่ในระบบไฟล์จริง** กลายเป็น "ไฟล์กำพร้า" (orphaned file) ที่กินพื้นที่ storage
ไปเรื่อย ๆ โดยไม่มีใครอ้างอิงถึงอีกต่อไป ในระบบที่มีผู้ใช้จำนวนมากและมีการลบโปรไฟล์
บ่อย ๆ ปัญหานี้จะสะสมจนกลายเป็นปัญหาใหญ่ด้าน storage cost ในระยะยาว

### 185.2 ทำไม Django ไม่ลบไฟล์ให้อัตโนมัติ

นี่เป็นการตัดสินใจออกแบบของทีม Django อย่างจงใจ ด้วยเหตุผลด้านความปลอดภัยของข้อมูล:
ถ้า Django ลบไฟล์ให้อัตโนมัติทุกครั้งที่ object ถูกลบ จะเกิดความเสี่ยงในสถานการณ์ที่
**หลาย object ใช้ไฟล์เดียวกัน** (เช่น ระบบที่ทำ deduplication ไฟล์ หรือ field สอง field
ที่ชี้ไปยัง path เดียวกัน) การลบไฟล์อัตโนมัติอาจทำให้ object อื่นที่ยังต้องการไฟล์นั้น
เสียหายไปด้วย — Django จึงปล่อยให้เป็นความรับผิดชอบของนักพัฒนาในการตัดสินใจเอง ซึ่งนี่
คือจุดที่ `pre_delete`/`post_delete` signal เข้ามาช่วย

### 185.3 `pre_delete` vs `post_delete` — เลือกใช้ตัวไหนสำหรับลบไฟล์

| Signal | timing | ใช้ลบไฟล์ได้ไหม | เหตุผล |
|---|---|---|---|
| `pre_delete` | ก่อน DELETE จริงในฐานข้อมูล | ✅ ใช้ได้ และเป็นตัวเลือกที่ปลอดภัยกว่า | `instance` ยังสมบูรณ์ 100% รวมถึง path ของไฟล์ ยังไม่มีความเสี่ยงเรื่อง transaction rollback |
| `post_delete` | หลัง DELETE จริงสำเร็จแล้ว | ✅ ใช้ได้เช่นกัน เป็นตัวเลือกที่นิยมมากกว่าในทางปฏิบัติ | มั่นใจได้ 100% ว่าแถวถูกลบจริงแล้ว ก่อนไปยุ่งกับไฟล์ในระบบไฟล์ |

ในทางปฏิบัติ **`post_delete` เป็นตัวเลือกที่นิยมมากกว่า** เพราะถ้าการลบล้มเหลวกลางคัน
(เช่น ติด `PROTECT` ของ foreign key อื่น หรือ transaction ถูก rollback) เราจะไม่ลบไฟล์
ไปโดยไม่จำเป็น — ถ้าใช้ `pre_delete` แล้วการ DELETE ล้มเหลวภายหลัง ไฟล์จะถูกลบไปแล้วทั้ง
ที่แถวข้อมูลยังอยู่ ทำให้เกิด **ข้อมูลไม่สอดคล้องกันแบบย้อนกลับ** (แถวยังอยู่ แต่ไฟล์หาย)
ซึ่งแย่กว่าไฟล์กำพร้าเสียอีก

### 185.4 เขียน Receiver ลบไฟล์ Avatar ด้วย `post_delete`

```python
# accounts/signals.py
import os

from django.conf import settings
from django.db.models.signals import post_delete, post_save
from django.dispatch import receiver

from .models import Profile


@receiver(post_save, sender=settings.AUTH_USER_MODEL, dispatch_uid='accounts_create_profile')
def create_profile(sender, instance, created, **kwargs):
    if created:
        Profile.objects.create(user=instance)


@receiver(post_save, sender=settings.AUTH_USER_MODEL, dispatch_uid='accounts_save_profile')
def save_profile(sender, instance, **kwargs):
    if hasattr(instance, 'profile'):
        instance.profile.save()


@receiver(post_delete, sender=Profile, dispatch_uid='accounts_delete_avatar_file')
def delete_avatar_file(sender, instance, **kwargs):
    """ลบไฟล์ avatar ออกจาก storage จริง เมื่อ Profile ที่เป็นเจ้าของถูกลบ"""
    if instance.avatar:
        if os.path.isfile(instance.avatar.path):
            os.remove(instance.avatar.path)
```

**อธิบายทีละบรรทัด:**

- `if instance.avatar:` เช็คก่อนว่า `Profile` นี้มีไฟล์ avatar อยู่จริงหรือไม่ (field
  ตั้ง `blank=True, null=True` ไว้ตั้งแต่ Part 012 ทำให้บาง `Profile` อาจไม่มีรูปเลย)
- `instance.avatar.path` คือ **absolute path** ของไฟล์ในระบบไฟล์จริง (ต่างจาก
  `instance.avatar.url` ที่เป็น URL สำหรับเข้าถึงผ่านเว็บ)
- `os.path.isfile(...)` ป้องกัน error กรณีไฟล์ถูกลบไปแล้วจากที่อื่น (เช่น ลบมือผ่าน
  server) ก่อนที่ signal จะทำงาน
- `os.remove(...)` คือคำสั่งลบไฟล์จริงของ Python มาตรฐาน (ใช้ได้เมื่อ storage backend
  เป็น local filesystem เท่านั้น — ดูหมายเหตุด้านล่างสำหรับ cloud storage)

### 185.5 กรณีสำคัญที่มักถูกลืม: ลบไฟล์เก่าเมื่อ Avatar ถูก "เปลี่ยน" (ไม่ใช่แค่ตอนลบ Profile)

ปัญหาไฟล์กำพร้าไม่ได้เกิดแค่ตอนลบ `Profile` เท่านั้น แต่เกิดบ่อยกว่านั้นมากตอนที่ผู้ใช้
**อัปโหลดรูปใหม่ทับรูปเก่า** — ถ้าไม่จัดการ ไฟล์เก่าจะค้างอยู่ใน storage ตลอดไปเช่นกัน
เราจึงต้องเสริม `pre_save` เพื่อดักจับกรณีนี้:

```python
@receiver(pre_save, sender=Profile, dispatch_uid='accounts_delete_old_avatar_on_change')
def delete_old_avatar_on_change(sender, instance, **kwargs):
    """ลบไฟล์ avatar เก่า เมื่อมีการอัปโหลดไฟล์ใหม่มาแทนที่ (ก่อน UPDATE จริง)"""
    if not instance.pk:
        return  # เป็นการสร้างใหม่ ยังไม่มีไฟล์เก่าให้เทียบ

    try:
        old_avatar = Profile.objects.get(pk=instance.pk).avatar
    except Profile.DoesNotExist:
        return

    new_avatar = instance.avatar
    if old_avatar and old_avatar != new_avatar:
        if os.path.isfile(old_avatar.path):
            os.remove(old_avatar.path)
```

ต้อง import `pre_save` เพิ่มเข้ามาในบรรทัด import ด้านบนของไฟล์ด้วย:

```python
from django.db.models.signals import post_delete, post_save, pre_save
```

จุดสำคัญของโค้ดนี้คือการ **query ค่าเก่าจากฐานข้อมูลก่อน** (`Profile.objects.get(pk=...)`)
เพราะ `instance` ที่ `pre_save` ได้รับมาคือ instance ใหม่ที่มี `avatar` ตัวใหม่ติดมาแล้ว
เราจึงต้องไปถามฐานข้อมูล (ค่าที่ยังไม่ถูกเขียนทับ) เพื่อรู้ว่า "ค่าเดิมก่อนหน้านี้คืออะไร"

### 185.6 ข้อควรระวังสำหรับ Cloud Storage (S3, Google Cloud Storage)

โค้ดข้างต้นใช้ `os.remove()` ซึ่งทำงานได้เฉพาะเมื่อ `DEFAULT_FILE_STORAGE` เป็น
**local filesystem** (`FileSystemStorage` ค่าเริ่มต้นของ Django) เท่านั้น ถ้าโปรเจกต์
ใช้ cloud storage อย่าง Amazon S3 ผ่าน package `django-storages` ต้องเปลี่ยนมาใช้
`instance.avatar.delete(save=False)` แทน เพราะ Django's `FieldFile.delete()` จะเรียก
ผ่าน **storage backend abstraction** ที่รองรับทั้ง local และ cloud โดยอัตโนมัติ:

```python
@receiver(post_delete, sender=Profile, dispatch_uid='accounts_delete_avatar_file')
def delete_avatar_file(sender, instance, **kwargs):
    """เวอร์ชันที่รองรับทั้ง Local Storage และ Cloud Storage (S3 ฯลฯ)"""
    if instance.avatar:
        instance.avatar.delete(save=False)
```

`save=False` สำคัญมาก เพราะบอก Django ว่า **"ไม่ต้องเรียก `instance.save()` ซ้ำ"**
(ปกติ `FieldFile.delete()` จะพยายาม save instance หลังลบไฟล์ แต่ในบริบทนี้ `instance`
กำลังจะถูกลบทิ้งอยู่แล้วจาก `post_delete` ไม่มีประโยชน์ที่จะ save ซ้ำ และอาจทำให้เกิด
`RecursionError` หรือ query ที่ไม่จำเป็นด้วย) เราจะเจาะลึกการตั้งค่า cloud storage เต็ม
รูปแบบใน Phase 11 (DevOps)

### 185.7 ทดสอบว่าไฟล์ถูกลบจริง

```bash
python manage.py shell
```

```python
>>> from accounts.models import Profile
>>> from django.core.files.uploadedfile import SimpleUploadedFile
>>> profile = Profile.objects.get(user__username='malee')
>>> profile.avatar.save('test.jpg', SimpleUploadedFile('test.jpg', b'fake-image-content'))
>>> import os
>>> os.path.exists(profile.avatar.path)
True
>>> profile.delete()
(1, {'accounts.Profile': 1})
>>> os.path.exists('media/avatars/test.jpg')  # ตรวจสอบด้วย path เดิม
False
```

ไฟล์ถูกลบออกจาก storage จริงตามที่ต้องการ

---

## ขั้นตอนที่ 186: `m2m_changed` — Log การเพิ่ม/ลบ Tag ของ `Post`

### 186.1 ทำไม `m2m_changed` แตกต่างจาก Signal อื่น

`post_save` และ `post_delete` ทำงานกับ **แถวเดียว** ในตารางเดียวเสมอ แต่ความสัมพันธ์
`ManyToManyField` (อย่าง `Post.tags` จาก Part 012) ถูกเก็บผ่าน **ตารางกลาง** ที่ไม่มี
model instance ของตัวเองให้ `save()`/`delete()` ตรง ๆ (ยกเว้นกรณีใช้ `through` model
กำหนดเอง) เมื่อคุณเรียก `post.tags.add(tag)` มันไม่ได้ไปเรียก `save()` ของ `Post` หรือ
`Tag` เลย — Django จึงต้องมี signal เฉพาะสำหรับเหตุการณ์นี้โดยตรง เรียกว่า `m2m_changed`

### 186.2 การเชื่อม `m2m_changed` ต้องระบุ `sender` เป็น Through Model เสมอ

จุดที่แตกต่างจาก signal อื่นอย่างชัดเจนคือ `sender` ของ `m2m_changed` **ไม่ใช่**
`Post` หรือ `Tag` แต่เป็น **through model** ของความสัมพันธ์นั้น (ตารางกลางที่ Django
สร้างให้อัตโนมัติ หรือ `through=` ที่กำหนดเอง) เข้าถึงได้ผ่าน
`<Model>.<m2m_field>.through`:

```python
# blog/signals.py
from django.db.models.signals import m2m_changed
from django.dispatch import receiver

from .models import Post


@receiver(m2m_changed, sender=Post.tags.through, dispatch_uid='blog_log_tag_changes')
def log_tag_changes(sender, instance, action, pk_set, **kwargs):
    ...
```

### 186.3 ทำความเข้าใจ `action` — พารามิเตอร์ที่สำคัญที่สุดของ `m2m_changed`

`m2m_changed` ส่งสัญญาณ**หลายครั้ง**ต่อการเรียก `add()`/`remove()`/`set()`/`clear()`
หนึ่งครั้ง เพราะแต่ละ operation จะยิง signal ทั้งช่วง **ก่อน** และ **หลัง** การเปลี่ยนแปลง
จริง โดยแยกแยะด้วยค่า `action`:

| `action` | เกิดขึ้นตอนไหน |
|---|---|
| `pre_add` | ก่อนเพิ่มความสัมพันธ์ใหม่ลงตารางกลาง (ก่อน `add()` ทำงานจริง) |
| `post_add` | หลังเพิ่มความสัมพันธ์ใหม่สำเร็จ |
| `pre_remove` | ก่อนลบความสัมพันธ์บางส่วนออก (ก่อน `remove()` ทำงานจริง) |
| `post_remove` | หลังลบความสัมพันธ์บางส่วนสำเร็จ |
| `pre_clear` | ก่อนลบความสัมพันธ์ทั้งหมดออก (ก่อน `clear()` ทำงานจริง) |
| `post_clear` | หลังลบความสัมพันธ์ทั้งหมดสำเร็จ |

**สิ่งที่ต้องรู้**: การเรียก `post.tags.set([tag1, tag2])` ภายในจะถูกแปลงเป็นชุดของ
`add()`/`remove()` ที่จำเป็นโดยอัตโนมัติ (Django คำนวณ diff ระหว่างชุดเก่ากับชุดใหม่ให้)
ดังนั้น `set()` หนึ่งครั้งอาจกระตุ้น action หลายแบบพร้อมกัน (`pre_remove`/`post_remove`
สำหรับตัวที่ถูกเอาออก และ `pre_add`/`post_add`` สำหรับตัวที่ถูกเพิ่มเข้ามาใหม่)

### 186.4 พารามิเตอร์อื่นที่สำคัญ: `reverse` และ `pk_set`

| พารามิเตอร์ | ความหมาย |
|---|---|
| `instance` | object ฝั่งที่ถูกเรียก method (เช่น `post` เมื่อเรียก `post.tags.add(...)`) |
| `reverse` | `False` ถ้าเรียกจากฝั่ง forward (`post.tags.add()`), `True` ถ้าเรียกจากฝั่ง reverse (`tag.posts.add()`) |
| `model` | class ของฝั่งตรงข้าม `instance` (เช่น `Tag` ถ้า `instance` คือ `post`) |
| `pk_set` | `set` ของ primary key ฝั่งตรงข้ามที่เกี่ยวข้องกับ action นี้ (เป็น `None` เมื่อ `action='pre_clear'`/`'post_clear'` เพราะยังไม่รู้ว่าจะลบอะไรบ้างจนกว่าจะ clear จริง) |

### 186.5 เขียน Receiver Log การเปลี่ยนแปลง Tag แบบสมบูรณ์

```python
# blog/signals.py
import logging

from django.db.models.signals import m2m_changed
from django.dispatch import receiver

from .models import Post, Tag

logger = logging.getLogger('blog.tags')


@receiver(m2m_changed, sender=Post.tags.through, dispatch_uid='blog_log_tag_changes')
def log_tag_changes(sender, instance, action, reverse, model, pk_set, **kwargs):
    if reverse:
        # ถูกเรียกจากฝั่ง Tag เช่น tag.posts.add(post) — ข้าม log แบบละเอียด
        return

    if action == 'post_add' and pk_set:
        tag_names = Tag.objects.filter(pk__in=pk_set).values_list('name', flat=True)
        logger.info(
            'เพิ่มแท็ก %s เข้าบทความ "%s" (post_id=%s)',
            list(tag_names), instance.title, instance.pk,
        )

    elif action == 'post_remove' and pk_set:
        tag_names = Tag.objects.filter(pk__in=pk_set).values_list('name', flat=True)
        logger.info(
            'ลบแท็ก %s ออกจากบทความ "%s" (post_id=%s)',
            list(tag_names), instance.title, instance.pk,
        )

    elif action == 'post_clear':
        logger.info('ล้างแท็กทั้งหมดของบทความ "%s" (post_id=%s)', instance.title, instance.pk)
```

**จุดสำคัญ**: เราเลือกดักเฉพาะ `post_add`/`post_remove`/`post_clear` (ไม่ใช่ `pre_*`)
เพราะต้องการ log ว่า "การเปลี่ยนแปลงเกิดขึ้นสำเร็จแล้ว" — และเลือก query ชื่อ `Tag` จาก
`pk_set` เฉพาะตอน `post_add`/`post_remove` เพราะตอนนั้น `pk_set` มีค่าจริง (ตอน
`pre_clear`/`post_clear` ค่าเป็น `None` เสมอ ตามที่อธิบายไว้ในตารางข้างต้น)

### 186.6 ทดสอบการทำงาน

```bash
python manage.py shell
```

```python
>>> import logging
>>> logging.basicConfig(level=logging.INFO)
>>> from blog.models import Post, Tag
>>> post = Post.objects.first()
>>> python_tag = Tag.objects.get(name='python')
>>> post.tags.add(python_tag)
INFO:blog.tags:เพิ่มแท็ก ['python'] เข้าบทความ "แนะนำ Django 5" (post_id=1)
>>> post.tags.remove(python_tag)
INFO:blog.tags:ลบแท็ก ['python'] ออกจากบทความ "แนะนำ Django 5" (post_id=1)
```

อย่าลืมเพิ่ม `import blog.signals` ใน `BlogConfig.ready()` ตามรูปแบบที่เรียนในขั้นตอนที่
184 มิฉะนั้น signal นี้จะไม่ถูกเชื่อมเลย:

```python
# blog/apps.py
from django.apps import AppConfig


class BlogConfig(AppConfig):
    default_auto_field = 'django.db.models.BigAutoField'
    name = 'blog'

    def ready(self):
        import blog.signals  # noqa: F401
```

---

## ขั้นตอนที่ 187: สร้าง Custom Signal ของตัวเอง (`django.dispatch.Signal`)

### 187.1 เมื่อไหร่ควรสร้าง Signal ของตัวเอง

Built-in signals ของ Django (`post_save`, `m2m_changed` ฯลฯ) ครอบคลุมเฉพาะเหตุการณ์
ระดับ **database operation** เท่านั้น (INSERT, UPDATE, DELETE) แต่ในทางธุรกิจ มักมี
**เหตุการณ์ระดับแนวคิด (business event)** ที่ไม่ตรงกับการบันทึกข้อมูลโดยตรง เช่น
"บทความถูกเผยแพร่แล้ว" (`post_published`) ซึ่งอาจไม่ได้เกิดขึ้นทุกครั้งที่ `Post` ถูก
`save()` (เพราะการ save อาจเป็นแค่การแก้ typo เล็กน้อยที่ไม่เกี่ยวกับสถานะเผยแพร่เลย)

สำหรับเหตุการณ์แบบนี้ Django ให้เราสร้าง **Custom Signal** ของตัวเองได้ ผ่าน class
`django.dispatch.Signal`

### 187.2 นิยาม Custom Signal

```python
# blog/signals.py
from django.dispatch import Signal

# สัญญาณที่ถูกส่งเมื่อบทความถูกเผยแพร่จริง ๆ (ไม่ใช่แค่ save() ธรรมดา)
post_published = Signal()
```

`Signal()` ไม่ต้องการพารามิเตอร์ตอนสร้าง (ในเวอร์ชันเก่าของ Django ก่อน 3.1 ต้องระบุ
`providing_args=[...]` แต่พารามิเตอร์นี้ถูก deprecate และลบออกไปแล้วตั้งแต่ Django 4.0
เพราะ Python ไม่บังคับตรวจสอบ argument ของ signal ที่ runtime อยู่แล้ว การระบุไว้จึงเป็น
แค่ documentation ที่ไม่มีผลจริง — หลักสูตรนี้จึงไม่ใช้พารามิเตอร์นี้)

### 187.3 ส่งสัญญาณด้วย `.send()`

เราจะเรียก `.send()` ที่จุดในโค้ด **ที่แสดงถึงเหตุการณ์ทางธุรกิจจริง ๆ** เช่น ใน method
ของ model หรือใน service layer (ไม่ใช่ในทุกจุดที่ `save()` ถูกเรียก):

```python
# blog/models.py
from django.utils import timezone

from .signals import post_published


class Post(models.Model):
    # ... field เดิมทั้งหมด ...

    def publish(self):
        """เผยแพร่บทความ และแจ้งเตือนทุกคนที่รับฟัง signal นี้"""
        was_published_before = self.is_published
        self.is_published = True
        self.published_at = timezone.now()
        self.save(update_fields=['is_published', 'published_at'])

        if not was_published_before:
            post_published.send(sender=self.__class__, post=self)
```

> **หมายเหตุ**: การเรียก `publish()` แบบนี้จำเป็นต้องมี field `published_at` เพิ่มเข้าไป
> ใน `Post` (`models.DateTimeField(null=True, blank=True)`) ซึ่งต้องรัน
> `makemigrations`/`migrate` เพิ่มเติมถ้ายังไม่มี field นี้ในโปรเจกต์ของคุณ

`.send()` รับพารามิเตอร์:

| พารามิเตอร์ | ความหมาย |
|---|---|
| `sender` | ตามธรรมเนียม Django มักส่งเป็น **class** ของ object ที่เกี่ยวข้อง (`self.__class__`) เพื่อให้ receiver กรอง `sender=Post` ได้ |
| `**kwargs` | ข้อมูลเพิ่มเติมอะไรก็ได้ที่ต้องการส่งไปให้ receiver (ในตัวอย่างนี้คือ `post=self`) |

**ทางเลือกที่สอง**: `.send_robust()` — ทำงานเหมือน `.send()` ทุกประการ แต่ถ้า receiver
ตัวใดตัวหนึ่ง raise exception ระหว่างทำงาน `.send_robust()` จะ**จับ exception นั้นไว้**
และส่งต่อให้ receiver ตัวถัดไปทำงานต่อได้ตามปกติ (ผลลัพธ์ที่คืนมาจะเป็น list ของ
`(receiver, response_or_exception)` ให้ตรวจสอบเองภายหลัง) ในขณะที่ `.send()` ปกติ
ถ้า receiver ตัวใดตัวหนึ่ง raise exception จะทำให้ **receiver ตัวถัดไปไม่ถูกเรียกเลย**
และ exception จะลอยกลับไปยังจุดที่เรียก `.send()` ทันที

> **คำแนะนำของหลักสูตรนี้**: ใช้ `.send_robust()` เมื่อ receiver หลายตัวเป็นอิสระจากกัน
> (เช่น ตัวหนึ่งส่งอีเมล ตัวหนึ่ง log, ตัวหนึ่งอัปเดต cache) และไม่ต้องการให้ความล้มเหลว
> ของตัวใดตัวหนึ่งไปขวางตัวอื่น — ใช้ `.send()` ปกติเมื่อต้องการให้ error หยุดกระบวนการ
> ทั้งหมดทันทีเพื่อความปลอดภัยของข้อมูล

### 187.4 เขียน Receiver รับฟัง Custom Signal

```python
# blog/signals.py
import logging

from django.dispatch import Signal, receiver

post_published = Signal()

logger = logging.getLogger('blog.publishing')


@receiver(post_published, dispatch_uid='blog_notify_post_published')
def notify_post_published(sender, post, **kwargs):
    logger.info('บทความ "%s" ถูกเผยแพร่แล้วเมื่อ %s', post.title, post.published_at)
    # ในระบบจริงอาจส่งอีเมลแจ้งผู้ติดตาม, โพสต์ลง social media อัตโนมัติ,
    # อัปเดต search index, invalidate cache ของหน้าแรก ฯลฯ
```

สังเกตว่า custom signal **ไม่มี `sender` บังคับให้ระบุตอน `@receiver`** เหมือน built-in
signal (เพราะ custom signal ของเราไม่ได้ผูกกับ model ใด model หนึ่งโดยตรง) แต่ถ้า
ต้องการกรองเฉพาะ sender บางประเภท ก็ยังระบุ `sender=Post` ได้ตามปกติ

### 187.5 ทดสอบ Custom Signal

```bash
python manage.py shell
```

```python
>>> import logging
>>> logging.basicConfig(level=logging.INFO)
>>> from blog.models import Post
>>> post = Post.objects.create(title='ทดสอบ Custom Signal', content='...')
>>> post.publish()
INFO:blog.publishing:บทความ "ทดสอบ Custom Signal" ถูกเผยแพร่แล้วเมื่อ 2026-09-26 10:30:00+00:00
>>> post.publish()  # เรียกซ้ำ — ไม่ควรเห็น log อีก เพราะ was_published_before เป็น True แล้ว
```

### 187.6 ตารางสรุป Built-in Signal เทียบกับ Custom Signal

| ประเด็น | Built-in Signal (`post_save` ฯลฯ) | Custom Signal (`post_published`) |
|---|---|---|
| ผู้สร้าง | ทีมงาน Django | นักพัฒนาแอปเอง |
| จุดที่ถูกส่ง | ภายในโค้ดของ Django ORM/HTTP handler | จุดใดก็ได้ในโค้ดของเราที่เราเลือกเรียก `.send()` เอง |
| ความสัมพันธ์กับ database operation | ผูกติดกับ INSERT/UPDATE/DELETE โดยตรง | ไม่จำเป็นต้องเกี่ยวกับ database เลยก็ได้ (เช่น "ผู้ใช้ login สำเร็จ", "การชำระเงินเสร็จสิ้น") |
| ใช้เมื่อ | ต้องการทำอะไรบางอย่างตาม lifecycle ของ database operation | ต้องการสื่อสาร "เหตุการณ์ทางธุรกิจ" ที่มีความหมายเฉพาะทาง ระหว่างส่วนต่าง ๆ ของระบบที่ไม่อยากให้ผูกกันตรง ๆ |

---

## ขั้นตอนที่ 188: ข้อควรระวังของ Signals — Performance, Hidden Side Effects, ทางเลือก

### 188.1 Performance Overhead ที่มองไม่เห็น

Signal ทุกตัวที่ถูกเชื่อมไว้จะทำงาน **แบบ synchronous (บล็อกรอ)** เสมอ ไม่ว่า receiver
จะทำงานหนักแค่ไหนก็ตาม ลองดูตัวอย่างนี้:

```python
@receiver(post_save, sender=Post)
def slow_receiver(sender, instance, created, **kwargs):
    send_email_to_all_subscribers(instance)  # ใช้เวลา 3 วินาที เพราะเรียก SMTP server
    update_search_index(instance)             # ใช้เวลา 1 วินาที เพราะเรียก Elasticsearch
    generate_thumbnail(instance)              # ใช้เวลา 2 วินาที เพราะประมวลผลรูปภาพ
```

ทุกครั้งที่ `Post.objects.create(...)` ถูกเรียก (แม้จะเป็นแค่จาก Django Admin ธรรมดา)
**ผู้ใช้จะต้องรอ 6 วินาทีเต็ม** ก่อนหน้าเว็บจะตอบกลับ เพราะ `save()` จะไม่ return จนกว่า
signal receiver ทั้งหมดจะทำงานเสร็จสิ้น — นี่คือกับดักสำคัญที่มือใหม่มักมองข้าม เพราะโค้ด
`post.save()` ดูเหมือนเป็นบรรทัดเดียวธรรมดา แต่จริง ๆ อาจแฝงงานหนักไว้ข้างในที่มองไม่เห็น
เลยจากจุดที่เรียก

**ทางแก้**: งานที่ใช้เวลานานหรือเรียก external service (ส่งอีเมล, เรียก API ภายนอก,
ประมวลผลไฟล์ขนาดใหญ่) ควรถูกส่งไปทำงานแบบ **asynchronous** ผ่าน task queue อย่าง
**Celery** แทนที่จะทำใน signal receiver โดยตรง — เราจะเรียนเรื่องนี้เต็มรูปแบบใน
**Part 076 (Celery และ Background Tasks)** ตอนนี้ให้จำหลักการไว้ก่อน: **signal receiver
ควรทำงานให้เร็วที่สุดเท่าที่จะทำได้เสมอ** ถ้าต้องทำงานหนัก ให้แค่ "จองคิว" งานนั้นไว้ใน
task queue แล้วปล่อยให้ worker แยกต่างหากทำงานจริงในภายหลัง

### 188.2 Hidden Side Effects — ปัญหาที่ Debug ยากที่สุดของ Signals

นี่คือข้อเสียเชิงโครงสร้างที่ร้ายแรงที่สุดของ signals: **โค้ดที่เรียก `save()` ไม่มีทาง
รู้เลยว่ามีอะไรเกิดขึ้นตามมาบ้าง** โดยไม่เปิดไฟล์ `signals.py` ดูเอง

```python
# views.py
def update_post_title(request, pk):
    post = Post.objects.get(pk=pk)
    post.title = request.POST['title']
    post.save()   # ← มองจากบรรทัดนี้บรรทัดเดียว ไม่มีทางรู้เลยว่ามี:
                   #   - signal ที่ invalidate cache
                   #   - signal ที่ log การเปลี่ยนแปลงลง audit trail
                   #   - signal ที่ trigger webhook ไปยังระบบภายนอก
                   #   - signal ที่ re-index ข้อมูลใน search engine
                   #   ทั้งหมดนี้ "ซ่อน" อยู่ในไฟล์อื่นที่ห่างไกลจากตรงนี้
```

เมื่อโปรเจกต์มีขนาดใหญ่ขึ้นและมี signal receiver จำนวนมากกระจายอยู่หลายแอป การ debug
ปัญหาอย่าง "ทำไมข้อมูลนี้ถึงถูกอัปเดตโดยที่ไม่มีใครเขียนโค้ดสั่งตรง ๆ" จะยากขึ้นเรื่อย ๆ
เพราะ **flow การทำงานไม่ได้ไหลเป็นเส้นตรงที่อ่านจากบนลงล่างได้อีกต่อไป** ต้องไล่ตามหา
ทุกจุดที่เชื่อม signal เข้ากับ model นั้นทั่วทั้งโปรเจกต์

### 188.3 ตารางสรุป: ข้อดี-ข้อเสียของ Signals

| ด้าน | ข้อดี | ข้อเสีย |
|---|---|---|
| Decoupling | โค้ดแอปต่าง ๆ ไม่ต้องรู้จักกันโดยตรง | ยากที่จะรู้ "ผลกระทบทั้งหมด" ของการเปลี่ยนแปลงหนึ่งจุด |
| การขยายระบบ | เพิ่ม receiver ใหม่ได้โดยไม่แตะโค้ดเดิมเลย | เพิ่ม receiver มากขึ้นเรื่อย ๆ = performance โดยรวมแย่ลงเรื่อย ๆ แบบไม่รู้ตัว |
| การอ่านโค้ด | เหมาะกับ event ที่เกิดจากหลายจุดเรียกจริง (เช่น `User` ถูกสร้างจากหลายทาง) | เส้นทางการทำงาน (control flow) ไม่ต่อเนื่อง ต้องกระโดดไปดูหลายไฟล์ |
| Testing | ทดสอบ receiver แยกเป็นหน่วยเดียวได้ | Unit test ของโค้ดหลักอาจ trigger side effect ที่ไม่ตั้งใจ (เช่น ส่งอีเมลจริงตอนรัน test) |

### 188.4 เมื่อไหร่ควรใช้ Signal เมื่อไหร่ควรใช้ Explicit Function Call ใน Service Layer แทน

นี่คือคำถามที่สำคัญที่สุดของขั้นตอนนี้ และเป็นสิ่งที่แยกนักพัฒนา Django มือใหม่กับมือ
อาชีพออกจากกันอย่างชัดเจน:

| สถานการณ์ | ควรใช้ Signal | ควรใช้ Explicit Function Call |
|---|---|---|
| Event เกิดจากหลายจุดเรียกที่เราแก้โค้ดไม่ได้ (เช่น `createsuperuser`, Django Admin, migration) | ✅ ใช่ — เป็นทางเดียวที่ครอบคลุมทุกเส้นทาง | ❌ ทำไม่ได้ เพราะแก้โค้ดของ Django เองไม่ได้ |
| Business logic ที่ซับซ้อน ต้องการ transaction แบบมีเงื่อนไข ควบคุม order การทำงาน | ❌ ควบคุม order ยาก เพราะ Django ไม่รับประกันลำดับการเรียก receiver หลายตัว | ✅ ใช่ — เขียนใน service function ที่เรียกทีละขั้นตอนชัดเจน อ่านง่าย debug ง่าย |
| งานที่ต้องการให้ error หยุดกระบวนการทั้งหมดทันที และรายงาน error ที่ชัดเจน | ⚠️ ทำได้ แต่ error message มักงงเพราะ stack trace ลึกเข้าไปใน signal dispatcher | ✅ ใช่ — โค้ดตรงไปตรงมา อ่าน stack trace เข้าใจง่ายกว่ามาก |
| Cross-cutting concern ที่ไม่เกี่ยวกับ business logic หลัก (logging, cache invalidation, audit trail) | ✅ ใช่ — เหมาะกับ signal เพราะเป็นงาน "แถม" ที่ไม่ควรปนกับ logic หลัก | ⚠️ ทำได้ แต่จะทำให้ฟังก์ชันหลักรกด้วยโค้ดที่ไม่ใช่ core logic |

**ตัวอย่างเปรียบเทียบจริง**: สมมติต้องเขียนระบบ "เมื่อคำสั่งซื้อชำระเงินสำเร็จ ต้อง (1)
ลดสต็อกสินค้า (2) ส่งอีเมลยืนยัน (3) สร้างใบเสร็จ (4) แจ้งระบบบัญชีภายนอก" — ถ้าเขียน
ด้วย signal ทั้งหมด:

```python
# ❌ ไม่แนะนำสำหรับ business logic ที่ซับซ้อนและมีลำดับความสำคัญชัดเจน
@receiver(post_save, sender=Order)
def handle_order_paid(sender, instance, **kwargs):
    if instance.status == 'paid':
        reduce_stock(instance)          # ถ้าล้มเหลว จะเกิดอะไรกับ 3 ข้อถัดไป?
        send_confirmation_email(instance)
        generate_receipt(instance)
        notify_accounting_system(instance)
```

ปัญหาคือถ้า `reduce_stock()` ล้มเหลว (เช่น สต็อกไม่พอ) เราต้องการ **หยุดทั้งกระบวนการ
และ rollback การชำระเงิน** แต่ signal ไม่ได้ถูกออกแบบมาเพื่อควบคุม flow แบบนี้ได้ดีนัก
ทางที่ดีกว่าคือเขียนเป็น **service function** ที่ชัดเจน:

```python
# ✅ แนะนำ — service layer function ที่อ่าน flow ได้ตรงไปตรงมา
# orders/services.py
from django.db import transaction


def mark_order_as_paid(order):
    with transaction.atomic():
        reduce_stock(order)              # ถ้าล้มเหลว raise exception ทันที
        order.status = 'paid'
        order.save(update_fields=['status'])
        generate_receipt(order)
        send_confirmation_email(order)   # งานนี้อาจแยกไป Celery task แทน (Part 076)
        notify_accounting_system(order)  # งานนี้ก็เช่นกัน
```

โค้ดแบบนี้อ่านจากบนลงล่างได้ตรงไปตรงมา ควบคุม transaction ได้ชัดเจน และถ้าเกิด error
stack trace จะชี้ตรงไปที่บรรทัดที่ผิดพลาดจริงทันที ไม่ต้องไล่ตามหาไฟล์ signal ที่ไหน

> **หลักการสรุปของหลักสูตรนี้**: ใช้ Signal สำหรับ **"ผลข้างเคียงที่เป็นทางเลือก"**
> (optional side effect) ที่ไม่กระทบความถูกต้องของ business logic หลัก และเกิดจาก
> เหตุการณ์ที่มาจากหลายจุดเรียกที่ควบคุมไม่ได้ทั้งหมด (เหมือนกรณี auto-create `Profile`
> ในขั้นตอนที่ 183) ส่วน **business logic หลักที่มีลำดับขั้นตอนชัดเจนและต้องควบคุม
> transaction** ให้เขียนเป็น explicit function ใน service layer เสมอ

---

## ขั้นตอนที่ 189: Signal อื่น ๆ ที่ควรรู้: `request_started`, `request_finished`, `got_request_exception`

### 189.1 Request/Response Signals คืออะไร

นอกจาก model signals ที่เราเรียนไปแล้ว Django ยังมี signal ระดับ **HTTP request/response
cycle ทั้งก้อน** อยู่ใน `django.core.signals` ซึ่งทำงานที่ระดับ WSGI/ASGI handler
ไม่ใช่ระดับ model:

| Signal | เกิดขึ้นเมื่อไหร่ | Arguments |
|---|---|---|
| `request_started` | ทันทีที่ Django เริ่มประมวลผล HTTP request ใหม่ (ก่อนเข้า middleware ใด ๆ) | `sender` (WSGI/ASGI handler class) |
| `request_finished` | หลังจาก Django ส่ง HTTP response กลับไปเรียบร้อยแล้ว | `sender` |
| `got_request_exception` | เมื่อเกิด exception ที่ไม่ถูกจัดการระหว่างประมวลผล request (unhandled exception) | `sender`, `request` |

### 189.2 ตัวอย่างการใช้งานจริง: วัดเวลาประมวลผล Request แบบหยาบ

```python
# core/signals.py
import logging
import time

from django.core.signals import got_request_exception, request_finished, request_started
from django.dispatch import receiver

logger = logging.getLogger('core.requests')

_request_start_times = {}


@receiver(request_started, dispatch_uid='core_log_request_started')
def log_request_started(sender, **kwargs):
    logger.debug('เริ่มประมวลผล request ใหม่')


@receiver(request_finished, dispatch_uid='core_log_request_finished')
def log_request_finished(sender, **kwargs):
    logger.debug('ประมวลผล request เสร็จสิ้น')


@receiver(got_request_exception, dispatch_uid='core_log_unhandled_exception')
def log_unhandled_exception(sender, request, **kwargs):
    logger.error('เกิด exception ที่ไม่ถูกจัดการ ใน request: %s %s', request.method, request.path)
```

> **ข้อควรระวัง**: `request_started`/`request_finished` **ไม่ได้ให้ object `request`
> ตรง ๆ** ผ่าน `kwargs` (ต่างจาก `got_request_exception` ที่ให้ `request` มาด้วย) เพราะ
> สองสัญญาณนี้ถูกออกแบบมาให้ทำงานที่ระดับต่ำมาก (ก่อน/หลัง request object ถูกสร้างเต็ม
> รูปแบบด้วยซ้ำในบางกรณี) จึงเหมาะกับงานระดับ infrastructure เช่น**ปิด database
> connection ที่ไม่ได้ใช้งาน** (ซึ่งเป็นสิ่งที่ Django เองใช้ signal นี้ภายในอยู่แล้วเพื่อ
> จัดการ connection pooling) มากกว่างานที่ต้องรู้รายละเอียดของ request

### 189.3 เมื่อไหร่ควรใช้ Signal เหล่านี้ในทางปฏิบัติ

ในทางปฏิบัติ **middleware เป็นเครื่องมือที่เหมาะสมกว่า** signal เหล่านี้สำหรับงานส่วนใหญ่
ที่เกี่ยวกับ request/response (เช่น วัดเวลา, logging, การจัดการ exception) เพราะ
middleware **เข้าถึง `request` และ `response` object ได้เต็มรูปแบบ** และควบคุม flow
ได้ดีกว่ามาก (เราจะเรียน Middleware เต็มรูปแบบใน **Part 024**) signal กลุ่มนี้จึงเหมาะกับ
กรณีพิเศษที่ค่อนข้างจำกัด เช่น:

- **ปิด resource ที่เปิดไว้ระดับ process** ที่ไม่ผูกกับ request ใดโดยเฉพาะ (เช่น
  Django เองใช้ `request_finished` เพื่อปิด database connection ที่หมดอายุ)
- **เขียน third-party package/library ที่ต้อง hook เข้ากับทุกโปรเจกต์** โดยไม่รู้ล่วงหน้า
  ว่าโปรเจกต์นั้นมี middleware อะไรติดตั้งอยู่บ้าง — signal จึงเป็นจุดเกาะที่แน่นอนกว่า
- **ระบบ monitoring/APM (Application Performance Monitoring)** ระดับต่ำที่ต้องการ hook
  เข้ากับทุก request โดยไม่ขึ้นกับลำดับ middleware ในโปรเจกต์นั้น ๆ

### 189.4 ตารางสรุป: Signal ระดับ Request เทียบกับ Middleware

| ประเด็น | Request/Response Signals | Middleware |
|---|---|---|
| เข้าถึง `request`/`response` เต็มรูปแบบ | ⚠️ จำกัด (`got_request_exception` เท่านั้นที่ให้ `request`) | ✅ เข้าถึงได้เต็มรูปแบบทั้งคู่ |
| ควบคุมลำดับการทำงาน | ❌ ไม่รับประกันลำดับ receiver | ✅ ควบคุมลำดับผ่าน `MIDDLEWARE` list ใน settings ได้ชัดเจน |
| แก้ไข response ก่อนส่งกลับผู้ใช้ | ❌ ทำไม่ได้ | ✅ ทำได้เต็มที่ |
| เหมาะกับ | Library/package ระดับ infrastructure, monitoring แบบ hook เข้าทุกโปรเจกต์ | งานทั่วไประดับ request/response ในโปรเจกต์ของเราเอง (แนะนำเป็นค่าเริ่มต้น) |

---

## ขั้นตอนที่ 190: สรุปและแบบฝึกหัด

### 190.1 สรุปสิ่งที่ได้เรียนรู้ใน Part นี้

- ✅ เข้าใจแนวคิด Observer Pattern เบื้องหลัง Signals และรายการ built-in signals หลัก
  (`pre_save`, `post_save`, `pre_delete`, `post_delete`, `m2m_changed`, `pre_init`,
  `post_init`)
- ✅ เชื่อม signal receiver ด้วย `@receiver` decorator และเข้าใจความแตกต่างจาก
  `.connect()` แบบ manual รวมถึงประโยชน์ของ `dispatch_uid`
- ✅ **แก้ปัญหาจริงจาก Part 012** — สร้าง `Profile` ให้อัตโนมัติทุกครั้งที่มี `User` ใหม่
  ด้วย `post_save` และเข้าใจความสำคัญของ `created` flag อย่างถ่องแท้
- ✅ รู้จักตำแหน่งไฟล์ `signals.py` ที่ถูกต้อง และวิธีเชื่อมผ่าน `AppConfig.ready()` เพื่อ
  ป้องกันปัญหา double-registration
- ✅ ใช้ `pre_save`/`post_delete` ลบไฟล์ avatar ออกจาก storage จริง ทั้งกรณีลบ `Profile`
  และกรณีอัปโหลดไฟล์ใหม่ทับไฟล์เก่า
- ✅ ใช้ `m2m_changed` เพื่อ log การเพิ่ม/ลบ tag ของ `Post` และเข้าใจความหมายของ `action`,
  `reverse`, `pk_set`
- ✅ สร้าง Custom Signal ของตัวเอง (`post_published`) ด้วย `django.dispatch.Signal` และ
  `.send()`/`.send_robust()`
- ✅ เข้าใจข้อควรระวังสำคัญของ signals: performance overhead จากการทำงานแบบ synchronous,
  hidden side effects ที่ debug ยาก, และหลักการเลือกระหว่าง signal กับ explicit function
  call ใน service layer
- ✅ รู้จัก request/response signals (`request_started`, `request_finished`,
  `got_request_exception`) และเข้าใจว่าทำไม middleware มักเหมาะกว่าสำหรับงานทั่วไป

### 190.2 Checklist ก่อนไป Part ถัดไป

- [ ] สร้างไฟล์ `accounts/signals.py` พร้อม `create_profile` และ `save_profile` receiver
- [ ] เชื่อม signal ใน `accounts/apps.py` ผ่าน `AppConfig.ready()` (ไม่ใช่ใน `models.py`)
- [ ] ทดสอบสร้าง `User` ใหม่ผ่าน shell แล้วเรียก `user.profile` สำเร็จโดยไม่มี
      `RelatedObjectDoesNotExist`
- [ ] ทดสอบสร้าง `User` ผ่าน `createsuperuser` แล้วตรวจสอบว่ามี `Profile` ให้อัตโนมัติ
- [ ] เพิ่ม `delete_avatar_file` (`post_delete`) และ `delete_old_avatar_on_change`
      (`pre_save`) ใน `accounts/signals.py`
- [ ] สร้างไฟล์ `blog/signals.py` พร้อม `log_tag_changes` (`m2m_changed`) และเชื่อมใน
      `blog/apps.py`
- [ ] สร้าง custom signal `post_published` และเรียก `.send()` จาก method `Post.publish()`
- [ ] อธิบายได้ด้วยคำพูดตัวเองว่าเมื่อไหร่ควรใช้ signal เมื่อไหร่ควรใช้ explicit function
      call ใน service layer

### 190.3 แบบฝึกหัดท้ายบท

**แบบฝึกหัดที่ 1**: implement ระบบ auto-create `Profile` และ cleanup avatar file signal
ให้ครบถ้วนตามที่สอนในขั้นตอนที่ 183-185 ในโปรเจกต์จริงของคุณ แล้วเขียนเทสสั้น ๆ (ยังไม่
ต้องใช้ `TestCase` เต็มรูปแบบ — เราจะเรียน Testing ใน Part 059) ผ่าน `python manage.py
shell` เพื่อพิสูจน์ 3 กรณี: (1) `User` ใหม่มี `Profile` ทันที (2) ลบ `Profile` ที่มี
avatar แล้วไฟล์หายจริงจาก `media/avatars/` (3) อัปโหลด avatar ใหม่ทับของเก่าแล้วไฟล์เก่า
ถูกลบออกไป

**แบบฝึกหัดที่ 2**: เพิ่ม field `published_at = models.DateTimeField(null=True,
blank=True)` ให้ `Post` (พร้อมรัน migration) แล้ว implement `post_published` custom
signal ให้ครบตามขั้นตอนที่ 187 จากนั้นเขียน receiver เพิ่มอีก 1 ตัวที่ทำงานเมื่อบทความ
ถูกเผยแพร่ ให้ print ข้อความ "กำลังส่งการแจ้งเตือนไปยังผู้ติดตาม X คน" (สมมติจำนวนคนเอง
ก็ได้ ยังไม่ต้องมีระบบผู้ติดตามจริง เดี๋ยวเราจะสร้างใน Phase 4)

**แบบฝึกหัดที่ 3**: ลองจงใจสร้างสถานการณ์ปัญหาจากขั้นตอนที่ 183.3 ด้วยตัวเอง — เขียน
receiver ที่ไม่เช็ค `if created:` แล้วสังเกตว่าเกิด `IntegrityError` อย่างไรเมื่อมีการ
อัปเดต `User` ที่มี `Profile` อยู่แล้ว จากนั้นแก้ไขให้ถูกต้อง และเขียนอธิบายด้วยคำพูดของ
ตัวเอง (บันทึกลงไฟล์ `notes.md`) ว่าทำไม `created` flag ถึงสำคัญขนาดนี้

**แบบฝึกหัดที่ 4 (ขั้นสูง)**: ออกแบบสถานการณ์ธุรกิจของตัวเอง (เช่น ระบบ "เมื่อคอมเมนต์
ใหม่ถูกโพสต์ ให้แจ้งเตือนเจ้าของบทความ") แล้วเขียนออกมา **สองแบบ**: แบบแรกใช้ Signal
(`post_save` บน `Comment`) แบบที่สองใช้ Explicit Function Call ในมุมมอง service layer
(สมมติเรียกจาก view โดยตรง) เปรียบเทียบข้อดี-ข้อเสียของทั้งสองแบบในสถานการณ์นี้โดยเฉพาะ
แล้วสรุปว่าคุณจะเลือกแบบไหนสำหรับ production จริง พร้อมให้เหตุผลอ้างอิงจากตารางในขั้นตอน
ที่ 188.4

### 190.4 คำถามที่พบบ่อย (FAQ)

**Q: ทำไม signal receiver ของฉันไม่ทำงานเลย ทั้งที่เขียนโค้ดถูกต้องตามตัวอย่างทุกอย่าง?**
A: สาเหตุที่พบบ่อยที่สุดคือ **ลืม import `signals.py` ใน `AppConfig.ready()`** ตรวจสอบ
ให้แน่ใจว่า (1) มีเมธอด `ready()` ใน `apps.py` ของแอปนั้นจริง (2) มีบรรทัด
`import <app>.signals` อยู่ข้างใน (3) `INSTALLED_APPS` ใน `settings.py` ชี้ไปที่แอปนั้น
ด้วยชื่อโมดูลปกติ (ไม่ใช่ path ไปที่ `AppConfig` class โดยตรงแบบเก่า) ลองตรวจสอบด้วย
`print()` ธรรมดาในฟังก์ชัน `ready()` เพื่อยืนยันว่ามันถูกเรียกจริงตอนรัน server

**Q: signal ทำงานตอนรัน `python manage.py shell` แต่ไม่ทำงานตอนรัน data migration
หรือ management command บางตัว ทำไม?**
A: บาง management command (โดยเฉพาะที่เขียนเองและมีการปิด app loading บางส่วน หรือใช้
`--run-syncdb` แบบพิเศษ) อาจไม่โหลด `AppConfig.ready()` ในลำดับปกติ นอกจากนี้
`loaddata`/fixtures จะส่ง `post_save` พร้อม `raw=True` เสมอ (ดูขั้นตอนที่ 181.3 ตาราง
arguments ของ `post_save`) ซึ่งหมายความว่า instance ที่ได้รับมาอาจ**ยังไม่ผ่านการ
resolve ความสัมพันธ์ทั้งหมด** — ทางแก้คือเช็ค `if raw: return` ไว้ต้น receiver เพื่อข้าม
การทำงานตอนโหลด fixture เสมอ:

```python
@receiver(post_save, sender=settings.AUTH_USER_MODEL, dispatch_uid='accounts_create_profile')
def create_profile(sender, instance, created, raw, **kwargs):
    if raw:
        return  # กำลังโหลดจาก fixture — ข้าม ไม่สร้าง Profile ที่นี่
    if created:
        Profile.objects.create(user=instance)
```

**Q: ใช้ signal คู่กับ `bulk_create()`/`bulk_update()`/`QuerySet.update()` ได้ไหม?**
A: **ไม่ได้** — นี่คือข้อจำกัดสำคัญที่ต้องจำให้ขึ้นใจ: `QuerySet.update()`,
`bulk_create()`, และ `bulk_update()` **ไม่เรียก `save()` ของแต่ละ instance เลย** (นั่นคือ
เหตุผลที่มันเร็วกว่าการ loop `save()` ทีละตัวมาก) ดังนั้น `pre_save`/`post_save` **จะไม่
ถูกส่งสัญญาณเลย** เมื่อใช้ method เหล่านี้ ถ้าโปรเจกต์ของคุณพึ่งพา signal สำหรับ logic
สำคัญ (เช่น auto-create `Profile`) และมีบางจุดในโค้ดที่ใช้ `bulk_create()` สร้าง `User`
จำนวนมาก ต้องเขียนโค้ดสร้าง `Profile` ให้ครบด้วยมือแยกต่างหากเสมอ — เราจะเจาะลึกเรื่องนี้
อีกครั้งเมื่อเรียน `bulk_create()` เต็มรูปแบบใน Part 013 (Advanced QuerySets)

**Q: ควรเขียน unit test ให้ signal receiver อย่างไร?**
A: ทดสอบได้ 2 ระดับ: (1) **ทดสอบฟังก์ชัน receiver โดยตรง** เหมือนฟังก์ชันธรรมดา โดยเรียก
มันด้วย argument ที่จำลองขึ้นเอง ไม่ต้องพึ่ง signal dispatcher จริง (2) **ทดสอบผ่าน
integration** โดยสร้าง object จริงแล้วตรวจสอบผลลัพธ์ปลายทาง (เช่น สร้าง `User` แล้วเช็คว่า
`user.profile` มีอยู่จริง) วิธีที่สองมักดีกว่าเพราะทดสอบ "พฤติกรรมจริงของระบบ" ไม่ใช่แค่
"โค้ดฟังก์ชันทำงานถูกไวยากรณ์" เราจะเจาะลึกการเขียนเทสสำหรับ signals โดยเฉพาะใน
**Part 059-065 (Testing & Quality Assurance)**

---

## เตรียมตัวสำหรับ Part ถัดไป

**Part 020: Multiple Databases และ Database Routing** จะพาไปสำรวจสถาปัตยกรรมฐานข้อมูล
ระดับสูงขึ้นไปอีกขั้น — การเชื่อมต่อ Django กับ **ฐานข้อมูลหลายตัวพร้อมกัน** ในโปรเจกต์
เดียว (เช่น แยก database สำหรับ analytics ออกจาก database หลักของแอป), การเขียน
**Database Router** เพื่อควบคุมว่า query แต่ละตัวควรวิ่งไปที่ฐานข้อมูลไหน, และการจัดการ
migration ข้ามหลายฐานข้อมูลอย่างปลอดภัย นี่คือหัวข้อสุดท้ายของ Phase 2 ก่อนที่เราจะก้าวสู่
**Phase 3: Views, Templates, Forms และ Class-Based Views** ซึ่งจะเปลี่ยนโฟกัสจากการ
ออกแบบข้อมูลไปสู่การสร้างหน้าเว็บที่ผู้ใช้จริงจะได้สัมผัส

ก่อนไปต่อ ลองทบทวนดูว่าตอนนี้โปรเจกต์บล็อกของคุณมี Model ที่เชื่อมกันครบสมบูรณ์แล้ว
(`Category`, `Post`, `Tag`, `PostTag`, `Comment`, `Profile`) และไม่มี `User` ตัวไหน
ขาด `Profile` อีกต่อไป — นี่คือรากฐานที่มั่นคงที่เราจะต่อยอดไปเรื่อย ๆ ตลอดหลักสูตรนี้
