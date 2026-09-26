# Part 058: File Upload, Image Processing และ Media Handling

> **ขั้นตอนที่ 571-580 ของหลักสูตร** | Phase 6: Frontend Integration (Part นี้เป็น Part สุดท้ายของ Phase)
>
> เป้าหมายของ Part นี้: ต่อยอดจาก `ImageField`/`FileField` เบื้องต้นที่ Part 009 เกริ่นไว้
> (ขั้นตอนที่ 85) ให้ลึกถึงระดับใช้งานจริงในโปรดักชัน คุณจะเขียน Validator ตรวจสอบขนาดและ
> ประเภทไฟล์อย่างเข้มงวด ใช้ **Pillow** สร้าง thumbnail และ resize รูปภาพอัตโนมัติตอนอัปโหลด
> ติดตั้ง **django-imagekit** เพื่อสร้างภาพหลายขนาด (thumbnail/medium/large) แบบ lazy โดยไม่
> ต้องเขียนโค้ด resize เอง จัดการไฟล์ขนาดใหญ่ด้วย chunked upload พร้อม progress bar,
> ทำ **Direct-to-S3 Upload** ด้วย Presigned URL เพื่อลดภาระ Django server, เกริ่นแนวคิดการ
> อัปโหลดวิดีโอและ transcoding, เรียนรู้ช่องโหว่ความปลอดภัยของ file upload ที่มือใหม่มักพลาด
> (เชื่อถือแค่นามสกุลไฟล์), สร้าง UI **Drag-and-drop Upload** ด้วย JavaScript ต่อยอดจาก
> Fetch API ที่เรียนใน Part 052 และปิดท้ายด้วย management command ทำความสะอาดไฟล์ media
> ที่ไม่มีใครอ้างอิงแล้ว เมื่อจบ Part นี้คุณจะปิด **Phase 6: Frontend Integration** ทั้งหมด
> (Part 051-058) อย่างสมบูรณ์ พร้อมสรุปทบทวน Bootstrap, JavaScript, HTMX, Alpine.js, React,
> Vue.js, Django Channels และ Media Handling ก่อนก้าวสู่ **Phase 7: Testing & Quality
> Assurance** ใน Part 059

---

## สารบัญของ Part นี้

- ขั้นตอนที่ 571: ทบทวน FileField/ImageField จาก Part 009 แล้วเจาะลึก Validation (จำกัดขนาดไฟล์, ประเภทไฟล์)
- ขั้นตอนที่ 572: ใช้ Pillow ประมวลผลรูปภาพ — สร้าง thumbnail, resize รูปตอนอัปโหลด (override `save()`)
- ขั้นตอนที่ 573: `django-imagekit` สำหรับสร้าง image variant หลายขนาดอัตโนมัติ (thumbnail, medium, large)
- ขั้นตอนที่ 574: จัดการไฟล์ขนาดใหญ่ — chunked upload, แสดง progress bar ด้วย JavaScript
- ขั้นตอนที่ 575: Direct-to-S3 Upload ด้วย Presigned URL (browser อัปโหลดตรงไป S3 โดยไม่ผ่าน Django server)
- ขั้นตอนที่ 576: ภาพรวมการอัปโหลดวิดีโอ (แนวคิด transcoding เกริ่นสั้น ๆ)
- ขั้นตอนที่ 577: ความปลอดภัยของ File Upload — validate content type จริง, เกริ่น virus scanning
- ขั้นตอนที่ 578: สร้าง UI Drag-and-drop Upload ด้วย JavaScript (เชื่อมกับ Part 052)
- ขั้นตอนที่ 579: จัดการไฟล์ media ที่ไม่มีใครอ้างอิงแล้ว (orphaned files) ด้วย management command
- ขั้นตอนที่ 580: สรุป Phase 6 ทั้งหมด (Part 051-058), quiz, แบบฝึกหัดใหญ่ และคำนำสู่ Phase 7

---

## ขั้นตอนที่ 571: ทบทวน FileField/ImageField แล้วเจาะลึก Validation

### 571.1 ทบทวนสิ่งที่ Part 009 วางรากฐานไว้

Part 009 ขั้นตอนที่ 84-85 สอนพื้นฐานของระบบไฟล์ใน Django ไว้ครบแล้ว:

| แนวคิดจาก Part 009 | สรุปสั้น |
|---|---|
| `MEDIA_URL` / `MEDIA_ROOT` (84.2) | URL prefix กับโฟลเดอร์จริงบนดิสก์สำหรับไฟล์ที่ผู้ใช้อัปโหลด |
| `models.FileField` / `models.ImageField` (85.1) | Field สำหรับรับไฟล์ทั่วไป / รูปภาพ (ต้องมี Pillow) |
| `upload_to="blog/covers/"` (85.2) | โฟลเดอร์ย่อยใต้ `MEDIA_ROOT` ที่เก็บไฟล์ |
| `{{ post.cover_image.url }}` (85.3) | เข้าถึง URL ของไฟล์ที่อัปโหลดแล้วใน template |
| `static()` สำหรับ dev, Nginx/S3 สำหรับ production (86) | วิธี serve media file แต่ละสภาพแวดล้อม |

Part 009 บอกไว้ชัดเจนว่า **รายละเอียดเชิงลึกจะมาที่ Part 058** — นี่คือ Part นั้น เราจะไม่
สอนพื้นฐานซ้ำอีก แต่จะเจาะลึกสิ่งที่ Part 009 "แค่เกริ่น" ให้ครบทุกมิติของการใช้งานจริง

### 571.2 ทำไม Validation ของไฟล์ถึงสำคัญกว่าที่คิด

Model field ธรรมดาอย่าง `ImageField(blank=True, null=True)` **ไม่ได้จำกัดขนาดไฟล์เลย**
และตรวจสอบแค่ว่า "เปิดเป็นรูปภาพได้หรือไม่" เท่านั้น ถ้าไม่เพิ่ม validator เอง ผู้ใช้จะ:

- อัปโหลดรูปภาพขนาด 50 MB ได้โดยไม่มีการเตือน (เปลืองพื้นที่ดิสก์ และทำให้เพจโหลดช้ามาก)
- อัปโหลดไฟล์ `.gif` แม้ระบบต้องการแค่ `.jpg`/`.png`/`.webp`
- ตั้งชื่อไฟล์เป็นภาษาที่มีอักขระพิเศษ ทำให้เกิดปัญหา path บนบาง filesystem

เราจะแก้ทั้งสามปัญหานี้ด้วย **Django Validators** ซึ่งเป็นกลไกมาตรฐานที่ทำงานทั้งกับ
Model field และ Form field โดยอัตโนมัติ (ทบทวนแนวคิด validator จาก Part 018/029 เรื่อง
Form validation หากจำไม่ได้)

### 571.3 สร้างไฟล์ Validator กลางของโปรเจกต์

```python
# blog/validators.py
import os

from django.core.exceptions import ValidationError
from django.template.defaultfilters import filesizeformat


def validate_image_file_size(value):
    """จำกัดขนาดไฟล์รูปภาพไม่เกิน 5 MB"""
    max_size_bytes = 5 * 1024 * 1024  # 5 MB

    if value.size > max_size_bytes:
        raise ValidationError(
            "ไฟล์มีขนาด %(size)s ซึ่งเกินขนาดสูงสุดที่อนุญาต (%(max_size)s)",
            params={
                "size": filesizeformat(value.size),
                "max_size": filesizeformat(max_size_bytes),
            },
        )


def validate_attachment_file_size(value):
    """จำกัดขนาดไฟล์แนบทั่วไปไม่เกิน 20 MB"""
    max_size_bytes = 20 * 1024 * 1024  # 20 MB

    if value.size > max_size_bytes:
        raise ValidationError(
            "ไฟล์มีขนาด %(size)s ซึ่งเกินขนาดสูงสุดที่อนุญาต (%(max_size)s)",
            params={
                "size": filesizeformat(value.size),
                "max_size": filesizeformat(max_size_bytes),
            },
        )


ALLOWED_IMAGE_EXTENSIONS = [".jpg", ".jpeg", ".png", ".webp"]
ALLOWED_ATTACHMENT_EXTENSIONS = [".pdf", ".zip", ".docx", ".xlsx"]


def validate_image_extension(value):
    """ตรวจสอบนามสกุลไฟล์ (การตรวจสอบชั้นที่ 1 เท่านั้น — ยังไม่ปลอดภัย 100%
    ดูขั้นตอนที่ 577 สำหรับการตรวจสอบเนื้อหาไฟล์จริงที่ปลอดภัยกว่านี้)"""
    ext = os.path.splitext(value.name)[1].lower()

    if ext not in ALLOWED_IMAGE_EXTENSIONS:
        raise ValidationError(
            "นามสกุลไฟล์ '%(ext)s' ไม่ได้รับอนุญาต ใช้ได้เฉพาะ: %(allowed)s",
            params={"ext": ext, "allowed": ", ".join(ALLOWED_IMAGE_EXTENSIONS)},
        )


def validate_attachment_extension(value):
    ext = os.path.splitext(value.name)[1].lower()

    if ext not in ALLOWED_ATTACHMENT_EXTENSIONS:
        raise ValidationError(
            "นามสกุลไฟล์ '%(ext)s' ไม่ได้รับอนุญาต ใช้ได้เฉพาะ: %(allowed)s",
            params={"ext": ext, "allowed": ", ".join(ALLOWED_ATTACHMENT_EXTENSIONS)},
        )
```

### 571.4 นำ Validator ไปใช้ใน Model

```python
# blog/models.py
from django.db import models

from .validators import (
    validate_attachment_extension,
    validate_attachment_file_size,
    validate_image_extension,
    validate_image_file_size,
)


class Post(models.Model):
    title = models.CharField(max_length=200)
    slug = models.SlugField(unique=True)
    content = models.TextField()
    cover_image = models.ImageField(
        upload_to="blog/covers/%Y/%m/",
        blank=True,
        null=True,
        validators=[validate_image_file_size, validate_image_extension],
        help_text="ไฟล์ .jpg, .jpeg, .png, .webp ขนาดไม่เกิน 5 MB",
    )
    attachment = models.FileField(
        upload_to="blog/attachments/%Y/%m/",
        blank=True,
        null=True,
        validators=[validate_attachment_file_size, validate_attachment_extension],
        help_text="ไฟล์ .pdf, .zip, .docx, .xlsx ขนาดไม่เกิน 20 MB",
    )
    created_at = models.DateTimeField(auto_now_add=True)

    def __str__(self):
        return self.title
```

สังเกต `upload_to="blog/covers/%Y/%m/"` — Django รองรับ `strftime` pattern ใน `upload_to`
โดยตรง ทำให้ไฟล์ถูกจัดกลุ่มตามปี/เดือนอัตโนมัติ (เช่น `blog/covers/2026/09/photo.jpg`)
ป้องกันปัญหาโฟลเดอร์เดียวมีไฟล์เป็นแสนไฟล์จนระบบไฟล์ช้าลงเมื่อเว็บไซต์โตขึ้น

**สำคัญ**: validator ที่ผูกกับ Model field จะทำงานก็ต่อเมื่อเรียก `full_clean()` เท่านั้น
(เช่นผ่าน `ModelForm` ที่เรียก `is_valid()` ให้อัตโนมัติ, Part 018) **การเรียก `.save()`
โดยตรงจะไม่รัน validator เหล่านี้** — นี่คือกับดักที่มือใหม่พลาดบ่อยมาก:

```python
# ตัวอย่างที่ผิด: validator จะไม่ถูกเรียกเลย
post = Post(title="ทดสอบ", cover_image=oversized_file)
post.save()   # ❌ ผ่านตลอด แม้ไฟล์จะใหญ่เกิน 5 MB

# ตัวอย่างที่ถูกต้อง: บังคับรัน validator ก่อน save
post = Post(title="ทดสอบ", cover_image=oversized_file)
post.full_clean()   # ✅ raise ValidationError ถ้าไฟล์ไม่ผ่าน
post.save()
```

### 571.5 นำ Validator ไปใช้ใน ModelForm (เส้นทางที่ใช้จริงเกือบทุกกรณี)

```python
# blog/forms.py
from django import forms

from .models import Post


class PostForm(forms.ModelForm):
    class Meta:
        model = Post
        fields = ["title", "slug", "content", "cover_image", "attachment"]
```

เพราะ `PostForm.is_valid()` เรียก `full_clean()` ให้อัตโนมัติเสมออยู่แล้ว (ทบทวนจาก
Part 018) validator ทั้งสี่ตัวที่ผูกไว้กับ field ใน `models.py` จะทำงานทันทีโดยไม่ต้องเขียน
อะไรเพิ่มใน `forms.py` เลย — นี่คือพลังของหลักการ DRY ที่ Django ยึดถือมาตั้งแต่ Part 001

### 571.6 ตั้งค่าระดับ Request ที่เกี่ยวกับการอัปโหลดไฟล์

นอกจาก validator ระดับ field แล้ว Django ยังมี setting ระดับ HTTP request ที่ควบคุมพฤติกรรม
การอัปโหลดไฟล์ทั้งระบบ:

```python
# config/settings.py

# ไฟล์ที่เล็กกว่าค่านี้ (bytes) จะถูกเก็บไว้ใน RAM ระหว่างประมวลผล request
# ไฟล์ที่ใหญ่กว่าจะถูกเขียนลง temp file บนดิสก์แทน (ป้องกัน RAM เต็ม)
FILE_UPLOAD_MAX_MEMORY_SIZE = 2621440  # 2.5 MB (ค่าเริ่มต้นของ Django)

# จำกัดขนาดรวมของ non-file POST data (form fields ทั่วไป ไม่รวมไฟล์)
DATA_UPLOAD_MAX_MEMORY_SIZE = 2621440  # 2.5 MB (ค่าเริ่มต้น)

# permission ของไฟล์ที่ถูกสร้างใหม่บนดิสก์ (Unix เท่านั้น)
FILE_UPLOAD_PERMISSIONS = 0o644

# permission ของโฟลเดอร์ย่อยที่ถูกสร้างใหม่ใต้ MEDIA_ROOT
FILE_UPLOAD_DIRECTORY_PERMISSIONS = 0o755
```

| Setting | ความหมาย | เมื่อไหร่ต้องปรับ |
|---|---|---|
| `FILE_UPLOAD_MAX_MEMORY_SIZE` | เกณฑ์สลับจาก in-memory handler ไป temp-file handler | ปรับสูงขึ้นถ้า server มี RAM เยอะและอยากลด disk I/O สำหรับไฟล์ขนาดกลาง |
| `DATA_UPLOAD_MAX_MEMORY_SIZE` | จำกัดขนาด POST body ที่ไม่ใช่ไฟล์ (ป้องกัน DoS จาก form data ขนาดมหึมา) | เพิ่มถ้ามีฟอร์มที่ต้องส่ง field จำนวนมากจริง ๆ (เช่น bulk import) |
| `FILE_UPLOAD_PERMISSIONS` | สิทธิ์ของไฟล์ที่สร้างใหม่ | ตั้งเป็น `0o644` เสมอเพื่อไม่ให้ไฟล์ media รันเป็น executable ได้ |

**ข้อควรระวังสำคัญ**: `DATA_UPLOAD_MAX_MEMORY_SIZE` **ไม่ได้จำกัดขนาดไฟล์ที่อัปโหลด**
(ไฟล์แยกไปใช้ `FILE_UPLOAD_MAX_MEMORY_SIZE` เป็นเกณฑ์เก็บชั่วคราวเท่านั้น ไม่ใช่ขีดจำกัด) การ
จำกัดขนาดไฟล์สูงสุดจริง ๆ ต้องทำผ่าน validator ตามขั้นตอนที่ 571.3 หรือตั้งค่าที่ระดับเว็บ
เซิร์ฟเวอร์ (เช่น `client_max_body_size` ใน Nginx) ควบคู่กันเสมอ เพราะ validator ของ Django
ทำงาน**หลังจาก**ไฟล์ถูกอัปโหลดเข้ามาเต็มจำนวนแล้ว — ถ้าไม่จำกัดที่ระดับเว็บเซิร์ฟเวอร์ด้วย
ผู้ใช้ที่ประสงค์ร้ายยังสามารถส่งไฟล์ขนาดหลาย GB มาถล่ม bandwidth ได้ก่อนที่ Django จะทัน
ปฏิเสธ

---

## ขั้นตอนที่ 572: ใช้ Pillow ประมวลผลรูปภาพ — สร้าง thumbnail, resize รูปตอนอัปโหลด

### 572.1 ปัญหาที่ต้องแก้: รูปต้นฉบับมักใหญ่เกินความจำเป็น

ผู้ใช้ทั่วไปอัปโหลดรูปจากมือถือที่มีความละเอียด 4000×3000 พิกเซล (12 ล้านพิกเซล ขนาดไฟล์
หลาย MB) ทั้งที่หน้าเว็บแสดงผลจริงแค่ 800×600 พิกเซล การส่งไฟล์ต้นฉบับตรง ๆ ไปให้ browser
ทำให้:

- หน้าเว็บโหลดช้าโดยไม่จำเป็น (ดาวน์โหลดพิกเซลที่ไม่มีทางแสดงผลจริง)
- เปลืองพื้นที่ storage และ bandwidth มหาศาลเมื่อมีผู้ใช้จำนวนมาก
- CPU ฝั่ง browser ต้อง downscale รูปใหญ่ทุกครั้งที่แสดงผล (แม้จะเป็นงานเบา แต่สะสมได้)

วิธีแก้มาตรฐานคือ **resize รูปภาพให้เหมาะสมตอนอัปโหลด (server-side)** แทนที่จะพึ่ง CSS
`width`/`height` ที่แค่บีบการแสดงผลแต่ไฟล์จริงยังใหญ่เท่าเดิม

### 572.2 ทบทวน Pillow และติดตั้งเพิ่มเติมถ้าจำเป็น

Pillow ถูกติดตั้งไปแล้วตั้งแต่ Part 009 (ขั้นตอนที่ 85.1) เพราะ `ImageField` ต้องพึ่ง Pillow
ตรวจสอบไฟล์รูปภาพอยู่แล้ว ตรวจสอบเวอร์ชันที่ติดตั้ง:

```bash
python -c "import PIL; print(PIL.__version__)"
```

### 572.3 Override `save()` เพื่อสร้าง Thumbnail อัตโนมัติ

แนวทางที่ตรงไปตรงมาที่สุดคือ override เมธอด `save()` ของ Model ให้ประมวลผลรูปภาพ
**หลังจากที่ Django บันทึกไฟล์ต้นฉบับเสร็จแล้ว**:

```python
# blog/models.py
import io

from django.core.files.base import ContentFile
from django.db import models
from PIL import Image

from .validators import (
    validate_attachment_extension,
    validate_attachment_file_size,
    validate_image_extension,
    validate_image_file_size,
)


class Post(models.Model):
    title = models.CharField(max_length=200)
    slug = models.SlugField(unique=True)
    content = models.TextField()
    cover_image = models.ImageField(
        upload_to="blog/covers/%Y/%m/",
        blank=True,
        null=True,
        validators=[validate_image_file_size, validate_image_extension],
    )
    cover_image_thumbnail = models.ImageField(
        upload_to="blog/covers/thumbnails/%Y/%m/",
        blank=True,
        null=True,
        editable=False,   # ผู้ใช้ไม่ต้องอัปโหลดเอง — ระบบสร้างให้อัตโนมัติ
    )
    attachment = models.FileField(
        upload_to="blog/attachments/%Y/%m/",
        blank=True,
        null=True,
        validators=[validate_attachment_file_size, validate_attachment_extension],
    )
    created_at = models.DateTimeField(auto_now_add=True)

    THUMBNAIL_SIZE = (400, 400)
    MAX_DIMENSION = 1920   # จำกัดด้านที่ยาวที่สุดของรูปต้นฉบับหลัง resize

    def __str__(self):
        return self.title

    def save(self, *args, **kwargs):
        is_new_image = False

        # ตรวจสอบว่ามีการอัปโหลดรูปใหม่หรือไม่ (เทียบกับค่าที่เคยบันทึกในฐานข้อมูล)
        if self.pk:
            old_instance = Post.objects.filter(pk=self.pk).first()
            if old_instance and old_instance.cover_image != self.cover_image:
                is_new_image = True
        elif self.cover_image:
            is_new_image = True

        super().save(*args, **kwargs)

        if is_new_image and self.cover_image:
            self._resize_cover_image()
            self._create_thumbnail()
            # บันทึกอีกครั้งเพื่ออัปเดต field cover_image (resize แล้ว) และ
            # cover_image_thumbnail — ใช้ update_fields ป้องกัน infinite loop
            # และลดงานเขียนฐานข้อมูลเฉพาะ field ที่เปลี่ยนจริง
            super().save(update_fields=["cover_image", "cover_image_thumbnail"])

    def _resize_cover_image(self):
        """ลดขนาดรูปต้นฉบับถ้าด้านที่ยาวที่สุดเกิน MAX_DIMENSION"""
        img = Image.open(self.cover_image)

        if max(img.width, img.height) <= self.MAX_DIMENSION:
            return   # รูปเล็กพออยู่แล้ว ไม่ต้อง resize

        img.thumbnail((self.MAX_DIMENSION, self.MAX_DIMENSION), Image.Resampling.LANCZOS)

        buffer = io.BytesIO()
        img_format = "JPEG" if img.mode != "RGBA" else "PNG"
        img.convert("RGB").save(buffer, format=img_format, quality=85, optimize=True)

        file_name = self.cover_image.name.rsplit("/", 1)[-1]
        self.cover_image.save(
            file_name, ContentFile(buffer.getvalue()), save=False
        )

    def _create_thumbnail(self):
        """สร้างรูป thumbnail ขนาดคงที่ (crop กึ่งกลางให้เป็นสี่เหลี่ยมจัตุรัส)"""
        img = Image.open(self.cover_image)
        img = img.convert("RGB")

        # crop ให้เป็นสัดส่วนสี่เหลี่ยมจัตุรัสก่อน แล้วค่อย resize (ป้องกันภาพยืด/บิดเบี้ยว)
        width, height = img.size
        side = min(width, height)
        left = (width - side) // 2
        top = (height - side) // 2
        img = img.crop((left, top, left + side, top + side))
        img.thumbnail(self.THUMBNAIL_SIZE, Image.Resampling.LANCZOS)

        buffer = io.BytesIO()
        img.save(buffer, format="JPEG", quality=80, optimize=True)

        file_name = self.cover_image.name.rsplit("/", 1)[-1]
        thumb_name = f"thumb_{file_name.rsplit('.', 1)[0]}.jpg"
        self.cover_image_thumbnail.save(
            thumb_name, ContentFile(buffer.getvalue()), save=False
        )
```

### 572.2 อธิบายจุดสำคัญของโค้ด

| จุดสำคัญ | เหตุผล |
|---|---|
| เช็ค `is_new_image` ก่อนประมวลผล | ป้องกันการ resize ซ้ำทุกครั้งที่ `save()` ถูกเรียก (เช่นตอนแก้ไข title เฉยๆ โดยไม่แตะรูป) |
| `save=False` ใน `self.cover_image.save(...)` | ป้องกัน infinite recursion — ถ้าใส่ `save=True` จะเรียก `Post.save()` ของเราซ้ำอีกรอบไม่รู้จบ |
| `super().save(update_fields=[...])` รอบสอง | เขียนแค่ field ที่เปลี่ยนจริง เร็วกว่าการ `UPDATE` ทุกคอลัมน์ |
| `Image.Resampling.LANCZOS` | อัลกอริทึม resize คุณภาพสูงสุดของ Pillow เหมาะกับการลดขนาดภาพ (ช้ากว่า `NEAREST` แต่ภาพคมกว่ามาก) |
| `img.convert("RGB")` ก่อน save เป็น JPEG | JPEG ไม่รองรับ alpha channel (RGBA) ถ้าไม่ convert ก่อนจะ error ทันทีเมื่อรูปต้นฉบับเป็น PNG โปร่งใส |
| `quality=85, optimize=True` | ค่าที่สมดุลระหว่างคุณภาพภาพกับขนาดไฟล์ที่ได้ (85 คือค่าที่แทบมองไม่ออกว่าถูกบีบอัด) |

### 572.3 ใช้งาน Thumbnail ใน Template

```html
<!-- blog/templates/blog/post_list.html -->
{% for post in posts %}
    <article class="post-card">
        {% if post.cover_image_thumbnail %}
            <img src="{{ post.cover_image_thumbnail.url }}" alt="{{ post.title }}"
                 width="400" height="400" loading="lazy">
        {% endif %}
        <h2>{{ post.title }}</h2>
    </article>
{% endfor %}
```

การใช้ thumbnail (400×400, ไฟล์เล็ก) ในหน้ารายการบทความ แล้วใช้รูปเต็ม (`cover_image`)
เฉพาะในหน้ารายละเอียดบทความ คือรูปแบบมาตรฐานที่เว็บไซต์ระดับโลกใช้กันทั้งหมด — ผู้ใช้ที่
กำลังเลื่อนดูรายการ 50 บทความไม่จำเป็นต้องดาวน์โหลดรูปความละเอียดสูง 50 รูปพร้อมกัน

### 572.4 ข้อจำกัดของแนวทางนี้ที่ควรรู้

การ override `save()` แบบข้างต้นใช้งานได้จริง แต่มีข้อจำกัด:

- ถ้าต้องการรูปหลายขนาด (thumbnail, medium, large, retina) จะต้องเขียนเมธอดคล้ายกันซ้ำ
  หลายตัว โค้ดเริ่มยืดยาว
- การประมวลผลรูปภาพเกิดขึ้น **แบบ synchronous** ระหว่าง request — ถ้ารูปใหญ่มากหรือ
  ต้องสร้างหลายขนาด ผู้ใช้ต้องรอ response นานขึ้น
- ถ้าเปลี่ยนขนาด thumbnail ในอนาคต (เช่นจาก 400×400 เป็น 300×300) ต้องเขียน script
  ไล่ re-generate รูปเก่าทั้งหมดเอง

ขั้นตอนถัดไปจะแนะนำ **django-imagekit** ซึ่งแก้ปัญหาทั้งสามข้อนี้ได้อย่างสวยงาม

---

## ขั้นตอนที่ 573: django-imagekit สำหรับสร้าง Image Variant หลายขนาดอัตโนมัติ

### 573.1 แนวคิดของ ImageKit: Lazy Generation แทน Eager Generation

**django-imagekit** ใช้แนวคิดต่างจากที่เราเขียนเองในขั้นตอนที่ 572 โดยสิ้นเชิง: แทนที่จะ
สร้างรูปทุกขนาด**ทันที**ตอนอัปโหลด (eager) มันจะสร้างรูป **"เมื่อถูกเรียกใช้ครั้งแรก"**
เท่านั้น (lazy) แล้ว cache ไฟล์ที่สร้างแล้วไว้ใช้ซ้ำ

```
Eager (แบบที่เราเขียนเอง 572)          Lazy (ImageKit)
┌──────────────┐                        ┌──────────────┐
│ อัปโหลดรูป    │                        │ อัปโหลดรูป    │
└──────┬───────┘                        └──────┬───────┘
       │ สร้างทุกขนาดทันที                      │ บันทึกแค่รูปต้นฉบับ
       ▼                                        ▼
┌──────────────┐                        ┌──────────────┐
│ thumbnail ✓  │                        │ (ยังไม่มี variant ใด ๆ)
│ medium ✓     │                        └──────┬───────┘
│ large ✓      │                               │ ครั้งแรกที่ template
└──────────────┘                               │ เรียก .url
                                                ▼
                                        ┌──────────────┐
                                        │ สร้าง + cache │
                                        │ variant นั้น   │
                                        └──────────────┘
```

ข้อดีของแนวทาง lazy: อัปโหลดไฟล์เสร็จเร็ว (ไม่ต้องรอประมวลผลหลายขนาด), เปลี่ยนขนาด/
คุณภาพของ variant ได้ตลอดเวลาโดยแค่ล้าง cache แล้วให้มันสร้างใหม่ ไม่ต้องรัน migration
หรือ script พิเศษ

### 573.2 ติดตั้ง django-imagekit

```bash
pip install django-imagekit
pip freeze > requirements.txt
```

```python
# config/settings.py
INSTALLED_APPS = [
    # ...
    "imagekit",
    "blog",
]

# โฟลเดอร์เก็บไฟล์ variant ที่ถูกสร้างขึ้น (cache files)
IMAGEKIT_CACHEFILE_DIR = "CACHE/images"

# กลยุทธ์การสร้าง cache file — Simple สร้างทันทีที่ถูกเรียก .url ครั้งแรก (เหมาะกับเริ่มต้น)
# ตัวเลือกอื่น: Optimistic, Async (ต้องใช้ร่วมกับ Celery จาก Phase หลัง)
IMAGEKIT_DEFAULT_CACHEFILE_STRATEGY = "imagekit.cachefiles.strategies.JustInTime"
```

### 573.3 กำหนด ImageSpecField สำหรับแต่ละขนาด

```python
# blog/models.py
from django.db import models
from imagekit.models import ImageSpecField
from imagekit.processors import ResizeToFill, ResizeToFit

from .validators import validate_image_extension, validate_image_file_size


class Post(models.Model):
    title = models.CharField(max_length=200)
    slug = models.SlugField(unique=True)
    content = models.TextField()

    # รูปต้นฉบับ (ยังคงเก็บไว้เต็มความละเอียด สำหรับกรณีต้องใช้ในอนาคต)
    cover_image = models.ImageField(
        upload_to="blog/covers/%Y/%m/",
        blank=True,
        null=True,
        validators=[validate_image_file_size, validate_image_extension],
    )

    # Thumbnail: crop ให้เป็นสี่เหลี่ยมจัตุรัส 300x300 เสมอ (สำหรับการ์ดรายการ)
    cover_thumbnail = ImageSpecField(
        source="cover_image",
        processors=[ResizeToFill(300, 300)],
        format="JPEG",
        options={"quality": 75},
    )

    # Medium: ย่อให้พอดีกรอบ 800x600 โดยรักษาสัดส่วนเดิม (ไม่ crop)
    cover_medium = ImageSpecField(
        source="cover_image",
        processors=[ResizeToFit(800, 600)],
        format="JPEG",
        options={"quality": 85},
    )

    # Large: สำหรับหน้ารายละเอียดบทความ ย่อเฉพาะถ้าใหญ่เกิน 1600px
    cover_large = ImageSpecField(
        source="cover_image",
        processors=[ResizeToFit(1600, 1200)],
        format="JPEG",
        options={"quality": 90},
    )

    created_at = models.DateTimeField(auto_now_add=True)

    def __str__(self):
        return self.title
```

| Processor | พฤติกรรม | เหมาะกับ |
|---|---|---|
| `ResizeToFill(w, h)` | Crop + resize ให้ได้ขนาดเป๊ะ `w × h` เสมอ (อาจตัดบางส่วนของภาพทิ้ง) | Thumbnail, avatar ที่ต้องการสัดส่วนคงที่ทุกใบ |
| `ResizeToFit(w, h)` | ย่อให้พอดีกรอบ โดยรักษาสัดส่วนเดิม (ไม่ crop เนื้อหา แต่ผลลัพธ์อาจไม่เท่ากรอบเป๊ะ) | รูปที่ต้องการแสดงเนื้อหาครบ ไม่อยากให้ crop |
| `SmartResize(w, h)` | คล้าย `ResizeToFill` แต่พยายามหาจุดที่น่าสนใจที่สุดของภาพก่อน crop | รูปที่วัตถุหลักไม่ได้อยู่กึ่งกลางเป๊ะ |
| `Adjust(contrast=, brightness=, sharpness=)` | ปรับค่าสี — มักใช้ร่วมกับ resize processor อื่น | ปรับภาพให้ดูคมชัดขึ้นหลังย่อขนาด |

### 573.4 รัน Migration

`ImageSpecField` **ไม่ได้สร้างคอลัมน์ใหม่ในฐานข้อมูล** (มันคำนวณจาก `cover_image` ทุกครั้ง
ที่ถูกเรียกใช้ ไม่ได้เก็บ path ลง DB) แต่ยังต้องรัน `makemigrations`/`migrate` ตามปกติ เพราะ
`ImageKit` อาจปรับ metadata ของ field เดิมเล็กน้อย:

```bash
python manage.py makemigrations blog
python manage.py migrate
```

### 573.5 ใช้งานใน Template — เหมือน ImageField ทุกประการ

```html
<!-- blog/templates/blog/post_list.html -->
{% for post in posts %}
    <article class="post-card">
        {% if post.cover_image %}
            <img src="{{ post.cover_thumbnail.url }}" alt="{{ post.title }}"
                 width="300" height="300" loading="lazy">
        {% endif %}
        <h2>{{ post.title }}</h2>
    </article>
{% endfor %}
```

```html
<!-- blog/templates/blog/post_detail.html -->
{% if post.cover_image %}
    <picture>
        <source media="(min-width: 1024px)" srcset="{{ post.cover_large.url }}">
        <img src="{{ post.cover_medium.url }}" alt="{{ post.title }}" loading="lazy">
    </picture>
{% endif %}
```

ครั้งแรกที่ template เข้าถึง `post.cover_thumbnail.url` ImageKit จะ:
1. ตรวจสอบว่ามีไฟล์ cache ของ variant นี้อยู่แล้วหรือยัง (ใน `IMAGEKIT_CACHEFILE_DIR`)
2. ถ้ายังไม่มี → เปิดรูปต้นฉบับ, รัน processor ที่กำหนด, บันทึกไฟล์ผลลัพธ์ลง cache
3. คืน URL ของไฟล์ cache นั้นกลับมา
4. ครั้งถัดไปที่เรียก `.url` (ไม่ว่าจาก request ไหน) จะได้ไฟล์ที่ cache ไว้แล้วทันที
   ไม่ต้องประมวลผลซ้ำ

### 573.6 คำสั่ง Management Command ที่มีประโยชน์

```bash
# สร้าง cache file ล่วงหน้าสำหรับทุก record ที่มีอยู่แล้ว (แนะนำให้รันหลัง deploy
# ฟีเจอร์ใหม่ที่เพิ่ม ImageSpecField เพื่อไม่ให้ผู้ใช้คนแรกต้องรอโหลดช้า)
python manage.py generateimages

# ล้าง cache file ทั้งหมด (ใช้เมื่อเปลี่ยนค่า processor เช่นเปลี่ยนขนาด thumbnail)
python manage.py clearcache 2>/dev/null || python manage.py cleanimages
```

### 573.7 เปรียบเทียบ: เขียนเอง (572) vs django-imagekit (573)

| ประเด็น | เขียนเอง (override `save()`) | django-imagekit |
|---|---|---|
| ความเร็วตอนอัปโหลด | ช้ากว่า (ประมวลผลทุกขนาดทันที) | เร็วกว่า (บันทึกแค่ต้นฉบับ) |
| เปลี่ยนขนาด variant ภายหลัง | ต้อง migrate ข้อมูลเก่าเอง | แค่แก้ processor แล้ว `clearcache` |
| ควบคุมได้ละเอียดแค่ไหน | ควบคุมได้ 100% (โค้ด Pillow ล้วน) | ควบคุมผ่าน processor ที่มีให้ (ขยายเองได้ผ่าน custom processor) |
| ต้องเรียนรู้ library เพิ่มไหม | ไม่ต้อง (ใช้ Pillow ที่มีอยู่แล้ว) | ต้องเรียนรู้ API ของ imagekit |
| เหมาะกับ | โปรเจกต์เล็ก, ต้องการ logic ประมวลผลภาพซับซ้อนเฉพาะทาง | โปรเจกต์ส่วนใหญ่ที่ต้องการหลายขนาดมาตรฐาน |

**คำแนะนำของหลักสูตรนี้**: ใช้ **django-imagekit** เป็นค่าเริ่มต้นสำหรับงาน resize/thumbnail
ทั่วไป และเขียน custom processing เอง (แบบขั้นตอนที่ 572) เฉพาะกรณีพิเศษที่ imagekit ไม่มี
processor รองรับ (เช่น การใส่ watermark แบบซับซ้อน หรือ face detection)

---

## ขั้นตอนที่ 574: จัดการไฟล์ขนาดใหญ่ — Chunked Upload พร้อม Progress Bar

### 574.1 ทำไมไฟล์ใหญ่ถึงต้องอัปโหลดแบบ Chunk

การอัปโหลดไฟล์แบบธรรมดา (single request) ส่งทั้งไฟล์เป็นก้อนเดียวใน HTTP request เดียว
สำหรับไฟล์เล็ก (< 20 MB) วิธีนี้ไม่มีปัญหา แต่สำหรับไฟล์ใหญ่ (วิดีโอ, ไฟล์ backup หลายร้อย MB)
จะเจอปัญหา:

- **Timeout**: เว็บเซิร์ฟเวอร์/reverse proxy มักตั้ง timeout ไว้ (เช่น Nginx `proxy_read_timeout`
  60 วินาที) ถ้าอัปโหลดไม่เสร็จภายในเวลานั้น request จะถูกตัดทิ้งทั้งหมด
- **ไม่มี resume**: ถ้าเน็ตหลุดตอนอัปโหลดไป 90% ต้องเริ่มใหม่ตั้งแต่ 0%
- **Memory spike**: แม้ Django จะ stream ไฟล์ใหญ่ลง temp file (ตามขั้นตอนที่ 571.6) แต่
  reverse proxy บางตัวยังอาจ buffer ทั้ง request ไว้ใน memory ก่อนส่งต่อ

**Chunked upload** แก้ปัญหานี้ด้วยการตัดไฟล์เป็นชิ้นเล็ก ๆ (เช่นชิ้นละ 5 MB) แล้วอัปโหลด
ทีละชิ้นเป็นหลาย request แยกกัน โดยฝั่งเซิร์ฟเวอร์จะประกอบชิ้นส่วนกลับเป็นไฟล์เดียวเมื่อ
รับครบทุกชิ้น

### 574.2 ออกแบบ Model และ View ฝั่งเซิร์ฟเวอร์

```python
# blog/models.py (เพิ่มเติม)
import uuid

from django.db import models


class ChunkedUpload(models.Model):
    """เก็บสถานะการอัปโหลดไฟล์แบบแบ่งชิ้น ระหว่างที่ยังอัปโหลดไม่ครบ"""

    STATUS_UPLOADING = "uploading"
    STATUS_COMPLETE = "complete"
    STATUS_CHOICES = [
        (STATUS_UPLOADING, "กำลังอัปโหลด"),
        (STATUS_COMPLETE, "อัปโหลดเสร็จสมบูรณ์"),
    ]

    upload_id = models.UUIDField(default=uuid.uuid4, unique=True, editable=False)
    file_name = models.CharField(max_length=255)
    total_size = models.PositiveBigIntegerField()
    uploaded_size = models.PositiveBigIntegerField(default=0)
    status = models.CharField(max_length=20, choices=STATUS_CHOICES, default=STATUS_UPLOADING)
    temp_path = models.CharField(max_length=500)
    created_at = models.DateTimeField(auto_now_add=True)

    def __str__(self):
        return f"{self.file_name} ({self.uploaded_size}/{self.total_size} bytes)"
```

```python
# blog/views.py
import os
import uuid

from django.conf import settings
from django.http import JsonResponse
from django.views import View
from django.views.decorators.csrf import csrf_exempt
from django.utils.decorators import method_decorator

from .models import ChunkedUpload, Post


CHUNK_UPLOAD_TEMP_DIR = os.path.join(settings.MEDIA_ROOT, "chunked_uploads")


@method_decorator(csrf_exempt, name="dispatch")  # ตัวอย่างนี้ยกเว้น CSRF ชั่วคราวเพื่อความ
                                                  # กระชับ — ในงานจริงควรแนบ X-CSRFToken
                                                  # แบบเดียวกับ Part 052 ขั้นตอนที่ 513 แทน
class ChunkedUploadInitView(View):
    """ขั้นตอนที่ 1: ขอเริ่มอัปโหลด — client บอกชื่อไฟล์และขนาดรวม"""

    def post(self, request):
        file_name = request.POST.get("file_name")
        total_size = int(request.POST.get("total_size", 0))

        if not file_name or total_size <= 0:
            return JsonResponse({"error": "ข้อมูลไม่ครบถ้วน"}, status=400)

        os.makedirs(CHUNK_UPLOAD_TEMP_DIR, exist_ok=True)

        upload = ChunkedUpload.objects.create(
            file_name=file_name,
            total_size=total_size,
            temp_path=os.path.join(CHUNK_UPLOAD_TEMP_DIR, f"{uuid.uuid4()}.part"),
        )

        return JsonResponse({"upload_id": str(upload.upload_id)})


@method_decorator(csrf_exempt, name="dispatch")
class ChunkedUploadAppendView(View):
    """ขั้นตอนที่ 2: รับไฟล์ทีละชิ้น แล้วเขียนต่อท้ายไฟล์ temp บนดิสก์"""

    def post(self, request, upload_id):
        try:
            upload = ChunkedUpload.objects.get(
                upload_id=upload_id, status=ChunkedUpload.STATUS_UPLOADING
            )
        except ChunkedUpload.DoesNotExist:
            return JsonResponse({"error": "ไม่พบรายการอัปโหลดนี้"}, status=404)

        chunk = request.FILES.get("chunk")
        if not chunk:
            return JsonResponse({"error": "ไม่พบไฟล์ชิ้นส่วน"}, status=400)

        # เขียนต่อท้ายไฟล์ temp เสมอ (mode "ab" = append binary)
        with open(upload.temp_path, "ab") as temp_file:
            for data in chunk.chunks():
                temp_file.write(data)

        upload.uploaded_size += chunk.size
        upload.save(update_fields=["uploaded_size"])

        progress_percent = round((upload.uploaded_size / upload.total_size) * 100, 1)

        return JsonResponse({
            "uploaded_size": upload.uploaded_size,
            "total_size": upload.total_size,
            "progress_percent": progress_percent,
        })


@method_decorator(csrf_exempt, name="dispatch")
class ChunkedUploadCompleteView(View):
    """ขั้นตอนที่ 3: เมื่อรับครบทุกชิ้นแล้ว ย้ายไฟล์ temp ไปผูกกับ Post จริง"""

    def post(self, request, upload_id):
        try:
            upload = ChunkedUpload.objects.get(
                upload_id=upload_id, status=ChunkedUpload.STATUS_UPLOADING
            )
        except ChunkedUpload.DoesNotExist:
            return JsonResponse({"error": "ไม่พบรายการอัปโหลดนี้"}, status=404)

        if upload.uploaded_size != upload.total_size:
            return JsonResponse({
                "error": f"ไฟล์ยังไม่ครบ ({upload.uploaded_size}/{upload.total_size} bytes)"
            }, status=400)

        from django.core.files import File

        post_id = request.POST.get("post_id")
        post = Post.objects.get(pk=post_id)

        with open(upload.temp_path, "rb") as f:
            post.attachment.save(upload.file_name, File(f), save=True)

        upload.status = ChunkedUpload.STATUS_COMPLETE
        upload.save(update_fields=["status"])
        os.remove(upload.temp_path)   # ลบไฟล์ temp ทิ้งหลังย้ายเสร็จ

        return JsonResponse({"success": True, "attachment_url": post.attachment.url})
```

```python
# blog/urls.py (เพิ่มเติม)
from django.urls import path

from . import views

urlpatterns = [
    # ... path เดิมจาก Part ก่อนหน้า
    path("uploads/init/", views.ChunkedUploadInitView.as_view(), name="chunked_upload_init"),
    path("uploads/<uuid:upload_id>/append/", views.ChunkedUploadAppendView.as_view(), name="chunked_upload_append"),
    path("uploads/<uuid:upload_id>/complete/", views.ChunkedUploadCompleteView.as_view(), name="chunked_upload_complete"),
]
```

### 574.3 ทำไมต้องใช้ XMLHttpRequest แทน fetch() สำหรับ Progress Bar

ทบทวนตารางจาก Part 052 ขั้นตอนที่ 512.5: **`fetch()` ไม่รองรับการติดตาม upload progress
โดยตรง** (รองรับแค่ download progress ผ่าน `ReadableStream` ซึ่งซับซ้อนกว่ามาก) ในขณะที่
`XMLHttpRequest` (XHR) รุ่นเก่ากว่ามี event `progress` ที่ใช้งานง่ายกว่ามาก — นี่คือกรณี
พิเศษหนึ่งเดียวในหลักสูตรนี้ที่เราจะกลับไปใช้ XHR แทน `fetch()`

```javascript
// blog/static/blog/js/chunked-upload.js

const CHUNK_SIZE = 5 * 1024 * 1024;   // 5 MB ต่อชิ้น

function uploadChunkWithProgress(url, chunkBlob, onProgress) {
    return new Promise((resolve, reject) => {
        const xhr = new XMLHttpRequest();
        const formData = new FormData();
        formData.append("chunk", chunkBlob);

        xhr.open("POST", url);
        xhr.setRequestHeader("X-CSRFToken", getCookie("csrftoken"));

        // event นี้คือสิ่งที่ fetch() ทำไม่ได้ — ติดตามความคืบหน้าการอัปโหลด "ต่อ 1 ชิ้น"
        xhr.upload.addEventListener("progress", (event) => {
            if (event.lengthComputable) {
                onProgress(event.loaded, event.total);
            }
        });

        xhr.addEventListener("load", () => {
            if (xhr.status >= 200 && xhr.status < 300) {
                resolve(JSON.parse(xhr.responseText));
            } else {
                reject(new Error(`อัปโหลดชิ้นส่วนล้มเหลว: ${xhr.status}`));
            }
        });

        xhr.addEventListener("error", () => reject(new Error("การเชื่อมต่อขัดข้อง")));
        xhr.send(formData);
    });
}

async function uploadFileInChunks(file, postId, progressBar, progressText) {
    // ขั้นตอนที่ 1: ขอเริ่มอัปโหลด
    const initResponse = await apiFetch("/posts/uploads/init/", {
        method: "POST",
        headers: {},   // ไม่ตั้ง Content-Type เอง เพราะใช้ FormData ด้านล่าง
        body: (() => {
            const fd = new FormData();
            fd.append("file_name", file.name);
            fd.append("total_size", file.size);
            return fd;
        })(),
    });
    const { upload_id: uploadId } = await initResponse.json();

    // ขั้นตอนที่ 2: ตัดไฟล์เป็นชิ้นแล้วอัปโหลดทีละชิ้นตามลำดับ (สำคัญ: ต้องเรียงลำดับ
    // เพราะ server เขียนไฟล์แบบ append ต่อท้ายเรื่อย ๆ — ถ้ายิงพร้อมกันไฟล์จะสลับลำดับ)
    let uploadedBytes = 0;

    for (let start = 0; start < file.size; start += CHUNK_SIZE) {
        const chunk = file.slice(start, start + CHUNK_SIZE);

        await uploadChunkWithProgress(
            `/posts/uploads/${uploadId}/append/`,
            chunk,
            (chunkLoaded) => {
                const overallLoaded = uploadedBytes + chunkLoaded;
                const percent = Math.round((overallLoaded / file.size) * 100);
                progressBar.style.width = `${percent}%`;
                progressText.textContent = `${percent}% (${formatBytes(overallLoaded)} / ${formatBytes(file.size)})`;
            }
        );

        uploadedBytes += chunk.size;
    }

    // ขั้นตอนที่ 3: แจ้งเซิร์ฟเวอร์ว่าอัปโหลดครบแล้ว ให้ประกอบไฟล์และผูกกับ Post
    const completeResponse = await apiFetch(`/posts/uploads/${uploadId}/complete/`, {
        method: "POST",
        headers: {},
        body: (() => {
            const fd = new FormData();
            fd.append("post_id", postId);
            return fd;
        })(),
    });

    return completeResponse.json();
}

function formatBytes(bytes) {
    if (bytes < 1024) return `${bytes} B`;
    if (bytes < 1024 * 1024) return `${(bytes / 1024).toFixed(1)} KB`;
    return `${(bytes / (1024 * 1024)).toFixed(1)} MB`;
}
```

### 574.4 HTML และการเชื่อมต่อ UI

```html
<!-- blog/templates/blog/post_upload_large_file.html -->
{% extends "base.html" %}
{% load static %}

{% block content %}
<div class="upload-form" data-post-id="{{ post.id }}">
    <h2>อัปโหลดไฟล์แนบขนาดใหญ่</h2>
    <input type="file" id="large-file-input">

    <div class="progress-track" hidden id="progress-container">
        <div class="progress-bar" id="progress-bar" style="width: 0%;"></div>
    </div>
    <p id="progress-text"></p>
</div>
{% endblock %}

{% block extra_js %}
    <script src="{% static 'blog/js/csrf.js' %}" defer></script>
    <script src="{% static 'blog/js/api.js' %}" defer></script>
    <script src="{% static 'blog/js/chunked-upload.js' %}" defer></script>
    <script defer>
        document.addEventListener("DOMContentLoaded", () => {
            const form = document.querySelector(".upload-form");
            const fileInput = document.getElementById("large-file-input");
            const progressContainer = document.getElementById("progress-container");
            const progressBar = document.getElementById("progress-bar");
            const progressText = document.getElementById("progress-text");

            fileInput.addEventListener("change", async () => {
                const file = fileInput.files[0];
                if (!file) return;

                progressContainer.hidden = false;
                const result = await uploadFileInChunks(
                    file, form.dataset.postId, progressBar, progressText
                );
                progressText.textContent = `อัปโหลดสำเร็จ: ${result.attachment_url}`;
            });
        });
    </script>
{% endblock %}
```

```css
/* blog/static/blog/css/blog.css (เพิ่มเติม) */
.progress-track {
    width: 100%;
    height: 20px;
    background-color: #e2e2e2;
    border-radius: 10px;
    overflow: hidden;
    margin: 1rem 0 0.5rem;
}

.progress-bar {
    height: 100%;
    background-color: #2b6cb0;
    transition: width 0.15s ease-out;
}
```

### 574.5 ทางเลือกในโลกจริง: django-chunked-upload

โค้ดข้างต้นสาธิตกลไกเบื้องหลังให้เข้าใจ แต่ในโปรเจกต์การผลิตจริง มี library สำเร็จรูปที่
จัดการ edge case ได้ครบกว่า (เช่น การ resume เมื่อ session ขาดหาย, checksum ตรวจสอบความ
ถูกต้องของไฟล์ที่ประกอบเสร็จ):

```bash
pip install django-chunked-upload
```

หลักสูตรนี้แนะนำให้เข้าใจกลไก manual ก่อนเสมอ (ตามที่เราเพิ่งเขียน) เพราะเมื่อเจอปัญหา
production จริง (เช่น chunk มาไม่ครบ, ไฟล์ temp ค้าง) คุณจะ debug ได้เร็วกว่าถ้าเข้าใจว่า
เบื้องหลัง library สำเร็จรูปทำงานอย่างไร

---

## ขั้นตอนที่ 575: Direct-to-S3 Upload ด้วย Presigned URL

### 575.1 ปัญหาของการอัปโหลดผ่าน Django Server

ทุกแนวทางที่ผ่านมา (571-574) ให้ไฟล์วิ่งผ่าน Django server เสมอ: `browser → Django →
disk/S3` สำหรับเว็บไซต์ขนาดใหญ่ที่มีผู้ใช้อัปโหลดพร้อมกันจำนวนมาก (เช่นแพลตฟอร์มแชร์วิดีโอ)
วิธีนี้มีปัญหา:

- Django server (worker process ของ Gunicorn/uWSGI) ถูก **ยึดครอง** ตลอดเวลาที่ไฟล์กำลัง
  อัปโหลด แทนที่จะว่างไปรับ request อื่น
- Bandwidth ของเซิร์ฟเวอร์ถูกใช้ไปกับการ "รับไฟล์แล้วส่งต่อไป S3" ทั้งที่ไม่จำเป็น
- Scale ยาก: ยิ่งมีคนอัปโหลดพร้อมกันมาก ยิ่งต้องเพิ่มจำนวน worker

**Direct-to-S3 Upload** แก้ปัญหานี้โดยให้ **browser อัปโหลดไฟล์ตรงไปยัง S3 เลย** โดยไม่ผ่าน
Django server แม้แต่ byte เดียว — Django มีหน้าที่แค่ "ออกใบอนุญาตชั่วคราว" (Presigned URL)
ให้ browser ใช้อัปโหลดเท่านั้น

```
แบบเดิม (ผ่าน Django):
Browser ──(ไฟล์ทั้งไฟล์)──> Django Server ──(ไฟล์ทั้งไฟล์)──> S3

แบบ Direct-to-S3:
Browser ──(ขอ presigned URL, ไม่มีไฟล์)──> Django Server
Browser <──(URL + fields ชั่วคราว)──────── Django Server
Browser ──(ไฟล์ทั้งไฟล์)─────────────────────────────────> S3  (ตรง ไม่ผ่าน Django)
Browser ──(แจ้งว่าอัปโหลดเสร็จแล้ว, ไม่มีไฟล์)──> Django Server (บันทึก path ลง DB)
```

### 575.2 ติดตั้ง boto3 และ django-storages

```bash
pip install boto3 django-storages
pip freeze > requirements.txt
```

```python
# config/settings.py
INSTALLED_APPS = [
    # ...
    "storages",
]

AWS_ACCESS_KEY_ID = env("AWS_ACCESS_KEY_ID")
AWS_SECRET_ACCESS_KEY = env("AWS_SECRET_ACCESS_KEY")
AWS_STORAGE_BUCKET_NAME = env("AWS_STORAGE_BUCKET_NAME")
AWS_S3_REGION_NAME = env("AWS_S3_REGION_NAME", default="ap-southeast-1")
```

(ทบทวนการใช้ `env()` จาก `django-environ` สำหรับอ่านค่าจาก `.env` ตามที่ Part 004/031
เคยแนะนำไว้ — **ห้าม hardcode AWS credentials ในโค้ดเด็ดขาด**)

### 575.3 View ที่สร้าง Presigned POST Data

```python
# blog/views.py (เพิ่มเติม)
import uuid

import boto3
from django.conf import settings
from django.http import JsonResponse
from django.views import View


class S3PresignedUploadView(View):
    """สร้าง presigned POST data ให้ browser ใช้อัปโหลดไฟล์ตรงไป S3"""

    ALLOWED_CONTENT_TYPES = ["image/jpeg", "image/png", "image/webp", "video/mp4"]
    MAX_FILE_SIZE = 100 * 1024 * 1024  # 100 MB

    def post(self, request):
        file_name = request.POST.get("file_name", "")
        content_type = request.POST.get("content_type", "")

        if content_type not in self.ALLOWED_CONTENT_TYPES:
            return JsonResponse({"error": "ประเภทไฟล์ไม่ได้รับอนุญาต"}, status=400)

        ext = file_name.rsplit(".", 1)[-1] if "." in file_name else "bin"
        object_key = f"uploads/{uuid.uuid4()}.{ext}"

        s3_client = boto3.client(
            "s3",
            region_name=settings.AWS_S3_REGION_NAME,
            aws_access_key_id=settings.AWS_ACCESS_KEY_ID,
            aws_secret_access_key=settings.AWS_SECRET_ACCESS_KEY,
        )

        presigned_data = s3_client.generate_presigned_post(
            Bucket=settings.AWS_STORAGE_BUCKET_NAME,
            Key=object_key,
            Fields={"Content-Type": content_type},
            Conditions=[
                {"Content-Type": content_type},
                ["content-length-range", 1, self.MAX_FILE_SIZE],
            ],
            ExpiresIn=300,   # URL ใช้ได้แค่ 5 นาที — จำกัดเวลาเพื่อความปลอดภัย
        )

        return JsonResponse({
            "url": presigned_data["url"],       # endpoint ของ S3 bucket ที่ต้อง POST ไปหา
            "fields": presigned_data["fields"],  # fields พิเศษที่ต้องแนบไปด้วย (signature ฯลฯ)
            "object_key": object_key,
        })
```

```python
# blog/urls.py (เพิ่มเติม)
urlpatterns += [
    path("uploads/s3-presign/", views.S3PresignedUploadView.as_view(), name="s3_presign"),
]
```

**อธิบาย `Conditions`**: นี่คือหัวใจของความปลอดภัยของ presigned URL — มันจำกัดว่า browser
สามารถอัปโหลดได้แค่ตามเงื่อนไขที่ Django กำหนดไว้ล่วงหน้าเท่านั้น (ประเภทไฟล์ต้องตรง, ขนาด
ไฟล์ต้องอยู่ในช่วง 1 byte ถึง 100 MB) แม้ browser จะพยายามส่งค่าอื่นมา S3 ก็จะปฏิเสธเอง
โดย Django server ไม่ต้องมาตรวจสอบไฟล์อีกรอบเลย

### 575.4 JavaScript ฝั่ง Client — อัปโหลดตรงไป S3

```javascript
// blog/static/blog/js/s3-direct-upload.js

async function uploadFileDirectToS3(file, onProgress) {
    // ขั้นตอนที่ 1: ขอ presigned data จาก Django (คำขอนี้เท่านั้นที่ผ่าน Django)
    const presignResponse = await apiFetch("/posts/uploads/s3-presign/", {
        method: "POST",
        headers: {},
        body: (() => {
            const fd = new FormData();
            fd.append("file_name", file.name);
            fd.append("content_type", file.type);
            return fd;
        })(),
    });

    if (!presignResponse.ok) {
        throw new Error("ขอสิทธิ์อัปโหลดไม่สำเร็จ");
    }

    const { url, fields, object_key: objectKey } = await presignResponse.json();

    // ขั้นตอนที่ 2: ประกอบ FormData ตามที่ S3 กำหนด (ต้องใส่ fields ก่อน "file" เสมอ)
    const s3FormData = new FormData();
    Object.entries(fields).forEach(([key, value]) => {
        s3FormData.append(key, value);
    });
    s3FormData.append("file", file);   // ไฟล์จริงต้องอยู่ "หลังสุด" ของ FormData

    // ขั้นตอนที่ 3: POST ตรงไปยัง S3 endpoint — ไม่ผ่าน apiFetch (ไม่ใช่ same-origin,
    // ไม่ต้องแนบ CSRF token หรือ cookie ใด ๆ เพราะ S3 authenticate ด้วย fields ที่แนบมาแทน)
    await new Promise((resolve, reject) => {
        const xhr = new XMLHttpRequest();
        xhr.open("POST", url);

        xhr.upload.addEventListener("progress", (event) => {
            if (event.lengthComputable) {
                onProgress(Math.round((event.loaded / event.total) * 100));
            }
        });

        xhr.addEventListener("load", () => {
            // S3 คืน 204 No Content เมื่อสำเร็จ
            xhr.status >= 200 && xhr.status < 300
                ? resolve()
                : reject(new Error(`S3 ปฏิเสธการอัปโหลด: ${xhr.status}`));
        });
        xhr.addEventListener("error", () => reject(new Error("เชื่อมต่อ S3 ไม่สำเร็จ")));
        xhr.send(s3FormData);
    });

    return objectKey;
}
```

### 575.5 แจ้งผลกลับ Django หลังอัปโหลดสำเร็จ

หลังไฟล์อยู่บน S3 แล้ว ต้องมีอีกขั้นตอนหนึ่งเพื่อ **บันทึก path ของไฟล์ลงฐานข้อมูล**
เพราะ Django server ไม่เคยรู้เลยว่าไฟล์นี้ถูกอัปโหลดสำเร็จ (มันแค่ออก URL ให้ตอนต้น):

```python
# blog/views.py (เพิ่มเติม)
class ConfirmS3UploadView(View):
    """บันทึก object_key ที่อัปโหลดสำเร็จแล้วลงกับ Post ในฐานข้อมูล"""

    def post(self, request):
        post_id = request.POST.get("post_id")
        object_key = request.POST.get("object_key")

        post = Post.objects.get(pk=post_id)
        # เก็บแค่ key ไว้ใน field (ไม่ต้องอัปโหลดไฟล์ผ่าน Django field.save() อีกรอบ
        # เพราะไฟล์อยู่บน S3 แล้ว — แค่ตั้งค่า .name ตรง ๆ ให้ field รู้ path)
        post.cover_image.name = object_key
        post.save(update_fields=["cover_image"])

        return JsonResponse({"success": True, "url": post.cover_image.url})
```

```javascript
// เรียกใช้งานฟังก์ชันทั้งหมดร่วมกัน
async function handleDirectUpload(file, postId, progressBar) {
    const objectKey = await uploadFileDirectToS3(file, (percent) => {
        progressBar.style.width = `${percent}%`;
    });

    const confirmResponse = await apiFetch("/posts/uploads/s3-confirm/", {
        method: "POST",
        headers: {},
        body: (() => {
            const fd = new FormData();
            fd.append("post_id", postId);
            fd.append("object_key", objectKey);
            return fd;
        })(),
    });

    return confirmResponse.json();
}
```

### 575.6 ตั้งค่า CORS บน S3 Bucket

เพราะ browser ยิง request ข้าม origin ไปยัง S3 โดยตรง (จาก `https://mysite.com` ไปยัง
`https://mybucket.s3.amazonaws.com`) ต้องตั้งค่า **CORS policy** บน S3 bucket ก่อน ไม่เช่นนั้น
browser จะบล็อก request ทันที:

```json
[
    {
        "AllowedOrigins": ["https://mysite.com"],
        "AllowedMethods": ["POST"],
        "AllowedHeaders": ["*"],
        "MaxAgeSeconds": 3000
    }
]
```

### 575.7 เปรียบเทียบแนวทางการอัปโหลดทั้งหมดที่เรียนมา

| แนวทาง | ไฟล์ผ่าน Django Server ไหม | เหมาะกับ |
|---|---|---|
| Model form ธรรมดา (Part 009 + 571) | ✅ ผ่าน | ไฟล์เล็ก ปริมาณผู้ใช้ไม่มาก |
| Chunked upload (574) | ✅ ผ่าน (แต่แบ่งเป็นชิ้นเล็ก) | ไฟล์ใหญ่มาก ที่ต้องการ resume/progress ละเอียด |
| Direct-to-S3 Presigned URL (575) | ❌ ไม่ผ่านเลย | ระบบที่มีผู้ใช้อัปโหลดพร้อมกันจำนวนมาก, ไฟล์ใหญ่, ต้องการลดภาระ server |

---

## ขั้นตอนที่ 576: ภาพรวมการอัปโหลดวิดีโอ (แนวคิด Transcoding)

### 576.1 ทำไมวิดีโอถึงซับซ้อนกว่ารูปภาพมาก

รูปภาพเปิดดูได้ทันทีในทุก browser ไม่ว่าจะเป็น `.jpg`, `.png` หรือ `.webp` แต่วิดีโอมีความ
ซับซ้อนกว่ามาก:

- **ไฟล์ต้นฉบับมักใหญ่มาก**: วิดีโอความละเอียดสูง (4K) จากมือถือ อาจมีขนาดหลาย GB ต่อไฟล์
- **ไม่ใช่ทุก browser รองรับทุก codec**: วิดีโอที่ผู้ใช้อัปโหลดมาอาจเข้ารหัสด้วย codec ที่
  บาง browser เปิดไม่ได้ (เช่น `.mov` จาก iPhone ที่ใช้ codec HEVC)
- **ต้องรองรับหลายความเร็วอินเทอร์เน็ต**: ผู้ชมที่เน็ตช้าควรได้วิดีโอความละเอียดต่ำกว่า
  แทนที่จะบังคับดาวน์โหลดไฟล์ 4K เต็ม

**Transcoding** คือกระบวนการแปลงวิดีโอต้นฉบับให้เป็นรูปแบบที่ browser เปิดได้แน่นอน (เช่น
H.264 ใน container MP4) และมักสร้างหลายความละเอียดพร้อมกัน (1080p, 720p, 480p) เพื่อให้
ผู้ชมเลือกตามความเร็วอินเทอร์เน็ต

### 576.2 เครื่องมือมาตรฐานในอุตสาหกรรม: FFmpeg

**FFmpeg** คือเครื่องมือ command-line โอเพนซอร์สที่เป็นมาตรฐานอุตสาหกรรมสำหรับประมวลผล
วิดีโอ/เสียงแทบทุกระบบที่มีวิดีโอเกี่ยวข้อง (YouTube, Netflix เบื้องหลังก็ใช้แนวคิดเดียวกัน
แม้จะมี infrastructure ซับซ้อนกว่ามาก)

```bash
# ตัวอย่างคำสั่ง FFmpeg แปลงวิดีโอเป็น MP4 (H.264) และย่อเป็น 720p
ffmpeg -i input.mov -vf "scale=-2:720" -c:v libx264 -crf 23 -c:a aac output_720p.mp4
```

### 576.3 สถาปัตยกรรมแนวคิด: ทำไมต้องทำแบบ Asynchronous

การ transcode วิดีโอ 1 ไฟล์อาจใช้เวลาหลายนาทีถึงหลายสิบนาที (ขึ้นกับความยาวและความละเอียด)
— **ห้าม** ทำในระหว่าง HTTP request เด็ดขาด เพราะผู้ใช้ต้องรอนานเกินกว่าที่ browser/reverse
proxy จะยอม timeout ให้ (ทบทวนปัญหา timeout จากขั้นตอนที่ 574.1)

```
Browser ──(อัปโหลดวิดีโอต้นฉบับ)──> Django View
                                          │
                                          │ บันทึกไฟล์ต้นฉบับ + สร้าง Task
                                          │ ส่งเข้าคิว แล้วตอบกลับ browser ทันที
                                          │ (status: "processing")
                                          ▼
                                    Task Queue (Celery + Redis/RabbitMQ)
                                          │
                                          │ Worker process แยกต่างหาก
                                          │ รัน FFmpeg (ใช้เวลานาน)
                                          ▼
                                    บันทึกผลลัพธ์ (URL วิดีโอที่ transcode แล้ว)
                                    ลงฐานข้อมูล เมื่อเสร็จ
```

```python
# blog/models.py (ตัวอย่างโครงสร้างเพื่อความเข้าใจภาพรวม)
class VideoUpload(models.Model):
    STATUS_PENDING = "pending"
    STATUS_PROCESSING = "processing"
    STATUS_READY = "ready"
    STATUS_FAILED = "failed"

    original_file = models.FileField(upload_to="videos/originals/")
    processed_file_1080p = models.FileField(upload_to="videos/processed/", blank=True)
    processed_file_720p = models.FileField(upload_to="videos/processed/", blank=True)
    thumbnail = models.ImageField(upload_to="videos/thumbnails/", blank=True)
    status = models.CharField(
        max_length=20,
        default=STATUS_PENDING,
        choices=[
            (STATUS_PENDING, "รอประมวลผล"),
            (STATUS_PROCESSING, "กำลังประมวลผล"),
            (STATUS_READY, "พร้อมใช้งาน"),
            (STATUS_FAILED, "ประมวลผลล้มเหลว"),
        ],
    )
    duration_seconds = models.PositiveIntegerField(null=True, blank=True)
```

### 576.4 แนวคิด HLS สำหรับ Adaptive Streaming (เกริ่นเท่านั้น)

แพลตฟอร์มวิดีโอระดับมืออาชีพ (YouTube, Netflix) ไม่ได้ส่งไฟล์ MP4 ไฟล์เดียวให้ผู้ชมทั้งหมด
แต่ใช้เทคนิค **HLS (HTTP Live Streaming)**: ตัดวิดีโอเป็นชิ้นเล็ก ๆ (ชิ้นละ 2-10 วินาที)
หลายความละเอียด แล้วให้ video player ฝั่ง browser **สลับความละเอียดแบบไดนามิก** ตามความเร็ว
เน็ตของผู้ชม ณ ขณะนั้น — เป็นหัวข้อขั้นสูงที่ต้องใช้ทั้ง FFmpeg, Celery, และ object storage
ร่วมกัน จะไม่ลงรายละเอียดเชิงลึกใน Part นี้ (Celery และ background task จะเรียนเต็มรูปแบบ
ใน Phase 10 — Performance & Scalability)

### 576.5 สรุปสิ่งที่ควรจำจากขั้นตอนนี้

| แนวคิด | สรุป |
|---|---|
| FFmpeg | เครื่องมือมาตรฐานสำหรับ transcode วิดีโอ/เสียง |
| ต้องทำแบบ async เสมอ | ใช้ task queue (Celery) ไม่ทำระหว่าง HTTP request |
| หลายความละเอียด | สร้างไว้หลายขนาดให้ผู้ชมเลือกตามความเร็วเน็ต |
| HLS | เทคนิค adaptive streaming ระดับมืออาชีพ (หัวข้อขั้นสูง อยู่นอกขอบเขต Part นี้) |
| จุดที่ Part นี้เกี่ยวข้องโดยตรง | การรับไฟล์วิดีโอเข้ามา (`FileField` + validator ขนาด/นามสกุลตามขั้นตอนที่ 571) และเก็บสถานะการประมวลผล |

---

## ขั้นตอนที่ 577: ความปลอดภัยของ File Upload

### 577.1 ช่องโหว่ที่มือใหม่มักพลาด: เชื่อนามสกุลไฟล์

Validator ที่เขียนไว้ในขั้นตอนที่ 571.3 (`validate_image_extension`) ตรวจสอบแค่
**นามสกุลไฟล์** (`.jpg`, `.png`) เท่านั้น — นี่คือจุดอ่อนร้ายแรง เพราะนามสกุลไฟล์เป็นแค่
**ชื่อ** ที่ผู้ใช้ตั้งเอง ไม่เกี่ยวข้องกับเนื้อหาจริงข้างในไฟล์เลย

ตัวอย่างการโจมตี: ผู้ใช้ประสงค์ร้ายสามารถเปลี่ยนชื่อไฟล์ `malicious.php` เป็น
`profile.jpg` แล้วอัปโหลดผ่านช่อง "รูปโปรไฟล์" ที่มีแค่ validator ตรวจนามสกุล — ถ้าเว็บ
เซิร์ฟเวอร์ (เช่น Apache ที่ตั้งค่าไม่รัดกุม) ถูกหลอกให้รันไฟล์ในโฟลเดอร์ media เป็น PHP
script ได้ ผู้โจมตีจะสามารถรันโค้ดอันตรายบนเซิร์ฟเวอร์ได้ทันที (Remote Code Execution)

### 577.2 ตรวจสอบเนื้อหาไฟล์จริงด้วย Magic Bytes

ไฟล์แต่ละประเภทมี **"ลายเซ็นไบต์" (magic bytes / file signature)** ที่ไม่ขึ้นกับชื่อไฟล์
เลย อยู่ในไบต์แรก ๆ ของไฟล์เสมอ:

| ประเภทไฟล์ | Magic Bytes (hex) | 
|---|---|
| JPEG | `FF D8 FF` |
| PNG | `89 50 4E 47` |
| PDF | `25 50 44 46` |
| ZIP (และ .docx/.xlsx ที่ใช้ format นี้) | `50 4B 03 04` |
| GIF | `47 49 46 38` |

```python
# blog/validators.py (เพิ่มเติม)
from django.core.exceptions import ValidationError

FILE_SIGNATURES = {
    b"\xff\xd8\xff": "image/jpeg",
    b"\x89\x50\x4e\x47": "image/png",
    b"\x52\x49\x46\x46": "image/webp",   # RIFF header (WebP ใช้ container นี้)
    b"\x25\x50\x44\x46": "application/pdf",
    b"\x50\x4b\x03\x04": "application/zip",
}


def validate_real_file_content(value):
    """ตรวจสอบเนื้อหาไฟล์จริงจาก magic bytes แทนการเชื่อนามสกุลไฟล์เพียงอย่างเดียว"""
    value.seek(0)
    file_header = value.read(8)
    value.seek(0)   # รีเซ็ต pointer กลับไปต้นไฟล์เสมอ ไม่เช่นนั้น Django จะบันทึกไฟล์
                     # ที่ขาดไบต์แรกไปเมื่อเรียก .save() ในขั้นตอนถัดไป

    is_valid = any(
        file_header.startswith(signature) for signature in FILE_SIGNATURES
    )

    if not is_valid:
        raise ValidationError(
            "ไม่สามารถตรวจสอบประเภทไฟล์ที่แท้จริงได้ ไฟล์นี้อาจไม่ใช่ไฟล์ที่อ้างว่าเป็น"
        )
```

### 577.3 ตรวจสอบไฟล์รูปภาพให้ลึกยิ่งขึ้นด้วย Pillow

สำหรับ `ImageField` โดยเฉพาะ วิธีที่แน่นอนที่สุดคือให้ Pillow **เปิดไฟล์จริง** แล้วตรวจสอบ
ว่ามันเป็นภาพที่ decode ได้จริงหรือไม่ (ไม่ใช่แค่เช็ค magic bytes ผิวเผิน):

```python
# blog/validators.py (เพิ่มเติม)
from PIL import Image, UnidentifiedImageError


def validate_image_is_genuine(value):
    """เปิดไฟล์ด้วย Pillow จริง ๆ เพื่อยืนยันว่าเป็นรูปภาพที่ decode ได้ ไม่ใช่ไฟล์ปลอมแปลง"""
    try:
        value.seek(0)
        img = Image.open(value)
        img.verify()   # ตรวจสอบโครงสร้างไฟล์ว่าไม่เสียหาย (แต่ verify() ทำให้ใช้ img ต่อไม่ได้)
    except (UnidentifiedImageError, OSError):
        raise ValidationError("ไฟล์นี้ไม่ใช่รูปภาพที่ถูกต้อง หรือไฟล์เสียหาย")
    finally:
        value.seek(0)   # รีเซ็ต pointer เสมอหลังอ่าน ไม่ว่าจะสำเร็จหรือ error
```

**หมายเหตุสำคัญ**: `ImageField` ของ Django เรียก Pillow ตรวจสอบแบบนี้อยู่แล้วเป็นค่า
เริ่มต้น (ตั้งแต่ Part 009 ขั้นตอนที่ 85.1 ที่บอกว่าต้องติดตั้ง Pillow) แต่การเขียน validator
เพิ่มเองแบบนี้ทำให้เห็น **ข้อความ error ที่ควบคุมเองได้** และเข้าใจกลไกเบื้องหลังอย่างแท้จริง
แทนที่จะพึ่งพฤติกรรม default อย่างเดียวโดยไม่รู้ว่าเกิดอะไรขึ้น

### 577.4 ป้องกัน Path Traversal จากชื่อไฟล์

ชื่อไฟล์ที่ผู้ใช้ตั้งเองอาจมีอักขระอันตราย เช่น `../../etc/passwd` (พยายามหลอกให้ระบบเขียน
ไฟล์นอกโฟลเดอร์ `MEDIA_ROOT` ที่ตั้งใจไว้) Django's `FileSystemStorage` **ป้องกันเรื่องนี้ให้
โดยอัตโนมัติอยู่แล้ว** (มันจะ sanitize path ก่อนบันทึกเสมอ) แต่ถ้าคุณเขียน custom storage
backend เอง หรือประมวลผลชื่อไฟล์ด้วยมือ (เช่นในขั้นตอนที่ 574 ที่เราตั้งชื่อไฟล์ temp เอง)
**ต้องระวังเสมอ**:

```python
# blog/validators.py (เพิ่มเติม)
import os

from django.core.exceptions import ValidationError
from django.utils.text import get_valid_filename


def sanitize_uploaded_filename(original_name):
    """ทำความสะอาดชื่อไฟล์ก่อนใช้งาน — ตัด path component และอักขระอันตรายทั้งหมด"""
    # ตัดเอาแค่ basename ทิ้ง path ใด ๆ ที่อาจแฝงมา (ป้องกัน ../../ )
    safe_name = os.path.basename(original_name)
    # ใช้ Django built-in helper ทำความสะอาดอักขระที่ไม่เหมาะกับชื่อไฟล์เพิ่มเติม
    safe_name = get_valid_filename(safe_name)

    if not safe_name:
        raise ValidationError("ชื่อไฟล์ไม่ถูกต้อง")

    return safe_name
```

### 577.5 เกริ่น Virus Scanning

สำหรับระบบที่รับไฟล์แนบจากผู้ใช้ทั่วไป (ไม่ใช่แค่รูปภาพ) เช่นระบบแนบเอกสาร PDF/ZIP
ความเสี่ยงเรื่องมัลแวร์ที่แฝงมากับไฟล์เป็นเรื่องจริงจังที่ต้องพิจารณา แนวทางมาตรฐานคือใช้
**ClamAV** (โอเพนซอร์ส antivirus engine) ร่วมกับ Python wrapper:

```bash
# ติดตั้ง ClamAV บนเซิร์ฟเวอร์ (Ubuntu/Debian)
sudo apt install clamav clamav-daemon -y
sudo freshclam   # อัปเดตฐานข้อมูลไวรัสล่าสุด

pip install pyclamd
```

```python
# blog/validators.py (ตัวอย่างแนวคิด — ต้องมี clamd daemon รันอยู่บนเซิร์ฟเวอร์)
import pyclamd
from django.core.exceptions import ValidationError


def validate_no_virus(value):
    """สแกนไฟล์ด้วย ClamAV daemon ก่อนอนุญาตให้บันทึก
    (ควรทำแบบ async ผ่าน task queue สำหรับไฟล์ใหญ่ เพื่อไม่ให้ request ค้างนาน)"""
    try:
        clamd_client = pyclamd.ClamdUnixSocket()
        value.seek(0)
        scan_result = clamd_client.scan_stream(value.read())
        value.seek(0)
    except Exception:
        # ในงานจริง ควรตัดสินใจนโยบายชัดเจนว่าถ้า ClamAV เชื่อมต่อไม่ได้ จะปฏิเสธไฟล์
        # ไปเลย (fail-closed, ปลอดภัยกว่า) หรือปล่อยผ่านชั่วคราว (fail-open)
        raise ValidationError("ไม่สามารถตรวจสอบความปลอดภัยของไฟล์ได้ในขณะนี้")

    if scan_result is not None:
        raise ValidationError("ไฟล์นี้ถูกตรวจพบว่ามีความเสี่ยงด้านความปลอดภัย")
```

**ข้อควรรู้ระดับมืออาชีพ**: การสแกนไวรัสแบบ synchronous (ระหว่าง request) เพิ่ม latency
ให้ทุกการอัปโหลด ระบบระดับ production มักออกแบบให้เป็น **ขั้นตอนหลังบันทึกไฟล์** ผ่าน task
queue เช่นเดียวกับแนวคิด transcoding ในขั้นตอนที่ 576 — บันทึกไฟล์ก่อนด้วยสถานะ "pending
scan" แล้วให้ worker แยกสแกนทีหลัง ถ้าพบไวรัสค่อยลบไฟล์และแจ้งเตือนออก

### 577.6 Checklist ความปลอดภัยของ File Upload

| หัวข้อ | สิ่งที่ต้องทำ |
|---|---|
| ขนาดไฟล์ | จำกัดทั้งที่ validator (571.3) และที่เว็บเซิร์ฟเวอร์/reverse proxy |
| นามสกุลไฟล์ | ตรวจสอบเป็นด่านแรก (571.3) แต่**ห้ามเชื่อเพียงอย่างเดียว** |
| เนื้อหาไฟล์จริง | ตรวจ magic bytes (577.2) หรือให้ Pillow เปิดไฟล์จริง (577.3) |
| ชื่อไฟล์ | Sanitize เสมอ ป้องกัน path traversal (577.4) |
| ที่เก็บไฟล์ | เก็บนอก webroot ที่รันเป็น executable ได้ (`MEDIA_ROOT` ต้องไม่ใช่โฟลเดอร์เดียวกับที่ serve `.py`/`.php`) |
| Virus scanning | พิจารณาใช้กับไฟล์แนบทั่วไปที่ไม่ใช่รูปภาพ โดยเฉพาะระบบที่รับไฟล์จากคนแปลกหน้า (577.5) |
| Content-Disposition | ตั้งค่าเว็บเซิร์ฟเวอร์ให้ไฟล์แนบที่ดาวน์โหลดถูก serve เป็น `attachment` ไม่ใช่ `inline` เพื่อป้องกัน browser รันไฟล์ HTML/SVG อันตรายที่แฝงมา |

---

## ขั้นตอนที่ 578: สร้าง UI Drag-and-drop Upload ด้วย JavaScript

### 578.1 ทบทวนรูปแบบ JavaScript จาก Part 052 ที่จะนำมาใช้ซ้ำ

Part 052 วางรากฐาน JavaScript ไว้ครบแล้ว เราจะนำมาประกอบกันสร้าง UI drag-and-drop:

| จาก Part 052 | นำมาใช้อย่างไรใน Part นี้ |
|---|---|
| `csrf.js` + `apiFetch()` (513.2-513.3) | แนบ CSRF token อัตโนมัติทุกครั้งที่อัปโหลด |
| `escapeHtml()` (512.4/514.2) | แสดงชื่อไฟล์ที่ผู้ใช้เลือกอย่างปลอดภัยจาก XSS |
| Loading/disabled state ระหว่างรอ response (515.5) | ปิดปุ่ม/dropzone ระหว่างกำลังอัปโหลด |
| Error handling pattern (`formatApiError`, 514.1) | แสดง error จาก validator (571, 577) ให้ผู้ใช้เห็นชัดเจน |

### 578.2 HTML โครงสร้าง Dropzone

```html
<!-- blog/templates/blog/post_upload_dropzone.html -->
{% extends "base.html" %}
{% load static %}

{% block extra_css %}
    <link rel="stylesheet" href="{% static 'blog/css/dropzone.css' %}">
{% endblock %}

{% block content %}
<div class="dropzone" id="dropzone" data-post-id="{{ post.id }}">
    <input type="file" id="file-input" accept="image/jpeg,image/png,image/webp" hidden>
    <div class="dropzone__prompt" id="dropzone-prompt">
        <p>ลากไฟล์รูปภาพมาวางที่นี่</p>
        <p>หรือ <button type="button" id="browse-button">เลือกไฟล์</button></p>
        <p class="dropzone__hint">รองรับ .jpg, .png, .webp ขนาดไม่เกิน 5 MB</p>
    </div>
    <div class="dropzone__preview" id="dropzone-preview" hidden>
        <img id="preview-image" alt="ตัวอย่างรูปภาพ">
        <div class="progress-track">
            <div class="progress-bar" id="dropzone-progress-bar" style="width: 0%;"></div>
        </div>
    </div>
    <p id="dropzone-error" class="error-message" hidden></p>
</div>
{% endblock %}

{% block extra_js %}
    <script src="{% static 'blog/js/csrf.js' %}" defer></script>
    <script src="{% static 'blog/js/api.js' %}" defer></script>
    <script src="{% static 'blog/js/dropzone-upload.js' %}" defer></script>
{% endblock %}
```

### 578.3 CSS ที่ทำให้ Dropzone ใช้งานง่าย

```css
/* blog/static/blog/css/dropzone.css */
.dropzone {
    border: 2px dashed #cbd5e0;
    border-radius: 12px;
    padding: 2.5rem;
    text-align: center;
    background-color: #f7fafc;
    transition: border-color 0.2s ease, background-color 0.2s ease;
    cursor: pointer;
}

/* class นี้ถูกเพิ่ม/ลบด้วย JavaScript ตอน dragenter/dragleave */
.dropzone--dragover {
    border-color: #2b6cb0;
    background-color: #ebf8ff;
}

.dropzone--uploading {
    pointer-events: none;
    opacity: 0.7;
}

.dropzone__hint {
    font-size: 0.85rem;
    color: #718096;
}

.dropzone__preview img {
    max-width: 100%;
    max-height: 240px;
    border-radius: 8px;
    margin-bottom: 1rem;
}
```

### 578.4 JavaScript: จัดการ Drag Events และอัปโหลด

```javascript
// blog/static/blog/js/dropzone-upload.js

const dropzone = document.getElementById("dropzone");

if (dropzone) {
    const fileInput = document.getElementById("file-input");
    const browseButton = document.getElementById("browse-button");
    const promptEl = document.getElementById("dropzone-prompt");
    const previewEl = document.getElementById("dropzone-preview");
    const previewImage = document.getElementById("preview-image");
    const progressBar = document.getElementById("dropzone-progress-bar");
    const errorBox = document.getElementById("dropzone-error");
    const postId = dropzone.dataset.postId;

    const ALLOWED_TYPES = ["image/jpeg", "image/png", "image/webp"];
    const MAX_SIZE = 5 * 1024 * 1024;

    // เปิด file picker เมื่อคลิกปุ่ม "เลือกไฟล์" หรือคลิกที่ dropzone เอง
    browseButton.addEventListener("click", () => fileInput.click());
    dropzone.addEventListener("click", (event) => {
        if (event.target === browseButton) return;   // ป้องกัน trigger ซ้ำซ้อน
        fileInput.click();
    });

    fileInput.addEventListener("change", () => {
        if (fileInput.files.length > 0) {
            handleFile(fileInput.files[0]);
        }
    });

    // --- Drag & Drop Events ---
    // ต้อง preventDefault() ทุก event ของ dragover เสมอ ไม่เช่นนั้น browser จะเปิดไฟล์
    // ในแท็บใหม่แทนที่จะยอมให้ drop ลงใน element ของเรา (พฤติกรรม default ของ browser)
    ["dragenter", "dragover"].forEach((eventName) => {
        dropzone.addEventListener(eventName, (event) => {
            event.preventDefault();
            event.stopPropagation();
            dropzone.classList.add("dropzone--dragover");
        });
    });

    ["dragleave", "drop"].forEach((eventName) => {
        dropzone.addEventListener(eventName, (event) => {
            event.preventDefault();
            event.stopPropagation();
            dropzone.classList.remove("dropzone--dragover");
        });
    });

    dropzone.addEventListener("drop", (event) => {
        const files = event.dataTransfer.files;
        if (files.length > 0) {
            handleFile(files[0]);
        }
    });

    function showError(message) {
        errorBox.textContent = message;
        errorBox.hidden = false;
    }

    function validateFileClientSide(file) {
        // Client-side validation ให้ feedback ทันทีโดยไม่ต้องรอ round-trip ไปเซิร์ฟเวอร์
        // แต่ "ไม่ใช่" ด่านความปลอดภัยจริง — server ต้อง validate ซ้ำเสมอ (ทบทวน 577)
        if (!ALLOWED_TYPES.includes(file.type)) {
            return "ประเภทไฟล์ไม่ได้รับอนุญาต ใช้ได้เฉพาะ .jpg, .png, .webp";
        }
        if (file.size > MAX_SIZE) {
            return `ไฟล์มีขนาด ${(file.size / 1024 / 1024).toFixed(1)} MB เกินขนาดสูงสุด 5 MB`;
        }
        return null;
    }

    async function handleFile(file) {
        errorBox.hidden = true;

        const clientError = validateFileClientSide(file);
        if (clientError) {
            showError(clientError);
            return;
        }

        // แสดง preview รูปทันทีจาก local file (ไม่ต้องรอเซิร์ฟเวอร์)
        const localPreviewUrl = URL.createObjectURL(file);
        previewImage.src = localPreviewUrl;
        promptEl.hidden = true;
        previewEl.hidden = false;
        dropzone.classList.add("dropzone--uploading");

        try {
            await uploadWithProgress(file);
        } catch (error) {
            showError(`อัปโหลดไม่สำเร็จ: ${error.message}`);
            promptEl.hidden = false;
            previewEl.hidden = true;
        } finally {
            dropzone.classList.remove("dropzone--uploading");
            URL.revokeObjectURL(localPreviewUrl);   // คืน memory ที่ browser จองไว้ให้ preview
        }
    }

    function uploadWithProgress(file) {
        return new Promise((resolve, reject) => {
            const xhr = new XMLHttpRequest();
            const formData = new FormData();
            formData.append("cover_image", file);
            formData.append("post_id", postId);

            xhr.open("POST", "/posts/upload-cover/");
            xhr.setRequestHeader("X-CSRFToken", getCookie("csrftoken"));

            xhr.upload.addEventListener("progress", (event) => {
                if (event.lengthComputable) {
                    const percent = Math.round((event.loaded / event.total) * 100);
                    progressBar.style.width = `${percent}%`;
                }
            });

            xhr.addEventListener("load", () => {
                if (xhr.status >= 200 && xhr.status < 300) {
                    const data = JSON.parse(xhr.responseText);
                    previewImage.src = data.thumbnail_url;   // เปลี่ยนเป็นรูปจริงจากเซิร์ฟเวอร์
                    resolve(data);
                } else {
                    const errorData = JSON.parse(xhr.responseText);
                    reject(new Error(formatApiError(errorData)));
                }
            });

            xhr.addEventListener("error", () => reject(new Error("การเชื่อมต่อขัดข้อง")));
            xhr.send(formData);
        });
    }
}
```

### 578.5 View ฝั่งเซิร์ฟเวอร์ที่รับไฟล์จาก Dropzone

```python
# blog/views.py (เพิ่มเติม)
from django.core.exceptions import ValidationError
from django.http import JsonResponse
from django.views import View

from .models import Post


class UploadCoverImageView(View):
    def post(self, request):
        post = Post.objects.get(pk=request.POST.get("post_id"))
        uploaded_file = request.FILES.get("cover_image")

        if not uploaded_file:
            return JsonResponse({"detail": "ไม่พบไฟล์ที่อัปโหลด"}, status=400)

        post.cover_image = uploaded_file

        try:
            post.full_clean()   # รัน validator ทั้งหมดจากขั้นตอนที่ 571 และ 577
        except ValidationError as exc:
            return JsonResponse(exc.message_dict, status=400)

        post.save()

        return JsonResponse({
            "success": True,
            "thumbnail_url": post.cover_thumbnail.url,
        })
```

จุดสำคัญ: **client-side validation (578.4) ไม่เคยแทนที่ server-side validation ได้เลย**
โค้ด JavaScript ตรวจสอบแค่เพื่อ **ประสบการณ์ผู้ใช้ที่ดีขึ้น** (แจ้ง error ทันทีไม่ต้องรอ
round-trip) ส่วนความถูกต้องและปลอดภัยที่แท้จริงต้องพึ่ง `full_clean()` ที่รัน validator
ทั้งหมดจากขั้นตอนที่ 571-577 เสมอ — หลักการนี้ตรงกับที่ Part 052 ขั้นตอนที่ 515.5 เคยเตือน
ไว้เรื่อง double submission เช่นกัน: **client-side คือ UX, server-side คือความปลอดภัย**

---

## ขั้นตอนที่ 579: จัดการไฟล์ Media ที่ไม่มีใครอ้างอิงแล้ว (Orphaned Files)

### 579.1 ปัญหา: ไฟล์ที่ถูก "ทิ้งค้าง" บนดิสก์

ไฟล์ media กลายเป็น **orphaned file** (ไฟล์กำพร้า) ได้จากหลายสาเหตุ:

- ผู้ใช้ลบ record `Post` ทิ้ง — Django **ไม่ลบไฟล์รูปภาพที่ผูกกับ record นั้นให้อัตโนมัติ**
  (เป็นพฤติกรรม default ที่ตั้งใจ เพื่อป้องกันการลบไฟล์โดยไม่ตั้งใจจาก signal ที่ผิดพลาด)
- ผู้ใช้อัปโหลดรูปใหม่ทับรูปเดิม (`cover_image` ถูกเปลี่ยนค่า) — ไฟล์เก่ายังอยู่บนดิสก์
  แต่ไม่มี record ไหนชี้มาที่มันอีกแล้ว
- Chunked upload (ขั้นตอนที่ 574) ที่ผู้ใช้อัปโหลดค้างไว้ไม่เสร็จ แล้วปิดหน้าเว็บไปเฉย ๆ
  ไฟล์ temp ยังค้างอยู่ใน `chunked_uploads/`

ถ้าไม่จัดการ ปัญหานี้จะสะสมและกิน storage โดยไม่มีประโยชน์ใด ๆ เพิ่มขึ้นเรื่อย ๆ ตามอายุ
ของระบบ

### 579.2 ทำไมไม่ใช้ Signal ลบไฟล์อัตโนมัติแบบ Naive

วิธีที่มือใหม่มักลองก่อนคือใช้ `post_delete` signal ลบไฟล์ทันทีที่ record ถูกลบ:

```python
# ตัวอย่างที่มีความเสี่ยง — ไม่แนะนำให้ใช้แบบนี้ตรง ๆ
from django.db.models.signals import post_delete
from django.dispatch import receiver


@receiver(post_delete, sender=Post)
def delete_files_on_post_delete(sender, instance, **kwargs):
    if instance.cover_image:
        instance.cover_image.delete(save=False)
```

วิธีนี้มีความเสี่ยงซ่อนอยู่: ถ้าลบ record ภายใน **database transaction ที่ rollback ในภาย
หลัง** (เช่น error เกิดขึ้นหลังจากนั้นในโค้ดเดียวกัน) ไฟล์จะถูกลบไปแล้วจริง ๆ บนดิสก์ แต่
record ในฐานข้อมูลกลับไม่ได้ถูกลบ (เพราะ rollback) ทำให้เกิดสถานะที่ไม่สอดคล้องกัน
(database บอกว่ามีรูป แต่ไฟล์จริงหายไปแล้ว) — ด้วยเหตุนี้ แนวทางที่ปลอดภัยกว่าคือ **แยก
กระบวนการทำความสะอาดไฟล์ออกจาก request-response cycle** โดยใช้ management command ที่รัน
เป็นระยะแทน (เช่นผ่าน cron job หรือ Celery Beat ในอนาคต)

### 579.3 เขียน Management Command ทำความสะอาด

```python
# blog/management/commands/cleanup_orphaned_media.py
import os
from datetime import timedelta

from django.conf import settings
from django.core.management.base import BaseCommand
from django.utils import timezone

from blog.models import ChunkedUpload, Post


class Command(BaseCommand):
    help = "ค้นหาและลบไฟล์ media ที่ไม่มี record ใดในฐานข้อมูลอ้างอิงอยู่แล้ว"

    def add_arguments(self, parser):
        parser.add_argument(
            "--dry-run",
            action="store_true",
            help="แสดงรายการไฟล์ที่จะถูกลบ โดยไม่ลบจริง",
        )
        parser.add_argument(
            "--min-age-hours",
            type=int,
            default=24,
            help="ลบเฉพาะไฟล์ที่เก่ากว่าจำนวนชั่วโมงนี้เท่านั้น (ป้องกันลบไฟล์ที่เพิ่ง"
                 " อัปโหลดแต่ยังไม่ทันบันทึกลง DB ในทันที เช่นตอน request กำลังประมวลผลอยู่)",
        )

    def handle(self, *args, **options):
        dry_run = options["dry_run"]
        min_age = timedelta(hours=options["min_age_hours"])
        cutoff_time = timezone.now() - min_age

        self._cleanup_unreferenced_files(dry_run, cutoff_time)
        self._cleanup_stale_chunked_uploads(dry_run, cutoff_time)

    def _get_referenced_file_paths(self):
        """รวบรวม path ของไฟล์ทั้งหมดที่ถูกอ้างอิงอยู่จริงในฐานข้อมูล ณ ขณะนี้"""
        referenced = set()

        for post in Post.objects.all():
            if post.cover_image:
                referenced.add(post.cover_image.name)
            if post.attachment:
                referenced.add(post.attachment.name)

        return referenced

    def _cleanup_unreferenced_files(self, dry_run, cutoff_time):
        referenced_paths = self._get_referenced_file_paths()
        media_root = settings.MEDIA_ROOT

        # โฟลเดอร์ย่อยที่ควรตรวจสอบ (ข้าม CACHE/ ของ django-imagekit เพราะเป็น cache
        # ที่ regenerate ใหม่ได้เสมอ ไม่ใช่ต้นฉบับ — ลบได้ด้วยคำสั่งของ imagekit เอง
        # ตามขั้นตอนที่ 573.6 แยกต่างหาก)
        scan_dirs = ["blog/covers", "blog/attachments"]

        deleted_count = 0
        total_freed_bytes = 0

        for scan_dir in scan_dirs:
            full_scan_path = os.path.join(media_root, scan_dir)
            if not os.path.isdir(full_scan_path):
                continue

            for root, _dirs, files in os.walk(full_scan_path):
                for file_name in files:
                    absolute_path = os.path.join(root, file_name)
                    relative_path = os.path.relpath(absolute_path, media_root)

                    if relative_path in referenced_paths:
                        continue   # ไฟล์นี้ยังมี record อ้างอิงอยู่ — ข้าม

                    file_modified_time = timezone.make_aware(
                        timezone.datetime.fromtimestamp(os.path.getmtime(absolute_path))
                    )
                    if file_modified_time > cutoff_time:
                        continue   # ไฟล์ยังใหม่เกินไป อาจกำลังถูกประมวลผลอยู่ ข้ามไปก่อน

                    file_size = os.path.getsize(absolute_path)

                    if dry_run:
                        self.stdout.write(f"[DRY RUN] จะลบ: {relative_path} ({file_size} bytes)")
                    else:
                        os.remove(absolute_path)
                        self.stdout.write(self.style.WARNING(f"ลบแล้ว: {relative_path}"))

                    deleted_count += 1
                    total_freed_bytes += file_size

        action_word = "จะลบ" if dry_run else "ลบไปแล้ว"
        self.stdout.write(self.style.SUCCESS(
            f"{action_word} {deleted_count} ไฟล์ (คิดเป็น {total_freed_bytes / 1024 / 1024:.2f} MB)"
        ))

    def _cleanup_stale_chunked_uploads(self, dry_run, cutoff_time):
        """ลบ record + ไฟล์ temp ของการอัปโหลดแบบ chunk ที่ค้างไม่เสร็จมานานเกินไป"""
        stale_uploads = ChunkedUpload.objects.filter(
            status=ChunkedUpload.STATUS_UPLOADING,
            created_at__lt=cutoff_time,
        )

        for upload in stale_uploads:
            if dry_run:
                self.stdout.write(f"[DRY RUN] จะลบ chunked upload ค้าง: {upload.file_name}")
                continue

            if os.path.exists(upload.temp_path):
                os.remove(upload.temp_path)
            upload.delete()
            self.stdout.write(self.style.WARNING(f"ลบ chunked upload ค้าง: {upload.file_name}"))
```

### 579.4 ทดสอบและใช้งานคำสั่ง

```bash
# ดูก่อนว่าจะลบอะไรบ้าง โดยยังไม่ลบจริง (สำคัญมาก — รันแบบนี้ก่อนเสมอในการใช้งานครั้งแรก)
python manage.py cleanup_orphaned_media --dry-run

# ลบไฟล์ที่ orphan มานานกว่า 48 ชั่วโมงจริง ๆ
python manage.py cleanup_orphaned_media --min-age-hours 48
```

### 579.5 ตั้งเวลารันอัตโนมัติด้วย Cron

```bash
# แก้ไข crontab ของระบบ (crontab -e) เพื่อรันคำสั่งนี้ทุกวันตอนตี 3
0 3 * * * cd /path/to/django-mastery-course && /path/to/venv/bin/python manage.py cleanup_orphaned_media --min-age-hours 48 >> /var/log/media-cleanup.log 2>&1
```

การตั้งเวลารันอัตโนมัติเช่นนี้เป็น pattern เดียวกับที่ระบบ production จริงใช้จัดการงาน
บำรุงรักษาเป็นระยะ (housekeeping tasks) — เราจะเรียนรู้ทางเลือกที่ทันสมัยกว่า cron อย่าง
**Celery Beat** อย่างเต็มรูปแบบใน Phase 10 (Performance & Scalability) ซึ่งรองรับการ
monitor, retry, และจัดการ error ได้ดีกว่า cron ธรรมดามาก

### 579.6 ข้อควรระวังสำคัญก่อนใช้งานจริง

| ข้อควรระวัง | เหตุผล |
|---|---|
| รัน `--dry-run` ก่อนเสมอในสภาพแวดล้อมใหม่ | ป้องกันการลบไฟล์ที่จำเป็นโดยไม่ตั้งใจจาก bug ในโค้ด scan |
| ตั้ง `--min-age-hours` ให้เหมาะสม ไม่ต่ำเกินไป | ไฟล์ที่เพิ่งอัปโหลดอาจยังไม่ถูกบันทึก path ลง DB ทันที (เช่น request ที่ยังประมวลผลไม่เสร็จ) |
| Backup ก่อนรันครั้งแรกในระบบเก่าที่ไม่เคยรันมาก่อน | ระบบที่สะสมไฟล์ orphan มานานอาจมีจำนวนมาก ควร sample ตรวจสอบด้วยตาก่อนลบจริง |
| ห้ามรันคำสั่งนี้ระหว่างมีการ migrate ข้อมูลไฟล์ครั้งใหญ่ | เช่นระหว่างย้ายจาก local storage ไป S3 (Part นี้ขั้นตอนที่ 575) ที่ path อาจอยู่ระหว่างเปลี่ยนผ่าน |

---

## ขั้นตอนที่ 580: สรุป Phase 6 ทั้งหมด (Part 051-058) และคำนำสู่ Phase 7

### 580.1 ภาพรวม Phase 6: Frontend Integration

Phase 6 เป็นจุดเปลี่ยนสำคัญของหลักสูตร — จาก Phase 1-5 ที่เน้น **ฝั่ง Backend ล้วน ๆ**
(Model, View, Template, REST API) เราได้ขยายไปสู่ **การเชื่อมต่อ Frontend** ในหลากหลาย
รูปแบบ ตั้งแต่เบาที่สุด (Bootstrap) ไปจนถึงหนักที่สุด (React/Vue SPA) แล้วปิดท้ายด้วยเรื่อง
media ที่จำเป็นสำหรับทุกเว็บแอปพลิเคชันจริง

```
Part 051 (Bootstrap/CSS) ─┐
Part 052 (Fetch API)      ├─ Frontend "เบา": ยังใช้ Django Template เป็นแกนหลัก
Part 053 (HTMX)           ├─ เพิ่ม interactivity แบบค่อยเป็นค่อยไป (progressive enhancement)
Part 054 (Alpine.js)      ─┘
Part 055 (React)          ─┐
Part 056 (Vue.js)          ├─ Frontend "หนัก": Django เป็น API-only backend, SPA แยก build
Part 057 (Channels)        ┘  + WebSocket สำหรับ real-time features
Part 058 (Media Handling) ── โครงสร้างพื้นฐานที่ทุกรูปแบบ frontend ข้างต้นต้องใช้ร่วมกัน
```

### 580.2 ตารางสรุปแต่ละ Part ใน Phase 6

| Part | หัวข้อหลัก | สิ่งที่ได้เรียนรู้ | เทคโนโลยี/Library หลัก |
|---|---|---|---|
| 051 | Bootstrap และ CSS Framework | Responsive grid, component สำเร็จรูป, การ customize theme | Bootstrap 5 |
| 052 | JavaScript และ Fetch API | `fetch()`, CSRF ผ่าน header, async/await, debounce, ES Modules | Vanilla JavaScript |
| 053 | HTMX | ส่ง HTML fragment แทน JSON, `hx-get`/`hx-post`, ลด JavaScript ที่ต้องเขียนเอง | HTMX |
| 054 | Alpine.js | Reactive state แบบเบา ผูกกับ DOM โดยตรงในหน้า Template | Alpine.js |
| 055 | React | Django เป็น API backend ล้วน, React SPA แยก build, JWT authentication | React, Django REST Framework |
| 056 | Vue.js | รูปแบบเดียวกับ React แต่ใช้ Vue ecosystem (Composition API, Pinia) | Vue.js, Django REST Framework |
| 057 | Django Channels | WebSocket, ASGI, Consumer, real-time notification/chat | Django Channels, Redis |
| 058 | Media Handling | Validation, Pillow, imagekit, chunked/S3 upload, security, cleanup | Pillow, django-imagekit, boto3 |

### 580.3 ทบทวนแนวคิดเชื่อมโยงข้ามทั้ง Phase

หัวใจของ Phase 6 ที่ทุก Part ใช้ร่วมกันคือ **"Django จะทำหน้าที่อะไร เมื่อ frontend ฉลาด
ขึ้นเรื่อย ๆ"**:

| ระดับความฉลาดของ Frontend | Django ทำหน้าที่อะไร | ตัวอย่างจาก Phase 6 |
|---|---|---|
| Server-rendered ล้วน (Part 001-050) | Render HTML เต็มรูปแบบทุก request | Template engine, `render()` |
| เพิ่ม CSS ให้สวยขึ้น | Serve static files เท่านั้น | Part 051 |
| เพิ่ม JS เรียก API เอง | เป็นทั้ง Template server และ REST API server พร้อมกัน | Part 052-054 |
| Frontend แยก SPA เต็มตัว | เป็นแค่ API backend, ไม่ render HTML ของหน้าเว็บอีกต่อไป | Part 055-056 |
| ต้องการ real-time | เปิด WebSocket connection คู่ขนานกับ HTTP | Part 057 |
| ไม่ว่ารูปแบบไหน ก็ต้องรับ/ประมวลผล/ปกป้องไฟล์ที่ผู้ใช้อัปโหลด | Media handling เป็นเลเยอร์ที่ใช้ร่วมกันเสมอ ไม่ขึ้นกับว่า frontend เป็นแบบไหน | Part 058 |

### 580.4 Quiz ทบทวนความเข้าใจ Phase 6 (10 ข้อ พร้อมเฉลย)

**คำถามที่ 1**: ทำไม `fetch()` ถึงไม่ reject Promise เมื่อได้รับ response สถานะ 404 หรือ 500?

<details>
<summary>เฉลย</summary>

`fetch()` ออกแบบมาให้ resolve เป็น success ตราบใดที่ได้รับ HTTP response กลับมาจริง
(ไม่ว่า status code จะเป็นอะไร) มันจะ reject ก็ต่อเมื่อเกิดปัญหาระดับเครือข่ายเท่านั้น
(เช่น DNS หาเซิร์ฟเวอร์ไม่เจอ, CORS บล็อก) จึงต้องเช็ค `response.ok` เองเสมอทุกครั้ง
(Part 052 ขั้นตอนที่ 512.2)
</details>

**คำถามที่ 2**: `SessionAuthentication` กับ `TokenAuthentication`/`JWTAuthentication` ต่างกัน
อย่างไรในเรื่องการแนบ CSRF token?

<details>
<summary>เฉลย</summary>

`SessionAuthentication` ใช้ cookie `sessionid` ซึ่งเสี่ยงต่อ CSRF attack จึง**ต้อง**แนบ
`X-CSRFToken` header เสมอสำหรับ unsafe method ส่วน Token/JWT authentication ใช้ header
`Authorization` ที่ browser ไม่แนบอัตโนมัติให้เว็บอื่น จึงไม่มีความเสี่ยง CSRF และไม่ต้อง
แนบ token (Part 052 ขั้นตอนที่ 513.1)
</details>

**คำถามที่ 3**: HTMX ต่างจาก React/Vue อย่างไรในเชิงปรัชญาการทำงาน?

<details>
<summary>เฉลย</summary>

HTMX ให้เซิร์ฟเวอร์ (Django) ส่ง **HTML fragment** กลับมาโดยตรง แล้ว swap เข้า DOM ตรง ๆ
ผ่าน attribute อย่าง `hx-get`/`hx-target` ไม่ต้องเขียน JavaScript render UI เอง ในขณะที่
React/Vue ให้เซิร์ฟเวอร์ส่งแค่ **JSON data** กลับมา แล้วให้ JavaScript framework ฝั่ง client
เป็นคนตัดสินใจ render UI ทั้งหมดเอง (Part 053, 055-056)
</details>

**คำถามที่ 4**: ทำไมการเชื่อมต่อ React/Vue เข้ากับ Django ถึงทำให้ Django "เป็นแค่ API
backend"?

<details>
<summary>เฉลย</summary>

เพราะ React/Vue SPA ถูก build แยกต่างหากเป็น static bundle (JS/CSS) ที่ browser โหลดมา
รันเองทั้งหมด แล้ว SPA จะคุยกับ Django ผ่าน REST API (JSON) เท่านั้น Django จึงไม่ต้อง
render HTML ของหน้าเว็บผ่าน Template engine อีกต่อไป มีหน้าที่แค่ตอบ JSON (Part 055-056)
</details>

**คำถามที่ 5**: Django Channels ใช้โปรโตคอลอะไรที่ต่างจาก HTTP ธรรมดา และทำไมถึงจำเป็น
สำหรับ real-time feature?

<details>
<summary>เฉลย</summary>

Django Channels ใช้ **WebSocket** ซึ่งเป็น connection แบบ full-duplex ที่เปิดค้างไว้
ต่อเนื่อง (ต่างจาก HTTP ที่เป็น request-response แบบครั้งเดียวจบ) ทำให้เซิร์ฟเวอร์สามารถ
**push ข้อมูลไปหา client ได้เองโดยไม่ต้องรอ client ถามก่อน** ซึ่งจำเป็นสำหรับฟีเจอร์อย่าง
แชทหรือ notification real-time (Part 057)
</details>

**คำถามที่ 6**: การเรียก `post.save()` ตรง ๆ กับการเรียก `post.full_clean()` ก่อน `save()`
ต่างกันอย่างไรในแง่ของ validator ที่ผูกกับ Model field?

<details>
<summary>เฉลย</summary>

Validator ที่ผูกไว้กับ Model field (เช่น `validate_image_file_size` ใน ขั้นตอนที่ 571)
จะทำงานก็ต่อเมื่อเรียก `full_clean()` เท่านั้น การเรียก `.save()` ตรง ๆ **จะไม่รัน
validator เหล่านี้เลย** — ต้องเรียก `full_clean()` เอง หรือผ่าน `ModelForm.is_valid()`
ที่เรียกให้อัตโนมัติ
</details>

**คำถามที่ 7**: ทำไมการตรวจสอบแค่นามสกุลไฟล์ (`.jpg`, `.png`) จึงไม่เพียงพอต่อความปลอดภัย?

<details>
<summary>เฉลย</summary>

เพราะนามสกุลไฟล์เป็นแค่ชื่อที่ผู้ใช้ตั้งเอง ไม่เกี่ยวข้องกับเนื้อหาจริงข้างในไฟล์
ผู้ใช้ประสงค์ร้ายสามารถเปลี่ยนชื่อไฟล์อันตราย (เช่น `.php`) ให้ลงท้ายด้วย `.jpg` ได้ ต้อง
ตรวจสอบ **magic bytes** หรือให้ Pillow เปิดไฟล์จริงเพื่อยืนยันเนื้อหาที่แท้จริงด้วย
(ขั้นตอนที่ 577)
</details>

**คำถามที่ 8**: เพราะเหตุใด `fetch()` จึงไม่เหมาะกับการติดตาม upload progress และต้องใช้
อะไรแทน?

<details>
<summary>เฉลย</summary>

`fetch()` ไม่มี event ติดตาม upload progress โดยตรง (รองรับแค่ download progress แบบ
ซับซ้อนผ่าน `ReadableStream`) ต้องใช้ `XMLHttpRequest` (XHR) แทน เพราะมี event
`xhr.upload.addEventListener('progress', ...)` ที่ใช้งานง่ายกว่ามาก (ขั้นตอนที่ 574.3)
</details>

**คำถามที่ 9**: Direct-to-S3 Upload ด้วย Presigned URL ช่วยแก้ปัญหาอะไรที่การอัปโหลดผ่าน
Django server ธรรมดามี?

<details>
<summary>เฉลย</summary>

ช่วยลดภาระของ Django server เพราะไฟล์วิ่งจาก browser ตรงไปยัง S3 โดยไม่ผ่าน Django
worker เลย ทำให้ server ไม่ถูก "ยึดครอง" ระหว่างที่ไฟล์กำลังอัปโหลด ไม่เปลือง bandwidth
ของเซิร์ฟเวอร์ และ scale ได้ดีกว่าเมื่อมีผู้ใช้อัปโหลดพร้อมกันจำนวนมาก (ขั้นตอนที่ 575.1)
</details>

**คำถามที่ 10**: ทำไมการลบไฟล์ orphaned media จึงควรทำผ่าน management command ที่รันเป็น
ระยะ แทนที่จะใช้ `post_delete` signal ลบทันที?

<details>
<summary>เฉลย</summary>

เพราะการลบไฟล์ใน signal ที่ทำงานภายใน transaction ที่อาจ rollback ภายหลัง จะทำให้เกิด
สถานะไม่สอดคล้องกัน (ไฟล์ถูกลบจริงบนดิสก์ แต่ database บอกว่า record ยังอยู่ เพราะ
transaction rollback) การแยกกระบวนการทำความสะอาดออกจาก request-response cycle
ด้วย management command ที่รันเป็นระยะจึงปลอดภัยกว่า (ขั้นตอนที่ 579.2)
</details>

**คำถามที่ 11 (โบนัส)**: เปรียบเทียบข้อดี-ข้อเสียของการ resize รูปภาพเอง (override
`save()`) เทียบกับใช้ `django-imagekit`

<details>
<summary>เฉลย</summary>

การเขียนเองควบคุมได้ 100% แต่ประมวลผลทุกขนาดทันที (eager) ทำให้อัปโหลดช้าลง และถ้าต้อง
เปลี่ยนขนาด variant ในอนาคตต้อง migrate ข้อมูลเก่าเอง ส่วน `django-imagekit` ใช้แนวคิด
lazy generation (สร้างเมื่อถูกเรียกใช้ครั้งแรกแล้ว cache ไว้) ทำให้อัปโหลดเร็วกว่า และ
เปลี่ยนขนาด variant ได้ง่ายแค่แก้ processor แล้ว clear cache (ขั้นตอนที่ 572-573)
</details>

### 580.5 แบบฝึกหัดใหญ่ปิดท้าย Phase 6

สร้างฟีเจอร์ **"แกลเลอรีรูปภาพของบทความ" (Post Gallery)** ที่รวมทุกแนวคิดจาก Phase 6
เข้าด้วยกัน โดยมีข้อกำหนดดังนี้:

1. **Model**: สร้าง Model `PostImage` ที่มีความสัมพันธ์แบบ `ForeignKey` ไปยัง `Post`
   (หนึ่งบทความมีได้หลายรูป) พร้อม `ImageField` ที่ผ่าน validator ขนาดไฟล์และนามสกุล
   (ทบทวนขั้นตอนที่ 571) และเพิ่ม `ImageSpecField` สร้าง thumbnail ด้วย django-imagekit
   (ขั้นตอนที่ 573)

2. **Drag-and-drop UI**: สร้างหน้าที่ผู้ใช้ลากรูปหลายไฟล์มาวางพร้อมกันได้ (ต่อยอดจาก
   ขั้นตอนที่ 578 ให้รองรับ multiple files แทนที่จะรับแค่ไฟล์เดียว) แสดง progress bar
   แยกของแต่ละไฟล์

3. **Security**: เพิ่ม validator ตรวจสอบ magic bytes (ขั้นตอนที่ 577.2) และให้ Pillow
   ยืนยันว่าเป็นรูปภาพจริง (577.3) ก่อนบันทึกทุกครั้ง

4. **Frontend เลือกได้อย่างใดอย่างหนึ่ง** (เลือกทำแค่ทางใดทางหนึ่งตามที่ถนัด):
   - ใช้ **HTMX** (ทบทวน Part 053) ให้ view คืน HTML fragment ของแกลเลอรีที่อัปเดตแล้ว
     กลับมาโดยตรง แทนการยิง JSON แล้ว render เอง
   - หรือใช้ **Alpine.js** (ทบทวน Part 054) จัดการ state ของแกลเลอรี (แสดง/ซ่อน modal
     ดูรูปขยาย, ลบรูปออกจาก DOM ทันทีที่ลบสำเร็จ)

5. **Cleanup**: ปรับ management command จากขั้นตอนที่ 579 ให้ครอบคลุมโฟลเดอร์ของ
   `PostImage` ด้วย

6. **ทดสอบด้วยตัวเอง**: อัปโหลดไฟล์ที่ไม่ใช่รูปภาพแต่เปลี่ยนนามสกุลเป็น `.jpg` เพื่อยืนยัน
   ว่า validator จากข้อ 3 ปฏิเสธไฟล์นั้นจริง แล้วบันทึกผลลัพธ์ที่เห็น

แบบฝึกหัดนี้ครอบคลุมทั้ง Model design, validation, image processing, JavaScript upload UX,
security และ maintenance — ครบทุกแนวคิดหลักของ Part 058 และเชื่อมโยงกลับไปยัง Part 053/054
ของ Phase 6 ทั้งหมด

### 580.6 Checklist ก่อนไป Phase ถัดไป

- [ ] เข้าใจว่าทำไม validator ที่ผูกกับ Model field ต้องเรียกผ่าน `full_clean()`/`ModelForm`
- [ ] เขียนโค้ด resize รูปภาพด้วย Pillow เองได้ และอธิบายได้ว่าทำไมต้องใช้ `save=False`
- [ ] ติดตั้งและใช้งาน `django-imagekit` สร้าง image variant หลายขนาดได้
- [ ] อธิบายความแตกต่างระหว่าง chunked upload กับ direct-to-S3 upload ได้
- [ ] รู้ว่าทำไมต้องตรวจสอบ magic bytes แทนการเชื่อนามสกุลไฟล์เพียงอย่างเดียว
- [ ] สร้าง UI drag-and-drop upload ที่เชื่อมกับ Fetch/XHR ได้จริง
- [ ] เขียน management command ทำความสะอาดไฟล์ orphaned ได้
- [ ] ทำแบบฝึกหัดใหญ่ท้าย Phase (580.5) สำเร็จอย่างน้อย 4 ใน 6 ข้อ

### 580.7 คำนำสู่ Phase 7: Testing & Quality Assurance

ตลอด Phase 1-6 ที่ผ่านมา (058 Part, 580 ขั้นตอน) เราสร้างฟีเจอร์จำนวนมหาศาล — Model,
View, Form, REST API, Authentication, Frontend integration หลายรูปแบบ, และระบบจัดการไฟล์
เต็มรูปแบบ แต่มีคำถามสำคัญข้อหนึ่งที่เรายังไม่ได้ตอบอย่างเป็นระบบ: **"เรารู้ได้อย่างไรว่า
โค้ดทั้งหมดที่เขียนมายังทำงานถูกต้องอยู่ หลังจากแก้ไขโค้ดใหม่ไปเรื่อย ๆ?"**

จนถึงตอนนี้เราตรวจสอบโค้ดด้วยการ **ทดสอบด้วยมือ** (manual testing) — เปิด browser, ลอง
คลิก, ลองอัปโหลดไฟล์, ดู console — วิธีนี้ใช้ได้ตอนโปรเจกต์เล็ก แต่เมื่อโปรเจกต์โตขึ้น
เรื่อย ๆ (ซึ่งของเราตอนนี้มีโค้ดมากกว่า 580 ขั้นตอนแล้ว) การทดสอบด้วยมือทุกครั้งที่แก้โค้ด
กลายเป็นเรื่องที่ทำไม่ไหวและเสี่ยงพลาดสูงมาก

**Phase 7: Testing & Quality Assurance (Part 059-065, ขั้นตอนที่ 581-650)** จะพาคุณเข้าสู่
โลกของ **automated testing** อย่างเต็มรูปแบบ:

- **Part 059**: Django Testing เบื้องต้นด้วย `unittest` — `TestCase`, `Client`, assertion
  พื้นฐานที่ทุกโปรเจกต์ Django ต้องมี
- **Part 060**: การเขียนเทสสำหรับ Views, Models และ Forms อย่างครบถ้วน
- **Part 061**: `pytest-django` และ Fixtures — เครื่องมือทดสอบที่ทีมมืออาชีพส่วนใหญ่เลือกใช้
- **Part 062**: Test Coverage และ Mocking — วัดว่าโค้ดส่วนไหนยังไม่มีเทสคลุมถึง และจำลอง
  dependency ภายนอก (เช่น S3, third-party API) โดยไม่ต้องเรียกจริง
- **Part 063-065**: เจาะลึกต่อไปถึง Integration testing, Testing REST API แบบเต็มรูปแบบ,
  และ CI pipeline พื้นฐานที่รันเทสอัตโนมัติทุกครั้งที่ push โค้ด

สิ่งที่คุณสร้างไว้ตลอด Phase 6 นี้ — โดยเฉพาะ validator ของไฟล์อัปโหลด (ขั้นตอนที่ 571),
การประมวลผลรูปภาพด้วย Pillow (572), และ view ที่รับไฟล์ผ่าน chunked/S3 upload (574-575)
— จะกลายเป็น**ตัวอย่างจริงที่ยอดเยี่ยม**สำหรับฝึกเขียนเทสใน Phase 7 เพราะมี logic ที่
ซับซ้อนพอจะทดสอบได้อย่างมีความหมาย (ต่างจากโค้ดง่าย ๆ ที่ไม่มีอะไรให้พลาด) เตรียมตัวให้
พร้อม เพราะ Phase 7 คือจุดที่แยกนักพัฒนา "มือสมัครเล่นที่เขียนโค้ดได้" ออกจาก
**"วิศวกรซอฟต์แวร์มืออาชีพที่มั่นใจในโค้ดของตัวเอง"** อย่างแท้จริง
