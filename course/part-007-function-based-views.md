# Part 007: Views แบบ Function-Based เบื้องต้น

> **ขั้นตอนที่ 61-70 ของหลักสูตร** | Phase 1: รากฐาน Python & Django
>
> เป้าหมายของ Part นี้: เจาะลึกฝั่ง "View" ของ Django อย่างเต็มรูปแบบ ตั้งแต่การอ่านค่า
> ทุกอย่างจาก `HttpRequest`, การเลือกใช้ `HttpResponse` และ subclass ที่เหมาะกับแต่ละ
> สถานการณ์, การจัดการหลาย HTTP method ในฟังก์ชันเดียว, shortcut อย่าง `render()` และ
> `redirect()`, การป้องกัน error ด้วย `get_object_or_404()`, ไปจนถึง decorator ที่ควบคุม
> HTTP method และความปลอดภัยของ view เมื่อจบ Part นี้ คุณจะเขียน view แบบ Function-Based
> ที่ query ข้อมูลจริงจาก `Post` model และเชื่อมกับ URL ที่ออกแบบไว้ใน Part 006 ได้อย่าง
> มืออาชีพ พร้อมเข้าใจว่าทำไม Phase 3 ของหลักสูตรถึงจะแนะนำ Class-Based Views ต่อยอด

---

## สารบัญของ Part นี้

- ขั้นตอนที่ 61: `HttpRequest` object เจาะลึก
- ขั้นตอนที่ 62: `HttpResponse` และ Subclasses (`JsonResponse`, Redirect, NotFound, `Http404`)
- ขั้นตอนที่ 63: `request.GET` / `request.POST` และ `QueryDict` เจาะลึก
- ขั้นตอนที่ 64: จัดการหลาย HTTP Method ในฟังก์ชัน View เดียว
- ขั้นตอนที่ 65: `render()` Shortcut เจาะลึก
- ขั้นตอนที่ 66: `redirect()` Shortcut และ Post/Redirect/Get Pattern
- ขั้นตอนที่ 67: `get_object_or_404()` และ `get_list_or_404()`
- ขั้นตอนที่ 68: View Decorators: `require_http_methods`, `require_GET`, `require_POST`, `csrf_exempt`
- ขั้นตอนที่ 69: Function-Based Views vs Class-Based Views
- ขั้นตอนที่ 70: สรุปและแบบฝึกหัด

---

## ขั้นตอนที่ 61: `HttpRequest` object เจาะลึก

### 61.1 ทบทวนสถานะโปรเจกต์จาก Part 005-006

ก่อนเริ่ม Part นี้ โปรเจกต์ของคุณควรมีสถานะดังนี้:

- แอป `blog` มี `Post` model พร้อม field `title`, `content`, `created_at`, `updated_at`,
  `is_published` (จาก Part 005 ขั้นตอนที่ 50)
- `blog/urls.py` มี `app_name = 'blog'` และออกแบบ URL ไว้ล่วงหน้าสำหรับ `list`, `detail`
  (รับ slug), `archive_year`, `archive_month` (จาก Part 006 ขั้นตอนที่ 54-57 และแบบฝึกหัดที่ 1)
- `config/urls.py` include `blog.urls` ไว้ที่ prefix `posts/`

เนื่องจาก URL `blog:detail` ที่ออกแบบไว้ใน Part 006 ใช้ `<slug:slug>` แต่ `Post` model ที่สร้าง
ใน Part 005 ยังไม่มี field `slug` อย่างเป็นทางการ (มีแค่ในแบบฝึกหัดที่ 2 ของ Part 005) เรามา
ทำให้ model สมบูรณ์ก่อนเริ่มเขียน view จริงใน Part นี้:

```python
# blog/models.py
from django.db import models
from django.utils.text import slugify


class Post(models.Model):
    title = models.CharField(max_length=200)
    slug = models.SlugField(max_length=220, unique=True, blank=True)
    content = models.TextField()
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
```

> **หมายเหตุ**: การ override `save()` เพื่อสร้าง slug อัตโนมัติเป็นเทคนิคที่ใช้กันจริงในงาน
> production แต่รายละเอียดเชิงลึกของ Model methods, `Meta` options และ field ประเภทต่าง ๆ
> จะเรียนแบบเต็มรูปแบบใน Part 011 และ Part 015 ตอนนี้ขอให้เข้าใจแค่ว่า "เรามี field `slug`
> ที่พร้อมใช้กับ URL ที่ออกแบบไว้แล้ว" ก็เพียงพอ

รัน migration ให้เรียบร้อยก่อนไปต่อ:

```bash
python manage.py makemigrations blog
python manage.py migrate
```

ทดลองสร้างข้อมูลตัวอย่างผ่าน shell เพื่อใช้ทดสอบ view ตลอด Part นี้:

```bash
python manage.py shell
```

```python
>>> from blog.models import Post
>>> Post.objects.create(
...     title="เริ่มต้นเขียน Function-Based Views",
...     content="เนื้อหาตัวอย่างสำหรับทดสอบ view",
...     is_published=True,
... )
>>> Post.objects.create(
...     title="ทำความเข้าใจ HttpRequest",
...     content="อีกหนึ่งบทความทดสอบ",
...     is_published=True,
... )
>>> Post.objects.create(
...     title="ฉบับร่างที่ยังไม่เผยแพร่",
...     content="โพสต์นี้ is_published=False",
...     is_published=False,
... )
```

### 61.2 `HttpRequest` คืออะไร

ทุกครั้งที่ browser ส่ง HTTP request มาถึงแอป Django, ก่อนที่ view function ของคุณจะถูกเรียก
Django จะสร้าง **instance ของ `django.http.HttpRequest`** ขึ้นมาหนึ่งตัว แล้วส่งเป็น
พารามิเตอร์แรกเข้า view function เสมอ (ตามธรรมเนียมตั้งชื่อว่า `request`)

```python
def my_view(request):
    #        └──┬──┘
    #    HttpRequest instance ที่ Django สร้างให้อัตโนมัติ
    ...
```

`HttpRequest` คือ object ที่ **ห่อหุ้ม (encapsulate)** ข้อมูลทั้งหมดที่มากับ HTTP request:
method, headers, query string, ข้อมูลฟอร์ม, cookies, ไฟล์ที่อัปโหลด, และข้อมูล session/user
(เมื่อเปิดใช้ middleware ที่เกี่ยวข้อง) มันคือ "กล่องข้อมูล" ที่ view function ทุกตัวต้องอ่านค่า
ออกมาใช้งาน

### 61.3 `request.method`: HTTP Method ของ request นี้

```python
def debug_method(request):
    return HttpResponse(f"Method ที่ใช้: {request.method}")
```

`request.method` เป็น string พิมพ์ใหญ่เสมอ เช่น `'GET'`, `'POST'`, `'PUT'`, `'DELETE'`,
`'HEAD'`, `'OPTIONS'`, `'PATCH'` — เราจะใช้ attribute นี้เป็นหัวใจหลักของขั้นตอนที่ 64

### 61.4 `request.GET` และ `request.POST`: ภาพรวมก่อนเจาะลึกในขั้นตอนที่ 63

```python
def preview(request):
    print(request.GET)    # QueryDict จาก query string เช่น ?page=2
    print(request.POST)   # QueryDict จากข้อมูลฟอร์มที่ POST มา
    return HttpResponse("ดู console")
```

ทั้งสองตัวเป็น instance ของ `QueryDict` ไม่ใช่ `dict` ธรรมดา — เหตุผลและวิธีใช้งานแบบเจาะลึก
จะอยู่ในขั้นตอนที่ 63 ทั้งหมด

### 61.5 `request.headers`: อ่าน HTTP Headers แบบสะดวก (Django 2.2+)

```python
def show_headers(request):
    user_agent = request.headers.get('User-Agent', 'ไม่ทราบ')
    accept_lang = request.headers.get('Accept-Language', 'ไม่ทราบ')
    is_htmx = request.headers.get('HX-Request') == 'true'  # จะใช้จริงใน Part 053
    return HttpResponse(
        f"User-Agent: {user_agent}<br>"
        f"Accept-Language: {accept_lang}<br>"
        f"เป็น HTMX request หรือไม่: {is_htmx}"
    )
```

`request.headers` เป็น dict-like object ที่ **ไม่สนใจตัวพิมพ์เล็ก-ใหญ่ของชื่อ header**
(case-insensitive) ดังนั้น `request.headers.get('user-agent')` กับ
`request.headers.get('User-Agent')` ให้ผลลัพธ์เหมือนกัน ซึ่งสอดคล้องกับสเปกของ HTTP ที่ถือว่า
ชื่อ header ไม่สนตัวพิมพ์

### 61.6 `request.META`: Metadata ระดับต่ำแบบดั้งเดิม (WSGI environ)

ก่อนที่ Django จะมี `request.headers` (เพิ่มมาใน Django 2.2) นักพัฒนาต้องอ่าน header ผ่าน
`request.META` ซึ่งเป็น dict ธรรมดาที่มาจาก WSGI environ โดยตรง `request.META` ยังคงมีข้อมูล
ที่ `request.headers` ไม่มี เช่น ข้อมูลเกี่ยวกับ connection และ server:

```python
def show_meta(request):
    return HttpResponse(
        f"REMOTE_ADDR (IP ผู้ใช้): {request.META.get('REMOTE_ADDR')}<br>"
        f"SERVER_NAME: {request.META.get('SERVER_NAME')}<br>"
        f"REQUEST_METHOD: {request.META.get('REQUEST_METHOD')}<br>"
        f"HTTP_USER_AGENT (แบบดั้งเดิม): {request.META.get('HTTP_USER_AGENT')}<br>"
        f"CONTENT_TYPE: {request.META.get('CONTENT_TYPE')}"
    )
```

สังเกตว่า header ทุกตัวใน `request.META` จะถูกเติม prefix `HTTP_` และแปลงเป็นตัวพิมพ์ใหญ่
พร้อมแทนที่ `-` ด้วย `_` เช่น header `User-Agent` จะกลายเป็น key `HTTP_USER_AGENT`

| ประเด็น | `request.headers` | `request.META` |
|---|---|---|
| Django version | 2.2+ | ทุกเวอร์ชัน |
| Case-sensitivity | ไม่สนตัวพิมพ์เล็ก-ใหญ่ | สนตัวพิมพ์ใหญ่ (ต้องเขียน `HTTP_USER_AGENT` เป๊ะ ๆ) |
| ครอบคลุมข้อมูล | เฉพาะ HTTP headers | HTTP headers + WSGI/server metadata (`REMOTE_ADDR`, `SERVER_PORT` ฯลฯ) |
| แนะนำให้ใช้เมื่อ | อ่านค่า header ทั่วไป (แนะนำเป็นค่าเริ่มต้น) | ต้องการข้อมูลระดับ server/connection ที่ `headers` ไม่มี |

### 61.7 `request.body`: Raw Request Body เป็น Bytes

`request.body` คือเนื้อหาดิบของ request แบบ `bytes` เหมาะสำหรับกรณีที่ client ส่งข้อมูลมาเป็น
JSON แทนที่จะเป็นฟอร์ม HTML ทั่วไป (เพราะ `request.POST` จะว่างเปล่าถ้า content type ไม่ใช่
`application/x-www-form-urlencoded` หรือ `multipart/form-data`):

```python
import json
from django.http import HttpResponse, JsonResponse


def echo_json(request):
    if request.method == 'POST' and request.headers.get('Content-Type') == 'application/json':
        try:
            data = json.loads(request.body)
        except json.JSONDecodeError:
            return JsonResponse({'error': 'JSON ไม่ถูกต้อง'}, status=400)
        return JsonResponse({'received': data})
    return HttpResponse("ส่ง POST พร้อม JSON body มาทดสอบ")
```

ทดสอบด้วย `curl`:

```bash
curl -X POST http://127.0.0.1:8000/posts/echo/ \
  -H "Content-Type: application/json" \
  -d '{"title": "ทดสอบ", "tags": ["python", "django"]}'
```

> **ข้อควรระวัง**: `request.body` อ่านได้ **ครั้งเดียวเท่านั้น** ถ้า Django (หรือ middleware)
> เคยเข้าถึง `request.POST` ไปแล้ว การอ่าน `request.body` ต่ออาจได้ค่าว่างหรือ error ในบาง
> สถานการณ์ ควรเลือกใช้อย่างใดอย่างหนึ่งให้เหมาะกับชนิดของ request ที่ view นั้นออกแบบมารองรับ

### 61.8 `request.path`, `request.path_info`, และ `request.get_full_path()`

```python
def show_path_info(request):
    return HttpResponse(
        f"path: {request.path}<br>"
        f"get_full_path(): {request.get_full_path()}<br>"
        f"build_absolute_uri(): {request.build_absolute_uri()}"
    )
```

| Attribute/Method | ตัวอย่างผลลัพธ์ (เข้า `/posts/?page=2`) | ใช้เมื่อ |
|---|---|---|
| `request.path` | `/posts/` | ต้องการเฉพาะ path ไม่รวม query string |
| `request.get_full_path()` | `/posts/?page=2` | ต้องการ path พร้อม query string (เช่น ใช้ทำ "login แล้วกลับมาหน้าเดิม") |
| `request.build_absolute_uri()` | `http://127.0.0.1:8000/posts/?page=2` | ต้องการ URL แบบเต็มรวม domain (เช่น ใช้ในอีเมลแจ้งเตือน) |

### 61.9 `request.user`: ตัวอย่างเบื้องต้น (จะเจาะลึกใน Phase 4)

ถ้าโปรเจกต์เปิดใช้ `AuthenticationMiddleware` (ค่าเริ่มต้นของ Django ที่มีมาให้ตั้งแต่
`startproject`) ทุก `request` จะมี attribute `request.user` ให้เสมอ แม้ผู้ใช้จะยังไม่ login:

```python
def preview_user(request):
    if request.user.is_authenticated:
        return HttpResponse(f"สวัสดีคุณ {request.user.username}")
    return HttpResponse("คุณยังไม่ได้เข้าสู่ระบบ (AnonymousUser)")
```

ตอนนี้เนื่องจากเรายังไม่เรียนเรื่อง Authentication (Phase 4 ของหลักสูตร) `request.user` จะเป็น
instance ของ `django.contrib.auth.models.AnonymousUser` เสมอสำหรับผู้เข้าชมทุกคน (มี
`is_authenticated` เป็น `False` ตายตัว) เราจะกลับมาใช้ attribute นี้อย่างจริงจังใน Part 031

### 61.10 ตารางสรุป `HttpRequest` attributes ที่สำคัญที่สุด

| Attribute/Method | ชนิดข้อมูล | คำอธิบาย |
|---|---|---|
| `request.method` | `str` | HTTP method เช่น `'GET'`, `'POST'` |
| `request.GET` | `QueryDict` | ค่าจาก query string |
| `request.POST` | `QueryDict` | ค่าจากฟอร์มที่ POST มา (form-encoded/multipart เท่านั้น) |
| `request.FILES` | `MultiValueDict` | ไฟล์ที่อัปโหลดมา (เรียนเต็มใน Part 009 และ 058) |
| `request.headers` | dict-like (case-insensitive) | HTTP headers แบบสะดวกใช้ |
| `request.META` | `dict` | Metadata ดิบจาก WSGI environ |
| `request.body` | `bytes` | เนื้อหาดิบของ request |
| `request.path` | `str` | path อย่างเดียว ไม่รวม query string |
| `request.get_full_path()` | `str` | path พร้อม query string |
| `request.user` | `User`/`AnonymousUser` | ผู้ใช้ปัจจุบัน (ต้องมี `AuthenticationMiddleware`) |
| `request.COOKIES` | `dict` | cookies ที่ client ส่งมา |
| `request.session` | `SessionBase` | session ปัจจุบัน (เรียนเต็มใน Part 034) |
| `request.content_type` | `str` | Content-Type ของ request |

---

## ขั้นตอนที่ 62: `HttpResponse` และ Subclasses

### 62.1 `HttpResponse` พื้นฐาน

ทุก view function **ต้อง return** object ที่เป็น instance ของ `HttpResponse` (หรือ subclass
ของมัน) เสมอ ไม่เช่นนั้น Django จะ raise `ValueError`:

```python
from django.http import HttpResponse


def hello(request):
    return HttpResponse("<h1>สวัสดี Django</h1>")
```

`HttpResponse` รับพารามิเตอร์เพิ่มเติมได้หลายตัว:

```python
def custom_response(request):
    response = HttpResponse(
        content="<h1>เนื้อหาที่กำหนดเอง</h1>",
        content_type="text/html; charset=utf-8",
        status=200,
    )
    response['X-Custom-Header'] = 'MyValue'   # ตั้งค่า header เพิ่มเติมได้แบบ dict
    return response
```

| พารามิเตอร์ | ความหมาย | ค่า default |
|---|---|---|
| `content` | เนื้อหาของ response | ค่าว่าง |
| `content_type` | MIME type | `'text/html; charset=utf-8'` |
| `status` | HTTP status code | `200` |
| `reason` | ข้อความอธิบาย status (ไม่ค่อยได้ใช้) | มาตรฐานตาม status code |
| `charset` | encoding | ตาม `DEFAULT_CHARSET` (`utf-8`) |

### 62.2 `JsonResponse`: ตอบกลับเป็น JSON

```python
from django.http import JsonResponse


def api_status(request):
    return JsonResponse({
        'status': 'ok',
        'version': '1.0',
        'app': 'blog',
    })
```

`JsonResponse` ตั้งค่า `content_type='application/json'` ให้อัตโนมัติ และแปลง dict/list
เป็น JSON string ด้วย `json.dumps()` ภายใน ไม่ต้อง import `json` เอง

```python
def api_post_list(request):
    from .models import Post
    posts = list(Post.objects.filter(is_published=True).values('id', 'title', 'slug'))
    return JsonResponse(posts, safe=False)   # ต้องระบุ safe=False เมื่อส่ง list ไม่ใช่ dict
```

> **ทำไมต้องมี `safe=False`?** โดยค่าเริ่มต้น `JsonResponse` จะยอมรับเฉพาะ `dict` เท่านั้น
> เพื่อป้องกันช่องโหว่ด้านความปลอดภัยแบบเก่าที่เกี่ยวกับการส่ง JSON array เป็น response
> โดยตรง (เบราว์เซอร์รุ่นเก่าบางตัวมีช่องโหว่ให้แอบอ่าน JSON array ข้าม origin ได้) ถ้าคุณ
> มั่นใจว่าปลอดภัย (เช่น endpoint นี้ไม่มีข้อมูลอ่อนไหว) ให้ระบุ `safe=False` เพื่อส่ง list
> ได้โดยตรง

### 62.3 `HttpResponseRedirect` และ `HttpResponsePermanentRedirect`

```python
from django.http import HttpResponseRedirect, HttpResponsePermanentRedirect


def old_post_url(request, pk):
    return HttpResponseRedirect(f'/posts/id/{pk}/')          # 302 Found (ชั่วคราว)


def moved_permanently(request):
    return HttpResponsePermanentRedirect('/posts/')          # 301 Moved Permanently
```

ในทางปฏิบัติ เราแทบไม่เขียน `HttpResponseRedirect` ตรง ๆ แบบนี้ เพราะจะกลายเป็นการ hardcode
URL string (ผิดกฎเหล็กที่ตั้งไว้ตั้งแต่ Part 006 ขั้นตอนที่ 53.7) เราจะใช้ shortcut `redirect()`
แทนเสมอ ซึ่งเจาะลึกในขั้นตอนที่ 66

### 62.4 `HttpResponseNotFound`, `HttpResponseForbidden`, `HttpResponseBadRequest`, `HttpResponseServerError`

```python
from django.http import (
    HttpResponseNotFound,
    HttpResponseForbidden,
    HttpResponseBadRequest,
    HttpResponseServerError,
)


def manual_404(request):
    return HttpResponseNotFound("<h1>ไม่พบหน้าที่คุณต้องการ</h1>")


def manual_403(request):
    return HttpResponseForbidden("<h1>คุณไม่มีสิทธิ์เข้าถึง</h1>")


def manual_400(request):
    return HttpResponseBadRequest("<h1>คำขอไม่ถูกต้อง</h1>")


def manual_500(request):
    return HttpResponseServerError("<h1>เกิดข้อผิดพลาดของระบบ</h1>")
```

### 62.5 การ Raise `Http404` เทียบกับการ Return `HttpResponseNotFound`

นี่คือจุดที่มือใหม่มักสับสน: `Http404` ไม่ใช่ subclass ของ `HttpResponse` แต่เป็น **Exception**
ที่ต้อง `raise` ไม่ใช่ `return`:

```python
from django.http import Http404, HttpResponseNotFound
from .models import Post


# วิธีที่ 1: raise Http404 (แนะนำ - Django จะแปลงเป็น handler404 ให้อัตโนมัติ)
def post_detail_v1(request, slug):
    try:
        post = Post.objects.get(slug=slug, is_published=True)
    except Post.DoesNotExist:
        raise Http404("ไม่พบบทความที่คุณต้องการ")
    return render(request, 'blog/post_detail.html', {'post': post})


# วิธีที่ 2: return HttpResponseNotFound โดยตรง (ทำงานได้ แต่ข้ามระบบ handler404)
def post_detail_v2(request, slug):
    try:
        post = Post.objects.get(slug=slug, is_published=True)
    except Post.DoesNotExist:
        return HttpResponseNotFound("<h1>ไม่พบบทความ</h1>")
    return render(request, 'blog/post_detail.html', {'post': post})
```

| ประเด็น | `raise Http404(...)` | `return HttpResponseNotFound(...)` |
|---|---|---|
| กลไกการทำงาน | Django ดักจับ exception แล้วเรียก `handler404` ที่ตั้งค่าไว้ (Part 006 ขั้นตอนที่ 58) ให้อัตโนมัติ | คุณต้องเขียน HTML/template ของหน้า 404 เองในทุก view ที่ใช้ |
| ความสม่ำเสมอของหน้า error | สม่ำเสมอทั้งเว็บไซต์ (ใช้ template เดียวกันทุกที่) | เสี่ยงหน้า 404 แต่ละจุดหน้าตาไม่เหมือนกัน |
| การพัฒนาตอน `DEBUG=True` | แสดง technical 404 page ที่มีประโยชน์ตอน debug | แสดง response ตามที่เขียนไว้ตรง ๆ ไม่มี debug info เพิ่ม |
| คำแนะนำ | **ใช้เป็นค่าเริ่มต้นเสมอ** เมื่อไม่พบข้อมูลที่ค้นหา | ใช้เฉพาะกรณีพิเศษที่ต้องการ response แบบกำหนดเองจริง ๆ |

**กฎของหลักสูตรนี้**: เมื่อ view ไม่พบข้อมูลที่ผู้ใช้ร้องขอ ให้ `raise Http404(...)` เสมอ
(หรือใช้ `get_object_or_404()` ในขั้นตอนที่ 67 ซึ่งทำสิ่งนี้ให้อัตโนมัติ)

### 62.6 ตารางสรุป `HttpResponse` Subclasses ทั้งหมด

| Class | HTTP Status Code | ใช้เมื่อ |
|---|---|---|
| `HttpResponse` | 200 | response ปกติทั่วไป |
| `HttpResponseRedirect` | 302 | redirect ชั่วคราว |
| `HttpResponsePermanentRedirect` | 301 | redirect ถาวร (URL เปลี่ยนแบบถาวร) |
| `HttpResponseNotModified` | 304 | บอกว่า resource ไม่เปลี่ยนแปลง (ใช้กับ caching) |
| `HttpResponseBadRequest` | 400 | request ผิดรูปแบบ |
| `HttpResponseForbidden` | 403 | ไม่มีสิทธิ์เข้าถึง |
| `HttpResponseNotFound` | 404 | ไม่พบ resource |
| `HttpResponseNotAllowed` | 405 | HTTP method ไม่ได้รับอนุญาต (ขั้นตอนที่ 68 จะสร้างให้อัตโนมัติ) |
| `HttpResponseGone` | 410 | resource เคยมีอยู่แต่ถูกลบถาวรแล้ว |
| `HttpResponseServerError` | 500 | ข้อผิดพลาดฝั่งเซิร์ฟเวอร์ |
| `JsonResponse` | 200 (ปรับได้) | ส่งข้อมูลเป็น JSON |
| `StreamingHttpResponse` | 200 (ปรับได้) | ส่งข้อมูลขนาดใหญ่แบบ stream (เรียนใน Part 058, 071) |
| `FileResponse` | 200 (ปรับได้) | ส่งไฟล์กลับไปให้ดาวน์โหลด (เรียนใน Part 058) |

`Http404` **ไม่อยู่ในตารางนี้** เพราะไม่ใช่ subclass ของ `HttpResponse` แต่เป็น
`django.http.Http404` ซึ่งสืบทอดจาก Python `Exception` — จำหลักไว้ว่า: **`raise Http404`,
ไม่ใช่ `return Http404`**

---

## ขั้นตอนที่ 63: `request.GET` / `request.POST` และ `QueryDict` เจาะลึก

### 63.1 ทำไม `request.GET`/`request.POST` ไม่ใช่ `dict` ธรรมดา

ลองดูตัวอย่าง query string ที่มี key ซ้ำกัน:

```
/posts/search/?tag=python&tag=django&tag=web
```

ถ้า `request.GET` เป็น `dict` ธรรมดา จะเก็บได้แค่ **ค่าเดียวต่อ key** (ค่าสุดท้ายจะทับค่าก่อน
หน้าเสมอ) แต่ HTTP อนุญาตให้ query string หรือฟอร์ม HTML ส่งค่าซ้ำ key เดียวกันได้หลายค่า
(เช่น checkbox หลายตัวที่ใช้ `name` เดียวกัน) Django จึงออกแบบ `QueryDict` ซึ่งเป็น
**subclass ของ `dict`** ที่รองรับ **หลายค่าต่อหนึ่ง key** (multi-value dictionary) โดยเฉพาะ

### 63.2 `.get()` vs `[]` vs `.getlist()`

```python
def search_view(request):
    # .get(key, default) - คืนค่าตัวสุดท้ายที่ match key นั้น (เหมือน dict.get ทั่วไป)
    single_tag = request.GET.get('tag', '')

    # [] - เหมือน .get() แต่ raise KeyError ถ้าไม่มี key (ไม่ค่อยแนะนำให้ใช้ตรง ๆ)
    # single_tag = request.GET['tag']

    # .getlist(key) - คืนค่าทั้งหมดที่ match key นั้นเป็น list เสมอ (แม้มีค่าเดียวหรือไม่มีเลย)
    all_tags = request.GET.getlist('tag')   # ['python', 'django', 'web']

    return HttpResponse(
        f"ค่าเดียว (.get): {single_tag}<br>"
        f"ค่าทั้งหมด (.getlist): {all_tags}"
    )
```

| Method | พฤติกรรมเมื่อมีหลายค่า | พฤติกรรมเมื่อไม่มี key นั้น |
|---|---|---|
| `.get(key)` | คืนค่า**ตัวสุดท้าย**เท่านั้น | คืน `None` (หรือ default ที่ระบุ) |
| `[key]` | คืนค่าตัวสุดท้ายเช่นกัน | raise `MultiValueDictKeyError` |
| `.getlist(key)` | คืน **list ของทุกค่า** | คืน `[]` (list ว่าง) |

### 63.3 QueryDict เป็น Immutable โดยค่าเริ่มต้น

```python
def try_modify_get(request):
    try:
        request.GET['page'] = '999'   # จะ error!
    except Exception as e:
        return HttpResponse(f"เกิดข้อผิดพลาด: {type(e).__name__}: {e}")
```

`request.GET` และ `request.POST` เป็น **immutable QueryDict** (`request.GET._mutable`
คือ `False`) เพื่อป้องกันไม่ให้ view function แก้ไขข้อมูลที่มาจาก client โดยไม่ตั้งใจ ซึ่งอาจ
ทำให้เกิด side-effect ที่ไม่คาดคิดในระบบใหญ่ ถ้าต้องการแก้ไขค่า ต้อง `.copy()` ก่อนเสมอ (ซึ่ง
จะได้ QueryDict ที่ mutable):

```python
def modify_copy(request):
    mutable_get = request.GET.copy()   # ได้ QueryDict ใหม่ที่แก้ไขได้
    mutable_get['page'] = '999'
    return HttpResponse(f"หลังแก้ไข: {mutable_get.urlencode()}")
```

### 63.4 `QueryDict.urlencode()`: แปลงกลับเป็น Query String

```python
def rebuild_query(request):
    params = request.GET.copy()
    params['sort'] = 'newest'
    new_url = f"{request.path}?{params.urlencode()}"
    return HttpResponse(f"URL ใหม่: {new_url}")
```

เทคนิคนี้ใช้บ่อยมากเมื่อทำระบบ filter/sort/pagination ที่ต้อง "คงค่า filter เดิมไว้ แล้วเปลี่ยน
แค่บาง parameter" (เราจะใช้จริงเมื่อเรียน Pagination ใน Part 071)

### 63.5 `request.POST`: เหมือนกันทุกประการแต่มาจากฟอร์ม

```python
def contact_form_data(request):
    if request.method == 'POST':
        name = request.POST.get('name', '').strip()
        email = request.POST.get('email', '').strip()
        interests = request.POST.getlist('interests')   # checkbox หลายตัว name="interests"
        return HttpResponse(
            f"ชื่อ: {name}<br>อีเมล: {email}<br>ความสนใจ: {', '.join(interests)}"
        )
    return HttpResponse("กรุณาส่งข้อมูลผ่าน POST")
```

**ข้อควรจำสำคัญ**: `request.POST` จะถูกเติมค่าให้ **เฉพาะเมื่อ** request มี
`Content-Type` เป็น `application/x-www-form-urlencoded` หรือ `multipart/form-data`
เท่านั้น (คือรูปแบบฟอร์ม HTML มาตรฐาน) ถ้า client ส่งข้อมูลมาเป็น JSON (`Content-Type:
application/json`) `request.POST` จะเป็น **QueryDict ว่างเปล่า** เสมอ ไม่ว่า body จะมี
ข้อมูลมากแค่ไหนก็ตาม — กรณีนั้นต้องอ่านผ่าน `request.body` แล้ว `json.loads()` เอง ตามที่
เรียนในขั้นตอนที่ 61.7

### 63.6 ตารางเปรียบเทียบ `QueryDict` กับ `dict` มาตรฐานของ Python

| คุณสมบัติ | `dict` มาตรฐาน | `QueryDict` |
|---|---|---|
| รองรับหลายค่าต่อ key | ❌ ไม่รองรับ | ✅ รองรับผ่าน `.getlist()` |
| Mutable โดย default | ✅ | ❌ (`request.GET`/`request.POST` เป็น immutable) |
| มี `.urlencode()` | ❌ | ✅ |
| ที่มา | สร้างเองในโค้ด | สร้างโดย Django จาก query string/form data |
| ใช้ `.copy()` แล้วได้อะไร | `dict` ใหม่ (mutable เหมือนเดิม) | `QueryDict` ใหม่ที่ **mutable** |

---

## ขั้นตอนที่ 64: จัดการหลาย HTTP Method ในฟังก์ชัน View เดียว

### 64.1 รูปแบบพื้นฐาน: `if request.method == 'POST':`

หนึ่งในรูปแบบที่พบบ่อยที่สุดในการเขียน Function-Based View คือการให้ view เดียวรองรับได้ทั้ง
`GET` (แสดงฟอร์มเปล่า) และ `POST` (ประมวลผลข้อมูลที่ส่งมา):

```python
from django.http import HttpResponse


def newsletter_signup(request):
    if request.method == 'POST':
        email = request.POST.get('email', '').strip()
        if not email:
            return HttpResponse("กรุณากรอกอีเมล", status=400)
        # ในระบบจริงจะบันทึกลงฐานข้อมูลตรงนี้ (เรียนใน Part 011)
        return HttpResponse(f"ลงทะเบียนสำเร็จด้วยอีเมล: {email}")

    # ถ้าไม่ใช่ POST ให้ถือว่าเป็น GET แล้วแสดงฟอร์ม
    return HttpResponse(
        '<form method="post">'
        '<input type="email" name="email" placeholder="อีเมลของคุณ">'
        '<button type="submit">สมัคร</button>'
        '</form>'
    )
```

### 64.2 รูปแบบที่รองรับหลาย Method อย่างชัดเจนด้วย `elif`

```python
def post_toggle_publish(request, slug):
    from .models import Post
    from django.shortcuts import get_object_or_404

    post = get_object_or_404(Post, slug=slug)

    if request.method == 'POST':
        post.is_published = not post.is_published
        post.save()
        return HttpResponse(f"เปลี่ยนสถานะเผยแพร่ของ '{post.title}' เป็น {post.is_published}")
    elif request.method == 'GET':
        return HttpResponse(
            f"บทความ: {post.title}<br>"
            f"สถานะปัจจุบัน: {'เผยแพร่แล้ว' if post.is_published else 'ฉบับร่าง'}<br>"
            '<form method="post"><button type="submit">สลับสถานะ</button></form>'
        )
    else:
        from django.http import HttpResponseNotAllowed
        return HttpResponseNotAllowed(['GET', 'POST'])
```

สังเกตบรรทัดสุดท้าย: การ return `HttpResponseNotAllowed(['GET', 'POST'])` ด้วยตัวเองเป็น
การเขียนแบบ manual ที่ **ทำงานถูกต้อง** แต่ค่อนข้างซ้ำซ้อนถ้าต้องเขียนแบบนี้ในทุก view —
Django มี decorator ที่ทำสิ่งนี้ให้อัตโนมัติ ซึ่งจะเรียนในขั้นตอนที่ 68

### 64.3 เหตุผลที่ไม่ควรใช้ `if/elif` ซ้อนกันเยอะเกินไป

ถ้า view หนึ่งต้องรองรับ method เกิน 2-3 ตัวและ logic ของแต่ละ method ซับซ้อนมาก โค้ดจะเริ่ม
อ่านยากและทดสอบยาก สัญญาณที่บ่งบอกว่าถึงเวลาพิจารณาแนวทางอื่น:

- ฟังก์ชันยาวเกิน 40-50 บรรทัดเพราะมี logic ของหลาย method ปนกัน
- ต้อง `import` และเตรียมข้อมูลชุดเดียวกันซ้ำใน `if` แต่ละ branch
- ทดสอบ (test) ต้องเขียน mock ที่ซับซ้อนเพื่อแยกแยะ branch ไหนถูกเรียก

ในสถานการณ์แบบนี้ **Class-Based Views** (Phase 3, Part 021 เป็นต้นไป) จะช่วยได้มาก เพราะให้
เขียน method `get(self, request)` และ `post(self, request)` แยกจากกันเป็นธรรมชาติ โดยไม่ต้อง
เขียน `if/elif` เอง — เราจะเปรียบเทียบรายละเอียดในขั้นตอนที่ 69

### 64.4 ตัวอย่างที่สมบูรณ์: Contact Form พร้อม Validation อย่างง่าย

```python
# blog/views.py (ตัวอย่างประกอบ)
from django.http import HttpResponse
from django.shortcuts import redirect


def contact(request):
    errors = []
    if request.method == 'POST':
        name = request.POST.get('name', '').strip()
        message = request.POST.get('message', '').strip()

        if not name:
            errors.append('กรุณากรอกชื่อ')
        if not message:
            errors.append('กรุณากรอกข้อความ')
        elif len(message) < 10:
            errors.append('ข้อความต้องมีความยาวอย่างน้อย 10 ตัวอักษร')

        if not errors:
            # บันทึกข้อมูลสำเร็จ (จำลอง) แล้ว redirect ตามหลัก Post/Redirect/Get
            # (รายละเอียดเต็มของ redirect() อยู่ในขั้นตอนที่ 66)
            return redirect('blog:list')

    error_html = ''.join(f'<li>{e}</li>' for e in errors)
    return HttpResponse(
        f'<ul style="color:red">{error_html}</ul>'
        '<form method="post">'
        '<input name="name" placeholder="ชื่อ"><br>'
        '<textarea name="message" placeholder="ข้อความ"></textarea><br>'
        '<button type="submit">ส่ง</button>'
        '</form>'
    )
```

> **หมายเหตุ**: ตัวอย่างนี้ตรวจ validation แบบ manual เพื่อให้เห็นกลไกเบื้องหลังชัดเจน ในงาน
> จริงเราจะใช้ **Django Forms** (Part 025-027) ซึ่งจัดการเรื่อง validation, error message,
> และการ re-render ฟอร์มพร้อมค่าเดิมให้อัตโนมัติ ทำให้โค้ดสั้นและปลอดภัยกว่ามาก

---

## ขั้นตอนที่ 65: `render()` Shortcut เจาะลึก

### 65.1 สิ่งที่ `render()` ทำให้อัตโนมัติ

`django.shortcuts.render()` คือฟังก์ชันที่ใช้บ่อยที่สุดในบรรดา Django shortcuts ทั้งหมด
เพราะมันรวม 3 ขั้นตอนที่ต้องทำซ้ำ ๆ ทุกครั้งที่ต้องการ render template ให้เหลือ **บรรทัดเดียว**:

```python
from django.shortcuts import render


def post_list(request):
    posts = Post.objects.filter(is_published=True)
    return render(request, 'blog/post_list.html', {'posts': posts})
```

### 65.2 เขียนแบบ Manual เพื่อเข้าใจว่า `render()` ทำอะไรอยู่ข้างใน

```python
from django.http import HttpResponse
from django.template import loader


def post_list_manual(request):
    posts = Post.objects.filter(is_published=True)
    template = loader.get_template('blog/post_list.html')   # (1) โหลด template
    context = {'posts': posts}
    html = template.render(context, request)                 # (2) render เป็น HTML string
    return HttpResponse(html)                                 # (3) ห่อด้วย HttpResponse
```

`render()` ทำสามขั้นตอนนี้ให้อัตโนมัติในบรรทัดเดียว:

```
render(request, template_name, context) เทียบเท่ากับ:

1. template = loader.get_template(template_name)
2. html = template.render(context, request)
3. return HttpResponse(html)
```

### 65.3 Signature เต็มของ `render()`

```python
render(request, template_name, context=None, content_type=None, status=None, using=None)
```

| พารามิเตอร์ | ความหมาย | ค่า default |
|---|---|---|
| `request` | HttpRequest object (บังคับ) | - |
| `template_name` | ชื่อไฟล์ template หรือ list ของชื่อ (ลองทีละตัวจนกว่าจะเจอ) | - |
| `context` | dict ของตัวแปรที่ส่งเข้า template | `None` (เท่ากับ dict ว่าง) |
| `content_type` | MIME type ของ response | `'text/html; charset=utf-8'` |
| `status` | HTTP status code | `200` |
| `using` | ชื่อ template engine ที่จะใช้ (ถ้ามีมากกว่า 1 engine) | engine แรกที่ match |

ตัวอย่างการใช้ `status` เพื่อ render หน้า error ด้วยสถานะที่ถูกต้อง:

```python
def post_detail(request, slug):
    try:
        post = Post.objects.get(slug=slug, is_published=True)
    except Post.DoesNotExist:
        return render(request, 'blog/not_found.html', status=404)
    return render(request, 'blog/post_detail.html', {'post': post})
```

### 65.4 `RequestContext` และ Context Processors (เกริ่นล่วงหน้า)

สิ่งหนึ่งที่ `render()` ทำให้แบบเนียน ๆ (และมือใหม่มักไม่รู้) คือการส่ง `request` เข้าไปใน
`template.render(context, request)` ด้วย ซึ่งทำให้ Django สร้าง **`RequestContext`** แทนที่
จะเป็น `Context` ธรรมดา `RequestContext` จะรัน **context processors** ทั้งหมดที่ตั้งค่าไว้ใน
`TEMPLATES` ก่อนเสมอ ทำให้ตัวแปรบางตัว (เช่น `request`, `user`, `messages`) พร้อมใช้งานใน
**ทุก** template โดยไม่ต้องส่งเข้ามาเองใน context dict:

```html
<!-- ใน template ใด ๆ ที่ render ผ่าน render() จะมีตัวแปรเหล่านี้ให้ใช้ฟรี -->
{{ request.path }}
{{ user }}
```

ถ้าคุณเขียนแบบ manual ด้วย `Context` ธรรมดา (ไม่ใช่ `RequestContext`) ตัวแปรเหล่านี้จะ
**ไม่มีให้ใช้** รายละเอียดเต็มของ context processors จะเรียนใน Part 030

### 65.5 ตัวอย่าง Template ง่าย ๆ สำหรับทดสอบ `render()`

```html
<!-- blog/templates/blog/post_list.html -->
<!DOCTYPE html>
<html lang="th">
<head>
    <meta charset="UTF-8">
    <title>รายการบทความ</title>
</head>
<body>
    <h1>บทความทั้งหมด</h1>
    <ul>
        {% for post in posts %}
            <li>{{ post.title }} ({{ post.created_at|date:"d/m/Y" }})</li>
        {% empty %}
            <li>ยังไม่มีบทความที่เผยแพร่</li>
        {% endfor %}
    </ul>
</body>
</html>
```

> **หมายเหตุ**: เราแอบใช้ template tag `{% for %}` และ filter `{{ ... |date:"..." }}` ใน
> ตัวอย่างนี้เพียงเพื่อให้ `render()` มีอะไรแสดงผลจริง รายละเอียดเต็มรูปแบบของ Django
> Template Language — tag, filter, template inheritance ทั้งหมด — จะเรียนอย่างละเอียดใน
> **Part 008** ทันทีหลังจาก Part นี้

### 65.6 ตารางเปรียบเทียบ `render()` vs การเขียนแบบ Manual

| ประเด็น | `render()` shortcut | เขียนแบบ Manual |
|---|---|---|
| จำนวนบรรทัด | 1 บรรทัด | 3 บรรทัดขึ้นไป |
| ใช้ `RequestContext` อัตโนมัติ | ✅ ใช่ (context processors ทำงานให้) | ❌ ต้องเขียน `RequestContext` เอง |
| ความเสี่ยงลืมขั้นตอนใดขั้นตอนหนึ่ง | ต่ำมาก | สูงกว่า (อาจลืมห่อ `HttpResponse` หรือลืมส่ง `request`) |
| ความยืดหยุ่นสำหรับกรณีพิเศษ | ครอบคลุมกรณีทั่วไป 95%+ | ใช้เมื่อต้องการควบคุมทุกขั้นตอนเอง (พบน้อยมากในทางปฏิบัติ) |
| คำแนะนำ | **ใช้เป็นค่าเริ่มต้นเสมอ** | ใช้เฉพาะกรณีศึกษาเพื่อความเข้าใจ หรือความต้องการพิเศษจริง ๆ |

---

## ขั้นตอนที่ 66: `redirect()` Shortcut และ Post/Redirect/Get Pattern

### 66.1 ทบทวน: `redirect()` คืออะไร (จาก Part 006)

ใน Part 006 ขั้นตอนที่ 53.3 เราแนะนำ `redirect()` แบบสั้น ๆ ในบริบทของ named URL มาแล้ว
ในขั้นตอนนี้เราจะเจาะลึกทุกรูปแบบการใช้งานของมัน `django.shortcuts.redirect()` คือ shortcut
ที่คืนค่า `HttpResponseRedirect` (หรือ `HttpResponsePermanentRedirect`) ให้อัตโนมัติ โดยรับ
อาร์กิวเมนต์ได้ 3 รูปแบบหลัก

### 66.2 รูปแบบที่ 1: ส่งชื่อ URL (แนะนำที่สุด)

```python
from django.shortcuts import redirect


def create_post_stub(request):
    # ... สมมติว่าบันทึกข้อมูลสำเร็จ ...
    return redirect('blog:list')                     # ไม่มีพารามิเตอร์

def go_to_post(request, slug):
    return redirect('blog:detail', slug=slug)         # ส่ง kwargs เข้า reverse()
```

เทียบเท่ากับการเขียนแบบเต็ม:

```python
from django.urls import reverse
from django.http import HttpResponseRedirect

return HttpResponseRedirect(reverse('blog:detail', kwargs={'slug': slug}))
```

### 66.3 รูปแบบที่ 2: ส่ง Model Instance ที่มี `get_absolute_url()`

Django มีธรรมเนียมที่แนะนำให้ทุก model ที่มีหน้ารายละเอียดของตัวเอง ควรมี method
`get_absolute_url()` เพื่อให้ระบบต่าง ๆ (รวมถึง `redirect()`) หา URL ของ object นั้นได้โดย
อัตโนมัติ:

```python
# blog/models.py
from django.db import models
from django.urls import reverse
from django.utils.text import slugify


class Post(models.Model):
    title = models.CharField(max_length=200)
    slug = models.SlugField(max_length=220, unique=True, blank=True)
    content = models.TextField()
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

    def get_absolute_url(self):
        return reverse('blog:detail', kwargs={'slug': self.slug})
```

เมื่อมี `get_absolute_url()` แล้ว สามารถ `redirect()` ด้วย instance ได้ตรง ๆ:

```python
def create_and_redirect(request, post_id):
    post = Post.objects.get(pk=post_id)
    return redirect(post)   # เทียบเท่า redirect(post.get_absolute_url())
```

`get_absolute_url()` ยังมีประโยชน์อื่นอีกมาก เช่น Django Admin จะใช้มันแสดงปุ่ม "View on
site" อัตโนมัติ (เรียนใน Part 017) และ template สามารถเรียก `{{ post.get_absolute_url }}`
แทนการเขียน `{% url %}` ซ้ำได้เช่นกัน

### 66.4 รูปแบบที่ 3: ส่ง URL String ตรง ๆ (ใช้เท่าที่จำเป็น)

```python
def redirect_to_external(request):
    return redirect('https://docs.djangoproject.com/')   # URL ภายนอกที่ไม่มีชื่อใน urls.py
```

ใช้ได้เฉพาะกรณีที่ปลายทางเป็น URL ภายนอกระบบ (external link) เท่านั้น ถ้าเป็น URL ภายใน
โปรเจกต์ **ต้องใช้ชื่อ URL เสมอ** ตามกฎเหล็กที่ตั้งไว้ใน Part 006

### 66.5 `permanent=True`: เมื่อไหร่ควร Redirect แบบถาวร

```python
def old_slug_redirect(request, old_slug):
    # สมมติมีระบบ mapping slug เก่าไปยัง slug ใหม่ (เพื่อ SEO)
    return redirect('blog:detail', slug=old_slug, permanent=True)
```

| การใช้งาน | Status Code | ผลกับ SEO/Browser |
|---|---|---|
| `redirect(..., permanent=False)` (ค่า default) | 302 Found | Browser/Search engine จะยัง "จำ" URL เดิมไว้ ครั้งหน้าจะยังลองเข้า URL เดิมก่อน |
| `redirect(..., permanent=True)` | 301 Moved Permanently | Browser/Search engine จะปรับปรุง link/bookmark ไปยัง URL ใหม่ทันที เหมาะกับ URL ที่ย้ายแบบถาวร |

**คำแนะนำ**: ใช้ `permanent=False` (ค่า default) สำหรับ Post/Redirect/Get pattern และ
สถานการณ์ทั่วไป ใช้ `permanent=True` เฉพาะเมื่อ URL หนึ่งถูกออกแบบให้ "ย้ายไปอีกที่แบบถาวร"
จริง ๆ เท่านั้น เพราะ browser จะ cache การ redirect แบบ 301 ไว้อย่างจริงจัง ถ้าตั้งผิดแล้ว
ต้องรอ cache หมดอายุถึงจะแก้ไขให้ผู้ใช้เห็นผลได้

### 66.6 Post/Redirect/Get (PRG) Pattern: ปัญหาที่มันแก้

ลองจินตนาการว่า view ประมวลผลฟอร์มแล้ว `render()` หน้าผลลัพธ์กลับไปตรง ๆ โดยไม่ redirect:

```python
# ตัวอย่างที่ "ไม่ดี" - ไม่ใช้ PRG pattern
def create_comment_bad(request, post_slug):
    if request.method == 'POST':
        # ... บันทึก comment ...
        return render(request, 'blog/comment_success.html')   # ไม่ redirect!
    return render(request, 'blog/comment_form.html')
```

ปัญหาคือ: browser จะจำ **request สุดท้ายที่ทำ** ไว้เป็น `POST` เมื่อผู้ใช้กด **Refresh/F5**
บนหน้าผลลัพธ์ browser จะถามว่า "คุณต้องการส่งข้อมูลฟอร์มซ้ำหรือไม่?" และถ้าผู้ใช้กดยืนยัน
โดยไม่ทันสังเกต จะเกิด**การบันทึกข้อมูลซ้ำ** (เช่น comment เดิมถูกสร้างซ้ำสองครั้ง) — ปัญหานี้
เรียกว่า **duplicate form submission**

### 66.7 วิธีแก้ด้วย Post/Redirect/Get Pattern

```python
# ตัวอย่างที่ "ดี" - ใช้ PRG pattern
def create_comment_good(request, post_slug):
    if request.method == 'POST':
        # ... บันทึก comment ...
        return redirect('blog:detail', slug=post_slug)   # redirect ไปหน้าอื่นเสมอหลัง POST สำเร็จ
    return render(request, 'blog/comment_form.html')
```

ลำดับการทำงานของ PRG pattern:

```
1. Browser ส่ง POST ไปยัง /posts/hello-django/comment/
2. Server ประมวลผล บันทึกข้อมูลสำเร็จ
3. Server ตอบกลับด้วย HTTP 302 Redirect ไปยัง /posts/hello-django/
4. Browser ทำ GET request ใหม่ไปยัง /posts/hello-django/ โดยอัตโนมัติ
5. หน้าสุดท้ายที่ browser "จำ" ไว้คือ GET request (ขั้นที่ 4) ไม่ใช่ POST อีกต่อไป
6. ถ้าผู้ใช้กด Refresh ตอนนี้ จะเป็นการทำ GET ซ้ำ (ปลอดภัย ไม่มีข้อมูลซ้ำ)
```

**กฎเหล็กของหลักสูตรนี้: หลัง `POST` ที่ประมวลผลสำเร็จ ต้อง `redirect()` เสมอ ห้าม
`render()` หน้าผลลัพธ์กลับไปตรง ๆ เด็ดขาด** (ยกเว้นกรณีพิเศษเช่น API ที่คืน JSON ซึ่งไม่มี
concept ของ "browser refresh" แบบหน้าเว็บทั่วไป)

### 66.8 ตารางสรุป `redirect()` ทั้ง 3 รูปแบบ

| รูปแบบการเรียก | ตัวอย่าง | ใช้เมื่อ |
|---|---|---|
| ชื่อ URL + kwargs | `redirect('blog:detail', slug='hello')` | กรณีทั่วไปเกือบทั้งหมด (แนะนำที่สุด) |
| Model instance ที่มี `get_absolute_url()` | `redirect(post)` | เมื่อมี object อยู่ในมือแล้วและ model มี `get_absolute_url()` |
| URL string ตรง ๆ | `redirect('https://example.com/')` | เฉพาะ URL ภายนอกระบบเท่านั้น |

---

## ขั้นตอนที่ 67: `get_object_or_404()` และ `get_list_or_404()`

### 67.1 ปัญหาที่ต้องเขียนซ้ำ ๆ: `try/except Model.DoesNotExist`

จากขั้นตอนที่ 62.5 เราเห็นแล้วว่าการดึง object เดี่ยวมาแสดงผลต้องเขียน `try/except` ทุกครั้ง:

```python
def post_detail_verbose(request, slug):
    try:
        post = Post.objects.get(slug=slug, is_published=True)
    except Post.DoesNotExist:
        raise Http404("ไม่พบบทความที่คุณต้องการ")
    return render(request, 'blog/post_detail.html', {'post': post})
```

รูปแบบนี้ซ้ำซากมากจนพบในเกือบทุก view ที่ต้องแสดงรายละเอียดของ object เดียว Django จึงมี
shortcut `get_object_or_404()` ที่ทำสิ่งเดียวกันในบรรทัดเดียว

### 67.2 `get_object_or_404()`: ใช้กับ `Post` Model จริง

```python
from django.shortcuts import render, get_object_or_404
from .models import Post


def post_detail(request, slug):
    post = get_object_or_404(Post, slug=slug, is_published=True)
    return render(request, 'blog/post_detail.html', {'post': post})
```

`get_object_or_404(klass, *args, **kwargs)` ทำงานดังนี้ภายใน:

```
1. เรียก klass.objects.get(*args, **kwargs) ให้อัตโนมัติ
2. ถ้าเจอ object → คืนค่า object นั้นกลับมาตรง ๆ
3. ถ้าไม่เจอ (Model.DoesNotExist) → raise Http404 ให้อัตโนมัติ
4. ถ้าพารามิเตอร์ที่ส่งมาผิดจนเกิด MultipleObjectsReturned → ปล่อยให้ exception นั้น
   ผ่านออกไปตามปกติ (ไม่ใช่ 404 เพราะเป็น bug ของโค้ด ไม่ใช่ "ไม่พบข้อมูล")
```

พารามิเตอร์ตัวแรกยังรับ **QuerySet** แทน Model class ตรง ๆ ได้ด้วย ซึ่งมีประโยชน์มากเมื่อ
ต้องการจำกัดขอบเขตการค้นหาไว้ล่วงหน้า:

```python
def post_detail_published_only(request, slug):
    # ใช้ queryset ที่กรองไว้แล้วแทน Model class ตรง ๆ
    published_posts = Post.objects.filter(is_published=True)
    post = get_object_or_404(published_posts, slug=slug)
    return render(request, 'blog/post_detail.html', {'post': post})
```

ทั้งสองแบบด้านบนให้ผลลัพธ์เหมือนกันทุกประการ แต่แบบที่สองอ่านง่ายกว่าเมื่อเงื่อนไขการกรอง
มีหลายอย่างและถูกใช้ซ้ำในหลาย view (เพราะแยก queryset ออกมาเป็นตัวแปรหรือ manager method ได้
— รายละเอียดเรื่อง Custom Manager จะเรียนใน Part 015)

### 67.3 `get_list_or_404()`: เมื่อคาดหวังรายการ ไม่ใช่ object เดียว

```python
from django.shortcuts import get_list_or_404


def post_archive(request, year):
    posts = get_list_or_404(Post, created_at__year=year, is_published=True)
    return render(request, 'blog/post_archive.html', {'posts': posts, 'year': year})
```

`get_list_or_404()` ทำงานคล้ายกันแต่ใช้ `filter()` แทน `get()` ภายใน:

```
1. เรียก klass.objects.filter(*args, **kwargs).all() ให้อัตโนมัติ
2. ถ้า list ที่ได้ "ไม่ว่างเปล่า" → คืนค่า list นั้นกลับมา
3. ถ้า list ว่างเปล่า (ไม่มีข้อมูลตรงเงื่อนไขเลยสักตัว) → raise Http404 ให้อัตโนมัติ
```

> **ข้อควรระวัง**: `get_list_or_404()` ให้ 404 ทันทีที่ผลลัพธ์ **ว่างเปล่า** ซึ่งอาจไม่ใช่
> พฤติกรรมที่ต้องการเสมอไป เช่น หน้า "บทความในปี 2099" ที่ยังไม่มีข้อมูลอาจควรแสดง "ยังไม่มี
> บทความในปีนี้" มากกว่าหน้า 404 ในสถานการณ์แบบนี้การใช้ `Post.objects.filter(...)` ตรง ๆ
> แล้วปล่อยให้ template แสดงข้อความ "ไม่มีข้อมูล" ด้วย `{% empty %}` (ที่เห็นในขั้นตอนที่ 65.5)
> อาจเหมาะสมกว่า ขึ้นอยู่กับการออกแบบ UX ของแต่ละหน้า

### 67.4 ตารางเปรียบเทียบ Manual vs Shortcut

| ประเด็น | เขียนแบบ Manual (`try/except`) | `get_object_or_404()` / `get_list_or_404()` |
|---|---|---|
| จำนวนบรรทัด | 4-5 บรรทัด | 1 บรรทัด |
| ความสม่ำเสมอของพฤติกรรม 404 | ขึ้นกับแต่ละคนเขียน อาจไม่เหมือนกันทั้งโปรเจกต์ | ใช้กลไก `Http404` เดียวกันเสมอ สม่ำเสมอทั้งระบบ |
| อ่านเจตนาของโค้ดง่ายไหม | ต้องอ่าน `try/except` ทั้งบล็อกถึงเข้าใจ | ชื่อฟังก์ชันสื่อเจตนาชัดเจนในตัวเอง |
| รองรับ QuerySet ที่กรองไว้ล่วงหน้า | ต้องเขียนเอง | ✅ รองรับในตัว (ส่ง queryset แทน Model class) |
| คำแนะนำ | ใช้เมื่อต้องการ custom error message หรือ logic พิเศษ | **ใช้เป็นค่าเริ่มต้นเสมอ** สำหรับกรณีทั่วไป |

---

## ขั้นตอนที่ 68: View Decorators: `require_http_methods`, `require_GET`, `require_POST`, `csrf_exempt`

### 68.1 ปัญหาที่ Decorator เหล่านี้แก้

จากขั้นตอนที่ 64.2 เราเห็นแล้วว่าการเขียน `HttpResponseNotAllowed` ด้วยตัวเองทุกครั้งที่
ต้องการจำกัด HTTP method เป็นเรื่องซ้ำซาก Django จึงมีชุด decorator ใน
`django.views.decorators.http` ที่ทำสิ่งนี้ให้อัตโนมัติ

### 68.2 `require_http_methods`: จำกัด Method ที่อนุญาตแบบกำหนดเอง

```python
from django.views.decorators.http import require_http_methods
from django.http import HttpResponse


@require_http_methods(["GET", "POST"])
def flexible_view(request):
    if request.method == 'POST':
        return HttpResponse("ประมวลผล POST")
    return HttpResponse("แสดงฟอร์ม GET")
```

ถ้ามีคนพยายามเข้าด้วย method อื่น เช่น `DELETE` หรือ `PUT` Django จะคืน **HTTP 405 Method
Not Allowed** ให้อัตโนมัติทันที **ก่อนที่ code ภายในฟังก์ชันจะถูกรันด้วยซ้ำ** — นี่คือข้อดี
สำคัญ เพราะโค้ดใน view ไม่ต้องกังวลเรื่องตรวจสอบ method อีกเลย รับประกันได้ว่าถ้าโค้ดถูกรัน
method ต้องอยู่ใน list ที่อนุญาตแน่นอน

### 68.3 `require_GET` และ `require_POST`: Shortcut ที่ใช้บ่อยที่สุด

```python
from django.views.decorators.http import require_GET, require_POST


@require_GET
def post_list(request):
    posts = Post.objects.filter(is_published=True)
    return render(request, 'blog/post_list.html', {'posts': posts})


@require_POST
def like_post(request, slug):
    post = get_object_or_404(Post, slug=slug)
    # ... เพิ่มจำนวน like (จะเรียนเรื่อง F() expression ใน Part 014) ...
    return redirect('blog:detail', slug=slug)
```

`require_GET` เทียบเท่ากับ `@require_http_methods(["GET"])` และ `require_POST` เทียบเท่า
กับ `@require_http_methods(["POST"])` เพียงแต่สั้นและสื่อความหมายชัดเจนกว่าสำหรับกรณีที่พบ
บ่อยที่สุด

### 68.4 `require_safe`: อนุญาตเฉพาะ GET และ HEAD

```python
from django.views.decorators.http import require_safe


@require_safe
def cached_page(request):
    return render(request, 'blog/post_list.html', {'posts': Post.objects.filter(is_published=True)})
```

"Safe method" ในความหมายของ HTTP คือ method ที่ **ไม่ควรมีผลข้างเคียง (side effect)** ต่อ
ข้อมูลในระบบ ได้แก่ `GET` และ `HEAD` เท่านั้น `require_safe` เหมาะกับ view ที่ทำหน้าที่แค่
"แสดงผล" ล้วน ๆ ไม่มีการแก้ไขข้อมูลใด ๆ

### 68.5 การซ้อน Decorator หลายตัว

```python
from django.views.decorators.http import require_POST
from django.views.decorators.cache import never_cache


@never_cache
@require_POST
def submit_vote(request, slug):
    # ...
    return redirect('blog:detail', slug=slug)
```

Decorator ทำงานจากล่างขึ้นบน (ใกล้ function มากที่สุดทำงานก่อน) ในตัวอย่างนี้
`require_POST` ตรวจสอบ method ก่อน แล้ว `never_cache` ค่อยเติม header ป้องกันไม่ให้
browser cache response (จะเรียนเรื่อง caching เต็มรูปแบบใน Part 068-069)

### 68.6 `csrf_exempt`: คืออะไร และทำไมต้องระวังอย่างมาก

Django เปิดใช้ **CSRF Protection** (Cross-Site Request Forgery protection) เป็นค่าเริ่มต้น
ผ่าน `CsrfViewMiddleware` ซึ่งจะปฏิเสธ (403 Forbidden) ทุก `POST`/`PUT`/`DELETE`/`PATCH`
request ที่ไม่มี CSRF token ที่ถูกต้องแนบมาด้วย นี่คือหนึ่งในฟีเจอร์ "Secure by default"
ที่เราพูดถึงตั้งแต่ Part 001

```python
from django.views.decorators.csrf import csrf_exempt
from django.http import JsonResponse
import json


@csrf_exempt
def webhook_receiver(request):
    """
    Endpoint สำหรับรับ webhook จากบริการภายนอก (เช่น Stripe, GitHub)
    ระบบภายนอกเหล่านี้ไม่มี CSRF token ของเว็บเรา จึงจำเป็นต้องปิดการตรวจสอบ CSRF
    """
    if request.method != 'POST':
        return JsonResponse({'error': 'method not allowed'}, status=405)

    try:
        payload = json.loads(request.body)
    except json.JSONDecodeError:
        return JsonResponse({'error': 'invalid json'}, status=400)

    # ควรตรวจสอบลายเซ็น (signature) ของ webhook แทน CSRF เพื่อยืนยันความถูกต้อง
    # (รายละเอียดการตรวจ signature จะเรียนใน Part 078 เรื่อง Real-time Notification)
    return JsonResponse({'received': True})
```

### 68.7 คำเตือนด้านความปลอดภัย: ทำไมไม่ควรใช้ `csrf_exempt` พร่ำเพรื่อ

**CSRF (Cross-Site Request Forgery)** คือการโจมตีที่หลอกให้ browser ของเหยื่อ (ที่ login
ค้างอยู่ในเว็บของเรา) ส่ง request ที่เป็นอันตรายไปยังเว็บของเราโดยที่เหยื่อไม่รู้ตัว เช่น
เปิดเว็บอันตรายที่มีฟอร์มซ่อนอยู่ ซึ่งจะ auto-submit ไปยัง `POST /posts/delete-all/` ของ
เว็บเรา ถ้าเว็บไซต์ไม่มีการป้องกัน CSRF, browser จะแนบ cookie session ของเหยื่อไปด้วย
อัตโนมัติ ทำให้ server เข้าใจผิดว่าเหยื่อเป็นคนสั่งลบข้อมูลเอง

| ความเสี่ยงเมื่อใช้ `@csrf_exempt` พร่ำเพรื่อ | ผลกระทบ |
|---|---|
| เปิดช่องให้เว็บไซต์อันตรายสั่งการแทนผู้ใช้ที่ login อยู่ | ข้อมูลถูกลบ/แก้ไข/โอนเงินโดยที่เจ้าของบัญชีไม่รู้ตัว |
| นักพัฒนามือใหม่มักใช้เพื่อ "แก้ปัญหา 403 อย่างรวดเร็ว" โดยไม่เข้าใจสาเหตุจริง | ปิดระบบป้องกันที่ Django ตั้งใจเปิดไว้ให้โดยเจตนา |
| ลืมถอด `@csrf_exempt` ออกหลัง debug เสร็จ | ช่องโหว่ค้างอยู่ใน production แบบถาวร |

**กฎเหล็กด้านความปลอดภัยของหลักสูตรนี้**:

1. **ห้ามใช้ `@csrf_exempt` เพื่อ "แก้ปัญหา 403" อย่างเดียว** ให้หาสาเหตุที่แท้จริงก่อนเสมอ
   (ส่วนใหญ่มักเป็นเพราะลืมใส่ `{% csrf_token %}` ในฟอร์ม HTML)
2. **ใช้ `@csrf_exempt` เฉพาะกับ endpoint ที่ออกแบบมาให้ third-party เรียกโดยเฉพาะ**
   (เช่น webhook จากบริการภายนอกที่ไม่มีทางแนบ CSRF token ของเว็บเราได้) และต้องมีกลไก
   ยืนยันตัวตนอื่นทดแทนเสมอ เช่น การตรวจสอบ signature หรือ API key ลับ
3. **ห้ามใช้กับ endpoint ที่รับข้อมูลจากฟอร์มของผู้ใช้ทั่วไปในเว็บของเราเอง** เด็ดขาด
   เพราะฟอร์มของเราเองสามารถแนบ `{% csrf_token %}` ได้อยู่แล้วโดยไม่มีเหตุผลต้องปิด
4. **ทุกครั้งที่เห็น `@csrf_exempt` ในโค้ด review ให้ถามเสมอว่า** "endpoint นี้มีการยืนยัน
   ตัวตนทดแทนหรือไม่" ถ้าไม่มี ควรถือเป็นสัญญาณอันตรายที่ต้องแก้ไข

รายละเอียดเชิงลึกเรื่อง CSRF, `{% csrf_token %}`, และการตั้งค่า `CSRF_COOKIE_SECURE` สำหรับ
production จะเรียนอย่างเต็มรูปแบบใน **Phase 10: Security (Part 80-85)**

### 68.8 ตารางสรุป Decorators ทั้งหมดในขั้นตอนนี้

| Decorator | Module | หน้าที่ |
|---|---|---|
| `require_http_methods(list)` | `django.views.decorators.http` | อนุญาตเฉพาะ method ใน list ที่ระบุ |
| `require_GET` | `django.views.decorators.http` | อนุญาตเฉพาะ `GET` |
| `require_POST` | `django.views.decorators.http` | อนุญาตเฉพาะ `POST` |
| `require_safe` | `django.views.decorators.http` | อนุญาตเฉพาะ `GET`/`HEAD` (safe methods) |
| `csrf_exempt` | `django.views.decorators.csrf` | ปิดการตรวจสอบ CSRF token (ใช้อย่างระมัดระวังสูงสุด) |
| `never_cache` | `django.views.decorators.cache` | บังคับไม่ให้ browser cache response (เรียนเต็มใน Part 068) |

---

## ขั้นตอนที่ 69: Function-Based Views vs Class-Based Views

### 69.1 ทบทวน: Function-Based View (FBV) คือรูปแบบที่เราเรียนมาตลอด Part นี้

ทุกตัวอย่างใน Part 006-007 เป็น **Function-Based View (FBV)**: view ที่เป็นฟังก์ชัน Python
ธรรมดา รับ `request` เป็นพารามิเตอร์แรก คืนค่า `HttpResponse` เสมอ

ในขณะที่ **Class-Based View (CBV)** ซึ่งจะเรียนอย่างเต็มรูปแบบใน **Part 021-024 (Phase 3)**
เขียน view เป็น Python class ที่สืบทอดจาก `django.views.View` (หรือ generic view ต่าง ๆ)
โดยแยก logic ของแต่ละ HTTP method ออกเป็น method ของ class:

```python
# ตัวอย่างเปรียบเทียบ (CBV จะเรียนเต็มรูปแบบใน Part 021 ยังไม่ต้องเข้าใจ syntax ทั้งหมดตอนนี้)
from django.views import View
from django.http import HttpResponse


class PostListCBV(View):
    def get(self, request):
        posts = Post.objects.filter(is_published=True)
        return render(request, 'blog/post_list.html', {'posts': posts})

    def post(self, request):
        # logic สำหรับ POST แยกออกมาเป็นคนละ method อย่างชัดเจน
        return HttpResponse("ประมวลผล POST")
```

```python
# blog/urls.py - CBV ต้องเรียกผ่าน .as_view() เสมอตอนใช้ใน urlpatterns
path('', PostListCBV.as_view(), name='list'),
```

### 69.2 ข้อดี-ข้อเสียของ Function-Based Views

| ข้อดี | ข้อเสีย |
|---|---|
| อ่านง่าย เข้าใจง่ายสำหรับผู้เริ่มต้น (เป็นแค่ฟังก์ชัน Python ธรรมดา) | เมื่อต้อง reuse logic ข้าม view หลายตัว ต้องใช้ decorator หรือฟังก์ชันช่วยเอง (ไม่มีกลไก inheritance ในตัว) |
| Debug ตรงไปตรงมา (มี stack trace เดียว ไม่ต้องไล่ผ่าน method resolution order) | โค้ดซ้ำซ้อนได้ง่ายเมื่อหลาย view ทำหน้าที่คล้ายกันมาก (เช่น list/detail ของหลาย model) |
| ควบคุม flow การทำงานได้ 100% แบบ explicit (เห็นทุกอย่างในฟังก์ชันเดียว) | ฟังก์ชันยาวขึ้นเรื่อย ๆ เมื่อต้องรองรับหลาย HTTP method + validation + permission ปนกัน |
| ใช้ decorator ธรรมดาของ Python ได้ตรงไปตรงมา | ไม่มี pattern มาตรฐานสำหรับงานที่พบบ่อยมาก (list, create, update, delete) ต้องเขียนเองทุกครั้ง |

### 69.3 ข้อดี-ข้อเสียของ Class-Based Views

| ข้อดี | ข้อเสีย |
|---|---|
| Generic CBV (`ListView`, `DetailView`, `CreateView` ฯลฯ) ครอบคลุมงาน CRUD ทั่วไปโดยเขียนโค้ดน้อยมาก | Learning curve สูงกว่า ต้องเข้าใจ inheritance, MRO (Method Resolution Order), และ mixin |
| แยก logic ตาม HTTP method เป็น method ของ class โดยธรรมชาติ (`get()`, `post()`) ไม่ต้องเขียน `if/elif` เอง | Debug ยากกว่าเมื่อเกิดปัญหา เพราะต้องไล่ตาม class hierarchy หลายชั้น (โดยเฉพาะ generic view ที่ mixin เยอะ) |
| Reuse โค้ดได้ง่ายผ่าน **Mixin** (เรียนใน Part 024) เช่น `LoginRequiredMixin` | บาง flow การทำงาน "ซ่อน" อยู่เบื้องหลัง framework มากเกินไป ทำให้ปรับแต่งเฉพาะจุดยากกว่าที่คิด |
| ลดโค้ดซ้ำซ้อนได้มากในโปรเจกต์ที่มี CRUD หลายสิบ model | สำหรับ view ที่มี logic เฉพาะตัวมาก ๆ อาจไม่ได้ประโยชน์จาก generic view เลย (ต้อง override เกือบทุก method จนไม่ต่างจากเขียน FBV) |

### 69.4 ตารางเปรียบเทียบโดยตรง

| ประเด็น | Function-Based View | Class-Based View |
|---|---|---|
| รูปแบบพื้นฐาน | ฟังก์ชัน Python | Class ที่สืบทอดจาก `View` หรือ generic view |
| การจัดการหลาย HTTP method | เขียน `if/elif` เอง (ขั้นตอนที่ 64) | แยกเป็น method `get()`, `post()` ฯลฯ อัตโนมัติ |
| Reuse logic ข้าม view | ใช้ decorator หรือฟังก์ชันช่วยแยกไฟล์ | ใช้ inheritance และ Mixin |
| เหมาะกับงาน CRUD มาตรฐาน (list/detail/create/update/delete) | ต้องเขียนเองทุกครั้ง ซ้ำซ้อนถ้ามีหลาย model | Generic CBV ให้มาสำเร็จรูป เขียนไม่กี่บรรทัด |
| เหมาะกับ Logic ที่ซับซ้อน เฉพาะทาง ไม่ตรง pattern มาตรฐาน | เหมาะมาก (ควบคุมได้เต็มที่) | อาจต้อง override เยอะจนไม่คุ้ม |
| Learning curve | ต่ำ เหมาะกับผู้เริ่มต้น | สูงกว่า ต้องเข้าใจ OOP/Inheritance ดี |
| การเรียกใช้ใน `urls.py` | `path('...', views.my_view)` | `path('...', MyView.as_view())` |
| Django มีให้ครบสำเร็จรูปหรือไม่ | ไม่มี (เขียนเองทั้งหมด) | มี generic views จำนวนมาก (`ListView`, `DetailView`, `CreateView`, `UpdateView`, `DeleteView`) |

### 69.5 คำแนะนำ: เมื่อไหร่ควรใช้แบบไหน

- **ใช้ FBV** เมื่อ: logic ของ view มีความเฉพาะตัวสูง ไม่ตรงกับ pattern CRUD มาตรฐาน,
  ต้องการความชัดเจนแบบ "อ่านจากบนลงล่างแล้วเข้าใจทันที", กำลังสอน/เรียนรู้พื้นฐาน Django
  เป็นครั้งแรก (เหตุผลที่ Part 007 นี้สอน FBV ก่อน CBV)
- **ใช้ CBV** เมื่อ: งานเป็น CRUD มาตรฐาน (แสดงรายการ, แสดงรายละเอียด, สร้าง, แก้ไข, ลบ)
  ที่ตรงกับ generic view ของ Django พอดี, ต้องการ reuse logic (เช่น permission check) ข้าม
  หลาย view ผ่าน Mixin, โปรเจกต์มีขนาดใหญ่และมี pattern ซ้ำ ๆ จำนวนมาก

ในทางปฏิบัติ ทีมงานมืออาชีพส่วนใหญ่ **ใช้ทั้งสองแบบผสมกัน** ในโปรเจกต์เดียว: ใช้ Generic
CBV สำหรับหน้า CRUD มาตรฐาน และใช้ FBV สำหรับ view พิเศษที่มี business logic ซับซ้อนเฉพาะตัว
(เช่น webhook receiver ในขั้นตอนที่ 68.6 ซึ่งเขียนเป็น FBV ได้ง่ายกว่ามาก)

เราจะเจาะลึก Class-Based Views แบบเต็มรูปแบบใน **Part 021 (CBV เบื้องต้น)**, **Part 022
(Generic ListView/DetailView)**, **Part 023 (CreateView/UpdateView/DeleteView)**, และ
**Part 024 (Mixins และการสร้าง CBV เอง)** — ทั้งหมดนี้อยู่ใน **Phase 3** ของหลักสูตร

---

## ขั้นตอนที่ 70: สรุปและแบบฝึกหัด

### 70.1 ลงมือทำจริง: เขียน View 3 ตัวเชื่อมกับ URL จาก Part 006

ตอนนี้ถึงเวลารวมทุกอย่างที่เรียนมาใน Part นี้เข้าด้วยกัน โดยเขียน view จริง 3 ตัวที่ query
ข้อมูลจาก `Post` model จริง แล้วเชื่อมกับ `blog/urls.py` ที่ออกแบบโครงสร้างไว้ตั้งแต่ Part 006

```python
# blog/views.py
from django.shortcuts import render, get_object_or_404
from .models import Post


def blog_list(request):
    """
    แสดงรายการบทความทั้งหมดที่เผยแพร่แล้ว เรียงจากใหม่ไปเก่า
    (การใช้ .filter()/.order_by() แบบเจาะลึก รวมถึง QuerySet ขั้นสูงอื่น ๆ
    จะเรียนเต็มรูปแบบใน Part 013)
    """
    posts = Post.objects.filter(is_published=True).order_by('-created_at')
    return render(request, 'blog/post_list.html', {'posts': posts})


def blog_detail(request, slug):
    """
    แสดงรายละเอียดบทความตาม slug
    ใช้ get_object_or_404() แทนการเขียน try/except เอง (ขั้นตอนที่ 67)
    """
    post = get_object_or_404(Post, slug=slug, is_published=True)
    return render(request, 'blog/post_detail.html', {'post': post})


def blog_archive(request, year, month=None):
    """
    แสดงบทความตามปี (และเดือน ถ้าระบุ)
    รองรับทั้ง URL /posts/archive/<yyyy>/ และ /posts/archive/<yyyy>/<mm>/
    ด้วย view เดียวกัน โดยใช้ month=None เป็นค่าเริ่มต้น
    """
    posts = Post.objects.filter(is_published=True, created_at__year=year)
    if month is not None:
        posts = posts.filter(created_at__month=month)
    posts = posts.order_by('-created_at')

    context = {'posts': posts, 'year': year, 'month': month}
    return render(request, 'blog/post_archive.html', context)
```

เชื่อม view ทั้งสามเข้ากับ URL ที่ออกแบบไว้ใน Part 006 (รวม custom path converter `yyyy`
และ `mm` จากขั้นตอนที่ 57 ของ Part 006):

```python
# blog/urls.py
from django.urls import path, register_converter
from . import converters, views

register_converter(converters.FourDigitYearConverter, 'yyyy')
register_converter(converters.TwoDigitMonthConverter, 'mm')

app_name = 'blog'

urlpatterns = [
    path('', views.blog_list, name='list'),
    path('archive/<yyyy:year>/', views.blog_archive, name='archive_year'),
    path('archive/<yyyy:year>/<mm:month>/', views.blog_archive, name='archive_month'),
    path('<slug:slug>/', views.blog_detail, name='detail'),
]
```

> **ข้อควรระวังเรื่องลำดับ**: สังเกตว่า `<slug:slug>/` ถูกวางไว้ **หลังสุด** เพราะเป็น
> pattern ทั่วไปที่สุด ในขณะที่ `archive/<yyyy:year>/` เฉพาะเจาะจงกว่า ต้องวางไว้ก่อน ตามกฎ
> ทองที่เรียนใน Part 006 ขั้นตอนที่ 51.6: **"เรียง URL pattern จากเฉพาะเจาะจงที่สุดไปหาทั่วไป
> ที่สุดเสมอ"**

และตรวจสอบว่า `blog/converters.py` (จาก Part 006 ขั้นตอนที่ 57) ยังอยู่ครบถ้วน:

```python
# blog/converters.py
class FourDigitYearConverter:
    regex = '[0-9]{4}'

    def to_python(self, value):
        return int(value)

    def to_url(self, value):
        return '%04d' % value


class TwoDigitMonthConverter:
    regex = '[0-1][0-9]'

    def to_python(self, value):
        month = int(value)
        if not (1 <= month <= 12):
            raise ValueError('เดือนต้องอยู่ระหว่าง 01-12 เท่านั้น')
        return month

    def to_url(self, value):
        return '%02d' % value
```

สุดท้าย สร้าง template อีก 2 ไฟล์ที่ยังขาดอยู่ (นอกจาก `post_list.html` จากขั้นตอนที่ 65.5):

```html
<!-- blog/templates/blog/post_detail.html -->
<!DOCTYPE html>
<html lang="th">
<head>
    <meta charset="UTF-8">
    <title>{{ post.title }}</title>
</head>
<body>
    <a href="{% url 'blog:list' %}">← กลับหน้ารายการ</a>
    <h1>{{ post.title }}</h1>
    <p>เผยแพร่เมื่อ {{ post.created_at|date:"d/m/Y H:i" }}</p>
    <div>{{ post.content|linebreaks }}</div>
</body>
</html>
```

```html
<!-- blog/templates/blog/post_archive.html -->
<!DOCTYPE html>
<html lang="th">
<head>
    <meta charset="UTF-8">
    <title>คลังบทความ {{ year }}{% if month %}/{{ month }}{% endif %}</title>
</head>
<body>
    <a href="{% url 'blog:list' %}">← กลับหน้ารายการ</a>
    <h1>
        บทความในปี {{ year }}
        {% if month %}เดือน {{ month }}{% endif %}
    </h1>
    <ul>
        {% for post in posts %}
            <li><a href="{% url 'blog:detail' slug=post.slug %}">{{ post.title }}</a></li>
        {% empty %}
            <li>ยังไม่มีบทความในช่วงเวลานี้</li>
        {% endfor %}
    </ul>
</body>
</html>
```

ทดสอบด้วย `python manage.py runserver` แล้วเข้า URL ต่อไปนี้ (ใช้ข้อมูลตัวอย่างที่สร้างไว้
ในขั้นตอนที่ 61.1):

```
http://127.0.0.1:8000/posts/
http://127.0.0.1:8000/posts/2026/    (404 ถ้าไม่มี converter yyyy จับคู่ - ต้องเป็น 4 หลัก)
http://127.0.0.1:8000/posts/archive/2026/
http://127.0.0.1:8000/posts/archive/2026/09/
http://127.0.0.1:8000/posts/เริ่มต้นเขียน-function-based-views/   (ตัวอย่าง slug ที่ slugify() สร้างให้)
```

> **หมายเหตุ**: `slugify()` โดยค่าเริ่มต้นจะตัดตัวอักษรที่ไม่ใช่ ASCII (รวมภาษาไทย) ออก
> ตามที่เรียนใน Part 006 ขั้นตอนที่ 59.3 ดังนั้น slug ของบทความภาษาไทยล้วนอาจว่างเปล่าหรือ
> สั้นผิดปกติ ลองสร้างข้อมูลทดสอบเป็นชื่อภาษาอังกฤษ (เช่น `"Hello Django World"`) เพื่อให้
> เห็น slug ที่มีความหมายชัดเจนระหว่างฝึกฝน — เราจะแก้ปัญหานี้อย่างเป็นระบบเมื่อเรียนเรื่อง
> `SlugField` เชิงลึกใน Part 011

### 70.2 สรุปสิ่งที่ได้เรียนรู้ใน Part นี้

- ✅ เข้าใจ `HttpRequest` object ทุก attribute สำคัญ: `method`, `GET`, `POST`, `headers`,
  `META`, `body`, `path`, และ `user`
- ✅ รู้จัก `HttpResponse` และ subclass ทั้งหมด พร้อมความแตกต่างระหว่าง `raise Http404`
  กับ `return HttpResponseNotFound`
- ✅ เข้าใจว่าทำไม `request.GET`/`request.POST` ต้องเป็น `QueryDict` แทน `dict` ธรรมดา
  และใช้ `.get()`, `.getlist()`, `.copy()`, `.urlencode()` ได้อย่างถูกต้อง
- ✅ เขียน view เดียวที่จัดการหลาย HTTP method ด้วย `if request.method == 'POST':`
- ✅ เข้าใจกลไกเบื้องหลัง `render()` เทียบกับการเขียน `loader.get_template()` +
  `template.render()` + `HttpResponse` เอง
- ✅ ใช้ `redirect()` ได้ทั้ง 3 รูปแบบ และเข้าใจ Post/Redirect/Get pattern อย่างลึกซึ้ง
- ✅ ใช้ `get_object_or_404()` และ `get_list_or_404()` แทนการเขียน `try/except` เอง
- ✅ ใช้ decorator `require_http_methods`, `require_GET`, `require_POST` และเข้าใจความ
  เสี่ยงด้านความปลอดภัยของ `csrf_exempt`
- ✅ เปรียบเทียบ FBV กับ CBV ได้ และรู้ว่าเมื่อไหร่ควรเลือกแบบไหน
- ✅ เขียน `blog_list`, `blog_detail`, `blog_archive` ที่ query จาก `Post` model จริง และ
  เชื่อมกับ `blog/urls.py` ที่ออกแบบไว้ตั้งแต่ Part 006 ได้สำเร็จ

### 70.3 Checklist ก่อนไป Part ถัดไป

- [ ] `Post` model มี field `slug` และ `save()` ที่สร้าง slug อัตโนมัติจาก `title`
- [ ] เขียน view ที่อ่านค่าจาก `request.GET`/`request.POST`/`request.headers`/`request.body`
  ได้อย่างถูกต้องอย่างน้อยคนละ 1 ตัวอย่าง
- [ ] แยกแยะได้ว่าเมื่อไหร่ควร `raise Http404` เมื่อไหร่ควร `return HttpResponseNotFound`
- [ ] เขียน view ที่รองรับทั้ง GET และ POST ในฟังก์ชันเดียวได้ถูกต้อง
- [ ] อธิบายได้ว่า `render()` ทำอะไรให้อัตโนมัติ 3 ขั้นตอน
- [ ] ใช้ `redirect()` ตามหลัก Post/Redirect/Get ทุกครั้งหลัง `POST` ที่ประมวลผลสำเร็จ
- [ ] แทนที่ `try/except Post.DoesNotExist` ด้วย `get_object_or_404()` ในทุก view ที่ทำได้
- [ ] เขียนและทดสอบ view ที่มี `@require_POST` อย่างน้อย 1 ตัว
- [ ] `blog_list`, `blog_detail`, `blog_archive` ทำงานได้จริงกับข้อมูลจริงใน `Post` model
  และเชื่อมกับ `blog/urls.py` ได้สมบูรณ์

### 70.4 แบบฝึกหัดท้ายบท

**แบบฝึกหัดที่ 1 (พื้นฐาน)**: เขียน view ชื่อ `post_debug` ที่ path `/posts/debug/` (สร้าง
URL name `blog:debug` เพิ่มเอง) ให้แสดงข้อมูลทั้งหมดนี้เป็น HTML: `request.method`,
`request.path`, `request.get_full_path()`, `request.headers.get('User-Agent')`, และ
`request.GET.urlencode()` ทดลองเข้าด้วย query string ต่าง ๆ เช่น `/posts/debug/?a=1&b=2`
เพื่อดูผลลัพธ์ที่เปลี่ยนไป

**แบบฝึกหัดที่ 2 (ประยุกต์)**: เพิ่ม view `blog_search` ที่ path `/posts/search/` รับ query
parameter `q` (เช่น `/posts/search/?q=django`) แล้วใช้
`Post.objects.filter(title__icontains=q, is_published=True)` ค้นหาบทความที่ชื่อมีคำนั้น
(อย่ากังวลกับ `icontains` มากตอนนี้ — เป็นการเกริ่นล่วงหน้าเล็กน้อยก่อนเรียน QuerySet เต็ม
รูปแบบใน Part 013) ให้ view จัดการกรณีที่ไม่มี query parameter `q` เลย (แสดงข้อความ "กรุณา
พิมพ์คำค้นหา" แทนการ error) ด้วย `request.GET.get('q', '').strip()`

**แบบฝึกหัดที่ 3 (Decorator + Security)**: เขียน view `blog_publish_toggle` ที่รับเฉพาะ
`POST` เท่านั้น (ใช้ `@require_POST`) สลับค่า `is_published` ของ `Post` ที่ระบุด้วย `slug`
แล้ว `redirect()` กลับไปหน้า `blog:detail` ของบทความนั้น ทดสอบว่าเมื่อพยายามเข้าด้วย `GET`
(เช่น พิมพ์ URL ตรง ๆ ใน browser) จะได้ **HTTP 405** กลับมาแทนที่จะประมวลผลสำเร็จ

**แบบฝึกหัดที่ 4 (ขั้นสูง)**: ปรับ `blog_archive` จากขั้นตอนที่ 70.1 ให้ถ้าไม่พบบทความใน
ปี/เดือนที่ระบุเลย (`posts` ว่างเปล่า) ให้ raise `Http404` แทนการแสดงหน้าเปล่า พร้อมเขียน
ข้อความ error ที่บอกปีและเดือนที่ค้นหาไม่พบอย่างชัดเจน (เช่น `Http404(f"ไม่พบบทความในปี
{year}")`) จากนั้นเขียน automated test ด้วย `django.test.Client` (ตัวอย่างรูปแบบดูได้จาก
Part 006 ขั้นตอนที่ 58.7) เพื่อยืนยันว่า URL ปีที่ไม่มีข้อมูลจริงคืน status code 404

### 70.5 คำถามที่พบบ่อย (FAQ)

**Q: ทำไม view function ต้อง return `HttpResponse` เท่านั้น จะ return string หรือ dict
ตรง ๆ ไม่ได้เลยหรือ?**
A: ไม่ได้ Django กำหนดไว้ชัดเจนว่า view function ต้อง return object ที่เป็น instance ของ
`HttpResponse` (หรือ subclass) เท่านั้น ถ้า return ค่าอื่น เช่น string ธรรมดาหรือ `None`
Django จะ raise `ValueError` ทันทีพร้อมข้อความบอกว่า view คืนค่าอะไรมาแทน เหตุผลคือ Django
ต้องรู้แน่ชัดว่าจะแปลงค่านั้นเป็น HTTP response จริง ๆ อย่างไร (status code, headers,
content-type ฯลฯ) ซึ่ง `HttpResponse` เป็นตัวกำหนดสิ่งเหล่านี้ให้ชัดเจน

**Q: ใช้ `request.POST.get('field')` กับ `request.POST['field']` ต่างกันตรงไหนในทางปฏิบัติ?**
A: `.get()` จะคืนค่า `None` (หรือ default ที่ระบุ) ถ้าไม่มี key นั้น ในขณะที่ `[]` จะ
`raise MultiValueDictKeyError` ทันที ในงานจริงเกือบทั้งหมดควรใช้ `.get('field', '')` เพื่อ
ป้องกัน error เมื่อผู้ใช้ส่งข้อมูลฟอร์มมาไม่ครบ (เช่น ปิด JavaScript validation แล้วส่งฟอร์ม
เปล่ามา)

**Q: ทำไมไม่ใช้ `HttpResponseRedirect` ตรง ๆ แทน `redirect()` ไปเลย ดูแล้วต่างกันแค่ชื่อ?**
A: `redirect()` ทำงาน "ฉลาดกว่า" เพราะรับได้ทั้งชื่อ URL, model instance, และ URL string
ตรง ๆ แล้วเลือก logic ที่เหมาะสมให้อัตโนมัติ (เบื้องหลังยังคง return `HttpResponseRedirect`
เหมือนกัน) การใช้ `redirect()` ทำให้โค้ดสั้นลงและไม่ต้อง import `reverse()` แยกทุกครั้ง
ในทางปฏิบัติแทบไม่มีใครเขียน `HttpResponseRedirect(reverse(...))` ตรง ๆ อีกแล้วในโค้ดสมัยใหม่

**Q: ควรใช้ `get_object_or_404()` เสมอไหม หรือบางทีควรเขียน `try/except` เอง?**
A: ใช้ `get_object_or_404()` เป็นค่าเริ่มต้นเสมอสำหรับกรณี "ไม่พบข้อมูล = แสดงหน้า 404"
แต่ถ้า business logic ต้องการทำอย่างอื่นเมื่อไม่พบข้อมูล (เช่น สร้างข้อมูลใหม่แทน หรือ
redirect ไปหน้าอื่นแทนการแสดง 404) ต้องเขียน `try/except Model.DoesNotExist` เอง เพราะ
`get_object_or_404()` ทำได้แค่ "คืน object หรือ raise 404" เท่านั้น ไม่มีทางเลือกที่สาม

**Q: `csrf_exempt` กับการปิด CSRF middleware ทั้งระบบต่างกันอย่างไร อันไหนอันตรายกว่า?**
A: การปิด `CsrfViewMiddleware` ทั้งระบบใน `settings.py` อันตรายกว่ามาก เพราะทำให้ **ทุก
view ในทั้งโปรเจกต์** ไม่มีการป้องกัน CSRF เลย ในขณะที่ `@csrf_exempt` จำกัดผลกระทบไว้แค่
view เดียวที่ระบุเท่านั้น หลักสูตรนี้ไม่แนะนำการปิด middleware ทั้งระบบเด็ดขาดไม่ว่ากรณีใด ๆ
ถ้าจำเป็นต้องยกเว้นบาง endpoint จริง ๆ ให้ใช้ `@csrf_exempt` เฉพาะจุดพร้อมมาตรการยืนยันตัวตน
ทดแทนเสมอ ตามที่อธิบายในขั้นตอนที่ 68.7

---

## เตรียมตัวสำหรับ Part ถัดไป

**Part 008: Django Template Language เบื้องต้น** จะพาคุณเจาะลึกฝั่ง "Template" อย่างเต็ม
รูปแบบ ที่ Part นี้เราแอบใช้ `{% for %}`, `{% empty %}`, `{% url %}`, และ filter อย่าง
`|date` และ `|linebreaks` ไปบ้างแล้วในตัวอย่างของ `render()` แต่ยังไม่ได้อธิบายกลไกเบื้องหลัง
อย่างละเอียด ใน Part ถัดไปคุณจะได้เรียนรู้ **Template Inheritance** (`{% extends %}`,
`{% block %}`) เพื่อไม่ต้องเขียน `<!DOCTYPE html>` ซ้ำทุกไฟล์, ระบบ **Context** และการส่ง
ข้อมูลจาก view เข้า template อย่างเป็นระบบ, **Template Tags และ Filters** มาตรฐานที่ใช้บ่อย
ที่สุด, และวิธีจัดโครงสร้างโฟลเดอร์ `templates/` ให้ถูกต้องตามหลักการ namespace ที่เรียนไว้ใน
Part 005 ขั้นตอนที่ 47.2

Template ทั้ง 3 ไฟล์ที่คุณสร้างไว้ใน Part นี้ (`post_list.html`, `post_detail.html`,
`post_archive.html`) จะถูกนำมาปรับปรุงให้ใช้ **Template Inheritance** ร่วมกับ
`base.html` กลาง ทำให้ไม่ต้องเขียนโครงสร้าง HTML ซ้ำในทุกไฟล์อีกต่อไป เตรียมเปิด
`blog/views.py`, `blog/templates/blog/` และ `config/settings.py` (ส่วน `TEMPLATES`) ของคุณ
ไว้ให้พร้อม แล้วไปต่อกันเลย!
