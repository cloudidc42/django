# Part 044: ViewSets และ Routers

> **ขั้นตอนที่ 431-440 ของหลักสูตร** | Phase 5: Django REST Framework และ API
>
> เป้าหมายของ Part นี้: เข้าใจแนวคิด `ViewSet` ของ Django REST Framework ที่ผูกกับ
> **action** (`list`, `create`, `retrieve`, `update`, `partial_update`, `destroy`) แทนที่จะ
> ผูกกับ HTTP method ตรง ๆ แบบ `APIView`/Generic Views ที่เรียนมาใน Part 042-043 คุณจะ
> เขียน `ModelViewSet` เต็มรูปแบบ, ให้ `Router` (`SimpleRouter`/`DefaultRouter`) สร้าง URL
> ให้อัตโนมัติแทนการเขียน `path()` ทีละเส้นทาง, เพิ่ม custom action ด้วย `@action`, ทำ
> nested resource (`/posts/{id}/comments/`) ด้วย `drf-nested-routers`, ใช้
> `ReadOnlyModelViewSet` สำหรับ endpoint อ่านอย่างเดียว, override `get_queryset()` และ
> `get_permissions()` แยกตาม action ภายในคลาสเดียว และปิดท้ายด้วยการประกอบ Blog API
> เต็มรูปแบบสำหรับ `Post`, `Category`, `Comment` ด้วย ViewSet + Router ล้วน ๆ

---

## สารบัญของ Part นี้

- ขั้นตอนที่ 431: แนวคิด `ViewSet` เทียบกับ `APIView` — ทำไม DRF แยกสองแนวคิดนี้
- ขั้นตอนที่ 432: `ModelViewSet` — CRUD เต็มรูปแบบในคลาสเดียว พร้อม Router สร้าง URL ให้อัตโนมัติ
- ขั้นตอนที่ 433: Router — `DefaultRouter` vs `SimpleRouter` เปรียบเทียบ
- ขั้นตอนที่ 434: `@action` decorator — เพิ่ม custom action บน ViewSet
- ขั้นตอนที่ 435: Nested Router ด้วย `drf-nested-routers`
- ขั้นตอนที่ 436: `ReadOnlyModelViewSet` — สำหรับ endpoint ที่ให้แค่อ่าน
- ขั้นตอนที่ 437: Override `get_queryset()` แยกตาม action ภายใน ViewSet เดียว
- ขั้นตอนที่ 438: ตารางเปรียบเทียบ ViewSet vs Generic View — เมื่อไหร่ควรเลือกอะไร
- ขั้นตอนที่ 439: ผสาน permission ที่ต่างกันต่อ action ด้วย `get_permissions()`
- ขั้นตอนที่ 440: สรุปและแบบฝึกหัด — สร้าง Blog API เต็มรูปแบบด้วย ViewSet + Router

---

## ขั้นตอนที่ 431: แนวคิด `ViewSet` เทียบกับ `APIView` — ทำไม DRF แยกสองแนวคิดนี้

### 431.1 ทบทวนเส้นทางที่พาเรามาถึงจุดนี้

- **Part 042**: เขียน `APIView` เต็มรูปแบบ — `PostListAPIView` (GET/POST) และ
  `PostDetailAPIView` (GET/PUT/PATCH/DELETE) แยกกันคนละคลาส เจาะกลไก `dispatch()`
- **Part 043**: ย่อโค้ดข้างต้นด้วย Generic Views และ Mixin (`ListCreateAPIView`,
  `RetrieveUpdateDestroyAPIView`) — สั้นลงมาก แต่ **ยังคงต้องมี 2 คลาสแยกกัน** และยังต้อง
  เขียน `path()` เองทุกเส้นทางใน `urls.py`

Part นี้จะตั้งคำถามต่อ: ถ้า `Post` หนึ่ง resource ต้องมีทั้ง list, create, retrieve,
update, partial_update, destroy ครบ 6 การกระทำ ทำไมต้องเขียนแยกเป็น 2 คลาส (`...List...`
กับ `...Detail...`) ทั้งที่มันคือ resource เดียวกัน แชร์ `queryset`, `serializer_class`,
`permission_classes` เหมือนกันทุกประการ? นี่คือช่องว่างที่ **`ViewSet`** เข้ามาอุด

### 431.2 ปัญหาซ้ำซ้อนของ Generic Views เมื่อ resource เดียวมีหลาย endpoint

ทบทวนโค้ด Generic Views จาก Part 043 (ย่อ):

```python
# blog/api_views.py (แบบ Part 043 — ยังต้องแยก 2 คลาส)
from rest_framework import generics
from .models import Post
from .serializers import PostSerializer


class PostListCreateAPIView(generics.ListCreateAPIView):
    queryset = Post.objects.all()
    serializer_class = PostSerializer

    def perform_create(self, serializer):
        serializer.save(author=self.request.user)


class PostRetrieveUpdateDestroyAPIView(generics.RetrieveUpdateDestroyAPIView):
    queryset = Post.objects.all()
    serializer_class = PostSerializer
    lookup_field = 'slug'
```

```python
# blog/api_urls.py (แบบ Part 043)
from django.urls import path
from . import api_views

urlpatterns = [
    path('posts/', api_views.PostListCreateAPIView.as_view(), name='post-list'),
    path('posts/<slug:slug>/', api_views.PostRetrieveUpdateDestroyAPIView.as_view(),
         name='post-detail'),
]
```

สังเกตความซ้ำซ้อน: `queryset = Post.objects.all()` และ `serializer_class = PostSerializer`
ถูกเขียนซ้ำ **สองครั้ง** ในสองคลาส ถ้าวันหนึ่งต้องเปลี่ยน queryset (เช่น กรอง
`is_published=True`) ต้องแก้ทั้งสองที่ และถ้าลืมแก้ที่ใดที่หนึ่งจะเกิดพฤติกรรมไม่ตรงกัน
แบบเงียบ ๆ — ปัญหานี้ยิ่งชัดขึ้นเมื่อมี resource จำนวนมากในโปรเจกต์จริง (สิบ, ยี่สิบ, หรือ
เป็นร้อย model ที่ต้องมี CRUD API)

### 431.3 แนวคิดหลักของ `ViewSet`: ผูกกับ "action" ไม่ใช่ HTTP method

`APIView` (และ Generic Views ที่สืบทอดจากมัน) นิยาม behavior ผ่าน **method ที่ชื่อตรงกับ
HTTP verb**: `get()`, `post()`, `put()`, `patch()`, `delete()` — 1 คลาส 1 URL pattern

`ViewSet` กลับด้าน: มันนิยาม behavior ผ่าน **method ที่ชื่อตรงกับ "action" เชิงความหมาย
ของ REST resource** — `list()`, `create()`, `retrieve()`, `update()`, `partial_update()`,
`destroy()` — **1 คลาสเดียวครอบคลุมทั้ง list URL และ detail URL** โดยให้ตัวช่วยภายนอก
(Router) เป็นคนแปลง action เหล่านี้กลับไปเป็น HTTP method + URL pattern อีกที

```
APIView (Part 042)                     ViewSet (Part นี้)
─────────────────────                  ─────────────────────
1 คลาส = 1 URL                         1 คลาส = ทุก URL ของ resource เดียว
method ชื่อ = HTTP verb                 method ชื่อ = action เชิงความหมาย
GET  /posts/      -> get()             GET    /posts/      -> list()
POST /posts/      -> post()            POST   /posts/      -> create()
GET  /posts/{id}/ -> get() (คนละคลาส)  GET    /posts/{id}/ -> retrieve()
PUT  /posts/{id}/ -> put()  (คนละคลาส) PUT    /posts/{id}/ -> update()
                                        PATCH  /posts/{id}/ -> partial_update()
                                        DELETE /posts/{id}/ -> destroy()
```

### 431.4 ตาราง Mapping: HTTP Method + URL Pattern → Action Name

| HTTP Method | URL Pattern | Action ที่ ViewSet เรียก | เทียบเท่า Generic View (Part 043) |
|---|---|---|---|
| `GET` | `/posts/` | `list()` | `ListAPIView` |
| `POST` | `/posts/` | `create()` | `CreateAPIView` |
| `GET` | `/posts/{pk}/` | `retrieve()` | `RetrieveAPIView` |
| `PUT` | `/posts/{pk}/` | `update()` | `UpdateAPIView` |
| `PATCH` | `/posts/{pk}/` | `partial_update()` | `UpdateAPIView` (partial) |
| `DELETE` | `/posts/{pk}/` | `destroy()` | `DestroyAPIView` |

ตารางนี้คือหัวใจของทั้ง Part — ทุกอย่างที่เรียนต่อจากนี้จะอ้างอิงกลับมาที่การ "แปลง" ระหว่าง
คอลัมน์ซ้าย (HTTP) กับคอลัมน์กลาง (action) ตลอดเวลา

### 431.5 `ViewSet` ไม่ได้เรียก `.as_view()` แบบตรง ๆ

ความแตกต่างเชิงกลไกที่สำคัญที่สุด: `View.as_view()` และ `APIView.as_view()` (ทบทวนจาก
Part 021 และ 042) รับ **ไม่มี argument พิเศษ** เพราะมันรู้อยู่แล้วว่า `get()` คู่กับ
`GET`, `post()` คู่กับ `POST` เสมอแบบตายตัว

แต่ `ViewSet.as_view()` **ต้องรับ dict ที่ map HTTP method ไปยังชื่อ action เอง** เพราะไม่มี
ความสัมพันธ์ตายตัวระหว่าง HTTP method กับ action (คลาสเดียวมีได้หลาย action ในหลาย URL):

```python
# blog/viewsets.py — ตัวอย่างเปล่า ๆ ก่อนเห็น ModelViewSet เต็มรูปแบบในขั้นตอนที่ 432
from rest_framework import status, viewsets
from rest_framework.response import Response
from .models import Post
from .serializers import PostSerializer


class PostViewSet(viewsets.ViewSet):
    """
    viewsets.ViewSet (ตัวฐานสุด ไม่มี mixin ใด ๆ) — ต้องเขียน action เองทุกตัว
    เหมือน APIView แต่เปลี่ยนชื่อ method จาก HTTP verb เป็นชื่อ action
    """

    def list(self, request):
        posts = Post.objects.filter(is_published=True)
        serializer = PostSerializer(posts, many=True)
        return Response(serializer.data)

    def create(self, request):
        serializer = PostSerializer(data=request.data)
        serializer.is_valid(raise_exception=True)
        serializer.save(author=request.user)
        return Response(serializer.data, status=status.HTTP_201_CREATED)

    def retrieve(self, request, pk=None):
        post = Post.objects.get(pk=pk)
        serializer = PostSerializer(post)
        return Response(serializer.data)
```

```python
# blog/api_urls.py — เชื่อม ViewSet โดยไม่ใช้ Router (แบบ manual เพื่อเห็นกลไกก่อน)
from django.urls import path
from .viewsets import PostViewSet

urlpatterns = [
    # ระบุ dict {HTTP method: action name} เองตรง ๆ — นี่คือสิ่งที่ Router
    # (ขั้นตอนที่ 432-433) จะทำให้อัตโนมัติแทนเรา
    path('posts/', PostViewSet.as_view({'get': 'list', 'post': 'create'}), name='post-list'),
    path('posts/<int:pk>/', PostViewSet.as_view({'get': 'retrieve'}), name='post-detail'),
]
```

สังเกตว่า `PostViewSet` **หนึ่งคลาส** ให้บริการทั้ง `/posts/` และ `/posts/<pk>/` ต่างจาก
Part 043 ที่ต้องมี 2 คลาสแยกกัน — นี่คือประโยชน์แรกที่จับต้องได้ของ `ViewSet`

### 431.6 เจาะซอร์สโค้ด (ย่อ) ของ `ViewSetMixin.as_view()`

```python
# แนวคิดจากซอร์สโค้ดจริงของ rest_framework.viewsets.ViewSetMixin
class ViewSetMixin:
    @classmethod
    def as_view(cls, actions=None, **initkwargs):
        if not actions:
            raise TypeError(
                "The `actions` argument must be provided when calling `.as_view()` "
                "on a ViewSet. For example `.as_view({'get': 'list'})`"
            )

        def view(request, *args, **kwargs):
            self = cls(**initkwargs)
            # (1) ผูก action ของ HTTP method นี้เข้ากับ self.action
            #     -- ใช้จริงในขั้นตอนที่ 437 ตอน override get_queryset() ตาม self.action
            if hasattr(self, 'get') and not hasattr(self, 'head'):
                self.head = self.get

            self.action_map = actions
            for method, action in actions.items():
                handler = getattr(self, action)
                # (2) ผูก handler จริง (list/create/retrieve/...) เข้ากับชื่อ HTTP method
                #     เพื่อให้ View.dispatch() เดิม (Part 021/042) เรียกมันได้แบบไม่รู้ตัว
                setattr(self, method, handler)

            self.request = request
            self.args = args
            self.kwargs = kwargs

            # (3) นี่คือกุญแจสำคัญที่สุด: หลัง "สับสาย" method เสร็จ ก็โยนต่อให้
            #     APIView.dispatch() ของ Part 042 ทำงานตามปกติทุกประการ!
            return self.dispatch(request, *args, **kwargs)

        return view
```

**นี่คือกุญแจของขั้นตอนนี้ทั้งหมด**: `ViewSetMixin.as_view()` ไม่ได้สร้างกลไก dispatch
ใหม่ขึ้นมาเลย มันแค่ **"สับสาย" (rewire)** ให้ `self.get = self.list`,
`self.post = self.create` เป็นต้น ตาม dict ที่ Router ส่งมา แล้วปล่อยให้
`APIView.dispatch()` (ที่เจาะลึกไปแล้วทั้งหมดใน Part 042 ขั้นตอนที่ 412) ทำงานเหมือนเดิม
ทุกประการ — `ViewSet` จึงไม่ใช่ระบบคู่ขนานใหม่ แต่เป็น **ชั้นบาง ๆ ที่ครอบ `APIView` อีกที
เพื่อแปลง "action name" กลับไปเป็น "HTTP method name"** เท่านั้นเอง

### 431.7 ตาราง Class Hierarchy: `ViewSet` อยู่ตรงไหนในโครงสร้างของ DRF

| คลาส | สืบทอดจาก | มี mixin (list/create/retrieve/...) หรือไม่ | ต้องเขียน action เองหรือไม่ |
|---|---|---|---|
| `APIView` (Part 042) | `django.views.View` | ไม่เกี่ยวข้อง (ไม่มี concept action) | เขียน `get()`/`post()`/... เอง |
| `GenericAPIView` (Part 043) | `APIView` | ไม่มี (ต้องผสม Mixin เอง) | ผสม Mixin แล้วประกาศ `get()` เรียก `self.list()` เอง |
| `ViewSet` | `ViewSetMixin` + `views.APIView` | ไม่มี | เขียน `list()`/`create()`/... เอง (431.5) |
| `GenericViewSet` | `ViewSetMixin` + `generics.GenericAPIView` | ไม่มี | ผสม Mixin เอง (คล้าย `GenericAPIView` แต่คืน action แทน HTTP verb) |
| `ModelViewSet` | `GenericViewSet` + Mixin ครบ 5 ตัว | มีครบ (ขั้นตอนที่ 432) | ไม่ต้องเขียนเลย ใช้ได้ทันที |
| `ReadOnlyModelViewSet` | `GenericViewSet` + Mixin แค่ 2 ตัว | มีแค่ `list`/`retrieve` (ขั้นตอนที่ 436) | ไม่ต้องเขียนเลย |

### 431.8 สรุปย่อก่อนไปขั้นตอนถัดไป

- `ViewSet` แก้ปัญหาการเขียนคลาสซ้ำซ้อนของ Generic Views โดยรวม list + detail endpoint
  ของ resource เดียวกันไว้ใน **คลาสเดียว**
- กลไกภายในไม่ได้สร้างระบบใหม่ แต่ "สับสาย" method ชื่อ action ให้กลายเป็น method ชื่อ
  HTTP verb แล้วส่งต่อให้ `APIView.dispatch()` เดิมทำงาน
- `ViewSet` เปล่า ๆ (ไม่มี Mixin) ยังต้องเขียน `list()`, `create()`, `retrieve()` เองทุกตัว
  เหมือน `APIView` — ประโยชน์เต็มรูปแบบจะเห็นเมื่อรวมกับ Mixin สำเร็จรูปในขั้นตอนที่ 432
  และรวมกับ **Router** ที่สร้าง URL ให้อัตโนมัติแทนการเขียน `path()` เอง

---

## ขั้นตอนที่ 432: `ModelViewSet` — CRUD เต็มรูปแบบในคลาสเดียว พร้อม Router สร้าง URL ให้อัตโนมัติ

### 432.1 โมเดลและ Serializer ที่จะใช้ตลอด Part นี้

เพื่อให้ทุกตัวอย่างในทั้ง 10 ขั้นตอนทำงานร่วมกันได้จริง เราจะยึด `Post`, `Category`,
`Tag`, `Comment` จากแอป `blog` (ต่อยอดจาก Part 012 และ Part 041) เป็นฐานร่วมกัน:

```python
# blog/models.py
from django.conf import settings
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
    author = models.ForeignKey(
        settings.AUTH_USER_MODEL, on_delete=models.CASCADE, related_name='posts',
    )
    category = models.ForeignKey(
        Category, on_delete=models.SET_NULL, null=True, blank=True,
        related_name='posts',
    )
    title = models.CharField(max_length=200)
    slug = models.SlugField(max_length=220, unique=True, blank=True)
    content = models.TextField()
    is_published = models.BooleanField(default=False)
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)

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
    author = models.ForeignKey(settings.AUTH_USER_MODEL, on_delete=models.CASCADE)
    content = models.TextField()
    created_at = models.DateTimeField(auto_now_add=True)

    class Meta:
        ordering = ['created_at']

    def __str__(self):
        return f'{self.author} on {self.post}'
```

```python
# blog/serializers.py
from rest_framework import serializers
from .models import Category, Comment, Post


class CategorySerializer(serializers.ModelSerializer):
    class Meta:
        model = Category
        fields = ['id', 'name', 'slug', 'description']
        read_only_fields = ['slug']


class CommentSerializer(serializers.ModelSerializer):
    author = serializers.ReadOnlyField(source='author.username')

    class Meta:
        model = Comment
        fields = ['id', 'post', 'author', 'content', 'created_at']
        read_only_fields = ['id', 'author', 'created_at']


class PostSerializer(serializers.ModelSerializer):
    author = serializers.ReadOnlyField(source='author.username')
    category_name = serializers.ReadOnlyField(source='category.name')

    class Meta:
        model = Post
        fields = [
            'id', 'author', 'title', 'slug', 'content', 'category',
            'category_name', 'is_published', 'created_at', 'updated_at',
        ]
        read_only_fields = ['id', 'slug', 'created_at', 'updated_at']
```

### 432.2 `ModelViewSet`: ไม่ต้องเขียน action เองเลยสักตัว

```python
# blog/viewsets.py
from rest_framework import viewsets
from .models import Post
from .serializers import PostSerializer


class PostViewSet(viewsets.ModelViewSet):
    """
    ModelViewSet มาพร้อม list/create/retrieve/update/partial_update/destroy
    ครบทั้ง 6 action โดยไม่ต้องเขียนเองสักบรรทัด — ประกาศแค่ queryset กับ
    serializer_class เหมือน Generic Views ของ Part 043 ทุกประการ
    """
    queryset = Post.objects.all()
    serializer_class = PostSerializer
    lookup_field = 'slug'

    def perform_create(self, serializer):
        # perform_create()/perform_update()/perform_destroy() ทำงานเหมือนกับ
        # Generic Views ของ Part 043 ทุกประการ เพราะ ModelViewSet สืบทอด Mixin
        # ชุดเดียวกัน (ดูขั้นตอนที่ 432.7)
        serializer.save(author=self.request.user)
```

เทียบกับ Part 043 ที่ต้องมี **2 คลาส** (`PostListCreateAPIView` +
`PostRetrieveUpdateDestroyAPIView`) ตอนนี้เหลือ **1 คลาส** ที่ประกาศ `queryset` และ
`serializer_class` เพียงครั้งเดียว — แก้ปัญหาความซ้ำซ้อนจากขั้นตอนที่ 431.2 ได้ตรงจุด

### 432.3 ให้ `Router` สร้าง URL ให้อัตโนมัติแทนการเขียน `path()` เอง

```python
# blog/api_urls.py
from django.urls import include, path
from rest_framework.routers import DefaultRouter
from .viewsets import PostViewSet

router = DefaultRouter()
router.register('posts', PostViewSet, basename='post')

app_name = 'blog_api'

urlpatterns = [
    path('', include(router.urls)),
]
```

```python
# config/urls.py
from django.contrib import admin
from django.urls import include, path

urlpatterns = [
    path('admin/', admin.site.urls),
    path('api/', include('blog.api_urls')),
]
```

`router.register('posts', PostViewSet, basename='post')` บรรทัดเดียวแทนที่การเขียน
`path()` ทั้งสองเส้นทางจากขั้นตอนที่ 431.5 — **Router คือตัวที่สร้าง dict
`{HTTP method: action name}` ที่เห็นในขั้นตอนที่ 431.5-431.6 ให้เราโดยอัตโนมัติ** ตาม
Mixin ที่ `PostViewSet` มีอยู่จริง (ถ้าไม่มี `create` mixin ก็จะไม่สร้างเส้นทาง `POST`
ให้ — สำคัญมากสำหรับขั้นตอนที่ 436 เรื่อง `ReadOnlyModelViewSet`)

### 432.4 URL ที่ Router สร้างให้อัตโนมัติทั้งหมด

รันคำสั่งต่อไปนี้เพื่อดู URL ทั้งหมดที่ Router ประกาศให้ `PostViewSet`:

```bash
python manage.py show_urls | grep posts   # ถ้าติดตั้ง django-extensions
# หรือดูจาก router.urls ตรง ๆ ใน shell
python manage.py shell
>>> from blog.api_urls import router
>>> for url in router.urls:
...     print(url.pattern, '->', url.name)
```

| Name | URL Pattern | Action | HTTP Method |
|---|---|---|---|
| `post-list` | `posts/` | `list` | `GET` |
| `post-list` | `posts/` | `create` | `POST` |
| `post-detail` | `posts/{slug}/` | `retrieve` | `GET` |
| `post-detail` | `posts/{slug}/` | `update` | `PUT` |
| `post-detail` | `posts/{slug}/` | `partial_update` | `PATCH` |
| `post-detail` | `posts/{slug}/` | `destroy` | `DELETE` |

สังเกตว่า Router สร้าง URL `name` แค่ **2 ชื่อ** (`post-list`, `post-detail`) ครอบคลุมทั้ง
6 การกระทำ ตรงกับตาราง mapping ในขั้นตอนที่ 431.4 ทุกประการ — ชื่อ `post-list`/`post-detail`
มาจาก `basename='post'` ที่ระบุตอน `router.register()` บวกคำต่อท้ายมาตรฐาน `-list`/`-detail`

### 432.5 ทดสอบ CRUD ครบวงจรด้วย `curl`

```bash
# LIST
curl http://127.0.0.1:8000/api/posts/

# CREATE
curl -X POST http://127.0.0.1:8000/api/posts/ \
  -H "Content-Type: application/json" -b cookies.txt \
  -d '{"title": "ทดสอบ ViewSet", "content": "เนื้อหาทดสอบ"}'

# RETRIEVE
curl http://127.0.0.1:8000/api/posts/ทดสอบ-viewset/

# UPDATE (PUT)
curl -X PUT http://127.0.0.1:8000/api/posts/ทดสอบ-viewset/ \
  -H "Content-Type: application/json" -b cookies.txt \
  -d '{"title": "แก้ไขแล้ว", "content": "เนื้อหาใหม่", "is_published": true}'

# PARTIAL UPDATE (PATCH)
curl -X PATCH http://127.0.0.1:8000/api/posts/ทดสอบ-viewset/ \
  -H "Content-Type: application/json" -b cookies.txt \
  -d '{"is_published": false}'

# DESTROY
curl -X DELETE http://127.0.0.1:8000/api/posts/ทดสอบ-viewset/ -b cookies.txt
```

CRUD ทั้ง 6 action ทำงานครบผ่าน `PostViewSet` เพียงคลาสเดียว โดยที่โค้ดใน `viewsets.py`
มีแค่ 9 บรรทัด (ไม่นับ import) — สั้นกว่า Generic Views ของ Part 043 ที่ต้องมี 2 คลาส
และสั้นกว่า `APIView` ของ Part 042 มาก

### 432.6 เจาะซอร์สโค้ด (ย่อ) ของ `ModelViewSet`: มันคือการรวม Mixin ล้วน ๆ

```python
# แนวคิดจากซอร์สโค้ดจริงของ rest_framework.viewsets
from rest_framework import mixins
from rest_framework.generics import GenericAPIView


class GenericViewSet(ViewSetMixin, GenericAPIView):
    """
    ผสาน ViewSetMixin (ขั้นตอนที่ 431.6 — สับสาย action->HTTP method)
    เข้ากับ GenericAPIView (Part 043 — queryset, get_object(), pagination, filter)
    แต่ยังไม่มี action ใด ๆ ให้ใช้งานจริง ต้องผสม Mixin เพิ่มเอง
    """
    pass


class ModelViewSet(
    mixins.CreateModelMixin,
    mixins.RetrieveModelMixin,
    mixins.UpdateModelMixin,
    mixins.DestroyModelMixin,
    mixins.ListModelMixin,
    GenericViewSet,
):
    """
    ModelViewSet = GenericViewSet + Mixin ครบทั้ง 5 ตัวจาก Part 043
    (CreateModelMixin.create(), RetrieveModelMixin.retrieve(), ...)
    นี่คือเหตุผลที่ perform_create()/perform_update() ใน 432.2 ทำงานเหมือนกับ
    Generic Views ของ Part 043 เป๊ะ ๆ — เพราะมันคือ Mixin ชุดเดียวกันจริง ๆ
    """
    pass
```

**นี่คือกุญแจของขั้นตอนนี้**: `ModelViewSet` **ไม่ได้เขียน logic ของ `create()`,
`retrieve()` ฯลฯ ขึ้นมาใหม่เลยแม้แต่บรรทัดเดียว** มันแค่ผสม (compose) Mixin ตัวเดิมจาก
Part 043 เข้ากับ `ViewSetMixin` จากขั้นตอนที่ 431.6 เท่านั้น พูดให้กระชับที่สุด:

```
ModelViewSet = ViewSetMixin (สับสาย action->method)
             + GenericAPIView (queryset, get_object, pagination — Part 043)
             + Mixin 5 ตัว (List/Create/Retrieve/Update/DestroyModelMixin — Part 043)
```

### 432.7 ตารางสรุป: โค้ดที่ประหยัดได้จาก Part 042 → 043 → 044

| | Part 042 (`APIView`) | Part 043 (Generic Views) | Part 044 (`ModelViewSet`) |
|---|---|---|---|
| จำนวนคลาสสำหรับ `Post` | 2 คลาส | 2 คลาส | **1 คลาส** |
| ต้องเขียน `get()`/`post()`/... เอง | ✅ เขียนเอง | ❌ (ใช้ Mixin) | ❌ (ใช้ Mixin) |
| ต้องเขียน `path()` กี่เส้นทาง | 2 | 2 | **0** (Router สร้างให้) |
| ประกาศ `queryset` กี่ครั้ง | ไม่มี concept นี้ | 2 ครั้ง (2 คลาส) | **1 ครั้ง** |
| จำนวนบรรทัดโค้ดโดยประมาณ (ไม่รวม import) | ~25 บรรทัด | ~14 บรรทัด | **~9 บรรทัด** |

---

## ขั้นตอนที่ 433: Router — `DefaultRouter` vs `SimpleRouter` เปรียบเทียบ

### 433.1 `SimpleRouter`: Router พื้นฐานที่สุด

```python
# blog/api_urls.py
from django.urls import include, path
from rest_framework.routers import SimpleRouter
from .viewsets import PostViewSet

router = SimpleRouter()
router.register('posts', PostViewSet, basename='post')

urlpatterns = [
    path('', include(router.urls)),
]
```

เข้า `http://127.0.0.1:8000/api/posts/` และ `http://127.0.0.1:8000/api/posts/1/` ได้ตามปกติ
แต่เข้า `http://127.0.0.1:8000/api/` (root path เปล่า ๆ) จะได้ `404 Not Found` — เพราะ
`SimpleRouter` สร้างเฉพาะ URL ของ ViewSet ที่ลงทะเบียนไว้เท่านั้น **ไม่มี view เพิ่มเติมใด ๆ**

### 433.2 `DefaultRouter`: เพิ่ม API Root View ให้อัตโนมัติ

```python
# blog/api_urls.py
from django.urls import include, path
from rest_framework.routers import DefaultRouter
from .viewsets import PostViewSet

router = DefaultRouter()
router.register('posts', PostViewSet, basename='post')

urlpatterns = [
    path('', include(router.urls)),
]
```

เข้า `http://127.0.0.1:8000/api/` ตอนนี้จะเห็นหน้า **API Root** ที่ DRF สร้างให้อัตโนมัติ
เป็น Browsable API แสดงลิงก์ไปยังทุก resource ที่ลงทะเบียนไว้:

```json
{
    "posts": "http://127.0.0.1:8000/api/posts/"
}
```

ถ้ามีหลาย ViewSet ลงทะเบียนไว้ (เช่น `categories`, `comments`) หน้า API Root จะแสดงลิงก์
ของทุก resource รวมกันในที่เดียว — มีประโยชน์มากตอนพัฒนา เพราะไม่ต้องจำ URL ทุกเส้นทางเอง

### 433.3 `DefaultRouter` ยังเพิ่ม format suffix ให้อัตโนมัติ

```python
# ด้วย DefaultRouter สามารถเข้าถึงแบบระบุ format ต่อท้ายได้ทันที โดยไม่ต้องตั้งค่าเพิ่ม
GET /api/posts.json    # เทียบเท่า Accept: application/json
GET /api/posts.api     # บังคับ Browsable API แม้เรียกจาก curl
```

ความสามารถนี้มาจากการที่ `DefaultRouter` ตั้ง `include_format_suffixes = True` เป็นค่า
เริ่มต้น (ต่างจาก `SimpleRouter` ที่ไม่มี attribute นี้เลย)

### 433.4 ตารางเปรียบเทียบ `DefaultRouter` vs `SimpleRouter`

| คุณสมบัติ | `SimpleRouter` | `DefaultRouter` |
|---|---|---|
| สร้าง URL list/detail ตาม ViewSet | ✅ | ✅ |
| API Root View (`/api/`) | ❌ ไม่มี | ✅ มีให้อัตโนมัติ |
| รองรับ format suffix (`.json`, `.api`) | ❌ ไม่มี | ✅ มีให้อัตโนมัติ |
| Trailing slash ท้าย URL (`/posts/` มี `/`) | ✅ (ปรับได้ผ่าน `trailing_slash=False`) | ✅ (ปรับได้เหมือนกัน) |
| เหมาะกับ | Internal API, Microservice ที่ไม่ต้องมี root view | Public REST API ที่ต้องการให้ผู้ใช้สำรวจ endpoint เองได้ |
| ความนิยมในโปรเจกต์จริง | ใช้เมื่อรวมกับ router อื่นแบบ nested (ขั้นตอนที่ 435) | ค่าเริ่มต้นที่แนะนำสำหรับ API หลักของโปรเจกต์ |

**คำแนะนำระดับมืออาชีพ**: ใช้ `DefaultRouter` เป็นค่าเริ่มต้นสำหรับ API หลักของโปรเจกต์
เพราะ API Root View ช่วยให้ทีม frontend/mobile หรือแม้แต่ตัวคุณเองในอนาคตสำรวจ endpoint
ทั้งหมดได้จากจุดเดียว ส่วน `SimpleRouter` เก็บไว้ใช้เมื่อต้องประกอบเข้ากับ nested router
(ขั้นตอนที่ 435) ที่ไม่ต้องการ root view ซ้อนกันหลายชั้น

### 433.5 ลงทะเบียนหลาย ViewSet ใน Router เดียว

```python
# blog/api_urls.py
from django.urls import include, path
from rest_framework.routers import DefaultRouter
from .viewsets import CategoryViewSet, CommentViewSet, PostViewSet

router = DefaultRouter()
router.register('posts', PostViewSet, basename='post')
router.register('categories', CategoryViewSet, basename='category')
router.register('comments', CommentViewSet, basename='comment')

app_name = 'blog_api'

urlpatterns = [
    path('', include(router.urls)),
]
```

หน้า API Root จะแสดงทั้ง 3 resource พร้อมกัน:

```json
{
    "posts": "http://127.0.0.1:8000/api/posts/",
    "categories": "http://127.0.0.1:8000/api/categories/",
    "comments": "http://127.0.0.1:8000/api/comments/"
}
```

### 433.6 ทำไมต้องระบุ `basename` เอง เมื่อไหร่ปล่อยให้ Router เดาเอง

```python
router.register('posts', PostViewSet)                       # ไม่ระบุ basename
router.register('posts', PostViewSet, basename='post')      # ระบุ basename เอง
```

ถ้า `PostViewSet` มี `queryset = Post.objects.all()` ประกาศไว้ Router จะเดา `basename`
จากชื่อ model โดยอัตโนมัติ (`post` มาจาก `Post.objects...`) แต่ **ถ้า ViewSet ไม่มี
`queryset` เป็น class attribute** (เช่น override `get_queryset()` แบบ dynamic ทั้งหมด
ตามที่จะเรียนในขั้นตอนที่ 437) Router จะไม่สามารถเดาได้ และจะ raise
`AssertionError: could not automatically determine the basename` — จึงเป็นแนวปฏิบัติที่ดี
ที่จะ **ระบุ `basename` เองเสมอ** ไม่ต้องพึ่งการเดาอัตโนมัติ เพื่อความชัดเจนและป้องกัน
error ที่ไม่คาดคิดเมื่อโค้ดถูกแก้ไขในอนาคต

---

## ขั้นตอนที่ 434: `@action` decorator — เพิ่ม custom action บน ViewSet

### 434.1 ปัญหา: บาง endpoint ไม่ตรงกับ 6 action มาตรฐานของ REST

`ModelViewSet` ให้ 6 action มาตรฐาน (list/create/retrieve/update/partial_update/destroy)
แต่ในงานจริงมักต้องการ endpoint พิเศษที่ไม่ตรง pattern นี้ เช่น:

- `POST /api/posts/{slug}/publish/` — เผยแพร่บทความ (toggle `is_published`)
- `GET /api/posts/published/` — ดูเฉพาะบทความที่เผยแพร่แล้ว (list พิเศษ ไม่ใช่ list เริ่มต้น)

Part 042 ขั้นตอนที่ 411.9 เคยพูดถึงปัญหานี้ในบริบทของ `@api_view` มาแล้ว — ทางออกสำหรับ
`ViewSet` คือ **`@action` decorator**

### 434.2 `@action(detail=True)`: custom action ระดับ "รายตัว"

```python
# blog/viewsets.py
from django.utils import timezone
from rest_framework import status, viewsets
from rest_framework.decorators import action
from rest_framework.response import Response
from .models import Post
from .serializers import PostSerializer


class PostViewSet(viewsets.ModelViewSet):
    queryset = Post.objects.all()
    serializer_class = PostSerializer
    lookup_field = 'slug'

    def perform_create(self, serializer):
        serializer.save(author=self.request.user)

    @action(detail=True, methods=['post'])
    def publish(self, request, slug=None):
        """
        detail=True หมายความว่า action นี้ทำงานกับ instance เดียว
        (ต้องระบุ pk/slug ใน URL) — เข้าถึง instance ผ่าน self.get_object()
        เหมือนกับที่ retrieve()/update() ทำ
        """
        post = self.get_object()
        post.is_published = True
        post.save(update_fields=['is_published', 'updated_at'])
        serializer = self.get_serializer(post)
        return Response(serializer.data)

    @action(detail=True, methods=['post'])
    def unpublish(self, request, slug=None):
        post = self.get_object()
        post.is_published = False
        post.save(update_fields=['is_published', 'updated_at'])
        serializer = self.get_serializer(post)
        return Response(serializer.data)
```

`@action(detail=True)` ทำให้ Router สร้าง URL เพิ่มโดยอัตโนมัติ:

```
POST /api/posts/{slug}/publish/    -> PostViewSet.publish()
POST /api/posts/{slug}/unpublish/  -> PostViewSet.unpublish()
```

### 434.3 `@action(detail=False)`: custom action ระดับ "รายการ"

```python
    @action(detail=False)
    def published(self, request):
        """
        detail=False หมายความว่า action นี้ไม่ต้องมี pk/slug ใน URL
        (ทำงานระดับ collection เหมือน list()) methods default เป็น ['get']
        เมื่อไม่ระบุ
        """
        queryset = self.get_queryset().filter(is_published=True)
        page = self.paginate_queryset(queryset)
        if page is not None:
            serializer = self.get_serializer(page, many=True)
            return self.get_paginated_response(serializer.data)
        serializer = self.get_serializer(queryset, many=True)
        return Response(serializer.data)

    @action(detail=False)
    def mine(self, request):
        queryset = self.get_queryset().filter(author=request.user)
        serializer = self.get_serializer(queryset, many=True)
        return Response(serializer.data)
```

Router สร้าง URL เพิ่ม:

```
GET /api/posts/published/   -> PostViewSet.published()
GET /api/posts/mine/        -> PostViewSet.mine()
```

**ข้อควรระวังเรื่องลำดับ URL**: `posts/published/` กับ `posts/{slug}/` อาจดูเหมือนชนกัน
แต่ Router เรียงให้ URL ของ `@action(detail=False)` (ไม่มี `<slug>`) มาก่อน URL ของ
`retrieve` (`posts/<slug>/`) เสมอโดยอัตโนมัติ ดังนั้นถ้ามีบทความที่ slug บังเอิญชื่อ
`published` จริง ๆ จะเข้าถึงไม่ได้ผ่าน `retrieve()` — เป็นข้อจำกัดที่ควรรู้ไว้เมื่อออกแบบ
custom action ที่ตั้งชื่อคล้ายกับค่าที่เป็นไปได้ของ `slug`

### 434.4 `url_path` และ `url_name`: กำหนดเส้นทาง/ชื่อ URL เอง

```python
    @action(detail=True, methods=['post'], url_path='toggle-publish', url_name='toggle-publish')
    def toggle_publish(self, request, slug=None):
        post = self.get_object()
        post.is_published = not post.is_published
        post.save(update_fields=['is_published', 'updated_at'])
        serializer = self.get_serializer(post)
        return Response(serializer.data)
```

ถ้าไม่ระบุ `url_path`/`url_name` DRF จะใช้ชื่อ method เป็นค่าเริ่มต้น (แปลง `_` เป็น `-`
ใน `url_path` อัตโนมัติ) การระบุเองมีประโยชน์เมื่อต้องการชื่อ URL ที่อ่านง่ายกว่าชื่อ
Python method (เช่น method ชื่อ `toggle_publish` แต่อยากได้ URL `toggle-publish` ซึ่งจริง
ๆ ได้อัตโนมัติอยู่แล้ว แต่ถ้าต้องการ URL ที่ต่างจากชื่อ method ไปเลย เช่น
`url_path='switch'` ก็ทำได้)

### 434.5 กำหนด permission เฉพาะ action เดียวผ่าน `@action`

```python
    @action(detail=True, methods=['post'], permission_classes=[IsAdminUser])
    def feature(self, request, slug=None):
        """
        เฉพาะ action นี้เท่านั้นที่ต้องเป็น admin — action อื่นใน ViewSet เดียวกัน
        ยังใช้ permission_classes ระดับคลาสตามปกติ (เจาะลึกเต็มรูปแบบในขั้นตอนที่ 439)
        """
        post = self.get_object()
        post.is_featured = True
        post.save(update_fields=['is_featured'])
        return Response({'status': 'featured'})
```

(ต้องเพิ่ม import `IsAdminUser` จาก `rest_framework.permissions` — จะเจาะลึก permission
เต็มรูปแบบใน Part 045 ตอนนี้ขอให้เห็นว่า `@action` รับ `permission_classes` ได้เหมือนกับ
ที่ `@api_view` รับ `@permission_classes` ใน Part 042 ขั้นตอนที่ 411.8)

### 434.6 ตารางสรุปพารามิเตอร์ของ `@action`

| พารามิเตอร์ | ความหมาย | ค่าเริ่มต้น |
|---|---|---|
| `detail` | `True` = ทำงานกับ instance เดียว (ต้องมี pk/slug ใน URL), `False` = ทำงานระดับ collection | **จำเป็นต้องระบุ** ไม่มีค่าเริ่มต้น |
| `methods` | list ของ HTTP method ที่รองรับ | `['get']` |
| `url_path` | ส่วนของ URL หลัง pk/slug (หรือหลัง base สำหรับ `detail=False`) | ชื่อ method (แปลง `_` เป็น `-`) |
| `url_name` | ชื่อสำหรับ `reverse()` | ชื่อ method (แปลง `_` เป็น `-`) |
| `permission_classes` | override permission เฉพาะ action นี้ | ใช้ค่าจากคลาส (`self.permission_classes`) |
| `serializer_class` | override serializer เฉพาะ action นี้ | ใช้ค่าจากคลาส (`self.serializer_class`) |

### 434.7 ทดสอบ custom action ด้วย `curl`

```bash
curl -X POST http://127.0.0.1:8000/api/posts/ทดสอบ-viewset/publish/ -b cookies.txt
# {"id": 1, "title": "ทดสอบ ViewSet", "is_published": true, ...}

curl http://127.0.0.1:8000/api/posts/published/
# [{"id": 1, "title": "ทดสอบ ViewSet", "is_published": true, ...}, ...]

curl http://127.0.0.1:8000/api/posts/mine/ -b cookies.txt
# [รายการบทความที่ request.user เป็นเจ้าของเท่านั้น]
```

---

## ขั้นตอนที่ 435: Nested Router ด้วย `drf-nested-routers`

### 435.1 ปัญหา: `Comment` เป็น sub-resource ของ `Post`

ตาม REST convention ที่ดี `Comment` ควรเข้าถึงผ่าน URL ที่แสดงความเป็นเจ้าของชัดเจน:

```
GET  /api/posts/{post_pk}/comments/         -> list comment เฉพาะของ post นี้
POST /api/posts/{post_pk}/comments/         -> สร้าง comment ผูกกับ post นี้
GET  /api/posts/{post_pk}/comments/{pk}/    -> ดู comment ตัวเดียว
```

`router.register()` ปกติของ `DefaultRouter`/`SimpleRouter` **ทำ URL แบบนี้ไม่ได้โดยตรง**
เพราะมันออกแบบมาสำหรับ resource ระดับบนสุด (top-level) เท่านั้น ไม่รองรับการซ้อน
`{post_pk}` ไว้หน้า resource ลูก

### 435.2 ติดตั้ง `drf-nested-routers`

```bash
pip install drf-nested-routers
pip freeze > requirements.txt
```

Package นี้ไม่ใช่ส่วนหนึ่งของ DRF core แต่เป็น third-party package ที่ได้รับความนิยมสูง
มากในระบบนิเวศ DRF สำหรับแก้ปัญหา nested resource โดยเฉพาะ

### 435.3 สร้าง `CommentViewSet` ที่กรองตาม `post_pk` ใน URL

```python
# blog/viewsets.py
from rest_framework import viewsets
from .models import Comment
from .serializers import CommentSerializer


class CommentViewSet(viewsets.ModelViewSet):
    serializer_class = CommentSerializer

    def get_queryset(self):
        # self.kwargs['post_pk'] มาจาก NestedDefaultRouter — ชื่อ kwarg นี้เกิดจาก
        # lookup='post' ที่ระบุตอนสร้าง nested router ในขั้นตอนที่ 435.4
        return Comment.objects.filter(post_id=self.kwargs['post_pk'])

    def perform_create(self, serializer):
        serializer.save(
            post_id=self.kwargs['post_pk'],
            author=self.request.user,
        )
```

### 435.4 ประกอบ `NestedDefaultRouter` เข้ากับ Router หลัก

```python
# blog/api_urls.py
from django.urls import include, path
from rest_framework.routers import DefaultRouter
from rest_framework_nested import routers as nested_routers
from .viewsets import CommentViewSet, PostViewSet

router = DefaultRouter()
router.register('posts', PostViewSet, basename='post')

# สร้าง nested router: 'posts' ต้องตรงกับ prefix ที่ลงทะเบียนไว้ใน router หลัก
# lookup='post' -> ทำให้เกิด kwarg ชื่อ 'post_pk' ใน URL (lookup + '_pk')
posts_router = nested_routers.NestedDefaultRouter(router, 'posts', lookup='post')
posts_router.register('comments', CommentViewSet, basename='post-comments')

app_name = 'blog_api'

urlpatterns = [
    path('', include(router.urls)),
    path('', include(posts_router.urls)),
]
```

**ข้อควรระวัง**: `lookup_field` ของ `PostViewSet` ในตัวอย่างก่อนหน้านี้ถูกตั้งเป็น
`'slug'` แต่ nested router ของ `drf-nested-routers` ค่าเริ่มต้นคาดหวัง `pk` เชิงตัวเลข
ในการ generate URL หากต้องการให้ nested URL ใช้ `slug` เช่นกัน (เช่น
`/posts/{post_slug}/comments/`) ต้องส่ง `lookup_field='slug'` ให้กับ `field(...)` ตอน
`register()` ด้วย เพื่อความง่ายในขั้นตอนนี้ เราจะสมมติว่า `PostViewSet` ที่ใช้คู่กับ
nested router นี้อ้างอิงด้วย `pk` ตามค่าเริ่มต้น

### 435.5 URL ที่ได้ทั้งหมดจากการประกอบ Nested Router

| URL | Action | คำอธิบาย |
|---|---|---|
| `GET /api/posts/` | `PostViewSet.list()` | รายการบทความทั้งหมด |
| `GET /api/posts/{pk}/` | `PostViewSet.retrieve()` | บทความตัวเดียว |
| `GET /api/posts/{post_pk}/comments/` | `CommentViewSet.list()` | comment เฉพาะของบทความนั้น |
| `POST /api/posts/{post_pk}/comments/` | `CommentViewSet.create()` | สร้าง comment ผูกกับบทความนั้นอัตโนมัติ |
| `GET /api/posts/{post_pk}/comments/{pk}/` | `CommentViewSet.retrieve()` | comment ตัวเดียว |
| `DELETE /api/posts/{post_pk}/comments/{pk}/` | `CommentViewSet.destroy()` | ลบ comment |

```bash
# ทดสอบสร้าง comment ผูกกับ post pk=1
curl -X POST http://127.0.0.1:8000/api/posts/1/comments/ \
  -H "Content-Type: application/json" -b cookies.txt \
  -d '{"content": "ความคิดเห็นทดสอบ nested router"}'

# ทดสอบ list comment เฉพาะของ post pk=1
curl http://127.0.0.1:8000/api/posts/1/comments/
```

### 435.6 ทางเลือกแบบ manual โดยไม่พึ่ง package ภายนอก

หากไม่ต้องการติดตั้ง `drf-nested-routers` (เช่น โปรเจกต์ที่ต้องการพึ่งพา dependency
น้อยที่สุด) สามารถเขียน URL ซ้อนเองแบบ manual ได้เช่นกัน โดยแลกกับการเสีย feature
บางอย่างของ Router (เช่น API Root ที่ list nested resource):

```python
# blog/api_urls.py — ทางเลือก manual แบบไม่ใช้ drf-nested-routers
from django.urls import path
from .viewsets import CommentViewSet

urlpatterns = [
    path(
        'posts/<int:post_pk>/comments/',
        CommentViewSet.as_view({'get': 'list', 'post': 'create'}),
        name='post-comments-list',
    ),
    path(
        'posts/<int:post_pk>/comments/<int:pk>/',
        CommentViewSet.as_view({
            'get': 'retrieve', 'put': 'update',
            'patch': 'partial_update', 'delete': 'destroy',
        }),
        name='post-comments-detail',
    ),
]
```

`CommentViewSet` (จากขั้นตอนที่ 435.3) ใช้โค้ดเดิมได้ทันทีโดยไม่ต้องแก้ไข เพราะ
`self.kwargs['post_pk']` มาจาก URL pattern `<int:post_pk>` ตรง ๆ ไม่เกี่ยวกับว่าจะสร้าง
URL ผ่าน Router หรือเขียน `path()` เอง — นี่คือเหตุผลที่แนะนำให้ใช้
`drf-nested-routers` ในโปรเจกต์ที่มี nested resource มากกว่า 1-2 จุด เพราะลดโค้ดซ้ำซ้อน
แบบนี้ได้มาก

---

## ขั้นตอนที่ 436: `ReadOnlyModelViewSet` — สำหรับ endpoint ที่ให้แค่อ่าน

### 436.1 กรณีใช้งาน: `Category` ให้ frontend อ่านได้อย่างเดียว

`Category` ในระบบนี้ถูกจัดการผ่าน Django Admin เท่านั้น (ทบทวนจาก Part 017-018) — ฝั่ง
public API จึงควร **อนุญาตให้อ่านได้อย่างเดียว** (`list`/`retrieve`) โดยไม่เปิด
`create`/`update`/`destroy` ให้เรียกผ่าน API เด็ดขาด

### 436.2 `ReadOnlyModelViewSet` เต็มรูปแบบ

```python
# blog/viewsets.py
from rest_framework import viewsets
from .models import Category
from .serializers import CategorySerializer


class CategoryViewSet(viewsets.ReadOnlyModelViewSet):
    queryset = Category.objects.all()
    serializer_class = CategorySerializer
    lookup_field = 'slug'
```

```python
# blog/api_urls.py
router.register('categories', CategoryViewSet, basename='category')
```

### 436.3 เจาะซอร์สโค้ด (ย่อ): มี Mixin แค่ 2 ตัวเทียบกับ `ModelViewSet` ที่มี 5 ตัว

```python
# แนวคิดจากซอร์สโค้ดจริงของ rest_framework.viewsets
from rest_framework import mixins


class ReadOnlyModelViewSet(
    mixins.RetrieveModelMixin,
    mixins.ListModelMixin,
    GenericViewSet,
):
    """
    มีแค่ RetrieveModelMixin + ListModelMixin เทียบกับ ModelViewSet
    (ขั้นตอนที่ 432.6) ที่มีครบ 5 ตัว — ไม่มี CreateModelMixin,
    UpdateModelMixin, DestroyModelMixin เลย
    """
    pass
```

### 436.4 ทดสอบว่า write operation ถูกปฏิเสธจริง

```bash
curl http://127.0.0.1:8000/api/categories/
# 200 OK — list ทำงานปกติ

curl -X POST http://127.0.0.1:8000/api/categories/ \
  -H "Content-Type: application/json" -d '{"name": "หมวดใหม่"}'
# 405 Method Not Allowed — Router ไม่ได้สร้าง route POST ให้เลยตั้งแต่แรก
```

สังเกตว่าผลลัพธ์คือ **405** ไม่ใช่ 403 หรือ 401 เพราะ `POST /api/categories/` **ไม่มี
route ใด ๆ ผูกกับมันเลย** ตั้งแต่ตอน Router สร้าง URL (ต่างจากการปฏิเสธด้วย permission
ที่จะเป็น 403 — ทบทวนความแตกต่างนี้จาก Part 042 ขั้นตอนที่ 417) นี่คือข้อดีด้าน security
อีกชั้นหนึ่งของ `ReadOnlyModelViewSet`: endpoint เขียนข้อมูล **ไม่มีอยู่จริงในระบบเลย**
ไม่ใช่แค่ถูกปฏิเสธด้วย permission check ที่อาจตั้งค่าผิดพลาดได้

### 436.5 ตารางเปรียบเทียบ URL ที่ Router สร้างให้: `ModelViewSet` vs `ReadOnlyModelViewSet`

| Action | `ModelViewSet` | `ReadOnlyModelViewSet` |
|---|---|---|
| `list` (`GET /categories/`) | ✅ | ✅ |
| `create` (`POST /categories/`) | ✅ | ❌ ไม่มี route |
| `retrieve` (`GET /categories/{pk}/`) | ✅ | ✅ |
| `update` (`PUT /categories/{pk}/`) | ✅ | ❌ ไม่มี route |
| `partial_update` (`PATCH /categories/{pk}/`) | ✅ | ❌ ไม่มี route |
| `destroy` (`DELETE /categories/{pk}/`) | ✅ | ❌ ไม่มี route |

---

## ขั้นตอนที่ 437: Override `get_queryset()` แยกตาม action ภายใน ViewSet เดียว

### 437.1 ปัญหา: ผู้ใช้ทั่วไปกับ staff ควรเห็นข้อมูลไม่เท่ากัน

ต้องการให้:

- ผู้ใช้ทั่วไป (`list`/`retrieve`) เห็นเฉพาะบทความที่ `is_published=True`
- เจ้าของบทความหรือ staff เห็นบทความของตัวเองที่ยังไม่เผยแพร่ได้ด้วย (`retrieve`)
- action `mine` (จากขั้นตอนที่ 434.3) ต้องเห็นบทความ **ทั้งหมด** ของตัวเอง ไม่ว่าเผยแพร่
  แล้วหรือไม่

การเขียน `queryset = Post.objects.filter(is_published=True)` เป็น class attribute ตรง ๆ
แบบขั้นตอนที่ 432.2 **ทำแบบนี้ไม่ได้** เพราะมันเป็นค่าคงที่ ไม่เปลี่ยนตาม action หรือ
`request.user` — ต้อง override `get_queryset()` แทน

### 437.2 `self.action`: attribute ที่ `ViewSetMixin` ผูกให้อัตโนมัติ

ทวนจากขั้นตอนที่ 431.6: `ViewSetMixin.as_view()` เก็บชื่อ action ปัจจุบันไว้ผ่าน
`self.action_map` และตั้ง `self.action` ให้ตรงกับ action ที่ Router routing มาให้ในแต่ละ
request — เราจึงใช้ `self.action` เป็นเงื่อนไขแยก behavior ได้:

```python
# blog/viewsets.py
from rest_framework import viewsets
from .models import Post
from .serializers import PostSerializer


class PostViewSet(viewsets.ModelViewSet):
    serializer_class = PostSerializer
    lookup_field = 'slug'

    def get_queryset(self):
        user = self.request.user

        if self.action == 'mine':
            # action พิเศษจากขั้นตอนที่ 434.3: เห็นของตัวเองทั้งหมด ไม่กรอง is_published
            return Post.objects.filter(author=user)

        if self.action in ('update', 'partial_update', 'destroy'):
            # แก้ไข/ลบได้เฉพาะบทความของตัวเอง (permission เพิ่มเติมใน 439)
            if user.is_staff:
                return Post.objects.all()
            return Post.objects.filter(author=user)

        if user.is_authenticated and user.is_staff:
            # staff เห็นทุกบทความรวมที่ยังไม่เผยแพร่ ตอน list/retrieve
            return Post.objects.all()

        # ผู้ใช้ทั่วไป/anonymous เห็นเฉพาะบทความที่เผยแพร่แล้วเท่านั้น
        return Post.objects.filter(is_published=True)

    def perform_create(self, serializer):
        serializer.save(author=self.request.user)
```

### 437.3 ผสาน `get_serializer_class()` แยกตาม action ด้วยหลักการเดียวกัน

```python
    def get_serializer_class(self):
        if self.action == 'list':
            # ตอน list ใช้ serializer แบบย่อ ไม่ต้องส่ง content เต็มทุกบทความ
            return PostListSerializer
        return PostSerializer   # retrieve/create/update ใช้ serializer เต็มรูปแบบ
```

รูปแบบนี้ (`get_queryset()`/`get_serializer_class()` เช็ค `self.action`) เป็นแพตเทิร์นที่
พบได้บ่อยที่สุดในโปรเจกต์ DRF ระดับมืออาชีพ เพราะช่วยให้ ViewSet เดียวปรับพฤติกรรมตาม
บริบทได้ครบถ้วนโดยไม่ต้องแยกคลาส

### 437.4 ตารางค่าที่เป็นไปได้ของ `self.action`

| `self.action` | เกิดขึ้นเมื่อ | ตัวอย่างการใช้ในเงื่อนไข |
|---|---|---|
| `'list'` | `GET /posts/` | ใช้ serializer แบบย่อ, กรอง `is_published` |
| `'create'` | `POST /posts/` | ตรวจสิทธิ์การสร้าง |
| `'retrieve'` | `GET /posts/{pk}/` | อนุญาตเจ้าของเห็น draft ของตัวเอง |
| `'update'` | `PUT /posts/{pk}/` | จำกัด queryset ให้เจ้าของเท่านั้น |
| `'partial_update'` | `PATCH /posts/{pk}/` | เหมือน `update` |
| `'destroy'` | `DELETE /posts/{pk}/` | เหมือน `update` |
| ชื่อ method ของ `@action` (เช่น `'publish'`, `'mine'`) | เรียก custom action นั้น ๆ | เงื่อนไขเฉพาะ action |

### 437.5 กับดักคลาสสิก: `self.action` ใช้ไม่ได้นอกวงจร request

```python
class PostViewSet(viewsets.ModelViewSet):
    # ผิด! self.action ยังไม่ถูกตั้งค่าตอนนี้ (ViewSetMixin ตั้งค่าตอน as_view()
    # ถูกเรียกใน request-response cycle เท่านั้น ทบทวนจากขั้นตอนที่ 431.6)
    queryset = Post.objects.filter(is_published=(action == 'list'))  # NameError!

    def get_queryset(self):
        # ถูกต้อง! get_queryset() ถูกเรียกระหว่าง request แล้ว self.action
        # ถูกตั้งค่าไว้แล้วแน่นอน
        if self.action == 'list':
            return Post.objects.filter(is_published=True)
        return Post.objects.all()
```

หลักการคือ: **`self.action` เข้าถึงได้เฉพาะภายใน method ที่ถูกเรียกระหว่าง request
เท่านั้น** (`get_queryset()`, `get_serializer_class()`, `get_permissions()`,
`perform_create()` ฯลฯ) ไม่ใช่ตอนนิยาม class attribute ตรง ๆ

---

## ขั้นตอนที่ 438: ตารางเปรียบเทียบ ViewSet vs Generic View — เมื่อไหร่ควรเลือกอะไร

### 438.1 ทบทวนภาพรวมทั้งสามแนวทางที่เรียนมา

ตอนนี้คุณมี 3 ทางเลือกในมือสำหรับเขียน API endpoint:

1. **`APIView`** (Part 042) — ควบคุมทุกอย่างเอง ยืดหยุ่นที่สุด
2. **Generic Views + Mixin** (Part 043) — ลดโค้ดซ้ำซ้อนสำหรับ CRUD pattern มาตรฐาน
   แต่ยังแยกคลาสตาม list/detail
3. **`ViewSet` + Router** (Part นี้) — รวม CRUD ทั้งหมดของ resource ไว้คลาสเดียว
   และให้ Router สร้าง URL ให้อัตโนมัติ

### 438.2 ตารางเปรียบเทียบละเอียด

| มิติ | `APIView` | Generic View + Mixin | `ViewSet` + Router |
|---|---|---|---|
| จำนวนคลาสต่อ resource (list+detail) | 2 | 2 | **1** |
| ต้องเขียน `path()` เอง | ✅ ต้องเขียนทุกเส้นทาง | ✅ ต้องเขียนทุกเส้นทาง | ❌ Router สร้างให้ |
| ควบคุม URL pattern ได้ละเอียดแค่ไหน | สูงสุด (กำหนดเองทุกจุด) | สูง | ปานกลาง (ผูกกับ convention ของ Router) |
| เพิ่ม custom endpoint นอกเหนือ CRUD | เขียน method/URL เพิ่มเองอิสระ | เขียน `APIView`/`@api_view` แยกต่างหาก | `@action` ในคลาสเดียวกัน (434) |
| เหมาะกับ endpoint เดี่ยว ๆ ไม่เกี่ยวกับ resource ใด | ดีมาก | ปานกลาง | ไม่เหมาะ (ออกแบบมาเพื่อ resource) |
| เหมาะกับ CRUD ปกติของ resource | ต้องเขียนเยอะ | ดี | **ดีที่สุด** |
| Nested resource (`/posts/{id}/comments/`) | เขียนเองอิสระเต็มที่ | เขียนเองอิสระเต็มที่ | ต้องพึ่ง `drf-nested-routers` (435) |
| Auto-generate API Root / Browsable API listing | ไม่มี | ไม่มี | ✅ (`DefaultRouter`) |
| Testing รายบุคคล (`view_instance.get(request)`) | ง่ายที่สุด | ง่าย | ต้องผ่าน `as_view({...})` ก่อน |
| Learning curve | ต่ำ (ใกล้เคียง FBV) | ปานกลาง | ปานกลาง-สูง (ต้องเข้าใจ action mapping) |
| ความนิยมในโปรเจกต์ DRF ระดับ enterprise | ใช้เสริมเฉพาะจุด | ใช้เสริมเฉพาะจุด | **เป็นค่าเริ่มต้นของ CRUD API ส่วนใหญ่** |

### 438.3 แนวทางตัดสินใจแบบย่อ

```
ต้องการ CRUD มาตรฐานสำหรับ resource (Model) หนึ่งตัว?
├── ใช่ → ใช้ ModelViewSet + Router (Part นี้)
│         ├── ต้องการอ่านอย่างเดียว? → ReadOnlyModelViewSet (436)
│         └── ต้องการ custom endpoint เพิ่ม? → @action (434)
└── ไม่ใช่ (endpoint เดี่ยว ๆ ไม่ผูกกับ resource)
          เช่น healthcheck, webhook, endpoint รวมสถิติข้ามหลาย model
          → ใช้ APIView หรือ @api_view (Part 042)
```

### 438.4 ตัวอย่างสถานการณ์จริงและคำแนะนำ

| สถานการณ์ | แนะนำใช้ | เหตุผล |
|---|---|---|
| CRUD API สำหรับ `Post`, `Category`, `Comment` | `ModelViewSet`/`ReadOnlyModelViewSet` | resource ชัดเจน ต้องการ 6 action มาตรฐาน |
| Endpoint `GET /api/health/` | `@api_view` | ไม่ผูกกับ resource ใด ไม่มี CRUD |
| Endpoint `POST /api/webhooks/stripe/` | `APIView` | logic เฉพาะทาง ไม่ตรง pattern REST resource |
| Endpoint `GET /api/dashboard/summary/` (รวมสถิติจากหลาย model) | `APIView` | ไม่ผูกกับ model เดียว ดึงข้อมูลข้าม resource |
| API ที่ต้องมี nested resource ลึกหลายชั้น | `ViewSet` + `drf-nested-routers` | ลด boilerplate ของ URL ซ้อนกันได้มาก |
| Endpoint ที่ทีม frontend ต้องการสำรวจเองผ่าน Browsable API | `ViewSet` + `DefaultRouter` | ได้ API Root View ให้ฟรี |

---

## ขั้นตอนที่ 439: ผสาน permission ที่ต่างกันต่อ action ด้วย `get_permissions()`

### 439.1 ปัญหา: permission ไม่เหมือนกันในทุก action ของ ViewSet เดียว

ต้องการกฎต่อไปนี้ใน `PostViewSet` เดียวกัน:

- `list`/`retrieve` — ใครก็เข้าถึงได้ (public read)
- `create` — ต้อง login แล้วเท่านั้น
- `update`/`partial_update`/`destroy` — ต้อง login **และ** ต้องเป็นเจ้าของบทความเท่านั้น
- `publish`/`unpublish` (custom action จากขั้นตอนที่ 434.2) — ต้องเป็น staff เท่านั้น

การประกาศ `permission_classes = [...]` เป็น class attribute ตัวเดียวแบบที่เรียนใน Part 042
ทำได้แค่กฎเดียวสำหรับทั้งคลาส ไม่สามารถแยกตาม action ได้ — ทางออกคือ override
`get_permissions()`

### 439.2 override `get_permissions()` ตาม `self.action`

```python
# blog/permissions.py — ทบทวนจาก Part 033/038: custom permission ตรวจความเป็นเจ้าของ
from rest_framework.permissions import BasePermission, SAFE_METHODS


class IsOwnerOrReadOnly(BasePermission):
    def has_object_permission(self, request, view, obj):
        if request.method in SAFE_METHODS:
            return True
        return obj.author_id == request.user.id
```

```python
# blog/viewsets.py
from rest_framework import viewsets
from rest_framework.permissions import (
    AllowAny, IsAdminUser, IsAuthenticated, IsAuthenticatedOrReadOnly,
)
from .models import Post
from .permissions import IsOwnerOrReadOnly
from .serializers import PostSerializer


class PostViewSet(viewsets.ModelViewSet):
    serializer_class = PostSerializer
    lookup_field = 'slug'
    # permission_classes ระดับคลาสยังคงมีไว้เป็นค่าเริ่มต้น (fallback)
    # สำหรับ action ที่ get_permissions() ไม่ได้จัดการเป็นพิเศษ
    permission_classes = [IsAuthenticatedOrReadOnly]

    def get_queryset(self):
        # (เหมือนขั้นตอนที่ 437.2)
        ...

    def get_permissions(self):
        if self.action in ('list', 'retrieve'):
            permission_classes = [AllowAny]
        elif self.action == 'create':
            permission_classes = [IsAuthenticated]
        elif self.action in ('update', 'partial_update', 'destroy'):
            permission_classes = [IsAuthenticated, IsOwnerOrReadOnly]
        elif self.action in ('publish', 'unpublish'):
            permission_classes = [IsAdminUser]
        else:
            permission_classes = self.permission_classes

        # ต้องคืนเป็น "instance" ของ permission class ไม่ใช่ class ตรง ๆ
        # (ตรงกับที่ check_permissions() ของ Part 042 ขั้นตอนที่ 412.6 คาดหวัง)
        return [permission() for permission in permission_classes]

    def perform_create(self, serializer):
        serializer.save(author=self.request.user)
```

### 439.3 ทำไมต้องคืนเป็น instance (`permission()`) ไม่ใช่ class (`permission`)

ทบทวนจาก Part 042 ขั้นตอนที่ 412.6: `self.check_permissions(request)` ที่ทำงานใน
`initial()` วน loop เรียก `.has_permission(request, view)` บนแต่ละ object ใน
`self.get_permissions()` — ถ้าคืน class เฉย ๆ (ไม่เรียก `()`) จะไม่มี method
`has_permission` ให้เรียก (มันเป็น method ของ instance ไม่ใช่ของ class) และจะเกิด
`TypeError` ทันที นี่คือกับดักที่พบบ่อยที่สุดตอนเขียน `get_permissions()` เอง

### 439.4 ตารางสรุป permission ต่อ action ของ `PostViewSet`

| Action | Permission Classes | ผลลัพธ์ |
|---|---|---|
| `list` | `AllowAny` | ทุกคนเข้าถึงได้ รวม anonymous |
| `retrieve` | `AllowAny` | ทุกคนเข้าถึงได้ |
| `create` | `IsAuthenticated` | ต้อง login เท่านั้น |
| `update`/`partial_update` | `IsAuthenticated` + `IsOwnerOrReadOnly` | ต้อง login และเป็นเจ้าของ |
| `destroy` | `IsAuthenticated` + `IsOwnerOrReadOnly` | ต้อง login และเป็นเจ้าของ |
| `publish`/`unpublish` | `IsAdminUser` | ต้องเป็น staff เท่านั้น |
| action อื่นที่ไม่ได้ระบุ | ใช้ `self.permission_classes` (`IsAuthenticatedOrReadOnly`) | ตามค่า fallback ระดับคลาส |

### 439.5 ทดสอบ permission ทั้ง 3 ระดับด้วย `curl`

```bash
# anonymous อ่านได้
curl http://127.0.0.1:8000/api/posts/
# 200 OK

# anonymous สร้างไม่ได้
curl -X POST http://127.0.0.1:8000/api/posts/ -d '{"title": "x"}'
# 401/403

# login แล้วแต่ไม่ใช่เจ้าของ พยายามลบบทความคนอื่น
curl -X DELETE http://127.0.0.1:8000/api/posts/บทความของคนอื่น/ -b cookies_user_b.txt
# 403 Forbidden (has_object_permission ของ IsOwnerOrReadOnly คืน False)

# login เป็น staff เรียก publish ได้
curl -X POST http://127.0.0.1:8000/api/posts/ทดสอบ-viewset/publish/ -b cookies_staff.txt
# 200 OK

# login เป็น user ธรรมดา (ไม่ใช่ staff) เรียก publish ไม่ได้
curl -X POST http://127.0.0.1:8000/api/posts/ทดสอบ-viewset/publish/ -b cookies_user_a.txt
# 403 Forbidden
```

---

## ขั้นตอนที่ 440: สรุปและแบบฝึกหัด — สร้าง Blog API เต็มรูปแบบด้วย ViewSet + Router

### 440.1 ประกอบทุกอย่างที่เรียนมาเข้าด้วยกัน: `blog/viewsets.py` ฉบับสมบูรณ์

```python
# blog/viewsets.py
from rest_framework import status, viewsets
from rest_framework.decorators import action
from rest_framework.permissions import (
    AllowAny, IsAdminUser, IsAuthenticated, IsAuthenticatedOrReadOnly,
)
from rest_framework.response import Response
from .models import Category, Comment, Post
from .permissions import IsOwnerOrReadOnly
from .serializers import CategorySerializer, CommentSerializer, PostSerializer


class CategoryViewSet(viewsets.ReadOnlyModelViewSet):
    """อ่านอย่างเดียว — จัดการผ่าน Django Admin เท่านั้น (ขั้นตอนที่ 436)"""
    queryset = Category.objects.all()
    serializer_class = CategorySerializer
    lookup_field = 'slug'
    permission_classes = [AllowAny]


class PostViewSet(viewsets.ModelViewSet):
    """CRUD เต็มรูปแบบ + custom action + queryset/permission แยกตาม action"""
    serializer_class = PostSerializer
    lookup_field = 'slug'
    permission_classes = [IsAuthenticatedOrReadOnly]

    def get_queryset(self):
        user = self.request.user
        if self.action == 'mine':
            return Post.objects.filter(author=user)
        if self.action in ('update', 'partial_update', 'destroy'):
            return Post.objects.all() if user.is_staff else Post.objects.filter(author=user)
        if user.is_authenticated and user.is_staff:
            return Post.objects.all()
        return Post.objects.filter(is_published=True)

    def get_permissions(self):
        if self.action in ('list', 'retrieve'):
            permission_classes = [AllowAny]
        elif self.action == 'create':
            permission_classes = [IsAuthenticated]
        elif self.action in ('update', 'partial_update', 'destroy'):
            permission_classes = [IsAuthenticated, IsOwnerOrReadOnly]
        elif self.action in ('publish', 'unpublish'):
            permission_classes = [IsAdminUser]
        else:
            permission_classes = self.permission_classes
        return [permission() for permission in permission_classes]

    def perform_create(self, serializer):
        serializer.save(author=self.request.user)

    @action(detail=True, methods=['post'])
    def publish(self, request, slug=None):
        post = self.get_object()
        post.is_published = True
        post.save(update_fields=['is_published', 'updated_at'])
        return Response(self.get_serializer(post).data)

    @action(detail=True, methods=['post'])
    def unpublish(self, request, slug=None):
        post = self.get_object()
        post.is_published = False
        post.save(update_fields=['is_published', 'updated_at'])
        return Response(self.get_serializer(post).data)

    @action(detail=False)
    def mine(self, request):
        queryset = self.get_queryset()
        serializer = self.get_serializer(queryset, many=True)
        return Response(serializer.data)


class CommentViewSet(viewsets.ModelViewSet):
    """nested resource ของ Post ผ่าน post_pk (ขั้นตอนที่ 435)"""
    serializer_class = CommentSerializer
    permission_classes = [IsAuthenticatedOrReadOnly, IsOwnerOrReadOnly]

    def get_queryset(self):
        return Comment.objects.filter(post_id=self.kwargs['post_pk'])

    def perform_create(self, serializer):
        serializer.save(post_id=self.kwargs['post_pk'], author=self.request.user)
```

### 440.2 `blog/api_urls.py` ฉบับสมบูรณ์

```python
# blog/api_urls.py
from django.urls import include, path
from rest_framework.routers import DefaultRouter
from rest_framework_nested import routers as nested_routers
from .viewsets import CategoryViewSet, CommentViewSet, PostViewSet

router = DefaultRouter()
router.register('posts', PostViewSet, basename='post')
router.register('categories', CategoryViewSet, basename='category')

posts_router = nested_routers.NestedDefaultRouter(router, 'posts', lookup='post')
posts_router.register('comments', CommentViewSet, basename='post-comments')

app_name = 'blog_api'

urlpatterns = [
    path('', include(router.urls)),
    path('', include(posts_router.urls)),
]
```

Blog API ทั้งระบบตอนนี้มี 3 ViewSet, 1 permission class ที่ใช้ซ้ำ, และ URL config
เพียง 10 บรรทัด ครอบคลุมทุก endpoint ของ `Post`, `Category`, `Comment` รวมถึง custom
action `publish`/`unpublish`/`mine` — เทียบกับถ้าเขียนแบบ `APIView` ล้วน ๆ ตาม Part 042
จะต้องมีอย่างน้อย 6-8 คลาสแยกกัน และ `path()` มากกว่า 10 เส้นทาง

### 440.3 สรุปสิ่งที่ได้เรียนรู้ใน Part นี้

- ✅ เข้าใจว่า `ViewSet` ผูกกับ **action** (`list`/`create`/`retrieve`/`update`/
  `partial_update`/`destroy`) แทนที่จะผูกกับ HTTP method ตรง ๆ แบบ `APIView`
- ✅ เจาะกลไก `ViewSetMixin.as_view()` ว่า "สับสาย" method ชื่อ action ให้กลายเป็นชื่อ
  HTTP verb แล้วส่งต่อให้ `APIView.dispatch()` เดิมทำงานตามปกติ
- ✅ ใช้ `ModelViewSet` เพื่อรวม CRUD เต็มรูปแบบไว้คลาสเดียว และรู้ว่ามันคือการผสม Mixin
  5 ตัวเข้ากับ `GenericViewSet`
- ✅ ใช้ `Router` (`DefaultRouter`/`SimpleRouter`) สร้าง URL ให้อัตโนมัติ และรู้ความ
  แตกต่างเรื่อง API Root View และ format suffix
- ✅ เพิ่ม custom action ด้วย `@action(detail=True/False)` พร้อม `url_path`,
  `permission_classes` เฉพาะ action
- ✅ ทำ nested resource (`/posts/{id}/comments/`) ด้วย `drf-nested-routers`
- ✅ ใช้ `ReadOnlyModelViewSet` สำหรับ endpoint ที่ให้อ่านอย่างเดียว และเข้าใจว่า
  write endpoint ไม่มี route อยู่จริงเลย ไม่ใช่แค่ถูกปฏิเสธด้วย permission
- ✅ override `get_queryset()`/`get_serializer_class()` แยกตาม `self.action`
- ✅ เปรียบเทียบ `APIView` vs Generic View vs `ViewSet` และรู้ว่าเมื่อไหร่ควรเลือกอะไร
- ✅ override `get_permissions()` เพื่อกำหนด permission ที่ต่างกันในแต่ละ action
  ของ ViewSet เดียว
- ✅ ประกอบ Blog API เต็มรูปแบบสำหรับ `Post`, `Category`, `Comment` ด้วย ViewSet +
  Router ล้วน ๆ

### 440.4 Checklist ก่อนไป Part ถัดไป

- [ ] อธิบายได้ว่า `ViewSetMixin.as_view({...})` ทำอะไรกับ dict ที่รับเข้ามา และทำไม
      สุดท้ายยังคงเรียก `APIView.dispatch()` เดิมอยู่
- [ ] เขียน `ModelViewSet` เต็มรูปแบบสำหรับ Model หนึ่งตัวได้เองโดยไม่ต้องเปิดเอกสาร
- [ ] อธิบายความแตกต่างระหว่าง `DefaultRouter` กับ `SimpleRouter` ได้ พร้อมบอกได้ว่า
      ควรเลือกใช้ตัวไหนในสถานการณ์ไหน
- [ ] เพิ่ม custom action ด้วย `@action` ทั้งแบบ `detail=True` และ `detail=False` ได้
- [ ] ตั้งค่า nested router ด้วย `drf-nested-routers` สำหรับ resource ที่มีความสัมพันธ์
      แบบ parent-child ได้
- [ ] อธิบายได้ว่าทำไม `ReadOnlyModelViewSet` ปลอดภัยกว่าการใช้ `ModelViewSet` แล้วปิด
      permission การเขียนด้วยมือ
- [ ] override `get_queryset()` และ `get_permissions()` โดยใช้ `self.action` เป็นเงื่อนไข
      ได้อย่างถูกต้อง พร้อมรู้ว่าต้องคืน permission เป็น instance ไม่ใช่ class
- [ ] เลือกระหว่าง `APIView`, Generic View, และ `ViewSet` ได้อย่างมีเหตุผลตามสถานการณ์
      โดยไม่ต้องเปิดตารางเปรียบเทียบดู

### 440.5 แบบฝึกหัดท้ายบท

**แบบฝึกหัดที่ 1 (พื้นฐาน)**: สร้าง `TagViewSet` เป็น `ReadOnlyModelViewSet` สำหรับโมเดล
`Tag` จาก Part 041 (`fields = ['id', 'name', 'slug']`) แล้วลงทะเบียนกับ `DefaultRouter`
ที่ path `tags/` ทดสอบว่า `GET /api/tags/` ทำงานได้ และ `POST /api/tags/` คืน 405

**แบบฝึกหัดที่ 2 (ประยุกต์)**: เพิ่ม custom action `@action(detail=False)` ชื่อ
`by_category` บน `PostViewSet` ที่รับ query parameter `?category=<slug>` แล้วคืนเฉพาะ
บทความในหมวดหมู่นั้น (ใช้ `request.query_params.get('category')` ที่เรียนจาก Part 042
ขั้นตอนที่ 414.4) ถ้าไม่ส่ง query parameter มาให้คืน 400 พร้อมข้อความอธิบาย

**แบบฝึกหัดที่ 3 (Nested Router)**: ต่อยอด `CommentViewSet` ให้มี custom action
`@action(detail=True, methods=['post'])` ชื่อ `report` สำหรับให้ผู้ใช้รายงานความคิดเห็นที่
ไม่เหมาะสม (สมมติแค่คืน `{"status": "reported"}` โดยไม่ต้องสร้าง model ใหม่ก็ได้) แล้ว
ทดสอบว่า URL ที่ได้คือ `/api/posts/{post_pk}/comments/{pk}/report/` ตามรูปแบบ nested
ที่เรียนในขั้นตอนที่ 435

**แบบฝึกหัดที่ 4 (ขั้นสูง)**: เขียน permission class ใหม่ชื่อ `IsStaffOrReadOnlyForDraft`
ที่ใช้ร่วมกับ `get_permissions()` ของ `PostViewSet`: ผู้ใช้ทั่วไป (ไม่ใช่ staff และไม่ใช่
เจ้าของ) ต้องเห็นได้แค่บทความที่ `is_published=True` เท่านั้นแม้จะเข้าถึงผ่าน
`retrieve()` โดยตรงด้วย pk/slug ที่รู้อยู่แล้ว (ไม่ใช่แค่กรองใน `get_queryset()` ของ
`list()`) ทดสอบว่าถ้า user ทั่วไปพยายามเข้าถึง draft ของคนอื่นโดยรู้ slug ตรง ๆ จะได้
404 (ไม่ใช่ 403 เพื่อไม่ให้รู้ว่าบทความนั้นมีอยู่จริง) — ใช้เทคนิค `get_queryset()`
กรองที่ database level ร่วมกับ `has_object_permission()`

### 440.6 คำถามที่พบบ่อย (FAQ)

**Q: ควรใช้ `ViewSet` เป็นค่าเริ่มต้นสำหรับทุก endpoint ในโปรเจกต์เลยหรือไม่?**
A: ไม่ควร ใช้ `ViewSet`/`ModelViewSet` สำหรับ endpoint ที่เป็น CRUD ของ resource ชัดเจน
(ตารางในขั้นตอนที่ 438.4) ส่วน endpoint ที่ไม่ผูกกับ resource ใด เช่น healthcheck,
webhook, endpoint รวมสถิติข้ามหลาย model ยังคงใช้ `APIView`/`@api_view` ได้เหมาะสมกว่า
เพราะ `ViewSet` ถูกออกแบบมาโดยมีสมมติฐานว่ามันแทน "resource" หนึ่งตัวเสมอ

**Q: ทำไมโค้ดใน `get()`/`post()` ของ `APIView` (Part 042) กับ `list()`/`create()` ของ
`ViewSet` ถึงดูคล้ายกันมาก แค่เปลี่ยนชื่อ method?**
A: เพราะโดยกลไกภายในแล้วมันคือระบบเดียวกัน — `ViewSet` แค่เพิ่มชั้น `ViewSetMixin` ที่
"สับสาย" ชื่อ method (ขั้นตอนที่ 431.6) ก่อนส่งต่อให้ `APIView.dispatch()` เดิมทำงาน
โค้ด business logic ภายในจึงเหมือนกันได้ทุกประการ ต่างกันแค่ "เปลือกนอก" ที่ห่ออยู่

**Q: `Router` รองรับ `lookup_field` ที่ไม่ใช่ `pk`/`slug` เช่น `uuid` ได้หรือไม่?**
A: ได้ ตั้ง `lookup_field = 'uuid'` และถ้าต้องการ custom regex pattern ของ URL segment
นั้นด้วย ให้ตั้ง `lookup_value_regex` เพิ่มเติมใน ViewSet เช่น
`lookup_value_regex = '[0-9a-f-]{36}'` สำหรับ UUID มาตรฐาน — Router จะอ่านค่าทั้งสองนี้
จาก ViewSet ไปสร้าง URL pattern ให้อัตโนมัติ

**Q: ถ้าลืมประกาศ `permission_classes` ทั้งใน class attribute และ `get_permissions()`
จะเกิดอะไรขึ้น?**
A: DRF จะ fallback ไปใช้ค่า `DEFAULT_PERMISSION_CLASSES` ที่ตั้งไว้ใน setting
`REST_FRAMEWORK` ระดับโปรเจกต์ (ทบทวนจาก Part 039) ซึ่งโดยทั่วไปมักตั้งเป็น
`AllowAny` หรือ `IsAuthenticatedOrReadOnly` เป็นค่าเริ่มต้น — ควรตรวจสอบ setting นี้
เสมอเพื่อไม่ให้ endpoint ที่ควรจำกัดสิทธิ์เปิดกว้างเกินไปโดยไม่ตั้งใจ

---

## เตรียมตัวสำหรับ Part ถัดไป

**Part 045: DRF Permissions และ Authentication** จะเจาะลึกระบบ Permission และ
Authentication ของ DRF แบบเต็มรูปแบบ ที่ Part นี้แตะไปเพียงผิวเผินผ่าน
`IsAuthenticatedOrReadOnly`, `IsOwnerOrReadOnly` และ `get_permissions()` คุณจะได้เรียนรู้
`AuthenticationClasses` ทั้งหมดที่ DRF มีให้ (`SessionAuthentication`,
`BasicAuthentication`), เขียน custom `Permission` class ตั้งแต่ต้นแบบเจาะลึกกลไก
`has_permission()`/`has_object_permission()`, และเข้าใจว่าทำไม endpoint บางจุดคืน 401
บางจุดคืน 403 ต่างกันอย่างไร ก่อนที่ Part 046 จะพาไปสู่ Authentication ขั้นสูงอย่าง JWT
และ OAuth2

เตรียมเปิด `blog/viewsets.py`, `blog/permissions.py`, และ `blog/api_urls.py` ของคุณไว้
ให้พร้อม เพราะ Part 045 จะแก้ไขไฟล์เหล่านี้ต่อจากที่ Part นี้ทิ้งไว้ทันที!
</content>
