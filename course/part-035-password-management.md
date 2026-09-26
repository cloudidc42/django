# Part 035: Password Management, Reset และ Security

> **ขั้นตอนที่ 341-350 ของหลักสูตร** | Phase 4: Authentication, Users และ Permissions
>
> เป้าหมายของ Part นี้: เจาะลึกทุกแง่มุมของการจัดการรหัสผ่านใน Django อย่างมืออาชีพ
> ตั้งแต่กลไก hashing แบบ PBKDF2 ที่ Django ใช้เป็นค่าเริ่มต้นและทางเลือกอย่าง Argon2,
> `AUTH_PASSWORD_VALIDATORS` ทั้ง 4 ตัวที่มากับ Django, การเขียน Custom Validator ของ
> ตัวเอง, ระบบเปลี่ยนรหัสผ่านตอน login อยู่ด้วย `PasswordChangeView`, Password Reset
> Flow เต็มรูปแบบตั้งแต่กรอกอีเมลจนถึงตั้งรหัสผ่านใหม่สำเร็จพร้อม template ที่ต้องมีครบ
> ทุกไฟล์, การทำ custom email ทั้งแบบ HTML และ plain text, การใช้ `set_password()`/
> `check_password()` แบบ manual ในสถานการณ์ที่ไม่ผ่านฟอร์ม, ไปจนถึงการเกริ่นเทคนิค
> ระดับโลกอย่าง Password Strength Meter ด้วย zxcvbn, การป้องกัน Brute Force ด้วย
> django-axes, และการเช็ครหัสผ่านที่เคยรั่วไหลผ่าน Have I Been Pwned API เมื่อจบ Part
> นี้ คุณจะสามารถสร้างระบบจัดการรหัสผ่านที่ปลอดภัยระดับ production ให้กับโปรเจกต์ `blog`
> ได้อย่างสมบูรณ์

---

## สารบัญของ Part นี้

- ขั้นตอนที่ 341: การ Hash รหัสผ่านของ Django — PBKDF2, Password Hashers และ `AUTH_PASSWORD_VALIDATORS` เจาะลึกทุกตัว
- ขั้นตอนที่ 342: เขียน Custom Password Validator เอง
- ขั้นตอนที่ 343: `PasswordChangeView`/`PasswordChangeForm` — เปลี่ยนรหัสผ่านตอน Login อยู่
- ขั้นตอนที่ 344: Password Reset Flow เต็มรูปแบบด้วย Built-in Views
- ขั้นตอนที่ 345: Custom Email Template สำหรับ Password Reset (HTML + Plain Text)
- ขั้นตอนที่ 346: `set_password()`/`check_password()` แบบ Manual
- ขั้นตอนที่ 347: Password Strength Meter ฝั่ง Client ด้วย zxcvbn
- ขั้นตอนที่ 348: Rate Limiting การพยายาม Login ด้วย django-axes (เกริ่นนำ)
- ขั้นตอนที่ 349: ตรวจสอบรหัสผ่านที่เคยรั่วไหลด้วย Have I Been Pwned API
- ขั้นตอนที่ 350: สรุปและแบบฝึกหัด — Password Reset Flow เต็มรูปแบบพร้อม Custom Validator

---

## ขั้นตอนที่ 341: การ Hash รหัสผ่านของ Django — PBKDF2, Password Hashers และ `AUTH_PASSWORD_VALIDATORS` เจาะลึกทุกตัว

### 341.1 ทบทวน: ทำไมห้ามเก็บรหัสผ่านแบบ Plaintext เด็ดขาด

ใน Part 031 ขั้นตอนที่ 301.2 คุณเคยเห็นแล้วว่า field `password` ของ `User` เก็บ
**hash** ไม่ใช่ plaintext แต่ Part นี้จะอธิบายว่า "ทำไม" และ "อย่างไร" อย่างละเอียด

ถ้าฐานข้อมูลของระบบถูกขโมย (data breach) — ซึ่งเกิดขึ้นจริงกับบริษัทใหญ่ ๆ นับไม่ถ้วน
— ผลกระทบจะต่างกันมหาศาลขึ้นอยู่กับว่าเก็บรหัสผ่านแบบไหน:

| วิธีเก็บรหัสผ่าน | ถ้าฐานข้อมูลรั่วไหล | ความปลอดภัย |
|---|---|---|
| Plaintext (`mypassword123`) | ผู้โจมตี login เข้าบัญชีผู้ใช้ได้ทันที ทุกบัญชี | ❌ อันตรายที่สุด ห้ามทำเด็ดขาด |
| Hash แบบเร็ว ไม่มี salt (เช่น `MD5(password)`) | ผู้โจมตีใช้ Rainbow Table หรือ GPU brute-force ถอดรหัสได้เร็วมาก (พัน-ล้านครั้ง/วินาที) | ❌ ไม่ปลอดภัยพอในปี 2026 |
| Hash + Salt แบบเร็ว (เช่น `SHA256(password + salt)`) | Rainbow Table ใช้ไม่ได้แล้ว แต่ brute-force ทีละบัญชียังเร็วเกินไป | ⚠️ ดีขึ้นแต่ยังไม่พอ |
| **Hash แบบช้าโดยตั้งใจ + Salt (PBKDF2, Argon2, bcrypt, scrypt)** | brute-force ต้องใช้เวลานานมาก เพราะแต่ละครั้งที่ลองรหัสผ่าน 1 ตัวใช้เวลาคำนวณหลายมิลลิวินาที | ✅ มาตรฐานที่ Django ใช้ |

หัวใจสำคัญคือ **การ hash รหัสผ่านต้อง "ช้าโดยตั้งใจ"** (deliberately slow) เพื่อทำให้
การลองรหัสผ่านทีละตัวแบบ brute-force ใช้เวลานานเกินคุ้มค่าสำหรับผู้โจมตี ต่างจาก
hash function ทั่วไปอย่าง MD5/SHA256 ที่ถูกออกแบบมาให้ **เร็วที่สุด** (เหมาะกับ
checksum ไฟล์ แต่ไม่เหมาะกับรหัสผ่านเลย)

### 341.2 โครงสร้างของค่าที่เก็บในฟิลด์ `password`

ลองดูค่าจริงที่ Django เก็บผ่าน shell:

```python
python manage.py shell
```

```python
from django.contrib.auth import get_user_model

User = get_user_model()
user = User.objects.first()
print(user.password)
```

ผลลัพธ์ตัวอย่าง:

```
pbkdf2_sha256$600000$xK8mPqR2vN4tYbZc$K3f5j8vQw9L2sT7pR4nM6xC1zA0bV3eH8dW5uY2iF9k=
```

ค่านี้แบ่งเป็น 4 ส่วนคั่นด้วย `$`:

| ส่วน | ตัวอย่าง | ความหมาย |
|---|---|---|
| Algorithm | `pbkdf2_sha256` | อัลกอริทึม hash ที่ใช้ (PBKDF2 ร่วมกับ SHA256) |
| Iterations | `600000` | จำนวนรอบที่ทำ hash ซ้ำ (ยิ่งมาก ยิ่งช้า ยิ่งปลอดภัยจาก brute-force) |
| Salt | `xK8mPqR2vN4tYbZc` | ค่าสุ่มเฉพาะของ user คนนี้ ป้องกัน Rainbow Table Attack |
| Hash | `K3f5j8vQw9...` | ผลลัพธ์สุดท้ายของการ hash `password + salt` ซ้ำตามจำนวน iterations |

**Salt** สำคัญมาก เพราะทำให้ผู้ใช้สองคนที่ตั้งรหัสผ่านเหมือนกันทุกตัวอักษร
(เช่น ทั้งคู่ใช้ `password123`) จะได้ค่า hash ที่**แตกต่างกันโดยสิ้นเชิง** เพราะ salt
ถูกสุ่มใหม่ทุกครั้งที่เรียก `set_password()` — นี่คือเหตุผลที่ผู้โจมตีไม่สามารถสร้าง
ตารางสำเร็จรูป (Rainbow Table) มาเทียบกับ hash ทั้งฐานข้อมูลได้ในคราวเดียว
ต้อง brute-force ทีละบัญชีเท่านั้น

### 341.3 PBKDF2: อัลกอริทึมเริ่มต้นของ Django และทำไมถึงเลือกมัน

**PBKDF2 (Password-Based Key Derivation Function 2)** คือ hash function ที่ออกแบบ
มาเฉพาะสำหรับรหัสผ่าน โดยมีคุณสมบัติหลักคือ **ทำ hash ซ้ำหลายรอบโดยตั้งใจ**
(configurable iteration count) เพื่อควบคุมความช้าได้ตามความเร็วของฮาร์ดแวร์ปัจจุบัน

```python
# ตรวจสอบจำนวน iterations ที่ Django เวอร์ชันของคุณใช้เป็นค่าเริ่มต้น
python -c "from django.contrib.auth.hashers import PBKDF2PasswordHasher; print(PBKDF2PasswordHasher().iterations)"
```

Django **เพิ่มจำนวน iterations ทุกรุ่นใหญ่** เพื่อตามความเร็วของ CPU/GPU ที่เร็วขึ้น
เรื่อย ๆ (ตัวเลขในตัวอย่างข้างต้น `600000` เป็นแค่ตัวอย่างเชิงแนวคิด — เวอร์ชันจริง
ของคุณอาจสูงกว่านี้) นี่คือเหตุผลที่ Django แนะนำให้อัปเกรดเวอร์ชันสม่ำเสมอ แม้แต่ใน
เรื่อง security parameter ที่มองไม่เห็นแบบนี้ก็ตาม

PBKDF2 ถูกเลือกเป็นค่าเริ่มต้นเพราะ:

- อยู่ใน Python standard library (`hashlib.pbkdf2_hmac`) โดยไม่ต้องพึ่ง C extension
  เพิ่มเติม ทำให้ใช้งานได้ทันทีบนทุกแพลตฟอร์มโดยไม่ต้อง compile อะไรเพิ่ม
- ผ่านการรับรองจาก NIST (สถาบันมาตรฐานสหรัฐอเมริกา)
- ปรับความช้า (cost factor) ได้ง่ายผ่านตัวเลข iterations เดียว

### 341.4 `PASSWORD_HASHERS`: รายชื่อ Hasher ทั้งหมดและลำดับความสำคัญ

Django ไม่ได้ผูกติดกับ PBKDF2 ตายตัว แต่มีระบบ **Hasher แบบ pluggable** ที่กำหนดผ่าน
setting `PASSWORD_HASHERS` ค่าเริ่มต้นของ Django คือ:

```python
# ค่าเริ่มต้นของ Django (ไม่ต้องเขียนเองถ้าไม่ต้องการเปลี่ยน)
PASSWORD_HASHERS = [
    "django.contrib.auth.hashers.PBKDF2PasswordHasher",
    "django.contrib.auth.hashers.PBKDF2SHA1PasswordHasher",
    "django.contrib.auth.hashers.Argon2PasswordHasher",
    "django.contrib.auth.hashers.BCryptSHA256PasswordHasher",
    "django.contrib.auth.hashers.ScryptPasswordHasher",
]
```

**กฎสำคัญ**: **ตัวแรกในรายการคือตัวที่ใช้ hash รหัสผ่านใหม่ทุกครั้ง** ส่วนตัวที่เหลือ
ถูกเก็บไว้เพื่อให้ Django ยังตรวจสอบ (`check_password`) รหัสผ่านเก่าที่เคย hash ด้วย
อัลกอริทึมอื่นได้ (เช่น ถ้าเคยใช้ bcrypt มาก่อนแล้วเปลี่ยน hasher ตัวแรก ผู้ใช้เก่าที่
ยังไม่เคย login ซ้ำก็ยัง login ได้ตามปกติ)

| Hasher | อัลกอริทึมพื้นฐาน | ต้องติดตั้ง Library เพิ่มหรือไม่ | ระดับความปลอดภัย |
|---|---|---|---|
| `PBKDF2PasswordHasher` | PBKDF2 + SHA256 | ❌ ไม่ต้อง (มากับ Python) | ดี (ค่าเริ่มต้น) |
| `PBKDF2SHA1PasswordHasher` | PBKDF2 + SHA1 | ❌ ไม่ต้อง | ดี (เก็บไว้เพื่อ backward compatibility) |
| `Argon2PasswordHasher` | Argon2id | ✅ ต้อง `pip install argon2-cffi` | **ดีที่สุดในปี 2026** (ผู้ชนะ Password Hashing Competition ปี 2015) |
| `BCryptSHA256PasswordHasher` | bcrypt | ✅ ต้อง `pip install bcrypt` | ดีมาก (มาตรฐานอุตสาหกรรมมานาน) |
| `ScryptPasswordHasher` | scrypt | ❌ ไม่ต้อง (ใช้ `hashlib.scrypt`) | ดีมาก (ทนทานต่อการโจมตีด้วย GPU/ASIC เพราะกินหน่วยความจำเยอะ) |

> **หมายเหตุ**: มี `MD5PasswordHasher` และ `UnsaltedMD5PasswordHasher` อยู่ในโค้ดของ
> Django ด้วย แต่มีไว้สำหรับ **การรัน test suite ให้เร็วขึ้นเท่านั้น** (MD5 เร็วมาก
> จึงทำให้ test รันไวขึ้น) **ห้ามใช้ใน production เด็ดขาด**

### 341.5 เปลี่ยนไปใช้ Argon2 (แนะนำสำหรับระบบใหม่ในปี 2026)

Argon2 ชนะการแข่งขัน **Password Hashing Competition** ปี 2015 และถูกแนะนำโดย OWASP
ให้เป็นตัวเลือกอันดับ 1 สำหรับระบบใหม่ เพราะทนทานทั้งต่อการโจมตีด้วย CPU และ GPU/ASIC
(ใช้ทั้งเวลาและหน่วยความจำในการคำนวณ ทำให้ผู้โจมตีขยาย scale การ brute-force ได้ยากกว่า
PBKDF2 ที่ใช้แค่เวลาอย่างเดียว)

```bash
pip install argon2-cffi
pip freeze > requirements.txt
```

```python
# config/settings.py
PASSWORD_HASHERS = [
    "django.contrib.auth.hashers.Argon2PasswordHasher",   # ย้ายมาเป็นตัวแรก
    "django.contrib.auth.hashers.PBKDF2PasswordHasher",   # เก็บไว้เผื่อ user เก่า
    "django.contrib.auth.hashers.PBKDF2SHA1PasswordHasher",
    "django.contrib.auth.hashers.BCryptSHA256PasswordHasher",
    "django.contrib.auth.hashers.ScryptPasswordHasher",
]
```

### 341.6 การอัปเกรด Hash อัตโนมัติแบบเงียบ ๆ (Transparent Rehashing)

จุดที่สวยงามที่สุดของระบบนี้คือ **คุณไม่จำเป็นต้องบังคับให้ผู้ใช้เปลี่ยนรหัสผ่านเอง**
เมื่อเปลี่ยน hasher ตัวแรก — Django จะอัปเกรด hash ให้อัตโนมัติทันทีที่ผู้ใช้คนนั้น
login สำเร็จครั้งถัดไป โค้ดจริงของ Django (ย่อเพื่อความเข้าใจ) แสดงกลไกนี้:

```python
# django/contrib/auth/base_user.py (โค้ดจริงย่อเพื่อความเข้าใจ)
class AbstractBaseUser(models.Model):
    def check_password(self, raw_password):
        def setter(raw_password):
            self.set_password(raw_password)
            # การอัปเกรด hash ไม่นับเป็นการ "เปลี่ยนรหัสผ่าน" ของผู้ใช้
            self._password = None
            self.save(update_fields=["password"])
        return check_password(raw_password, self.password, setter)
```

```python
# django/contrib/auth/hashers.py (โค้ดจริงย่อเพื่อความเข้าใจ)
def check_password(password, encoded, setter=None, preferred="default"):
    preferred = get_hasher(preferred)
    hasher = identify_hasher(encoded)

    is_correct = hasher.verify(password, encoded)
    if not is_correct:
        return False

    # รหัสผ่านถูกต้อง แต่ hash ตัวนี้ hash ด้วยอัลกอริทึมเก่ากว่าตัวที่ตั้งไว้เป็น
    # ตัวแรกใน PASSWORD_HASHERS ตอนนี้ → อัปเกรด hash ให้ทันทีแบบเงียบ ๆ
    if setter and hasher is not preferred:
        if not is_password_usable(encoded):
            setter(password)
        elif preferred.verify(password, encoded) is False:
            setter(password)
    return True
```

พูดง่าย ๆ คือ: ทุกครั้งที่ผู้ใช้ **login สำเร็จ** (ผ่าน `authenticate()`) Django จะเช็ค
ว่า hash ปัจจุบันของ user คนนั้นถูก hash ด้วยอัลกอริทึมตัวแรกใน `PASSWORD_HASHERS`
หรือไม่ ถ้าไม่ใช่ (เช่น ยังเป็น PBKDF2 อยู่ แต่ตอนนี้เปลี่ยนตัวแรกเป็น Argon2 แล้ว)
Django จะ hash รหัสผ่านที่ผู้ใช้เพิ่งกรอกถูกต้องนั้นใหม่ด้วย Argon2 แล้วบันทึกทับทันที
**โดยผู้ใช้ไม่รู้ตัวเลยว่ามีการอัปเกรดเกิดขึ้น**

### 341.7 `AUTH_PASSWORD_VALIDATORS` เจาะลึกทุกตัว

Hashing ตอบคำถาม "เก็บรหัสผ่านอย่างไรให้ปลอดภัย" ส่วน **Password Validators** ตอบ
คำถามคนละข้อ: "จะยอมให้ผู้ใช้ตั้งรหัสผ่านแบบไหนได้บ้าง" — ทั้งสองเรื่องทำงานร่วมกัน
เป็นเกราะป้องกัน 2 ชั้น

ค่าเริ่มต้นของ Django (สร้างมาให้อัตโนมัติตอน `startproject` ตั้งแต่ Part 004):

```python
# config/settings.py
AUTH_PASSWORD_VALIDATORS = [
    {
        "NAME": "django.contrib.auth.password_validation.UserAttributeSimilarityValidator",
    },
    {
        "NAME": "django.contrib.auth.password_validation.MinimumLengthValidator",
    },
    {
        "NAME": "django.contrib.auth.password_validation.CommonPasswordValidator",
    },
    {
        "NAME": "django.contrib.auth.password_validation.NumericPasswordValidator",
    },
]
```

validator ทั้ง 4 ตัวนี้ถูกเรียกใช้โดย `validate_password()` ซึ่งถูกเรียกจากทั้ง
`UserCreationForm._post_clean()` (ที่เห็นไปแล้วใน Part 031 ขั้นตอนที่ 304.1) และ
`PasswordChangeForm`/`SetPasswordForm` (ที่จะเห็นในขั้นตอนที่ 343-344 ของ Part นี้)

#### `UserAttributeSimilarityValidator` — ห้ามคล้ายข้อมูลส่วนตัวของผู้ใช้

```python
{
    "NAME": "django.contrib.auth.password_validation.UserAttributeSimilarityValidator",
    "OPTIONS": {
        "user_attributes": ("username", "first_name", "last_name", "email"),
        "max_similarity": 0.7,
    },
},
```

- ตรวจสอบว่ารหัสผ่านที่ตั้ง **คล้าย** กับ `username`, `first_name`, `last_name`,
  หรือ `email` ของผู้ใช้คนนั้นมากเกินไปหรือไม่ (ใช้อัลกอริทึม
  `difflib.SequenceMatcher` เทียบความคล้ายเป็นสัดส่วน 0-1)
- `max_similarity` ค่าเริ่มต้น `0.7` หมายถึงถ้ารหัสผ่านคล้ายกับ attribute ใด
  attribute หนึ่งเกิน 70% จะถูกปฏิเสธ
- ตัวอย่างที่จะถูกบล็อก: username `somchai2026` ตั้งรหัสผ่านเป็น `somchai2026!`
  (คล้ายกันเกือบทั้งหมด)

#### `MinimumLengthValidator` — บังคับความยาวขั้นต่ำ

```python
{
    "NAME": "django.contrib.auth.password_validation.MinimumLengthValidator",
    "OPTIONS": {"min_length": 10},   # ค่าเริ่มต้นคือ 8 ถ้าไม่ระบุ OPTIONS
},
```

- ตรวจสอบเพียงอย่างเดียวคือ **จำนวนตัวอักษร** ต้องไม่น้อยกว่า `min_length`
- NIST Special Publication 800-63B (มาตรฐานความปลอดภัยรหัสผ่านที่ได้รับการยอมรับ
  กว้างขวางที่สุดในปี 2026) แนะนำความยาวขั้นต่ำ **8 ตัวอักษร** สำหรับระบบทั่วไป และ
  แนะนำว่า **ความยาวสำคัญกว่าความซับซ้อน** (เช่น รหัสผ่านยาว 16 ตัวที่เป็นประโยค
  ธรรมดาปลอดภัยกว่ารหัสผ่านสั้น 8 ตัวที่ผสมอักขระพิเศษ)

#### `CommonPasswordValidator` — ห้ามใช้รหัสผ่านยอดฮิต

```python
{
    "NAME": "django.contrib.auth.password_validation.CommonPasswordValidator",
    "OPTIONS": {"password_list_path": "/path/to/custom/common-passwords.txt.gz"},
    # ถ้าไม่ระบุ OPTIONS จะใช้ไฟล์ default ของ Django ที่มีรหัสผ่านยอดฮิตกว่า 20,000 คำ
},
```

- Django มาพร้อมไฟล์ `common-passwords.txt.gz` ที่บรรจุรหัสผ่านที่พบบ่อยที่สุดใน
  โลกกว่า **20,000 รายการ** (รวบรวมจากข้อมูลรหัสผ่านที่เคยรั่วไหลจริง) เช่น
  `password`, `123456`, `qwerty`, `letmein`
- ถ้ารหัสผ่านที่ผู้ใช้กรอก (แปลงเป็นตัวพิมพ์เล็กก่อนเทียบ) ตรงกับรายการนี้ **แม้แต่
  ตัวเดียว** จะถูกปฏิเสธทันที ไม่ว่าจะยาวแค่ไหนก็ตาม
- สามารถกำหนด `password_list_path` เป็นไฟล์ของตัวเองได้ เช่น เพิ่มรายการคำที่
  เกี่ยวข้องกับชื่อบริษัท/สินค้าของคุณเข้าไปด้วย (เช่น ห้ามใช้ `mycompany2026`)

#### `NumericPasswordValidator` — ห้ามเป็นตัวเลขล้วน

```python
{
    "NAME": "django.contrib.auth.password_validation.NumericPasswordValidator",
},
```

- ตรวจสอบง่าย ๆ ว่ารหัสผ่าน **ไม่ได้ประกอบด้วยตัวเลขล้วน ๆ** ทั้งหมด (เช่น
  `12345678` หรือ `19900115` ที่มักเป็นวันเกิด) เพราะตัวเลขล้วนถูก brute-force
  ได้ง่ายกว่ารหัสผ่านที่ผสมตัวอักษร แม้จะมีความยาวเท่ากันก็ตาม
- ไม่มี `OPTIONS` ให้ปรับแต่ง เป็น validator ที่ทำงานแบบ all-or-nothing

### 341.8 ตารางสรุป Validator ทั้ง 4 ตัว

| Validator | เช็คอะไร | ปรับแต่งผ่าน OPTIONS ได้ | ตัวอย่างรหัสผ่านที่ถูกบล็อก |
|---|---|---|---|
| `UserAttributeSimilarityValidator` | ความคล้ายกับ username/email/ชื่อ | ✅ `user_attributes`, `max_similarity` | `somchai123` (ถ้า username คือ `somchai`) |
| `MinimumLengthValidator` | จำนวนตัวอักษรขั้นต่ำ | ✅ `min_length` | `abc123` (สั้นกว่า 8 ตัว) |
| `CommonPasswordValidator` | อยู่ในลิสต์รหัสผ่านยอดฮิตหรือไม่ | ✅ `password_list_path` | `password123`, `qwerty123` |
| `NumericPasswordValidator` | เป็นตัวเลขล้วนหรือไม่ | ❌ ไม่มี OPTIONS | `12345678` |

### 341.9 ทดสอบ Validator ทั้งหมดผ่าน Shell

```python
python manage.py shell
```

```python
from django.contrib.auth.password_validation import validate_password
from django.core.exceptions import ValidationError

try:
    validate_password("12345678")
except ValidationError as e:
    for msg in e.messages:
        print("-", msg)
```

ผลลัพธ์ (แปลเป็นภาษาไทยถ้าตั้ง `LANGUAGE_CODE = "th"` และแปล locale ครบ):

```
- This password is too common.
- This password is entirely numeric.
```

สังเกตว่า `UserAttributeSimilarityValidator` ไม่ error เพราะเราไม่ได้ส่ง `user`
instance เข้าไป (ทดสอบแบบไม่ผูกกับ user คนใดคนหนึ่ง) — ถ้าต้องการทดสอบ validator
ตัวนี้ด้วย ต้องส่ง instance ของ user เข้าไปเป็น argument ที่สอง:

```python
from django.contrib.auth import get_user_model

User = get_user_model()
user = User(username="somchai", email="somchai@example.com")

try:
    validate_password("somchai123", user=user)
except ValidationError as e:
    for msg in e.messages:
        print("-", msg)
```

### 341.10 แสดง Help Text ของ Validator ทั้งหมดในฟอร์ม

```python
from django.contrib.auth.password_validation import password_validators_help_text_html

print(password_validators_help_text_html())
```

ผลลัพธ์เป็น HTML `<ul>` ที่รวม help text ของทุก validator ที่ตั้งค่าไว้ ใช้แสดงใต้
ช่องกรอกรหัสผ่านในหน้า signup/password change เพื่อบอกผู้ใช้**ล่วงหน้า**ว่าต้องตั้ง
รหัสผ่านแบบไหน (ดีกว่าปล่อยให้ error หลังกด submit เท่านั้น):

```html
<!-- templates/registration/signup.html -->
{% extends "base.html" %}

{% block content %}
<h1>สมัครสมาชิก</h1>
<form method="post">
    {% csrf_token %}
    {{ form.as_p }}
    <div class="password-help">
        {{ form.password1.help_text|safe }}
    </div>
    <button type="submit">สมัครสมาชิก</button>
</form>
{% endblock %}
```

`UserCreationForm.password1` field มี `help_text` ที่ถูกเซตให้เท่ากับผลลัพธ์ของ
`password_validators_help_text_html()` โดยอัตโนมัติอยู่แล้ว จึงเรียกใช้ผ่าน
`{{ form.password1.help_text|safe }}` ได้ทันทีโดยไม่ต้องเขียนโค้ด view เพิ่ม

---

## ขั้นตอนที่ 342: เขียน Custom Password Validator เอง

### 342.1 Interface Contract ที่ Validator ทุกตัวต้องมี

Validator ที่ Django เรียกใช้ผ่าน `AUTH_PASSWORD_VALIDATORS` ไม่จำเป็นต้องสืบทอด
จาก class ใด ๆ (ไม่มี abstract base class บังคับ) แต่ต้องมี **2 เมธอด** ตาม
"duck typing contract" นี้:

```python
class ExampleValidator:
    def validate(self, password, user=None):
        """
        ตรวจสอบ password (string) ถ้าไม่ผ่านเงื่อนไข ให้ raise
        django.core.exceptions.ValidationError พร้อมข้อความและ code
        ถ้าผ่าน ไม่ต้อง return อะไร (return None)
        """
        pass

    def get_help_text(self):
        """
        คืนค่า string อธิบายเงื่อนไขของ validator นี้ให้ผู้ใช้เห็นล่วงหน้า
        ถูกรวมเข้ากับ validator อื่นผ่าน password_validators_help_text_html()
        """
        pass
```

### 342.2 เขียน `NumberValidator`: บังคับต้องมีตัวเลขอย่างน้อย 1 ตัว

สังเกตว่า `NumericPasswordValidator` ในขั้นตอนที่ 341 ทำตรงข้ามกับสิ่งที่เราต้องการ
(มันห้าม "เป็นตัวเลขล้วน" ไม่ได้บังคับว่า "ต้องมีตัวเลข") เราจึงต้องเขียนเอง:

```python
# accounts/validators.py
import re

from django.core.exceptions import ValidationError
from django.utils.translation import gettext as _


class NumberValidator:
    """บังคับว่ารหัสผ่านต้องมีตัวเลขอย่างน้อย 1 ตัว"""

    def validate(self, password, user=None):
        if not re.findall(r"\d", password):
            raise ValidationError(
                _("รหัสผ่านต้องมีตัวเลขอย่างน้อย 1 ตัว"),
                code="password_no_number",
            )

    def get_help_text(self):
        return _("รหัสผ่านของคุณต้องมีตัวเลขอย่างน้อย 1 ตัว")
```

### 342.3 เขียน `UppercaseValidator`: บังคับต้องมีตัวพิมพ์ใหญ่

```python
# accounts/validators.py (ต่อจากด้านบน)
class UppercaseValidator:
    """บังคับว่ารหัสผ่านต้องมีตัวอักษรพิมพ์ใหญ่อย่างน้อย 1 ตัว"""

    def validate(self, password, user=None):
        if not re.findall(r"[A-Z]", password):
            raise ValidationError(
                _("รหัสผ่านต้องมีตัวอักษรพิมพ์ใหญ่ (A-Z) อย่างน้อย 1 ตัว"),
                code="password_no_upper",
            )

    def get_help_text(self):
        return _("รหัสผ่านของคุณต้องมีตัวอักษรพิมพ์ใหญ่ (A-Z) อย่างน้อย 1 ตัว")
```

### 342.4 เขียน `SpecialCharacterValidator`: บังคับต้องมีอักขระพิเศษ (bonus)

เพื่อความครบถ้วนของระบบระดับ professional สามารถเพิ่ม validator ที่สามอีกตัวได้
ในรูปแบบเดียวกัน:

```python
# accounts/validators.py (ต่อจากด้านบน)
class SpecialCharacterValidator:
    """บังคับว่ารหัสผ่านต้องมีอักขระพิเศษอย่างน้อย 1 ตัว"""

    SPECIAL_CHARACTERS = "!@#$%^&*()_+-=[]{}|;:,.<>?"

    def validate(self, password, user=None):
        if not any(char in self.SPECIAL_CHARACTERS for char in password):
            raise ValidationError(
                _("รหัสผ่านต้องมีอักขระพิเศษอย่างน้อย 1 ตัว เช่น !@#$%^&*"),
                code="password_no_special",
            )

    def get_help_text(self):
        return _("รหัสผ่านของคุณต้องมีอักขระพิเศษอย่างน้อย 1 ตัว เช่น !@#$%^&*")
```

### 342.5 ลงทะเบียนใน `AUTH_PASSWORD_VALIDATORS`

```python
# config/settings.py
AUTH_PASSWORD_VALIDATORS = [
    {
        "NAME": "django.contrib.auth.password_validation.UserAttributeSimilarityValidator",
    },
    {
        "NAME": "django.contrib.auth.password_validation.MinimumLengthValidator",
        "OPTIONS": {"min_length": 10},
    },
    {
        "NAME": "django.contrib.auth.password_validation.CommonPasswordValidator",
    },
    {
        "NAME": "django.contrib.auth.password_validation.NumericPasswordValidator",
    },
    # Custom validators ของเราเอง — ใช้ path เต็มแบบเดียวกับ built-in
    {
        "NAME": "accounts.validators.NumberValidator",
    },
    {
        "NAME": "accounts.validators.UppercaseValidator",
    },
]
```

> **ข้อสังเกต**: `NAME` ใช้ **dotted path string** เหมือน `INSTALLED_APPS`/
> `MIDDLEWARE` ไม่ใช่ import class ตรง ๆ — Django จะ import และสร้าง instance ให้
> เองตอนเริ่มระบบ (lazy loading pattern เดียวกับที่เรียนไปใน Part 006 เรื่อง
> `MIDDLEWARE`)

### 342.6 ทดสอบ Custom Validator ผ่าน Shell และฟอร์ม Signup จริง

```python
from django.contrib.auth.password_validation import validate_password
from django.core.exceptions import ValidationError

try:
    validate_password("weakpass")
except ValidationError as e:
    for msg in e.messages:
        print("-", msg)
```

```
- This password is too short. It must contain at least 10 characters.
- รหัสผ่านต้องมีตัวเลขอย่างน้อย 1 ตัว
- รหัสผ่านต้องมีตัวอักษรพิมพ์ใหญ่ (A-Z) อย่างน้อย 1 ตัว
```

ลองรหัสผ่านที่ผ่านทุกเงื่อนไข:

```python
validate_password("MySecure2026Pass")   # ไม่ raise อะไรเลย = ผ่านทุก validator
```

เมื่อเปิดหน้า `/accounts/signup/` ตอนนี้ `{{ form.password1.help_text }}` จะแสดง
help text ของ validator ทั้งหมดรวมกัน (built-in 4 ตัว + custom 2 ตัวที่เพิ่งเขียน)
โดยอัตโนมัติ ไม่ต้องแก้โค้ด view หรือ template เพิ่มเลย

### 342.7 เขียน Unit Test สำหรับ Custom Validator

ในงานจริงระดับมืออาชีพ logic ด้านความปลอดภัยแบบนี้**ต้องมี test คลุมเสมอ**
เพื่อป้องกันไม่ให้ใครมาแก้ regex แล้วทำให้ validator รั่วโดยไม่รู้ตัว:

```python
# accounts/tests.py
from django.core.exceptions import ValidationError
from django.test import TestCase

from .validators import NumberValidator, UppercaseValidator


class NumberValidatorTests(TestCase):
    def setUp(self):
        self.validator = NumberValidator()

    def test_password_with_number_passes(self):
        # ไม่ raise = ผ่าน
        self.validator.validate("MyPass1word")

    def test_password_without_number_fails(self):
        with self.assertRaises(ValidationError):
            self.validator.validate("MyPassword")

    def test_help_text_is_not_empty(self):
        self.assertTrue(self.validator.get_help_text())


class UppercaseValidatorTests(TestCase):
    def setUp(self):
        self.validator = UppercaseValidator()

    def test_password_with_uppercase_passes(self):
        self.validator.validate("MyPassword1")

    def test_password_without_uppercase_fails(self):
        with self.assertRaises(ValidationError):
            self.validator.validate("mypassword1")
```

```bash
python manage.py test accounts
```

---

## ขั้นตอนที่ 343: `PasswordChangeView`/`PasswordChangeForm` — เปลี่ยนรหัสผ่านตอน Login อยู่

### 343.1 ทบทวน URL ที่มีอยู่แล้วจาก Part 031

Part 031 ขั้นตอนที่ 306.2 แสดงตาราง URL ทั้งหมดจาก `django.contrib.auth.urls` ไว้
แล้ว รวมถึง URL สำหรับเปลี่ยนรหัสผ่านที่เรากำลังจะใช้จริงใน Part นี้:

| URL name | Path เต็ม | CBV ที่ใช้ |
|---|---|---|
| `password_change` | `/accounts/password_change/` | `PasswordChangeView` |
| `password_change_done` | `/accounts/password_change/done/` | `PasswordChangeDoneView` |

เนื่องจาก `config/urls.py` ของโปรเจกต์ `blog` มี
`path("accounts/", include("django.contrib.auth.urls"))` อยู่แล้วตั้งแต่ Part 031
เราจึง**ไม่ต้องเพิ่ม URL เอง** เหลือแค่สร้าง template 2 ไฟล์เท่านั้น

### 343.2 Template ที่ต้องสร้าง

```html
<!-- templates/registration/password_change_form.html -->
{% extends "base.html" %}

{% block content %}
<h1>เปลี่ยนรหัสผ่าน</h1>

<form method="post">
    {% csrf_token %}
    {{ form.as_p }}
    <div class="password-help">
        {{ form.new_password1.help_text|safe }}
    </div>
    <button type="submit">บันทึกรหัสผ่านใหม่</button>
</form>
{% endblock %}
```

```html
<!-- templates/registration/password_change_done.html -->
{% extends "base.html" %}

{% block content %}
<h1>เปลี่ยนรหัสผ่านสำเร็จ</h1>
<p>รหัสผ่านของคุณถูกเปลี่ยนเรียบร้อยแล้ว</p>
<a href="{% url 'blog:post-list' %}">กลับหน้าแรก</a>
{% endblock %}
```

`PasswordChangeView` ใช้ฟอร์ม `PasswordChangeForm` ที่มี 3 field: `old_password`
(รหัสผ่านปัจจุบัน ต้องกรอกให้ถูกก่อนถึงจะเปลี่ยนได้), `new_password1`,
`new_password2` — ทั้ง `new_password1`/`new_password2` ผ่าน
`validate_password()` เหมือน `UserCreationForm` ทุกประการ (ใช้
`AUTH_PASSWORD_VALIDATORS` ชุดเดียวกันจากขั้นตอนที่ 341-342)

### 343.3 ปัญหาเงียบที่ต้องเข้าใจ: เปลี่ยนรหัสผ่านแล้ว Session จะพังไหม

ในเชิงเทคนิค เมื่อ `password` ของ user เปลี่ยน ค่า hash ในฐานข้อมูลก็เปลี่ยนตามไป
ด้วย — และถ้า Django ไม่ทำอะไรเพิ่มเติม ระบบจะ **logout ผู้ใช้ทันทีหลังเปลี่ยนรหัสผ่าน
สำเร็จ** เพราะ session ผูกกับ hash ของรหัสผ่านเก่าอยู่ (จะอธิบายกลไกนี้ในขั้นตอนที่
343.4) แต่ในความเป็นจริง **`PasswordChangeView` แก้ปัญหานี้ให้อัตโนมัติแล้ว**
โค้ดจริงของ Django (ย่อเพื่อความเข้าใจ):

```python
# django/contrib/auth/views.py (โค้ดจริงย่อเพื่อความเข้าใจ)
class PasswordChangeView(PasswordContextMixin, FormView):
    form_class = PasswordChangeForm
    success_url = reverse_lazy("password_change_done")
    template_name = "registration/password_change_form.html"

    @method_decorator(login_required)
    def dispatch(self, *args, **kwargs):
        return super().dispatch(*args, **kwargs)

    def form_valid(self, form):
        form.save()
        # อัปเดต session hash ให้ตรงกับรหัสผ่านใหม่ทันที เพื่อไม่ให้
        # session ปัจจุบัน (ของ tab/browser นี้) ถูก logout ไปด้วย
        update_session_auth_hash(self.request, form.user)
        return super().form_valid(form)
```

### 343.4 เชื่อมโยงกับ `get_session_auth_hash()` (ต่อยอดจาก Part 034)

ตามที่ Part 034 ขั้นตอนที่ "เตรียมตัวสำหรับ Part ถัดไป" เกริ่นไว้ กลไกนี้เชื่อมโยง
โดยตรงกับระบบ Session ที่เรียนจบไป โค้ดจริงของ Django:

```python
# django/contrib/auth/base_user.py (โค้ดจริงย่อเพื่อความเข้าใจ)
class AbstractBaseUser(models.Model):
    def get_session_auth_hash(self):
        """
        คืนค่า HMAC hash ที่คำนวณจากฟิลด์ password ปัจจุบัน
        ใช้ตรวจสอบว่า password ไม่ได้ถูกเปลี่ยนไปนับตั้งแต่ session ถูกสร้าง
        """
        key_salt = "django.contrib.auth.base_user.AbstractBaseUser.get_session_auth_hash"
        return salted_hmac(key_salt, self.password, algorithm="sha256").hexdigest()
```

```python
# django/contrib/auth/__init__.py (โค้ดจริงย่อเพื่อความเข้าใจ)
def update_session_auth_hash(request, user):
    request.session.cycle_key()
    if hasattr(user, "get_session_auth_hash") and request.user == user:
        request.session[HASH_SESSION_KEY] = user.get_session_auth_hash()
```

กลไกทำงานดังนี้:

1. ตอน `login()` (Part 031 ขั้นตอนที่ 303.4) Django บันทึก
   `user.get_session_auth_hash()` ไว้ใน session data ด้วย
2. ทุก request ถัดไป `AuthenticationMiddleware` จะเทียบค่า hash ที่เก็บไว้ใน
   session กับ `user.get_session_auth_hash()` ที่คำนวณสดจากฐานข้อมูล ณ ขณะนั้น
3. ถ้าค่า**ไม่ตรงกัน** (เพราะ `password` เปลี่ยนไปแล้ว) Django จะถือว่า session
   นั้น**ไม่ถูกต้องอีกต่อไป** และ logout ผู้ใช้ทันที
4. `update_session_auth_hash()` แก้ปัญหานี้โดยอัปเดตค่า hash ใน session ปัจจุบัน
   ให้ตรงกับรหัสผ่านใหม่ **ทันทีหลัง save()** — ทำให้ browser/tab ที่กำลังใช้อยู่
   ไม่ถูก logout

**ผลลัพธ์ด้านความปลอดภัยที่สำคัญมาก**: เนื่องจาก `update_session_auth_hash()` แก้
เฉพาะ session ของ `request` ปัจจุบันเท่านั้น **session อื่น ๆ ทั้งหมดของผู้ใช้คนนี้
(เช่น ที่ login ค้างไว้ในโทรศัพท์ หรือเบราว์เซอร์เครื่องอื่น) จะถูก logout โดย
อัตโนมัติทันที** เมื่อรหัสผ่านถูกเปลี่ยน — นี่คือฟีเจอร์ความปลอดภัยที่ตั้งใจออกแบบ
มา: ถ้ามีใครขโมยบัญชีไปแล้วเจ้าของบัญชีรีบเปลี่ยนรหัสผ่าน ผู้บุกรุกจะถูกเตะออกจาก
ทุก session ที่ค้างอยู่ทันที โดยไม่ต้องเขียนโค้ดเพิ่มเลยแม้แต่บรรทัดเดียว

### 343.5 เขียน Password Change View แบบ Manual (สำหรับ Custom UI)

ในกรณีที่ต้องการควบคุม UI/UX เองทั้งหมด (เช่น เปลี่ยนรหัสผ่านผ่าน modal หรือ AJAX)
สามารถเขียนโดยใช้ `PasswordChangeForm` ตรง ๆ ได้:

```python
# accounts/views.py
from django.contrib.auth import update_session_auth_hash
from django.contrib.auth.decorators import login_required
from django.contrib.auth.forms import PasswordChangeForm
from django.contrib import messages
from django.shortcuts import redirect, render


@login_required
def change_password(request):
    if request.method == "POST":
        form = PasswordChangeForm(user=request.user, data=request.POST)
        if form.is_valid():
            user = form.save()
            update_session_auth_hash(request, user)   # ห้ามลืมบรรทัดนี้!
            messages.success(request, "เปลี่ยนรหัสผ่านสำเร็จ")
            return redirect("password_change_done")
    else:
        form = PasswordChangeForm(user=request.user)
    return render(request, "registration/password_change_form.html", {"form": form})
```

> **กฎเหล็ก**: ถ้าเขียน password change view เอง **ต้องเรียก
> `update_session_auth_hash(request, user)` เสมอหลัง save สำเร็จ** ไม่เช่นนั้น
> ผู้ใช้จะถูก logout ทันทีหลังเปลี่ยนรหัสผ่านสำเร็จโดยไม่มีใครแจ้งเตือนล่วงหน้า
> ซึ่งเป็นบั๊กด้าน UX ที่พบบ่อยมากเมื่อทีมพัฒนาไม่รู้กลไกนี้

### 343.6 Custom `PasswordChangeView` เพื่อเพิ่ม Success Message

```python
# accounts/views.py
from django.contrib import messages
from django.contrib.auth import views as auth_views
from django.urls import reverse_lazy


class CustomPasswordChangeView(auth_views.PasswordChangeView):
    template_name = "registration/password_change_form.html"
    success_url = reverse_lazy("blog:post-list")

    def form_valid(self, form):
        messages.success(self.request, "เปลี่ยนรหัสผ่านสำเร็จ")
        return super().form_valid(form)
```

```python
# config/urls.py — วางไว้ก่อน include() เพื่อ override URL name เดียวกัน
from django.urls import include, path

from accounts.views import CustomPasswordChangeView

urlpatterns = [
    path("accounts/password_change/", CustomPasswordChangeView.as_view(), name="password_change"),
    path("accounts/", include("django.contrib.auth.urls")),
    # ...
]
```

### 343.7 เพิ่มลิงก์ "เปลี่ยนรหัสผ่าน" ใน Navigation

```html
<!-- templates/base.html (ต่อยอดจาก Part 031 ขั้นตอนที่ 310.8) -->
{% if user.is_authenticated %}
    <span>สวัสดี, {{ user.username }}</span>
    <a href="{% url 'password_change' %}">เปลี่ยนรหัสผ่าน</a>
    <a href="{% url 'blog:post-create' %}">เขียนบทความ</a>
    <form method="post" action="{% url 'logout' %}" style="display: inline;">
        {% csrf_token %}
        <button type="submit">ออกจากระบบ</button>
    </form>
{% endif %}
```

---

## ขั้นตอนที่ 344: Password Reset Flow เต็มรูปแบบด้วย Built-in Views

### 344.1 ภาพรวมของ Flow ทั้งหมด

```
1. ผู้ใช้ลืมรหัสผ่าน กด "ลืมรหัสผ่าน?" ที่หน้า login
                            │
                            ▼
2. PasswordResetView (/accounts/password_reset/)
   ผู้ใช้กรอกอีเมล → ระบบส่งอีเมลที่มีลิงก์รีเซ็ตพร้อม token
                            │
                            ▼
3. PasswordResetDoneView (/accounts/password_reset/done/)
   หน้าแจ้งว่า "ถ้าอีเมลนี้มีในระบบ เราได้ส่งลิงก์ไปให้แล้ว"
                            │
                            ▼ (ผู้ใช้เปิดอีเมล กดลิงก์)
4. PasswordResetConfirmView (/accounts/reset/<uidb64>/<token>/)
   ตรวจสอบ token ว่าถูกต้องและยังไม่หมดอายุ → ให้กรอกรหัสผ่านใหม่
                            │
                            ▼
5. PasswordResetCompleteView (/accounts/reset/done/)
   หน้าแจ้งว่ารีเซ็ตสำเร็จ พร้อมลิงก์ไป login
```

ทั้ง 4 view นี้มาจาก `django.contrib.auth.urls` ที่รวมไว้แล้วตั้งแต่ Part 031 —
สิ่งที่ต้องทำใน Part นี้คือสร้าง **template ที่ยังขาดอยู่** และตั้งค่า **ระบบอีเมล**
ให้พร้อมใช้งานจริง

### 344.2 ตั้งค่าระบบอีเมลสำหรับ Development

ระหว่างพัฒนา ยังไม่ต้องเชื่อมต่อ SMTP server จริง ใช้ **console backend** ที่พิมพ์
เนื้อหาอีเมลออกมาที่ terminal แทนได้:

```python
# config/settings.py

# สำหรับ Development: พิมพ์อีเมลออกทาง terminal แทนการส่งจริง
EMAIL_BACKEND = "django.core.mail.backends.console.EmailBackend"

DEFAULT_FROM_EMAIL = "noreply@myblog.example.com"

# ระยะเวลาที่ลิงก์รีเซ็ตยังใช้ได้ (วินาที) — ค่าเริ่มต้นของ Django คือ 259200
# (3 วัน) เราลดเหลือ 1 วันเพื่อความปลอดภัยที่สูงขึ้นสำหรับบล็อกของเรา
PASSWORD_RESET_TIMEOUT = 60 * 60 * 24   # 1 วัน
```

สำหรับ production จะเปลี่ยนเป็น `django.core.mail.backends.smtp.EmailBackend`
พร้อมตั้งค่า `EMAIL_HOST`, `EMAIL_PORT`, `EMAIL_HOST_USER`,
`EMAIL_HOST_PASSWORD`, `EMAIL_USE_TLS` ตามผู้ให้บริการอีเมล (เช่น SendGrid,
Amazon SES, Mailgun) — รายละเอียดการ deploy ระบบอีเมล production เต็มรูปแบบจะ
อยู่ในช่วง Phase Deployment ของหลักสูตร

### 344.3 Template ทั้งหมดที่ต้องมีครบ

Password Reset Flow ต้องการ template **6 ไฟล์** วางไว้ใน `templates/registration/`:

| Template | ใช้โดย View | เนื้อหา |
|---|---|---|
| `password_reset_form.html` | `PasswordResetView` | ฟอร์มกรอกอีเมล |
| `password_reset_email.html` | `PasswordResetView` | เนื้อหาอีเมลที่จะส่งออก (จะเจาะลึกในขั้นตอนที่ 345) |
| `password_reset_subject.txt` | `PasswordResetView` | หัวข้ออีเมล (ต้องเป็นบรรทัดเดียว) |
| `password_reset_done.html` | `PasswordResetDoneView` | หน้าแจ้งว่าส่งอีเมลแล้ว |
| `password_reset_confirm.html` | `PasswordResetConfirmView` | ฟอร์มตั้งรหัสผ่านใหม่ |
| `password_reset_complete.html` | `PasswordResetCompleteView` | หน้าแจ้งว่าสำเร็จ |

```html
<!-- templates/registration/password_reset_form.html -->
{% extends "base.html" %}

{% block content %}
<h1>ลืมรหัสผ่าน?</h1>
<p>กรอกอีเมลที่ใช้สมัครสมาชิก เราจะส่งลิงก์สำหรับตั้งรหัสผ่านใหม่ไปให้</p>

<form method="post">
    {% csrf_token %}
    {{ form.as_p }}
    <button type="submit">ส่งลิงก์รีเซ็ตรหัสผ่าน</button>
</form>

<p><a href="{% url 'login' %}">กลับไปหน้าเข้าสู่ระบบ</a></p>
{% endblock %}
```

```html
<!-- templates/registration/password_reset_done.html -->
{% extends "base.html" %}

{% block content %}
<h1>ตรวจสอบอีเมลของคุณ</h1>
<p>
    หากอีเมลที่คุณกรอกมีอยู่ในระบบของเรา คุณจะได้รับอีเมลพร้อมคำแนะนำในการ
    ตั้งรหัสผ่านใหม่ในอีกไม่กี่นาที
</p>
<p>ไม่พบอีเมล? ลองตรวจสอบโฟลเดอร์ Spam หรือ Junk</p>
{% endblock %}
```

```html
<!-- templates/registration/password_reset_confirm.html -->
{% extends "base.html" %}

{% block content %}
<h1>ตั้งรหัสผ่านใหม่</h1>

{% if validlink %}
    <form method="post">
        {% csrf_token %}
        {{ form.as_p }}
        <div class="password-help">
            {{ form.new_password1.help_text|safe }}
        </div>
        <button type="submit">บันทึกรหัสผ่านใหม่</button>
    </form>
{% else %}
    <p>
        ลิงก์รีเซ็ตรหัสผ่านนี้ไม่ถูกต้องหรือหมดอายุแล้ว
        (อาจเป็นเพราะถูกใช้ไปแล้ว หรือเกิน
        {{ password_reset_timeout_days|default:"1" }} วันนับจากที่ขอ)
    </p>
    <a href="{% url 'password_reset' %}">ขอลิงก์ใหม่อีกครั้ง</a>
{% endif %}
{% endblock %}
```

```html
<!-- templates/registration/password_reset_complete.html -->
{% extends "base.html" %}

{% block content %}
<h1>ตั้งรหัสผ่านใหม่สำเร็จ</h1>
<p>คุณสามารถเข้าสู่ระบบด้วยรหัสผ่านใหม่ได้แล้ว</p>
<a href="{% url 'login' %}">เข้าสู่ระบบ</a>
{% endblock %}
```

> **สำคัญ**: `password_reset_confirm.html` ต้องเช็ค `{% if validlink %}` เสมอ
> เพราะ `PasswordResetConfirmView` จะ render template เดียวกันนี้ **ทั้งกรณี
> token ถูกต้องและไม่ถูกต้อง** โดยส่งตัวแปร `validlink` (`True`/`False`) มาบอกให้
> template ตัดสินใจว่าจะแสดงฟอร์มหรือข้อความ error

### 344.4 ความปลอดภัยของ `uidb64`/`token`: `PasswordResetTokenGenerator`

URL ของหน้า set รหัสผ่านใหม่มีรูปแบบ `/accounts/reset/<uidb64>/<token>/` เช่น:

```
http://127.0.0.1:8000/accounts/reset/MQ/c1a2b3-d4e5f6789abc/
```

| ส่วน | คืออะไร |
|---|---|
| `uidb64` | primary key ของ user ที่ถูก encode ด้วย `urlsafe_base64_encode()` (เช่น `MQ` คือ base64 ของเลข `1`) |
| `token` | token ที่สร้างโดย `PasswordResetTokenGenerator` ผูกกับ user คนนั้นโดยเฉพาะ |

จุดที่ฉลาดของ `PasswordResetTokenGenerator` คือ token ถูกคำนวณจากค่าที่**เปลี่ยนไป
ทันทีที่มีการใช้งาน** — โค้ดจริงของ Django (ย่อเพื่อความเข้าใจ):

```python
# django/contrib/auth/tokens.py (โค้ดจริงย่อเพื่อความเข้าใจ)
class PasswordResetTokenGenerator:
    def _make_hash_value(self, user, timestamp):
        login_timestamp = "" if user.last_login is None else user.last_login.replace(microsecond=0, tzinfo=None)
        email_field = user.get_email_field_name()
        email = getattr(user, email_field, "") or ""
        return f"{user.pk}{user.password}{login_timestamp}{timestamp}{email}"
```

Token ผูกกับ **`user.password` (hash ปัจจุบัน)** และ **`user.last_login`**
โดยตรง หมายความว่า:

- ถ้าผู้ใช้ login สำเร็จหลังขอ reset link → `last_login` เปลี่ยน → token เดิม
  **ใช้ไม่ได้อีกต่อไปทันที**
- ถ้ามีคนใช้ token นี้รีเซ็ตรหัสผ่านสำเร็จไปแล้ว → `user.password` เปลี่ยน →
  token เดิมกลายเป็นโมฆะทันที (ป้องกันการใช้ลิงก์เดิมซ้ำสอง)
- ยังมีการเช็ค **timestamp** เทียบกับ `PASSWORD_RESET_TIMEOUT` ที่ตั้งไว้ใน
  ขั้นตอนที่ 344.2 อีกชั้นหนึ่งด้วย

นี่คือเหตุผลที่ลิงก์รีเซ็ตรหัสผ่านของ Django **ใช้ได้แค่ครั้งเดียว** และ **หมดอายุ
อัตโนมัติ** โดยไม่ต้องเก็บ token แยกไว้ในฐานข้อมูลเลย (เป็น **stateless token**
ที่คำนวณใหม่ทุกครั้งเพื่อเทียบ ไม่ใช่ token ที่บันทึกไว้ล่วงหน้า)

### 344.5 ทดสอบ Flow ทั้งหมดด้วย Console Email Backend

```bash
python manage.py runserver
```

1. เปิด `http://127.0.0.1:8000/accounts/password_reset/`
2. กรอกอีเมลของ user ที่มีอยู่จริงในระบบ (เช่น `admin@example.com`)
3. กด submit → ควรถูก redirect ไป `password_reset_done`
4. กลับไปดู terminal ที่รัน `runserver` ควรเห็นเนื้อหาอีเมลถูกพิมพ์ออกมา:

```
Content-Type: text/plain; charset="utf-8"
Subject: Password reset on 127.0.0.1:8000
From: noreply@myblog.example.com
To: admin@example.com

You're receiving this email because you requested a password reset...

Please go to the following page and choose a new password:

http://127.0.0.1:8000/accounts/reset/MQ/c1a2b3-d4e5f6789abc/

Your username, in case you've forgotten: admin
```

5. คัดลอกลิงก์จาก terminal ไปวางในเบราว์เซอร์ → ควรเห็นฟอร์มตั้งรหัสผ่านใหม่
6. กรอกรหัสผ่านใหม่ที่ผ่าน validator ทุกตัว → ควรถูก redirect ไปหน้าสำเร็จ
7. ลอง login ด้วยรหัสผ่านใหม่ → ควรสำเร็จ
8. ลองย้อนกลับไปใช้ลิงก์เดิมซ้ำอีกครั้ง → ควรเห็น `validlink = False`

### 344.6 เพิ่มลิงก์ "ลืมรหัสผ่าน?" ที่หน้า Login

```html
<!-- templates/registration/login.html (ต่อยอดจาก Part 031 ขั้นตอนที่ 310.8) -->
{% extends "base.html" %}

{% block content %}
<h1>เข้าสู่ระบบ</h1>
<form method="post">
    {% csrf_token %}
    {{ form.as_p }}
    <button type="submit">เข้าสู่ระบบ</button>
</form>
<p><a href="{% url 'password_reset' %}">ลืมรหัสผ่าน?</a></p>
<p>ยังไม่มีบัญชี? <a href="{% url 'signup' %}">สมัครสมาชิก</a></p>
{% endblock %}
```

---

## ขั้นตอนที่ 345: Custom Email Template สำหรับ Password Reset (HTML + Plain Text)

### 345.1 Template อีเมลเริ่มต้นของ Django เรียบง่ายเกินไปสำหรับงานจริง

Django มี `password_reset_email.html` เริ่มต้นซ่อนอยู่ในแอป
`django.contrib.admin` (เป็น plain text ล้วน ไม่มีการจัดรูปแบบใด ๆ) ซึ่งใช้งานได้
แต่ดูไม่เป็นมืออาชีพ ในงานจริงเราต้องการอีเมลที่มีทั้งแบรนด์ ดีไซน์ และรองรับทั้ง
**HTML** และ **plain text** (สำหรับอีเมลไคลเอนต์รุ่นเก่าหรือผู้ใช้ที่ปิด HTML)

### 345.2 หัวข้ออีเมล: `password_reset_subject.txt`

```
<!-- templates/registration/password_reset_subject.txt -->
รีเซ็ตรหัสผ่านบัญชี {{ user.username }} ที่ My Blog
```

> **ข้อควรระวัง**: ไฟล์นี้ **ต้องเป็นบรรทัดเดียวเท่านั้น** เพราะ subject อีเมลตาม
> มาตรฐาน RFC 2822 ห้ามมีการขึ้นบรรทัดใหม่ — Django จะ `.strip()` ค่าที่ render
> ได้ให้อัตโนมัติ แต่ถ้าเผลอใส่ขึ้นบรรทัดใหม่กลางข้อความจะทำให้ header ของอีเมล
> ผิดรูปแบบและอาจถูกระบบอีเมลปลายทางปฏิเสธ

### 345.3 Plain Text Body: `password_reset_email.html`

แม้ชื่อไฟล์จะลงท้ายด้วย `.html` แต่ `PasswordResetForm.send_mail()` render มันเป็น
**plain text** เสมอ (นี่คือ convention ของ Django ที่มือใหม่มักสับสน) — เนื้อหา
ไฟล์นี้จึงควรเป็นข้อความธรรมดา ไม่ใช่ HTML tag:

```
<!-- templates/registration/password_reset_email.html -->
สวัสดีคุณ {{ user.username }},

คุณได้รับอีเมลนี้เพราะมีการขอรีเซ็ตรหัสผ่านสำหรับบัญชีของคุณที่ My Blog
({{ domain }})

กรุณาคลิกลิงก์ด้านล่างเพื่อตั้งรหัสผ่านใหม่:

{{ protocol }}://{{ domain }}{% url 'password_reset_confirm' uidb64=uid token=token %}

ลิงก์นี้จะหมดอายุภายใน 24 ชั่วโมง และใช้ได้เพียงครั้งเดียว

หากคุณไม่ได้เป็นผู้ขอรีเซ็ตรหัสผ่านนี้ กรุณาเพิกเฉยต่ออีเมลฉบับนี้ได้เลย
รหัสผ่านของคุณจะไม่ถูกเปลี่ยนแปลงใด ๆ ทั้งสิ้น

ขอบคุณที่ใช้บริการ
ทีมงาน My Blog
```

### 345.4 HTML Body: `password_reset_email_html.html`

เพื่อให้อีเมลดูเป็นมืออาชีพ สร้างไฟล์ HTML แยกต่างหาก:

```html
<!-- templates/registration/password_reset_email_html.html -->
<!DOCTYPE html>
<html lang="th">
<head>
    <meta charset="UTF-8">
    <title>รีเซ็ตรหัสผ่าน</title>
</head>
<body style="margin: 0; padding: 0; background-color: #f4f4f5; font-family: Arial, sans-serif;">
    <table role="presentation" width="100%" cellpadding="0" cellspacing="0" style="background-color: #f4f4f5; padding: 32px 0;">
        <tr>
            <td align="center">
                <table role="presentation" width="480" cellpadding="0" cellspacing="0" style="background-color: #ffffff; border-radius: 8px; overflow: hidden;">
                    <tr>
                        <td style="background-color: #1f2937; padding: 24px; text-align: center;">
                            <h1 style="color: #ffffff; margin: 0; font-size: 20px;">My Blog</h1>
                        </td>
                    </tr>
                    <tr>
                        <td style="padding: 32px 24px;">
                            <p style="font-size: 16px; color: #111827;">สวัสดีคุณ {{ user.username }},</p>
                            <p style="font-size: 14px; color: #374151; line-height: 1.6;">
                                คุณได้รับอีเมลนี้เพราะมีการขอรีเซ็ตรหัสผ่านสำหรับบัญชีของคุณที่
                                <strong>{{ domain }}</strong>
                            </p>
                            <table role="presentation" cellpadding="0" cellspacing="0" style="margin: 24px 0;">
                                <tr>
                                    <td style="border-radius: 6px; background-color: #2563eb;">
                                        <a href="{{ protocol }}://{{ domain }}{% url 'password_reset_confirm' uidb64=uid token=token %}"
                                           style="display: inline-block; padding: 12px 24px; color: #ffffff; text-decoration: none; font-size: 14px; font-weight: bold;">
                                            ตั้งรหัสผ่านใหม่
                                        </a>
                                    </td>
                                </tr>
                            </table>
                            <p style="font-size: 13px; color: #6b7280;">
                                ลิงก์นี้จะหมดอายุภายใน 24 ชั่วโมง และใช้ได้เพียงครั้งเดียว
                                หากคุณไม่ได้เป็นผู้ขอ กรุณาเพิกเฉยต่ออีเมลนี้ได้เลย
                            </p>
                        </td>
                    </tr>
                </table>
            </td>
        </tr>
    </table>
</body>
</html>
```

> **ข้อควรระวังสำหรับอีเมล HTML**: ใช้ **inline CSS เท่านั้น** (ไม่ใช้ `<link>`
> หรือ `<style>` block แยก) และใช้ `<table>` จัด layout แทน `<div>`/flexbox/grid
> เพราะอีเมลไคลเอนต์จำนวนมาก (โดยเฉพาะ Outlook) มี rendering engine ที่รองรับ
> CSS สมัยใหม่ได้จำกัดมาก นี่คือข้อจำกัดเฉพาะตัวของการเขียน HTML สำหรับอีเมลที่
> ต่างจากการเขียนหน้าเว็บทั่วไปโดยสิ้นเชิง

### 345.5 เชื่อม HTML Template เข้ากับ `PasswordResetView`

ต้อง override `PasswordResetView` เพื่อระบุ `html_email_template_name` เพราะ
ค่าเริ่มต้นของ Django ไม่ได้เปิดใช้ HTML email:

```python
# accounts/views.py
from django.contrib.auth import views as auth_views
from django.urls import reverse_lazy


class CustomPasswordResetView(auth_views.PasswordResetView):
    template_name = "registration/password_reset_form.html"
    email_template_name = "registration/password_reset_email.html"
    html_email_template_name = "registration/password_reset_email_html.html"
    subject_template_name = "registration/password_reset_subject.txt"
    success_url = reverse_lazy("password_reset_done")
```

```python
# config/urls.py
from accounts.views import CustomPasswordResetView

urlpatterns = [
    path("accounts/password_reset/", CustomPasswordResetView.as_view(), name="password_reset"),
    path("accounts/", include("django.contrib.auth.urls")),
    # ...
]
```

เบื้องหลัง `PasswordResetForm.send_mail()` ของ Django จะสร้างอีเมลแบบ
**multipart/alternative** ที่มีทั้งสองเวอร์ชันฝังอยู่ในฉบับเดียวกัน โค้ดจริง
(ย่อเพื่อความเข้าใจ):

```python
# django/contrib/auth/forms.py (โค้ดจริงย่อเพื่อความเข้าใจ)
def send_mail(self, subject_template_name, email_template_name, context, from_email, to_email, html_email_template_name=None):
    subject = loader.render_to_string(subject_template_name, context)
    subject = "".join(subject.splitlines())
    body = loader.render_to_string(email_template_name, context)

    email_message = EmailMultiAlternatives(subject, body, from_email, [to_email])
    if html_email_template_name is not None:
        html_email = loader.render_to_string(html_email_template_name, context)
        email_message.attach_alternative(html_email, "text/html")

    email_message.send()
```

อีเมลไคลเอนต์ที่รองรับ HTML จะแสดงเวอร์ชัน HTML ที่สวยงาม ส่วนไคลเอนต์ที่ไม่รองรับ
(หรือผู้ใช้ตั้งค่าให้อ่านแค่ plain text) จะเห็นเวอร์ชัน plain text แทนโดยอัตโนมัติ
— ผู้ใช้ไม่ต้องทำอะไรเอง ระบบเลือกให้เอง

### 345.6 ทดสอบด้วย Console Backend อีกครั้ง

รันซ้ำตามขั้นตอนที่ 344.5 แล้วสังเกต output จาก console backend คราวนี้จะเห็น
โครงสร้าง multipart:

```
Content-Type: multipart/alternative;
 boundary="===============1234567890=="
MIME-Version: 1.0
Subject: รีเซ็ตรหัสผ่านบัญชี admin ที่ My Blog
From: noreply@myblog.example.com
To: admin@example.com

--===============1234567890==
Content-Type: text/plain; charset="utf-8"

สวัสดีคุณ admin,
...

--===============1234567890==
Content-Type: text/html; charset="utf-8"

<!DOCTYPE html>
<html lang="th">
...
```

สำหรับการทดสอบแบบเห็นผลจริงในกล่องจดหมาย แนะนำให้ใช้บริการอย่าง **Mailtrap** หรือ
**Mailhog** ระหว่างพัฒนา (จะแนะนำการตั้งค่าเต็มรูปแบบในช่วง Phase Deployment)
แทนการเดา rendering จาก console เพียงอย่างเดียว เพราะอีเมลไคลเอนต์แต่ละเจ้า
render CSS ต่างกันพอสมควร

---

## ขั้นตอนที่ 346: `set_password()`/`check_password()` แบบ Manual

### 346.1 กรณีที่ต้องใช้ Manual API แทนฟอร์มสำเร็จรูป

ฟอร์มอย่าง `UserCreationForm`, `PasswordChangeForm` เรียก `set_password()` ให้
อัตโนมัติอยู่แล้วเบื้องหลัง แต่มีหลายสถานการณ์ในงานจริงที่ต้องเรียกเมธอดเหล่านี้
**ตรง ๆ ด้วยตัวเอง**:

| สถานการณ์ | ทำไมต้องเรียกเอง |
|---|---|
| Management command สำหรับแอดมินระบบรีเซ็ตรหัสผ่านให้ user | ไม่มีฟอร์ม HTTP เกี่ยวข้อง |
| Data migration ที่ต้องตั้งรหัสผ่านเริ่มต้นให้ user ที่ import มาจากระบบเก่า | รันนอก request-response cycle |
| API endpoint แบบเขียนเอง (ก่อนเรียนเรื่อง DRF ใน Phase 6) | ควบคุม logic เองทั้งหมด |
| ยืนยันรหัสผ่านปัจจุบันก่อนอนุญาตให้ทำ action ที่ sensitive เช่น ลบบัญชี | ต้องเช็คโดยไม่ผ่านฟอร์ม login |

### 346.2 `set_password()`: Hash และเตรียมพร้อมสำหรับ `save()`

```python
python manage.py shell
```

```python
from django.contrib.auth import get_user_model

User = get_user_model()
user = User.objects.get(username="somchai")

user.set_password("NewSecure2026Pass")
user.save()   # ห้ามลืมบรรทัดนี้! set_password() แค่เตรียม hash ไว้ใน memory เท่านั้น
```

`set_password(raw_password)` ทำหน้าที่แค่ **คำนวณ hash แล้วเก็บลง
`self.password`** เท่านั้น **ไม่ได้บันทึกลงฐานข้อมูลให้อัตโนมัติ** — ต้องเรียก
`.save()` เองเสมอ (หรือ `.save(update_fields=["password"])` ถ้าต้องการอัปเดต
เฉพาะ field นี้เพื่อประสิทธิภาพ ตามที่เรียนเรื่อง `update_fields` ไปใน Part 011)

### 346.3 `check_password()`: ตรวจสอบรหัสผ่านโดยไม่ผ่าน `authenticate()`

```python
user = User.objects.get(username="somchai")

is_correct = user.check_password("NewSecure2026Pass")
print(is_correct)   # True

is_correct = user.check_password("WrongPassword")
print(is_correct)   # False
```

`check_password()` มีประโยชน์มากในกรณีที่ **ผู้ใช้ login อยู่แล้ว** (มี
`request.user` พร้อมใช้) แต่ต้องการยืนยันตัวตนซ้ำก่อนทำ action ที่สำคัญมาก เช่น
ลบบัญชี หรือเปลี่ยนอีเมล — ไม่จำเป็นต้องใช้ `authenticate()` ที่ต้องส่ง
`username`/`password` ใหม่ทั้งคู่ เพราะเรามี user instance อยู่แล้ว:

```python
# accounts/views.py
from django.contrib import messages
from django.contrib.auth.decorators import login_required
from django.shortcuts import redirect, render


@login_required
def delete_account(request):
    if request.method == "POST":
        password = request.POST.get("password")
        if request.user.check_password(password):
            username = request.user.username
            request.user.delete()
            messages.success(request, f"ลบบัญชี {username} เรียบร้อยแล้ว")
            return redirect("blog:post-list")
        else:
            messages.error(request, "รหัสผ่านไม่ถูกต้อง ไม่สามารถลบบัญชีได้")
    return render(request, "accounts/delete_account.html")
```

### 346.4 Management Command: รีเซ็ตรหัสผ่านให้ User โดยแอดมิน

```python
# accounts/management/commands/reset_user_password.py
import getpass

from django.contrib.auth import get_user_model
from django.contrib.auth.password_validation import validate_password
from django.core.exceptions import ValidationError
from django.core.management.base import BaseCommand, CommandError

User = get_user_model()


class Command(BaseCommand):
    help = "รีเซ็ตรหัสผ่านให้ user ที่ระบุ (ใช้สำหรับแอดมินช่วยเหลือ user ที่ล็อกอินไม่ได้)"

    def add_arguments(self, parser):
        parser.add_argument("username", type=str, help="username ของ user ที่ต้องการรีเซ็ตรหัสผ่าน")

    def handle(self, *args, **options):
        username = options["username"]

        try:
            user = User.objects.get(username=username)
        except User.DoesNotExist:
            raise CommandError(f'ไม่พบ user ที่มี username "{username}"')

        new_password = getpass.getpass("รหัสผ่านใหม่: ")
        confirm_password = getpass.getpass("ยืนยันรหัสผ่านใหม่อีกครั้ง: ")

        if new_password != confirm_password:
            raise CommandError("รหัสผ่านทั้งสองครั้งไม่ตรงกัน")

        try:
            validate_password(new_password, user=user)
        except ValidationError as e:
            raise CommandError("\n".join(e.messages))

        user.set_password(new_password)
        user.save(update_fields=["password"])

        self.stdout.write(
            self.style.SUCCESS(f'รีเซ็ตรหัสผ่านให้ "{username}" สำเร็จแล้ว')
        )
```

```bash
python manage.py reset_user_password somchai
```

```
รหัสผ่านใหม่:
ยืนยันรหัสผ่านใหม่อีกครั้ง:
รีเซ็ตรหัสผ่านให้ "somchai" สำเร็จแล้ว
```

สังเกตว่า command นี้ใช้ `getpass.getpass()` แทน `input()` ธรรมดา เพื่อไม่ให้
รหัสผ่านที่พิมพ์ปรากฏบนหน้าจอ terminal (เทคนิคเดียวกับที่ `createsuperuser` ใช้)
และเรียก `validate_password()` เพื่อบังคับใช้ `AUTH_PASSWORD_VALIDATORS` ชุด
เดียวกับที่ผู้ใช้ทั่วไปต้องผ่าน แม้จะเป็นแอดมินตั้งให้ก็ตาม

### 346.5 ตารางสรุปข้อควรระวังของ Manual API

| ข้อควรระวัง | คำอธิบาย |
|---|---|
| ห้ามใช้ `user.password = "..."` ตรง ๆ | จะเก็บ plaintext ทันที **ไม่ผ่านการ hash เลย** เป็นข้อผิดพลาดร้ายแรงที่สุดที่เกิดขึ้นได้ |
| ต้องเรียก `.save()` เสมอหลัง `set_password()` | `set_password()` ไม่บันทึกฐานข้อมูลให้อัตโนมัติ |
| เรียก `validate_password()` เองถ้า bypass ฟอร์ม | ฟอร์มอย่าง `UserCreationForm` เรียกให้อัตโนมัติ แต่การเรียก `set_password()` ตรง ๆ **ไม่มีการ validate ใด ๆ ทั้งสิ้น** |
| เรียก `update_session_auth_hash()` ถ้าเปลี่ยนรหัสผ่านของ user ที่ login request นี้อยู่ | ไม่เช่นนั้นผู้ใช้จะถูก logout ทันที (ตามขั้นตอนที่ 343.4) |
| `check_password()` คืน `False` เสมอถ้า `user.is_active = False` เมื่อเรียกผ่าน `authenticate()` | แต่ `check_password()` ที่เรียกตรงบน instance **ไม่เช็ค `is_active`** เพราะแค่เทียบ hash เท่านั้น ต้องเช็ค `is_active` เพิ่มเองถ้าต้องการ |

---

## ขั้นตอนที่ 347: Password Strength Meter ฝั่ง Client ด้วย zxcvbn

### 347.1 ทำไม Server-Side Validation อย่างเดียวไม่พอสำหรับ UX ที่ดี

Validator จากขั้นตอนที่ 341-342 ทำงานได้ดีมากในแง่ความปลอดภัย แต่มีข้อจำกัดด้าน
ประสบการณ์ผู้ใช้: **ผู้ใช้ต้องกด submit ฟอร์มก่อนถึงจะรู้ว่ารหัสผ่านผ่านหรือไม่**
ระบบระดับโลกอย่าง Dropbox, GitHub เลือกแสดง **Password Strength Meter** ที่อัปเดต
สดขณะพิมพ์ เพื่อให้ผู้ใช้ปรับรหัสผ่านได้ทันทีโดยไม่ต้องรอ error หลัง submit

### 347.2 zxcvbn: Library วิเคราะห์ความแข็งแรงของรหัสผ่านจาก Dropbox

**zxcvbn** เป็น JavaScript library ที่ Dropbox พัฒนาและเปิด source ให้ใช้ฟรี
จุดเด่นคือมันไม่ได้เช็คแค่ "มีตัวเลข/ตัวพิมพ์ใหญ่หรือไม่" แบบ regex ทั่วไป แต่
วิเคราะห์แบบ **pattern matching** ที่คำนึงถึงรูปแบบที่คนจริงมักใช้ เช่น คำในพจนานุกรม,
วันที่, การเรียงคีย์บอร์ด (เช่น `qwerty`, `asdf`), ชื่อยอดนิยม และให้คะแนนออกมาเป็น
ตัวเลข 0-4

```html
<!-- ใส่ใน <head> หรือก่อนปิด </body> ของ signup.html -->
<script src="https://cdnjs.cloudflare.com/ajax/libs/zxcvbn/4.4.2/zxcvbn.js"></script>
```

### 347.3 นำ zxcvbn มาใช้ในหน้า Signup

```html
<!-- templates/registration/signup.html -->
{% extends "base.html" %}

{% block content %}
<h1>สมัครสมาชิก</h1>

<form method="post" id="signup-form">
    {% csrf_token %}
    {{ form.username }}
    {{ form.email }}
    {{ form.password1 }}

    <div id="password-strength-meter" style="height: 6px; background: #e5e7eb; border-radius: 3px; margin-top: 8px;">
        <div id="password-strength-bar" style="height: 100%; width: 0%; border-radius: 3px; transition: width 0.2s, background-color 0.2s;"></div>
    </div>
    <p id="password-strength-label" style="font-size: 13px; margin-top: 4px;"></p>

    {{ form.password2 }}
    <button type="submit">สมัครสมาชิก</button>
</form>

<script src="https://cdnjs.cloudflare.com/ajax/libs/zxcvbn/4.4.2/zxcvbn.js"></script>
<script>
    const passwordInput = document.querySelector("#id_password1");
    const strengthBar = document.querySelector("#password-strength-bar");
    const strengthLabel = document.querySelector("#password-strength-label");

    const STRENGTH_LEVELS = [
        { label: "แย่มาก", color: "#ef4444", width: "20%" },
        { label: "อ่อน", color: "#f97316", width: "40%" },
        { label: "พอใช้", color: "#eab308", width: "60%" },
        { label: "ดี", color: "#84cc16", width: "80%" },
        { label: "ดีเยี่ยม", color: "#22c55e", width: "100%" },
    ];

    passwordInput.addEventListener("input", function () {
        const password = passwordInput.value;

        if (password.length === 0) {
            strengthBar.style.width = "0%";
            strengthLabel.textContent = "";
            return;
        }

        // zxcvbn() คืนค่า object ที่มี score ตั้งแต่ 0 (แย่ที่สุด) ถึง 4 (ดีที่สุด)
        const result = zxcvbn(password);
        const level = STRENGTH_LEVELS[result.score];

        strengthBar.style.width = level.width;
        strengthBar.style.backgroundColor = level.color;
        strengthLabel.textContent = "ความแข็งแรงของรหัสผ่าน: " + level.label;

        // zxcvbn ยังให้คำแนะนำเป็นภาษาอังกฤษผ่าน result.feedback ด้วย
        if (result.feedback.warning) {
            strengthLabel.textContent += " (" + result.feedback.warning + ")";
        }
    });
</script>
{% endblock %}
```

### 347.4 กฎเหล็ก: Client-Side คือ UX เท่านั้น ไม่ใช่เกราะป้องกันจริง

| ประเด็น | Client-Side (zxcvbn) | Server-Side (`AUTH_PASSWORD_VALIDATORS`) |
|---|---|---|
| วัตถุประสงค์ | ให้ feedback ทันทีเพื่อ UX ที่ดี | บังคับใช้กฎความปลอดภัยจริง |
| ผู้ใช้สามารถ bypass ได้หรือไม่ | ✅ ได้ทันที (ปิด JavaScript, แก้ DevTools, หรือยิง request ตรงด้วย `curl`) | ❌ ไม่ได้ (รันบนเซิร์ฟเวอร์ ผู้ใช้ควบคุมไม่ได้) |
| ควรเชื่อถือเป็นแหล่งอ้างอิงสุดท้ายหรือไม่ | ❌ ไม่ควรเด็ดขาด | ✅ ใช่ เป็นด่านสุดท้ายที่แท้จริง |
| ควรใช้งานอย่างไร | เสริม UX ควบคู่ไปกับ server-side เสมอ | บังคับใช้เสมอ ไม่ว่า client จะเช็คมาก่อนหรือไม่ |

**กฎเหล็กด้านความปลอดภัยที่ต้องจำขึ้นใจ**: **ห้ามเชื่อ validation ฝั่ง client
เพียงอย่างเดียวเด็ดขาด** เพราะผู้โจมตีสามารถส่ง HTTP request ตรงไปที่ endpoint
โดยไม่ผ่านหน้าเว็บที่มี JavaScript เลยก็ได้ (เช่นใช้ `curl` หรือ Postman) ระบบ
`AUTH_PASSWORD_VALIDATORS` และ Custom Validator จาก Part นี้จึงยังคง**จำเป็น
เสมอ** ไม่ว่าจะมี zxcvbn อยู่หน้าบ้านหรือไม่ก็ตาม

---

## ขั้นตอนที่ 348: Rate Limiting การพยายาม Login ด้วย django-axes (เกริ่นนำ)

### 348.1 ทำไม Hashing ที่ดีอย่างเดียวยังไม่พอ

แม้ Part นี้จะสอนการ hash รหัสผ่านอย่างปลอดภัย (ขั้นตอนที่ 341) และบังคับความ
ซับซ้อนของรหัสผ่าน (ขั้นตอนที่ 342) ไปแล้ว แต่ยังมีช่องโหว่อีกจุดที่ยังไม่ได้ปิด:
**ผู้โจมตีสามารถลองรหัสผ่านซ้ำ ๆ ผ่านหน้า login ได้ไม่จำกัดจำนวนครั้ง**
(Brute Force Attack) หรือใช้รายชื่อ username/password ที่หลุดจากเว็บอื่นมาลองไล่
ทีละคู่ (Credential Stuffing Attack) — ทั้งสองวิธีนี้ไม่เกี่ยวกับความแข็งแรงของ
hash เลย แต่อาศัยการ "เดา" ซ้ำ ๆ จนกว่าจะถูก

### 348.2 ติดตั้ง django-axes

**django-axes** เป็น package ยอดนิยมที่ล็อกบัญชีหรือ IP ชั่วคราวหลังพยายาม login
ผิดพลาดเกินจำนวนที่กำหนด (คล้ายกลไก "ใส่ PIN บัตร ATM ผิด 3 ครั้งแล้วบัตรถูกยึด"):

```bash
pip install django-axes
pip freeze > requirements.txt
```

```python
# config/settings.py
INSTALLED_APPS = [
    # ...
    "django.contrib.auth",
    "django.contrib.contenttypes",
    "axes",   # ต้องอยู่หลัง contenttypes
    "blog",
    "accounts",
]

MIDDLEWARE = [
    "django.middleware.security.SecurityMiddleware",
    "django.contrib.sessions.middleware.SessionMiddleware",
    "django.middleware.common.CommonMiddleware",
    "django.middleware.csrf.CsrfViewMiddleware",
    "django.contrib.auth.middleware.AuthenticationMiddleware",
    "axes.middleware.AxesMiddleware",   # ต้องอยู่ "ท้ายสุด" ของ MIDDLEWARE เสมอ
    "django.contrib.messages.middleware.MessageMiddleware",
    "django.middleware.clickjacking.XFrameOptionsMiddleware",
]

AUTHENTICATION_BACKENDS = [
    "axes.backends.AxesStandaloneBackend",   # ต้องอยู่ "ตัวแรก" ของรายการเสมอ
    "django.contrib.auth.backends.ModelBackend",
]
```

```bash
python manage.py migrate axes
```

### 348.3 ตั้งค่าพื้นฐานของ django-axes

```python
# config/settings.py

# ล็อกหลังพยายาม login ผิด 5 ครั้งติดต่อกัน
AXES_FAILURE_LIMIT = 5

# ล็อกไว้ 1 ชั่วโมง (หน่วยเป็นชั่วโมง)
AXES_COOLOFF_TIME = 1

# นับความผิดพลาดแยกตามคู่ username + IP address
# (ป้องกันไม่ให้ผู้ใช้คนหนึ่งถูกล็อกเพราะคนอื่นพิมพ์ username ผิดจาก IP เดียวกัน
# เช่น สำนักงานที่ใช้ IP สาธารณะร่วมกัน)
AXES_LOCKOUT_PARAMETERS = ["ip_address", "username"]

# แสดงข้อความแจ้งเตือนที่เป็นมิตรแทน error ทั่วไปเมื่อถูกล็อก
AXES_LOCKOUT_TEMPLATE = "registration/account_locked.html"
```

### 348.4 ตารางสรุป Settings หลักของ django-axes

| Setting | ความหมาย |
|---|---|
| `AXES_FAILURE_LIMIT` | จำนวนครั้งที่ยอมให้ login ผิดก่อนถูกล็อก (ค่าเริ่มต้น 3) |
| `AXES_COOLOFF_TIME` | ระยะเวลาที่ถูกล็อก (ชั่วโมง) ก่อนลองใหม่ได้อีกครั้ง |
| `AXES_LOCKOUT_PARAMETERS` | เกณฑ์ที่ใช้นับว่า "ผิดกี่ครั้ง" (ตาม IP, username, หรือทั้งคู่) |
| `AXES_RESET_ON_SUCCESS` | ถ้า `True` การ login สำเร็จจะล้างตัวนับความผิดพลาดทันที |
| `AXES_ENABLE_ADMIN` | เปิดให้ดูและปลดล็อกบัญชีผ่าน Django Admin ได้ |

### 348.5 ขอบเขตของ Part นี้และสิ่งที่รอใน Part 083

Part นี้ **เกริ่นให้รู้จักแนวคิดและวิธีติดตั้งพื้นฐาน** เท่านั้น การเจาะลึกเรื่อง
Rate Limiting แบบเต็มรูปแบบ — รวมถึงการทำ rate limiting ระดับ view/endpoint ทั่วไป
ด้วย `django-ratelimit`, การออกแบบระบบ throttling แบบ distributed ด้วย Redis
สำหรับระบบที่มีหลายเซิร์ฟเวอร์, การป้องกัน DDoS ระดับ infrastructure, และการปรับแต่ง
django-axes ขั้นสูง (custom lockout logic, IP allowlist, integration กับ
CAPTCHA) จะอยู่ใน **Part 083: Rate Limiting และ Brute Force Protection**
(ขั้นตอนที่ 821-830) ซึ่งอยู่ในช่วง Phase ความปลอดภัยขั้นสูงของหลักสูตร

---

## ขั้นตอนที่ 349: ตรวจสอบรหัสผ่านที่เคยรั่วไหลด้วย Have I Been Pwned API

### 349.1 ปัญหาที่ Validator ทั่วไปแก้ไม่ได้: การใช้รหัสผ่านซ้ำข้ามเว็บไซต์

รหัสผ่านอย่าง `Summer2024!` ผ่านทุก validator ในขั้นตอนที่ 341-342 ได้สบาย ๆ
(ยาวพอ, มีตัวเลข, มีตัวพิมพ์ใหญ่, ไม่ใช่ตัวเลขล้วน) แต่ถ้ารหัสผ่านนี้เคยหลุดออกมา
จากเว็บไซต์อื่นที่เคยถูกแฮ็กมาก่อน (ซึ่งเกิดขึ้นบ่อยมากในโลกจริง — มีฐานข้อมูล
รหัสผ่านที่รั่วไหลรวมกันหลายพันล้านรายการ) ผู้โจมตีสามารถนำรายชื่อนี้มาทดลอง
"เดา" กับบัญชีในระบบของเราได้ทันที (Credential Stuffing)

**Have I Been Pwned (HIBP)** โดย Troy Hunt เป็นบริการที่รวบรวมรหัสผ่านที่เคยรั่ว
ไหลจริงกว่า **หลายพันล้านรายการ** และเปิด API ให้ตรวจสอบได้ฟรี โดยไม่ต้องขอ
API key สำหรับ endpoint ตรวจสอบรหัสผ่าน (Pwned Passwords)

### 349.2 กลไก k-Anonymity: ตรวจสอบโดยไม่ส่งรหัสผ่านจริงออกไปที่ไหนเลย

จุดที่ฉลาดที่สุดของ API นี้คือการออกแบบด้วยหลักการ **k-Anonymity** เพื่อไม่ให้
ต้องส่งรหัสผ่านจริง (แม้จะ hash แล้ว) ออกไปให้บริการภายนอกรู้ทั้งหมด:

```
1. คำนวณ SHA-1 hash ของรหัสผ่าน (ใช้ SHA-1 เพราะเป็น protocol ที่ HIBP กำหนด
   ไว้สำหรับ endpoint นี้เท่านั้น — ไม่ได้เกี่ยวข้องกับการเก็บรหัสผ่านจริงในระบบ
   ของเราซึ่งยังคงใช้ PBKDF2/Argon2 ตามขั้นตอนที่ 341 เหมือนเดิมทุกประการ)

   เช่น sha1("password123") = CBFDAC6008F9CAB4083784CBD1874F76618D2A97

2. ส่งแค่ 5 ตัวอักษรแรกของ hash ("CBFDA") ไปที่ API
   (ไม่ส่ง hash เต็ม ไม่ส่งรหัสผ่านจริงออกไปนอกเซิร์ฟเวอร์ของเราเลย)

3. API ตอบกลับมาเป็น "รายชื่อ suffix ทั้งหมด" ที่ขึ้นต้นด้วย prefix เดียวกันนี้
   พร้อมจำนวนครั้งที่เคยพบรหัสผ่านนั้นในฐานข้อมูลที่รั่วไหล (อาจได้กลับมาหลาย
   ร้อยถึงหลายพันรายการ)

4. เราเทียบ suffix ที่เหลือ (35 ตัวอักษรที่เหลือของ hash) กับรายชื่อที่ได้มา
   "ด้วยตัวเองในเครื่อง" — ถ้าตรงกัน แปลว่ารหัสผ่านนี้เคยรั่วไหลมาก่อน
```

วิธีนี้ทำให้ HIBP **ไม่มีทางรู้เลยว่าเรากำลังเช็ครหัสผ่านตัวเต็มอะไรอยู่** เพราะมี
รหัสผ่านนับพันตัวที่ hash แล้วขึ้นต้นด้วย prefix เดียวกัน

### 349.3 เขียน Custom Validator เชื่อมต่อ HIBP API

```bash
pip install requests
pip freeze > requirements.txt
```

```python
# accounts/validators.py (เพิ่มต่อจากขั้นตอนที่ 342)
import hashlib
import logging

import requests
from django.core.exceptions import ValidationError
from django.utils.translation import gettext as _

logger = logging.getLogger(__name__)

HIBP_API_URL = "https://api.pwnedpasswords.com/range/{prefix}"


class PwnedPasswordValidator:
    """
    ตรวจสอบว่ารหัสผ่านเคยปรากฏในฐานข้อมูลรหัสผ่านที่รั่วไหลจริง
    (Have I Been Pwned) หรือไม่ โดยใช้กลไก k-Anonymity ที่ไม่ส่งรหัสผ่าน
    จริงออกจากเซิร์ฟเวอร์เลย
    """

    def __init__(self, timeout=2, fail_open=True):
        # timeout สั้น ๆ เพื่อไม่ให้หน้าสมัครสมาชิกช้าถ้า HIBP ตอบช้า
        self.timeout = timeout
        # fail_open=True หมายถึงถ้าเรียก API ไม่สำเร็จ (เน็ตล่ม, timeout, ฯลฯ)
        # จะ "ปล่อยผ่าน" แทนที่จะบล็อกผู้ใช้ไม่ให้สมัครสมาชิกได้เลย
        self.fail_open = fail_open

    def validate(self, password, user=None):
        sha1_hash = hashlib.sha1(password.encode("utf-8")).hexdigest().upper()
        prefix, suffix = sha1_hash[:5], sha1_hash[5:]

        try:
            response = requests.get(
                HIBP_API_URL.format(prefix=prefix),
                headers={"User-Agent": "MyBlog-Django-PwnedPasswordValidator"},
                timeout=self.timeout,
            )
            response.raise_for_status()
        except requests.RequestException as exc:
            logger.warning("ไม่สามารถเชื่อมต่อ HIBP API ได้: %s", exc)
            if self.fail_open:
                return   # ปล่อยผ่านเมื่อ API ใช้งานไม่ได้ ไม่ให้กระทบผู้ใช้ทั่วไป
            raise ValidationError(
                _("ไม่สามารถตรวจสอบความปลอดภัยของรหัสผ่านได้ในขณะนี้ กรุณาลองใหม่อีกครั้ง"),
                code="password_pwned_check_failed",
            )

        # response.text เป็นรายการ "suffix:count" คั่นด้วยขึ้นบรรทัดใหม่
        for line in response.text.splitlines():
            candidate_suffix, count = line.split(":")
            if candidate_suffix == suffix:
                raise ValidationError(
                    _(
                        "รหัสผ่านนี้เคยปรากฏในฐานข้อมูลรหัสผ่านที่รั่วไหลมาแล้ว "
                        "อย่างน้อย %(count)s ครั้ง กรุณาเลือกรหัสผ่านอื่นเพื่อความปลอดภัย"
                    ),
                    code="password_pwned",
                    params={"count": count},
                )

    def get_help_text(self):
        return _("รหัสผ่านของคุณจะถูกตรวจสอบว่าเคยรั่วไหลในเหตุการณ์ข้อมูลรั่วไหลที่รู้จักหรือไม่")
```

### 349.4 ลงทะเบียนและทดสอบ

```python
# config/settings.py
AUTH_PASSWORD_VALIDATORS = [
    # ... validator เดิมทั้งหมดจากขั้นตอนที่ 341-342 ...
    {
        "NAME": "accounts.validators.PwnedPasswordValidator",
    },
]
```

```python
from django.contrib.auth.password_validation import validate_password
from django.core.exceptions import ValidationError

try:
    validate_password("Password123!")   # รหัสผ่านยอดฮิตที่แน่นอนว่าเคยรั่วไหล
except ValidationError as e:
    for msg in e.messages:
        print("-", msg)
```

```
- รหัสผ่านนี้เคยปรากฏในฐานข้อมูลรหัสผ่านที่รั่วไหลมาแล้ว อย่างน้อย 123456 ครั้ง กรุณาเลือกรหัสผ่านอื่นเพื่อความปลอดภัย
```

### 349.5 ข้อควรพิจารณาก่อนใช้งานจริงใน Production

| ประเด็น | คำอธิบาย |
|---|---|
| Latency เพิ่มขึ้นทุกครั้งที่สมัคร/เปลี่ยนรหัสผ่าน | เรียก API ภายนอกทุกครั้งที่ validate ทำให้ช้าลงเล็กน้อย (ตั้ง `timeout` สั้น ๆ ช่วยจำกัดผลกระทบ) |
| จุดเดียวที่อาจล้มเหลว (Single Point of Failure) | ถ้า HIBP ล่มและตั้ง `fail_open=False` ผู้ใช้จะสมัครสมาชิกไม่ได้เลยทั้งระบบ — ควรตั้ง `fail_open=True` เป็นค่าเริ่มต้นเสมอสำหรับระบบทั่วไป |
| ความเป็นส่วนตัว | ส่งแค่ 5 ตัวอักษรแรกของ SHA-1 hash เท่านั้น ไม่ส่งรหัสผ่านจริงหรือ hash เต็มออกไป (ตามหลัก k-Anonymity ในขั้นตอนที่ 349.2) |
| ควรใช้เป็น validator บังคับ หรือแค่คำเตือน | ระบบส่วนใหญ่เลือก **บังคับ** ตอนสมัครสมาชิก/เปลี่ยนรหัสผ่าน (เหมือนที่ทำในขั้นตอนนี้) แต่บางระบบเลือกแค่ **แจ้งเตือน** โดยไม่บล็อก เพื่อไม่ให้กระทบ conversion rate ของหน้าสมัครสมาชิก |
| Rate Limit ของ HIBP เอง | Pwned Passwords API (ต่างจาก Breach API) **ไม่ต้องใช้ API key และไม่มี rate limit อย่างเป็นทางการ** สำหรับการใช้งานทั่วไป แต่ควรใส่ `User-Agent` header เสมอตามที่ HIBP กำหนด |

---

## ขั้นตอนที่ 350: สรุปและแบบฝึกหัด

### 350.1 สรุปสิ่งที่ได้เรียนรู้ใน Part นี้

- เข้าใจกลไก hashing รหัสผ่านของ Django อย่างละเอียด ตั้งแต่โครงสร้างค่าที่เก็บ
  ในฟิลด์ `password`, อัลกอริทึม PBKDF2 ที่เป็นค่าเริ่มต้น, `PASSWORD_HASHERS`,
  และการอัปเกรด hash แบบเงียบ ๆ อัตโนมัติเมื่อเปลี่ยน hasher
- รู้จักและเข้าใจ `AUTH_PASSWORD_VALIDATORS` ทั้ง 4 ตัวที่มากับ Django อย่างลึกซึ้ง
  (`UserAttributeSimilarityValidator`, `MinimumLengthValidator`,
  `CommonPasswordValidator`, `NumericPasswordValidator`)
- เขียน Custom Password Validator ของตัวเองได้ตาม interface contract ที่ Django
  กำหนด พร้อมเขียน unit test คลุม logic ด้านความปลอดภัย
- สร้างระบบเปลี่ยนรหัสผ่านตอน login อยู่ด้วย `PasswordChangeView` และเข้าใจกลไก
  `update_session_auth_hash()`/`get_session_auth_hash()` ที่เชื่อมโยงกับระบบ
  Session จาก Part 034
- สร้าง Password Reset Flow เต็มรูปแบบด้วย built-in views ครบทั้ง 4 ขั้นตอน
  พร้อม template ทั้ง 6 ไฟล์ และเข้าใจความปลอดภัยของ `PasswordResetTokenGenerator`
- ทำ Custom Email Template ทั้งแบบ HTML และ plain text สำหรับ password reset
  ด้วย `EmailMultiAlternatives`
- ใช้ `set_password()`/`check_password()` แบบ manual ในสถานการณ์ที่ไม่ผ่านฟอร์ม
  เช่น management command และการยืนยันตัวตนซ้ำก่อน action สำคัญ
- รู้จัก Password Strength Meter ด้วย zxcvbn เพื่อ UX ที่ดีขึ้น พร้อมเข้าใจว่า
  client-side validation ไม่ใช่เกราะป้องกันจริง
- รู้จักภาพรวมของ django-axes สำหรับป้องกัน Brute Force (เจาะลึกเต็มรูปแบบที่
  Part 083) และการเช็ครหัสผ่านที่เคยรั่วไหลผ่าน Have I Been Pwned API

### 350.2 Checklist ก่อนไป Part ถัดไป

- [ ] อธิบายโครงสร้าง `algorithm$iterations$salt$hash` ในฟิลด์ `password` ได้
- [ ] อธิบายได้ว่าทำไมต้อง hash แบบ "ช้าโดยตั้งใจ" ต่างจาก MD5/SHA256 ทั่วไป
- [ ] ตั้งค่า `PASSWORD_HASHERS` ให้ใช้ Argon2 เป็นตัวแรกได้
- [ ] อธิบาย validator ทั้ง 4 ตัวของ Django ได้ว่าแต่ละตัวเช็คอะไร
- [ ] เขียน Custom Password Validator ของตัวเองได้ พร้อม unit test
- [ ] ทำระบบเปลี่ยนรหัสผ่านที่ไม่ logout ผู้ใช้ปัจจุบันโดยไม่ตั้งใจได้
- [ ] อธิบายว่าทำไมเปลี่ยนรหัสผ่านแล้ว session อื่น ๆ ถูก logout อัตโนมัติได้
- [ ] ทำ Password Reset Flow เต็มรูปแบบสำเร็จ ตั้งแต่กรอกอีเมลจนถึงตั้งรหัสผ่านใหม่
- [ ] ทำ custom email template ทั้ง HTML และ plain text สำหรับ password reset ได้
- [ ] ใช้ `set_password()`/`check_password()` แบบ manual ได้อย่างถูกต้องและปลอดภัย

### 350.3 แบบฝึกหัดท้ายบท

**แบบฝึกหัดที่ 1**: implement Password Reset Flow เต็มรูปแบบให้กับโปรเจกต์
`blog` ของคุณตามขั้นตอนที่ 344-345 ทั้งหมด (URL, template ทั้ง 6 ไฟล์, custom
HTML email) แล้วทดสอบ end-to-end ด้วย console email backend ตามลำดับ: สมัคร
สมาชิกใหม่ → ขอรีเซ็ตรหัสผ่าน → คัดลอกลิงก์จาก terminal → ตั้งรหัสผ่านใหม่ →
login ด้วยรหัสผ่านใหม่สำเร็จ → ลองใช้ลิงก์เดิมซ้ำแล้วยืนยันว่าขึ้น "ลิงก์ไม่ถูก
ต้องหรือหมดอายุแล้ว"

**แบบฝึกหัดที่ 2**: เขียน Custom Password Validator ชื่อ `NoRepeatingCharacterValidator`
ที่ปฏิเสธรหัสผ่านที่มีตัวอักษรซ้ำติดกันเกิน 3 ตัว (เช่น `aaaa1234` หรือ
`Password1111` ต้องถูกบล็อก แต่ `Password11` ผ่านได้) พร้อมเขียน `get_help_text()`
ที่อธิบายกฎนี้เป็นภาษาไทย และเขียน unit test อย่างน้อย 3 test case (ผ่าน, ไม่ผ่าน
เพราะซ้ำที่ต้น, ไม่ผ่านเพราะซ้ำที่กลางคำ)

**แบบฝึกหัดที่ 3**: เพิ่ม Password Strength Meter ด้วย zxcvbn ตามขั้นตอนที่ 347
ให้กับทั้งหน้า signup **และ** หน้า password change แล้วปรับให้ปุ่ม submit ถูก
`disabled` ไว้ตราบใดที่ `zxcvbn(password).score` น้อยกว่า 2 (บังคับ UX ให้ผู้ใช้
เห็นคำแนะนำก่อนจะกด submit ได้ แม้ server-side validator จะเป็นด่านสุดท้ายที่แท้จริง
อยู่ดีก็ตาม)

**แบบฝึกหัดที่ 4 (ขั้นสูง)**: เลือกทำอย่างใดอย่างหนึ่งต่อไปนี้ (1) ติดตั้งและตั้งค่า
django-axes ตามขั้นตอนที่ 348 แล้วทดสอบด้วยการพยายาม login ผิดติดต่อกันเกิน
`AXES_FAILURE_LIMIT` ครั้ง บันทึกผลว่าระบบล็อกบัญชีจริงหรือไม่ และปลดล็อกผ่าน
Django Admin ได้หรือไม่ หรือ (2) implement `PwnedPasswordValidator` ตามขั้นตอนที่
349 แบบเต็มรูปแบบ แล้วทดสอบด้วยรหัสผ่านที่แน่ใจว่าเคยรั่วไหล (เช่น `123456789`)
เทียบกับรหัสผ่านที่สุ่มสร้างขึ้นใหม่แบบไม่ซ้ำใคร แล้วบันทึกเวลาที่ใช้ในการเรียก API
แต่ละครั้งเพื่อประเมินผลกระทบต่อ UX

### 350.4 คำถามที่พบบ่อย (FAQ)

**Q: ถ้าเปลี่ยน `PASSWORD_HASHERS` ให้ใช้ Argon2 เป็นตัวแรก ผู้ใช้เก่าที่ hash
ด้วย PBKDF2 จะ login ไม่ได้เลยหรือไม่?**
A: ไม่ ผู้ใช้เก่ายัง login ได้ตามปกติทันที เพราะ `PBKDF2PasswordHasher` ยังคงอยู่
ในรายการ `PASSWORD_HASHERS` (แค่ไม่ใช่ตัวแรก) Django จึงยังตรวจสอบ hash แบบเก่าได้
และจะอัปเกรดเป็น Argon2 ให้อัตโนมัติแบบเงียบ ๆ ทันทีที่ login สำเร็จครั้งถัดไป
ตามกลไกในขั้นตอนที่ 341.6 — ห้ามลบ hasher ตัวเก่าออกจากรายการโดยเด็ดขาดถ้ายังมี
ผู้ใช้ที่ hash ด้วยอัลกอริทึมนั้นอยู่ในระบบ

**Q: ทำไม `PasswordResetView` ไม่บอกตรง ๆ ว่า "อีเมลนี้ไม่มีในระบบ" เมื่อกรอกอีเมล
ที่ไม่มีอยู่จริง?**
A: เป็นการออกแบบเพื่อความปลอดภัยเช่นเดียวกับที่ Part 031 ขั้นตอนที่ 310.13 อธิบาย
เรื่อง `authenticate()` — ถ้าบอกตรง ๆ ว่า "อีเมลนี้ไม่มีในระบบ" ผู้โจมตีจะสามารถ
ใช้ฟอร์มนี้เป็นเครื่องมือตรวจสอบว่าอีเมลไหน "มีบัญชีอยู่จริง" ในระบบของคุณได้
(User Enumeration Attack) `PasswordResetView` จึงแสดงหน้า `password_reset_done`
เหมือนกันเสมอไม่ว่าอีเมลนั้นจะมีอยู่จริงหรือไม่ (ถ้ามีอยู่จริง ระบบจะส่งอีเมลออก
เบื้องหลัง ถ้าไม่มีอยู่จริง ระบบจะไม่ทำอะไรเลยแต่ยังคง redirect ไปหน้าเดียวกัน)

**Q: ควรใช้ `MinimumLengthValidator` กำหนดความยาวเท่าไหร่ถึงจะเหมาะสมในปี 2026?**
A: NIST SP 800-63B แนะนำอย่างน้อย 8 ตัวอักษรเป็นขั้นต่ำ แต่แนวโน้มของอุตสาหกรรม
ในปี 2026 นิยมตั้งไว้ที่ **10-12 ตัวอักษร** ร่วมกับการเลิกบังคับเปลี่ยนรหัสผ่าน
ตามระยะเวลา (periodic password rotation) เพราะงานวิจัยพบว่าการบังคับเปลี่ยนบ่อย
เกินไปทำให้ผู้ใช้ตั้งรหัสผ่านที่คาดเดาง่ายขึ้น (เช่น เปลี่ยนจาก `Pass2025!` เป็น
`Pass2026!`) หลักการที่สำคัญกว่าคือความยาวที่เพียงพอ ผสมกับการเช็ครหัสผ่านที่
รั่วไหลจริงตามขั้นตอนที่ 349 มากกว่าการบังคับกฎความซับซ้อนที่เข้มงวดเกินไป

**Q: จำเป็นต้องใช้ทั้ง django-axes และ Have I Been Pwned Validator พร้อมกันหรือไม่?**
A: ทั้งสองแก้ปัญหาคนละมุมและ**เสริมกันได้ดีมาก** ไม่ใช่ทางเลือกที่ต้องเลือกอย่าง
ใดอย่างหนึ่ง — `PwnedPasswordValidator` ป้องกันไม่ให้ผู้ใช้ "ตั้ง" รหัสผ่านที่
อ่อนแอตั้งแต่ต้น (เชิงป้องกัน/preventive) ส่วน django-axes ป้องกันไม่ให้ผู้โจมตี
"เดา" รหัสผ่านซ้ำ ๆ ไม่จำกัดครั้ง (เชิงตรวจจับ/detective) ระบบระดับ production
จริงส่วนใหญ่ใช้ทั้งคู่ร่วมกันเป็นเกราะป้องกันหลายชั้น (Defense in Depth)

**Q: ทำไมไฟล์ `password_reset_email.html` ถึงลงท้ายด้วย `.html` ทั้งที่เนื้อหา
เป็น plain text?**
A: เป็น naming convention เก่าแก่ของ Django ที่คงไว้เพื่อ backward compatibility
กับโปรเจกต์เก่าจำนวนมาก ในทางเทคนิคใช้นามสกุลอะไรก็ได้ตราบใดที่ตรงกับค่า
`email_template_name` ที่ตั้งไว้ใน view (บางทีมเปลี่ยนไปใช้ `.txt` แทนเพื่อความ
ชัดเจนกว่า) สิ่งสำคัญคือต้องจำไว้ว่า `PasswordResetForm.send_mail()` render
ไฟล์นี้เป็น plain text เสมอไม่ว่าจะตั้งชื่อไฟล์อย่างไร ส่วน HTML จริงต้องระบุแยก
ผ่าน `html_email_template_name` ตามขั้นตอนที่ 345.5 เท่านั้น

---

## เตรียมตัวสำหรับ Part ถัดไป

**Part 036: Social Authentication (OAuth, django-allauth)** จะพาคุณไปเจาะลึก
การ login ผ่านบัญชีโซเชียล เช่น Google และ GitHub อย่างเต็มรูปแบบ ต่อยอดจากที่
Part 031 ขั้นตอนที่ 309 เกริ่นภาพรวม django-allauth ไว้แบบผิวเผิน คุณจะได้เรียนรู้
หลักการทำงานของ **OAuth 2.0** ตั้งแต่ authorization code flow, การขอ Client ID/
Client Secret จากผู้ให้บริการแต่ละราย, การติดตั้งและตั้งค่า django-allauth แบบ
เต็มรูปแบบ, การเชื่อมบัญชีโซเชียลเข้ากับ `CustomUser` ที่สร้างไว้ตั้งแต่ Part 032,
ไปจนถึงการจัดการกรณีที่ผู้ใช้คนเดียวกัน login ผ่านทั้งอีเมล/รหัสผ่านและโซเชียล
พร้อมกัน (account linking)

เนื่องจาก Social Authentication ยังคงต้องพึ่งพาระบบรหัสผ่านสำหรับผู้ใช้ที่เลือก
สมัครแบบดั้งเดิม (ไม่ผ่านโซเชียล) ทุกสิ่งที่เรียนไปใน Part นี้ — โดยเฉพาะ
`AUTH_PASSWORD_VALIDATORS`, Password Reset Flow, และ Custom Validator — จะยังคง
ใช้งานควบคู่กันไปเสมอ เตรียมทบทวนเรื่อง Custom User Model จาก Part 032 ให้แม่น
เพราะ Part ถัดไปจะเชื่อมโยง `CustomUser` เข้ากับข้อมูลจากผู้ให้บริการ OAuth
โดยตรง
