# Part 042: API Views: Function-Based และ APIView

> **ขั้นตอนที่ 411-420 ของหลักสูตร** | Phase 5: Django REST Framework และ API
>
> เป้าหมายของ Part นี้: เข้าใจสองวิธีหลักในการเขียน API View ด้วย Django REST Framework
> (DRF) คือ `@api_view` (แบบ function-based) และ `APIView` (แบบ class-based) อย่างถ่องแท้
> ระดับกลไกภายใน โดยเฉพาะการเจาะลึก `dispatch()` ของ `APIView` เทียบกับ `View` ธรรมดาของ
> Django ที่เรียนไปแล้วใน Part 021 คุณจะเข้าใจว่า `request.data`, `Response`, Content
> Negotiation และ Exception Handling ของ DRF ทำงานอย่างไรเบื้องหลัง และปิดท้ายด้วยการสร้าง
> CRUD API เต็มรูปแบบสำหรับ `Post` ด้วย `APIView` ล้วน ๆ (ยังไม่ใช้ Generic Views ซึ่งจะเรียน
> เต็มรูปแบบใน Part 043) พร้อมทดสอบจริงด้วย `curl` และ `httpie` จาก command line

---

## สารบัญของ Part นี้

- ขั้นตอนที่ 411: `@api_view` decorator — เขียน API แบบ function-based (FBV) ของ DRF
- ขั้นตอนที่ 412: `APIView` class — เจาะลึก `dispatch()` mechanism เทียบกับ Django `View` ธรรมดาจาก Part 021
- ขั้นตอนที่ 413: จัดการ GET/POST/PUT/PATCH/DELETE ใน `APIView` เดียวกัน
- ขั้นตอนที่ 414: `request.data` เทียบกับ `request.POST`/`request.body` ของ Django ธรรมดา
- ขั้นตอนที่ 415: `Response` object และ Content Negotiation เจาะลึกกว่า Part 039
- ขั้นตอนที่ 416: Exception Handling ใน DRF — `APIException`, custom exception handler
- ขั้นตอนที่ 417: การเลือกใช้ HTTP Status Code ที่ถูกต้องตามหลัก REST
- ขั้นตอนที่ 418: สร้าง CRUD เต็มรูปแบบด้วย `APIView` ธรรมดาสำหรับ `Post`
- ขั้นตอนที่ 419: ทดสอบ API ด้วย `curl`/`httpie` จาก command line
- ขั้นตอนที่ 420: สรุปและแบบฝึกหัด

---

## ขั้นตอนที่ 411: `@api_view` decorator — เขียน API แบบ function-based (FBV) ของ DRF

### 411.1 ทบทวนเส้นทางที่พาเรามาถึงจุดนี้

ก่อนเข้าเนื้อหาใหม่ มาไล่ทบทวนสั้น ๆ ว่าเรามาถึงจุดนี้ได้อย่างไร:

- **Part 007**: เขียน View แบบ function-based (FBV) ธรรมดาที่คืน `HttpResponse`/`JsonResponse`
- **Part 021**: เจาะลึก Class-Based View (CBV) ของ Django เอง — `View.as_view()`,
  `View.dispatch()` และกลไกการเลือก method ตาม `request.method`
- **Part 039**: ติดตั้ง Django REST Framework (`pip install djangorestframework`), เพิ่ม
  `'rest_framework'` ใน `INSTALLED_APPS`, และรู้จัก `Response` object แบบเบื้องต้น
- **Part 040-041**: สร้าง `Serializer`/`ModelSerializer` เพื่อแปลง Model เป็น JSON และกลับกัน
  รวมถึง `PostSerializer` สำหรับโมเดล `Post` ของแอป `blog`

Part นี้จะเอาความรู้ทั้งหมดข้างต้นมาประกอบร่างเป็น **API View ที่ใช้งานได้จริง** โดยเริ่มจาก
วิธีที่ง่ายและใกล้เคียง FBV เดิมที่สุดก่อน นั่นคือ `@api_view` decorator

### 411.2 ทบทวนโมเดลและ Serializer ที่จะใช้ตลอด Part นี้

เพื่อให้ทุกตัวอย่างในทั้ง 10 ขั้นตอนนี้ทำงานร่วมกันได้จริง เราจะยึด `Post` model จากแอป
`blog` (ต่อยอดจาก Part 007/011/012) และ `PostSerializer` จาก Part 041 เป็นฐานร่วมกัน:

```python
# blog/models.py
from django.conf import settings
from django.db import models
from django.utils.text import slugify


class Post(models.Model):
    author = models.ForeignKey(
        settings.AUTH_USER_MODEL,
        on_delete=models.CASCADE,
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
```

```python
# blog/serializers.py
from rest_framework import serializers
from .models import Post


class PostSerializer(serializers.ModelSerializer):
    author = serializers.ReadOnlyField(source='author.username')

    class Meta:
        model = Post
        fields = [
            'id', 'author', 'title', 'slug',
            'content', 'is_published', 'created_at', 'updated_at',
        ]
        read_only_fields = ['id', 'slug', 'created_at', 'updated_at']
```

ตลอด Part นี้ เราจะสร้างไฟล์ `blog/api_views.py` แยกจาก `blog/views.py` เดิม (ที่ใช้กับ
Template-based View จาก Part 007/021-024) เพื่อไม่ให้โค้ด HTML view กับ API view ปนกัน
และเชื่อมเข้า `config/urls.py` ผ่าน prefix `/api/` ตามแบบแผนที่วางไว้ตั้งแต่ Part 039

### 411.3 ปัญหาของการใช้ FBV ธรรมดา (แบบ Part 007) กับ JSON API

ก่อนเห็นว่า `@api_view` ช่วยอะไร มาดูก่อนว่าถ้าไม่มี DRV เลย เขียน JSON API ด้วย FBV ธรรมดา
จาก Part 007 จะเจอปัญหาอะไรบ้าง:

```python
# ตัวอย่าง FBV ธรรมดาที่ "พยายาม" ทำ JSON API โดยไม่ใช้ DRF
import json
from django.http import JsonResponse, HttpResponseNotAllowed
from .models import Post


def post_list_plain(request):
    if request.method == 'GET':
        posts = Post.objects.filter(is_published=True)
        data = [
            {'id': p.id, 'title': p.title, 'slug': p.slug}
            for p in posts
        ]
        return JsonResponse(data, safe=False)

    elif request.method == 'POST':
        # ปัญหาที่ 1: request.POST ไม่รู้จัก JSON body เลย ต้อง parse เอง
        try:
            payload = json.loads(request.body)
        except json.JSONDecodeError:
            return JsonResponse({'error': 'invalid JSON'}, status=400)

        # ปัญหาที่ 2: validation ต้องเขียนเองทั้งหมด ไม่มี Serializer ช่วย
        title = payload.get('title')
        if not title:
            return JsonResponse({'error': 'title is required'}, status=400)

        post = Post.objects.create(title=title, content=payload.get('content', ''))
        return JsonResponse({'id': post.id, 'title': post.title}, status=201)

    else:
        return HttpResponseNotAllowed(['GET', 'POST'])
```

ปัญหาที่พบเจอ:

| ปัญหา | รายละเอียด |
|---|---|
| ไม่มี Content Negotiation | คืน JSON เสมอ ไม่ว่า client จะขอ format อะไร (ไม่รองรับ Browsable API) |
| Body parsing ต้องเขียนเอง | `json.loads(request.body)` ต้อง try/except เอง ทุก view |
| ไม่มี Validation Layer มาตรฐาน | เขียน `if not title: ...` เองทุกฟิลด์ ไม่มี Serializer ช่วย |
| Error format ไม่สม่ำเสมอ | แต่ละ view คืน error shape ต่างกันได้ตามใจนักพัฒนา |
| Exception ที่ไม่ได้ดักไว้ | ถ้าเกิด `Exception` ที่ไม่คาดคิด Django จะคืนหน้า HTML error 500 ให้ client ที่คาดหวัง JSON |
| ไม่มี authentication/permission framework | ต้องเขียน decorator ตรวจสิทธิ์เองทุกจุด |

DRF ถูกออกแบบมาเพื่อแก้ปัญหาทั้งหมดนี้แบบรวมศูนย์ และ `@api_view` คือประตูที่ง่ายที่สุดใน
การเข้าสู่โลกของ DRF โดยยังคงความรู้สึกแบบ "เขียนฟังก์ชันเดียว" ของ FBV ไว้เหมือนเดิม

### 411.4 `@api_view`: จุดเริ่มต้นที่ง่ายที่สุดในการเข้าสู่ DRF

```python
# blog/api_views.py
from rest_framework.decorators import api_view
from rest_framework.response import Response
from .models import Post
from .serializers import PostSerializer


@api_view(['GET'])
def post_list(request):
    posts = Post.objects.filter(is_published=True)
    serializer = PostSerializer(posts, many=True)
    return Response(serializer.data)
```

```python
# blog/api_urls.py
from django.urls import path
from . import api_views

app_name = 'blog_api'

urlpatterns = [
    path('posts/', api_views.post_list, name='post-list'),
]
```

```python
# config/urls.py
from django.contrib import admin
from django.urls import path, include

urlpatterns = [
    path('admin/', admin.site.urls),
    path('api/', include('blog.api_urls')),
    # ... urlpatterns เดิมของ Template views จาก Part 007-030
]
```

เปิด `http://127.0.0.1:8000/api/posts/` ด้วยเบราว์เซอร์ คุณจะเห็นหน้า **Browsable API**
ของ DRF ทันที (หน้า HTML สวยงามที่แสดงข้อมูล JSON พร้อมปุ่มทดสอบ) — นี่คือผลของ Content
Negotiation ที่จะเจาะลึกในขั้นตอนที่ 415

### 411.5 เจาะโค้ดจริง (แบบย่อ) ของ `api_view` decorator

คำถามสำคัญ: `@api_view(['GET'])` ทำอะไรกับฟังก์ชัน `post_list` กันแน่ ถึงทำให้ฟังก์ชัน
ธรรมดากลายเป็น "DRF-aware" ได้ทันที? คำตอบอยู่ในซอร์สโค้ดของ
`rest_framework/decorators.py` (ย่อเพื่อการศึกษา โครงสร้างตรงกับของจริง):

```python
# แนวคิดจากซอร์สโค้ดจริงของ rest_framework.decorators.api_view
from rest_framework.views import APIView


def api_view(http_method_names=None):
    http_method_names = ['GET'] if http_method_names is None else http_method_names

    def decorator(func):
        # (1) สร้าง class ใหม่ขึ้นมาแบบ dynamic โดยสืบทอดจาก APIView โดยตรง!
        WrappedAPIView = type(
            'WrappedAPIView',
            (APIView,),
            {'__doc__': func.__doc__},
        )

        WrappedAPIView.http_method_names = [
            method.lower() for method in http_method_names
        ]

        # (2) สร้าง handler ที่แค่ "ส่งต่อ" ไปเรียกฟังก์ชันเดิมของเรา
        def handler(self, *args, **kwargs):
            return func(*args, **kwargs)

        # (3) ผูก handler นี้เข้ากับทุก method ที่ประกาศไว้ (GET -> get, POST -> post, ...)
        for method in http_method_names:
            setattr(WrappedAPIView, method.lower(), handler)

        WrappedAPIView.__name__ = func.__name__
        WrappedAPIView.__module__ = func.__module__

        # (4) คัดลอก attribute พิเศษที่อาจแปะไว้กับฟังก์ชัน (จาก decorator อื่นซ้อนกัน)
        WrappedAPIView.renderer_classes = getattr(
            func, 'renderer_classes', APIView.renderer_classes)
        WrappedAPIView.parser_classes = getattr(
            func, 'parser_classes', APIView.parser_classes)
        WrappedAPIView.authentication_classes = getattr(
            func, 'authentication_classes', APIView.authentication_classes)
        WrappedAPIView.throttle_classes = getattr(
            func, 'throttle_classes', APIView.throttle_classes)
        WrappedAPIView.permission_classes = getattr(
            func, 'permission_classes', APIView.permission_classes)

        # (5) คืน callable ที่ path() ต้องการ — เหมือนที่ View.as_view() ทำใน Part 021!
        return WrappedAPIView.as_view()

    return decorator
```

**นี่คือกุญแจสำคัญที่สุดของขั้นตอนนี้**: `@api_view` **ไม่ได้สร้างกลไกใหม่ขึ้นมาเอง** แต่
สร้าง **`APIView` subclass แบบ dynamic** (ด้วย `type()` — เทคนิคเดียวกับที่ใช้สร้าง class
runtime ที่คุณอาจเคยเห็นใน Part 002 เรื่อง metaclass เบื้องต้น) แล้วเรียก `.as_view()` ของ
มันเหมือนที่ CBV ทุกตัวทำ ฟังก์ชัน `post_list` ที่เราเขียนจึงถูกแปลงร่างเป็น method `get()`
ของ class ที่สืบทอดจาก `APIView` ในที่สุด — พูดอีกแบบคือ **`@api_view` ไม่ใช่ทางเลือกที่
"ต่างจาก" `APIView` แต่เป็นแค่ทางลัดในการสร้าง `APIView` โดยไม่ต้องเขียน class เอง**
เราจะเจาะกลไกของ `APIView` เต็มรูปแบบในขั้นตอนที่ 412 ถัดไป

### 411.6 รองรับหลาย HTTP method ในฟังก์ชันเดียว

```python
# blog/api_views.py
from rest_framework import status
from rest_framework.decorators import api_view
from rest_framework.response import Response
from .models import Post
from .serializers import PostSerializer


@api_view(['GET', 'POST'])
def post_list(request):
    if request.method == 'GET':
        posts = Post.objects.filter(is_published=True)
        serializer = PostSerializer(posts, many=True)
        return Response(serializer.data)

    elif request.method == 'POST':
        serializer = PostSerializer(data=request.data)
        if serializer.is_valid():
            serializer.save(author=request.user)
            return Response(serializer.data, status=status.HTTP_201_CREATED)
        return Response(serializer.errors, status=status.HTTP_400_BAD_REQUEST)
```

สังเกตว่าเรายังคงเขียน `if/elif` ตาม `request.method` เหมือน FBV แบบ Part 007 ทุกประการ
— `@api_view` ไม่ได้เปลี่ยนวิธีจัดการหลาย method ภายในฟังก์ชันเดียว มันแค่ครอบฟังก์ชันด้วย
เครื่องมือของ DRF (parsing, content negotiation, exception handling) เท่านั้น

ทดสอบว่า method ที่ไม่ได้ประกาศไว้ใน `@api_view([...])` ถูกปฏิเสธอัตโนมัติ:

```python
>>> from django.test import RequestFactory
>>> from blog.api_views import post_list
>>> factory = RequestFactory()

>>> request = factory.delete('/api/posts/')
>>> response = post_list(request)
>>> response.status_code
405
>>> response.data
{'detail': ErrorDetail(string='Method "DELETE" not allowed.', code='method_not_allowed')}
```

สังเกตว่าเราได้ **405 พร้อม error message มาตรฐานของ DRF** โดยไม่ต้องเขียน
`HttpResponseNotAllowed` เองเลย นี่คือประโยชน์อีกข้อของการห่อฟังก์ชันด้วย `@api_view`

### 411.7 ตารางสรุป: FBV ธรรมดา (Part 007) เทียบกับ `@api_view` (DRF)

| คุณสมบัติ | FBV ธรรมดา (Part 007) | `@api_view` (DRF) |
|---|---|---|
| Body parsing | ต้อง `json.loads(request.body)` เอง | `request.data` parse ให้อัตโนมัติ (ขั้นตอนที่ 414) |
| Response object | `JsonResponse` เท่านั้น | `Response` รองรับ content negotiation (ขั้นตอนที่ 415) |
| Method ที่ไม่รองรับ | ต้องเขียน `HttpResponseNotAllowed` เอง | 405 อัตโนมัติจากรายชื่อใน `@api_view([...])` |
| Exception ที่ไม่คาดคิด | Django คืนหน้า HTML 500 | DRF ดักและแปลงเป็น JSON error (ขั้นตอนที่ 416) |
| Browsable API | ไม่มี | มีให้อัตโนมัติ |
| Authentication/Permission | เขียน decorator เอง | ใช้ `@permission_classes`, `@authentication_classes` (Part 045) |
| ความรู้สึกตอนเขียน | ฟังก์ชันเดียว, ตรงไปตรงมา | ฟังก์ชันเดียวเหมือนเดิม แค่เพิ่ม decorator |

### 411.8 `@permission_classes`, `@throttle_classes` ฯลฯ: decorator เสริมของ `@api_view`

DRF มี decorator เสริมอีกชุดหนึ่งที่ใช้คู่กับ `@api_view` เพื่อกำหนดค่าที่ปกติเป็น class
attribute ของ `APIView` (เราจะเจาะลึก permission/authentication เต็มรูปแบบใน Part 045
ตอนนี้ขอให้เห็นว่า decorator เหล่านี้มีอยู่และวางซ้อนกับ `@api_view` ได้อย่างไร):

```python
from rest_framework.decorators import api_view, permission_classes
from rest_framework.permissions import IsAuthenticatedOrReadOnly


@api_view(['GET', 'POST'])
@permission_classes([IsAuthenticatedOrReadOnly])
def post_list(request):
    ...
```

**ข้อควรระวัง**: `@api_view` ต้องอยู่ **บนสุด** (นอกสุด) เสมอเมื่อวาง decorator ซ้อนกัน
เพราะมันคือตัวที่แปลงฟังก์ชันให้เป็น `APIView` subclass ก่อน แล้ว decorator อื่น ๆ อย่าง
`@permission_classes` ค่อยแปะ attribute เพิ่มเข้าไปบนฟังก์ชันนั้น (ตามที่เห็นในซอร์สโค้ด
ขั้นตอนที่ 411.5 ข้อ 4 ที่ `WrappedAPIView` อ่านค่าจาก `getattr(func, 'permission_classes', ...)`)

### 411.9 ข้อจำกัดของ `@api_view` เมื่อ endpoint ซับซ้อนขึ้น

`@api_view` เหมาะกับ endpoint ที่มี logic ไม่ซับซ้อนมาก แต่เมื่อ endpoint ต้องการ:

- แชร์ logic บางส่วนร่วมกันระหว่างหลาย method (เช่น `get_object()` ที่ใช้ทั้ง GET/PUT/DELETE)
- Override behavior เฉพาะจุด เช่น `get_queryset()`, `perform_create()` (จะเจอเต็มรูปแบบใน
  Part 043)
- จัดกลุ่ม method หลายตัวให้อยู่ใน object เดียวเพื่อทดสอบและจัดระเบียบง่ายกว่า

โค้ดแบบฟังก์ชันเดียวจะเริ่มยาวและอ่านยากขึ้นเรื่อย ๆ นี่คือจุดที่ **`APIView` class** เข้ามา
มีบทบาท ซึ่งเราจะเจาะลึกกลไกเบื้องหลังทั้งหมดในขั้นตอนถัดไป

---

## ขั้นตอนที่ 412: `APIView` class — เจาะลึก `dispatch()` mechanism เทียบกับ Django `View` ธรรมดาจาก Part 021

### 412.1 ทบทวน `django.views.View.dispatch()` จาก Part 021 ขั้นตอนที่ 203

ก่อนเจาะ `APIView` มาทวนโครงของ Django `View.dispatch()` แบบเต็ม ๆ อีกครั้ง (จาก Part 021
ขั้นตอนที่ 203.1):

```python
# django.views.generic.base.View (ทบทวนจาก Part 021)
class View:
    http_method_names = [
        'get', 'post', 'put', 'patch', 'delete', 'head', 'options', 'trace',
    ]

    def dispatch(self, request, *args, **kwargs):
        if request.method.lower() in self.http_method_names:
            handler = getattr(
                self, request.method.lower(), self.http_method_not_allowed
            )
        else:
            handler = self.http_method_not_allowed
        return handler(request, *args, **kwargs)
```

สั้น ตรงไปตรงมา: เลือก method จาก `request.method` แล้วเรียกมันตรง ๆ ไม่มีขั้นตอนอื่นแทรก
เลย นี่คือสิ่งที่เพียงพอสำหรับ view ที่คืน HTML แต่ **ไม่เพียงพอสำหรับ API** ที่ต้องการ
การตรวจสอบสิทธิ์, การแปลง request/response format, และการดักจับ exception ให้เป็นมาตรฐาน
เดียวกันทุก endpoint — นี่คือเหตุผลที่ DRF ไม่ใช้ `View.dispatch()` ตรง ๆ แต่ **override
`dispatch()` ใหม่ทั้งหมด** ใน `APIView`

### 412.2 `APIView` สืบทอดจาก `View` ของ Django โดยตรง

ข้อเท็จจริงที่สำคัญที่สุดของขั้นตอนนี้: **`rest_framework.views.APIView` ไม่ได้เป็นระบบ
คู่ขนานที่แยกจาก Django CBV แต่เป็น subclass ของ `django.views.generic.base.View`
โดยตรง**:

```python
# แนวคิดจากซอร์สโค้ดจริงของ rest_framework.views
from django.views.generic import View
from django.views.decorators.csrf import csrf_exempt


class APIView(View):
    renderer_classes = api_settings.DEFAULT_RENDERER_CLASSES
    parser_classes = api_settings.DEFAULT_PARSER_CLASSES
    authentication_classes = api_settings.DEFAULT_AUTHENTICATION_CLASSES
    throttle_classes = api_settings.DEFAULT_THROTTLE_CLASSES
    permission_classes = api_settings.DEFAULT_PERMISSION_CLASSES
    content_negotiation_class = api_settings.DEFAULT_CONTENT_NEGOTIATION_CLASS
```

นี่คือเหตุผลที่ Part 021 ขั้นตอนที่ 201.4 (ตารางสรุป) พูดล่วงหน้าไว้ว่า *"Django REST
Framework (DRF): CBV เกือบทั้งหมด (`APIView`, `ViewSet`) — สืบทอดแนวคิดเดียวกับ `View`
ของ Django แต่ปรับสำหรับ API"* — ตอนนี้เราเห็นแล้วว่า "สืบทอดแนวคิดเดียวกับ" หมายถึง
**สืบทอด class จริง ๆ** ไม่ใช่แค่คล้ายกันโดยบังเอิญ ทุกอย่างที่เรียนใน Part 021 เรื่อง
`as_view()` สร้าง instance ใหม่ทุก request, thread safety ฯลฯ ยังคงเป็นจริงกับ `APIView`
ทุกประการ เพราะมันคือ `View` ตัวเดิมที่ถูกขยายความสามารถ

### 412.3 `APIView.as_view()`: เพิ่มอะไรจาก `View.as_view()` ของ Part 021

```python
# แนวคิดจากซอร์สโค้ดจริงของ rest_framework.views.APIView
class APIView(View):
    @classmethod
    def as_view(cls, **initkwargs):
        # (1) เรียก View.as_view() ของ Django เดิมก่อน (Part 021 ขั้นตอนที่ 202)
        #     ได้ฟังก์ชัน `view` แบบเดียวกับที่เรียนมาทุกประการ
        view = super().as_view(**initkwargs)

        view.cls = cls
        view.initkwargs = initkwargs

        # (2) ครอบด้วย csrf_exempt() เพิ่มอีกชั้นหนึ่ง!
        return csrf_exempt(view)
```

ความแตกต่างจุดแรก: **`APIView.as_view()` ครอบผลลัพธ์ด้วย `csrf_exempt()` เสมอ** เหตุผล
คือ API ที่ใช้ Token/JWT authentication (จะเรียนใน Part 046) ไม่ได้พึ่งพา session cookie
แบบ Django ปกติ จึงไม่จำเป็นต้องมี CSRF token แบบฟอร์ม HTML — DRF จัดการเรื่อง CSRF ของ
ตัวเองแยกต่างหาก (เฉพาะตอนใช้ `SessionAuthentication` เท่านั้นที่ยังตรวจ CSRF อยู่ ซึ่งจะ
อธิบายใน Part 045)

### 412.4 `APIView.dispatch()`: หัวใจของ Part นี้

นี่คือส่วนสำคัญที่สุด มาดู `dispatch()` ฉบับเต็มของ `APIView` (ย่อเพื่อการศึกษา โครงสร้าง
ตรงกับซอร์สโค้ดจริง):

```python
# แนวคิดจากซอร์สโค้ดจริงของ rest_framework.views.APIView
class APIView(View):
    def dispatch(self, request, *args, **kwargs):
        self.args = args
        self.kwargs = kwargs

        # (1) แปลง Django HttpRequest ให้เป็น DRF Request (ขั้นตอนที่ 414)
        request = self.initialize_request(request, *args, **kwargs)
        self.request = request
        self.headers = self.default_response_headers

        try:
            # (2) รัน authentication, permission, throttle "ก่อน" เรียก handler
            self.initial(request, *args, **kwargs)

            # (3) เลือก handler ตาม HTTP method — โครงเดียวกับ View.dispatch() ของ Part 021
            if request.method.lower() in self.http_method_names:
                handler = getattr(
                    self, request.method.lower(), self.http_method_not_allowed
                )
            else:
                handler = self.http_method_not_allowed

            response = handler(request, *args, **kwargs)

        except Exception as exc:
            # (4) ดักทุก Exception ที่เกิดขึ้นระหว่าง initial() หรือ handler
            response = self.handle_exception(exc)

        # (5) เลือก renderer ที่ถูกต้องให้ response ก่อนส่งกลับ (ขั้นตอนที่ 415)
        self.response = self.finalize_response(request, response, *args, **kwargs)
        return self.response
```

### 412.5 เปรียบเทียบแบบคู่ขนาน: `View.dispatch()` vs `APIView.dispatch()`

| ขั้นตอน | `django.views.View.dispatch()` (Part 021) | `rest_framework.views.APIView.dispatch()` |
|---|---|---|
| แปลง request | ใช้ `HttpRequest` ตรง ๆ | ห่อเป็น `Request` object ผ่าน `initialize_request()` |
| ก่อนเลือก handler | ไม่มีขั้นตอนเพิ่มเติม | เรียก `self.initial()` (authentication + permission + throttle) |
| การเลือก handler | `getattr(self, request.method.lower(), self.http_method_not_allowed)` | เหมือนกันทุกประการ (โค้ดชุดเดียวกัน) |
| Error handling | ไม่มี try/except ครอบ — exception หลุดขึ้นไปให้ Django middleware จัดการ | ครอบด้วย `try/except Exception` ทั้งหมด แล้วส่งเข้า `handle_exception()` |
| หลังได้ response | คืนค่าตรง ๆ | เรียก `finalize_response()` เพื่อทำ content negotiation ก่อนคืนค่า |
| ผลลัพธ์สุดท้าย | `HttpResponse` ใด ๆ | `Response` ที่มี `.accepted_renderer` ผูกไว้แล้ว |

พูดให้กระชับที่สุด: **`APIView.dispatch()` คือ `View.dispatch()` ของ Part 021 ที่ถูก
"แซนด์วิช" ด้วยขั้นตอนเพิ่มอีก 3 ชั้น** — ชั้นก่อน (`initial()`), ชั้นครอบ (try/except),
และชั้นหลัง (`finalize_response()`) แก่นกลางที่เลือก method ยังเป็นโค้ดแบบเดียวกับที่เรียน
ใน Part 021 ทุกตัวอักษร

### 412.6 `self.initial()`: ประตูตรวจสอบก่อนเข้าถึง handler

```python
# แนวคิดจากซอร์สโค้ดจริงของ rest_framework.views.APIView
def initial(self, request, *args, **kwargs):
    self.format_kwarg = self.get_format_suffix(**kwargs)

    # Content negotiation: ตัดสินใจว่าจะ "ตอบกลับ" เป็น format ไหน (ขั้นตอนที่ 415)
    neg = self.perform_content_negotiation(request)
    request.accepted_renderer, request.accepted_media_type = neg

    # API Versioning (จะเรียนเต็มรูปแบบใน Part 048)
    version, scheme = self.determine_version(request, *args, **kwargs)
    request.version, request.versioning_scheme = version, scheme

    # Authentication: request.user ถูกกำหนดค่าจริงตรงนี้ (จะเรียนเต็มรูปแบบใน Part 045)
    self.perform_authentication(request)

    # Permission: เช็คว่า user มีสิทธิ์เข้าถึง endpoint นี้หรือไม่
    self.check_permissions(request)

    # Throttle: เช็ค rate limit (จะเรียนเต็มรูปแบบใน Part 048)
    self.check_throttles(request)
```

ข้อสังเกตสำคัญ: **`self.perform_authentication(request)` และ `self.check_permissions(request)`
ทำงาน "ก่อน" ที่ `handler` (เช่น `self.get()`) จะถูกเรียกเสมอ** นี่คือเหตุผลที่เขียน
`def get(self, request): ...` ใน `APIView` แล้วไม่ต้องเช็ค `if not request.user.is_authenticated`
เองในทุก method — `check_permissions()` จะ raise exception ให้ `dispatch()` ดักไว้ก่อนที่
handler จะถูกเรียกด้วยซ้ำ ถ้า permission ไม่ผ่าน (รายละเอียดเต็มรูปแบบใน Part 045)

### 412.7 `self.finalize_response()`: ขั้นตอนสุดท้ายก่อนส่งกลับ

```python
# แนวคิดจากซอร์สโค้ดจริงของ rest_framework.views.APIView
def finalize_response(self, request, response, *args, **kwargs):
    if isinstance(response, Response):
        if not getattr(request, 'accepted_renderer', None):
            neg = self.perform_content_negotiation(request, force=True)
            request.accepted_renderer, request.accepted_media_type = neg

        response.accepted_renderer = request.accepted_renderer
        response.accepted_media_type = request.accepted_media_type
        response.renderer_context = self.get_renderer_context()

    for key, value in self.headers.items():
        response[key] = value

    return response
```

`finalize_response()` ผูก renderer ที่เลือกไว้ตอน `initial()` เข้ากับ `response` object
จริง ๆ — นี่คือกลไกที่ทำให้ `Response` object "ยังไม่ render เป็น JSON/HTML จนกว่าจะถึง
จุดนี้" ซึ่งเป็นหัวใจของ Content Negotiation ที่เราจะเจาะลึกในขั้นตอนที่ 415

### 412.8 เขียน `PostListAPIView` ด้วย `APIView` (เทียบกับ `@api_view` จากขั้นตอนที่ 411)

```python
# blog/api_views.py
from rest_framework import status
from rest_framework.response import Response
from rest_framework.views import APIView
from .models import Post
from .serializers import PostSerializer


class PostListAPIView(APIView):
    def get(self, request):
        posts = Post.objects.filter(is_published=True)
        serializer = PostSerializer(posts, many=True)
        return Response(serializer.data)

    def post(self, request):
        serializer = PostSerializer(data=request.data)
        if serializer.is_valid():
            serializer.save(author=request.user)
            return Response(serializer.data, status=status.HTTP_201_CREATED)
        return Response(serializer.errors, status=status.HTTP_400_BAD_REQUEST)
```

```python
# blog/api_urls.py
from django.urls import path
from . import api_views

app_name = 'blog_api'

urlpatterns = [
    path('posts/', api_views.PostListAPIView.as_view(), name='post-list'),
]
```

สังเกตว่า **โค้ดในตัว `get()`/`post()` เหมือนกับตัวอย่าง `@api_view` ในขั้นตอนที่ 411.6
ทุกบรรทัด** — ความแตกต่างมีแค่เปลือกนอก: `@api_view` ให้เขียนเป็นฟังก์ชันเดียวแล้วใช้
`if/elif`, ส่วน `APIView` ให้แยกเป็น method ต่างหากเหมือน `View` ธรรมดาของ Part 021
ขั้นตอนที่ 203

### 412.9 ตารางสรุป: เมื่อไหร่ควรใช้ `@api_view` เมื่อไหร่ควรใช้ `APIView`

| สถานการณ์ | แนะนำใช้ |
|---|---|
| Endpoint ง่าย ๆ ที่ทำอย่างเดียว (เช่น health check, webhook รับ callback) | `@api_view` |
| ต้องการ logic ที่แชร์ระหว่างหลาย HTTP method (เช่น `get_object()` ใช้ร่วมกัน) | `APIView` |
| ต้องการ override behavior เช่น `get_permissions()`, `handle_exception()` เฉพาะ view นี้ | `APIView` |
| ทีมคุ้นเคยกับสไตล์ FBV จาก Part 007 มากกว่า | `@api_view` |
| Endpoint ที่จะขยายเป็น Generic View ในอนาคต (Part 043) | `APIView` (โครงสร้างใกล้เคียงกว่า) |
| ต้องการเขียน test แยกทดสอบทีละ method ได้ง่าย (`view_instance.get(request)`) | `APIView` |

ในทางปฏิบัติ โปรเจกต์ระดับมืออาชีพส่วนใหญ่ใช้ `APIView`/Generic Views เป็นหลัก และเก็บ
`@api_view` ไว้สำหรับ endpoint พิเศษเล็ก ๆ น้อย ๆ เท่านั้น (เช่น endpoint สำหรับ
health-check หรือ endpoint แบบ RPC-style ที่ไม่ตรง pattern ของ resource ใด ๆ)

---

## ขั้นตอนที่ 413: จัดการ GET/POST/PUT/PATCH/DELETE ใน `APIView` เดียวกัน

### 413.1 Endpoint ระดับ "รายการ" (list) vs ระดับ "รายตัว" (detail)

ตามหลัก REST มาตรฐาน API ของ resource หนึ่งตัวมักแยกเป็น 2 endpoint:

```
GET/POST    /api/posts/            → PostListAPIView (ขั้นตอนที่ 412)
GET/PUT/PATCH/DELETE  /api/posts/<slug>/  → PostDetailAPIView (ขั้นตอนนี้)
```

### 413.2 `PostDetailAPIView`: รองรับ 4 methods ในคลาสเดียว

```python
# blog/api_views.py
from django.shortcuts import get_object_or_404
from rest_framework import status
from rest_framework.response import Response
from rest_framework.views import APIView
from .models import Post
from .serializers import PostSerializer


class PostDetailAPIView(APIView):
    def get_object(self, slug):
        """
        Helper method ที่ใช้ร่วมกันทั้ง 4 handler — นี่คือข้อดีของ APIView
        เทียบกับการเขียนฟังก์ชันแยกแบบ @api_view ที่ทำแบบนี้ได้ยากกว่า
        """
        return get_object_or_404(Post, slug=slug)

    def get(self, request, slug):
        post = self.get_object(slug)
        serializer = PostSerializer(post)
        return Response(serializer.data)

    def put(self, request, slug):
        post = self.get_object(slug)
        serializer = PostSerializer(post, data=request.data)
        if serializer.is_valid():
            serializer.save()
            return Response(serializer.data)
        return Response(serializer.errors, status=status.HTTP_400_BAD_REQUEST)

    def patch(self, request, slug):
        post = self.get_object(slug)
        serializer = PostSerializer(post, data=request.data, partial=True)
        if serializer.is_valid():
            serializer.save()
            return Response(serializer.data)
        return Response(serializer.errors, status=status.HTTP_400_BAD_REQUEST)

    def delete(self, request, slug):
        post = self.get_object(slug)
        post.delete()
        return Response(status=status.HTTP_204_NO_CONTENT)
```

```python
# blog/api_urls.py
from django.urls import path
from . import api_views

app_name = 'blog_api'

urlpatterns = [
    path('posts/', api_views.PostListAPIView.as_view(), name='post-list'),
    path('posts/<slug:slug>/', api_views.PostDetailAPIView.as_view(), name='post-detail'),
]
```

### 413.3 `PUT` เทียบกับ `PATCH`: ความแตกต่างที่มือใหม่มักสับสน

| ประเด็น | `PUT` | `PATCH` |
|---|---|---|
| ความหมายตามมาตรฐาน HTTP | แทนที่ resource ทั้งตัว (full replace) | แก้ไขบางส่วน (partial update) |
| ต้องส่งฟิลด์ครบทุกตัวหรือไม่ | ควรส่งครบ (ฟิลด์ที่ขาดอาจถูกล้างค่า) | ส่งเฉพาะฟิลด์ที่ต้องการแก้ |
| พารามิเตอร์ Serializer ที่ต้องใช้ | `PostSerializer(post, data=request.data)` | `PostSerializer(post, data=request.data, partial=True)` |
| Idempotent (เรียกซ้ำผลลัพธ์เหมือนเดิม) | ✅ ใช่ | ✅ ใช่ (ในทางปฏิบัติทั่วไป) |
| ตัวอย่างการใช้งาน | แก้ไขบทความทั้งบทความในหน้าฟอร์มแก้ไข | ปุ่ม "toggle publish" ที่แก้แค่ `is_published` |

**กับดักคลาสสิก**: ถ้าลืมใส่ `partial=True` ใน `PATCH` handler แต่ client ส่งมาแค่บาง
ฟิลด์ `serializer.is_valid()` จะคืน `False` ทันทีเพราะ Serializer คิดว่าฟิลด์ที่ required
แต่ไม่ได้ส่งมาคือ error — นี่คือความแตกต่างเชิงพฤติกรรมที่สำคัญที่สุดระหว่าง `PUT`/`PATCH`
ในเชิงโค้ด ไม่ใช่แค่ชื่อ method ที่ต่างกัน

### 413.4 ทดสอบ 405 เมื่อเรียก method ที่ยังไม่ได้นิยาม

เช่นเดียวกับ Django `View` ธรรมดาที่เรียนใน Part 021 ขั้นตอนที่ 203.3 — ถ้า `APIView`
ไม่มี method ที่ตรงกับ `request.method` จะได้ 405 อัตโนมัติ (เพราะโค้ดเลือก handler ใน
`dispatch()` เหมือนกันทุกประการ ตามที่เห็นในขั้นตอนที่ 412.5):

```python
>>> from django.test import RequestFactory
>>> from blog.api_views import PostListAPIView
>>> factory = RequestFactory()
>>> view = PostListAPIView.as_view()

>>> response = view(factory.delete('/api/posts/'))
>>> response.status_code
405
>>> response.data
{'detail': ErrorDetail(string='Method "DELETE" not allowed.', code='method_not_allowed')}
```

`PostListAPIView` มีแค่ `get()` และ `post()` จึงไม่รองรับ `DELETE` — สังเกตว่า
`response.data` เป็น dict ที่มี structure ชัดเจน ต่างจาก `HttpResponseNotAllowed` ของ
Django ธรรมดาที่ไม่มี body มาตรฐาน (จะเจาะลึกกลไกนี้ในขั้นตอนที่ 416)

### 413.5 ตารางสรุป endpoint ทั้งหมดของ Part นี้

| Method | URL | Handler | ผลลัพธ์ที่คาดหวัง |
|---|---|---|---|
| `GET` | `/api/posts/` | `PostListAPIView.get()` | รายการบทความที่เผยแพร่แล้ว (200) |
| `POST` | `/api/posts/` | `PostListAPIView.post()` | สร้างบทความใหม่ (201) หรือ error (400) |
| `GET` | `/api/posts/<slug>/` | `PostDetailAPIView.get()` | รายละเอียดบทความ (200) หรือ (404) |
| `PUT` | `/api/posts/<slug>/` | `PostDetailAPIView.put()` | แก้ไขทั้งบทความ (200) หรือ error (400) |
| `PATCH` | `/api/posts/<slug>/` | `PostDetailAPIView.patch()` | แก้ไขบางส่วน (200) หรือ error (400) |
| `DELETE` | `/api/posts/<slug>/` | `PostDetailAPIView.delete()` | ลบบทความ (204, ไม่มี body) |

---

## ขั้นตอนที่ 414: `request.data` เทียบกับ `request.POST`/`request.body` ของ Django ธรรมดา

### 414.1 ทบทวนปัญหาของ `request.POST`/`request.body` แบบ Django ธรรมดา

จาก Part 007 เราเรียนรู้ว่า Django `HttpRequest` มีสองช่องทางหลักในการอ่านข้อมูลที่ client
ส่งมา:

```python
request.POST   # QueryDict — parse ได้เฉพาะ Content-Type: application/x-www-form-urlencoded
                # หรือ multipart/form-data เท่านั้น
request.body   # bytes ดิบ — ใช้ได้กับทุก Content-Type แต่ต้อง parse เอง
```

ทดลองพิสูจน์ปัญหาผ่าน `python manage.py shell` ด้วย `RequestFactory`:

```python
>>> from django.test import RequestFactory
>>> factory = RequestFactory()

>>> # ส่งข้อมูลแบบ JSON (Content-Type: application/json)
>>> request = factory.post(
...     '/api/posts/',
...     data='{"title": "สวัสดี Django"}',
...     content_type='application/json',
... )
>>> request.POST
<QueryDict: {}>
>>> # request.POST ว่างเปล่า! เพราะไม่ใช่ form-encoded

>>> request.body
b'{"title": "\\u0e2a\\u0e27\\u0e31\\u0e2a\\u0e14\\u0e35 Django"}'
>>> # ต้อง json.loads(request.body) เองถึงจะใช้ได้
```

นี่คือปัญหาที่ Part 007 ยังไม่เคยเจอ เพราะตอนนั้นเราทำงานกับฟอร์ม HTML ที่ส่งข้อมูลแบบ
`application/x-www-form-urlencoded` เสมอ แต่ API ยุคใหม่เกือบทั้งหมดสื่อสารด้วย **JSON**
เป็นหลัก ทำให้ `request.POST` ใช้งานไม่ได้เลยในสถานการณ์นี้

### 414.2 `request.data` ของ DRF: ทางออกที่รวมทุก format เข้าด้วยกัน

`request` ที่ handler ของ `APIView`/`@api_view` ได้รับ **ไม่ใช่ `HttpRequest` ของ Django
ตรง ๆ** แต่เป็น `rest_framework.request.Request` ที่ **ห่อ (wrap)** `HttpRequest` เดิมไว้
อีกชั้นหนึ่ง (จำได้จากขั้นตอนที่ 412.4 ข้อ (1): `self.initialize_request(request, ...)`)
`Request` object นี้มี attribute `.data` ที่ parse body ให้อัตโนมัติไม่ว่า Content-Type
จะเป็นอะไร:

```python
>>> from django.test import RequestFactory
>>> from blog.api_views import PostListAPIView

>>> factory = RequestFactory()
>>> view_instance = PostListAPIView()

>>> # ทดสอบผ่าน initialize_request() ตรง ๆ เพื่อดู Request object
>>> django_request = factory.post(
...     '/api/posts/',
...     data='{"title": "สวัสดี Django"}',
...     content_type='application/json',
... )
>>> drf_request = view_instance.initialize_request(django_request)
>>> drf_request.data
{'title': 'สวัสดี Django'}
>>> # request.data parse JSON ให้อัตโนมัติ เป็น dict ของ Python ตรง ๆ
```

ลองส่งข้อมูลแบบ form-encoded และ multipart เทียบกัน:

```python
>>> # แบบ form-encoded (เหมือนฟอร์ม HTML ธรรมดา)
>>> django_request = factory.post('/api/posts/', data={'title': 'จากฟอร์ม HTML'})
>>> drf_request = view_instance.initialize_request(django_request)
>>> drf_request.data
<QueryDict: {'title': ['จากฟอร์ม HTML']}>

>>> # แบบ multipart (มีไฟล์แนบด้วย)
>>> from django.core.files.uploadedfile import SimpleUploadedFile
>>> cover = SimpleUploadedFile('cover.jpg', b'fake-image-bytes', content_type='image/jpeg')
>>> django_request = factory.post(
...     '/api/posts/',
...     data={'title': 'มีรูปปก', 'cover': cover},
...     format='multipart',
... )
>>> drf_request = view_instance.initialize_request(django_request)
>>> drf_request.data
<QueryDict: {'title': ['มีรูปปก'], 'cover': [<InMemoryUploadedFile: cover.jpg (image/jpeg)>]}>
```

**`request.data` ใช้ได้เหมือนกันทุก format** — โค้ดใน view เดียวกันไม่ต้องรู้เลยว่า client
ส่งมาแบบ JSON, form-encoded หรือ multipart เพราะ DRF เลือก **Parser** ที่เหมาะสมให้
อัตโนมัติจาก header `Content-Type` ของ request

### 414.3 กลไกเบื้องหลัง: `parser_classes` และ `DEFAULT_PARSER_CLASSES`

ความสามารถนี้มาจาก `parser_classes` ที่เป็น class attribute ของ `APIView` (ทวนจาก
ขั้นตอนที่ 412.2):

```python
# settings.py — ค่า default ที่ตั้งไว้ตั้งแต่ Part 039
REST_FRAMEWORK = {
    'DEFAULT_PARSER_CLASSES': [
        'rest_framework.parsers.JSONParser',
        'rest_framework.parsers.FormParser',
        'rest_framework.parsers.MultiPartParser',
    ],
}
```

เมื่อ request เข้ามา `Request.data` (ซึ่งเป็น `@property` ภายใน) จะไล่ดู `Content-Type`
header แล้วเลือก Parser ตัวแรกใน `parser_classes` ที่ `.media_type` ตรงกับ
`Content-Type` นั้น เพื่อ parse body ให้ ถ้าไม่มี Parser ตัวไหนตรงกันเลย DRF จะ raise
`UnsupportedMediaType` (415) อัตโนมัติ — ซึ่งจะเจาะลึก exception เหล่านี้ในขั้นตอนที่ 416

จำกัด parser เฉพาะ view ได้เช่นกัน (เช่น endpoint ที่รับเฉพาะ JSON เท่านั้น ไม่รับฟอร์ม):

```python
from rest_framework.parsers import JSONParser
from rest_framework.views import APIView


class PostListAPIView(APIView):
    parser_classes = [JSONParser]   # ปฏิเสธ form-encoded/multipart โดยอัตโนมัติ
    ...
```

### 414.4 `request.query_params` เทียบกับ `request.GET`

DRF ยังเพิ่ม alias `request.query_params` ซึ่งเป็นตัวเดียวกับ `request.GET` ของ Django
เป๊ะ ๆ (ไม่มีความแตกต่างเชิงพฤติกรรม) แต่ DRF แนะนำให้ใช้ชื่อนี้แทนเพราะสื่อความหมายชัดเจน
กว่าในบริบทของ API:

```python
class PostListAPIView(APIView):
    def get(self, request):
        # ทั้งสองบรรทัดนี้ทำงานเหมือนกันทุกประการ
        search = request.query_params.get('search', '')
        search = request.GET.get('search', '')   # ใช้ได้เหมือนกัน แต่ DRF ไม่แนะนำชื่อนี้
        ...
```

เหตุผลที่ DRF ไม่ชอบชื่อ `request.GET`: **query string มีอยู่ได้ในทุก HTTP method**
(เช่น `POST /api/posts/?draft=true`) การเรียกมันว่า `GET` จึงทำให้เข้าใจผิดว่าเกี่ยวข้อง
กับ HTTP GET method เท่านั้น ทั้งที่จริงแล้วเป็นคนละเรื่องกัน `request.query_params` จึงเป็น
ชื่อที่ตรงความหมายกว่าและเป็น convention มาตรฐานของโค้ด DRF ทั่วโลก

### 414.5 ตารางสรุปเปรียบเทียบทั้งหมด

| Attribute | มาจาก | รองรับ Content-Type | ผลลัพธ์ที่ได้ | ใช้เมื่อไหร่ |
|---|---|---|---|---|
| `request.POST` | Django `HttpRequest` | form-urlencoded, multipart เท่านั้น | `QueryDict` | ฟอร์ม HTML ธรรมดา (Part 007-030) |
| `request.body` | Django `HttpRequest` | ทุกชนิด (raw bytes) | `bytes` ดิบ | ต้อง parse เอง กรณีพิเศษ (เช่น webhook signature) |
| `request.data` | DRF `Request` | JSON, form-urlencoded, multipart (ตาม `parser_classes`) | `dict`/`QueryDict` ที่ parse แล้ว | API views ทุกชนิด (แนะนำเสมอ) |
| `request.GET` | Django `HttpRequest` | Query string (`?key=value`) | `QueryDict` | ใช้ได้ แต่ DRF แนะนำ `query_params` แทน |
| `request.query_params` | DRF `Request` (alias ของ `request.GET`) | Query string (`?key=value`) | `QueryDict` | API views ทุกชนิด (แนะนำเสมอ) |
| `request.FILES` | Django `HttpRequest` (DRF สืบทอดมาใช้ต่อ) | multipart เท่านั้น | `MultiValueDict` ของไฟล์ | Upload ไฟล์ (จะเจาะลึกใน Part 058) |

### 414.6 กับดักคลาสสิก: เข้าใจผิดว่า `request.data` แทนที่ `request.POST` ได้ทุกที่

`request.data` ทำงานเฉพาะใน view ที่สืบทอดจาก `APIView` (หรือฟังก์ชันที่ครอบด้วย
`@api_view`) เท่านั้น เพราะมันคือ attribute ของ DRF `Request`, **ไม่ใช่** ของ Django
`HttpRequest` — ถ้านำ `request.data` ไปใช้ใน Template-based View ธรรมดาแบบ Part 007/021
จะได้ `AttributeError` ทันที เพราะ `HttpRequest` ของ Django ไม่มี attribute นี้อยู่เลย

---

## ขั้นตอนที่ 415: `Response` object และ Content Negotiation เจาะลึกกว่า Part 039

### 415.1 ทบทวนสิ่งที่ Part 039 เคยใช้แบบเบื้องต้น

ใน Part 039 เราใช้ `Response(serializer.data)` เพื่อคืนค่า JSON แบบง่าย ๆ ไปแล้ว Part นี้
จะพาไปดูว่าเบื้องหลังมันทำงานอย่างไรกันแน่ และทำไมมันไม่ใช่แค่ `JsonResponse` ที่เปลี่ยน
ชื่อ

### 415.2 `Response` ต่างจาก `JsonResponse` ตรงไหน

```python
# JsonResponse ของ Django (Part 007) — "ตัดสินใจ" ทันทีว่าเป็น JSON ตั้งแต่สร้าง object
from django.http import JsonResponse
return JsonResponse({'title': 'สวัสดี'})   # แปลงเป็น JSON string ทันทีตอนสร้าง object

# Response ของ DRF — "ยังไม่ตัดสินใจ" format จนกว่าจะถึงตอน finalize_response()
from rest_framework.response import Response
return Response({'title': 'สวัสดี'})   # เก็บ Python dict ดิบไว้ก่อน ยังไม่ render
```

`Response` ทำงานคล้าย Django `TemplateResponse` มากกว่า `JsonResponse` — คือเก็บ
**ข้อมูลดิบ** (`self.data`) ไว้ก่อน แล้วค่อย **render** เป็น byte string จริง ๆ ในภายหลัง
ตอน middleware เรียก `response.render()` (Django เรียก render ของ
`TemplateResponse`-like object โดยอัตโนมัติผ่าน middleware `SimpleTemplateResponse`
mechanism) จุดที่ตัดสินใจว่าจะ render เป็น format ไหนคือ `finalize_response()` ที่เจาะลึก
ไปแล้วในขั้นตอนที่ 412.7

### 415.3 Content Negotiation คืออะไร

**Content Negotiation** คือกระบวนการที่ server ตัดสินใจว่าจะตอบกลับด้วย format ไหน
โดยพิจารณาจาก:

1. HTTP header `Accept` ที่ client ส่งมา (เช่น `Accept: application/json`)
2. `renderer_classes` ที่ view นั้นประกาศไว้ (หรือ default จาก settings)
3. Query parameter พิเศษ `?format=json` (ถ้าเปิดใช้งาน `URLPathVersioning`/format suffix)

```python
# settings.py — ค่า default ที่ตั้งไว้ตั้งแต่ Part 039
REST_FRAMEWORK = {
    'DEFAULT_RENDERER_CLASSES': [
        'rest_framework.renderers.JSONRenderer',
        'rest_framework.renderers.BrowsableAPIRenderer',
    ],
}
```

ค่า default มี renderer สองตัว: `JSONRenderer` (คืน JSON จริง) และ `BrowsableAPIRenderer`
(คืนหน้า HTML สวย ๆ ที่ห่อ JSON ไว้ให้ดูง่ายในเบราว์เซอร์ — นี่คือหน้าที่เราเห็นตอนเปิด
`/api/posts/` ด้วยเบราว์เซอร์ในขั้นตอนที่ 411.4)

### 415.4 ทดสอบ Content Negotiation ด้วย header `Accept` ต่างกัน

```bash
# ขอ JSON ตรง ๆ
curl -H "Accept: application/json" http://127.0.0.1:8000/api/posts/
# {"count": ..., "results": [...]}   ← ได้ JSON ดิบ

# ขอ HTML (เบราว์เซอร์ส่ง header นี้เป็นค่าเริ่มต้น)
curl -H "Accept: text/html" http://127.0.0.1:8000/api/posts/
# <!DOCTYPE html>...   ← ได้หน้า Browsable API แบบเต็ม
```

### 415.5 อัลกอริทึมการเลือก Renderer (`DefaultContentNegotiation`)

```python
# แนวคิดจากซอร์สโค้ดจริงของ rest_framework.negotiation.DefaultContentNegotiation
class DefaultContentNegotiation:
    def select_renderer(self, request, renderers, format_suffix=None):
        accepts = self.get_accept_list(request)
        # ไล่ทีละ media type ใน header Accept (เรียงตาม priority/q-value)
        for media_type_set in accepts:
            for renderer in renderers:
                if media_type_matches(renderer.media_type, media_type_set):
                    return renderer, renderer.media_type

        # ไม่มี renderer ตัวไหนตรงกับที่ client ขอเลย
        raise NotAcceptable(available_renderers=renderers)
```

พูดง่าย ๆ: DRF ไล่เทียบ `Accept` header ของ client กับ `media_type` ของแต่ละ renderer
ใน `renderer_classes` ตามลำดับที่ประกาศไว้ ตัวไหนตรงกันก่อนก็ใช้ตัวนั้น ถ้าไม่มีตัวไหนตรง
เลยจะ raise `NotAcceptable` (HTTP 406 — เจาะลึกใน ขั้นตอนที่ 416)

### 415.6 จำกัด renderer เฉพาะ view (ปิด Browsable API)

ในโปรเจกต์ production บาง endpoint ต้องการคืน **JSON เท่านั้น** ไม่ต้องการหน้า Browsable
API (เช่น endpoint ที่มี response ขนาดใหญ่มาก หรือ endpoint ที่ใช้เฉพาะ mobile app):

```python
from rest_framework.renderers import JSONRenderer
from rest_framework.views import APIView


class PostListAPIView(APIView):
    renderer_classes = [JSONRenderer]   # ปิด BrowsableAPIRenderer สำหรับ view นี้เท่านั้น
    ...
```

### 415.7 พารามิเตอร์ทั้งหมดของ `Response()`

```python
Response(
    data,                      # ข้อมูลดิบ (dict, list, OrderedDict) — DRF จะ serialize ให้ตอน render
    status=None,               # HTTP status code (ค่า default คือ 200) — ขั้นตอนที่ 417
    template_name=None,        # ใช้เฉพาะกับ HTMLRenderer เท่านั้น (เจอน้อยมาก)
    headers=None,              # dict ของ extra headers เช่น {'X-Custom-Header': 'value'}
    exception=False,           # True เมื่อ Response นี้ถูกสร้างจาก exception handler
    content_type=None,         # override content type เอง (ปกติไม่ต้องระบุ ให้ renderer จัดการ)
)
```

ตัวอย่างการใช้ `headers`:

```python
return Response(
    serializer.data,
    status=status.HTTP_201_CREATED,
    headers={'Location': f'/api/posts/{post.slug}/'},   # บอก client ว่า resource ใหม่อยู่ที่ไหน
)
```

การใส่ header `Location` ตอนสร้าง resource ใหม่ (`201 Created`) เป็นแนวปฏิบัติที่ดีตาม
มาตรฐาน REST — บอก client ว่าจะเข้าถึง resource ที่เพิ่งสร้างได้จาก URL ไหนโดยไม่ต้อง
เดาเอง

### 415.8 ตารางสรุป Content Negotiation

| องค์ประกอบ | หน้าที่ | ตั้งค่าได้ที่ไหน |
|---|---|---|
| `Accept` header | client บอกว่าต้องการ format ไหน | ฝั่ง client เป็นผู้กำหนด |
| `renderer_classes` | รายชื่อ format ที่ view นี้ "เสนอให้เลือกได้" | class attribute ของ `APIView` หรือ `DEFAULT_RENDERER_CLASSES` ใน settings |
| `content_negotiation_class` | อัลกอริทึมที่ใช้จับคู่ `Accept` กับ `renderer_classes` | ปกติใช้ `DefaultContentNegotiation` ค่า default |
| `JSONRenderer` | render `response.data` เป็น JSON string | เปิดใช้เสมอในโปรเจกต์ API จริง |
| `BrowsableAPIRenderer` | render เป็นหน้า HTML สำหรับทดสอบผ่านเบราว์เซอร์ | เปิดตอน dev, พิจารณาปิดใน production ที่ endpoint สำคัญ |

---

## ขั้นตอนที่ 416: Exception Handling ใน DRF — `APIException`, custom exception handler

### 416.1 ปัญหาของ unhandled exception ใน Django ธรรมดา

ถ้า view ธรรมดาแบบ Part 007 เกิด exception ที่ไม่ได้ดักไว้ (เช่น `KeyError`,
`AttributeError`) Django จะคืน **หน้า HTML error 500** (หรือ debug page ถ้า
`DEBUG=True`) กลับไปให้ client — สำหรับเว็บทั่วไปนี่พอรับได้ แต่สำหรับ API ที่ client
คาดหวัง **JSON เสมอ** การได้ HTML กลับมาจะทำให้ mobile app หรือ frontend JavaScript
พังทันทีตอนพยายาม `response.json()`

### 416.2 `handle_exception()`: เกราะป้องกันของ `APIView`

ทวนจากขั้นตอนที่ 412.4: `dispatch()` ของ `APIView` ครอบ handler ทั้งหมดด้วย
`try/except Exception as exc: response = self.handle_exception(exc)` มาดูว่า
`handle_exception()` ทำอะไรบ้าง:

```python
# แนวคิดจากซอร์สโค้ดจริงของ rest_framework.views.APIView
def handle_exception(self, exc):
    if isinstance(exc, (exceptions.NotAuthenticated, exceptions.AuthenticationFailed)):
        auth_header = self.get_authenticate_header(self.request)
        if auth_header:
            exc.auth_header = auth_header
        else:
            exc.status_code = status.HTTP_403_FORBIDDEN

    exception_handler = self.get_exception_handler()

    context = self.get_exception_handler_context()
    response = exception_handler(exc, context)

    if response is None:
        self.raise_uncaught_exception(exc)   # re-raise ให้ Django จัดการต่อ (500)

    response.exception = True
    return response
```

`self.get_exception_handler()` อ่านค่าจาก setting `EXCEPTION_HANDLER` (ค่า default คือ
`rest_framework.views.exception_handler`) — ฟังก์ชันนี้คือจุดที่แปลง exception ให้เป็น
`Response` มาตรฐาน มาดูโค้ดจริงกัน:

```python
# แนวคิดจากซอร์สโค้ดจริงของ rest_framework.views.exception_handler
def exception_handler(exc, context):
    if isinstance(exc, Http404):
        exc = exceptions.NotFound(*(exc.args))
    elif isinstance(exc, PermissionDenied):
        exc = exceptions.PermissionDenied(*(exc.args))

    if isinstance(exc, exceptions.APIException):
        headers = {}
        if getattr(exc, 'auth_header', None):
            headers['WWW-Authenticate'] = exc.auth_header
        if getattr(exc, 'wait', None):
            headers['Retry-After'] = '%d' % exc.wait

        if isinstance(exc.detail, (list, dict)):
            data = exc.detail
        else:
            data = {'detail': exc.detail}

        set_rollback()
        return Response(data, status=exc.status_code, headers=headers)

    return None
```

ข้อสังเกตสำคัญสองข้อ:

1. **Django's `Http404` และ `PermissionDenied` ถูกแปลงเป็น DRF exception ก่อน** — นี่คือ
   เหตุผลที่ `get_object_or_404()` ของ Django ธรรมดา (ที่เราใช้ใน `PostDetailAPIView`
   ขั้นตอนที่ 413.2) ทำงานได้ถูกต้องใน `APIView` โดยไม่ต้อง import อะไรเพิ่มจาก DRF เลย
2. **ถ้า exception ไม่ใช่ `APIException` (หลังแปลงแล้ว) ฟังก์ชันคืน `None`** ทำให้
   `handle_exception()` re-raise exception นั้นต่อไปให้ Django จัดการแบบเดิม (คือหน้า 500
   ตามปกติ — สำคัญมากตอน debug เพราะถ้า DRF "กลืน" ทุก exception ไปหมด นักพัฒนาจะไม่เห็น
   traceback เต็ม ๆ ตอน `DEBUG=True`)

### 416.3 raise `APIException` เองใน view

```python
# blog/api_views.py
from rest_framework.exceptions import APIException


class PostLimitExceeded(APIException):
    status_code = 429
    default_detail = 'คุณสร้างบทความเกินจำนวนที่กำหนดในหนึ่งวัน (สูงสุด 5 บทความ/วัน)'
    default_code = 'post_limit_exceeded'


class PostListAPIView(APIView):
    def post(self, request):
        today_count = Post.objects.filter(
            author=request.user,
            created_at__date=timezone.now().date(),
        ).count()
        if today_count >= 5:
            raise PostLimitExceeded()   # ไม่ต้องเขียน try/except เอง — dispatch() ดักให้

        serializer = PostSerializer(data=request.data)
        serializer.is_valid(raise_exception=True)   # raise ValidationError อัตโนมัติถ้าไม่ผ่าน
        serializer.save(author=request.user)
        return Response(serializer.data, status=status.HTTP_201_CREATED)
```

สังเกตสองจุดสำคัญ:

- `raise PostLimitExceeded()` โยน exception ตรง ๆ โดยไม่ต้องเขียน `try/except` ล้อมรอบ
  เพราะ `dispatch()` ของ `APIView` ดักไว้ให้แล้วในขั้นตอนที่ 412.4
- `serializer.is_valid(raise_exception=True)` เป็นทางเลือกแทนการเช็ค
  `if serializer.is_valid():` แบบ if/else ที่เห็นในขั้นตอนก่อนหน้า — เมื่อ validation
  ไม่ผ่าน มันจะ raise `rest_framework.exceptions.ValidationError` ให้อัตโนมัติ ซึ่งถูก
  ดักและแปลงเป็น `Response(..., status=400)` โดย `exception_handler` เช่นกัน ทำให้โค้ด
  สั้นลงและไม่ต้องเขียน `return Response(serializer.errors, status=400)` ซ้ำทุกที่

### 416.4 ตาราง Exception class มาตรฐานของ DRF

| Exception class | `status_code` | ใช้เมื่อไหร่ |
|---|---|---|
| `ValidationError` | 400 | ข้อมูลที่ client ส่งมาไม่ผ่าน validation (raise อัตโนมัติจาก Serializer) |
| `ParseError` | 400 | Body ที่ parse ไม่ได้ (เช่น JSON ผิด syntax) |
| `AuthenticationFailed` | 401 | credential ที่ส่งมาไม่ถูกต้อง (เช่น token ผิด) |
| `NotAuthenticated` | 401 | ไม่ได้แนบ credential มาเลย และ endpoint ต้องการ authentication |
| `PermissionDenied` | 403 | Authenticated แล้วแต่ไม่มีสิทธิ์ทำ action นี้ |
| `NotFound` | 404 | ไม่พบ resource (แปลงมาจาก Django `Http404` อัตโนมัติ) |
| `MethodNotAllowed` | 405 | HTTP method ที่ view ไม่รองรับ (raise อัตโนมัติจาก `dispatch()`) |
| `NotAcceptable` | 406 | ไม่มี renderer ตัวไหนตรงกับ `Accept` header ของ client (ขั้นตอนที่ 415.5) |
| `UnsupportedMediaType` | 415 | ไม่มี parser ตัวไหนตรงกับ `Content-Type` ของ request (ขั้นตอนที่ 414.3) |
| `Throttled` | 429 | ส่ง request ถี่เกินขีดจำกัด (จะเรียนเต็มรูปแบบใน Part 048) |
| `APIException` | 500 (default) | Base class — สืบทอดเพื่อสร้าง custom exception ของตัวเอง (ขั้นตอนที่ 416.3) |

### 416.5 Custom Exception Handler: ปรับ format error ให้เป็นมาตรฐานเดียวทั้งโปรเจกต์

ในโปรเจกต์ระดับทีม มักต้องการ error response ที่มี **shape เดียวกันทุก endpoint** เช่น
เพิ่ม field `success: false` และ `status_code` เข้าไปเสมอ ทำได้โดยเขียน exception
handler ของตัวเองที่ **ห่อ** (wrap) ผลลัพธ์จาก `exception_handler` ค่า default ของ DRF
อีกชั้น:

```python
# config/exception_handlers.py
from rest_framework.views import exception_handler as drf_exception_handler


def custom_exception_handler(exc, context):
    # เรียก default handler ของ DRF ก่อนเสมอ ให้มันแปลง exception เป็น Response ตามปกติ
    response = drf_exception_handler(exc, context)

    if response is not None:
        # response is None แปลว่า exception ไม่ใช่ APIException (ปล่อยให้ Django จัดการ 500)
        response.data = {
            'success': False,
            'status_code': response.status_code,
            'errors': response.data,
        }

    return response
```

```python
# settings.py
REST_FRAMEWORK = {
    'DEFAULT_PARSER_CLASSES': [
        'rest_framework.parsers.JSONParser',
        'rest_framework.parsers.FormParser',
        'rest_framework.parsers.MultiPartParser',
    ],
    'DEFAULT_RENDERER_CLASSES': [
        'rest_framework.renderers.JSONRenderer',
        'rest_framework.renderers.BrowsableAPIRenderer',
    ],
    'EXCEPTION_HANDLER': 'config.exception_handlers.custom_exception_handler',
}
```

ผลลัพธ์ที่ client จะได้รับเปลี่ยนจาก:

```json
{"detail": "Not found."}
```

เป็น:

```json
{
    "success": false,
    "status_code": 404,
    "errors": {"detail": "Not found."}
}
```

**กฎเหล็กของหลักสูตรนี้**: ทุกโปรเจกต์ API ระดับ production ควรมี custom exception
handler แบบนี้อย่างน้อยหนึ่งชั้น เพื่อให้ frontend/mobile app เขียนโค้ดจัดการ error ได้
ด้วย logic เดียว ไม่ต้องเช็ค shape ของ response ต่างกันไปในแต่ละ endpoint

### 416.6 ตารางสรุปกลไก Exception Handling ทั้งหมด

| ขั้นตอน | เกิดที่ไหน | หน้าที่ |
|---|---|---|
| Handler (`get()`/`post()`) raise exception | โค้ดของนักพัฒนา | ส่งสัญญาณว่ามีข้อผิดพลาดเกิดขึ้น |
| `dispatch()` ดัก exception | `try/except Exception` ใน `APIView.dispatch()` | ป้องกันไม่ให้ exception หลุดออกไปแบบดิบ ๆ |
| `handle_exception(exc)` | `APIView` method | แปลง `Http404`/`PermissionDenied` เป็น DRF exception ก่อน |
| `EXCEPTION_HANDLER` (`exception_handler`) | ฟังก์ชันที่ตั้งค่าไว้ใน settings | แปลง `APIException` เป็น `Response` ที่มี status code ถูกต้อง |
| Custom exception handler (ถ้ามี) | ฟังก์ชันของนักพัฒนาเอง | ปรับ shape ของ error response ให้สม่ำเสมอทั้งโปรเจกต์ |
| `response is None` | เมื่อ exception ไม่ใช่ `APIException` | Re-raise ให้ Django คืนหน้า 500 ตามปกติ (สำคัญตอน debug) |

---

## ขั้นตอนที่ 417: การเลือกใช้ HTTP Status Code ที่ถูกต้องตามหลัก REST

### 417.1 ทำไม Status Code ถึงสำคัญมากใน API design

Status code คือ "ภาษากลาง" ที่ client ทุกภาษา (JavaScript, Swift, Kotlin, Python) เข้าใจ
ตรงกันโดยไม่ต้อง parse body เลย การเลือก status code ผิด (เช่น คืน `200 OK` ทั้งที่
validation ล้มเหลว) จะทำให้ client เขียน logic ตรวจสอบ error ผิดพลาดได้ง่าย

### 417.2 ใช้ `rest_framework.status` แทนการเขียนตัวเลขตรง ๆ

```python
from rest_framework import status

# ❌ ไม่แนะนำ: ตัวเลขดิบอ่านยาก ต้องจำเองว่า 201 คืออะไร
return Response(data, status=201)

# ✅ แนะนำ: ชื่อ constant สื่อความหมายชัดเจน อ่านโค้ดแล้วเข้าใจทันที
return Response(data, status=status.HTTP_201_CREATED)
```

`rest_framework.status` มี constant ให้ครบทุก status code มาตรฐาน แบ่งกลุ่มตามหลักร้อย:

```python
status.HTTP_200_OK
status.HTTP_201_CREATED
status.HTTP_204_NO_CONTENT
status.HTTP_400_BAD_REQUEST
status.HTTP_401_UNAUTHORIZED
status.HTTP_403_FORBIDDEN
status.HTTP_404_NOT_FOUND
status.HTTP_405_METHOD_NOT_ALLOWED
status.HTTP_409_CONFLICT
status.HTTP_429_TOO_MANY_REQUESTS
status.HTTP_500_INTERNAL_SERVER_ERROR

# ยังมี helper function เช็คกลุ่ม status code ด้วย
status.is_success(200)       # True (2xx)
status.is_client_error(404)  # True (4xx)
status.is_server_error(500)  # True (5xx)
```

### 417.3 ตารางเลือก Status Code ตาม action ของ CRUD

| Action | Method | Status Code สำเร็จ | Status Code ล้มเหลว |
|---|---|---|---|
| ดูรายการทั้งหมด | `GET /posts/` | `200 OK` | - |
| ดูรายละเอียด 1 รายการ | `GET /posts/<slug>/` | `200 OK` | `404 Not Found` (ไม่พบ) |
| สร้างใหม่ | `POST /posts/` | `201 Created` | `400 Bad Request` (validation ล้มเหลว) |
| แก้ไขทั้งหมด | `PUT /posts/<slug>/` | `200 OK` | `400`/`404` |
| แก้ไขบางส่วน | `PATCH /posts/<slug>/` | `200 OK` | `400`/`404` |
| ลบ | `DELETE /posts/<slug>/` | `204 No Content` | `404 Not Found` |
| ไม่ได้ authenticate | ทุก method ที่ต้อง login | - | `401 Unauthorized` |
| Authenticate แล้วแต่ไม่มีสิทธิ์ | ทุก method ที่มี permission check | - | `403 Forbidden` |
| Method ที่ view ไม่รองรับ | เช่น `DELETE` ที่ไม่มี handler | - | `405 Method Not Allowed` |
| ข้อมูลซ้ำ/ขัดแย้งกัน | เช่น สร้าง slug ที่มีอยู่แล้ว | - | `409 Conflict` |

### 417.4 `204 No Content`: ต้องไม่มี body

```python
# ✅ ถูกต้อง: 204 ไม่มี body เลย
def delete(self, request, slug):
    post = self.get_object(slug)
    post.delete()
    return Response(status=status.HTTP_204_NO_CONTENT)

# ❌ ผิดหลักการ: 204 ไม่ควรมี data ติดมาด้วย (บาง client ถึงกับ error ถ้าเจอ body ใน 204)
def delete(self, request, slug):
    post = self.get_object(slug)
    post.delete()
    return Response({'message': 'ลบสำเร็จ'}, status=status.HTTP_204_NO_CONTENT)
```

ตามมาตรฐาน HTTP/1.1 (RFC 9110) **response ที่มี status `204 No Content` ต้องไม่มี
message body** เพราะความหมายของมันคือ "สำเร็จ แต่ไม่มีอะไรจะส่งกลับ" ถ้าต้องการส่งข้อมูล
กลับไปด้วย (เช่นข้อความยืนยัน) ให้ใช้ `200 OK` แทน

### 417.5 Idempotency: คุณสมบัติที่ต้องคำนึงถึงตอนเลือก method

| Method | Idempotent? | ความหมาย |
|---|---|---|
| `GET` | ✅ ใช่ | เรียกกี่ครั้งก็ได้ผลลัพธ์เดิม ไม่เปลี่ยนแปลงข้อมูล |
| `PUT` | ✅ ใช่ | เรียกซ้ำด้วยข้อมูลเดิม ผลลัพธ์สุดท้ายเหมือนกันทุกครั้ง |
| `DELETE` | ✅ ใช่ | ลบซ้ำ resource ที่ถูกลบไปแล้วควรได้ 404 ไม่ใช่ error แปลก ๆ |
| `PATCH` | ⚠️ ควรเป็น | ในทางปฏิบัติทั่วไปถือว่า idempotent แต่ไม่บังคับตาม spec เป๊ะ ๆ |
| `POST` | ❌ ไม่ใช่ | เรียกซ้ำมักสร้าง resource ใหม่ซ้ำอีกรายการ (เช่นสร้างบทความซ้ำ) |

เข้าใจ idempotency ช่วยตัดสินใจได้ว่าควรใช้ method ไหนตอนออกแบบ endpoint ใหม่ ๆ เช่น
endpoint "toggle publish" ควรใช้ `PATCH` (แก้ค่าเฉพาะฟิลด์ `is_published`) ไม่ใช่ `POST`
เพราะเป็นการแก้ไขข้อมูลเดิม ไม่ใช่การสร้างสิ่งใหม่

---

## ขั้นตอนที่ 418: สร้าง CRUD เต็มรูปแบบด้วย `APIView` ธรรมดาสำหรับ `Post`

### 418.1 ภาพรวมของ endpoint ทั้งหมดที่จะสร้าง

Part นี้ประกอบทุกความรู้จากขั้นตอนที่ 411-417 เข้าด้วยกันเป็น CRUD API ที่สมบูรณ์สำหรับ
`Post` **โดยเจตนาใช้ `APIView` ธรรมดา ยังไม่ใช้ Generic Views/Mixins** (ซึ่งจะเรียนเต็ม
รูปแบบใน Part 043 ถัดไป) เพื่อให้เห็นภาพชัดว่าโค้ดที่ Generic Views จะย่อให้สั้นลงนั้น
"หน้าตาแบบเต็ม" เป็นอย่างไร (แนวคิดเดียวกับที่ Part 021 ทำ CBV แบบ manual ก่อนเรียน
`ListView`/`DetailView` ใน Part 022)

### 418.2 `blog/serializers.py` (recap เต็มจาก Part 041)

```python
# blog/serializers.py
from rest_framework import serializers
from .models import Post


class PostSerializer(serializers.ModelSerializer):
    author = serializers.ReadOnlyField(source='author.username')

    class Meta:
        model = Post
        fields = [
            'id', 'author', 'title', 'slug',
            'content', 'is_published', 'created_at', 'updated_at',
        ]
        read_only_fields = ['id', 'slug', 'created_at', 'updated_at']
```

### 418.3 `blog/api_views.py` (เวอร์ชันเต็มสมบูรณ์)

```python
# blog/api_views.py
from django.shortcuts import get_object_or_404
from rest_framework import permissions, status
from rest_framework.response import Response
from rest_framework.views import APIView

from .models import Post
from .serializers import PostSerializer


class PostListAPIView(APIView):
    """
    GET  /api/posts/  -> รายการบทความที่เผยแพร่แล้วทั้งหมด
    POST /api/posts/  -> สร้างบทความใหม่ (ต้อง login)
    """
    permission_classes = [permissions.IsAuthenticatedOrReadOnly]

    def get(self, request):
        posts = Post.objects.filter(is_published=True)
        serializer = PostSerializer(posts, many=True)
        return Response(serializer.data, status=status.HTTP_200_OK)

    def post(self, request):
        serializer = PostSerializer(data=request.data)
        serializer.is_valid(raise_exception=True)
        post = serializer.save(author=request.user)
        return Response(
            PostSerializer(post).data,
            status=status.HTTP_201_CREATED,
            headers={'Location': f'/api/posts/{post.slug}/'},
        )


class PostDetailAPIView(APIView):
    """
    GET    /api/posts/<slug>/  -> รายละเอียดบทความ
    PUT    /api/posts/<slug>/  -> แก้ไขทั้งหมด (ต้องเป็นเจ้าของ)
    PATCH  /api/posts/<slug>/  -> แก้ไขบางส่วน (ต้องเป็นเจ้าของ)
    DELETE /api/posts/<slug>/  -> ลบ (ต้องเป็นเจ้าของ)
    """
    permission_classes = [permissions.IsAuthenticatedOrReadOnly]

    def get_object(self, slug):
        post = get_object_or_404(Post, slug=slug)
        self.check_object_permissions(self.request, post)
        return post

    def check_object_permissions(self, request, post):
        """
        ตรวจสอบว่าเฉพาะเจ้าของบทความเท่านั้นที่แก้ไข/ลบได้ (การตรวจสอบสิทธิ์แบบ
        object-level เต็มรูปแบบด้วย permission class custom จะเรียนใน Part 045 —
        ตอนนี้ตรวจแบบง่าย ๆ ตรงนี้ก่อนเพื่อให้ endpoint ใช้งานได้จริงและปลอดภัย)
        """
        if request.method not in permissions.SAFE_METHODS:
            if post.author != request.user:
                from rest_framework.exceptions import PermissionDenied
                raise PermissionDenied('คุณไม่มีสิทธิ์แก้ไขบทความของผู้อื่น')

    def get(self, request, slug):
        post = get_object_or_404(Post, slug=slug)
        serializer = PostSerializer(post)
        return Response(serializer.data, status=status.HTTP_200_OK)

    def put(self, request, slug):
        post = self.get_object(slug)
        serializer = PostSerializer(post, data=request.data)
        serializer.is_valid(raise_exception=True)
        serializer.save()
        return Response(serializer.data, status=status.HTTP_200_OK)

    def patch(self, request, slug):
        post = self.get_object(slug)
        serializer = PostSerializer(post, data=request.data, partial=True)
        serializer.is_valid(raise_exception=True)
        serializer.save()
        return Response(serializer.data, status=status.HTTP_200_OK)

    def delete(self, request, slug):
        post = self.get_object(slug)
        post.delete()
        return Response(status=status.HTTP_204_NO_CONTENT)
```

**หมายเหตุสำคัญ**: `permissions.IsAuthenticatedOrReadOnly` และการเช็คสิทธิ์แบบละเอียด
(object-level permissions) จะเรียนเต็มรูปแบบใน **Part 045: DRF Permissions และ
Authentication** โค้ดข้างต้นใช้แบบง่ายที่สุดเท่าที่จำเป็นเพื่อให้ endpoint ปลอดภัยพอ
สำหรับทดสอบจริงใน Part นี้

### 418.4 `blog/api_urls.py` (เวอร์ชันเต็ม)

```python
# blog/api_urls.py
from django.urls import path
from . import api_views

app_name = 'blog_api'

urlpatterns = [
    path('posts/', api_views.PostListAPIView.as_view(), name='post-list'),
    path('posts/<slug:slug>/', api_views.PostDetailAPIView.as_view(), name='post-detail'),
]
```

```python
# config/urls.py
from django.contrib import admin
from django.urls import path, include

urlpatterns = [
    path('admin/', admin.site.urls),
    path('api/', include('blog.api_urls')),
]
```

### 418.5 ตั้งค่า `settings.py` ให้ครบสำหรับ Part นี้

```python
# config/settings.py
INSTALLED_APPS = [
    # ...
    'rest_framework',
    'blog',
]

REST_FRAMEWORK = {
    'DEFAULT_PARSER_CLASSES': [
        'rest_framework.parsers.JSONParser',
        'rest_framework.parsers.FormParser',
        'rest_framework.parsers.MultiPartParser',
    ],
    'DEFAULT_RENDERER_CLASSES': [
        'rest_framework.renderers.JSONRenderer',
        'rest_framework.renderers.BrowsableAPIRenderer',
    ],
    'DEFAULT_AUTHENTICATION_CLASSES': [
        'rest_framework.authentication.SessionAuthentication',
    ],
    'EXCEPTION_HANDLER': 'config.exception_handlers.custom_exception_handler',
}
```

**หมายเหตุ**: `SessionAuthentication` ใช้ session cookie ของ Django ที่มีอยู่แล้วจาก
Part 031 เป็นวิธี authenticate ชั่วคราวสำหรับ Part นี้ — Part 046 จะแนะนำ Token/JWT
authentication ที่เหมาะกับ API จริงมากกว่า (ไม่ต้องพึ่ง cookie/session)

### 418.6 ทดสอบด้วย `python manage.py shell` ก่อนทดสอบผ่าน HTTP จริง

```python
>>> from django.test import RequestFactory
>>> from django.contrib.auth import get_user_model
>>> from blog.api_views import PostListAPIView, PostDetailAPIView

>>> User = get_user_model()
>>> user = User.objects.get(username='demo')
>>> factory = RequestFactory()

>>> # สร้างบทความใหม่
>>> request = factory.post(
...     '/api/posts/',
...     data='{"title": "ทดสอบ APIView", "content": "เนื้อหาทดสอบ"}',
...     content_type='application/json',
... )
>>> request.user = user
>>> response = PostListAPIView.as_view()(request)
>>> response.status_code
201
>>> response.data['slug']
'ทดสอบ-apiview'
```

---

## ขั้นตอนที่ 419: ทดสอบ API ด้วย `curl`/`httpie` จาก command line

### 419.1 ทำไมต้องทดสอบผ่าน command line

Browsable API (ขั้นตอนที่ 415) สะดวกสำหรับ debug ด้วยตา แต่ **ไม่เหมาะกับการทดสอบ
อัตโนมัติหรือทดสอบ header/status code แบบละเอียด** เครื่องมือ command line อย่าง `curl`
(มีติดเครื่องเกือบทุกระบบปฏิบัติการ) และ `httpie` (syntax อ่านง่ายกว่า ติดตั้งแยก)
คือเครื่องมือมาตรฐานที่นักพัฒนา backend ใช้ทดสอบ API ก่อนเชื่อม frontend จริง

### 419.2 ติดตั้ง `httpie` (ถ้ายังไม่มี)

```bash
pip install httpie
# หรือ
brew install httpie      # macOS
sudo apt install httpie  # Ubuntu/Debian
```

### 419.3 ทดสอบ `GET /api/posts/`

```bash
# curl
curl -i http://127.0.0.1:8000/api/posts/

# httpie (syntax สั้นกว่า และ pretty-print JSON ให้อัตโนมัติ)
http GET 127.0.0.1:8000/api/posts/
```

ผลลัพธ์ที่คาดหวัง (`-i` ของ curl แสดง header ด้วย):

```
HTTP/1.1 200 OK
Content-Type: application/json

[
    {
        "id": 1,
        "author": "demo",
        "title": "ทดสอบ APIView",
        "slug": "ทดสอบ-apiview",
        "content": "เนื้อหาทดสอบ",
        "is_published": true,
        "created_at": "2026-09-26T10:00:00Z",
        "updated_at": "2026-09-26T10:00:00Z"
    }
]
```

### 419.4 ทดสอบ `POST /api/posts/` พร้อม authentication

```bash
# curl: ต้องส่ง session cookie ที่ได้จาก login มาด้วย (-b) และปิด CSRF check ด้วยการใช้
# Basic/Token auth แทนในโปรเจกต์จริง (Part 046) — ในที่นี้สมมติทดสอบด้วย Token ที่จะเรียน
# วิธีสร้างจริงใน Part 046 ก่อน ใช้ SessionAuthentication ผ่าน login form ธรรมดาไปพลาง ๆ
curl -i -X POST http://127.0.0.1:8000/api/posts/ \
  -H "Content-Type: application/json" \
  -b cookies.txt \
  -d '{"title": "บทความจาก curl", "content": "เขียนจาก command line"}'

# httpie: syntax สั้นกว่ามาก ระบุ field=value ตรง ๆ ไม่ต้องเขียน JSON เอง
http POST 127.0.0.1:8000/api/posts/ \
  title="บทความจาก httpie" \
  content="เขียนจาก command line" \
  --session=demo
```

ผลลัพธ์ที่คาดหวัง:

```
HTTP/1.1 201 Created
Content-Type: application/json
Location: /api/posts/บทความจาก-httpie/

{
    "id": 2,
    "author": "demo",
    "title": "บทความจาก httpie",
    ...
}
```

### 419.5 ทดสอบ `PATCH` (partial update)

```bash
# curl
curl -i -X PATCH http://127.0.0.1:8000/api/posts/ทดสอบ-apiview/ \
  -H "Content-Type: application/json" \
  -b cookies.txt \
  -d '{"is_published": false}'

# httpie
http PATCH 127.0.0.1:8000/api/posts/ทดสอบ-apiview/ \
  is_published:=false \
  --session=demo
```

สังเกตว่า httpie ใช้ `:=` แทน `=` เมื่อค่าที่ส่งไม่ใช่ string (เช่น boolean, number,
JSON object/array) — `is_published:=false` จะถูกส่งเป็น JSON boolean `false` จริง ๆ
ในขณะที่ `is_published=false` จะถูกส่งเป็น string `"false"` ซึ่งอาจทำให้ Serializer
ตีความผิดได้

### 419.6 ทดสอบ `DELETE` และตรวจสอบ 204

```bash
curl -i -X DELETE http://127.0.0.1:8000/api/posts/ทดสอบ-apiview/ -b cookies.txt
# HTTP/1.1 204 No Content
# (ไม่มี body ตามที่เรียนในขั้นตอนที่ 417.4)

http DELETE 127.0.0.1:8000/api/posts/ทดสอบ-apiview/ --session=demo
```

### 419.7 ทดสอบกรณี error เพื่อยืนยัน Exception Handling (ขั้นตอนที่ 416)

```bash
# ส่งข้อมูลไม่ครบ (ไม่มี title) -> คาดหวัง 400
http POST 127.0.0.1:8000/api/posts/ content="ไม่มีชื่อเรื่อง" --session=demo
# HTTP/1.1 400 Bad Request
# {"success": false, "status_code": 400, "errors": {"title": ["This field is required."]}}

# เข้าถึง slug ที่ไม่มีอยู่จริง -> คาดหวัง 404
http GET 127.0.0.1:8000/api/posts/ไม่มี-บทความนี้/
# HTTP/1.1 404 Not Found
# {"success": false, "status_code": 404, "errors": {"detail": "Not found."}}

# ยิงโดยไม่ login เลย -> ลอง POST -> คาดหวัง 403 (เพราะ IsAuthenticatedOrReadOnly)
http POST 127.0.0.1:8000/api/posts/ title="ไม่ได้ login" content="..."
# HTTP/1.1 403 Forbidden
```

### 419.8 เทคนิคเสริม: `jq` สำหรับกรอง JSON response

```bash
# ติดตั้ง jq (ตัวช่วยกรอง/format JSON บน command line)
sudo apt install jq   # Ubuntu/Debian
brew install jq       # macOS

# ดึงเฉพาะ field "title" ของทุกบทความ
curl -s http://127.0.0.1:8000/api/posts/ | jq '.[].title'

# ดูเฉพาะ status code โดยไม่ต้องพิมพ์ body (มีประโยชน์ตอนเขียน script ทดสอบ)
curl -s -o /dev/null -w "%{http_code}\n" http://127.0.0.1:8000/api/posts/
```

### 419.9 ตารางสรุปคำสั่งที่ใช้บ่อยที่สุด

| งาน | curl | httpie |
|---|---|---|
| GET แบบดู header | `curl -i URL` | `http URL` |
| POST พร้อม JSON | `curl -X POST -H "Content-Type: application/json" -d '{"k":"v"}' URL` | `http POST URL k=v` |
| ส่งค่า boolean/number | เขียน JSON เองใน `-d` | `http POST URL flag:=true count:=5` |
| PATCH | `curl -X PATCH ...` | `http PATCH URL k=v` |
| DELETE | `curl -X DELETE URL` | `http DELETE URL` |
| แนบ Authorization header | `curl -H "Authorization: Token xxx" URL` | `http URL "Authorization:Token xxx"` |
| ดูเฉพาะ status code | `curl -s -o /dev/null -w "%{http_code}"` | `http --print=h URL` (ดู header ทั้งหมด) |

Part 050 (Testing REST APIs) จะสอนวิธีเขียนการทดสอบเหล่านี้ให้เป็น **automated test**
ด้วย `APIClient` ของ DRF แทนการพิมพ์คำสั่งซ้ำ ๆ ทุกครั้งด้วยมือ

---

## ขั้นตอนที่ 420: สรุปและแบบฝึกหัด

### 420.1 สรุปสิ่งที่ได้เรียนรู้ใน Part นี้

- ✅ เขียน API แบบ function-based ด้วย `@api_view` และเข้าใจว่ามันสร้าง `APIView`
  subclass แบบ dynamic ด้วย `type()` เบื้องหลัง
- ✅ เจาะลึก `APIView` ว่าสืบทอดจาก Django `View` ของ Part 021 โดยตรง และ `dispatch()`
  ของมันคือ `View.dispatch()` เดิมที่ถูก "แซนด์วิช" ด้วย `initial()` (authentication,
  permission, throttle), try/except (exception handling), และ `finalize_response()`
  (content negotiation)
- ✅ จัดการ `GET`/`POST`/`PUT`/`PATCH`/`DELETE` ในคลาสเดียวด้วย method แยกกัน พร้อม
  `get_object()` helper ที่ใช้ร่วมกันทุก method
- ✅ เข้าใจว่า `request.data` ของ DRF parse ได้ทุก Content-Type ผ่านระบบ
  `parser_classes` ต่างจาก `request.POST`/`request.body` ของ Django ธรรมดาที่ทำได้จำกัด
- ✅ เข้าใจว่า `Response` ยังไม่ render จนกว่าจะถึง `finalize_response()` และ Content
  Negotiation เลือก renderer จาก header `Accept` อย่างไร
- ✅ ใช้ `APIException`, exception class มาตรฐานของ DRF, และเขียน custom exception
  handler ผ่าน setting `EXCEPTION_HANDLER` เพื่อให้ error response มี shape เดียวกันทั้ง
  โปรเจกต์
- ✅ เลือก HTTP status code ที่ถูกต้องตามหลัก REST สำหรับแต่ละ action ของ CRUD
- ✅ สร้าง CRUD API เต็มรูปแบบสำหรับ `Post` ด้วย `APIView` ล้วน ๆ (ไม่ใช้ Generic Views)
- ✅ ทดสอบ API จริงด้วย `curl` และ `httpie` ครบทุก HTTP method รวมถึงกรณี error

### 420.2 Checklist ก่อนไป Part ถัดไป

- [ ] อธิบายได้ว่า `@api_view` สร้างอะไรขึ้นมาเบื้องหลัง และทำไมมันถึงเกี่ยวข้องกับ
  `APIView`
- [ ] อธิบายลำดับขั้นตอนทั้งหมดใน `APIView.dispatch()` ได้ตั้งแต่ `initialize_request()`
  จนถึง `finalize_response()`
- [ ] เขียน `APIView` ที่รองรับครบทั้ง 4 methods (`GET`/`PUT`/`PATCH`/`DELETE`) พร้อม
  `get_object()` helper ได้เอง
- [ ] อธิบายความแตกต่างระหว่าง `request.data`, `request.POST`, และ `request.body` ได้
  ชัดเจน พร้อมยกตัวอย่างสถานการณ์ที่แต่ละตัวเหมาะสม
- [ ] อธิบาย Content Negotiation ได้ว่า DRF เลือก renderer จาก header `Accept` อย่างไร
- [ ] เขียน custom `APIException` ของตัวเอง และเขียน custom exception handler ผ่าน
  `EXCEPTION_HANDLER` ได้
- [ ] เลือก status code ที่ถูกต้องสำหรับทุก action ของ CRUD (200/201/204/400/404 ฯลฯ)
  โดยไม่ต้องเปิดตารางดู
- [ ] สร้าง CRUD API เต็มรูปแบบสำหรับ resource หนึ่งตัวด้วย `APIView` ธรรมดาได้เอง
- [ ] ทดสอบ API ทุก endpoint ด้วย `curl` หรือ `httpie` ได้คล่อง รวมถึงกรณี error

### 420.3 แบบฝึกหัดท้ายบท

**แบบฝึกหัดที่ 1 (พื้นฐาน)**: เขียน endpoint `GET /api/posts/stats/` ด้วย `@api_view`
ที่คืนสถิติของบทความในระบบเป็น JSON เช่น `{"total": 10, "published": 7, "draft": 3}`
โดยใช้ QuerySet aggregation ที่เรียนมาจาก Part 014 (`Count`, filter) แล้วทดสอบด้วย
`httpie`

**แบบฝึกหัดที่ 2 (ประยุกต์)**: เขียน `APIView` ชื่อ `PostPublishToggleAPIView` ที่มี
เฉพาะ method `patch()` สำหรับสลับค่า `is_published` ของบทความ (ไม่ต้องรับ body ใด ๆ
จาก client) โดยต้องตรวจสอบว่า `request.user` เป็นเจ้าของบทความเท่านั้น (ใช้เทคนิคจาก
ขั้นตอนที่ 418.3) ถ้าไม่ใช่เจ้าของให้ raise `PermissionDenied` แล้วทดสอบทั้งกรณีสำเร็จ
(200) และกรณีไม่มีสิทธิ์ (403) ด้วย `curl`

**แบบฝึกหัดที่ 3 (Exception Handling)**: สร้าง custom exception ชื่อ `DuplicateSlugError`
(status code 409 `HTTP_409_CONFLICT`) แล้วแก้ `PostListAPIView.post()` ให้ raise
exception นี้เมื่อ client ส่ง `title` ที่ slugify แล้วซ้ำกับบทความที่มีอยู่แล้วในระบบ
(ตรวจสอบก่อนเรียก `serializer.save()`) ทดสอบด้วย `httpie` ว่าได้ `409 Conflict` กลับมา
พร้อมข้อความอธิบายที่ชัดเจน

**แบบฝึกหัดที่ 4 (ขั้นสูง)**: เขียน custom exception handler ของตัวเอง (ต่อยอดจาก
ขั้นตอนที่ 416.5) ที่เพิ่มการ **log** ทุก error ที่มี status code ตั้งแต่ 500 ขึ้นไปลงไฟล์
log (ใช้ Python `logging` module) ก่อนคืน response กลับไปตามปกติ โดยต้องไม่กระทบ
error response ที่ client ได้รับเลย (client ควรเห็น response เหมือนเดิมทุกประการ)
ทดสอบด้วยการสร้าง view ทดลองที่ raise `Exception` ธรรมดา (ไม่ใช่ `APIException`) แล้ว
ตรวจสอบว่า log ถูกบันทึกจริงและ client ยังได้ HTML error page ตามปกติของ Django
(เพราะ `response is None` ตามที่เรียนในขั้นตอนที่ 416.2)

### 420.4 คำถามที่พบบ่อย (FAQ)

**Q: `@api_view` กับ `APIView` อันไหนควรใช้เป็นค่าเริ่มต้นของทีม?**
A: ทีมมืออาชีพส่วนใหญ่เลือก `APIView` (หรือ Generic Views ที่จะเรียนใน Part 043) เป็น
มาตรฐานหลักของโปรเจกต์ เพราะโครงสร้างสม่ำเสมอ ทดสอบง่าย และขยายเป็น Generic View ได้ใน
ภายหลังโดยไม่ต้องเขียนใหม่ทั้งหมด ส่วน `@api_view` เก็บไว้สำหรับ endpoint พิเศษเล็ก ๆ ที่
ไม่ตรง pattern ของ resource ใด ๆ เช่น endpoint healthcheck หรือ webhook callback

**Q: ทำไมตอนเรียก `serializer.is_valid()` บางที่ใช้ `if serializer.is_valid():` บางที่ใช้
`serializer.is_valid(raise_exception=True)` ต่างกันยังไง ใช้แบบไหนดีกว่า?**
A: ทั้งสองแบบให้ผลลัพธ์สุดท้ายเหมือนกัน (คืน 400 พร้อม errors) แต่
`raise_exception=True` สั้นกว่าและสอดคล้องกับ error handling แบบ "raise แล้วให้
`dispatch()` ดักให้" ตามที่เรียนในขั้นตอนที่ 416 ในโปรเจกต์จริงแนะนำใช้
`raise_exception=True` เป็นค่าเริ่มต้นเพื่อลดโค้ดซ้ำ ยกเว้นกรณีที่ต้องการทำอะไรเพิ่มเติม
ก่อนคืน response เมื่อ validation ล้มเหลว (เช่น log เหตุผลพิเศษ) ซึ่งค่อยใช้ `if/else`
แบบเดิม

**Q: `request.data` เก็บ cache ไว้หรือเรียก parse ใหม่ทุกครั้งที่เข้าถึง?**
A: `request.data` เป็น `@property` ที่ parse body แค่ **ครั้งแรก** ที่ถูกเข้าถึง แล้ว
cache ผลลัพธ์ไว้ใน instance ของ `Request` เอง เรียกซ้ำกี่ครั้งในฟังก์ชันเดียวกันก็ได้ผล
ลัพธ์เดิมโดยไม่ parse ซ้ำ (สำคัญเพราะ request body เป็น stream ที่อ่านได้ครั้งเดียวใน
ระดับ WSGI/ASGI ถ้าไม่ cache ไว้ การเข้าถึงครั้งที่สองอาจได้ค่าว่างเปล่า)

**Q: ทำไม `PostDetailAPIView` ในขั้นตอนที่ 418 ใช้ `get_object_or_404` ของ Django
(`django.shortcuts`) แทนที่จะมีของ DRF เอง?**
A: DRF มี `rest_framework.generics.get_object_or_404` ให้เช่นกัน (ทำงานเกือบเหมือนกัน
ต่างแค่รองรับ QuerySet ที่ยังไม่ evaluate ได้ยืดหยุ่นกว่าเล็กน้อย) แต่ของ Django ธรรมดา
จาก Part 007 ก็ใช้ได้ผลลัพธ์เหมือนกันทุกประการเพราะ `handle_exception()` แปลง Django's
`Http404` เป็น DRF's `NotFound` ให้อัตโนมัติอยู่แล้ว (ตามที่เรียนในขั้นตอนที่ 416.2)
Part 043 จะแนะนำเวอร์ชันของ DRF เมื่อเริ่มทำงานกับ `queryset` attribute ของ Generic View

---

## เตรียมตัวสำหรับ Part ถัดไป

**Part 043: Generic API Views และ Mixins** จะนำ `PostListAPIView` และ `PostDetailAPIView`
ที่เขียนแบบเต็ม (manual) ใน Part นี้ไปย่อให้สั้นลงเหลือเพียงไม่กี่บรรทัด โดยใช้
`ListCreateAPIView`, `RetrieveUpdateDestroyAPIView` และ Mixin ตัวประกอบเบื้องหลังอย่าง
`ListModelMixin`, `CreateModelMixin`, `RetrieveModelMixin`, `UpdateModelMixin`,
`DestroyModelMixin` คุณจะได้เห็นว่าโค้ดที่เขียนซ้ำ ๆ อย่าง `serializer.is_valid()`,
`Response(..., status=...)`, `get_object()` ที่ทำใน Part นี้ทั้งหมด แท้จริงแล้วถูกเขียน
ไว้ให้สำเร็จรูปใน Mixin เหล่านี้อย่างไร โดยยึดหลักการเดียวกับที่ Part 022 ย่อ CBV แบบ
manual ของ Part 021 ให้กลายเป็น `ListView`/`DetailView`

เตรียมเปิด `blog/api_views.py`, `blog/serializers.py`, และ `blog/api_urls.py` ของคุณไว้
ให้พร้อม เพราะ Part 043 จะแก้ไขไฟล์เหล่านี้ต่อจากที่ Part นี้ทิ้งไว้ทันที!
