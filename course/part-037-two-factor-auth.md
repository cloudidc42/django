# Part 037: Two-Factor Authentication

> **ขั้นตอนที่ 361-370 ของหลักสูตร** | Phase 4: Authentication, Users และ Permissions
>
> เป้าหมายของ Part นี้: เติมเต็มสิ่งที่ Part 018 (Django Admin ขั้นสูง) และ Part 035
> (Password Management) แขวนคำสัญญาไว้ — การเปิดใช้ **Two-Factor Authentication (2FA)**
> แบบเต็มรูปแบบ ตั้งแต่แนวคิด TOTP, การติดตั้ง `django-otp`, การสร้างหน้าสแกน QR Code
> ด้วยตัวเอง, Backup Codes กรณีมือถือหาย, การบังคับให้ staff/admin ทุกคนต้องเปิด 2FA,
> การผูก 2FA เข้ากับ Django Admin, ไปจนถึงภาพรวม SMS 2FA และ WebAuthn/Passkeys ซึ่งเป็น
> อนาคตของการยืนยันตัวตน เมื่อจบ Part นี้ ระบบบล็อกของคุณจะมี 2FA ที่ใช้งานได้จริงระดับ
> production สำหรับบัญชี staff ทุกคน พร้อม test ครอบคลุม flow ทั้งหมด

---

## สารบัญของ Part นี้

- ขั้นตอนที่ 361: แนวคิด 2FA และ TOTP (Time-based One-Time Password)
- ขั้นตอนที่ 362: ติดตั้งและตั้งค่า django-otp
- ขั้นตอนที่ 363: TOTP Device Setup Flow เต็มรูปแบบ พร้อม QR Code
- ขั้นตอนที่ 364: Backup Codes — รหัสสำรองกรณีมือถือหาย
- ขั้นตอนที่ 365: บังคับ Staff/Admin ให้เปิด 2FA ด้วย Middleware และ Decorator
- ขั้นตอนที่ 366: ผสาน 2FA เข้ากับ Django Admin ด้วย django-otp
- ขั้นตอนที่ 367: SMS-based 2FA เทียบกับ TOTP — ต้นทุนและความเสี่ยง SIM Swap
- ขั้นตอนที่ 368: WebAuthn/Passkeys — อนาคตของการยืนยันตัวตนแบบไม่ใช้รหัสผ่าน
- ขั้นตอนที่ 369: การเขียน Test สำหรับ Flow ที่มี 2FA
- ขั้นตอนที่ 370: สรุปและแบบฝึกหัด — เปิดใช้ 2FA จริงสำหรับ staff ของบล็อก

---

## ขั้นตอนที่ 361: แนวคิด 2FA และ TOTP (Time-based One-Time Password)

### 361.1 ทำไมรหัสผ่านอย่างเดียวไม่พอ

ต่อให้บังคับใช้ password policy ที่รัดกุมที่สุดตามที่เรียนไปใน Part 035 (ความยาว
ขั้นต่ำ, ตรวจสอบกับ breach database ผ่าน `django-zxcvbn`/HaveIBeenPwned, hasher ที่
แข็งแรงอย่าง Argon2) รหัสผ่านก็ยังมีจุดอ่อนที่แก้ไม่ได้ด้วยนโยบายเพียงอย่างเดียว:

- **Phishing**: ผู้ใช้ถูกหลอกให้กรอกรหัสผ่านในเว็บปลอม
- **Credential Stuffing**: รหัสผ่านรั่วจากเว็บอื่นแล้วถูกเอามาลองซ้ำที่เว็บเรา
  (คนจำนวนมากใช้รหัสผ่านซ้ำกันหลายเว็บ)
- **Keylogger / Malware**: มัลแวร์บนเครื่องผู้ใช้ดักจับรหัสผ่านตอนพิมพ์
- **Data Breach ฝั่งเซิร์ฟเวอร์อื่น**: แม้เราจะ hash รหัสผ่านอย่างถูกต้อง แต่เว็บอื่น
  ที่ผู้ใช้ใช้รหัสผ่านเดียวกันอาจไม่ได้ทำแบบนั้น

**หลักการของ Multi-Factor Authentication (MFA)** คือการยืนยันตัวตนด้วย **ปัจจัยที่
เป็นอิสระต่อกันตั้งแต่ 2 ประเภทขึ้นไป** เพื่อให้ผู้โจมตีที่ขโมยปัจจัยหนึ่งได้ ยังคง
เข้าระบบไม่ได้เพราะขาดอีกปัจจัย:

| ประเภทปัจจัย | ตัวอย่าง | จุดอ่อน |
|---|---|---|
| **สิ่งที่คุณรู้** (Something you know) | รหัสผ่าน, PIN, คำตอบคำถามลับ | ขโมย/เดา/รั่วได้ |
| **สิ่งที่คุณมี** (Something you have) | มือถือที่รัน Authenticator App, Hardware Security Key, ซิมการ์ด | มือถือหาย/ถูกขโมย, SIM Swap |
| **สิ่งที่คุณเป็น** (Something you are) | ลายนิ้วมือ, Face ID, ม่านตา | ปลอมยากมาก แต่เปลี่ยนไม่ได้ถ้ารั่ว |

**2FA (Two-Factor Authentication)** คือ MFA ที่ใช้ปัจจัย 2 ประเภท โดยทั่วไปคือ
"รหัสผ่าน" (สิ่งที่รู้) + "รหัสจากมือถือ" (สิ่งที่มี) — Part นี้เจาะลึกปัจจัยที่สอง

### 361.2 TOTP คืออะไร

**TOTP (Time-based One-Time Password)** คือมาตรฐานเปิด (**RFC 6238**) ที่สร้างรหัส
6 หลักซึ่ง **เปลี่ยนทุก 30 วินาที** และ **ใช้ได้เพียงครั้งเดียว** โดยไม่ต้องมี
การเชื่อมต่ออินเทอร์เน็ตระหว่างมือถือกับเซิร์ฟเวอร์เลย นี่คือกลไกเบื้องหลังแอปอย่าง
Google Authenticator, Microsoft Authenticator, Authy, และ 1Password

### 361.3 TOTP ทำงานอย่างไร (เจาะลึกอัลกอริทึม)

TOTP ต่อยอดจาก **HOTP (HMAC-based One-Time Password, RFC 4226)** โดยแทนที่ตัวนับ
(counter) ด้วยเวลาปัจจุบัน สูตรคร่าว ๆ คือ:

```
TOTP(K, T) = HOTP(K, T)
T = floor((เวลาปัจจุบันแบบ Unix timestamp - T0) / X)

K  = Shared Secret Key (สุ่มสร้างครั้งเดียว ตอนตั้งค่า 2FA — เก็บทั้งฝั่ง server และในแอป)
T0 = เวลาเริ่มต้นนับ (ปกติคือ 0 = Unix Epoch)
X  = ช่วงเวลาต่อ 1 step (ค่ามาตรฐานคือ 30 วินาที)
```

ขั้นตอนแบบละเอียด:

1. ตอนตั้งค่า 2FA เซิร์ฟเวอร์สุ่มสร้าง **Secret Key** (K) หนึ่งชุด แล้วส่งให้ผู้ใช้
   ผ่าน QR Code (หรือพิมพ์เอง) — Secret Key นี้ถูกเก็บไว้ **ทั้งสองฝั่ง**: ในฐานข้อมูล
   ของเซิร์ฟเวอร์ และในแอป Authenticator บนมือถือผู้ใช้
2. ทุก ๆ 30 วินาที ทั้งมือถือและเซิร์ฟเวอร์คำนวณ `T` จากเวลาปัจจุบันแบบเดียวกัน
   (จึงต้องให้ **นาฬิกาเครื่อง sync กันแม่นยำ** ผ่าน NTP — นี่คือสาเหตุอันดับหนึ่งที่
   ผู้ใช้เจอปัญหา "กรอกรหัสถูกแต่ระบบบอกผิด" เมื่อนาฬิกามือถือคลาดเคลื่อน)
3. ทั้งสองฝั่งคำนวณ `HMAC-SHA1(K, T)` แล้วตัดผลลัพธ์เหลือ 6 หลัก (Dynamic Truncation)
4. ถ้าตัวเลข 6 หลักที่ผู้ใช้กรอกตรงกับที่เซิร์ฟเวอร์คำนวณได้ (โดยทั่วไปยอมรับ step
   ก่อนหน้า/ถัดไปเล็กน้อยเผื่อ clock drift) → ยืนยันตัวตนสำเร็จ

```
┌─────────────────┐                              ┌─────────────────┐
│   มือถือผู้ใช้    │                              │    เซิร์ฟเวอร์    │
│  (Authenticator) │                              │  (เก็บ Secret K) │
│                  │                              │                  │
│  K + เวลาปัจจุบัน │                              │  K + เวลาปัจจุบัน │
│       ↓          │                              │       ↓          │
│  HMAC-SHA1(K, T)  │                              │  HMAC-SHA1(K, T)  │
│       ↓          │                              │       ↓          │
│   "482913"       │  ── ผู้ใช้พิมพ์รหัสนี้เข้าเว็บ →  │   เทียบว่าตรงกัน  │
└─────────────────┘                              └─────────────────┘
       ไม่ต้องต่อเน็ต!                                  (ไม่ต้องส่งรหัส
                                                         ผ่านเครือข่ายใด ๆ
                                                         นอกจากตอน login)
```

### 361.4 ทำไม TOTP ปลอดภัยและได้รับความนิยม

- **ไม่ต้องพึ่งเครือข่ายโทรศัพท์**: ต่างจาก SMS ที่ต้องอาศัยผู้ให้บริการมือถือ
  (ซึ่งมีความเสี่ยง SIM Swap ตามที่จะเรียนในขั้นตอนที่ 367) TOTP ทำงานแบบ offline
  ล้วน ๆ บนตัวมือถือ
- **Secret Key ไม่เคยถูกส่งผ่านเครือข่ายซ้ำ**: ส่งแค่ครั้งเดียวตอนตั้งค่า (ผ่าน QR
  Code) หลังจากนั้นมีแค่รหัส 6 หลักที่ใช้ครั้งเดียวเท่านั้นที่วิ่งผ่านเครือข่าย
- **เป็นมาตรฐานเปิด**: RFC 6238 ทำให้แอป Authenticator ของค่ายไหนก็ใช้ร่วมกับระบบ
  ของเราได้ ไม่ผูกติดกับ vendor ใดเป็นพิเศษ
- **ฟรี**: ต่างจาก SMS ที่มีค่าใช้จ่ายต่อข้อความ (ผ่าน Twilio หรือผู้ให้บริการ SMS
  Gateway) TOTP ไม่มีต้นทุนต่อการยืนยันแต่ละครั้งเลย

### 361.5 ปัจจัยที่สองแบบอื่นที่ควรรู้จักไว้ก่อน (จะเจาะลึกในขั้นตอนถัดไป)

| วิธี | หลักการ | รายละเอียดอยู่ที่ |
|---|---|---|
| **TOTP** | รหัส 6 หลักเปลี่ยนทุก 30 วิ จากแอป Authenticator | ขั้นตอนที่ 362-364 (Part นี้) |
| **Backup/Recovery Codes** | รหัสสำรองใช้ครั้งเดียว เผื่อมือถือหาย | ขั้นตอนที่ 364 |
| **SMS OTP** | รหัสส่งผ่านข้อความ SMS | ขั้นตอนที่ 367 |
| **Push Notification** | กดยืนยัน "ใช่ นี่คือฉัน" ในแอป (เช่น Duo, Okta Verify) | กล่าวถึงสั้น ๆ ในขั้นตอนที่ 367 |
| **WebAuthn / Passkeys** | กุญแจเข้ารหัสสาธารณะ ผูกกับอุปกรณ์ ไม่มีรหัสให้พิมพ์เลย | ขั้นตอนที่ 368 |
| **Hardware Security Key** | อุปกรณ์ USB/NFC เฉพาะ เช่น YubiKey (ใช้โปรโตคอล WebAuthn) | ขั้นตอนที่ 368 |

หลักสูตรนี้เลือก **TOTP เป็นแกนหลัก** เพราะเป็นมาตรฐานที่ทีมงานส่วนใหญ่เลือกใช้จริง
ในโปรเจกต์ Django (ผ่านไลบรารี `django-otp`) มีต้นทุนเป็นศูนย์ และปลอดภัยกว่า SMS
อย่างชัดเจน

---

## ขั้นตอนที่ 362: ติดตั้งและตั้งค่า django-otp

### 362.1 ทำไมเลือก django-otp

**django-otp** คือไลบรารีมาตรฐานของระบบนิเวศ Django สำหรับ One-Time Password
ออกแบบมาแบบ **plugin-based**: มี core framework กลาง แล้วแยก "ชนิดของ device"
ออกเป็น plugin ต่างหาก (TOTP, HOTP, Static/Backup codes, SMS ผ่าน 3rd-party) ทำให้
เราเลือกใช้เฉพาะส่วนที่ต้องการ และเขียน UI/flow เองได้อย่างอิสระ ต่างจากไลบรารี
สำเร็จรูปอย่าง `django-two-factor-auth` ที่มาพร้อม view/template สำเร็จรูปแต่
ปรับแต่งยากกว่า — หลักสูตรนี้เลือก `django-otp` เพื่อให้คุณเข้าใจกลไกเบื้องหลัง
อย่างถ่องแท้ ก่อนจะไปหยิบไลบรารีสำเร็จรูปมาใช้ในงานจริงก็ได้

### 362.2 ติดตั้ง Package

```bash
# ต้อง activate venv ก่อนเสมอ (ตามหลักการจาก Part 001)
pip install django-otp qrcode
```

- `django-otp`: core framework + TOTP + Static device plugin
- `qrcode`: สร้างรูปภาพ QR Code จาก provisioning URI (ใช้ในขั้นตอนที่ 363)

บันทึกลง `requirements.txt` ทันทีตามวินัยที่ฝึกมาตั้งแต่ Part 001:

```bash
pip freeze > requirements.txt
```

### 362.3 เพิ่มเข้า `INSTALLED_APPS`

```python
# config/settings/base.py
INSTALLED_APPS = [
    "django.contrib.admin",
    "django.contrib.auth",
    "django.contrib.contenttypes",
    "django.contrib.sessions",
    "django.contrib.messages",
    "django.contrib.staticfiles",
    "django_otp",                        # core framework
    "django_otp.plugins.otp_totp",       # TOTP device (ขั้นตอนที่ 363)
    "django_otp.plugins.otp_static",     # Backup codes (ขั้นตอนที่ 364)
    "accounts",                          # custom User model จาก Part 032
    "blog",
]
```

### 362.4 เพิ่ม `OTPMiddleware`

`OTPMiddleware` คือหัวใจของ django-otp มันทำงาน **ต่อจาก** `AuthenticationMiddleware`
เสมอ (ต้องมาหลังเท่านั้น เพราะต้องใช้ `request.user` ที่ตั้งไว้แล้ว) หน้าที่ของมันคือ
ผูก **OTP Device ที่ผ่านการยืนยันแล้วในเซสชันนี้** เข้ากับ `request.user` และเพิ่ม
method `request.user.is_verified()` ให้ใช้งานได้ทุกที่ในระบบ:

```python
# config/settings/base.py
MIDDLEWARE = [
    "django.middleware.security.SecurityMiddleware",
    "django.contrib.sessions.middleware.SessionMiddleware",
    "django.middleware.common.CommonMiddleware",
    "django.middleware.csrf.CsrfViewMiddleware",
    "django.contrib.auth.middleware.AuthenticationMiddleware",   # ต้องมาก่อน
    "django_otp.middleware.OTPMiddleware",                       # ← เพิ่มบรรทัดนี้ต่อทันที
    "django.contrib.messages.middleware.MessageMiddleware",
    "django.middleware.clickjacking.XFrameOptionsMiddleware",
]
```

**ข้อควรระวัง**: ถ้าลืมใส่ `OTPMiddleware` หรือใส่ผิดตำแหน่ง (ก่อน
`AuthenticationMiddleware`) การเรียก `request.user.is_verified()` จะโยน
`AttributeError` ทันที เพราะ method นี้ถูก "แปะ" เข้าไปที่ instance ของ `request.user`
โดย middleware ตัวนี้เองเท่านั้น ไม่ใช่ method ปกติของ `User` model

### 362.5 รัน Migration

`django_otp`, `otp_totp`, และ `otp_static` แต่ละตัวมี model ของตัวเอง (เช่น
`TOTPDevice`, `StaticDevice`, `StaticToken`) ต้อง migrate เหมือน app ทั่วไป:

```bash
python manage.py migrate
```

ผลลัพธ์ควรเห็นตารางใหม่ถูกสร้าง เช่น `otp_totp_totpdevice`,
`otp_static_staticdevice`, `otp_static_statictoken`

### 362.6 ตรวจสอบว่าติดตั้งถูกต้อง

เปิด `python manage.py shell` แล้วลองสร้าง `TOTPDevice` เปล่า ๆ:

```python
from django.contrib.auth import get_user_model
from django_otp.plugins.otp_totp.models import TOTPDevice

User = get_user_model()
user = User.objects.first()
device = TOTPDevice.objects.create(user=user, name="ทดสอบ", confirmed=False)
print(device.key)          # secret key แบบ hex string
print(device.confirmed)    # False — ยังไม่ผ่านการยืนยัน
device.delete()            # ลบทิ้ง เป็นแค่การทดสอบ
```

ถ้ารันผ่านโดยไม่มี error แสดงว่าการติดตั้งสมบูรณ์แล้ว พร้อมสร้าง flow จริงใน
ขั้นตอนถัดไป

---

## ขั้นตอนที่ 363: TOTP Device Setup Flow เต็มรูปแบบ พร้อม QR Code

### 363.1 ภาพรวมของ Flow ที่จะสร้าง

```
1. ผู้ใช้ล็อกอินปกติ (username + password) แต่ยังไม่มี 2FA
2. ผู้ใช้กด "เปิดใช้ 2FA" → ระบบสร้าง TOTPDevice ที่ confirmed=False
3. ระบบแสดง QR Code ที่เข้ารหัส Secret Key ของ device นั้น
4. ผู้ใช้เปิดแอป Google Authenticator/Authy สแกน QR Code
   → แอปเก็บ Secret Key และเริ่มสร้างรหัส 6 หลักทุก 30 วินาที
5. ผู้ใช้พิมพ์รหัส 6 หลักปัจจุบันกลับมายืนยันในเว็บ
6. ระบบตรวจสอบด้วย device.verify_token() ถ้าถูกต้อง → confirmed=True
7. นับจากนี้ ทุกครั้งที่ login ต้องกรอกรหัส TOTP เพิ่มเติมจากรหัสผ่าน
```

### 363.2 View: สร้าง Device และแสดงหน้าตั้งค่า

```python
# accounts/views_2fa.py
from django.contrib.auth.decorators import login_required
from django.shortcuts import render, redirect
from django.contrib import messages
from django_otp.plugins.otp_totp.models import TOTPDevice
from django_otp import login as otp_login


@login_required
def totp_setup(request):
    """หน้าเริ่มต้นเปิดใช้ 2FA แบบ TOTP"""
    # ถ้ามี device ที่ confirmed แล้ว ไม่ต้องตั้งใหม่ซ้ำ
    existing = TOTPDevice.objects.filter(user=request.user, confirmed=True).first()
    if existing:
        messages.info(request, "คุณเปิดใช้ 2FA อยู่แล้ว")
        return redirect("accounts:2fa_status")

    # หา device ที่ยังไม่ confirm ของ user นี้ หรือสร้างใหม่ (กันสร้างซ้ำเวลา refresh หน้า)
    device, _created = TOTPDevice.objects.get_or_create(
        user=request.user,
        confirmed=False,
        defaults={"name": "default"},
    )

    if request.method == "POST":
        token = request.POST.get("token", "").strip()
        if device.verify_token(token):
            device.confirmed = True
            device.save()
            # ผูก device นี้เข้ากับ session ปัจจุบันทันที ไม่ต้องกรอกซ้ำรอบนี้
            otp_login(request, device)
            messages.success(request, "เปิดใช้ 2FA สำเร็จแล้ว! กรุณาบันทึก Backup Codes ต่อไป")
            return redirect("accounts:2fa_backup_codes")
        messages.error(request, "รหัสไม่ถูกต้อง กรุณาลองใหม่ (ตรวจสอบเวลาบนมือถือด้วย)")

    return render(request, "accounts/2fa_setup.html", {"device": device})
```

### 363.3 สร้าง Provisioning URI และ QR Code

มาตรฐาน **Key URI Format** ที่แอป Authenticator ทุกยี่ห้อเข้าใจตรงกันคือรูปแบบ
`otpauth://totp/...` เราสร้าง URI นี้เองจาก `device.bin_key` (raw secret เป็น
bytes) แล้วเข้ารหัสเป็น Base32 ตามสเปกของ RFC 6238:

```python
# accounts/otp_utils.py
import base64
import urllib.parse


def build_totp_provisioning_uri(device, issuer="MyCompany Blog"):
    """สร้าง otpauth:// URI ตามสเปกที่ Google Authenticator/Authy เข้าใจ"""
    secret_base32 = base64.b32encode(device.bin_key).decode("utf-8").rstrip("=")
    label = urllib.parse.quote(f"{issuer}:{device.user.email}")
    params = urllib.parse.urlencode({
        "secret": secret_base32,
        "issuer": issuer,
        "algorithm": "SHA1",
        "digits": device.digits,   # ปกติคือ 6
        "period": device.step,     # ปกติคือ 30 วินาที
    })
    return f"otpauth://totp/{label}?{params}"
```

ต่อด้วย view ที่ render URI นี้เป็นรูปภาพ QR Code แบบ PNG โดยไม่ต้องเซฟไฟล์ลงดิสก์
(สร้างในหน่วยความจำแล้วส่งกลับเป็น `HttpResponse` ทันที):

```python
# accounts/views_2fa.py (เพิ่มต่อจากด้านบน)
import io
import qrcode
from django.http import HttpResponse, Http404
from .otp_utils import build_totp_provisioning_uri


@login_required
def totp_qr_code(request):
    """คืนค่าเป็นรูปภาพ PNG ของ QR Code สำหรับ device ที่ยังไม่ confirm ของ user นี้"""
    device = TOTPDevice.objects.filter(user=request.user, confirmed=False).first()
    if device is None:
        raise Http404("ไม่พบ TOTP device ที่รอการยืนยัน")

    uri = build_totp_provisioning_uri(device)
    qr_image = qrcode.make(uri)

    buffer = io.BytesIO()
    qr_image.save(buffer, format="PNG")
    return HttpResponse(buffer.getvalue(), content_type="image/png")
```

### 363.4 URLs

```python
# accounts/urls.py
from django.urls import path
from . import views_2fa

app_name = "accounts"

urlpatterns = [
    # ... login/logout/password urls จาก Part 031, 035 ...
    path("2fa/setup/", views_2fa.totp_setup, name="2fa_setup"),
    path("2fa/qr-code/", views_2fa.totp_qr_code, name="2fa_qr_code"),
]
```

### 363.5 Template หน้าตั้งค่า

```html
{# templates/accounts/2fa_setup.html #}
{% extends "base.html" %}

{% block content %}
<div class="container" style="max-width: 480px; margin: 40px auto;">
    <h1>เปิดใช้ Two-Factor Authentication</h1>

    <ol>
        <li>เปิดแอป <strong>Google Authenticator</strong>, <strong>Authy</strong>
            หรือ <strong>Microsoft Authenticator</strong> บนมือถือของคุณ</li>
        <li>เลือก "เพิ่มบัญชี" (Add Account) แล้วสแกน QR Code ด้านล่าง</li>
        <li>กรอกรหัส 6 หลักที่แอปแสดงในช่องด้านล่าง แล้วกดยืนยัน</li>
    </ol>

    <div style="text-align: center; margin: 24px 0;">
        <img src="{% url 'accounts:2fa_qr_code' %}" alt="TOTP QR Code"
             style="border: 8px solid #fff; box-shadow: 0 0 4px rgba(0,0,0,.2);">
    </div>

    <details style="margin-bottom: 16px;">
        <summary>สแกนไม่ได้? กรอก Secret Key ด้วยมือ</summary>
        <p>เปิดแอป Authenticator เลือก "กรอกรหัสด้วยตัวเอง" แล้วใช้ค่าที่แสดงในหน้า
        รายละเอียดบัญชีของคุณ</p>
    </details>

    <form method="post">
        {% csrf_token %}
        <label for="token">รหัส 6 หลักจากแอป Authenticator</label>
        <input type="text" id="token" name="token" inputmode="numeric"
               pattern="[0-9]{6}" maxlength="6" autocomplete="one-time-code" required
               style="font-size: 1.5rem; letter-spacing: 0.3rem; width: 100%; padding: 8px;">
        <button type="submit" style="margin-top: 12px;">ยืนยันและเปิดใช้ 2FA</button>
    </form>
</div>
{% endblock %}
```

### 363.6 อธิบายจุดสำคัญที่มือใหม่มักพลาด

- **`device.bin_key` ไม่ใช่ `device.key`**: `device.key` คือ hex string (มนุษย์อ่านได้
  ง่ายสำหรับ debug) แต่ Base32 ที่ต้องใส่ใน URI ต้องเข้ารหัสจาก `device.bin_key`
  (raw bytes) เท่านั้น ถ้าเผลอเอา `device.key` (hex) ไป base32-encode ตรง ๆ
  QR Code จะสแกนได้แต่รหัสจะไม่ตรงกับที่เซิร์ฟเวอร์คำนวณ
- **`get_or_create(confirmed=False, ...)`**: ป้องกันไม่ให้ผู้ใช้ refresh หน้าแล้ว
  ได้ Secret Key ใหม่ทุกครั้ง (ถ้าสร้างใหม่ทุกครั้ง QR Code เดิมที่สแกนไปแล้วจะใช้
  ไม่ได้อีก เพราะ Secret Key เปลี่ยนไป)
- **`otp_login(request, device)`**: เป็นคนละฟังก์ชันกับ `django.contrib.auth.login()`
  ฟังก์ชันนี้ทำหน้าที่ "บอกเซสชันปัจจุบันว่า device นี้ผ่านการยืนยันแล้ว" ทำให้
  `request.user.is_verified()` คืนค่า `True` ทันทีโดยไม่ต้องให้ผู้ใช้กรอกรหัสซ้ำ
  รอบเดียวกัน
- **`autocomplete="one-time-code"`**: attribute HTML มาตรฐานที่ทำให้เบราว์เซอร์/iOS
  เสนอ autofill รหัส OTP จาก SMS หรือ Keychain ให้อัตโนมัติ เป็น UX ที่ดีที่ควรใส่ไว้
  เสมอในฟอร์มกรอกรหัส OTP

---

## ขั้นตอนที่ 364: Backup Codes — รหัสสำรองกรณีมือถือหาย

### 364.1 ทำไม Backup Codes จำเป็นเสมอ

ถ้าผู้ใช้เปิด TOTP แล้ว **ทำมือถือหาย พัง หรือรีเซ็ตเครื่องโดยไม่ได้สำรอง** Secret
Key ไว้ พวกเขาจะ **ล็อกตัวเองออกจากระบบถาวร** เพราะไม่มีทางสร้างรหัส 6 หลักที่ถูกต้อง
ได้อีก นี่คือเหตุผลที่ระบบ 2FA ระดับมืออาชีพทุกระบบ (Google, GitHub, AWS) ต้องมี
**Backup Codes** หรือ **Recovery Codes** เป็นทางออกสำรองเสมอ

### 364.2 กลไก `StaticDevice` ของ django-otp

`django_otp.plugins.otp_static` ให้ model 2 ตัวสำหรับกรณีนี้โดยเฉพาะ:

- `StaticDevice`: ตัวแทน "อุปกรณ์" ที่เป็นชุดรหัสสำรอง (1 ผู้ใช้มักมีแค่ 1 device
  ชื่อ `backup`)
- `StaticToken`: รหัสแต่ละตัวในชุดนั้น (ปกติสร้าง 10 รหัสต่อครั้ง) — **แต่ละรหัส
  ใช้ได้เพียงครั้งเดียว** เมื่อใช้แล้ว django-otp จะลบ token นั้นออกจากฐานข้อมูล
  โดยอัตโนมัติ

### 364.3 View: สร้าง Backup Codes ชุดใหม่

```python
# accounts/views_2fa.py (เพิ่มต่อ)
from django_otp.plugins.otp_static.models import StaticDevice, StaticToken

BACKUP_CODE_COUNT = 10


@login_required
def totp_backup_codes(request):
    """สร้าง (หรือสร้างใหม่ทับของเดิม) ชุด Backup Codes 10 รหัส"""
    has_totp = TOTPDevice.objects.filter(user=request.user, confirmed=True).exists()
    if not has_totp:
        messages.error(request, "ต้องเปิดใช้ TOTP ก่อนจึงจะสร้าง Backup Codes ได้")
        return redirect("accounts:2fa_setup")

    static_device, _created = StaticDevice.objects.get_or_create(
        user=request.user, name="backup"
    )

    generated_codes = None
    if request.method == "POST":
        # ลบชุดเก่าทิ้งทั้งหมดก่อนสร้างชุดใหม่ (regenerate = revoke ของเก่าทันที)
        static_device.token_set.all().delete()
        generated_codes = []
        for _ in range(BACKUP_CODE_COUNT):
            token_value = StaticToken.random_token()
            StaticToken.objects.create(device=static_device, token=token_value)
            generated_codes.append(token_value)
        messages.success(
            request,
            "สร้าง Backup Codes ชุดใหม่แล้ว กรุณาบันทึกไว้ในที่ปลอดภัย "
            "(รหัสชุดเก่าทั้งหมดถูกยกเลิกแล้ว)",
        )

    remaining_count = static_device.token_set.count()
    return render(
        request,
        "accounts/2fa_backup_codes.html",
        {"generated_codes": generated_codes, "remaining_count": remaining_count},
    )
```

### 364.4 Template แสดงและดาวน์โหลด Backup Codes

```html
{# templates/accounts/2fa_backup_codes.html #}
{% extends "base.html" %}

{% block content %}
<div class="container" style="max-width: 480px; margin: 40px auto;">
    <h1>Backup Codes</h1>

    {% if generated_codes %}
        <p style="color: #b00020; font-weight: bold;">
            ⚠️ นี่เป็นครั้งเดียวที่ระบบจะแสดงรหัสเหล่านี้ให้เห็นแบบเต็ม!
            กรุณาบันทึกไว้ในที่ปลอดภัย (password manager หรือพิมพ์เก็บใส่ลิ้นชัก)
            ก่อนออกจากหน้านี้
        </p>
        <pre style="background: #f5f5f5; padding: 16px; font-size: 1.1rem;
                    line-height: 1.8; user-select: all;">{% for code in generated_codes %}{{ code }}
{% endfor %}</pre>
        <a href="data:text/plain;charset=utf-8,{{ generated_codes|join:'%0A' }}"
           download="blog-backup-codes.txt" class="button">⬇️ ดาวน์โหลดเป็นไฟล์ .txt</a>
    {% else %}
        <p>คุณมี Backup Codes เหลืออยู่ <strong>{{ remaining_count }}</strong> รหัส
        แต่ละรหัสใช้ได้เพียงครั้งเดียว</p>
    {% endif %}

    <form method="post" style="margin-top: 24px;">
        {% csrf_token %}
        <button type="submit" onclick="return confirm('รหัสชุดเก่าทั้งหมดจะถูกยกเลิกทันที ยืนยันสร้างชุดใหม่?');">
            🔄 สร้าง Backup Codes ชุดใหม่ (ยกเลิกชุดเก่า)
        </button>
    </form>
</div>
{% endblock %}
```

### 364.5 การใช้ Backup Code ตอน Login (แทน TOTP)

ที่หน้ากรอกรหัส 2FA ตอน login เราต้องรับได้ทั้ง TOTP token และ Backup Code ในช่อง
เดียวกัน โดยลองไล่ตรวจสอบทีละ device ของผู้ใช้ด้วย `django_otp.match_token()`:

```python
# accounts/views_2fa.py (เพิ่มต่อ — ใช้ในหน้า login step ที่สอง)
from django_otp import match_token


def verify_2fa_code(request, user, submitted_code):
    """ลองจับคู่รหัสที่กรอกกับ device ทุกประเภทของ user คนนี้
    ใช้ได้ทั้งรหัส TOTP 6 หลัก และ Backup Code"""
    device = match_token(user, submitted_code)
    if device is not None:
        otp_login(request, device)
        return True
    return False
```

`match_token()` เป็นฟังก์ชัน core ของ django-otp ที่ไล่ตรวจสอบทุก device
(`TOTPDevice`, `StaticDevice`) ของผู้ใช้คนนั้นให้อัตโนมัติ ไม่ต้องเขียน if-else
แยกตามประเภท device เอง — เมื่อ backup code ถูกใช้ มันจะถูกลบออกจาก
`StaticToken` โดยอัตโนมัติ (consumed = ใช้ครั้งเดียวจริง)

### 364.6 แจ้งเตือนผู้ใช้เมื่อ Backup Codes เหลือน้อย

Business logic ที่ระบบระดับมืออาชีพควรมี: แจ้งเตือนเมื่อเหลือ backup code น้อยกว่า
เกณฑ์ที่กำหนด เพื่อกันผู้ใช้ใช้จนหมดโดยไม่รู้ตัว:

```python
# accounts/views_2fa.py
LOW_BACKUP_CODE_THRESHOLD = 3


def check_low_backup_codes(request, user):
    static_device = StaticDevice.objects.filter(user=user, name="backup").first()
    if static_device and static_device.token_set.count() <= LOW_BACKUP_CODE_THRESHOLD:
        messages.warning(
            request,
            f"⚠️ คุณเหลือ Backup Codes เพียง {static_device.token_set.count()} รหัส "
            "แนะนำให้สร้างชุดใหม่ในหน้าตั้งค่าความปลอดภัย",
        )
```

เรียกฟังก์ชันนี้ทุกครั้งหลัง login สำเร็จ (เช่นใน middleware หรือ signal
`user_logged_in` ที่เรียนไปใน Part 019)

---

## ขั้นตอนที่ 365: บังคับ Staff/Admin ให้เปิด 2FA ด้วย Middleware และ Decorator

### 365.1 ทำไมต้อง "บังคับ" ไม่ใช่แค่ "แนะนำ"

Checklist ความปลอดภัยของ Admin ใน Part 018 (ขั้นตอนที่ 178.7) ระบุไว้ชัดเจนว่า
**"เปิดใช้ 2FA สำหรับทุกบัญชี staff"** ต้องเป็นข้อบังคับ ไม่ใช่ทางเลือก เพราะบัญชี
staff มีสิทธิ์เข้าถึง/แก้ไขข้อมูลระดับระบบ ถ้าปล่อยให้ staff คนใดคนหนึ่งเลือกไม่เปิด
2FA บัญชีนั้นจะกลายเป็นจุดอ่อนที่สุดของทั้งระบบทันที (weakest link)

### 365.2 กลยุทธ์: Middleware บังคับ Enroll ก่อนใช้งานต่อ

เราจะเขียน Middleware ที่ตรวจสอบทุก request ของ staff/superuser ว่า **"ผ่านการ
ยืนยัน 2FA ในเซสชันนี้แล้วหรือยัง"** ถ้ายัง ให้ redirect ไปหน้าบังคับตั้งค่า 2FA
ก่อนเสมอ ไม่ว่าจะพยายามเข้าหน้าไหนก็ตาม (ยกเว้นหน้าที่จำเป็นต่อ flow เอง เช่น
หน้าตั้งค่า 2FA, หน้า logout):

```python
# accounts/middleware.py
from django.shortcuts import redirect
from django.urls import reverse
from django.contrib import messages

# path ที่ยกเว้นไม่ต้องเช็ค (ไม่งั้นจะเกิด redirect loop)
EXEMPT_PATH_NAMES = {
    "accounts:2fa_setup",
    "accounts:2fa_qr_code",
    "accounts:2fa_backup_codes",
    "accounts:logout",
    "accounts:login",
}


class EnforceStaff2FAMiddleware:
    """บังคับให้ staff/superuser ทุกคนต้องเปิดและยืนยัน 2FA ก่อนใช้งานระบบต่อ"""

    def __init__(self, get_response):
        self.get_response = get_response
        # cache reverse() ไว้ล่วงหน้า ลดภาระคำนวณซ้ำทุก request
        self.exempt_paths = {reverse(name) for name in EXEMPT_PATH_NAMES}

    def __call__(self, request):
        user = request.user
        if (
            user.is_authenticated
            and (user.is_staff or user.is_superuser)
            and request.path not in self.exempt_paths
            and not user.is_verified()
        ):
            messages.warning(
                request,
                "บัญชีของคุณมีสิทธิ์ staff จำเป็นต้องเปิดใช้ Two-Factor "
                "Authentication ก่อนใช้งานระบบต่อ",
            )
            return redirect("accounts:2fa_setup")
        return self.get_response(request)
```

ลงทะเบียนต่อจาก `OTPMiddleware` เสมอ (เพราะต้องใช้ `is_verified()` ที่
`OTPMiddleware` เตรียมไว้ให้):

```python
# config/settings/base.py
MIDDLEWARE = [
    # ...
    "django.contrib.auth.middleware.AuthenticationMiddleware",
    "django_otp.middleware.OTPMiddleware",
    "accounts.middleware.EnforceStaff2FAMiddleware",   # ← ต้องมาหลัง OTPMiddleware เสมอ
    "django.contrib.messages.middleware.MessageMiddleware",
]
```

**ข้อควรระวังสำคัญ**: `user.is_verified()` คืนค่า `False` เสมอถ้า user ยังไม่มี
device ที่ confirmed เลย (ยังไม่เคย enroll) **และ** คืนค่า `False` ถ้า enroll แล้ว
แต่ยังไม่ได้ยืนยันในเซสชันปัจจุบัน (เช่น login รอบใหม่แต่ยังไม่กรอกรหัส 2FA) —
Middleware นี้จึงครอบคลุมทั้ง 2 กรณีในตรรกะเดียว

### 365.3 Decorator สำหรับ View รายตัว (ทางเลือกที่ยืดหยุ่นกว่า)

บางระบบอาจไม่ต้องการบังคับทั้งเว็บ แต่ต้องการบังคับเฉพาะบาง view ที่สำคัญ
`django_otp.decorators.otp_required` ให้มาพร้อมใช้:

```python
# blog/views.py
from django_otp.decorators import otp_required


@otp_required(login_url="accounts:login", if_configured=True)
def sensitive_report_view(request):
    """หน้าที่ต้องผ่าน 2FA เท่านั้น (ถ้า user มี device ตั้งไว้แล้ว)"""
    ...
```

- `if_configured=True`: บังคับ 2FA เฉพาะ user ที่ **มี device ตั้งไว้แล้ว**
  เท่านั้น (user ที่ยังไม่เคย enroll จะผ่านได้ปกติ) — เหมาะกับช่วง rollout ที่ยัง
  ไม่บังคับทุกคน
- `if_configured=False` (ค่า default): บังคับทุกคนต้องผ่าน 2FA ไม่ว่าจะ enroll
  ไว้หรือไม่ (คนที่ไม่เคย enroll จะเข้าไม่ได้เลยจนกว่าจะไปตั้งค่าก่อน)

### 365.4 Custom Decorator: บังคับเฉพาะ Staff ต้อง Enroll (ผสาน Middleware + Decorator)

สำหรับ view ที่อยากประกาศชัดเจนในระดับโค้ดว่า "ต้องเป็น staff และต้องผ่าน 2FA"
โดยไม่พึ่ง middleware ระดับ global เขียน decorator เองได้ดังนี้:

```python
# accounts/decorators.py
from functools import wraps
from django.contrib.auth.decorators import login_required
from django.shortcuts import redirect
from django.contrib import messages


def staff_2fa_required(view_func):
    @wraps(view_func)
    @login_required
    def _wrapped(request, *args, **kwargs):
        if not (request.user.is_staff or request.user.is_superuser):
            messages.error(request, "ต้องเป็น staff เท่านั้นจึงจะเข้าถึงหน้านี้ได้")
            return redirect("home")
        if not request.user.is_verified():
            messages.warning(request, "กรุณายืนยัน 2FA ก่อนเข้าใช้งานหน้านี้")
            return redirect("accounts:2fa_setup")
        return view_func(request, *args, **kwargs)

    return _wrapped
```

```python
# blog/views.py
from accounts.decorators import staff_2fa_required


@staff_2fa_required
def internal_analytics_dashboard(request):
    ...
```

### 365.5 ตารางเปรียบเทียบ 3 แนวทาง

| แนวทาง | ขอบเขต | เหมาะกับ |
|---|---|---|
| `EnforceStaff2FAMiddleware` | ทั้งเว็บ (global) | บังคับ staff/admin ทุกคนไม่มีข้อยกเว้น (แนะนำสำหรับ production) |
| `@otp_required` (built-in) | รายมี view | ต้องการความยืดหยุ่นสูง มี `if_configured` ให้ rollout แบบค่อยเป็นค่อยไป |
| `@staff_2fa_required` (custom) | รายมี view | ต้องการรวม logic ตรวจสอบ staff + 2FA ไว้ในที่เดียว อ่านง่ายตรง view |

หลักสูตรนี้แนะนำให้ใช้ **Middleware เป็นแนวป้องกันหลัก** (ครอบคลุมทุกจุดโดยไม่มี
ใครลืมใส่ decorator) และใช้ decorator เสริมเฉพาะจุดที่ต้องการ logic พิเศษเพิ่มเติม

---

## ขั้นตอนที่ 366: ผสาน 2FA เข้ากับ Django Admin ด้วย django-otp

### 366.1 ปัญหาของ Middleware ในขั้นตอนที่ 365 เมื่อใช้กับ Admin

Middleware ที่เขียนไว้ครอบคลุม view ทั้งเว็บที่เราคุมโค้ดเองได้ แต่ **Django Admin
เป็น app สำเร็จรูปที่มี view ของตัวเองอยู่แล้ว** (`django.contrib.admin.sites`)
เราไม่ได้เขียน view เหล่านั้นเอง django-otp จึงเตรียมกลไกเฉพาะสำหรับ Admin ไว้ให้
โดยตรง เรียกว่า **`OTPAdminSite`**

### 366.2 `OTPAdminSite` คืออะไร

`django_otp.admin.OTPAdminSite` คือ `AdminSite` subclass ที่ override
`has_permission()` ให้ตรวจสอบเพิ่มอีกเงื่อนไขหนึ่งนอกจาก `is_active` และ
`is_staff` ตามปกติ: **ต้อง `request.user.is_verified()` เป็น `True` ด้วย** ผู้ใช้
ที่ login สำเร็จแต่ยังไม่ผ่าน 2FA จะ **เข้าหน้า Admin ไม่ได้เลย** แม้จะเป็น
superuser ก็ตาม

### 366.3 วิธีที่ 1: แก้ Class ของ `admin.site` ที่มีอยู่แล้ว (เร็วที่สุด)

```python
# config/admin.py หรือท้ายไฟล์ blog/admin.py (ที่ import ครั้งแรกตอน startup)
from django.contrib import admin
from django_otp.admin import OTPAdminSite

admin.site.__class__ = OTPAdminSite
```

วิธีนี้ **"สลับคลาส" ของ instance `admin.site`** ที่มีอยู่แล้วให้กลายเป็น
`OTPAdminSite` แบบ runtime โดยไม่ต้องแก้ `urls.py` หรือ re-register model ใด ๆ
เลย เป็นวิธีที่ django-otp official docs แนะนำสำหรับกรณีที่ยังใช้ `admin.site`
ตัวเดิม (ไม่มี custom `AdminSite` ของตัวเอง)

### 366.4 วิธีที่ 2: สร้าง Custom AdminSite ที่สืบทอดจาก `OTPAdminSite` (แนะนำถ้ามี custom AdminSite อยู่แล้ว)

ถ้าโปรเจกต์ของคุณสร้าง `AdminSite` ของตัวเองไว้แล้วตาม Part 018 (ขั้นตอนที่ 173.2)
ให้เปลี่ยน base class จาก `AdminSite` เป็น `OTPAdminSite` แทน:

```python
# config/admin.py
from django_otp.admin import OTPAdminSite


class BlogAdminSite(OTPAdminSite):   # เปลี่ยนจาก AdminSite เป็น OTPAdminSite
    site_header = "ระบบจัดการบล็อก MyCompany"
    site_title = "MyCompany Admin"
    index_title = "แผงควบคุมผู้ดูแลระบบ (ต้องผ่าน 2FA)"


blog_admin_site = BlogAdminSite(name="blog_admin")
```

โค้ดส่วน `register()` model ต่าง ๆ ที่มีอยู่แล้วไม่ต้องแก้ไขอะไรเพิ่มเติม เพราะ
`OTPAdminSite` ยังคง API เดิมของ `AdminSite` ทุกอย่าง เปลี่ยนแค่ตรรกะการตรวจสอบ
สิทธิ์เท่านั้น

### 366.5 ผลลัพธ์ที่ผู้ใช้จะเห็น

เมื่อ staff ที่ยังไม่เปิด 2FA (หรือเปิดแล้วแต่ยังไม่ยืนยันในเซสชันนี้) พยายามเข้า
`/admin/` Django Admin จะแสดงหน้า **"คุณไม่มีสิทธิ์เข้าถึง"** (permission denied)
แบบเดียวกับ user ที่ไม่ใช่ staff เลย — นี่คือพฤติกรรมที่ตั้งใจ เพื่อไม่เปิดเผย
ข้อมูลใด ๆ เกี่ยวกับสถานะ 2FA ของบัญชีให้ผู้โจมตีที่ขโมยรหัสผ่านมาได้รู้

**คำแนะนำ UX**: เพื่อไม่ให้ staff ที่ถูกบล็อกสับสนว่าเกิดอะไรขึ้น ให้ผสาน
`EnforceStaff2FAMiddleware` จากขั้นตอนที่ 365 เข้าไปด้วย เพราะ middleware นั้นจะ
ดักและ redirect ไปหน้าตั้งค่า 2FA **ก่อน** ที่ request จะไปถึง `OTPAdminSite`
ด้วยซ้ำ ทำให้ staff เห็นข้อความแจ้งเตือนที่ชัดเจนแทนหน้า permission denied เฉย ๆ

### 366.6 ลงทะเบียน Device Model เข้า Admin เพื่อให้ทีมงานจัดการได้

บางครั้งทีม support ต้องการดู/ลบ device ของผู้ใช้ที่ทำมือถือหายและ backup code
หมดด้วย (เพื่อบังคับให้ enroll ใหม่) ลงทะเบียน model เหล่านี้เข้า Admin แบบ
read-mostly:

```python
# accounts/admin.py
from django.contrib import admin
from django_otp.plugins.otp_totp.models import TOTPDevice
from django_otp.plugins.otp_static.models import StaticDevice


@admin.register(TOTPDevice)
class TOTPDeviceAdmin(admin.ModelAdmin):
    list_display = ["user", "name", "confirmed", "last_used_at" if hasattr(TOTPDevice, "last_used_at") else "id"]
    list_filter = ["confirmed"]
    search_fields = ["user__username", "user__email"]
    readonly_fields = ["key", "user"]   # ป้องกัน staff แก้ secret key ผ่าน Admin โดยไม่ตั้งใจ

    def has_add_permission(self, request):
        return False  # ต้องสร้างผ่าน flow enroll เท่านั้น ห้ามสร้างมั่ว ๆ จาก Admin


@admin.register(StaticDevice)
class StaticDeviceAdmin(admin.ModelAdmin):
    list_display = ["user", "name"]
    search_fields = ["user__username", "user__email"]

    def has_add_permission(self, request):
        return False
```

**เหตุผลที่ `has_add_permission` คืน `False` เสมอ**: การสร้าง `TOTPDevice` หรือ
`StaticDevice` ต้องผ่าน flow ที่ generate secret key และให้ผู้ใช้สแกน/ยืนยันจริง
เท่านั้น การให้ staff สร้าง device เปล่า ๆ ผ่านฟอร์ม Admin ตรง ๆ จะทำให้ device
นั้นไม่มี secret key ที่ตรงกับอุปกรณ์ของผู้ใช้จริงเลย — เปิดสิทธิ์ไว้แค่ **ดูและ
ลบ** เท่านั้น เหมาะกับ use case "ช่วยผู้ใช้ที่มือถือหายให้ enroll ใหม่ได้"

---

## ขั้นตอนที่ 367: SMS-based 2FA เทียบกับ TOTP — ต้นทุนและความเสี่ยง SIM Swap

### 367.1 SMS OTP ทำงานอย่างไร

ระบบสร้างรหัส OTP แบบสุ่ม (ไม่จำเป็นต้องเป็น TOTP algorithm ก็ได้ อาจเป็นแค่เลข
สุ่ม 6 หลักที่เก็บ state ไว้ในฐานข้อมูล/cache) แล้วส่งผ่าน **SMS Gateway** (เช่น
Twilio, AWS SNS, Amazon Pinpoint) ไปยังเบอร์โทรศัพท์ที่ผู้ใช้ลงทะเบียนไว้ ตัวอย่าง
โครงร่างการเชื่อมต่อ (แสดงเพื่อความเข้าใจภาพรวม ไม่ใช่ระบบที่ใช้งานจริงในหลักสูตร):

```python
# ตัวอย่างแนวคิด (ไม่ใช่โค้ด production) — การส่ง SMS ผ่าน Twilio
import random
from django.core.cache import cache


def send_sms_otp(phone_number):
    code = f"{random.randint(0, 999999):06d}"
    cache.set(f"sms_otp:{phone_number}", code, timeout=300)  # หมดอายุใน 5 นาที

    # ส่งผ่าน SMS Gateway (ตัวอย่างใช้ Twilio SDK)
    # from twilio.rest import Client
    # client = Client(TWILIO_SID, TWILIO_AUTH_TOKEN)
    # client.messages.create(body=f"รหัส OTP ของคุณคือ {code}",
    #                         from_=TWILIO_PHONE_NUMBER, to=phone_number)
    return code
```

### 367.2 ตารางเปรียบเทียบ TOTP กับ SMS OTP

| ประเด็น | TOTP (django-otp) | SMS OTP |
|---|---|---|
| **ต้นทุนต่อการยืนยัน** | ฟรี (คำนวณในเครื่อง) | มีค่าใช้จ่ายต่อข้อความ (ผ่าน SMS Gateway เช่น Twilio ~0.3-1 บาท/ข้อความ) |
| **ความเสี่ยง SIM Swap** | ไม่มี (ไม่พึ่งเบอร์โทรศัพท์เลย) | **สูง** — ผู้โจมตีสวมรอยขอย้ายเบอร์ไปซิมใหม่กับผู้ให้บริการมือถือได้ |
| **ต้องมีสัญญาณเครือข่าย** | ไม่ต้อง (ทำงาน offline) | ต้องมีสัญญาณมือถือ/SMS ถึงจะรับรหัสได้ |
| **ความเร็วในการรับรหัส** | ทันที (คำนวณในเครื่อง) | อาจล่าช้าหลายวินาทีถึงหลายนาที ขึ้นกับผู้ให้บริการ |
| **ความน่าเชื่อถือ (Reliability)** | สูงมาก | อาจส่งไม่ถึงถ้าสัญญาณไม่ดี, roaming ต่างประเทศ, เบอร์ผิด |
| **ความซับซ้อนในการพัฒนา** | ต้องมี Authenticator App ติดตั้งก่อน | ผู้ใช้ทั่วไปเข้าใจง่ายกว่า (คุ้นเคยกับ SMS อยู่แล้ว) |
| **มาตรฐานความปลอดภัยล่าสุด** | แนะนำโดย NIST, OWASP | **NIST SP 800-63B ไม่แนะนำให้ใช้เป็นปัจจัยหลักอีกต่อไป** เพราะความเสี่ยง SIM Swap |

### 367.3 SIM Swap Attack คืออะไร — ทำไมถึงอันตรายกับ SMS 2FA

**SIM Swap** คือการที่ผู้โจมตี **หลอกผู้ให้บริการเครือข่ายมือถือ** (เช่น โทรเข้า
Call Center อ้างว่าซิมหาย ขอย้ายเบอร์เดิมไปซิมใหม่ที่ตนถืออยู่) โดยใช้ข้อมูลส่วนตัว
ของเหยื่อที่หาได้จาก social engineering หรือข้อมูลรั่วจากที่อื่น เมื่อสำเร็จ:

```
1. ผู้โจมตีขอย้ายเบอร์ของเหยื่อไปซิมใหม่ที่ตนถือ (ผ่าน Call Center/ร้านตัวแทน)
2. ซิมเดิมของเหยื่อถูกตัดสัญญาณทันที (เหยื่อจะสังเกตว่ามือถือไม่มีสัญญาณกะทันหัน)
3. SMS OTP ทั้งหมดที่ควรส่งไปเบอร์เหยื่อ วิ่งไปเข้าซิมใหม่ของผู้โจมตีแทน
4. ผู้โจมตีใช้รหัสผ่านที่ขโมยมา (จาก phishing/breach) + SMS OTP ที่ได้รับ
   → เข้าระบบสำเร็จ ทั้งที่เหยื่อไม่รู้ตัวเลย
```

เหตุการณ์นี้เคยเกิดขึ้นจริงกับบัญชี Twitter ของผู้บริหารระดับสูงหลายราย และเป็น
เหตุผลหลักที่มาตรฐานความปลอดภัยสมัยใหม่ (NIST SP 800-63B) จัดให้ SMS เป็น
**"restricted authenticator"** ไม่แนะนำใช้เป็นปัจจัยเดียวของ 2FA อีกต่อไป

### 367.4 เมื่อไหร่ SMS OTP ยังพอมีที่ยืนอยู่บ้าง

- เป็น **fallback ปัจจัยที่ 3** เสริมจาก TOTP (ไม่ใช่ปัจจัยหลัก) สำหรับกรณีผู้ใช้
  ทำมือถือ Authenticator หายและ backup code หมดพร้อมกัน
- กลุ่มผู้ใช้ที่ไม่คุ้นเคยกับการติดตั้งแอป Authenticator เลย (ผู้สูงอายุ, ตลาด
  ที่การเข้าถึงสมาร์ทโฟนจำกัด) — ยังดีกว่าไม่มี 2FA เลย
- ใช้ควบคู่กับการยืนยันตัวตนแบบอื่นเพิ่มเติมสำหรับ action ที่มีความเสี่ยงสูงมาก
  (เช่น เปลี่ยนรหัสผ่าน, ถอนเงิน)

**คำแนะนำของหลักสูตรนี้**: ใช้ **TOTP เป็นค่าเริ่มต้นเสมอ** และถ้าจำเป็นต้องมี
SMS ให้เสนอเป็น "ตัวเลือกเสริม" ไม่ใช่ "ตัวเลือกเดียว" พร้อมแจ้งเตือนผู้ใช้ถึง
ความเสี่ยง SIM Swap อย่างชัดเจนในหน้าตั้งค่า

---

## ขั้นตอนที่ 368: WebAuthn/Passkeys — อนาคตของการยืนยันตัวตนแบบไม่ใช้รหัสผ่าน

### 368.1 ปัญหาที่ WebAuthn แก้ไข

ทั้ง TOTP และ SMS ยังคงต้องพึ่ง **รหัสผ่าน** เป็นปัจจัยแรกอยู่ดี ซึ่งยังเสี่ยงต่อ
phishing เสมอ (ผู้ใช้ยังคงพิมพ์รหัสผ่านลงเว็บปลอมได้อยู่ แม้จะมี 2FA ตามมา บาง
เทคนิค phishing ขั้นสูงอย่าง "real-time relay attack" ยังดัก TOTP token ที่ผู้ใช้
กรอกได้ทันทีด้วยซ้ำ) **WebAuthn** คือมาตรฐานเปิดจาก W3C/FIDO Alliance ที่ออกแบบมา
เพื่อกำจัดจุดอ่อนนี้โดยสิ้นเชิง

### 368.2 หลักการ: Public Key Cryptography แทนรหัสผ่าน

```
ตอนลงทะเบียน (Registration):
┌──────────────┐                                    ┌──────────────┐
│   อุปกรณ์     │  สร้างคู่กุญแจ (Key Pair) เฉพาะเว็บนี้  │   เซิร์ฟเวอร์   │
│ (มือถือ/      │  Private Key → เก็บในอุปกรณ์เท่านั้น   │              │
│  YubiKey)     │  Public Key  → ส่งให้เซิร์ฟเวอร์ ────> │  เก็บ Public  │
└──────────────┘                                    │  Key ไว้      │
                                                      └──────────────┘

ตอน Login (Authentication):
┌──────────────┐   เซิร์ฟเวอร์ส่ง "challenge" สุ่มมา      ┌──────────────┐
│   อุปกรณ์     │ <──────────────────────────────────── │   เซิร์ฟเวอร์   │
│  เซ็น challenge ด้วย Private Key (ยืนยันตัวตนด้วย       │              │
│  ลายนิ้วมือ/Face ID/PIN ของอุปกรณ์ก่อนเซ็นเสมอ)         │              │
│  ส่ง signature กลับ ─────────────────────────────>    │  ตรวจสอบด้วย  │
└──────────────┘                                       │  Public Key   │
                                                        └──────────────┘
```

จุดสำคัญคือ **Private Key ไม่เคยออกจากอุปกรณ์เลยแม้แต่ครั้งเดียว** และแต่ละเว็บ
มีคู่กุญแจแยกกันคนละชุด (bound ตาม origin/domain) ทำให้:

- **Phishing-resistant โดยธรรมชาติ**: ถ้าผู้ใช้ถูกหลอกไปกรอกข้อมูลที่เว็บปลอม
  (domain ต่างจากเว็บจริง) อุปกรณ์จะปฏิเสธเซ็น challenge ให้เอง เพราะกุญแจถูก
  ผูกกับ origin ที่ถูกต้องเท่านั้น
- **ไม่มีความลับ (secret) ใดที่ต้องพิมพ์หรือส่งผ่านเครือข่ายเลย** — ไม่มีอะไรให้
  ขโมยแม้จะดักฟังเครือข่ายได้ทั้งหมด

### 368.3 Passkeys คืออะไร (ต่างจาก WebAuthn ทั่วไปอย่างไร)

**Passkey** คือ WebAuthn credential ที่ถูก **sync ข้ามอุปกรณ์ผ่าน cloud** ของ
แพลตฟอร์ม (iCloud Keychain สำหรับ Apple, Google Password Manager สำหรับ Android/
Chrome, Windows Hello สำหรับ Microsoft) ทำให้ผู้ใช้ไม่ต้องลงทะเบียนอุปกรณ์ใหม่
ทุกเครื่องซ้ำ ๆ เหมือน Hardware Security Key แบบดั้งเดิม — นี่คือเหตุผลที่ตั้งแต่
ปี 2023 เป็นต้นมา บริษัทใหญ่ (Google, Apple, Microsoft, GitHub, PayPal) เริ่ม
ผลักดัน Passkeys ให้เป็นทางเลือกแทนรหัสผ่าน + 2FA แบบเดิมทั้งระบบ

### 368.4 ตารางเปรียบเทียบ TOTP, SMS, และ WebAuthn/Passkeys

| ประเด็น | TOTP | SMS OTP | WebAuthn/Passkeys |
|---|---|---|---|
| ยังต้องใช้รหัสผ่านคู่กันไหม | ต้อง | ต้อง | **ไม่ต้อง** (ใช้แทนรหัสผ่านได้เลย) |
| ป้องกัน Phishing ได้ไหม | ป้องกันบางส่วน (ยังมี relay attack) | ป้องกันไม่ได้ | **ป้องกันได้โดยธรรมชาติ** |
| ต้องพิมพ์รหัสเอง | ต้อง (6 หลัก) | ต้อง (6 หลัก) | **ไม่ต้อง** (สแกนนิ้ว/หน้า/กด PIN อุปกรณ์) |
| ความเสี่ยง SIM Swap | ไม่มี | มี | ไม่มี |
| ความซับซ้อนการ implement ฝั่ง server | ปานกลาง (`django-otp`) | ปานกลาง + ต้องมี SMS Gateway | สูงกว่า (ต้อง handle challenge/response, attestation) |
| Browser/Device support | ทุกที่ (ไม่พึ่ง browser API) | ทุกที่ | ต้องใช้ browser ที่รองรับ WebAuthn API (เบราว์เซอร์สมัยใหม่ทุกตัวรองรับแล้ว) |

### 368.5 แนวทางนำ WebAuthn มาใช้กับ Django

ในระบบนิเวศ Django มี package อย่าง `django-otp-webauthn` (ผสาน WebAuthn เข้ากับ
`django-otp` framework เดียวกับที่เรียนใน Part นี้) หรือไลบรารีระดับล่างอย่าง
`webauthn` (Python package ที่ implement WebAuthn spec ตรง ๆ ให้เราประกอบ view
เอง) โครงร่างแนวคิดของ view ฝั่งเซิร์ฟเวอร์ (แสดงเพื่อความเข้าใจ ไม่ใช่โค้ดที่รัน
ได้ทันทีโดยไม่ติดตั้ง dependency เพิ่มเติม):

```python
# ตัวอย่างแนวคิด (ไม่ใช่โค้ด production พร้อมใช้) — เริ่มกระบวนการลงทะเบียน WebAuthn
# pip install webauthn
from webauthn import generate_registration_options, options_to_json


def webauthn_register_begin(request):
    options = generate_registration_options(
        rp_id="myblog.com",              # domain ของเว็บเรา
        rp_name="MyCompany Blog",
        user_id=str(request.user.id).encode(),
        user_name=request.user.username,
    )
    request.session["webauthn_challenge"] = options.challenge
    return HttpResponse(options_to_json(options), content_type="application/json")

# ฝั่ง JavaScript จะเรียก navigator.credentials.create(options) เพื่อสั่งให้
# เบราว์เซอร์คุยกับอุปกรณ์ (Touch ID, Windows Hello, YubiKey) แล้วส่งผลลัพธ์
# กลับมาให้ view ปลายทาง verify_registration_response() ตรวจสอบต่อ
```

### 368.6 ทำไมหลักสูตรนี้ยังสอน TOTP เป็นหลักในตอนนี้

WebAuthn/Passkeys คือทิศทางที่อุตสาหกรรมกำลังมุ่งไป แต่ ณ ตอนนี้ (2025-2026)
ระบบส่วนใหญ่ยัง **ใช้ TOTP เป็นมาตรฐานหลักที่ implement ง่ายและครอบคลุมผู้ใช้ทุก
กลุ่มมากกว่า** (ไม่ต้องพึ่ง browser API เฉพาะ, ใช้ได้แม้ในสภาพแวดล้อมที่จำกัด)
หลักสูตรนี้จึงให้ TOTP เป็นแกนหลักที่ implement ได้จริงครบวงจร และเกริ่น WebAuthn
ไว้เป็นความรู้สำหรับต่อยอดเมื่อทีมของคุณพร้อมยกระดับในอนาคต — หลักการ Public
Key Cryptography ที่อยู่เบื้องหลัง WebAuthn จะเรียนละเอียดอีกครั้งเมื่อถึงเรื่อง
JWT และ OAuth2 ใน Part 046

---

## ขั้นตอนที่ 369: การเขียน Test สำหรับ Flow ที่มี 2FA

### 369.1 ความท้าทายของการ Test ระบบที่มี 2FA

Test ปกติที่ใช้ `self.client.login(username=..., password=...)` จะยังคง **ผ่าน
`is_verified()` เป็น `False`** เสมอ เพราะ `login()` ของ Django test client จำลอง
แค่การยืนยันรหัสผ่าน ไม่ได้จำลองการยืนยัน OTP ในเซสชัน ถ้า test เดิมของคุณทดสอบ
view ที่ตอนนี้ถูกป้องกันด้วย `staff_2fa_required` หรือ `EnforceStaff2FAMiddleware`
test เหล่านั้นจะเริ่ม **fail ทันที** เพราะถูก redirect ไปหน้า 2FA setup แทน

### 369.2 เทคนิคที่ 1: จำลองการยืนยัน 2FA ผ่าน Session โดยตรง

django-otp เก็บ device ที่ผ่านการยืนยันไว้ใน session ภายใต้ key คงที่
`django_otp.DEVICE_ID_SESSION_KEY` เราจึงตั้งค่านี้ตรง ๆ ใน test ได้เลยโดยไม่ต้อง
จำลองการกรอกรหัส 6 หลักจริง:

```python
# accounts/tests/test_2fa.py
from django.test import TestCase
from django.urls import reverse
from django.contrib.auth import get_user_model
from django_otp import DEVICE_ID_SESSION_KEY
from django_otp.plugins.otp_totp.models import TOTPDevice

User = get_user_model()


class TwoFactorLoginRequiredMixin:
    """Mixin สำหรับ TestCase ที่ต้องการ user ที่ผ่าน 2FA แล้ว"""

    def login_with_2fa(self, user, password="testpass123"):
        self.client.login(username=user.username, password=password)
        device = TOTPDevice.objects.create(user=user, name="test-device", confirmed=True)
        session = self.client.session
        session[DEVICE_ID_SESSION_KEY] = device.persistent_id
        session.save()
        return device


class StaffDashboardAccessTests(TwoFactorLoginRequiredMixin, TestCase):
    def setUp(self):
        self.staff_user = User.objects.create_user(
            username="staff_test",
            email="staff@example.com",
            password="testpass123",
            is_staff=True,
        )

    def test_staff_without_2fa_is_redirected_to_setup(self):
        """staff ที่ยังไม่ผ่าน 2FA ต้องถูกเด้งไปหน้าตั้งค่า 2FA เสมอ"""
        self.client.login(username="staff_test", password="testpass123")
        response = self.client.get(reverse("blog:internal_dashboard"))
        self.assertRedirects(response, reverse("accounts:2fa_setup"))

    def test_staff_with_verified_2fa_can_access_dashboard(self):
        """staff ที่ผ่าน 2FA แล้วต้องเข้าหน้าที่ป้องกันไว้ได้ปกติ"""
        self.login_with_2fa(self.staff_user)
        response = self.client.get(reverse("blog:internal_dashboard"))
        self.assertEqual(response.status_code, 200)
```

### 369.3 เทคนิคที่ 2: ทดสอบ Flow การ Enroll TOTP แบบเต็ม (จำลองการคำนวณรหัสจริง)

เพื่อทดสอบ view `totp_setup` ให้ครอบคลุมของจริง เราต้องคำนวณรหัส TOTP ที่ถูกต้อง
ณ เวลาปัจจุบันด้วยตัวเอง แทนที่จะเดามั่ว ๆ — django-otp เปิด method
`device.verify_token()` แต่เราต้องมีค่า token ที่ถูกต้องส่งเข้าไปก่อน วิธีที่
สะดวกที่สุดคือใช้ method ภายในของ `TOTPDevice` เอง (`generate_token()` มีให้ใน
บาง environment ทดสอบ) หรือคำนวณตรงด้วยไลบรารีมาตรฐาน:

```python
# accounts/tests/test_2fa.py (เพิ่มต่อ)
import base64
import hmac
import hashlib
import struct
import time


def calculate_totp(secret_hex, digits=6, step=30):
    """คำนวณรหัส TOTP ปัจจุบันจาก secret key (hex) — จำลองสิ่งที่แอป Authenticator ทำ"""
    key = bytes.fromhex(secret_hex)
    counter = int(time.time() // step)
    counter_bytes = struct.pack(">Q", counter)
    digest = hmac.new(key, counter_bytes, hashlib.sha1).digest()
    offset = digest[-1] & 0x0F
    truncated = struct.unpack(">I", digest[offset:offset + 4])[0] & 0x7FFFFFFF
    return str(truncated % (10 ** digits)).zfill(digits)


class TOTPEnrollmentFlowTests(TestCase):
    def setUp(self):
        self.user = User.objects.create_user(
            username="new_staff", email="new@example.com",
            password="testpass123", is_staff=True,
        )
        self.client.login(username="new_staff", password="testpass123")

    def test_enrollment_with_correct_code_confirms_device(self):
        # เข้าหน้า setup ครั้งแรก → สร้าง device ที่ confirmed=False อัตโนมัติ
        self.client.get(reverse("accounts:2fa_setup"))
        device = TOTPDevice.objects.get(user=self.user, confirmed=False)

        correct_token = calculate_totp(device.key)
        response = self.client.post(reverse("accounts:2fa_setup"), {"token": correct_token})

        device.refresh_from_db()
        self.assertTrue(device.confirmed)
        self.assertRedirects(response, reverse("accounts:2fa_backup_codes"))

    def test_enrollment_with_wrong_code_stays_unconfirmed(self):
        self.client.get(reverse("accounts:2fa_setup"))
        response = self.client.post(reverse("accounts:2fa_setup"), {"token": "000000"})

        device = TOTPDevice.objects.get(user=self.user)
        self.assertFalse(device.confirmed)
        self.assertEqual(response.status_code, 200)  # แสดงฟอร์มเดิมพร้อม error message
```

### 369.4 ทดสอบ Backup Codes: สร้าง, ใช้งาน, และการ Consume

```python
# accounts/tests/test_2fa.py (เพิ่มต่อ)
from django_otp.plugins.otp_static.models import StaticDevice, StaticToken
from accounts.views_2fa import verify_2fa_code


class BackupCodeTests(TestCase):
    def setUp(self):
        self.user = User.objects.create_user(
            username="backup_test", email="backup@example.com", password="testpass123",
        )
        self.device = StaticDevice.objects.create(user=self.user, name="backup")
        self.token = StaticToken.objects.create(device=self.device, token="12345678")

    def test_backup_code_works_once(self):
        request = self.client.request().wsgi_request  # จำลอง request object แบบง่าย
        request.user = self.user

        matched = verify_2fa_code(request, self.user, "12345678")
        self.assertTrue(matched)

        # รหัสเดิมต้องถูกลบออกไปแล้ว ใช้ซ้ำไม่ได้อีก
        self.assertFalse(StaticToken.objects.filter(pk=self.token.pk).exists())

    def test_wrong_backup_code_fails(self):
        request = self.client.request().wsgi_request
        request.user = self.user

        matched = verify_2fa_code(request, self.user, "99999999")
        self.assertFalse(matched)
        # รหัสที่ถูกต้องยังคงอยู่ ไม่ถูกลบ เพราะไม่ตรงกัน
        self.assertTrue(StaticToken.objects.filter(pk=self.token.pk).exists())
```

### 369.5 ปิด 2FA ทั้งระบบชั่วคราวสำหรับ Test Suite ที่ไม่เกี่ยวข้องกับ 2FA โดยตรง

สำหรับ test ทั้งชุดที่ไม่ได้ทดสอบเรื่อง 2FA โดยตรง (เช่น test ของ `blog` app ที่มี
มาก่อน Part นี้) การต้องเพิ่ม `login_with_2fa()` ทุกที่จะรบกวนมาก แนวทางที่สะอาด
กว่าคือ **override settings เพื่อปิด middleware บังคับ 2FA เฉพาะตอนรัน test
กลุ่มนั้น**:

```python
# blog/tests/test_views.py
from django.test import TestCase, override_settings


@override_settings(
    MIDDLEWARE=[m for m in __import__("django.conf").conf.settings.MIDDLEWARE
                if m != "accounts.middleware.EnforceStaff2FAMiddleware"]
)
class BlogViewsWithout2FAEnforcementTests(TestCase):
    """Test เดิมของ blog app ที่ไม่เกี่ยวกับ 2FA — ปิด middleware บังคับ 2FA ชั่วคราว"""
    def test_post_list_view_still_works(self):
        response = self.client.get("/")
        self.assertEqual(response.status_code, 200)
```

**หลักการที่ควรยึดถือ**: ใช้ `override_settings` ปิด middleware เฉพาะใน test ที่
**ไม่ได้ตั้งใจทดสอบพฤติกรรม 2FA** เท่านั้น ส่วน test ที่ทดสอบ security flow ของ
2FA โดยตรง (อย่างในหัวข้อ 369.2-369.4) ต้องปล่อยให้ middleware ทำงานตามจริงเสมอ
เพื่อให้ test นั้นมีความหมายในการป้องกันการ regression ของฟีเจอร์ความปลอดภัย

---

## ขั้นตอนที่ 370: สรุปและแบบฝึกหัด — เปิดใช้ 2FA จริงสำหรับ staff ของบล็อก

### 370.1 สรุปสิ่งที่ได้เรียนรู้ใน Part นี้

- ✅ เข้าใจแนวคิด Multi-Factor Authentication และปัจจัยทั้ง 3 ประเภท
- ✅ เข้าใจอัลกอริทึม TOTP (RFC 6238) ตั้งแต่ HMAC-SHA1 ไปจนถึง Dynamic Truncation
- ✅ ติดตั้งและตั้งค่า `django-otp` พร้อม `OTPMiddleware` ครบถ้วน
- ✅ สร้าง Flow เปิดใช้ 2FA เต็มรูปแบบ: สร้าง `TOTPDevice`, generate provisioning
  URI, render QR Code เป็น PNG, ยืนยันด้วย `verify_token()`
- ✅ สร้างและจัดการ Backup Codes ด้วย `StaticDevice`/`StaticToken` พร้อมกลไก
  consume แบบใช้ครั้งเดียว
- ✅ เขียน Middleware และ Decorator บังคับ staff/admin ให้ต้องเปิด 2FA
- ✅ ผสาน 2FA เข้ากับ Django Admin ด้วย `OTPAdminSite`
- ✅ เข้าใจความเสี่ยง SIM Swap ของ SMS 2FA เทียบกับความปลอดภัยของ TOTP
- ✅ รู้จัก WebAuthn/Passkeys ในฐานะอนาคตของการยืนยันตัวตนแบบ phishing-resistant
- ✅ เขียน Test ครอบคลุม flow การ enroll, การ login ด้วย 2FA, และ backup codes

### 370.2 Checklist ก่อนไป Part ถัดไป

- [ ] ติดตั้ง `django-otp` และ `qrcode` แล้ว บันทึกใน `requirements.txt`
- [ ] เพิ่ม `django_otp`, `otp_totp`, `otp_static` ใน `INSTALLED_APPS`
- [ ] เพิ่ม `OTPMiddleware` ต่อจาก `AuthenticationMiddleware` และรัน `migrate` แล้ว
- [ ] สร้างหน้า `/accounts/2fa/setup/` ที่แสดง QR Code และยืนยันรหัส TOTP ได้จริง
- [ ] ทดสอบสแกน QR Code ด้วย Google Authenticator หรือ Authy บนมือถือจริงสำเร็จ
- [ ] สร้างหน้า Backup Codes ที่ generate/regenerate ชุดรหัสสำรองได้
- [ ] เพิ่ม `EnforceStaff2FAMiddleware` และยืนยันว่า staff ที่ยังไม่เปิด 2FA ถูก
      เด้งไปหน้า setup จริง
- [ ] เปลี่ยน `admin.site` ให้เป็น `OTPAdminSite` และยืนยันว่า staff ที่ไม่ผ่าน
      2FA เข้า `/admin/` ไม่ได้
- [ ] รัน test suite ทั้งหมดของ Part นี้ผ่านสำเร็จด้วย `python manage.py test accounts`

### 370.3 แบบฝึกหัดท้ายบท

**แบบฝึกหัดที่ 1**: เปิดใช้ 2FA จริงสำหรับบัญชี superuser ของโปรเจกต์บล็อกของคุณ
เอง สแกน QR Code ด้วยแอป Google Authenticator หรือ Authy บนมือถือจริง บันทึก
Backup Codes เก็บไว้ในที่ปลอดภัย แล้วลอง logout/login ใหม่เพื่อยืนยันว่าระบบถาม
รหัส TOTP จริง

**แบบฝึกหัดที่ 2**: เพิ่มปุ่ม "ปิดใช้ 2FA" (disable) ในหน้าตั้งค่าบัญชีผู้ใช้ ที่
ลบทั้ง `TOTPDevice` และ `StaticDevice` ของ user คนนั้น แต่ต้อง **บังคับให้กรอก
รหัสผ่านซ้ำอีกครั้ง** ก่อนอนุญาตให้ปิด (re-authentication) เพื่อป้องกันคนอื่นที่
แอบใช้เครื่องที่ล็อกอินค้างไว้มาปิด 2FA แทนเจ้าของบัญชี

**แบบฝึกหัดที่ 3**: เขียน management command ชื่อ `report_staff_without_2fa` ที่
พิมพ์รายชื่อ staff/superuser ทุกคนที่ **ยังไม่มี** `TOTPDevice` ที่ confirmed
ออกมาเป็นตาราง เพื่อให้ทีม security เอาไปติดตามและแจ้งเตือนได้ (ใบ้: ใช้ความรู้
management command จาก Part 016 ผสานกับ `TOTPDevice.objects.filter(...)`)

**แบบฝึกหัดที่ 4 (ขั้นสูง)**: ทดลองอ่านเอกสารของไลบรารี `webauthn` (Python) หรือ
`django-otp-webauthn` แล้วเขียนแผนผัง (diagram หรือ pseudo-code) ของ flow การ
ลงทะเบียน Passkey สำหรับระบบบล็อกของคุณ ระบุว่าต้องเก็บข้อมูลอะไรเพิ่มในฐานข้อมูล
(credential ID, public key, sign counter) โดยยังไม่ต้อง implement จริงก็ได้
เป้าหมายคือฝึกอ่านสเปกและออกแบบ data model ก่อนลงมือเขียนโค้ด

### 370.4 คำถามที่พบบ่อย (FAQ)

**Q: ถ้าผู้ใช้ทำมือถือหายและ Backup Codes ก็หมดพร้อมกัน ต้องทำอย่างไร?**
A: ต้องมี "Manual Recovery Process" ที่ทีม support ตรวจสอบตัวตนผู้ใช้ด้วยวิธีอื่น
(เช่น ยืนยันเอกสารตัวตน, ตอบคำถามความปลอดภัยที่ตั้งไว้ล่วงหน้า) แล้วให้ staff ที่
มีสิทธิ์ลบ `TOTPDevice`/`StaticDevice` ของผู้ใช้คนนั้นผ่าน Admin (ตามที่ตั้งค่าไว้
ในขั้นตอนที่ 366.6) เพื่อให้ผู้ใช้ enroll device ใหม่ได้ กระบวนการนี้ควรมีการ
log และ audit อย่างเข้มงวด เพราะเป็นจุดที่ผู้โจมตีอาจพยายามใช้ social engineering
หลอกทีม support แทน

**Q: ทำไมรหัส TOTP ที่กรอกถูกต้องแต่ระบบบอกว่าผิด?**
A: สาเหตุอันดับหนึ่งคือ **นาฬิกาของเซิร์ฟเวอร์หรือมือถือคลาดเคลื่อน** (clock
drift) ตรวจสอบว่าเซิร์ฟเวอร์ sync เวลาผ่าน NTP อยู่เสมอ (`timedatectl status`
บน Linux) และแนะนำให้ผู้ใช้เปิด "ตั้งเวลาอัตโนมัติ" บนมือถือ นอกจากนี้ django-otp
มี parameter `tolerance` ใน `TOTPDevice` ที่ยอมรับ time step ก่อนหน้า/ถัดไปได้
เล็กน้อยเพื่อรองรับ clock drift ปกติ (ค่า default ยอมรับ ±1 step)

**Q: ควรบังคับ 2FA กับผู้ใช้ทั่วไป (ไม่ใช่ staff) ด้วยหรือไม่?**
A: ขึ้นอยู่กับความเสี่ยงของระบบ สำหรับบล็อกทั่วไปอาจ **แนะนำ** (optional) ให้
ผู้ใช้ทั่วไปเปิดเองได้ แต่ไม่บังคับ เพราะเพิ่ม friction ในการ signup/login แต่
สำหรับระบบที่มีข้อมูลการเงินหรือข้อมูลอ่อนไหว (เช่น e-commerce ที่มีข้อมูลบัตร
เครดิต) ควรพิจารณาบังคับสำหรับทุกบัญชีเลย

**Q: `django-otp` กับ `django-two-factor-auth` ต่างกันอย่างไร ควรเลือกอันไหน?**
A: `django-two-factor-auth` สร้างอยู่ **บนฐานของ `django-otp` อีกที** โดยมาพร้อม
view, template, และ wizard flow สำเร็จรูปทั้งหมด (login แบบ 2 ขั้นตอนในตัว) เหมาะ
กับทีมที่ต้องการความเร็วในการ implement โดยไม่ต้องเขียน view เอง ส่วน `django-otp`
เปล่า ๆ (ที่ Part นี้สอน) ให้ความยืดหยุ่นสูงกว่ามากในการออกแบบ UX ของตัวเอง เหมาะ
กับทีมที่มี design system หรือ flow เฉพาะทางอยู่แล้ว ไม่อยากถูกจำกัดด้วย template
สำเร็จรูป — ทั้งสองใช้ device model ชุดเดียวกัน จึงสลับไปมาได้ในภายหลังโดยไม่ต้อง
ย้ายข้อมูลผู้ใช้ใหม่

---

## เตรียมตัวสำหรับ Part ถัดไป

**Part 038: Row-Level Permissions และ Object-Level Permission** จะพาไปแก้ปัญหาที่
ระบบ permission มาตรฐานของ Django (ที่เรียนใน Part 033) ทำไม่ได้: การกำหนดสิทธิ์
**ราย object** เช่น "ผู้เขียนคนนี้แก้ไขได้เฉพาะโพสต์ของตัวเอง" หรือ "บรรณาธิการ
ทีม A เห็นเฉพาะโพสต์ของทีม A เท่านั้น" คุณจะได้รู้จัก django-guardian, เปรียบเทียบ
กับการเขียน permission logic เองแบบ custom, และเชื่อมโยงกับ 2FA middleware ที่
เพิ่งสร้างใน Part นี้ เพื่อออกแบบระบบสิทธิ์ที่ละเอียดและปลอดภัยระดับ production
อย่างสมบูรณ์

เตรียมทบทวน Groups และ Permissions จาก Part 033 ให้แม่นก่อน แล้วไปต่อกันเลย!
