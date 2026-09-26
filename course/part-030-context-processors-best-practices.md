# Part 030: Context Processors และ Template Best Practices

> **ขั้นตอนที่ 291-300 ของหลักสูตร** | Phase 3: Views, Templates, Forms และ CBV (ตอนสุดท้ายของ Phase นี้)
>
> เป้าหมายของ Part นี้: กลับไปเจาะลึก **Context Processors** ที่แนะนำแบบเบื้องต้นไปแล้ว
> ใน Part 008 ให้ลึกถึงระดับ production — หลาย context processor ทำงานร่วมกันอย่างไร,
> การ query ฐานข้อมูลใน context processor พร้อมข้อควรระวังเรื่อง performance,
> ลำดับความสำคัญเมื่อชื่อตัวแปรชนกัน, และรู้จัก **Jinja2** เป็นทางเลือกแทน Django
> Template Language จากนั้นปิดท้ายด้วย **Template Best Practices** ระดับมืออาชีพ:
> การ debug template, การเขียนเทสต์ให้ template, ความรู้พื้นฐานด้าน
> **Internationalization (i18n)** และ **Accessibility (a11y)** เมื่อจบ Part นี้จะเป็นการ
> **ปิดฉาก Phase 3 ทั้งหมด** ด้วยการทบทวนทุกหัวข้อตั้งแต่ Part 021-030, ทำ Quiz ทบทวน,
> และลงมือสร้างระบบบล็อกที่มี CRUD ครบวงจรผ่าน CBV พร้อมฟอร์มคอมเมนต์แบบ ModelForm
> และ custom template tag ของตัวเอง ก่อนเดินหน้าสู่ Phase 4: Authentication

---

## สารบัญของ Part นี้

- ขั้นตอนที่ 291: ทบทวน Custom Context Processor จาก Part 008 แล้วเจาะลึกกว่า — หลาย Context Processor พร้อมกัน, ลำดับการทำงาน
- ขั้นตอนที่ 292: Context Processor ที่ Query ฐานข้อมูล — หมวดหมู่สำหรับ Sidebar ทุกหน้า พร้อมคำเตือนเรื่อง Performance และแนวทาง Cache
- ขั้นตอนที่ 293: ลำดับความสำคัญของ `context_processors` ใน settings.py และผลกระทบเมื่อชื่อตัวแปรชนกัน
- ขั้นตอนที่ 294: ภาพรวม Jinja2 เป็น Alternative Template Backend ใน Django
- ขั้นตอนที่ 295: Template Best Practices Checklist ระดับมืออาชีพ
- ขั้นตอนที่ 296: การ Debug Template — `{% debug %}`, Django Debug Toolbar's Template Panel, ข้อความ Error ที่พบบ่อย
- ขั้นตอนที่ 297: การเขียน Test สำหรับ Template — `assertTemplateUsed`, `assertContains`, การ Render แยกทดสอบ
- ขั้นตอนที่ 298: เกริ่น Internationalization (i18n) ใน Template
- ขั้นตอนที่ 299: Accessibility (a11y) ใน Django Template
- ขั้นตอนที่ 300: สรุป Phase 3 ทั้งหมด (Part 021-030), Quiz, แบบฝึกหัดใหญ่ปิดท้าย Phase และคำนำสู่ Phase 4

---

## ขั้นตอนที่ 291: ทบทวน Custom Context Processor จาก Part 008 แล้วเจาะลึกกว่า

### 291.1 ทบทวนโครงสร้างโปรเจกต์บล็อก ณ จุดนี้

ก่อนเจาะลึก มาทบทวนโครงสร้าง Model ของแอป `blog` ที่สะสมมาตลอด Phase 2-3 ให้เป็นเวอร์ชัน
เดียวกันที่จะใช้อ้างอิงตลอดทั้ง Part นี้:

```python
# blog/models.py
from django.conf import settings
from django.db import models
from django.urls import reverse
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
        Category, on_delete=models.SET_NULL, null=True, blank=True,
        related_name='posts',
    )
    author = models.ForeignKey(
        settings.AUTH_USER_MODEL, on_delete=models.CASCADE, related_name='posts',
    )
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)
    is_published = models.BooleanField(default=False)

    class Meta:
        ordering = ['-created_at']

    def __str__(self):
        return self.title

    def get_absolute_url(self):
        return reverse('blog:detail', kwargs={'slug': self.slug})

    def save(self, *args, **kwargs):
        if not self.slug:
            self.slug = slugify(self.title)
        super().save(*args, **kwargs)


class Comment(models.Model):
    post = models.ForeignKey(Post, on_delete=models.CASCADE, related_name='comments')
    author_name = models.CharField(max_length=100)
    text = models.TextField()
    created_at = models.DateTimeField(auto_now_add=True)

    class Meta:
        ordering = ['-created_at']

    def __str__(self):
        return f"ความคิดเห็นโดย {self.author_name} บน {self.post.title}"
```

โครงนี้รวมทุกอย่างที่เรียนมา: `Category` และความสัมพันธ์แบบ FK จาก Phase 2,
`Post.author` ที่ผูกกับ `User` (เกริ่นไว้ก่อนเจาะลึกเต็มใน Phase 4), และ `Comment`
ที่จะกลับมาใช้ทำ ModelForm ในขั้นตอนที่ 300

### 291.2 ทบทวน Context Processor: ฟังก์ชัน Python ธรรมดาที่คืนค่า Dict

ย้อนกลับไปที่ Part 008 ขั้นตอนที่ 79: **Context Processor คือฟังก์ชันที่รับ `request`
เป็น argument แล้วคืนค่าเป็น `dict`** ที่จะถูกเติมเข้า context ของ**ทุก**เทมเพลตที่
render ผ่าน engine นั้นโดยอัตโนมัติ ไม่มีอะไรพิเศษไปกว่านี้ — มันคือฟังก์ชัน Python
ปกติทุกประการ ไม่มี base class ให้ inherit ไม่มี decorator บังคับ:

```python
# blog/context_processors.py (ทบทวนจาก Part 008)
from .models import Post


def site_metadata(request):
    return {
        'site_name': 'Django Mastery Blog',
        'published_post_count': Post.objects.filter(is_published=True).count(),
    }
```

สิ่งที่ Part 008 ยังไม่ได้พูดถึงคือ: ในโปรเจกต์จริงระดับมืออาชีพ แทบไม่มีโปรเจกต์ไหน
มี custom context processor แค่ตัวเดียว — ส่วนใหญ่จะมี **หลายตัว** ทำงานร่วมกับ
context processor ของ Django เองอีก 4 ตัว (`debug`, `request`, `auth`, `messages`)
รวมเป็นทั้งหมด 6-10 ตัวในโปรเจกต์ขนาดกลางถึงใหญ่ ขั้นตอนนี้จะพาไปดูว่าเมื่อมีหลายตัว
พร้อมกัน Django จัดการอย่างไร

### 291.3 เพิ่ม Context Processor ตัวที่สอง: เมนู Navigation

สมมติเราต้องการให้เมนู navbar กำหนดจากที่เดียว (ไม่ hardcode ใน `navbar.html`
โดยตรง) เพื่อให้แก้เมนูได้จากจุดเดียวในอนาคต (เช่น เมื่อทำเป็นระบบจัดการเมนูผ่าน Admin
ในภายหลัง) เขียน context processor ตัวที่สองเพิ่มเข้าไปในไฟล์เดิม:

```python
# blog/context_processors.py
from .models import Post


def site_metadata(request):
    return {
        'site_name': 'Django Mastery Blog',
        'published_post_count': Post.objects.filter(is_published=True).count(),
    }


def navigation_menu(request):
    """เมนู navbar แบบ static — ยังไม่ query ฐานข้อมูล เพื่อเปรียบเทียบกับ
    ขั้นตอนที่ 292 ที่จะเพิ่มเวอร์ชันที่ query ฐานข้อมูลจริง"""
    return {
        'nav_links': [
            {'label': 'หน้าแรก', 'url_name': 'blog:list'},
            {'label': 'เกี่ยวกับเรา', 'url_name': 'blog:about'},
            {'label': 'ติดต่อเรา', 'url_name': 'blog:contact'},
        ],
    }
```

ลงทะเบียนทั้งสองตัวใน `TEMPLATES.OPTIONS.context_processors` **ตามลำดับที่ต้องการ**:

```python
# config/settings.py
TEMPLATES = [
    {
        'BACKEND': 'django.template.backends.django.DjangoTemplates',
        'DIRS': [BASE_DIR / 'templates'],
        'APP_DIRS': True,
        'OPTIONS': {
            'context_processors': [
                'django.template.context_processors.debug',
                'django.template.context_processors.request',
                'django.contrib.auth.context_processors.auth',
                'django.contrib.messages.context_processors.messages',
                'blog.context_processors.site_metadata',
                'blog.context_processors.navigation_menu',
            ],
        },
    },
]
```

ใช้ใน `navbar.html` ด้วย `{% for %}`:

```html
<!-- templates/partials/navbar.html -->
<nav class="navbar">
    <a href="{% url 'blog:list' %}" class="navbar__brand">{{ site_name }}</a>
    <ul class="navbar__menu">
        {% for link in nav_links %}
            <li><a href="{% url link.url_name %}">{{ link.label }}</a></li>
        {% endfor %}
    </ul>
</nav>
```

### 291.4 ลำดับการทำงานเบื้องหลังเมื่อมีหลาย Context Processor

เมื่อ Django เรียก `render()` (หรือทุกครั้งที่ template ถูก render ผ่าน
`RequestContext`) มันจะ**ไล่เรียกทุก context processor ในลิสต์ตามลำดับที่ประกาศไว้
ทีละตัว** แล้วนำ dict ที่แต่ละตัวคืนมา **update เข้าด้วยกันเป็น dict เดียว** พฤติกรรม
นี้เทียบเท่ากับโค้ด Python ง่าย ๆ นี้:

```python
# แนวคิดเบื้องหลัง (ไม่ใช่โค้ดจริงของ Django แต่พฤติกรรมเหมือนกันทุกประการ)
merged_context = {}
for processor in context_processors_list:
    merged_context.update(processor(request))
```

ประเด็นสำคัญ 2 ข้อจากโค้ดจำลองนี้:

1. **processor ทำงานเรียงตามลำดับในลิสต์เสมอ** — จากบนลงล่าง
2. **ถ้า processor สองตัวคืนค่า key ชื่อเดียวกัน ตัวที่อยู่ "หลังกว่า" ในลิสต์จะ
   ชนะเสมอ** เพราะ `dict.update()` เขียนทับค่าเดิม — นี่คือหัวข้อที่จะเจาะลึกเต็ม ๆ
   ในขั้นตอนที่ 293

### 291.5 ทดสอบยืนยันว่า Context Processor ทำงานตามลำดับจริง

เขียนเทสต์เล็ก ๆ เพื่อพิสูจน์พฤติกรรมนี้ด้วยตัวเอง (ไม่ต้องรอถึง Part เรื่อง Testing
เต็มรูปแบบใน Phase 7 — หลักการพื้นฐานเรียนได้ตั้งแต่ตอนนี้):

```python
# blog/tests/test_context_processors.py
from django.test import RequestFactory, TestCase

from blog.context_processors import navigation_menu, site_metadata


class ContextProcessorTests(TestCase):
    def setUp(self):
        self.factory = RequestFactory()
        self.request = self.factory.get('/')

    def test_site_metadata_returns_expected_keys(self):
        context = site_metadata(self.request)
        self.assertIn('site_name', context)
        self.assertIn('published_post_count', context)
        self.assertEqual(context['site_name'], 'Django Mastery Blog')

    def test_navigation_menu_returns_three_links(self):
        context = navigation_menu(self.request)
        self.assertEqual(len(context['nav_links']), 3)
        self.assertEqual(context['nav_links'][0]['url_name'], 'blog:list')
```

สังเกตว่าเราเรียก context processor **ตรง ๆ เหมือนฟังก์ชัน Python ทั่วไป** โดยไม่ต้อง
ผ่านระบบ template หรือ view ใด ๆ เลย นี่คือข้อดีของการที่ context processor เป็นแค่
ฟังก์ชันธรรมดา — มัน**เทสต์ง่ายมาก**เมื่อเทียบกับกลไก Django อื่น ๆ ที่ผูกกับ HTTP
cycle แน่นกว่า

### 291.6 คำแนะนำ: จัดกลุ่ม Context Processor ตามความรับผิดชอบ

เมื่อโปรเจกต์โตขึ้นและมี context processor หลายตัว ทีมมืออาชีพนิยมแยกไฟล์ตามหน้าที่
แทนที่จะยัดทุกอย่างในไฟล์เดียว เช่น:

```
blog/
└── context_processors.py       # metadata, navigation ของแอป blog
accounts/
└── context_processors.py       # ข้อมูลเกี่ยวกับผู้ใช้ที่ล็อกอิน (Phase 4)
config/
└── context_processors.py       # ค่าระดับ "ทั้งเว็บไซต์" ที่ไม่ผูกกับแอปไหน
    (เช่น SITE_URL, GOOGLE_ANALYTICS_ID จาก settings)
```

```python
# config/context_processors.py
from django.conf import settings


def site_settings(request):
    """ค่า config ที่ template อาจต้องใช้ (อ่านจาก settings.py โดยตรง)"""
    return {
        'GOOGLE_ANALYTICS_ID': getattr(settings, 'GOOGLE_ANALYTICS_ID', ''),
        'SITE_URL': getattr(settings, 'SITE_URL', 'http://localhost:8000'),
    }
```

การแยกไฟล์แบบนี้ทำให้เห็นภาพรวมง่ายขึ้นว่า context processor ไหนเป็นของแอปไหน
เมื่อต้อง debug ปัญหาเรื่องตัวแปรชนกัน (ขั้นตอนที่ 293) หรือปัญหาเรื่อง performance
(ขั้นตอนที่ 292)

---

## ขั้นตอนที่ 292: Context Processor ที่ Query ฐานข้อมูล — หมวดหมู่สำหรับ Sidebar ทุกหน้า

### 292.1 โจทย์: อยากให้รายการหมวดหมู่ปรากฏใน Sidebar ทุกหน้าโดยไม่ต้องส่งจาก View ทุกตัว

บล็อกของเรามี `Category` อยู่แล้ว และอยากให้ **ทุกหน้า** (ไม่ใช่แค่หน้า list) แสดง
sidebar รายการหมวดหมู่พร้อมจำนวนบทความในแต่ละหมวด ถ้าไม่ใช้ context processor
เราจะต้องเขียนโค้ดแบบนี้ซ้ำใน**ทุก View** ของทุกแอป:

```python
# ❌ ต้องเขียนซ้ำในทุก View ถ้าไม่ใช้ context processor
class PostListView(ListView):
    def get_context_data(self, **kwargs):
        context = super().get_context_data(**kwargs)
        context['sidebar_categories'] = Category.objects.annotate(
            post_count=Count('posts')
        ).order_by('name')
        return context


class PostDetailView(DetailView):
    def get_context_data(self, **kwargs):
        context = super().get_context_data(**kwargs)
        context['sidebar_categories'] = Category.objects.annotate(   # ← ซ้ำ!
            post_count=Count('posts')
        ).order_by('name')
        return context

# ... ต้องซ้ำแบบนี้ในทุก View ที่ต้องแสดง sidebar
```

นี่คือสถานการณ์ที่ context processor ถูกออกแบบมาให้แก้โดยเฉพาะ — logic ที่ต้องปรากฏ
**ทุกหน้าโดยไม่มีข้อยกเว้น** ควรอยู่ใน context processor ไม่ใช่กระจายอยู่ในทุก View

### 292.2 เขียน `categories_for_sidebar` Context Processor

```python
# blog/context_processors.py
from django.db.models import Count, Q

from .models import Category, Post


def site_metadata(request):
    return {
        'site_name': 'Django Mastery Blog',
        'published_post_count': Post.objects.filter(is_published=True).count(),
    }


def categories_for_sidebar(request):
    categories = (
        Category.objects
        .annotate(
            post_count=Count('posts', filter=Q(posts__is_published=True))
        )
        .filter(post_count__gt=0)
        .order_by('name')
    )
    return {'sidebar_categories': categories}
```

ลงทะเบียนเพิ่มเข้า `context_processors`:

```python
# config/settings.py (ส่วนที่เกี่ยวข้อง)
'context_processors': [
    'django.template.context_processors.debug',
    'django.template.context_processors.request',
    'django.contrib.auth.context_processors.auth',
    'django.contrib.messages.context_processors.messages',
    'blog.context_processors.site_metadata',
    'blog.context_processors.navigation_menu',
    'blog.context_processors.categories_for_sidebar',
],
```

ใช้งานใน partial ใหม่ `sidebar.html`:

```html
<!-- templates/partials/sidebar.html -->
<aside class="sidebar">
    <h3>หมวดหมู่</h3>
    <ul class="sidebar__categories">
        {% for category in sidebar_categories %}
            <li>
                <a href="{% url 'blog:category-detail' slug=category.slug %}">
                    {{ category.name }} ({{ category.post_count }})
                </a>
            </li>
        {% empty %}
            <li>ยังไม่มีหมวดหมู่ที่มีบทความ</li>
        {% endfor %}
    </ul>
</aside>
```

### 292.3 คำเตือนสำคัญที่สุดของขั้นตอนนี้: Context Processor ทำงาน "ทุก Request"

ย้ำสิ่งที่ Part 008 ขั้นตอนที่ 79.5 เกริ่นไว้ให้ชัดเจนยิ่งขึ้น: **`categories_for_sidebar`
จะถูกเรียก query ฐานข้อมูลใหม่ทุกครั้ง ที่มีการ render template ใด ๆ ก็ตามที่ใช้ engine
ตัวนี้** ไม่ว่าหน้านั้นจะแสดง sidebar หรือไม่ก็ตาม!

| สถานการณ์ | Context Processor ถูกเรียกไหม | ผลกระทบ |
|---|---|---|
| หน้า `PostListView` (มี sidebar) | ✅ เรียก | จำเป็น ใช้ผลลัพธ์จริง |
| หน้า `PostDetailView` (มี sidebar) | ✅ เรียก | จำเป็น ใช้ผลลัพธ์จริง |
| หน้า `AboutView` แบบ `TemplateView` ธรรมดา (ไม่มี sidebar) | ✅ เรียก (สิ้นเปลือง!) | Query ฐานข้อมูลทิ้งเปล่า ๆ เพราะ template ไม่ได้ใช้ `sidebar_categories` เลย |
| View ที่คืน `JsonResponse` โดยตรง (ไม่เรียก `render()`) | ❌ ไม่เรียก | เพราะ context processor ทำงานเฉพาะตอน `render()`/`RequestContext` เท่านั้น API endpoint ที่ไม่ผ่าน template จึงไม่ได้รับผลกระทบ |

ข้อสังเกตที่สำคัญ: **ถ้าเว็บไซต์มี 50 หน้า และมีแค่ 5 หน้าที่ต้องการ sidebar
หมวดหมู่ แต่เขียนเป็น context processor ทั้ง 50 หน้าจะโดน query นี้ทั้งหมด** นี่คือ
เหตุผลที่ต้องชั่งน้ำหนักเสมอว่า **สิ่งที่จะใส่ใน context processor ต้องจำเป็นสำหรับ
"เกือบทุกหน้าจริง ๆ"** ถ้าจำเป็นแค่บางหน้า ควรใส่ผ่าน `get_context_data()` ของ CBV
เฉพาะหน้านั้น (Part 021-024) หรือใช้ **inclusion tag** จาก Part 029 แทน (ดูตาราง
เปรียบเทียบในขั้อ 292.5)

### 292.4 แนวทางบรรเทาปัญหาด้วย Caching (เกริ่น — เจาะลึกเต็มใน Part 068)

เพราะรายการหมวดหมู่เปลี่ยนแปลงไม่บ่อย (ไม่เหมือน `published_post_count` ที่เปลี่ยน
ทุกครั้งที่มีบทความใหม่) จึงเป็นตัวเลือกที่ดีมากสำหรับการทำ **caching** — เก็บผลลัพธ์
ไว้ใน memory ช่วงเวลาหนึ่งแทนที่จะ query ฐานข้อมูลทุก request:

```python
# blog/context_processors.py
from django.core.cache import cache
from django.db.models import Count, Q

from .models import Category

CACHE_KEY_SIDEBAR_CATEGORIES = 'blog:sidebar_categories'
CACHE_TIMEOUT_SECONDS = 300   # 5 นาที


def categories_for_sidebar(request):
    def _fetch_categories():
        return list(
            Category.objects
            .annotate(post_count=Count('posts', filter=Q(posts__is_published=True)))
            .filter(post_count__gt=0)
            .order_by('name')
        )

    categories = cache.get_or_set(
        CACHE_KEY_SIDEBAR_CATEGORIES, _fetch_categories, timeout=CACHE_TIMEOUT_SECONDS,
    )
    return {'sidebar_categories': categories}
```

`cache.get_or_set(key, default, timeout)` คือ helper ของ Django's Caching
Framework: ถ้ามีค่าอยู่ใน cache แล้ว (ยังไม่หมดอายุ) จะคืนค่านั้นทันทีโดย**ไม่แตะ
ฐานข้อมูลเลย** ถ้ายังไม่มีหรือหมดอายุแล้ว จะเรียกฟังก์ชัน `_fetch_categories()`
มา query ฐานข้อมูลครั้งเดียว แล้วเก็บผลลัพธ์ไว้ใน cache ให้ request ถัดไปใช้ต่อ
ภายใน `timeout` วินาทีที่กำหนด

> **สำคัญ**: นี่เป็นเพียงการเกริ่นให้เห็นแนวทางเท่านั้น รายละเอียดเชิงลึกทั้งหมด
> — การเลือก cache backend (LocMemCache vs Redis vs Memcached), กลยุทธ์การ
> invalidate cache เมื่อมีการเพิ่ม/ลบ `Category` (ผ่าน Signal จาก Part 019),
> per-view caching, template fragment caching, และ cache versioning — จะถูก
> เจาะลึกแบบเต็มรูปแบบใน **Part 068: Django Caching Framework เบื้องต้น** และ
> **Part 069: Redis Caching ขั้นสูง** ตอนนี้ขอให้คุณเข้าใจแค่หลักการว่า **"ข้อมูลที่
> เปลี่ยนไม่บ่อยแต่ถูกอ่านบ่อยมาก คือผู้สมัครอันดับหนึ่งสำหรับ caching"**

### 292.5 ทางเลือก: Context Processor vs Inclusion Tag (ตารางตัดสินใจ)

Part 029 สอนการเขียน `inclusion_tag` ที่ query ข้อมูลเองได้เหมือนกัน คำถามที่ตามมา
คือ "แล้วเมื่อไหร่ควรใช้ context processor เมื่อไหร่ควรใช้ inclusion tag?"

| ประเด็น | Context Processor | Inclusion Tag |
|---|---|---|
| ทำงานเมื่อไหร่ | **ทุกครั้ง** ที่ render template ผ่าน engine นั้น (ควบคุมไม่ได้เป็นรายหน้า) | เฉพาะหน้าที่มี `{% load %}` และเรียก tag นั้นจริง ๆ (ควบคุมได้ชัดเจน) |
| เหมาะกับข้อมูลที่ | ต้องปรากฏ **เกือบทุกหน้าจริง ๆ** (เช่น เมนู, ชื่อเว็บไซต์) | ปรากฏเฉพาะบางหน้า/บาง section (เช่น sidebar ที่โชว์แค่หน้า blog ไม่โชว์หน้า checkout) |
| ผลกระทบต่อ Performance เมื่อไม่ได้ใช้ | เสียของเงียบ ๆ ทุกหน้าที่ไม่ได้ใช้ก็ยัง query | ไม่เสียเลย เพราะไม่ถูกเรียกถ้าไม่ใส่ tag ในหน้านั้น |
| ความชัดเจนของ Template | ไม่ชัดเจนว่า `sidebar_categories` มาจากไหน (ต้องไปดู settings.py) | ชัดเจนในตัวมันเอง — เห็น `{% render_sidebar_categories %}` ก็รู้ทันทีว่ามาจากไหน |

**คำแนะนำระดับมืออาชีพ**: ถ้า sidebar หมวดหมู่นี้จะปรากฏแค่ในโซน "บล็อก" ของเว็บไซต์
(ไม่ใช่ทุกหน้าของทั้งระบบ เช่น ไม่ต้องโชว์ในหน้า checkout ของระบบ e-commerce ที่อาจ
รวมอยู่ในโปรเจกต์เดียวกัน) **inclusion tag มักเป็นตัวเลือกที่ดีกว่า context
processor** เพราะชัดเจนกว่าและไม่เสีย performance กับหน้าที่ไม่เกี่ยวข้อง
context processor ควรสงวนไว้สำหรับข้อมูลที่ "เป็นของทั้งเว็บไซต์จริง ๆ" เท่านั้น
เช่น `site_name`, ข้อมูล user ที่ล็อกอิน (ซึ่ง Django ทำให้อยู่แล้วผ่าน `auth`
context processor)

---

## ขั้นตอนที่ 293: ลำดับความสำคัญของ `context_processors` และผลกระทบเมื่อชื่อตัวแปรชนกัน

### 293.1 ทบทวน: Dict Update ทำงานตามลำดับในลิสต์

จากขั้นตอนที่ 291.4 เรารู้แล้วว่า context processor ทำงานเรียงตามลำดับ และตัวที่มา
**ทีหลัง** ในลิสต์จะเขียนทับตัวที่มาก่อนถ้าคืนค่า key ชื่อเดียวกัน มาดูตัวอย่างที่ทำให้
เกิดบั๊กจริงจากความชนกันนี้

### 293.2 ตัวอย่างบั๊กจริง: สองแอปตั้งชื่อ Key ชนกัน

สมมติแอป `blog` มี context processor นี้:

```python
# blog/context_processors.py
def site_metadata(request):
    return {
        'site_name': 'Django Mastery Blog',
        'items': ['บทความ 1', 'บทความ 2', 'บทความ 3'],   # ตั้งใจไว้ใช้เป็น "latest posts"
    }
```

และแอป `shop` (สมมติว่าอนาคตโปรเจกต์นี้ขยายมาขายของด้วย) มี context processor
ของตัวเองที่ไม่รู้เรื่องแอป `blog` เลย:

```python
# shop/context_processors.py
def cart_summary(request):
    return {
        'items': request.session.get('cart_items', []),   # รายการสินค้าในตะกร้า
        'cart_total': request.session.get('cart_total', 0),
    }
```

ถ้าลงทะเบียนทั้งสองตัวในลิสต์ (ไม่ว่าลำดับไหน) **ตัวหลังจะเขียนทับตัวแรกเสมอสำหรับ
key `items`**:

```python
'context_processors': [
    ...
    'blog.context_processors.site_metadata',   # ← คืน items = รายการบทความ
    'shop.context_processors.cart_summary',    # ← คืน items = รายการสินค้าในตะกร้า (ชนะ!)
],
```

ผลลัพธ์: ทุกที่ใน template ที่เขียน `{{ items }}` โดยหวังจะได้ "รายการบทความ" จาก
`blog` จะได้ "รายการสินค้าในตะกร้า" จาก `shop` แทน — **โดยไม่มี error ใด ๆ เตือนเลย**
เพราะ Python `dict.update()` ไม่ raise exception เมื่อ key ซ้ำ นี่คือบั๊กประเภทที่
อันตรายที่สุด เพราะมันเงียบและมักถูกค้นพบช้ามากในโปรเจกต์จริง

### 293.3 ห้ามตั้งชื่อชนกับ Context Processor ในตัวของ Django เอง

ตารางต่อไปนี้คือชื่อตัวแปรที่ **สงวนไว้แล้ว** โดย context processor ที่ Django
ติดตั้งมาให้ (จาก Part 008 ขั้นตอนที่ 79.3) — ห้ามตั้งชื่อตัวแปรของคุณเองซ้ำกับสิ่ง
เหล่านี้เด็ดขาด:

| ชื่อตัวแปรที่สงวนไว้ | มาจาก Context Processor |
|---|---|
| `debug`, `sql_queries` | `django.template.context_processors.debug` |
| `request` | `django.template.context_processors.request` |
| `user`, `perms` | `django.contrib.auth.context_processors.auth` |
| `messages` | `django.contrib.messages.context_processors.messages` |
| `csrf_token` | มาจากกลไกภายในของ Django Template Engine โดยตรง (ไม่ใช่ context processor แต่ถูกเติมเข้าเสมอ) |
| `LANGUAGES`, `LANGUAGE_CODE`, `LANGUAGE_BIDI` | `django.template.context_processors.i18n` (ขั้นตอนที่ 298) |
| `STATIC_URL`, `MEDIA_URL` | `django.template.context_processors.static`, `django.template.context_processors.media` |

ถ้าคุณเผลอเขียน context processor ของตัวเองที่คืนค่า `{'user': some_custom_dict}`
มันจะเขียนทับ `request.user` object จริงที่ auth context processor เตรียมไว้ให้
(หรือถูกเขียนทับกลับ ขึ้นอยู่กับลำดับ) ทำให้ `{% if user.is_authenticated %}` ทั่ว
ทั้งเว็บไซต์พังทันที — เป็นบั๊กระดับ critical ที่ป้องกันได้ง่าย ๆ แค่ไม่ตั้งชื่อชนกัน

### 293.4 ข้อมูลเชิงลึก: Context จาก View เอง "ชนะ" Context Processor เสมอ

มีกฎสำคัญอีกข้อที่หลายคนไม่รู้: **ถ้า View ส่ง context variable ชื่อเดียวกับที่
context processor คืนค่าไว้ ค่าที่ View ส่งมาโดยตรงจะชนะเสมอ** ไม่ว่า context
processor นั้นจะอยู่ตำแหน่งไหนในลิสต์ก็ตาม เพราะกลไกภายในของ Django ถูกออกแบบมา
โดยเจตนาให้ **"ค่าที่ระบุมาชัดเจนจาก View ต้อง override ค่าอัตโนมัติจาก context
processor เสมอ"**

ทดสอบพิสูจน์ได้ง่าย ๆ:

```python
# blog/context_processors.py
def site_metadata(request):
    return {'site_name': 'Django Mastery Blog'}
```

```python
# blog/views.py
from django.views.generic import TemplateView


class OverrideDemoView(TemplateView):
    template_name = 'blog/override_demo.html'

    def get_context_data(self, **kwargs):
        context = super().get_context_data(**kwargs)
        context['site_name'] = 'ชื่อที่ View กำหนดเอง'   # ← ตั้งใจชนกับ context processor
        return context
```

```html
<!-- blog/templates/blog/override_demo.html -->
<p>{{ site_name }}</p>
```

เมื่อเปิดหน้านี้ ผลลัพธ์ที่ได้คือ **"ชื่อที่ View กำหนดเอง"** เสมอ ไม่ใช่
"Django Mastery Blog" จาก context processor เขียนเทสต์ยืนยันพฤติกรรมนี้ได้ทันที:

```python
# blog/tests/test_context_override.py
from django.test import TestCase
from django.urls import reverse


class ContextOverrideTests(TestCase):
    def test_view_context_wins_over_context_processor(self):
        response = self.client.get(reverse('blog:override-demo'))
        self.assertContains(response, 'ชื่อที่ View กำหนดเอง')
        self.assertNotContains(response, 'Django Mastery Blog')
```

**เหตุผลเชิงออกแบบ**: Context processor มีไว้ให้ค่า "เริ่มต้น"/"ค่าทั่วไป" ที่ใช้ได้
กับเกือบทุกหน้า ส่วน View คือจุดที่รู้ข้อมูล "เฉพาะเจาะจงที่สุด" ของหน้านั้น ๆ
Django จึงออกแบบให้สิ่งที่เจาะจงกว่าชนะเสมอ — เป็นหลักการเดียวกับ CSS specificity
ที่ inline style ชนะ class ชนะ tag selector

### 293.5 แนวทางป้องกันปัญหาชื่อชนกันในทีม

| แนวทาง | รายละเอียด |
|---|---|
| **ตั้ง Prefix ตามแอป** | ใช้ `blog_categories` แทน `categories` เฉย ๆ, `shop_cart_items` แทน `items` เฉย ๆ — ลดโอกาสชนข้ามแอปได้มาก |
| **เอกสารกลาง 1 หน้า** | เก็บรายชื่อ context processor ทั้งหมดของโปรเจกต์ พร้อม key ที่แต่ละตัวคืนค่า ไว้ในเอกสารเดียว (เช่น `docs/context_processors.md`) ให้ทุกคนในทีมเช็คก่อนเพิ่มตัวใหม่ |
| **Code Review บังคับ** | กำหนดว่าทุก PR ที่แก้ `TEMPLATES.OPTIONS.context_processors` ต้องมีคน review อย่างน้อย 1 คน เพราะผลกระทบเป็นวงกว้างทั้งเว็บไซต์ |
| **เขียนเทสต์ตรวจสอบ Key ไม่ชนกัน** | เขียนเทสต์ที่เรียก context processor ทั้งหมดแล้วเช็คว่าไม่มี key ซ้ำกันเลย (ดูตัวอย่างขั้นตอนที่ 293.6) |

### 293.6 เขียนเทสต์ป้องกัน Key ชนกันแบบอัตโนมัติ

```python
# blog/tests/test_context_processor_collisions.py
from django.conf import settings
from django.test import RequestFactory, TestCase
from django.utils.module_loading import import_string


class ContextProcessorCollisionTests(TestCase):
    """เทสต์นี้จะ fail ทันทีถ้ามีใครในทีมเผลอเพิ่ม context processor ที่คืน
    key ชนกับตัวที่มีอยู่แล้ว — ควรรันใน CI ทุกครั้งที่มีการแก้ settings.py"""

    def test_no_duplicate_keys_among_custom_processors(self):
        request = RequestFactory().get('/')
        engine_config = settings.TEMPLATES[0]['OPTIONS']['context_processors']

        # ตัดตัวของ Django เองออก เพราะเราตั้งใจให้ auth/messages/request ทำงานปกติ
        custom_processors = [
            path for path in engine_config if path.startswith('blog.')
        ]

        seen_keys = {}
        for path in custom_processors:
            processor = import_string(path)
            result = processor(request)
            for key in result:
                self.assertNotIn(
                    key, seen_keys,
                    msg=(
                        f"Key '{key}' ชนกันระหว่าง '{path}' กับ '{seen_keys.get(key)}' "
                        "— context processor สองตัวคืนค่าชื่อตัวแปรซ้ำกัน"
                    ),
                )
                seen_keys[key] = path
```

เทสต์นี้เป็นตัวอย่างที่ดีของหลักการ **"ป้องกันบั๊กที่เงียบที่สุดด้วยการเขียนเทสต์ที่
ชัดเจนที่สุด"** — เพราะการชนกันของ context processor ไม่มีทาง error ตอนรัน
`runserver` เลย มันจะแสดงผลผิดเงียบ ๆ เท่านั้น เทสต์แบบนี้จึงมีคุณค่ามากในทีมที่มี
นักพัฒนาหลายคนเพิ่ม context processor ของตัวเองเข้ามาเรื่อย ๆ

---

## ขั้นตอนที่ 294: ภาพรวม Jinja2 เป็น Alternative Template Backend ใน Django

### 294.1 ทบทวนจาก Part 008: ทำไมหลักสูตรนี้เลือกใช้ DTL

Part 008 ขั้อ 71.7 เกริ่นไว้แล้วว่า Django รองรับ **Jinja2** เป็นทางเลือกแทน DTL
ขั้นตอนนี้จะพาไปตั้งค่าและใช้งานจริง เพื่อให้คุณรู้จักไว้ — แม้หลักสูตรนี้จะยึด DTL
ตลอดทั้งหลักสูตร แต่หลายทีมในโลกจริง (โดยเฉพาะทีมที่ย้ายมาจาก Flask หรือทีมที่ให้
ความสำคัญกับความเร็วในการ render สูงมาก) เลือกใช้ Jinja2 กับ Django และคุณควรอ่าน
โค้ดแบบนี้ออกเมื่อเจอในโปรเจกต์อื่น

### 294.2 ติดตั้งและตั้งค่า Jinja2 Backend

```bash
pip install Jinja2
```

Django มี backend สำเร็จรูปให้ในตัวอยู่แล้วที่ `django.template.backends.jinja2.Jinja2`
เพิ่มเข้าไปเป็น**อีกตัวหนึ่ง**ใน `TEMPLATES` (list เดิมที่มี DTL อยู่แล้ว):

```python
# config/settings.py
TEMPLATES = [
    {
        'BACKEND': 'django.template.backends.django.DjangoTemplates',
        'DIRS': [BASE_DIR / 'templates'],
        'APP_DIRS': True,
        'OPTIONS': {
            'context_processors': [
                'django.template.context_processors.debug',
                'django.template.context_processors.request',
                'django.contrib.auth.context_processors.auth',
                'django.contrib.messages.context_processors.messages',
                'blog.context_processors.site_metadata',
                'blog.context_processors.navigation_menu',
                'blog.context_processors.categories_for_sidebar',
            ],
        },
    },
    {
        'BACKEND': 'django.template.backends.jinja2.Jinja2',
        'DIRS': [BASE_DIR / 'jinja2'],
        'APP_DIRS': True,
        'OPTIONS': {
            'environment': 'config.jinja2_env.environment',
            'context_processors': [
                'django.template.context_processors.request',
                'blog.context_processors.site_metadata',
            ],
        },
    },
]
```

> **สังเกตชื่อโฟลเดอร์ `jinja2` ไม่ใช่ `templates`**: เมื่อ `APP_DIRS: True` สำหรับ
> engine นี้ Django จะมองหาโฟลเดอร์ชื่อ **`jinja2/`** ในแต่ละแอป (ไม่ใช่ `templates/`)
> นี่คือธรรมเนียมที่ Django กำหนดไว้เพื่อไม่ให้ไฟล์ของสอง engine ปะปนกันในโฟลเดอร์
> เดียวกัน — ถ้าทั้งสอง backend มองหาไฟล์ชื่อเดียวกันในโฟลเดอร์เดียวกัน Django จะไม่รู้
> ว่าไฟล์นั้นควรถูก parse ด้วย engine ไหน การแยกโฟลเดอร์จึงทำให้ชัดเจนตั้งแต่ต้น

### 294.3 `environment` Function: จุดที่ Register Global Function/Filter ให้ Jinja2

Jinja2 backend **ไม่มีระบบ `{% load %}` แบบ DTL** ทุกอย่างที่ template จะเรียกใช้ได้
ต้องถูกลงทะเบียนไว้ล่วงหน้าผ่าน **environment function**:

```python
# config/jinja2_env.py
from django.templatetags.static import static
from django.urls import reverse
from jinja2 import Environment


def environment(**options):
    env = Environment(**options)
    env.globals.update({
        'static': static,
        'url': reverse,
    })
    return env
```

จากนั้นใน Jinja2 template จะเรียก `static()`/`url()` เหมือนฟังก์ชัน Python ปกติ
(มีวงเล็บ ใส่ argument ได้ตรง ๆ) แทนที่จะเป็น tag แบบ DTL:

```html
<!-- jinja2/base.html -->
<!DOCTYPE html>
<html lang="th">
<head>
    <meta charset="UTF-8">
    <title>{% block title %}{{ site_name }}{% endblock %}</title>
    <link rel="stylesheet" href="{{ static('blog/css/style.css') }}">
</head>
<body>
    <nav>
        <a href="{{ url('blog:list') }}">{{ site_name }}</a>
    </nav>
    <main>
        {% block content %}{% endblock %}
    </main>
</body>
</html>
```

### 294.4 ตารางเปรียบเทียบ Syntax: DTL vs Jinja2 แบบละเอียด

| ประเด็น | Django Template Language (DTL) | Jinja2 |
|---|---|---|
| แสดงตัวแปร | `{{ post.title }}` | `{{ post.title }}` (เหมือนกัน) |
| เรียก Method ที่รับ Argument | ❌ ทำไม่ได้ (Part 008 ขั้อ 72.5) | ✅ ทำได้ตรง ๆ `{{ post.get_summary(20) }}` |
| Static file | `{% load static %}` แล้ว `{% static 'x.css' %}` | เรียกฟังก์ชันที่ลงทะเบียนไว้ `{{ static('x.css') }}` |
| URL reverse | `{% url 'blog:list' %}` | `{{ url('blog:list') }}` |
| Loop ว่างเปล่า | `{% for %}...{% empty %}...{% endfor %}` | `{% for %}...{% else %}...{% endfor %}` (ใช้ `else` ไม่ใช่ `empty`!) |
| นิพจน์คณิตศาสตร์ใน Template | ❌ ทำไม่ได้ `{{ 1 + 1 }}` | ✅ ทำได้ `{{ 1 + 1 }}`, `{{ price * quantity }}` |
| จัดกลุ่มเงื่อนไขด้วยวงเล็บ | ❌ ทำไม่ได้ (Part 008 ขั้อ 73.2) | ✅ ทำได้ `{% if (a and b) or c %}` |
| สืบทอด Template | `{% extends 'base.html' %}` / `{% block %}...{% endblock %}` | เหมือนกันทุกประการ |
| เรียกเนื้อหา Parent Block | `{{ block.super }}` | `{{ super() }}` (มีวงเล็บ เพราะเป็น function call) |
| Comment | `{# ... #}` | `{# ... #}` (เหมือนกัน) |
| Reusable Component | `{% include %}`, custom tag (Part 029) | `{% include %}`, **Macro** (`{% macro %}...{% endmacro %}`) |
| Auto-escaping (ป้องกัน XSS) | เปิดเสมอโดย default | Django's Jinja2 backend ตั้ง `autoescape=True` ให้อัตโนมัติ (ต่างจาก Jinja2 เปล่า ๆ นอก Django ที่ปิดโดย default) |
| Custom Filter/Tag | `templatetags/` + `@register.filter`/`@register.simple_tag` (Part 029) | เพิ่มเข้า `env.filters[...]` / `env.globals[...]` ใน environment function |

### 294.5 ข้อควรระวัง: พฤติกรรม Context Processor ต่างจาก DTL เล็กน้อย

ทั้ง DTL และ Jinja2 backend รองรับ `context_processors` ใน `OPTIONS` เหมือนกัน
แต่กลไกเบื้องหลัง**ต่างกัน**: DTL ใช้ระบบ context stack ที่ทำให้ **context จาก View
ชนะ context processor เสมอ** (ตามที่พิสูจน์ไว้ในขั้อ 293.4) ส่วน Jinja2 backend
ใช้วิธี merge ค่าจาก context processor เข้ากับ dict ที่ได้รับมาแบบตรงไปตรงมากว่า
ซึ่ง**อาจให้ผลต่างกันในกรณีที่ชื่อตัวแปรชนกัน**ระหว่างสอง backend

**บทเรียนที่สำคัญกว่ารายละเอียดทางเทคนิค**: ไม่ว่าจะใช้ backend ไหน **การตั้งชื่อ
ตัวแปรชนกันคือสิ่งที่ต้องหลีกเลี่ยงเสมอ** (ตามหลักการในขั้อ 293.5) อย่าพึ่งพา
พฤติกรรม "ใครชนะเมื่อชนกัน" ของ backend ใดเป็นกลยุทธ์การออกแบบ — ให้ถือว่าเป็น
พฤติกรรม undefined ที่ไม่ควรเกิดขึ้นตั้งแต่แรก

### 294.6 คำแนะนำระดับมืออาชีพ: ควรผสมสอง Backend ในโปรเจกต์เดียวหรือไม่

| สถานการณ์ | คำแนะนำ |
|---|---|
| โปรเจกต์ใหม่ ทีมคุ้นเคย Django เป็นหลัก | ใช้ DTL อย่างเดียวตลอดทั้งหลักสูตรนี้ — ปลอดภัยกว่า, ระบบนิเวศ 3rd-party (django-crispy-forms, django-widget-tweaks จาก Part 027) ส่วนใหญ่เขียนมาให้ DTL |
| ทีมย้ายมาจาก Flask ทั้งทีม คุ้นเคย Jinja2 มาก | พิจารณาใช้ Jinja2 ทั้งโปรเจกต์ได้ แต่ต้องเตรียมใจว่า 3rd-party package บางตัวอาจต้องปรับ adapter เอง |
| ต้องการ render หน้าที่ทราฟฟิกสูงมากด้วยความเร็วสูงสุด | ใช้ทั้งสองพร้อมกัน: DTL สำหรับหน้าทั่วไป/Admin-like, Jinja2 เฉพาะหน้า public ที่ต้องการความเร็ว (Django จะเลือก backend อัตโนมัติจากนามสกุลโฟลเดอร์ที่เจอไฟล์) |
| โปรเจกต์เล็ก-กลางทั่วไป | **ไม่แนะนำผสมสอง backend** เพิ่มความซับซ้อนโดยไม่คุ้มค่า — เลือกอย่างใดอย่างหนึ่งแล้วยึดตลอดทั้งโปรเจกต์ |

หลักสูตรนี้จะใช้ **DTL ตลอดทั้งหลักสูตรที่เหลือ** เนื่องจากเหตุผลด้าน security by
default และความเข้ากันได้กับ 3rd-party ecosystem ที่กว้างกว่ามาก

---

## ขั้นตอนที่ 295: Template Best Practices Checklist ระดับมืออาชีพ

### 295.1 หลักการที่ 1: DRY Templates — เลือกเครื่องมือให้ถูกกับปัญหา

Phase 3 สอนเครื่องมือลด HTML ซ้ำมาแล้วถึง 4 แบบ คำถามที่มือใหม่มักสับสนคือ "ควรใช้
อันไหนตอนไหน" ใช้แผนผังการตัดสินใจนี้:

```
ต้องการลด HTML ซ้ำ?
│
├─ ทั้งหน้ามีโครงเหมือนกัน (navbar, footer, <head>) ?
│   └─ ใช้ {% extends %} + {% block %} (Part 008 ขั้อ 75)
│
├─ ชิ้นส่วนเล็ก ๆ ใช้ซ้ำหลายที่ ไม่มี logic คำนวณเพิ่ม ?
│   └─ ใช้ {% include %} (Part 008 ขั้อ 76)
│
├─ ชิ้นส่วนต้อง "คำนวณ/query" ข้อมูลเองก่อนแสดงผล ?
│   └─ ใช้ inclusion_tag (Part 029 ขั้อ 284)
│
├─ ต้องการ "ฟังก์ชัน" แปลงค่าเดียว ไม่ render HTML ?
│   └─ ใช้ custom filter (Part 029 ขั้อ 283)
│
└─ ข้อมูลต้องปรากฏแทบทุกหน้าของทั้งเว็บไซต์แบบไม่มีเงื่อนไข ?
    └─ ใช้ context processor (Part 030 ขั้อ 291-292) — ใช้เท่าที่จำเป็นจริง ๆ
```

### 295.2 หลักการที่ 2: ห้ามใส่ Business Logic หนักใน Template

ย้ำหลักการที่ปรากฏซ้ำตลอด Phase 3: **"Template ทำหน้าที่แค่แสดงผล ไม่ใช่คิดคำนวณ"**
ตัวอย่างเปรียบเทียบ:

```html
<!-- ❌ ไม่ควรทำ: คำนวณราคารวมพร้อมส่วนลดตรงใน template -->
{% if order.subtotal > 1000 %}
    {% with discount=order.subtotal|floatformat:2 %}
        <p>ราคารวมหลังหักส่วนลด 10%:
           {{ order.subtotal|add:"-100"|floatformat:2 }} บาท</p>
    {% endwith %}
{% else %}
    <p>ราคารวม: {{ order.subtotal|floatformat:2 }} บาท</p>
{% endif %}
```

```python
# ✅ ควรทำ: คำนวณใน Model หรือ View แล้วส่งค่าสำเร็จรูปเข้า template
# models.py
class Order(models.Model):
    subtotal = models.DecimalField(max_digits=10, decimal_places=2)

    @property
    def total_after_discount(self):
        if self.subtotal > 1000:
            return self.subtotal * Decimal('0.9')
        return self.subtotal
```

```html
<!-- ✅ Template สั้น อ่านง่าย ไม่มี logic ทางธุรกิจปนอยู่เลย -->
<p>ราคารวมหลังหักส่วนลด: {{ order.total_after_discount|floatformat:2 }} บาท</p>
```

**เหตุผล**: logic ที่อยู่ใน Model/View **เทสต์ได้ตรง ๆ ด้วย unit test ธรรมดา**
(`self.assertEqual(order.total_after_discount, ...)`) แต่ logic ที่ฝังอยู่ใน
template ต้องเทสต์ผ่านการ render ทั้งหน้าเท่านั้น (ซับซ้อนกว่า, เปราะบางกว่า,
ตรวจจับ edge case ยากกว่ามาก) — นี่คือเหตุผลเดียวกับที่ DTL จงใจจำกัดความสามารถ
(Part 008 ขั้อ 71.7)

### 295.3 หลักการที่ 3: ตั้งชื่อไฟล์และตัวแปรให้สม่ำเสมอ

| หมวด | ธรรมเนียม | ตัวอย่าง |
|---|---|---|
| Template ของหน้าเต็ม | `<action>.html` หรือ `<model>_<action>.html` | `list.html`, `detail.html`, `post_form.html` |
| Partial (ใช้ผ่าน `include` เท่านั้น) | ขึ้นต้นด้วย `_` | `_post_card.html`, `_pagination.html` |
| Namespace ตามแอป | ซ้อนโฟลเดอร์ชื่อแอปเสมอ (Part 008 ขั้อ 71.4) | `blog/templates/blog/list.html` |
| Context Variable | `snake_case`, ชื่อสื่อความหมาย, ใส่ prefix แอปถ้าเสี่ยงชนกัน (ขั้อ 293.5) | `sidebar_categories`, `published_post_count` |
| Block Name | `snake_case`, บอกตำแหน่ง/หน้าที่ชัดเจน ไม่ใช้ชื่อกำกวมเช่น `block1` | `content`, `extra_head`, `breadcrumb` |
| Custom Template Tag/Filter | `snake_case`, ชื่อกริยา/คำนามที่สื่อการกระทำ | `reading_time`, `truncate_smart`, `render_post_card` |
| CSS Class ที่ผูกกับ Component | BEM (`block__element--modifier`) | `post-card__meta`, `navbar__brand` |

### 295.4 Template Best Practices Checklist ฉบับเต็ม

ใช้ checklist นี้ตรวจสอบ template ทุกครั้งก่อนส่ง code review:

- [ ] ไม่มี HTML โครงสร้างหลัก (head, navbar, footer) ที่ copy-paste ซ้ำในหลายไฟล์
- [ ] ทุก partial ที่ไม่ได้ตั้งใจให้ extends ตั้งชื่อขึ้นต้นด้วย `_`
- [ ] ไม่มีการคำนวณที่ซับซ้อนกว่า filter ธรรมดา 1 ตัวใน `{{ }}` — ถ้าซับซ้อนกว่านั้น
      ย้ายไป Model property หรือ View
- [ ] ทุก URL ใช้ `{% url %}` ไม่มี hardcode string path เด็ดขาด
- [ ] ทุกไฟล์ static ใช้ `{% static %}` ไม่มี hardcode `/static/...` ตรง ๆ
- [ ] Context variable ที่ชื่อกำกวม/สั้นเกินไป (เช่น `data`, `items`, `obj`) ถูก
      เปลี่ยนเป็นชื่อที่สื่อความหมายชัดเจน
- [ ] ไม่มี context processor ตัวใหม่ถูกเพิ่มโดยไม่ผ่าน code review (ขั้อ 293.5)
- [ ] ทุก user-generated content ที่แสดงผล **ไม่ได้** ใช้ `|safe` โดยไม่จำเป็น
      (ทบทวน Part 008 ขั้อ 78 และ Part 029 ขั้อ 288)
- [ ] เขียนเทสต์อย่างน้อย `assertTemplateUsed` + `assertContains` สำหรับหน้าใหม่
      ทุกหน้า (ขั้อ 297)
- [ ] ใช้ semantic HTML tag ที่ถูกต้อง (`<nav>`, `<main>`, `<article>`) ไม่ใช้
      `<div>` ล้วนทั้งหน้า (ขั้อ 299)
- [ ] ฟอร์มทุกฟอร์มมี `<label for="...">` จับคู่กับ input ถูกต้อง (ขั้อ 299.4)

---

## ขั้นตอนที่ 296: การ Debug Template

### 296.1 `{% debug %}`: ดูสถานะ Context ทั้งหมด ณ จุดนั้นในเทมเพลต

Django มี built-in tag ชื่อ `{% debug %}` ที่แสดงข้อมูล debug ทั้งหมด ณ จุดที่วางไว้
ในเทมเพลต — ทั้ง context ทุก layer ที่มีอยู่ ณ ตอนนั้น และรายชื่อ module ที่ระบบ
import ไว้:

```html
{% if debug %}
    <div style="background:#111;color:#0f0;padding:1rem;font-family:monospace;">
        {% debug %}
    </div>
{% endif %}
```

**ข้อควรระวังสำคัญ**: `{% debug %}` แสดงข้อมูลละเอียดมาก (รวมถึงค่าที่อาจ sensitive
เช่น session data) **ต้องครอบด้วย `{% if debug %}`เสมอ** (ตัวแปร `debug` มาจาก
`django.template.context_processors.debug` ซึ่งเป็น `True` ก็ต่อเมื่อ
`settings.DEBUG = True` และ IP อยู่ใน `INTERNAL_IPS` เท่านั้น — ทบทวนจาก Part 008
ขั้อ 79.3) เพื่อไม่ให้หลุดไปแสดงใน production โดยไม่ได้ตั้งใจ

### 296.2 Django Debug Toolbar's Templates Panel

Part 010 ติดตั้ง **Django Debug Toolbar** ไว้แล้วสำหรับดู SQL queries (ที่ใช้จริง
ใน Part 013) มาดูอีกหนึ่ง panel ที่มีประโยชน์มากสำหรับ Part นี้: **Templates panel**

เปิด Debug Toolbar (มุมขวาของหน้าเว็บ เมื่อ `DEBUG=True` และ IP อยู่ใน
`INTERNAL_IPS`) แล้วคลิกแท็บ **"Templates"** จะเห็นข้อมูลสำคัญ 3 อย่าง:

| ข้อมูลที่แสดง | ประโยชน์ |
|---|---|
| รายชื่อ **ทุก template** ที่ถูก render สำหรับ request นี้ (รวม `{% include %}` และ inheritance chain) | เห็นทันทีว่า `list.html` จริง ๆ แล้วดึง `base.html`, `_post_card.html`, `partials/navbar.html` มาต่อกันกี่ไฟล์ |
| **Context** ของแต่ละ template แยกเป็นรายการตัวแปร | ใช้ตรวจสอบว่าตัวแปรที่คาดหวัง (เช่น `sidebar_categories` จากขั้อ 292) มาถึง template จริงหรือไม่ ค่าที่ได้ถูกต้องหรือไม่ |
| รายชื่อ **Context Processors** ที่ทำงานสำหรับ request นี้ | ใช้ตรวจสอบโดยตรงว่า context processor ตัวไหนทำงานบ้าง เรียงลำดับอย่างไร — มีประโยชน์มากเวลา debug ปัญหาจากขั้อ 293 |

Templates panel จึงเป็นเครื่องมือที่ตอบโจทย์ Part นี้โดยตรง: เวลาสงสัยว่า
"ทำไมตัวแปรนี้ค่าไม่ตรงกับที่คาด" หรือ "context processor ตัวไหนกันแน่ที่ชนะ"
เปิด panel นี้ดูได้ทันทีโดยไม่ต้องเขียน debug code เพิ่มเลย

### 296.3 ข้อความ Error ที่พบบ่อยที่สุดเมื่อ Template ผิดพลาด

เมื่อ `DEBUG=True` Django จะแสดงหน้า error แบบละเอียด (technical error page)
พร้อมชี้ตำแหน่งที่ผิดในไฟล์ template โดยตรง มาดู error ที่พบบ่อยที่สุด 5 แบบ:

#### 1. `TemplateDoesNotExist`

```
TemplateDoesNotExist at /posts/
blog/lsit.html
```

**สาเหตุที่พบบ่อย**: พิมพ์ชื่อไฟล์ผิด (`lsit.html` แทน `list.html`) หรือไฟล์ไม่ได้
อยู่ในตำแหน่งที่ `DIRS`/`APP_DIRS` มองหา (Part 008 ขั้อ 71.6) หน้า error จะมี
ส่วน **"Template-loader postmortem"** ที่แสดงรายการ **ทุก path ที่ Django ลองหา
มาแล้ว** เรียงตาม engine และ loader — เป็นข้อมูลที่มีประโยชน์มากในการหาสาเหตุ
เพราะบอกตรง ๆ ว่า Django มองหาที่ไหนบ้างและทำไมไม่เจอ

#### 2. `TemplateSyntaxError` — Tag ไม่ปิด

```
TemplateSyntaxError at /posts/
Unclosed tag on line 12: 'block'. Looking for one of: endblock.
```

**สาเหตุ**: เปิด `{% block content %}` แล้วลืมปิดด้วย `{% endblock %}`

#### 3. `TemplateSyntaxError` — Tag ผิดตำแหน่งหรือไม่รู้จัก

```
TemplateSyntaxError at /posts/
Invalid block tag on line 3: 'static'. Did you forget to register or load this tag?
```

**สาเหตุที่พบบ่อยที่สุด**: ลืม `{% load static %}` ก่อนใช้ `{% static %}`
(ทบทวน Part 008 ขั้อ 77.1) หรือใช้ custom tag จาก `blog_extras` โดยลืม
`{% load blog_extras %}` (Part 029 ขั้อ 281) ข้อความ error จะบอกชัดเจนว่า
"Did you forget to register or load this tag?" ซึ่งเป็นคำใบ้ตรงประเด็นเสมอ

#### 4. `NoReverseMatch` จาก `{% url %}`

```
django.urls.exceptions.NoReverseMatch: Reverse for 'detial' not found.
'detial' is not a valid view function or pattern name.
```

**สาเหตุ**: พิมพ์ชื่อ URL pattern ผิด (`detial` แทน `detail`) หรือลืมส่ง argument
ที่ URL pattern ต้องการ (เช่น `{% url 'blog:detail' %}` โดยไม่ส่ง `slug=...`)

#### 5. ตัวแปรผิด — "เงียบ" ไม่มี Error แต่ผลลัพธ์ผิด

ทบทวนจาก Part 008 ขั้อ 72.6: `{{ post.titel }}` (พิมพ์ผิด) จะ**ไม่ error เลย**
เพียงแค่แสดงเป็นค่าว่าง — เป็น error ประเภทเดียวที่ Django ไม่แจ้งเตือนอัตโนมัติ
วิธี debug ที่ได้ผลที่สุดคือเปิด **Templates panel** ของ Debug Toolbar (ขั้อ 296.2)
ดู context จริงที่ template ได้รับ แล้วเทียบชื่อ key ทีละตัว หรือใช้เทคนิคชั่วคราว
ตั้งค่า `'string_if_invalid': 'INVALID: %s'` ตามที่แนะนำใน Part 008 ขั้อ 80.5

### 296.4 ขั้นตอนการ Debug Template อย่างเป็นระบบ

เมื่อเจอปัญหา template แนะนำให้ไล่ตามลำดับนี้เสมอ:

1. **อ่านข้อความ error เต็ม ๆ ก่อนเสมอ** — Django บอกบรรทัดและไฟล์ที่ผิดตรง ๆ
   ในกรณีส่วนใหญ่ (ยกเว้นกรณีตัวแปรผิดที่เงียบตามข้อ 5)
2. **เปิด Debug Toolbar → Templates panel** เพื่อดู context จริงและรายการ
   template ที่ render จริง
3. **ตรวจสอบ `{% load %}`** ว่าครบทุกตัวที่ต้องใช้หรือยัง (static, custom tag
   library)
4. **ใส่ `{% debug %}` ชั่วคราว** (ครอบด้วย `{% if debug %}` เสมอ) ตรงจุดที่สงสัย
   เพื่อดู context stack เต็ม ๆ ณ จุดนั้น
5. **ลบ debug code ทั้งหมดก่อน commit** — `{% debug %}` ไม่ควรเหลืออยู่ใน
   template ที่ merge เข้า main branch

---

## ขั้นตอนที่ 297: การเขียน Test สำหรับ Template

### 297.1 ทบทวนเครื่องมือจาก Part 024 แล้วโฟกัสที่ฝั่ง Template

Part 024 ขั้อ 238 สอนการเทสต์ CBV ด้วย `self.client` และ `RequestFactory` ไปแล้ว
ขั้นตอนนี้จะโฟกัสเฉพาะการเทสต์ **ผลลัพธ์ HTML ที่ template render ออกมา** ซึ่งเป็น
คนละมุมกับการเทสต์ logic ของ View

### 297.2 `assertTemplateUsed`: ยืนยันว่า View เรียกใช้ Template ที่ถูกต้อง

```python
# blog/tests/test_templates.py
from django.test import TestCase
from django.urls import reverse

from blog.models import Category, Post
from django.contrib.auth import get_user_model

User = get_user_model()


class PostListTemplateTests(TestCase):
    @classmethod
    def setUpTestData(cls):
        cls.author = User.objects.create_user(username="tester", password="x")
        cls.category = Category.objects.create(name="Django")
        cls.post = Post.objects.create(
            title="ทดสอบ Template",
            content="เนื้อหาทดสอบ",
            category=cls.category,
            author=cls.author,
            is_published=True,
        )

    def test_uses_correct_templates(self):
        response = self.client.get(reverse('blog:list'))
        self.assertTemplateUsed(response, 'blog/list.html')
        self.assertTemplateUsed(response, 'base.html')          # ทดสอบ inheritance chain ด้วยได้
        self.assertTemplateUsed(response, 'blog/_post_card.html')  # ทดสอบ include ด้วยได้

    def test_does_not_use_detail_template(self):
        response = self.client.get(reverse('blog:list'))
        self.assertTemplateNotUsed(response, 'blog/detail.html')
```

**ข้อสังเกตสำคัญ**: `assertTemplateUsed` เช็คได้ทั้ง template หลักที่ view ระบุตรง ๆ
**และ** ทุก template ที่ถูกดึงมาผ่าน `{% extends %}`/`{% include %}`/inclusion tag
เพราะ Django เก็บ log ของทุกไฟล์ที่ engine โหลดจริงระหว่าง render ไม่ใช่แค่ไฟล์แรก
ที่ view ส่งชื่อมา

### 297.3 `assertContains` / `assertNotContains`: ตรวจสอบเนื้อหาจริงใน HTML

```python
class PostDetailTemplateTests(TestCase):
    @classmethod
    def setUpTestData(cls):
        cls.author = User.objects.create_user(username="tester2", password="x")
        cls.post = Post.objects.create(
            title="Django คือเฟรมเวิร์กที่ดีที่สุด",
            content="เนื้อหาโดยละเอียด " * 20,
            author=cls.author,
            is_published=True,
        )

    def test_page_shows_post_title(self):
        response = self.client.get(self.post.get_absolute_url())
        self.assertContains(response, "Django คือเฟรมเวิร์กที่ดีที่สุด")
        self.assertContains(response, "<article", count=1)   # ตรวจสอบว่ามี <article> แค่ 1 ครั้ง

    def test_page_does_not_leak_admin_only_content(self):
        response = self.client.get(self.post.get_absolute_url())
        self.assertNotContains(response, "ลบบทความ")   # ปุ่มลบไม่ควรโชว์กับ anonymous user

    def test_html_snippet_matches_exactly(self):
        response = self.client.get(self.post.get_absolute_url())
        self.assertContains(
            response,
            '<h1>Django คือเฟรมเวิร์กที่ดีที่สุด</h1>',
            html=True,   # เปรียบเทียบแบบ "เข้าใจโครงสร้าง HTML" ไม่สนใจช่องว่าง/ลำดับ attribute
        )
```

`assertContains(response, text, count=N)` มีประโยชน์มากในการยืนยันว่าองค์ประกอบ
สำคัญปรากฏ**ตามจำนวนที่ถูกต้องเป๊ะ** (เช่น การ์ดบทความต้องมีตามจำนวนบทความจริง
ไม่ใช่ซ้ำโดยไม่ตั้งใจจาก bug ใน `{% for %}`) ส่วน `html=True` ทำให้การเทียบ HTML
snippet ทนทานต่อความต่างเล็กน้อยของช่องว่างหรือลำดับ attribute ที่ไม่สำคัญ

### 297.4 Render Template แยกทดสอบโดยไม่ผ่าน View/URL เลย

บางครั้งอยากเทสต์แค่ partial เดียว (เช่น `_post_card.html`) โดยไม่ต้องสร้าง
View/URL เต็มรูปแบบ ใช้ `render_to_string()` ได้ตรง ๆ:

```python
# blog/tests/test_partials.py
from django.template.loader import render_to_string
from django.test import TestCase

from blog.models import Post
from django.contrib.auth import get_user_model

User = get_user_model()


class PostCardPartialTests(TestCase):
    @classmethod
    def setUpTestData(cls):
        cls.author = User.objects.create_user(username="tester3", password="x")
        cls.post = Post.objects.create(
            title="ทดสอบ Partial แยก",
            content="a" * 500,
            author=cls.author,
            is_published=True,
        )

    def test_post_card_renders_title_and_truncated_content(self):
        html = render_to_string('blog/_post_card.html', {'post': self.post})
        self.assertIn('ทดสอบ Partial แยก', html)
        self.assertIn('…', html)   # ยืนยันว่า truncatewords ตัดข้อความจริง (Part 008 ขั้อ 74.2)
```

วิธีนี้**เร็วกว่า**การทดสอบผ่าน `self.client.get()` มาก เพราะข้าม URL routing,
middleware, และ View logic ทั้งหมด — เหมาะกับการเทสต์ partial ที่ซับซ้อนแบบแยกส่วน
(unit test) เช่นเดียวกับหลักการ `RequestFactory` ในขั้อ 297.1 (Part 024 ขั้อ 238.3)

### 297.5 เทสต์ Context Processor ผ่าน Integration Test เต็มรูปแบบ

ปิดท้ายด้วยการรวมทุกเทคนิคเข้าด้วยกัน เทสต์ว่า context processor จากขั้อ 292 ทำงาน
ถูกต้องจริงเมื่อผ่าน HTTP request เต็มรูปแบบ (ไม่ใช่แค่เรียกฟังก์ชันตรง ๆ แบบขั้อ 291.5):

```python
class SidebarIntegrationTests(TestCase):
    @classmethod
    def setUpTestData(cls):
        cls.author = User.objects.create_user(username="tester4", password="x")
        cls.category_with_posts = Category.objects.create(name="มีบทความ")
        cls.category_empty = Category.objects.create(name="ไม่มีบทความเลย")
        Post.objects.create(
            title="บทความในหมวดที่มีข้อมูล",
            content="...",
            category=cls.category_with_posts,
            author=cls.author,
            is_published=True,
        )

    def test_sidebar_shows_only_categories_with_published_posts(self):
        response = self.client.get(reverse('blog:list'))
        self.assertContains(response, "มีบทความ")
        self.assertNotContains(response, "ไม่มีบทความเลย")
```

เทสต์นี้ยืนยันพฤติกรรมจริงของ `categories_for_sidebar` (การ `filter(post_count__gt=0)`
ในขั้อ 292.2) ผ่านหน้าเว็บจริงทั้งหน้า — เป็นตัวอย่างที่ดีว่าทำไมการเทสต์ทั้งระดับ
unit (297.4, 291.5) และระดับ integration (297.2-297.5) ต้องทำควบคู่กันเสมอ

---

## ขั้นตอนที่ 298: เกริ่น Internationalization (i18n) ใน Template

### 298.1 ทำไมต้องรู้ i18n แม้หลักสูตรนี้จะเน้นภาษาไทยเป็นหลัก

ตลอดหลักสูตรนี้เรา hardcode ข้อความภาษาไทยลงใน template ตรง ๆ เช่น
`<h1>บทความทั้งหมด</h1>` เพราะเป้าหมายของหลักสูตรคือสร้างเว็บไซต์ภาษาไทยเป็นหลัก
ซึ่งเป็นแนวทางที่ถูกต้องและเหมาะสมสำหรับโปรเจกต์ที่มีกลุ่มเป้าหมายเป็นคนไทยเท่านั้น

แต่ **นักพัฒนาระดับมืออาชีพต้องรู้จักระบบ i18n ไว้เสมอ** เพราะ:

- โปรเจกต์จริงจำนวนมาก (โดยเฉพาะ SaaS หรือ product ที่ขยายตลาดต่างประเทศ) ต้อง
  รองรับหลายภาษาตั้งแต่วันแรกหรือในอนาคตอันใกล้
- Package แบบ open source ที่คุณอาจสร้างและแชร์ให้คนทั่วโลกใช้ ควรรองรับ i18n
  เพื่อให้ทีมอื่นแปลเป็นภาษาของตัวเองได้โดยไม่ต้องแก้โค้ด
- Django เองก็ใช้ i18n ในทุกส่วน (Django Admin ที่เปลี่ยนภาษาได้จาก `LANGUAGE_CODE`
  คือตัวอย่างที่ชัดเจนที่สุด)

ขั้นตอนนี้จึงเป็นเพียง**การเกริ่นให้รู้จักพื้นฐานเท่านั้น** ไม่ใช่การนำมาใช้จริงใน
โปรเจกต์บล็อกของหลักสูตรนี้ (ซึ่งจะ hardcode ภาษาไทยต่อไปตลอดทั้งหลักสูตร)

### 298.2 ตั้งค่าพื้นฐานที่จำเป็นสำหรับ i18n

```python
# config/settings.py
USE_I18N = True   # ค่า default ของ startproject อยู่แล้ว

MIDDLEWARE = [
    'django.middleware.security.SecurityMiddleware',
    'django.contrib.sessions.middleware.SessionMiddleware',
    'django.middleware.locale.LocaleMiddleware',   # ← ต้องอยู่หลัง Session, ก่อน Common
    'django.middleware.common.CommonMiddleware',
    # ... middleware อื่น ๆ
]

LANGUAGE_CODE = 'th'   # ภาษาเริ่มต้นของโปรเจกต์นี้

LANGUAGES = [
    ('th', 'ไทย'),
    ('en', 'English'),
]

LOCALE_PATHS = [BASE_DIR / 'locale']   # โฟลเดอร์เก็บไฟล์แปลภาษา (.po/.mo)
```

`LocaleMiddleware` คือตัวที่ตรวจจับภาษาที่ผู้ใช้ต้องการ (จาก URL prefix, session,
cookie, หรือ `Accept-Language` header ของเบราว์เซอร์ตามลำดับ) แล้วตั้งค่าภาษาที่ใช้
สำหรับ request นั้นให้อัตโนมัติ

### 298.3 `{% load i18n %}`, `{% translate %}`/`{% trans %}`, `{% blocktranslate %}`/`{% blocktrans %}`

```html
{% load i18n %}

<!-- {% translate %} คือชื่อที่แนะนำตั้งแต่ Django 3.1 เป็นต้นไป
     {% trans %} ยังใช้ได้เหมือนเดิม (เป็น alias ที่ Django คงไว้ให้ใช้ต่อไป) -->
<h1>{% translate "บทความทั้งหมด" %}</h1>
<a href="{% url 'blog:list' %}">{% trans "กลับหน้าแรก" %}</a>

<!-- {% blocktranslate %} ใช้เมื่อข้อความมีตัวแปรผสมอยู่ด้วย -->
{% blocktranslate with count=posts|length %}
    พบบทความทั้งหมด {{ count }} เรื่อง
{% endblocktranslate %}

<!-- รองรับ pluralization (เอกพจน์/พหูพจน์) ด้วย count + plural -->
{% blocktranslate count counter=posts|length %}
    มีบทความ {{ counter }} เรื่อง
{% plural %}
    มีบทความ {{ counter }} เรื่อง
{% endblocktranslate %}
```

> **หมายเหตุ**: ภาษาไทยไม่มีรูปพหูพจน์ต่างจากเอกพจน์ (ไม่เหมือนภาษาอังกฤษที่
> "1 post" vs "2 posts") ตัวอย่าง `{% plural %}` ข้างบนจึงแสดงข้อความเดียวกัน
> ทั้งสองกรณีสำหรับภาษาไทย แต่ระบบยังจำเป็นสำหรับภาษาอื่นที่ต้องแปล (เช่น
> ภาษาอังกฤษที่ไฟล์ `.po` จะแปล 2 บรรทัดนี้แยกกัน)

### 298.4 คำสั่งจัดการไฟล์แปลภาษา: `makemessages` และ `compilemessages`

```bash
# สแกนหาข้อความทั้งหมดที่ห่อด้วย {% translate %}/{% blocktranslate %}
# (และ gettext() ในฝั่ง Python) แล้วสร้างไฟล์ .po สำหรับภาษาอังกฤษ
python manage.py makemessages -l en

# หลังแปลข้อความในไฟล์ locale/en/LC_MESSAGES/django.po เสร็จแล้ว
# คอมไพล์เป็นไฟล์ .mo ที่ Django อ่านตอน runtime ได้จริง
python manage.py compilemessages
```

ไฟล์ `.po` ที่ได้จะมีรูปแบบประมาณนี้ (แก้ไขด้วยมือหรือเครื่องมือแปลเฉพาะทางเช่น
Poedit):

```po
#: blog/templates/blog/list.html:5
msgid "บทความทั้งหมด"
msgstr "All Posts"

#: blog/templates/blog/list.html:8
msgid "กลับหน้าแรก"
msgstr "Back to Home"
```

`makemessages` ต้องใช้เครื่องมือ `gettext` ของระบบปฏิบัติการ (ติดตั้งผ่าน
`brew install gettext` บน macOS หรือ `apt install gettext` บน Linux — Windows
มักต้องติดตั้งแยกผ่าน WSL2 ตามที่แนะนำใน Part 001 ขั้อ 6.3)

### 298.5 สรุปสั้น ๆ: หลักสูตรนี้ยังคง Hardcode ภาษาไทยต่อไป

เพื่อความชัดเจนของหลักสูตร เราจะ**ไม่**เปลี่ยน template ทั้งหมดในหลักสูตรที่เหลือ
ให้ใช้ `{% translate %}` เพราะจะเพิ่มความซับซ้อนโดยไม่จำเป็นสำหรับเป้าหมายของ
หลักสูตร (สร้างเว็บไซต์ภาษาไทย) แต่ตอนนี้คุณรู้แล้วว่าเมื่อไหร่ในอนาคตที่โปรเจกต์
จริงของคุณต้องรองรับหลายภาษา ระบบและคำสั่งที่ต้องใช้คืออะไร

---

## ขั้นตอนที่ 299: Accessibility (a11y) ใน Django Template

### 299.1 ทำไม Developer ระดับโลกต้องใส่ใจเรื่อง Accessibility

**Accessibility (การเข้าถึงได้)** หรือเรียกย่อว่า **a11y** (a + 11 ตัวอักษร + y)
คือการออกแบบเว็บไซต์ให้ **ทุกคนใช้งานได้จริง** รวมถึงผู้ใช้ที่มีความบกพร่องทาง
การมองเห็น การได้ยิน การเคลื่อนไหว หรือการรับรู้ เหตุผลที่นักพัฒนาระดับมืออาชีพต้อง
ให้ความสำคัญ:

| เหตุผล | รายละเอียด |
|---|---|
| **จริยธรรม** | เว็บไซต์คือโครงสร้างพื้นฐานสาธารณะยุคใหม่ การกีดกันคนกลุ่มใดกลุ่มหนึ่งออกจากการใช้งานเทียบเท่าการสร้างอาคารที่ไม่มีทางลาดสำหรับรถเข็น |
| **กฎหมาย** | หลายประเทศมีกฎหมายบังคับ (เช่น ADA ในสหรัฐฯ, EN 301 549 ในยุโรป) เว็บไซต์ภาครัฐและองค์กรขนาดใหญ่จำนวนมากถูกฟ้องร้องเพราะไม่รองรับ a11y |
| **ธุรกิจ** | ผู้ใช้ที่มีความบกพร่องคือกลุ่มลูกค้าจริงที่มีกำลังซื้อ การตัดกลุ่มนี้ออกคือการเสียโอกาสทางธุรกิจ |
| **SEO** | โครงสร้าง semantic HTML ที่ดีต่อ accessibility มักดีต่อ SEO ไปพร้อมกันด้วย (screen reader กับ search engine crawler "อ่าน" หน้าเว็บด้วยวิธีคล้ายกัน) |
| **คุณภาพโค้ดโดยรวม** | โค้ดที่ใส่ใจ a11y มักมีโครงสร้าง HTML ที่สะอาดและมีความหมาย (semantic) มากกว่าโค้ดที่ไม่ใส่ใจเรื่องนี้เลย |

มาตรฐานสากลที่ใช้อ้างอิงคือ **WCAG (Web Content Accessibility Guidelines)**
ขององค์กร W3C — หลักสูตรนี้จะไม่ลงลึกทุกข้อของ WCAG แต่จะแนะนำหลักปฏิบัติที่นำไปใช้
ได้ทันทีกับ Django Template

### 299.2 Semantic HTML: ใช้ Tag ที่มีความหมาย แทน `<div>` ทั้งหมด

ทบทวน `base.html` จาก Part 008 — สังเกตว่าเราใช้ `<nav>`, `<main>`, `<footer>`
อยู่แล้วโดยไม่รู้ตัวว่านี่คือจุดเริ่มต้นของ accessibility ที่ดี มาเสริมให้ครบขึ้น:

```html
<!-- templates/base.html (เวอร์ชันปรับปรุงเพื่อ Accessibility) -->
{% load static %}
<!DOCTYPE html>
<html lang="th">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>{% block title %}{{ site_name }}{% endblock %}</title>
    <link rel="stylesheet" href="{% static 'blog/css/style.css' %}">
</head>
<body>
    <a href="#main-content" class="skip-link">ข้ามไปยังเนื้อหาหลัก</a>

    <header>
        {% include 'partials/navbar.html' %}
    </header>

    <div class="layout">
        <main id="main-content" class="container">
            {% block content %}{% endblock %}
        </main>

        <aside class="sidebar" aria-label="หมวดหมู่บทความ">
            {% include 'partials/sidebar.html' %}
        </aside>
    </div>

    <footer>
        {% include 'partials/footer.html' %}
    </footer>
</body>
</html>
```

จุดสำคัญที่เพิ่มเข้ามา:

| องค์ประกอบ | เหตุผลด้าน Accessibility |
|---|---|
| `<html lang="th">` | บอก screen reader ว่าเนื้อหาเป็นภาษาไทย ทำให้ออกเสียงถูกต้อง (มีมาตั้งแต่ Part 001 อยู่แล้ว — เป็นตัวอย่างที่ดีว่า a11y basics ถูกสอนมาตลอดโดยไม่รู้ตัว) |
| **Skip link** (`ข้ามไปยังเนื้อหาหลัก`) | ผู้ใช้ที่ควบคุมเว็บด้วยคีย์บอร์ดล้วน (ไม่ใช้เมาส์) จะได้ไม่ต้อง Tab ผ่าน navbar ทุกลิงก์ก่อนถึงเนื้อหาจริงทุกหน้า |
| `<header>`, `<main>`, `<aside>`, `<footer>` | Semantic landmark — screen reader จับกลุ่มและอ่านให้ผู้ใช้ "กระโดด" ไปยังส่วนที่ต้องการได้ทันที (เช่น สั่ง "ไปที่ main content" ได้เลย) |
| `aria-label="หมวดหมู่บทความ"` บน `<aside>` | อธิบายว่า landmark นี้คืออะไร เพราะ `<aside>` เฉย ๆ ไม่บอกหน้าที่ชัดเจนพอ |

```css
/* blog/static/blog/css/style.css (เพิ่มเติม) */
.skip-link {
    position: absolute;
    left: -9999px;
    top: 0;
    background: #000;
    color: #fff;
    padding: 0.5rem 1rem;
    z-index: 999;
}

.skip-link:focus {
    left: 0;   /* ปรากฏให้เห็นเมื่อถูก focus ด้วยปุ่ม Tab เท่านั้น */
}
```

### 299.3 ARIA Attributes: เสริมความหมายเมื่อ HTML เฉย ๆ ไม่พอ

**ARIA (Accessible Rich Internet Applications)** คือชุด attribute ที่เพิ่มข้อมูล
ให้ assistive technology (เช่น screen reader) เข้าใจ element ที่ HTML มาตรฐาน
สื่อความหมายไม่ครบ **กฎทอง: ใช้ semantic HTML ที่ถูกต้องก่อนเสมอ ใช้ ARIA เมื่อจำเป็น
จริง ๆ เท่านั้น** (ARIA ที่ผิดอาจแย่กว่าไม่มี ARIA เลย)

ตัวอย่างที่นำ `active_link` custom tag จาก Part 029 ขั้อ 286 มาเสริม `aria-current`:

```python
# blog/templatetags/blog_extras.py (ปรับปรุงจาก Part 029 ขั้อ 286)
from django.urls import reverse, NoReverseMatch
from django.utils.html import format_html


@register.simple_tag(takes_context=True)
def nav_link_attrs(context, url_name, css_class="active", **url_kwargs):
    """คืน attribute string พร้อม aria-current เมื่อเป็นหน้าปัจจุบัน"""
    request = context.get('request')
    if request is None:
        return ""
    try:
        matched_url = reverse(url_name, kwargs=url_kwargs)
    except NoReverseMatch:
        return ""
    if request.path == matched_url:
        # ใช้ format_html() แทน mark_safe() ตามหลักความปลอดภัยจาก Part 029 ขั้อ 288
        # เพราะ css_class มาจาก argument ของ tag ไม่ควร mark_safe ค่าที่ยังไม่ escape ตรง ๆ
        return format_html('class="{}" aria-current="page"', css_class)
    return ""
```

```html
<!-- templates/partials/navbar.html -->
<nav class="navbar" aria-label="เมนูหลัก">
    <ul class="navbar__menu">
        {% for link in nav_links %}
            <li><a href="{% url link.url_name %}" {% nav_link_attrs link.url_name %}>{{ link.label }}</a></li>
        {% endfor %}
    </ul>
</nav>
```

`aria-current="page"` บอก screen reader อย่างชัดเจนว่า "ลิงก์นี้คือหน้าที่กำลังเปิด
อยู่" ซึ่ง class CSS ชื่อ `active` เฉย ๆ สื่อความหมายได้แค่กับคนที่มองเห็นสี/สไตล์
เท่านั้น ไม่สื่อความหมายอะไรเลยกับ screen reader

Attribute ARIA ที่ใช้บ่อยอื่น ๆ:

| Attribute | ใช้เมื่อไหร่ |
|---|---|
| `aria-label="..."` | ตั้งชื่อ element ที่ไม่มีข้อความมองเห็นได้ เช่น ปุ่มไอคอนอย่างเดียว |
| `aria-current="page"` | บอกลิงก์ที่ตรงกับหน้าปัจจุบันใน navigation |
| `aria-live="polite"` | บอกว่า element นี้จะมีเนื้อหาเปลี่ยนแปลงแบบไดนามิก (เช่น ข้อความแจ้งเตือนจาก Messages Framework) ให้ screen reader ประกาศให้ผู้ใช้ทราบโดยอัตโนมัติ |
| `aria-describedby="..."` | เชื่อม element กับข้อความอธิบายเพิ่มเติม (มีประโยชน์มากกับ error message ของฟอร์ม — ดูขั้อ 299.4) |
| `role="alert"` | บอกว่า element นี้คือข้อความสำคัญที่ต้องแจ้งผู้ใช้ทันที |

ตัวอย่าง Messages Framework (จาก context processor `messages` ที่ Django ติดตั้ง
มาให้ตาม Part 008 ขั้อ 79.3) กับ `aria-live`:

```html
<!-- templates/partials/messages.html -->
{% if messages %}
    <ul class="messages" aria-live="polite" role="status">
        {% for message in messages %}
            <li class="message message--{{ message.tags }}">{{ message }}</li>
        {% endfor %}
    </ul>
{% endif %}
```

### 299.4 Accessibility ในฟอร์ม: `<label>`, `aria-describedby`, และ Error Message

ทบทวน `CommentForm` จาก Part 025 ขั้อ 250 — Django's default form rendering
(`{{ form.as_p }}`) สร้าง `<label for="...">` ที่จับคู่กับ `id` ของ input ให้อัตโนมัติ
อยู่แล้ว **นี่คือเหตุผลสำคัญที่ไม่ควรเขียน HTML ฟอร์มเองแบบ hardcode โดยไม่ผ่าน
Django Form** เพราะจะพลาดการจับคู่ `label`/`input` ที่ถูกต้องได้ง่ายมาก:

```html
<!-- ❌ อันตราย: label ไม่ได้จับคู่กับ input (ไม่มี for/id ตรงกัน) -->
<label>ชื่อของคุณ</label>
<input type="text" name="author">

<!-- ✅ ถูกต้อง: Django สร้าง id/for ให้ตรงกันอัตโนมัติเมื่อใช้ {{ form.field }} -->
<label for="{{ form.author.id_for_label }}">{{ form.author.label }}</label>
{{ form.author }}
```

เมื่อ `<label>` ไม่ได้จับคู่กับ `<input>` ถูกต้อง ผู้ใช้ screen reader จะไม่ได้ยิน
คำอธิบายของช่องกรอกข้อมูลเลย — เป็นปัญหาที่พบบ่อยมากในเว็บไซต์ที่เขียน HTML ฟอร์ม
เองแทนที่จะใช้ Django Form rendering ที่มีมาให้

จัดการ error message ให้ screen reader อ่านได้ ด้วย `aria-describedby`:

```html
<!-- blog/templates/blog/_comment_form.html -->
<div class="form-field">
    <label for="{{ form.text.id_for_label }}">{{ form.text.label }}</label>
    {% if form.text.errors %}
        <div id="{{ form.text.id_for_label }}-error" class="field-error" role="alert">
            {{ form.text.errors.0 }}
        </div>
    {% endif %}
    <textarea
        name="{{ form.text.html_name }}"
        id="{{ form.text.id_for_label }}"
        {% if form.text.errors %}aria-describedby="{{ form.text.id_for_label }}-error" aria-invalid="true"{% endif %}
    >{{ form.text.value|default:'' }}</textarea>
</div>
```

`aria-describedby` เชื่อม `<textarea>` เข้ากับข้อความ error ทำให้เมื่อ screen
reader focus ที่ช่องกรอกข้อมูลนี้ จะอ่านทั้ง label และข้อความ error ต่อกันให้ผู้ใช้
ฟังทันที โดยไม่ต้องให้ผู้ใช้เดาเองว่า error message ที่อยู่ใกล้ ๆ เกี่ยวข้องกับช่อง
ไหน

### 299.5 รูปภาพและ Alt Text

```html
<!-- ✅ รูปภาพที่มีความหมาย (เช่น รูปหน้าปกบทความ) ต้องมี alt ที่บอกเนื้อหา -->
<img src="{{ post.cover_image.url }}" alt="ภาพประกอบบทความเรื่อง {{ post.title }}">

<!-- ✅ รูปภาพตกแต่งล้วน ๆ ที่ไม่มีความหมายเชิงเนื้อหา ใช้ alt="" (ว่างเปล่าโดยตั้งใจ) -->
<img src="{% static 'blog/img/divider-decoration.svg' %}" alt="">

<!-- ❌ ห้ามลืม alt ไปเลย -->
<img src="{{ post.cover_image.url }}">
```

**กฎสำคัญ**: `alt=""` (ค่าว่างที่ตั้งใจใส่) ≠ ไม่มี `alt` เลย — `alt=""` บอก
screen reader ว่า "รูปนี้ตกแต่งล้วน ๆ ข้ามไปได้เลยไม่ต้องอ่าน" ส่วนการไม่มี `alt`
เลยจะทำให้ screen reader อ่านชื่อไฟล์เต็ม ๆ ออกมาแทน (เช่น
"IMG_20260926_143022.jpg") ซึ่งไม่มีประโยชน์และรบกวนผู้ใช้

### 299.6 การทดสอบ Accessibility เบื้องต้น

| เครื่องมือ | ใช้อย่างไร |
|---|---|
| **Lighthouse** (มีอยู่ใน Chrome DevTools อยู่แล้ว) | เปิด DevTools → แท็บ Lighthouse → รัน audit เลือกหมวด "Accessibility" ได้คะแนนและคำแนะนำทันที |
| **axe DevTools** (browser extension ฟรี) | สแกนหน้าเว็บหา a11y violation ละเอียดกว่า Lighthouse พร้อมลิงก์อธิบายแต่ละปัญหา |
| **ทดสอบด้วยคีย์บอร์ดล้วน** | ลองกด Tab ไล่ทั้งหน้าโดยไม่แตะเมาส์เลย ทุก element ที่คลิกได้ต้อง focus ได้และเห็น focus indicator ชัดเจน |
| **ทดสอบด้วย Screen Reader จริง** | macOS มี VoiceOver ในตัว (Cmd+F5 เปิด/ปิด), Windows มี Narrator — ลองปิดตาแล้วใช้เว็บไซต์ตัวเองด้วยเสียงอย่างเดียว |

การทดสอบ a11y ควรเป็นส่วนหนึ่งของ checklist ก่อน merge ทุก feature ที่มี UI ใหม่
เช่นเดียวกับการเทสต์ฟังก์ชันการทำงาน — เพราะ a11y ที่ถูกละเลยตั้งแต่ต้นมักแก้ไข
ทีหลังได้ยากและแพงกว่าออกแบบให้ถูกต้องตั้งแต่แรกมาก

---

## ขั้นตอนที่ 300: สรุป Phase 3 ทั้งหมด (Part 021-030) และเตรียมตัวสู่ Phase 4

### 300.1 ภาพรวมการเดินทางตลอด Phase 3

Phase 3 พาคุณจากการมีแค่ Function-Based View ธรรมดา ไปสู่เว็บไซต์บล็อกที่ผู้ใช้จริง
โต้ตอบได้ครบวงจร — สร้าง/แก้ไข/ลบบทความผ่านหน้าเว็บ, กรอกฟอร์มคอมเมนต์, เห็นหน้าตา
ที่สม่ำเสมอทุกหน้าด้วย Template Inheritance และ Context Processors:

| Part | หัวข้อหลัก | สิ่งที่ได้เพิ่มเข้าโปรเจกต์ |
|---|---|---|
| 021 | Class-Based Views เบื้องต้น | `as_view()`, `TemplateView`, `RedirectView`, `@method_decorator`, แปลง FBV เป็น CBV |
| 022 | Generic CBV: `ListView`/`DetailView` | `PostListView`, `PostDetailView`, `paginate_by`, `get_queryset()`/`get_context_data()` override |
| 023 | Generic CBV ขั้นสูง: `CreateView`/`UpdateView`/`DeleteView` | CRUD เต็มรูปแบบสำหรับ `Post`, `get_success_url()` แบบ dynamic, `LoginRequiredMixin` เกริ่นก่อนเจาะลึกเต็มใน Phase 4 |
| 024 | Mixins และ CBV กำหนดเอง | `AuthorRequiredMixin`, ลำดับ MRO ของ Mixin, `JsonResponseMixin`, เทสต์ CBV ด้วย `self.client`/`RequestFactory` |
| 025 | Django Forms เบื้องต้น | `forms.Form`, bound/unbound, `clean_<field>()`/`clean()`, widgets, `CommentForm` แบบ manual |
| 026 | ModelForms และ Formsets | `ModelForm` เชื่อมกับ `Post`/`Comment` โดยตรง (`Meta.model`/`fields`), `.save()`/`.save(commit=False)`, `formset_factory`/`inlineformset_factory` สำหรับแก้ไขหลาย object พร้อมกัน |
| 027 | Form Validation ขั้นสูงและ Custom Widgets | Built-in/Custom Validators, Custom Widget subclass, `Widget.Media`, django-crispy-forms/widget-tweaks |
| 028 | Template Inheritance และ Template Tags | ต่อยอด `{% extends %}`/`{% block %}` จาก Part 008 ให้ลึกขึ้นในระดับโปรเจกต์ใหญ่ (multi-level inheritance ระหว่างหลายโซนของเว็บไซต์), การจัดระเบียบ block/tag ให้ scale ได้ |
| 029 | Custom Template Tags และ Filters | `templatetags/blog_extras.py`, `simple_tag`, custom filter, `inclusion_tag`, `takes_context=True`, custom `Node` class, ความปลอดภัยของ `mark_safe`/`format_html` |
| 030 | Context Processors และ Best Practices | หลาย context processor พร้อมกัน, ลำดับความสำคัญ, Jinja2, Debug, Testing template, i18n, a11y |

### 300.2 แผนภาพสถาปัตยกรรมทั้งหมด ณ จุดสิ้นสุด Phase 3

```
                          HTTP Request: GET /posts/django-101/
                                        │
                                        ▼
                              urls.py (blog:detail)
                                        │
                                        ▼
                    ┌───────────────────────────────────┐
                    │   PostDetailView (DetailView)      │  ← Part 021-024
                    │   + LoginRequiredMixin (เกริ่น)     │
                    └──────────────────┬──────────────────┘
                                        │ get_context_data()
                       ┌────────────────┼─────────────────┐
                       ▼                ▼                  ▼
                  Post.objects    CommentForm         (query เพิ่มเติม
                  .get(slug=..)   (ModelForm,          ตามที่ view ต้องการ)
                  (Part 011-024)   Part 026)
                       │                │                  │
                       └────────────────┼──────────────────┘
                                        ▼
                         context = {post, comment_form, ...}
                                        │
                                        ▼
                    render(request, 'blog/detail.html', context)
                                        │
              ┌─────────────────────────┼─────────────────────────┐
              ▼                         ▼                         ▼
   Context Processors ทำงาน    Template Engine โหลด          csrf_token,
   ทุกตัวตามลำดับ (Part 030)    detail.html → extends          request.user
   site_metadata,               base.html (Part 008, 028)      เติมอัตโนมัติ
   navigation_menu,             + include navbar/footer/
   categories_for_sidebar       sidebar (Part 008)
   → dict ถูก update() รวมกัน   + {% render_post_card %}
     (View context ชนะเสมอ      inclusion tag (Part 029)
      ถ้าชื่อชนกัน — ขั้อ 293)  + custom filter/tag อื่น ๆ
              │                         │                         │
              └─────────────────────────┼─────────────────────────┘
                                        ▼
                              HTML String สมบูรณ์
                                        │
                                        ▼
                         HttpResponse กลับไปยัง Browser
```

### 300.3 Quiz ทบทวน Phase 3 (พร้อมเฉลย)

พยายามตอบเองก่อนเลื่อนไปดูเฉลยแต่ละข้อ

**คำถามที่ 1**: `as_view()` ทำหน้าที่อะไร และทำไม Django ถึงต้องมี method นี้แทนที่
จะส่ง class ตรง ๆ เข้า `urlpatterns` ได้เลย?

> **เฉลย**: `as_view()` เป็น classmethod ที่คืนค่าเป็น**ฟังก์ชัน** (ไม่ใช่ class)
> เพราะ Django's URL resolver ต้องการฟังก์ชันที่เรียกได้ (callable ที่รับ
> `request` แล้วคืน `HttpResponse`) ฟังก์ชันที่ `as_view()` สร้างขึ้นจะสร้าง
> instance ใหม่ของ class นั้นทุกครั้งที่มี request เข้ามา แล้วเรียก `dispatch()`
> ของ instance นั้นให้ (ทบทวน Part 021 ขั้อ 202)

**คำถามที่ 2**: `AuthorRequiredMixin` ต้องวางไว้ก่อนหรือหลัง `UpdateView` ใน
`class PostUpdateView(LoginRequiredMixin, AuthorRequiredMixin, UpdateView)` และ
เพราะอะไร?

> **เฉลย**: ต้องวาง Mixin ทั้งหมด**ก่อน** generic view class เสมอ (ซ้ายไปขวา คือ
> ลำดับที่ Python ค้นหา method ตาม MRO) เพราะ Mixin ต้องได้ "แทรก" ตัวเองเข้าไป
> ในสาย `dispatch()`/`get()`/`post()` ก่อนที่ logic หลักของ generic view จะทำงาน
> ถ้าวางสลับกัน (`UpdateView` มาก่อน) MRO จะเรียก method ของ `UpdateView` ก่อน
> ทำให้ permission check ของ Mixin ไม่ทำงานตามที่ตั้งใจ (Part 024 ขั้อ 233)

**คำถามที่ 3**: ทำไม `ModelForm` (Part 026) ถึงลดโค้ดได้มากกว่า `forms.Form`
ธรรมดา (Part 025) เมื่อต้องสร้าง object ใหม่ลงฐานข้อมูล?

> **เฉลย**: `ModelForm` สร้าง field ทั้งหมดอัตโนมัติจาก `Meta.model` และ
> `Meta.fields` โดยไม่ต้องประกาศ field ซ้ำ (สอดคล้องหลัก DRY) และมี method
> `.save()` ที่สร้าง/อัปเดต object ลงฐานข้อมูลให้ในบรรทัดเดียว ต่างจาก `forms.Form`
> ที่ต้องดึงค่าจาก `cleaned_data` มาสร้าง object เองทีละ field ด้วยมือ (ตามที่ทำใน
> `CommentForm` ของ Part 025 ขั้อ 250)

**คำถามที่ 4**: `Formset` ใน Part 026 แก้ปัญหาอะไร ที่ `ModelForm` เดี่ยว ๆ
แก้ไม่ได้?

> **เฉลย**: `ModelForm` จัดการได้แค่ object เดียวต่อฟอร์มหนึ่งชุด ส่วน
> **Formset** ช่วยให้จัดการฟอร์มชนิดเดียวกันหลาย ๆ ชุดพร้อมกันในหน้าเดียว
> (เช่น เพิ่ม `Tag` ให้ `Post` พร้อมกันหลายรายการในฟอร์มเดียว) โดยเฉพาะ
> `inlineformset_factory` ที่ผูกกับความสัมพันธ์ ForeignKey ได้โดยตรง (เช่น
> แก้ไข `Comment` หลายรายการของ `Post` เดียวพร้อมกันในหน้าเดียว)

**คำถามที่ 5**: Custom Widget (Part 027) ต่างจาก Custom Form Field อย่างไร?

> **เฉลย**: **Field** รับผิดชอบเรื่อง validation และการแปลงข้อมูล (Python
> type ↔ string ที่ผู้ใช้กรอก) ส่วน **Widget** รับผิดชอบแค่ "หน้าตา HTML"
> ของ input element เท่านั้น ฟิลด์หนึ่งตัวสามารถใช้ widget ต่างกันได้โดยไม่กระทบ
> logic การ validate เช่น `DateField` ใช้ได้ทั้ง `TextInput` หรือ `SelectDateWidget`
> โดย validation ยังทำงานเหมือนเดิม (Part 027 ขั้อ 267)

**คำถามที่ 6**: `{% include %}` กับ `{% extends %}` ต่างกันอย่างไรในเชิงทิศทาง
ความสัมพันธ์?

> **เฉลย**: `{% extends %}` คือ "เทมเพลตลูกสืบทอดจากเทมเพลตแม่" ใช้ได้ครั้งเดียว
> ต่อไฟล์และต้องเป็น tag แรกสุด ส่วน `{% include %}` คือ "ดึงชิ้นส่วนมาแปะ" ใช้ได้
> หลายครั้งตรงไหนก็ได้ในไฟล์ (Part 008 ขั้อ 76.1)

**คำถามที่ 7**: `inclusion_tag` (Part 029) ต่างจาก `{% include %}` (Part 008)
อย่างไร ควรเลือกใช้อันไหนเมื่อไหร่?

> **เฉลย**: `{% include %}` เหมาะกับกรณีง่าย ๆ ที่รับ context จากหน้าที่เรียกอยู่
> แล้ว ไม่มี logic คำนวณเพิ่ม ส่วน `inclusion_tag` เหมาะเมื่อ partial นั้นต้อง
> **คำนวณ/query ข้อมูลเพิ่มเติมเอง** ก่อนแสดงผล (เช่น `render_post_card` ที่รับ
> แค่ `post` object แล้วไปคำนวณ reading time เองข้างใน) (Part 008 ขั้อ 80.5
> FAQ, Part 029 ขั้อ 284)

**คำถามที่ 8**: ถ้า context processor สองตัวคืนค่า key ชื่อเดียวกัน ตัวไหนชนะ
และถ้า View เองก็ส่ง context variable ชื่อเดียวกันมาด้วย ใครชนะสุดท้าย?

> **เฉลย**: ระหว่าง context processor ด้วยกันเอง **ตัวที่อยู่ทีหลังในลิสต์ของ
> `TEMPLATES.OPTIONS.context_processors` ชนะ** (เพราะกลไก `dict.update()` ไล่
> ทับกันตามลำดับ) แต่ไม่ว่า context processor ไหนจะชนะกันเอง **ค่าที่ View ส่งมา
> ตรง ๆ ผ่าน context dict ของ `render()`/`get_context_data()` จะชนะเสมอเป็น
> อันดับสุดท้าย** เพราะ Django ออกแบบให้ค่าที่เจาะจงกว่า (มาจาก View) override
> ค่าทั่วไป (มาจาก context processor) เสมอ (Part 030 ขั้อ 293)

**คำถามที่ 9**: ทำไม Jinja2 backend ของ Django ถึงมองหา template ในโฟลเดอร์ชื่อ
`jinja2/` แทนที่จะเป็น `templates/` เหมือน DTL?

> **เฉลย**: เพื่อไม่ให้ไฟล์ของสอง backend ปะปนกันในโฟลเดอร์เดียว ถ้าทั้งสอง
> engine ค้นหาไฟล์ชื่อเดียวกันในโฟลเดอร์เดียวกัน Django จะไม่มีทางรู้ว่าไฟล์นั้น
> ควรถูก parse ด้วย syntax ของ engine ไหน การแยกชื่อโฟลเดอร์ (`app_dirname`
> ของแต่ละ backend) จึงทำให้ทั้งสอง engine ทำงานร่วมกันในโปรเจกต์เดียวได้อย่าง
> ชัดเจนไม่ชนกัน (Part 030 ขั้อ 294.2)

**คำถามที่ 10**: ทำไม `{% debug %}` ต้องครอบด้วย `{% if debug %}` เสมอ ห้ามวาง
ตรง ๆ โดยไม่มีเงื่อนไข?

> **เฉลย**: `{% debug %}` แสดงข้อมูล context ทั้งหมดอย่างละเอียด ซึ่งอาจมีข้อมูล
> sensitive ปนอยู่ (เช่น session data, ค่าจาก settings) ตัวแปร `debug` (จาก
> `django.template.context_processors.debug`) จะเป็น `True` ก็ต่อเมื่อ
> `settings.DEBUG = True` และ IP อยู่ใน `INTERNAL_IPS` เท่านั้น การครอบด้วย
> `{% if debug %}` จึงรับประกันว่าข้อมูลนี้จะไม่มีวันหลุดไปแสดงใน production
> โดยไม่ได้ตั้งใจ (Part 030 ขั้อ 296.1)

**คำถามที่ 11**: `assertContains(response, text, html=True)` ต่างจากการไม่ใส่
`html=True` อย่างไร?

> **เฉลย**: เมื่อใส่ `html=True` Django จะ parse ทั้ง response และ `text` ที่
> ให้มาเป็นโครงสร้าง HTML ก่อนเปรียบเทียบ ทำให้การเทียบทนทานต่อความต่างที่ไม่มี
> นัยสำคัญ เช่น ลำดับของ attribute หรือช่องว่างระหว่าง tag ต่างกัน ถ้าไม่ใส่
> `html=True` Django จะเทียบแบบ substring string ธรรมดา ซึ่งเปราะบางกว่ามาก
> ต่อการเปลี่ยนแปลงเล็กน้อยของ HTML ที่ไม่กระทบความหมาย (Part 030 ขั้อ 297.3)

**คำถามที่ 12 (โบนัส)**: `alt=""` กับการไม่ใส่ `alt` เลยบน `<img>` ต่างกัน
อย่างไรสำหรับผู้ใช้ screen reader?

> **เฉลย**: `alt=""` (ค่าว่างที่ตั้งใจใส่) บอก screen reader อย่างชัดเจนว่า
> "รูปนี้เป็นการตกแต่งล้วน ๆ ไม่มีความหมายเชิงเนื้อหา ข้ามไปได้เลย" ส่วนการไม่ใส่
> `alt` เลยจะทำให้ screen reader หาข้อมูลอื่นมาอ่านแทน (มักจะอ่านชื่อไฟล์เต็ม ๆ
> เช่น "IMG_20260926.jpg") ซึ่งไม่มีประโยชน์และรบกวนประสบการณ์ผู้ใช้ (Part 030
> ขั้อ 299.5)

### 300.4 แบบฝึกหัดใหญ่ปิดท้าย Phase 3: ระบบบล็อกที่มี CRUD ครบวงจร

แบบฝึกหัดนี้รวบยอดทุกหัวข้อของ Phase 3 เข้าด้วยกัน ให้ทำตามลำดับขั้นตอนต่อไปนี้บน
โปรเจกต์บล็อกของคุณเอง

**ส่วนที่ 1 — Full CRUD สำหรับ `Post` ผ่าน Class-Based Views**

1. สร้าง `PostListView`, `PostDetailView`, `PostCreateView`, `PostUpdateView`,
   `PostDeleteView` ครบทั้ง 5 ตัวใน `blog/views.py` โดยใช้เทคนิคจาก Part 021-024:
   - `PostCreateView`/`PostUpdateView`: ใช้ `LoginRequiredMixin` ป้องกัน
     anonymous user, ใช้ `AuthorRequiredMixin` (เขียนเองตาม Part 024 ขั้อ 232)
     ป้องกันไม่ให้ผู้ใช้แก้ไข/ลบบทความของคนอื่น
   - `PostCreateView`: override `form_valid()` เพื่อผูก `self.request.user`
     เข้ากับ `form.instance.author` ก่อนบันทึก (Part 023 ขั้อ 225)
   - `PostListView`: เพิ่ม `paginate_by = 10` และรองรับกรองตาม query parameter
     `?category=<slug>`
2. เขียน `blog/urls.py` เชื่อมทั้ง 5 View ด้วย `app_name = 'blog'` และตั้งชื่อ URL
   pattern ให้สื่อความหมาย (`list`, `detail`, `post-create`, `post-update`,
   `post-delete`)
3. สร้าง template ครบทุกหน้า (`list.html`, `detail.html`, `post_form.html`
   ใช้ร่วมกันทั้ง create/update ตามธรรมเนียมของ `CreateView`/`UpdateView`,
   `post_confirm_delete.html`) โดย**ทุกหน้าต้อง extends จาก `base.html`**

**ส่วนที่ 2 — ฟอร์มคอมเมนต์แบบ `ModelForm`**

4. เขียน `CommentForm` เป็น `ModelForm` (ตามที่เรียนใน Part 026) แทนที่
   `forms.Form` แบบ manual จาก Part 025:

   ```python
   # blog/forms.py
   from django import forms
   from .models import Comment

   BANNED_WORDS = ["สแปม", "โฆษณา", "คลิกที่นี่"]


   class CommentForm(forms.ModelForm):
       class Meta:
           model = Comment
           fields = ['author_name', 'text']
           widgets = {
               'author_name': forms.TextInput(attrs={
                   'class': 'form-control',
                   'placeholder': 'ชื่อที่จะแสดงบนความคิดเห็น',
               }),
               'text': forms.Textarea(attrs={
                   'class': 'form-control', 'rows': 4,
                   'placeholder': 'แสดงความคิดเห็นของคุณ...',
               }),
           }

       def clean_text(self):
           text = self.cleaned_data['text']
           if len(text.strip()) < 10:
               raise forms.ValidationError(
                   "ความคิดเห็นสั้นเกินไป กรุณาเขียนอย่างน้อย 10 ตัวอักษร"
               )
           lowered = text.lower()
           for word in BANNED_WORDS:
               if word in lowered:
                   raise forms.ValidationError(f'ข้อความมีคำที่ไม่อนุญาต: "{word}"')
           return text.strip()
   ```

5. เขียน View ที่รับฟอร์มนี้ในหน้า `PostDetailView` (ผสม `SingleObjectMixin` +
   `FormMixin` ตามที่เกริ่นไว้ใน Part 023 ขั้อ 228 หรือเขียนเป็น View แยกสำหรับ
   POST ก็ได้) โดยใช้ `form.save(commit=False)` เพื่อผูก `post` เข้ากับ comment
   ก่อนบันทึกจริง:

   ```python
   def form_valid(self, form):
       comment = form.save(commit=False)
       comment.post = self.object   # หรือ get_object_or_404(Post, slug=...)
       comment.save()
       return redirect(comment.post.get_absolute_url())
   ```

6. เปรียบเทียบจำนวนบรรทัดโค้ดกับ `CommentForm` แบบ `forms.Form` เดิมจาก Part 025
   ขั้อ 250 แล้วบันทึกลง `notes.md` ว่า `ModelForm` ลดโค้ดไปกี่บรรทัด

**ส่วนที่ 3 — Custom Template Tag อย่างน้อย 1 ตัว**

7. เพิ่ม custom template tag ใหม่ชื่อ `comment_count_badge` ใน
   `blog/templatetags/blog_extras.py` (`simple_tag` หรือ `inclusion_tag` ก็ได้)
   ที่รับ `post` แล้วแสดงจำนวนคอมเมนต์เป็น badge (เช่น "5 ความคิดเห็น") นำไปแสดง
   ทั้งใน `list.html` (การ์ดแต่ละบทความ) และ `detail.html`

**ส่วนที่ 4 — Context Processor และ Best Practices**

8. เพิ่ม context processor `categories_for_sidebar` ตามขั้อ 292 ของ Part นี้
   เข้าไปในโปรเจกต์จริง พร้อม cache ตามขั้อ 292.4
9. รัน checklist ทั้งหมดจากขั้อ 295.4 กับทุก template ที่สร้างขึ้นในแบบฝึกหัดนี้
   แก้ไขจนผ่านครบทุกข้อ
10. เพิ่ม accessibility ตามขั้อ 299: skip link, semantic landmark, `label for`
    ที่ถูกต้องในฟอร์มคอมเมนต์, `aria-current="page"` บน navbar

**ส่วนที่ 5 — เขียนเทสต์ครอบคลุม**

11. เขียนเทสต์ครบทั้ง 3 ระดับตามขั้อ 297: unit test สำหรับ context processor
    (297.1), unit test สำหรับ partial แยก (297.4), และ integration test ผ่าน
    `self.client` สำหรับ CRUD ทั้ง 5 View รวมถึงการโพสต์คอมเมนต์ (297.2-297.5)
12. ยืนยันว่าผู้ใช้ที่ไม่ใช่เจ้าของบทความไม่สามารถแก้ไข/ลบบทความของคนอื่นได้
    (`AuthorRequiredMixin`) ด้วยเทสต์ที่คาดหวัง `403 Forbidden`

**เกณฑ์ความสำเร็จของแบบฝึกหัด**:

- [ ] เปิด `/posts/` เห็นรายการบทความพร้อม pagination, sidebar หมวดหมู่,
      และ badge จำนวนคอมเมนต์ในแต่ละการ์ด
- [ ] ล็อกอินแล้วกด "เขียนบทความใหม่" กรอกฟอร์มสำเร็จ แล้ว redirect ไปหน้า
      detail ของบทความที่เพิ่งสร้าง
- [ ] ลองแก้ไข/ลบบทความของผู้ใช้อื่นด้วยบัญชีที่ไม่ใช่เจ้าของ ต้องได้
      `403 Forbidden`
- [ ] กรอกฟอร์มคอมเมนต์ในหน้า detail สำเร็จ เห็นคอมเมนต์ใหม่ปรากฏทันทีพร้อม
      badge จำนวนคอมเมนต์ที่อัปเดต
- [ ] เปิด Debug Toolbar → Templates panel เห็น context processors ทั้งหมด
      ทำงานถูกต้องตามลำดับที่ตั้งไว้
- [ ] ทดสอบด้วยคีย์บอร์ดล้วน (ไม่ใช้เมาส์) กด Tab ผ่าน skip link → navbar →
      เนื้อหาหลัก → ฟอร์มคอมเมนต์ได้ครบทุกจุดที่คลิกได้
- [ ] `python manage.py test blog` ผ่านทั้งหมดโดยไม่มี error

### 300.5 Checklist รวม Phase 3 — สิ่งที่คุณควรทำได้แล้วตอนนี้

- [ ] เขียน Class-Based View ตั้งแต่เบื้องต้น (`TemplateView`) ไปจนถึง Generic
      CBV ครบชุด (`ListView`/`DetailView`/`CreateView`/`UpdateView`/`DeleteView`)
- [ ] เข้าใจ MRO และเขียน Mixin ของตัวเองที่ทำงานถูกลำดับ
- [ ] เขียน Django Form ทั้งแบบ `forms.Form` และ `ModelForm` พร้อม validation
      ระดับ field และระดับฟอร์มทั้งหมด
- [ ] เขียน Formset สำหรับจัดการหลาย object พร้อมกันในฟอร์มเดียว
- [ ] สร้าง Custom Widget และปรับแต่ง Widget ที่มีอยู่แล้ว
- [ ] สร้าง Template Inheritance หลายชั้นและ Partial ที่ reuse ได้ทั่วทั้งเว็บไซต์
- [ ] เขียน Custom Template Tag ครบทั้ง 3 แบบ (simple, filter, inclusion) และ
      custom `Node` สำหรับกรณีขั้นสูง
- [ ] เขียนและจัดการ Context Processor หลายตัวพร้อมกันโดยไม่ให้ชื่อตัวแปรชนกัน
- [ ] เข้าใจ Jinja2 เป็นทางเลือกและอ่านโค้ด Jinja2 ของคนอื่นออก
- [ ] Debug template ได้อย่างเป็นระบบด้วย error message, `{% debug %}`, และ
      Debug Toolbar
- [ ] เขียนเทสต์ครอบคลุมทั้ง View, Template, และ Context Processor
- [ ] รู้จักพื้นฐาน i18n และนำไปใช้ได้เมื่อโปรเจกต์จริงต้องการ
- [ ] ออกแบบ template ที่คำนึงถึง Accessibility ตั้งแต่ semantic HTML,
      ARIA attributes, ไปจนถึงฟอร์มที่ screen reader ใช้งานได้จริง

หากทำเครื่องหมายได้ครบทุกข้อ แปลว่าคุณมีทักษะด้าน **Views, Templates และ Forms
ระดับมืออาชีพ** ที่พร้อมรับมือกับระบบผู้ใช้จริงใน Phase ถัดไป

### 300.6 คำถามที่พบบ่อยส่งท้าย Phase 3

**Q: จำเป็นต้องใช้ Class-Based View ตลอดหรือ Function-Based View ยังมีที่ยืนอยู่
ไหม?**
A: ทั้งสองแบบยังมีที่ยืนแน่นอน (ทบทวนกรอบการตัดสินใจจาก Part 024 ขั้อ 239) — CBV
เหมาะกับ CRUD มาตรฐานที่ Generic View ครอบคลุมอยู่แล้ว (ประหยัดโค้ดมาก) ส่วน FBV
ยังเหมาะกับ logic ที่ซับซ้อนเฉพาะทางมาก ๆ ที่การพยายามยัดเข้า Generic CBV กลับทำให้
โค้ดอ่านยากกว่าเขียน FBV ตรง ๆ ทีมมืออาชีพส่วนใหญ่ใช้ทั้งสองแบบผสมกันในโปรเจกต์
เดียวตามความเหมาะสมของแต่ละ View

**Q: ควรใช้ Context Processor หรือใส่ตัวแปรผ่าน Middleware แทน?**
A: ทั้งสองมีจุดประสงค์ต่างกัน Context Processor มีไว้เติมตัวแปรเข้า **template
context** เท่านั้น (ใช้ได้แค่ตอน render template) ส่วน Middleware ทำงานกับ
**request/response cycle ทั้งหมด** (แก้ไข request ก่อนถึง view, หรือแก้ไข
response หลัง view ทำงานเสร็จ ไม่ว่า view นั้นจะ render template หรือคืน JSON
ก็ตาม) ถ้าข้อมูลนั้นต้องใช้ใน template เท่านั้น ใช้ Context Processor ถ้าต้องการ
แก้ไขพฤติกรรมของทุก request/response (เช่น เพิ่ม header, ตรวจสอบสิทธิ์ก่อนถึง
view) ใช้ Middleware

**Q: ทำไม Django ไม่ใส่ระบบ caching ให้ context processor อัตโนมัติเลยตั้งแต่
ต้น เพื่อแก้ปัญหา performance ในขั้อ 292?**
A: เพราะ Django ออกแบบให้นักพัฒนาเป็นคนตัดสินใจเองว่าข้อมูลไหนควร cache นานแค่ไหน
(บาง context processor เช่น `auth` ต้องเป็นข้อมูลสดใหม่ทุก request ห้าม cache
เด็ดขาด เพราะเกี่ยวข้องกับ security) การให้นักพัฒนาเลือกเองผ่าน Caching Framework
(Part 068-069) จึงยืดหยุ่นและปลอดภัยกว่าการ cache อัตโนมัติแบบเหมาว่าทุกอย่าง
cache ได้เหมือนกันหมด

**Q: ควรเรียน Jinja2 ให้ลึกแค่ไหน ถ้าหลักสูตรนี้จะใช้ DTL ตลอด?**
A: แค่ระดับที่อ่านโค้ดคนอื่นออกก็เพียงพอสำหรับตอนนี้ (ตามที่ Part นี้สอน) ถ้าใน
อนาคตคุณเข้าทำงานในทีมที่ใช้ Jinja2 จริง ค่อยศึกษาเชิงลึกตอนนั้น หลักการ MTV,
CBV, Forms, Context Processors ที่เรียนมาทั้งหมดใน Phase 3 ใช้ได้เหมือนกันทุก
ประการไม่ว่าจะใช้ template backend ไหน — สิ่งที่เปลี่ยนมีแค่ syntax ของไฟล์
`.html` เท่านั้น

**Q: Accessibility สำคัญจริงหรือแค่ "nice to have" สำหรับโปรเจกต์เล็ก ๆ?**
A: สำหรับโปรเจกต์ personal เล็ก ๆ ที่ไม่มีใครใช้จริงจัง อาจไม่ใช่สิ่งเร่งด่วน
ที่สุด แต่ **นิสัยการเขียน semantic HTML และ label ฟอร์มให้ถูกต้องควรติดตัวไว้
ตั้งแต่ต้น** เพราะเปลี่ยนพฤติกรรมทีหลังยากกว่าทำถูกตั้งแต่แรกมาก และเมื่อ
โปรเจกต์เติบโตหรือเข้าทำงานกับองค์กรที่มีข้อกำหนดด้าน accessibility (ภาครัฐ,
สถาบันการศึกษา, องค์กรขนาดใหญ่จำนวนมาก) นิสัยนี้จะทำให้คุณปรับตัวได้เร็วกว่า
เพื่อนร่วมทีมที่ไม่เคยฝึกมาก่อนมาก

---

## เตรียมตัวสู่ Phase 4: Authentication, Users และ Permissions

Phase 3 ปิดฉากลงแล้วด้วยเว็บไซต์บล็อกที่ใช้งานได้จริงครบวงจร — ผู้ใช้เห็นหน้ารายการ
บทความ, กดเข้าไปอ่านรายละเอียด, เขียน/แก้ไข/ลบบทความผ่าน CBV, กรอกฟอร์มคอมเมนต์
ผ่าน ModelForm, และทุกหน้ามีดีไซน์สม่ำเสมอด้วย Template Inheritance และ Context
Processors

แต่สังเกตว่าตลอด Phase 3 เรา**สมมติ**ว่ามีระบบผู้ใช้ (`request.user`,
`LoginRequiredMixin`, `Post.author`) อยู่แล้วโดยยังไม่เคยเจาะลึกว่าระบบนี้ทำงาน
อย่างไรจริง ๆ เบื้องหลัง — ผู้ใช้ยังสมัครสมาชิกเองไม่ได้, ยังไม่มีหน้า login/logout
จริง, ยังไม่มีระบบสิทธิ์ที่แยกแยะว่าใครทำอะไรได้บ้างนอกเหนือจาก "เป็นเจ้าของหรือไม่"

**Phase 4: Authentication, Users และ Permissions (Part 031-038 | Step 301-380)**
จะพาคุณเจาะลึกระบบผู้ใช้ของ Django อย่างเต็มรูปแบบ:

- **Part 031**: Django Authentication System เบื้องต้น — หน้า login/logout/
  signup จริง, `django.contrib.auth`, `authenticate()`/`login()`/`logout()`
- **Part 032**: Custom User Model — เหตุผลที่ควรทำตั้งแต่ต้นโปรเจกต์ (แม้ยัง
  ไม่ต้องการฟิลด์เพิ่มก็ตาม) และวิธี migrate ปลอดภัยถ้าต้องทำทีหลัง
- **Part 033**: Permissions และ Groups — ระบบสิทธิ์ที่ละเอียดกว่า "เป็นเจ้าของ
  หรือไม่" ที่ `AuthorRequiredMixin` ทำได้ในตอนนี้
- **Part 034**: Django Sessions และ Cookies เจาะลึก
- **Part 035**: Password Management, Reset และ Security
- **Part 036**: Social Authentication (OAuth, django-allauth)
- **Part 037**: Two-Factor Authentication
- **Part 038**: Row-Level Permissions และ Object-Level Permission (เวอร์ชัน
  เต็มรูปแบบของแนวคิดที่ `AuthorRequiredMixin` ทำแบบง่าย ๆ ไว้ใน Part 024)

เมื่อจบ Phase 4 บล็อกของคุณจะมีระบบสมาชิกที่สมบูรณ์ — ผู้ใช้สมัครสมาชิกได้เอง,
ล็อกอิน/ล็อกเอาต์ได้จริงผ่านหน้าเว็บ, มีระบบสิทธิ์ที่ยืดหยุ่นกว่าเดิมมาก, และปลอดภัย
ตามมาตรฐานที่แอปพลิเคชันระดับ production ต้องมี

เตรียมเปิด VS Code, Terminal, และรัน `python manage.py runserver` ทิ้งไว้ให้พร้อม
เพราะ Part 031 เราจะสร้างหน้า login จริงหน้าแรกของหลักสูตรกันทันที!
