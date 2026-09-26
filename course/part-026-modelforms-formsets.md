# Part 026: ModelForms และ Formsets

> **ขั้นตอนที่ 251-260 ของหลักสูตร** | Phase 3: Views, Templates, Forms และ CBV
>
> เป้าหมายของ Part นี้: นำทุก pattern ที่คุณเขียนด้วยมือใน Part 025 (การ copy ค่า
> จาก `cleaned_data` ไปสร้าง object เอง, การ pre-fill ค่าจาก instance เดิมเอง)
> มาย่นระยะให้สั้นลงมากด้วย **`ModelForm`** — ฟอร์มที่สร้าง Form field ให้อัตโนมัติ
> จาก Model field พร้อม method `.save()` ที่บันทึกหรืออัปเดต object ให้ในบรรทัด
> เดียว คุณจะเข้าใจ `class Meta`, ความสัมพันธ์ระหว่าง Model validator กับ Form
> validation, การปรับฟอร์มแบบ dynamic ตาม user ที่ login ผ่านการ override
> `__init__`, และขยายไปสู่ **Formset** — กลไกจัดการฟอร์มชนิดเดียวกันหลายชุดพร้อมกัน
> ทั้ง `formset_factory` (ฟอร์มเปล่าหลายชุด), `modelformset_factory` (แก้ไขหลาย
> object พร้อมกัน) และ `inlineformset_factory` (จัดการความสัมพันธ์ parent-child
> เช่น `Post` ที่มีหลายรูปภาพ `PostImage`) เมื่อจบ Part นี้ คุณจะแปลง `CommentForm`
> จาก Part 025 เป็น `ModelForm` ตัวจริง และสร้างฟอร์มเพิ่ม `Post` พร้อมรูปภาพ
> หลายรูปในหน้าเดียวได้อย่างมืออาชีพ

---

## สารบัญของ Part นี้

- ขั้นตอนที่ 251: `ModelForm` เบื้องต้น — `class Meta`, `fields`/`exclude` และ Widget ที่ Django สร้างให้อัตโนมัติจาก Model Field
- ขั้นตอนที่ 252: `ModelForm.save()` และรูปแบบ `commit=False`
- ขั้นตอนที่ 253: Override `__init__` ของ ModelForm เพื่อปรับ Widget/Queryset แบบ Dynamic
- ขั้นตอนที่ 254: Model Field Validator กับ ModelForm Validation และ `clean_<field>()`
- ขั้นตอนที่ 255: Formset เบื้องต้นด้วย `formset_factory`
- ขั้นตอนที่ 256: `modelformset_factory` — แก้ไขหลาย Object พร้อมกันในหน้าเดียว
- ขั้นตอนที่ 257: `inlineformset_factory` — จัดการ Parent-Child Form
- ขั้นตอนที่ 258: Formset Management Form เจาะลึก — `TOTAL_FORMS`, `INITIAL_FORMS`, `extra`, `can_delete`, `max_num`, `min_num`
- ขั้นตอนที่ 259: Validation ข้าม Formset ทั้งชุดด้วย `formset.clean()`
- ขั้นตอนที่ 260: สรุปและแบบฝึกหัด — แปลง `CommentForm` เป็น `ModelForm` และสร้าง Inline Formset สำหรับ `PostImage`

---

## ขั้นตอนที่ 251: `ModelForm` เบื้องต้น — `class Meta`, `fields`/`exclude` และ Widget อัตโนมัติ

### 251.1 ทบทวนปัญหาที่ `ModelForm` มาแก้

ใน Part 025 ขั้นตอนที่ 250 คุณเขียน `CommentForm` แบบ plain `forms.Form` แล้วต้อง
ทำสองอย่างด้วยมือทุกครั้ง:

1. **ประกาศ field ซ้ำ** — `author = forms.CharField(max_length=100)` ทั้งที่
   `Comment.author` ใน `models.py` ก็เป็น `CharField(max_length=100)` อยู่แล้ว
   ถ้าวันหนึ่งเปลี่ยน `max_length` ใน Model ต้องไม่ลืมมาแก้ใน Form ด้วย —
   ขัดกับหลักการ **DRY** ที่เรียนไปตั้งแต่ Part 001
2. **ประกอบ object เอง** — `Comment.objects.create(post=post, author=form.cleaned_data["author"], text=form.cleaned_data["text"])`
   ต้อง copy ค่าทีละ field ด้วยมือ ยิ่ง Model มี field เยอะ โค้ดส่วนนี้ก็ยิ่งยาว
   และเสี่ยงพิมพ์ชื่อ field ผิดโดยไม่มี error เตือนจนกว่าจะรันจริง

**`ModelForm`** แก้ปัญหาทั้งสองข้อพร้อมกัน: มันอ่านโครงสร้างจาก Model โดยตรง
เพื่อสร้าง Form field ให้อัตโนมัติ (ข้อ 1) และมี method `.save()` ที่ประกอบ object
พร้อมบันทึกลงฐานข้อมูลให้ในบรรทัดเดียว (ข้อ 2) — มันคือ **สะพานเชื่อม** ระหว่าง
Model กับ Form ที่ Django สร้างไว้ให้ เพราะทั้งสองระบบนี้มักต้องทำงานคู่กันบ่อย
มากในทางปฏิบัติ (ฟอร์มส่วนใหญ่ในเว็บแอปพลิเคชันจริงมีไว้เพื่อสร้าง/แก้ไข object
ในฐานข้อมูลนั่นเอง)

### 251.2 Model ที่จะใช้ตลอด Part นี้

Part นี้ขยาย Model ของแอป `blog` จาก Part 025 ให้สมบูรณ์ขึ้น เพิ่ม `Category`,
`Tag`, `PostImage` และ field ใหม่ใน `Post` เพื่อให้มีตัวอย่างครบทุกชนิด field
ที่ `ModelForm` ต้องรับมือ (`ForeignKey`, `ManyToManyField`, `ImageField`,
`BooleanField` ฯลฯ):

```python
# blog/models.py
from django.conf import settings
from django.core.exceptions import ValidationError
from django.core.validators import MinLengthValidator
from django.db import models


def validate_no_banned_words(value):
    """Custom model validator: ห้ามมีคำต้องห้ามปนอยู่ในเนื้อหา (ใช้ต่อในขั้นตอนที่ 254)"""
    banned_words = ["สแปม", "โฆษณา", "คลิกที่นี่"]
    lowered = value.lower()
    for word in banned_words:
        if word in lowered:
            raise ValidationError(f'ห้ามมีคำว่า "{word}" ปรากฏอยู่ในเนื้อหา')


class Category(models.Model):
    name = models.CharField(max_length=100)
    slug = models.SlugField(max_length=120, unique=True)
    # created_by = None หมายถึงหมวดหมู่กลางที่ทุกคนใช้ร่วมกันได้
    # created_by = user ใดยูสเซอร์หนึ่ง หมายถึงหมวดหมู่ส่วนตัวที่คนอื่นมองไม่เห็น
    created_by = models.ForeignKey(
        settings.AUTH_USER_MODEL,
        on_delete=models.CASCADE,
        null=True,
        blank=True,
        related_name="categories",
    )

    class Meta:
        verbose_name_plural = "categories"

    def __str__(self):
        return self.name


class Tag(models.Model):
    name = models.CharField(max_length=50, unique=True)

    def __str__(self):
        return self.name


class Post(models.Model):
    title = models.CharField(
        max_length=200,
        validators=[MinLengthValidator(5, message="หัวข้อต้องมีความยาวอย่างน้อย 5 ตัวอักษร")],
    )
    slug = models.SlugField(max_length=220, unique=True)
    content = models.TextField(validators=[validate_no_banned_words])
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)
    is_published = models.BooleanField(default=False)
    category = models.ForeignKey(
        Category, on_delete=models.SET_NULL, null=True, blank=True, related_name="posts",
    )
    tags = models.ManyToManyField(Tag, blank=True, related_name="posts")
    author = models.ForeignKey(
        settings.AUTH_USER_MODEL, on_delete=models.CASCADE, related_name="posts",
    )

    def __str__(self):
        return self.title


class Comment(models.Model):
    post = models.ForeignKey(Post, on_delete=models.CASCADE, related_name="comments")
    author = models.CharField(max_length=100)
    text = models.TextField()
    created_at = models.DateTimeField(auto_now_add=True)

    class Meta:
        ordering = ["-created_at"]

    def __str__(self):
        return f"ความคิดเห็นโดย {self.author} บน {self.post.title}"


class PostImage(models.Model):
    post = models.ForeignKey(Post, on_delete=models.CASCADE, related_name="images")
    image = models.ImageField(upload_to="post_images/%Y/%m/")
    caption = models.CharField(max_length=200, blank=True)
    is_primary = models.BooleanField(default=False, help_text="รูปหลักที่แสดงเป็นภาพปก")
    order = models.PositiveIntegerField(default=0)

    class Meta:
        ordering = ["order"]

    def __str__(self):
        return f"รูปของ {self.post.title} (#{self.order})"
```

หลังเพิ่ม field ใหม่และ Model ใหม่แล้ว อย่าลืมรัน migration ตามธรรมเนียมที่เรียน
มาตั้งแต่ Phase 2:

```bash
python manage.py makemigrations blog
python manage.py migrate
```

### 251.3 สร้าง `ModelForm` แรก: แปลง `CommentForm` เบื้องต้น

เทียบกับ `forms.Form` ที่สืบทอดจาก `forms.Form`, `ModelForm` สืบทอดจาก
**`forms.ModelForm`** และไม่ต้องประกาศ field เองเลยแม้แต่ตัวเดียว — บอกแค่ว่า
"อ้างอิง Model ไหน" และ "เอา field ไหนบ้าง" ผ่าน nested class ที่ชื่อ `Meta`
เสมอ (ชื่อนี้ตายตัว Django มองหา class ชื่อ `Meta` เท่านั้น):

```python
# blog/forms.py
from django import forms
from .models import Comment


class CommentForm(forms.ModelForm):
    class Meta:
        model = Comment
        fields = ["author", "text"]
```

ทดสอบใน shell ให้เห็นว่า Django สร้าง field ให้ครบโดยที่เราไม่ได้เขียนเอง:

```python
>>> from blog.forms import CommentForm
>>> form = CommentForm()
>>> print(form.as_p())
<p><label for="id_author">Author:</label>
<input type="text" name="author" maxlength="100" required id="id_author"></p>
<p><label for="id_text">Text:</label>
<textarea name="text" cols="40" rows="10" required id="id_text"></textarea></p>
```

สังเกตว่า `author` ได้ `maxlength="100"` มาโดยอัตโนมัติ — ค่านี้ Django **อ่านมาจาก
`Comment.author = models.CharField(max_length=100)` โดยตรง** ไม่ต้องพิมพ์ซ้ำเอง
เลย นี่คือหัวใจสำคัญที่สุดของ `ModelForm`: **Model เป็นแหล่งความจริงเดียว
(Single Source of Truth)** ตามหลัก DRY

### 251.4 `class Meta` คืออะไรกันแน่

`class Meta` ที่นี่เป็นแนวคิดเดียวกับ `class Meta` ที่คุณเคยเขียนใน Model
(เช่น `ordering`, `verbose_name_plural` ใน `Comment.Meta` และ `Category.Meta`
ด้านบน) — เป็นเพียง **class ย่อยสำหรับใส่ metadata (ข้อมูลเกี่ยวกับ class หลัก)**
โดยไม่ปนกับ logic ปกติของ class นั้น Django ใช้ pattern นี้ในหลายที่เพื่อแยก
"การตั้งค่า" ออกจาก "พฤติกรรม" ให้อ่านง่าย

Attribute ที่ `ModelForm.Meta` **ต้องมีเสมอ 2 ตัว**:

| Attribute | ความหมาย | บังคับไหม |
|---|---|---|
| `model` | Model class ที่ฟอร์มนี้อ้างอิง | **บังคับ** |
| `fields` หรือ `exclude` | ระบุว่าจะเอา field ไหนจาก Model มาสร้างเป็น Form field | **บังคับต้องมีอย่างใดอย่างหนึ่ง** |

ถ้าลืมทั้ง `fields` และ `exclude` ทั้งคู่ Django จะ raise `ImproperlyConfigured`
ทันทีตอน import พร้อมข้อความชัดเจนว่าต้องระบุอย่างใดอย่างหนึ่งเสมอ (นี่เป็น
การป้องกันไว้ก่อน เพราะเวอร์ชันเก่าของ Django เคยอนุญาตให้ไม่ระบุแล้ว fallback
เป็น "เอาทุก field" ซึ่งเป็นความเสี่ยงด้านความปลอดภัยที่จะอธิบายต่อไป)

### 251.5 `fields`, `exclude`, และทำไมห้ามใช้ `"__all__"` พร่ำเพรื่อ

Django มี 3 วิธีระบุว่าเอา field ไหนบ้าง:

| วิธี | ตัวอย่าง | พฤติกรรม |
|---|---|---|
| `fields = [...]` | `fields = ["author", "text"]` | เอาเฉพาะ field ที่ระบุ (แนะนำที่สุด) |
| `exclude = [...]` | `exclude = ["post", "created_at"]` | เอาทุก field **ยกเว้น** ที่ระบุ |
| `fields = "__all__"` | `fields = "__all__"` | เอาทุก field ของ Model มาทั้งหมด |

```python
class CommentForm(forms.ModelForm):
    class Meta:
        model = Comment
        exclude = ["post", "created_at"]   # เอาทุก field ยกเว้น post กับ created_at
```

ผลลัพธ์ของ `exclude` ด้านบนเหมือนกับ `fields = ["author", "text"]` ทุกประการ
ในกรณีนี้ (เพราะ `Comment` มีแค่ 4 field) แต่ **`exclude` อันตรายกว่าเมื่อ Model
มีการเพิ่ม field ใหม่ในอนาคต**: สมมติทีมเพิ่ม field `Comment.is_approved =
models.BooleanField(default=False)` เข้ามาทีหลัง ถ้าฟอร์มใช้ `exclude` field
ใหม่นี้จะ **โผล่เข้ามาในฟอร์มโดยอัตโนมัติทันที** โดยที่ไม่มีใครตั้งใจ — ถ้าเป็น
field ที่ผู้ใช้ทั่วไปไม่ควรแก้ไขได้เอง (เช่น สถานะอนุมัติ) นี่คือช่องโหว่ด้าน
ความปลอดภัยที่เรียกว่า **Mass Assignment Vulnerability**

> **กฎเหล็กของหลักสูตรนี้: ห้ามใช้ `fields = "__all__"` กับฟอร์มที่รับข้อมูล
> จากผู้ใช้ทั่วไปเด็ดขาด** โดยเฉพาะ Model ที่มี field อ่อนไหวอย่าง
> `is_published`, `is_staff`, `is_approved`, `price`, `author` เพราะถ้ามีคน
> แอบเติม input ที่ชื่อตรงกับ field เหล่านั้นเข้าไปใน HTML form (ผ่าน
> DevTools หรือส่ง POST request ตรง ๆ ด้วยเครื่องมืออย่าง `curl`/Postman)
> ฟอร์มจะ **รับค่านั้นมา validate และบันทึกให้ทันที** เพราะมันเป็น field ที่
> ฟอร์ม "รู้จัก" ทั้งที่ผู้พัฒนาไม่เคยตั้งใจให้แก้ไขได้จากภายนอกเลย

**คำแนะนำของหลักสูตรนี้**: ใช้ `fields = [...]` แบบระบุชัดเจนเสมอ (explicit
list) เพราะเมื่อ Model เปลี่ยนแปลงในอนาคต ฟอร์มจะ **ไม่เปลี่ยนพฤติกรรมโดย
อัตโนมัติ** ต้องมีคนมาแก้ `fields` เองอย่างตั้งใจเท่านั้น — นี่คือหลักการ
**"Explicit is better than implicit"** ที่ปรากฏใน Zen of Python ซึ่ง Django
สืบทอดปรัชญานี้มาเต็มตัว

### 251.6 Widget ที่ Django สร้างให้อัตโนมัติจาก Model Field Type

นี่คือตารางที่สำคัญที่สุดของขั้นตอนนี้ — Django มีกฎการแปลง (mapping) ที่ตายตัว
จาก Model field type ไปเป็น Form field + Widget:

| Model Field | Form Field ที่ได้ | Widget เริ่มต้น |
|---|---|---|
| `CharField` | `forms.CharField` | `TextInput` |
| `CharField(choices=...)` | `forms.TypedChoiceField` | `Select` |
| `TextField` | `forms.CharField` | `Textarea` |
| `SlugField` | `forms.SlugField` | `TextInput` |
| `EmailField` | `forms.EmailField` | `EmailInput` |
| `URLField` | `forms.URLField` | `URLInput` |
| `IntegerField` | `forms.IntegerField` | `NumberInput` |
| `PositiveIntegerField` | `forms.IntegerField` (มี `min_value=0`) | `NumberInput` |
| `DecimalField` | `forms.DecimalField` | `NumberInput` |
| `BooleanField` | `forms.BooleanField` | `CheckboxInput` |
| `BooleanField(null=True)` | `forms.NullBooleanField` | `Select` (Unknown/Yes/No) |
| `DateField` | `forms.DateField` | `DateInput` |
| `DateTimeField` | `forms.DateTimeField` | `DateTimeInput` |
| `ImageField` | `forms.ImageField` | `ClearableFileInput` |
| `FileField` | `forms.FileField` | `ClearableFileInput` |
| `ForeignKey` | `forms.ModelChoiceField` | `Select` (ตัวเลือก = ทุก row ในตาราง) |
| `ManyToManyField` | `forms.ModelMultipleChoiceField` | `SelectMultiple` |
| `auto_now_add=True` / `auto_now=True` | **ไม่สร้าง Form field ให้เลย** | — |

ทดสอบตารางนี้กับ Model `Post` ทั้งก้อนใน shell:

```python
>>> from blog.forms import PostForm   # สมมติสร้างไว้แล้วตามขั้นตอนที่ 251.7
>>> form = PostForm()
>>> for name, field in form.fields.items():
...     print(name, type(field).__name__, type(field.widget).__name__)
title CharField TextInput
slug SlugField TextInput
content CharField Textarea
category ModelChoiceField Select
tags ModelMultipleChoiceField SelectMultiple
is_published BooleanField CheckboxInput
```

สังเกตว่า `created_at` และ `updated_at` (ที่มี `auto_now_add=True`/`auto_now=True`)
**ไม่ปรากฏใน `form.fields` เลย แม้จะใส่ชื่อไว้ใน `fields` ของ `Meta`** — Django
รู้ว่า field ประเภทนี้ระบบจัดการให้อัตโนมัติ ผู้ใช้ไม่ควรมีสิทธิ์กรอกเองอยู่แล้ว
จึงข้ามให้โดยไม่ error (แต่ถ้าใส่ชื่อ field ที่ไม่มีอยู่จริงใน Model เข้าไปใน
`fields` เช่นพิมพ์ผิด Django จะ raise `FieldError` ทันทีตอน import — ต่างจากกรณี
`auto_now`/`auto_now_add` ที่ถูกละเว้นอย่างเงียบ ๆ โดยตั้งใจ)

### 251.7 Override Widget/Label/Help Text/Error Message ใน `Meta`

Widget ที่ได้อัตโนมัติอาจไม่ตรงกับที่ต้องการเสมอไป (เช่น อยากได้
`CheckboxSelectMultiple` แทน `SelectMultiple` สำหรับ `tags`) `Meta` มี attribute
เสริมให้ override ได้แบบเฉพาะเจาะจงทีละ field โดยไม่ต้องเขียน field นั้นใหม่
ทั้งตัว:

```python
# blog/forms.py
from django import forms
from .models import Post


class PostForm(forms.ModelForm):
    class Meta:
        model = Post
        fields = ["title", "slug", "content", "category", "tags", "is_published"]
        widgets = {
            "content": forms.Textarea(attrs={"rows": 10, "class": "form-control"}),
            "tags": forms.CheckboxSelectMultiple,
            "title": forms.TextInput(attrs={"class": "form-control", "placeholder": "หัวข้อบทความ"}),
        }
        labels = {
            "is_published": "เผยแพร่ทันทีหลังบันทึก",
        }
        help_texts = {
            "slug": "ใช้เป็นส่วนหนึ่งของ URL เช่น my-first-post (ตัวพิมพ์เล็ก คั่นด้วยขีดกลาง)",
        }
        error_messages = {
            "title": {
                "required": "กรุณากรอกหัวข้อบทความ",
                "max_length": "หัวข้อยาวเกินไป (ไม่เกิน 200 ตัวอักษร)",
            },
        }
```

| Attribute ของ `Meta` | ปรับอะไร |
|---|---|
| `widgets` | เปลี่ยน widget ของ field ที่ระบุ (dict: `{field_name: widget}`) |
| `labels` | เปลี่ยนข้อความ label ที่แสดงหน้าฟอร์ม |
| `help_texts` | เปลี่ยน/เพิ่มข้อความอธิบายใต้ field |
| `error_messages` | กำหนดข้อความ error เฉพาะของแต่ละ field แยกตาม error code (`required`, `max_length` ฯลฯ) |
| `field_classes` | เปลี่ยน Form field class ทั้งตัว (ใช้น้อยมาก เจาะลึกใน Part 027) |

**หลักการเลือก**: ถ้าปรับแค่หน้าตา/ข้อความ ให้ใช้ attribute ใน `Meta` เหล่านี้
ก่อนเสมอ เพราะสั้นและยังคง auto-generate field type ให้อยู่ ค่อยไปเขียน field
ประกาศเองตรง ๆ ในคลาส (เหมือน `forms.Form`) เฉพาะตอนที่ต้องการ validation
เพิ่มเติมที่ `Meta` ทำไม่ได้ (จะเห็นตัวอย่างใน 254.4)

### 251.8 ตารางเปรียบเทียบ `forms.Form` vs `ModelForm`

| ประเด็น | `forms.Form` (Part 025) | `ModelForm` (Part นี้) |
|---|---|---|
| ต้องประกาศ field เองไหม | ต้อง ทุกตัว | ไม่ต้อง (สร้างจาก Model อัตโนมัติ) |
| ผูกกับ Model ไหม | ไม่ผูก (เป็นอิสระ) | ผูกกับ Model ที่ระบุใน `Meta.model` เสมอ |
| มี method `.save()` ไหม | ไม่มี | มี — สร้าง/อัปเดต object ให้อัตโนมัติ |
| เหมาะกับ | ฟอร์มค้นหา, login, contact ที่ไม่ตรงกับ Model ใด ๆ | ฟอร์มสร้าง/แก้ไข object ในฐานข้อมูล |
| `class Meta` | ไม่มี | มี (`model`, `fields`/`exclude` บังคับ) |
| `clean_<field>()` / `clean()` | ใช้ได้ | ใช้ได้เหมือนกันทุกประการ (ขั้นตอนที่ 254) |
| Validator ระดับ Model ทำงานด้วยไหม | ไม่เกี่ยว (ไม่มี instance) | ทำงานอัตโนมัติผ่าน `full_clean()` (ขั้นตอนที่ 254) |

---

## ขั้นตอนที่ 252: `ModelForm.save()` และรูปแบบ `commit=False`

### 252.1 `save()` เริ่มต้น: บันทึกลง DB ทันที

ค่าเริ่มต้นของ `.save()` คือ `commit=True` — เมื่อเรียกแล้ว Django จะสร้าง
instance ของ Model จาก `cleaned_data` แล้วยิง `INSERT`/`UPDATE` ลงฐานข้อมูลทันที
พร้อมคืน instance ที่บันทึกเสร็จแล้วกลับมา:

```python
>>> from blog.forms import CommentForm
>>> form = CommentForm(data={"author": "สมชาย", "text": "เนื้อหาที่ยาวพอสมควรครับ"})
>>> form.is_valid()
True
>>> comment = form.save()   # ยิง INSERT ลง DB ทันที ณ บรรทัดนี้
>>> comment.pk
1
>>> comment.post
```

ปัญหาคือบรรทัดสุดท้าย — `comment.post` จะ error เพราะ `Comment.post` เป็น
`ForeignKey` ที่ `null=False` (บังคับต้องมีค่า) แต่เราไม่ได้ใส่ `post` ไว้ใน
`Meta.fields` ของ `CommentForm` เลย (ตั้งใจ เพราะผู้ใช้ไม่ควรเลือก post เองจาก
หน้าเว็บ — post ต้องมาจาก URL ที่กำลังดูอยู่) เมื่อ Django พยายาม `INSERT` แถวที่
ไม่มีค่า `post_id` มันจะ raise `IntegrityError` (`NOT NULL constraint failed`)
ทันที นี่คือปัญหาคลาสสิกที่ `commit=False` มาแก้

### 252.2 `commit=False`: ขอ Instance มาก่อน ยังไม่ยิง SQL จริง

```python
form = CommentForm(data=request.POST)
if form.is_valid():
    comment = form.save(commit=False)   # สร้าง instance ในหน่วยความจำ ยังไม่ INSERT
    comment.post = post                 # เติม field ที่ฟอร์มไม่รู้จักเอง
    comment.save()                      # ค่อยยิง INSERT จริงตรงนี้
```

`form.save(commit=False)` ทำสิ่งเดียวกับ `form.save()` ทุกอย่าง **ยกเว้น** ขั้นตอน
สุดท้ายที่ยิง SQL ลงฐานข้อมูล — มันสร้าง Python object ของ Model ขึ้นมาในหน่วย
ความจำ (พร้อมค่าที่ผ่าน validation แล้วทุก field ที่ฟอร์มรู้จัก) แล้วคืนกลับมา
ให้เราแก้ไข/เติมค่าเพิ่มก่อนได้ตามใจ จากนั้นเรียก `.save()` ของ Model ปกติเอง
อีกทีเมื่อพร้อม

นี่คือ **View `post_detail_view` เวอร์ชัน `ModelForm`** เทียบกับเวอร์ชันมือ
(manual) จาก Part 025 ขั้นตอนที่ 250.3:

```python
# blog/views.py
from django.shortcuts import render, redirect, get_object_or_404
from .models import Post
from .forms import CommentForm


def post_detail_view(request, slug):
    post = get_object_or_404(Post, slug=slug)
    comments = post.comments.all()

    if request.method == "POST":
        form = CommentForm(request.POST)
        if form.is_valid():
            comment = form.save(commit=False)
            comment.post = post
            comment.save()
            return redirect("blog:detail", slug=post.slug)
    else:
        form = CommentForm()

    return render(request, "blog/post_detail.html", {
        "post": post,
        "comments": comments,
        "form": form,
    })
```

เทียบเฉพาะส่วนสร้าง `Comment`:

| | Part 025 (manual) | Part 026 (`ModelForm`) |
|---|---|---|
| โค้ด | `Comment.objects.create(post=post, author=form.cleaned_data["author"], text=form.cleaned_data["text"])` | `comment = form.save(commit=False); comment.post = post; comment.save()` |
| ต้อง copy field ทีละตัวไหม | ต้อง (เสี่ยงพิมพ์ชื่อผิด) | ไม่ต้อง (Django จัดการให้ทุก field ที่ฟอร์มรู้จัก) |
| ถ้า `Comment` เพิ่ม field ใหม่ (เช่น `email`) | ต้องแก้ทั้ง Form และ View | แก้แค่ `Meta.fields` ใน Form พอ |

### 252.3 `commit=False` กับ `ManyToManyField`: ทำไมต้องเรียก `save_m2m()`

ความสัมพันธ์แบบ `ManyToManyField` ถูกเก็บใน **ตารางกลาง (through table)**
แยกต่างหากจากตารางหลัก การจะเขียนแถวลงตารางกลางได้ ต้องมี **primary key ของ
ทั้งสองฝั่ง** อยู่แล้วเท่านั้น (เช่น จะบอกว่า "Post #5 มี Tag #2 และ #3" ได้
ก็ต่อเมื่อ Post #5 มีอยู่จริงในตารางก่อน) ดังนั้นเมื่อเรียก
`form.save(commit=False)` กับฟอร์มที่มี M2M field Django **จะยังไม่บันทึกข้อมูล
M2M ให้เลย** (เพราะ instance ยังไม่มี pk) แต่จะเก็บข้อมูล M2M ที่รอบันทึกไว้ใน
attribute พิเศษของฟอร์ม แล้วรอให้เราเรียก **`form.save_m2m()`** เองหลังจาก
instance ถูกบันทึกแล้ว:

```python
# blog/views.py
def post_create_view(request):
    if request.method == "POST":
        form = PostForm(request.POST)
        if form.is_valid():
            post = form.save(commit=False)   # ยังไม่บันทึก tags (M2M) ตอนนี้
            post.author = request.user        # เติม field ที่ไม่ได้อยู่ใน Meta.fields
            post.save()                       # ตอนนี้ post มี pk แล้ว
            form.save_m2m()                   # ค่อยบันทึกความสัมพันธ์ tags ได้
            return redirect("blog:detail", slug=post.slug)
    else:
        form = PostForm()
    return render(request, "blog/post_form.html", {"form": form})
```

> **กฎเหล็ก**: ทุกครั้งที่ `ModelForm` มี field ประเภท `ManyToManyField` อยู่ใน
> `Meta.fields` และคุณใช้ `commit=False` **ต้องเรียก `form.save_m2m()` เอง
> เสมอหลังจาก instance ถูก `.save()` แล้ว** ไม่เช่นนั้นข้อมูล M2M ที่ผู้ใช้
> เลือกไว้ (เช่น tags ที่ติ๊กเลือก) **จะหายไปเงียบ ๆ โดยไม่มี error ใด ๆ
> เตือน** — เป็นบั๊กที่พบบ่อยมากและ debug ยาก เพราะฟอร์มดู `is_valid()` แล้ว
> ทุกอย่างก็ดูปกติดี

ถ้าใช้ `form.save()` แบบธรรมดา (`commit=True` ค่าเริ่มต้น) Django จัดการเรื่องนี้
ให้อัตโนมัติอยู่แล้ว — เบื้องหลังคือ `commit=True` เทียบเท่ากับการเรียก
`instance.save()` ตามด้วย `self.save_m2m()` ให้เองในลำดับที่ถูกต้อง เราต้องมา
เรียกเองก็ต่อเมื่อ **ต้องแทรกโค้ดคั่นกลาง** ระหว่างสองขั้นตอนนี้เท่านั้น (เช่น
ต้องเซ็ต `post.author` ก่อน ซึ่งเป็น field ที่ฟอร์มไม่รู้จักเลย)

### 252.4 ตารางสรุป: เมื่อไรใช้ `save()` แบบไหน

| สถานการณ์ | วิธีที่ถูกต้อง |
|---|---|
| ทุก field ที่ต้องบันทึกอยู่ใน `Meta.fields` ครบแล้ว ไม่มี field ภายนอกต้องเติม | `form.save()` เฉย ๆ พอ |
| มี field ที่ต้องเติมเอง (เช่น `post`, `author` ที่มาจาก request/URL ไม่ใช่จากฟอร์ม) และ**ไม่มี** M2M field | `instance = form.save(commit=False)` → เติมค่า → `instance.save()` |
| มี field ที่ต้องเติมเอง **และมี** M2M field ด้วย | `instance = form.save(commit=False)` → เติมค่า → `instance.save()` → `form.save_m2m()` |
| แก้ไข object เดิม (ส่ง `instance=` ตอนสร้างฟอร์ม) | เหมือนสร้างใหม่ทุกประการ เพียงแต่ `form.save()`/`form.save(commit=False)` จะ `UPDATE` แทน `INSERT` |

การส่ง `instance=` ตอนสร้างฟอร์มคือวิธีบอก Django ว่า "นี่คือการแก้ไข ไม่ใช่
สร้างใหม่" — ทบทวนจาก Part 025 ที่ใช้ `initial=` กับ plain Form pre-fill ค่า
เก่า `ModelForm` ทำสิ่งเดียวกันได้ง่ายกว่ามาก:

```python
def post_edit_view(request, slug):
    post = get_object_or_404(Post, slug=slug)

    if request.method == "POST":
        form = PostForm(request.POST, instance=post)   # bound + ผูกกับ object เดิม
        if form.is_valid():
            form.save()   # UPDATE แถวเดิม ไม่ใช่สร้างแถวใหม่
            return redirect("blog:detail", slug=post.slug)
    else:
        form = PostForm(instance=post)   # unbound แต่ pre-fill ด้วยค่าปัจจุบันของ post

    return render(request, "blog/post_form.html", {"form": form})
```

สังเกตว่า `PostForm(instance=post)` ตอน GET **แสดงค่าปัจจุบันในฐานข้อมูลให้ทันที
โดยไม่ต้องเขียน `initial={...}` เองทีละ field เหมือนที่ทำใน Part 025 ขั้นตอนที่
247.4** — นี่คืออีกจุดที่ `ModelForm` ย่นระยะขั้นตอนให้สั้นลงอย่างชัดเจน

---

## ขั้นตอนที่ 253: Override `__init__` เพื่อปรับ Widget/Queryset แบบ Dynamic

### 253.1 ทำไมต้อง Override `__init__`

`class Meta` ปรับแต่งได้แค่สิ่งที่ **รู้ล่วงหน้าตอนเขียนโค้ด** (static) เช่น
"field `tags` ให้ใช้ `CheckboxSelectMultiple` เสมอ" แต่บางครั้งการปรับแต่งต้อง
ขึ้นอยู่กับ **ข้อมูล ณ ตอน runtime** ที่รู้ได้ก็ต่อเมื่อมี request เข้ามาแล้ว
เท่านั้น เช่น:

- จำกัด queryset ของ field `category` ให้เห็นเฉพาะหมวดหมู่ของ user ที่ login
  อยู่ บวกกับหมวดหมู่กลางที่ทุกคนใช้ร่วมกันได้
- ปิดการแก้ไข field บางตัว (`disabled`) ถ้า user ไม่มีสิทธิ์เพียงพอ
- เติม CSS class ให้ทุก field อัตโนมัติโดยไม่ต้องเขียนซ้ำใน `Meta.widgets`
  ทีละตัว

สิ่งเหล่านี้ทำไม่ได้ใน `class Meta` เพราะ `Meta` ถูกกำหนดตอน import module
(ครั้งเดียวตอนเริ่มโปรแกรม) แต่ "user ที่ login อยู่" รู้ได้เฉพาะตอนมี request
เข้ามาแล้วเท่านั้น จุดที่เหมาะสมที่สุดในการแทรก logic แบบนี้คือ **method
`__init__` ของฟอร์ม** ซึ่งถูกเรียกใหม่ทุกครั้งที่ view สร้างฟอร์มขึ้นมา

### 253.2 รูปแบบมาตรฐานของการ Override `__init__`

```python
# blog/forms.py
from django import forms
from django.db.models import Q
from .models import Post, Category


class PostForm(forms.ModelForm):
    class Meta:
        model = Post
        fields = ["title", "slug", "content", "category", "tags", "is_published"]

    def __init__(self, *args, **kwargs):
        self.user = kwargs.pop("user", None)   # (1) ดึง kwarg พิเศษออกก่อนเสมอ
        super().__init__(*args, **kwargs)        # (2) ส่งที่เหลือให้ parent ตามปกติ

        if self.user is not None:
            # (3) จำกัด queryset ของ category ให้เห็นแค่ของตัวเองกับหมวดกลาง
            self.fields["category"].queryset = Category.objects.filter(
                Q(created_by=self.user) | Q(created_by__isnull=True)
            )
```

> **กฎเหล็ก**: ต้อง **`.pop()` kwarg ที่เพิ่มขึ้นเองออกจาก `kwargs` ก่อนเรียก
> `super().__init__(*args, **kwargs)` เสมอ** เพราะ `forms.ModelForm.__init__()`
> ดั้งเดิม **ไม่รู้จัก** kwarg แปลกอย่าง `user` เลย ถ้าส่งต่อเข้าไปตรง ๆ
> โดยไม่ pop ออกก่อน จะได้ error `TypeError: __init__() got an unexpected
> keyword argument 'user'` ทันที — เป็น error ที่พบบ่อยที่สุดของมือใหม่ที่
> เริ่มเขียน custom `__init__`

### 253.3 View ที่ส่ง `user=` เข้าฟอร์ม

```python
# blog/views.py
from django.contrib.auth.decorators import login_required
from django.shortcuts import render, redirect
from .forms import PostForm


@login_required
def post_create_view(request):
    if request.method == "POST":
        form = PostForm(request.POST, user=request.user)   # ส่ง user= เพิ่มเข้าไป
        if form.is_valid():
            post = form.save(commit=False)
            post.author = request.user
            post.save()
            form.save_m2m()
            return redirect("blog:detail", slug=post.slug)
    else:
        form = PostForm(user=request.user)   # ต้องส่ง user= ทั้งตอน GET และ POST

    return render(request, "blog/post_form.html", {"form": form})
```

สังเกตว่า `user=request.user` ถูกส่งเข้าไปเป็น **keyword argument ตัวสุดท้าย
ทุกครั้งที่สร้างฟอร์ม** (ทั้ง GET และ POST) เพราะฟอร์มต้องรู้จัก user เสมอไม่ว่า
จะแสดงฟอร์มเปล่าหรือกำลัง validate ข้อมูลที่ส่งมา — ถ้าลืมส่งตอนใดตอนหนึ่ง
`self.user` จะเป็น `None` และเงื่อนไขจำกัด queryset ใน `__init__` จะไม่ทำงาน

### 253.4 ปรับ Widget ทุก Field พร้อมกันด้วย Loop

อีกประโยชน์ที่พบบ่อยของการ override `__init__` คือการวนลูป `self.fields`
เพื่อปรับ attribute ให้ทุก field พร้อมกัน โดยไม่ต้องเขียนซ้ำใน `Meta.widgets`
ทีละตัว (มีประโยชน์มากเมื่อ Form มี field จำนวนมาก หรือเมื่อสร้าง Base Form
Class ให้ Form อื่นสืบทอดต่อ):

```python
class PostForm(forms.ModelForm):
    class Meta:
        model = Post
        fields = ["title", "slug", "content", "category", "tags", "is_published"]

    def __init__(self, *args, **kwargs):
        self.user = kwargs.pop("user", None)
        super().__init__(*args, **kwargs)

        # เติม CSS class ให้ทุก field อัตโนมัติ (ยกเว้น checkbox ที่ควรใช้ class อื่น)
        for field_name, field in self.fields.items():
            if isinstance(field.widget, (forms.CheckboxInput, forms.CheckboxSelectMultiple)):
                field.widget.attrs.setdefault("class", "form-check-input")
            else:
                field.widget.attrs.setdefault("class", "form-control")
```

`.setdefault()` (ไม่ใช่การเขียนทับตรง ๆ ด้วย `=`) สำคัญตรงที่ **ไม่เขียนทับ
`class` ที่ตั้งค่าไว้แล้วใน `Meta.widgets` ของ field นั้น ๆ** ถ้า field ไหน
ระบุ `class` เฉพาะไว้แล้ว ลูปนี้จะข้ามไปเฉย ๆ ถ้า field ไหนยังไม่มี `class`
เลยจึงจะเติมค่า default ให้

### 253.5 จำกัดสิทธิ์ด้วย `disabled=True`

สมมติต้องการให้เฉพาะ staff เท่านั้นที่แก้ไขสถานะ `is_published` ได้ ส่วน user
ทั่วไปเห็น field นี้แต่แก้ไม่ได้:

```python
def __init__(self, *args, **kwargs):
    self.user = kwargs.pop("user", None)
    super().__init__(*args, **kwargs)

    if self.user is not None and not self.user.is_staff:
        self.fields["is_published"].disabled = True
        self.fields["is_published"].help_text = "เฉพาะทีมงานเท่านั้นที่เปลี่ยนสถานะนี้ได้"
```

**ทำไมต้องใช้ `disabled = True` ของ Django แทนการซ่อนด้วย CSS หรือลบ field
ทิ้งไปเลย**: field ที่ `disabled=True` จะแสดงผลใน HTML ปกติ (พร้อม attribute
`disabled`) แต่จุดสำคัญคือ **ทาง Django ฝั่ง server จะบังคับใช้ค่า `initial`
เสมอ โดยไม่สนใจค่าที่ส่งมาใน POST data เลย** แม้ว่าจะมีคนพยายามแก้ไข HTML
ด้วย DevTools เพื่อลบ attribute `disabled` ออกแล้วส่งค่าอื่นมาแทนก็ตาม —
นี่คือการป้องกันฝั่ง **server-side ที่แท้จริง** ต่างจากการซ่อนด้วย CSS หรือ
`readonly` attribute ที่เป็นเพียงการป้องกันฝั่ง client-side ที่ผู้ใช้สามารถ
ข้ามได้ง่าย ๆ

### 253.6 ตารางสรุป: สิ่งที่มักปรับผ่าน `__init__`

| ความต้องการ | โค้ดใน `__init__` |
|---|---|
| จำกัด queryset ของ FK/M2M field ตาม user | `self.fields["category"].queryset = Category.objects.filter(...)` |
| ปิดไม่ให้แก้ field ตามสิทธิ์ (ป้องกันฝั่ง server จริง) | `self.fields["is_published"].disabled = True` |
| ลบ field ออกจากฟอร์มแบบมีเงื่อนไข | `del self.fields["category"]` หรือ `self.fields.pop("category", None)` |
| เปลี่ยน field เป็น optional ตามเงื่อนไข | `self.fields["email"].required = False` |
| เติม CSS class ให้ทุก field พร้อมกัน | วนลูป `self.fields.items()` แล้ว `.widget.attrs.setdefault(...)` |
| ตั้งค่า `initial` แบบ dynamic ที่คำนวณจาก user | `self.fields["author"].initial = self.user.get_full_name()` |

> **หมายเหตุ**: การลบ field ออกจากฟอร์มแบบมีเงื่อนไข (`del self.fields[...]`)
> และการสร้างฟอร์มที่ field เปลี่ยนแปลงไปตามข้อมูลอื่นแบบซับซ้อนกว่านี้ (Dynamic
> Forms) จะเจาะลึกอีกครั้งใน **Part 027 ขั้นตอนที่ 268**

---

## ขั้นตอนที่ 254: Model Field Validator กับ ModelForm Validation

### 254.1 ทบทวน Validator ใน Model (จากที่เห็นใน 251.2)

สังเกต Model `Post` ที่ประกาศไว้ในขั้นตอนที่ 251.2 มี validator ติดอยู่ที่ field
โดยตรง:

```python
title = models.CharField(
    max_length=200,
    validators=[MinLengthValidator(5, message="หัวข้อต้องมีความยาวอย่างน้อย 5 ตัวอักษร")],
)
content = models.TextField(validators=[validate_no_banned_words])
```

**ประเด็นสำคัญที่มือใหม่มักไม่รู้**: validator เหล่านี้ **ไม่ทำงานอัตโนมัติ
เมื่อเรียก `Model.objects.create()` หรือ `instance.save()` ตรง ๆ** ต้องเรียก
`instance.full_clean()` ด้วยตัวเองก่อนเสมอถึงจะ trigger validator:

```python
>>> from blog.models import Post
>>> post = Post(title="สั้น", content="เนื้อหาปกติ", slug="test", author_id=1)
>>> post.save()   # บันทึกสำเร็จ! validator ไม่ถูกเรียกเลย เพราะ .save() ไม่เรียก full_clean() ให้
>>> post.title
'สั้น'   # ผ่านทั้งที่สั้นกว่า 5 ตัวอักษร (ตัวอักษรเดียวคือ "สั้น" มี 2 ตัวจริง ๆ)
```

นี่เป็นการออกแบบที่ตั้งใจของ Django (เพื่อความยืดหยุ่น เช่น การ import ข้อมูล
จำนวนมากผ่าน script ที่ไม่ต้องการ overhead จาก validation ทุกครั้ง) แต่ก็เป็น
กับดักสำหรับมือใหม่ที่คาดหวังว่า validator จะทำงานเสมอ

### 254.2 `ModelForm` เรียก `full_clean()` ให้อัตโนมัติ — นี่คือจุดต่างสำคัญ

**`ModelForm.is_valid()` เรียก `instance.full_clean()` ให้อัตโนมัติเสมอ**
เป็นส่วนหนึ่งของขั้นตอนภายในที่เรียกว่า **`_post_clean()`** — ทำให้ validator
ที่ประกาศไว้ใน Model **ทำงานทุกครั้งที่ผ่าน `ModelForm`** แม้ว่าฟอร์มจะไม่ได้
เขียน validation อะไรเพิ่มเติมเองเลยก็ตาม

ลำดับขั้นตอนเต็มของ `ModelForm.is_valid()` (ต่อยอดจากลำดับของ `forms.Form`
ที่เรียนใน Part 025 ขั้นตอนที่ 243.3):

| ลำดับ | ขั้นตอน | ทำอะไร |
|---|---|---|
| 1 | Field-level validation | ตรวจ `required`, `max_length` ฯลฯ ของแต่ละ Form field ตามปกติ |
| 2 | `clean_<fieldname>()` | รัน custom validation ต่อ field ถ้ามีเขียนไว้ (เหมือน `forms.Form` ทุกประการ) |
| 3 | `clean()` ของฟอร์ม | รัน custom validation ข้าม field ถ้ามีเขียนไว้ |
| 4 | **`_post_clean()`** (มีเฉพาะใน `ModelForm`) | สร้าง instance จาก `cleaned_data` แล้วเรียก `instance.full_clean()` |
| 4.1 | ↳ `instance.clean_fields()` | รัน **validator ของทุก Model field** (`validators=[...]` ใน `models.py`) |
| 4.2 | ↳ `instance.clean()` | รัน custom validation ระดับ Model เอง (ถ้า override `Model.clean()` ไว้) |
| 4.3 | ↳ `instance.validate_unique()` | ตรวจ `unique=True` / `unique_together` กับข้อมูลในฐานข้อมูลจริง |
| 5 | รวม error ทั้งหมด | ถ้าไม่มี error เลยตลอดขั้นตอน 1-4 → `is_valid()` คืน `True` |

ขั้นตอนที่ 4 (`_post_clean()`) นี่เองที่เป็นเหตุผลว่า **ทำไม validator ที่เขียน
ไว้ใน Model จึงทำงานโดยอัตโนมัติผ่าน `ModelForm` ทั้งที่ไม่ทำงานเมื่อเรียก
`.save()` ตรง ๆ** — `ModelForm` เป็นเสมือน "ประตู" ที่บังคับให้ข้อมูลผ่าน
`full_clean()` เสมอก่อนจะไปถึงขั้นตอนบันทึกจริง

### 254.3 ทดสอบ: Error จาก Model Validator ปรากฏใน ModelForm โดยไม่ต้องเขียนอะไรเพิ่ม

```python
>>> from blog.forms import PostForm
>>> form = PostForm(data={
...     "title": "สั้น",
...     "slug": "test-post",
...     "content": "คลิกที่นี่เพื่อรับรางวัลฟรี",
...     "is_published": False,
... })
>>> form.is_valid()
False
>>> form.errors
{'title': ['หัวข้อต้องมีความยาวอย่างน้อย 5 ตัวอักษร'],
 'content': ['ห้ามมีคำว่า "คลิกที่นี่" ปรากฏอยู่ในเนื้อหา']}
```

สังเกตว่า **`PostForm` ไม่ได้เขียน `clean_title()` หรือ `clean_content()` เอง
เลยแม้แต่บรรทัดเดียว** — error ทั้งสองข้อความนี้มาจาก `MinLengthValidator` และ
`validate_no_banned_words` ที่ประกาศไว้ใน `models.py` ล้วน ๆ นี่คือพลังของการ
เขียน validator ไว้ที่ Model แทนที่จะเขียนซ้ำในทุกฟอร์มที่เกี่ยวข้องกับ Model
นั้น (ฟอร์มไหนก็ตามที่อ้างอิง `Post.title` จะได้ validator นี้ฟรีทันที รวมถึง
Django Admin ด้วย เพราะ Admin ก็สร้าง `ModelForm` ให้อัตโนมัติเบื้องหลัง
เช่นกัน)

### 254.4 `clean_<field>()` ใน ModelForm: เพิ่มเติมจาก Model Validator ได้ตามปกติ

`clean_<fieldname>()` และ `clean()` ยังคงเขียนได้ใน `ModelForm` เหมือนกับ
`forms.Form` ทุกประการ (เพราะ `ModelForm` สืบทอดพฤติกรรมนี้มาเต็ม ๆ) — และมัน
จะ**ทำงานก่อน** ขั้นตอน `_post_clean()` (Model validator) เสมอ ตามลำดับใน
ตาราง 254.2

ตัวอย่าง: เพิ่มการตรวจสอบรูปแบบ `slug` ที่ Model validator ไม่ครอบคลุม (เช่น
ห้ามขึ้นต้นด้วยตัวเลข แม้ `SlugField` ปกติจะอนุญาต):

```python
class PostForm(forms.ModelForm):
    class Meta:
        model = Post
        fields = ["title", "slug", "content", "category", "tags", "is_published"]

    def clean_slug(self):
        slug = self.cleaned_data["slug"]
        if slug[0].isdigit():
            raise forms.ValidationError("Slug ห้ามขึ้นต้นด้วยตัวเลข")
        return slug
```

เมื่อ `is_valid()` ถูกเรียก ลำดับที่แท้จริงคือ: `clean_slug()` (ที่เราเขียน)
ทำงานก่อน → ถ้าผ่าน ค่อยไปถึง `_post_clean()` ที่ตรวจ `MinLengthValidator` ของ
`title` และ `validate_no_banned_words` ของ `content` ต่อ — **ทั้งสองชั้นทำงาน
ร่วมกันได้อย่างสมบูรณ์ ไม่ชนกัน** เพราะแต่ละ field ถูกตรวจครบทุกชั้นก่อนที่
ฟอร์มจะสรุปผลรวมเป็น `form.errors`

### 254.5 `validate_unique()` ทำงานอัตโนมัติผ่าน ModelForm เช่นกัน

`Post.slug` ประกาศไว้ว่า `unique=True` — ลองทดสอบสร้าง Post ที่ slug ซ้ำกับ
ที่มีอยู่แล้ว:

```python
>>> Post.objects.create(title="บทความแรก", slug="my-first-post", content="...", author_id=1)
>>> form = PostForm(data={
...     "title": "บทความที่สอง",
...     "slug": "my-first-post",   # ซ้ำกับที่มีอยู่แล้ว!
...     "content": "เนื้อหาปกติทั่วไปไม่มีปัญหาอะไร",
... })
>>> form.is_valid()
False
>>> form.errors
{'slug': ['Post with this Slug already exists.']}
```

ข้อความ error นี้ Django สร้างให้อัตโนมัติจากขั้นตอน `instance.validate_unique()`
ใน `_post_clean()` (ตาราง 254.2 แถวที่ 4.3) โดยที่เราไม่ได้เขียน validation
เรื่อง uniqueness เองเลย — **สำหรับกรณีแก้ไข object เดิม** (`ModelForm(instance=post)`)
Django ฉลาดพอที่จะ **ไม่นับ instance ปัจจุบันเป็นการซ้ำกับตัวมันเอง** (มันรู้
ว่ากำลังแก้ไข object ที่มี pk นี้อยู่แล้ว จึง exclude ตัวเองออกจากการเช็ค
uniqueness โดยอัตโนมัติ)

### 254.6 ข้อควรระวัง: Field ที่ไม่อยู่ใน `Meta.fields` จะถูกยกเว้นจาก Validator ด้วย

ถ้า field ใดไม่ได้อยู่ใน `Meta.fields` (เช่น `Comment.post` ในตัวอย่างขั้นตอนที่
252) `_post_clean()` จะรู้ตัวว่าฟอร์มไม่มีข้อมูลของ field นั้น และจะ **ข้าม
การตรวจ validator ของ field นั้นไปโดยอัตโนมัติ** (ผ่าน mechanism ภายในชื่อ
`_get_validation_exclusions()`) นี่คือเหตุผลที่ตัวอย่าง `CommentForm(fields=
["author", "text"])` ในขั้นตอนที่ 252 ไม่ error เรื่อง `post` ตอนเรียก
`is_valid()` — แต่ก็ยังต้องเติมค่า `post` เองก่อน `.save()` จริงอยู่ดี เพราะ
`NOT NULL constraint` ของฐานข้อมูลไม่รู้จักแนวคิด "field ที่ฟอร์มไม่ได้ตรวจ"
เลย มันตรวจแค่ตอนที่ SQL ยิงจริงเท่านั้น — นี่คือเหตุผลที่ต้องใช้ pattern
`commit=False` เติมค่าก่อนเสมอตามที่เรียนในขั้นตอนที่ 252

> **หมายเหตุ**: Part นี้แนะนำแค่การใช้ validator ที่มีอยู่แล้ว (`MinLengthValidator`)
> และ custom validator function อย่างง่าย (`validate_no_banned_words`) เพื่อ
> ให้เห็นความสัมพันธ์กับ `ModelForm` เท่านั้น ส่วนแคตตาล็อกเต็มของ built-in
> validator ทั้งหมด (`RegexValidator`, `EmailValidator`, `URLValidator` ฯลฯ)
> และการเขียน custom validator แบบเจาะลึกจะเรียนเต็มรูปแบบใน **Part 027
> ขั้นตอนที่ 261-262**

---

## ขั้นตอนที่ 255: Formset เบื้องต้นด้วย `formset_factory`

### 255.1 Formset คืออะไร

**Formset** คือกลไกของ Django สำหรับจัดการ **ฟอร์มชนิดเดียวกันหลายชุดพร้อมกัน**
ในหน้าเดียว เช่น ฟอร์มเพิ่มแท็กหลายชื่อพร้อมกันในครั้งเดียว แทนที่จะต้องกด
"เพิ่ม" ทีละชื่อ Django สร้าง Formset จาก Form class ที่มีอยู่แล้วผ่านฟังก์ชัน
`formset_factory()` โดยไม่ต้องเขียนคลาสใหม่:

```python
# blog/forms.py
from django import forms
from django.forms import formset_factory


class TagNameForm(forms.Form):
    name = forms.CharField(max_length=50, label="ชื่อแท็ก")


TagFormSet = formset_factory(TagNameForm, extra=3)
```

`formset_factory(TagNameForm, extra=3)` สร้าง **class ใหม่ชื่อ `TagFormSet`**
ขึ้นมาแบบ dynamic (คล้ายกับที่ `modelformset_factory`/`inlineformset_factory`
ทำในขั้นตอนถัดไป) พารามิเตอร์ `extra=3` บอกว่า **เมื่อสร้าง formset แบบ unbound
(ไม่มี data) ให้แสดงฟอร์มเปล่า 3 ชุด** ให้กรอกทันที

### 255.2 View ที่ใช้ Formset

```python
# blog/views.py
from django.shortcuts import render, redirect
from .forms import TagFormSet
from .models import Tag


def bulk_add_tags_view(request):
    if request.method == "POST":
        formset = TagFormSet(request.POST)
        if formset.is_valid():
            for form in formset:
                name = form.cleaned_data.get("name")
                if name:   # ข้ามฟอร์มที่เว้นว่างไว้ (ไม่ได้กรอกอะไรเลย)
                    Tag.objects.get_or_create(name=name)
            return redirect("blog:tag_list")
    else:
        formset = TagFormSet()

    return render(request, "blog/bulk_add_tags.html", {"formset": formset})
```

สังเกตว่า `formset.is_valid()` ตรวจ**ทุกฟอร์มในชุดพร้อมกัน** และคืน `True`
ก็ต่อเมื่อ**ทุกฟอร์ม** valid หมด (ฟอร์มที่เว้นว่างไว้ทั้งหมดโดยไม่กรอกอะไรเลย
จะไม่นับเป็น invalid — Django ใช้ method `form.has_changed()` ภายในเพื่อ
ตรวจจับ "ฟอร์มที่ไม่มีข้อมูลอะไรเลย" แล้วข้ามการ validate `required` ให้
โดยอัตโนมัติ เพื่อรองรับกรณีผู้ใช้ไม่อยากกรอกครบทุกช่องที่เตรียมไว้)

### 255.3 Template: ต้องมี `{{ formset.management_form }}` เสมอ

```html
<!-- blog/templates/blog/bulk_add_tags.html -->
{% extends 'base.html' %}

{% block content %}
<h1>เพิ่มแท็กหลายรายการพร้อมกัน</h1>
<form method="post">
    {% csrf_token %}
    {{ formset.management_form }}
    {% for form in formset %}
        <div class="tag-row">
            {{ form.as_p }}
        </div>
    {% endfor %}
    <button type="submit">บันทึกทั้งหมด</button>
</form>
{% endblock %}
```

> **กฎเหล็ก**: ทุกครั้งที่ render formset ใน template **ต้องมี
> `{{ formset.management_form }}` อยู่ในฟอร์มเสมอ** มันไม่แสดงผลอะไรให้เห็น
> (เป็นแค่ hidden input หลายตัว — จะเจาะลึกในขั้นตอนที่ 258) แต่ Django ใช้
> ข้อมูลจากมันเพื่อรู้ว่า **ต้องประมวลผลกี่ฟอร์มในชุดนี้** ถ้าลืมใส่ ตอน POST
> กลับมา Django จะ raise `ValidationError: ManagementForm data is missing or
> has been tampered with` ทันที เป็น error ที่มือใหม่เจอบ่อยที่สุดเมื่อเริ่ม
> ใช้ formset ครั้งแรก

### 255.4 `formset.forms` และ `formset.empty_form`

`formset` เป็น iterable โดยตรง (`for form in formset` วนลูปได้เลยเหมือนใน
ตัวอย่างด้านบน) แต่ก็เข้าถึง list ของฟอร์มทั้งหมดผ่าน `formset.forms` ได้เช่นกัน
(`{% for form in formset %}` เทียบเท่ากับ `{% for form in formset.forms %}`)

`formset.empty_form` คือฟอร์มพิเศษที่มี prefix เป็น `__prefix__` (placeholder
ที่ยังไม่ใช่ index จริง) ใช้เป็น "แม่แบบ" สำหรับเพิ่มฟอร์มใหม่ด้วย JavaScript
แบบ dynamic โดยไม่ต้อง reload หน้า — จะสาธิตการใช้งานเต็มรูปแบบในขั้นตอนที่
258.5

---

## ขั้นตอนที่ 256: `modelformset_factory` — แก้ไขหลาย Object พร้อมกันในหน้าเดียว

### 256.1 ต่างจาก `formset_factory` อย่างไร

`formset_factory` สร้าง formset จาก plain `forms.Form` ที่ไม่ผูกกับ Model เลย
ส่วน **`modelformset_factory`** สร้าง formset ที่แต่ละฟอร์มข้างในเป็น
**`ModelForm`** ผูกกับ object จริงในฐานข้อมูล — เหมาะกับหน้า "แก้ไขหลาย object
พร้อมกันในตารางเดียว" ซึ่งเป็นสิ่งที่ Django Admin ใช้ในหน้า changelist
(เวลาติ๊กเลือกหลายแถวแล้วแก้ไขพร้อมกัน) นั่นเอง

```python
# blog/forms.py
from django.forms import modelformset_factory
from .models import Post

PostFormSet = modelformset_factory(
    Post,
    fields=["title", "is_published"],
    extra=0,   # ไม่ต้องการฟอร์มเปล่าเพิ่ม เอาแค่ object ที่มีอยู่แล้ว
)
```

พารามิเตอร์ตัวแรกของ `modelformset_factory()` คือ Model โดยตรง (ไม่ต้องเขียน
`ModelForm` เองก่อนก็ได้ — Django สร้างให้อัตโนมัติจาก `fields` ที่ระบุ เหมือน
กับที่ `ModelForm.Meta` ทำ) หรือถ้าต้องการ validation พิเศษ (เช่น `clean_title()`
ที่เขียนไว้แล้ว) ก็ส่ง `form=PostForm` เข้าไปแทนได้:

```python
PostFormSet = modelformset_factory(
    Post,
    form=PostForm,   # ใช้ ModelForm ที่เขียน validation ไว้เองแทน
    extra=0,
)
```

### 256.2 View: แก้ไขโพสต์ของ user คนเดียวทั้งหมดในหน้าเดียว

```python
# blog/views.py
from django.contrib.auth.decorators import login_required
from django.shortcuts import render, redirect
from .forms import PostFormSet
from .models import Post


@login_required
def bulk_edit_posts_view(request):
    queryset = Post.objects.filter(author=request.user).order_by("-created_at")

    if request.method == "POST":
        formset = PostFormSet(request.POST, queryset=queryset)
        if formset.is_valid():
            formset.save()   # บันทึกทุกฟอร์มที่มีการแก้ไขพร้อมกัน
            return redirect("blog:my_posts")
    else:
        formset = PostFormSet(queryset=queryset)

    return render(request, "blog/bulk_edit_posts.html", {"formset": formset})
```

พารามิเตอร์ **`queryset=`** เป็นตัวกำหนดว่า formset จะแสดง object ไหนบ้าง —
ต่างจาก `formset_factory` ที่ไม่มี concept ของ object ที่มีอยู่แล้วเลย
`modelformset_factory` ใช้ `queryset` เพื่อรู้ว่า **แต่ละฟอร์มควรผูกกับ object
ตัวไหน** (แต่ละแถวใน queryset จะกลายเป็นหนึ่งฟอร์มที่มี `form.instance` ชี้ไป
ที่ object นั้น พร้อม pre-fill ค่าปัจจุบันให้อัตโนมัติ)

`formset.save()` จะวนลูปทุกฟอร์มที่ **มีการเปลี่ยนแปลงข้อมูลจริง** (เช็คด้วย
`has_changed()` เหมือนกัน) แล้วเรียก `.save()` ของแต่ละฟอร์มให้ พร้อมคืน list
ของ instance ที่ถูกบันทึกกลับมา — ฟอร์มที่ผู้ใช้ไม่ได้แก้ไขอะไรเลยจะถูกข้าม
ไปโดยไม่ยิง `UPDATE` โดยเปล่าประโยชน์

### 256.3 ทำไมต้องตั้ง `extra=0` สำหรับหน้าแก้ไขอย่างเดียว

ค่าเริ่มต้นของ `extra` คือ `1` — ถ้าไม่ระบุ `extra=0` หน้าแก้ไขจะมีฟอร์มเปล่า
เพิ่มมาอีก 1 ชุดต่อท้ายเสมอ ซึ่งถ้าผู้ใช้บังเอิญกรอกอะไรลงไปในฟอร์มเปล่านั้น
(ทั้งที่ไม่ได้ตั้งใจจะสร้าง Post ใหม่) `formset.save()` **จะสร้าง Post ใหม่ให้
โดยอัตโนมัติ** เพราะมันไม่มี pk ผูกอยู่ — สำหรับหน้าที่ตั้งใจให้ "แก้ไขเท่านั้น"
ควรตั้ง `extra=0` เสมอเพื่อป้องกันการสร้าง object โดยไม่ได้ตั้งใจ

### 256.4 การสร้าง Object ใหม่พร้อมกับแก้ไขของเดิมในหน้าเดียว

ถ้าต้องการอนุญาตให้ "เพิ่มโพสต์ใหม่" ในหน้าเดียวกันด้วย ให้ตั้ง `extra` เป็น
จำนวนที่มากกว่า 0:

```python
PostFormSet = modelformset_factory(
    Post,
    fields=["title", "is_published"],
    extra=2,   # เผื่อฟอร์มเปล่าไว้เพิ่มโพสต์ใหม่ได้อีก 2 รายการ
)
```

ฟอร์มที่เกินจำนวน `queryset` เดิม (เรียกว่า **extra forms**) จะเป็นฟอร์มเปล่า
ไม่มี `instance.pk` — ถ้าผู้ใช้กรอกข้อมูลลงไป `formset.save()` จะสร้าง object
ใหม่ให้ ถ้าปล่อยว่างไว้ก็จะถูกข้ามไปเฉย ๆ (เป็นพฤติกรรมเดียวกับ `formset_factory`
ในขั้นตอนที่ 255)

### 256.5 ตารางสรุป: `extra`, `queryset` ส่งผลต่อพฤติกรรมอย่างไร

| ค่า `extra` | ค่า `queryset` | ผลลัพธ์ |
|---|---|---|
| `0` | มี object 5 ตัว | แสดง 5 ฟอร์ม (แก้ไขได้อย่างเดียว ไม่มีฟอร์มเปล่าเพิ่ม) |
| `2` | มี object 5 ตัว | แสดง 5 ฟอร์ม (แก้ไขของเดิม) + 2 ฟอร์มเปล่า (เพิ่มใหม่ได้) |
| `3` | ไม่ระบุ (queryset ว่างเปล่า) | แสดง 3 ฟอร์มเปล่าล้วน (ใช้เป็นหน้าสร้างหลาย object พร้อมกันครั้งแรก) |

---

## ขั้นตอนที่ 257: `inlineformset_factory` — จัดการ Parent-Child Form

### 257.1 ความสัมพันธ์ Parent-Child: `Post` กับ `PostImage`

`inlineformset_factory` คือ formset พิเศษที่ออกแบบมาสำหรับความสัมพันธ์
**หนึ่งไปหลาย (One-to-Many)** ผ่าน `ForeignKey` โดยเฉพาะ — ใช้เมื่อต้องการ
แก้ไข **parent object 1 ตัว พร้อมกับ child object หลายตัวที่เชื่อมกันอยู่**
ในหน้าเดียว เช่น หน้าแก้ไข `Post` ที่ต้องการจัดการ `PostImage` (หลายรูปภาพ
ที่ผูกกับ Post นั้น) ไปพร้อมกัน:

```python
# blog/forms.py
from django.forms import inlineformset_factory
from .models import Post, PostImage

PostImageFormSet = inlineformset_factory(
    Post,             # parent model
    PostImage,        # child model (ต้องมี ForeignKey ชี้กลับไปหา parent)
    fields=["image", "caption", "is_primary", "order"],
    extra=3,
    can_delete=True,
)
```

`inlineformset_factory(Post, PostImage, ...)` **หา `ForeignKey` ของ `PostImage`
ที่ชี้ไปยัง `Post` ให้อัตโนมัติ** (คือ field `post`) แล้ว**ซ่อน field นั้นออก
จากฟอร์มโดยสิ้นเชิง** — ผู้ใช้ไม่มีโอกาสเลือกหรือแก้ไข parent เองเลย เพราะ
formset จะเซ็ตค่านี้ให้อัตโนมัติจาก `instance=` ที่ส่งเข้าไปตอนสร้าง formset
เสมอ (ถ้า `PostImage` มี `ForeignKey` มากกว่า 1 ตัวที่ชี้ไปยัง `Post` ต้องระบุ
`fk_name="post"` เพิ่มเพื่อบอกว่าจะใช้ field ไหนเป็นตัวเชื่อม)

### 257.2 View: แก้ไข Post พร้อมรูปภาพในหน้าเดียว (กรณีแก้ไข Object ที่มีอยู่แล้ว)

```python
# blog/views.py
from django.contrib.auth.decorators import login_required
from django.db import transaction
from django.shortcuts import render, redirect, get_object_or_404
from .forms import PostForm, PostImageFormSet
from .models import Post


@login_required
def post_edit_view(request, slug):
    post = get_object_or_404(Post, slug=slug, author=request.user)

    if request.method == "POST":
        post_form = PostForm(request.POST, instance=post, user=request.user)
        image_formset = PostImageFormSet(
            request.POST, request.FILES, instance=post, prefix="images",
        )
        if post_form.is_valid() and image_formset.is_valid():
            with transaction.atomic():
                post = post_form.save()
                image_formset.save()
            return redirect("blog:detail", slug=post.slug)
    else:
        post_form = PostForm(instance=post, user=request.user)
        image_formset = PostImageFormSet(instance=post, prefix="images")

    return render(request, "blog/post_form.html", {
        "post_form": post_form,
        "image_formset": image_formset,
    })
```

จุดสำคัญที่ควรสังเกต:

- **`instance=post`** ถูกส่งเข้าทั้ง `PostForm` และ `PostImageFormSet` —
  ฝั่ง formset ใช้มันเพื่อรู้ว่า "จัดการ `PostImage` ที่ผูกกับ `post` ตัวนี้
  เท่านั้น" (เทียบเท่ากับการกรอง `PostImage.objects.filter(post=post)` ให้
  อัตโนมัติ)
- **`prefix="images"`** ตั้งชื่อ prefix ของ field ใน formset นี้อย่างชัดเจน
  (ผลลัพธ์คือ input จะมีชื่อ `images-0-image`, `images-0-caption` ฯลฯ) —
  จำเป็นเมื่อหน้าเดียวมีมากกว่าหนึ่งฟอร์ม/formset พร้อมกัน เพื่อไม่ให้ชื่อ
  field ชนกัน (ถ้าไม่ระบุ Django จะใช้ prefix อัตโนมัติจากชื่อ related_name
  ของ Model ซึ่งอ่านเข้าใจยากกว่า)
- **`transaction.atomic()`** ห่อการบันทึกทั้งสองส่วนไว้ด้วยกัน — ถ้า
  `image_formset.save()` ล้มเหลวกลางคัน (เช่น เขียนไฟล์รูปไม่สำเร็จ) การแก้ไข
  `post_form.save()` ที่ทำไปก่อนหน้าจะถูก **rollback กลับเหมือนไม่มีอะไรเกิดขึ้น**
  ป้องกันไม่ให้ฐานข้อมูลอยู่ในสถานะครึ่ง ๆ กลาง ๆ (จะเจาะลึกเรื่อง Transaction
  เต็มรูปแบบใน Phase ถัดไป)

### 257.3 กรณีสร้าง Post ใหม่พร้อมรูปภาพ (ยังไม่มี Parent Instance)

การสร้างใหม่ซับซ้อนกว่าการแก้ไขเล็กน้อย เพราะ `PostImage` ต้องการ `post_id`
ที่มีอยู่จริงก่อนถึงจะบันทึกได้ (คล้ายปัญหา M2M ในขั้นตอนที่ 252.3) แนวทาง
มาตรฐานคือ **บันทึก parent (`Post`) ให้เสร็จก่อน แล้วค่อยผูก formset เข้ากับ
instance ที่มี pk แล้วอีกที**:

```python
@login_required
def post_create_with_images_view(request):
    if request.method == "POST":
        post_form = PostForm(request.POST, user=request.user)
        # ตอนนี้ post ยังไม่มี pk เลย ส่ง instance=Post() (ว่างเปล่า) ไปพลาง ๆ ก่อน
        image_formset = PostImageFormSet(
            request.POST, request.FILES, instance=Post(), prefix="images",
        )

        if post_form.is_valid() and image_formset.is_valid():
            with transaction.atomic():
                post = post_form.save(commit=False)
                post.author = request.user
                post.save()          # ตอนนี้ post มี pk แล้ว
                post_form.save_m2m()

                image_formset.instance = post   # ผูก formset เข้ากับ post ตัวจริง
                image_formset.save()            # ค่อยบันทึกรูปภาพทั้งหมด

            return redirect("blog:detail", slug=post.slug)
    else:
        post_form = PostForm(user=request.user)
        image_formset = PostImageFormSet(instance=Post(), prefix="images")

    return render(request, "blog/post_form.html", {
        "post_form": post_form,
        "image_formset": image_formset,
    })
```

**เทคนิคสำคัญ**: `image_formset.is_valid()` เรียกได้ตามปกติแม้ `instance=Post()`
จะยังไม่มี pk เลยก็ตาม เพราะขั้นตอน validate ของแต่ละฟอร์มลูกไม่จำเป็นต้องรู้
pk ของ parent (field `post` ถูกซ่อนออกจากฟอร์มไปแล้วตั้งแต่ต้น) — สิ่งที่ต้อง
รอ pk จริง ๆ คือขั้นตอน **`.save()`** เท่านั้น จึงต้องเปลี่ยน
`image_formset.instance = post` (แทนที่ `Post()` เปล่า ด้วย post ตัวจริงที่มี
pk แล้ว) **ก่อน** เรียก `image_formset.save()` เสมอ

### 257.4 Template: Render Formset ที่ผูกกับ Parent Form

```html
<!-- blog/templates/blog/post_form.html -->
{% extends 'base.html' %}

{% block content %}
<h1>{% if post_form.instance.pk %}แก้ไขบทความ{% else %}เขียนบทความใหม่{% endif %}</h1>

<form method="post" enctype="multipart/form-data">
    {% csrf_token %}
    {{ post_form.as_p }}

    <h3>รูปภาพประกอบ</h3>
    {{ image_formset.management_form }}
    {% for form in image_formset %}
        <div class="image-row">
            {{ form.as_p }}
            {% if form.instance.pk %}
                <label>{{ form.DELETE }} ลบรูปนี้</label>
            {% endif %}
        </div>
    {% endfor %}
    {% if image_formset.non_form_errors %}
        <div class="form-error">{{ image_formset.non_form_errors }}</div>
    {% endif %}

    <button type="submit">บันทึก</button>
</form>
{% endblock %}
```

สังเกต `enctype="multipart/form-data"` ที่ `<form>` — จำเป็นเพราะ
`PostImageFormSet` มี field `image` เป็น `ImageField` (ทบทวนกฎเหล็กจาก Part 025
ขั้นตอนที่ 248.1) และ `{{ form.DELETE }}` คือ checkbox ที่ `can_delete=True`
เพิ่มให้อัตโนมัติ (จะเจาะลึกในขั้นตอนที่ 258.3) — แสดงเฉพาะฟอร์มที่ผูกกับ
object ที่มีอยู่แล้วเท่านั้น (`form.instance.pk` มีค่า) เพราะฟอร์มเปล่าที่ยัง
ไม่มีรูปจริงไม่มีอะไรให้ลบ

### 257.5 ตารางเปรียบเทียบ Formset ทั้ง 3 ชนิด

| | `formset_factory` | `modelformset_factory` | `inlineformset_factory` |
|---|---|---|---|
| ผูกกับ Model ไหม | ไม่ผูก (plain `Form`) | ผูก (`ModelForm`) | ผูก (`ModelForm`) |
| ต้องมี parent-child relationship ไหม | ไม่ต้อง | ไม่ต้อง | **ต้อง** มี `ForeignKey` เชื่อมสองฝั่ง |
| ใช้ parameter ไหนกำหนด object | ไม่มี (สร้างใหม่ล้วน) | `queryset=` | `instance=` (ของ parent) |
| ตัวอย่าง use case | เพิ่มแท็กหลายชื่อพร้อมกัน (ไม่ผูก Model) | แก้ไขสถานะ `is_published` ของหลายโพสต์พร้อมกัน | แก้ไข `Post` พร้อมจัดการ `PostImage` หลายรูปพร้อมกัน |
| field เชื่อม parent ถูกซ่อนอัตโนมัติไหม | ไม่เกี่ยว | ไม่เกี่ยว | **ใช่** ซ่อนอัตโนมัติเสมอ |

---

## ขั้นตอนที่ 258: Formset Management Form เจาะลึก

### 258.1 Hidden Input ที่ `{{ formset.management_form }}` สร้างให้

ลอง inspect HTML ที่เกิดจาก `{{ image_formset.management_form }}` ในตัวอย่าง
ขั้นตอนที่ 257 (สมมติมีรูปเดิมอยู่แล้ว 2 รูป และ `extra=3`):

```html
<input type="hidden" name="images-TOTAL_FORMS" value="5">
<input type="hidden" name="images-INITIAL_FORMS" value="2">
<input type="hidden" name="images-MIN_NUM_FORMS" value="0">
<input type="hidden" name="images-MAX_NUM_FORMS" value="1000">
```

| Field | ความหมาย |
|---|---|
| `TOTAL_FORMS` | จำนวนฟอร์มทั้งหมดที่ถูกส่งมาใน request นี้ (2 เดิม + 3 extra = 5) — **Django ใช้ตัวเลขนี้เพื่อรู้ว่าต้องวนลูปสร้าง Form object กี่ตัวตอนรับ POST กลับมา** |
| `INITIAL_FORMS` | จำนวนฟอร์มที่ผูกกับ object ที่มีอยู่แล้วจริง ณ ตอน render (2 รูปเดิม) — ใช้แยกว่าฟอร์มไหนคือ "ของเดิม" (`instance.pk` มีค่า) กับฟอร์มไหนคือ "extra" (ของใหม่) |
| `MIN_NUM_FORMS` | ค่าต่ำสุดที่ยอมรับ มาจากพารามิเตอร์ `min_num` |
| `MAX_NUM_FORMS` | ค่าสูงสุดที่ยอมรับ มาจากพารามิเตอร์ `max_num` |

> **กฎเหล็ก**: ค่าพวกนี้เป็น hidden input ที่ **ผู้ใช้ (หรือบอท) สามารถแก้ไข
> ผ่าน DevTools ได้ในทางเทคนิค** — แต่ Django มีการตรวจสอบฝั่ง server อยู่แล้ว
> ว่าตัวเลขที่ส่งมาต้องสอดคล้องกับข้อมูลจริง (เช่นถ้า `TOTAL_FORMS` บอกว่ามี 10
> ฟอร์ม แต่ field ของฟอร์มที่ 8-10 ไม่มีข้อมูลส่งมาเลย ฟอร์มเหล่านั้นจะถูก
> ปฏิบัติเหมือนฟอร์มเปล่าตามปกติ ไม่ใช่ error) สิ่งที่ต้องระวังจริง ๆ คือเวลา
> เขียน JavaScript เพิ่มฟอร์มเอง (258.5) ต้อง**อัปเดตค่า `TOTAL_FORMS` ให้ตรง
> กับจำนวนฟอร์มที่ render จริงบนหน้าเว็บเสมอ** ไม่เช่นนั้นฟอร์มที่เพิ่มมาจะ
> ไม่ถูกประมวลผลเลยฝั่ง server (เพราะ loop ตาม `TOTAL_FORMS` เดิมเท่านั้น)

### 258.2 `extra`: จำนวนฟอร์มเปล่าเพิ่มเติม

ทบทวนจากขั้นตอนที่ 255-257: `extra=N` กำหนดจำนวนฟอร์มเปล่าที่จะแสดงเพิ่ม
**นอกเหนือจาก** จำนวนที่มาจาก `queryset`/`instance` เดิม พารามิเตอร์นี้มีผล
เฉพาะตอน **render ฟอร์มแบบ unbound** (GET) เท่านั้น — ตอน POST กลับมา
`TOTAL_FORMS` จาก management form จะเป็นตัวกำหนดจำนวนฟอร์มจริงแทน ไม่ใช่ `extra`

### 258.3 `can_delete` และ `can_delete_extra`

```python
PostImageFormSet = inlineformset_factory(
    Post, PostImage,
    fields=["image", "caption", "is_primary", "order"],
    extra=1,
    can_delete=True,          # เพิ่ม checkbox "DELETE" ให้ทุกฟอร์ม
    can_delete_extra=False,   # แต่ไม่ต้องมี checkbox นี้ในฟอร์ม extra ที่ยังว่างเปล่า
)
```

- **`can_delete=True`** เพิ่ม field พิเศษชื่อ `DELETE` (`BooleanField`,
  `required=False`) ให้ทุกฟอร์มในชุดโดยอัตโนมัติ เมื่อผู้ใช้ติ๊กช่องนี้แล้ว
  submit `formset.save()` จะ **ลบ object ที่ฟอร์มนั้นผูกอยู่ออกจากฐานข้อมูล
  ทันที** (แทนที่จะอัปเดต)
- **`can_delete_extra`** (Django 4.1+) กำหนดว่าฟอร์ม extra (ที่ยังไม่มี
  `instance.pk` เพราะเป็นฟอร์มว่างสำหรับสร้างใหม่) ควรมี checkbox `DELETE`
  ด้วยหรือไม่ — ปกติควรตั้งเป็น `False` เพราะการ "ลบ" ฟอร์มที่ยังไม่มีข้อมูล
  อะไรเลยไม่มีความหมายอะไร มีแต่จะสร้างความสับสนให้ผู้ใช้

### 258.4 `max_num`, `min_num`, `validate_max`, `validate_min`

```python
PostImageFormSet = inlineformset_factory(
    Post, PostImage,
    fields=["image", "caption", "is_primary", "order"],
    extra=1,
    max_num=10,
    min_num=1,
    validate_max=True,
    validate_min=True,
)
```

| พารามิเตอร์ | ผลลัพธ์ |
|---|---|
| `max_num=10` (ไม่มี `validate_max`) | จำกัดแค่ **จำนวนฟอร์มที่ render** ไม่เกิน 10 (ตัด `extra` ส่วนเกินทิ้ง) แต่**ไม่บังคับ** ตอน validate ถ้ามีคนส่งข้อมูลเกิน 10 ฟอร์มมาทาง POST ตรง ๆ |
| `max_num=10, validate_max=True` | เพิ่มการ**บังคับ**ตอน `is_valid()` ด้วย — ถ้าจำนวนฟอร์มที่มีข้อมูล (ไม่นับฟอร์มว่าง) เกิน 10 จะ error ทันที |
| `min_num=1` (ไม่มี `validate_min`) | มีผลแค่กับการ**คำนวณจำนวน `extra` ที่ต้องแสดง** เพื่อให้รวมแล้วไม่น้อยกว่า `min_num` แต่ไม่บังคับ validate |
| `min_num=1, validate_min=True` | **บังคับ**ว่าต้องมีอย่างน้อย 1 ฟอร์มที่กรอกข้อมูลจริง ไม่เช่นนั้น `is_valid()` จะคืน `False` พร้อม error ใน `formset.non_form_errors()` |

สำหรับ use case "ต้องมีรูปภาพอย่างน้อย 1 รูปเสมอ" การตั้ง `min_num=1,
validate_min=True` คือวิธีมาตรฐานที่สั้นที่สุด — แต่ถ้าต้องการเงื่อนไขที่
ซับซ้อนกว่านั้น (เช่น "ต้องมีรูปที่ติ๊ก `is_primary` อย่างน้อย 1 รูป" ซึ่งเป็น
เงื่อนไขเกี่ยวกับ**ค่าใน field** ไม่ใช่แค่จำนวนฟอร์ม) ต้องเขียน
`formset.clean()` เอง ซึ่งเป็นเนื้อหาของขั้นตอนถัดไป

### 258.5 เพิ่มฟอร์มใหม่แบบ Dynamic ด้วย JavaScript (ปุ่ม "เพิ่มอีกแถว")

ใช้ `formset.empty_form` เป็นแม่แบบ (มี prefix เป็น `__prefix__` placeholder)
แล้วแทนที่ด้วย index จริงตอนคลิกปุ่ม:

```html
<!-- blog/templates/blog/post_form.html (ส่วนขยายจากขั้นตอนที่ 257.4) -->
<div id="image-forms-container">
    {% for form in image_formset %}
        <div class="image-row">{{ form.as_p }}</div>
    {% endfor %}
</div>

<template id="empty-form-template">
    <div class="image-row">{{ image_formset.empty_form.as_p }}</div>
</template>

<button type="button" id="add-image-form">+ เพิ่มรูปภาพอีกแถว</button>
```

```javascript
// blog/static/blog/js/formset.js
document.getElementById("add-image-form").addEventListener("click", function () {
    const totalFormsInput = document.getElementById("id_images-TOTAL_FORMS");
    const formIndex = parseInt(totalFormsInput.value, 10);

    const templateHtml = document
        .getElementById("empty-form-template")
        .innerHTML.replace(/__prefix__/g, formIndex);   // แทน __prefix__ ด้วย index จริง

    document
        .getElementById("image-forms-container")
        .insertAdjacentHTML("beforeend", templateHtml);

    totalFormsInput.value = formIndex + 1;   // อัปเดต TOTAL_FORMS ให้ตรงกับจำนวนฟอร์มจริงเสมอ
});
```

`id="id_images-TOTAL_FORMS"` มาจาก prefix `"images"` ที่ตั้งไว้ตอนสร้าง
formset ในขั้นตอนที่ 257.2 (`PostImageFormSet(..., prefix="images")`) —
รูปแบบ id ของ management form เสมอคือ `id_<prefix>-TOTAL_FORMS` จุดสำคัญ
ที่สุดของโค้ด JS นี้คือบรรทัดสุดท้าย: **ทุกครั้งที่เพิ่มฟอร์มใหม่ด้วย JS ต้อง
เพิ่มค่า `TOTAL_FORMS` ให้ตรงกันเสมอ** ตามคำเตือนในขั้นตอนที่ 258.1 ไม่เช่นนั้น
ฟอร์มที่เพิ่มมาจะไม่ถูกประมวลผลฝั่ง server เลยแม้จะกรอกข้อมูลถูกต้องทุกอย่าง

---

## ขั้นตอนที่ 259: Validation ข้าม Formset ทั้งชุดด้วย `formset.clean()`

### 259.1 ทำไมต้องมี `clean()` ระดับ Formset แยกต่างหาก

`clean_<field>()` และ `clean()` ที่เรียนใน Part 025 ตรวจสอบได้แค่**ภายในฟอร์ม
เดียว** แต่บางเงื่อนไขต้อง **เปรียบเทียบข้อมูลข้ามหลายฟอร์มในชุดเดียวกัน**
เช่น "ต้องมีรูปภาพที่ติ๊ก `is_primary` อย่างน้อย 1 รูปในทั้งชุด" หรือ "ค่า
`order` ของทุกฟอร์มห้ามซ้ำกัน" — เงื่อนไขแบบนี้ไม่มีทางเขียนไว้ใน form เดียว
ได้ เพราะมันต้อง**มองเห็นข้อมูลของฟอร์มอื่นทั้งหมดในชุดพร้อมกัน**

Django รองรับสิ่งนี้ผ่านการเขียน custom formset class ที่สืบทอดจาก
**`BaseModelFormSet`** (สำหรับ `modelformset_factory`) หรือ
**`BaseInlineFormSet`** (สำหรับ `inlineformset_factory`) แล้ว override
method `clean()` ของมัน

### 259.2 เขียน `BasePostImageFormSet`

```python
# blog/forms.py
from django.core.exceptions import ValidationError
from django.forms import BaseInlineFormSet, inlineformset_factory
from .models import Post, PostImage


class BasePostImageFormSet(BaseInlineFormSet):
    def clean(self):
        super().clean()

        # ถ้ามีฟอร์มใดฟอร์มหนึ่งยัง invalid อยู่แล้วจาก field-level validation
        # ไม่ต้องตรวจเงื่อนไขข้ามฟอร์มซ้ำ เพราะ error เดิมจะสับสนปนกับ error ใหม่
        if any(self.errors):
            return

        primary_count = 0
        orders_seen = []

        for form in self.forms:
            # ข้ามฟอร์มที่ไม่มีข้อมูลอะไรเลย (extra form ที่ผู้ใช้ไม่ได้กรอก)
            if not hasattr(form, "cleaned_data") or not form.cleaned_data:
                continue
            # ข้ามฟอร์มที่ผู้ใช้ติ๊กลบทิ้ง (ไม่นับเป็นรูปที่จะเก็บไว้จริง)
            if form.cleaned_data.get("DELETE"):
                continue

            if form.cleaned_data.get("is_primary"):
                primary_count += 1

            order = form.cleaned_data.get("order")
            if order is not None:
                orders_seen.append(order)

        if primary_count == 0:
            raise ValidationError("ต้องเลือกรูปภาพหลัก (is_primary) อย่างน้อย 1 รูป")
        if primary_count > 1:
            raise ValidationError("เลือกรูปภาพหลักได้เพียง 1 รูปเท่านั้น กรุณาเลือกใหม่")
        if len(orders_seen) != len(set(orders_seen)):
            raise ValidationError("ลำดับรูปภาพ (order) ของแต่ละรูปห้ามซ้ำกัน")


PostImageFormSet = inlineformset_factory(
    Post,
    PostImage,
    formset=BasePostImageFormSet,   # ใช้ custom formset class ที่เขียนไว้
    fields=["image", "caption", "is_primary", "order"],
    extra=1,
    can_delete=True,
    can_delete_extra=False,
    min_num=1,
    validate_min=True,
)
```

### 259.3 อธิบายทีละจุดสำคัญ

- **`super().clean()` เป็นบรรทัดแรกเสมอ** — เหมือนกับกฎเหล็กของ `Form.clean()`
  ใน Part 025 ขั้นตอนที่ 246.2 ทุกประการ เพื่อให้ validation พื้นฐานของ
  formset (เช่น `validate_min`/`validate_max` ที่ตั้งค่าไว้) ทำงานก่อนเสมอ
- **`if any(self.errors): return`** — `self.errors` เป็น list ของ error dict
  ของแต่ละฟอร์ม (list ว่างเปล่าถ้าฟอร์มนั้น valid) `any(self.errors)` จะเป็น
  `True` ถ้ามีฟอร์มใดฟอร์มหนึ่งมี error อยู่แล้วจาก field-level validation
  — การ `return` ออกก่อนป้องกันไม่ให้ error ข้ามฟอร์ม (ซึ่งอาจตีความข้อมูล
  ที่ invalid ผิดพลาด) ไปสร้างความสับสนซ้อนกับ error ที่มีอยู่แล้ว
- **`form.cleaned_data.get("DELETE")`** — field พิเศษที่มาจาก
  `can_delete=True` (ขั้นตอนที่ 258.3) ต้องเช็คและข้ามฟอร์มที่ถูกทำเครื่องหมาย
  ลบทิ้ง เพราะไม่ควรเอามานับรวมในเงื่อนไข "ต้องมีรูปหลัก" (รูปที่กำลังจะถูก
  ลบไม่ควรถูกนับเป็นรูปที่ "มีอยู่" อีกต่อไป)
- **error ที่ `raise ValidationError(...)` ในระดับนี้** จะไปปรากฏใน
  `formset.non_form_errors()` (คล้ายกับ `form.non_field_errors()` ของ Part
  025 ขั้นตอนที่ 246.3 แต่เป็นระดับ formset ทั้งชุดแทนที่จะเป็นระดับฟอร์มเดียว)

### 259.4 แสดง `non_form_errors()` ในเทมเพลต

```html
{{ image_formset.management_form }}
{% for form in image_formset %}
    <div class="image-row">{{ form.as_p }}</div>
{% endfor %}

{% if image_formset.non_form_errors %}
    <div class="form-error form-error--general">
        {{ image_formset.non_form_errors }}
    </div>
{% endif %}
```

`non_form_errors()` ต่างจาก `form.errors` ของแต่ละฟอร์มโดยสิ้นเชิง — มันคือ
error ที่เกิดจาก `formset.clean()` เท่านั้น (เทียบเท่ากับ error ที่เกิดจาก
`raise ValidationError(...)` ใน `Form.clean()` แต่ยกระดับขึ้นมาเป็น "ทั้งชุด"
แทนที่จะเป็น "ทั้งฟอร์ม") ควรแสดงแยกต่างหากจาก error ของฟอร์มย่อยแต่ละอันเสมอ
เพื่อไม่ให้ผู้ใช้สับสนว่า error นี้เกี่ยวกับฟอร์มไหนกันแน่

---

## ขั้นตอนที่ 260: สรุปและแบบฝึกหัด

### 260.1 ตัวอย่างเต็ม: แปลง `CommentForm` จาก Part 025 เป็น `ModelForm` ตัวจริง

รวมทุกเทคนิคของ Part นี้เข้าด้วยกัน แปลง `CommentForm` (plain `forms.Form`
จาก Part 025 ขั้นตอนที่ 250.2) ให้เป็น `ModelForm` เต็มรูปแบบ พร้อมคง custom
validation เดิมทั้งหมดไว้ (`clean_author()`, `clean_text()`, `clean()` สำหรับ
honeypot):

```python
# blog/forms.py
from django import forms
from .models import Comment

BANNED_WORDS = ["สแปม", "โฆษณา", "คลิกที่นี่"]


class CommentForm(forms.ModelForm):
    # field ที่ไม่มีอยู่ใน Model เลย (honeypot) ยังคงประกาศเองตรง ๆ ได้ตามปกติ
    # ModelForm อนุญาตให้ผสมทั้ง field จาก Model และ field เสริมแบบนี้ในฟอร์มเดียวกัน
    honeypot = forms.CharField(required=False, widget=forms.HiddenInput)

    class Meta:
        model = Comment
        fields = ["author", "text"]   # ไม่รวม "post" เพราะต้องเซ็ตจาก View เท่านั้น
        widgets = {
            "author": forms.TextInput(attrs={
                "class": "form-control",
                "placeholder": "ชื่อที่จะแสดงบนความคิดเห็น",
            }),
            "text": forms.Textarea(attrs={
                "class": "form-control",
                "rows": 4,
                "placeholder": "แสดงความคิดเห็นของคุณ...",
            }),
        }
        labels = {
            "author": "ชื่อของคุณ",
            "text": "ความคิดเห็น",
        }

    def clean_author(self):
        author = self.cleaned_data["author"]
        if len(author.strip()) < 2:
            raise forms.ValidationError("ชื่อสั้นเกินไป กรุณาระบุอย่างน้อย 2 ตัวอักษร")
        return author.strip()

    def clean_text(self):
        text = self.cleaned_data["text"]
        if len(text.strip()) < 10:
            raise forms.ValidationError(
                "ความคิดเห็นสั้นเกินไป กรุณาเขียนอย่างน้อย 10 ตัวอักษร"
            )
        lowered = text.lower()
        for word in BANNED_WORDS:
            if word in lowered:
                raise forms.ValidationError(f'ข้อความมีคำที่ไม่อนุญาต: "{word}"')
        return text.strip()

    def clean(self):
        cleaned_data = super().clean()
        if cleaned_data.get("honeypot"):
            raise forms.ValidationError("ตรวจพบความผิดปกติ กรุณาลองใหม่อีกครั้ง")
        return cleaned_data
```

และ View `post_detail_view` เวอร์ชันสุดท้ายที่สั้นลงจาก Part 025 อย่างชัดเจน
(เทียบเต็ม ๆ ได้กับขั้นตอนที่ 250.3 ของ Part 025):

```python
# blog/views.py
from django.shortcuts import render, redirect, get_object_or_404
from .models import Post
from .forms import CommentForm


def post_detail_view(request, slug):
    post = get_object_or_404(Post, slug=slug)
    comments = post.comments.all()

    if request.method == "POST":
        form = CommentForm(request.POST)
        if form.is_valid():
            comment = form.save(commit=False)
            comment.post = post
            comment.save()
            return redirect("blog:detail", slug=post.slug)
    else:
        form = CommentForm()

    return render(request, "blog/post_detail.html", {
        "post": post,
        "comments": comments,
        "form": form,
    })
```

| | Part 025 (plain `forms.Form`) | Part 026 (`ModelForm`) |
|---|---|---|
| จำนวนบรรทัดประกาศ field | 3 field ประกาศเอง (`author`, `text`, `honeypot`) | 1 field ประกาศเอง (`honeypot`) + 2 field มาจาก `Meta.fields` อัตโนมัติ |
| การสร้าง `Comment` ใน View | `Comment.objects.create(post=post, author=..., text=...)` (copy ทีละ field) | `form.save(commit=False)` + เติม `post` + `.save()` |
| ถ้า `Comment` เพิ่ม field ใหม่ | ต้องแก้ทั้ง Form และ View | แก้แค่ `Meta.fields` |
| Custom validation (`clean_author`, `clean_text`, `clean`) | ทำงานเหมือนกันทุกประการ | ทำงานเหมือนกันทุกประการ (ไม่ต้องแก้อะไรเลย) |

### 260.2 สรุปสิ่งที่ได้เรียนรู้ใน Part นี้

- ✅ เข้าใจว่า `ModelForm` แก้ปัญหาการประกาศ field ซ้ำซ้อนและการประกอบ object
  ด้วยมือที่เจอใน Part 025 ได้อย่างไร ผ่าน `class Meta` ที่มี `model` และ
  `fields`/`exclude`
- ✅ รู้ตารางการแปลง Model field type เป็น Form field + Widget อัตโนมัติ และ
  วิธี override widget/label/help_text/error_message ผ่าน `Meta`
- ✅ เข้าใจความเสี่ยงของ `fields = "__all__"` (Mass Assignment) และเหตุผลที่
  ควรใช้ `fields = [...]` แบบระบุชัดเจนเสมอ
- ✅ ใช้ `save(commit=False)` เพื่อเติมค่า field ที่ฟอร์มไม่รู้จักก่อนบันทึกจริง
  และรู้ว่าทำไมต้องเรียก `save_m2m()` เองเมื่อฟอร์มมี `ManyToManyField`
- ✅ Override `__init__` ของ `ModelForm` เพื่อจำกัด queryset ของ FK field
  ตาม user, ปรับ widget แบบ dynamic, และปิดการแก้ไข field ด้วย `disabled=True`
  อย่างปลอดภัยจริงฝั่ง server
- ✅ เข้าใจว่า `ModelForm.is_valid()` เรียก `instance.full_clean()` ให้
  อัตโนมัติผ่าน `_post_clean()` ทำให้ Model validator, `Model.clean()` และ
  `validate_unique()` ทำงานโดยไม่ต้องเขียนอะไรเพิ่มในฟอร์ม
- ✅ สร้าง Formset ได้ทั้ง 3 แบบ: `formset_factory` (ฟอร์มเปล่าหลายชุด),
  `modelformset_factory` (แก้ไขหลาย object พร้อมกัน), `inlineformset_factory`
  (จัดการ parent-child เช่น `Post` กับ `PostImage`)
- ✅ เข้าใจกลไกของ Management Form (`TOTAL_FORMS`, `INITIAL_FORMS`,
  `MIN_NUM_FORMS`, `MAX_NUM_FORMS`) และพารามิเตอร์ `extra`, `can_delete`,
  `max_num`, `min_num`
- ✅ เขียน `formset.clean()` ผ่าน custom `BaseInlineFormSet`/`BaseModelFormSet`
  เพื่อ validate เงื่อนไขที่ต้องมองข้ามทุกฟอร์มในชุดพร้อมกัน
- ✅ แปลง `CommentForm` จาก Part 025 เป็น `ModelForm` ตัวจริงได้สำเร็จ พร้อม
  คง custom validation เดิมไว้ครบถ้วน

### 260.3 Checklist ก่อนไป Part ถัดไป

- [ ] เขียน `ModelForm` พร้อม `class Meta` ที่มี `model` และ `fields` (ไม่ใช้
      `"__all__"`) ได้เอง
- [ ] อธิบายได้ว่า Widget แบบไหนถูกสร้างอัตโนมัติจาก Model field type ไหนบ้าง
- [ ] ใช้ `form.save(commit=False)` เติมค่า field ที่ฟอร์มไม่รู้จักก่อนบันทึก
      ได้ถูกต้อง และเรียก `save_m2m()` เมื่อมี M2M field
- [ ] Override `__init__` เพื่อจำกัด queryset ของ FK field ตาม user ที่ login
      ได้สำเร็จ (ทดสอบด้วย user คนละคนเห็นตัวเลือกต่างกันจริง)
- [ ] อธิบายลำดับการทำงานของ `ModelForm.is_valid()` ได้ครบทั้ง field-level,
      `clean_<field>()`, `clean()`, และ `_post_clean()` (Model validator)
- [ ] สร้าง Formset ทั้ง 3 แบบได้เอง และรู้ว่าแบบไหนควรใช้ในสถานการณ์ไหน
- [ ] เข้าใจว่าทำไมต้องมี `{{ formset.management_form }}` เสมอ และแก้ error
      "ManagementForm data is missing" ได้ถ้าเจอ
- [ ] เขียน `formset.clean()` เพื่อ validate เงื่อนไขข้ามฟอร์ม (เช่น ต้องมี
      รูปหลักอย่างน้อย 1 รูป) ได้สำเร็จ
- [ ] แปลง `CommentForm` จาก Part 025 เป็น `ModelForm` และทดสอบผ่านเบราว์เซอร์
      จริงว่ายังทำงานถูกต้องเหมือนเดิมทุกประการ

### 260.4 แบบฝึกหัดท้ายบท

**แบบฝึกหัดที่ 1**: แปลง `PostForm` ในบทนี้ให้เป็น `ModelForm` ที่สมบูรณ์
พร้อม override `__init__` ให้จำกัด queryset ของ `category` ตาม user (ตามที่
สาธิตในขั้นตอนที่ 253) แล้วเขียน View `post_create_view` และ `post_edit_view`
ที่ใช้งานฟอร์มนี้ได้จริงทั้งสองกรณี (สร้างใหม่และแก้ไขของเดิม) ทดสอบด้วยการ
login ด้วย user 2 คนที่มีหมวดหมู่ต่างกัน แล้วยืนยันว่าแต่ละคนเห็นตัวเลือก
`category` ไม่เหมือนกันจริง

**แบบฝึกหัดที่ 2**: สร้าง `modelformset_factory` สำหรับ `Comment` ที่อนุญาต
ให้ staff แก้ไข `text` ของคอมเมนต์หลายรายการพร้อมกันในหน้าเดียว (ใช้
`queryset=Comment.objects.filter(post=post)`, `extra=0`, `can_delete=True`)
แล้วเขียน View และ Template ที่แสดงคอมเมนต์ทั้งหมดของโพสต์หนึ่งโพสต์ พร้อม
ปุ่ม "บันทึกการแก้ไขทั้งหมด" ที่เรียก `formset.save()`

**แบบฝึกหัดที่ 3**: สร้าง `inlineformset_factory` สำหรับ `PostImage` แบบเต็ม
ตามที่สาธิตในขั้นตอนที่ 257-259 (รวม custom `BasePostImageFormSet.clean()`
ที่บังคับว่าต้องมีรูปหลักพอดี 1 รูป) แล้วนำไปประกอบกับ `PostForm` ในหน้าเดียว
ทั้งกรณีสร้างโพสต์ใหม่ (257.3) และแก้ไขโพสต์เดิม (257.2) ทดสอบให้ครบทุกกรณี:
เพิ่มรูปใหม่, ลบรูปเดิมด้วย checkbox `DELETE`, และลองไม่เลือกรูปหลักเลยเพื่อ
ดูว่า error message ที่เขียนไว้ปรากฏถูกต้อง

**แบบฝึกหัดที่ 4 (ขั้นสูง)**: เพิ่มปุ่ม "เพิ่มรูปภาพอีกแถว" แบบ JavaScript
ตามขั้นตอนที่ 258.5 เข้าไปในหน้าฟอร์มของแบบฝึกหัดที่ 3 ให้ทำงานได้จริงโดยไม่
ต้อง reload หน้าเว็บ จากนั้นเพิ่มปุ่ม "ลบแถวนี้ก่อน submit" สำหรับฟอร์ม extra
ที่ยังไม่มี `instance.pk` (ใช้ JavaScript ลบ DOM element ออกไปเลย พร้อมลดค่า
`TOTAL_FORMS` ลง 1 — ต่างจากฟอร์มที่มี `instance.pk` อยู่แล้วซึ่งต้องใช้
checkbox `DELETE` แทนการลบ DOM ทิ้งตรง ๆ เพราะต้องส่งคำสั่งลบไปถึงฐานข้อมูล
ด้วย ลองอธิบายด้วยคำพูดตัวเองว่าทำไมสองกรณีนี้ต้องจัดการต่างกัน)

### 260.5 คำถามที่พบบ่อย (FAQ)

**Q: ควรใช้ `ModelForm` เสมอไป หรือยังมีกรณีที่ต้องใช้ plain `forms.Form`
อยู่บ้าง?**
A: ใช้ `ModelForm` เมื่อฟอร์มมีเป้าหมายชัดเจนคือสร้าง/แก้ไข object ในฐานข้อมูล
โดยตรง (กรณีส่วนใหญ่ของฟอร์มในเว็บแอปพลิเคชันจริง) แต่ยังต้องใช้ plain
`forms.Form` เมื่อฟอร์มไม่ได้ผูกกับ Model ใดโดยตรง เช่น ฟอร์มค้นหา/กรองข้อมูล
(`SearchForm` จาก Part 025), ฟอร์ม login, ฟอร์มติดต่อที่แค่ส่งอีเมลไม่ได้
บันทึกลงฐานข้อมูล หรือฟอร์มที่รวมข้อมูลจากหลาย Model เข้าด้วยกันในฟอร์มเดียว
ซึ่งไม่มี Model ตัวใดตัวหนึ่งที่ตรงกับโครงสร้างฟอร์มพอดี

**Q: ทำไม field ที่มี `auto_now_add=True` หรือ `editable=False` ไม่ปรากฏใน
`ModelForm` แม้จะใส่ชื่อไว้ใน `Meta.fields`?**
A: Django ถือว่า field ประเภทนี้ "ระบบจัดการเองอัตโนมัติ ผู้ใช้ไม่ควรกรอกเอง
ได้" (เช่น `created_at` ถูกตั้งค่าจากเวลาปัจจุบันเสมอ ไม่ควรให้ผู้ใช้พิมพ์
วันที่เองได้) จึงถูกข้ามออกจากฟอร์มโดยอัตโนมัติเป็นค่าพื้นฐาน (default) ของ
Django เอง โดยไม่ต้อง exclude ด้วยตัวเอง

**Q: `form.save()` กับ `formset.save()` คืน object แบบไหนกลับมา ต่างกันไหม?**
A: `form.save()` ของ `ModelForm` เดี่ยว ๆ คืน **instance เดียว** ที่ถูกบันทึก
แล้ว ส่วน `formset.save()` คืน **list ของ instance ทุกตัวที่ถูกสร้างหรือ
อัปเดต** (ไม่รวม instance ที่ถูกลบทิ้งผ่าน `DELETE` checkbox และไม่รวมฟอร์ม
ที่ไม่มีการเปลี่ยนแปลงข้อมูลเลย) ถ้าต้องการรวมข้อมูลของ object ที่ถูกลบด้วย
สามารถเข้าถึงผ่าน `formset.deleted_objects` ได้แยกต่างหากหลังเรียก `.save()`
แล้ว

**Q: ถ้าอยากให้ `ModelForm` validate ผ่านโดยไม่ต้องพึ่ง Model validator เลย
(เช่น กรณี import ข้อมูลจำนวนมากที่ไม่อยากให้ validate เข้มงวด) ทำได้ไหม?**
A: ได้ โดยส่ง `Meta.exclude` field นั้นออกจากฟอร์ม (validator ของ field ที่
ไม่อยู่ใน `Meta.fields` จะไม่ถูกตรวจตามที่อธิบายในขั้นตอนที่ 254.6) หรือถ้า
ต้องการข้าม validation ทั้งหมดจริง ๆ ในสถานการณ์พิเศษ (เช่น script import
ข้อมูล) แนะนำให้ใช้ `Model.objects.create()`/`bulk_create()` ตรง ๆ แทนการผ่าน
`ModelForm` เลย เพราะ `ModelForm` ถูกออกแบบมาสำหรับ "รับข้อมูลจากผู้ใช้ผ่าน
เว็บ" ซึ่งควร validate เข้มงวดเป็นค่าเริ่มต้นเสมอ

---

## เตรียมตัวสำหรับ Part ถัดไป

**Part 027: Form Validation ขั้นสูงและ Custom Widgets** จะพาคุณเจาะลึกระบบ
Validator ของ Django อย่างเต็มรูปแบบ ทั้ง built-in validator ที่หลากหลายกว่า
`MinLengthValidator` ที่เห็นใน Part นี้มาก (`RegexValidator`, `EmailValidator`,
`URLValidator` ฯลฯ) การเขียน custom validator function ของตัวเอง (เช่น
validate เบอร์โทรไทย), การสร้าง **Custom Widget** ตั้งแต่ระดับง่ายไปจนถึง
widget ที่ต้องแนบ CSS/JS ของตัวเองผ่าน `Media` class, และการผสาน Custom
Widget เข้ากับ Formset ที่เพิ่งเรียนใน Part นี้ เตรียม `PostForm` และ
`PostImageFormSet` ที่สร้างไว้ให้พร้อม เพราะเราจะนำมาต่อยอดใส่ Custom Widget
และปรับ validation ให้รัดกุมยิ่งขึ้นไปอีกขั้นในบทถัดไป
