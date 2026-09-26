# Part 023: Generic CBV ขั้นสูง: CreateView, UpdateView, DeleteView

> **ขั้นตอนที่ 221-230 ของหลักสูตร** | Phase 3: Views, Templates, Forms และ CBV
>
> เป้าหมายของ Part นี้: ต่อยอดจาก `ListView` และ `DetailView` ใน Part 022 ไปสู่
> generic CBV ที่จัดการ "การเขียนข้อมูล" (create/update/delete) ให้ครบวงจร คุณจะได้เรียนรู้
> `CreateView`, `UpdateView`, `DeleteView`, `FormView`, การ override `form_valid()`/
> `get_success_url()` แบบมืออาชีพ, เข้าใจ "เบื้องหลัง" ว่า mixin อะไรประกอบกันเป็น
> generic view เหล่านี้ และปิดท้ายด้วยการสร้าง Full CRUD สำหรับ `Post` ด้วย CBV ล้วน ๆ

---

## สารบัญของ Part นี้

- ขั้นตอนที่ 221: `CreateView` เบื้องต้น — `fields` vs `form_class`, `success_url`, และ template convention
- ขั้นตอนที่ 222: `UpdateView` เบื้องต้น — ความคล้ายกับ `CreateView` และ `get_object()`
- ขั้นตอนที่ 223: `DeleteView` เบื้องต้น — หน้ายืนยันการลบและความปลอดภัย
- ขั้นตอนที่ 224: `get_success_url()` แบบ dynamic — redirect ไปหน้า detail ของ object ที่เพิ่งสร้าง/แก้ไข
- ขั้นตอนที่ 225: Override `form_valid()` / `form_invalid()` — ผูก request.user และ logging
- ขั้นตอนที่ 226: `LoginRequiredMixin` เกริ่นสั้น ๆ ก่อนเจาะลึกเต็มใน Part 031-033
- ขั้นตอนที่ 227: `FormView` — generic view สำหรับฟอร์มที่ไม่ผูกกับ Model
- ขั้นตอนที่ 228: เบื้องหลัง `CreateView`/`UpdateView` — `SingleObjectMixin`, `ModelFormMixin`, `ProcessFormView`
- ขั้นตอนที่ 229: เกริ่น `CreateView` + Inline Formset (ก่อนเจาะลึกเต็มใน Part 026)
- ขั้นตอนที่ 230: สรุปและแบบฝึกหัด — สร้าง Full CRUD สำหรับ `Post` ด้วย CBV ทั้งหมด

---

## ขั้นตอนที่ 221: `CreateView` เบื้องต้น — `fields` vs `form_class`, `success_url`, template convention

### 221.1 ทบทวนสถานะปัจจุบันของแอป `blog`

ก่อนเริ่ม Part นี้ ให้ทบทวนว่าจาก Part 022 คุณมี `PostListView` และ `PostDetailView`
พร้อมใช้งานแล้วดังนี้:

```python
# blog/views.py (สถานะจาก Part 022)
from django.views.generic import ListView, DetailView

from .models import Post


class PostListView(ListView):
    model = Post
    template_name = "blog/post_list.html"
    context_object_name = "posts"
    paginate_by = 10

    def get_queryset(self):
        return (
            Post.objects.filter(is_published=True)
            .select_related("category")
            .prefetch_related("tags")
        )


class PostDetailView(DetailView):
    model = Post
    template_name = "blog/post_detail.html"
    context_object_name = "post"
    slug_field = "slug"
    slug_url_kwarg = "slug"
```

```python
# blog/urls.py (สถานะจาก Part 022)
from django.urls import path

from . import views

app_name = "blog"

urlpatterns = [
    path("", views.PostListView.as_view(), name="list"),
    path("<slug:slug>/", views.PostDetailView.as_view(), name="detail"),
]
```

สองตัวนี้จัดการฝั่ง "อ่านข้อมูล" (Read) ได้ครบแล้ว Part นี้เราจะเติมฝั่ง "เขียนข้อมูล"
(Create, Update, Delete) ให้ครบเป็น CRUD เต็มรูปแบบ โดยไม่ต้องเขียน `def post(request)`
ยาว ๆ แบบ Function-Based View อีกต่อไป

### 221.2 `CreateView` คืออะไร

`CreateView` คือ generic class-based view ที่ Django เตรียมไว้สำหรับ "สร้าง object ใหม่
ผ่านฟอร์ม" มันรวมงาน 3 อย่างที่คุณเคยเขียนเองด้วยมือใน Function-Based View (Part 007)
ไว้ในคลาสเดียว:

1. แสดงฟอร์มเปล่าเมื่อ request เป็น `GET`
2. รับข้อมูลจากฟอร์มเมื่อ request เป็น `POST`, validate, และสร้าง object ใหม่ถ้าข้อมูลถูกต้อง
3. Redirect ไปหน้าอื่นเมื่อสร้างสำเร็จ หรือแสดงฟอร์มพร้อม error ถ้าข้อมูลผิด

ตัวอย่างที่ง่ายที่สุดที่ใช้งานได้จริง:

```python
# blog/views.py
from django.urls import reverse_lazy
from django.views.generic import CreateView, ListView, DetailView

from .models import Post


class PostCreateView(CreateView):
    model = Post
    fields = ["title", "content", "category", "tags", "is_published"]
    success_url = reverse_lazy("blog:list")
```

โค้ดแค่ 4 บรรทัดนี้ทำงานเทียบเท่ากับ function-based view ที่ต้องเขียนฟอร์ม, จัดการ
`request.method`, เรียก `form.is_valid()`, `form.save()`, และ `redirect()` เอง — ทั้งหมด
ถูกจัดการให้โดยอัตโนมัติ

### 221.3 `fields` vs `form_class`: สองวิธีในการบอกว่าฟอร์มหน้าตาเป็นอย่างไร

`CreateView` ต้องรู้ว่าจะสร้างฟอร์มจากอะไร มีสองทางเลือก:

| แนวทาง | วิธีเขียน | เหมาะกับ |
|---|---|---|
| `fields` | ระบุ list ชื่อ field ตรง ๆ ใน view | ฟอร์มง่าย ๆ ไม่มี validation พิเศษ ต้นแบบเร็ว (prototype) |
| `form_class` | สร้าง `ModelForm` แยกไฟล์ `forms.py` แล้วชี้มาที่ view | ฟอร์มที่ต้องการ custom validation, custom widget, หรือ field เพิ่มเติมที่ไม่ได้อยู่ใน model |

**แบบที่ 1: ใช้ `fields`** (เหมาะกับต้นแบบเร็ว หรือฟอร์มที่ไม่ซับซ้อน)

```python
class PostCreateView(CreateView):
    model = Post
    fields = ["title", "content", "category", "tags", "is_published"]
    success_url = reverse_lazy("blog:list")
```

Django จะสร้าง `ModelForm` ให้อัตโนมัติแบบเดียวกับที่ `modelform_factory()` ทำ (เราจะเรียน
`ModelForm` แบบเต็มรูปแบบใน Part 026) โดยใช้ widget เริ่มต้นของแต่ละ field type

**แบบที่ 2: ใช้ `form_class`** (แนะนำสำหรับงานจริงระดับมืออาชีพ)

```python
# blog/forms.py
from django import forms

from .models import Post


class PostForm(forms.ModelForm):
    class Meta:
        model = Post
        fields = ["title", "content", "category", "tags", "is_published"]
        widgets = {
            "content": forms.Textarea(attrs={"rows": 10, "class": "form-control"}),
            "title": forms.TextInput(attrs={"class": "form-control"}),
        }
```

```python
# blog/views.py
from django.urls import reverse_lazy
from django.views.generic import CreateView

from .forms import PostForm
from .models import Post


class PostCreateView(CreateView):
    model = Post
    form_class = PostForm
    success_url = reverse_lazy("blog:list")
```

> **กฎสำคัญ**: `fields` และ `form_class` **ใช้พร้อมกันไม่ได้** ถ้าใส่ทั้งคู่ Django จะโยน
> `ImproperlyConfigured` ทันทีตอนรัน เพราะทั้งสองทำหน้าที่เดียวกันคือ "กำหนดว่าฟอร์มหน้าตา
> เป็นอย่างไร" ต้องเลือกอย่างใดอย่างหนึ่ง

### 221.4 ทำไมควรใช้ `form_class` มากกว่า `fields` ในงานจริง

| ประเด็น | `fields` | `form_class` |
|---|---|---|
| Custom widget (เช่น `Textarea` ที่มี `rows`) | ทำไม่ได้ตรง ๆ ต้อง override `get_form()` | กำหนดใน `Meta.widgets` ได้เลย |
| Custom validation (`clean_title()`) | ทำไม่ได้เลย | เขียนใน form class ได้อิสระ |
| Field ที่ไม่ได้อยู่ใน model (เช่น checkbox "ฉันยอมรับเงื่อนไข") | ทำไม่ได้ | เพิ่ม field ปกติใน form class ได้ |
| นำฟอร์มเดิมไปใช้ที่อื่น (เช่น admin form, API form) | ทำไม่ได้ (ผูกกับ view) | นำ class ไป import ใช้ที่ไหนก็ได้ |
| ความเร็วในการเขียนต้นแบบ | เร็วกว่า (บรรทัดเดียว) | ต้องสร้างไฟล์/class เพิ่ม |

หลักสูตรนี้จะสอน `ModelForm` แบบละเอียดใน Part 026 แต่ตั้งแต่ตอนนี้ขอแนะนำให้คุณ
**เริ่มใช้ `form_class` เป็นนิสัย** แม้ฟอร์มจะยังง่ายอยู่ก็ตาม เพราะเมื่อโปรเจกต์โตขึ้น
คุณมักต้องเพิ่ม validation หรือ widget แบบกำหนดเองอยู่เสมอ

### 221.5 `success_url`: ต้องมีเสมอ (หรือมี `get_absolute_url()` ทดแทน)

`CreateView` ต้องรู้ว่าเมื่อสร้าง object สำเร็จแล้วจะ redirect ผู้ใช้ไปที่ไหน มี 2 ทาง:

**ทางที่ 1: กำหนด `success_url` ตรง ๆ**

```python
class PostCreateView(CreateView):
    model = Post
    form_class = PostForm
    success_url = reverse_lazy("blog:list")
```

> **ทำไมต้องใช้ `reverse_lazy()` ไม่ใช่ `reverse()`?** เพราะ `success_url` เป็น
> **class attribute** ที่ถูกประเมินค่าตอนไฟล์ `views.py` ถูก import (คือตอนที่ Django
> โหลดแอปขึ้นมา) ซึ่งเป็นช่วงเวลาที่ `urls.py` **อาจยังโหลดไม่เสร็จ** การเรียก `reverse()`
> ตรง ๆ ตอนนั้นจะทำให้เกิด `NoReverseMatch` เพราะ URLconf ยังไม่พร้อม ส่วน `reverse_lazy()`
> จะคืนค่าเป็น "lazy object" ที่ยังไม่ประเมินค่าจริงจนกว่าจะถูกใช้งานจริง (ตอน request
> เข้ามา ซึ่ง URLconf โหลดเสร็จแล้วแน่นอน)

**ทางที่ 2: ไม่กำหนด `success_url` เลย แต่ให้ model มี `get_absolute_url()`**

```python
# blog/models.py
from django.urls import reverse


class Post(models.Model):
    # ... fields เดิม ...

    def get_absolute_url(self):
        return reverse("blog:detail", kwargs={"slug": self.slug})
```

```python
class PostCreateView(CreateView):
    model = Post
    form_class = PostForm
    # ไม่ต้องมี success_url เลย!
```

ถ้าไม่กำหนด `success_url`, Django จะไปเรียก `self.object.get_absolute_url()` ให้
อัตโนมัติ (`self.object` คือ instance ที่เพิ่งสร้างเสร็จ) ถ้า model ไม่มี
`get_absolute_url()` และไม่มี `success_url` ด้วย จะได้ `ImproperlyConfigured`:

```
ImproperlyConfigured: No URL to redirect to. Either provide a url or define
a get_absolute_url method on the Model.
```

### 221.6 Template Convention: `<app>/<model>_form.html`

ถ้าไม่กำหนด `template_name`, `CreateView` จะมองหา template ตามชื่อ **อัตโนมัติ**
ด้วยรูปแบบ:

```
<app_label>/<model_name>_form.html
```

สำหรับ `Post` ในแอป `blog` คือ `blog/post_form.html` — สังเกตว่า **ทั้ง `CreateView`
และ `UpdateView` ใช้ชื่อ template เดียวกัน** (`<model>_form.html` ไม่ใช่ `_create.html`
หรือ `_update.html`) นี่คือ convention ที่ Django ตั้งใจออกแบบให้ใช้ template เดียว
ซ้ำกันสำหรับทั้งสร้างและแก้ไข เพราะหน้าตาฟอร์มมักเหมือนกันทุกประการ

สร้าง template:

```html
<!-- blog/templates/blog/post_form.html -->
{% extends "base.html" %}

{% block title %}{% if object %}แก้ไขบทความ{% else %}เขียนบทความใหม่{% endif %}{% endblock %}

{% block content %}
<h1>{% if object %}แก้ไขบทความ: {{ object.title }}{% else %}เขียนบทความใหม่{% endif %}</h1>

<form method="post" novalidate>
    {% csrf_token %}
    {{ form.as_p }}
    <button type="submit">บันทึก</button>
</form>
{% endblock %}
```

สังเกต 2 จุดสำคัญ:

- `{% csrf_token %}` **ต้องมีเสมอ** สำหรับฟอร์มที่ใช้ `method="post"` มิฉะนั้น Django
  จะปฏิเสธ request ด้วย `403 Forbidden` (CSRF protection ที่เปิดใช้งานโดยค่าเริ่มต้น
  ตามที่เรียนไปใน Part 001)
- `{{ object }}` คือตัวแปรที่ `CreateView`/`UpdateView` ส่งเข้า context ให้อัตโนมัติ
  (เป็นชื่อ context variable มาตรฐานจาก `SingleObjectMixin` — จะอธิบายลึกใน ขั้นตอนที่ 228)
  ตอน `CreateView` แสดงฟอร์มเปล่า `object` จะเป็น `None` เสมอ ส่วนตอน `UpdateView`
  `object` จะเป็น instance ที่กำลังแก้ไข

### 221.7 เชื่อม URL และทดสอบ

```python
# blog/urls.py
from django.urls import path

from . import views

app_name = "blog"

urlpatterns = [
    path("", views.PostListView.as_view(), name="list"),
    path("new/", views.PostCreateView.as_view(), name="create"),
    path("<slug:slug>/", views.PostDetailView.as_view(), name="detail"),
]
```

> **ข้อควรระวังเรื่องลำดับ URL**: `path("new/", ...)` ต้องอยู่ **ก่อน**
> `path("<slug:slug>/", ...)` เสมอ ไม่เช่นนั้น Django จะจับคู่ `/new/` เข้ากับ pattern
> `<slug:slug>/` ก่อน (โดยตีความว่า `slug == "new"`) แล้วพยายามหาโพสต์ที่มี slug ว่า
> `"new"` ซึ่งจะได้ `404` แทนที่จะเปิดฟอร์มสร้างโพสต์ — นี่คือกับดักคลาสสิกที่มือใหม่
> เจอบ่อยมาก (ทบทวนหลักการ "จากบนลงล่าง" จาก Part 006 ขั้นตอนที่ 52)

รันเซิร์ฟเวอร์แล้วเปิด `http://127.0.0.1:8000/new/` คุณควรเห็นฟอร์มสร้างโพสต์ กรอกข้อมูล
แล้วกด "บันทึก" ควรถูก redirect กลับไปหน้ารายการโพสต์ (`blog:list`) พร้อมโพสต์ใหม่
ปรากฏในรายการ

### 221.8 GET vs POST: `CreateView` ทำอะไรบ้างเบื้องหลัง

| Request | สิ่งที่ `CreateView` ทำ |
|---|---|
| `GET /new/` | สร้างฟอร์มเปล่า (unbound form) → render `post_form.html` |
| `POST /new/` (ข้อมูลถูกต้อง) | validate → `form.save()` → สร้าง `Post` ใหม่ในฐานข้อมูล → redirect ไป `success_url` |
| `POST /new/` (ข้อมูลผิด) | validate ไม่ผ่าน → render `post_form.html` เดิมพร้อม error message ใน `form.errors` (สถานะ HTTP 200 ไม่ใช่ redirect) |

---

## ขั้นตอนที่ 222: `UpdateView` เบื้องต้น — ความคล้ายกับ `CreateView` และ `get_object()`

### 222.1 `UpdateView` แทบจะเป็นฝาแฝดของ `CreateView`

`UpdateView` ทำงานเหมือน `CreateView` เกือบทุกอย่าง ต่างกันแค่จุดเดียว:
**`CreateView` สร้าง instance ใหม่เปล่า ๆ ส่วน `UpdateView` โหลด instance ที่มีอยู่แล้ว
จากฐานข้อมูลมาใส่ในฟอร์มก่อน**

```python
# blog/views.py
from django.views.generic import CreateView, UpdateView

from .forms import PostForm
from .models import Post


class PostCreateView(CreateView):
    model = Post
    form_class = PostForm
    success_url = reverse_lazy("blog:list")


class PostUpdateView(UpdateView):
    model = Post
    form_class = PostForm
    slug_field = "slug"
    slug_url_kwarg = "slug"
    success_url = reverse_lazy("blog:list")
```

```python
# blog/urls.py
urlpatterns = [
    path("", views.PostListView.as_view(), name="list"),
    path("new/", views.PostCreateView.as_view(), name="create"),
    path("<slug:slug>/", views.PostDetailView.as_view(), name="detail"),
    path("<slug:slug>/edit/", views.PostUpdateView.as_view(), name="update"),
]
```

### 222.2 `get_object()`: หัวใจที่ทำให้ `UpdateView` รู้ว่าจะแก้ไข object ไหน

`get_object()` คือ method ที่ `UpdateView` (และ `DetailView`) ใช้เพื่อหา instance ที่
ถูกต้องจากค่าที่มากับ URL คุณเคยเห็น `slug_field`/`slug_url_kwarg` มาแล้วใน `DetailView`
(Part 022) และมันทำงานแบบเดียวกันเป๊ะใน `UpdateView` เพราะทั้งคู่สืบทอดมาจาก
`SingleObjectMixin` ตัวเดียวกัน (จะเจาะลึกใน ขั้นตอนที่ 228)

ลำดับการทำงานของ `get_object()` แบบง่าย (เทียบเท่าโค้ดจริงใน Django):

```python
# นี่คือสิ่งที่ SingleObjectMixin.get_object() ทำภายใน (โค้ดย่อเพื่อความเข้าใจ)
def get_object(self, queryset=None):
    if queryset is None:
        queryset = self.get_queryset()          # ปกติคือ Post.objects.all()

    slug = self.kwargs.get(self.slug_url_kwarg)  # ดึงค่าจาก URL เช่น "my-first-post"
    pk = self.kwargs.get(self.pk_url_kwarg)       # หรือดึง pk ถ้า URL ใช้ <int:pk>

    if pk is not None:
        queryset = queryset.filter(pk=pk)
    elif slug is not None:
        queryset = queryset.filter(**{self.slug_field: slug})
    else:
        raise AttributeError("ต้องมี pk หรือ slug ใน URL")

    try:
        return queryset.get()   # ใช้ get_object_or_404 ในทางปฏิบัติ
    except queryset.model.DoesNotExist:
        raise Http404("ไม่พบ object ที่ต้องการ")
```

ดังนั้นเมื่อผู้ใช้เข้า `/my-first-post/edit/`:

1. Django จับคู่ URL pattern `<slug:slug>/edit/` ได้ `slug = "my-first-post"`
2. `UpdateView` เรียก `get_object()` ซึ่งไปหาแถวในตาราง `Post` ที่ `slug == "my-first-post"`
3. ถ้าเจอ → นำข้อมูลนั้นมาสร้างเป็น **bound form** (ฟอร์มที่มีค่าเริ่มต้นเติมไว้แล้ว)
4. ถ้าไม่เจอ → คืน `Http404` อัตโนมัติ (เหมือน `get_object_or_404` ที่คุณเคยใช้ใน
   Function-Based View)

### 222.3 กำหนด `queryset` เพื่อจำกัดสิทธิ์การแก้ไข

จุดที่มือใหม่มักพลาดคือ: ถ้าไม่จำกัด `queryset`, ผู้ใช้จะสามารถแก้ไข **โพสต์ไหนก็ได้ในระบบ**
แค่รู้ slug ตรง ๆ จาก URL วิธีป้องกันเบื้องต้น (ก่อนที่เราจะเรียนเรื่อง permission
เต็มรูปแบบใน Part 033) คือ override `get_queryset()`:

```python
class PostUpdateView(UpdateView):
    model = Post
    form_class = PostForm
    slug_field = "slug"
    slug_url_kwarg = "slug"
    success_url = reverse_lazy("blog:list")

    def get_queryset(self):
        # จำกัดให้แก้ไขได้เฉพาะโพสต์ของตัวเอง (ต้องมี field author ก่อน — ดู ขั้นตอนที่ 225)
        return Post.objects.filter(author=self.request.user)
```

ถ้า `get_object()` หา object จาก queryset ที่ถูกกรองแล้วไม่เจอ (เช่น พยายามแก้ไขโพสต์
ของคนอื่น) ผลลัพธ์คือ `404 Not Found` แทนที่จะเป็น "no permission" ตรง ๆ — ซึ่งเป็นแนวทาง
ที่ปลอดภัยกว่า เพราะไม่บอกผู้โจมตีว่า "object นี้มีอยู่จริงแต่คุณไม่มีสิทธิ์" (เทคนิคนี้
เรียกว่า **object-level permission ผ่าน queryset filtering** เราจะเรียนแบบเต็มรูปแบบ
ใน Part 038)

### 222.4 ตารางเปรียบเทียบ `CreateView` กับ `UpdateView`

| คุณสมบัติ | `CreateView` | `UpdateView` |
|---|---|---|
| ต้องการ pk/slug ใน URL หรือไม่ | ไม่ต้องการ | ต้องการ (เพื่อหา object เดิม) |
| `self.object` ตอน `GET` | `None` | Instance ที่โหลดมาจากฐานข้อมูล |
| ฟอร์มเริ่มต้น | ว่างเปล่า (unbound, ไม่มีค่าเริ่มต้น) | มีค่าเริ่มต้นจาก instance เดิม (bound to instance) |
| Template convention | `<model>_form.html` | `<model>_form.html` (เหมือนกัน!) |
| HTTP method ที่ใช้บันทึก | `POST` | `POST` (Django ไม่รองรับ `PUT`/`PATCH` ใน HTML form โดยตรง) |
| Mixin หลักที่ใช้ | `ModelFormMixin` + `ProcessFormView` | `SingleObjectMixin` + `ModelFormMixin` + `ProcessFormView` |
| SQL ที่เกิดขึ้นตอนบันทึก | `INSERT INTO blog_post ...` | `UPDATE blog_post SET ... WHERE id = ...` |

### 222.5 การแยกความแตกต่างในหน้าเว็บด้วย `{% if object %}`

เนื่องจากทั้งสอง view ใช้ template เดียวกัน คุณสามารถปรับข้อความในหน้าให้ต่างกันได้
โดยเช็คว่า `object` มีค่าหรือไม่ (ตามที่แสดงใน ขั้นตอนที่ 221.6):

```html
<h1>{% if object %}แก้ไขบทความ: {{ object.title }}{% else %}เขียนบทความใหม่{% endif %}</h1>
```

---

## ขั้นตอนที่ 223: `DeleteView` เบื้องต้น — หน้ายืนยันการลบและความปลอดภัย

### 223.1 `DeleteView` คืออะไร

`DeleteView` คือ generic view สำหรับลบ object โดยมีขั้นตอนความปลอดภัยพื้นฐานในตัว:
**ต้องแสดงหน้ายืนยันก่อนเสมอ** — ไม่ลบทันทีที่คลิกลิงก์ ป้องกันอุบัติเหตุ (เช่น
web crawler ไล่ตามลิงก์ทุกอันในหน้าเว็บ ถ้าลิงก์ลบเป็น `GET` ธรรมดา ข้อมูลจะหายหมดโดย
ไม่ได้ตั้งใจ!)

```python
# blog/views.py
from django.urls import reverse_lazy
from django.views.generic import DeleteView

from .models import Post


class PostDeleteView(DeleteView):
    model = Post
    slug_field = "slug"
    slug_url_kwarg = "slug"
    success_url = reverse_lazy("blog:list")
```

```python
# blog/urls.py
urlpatterns = [
    path("", views.PostListView.as_view(), name="list"),
    path("new/", views.PostCreateView.as_view(), name="create"),
    path("<slug:slug>/", views.PostDetailView.as_view(), name="detail"),
    path("<slug:slug>/edit/", views.PostUpdateView.as_view(), name="update"),
    path("<slug:slug>/delete/", views.PostDeleteView.as_view(), name="delete"),
]
```

### 223.2 วงจรการทำงานของ `DeleteView`

| Request | สิ่งที่เกิดขึ้น |
|---|---|
| `GET /my-first-post/delete/` | `get_object()` หาโพสต์ → render หน้ายืนยัน (`<model>_confirm_delete.html`) โดย **ยังไม่ลบข้อมูล** |
| `POST /my-first-post/delete/` | `get_object()` หาโพสต์ → เรียก `object.delete()` → redirect ไป `success_url` |

สังเกตว่า `DeleteView` **ไม่มี** `DELETE` HTTP method involve เลย มันใช้แค่ `GET`
(แสดงหน้ายืนยัน) กับ `POST` (ยืนยันแล้วลบจริง) ซึ่งเป็นสองอย่างที่ HTML `<form>`
รองรับตามธรรมชาติ โดยไม่ต้องพึ่ง JavaScript

### 223.3 Template Convention: `<app>/<model>_confirm_delete.html`

```html
<!-- blog/templates/blog/post_confirm_delete.html -->
{% extends "base.html" %}

{% block title %}ยืนยันการลบบทความ{% endblock %}

{% block content %}
<h1>ยืนยันการลบบทความ</h1>

<p>คุณแน่ใจหรือไม่ว่าต้องการลบบทความ "<strong>{{ object.title }}</strong>"?
   การกระทำนี้ไม่สามารถย้อนกลับได้!</p>

<form method="post">
    {% csrf_token %}
    <button type="submit" class="btn-danger">ยืนยันการลบ</button>
    <a href="{% url 'blog:detail' object.slug %}">ยกเลิก</a>
</form>
{% endblock %}
```

### 223.4 ทำไม `{% csrf_token %}` สำคัญเป็นพิเศษสำหรับการลบ

ถ้าไม่มี CSRF protection ผู้โจมตีสามารถหลอกให้ผู้ใช้ที่ login อยู่กดปุ่ม หรือแม้แต่โหลด
หน้าเว็บที่มี `<img>` ซ่อนไว้ชี้ไปยัง URL ลบข้อมูล แล้วลบข้อมูลแทนผู้ใช้โดยไม่รู้ตัว —
นี่คือการโจมตีแบบ **CSRF (Cross-Site Request Forgery)** ที่ Django ป้องกันให้อัตโนมัติ
ตราบใดที่คุณ:

1. ใช้ `method="post"` สำหรับ action ที่เปลี่ยนแปลงข้อมูล (ไม่ใช่ `GET` ที่คลิกแล้วลบทันที)
2. ใส่ `{% csrf_token %}` ในทุกฟอร์มที่เป็น `POST`

> **ข้อควรระวังที่มือใหม่ทำผิดบ่อยที่สุด**: การทำลิงก์ `<a href="/posts/1/delete/">ลบ</a>`
> ตรง ๆ โดยไม่ผ่านฟอร์ม `POST` เพราะ `<a>` ทำได้แค่ `GET` เท่านั้น ซึ่งจะทำให้ทั้ง
> CSRF protection ใช้ไม่ได้ และเสี่ยงต่อการถูก crawler/prefetch ลบข้อมูลโดยไม่ตั้งใจ
> **`DeleteView` เองก็รองรับเฉพาะ `POST` สำหรับการลบจริง** เพื่อป้องกันปัญหานี้
> โดยธรรมชาติอยู่แล้ว ตราบใดที่คุณใช้ `<form method="post">` ตามตัวอย่างข้างต้น

### 223.5 เพิ่มลิงก์ไปหน้าแก้ไข/ลบในหน้ารายละเอียด

```html
<!-- blog/templates/blog/post_detail.html (เพิ่มเติมจาก Part 022) -->
{% extends "base.html" %}

{% block content %}
<article>
    <h1>{{ post.title }}</h1>
    <p>{{ post.content }}</p>
</article>

<div class="post-actions">
    <a href="{% url 'blog:update' post.slug %}">แก้ไข</a>
    <a href="{% url 'blog:delete' post.slug %}">ลบ</a>
</div>
{% endblock %}
```

---

## ขั้นตอนที่ 224: `get_success_url()` แบบ dynamic — redirect ไปหน้า detail ของ object ที่เพิ่งสร้าง/แก้ไข

### 224.1 ปัญหาของ `success_url` แบบ static

`success_url = reverse_lazy("blog:list")` เหมาะกับกรณีที่ redirect ไปที่เดิมเสมอ
(เช่น หน้ารายการ) แต่ถ้าคุณต้องการ redirect ไป **หน้ารายละเอียดของ object ที่เพิ่งสร้าง
หรือแก้ไข** ซึ่ง URL ขึ้นกับ `slug`/`pk` ของ object นั้น — `success_url` แบบ static
ทำไม่ได้ เพราะมันถูกกำหนดตอน class ถูกโหลด (ก่อนรู้ด้วยซ้ำว่า object ไหนจะถูกสร้าง)

### 224.2 ทางออก: override `get_success_url()`

`get_success_url()` คือ method version ของ `success_url` ที่ถูกเรียก **หลังจาก**
`self.object` ถูกสร้าง/อัปเดตเรียบร้อยแล้ว จึงสามารถใช้ `self.object` ในการคำนวณ URL
ปลายทางได้:

```python
# blog/views.py
from django.urls import reverse
from django.views.generic import CreateView, UpdateView

from .forms import PostForm
from .models import Post


class PostCreateView(CreateView):
    model = Post
    form_class = PostForm

    def get_success_url(self):
        return reverse("blog:detail", kwargs={"slug": self.object.slug})


class PostUpdateView(UpdateView):
    model = Post
    form_class = PostForm
    slug_field = "slug"
    slug_url_kwarg = "slug"

    def get_success_url(self):
        return reverse("blog:detail", kwargs={"slug": self.object.slug})
```

สังเกตว่าใน `get_success_url()` เราใช้ `reverse()` ธรรมดา **ไม่ใช่** `reverse_lazy()`
เพราะ method นี้ถูกเรียกตอน request จริงกำลังประมวลผลอยู่ (URLconf โหลดเสร็จแล้วแน่นอน)
ไม่ใช่ตอนโหลด class เหมือน `success_url` — กฎคือ **`reverse_lazy()` ใช้กับ class
attribute เท่านั้น ส่วนใน method ให้ใช้ `reverse()` ธรรมดาเสมอ**

### 224.3 ลด duplication ด้วย `get_absolute_url()`

เนื่องจากทั้ง `PostCreateView` และ `PostUpdateView` ต้องการ logic เดียวกันทุกประการ
(redirect ไป detail page ของ `self.object`) วิธีที่สะอาดกว่าคือใช้ `get_absolute_url()`
ที่ตัว model แล้วปล่อยให้ `CreateView`/`UpdateView` fallback ไปใช้เองตามที่อธิบายใน
ขั้นตอนที่ 221.5:

```python
# blog/models.py
from django.urls import reverse


class Post(models.Model):
    # ... fields เดิม ...

    def get_absolute_url(self):
        return reverse("blog:detail", kwargs={"slug": self.slug})
```

```python
# blog/views.py — ไม่ต้องเขียน get_success_url() เลย!
class PostCreateView(CreateView):
    model = Post
    form_class = PostForm


class PostUpdateView(UpdateView):
    model = Post
    form_class = PostForm
    slug_field = "slug"
    slug_url_kwarg = "slug"
```

นี่คือตัวอย่างที่ดีของหลักการ **DRY** ที่เรียนไปใน Part 001: กำหนด logic "URL ของ Post
คืออะไร" ไว้ที่เดียวคือ `get_absolute_url()` บน model แล้วทุกที่ที่ต้องการ URL ของ Post
(template, view, admin, shell) ก็เรียกใช้ method เดียวกันนี้ได้หมด

### 224.4 ตารางสรุป: เมื่อไหร่ใช้อะไร

| สถานการณ์ | วิธีแก้ |
|---|---|
| Redirect ไปที่เดิมเสมอ (เช่น หน้ารายการ) ไม่ว่าจะสร้าง object ไหน | `success_url = reverse_lazy(...)` |
| Redirect ไปหน้า detail ของ object ที่เพิ่งสร้าง/แก้ไข และ logic นี้ใช้ที่เดียว | override `get_success_url()` ใน view |
| Redirect ไปหน้า detail และต้องการให้ logic นี้ใช้ซ้ำได้ทั้ง view/template/admin | เพิ่ม `get_absolute_url()` บน model แล้วไม่ต้องเขียนอะไรเพิ่มใน view |
| ต้องการ query string เพิ่มเติมใน URL ปลายทาง (เช่น `?created=1`) | override `get_success_url()` แล้วต่อ query string เอง |

ตัวอย่างการต่อ query string:

```python
def get_success_url(self):
    url = reverse("blog:detail", kwargs={"slug": self.object.slug})
    return f"{url}?created=1"
```

---

## ขั้นตอนที่ 225: Override `form_valid()` / `form_invalid()` — ผูก request.user และ logging

### 225.1 `form_valid()` คือจุดที่ควบคุมได้มากที่สุด

`form_valid(form)` คือ method ที่ถูกเรียก **หลังจาก** ฟอร์ม validate ผ่านแล้ว
(เทียบเท่ากับส่วน `if form.is_valid():` ใน Function-Based View) พฤติกรรมเริ่มต้นของ
`ModelFormMixin.form_valid()` คือ:

```python
# นี่คือสิ่งที่ ModelFormMixin.form_valid() ทำภายใน (โค้ดจริงจาก Django, ย่อเล็กน้อย)
def form_valid(self, form):
    self.object = form.save()
    return super().form_valid(form)   # ไปเรียก FormMixin.form_valid() ที่ทำ redirect
```

การ override `form_valid()` ทำให้คุณสามารถแทรก logic ก่อน/หลัง `form.save()` ได้อย่างอิสระ
— เป็นจุดที่ใช้บ่อยที่สุดในงานจริงเมื่อ CBV ทำสิ่งที่ต้องการยังไม่ครบ

### 225.2 ตัวอย่างที่พบบ่อยที่สุด: ผูก `author` ก่อนบันทึก

ในโลกจริง บล็อกแทบทุกระบบต้องรู้ว่า "ใครเป็นคนเขียนโพสต์นี้" ให้เพิ่ม field `author`
ให้ `Post` ก่อน (ใช้ `settings.AUTH_USER_MODEL` ตามที่เรียนไปใน Part 012):

```python
# blog/models.py
from django.conf import settings
from django.db import models


class Post(models.Model):
    title = models.CharField(max_length=200)
    slug = models.SlugField(max_length=220, unique=True, blank=True)
    content = models.TextField()
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)
    is_published = models.BooleanField(default=False)
    category = models.ForeignKey(
        "Category", on_delete=models.SET_NULL, null=True, blank=True,
        related_name="posts",
    )
    tags = models.ManyToManyField("Tag", blank=True, related_name="posts")
    author = models.ForeignKey(
        settings.AUTH_USER_MODEL, on_delete=models.CASCADE,
        related_name="posts", null=True, blank=True,
    )
    # null=True/blank=True ชั่วคราวเพื่อไม่ให้ migration พังกับข้อมูลเดิมที่มีอยู่แล้ว
    # เมื่อเรียนเรื่อง Authentication เต็มรูปแบบใน Part 031-033 ค่อยพิจารณาตัด null ออก
```

```bash
python manage.py makemigrations blog
python manage.py migrate
```

จากนั้น override `form_valid()` เพื่อผูก `request.user` เข้ากับ `instance` **ก่อน**
ที่จะบันทึกลงฐานข้อมูลจริง (สำคัญมาก: ต้องผูกก่อน `save()` ไม่ใช่หลัง เพราะ `author`
เป็น field ที่ `NOT NULL` ในทางตรรกะของธุรกิจ แม้ใน migration จะยอม `null=True` ไว้ก่อน):

```python
# blog/views.py
from django.urls import reverse
from django.views.generic import CreateView

from .forms import PostForm
from .models import Post


class PostCreateView(CreateView):
    model = Post
    form_class = PostForm

    def form_valid(self, form):
        form.instance.author = self.request.user
        return super().form_valid(form)
```

อธิบายทีละบรรทัด:

- `form.instance` คือ object ที่ฟอร์มกำลังจะสร้าง/บันทึก แต่ **ยังไม่ถูก save()**
  ลงฐานข้อมูล ณ จุดนี้ (มันคือ instance ที่ Django สร้างขึ้นในหน่วยความจำจากข้อมูล
  ในฟอร์มที่ validate ผ่านแล้ว)
- การกำหนด `form.instance.author = self.request.user` คือการเติมค่าที่ **ฟอร์มไม่ได้
  ถามผู้ใช้เลย** (เราไม่อยากให้ผู้ใช้เลือกเองว่าใครคือ author เพราะเสี่ยงต่อการปลอมแปลง —
  ต้องดึงจาก session ที่ login อยู่เท่านั้น)
- `super().form_valid(form)` เรียก parent method ที่จะ `form.save()` จริง (ตอนนี้
  `author` ถูกกำหนดแล้ว) แล้ว redirect ไป `success_url`/`get_success_url()`/
  `get_absolute_url()` ตามลำดับความสำคัญที่อธิบายไปแล้ว

### 225.3 ตัวอย่างที่สอง: Logging การกระทำ (audit trail เบื้องต้น)

อีกกรณีที่ใช้บ่อยคือการบันทึก log ว่าใครทำอะไรกับข้อมูลเมื่อไหร่ (จะเรียนเรื่อง logging
framework เต็มรูปแบบในภายหลัง แต่แนวคิดพื้นฐานใช้ได้ตั้งแต่ตอนนี้):

```python
# blog/views.py
import logging

from django.urls import reverse
from django.views.generic import UpdateView

logger = logging.getLogger(__name__)


class PostUpdateView(UpdateView):
    model = Post
    form_class = PostForm
    slug_field = "slug"
    slug_url_kwarg = "slug"

    def form_valid(self, form):
        response = super().form_valid(form)
        logger.info(
            "Post '%s' (id=%s) ถูกแก้ไขโดย %s",
            self.object.title, self.object.pk, self.request.user,
        )
        return response
```

สังเกตว่ารอบนี้เรา `super().form_valid(form)` **ก่อน** แล้วค่อย log เพราะเราต้องการ
`self.object.pk` ที่มีค่าแน่นอนแล้ว (สำหรับ `UpdateView`, `pk` มีอยู่แล้วตั้งแต่ต้น
เพราะเป็นการแก้ไขของเดิม แต่การเขียนแบบนี้เป็น pattern ที่ปลอดภัยและอ่านง่ายกว่าเสมอ
ไม่ว่าจะเป็น `CreateView` หรือ `UpdateView`)

### 225.4 `form_invalid()`: เมื่อฟอร์มไม่ผ่าน validation

พฤติกรรมเริ่มต้นของ `form_invalid()` คือ render template เดิมพร้อม error message
กลับไป (HTTP status 200) แต่บางครั้งคุณอาจต้องการทำอะไรเพิ่มเติมเมื่อฟอร์มผิด เช่น
log ความพยายามที่ล้มเหลว หรือส่ง message แจ้งเตือน:

```python
from django.contrib import messages


class PostCreateView(CreateView):
    model = Post
    form_class = PostForm

    def form_valid(self, form):
        form.instance.author = self.request.user
        messages.success(self.request, "สร้างบทความสำเร็จแล้ว!")
        return super().form_valid(form)

    def form_invalid(self, form):
        messages.error(self.request, "กรุณาตรวจสอบข้อมูลในฟอร์มอีกครั้ง")
        return super().form_invalid(form)
```

> **หมายเหตุ**: `django.contrib.messages` เป็นระบบแจ้งเตือนแบบ one-time (flash message)
> ที่มากับ Django อยู่แล้ว เราจะเรียนรายละเอียดเต็มรูปแบบเรื่อง sessions/messages
> ใน Part 034 ตอนนี้ขอให้เข้าใจแค่ว่ามันใช้แสดงข้อความแจ้งเตือนหลัง redirect ได้

### 225.5 ตารางสรุป lifecycle methods ที่ override ได้บ่อยที่สุด

| Method | ถูกเรียกเมื่อไหร่ | ใช้ทำอะไรบ่อย ๆ |
|---|---|---|
| `get_form_kwargs()` | ก่อนสร้างฟอร์ม (ทั้ง GET และ POST) | ส่ง `request.user` เข้าไปใน `__init__` ของฟอร์ม |
| `get_initial()` | ตอน `GET` เท่านั้น | กำหนดค่าเริ่มต้นให้ฟอร์ม (เช่น pre-fill category จาก query string) |
| `form_valid(form)` | หลัง validate ผ่าน, ก่อน/หลัง save | ผูก `request.user`, logging, ส่ง email แจ้งเตือน |
| `form_invalid(form)` | หลัง validate ไม่ผ่าน | แจ้งเตือนผู้ใช้, log ความพยายามที่ผิดพลาด |
| `get_success_url()` | หลัง `form_valid()` สำเร็จ | คำนวณ URL ปลายทางแบบ dynamic (ขั้นตอนที่ 224) |

---

## ขั้นตอนที่ 226: `LoginRequiredMixin` เกริ่นสั้น ๆ ก่อนเจาะลึกเต็มใน Part 031-033

### 226.1 ปัญหาที่ยังไม่ได้แก้: ใครก็สร้าง/แก้ไข/ลบโพสต์ได้หมด!

ถ้าคุณลองเปิด `/new/`, `/my-first-post/edit/`, หรือ `/my-first-post/delete/` ตอนนี้
โดยที่ยังไม่ได้ login เลย คุณจะพบว่า **มันยังทำงานได้ปกติ** — นี่คือปัญหาใหญ่สำหรับ
เว็บจริง เพราะใครก็ตามที่รู้ URL สามารถสร้าง/แก้ไข/ลบข้อมูลของคนอื่นได้ทั้งหมด

Django มี mixin ชื่อ `LoginRequiredMixin` ที่แก้ปัญหานี้ได้ในบรรทัดเดียว:

```python
# blog/views.py
from django.contrib.auth.mixins import LoginRequiredMixin
from django.urls import reverse_lazy
from django.views.generic import CreateView, DeleteView, UpdateView

from .forms import PostForm
from .models import Post


class PostCreateView(LoginRequiredMixin, CreateView):
    model = Post
    form_class = PostForm

    def form_valid(self, form):
        form.instance.author = self.request.user
        return super().form_valid(form)


class PostUpdateView(LoginRequiredMixin, UpdateView):
    model = Post
    form_class = PostForm
    slug_field = "slug"
    slug_url_kwarg = "slug"


class PostDeleteView(LoginRequiredMixin, DeleteView):
    model = Post
    slug_field = "slug"
    slug_url_kwarg = "slug"
    success_url = reverse_lazy("blog:list")
```

### 226.2 กฎสำคัญ: `LoginRequiredMixin` ต้องอยู่ "ซ้ายสุด" เสมอ

```python
class PostCreateView(LoginRequiredMixin, CreateView):   # ✅ ถูกต้อง
    ...

class PostCreateView(CreateView, LoginRequiredMixin):   # ❌ ผิด! ไม่ทำงานตามที่คาด
    ...
```

เหตุผลเกี่ยวข้องกับ **Method Resolution Order (MRO)** ของ Python — mixin ที่อยู่ซ้ายสุด
จะถูกตรวจสอบ (`dispatch()`) ก่อนเสมอ เพราะ `LoginRequiredMixin.dispatch()` เป็นจุดที่
เช็คว่า login หรือยัง **ก่อน** ที่จะปล่อยให้ request ไปถึง logic จริงของ `CreateView`
ถ้าสลับตำแหน่งผิด `dispatch()` ของ `CreateView`/`View` อาจถูกเรียกก่อน ทำให้การเช็ค
login ไม่ทำงานเลย เราจะอธิบายกลไก MRO นี้อย่างละเอียดใน Part 024 เมื่อเรียนเรื่อง
Mixin โดยเฉพาะ

### 226.3 พฤติกรรมเมื่อยังไม่ login

เมื่อผู้ใช้ที่ยังไม่ login พยายามเข้าถึง view ที่มี `LoginRequiredMixin`:

1. Django จะ redirect ไปที่ `settings.LOGIN_URL` (ค่าเริ่มต้นคือ `/accounts/login/`)
2. ต่อท้ายด้วย query string `?next=<url เดิมที่พยายามเข้า>` เพื่อให้หลัง login สำเร็จ
   กลับมาที่หน้าเดิมได้อัตโนมัติ

```python
# blog/views.py
class PostCreateView(LoginRequiredMixin, CreateView):
    model = Post
    form_class = PostForm
    login_url = "/accounts/login/"       # กำหนดเองได้ ถ้าไม่กำหนดจะใช้ settings.LOGIN_URL
    redirect_field_name = "next"          # ชื่อ query string parameter (ค่าเริ่มต้นคือ "next" อยู่แล้ว)
```

### 226.4 นี่เป็นแค่การเกริ่นเบื้องต้นเท่านั้น

สิ่งที่ยังไม่ได้พูดถึงในขั้นตอนนี้ (และตั้งใจเก็บไว้สอนแบบเจาะลึกใน Phase 4):

- ระบบ `django.contrib.auth` ทำงานอย่างไรเบื้องหลัง (Part 031)
- การสร้างหน้า login/logout/register ของจริง (Part 031)
- `PermissionRequiredMixin`, `UserPassesTestMixin` สำหรับตรวจสอบสิทธิ์ที่ซับซ้อนกว่า
  "login หรือยัง" (Part 033)
- Custom User Model (Part 032)
- Object-level permission (Part 038)

ตอนนี้ขอให้จำแค่ว่า `LoginRequiredMixin` คือ "ยาม" ตัวแรกสุดที่ป้องกันไม่ให้คนที่
ไม่ได้ login เข้าถึง view ที่ควรมีแค่สมาชิกเท่านั้นถึงจะใช้ได้

---

## ขั้นตอนที่ 227: `FormView` — generic view สำหรับฟอร์มที่ไม่ผูกกับ Model

### 227.1 เมื่อไหร่ที่ `CreateView`/`UpdateView` ใช้ไม่ได้

`CreateView` และ `UpdateView` ถูกออกแบบมาสำหรับฟอร์มที่ **ผูกกับ Model โดยตรง**
(สร้าง/แก้ไข instance ในฐานข้อมูล) แต่ในงานจริงมีฟอร์มจำนวนมากที่ **ไม่ได้สร้าง object
ในฐานข้อมูลเลย** เช่น:

- หน้า "ติดต่อเรา" (Contact Form) ที่แค่ส่งอีเมล ไม่ได้บันทึกลงตาราง
- หน้าค้นหา (Search Form) ที่รับ query แล้วนำไป filter ข้อมูล
- หน้าตั้งค่า (Settings Form) ที่แก้ไขค่าใน `settings` หรือไฟล์ config
- หน้า "สมัครรับข่าวสาร" ที่ส่งข้อมูลไปยัง external API

สำหรับกรณีเหล่านี้ Django มี **`FormView`** ซึ่งจัดการ GET/POST/validation เหมือน
`CreateView` ทุกประการ แต่ **ไม่มี** การผูกกับ Model เลย

### 227.2 ตัวอย่างการใช้งานจริง: หน้าติดต่อเรา

```python
# blog/forms.py
from django import forms


class ContactForm(forms.Form):
    name = forms.CharField(max_length=100, label="ชื่อของคุณ")
    email = forms.EmailField(label="อีเมล")
    message = forms.CharField(widget=forms.Textarea, label="ข้อความ")
```

```python
# blog/views.py
from django.contrib import messages
from django.core.mail import send_mail
from django.urls import reverse_lazy
from django.views.generic import FormView

from .forms import ContactForm


class ContactFormView(FormView):
    template_name = "blog/contact.html"
    form_class = ContactForm
    success_url = reverse_lazy("blog:contact")

    def form_valid(self, form):
        send_mail(
            subject=f"ข้อความจาก {form.cleaned_data['name']}",
            message=form.cleaned_data["message"],
            from_email=form.cleaned_data["email"],
            recipient_list=["admin@example.com"],
            fail_silently=True,
        )
        messages.success(self.request, "ส่งข้อความสำเร็จแล้ว! เราจะติดต่อกลับโดยเร็ว")
        return super().form_valid(form)
```

```python
# blog/urls.py
urlpatterns = [
    # ... URL เดิม ...
    path("contact/", views.ContactFormView.as_view(), name="contact"),
]
```

```html
<!-- blog/templates/blog/contact.html -->
{% extends "base.html" %}

{% block content %}
<h1>ติดต่อเรา</h1>
<form method="post">
    {% csrf_token %}
    {{ form.as_p }}
    <button type="submit">ส่งข้อความ</button>
</form>
{% endblock %}
```

> **หมายเหตุ**: `send_mail()` ต้องการตั้งค่า `EMAIL_BACKEND` ใน `settings.py` ก่อนถึงจะ
> ส่งอีเมลได้จริง สำหรับการพัฒนา (development) แนะนำใช้
> `EMAIL_BACKEND = "django.core.mail.backends.console.EmailBackend"` ซึ่งจะ print
> เนื้อหาอีเมลออกทาง terminal แทนการส่งจริง (เราจะเรียนเรื่องระบบอีเมลแบบเต็มรูปแบบ
> ในภายหลัง ตอนนี้ใช้ console backend เพื่อทดสอบ flow ได้พอ)

### 227.3 จุดต่างสำคัญ: `FormView` ไม่มี `self.object`

เนื่องจากไม่มี Model เกี่ยวข้อง `FormView` จึงไม่มี `self.object`, ไม่มี `get_object()`,
และ default `get_success_url()` ก็ไม่สามารถ fallback ไปที่ `get_absolute_url()`
ได้ (เพราะไม่รู้ว่า object ไหน) — คุณจึงต้องกำหนด `success_url` หรือ override
`get_success_url()` เสมอ ไม่มีทางลัดแบบ `CreateView`/`UpdateView`

### 227.4 ตารางเปรียบเทียบ `FormView` กับ `CreateView`

| คุณสมบัติ | `FormView` | `CreateView` |
|---|---|---|
| ผูกกับ Model | ❌ ไม่ผูก | ✅ ผูกกับ `model` ที่กำหนด |
| ใช้ `forms.Form` หรือ `forms.ModelForm` | `forms.Form` (ปกติ) | `forms.ModelForm` (บังคับ, ไม่ว่าจะผ่าน `fields` หรือ `form_class`) |
| `self.object` | ไม่มี | มี (คือ instance ที่สร้าง/แก้ไข) |
| `form_valid()` เริ่มต้นทำอะไร | แค่ redirect ไป `success_url` (ไม่ save อะไรเลย) | `form.save()` แล้ว redirect |
| Template convention อัตโนมัติ | ❌ ไม่มี ต้องกำหนด `template_name` เสมอ | ✅ มี (`<model>_form.html`) |
| Mixin หลักที่ใช้ | `FormMixin` + `ProcessFormView` | `SingleObjectMixin` + `ModelFormMixin` + `ProcessFormView` |

### 227.5 mental model ที่ควรจำ

> **`CreateView` = `FormView` + ความสามารถผูกกับ Model (ผ่าน `ModelFormMixin` และ
> `SingleObjectMixin`)**

ความเข้าใจนี้จะช่วยได้มากในขั้นตอนถัดไปที่เราจะผ่าเข้าไปดู "เบื้องหลัง" ว่า Django
ประกอบร่าง `CreateView` ขึ้นมาจาก mixin เล็ก ๆ หลายตัวได้อย่างไร

---

## ขั้นตอนที่ 228: เบื้องหลัง `CreateView`/`UpdateView` — `SingleObjectMixin`, `ModelFormMixin`, `ProcessFormView`

### 228.1 ทำไมต้องเข้าใจเบื้องหลัง

หลาย ๆ คนใช้ `CreateView`/`UpdateView` ได้คล่องแต่ไม่เข้าใจว่าทำไม override บาง method
แล้วได้ผล บาง method แล้วไม่ได้ผล การเข้าใจว่า Django **ประกอบ (compose)** generic view
เหล่านี้ขึ้นจาก mixin เล็ก ๆ หลายตัวจะทำให้คุณ:

- รู้ว่าจะ override method ไหนเมื่อต้องการปรับพฤติกรรมเฉพาะจุด
- Debug ได้เร็วขึ้นเมื่อเจอ error แปลก ๆ จาก generic view
- สร้าง CBV กำหนดเองได้อย่างมั่นใจ (จะฝึกจริงจังใน Part 024)

### 228.2 ดู Method Resolution Order (MRO) ด้วยตัวเอง

เปิด Django shell แล้วรันคำสั่งนี้เพื่อดู "สายเลือด" ที่แท้จริงของ `CreateView`:

```bash
python manage.py shell
```

```python
>>> from django.views.generic import CreateView
>>> for cls in CreateView.__mro__:
...     print(cls.__name__)
...
CreateView
SingleObjectTemplateResponseMixin
TemplateResponseMixin
BaseCreateView
ModelFormMixin
FormMixin
SingleObjectMixin
ContextMixin
ProcessFormView
View
object
```

นี่คือลำดับ class จริงที่ Django ประกอบขึ้นเป็น `CreateView` (เวอร์ชันอาจต่างกันเล็กน้อย
ตาม Django version แต่โครงสร้างหลักเหมือนกัน) มาดูว่าแต่ละตัวทำหน้าที่อะไร

### 228.3 แผนภาพความรับผิดชอบของแต่ละ mixin

```
CreateView
    │
    ├── SingleObjectTemplateResponseMixin   → เลือกชื่อ template อัตโนมัติ
    │       (สร้างชื่อ "<app>/<model>_form.html" ให้)
    │
    ├── ModelFormMixin                       → เชื่อม Form เข้ากับ Model
    │       ├── สร้าง ModelForm จาก fields/form_class
    │       ├── form_valid() → form.save() แล้ว redirect
    │       └── get_success_url() → ใช้ self.object.get_absolute_url() ถ้าไม่มี success_url
    │
    ├── SingleObjectMixin                    → จัดการ "object เดี่ยว 1 ตัว"
    │       ├── get_object() / get_queryset()
    │       └── get_context_object_name() → ใส่ตัวแปร "object" (และชื่อ model) ลง context
    │
    └── ProcessFormView                      → ควบคุมการไหลของ GET/POST
            ├── get()  → สร้างฟอร์มเปล่า, render
            └── post() → validate ฟอร์ม → form_valid() หรือ form_invalid()
```

### 228.4 ตารางสรุปหน้าที่ของแต่ละ mixin

| Mixin | Method เด่นที่ให้มา | หน้าที่หลัก |
|---|---|---|
| `ContextMixin` | `get_context_data()` | ฐานสุดของทุก generic view จัดการ context dict |
| `SingleObjectMixin` | `get_object()`, `get_queryset()` | หา/จัดการ object เดี่ยวจาก pk/slug (ใช้ร่วมกับ `DetailView`, `UpdateView`, `DeleteView`) |
| `FormMixin` | `get_form()`, `get_form_class()`, `get_form_kwargs()`, `form_valid()`, `form_invalid()` | จัดการวงจรชีวิตของฟอร์มทั่วไป (ใช้ร่วมกับ `FormView`) |
| `ModelFormMixin` | (สืบทอด `FormMixin` + `SingleObjectMixin`) `form_valid()` | ผสาน Form เข้ากับ Model — เพิ่ม `instance=self.object` ให้ฟอร์ม, `form.save()` |
| `ProcessFormView` | `get()`, `post()` | จุดตัดสินใจว่า GET แสดงฟอร์ม, POST validate แล้วไปเรียก `form_valid()`/`form_invalid()` |
| `SingleObjectTemplateResponseMixin` | `get_template_names()` | สร้างชื่อ template อัตโนมัติตาม convention |
| `TemplateResponseMixin` | `render_to_response()` | แปลง context เป็น `HttpResponse` จริงด้วย template engine |

### 228.5 พิสูจน์ความเข้าใจ: เขียน `CreateView` เองจาก mixin ดิบ ๆ

เพื่อให้เห็นภาพชัดที่สุด ลองประกอบ view ที่ทำงานเหมือน `CreateView` ขึ้นมาเองจาก mixin
พื้นฐาน (โค้ดนี้รันได้จริง และให้ผลลัพธ์เทียบเท่า `CreateView` เกือบทุกประการ):

```python
# ตัวอย่างเพื่อการศึกษา: จำลอง CreateView ขึ้นเองจาก mixin ดิบ
from django.views.generic.edit import ModelFormMixin, ProcessFormView
from django.views.generic.base import TemplateResponseMixin, View


class MyOwnCreateView(TemplateResponseMixin, ModelFormMixin, ProcessFormView, View):
    model = Post
    form_class = PostForm
    template_name = "blog/post_form.html"   # ต้องกำหนดเอง เพราะไม่มี
                                              # SingleObjectTemplateResponseMixin
    success_url = reverse_lazy("blog:list")

    def get(self, request, *args, **kwargs):
        self.object = None   # ModelFormMixin คาดหวังว่าจะมี self.object เสมอ
        return super().get(request, *args, **kwargs)

    def post(self, request, *args, **kwargs):
        self.object = None
        return super().post(request, *args, **kwargs)
```

โค้ดนี้ใช้งานได้จริงเทียบเท่า `CreateView` ทุกประการ ต่างแค่ต้องกำหนด `template_name`
เอง (เพราะขาด `SingleObjectTemplateResponseMixin` ที่สร้างชื่อให้อัตโนมัติ) และต้อง
เขียน `get()`/`post()` เพื่อตั้งค่า `self.object = None` เอง (ซึ่งใน `CreateView` จริง
มันถูกซ่อนไว้ใน `BaseCreateView.get()`/`.post()`) — นี่คือเหตุผลว่าทำไม Django ถึงสร้าง
`CreateView` ให้เป็น shortcut ที่รวม 3-4 mixin เหล่านี้ไว้แล้ว เพื่อไม่ให้คุณต้องเขียน
โค้ดซ้ำแบบนี้ทุกครั้ง

> **ข้อคิดสำคัญ**: generic CBV ของ Django ไม่ใช่ "เวทมนตร์" แต่เป็นการนำ mixin เล็ก ๆ
> ที่แต่ละตัวทำหน้าที่เดียวชัดเจน (Single Responsibility) มาต่อกันด้วย Python's
> multiple inheritance เมื่อคุณเข้าใจว่าแต่ละตัวทำอะไร การ debug และ customize
> จะกลายเป็นเรื่องที่คาดเดาได้ ไม่ใช่การลองผิดลองถูก

### 228.6 ทำไม `UpdateView` MRO ถึงมี `SingleObjectMixin` ปรากฏสองบทบาท

```python
>>> from django.views.generic import UpdateView
>>> for cls in UpdateView.__mro__:
...     print(cls.__name__)
...
UpdateView
SingleObjectTemplateResponseMixin
TemplateResponseMixin
BaseUpdateView
ModelFormMixin
FormMixin
SingleObjectMixin
ContextMixin
ProcessFormView
View
object
```

สังเกตว่า MRO ของ `UpdateView` แทบจะเหมือน `CreateView` เป๊ะ (ต่างแค่ `BaseUpdateView`
แทน `BaseCreateView`) เหตุผลคือ `SingleObjectMixin` ถูกใช้งานอยู่แล้วผ่าน
`ModelFormMixin` (ซึ่งสืบทอดมันมา) ความต่างจริง ๆ อยู่ที่ `BaseUpdateView.get()`/`.post()`
ที่เรียก `self.object = self.get_object()` ก่อนเสมอ (ไม่ใช่ `self.object = None`
เหมือน `BaseCreateView`) — นี่คือจุดต่างเดียวที่ทำให้ `UpdateView` "รู้จัก" object เดิม
ส่วน `CreateView` เริ่มจาก object ว่างเปล่า ตามที่อธิบายไปแล้วใน ขั้นตอนที่ 222.1

---

## ขั้นตอนที่ 229: เกริ่น `CreateView` + Inline Formset (ก่อนเจาะลึกเต็มใน Part 026)

### 229.1 ปัญหา: บางครั้งการสร้าง object เดียวไม่พอ

จนถึงตอนนี้ `CreateView` ของเราสร้าง `Post` ได้ทีละ 1 แถวเท่านั้น แต่ในงานจริงมักมี
ความต้องการที่ซับซ้อนกว่านั้น เช่น "สร้างโพสต์พร้อมกับรูปภาพประกอบหลายรูปในหน้าเดียว"
หรือ "สร้างใบสั่งซื้อพร้อมรายการสินค้าหลายรายการในหน้าเดียว" — สถานการณ์แบบนี้เรียกว่า
**parent-child form** ซึ่ง Django มีเครื่องมือชื่อ **inline formset** สำหรับจัดการ
โดยเฉพาะ

### 229.2 กรณีของแอป `blog`: `tags` เป็น M2M ธรรมดา ไม่ต้องใช้ formset

ก่อนอื่นต้องแยกให้ออกว่า field `tags` ของเรา (ManyToManyField ไป `Tag`) **ไม่จำเป็น
ต้องใช้ inline formset** เพราะ `ModelForm` รองรับ M2M field ได้อยู่แล้วผ่าน
`forms.ModelMultipleChoiceField` แบบตรงไปตรงมา:

```python
# blog/forms.py — เพียงพอสำหรับเลือกหลาย Tag ต่อ 1 Post แล้ว!
class PostForm(forms.ModelForm):
    class Meta:
        model = Post
        fields = ["title", "content", "category", "tags", "is_published"]
        widgets = {
            "tags": forms.CheckboxSelectMultiple,   # แสดงเป็น checkbox หลายอันแทน select box
        }
```

โค้ดนี้ทำให้ผู้ใช้เลือก tag ที่มีอยู่แล้วได้หลายอันในฟอร์มเดียว **โดยไม่ต้องใช้ formset
เลย** — inline formset จะจำเป็นก็ต่อเมื่อคุณต้องการ "สร้าง object ใหม่ที่ผูกกับ Post
ด้วย ForeignKey" (ความสัมพันธ์แบบ one-to-many) ไม่ใช่ M2M

### 229.3 ตัวอย่างสถานการณ์ที่ต้องใช้ inline formset จริง ๆ

สมมติคุณขยาย model เพิ่ม `PostImage` ที่แต่ละโพสต์มีรูปภาพประกอบได้หลายรูป:

```python
# blog/models.py
class PostImage(models.Model):
    post = models.ForeignKey(Post, on_delete=models.CASCADE, related_name="images")
    image = models.ImageField(upload_to="post_images/")
    caption = models.CharField(max_length=200, blank=True)
```

ความต้องการคือ: หน้าสร้างโพสต์ 1 หน้า ต้องกรอกข้อมูล `Post` **และ** เพิ่มรูปภาพได้
หลายรูปพร้อมกัน — นี่คือกรณีคลาสสิกของ **inline formset** โครงร่างคร่าว ๆ
(รายละเอียดเต็มรูปแบบ รวมถึง `extra`, `can_delete`, JavaScript สำหรับเพิ่ม/ลบแถวแบบ
dynamic จะอยู่ใน **Part 026**):

```python
# blog/forms.py
from django.forms import inlineformset_factory

from .models import Post, PostImage

PostImageFormSet = inlineformset_factory(
    Post, PostImage,
    fields=["image", "caption"],
    extra=3,          # แสดงช่องกรอกว่าง 3 ช่องให้เพิ่มรูป
    can_delete=True,
)
```

```python
# blog/views.py — ตัวอย่างแนวคิดเบื้องต้นเท่านั้น (ยังไม่สมบูรณ์ 100%)
class PostCreateView(LoginRequiredMixin, CreateView):
    model = Post
    form_class = PostForm

    def get_context_data(self, **kwargs):
        context = super().get_context_data(**kwargs)
        if self.request.POST:
            context["image_formset"] = PostImageFormSet(self.request.POST, self.request.FILES)
        else:
            context["image_formset"] = PostImageFormSet()
        return context

    def form_valid(self, form):
        context = self.get_context_data()
        image_formset = context["image_formset"]
        form.instance.author = self.request.user

        if image_formset.is_valid():
            response = super().form_valid(form)
            image_formset.instance = self.object   # ผูก formset เข้ากับ Post ที่เพิ่งสร้าง
            image_formset.save()
            return response

        return self.render_to_response(self.get_context_data(form=form))
```

### 229.4 ทำไมเราถึงไม่เจาะลึกตอนนี้

โค้ดข้างบนแสดงให้เห็น "หลักการ" แต่ยังขาดรายละเอียดสำคัญหลายอย่างที่จำเป็นต่อการใช้
งานจริง เช่น:

- การจัดการ `request.FILES` สำหรับอัปโหลดไฟล์ภาพ (จะเรียนเรื่อง File Upload เต็ม
  รูปแบบใน Part 058)
- การแสดง formset ใน template ด้วย `{{ image_formset.management_form }}`
- การเพิ่ม/ลบแถวแบบ dynamic ด้วย JavaScript โดยไม่ reload หน้า
- Validation ข้าม form/formset (เช่น "ต้องมีรูปอย่างน้อย 1 รูป")
- `formset_factory()` (สำหรับ form ที่ไม่ผูกกับ model) เทียบกับ
  `inlineformset_factory()` (ผูกกับ FK)

หัวข้อทั้งหมดนี้จะถูกอธิบายอย่างละเอียดครบถ้วนใน **Part 026: ModelForms และ Formsets**
ตอนนี้ขอให้คุณจำแค่ **mental model** สำคัญไว้ก่อน:

> **Formset คือ "ฟอร์มของหลาย ๆ instance ของ model เดียวกัน" ที่ทำงานร่วมกันเป็นชุด
> ส่วน Inline Formset คือ formset พิเศษที่ผูกกับ parent object ผ่าน ForeignKey โดย
> อัตโนมัติ — เหมาะกับความสัมพันธ์แบบ "1 ต่อกลาย" ไม่ใช่ M2M**

---

## ขั้นตอนที่ 230: สรุปและแบบฝึกหัด — สร้าง Full CRUD สำหรับ `Post` ด้วย CBV ทั้งหมด

### 230.1 ประกอบร่างทุกอย่างเข้าด้วยกัน: โค้ดสมบูรณ์ของ `blog/views.py`

```python
# blog/views.py
import logging

from django.contrib import messages
from django.contrib.auth.mixins import LoginRequiredMixin
from django.urls import reverse_lazy
from django.views.generic import (
    CreateView, DeleteView, DetailView, FormView, ListView, UpdateView,
)

from .forms import ContactForm, PostForm
from .models import Post

logger = logging.getLogger(__name__)


class PostListView(ListView):
    model = Post
    template_name = "blog/post_list.html"
    context_object_name = "posts"
    paginate_by = 10

    def get_queryset(self):
        return (
            Post.objects.filter(is_published=True)
            .select_related("category", "author")
            .prefetch_related("tags")
        )


class PostDetailView(DetailView):
    model = Post
    template_name = "blog/post_detail.html"
    context_object_name = "post"
    slug_field = "slug"
    slug_url_kwarg = "slug"


class PostCreateView(LoginRequiredMixin, CreateView):
    model = Post
    form_class = PostForm
    template_name = "blog/post_form.html"

    def form_valid(self, form):
        form.instance.author = self.request.user
        messages.success(self.request, "สร้างบทความสำเร็จแล้ว!")
        return super().form_valid(form)

    def form_invalid(self, form):
        messages.error(self.request, "กรุณาตรวจสอบข้อมูลในฟอร์มอีกครั้ง")
        return super().form_invalid(form)


class PostUpdateView(LoginRequiredMixin, UpdateView):
    model = Post
    form_class = PostForm
    template_name = "blog/post_form.html"
    slug_field = "slug"
    slug_url_kwarg = "slug"

    def get_queryset(self):
        return Post.objects.filter(author=self.request.user)

    def form_valid(self, form):
        response = super().form_valid(form)
        logger.info(
            "Post '%s' (id=%s) ถูกแก้ไขโดย %s",
            self.object.title, self.object.pk, self.request.user,
        )
        messages.success(self.request, "แก้ไขบทความสำเร็จแล้ว!")
        return response


class PostDeleteView(LoginRequiredMixin, DeleteView):
    model = Post
    template_name = "blog/post_confirm_delete.html"
    slug_field = "slug"
    slug_url_kwarg = "slug"
    success_url = reverse_lazy("blog:list")

    def get_queryset(self):
        return Post.objects.filter(author=self.request.user)

    def form_valid(self, form):
        messages.success(self.request, "ลบบทความสำเร็จแล้ว")
        return super().form_valid(form)


class ContactFormView(FormView):
    template_name = "blog/contact.html"
    form_class = ContactForm
    success_url = reverse_lazy("blog:contact")

    def form_valid(self, form):
        logger.info("ข้อความติดต่อจาก %s <%s>", form.cleaned_data["name"], form.cleaned_data["email"])
        messages.success(self.request, "ส่งข้อความสำเร็จแล้ว! เราจะติดต่อกลับโดยเร็ว")
        return super().form_valid(form)
```

```python
# blog/models.py (เพิ่ม get_absolute_url เพื่อใช้ร่วมกับ get_success_url)
from django.conf import settings
from django.db import models
from django.urls import reverse


class Post(models.Model):
    title = models.CharField(max_length=200)
    slug = models.SlugField(max_length=220, unique=True, blank=True)
    content = models.TextField()
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)
    is_published = models.BooleanField(default=False)
    category = models.ForeignKey(
        "Category", on_delete=models.SET_NULL, null=True, blank=True,
        related_name="posts",
    )
    tags = models.ManyToManyField("Tag", blank=True, related_name="posts")
    author = models.ForeignKey(
        settings.AUTH_USER_MODEL, on_delete=models.CASCADE,
        related_name="posts", null=True, blank=True,
    )

    class Meta:
        ordering = ["-created_at"]

    def __str__(self):
        return self.title

    def get_absolute_url(self):
        return reverse("blog:detail", kwargs={"slug": self.slug})
```

```python
# blog/urls.py — URL ครบทั้ง CRUD
from django.urls import path

from . import views

app_name = "blog"

urlpatterns = [
    path("", views.PostListView.as_view(), name="list"),
    path("new/", views.PostCreateView.as_view(), name="create"),
    path("contact/", views.ContactFormView.as_view(), name="contact"),
    path("<slug:slug>/", views.PostDetailView.as_view(), name="detail"),
    path("<slug:slug>/edit/", views.PostUpdateView.as_view(), name="update"),
    path("<slug:slug>/delete/", views.PostDeleteView.as_view(), name="delete"),
]
```

### 230.2 ตารางสรุป URL ทั้งหมดของแอป `blog`

| URL Pattern | View | ชื่อ URL (namespace: `blog`) | HTTP Method ที่ใช้จริง |
|---|---|---|---|
| `/` | `PostListView` | `blog:list` | `GET` |
| `/new/` | `PostCreateView` | `blog:create` | `GET`, `POST` |
| `/contact/` | `ContactFormView` | `blog:contact` | `GET`, `POST` |
| `/<slug>/` | `PostDetailView` | `blog:detail` | `GET` |
| `/<slug>/edit/` | `PostUpdateView` | `blog:update` | `GET`, `POST` |
| `/<slug>/delete/` | `PostDeleteView` | `blog:delete` | `GET`, `POST` |

### 230.3 สรุปสิ่งที่ได้เรียนรู้ใน Part นี้

- ✅ ใช้ `CreateView` สร้าง object ใหม่ผ่านฟอร์มได้ ทั้งแบบ `fields` และ `form_class`
- ✅ เข้าใจว่า `success_url` ต้องใช้ `reverse_lazy()` เสมอ เพราะเป็น class attribute
- ✅ ใช้ `UpdateView` แก้ไข object เดิมได้ และเข้าใจกลไก `get_object()` ที่ใช้ร่วมกับ
  `DetailView`
- ✅ ใช้ `DeleteView` ลบ object พร้อมหน้ายืนยัน และเข้าใจว่าทำไม `{% csrf_token %}`
  สำคัญเป็นพิเศษกับการลบ
- ✅ Override `get_success_url()` เพื่อ redirect แบบ dynamic ไปหน้า detail ของ object
  ที่เพิ่งสร้าง/แก้ไข และรู้จักทางลัดด้วย `get_absolute_url()` บน model
- ✅ Override `form_valid()`/`form_invalid()` เพื่อผูก `request.user`, ทำ logging,
  และแจ้งเตือนผู้ใช้ด้วย `messages`
- ✅ ใช้ `LoginRequiredMixin` ป้องกันไม่ให้ผู้ใช้ที่ไม่ login เข้าถึง view ที่แก้ไขข้อมูลได้
  (พร้อมรู้ว่าต้องเจาะลึกเต็มรูปแบบใน Part 031-033)
- ✅ ใช้ `FormView` สำหรับฟอร์มที่ไม่ผูกกับ Model เช่นหน้าติดต่อเรา
- ✅ เข้าใจ MRO เบื้องหลัง `CreateView`/`UpdateView` ว่าประกอบจาก
  `SingleObjectMixin`, `ModelFormMixin`, `ProcessFormView` อย่างไร
- ✅ รู้จักแนวคิด inline formset เบื้องต้น และรู้ขอบเขตว่าอะไรที่ยังไม่ได้เรียนเจาะลึก
  (จะไปเรียนเต็มใน Part 026)

### 230.4 Checklist ก่อนไป Part ถัดไป

- [ ] สร้าง `PostCreateView`, `PostUpdateView`, `PostDeleteView` ในแอป `blog` สำเร็จ
- [ ] สร้างไฟล์ `blog/forms.py` ที่มี `PostForm` (ModelForm) และใช้ `form_class` แทน
      `fields` ตรง ๆ
- [ ] สร้าง template `post_form.html` และ `post_confirm_delete.html` ตาม naming
      convention ที่ถูกต้อง
- [ ] เพิ่ม field `author` ให้ `Post` และรัน migration สำเร็จ
- [ ] Override `form_valid()` เพื่อผูก `author = request.user` ก่อนบันทึกได้จริง
- [ ] ทดสอบว่าเมื่อยังไม่ login แล้วเข้า `/new/` จะถูก redirect ไปหน้า login
- [ ] สร้าง `ContactFormView` ด้วย `FormView` และทดสอบส่งฟอร์มสำเร็จ (ตรวจสอบผลลัพธ์ใน
      terminal ถ้าใช้ console email backend)
- [ ] รันคำสั่ง `for cls in CreateView.__mro__: print(cls.__name__)` ใน shell และ
      อธิบายหน้าที่ของแต่ละ class ได้ด้วยคำพูดตัวเอง
- [ ] URL ครบทั้ง 6 เส้นทางตามตารางใน 230.2 ทำงานถูกต้องทุกเส้นทาง

### 230.5 แบบฝึกหัดท้ายบท

**แบบฝึกหัดที่ 1**: สร้าง Full CRUD สำหรับโมเดล `Category` ด้วย CBV ทั้งหมด
(`CategoryListView`, `CategoryCreateView`, `CategoryUpdateView`, `CategoryDeleteView`)
โดยใช้หลักการเดียวกับที่เรียนใน Part นี้ทุกประการ ต้องมี:
- `fields = ["name", "slug", "description"]` ผ่าน `form_class` ไม่ใช่ `fields` ตรง ๆ
- ป้องกันด้วย `LoginRequiredMixin` ทุก view ที่แก้ไขข้อมูล
- `success_url` ที่ redirect กลับไปหน้ารายการ `Category` เสมอ

**แบบฝึกหัดที่ 2**: แก้ไข `PostCreateView` ให้เพิ่ม field พิเศษที่ไม่ได้อยู่ใน model
ชื่อ `notify_subscribers` (เป็น `BooleanField` ใน `forms.Form` ธรรมดา ไม่ใช่ของ
`ModelForm`) แล้วใน `form_valid()` ให้เช็คค่านี้จาก `form.cleaned_data` — ถ้าเป็น `True`
ให้ `print()` ข้อความ `"กำลังส่งอีเมลแจ้งเตือนสมาชิก..."` ออกทาง terminal (จำลอง
การส่งจริง) โจทย์นี้ฝึกให้เห็นว่า `form_class` ที่กำหนดเองสามารถมี field เกินกว่า
model ได้ ตราบใดที่คุณจัดการ field พิเศษนั้นเองใน `form_valid()` (ไม่ส่งต่อให้
`form.save()` ตรง ๆ)

**แบบฝึกหัดที่ 3**: เขียน `PostUpdateView` เวอร์ชันใหม่ที่ override
`get_success_url()` ให้ต่อ query string `?updated=1` ท้าย URL ปลายทาง แล้วแก้
`post_detail.html` ให้แสดงข้อความ "แก้ไขข้อมูลสำเร็จ" เมื่อ `request.GET.updated == "1"`
โจทย์นี้ฝึกการอ่าน query string ใน template ด้วย `{% if request.GET.updated %}`

**แบบฝึกหัดที่ 4 (ขั้นสูง)**: รันคำสั่งต่อไปนี้ใน Django shell แล้วเปรียบเทียบผลลัพธ์
ของ `CreateView.__mro__`, `UpdateView.__mro__`, `DeleteView.__mro__`, และ
`FormView.__mro__` เขียนตารางเปรียบเทียบลงในไฟล์ `notes.md` ว่า mixin ตัวไหนปรากฏใน
generic view ไหนบ้าง แล้วอธิบายว่าทำไม `DeleteView` ถึง **ไม่มี** `ModelFormMixin`
ในสายเลือดของมัน (คำใบ้: `DeleteView` ไม่ต้องใช้ฟอร์มเพื่อ validate ข้อมูล เพราะไม่มี
field ให้กรอกเลย มันแค่ต้องการปุ่มยืนยันเท่านั้น)

### 230.6 คำถามที่พบบ่อย (FAQ)

**Q: ทำไม `CreateView` ไม่รองรับการอัปโหลดไฟล์โดยอัตโนมัติ ต้องทำอะไรเพิ่ม?**
A: ฟอร์ม HTML ที่มีการอัปโหลดไฟล์ต้องมี `enctype="multipart/form-data"` ใน `<form>`
tag เสมอ (Django ไม่ได้เติมให้อัตโนมัติเพราะไม่ใช่ทุกฟอร์มต้องการ) และถ้า model มี
`ImageField`/`FileField`, `CreateView`/`UpdateView` จะจัดการ `request.FILES` ให้
อัตโนมัติอยู่แล้วผ่าน `get_form_kwargs()` ตราบใดที่ template มี `enctype` ที่ถูกต้อง
รายละเอียดเต็มรูปแบบเรื่อง File Upload จะอยู่ใน Part 058

**Q: ถ้าอยากให้ `DeleteView` ลบแบบ soft delete (ไม่ลบจริง แค่ตั้ง flag `is_deleted=True`)
ทำอย่างไร?**
A: Override method `form_valid()` ของ `DeleteView` (Django 5.x เปลี่ยนจาก `delete()`
เป็น `form_valid()` เป็น entry point หลักแล้ว) โดยไม่เรียก `self.object.delete()`
แต่ตั้งค่า flag แล้ว `save()` แทน:
```python
class PostDeleteView(LoginRequiredMixin, DeleteView):
    model = Post
    success_url = reverse_lazy("blog:list")

    def form_valid(self, form):
        self.object.is_deleted = True
        self.object.save(update_fields=["is_deleted"])
        return redirect(self.get_success_url())
```
(ต้องเพิ่ม field `is_deleted = models.BooleanField(default=False)` ใน model และ
ปรับ `get_queryset()` ของทุก view ให้กรอง `is_deleted=False` ด้วย)

**Q: `CreateView` กับ `UpdateView` ใช้ template ชื่อเดียวกัน แล้วถ้าอยากแยกหน้าตาจริง ๆ
ล่ะ?**
A: กำหนด `template_name` แยกกันตรง ๆ ใน view แต่ละตัวได้เลย เช่น
`template_name = "blog/post_create_form.html"` ใน `PostCreateView` และ
`template_name = "blog/post_edit_form.html"` ใน `PostUpdateView` — การใช้ template
ร่วมกันเป็นแค่ **ค่าเริ่มต้นที่สะดวก** ไม่ใช่ข้อบังคับ

**Q: ทำไมตอน `POST` ข้อมูลผิด แล้ว render ฟอร์มกลับมา ข้อมูลที่ผู้ใช้กรอกไปแล้วไม่หายไป?**
A: เพราะ `form_invalid()` render ฟอร์มแบบ **bound** (ผูกกับข้อมูลที่ส่งมาใน
`request.POST`) ไม่ใช่ฟอร์มเปล่า ๆ ผู้ใช้จึงเห็นค่าที่ตัวเองกรอกไว้เดิม พร้อมข้อความ
error เฉพาะ field ที่ผิด — นี่คือ UX ที่ดีที่ Django ออกแบบมาให้เป็นค่าเริ่มต้นอยู่แล้ว
โดยที่คุณไม่ต้องเขียนโค้ดอะไรเพิ่มเลย

---

## เตรียมตัวสำหรับ Part ถัดไป

**Part 024: Mixins และการสร้าง CBV แบบกำหนดเอง** จะพาคุณเจาะลึกกลไก **Mixin** และ
**Method Resolution Order (MRO)** ของ Python อย่างละเอียด ต่อยอดจากที่เกริ่นไว้ใน
ขั้นตอนที่ 226 และ 228 ของ Part นี้ คุณจะได้เรียนรู้วิธีสร้าง mixin กำหนดเอง (custom
mixin) เพื่อใช้ซ้ำ logic ข้าม view หลายตัว เช่น mixin ที่ตรวจสอบว่าผู้ใช้เป็นเจ้าของ
object หรือไม่ (`AuthorRequiredMixin`), mixin ที่เพิ่ม breadcrumb เข้า context
อัตโนมัติ, และวิธี debug ปัญหา MRO ที่ซับซ้อนเมื่อ mixin หลายตัวชนกัน

เตรียมทบทวนเรื่อง Python OOP จาก Part 002 (โดยเฉพาะเรื่อง multiple inheritance และ
`super()`) ไว้ให้แม่น เพราะ Part 024 จะใช้ความรู้นี้อย่างเข้มข้น!
