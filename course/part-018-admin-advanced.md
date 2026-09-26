# Part 018: Django Admin ขั้นสูง: Customization และ Actions

> **ขั้นตอนที่ 171-180 ของหลักสูตร** | Phase 2: Models, ORM และ Admin
>
> เป้าหมายของ Part นี้: ต่อยอดจาก Django Admin เบื้องต้นใน Part 017 (ที่คุณลงทะเบียน
> `Post`, `Category`, `Tag`, `Comment` ด้วย `ModelAdmin` พร้อม `list_display` และ
> `inlines` แล้ว) ไปสู่การ **ปรับแต่ง Admin ระดับมืออาชีพ**: เขียน custom action
> ของตัวเอง, override template, เปลี่ยนแบรนด์, ควบคุม permission ตามตรรกะธุรกิจ,
> เพิ่มหน้าใหม่เข้าไปใน Admin เอง, รู้จัก theme ของบุคคลที่สาม, ทำความเข้าใจแนวทาง
> รักษาความปลอดภัย และอ่านประวัติการแก้ไขผ่าน `LogEntry` เมื่อจบ Part นี้ Admin ของ
> ระบบบล็อกจะกลายเป็นเครื่องมือจัดการเนื้อหาที่ทีมงานใช้งานได้จริงในระดับ production

---

## สารบัญของ Part นี้

- ขั้นตอนที่ 171: Custom Admin Actions — เขียน bulk action เอง
- ขั้นตอนที่ 172: Overriding Admin Templates — เจาะลึก template override hierarchy
- ขั้นตอนที่ 173: Customizing Admin Site Branding — `site_header`, `AdminSite` ของตัวเอง
- ขั้นตอนที่ 174: Permissions ใน Admin ตามตรรกะธุรกิจ
- ขั้นตอนที่ 175: `@admin.register` เทียบกับ `admin.site.register()`
- ขั้นตอนที่ 176: เพิ่ม Custom View เข้า Admin เอง (`get_urls()`)
- ขั้นตอนที่ 177: ภาพรวม Django Admin Theme ของบุคคลที่สาม
- ขั้นตอนที่ 178: Django Admin Security Best Practices
- ขั้นตอนที่ 179: Admin Log — `LogEntry` ดูประวัติการแก้ไข
- ขั้นตอนที่ 180: สรุปและแบบฝึกหัด — ประกอบร่าง Admin บล็อกฉบับสมบูรณ์

---

## ขั้นตอนที่ 171: Custom Admin Actions — เขียน bulk action เอง

### 171.1 Admin Actions คืออะไร

**Admin Action** คือฟังก์ชันที่ทำงานกับ **queryset ของแถวที่ถูกเลือก** ในหน้า
change list ของ Admin คุณคงเคยเห็น dropdown "Action" มุมบนซ้ายของตารางที่มีตัวเลือก
default ชื่อ "Delete selected ..." มาให้แล้ว นั่นคือ built-in action ที่ Django ให้มา
Django อนุญาตให้เราเพิ่ม action ของตัวเองเข้าไปในลิสต์นั้นได้ไม่จำกัดจำนวน

Action ทุกตัวมี signature เหมือนกันหมด:

```python
def my_action(modeladmin, request, queryset):
    ...
```

- `modeladmin`: instance ของ `ModelAdmin` ที่ action นี้ผูกอยู่ (เข้าถึง `self` ไม่ได้
  เพราะ action ไม่ใช่ method ของ class โดยตรง แต่รับ `modeladmin` เข้ามาแทน)
- `request`: `HttpRequest` object ของ request ปัจจุบัน (ใช้เช็ค `request.user`,
  ส่ง message กลับไปได้)
- `queryset`: QuerySet ของแถวทั้งหมดที่ผู้ใช้ติ๊กเลือกไว้ในตาราง

### 171.2 เขียน Action แรก: "Mark selected posts as published"

สมมติว่าไฟล์ `blog/admin.py` ของคุณจาก Part 017 มีหน้าตาประมาณนี้อยู่แล้ว:

```python
# blog/admin.py (จาก Part 017)
from django.contrib import admin
from .models import Post, Category, Tag, Comment


class CommentInline(admin.TabularInline):
    model = Comment
    extra = 0
    fields = ["author", "text", "parent", "created_at"]
    readonly_fields = ["created_at"]


class PostAdmin(admin.ModelAdmin):
    list_display = ["title", "category", "is_published", "view_count", "created_at"]
    list_filter = ["is_published", "category", "created_at"]
    search_fields = ["title", "content"]
    prepopulated_fields = {"slug": ("title",)}
    inlines = [CommentInline]


admin.site.register(Post, PostAdmin)
admin.site.register(Category)
admin.site.register(Tag)
admin.site.register(Comment)
```

เราจะเพิ่ม action ที่เลือกโพสต์หลายแถวพร้อมกัน แล้วกด "เผยแพร่ทั้งหมด" ในคลิกเดียว
โดยใช้ **decorator `@admin.action`** (เพิ่มมาตั้งแต่ Django 3.2) ซึ่งเป็นวิธีมาตรฐาน
ในปัจจุบัน:

```python
# blog/admin.py
from django.contrib import admin
from django.contrib import messages
from django.utils.translation import ngettext
from .models import Post, Category, Tag, Comment


@admin.action(description="✅ Mark selected posts as published")
def mark_as_published(modeladmin, request, queryset):
    # update() คือ bulk update ระดับ SQL เดียว เร็วกว่า loop เยอะมาก (ดู Part 013)
    updated_count = queryset.update(is_published=True)
    modeladmin.message_user(
        request,
        ngettext(
            "%d โพสต์ถูกเผยแพร่แล้ว",
            "%d โพสต์ถูกเผยแพร่แล้ว",
            updated_count,
        )
        % updated_count,
        messages.SUCCESS,
    )


@admin.action(description="🚫 Mark selected posts as draft")
def mark_as_draft(modeladmin, request, queryset):
    updated_count = queryset.update(is_published=False)
    modeladmin.message_user(
        request,
        f"{updated_count} โพสต์ถูกเปลี่ยนเป็นฉบับร่างแล้ว",
        messages.WARNING,
    )


class PostAdmin(admin.ModelAdmin):
    list_display = ["title", "category", "is_published", "view_count", "created_at"]
    list_filter = ["is_published", "category", "created_at"]
    search_fields = ["title", "content"]
    prepopulated_fields = {"slug": ("title",)}
    actions = [mark_as_published, mark_as_draft]
```

**สิ่งที่เกิดขึ้น**:

1. `@admin.action(description=...)` กำหนดข้อความที่จะโชว์ใน dropdown ของ action
2. `queryset.update(...)` คือการอัปเดตแบบ bulk ในคำสั่ง SQL เดียว **ไม่เรียก**
   `save()` ของแต่ละ instance และ **ไม่ยิง signal** `pre_save`/`post_save`
   (สำคัญมาก จะกล่าวถึงอีกครั้งใน Part 019 เรื่อง Signals)
3. `modeladmin.message_user()` คือวิธีมาตรฐานในการส่งข้อความแจ้งเตือนกลับไปแสดงที่
   ด้านบนของหน้า Admin หลัง action ทำงานเสร็จ (ใช้ระบบ `django.contrib.messages`
   framework เดียวกับที่ใช้ในหน้าเว็บทั่วไป)

### 171.3 Action แบบ Method ของ Class (ทางเลือกเดิมที่ยังใช้ได้)

ก่อน Django 3.2 เราต้องเขียน action เป็น **method** ของ `ModelAdmin` แทน แล้วตั้ง
attribute `short_description` เอง วิธีนี้ยังใช้ได้ และบางทีมยังนิยมเพราะเข้าถึง
`self` (นั่นคือ modeladmin) ได้ตรง ๆ:

```python
class PostAdmin(admin.ModelAdmin):
    actions = ["mark_as_published_method"]

    @admin.action(description="✅ Mark selected posts as published (method style)")
    def mark_as_published_method(self, request, queryset):
        updated_count = queryset.update(is_published=True)
        self.message_user(request, f"{updated_count} โพสต์ถูกเผยแพร่แล้ว")
```

| รูปแบบ | ข้อดี | ข้อเสีย |
|---|---|---|
| ฟังก์ชันอิสระ + `@admin.action` | นำไป reuse กับหลาย `ModelAdmin` ได้ | ต้อง import `modeladmin` เข้ามาเป็น parameter |
| Method ของ class | เข้าถึง `self.model`, `self.opts` ได้สะดวก | ผูกกับ class เดียว reuse ยากกว่า |

### 171.4 Action ที่ซับซ้อนขึ้น: Export เป็นไฟล์ CSV

Action ที่ทรงพลังมากในงานจริงคือการ **export ข้อมูลที่เลือกออกมาเป็นไฟล์** เพื่อให้
ทีมการตลาดหรือทีมข้อมูลเอาไปเปิดใน Excel ได้ทันที:

```python
import csv
from django.http import HttpResponse
from django.contrib import admin


@admin.action(description="⬇️ Export selected posts as CSV")
def export_posts_as_csv(modeladmin, request, queryset):
    # ตั้งค่า HttpResponse ให้เบราว์เซอร์รู้ว่านี่คือไฟล์ให้ดาวน์โหลด ไม่ใช่หน้าเว็บ
    response = HttpResponse(content_type="text/csv")
    response["Content-Disposition"] = 'attachment; filename="posts_export.csv"'

    writer = csv.writer(response)
    # เขียนหัวตาราง (header row)
    writer.writerow(["ID", "Title", "Category", "Published", "Views", "Created At"])

    # เขียนแต่ละแถวจาก queryset ที่ผู้ใช้เลือก
    for post in queryset.select_related("category"):
        writer.writerow([
            post.id,
            post.title,
            post.category.name if post.category else "-",
            "Yes" if post.is_published else "No",
            post.view_count,
            post.created_at.strftime("%Y-%m-%d %H:%M"),
        ])

    return response  # คืนค่าเป็น HttpResponse ตรง ๆ = เบราว์เซอร์เริ่มดาวน์โหลดไฟล์ทันที


class PostAdmin(admin.ModelAdmin):
    actions = [mark_as_published, mark_as_draft, export_posts_as_csv]
    list_select_related = ["category"]  # ลด query ซ้ำตอนแสดง list_display (Part 013)
```

**ข้อสังเกตสำคัญ**: action ปกติไม่ต้อง `return` อะไร (Django จะ redirect กลับไปหน้า
change list ให้เอง) แต่ถ้า action `return HttpResponse(...)` แบบนี้ Django จะส่ง
response นั้นกลับไปยังผู้ใช้แทนการ redirect — นี่คือกลไกเดียวกับการทำให้ action
กลายเป็น "ปุ่มดาวน์โหลดไฟล์"

### 171.5 Action ที่มีหน้ายืนยันก่อน (Intermediate Confirmation Page)

บาง action อันตรายเกินกว่าจะให้กดครั้งเดียวจบ เช่น "ลบโพสต์และคอมเมนต์ทั้งหมดถาวร"
เราสามารถให้ action redirect ไปหน้ายืนยันก่อนได้ โดยใช้เทคนิค "ส่ง ID ที่เลือกผ่าน
querystring แล้ว redirect ไปหน้า custom view":

```python
from django.shortcuts import render, redirect
from django.urls import reverse


@admin.action(description="🗑️ Permanently delete posts and all comments")
def delete_with_confirmation(modeladmin, request, queryset):
    selected_ids = list(queryset.values_list("pk", flat=True))
    request.session["posts_to_delete"] = selected_ids
    return redirect(reverse("admin:blog_post_confirm_delete"))
```

เราจะเห็นวิธีเพิ่ม URL `admin:blog_post_confirm_delete` เข้าไปจริง ๆ ใน **ขั้นตอนที่
176** เมื่อเรียนเรื่อง `get_urls()` เพราะต้องใช้เทคนิคเดียวกัน

### 171.6 จำกัดสิทธิ์การเห็น Action ด้วย `get_actions()`

ไม่ใช่ทุก action ควรให้ทุก staff user เห็น เราสามารถ override `get_actions()` เพื่อ
ซ่อน action บางตัวตามสิทธิ์ของผู้ใช้:

```python
class PostAdmin(admin.ModelAdmin):
    actions = [mark_as_published, mark_as_draft, export_posts_as_csv]

    def get_actions(self, request):
        actions = super().get_actions(request)
        # เฉพาะ superuser เท่านั้นที่เห็นปุ่ม export (ข้อมูลอาจมี PII)
        if not request.user.is_superuser and "export_posts_as_csv" in actions:
            del actions["export_posts_as_csv"]
        return actions
```

### 171.7 Action ระดับ Global (ผูกกับ AdminSite แทนที่จะผูกกับ Model เดียว)

ถ้าต้องการ action ที่ใช้ได้กับ **ทุก ModelAdmin** ในระบบ (เช่น export เป็น CSV แบบ
generic) สามารถเพิ่มเข้าไปที่ `admin.site.add_action()`:

```python
# blog/admin.py หรือไฟล์ admin กลาง เช่น config/admin_actions.py
from django.contrib import admin


@admin.action(description="⬇️ Export selected as CSV (generic)")
def generic_export_csv(modeladmin, request, queryset):
    import csv
    from django.http import HttpResponse

    meta = modeladmin.model._meta
    response = HttpResponse(content_type="text/csv")
    response["Content-Disposition"] = f'attachment; filename="{meta.model_name}.csv"'
    writer = csv.writer(response)
    field_names = [f.name for f in meta.fields]
    writer.writerow(field_names)
    for obj in queryset:
        writer.writerow([getattr(obj, field) for field in field_names])
    return response


admin.site.add_action(generic_export_csv)  # จะปรากฏใน "ทุก" ModelAdmin ที่ลงทะเบียนไว้
```

Action ระดับ global สะดวกมากสำหรับงาน export ทั่วไป แต่ต้องระวังว่ามันจะโผล่ในทุก
model จริง ๆ รวมถึง model ที่อาจไม่เหมาะจะ export ตรง ๆ (เช่นมี field รูปภาพ)

---

## ขั้นตอนที่ 172: Overriding Admin Templates

### 172.1 ทำไมต้อง Override Template ของ Admin

Django Admin สร้างมาจากชุด template ธรรมดา ๆ ที่อยู่ใน
`django/contrib/admin/templates/admin/` ภายใน Django package เอง เราสามารถ
"แทนที่" (override) template เหล่านั้นด้วยไฟล์ของเราเองได้ โดยไม่ต้องแก้โค้ดของ
Django เลย นี่คือแนวทางที่ถูกต้องเมื่อต้องการ:

- เพิ่มปุ่มหรือข้อความพิเศษในหน้าแก้ไข (change form)
- ปรับ layout ของหน้า list บาง model
- ใส่ branding, banner แจ้งเตือน, หรือ widget พิเศษ

### 172.2 Template Override Hierarchy (ลำดับความสำคัญ)

Django ค้นหา template จากหลาย location ตามลำดับ **จากเจาะจงที่สุด → ทั่วไปที่สุด**
สำหรับ change form ของ model หนึ่ง ๆ ลำดับคือ:

```
1. templates/admin/<app_label>/<model_name>/change_form.html   ← เจาะจงที่สุด
2. templates/admin/<app_label>/change_form.html
3. templates/admin/change_form.html
4. django/contrib/admin/templates/admin/change_form.html        ← default ของ Django
```

Django จะใช้ไฟล์แรกที่หาเจอตามลำดับนี้ (ไล่จากบนลงล่าง) ตารางสรุป template สำคัญ
ที่มักถูก override:

| Template | ใช้ตอนไหน |
|---|---|
| `admin/base_site.html` | โครง layout หลักของทุกหน้า Admin (ใช้เปลี่ยน header/logo) |
| `admin/index.html` | หน้าแรกของ Admin (รายการ app/model ทั้งหมด) |
| `admin/app_index.html` | หน้ารวม model ของ app หนึ่ง |
| `admin/<app>/<model>/change_list.html` | หน้าตารางแสดงรายการของ model นั้น |
| `admin/<app>/<model>/change_form.html` | หน้าเพิ่ม/แก้ไข object ของ model นั้น |
| `admin/<app>/<model>/object_history.html` | หน้าประวัติการแก้ไข object |
| `admin/login.html` | หน้า login ของ Admin |
| `admin/delete_confirmation.html` | หน้ายืนยันก่อนลบ |

### 172.3 ตั้งค่า `TEMPLATES` ให้หา Template ของเราก่อน

ตรวจสอบใน `config/settings/base.py` (จาก Part 010) ว่า `APP_DIRS` เป็น `True`
และมี `DIRS` ชี้ไปที่โฟลเดอร์ `templates/` ระดับโปรเจกต์:

```python
# config/settings/base.py
TEMPLATES = [
    {
        "BACKEND": "django.template.backends.django.DjangoTemplates",
        "DIRS": [BASE_DIR / "templates"],   # ← โฟลเดอร์ templates ระดับโปรเจกต์
        "APP_DIRS": True,                    # ← ค้นหาใน <app>/templates/ ด้วย
        "OPTIONS": {
            "context_processors": [
                "django.template.context_processors.debug",
                "django.template.context_processors.request",
                "django.contrib.auth.context_processors.auth",
                "django.contrib.messages.context_processors.messages",
            ],
        },
    },
]
```

จากนั้นสร้างโครงสร้างโฟลเดอร์ตาม hierarchy ด้านบน:

```
templates/
└── admin/
    └── blog/
        └── post/
            └── change_form.html
```

### 172.4 ตัวอย่างจริง: เพิ่มกล่องข้อมูล "สถิติของโพสต์" ในหน้าแก้ไข

Django ให้เทคนิค `{% extends %}` + `{% block %}` เพื่อ "สืบทอด" template เดิมของ
Django แล้วแทรกเนื้อหาเพิ่มเข้าไปเฉพาะจุด โดยไม่ต้อง copy ทั้งไฟล์:

```html
{# templates/admin/blog/post/change_form.html #}
{% extends "admin/change_form.html" %}
{% load i18n %}

{% block field_sets %}
    {% if original %}
    <div class="module aligned" style="margin-bottom: 20px; padding: 12px 16px;
                background: #f0f7ff; border-left: 4px solid #417690;">
        <h2 style="margin-top: 0;">📊 สถิติของโพสต์นี้</h2>
        <p><strong>จำนวนคอมเมนต์ทั้งหมด:</strong> {{ original.comments.count }}</p>
        <p><strong>จำนวนการเข้าชม:</strong> {{ original.view_count }} ครั้ง</p>
        <p><strong>สถานะ:</strong>
            {% if original.is_published %}
                <span style="color: green;">เผยแพร่แล้ว</span>
            {% else %}
                <span style="color: orange;">ฉบับร่าง</span>
            {% endif %}
        </p>
    </div>
    {% endif %}

    {{ block.super }}
{% endblock %}
```

**อธิบายทีละบรรทัด**:

- `{% extends "admin/change_form.html" %}`: สืบทอด template ต้นฉบับของ Django
  (Django จะไปหาไฟล์นี้ตาม hierarchy เดิม แต่ **ไม่รวมไฟล์ที่เรากำลังเขียนอยู่**
  ป้องกัน infinite loop)
- `{% block field_sets %}`: `change_form.html` เดิมของ Django แบ่งเป็นหลาย block
  (`object-tools`, `field_sets`, `after_field_sets`, ฯลฯ) เราเลือก override
  เฉพาะ block ที่ต้องการ
- `{{ original }}`: ตัวแปร context ที่ Django ส่งมาให้เสมอในหน้า change form
  หมายถึง object เดิมที่กำลังแก้ไข (เป็น `None` ถ้าเป็นหน้า "เพิ่มใหม่")
- `{{ block.super }}`: เรียก **เนื้อหาเดิม** ของ block นั้นจาก parent template
  (สำคัญมาก! ถ้าลืมใส่ ฟอร์มแก้ไขทั้งหมดของ Django จะหายไป)

### 172.5 ตัวอย่าง: เพิ่มปุ่มใน Change List ด้วย `change_list.html`

```html
{# templates/admin/blog/post/change_list.html #}
{% extends "admin/change_list.html" %}

{% block object-tools-items %}
    <li>
        <a href="{% url 'admin:blog_dashboard' %}" class="button">
            📈 ดู Dashboard สถิติ
        </a>
    </li>
    {{ block.super }}
{% endblock %}
```

ลิงก์ `admin:blog_dashboard` นี้ยังไม่มีจริงในตอนนี้ — เราจะสร้าง URL name นี้ขึ้นมา
จริงในขั้นตอนที่ 176 ด้วย `get_urls()`

### 172.6 ค้นหาว่า Template ต้นฉบับของ Django มี Block อะไรบ้าง

วิธีที่เร็วที่สุดคือเปิดไฟล์ต้นฉบับใน virtual environment ของคุณโดยตรง:

```bash
python -c "import django, os; print(os.path.join(os.path.dirname(django.__file__), 'contrib/admin/templates/admin'))"
```

คำสั่งนี้จะพิมพ์ path เต็มไปยังโฟลเดอร์ template ต้นฉบับ เปิดไฟล์ `change_form.html`
ในนั้นดูได้เลยว่ามี `{% block %}` ชื่ออะไรบ้างที่ override ได้ — เป็นนิสัยที่มืออาชีพ
ทำเป็นประจำเวลาต้องปรับแต่ง Admin แบบเจาะลึก

### 172.7 คำเตือนสำคัญเรื่อง Version Upgrade

Template ภายในของ Django **ไม่ได้อยู่ใน public API ที่รับประกันความเข้ากันได้**
เวลาอัปเกรด Django เป็นเวอร์ชันใหม่ ควรตรวจสอบ release notes เสมอว่า template ที่
คุณ override ไว้มีการเปลี่ยน block name หรือโครงสร้างหรือไม่ (มักเกิดตอนอัปเกรด
major version เช่น 4.x → 5.x)

---

## ขั้นตอนที่ 173: Customizing Admin Site Branding

### 173.1 เปลี่ยนแบรนด์แบบง่ายที่สุด: `site_header`, `site_title`, `index_title`

วิธีที่เร็วและง่ายที่สุดคือตั้งค่า attribute 3 ตัวนี้บน `admin.site` (default
`AdminSite` instance ที่ Django สร้างให้อัตโนมัติ) ใน `blog/admin.py` หรือไฟล์
`apps.py` ของ app หลัก:

```python
# blog/admin.py
from django.contrib import admin

admin.site.site_header = "ระบบจัดการบล็อก MyCompany"      # หัวข้อบนสุดของทุกหน้า
admin.site.site_title = "MyCompany Admin"                   # <title> ของ browser tab
admin.site.index_title = "แผงควบคุมผู้ดูแลระบบ"              # หัวข้อของหน้าแรก
```

| Attribute | ตำแหน่งที่แสดงผล |
|---|---|
| `site_header` | แถบบนสุดของทุกหน้า Admin (`<h1 id="site-name">`) |
| `site_title` | ป้ายชื่อบน browser tab (`<title>`) |
| `index_title` | หัวข้อของหน้า index (`admin/index.html`) |

### 173.2 สร้าง `AdminSite` Subclass ของตัวเอง

การตั้งค่าข้างต้นแก้ที่ instance เดียว (`admin.site`) แต่ในระบบที่ซับซ้อนขึ้น เช่น
ต้องการมี **Admin หลายชุด** (เช่น Admin สำหรับทีมเนื้อหา กับ Admin สำหรับทีมการเงิน
ที่แยกสิทธิ์กันเด็ดขาด) เราควรสร้าง `AdminSite` subclass ของตัวเอง:

```python
# config/admin.py
from django.contrib.admin import AdminSite
from django.utils.translation import gettext_lazy as _


class BlogAdminSite(AdminSite):
    site_header = "ระบบจัดการบล็อก MyCompany"
    site_title = "MyCompany Admin"
    index_title = "แผงควบคุมผู้ดูแลระบบ"
    site_url = "/"                 # ลิงก์ "View site" มุมบนขวา ชี้ไปหน้าแรกของเว็บจริง
    empty_value_display = "-ไม่มีข้อมูล-"   # ค่า default เวลา field เป็น None/blank ทั้งระบบ

    def has_permission(self, request):
        # กำหนดว่าใครเข้า Admin ชุดนี้ได้บ้าง (ค่า default คือ is_active and is_staff)
        return request.user.is_active and request.user.is_staff


blog_admin_site = BlogAdminSite(name="blog_admin")
```

จากนั้นลงทะเบียน model เข้ากับ site ของเราแทน `admin.site` เดิม:

```python
# blog/admin.py
from config.admin import blog_admin_site
from .models import Post, Category, Tag, Comment
from django.contrib import admin


class PostAdmin(admin.ModelAdmin):
    list_display = ["title", "category", "is_published"]


blog_admin_site.register(Post, PostAdmin)
blog_admin_site.register(Category)
blog_admin_site.register(Tag)
blog_admin_site.register(Comment)
```

และเชื่อมเข้ากับ `urls.py` แทนที่ (หรือควบคู่กับ) `admin.site.urls` เดิม:

```python
# config/urls.py
from django.urls import path
from config.admin import blog_admin_site

urlpatterns = [
    path("admin/", blog_admin_site.urls),   # ใช้ site ของเราเองแทน admin.site
]
```

### 173.3 เมื่อไหร่ควรใช้ `AdminSite` Subclass เทียบกับตั้งค่า `admin.site` ตรง ๆ

| สถานการณ์ | แนวทางที่แนะนำ |
|---|---|
| ต้องการแค่เปลี่ยนโลโก้/ชื่อของ Admin เดียว | ตั้งค่า `admin.site.site_header` ตรง ๆ พอ |
| ต้องการ Admin แยก 2 ชุดที่มี URL, สิทธิ์, model ต่างกัน (เช่น `/admin/` กับ `/finance-admin/`) | สร้าง `AdminSite` subclass แยก instance |
| ต้องการ override พฤติกรรมทั่วทั้งระบบ เช่น `has_permission()`, `each_context()` | สร้าง `AdminSite` subclass |
| ทำ multi-tenant SaaS ที่แต่ละ tenant เห็น branding ต่างกัน | สร้าง `AdminSite` subclass แบบ dynamic ตาม request |

### 173.4 ปรับ Logo และ CSS ด้วยการ Override `base_site.html`

```html
{# templates/admin/base_site.html #}
{% extends "admin/base.html" %}

{% block branding %}
<h1 id="site-name">
    <a href="{% url 'admin:index' %}">
        <img src="{% static 'blog/img/logo-white.svg' %}" alt="MyCompany"
             style="height: 32px; vertical-align: middle; margin-right: 8px;">
        {{ site_header }}
    </a>
</h1>
{% endblock %}

{% block extrastyle %}
{{ block.super }}
<style>
    #header { background: #1a2b4c; }               /* เปลี่ยนสีแถบหัว */
    .module h2, .module caption { background: #2c3e6d; }
</style>
{% endblock %}
```

ต้องเพิ่ม `{% load static %}` ที่บรรทัดบนสุดถ้ายังไม่มี (โดยปกติ `base.html` ของ
Django โหลด `static` tag ไว้ให้แล้ว แต่ตรวจสอบเสมอเมื่อ `{% static %}` ใช้งานไม่ได้)

---

## ขั้นตอนที่ 174: Permissions ใน Admin ตามตรรกะธุรกิจ

### 174.1 Permission Hook ทั้ง 4 ตัวของ `ModelAdmin`

`ModelAdmin` มี method 4 ตัวที่ควบคุมว่าผู้ใช้แต่ละคน **เห็น/ทำอะไรได้บ้าง** กับ
model นั้น นอกเหนือจากระบบ permission มาตรฐาน (`app_label.add_post` เป็นต้น ที่จะ
เรียนละเอียดใน Part 033):

```python
class PostAdmin(admin.ModelAdmin):
    def has_view_permission(self, request, obj=None):
        """คืนค่า True/False ว่า user นี้ 'เห็น' object นี้ได้ไหม"""
        return super().has_view_permission(request, obj)

    def has_add_permission(self, request):
        """คืนค่า True/False ว่า user นี้ 'เพิ่ม' object ใหม่ได้ไหม (ไม่มี obj เพราะยังไม่มีของให้เช็ค)"""
        return super().has_add_permission(request)

    def has_change_permission(self, request, obj=None):
        """คืนค่า True/False ว่า user นี้ 'แก้ไข' object นี้ได้ไหม"""
        return super().has_change_permission(request, obj)

    def has_delete_permission(self, request, obj=None):
        """คืนค่า True/False ว่า user นี้ 'ลบ' object นี้ได้ไหม"""
        return super().has_delete_permission(request, obj)
```

**ข้อสังเกตสำคัญ**: `obj` เป็น `None` ได้เสมอ (เช่นตอนแสดง change list ที่ยังไม่รู้ว่า
กำลังจะทำอะไรกับ object ไหน) ดังนั้นโค้ดของเราต้อง **เช็ค `obj is None` ก่อนเสมอ**
ก่อนจะเข้าถึง attribute ของมัน ไม่งั้นจะพังตอน Django เรียก method นี้แบบไม่มี obj

### 174.2 Business Case: "ผู้เขียนแก้ไขได้เฉพาะโพสต์ของตัวเอง"

สมมติว่าระบบบล็อกของเรามี field `author` ผูกกับ `User` ใน model `Post` (เพิ่มจาก
Part 012) และมีกฎธุรกิจว่า:

- **Superuser**: แก้ไข/ลบโพสต์ของใครก็ได้
- **Staff ที่อยู่กลุ่ม "Editors"**: แก้ไขโพสต์ของใครก็ได้ แต่ **ลบไม่ได้**
- **Staff ทั่วไป (ผู้เขียน)**: แก้ไข/ลบได้เฉพาะโพสต์ที่ตัวเองเป็น `author`

```python
# blog/admin.py
from django.contrib import admin


class PostAdmin(admin.ModelAdmin):
    list_display = ["title", "author", "category", "is_published"]

    def get_queryset(self, request):
        """จำกัดว่า 'เห็น' แถวไหนบ้างในหน้า list ตั้งแต่แรก"""
        qs = super().get_queryset(request)
        if request.user.is_superuser or request.user.groups.filter(name="Editors").exists():
            return qs
        return qs.filter(author=request.user)

    def has_change_permission(self, request, obj=None):
        if obj is None:
            # obj เป็น None ตอนแสดง list → staff ทุกคนควรเข้าหน้า list ได้
            return request.user.is_staff
        if request.user.is_superuser or request.user.groups.filter(name="Editors").exists():
            return True
        return obj.author_id == request.user.id

    def has_delete_permission(self, request, obj=None):
        if request.user.groups.filter(name="Editors").exists() and not request.user.is_superuser:
            return False  # Editors แก้ไขได้แต่ลบไม่ได้ ตามกฎธุรกิจ
        if obj is None:
            return request.user.is_staff
        if request.user.is_superuser:
            return True
        return obj.author_id == request.user.id

    def save_model(self, request, obj, form, change):
        """เซ็ต author อัตโนมัติเป็นผู้ login ตอนสร้างโพสต์ใหม่"""
        if not change:  # change=False หมายถึงเป็นการ "เพิ่มใหม่" ไม่ใช่ "แก้ไข"
            obj.author = request.user
        super().save_model(request, obj, form, change)
```

**เชื่อมโยงกับสิ่งที่เรียนมาก่อน**: `get_queryset()` ควบคุม "เห็นแถวไหนในตาราง"
ส่วน `has_change_permission(obj=...)` ควบคุม "แก้ไขแถวนั้นได้ไหมเมื่อกดเข้าไปแล้ว"
— ทั้งสองต้องทำงานสอดคล้องกันเสมอ ไม่งั้นผู้ใช้อาจเห็นโพสต์ในตาราง (เพราะลืมกรองใน
`get_queryset`) แต่กดเข้าไปแก้ไม่ได้ (เพราะ `has_change_permission` บล็อกไว้ถูกต้อง)
ซึ่งสร้างประสบการณ์ที่สับสน จึงต้อง **ออกแบบทั้งสองจุดให้ตรงกันเสมอ**

### 174.3 ซ่อน Field บางตัวตามสิทธิ์ด้วย `get_fields()` / `get_readonly_fields()`

```python
class PostAdmin(admin.ModelAdmin):
    def get_readonly_fields(self, request, obj=None):
        if request.user.is_superuser:
            return []
        # staff ทั่วไปแก้ view_count เองไม่ได้ (ควรมาจาก logic ของระบบเท่านั้น)
        return ["view_count", "author"]
```

### 174.4 ตารางสรุป Permission Hooks ทั้งหมด

| Method | ควบคุมอะไร | `obj` เป็น `None` ได้ไหม |
|---|---|---|
| `has_view_permission` | เห็น object ในหน้า list/detail | ได้ (ตอนเช็คสิทธิ์ระดับ model) |
| `has_add_permission` | เพิ่ม object ใหม่ | ไม่มี parameter `obj` เลย |
| `has_change_permission` | แก้ไข object ที่มีอยู่ | ได้ |
| `has_delete_permission` | ลบ object | ได้ |
| `has_module_permission` | เห็น app นี้ในหน้า index เลยหรือไม่ | ไม่มี parameter `obj` |
| `get_queryset` | กรองว่าแถวไหนโผล่ในตารางตั้งแต่ต้น | (ไม่ใช่ permission hook โดยตรง แต่ทำงานคู่กันเสมอ) |

---

## ขั้นตอนที่ 175: `@admin.register` เทียบกับ `admin.site.register()`

### 175.1 สองวิธีที่ทำสิ่งเดียวกัน

Django มี 2 วิธีมาตรฐานในการลงทะเบียน `ModelAdmin` เข้ากับ Admin site:

```python
# วิธีที่ 1: admin.site.register() แบบดั้งเดิม
class PostAdmin(admin.ModelAdmin):
    list_display = ["title", "is_published"]

admin.site.register(Post, PostAdmin)


# วิธีที่ 2: @admin.register decorator (แนะนำในปัจจุบัน)
@admin.register(Post)
class PostAdmin(admin.ModelAdmin):
    list_display = ["title", "is_published"]
```

ทั้งสองแบบทำงาน **เหมือนกันทุกประการ** — `@admin.register` เป็นเพียง syntactic
sugar ที่เรียก `admin.site.register()` ให้อัตโนมัติหลัง class ถูกประกาศเสร็จ

### 175.2 ตารางเปรียบเทียบ

| ประเด็น | `admin.site.register()` | `@admin.register` |
|---|---|---|
| จำนวนบรรทัด | ต้องเขียนแยก 2 บรรทัด | รวมเป็นบรรทัดเดียวเหนือ class |
| ลงทะเบียนหลาย model ด้วย class เดียว | เขียนซ้ำหลายครั้ง หรือใช้ loop | ส่ง tuple ของหลาย model เข้า decorator ได้เลย |
| ใช้กับ custom `AdminSite` | `blog_admin_site.register(...)` ตรง ๆ | ส่ง `site=blog_admin_site` เป็น argument |
| ความนิยมในโค้ดปัจจุบัน (Django docs, โปรเจกต์ใหม่) | ลดลง แต่ยังพบได้บ่อยในโค้ดเก่า | เป็นมาตรฐานแนะนำในเอกสาร Django ตั้งแต่ 1.7 |
| Unregister model ที่ decorator ลงทะเบียนแล้ว | `admin.site.unregister(Post)` เหมือนกันทั้งคู่ | เหมือนกัน |

### 175.3 ลงทะเบียนหลาย Model ด้วย Decorator เดียว

```python
@admin.register(Category, Tag)
class SimpleNameAdmin(admin.ModelAdmin):
    list_display = ["name"]
    search_fields = ["name"]
```

โค้ดข้างบนใช้ `ModelAdmin` เดียวกันกับทั้ง `Category` และ `Tag` เพราะทั้งสอง model
มีโครงสร้างคล้ายกันมาก (มี field `name` เหมือนกัน) — ลดโค้ดซ้ำซ้อนได้ดี

### 175.4 ลงทะเบียนกับ Custom AdminSite ผ่าน Decorator

```python
from config.admin import blog_admin_site

@admin.register(Post, site=blog_admin_site)
class PostAdmin(admin.ModelAdmin):
    list_display = ["title", "is_published"]
```

### 175.5 Unregister แล้ว Register ใหม่ (แก้ไข Admin ของ Third-Party App)

บางครั้งเราต้องการปรับแต่ง Admin ของ model ที่มาจาก app อื่น (เช่น `User` จาก
`django.contrib.auth`) วิธีมาตรฐานคือ unregister ของเดิมก่อนแล้วค่อย register ใหม่:

```python
# accounts/admin.py
from django.contrib import admin
from django.contrib.auth.admin import UserAdmin
from django.contrib.auth.models import User
from .models import Profile


class ProfileInline(admin.StackedInline):
    model = Profile
    can_delete = False
    verbose_name_plural = "โปรไฟล์"


admin.site.unregister(User)   # ต้อง unregister ก่อน ไม่งั้น register ซ้ำจะ error


@admin.register(User)
class CustomUserAdmin(UserAdmin):
    inlines = [ProfileInline]                     # แสดง Profile ในหน้าแก้ไข User เดียวกัน
    list_display = UserAdmin.list_display + ("date_joined",)
```

**ข้อควรระวัง**: ถ้าลืม `admin.site.unregister(User)` ก่อนแล้วพยายาม
`@admin.register(User)` ซ้ำ Django จะโยน `django.contrib.admin.sites.AlreadyRegistered`
exception ทันทีตอน startup

---

## ขั้นตอนที่ 176: เพิ่ม Custom View เข้า Admin เอง (`get_urls()`)

### 176.1 แนวคิด: Admin ก็คือ Django View ธรรมดา ๆ

Django Admin ทั้งระบบสร้างจาก URL patterns กับ view functions เหมือนเว็บทั่วไป
ทุกประการ — เราจึงสามารถ **เพิ่ม URL/view ของตัวเอง** เข้าไปในระบบ Admin ได้โดย
override method `get_urls()` ของ `ModelAdmin` (หรือ `AdminSite`)

### 176.2 เพิ่มหน้า Dashboard สถิติเข้าไปใน Admin

```python
# blog/admin.py
from django.contrib import admin
from django.urls import path
from django.shortcuts import render
from django.db.models import Count, Sum
from django.utils import timezone
from datetime import timedelta
from .models import Post, Category, Comment


class PostAdmin(admin.ModelAdmin):
    list_display = ["title", "is_published"]

    def get_urls(self):
        # ต้องเรียก super().get_urls() ก่อนเสมอ แล้วค่อย "เติม" URL ของเราไว้ด้านหน้า
        custom_urls = [
            path(
                "dashboard/",
                self.admin_site.admin_view(self.dashboard_view),  # ← ดูอธิบายด้านล่าง
                name="blog_dashboard",
            ),
        ]
        return custom_urls + super().get_urls()

    def dashboard_view(self, request):
        thirty_days_ago = timezone.now() - timedelta(days=30)

        stats = {
            "total_posts": Post.objects.count(),
            "published_posts": Post.objects.filter(is_published=True).count(),
            "draft_posts": Post.objects.filter(is_published=False).count(),
            "total_comments": Comment.objects.count(),
            "total_views": Post.objects.aggregate(total=Sum("view_count"))["total"] or 0,
            "recent_posts": Post.objects.filter(created_at__gte=thirty_days_ago).count(),
        }

        top_categories = (
            Category.objects.annotate(post_count=Count("posts"))
            .order_by("-post_count")[:5]
        )

        context = {
            **self.admin_site.each_context(request),  # ต้องใส่เสมอ! ให้ template ได้ context มาตรฐาน (site_header, user, ฯลฯ)
            "title": "Dashboard สถิติบล็อก",
            "stats": stats,
            "top_categories": top_categories,
        }
        return render(request, "admin/blog/dashboard.html", context)
```

**อธิบายจุดสำคัญที่มือใหม่มักพลาด**:

1. **ลำดับการ merge URL**: ต้องใส่ `custom_urls + super().get_urls()` (ของเราไว้
   ก่อน) ไม่ใช่ `super().get_urls() + custom_urls` เพราะ URL pattern ของ Django
   Admin เดิมมี catch-all pattern อย่าง `<path:object_id>/change/` ที่จะจับ
   `dashboard/` ไปตีความว่าเป็น `object_id` ถ้าเราใส่สลับลำดับผิด
2. **`self.admin_site.admin_view(...)`**: ต้องห่อ view function ของเราด้วย
   `admin_view()` เสมอ มันทำ 2 หน้าที่สำคัญ: (ก) เช็คว่า user login และมีสิทธิ์
   staff หรือไม่ ก่อนปล่อยให้เข้าหน้านั้น (ข) ครอบด้วย `never_cache` เพื่อไม่ให้
   เบราว์เซอร์ cache หน้า Admin ที่มีข้อมูล sensitive ถ้าลืมห่อ ใครก็เข้าหน้านี้
   ได้โดยไม่ต้อง login เลย ซึ่งเป็นช่องโหว่ความปลอดภัยร้ายแรง
3. **`self.admin_site.each_context(request)`**: ต้องรวมเข้าไปใน context เสมอ
   มันให้ตัวแปรมาตรฐานที่ template ของ Admin ต้องใช้ เช่น `site_header`,
   `has_permission`, `available_apps` — ถ้าลืม หน้า dashboard จะ layout พังเพราะ
   `base_site.html` เดิมอ้างถึงตัวแปรพวกนี้

### 176.3 สร้าง Template สำหรับหน้า Dashboard

```html
{# templates/admin/blog/dashboard.html #}
{% extends "admin/base_site.html" %}
{% load i18n %}

{% block content %}
<div id="content-main">
    <h1>📈 Dashboard สถิติบล็อก</h1>

    <div style="display: grid; grid-template-columns: repeat(3, 1fr); gap: 16px; margin: 20px 0;">
        <div class="module" style="padding: 16px; text-align: center;">
            <h2 style="font-size: 2rem; margin: 0;">{{ stats.total_posts }}</h2>
            <p>โพสต์ทั้งหมด</p>
        </div>
        <div class="module" style="padding: 16px; text-align: center;">
            <h2 style="font-size: 2rem; margin: 0; color: green;">{{ stats.published_posts }}</h2>
            <p>เผยแพร่แล้ว</p>
        </div>
        <div class="module" style="padding: 16px; text-align: center;">
            <h2 style="font-size: 2rem; margin: 0; color: orange;">{{ stats.draft_posts }}</h2>
            <p>ฉบับร่าง</p>
        </div>
        <div class="module" style="padding: 16px; text-align: center;">
            <h2 style="font-size: 2rem; margin: 0;">{{ stats.total_comments }}</h2>
            <p>คอมเมนต์ทั้งหมด</p>
        </div>
        <div class="module" style="padding: 16px; text-align: center;">
            <h2 style="font-size: 2rem; margin: 0;">{{ stats.total_views }}</h2>
            <p>ยอดวิวรวม</p>
        </div>
        <div class="module" style="padding: 16px; text-align: center;">
            <h2 style="font-size: 2rem; margin: 0;">{{ stats.recent_posts }}</h2>
            <p>โพสต์ใหม่ใน 30 วัน</p>
        </div>
    </div>

    <div class="module">
        <h2>หมวดหมู่ยอดนิยม (Top 5)</h2>
        <table>
            <thead>
                <tr><th>หมวดหมู่</th><th>จำนวนโพสต์</th></tr>
            </thead>
            <tbody>
                {% for category in top_categories %}
                <tr>
                    <td>{{ category.name }}</td>
                    <td>{{ category.post_count }}</td>
                </tr>
                {% empty %}
                <tr><td colspan="2">ยังไม่มีข้อมูล</td></tr>
                {% endfor %}
            </tbody>
        </table>
    </div>
</div>
{% endblock %}
```

### 176.4 เพิ่มลิงก์ไปหน้า Dashboard บนหน้า Index ของ Admin

ผูกกับสิ่งที่เรียนใน 172.5 (override `change_list.html`) เพื่อให้ผู้ใช้กดเข้าถึง
หน้า dashboard ได้จากปุ่มบนตาราง ไม่ต้องพิมพ์ URL เอง

### 176.5 เพิ่ม Custom View ระดับ `AdminSite` (ครอบคลุมทุก App)

ถ้าอยากให้หน้า dashboard อยู่ที่ระดับบนสุดของ Admin (ไม่ผูกกับ model ใดโดยเฉพาะ)
ให้ override `get_urls()` ที่ `AdminSite` แทน:

```python
# config/admin.py
from django.contrib.admin import AdminSite
from django.urls import path


class BlogAdminSite(AdminSite):
    site_header = "ระบบจัดการบล็อก MyCompany"

    def get_urls(self):
        from blog.views_admin import global_dashboard_view
        custom_urls = [
            path("global-dashboard/", self.admin_view(global_dashboard_view), name="global_dashboard"),
        ]
        return custom_urls + super().get_urls()


blog_admin_site = BlogAdminSite(name="blog_admin")
```

---

## ขั้นตอนที่ 177: ภาพรวม Django Admin Theme ของบุคคลที่สาม

### 177.1 ทำไมถึงมี Package Theme แยกต่างหาก

Admin ดั้งเดิมของ Django ออกแบบมาให้ **ใช้งานได้ ปลอดภัย และเสถียร** เป็นหลัก
ไม่ได้เน้นความสวยงามระดับ modern UI เท่าไหร่ จึงมี package บุคคลที่สามหลายตัวที่มา
"แต่งหน้าทาปาก" ให้ Admin ดูทันสมัยขึ้น โดยยังใช้ระบบ `ModelAdmin`,
`register()`, permission ฯลฯ เหมือนเดิมทุกประการ — เปลี่ยนแค่ theme/CSS/JS

### 177.2 ตารางเปรียบเทียบ 3 Package ยอดนิยม

| Package | จุดเด่น | ติดตั้งง่ายแค่ไหน | เหมาะกับ |
|---|---|---|---|
| **django-jazzmin** | UI สวยแบบ AdminLTE, ปรับแต่ง sidebar/สี/ไอคอนผ่าน settings ได้เยอะ, dark mode | ง่ายมาก แค่ใส่ใน `INSTALLED_APPS` ก่อน `django.contrib.admin` | โปรเจกต์ที่อยากได้ modern look เร็ว ๆ โดยไม่ต้องเขียน CSS เอง |
| **django-unfold** | ดีไซน์ทันสมัยแบบ Tailwind CSS, รองรับ dashboard แบบ component, responsive ดีมาก | ง่าย-ปานกลาง ต้องปรับ `ModelAdmin` ให้สืบทอดจาก `ModelAdmin` ของ Unfold บางส่วน | โปรเจกต์ใหม่ที่ต้องการ UI ระดับ production จริงจัง |
| **django-grappelli** | เก่าแก่ที่สุด ใช้กันมานาน มี autocomplete ในตัว, การจัดกลุ่มเมนูดี | ปานกลาง ต้องตั้งค่า `GRAPPELLI_ADMIN_TITLE` และปรับ `urls.py` | ทีมที่คุ้นเคยกับ jQuery UI แบบดั้งเดิม, โปรเจกต์เก่าที่ใช้อยู่แล้ว |

### 177.3 ตัวอย่างการติดตั้ง django-jazzmin

```bash
pip install django-jazzmin
```

```python
# config/settings/base.py
INSTALLED_APPS = [
    "jazzmin",                    # ⚠️ ต้องอยู่ "ก่อน" django.contrib.admin เสมอ
    "django.contrib.admin",
    "django.contrib.auth",
    # ...
    "blog",
    "accounts",
]

JAZZMIN_SETTINGS = {
    "site_title": "MyCompany Admin",
    "site_header": "MyCompany",
    "welcome_sign": "ยินดีต้อนรับเข้าสู่ระบบจัดการบล็อก",
    "topmenu_links": [
        {"name": "หน้าแรกเว็บไซต์", "url": "/", "new_window": True},
    ],
    "show_sidebar": True,
    "navigation_expanded": True,
    "icons": {
        "blog.Post": "fas fa-newspaper",
        "blog.Category": "fas fa-folder",
        "blog.Tag": "fas fa-tags",
    },
}
```

### 177.4 ตัวอย่างการติดตั้ง django-unfold

```bash
pip install django-unfold
```

```python
# config/settings/base.py
INSTALLED_APPS = [
    "unfold",                     # ต้องอยู่ก่อน django.contrib.admin เช่นกัน
    "django.contrib.admin",
    # ...
]

UNFOLD = {
    "SITE_TITLE": "MyCompany Admin",
    "SITE_HEADER": "MyCompany",
    "SITE_ICON": lambda request: "/static/blog/img/icon.svg",
}
```

```python
# blog/admin.py — ModelAdmin ของ Unfold ต้องสืบทอดจาก unfold.admin.ModelAdmin
from unfold.admin import ModelAdmin

@admin.register(Post)
class PostAdmin(ModelAdmin):   # แทนที่ admin.ModelAdmin ปกติ
    list_display = ["title", "is_published"]
```

### 177.5 ข้อควรพิจารณาก่อนติดตั้ง Theme ใด ๆ

- **การ override template ที่ทำไว้เอง (ขั้นตอนที่ 172) อาจต้องปรับใหม่** เพราะ
  theme บุคคลที่สามมักเปลี่ยนโครงสร้าง block ใน `base_site.html`
- ตรวจสอบว่า package รองรับ **Django เวอร์ชันปัจจุบัน** ที่ใช้ในหลักสูตร (5.x)
  ก่อนติดตั้งเสมอ (ดู compatibility matrix ใน PyPI/GitHub ของแต่ละ package)
- Custom view ที่เพิ่มด้วย `get_urls()` (ขั้นตอนที่ 176) ยังทำงานได้ปกติเสมอ ไม่ว่า
  จะใช้ theme ไหน เพราะเป็นกลไกระดับ URL routing ไม่เกี่ยวกับ CSS/JS ของ theme
- สำหรับหลักสูตรนี้ เราจะยังคงใช้ **Admin ดั้งเดิมของ Django** ต่อไปในทุก Part
  เพื่อให้เข้าใจกลไกจริงเบื้องหลังก่อน แต่คุณสามารถเลือกติดตั้ง theme ใดก็ได้ตาม
  ความชอบในโปรเจกต์ของตัวเอง เพราะ concept ที่เรียนไปทั้งหมดยังใช้ได้เหมือนเดิม

---

## ขั้นตอนที่ 178: Django Admin Security Best Practices

### 178.1 ทำไม Admin ถึงเป็นเป้าหมายการโจมตีอันดับต้น ๆ

Admin panel มีสิทธิ์เข้าถึงและแก้ไขข้อมูลทั้งระบบ จึงเป็นเป้าหมายแรกที่ผู้ไม่หวังดี
พยายามเจาะ (brute-force login, credential stuffing) แนวทางด้านล่างเป็น **best
practice ระดับพื้นฐานถึงกลาง** ส่วน 2FA แบบเต็มรูปแบบจะเรียนลึกใน **Part 037**

### 178.2 เปลี่ยน URL Default ของ Admin (Security by Obscurity — เสริม ไม่ใช่หลัก)

```python
# config/urls.py
from django.urls import path

urlpatterns = [
    # เปลี่ยนจาก "admin/" เป็น path ที่เดายากและไม่ตรงตัวชัดเจน
    path("secure-portal-7k2x/", admin.site.urls),
]
```

**ข้อควรเข้าใจให้ถูกต้อง**: การเปลี่ยน URL **ไม่ใช่มาตรการความปลอดภัยหลัก** มันแค่
ลด "สัญญาณรบกวน" จาก bot ที่ scan `/admin/` แบบอัตโนมัติทั่วอินเทอร์เน็ต ผู้โจมตี
ที่ตั้งใจเจาะจริง ๆ ยังหา path ได้อยู่ดี (เช่นดูจาก sitemap, error message, หรือ
source code ที่รั่ว) จึงต้องใช้ร่วมกับมาตรการอื่นเสมอ **ห้ามพึ่งข้อนี้ข้อเดียว**

### 178.3 จำกัดการเข้าถึงด้วย IP Allowlist (Middleware)

```python
# config/middleware.py
from django.http import HttpResponseForbidden

ALLOWED_ADMIN_IPS = ["203.0.113.10", "203.0.113.11", "127.0.0.1"]


class AdminIPRestrictionMiddleware:
    def __init__(self, get_response):
        self.get_response = get_response

    def __call__(self, request):
        if request.path.startswith("/secure-portal-7k2x/"):
            client_ip = self._get_client_ip(request)
            if client_ip not in ALLOWED_ADMIN_IPS:
                return HttpResponseForbidden("Access denied: IP not allowed")
        return self.get_response(request)

    def _get_client_ip(self, request):
        # ใช้ X-Forwarded-For เมื่ออยู่หลัง reverse proxy/load balancer (เรียนละเอียด Part 91)
        forwarded_for = request.META.get("HTTP_X_FORWARDED_FOR")
        if forwarded_for:
            return forwarded_for.split(",")[0].strip()
        return request.META.get("REMOTE_ADDR")
```

```python
# config/settings/base.py
MIDDLEWARE = [
    "django.middleware.security.SecurityMiddleware",
    # ...
    "config.middleware.AdminIPRestrictionMiddleware",   # ใส่ต่อจาก security middleware
]
```

**คำเตือน**: การเชื่อ `HTTP_X_FORWARDED_FOR` โดยตรงมีความเสี่ยงถ้าไม่ได้อยู่หลัง
reverse proxy ที่เชื่อถือได้ (เพราะ header นี้ผู้ใช้ปลอมแปลงเองได้) ใน production
จริงควรตั้งค่า `USE_X_FORWARDED_FOR` และกำหนด proxy ที่เชื่อถือได้อย่างชัดเจน —
รายละเอียดเชิงลึกจะอยู่ใน Part เรื่อง Deployment (Phase 11)

### 178.4 บังคับ Staff-Only ด้วย Middleware เสริม

```python
# config/middleware.py
from django.shortcuts import redirect


class StaffOnlyAdminMiddleware:
    def __init__(self, get_response):
        self.get_response = get_response

    def __call__(self, request):
        if request.path.startswith("/secure-portal-7k2x/") and request.path != "/secure-portal-7k2x/login/":
            if request.user.is_authenticated and not request.user.is_staff:
                return redirect("home")  # เตะ user ที่ login แล้วแต่ไม่ใช่ staff ออกทันที
        return self.get_response(request)
```

โดยทั่วไป Django Admin เช็ค `is_staff` ให้อยู่แล้วผ่าน `has_permission()` ของ
`AdminSite` (ดูขั้นตอนที่ 173.2) แต่การเพิ่ม middleware ชั้นนี้ช่วยให้ **บล็อกตั้งแต่
ก่อนถึงชั้น URL routing ของ Admin เลย** ลด attack surface ลงไปอีกชั้นหนึ่ง

### 178.5 ตั้งค่า Session และ Cookie ให้ปลอดภัยขึ้นสำหรับ Production

```python
# config/settings/prod.py
SESSION_COOKIE_SECURE = True       # cookie ส่งผ่าน HTTPS เท่านั้น
SESSION_COOKIE_HTTPONLY = True     # JavaScript อ่าน cookie นี้ไม่ได้ (กัน XSS ขโมย session)
SESSION_COOKIE_AGE = 3600          # หมดอายุใน 1 ชั่วโมง (ปรับตามความเหมาะสมของทีม)
SESSION_EXPIRE_AT_BROWSER_CLOSE = True
CSRF_COOKIE_SECURE = True
X_FRAME_OPTIONS = "DENY"           # กัน Admin ถูกฝังใน iframe (clickjacking)
```

### 178.6 พิจารณาใช้ `django-admin-honeypot`

Package นี้สร้างหน้า Admin **ปลอมทั้งหน้า** ไว้ที่ `/admin/` (URL เดิมที่บอทมักลอง)
เพื่อดักจับและบันทึก IP ของผู้พยายามเข้า login โดยไม่รู้ตัวว่าถูกหลอก ในขณะที่ Admin
จริงถูกย้ายไป URL ลับตามข้อ 178.2:

```bash
pip install django-admin-honeypot
```

```python
# config/settings/base.py
INSTALLED_APPS = [
    "admin_honeypot",
    # ...
]
```

```python
# config/urls.py
urlpatterns = [
    path("admin/", include("admin_honeypot.urls", namespace="admin_honeypot")),
    path("secure-portal-7k2x/", admin.site.urls),  # Admin จริง
]
```

### 178.7 Checklist ความปลอดภัยของ Admin ก่อนขึ้น Production

| รายการ | ทำแล้วหรือยัง |
|---|---|
| เปลี่ยน URL admin จาก default | ⬜ |
| ตั้งค่า `SESSION_COOKIE_SECURE`, `CSRF_COOKIE_SECURE` เป็น `True` | ⬜ |
| จำกัด IP หรือใช้ VPN สำหรับเข้า Admin (ถ้าทีมงานมี IP คงที่) | ⬜ |
| ตั้งรหัสผ่านของทุก staff ตาม password policy ที่รัดกุม (Part 035) | ⬜ |
| เปิดใช้ 2FA สำหรับทุกบัญชี staff (Part 037) | ⬜ |
| ใช้ HTTPS เท่านั้น ไม่มี HTTP fallback | ⬜ |
| ตรวจสอบ `LogEntry` เป็นระยะเพื่อดูกิจกรรมผิดปกติ (ขั้นตอนที่ 179) | ⬜ |
| จำกัดจำนวน staff user ให้น้อยที่สุดเท่าที่จำเป็น (Principle of Least Privilege) | ⬜ |

---

## ขั้นตอนที่ 179: Admin Log — `LogEntry` ดูประวัติการแก้ไข

### 179.1 Django บันทึกทุกการกระทำใน Admin อัตโนมัติอยู่แล้ว

ทุกครั้งที่มีการเพิ่ม/แก้ไข/ลบ object ผ่าน **Django Admin** (ไม่ใช่ผ่าน API หรือ
shell) Django จะบันทึกลง table `django_admin_log` โดยอัตโนมัติ ผ่าน model
`django.contrib.admin.models.LogEntry` — คุณเคยเห็นผลลัพธ์นี้แล้วโดยไม่รู้ตัว:
มันคือข้อมูลที่แสดงในปุ่ม **"History"** มุมบนขวาของหน้าแก้ไข object ทุกหน้า

### 179.2 โครงสร้างของ `LogEntry`

```python
from django.contrib.admin.models import LogEntry, ADDITION, CHANGE, DELETION

# Field สำคัญของ LogEntry:
# - action_time: datetime ที่เกิดเหตุการณ์
# - user: FK ไปยัง User ที่ทำรายการ
# - content_type: FK ไปยัง ContentType (บอกว่าเป็น model อะไร ผ่าน Generic relation)
# - object_id: PK ของ object ที่ถูกกระทำ (เก็บเป็น string)
# - object_repr: str(obj) ตอนที่ถูกบันทึก (เผื่อ object ถูกลบไปแล้วในภายหลัง)
# - action_flag: ADDITION (1), CHANGE (2), หรือ DELETION (3)
# - change_message: JSON string อธิบายว่า field ไหนถูกเปลี่ยนเป็นอะไร
```

### 179.3 Query ประวัติการแก้ไขทั้งหมดของ User คนหนึ่ง

```python
from django.contrib.admin.models import LogEntry
from django.contrib.auth.models import User

user = User.objects.get(username="editor_somchai")
logs = LogEntry.objects.filter(user=user).order_by("-action_time")

for log in logs:
    print(f"{log.action_time}: {log.get_action_flag_display()} on {log.object_repr}")
```

### 179.4 Query ประวัติการแก้ไขของ Object เจาะจง (เช่นโพสต์ตัวหนึ่ง)

```python
from django.contrib.admin.models import LogEntry
from django.contrib.contenttypes.models import ContentType
from blog.models import Post

post = Post.objects.get(pk=1)
content_type = ContentType.objects.get_for_model(Post)

logs = LogEntry.objects.filter(
    content_type=content_type,
    object_id=str(post.pk),   # object_id เก็บเป็น string เสมอ ต้อง cast ให้ตรงกัน
).order_by("-action_time")

for log in logs:
    print(f"{log.action_time} โดย {log.user}: {log.change_message}")
```

### 179.5 แยกประเภท Action ด้วย `ADDITION`, `CHANGE`, `DELETION`

```python
from django.contrib.admin.models import LogEntry, ADDITION, CHANGE, DELETION

recent_deletions = LogEntry.objects.filter(action_flag=DELETION).order_by("-action_time")[:20]

for log in recent_deletions:
    print(f"⚠️ {log.user} ลบ {log.object_repr} เมื่อ {log.action_time}")
```

### 179.6 เพิ่มหน้า "Recent Activity" เข้า Dashboard ที่สร้างไว้ในขั้นตอนที่ 176

ต่อยอดจาก `dashboard_view` ในขั้นตอนที่ 176 เพื่อโชว์กิจกรรมล่าสุดทั้งระบบ:

```python
# blog/admin.py (เพิ่มเข้าไปใน dashboard_view เดิม)
from django.contrib.admin.models import LogEntry

class PostAdmin(admin.ModelAdmin):
    # ... (โค้ดเดิมจากขั้นตอนที่ 176)

    def dashboard_view(self, request):
        # ... (โค้ดเดิม: stats, top_categories)

        recent_activity = (
            LogEntry.objects.select_related("user", "content_type")
            .order_by("-action_time")[:15]
        )

        context = {
            **self.admin_site.each_context(request),
            "title": "Dashboard สถิติบล็อก",
            "stats": stats,
            "top_categories": top_categories,
            "recent_activity": recent_activity,   # ← เพิ่มตัวแปรใหม่
        }
        return render(request, "admin/blog/dashboard.html", context)
```

```html
{# เพิ่มใน templates/admin/blog/dashboard.html #}
<div class="module" style="margin-top: 20px;">
    <h2>🕒 กิจกรรมล่าสุดใน Admin</h2>
    <table>
        <thead>
            <tr><th>เวลา</th><th>ผู้ใช้</th><th>การกระทำ</th><th>Object</th></tr>
        </thead>
        <tbody>
            {% for log in recent_activity %}
            <tr>
                <td>{{ log.action_time|date:"d/m/Y H:i" }}</td>
                <td>{{ log.user }}</td>
                <td>{{ log.get_action_flag_display }}</td>
                <td>{{ log.object_repr }}</td>
            </tr>
            {% empty %}
            <tr><td colspan="4">ยังไม่มีกิจกรรม</td></tr>
            {% endfor %}
        </tbody>
    </table>
</div>
```

### 179.7 ข้อจำกัดสำคัญของ `LogEntry` ที่ต้องรู้

- `LogEntry` **บันทึกเฉพาะการกระทำที่ผ่าน Django Admin เท่านั้น** การเปลี่ยนแปลง
  ผ่าน API (Django REST Framework), management command, หรือ Django shell
  **จะไม่ถูกบันทึก** ถ้าต้องการ audit log ที่ครอบคลุมทุกช่องทาง ต้องใช้ signals
  (Part 019) หรือ package เฉพาะทางอย่าง `django-simple-history` /
  `django-auditlog`
- `change_message` เป็น JSON string ที่ไม่ได้ระบุ **ค่าเก่า/ค่าใหม่แบบละเอียด**
  เสมอไป (บอกแค่ชื่อ field ที่เปลี่ยน ไม่บอกค่าเดิม) หากต้องการ diff แบบเต็ม
  ต้องใช้ package เสริมเช่นกัน
- ตาราง `django_admin_log` **ไม่มีการลบอัตโนมัติ** ระบบที่มีกิจกรรมเยอะมากควรมี
  แผนเก็บ archive/ล้างข้อมูลเก่าเป็นระยะ (จะเรียนเรื่อง data retention ใน Phase
  Database ขั้นสูง)

---

## ขั้นตอนที่ 180: สรุปและแบบฝึกหัด

### 180.1 ประกอบร่าง: `blog/admin.py` ฉบับสมบูรณ์ของ Part นี้

นี่คือไฟล์ `admin.py` ที่รวมทุกเทคนิคจาก Part นี้เข้าด้วยกันเป็นระบบเดียว
(ต่อยอดจาก Part 017 อย่างสมบูรณ์):

```python
# blog/admin.py
import csv

from django.contrib import admin, messages
from django.contrib.admin.models import LogEntry
from django.db.models import Count, Sum
from django.http import HttpResponse
from django.shortcuts import render
from django.urls import path
from django.utils import timezone
from django.utils.translation import ngettext
from datetime import timedelta

from .models import Post, Category, Tag, Comment

admin.site.site_header = "ระบบจัดการบล็อก MyCompany"
admin.site.site_title = "MyCompany Admin"
admin.site.index_title = "แผงควบคุมผู้ดูแลระบบ"


@admin.action(description="✅ Mark selected posts as published")
def mark_as_published(modeladmin, request, queryset):
    updated_count = queryset.update(is_published=True)
    modeladmin.message_user(
        request,
        ngettext("%d โพสต์ถูกเผยแพร่แล้ว", "%d โพสต์ถูกเผยแพร่แล้ว", updated_count)
        % updated_count,
        messages.SUCCESS,
    )


@admin.action(description="🚫 Mark selected posts as draft")
def mark_as_draft(modeladmin, request, queryset):
    updated_count = queryset.update(is_published=False)
    modeladmin.message_user(request, f"{updated_count} โพสต์ถูกเปลี่ยนเป็นฉบับร่างแล้ว", messages.WARNING)


@admin.action(description="⬇️ Export selected posts as CSV")
def export_posts_as_csv(modeladmin, request, queryset):
    response = HttpResponse(content_type="text/csv")
    response["Content-Disposition"] = 'attachment; filename="posts_export.csv"'
    writer = csv.writer(response)
    writer.writerow(["ID", "Title", "Category", "Published", "Views", "Created At"])
    for post in queryset.select_related("category"):
        writer.writerow([
            post.id,
            post.title,
            post.category.name if post.category else "-",
            "Yes" if post.is_published else "No",
            post.view_count,
            post.created_at.strftime("%Y-%m-%d %H:%M"),
        ])
    return response


class CommentInline(admin.TabularInline):
    model = Comment
    extra = 0
    fields = ["author", "text", "parent", "created_at"]
    readonly_fields = ["created_at"]


@admin.register(Post)
class PostAdmin(admin.ModelAdmin):
    list_display = ["title", "category", "is_published", "view_count", "created_at"]
    list_filter = ["is_published", "category", "created_at"]
    search_fields = ["title", "content"]
    prepopulated_fields = {"slug": ("title",)}
    list_select_related = ["category"]
    inlines = [CommentInline]
    actions = [mark_as_published, mark_as_draft, export_posts_as_csv]

    def get_urls(self):
        custom_urls = [
            path(
                "dashboard/",
                self.admin_site.admin_view(self.dashboard_view),
                name="blog_dashboard",
            ),
        ]
        return custom_urls + super().get_urls()

    def dashboard_view(self, request):
        thirty_days_ago = timezone.now() - timedelta(days=30)
        stats = {
            "total_posts": Post.objects.count(),
            "published_posts": Post.objects.filter(is_published=True).count(),
            "draft_posts": Post.objects.filter(is_published=False).count(),
            "total_comments": Comment.objects.count(),
            "total_views": Post.objects.aggregate(total=Sum("view_count"))["total"] or 0,
            "recent_posts": Post.objects.filter(created_at__gte=thirty_days_ago).count(),
        }
        top_categories = Category.objects.annotate(post_count=Count("posts")).order_by("-post_count")[:5]
        recent_activity = LogEntry.objects.select_related("user", "content_type").order_by("-action_time")[:15]

        context = {
            **self.admin_site.each_context(request),
            "title": "Dashboard สถิติบล็อก",
            "stats": stats,
            "top_categories": top_categories,
            "recent_activity": recent_activity,
        }
        return render(request, "admin/blog/dashboard.html", context)


@admin.register(Category, Tag)
class SimpleNameAdmin(admin.ModelAdmin):
    list_display = ["name"]
    search_fields = ["name"]


@admin.register(Comment)
class CommentAdmin(admin.ModelAdmin):
    list_display = ["post", "author", "parent", "created_at"]
    list_filter = ["created_at"]
    search_fields = ["text", "author__username"]
```

และไฟล์ `templates/admin/blog/post/change_list.html` ที่เพิ่มลิงก์เข้าหน้า
dashboard:

```html
{% extends "admin/change_list.html" %}

{% block object-tools-items %}
    <li>
        <a href="{% url 'admin:blog_dashboard' %}" class="button">📈 ดู Dashboard สถิติ</a>
    </li>
    {{ block.super }}
{% endblock %}
```

### 180.2 สรุปสิ่งที่ได้เรียนรู้ใน Part นี้

- ✅ เขียน Custom Admin Action ด้วย `@admin.action` ทั้งแบบฟังก์ชันอิสระและแบบ
  method, ทำ bulk update, export เป็น CSV, และจำกัดสิทธิ์การเห็น action
- ✅ เข้าใจ Template Override Hierarchy ของ Admin ตั้งแต่ระดับ model เฉพาะเจาะจง
  ไปจนถึง default ของ Django และรู้จัก block สำคัญอย่าง `field_sets`,
  `object-tools-items`, `branding`
- ✅ ปรับแบรนด์ Admin ด้วย `site_header`/`site_title`/`index_title` และสร้าง
  `AdminSite` subclass ของตัวเองเมื่อโปรเจกต์ซับซ้อนขึ้น
- ✅ ควบคุม permission ใน Admin ด้วย `has_add/change/delete/view_permission`
  ตามตรรกะธุรกิจจริง ควบคู่กับการกรอง `get_queryset()` ให้สอดคล้องกัน
- ✅ เข้าใจความแตกต่างระหว่าง `@admin.register` กับ `admin.site.register()` และ
  วิธี unregister/register ใหม่สำหรับ model ของ app อื่น
- ✅ เพิ่มหน้า custom view (dashboard) เข้า Admin ด้วย `get_urls()` พร้อมเข้าใจ
  ความสำคัญของ `admin_view()` และ `each_context()`
- ✅ รู้จัก theme บุคคลที่สาม (django-jazzmin, django-unfold, django-grappelli)
  และเมื่อไหร่ควรเลือกใช้
- ✅ เข้าใจแนวทางความปลอดภัยของ Admin: เปลี่ยน URL, จำกัด IP, staff-only
  middleware, ตั้งค่า cookie/session สำหรับ production
- ✅ Query และแสดงผล `LogEntry` เพื่อดูประวัติการแก้ไขผ่าน Admin พร้อมรู้ข้อจำกัด
  ของมัน

### 180.3 Checklist ก่อนไป Part ถัดไป

- [ ] เขียน custom admin action อย่างน้อย 2 ตัว (bulk update + export CSV) และ
      ทดสอบใช้งานจริงในเบราว์เซอร์
- [ ] Override `change_form.html` หรือ `change_list.html` ของ model อย่างน้อย
      1 model สำเร็จ และเห็นการเปลี่ยนแปลงจริงในหน้า Admin
- [ ] ตั้งค่า `site_header`/`site_title`/`index_title` ของโปรเจกต์ตัวเอง
- [ ] เขียน `has_change_permission`/`has_delete_permission` ตามกฎธุรกิจอย่างน้อย
      1 กรณี และทดสอบด้วย user 2 แบบที่สิทธิ์ต่างกัน
- [ ] สร้างหน้า custom view ผ่าน `get_urls()` อย่างน้อย 1 หน้า (เช่น dashboard)
      และเข้าถึงได้จริงผ่าน URL `admin:<name>`
- [ ] Query `LogEntry` ผ่าน Django shell แล้วเห็นประวัติการกระทำของตัวเอง

### 180.4 แบบฝึกหัดท้ายบท

**แบบฝึกหัดที่ 1**: เพิ่ม action ใหม่ชื่อ `duplicate_posts` ที่รับโพสต์ที่เลือกไว้
แล้วสร้างสำเนา (copy) ของแต่ละโพสต์ขึ้นมาใหม่ โดยตั้ง `title` ใหม่เป็น
`"{original.title} (Copy)"`, `is_published=False` เสมอ, และต้อง generate `slug`
ใหม่ที่ไม่ชนกับของเดิม (คำใบ้: ใช้ `pk=None` แล้ว `save()` เพื่อสร้าง object ใหม่
ใน Django ORM, และดูวิธี generate slug จาก Part 015)

**แบบฝึกหัดที่ 2**: Override `templates/admin/blog/comment/change_form.html` ให้
แสดงกล่องข้อความเตือนสีแดงเมื่อคอมเมนต์นั้นมี `parent` ไม่เป็น `None` (คือเป็น
comment ตอบกลับ) พร้อมแสดงข้อความต้นฉบับของคอมเมนต์แม่ (`{{ original.parent.text }}`)

**แบบฝึกหัดที่ 3**: เขียน permission logic ให้ `CommentAdmin` โดยที่ staff ทั่วไป
**ลบคอมเมนต์ของคนอื่นไม่ได้** (ลบได้เฉพาะคอมเมนต์ของตัวเอง) แต่ **แก้ไขได้ทุก
คอมเมนต์** (เพื่อแก้คำหยาบหรือ spam) ส่วน superuser ทำได้ทุกอย่างตามปกติ

**แบบฝึกหัดที่ 4 (ขั้นสูง)**: ขยายหน้า `dashboard_view` ที่สร้างไว้ในบทนี้ ให้เพิ่ม
กราฟแท่งอย่างง่าย (ใช้ inline `<div>` ที่มี `width` เป็นเปอร์เซ็นต์ตามสัดส่วนข้อมูล
ก็พอ ไม่ต้องใช้ library JS ภายนอก) แสดงจำนวนโพสต์แยกตามหมวดหมู่ 5 อันดับแรก และ
เพิ่มการจำกัดสิทธิ์ว่าเฉพาะ superuser เท่านั้นที่เข้าหน้านี้ได้ (คำใบ้: เช็ค
`request.user.is_superuser` ใน `dashboard_view` แล้ว `return HttpResponseForbidden()`
ถ้าไม่ผ่าน)

### 180.5 คำถามที่พบบ่อย (FAQ)

**Q: `queryset.update()` ใน action เร็วกว่า loop `for obj in queryset: obj.save()`
แค่ไหน และมีข้อเสียอะไรบ้าง?**
A: `update()` ยิง SQL คำสั่งเดียว (`UPDATE ... WHERE ...`) จึงเร็วกว่ามากเมื่อมี
หลายพันแถว แต่ข้อเสียคือ **ไม่เรียก `save()` ของแต่ละ instance** ทำให้ logic ใน
`save()` ที่ override ไว้ (เช่น auto-generate slug ใน Part 015) และ **signals**
`pre_save`/`post_save` (Part 019) จะไม่ถูกยิง ต้องเลือกให้เหมาะกับสถานการณ์:
ถ้า field ที่อัปเดตไม่เกี่ยวกับ logic พิเศษใด ๆ ใช้ `update()` ได้เต็มที่

**Q: ทำไม override template ของ Admin แล้วบางทีไม่เห็นผลเลย ทั้งที่ path ถูกแล้ว?**
A: สาเหตุที่พบบ่อยที่สุดคือ **path การจัดวางโฟลเดอร์ผิด** (ต้องเป็น
`templates/admin/<app_label>/<model_name>/...` โดย `app_label` และ
`model_name` ต้องเป็นตัวพิมพ์เล็กเสมอ) หรือ Django ยัง cache template อยู่ระหว่าง
develop (ลองรีสตาร์ท `runserver`) หรือ `APP_DIRS`/`DIRS` ใน `TEMPLATES` setting
ตั้งค่าไม่ถูกต้องตามที่อธิบายในขั้นตอนที่ 172.3

**Q: ควรใช้ custom `AdminSite` หรือ third-party theme อย่าง jazzmin/unfold ดี?**
A: ไม่ขัดแย้งกัน ใช้ร่วมกันได้เสมอ — `AdminSite` subclass ควบคุม **โครงสร้าง
สิทธิ์และ URL** ส่วน theme package ควบคุม **หน้าตา CSS/JS** เลือกใช้ตามความ
ต้องการจริง: ถ้าต้องการแค่สวยขึ้นเร็ว ๆ ใช้ theme พอ ถ้าต้องการแยกระบบ Admin หลาย
ชุดตาม business logic ให้สร้าง `AdminSite` subclass ควบคู่ไปด้วย

**Q: `LogEntry` เก็บข้อมูลตลอดไปโดยไม่มีวันหมดอายุใช่ไหม จะกระทบ performance
ไหมถ้าใช้งานนานหลายปี?**
A: ใช่ Django ไม่ลบ `LogEntry` ให้อัตโนมัติ ในระบบที่มีกิจกรรมสูงควรตั้ง
management command หรือ Celery periodic task (Part 076) ลบ record ที่เก่าเกิน
กำหนด (เช่นเกิน 1 ปี) เป็นระยะ เพื่อไม่ให้ตาราง `django_admin_log` โตจนกระทบ
performance ของ query ที่เกี่ยวข้อง

---

## เตรียมตัวสำหรับ Part ถัดไป

**Part 019: Signals และ Django Lifecycle Hooks** จะพาไปเจาะลึกกลไกเบื้องหลังที่
เราพูดถึงไปหลายครั้งใน Part นี้ (เช่นตอนอธิบายว่า `queryset.update()` ไม่ยิง
signal) คุณจะได้เรียนรู้ `pre_save`, `post_save`, `pre_delete`, `post_delete`,
`m2m_changed` และการเขียน custom signal ของตัวเอง รวมถึงรู้ว่าเมื่อไหร่ควรใช้
signal เมื่อไหร่ควรใช้ `save()` override หรือ service layer แทน — เป็นความรู้ที่
เชื่อมโยงตรงกับ `LogEntry` และ Admin actions ที่เพิ่งเรียนไปในบทนี้อย่างแนบแน่น

เตรียม Django shell และไฟล์ `models.py` ของแอป `blog` ให้พร้อม แล้วไปต่อกันเลย!
