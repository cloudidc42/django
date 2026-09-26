# Part 024: Mixins และการสร้าง CBV แบบกำหนดเอง

> **ขั้นตอนที่ 231-240 ของหลักสูตร** | Phase 3: Views, Templates, Forms และ CBV
>
> เป้าหมายของ Part นี้: เจาะลึกกลไกที่แท้จริงเบื้องหลัง Class-Based View ทุกตัวที่คุณ
> เคยใช้มาตั้งแต่ Part 021-023 นั่นคือ **Mixin** และ **MRO (Method Resolution Order)**
> เมื่อจบ Part นี้ คุณจะไม่ใช่แค่ "ใช้" `ListView`, `CreateView`, `UpdateView` เป็นแล้ว
> แต่จะสามารถ **ออกแบบ Mixin ของตัวเอง** เพื่อแก้ปัญหาจริงที่เกิดขึ้นเสมอในโปรเจกต์
> ระดับมืออาชีพ เช่น จำกัดสิทธิ์เจ้าของข้อมูล, แชร์ context ร่วมกันหลายหน้า, ทำ Logging,
> และตอบกลับเป็น JSON — ทั้งหมดนี้จะถูกนำไปใช้จริงกับระบบ CRUD ของ `Post` ที่คุณสร้าง
> ไว้ตั้งแต่ Part 023

---

## สารบัญของ Part นี้

- ขั้นตอนที่ 231: ทบทวน Mixin และ MRO (Method Resolution Order) แล้วโยงเข้ากับ Django CBV
- ขั้นตอนที่ 232: เขียน Custom Mixin เอง — `AuthorRequiredMixin`
- ขั้นตอนที่ 233: ลำดับการวาง Mixin ใน class definition สำคัญอย่างไร
- ขั้นตอนที่ 234: `UserPassesTestMixin` และ `PermissionRequiredMixin` ของ Django เอง
- ขั้นตอนที่ 235: Override `dispatch()` ใน Mixin เพื่อทำ logic ก่อนทุก HTTP method
- ขั้นตอนที่ 236: สร้าง Base ListView/DetailView ที่ inject context ร่วมกัน
- ขั้นตอนที่ 237: สร้าง `JsonResponseMixin` สำหรับ view ที่ตอบกลับเป็น JSON
- ขั้นตอนที่ 238: การเขียน Test สำหรับ CBV เบื้องต้น (`self.client`, `RequestFactory`)
- ขั้นตอนที่ 239: กรอบการตัดสินใจ FBV vs CBV แบบมืออาชีพ
- ขั้นตอนที่ 240: สรุปและแบบฝึกหัด — ประกอบ Mixin ทั้งหมดเข้ากับ Post CRUD จริง

---

## ขั้นตอนที่ 231: ทบทวน Mixin และ MRO (Method Resolution Order) แล้วโยงเข้ากับ Django CBV

### 231.1 ย้อนกลับไปที่ Part 002: Mixin คืออะไร

ใน **Part 002 ขั้นตอนที่ 13.5** เราเคยเห็นตัวอย่างนี้มาแล้ว:

```python
class LoginRequiredMixin:
    def dispatch(self, request, *args, **kwargs):
        if not request.user.is_authenticated:
            return redirect("login")
        return super().dispatch(request, *args, **kwargs)


class StaffRequiredMixin:
    def dispatch(self, request, *args, **kwargs):
        if not request.user.is_staff:
            raise PermissionDenied
        return super().dispatch(request, *args, **kwargs)


class AdminDashboardView(LoginRequiredMixin, StaffRequiredMixin, ListView):
    model = Product
```

ตอนนั้นเราเรียนแค่ "แนวคิด" ของ Mixin ใน Python ล้วน ๆ ยังไม่ได้ลงลึกว่า Django เอง
ใช้แนวคิดนี้สร้าง Generic CBV ทั้งหมดที่เราใช้ใน Part 021-023 (`ListView`, `DetailView`,
`CreateView`, `UpdateView`, `DeleteView`) ได้อย่างไร นี่คือสิ่งที่ Part นี้จะเจาะลึก

สรุปสั้น ๆ อีกครั้ง: **Mixin** คือ class เล็ก ๆ ที่ไม่ได้ออกแบบมาให้ instantiate ใช้งาน
เดี่ยว ๆ แต่ออกแบบมาให้ "ผสม" (mix in) เข้ากับ class อื่นผ่าน multiple inheritance
เพื่อเพิ่มพฤติกรรมเฉพาะทางเข้าไป โดยไม่ต้องเขียนโค้ดซ้ำในทุก view

### 231.2 MRO และอัลกอริทึม C3 Linearization แบบเจาะลึกขึ้น

Part 002 แนะนำให้ตรวจสอบ MRO ด้วย `.__mro__` หรือ `.mro()` ไปแล้ว ใน Part นี้เราจะใช้
เครื่องมือเดียวกันนี้ตลอดทั้งบท เพราะการเข้าใจ MRO คือกุญแจสำคัญที่สุดในการ debug ปัญหา
Mixin ที่ทำงานผิดคาด

กฎของ MRO ที่ต้องจำขึ้นใจก่อนไปต่อ:

1. Python ใช้อัลกอริทึม **C3 Linearization** ในการเรียง MRO ไม่ใช่แค่ depth-first
   แบบภาษาอื่น
2. class ที่เขียนอยู่ **ซ้ายมือสุด** ใน parentheses จะถูกพิจารณาก่อนเสมอ
3. เมื่อเมธอดใน class หนึ่งเรียก `super()` มันจะไม่ได้เรียก "parent class ของตัวเอง"
   ตรง ๆ เสมอไป แต่เรียก **class ถัดไปใน MRO chain ของ instance นั้น** — นี่คือจุดที่
   ทำให้ Mixin ทำงานร่วมกันได้แบบ "cooperative"

```python
class A:
    def greet(self):
        print("A.greet")


class B(A):
    def greet(self):
        print("B.greet เริ่ม")
        super().greet()
        print("B.greet จบ")


class C(A):
    def greet(self):
        print("C.greet เริ่ม")
        super().greet()
        print("C.greet จบ")


class D(B, C):
    def greet(self):
        print("D.greet เริ่ม")
        super().greet()
        print("D.greet จบ")


print([cls.__name__ for cls in D.__mro__])
# ['D', 'B', 'C', 'A', 'object']

D().greet()
# D.greet เริ่ม
# B.greet เริ่ม
# C.greet เริ่ม
# A.greet
# C.greet จบ
# B.greet จบ
# D.greet จบ
```

สังเกตว่า `B.greet` เรียก `super().greet()` แต่สิ่งที่ถูกเรียกจริงคือ `C.greet` ไม่ใช่
`A.greet` — เพราะ MRO ของ `D` คือ `[D, B, C, A, object]` และ `super()` จะไล่ไปหา
"ตัวถัดไปใน MRO chain" เสมอ ไม่ใช่ parent ของ class ที่กำลังรันอยู่ นี่คือกลไกที่ทำให้
เราวาง Mixin หลายตัวต่อกันเป็นสายและทุกตัวทำงานครบทุกตัวได้ ตราบใดที่ **ทุกตัวเรียก
`super()`**

### 231.3 Django CBV ทั้งหมดคือ "ต้นไม้ Mixin" ขนาดใหญ่

Generic CBV ที่คุณใช้มาตั้งแต่ Part 022-023 ไม่ได้เป็น class เดี่ยว ๆ แต่เป็นการรวม
Mixin เล็ก ๆ หลายตัวเข้าด้วยกัน ลองดูโครงสร้างจริงของ Django (แบบย่อเพื่อความเข้าใจ):

```
View
 └── TemplateResponseMixin ─┐
                             ├── ListView
     MultipleObjectMixin ───┘

View
 └── TemplateResponseMixin ─┐
                             ├── DetailView
     SingleObjectMixin ─────┘

View
 └── TemplateResponseMixin ──┐
     SingleObjectMixin ──────┼── CreateView / UpdateView
     ModelFormMixin ─────────┤   (ผ่าน ProcessFormView, FormMixin)
     ProcessFormView ────────┘

View
 └── SingleObjectMixin ──┐
     DeletionMixin ───────┼── DeleteView
     TemplateResponseMixin┘
```

ตรวจสอบได้ด้วยตัวเองจริง ๆ ผ่าน Python shell (`python manage.py shell`):

```python
from django.views.generic import CreateView

for cls in CreateView.__mro__:
    print(cls.__module__, "->", cls.__name__)
```

ผลลัพธ์ (ย่อ):

```
django.views.generic.edit CreateView
django.views.generic.detail SingleObjectMixin
django.views.generic.edit ModelFormMixin
django.views.generic.edit FormMixin
django.views.generic.base ContextMixin
django.views.generic.edit ProcessFormView
django.views.generic.base TemplateResponseMixin
django.views.generic.base View
builtins object
```

### 231.4 ตารางสรุป Mixin หลักที่ประกอบกันเป็น Generic CBV

| Mixin | ให้ความสามารถอะไร | ใช้ในกรณี |
|---|---|---|
| `View` | ฐานสุดของทุก CBV, มี `dispatch()`, `as_view()` | ทุกคลาส |
| `ContextMixin` | เมธอด `get_context_data()` มาตรฐาน | ทุกคลาสที่ render template |
| `TemplateResponseMixin` | เมธอด `render_to_response()`, จัดการ `template_name` | ทุกคลาสที่ render HTML |
| `SingleObjectMixin` | ดึง object เดียวด้วย `get_object()`, `get_queryset()` | `DetailView`, `UpdateView`, `DeleteView` |
| `MultipleObjectMixin` | ดึงหลาย object พร้อม pagination | `ListView` |
| `FormMixin` | จัดการฟอร์มทั่วไป: `get_form()`, `form_valid()`, `form_invalid()` | `CreateView`, `UpdateView`, `FormView` |
| `ModelFormMixin` | ผูก `FormMixin` เข้ากับ Model โดยตรง | `CreateView`, `UpdateView` |
| `ProcessFormView` | จัดการ `get()`/`post()` มาตรฐานของฟอร์ม | `CreateView`, `UpdateView` |
| `DeletionMixin` | จัดการ `post()` สำหรับลบ object | `DeleteView` |

การเข้าใจตารางนี้สำคัญมาก เพราะเมื่อคุณเขียน Mixin ของตัวเองในขั้นตอนถัดไป คุณกำลัง
"เพิ่ม node" เข้าไปในต้นไม้นี้ ไม่ใช่เขียนอะไรที่แยกขาดจากกลไกเดิมของ Django เลย

---

## ขั้นตอนที่ 232: เขียน Custom Mixin เอง — `AuthorRequiredMixin`

### 232.1 โจทย์: อนุญาตให้เฉพาะเจ้าของโพสต์แก้ไข/ลบได้

ระบบ `Post` CRUD ที่สร้างไว้ใน Part 023 มี `PostUpdateView` และ `PostDeleteView` ที่ตอนนี้
ผู้ใช้ที่ล็อกอินแล้ว **คนไหนก็ได้** สามารถแก้ไขหรือลบโพสต์ของคนอื่นได้ — นี่คือช่องโหว่
ด้านความปลอดภัยที่ร้ายแรงมากในระบบจริง เราต้องการกฎว่า:

> ผู้ใช้ต้องล็อกอิน **และ** ต้องเป็นเจ้าของโพสต์นั้น (`request.user == post.author`)
> เท่านั้น จึงจะแก้ไขหรือลบโพสต์ได้

สมมติโครงสร้าง Model ต่อจาก Part 023 มีลักษณะนี้ (ใน `blog/models.py`):

```python
from django.conf import settings
from django.db import models
from django.urls import reverse


class Category(models.Model):
    name = models.CharField(max_length=100, unique=True)
    slug = models.SlugField(max_length=100, unique=True)

    class Meta:
        verbose_name_plural = "categories"
        ordering = ["name"]

    def __str__(self):
        return self.name


class Post(models.Model):
    class Status(models.TextChoices):
        DRAFT = "draft", "ฉบับร่าง"
        PUBLISHED = "published", "เผยแพร่แล้ว"

    title = models.CharField(max_length=200)
    slug = models.SlugField(max_length=220, unique=True)
    content = models.TextField()
    author = models.ForeignKey(
        settings.AUTH_USER_MODEL,
        on_delete=models.CASCADE,
        related_name="posts",
    )
    category = models.ForeignKey(
        Category, on_delete=models.SET_NULL, null=True, blank=True, related_name="posts"
    )
    status = models.CharField(max_length=10, choices=Status.choices, default=Status.DRAFT)
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)

    class Meta:
        ordering = ["-created_at"]

    def __str__(self):
        return self.title

    def get_absolute_url(self):
        return reverse("blog:post-detail", kwargs={"slug": self.slug})
```

### 232.2 เขียน `AuthorRequiredMixin` เวอร์ชันแรก (override `dispatch()`)

เราจะเริ่มจากการเขียน Mixin แบบดิบที่สุดก่อน โดย override `dispatch()` ตามแนวทางที่
ทบทวนไปใน ขั้นตอนที่ 231 (และ Part 002 ขั้นตอนที่ 13.5) สร้างไฟล์ `blog/mixins.py`:

```python
# blog/mixins.py
from django.core.exceptions import PermissionDenied


class AuthorRequiredMixin:
    """
    อนุญาตให้เฉพาะ request.user ที่เป็นเจ้าของ object (ผ่าน field `author`)
    เท่านั้นที่จะผ่านเข้าไปทำงานใน view ได้ ใช้ร่วมกับ CBV ที่มี SingleObjectMixin
    (เช่น UpdateView, DeleteView) เพราะต้องพึ่งเมธอด self.get_object()
    """

    def dispatch(self, request, *args, **kwargs):
        obj = self.get_object()
        if obj.author != request.user:
            raise PermissionDenied("คุณไม่ใช่เจ้าของโพสต์นี้ จึงไม่มีสิทธิ์ดำเนินการ")
        return super().dispatch(request, *args, **kwargs)
```

จุดสำคัญที่ต้องสังเกต:

- `self.get_object()` มาจาก `SingleObjectMixin` ที่ `UpdateView`/`DeleteView` มีอยู่แล้ว
  Mixin ของเราจึง **พึ่งพา** (depend on) ว่า class ที่นำไปผสมต้องมีเมธอดนี้อยู่ก่อน —
  นี่คือข้อจำกัดโดยธรรมชาติของ Mixin ที่ต้องระวังเสมอ
- เราใช้ `PermissionDenied` ซึ่ง Django จะจับแล้วแสดงหน้า **403 Forbidden** ให้อัตโนมัติ
  (ไม่ใช่ 404 หรือ redirect เงียบ ๆ) ทำให้ผู้ใช้เข้าใจสถานการณ์ชัดเจน

### 232.3 ปรับปรุง: cache `get_object()`, จัดการ `AnonymousUser`, กำหนดพฤติกรรมเมื่อไม่ผ่าน

เวอร์ชันแรกมีจุดอ่อน 2 อย่าง:

1. `self.get_object()` อาจถูกเรียกซ้ำหลายครั้งในระหว่าง request เดียวกัน (เช่นใน
   `dispatch()` ของเรา แล้ว Django เรียกอีกครั้งใน `get()`/`post()`) ทำให้ query
   ฐานข้อมูลซ้ำซ้อนโดยไม่จำเป็น
2. ถ้า `AnonymousUser` เข้ามา (ยังไม่ล็อกอิน) การเทียบ `obj.author != request.user`
   จะได้ `True` เสมอ (เพราะ author ไม่มีทาง `== AnonymousUser`) ซึ่งบังเอิญ "ถูกต้อง"
   แต่ error message ที่ได้จะสับสน (ควรบอกว่า "กรุณาล็อกอินก่อน" ไม่ใช่ "ไม่ใช่เจ้าของ")

เวอร์ชันที่ปรับปรุงแล้ว:

```python
# blog/mixins.py
from django.contrib.auth.mixins import LoginRequiredMixin
from django.core.exceptions import PermissionDenied


class AuthorRequiredMixin(LoginRequiredMixin):
    """
    อนุญาตเฉพาะ request.user ที่ล็อกอินแล้ว และเป็นเจ้าของ object (author == request.user)

    หมายเหตุ: สืบทอดจาก LoginRequiredMixin ของ Django เอง (จะเจาะลึกใน ขั้นตอนที่ 234)
    เพื่อให้ผู้ใช้ที่ยังไม่ล็อกอินถูก redirect ไปหน้า login ก่อน แทนที่จะเจอ 403 ทันที
    """

    raise_exception = False  # False = redirect ไป login, True = ยิง PermissionDenied ทันที

    def dispatch(self, request, *args, **kwargs):
        # cache object ไว้ใน self._author_check_object กัน get_object() ถูกเรียกซ้ำ
        self.object = self.get_object()
        if self.object.author_id != request.user.id:
            raise PermissionDenied("คุณไม่ใช่เจ้าของโพสต์นี้ จึงไม่มีสิทธิ์ดำเนินการ")
        return super().dispatch(request, *args, **kwargs)

    def get_object(self, queryset=None):
        # ถ้าเคยดึงไปแล้วใน dispatch() ให้คืนค่าที่ cache ไว้ ไม่ query ซ้ำ
        if hasattr(self, "object") and self.object is not None:
            return self.object
        return super().get_object(queryset)
```

สังเกตการเปลี่ยนแปลงที่สำคัญ:

- ใช้ `obj.author_id != request.user.id` แทน `obj.author != request.user` — เร็วกว่า
  เพราะเทียบเลข primary key ตรง ๆ โดยไม่ต้อง query object `User` เต็มรูปแบบ (เทคนิคที่
  ทีมมืออาชีพใช้ประจำเมื่อ compare FK)
- `AuthorRequiredMixin` สืบทอดจาก `LoginRequiredMixin` (ของ Django) ทำให้ได้พฤติกรรม
  "redirect ไป login ก่อนถ้ายังไม่ล็อกอิน" มาฟรี ๆ โดยไม่ต้องเขียนเอง — นี่คือตัวอย่าง
  ที่ดีของการ **compose Mixin ต่อ Mixin** ซึ่งเราจะเจาะลึกกลไกนี้ใน ขั้นตอนที่ 234
- การ cache `self.object` ยังมีประโยชน์อีกทาง: Django's `UpdateView`/`DeleteView`
  เรียก `self.object = self.get_object()` เองอยู่แล้วใน `get()`/`post()` เดิม
  ดังนั้นการ cache ไว้ตั้งแต่ `dispatch()` ทำให้ query ฐานข้อมูลเกิดขึ้น **ครั้งเดียว**
  ต่อ request แทนที่จะเป็น 2 ครั้ง

### 232.4 นำไปใช้กับ `PostUpdateView` และ `PostDeleteView`

```python
# blog/views.py
from django.contrib.auth.mixins import LoginRequiredMixin
from django.urls import reverse_lazy
from django.views.generic import CreateView, DeleteView, DetailView, ListView, UpdateView

from .forms import PostForm
from .mixins import AuthorRequiredMixin
from .models import Post


class PostListView(ListView):
    model = Post
    template_name = "blog/post_list.html"
    context_object_name = "posts"
    paginate_by = 10

    def get_queryset(self):
        return Post.objects.filter(status=Post.Status.PUBLISHED).select_related(
            "author", "category"
        )


class PostDetailView(DetailView):
    model = Post
    template_name = "blog/post_detail.html"
    context_object_name = "post"


class PostCreateView(LoginRequiredMixin, CreateView):
    model = Post
    form_class = PostForm
    template_name = "blog/post_form.html"

    def form_valid(self, form):
        form.instance.author = self.request.user
        return super().form_valid(form)


class PostUpdateView(LoginRequiredMixin, AuthorRequiredMixin, UpdateView):
    model = Post
    form_class = PostForm
    template_name = "blog/post_form.html"


class PostDeleteView(LoginRequiredMixin, AuthorRequiredMixin, DeleteView):
    model = Post
    template_name = "blog/post_confirm_delete.html"
    success_url = reverse_lazy("blog:post-list")
```

ตอนนี้ถ้า user B พยายามยิง `POST /blog/my-post/edit/` ไปยังโพสต์ของ user A ระบบจะตอบ
**403 Forbidden** ทันที โดยไม่ query หรือ render ฟอร์มใด ๆ ให้เห็นเลย — ปลอดภัยตั้งแต่
จุดแรกที่ request เข้ามา (`dispatch()`)

---

## ขั้นตอนที่ 233: ลำดับการวาง Mixin สำคัญอย่างไร

### 233.1 กฎทอง: Mixin ต้องมาก่อน View เสมอ

จากโค้ดในขั้นตอนที่แล้ว สังเกตว่าเราเขียน:

```python
class PostUpdateView(LoginRequiredMixin, AuthorRequiredMixin, UpdateView):
    ...
```

**ไม่ใช่**

```python
class PostUpdateView(UpdateView, LoginRequiredMixin, AuthorRequiredMixin):
    ...
```

กฎทองของ Django CBV คือ: **Mixin ทุกตัวต้องอยู่ทางซ้ายของ base View class เสมอ**
(`LoginRequiredMixin`, `AuthorRequiredMixin`, ..., `UpdateView`) เหตุผลมาจากกลไก MRO
ที่เราทบทวนใน ขั้นตอนที่ 231 นั่นเอง: Python จะพิจารณา class ที่อยู่ซ้ายมือก่อนเสมอ
ถ้า `UpdateView` (ซึ่งมี `dispatch()` ของตัวเองอยู่แล้วผ่าน `View`) อยู่ซ้ายสุด
เมธอด `dispatch()` ของมันจะถูกเรียกก่อน และมันจะ **ไม่รู้จัก** logic การเช็ค permission
ใน Mixin ของเราเลย

### 233.2 ตัวอย่างที่ผิดลำดับแล้วพัง

ลองจำลองสถานการณ์นี้จริง ๆ:

```python
# ผิดลำดับ! อย่าทำแบบนี้
class BrokenPostUpdateView(UpdateView, LoginRequiredMixin, AuthorRequiredMixin):
    model = Post
    form_class = PostForm
    template_name = "blog/post_form.html"


print([cls.__name__ for cls in BrokenPostUpdateView.__mro__])
```

ผลลัพธ์:

```
['BrokenPostUpdateView', 'UpdateView', 'SingleObjectTemplateResponseMixin',
 'TemplateResponseMixin', 'BaseUpdateView', 'ModelFormMixin', 'FormMixin',
 'ContextMixin', 'ProcessFormView', 'View', 'LoginRequiredMixin', 'AccessMixin',
 'AuthorRequiredMixin', 'object']
```

สังเกตว่า `View` (ซึ่งมี `dispatch()` เวอร์ชันมาตรฐานที่แค่เลือกเรียก `get()`/`post()`
ตาม HTTP method โดยไม่เช็ค permission ใด ๆ) อยู่ **ก่อน** `LoginRequiredMixin` และ
`AuthorRequiredMixin` ใน MRO ผลคือเมื่อ Django เรียก `dispatch()`:

1. Python มองหา `dispatch` ตามลำดับ MRO เจอที่ `View.dispatch()` (เพราะ
   `UpdateView`/`ProcessFormView` ไม่ได้ override `dispatch()` เอง)
2. `View.dispatch()` เรียก `self.get()`/`self.post()` ตรง ๆ โดยไม่เคยเรียก
   `super().dispatch()` ไปหา `LoginRequiredMixin.dispatch()` หรือ
   `AuthorRequiredMixin.dispatch()` เลย
3. **ผลลัพธ์**: ผู้ใช้ที่ไม่ได้ล็อกอิน หรือไม่ใช่เจ้าของโพสต์ สามารถแก้ไขโพสต์ของคนอื่น
   ได้ตามสบาย ทั้งที่โค้ด Mixin ถูกเขียนไว้ถูกต้องทุกอย่าง! **นี่คือช่องโหว่ความปลอดภัย
   ที่อันตรายที่สุดแบบหนึ่งที่เกิดจากความเข้าใจ MRO ผิดพลาด**

### 233.3 ทำไมถึงพัง: อ่าน MRO และ `super()` chain อีกครั้ง

จำหลักจาก ขั้นตอนที่ 231.2 ได้ไหมว่า `super()` เรียก "ตัวถัดไปใน MRO chain" ไม่ใช่
"parent ของตัวเอง" — ปัญหาของโค้ดที่ผิดลำดับคือ **ไม่มีใครเรียก `super().dispatch()`
มาถึง `LoginRequiredMixin` เลยตั้งแต่แรก** เพราะ MRO วิ่งไปเจอ `View.dispatch()` ก่อน
แล้ว `View.dispatch()` (ซึ่งเป็นจุดสิ้นสุดของ chain ที่แท้จริง ไม่เรียก `super()` ต่อ
ไปยัง Mixin ที่อยู่ *หลัง* มันใน MRO) จบการทำงานไปเลย

เทียบกับเวอร์ชันที่ถูกต้อง:

```python
class PostUpdateView(LoginRequiredMixin, AuthorRequiredMixin, UpdateView):
    ...

print([cls.__name__ for cls in PostUpdateView.__mro__])
```

```
['PostUpdateView', 'LoginRequiredMixin', 'AccessMixin', 'AuthorRequiredMixin',
 'UpdateView', 'SingleObjectTemplateResponseMixin', 'TemplateResponseMixin',
 'BaseUpdateView', 'ModelFormMixin', 'FormMixin', 'ContextMixin',
 'ProcessFormView', 'View', 'object']
```

คราวนี้ `LoginRequiredMixin` มาก่อน จึงถูกเรียกก่อน แล้ว `super().dispatch()` ของมัน
เรียก `AuthorRequiredMixin.dispatch()` ต่อ (เพราะเป็นตัวถัดไปใน MRO) ก่อนจะไปจบที่
`View.dispatch()` ในที่สุด **ทุก Mixin ในสายได้ทำงานครบตามลำดับที่ตั้งใจไว้**

### 233.4 ตารางสรุป: ลำดับที่ถูกและผิด

| การจัดลำดับ | ผลลัพธ์ | เหตุผล |
|---|---|---|
| `(LoginRequiredMixin, AuthorRequiredMixin, UpdateView)` | ✅ ถูกต้อง | Mixin ทั้งคู่อยู่ก่อน View, `super()` chain ไหลผ่านทุกตัว |
| `(AuthorRequiredMixin, LoginRequiredMixin, UpdateView)` | ⚠️ ทำงานได้ แต่ผิด priority | เช็ค author ก่อนเช็ค login → ถ้ายังไม่ล็อกอิน `request.user` เป็น `AnonymousUser` อาจ error หรือ error message สับสน |
| `(UpdateView, LoginRequiredMixin, AuthorRequiredMixin)` | ❌ พัง (ช่องโหว่ความปลอดภัย) | `View.dispatch()` ถูกเจอก่อน ไม่เรียก Mixin เลย |
| `(LoginRequiredMixin, UpdateView, AuthorRequiredMixin)` | ❌ พัง (Mixin ตัวท้ายไม่ทำงาน) | `super().dispatch()` ของ `LoginRequiredMixin` ไปเจอ `View.dispatch()` ก่อนถึง `AuthorRequiredMixin` |

**กฎเหล็กที่ต้องจำ**: เขียน Mixin ที่ override `dispatch()` เรียงจาก
**เช็คทั่วไปที่สุดไปหาเฉพาะเจาะจงที่สุด** แล้ววาง generic View (`ListView`,
`UpdateView`, ...) **ไว้ขวาสุดเสมอ** ไม่มีข้อยกเว้น

---

## ขั้นตอนที่ 234: `UserPassesTestMixin` และ `PermissionRequiredMixin` ของ Django เอง

### 234.1 ทำไมไม่เขียน `dispatch()` เองตลอดไป

การ override `dispatch()` ตรง ๆ แบบที่ทำใน ขั้นตอนที่ 232 ใช้งานได้ แต่ Django มี Mixin
สำเร็จรูปที่ครอบคลุม pattern "ตรวจสอบเงื่อนไขก่อนอนุญาตให้เข้า view" ไว้ให้แล้ว ทำให้
โค้ดเราสั้นลงและสอดคล้องกับ convention ของ Django มากขึ้น ทั้งสองตัวอยู่ใน
`django.contrib.auth.mixins` และล้วนสืบทอดจาก `AccessMixin` ตัวเดียวกัน (ซึ่งเป็นตัวที่
ให้ความสามารถ `raise_exception`, `login_url`, `permission_denied_message` ฯลฯ)

### 234.2 เขียน `AuthorRequiredMixin` ใหม่ด้วย `UserPassesTestMixin`

```python
# blog/mixins.py
from django.contrib.auth.mixins import LoginRequiredMixin, UserPassesTestMixin


class AuthorRequiredMixin(LoginRequiredMixin, UserPassesTestMixin):
    """
    เวอร์ชันที่ใช้ UserPassesTestMixin ของ Django แทนการ override dispatch() เอง
    เขียนสั้นกว่า และสอดคล้องกับแนวทางมาตรฐานของ Django
    """

    def test_func(self):
        # UserPassesTestMixin จะเรียกเมธอดนี้ให้อัตโนมัติใน dispatch() ของมันเอง
        post = self.get_object()
        return post.author_id == self.request.user.id

    def handle_no_permission(self):
        # override ได้ถ้าต้องการ custom message/behavior ตอนไม่ผ่าน test_func()
        from django.core.exceptions import PermissionDenied

        if self.request.user.is_authenticated:
            raise PermissionDenied("คุณไม่ใช่เจ้าของโพสต์นี้ จึงไม่มีสิทธิ์ดำเนินการ")
        return super().handle_no_permission()
```

สังเกตว่าเราไม่ต้อง override `dispatch()` เองอีกต่อไปเลย — `UserPassesTestMixin` มี
`dispatch()` ของตัวเองที่เรียก `test_func()` ให้อัตโนมัติ ถ้าคืนค่า `False` มันจะเรียก
`handle_no_permission()` ให้เอง (ซึ่งเราปรับแต่งได้ตามต้องการ) นี่คือตัวอย่างที่ดีว่า
เมื่อเข้าใจกลไก Mixin แล้ว เราสามารถเลือกได้ว่าจะ "เขียนเองทั้งหมด" หรือ "ต่อยอดจาก
Mixin มาตรฐานที่ Django ให้มา"

**สำคัญ**: ลำดับ `(LoginRequiredMixin, UserPassesTestMixin)` ก็ยังคงต้องถูกต้องตามกฎ
MRO เหมือน ขั้นตอนที่ 233 ทุกประการ — `LoginRequiredMixin` ควรอยู่ก่อนเสมอเพื่อเช็ค
login ก่อนที่จะไปเช็คเงื่อนไขเฉพาะทางใน `test_func()`

### 234.3 `PermissionRequiredMixin` เบื้องต้น

อีกทางเลือกหนึ่งคือ `PermissionRequiredMixin` ซึ่งใช้ระบบ **Django permission
framework** (ที่ผูกกับ `auth_permission` table ในฐานข้อมูล ไม่ใช่การเทียบ field
ตรง ๆ แบบ `test_func()`):

```python
from django.contrib.auth.mixins import LoginRequiredMixin, PermissionRequiredMixin


class PostPublishView(LoginRequiredMixin, PermissionRequiredMixin, UpdateView):
    model = Post
    fields = ["status"]
    permission_required = "blog.can_publish_post"  # ต้องกำหนดใน Meta.permissions ของ Model
    template_name = "blog/post_publish.html"
```

ต่างจาก `AuthorRequiredMixin` ตรงที่ `PermissionRequiredMixin` ไม่ได้เช็คว่า
"เป็นเจ้าของ object นี้หรือไม่" แต่เช็คว่า "user คนนี้มี permission ชื่อนี้อยู่ในระบบ
หรือไม่" (permission อาจถูกให้ผ่าน Group ก็ได้) เหมาะกับสิทธิ์ระดับ role เช่น
บรรณาธิการ (editor) ที่เผยแพร่โพสต์ของใครก็ได้ ต่างจากสิทธิ์ระดับ "เจ้าของ" ที่เราทำใน
`AuthorRequiredMixin`

### 234.4 ตารางเปรียบเทียบตัวเลือกทั้งหมดสำหรับควบคุมการเข้าถึง CBV

| Mixin | ตรวจสอบอะไร | เหมาะกับ | ต้องเขียนเพิ่ม |
|---|---|---|---|
| `LoginRequiredMixin` | ล็อกอินแล้วหรือยัง | ทุก view ที่ต้องล็อกอินก่อนใช้งาน | ไม่ต้อง |
| `PermissionRequiredMixin` | มี Django permission ที่กำหนดหรือไม่ | สิทธิ์ระดับ role/กลุ่ม เช่น editor, staff | ต้องกำหนด `permission_required` |
| `UserPassesTestMixin` | เงื่อนไข custom ใด ๆ ผ่าน `test_func()` | เงื่อนไขเฉพาะทาง เช่น ตรวจสอบความเป็นเจ้าของ | ต้องเขียน `test_func()` เอง |
| Custom Mixin (override `dispatch()`) | อะไรก็ได้ที่เขียนเอง | เมื่อต้องการควบคุมทุกรายละเอียด เช่น logging ร่วมด้วย | ต้องเขียนเองทั้งหมด รวมถึงเรียก `super()` |

**หมายเหตุสำคัญ**: Part นี้แนะนำ `UserPassesTestMixin` และ `PermissionRequiredMixin`
เพียงพอให้เข้าใจภาพรวมและนำไปใช้ได้ทันที ส่วนรายละเอียดเชิงลึกของระบบ Permission,
Group, และการสร้าง custom permission เต็มรูปแบบจะอยู่ใน **Part 033 (Authorization
และ Permission System)**

---

## ขั้นตอนที่ 235: Override `dispatch()` เพื่อทำงานก่อนทุก HTTP Method

### 235.1 `dispatch()` คือประตูด่านแรกของทุก CBV

จาก MRO ที่เห็นมาตลอด 4 ขั้นตอนที่ผ่านมา `dispatch()` คือเมธอดแรกสุดที่ Django เรียก
เมื่อ request เข้ามาถึง CBV (ผ่าน `as_view()`) หน้าที่ของมันคือดูค่า `request.method`
แล้วเรียกเมธอดที่ตรงกัน (`get()`, `post()`, `put()`, `delete()`, ...) การ override
`dispatch()` จึงเป็นจุดที่สมบูรณ์แบบที่สุดสำหรับ logic ที่ต้อง **ทำงานก่อนทุก HTTP
method โดยไม่สนใจว่าเป็น GET หรือ POST**

ตัวอย่าง use case ที่พบบ่อยในงานจริง: Logging, Rate limiting, Feature flag check,
Maintenance mode check, การบันทึก analytics

### 235.2 `LoggingMixin`: บันทึก log ทุก request ที่เข้ามาที่ view

```python
# blog/mixins.py
import logging
import time

logger = logging.getLogger("blog.views")


class LoggingMixin:
    """บันทึก log ว่า view ไหนถูกเรียก โดยใคร ใช้เวลาเท่าไหร่ และผลลัพธ์เป็นอย่างไร"""

    def dispatch(self, request, *args, **kwargs):
        start_time = time.monotonic()
        user_display = request.user if request.user.is_authenticated else "anonymous"

        logger.info(
            "เริ่ม %s %s โดย %s", request.method, request.path, user_display
        )

        response = super().dispatch(request, *args, **kwargs)

        duration_ms = (time.monotonic() - start_time) * 1000
        logger.info(
            "จบ %s %s -> status %s (%.1f ms)",
            request.method,
            request.path,
            response.status_code,
            duration_ms,
        )
        return response
```

สังเกตรูปแบบสำคัญ: เราเรียก `super().dispatch(request, *args, **kwargs)` แล้ว **เก็บ
ค่า return ไว้ในตัวแปร** (`response`) แทนที่จะ `return super().dispatch(...)` ทันที
เพราะเราต้องการทำงานเพิ่มเติม **หลังจาก** view ทำงานเสร็จด้วย (วัดเวลาที่ใช้ และ log
status code) นี่คือรูปแบบ "before + after" ที่ตรงกับแนวคิด decorator ใน Part 002
ขั้นตอนที่ 14 ทุกประการ เพียงแต่ทำในรูปแบบ Mixin แทน

### 235.3 ผสาน Logging + Permission ใน Mixin เดียวกัน

```python
class PostUpdateView(LoggingMixin, LoginRequiredMixin, AuthorRequiredMixin, UpdateView):
    model = Post
    form_class = PostForm
    template_name = "blog/post_form.html"
```

ลำดับตรงนี้สำคัญตามหลักการที่เรียนใน ขั้นตอนที่ 233: `LoggingMixin` ควรอยู่ **ซ้ายสุด**
เพราะเราต้องการ log **ทุก** request รวมถึง request ที่ถูกบล็อกโดย `LoginRequiredMixin`
หรือ `AuthorRequiredMixin` ด้วย (เพื่อ audit trail ที่สมบูรณ์ — รู้ว่าใครพยายามเข้าถึง
อะไรบ้าง แม้จะถูกปฏิเสธก็ตาม) ถ้าวาง `LoggingMixin` ไว้หลัง permission mixin
request ที่ถูกบล็อกจะไม่ถูก log เลย เพราะ `PermissionDenied` จะถูก raise ขึ้นก่อนที่
`super().dispatch()` จะไปถึง `LoggingMixin`

### 235.4 ข้อควรระวัง: อย่าลืม `super().dispatch()`

ข้อผิดพลาดที่พบบ่อยที่สุดเมื่อเขียน Mixin ที่ override `dispatch()`:

```python
# ผิด! ลืมเรียก super().dispatch()
class BrokenLoggingMixin:
    def dispatch(self, request, *args, **kwargs):
        logger.info("มี request เข้ามา")
        # ไม่มี return super().dispatch(...) !!
```

ถ้าไม่ `return super().dispatch(request, *args, **kwargs)` เมธอดนี้จะคืนค่า `None`
กลับไป แล้ว Django (ผ่าน WSGI/ASGI handler) จะ error ทันทีด้วย:

```
ValueError: The view blog.views.PostUpdateView didn't return an HttpResponse object.
It returned None instead.
```

**กฎเหล็ก**: Mixin ที่ override `dispatch()` ต้อง `return` ผลลัพธ์ของ
`super().dispatch(...)` เสมอ ไม่ว่าจะเป็น flow ปกติหรือ flow ที่ถูกบล็อก (ซึ่งในกรณี
ถูกบล็อกจะ `return` ค่าอื่นแทน เช่น `redirect(...)` หรือ `raise` exception ไปเลย
ไม่ใช่ปล่อยให้ตกไปที่ท้ายฟังก์ชันแบบไม่มี return)

---

## ขั้นตอนที่ 236: สร้าง Base ListView/DetailView ที่ Inject Context ร่วมกัน

### 236.1 ปัญหา: sidebar หมวดหมู่ต้องแสดงในหลายหน้า

สมมติว่าทุกหน้าของ blog (`PostListView`, `PostDetailView`, และหน้าอื่น ๆ ในอนาคต เช่น
`CategoryDetailView`) ต้องแสดง **sidebar รายการหมวดหมู่ทั้งหมดพร้อมจำนวนโพสต์** และ
**5 โพสต์ล่าสุด** ถ้าเขียน `get_context_data()` ซ้ำในทุก view จะผิดหลัก DRY ที่เรียนมา
ตั้งแต่ Part 001 (ขั้นตอนที่ 2.3) ทันที

### 236.2 `SidebarContextMixin`

```python
# blog/mixins.py
from django.db.models import Count, Q

from .models import Category, Post


class SidebarContextMixin:
    """
    เพิ่ม context ที่ใช้ร่วมกันในทุกหน้าของ blog: รายการหมวดหมู่พร้อมจำนวนโพสต์
    และโพสต์ล่าสุด 5 รายการ ใช้กับ view ใดก็ได้ที่มี get_context_data() มาตรฐาน
    (เช่น ListView, DetailView ที่สืบทอดจาก ContextMixin)
    """

    sidebar_recent_count = 5

    def get_context_data(self, **kwargs):
        context = super().get_context_data(**kwargs)
        context["sidebar_categories"] = Category.objects.annotate(
            post_count=Count("posts", filter=Q(posts__status=Post.Status.PUBLISHED))
        ).order_by("-post_count")
        context["sidebar_recent_posts"] = Post.objects.filter(
            status=Post.Status.PUBLISHED
        ).order_by("-created_at")[: self.sidebar_recent_count]
        return context
```

จุดสำคัญที่สุดของโค้ดนี้คือรูปแบบ:

```python
def get_context_data(self, **kwargs):
    context = super().get_context_data(**kwargs)   # 1. ดึง context เดิมจาก MRO chain มาก่อน
    context["key"] = value                           # 2. เพิ่มข้อมูลของเราเข้าไป
    return context                                    # 3. คืนค่า dict ที่รวมกันแล้ว
```

รูปแบบนี้คือ "cooperative extension" เหมือนที่เราเห็นใน `dispatch()` มาตลอด Part นี้
เพียงแต่ทำงานกับ `dict` (context) แทนที่จะเป็น `HttpResponse` — Mixin **ไม่เคยแทนที่**
context เดิมทั้งหมด แต่ **ต่อยอด** จากสิ่งที่ Mixin/View ตัวก่อนหน้าใน MRO เตรียมไว้แล้ว

### 236.3 `BaseBlogListView` / `BaseBlogDetailView`

เพื่อไม่ต้องเขียน `SidebarContextMixin` ซ้ำหน้า class ทุกครั้ง เราสร้าง Base View
ของตัวเองที่รวม Mixin ที่ใช้ร่วมกันไว้ในที่เดียว:

```python
# blog/views_base.py
from django.views.generic import DetailView, ListView

from .mixins import SidebarContextMixin


class BaseBlogListView(SidebarContextMixin, ListView):
    """ListView พื้นฐานของทุกหน้าใน blog app ที่ต้องมี sidebar"""

    paginate_by = 10


class BaseBlogDetailView(SidebarContextMixin, DetailView):
    """DetailView พื้นฐานของทุกหน้าใน blog app ที่ต้องมี sidebar"""

    pass
```

นี่คือ pattern ที่เรียกว่า **"Abstract Base View"** เทียบเท่ากับ `abstract = True`
ของ Model ที่เราเรียนใน Part 002 (ขั้นตอนที่ 13.4, `TimestampedModel`) — สร้าง class
กลางที่รวมพฤติกรรมร่วม แล้วให้ view เฉพาะทางสืบทอดต่อไปอีกที

### 236.4 นำไปใช้กับ Post views ทั้งหมด

```python
# blog/views.py
from .views_base import BaseBlogDetailView, BaseBlogListView


class PostListView(BaseBlogListView):
    model = Post
    template_name = "blog/post_list.html"
    context_object_name = "posts"

    def get_queryset(self):
        return Post.objects.filter(status=Post.Status.PUBLISHED).select_related(
            "author", "category"
        )


class PostDetailView(BaseBlogDetailView):
    model = Post
    template_name = "blog/post_detail.html"
    context_object_name = "post"
```

ตอนนี้ template `post_list.html` และ `post_detail.html` สามารถใช้ตัวแปร
`{{ sidebar_categories }}` และ `{{ sidebar_recent_posts }}` ได้ทันทีโดยไม่ต้องเขียน
โค้ด query ซ้ำเลยแม้แต่บรรทัดเดียวใน view ทั้งสองตัว:

```html
<!-- templates/blog/_sidebar.html -->
<aside class="sidebar">
    <h3>หมวดหมู่</h3>
    <ul>
        {% for category in sidebar_categories %}
            <li>{{ category.name }} ({{ category.post_count }})</li>
        {% endfor %}
    </ul>

    <h3>โพสต์ล่าสุด</h3>
    <ul>
        {% for post in sidebar_recent_posts %}
            <li><a href="{{ post.get_absolute_url }}">{{ post.title }}</a></li>
        {% endfor %}
    </ul>
</aside>
```

ถ้าในอนาคตต้องการเพิ่ม `CategoryDetailView` หรือ `TagListView` ก็เพียงสืบทอดจาก
`BaseBlogListView`/`BaseBlogDetailView` เท่านั้น sidebar จะติดมาให้อัตโนมัติ — นี่คือ
พลังที่แท้จริงของการออกแบบ Mixin ให้ดีตั้งแต่ต้น

---

## ขั้นตอนที่ 237: สร้าง `JsonResponseMixin` สำหรับ View ที่ตอบกลับเป็น JSON

### 237.1 เมื่อไหร่ต้องตอบ JSON แทน HTML

ในงานจริง view เดียวกันมักต้องรองรับทั้งการเรียกแบบปกติ (browser ขอ HTML) และการเรียก
แบบ AJAX/fetch จาก JavaScript (ขอ JSON) เช่น ปุ่ม "กดถูกใจ" ที่ไม่ต้อง reload หน้า
หรือ endpoint สำหรับดึงรายละเอียดโพสต์ไปแสดงใน modal โดยไม่ reload

### 237.2 เขียน `JsonResponseMixin`

```python
# blog/mixins.py
from django.http import JsonResponse


class JsonResponseMixin:
    """
    Mixin สำหรับ View ที่ต้องการตอบกลับเป็น JSON แทน HTML
    ใช้ผสมกับ django.views.generic.View ธรรมดา (ไม่ต้องมี Template ใด ๆ)
    """

    def render_to_json_response(self, context, **response_kwargs):
        return JsonResponse(self.get_data(context), **response_kwargs)

    def get_data(self, context):
        """
        override เมธอดนี้เพื่อกำหนดว่าจะแปลง context เป็น dict สำหรับ JSON อย่างไร
        ค่า default คือคืน context ตรง ๆ (ใช้ได้เฉพาะกรณีที่ทุกค่าเป็น JSON-serializable)
        """
        return context
```

### 237.3 ผสมกับ `View` ธรรมดา: `PostDetailJsonView`

```python
# blog/views.py
from django.http import JsonResponseNotAllowed
from django.shortcuts import get_object_or_404
from django.views import View

from .mixins import JsonResponseMixin
from .models import Post


class PostDetailJsonView(JsonResponseMixin, View):
    """
    Endpoint สำหรับดึงรายละเอียดโพสต์เป็น JSON เช่น GET /blog/api/posts/<slug>/
    ใช้ View ธรรมดา (ไม่ใช่ DetailView) เพราะไม่ต้อง render template ใด ๆ เลย
    """

    def get(self, request, slug, *args, **kwargs):
        post = get_object_or_404(Post, slug=slug, status=Post.Status.PUBLISHED)
        data = {
            "id": post.id,
            "title": post.title,
            "slug": post.slug,
            "content": post.content,
            "author": post.author.get_full_name() or post.author.username,
            "category": post.category.name if post.category else None,
            "created_at": post.created_at.isoformat(),
        }
        return self.render_to_json_response(data)
```

สังเกตว่า `PostDetailJsonView` สืบทอดจาก `View` ตรง ๆ ไม่ใช่ `DetailView` เพราะเราไม่
ต้องการความสามารถของ `SingleObjectMixin`/`TemplateResponseMixin` เลย (ไม่มี template
ให้ render) — นี่คือตัวอย่างที่ดีว่า **ไม่ใช่ทุก view ที่ต้องเริ่มจาก Generic CBV เสมอไป**
บางครั้ง `View` เปล่า ๆ ผสมกับ Mixin ที่เขียนเองคือทางเลือกที่สะอาดที่สุด

อย่าลืมเพิ่ม URL:

```python
# blog/urls.py
from django.urls import path

from . import views

app_name = "blog"

urlpatterns = [
    path("", views.PostListView.as_view(), name="post-list"),
    path("<slug:slug>/", views.PostDetailView.as_view(), name="post-detail"),
    path("create/", views.PostCreateView.as_view(), name="post-create"),
    path("<slug:slug>/edit/", views.PostUpdateView.as_view(), name="post-update"),
    path("<slug:slug>/delete/", views.PostDeleteView.as_view(), name="post-delete"),
    path("api/posts/<slug:slug>/", views.PostDetailJsonView.as_view(), name="post-detail-json"),
]
```

### 237.4 `AjaxableResponseMixin`: ผสม JSON เข้ากับ `CreateView`/`UpdateView` เดิม

บางครั้งเราไม่อยากเขียน endpoint JSON แยกต่างหาก แต่อยากให้ `PostCreateView` เดิม
ตอบกลับเป็น JSON เมื่อถูกเรียกผ่าน AJAX (มี header `X-Requested-With`) และตอบกลับ
เป็น redirect ปกติเมื่อถูกเรียกจากฟอร์ม HTML ธรรมดา:

```python
# blog/mixins.py
from django.http import JsonResponse


class AjaxableResponseMixin:
    """
    ผสมกับ CreateView/UpdateView เพื่อให้ตอบกลับเป็น JSON เมื่อ request มาจาก AJAX
    (ตรวจสอบจาก header X-Requested-With) และตอบกลับแบบ HTML/redirect ตามปกติในกรณีอื่น
    """

    def form_invalid(self, form):
        response = super().form_invalid(form)
        if self.request.headers.get("x-requested-with") == "XMLHttpRequest":
            return JsonResponse(form.errors, status=400)
        return response

    def form_valid(self, form):
        response = super().form_valid(form)
        if self.request.headers.get("x-requested-with") == "XMLHttpRequest":
            data = {"pk": self.object.pk, "url": self.object.get_absolute_url()}
            return JsonResponse(data)
        return response
```

```python
class PostCreateView(AjaxableResponseMixin, LoginRequiredMixin, CreateView):
    model = Post
    form_class = PostForm
    template_name = "blog/post_form.html"

    def form_valid(self, form):
        form.instance.author = self.request.user
        return super().form_valid(form)
```

รูปแบบนี้เรียกว่า **"content negotiation แบบง่าย"** — view เดียวกันตอบกลับต่างชนิด
ข้อมูลตาม request ที่เข้ามา นี่คือ pattern พื้นฐานที่ Django REST Framework (ที่จะ
เรียนใน Phase 9) ทำแบบเต็มรูปแบบและซับซ้อนกว่านี้มาก

---

## ขั้นตอนที่ 238: การเขียน Test สำหรับ Class-Based View เบื้องต้น

### 238.1 เครื่องมือสองตัว: `self.client` และ `RequestFactory`

Django มีเครื่องมือทดสอบ view หลัก 2 แบบที่ควรรู้จักตั้งแต่ตอนนี้ (จะเจาะลึกเต็มรูปแบบ
ใน **Phase 7: Testing**):

| เครื่องมือ | ทำงานอย่างไร | เหมาะกับ |
|---|---|---|
| `self.client` (`django.test.Client`) | จำลอง HTTP request เต็มรูปแบบ ผ่าน middleware, URL routing ทั้งหมด | ทดสอบพฤติกรรมแบบ end-to-end ของทั้งหน้า (integration test) |
| `RequestFactory` (`django.test.RequestFactory`) | สร้าง `HttpRequest` object เปล่า ๆ โดยไม่ผ่าน middleware/URL routing | ทดสอบ view หรือ Mixin แบบแยกส่วน (unit test) เร็วกว่าเพราะข้าม overhead |

### 238.2 ทดสอบ Permission ของ `AuthorRequiredMixin` ด้วย `self.client`

```python
# blog/tests/test_views.py
from django.contrib.auth import get_user_model
from django.test import TestCase
from django.urls import reverse

from blog.models import Post

User = get_user_model()


class PostUpdateViewPermissionTests(TestCase):
    @classmethod
    def setUpTestData(cls):
        # setUpTestData รันครั้งเดียวต่อทั้ง TestCase class (เร็วกว่า setUp ที่รันทุก test)
        cls.author = User.objects.create_user(username="author", password="pass1234")
        cls.other_user = User.objects.create_user(username="other", password="pass1234")
        cls.post = Post.objects.create(
            title="โพสต์ทดสอบ",
            slug="post-tests",
            content="เนื้อหาทดสอบ",
            author=cls.author,
            status=Post.Status.PUBLISHED,
        )

    def test_anonymous_user_redirected_to_login(self):
        url = reverse("blog:post-update", kwargs={"slug": self.post.slug})
        response = self.client.get(url)
        self.assertEqual(response.status_code, 302)
        self.assertIn("/accounts/login/", response.url)

    def test_non_author_gets_403(self):
        self.client.login(username="other", password="pass1234")
        url = reverse("blog:post-update", kwargs={"slug": self.post.slug})
        response = self.client.get(url)
        self.assertEqual(response.status_code, 403)

    def test_author_can_access_update_form(self):
        self.client.login(username="author", password="pass1234")
        url = reverse("blog:post-update", kwargs={"slug": self.post.slug})
        response = self.client.get(url)
        self.assertEqual(response.status_code, 200)
        self.assertContains(response, self.post.title)
```

สังเกตว่าการทดสอบผ่าน `self.client` จะทดสอบ **ทั้งระบบจริง** ตั้งแต่ URL routing,
middleware (รวมถึง authentication middleware ที่ทำให้ `request.user` มีค่าถูกต้อง),
ไปจนถึง MRO ทั้งสายของ `PostUpdateView` — เป็นการยืนยันว่าลำดับ Mixin ที่เราจัดไว้ใน
ขั้นตอนที่ 233 ทำงานถูกต้องจริงในสถานการณ์ที่ใกล้เคียงของจริงที่สุด

### 238.3 ทดสอบแบบ Unit ด้วย `RequestFactory`

```python
# blog/tests/test_mixins.py
from django.contrib.auth import get_user_model
from django.core.exceptions import PermissionDenied
from django.test import RequestFactory, TestCase

from blog.models import Post
from blog.views import PostUpdateView

User = get_user_model()


class AuthorRequiredMixinUnitTests(TestCase):
    @classmethod
    def setUpTestData(cls):
        cls.author = User.objects.create_user(username="author", password="pass1234")
        cls.other_user = User.objects.create_user(username="other", password="pass1234")
        cls.post = Post.objects.create(
            title="โพสต์ทดสอบ", slug="unit-test-post", content="...", author=cls.author,
        )
        cls.factory = RequestFactory()

    def test_dispatch_raises_permission_denied_for_non_author(self):
        request = self.factory.get(f"/blog/{self.post.slug}/edit/")
        request.user = self.other_user

        view = PostUpdateView()
        view.request = request
        view.kwargs = {"slug": self.post.slug}
        view.args = ()

        with self.assertRaises(PermissionDenied):
            view.dispatch(request, slug=self.post.slug)
```

ความต่างที่สำคัญ: `RequestFactory` ไม่รัน middleware ให้เราอัตโนมัติ ดังนั้นเราต้อง
กำหนด `request.user` **ด้วยตัวเอง** (โดยปกติ `AuthenticationMiddleware` เป็นคนทำสิ่งนี้
ให้ตอน request จริงเข้ามา) การทดสอบแบบนี้เร็วกว่าและ "แคบ" กว่า (ทดสอบเฉพาะ logic ใน
`dispatch()` โดยไม่แตะ URL routing/middleware อื่น) เหมาะกับการทดสอบ Mixin ที่ซับซ้อน
แบบแยกส่วนจากกลไกอื่นของ Django

### 238.4 เกริ่นสิ่งที่จะเจาะลึกใน Phase 7

ตัวอย่างข้างต้นเป็นเพียงจุดเริ่มต้น การเขียน Test อย่างมืออาชีพยังมีอีกหลายเรื่องที่
Part นี้ยังไม่ครอบคลุม เช่น `pytest-django`, `factory_boy` สำหรับสร้าง test data,
Coverage measurement, Mocking external services, และ Test ของ Form/Template แยกกัน
— ทั้งหมดนี้จะถูกเจาะลึกแบบเต็มรูปแบบใน **Phase 7: Testing (Part 61-70)**
ตอนนี้ขอให้คุณคุ้นเคยกับแนวคิดพื้นฐานว่า **CBV ก็คือ class ธรรมดาที่ทดสอบได้เหมือน
class อื่น ๆ ทุกประการ** เพียงแต่ต้องเข้าใจว่ามันถูกเรียกใช้งานผ่าน `as_view()` และ
`dispatch()` อย่างไร

---

## ขั้นตอนที่ 239: กรอบการตัดสินใจ FBV vs CBV แบบมืออาชีพ

### 239.1 ทบทวนจาก Part 021

ใน **Part 021** เราแนะนำหลักการคร่าว ๆ ว่า CBV เหมาะกับ logic ที่ทำตาม pattern
มาตรฐาน (list, detail, create, update, delete) ส่วน FBV เหมาะกับ logic ที่ซับซ้อน
เฉพาะทาง หลังจากที่เราเข้าใจ Mixin และ MRO อย่างลึกซึ้งแล้วใน Part นี้ เราสามารถ
ตั้งกรอบการตัดสินใจที่ละเอียดและเป็นมืออาชีพกว่าเดิมได้

### 239.2 กรอบการตัดสินใจ 5 คำถาม

เมื่อต้องเขียน view ใหม่สักตัว ให้ถามคำถามเหล่านี้ตามลำดับ:

1. **View นี้ทำ CRUD มาตรฐานหรือไม่?** (list, detail, create, update, delete โดยไม่มี
   logic พิเศษมาก) → ถ้าใช่ ใช้ Generic CBV ทันที (`ListView`, `CreateView`, ...)
2. **ต้องใช้ permission/behavior ร่วมกับ view อื่นหลายตัวหรือไม่?** (เช่น
   `AuthorRequiredMixin`, `SidebarContextMixin`) → ถ้าใช่ นี่คือสัญญาณชัดเจนว่าควรใช้
   CBV + Mixin เพราะจะ reuse โค้ดได้สะดวกกว่า FBV + decorator มาก
3. **Logic ของ view มีหลาย HTTP method ที่ทำงานต่างกันมากและซับซ้อน** (เช่น รับทั้ง
   GET, POST, PUT พร้อม logic เฉพาะที่ไม่เกี่ยวข้องกันเลย) → พิจารณา FBV เพราะการยัด
   logic ที่ไม่เกี่ยวกันลงใน `get()`/`post()` ของ CBV เดียวบางครั้งอ่านยากกว่าฟังก์ชัน
   ที่แยก `if request.method == "POST":` ชัดเจน
4. **ทีมของคุณคุ้นเคยกับ OOP/MRO ดีแค่ไหน?** ถ้าทีมส่วนใหญ่เพิ่งเริ่มต้น การ debug
   MRO ที่ซับซ้อน (แบบที่เห็นใน ขั้นตอนที่ 233) อาจทำให้เสียเวลามากกว่าประโยชน์ที่ได้
   ในระยะสั้น — แต่ในระยะยาวของโปรเจกต์ใหญ่ Mixin แทบจะเป็นสิ่งจำเป็นเพื่อไม่ให้โค้ด
   ซ้ำซ้อนมหาศาล
5. **นี่คือ one-off view ที่ไม่มีวันถูกใช้ pattern ซ้ำหรือไม่?** (เช่น endpoint พิเศษ
   สำหรับ webhook หนึ่งตัว, health check endpoint) → FBV มักจะกระชับและอ่านง่ายกว่า
   สำหรับ view เดี่ยว ๆ ที่ไม่ต้องแชร์ logic กับใคร

### 239.3 ตัวอย่างสถานการณ์จริง 5 แบบ

| สถานการณ์ | คำแนะนำ | เหตุผล |
|---|---|---|
| หน้ารายการสินค้าที่มี filter, search, pagination มาตรฐาน | **CBV** (`ListView` + Mixin) | ตรงกับ pattern ของ `MultipleObjectMixin` เป๊ะ, reuse ได้ |
| Endpoint รับ Webhook จาก Stripe payment gateway | **FBV** | Logic เฉพาะทางมาก, ไม่มี pattern ร่วมกับ view อื่น, ต้อง `@csrf_exempt` เดี่ยว ๆ |
| ระบบ CRUD ของ Post/Comment/Category ที่ทุกตัวต้องเช็คว่าเป็นเจ้าของก่อนแก้ไข | **CBV + Custom Mixin** | ตรงกับสิ่งที่ Part นี้สอนเป๊ะ ๆ — `AuthorRequiredMixin` reuse ได้กับทุก Model |
| หน้า Dashboard ที่รวมข้อมูลจากหลาย Model, คำนวณสถิติซับซ้อน | **FBV** | Logic การรวมข้อมูลซับซ้อนเกินกว่าจะ fit ใน pattern ของ Generic CBV ใด ๆ |
| Multi-step form wizard (กรอกฟอร์มหลายหน้าต่อเนื่องกัน) | **CBV** (ใช้ `FormView` + session state หรือ `django-formtools`) | มี pattern มาตรฐานรองรับอยู่แล้วผ่าน library, CBV จัดการ state ได้เป็นระเบียบกว่า |

### 239.4 กฎที่ทีมระดับโลกใช้จริง

ทีมพัฒนา Django ระดับโลก (เช่นทีมที่ดูแล Instagram หรือ Mozilla ที่กล่าวถึงใน Part 001)
มักยึดหลักปฏิบัติเหล่านี้:

- **เริ่มจาก Generic CBV เสมอเมื่อทำได้** เพราะให้ความสม่ำเสมอ (consistency) ของโค้ด
  ทั้งโปรเจกต์ ทำให้นักพัฒนาใหม่เข้าใจโครงสร้างได้เร็ว
- **เขียน Mixin ที่ reuse ได้ทันทีที่เจอ logic ซ้ำเป็นครั้งที่ 2** (ไม่ต้องรอถึงครั้งที่
  3 ตามกฎ "Rule of Three" แบบดั้งเดิม เพราะ permission logic ที่ผิดพลาดมีความเสี่ยง
  ด้านความปลอดภัยสูงกว่าปกติ)
- **จำกัดความลึกของ Mixin chain ไม่ให้เกิน 3-4 ตัว** ต่อ view เดียว — ถ้าเกินกว่านี้
  มักเป็นสัญญาณว่าควรแยก logic บางส่วนออกไปเป็น Service function/class แทนที่จะยัด
  ทุกอย่างลงใน Mixin
- **เขียน docstring อธิบาย dependency ของ Mixin เสมอ** (เช่น "ต้องใช้กับ class ที่มี
  `get_object()`") เพราะ Mixin ที่ดีควรระบุ **contract** ที่ต้องการจาก class ที่นำไปใช้
  ให้ชัดเจน ไม่ใช่ให้คนอื่นต้องอ่านโค้ดแล้วเดาเอง
- **ใช้ FBV อย่างไม่เขินอาย** เมื่อ CBV ทำให้โค้ดซับซ้อนกว่าที่ควร — Django ไม่ได้บังคับ
  ว่าต้องใช้ CBV ทุกที่ ทั้งสองแบบอยู่ร่วมกันในโปรเจกต์เดียวได้อย่างสมบูรณ์แบบ

---

## ขั้นตอนที่ 240: สรุปและแบบฝึกหัด

### 240.1 รวม Mixin ทั้งหมดของ Part นี้ไว้ในที่เดียว

ก่อนนำไปประกอบกับ Post CRUD แบบสมบูรณ์ นี่คือไฟล์ `blog/mixins.py` ฉบับรวมทุกอย่างที่
เรียนมาใน Part นี้:

```python
# blog/mixins.py
import logging
import time

from django.contrib.auth.mixins import LoginRequiredMixin, UserPassesTestMixin
from django.core.exceptions import PermissionDenied
from django.db.models import Count, Q
from django.http import JsonResponse

from .models import Category, Post

logger = logging.getLogger("blog.views")


class LoggingMixin:
    """บันทึก log ทุก request ที่เข้ามาที่ view (ทำงานก่อนทุก HTTP method)"""

    def dispatch(self, request, *args, **kwargs):
        start_time = time.monotonic()
        user_display = request.user if request.user.is_authenticated else "anonymous"
        logger.info("เริ่ม %s %s โดย %s", request.method, request.path, user_display)

        response = super().dispatch(request, *args, **kwargs)

        duration_ms = (time.monotonic() - start_time) * 1000
        logger.info(
            "จบ %s %s -> status %s (%.1f ms)",
            request.method, request.path, response.status_code, duration_ms,
        )
        return response


class AuthorRequiredMixin(LoginRequiredMixin, UserPassesTestMixin):
    """อนุญาตเฉพาะ user ที่ล็อกอินแล้วและเป็นเจ้าของ object (ผ่าน field author)"""

    def test_func(self):
        obj = self.get_object()
        return obj.author_id == self.request.user.id

    def handle_no_permission(self):
        if self.request.user.is_authenticated:
            raise PermissionDenied("คุณไม่ใช่เจ้าของโพสต์นี้ จึงไม่มีสิทธิ์ดำเนินการ")
        return super().handle_no_permission()


class SidebarContextMixin:
    """เพิ่ม context หมวดหมู่และโพสต์ล่าสุดให้ทุกหน้าที่ผสม Mixin นี้"""

    sidebar_recent_count = 5

    def get_context_data(self, **kwargs):
        context = super().get_context_data(**kwargs)
        context["sidebar_categories"] = Category.objects.annotate(
            post_count=Count("posts", filter=Q(posts__status=Post.Status.PUBLISHED))
        ).order_by("-post_count")
        context["sidebar_recent_posts"] = Post.objects.filter(
            status=Post.Status.PUBLISHED
        ).order_by("-created_at")[: self.sidebar_recent_count]
        return context


class JsonResponseMixin:
    """ตอบกลับเป็น JSON แทน HTML สำหรับ View ที่ผสมกับ django.views.generic.View"""

    def render_to_json_response(self, context, **response_kwargs):
        return JsonResponse(self.get_data(context), **response_kwargs)

    def get_data(self, context):
        return context


class AjaxableResponseMixin:
    """ผสมกับ CreateView/UpdateView เพื่อตอบกลับ JSON เมื่อถูกเรียกผ่าน AJAX"""

    def form_invalid(self, form):
        response = super().form_invalid(form)
        if self.request.headers.get("x-requested-with") == "XMLHttpRequest":
            return JsonResponse(form.errors, status=400)
        return response

    def form_valid(self, form):
        response = super().form_valid(form)
        if self.request.headers.get("x-requested-with") == "XMLHttpRequest":
            return JsonResponse({"pk": self.object.pk, "url": self.object.get_absolute_url()})
        return response
```

### 240.2 นำไปใช้กับ Post CRUD แบบเต็มรูปแบบ

```python
# blog/views_base.py
from django.views.generic import DetailView, ListView

from .mixins import SidebarContextMixin


class BaseBlogListView(SidebarContextMixin, ListView):
    paginate_by = 10


class BaseBlogDetailView(SidebarContextMixin, DetailView):
    pass
```

```python
# blog/views.py
from django.contrib.auth.mixins import LoginRequiredMixin
from django.http import JsonResponse
from django.shortcuts import get_object_or_404
from django.urls import reverse_lazy
from django.views import View
from django.views.generic import CreateView, DeleteView, UpdateView

from .forms import PostForm
from .mixins import AjaxableResponseMixin, AuthorRequiredMixin, JsonResponseMixin, LoggingMixin
from .models import Post
from .views_base import BaseBlogDetailView, BaseBlogListView


class PostListView(BaseBlogListView):
    model = Post
    template_name = "blog/post_list.html"
    context_object_name = "posts"

    def get_queryset(self):
        return Post.objects.filter(status=Post.Status.PUBLISHED).select_related(
            "author", "category"
        )


class PostDetailView(BaseBlogDetailView):
    model = Post
    template_name = "blog/post_detail.html"
    context_object_name = "post"


class PostCreateView(LoggingMixin, LoginRequiredMixin, AjaxableResponseMixin, CreateView):
    model = Post
    form_class = PostForm
    template_name = "blog/post_form.html"

    def form_valid(self, form):
        form.instance.author = self.request.user
        return super().form_valid(form)


class PostUpdateView(
    LoggingMixin, LoginRequiredMixin, AuthorRequiredMixin, AjaxableResponseMixin, UpdateView
):
    model = Post
    form_class = PostForm
    template_name = "blog/post_form.html"


class PostDeleteView(LoggingMixin, LoginRequiredMixin, AuthorRequiredMixin, DeleteView):
    model = Post
    template_name = "blog/post_confirm_delete.html"
    success_url = reverse_lazy("blog:post-list")


class PostDetailJsonView(JsonResponseMixin, View):
    def get(self, request, slug, *args, **kwargs):
        post = get_object_or_404(Post, slug=slug, status=Post.Status.PUBLISHED)
        data = {
            "id": post.id,
            "title": post.title,
            "content": post.content,
            "author": post.author.get_full_name() or post.author.username,
        }
        return self.render_to_json_response(data)
```

สังเกตความสวยงามของภาพรวมนี้: `PostUpdateView` มี Mixin ต่อกันถึง 4 ตัว
(`LoggingMixin`, `LoginRequiredMixin`, `AuthorRequiredMixin`, `AjaxableResponseMixin`)
ก่อนถึง `UpdateView` แต่ทุกตัวเรียงตามกฎ MRO ที่เรียนใน ขั้นตอนที่ 233 อย่างถูกต้อง
ทั้ง flow การ log, การเช็ค login, การเช็คความเป็นเจ้าของ และการตอบกลับ JSON ทำงาน
ร่วมกันได้อย่างสอดคล้อง โดยที่ **ไม่มีโค้ดจุดไหนซ้ำกับ `PostDeleteView` หรือ
`PostCreateView` เลย** — นี่คือเป้าหมายที่แท้จริงของการเรียนรู้ Mixin และ MRO ให้แม่นยำ

### 240.3 สรุปสิ่งที่ได้เรียนรู้ใน Part นี้

- ✅ ทบทวน Mixin และ MRO เชิงลึกกว่า Part 002 พร้อมเห็นว่า Generic CBV ทั้งหมดของ
  Django คือต้นไม้ Mixin ขนาดใหญ่
- ✅ เขียน Custom Mixin เอง (`AuthorRequiredMixin`) ที่ตรวจสอบความเป็นเจ้าของ object
- ✅ เข้าใจอย่างลึกซึ้งว่าทำไมลำดับการวาง Mixin ใน class definition ถึงสำคัญ และเห็น
  ตัวอย่างที่ผิดลำดับแล้วเกิดช่องโหว่ความปลอดภัยจริง
- ✅ รู้จัก `UserPassesTestMixin` และ `PermissionRequiredMixin` ของ Django และเลือกใช้
  ให้เหมาะกับสถานการณ์ได้
- ✅ Override `dispatch()` เพื่อทำ logic ก่อนทุก HTTP method เช่น Logging
- ✅ สร้าง Base ListView/DetailView ที่ inject context ร่วมกัน (sidebar)
- ✅ สร้าง `JsonResponseMixin` และ `AjaxableResponseMixin` สำหรับ view ที่ตอบ JSON
- ✅ เขียน Test เบื้องต้นสำหรับ CBV ด้วยทั้ง `self.client` และ `RequestFactory`
- ✅ มีกรอบการตัดสินใจ FBV vs CBV แบบมืออาชีพ พร้อมตัวอย่างสถานการณ์จริง

### 240.4 Checklist ก่อนไป Part ถัดไป

- [ ] อธิบายได้ว่า MRO คืออะไร และ `super()` เรียก "ตัวถัดไปใน MRO chain" ไม่ใช่
      parent class ตรง ๆ
- [ ] เขียน `AuthorRequiredMixin` เองได้โดยไม่ต้องเปิดดูโค้ดใน Part นี้
- [ ] อธิบายได้ว่าทำไมการวาง Generic View (เช่น `UpdateView`) ไว้ซ้ายสุดถึงทำให้ Mixin
      ทั้งหมดไม่ทำงาน
- [ ] แยกความแตกต่างระหว่าง `UserPassesTestMixin`, `PermissionRequiredMixin`, และ
      Custom Mixin ได้ว่าแต่ละแบบเหมาะกับสถานการณ์ไหน
- [ ] เขียน Mixin ที่ override `dispatch()` และไม่ลืมเรียก `super().dispatch()` ได้
- [ ] สร้าง Base View ของตัวเองที่ inject context ร่วมกันได้
- [ ] เขียน Test สำหรับ permission ของ CBV ได้ทั้งแบบ `self.client` และ `RequestFactory`
- [ ] อธิบายกรอบการตัดสินใจ FBV vs CBV ได้อย่างมีเหตุผล ไม่ใช่แค่ความชอบส่วนตัว

### 240.5 แบบฝึกหัดท้ายบท

**แบบฝึกหัดที่ 1**: เขียน Mixin ชื่อ `RateLimitMixin` ที่จำกัดให้ user หนึ่งคนสร้างโพสต์
ได้ไม่เกิน 5 โพสต์ต่อชั่วโมง (ใช้ `django.core.cache` เก็บ counter ชั่วคราวก็ได้ หรือ
query นับจำนวน `Post` ที่ `created_at` อยู่ในชั่วโมงที่ผ่านมาก็ได้) แล้วนำไปผสมกับ
`PostCreateView` โดยต้องวางลำดับให้ถูกต้องตามกฎ MRO ที่เรียนใน ขั้นตอนที่ 233

**แบบฝึกหัดที่ 2**: สร้าง `Comment` model ใหม่ที่มี field `author` และ `post` (FK)
แล้วเขียน `CommentUpdateView`/`CommentDeleteView` ที่ใช้ `AuthorRequiredMixin` ตัว
เดียวกับที่เขียนไว้สำหรับ `Post` (พิสูจน์ว่า Mixin ที่ออกแบบดีสามารถ reuse ข้าม Model
ได้จริง โดยไม่ต้องแก้โค้ดใน `AuthorRequiredMixin` เลยแม้แต่บรรทัดเดียว)

**แบบฝึกหัดที่ 3**: จงจงใจเขียน `class BrokenView(SomeGenericView, YourCustomMixin)`
(สลับลำดับผิด) แล้วรัน `print(BrokenView.__mro__)` เปรียบเทียบกับเวอร์ชันที่ถูกต้อง
บันทึกความแตกต่างของ MRO ทั้งสองแบบ และอธิบายด้วยคำพูดตัวเองว่าทำไมเวอร์ชันที่ผิด
ถึงทำให้ Mixin ไม่ทำงาน

**แบบฝึกหัดที่ 4 (ขั้นสูง)**: เขียน Test แบบ `RequestFactory` เพื่อพิสูจน์ว่า
`LoggingMixin` log ทั้ง request ที่ผ่าน permission check และ request ที่ถูกบล็อกโดย
`AuthorRequiredMixin` (ใช้ `self.assertLogs()` ของ Python's `unittest` เพื่อตรวจสอบ
ข้อความ log ที่เกิดขึ้นจริงระหว่างการทดสอบ)

### 240.6 คำถามที่พบบ่อย (FAQ)

**Q: ถ้า Mixin สองตัวมีเมธอดชื่อเดียวกันแต่ไม่ได้ตั้งใจให้ทำงานร่วมกัน (ไม่ได้เรียก
`super()`) จะเกิดอะไรขึ้น?**
A: เมธอดของ class ที่อยู่ซ้ายกว่าใน MRO จะถูกเรียก และเมธอดของอีกตัวจะถูก "บัง" ไปเลย
โดยไม่มี error ใด ๆ เตือน นี่คือเหตุผลที่ Mixin ที่ดีควรตั้งชื่อเมธอดให้เฉพาะเจาะจง
(เช่น `get_sidebar_categories()` แทน `get_data()` ที่กว้างเกินไป) เพื่อลดโอกาสชนกัน

**Q: ควรเขียน Custom Mixin เอง หรือใช้ package สำเร็จรูปอย่าง `django-braces`?**
A: `django-braces` เป็น package ยอดนิยมในอดีตที่รวม Mixin สำเร็จรูปไว้เยอะมาก แต่
Django เวอร์ชันปัจจุบัน (5.x) มี Mixin ที่จำเป็นส่วนใหญ่ในตัวอยู่แล้ว
(`LoginRequiredMixin`, `PermissionRequiredMixin`, `UserPassesTestMixin`) ส่วน Mixin
เฉพาะทางของธุรกิจ (เช่น `AuthorRequiredMixin`) ควรเขียนเองเสมอ เพราะ logic เหล่านี้
ผูกกับ domain ของแอปคุณโดยตรง ไม่ควรพึ่งพา package ภายนอกสำหรับ business logic หลัก

**Q: ทำไมไม่ใช้ decorator อย่าง `@login_required` กับ CBV แทน Mixin ไปเลย?**
A: `@login_required` decorator ออกแบบมาสำหรับ FBV เป็นหลัก ถ้าจะใช้กับ CBV ต้องห่อ
ผ่าน `method_decorator` และ apply กับเมธอดใน class โดยตรง (เช่น
`@method_decorator(login_required, name="dispatch")`) ซึ่งซับซ้อนกว่าการใช้
`LoginRequiredMixin` ตรง ๆ และไม่ได้ประโยชน์จากกลไก MRO/`super()` ที่ทำให้ compose
กับ Mixin อื่นได้อย่างสวยงามเท่า

**Q: ถ้า Mixin เดียวต้องใช้กับทั้ง `ListView` และ `View` ธรรมดา (ที่ไม่มี
`get_context_data()`) จะทำอย่างไร?**
A: ให้ Mixin นั้นตรวจสอบก่อนว่า method ที่ต้องการมีอยู่หรือไม่ (เช่น
`hasattr(super(), "get_context_data")`) หรือแยก Mixin ออกเป็นสองตัวที่เฉพาะเจาะจง
กว่าเดิม เช่น `TemplateSidebarMixin` (สำหรับ view ที่ render template) และ
`JsonResponseMixin` (สำหรับ view ที่ตอบ JSON) แยกกันชัดเจน — การแยก Mixin ให้แคบและ
เฉพาะเจาะจงมักดีกว่าการพยายามทำ Mixin เดียวที่ "ฉลาด" เกินไป

---

## เตรียมตัวสำหรับ Part ถัดไป

**Part 025: Django Forms เบื้องต้น** (ขั้นตอนที่ 241-250) จะพาคุณไปเจาะลึกระบบฟอร์ม
ของ Django อย่างละเอียด ตั้งแต่ `forms.Form` พื้นฐาน, field types ต่าง ๆ, validation,
widgets, ไปจนถึงการ render ฟอร์มใน template ด้วยมือ — ซึ่งจะทำให้คุณเข้าใจกลไกเบื้องหลัง
`PostForm` ที่เราใช้ตลอด Part นี้อย่างถ่องแท้ ก่อนจะไปเจาะลึก `ModelForm` และ Formsets
ใน Part 026 และ Custom Widgets ใน Part 027 ตามลำดับ

เตรียมเปิดไฟล์ `blog/views.py` และ `blog/mixins.py` ที่เพิ่งสร้างไว้ในเครื่อง เพราะ
Part 025 จะกลับมาแก้ไข `PostForm` ที่เราใช้ `form_class = PostForm` ไปแล้วในหลาย view
ของ Part นี้ให้ลึกซึ้งยิ่งขึ้น!
