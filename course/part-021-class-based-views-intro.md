# Part 021: Class-Based Views เบื้องต้น

> **ขั้นตอนที่ 201-210 ของหลักสูตร** | Phase 3: Views, Templates, Forms, CBV (ตอนที่ 1)
>
> เป้าหมายของ Part นี้: เข้าใจกลไกเบื้องหลัง Class-Based Views (CBV) อย่างถ่องแท้ ตั้งแต่
> `View` base class คืออะไร, `as_view()` ทำงานอย่างไรจนฟังก์ชัน Python ธรรมดาที่ Django
> ต้องการกลายเป็น class ได้, `dispatch()` เลือก method ตาม HTTP verb อย่างไร, ไปจนถึง
> `TemplateView` และ `RedirectView` ซึ่งเป็น generic CBV ที่ง่ายที่สุดสองตัว เมื่อจบ Part นี้
> คุณจะแปลง `blog_list`/`blog_detail` จาก Part 007 ให้เป็น CBV แบบ manual ได้เอง เข้าใจว่า
> ทำไม Django ถึงมีทั้ง FBV และ CBV ควบคู่กัน และพร้อมเรียนรู้ Generic CBV เต็มรูปแบบ
> (`ListView`, `DetailView`) ใน Part 022 ถัดไป

---

## สารบัญของ Part นี้

- ขั้นตอนที่ 201: View class คืออะไร และทำไม Django มีทั้ง FBV และ CBV
- ขั้นตอนที่ 202: กลไกภายในของ `as_view()` — จาก class สู่ฟังก์ชันที่ Django เรียกได้
- ขั้นตอนที่ 203: จัดการ GET/POST ด้วยการนิยาม method แยกกันในคลาสเดียว
- ขั้นตอนที่ 204: `TemplateView` — CBV ตัวแรกที่ง่ายที่สุด และ `get_context_data()`
- ขั้นตอนที่ 205: `RedirectView` — ใช้ redirect แบบไม่ต้องเขียน view เอง
- ขั้นตอนที่ 206: `http_method_names` — จำกัด HTTP method ที่ view รองรับ
- ขั้นตอนที่ 207: แปลง FBV `blog_list`/`blog_detail` จาก Part 007 ให้เป็น CBV แบบ manual
- ขั้นตอนที่ 208: `@method_decorator` — ใช้ decorator แบบ FBV กับ CBV method
- ขั้นตอนที่ 209: เชื่อม CBV เข้า `urls.py` ด้วย `.as_view()` และส่ง initial kwargs
- ขั้นตอนที่ 210: สรุปและแบบฝึกหัด

---

## ขั้นตอนที่ 201: View class คืออะไร และทำไม Django มีทั้ง FBV และ CBV

### 201.1 ทบทวนจาก Part 007 ขั้นตอนที่ 69

ใน Part 007 ขั้นตอนที่ 69 เราเกริ่นไว้แล้วว่า Class-Based View (CBV) คือการเขียน view เป็น
Python class แทนฟังก์ชัน โดยแยก logic ของแต่ละ HTTP method ออกเป็น method ของ class:

```python
from django.views import View
from django.http import HttpResponse


class PostListCBV(View):
    def get(self, request):
        return HttpResponse("แสดงรายการบทความ")
```

Part นี้จะเจาะลึกกลไกเบื้องหลังทั้งหมดที่ทำให้โค้ดข้างบนทำงานได้จริง เริ่มจากคำถามพื้นฐาน
ที่สุด: `View` base class คืออะไรกันแน่

### 201.2 `django.views.generic.base.View`: จุดเริ่มต้นของ CBV ทุกตัว

CBV ทุกตัวใน Django ไม่ว่าจะเป็น `TemplateView`, `ListView`, `CreateView` หรือ CBV ที่คุณ
เขียนเอง ล้วนสืบทอด (inherit) มาจากคลาสรากเดียวกันคือ `django.views.generic.base.View`
(นำเข้าแบบสั้นได้ผ่าน `django.views.View`) ลองดูโค้ดจริงของ `View` แบบย่อ (ตัด docstring
และส่วนที่ไม่จำเป็นออกเพื่อความกระชับ แต่โครงสร้างตรงกับซอร์สโค้ดจริงของ Django):

```python
# แนวคิดของ django.views.generic.base.View (ย่อจากซอร์สโค้ดจริง)
class View:
    http_method_names = [
        'get', 'post', 'put', 'patch', 'delete', 'head', 'options', 'trace',
    ]

    def __init__(self, **kwargs):
        for key, value in kwargs.items():
            setattr(self, key, value)

    @classmethod
    def as_view(cls, **initkwargs):
        # ตรวจสอบว่า initkwargs ไม่ทับ attribute ที่สำคัญ แล้วคืนฟังก์ชันตัวหนึ่ง
        def view(request, *args, **kwargs):
            self = cls(**initkwargs)
            self.setup(request, *args, **kwargs)
            return self.dispatch(request, *args, **kwargs)
        view.view_class = cls
        return view

    def setup(self, request, *args, **kwargs):
        self.request = request
        self.args = args
        self.kwargs = kwargs

    def dispatch(self, request, *args, **kwargs):
        if request.method.lower() in self.http_method_names:
            handler = getattr(
                self, request.method.lower(), self.http_method_not_allowed
            )
        else:
            handler = self.http_method_not_allowed
        return handler(request, *args, **kwargs)

    def http_method_not_allowed(self, request, *args, **kwargs):
        from django.http import HttpResponseNotAllowed
        return HttpResponseNotAllowed(self._allowed_methods())
```

**อย่าเพิ่งกังวลถ้ายังไม่เข้าใจทุกบรรทัด** — นี่คือแผนที่รวมของ Part นี้ทั้งหมด เราจะเจาะลึก
ทีละส่วน: `as_view()` ในขั้นตอนที่ 202, `dispatch()` และ `get()`/`post()` ในขั้นตอนที่ 203,
`http_method_names` ในขั้นตอนที่ 206

### 201.3 ทำไม Django ต้องมีทั้ง FBV และ CBV (ไม่ใช่แค่เลือกอย่างใดอย่างหนึ่ง)

คำถามที่มือใหม่มักสงสัย: "ถ้า CBV ดีกว่า ทำไม Django ไม่เลิกรองรับ FBV ไปเลย?" คำตอบคือ
FBV และ CBV แก้ปัญหาคนละแบบ และ Django ออกแบบมาให้ **ใช้ร่วมกัน** ในโปรเจกต์เดียวได้เสมอ
เพราะทั้งคู่สุดท้ายแล้วเป็นแค่ **"ฟังก์ชัน Python ที่รับ `request` แล้วคืน `HttpResponse`"**
เหมือนกันในมุมมองของ `urls.py` — urlpatterns ไม่สนใจว่าเบื้องหลังเป็น class หรือ function
ตราบใดที่ผลลัพธ์สุดท้ายที่ส่งเข้า `path()` เป็น **callable ที่รับ request แล้วคืน
HttpResponse**

```
FBV:  def my_view(request): ...              → callable อยู่แล้วในตัวเอง
CBV:  MyView.as_view()                        → เรียกแล้วได้ callable ตัวหนึ่งกลับมา
      (ไม่ใช่ MyView เอง แต่เป็นฟังก์ชัน view() ที่ as_view() สร้างขึ้น)
```

นี่คือกุญแจสำคัญที่สุดของ Part นี้: **`MyView` เองไม่ใช่ view function และ Django ไม่เคย
เรียก `MyView(request)` ตรง ๆ** สิ่งที่ Django เรียกจริงคือฟังก์ชันที่ `as_view()` สร้างขึ้น
มาใหม่ ซึ่งเราจะดูกลไกทั้งหมดในขั้นตอนที่ 202

### 201.4 ตารางเปรียบเทียบ: เมื่อไหร่ Django "เลือกใช้" อะไรเป็นค่าเริ่มต้น

| สถานการณ์ | Django แนะนำ | เหตุผล |
|---|---|---|
| `startapp` สร้าง `views.py` เปล่า | ไม่กำหนดฝั่งไหน | Django ให้อิสระเลือกเอง ไม่บังคับ |
| เอกสารทางการ (docs.djangoproject.com) สอน tutorial แรก | FBV ก่อน แล้วค่อยแนะนำ CBV | อ่านง่ายกว่าสำหรับผู้เริ่มต้น (เหตุผลเดียวกับที่หลักสูตรนี้สอน Part 007 ก่อน Part 021) |
| งาน CRUD มาตรฐาน (list/detail/create/update/delete) | CBV (Generic View) | Django มี `ListView`, `DetailView` ฯลฯ ให้พร้อมใช้ ลดโค้ดซ้ำ |
| Admin panel ภายในของ Django เอง | CBV เกือบทั้งหมด | `django.contrib.admin` สร้างด้วย CBV ภายใน |
| Django REST Framework (DRF) | CBV เกือบทั้งหมด (`APIView`, `ViewSet`) | สืบทอดแนวคิดเดียวกับ `View` ของ Django แต่ปรับสำหรับ API |

### 201.5 สรุปแนวคิดของขั้นตอนนี้

- CBV ทุกตัวใน Django สืบทอดจาก `django.views.generic.base.View`
- `View` มี 4 ส่วนหลัก: `http_method_names` (รายชื่อ method ที่รองรับ), `as_view()`
  (classmethod ที่แปลง class เป็น callable), `setup()` (เตรียม attribute พื้นฐาน),
  `dispatch()` (ตัดสินใจว่าจะเรียก method ไหนของ instance)
- FBV และ CBV ไม่ใช่คู่แข่งกัน แต่เป็นเครื่องมือคนละแบบที่ Django ตั้งใจให้ใช้ร่วมกัน
- จุดเชื่อมสำคัญระหว่างสองโลกคือ: **ใน `urls.py` สิ่งที่ `path()` ต้องการเสมอคือ callable
  ที่รับ `request` แล้วคืน `HttpResponse`** — FBV ให้ callable นั้นมาตรง ๆ ส่วน CBV ต้องเรียก
  `.as_view()` เพื่อสร้าง callable นั้นขึ้นมาก่อน

---

## ขั้นตอนที่ 202: กลไกภายในของ `as_view()` — จาก class สู่ฟังก์ชันที่ Django เรียกได้

### 202.1 คำถามหลักของขั้นตอนนี้

ทำไมเขียน `path('', PostListView.as_view(), name='list')` ใน `urls.py` แล้วมันทำงานได้?
`PostListView` เป็น **class** ไม่ใช่ฟังก์ชัน แล้ว `PostListView.as_view()` คืนอะไรออกมากันแน่
ที่ทำให้ `path()` ยอมรับได้เหมือนกับ FBV ทุกประการ?

### 202.2 ทดลองพิสูจน์ด้วย Python shell

มาพิสูจน์ด้วยตัวเองผ่าน `python manage.py shell`:

```python
>>> from django.views import View
>>> class PingView(View):
...     def get(self, request):
...         return None
...
>>> PingView
<class '__main__.PingView'>

>>> result = PingView.as_view()
>>> result
<function View.as_view.<locals>.view at 0x7f2a1c0b2ca0>

>>> callable(result)
True
>>> import inspect
>>> inspect.isfunction(result)
True
>>> inspect.isclass(result)
False
```

ผลลัพธ์ยืนยันสิ่งที่เกริ่นไว้ในขั้นตอนที่ 201.3: `PingView.as_view()` **ไม่ได้คืน class**
แต่คืน **ฟังก์ชัน Python ธรรมดา** ตัวหนึ่งที่ชื่อ `view` (สร้างขึ้นภายใน `as_view()` แบบ
closure) นี่คือเหตุผลที่ `path()` ยอมรับมันได้เหมือน FBV ทุกประการ เพราะในมุมมองของ
`path()` มันคือฟังก์ชันจริง ๆ ไม่ใช่ class

### 202.3 เขียน `as_view()` ใหม่แบบเข้าใจง่าย (Simplified Version)

มาดูเวอร์ชันที่ตัดรายละเอียดการตรวจสอบ error ออกเพื่อเห็นแก่นของกลไก:

```python
# เวอร์ชันย่อเพื่อการศึกษา — ตัดการตรวจสอบ error ออกจากของจริง
class View:
    @classmethod
    def as_view(cls, **initkwargs):
        """
        cls คือ class ที่เรียก .as_view() (เช่น PostListView)
        initkwargs คือ keyword arguments ที่ส่งเข้ามาตอนเรียก .as_view(...)
        เช่น .as_view(template_name='custom.html')
        """

        def view(request, *args, **kwargs):
            # ทุกครั้งที่มี HTTP request เข้ามาที่ URL นี้ ฟังก์ชัน view() นี้จะถูกเรียก
            self = cls(**initkwargs)          # (1) สร้าง instance ใหม่ของ PostListView
            self.setup(request, *args, **kwargs)  # (2) ผูก request/args/kwargs เข้ากับ self
            return self.dispatch(request, *args, **kwargs)  # (3) หา method ที่ถูกต้องแล้วเรียก

        view.view_class = cls        # เก็บ reference กลับไปที่ class ต้นฉบับ (มีประโยชน์ตอน debug)
        view.view_initkwargs = initkwargs
        return view                  # คืนฟังก์ชัน view นี้กลับไป (ไม่ใช่ class!)
```

### 202.4 ไล่ทีละขั้นตอนตอนมี request เข้ามาจริง

```
1. Browser ส่ง GET /posts/ เข้ามา
2. Django เจอ path('', PostListView.as_view(), name='list') ใน urls.py
   → สิ่งที่ Django ถืออยู่จริง ๆ คือฟังก์ชัน `view` ที่ as_view() คืนมา (ตอน urls.py
     ถูก import ครั้งแรกตอน server เริ่มทำงาน .as_view() ถูกเรียกไปแล้วครั้งเดียว)
3. Django เรียก view(request) เหมือนเรียก FBV ทุกประการ
4. ภายใน view():
   a. self = PostListView()          ← สร้าง instance ใหม่ "ทุกครั้งที่มี request เข้ามา"
   b. self.setup(request)             ← self.request = request, self.args, self.kwargs
   c. self.dispatch(request)          ← เลือกเรียก self.get(request) เพราะ method เป็น GET
5. self.get(request) คืน HttpResponse กลับไปตามปกติ
6. view() คืนค่านั้นต่อให้ Django
7. Django ส่ง HttpResponse กลับไปยัง Browser
```

### 202.5 ข้อเท็จจริงสำคัญ: instance ใหม่ทุกครั้งที่มี request

ประเด็นที่มือใหม่มักเข้าใจผิด: `PostListView.as_view()` ถูกเรียก **แค่ครั้งเดียว** ตอน
Django โหลด `urls.py` (ตอน server เริ่มทำงาน) แต่ฟังก์ชัน `view()` ที่ได้กลับมาจะถูกเรียก
**ทุกครั้ง** ที่มี HTTP request เข้ามาที่ URL นั้น และทุกครั้งที่ `view()` ถูกเรียก มันจะสร้าง
`self = cls(**initkwargs)` **instance ใหม่เอี่ยม** ขึ้นมาเสมอ

```
เปิด server 1 ครั้ง  →  .as_view() ถูกเรียก 1 ครั้ง (ตอน urls.py import)
มี request เข้ามา 1000 ครั้ง  →  PostListView() ถูกสร้างขึ้นใหม่ 1000 ครั้ง
                                   (คนละ instance กันทุกครั้ง ไม่มีการใช้ instance ซ้ำ)
```

นี่คือเหตุผลด้าน **thread safety** ที่สำคัญมาก: เพราะแต่ละ request ได้ instance ของตัวเอง
การเก็บ state ไว้ใน `self.xxx = ...` ระหว่างการประมวลผล request หนึ่ง ๆ จึงปลอดภัย ไม่มีทาง
ที่ request ของผู้ใช้ A จะไปเห็นข้อมูลที่ผูกกับ `self` ของผู้ใช้ B ปนกัน (ตราบใดที่ไม่ไปเก็บ
state ไว้ที่ **class attribute ที่ mutable** เช่น `list`/`dict` ซึ่งจะถูกแชร์ข้าม instance
— นี่คือกับดักคลาสสิกของ Python OOP ที่ต้องระวังเสมอไม่ใช่เฉพาะกับ Django)

### 202.6 การตรวจสอบ error ที่ `as_view()` ทำจริง (ที่เวอร์ชันย่อตัดออกไป)

ซอร์สโค้ดจริงของ `as_view()` มีการตรวจสอบเพิ่มเติมสองอย่างที่สำคัญ:

```python
@classmethod
def as_view(cls, **initkwargs):
    for key in initkwargs:
        if key in cls.http_method_names:
            raise TypeError(
                "You tried to pass in the %s method name as a "
                "keyword argument to %s(). Don't do that."
                % (key, cls.__name__)
            )
        if not hasattr(cls, key):
            raise TypeError(
                "%s() received an invalid keyword %r. as_view "
                "only accepts arguments that are already "
                "attributes of the class." % (cls.__name__, key)
            )
    ...
```

ลองทำผิดดูเพื่อเห็น error จริง:

```python
>>> from django.views import View
>>> class DemoView(View):
...     def get(self, request):
...         return None
...
>>> DemoView.as_view(get="hacked")
Traceback (most recent call last):
    ...
TypeError: You tried to pass in the get method name as a keyword
argument to DemoView(). Don't do that.

>>> DemoView.as_view(nonexistent_field="value")
Traceback (most recent call last):
    ...
TypeError: DemoView() received an invalid keyword 'nonexistent_field'.
as_view only accepts arguments that are already attributes of the class.
```

การตรวจสอบนี้ป้องกันข้อผิดพลาดสองแบบ: (1) ห้ามส่งชื่อ HTTP method (`get`, `post` ฯลฯ) เป็น
keyword argument เพราะจะไปทับ method ที่ dispatch ต้องใช้เลือก handler และ (2) ห้ามส่ง
keyword argument ที่ class ไม่มี attribute นั้นอยู่แล้ว (ป้องกันการพิมพ์ชื่อผิดโดยไม่รู้ตัว)
เราจะกลับมาใช้ความสามารถ "ส่ง initial kwargs" นี้จริงจังในขั้นตอนที่ 209

### 202.7 ตารางสรุปสิ่งที่ `as_view()` ทำทั้งหมด

| ขั้นตอนภายใน `as_view()` | ทำเมื่อไหร่ | หน้าที่ |
|---|---|---|
| ตรวจสอบ `initkwargs` ถูกต้อง | ตอนเรียก `.as_view(...)` (ครั้งเดียว ตอน import `urls.py`) | ป้องกันการส่งชื่อ HTTP method หรือชื่อ attribute ที่ไม่มีจริง |
| สร้างฟังก์ชัน `view()` | ตอนเรียก `.as_view(...)` (ครั้งเดียว) | สร้าง closure ที่ผูก `cls` และ `initkwargs` ไว้ |
| แนบ `view.view_class = cls` | ตอนเรียก `.as_view(...)` (ครั้งเดียว) | ให้ debug tool/introspection รู้ว่า view function นี้มาจาก class ไหน |
| `self = cls(**initkwargs)` | ทุกครั้งที่มี request เข้ามา | สร้าง instance ใหม่ต่อ request หนึ่งครั้งเสมอ |
| `self.setup(request, ...)` | ทุกครั้งที่มี request เข้ามา | ผูก `request`, `args`, `kwargs` เข้ากับ instance |
| `self.dispatch(request, ...)` | ทุกครั้งที่มี request เข้ามา | เลือก method ที่ถูกต้องแล้วเรียกมันจริง |

---

## ขั้นตอนที่ 203: จัดการ GET/POST ด้วยการนิยาม method แยกกันในคลาสเดียว

### 203.1 เจาะลึก `dispatch()` แบบทีละบรรทัด

จากขั้นตอนที่ 201.2 เราเห็นโครงของ `dispatch()` มาแล้ว มาดูแบบเต็มพร้อมคำอธิบาย:

```python
class View:
    http_method_names = [
        'get', 'post', 'put', 'patch', 'delete', 'head', 'options', 'trace',
    ]

    def dispatch(self, request, *args, **kwargs):
        # request.method คือ string ตัวพิมพ์ใหญ่เสมอ เช่น 'GET', 'POST' (ทบทวนจาก Part 007 ขั้นตอนที่ 61.3)
        if request.method.lower() in self.http_method_names:
            # getattr(self, 'get', ...) → หา method ชื่อ get ที่นิยามไว้ใน self
            # ถ้าไม่มี method นั้น ให้ใช้ self.http_method_not_allowed แทน (fallback)
            handler = getattr(
                self, request.method.lower(), self.http_method_not_allowed
            )
        else:
            # method แปลก ๆ ที่ไม่อยู่ใน http_method_names เลย (พบน้อยมากในทางปฏิบัติ)
            handler = self.http_method_not_allowed
        return handler(request, *args, **kwargs)
```

พูดเป็นภาษาง่าย ๆ: `dispatch()` ทำสิ่งเดียวกับที่เราเคยเขียนด้วยมือใน Part 007 ขั้นตอนที่
64 (`if request.method == 'POST': ... elif ...`) แต่ใช้ `getattr()` ของ Python เพื่อ
**"เดา" ชื่อ method จาก `request.method` โดยอัตโนมัติ** แทนการเขียน `if/elif` เอง

### 203.2 เปรียบเทียบโค้ดคู่กัน: FBV เดิม vs CBV ใหม่

โค้ด FBV จาก Part 007 ขั้นตอนที่ 64.2:

```python
# FBV แบบเดิม (Part 007)
def post_toggle_publish(request, slug):
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

เขียนใหม่เป็น CBV โดยใช้ `dispatch()` ที่มากับ `View` แล้ว:

```python
# blog/views.py
from django.http import HttpResponse
from django.shortcuts import get_object_or_404
from django.views import View
from .models import Post


class PostTogglePublishView(View):
    def get(self, request, slug):
        post = get_object_or_404(Post, slug=slug)
        return HttpResponse(
            f"บทความ: {post.title}<br>"
            f"สถานะปัจจุบัน: {'เผยแพร่แล้ว' if post.is_published else 'ฉบับร่าง'}<br>"
            '<form method="post"><button type="submit">สลับสถานะ</button></form>'
        )

    def post(self, request, slug):
        post = get_object_or_404(Post, slug=slug)
        post.is_published = not post.is_published
        post.save()
        return HttpResponse(f"เปลี่ยนสถานะเผยแพร่ของ '{post.title}' เป็น {post.is_published}")
```

สังเกตว่า **เราไม่ต้องเขียน `else: return HttpResponseNotAllowed(...)` เองอีกแล้ว** เพราะ
`dispatch()` ที่สืบทอดมาจัดการกรณี method ที่ไม่มี handler ให้อัตโนมัติผ่าน
`http_method_not_allowed` (จะเจาะลึกในขั้นตอนที่ 206)

### 203.3 ทดสอบว่า `dispatch()` ทำงานถูกต้องจริงด้วย `python manage.py shell`

```python
>>> from django.test import RequestFactory
>>> from blog.views import PostTogglePublishView
>>> factory = RequestFactory()

>>> request = factory.get('/posts/hello-django/toggle/')
>>> view = PostTogglePublishView.as_view()
>>> response = view(request, slug='hello-django')
>>> response.status_code
200

>>> request = factory.post('/posts/hello-django/toggle/')
>>> response = view(request, slug='hello-django')
>>> response.status_code
200

>>> request = factory.delete('/posts/hello-django/toggle/')
>>> response = view(request, slug='hello-django')
>>> response.status_code
405
```

`RequestFactory` (จาก `django.test`) คือเครื่องมือช่วยสร้าง `HttpRequest` ปลอมสำหรับทดสอบ
โดยไม่ต้องรัน development server จริง (เราจะใช้เครื่องมือนี้เต็มรูปแบบเมื่อเรียนเรื่อง
Automated Testing ใน Phase 7) สังเกตว่า request แบบ `DELETE` ได้ status `405` กลับมาทันที
เพราะ `PostTogglePublishView` ไม่มี method ชื่อ `delete` นิยามไว้ (แม้ `delete` จะอยู่ใน
`http_method_names` ของ class แม่ก็ตาม — `http_method_names` แค่บอกว่า method ไหน
"อนุญาตให้มี handler ได้" ไม่ได้แปลว่า "มี handler ให้อัตโนมัติ")

### 203.4 กรณีศึกษา: view ที่รองรับ 4 methods พร้อมกัน (สาธิตพลังของ CBV)

```python
from django.http import HttpResponse, JsonResponse
from django.views import View


class NoteAPIView(View):
    """
    สาธิต CBV ที่รองรับ 4 HTTP methods โดยแต่ละ method แยก logic ชัดเจน
    ไม่มี if/elif ปนกันเป็นบล็อกยาวเหมือน FBV (เทียบกับปัญหาที่พูดถึงใน
    Part 007 ขั้นตอนที่ 64.3)
    """

    def get(self, request, note_id=None):
        return JsonResponse({'action': 'read', 'note_id': note_id})

    def post(self, request):
        return JsonResponse({'action': 'create', 'title': request.POST.get('title', '')})

    def put(self, request, note_id):
        return JsonResponse({'action': 'update', 'note_id': note_id})

    def delete(self, request, note_id):
        return JsonResponse({'action': 'delete', 'note_id': note_id})
```

ลองเทียบว่าถ้าเขียนเป็น FBV เดียวจะมีลักษณะอย่างไร (เพื่อเห็นความแตกต่างชัดเจน):

```python
# เทียบเท่าแบบ FBV (โค้ดยาวและอ่านยากกว่าเมื่อ methods เยอะขึ้น)
def note_api(request, note_id=None):
    if request.method == 'GET':
        return JsonResponse({'action': 'read', 'note_id': note_id})
    elif request.method == 'POST':
        return JsonResponse({'action': 'create', 'title': request.POST.get('title', '')})
    elif request.method == 'PUT':
        return JsonResponse({'action': 'update', 'note_id': note_id})
    elif request.method == 'DELETE':
        return JsonResponse({'action': 'delete', 'note_id': note_id})
    else:
        from django.http import HttpResponseNotAllowed
        return HttpResponseNotAllowed(['GET', 'POST', 'PUT', 'DELETE'])
```

### 203.5 ตารางสรุปข้อดีของการแยก method เทียบกับ if/elif

| ประเด็น | `if/elif` ใน FBV | Method แยกใน CBV |
|---|---|---|
| จำนวน indent level เมื่อ logic ซับซ้อน | เพิ่มขึ้นเรื่อย ๆ ตามจำนวน `if` ที่ซ้อน | แต่ละ method เริ่มที่ indent level เดียวกันเสมอ |
| การเขียน unit test แยกทีละ method | ต้อง mock `request.method` ให้ตรงก่อนเรียกฟังก์ชันเดียว | เรียก `view_instance.get(request)` หรือ `.post(request)` ตรง ๆ ได้เลย |
| การเพิ่ม method ใหม่ภายหลัง | ต้องเพิ่ม `elif` อีกก้อนในฟังก์ชันเดิม (เสี่ยงกระทบโค้ดเดิม) | เพิ่ม method ใหม่แยกก้อน ไม่กระทบ method อื่น |
| ต้องเขียน `HttpResponseNotAllowed` เองไหม | ต้องเขียนเอง (Part 007 ขั้นตอนที่ 64.2) | `dispatch()` จัดการให้อัตโนมัติ |
| ความยาวรวมของโค้ด | รวมกันเป็นฟังก์ชันเดียวยาว | กระจายเป็นหลาย method สั้น ๆ |

---

## ขั้นตอนที่ 204: `TemplateView` — CBV ตัวแรกที่ง่ายที่สุด และ `get_context_data()`

### 204.1 `TemplateView` แก้ปัญหาอะไร

รูปแบบที่พบบ่อยมากคือ view ที่แค่ render template อย่างเดียว ไม่มี logic ซับซ้อน (เช่น
หน้า "เกี่ยวกับเรา", หน้า "ติดต่อเรา" แบบ static, หน้า landing page) ถ้าเขียนด้วย
`View` ธรรมดาต้องเขียน `get()` เองทุกครั้ง:

```python
from django.shortcuts import render
from django.views import View


class AboutView(View):
    def get(self, request):
        return render(request, 'blog/about.html')
```

`TemplateView` (จาก `django.views.generic`) ทำสิ่งนี้ให้สำเร็จรูป โดยแค่กำหนด
`template_name` เป็น class attribute:

```python
# blog/views.py
from django.views.generic import TemplateView


class AboutView(TemplateView):
    template_name = 'blog/about.html'
```

```python
# blog/urls.py
from .views import AboutView

urlpatterns = [
    # ...
    path('about/', AboutView.as_view(), name='about'),
]
```

โค้ดสองบรรทัดนี้เทียบเท่ากับ `AboutView` แบบเขียนมือด้านบนทุกประการ

### 204.2 เจาะลึกซอร์สโค้ดจริงของ `TemplateView`

`TemplateView` ใน Django ประกอบขึ้นจาก mixin หลายตัว (เราจะเรียนเรื่อง Mixin เต็มรูปแบบ
ใน Part 024 ตอนนี้ขอให้เห็นภาพรวมก่อน):

```python
# แนวคิดจากซอร์สโค้ดจริงของ django.views.generic.base
class ContextMixin:
    extra_context = None

    def get_context_data(self, **kwargs):
        kwargs.setdefault('view', self)
        if self.extra_context is not None:
            kwargs.update(self.extra_context)
        return kwargs


class TemplateResponseMixin:
    template_name = None
    template_engine = None
    response_class = TemplateResponse
    content_type = None

    def render_to_response(self, context, **response_kwargs):
        response_kwargs.setdefault('content_type', self.content_type)
        return self.response_class(
            request=self.request,
            template=self.get_template_names(),
            context=context,
            using=self.template_engine,
            **response_kwargs,
        )

    def get_template_names(self):
        if self.template_name is None:
            raise ImproperlyConfigured(
                "TemplateResponseMixin requires either a definition of "
                "'template_name' or an implementation of 'get_template_names()'"
            )
        return [self.template_name]


class TemplateView(TemplateResponseMixin, ContextMixin, View):
    def get(self, request, *args, **kwargs):
        context = self.get_context_data(**kwargs)
        return self.render_to_response(context)
```

โครงสร้างนี้แสดงให้เห็นแนวคิดสำคัญของ Generic CBV ทั้งหมดใน Django: **แยกความรับผิดชอบ
ออกเป็น mixin เล็ก ๆ แล้วประกอบ (compose) เข้าด้วยกัน** — `ContextMixin` รับผิดชอบเรื่อง
"เตรียมข้อมูลสำหรับ template", `TemplateResponseMixin` รับผิดชอบเรื่อง "แปลง context เป็น
HTTP response", และ `View` รับผิดชอบเรื่อง "routing ตาม HTTP method" ตามที่เรียนมาแล้ว

### 204.3 `get_context_data()`: จุดที่ override บ่อยที่สุดใน Generic CBV

เมื่อต้องการส่งข้อมูลเพิ่มเติมเข้า template นอกเหนือจากค่า default ให้ override
`get_context_data()` แล้วเรียก `super()` เสมอ:

```python
# blog/views.py
from django.views.generic import TemplateView
from .models import Post


class HomePageView(TemplateView):
    template_name = 'blog/home.html'

    def get_context_data(self, **kwargs):
        # สำคัญมาก: ต้องเรียก super().get_context_data(**kwargs) ก่อนเสมอ
        # เพื่อให้ context ที่ ContextMixin เตรียมไว้ (เช่น key 'view') ไม่หายไป
        context = super().get_context_data(**kwargs)
        context['latest_posts'] = Post.objects.filter(is_published=True)[:5]
        context['total_published'] = Post.objects.filter(is_published=True).count()
        context['page_title'] = 'หน้าแรก'
        return context
```

```html
<!-- blog/templates/blog/home.html -->
<!DOCTYPE html>
<html lang="th">
<head>
    <meta charset="UTF-8">
    <title>{{ page_title }}</title>
</head>
<body>
    <h1>บทความล่าสุด (ทั้งหมด {{ total_published }} บทความที่เผยแพร่แล้ว)</h1>
    <ul>
        {% for post in latest_posts %}
            <li>{{ post.title }}</li>
        {% empty %}
            <li>ยังไม่มีบทความ</li>
        {% endfor %}
    </ul>
</body>
</html>
```

### 204.4 กับดักคลาสสิก: ลืมเรียก `super().get_context_data()`

```python
# ❌ ผิด: ไม่เรียก super() เลย
class BadHomeView(TemplateView):
    template_name = 'blog/home.html'

    def get_context_data(self, **kwargs):
        # kwargs ตรงนี้มีแค่ค่าที่มาจาก URL (เช่น จาก path converter)
        # แต่ยังไม่มี key 'view' และ extra_context ที่ ContextMixin เตรียมให้
        return {'latest_posts': Post.objects.all()}
```

```python
# ✅ ถูก: เรียก super() ก่อนเสมอแล้วค่อยเพิ่มเข้าไป
class GoodHomeView(TemplateView):
    template_name = 'blog/home.html'

    def get_context_data(self, **kwargs):
        context = super().get_context_data(**kwargs)
        context['latest_posts'] = Post.objects.all()
        return context
```

**กฎเหล็กของหลักสูตรนี้: เมื่อ override `get_context_data()` (หรือ method อื่นที่เริ่มต้น
ด้วย `get_`/`post_` ใน Generic CBV) ให้เรียก `super()` เป็นบรรทัดแรกเสมอ แล้วค่อยแก้ไข
`context` ที่ได้กลับมา** หลักการนี้จะสำคัญยิ่งขึ้นเมื่อเรียน Mixin หลายตัวซ้อนกันใน Part 024
เพราะถ้าข้าม `super()` ไปแม้แต่จุดเดียว ห่วงโซ่ Method Resolution Order (MRO) จะขาดตอน
และ mixin ตัวถัดไปจะไม่ทำงานตามที่ออกแบบไว้

### 204.5 `extra_context`: ทางลัดเมื่อไม่ต้องคำนวณอะไรซับซ้อน

ถ้าข้อมูลที่ต้องการส่งเข้า template เป็นค่าคงที่ ไม่ต้อง query ฐานข้อมูล สามารถใช้
`extra_context` แทนการ override `get_context_data()` ทั้งหมดได้:

```python
class ContactPageView(TemplateView):
    template_name = 'blog/contact.html'
    extra_context = {
        'page_title': 'ติดต่อเรา',
        'support_email': 'support@example.com',
    }
```

### 204.6 ตารางสรุป `TemplateView`

| Attribute/Method | หน้าที่ | Override เมื่อไหร่ |
|---|---|---|
| `template_name` | ชื่อไฟล์ template ที่จะ render | เกือบทุกครั้ง (บังคับต้องมี) |
| `extra_context` | dict ค่าคงที่ที่จะรวมเข้า context อัตโนมัติ | เมื่อข้อมูลเป็นค่าคงที่ ไม่ต้องคำนวณ |
| `get_context_data(**kwargs)` | เตรียม context dict ทั้งหมดที่จะส่งเข้า template | เมื่อต้อง query ฐานข้อมูลหรือคำนวณค่า dynamic |
| `get_template_names()` | คืน list ของชื่อ template (รองรับหลาย candidate) | เมื่อต้องเลือก template แบบมีเงื่อนไข (เช่น mobile/desktop) |
| `get(request, *args, **kwargs)` | Handler สำหรับ HTTP GET (นิยามไว้ให้แล้ว) | แทบไม่ต้อง override เลยสำหรับ `TemplateView` |

---

## ขั้นตอนที่ 205: `RedirectView` — ใช้ redirect แบบไม่ต้องเขียน view เอง

### 205.1 ปัญหาที่ `RedirectView` แก้

บางครั้ง view ทั้งตัวมีหน้าที่แค่ "redirect ไปที่อื่น" ล้วน ๆ ไม่มี logic อื่นเลย เช่น URL
เก่าที่ย้ายไปที่ใหม่แบบถาวร หรือ URL แบบสั้น (`/gh/` → GitHub ขององค์กร) เขียนด้วย FBV:

```python
from django.shortcuts import redirect


def old_blog_url(request):
    return redirect('blog:list', permanent=True)
```

`RedirectView` (จาก `django.views.generic`) ทำสิ่งเดียวกันแบบไม่ต้องเขียน function เลย:

```python
# blog/urls.py
from django.views.generic import RedirectView

urlpatterns = [
    path('old-posts/', RedirectView.as_view(pattern_name='blog:list', permanent=True)),
]
```

### 205.2 พารามิเตอร์หลักของ `RedirectView`

```python
from django.views.generic import RedirectView

# แบบที่ 1: ระบุ URL ตรง ๆ ด้วย url
class ToDocsView(RedirectView):
    url = 'https://docs.djangoproject.com/'
    permanent = False   # 302 (ค่า default ของ RedirectView คือ True ต้องระวัง!)


# แบบที่ 2: ระบุด้วย pattern_name (แนะนำที่สุด - สอดคล้องกับกฎเหล็กเรื่อง named URL จาก Part 006)
class ToPostListView(RedirectView):
    pattern_name = 'blog:list'
    permanent = True    # 301


# แบบที่ 3: กำหนด query string เพิ่มเติม
class SearchRedirectView(RedirectView):
    pattern_name = 'blog:search'
    query_string = True   # คง query string เดิมไว้ตอน redirect (เช่น ?q=django)
```

> **ข้อควรระวังสำคัญ**: `RedirectView.permanent` มีค่า default เป็น **`True`** (คือ 301
> ถาวร) ซึ่งตรงข้ามกับ `redirect()` shortcut ที่เรียนใน Part 007 ขั้นตอนที่ 66.5 ที่มีค่า
> default เป็น `permanent=False` (302 ชั่วคราว) **ต้องระบุ `permanent=False` เองอย่าง
> ชัดเจนเสมอถ้าไม่ต้องการให้ browser จำ redirect นี้แบบถาวร** เพราะถ้าตั้งผิดแล้ว ผู้ใช้จะ
> ต้องรอ cache ของ browser หมดอายุก่อนถึงจะเห็นการเปลี่ยนแปลงเมื่อแก้ไขในภายหลัง (ปัญหา
> เดียวกับที่เตือนไว้ใน Part 007 ขั้นตอนที่ 66.5)

### 205.3 การใช้ `pattern_name` พร้อม URL kwargs

`RedirectView` ส่งต่อ URL kwargs ที่ได้รับมาจาก `urls.py` เข้าไปยัง `reverse()` ของ
`pattern_name` ให้อัตโนมัติ:

```python
# blog/urls.py
from django.views.generic import RedirectView

urlpatterns = [
    # URL เก่าที่เคยใช้ /articles/<slug>/ ก่อนเปลี่ยนมาเป็น /posts/<slug>/
    path(
        'articles/<slug:slug>/',
        RedirectView.as_view(pattern_name='blog:detail', permanent=True),
    ),
]
```

เมื่อผู้ใช้เข้า `/articles/hello-django/` Django จะจับ `slug='hello-django'` จาก URL แล้ว
`RedirectView` จะเรียก `reverse('blog:detail', kwargs={'slug': 'hello-django'})` ให้
อัตโนมัติ ก่อน redirect ไปยัง `/posts/hello-django/`

### 205.4 เจาะลึกซอร์สโค้ดจริงของ `RedirectView`

```python
# แนวคิดจากซอร์สโค้ดจริงของ django.views.generic.base
class RedirectView(View):
    permanent = False
    url = None
    pattern_name = None
    query_string = False

    def get_redirect_url(self, *args, **kwargs):
        if self.url:
            url = self.url % kwargs
        elif self.pattern_name:
            url = reverse(self.pattern_name, args=args, kwargs=kwargs)
        else:
            return None

        args_qs = self.request.META.get('QUERY_STRING', '')
        if args_qs and self.query_string:
            url = "%s?%s" % (url, args_qs)
        return url

    def get(self, request, *args, **kwargs):
        url = self.get_redirect_url(*args, **kwargs)
        if url:
            if self.permanent:
                return HttpResponsePermanentRedirect(url)
            return HttpResponseRedirect(url)
        else:
            return HttpResponseGone()

    # head, post, options, delete, put, patch ทั้งหมดเรียก self.get() ต่อ
    # (RedirectView ตั้งใจให้ redirect ได้ไม่ว่า method อะไรเข้ามา)
    head = get
    post = get
    options = get
    delete = get
    put = get
    patch = get
```

สังเกตบรรทัดสุดท้าย: `RedirectView` กำหนดให้ **ทุก HTTP method ชี้ไปที่ `get()` เดียวกัน**
ผ่านการ assign function reference ตรง ๆ (`head = get`) นี่คือเทคนิคที่ใช้พลังของการที่
Python method เป็นแค่ function object ธรรมดา — เราจะใช้เทคนิคคล้ายกันเมื่อออกแบบ Mixin เอง
ใน Part 024

### 205.5 ตารางสรุป `RedirectView`

| Attribute | หน้าที่ | ค่า default |
|---|---|---|
| `url` | URL string ตรง ๆ ที่จะ redirect ไป (รองรับ `%(key)s` แทนค่าจาก kwargs) | `None` |
| `pattern_name` | ชื่อ URL pattern (แนะนำที่สุด สอดคล้องกับกฎเหล็กจาก Part 006) | `None` |
| `permanent` | `True` = 301 ถาวร, `False` = 302 ชั่วคราว | `True` (**ต่างจาก `redirect()` shortcut ที่ default เป็น `False`**) |
| `query_string` | คง query string เดิมไว้ตอน redirect หรือไม่ | `False` |

---

## ขั้นตอนที่ 206: `http_method_names` — จำกัด HTTP method ที่ view รองรับ

### 206.1 ทบทวน `http_method_names` จากขั้นตอนที่ 201-203

`http_method_names` คือ class attribute ของ `View` ที่กำหนดว่า HTTP method ไหนบ้างที่
`dispatch()` ยอม "พิจารณา" หา handler ให้ ค่า default คือ:

```python
class View:
    http_method_names = [
        'get', 'post', 'put', 'patch', 'delete', 'head', 'options', 'trace',
    ]
```

สังเกตว่านี่เป็นแค่ **"รายชื่อ method ที่อนุญาตให้มี handler ได้"** ไม่ใช่ "รายชื่อ method
ที่ view นี้รองรับจริง" — การจะรองรับ method ไหนจริง ๆ ขึ้นอยู่กับว่า class นั้นนิยาม method
ชื่อนั้นไว้หรือไม่ (ตามที่เห็นในขั้นตอนที่ 203.3 ที่ `DELETE` ได้ 405 เพราะไม่มี `delete()`
นิยามไว้ แม้ `'delete'` จะอยู่ใน `http_method_names` ก็ตาม)

### 206.2 เมื่อไหร่ต้อง override `http_method_names` เอง

การ override `http_method_names` มีประโยชน์เมื่อต้องการ **บล็อก method บางตัวไว้ตั้งแต่
ระดับ class โดยเด็ดขาด** แม้จะนิยาม handler ของ method นั้นไว้โดยไม่ตั้งใจในอนาคตก็ตาม:

```python
from django.views import View
from django.http import JsonResponse


class ReadOnlyReportView(View):
    """
    View สำหรับ endpoint รายงานที่ต้องการ "อ่านอย่างเดียว" อย่างเข้มงวด
    จำกัดไม่ให้มี method ที่แก้ไขข้อมูลได้เลยในระดับ class แม้แต่ในอนาคต
    """
    http_method_names = ['get', 'head', 'options']

    def get(self, request):
        return JsonResponse({'report': 'ข้อมูลรายงานสรุป'})

    # ถ้ามีใครมาเพิ่ม def post(self, request): ... ในอนาคตโดยไม่ตั้งใจ
    # dispatch() จะยังคงปฏิเสธ POST ด้วย 405 อยู่ดี เพราะ 'post' ไม่อยู่ใน
    # http_method_names ที่ override ไว้ — เป็นเกราะป้องกันสองชั้น
```

ทดสอบพฤติกรรม:

```python
>>> from django.test import RequestFactory
>>> factory = RequestFactory()
>>> view = ReadOnlyReportView.as_view()

>>> response = view(factory.get('/reports/summary/'))
>>> response.status_code
200

>>> response = view(factory.post('/reports/summary/'))
>>> response.status_code
405
>>> response.headers['Allow']
'GET, HEAD, OPTIONS'
```

### 206.3 ความสัมพันธ์กับ `_allowed_methods()` และ header `Allow`

เมื่อ `dispatch()` เรียก `self.http_method_not_allowed(request, ...)` มันจะสร้าง
`HttpResponseNotAllowed` พร้อม header `Allow` ที่บอกรายชื่อ method ที่ใช้ได้จริงให้ client
รู้ (สอดคล้องกับมาตรฐาน HTTP):

```python
# แนวคิดจากซอร์สโค้ดจริง
class View:
    def http_method_not_allowed(self, request, *args, **kwargs):
        logger.warning(
            'Method Not Allowed (%s): %s',
            request.method, request.path,
            extra={'status_code': 405, 'request': request},
        )
        return HttpResponseNotAllowed(self._allowed_methods())

    def _allowed_methods(self):
        return [m.upper() for m in self.http_method_names if hasattr(self, m)]
```

สังเกต `_allowed_methods()`: มันวนดู `http_method_names` **ทุกตัว** แล้วเช็คว่า
`hasattr(self, m)` (มี method นั้นนิยามไว้จริงหรือไม่) จึงได้ list ของ method ที่ "อยู่ใน
`http_method_names` และมี handler จริง" มาแสดงใน header `Allow` — นี่คือเหตุผลที่ตัวอย่าง
ในขั้นตอนที่ 206.2 ได้ `'GET, HEAD, OPTIONS'` แม้ `ReadOnlyReportView` จะนิยามแค่ `get()`
เพียง method เดียว เพราะ `View` มี `head()` และ `options()` เตรียมไว้ให้เป็นค่าเริ่มต้น
อยู่แล้ว (`head` จะ fallback ไปเรียก `get` ให้อัตโนมัติถ้าไม่ได้ override, ส่วน `options`
คืนรายชื่อ method ที่อนุญาตเป็นค่าเริ่มต้น)

### 206.4 ตารางสรุป `http_method_names`

| ประเด็น | รายละเอียด |
|---|---|
| ค่า default | `['get', 'post', 'put', 'patch', 'delete', 'head', 'options', 'trace']` |
| ความหมาย | รายชื่อ method ที่ `dispatch()` "ยอมพิจารณา" หา handler ให้ |
| ไม่ได้แปลว่าอะไร | ไม่ได้แปลว่า view รองรับ method นั้นจริง (ต้องมี method นิยามไว้ด้วย) |
| เมื่อไหร่ควร override | ต้องการบล็อก method บางตัวไว้แบบเข้มงวดในระดับ class (defense in depth) |
| ผลกับ header `Allow` | `_allowed_methods()` ใช้ทั้ง `http_method_names` และ `hasattr()` ร่วมกันคำนวณ |

---

## ขั้นตอนที่ 207: แปลง FBV `blog_list`/`blog_detail` จาก Part 007 ให้เป็น CBV แบบ manual

### 207.1 ทบทวนโค้ดต้นฉบับจาก Part 007 ขั้นตอนที่ 70.1

```python
# blog/views.py (เวอร์ชัน FBV จาก Part 007)
from django.shortcuts import render, get_object_or_404
from .models import Post


def blog_list(request):
    posts = Post.objects.filter(is_published=True).order_by('-created_at')
    return render(request, 'blog/post_list.html', {'posts': posts})


def blog_detail(request, slug):
    post = get_object_or_404(Post, slug=slug, is_published=True)
    return render(request, 'blog/post_detail.html', {'post': post})
```

Part นี้จงใจแปลงด้วย `View` ธรรมดา (**ไม่ใช้** `ListView`/`DetailView`) เพื่อให้เห็นภาพว่า
Generic CBV ที่จะเรียนใน Part 022 นั้น **ไม่ใช่เวทมนตร์** แต่เป็นแค่โค้ดแบบนี้ที่ Django
เขียนไว้ให้สำเร็จรูปแล้วเท่านั้นเอง

### 207.2 แปลง `blog_list` เป็น CBV แบบ manual ทีละขั้น

```python
# blog/views.py
from django.shortcuts import render, get_object_or_404
from django.views import View
from .models import Post


class BlogListView(View):
    """
    เทียบเท่ากับ blog_list(request) แบบ FBV จาก Part 007 ขั้นตอนที่ 70.1
    ทุกประการ เพียงแต่ห่อ logic ไว้ใน method get() ของ class แทน
    """
    template_name = 'blog/post_list.html'

    def get(self, request):
        posts = Post.objects.filter(is_published=True).order_by('-created_at')
        context = {'posts': posts}
        return render(request, self.template_name, context)
```

### 207.3 แปลง `blog_detail` เป็น CBV แบบ manual ทีละขั้น

```python
class BlogDetailView(View):
    """
    เทียบเท่ากับ blog_detail(request, slug) แบบ FBV จาก Part 007 ขั้นตอนที่ 70.1
    ทุกประการ พารามิเตอร์ slug ที่เคยรับตรง ๆ ในฟังก์ชัน ตอนนี้กลายเป็น
    พารามิเตอร์ของ method get() แทน (Django ส่งผ่าน dispatch() ให้อัตโนมัติ
    ตามที่เรียนในขั้นตอนที่ 202-203)
    """
    template_name = 'blog/post_detail.html'

    def get(self, request, slug):
        post = get_object_or_404(Post, slug=slug, is_published=True)
        context = {'post': post}
        return render(request, self.template_name, context)
```

### 207.4 แปลง `blog_archive` (ที่มีพารามิเตอร์ default) เป็น CBV

`blog_archive` จาก Part 007 มีความซับซ้อนเพิ่มเติมคือพารามิเตอร์ `month=None` ที่เป็น
optional ลองแปลงดู:

```python
class BlogArchiveView(View):
    template_name = 'blog/post_archive.html'

    def get(self, request, year, month=None):
        posts = Post.objects.filter(is_published=True, created_at__year=year)
        if month is not None:
            posts = posts.filter(created_at__month=month)
        posts = posts.order_by('-created_at')

        context = {'posts': posts, 'year': year, 'month': month}
        return render(request, self.template_name, context)
```

สังเกตว่า **ทั้ง signature ของ method และ logic ภายในเหมือนกับ FBV เดิมทุกประการ** สิ่ง
เดียวที่เปลี่ยนคือ: (1) ต้องมี `self` เป็นพารามิเตอร์แรกเสมอ (มาตรฐานของ instance method
ใน Python) และ (2) ต้องอยู่ภายใน method ที่ชื่อ `get` แทนที่จะเป็นชื่อฟังก์ชันอิสระ

### 207.5 อัปเดต `urls.py` ให้ใช้ CBV แทน

```python
# blog/urls.py
from django.urls import path, register_converter
from . import converters
from .views import BlogListView, BlogDetailView, BlogArchiveView

register_converter(converters.FourDigitYearConverter, 'yyyy')
register_converter(converters.TwoDigitMonthConverter, 'mm')

app_name = 'blog'

urlpatterns = [
    path('', BlogListView.as_view(), name='list'),
    path('archive/<yyyy:year>/', BlogArchiveView.as_view(), name='archive_year'),
    path('archive/<yyyy:year>/<mm:month>/', BlogArchiveView.as_view(), name='archive_month'),
    path('<slug:slug>/', BlogDetailView.as_view(), name='detail'),
]
```

สังเกตว่ากฎการเรียงลำดับ URL pattern จากเฉพาะเจาะจงไปหาทั่วไป (Part 006 ขั้นตอนที่ 51.6)
และการใช้ custom path converter (Part 006 ขั้นตอนที่ 57) **ยังใช้ได้เหมือนเดิมทุกประการ**
เพราะ `urls.py` ไม่สนใจว่า view ที่อยู่เบื้องหลังเป็น FBV หรือ CBV ตามที่อธิบายไว้ใน
ขั้นตอนที่ 201.3

### 207.6 ตารางเทียบโค้ดคู่กันแบบเต็ม (FBV → CBV)

| ส่วนประกอบ | FBV (Part 007) | CBV (Part 021) |
|---|---|---|
| นิยาม view | `def blog_list(request):` | `class BlogListView(View):` + `def get(self, request):` |
| พารามิเตอร์แรก | `request` | `self` แล้วตามด้วย `request` |
| การเรียกใน `urls.py` | `views.blog_list` | `BlogListView.as_view()` |
| การรับพารามิเตอร์จาก URL | `def blog_detail(request, slug):` | `def get(self, request, slug):` |
| Logic ภายในฟังก์ชัน/method | เหมือนกันทุกประการ | เหมือนกันทุกประการ |
| การจัดการ method ที่ไม่รองรับ | ต้อง `if/elif/else` เอง | `dispatch()` จัดการให้อัตโนมัติ |

### 207.7 คำถามสำคัญ: ควรแปลง `blog_list`/`blog_detail` จริง ๆ ในโปรเจกต์ตอนนี้ไหม

**ไม่จำเป็น** ในขั้นตอนนี้ Part นี้แปลงให้ดูเป็น **แบบฝึกหัดเพื่อความเข้าใจกลไก** เท่านั้น
ในทางปฏิบัติ เมื่อเรียน Part 022 (Generic `ListView`/`DetailView`) แล้ว เราจะแปลง
`BlogListView`/`BlogDetailView` เหล่านี้อีกครั้งให้สั้นลงอย่างมากโดยใช้ Generic CBV แทน
(ไม่ใช่ `View` เปล่า ๆ แบบใน Part นี้) — Part นี้จึงเป็น **สะพานเชื่อม** ระหว่าง FBV ที่คุ้น
เคยกับ Generic CBV ที่จะเรียนต่อไป ให้เห็นว่ากลไกภายในไม่มีอะไรลึกลับ

---

## ขั้นตอนที่ 208: `@method_decorator` — ใช้ decorator แบบ FBV กับ CBV method

### 208.1 ปัญหา: decorator ของ Python ใช้กับ instance method ตรง ๆ ไม่ได้ทันที

Decorator อย่าง `@require_POST` ที่เรียนใน Part 007 ขั้นตอนที่ 68 ถูกออกแบบมาสำหรับ
ฟังก์ชันที่รับ `request` เป็นพารามิเตอร์แรก:

```python
from django.views.decorators.http import require_POST


@require_POST
def like_post(request, slug):
    ...
```

แต่ instance method ของ class มีพารามิเตอร์แรกเป็น `self` ไม่ใช่ `request`:

```python
# ❌ ผิด: จะพังเพราะ require_POST คาดหวังพารามิเตอร์แรกเป็น request ไม่ใช่ self
class LikePostView(View):
    @require_POST
    def post(self, request, slug):
        ...
```

ถ้าลองรันโค้ดข้างบนจริง ๆ จะไม่ error ทันทีตอน import แต่จะทำงานผิดพลาดตอนเรียกจริง เพราะ
`require_POST` จะไปตรวจสอบ `self` (ซึ่งเป็น instance ของ `LikePostView`) ว่ามี
`.method == 'POST'` หรือไม่ แทนที่จะตรวจสอบ `request` — ผลลัพธ์คือ logic จะทำงานผิดเพี้ยน
โดยไม่ raise error ให้เห็นชัดเจน (bug ประเภทที่ debug ยากมาก)

### 208.2 `method_decorator`: ตัวแปลง decorator ให้ใช้กับ method ได้

Django มี `django.utils.decorators.method_decorator` ที่ทำหน้าที่ "ห่อ" decorator แบบ
FBV ให้ใช้กับ method ของ class ได้อย่างถูกต้อง:

```python
from django.utils.decorators import method_decorator
from django.views.decorators.http import require_POST
from django.views import View
from django.shortcuts import get_object_or_404, redirect
from .models import Post


class LikePostView(View):
    @method_decorator(require_POST)
    def post(self, request, slug):
        post = get_object_or_404(Post, slug=slug)
        # ... เพิ่มจำนวน like (จะเรียนเรื่อง F() expression ใน Part 014) ...
        return redirect('blog:detail', slug=slug)
```

`method_decorator` ทำงานโดย **สลับตำแหน่งการห่อ**: มันเข้าใจว่า argument ตัวแรกที่ decorator
ดั้งเดิมคาดหวัง (`request`) จริง ๆ แล้วอยู่ที่ตำแหน่งที่สองของ method (หลัง `self`) จึงปรับ
ให้ decorator ทำงานถูกจุดโดยอัตโนมัติ

### 208.3 การใช้ `method_decorator` บน `dispatch()` เพื่อครอบคลุมทุก HTTP method

ถ้าต้องการให้ decorator มีผลกับ **ทุก method** ของ class (ทั้ง `get`, `post` ฯลฯ) ให้ apply
ที่ `dispatch()` แทนที่จะ apply ทีละ method:

```python
from django.utils.decorators import method_decorator
from django.views.decorators.csrf import csrf_exempt
from django.views import View


@method_decorator(csrf_exempt, name='dispatch')
class WebhookView(View):
    """
    ทุก method (get, post, put, ...) ของ view นี้จะถูกยกเว้น CSRF check
    เหมาะกับ webhook receiver เช่นเดียวกับตัวอย่างใน Part 007 ขั้นตอนที่ 68.6
    (คำเตือนด้านความปลอดภัยเดียวกันยังใช้ได้: ต้องมีการยืนยันตัวตนทดแทนเสมอ)
    """
    def post(self, request):
        from django.http import JsonResponse
        return JsonResponse({'received': True})
```

สังเกตว่าตัวอย่างนี้ apply `@method_decorator(..., name='dispatch')` **บน class เอง**
(ไม่ใช่บน method) พร้อมระบุ `name='dispatch'` เพื่อบอกว่าให้ไปแปะ decorator ไว้ที่
`dispatch()` ของ class นั้น นี่คือรูปแบบที่ Django แนะนำในเอกสารทางการสำหรับกรณีที่ต้องการ
ให้ decorator มีผลกับทุก HTTP method พร้อมกัน โดยไม่ต้องเขียน `@method_decorator(...)`
ซ้ำหน้าทุก method

### 208.4 เปรียบเทียบ: apply ที่ method เดียว vs apply ที่ `dispatch()`

```python
# แบบที่ 1: apply เฉพาะ method เดียว (เมื่อต้องการผลเฉพาะ method นั้น)
class ExampleViewA(View):
    @method_decorator(require_POST)
    def post(self, request):
        ...

    def get(self, request):
        # get() ไม่ถูกครอบด้วย require_POST เลย
        ...


# แบบที่ 2: apply ที่ dispatch() ผ่าน class decorator (มีผลกับทุก method)
@method_decorator(never_cache, name='dispatch')
class ExampleViewB(View):
    def get(self, request):
        # ถูกครอบด้วย never_cache ด้วย
        ...

    def post(self, request):
        # ถูกครอบด้วย never_cache ด้วยเช่นกัน
        ...
```

### 208.5 การซ้อน decorator หลายตัวบน class เดียวกัน

```python
from django.utils.decorators import method_decorator
from django.views.decorators.cache import never_cache
from django.views.decorators.http import require_http_methods


@method_decorator(never_cache, name='dispatch')
@method_decorator(require_http_methods(['GET', 'POST']), name='dispatch')
class SecureFormView(View):
    def get(self, request):
        ...

    def post(self, request):
        ...
```

หลักการทำงานจากบนลงล่างเหมือน decorator ปกติของ Python (decorator ที่อยู่ **ใกล้ class
มากที่สุด** ทำงานก่อน) เช่นเดียวกับที่เรียนเรื่องการซ้อน decorator บน FBV ใน Part 007
ขั้นตอนที่ 68.5

### 208.6 ตารางสรุป `method_decorator`

| การใช้งาน | Syntax | ผลลัพธ์ |
|---|---|---|
| Apply กับ method เดียว | `@method_decorator(decorator)` เหนือ method | มีผลเฉพาะ method นั้น |
| Apply กับทุก HTTP method | `@method_decorator(decorator, name='dispatch')` เหนือ class | มีผลกับทุก method เพราะครอบที่ `dispatch()` |
| Apply กับ method ที่ระบุชื่อ | `@method_decorator(decorator, name='get')` เหนือ class | มีผลเฉพาะ method ที่ระบุ (ทางเลือกแทนการแปะเหนือ method โดยตรง) |
| ใช้กับ decorator ที่รับ argument | `@method_decorator(require_http_methods(['GET']))` | ทำงานเหมือน decorator ปกติที่รับ argument |

---

## ขั้นตอนที่ 209: เชื่อม CBV เข้า `urls.py` ด้วย `.as_view()` และส่ง initial kwargs

### 209.1 ทบทวน: `.as_view()` ต้องถูกเรียกเสมอใน `urls.py`

จากขั้นตอนที่ 202 เราเข้าใจแล้วว่า `path()` ต้องการ **callable** ไม่ใช่ class ดังนั้น
**ต้องเรียก `.as_view()` เสมอ** เมื่อเชื่อม CBV เข้า `urls.py` — นี่คือความผิดพลาดที่พบบ่อย
ที่สุดของมือใหม่:

```python
# ❌ ผิด: ลืมเรียก .as_view() — ส่ง class ตรง ๆ เข้า path()
urlpatterns = [
    path('', BlogListView, name='list'),
]
```

```python
# ✅ ถูก: เรียก .as_view() เสมอ
urlpatterns = [
    path('', BlogListView.as_view(), name='list'),
]
```

ถ้าลืม `.as_view()` Django จะไม่ error ทันทีตอน `runserver` แต่จะ error ตอนมี request
เข้ามาจริงด้วยข้อความประมาณ `TypeError: __init__() takes 1 positional argument but 2 were
given` (เพราะ Django พยายามเรียก `BlogListView(request)` ตรง ๆ ซึ่งไปตรงกับ `__init__` ของ
class แทนที่จะเป็น `dispatch()` ที่ถูกต้อง)

### 209.2 ส่ง initial kwargs ผ่าน `as_view(param=value)`

ย้อนกลับไปที่ขั้นตอนที่ 202.6 เราเห็นแล้วว่า `as_view(**initkwargs)` รับ keyword argument
ได้ และค่านั้นจะถูกนำไปตั้งเป็น attribute ของ instance ผ่าน `__init__`:

```python
class View:
    def __init__(self, **kwargs):
        for key, value in kwargs.items():
            setattr(self, key, value)
```

นี่เปิดโอกาสให้ **ใช้ view class เดียวกันซ้ำได้หลาย URL โดยพฤติกรรมต่างกัน** เพียงส่ง
kwargs ต่างกันตอนเรียก `.as_view()`:

```python
# blog/views.py
from django.views.generic import TemplateView


class InfoPageView(TemplateView):
    template_name = 'blog/info_page.html'
    page_heading = 'ข้อมูลทั่วไป'   # ค่า default

    def get_context_data(self, **kwargs):
        context = super().get_context_data(**kwargs)
        context['heading'] = self.page_heading
        return context
```

```python
# blog/urls.py
from .views import InfoPageView

urlpatterns = [
    # ใช้ view class เดียวกัน แต่ตั้งค่า page_heading ต่างกันในแต่ละ URL
    path('privacy/', InfoPageView.as_view(page_heading='นโยบายความเป็นส่วนตัว'), name='privacy'),
    path('terms/', InfoPageView.as_view(page_heading='ข้อตกลงการใช้งาน'), name='terms'),
    path('faq/', InfoPageView.as_view(page_heading='คำถามที่พบบ่อย'), name='faq'),
]
```

เมื่อเข้า `/privacy/` `self.page_heading` จะเป็น `'นโยบายความเป็นส่วนตัว'` ในขณะที่เข้า
`/terms/` `self.page_heading` จะเป็น `'ข้อตกลงการใช้งาน'` **ทั้งที่เป็น class เดียวกัน**
เพราะ `as_view(page_heading=...)` ถูกเรียกแยกกันในแต่ละบรรทัดของ `urlpatterns` (แต่ละครั้ง
ที่เรียก `.as_view()` จะได้ closure/`initkwargs` เป็นของตัวเอง)

### 209.3 ตัวอย่างจริงที่ใช้บ่อยที่สุด: ปรับ `template_name` ต่อ URL

```python
# blog/urls.py
from django.views.generic import TemplateView

urlpatterns = [
    path(
        'success/',
        TemplateView.as_view(template_name='blog/generic_message.html'),
    ),
]
```

นี่คือเหตุผลที่ `TemplateView` เกือบไม่ต้องเขียน subclass เองเลยในหลายกรณี — ส่ง
`template_name` เข้า `as_view()` ตรง ๆ ได้เลยโดยไม่ต้องสร้าง class ใหม่แม้แต่ตัวเดียว
(ใช้ `TemplateView` เปล่า ๆ จาก `django.views.generic` ได้ทันที)

### 209.4 ข้อจำกัดสำคัญ: kwargs ต้องเป็น attribute ที่มีอยู่แล้วในคลาสเท่านั้น

ย้อนกลับไปดูการตรวจสอบใน `as_view()` จากขั้นตอนที่ 202.6 อีกครั้ง: `as_view()` จะ
`raise TypeError` ถ้า kwargs ที่ส่งเข้ามาไม่ใช่ attribute ที่มีอยู่แล้วบน class:

```python
>>> InfoPageView.as_view(page_heading='ok')      # ผ่าน เพราะ page_heading เป็น class attribute
>>> InfoPageView.as_view(random_typo='oops')     # TypeError! ไม่มี attribute นี้ใน class
```

**กฎเหล็กของหลักสูตรนี้: ทุก parameter ที่จะส่งผ่าน `as_view(key=value)` ต้องถูกประกาศเป็น
class attribute ไว้ล่วงหน้าเสมอ (แม้จะเป็นค่า default ธรรมดาก็ตาม) เพื่อให้ Django ตรวจจับ
การพิมพ์ชื่อผิดได้ตั้งแต่ตอนพัฒนา** แทนที่จะปล่อยให้ error ไปโผล่ตอน production

### 209.5 ตารางสรุปวิธีเชื่อม CBV เข้า `urls.py`

| รูปแบบ | ตัวอย่าง | ใช้เมื่อ |
|---|---|---|
| `.as_view()` เปล่า | `path('', PostListView.as_view())` | กรณีทั่วไปที่ไม่ต้องปรับค่าอะไรเพิ่ม |
| `.as_view(param=value)` | `path('a/', View.as_view(template_name='a.html'))` | ต้องการปรับ attribute บางตัวแยกตาม URL โดยไม่สร้าง subclass ใหม่ |
| ใช้ class เดียวกันหลาย URL | `path('privacy/', V.as_view(x=1))`, `path('terms/', V.as_view(x=2))` | ต้องการ reuse view class เดียวกันแต่ให้พฤติกรรมต่างกันเล็กน้อย |
| ลืม `.as_view()` | `path('', PostListView)` | **ห้ามทำ** — จะ error ตอนมี request เข้ามาจริง |

---

## ขั้นตอนที่ 210: สรุปและแบบฝึกหัด

### 210.1 ประกอบร่างทุกอย่างเข้าด้วยกัน: `blog/views.py` ฉบับสมบูรณ์ของ Part นี้

```python
# blog/views.py
from django.shortcuts import render, get_object_or_404, redirect
from django.utils.decorators import method_decorator
from django.views import View
from django.views.decorators.http import require_POST
from django.views.generic import TemplateView, RedirectView
from .models import Post


class BlogListView(View):
    """CBV แบบ manual เทียบเท่า blog_list() จาก Part 007 (ขั้นตอนที่ 207)"""
    template_name = 'blog/post_list.html'

    def get(self, request):
        posts = Post.objects.filter(is_published=True).order_by('-created_at')
        return render(request, self.template_name, {'posts': posts})


class BlogDetailView(View):
    """CBV แบบ manual เทียบเท่า blog_detail() จาก Part 007 (ขั้นตอนที่ 207)"""
    template_name = 'blog/post_detail.html'

    def get(self, request, slug):
        post = get_object_or_404(Post, slug=slug, is_published=True)
        return render(request, self.template_name, {'post': post})


class BlogArchiveView(View):
    """CBV แบบ manual เทียบเท่า blog_archive() จาก Part 007 (ขั้นตอนที่ 207)"""
    template_name = 'blog/post_archive.html'

    def get(self, request, year, month=None):
        posts = Post.objects.filter(is_published=True, created_at__year=year)
        if month is not None:
            posts = posts.filter(created_at__month=month)
        posts = posts.order_by('-created_at')
        context = {'posts': posts, 'year': year, 'month': month}
        return render(request, self.template_name, context)


class LikePostView(View):
    """สาธิต method_decorator กับ HTTP method เดียว (ขั้นตอนที่ 208)"""

    @method_decorator(require_POST)
    def post(self, request, slug):
        post = get_object_or_404(Post, slug=slug)
        # จะใช้ F() expression จริงเมื่อเรียน Part 014
        return redirect('blog:detail', slug=slug)


class HomePageView(TemplateView):
    """สาธิต TemplateView + get_context_data (ขั้นตอนที่ 204)"""
    template_name = 'blog/home.html'

    def get_context_data(self, **kwargs):
        context = super().get_context_data(**kwargs)
        context['latest_posts'] = Post.objects.filter(is_published=True)[:5]
        context['total_published'] = Post.objects.filter(is_published=True).count()
        return context


class OldArticlesRedirectView(RedirectView):
    """สาธิต RedirectView (ขั้นตอนที่ 205)"""
    pattern_name = 'blog:list'
    permanent = True
```

```python
# blog/urls.py
from django.urls import path, register_converter
from . import converters
from .views import (
    BlogListView,
    BlogDetailView,
    BlogArchiveView,
    LikePostView,
    HomePageView,
    OldArticlesRedirectView,
)

register_converter(converters.FourDigitYearConverter, 'yyyy')
register_converter(converters.TwoDigitMonthConverter, 'mm')

app_name = 'blog'

urlpatterns = [
    path('', BlogListView.as_view(), name='list'),
    path('home/', HomePageView.as_view(), name='home'),
    path('archive/<yyyy:year>/', BlogArchiveView.as_view(), name='archive_year'),
    path('archive/<yyyy:year>/<mm:month>/', BlogArchiveView.as_view(), name='archive_month'),
    path('old-articles/', OldArticlesRedirectView.as_view(), name='old_articles'),
    path('<slug:slug>/like/', LikePostView.as_view(), name='like'),
    path('<slug:slug>/', BlogDetailView.as_view(), name='detail'),
]
```

ทดสอบด้วย `python manage.py runserver` แล้วเข้า URL ต่าง ๆ เพื่อยืนยันว่าทำงานเหมือนกับ
เวอร์ชัน FBV ของ Part 007 ทุกประการ:

```
http://127.0.0.1:8000/posts/
http://127.0.0.1:8000/posts/home/
http://127.0.0.1:8000/posts/archive/2026/
http://127.0.0.1:8000/posts/old-articles/    (ควร redirect แบบ 301 ไปที่ /posts/)
```

### 210.2 สรุปสิ่งที่ได้เรียนรู้ใน Part นี้

- ✅ เข้าใจว่า CBV ทุกตัวสืบทอดจาก `django.views.generic.base.View` และเหตุผลที่ Django
  มีทั้ง FBV และ CBV ควบคู่กันโดยไม่ทดแทนกันทั้งหมด
- ✅ เจาะลึกกลไกของ `as_view()`: มันคืนฟังก์ชัน (ไม่ใช่ class) และสร้าง instance ใหม่ทุก
  ครั้งที่มี request เข้ามา (thread safety)
- ✅ เข้าใจ `dispatch()` ว่าใช้ `getattr()` เลือก method ตาม `request.method` แทนการเขียน
  `if/elif` เอง และเข้าใจว่าทำไม method ที่ไม่ได้นิยามไว้จะได้ 405 อัตโนมัติ
- ✅ ใช้ `TemplateView` พร้อม `get_context_data()` และเข้าใจความสำคัญของการเรียก `super()`
  ก่อนเสมอ
- ✅ ใช้ `RedirectView` แทนการเขียน view สำหรับ redirect เอง และรู้ว่า `permanent` มี
  default ต่างจาก `redirect()` shortcut
- ✅ override `http_method_names` เพื่อบล็อก method บางตัวไว้ตั้งแต่ระดับ class
- ✅ แปลง FBV `blog_list`/`blog_detail`/`blog_archive` จาก Part 007 ให้เป็น CBV แบบ manual
  ได้ด้วยตัวเอง
- ✅ ใช้ `@method_decorator` ทั้งแบบครอบ method เดียวและครอบ `dispatch()` ทั้งหมดผ่าน
  `name='dispatch'`
- ✅ เชื่อม CBV เข้า `urls.py` ด้วย `.as_view()` อย่างถูกต้อง และส่ง initial kwargs ผ่าน
  `as_view(param=value)` เพื่อ reuse view class เดียวกันในหลาย URL

### 210.3 Checklist ก่อนไป Part ถัดไป

- [ ] อธิบายได้ว่าทำไม `PostListView.as_view()` คืนฟังก์ชัน ไม่ใช่ class
- [ ] อธิบายได้ว่า instance ของ CBV ถูกสร้างใหม่กี่ครั้งเมื่อมี request 100 ครั้งเข้ามา
- [ ] เขียน CBV ที่มีทั้ง `get()` และ `post()` แยกกันได้โดยไม่ต้องเขียน `if/elif`
- [ ] เขียน `TemplateView` ที่ override `get_context_data()` พร้อมเรียก `super()` ถูกต้อง
- [ ] ใช้ `RedirectView` แทน view redirect ที่เขียนเองได้อย่างน้อย 1 จุด
- [ ] override `http_method_names` เพื่อจำกัด method ของ view ได้
- [ ] แปลง `blog_list`, `blog_detail`, `blog_archive` เป็น CBV แบบ manual สำเร็จและทดสอบ
  ผ่าน `runserver` แล้วว่าทำงานเหมือนเดิม
- [ ] ใช้ `@method_decorator(require_POST)` กับ method `post()` ของ CBV ได้ถูกต้อง
- [ ] ใช้ `as_view(param=value)` ส่งค่าเริ่มต้นให้ view class เดียวกันทำงานต่างกันในหลาย URL

### 210.4 แบบฝึกหัดท้ายบท

**แบบฝึกหัดที่ 1 (พื้นฐาน)**: เขียน CBV ชื่อ `RequestDebugView` (สืบทอดจาก `View` ตรง ๆ)
ที่มี method `get()` แสดง `request.method`, `request.path`, และ `self.__class__.__name__`
เป็น HTML (คล้ายกับ `post_debug` ใน Part 007 แบบฝึกหัดที่ 1 แต่เขียนเป็น CBV) แล้วเชื่อมเข้า
`urls.py` ที่ path `/posts/cbv-debug/`

**แบบฝึกหัดที่ 2 (ประยุกต์)**: เขียน CBV ชื่อ `BlogSearchView` ที่มี `get()` รับ query
parameter `q` แล้วค้นหาบทความด้วย `title__icontains` (เทียบเท่า `blog_search` จาก Part 007
แบบฝึกหัดที่ 2) จากนั้นเพิ่ม `post()` ให้ view เดียวกันที่รับค่า `q` จากฟอร์ม POST แทน
query string แล้วทดสอบว่าทั้งสอง method ทำงานถูกต้องด้วย `RequestFactory` ใน
`python manage.py shell`

**แบบฝึกหัดที่ 3 (Decorator + Security)**: แปลง `blog_publish_toggle` จาก Part 007
แบบฝึกหัดที่ 3 ให้เป็น CBV ชื่อ `PostPublishToggleView` โดยใช้ `@method_decorator(require_POST)`
กับ method `post()` ทดสอบว่าเมื่อเข้าด้วย `GET` จะได้ **HTTP 405** เหมือนกับพฤติกรรมเดิม
ของ FBV ทุกประการ

**แบบฝึกหัดที่ 4 (ขั้นสูง)**: สร้าง CBV ชื่อ `LegalPageView` ที่สืบทอดจาก `TemplateView`
มี class attribute `template_name = 'blog/legal_page.html'` และ `page_slug = None` แล้วเขียน
`get_context_data()` ให้ส่ง `page_slug` เข้า context จากนั้นเชื่อมเข้า `urls.py` สาม URL
(`/privacy/`, `/terms/`, `/cookies/`) โดยใช้ `LegalPageView.as_view(page_slug=...)` ค่าต่างกัน
ในแต่ละ URL (เทคนิคเดียวกับขั้นตอนที่ 209.2) แล้วให้ template แสดงข้อความต่างกันตาม
`{% if page_slug == "privacy" %}...{% endif %}`

### 210.5 คำถามที่พบบ่อย (FAQ)

**Q: ควรใช้ CBV หรือ FBV ในโปรเจกต์จริง แยกยังไงว่าตอนไหนควรใช้อะไร?**
A: หลักการเดียวกับที่สรุปไว้ใน Part 007 ขั้นตอนที่ 69.5 ยังใช้ได้เต็มที่: ใช้ Generic CBV
(ที่จะเรียนใน Part 022-023) สำหรับงาน CRUD มาตรฐาน และใช้ FBV หรือ `View` เปล่า ๆ (แบบใน
Part นี้) สำหรับ logic เฉพาะตัวที่ไม่ตรง pattern สำเร็จรูป ไม่มีกฎตายตัวว่าต้องเลือกอย่างใด
อย่างหนึ่งทั้งโปรเจกต์ ทีมมืออาชีพส่วนใหญ่ใช้ทั้งสองแบบผสมกันตามความเหมาะสมของแต่ละ view

**Q: ทำไม `self` ต้องเป็นพารามิเตอร์แรกของ `get()`/`post()` เสมอ ลืมใส่ได้ไหม?**
A: `self` เป็นข้อกำหนดพื้นฐานของ Python เองสำหรับ instance method ไม่ใช่กฎเฉพาะของ Django
เมื่อ Python เรียก `instance.get(request)` เบื้องหลังจริง ๆ คือการเรียก
`ClassName.get(instance, request)` — ถ้าลืมใส่ `self` ตอนนิยาม method จะเกิด
`TypeError: get() takes 1 positional argument but 2 were given` ทันทีที่ Django พยายาม
เรียก method นั้น (เพราะ `request` จะถูกส่งเข้าไปในตำแหน่งที่ Python คาดหวังว่าเป็น `self`)

**Q: `TemplateView` กับการเขียน `View` ที่มี `get()` เรียก `render()` เองต่างกันจริง ๆ
แค่ไหน ควรใช้ `TemplateView` เสมอไหม?**
A: สำหรับ view ที่ **แค่ render template อย่างเดียวไม่มี logic พิเศษ** ควรใช้
`TemplateView` เสมอเพราะสั้นกว่าและสื่อเจตนาชัดเจนกว่า (คนอ่านโค้ดเห็น `TemplateView` แล้ว
รู้ทันทีว่า view นี้ไม่มีอะไรซับซ้อน) แต่ถ้า view มี logic ที่ไม่ใช่แค่เตรียม context
(เช่น ต้องตรวจสอบ permission แบบพิเศษ, ต้องแตกแขนง response เป็นหลายรูปแบบ) การเขียน `View`
เปล่า ๆ แล้วควบคุมเองทั้งหมดอาจอ่านง่ายกว่าในระยะยาว

**Q: ทำไม `RedirectView.permanent` ถึง default เป็น `True` ในขณะที่ `redirect()` shortcut
default เป็น `False` ดูขัดแย้งกันแปลก ๆ?**
A: นี่เป็นความไม่สอดคล้องกันในประวัติศาสตร์ของ Django API ที่หลายคนก็มองว่าน่าสับสน (ทั้งสอง
ส่วนถูกออกแบบและเพิ่มเข้ามาคนละช่วงเวลากัน) ในทางปฏิบัติ **ควรระบุ `permanent=` ให้ชัดเจน
เสมอไม่ว่าจะใช้เครื่องมือไหน** อย่าพึ่งพา default เพราะผลกระทบของการตั้งผิด (โดยเฉพาะ 301
ที่ browser cache ไว้อย่างจริงจัง) รุนแรงกว่าความสะดวกที่ประหยัดได้จากการไม่ต้องพิมพ์
พารามิเตอร์นี้

**Q: `@method_decorator(decorator, name='dispatch')` กับการ override `dispatch()` เองแล้ว
เรียก decorator ข้างในต่างกันยังไง?**
A: ทั้งสองวิธีให้ผลลัพธ์เหมือนกันในทางปฏิบัติ แต่ `@method_decorator(..., name='dispatch')`
สั้นกว่าและเป็นรูปแบบที่เอกสารทางการของ Django แนะนำสำหรับ decorator ที่มีอยู่แล้ว (เช่น
`csrf_exempt`, `never_cache`) ส่วนการ override `dispatch()` เองเหมาะกับกรณีที่ต้องมี logic
เพิ่มเติมที่ซับซ้อนกว่าแค่ apply decorator ตรง ๆ เช่น ต้องเช็คเงื่อนไขบางอย่างก่อนตัดสินใจ
ว่าจะเรียก `super().dispatch()` ต่อหรือไม่ (ตัวอย่างแบบนี้จะเห็นชัดเจนเมื่อเรียนเรื่อง
`LoginRequiredMixin` ใน Part 024)

---

## เตรียมตัวสำหรับ Part ถัดไป

**Part 022: Generic Class-Based Views: ListView, DetailView** จะนำกลไกทั้งหมดที่เรียนใน
Part นี้ไปต่อยอดกับ Generic CBV ตัวที่ใช้บ่อยที่สุดสองตัวของ Django ได้แก่ `ListView` และ
`DetailView` คุณจะได้เห็นว่า `BlogListView` และ `BlogDetailView` ที่เขียนแบบ manual ไว้ใน
ขั้นตอนที่ 207 ของ Part นี้ สามารถย่อให้สั้นลงเหลือเพียงไม่กี่บรรทัดได้อย่างไรโดยใช้
`model`, `context_object_name`, `paginate_by`, `get_queryset()` และ `slug_field` ที่
`ListView`/`DetailView` เตรียมไว้ให้ พร้อมเจาะลึกว่า mixin เบื้องหลังทั้งสองตัวนี้ทำงาน
ร่วมกับ `dispatch()`, `get_context_data()`, และ `TemplateResponseMixin` ที่เรียนไปแล้วใน
Part นี้อย่างไร

เตรียมเปิด `blog/views.py`, `blog/urls.py`, และ `blog/models.py` ของคุณไว้ให้พร้อม เพราะ
Part 022 จะแก้ไขไฟล์เหล่านี้ต่อจากที่ Part 021 ทิ้งไว้ทันที!
