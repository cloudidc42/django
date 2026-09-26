# Part 032: Custom User Model

> **ขั้นตอนที่ 311-320 ของหลักสูตร** | Phase 4: Authentication, Users และ Permissions
>
> เป้าหมายของ Part นี้: เข้าใจการตัดสินใจที่ **สำคัญที่สุดอย่างหนึ่งในชีวิตของโปรเจกต์
> Django** นั่นคือเรื่อง Custom User Model คุณจะเรียนรู้ว่าทำไมการตัดสินใจนี้ต้องทำ
> "ตั้งแต่ต้น" ก่อนรัน `migrate` ครั้งแรก, ความแตกต่างระหว่าง `AbstractUser` กับ
> `AbstractBaseUser`, วิธีสร้าง Custom User Model ทั้งสองแบบด้วยโค้ดที่รันได้จริง
> (รวมถึงระบบ login ด้วยอีเมลแทน username), การเขียน Custom `UserManager`, ผลกระทบต่อ
> migration ผ่าน `swappable_dependency`, ความเสี่ยงของการย้ายโปรเจกต์ที่มีอยู่แล้วไปใช้
> Custom User Model, กฎเหล็ก `get_user_model()`/`settings.AUTH_USER_MODEL`, และการ
> ปรับแต่ง Django Admin ให้รองรับ User model แบบใหม่ทั้งหมด เมื่อจบ Part นี้ คุณจะสามารถ
> ตัดสินใจเรื่องนี้ได้อย่างมั่นใจตั้งแต่วันแรกของทุกโปรเจกต์ในอนาคต

---

## สารบัญของ Part นี้

- ขั้นตอนที่ 311: ทำไมควรตัดสินใจเรื่อง Custom User Model ตั้งแต่ต้นโปรเจกต์
- ขั้นตอนที่ 312: `AbstractUser` vs `AbstractBaseUser` — เมื่อไหร่ควรใช้อะไร
- ขั้นตอนที่ 313: สร้าง Custom User ด้วย `AbstractUser` เทียบกับแนวทาง Profile จาก Part 012
- ขั้นตอนที่ 314: ตั้งค่า `AUTH_USER_MODEL` และผลกระทบต่อ migration
- ขั้นตอนที่ 315: เขียน Custom `UserManager` ของตัวเอง
- ขั้นตอนที่ 316: Custom User แบบเต็มรูปแบบด้วย `AbstractBaseUser` — login ด้วยอีเมล
- ขั้นตอนที่ 317: การย้ายโปรเจกต์ที่มีอยู่แล้วไปใช้ Custom User Model
- ขั้นตอนที่ 318: `get_user_model()` เทียบกับการ import `User` ตรง ๆ
- ขั้นตอนที่ 319: ปรับแต่ง `UserAdmin` ให้รองรับ Custom User Model
- ขั้นตอนที่ 320: สรุปและแบบฝึกหัด

---

## ขั้นตอนที่ 311: ทำไมควรตัดสินใจเรื่อง Custom User Model ตั้งแต่ต้นโปรเจกต์

### 311.1 ทบทวนสถานะโปรเจกต์ก่อนเริ่ม Part นี้

โปรเจกต์บล็อกที่คุณสร้างต่อเนื่องมาตั้งแต่ Part 004 ใช้ Django `User` model มาตรฐาน
(`django.contrib.auth.models.User`) และรัน `migrate` ครั้งแรกไปแล้วตั้งแต่ Part 004
ต่อมาใน Part 012 คุณขยายผู้ใช้ด้วย `Profile` ผ่าน `OneToOneField` เพื่อเพิ่ม `bio` และ
`avatar` โดยไม่แตะต้อง `User` เดิมเลย — ตอนนั้นมีข้อความเตือนไว้แล้วว่า:

> "Custom User Model ต้องตั้งค่าไว้**ก่อน**รัน `migrate` ครั้งแรกของโปรเจกต์เท่านั้น
> เปลี่ยนทีหลังยากมาก"

Part นี้จะพาไปเรียนรู้ Custom User Model แบบเต็มรูปแบบ **แต่เนื่องจากโปรเจกต์บล็อกหลัก
migrate ไปไกลแล้วและมีข้อมูลผู้ใช้อยู่ในระบบ** เราจะไม่ย้อนกลับไปเปลี่ยน `AUTH_USER_MODEL`
ของโปรเจกต์บล็อก (นั่นคือสิ่งที่ Part นี้กำลังจะอธิบายว่าทำไมมันอันตราย) แทนที่จะทำแบบนั้น
เราจะ:

1. สร้าง **โปรเจกต์ทดลองแยกต่างหาก** (sandbox project) เพื่อฝึกสร้าง Custom User Model
   ตั้งแต่ต้นแบบปลอดภัย 100% — นี่คือสิ่งที่คุณควรทำจริงกับทุกโปรเจกต์ใหม่ในอนาคต
2. อธิบายเทคนิคการย้ายโปรเจกต์ที่มีอยู่แล้ว (ขั้นตอนที่ 317) ในเชิงทฤษฎีและโค้ดจริง
   สำหรับกรณีที่คุณเจอสถานการณ์นี้ในงานจริง แต่จะไม่นำไปใช้กับโปรเจกต์บล็อกหลักของเรา
3. โปรเจกต์บล็อกหลักจะยังคงใช้ `User` มาตรฐาน + `Profile` (Part 012) ต่อไปตลอดหลักสูตร
   ซึ่งเป็นทางเลือกที่ถูกต้องเมื่อ migrate ไปไกลแล้ว

### 311.2 ปัญหาที่แท้จริง: `ForeignKey` ไปยัง `User` กระจายอยู่ทั่วโปรเจกต์

ลองนึกภาพโปรเจกต์ Django ขนาดกลางหลังผ่านไปสัก 6 เดือน คำว่า "ผู้ใช้" จะไม่ได้ถูกอ้างอิง
แค่จุดเดียว แต่กระจายอยู่แทบทุกมุมของระบบ:

```
blog/models.py         → Post.author = ForeignKey(User)
blog/models.py         → Comment.author = ForeignKey(User)
shop/models.py         → Order.customer = ForeignKey(User)
shop/models.py         → Review.user = ForeignKey(User)
accounts/models.py     → Profile.user = OneToOneField(User)
notifications/models.py → Notification.recipient = ForeignKey(User)
audit/models.py        → AuditLog.actor = ForeignKey(User)
django.contrib.admin   → LogEntry.user = ForeignKey(User)   (ภายใน Django เอง!)
django.contrib.auth    → Permission ↔ User (ผ่าน Group, user_permissions)
django-allauth          → EmailAddress.user = ForeignKey(User)
Django REST Framework   → Token.user = OneToOneField(User)  (ถ้าใช้ TokenAuthentication)
```

นี่ยังไม่นับ **migration files ที่ apply ไปแล้วทุกไฟล์** ซึ่งบันทึกไว้อย่างถาวรในฐานข้อมูล
(ตาราง `django_migrations`) ว่าแต่ละ `ForeignKey` ชี้ไปที่ตารางไหน และ **ตาราง
`django_content_type`** ที่เก็บว่า "model `user` อยู่ใน app `auth`" ซึ่งถูกอ้างอิงต่อโดย
ระบบ Permission ทั้งหมดของ Django

### 311.3 ทำไมการเปลี่ยนกลางทางถึง "ยากมาก" ไม่ใช่แค่ "ยาก"

คำเตือนจากเอกสารทางการของ Django ระบุไว้ชัดเจนมาก:

> "Changing `AUTH_USER_MODEL` after you've created database tables is significantly
> more difficult since it affects many pieces of Django... This change is
> particularly difficult if you need to change it for an app with existing data,
> so plan and test this change carefully."
> — Django Documentation, [Using a custom user model when starting a project](https://docs.djangoproject.com/en/5.1/topics/auth/customizing/)

ปัญหาไม่ใช่แค่ "แก้โค้ด" แต่เป็นปัญหาระดับ**ข้อมูลจริงในฐานข้อมูล**:

- ถ้าคุณเปลี่ยน `AUTH_USER_MODEL` จาก `auth.User` เป็น `accounts.CustomUser` หลังจากมี
  ผู้ใช้จริงในระบบแล้ว Django จะพยายามสร้าง**ตารางใหม่** (`accounts_customuser`) ที่ไม่มี
  ข้อมูลผู้ใช้เดิมอยู่เลย
- `ForeignKey` ทุกตัวที่ชี้ไป `auth_user.id` เดิม (เช่น `blog_post.author_id`) จะยังคงเก็บ
  ตัวเลข id เดิมไว้ แต่ไม่มีแถวในตารางใหม่ที่ id ตรงกันอีกต่อไป → **ข้อมูลเสียหายทันที**
  (orphaned foreign keys)
- Permission และ Group ที่ผูกกับ `ContentType` ของ `auth.User` เดิมจะใช้งานไม่ได้กับ
  model ใหม่
- Session ที่ล็อกอินอยู่ของผู้ใช้จริงทุกคนจะขาดการเชื่อมโยง (ผู้ใช้ทุกคนต้องล็อกอินใหม่
  เป็นอย่างน้อย)

การแก้ปัญหานี้ให้ถูกต้อง 100% ต้องเขียน **data migration ที่กำหนดเอง** อย่างระมัดระวัง
(รายละเอียดในขั้นตอนที่ 317) ซึ่งใช้เวลาและความเสี่ยงสูงกว่าการวางแผนไว้ล่วงหน้ามาก

### 311.4 กฎการตัดสินใจของหลักสูตรนี้

| สถานการณ์ | คำแนะนำ |
|---|---|
| โปรเจกต์ใหม่ ยังไม่เคยรัน `migrate` เลย | **สร้าง Custom User Model เสมอ** แม้ยังไม่รู้ว่าจะต้องใช้ field พิเศษหรือไม่ (ต้นทุนแทบเป็นศูนย์ ผลตอบแทนสูงมากในระยะยาว) |
| โปรเจกต์อยู่ระหว่างพัฒนา (dev/staging เท่านั้น ยังไม่มีผู้ใช้จริง) | ยังพอทำได้: ลบฐานข้อมูลและไฟล์ migration ทั้งหมด แล้วเริ่ม Custom User Model ตั้งแต่ต้น |
| โปรเจกต์ production ที่มีผู้ใช้จริงแล้ว | หลีกเลี่ยงการเปลี่ยน `AUTH_USER_MODEL` อย่างยิ่ง ใช้ `Profile` แบบ `OneToOneField` (Part 012) แทนสำหรับเพิ่มข้อมูล หรือวางแผน migration ขนาดใหญ่แบบมีทีมและ maintenance window (ขั้นตอนที่ 317) เฉพาะเมื่อจำเป็นจริง ๆ เท่านั้น |

> **กฎเหล็กของหลักสูตรนี้**: คำถามแรกที่ต้องตอบให้ได้ในวันที่สร้างโปรเจกต์ Django ใหม่
> ทุกโปรเจกต์ (แม้แต่ก่อนเขียนโมเดลตัวแรก) คือ **"เราจะใช้ Custom User Model ไหม?"**
> ถ้าไม่แน่ใจ ให้ตอบว่า "ใช่" ไว้ก่อนเสมอ เพราะการมี `AbstractUser` เปล่า ๆ ที่ยังไม่เพิ่ม
> field อะไรเลย ไม่มีต้นทุนเพิ่มขึ้นแม้แต่น้อย แต่เปิดทางเลือกไว้สำหรับอนาคตอย่างสมบูรณ์

### 311.5 สรุปสิ่งที่ Part นี้จะพาไปสร้าง

เพื่อไม่ให้กระทบโปรเจกต์บล็อกหลัก เราจะสร้างโปรเจกต์ทดลองใหม่ชื่อ `usermodel-lab`
สำหรับฝึกฝนแนวคิดทั้งหมดในขั้นตอนที่ 313-316 และ 319 แบบครบวงจร ก่อนกลับมาสรุปด้วย
หลักการที่ใช้ได้กับทุกโปรเจกต์ (313.4, 317, 318) ซึ่งนำไปปรับใช้กับโปรเจกต์บล็อกหลัก
หรือโปรเจกต์ใหม่ในอนาคตของคุณได้ทันที

---

## ขั้นตอนที่ 312: `AbstractUser` vs `AbstractBaseUser` — เมื่อไหร่ควรใช้อะไร

### 312.1 โครงสร้างภายในระบบ Auth ของ Django

Django แบ่งชั้นของระบบผู้ใช้ออกเป็นส่วนประกอบย่อยที่ประกอบกันได้ (composable):

```
AbstractBaseUser          ← มีแค่ password, last_login และ method จัดการรหัสผ่าน
        │
        ├── PermissionsMixin   ← เพิ่ม is_superuser, groups, user_permissions
        │
        └── AbstractUser       ← AbstractBaseUser + PermissionsMixin
                                  + username, first_name, last_name, email,
                                    is_staff, is_active, date_joined (ครบสมบูรณ์)
                │
                └── User (django.contrib.auth.models.User)
                      ← AbstractUser ตัวจริงที่ Django ใช้เป็นค่าเริ่มต้น
```

พูดง่าย ๆ คือ `User` ที่คุณใช้มาตลอดหลักสูตร **ก็คือ `AbstractUser` ที่ Django สร้างเป็น
concrete model ไว้ให้เรียบร้อยแล้ว** ไม่มีอะไรพิเศษไปกว่านั้น

### 312.2 `AbstractUser` คืออะไร

`AbstractUser` คือ **abstract base class ที่มี field ครบทุกตัวเหมือน `User` มาตรฐาน**
(`username`, `first_name`, `last_name`, `email`, `password`, `is_staff`, `is_active`,
`is_superuser`, `last_login`, `date_joined` และ relation ไปยัง `Group`/`Permission`)
สิ่งเดียวที่คุณต้องทำคือ **สืบทอด (subclass) แล้วเพิ่ม field ใหม่เข้าไป** — ไม่ต้องเขียน
field เดิมซ้ำเลยแม้แต่ตัวเดียว

```python
# ตัวอย่างแนวคิด (รายละเอียดเต็มในขั้นตอนที่ 313)
from django.contrib.auth.models import AbstractUser

class CustomUser(AbstractUser):
    bio = models.TextField(blank=True)
    phone_number = models.CharField(max_length=20, blank=True)
    # username, email, password, is_staff ฯลฯ มีให้ครบโดยอัตโนมัติ
```

### 312.3 `AbstractBaseUser` คืออะไร

`AbstractBaseUser` คือ **abstract base class แบบขั้นต่ำที่สุด** มีให้แค่:

- `password` (จัดการ hashing ให้อัตโนมัติผ่าน `set_password()`/`check_password()`)
- `last_login`
- method ช่วยเหลือ เช่น `get_session_auth_hash()`

**ไม่มี** `username`, `email`, `is_staff`, `is_active`, `is_superuser`, `groups`,
`user_permissions` ให้เลยแม้แต่ตัวเดียว — คุณต้องประกาศเองทั้งหมด รวมถึงต้องผสม
`PermissionsMixin` เองถ้าต้องการให้ระบบ Group/Permission ทำงานได้ และต้องเขียน
**Custom `UserManager`** เองเสมอ (ขั้นตอนที่ 315) เพราะ `AbstractBaseUser` ไม่มี
manager มาให้

### 312.4 ตารางเปรียบเทียบ `AbstractUser` กับ `AbstractBaseUser`

| ประเด็น | `AbstractUser` | `AbstractBaseUser` |
|---|---|---|
| Field ที่มีมาให้ | ครบทุกตัวเหมือน `User` มาตรฐาน | มีแค่ `password`, `last_login` |
| ต้องเขียน `UserManager` เอง | ไม่บังคับ (มี `UserManager` เริ่มต้นมาให้แล้ว) | **บังคับเสมอ** |
| ต้องผสม `PermissionsMixin` เอง | ไม่ต้อง (มีมาให้แล้วในตัว) | ต้องผสมเองถ้าต้องการ groups/permissions |
| เปลี่ยน field ที่ใช้ login ได้ไหม | เปลี่ยนได้ (ตั้ง `USERNAME_FIELD` ใหม่) แต่ field เดิม (`username`) ยังอยู่ในตาราง เว้นแต่ตั้ง `username = None` เอง | ออกแบบได้อิสระ 100% ตั้งแต่ต้น ไม่มี field เกินความจำเป็น |
| ความซับซ้อนในการเขียน | ต่ำ — แทบจะแค่เพิ่ม field | สูง — ต้องออกแบบทุก field และ manager เอง |
| เหมาะกับ | ต้องการ "เพิ่ม" ข้อมูลลง User หรือเปลี่ยน field login เล็กน้อย โดยยังอยากได้พฤติกรรมมาตรฐานของ Django | ต้องการควบคุมโครงสร้าง User model แบบเต็มรูปแบบ เช่น ไม่มี `username` เลย ใช้ email/เบอร์โทร/UUID เป็น identity หลัก |
| ตัวอย่างการใช้งานจริง | ระบบทั่วไปที่แค่อยากเพิ่ม `bio`, `phone_number`, `is_verified_seller` | แอปที่ login ด้วยอีเมลล้วน (SaaS สมัยใหม่ส่วนใหญ่), ระบบที่ login ด้วยเบอร์โทร/OTP |

### 312.5 เมื่อไหร่ควรใช้อะไร — คำแนะนำเชิงปฏิบัติ

- **เริ่มจาก `AbstractUser` ก่อนเสมอ** ถ้าไม่มั่นใจ เพราะเขียนง่าย เสี่ยงต่ำ ครอบคลุม
  ความต้องการส่วนใหญ่ของโปรเจกต์ทั่วไป (เพิ่ม field, ปรับ admin, ยังใช้ username ได้ปกติ)
- **ใช้ `AbstractBaseUser`** เมื่อคุณต้องการ **เปลี่ยนกลไกการยืนยันตัวตนหลัก** อย่างจริงจัง
  เช่น ไม่ต้องการ `username` เลย ต้องการ login ด้วยอีเมลอย่างเดียว หรือระบบมีข้อกำหนด
  ด้าน compliance ที่ต้องควบคุมทุก field ของผู้ใช้อย่างละเอียด
- ทั้งสองแบบ **ต้องตั้งค่าไว้ตั้งแต่ก่อน `migrate` ครั้งแรกเหมือนกันทุกประการ** — ความเสี่ยง
  เรื่องการเปลี่ยนกลางทางที่อธิบายในขั้นตอนที่ 311 ใช้กับทั้งสองแบบเท่ากัน

ในขั้นตอนที่ 313 เราจะเริ่มจาก `AbstractUser` ก่อน (ทางเลือกที่ใช้บ่อยที่สุด) แล้วค่อยไปดู
`AbstractBaseUser` แบบเต็มรูปแบบในขั้นตอนที่ 316

---

## ขั้นตอนที่ 313: สร้าง Custom User ด้วย `AbstractUser` เทียบกับแนวทาง Profile จาก Part 012

### 313.1 สร้างโปรเจกต์ทดลอง `usermodel-lab`

เพื่อฝึกฝนอย่างปลอดภัย มาสร้างโปรเจกต์ใหม่แยกต่างหาก (ไม่เกี่ยวกับโปรเจกต์บล็อกหลัก):

```bash
mkdir usermodel-lab
cd usermodel-lab
python3 -m venv venv
source venv/bin/activate        # Windows: venv\Scripts\activate

pip install "django>=5.1,<5.2"
django-admin startproject config .
python manage.py startapp accounts
```

**ขั้นตอนที่สำคัญที่สุดของทั้ง Part นี้**: เพิ่ม `accounts` เข้า `INSTALLED_APPS`
**ก่อน**เขียน model หรือรัน `migrate` ใด ๆ ทั้งสิ้น:

```python
# config/settings.py
INSTALLED_APPS = [
    'django.contrib.admin',
    'django.contrib.auth',
    'django.contrib.contenttypes',
    'django.contrib.sessions',
    'django.contrib.messages',
    'django.contrib.staticfiles',
    'accounts',   # ต้องอยู่ก่อนรัน migrate ครั้งแรกเสมอ
]
```

> **ตรวจสอบให้แน่ใจ**: ยังไม่รัน `python manage.py migrate` จนกว่าจะตั้งค่า
> `AUTH_USER_MODEL` เสร็จในขั้นตอนที่ 314 — ถ้ารัน `migrate` ไปแล้วตอนนี้ ตาราง
> `auth_user` มาตรฐานจะถูกสร้างขึ้นมาก่อน ทำให้ต้องลบฐานข้อมูลทิ้งแล้วเริ่มใหม่

### 313.2 เขียน `CustomUser` ด้วย `AbstractUser`

```python
# accounts/models.py
from django.contrib.auth.models import AbstractUser
from django.core.validators import RegexValidator
from django.db import models


phone_validator = RegexValidator(
    regex=r'^\+?[0-9]{9,15}$',
    message='กรุณากรอกเบอร์โทรศัพท์ในรูปแบบที่ถูกต้อง เช่น +66812345678',
)


class CustomUser(AbstractUser):
    bio = models.TextField(max_length=500, blank=True)
    phone_number = models.CharField(
        max_length=20, blank=True, validators=[phone_validator],
    )
    is_verified_author = models.BooleanField(
        default=False,
        help_text='ผู้เขียนที่ยืนยันตัวตนแล้ว สามารถตีพิมพ์บทความได้ทันทีโดยไม่ต้องรอตรวจสอบ',
    )

    def __str__(self):
        return self.username
```

สังเกตว่าเราไม่ต้องประกาศ `username`, `email`, `password`, `is_staff`, `is_active`,
`first_name`, `last_name` เองเลยแม้แต่ตัวเดียว — ทุกอย่างสืบทอดมาจาก `AbstractUser`
โดยอัตโนมัติ สิ่งที่เราเพิ่มเข้ามามีแค่ 3 field ที่เป็นความต้องการเฉพาะของระบบบล็อก:
`bio`, `phone_number`, และ `is_verified_author`

### 313.3 ทำไมต้องเขียนก่อน `migrate` ครั้งแรกเท่านั้น

เหตุผลเชิงเทคนิคคือ: เมื่อรัน `migrate` ครั้งแรก Django จะสร้างตารางสำหรับทุก model ใน
`INSTALLED_APPS` **รวมถึง `auth.User` มาตรฐานด้วยเสมอ ถ้า `AUTH_USER_MODEL` ยังชี้ไปที่
`auth.User`** — และ Django ไม่มีกลไกใด ๆ ที่จะ "เปลี่ยนใจ" ภายหลังว่าตารางไหนคือตาราง
ผู้ใช้จริงโดยไม่ทำให้ระบบ Permission, ContentType, และ Foreign Key ทั้งหมดสับสน (ดู
รายละเอียดเต็มในขั้นตอนที่ 311.3 และ 317)

### 313.4 เปรียบเทียบกับแนวทาง `Profile` แบบ `OneToOneField` (Part 012)

ใน Part 012 เราสร้าง `Profile` เชื่อมกับ `User` มาตรฐานผ่าน `OneToOneField` เพื่อเพิ่ม
`bio` และ `avatar` โดยไม่แตะต้อง `User` เดิมเลย ทั้งสองแนวทางนี้แก้ปัญหาคล้ายกันแต่มี
ข้อแลกเปลี่ยนต่างกันอย่างชัดเจน:

| ประเด็น | Custom User ด้วย `AbstractUser` (Part นี้) | `Profile` ผ่าน `OneToOneField` (Part 012) |
|---|---|---|
| ต้องตัดสินใจก่อน `migrate` ครั้งแรก | ✅ บังคับ | ❌ ไม่บังคับ ทำได้ทุกเมื่อ |
| Syntax เข้าถึงข้อมูล | `user.bio` (ตรง ๆ) | `user.profile.bio` (ต้องผ่าน relation) |
| Query ต้อง join ตารางเพิ่มไหม | ไม่ต้อง (อยู่ตารางเดียวกับ user) | ต้อง join กับตาราง `profile` เสมอ |
| ใช้กับโปรเจกต์ที่ migrate ไปแล้วได้ไหม | ❌ เสี่ยงสูงมาก (ขั้นตอนที่ 317) | ✅ ทำได้ปลอดภัย 100% |
| ความเข้ากันได้กับ third-party package ที่คาดหวัง `auth.User` | ต้องตรวจสอบว่า package รองรับ custom user model หรือไม่ (ส่วนใหญ่ในปี 2025-2026 รองรับแล้ว) | เข้ากันได้เสมอ เพราะ `User` เดิมไม่เปลี่ยนแปลง |
| เหมาะกับ | โปรเจกต์ใหม่ที่วางแผนไว้ตั้งแต่ต้น หรือต้องเปลี่ยนกลไก login | โปรเจกต์ที่ migrate ไปแล้ว หรือแค่ต้องการ "เพิ่มข้อมูล" ไม่ใช่เปลี่ยนแก่นระบบ auth |
| ใช้ร่วมกันได้ไหม | ✅ ได้ — Custom User Model ยังสามารถมี `Profile` แยกต่างหากอีกชั้นได้ ถ้าข้อมูลเสริมนั้นควรแยกตาราง (เช่น ข้อมูลที่ใหญ่หรือไม่ query บ่อย) | — |

> **ข้อสรุปสำคัญ**: ทั้งสองแนวทางไม่ได้แข่งกันเสมอไป โปรเจกต์ระดับมืออาชีพจำนวนมากใช้
> **ทั้งสองแบบร่วมกัน** — ตั้ง Custom User Model (`AbstractUser`) ไว้ตั้งแต่ต้นเพื่อเก็บ
> field ที่ใช้บ่อยและเกี่ยวข้องกับ authentication โดยตรง (เช่น `phone_number` สำหรับ OTP)
> แล้วยังใช้ `Profile` แยกต่างหากสำหรับข้อมูลที่ใหญ่หรือเปลี่ยนบ่อยกว่า เช่น `avatar`,
> ประวัติส่วนตัวยาว ๆ, การตั้งค่าการแจ้งเตือน ฯลฯ โปรเจกต์บล็อกหลักของเราเลือกใช้เฉพาะ
> `Profile` เพราะ migrate ไปไกลแล้ว ซึ่งเป็นทางเลือกที่ถูกต้องตามกฎในขั้นตอนที่ 311.4

---

## ขั้นตอนที่ 314: ตั้งค่า `AUTH_USER_MODEL` และผลกระทบต่อ Migration

### 314.1 ตั้งค่า `AUTH_USER_MODEL` ใน settings

```python
# config/settings.py
AUTH_USER_MODEL = 'accounts.CustomUser'
```

รูปแบบคือ `'<app_label>.<ModelName>'` — สังเกตว่า Django **ไม่ได้ใช้ path ของ Python
module** (เช่น `accounts.models.CustomUser`) แต่ใช้ **app label** ที่มาจากชื่อแอปใน
`INSTALLED_APPS` ต่อด้วยชื่อ class ของ model เท่านั้น

### 314.2 รัน Migration ครั้งแรก

```bash
python manage.py makemigrations accounts
python manage.py migrate
```

ผลลัพธ์ที่ควรเห็น:

```
Migrations for 'accounts':
  accounts/migrations/0001_initial.py
    - Create model CustomUser

Operations to perform:
  Apply all migrations: accounts, admin, auth, contenttypes, sessions
Running migrations:
  Applying accounts.0001_initial... OK
  Applying admin.0001_initial... OK
  Applying auth.0001_initial... OK
  ...
```

สังเกตว่า `accounts.0001_initial` ถูก apply **ก่อน** `admin` และ `auth` เสมอ นี่ไม่ใช่
เรื่องบังเอิญ — มันคือผลของกลไกที่ชื่อว่า `swappable_dependency`

### 314.3 `swappable_dependency` คืออะไร

เปิดไฟล์ migration ของแอปที่มี `ForeignKey`/`OneToOneField` ไปยังผู้ใช้ (เช่นถ้าเรามีแอป
`blog` ในโปรเจกต์นี้ที่มี `Post.author`) จะเห็นโครงสร้างประมาณนี้:

```python
# blog/migrations/0001_initial.py
from django.conf import settings
from django.db import migrations, models
import django.db.models.deletion


class Migration(migrations.Migration):

    initial = True

    dependencies = [
        migrations.swappable_dependency(settings.AUTH_USER_MODEL),
    ]

    operations = [
        migrations.CreateModel(
            name='Post',
            fields=[
                ('id', models.BigAutoField(auto_created=True, primary_key=True, serialize=False)),
                ('title', models.CharField(max_length=200)),
                ('author', models.ForeignKey(
                    on_delete=django.db.models.deletion.CASCADE,
                    to=settings.AUTH_USER_MODEL,
                )),
            ],
        ),
    ]
```

`migrations.swappable_dependency(settings.AUTH_USER_MODEL)` บอก Django migration
framework ว่า **"migration นี้ต้องรอให้ migration ของโมเดลผู้ใช้ปัจจุบัน (ไม่ว่าจะเป็น
`auth.User` หรือ Custom User Model ใดก็ตาม) ถูก apply ก่อนเสมอ"** — คำว่า "swappable"
หมายถึง Django ไม่ได้ hardcode ว่าต้องรอ `auth.User` โดยเฉพาะ แต่รอ**โมเดลที่ถูกตั้งค่า
ไว้ใน `AUTH_USER_MODEL` ณ ตอนนั้น** ซึ่งทำให้ทั้งระบบ migration ทำงานถูกต้องไม่ว่าคุณจะ
ใช้ `User` มาตรฐานหรือ Custom User Model ก็ตาม โดยไม่ต้องแก้ migration เก่าเลย

### 314.4 ทำไมการตั้งค่าทีหลังถึง "พัง" ทันที

ลองจินตนาการว่าคุณรัน `migrate` ไปแล้วด้วย `AUTH_USER_MODEL` เริ่มต้น (`auth.User`)
แล้วภายหลังมาแก้เป็น `accounts.CustomUser` — เมื่อรัน `migrate` อีกครั้ง Django จะเจอ
ปัญหาทันทีเพราะ:

```
django.db.utils.OperationalError: no such table: accounts_customuser
```

หรือถ้าตารางถูกสร้างไปแล้วบางส่วน อาจเจอ:

```
django.core.exceptions.ImproperlyConfigured: AUTH_USER_MODEL refers to model
'accounts.CustomUser' that has not been installed
```

หรือในกรณีที่ร้ายแรงกว่า (มี FK ที่ apply ไปแล้วชี้ไปยัง `auth_user`):

```
django.db.utils.IntegrityError: FOREIGN KEY constraint failed
```

เพราะ migration state ที่บันทึกไว้ในตาราง `django_migrations` ยังจำได้ว่า
`ForeignKey` เก่าชี้ไปที่ตารางเดิม ในขณะที่ `AUTH_USER_MODEL` ใหม่ชี้ไปที่ตารางที่ไม่มี
ข้อมูลเดิมอยู่เลย — นี่คือสาเหตุที่ Django Documentation เตือนอย่างหนักแน่นว่าการ
เปลี่ยนแปลงนี้ **ต้องทำก่อน `migrate` ครั้งแรกเท่านั้น** ในสถานการณ์ปกติ

---

## ขั้นตอนที่ 315: เขียน Custom `UserManager` ของตัวเอง

### 315.1 ทำไมต้องเขียน Manager เอง

`AbstractUser` มี `UserManager` มาตรฐานมาให้อยู่แล้ว (สืบทอดจาก Django) ซึ่งเพียงพอถ้าคุณ
ยังใช้ `username` เป็นหลักในการ login แต่มี 2 สถานการณ์ที่ **ต้อง** เขียน Manager เอง:

1. คุณใช้ `AbstractBaseUser` (ไม่มี Manager มาให้เลย — บังคับต้องเขียนเอง)
2. คุณต้องการปรับพฤติกรรมการสร้างผู้ใช้ เช่น normalize อีเมลให้เป็นตัวพิมพ์เล็กเสมอ,
   บังคับให้ต้องมีอีเมลตอนสร้าง superuser, หรือเพิ่ม validation พิเศษตอนสร้างผู้ใช้

### 315.2 เขียน `CustomUserManager`

```python
# accounts/managers.py
from django.contrib.auth.base_user import BaseUserManager


class CustomUserManager(BaseUserManager):
    """Manager สำหรับ CustomUser ที่ normalize อีเมลและบังคับกฎการสร้าง superuser"""

    use_in_migrations = True

    def _create_user(self, username, email, password, **extra_fields):
        if not email:
            raise ValueError('ผู้ใช้ต้องมีอีเมลเสมอ')
        email = self.normalize_email(email)
        user = self.model(username=username, email=email, **extra_fields)
        user.set_password(password)
        user.save(using=self._db)
        return user

    def create_user(self, username, email=None, password=None, **extra_fields):
        extra_fields.setdefault('is_staff', False)
        extra_fields.setdefault('is_superuser', False)
        return self._create_user(username, email, password, **extra_fields)

    def create_superuser(self, username, email=None, password=None, **extra_fields):
        extra_fields.setdefault('is_staff', True)
        extra_fields.setdefault('is_superuser', True)

        if extra_fields.get('is_staff') is not True:
            raise ValueError('Superuser ต้องมี is_staff=True')
        if extra_fields.get('is_superuser') is not True:
            raise ValueError('Superuser ต้องมี is_superuser=True')

        return self._create_user(username, email, password, **extra_fields)
```

### 315.3 อธิบายส่วนสำคัญของ Manager

- **`use_in_migrations = True`**: บอก Django ว่า Manager นี้ปลอดภัยที่จะถูก serialize
  เก็บไว้ใน migration files ได้ (จำเป็นเมื่อใช้ `RunPython` ที่ต้องเรียก
  `apps.get_model(...).objects.create_user(...)` ภายใน data migration)
- **`normalize_email()`**: method ที่มีมาให้จาก `BaseUserManager` ทำให้ domain ส่วน
  ของอีเมล (หลัง `@`) กลายเป็นตัวพิมพ์เล็กเสมอ (เช่น `USER@Example.COM` →
  `USER@example.com`) เพื่อป้องกันการสร้างบัญชีซ้ำซ้อนจากตัวพิมพ์เล็ก-ใหญ่ที่ต่างกัน
- **`set_password()`**: hash รหัสผ่านให้อัตโนมัติด้วยอัลกอริทึมที่ตั้งค่าไว้ใน
  `PASSWORD_HASHERS` (ค่าเริ่มต้นคือ PBKDF2) — **ห้ามเก็บรหัสผ่านตรง ๆ ลงฟิลด์เด็ดขาด**
- **`_create_user()`**: method ภายในที่ทั้ง `create_user()` และ `create_superuser()`
  เรียกใช้ร่วมกัน (DRY) ส่วน `create_user()`/`create_superuser()` แค่กำหนดค่า default
  ของ `is_staff`/`is_superuser` ให้ต่างกัน

### 315.4 ผูก Manager เข้ากับ `CustomUser`

```python
# accounts/models.py
from django.contrib.auth.models import AbstractUser
from django.db import models

from .managers import CustomUserManager


class CustomUser(AbstractUser):
    bio = models.TextField(max_length=500, blank=True)
    phone_number = models.CharField(max_length=20, blank=True)
    is_verified_author = models.BooleanField(default=False)

    objects = CustomUserManager()

    def __str__(self):
        return self.username
```

ทดสอบใน shell:

```bash
python manage.py shell
```

```python
>>> from accounts.models import CustomUser
>>> user = CustomUser.objects.create_user(
...     username='sireeporn', email='SIREEPORN@Example.COM', password='securepass123',
... )
>>> user.email
'SIREEPORN@example.com'   # domain ถูก normalize เป็นตัวพิมพ์เล็กแล้ว
>>> user.check_password('securepass123')
True
>>> superuser = CustomUser.objects.create_superuser(
...     username='admin', email='admin@example.com', password='adminpass123',
... )
>>> superuser.is_staff, superuser.is_superuser
(True, True)
```

---

## ขั้นตอนที่ 316: Custom User แบบเต็มรูปแบบด้วย `AbstractBaseUser` — Login ด้วยอีเมล

### 316.1 ทำไมหลายระบบเลือก Login ด้วยอีเมลแทน Username

ระบบ SaaS และแอปพลิเคชันสมัยใหม่จำนวนมาก (ปี 2025-2026) เลือกให้ผู้ใช้ login ด้วย
**อีเมล** แทน username เพราะ:

- ผู้ใช้จำอีเมลตัวเองได้เสมอ แต่มักจำ username ที่ตั้งไว้ไม่ได้
- ลดขั้นตอนตอนสมัครสมาชิก (ไม่ต้องคิด username ที่ไม่ซ้ำใคร)
- อีเมลใช้ยืนยันตัวตน (verification) และรีเซ็ตรหัสผ่านอยู่แล้ว การมี username แยกจึง
  เป็น field ที่ไม่จำเป็น

สถานการณ์นี้คือตัวอย่างที่ชัดเจนที่สุดว่าทำไมต้องใช้ `AbstractBaseUser` แทน
`AbstractUser` — เพราะเราต้องการ **ลบ `username` ออกไปเลย** ไม่ใช่แค่เพิ่ม field

### 316.2 เขียน `CustomUser` ด้วย `AbstractBaseUser` แบบเต็มรูปแบบ

```python
# accounts/models.py
from django.contrib.auth.base_user import AbstractBaseUser
from django.contrib.auth.models import PermissionsMixin
from django.db import models
from django.utils import timezone

from .managers import CustomUserManager


class CustomUser(AbstractBaseUser, PermissionsMixin):
    email = models.EmailField(unique=True, verbose_name='อีเมล')
    first_name = models.CharField(max_length=150, blank=True)
    last_name = models.CharField(max_length=150, blank=True)
    is_active = models.BooleanField(default=True)
    is_staff = models.BooleanField(default=False)
    date_joined = models.DateTimeField(default=timezone.now)

    objects = CustomUserManager()

    USERNAME_FIELD = 'email'
    EMAIL_FIELD = 'email'
    REQUIRED_FIELDS = []   # 'email' และ 'password' ถูกถามเสมออยู่แล้ว ไม่ต้องระบุซ้ำ

    class Meta:
        verbose_name = 'ผู้ใช้'
        verbose_name_plural = 'ผู้ใช้'

    def __str__(self):
        return self.email

    def get_full_name(self):
        full_name = f'{self.first_name} {self.last_name}'.strip()
        return full_name or self.email

    def get_short_name(self):
        return self.first_name or self.email
```

### 316.3 อธิบายทีละส่วน

- **`AbstractBaseUser`**: ให้ `password`, `last_login`, และ method จัดการรหัสผ่าน
- **`PermissionsMixin`**: ให้ `is_superuser`, `groups`, `user_permissions`, และ method
  ตรวจสอบสิทธิ์อย่าง `has_perm()`, `has_module_perms()` — ถ้าไม่ผสม mixin นี้ ระบบ
  Permission และ Group ของ Django (Part 033) จะใช้กับ User model นี้ไม่ได้เลย
- **ไม่มี `username` เลย**: เพราะเราตัดสินใจใช้อีเมลเป็น identity หลักเพียงอย่างเดียว
- **`USERNAME_FIELD = 'email'`**: บอก Django ว่า field ไหนคือ "field ที่ใช้ login"
  ระบบ `authenticate()`, `AuthenticationForm`, Django Admin login จะใช้ field นี้แทน
  `username` โดยอัตโนมัติทันที
- **`EMAIL_FIELD = 'email'`**: บอก Django ว่า field ไหนคืออีเมล (ใช้โดยฟีเจอร์อย่าง
  password reset ที่ต้องรู้ว่าจะส่งอีเมลไปที่ field ไหน) ปกติจะเป็น field เดียวกับ
  `USERNAME_FIELD` เมื่อ login ด้วยอีเมลอยู่แล้ว
- **`REQUIRED_FIELDS = []`**: รายการ field ที่ `createsuperuser` จะถามเพิ่มเติม **นอกเหนือ
  จาก** `USERNAME_FIELD` และ `password` (ซึ่งถูกถามเสมออยู่แล้วโดยไม่ต้องระบุ) เนื่องจาก
  เราไม่มี field บังคับอื่นที่ต้องถามตอนสร้าง superuser จึงปล่อยเป็น list ว่าง
- **`get_full_name()`/`get_short_name()`**: Django Admin และ template บางส่วนคาดหวัง
  ว่า User model จะมี method เหล่านี้ (แม้ไม่ได้บังคับอย่างเป็นทางการใน `AbstractBaseUser`
  แต่เป็น convention ที่ third-party package จำนวนมากคาดหวังไว้) การใส่ไว้ช่วยเพิ่ม
  ความเข้ากันได้

### 316.4 ทดสอบสร้าง Superuser ด้วยอีเมล

```bash
python manage.py makemigrations accounts
python manage.py migrate
python manage.py createsuperuser
```

```
อีเมล: admin@example.com
Password: ********
Password (again): ********
Superuser created successfully.
```

สังเกตว่าคำถามแรกคือ **"อีเมล"** ไม่ใช่ "Username" อีกต่อไป เพราะ Django อ่านค่าจาก
`USERNAME_FIELD` โดยอัตโนมัติเวลาสร้างคำสั่ง `createsuperuser` ให้เข้ากับโครงสร้าง
User model ของคุณเสมอ ไม่ต้องเขียนโค้ดเพิ่มใด ๆ

### 316.5 ผลกระทบต่อระบบ Login ที่มีอยู่

ระบบ `django.contrib.auth.views.LoginView` และ `AuthenticationForm` มาตรฐานจะทำงาน
ได้ทันทีโดยไม่ต้องแก้โค้ด เพราะทั้งคู่ query จาก `USERNAME_FIELD` แบบไดนามิกอยู่แล้ว
สิ่งเดียวที่ต้องปรับคือ **label ในฟอร์ม** (ค่าเริ่มต้นอาจยังเขียนว่า "Username" ในบาง
เทมเพลตเก่า) ซึ่งแก้ได้ง่าย ๆ ด้วยการ override field label ใน custom login form —
รายละเอียดเต็มเรื่อง login form และ template อยู่ใน Part 031 ที่ Part นี้ต่อยอดมา

---

## ขั้นตอนที่ 317: การย้ายโปรเจกต์ที่มีอยู่แล้วไปใช้ Custom User Model

### 317.1 คำเตือนตรง ๆ ก่อนอ่านต่อ

หัวข้อนี้อธิบายเทคนิคสำหรับสถานการณ์ที่ **ควรหลีกเลี่ยงถ้าเป็นไปได้** หากคุณกำลังเริ่ม
โปรเจกต์ใหม่ ให้กลับไปอ่านขั้นตอนที่ 311 และตัดสินใจเรื่อง Custom User Model ตั้งแต่ตอนนี้
เนื้อหาต่อไปนี้มีไว้สำหรับกรณีที่คุณ**สืบทอดโปรเจกต์เก่าที่มีข้อมูลจริงในระบบ**แล้ว
พบว่าจำเป็นต้องเปลี่ยนจริง ๆ — ซึ่งควรเป็น**ทางเลือกสุดท้าย**เท่านั้น

### 317.2 สถานการณ์ที่ยังพอทำได้อย่างปลอดภัย: ไม่มีข้อมูลจริง

ถ้าโปรเจกต์ยังอยู่ในขั้น dev/staging และยังไม่มีผู้ใช้จริงในระบบ (หรือยอมสูญเสียข้อมูล
ทดสอบได้) วิธีที่ง่ายและปลอดภัยที่สุดคือ **เริ่มใหม่ทั้งหมด**:

```bash
# 1. ลบไฟล์ migration เดิมทั้งหมด (เก็บ __init__.py ไว้)
find . -path "*/migrations/*.py" -not -name "__init__.py" -delete

# 2. ลบฐานข้อมูล (ตัวอย่าง SQLite)
rm db.sqlite3

# 3. เพิ่ม Custom User Model และตั้งค่า AUTH_USER_MODEL (ขั้นตอนที่ 313-314)

# 4. สร้าง migration ใหม่ทั้งหมดตั้งแต่ต้น
python manage.py makemigrations
python manage.py migrate
```

นี่คือเหตุผลที่ทีมพัฒนามืออาชีพจำนวนมากตัดสินใจเรื่อง Custom User Model ใน **สัปดาห์
แรก** ของโปรเจกต์เสมอ — เพราะต้นทุนของการ "เริ่มใหม่" ในช่วงนั้นแทบเป็นศูนย์

### 317.3 สถานการณ์ที่ยากมาก: มีข้อมูลผู้ใช้จริงในระบบแล้ว

ถ้าจำเป็นต้องเปลี่ยนจริง ๆ ในระบบที่มีผู้ใช้จริงอยู่แล้ว ต้องวางแผนอย่างละเอียดและทดสอบ
ใน staging environment ที่เหมือน production ทุกประการก่อนเสมอ ขั้นตอนระดับสูง
(high-level) มีดังนี้:

**ขั้นที่ 1: สำรองข้อมูลเต็มรูปแบบ**

```bash
# PostgreSQL ตัวอย่าง
pg_dump -Fc mydatabase > backup_before_user_migration.dump
```

**ขั้นที่ 2: สร้าง Custom User Model ที่มี field ครบเหมือน `auth.User` เดิมทุกประการ
(ยังไม่เพิ่ม field ใหม่ในขั้นนี้) และใช้ trick `db_table` เพื่อให้ Django ใช้**ตารางเดิม**
โดยไม่ต้องคัดลอกข้อมูลข้ามตาราง**:

```python
# accounts/models.py
from django.contrib.auth.models import AbstractUser


class CustomUser(AbstractUser):
    class Meta:
        db_table = 'auth_user'   # ใช้ตารางเดิม ไม่สร้างตารางใหม่ ไม่ต้องคัดลอกข้อมูล
```

**ขั้นที่ 3: ตั้งค่า `AUTH_USER_MODEL` และสร้าง migration แบบพิเศษที่บอก Django ว่า
"ตารางนี้มีอยู่แล้ว ไม่ต้องสร้างใหม่"** โดยใช้ `migrations.CreateModel` ร่วมกับ
`SeparateDatabaseAndState` เพื่อแยก "สถานะที่ Django รับรู้" ออกจาก "คำสั่ง SQL จริงที่รัน"

```python
# accounts/migrations/0001_initial.py (ตัวอย่างแนวคิด — ต้องปรับ fields ให้ตรงกับ auth.User จริง)
from django.db import migrations, models
import django.contrib.auth.models
import django.contrib.auth.validators


class Migration(migrations.Migration):

    initial = True
    dependencies = [
        ('auth', '0012_alter_user_first_name_max_length'),
    ]

    operations = [
        migrations.SeparateDatabaseAndState(
            # state_operations: สิ่งที่ Django "คิดว่า" เกิดขึ้น (สร้าง state ใหม่)
            state_operations=[
                migrations.CreateModel(
                    name='CustomUser',
                    fields=[
                        # ต้องคัดลอก field ทั้งหมดจาก django.contrib.auth.models.AbstractUser
                        # ให้ตรงกันทุกประการ (ดูซอร์สจริงของ Django เวอร์ชันที่ใช้)
                    ],
                    options={'db_table': 'auth_user'},
                    bases=(django.contrib.auth.models.AbstractUser,),
                ),
            ],
            # database_operations: คำสั่ง SQL จริงที่รัน — ว่างเปล่า เพราะตาราง auth_user มีอยู่แล้ว
            database_operations=[],
        ),
    ]
```

**ขั้นที่ 4: อัปเดต `django_content_type` ให้ permission และ admin log เดิมยังใช้งานได้**
ด้วย data migration:

```python
# accounts/migrations/0002_update_content_types.py
from django.db import migrations


def update_content_types(apps, schema_editor):
    ContentType = apps.get_model('contenttypes', 'ContentType')
    ContentType.objects.filter(app_label='auth', model='user').update(
        app_label='accounts', model='customuser',
    )


def reverse_update_content_types(apps, schema_editor):
    ContentType = apps.get_model('contenttypes', 'ContentType')
    ContentType.objects.filter(app_label='accounts', model='customuser').update(
        app_label='auth', model='user',
    )


class Migration(migrations.Migration):
    dependencies = [('accounts', '0001_initial')]
    operations = [
        migrations.RunPython(update_content_types, reverse_update_content_types),
    ]
```

**ขั้นที่ 5: ทดสอบใน staging environment อย่างละเอียดที่สุด** ครอบคลุม: login/logout,
password reset, Django Admin ทุกหน้าที่เกี่ยวกับ user, permission ทุกจุด, ทุก
`ForeignKey` ที่ชี้ไปยังผู้ใช้ในทุกแอป, sessions ของผู้ใช้ที่ login ค้างอยู่

**ขั้นที่ 6: วางแผน deploy พร้อม maintenance window** — ปิดระบบชั่วคราว รัน migration
บน production, ตรวจสอบผลลัพธ์ทันที, มีแผน rollback ที่ทดสอบแล้วจริง ๆ (ไม่ใช่แค่เขียนไว้
เฉย ๆ) ก่อนเปิดระบบกลับมา

### 317.4 เพิ่ม Field ใหม่หลังย้ายสำเร็จ

หลังจากขั้นตอนที่ 317.3 สำเร็จและ `CustomUser` ใช้ตาราง `auth_user` เดิมเรียบร้อยแล้ว
การเพิ่ม field ใหม่ (เช่น `bio`, `phone_number`) ในภายหลังจะเป็นการทำ migration ปกติ
ธรรมดา (`ALTER TABLE ADD COLUMN`) ไม่ต่างจาก field อื่น ๆ ในโปรเจกต์เลย เพราะจุดที่ยาก
ที่สุด (การเปลี่ยนว่า "model ไหนคือ user model") ผ่านไปแล้ว

### 317.5 คำแนะนำสุดท้าย

> เทคนิคในขั้นตอนที่ 317.3 เป็นเทคนิคระดับสูงที่ทีมงานหลายทีมในอุตสาหกรรมเคยใช้จริง แต่
> ต้องการความระมัดระวังสูงมาก ควรมีวิศวกรที่มีประสบการณ์ตรวจทาน (code review) และทดสอบใน
> staging ที่จำลอง production จริงก่อนเสมอ **ทางเลือกที่ปลอดภัยกว่าเสมอคือการตัดสินใจ
> เรื่องนี้ตั้งแต่ Part 004 ของหลักสูตรนี้** (ตอนสร้างโปรเจกต์ครั้งแรก) หากคุณย้อนเวลาได้
> นี่คือจุดที่ควรตัดสินใจ ไม่ใช่หลังจากระบบมีผู้ใช้จริงหลักพันหรือหลักหมื่นคนแล้ว

---

## ขั้นตอนที่ 318: `get_user_model()` เทียบกับการ Import `User` ตรง ๆ

### 318.1 ปัญหาของการ Import `User` ตรง ๆ

โค้ดแบบนี้ดูเหมือนไม่มีปัญหาอะไร:

```python
# ❌ ไม่แนะนำในโค้ดที่ต้อง reusable
from django.contrib.auth.models import User

def get_active_authors():
    return User.objects.filter(is_active=True)
```

แต่ถ้าโปรเจกต์เปลี่ยนไปใช้ `CustomUser` (`accounts.CustomUser`) โค้ดนี้จะยังคง import
`auth.User` ตัวเดิมอยู่ ซึ่งเป็น **model คนละตัว** กับที่ตั้งค่าไว้ใน `AUTH_USER_MODEL`
จริง ทำให้ query ผิดตาราง หรือแย่กว่านั้นคือทำให้เกิด error ทันทีเมื่อพยายามสร้าง
relation ข้ามกัน:

```
ValueError: Cannot alias imported name 'User' ... / RuntimeError: Conflicting
'user' models in application 'auth': <class 'django.contrib.auth.models.User'> and ...
```

### 318.2 `get_user_model()` คือทางแก้ที่ถูกต้อง

```python
# ✅ ถูกต้อง
from django.contrib.auth import get_user_model

def get_active_authors():
    User = get_user_model()   # อ่านจาก AUTH_USER_MODEL แบบไดนามิกเสมอ ณ เวลาที่เรียก
    return User.objects.filter(is_active=True)
```

`get_user_model()` คือฟังก์ชันจาก `django.contrib.auth` ที่คืน **model class ที่ตั้งค่า
ไว้จริงใน `AUTH_USER_MODEL`** ไม่ว่าจะเป็น `auth.User` มาตรฐานหรือ Custom User Model
ใดก็ตาม โค้ดที่เขียนด้วย `get_user_model()` จะทำงานถูกต้องเสมอไม่ว่าโปรเจกต์จะสลับไปใช้
User model แบบไหนในอนาคต

### 318.3 ทำไมต้องเรียก "ในฟังก์ชัน" ไม่ใช่ "ระดับบนสุดของไฟล์"

```python
# ❌ อันตราย — อาจพังตอน Django เริ่มโหลด apps
from django.contrib.auth import get_user_model
User = get_user_model()   # เรียกตอน import module — เสี่ยง AppRegistryNotReady


# ✅ ปลอดภัย — เรียกเมื่อฟังก์ชันถูกใช้งานจริงเท่านั้น
from django.contrib.auth import get_user_model

def some_view(request):
    User = get_user_model()
    ...
```

`get_user_model()` ต้องการให้ Django app registry โหลดเสร็จสมบูรณ์ก่อน (ทุก app ใน
`INSTALLED_APPS` พร้อมใช้งาน) ถ้าเรียกที่ระดับบนสุดของไฟล์ที่ถูก import ตั้งแต่ตอน Django
เริ่มต้น (เช่นใน `models.py` ของแอปอื่นที่ import ไฟล์นี้เร็วเกินไป) อาจเจอ:

```
django.core.exceptions.AppRegistryNotReady: Apps aren't loaded yet.
```

### 318.4 แล้วใน `models.py` ล่ะ ควรใช้อะไร

ข้อควรระวังสำคัญ: **`models.py` ไม่ควรใช้ `get_user_model()` สำหรับ `ForeignKey`/
`OneToOneField`** เพราะตอนที่ Python ประมวลผล class definition ของ model ระดับบนสุด
ของไฟล์ ระบบ apps อาจยังไม่พร้อม 100% เช่นกัน — วิธีที่ถูกต้องสำหรับ `models.py` คือใช้
**`settings.AUTH_USER_MODEL` เป็น string** ตามที่เรียนไปแล้วใน Part 012:

```python
# accounts/models.py — ถูกต้อง
from django.conf import settings
from django.db import models

class Profile(models.Model):
    user = models.OneToOneField(
        settings.AUTH_USER_MODEL,   # string เสมอ ไม่ใช่ model class
        on_delete=models.CASCADE,
    )
```

### 318.5 ตารางสรุป: ใช้อะไรที่ไหน

| บริบท | ใช้ | เหตุผล |
|---|---|---|
| `models.py` — `ForeignKey`/`OneToOneField` ไปยังผู้ใช้ | `settings.AUTH_USER_MODEL` (string) | ปลอดภัยที่ class-definition time ก่อน app registry พร้อม |
| `views.py`, `forms.py`, `signals.py` (ในฟังก์ชัน) | `get_user_model()` | ได้ model class จริงที่ query/สร้าง instance ได้ทันที |
| Management commands | `get_user_model()` | เรียกภายใน `handle()` เสมอ ไม่ใช่ระดับบนของไฟล์ |
| แอปที่แจกจ่ายเป็น pip package (reusable apps) | ทั้งสองแบบข้างต้นเท่านั้น | **ห้าม** import `django.contrib.auth.models.User` โดยตรงเด็ดขาด เพราะ package ต้องทำงานได้กับ Custom User Model ของทุกโปรเจกต์ที่นำไปใช้ |
| เขียนทดสอบ (tests.py) | `get_user_model()` | เพื่อให้ test รันได้ถูกต้องไม่ว่าโปรเจกต์จะใช้ User model แบบไหน |

### 318.6 ตัวอย่างการใช้งานครบวงจร

```python
# accounts/views.py
from django.contrib.auth import get_user_model
from django.shortcuts import render


def author_list(request):
    User = get_user_model()
    authors = User.objects.filter(is_verified_author=True)
    return render(request, 'accounts/author_list.html', {'authors': authors})
```

```python
# accounts/management/commands/promote_author.py
from django.contrib.auth import get_user_model
from django.core.management.base import BaseCommand, CommandError


class Command(BaseCommand):
    help = 'ยกระดับผู้ใช้ให้เป็นผู้เขียนที่ยืนยันตัวตนแล้ว'

    def add_arguments(self, parser):
        parser.add_argument('username', type=str)

    def handle(self, *args, **options):
        User = get_user_model()   # เรียกภายใน handle() เท่านั้น ไม่ใช่ระดับบนของไฟล์
        try:
            user = User.objects.get(username=options['username'])
        except User.DoesNotExist:
            raise CommandError(f'ไม่พบผู้ใช้ "{options["username"]}"')

        user.is_verified_author = True
        user.save(update_fields=['is_verified_author'])
        self.stdout.write(self.style.SUCCESS(f'ยกระดับ {user} เป็นผู้เขียนแล้ว'))
```

> **กฎเหล็กของหลักสูตรนี้**: ห้าม `from django.contrib.auth.models import User` แล้ว
> ใช้ `User` ตรง ๆ ในโค้ดแอปที่คุณเขียนเองเด็ดขาด (เว้นแต่กำลังเขียน migration ของ
> `django.contrib.auth` เองซึ่งไม่เกี่ยวกับเรา) ให้ใช้ `settings.AUTH_USER_MODEL` ใน
> `models.py` และ `get_user_model()` ในทุกที่อื่นเสมอ

---

## ขั้นตอนที่ 319: ปรับแต่ง `UserAdmin` ให้รองรับ Custom User Model

### 319.1 ปัญหาเมื่อเปิด Django Admin ครั้งแรกหลังเปลี่ยน User Model

ถ้าคุณเปิด `/admin/` หลังตั้งค่า `AUTH_USER_MODEL` เป็น `accounts.CustomUser` แล้ว
**ไม่ปรับแต่ง admin เพิ่มเติมเลย** คุณจะพบว่า **หน้า User ในแอป "Authentication and
Authorization" หายไปจาก Admin** เพราะ `django.contrib.auth` ลงทะเบียน `UserAdmin`
ไว้กับ `auth.User` เท่านั้น ไม่ได้ลงทะเบียนให้ Custom User Model โดยอัตโนมัติ

### 319.2 กรณี `AbstractUser` (มี field เดิมครบ แค่เพิ่มใหม่): Extend `UserAdmin`

```python
# accounts/admin.py
from django.contrib import admin
from django.contrib.auth.admin import UserAdmin

from .models import CustomUser


@admin.register(CustomUser)
class CustomUserAdmin(UserAdmin):
    # เพิ่ม field ใหม่เข้าไปในหน้าแก้ไข โดยยังคงกลุ่ม fieldset เดิมของ UserAdmin ไว้ทั้งหมด
    fieldsets = UserAdmin.fieldsets + (
        ('ข้อมูลเพิ่มเติม', {
            'fields': ('bio', 'phone_number', 'is_verified_author'),
        }),
    )

    # เพิ่ม field ใหม่ในหน้า "เพิ่มผู้ใช้ใหม่" ด้วยเช่นกัน (ถ้าต้องการ)
    add_fieldsets = UserAdmin.add_fieldsets + (
        ('ข้อมูลเพิ่มเติม', {
            'fields': ('bio', 'phone_number'),
        }),
    )

    list_display = UserAdmin.list_display + ('is_verified_author',)
    list_filter = UserAdmin.list_filter + ('is_verified_author',)
```

เพราะ `AbstractUser` มี field เดิมครบทุกตัวเหมือน `User` มาตรฐาน การ extend
`UserAdmin` เดิมแล้ว "ต่อท้าย" ด้วย tuple ใหม่จึงทำได้ง่ายมาก — ไม่ต้องเขียน
`fieldsets` ใหม่ทั้งหมด

### 319.3 กรณี `AbstractBaseUser` (โครงสร้างต่างไปจากเดิมทั้งหมด): เขียน Admin ใหม่ทั้งหมด

เนื่องจากไม่มี `username` และมี field ต่างไปจากเดิมโดยสิ้นเชิง เราต้องเขียน
`UserCreationForm`, `UserChangeForm`, และ `UserAdmin` ใหม่ทั้งหมด:

```python
# accounts/forms.py
from django.contrib.auth.forms import UserChangeForm, UserCreationForm

from .models import CustomUser


class CustomUserCreationForm(UserCreationForm):
    class Meta(UserCreationForm.Meta):
        model = CustomUser
        fields = ('email',)   # ไม่มี username แล้ว ใช้ email เป็นตัวระบุตัวตนแทน


class CustomUserChangeForm(UserChangeForm):
    class Meta(UserChangeForm.Meta):
        model = CustomUser
        fields = ('email', 'first_name', 'last_name')
```

```python
# accounts/admin.py
from django.contrib import admin
from django.contrib.auth.admin import UserAdmin

from .forms import CustomUserChangeForm, CustomUserCreationForm
from .models import CustomUser


@admin.register(CustomUser)
class CustomUserAdmin(UserAdmin):
    add_form = CustomUserCreationForm
    form = CustomUserChangeForm
    model = CustomUser

    list_display = ('email', 'first_name', 'last_name', 'is_staff', 'is_active')
    list_filter = ('is_staff', 'is_active', 'is_superuser')
    ordering = ('email',)
    search_fields = ('email', 'first_name', 'last_name')

    fieldsets = (
        (None, {'fields': ('email', 'password')}),
        ('ข้อมูลส่วนตัว', {'fields': ('first_name', 'last_name')}),
        ('สิทธิ์การใช้งาน', {
            'fields': ('is_active', 'is_staff', 'is_superuser', 'groups', 'user_permissions'),
        }),
        ('วันที่สำคัญ', {'fields': ('last_login', 'date_joined')}),
    )

    add_fieldsets = (
        (None, {
            'classes': ('wide',),
            'fields': ('email', 'password1', 'password2', 'is_staff', 'is_active'),
        }),
    )
```

### 319.4 อธิบายส่วนสำคัญ

- **`add_form`/`form`**: บอก Django Admin ว่าจะใช้ฟอร์มที่เราเขียนเองสำหรับหน้า
  "เพิ่มผู้ใช้ใหม่" และ "แก้ไขผู้ใช้" ตามลำดับ แทนฟอร์มเริ่มต้นที่คาดหวัง `username`
- **`fieldsets`/`add_fieldsets`**: ต้องเขียนใหม่ทั้งหมดเพราะโครงสร้าง field ต่างจาก
  `User` มาตรฐานโดยสิ้นเชิง (ไม่มี `username` อีกต่อไป)
- **`ordering = ('email',)`**: เนื่องจากไม่มี `username` ให้เรียงลำดับ ต้องเปลี่ยนเป็น
  `email` แทน มิฉะนั้น Admin จะพยายามเรียงตาม `username` ที่ไม่มีอยู่จริงแล้ว raise error
- **`groups`, `user_permissions`**: ยังใช้งานได้ปกติเพราะเรา mixin `PermissionsMixin`
  ไว้ใน `CustomUser` (ขั้นตอนที่ 316) — จะเจาะลึกเรื่อง Permission และ Group เต็มรูปแบบ
  ใน Part 033

### 319.5 ตรวจสอบผลลัพธ์

```bash
python manage.py runserver
```

เปิด `http://127.0.0.1:8000/admin/` แล้ว login ด้วยบัญชี superuser (จากขั้นตอนที่ 316.4)
คุณควรเห็นเมนู **"ผู้ใช้"** ในแอป "Authentication and Authorization" พร้อมฟอร์ม
เพิ่ม/แก้ไขที่ถามอีเมลแทน username ทุกจุด และ field พิเศษที่เพิ่มเข้ามาปรากฏครบถ้วน

---

## ขั้นตอนที่ 320: สรุปและแบบฝึกหัด

### 320.1 สรุปสิ่งที่ได้เรียนรู้ใน Part นี้

- ✅ เข้าใจว่าทำไมการตัดสินใจเรื่อง Custom User Model ต้องทำ**ก่อน** `migrate` ครั้งแรก
  เสมอ เพราะ `ForeignKey` ไปยัง `User` กระจายอยู่ทั่วโปรเจกต์และในตัว Django เอง
- ✅ เข้าใจความแตกต่างระหว่าง `AbstractUser` (มี field ครบ แค่เพิ่มใหม่) กับ
  `AbstractBaseUser` (ต้องออกแบบทุกอย่างเอง รวมถึง Manager)
- ✅ สร้าง Custom User Model ด้วย `AbstractUser` และเข้าใจข้อแลกเปลี่ยนเทียบกับแนวทาง
  `Profile` แบบ `OneToOneField` จาก Part 012
- ✅ ตั้งค่า `AUTH_USER_MODEL` และเข้าใจกลไก `swappable_dependency` ในไฟล์ migration
- ✅ เขียน Custom `UserManager` พร้อม `create_user()`/`create_superuser()` ของตัวเอง
- ✅ สร้าง Custom User Model แบบเต็มรูปแบบด้วย `AbstractBaseUser` สำหรับ login ด้วยอีเมล
  (`USERNAME_FIELD`, `REQUIRED_FIELDS`, `PermissionsMixin`)
- ✅ เข้าใจความเสี่ยงและเทคนิคการย้ายโปรเจกต์ที่มีข้อมูลจริงไปใช้ Custom User Model
- ✅ ใช้ `get_user_model()` และ `settings.AUTH_USER_MODEL` อย่างถูกต้องตามบริบท แทนการ
  import `User` ตรง ๆ
- ✅ ปรับแต่ง `UserAdmin` ให้รองรับ Custom User Model ทั้งสองแบบใน Django Admin

### 320.2 Checklist ก่อนไป Part ถัดไป

- [ ] อธิบายได้ว่าทำไมการเปลี่ยน `AUTH_USER_MODEL` กลางทางถึงเสี่ยงต่อข้อมูลเสียหาย
- [ ] แยกความแตกต่างระหว่าง `AbstractUser` กับ `AbstractBaseUser` ได้โดยไม่ต้องเปิดตำรา
- [ ] สร้างโปรเจกต์ `usermodel-lab` และมี `CustomUser` ที่ extend `AbstractUser` ทำงานได้
- [ ] อธิบาย `migrations.swappable_dependency()` ได้ว่ามันแก้ปัญหาอะไร
- [ ] เขียน `CustomUserManager` ที่มี `create_user()`/`create_superuser()` เองได้
- [ ] สร้าง Custom User Model ที่ login ด้วยอีเมลได้สำเร็จ พร้อม `createsuperuser`
- [ ] อธิบายได้ว่าเมื่อไหร่ควรใช้ `get_user_model()` เมื่อไหร่ควรใช้
      `settings.AUTH_USER_MODEL`
- [ ] ปรับแต่ง `UserAdmin` ให้ใช้งานได้ครบทั้งหน้าเพิ่มและหน้าแก้ไขผู้ใช้

### 320.3 แบบฝึกหัดท้ายบท

**แบบฝึกหัดที่ 1**: ในโปรเจกต์ `usermodel-lab` ที่สร้างไว้ในขั้นตอนที่ 313 ให้เพิ่ม field
ใหม่เข้าไปใน `CustomUser` (แบบ `AbstractUser`) ชื่อ `preferred_language` เป็น
`CharField` ที่มีตัวเลือกจำกัด (`choices`) ระหว่าง `'th'` (ไทย) และ `'en'` (อังกฤษ)
ค่าเริ่มต้นเป็น `'th'` แล้วปรับ `CustomUserAdmin` ให้แสดง field นี้ทั้งใน `list_display`
และ `fieldsets`

**แบบฝึกหัดที่ 2**: สร้างโปรเจกต์ทดลองใหม่อีกโปรเจกต์หนึ่ง (`usermodel-lab-email`)
ที่ใช้ `AbstractBaseUser` แบบเต็มรูปแบบตามขั้นตอนที่ 316 แต่เปลี่ยนให้ระบบ login ด้วย
**เบอร์โทรศัพท์** แทนอีเมล (`USERNAME_FIELD = 'phone_number'`) โดยยังคงเก็บ `email`
ไว้เป็น field เสริมใน `REQUIRED_FIELDS` แล้วทดสอบสร้าง superuser ให้สำเร็จ

**แบบฝึกหัดที่ 3**: เขียนอธิบายด้วยคำพูดของตัวเอง (บันทึกลงไฟล์ `notes.md`) ว่าถ้า
โปรเจกต์บล็อกหลักของหลักสูตรนี้ (ที่ migrate ไปแล้วตั้งแต่ Part 004 และมี `Profile`
จาก Part 012) ต้องการเปลี่ยนไปใช้ Custom User Model จริง ๆ ในอนาคต ควรทำตามขั้นตอนใด
บ้างตามที่เรียนในขั้นตอนที่ 317 พร้อมระบุความเสี่ยงที่ต้องระวังในแต่ละขั้น

**แบบฝึกหัดที่ 4 (ขั้นสูง)**: ในโปรเจกต์ `usermodel-lab` ให้เขียน management command
ชื่อ `list_verified_authors` ที่ใช้ `get_user_model()` (ไม่ import `CustomUser` ตรง ๆ)
เพื่อแสดงรายชื่อผู้ใช้ทั้งหมดที่ `is_verified_author=True` เรียงตามวันที่สมัคร
(`date_joined`) จากเก่าไปใหม่ พร้อมทดสอบว่าคำสั่งทำงานถูกต้องด้วยข้อมูลตัวอย่างอย่างน้อย
3 บัญชี

### 320.4 คำถามที่พบบ่อย (FAQ)

**Q: ถ้าฉันไม่รู้ตั้งแต่ต้นว่าจะต้องเพิ่ม field พิเศษให้ User ควรสร้าง Custom User Model
ไว้เผื่อไหม?**
A: ควรอย่างยิ่ง สร้าง `AbstractUser` เปล่า ๆ ที่ยังไม่เพิ่ม field อะไรเลยไว้ตั้งแต่ต้น
ก็เพียงพอแล้ว ต้นทุนแทบเป็นศูนย์ (แค่ไฟล์ model กับการตั้งค่า `AUTH_USER_MODEL`) แต่เปิด
ทางเลือกไว้เต็มที่สำหรับอนาคต ต่างจากการไม่ทำอะไรเลยที่จะทำให้ต้องเจอสถานการณ์ในขั้นตอน
ที่ 317 ถ้าวันหนึ่งจำเป็นต้องเปลี่ยนจริง ๆ

**Q: ทำไมโปรเจกต์บล็อกหลักของหลักสูตรนี้ถึงไม่เปลี่ยนไปใช้ Custom User Model ตาม Part นี้?**
A: เพราะโปรเจกต์บล็อกหลัก migrate ไปตั้งแต่ Part 004 ด้วย `User` มาตรฐานแล้ว และมี
ข้อมูลผู้ใช้ทดสอบสะสมมาตลอดหลักสูตร การเปลี่ยนตอนนี้จะมีความเสี่ยงตามที่อธิบายในขั้นตอน
ที่ 311 และ 317 โดยไม่จำเป็น เนื่องจาก `Profile` แบบ `OneToOneField` (Part 012)
ก็ตอบโจทย์ความต้องการเพิ่มข้อมูลผู้ใช้ได้เพียงพอแล้วสำหรับกรณีนี้ นี่คือตัวอย่างการนำ
กฎในขั้นตอนที่ 311.4 มาใช้จริงในทางปฏิบัติ

**Q: `AbstractUser` กับ `AbstractBaseUser` ใช้ผสมกันในโปรเจกต์เดียวได้ไหม?**
A: ไม่ได้ — ทั้งโปรเจกต์มี User model ที่ `AUTH_USER_MODEL` ชี้ไปได้แค่ตัวเดียวเท่านั้น
ต้องเลือกแบบใดแบบหนึ่งตั้งแต่ต้น (ตามตารางเปรียบเทียบในขั้นตอนที่ 312.4) แต่สามารถมี
`Profile` หรือโมเดลเสริมอื่น ๆ เชื่อมด้วย `OneToOneField` เพิ่มได้อีกชั้นเสมอไม่ว่าจะเลือก
แบบไหน

**Q: ถ้าใช้ django-allauth หรือ third-party authentication package ต้องทำอะไรเพิ่มไหม?**
A: package สมัยใหม่ส่วนใหญ่ (รวมถึง django-allauth) ออกแบบมาให้ทำงานกับ Custom User
Model ได้อยู่แล้วผ่าน `get_user_model()`/`settings.AUTH_USER_MODEL` ตามมาตรฐานที่เรียน
ใน Part นี้ แต่ควรอ่านเอกสารของ package นั้น ๆ ก่อนติดตั้งเสมอว่ามีขั้นตอนตั้งค่าเพิ่มเติม
เฉพาะหรือไม่ (เช่น บาง package ต้องการให้ `USERNAME_FIELD` เป็น `email` โดยเฉพาะ) —
เราจะเจาะลึกการรวม `django-allauth` เข้ากับ Custom User Model ใน Part 036

### 320.5 เตรียมตัวสำหรับ Part ถัดไป

**Part 033: Permissions และ Groups** จะพาไปเจาะลึกระบบสิทธิ์ของ Django ที่เราแตะไป
เพียงผิวเผินใน Part นี้ผ่าน `PermissionsMixin` — คุณจะได้เรียนรู้ `Permission` model
มาตรฐาน 4 สิทธิ์ต่อโมเดล (add/change/delete/view), การสร้าง Custom Permission ของตัวเอง,
การจัดกลุ่มผู้ใช้ด้วย `Group`, การตรวจสอบสิทธิ์ใน view ด้วย `@permission_required` และ
`PermissionRequiredMixin`, และการออกแบบระบบสิทธิ์ให้เหมาะกับบทบาทต่าง ๆ ในระบบบล็อก
(ผู้เขียน, บรรณาธิการ, ผู้ดูแลระบบ) ซึ่งจะใช้ทั้ง `User`/`CustomUser` ที่เรียนใน Part นี้
เป็นฐานทั้งหมด

เตรียมทบทวนแนวคิด `is_staff`, `is_superuser`, `groups`, และ `user_permissions` ที่ผ่าน
มาให้แม่นก่อนไปต่อ เพราะ Part 033 จะใช้ทุก field เหล่านี้อย่างเข้มข้น!
