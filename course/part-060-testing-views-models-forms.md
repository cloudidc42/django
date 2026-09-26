# Part 060: Testing Views, Models และ Forms

> **ขั้นตอนที่ 591-600 ของหลักสูตร** | Phase 7: Testing & Quality Assurance
>
> เป้าหมายของ Part นี้: ต่อยอดจาก Part 059 ที่คุณเริ่มรู้จัก `unittest`/`TestCase`
> เบื้องต้น มาเจาะลึก **การเขียนเทสต์ให้ครอบคลุมสามส่วนหลักของแอป Django ทุกแอป**
> คือ Model, View และ Form อย่างเป็นระบบ คุณจะได้ทดสอบ `full_clean()` และ custom
> method ของ Model ที่สร้างไว้ตั้งแต่ Part 026, ทดสอบ `PostListView`/`PostDetailView`
> จาก Part 022 ทั้งแบบผ่าน `self.client` และแบบ unit-level ด้วย `RequestFactory`,
> ทดสอบ view ที่ต้อง login ด้วยทั้ง `client.login()` และ `client.force_login()`,
> ทดสอบ `CommentForm`/`PostForm` ที่เป็น `ModelForm`, ทดสอบการ render template และ
> redirect อย่างละเอียด, ทดสอบการปรับแต่ง Django Admin ของแอป blog, และปิดท้ายด้วย
> การทดสอบว่า signal auto-create `Profile` จาก Part 019 ยังทำงานถูกต้องจริง เมื่อจบ
> Part นี้ แอป `blog` และ `accounts` ของคุณจะมี test suite ที่ครอบคลุมเกือบทุกจุดที่
> เสี่ยงพังเมื่อมีคนมาแก้โค้ดในอนาคต

---

## สารบัญของ Part นี้

- ขั้นตอนที่ 591: Testing Model เจาะลึกกว่า Part 059 — `full_clean()`, custom methods, `Meta.ordering`
- ขั้นตอนที่ 592: Testing Views — `self.client.get()/post()`, ตรวจสอบ status code, `response.context`
- ขั้นตอนที่ 593: Testing View ที่ต้อง login — `self.client.login()` vs `self.client.force_login()`
- ขั้นตอนที่ 594: Testing Forms — `is_valid()`, `form.errors`, testing custom `clean()`/`clean_<field>()`
- ขั้นตอนที่ 595: Testing Class-Based Views ด้วย `RequestFactory` (unit-level เทียบกับผ่าน client)
- ขั้นตอนที่ 596: Testing การ render Template — `assertTemplateUsed`, `assertContains`, `assertNotContains`
- ขั้นตอนที่ 597: Testing Redirect — `assertRedirects` เจาะลึก (status code, target URL, `follow=True`)
- ขั้นตอนที่ 598: Testing การปรับแต่ง Django Admin (custom action, `list_display`)
- ขั้นตอนที่ 599: Testing Signals (เชื่อมกับ Part 019 — ทดสอบว่า `Profile` ถูกสร้างอัตโนมัติจริง)
- ขั้นตอนที่ 600: สรุปและแบบฝึกหัด — เขียน test suite ที่สมบูรณ์ครอบคลุม Model/View/Form ของแอป blog

---

## ขั้นตอนที่ 591: Testing Model เจาะลึกกว่า Part 059 — `full_clean()`, custom methods, `Meta.ordering`

### 591.1 ทบทวนโครงสร้างโปรเจกต์ทั้งหมดก่อนเข้าสู่ Part นี้

Part นี้เขียนเทสต์ให้กับโค้ดที่คุณสร้างมาแล้วตลอดหลักสูตร โดยเฉพาะแอป `blog`
(Model จาก Part 026, View จาก Part 022) และแอป `accounts` (Profile + Signal
จาก Part 019) เพื่อไม่ให้ต้องย้อนกลับไปเปิดไฟล์เก่า นี่คือสถานะปัจจุบันของทั้ง
สองแอปแบบเต็ม รวมถึง**custom method 3 ตัวที่เพิ่มเข้ามาใหม่**สำหรับ Part นี้
โดยเฉพาะ (`excerpt()`, `reading_time_minutes()`, `get_absolute_url()`) ซึ่งเป็น
รูปแบบ method ที่โปรเจกต์ Django จริงแทบทุกโปรเจกต์มี:

```python
# accounts/models.py
from django.conf import settings
from django.db import models


class Profile(models.Model):
    user = models.OneToOneField(
        settings.AUTH_USER_MODEL, on_delete=models.CASCADE, related_name="profile",
    )
    bio = models.TextField(blank=True)
    avatar = models.ImageField(upload_to="avatars/", blank=True, null=True)

    def __str__(self):
        return f"โปรไฟล์ของ {self.user.username}"
```

```python
# accounts/signals.py (ทบทวนจาก Part 019 ขั้นตอนที่ 184)
from django.conf import settings
from django.db.models.signals import post_save
from django.dispatch import receiver

from .models import Profile


@receiver(post_save, sender=settings.AUTH_USER_MODEL, dispatch_uid="accounts_create_profile")
def create_profile(sender, instance, created, **kwargs):
    if created:
        Profile.objects.create(user=instance)


@receiver(post_save, sender=settings.AUTH_USER_MODEL, dispatch_uid="accounts_save_profile")
def save_profile(sender, instance, **kwargs):
    if hasattr(instance, "profile"):
        instance.profile.save()
```

```python
# blog/models.py (ทบทวนจาก Part 026 ขั้นตอนที่ 251.2 + เพิ่ม custom method ใหม่)
from django.conf import settings
from django.core.exceptions import ValidationError
from django.core.validators import MinLengthValidator
from django.db import models
from django.urls import reverse
from django.utils import timezone


def validate_no_banned_words(value):
    banned_words = ["สแปม", "โฆษณา", "คลิกที่นี่"]
    lowered = value.lower()
    for word in banned_words:
        if word in lowered:
            raise ValidationError(f'ห้ามมีคำว่า "{word}" ปรากฏอยู่ในเนื้อหา')


class Category(models.Model):
    name = models.CharField(max_length=100)
    slug = models.SlugField(max_length=120, unique=True)
    created_by = models.ForeignKey(
        settings.AUTH_USER_MODEL,
        on_delete=models.CASCADE,
        null=True,
        blank=True,
        related_name="categories",
    )

    class Meta:
        ordering = ["name"]
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

    class Meta:
        ordering = ["-created_at"]

    def __str__(self):
        return self.title

    def get_absolute_url(self):
        """ทางลัดมาตรฐานที่ Django แนะนำ: รวม logic การสร้าง URL ของ object ไว้ที่เดียว"""
        return reverse("blog:detail", kwargs={"slug": self.slug})

    def excerpt(self, length=80):
        """ตัดเนื้อหาให้สั้นลงสำหรับแสดงในหน้ารายการ (ไม่ตัดกลางคำถ้าเป็นไปได้)"""
        text = self.content.strip()
        if len(text) <= length:
            return text
        return text[:length].rsplit(" ", 1)[0] + "…"

    def reading_time_minutes(self):
        """ประมาณเวลาที่ใช้อ่าน โดยสมมติความเร็วอ่านเฉลี่ย 200 คำต่อนาที (ขั้นต่ำ 1 นาที)"""
        word_count = len(self.content.split())
        minutes = max(1, round(word_count / 200))
        return minutes

    def is_recent(self, days=7):
        """True ถ้าโพสต์นี้ถูกสร้างภายในจำนวนวันที่กำหนด (ใช้แสดง badge 'ใหม่' บน UI)"""
        return (timezone.now() - self.created_at).days < days


class Comment(models.Model):
    post = models.ForeignKey(Post, on_delete=models.CASCADE, related_name="comments")
    author = models.CharField(max_length=100)
    text = models.TextField()
    created_at = models.DateTimeField(auto_now_add=True)

    class Meta:
        ordering = ["created_at"]

    def __str__(self):
        return f"ความคิดเห็นโดย {self.author} บน {self.post.title}"
```

```python
# blog/forms.py (ทบทวนจาก Part 026 ขั้นตอนที่ 251.3 และ 254.4)
from django import forms

from .models import Comment, Post


class CommentForm(forms.ModelForm):
    class Meta:
        model = Comment
        fields = ["author", "text"]


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

```python
# blog/views.py (ทบทวนจาก Part 022 ขั้นตอนที่ 214-215 + Part 026 ขั้นตอนที่ 252 สำหรับ post_create_view)
from django.contrib.auth.decorators import login_required
from django.shortcuts import get_object_or_404, redirect, render
from django.views.generic import DetailView, ListView

from .forms import CommentForm, PostForm
from .models import Category, Post


class PostListView(ListView):
    model = Post
    template_name = "blog/post_list.html"
    context_object_name = "posts"
    paginate_by = 10

    def get_queryset(self):
        return Post.objects.filter(is_published=True).select_related("category")

    def get_context_data(self, **kwargs):
        context = super().get_context_data(**kwargs)
        context["categories"] = Category.objects.all()
        return context


class PostDetailView(DetailView):
    model = Post
    template_name = "blog/post_detail.html"
    context_object_name = "post"

    def get_queryset(self):
        return Post.objects.filter(is_published=True).select_related("category")

    def get_context_data(self, **kwargs):
        context = super().get_context_data(**kwargs)
        post = self.object
        context["comments"] = post.comments.all()
        context["comment_form"] = CommentForm()
        return context

    def post(self, request, *args, **kwargs):
        self.object = self.get_object()
        form = CommentForm(request.POST)
        if form.is_valid():
            comment = form.save(commit=False)
            comment.post = self.object
            comment.save()
            return redirect("blog:detail", slug=self.object.slug)
        context = self.get_context_data(comment_form=form)
        return self.render_to_response(context)


@login_required
def post_create_view(request):
    if request.method == "POST":
        form = PostForm(request.POST)
        if form.is_valid():
            post = form.save(commit=False)
            post.author = request.user
            post.save()
            form.save_m2m()
            return redirect("blog:detail", slug=post.slug)
    else:
        form = PostForm()
    return render(request, "blog/post_form.html", {"form": form})
```

```python
# blog/urls.py
from django.urls import path

from . import views

app_name = "blog"

urlpatterns = [
    path("", views.PostListView.as_view(), name="list"),
    path("new/", views.post_create_view, name="create"),
    path("<slug:slug>/", views.PostDetailView.as_view(), name="detail"),
]
```

โครงสร้างนี้คือ **จุดตั้งต้นของ Part นี้ทั้งหมด** — ทุกเทสต์ที่จะเขียนต่อไปนี้
ทดสอบโค้ดชุดนี้โดยตรง

### 591.2 ทบทวนสั้น ๆ จาก Part 059: `TestCase`, `setUp()`, ฐานข้อมูลทดสอบ

Part 059 สอนพื้นฐานว่า Django สร้าง**ฐานข้อมูลทดสอบแยกต่างหาก** (ปกติชื่อ
`test_<ชื่อ db เดิม>`) ทุกครั้งที่รัน `python manage.py test`, ลบทิ้งอัตโนมัติหลัง
รันเสร็จ, และแต่ละ **test method จะถูกครอบด้วย transaction ของตัวเอง** แล้ว
`rollback` เมื่อจบ method ทำให้เทสต์แต่ละตัว**ไม่เห็นข้อมูลที่ method อื่นสร้างไว้
เลย** — Part นี้จะใช้พื้นฐานนี้ตลอดทั้งบท โดยเริ่มจากไฟล์ทดสอบ Model:

```python
# blog/tests/test_models.py
from django.contrib.auth import get_user_model
from django.core.exceptions import ValidationError
from django.test import TestCase

from blog.models import Category, Comment, Post

User = get_user_model()


class PostModelTests(TestCase):
    def setUp(self):
        self.user = User.objects.create_user(username="malee", password="pass123456")
        self.category = Category.objects.create(name="เทคโนโลยี", slug="technology")
        self.post = Post.objects.create(
            title="แนะนำ Django สำหรับมือใหม่",
            slug="intro-django",
            content="Django เป็น web framework ของ Python ที่ทรงพลังมาก " * 5,
            author=self.user,
            category=self.category,
            is_published=True,
        )
```

> **หมายเหตุเรื่องโครงสร้างไฟล์**: ตั้งแต่ Part นี้เป็นต้นไป เราแยกไฟล์เทสต์เป็น
> `blog/tests/test_models.py`, `test_views.py`, `test_forms.py` แทนไฟล์
> `blog/tests.py` ไฟล์เดียว (ต้องมี `blog/tests/__init__.py` ด้วยเพื่อให้เป็น
> package) — Django ค้นหาเทสต์แบบ auto-discovery ทั้งสองรูปแบบได้เหมือนกัน
> การแยกไฟล์เป็นธรรมเนียมที่ทีมมืออาชีพใช้เมื่อแอปมีเทสต์จำนวนมาก

### 591.3 ทบทวน `full_clean()`: ทำไมต้องเรียกเองเสมอเมื่อทดสอบ Model ตรง ๆ

จาก Part 026 ขั้นตอนที่ 254.1 คุณรู้แล้วว่า `Model.objects.create()` และ
`instance.save()` **ไม่เรียก validator ให้อัตโนมัติ** ต้องเรียก
`instance.full_clean()` เองเสมอ กฎนี้สำคัญที่สุดตอนเขียนเทสต์ Model เพราะถ้าลืม
เรียก `full_clean()` เทสต์ของคุณจะ**ไม่มีทางจับ validator ที่พังได้เลย** แม้ว่า
Model จะประกาศ `validators=[...]` ไว้ถูกต้องแล้วก็ตาม

```python
class PostValidationTests(TestCase):
    def setUp(self):
        self.user = User.objects.create_user(username="malee", password="pass123456")

    def build_post(self, **overrides):
        """helper method สร้าง Post instance (ยังไม่ save) พร้อมค่า default ที่ผ่าน validation ได้"""
        defaults = {
            "title": "หัวข้อที่ยาวพอสมควร",
            "slug": "valid-slug",
            "content": "เนื้อหาปกติทั่วไปที่ไม่มีคำต้องห้ามอะไรเลย",
            "author": self.user,
        }
        defaults.update(overrides)
        return Post(**defaults)

    def test_title_shorter_than_5_chars_raises_validation_error(self):
        post = self.build_post(title="สั้น")
        with self.assertRaises(ValidationError) as ctx:
            post.full_clean()
        self.assertIn("title", ctx.exception.message_dict)

    def test_title_with_valid_length_passes_full_clean(self):
        post = self.build_post(title="หัวข้อที่ยาวพอ")
        post.full_clean()  # ไม่ raise = ผ่าน

    def test_content_with_banned_word_raises_validation_error(self):
        post = self.build_post(content="คลิกที่นี่เพื่อรับรางวัลฟรี")
        with self.assertRaises(ValidationError) as ctx:
            post.full_clean()
        self.assertIn("content", ctx.exception.message_dict)

    def test_duplicate_slug_raises_validation_error_via_validate_unique(self):
        Post.objects.create(**{
            "title": "โพสต์แรกในระบบ", "slug": "duplicate-slug",
            "content": "เนื้อหาปกติ", "author": self.user,
        })
        second = self.build_post(slug="duplicate-slug")
        with self.assertRaises(ValidationError) as ctx:
            second.full_clean()
        self.assertIn("slug", ctx.exception.message_dict)
```

### 591.4 จุดที่มือใหม่พลาดบ่อย: `full_clean()` ตรวจ `unique` กับ**ตัวเอง**ตอนแก้ไข

`validate_unique()` (ส่วนหนึ่งของ `full_clean()`) จะ**ไม่**ฟ้อง error ถ้า pk ของ
instance ที่กำลังตรวจตรงกับแถวที่เจอในฐานข้อมูล (คือ "ชนกับตัวเอง" ตอนแก้ไข
object เดิม) — เขียนเทสต์ยืนยันพฤติกรรมนี้ไว้เพื่อป้องกัน regression:

```python
    def test_editing_existing_post_keeps_its_own_slug_without_error(self):
        post = Post.objects.create(
            title="โพสต์ที่จะแก้ไข", slug="editable-post",
            content="เนื้อหาปกติ", author=self.user,
        )
        post.title = "โพสต์ที่แก้ไขหัวข้อแล้ว"
        post.full_clean()  # ต้องไม่ raise แม้ slug จะเหมือนเดิมทุกประการ
```

### 591.5 ทดสอบ Custom Method ของ Model

Custom method อย่าง `excerpt()`, `reading_time_minutes()`, `is_recent()` และ
`get_absolute_url()` คือ**ตรรกะทางธุรกิจ (business logic)** ที่ทีมมืออาชีพให้
ความสำคัญกับการเทสต์มากที่สุด เพราะมันคือโค้ดที่ทีมเขียนเองล้วน ๆ (ต่างจาก field
validation ที่ Django framework ช่วยทำงานส่วนใหญ่ให้) — ถ้า method พวกนี้พังโดย
ไม่มีใครรู้ ผลกระทบจะไปโผล่ที่หน้าเว็บจริงตรง ๆ

```python
class PostCustomMethodTests(TestCase):
    def setUp(self):
        self.user = User.objects.create_user(username="malee", password="pass123456")

    def test_excerpt_returns_full_content_when_shorter_than_limit(self):
        post = Post.objects.create(
            title="หัวข้อทดสอบ", slug="short-content",
            content="เนื้อหาสั้น ๆ", author=self.user,
        )
        self.assertEqual(post.excerpt(length=80), "เนื้อหาสั้น ๆ")

    def test_excerpt_truncates_long_content_and_ends_with_ellipsis(self):
        long_content = "คำ " * 100  # ยาวเกิน 80 ตัวอักษรแน่นอน
        post = Post.objects.create(
            title="หัวข้อทดสอบ", slug="long-content",
            content=long_content, author=self.user,
        )
        excerpt = post.excerpt(length=80)
        self.assertTrue(excerpt.endswith("…"))
        self.assertLessEqual(len(excerpt), 81)  # 80 ตัวอักษร + เครื่องหมาย …

    def test_excerpt_does_not_cut_a_word_in_half(self):
        post = Post.objects.create(
            title="หัวข้อทดสอบ", slug="word-boundary",
            content="Django เป็นเฟรมเวิร์กที่ยอดเยี่ยมมากสำหรับงาน backend ทุกประเภท",
            author=self.user,
        )
        excerpt = post.excerpt(length=20)
        # ต้องไม่มี fragment ของคำที่ถูกตัดกลาง เช่น "เฟรม" ค้างอยู่ตอนท้ายก่อน …
        self.assertFalse(excerpt.rstrip("…").endswith(" "))

    def test_reading_time_minimum_is_one_minute(self):
        post = Post.objects.create(
            title="หัวข้อทดสอบ", slug="tiny-post",
            content="สั้นมาก", author=self.user,
        )
        self.assertEqual(post.reading_time_minutes(), 1)

    def test_reading_time_scales_with_word_count(self):
        post = Post.objects.create(
            title="หัวข้อทดสอบ", slug="long-post",
            content="คำ " * 600,  # ประมาณ 600 คำ ที่ 200 คำ/นาที = 3 นาที
            author=self.user,
        )
        self.assertEqual(post.reading_time_minutes(), 3)

    def test_get_absolute_url_uses_slug(self):
        post = Post.objects.create(
            title="หัวข้อทดสอบ", slug="url-test",
            content="เนื้อหาปกติ", author=self.user,
        )
        self.assertEqual(post.get_absolute_url(), "/url-test/")

    def test_is_recent_true_for_newly_created_post(self):
        post = Post.objects.create(
            title="หัวข้อทดสอบ", slug="brand-new",
            content="เนื้อหาปกติ", author=self.user,
        )
        self.assertTrue(post.is_recent())

    def test_is_recent_false_for_old_post(self):
        from datetime import timedelta
        from django.utils import timezone

        post = Post.objects.create(
            title="หัวข้อทดสอบ", slug="old-post",
            content="เนื้อหาปกติ", author=self.user,
        )
        # created_at มี auto_now_add=True แก้ตรง ๆ ไม่ได้ผ่าน save() ปกติ
        # ต้อง update() ผ่าน queryset เพื่อ bypass auto_now_add
        Post.objects.filter(pk=post.pk).update(
            created_at=timezone.now() - timedelta(days=30)
        )
        post.refresh_from_db()
        self.assertFalse(post.is_recent())
```

> **เทคนิคสำคัญที่ใช้ในเทสต์สุดท้าย**: field ที่มี `auto_now_add=True` จะถูก
> Django **บังคับเขียนทับด้วยเวลาปัจจุบันเสมอทุกครั้งที่เรียก `instance.save()`**
> ไม่ว่าจะตั้งค่าอะไรไว้ก่อนหน้าก็ตาม วิธีเดียวที่จะ "ปลอมวันที่ในอดีต" สำหรับ
> เทสต์ได้คือใช้ `QuerySet.update()` ซึ่ง**ไม่ผ่าน `save()` ของ Model เลย** จึงไม่
> ถูก `auto_now_add` เขียนทับ ตามด้วย `instance.refresh_from_db()` เพื่อดึงค่า
> ใหม่จากฐานข้อมูลกลับเข้า object ในหน่วยความจำ

### 591.6 ทดสอบ `Meta.ordering`

`Meta.ordering` เป็นจุดที่เทสต์แบบ "ผิดโดยไม่รู้ตัว" ได้ง่ายที่สุด เพราะถ้าเขียน
เทสต์เทียบ queryset กับ list โดยไม่ระวังเรื่องลำดับ เทสต์อาจ**ผ่านได้แม้ ordering
จะพังจริง** (เช่นบางฐานข้อมูลคืนผลลัพธ์ตามลำดับที่แทรกโดยบังเอิญ) วิธีที่ถูกต้อง
คือใช้ `assertQuerySetEqual` พร้อม `ordered=True` (ค่า default) เทียบกับ list ที่
เรียงลำดับตามที่คาดหวังไว้อย่างชัดเจน:

```python
class PostOrderingTests(TestCase):
    def setUp(self):
        self.user = User.objects.create_user(username="malee", password="pass123456")

    def test_posts_are_ordered_newest_first_by_default(self):
        from datetime import timedelta
        from django.utils import timezone

        oldest = Post.objects.create(
            title="โพสต์แรกสุด", slug="oldest", content="เนื้อหา", author=self.user,
        )
        newest = Post.objects.create(
            title="โพสต์ล่าสุด", slug="newest", content="เนื้อหา", author=self.user,
        )
        middle = Post.objects.create(
            title="โพสต์กลาง", slug="middle", content="เนื้อหา", author=self.user,
        )
        # จำลองเวลาสร้างที่ต่างกันชัดเจน (created_at ปกติเท่ากันเกือบหมดถ้าสร้างเร็วมาก)
        now = timezone.now()
        Post.objects.filter(pk=oldest.pk).update(created_at=now - timedelta(days=2))
        Post.objects.filter(pk=middle.pk).update(created_at=now - timedelta(days=1))
        Post.objects.filter(pk=newest.pk).update(created_at=now)

        self.assertQuerySetEqual(
            Post.objects.all(),
            [newest, middle, oldest],   # ต้องเรียงจากใหม่สุดไปเก่าสุด (-created_at)
        )

    def test_comments_are_ordered_oldest_first(self):
        post = Post.objects.create(
            title="โพสต์สำหรับทดสอบคอมเมนต์", slug="comment-order-test",
            content="เนื้อหา", author=self.user,
        )
        c1 = Comment.objects.create(post=post, author="สมชาย", text="ความเห็นแรก")
        c2 = Comment.objects.create(post=post, author="สมหญิง", text="ความเห็นที่สอง")

        self.assertQuerySetEqual(post.comments.all(), [c1, c2])
```

> **หมายเหตุ**: `assertQuerySetEqual` (Django 4.1+ แทนที่ `assertQuerysetEqual`
> รุ่นเก่าที่สะกดตัว `s` เล็ก ซึ่งยัง alias ใช้ได้อยู่แต่จะถูกลบในอนาคต) เทียบ
> object ทีละตัวตามลำดับโดยตรงถ้าไม่ระบุ `transform` — ค่า default นี้เพียงพอ
> สำหรับกรณีส่วนใหญ่ เพราะ Django Model instance เทียบ equality กันด้วย `pk`
> และ class เป็นหลักอยู่แล้ว

### 591.7 ตารางสรุป Assertion ที่ใช้บ่อยเมื่อทดสอบ Model

| Assertion | ใช้ตรวจอะไร |
|---|---|
| `assertRaises(ValidationError)` | validator (Model หรือ custom) ทำงานและ raise error ตามที่คาด |
| `exception.message_dict` | error message ผูกกับ field ไหน (ใช้คู่กับ `assertRaises` ผ่าน `as ctx`) |
| `assertEqual(obj.method(), expected)` | ผลลัพธ์ของ custom method ตรงตามที่คาดหวัง |
| `assertQuerySetEqual(qs, [...])` | ลำดับและเนื้อหาของ queryset ตรงกับ `Meta.ordering` ที่คาดไว้ |
| `assertTrue(os.path.exists(...))` | ผลข้างเคียงกับระบบไฟล์ (ทบทวนแนวคิดจาก Part 019 ขั้นตอนที่ 185.7) |
| `refresh_from_db()` | ดึงค่าล่าสุดจากฐานข้อมูลกลับเข้า instance หลัง update ผ่าน queryset หรือ signal |

---

## ขั้นตอนที่ 592: Testing Views — `self.client.get()/post()`, ตรวจสอบ status code, `response.context`

### 592.1 `self.client`: Test Client จำลองเบราว์เซอร์แบบเต็มรูปแบบ

`django.test.TestCase` เตรียม attribute `self.client` (instance ของ
`django.test.Client`) ให้อัตโนมัติทุกคลาส — มันคือ**เบราว์เซอร์จำลอง**ที่ยิง
request ผ่าน URL dispatcher ของ Django จริง (ทบทวนแผนภาพ MTV จาก Part 001) โดย
ไม่ต้องรัน development server จริงเลย ทำให้เทสต์เร็วและไม่ต้องพึ่งพอร์ตเครือข่าย

```python
# blog/tests/test_views.py
from django.contrib.auth import get_user_model
from django.test import TestCase
from django.urls import reverse

from blog.models import Category, Post

User = get_user_model()


class PostListViewTests(TestCase):
    def setUp(self):
        self.user = User.objects.create_user(username="malee", password="pass123456")
        self.category = Category.objects.create(name="เทคโนโลยี", slug="technology")
        self.published_post = Post.objects.create(
            title="บทความที่เผยแพร่แล้ว", slug="published-post",
            content="เนื้อหาปกติทั่วไป", author=self.user,
            category=self.category, is_published=True,
        )
        self.draft_post = Post.objects.create(
            title="บทความฉบับร่าง", slug="draft-post",
            content="เนื้อหาปกติทั่วไป", author=self.user,
            is_published=False,
        )
```

### 592.2 ทดสอบ GET Request และ Status Code

```python
    def test_list_view_returns_200(self):
        response = self.client.get(reverse("blog:list"))
        self.assertEqual(response.status_code, 200)

    def test_detail_view_of_published_post_returns_200(self):
        url = reverse("blog:detail", kwargs={"slug": self.published_post.slug})
        response = self.client.get(url)
        self.assertEqual(response.status_code, 200)

    def test_detail_view_of_draft_post_returns_404(self):
        """draft_post ถูกกรองออกด้วย get_queryset() ของ PostDetailView (Part 022 ขั้นตอนที่ 214.3)"""
        url = reverse("blog:detail", kwargs={"slug": self.draft_post.slug})
        response = self.client.get(url)
        self.assertEqual(response.status_code, 404)

    def test_nonexistent_slug_returns_404(self):
        response = self.client.get("/this-slug-does-not-exist/")
        self.assertEqual(response.status_code, 404)
```

### 592.3 ตรวจสอบ `response.context`

`response.context` คือ dictionary-like object ที่เก็บ**ทุกตัวแปร**ที่ view ส่ง
เข้า template ตอน render — เป็นวิธีตรวจสอบว่า view "เตรียมข้อมูลถูกต้อง" โดยไม่
ต้องไปแคะ HTML ที่ render ออกมา (ทดสอบ HTML ที่ render จริงเป็นเรื่องของขั้นตอน
ที่ 596):

```python
    def test_list_view_context_contains_only_published_posts(self):
        response = self.client.get(reverse("blog:list"))
        posts_in_context = list(response.context["posts"])
        self.assertIn(self.published_post, posts_in_context)
        self.assertNotIn(self.draft_post, posts_in_context)

    def test_list_view_context_contains_categories_for_sidebar(self):
        response = self.client.get(reverse("blog:list"))
        self.assertIn(self.category, response.context["categories"])

    def test_detail_view_context_contains_correct_post(self):
        url = reverse("blog:detail", kwargs={"slug": self.published_post.slug})
        response = self.client.get(url)
        self.assertEqual(response.context["post"], self.published_post)

    def test_detail_view_context_contains_empty_comment_form(self):
        url = reverse("blog:detail", kwargs={"slug": self.published_post.slug})
        response = self.client.get(url)
        from blog.forms import CommentForm
        self.assertIsInstance(response.context["comment_form"], CommentForm)
        self.assertFalse(response.context["comment_form"].is_bound)
```

### 592.4 ทดสอบ POST Request ผ่าน View

`self.client.post(url, data)` จำลองการ submit ฟอร์มจริง — ส่ง `data` เป็น
dictionary ที่ Django แปลงเป็น `POST` body ให้อัตโนมัติ (รวม CSRF token ให้ด้วย
เพราะ Test Client ปิดการตรวจสอบ CSRF ไว้เป็นค่าเริ่มต้นเพื่อความสะดวก — จะเจาะลึก
ข้อยกเว้นนี้ใน Part 062):

```python
class PostDetailViewCommentTests(TestCase):
    def setUp(self):
        self.user = User.objects.create_user(username="malee", password="pass123456")
        self.post = Post.objects.create(
            title="บทความสำหรับทดสอบคอมเมนต์", slug="comment-target",
            content="เนื้อหาปกติ", author=self.user, is_published=True,
        )
        self.url = reverse("blog:detail", kwargs={"slug": self.post.slug})

    def test_post_valid_comment_creates_comment_and_redirects(self):
        response = self.client.post(self.url, data={
            "author": "สมชาย",
            "text": "ความคิดเห็นที่ยาวพอสมควรครับ",
        })
        self.assertEqual(response.status_code, 302)
        self.assertEqual(self.post.comments.count(), 1)
        self.assertEqual(self.post.comments.first().author, "สมชาย")

    def test_post_invalid_comment_does_not_create_comment(self):
        response = self.client.post(self.url, data={
            "author": "",   # required field ที่ขาดไป
            "text": "ความคิดเห็น",
        })
        self.assertEqual(response.status_code, 200)  # render ฟอร์มพร้อม error กลับมา ไม่ redirect
        self.assertEqual(self.post.comments.count(), 0)

    def test_post_invalid_comment_returns_form_with_errors_in_context(self):
        response = self.client.post(self.url, data={"author": "", "text": ""})
        form = response.context["comment_form"]
        self.assertFalse(form.is_valid())
        self.assertIn("author", form.errors)
        self.assertIn("text", form.errors)
```

### 592.5 ตารางสรุป HTTP Status Code ที่ต้องทดสอบเสมอ

| Status Code | สถานการณ์ | เทสต์ที่ควรมี |
|---|---|---|
| `200 OK` | GET หน้าที่มีอยู่จริงและเข้าถึงได้ | ทดสอบทุก view หลักอย่างน้อย 1 เคส |
| `302 Found` | หลัง POST สำเร็จ (redirect ตาม Post/Redirect/Get pattern) หรือ login required เด้งไป login | ทดสอบคู่กับ `assertRedirects` (ขั้นตอนที่ 597) |
| `404 Not Found` | `slug`/`pk` ไม่มีอยู่จริง หรือถูก `get_queryset()` กรองออก | ทดสอบทั้งสองสาเหตุแยกกันเสมอ |
| `403 Forbidden` | ผู้ใช้ login แล้วแต่ไม่มีสิทธิ์ (เจาะลึก permission ใน Part 034) | ทดสอบเมื่อ view มีระบบสิทธิ์ |
| `200` พร้อม `form.errors` | POST ข้อมูลไม่ผ่าน validation | ทดสอบว่า**ไม่มี** object ใหม่ถูกสร้างขึ้นด้วย |

---

## ขั้นตอนที่ 593: Testing View ที่ต้อง Login — `self.client.login()` vs `self.client.force_login()`

### 593.1 ทบทวน `post_create_view` ที่มี `@login_required`

จากขั้นตอนที่ 591.1 `post_create_view` ถูกครอบด้วย `@login_required` (decorator
มาตรฐานที่จะเจาะลึกเต็มรูปแบบใน Part 031) พฤติกรรมที่ต้องทดสอบมี 2 กรณีชัดเจน:
**ไม่ login → redirect ไปหน้า login**, และ **login แล้ว → เข้าถึงและใช้งานได้ปกติ**

```python
# blog/tests/test_views.py (ต่อ)
class PostCreateViewAuthTests(TestCase):
    def setUp(self):
        self.user = User.objects.create_user(username="malee", password="pass123456")
        self.url = reverse("blog:create")

    def test_anonymous_user_is_redirected_to_login(self):
        response = self.client.get(self.url)
        self.assertEqual(response.status_code, 302)
        self.assertIn("/accounts/login/", response.url)
```

### 593.2 `client.login()`: จำลองการกรอกฟอร์ม Login จริง

`client.login(username=..., password=...)` **ต้องใช้รหัสผ่านตัวจริงที่ยังไม่ถูก
เข้ารหัส (plaintext)** เพราะมันเรียก authentication backend เบื้องหลังเหมือนที่
ผู้ใช้กรอกฟอร์ม login จริง (ตรวจสอบ password ผ่าน hashing algorithm เต็มรูปแบบ)
และ**คืนค่า `True`/`False`** บอกว่า login สำเร็จหรือไม่ — เป็นจุดที่มือใหม่มัก
ลืมเช็ค แล้วเทสต์ผ่านทั้งที่ login ไม่สำเร็จจริง (เพราะโค้ดหลังจากนั้นเห็นเป็น
anonymous user แต่ view redirect ไป login พอดี ทำให้ status code ดูถูกต้องโดย
บังเอิญในบางเคส):

```python
    def test_authenticated_user_can_access_create_page_with_client_login(self):
        logged_in = self.client.login(username="malee", password="pass123456")
        self.assertTrue(logged_in, "client.login() ล้มเหลว — ตรวจสอบ username/password ที่ใช้สร้าง user")

        response = self.client.get(self.url)
        self.assertEqual(response.status_code, 200)

    def test_wrong_password_login_fails_and_returns_false(self):
        logged_in = self.client.login(username="malee", password="ผิดแน่นอน")
        self.assertFalse(logged_in)
```

> **กฎเหล็ก**: ทุกครั้งที่ใช้ `client.login()` ในเทสต์ ให้ **`assertTrue()` ค่าที่
> คืนกลับมาเสมอ** (หรืออย่างน้อยคอมเมนต์อธิบายว่าทำไมไม่เช็ค) ไม่เช่นนั้นถ้า
> `login()` ล้มเหลวเงียบ ๆ (เช่น พิมพ์ username ผิด) เทสต์ที่ควรทดสอบ "user ที่
> login แล้ว" จะกลายเป็นทดสอบ "anonymous user" โดยไม่มีใครรู้ตัว

### 593.3 `client.force_login()`: ข้ามการตรวจสอบรหัสผ่านทั้งหมด

`client.force_login(user)` รับ **User instance ตรง ๆ** (ไม่ใช่ username/password)
แล้วฝัง session ให้ทันทีโดย**ไม่เรียก authentication backend หรือ hash password
เลย** ทำให้เร็วกว่ามากและใช้ได้แม้ password ของ user จะไม่รู้ (เช่น user ที่สร้าง
ผ่าน factory แบบสุ่ม password จะเจาะลึกใน Part 063):

```python
    def test_authenticated_user_can_access_create_page_with_force_login(self):
        self.client.force_login(self.user)
        response = self.client.get(self.url)
        self.assertEqual(response.status_code, 200)

    def test_force_login_can_switch_between_users_easily(self):
        other_user = User.objects.create_user(username="somchai", password="anotherpass")

        self.client.force_login(self.user)
        self.client.force_login(other_user)  # สลับ user ได้ทันทีโดยไม่ต้อง logout ก่อน

        response = self.client.get(self.url)
        self.assertEqual(response.status_code, 200)
        self.assertEqual(response.wsgi_request.user, other_user)
```

### 593.4 ตารางเปรียบเทียบ `client.login()` vs `client.force_login()`

| ประเด็น | `client.login()` | `client.force_login()` |
|---|---|---|
| ต้องรู้รหัสผ่าน plaintext | ✅ ต้องรู้ | ❌ ไม่ต้องรู้เลย |
| ผ่าน authentication backend จริง | ✅ ผ่านเต็มรูปแบบ (hash check) | ❌ ข้ามไปเลย ฝัง session ตรง ๆ |
| ความเร็ว | ช้ากว่าเล็กน้อย (คำนวณ password hash) | เร็วกว่า |
| คืนค่า boolean บอกผลลัพธ์ | ✅ คืน `True`/`False` | ❌ ไม่คืนค่า (ไม่มีทางล้มเหลวถ้า user ถูกต้อง) |
| เหมาะกับ | เทสต์ที่ต้องการยืนยัน**ระบบ login ทำงานถูกต้องจริง** (เช่น เทสต์หน้า login เอง) | เทสต์ view อื่น ๆ ที่ **สมมติว่า login สำเร็จแล้ว** และสนใจแค่พฤติกรรมหลัง login |
| ใช้บ่อยแค่ไหนในทางปฏิบัติ | เฉพาะเทสต์ระบบ authentication โดยตรง | ใช้เป็นค่าเริ่มต้นเกือบทุกเทสต์ view ที่ต้อง login |

**คำแนะนำของหลักสูตรนี้**: ใช้ `force_login()` เป็นค่าเริ่มต้นเสมอเมื่อเทสต์
view ที่ไม่ได้เกี่ยวกับระบบ login โดยตรง เพราะเร็วกว่าและโค้ดเทสต์อ่านง่ายกว่า
(ไม่ต้องนึกรหัสผ่านปลอมทุกครั้ง) — ใช้ `login()` เฉพาะตอนเทสต์ฟีเจอร์ login/
authentication เอง (เช่น "ใส่รหัสผ่านผิด 3 ครั้งแล้ว lock account" ที่จะเจาะลึก
ใน Part 033)

### 593.5 ทดสอบ Redirect เมื่อไม่ Login พร้อม `?next=`

```python
    def test_redirect_url_contains_next_parameter_pointing_back(self):
        response = self.client.get(self.url)
        expected_next = f"?next={self.url}"
        self.assertIn(expected_next, response.url)
```

`?next=` คือ query parameter มาตรฐานที่ `login_required` เติมให้อัตโนมัติ เพื่อ
ให้หน้า login รู้ว่า **login สำเร็จแล้วต้องพากลับไปหน้าไหน** — เราจะเจาะลึกกลไก
นี้เต็มรูปแบบใน Part 031

---

## ขั้นตอนที่ 594: Testing Forms — `is_valid()`, `form.errors`, testing custom `clean()`/`clean_<field>()`

### 594.1 หลักการ: ทดสอบ Form แยกจาก View เสมอเมื่อทำได้

Form มี logic ของตัวเองที่ทดสอบได้**โดยไม่ต้องผ่าน HTTP request เลย** — สร้าง
instance ของ Form ตรง ๆ พร้อม `data` dictionary แล้วเรียก `is_valid()` ได้ทันที
วิธีนี้เร็วกว่าทดสอบผ่าน `self.client.post()` มาก (ไม่ต้องผ่าน URL routing,
middleware, การ render template) และทำให้รู้ทันทีว่า**บั๊กอยู่ที่ Form หรือ View**
เมื่อเทสต์ตัวใดตัวหนึ่งพัง

```python
# blog/tests/test_forms.py
from django.test import TestCase

from blog.forms import CommentForm, PostForm


class CommentFormTests(TestCase):
    def test_valid_data_passes(self):
        form = CommentForm(data={
            "author": "สมชาย",
            "text": "ความคิดเห็นที่ยาวพอสมควรครับ",
        })
        self.assertTrue(form.is_valid())

    def test_missing_author_is_invalid(self):
        form = CommentForm(data={"author": "", "text": "มีเนื้อหา"})
        self.assertFalse(form.is_valid())
        self.assertIn("author", form.errors)

    def test_missing_text_is_invalid(self):
        form = CommentForm(data={"author": "สมชาย", "text": ""})
        self.assertFalse(form.is_valid())
        self.assertIn("text", form.errors)
```

### 594.2 ทดสอบ `form.errors` แบบเจาะจงข้อความ

การเช็คแค่ `assertIn("field", form.errors)` บอกได้แค่ว่า "field นี้มี error"
แต่ในงานจริงบางครั้งต้องมั่นใจว่า **ข้อความ error ตรงกับที่ตั้งใจไว้เป๊ะ ๆ**
โดยเฉพาะเมื่อกำหนด `error_messages` เองใน `Meta` (ทบทวนจาก Part 026 ขั้นตอนที่
251.7):

```python
    def test_error_message_content_for_missing_author(self):
        form = CommentForm(data={"author": "", "text": "มีเนื้อหา"})
        form.is_valid()
        self.assertEqual(form.errors["author"], ["This field is required."])
```

> **ข้อควรระวังเรื่องภาษา**: ข้อความ error เริ่มต้นของ Django เป็นภาษาอังกฤษ
> เสมอ ไม่ว่า `LANGUAGE_CODE` ในโปรเจกต์จะตั้งเป็นอะไรก็ตาม **นอกจากจะเปิดใช้
> Django's i18n translation catalog ที่มีคำแปลไทยไว้แล้ว** (เจาะลึกเรื่อง i18n
> เต็มรูปแบบใน Phase 10) ถ้าเทสต์ของคุณเทียบข้อความ error ตรง ๆ แบบนี้ ต้องรู้
> ว่ากำลังเทียบกับข้อความในภาษาที่ระบบ**กำลังทำงานอยู่จริง** ไม่ใช่ข้อความไทยที่
> อาจยังไม่มีคำแปล

### 594.3 ทดสอบ Custom `clean_slug()` ของ `PostForm`

จาก Part 026 ขั้นตอนที่ 254.4, `PostForm.clean_slug()` เพิ่มกฎ "slug ห้ามขึ้นต้น
ด้วยตัวเลข" ที่ Model validator ไม่ครอบคลุม:

```python
class PostFormTests(TestCase):
    def valid_data(self, **overrides):
        data = {
            "title": "หัวข้อที่ยาวพอสมควร",
            "slug": "valid-slug",
            "content": "เนื้อหาปกติทั่วไปไม่มีคำต้องห้าม",
            "is_published": False,
        }
        data.update(overrides)
        return data

    def test_slug_starting_with_digit_is_rejected_by_clean_slug(self):
        form = PostForm(data=self.valid_data(slug="123-invalid"))
        self.assertFalse(form.is_valid())
        self.assertEqual(form.errors["slug"], ["Slug ห้ามขึ้นต้นด้วยตัวเลข"])

    def test_slug_starting_with_letter_passes_clean_slug(self):
        form = PostForm(data=self.valid_data(slug="valid-slug"))
        self.assertTrue(form.is_valid())
```

### 594.4 ทดสอบว่า Model Validator "ไหลผ่าน" มาถึง ModelForm จริง (`_post_clean()`)

จาก Part 026 ขั้นตอนที่ 254.2 คุณรู้ว่า `ModelForm.is_valid()` เรียก
`instance.full_clean()` ให้อัตโนมัติผ่าน `_post_clean()` — เทสต์ต่อไปนี้ยืนยันว่า
พฤติกรรมนี้**ยังทำงานถูกต้องแม้ Form จะไม่ได้เขียน validation อะไรเพิ่มเติมเอง
เลย**:

```python
    def test_model_validator_min_length_flows_into_modelform_errors(self):
        """title ต้องยาวอย่างน้อย 5 ตัวอักษร (MinLengthValidator ใน models.py)
        แม้ PostForm จะไม่มี clean_title() เขียนไว้เองเลยก็ตาม"""
        form = PostForm(data=self.valid_data(title="สั้น"))
        self.assertFalse(form.is_valid())
        self.assertIn("title", form.errors)

    def test_model_validator_banned_words_flows_into_modelform_errors(self):
        form = PostForm(data=self.valid_data(content="คลิกที่นี่ด่วน"))
        self.assertFalse(form.is_valid())
        self.assertIn("content", form.errors)

    def test_clean_slug_runs_before_model_validators_but_both_report_together(self):
        """ส่งข้อมูลที่ผิดทั้ง clean_slug() (custom) และ MinLengthValidator (Model)
        พร้อมกัน — form.errors ต้องมี error ครบทั้งสอง field"""
        form = PostForm(data=self.valid_data(slug="1-invalid", title="สั้น"))
        self.assertFalse(form.is_valid())
        self.assertIn("slug", form.errors)
        self.assertIn("title", form.errors)
```

### 594.5 ทดสอบ `save(commit=False)` และ `save_m2m()`

ทบทวนจาก Part 026 ขั้นตอนที่ 252.2-252.3: field ที่ฟอร์มไม่รู้จัก (`author`) ต้อง
เติมเองก่อน `.save()` จริง และ M2M field (`tags`) ต้องเรียก `save_m2m()` แยก
ต่างหากหลัง instance มี `pk` แล้ว — นี่คือจุดที่เทสต์ควรครอบคลุมเป็นพิเศษ เพราะ
เป็นบั๊กที่ "ดูเหมือนใช้ได้" แต่ข้อมูลหายเงียบ ๆ ถ้าลืมขั้นตอนใดขั้นตอนหนึ่ง:

```python
from django.contrib.auth import get_user_model

from blog.models import Tag

User = get_user_model()


class PostFormSaveTests(TestCase):
    def setUp(self):
        self.user = User.objects.create_user(username="malee", password="pass123456")
        self.tag_python = Tag.objects.create(name="python")
        self.tag_django = Tag.objects.create(name="django")

    def test_commit_false_does_not_save_to_database_yet(self):
        form = PostForm(data={
            "title": "หัวข้อที่ยาวพอสมควร", "slug": "not-saved-yet",
            "content": "เนื้อหาปกติทั่วไป", "is_published": False,
            "tags": [self.tag_python.pk],
        })
        self.assertTrue(form.is_valid())
        post = form.save(commit=False)

        self.assertIsNone(post.pk)  # ยังไม่ถูก INSERT จริง
        self.assertEqual(Post.objects.count(), 0)

    def test_full_save_flow_with_m2m_persists_tags_correctly(self):
        form = PostForm(data={
            "title": "หัวข้อที่ยาวพอสมควร", "slug": "with-tags",
            "content": "เนื้อหาปกติทั่วไป", "is_published": True,
            "tags": [self.tag_python.pk, self.tag_django.pk],
        })
        self.assertTrue(form.is_valid())

        post = form.save(commit=False)
        post.author = self.user   # เติม field ที่ฟอร์มไม่รู้จัก
        post.save()
        form.save_m2m()           # ขั้นตอนที่ห้ามลืมเด็ดขาด

        post.refresh_from_db()
        self.assertEqual(post.tags.count(), 2)
        self.assertIn(self.tag_python, post.tags.all())
        self.assertIn(self.tag_django, post.tags.all())

    def test_forgetting_save_m2m_loses_tags_silently(self):
        """เทสต์นี้จงใจแสดงบั๊กคลาสสิกจาก Part 026 ขั้นตอนที่ 252.3
        เพื่อเป็นตัวอย่างเตือนใจว่าทำไมต้องเขียนเทสต์คู่กับ save_m2m() เสมอ"""
        form = PostForm(data={
            "title": "หัวข้อที่ยาวพอสมควร", "slug": "forgot-m2m",
            "content": "เนื้อหาปกติทั่วไป", "is_published": True,
            "tags": [self.tag_python.pk],
        })
        self.assertTrue(form.is_valid())

        post = form.save(commit=False)
        post.author = self.user
        post.save()
        # ตั้งใจไม่เรียก form.save_m2m() เพื่อพิสูจน์ว่า tags จะหายไปเงียบ ๆ

        post.refresh_from_db()
        self.assertEqual(post.tags.count(), 0)  # ยืนยันบั๊ก: ควรเป็น 1 แต่ได้ 0
```

> **ประโยชน์ของเทสต์ตัวสุดท้าย**: การเขียนเทสต์ที่ "พิสูจน์บั๊กที่รู้จักอยู่แล้ว"
> แบบนี้ดูขัดสามัญสำนึกในตอนแรก แต่มีประโยชน์จริงเมื่อใช้เป็น **documentation ที่
> รันได้** (executable documentation) — สมาชิกทีมใหม่ที่มาอ่านเทสต์นี้จะเข้าใจ
> ทันทีว่าทำไมโค้ดจริงใน view ถึงต้องเรียก `form.save_m2m()` เสมอ โดยไม่ต้องไป
> อ่านคอมเมนต์หรือเอกสารแยกต่างหาก

---

## ขั้นตอนที่ 595: Testing Class-Based Views ด้วย `RequestFactory` (Unit-level เทียบกับผ่าน Client)

### 595.1 `RequestFactory` คืออะไร และต่างจาก `self.client` อย่างไร

`django.test.RequestFactory` สร้าง **`HttpRequest` object ดิบ ๆ** โดยไม่ผ่าน
URL dispatcher, middleware stack, หรือระบบ session/authentication ใด ๆ เลย —
มันคือเครื่องมือสำหรับเทสต์ **view function หรือ view class เพียงตัวเดียวแบบ
โดดเดี่ยว (unit test ที่แท้จริง)** ต่างจาก `self.client` ที่จำลอง "การเข้าเว็บ
แบบเต็มรูปแบบ" (integration test ระดับ HTTP)

```python
from django.contrib.auth import get_user_model
from django.test import RequestFactory, TestCase

from blog.models import Post
from blog.views import PostListView, PostDetailView

User = get_user_model()
```

### 595.2 ทดสอบ `PostListView` แบบ Unit-level ด้วย `RequestFactory`

```python
class PostListViewRequestFactoryTests(TestCase):
    def setUp(self):
        self.factory = RequestFactory()
        self.user = User.objects.create_user(username="malee", password="pass123456")
        Post.objects.create(
            title="บทความที่เผยแพร่แล้ว", slug="published-1",
            content="เนื้อหา", author=self.user, is_published=True,
        )
        Post.objects.create(
            title="ฉบับร่าง", slug="draft-1",
            content="เนื้อหา", author=self.user, is_published=False,
        )

    def test_get_returns_200_via_request_factory(self):
        request = self.factory.get("/")
        response = PostListView.as_view()(request)
        # CBV คืน TemplateResponse ที่ยังไม่ render จนกว่าจะเรียก .render() เอง
        response.render()
        self.assertEqual(response.status_code, 200)

    def test_context_data_excludes_draft_posts(self):
        request = self.factory.get("/")
        response = PostListView.as_view()(request)
        response.render()
        posts = list(response.context_data["posts"])
        self.assertEqual(len(posts), 1)
        self.assertEqual(posts[0].slug, "published-1")
```

> **จุดสำคัญที่ต้องจำ**: เมื่อเรียก CBV ผ่าน `View.as_view()(request)` ตรง ๆ
> ผลลัพธ์ที่ได้เป็น `TemplateResponse` ซึ่ง**ยัง lazy อยู่** (ยังไม่ render HTML
> จริง) ต้องเรียก `response.render()` เองก่อนถึงจะเข้าถึง `response.content`
> หรือ `response.context_data` ได้ครบ — ต่างจากตอนใช้ `self.client.get()` ที่
> Django เรียก `.render()` ให้อัตโนมัติเป็นส่วนหนึ่งของ request-response cycle
> เต็มรูปแบบอยู่แล้ว (สังเกตด้วยว่าใช้ `response.context_data` ไม่ใช่
> `response.context` เมื่อเรียกผ่าน `RequestFactory` ตรง ๆ — `response.context`
> เป็น attribute พิเศษที่ `self.client` เติมให้ทีหลังเท่านั้น)

### 595.3 ทดสอบ `get_queryset()` โดยตรง โดยไม่ต้องสร้าง Request เลยด้วยซ้ำ

ระดับที่ "unit" ที่สุดคือการเรียก method ของ view class **ตรง ๆ โดยไม่ผ่าน
`dispatch()` เลย** — ทำได้เพราะ `get_queryset()` ใช้แค่ `self.request` เท่านั้น
เราจึงสร้าง instance ของ view เอง แล้วเซ็ต `.request` ให้ด้วยมือ:

```python
    def test_get_queryset_directly_without_full_dispatch_cycle(self):
        view = PostListView()
        view.request = self.factory.get("/")
        queryset = view.get_queryset()

        self.assertEqual(queryset.count(), 1)
        self.assertTrue(all(post.is_published for post in queryset))
```

วิธีนี้เร็วที่สุดเพราะข้าม `dispatch()`, `http_method_not_allowed()`, และ
middleware ทั้งหมด — เหมาะกับการเทสต์ logic ที่ซับซ้อนภายใน method เดียว แต่
**ไม่ครอบคลุม** พฤติกรรมที่เกิดจาก method อื่นที่ทำงานร่วมกัน (เช่น
`get_context_data()` ที่พึ่ง `self.object_list` ซึ่งถูกเซ็ตใน `get()`)

### 595.4 ทดสอบ `DetailView.post()` (custom method จาก Part 060 ขั้นตอนที่ 591.1) ด้วย `RequestFactory`

```python
class PostDetailViewCommentRequestFactoryTests(TestCase):
    def setUp(self):
        self.factory = RequestFactory()
        self.user = User.objects.create_user(username="malee", password="pass123456")
        self.post = Post.objects.create(
            title="บทความสำหรับทดสอบ", slug="rf-comment-test",
            content="เนื้อหา", author=self.user, is_published=True,
        )

    def test_post_comment_via_request_factory_creates_comment(self):
        request = self.factory.post("/", data={
            "author": "สมชาย", "text": "ความเห็นที่ยาวพอสมควร",
        })
        response = PostDetailView.as_view()(request, slug=self.post.slug)

        self.assertEqual(response.status_code, 302)
        self.assertEqual(self.post.comments.count(), 1)
```

### 595.5 ตารางเปรียบเทียบ `RequestFactory` vs `self.client`

| ประเด็น | `RequestFactory` | `self.client` |
|---|---|---|
| ผ่าน URL dispatcher (`urls.py`) จริงไหม | ❌ ไม่ผ่าน (เรียก view โดยตรง ต้องใส่ URL kwargs เอง) | ✅ ผ่านเต็มรูปแบบ (ใช้ `reverse()` ได้) |
| ผ่าน Middleware stack ไหม | ❌ ไม่ผ่านเลย | ✅ ผ่านทุกตัวใน `MIDDLEWARE` |
| Session/Authentication พร้อมใช้ทันทีไหม | ❌ ต้องเซ็ต `request.user` เองด้วยมือถ้าต้องการ | ✅ จัดการให้ผ่าน `login()`/`force_login()` |
| ความเร็ว | เร็วที่สุด (unit test แท้จริง) | ช้ากว่าเล็กน้อย (integration test) |
| `response.context` ใช้ได้ทันทีไหม | ❌ ต้องเรียก `.render()` ก่อน แล้วใช้ `.context_data` | ✅ ใช้ `.context` ได้ทันที |
| เหมาะกับ | เทสต์ logic เฉพาะจุดของ view/method เดียว แบบแยกส่วนสมบูรณ์ | เทสต์พฤติกรรมจริงของทั้งระบบตั้งแต่ URL ถึง response (ค่าเริ่มต้นที่แนะนำสำหรับเทสต์ view ส่วนใหญ่) |

**คำแนะนำของหลักสูตรนี้**: ใช้ `self.client` เป็นค่าเริ่มต้นเสมอสำหรับเทสต์ view
ทั่วไป (ครอบคลุมมากกว่าและใกล้เคียงพฤติกรรมจริงที่ผู้ใช้เจอ) แล้วสลับมาใช้
`RequestFactory` เฉพาะเมื่อต้องการแยกทดสอบ method เดียวของ CBV ที่ซับซ้อนมาก
โดยไม่อยากให้ผลเทสต์ปนกับ URL routing หรือ middleware อื่น

---

## ขั้นตอนที่ 596: Testing การ Render Template — `assertTemplateUsed`, `assertContains`, `assertNotContains`

### 596.1 `assertTemplateUsed`: ยืนยันว่า View Render Template ที่ถูกต้อง

```python
# blog/tests/test_views.py (ต่อ)
class PostTemplateTests(TestCase):
    def setUp(self):
        self.user = User.objects.create_user(username="malee", password="pass123456")
        self.post = Post.objects.create(
            title="บทความสำหรับทดสอบ Template", slug="template-test",
            content="เนื้อหาปกติ", author=self.user, is_published=True,
        )

    def test_list_view_uses_correct_template(self):
        response = self.client.get(reverse("blog:list"))
        self.assertTemplateUsed(response, "blog/post_list.html")

    def test_detail_view_uses_correct_template(self):
        url = reverse("blog:detail", kwargs={"slug": self.post.slug})
        response = self.client.get(url)
        self.assertTemplateUsed(response, "blog/post_detail.html")

    def test_response_also_uses_base_template(self):
        """ตรวจว่า template แม่ (base.html) ถูก extends จริง — ป้องกัน
        กรณีลืมเขียน {% extends "base.html" %} ในเทมเพลตลูกโดยไม่มีใครสังเกต"""
        response = self.client.get(reverse("blog:list"))
        self.assertTemplateUsed(response, "base.html")
```

`assertTemplateUsed` ทำงานได้เพราะ Test Client บันทึก**รายการ template ทั้งหมด
ที่ถูก render ระหว่าง request นั้น** ไว้ใน `response.templates` — ไม่ว่าจะเป็น
template หลักหรือ template ที่ถูก `{% extends %}`/`{% include %}` เข้ามาก็ตาม
ทั้งหมดจะถูกนับด้วย

### 596.2 `assertContains` / `assertNotContains`: ตรวจ HTML ที่ Render ออกมาจริง

สอง assertion นี้ตรวจ **`response.content`** (HTML ดิบที่ส่งกลับไปเบราว์เซอร์
จริง) — ต่างจาก `response.context` ที่ตรวจแค่ข้อมูล**ก่อน** render:

```python
    def test_list_view_shows_published_post_title(self):
        response = self.client.get(reverse("blog:list"))
        self.assertContains(response, "บทความสำหรับทดสอบ Template")

    def test_list_view_does_not_show_draft_post_title(self):
        Post.objects.create(
            title="หัวข้อฉบับร่างที่ไม่ควรเห็น", slug="hidden-draft",
            content="เนื้อหา", author=self.user, is_published=False,
        )
        response = self.client.get(reverse("blog:list"))
        self.assertNotContains(response, "หัวข้อฉบับร่างที่ไม่ควรเห็น")

    def test_detail_view_shows_reading_time(self):
        url = reverse("blog:detail", kwargs={"slug": self.post.slug})
        response = self.client.get(url)
        expected_text = f"{self.post.reading_time_minutes()} นาที"
        self.assertContains(response, expected_text)

    def test_assertContains_checks_status_code_by_default_too(self):
        """assertContains เช็ค status_code == 200 ให้อัตโนมัติเป็นค่าเริ่มต้น
        ถ้าหน้าเป็น 404/500 มันจะ fail ทันทีก่อนแม้แต่จะไปดู content"""
        response = self.client.get("/does-not-exist/")
        with self.assertRaises(AssertionError):
            self.assertContains(response, "อะไรก็ได้")
```

### 596.3 ระบุจำนวนครั้งที่คาดว่าจะเจอด้วย `count=`

`assertContains` รับ argument `count=` เพื่อตรวจว่าข้อความปรากฏ**กี่ครั้ง**พอดี
(ไม่ใช่แค่ "มีอย่างน้อย 1 ครั้ง") มีประโยชน์มากเมื่อเทสต์รายการที่แสดงซ้ำ เช่น
จำนวนคอมเมนต์ที่ถูก render:

```python
    def test_detail_view_renders_every_comment_exactly_once(self):
        from blog.models import Comment

        Comment.objects.create(post=self.post, author="สมชาย", text="ความเห็นที่หนึ่ง")
        Comment.objects.create(post=self.post, author="สมหญิง", text="ความเห็นที่สอง")

        url = reverse("blog:detail", kwargs={"slug": self.post.slug})
        response = self.client.get(url)

        self.assertContains(response, "ความเห็นที่หนึ่ง", count=1)
        self.assertContains(response, "ความเห็นที่สอง", count=1)
```

### 596.4 ทดสอบ Pagination ผ่าน Template ที่ Render จริง

ทบทวนจาก Part 022 ขั้นตอนที่ 212: `PostListView` เปิด `paginate_by = 10` เทสต์
ต่อไปนี้ยืนยันว่า pagination **ทำงานจริงทั้งฝั่ง context และฝั่ง HTML**:

```python
    def test_pagination_link_appears_when_more_than_one_page(self):
        for i in range(15):
            Post.objects.create(
                title=f"บทความที่ {i}", slug=f"paginated-post-{i}",
                content="เนื้อหา", author=self.user, is_published=True,
            )
        response = self.client.get(reverse("blog:list"))

        self.assertTrue(response.context["is_paginated"])
        self.assertEqual(len(response.context["posts"]), 10)
        self.assertContains(response, "ถัดไป")

    def test_pagination_link_does_not_appear_on_single_page(self):
        response = self.client.get(reverse("blog:list"))  # มีแค่ 1 post จาก setUp
        self.assertFalse(response.context["is_paginated"])
        self.assertNotContains(response, "ถัดไป")
```

---

## ขั้นตอนที่ 597: Testing Redirect — `assertRedirects` เจาะลึก (Status Code, Target URL, `follow=True`)

### 597.1 ปัญหาของการเช็คแค่ `status_code == 302`

การเช็คแค่ `self.assertEqual(response.status_code, 302)` **ยืนยันแค่ว่า "มีการ
redirect เกิดขึ้น"** แต่ไม่ยืนยันว่า **redirect ไปที่ถูกต้องหรือไม่** — ถ้า view
พลาด redirect ไปผิดหน้า (เช่น bug ที่ hardcode URL ผิด) เทสต์แบบเช็คแค่
status code จะยังคง**ผ่าน**ทั้งที่พฤติกรรมจริงพังไปแล้ว `assertRedirects` แก้
ปัญหานี้โดยตรวจทั้ง URL ปลายทางและ status code ในคำสั่งเดียว

### 597.2 `assertRedirects` พื้นฐาน

```python
# blog/tests/test_views.py (ต่อ)
class CommentRedirectTests(TestCase):
    def setUp(self):
        self.user = User.objects.create_user(username="malee", password="pass123456")
        self.post = Post.objects.create(
            title="บทความสำหรับทดสอบ redirect", slug="redirect-test",
            content="เนื้อหา", author=self.user, is_published=True,
        )
        self.url = reverse("blog:detail", kwargs={"slug": self.post.slug})

    def test_valid_comment_redirects_to_post_detail_page(self):
        response = self.client.post(self.url, data={
            "author": "สมชาย", "text": "ความเห็นที่ยาวพอสมควร",
        })
        self.assertRedirects(response, self.url)
```

พฤติกรรมภายในของ `assertRedirects(response, expected_url)` ทำ 3 อย่างพร้อมกัน:

1. ตรวจว่า `response.status_code` เป็นโค้ด redirect ที่คาดไว้ (ค่า default
   `target_status_code=200` คือสถานะของหน้าปลายทาง**หลังตามลิงก์ไปแล้ว**)
2. **ตามลิงก์ redirect ไปจริง** (ยิง request ที่สองไปยัง URL ปลายทางให้อัตโนมัติ)
3. ตรวจว่าหน้าปลายทางนั้นตอบกลับด้วย status code ตามข้อ 1 (ปกติคือ `200`)

### 597.3 พารามิเตอร์ครบของ `assertRedirects`

```python
def assertRedirects(
    response,
    expected_url,
    status_code=302,
    target_status_code=200,
    fetch_redirect_response=True,
):
    ...
```

| พารามิเตอร์ | ความหมาย | ใช้เมื่อไหร่ |
|---|---|---|
| `expected_url` | URL ปลายทางที่คาดว่า response จะ redirect ไป (**บังคับ**) | ทุกครั้ง |
| `status_code` | status code ของ response ตัวแรก (ค่า default `302`) | เปลี่ยนเมื่อ view ใช้ `301` (permanent redirect) แทน |
| `target_status_code` | status code ที่คาดว่าหน้าปลายทางจะตอบกลับ | เปลี่ยนเมื่อ redirect ไปหน้าที่ตอบ `404`/`403` โดยตั้งใจ (พบน้อย) |
| `fetch_redirect_response` | ถ้า `False` จะ**ไม่ตามลิงก์ไปจริง** เช็คแค่ URL ปลายทางเฉย ๆ | ใช้เมื่อหน้าปลายทางต้องพึ่งพา state ที่เทสต์นี้ยังไม่ได้เตรียมไว้ (เช่น redirect ไป external URL ที่ Django มองไม่เห็น) |

```python
    def test_redirect_to_external_url_without_fetching(self):
        """ตัวอย่างสมมติ: redirect ไป URL ภายนอกที่ Test Client เข้าถึงไม่ได้"""
        # (ตัวอย่างนี้จำลอง view ที่ redirect ไป URL ภายนอก)
        from django.http import HttpResponseRedirect
        from django.test import RequestFactory

        request = RequestFactory().get("/")
        response = HttpResponseRedirect("https://external-payment-gateway.example.com/")
        self.assertRedirects(
            response, "https://external-payment-gateway.example.com/",
            fetch_redirect_response=False,
        )
```

### 597.4 ทดสอบ Redirect ของ `login_required` แบบเจาะลึกด้วย `assertRedirects`

ผสานความรู้จากขั้นตอนที่ 593.5 (`?next=`) เข้ากับ `assertRedirects` เต็มรูปแบบ:

```python
class PostCreateRedirectTests(TestCase):
    def test_anonymous_user_redirect_target_is_exact(self):
        create_url = reverse("blog:create")
        response = self.client.get(create_url)

        expected_login_url = f"/accounts/login/?next={create_url}"
        self.assertRedirects(response, expected_login_url)
```

> **ข้อควรระวัง**: เทสต์ข้างต้นจะ**ผ่านก็ต่อเมื่อมี URL pattern ชื่อ
> `accounts/login/` ที่ตอบกลับ `200` จริงในโปรเจกต์แล้วเท่านั้น** เพราะ
> `assertRedirects` ตามลิงก์ไปจริงตามที่อธิบายในขั้นตอนที่ 597.2 ถ้ายังไม่ได้ตั้ง
> ค่าระบบ login (จะเรียนเต็มรูปแบบใน Part 031) ให้ใช้
> `fetch_redirect_response=False` ไปก่อนชั่วคราว

### 597.5 `follow=True`: ตามลิงก์ Redirect ตั้งแต่ตอนยิง Request เลย

บางครั้งเราไม่ได้สนใจจะเช็ค URL ปลายทางแบบเจาะจง แต่อยากรู้ว่า **หลัง redirect
ไปแล้ว หน้าสุดท้ายแสดงเนื้อหาถูกต้องไหม** — ส่ง `follow=True` เข้าไปใน
`self.client.post()`/`get()` เพื่อให้ Test Client ตามลิงก์ redirect ให้อัตโนมัติ
ในการเรียกครั้งเดียว:

```python
    def test_valid_comment_redirect_followed_shows_new_comment_on_page(self):
        user = User.objects.create_user(username="malee", password="pass123456")
        post = Post.objects.create(
            title="บทความทดสอบ follow", slug="follow-test",
            content="เนื้อหา", author=user, is_published=True,
        )
        url = reverse("blog:detail", kwargs={"slug": post.slug})

        response = self.client.post(url, data={
            "author": "สมชาย", "text": "ความเห็นที่เพิ่งโพสต์ไป",
        }, follow=True)

        self.assertEqual(response.status_code, 200)  # สถานะของหน้าสุดท้ายหลังตาม redirect
        self.assertContains(response, "ความเห็นที่เพิ่งโพสต์ไป")
        # response.redirect_chain เก็บลำดับการ redirect ทั้งหมดที่เกิดขึ้น
        self.assertEqual(response.redirect_chain, [(url, 302)])
```

`response.redirect_chain` เป็น list ของ tuple `(url, status_code)` ตามลำดับที่
ถูก redirect ผ่าน — มีประโยชน์มากเมื่อเทสต์ระบบที่ redirect **หลายต่อ** (เช่น
A → B → C) เพราะบอกได้ว่าแต่ละขั้นตอนไปที่ไหนบ้าง

### 597.6 ตารางสรุป: เมื่อไรใช้ `assertRedirects` เมื่อไรใช้ `follow=True`

| สถานการณ์ | วิธีที่เหมาะสม |
|---|---|
| ต้องการยืนยัน URL ปลายทางเป๊ะ ๆ พร้อม status code ครบ | `assertRedirects(response, expected_url)` |
| ต้องการดูเนื้อหาของหน้าสุดท้ายหลังตาม redirect (ไม่สนใจ URL เป๊ะ) | `self.client.post(url, data, follow=True)` แล้วเช็ค `response.content`/`assertContains` |
| Redirect หลายต่อ ต้องการดูลำดับทั้งหมด | `follow=True` แล้วตรวจ `response.redirect_chain` |
| Redirect ไปหน้าที่ Test Client เข้าถึงไม่ได้ (external URL) | `assertRedirects(..., fetch_redirect_response=False)` |

---

## ขั้นตอนที่ 598: Testing การปรับแต่ง Django Admin (Custom Action, `list_display`)

### 598.1 ทบทวน/เพิ่ม `PostAdmin` สำหรับ Part นี้

ก่อนเขียนเทสต์ ต้องมี Admin ที่ปรับแต่งแล้วให้ทดสอบก่อน — นี่คือ `PostAdmin` ที่
เพิ่ม `list_display` และ **custom action** `mark_as_published` (แนวคิด custom
action จะเจาะลึกเต็มรูปแบบใน Part 040 เรื่อง Admin ขั้นสูง แต่ในที่นี้เราสร้าง
เวอร์ชันที่ใช้งานได้จริงเพื่อทดสอบ):

```python
# blog/admin.py
from django.contrib import admin

from .models import Category, Comment, Post, Tag


@admin.action(description="ทำเครื่องหมายว่าเผยแพร่แล้ว")
def mark_as_published(modeladmin, request, queryset):
    updated_count = queryset.update(is_published=True)
    modeladmin.message_user(request, f"เผยแพร่บทความสำเร็จ {updated_count} บทความ")


@admin.action(description="ยกเลิกการเผยแพร่ (เปลี่ยนเป็นฉบับร่าง)")
def mark_as_draft(modeladmin, request, queryset):
    updated_count = queryset.update(is_published=False)
    modeladmin.message_user(request, f"เปลี่ยนเป็นฉบับร่างสำเร็จ {updated_count} บทความ")


@admin.register(Post)
class PostAdmin(admin.ModelAdmin):
    list_display = ["title", "author", "category", "is_published", "created_at"]
    list_filter = ["is_published", "category"]
    search_fields = ["title", "content"]
    prepopulated_fields = {"slug": ("title",)}
    actions = [mark_as_published, mark_as_draft]


admin.site.register(Category)
admin.site.register(Tag)
admin.site.register(Comment)
```

### 598.2 ทดสอบว่าเข้าหน้า Admin Changelist ได้ และมีคอลัมน์ตาม `list_display`

การเทสต์ Django Admin ใช้หลักการเดียวกับเทสต์ view ทั่วไปทุกประการ (เพราะ Admin
ก็คือชุด view ที่ Django สร้างให้อัตโนมัติ) เพียงแต่ต้อง **login เป็น staff user
เสมอ** (Admin บังคับ `is_staff=True` ไม่เช่นนั้นจะถูก redirect ไปหน้า login ของ
Admin):

```python
# blog/tests/test_admin.py
from django.contrib.auth import get_user_model
from django.test import TestCase
from django.urls import reverse

from blog.models import Category, Post

User = get_user_model()


class PostAdminChangelistTests(TestCase):
    def setUp(self):
        self.staff_user = User.objects.create_user(
            username="admin_user", password="adminpass123", is_staff=True,
        )
        self.regular_user = User.objects.create_user(
            username="regular_user", password="regularpass123",
        )
        self.category = Category.objects.create(name="เทคโนโลยี", slug="technology")
        self.post = Post.objects.create(
            title="บทความสำหรับทดสอบ Admin", slug="admin-test-post",
            content="เนื้อหา", author=self.staff_user,
            category=self.category, is_published=False,
        )
        self.changelist_url = reverse("admin:blog_post_changelist")

    def test_staff_user_can_access_changelist(self):
        self.client.force_login(self.staff_user)
        response = self.client.get(self.changelist_url)
        self.assertEqual(response.status_code, 200)

    def test_regular_user_cannot_access_admin(self):
        self.client.force_login(self.regular_user)
        response = self.client.get(self.changelist_url)
        self.assertEqual(response.status_code, 302)  # redirect ไปหน้า login ของ admin

    def test_anonymous_user_cannot_access_admin(self):
        response = self.client.get(self.changelist_url)
        self.assertEqual(response.status_code, 302)

    def test_changelist_shows_post_title(self):
        self.client.force_login(self.staff_user)
        response = self.client.get(self.changelist_url)
        self.assertContains(response, "บทความสำหรับทดสอบ Admin")

    def test_changelist_columns_match_list_display(self):
        """ตรวจว่า ModelAdmin.list_display ตั้งค่าตามที่ตั้งใจไว้จริง
        (unit test ระดับ configuration ไม่ต้องผ่าน HTTP เลยด้วยซ้ำ)"""
        from django.contrib import admin as django_admin
        post_admin = django_admin.site._registry[Post]
        self.assertEqual(
            list(post_admin.list_display),
            ["title", "author", "category", "is_published", "created_at"],
        )
```

> **สังเกต**: `test_changelist_columns_match_list_display` ไม่ได้ยิง HTTP
> request เลย แต่เข้าถึง `admin.site._registry` ตรง ๆ เพื่อดึง `ModelAdmin`
> instance ที่ลงทะเบียนไว้ — เป็นเทคนิคที่รวดเร็วเมื่อต้องการเช็คแค่
> **configuration** ของ Admin โดยไม่สนใจว่า HTML จะ render ออกมาถูกหรือไม่
> (ซึ่งเป็นหน้าที่ของเทสต์ระดับ HTTP อย่าง `test_changelist_shows_post_title`)

### 598.3 ทดสอบ Custom Action `mark_as_published`

Custom action ของ Django Admin ถูกเรียกผ่านการ **POST ไปยัง changelist URL**
พร้อม field พิเศษ 2 ตัวเสมอ: `action` (ชื่อ action ที่เลือกจาก dropdown) และ
`_selected_action` (list ของ pk ที่ติ๊กเลือกไว้) — ทดสอบโดยจำลอง POST แบบนี้
ตรง ๆ:

```python
class PostAdminActionTests(TestCase):
    def setUp(self):
        self.staff_user = User.objects.create_user(
            username="admin_user", password="adminpass123", is_staff=True,
        )
        self.draft_post_1 = Post.objects.create(
            title="ฉบับร่างที่ 1", slug="draft-1",
            content="เนื้อหา", author=self.staff_user, is_published=False,
        )
        self.draft_post_2 = Post.objects.create(
            title="ฉบับร่างที่ 2", slug="draft-2",
            content="เนื้อหา", author=self.staff_user, is_published=False,
        )
        self.already_published = Post.objects.create(
            title="เผยแพร่แล้ว", slug="already-published",
            content="เนื้อหา", author=self.staff_user, is_published=True,
        )
        self.changelist_url = reverse("admin:blog_post_changelist")
        self.client.force_login(self.staff_user)

    def test_mark_as_published_action_updates_selected_posts_only(self):
        response = self.client.post(self.changelist_url, data={
            "action": "mark_as_published",
            "_selected_action": [self.draft_post_1.pk, self.draft_post_2.pk],
        }, follow=True)

        self.assertEqual(response.status_code, 200)

        self.draft_post_1.refresh_from_db()
        self.draft_post_2.refresh_from_db()
        self.assertTrue(self.draft_post_1.is_published)
        self.assertTrue(self.draft_post_2.is_published)

    def test_mark_as_published_action_does_not_affect_unselected_posts(self):
        self.client.post(self.changelist_url, data={
            "action": "mark_as_published",
            "_selected_action": [self.draft_post_1.pk],  # เลือกแค่ post 1
        })

        self.draft_post_2.refresh_from_db()
        self.assertFalse(self.draft_post_2.is_published)  # ต้องยังเป็น draft เหมือนเดิม

    def test_mark_as_published_shows_success_message(self):
        response = self.client.post(self.changelist_url, data={
            "action": "mark_as_published",
            "_selected_action": [self.draft_post_1.pk, self.draft_post_2.pk],
        }, follow=True)

        messages = list(response.context["messages"])
        self.assertEqual(len(messages), 1)
        self.assertIn("2 บทความ", str(messages[0]))

    def test_mark_as_draft_action_reverses_publication(self):
        self.client.post(self.changelist_url, data={
            "action": "mark_as_draft",
            "_selected_action": [self.already_published.pk],
        })

        self.already_published.refresh_from_db()
        self.assertFalse(self.already_published.is_published)
```

### 598.4 ตารางสรุป: จุดที่ต้องทดสอบเมื่อปรับแต่ง Django Admin

| สิ่งที่ปรับแต่ง | ทดสอบอะไร |
|---|---|
| `list_display` | คอลัมน์ที่ตั้งค่าไว้ตรงกับที่ตั้งใจ (ผ่าน `admin.site._registry`) และแสดงข้อมูลถูกต้องในหน้า changelist |
| `list_filter` | สามารถกรองผ่าน query parameter ได้ถูกต้อง (เช่น `?is_published__exact=1`) |
| `search_fields` | ค้นหาผ่าน `?q=...` แล้วได้ผลลัพธ์ที่ถูกต้อง |
| Custom `@admin.action` | action ทำงานถูกต้องกับ record ที่เลือกเท่านั้น ไม่กระทบ record ที่ไม่ได้เลือก และแสดงข้อความยืนยันถูกต้อง |
| การเข้าถึง (permission) | staff เข้าได้, user ทั่วไปเข้าไม่ได้, anonymous ถูก redirect |
| `prepopulated_fields` | เป็น configuration ที่ทดสอบผ่านการเช็ค attribute ตรง ๆ เหมือน `list_display` (JavaScript ฝั่ง client ไม่ได้ถูกทดสอบด้วย unit test ปกติ — ต้องใช้ Selenium ตาม Part 064) |

---

## ขั้นตอนที่ 599: Testing Signals (เชื่อมกับ Part 019 — ทดสอบว่า `Profile` ถูกสร้างอัตโนมัติจริง)

### 599.1 ทบทวน Signal จาก Part 019 ที่ต้องทดสอบ

ทบทวนจากขั้นตอนที่ 591.1: `accounts/signals.py` มี receiver 2 ตัว —
`create_profile` (สร้าง `Profile` เมื่อ `User` ใหม่ถูกสร้าง) และ `save_profile`
(บันทึก `Profile` ที่มีอยู่ทุกครั้งที่ `User` ถูกอัปเดต) การทดสอบ signal มี
หลักการเดียวกับทดสอบ Model ทั่วไป: **สร้าง trigger แล้วเช็คผลลัพธ์** เพียงแต่
สิ่งที่เกิดขึ้นเป็น "ผลข้างเคียง" (side effect) ต่อ object อื่น ไม่ใช่ต่อ object
ที่เรากำลังบันทึกโดยตรง

```python
# accounts/tests/test_signals.py
from django.contrib.auth import get_user_model
from django.test import TestCase

from accounts.models import Profile

User = get_user_model()


class ProfileAutoCreationSignalTests(TestCase):
    def test_creating_user_automatically_creates_profile(self):
        user = User.objects.create_user(username="malee", password="pass123456")
        self.assertTrue(Profile.objects.filter(user=user).exists())

    def test_user_can_access_profile_immediately_without_error(self):
        """ยืนยันว่าปัญหา RelatedObjectDoesNotExist จาก Part 012 ขั้นตอนที่ 112.5
        ถูกแก้ไขแล้วอย่างถาวรจริง"""
        user = User.objects.create_user(username="malee", password="pass123456")
        try:
            profile = user.profile
        except Exception as exc:
            self.fail(f"user.profile ควรเข้าถึงได้ทันที แต่เกิด error: {exc}")
        self.assertIsInstance(profile, Profile)

    def test_profile_starts_with_empty_bio(self):
        user = User.objects.create_user(username="malee", password="pass123456")
        self.assertEqual(user.profile.bio, "")

    def test_superuser_created_via_create_superuser_also_gets_profile(self):
        """ทดสอบเส้นทางที่มักถูกลืม (ทบทวนขั้นตอนที่ 183.6 ของ Part 019):
        createsuperuser ก็ผ่าน save() เหมือนกัน จึง trigger signal เช่นกัน"""
        admin = User.objects.create_superuser(
            username="admin", email="admin@example.com", password="adminpass123",
        )
        self.assertTrue(Profile.objects.filter(user=admin).exists())
```

### 599.2 ทดสอบว่าไม่มีการสร้าง `Profile` ซ้ำตอน `User` ถูกอัปเดต

นี่คือเทสต์ที่ป้องกัน**การกลับมาของบั๊กคลาสสิก**จาก Part 019 ขั้นตอนที่ 183.3
(`IntegrityError: UNIQUE constraint failed`) หากในอนาคตมีใครมาแก้
`create_profile` แล้วลืมเช็ค `if created:` เทสต์นี้จะจับได้ทันที:

```python
    def test_updating_existing_user_does_not_create_duplicate_profile(self):
        user = User.objects.create_user(username="malee", password="pass123456")
        self.assertEqual(Profile.objects.filter(user=user).count(), 1)

        user.email = "malee_updated@example.com"
        user.save()   # เรียก post_save อีกครั้ง แต่ created=False

        self.assertEqual(Profile.objects.filter(user=user).count(), 1)

    def test_updating_user_multiple_times_never_duplicates_profile(self):
        user = User.objects.create_user(username="malee", password="pass123456")
        for i in range(5):
            user.email = f"malee{i}@example.com"
            user.save()

        self.assertEqual(Profile.objects.filter(user=user).count(), 1)

    def test_save_profile_receiver_persists_changes_made_to_profile(self):
        """ทดสอบ save_profile receiver (183.4): เมื่อ user.profile ถูกแก้ไข
        ในหน่วยความจำ แล้ว user.save() ถูกเรียก profile ต้องถูกบันทึกตามไปด้วย"""
        user = User.objects.create_user(username="malee", password="pass123456")
        user.profile.bio = "นักพัฒนา Django ตัวยง"
        # ตั้งใจไม่เรียก user.profile.save() ตรง ๆ เพื่อพิสูจน์ว่า save_profile ทำงานแทน
        user.save()

        user.profile.refresh_from_db()
        self.assertEqual(user.profile.bio, "นักพัฒนา Django ตัวยง")
```

### 599.3 ทดสอบด้วยการ Disconnect Signal ชั่วคราว (เทคนิคขั้นสูง)

บางสถานการณ์ (เช่น เทสต์ที่ต้องการสร้าง `User` จำนวนมากเพื่อเทสต์ performance
โดยไม่สนใจ `Profile` เลย) อาจต้องการ**ปิด signal ชั่วคราว**ระหว่างเทสต์ Django
อนุญาตให้ `disconnect()`/`connect()` receiver ได้ตรง ๆ เหมือนที่เรียนใน Part 019
ขั้นตอนที่ 182.1 — ควรทำในรูปแบบ context manager หรือใน `setUp()`/`tearDown()`
เพื่อไม่ให้กระทบเทสต์อื่นในไฟล์เดียวกัน:

```python
    def test_behavior_when_signal_is_temporarily_disconnected(self):
        from django.db.models.signals import post_save
        from accounts.signals import create_profile

        post_save.disconnect(create_profile, sender=User, dispatch_uid="accounts_create_profile")
        try:
            user = User.objects.create_user(username="no_profile_user", password="pass123456")
            self.assertFalse(Profile.objects.filter(user=user).exists())
        finally:
            # เชื่อมกลับเสมอใน finally เพื่อไม่ให้เทสต์อื่นในไฟล์เดียวกันได้รับผลกระทบ
            post_save.connect(create_profile, sender=User, dispatch_uid="accounts_create_profile")
```

> **คำเตือนสำคัญ**: ต้อง `.connect()` กลับใน `finally` เสมอ (หรือใช้
> `self.addCleanup(...)` ซึ่งเชื่อถือได้กว่า เพราะทำงานแม้ assertion ก่อนหน้าจะ
> fail) มิเช่นนั้น receiver จะยังคง**ถูกตัดการเชื่อมต่อไปตลอดทั้ง test run**
> (signal registry เป็น global state ที่ใช้ร่วมกันข้ามทุก test case ในโปรเซส
> เดียวกัน) ทำให้เทสต์อื่นที่รันหลังจากนี้พังตามไปด้วยอย่างมึนงง — รูปแบบที่
> ปลอดภัยกว่าคือใช้ `self.addCleanup()`:

```python
    def test_behavior_when_signal_disconnected_using_addCleanup(self):
        from django.db.models.signals import post_save
        from accounts.signals import create_profile

        post_save.disconnect(create_profile, sender=User, dispatch_uid="accounts_create_profile")
        self.addCleanup(
            post_save.connect, create_profile, sender=User,
            dispatch_uid="accounts_create_profile",
        )

        user = User.objects.create_user(username="no_profile_user_2", password="pass123456")
        self.assertFalse(Profile.objects.filter(user=user).exists())
```

`self.addCleanup(func, *args, **kwargs)` ลงทะเบียนฟังก์ชันให้ Django เรียก
**หลังจาก test method จบเสมอ ไม่ว่าจะผ่านหรือ fail ก็ตาม** (ทำงานคล้าย `finally`
แต่เขียนสั้นกว่าและอ่านเจตนาได้ชัดเจนกว่า) — เป็นรูปแบบที่แนะนำเสมอเมื่อเทสต์ต้อง
เปลี่ยน global state ชั่วคราว

### 599.4 ตารางสรุป: แนวทางการทดสอบ Signal

| สิ่งที่ต้องทดสอบ | วิธีทดสอบ |
|---|---|
| Receiver สร้าง object ใหม่ถูกต้องเมื่อ sender ถูกสร้าง | trigger event (เช่น `User.objects.create_user()`) แล้ว query object ที่คาดว่าจะถูกสร้าง |
| Receiver ไม่ทำงานซ้ำตอน update (`if created:`) | เรียก `.save()` object เดิมหลายครั้ง แล้วนับจำนวน object ที่ถูกสร้างต้องคงที่ |
| Receiver ทำงานจากทุกเส้นทางที่ trigger จริง (ทบทวน Part 019 ขั้นตอนที่ 183.6) | ทดสอบทั้ง `create_user()`, `create_superuser()`, และถ้าเป็นไปได้ผ่าน view/form จริงด้วย |
| พฤติกรรมเมื่อ signal ถูกปิดชั่วคราว | `disconnect()`/`connect()` คู่กันเสมอ ผ่าน `try/finally` หรือ `self.addCleanup()` |
| Custom signal ของตัวเอง (Part 019 ขั้นตอนที่ 187) | ใช้ `unittest.mock.Mock()` เป็น receiver ปลอม เชื่อมชั่วคราว แล้วเช็คว่าถูกเรียกด้วย `assert_called_once_with(...)` (เจาะลึกเทคนิค mock เต็มรูปแบบใน Part 062) |

---

## ขั้นตอนที่ 600: สรุปและแบบฝึกหัด

### 600.1 สรุปสิ่งที่ได้เรียนรู้ใน Part นี้

- ✅ ทดสอบ Model เจาะลึกด้วย `full_clean()` เพื่อ trigger validator ที่ Django
  ไม่เรียกให้อัตโนมัติเมื่อ `.save()` ตรง ๆ
- ✅ ทดสอบ custom method ของ Model (`excerpt()`, `reading_time_minutes()`,
  `is_recent()`, `get_absolute_url()`) ในฐานะ business logic ที่สำคัญที่สุด
- ✅ ทดสอบ `Meta.ordering` ด้วย `assertQuerySetEqual` อย่างถูกวิธี (ไม่พึ่งลำดับ
  ที่ฐานข้อมูลคืนมาโดยบังเอิญ)
- ✅ ทดสอบ View ผ่าน `self.client.get()`/`post()` ครบทั้ง status code,
  `response.context`, และผลข้างเคียงในฐานข้อมูล
- ✅ เข้าใจความต่างและเลือกใช้ `client.login()` vs `client.force_login()` ได้
  ถูกสถานการณ์
- ✅ ทดสอบ Form แยกจาก View เพื่อความเร็วและความชัดเจน ครอบคลุมทั้ง
  `is_valid()`, `form.errors`, custom `clean_<field>()`, และการไหลของ Model
  validator ผ่าน `_post_clean()`
- ✅ ทดสอบ `commit=False` และ `save_m2m()` เพื่อป้องกันบั๊กข้อมูลหายเงียบ ๆ
- ✅ ใช้ `RequestFactory` ทดสอบ CBV แบบ unit-level แยกจาก URL routing และ
  middleware ได้ เมื่อจำเป็น
- ✅ ทดสอบการ render template ด้วย `assertTemplateUsed`, `assertContains`,
  `assertNotContains` รวมถึงกรณี pagination
- ✅ ทดสอบ redirect อย่างเจาะลึกด้วย `assertRedirects` ครบทุกพารามิเตอร์ และ
  ใช้ `follow=True`/`redirect_chain` เมื่อเหมาะสม
- ✅ ทดสอบการปรับแต่ง Django Admin ทั้ง `list_display` และ custom action
  พร้อม permission ของผู้ใช้แต่ละประเภท
- ✅ ทดสอบ signal auto-create `Profile` จาก Part 019 ครบทุกเส้นทาง รวมถึง
  เทคนิค disconnect/connect signal ชั่วคราวอย่างปลอดภัยด้วย `addCleanup()`

### 600.2 Checklist ก่อนไป Part ถัดไป

- [ ] แอป `blog` มีโครงสร้าง `blog/tests/` เป็น package (`__init__.py`,
      `test_models.py`, `test_views.py`, `test_forms.py`, `test_admin.py`)
- [ ] แอป `accounts` มี `accounts/tests/test_signals.py`
- [ ] รัน `python manage.py test blog accounts` แล้วเทสต์ทั้งหมด**ผ่านทุกตัว**
- [ ] เขียนเทสต์ที่ยืนยันว่า `full_clean()` trigger validator ทั้งจาก Model
      field และ custom validator function ได้ครบ
- [ ] เขียนเทสต์ที่แยก `client.login()` และ `client.force_login()` และอธิบาย
      ได้ว่าทำไมเลือกใช้แบบไหนในแต่ละเทสต์
- [ ] เขียนเทสต์ `PostForm`/`CommentForm` ที่ไม่ต้องผ่าน `self.client` เลย
- [ ] เขียนเทสต์ที่ใช้ `RequestFactory` อย่างน้อย 1 เทสต์ และอธิบายความต่างจาก
      การเทสต์ผ่าน `self.client` ได้
- [ ] เขียนเทสต์ `assertRedirects` ที่ตรวจ URL ปลายทางแบบเจาะจง ไม่ใช่แค่เช็ค
      `status_code == 302`
- [ ] เขียนเทสต์ custom action ของ Django Admin อย่างน้อย 1 action
- [ ] เขียนเทสต์ signal ที่ครอบคลุมทั้งกรณีสร้างใหม่และกรณีอัปเดต

### 600.3 แบบฝึกหัดท้ายบท

**แบบฝึกหัดที่ 1**: เพิ่ม custom method ใหม่ให้ `Category` model ชื่อ
`published_post_count()` ที่คืนจำนวน `Post` ที่ `is_published=True` ในหมวดหมู่
นั้น จากนั้นเขียนเทสต์อย่างน้อย 3 เคส: หมวดหมู่ที่ไม่มีโพสต์เลย, หมวดหมู่ที่มี
ทั้งโพสต์เผยแพร่แล้วและฉบับร่างปนกัน (ต้องนับเฉพาะที่เผยแพร่แล้ว), และหมวดหมู่ที่
โพสต์ทั้งหมดเผยแพร่แล้ว

**แบบฝึกหัดที่ 2**: เพิ่ม view ใหม่ชื่อ `post_edit_view` (function-based, ครอบ
ด้วย `@login_required`) ที่อนุญาตให้**เฉพาะ author ของโพสต์นั้นเท่านั้น**แก้ไขได้
(ถ้า user คนอื่นพยายามเข้าให้ตอบกลับ `403 Forbidden` ด้วย
`django.core.exceptions.PermissionDenied`) จากนั้นเขียนเทสต์ครบ 4 เคส:
author เข้าถึงได้ปกติ, user คนอื่นได้ `403`, anonymous ถูก redirect ไป login,
และการแก้ไขที่ถูกต้องบันทึกข้อมูลจริงพร้อม redirect ไปหน้า detail

**แบบฝึกหัดที่ 3**: เพิ่ม custom action ใหม่ให้ `PostAdmin` ชื่อ
`duplicate_posts` ที่ทำสำเนา `Post` ที่เลือกไว้ (สร้าง object ใหม่โดยเปลี่ยน
`slug` เป็น `<slug เดิม>-copy` และตั้ง `is_published=False` เสมอ ไม่ว่าต้นฉบับจะ
เผยแพร่อยู่หรือไม่) เขียนเทสต์ยืนยันว่า: จำนวน `Post` ในฐานข้อมูลเพิ่มขึ้นตาม
จำนวนที่เลือก, สำเนาใหม่มี `is_published=False` เสมอ, และต้นฉบับไม่ถูกแก้ไขเลย

**แบบฝึกหัดที่ 4 (ขั้นสูง)**: เพิ่ม signal ใหม่ในแอป `blog` ที่ทำงานตอน
`post_delete` ของ `Post` เพื่อลบ `Comment` ที่กำพร้าออกอย่างชัดเจน (แม้
`on_delete=models.CASCADE` จะลบให้อัตโนมัติอยู่แล้วก็ตาม ให้เพิ่ม log ผ่าน
Python `logging` module แทน) จากนั้นเขียนเทสต์โดยใช้
`self.assertLogs(logger_name, level="INFO")` (ทบทวนได้จากเอกสาร Python
`unittest` — เป็น context manager ที่จับข้อความ log ระหว่างเทสต์) เพื่อยืนยันว่า
log ถูกเขียนออกมาจริงทุกครั้งที่ `Post` ที่มีคอมเมนต์อยู่ถูกลบ

### 600.4 คำถามที่พบบ่อย (FAQ)

**Q: ทำไมเทสต์ Model ต้องเรียก `full_clean()` เอง ทั้งที่ Django Admin เรียกให้
อัตโนมัติ?**
A: เพราะ `full_clean()` ถูกออกแบบมาให้ทำงานเฉพาะจุดที่ **มีการรับข้อมูลจาก
ภายนอกเข้ามาจริง ๆ** เช่น ผ่าน `ModelForm` (ที่ Admin ก็ใช้เบื้องหลังตามที่เรียน
ใน Part 026 ขั้นตอนที่ 254.3) ส่วนการเรียก `Model.objects.create()` หรือ
`instance.save()` ตรง ๆ ถือว่าเป็นการเขียนโค้ด Python ปกติที่นักพัฒนาควรรู้อยู่
แล้วว่าข้อมูลถูกต้อง (เช่น data migration หรือ script นำเข้าข้อมูลจำนวนมาก) การ
บังคับ validate ทุกครั้งจะทำให้ script เหล่านี้ช้าลงโดยไม่จำเป็น — เมื่อเขียนเทสต์
Model ตรง ๆ (ไม่ผ่าน Form) คุณจึงต้องเรียก `full_clean()` เองเสมอถ้าต้องการ
ทดสอบ validator

**Q: ควรเทสต์ view ผ่าน `self.client` หรือ `RequestFactory` เป็นหลัก?**
A: ใช้ `self.client` เป็นค่าเริ่มต้นสำหรับเทสต์ view ส่วนใหญ่ เพราะครอบคลุม
พฤติกรรมทั้งระบบ (URL routing, middleware, template rendering) ใกล้เคียงกับที่
ผู้ใช้จริงเจอที่สุด ใช้ `RequestFactory` เฉพาะเมื่อต้องแยกทดสอบ method เดียวของ
CBV ที่ซับซ้อนมาก หรือเมื่อต้องการความเร็วสูงสุดในการรันเทสต์จำนวนมาก (เจาะลึก
เรื่อง test performance เต็มรูปแบบใน Part 065)

**Q: ทำไมต้องแยกเทสต์ Form ออกจากเทสต์ View ทั้งที่ทดสอบผ่าน View ก็ครอบคลุม
Form ไปด้วยอยู่แล้ว?**
A: เพราะเมื่อเทสต์ผ่าน View แล้วพัง คุณจะไม่รู้ทันทีว่า **บั๊กอยู่ที่ Form
(validation ผิด) หรือ View (logic การเรียกใช้ Form ผิด)** การแยกเทสต์ Form
ออกมาต่างหากทำให้ pinpoint ปัญหาได้เร็วกว่ามาก และยังรันเร็วกว่าด้วย (ไม่ต้อง
ผ่าน URL dispatcher และ template rendering) หลักการนี้คือ **Test Pyramid** ที่
เน้นให้มี unit test (เทสต์ส่วนเล็ก ๆ แยกกัน) จำนวนมากกว่า integration test
(เทสต์ผ่านระบบเต็ม) เสมอ

**Q: `assertRedirects` ตามลิงก์ไปจริงทุกครั้ง ทำให้เทสต์ช้าไหม?**
A: ช้ากว่าการเช็คแค่ `status_code == 302` เล็กน้อยจริง (เพราะยิง request เพิ่ม
อีกครั้ง) แต่ในทางปฏิบัติความช้าที่เพิ่มขึ้นนี้ **น้อยมากเมื่อเทียบกับความ
ปลอดภัยที่ได้** (จับบั๊ก redirect ผิด URL ได้ทันที) ถ้าต้องการเทสต์จำนวนมากที่
เน้นความเร็วสูงสุดจริง ๆ ให้ใช้ `fetch_redirect_response=False` แทน แต่ต้องแลก
กับการไม่ยืนยันว่าหน้าปลายทางเข้าถึงได้จริง

### เตรียมตัวสำหรับ Part ถัดไป

**Part 061: pytest-django และ Fixtures** จะพาคุณเปลี่ยนจาก `unittest`/`TestCase`
มาตรฐานของ Django มาสู่ **pytest** ซึ่งเป็นเครื่องมือทดสอบที่นิยมที่สุดในวงการ
Python ปัจจุบัน คุณจะได้เห็นว่าเทสต์ทั้งหมดที่เขียนไว้ใน Part นี้ (`test_models.py`,
`test_views.py`, `test_forms.py`, `test_admin.py`) แปลงมาเป็นรูปแบบ pytest ที่
กระชับกว่าได้อย่างไรด้วย `pytest-django`, เรียนรู้ **Fixtures** ที่มาแทนที่
`setUp()` ในรูปแบบที่ยืดหยุ่นและ reuse ข้ามไฟล์ได้ง่ายกว่ามาก, `parametrize`
สำหรับรันเทสต์เดียวกันซ้ำหลายชุดข้อมูล, และปูทางไปสู่ Part 062 ที่จะเจาะลึก
**Test Coverage** (วัดว่าโค้ดส่วนไหนยังไม่มีเทสต์ครอบคลุม) และ **Mocking**
(จำลอง dependency ภายนอกอย่าง API หรือ email service) เตรียมติดตั้ง
`pip install pytest pytest-django` ไว้ล่วงหน้าได้เลย
