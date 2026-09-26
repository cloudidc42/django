# Part 022: Generic Class-Based Views: ListView, DetailView

> **ขั้นตอนที่ 211-220 ของหลักสูตร** | Phase 3: Views, Templates, Forms และ CBV
>
> เป้าหมายของ Part นี้: เลิกเขียน CBV แบบ manual ที่คุณสร้างเองใน Part 021 แล้วเปลี่ยนมาใช้
> **Generic Class-Based Views** ที่ Django เตรียมไว้ให้สำเร็จรูป โดยเฉพาะ `ListView` และ
> `DetailView` สองตัวที่ใช้บ่อยที่สุดในโปรเจกต์จริงทุกโปรเจกต์ เมื่อจบ Part นี้ คุณจะเข้าใจ
> ธรรมเนียมการตั้งชื่อ template อัตโนมัติ, การทำ pagination แทบไม่ต้องเขียนโค้ดเอง, การ
> override `get_queryset()`/`get_context_data()`/`get_object()` เพื่อควบคุมพฤติกรรมของ view
> ทุกจุด และสามารถแปลง `blog_list`/`blog_detail` ทั้งหมดให้เป็น `PostListView`/`PostDetailView`
> ระดับ production ได้อย่างมั่นใจ

---

## สารบัญของ Part นี้

- ขั้นตอนที่ 211: `ListView` เบื้องต้น — `model`, `template_name`, `context_object_name`
- ขั้นตอนที่ 212: Pagination ใน ListView ด้วย `paginate_by`
- ขั้นตอนที่ 213: `DetailView` เบื้องต้น — `get_object()`, `slug_field`, `slug_url_kwarg`
- ขั้นตอนที่ 214: Override `get_queryset()` ใน ListView/DetailView
- ขั้นตอนที่ 215: Override `get_context_data()` เพื่อเพิ่มข้อมูลพิเศษเข้า template
- ขั้นตอนที่ 216: ListView พร้อมระบบค้นหา/กรองผ่าน query parameter
- ขั้นตอนที่ 217: การจัดการ 404 ใน DetailView
- ขั้นตอนที่ 218: รวมข้อมูลหลายอย่างใน `get_context_data()` ของ DetailView
- ขั้นตอนที่ 219: Case study — แปลง `blog_list`/`blog_detail` เป็น `PostListView`/`PostDetailView` เต็มรูปแบบ
- ขั้นตอนที่ 220: สรุปและแบบฝึกหัด

---

## ขั้นตอนที่ 211: `ListView` เบื้องต้น — `model`, `template_name`, `context_object_name`

### 211.1 ทบทวนสถานะโปรเจกต์จาก Part 021

ก่อนเริ่ม Part นี้ โปรเจกต์ของคุณควรมีสถานะดังนี้ (ต่อยอดจาก Part 005-012):

```python
# blog/models.py
from django.db import models
from django.utils.text import slugify


class Category(models.Model):
    name = models.CharField(max_length=100, unique=True)
    slug = models.SlugField(max_length=120, unique=True, blank=True)
    description = models.TextField(blank=True)

    class Meta:
        ordering = ['name']
        verbose_name_plural = 'categories'

    def __str__(self):
        return self.name

    def save(self, *args, **kwargs):
        if not self.slug:
            self.slug = slugify(self.name)
        super().save(*args, **kwargs)


class Post(models.Model):
    title = models.CharField(max_length=200)
    slug = models.SlugField(max_length=220, unique=True, blank=True)
    content = models.TextField()
    category = models.ForeignKey(
        Category,
        on_delete=models.SET_NULL,
        null=True,
        blank=True,
        related_name='posts',
    )
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)
    is_published = models.BooleanField(default=False)

    class Meta:
        ordering = ['-created_at']

    def __str__(self):
        return self.title

    def save(self, *args, **kwargs):
        if not self.slug:
            self.slug = slugify(self.title)
        super().save(*args, **kwargs)


class Comment(models.Model):
    post = models.ForeignKey(Post, on_delete=models.CASCADE, related_name='comments')
    author = models.CharField(max_length=100)
    content = models.TextField()
    created_at = models.DateTimeField(auto_now_add=True)

    class Meta:
        ordering = ['created_at']

    def __str__(self):
        return f'{self.author} แสดงความเห็นใน "{self.post.title}"'
```

และใน Part 021 คุณได้แปลง `blog_list`/`blog_detail` (function-based views จาก Part 007) ให้เป็น
**manual Class-Based Views** ที่สืบทอดจาก `django.views.View` โดยตรง เขียน `get()` เอง
ด้วยมือ ประมาณนี้:

```python
# blog/views.py (ผลลัพธ์จาก Part 021 — CBV แบบ manual)
from django.shortcuts import render, get_object_or_404
from django.views import View
from .models import Post


class PostListView(View):
    def get(self, request):
        posts = Post.objects.filter(is_published=True)
        return render(request, 'blog/post_list.html', {'posts': posts})


class PostDetailView(View):
    def get(self, request, slug):
        post = get_object_or_404(Post, slug=slug, is_published=True)
        return render(request, 'blog/post_detail.html', {'post': post})
```

โค้ดนี้ **ทำงานถูกต้องสมบูรณ์** และดีกว่า FBV ในแง่โครงสร้าง (แยก method ตาม HTTP verb ได้
ตาม Part 021 ขั้นตอนที่ 204-206) แต่ถ้าสังเกตดี ๆ จะเห็นว่ารูปแบบ "ดึงข้อมูลทั้งหมด/ดึงข้อมูล
ทีละตัว → render template" นี้ **เกิดซ้ำ ๆ ในแทบทุกแอปของทุกโปรเจกต์ Django** — Django จึง
เตรียม **Generic Class-Based Views** ไว้ให้ทำสิ่งนี้แทนคุณทั้งหมด

### 211.2 ปัญหาของการเขียน View ซ้ำ ๆ แบบนี้

ลองนึกภาพว่าโปรเจกต์มี 10 โมเดลที่ต้องมีหน้า "รายการ" และ "รายละเอียด" (Post, Product,
Event, Job, Course, ...) ถ้าเขียน manual CBV แบบข้างต้นทุกโมเดล คุณจะต้องเขียนโค้ดที่
**โครงสร้างเหมือนกันทุกตัวอักษร** ซ้ำ 10 รอบ ผิดหลัก **DRY** ที่เราย้ำมาตั้งแต่ Part 001

Generic CBV แก้ปัญหานี้โดยการ **สรุปรูปแบบที่ซ้ำกันไว้เป็น class สำเร็จรูป** แล้วให้คุณ
เพียงแค่ **บอกว่าใช้ model ไหน กับ template ไหน** — ที่เหลือ Django ทำให้ทั้งหมด

### 211.3 `ListView`: เขียนโค้ดแทนที่ `PostListView` แบบ manual

```python
# blog/views.py
from django.views.generic import ListView
from .models import Post


class PostListView(ListView):
    model = Post
    template_name = 'blog/post_list.html'
    context_object_name = 'posts'
```

โค้ด 4 บรรทัดนี้ทำงาน **เทียบเท่า** กับ `PostListView(View)` ที่เขียนด้วยมือในขั้นตอนที่
211.1 ทุกประการ (ยกเว้นยังไม่มี filter `is_published=True` ซึ่งจะเพิ่มในขั้นตอนที่ 214)

### 211.4 อธิบาย attribute ทีละตัว

| Attribute | ความหมาย | ค่า default ถ้าไม่ระบุ |
|---|---|---|
| `model` | โมเดลที่ `ListView` จะดึงข้อมูลมาแสดง (ผ่าน `Model.objects.all()`) | ต้องระบุ (หรือ override `get_queryset()` แทน) |
| `template_name` | ชื่อไฟล์ template ที่จะ render | `<app_label>/<model_name>_list.html` (ดูขั้นตอนที่ 211.5) |
| `context_object_name` | ชื่อตัวแปรที่ template จะใช้เข้าถึง queryset | `object_list` (และ `<model_name>_list` ให้ด้วยเสมอ) |
| `queryset` | กำหนด queryset ตรง ๆ แทน `model` (ใช้เมื่อต้องการ custom queryset แบบ static) | ไม่มี |
| `paginate_by` | จำนวนรายการต่อหน้า (เจาะลึกในขั้นตอนที่ 212) | `None` (ไม่แบ่งหน้า) |
| `ordering` | ลำดับการเรียงข้อมูล (ถ้าไม่ระบุใช้ `Meta.ordering` ของ model) | `None` |

### 211.5 ธรรมเนียมการตั้งชื่อ Template อัตโนมัติ

นี่คือจุดที่ทำให้หลายคนตกใจว่า "ทำไม view สั้นจัง" — ถ้า**ไม่ระบุ** `template_name` เลย
Django จะคาดเดาชื่อ template ให้อัตโนมัติตามสูตรนี้:

```
<app_label>/<model_name ตัวพิมพ์เล็ก>_list.html
```

สำหรับ `PostListView` ที่ `model = Post` และอยู่ในแอป `blog` คือ:

```
blog/post_list.html
```

สังเกตว่านี่คือ **ชื่อไฟล์เดียวกันเป๊ะ ๆ** กับ template ที่ Part 007 (ขั้นตอนที่ 65) และ
Part 021 เขียนไว้อยู่แล้ว! นั่นหมายความว่าคุณสามารถลบ `template_name = 'blog/post_list.html'`
ออกจาก `PostListView` ได้เลย โดยที่ทุกอย่างยังทำงานเหมือนเดิมทุกประการ:

```python
# blog/views.py — ไม่ต้องระบุ template_name เพราะตรงกับ convention อยู่แล้ว
class PostListView(ListView):
    model = Post
    context_object_name = 'posts'
```

> **คำแนะนำระดับมืออาชีพ**: แม้ Django จะเดา template ให้ได้ แต่ทีมงานหลายทีมยังคงเขียน
> `template_name` ไว้อย่างชัดเจนเสมอ เพราะทำให้อ่านโค้ดแล้วรู้ทันทีว่า view นี้ render
> ไฟล์ไหน โดยไม่ต้องจำสูตรการตั้งชื่อ — เลือกใช้แนวทางไหนก็ได้ตามธรรมเนียมทีมของคุณ

### 211.6 ธรรมเนียมการตั้งชื่อ Context Variable อัตโนมัติ

ถ้า**ไม่ระบุ** `context_object_name` เลย `ListView` จะส่งข้อมูลเข้า template ผ่าน **สองชื่อ
พร้อมกันเสมอ**:

1. `object_list` — ชื่อทั่วไปที่ใช้ได้กับทุกโมเดล (Django ใช้ชื่อนี้เป็นค่าเริ่มต้นสากล)
2. `<model_name ตัวพิมพ์เล็ก>_list` — ชื่อเฉพาะของโมเดลนั้น เช่น `post_list`

```python
# blog/views.py — ไม่ระบุ context_object_name
class PostListView(ListView):
    model = Post
```

```html
<!-- ใช้ได้ทั้งสองแบบในเทมเพลตเดียวกัน (ผลลัพธ์เหมือนกันทุกประการ) -->
{% for post in object_list %}...{% endfor %}
{% for post in post_list %}...{% endfor %}
```

การระบุ `context_object_name = 'posts'` เองทำให้ template อ่านง่ายขึ้นและสื่อความหมาย
ชัดเจนกว่า `object_list`/`post_list` — เป็นแนวทางที่แนะนำเสมอในหลักสูตรนี้

### 211.7 เชื่อมต่อกับ `urls.py` ด้วย `.as_view()`

ไม่มีอะไรเปลี่ยนจากที่เรียนใน Part 021 — Generic CBV ก็เป็น class เหมือนกัน จึงต้องเรียก
`.as_view()` เสมอเมื่อ map เข้า `urls.py`:

```python
# blog/urls.py
from django.urls import path
from . import views

app_name = 'blog'

urlpatterns = [
    path('', views.PostListView.as_view(), name='list'),
    path('<slug:slug>/', views.PostDetailView.as_view(), name='detail'),
]
```

### 211.8 ตารางเปรียบเทียบ: Manual CBV (Part 021) vs Generic `ListView`

| ประเด็น | Manual CBV (`View`) | Generic `ListView` |
|---|---|---|
| จำนวนบรรทัดโค้ด | 4-6 บรรทัดขึ้นไป (ต้องเขียน `get()` เอง) | 2-4 บรรทัด (ประกาศ attribute เท่านั้น) |
| ต้องเรียก `render()` เอง | ✅ ต้องเขียนเอง | ❌ Django จัดการให้ทั้งหมด |
| Pagination | ต้องเขียนเอง (`Paginator` เอง) | ✅ แค่ตั้ง `paginate_by` (ขั้นตอนที่ 212) |
| ความยืดหยุ่นสำหรับ logic ซับซ้อนมาก ๆ | สูงมาก (ควบคุมทุกบรรทัด) | สูง (override method เฉพาะจุดได้ — ขั้นตอนที่ 214-215) |
| ความเร็วในการเขียนโค้ดสำหรับ CRUD ทั่วไป | ช้ากว่า | **เร็วกว่ามาก** |
| เหมาะกับ | Logic ที่ไม่ตรงกับรูปแบบ list/detail มาตรฐานเลย | Logic แบบ list/detail มาตรฐาน (~80% ของ view ในโปรเจกต์จริง) |

**กฎของหลักสูตรนี้**: เมื่อ view ของคุณทำสิ่งที่ตรงกับรูปแบบ "แสดงรายการ" หรือ "แสดง
รายละเอียด" ให้เริ่มต้นด้วย Generic CBV เสมอ แล้วค่อย override เฉพาะจุดที่ต้องการ
ปรับแต่ง — อย่าเขียน `View` แบบ manual ใหม่ทั้งหมดโดยไม่จำเป็น

---

## ขั้นตอนที่ 212: Pagination ใน ListView ด้วย `paginate_by`

### 212.1 ทำไม Pagination ถึงจำเป็น

ลองจินตนาการว่าบล็อกของคุณมีบทความ 5,000 บทความ ถ้า `PostListView` ดึงมาแสดง **ทั้งหมด**
ในหน้าเดียว จะเกิดปัญหา:

- หน้าเว็บโหลดช้ามาก (ต้อง render HTML 5,000 รายการ)
- ฐานข้อมูลต้องส่งข้อมูลจำนวนมหาศาลออกมาโดยไม่จำเป็น
- ผู้ใช้เลื่อนหาเนื้อหาที่ต้องการไม่สะดวก

**Pagination** คือการแบ่งข้อมูลออกเป็น "หน้า ๆ" (เช่น หน้าละ 10 รายการ) แล้วให้ผู้ใช้กด
ไปหน้าถัดไป/ก่อนหน้าได้

### 212.2 เปิดใช้งาน Pagination ด้วย `paginate_by`

```python
# blog/views.py
from django.views.generic import ListView
from .models import Post


class PostListView(ListView):
    model = Post
    template_name = 'blog/post_list.html'
    context_object_name = 'posts'
    paginate_by = 10   # แสดง 10 บทความต่อหน้า
```

เพียงเพิ่ม 1 บรรทัดนี้ `ListView` จะจัดการทุกอย่างให้อัตโนมัติ:

- อ่าน query parameter `?page=2`, `?page=3`, ... จาก URL เอง
- ตัดข้อมูลให้เหลือเฉพาะหน้าที่ร้องขอ (ใช้ `django.core.paginator.Paginator` ภายใน)
- ถ้า `page` ที่ขอไม่มีอยู่จริง (เช่น `?page=999` แต่มีแค่ 5 หน้า) จะ raise `Http404`
  ให้อัตโนมัติ

### 212.3 ตัวแปรที่ `ListView` ส่งเข้า Context เมื่อเปิด Pagination

เมื่อตั้งค่า `paginate_by` แล้ว context จะมีตัวแปรเพิ่มขึ้นมาอีก 3 ตัวนอกเหนือจาก
`context_object_name` เดิม:

| ตัวแปรใน Context | ชนิดข้อมูล | ความหมาย |
|---|---|---|
| `page_obj` | `Page` (จาก `django.core.paginator`) | ข้อมูลของหน้าปัจจุบัน มี `.number`, `.has_next()`, `.has_previous()`, `.next_page_number()` |
| `is_paginated` | `bool` | `True` ถ้าข้อมูลทั้งหมดมีมากกว่า 1 หน้า (ถูกแบ่งหน้าจริง ๆ) |
| `paginator` | `Paginator` | object หลักที่คุม pagination ทั้งหมด มี `.num_pages`, `.count` |
| `posts` (`context_object_name`) | `QuerySet` (slice แล้ว) | มีเฉพาะรายการของหน้าปัจจุบันเท่านั้น (ไม่ใช่ทั้งหมด) |

### 212.4 เขียน Template แสดงปุ่ม Pagination

```html
<!-- blog/templates/blog/post_list.html -->
<!DOCTYPE html>
<html lang="th">
<head>
    <meta charset="UTF-8">
    <title>บทความทั้งหมด</title>
</head>
<body>
    <h1>บทความทั้งหมด</h1>

    <ul>
        {% for post in posts %}
            <li>
                <a href="{% url 'blog:detail' post.slug %}">{{ post.title }}</a>
                ({{ post.created_at|date:"d/m/Y" }})
            </li>
        {% empty %}
            <li>ยังไม่มีบทความที่เผยแพร่</li>
        {% endfor %}
    </ul>

    {% if is_paginated %}
        <nav aria-label="Pagination">
            {% if page_obj.has_previous %}
                <a href="?page=1">« หน้าแรก</a>
                <a href="?page={{ page_obj.previous_page_number }}">ก่อนหน้า</a>
            {% endif %}

            <span>
                หน้า {{ page_obj.number }} จาก {{ page_obj.paginator.num_pages }}
                (ทั้งหมด {{ paginator.count }} บทความ)
            </span>

            {% if page_obj.has_next %}
                <a href="?page={{ page_obj.next_page_number }}">ถัดไป</a>
                <a href="?page={{ page_obj.paginator.num_pages }}">หน้าสุดท้าย »</a>
            {% endif %}
        </nav>
    {% endif %}
</body>
</html>
```

**สังเกต**: เราตรวจสอบ `{% if is_paginated %}` ก่อนเสมอ เพื่อไม่ให้แสดงปุ่ม pagination
เมื่อข้อมูลมีแค่หน้าเดียว (จะดูรกโดยไม่จำเป็น)

### 212.5 แสดงเลขหน้าแบบเต็ม (1 2 3 4 5 ...)

```html
{% if is_paginated %}
    <nav aria-label="Pagination">
        {% for num in paginator.page_range %}
            {% if num == page_obj.number %}
                <strong>{{ num }}</strong>
            {% else %}
                <a href="?page={{ num }}">{{ num }}</a>
            {% endif %}
        {% endfor %}
    </nav>
{% endif %}
```

> **ข้อควรระวังสำหรับข้อมูลจำนวนมาก**: ถ้ามี 500 หน้า `paginator.page_range` จะสร้างปุ่ม
> 500 ปุ่ม! ในงานจริงควรใช้ `paginator.get_elided_page_range()` (Django 4.0+) ที่ตัดเลขหน้า
> ตรงกลางเป็น `...` ให้อัตโนมัติ เช่น `1 2 3 ... 49 50` — เราจะกลับมาใช้เทคนิคนี้แบบเต็ม
> รูปแบบใน Part 040 (UI/UX สำหรับหน้ารายการข้อมูลขนาดใหญ่)

### 212.6 ปรับแต่ง Pagination เพิ่มเติม: `page_kwarg` และ `ordering`

```python
class PostListView(ListView):
    model = Post
    template_name = 'blog/post_list.html'
    context_object_name = 'posts'
    paginate_by = 10
    page_kwarg = 'p'          # เปลี่ยนจาก ?page=2 เป็น ?p=2
    ordering = ['-created_at', 'title']   # เรียงข้อมูลก่อนแบ่งหน้า (override Meta.ordering)
```

| Attribute | ค่า default | ใช้เมื่อ |
|---|---|---|
| `paginate_by` | `None` | ต้องการเปิด pagination และกำหนดจำนวนรายการต่อหน้า |
| `page_kwarg` | `'page'` | ต้องการเปลี่ยนชื่อ query parameter (พบน้อยในทางปฏิบัติ) |
| `paginate_orphans` | `0` | ป้องกันหน้าสุดท้ายมีรายการน้อยเกินไป (เช่น 1 รายการโดด ๆ) โดยรวมเข้ากับหน้าก่อนหน้าถ้าน้อยกว่าค่านี้ |
| `ordering` | `None` (ใช้ `Meta.ordering` ของ model) | ต้องการเรียงข้อมูลต่างจาก default ของ model เฉพาะใน view นี้ |

### 212.7 ตารางเปรียบเทียบ: Pagination แบบ Manual vs `ListView`

| ประเด็น | เขียนเอง (`Paginator` ใน FBV) | `ListView` + `paginate_by` |
|---|---|---|
| โค้ดที่ต้องเขียนใน view | ~8-10 บรรทัด (import `Paginator`, จัดการ `EmptyPage`, `PageNotAnInteger`) | 1 บรรทัด (`paginate_by = 10`) |
| จัดการ page number ที่ไม่ถูกต้อง | ต้องเขียน try/except เอง | จัดการให้อัตโนมัติ (raise `Http404`) |
| Context variables | ต้องส่งเองทุกตัว | ส่งให้อัตโนมัติครบ (`page_obj`, `is_paginated`, `paginator`) |

---

## ขั้นตอนที่ 213: `DetailView` เบื้องต้น — `get_object()`, `slug_field`, `slug_url_kwarg`

### 213.1 เขียน `PostDetailView` ด้วย Generic CBV

```python
# blog/views.py
from django.views.generic import DetailView
from .models import Post


class PostDetailView(DetailView):
    model = Post
    template_name = 'blog/post_detail.html'
    context_object_name = 'post'
```

```python
# blog/urls.py
path('<slug:slug>/', views.PostDetailView.as_view(), name='detail'),
```

โค้ดนี้ทำงานเทียบเท่ากับ `PostDetailView(View)` แบบ manual ที่เขียนใน Part 021 ทุกประการ
(ยกเว้นยังไม่มี filter `is_published=True` — เพิ่มในขั้นตอนที่ 214)

### 213.2 `get_object()` ทำงานอย่างไรภายใน

หัวใจของ `DetailView` คือ method `get_object()` ซึ่งมี logic (แบบย่อ) ประมาณนี้:

```python
# นี่คือ pseudocode อธิบายแนวคิดของ Django เอง (ไม่ต้องเขียนเอง)
def get_object(self, queryset=None):
    queryset = queryset or self.get_queryset()

    pk = self.kwargs.get(self.pk_url_kwarg)          # ค่า default: 'pk'
    slug = self.kwargs.get(self.slug_url_kwarg)       # ค่า default: 'slug'

    if pk is not None:
        queryset = queryset.filter(pk=pk)
    elif slug is not None:
        slug_field = self.get_slug_field()             # ค่า default: 'slug'
        queryset = queryset.filter(**{slug_field: slug})
    else:
        raise AttributeError('ต้องมี pk หรือ slug ใน URL kwargs')

    try:
        obj = queryset.get()
    except queryset.model.DoesNotExist:
        raise Http404(f'ไม่พบ {queryset.model._meta.verbose_name} ที่ต้องการ')
    return obj
```

พูดง่าย ๆ คือ `DetailView` จะมองหาค่าจาก URL (`pk` หรือ `slug`) แล้วใช้ค่านั้น `filter()`
queryset และ `.get()` object เดียวออกมา ถ้าไม่พบจะ **raise `Http404` ให้อัตโนมัติ** — พฤติกรรม
เดียวกับที่คุณเขียนด้วยตัวเองผ่าน `get_object_or_404()` ใน Part 007 ขั้นตอนที่ 67 ทุกประการ

### 213.3 ตัวเลือกที่ 1: ใช้ `pk` (Primary Key) แทน `slug`

```python
# blog/urls.py
path('<int:pk>/', views.PostDetailView.as_view(), name='detail'),
```

`DetailView` มองหาค่าชื่อ `pk` ใน URL kwargs โดยอัตโนมัติ (ควบคุมด้วย attribute
`pk_url_kwarg = 'pk'`) ไม่ต้องตั้งค่าอะไรเพิ่มถ้าตั้งชื่อ URL parameter ว่า `pk` อยู่แล้ว

### 213.4 ตัวเลือกที่ 2: ใช้ `slug` — ต้องตั้งค่า `slug_field`/`slug_url_kwarg` เมื่อชื่อไม่ตรงกัน

โดย default `DetailView` คาดหวังว่า:

- URL kwarg ชื่อ `slug` (`slug_url_kwarg = 'slug'`)
- field ใน model ชื่อ `slug` (`slug_field = 'slug'`)

เนื่องจาก `Post` model ของเรามี field ชื่อ `slug` อยู่แล้ว และ `urls.py` ใช้
`<slug:slug>` (ชื่อ kwarg คือ `slug` พอดี) จึง**ไม่ต้องตั้งค่าอะไรเพิ่มเลย** แต่ถ้าสมมติ
ว่า URL หรือ field ตั้งชื่อไม่ตรงกัน เช่น:

```python
# blog/urls.py
path('<slug:post_slug>/', views.PostDetailView.as_view(), name='detail'),
```

```python
# blog/models.py — สมมติว่า field ชื่อ url_slug แทน slug
class Post(models.Model):
    url_slug = models.SlugField(max_length=220, unique=True)
    # ...
```

ต้องระบุทั้งสองค่าให้ตรงกับความจริง:

```python
# blog/views.py
class PostDetailView(DetailView):
    model = Post
    slug_field = 'url_slug'         # ชื่อ field ใน model
    slug_url_kwarg = 'post_slug'    # ชื่อ kwarg ใน urls.py
    context_object_name = 'post'
```

### 213.5 ธรรมเนียมการตั้งชื่อ Template และ Context สำหรับ `DetailView`

เหมือนกับ `ListView` ทุกประการในหลักการ:

| สิ่งที่ต้องการ | ค่า default ถ้าไม่ระบุ |
|---|---|
| `template_name` | `<app_label>/<model_name ตัวพิมพ์เล็ก>_detail.html` เช่น `blog/post_detail.html` |
| Context variable | `object` เสมอ **และ** `<model_name ตัวพิมพ์เล็ก>` เช่น `post` |

```html
<!-- ใช้ได้ทั้งสองแบบถ้าไม่ระบุ context_object_name -->
<h1>{{ object.title }}</h1>
<h1>{{ post.title }}</h1>
```

### 213.6 ตารางสรุป Attribute ทั้งหมดของ `DetailView`

| Attribute | ความหมาย | ค่า default |
|---|---|---|
| `model` | โมเดลที่จะดึง object เดียวมาแสดง | ต้องระบุ (หรือ override `get_queryset()`) |
| `queryset` | queryset สำหรับค้นหา object (แทน `model`) | ไม่มี |
| `template_name` | ชื่อไฟล์ template | `<app_label>/<model_name>_detail.html` |
| `context_object_name` | ชื่อตัวแปรใน template | `object` (และ `<model_name>` ให้ด้วย) |
| `pk_url_kwarg` | ชื่อ URL kwarg สำหรับ primary key | `'pk'` |
| `slug_field` | ชื่อ field ใน model ที่ใช้เป็น slug | `'slug'` |
| `slug_url_kwarg` | ชื่อ URL kwarg สำหรับ slug | `'slug'` |
| `query_pk_and_slug` | ถ้า `True` จะ filter ด้วยทั้ง `pk` **และ** `slug` พร้อมกัน (ป้องกัน URL เดาสุ่ม) | `False` |

---

## ขั้นตอนที่ 214: Override `get_queryset()` ใน ListView/DetailView

### 214.1 ปัญหา: `model = Post` เห็นข้อมูลทุกแถวรวมฉบับร่างด้วย

ตอนนี้ `PostListView` และ `PostDetailView` ที่เขียนไว้ยังมีปัญหาสำคัญ: `model = Post` ทำให้
Django ใช้ `Post.objects.all()` เป็น queryset เริ่มต้น ซึ่ง**รวมโพสต์ที่ยังไม่เผยแพร่
(`is_published=False`) ด้วย** ผิดจากพฤติกรรมของ `blog_list`/`blog_detail` ใน Part 007 ที่
กรองเฉพาะ `is_published=True` เสมอ

### 214.2 Override `get_queryset()` ใน `ListView`

```python
# blog/views.py
from django.views.generic import ListView
from .models import Post


class PostListView(ListView):
    model = Post
    template_name = 'blog/post_list.html'
    context_object_name = 'posts'
    paginate_by = 10

    def get_queryset(self):
        return Post.objects.filter(is_published=True)
```

เมื่อ override `get_queryset()` แล้ว **attribute `model` ยังจำเป็นอยู่หรือไม่?** — จำเป็น
เล็กน้อย: Django ยังใช้ `model` เพื่อเดาชื่อ template (`blog/post_list.html`) และชื่อ
context variable (`post_list`) แต่ **ไม่ได้ใช้ `model.objects.all()` อีกต่อไป** เพราะ
`get_queryset()` ที่คุณเขียนเองมี priority สูงกว่าเสมอ

### 214.3 Override `get_queryset()` ใน `DetailView`

```python
# blog/views.py
from django.views.generic import DetailView
from .models import Post


class PostDetailView(DetailView):
    model = Post
    template_name = 'blog/post_detail.html'
    context_object_name = 'post'

    def get_queryset(self):
        return Post.objects.filter(is_published=True)
```

ผลลัพธ์: ถ้าผู้ใช้พยายามเข้าถึง slug ของโพสต์ที่ `is_published=False` โดยตรง (เช่น เดา URL)
`get_object()` จะหาไม่เจอในของ queryset ที่ filter แล้ว และ **raise `Http404` ให้อัตโนมัติ**
— พฤติกรรมเดียวกับ `get_object_or_404(Post, slug=slug, is_published=True)` ใน Part 007
เป๊ะ ๆ แต่เขียนน้อยกว่ามาก

### 214.4 ทำไมต้อง Override `get_queryset()` แทนที่จะ Override `queryset` Attribute ตรง ๆ

Django อนุญาตให้ตั้งค่า `queryset` เป็น attribute ได้เหมือนกัน:

```python
class PostListView(ListView):
    queryset = Post.objects.filter(is_published=True)   # ⚠️ ระวัง!
```

แต่วิธีนี้มี**ข้อเสียร้ายแรง**: `Post.objects.filter(...)` ถูกประเมิน (evaluate) เพียง
**ครั้งเดียวตอนที่ Python โหลดไฟล์ `views.py`** (ตอน server เริ่มทำงาน) แม้ QuerySet จะเป็น
lazy และ query จริงจะยิงใหม่ทุก request ก็ตาม แต่การเขียนแบบนี้เสี่ยงเกิดปัญหาเมื่อ
queryset ต้องพึ่งค่าที่เปลี่ยนแปลงตาม request (เช่น `self.request.user`, วันที่ปัจจุบัน)
ซึ่งยังไม่มีในตอนที่ class ถูกโหลด

**กฎของหลักสูตรนี้**: ให้ override `get_queryset()` เป็น method เสมอ แทนที่จะตั้งค่า
`queryset` เป็น attribute ตรง ๆ เพราะ method จะถูกเรียกใหม่ **ทุกครั้งที่มี request เข้ามา**
ทำให้ปลอดภัยกว่าและรองรับ logic ที่ซับซ้อนขึ้นได้ในอนาคตโดยไม่ต้องแก้โครงสร้าง

### 214.5 รวม Filter และ Ordering เข้าด้วยกัน

```python
class PostListView(ListView):
    model = Post
    template_name = 'blog/post_list.html'
    context_object_name = 'posts'
    paginate_by = 10

    def get_queryset(self):
        return (
            Post.objects
            .filter(is_published=True)
            .select_related('category')       # ป้องกัน N+1 query (Part 012 ขั้นตอนที่ 116)
            .order_by('-created_at')
        )
```

### 214.6 ตารางเปรียบเทียบ: `model`, `queryset` Attribute, และ `get_queryset()` Method

| แนวทาง | ประเมินเมื่อไหร่ | เข้าถึง `self.request` ได้ไหม | คำแนะนำ |
|---|---|---|---|
| `model = Post` | ทุก request (ใช้ `Post.objects.all()`) | ❌ ไม่ผ่านทางนี้ | ใช้เมื่อไม่ต้อง filter อะไรเลย |
| `queryset = Post.objects.filter(...)` | **ครั้งเดียว** ตอนโหลด class | ❌ ไม่ได้ | หลีกเลี่ยงในโปรเจกต์จริง |
| `def get_queryset(self): ...` | ทุก request ใหม่เสมอ | ✅ ได้เต็มที่ | **แนะนำเสมอ** เมื่อต้อง filter/customize |

---

## ขั้นตอนที่ 215: Override `get_context_data()` เพื่อเพิ่มข้อมูลพิเศษเข้า Template

### 215.1 ปัญหา: Sidebar ต้องการรายชื่อหมวดหมู่ทั้งหมด แต่ `ListView` ส่งมาแค่ `posts`

สมมติว่า template `blog/post_list.html` ต้องการแสดง **sidebar รายชื่อหมวดหมู่ทั้งหมด**
เพื่อให้ผู้ใช้กดกรองได้ — แต่ context ที่ `ListView` ส่งมาให้อัตโนมัติมีแค่ `posts`,
`page_obj`, `is_paginated`, `paginator` เท่านั้น ไม่มีข้อมูล `Category` เลย

### 215.2 Override `get_context_data()` เพื่อเพิ่มข้อมูล

```python
# blog/views.py
from django.views.generic import ListView
from .models import Post, Category


class PostListView(ListView):
    model = Post
    template_name = 'blog/post_list.html'
    context_object_name = 'posts'
    paginate_by = 10

    def get_queryset(self):
        return Post.objects.filter(is_published=True).select_related('category')

    def get_context_data(self, **kwargs):
        context = super().get_context_data(**kwargs)   # ⚠️ ต้องเรียกก่อนเสมอ
        context['categories'] = Category.objects.all()
        return context
```

### 215.3 ทำไมต้องเรียก `super().get_context_data(**kwargs)` ก่อนเสมอ

`get_context_data()` ของ `ListView` (และทุก Generic CBV) มีหน้าที่สร้าง context พื้นฐาน
ทั้งหมดที่กล่าวถึงในขั้นตอนที่ 211-213 (เช่น `posts`, `page_obj`, `is_paginated`) ถ้าคุณ
override method นี้โดย **ไม่เรียก `super()` ก่อน** context เหล่านั้นจะ**หายไปทั้งหมด**
และ template จะพังทันที (`posts` จะไม่มีอยู่ใน context เลย)

```python
# ❌ ผิด — context พื้นฐานหายหมด
def get_context_data(self, **kwargs):
    context = {}
    context['categories'] = Category.objects.all()
    return context   # posts, page_obj หายไปหมด!

# ✅ ถูกต้อง — ได้ context เดิมครบ แล้วค่อยเพิ่มของใหม่
def get_context_data(self, **kwargs):
    context = super().get_context_data(**kwargs)
    context['categories'] = Category.objects.all()
    return context
```

### 215.4 ใช้งานใน Template

```html
<!-- blog/templates/blog/post_list.html -->
<div class="sidebar">
    <h3>หมวดหมู่</h3>
    <ul>
        {% for category in categories %}
            <li>
                <a href="{% url 'blog:list' %}?category={{ category.slug }}">
                    {{ category.name }}
                </a>
                ({{ category.posts.count }})
            </li>
        {% empty %}
            <li>ยังไม่มีหมวดหมู่</li>
        {% endfor %}
    </ul>
</div>
```

### 215.5 ใช้ `get_context_data()` กับ `DetailView` เช่นกัน

```python
# blog/views.py
from django.views.generic import DetailView
from .models import Post, Category


class PostDetailView(DetailView):
    model = Post
    template_name = 'blog/post_detail.html'
    context_object_name = 'post'

    def get_queryset(self):
        return Post.objects.filter(is_published=True)

    def get_context_data(self, **kwargs):
        context = super().get_context_data(**kwargs)
        context['categories'] = Category.objects.all()
        context['other_posts'] = (
            Post.objects.filter(is_published=True)
            .exclude(pk=self.object.pk)
            .order_by('-created_at')[:5]
        )
        return context
```

สังเกตว่าเราใช้ `self.object` เพื่ออ้างถึงโพสต์ปัจจุบันที่ `get_object()` หามาให้แล้ว
(`DetailView` เก็บผลลัพธ์ของ `get_object()` ไว้ที่ `self.object` เสมอก่อนเรียก
`get_context_data()`) — เราจะใช้ `self.object` แบบนี้ซ้ำอีกในขั้นตอนที่ 218

### 215.6 ตารางสรุป: Context ที่มีอยู่ ณ จุดที่ `get_context_data()` ถูกเรียก

| Generic CBV | `self.object` มีค่าหรือไม่ | `self.object_list` มีค่าหรือไม่ |
|---|---|---|
| `ListView` | ❌ ไม่มี | ✅ มี (queryset ของหน้าปัจจุบัน) |
| `DetailView` | ✅ มี (object ที่ `get_object()` หาเจอแล้ว) | ❌ ไม่มี |

---

## ขั้นตอนที่ 216: ListView พร้อมระบบค้นหา/กรองผ่าน Query Parameter

### 216.1 เป้าหมาย: รองรับ `?q=...` และ `?category=...`

เราต้องการให้ URL แบบนี้ทำงานได้:

```
/posts/?q=django              → ค้นหาคำว่า "django" ใน title/content
/posts/?category=technology    → กรองเฉพาะหมวดหมู่ที่มี slug "technology"
/posts/?q=django&category=technology   → ใช้ทั้งสองเงื่อนไขพร้อมกัน
```

### 216.2 อ่านค่าจาก `self.request.GET` ใน `get_queryset()`

จุดสำคัญที่ต้องรู้: ภายใน method ของ Generic CBV คุณเข้าถึง request object ปัจจุบันได้
เสมอผ่าน **`self.request`** (Django ตั้งค่านี้ให้อัตโนมัติใน `dispatch()` ก่อนเรียก
method อื่นทุกตัว — เรียนรายละเอียดเต็มใน Part 023 ขั้นตอนที่ 227)

```python
# blog/views.py
from django.db.models import Q
from django.views.generic import ListView
from .models import Post, Category


class PostListView(ListView):
    model = Post
    template_name = 'blog/post_list.html'
    context_object_name = 'posts'
    paginate_by = 10

    def get_queryset(self):
        queryset = Post.objects.filter(is_published=True).select_related('category')

        query = self.request.GET.get('q', '').strip()
        if query:
            queryset = queryset.filter(
                Q(title__icontains=query) | Q(content__icontains=query)
            )

        category_slug = self.request.GET.get('category', '').strip()
        if category_slug:
            queryset = queryset.filter(category__slug=category_slug)

        return queryset.order_by('-created_at')

    def get_context_data(self, **kwargs):
        context = super().get_context_data(**kwargs)
        context['categories'] = Category.objects.all()
        context['query'] = self.request.GET.get('q', '')
        context['selected_category'] = self.request.GET.get('category', '')
        return context
```

### 216.3 เขียน Template ฟอร์มค้นหา

```html
<!-- blog/templates/blog/post_list.html -->
<form method="get">
    <input type="text" name="q" value="{{ query }}" placeholder="ค้นหาบทความ...">

    <select name="category">
        <option value="">ทุกหมวดหมู่</option>
        {% for category in categories %}
            <option value="{{ category.slug }}"
                {% if category.slug == selected_category %}selected{% endif %}>
                {{ category.name }}
            </option>
        {% endfor %}
    </select>

    <button type="submit">ค้นหา</button>
</form>
```

### 216.4 คงค่า Query String เดิมไว้เมื่อกด Pagination

ปัญหาที่พบบ่อยมาก: เมื่อผู้ใช้ค้นหา `?q=django` แล้วกดปุ่ม "หน้าถัดไป" ลิงก์
`?page=2` ธรรมดาจะ**ทำให้ค่าค้นหาหายไป** (กลายเป็นแสดงทุกบทความแทน) ต้องใช้เทคนิค
`QueryDict.urlencode()` ที่เรียนใน Part 007 ขั้นตอนที่ 63.4 มาแก้:

```python
def get_context_data(self, **kwargs):
    context = super().get_context_data(**kwargs)
    context['categories'] = Category.objects.all()
    context['query'] = self.request.GET.get('q', '')
    context['selected_category'] = self.request.GET.get('category', '')

    # เตรียม query string ที่ไม่มี page เพื่อต่อท้ายลิงก์ pagination
    params = self.request.GET.copy()
    params.pop('page', None)
    context['querystring'] = params.urlencode()
    return context
```

```html
{% if is_paginated %}
    <nav>
        {% if page_obj.has_previous %}
            <a href="?page={{ page_obj.previous_page_number }}{% if querystring %}&{{ querystring }}{% endif %}">
                ก่อนหน้า
            </a>
        {% endif %}
        {% if page_obj.has_next %}
            <a href="?page={{ page_obj.next_page_number }}{% if querystring %}&{{ querystring }}{% endif %}">
                ถัดไป
            </a>
        {% endif %}
    </nav>
{% endif %}
```

### 216.5 ตารางสรุปรูปแบบการกรองที่ใช้บ่อย

| รูปแบบการค้นหา/กรอง | Django ORM ที่ใช้ | ตัวอย่าง Query String |
|---|---|---|
| ค้นหาคำในหลาย field | `Q(field1__icontains=x) \| Q(field2__icontains=x)` | `?q=django` |
| กรองด้วยความสัมพันธ์ (FK) | `filter(category__slug=x)` | `?category=technology` |
| กรองช่วงวันที่ | `filter(created_at__year=x, created_at__month=y)` | `?year=2026&month=1` |
| กรองหลายเงื่อนไขพร้อมกัน | เชื่อม `.filter()` ต่อกันเรื่อย ๆ (AND โดยอัตโนมัติ) | `?q=django&category=technology` |

---

## ขั้นตอนที่ 217: การจัดการ 404 ใน DetailView

### 217.1 ทบทวน: `DetailView` Raise `Http404` ให้อัตโนมัติอยู่แล้ว

ตามที่อธิบายในขั้นตอนที่ 213.2 ถ้า `get_object()` หา object ไม่เจอในของ queryset ที่กำหนด
(ไม่ว่าจะเพราะ `pk`/`slug` ผิด หรือถูก `get_queryset()` กรองออกไป) `DetailView` จะ
**`raise Http404` ให้อัตโนมัติ** โดยไม่ต้องเขียนอะไรเพิ่มเลย — พฤติกรรมนี้เทียบเท่ากับการ
เรียก `get_object_or_404()` ด้วยตัวเองใน FBV ทุกประการ

```python
# ทั้งสองแบบนี้ทำงานเหมือนกันทุกประการเมื่อไม่พบข้อมูล
# แบบ FBV (Part 007 ขั้นตอนที่ 67)
def post_detail(request, slug):
    post = get_object_or_404(Post, slug=slug, is_published=True)
    return render(request, 'blog/post_detail.html', {'post': post})

# แบบ DetailView
class PostDetailView(DetailView):
    model = Post
    def get_queryset(self):
        return Post.objects.filter(is_published=True)
```

### 217.2 กรณีที่ต้อง Override `get_object()` เอง: เพิ่มเงื่อนไขพิเศษ

บางครั้ง logic การตัดสินใจว่า "เจอหรือไม่เจอ" ซับซ้อนกว่าการ filter queryset ธรรมดา เช่น
ต้องการให้ **ผู้เขียนบทความเห็นฉบับร่างของตัวเองได้ แต่คนอื่นเห็น 404**:

```python
# blog/views.py
from django.http import Http404
from django.views.generic import DetailView
from .models import Post


class PostDetailView(DetailView):
    model = Post
    template_name = 'blog/post_detail.html'
    context_object_name = 'post'

    def get_object(self, queryset=None):
        # เรียก get_object() ของ parent ก่อน เพื่อใช้ logic ค้นหาด้วย pk/slug เดิม
        # แต่ยกเลิก filter is_published ออกจาก get_queryset() ปกติก่อน
        obj = super().get_object(queryset=Post.objects.all())

        if not obj.is_published:
            # ตัวอย่าง logic: ยอมให้เห็นฉบับร่างเฉพาะ staff เท่านั้น
            # (request.user จะเรียนเต็มรูปแบบใน Phase 4 ตั้งแต่ Part 031)
            if not self.request.user.is_staff:
                raise Http404('ไม่พบบทความที่คุณต้องการ')

        return obj
```

### 217.3 ตัวอย่าง: จำกัดสิทธิ์ด้วยเงื่อนไขทางธุรกิจแทน Authentication

ในกรณีที่ยังไม่มีระบบ login (ก่อนถึง Phase 4) เราอาจต้องการ raise 404 ด้วยเงื่อนไขทาง
ธุรกิจอื่น เช่น "บทความที่ตั้งเวลาเผยแพร่ในอนาคต ยังไม่ควรเห็นแม้จะ `is_published=True`":

```python
from django.utils import timezone
from django.http import Http404
from django.views.generic import DetailView
from .models import Post


class PostDetailView(DetailView):
    model = Post
    template_name = 'blog/post_detail.html'
    context_object_name = 'post'

    def get_object(self, queryset=None):
        obj = super().get_object(queryset=Post.objects.filter(is_published=True))

        if obj.created_at > timezone.now():
            raise Http404('บทความนี้ยังไม่ถึงเวลาเผยแพร่')

        return obj
```

### 217.4 อย่าลืม: `Http404` ต้อง `raise` เสมอ ไม่ใช่ `return`

ทบทวนกฎเหล็กจาก Part 007 ขั้นตอนที่ 62.5: `Http404` เป็น **Exception** ไม่ใช่
`HttpResponse` ต้องเขียน `raise Http404(...)` เท่านั้น — เขียน `return Http404(...)`
จะทำให้ Django error ทันที (`ValueError: HttpResponse content must be...`)

### 217.5 Custom หน้า 404 สำหรับ `DetailView` โดยเฉพาะ

หน้า 404 ที่แสดงเมื่อ `Http404` ถูก raise จาก `DetailView` จะใช้ template เดียวกับที่
ตั้งค่าไว้ใน `handler404` ของโปรเจกต์ (Part 006 ขั้นตอนที่ 58) เพราะ Django จัดการ
exception นี้ที่ระดับ **middleware** ไม่ใช่ที่ระดับ view — ไม่ว่า `Http404` จะถูก raise
จาก FBV, manual CBV, หรือ Generic CBV ก็ตาม จะแสดงหน้า 404 เดียวกันเสมอ นี่คือข้อดีของ
การใช้ `raise Http404` แทนการ `return HttpResponseNotFound` เอง

### 217.6 ตารางสรุป: จุดที่ 404 อาจเกิดขึ้นได้ใน `DetailView`

| สถานการณ์ | เกิด 404 เพราะอะไร |
|---|---|
| `slug`/`pk` ใน URL ไม่ตรงกับข้อมูลใด ๆ ในฐานข้อมูลเลย | `get_object()` เรียก `queryset.get()` แล้วเจอ `DoesNotExist` |
| `slug`/`pk` มีอยู่จริง แต่ถูก `get_queryset()` กรองออก (เช่น `is_published=False`) | queryset หลัง filter ไม่มีแถวนั้นอีกต่อไป |
| Override `get_object()` แล้ว `raise Http404` เองตามเงื่อนไขธุรกิจ | โค้ดที่คุณเขียนเองสั่ง raise ตรง ๆ |
| ระบุทั้ง `pk` และ `slug` พร้อมกับ `query_pk_and_slug = True` แต่ไม่ตรงกันทั้งคู่ | `get_object()` filter ด้วยทั้งสองเงื่อนไข ไม่พบแถวที่ตรงทั้งคู่ |

---

## ขั้นตอนที่ 218: รวมข้อมูลหลายอย่างใน `get_context_data()` ของ DetailView

### 218.1 เป้าหมาย: หน้ารายละเอียดบทความที่สมบูรณ์

หน้า `blog/post_detail.html` ระดับ production ที่แท้จริงมักต้องแสดงมากกว่าแค่เนื้อหา
บทความ ได้แก่:

- คอมเมนต์ทั้งหมดของบทความนี้ (`post.comments`)
- จำนวนคอมเมนต์ทั้งหมด
- บทความอื่นในหมวดหมู่เดียวกัน (related posts)
- รายชื่อหมวดหมู่ทั้งหมดสำหรับ sidebar

### 218.2 เขียน `get_context_data()` แบบครบทุกส่วน

```python
# blog/views.py
from django.views.generic import DetailView
from .models import Post, Category


class PostDetailView(DetailView):
    model = Post
    template_name = 'blog/post_detail.html'
    context_object_name = 'post'

    def get_queryset(self):
        return Post.objects.filter(is_published=True).select_related('category')

    def get_context_data(self, **kwargs):
        context = super().get_context_data(**kwargs)

        # self.object คือ Post ที่ get_object() หาเจอแล้ว (เห็นได้ทุก method หลัง get_object())
        post = self.object

        # 1) คอมเมนต์ทั้งหมดของบทความนี้ เรียงตามเวลาเก่าสุดก่อน (ตาม Meta.ordering ของ Comment)
        comments = post.comments.all()

        # 2) บทความอื่นในหมวดหมู่เดียวกัน (ถ้ามีหมวดหมู่)
        related_posts = Post.objects.none()
        if post.category:
            related_posts = (
                Post.objects.filter(is_published=True, category=post.category)
                .exclude(pk=post.pk)
                .order_by('-created_at')[:5]
            )

        context.update({
            'comments': comments,
            'comment_count': comments.count(),
            'related_posts': related_posts,
            'categories': Category.objects.all(),
        })
        return context
```

### 218.3 เขียน Template แสดงข้อมูลทั้งหมด

```html
<!-- blog/templates/blog/post_detail.html -->
<!DOCTYPE html>
<html lang="th">
<head>
    <meta charset="UTF-8">
    <title>{{ post.title }}</title>
</head>
<body>
    <article>
        <h1>{{ post.title }}</h1>
        <p>
            หมวดหมู่:
            {% if post.category %}{{ post.category.name }}{% else %}ไม่มีหมวดหมู่{% endif %}
            | เผยแพร่เมื่อ {{ post.created_at|date:"d/m/Y H:i" }}
        </p>
        <div>{{ post.content|linebreaks }}</div>
    </article>

    <section>
        <h2>ความคิดเห็น ({{ comment_count }})</h2>
        <ul>
            {% for comment in comments %}
                <li>
                    <strong>{{ comment.author }}</strong>
                    ({{ comment.created_at|date:"d/m/Y H:i" }}):
                    {{ comment.content }}
                </li>
            {% empty %}
                <li>ยังไม่มีความคิดเห็น เป็นคนแรกที่แสดงความเห็นสิ!</li>
            {% endfor %}
        </ul>
    </section>

    {% if related_posts %}
        <aside>
            <h3>บทความที่เกี่ยวข้อง</h3>
            <ul>
                {% for related in related_posts %}
                    <li>
                        <a href="{% url 'blog:detail' related.slug %}">{{ related.title }}</a>
                    </li>
                {% endfor %}
            </ul>
        </aside>
    {% endif %}
</body>
</html>
```

### 218.4 ป้องกัน N+1 Query ด้วย `prefetch_related()`

ถ้าเทมเพลตวนลูป `{% for comment in comments %}` แล้วในแต่ละ comment เข้าถึง field ที่
เป็นความสัมพันธ์เพิ่มเติม (เช่น `comment.post.title` ซ้ำ) ควรใช้ `prefetch_related()`
ตั้งแต่ตอน `get_queryset()` ของ view หลัก (ทบทวนเทคนิคเต็มรูปแบบจาก Part 012 ขั้นตอนที่
116-118):

```python
def get_queryset(self):
    return (
        Post.objects.filter(is_published=True)
        .select_related('category')
        .prefetch_related('comments')
    )
```

### 218.5 ตารางสรุป: ข้อมูลที่ควรอยู่ใน `get_context_data()` ของหน้า Detail ทั่วไป

| ข้อมูล | มาจากไหน | เหตุผลที่ต้องเพิ่มเอง |
|---|---|---|
| Object หลัก (`post`) | `self.object` (Django ใส่ให้อัตโนมัติแล้ว) | ไม่ต้องเพิ่มเอง |
| ข้อมูลที่สัมพันธ์กับ object หลัก (`comments`) | reverse relation ผ่าน `related_name` | ต้องดึงเองเสมอ ไม่มีมาให้อัตโนมัติ |
| ข้อมูล aggregate (`comment_count`) | `.count()` หรือ `annotate()` | Django ไม่รู้ว่าคุณต้องการนับอะไร |
| ข้อมูลสำหรับ navigation ทั้งหน้า (`categories` สำหรับ sidebar) | query แยกจาก object หลัก | ไม่เกี่ยวกับ object โดยตรง ต้องดึงเพิ่มเสมอ |

---

## ขั้นตอนที่ 219: Case Study — แปลง `blog_list`/`blog_detail` เป็น `PostListView`/`PostDetailView` เต็มรูปแบบ

### 219.1 จุดเริ่มต้น: FBV เดิมจาก Part 007

```python
# blog/views.py (Part 007 — ก่อนแปลงเป็น CBV)
from django.shortcuts import render, get_object_or_404
from .models import Post


def blog_list(request):
    posts = Post.objects.filter(is_published=True).order_by('-created_at')
    return render(request, 'blog/post_list.html', {'posts': posts})


def blog_detail(request, slug):
    post = get_object_or_404(Post, slug=slug, is_published=True)
    return render(request, 'blog/post_detail.html', {'post': post})


def blog_archive(request, year, month):
    posts = Post.objects.filter(
        is_published=True,
        created_at__year=year,
        created_at__month=month,
    ).order_by('-created_at')
    return render(request, 'blog/post_archive.html', {
        'posts': posts,
        'year': year,
        'month': month,
    })
```

### 219.2 จุดหมาย: `PostListView`/`PostDetailView` เต็มรูปแบบ

รวมทุกเทคนิคจากขั้นตอนที่ 211-218 เข้าด้วยกันเป็นไฟล์ `blog/views.py` ฉบับสมบูรณ์:

```python
# blog/views.py (หลังแปลงเป็น Generic CBV เต็มรูปแบบ)
from django.db.models import Q
from django.http import Http404
from django.views.generic import ListView, DetailView
from .models import Post, Category


class PostListView(ListView):
    """แทนที่ blog_list — แสดงรายการบทความที่เผยแพร่แล้ว พร้อมค้นหา/กรอง/แบ่งหน้า"""

    model = Post
    template_name = 'blog/post_list.html'
    context_object_name = 'posts'
    paginate_by = 10

    def get_queryset(self):
        queryset = (
            Post.objects.filter(is_published=True)
            .select_related('category')
            .order_by('-created_at')
        )

        query = self.request.GET.get('q', '').strip()
        if query:
            queryset = queryset.filter(
                Q(title__icontains=query) | Q(content__icontains=query)
            )

        category_slug = self.request.GET.get('category', '').strip()
        if category_slug:
            queryset = queryset.filter(category__slug=category_slug)

        return queryset

    def get_context_data(self, **kwargs):
        context = super().get_context_data(**kwargs)
        context['categories'] = Category.objects.all()
        context['query'] = self.request.GET.get('q', '')
        context['selected_category'] = self.request.GET.get('category', '')

        params = self.request.GET.copy()
        params.pop('page', None)
        context['querystring'] = params.urlencode()
        return context


class PostDetailView(DetailView):
    """แทนที่ blog_detail — แสดงบทความเดียว พร้อมคอมเมนต์และบทความที่เกี่ยวข้อง"""

    model = Post
    template_name = 'blog/post_detail.html'
    context_object_name = 'post'

    def get_queryset(self):
        return (
            Post.objects.filter(is_published=True)
            .select_related('category')
            .prefetch_related('comments')
        )

    def get_context_data(self, **kwargs):
        context = super().get_context_data(**kwargs)
        post = self.object

        comments = post.comments.all()
        related_posts = Post.objects.none()
        if post.category:
            related_posts = (
                Post.objects.filter(is_published=True, category=post.category)
                .exclude(pk=post.pk)
                .order_by('-created_at')[:5]
            )

        context.update({
            'comments': comments,
            'comment_count': comments.count(),
            'related_posts': related_posts,
            'categories': Category.objects.all(),
        })
        return context


class PostArchiveView(ListView):
    """แทนที่ blog_archive — แสดงบทความตามปี/เดือนที่ระบุใน URL"""

    model = Post
    template_name = 'blog/post_archive.html'
    context_object_name = 'posts'
    paginate_by = 10

    def get_queryset(self):
        self.year = self.kwargs['year']
        self.month = self.kwargs['month']
        return Post.objects.filter(
            is_published=True,
            created_at__year=self.year,
            created_at__month=self.month,
        ).order_by('-created_at')

    def get_context_data(self, **kwargs):
        context = super().get_context_data(**kwargs)
        context['year'] = self.year
        context['month'] = self.month
        return context
```

> **หมายเหตุเกี่ยวกับ `PostArchiveView`**: Django มี Generic CBV เฉพาะทางสำหรับหน้า
> archive ตามวันที่อยู่แล้ว (`YearArchiveView`, `MonthArchiveView`, `DateDetailView`) แต่
> ต้องอาศัย field ประเภทวันที่และการตั้งค่าเพิ่มเติมหลายจุด เราจะเรียนกลุ่มนี้แบบเต็ม
> รูปแบบใน Part 024 (Generic CBV ขั้นสูง: Date-based views) ตอนนี้ขอใช้ `ListView` ธรรมดา
> พร้อม override `get_queryset()` ไปก่อน เพราะหลักการเบื้องหลังเหมือนกันทุกประการ

### 219.3 อัปเดต `urls.py`

```python
# blog/urls.py
from django.urls import path
from . import views

app_name = 'blog'

urlpatterns = [
    path('', views.PostListView.as_view(), name='list'),
    path('archive/<int:year>/<int:month>/', views.PostArchiveView.as_view(), name='archive'),
    path('<slug:slug>/', views.PostDetailView.as_view(), name='detail'),
]
```

**ข้อควรระวังเรื่องลำดับ URL pattern**: `archive/<int:year>/<int:month>/` ต้องอยู่
**ก่อน** `<slug:slug>/` เสมอ (ทบทวนกฎการจับคู่ URL แบบ "บนลงล่าง, ตัวแรกที่ตรงชนะ" จาก
Part 006 ขั้นตอนที่ 56) ไม่เช่นนั้น URL อย่าง `/posts/archive/` อาจถูกตีความผิดเป็น slug
ชื่อ `archive` แทน

### 219.4 ทดสอบการทำงานด้วย `runserver`

```bash
python manage.py runserver
```

ทดสอบ URL ต่อไปนี้ด้วยเบราว์เซอร์หรือ `curl`:

```bash
curl -s http://127.0.0.1:8000/posts/ | grep -o '<h1>.*</h1>'
curl -s "http://127.0.0.1:8000/posts/?q=django" | grep -o '<title>.*</title>'
curl -s "http://127.0.0.1:8000/posts/?category=technology&page=2"
curl -sI http://127.0.0.1:8000/posts/blog-postt-ไม่มีจริง/   # ควรได้ 404
```

### 219.5 ตารางสรุปการแปลง: FBV → Generic CBV

| ส่วนของโค้ด | FBV (Part 007) | Generic CBV (Part 022) |
|---|---|---|
| ดึงข้อมูลรายการ | เขียน `Post.objects.filter(...)` ใน function ตรง ๆ | override `get_queryset()` |
| Render template | เรียก `render()` เอง 1 บรรทัดท้ายฟังก์ชัน | Django เรียกให้อัตโนมัติจาก `template_name` |
| Pagination | ต้องเขียน `Paginator` เอง (ไม่มีใน Part 007 เดิม) | ตั้งค่า `paginate_by` บรรทัดเดียว |
| 404 เมื่อไม่พบข้อมูล | เรียก `get_object_or_404()` เอง | Override `get_queryset()`/`get_object()` — จัดการให้อัตโนมัติ |
| ข้อมูลเสริมสำหรับ template (sidebar, comments) | เพิ่มเข้า dict context ตรง ๆ ใน `render()` | override `get_context_data()` แล้วเรียก `super()` ก่อน |
| จำนวนบรรทัดโค้ดรวมทั้ง 3 views | ~20 บรรทัด (ยังไม่มี search/pagination) | ~75 บรรทัด (มี search, filter, pagination, related posts, comments ครบ) |

> **ข้อสังเกตสำคัญ**: Generic CBV ไม่ได้ทำให้โค้ด "สั้นกว่าเสมอ" เมื่อเทียบฟีเจอร์ต่อฟีเจอร์
> — ข้อดีที่แท้จริงคือ **โครงสร้างที่สม่ำเสมอและขยายได้ง่าย** เมื่อ requirement เพิ่มขึ้น
> (เพิ่ม pagination, เพิ่ม filter, เพิ่ม context) คุณ override method ที่ถูกจุดแทนที่จะ
> ต้องเขียน `if/else` ซ้อนกันไปเรื่อย ๆ ในฟังก์ชันเดียวแบบ FBV

---

## ขั้นตอนที่ 220: สรุปและแบบฝึกหัด

### 220.1 สรุปสิ่งที่ได้เรียนรู้ใน Part นี้

- ✅ ใช้ `ListView` แสดงรายการข้อมูลแทนการเขียน `render()` เอง พร้อมเข้าใจธรรมเนียมการ
  ตั้งชื่อ template (`<app>/<model>_list.html`) และ context (`object_list`,
  `<model>_list`)
- ✅ เปิดใช้งาน pagination ด้วย `paginate_by` และเข้าใจตัวแปร `page_obj`, `is_paginated`,
  `paginator` ที่ Django ส่งเข้า context ให้อัตโนมัติ
- ✅ ใช้ `DetailView` แสดงข้อมูลรายการเดียว เข้าใจกลไก `get_object()`, `slug_field`,
  `slug_url_kwarg`, และธรรมเนียมชื่อ template (`<app>/<model>_detail.html`)
- ✅ Override `get_queryset()` เพื่อกรองข้อมูล (`is_published=True`) อย่างปลอดภัย แทนการ
  ตั้งค่า `queryset` attribute ตรง ๆ
- ✅ Override `get_context_data()` พร้อมเรียก `super()` ก่อนเสมอ เพื่อเพิ่มข้อมูลพิเศษ
  เข้า template (หมวดหมู่สำหรับ sidebar, คอมเมนต์, บทความที่เกี่ยวข้อง)
- ✅ สร้างระบบค้นหา/กรองผ่าน query parameter (`?q=`, `?category=`) โดยอ่านค่าจาก
  `self.request.GET` และคงค่าไว้ระหว่างเปลี่ยนหน้า pagination
- ✅ จัดการ 404 ใน `DetailView` ทั้งแบบอัตโนมัติและแบบ override `get_object()` เอง
- ✅ แปลง `blog_list`/`blog_detail`/`blog_archive` เต็มรูปแบบเป็น
  `PostListView`/`PostDetailView`/`PostArchiveView` ที่พร้อมใช้งานจริง

### 220.2 Checklist ก่อนไป Part ถัดไป

- [ ] `PostListView` แสดงเฉพาะบทความที่ `is_published=True` และแบ่งหน้าได้ถูกต้อง
- [ ] URL `?page=2`, `?page=999` (ที่ไม่มีจริง) ทำงานตามที่คาดหวัง (หน้า 2 แสดงผล,
      หน้า 999 ได้ 404)
- [ ] `PostDetailView` ใช้ `slug` หาโพสต์ได้ถูกต้อง และคืน 404 เมื่อ slug ไม่ตรงหรือ
      โพสต์เป็นฉบับร่าง
- [ ] Sidebar แสดงรายชื่อหมวดหมู่ทั้งหมดผ่าน `get_context_data()` ใน `PostListView`
- [ ] หน้ารายละเอียดแสดงคอมเมนต์และบทความที่เกี่ยวข้องผ่าน `get_context_data()` ใน
      `PostDetailView`
- [ ] ค้นหาด้วย `?q=...` และกรองด้วย `?category=...` ทำงานพร้อมกันได้ถูกต้อง
- [ ] กดปุ่ม pagination ขณะค้นหาอยู่แล้ว ค่า `q`/`category` ไม่หายไป
- [ ] `blog/urls.py` เรียงลำดับ pattern ถูกต้อง (`archive/...` มาก่อน `<slug:slug>/`)

### 220.3 แบบฝึกหัดท้ายบท

**แบบฝึกหัดที่ 1**: เพิ่ม `ordering` attribute ให้ `PostListView` เพื่อให้รองรับ
`?sort=oldest` (เรียงเก่าไปใหม่) และ `?sort=newest` (ค่าเริ่มต้น เรียงใหม่ไปเก่า) โดย
อ่านค่าจาก `self.request.GET.get('sort')` แล้วปรับ `.order_by()` ใน `get_queryset()`
ให้ตรงกัน พร้อมเพิ่มปุ่มเลือกใน template

**แบบฝึกหัดที่ 2**: สร้าง `CategoryDetailView` ใหม่โดยใช้ `DetailView` กับ `model =
Category` (ใช้ `slug` เป็น URL parameter) แล้ว override `get_context_data()` เพื่อแสดง
"บทความทั้งหมดในหมวดหมู่นี้" พร้อม pagination (คำใบ้: ผสม `DetailView` เข้ากับ
`Paginator` ที่เรียกเองใน `get_context_data()` เพราะ `DetailView` ไม่มี `paginate_by`
ในตัว)

**แบบฝึกหัดที่ 3**: ปรับ `PostDetailView.get_object()` ให้ raise `Http404` แบบกำหนดข้อความ
เอง เมื่อบทความนั้นถูกสร้างมานานเกิน 5 ปี (สมมติเป็นกฎ "บทความเก่าเกินไป ถูกเก็บเข้า
คลังแล้ว") โดยใช้ `django.utils.timezone.now()` เปรียบเทียบกับ `obj.created_at`

**แบบฝึกหัดที่ 4 (ขั้นสูง)**: เขียน mixin ของคุณเอง ชื่อ `PublishedQuerySetMixin` ที่มี
method `get_queryset()` คืนค่า `self.model.objects.filter(is_published=True)` แล้วให้
`PostListView` และ `PostDetailView` inherit จาก mixin นี้ร่วมกับ `ListView`/`DetailView`
เพื่อลดโค้ดซ้ำซ้อนของเงื่อนไข `is_published=True` ที่เขียนซ้ำอยู่ 2 ที่ (คำใบ้: ลำดับ
การ inherit สำคัญมาก — mixin ต้องอยู่**ก่อน** `ListView`/`DetailView` เสมอ เราจะเรียน
เรื่อง Mixin แบบเต็มรูปแบบใน Part 023 ขั้นตอนที่ 226-227)

### 220.4 คำถามที่พบบ่อย (FAQ)

**Q: ต้องใช้ `model` attribute เสมอไหม หรือใช้แค่ `get_queryset()` อย่างเดียวได้?**
A: ใช้ `get_queryset()` อย่างเดียวได้ แต่แนะนำให้เก็บ `model` ไว้ด้วยเสมอ เพราะ Django
ใช้ `model` ในการเดาชื่อ template และชื่อ context variable อัตโนมัติ ถ้าไม่มี `model`
เลยและไม่ระบุ `template_name`/`context_object_name` เอง Django จะหา template ไม่เจอ

**Q: ทำไม `get_context_data()` ต้อง `return context` เสมอ ลืมได้ไหม?**
A: ลืมไม่ได้เด็ดขาด ถ้าลืม `return context` method จะคืนค่า `None` โดยปริยาย (พฤติกรรม
ปกติของ Python) ทำให้ Django error ทันทีว่า context ไม่ใช่ dict — นี่คือหนึ่งใน bug
ที่มือใหม่พลาดบ่อยที่สุดตอนเริ่มเขียน Generic CBV

**Q: `ListView` กับ `DetailView` ใช้ `get()` method ภายในไหม เหมือนที่เรียนใน Part 021?**
A: ใช้ครับ ภายใน `ListView`/`DetailView` มี `get(self, request, *args, **kwargs)` ที่
เรียก `get_queryset()`/`get_object()` แล้วส่งต่อไป `get_context_data()` และสุดท้ายเรียก
`render_to_response()` ให้อัตโนมัติ — โครงสร้างเดียวกับที่คุณเขียนมือใน Part 021 ทุก
ประการ เพียงแต่ Django เขียนให้เสร็จแล้วและแบ่งเป็น method ย่อยที่ override ได้สะดวก

**Q: ควรเลิกใช้ FBV ไปเลยหรือไม่ หลังจากเรียน Generic CBV แล้ว?**
A: ไม่ควร ทั้งสองแบบยังมีที่ใช้งานคู่กันในโปรเจกต์จริงเสมอ Generic CBV เหมาะกับ
รูปแบบ list/detail/create/update/delete มาตรฐาน ส่วน FBV ยังเหมาะกับ view ที่ logic
ไม่ตรงกับรูปแบบมาตรฐานเลย (เช่น webhook, API endpoint เฉพาะกิจ, หน้า dashboard ที่รวม
หลายโมเดลแบบซับซ้อนมาก) — ทีมงานมืออาชีพเลือกใช้ตามความเหมาะสม ไม่ยึดติดแบบใดแบบหนึ่ง

### 220.5 เตรียมตัวสำหรับ Part ถัดไป

**Part 023: Generic CBV ขั้นสูง: CreateView, UpdateView, DeleteView** จะพาไปต่อยอดจาก
`ListView`/`DetailView` ที่เพิ่งเรียนจบ สู่กลุ่ม Generic CBV ที่จัดการการ **สร้าง แก้ไข
และลบข้อมูล** ผ่านฟอร์ม HTML ให้อัตโนมัติ คุณจะได้เรียนรู้ `fields`/`form_class`,
`success_url`, `get_success_url()`, `form_valid()`/`form_invalid()`, และวิธีป้องกันการ
ลบข้อมูลโดยไม่ตั้งใจด้วยหน้ายืนยัน — ทั้งหมดนี้จะทำให้ `Post` ของคุณมีระบบจัดการเนื้อหา
(CRUD) ที่สมบูรณ์เป็นครั้งแรกในหลักสูตร โดยไม่ต้องเขียนฟอร์มหรือ view จัดการข้อมูลเองสัก
บรรทัดเดียว

เตรียมทบทวน Django Forms เบื้องต้น (ถ้ายังไม่คุ้นเคย) และเปิดโปรเจกต์ `blog` ของคุณไว้
ให้พร้อม แล้วไปต่อกันเลย!
