# Part 043: Generic API Views และ Mixins

> **ขั้นตอนที่ 421-430 ของหลักสูตร** | Phase 5: Django REST Framework และ API
>
> เป้าหมายของ Part นี้: นำ `PostListAPIView` และ `PostDetailAPIView` ที่เขียนด้วย
> `APIView` ล้วน ๆ ใน Part 042 มาย่อให้สั้นลงอย่างมาก โดยใช้ **Generic API Views** และ
> **Mixins** ของ Django REST Framework คุณจะเห็นว่าโค้ดที่เขียนซ้ำ ๆ ทุก view อย่าง
> `serializer.is_valid()`, `Response(..., status=...)`, `get_object()` ถูกดึงออกมาเป็น
> Mixin สำเร็จรูปอย่างไร โดยยึดหลักการเดียวกับที่ Part 024 สอนเรื่อง Mixin และ MRO ของ
> Django `View` ธรรมดา — เพียงแต่ครั้งนี้เป็น Mixin สำหรับ API ปิดท้ายด้วยการแปลง CRUD
> เต็มรูปแบบของ `Post` จาก Part 042 ให้เหลือโค้ดไม่ถึงครึ่งหนึ่งของเดิม

---

## สารบัญของ Part นี้

- ขั้นตอนที่ 421: `generics.GenericAPIView` — `queryset`, `serializer_class`, `get_queryset()`, `get_serializer()`
- ขั้นตอนที่ 422: DRF Mixins — `ListModelMixin`, `CreateModelMixin`, `RetrieveModelMixin`, `UpdateModelMixin`, `DestroyModelMixin`
- ขั้นตอนที่ 423: การผสม Mixin + `GenericAPIView` เอง (pattern ของ `ListCreateAPIView`)
- ขั้นตอนที่ 424: Convenience class ที่ DRF เตรียมไว้ให้ — `ListCreateAPIView`, `RetrieveUpdateDestroyAPIView` ฯลฯ
- ขั้นตอนที่ 425: Override `get_queryset()`/`get_serializer_class()` แบบ dynamic ตาม action
- ขั้นตอนที่ 426: `perform_create()`, `perform_update()`, `perform_destroy()` hooks
- ขั้นตอนที่ 427: `lookup_field`/`lookup_url_kwarg` customization — ใช้ slug แทน pk
- ขั้นตอนที่ 428: ผสาน Generic View กับ custom filtering logic ของตัวเอง
- ขั้นตอนที่ 429: แปลง CRUD จาก `APIView` ธรรมดา (Part 042) ให้เป็น Generic Views เต็มรูปแบบ
- ขั้นตอนที่ 430: สรุปและแบบฝึกหัด

---

## ขั้นตอนที่ 421: `generics.GenericAPIView` — `queryset`, `serializer_class`, `get_queryset()`, `get_serializer()`

### 421.1 ทบทวนสิ่งที่ Part 042 ทิ้งไว้

ใน Part 042 ขั้นตอนที่ 418 เราสร้าง `PostListAPIView` และ `PostDetailAPIView` ด้วย
`APIView` ล้วน ๆ โค้ดหน้าตาประมาณนี้ (ย่อเฉพาะส่วนที่เกี่ยวข้อง):

```python
# blog/api_views.py (จาก Part 042 — เวอร์ชัน APIView ธรรมดา)
class PostListAPIView(APIView):
    def get(self, request):
        posts = Post.objects.filter(is_published=True)
        serializer = PostSerializer(posts, many=True)
        return Response(serializer.data, status=status.HTTP_200_OK)

    def post(self, request):
        serializer = PostSerializer(data=request.data)
        serializer.is_valid(raise_exception=True)
        post = serializer.save(author=request.user)
        return Response(PostSerializer(post).data, status=status.HTTP_201_CREATED)
```

สังเกตว่าเกือบทุก method ทำสิ่งเดียวกันในรูปแบบซ้ำ ๆ: **ดึง queryset มา → สร้าง
serializer ผูกกับมัน → validate/save → คืน `Response`** รูปแบบที่ซ้ำกันแบบนี้คือสัญญาณ
คลาสสิกที่บอกว่าถึงเวลาดึงมันออกมาเป็นเมธอด/class กลาง — เหมือนกับที่ Part 022 ดึงโค้ด
`get()`/`post()` ของ FBV แบบ manual ออกมาเป็น `ListView`/`CreateView` และ Part 024
เจาะลึกว่า Generic CBV เหล่านั้นเป็น "ต้นไม้ Mixin" ที่ประกอบจาก
`SingleObjectMixin`, `MultipleObjectMixin`, `FormMixin` ฯลฯ

DRF ใช้แนวคิดเดียวกันเป๊ะ ๆ กับ API Views โดยมี **`GenericAPIView`** เป็นฐานราก

### 421.2 `GenericAPIView` สืบทอดจาก `APIView` ที่เรียนใน Part 042 โดยตรง

```python
# แนวคิดจากซอร์สโค้ดจริงของ rest_framework.generics.GenericAPIView
from rest_framework import views
from rest_framework.settings import api_settings


class GenericAPIView(views.APIView):
    queryset = None
    serializer_class = None

    lookup_field = 'pk'
    lookup_url_kwarg = None

    filter_backends = api_settings.DEFAULT_FILTER_BACKENDS
    pagination_class = api_settings.DEFAULT_PAGINATION_CLASS
```

ข้อเท็จจริงสำคัญเช่นเดียวกับที่เห็นใน Part 042 ขั้นตอนที่ 412.2 (`APIView` สืบทอดจาก
Django `View` โดยตรง): **`GenericAPIView` ไม่ใช่ระบบใหม่ที่แยกขาดจาก `APIView`
แต่เป็น `APIView` เดิมที่เพิ่ม class attribute และ helper methods เข้าไปอีกชั้นหนึ่ง**
`dispatch()`, `initial()`, `handle_exception()`, `finalize_response()` ที่เจาะลึกไปแล้ว
ทั้งหมดใน Part 042 **ยังทำงานเหมือนเดิมทุกประการ** — `GenericAPIView` ไม่ได้แตะ
กลไกเหล่านั้นเลยแม้แต่บรรทัดเดียว มันแค่เพิ่มเมธอดใหม่ที่ช่วยเรื่อง queryset/serializer

### 421.3 `queryset` และ `get_queryset()`

```python
# แนวคิดจากซอร์สโค้ดจริงของ rest_framework.generics.GenericAPIView
from django.db.models import QuerySet


class GenericAPIView(views.APIView):
    queryset = None

    def get_queryset(self):
        assert self.queryset is not None, (
            "'%s' should either include a `queryset` attribute, "
            "or override the `get_queryset()` method."
            % self.__class__.__name__
        )
        queryset = self.queryset
        if isinstance(queryset, QuerySet):
            # สำคัญมาก: .all() สร้าง QuerySet ใหม่ทุกครั้ง ไม่ใช้ queryset ตัวเดิม
            # ที่ถูก cache ไว้ตอน import class — ป้องกันปัญหา queryset ค้างข้าม request
            queryset = queryset.all()
        return queryset
```

จุดที่มือใหม่พลาดบ่อยที่สุด: `queryset = Post.objects.filter(is_published=True)` ที่ตั้ง
เป็น **class attribute** จะถูก evaluate แค่ **ครั้งเดียว** ตอนโหลด module (แม้ QuerySet
จะเป็น lazy แต่ตัว object ยังเป็นตัวเดียวกันข้าม request) `get_queryset()` เวอร์ชัน
default จึงเรียก `.all()` ซ้ำเสมอเพื่อคืน QuerySet ใหม่ทุกครั้งที่ถูกเรียก ทำให้แต่ละ
request query ฐานข้อมูลสดใหม่จริง ๆ ไม่ค้างผลลัพธ์จาก request ก่อนหน้า

```python
# blog/api_views.py
from rest_framework import generics
from .models import Post
from .serializers import PostSerializer


class PostListAPIView(generics.GenericAPIView):
    queryset = Post.objects.filter(is_published=True)
    serializer_class = PostSerializer

    def get(self, request):
        posts = self.get_queryset()          # แทนที่ Post.objects.filter(...) ตรง ๆ
        serializer = self.get_serializer(posts, many=True)
        return Response(serializer.data)
```

**ทางเลือก: override `get_queryset()` แทนการตั้ง `queryset` แบบ static** เมื่อ query
ต้องพึ่งพา `self.request` (เช่น กรองตาม user ที่ล็อกอิน) ซึ่งเป็นแนวทางที่แนะนำเสมอเมื่อ
query ไม่ใช่ค่าคงที่:

```python
class PostListAPIView(generics.GenericAPIView):
    serializer_class = PostSerializer

    def get_queryset(self):
        # ปลอดภัยกว่า: query สดใหม่ทุกครั้ง และเข้าถึง self.request ได้
        return Post.objects.filter(is_published=True).select_related('author')
```

### 421.4 `serializer_class` และ `get_serializer()`/`get_serializer_class()`

```python
# แนวคิดจากซอร์สโค้ดจริงของ rest_framework.generics.GenericAPIView
class GenericAPIView(views.APIView):
    serializer_class = None

    def get_serializer(self, *args, **kwargs):
        serializer_class = self.get_serializer_class()
        kwargs.setdefault('context', self.get_serializer_context())
        return serializer_class(*args, **kwargs)

    def get_serializer_class(self):
        assert self.serializer_class is not None, (
            "'%s' should either include a `serializer_class` attribute, "
            "or override the `get_serializer_class()` method."
            % self.__class__.__name__
        )
        return self.serializer_class

    def get_serializer_context(self):
        return {
            'request': self.request,
            'format': self.format_kwarg,
            'view': self,
        }
```

สามเมธอดนี้ทำงานร่วมกันเป็นชั้น ๆ:

1. `get_serializer_class()` — เลือกว่าจะใช้ Serializer **class** ไหน (default: อ่านจาก
   `self.serializer_class` ตรง ๆ — จะ override ให้ dynamic ในขั้นตอนที่ 425)
2. `get_serializer_context()` — เตรียม **context dict** ที่ทุก Serializer ที่สร้างผ่าน
   `get_serializer()` จะได้รับ โดยเฉพาะ `'request': self.request` ที่สำคัญมาก เพราะทำให้
   Serializer เข้าถึง `request.user` ได้ผ่าน `self.context['request'].user`
   (ใช้บ่อยเวลาต้องเขียน field ที่ค่าขึ้นกับ user ปัจจุบัน เช่น
   `is_liked_by_me = serializers.SerializerMethodField()`)
3. `get_serializer()` — จุดที่ควร **ใช้แทน `PostSerializer(...)` ตรง ๆ เสมอ** เพราะมัน
   ผูก context ให้อัตโนมัติ และเปิดทางให้ override `get_serializer_class()` แบบ dynamic
   ได้โดยไม่ต้องแก้โค้ดจุดที่เรียกใช้เลย

```python
# blog/api_views.py
class PostListAPIView(generics.GenericAPIView):
    queryset = Post.objects.filter(is_published=True)
    serializer_class = PostSerializer

    def get(self, request):
        posts = self.get_queryset()
        # self.get_serializer(...) แทน PostSerializer(...) ตรง ๆ
        serializer = self.get_serializer(posts, many=True)
        return Response(serializer.data)

    def post(self, request):
        serializer = self.get_serializer(data=request.data)
        serializer.is_valid(raise_exception=True)
        post = serializer.save(author=request.user)
        return Response(
            self.get_serializer(post).data,
            status=status.HTTP_201_CREATED,
        )
```

### 421.5 `get_object()`: ดึง object เดียวพร้อมเช็ค permission ให้อัตโนมัติ

```python
# แนวคิดจากซอร์สโค้ดจริงของ rest_framework.generics.GenericAPIView
from django.shortcuts import get_object_or_404 as _get_object_or_404


class GenericAPIView(views.APIView):
    lookup_field = 'pk'
    lookup_url_kwarg = None

    def get_object(self):
        queryset = self.filter_queryset(self.get_queryset())

        lookup_url_kwarg = self.lookup_url_kwarg or self.lookup_field
        assert lookup_url_kwarg in self.kwargs, (
            'Expected view %s to be called with a URL keyword argument '
            'named "%s". Fix your URL conf, or set the `.lookup_field` '
            'attribute on the view correctly.'
            % (self.__class__.__name__, lookup_url_kwarg)
        )

        filter_kwargs = {self.lookup_field: self.kwargs[lookup_url_kwarg]}
        obj = _get_object_or_404(queryset, **filter_kwargs)

        # ตรวจสอบ object-level permission ให้อัตโนมัติ (จะเรียนเต็มรูปแบบใน Part 045)
        self.check_object_permissions(self.request, obj)
        return obj
```

เทียบกับ `PostDetailAPIView.get_object()` ที่เขียนเองใน Part 042 ขั้นตอนที่ 418.3
(`get_object_or_404(Post, slug=slug)` + เรียก `check_object_permissions()` เอง)
`GenericAPIView.get_object()` ทำสิ่งเดียวกันให้ครบทุกขั้นตอน **โดยอัตโนมัติ** รวมถึง
`filter_queryset()` (ขั้นตอนที่ 428) ก่อนหา object ด้วย — นี่คือเหตุผลที่ Part 042
ขั้นตอนที่ 420.4 (FAQ ข้อสุดท้าย) บอกไว้ล่วงหน้าว่า "Part 043 จะแนะนำเวอร์ชันของ DRF
เมื่อเริ่มทำงานกับ `queryset` attribute ของ Generic View"

**หมายเหตุ**: ค่า default ของ `lookup_field` คือ `'pk'` เราจะปรับให้ใช้ `'slug'` แทนใน
ขั้นตอนที่ 427

### 421.6 ตารางสรุป: `APIView` (Part 042) เทียบกับ `GenericAPIView` (Part นี้)

| ความสามารถ | `APIView` (Part 042) | `GenericAPIView` (Part นี้) |
|---|---|---|
| `dispatch()`, `initial()`, exception handling | ✅ มี (สืบทอดจาก Django `View`) | ✅ มีเหมือนกันทุกประการ (สืบทอดต่อมาอีกที) |
| ดึง QuerySet | เขียน `Post.objects.filter(...)` เอง | `self.queryset` + `get_queryset()` มาตรฐาน |
| สร้าง Serializer | เขียน `PostSerializer(...)` เอง | `self.serializer_class` + `get_serializer()` (ผูก context ให้อัตโนมัติ) |
| ดึง object เดียว | เขียน `get_object_or_404()` + เช็ค permission เอง | `self.get_object()` ทำให้ครบทุกขั้นตอน |
| กรองข้อมูลเพิ่มเติม (search, ordering) | เขียนเอง | `filter_backends` + `filter_queryset()` (ขั้นตอนที่ 428) |
| Pagination | เขียนเอง | `pagination_class` + `paginate_queryset()` (Part 044/047) |
| ต้องเขียน `get()`/`post()`/ฯลฯ เอง | ✅ ต้องเขียน | ✅ ยังต้องเขียน (จะหมดไปเมื่อผสม Mixin ในขั้นตอนที่ 422-423) |

สังเกตแถวสุดท้าย: `GenericAPIView` เพียงอย่างเดียว **ยังไม่ลดโค้ดใน `get()`/`post()`
เลย** มันแค่ให้เครื่องมือ (`get_queryset()`, `get_serializer()`, `get_object()`) ที่
Mixin ในขั้นตอนถัดไปจะ "ใช้" เพื่อเขียน `list()`, `create()`, `retrieve()` ฯลฯ ให้สำเร็จ
รูปสมบูรณ์ — เหมือนกับที่ Part 024 ขั้นตอนที่ 231.3 อธิบายว่า `SingleObjectMixin` เอง
ไม่ได้ทำให้ `DetailView` เสร็จสมบูรณ์ ต้องผสมกับ `TemplateResponseMixin` อีกที

---

## ขั้นตอนที่ 422: DRF Mixins — `ListModelMixin`, `CreateModelMixin`, `RetrieveModelMixin`, `UpdateModelMixin`, `DestroyModelMixin`

### 422.1 ย้อนกลับไปที่แนวคิด Mixin ของ Part 024

Part 024 ขั้นตอนที่ 231.4 มีตารางสรุป Mixin ที่ประกอบกันเป็น Generic CBV ของ Django เอง
(`SingleObjectMixin`, `MultipleObjectMixin`, `FormMixin` ฯลฯ) DRF ใช้สถาปัตยกรรมแบบ
เดียวกันเป๊ะสำหรับ API — มี Mixin ห้าตัวใน `rest_framework.mixins` ที่แต่ละตัวรับผิดชอบ
"หนึ่ง action ของ CRUD" เท่านั้น:

| DRF Mixin (`rest_framework.mixins`) | Method ที่เพิ่มให้ | เทียบเท่า Django Mixin (Part 024) |
|---|---|---|
| `ListModelMixin` | `.list(request)` | `MultipleObjectMixin` (ให้ `ListView`) |
| `CreateModelMixin` | `.create(request)` | `ModelFormMixin` ฝั่งสร้างใหม่ (ให้ `CreateView`) |
| `RetrieveModelMixin` | `.retrieve(request)` | `SingleObjectMixin` (ให้ `DetailView`) |
| `UpdateModelMixin` | `.update(request)`, `.partial_update(request)` | `ModelFormMixin` ฝั่งแก้ไข (ให้ `UpdateView`) |
| `DestroyModelMixin` | `.destroy(request)` | `DeletionMixin` (ให้ `DeleteView`) |

ข้อแตกต่างสำคัญจาก Django Mixin ใน Part 024: **Mixin เหล่านี้ไม่ได้ผูกกับ HTTP method
โดยตรง** สังเกตว่าชื่อเมธอดคือ `.list()`, `.create()` ไม่ใช่ `.get()`, `.post()` — ทุกตัว
**ต้องพึ่งพา** `GenericAPIView` ที่มี `get_queryset()`, `get_serializer()`, `get_object()`
อยู่แล้ว (เหมือนที่ `AuthorRequiredMixin` ใน Part 024 ขั้นตอนที่ 232.2 พึ่งพา
`self.get_object()` จาก `SingleObjectMixin`) และ **ใครเป็นคนเรียกเมธอดเหล่านี้จริง ๆ
ตอน HTTP request เข้ามา** คือสิ่งที่ขั้นตอนที่ 423 จะตอบ

### 422.2 `ListModelMixin`: ให้ `.list()`

```python
# แนวคิดจากซอร์สโค้ดจริงของ rest_framework.mixins.ListModelMixin
from rest_framework.response import Response


class ListModelMixin:
    """
    List a queryset.
    """
    def list(self, request, *args, **kwargs):
        queryset = self.filter_queryset(self.get_queryset())

        page = self.paginate_queryset(queryset)
        if page is not None:
            serializer = self.get_serializer(page, many=True)
            return self.get_paginated_response(serializer.data)

        serializer = self.get_serializer(queryset, many=True)
        return Response(serializer.data)
```

โค้ดนี้คือสิ่งที่เราเขียนเองใน `PostListAPIView.get()` ของ Part 042 ทุกประการ (ดึง
queryset → serialize แบบ `many=True` → คืน `Response`) บวกกับ pagination ที่ยังไม่ได้
เรียนอย่างละเอียด (จะเจาะลึกเต็มรูปแบบใน Part 047) — ถ้า `pagination_class` ไม่ได้ตั้งไว้
`self.paginate_queryset(queryset)` จะคืน `None` ทำให้ข้ามไปคืนผลลัพธ์แบบไม่แบ่งหน้าตามปกติ

### 422.3 `CreateModelMixin`: ให้ `.create()`

```python
# แนวคิดจากซอร์สโค้ดจริงของ rest_framework.mixins.CreateModelMixin
from rest_framework import status
from rest_framework.response import Response


class CreateModelMixin:
    """
    Create a model instance.
    """
    def create(self, request, *args, **kwargs):
        serializer = self.get_serializer(data=request.data)
        serializer.is_valid(raise_exception=True)
        self.perform_create(serializer)
        headers = self.get_success_headers(serializer.data)
        return Response(serializer.data, status=status.HTTP_201_CREATED, headers=headers)

    def perform_create(self, serializer):
        serializer.save()

    def get_success_headers(self, data):
        try:
            return {'Location': str(data[api_settings.URL_FIELD_NAME])}
        except (TypeError, KeyError):
            return {}
```

จุดสำคัญที่สุดของ Mixin นี้คือการแยก **`create()`** (ควบคุม flow ทั้งหมด: validate →
save → response) ออกจาก **`perform_create()`** (ทำแค่ `.save()`) — การแยกนี้คือกุญแจ
สำคัญของขั้นตอนที่ 426 เพราะ `perform_create()` คือจุดที่เราจะ override เพื่อใส่
`author=self.request.user` โดยไม่ต้องแตะ logic การ validate/response เลย

### 422.4 `RetrieveModelMixin`: ให้ `.retrieve()`

```python
# แนวคิดจากซอร์สโค้ดจริงของ rest_framework.mixins.RetrieveModelMixin
class RetrieveModelMixin:
    """
    Retrieve a model instance.
    """
    def retrieve(self, request, *args, **kwargs):
        instance = self.get_object()
        serializer = self.get_serializer(instance)
        return Response(serializer.data)
```

ตัวที่สั้นที่สุดในบรรดา Mixin ทั้งห้าตัว เพราะ `self.get_object()` ของ `GenericAPIView`
(ขั้นตอนที่ 421.5) ทำงานหนักให้หมดแล้ว (หา object + เช็ค permission)

### 422.5 `UpdateModelMixin`: ให้ `.update()` และ `.partial_update()`

```python
# แนวคิดจากซอร์สโค้ดจริงของ rest_framework.mixins.UpdateModelMixin
class UpdateModelMixin:
    """
    Update a model instance.
    """
    def update(self, request, *args, **kwargs):
        partial = kwargs.pop('partial', False)
        instance = self.get_object()
        serializer = self.get_serializer(instance, data=request.data, partial=partial)
        serializer.is_valid(raise_exception=True)
        self.perform_update(serializer)

        if getattr(instance, '_prefetched_objects_cache', None):
            # ถ้ามี prefetch_related() ค้างอยู่ ต้องล้าง cache เพราะ instance
            # อาจถูกแก้ field ที่กระทบความสัมพันธ์ที่ prefetch ไว้
            instance._prefetched_objects_cache = {}

        return Response(serializer.data)

    def perform_update(self, serializer):
        serializer.save()

    def partial_update(self, request, *args, **kwargs):
        kwargs['partial'] = True
        return self.update(request, *args, **kwargs)
```

นี่คือคำตอบเชิงกลไกของสิ่งที่ Part 042 ขั้นตอนที่ 413.3 อธิบายไว้แค่ระดับพฤติกรรม:
**`PATCH` ก็คือ `PUT` ตัวเดียวกัน เพียงแต่ตั้ง `partial=True` ก่อนเรียก** — สังเกตว่า
`partial_update()` **ไม่ได้เขียน logic ซ้ำ** เลยแม้แต่บรรทัดเดียว มันแค่ตั้งค่า
`kwargs['partial'] = True` แล้วโยนต่อให้ `update()` ทำงานแทนทั้งหมด — ตัวอย่างที่ดีของ
การ reuse โค้ดแบบ "compose เป็นชั้น ๆ" เหมือนที่ `AjaxableResponseMixin` ใน Part 024
ขั้นตอนที่ 237.4 เรียก `super().form_valid()` ก่อนแล้วค่อยเพิ่มพฤติกรรม

### 422.6 `DestroyModelMixin`: ให้ `.destroy()`

```python
# แนวคิดจากซอร์สโค้ดจริงของ rest_framework.mixins.DestroyModelMixin
class DestroyModelMixin:
    """
    Destroy a model instance.
    """
    def destroy(self, request, *args, **kwargs):
        instance = self.get_object()
        self.perform_destroy(instance)
        return Response(status=status.HTTP_204_NO_CONTENT)

    def perform_destroy(self, instance):
        instance.delete()
```

เช่นเดียวกับ `CreateModelMixin`/`UpdateModelMixin` มีการแยก `destroy()` (flow ทั้งหมด)
ออกจาก `perform_destroy()` (แค่ `.delete()`) — จุดนี้จะถูกใช้ในขั้นตอนที่ 426 เพื่อทำ
**soft delete** (ตั้ง flag แทนการลบจริง) โดยไม่ต้องแก้ `destroy()` เลย

### 422.7 ตารางสรุปทั้งห้า Mixin พร้อม dependency ที่แต่ละตัวต้องการ

| Mixin | เมธอดที่ได้ | ต้องพึ่งพาอะไรจาก `GenericAPIView` | HTTP status สำเร็จ |
|---|---|---|---|
| `ListModelMixin` | `.list()` | `get_queryset()`, `filter_queryset()`, `get_serializer()`, `paginate_queryset()` | 200 |
| `CreateModelMixin` | `.create()` | `get_serializer()` | 201 |
| `RetrieveModelMixin` | `.retrieve()` | `get_object()`, `get_serializer()` | 200 |
| `UpdateModelMixin` | `.update()`, `.partial_update()` | `get_object()`, `get_serializer()` | 200 |
| `DestroyModelMixin` | `.destroy()` | `get_object()` | 204 |

**ข้อสังเกตสำคัญที่เชื่อมกับ Part 024**: Mixin ทั้งห้าตัวนี้ **"พึ่งพา" เมธอดจาก
`GenericAPIView` เหมือนที่ `AuthorRequiredMixin` พึ่งพา `self.get_object()` จาก
`SingleObjectMixin`** — มันจึง **ใช้เดี่ยว ๆ ไม่ได้เลย** ถ้าไม่ผสมกับ `GenericAPIView`
(หรือ class ลูกของมัน) ทดลองพิสูจน์:

```python
>>> from rest_framework import mixins
>>> class BrokenView(mixins.ListModelMixin):
...     pass
...
>>> BrokenView().list(None)
Traceback (most recent call last):
  ...
AttributeError: 'BrokenView' object has no attribute 'get_queryset'
```

`ListModelMixin` เรียก `self.get_queryset()` และ `self.filter_queryset()` ตรง ๆ โดยไม่
ตรวจสอบว่ามีอยู่จริงหรือไม่ (Python ไม่เช็ค interface ล่วงหน้าแบบภาษา statically-typed)
— นี่คือ **contract แบบไม่เป็นทางการ** ที่ Mixin กำหนดไว้ เหมือนที่ Part 024 ขั้นตอนที่
239.4 แนะนำว่า "เขียน docstring อธิบาย dependency ของ Mixin เสมอ"

---

## ขั้นตอนที่ 423: การผสม Mixin + `GenericAPIView` เอง (pattern ของ `ListCreateAPIView`)

### 423.1 ประกอบ Mixin เข้ากับ `GenericAPIView` ด้วยมือ

ตอนนี้เรามีทั้ง "เครื่องมือ" (`GenericAPIView` จากขั้นตอนที่ 421) และ "action สำเร็จรูป"
(Mixin ทั้งห้าจากขั้นตอนที่ 422) สิ่งที่ขาดไปคือ **การเชื่อม HTTP method
(`GET`/`POST`/...) เข้ากับเมธอดของ Mixin (`.list()`/`.create()`/...)** — DRF ไม่ได้ทำ
สิ่งนี้ให้อัตโนมัติแบบเวทมนตร์ เราต้องเขียน `get()`/`post()` เอง แต่ให้มัน **เรียกต่อ**
ไปยังเมธอดของ Mixin แทนที่จะเขียน logic ทั้งหมดเอง:

```python
# blog/api_views.py
from rest_framework import generics, mixins
from .models import Post
from .serializers import PostSerializer


class PostListCreateAPIView(
    mixins.ListModelMixin,
    mixins.CreateModelMixin,
    generics.GenericAPIView,
):
    queryset = Post.objects.filter(is_published=True)
    serializer_class = PostSerializer

    def get(self, request, *args, **kwargs):
        return self.list(request, *args, **kwargs)

    def post(self, request, *args, **kwargs):
        return self.create(request, *args, **kwargs)
```

โค้ดนี้ **ทำงานเหมือนกับ `PostListAPIView` เวอร์ชันเต็มของ Part 042 ทุกประการ** (list +
create, สถานะ 200/201, validation, exception handling ทั้งหมด) แต่สั้นกว่ามาก เพราะ
`get()`/`post()` แค่ "ส่งต่อ" งานให้ Mixin ทำ — เหมือนกับที่ `PostUpdateView` ใน Part
024 ไม่ต้องเขียน `get()`/`post()` เองเลย เพราะ `ProcessFormView` (ส่วนหนึ่งของ
`UpdateView`) จัดการให้แล้ว

### 423.2 ตรวจสอบ MRO ของ class ที่เพิ่งสร้าง

เช่นเดียวกับที่ Part 024 ขั้นตอนที่ 231.3 ใช้ `.__mro__` ตรวจสอบ `CreateView` ของ
Django มาลองตรวจ `PostListCreateAPIView` ของเราบ้าง:

```python
>>> from blog.api_views import PostListCreateAPIView
>>> for cls in PostListCreateAPIView.__mro__:
...     print(cls.__module__, "->", cls.__name__)
```

ผลลัพธ์ (ย่อ):

```
blog.api_views PostListCreateAPIView
rest_framework.mixins ListModelMixin
rest_framework.mixins CreateModelMixin
rest_framework.generics GenericAPIView
rest_framework.views APIView
django.views.generic.base View
builtins object
```

สังเกตความคล้ายกับ MRO ของ `CreateView` ใน Part 024 ขั้นตอนที่ 231.3 อย่างชัดเจน:
Mixin เรียงต่อกันทางซ้าย ตามด้วย base class (`GenericAPIView` → `APIView` → `View`)
อยู่ขวาสุด **หลักการ "Mixin ต้องอยู่ก่อน base view เสมอ" จาก Part 024 ขั้นตอนที่ 233.1
ใช้ได้กับ DRF Generic Views เป๊ะ ๆ เช่นกัน**

### 423.3 นี่คือ pattern เดียวกับที่ DRF ใช้สร้าง `ListCreateAPIView` เอง

ความลับที่ทำให้ขั้นตอนนี้สำคัญที่สุดของ Part นี้: `PostListCreateAPIView` ที่เราเพิ่ง
เขียนด้วยมือ **มีโครงสร้างเหมือนกับ `rest_framework.generics.ListCreateAPIView` ทุก
ประการ** มาดูซอร์สโค้ดจริงของมันเทียบกัน:

```python
# แนวคิดจากซอร์สโค้ดจริงของ rest_framework.generics.ListCreateAPIView
class ListCreateAPIView(mixins.ListModelMixin,
                         mixins.CreateModelMixin,
                         GenericAPIView):
    """
    Concrete view for listing a queryset or creating a model instance.
    """
    def get(self, request, *args, **kwargs):
        return self.list(request, *args, **kwargs)

    def post(self, request, *args, **kwargs):
        return self.create(request, *args, **kwargs)
```

**เหมือนกันทุกตัวอักษร** — DRF ไม่ได้มีกลไกพิเศษอะไรซ่อนอยู่เบื้องหลัง
`ListCreateAPIView` เลย มันคือ "Mixin + GenericAPIView + `get()`/`post()` ที่ส่งต่อ"
แบบเดียวกับที่เราเพิ่งเขียนเอง เพียงแต่ DRF เตรียมชื่อ class มาตรฐานไว้ให้แล้วเพื่อไม่
ต้องเขียนซ้ำทุกโปรเจกต์ — ขั้นตอนที่ 424 จะแนะนำ class สำเร็จรูปเหล่านี้ทั้งหมด

### 423.4 ตารางสรุปขั้นตอนการประกอบ Generic View ด้วยมือ

| ขั้นตอน | ทำอะไร | ตัวอย่างในโค้ด |
|---|---|---|
| 1. เลือก base | `generics.GenericAPIView` เสมอ | `class X(..., generics.GenericAPIView):` |
| 2. เลือก Mixin ตาม action ที่ต้องการ | วางไว้ **ก่อน** `GenericAPIView` | `mixins.ListModelMixin, mixins.CreateModelMixin` |
| 3. ตั้งค่า `queryset`/`serializer_class` | class attribute หรือ override เมธอด | `queryset = Post.objects.filter(...)` |
| 4. เขียน `get()`/`post()`/ฯลฯ ที่ส่งต่อ | เรียก `self.list()`, `self.create()` ตรง ๆ | `def get(self, request, *a, **kw): return self.list(request, *a, **kw)` |

ในทางปฏิบัติจริง **แทบไม่มีใครเขียนแบบขั้นตอนที่ 423.1 ด้วยมือ** เพราะ DRF เตรียม
convenience class ที่รวมขั้นตอน 1-2-4 ไว้ให้แล้ว เหลือแค่ขั้นตอนที่ 3 ให้เราทำเอง — แต่
การเข้าใจว่ามันประกอบกันอย่างไร (เหมือนที่ Part 024 สอน MRO ของ `CreateView`) คือสิ่งที่
ทำให้เรา debug และ override พฤติกรรมเฉพาะจุดได้อย่างมั่นใจในขั้นตอนที่ 425-428

---

## ขั้นตอนที่ 424: Convenience class ที่ DRF เตรียมไว้ให้ — `ListCreateAPIView`, `RetrieveUpdateDestroyAPIView`, `RetrieveUpdateAPIView` ฯลฯ

### 424.1 DRF เตรียม class สำเร็จรูปไว้ 9 ตัว

`rest_framework.generics` มี concrete view class สำเร็จรูปที่ประกอบ Mixin +
`GenericAPIView` + `get()`/`post()`/ฯลฯ ไว้ให้ครบทุก combination ที่ใช้บ่อยในทางปฏิบัติ
แล้ว:

```python
# แนวคิดจากซอร์สโค้ดจริงของ rest_framework.generics (ทั้ง 9 class)
class CreateAPIView(mixins.CreateModelMixin, GenericAPIView):
    def post(self, request, *args, **kwargs):
        return self.create(request, *args, **kwargs)


class ListAPIView(mixins.ListModelMixin, GenericAPIView):
    def get(self, request, *args, **kwargs):
        return self.list(request, *args, **kwargs)


class RetrieveAPIView(mixins.RetrieveModelMixin, GenericAPIView):
    def get(self, request, *args, **kwargs):
        return self.retrieve(request, *args, **kwargs)


class DestroyAPIView(mixins.DestroyModelMixin, GenericAPIView):
    def delete(self, request, *args, **kwargs):
        return self.destroy(request, *args, **kwargs)


class UpdateAPIView(mixins.UpdateModelMixin, GenericAPIView):
    def put(self, request, *args, **kwargs):
        return self.update(request, *args, **kwargs)

    def patch(self, request, *args, **kwargs):
        return self.partial_update(request, *args, **kwargs)


class ListCreateAPIView(mixins.ListModelMixin,
                         mixins.CreateModelMixin,
                         GenericAPIView):
    def get(self, request, *args, **kwargs):
        return self.list(request, *args, **kwargs)

    def post(self, request, *args, **kwargs):
        return self.create(request, *args, **kwargs)


class RetrieveUpdateAPIView(mixins.RetrieveModelMixin,
                             mixins.UpdateModelMixin,
                             GenericAPIView):
    def get(self, request, *args, **kwargs):
        return self.retrieve(request, *args, **kwargs)

    def put(self, request, *args, **kwargs):
        return self.update(request, *args, **kwargs)

    def patch(self, request, *args, **kwargs):
        return self.partial_update(request, *args, **kwargs)


class RetrieveDestroyAPIView(mixins.RetrieveModelMixin,
                              mixins.DestroyModelMixin,
                              GenericAPIView):
    def get(self, request, *args, **kwargs):
        return self.retrieve(request, *args, **kwargs)

    def delete(self, request, *args, **kwargs):
        return self.destroy(request, *args, **kwargs)


class RetrieveUpdateDestroyAPIView(mixins.RetrieveModelMixin,
                                    mixins.UpdateModelMixin,
                                    mixins.DestroyModelMixin,
                                    GenericAPIView):
    def get(self, request, *args, **kwargs):
        return self.retrieve(request, *args, **kwargs)

    def put(self, request, *args, **kwargs):
        return self.update(request, *args, **kwargs)

    def patch(self, request, *args, **kwargs):
        return self.partial_update(request, *args, **kwargs)

    def delete(self, request, *args, **kwargs):
        return self.destroy(request, *args, **kwargs)
```

### 424.2 ตารางสรุปทั้ง 9 class: HTTP method ที่รองรับ และ Mixin ที่ประกอบกัน

| Class | Method ที่รองรับ | ประกอบจาก Mixin | เหมาะกับ endpoint แบบไหน |
|---|---|---|---|
| `CreateAPIView` | `POST` | `CreateModelMixin` | สร้างอย่างเดียว เช่น endpoint submit ฟอร์มติดต่อ |
| `ListAPIView` | `GET` (list) | `ListModelMixin` | อ่านอย่างเดียว ไม่ให้แก้ไข เช่น รายการจังหวัด/หมวดหมู่ read-only |
| `RetrieveAPIView` | `GET` (detail) | `RetrieveModelMixin` | ดูรายละเอียดอย่างเดียว |
| `DestroyAPIView` | `DELETE` | `DestroyModelMixin` | ลบอย่างเดียว (พบน้อย มักรวมกับตัวอื่น) |
| `UpdateAPIView` | `PUT`, `PATCH` | `UpdateModelMixin` | แก้ไขอย่างเดียว ไม่ให้ลบ/ดูรายละเอียดแยก |
| **`ListCreateAPIView`** | `GET` (list), `POST` | `ListModelMixin` + `CreateModelMixin` | **endpoint ระดับ "collection"** เช่น `/api/posts/` |
| `RetrieveUpdateAPIView` | `GET`, `PUT`, `PATCH` | `RetrieveModelMixin` + `UpdateModelMixin` | ดูและแก้ไขได้ แต่ห้ามลบผ่าน API |
| `RetrieveDestroyAPIView` | `GET`, `DELETE` | `RetrieveModelMixin` + `DestroyModelMixin` | ดูและลบได้ แต่แก้ไขต้องผ่านช่องทางอื่น |
| **`RetrieveUpdateDestroyAPIView`** | `GET`, `PUT`, `PATCH`, `DELETE` | `RetrieveModelMixin` + `UpdateModelMixin` + `DestroyModelMixin` | **endpoint ระดับ "detail"** เช่น `/api/posts/<slug>/` |

สอง class ตัวหนา (`ListCreateAPIView`, `RetrieveUpdateDestroyAPIView`) คือคู่ที่ใช้บ่อย
ที่สุดในโลกจริง เพราะตรงกับ pattern REST มาตรฐาน "list+create" / "retrieve+update+destroy"
ที่ Part 042 ขั้นตอนที่ 413.1 อธิบายไว้ทุกประการ

### 424.3 เขียน `PostListAPIView`/`PostDetailAPIView` ใหม่ด้วย convenience class

```python
# blog/api_views.py
from rest_framework import generics, permissions
from .models import Post
from .serializers import PostSerializer


class PostListAPIView(generics.ListCreateAPIView):
    """
    GET  /api/posts/  -> รายการบทความที่เผยแพร่แล้ว
    POST /api/posts/  -> สร้างบทความใหม่ (ต้อง login)
    """
    queryset = Post.objects.filter(is_published=True)
    serializer_class = PostSerializer
    permission_classes = [permissions.IsAuthenticatedOrReadOnly]


class PostDetailAPIView(generics.RetrieveUpdateDestroyAPIView):
    """
    GET/PUT/PATCH/DELETE /api/posts/<pk>/
    """
    queryset = Post.objects.all()
    serializer_class = PostSerializer
    permission_classes = [permissions.IsAuthenticatedOrReadOnly]
```

เทียบกับ Part 042 ขั้นตอนที่ 418.3 ที่ `PostListAPIView`/`PostDetailAPIView` รวมกันยาว
กว่า 70 บรรทัด (นับ `get_object()`, `check_object_permissions()`, และทั้ง 6 handler
method) เวอร์ชันนี้เหลือเพียง **12 บรรทัด** และยังคงพฤติกรรมครบทุกอย่าง: list, create,
retrieve, update (PUT), partial update (PATCH), destroy — พร้อม status code, exception
handling, permission check ที่ถูกต้องตามที่เรียนมาทั้งหมดใน Part 042

**สังเกต**: ตอนนี้ยังใช้ `pk` เป็น lookup (ค่า default ของ `lookup_field`) ยังไม่ใช่
`slug` แบบ Part 042 — เราจะปรับเป็น `slug` ในขั้นตอนที่ 427 พร้อมอธิบายเหตุผลที่ต้อง
override เพิ่ม

### 424.4 อย่าลืมอัปเดต `urls.py`

```python
# blog/api_urls.py
from django.urls import path
from . import api_views

app_name = 'blog_api'

urlpatterns = [
    path('posts/', api_views.PostListAPIView.as_view(), name='post-list'),
    path('posts/<int:pk>/', api_views.PostDetailAPIView.as_view(), name='post-detail'),
]
```

### 424.5 กับดักคลาสสิก: ลืมว่า convenience class ก็คือ class ธรรมดา override ได้ทุกจุด

มือใหม่บางคนเข้าใจผิดว่า `ListCreateAPIView` เป็น "กล่องดำ" ที่แก้ไขไม่ได้ ความจริงคือ
มันเป็นแค่ Python class ธรรมดาที่ override เมธอดใดก็ได้เหมือน class อื่นทุกประการ —
เหมือนที่ Part 024 สอนว่า `ListView`/`CreateView` ของ Django ก็ override
`get_context_data()`, `get_queryset()`, `form_valid()` ได้ทุกจุด:

```python
class PostListAPIView(generics.ListCreateAPIView):
    queryset = Post.objects.filter(is_published=True)
    serializer_class = PostSerializer

    def list(self, request, *args, **kwargs):
        # override .list() ทั้งเมธอดได้ ถ้า ListModelMixin.list() ไม่ตรงความต้องการ
        response = super().list(request, *args, **kwargs)
        response.data = {'results': response.data, 'note': 'custom wrapper'}
        return response
```

Override แบบนี้ควรทำเมื่อจำเป็นจริง ๆ เท่านั้น — ขั้นตอนที่ 425-428 จะแนะนำจุด override
ที่ "ตรงประเด็นกว่า" (เช่น `get_queryset()`, `perform_create()`) ที่ **ไม่ต้องแตะ**
`.list()`/`.create()` เต็มเมธอดเลย

---

## ขั้นตอนที่ 425: Override `get_queryset()`/`get_serializer_class()` แบบ dynamic ตาม action

### 425.1 โจทย์: list แสดงน้อยฟิลด์ แต่ detail แสดงครบ

ในงานจริง endpoint แบบ "list" มักไม่ต้องการส่งทุกฟิลด์กลับไป (ประหยัด bandwidth,
ซ่อนข้อมูลที่ไม่จำเป็นสำหรับหน้ารายการ) ในขณะที่ endpoint "detail" ต้องการข้อมูลครบ
ตัวอย่าง: หน้ารายการบทความไม่จำเป็นต้องส่ง `content` เต็ม (อาจยาวหลายพันตัวอักษร) แต่
หน้ารายละเอียดต้องมี

```python
# blog/serializers.py
from rest_framework import serializers
from .models import Post


class PostListSerializer(serializers.ModelSerializer):
    """ใช้กับหน้ารายการ — ไม่มี content เต็ม มีแค่ preview สั้น ๆ"""
    author = serializers.ReadOnlyField(source='author.username')
    content_preview = serializers.SerializerMethodField()

    class Meta:
        model = Post
        fields = ['id', 'author', 'title', 'slug', 'content_preview', 'created_at']

    def get_content_preview(self, obj):
        return obj.content[:120] + ('...' if len(obj.content) > 120 else '')


class PostDetailSerializer(serializers.ModelSerializer):
    """ใช้กับหน้ารายละเอียด — มี content เต็ม"""
    author = serializers.ReadOnlyField(source='author.username')

    class Meta:
        model = Post
        fields = [
            'id', 'author', 'title', 'slug',
            'content', 'is_published', 'created_at', 'updated_at',
        ]
        read_only_fields = ['id', 'slug', 'created_at', 'updated_at']
```

### 425.2 Override `get_serializer_class()`

```python
# blog/api_views.py
from rest_framework import generics, permissions
from .models import Post
from .serializers import PostListSerializer, PostDetailSerializer


class PostListAPIView(generics.ListCreateAPIView):
    queryset = Post.objects.filter(is_published=True)
    permission_classes = [permissions.IsAuthenticatedOrReadOnly]

    def get_serializer_class(self):
        if self.request.method == 'POST':
            # ตอนสร้างใหม่ ต้องการฟิลด์ครบ (content, is_published) ไม่ใช่แค่ preview
            return PostDetailSerializer
        return PostListSerializer
```

ทวนจากขั้นตอนที่ 421.4: `ListModelMixin.list()` และ `CreateModelMixin.create()`
**ไม่เคยเรียก `PostSerializer` ตรง ๆ เลย** ทั้งคู่เรียกผ่าน `self.get_serializer(...)`
ซึ่งเรียก `self.get_serializer_class()` ก่อนเสมอ — นี่คือเหตุผลที่การ override
`get_serializer_class()` เพียงเมธอดเดียวส่งผลไปถึงทั้ง `.list()` และ `.create()` โดย
**ไม่ต้องแตะ Mixin หรือ `.list()`/`.create()` เลยแม้แต่บรรทัดเดียว** — นี่คือพลังที่
แท้จริงของสถาปัตยกรรมแบบ "layer ของเมธอดเล็ก ๆ ที่เรียกกันเป็นทอด ๆ" ที่ Part 024
ขั้นตอนที่ 236.2 เรียกว่า "cooperative extension"

### 425.3 Override `get_queryset()` แบบ dynamic ตาม user ที่ล็อกอิน

โจทย์: ผู้ใช้ทั่วไปเห็นเฉพาะบทความที่เผยแพร่แล้ว แต่เจ้าของบทความควรเห็นบทความฉบับร่าง
ของตัวเองด้วยตอนดูหน้ารายการ (เพื่อจัดการบทความที่ยังไม่เผยแพร่)

```python
# blog/api_views.py
from django.db.models import Q
from rest_framework import generics, permissions


class PostListAPIView(generics.ListCreateAPIView):
    permission_classes = [permissions.IsAuthenticatedOrReadOnly]

    def get_queryset(self):
        user = self.request.user
        if user.is_authenticated:
            # เห็นบทความที่เผยแพร่แล้วทั้งหมด + บทความฉบับร่างของตัวเอง
            return Post.objects.filter(
                Q(is_published=True) | Q(author=user)
            ).distinct().select_related('author')
        # ผู้ใช้ที่ไม่ล็อกอินเห็นเฉพาะบทความที่เผยแพร่แล้ว
        return Post.objects.filter(is_published=True).select_related('author')

    def get_serializer_class(self):
        if self.request.method == 'POST':
            return PostDetailSerializer
        return PostListSerializer
```

**ข้อควรระวังสำคัญ**: เมื่อ override `get_queryset()` แบบนี้ ห้ามตั้ง `queryset = ...`
เป็น class attribute ควบคู่กันไปด้วย เพราะจะทำให้สับสนว่าค่าไหนถูกใช้จริง (คำตอบคือ
`get_queryset()` override จะถูกใช้เสมอ เพราะ Python method resolution มองหาเมธอดที่
override ก่อน แต่ `queryset` attribute ที่ตั้งค้างไว้จะทำให้คนอ่านโค้ดคนอื่นสับสนว่า
"งั้น `queryset` attribute มีไว้ทำไม" — เขียนโค้ดให้ชัดเจนคือแนวทางที่ถูกต้องกว่า)

### 425.4 เทคนิคขั้นสูง: `get_serializer_class()` ตาม action ของ `ViewSet` (เกริ่นล่วงหน้า)

ในขั้นตอนที่ผ่านมาเราแยกตาม `self.request.method` (`'POST'` vs อื่น ๆ) ซึ่งใช้ได้กับ
Generic View ทุกตัว แต่เมื่อไปถึง **`ViewSet`** ใน Part 044 คุณจะมี `self.action`
(`'list'`, `'create'`, `'retrieve'`, `'update'`, `'partial_update'`, `'destroy'`) ที่
ละเอียดกว่าให้ตรวจสอบแทน:

```python
# ตัวอย่างล่วงหน้า (จะเจาะลึกจริงใน Part 044) — ใช้ self.action แทน self.request.method
def get_serializer_class(self):
    if self.action == 'list':
        return PostListSerializer
    return PostDetailSerializer
```

หลักการ **"override `get_serializer_class()` ให้คืนค่าต่างกันตามเงื่อนไข"** เหมือนกัน
ทุกประการ ต่างแค่ว่า Generic View ธรรมดามีแค่ `self.request.method` ให้ใช้ ในขณะที่
`ViewSet` มี `self.action` ที่สื่อความหมายชัดเจนกว่า

### 425.5 ตารางสรุป: จุด override ที่ dynamic ได้ทั้งหมดในขั้นตอนนี้

| เมธอดที่ override | ใช้ตัดสินใจจากอะไรได้ | ผลกระทบไปถึง |
|---|---|---|
| `get_serializer_class()` | `self.request.method`, `self.request.user`, `self.kwargs` | `.list()`, `.create()`, `.retrieve()`, `.update()` ทุกตัวที่เรียก `get_serializer()` |
| `get_queryset()` | `self.request.user`, query parameters, `self.kwargs` | `.list()` โดยตรง และ `.get_object()` ผ่าน `filter_queryset(self.get_queryset())` |
| `get_serializer_context()` | เพิ่ม key พิเศษเข้า context (นอกจาก `request`/`view`/`format`) | ทุก Serializer ที่สร้างผ่าน `get_serializer()` เข้าถึง context เพิ่มได้ |
| `filter_queryset()` | (ขั้นตอนที่ 428) เพิ่ม logic กรองข้อมูลนอกเหนือจาก `filter_backends` | `.list()` และ `.get_object()` |

---

## ขั้นตอนที่ 426: `perform_create()`, `perform_update()`, `perform_destroy()` hooks

### 426.1 ทบทวนจุดที่ Mixin เตรียม hook ไว้ให้ (ขั้นตอนที่ 422)

ย้อนกลับไปดูซอร์สโค้ดของ `CreateModelMixin.create()` ในขั้นตอนที่ 422.3 อีกครั้ง:

```python
def create(self, request, *args, **kwargs):
    serializer = self.get_serializer(data=request.data)
    serializer.is_valid(raise_exception=True)
    self.perform_create(serializer)      # <-- hook point
    headers = self.get_success_headers(serializer.data)
    return Response(serializer.data, status=status.HTTP_201_CREATED, headers=headers)

def perform_create(self, serializer):
    serializer.save()
```

`perform_create()` มีอยู่แยกจาก `create()` **โดยเจตนา** เพื่อให้ override ได้โดยไม่ต้อง
ยุ่งกับ validate/response/headers เลย — นี่คือคำตอบของปัญหาที่ Part 042 ขั้นตอนที่
418.3 แก้แบบ manual ด้วย `serializer.save(author=request.user)` ตรง ๆ ใน `post()`

### 426.2 Override `perform_create()`: ผูก `author` อัตโนมัติ

```python
# blog/api_views.py
from rest_framework import generics, permissions


class PostListAPIView(generics.ListCreateAPIView):
    queryset = Post.objects.filter(is_published=True)
    serializer_class = PostSerializer
    permission_classes = [permissions.IsAuthenticatedOrReadOnly]

    def perform_create(self, serializer):
        # เทียบเท่ากับ serializer.save(author=request.user) ของ Part 042
        # แต่แยกออกมาเป็นจุดเดียวที่ควบคุมทุกอย่างเกี่ยวกับ "การสร้าง" object
        serializer.save(author=self.request.user)
```

**ทำไมวิธีนี้ดีกว่าเขียน `serializer.save(author=request.user)` ตรงใน `post()` แบบ
Part 042?**

1. **แยกความรับผิดชอบชัดเจน**: `create()` (จาก Mixin) รับผิดชอบเรื่อง HTTP flow
   (validate, status code, headers) ส่วน `perform_create()` รับผิดชอบเรื่อง "จะบันทึก
   ข้อมูลลงฐานข้อมูลอย่างไร" — แยกกันทำให้แต่ละส่วนทดสอบและแก้ไขแยกกันได้ง่ายกว่า
2. **Reusable ข้าม HTTP method**: ถ้าในอนาคตมี endpoint อื่นที่ผสม `CreateModelMixin`
   เดียวกัน (เช่น bulk-create endpoint) การ override `perform_create()` เพียงจุดเดียว
   จะถูกเรียกใช้ทุกที่ที่ `create()` ถูกเรียก
3. **ทำงานหลายอย่างพร้อม `.save()` ได้ในที่เดียว**: เช่น ส่ง notification, log,
   invalidate cache — ทั้งหมดนี้ไม่ควรปนอยู่ใน logic การ validate/response

### 426.3 ตัวอย่างที่ซับซ้อนขึ้น: `perform_create()` ที่ทำมากกว่าหนึ่งอย่าง

```python
class PostListAPIView(generics.ListCreateAPIView):
    queryset = Post.objects.filter(is_published=True)
    serializer_class = PostSerializer
    permission_classes = [permissions.IsAuthenticatedOrReadOnly]

    def perform_create(self, serializer):
        post = serializer.save(author=self.request.user)
        # ทำงานเสริมหลังบันทึกสำเร็จ โดยไม่กระทบ response ที่ client ได้รับเลย
        logger.info("บทความใหม่ '%s' ถูกสร้างโดย %s", post.title, self.request.user)
        # ตัวอย่าง: ล้าง cache รายการบทความ (จะเจาะลึก caching เต็มรูปแบบใน Phase 8)
        cache.delete('post_list_cache_key')
```

### 426.4 Override `perform_update()`: บันทึก `updated_by` เพิ่มเติม

```python
class PostDetailAPIView(generics.RetrieveUpdateDestroyAPIView):
    queryset = Post.objects.all()
    serializer_class = PostSerializer
    permission_classes = [permissions.IsAuthenticatedOrReadOnly]
    lookup_field = 'slug'

    def perform_update(self, serializer):
        # สมมติเพิ่ม field updated_by ใน Model (ไม่ได้อยู่ใน serializer fields
        # เพราะไม่ต้องการให้ client กำหนดเอง — ตั้งค่าจาก request.user เท่านั้น)
        serializer.save(updated_by=self.request.user)
```

### 426.5 Override `perform_destroy()`: Soft Delete แทนการลบจริง

หนึ่งใน use case ที่พบบ่อยที่สุดของ `perform_destroy()` ในงานจริงคือการทำ **soft
delete** — ไม่ลบแถวออกจากฐานข้อมูลจริง แต่ตั้ง flag ว่า "ถูกลบแล้ว" เพื่อให้กู้คืนได้
ในภายหลังและรักษาความสัมพันธ์กับ record อื่น (เช่น comment ที่ผูกกับ post นั้น):

```python
class PostDetailAPIView(generics.RetrieveUpdateDestroyAPIView):
    queryset = Post.objects.all()
    serializer_class = PostSerializer
    permission_classes = [permissions.IsAuthenticatedOrReadOnly]
    lookup_field = 'slug'

    def perform_destroy(self, instance):
        # แทนที่จะเรียก instance.delete() ตรง ๆ (ค่า default ของ DestroyModelMixin)
        # เราตั้ง flag แทน — สังเกตว่า destroy() ของ Mixin ยังคืน 204 ให้เหมือนเดิม
        # โดยไม่ต้องแก้ destroy() เลยแม้แต่บรรทัดเดียว
        instance.is_published = False
        instance.is_deleted = True
        instance.save(update_fields=['is_published', 'is_deleted'])
```

**ข้อสังเกตสำคัญ**: `DestroyModelMixin.destroy()` เขียนไว้ว่า
`self.perform_destroy(instance)` แล้วคืน `Response(status=204)` ทันที **โดยไม่สนใจว่า
`perform_destroy()` จะทำอะไรข้างในจริง ๆ** — client ที่เรียก `DELETE` จะได้ `204 No
Content` เหมือนเดิมทุกประการ ไม่ว่าเบื้องหลังจะเป็นการลบจริงหรือ soft delete ก็ตาม
นี่คือตัวอย่างที่ชัดเจนของหลักการ **encapsulation**: รายละเอียดการ implement
เปลี่ยนแปลงได้อย่างอิสระ ตราบใดที่ "สัญญา" (contract) ที่ client เห็นยังเหมือนเดิม

### 426.6 ตารางสรุปทั้งสาม hook

| Hook method | ถูกเรียกจาก | ค่า default | ตัวอย่าง use case ในงานจริง |
|---|---|---|---|
| `perform_create(serializer)` | `CreateModelMixin.create()` | `serializer.save()` | ผูก `author=request.user`, ส่ง notification, invalidate cache |
| `perform_update(serializer)` | `UpdateModelMixin.update()` | `serializer.save()` | บันทึก `updated_by`, สร้าง audit log ของการแก้ไข |
| `perform_destroy(instance)` | `DestroyModelMixin.destroy()` | `instance.delete()` | Soft delete, ลบไฟล์ที่เกี่ยวข้องออกจาก storage ก่อนลบ record |

**กฎเหล็กของหลักสูตรนี้**: เมื่อไหร่ก็ตามที่ต้องการทำอะไรเพิ่มเติม "รอบ ๆ" การ
save/delete (ไม่ใช่เปลี่ยน logic การ validate หรือ response) ให้ override เมธอด
`perform_*` เสมอ **อย่า override `.create()`/`.update()`/`.destroy()` ทั้งเมธอง**
เพราะจะทำให้เสียประโยชน์จากโค้ดมาตรฐานที่ DRF เตรียมไว้ให้ (headers, status code,
`_prefetched_objects_cache` handling ฯลฯ) โดยไม่จำเป็น

---

## ขั้นตอนที่ 427: `lookup_field`/`lookup_url_kwarg` customization — ใช้ slug แทน pk

### 427.1 ทบทวน `get_object()` จากขั้นตอนที่ 421.5

```python
# แนวคิดจากซอร์สโค้ดจริงของ rest_framework.generics.GenericAPIView
def get_object(self):
    queryset = self.filter_queryset(self.get_queryset())
    lookup_url_kwarg = self.lookup_url_kwarg or self.lookup_field
    filter_kwargs = {self.lookup_field: self.kwargs[lookup_url_kwarg]}
    obj = get_object_or_404(queryset, **filter_kwargs)
    self.check_object_permissions(self.request, obj)
    return obj
```

ค่า default: `lookup_field = 'pk'` และ `lookup_url_kwarg = None` (แปลว่า "ใช้ค่าเดียว
กับ `lookup_field`") ซึ่งหมายความว่าค่า default ของ DRF คาดหวัง URL pattern ที่มี
`<int:pk>` และค้นหาด้วย `Post.objects.get(pk=...)` — แต่ระบบ `Post` ของเราตั้งแต่
Part 042 ใช้ **`slug`** เป็นตัวระบุใน URL มาตลอด (`/api/posts/<slug:slug>/`) ไม่ใช่
`pk`

### 427.2 กรณีที่ 1: ชื่อ field ตรงกับชื่อ URL kwarg — ตั้ง `lookup_field` อย่างเดียวพอ

ถ้า URL pattern ตั้งชื่อ kwarg ว่า `slug` และ Model field ก็ชื่อ `slug` เหมือนกัน
(กรณีของเรา) ตั้งแค่ `lookup_field` ก็เพียงพอ:

```python
# blog/api_views.py
class PostDetailAPIView(generics.RetrieveUpdateDestroyAPIView):
    queryset = Post.objects.all()
    serializer_class = PostSerializer
    permission_classes = [permissions.IsAuthenticatedOrReadOnly]
    lookup_field = 'slug'   # ค้นหาด้วย Post.objects.get(slug=...) แทน pk
```

```python
# blog/api_urls.py
urlpatterns = [
    path('posts/', api_views.PostListAPIView.as_view(), name='post-list'),
    path('posts/<slug:slug>/', api_views.PostDetailAPIView.as_view(), name='post-detail'),
]
```

เมื่อ `lookup_field = 'slug'` แล้ว `get_object()` จะคำนวณ:

```python
lookup_url_kwarg = self.lookup_url_kwarg or self.lookup_field  # None or 'slug' -> 'slug'
filter_kwargs = {self.lookup_field: self.kwargs[lookup_url_kwarg]}
# filter_kwargs = {'slug': self.kwargs['slug']}
```

ตรงกับพฤติกรรมของ `get_object_or_404(Post, slug=slug)` ที่เขียนเองใน Part 042
ขั้นตอนที่ 418.3 ทุกประการ

### 427.3 กรณีที่ 2: ชื่อ URL kwarg ต่างจากชื่อ field — ต้องเพิ่ม `lookup_url_kwarg`

สมมติทีม frontend ขอให้ URL parameter ชื่อ `post_slug` แทน `slug` เฉย ๆ (เพื่อความ
ชัดเจนเมื่อมี URL ซ้อนกันหลายระดับ เช่น `/api/authors/<username>/posts/<post_slug>/`)
แต่ Model field ยังคงชื่อ `slug` เหมือนเดิม (ไม่อยากเปลี่ยนชื่อ field เพราะกระทบ
migration และโค้ดส่วนอื่นทั้งระบบ):

```python
# blog/api_views.py
class PostDetailAPIView(generics.RetrieveUpdateDestroyAPIView):
    queryset = Post.objects.all()
    serializer_class = PostSerializer
    lookup_field = 'slug'           # ชื่อ field ใน Model
    lookup_url_kwarg = 'post_slug'  # ชื่อ kwarg ใน URL pattern (ต่างจาก lookup_field)
```

```python
# blog/api_urls.py
urlpatterns = [
    path('posts/<slug:post_slug>/', api_views.PostDetailAPIView.as_view(), name='post-detail'),
]
```

ตอนนี้ `get_object()` จะคำนวณ:

```python
lookup_url_kwarg = self.lookup_url_kwarg or self.lookup_field  # 'post_slug' or 'slug' -> 'post_slug'
filter_kwargs = {self.lookup_field: self.kwargs[lookup_url_kwarg]}
# filter_kwargs = {'slug': self.kwargs['post_slug']}
```

สังเกตว่า **`self.lookup_field` (`'slug'`) ยังคงถูกใช้เป็นชื่อ field ตอน query
ฐานข้อมูล** ในขณะที่ `self.kwargs[lookup_url_kwarg]` (`self.kwargs['post_slug']`) คือ
ค่าที่ดึงมาจาก URL — สอง attribute นี้ทำหน้าที่ต่างกันโดยสิ้นเชิง: `lookup_field` บอกว่า
"query ด้วย field ไหน" ส่วน `lookup_url_kwarg` บอกว่า "อ่านค่ามาจาก URL kwarg ชื่ออะไร"

### 427.4 กับดักคลาสสิก: ลืม `lookup_field` แต่ URL ใช้ slug

```python
# ผิด! ลืมตั้ง lookup_field = 'slug'
class PostDetailAPIView(generics.RetrieveUpdateDestroyAPIView):
    queryset = Post.objects.all()
    serializer_class = PostSerializer
    # ไม่ได้ตั้ง lookup_field เลย -> ใช้ค่า default 'pk'
```

```python
# blog/api_urls.py
path('posts/<slug:slug>/', api_views.PostDetailAPIView.as_view(), name='post-detail'),
```

ผลลัพธ์ที่เกิดขึ้น:

```python
>>> import requests
>>> requests.get('http://127.0.0.1:8000/api/posts/my-first-post/').json()
{'detail': "Expected view PostDetailAPIView to be called with a URL keyword argument "
           "named \"pk\". Fix your URL conf, or set the `.lookup_field` attribute on "
           "the view correctly."}
```

Error message นี้มาจาก `assert` ใน `get_object()` (ขั้นตอนที่ 421.5) ที่เช็คว่า
`lookup_url_kwarg in self.kwargs` — เพราะ URL ส่งค่ามาเป็น `self.kwargs['slug']` แต่
`get_object()` (ด้วยค่า default) มองหา `self.kwargs['pk']` ซึ่งไม่มีอยู่จริง DRF จง
ใจเขียน error message ให้ชัดเจนขนาดนี้เพื่อให้ debug ได้เร็ว — นี่คือตัวอย่างที่ดีของ
"fail loudly" แทนที่จะปล่อยให้เกิด error ที่คลุมเครือกว่านี้

### 427.5 ตารางสรุปความสัมพันธ์ `lookup_field` / `lookup_url_kwarg` / URL pattern

| สถานการณ์ | `lookup_field` | `lookup_url_kwarg` | URL pattern ตัวอย่าง |
|---|---|---|---|
| ค่า default (`pk`) | `'pk'` (ไม่ต้องตั้ง) | `None` (ไม่ต้องตั้ง) | `path('posts/<int:pk>/', ...)` |
| ใช้ slug, ชื่อ kwarg ตรงกับ field | `'slug'` | `None` (ไม่ต้องตั้ง) | `path('posts/<slug:slug>/', ...)` |
| ใช้ slug, ชื่อ kwarg ต่างจาก field | `'slug'` | `'post_slug'` | `path('posts/<slug:post_slug>/', ...)` |
| ใช้ field ประกอบ (uuid) | `'uuid'` | `None` | `path('posts/<uuid:uuid>/', ...)` |
| Nested resource (ดึงจาก parent URL) | `'slug'` | `'post_slug'` | `path('authors/<username>/posts/<post_slug>/', ...)` |

---

## ขั้นตอนที่ 428: ผสาน Generic View กับ custom filtering logic ของตัวเอง

### 428.1 ทบทวน `filter_queryset()` จากขั้นตอนที่ 421

```python
# แนวคิดจากซอร์สโค้ดจริงของ rest_framework.generics.GenericAPIView
class GenericAPIView(views.APIView):
    filter_backends = api_settings.DEFAULT_FILTER_BACKENDS

    def filter_queryset(self, queryset):
        for backend in list(self.filter_backends):
            queryset = backend().filter_queryset(self.request, queryset, self)
        return queryset
```

`filter_queryset()` ถูกเรียกใน **สองที่** เสมอ: `ListModelMixin.list()` (ขั้นตอนที่
422.2) และ `GenericAPIView.get_object()` (ขั้นตอนที่ 421.5) — หมายความว่า logic
filtering ที่เขียนผ่านกลไกนี้จะมีผลทั้งกับหน้ารายการ **และ** ป้องกันไม่ให้เข้าถึง
object เดี่ยวที่ไม่ผ่านเงื่อนไขด้วย (เช่น ถ้า filter กรองเฉพาะบทความที่เผยแพร่แล้ว
การเปิด URL detail ของบทความที่ยังไม่เผยแพร่ก็จะได้ 404 ไปด้วยโดยอัตโนมัติ)

**หมายเหตุ**: การติดตั้งและใช้ `django-filter` (`DjangoFilterBackend`) แบบเต็มรูปแบบ
พร้อม `SearchFilter`, `OrderingFilter` จะเรียนละเอียดใน **Part 047: Filtering,
Searching และ Ordering** — ขั้นตอนนี้จะเน้นการเขียน filtering logic **เองแบบง่าย
ผ่าน `get_queryset()`** ก่อน เพื่อให้เข้าใจกลไกพื้นฐานที่ `django-filter` จะมาต่อยอด

### 428.2 กรองด้วย query parameter อย่างง่ายผ่าน `get_queryset()`

```python
# blog/api_views.py
class PostListAPIView(generics.ListCreateAPIView):
    serializer_class = PostSerializer
    permission_classes = [permissions.IsAuthenticatedOrReadOnly]

    def get_queryset(self):
        queryset = Post.objects.filter(is_published=True).select_related('author')

        # ?search=django -> ค้นหาคำในหัวข้อหรือเนื้อหา
        search = self.request.query_params.get('search')
        if search:
            queryset = queryset.filter(
                Q(title__icontains=search) | Q(content__icontains=search)
            )

        # ?author=demo -> กรองตาม username ของผู้เขียน
        author_username = self.request.query_params.get('author')
        if author_username:
            queryset = queryset.filter(author__username=author_username)

        # ?ordering=-created_at หรือ ?ordering=title -> เรียงลำดับผลลัพธ์
        ordering = self.request.query_params.get('ordering', '-created_at')
        allowed_orderings = {'created_at', '-created_at', 'title', '-title'}
        if ordering in allowed_orderings:
            queryset = queryset.order_by(ordering)

        return queryset
```

ทดสอบ:

```bash
http GET '127.0.0.1:8000/api/posts/?search=django'
http GET '127.0.0.1:8000/api/posts/?author=demo&ordering=title'
```

**ข้อควรระวังสำคัญ**: `allowed_orderings` เป็น whitelist ที่จำเป็นมาก — ถ้าปล่อยให้
client ส่งชื่อ field อะไรก็ได้ไปที่ `.order_by()` ตรง ๆ (เช่น
`queryset.order_by(self.request.query_params.get('ordering'))`) จะเสี่ยงต่อการที่
client ระบุ field ที่ไม่มีอยู่จริง (เกิด `FieldError`) หรือแย่กว่านั้นคือระบุ field ที่
sensitive ที่ไม่ควร expose (เช่น `order_by('author__password')` ถ้า field นั้นอยู่ใน
Model ที่ join ถึงได้ — แม้ Django จะ query ผิดพลาดเพราะ password เป็น hash ไม่ได้ leak
ค่าจริง แต่หลักการ "อย่าไว้ใจ input จาก client โดยไม่ตรวจสอบ" ยังคงสำคัญเสมอ)

### 428.3 ผสาน custom filtering เข้ากับ `filter_queryset()` แทน `get_queryset()`

อีกแนวทางหนึ่งที่ "ตรงจุด" กว่าคือ override `filter_queryset()` แทน `get_queryset()`
เพื่อแยกความรับผิดชอบให้ชัดเจนกว่าเดิม (`get_queryset()` ตอบคำถาม "ข้อมูลชุดฐานคือ
อะไร" ส่วน `filter_queryset()` ตอบคำถาม "จะกรองข้อมูลชุดฐานนั้นเพิ่มเติมอย่างไรตาม
request"):

```python
class PostListAPIView(generics.ListCreateAPIView):
    serializer_class = PostSerializer
    permission_classes = [permissions.IsAuthenticatedOrReadOnly]
    queryset = Post.objects.filter(is_published=True).select_related('author')

    def filter_queryset(self, queryset):
        # เรียก filter_backends ที่ตั้งค่าไว้ (ถ้ามี) ก่อนเสมอ ตามพฤติกรรม default
        queryset = super().filter_queryset(queryset)

        search = self.request.query_params.get('search')
        if search:
            queryset = queryset.filter(
                Q(title__icontains=search) | Q(content__icontains=search)
            )
        return queryset
```

ข้อดีของแนวทางนี้: เพราะ `get_object()` เรียก `self.filter_queryset(self.get_queryset())`
ด้วยเช่นกัน (ขั้นตอนที่ 421.5) การกรองที่ทำผ่าน `filter_queryset()` จะมีผลถึง endpoint
detail ด้วย — ตัวอย่างเช่น ถ้า filtering เดียวกันนี้ใช้ตรวจสอบว่า post ต้องอยู่ใน
`category` ที่ user มีสิทธิ์เข้าถึง การเปิด URL detail ของ post ที่ไม่ผ่านเงื่อนไขก็จะ
ได้ 404 automatically เช่นกัน โดยไม่ต้องเขียนโค้ดตรวจสอบซ้ำใน `get_object()`

### 428.4 ตารางสรุป: เลือก override จุดไหนสำหรับ custom logic

| ต้องการทำอะไร | Override เมธอดไหน | มีผลกับ `.list()` | มีผลกับ `.retrieve()`/`.update()`/`.destroy()` |
|---|---|---|---|
| กำหนด "ข้อมูลชุดฐาน" ที่ endpoint นี้เข้าถึงได้ | `get_queryset()` | ✅ | ✅ (ผ่าน `get_object()`) |
| กรองเพิ่มเติมตาม query parameter (search/filter) | `filter_queryset()` | ✅ | ✅ (ผ่าน `get_object()`) |
| เลือก Serializer class ต่างกันตามเงื่อนไข | `get_serializer_class()` | ✅ | ✅ |
| ทำงานเพิ่มเติมตอน save/delete | `perform_create()`/`perform_update()`/`perform_destroy()` | ❌ (ไม่เกี่ยว) | ✅ |
| เปลี่ยน field ที่ใช้ค้นหา object เดี่ยว | `lookup_field`/`lookup_url_kwarg` | ❌ (ไม่เกี่ยว) | ✅ |

---

## ขั้นตอนที่ 429: แปลง CRUD จาก `APIView` ธรรมดา (Part 042) ให้เป็น Generic Views เต็มรูปแบบ

### 429.1 เวอร์ชัน "ก่อน" — `APIView` ล้วน ๆ จาก Part 042 (เต็มรูปแบบ)

```python
# blog/api_views.py — เวอร์ชัน Part 042 (APIView ธรรมดา) — 72 บรรทัด
from django.shortcuts import get_object_or_404
from rest_framework import permissions, status
from rest_framework.exceptions import PermissionDenied
from rest_framework.response import Response
from rest_framework.views import APIView

from .models import Post
from .serializers import PostSerializer


class PostListAPIView(APIView):
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
    permission_classes = [permissions.IsAuthenticatedOrReadOnly]

    def get_object(self, slug):
        post = get_object_or_404(Post, slug=slug)
        self.check_object_permissions(self.request, post)
        return post

    def check_object_permissions(self, request, post):
        if request.method not in permissions.SAFE_METHODS:
            if post.author != request.user:
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

### 429.2 เวอร์ชัน "หลัง" — Generic Views เต็มรูปแบบ (Part นี้)

```python
# blog/api_views.py — เวอร์ชัน Part 043 (Generic Views) — 26 บรรทัด
from rest_framework import generics, permissions

from .models import Post
from .serializers import PostSerializer


class PostListAPIView(generics.ListCreateAPIView):
    """
    GET  /api/posts/  -> รายการบทความที่เผยแพร่แล้ว
    POST /api/posts/  -> สร้างบทความใหม่ (ต้อง login)
    """
    serializer_class = PostSerializer
    permission_classes = [permissions.IsAuthenticatedOrReadOnly]

    def get_queryset(self):
        return Post.objects.filter(is_published=True).select_related('author')

    def perform_create(self, serializer):
        serializer.save(author=self.request.user)


class PostDetailAPIView(generics.RetrieveUpdateDestroyAPIView):
    """
    GET/PUT/PATCH/DELETE /api/posts/<slug>/
    """
    queryset = Post.objects.select_related('author')
    serializer_class = PostSerializer
    permission_classes = [permissions.IsAuthenticatedOrReadOnly]
    lookup_field = 'slug'
```

**26 บรรทัด เทียบกับ 72 บรรทัดเดิม (ลดลงกว่า 60%)** และยัง**ไม่ได้ตัดฟีเจอร์ใดออกเลย**
— ทุก endpoint, ทุก status code, ทุก validation ยังทำงานครบเหมือนเดิม (ดูตารางเทียบ
พฤติกรรมในขั้นตอนที่ 429.3)

**หมายเหตุเรื่อง object-level permission**: เวอร์ชัน Part 042 เขียน
`check_object_permissions()` เองแบบตรวจสอบ `post.author != request.user` สำหรับ
method ที่ไม่ใช่ `SAFE_METHODS` ส่วนเวอร์ชัน Part 043 นี้ยังคง**ไม่มี** logic
ตรวจสอบความเป็นเจ้าของแบบละเอียดใน `permission_classes` เพราะ `IsAuthenticatedOrReadOnly`
ของ DRF เช็คแค่ "ล็อกอินหรือยัง" ไม่เช็คความเป็นเจ้าของ — **Part 045 (DRF Permissions)**
จะแนะนำการเขียน custom `permissions.BasePermission` ที่ override
`has_object_permission()` เพื่อทดแทน `check_object_permissions()` แบบ manual ให้ถูก
หลักการของ DRF อย่างสมบูรณ์ ตอนนี้ขอให้เข้าใจโครงสร้าง Generic View ให้แน่นก่อน

### 429.3 ตารางเทียบพฤติกรรม: ยืนยันว่าไม่มีอะไรหายไป

| พฤติกรรม | Part 042 (`APIView`) | Part 043 (Generic Views) | เหมือนกันหรือไม่ |
|---|---|---|---|
| `GET /api/posts/` คืนบทความที่เผยแพร่แล้ว | `Post.objects.filter(is_published=True)` ใน `get()` | `get_queryset()` เดียวกัน | ✅ เหมือนกัน |
| `POST /api/posts/` สร้างบทความ + ผูก author | `serializer.save(author=request.user)` ใน `post()` | `perform_create()` | ✅ เหมือนกัน |
| Status code ตอนสร้างสำเร็จ | `201 Created` (manual) | `201 Created` (จาก `CreateModelMixin`) | ✅ เหมือนกัน |
| Header `Location` ตอนสร้างสำเร็จ | ตั้งเอง `headers={'Location': ...}` | อัตโนมัติจาก `get_success_headers()` (ถ้า serializer มี URL field) | ⚠️ ต้องตรวจสอบ (ดูหมายเหตุด้านล่าง) |
| `GET /api/posts/<slug>/` คืนรายละเอียด | `get_object_or_404(Post, slug=slug)` ใน `get()` | `get_object()` (ผ่าน `lookup_field = 'slug'`) | ✅ เหมือนกัน |
| `PUT` แก้ไขทั้งหมด | `serializer.save()` แบบไม่ `partial` | `UpdateModelMixin.update()` (`partial=False`) | ✅ เหมือนกัน |
| `PATCH` แก้ไขบางส่วน | `serializer.save()` แบบ `partial=True` | `UpdateModelMixin.partial_update()` | ✅ เหมือนกัน |
| `DELETE` ลบและคืน 204 | `post.delete()` + `Response(status=204)` | `DestroyModelMixin.destroy()` | ✅ เหมือนกัน |
| 404 เมื่อไม่พบบทความ | `get_object_or_404()` raise `Http404` | `get_object()` ใช้ `get_object_or_404()` ภายในเช่นกัน | ✅ เหมือนกัน |
| Validation error คืน 400 | `serializer.is_valid(raise_exception=True)` | เหมือนกันทุกประการใน Mixin | ✅ เหมือนกัน |

**หมายเหตุเรื่อง `Location` header**: `get_success_headers()` (ขั้นตอนที่ 422.3) พยายาม
อ่านค่าจาก `data[api_settings.URL_FIELD_NAME]` (ค่า default ของ `URL_FIELD_NAME` คือ
`'url'`) ซึ่ง `PostSerializer` ของเราไม่มี field ชื่อ `url` จึงเกิด `KeyError` ที่ถูก
ดักไว้แล้ว (`try/except (TypeError, KeyError): return {}`) ผลคือ **เวอร์ชัน Generic
View นี้จะไม่ส่ง header `Location` กลับไป** ต่างจากเวอร์ชัน Part 042 ที่ตั้งเองแบบ
manual — ถ้าต้องการ header นี้กลับคืนมา ให้ override `get_success_headers()` เอง หรือ
เพิ่ม `HyperlinkedIdentityField(view_name='blog_api:post-detail', lookup_field='slug')`
ชื่อ `url` เข้าไปใน serializer (เทคนิคของ `HyperlinkedModelSerializer` จะเรียนใน
Part 044)

```python
class PostListAPIView(generics.ListCreateAPIView):
    ...
    def get_success_headers(self, data):
        try:
            return {'Location': f"/api/posts/{data['slug']}/"}
        except (TypeError, KeyError):
            return {}
```

### 429.4 `blog/api_urls.py` (ไม่เปลี่ยนแปลงจาก Part 042)

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

จุดที่น่าสนใจ: **`urls.py` ไม่ต้องแก้ไขอะไรเลยแม้แต่บรรทัดเดียว** เพราะ Generic Views
ยังคงคืน callable จาก `.as_view()` เหมือน `APIView` ทุกประการ (สืบทอดมาจาก Django
`View.as_view()` ตั้งแต่ Part 021) — นี่คือเหตุผลที่การเปลี่ยนจาก `APIView` ไปเป็น
Generic Views (หรือย้อนกลับ) เป็นการเปลี่ยนแปลง**ภายใน**ของ view เท่านั้น ไม่กระทบ
"สัญญา" ภายนอกที่ URL config หรือ client เห็นเลย

### 429.5 ทดสอบว่าเวอร์ชันใหม่ทำงานเหมือนเดิมทุกประการ

```python
>>> from django.test import RequestFactory
>>> from django.contrib.auth import get_user_model
>>> from blog.api_views import PostListAPIView, PostDetailAPIView

>>> User = get_user_model()
>>> user = User.objects.get(username='demo')
>>> factory = RequestFactory()

>>> # สร้างบทความใหม่ — ต้องได้ผลลัพธ์เหมือนขั้นตอนที่ 418.6 ของ Part 042 ทุกประการ
>>> request = factory.post(
...     '/api/posts/',
...     data='{"title": "ทดสอบ Generic View", "content": "เนื้อหาทดสอบ"}',
...     content_type='application/json',
... )
>>> request.user = user
>>> response = PostListAPIView.as_view()(request)
>>> response.status_code
201
>>> response.data['author']
'demo'
>>> response.data['slug']
'ทดสอบ-generic-view'
```

### 429.6 ตารางสรุปจำนวนบรรทัดที่ลดลงในแต่ละส่วน

| ส่วนของโค้ด | Part 042 (`APIView`) | Part 043 (Generic Views) | ลดลง |
|---|---|---|---|
| `PostListAPIView` (GET+POST) | 17 บรรทัด | 11 บรรทัด | ~35% |
| `PostDetailAPIView` (GET/PUT/PATCH/DELETE + `get_object`) | 30 บรรทัด | 7 บรรทัด | ~77% |
| Import statements | 6 บรรทัด | 3 บรรทัด | 50% |
| **รวมทั้งไฟล์** | **~72 บรรทัด** | **~26 บรรทัด** | **~64%** |

---

## ขั้นตอนที่ 430: สรุปและแบบฝึกหัด

### 430.1 สรุปสิ่งที่ได้เรียนรู้ใน Part นี้

- ✅ เข้าใจว่า `generics.GenericAPIView` สืบทอดจาก `APIView` (Part 042) โดยตรง และ
  เพิ่มความสามารถเรื่อง `queryset`/`serializer_class` ผ่าน `get_queryset()`,
  `get_serializer()`, `get_serializer_class()`, `get_object()`
- ✅ เจาะลึก 5 Mixin ของ `rest_framework.mixins` (`ListModelMixin`, `CreateModelMixin`,
  `RetrieveModelMixin`, `UpdateModelMixin`, `DestroyModelMixin`) และเห็นว่าแต่ละตัว
  พึ่งพาเมธอดจาก `GenericAPIView` เหมือนที่ Mixin ของ Django CBV ใน Part 024 พึ่งพา
  `SingleObjectMixin`
- ✅ ประกอบ Mixin + `GenericAPIView` ด้วยมือ และพิสูจน์ว่า pattern เดียวกันนี้คือสิ่งที่
  DRF ใช้สร้าง `ListCreateAPIView` เองภายใน (MRO เรียงแบบเดียวกับที่เรียนใน Part 024)
- ✅ รู้จัก convenience class ทั้ง 9 ตัวของ DRF และเลือกใช้ตัวที่ตรงกับ endpoint ได้
  (`ListCreateAPIView` สำหรับ collection, `RetrieveUpdateDestroyAPIView` สำหรับ detail)
- ✅ Override `get_queryset()`/`get_serializer_class()` แบบ dynamic ตาม
  `request.method`, `request.user` เพื่อแยก serializer สำหรับ list/create/detail
- ✅ ใช้ `perform_create()`, `perform_update()`, `perform_destroy()` เป็นจุดผูก
  `author=request.user`, audit log, และทำ soft delete โดยไม่แตะ flow หลักของ Mixin
- ✅ ปรับ `lookup_field`/`lookup_url_kwarg` ให้ค้นหาด้วย `slug` แทน `pk` และเข้าใจว่า
  ทั้งสอง attribute ทำหน้าที่ต่างกันอย่างไร
- ✅ เขียน custom filtering logic ผ่าน `get_queryset()`/`filter_queryset()` ที่มีผลถึง
  ทั้ง `.list()` และ `.get_object()` โดยอัตโนมัติ
- ✅ แปลง CRUD API เต็มรูปแบบของ `Post` จาก `APIView` (Part 042, ~72 บรรทัด) ให้เป็น
  Generic Views (~26 บรรทัด) โดยพฤติกรรมทุกอย่างเหมือนเดิมทุกประการ

### 430.2 Checklist ก่อนไป Part ถัดไป

- [ ] อธิบายได้ว่า `GenericAPIView` เพิ่มอะไรจาก `APIView` และทำไม `dispatch()`/
  `initial()`/`handle_exception()` ยังทำงานเหมือนเดิมทุกประการ
- [ ] เขียน `get_queryset()`, `get_serializer()`, `get_object()` เองได้โดยไม่ต้องเปิด
  เอกสารดู
- [ ] อธิบายความแตกต่างระหว่าง Mixin ทั้ง 5 ตัวได้ว่าแต่ละตัวให้เมธอดอะไร และพึ่งพา
  อะไรจาก `GenericAPIView`
- [ ] ประกอบ Mixin + `GenericAPIView` ด้วยมือเองได้ และอธิบาย MRO ที่ได้
- [ ] เลือก convenience class ที่ถูกต้อง (จากทั้ง 9 ตัว) ให้ตรงกับ endpoint ที่ต้องการ
  ได้ทันทีโดยไม่ต้องเปิดตารางดู
- [ ] Override `get_serializer_class()` แบบ dynamic ตาม HTTP method ได้
- [ ] อธิบายความแตกต่างระหว่าง `create()`/`update()`/`destroy()` กับ
  `perform_create()`/`perform_update()`/`perform_destroy()` ได้ชัดเจน พร้อมยกตัวอย่าง
  ว่าเมื่อไหร่ควร override ตัวไหน
- [ ] ตั้งค่า `lookup_field`/`lookup_url_kwarg` ให้ endpoint ใช้ slug แทน pk ได้
- [ ] เขียน custom filtering ผ่าน `get_queryset()` พร้อม whitelist ป้องกัน field
  ที่ไม่ได้รับอนุญาตได้
- [ ] แปลง `APIView` ที่เขียนแบบ manual ให้เป็น Generic Views ได้เองครบทุก endpoint

### 430.3 แบบฝึกหัดท้ายบท

**แบบฝึกหัดที่ 1 (พื้นฐาน)**: เขียน `CategoryListCreateAPIView` และ
`CategoryDetailAPIView` สำหรับ Model `Category` (สมมติมี field `name`, `slug`) โดยใช้
`generics.ListCreateAPIView` และ `generics.RetrieveUpdateDestroyAPIView` ตามลำดับ
ตั้ง `lookup_field = 'slug'` ให้ถูกต้อง แล้วทดสอบทั้ง 6 การกระทำ (list, create,
retrieve, update, partial update, destroy) ด้วย `httpie`

**แบบฝึกหัดที่ 2 (ประยุกต์)**: เพิ่ม `get_serializer_class()` ให้ `PostDetailAPIView`
คืน `PostSerializer` ธรรมดาเมื่อ method เป็น `GET` แต่คืน serializer อีกตัวชื่อ
`PostUpdateSerializer` (ที่ไม่อนุญาตให้แก้ไข field `slug` แม้จะส่งมาก็ตาม โดยใส่
`slug` ไว้ใน `read_only_fields`) เมื่อ method เป็น `PUT`/`PATCH` แล้วทดสอบว่าการส่ง
`slug` ใหม่มาตอน `PATCH` ไม่มีผลใด ๆ ต่อค่าที่บันทึกจริง

**แบบฝึกหัดที่ 3 (Hooks)**: เพิ่ม field `view_count` (จำนวนเข้าชม) ให้ Model `Post`
แล้วเขียน `PostDetailAPIView.retrieve()` override เพื่อเพิ่มค่า `view_count` ขึ้น 1
ทุกครั้งที่มีการเรียก `GET` (ใช้ `F('view_count') + 1` จาก `django.db.models` เพื่อ
หลีกเลี่ยง race condition ตามที่เรียนใน Part 014) โดยยังคงเรียก
`super().retrieve(request, *args, **kwargs)` เพื่อไม่ต้องเขียน logic การ serialize เอง

**แบบฝึกหัดที่ 4 (ขั้นสูง)**: สร้าง custom Mixin ของตัวเองชื่อ `SoftDeleteMixin` ที่
override `perform_destroy()` ให้ทำ soft delete เสมอ (ตาม pattern ในขั้นตอนที่ 426.5)
แล้วนำไปผสมกับทั้ง `PostDetailAPIView` และ `CategoryDetailAPIView` (พิสูจน์ว่า Mixin
ที่เขียนเองสำหรับ Generic API View ก็ reuse ข้าม resource ได้เหมือนกับ
`AuthorRequiredMixin` ใน Part 024 ที่ reuse ข้าม Model) เขียน `Manager` ใหม่ให้
`Post.objects` กรอง `is_deleted=False` ออกจาก queryset ปกติโดยอัตโนมัติด้วย

### 430.4 คำถามที่พบบ่อย (FAQ)

**Q: ควรใช้ `APIView` (Part 042) หรือ Generic Views (Part นี้) เป็นค่าเริ่มต้นของทีม?**
A: สำหรับ endpoint ที่ทำ CRUD มาตรฐานตรงกับ pattern list/detail (ซึ่งเป็นส่วนใหญ่ของ
API ในโปรเจกต์จริง) **ควรใช้ Generic Views เป็นค่าเริ่มต้นเสมอ** เพราะโค้ดสั้นกว่ามาก
สม่ำเสมอกว่า และลดโอกาสเกิด bug จากการเขียน validate/response ซ้ำเอง เก็บ `APIView`
ไว้สำหรับ endpoint ที่ logic ไม่ตรง pattern CRUD เลย (เช่น endpoint คำนวณสถิติซับซ้อน
ที่ไม่ได้ผูกกับ Model เดียว) หลักการเดียวกับที่ Part 024 ขั้นตอนที่ 239 แนะนำเรื่อง
FBV vs CBV ของ Django ธรรมดา

**Q: ทำไม `ListModelMixin.list()` เรียก `self.filter_queryset(self.get_queryset())`
สองครั้งแยกกัน (ใน `list()` และใน `get_object()`) แทนที่จะ cache ไว้?**
A: เพราะแต่ละ HTTP request สร้าง view instance ใหม่เสมอ (ทวนจาก Part 021/042 เรื่อง
`as_view()`) และ `list()`/`get_object()` ไม่เคยถูกเรียกพร้อมกันในคำขอเดียว (คำขอหนึ่ง
ครั้งจะเป็น list **หรือ** detail อย่างใดอย่างหนึ่งเท่านั้น) การเรียก
`filter_queryset(get_queryset())` แยกกันในแต่ละเมธอดจึงไม่มี query ซ้ำซ้อนเกิดขึ้นจริง
ในทางปฏิบัติ

**Q: ถ้าอยากให้ endpoint เดียวรองรับทั้ง list+create+retrieve+update+destroy
(ครบทั้ง 5 action ในหนึ่ง URL pattern เดียว ไม่แยกเป็นสอง class) ทำได้ไหม?**
A: ทำได้โดยผสม Mixin ทั้ง 5 ตัวเข้ากับ `GenericAPIView` ตัวเดียว แต่ในทางปฏิบัติ
**ไม่แนะนำ** เพราะขัดกับหลัก REST ที่แยก "collection" (`/posts/`) กับ "member"
(`/posts/<slug>/`) เป็นคนละ URL เสมอ (ตามที่เรียนใน Part 042 ขั้นตอนที่ 413.1) ถ้า
ต้องการลดความซ้ำซ้อนของโค้ดระหว่างสอง View ที่เกี่ยวกับ resource เดียวกัน คำตอบที่ถูก
ต้องกว่าคือการใช้ **`ViewSet`** ซึ่งรวม logic ของทั้งสอง URL ไว้ใน class เดียวโดยที่
`Router` เป็นคนแยกสร้าง URL ให้เอง — นี่คือหัวข้อหลักของ Part 044 ถัดไป

**Q: `get_serializer_class()` กับ `serializer_class` (attribute) ใช้ร่วมกันได้ไหม
หรือต้องเลือกอย่างใดอย่างหนึ่ง?**
A: ใช้ร่วมกันได้และเป็นแนวทางที่แนะนำด้วยซ้ำ: ตั้ง `serializer_class` เป็นค่า default
สำหรับกรณีทั่วไป แล้ว override `get_serializer_class()` เฉพาะกรณีที่ต้องการค่าอื่น
พร้อม fallback กลับไปที่ `super().get_serializer_class()` หรือ `self.serializer_class`
เมื่อไม่เข้าเงื่อนไขพิเศษใด ๆ — วิธีนี้ทำให้อ่านโค้ดง่ายกว่าใส่ทุกเงื่อนไขไว้ใน
`get_serializer_class()` ทั้งหมดโดยไม่มีค่า default ให้อ้างอิง

---

## เตรียมตัวสำหรับ Part ถัดไป

**Part 044: ViewSets และ Routers** (ขั้นตอนที่ 431-440) จะพา `PostListAPIView` และ
`PostDetailAPIView` ที่เพิ่งย่อให้สั้นลงใน Part นี้ไปรวมเป็น **`PostViewSet`** class
เดียว โดยใช้ `ModelViewSet` ที่ประกอบ Mixin ทั้ง 5 ตัวที่เรียนไปแล้วไว้ในที่เดียวกัน
พร้อม `Router` ที่สร้าง URL pattern ทั้งหมด (`list`, `create`, `retrieve`, `update`,
`partial_update`, `destroy`) ให้อัตโนมัติจาก class เดียว โดยไม่ต้องเขียน `urls.py`
เองทีละบรรทัดอีกต่อไป คุณจะได้เห็นว่า `self.action` ที่เกริ่นไว้ในขั้นตอนที่ 425.4
ทำงานอย่างไรจริง ๆ และเมื่อไหร่ควรเลือก `ViewSet` แทน Generic Views แบบที่เรียนใน
Part นี้

เตรียมเปิด `blog/api_views.py` และ `blog/api_urls.py` ของคุณไว้ให้พร้อม เพราะ Part 044
จะรวมทั้งสอง class ที่แยกกันอยู่ตอนนี้เข้าเป็น class เดียว และเขียน `urls.py` ใหม่ด้วย
`DefaultRouter` ทั้งหมด!
