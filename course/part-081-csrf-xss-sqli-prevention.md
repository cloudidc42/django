# Part 081: CSRF, XSS และ SQL Injection Prevention

## ขั้นตอนที่ 801-810

---

## ภาพรวม

Part นี้เป็น **configuration reference** สำหรับกลไกป้องกันที่ Django มีให้ในตัว
ครอบคลุม 3 ประเภทหลัก:

1. **CSRF Protection** — `CsrfViewMiddleware` และ settings ที่เกี่ยวข้อง
2. **XSS Prevention** — Django Template Engine's autoescape system และ `mark_safe()`/`|safe`
3. **SQL Injection Prevention** — ORM parameterized queries, `raw()`, `extra()`

แต่ละหัวข้อนำเสนอในรูปแบบ **API reference และ configuration guide** เพื่อให้
นำไปใช้ได้ทันทีในโปรเจกต์จริง

---

## ขั้นตอนที่ 801: CSRF Protection — CsrfViewMiddleware

### กลไกการทำงาน (ระดับ Configuration)

Django's CSRF protection ทำงานผ่าน **Synchronizer Token Pattern**:

1. Server สร้าง random token และเก็บใน cookie (`csrftoken`)
2. Template ใส่ token เดิมใน hidden form field (`csrfmiddlewaretoken`)
3. เมื่อ submit form → Server เปรียบเทียบ cookie value กับ form field value
4. ถ้าไม่ตรงกัน → ตอบกลับ 403 Forbidden

```
Browser                           Django Server
  │                                     │
  │── GET /form/ ──────────────────────►│
  │                                     │── สร้าง CSRF token
  │◄── Response (cookie: csrftoken) ────│
  │    + form: <input name=csrfmiddlewaretoken>
  │                                     │
  │── POST /form/ (cookie + form field)►│
  │                                     │── เปรียบเทียบ token
  │◄── Response ────────────────────────│
```

### CsrfViewMiddleware Configuration

```python
# settings/base.py

MIDDLEWARE = [
    'django.middleware.security.SecurityMiddleware',
    'django.contrib.sessions.middleware.SessionMiddleware',
    'django.middleware.common.CommonMiddleware',
    
    # CsrfViewMiddleware ต้องอยู่ก่อน views ทั้งหมด
    # แต่หลัง SessionMiddleware
    'django.middleware.csrf.CsrfViewMiddleware',
    
    'django.contrib.auth.middleware.AuthenticationMiddleware',
    'django.contrib.messages.middleware.MessageMiddleware',
    'django.middleware.clickjacking.XFrameOptionsMiddleware',
]
```

### CSRF Settings Reference

| Setting | Default | คำอธิบาย |
|---------|---------|---------|
| `CSRF_COOKIE_NAME` | `'csrftoken'` | ชื่อ cookie |
| `CSRF_COOKIE_AGE` | `31449600` (1 ปี, วินาที) | อายุ cookie |
| `CSRF_COOKIE_DOMAIN` | `None` | Domain ของ cookie |
| `CSRF_COOKIE_PATH` | `'/'` | Path ของ cookie |
| `CSRF_COOKIE_SECURE` | `False` | ส่งเฉพาะ HTTPS (ตั้ง True ใน production) |
| `CSRF_COOKIE_HTTPONLY` | `False` | ห้าม JS เข้าถึง (ต้อง False เพื่อให้ Fetch API ทำงาน) |
| `CSRF_COOKIE_SAMESITE` | `'Lax'` | SameSite policy |
| `CSRF_COOKIE_MASKED` | `False` | Mask token สำหรับ per-request token (Django 5.x) |
| `CSRF_HEADER_NAME` | `'HTTP_X_CSRFTOKEN'` | Header name สำหรับ AJAX requests |
| `CSRF_TRUSTED_ORIGINS` | `[]` | Origins ที่เชื่อถือ (จำเป็นสำหรับ cross-origin) |
| `CSRF_USE_SESSIONS` | `False` | เก็บ token ใน session แทน cookie |
| `CSRF_FAILURE_VIEW` | (Django built-in) | View ที่แสดงเมื่อ CSRF fail |

### CSRF_TRUSTED_ORIGINS

ตั้งแต่ Django 4.0 เป็นต้นมา Django ตรวจสอบ `Origin` header
`CSRF_TRUSTED_ORIGINS` ใช้สำหรับกรณีที่ frontend และ backend อยู่คนละ domain

```python
# settings/production.py

CSRF_TRUSTED_ORIGINS = [
    'https://www.example.com',
    'https://api.example.com',
    # Note: ต้องเป็น full origin รวม scheme (https://)
    # ตั้งแต่ Django 4.0 เป็นต้นไป
]

# ตั้งแต่ Django 4.1: รองรับ subdomain wildcard
CSRF_TRUSTED_ORIGINS = [
    'https://*.example.com',   # ทุก subdomain ของ example.com
    'https://example.com',
]
```

### CSRF_COOKIE_SECURE และ CSRF_USE_SESSIONS

```python
# settings/production.py

# เปิดใช้ HTTPS-only cookie
CSRF_COOKIE_SECURE = True   # default: False

# ทางเลือก: เก็บ CSRF token ใน session แทน cookie
# ข้อดี: ซ่อนจาก JavaScript ทั้งหมด (รวมถึง first-party JS)
# ข้อเสีย: ต้องใช้ server-side session → overhead เพิ่มขึ้น
CSRF_USE_SESSIONS = True    # default: False
# ถ้าตั้ง True → CSRF_COOKIE_* settings ไม่มีผล
```

---

## ขั้นตอนที่ 802: CSRF Protection ใน Templates

### {% csrf_token %} Tag

Django Template Language มี built-in tag สำหรับ CSRF:

```django
{# Template: blog/post_create.html #}
<form method="post" action="{% url 'blog:post-create' %}">
    {% csrf_token %}
    {# Django render เป็น: #}
    {# <input type="hidden" name="csrfmiddlewaretoken" value="<token>"> #}
    
    {{ form.as_p }}
    <button type="submit">สร้างบทความ</button>
</form>
```

### CSRF สำหรับ AJAX Requests

สำหรับ Fetch API หรือ XMLHttpRequest ต้องส่ง CSRF token ใน header:

```javascript
// วิธีที่ 1: อ่านจาก cookie (แนะนำสำหรับ Single-Page Application)
function getCookie(name) {
    let cookieValue = null;
    if (document.cookie && document.cookie !== '') {
        const cookies = document.cookie.split(';');
        for (let i = 0; i < cookies.length; i++) {
            const cookie = cookies[i].trim();
            if (cookie.startsWith(name + '=')) {
                cookieValue = decodeURIComponent(cookie.substring(name.length + 1));
                break;
            }
        }
    }
    return cookieValue;
}

// ใช้ใน fetch
fetch('/api/posts/', {
    method: 'POST',
    headers: {
        'Content-Type': 'application/json',
        'X-CSRFToken': getCookie('csrftoken'),  // ← header name ตรงกับ CSRF_HEADER_NAME
    },
    body: JSON.stringify({ title: 'New Post' }),
});

// วิธีที่ 2: อ่านจาก meta tag (ถ้า CSRF_COOKIE_HTTPONLY = True)
// ใน base template:
// <meta name="csrf-token" content="{{ csrf_token }}">
// ใน JavaScript:
// const token = document.querySelector('meta[name="csrf-token"]').content;
```

### Django REST Framework และ CSRF

DRF ปิด CSRF enforcement สำหรับ `APIView` โดยอัตโนมัติสำหรับ
non-session authentication (เช่น Token, JWT) แต่เปิดสำหรับ `SessionAuthentication`:

```python
# views.py
from rest_framework.views import APIView
from rest_framework.authentication import SessionAuthentication

class MyApiView(APIView):
    # ถ้าใช้ SessionAuthentication → DRF enforce CSRF
    authentication_classes = [SessionAuthentication]
    
    # ถ้าใช้ JWT → ไม่ต้อง CSRF (เพราะ credentials ไม่ถูกส่งอัตโนมัติโดย browser)
    # authentication_classes = [JWTAuthentication]
```

---

## ขั้นตอนที่ 803: @csrf_protect, @csrf_exempt และ @requires_csrf_token

### Decorator Reference

| Decorator | Import | พฤติกรรม |
|-----------|--------|---------|
| `@csrf_protect` | `django.views.decorators.csrf` | บังคับ CSRF check สำหรับ view นี้ (แม้จะปิด middleware) |
| `@csrf_exempt` | `django.views.decorators.csrf` | ยกเว้น CSRF check สำหรับ view นี้ |
| `@requires_csrf_token` | `django.views.decorators.csrf` | ทำให้มี `csrf_token` ใน context แต่ไม่ reject request |
| `@ensure_csrf_cookie` | `django.views.decorators.csrf` | ส่ง CSRF cookie ใน response เสมอ |

```python
# views.py
from django.views.decorators.csrf import (
    csrf_protect,
    csrf_exempt,
    requires_csrf_token,
    ensure_csrf_cookie,
)
from django.utils.decorators import method_decorator
from django.views import View

# ใช้กับ Function-Based View
@csrf_protect
def protected_view(request):
    ...

@csrf_exempt  
def webhook_receiver(request):
    # ✓ Legitimate use case: รับ webhook จาก third-party ที่ส่ง HMAC signature แทน
    # ต้องตรวจสอบ HMAC signature ใน view body แทน
    ...

# ใช้กับ Class-Based View
@method_decorator(csrf_exempt, name='dispatch')
class WebhookView(View):
    def post(self, request):
        # ตรวจสอบ signature จาก header แทน
        signature = request.headers.get('X-Hub-Signature-256', '')
        # ... verify HMAC ...
        ...

# ensure_csrf_cookie — ใช้สำหรับ SPA ที่ต้องการ CSRF cookie
# ก่อนที่จะทำ AJAX request ครั้งแรก
@ensure_csrf_cookie
def spa_entry_point(request):
    return render(request, 'spa/index.html')
```

### CSRF Exempt สำหรับ API Endpoints ที่ใช้ Token Auth

```python
# ถ้าใช้ Django REST Framework กับ Token/JWT Authentication
# CSRF ไม่จำเป็นสำหรับ endpoints เหล่านี้เพราะ:
# 1. Token ไม่ถูกส่งอัตโนมัติโดย browser (ต้อง set header เอง)
# 2. JavaScript malicious ต้องรู้ token ก่อนถึงจะ attach ได้

# DRF จัดการเรื่องนี้อัตโนมัติ แต่ถ้าเขียน pure Django API:
from django.views.decorators.csrf import csrf_exempt
from django.contrib.auth.models import User
from rest_framework_simplejwt.authentication import JWTAuthentication

@csrf_exempt
def token_api_view(request):
    # ตรวจสอบ Authorization: Bearer <token> header
    ...
```

---

## ขั้นตอนที่ 804: XSS Prevention — Django Template Autoescape System

### กลไก Autoescape

Django Template Engine มี autoescape เปิดอยู่เสมอสำหรับ Django templates
ซึ่งหมายความว่า Django จะ escape HTML characters อัตโนมัติก่อน render

**ตารางการ escape:**

| อักขระ | Escaped เป็น |
|--------|-------------|
| `<` | `&lt;` |
| `>` | `&gt;` |
| `'` | `&#x27;` |
| `"` | `&quot;` |
| `&` | `&amp;` |

### Autoescape Settings

```python
# settings/base.py — Template configuration
TEMPLATES = [
    {
        'BACKEND': 'django.template.backends.django.DjangoTemplates',
        'OPTIONS': {
            # autoescape เปิดอยู่โดยอัตโนมัติ ไม่ต้องระบุ
            # ห้ามตั้ง 'autoescape': False ที่ level นี้
        },
    },
]
```

**สำหรับ Jinja2 backend:** autoescape ต้องตั้งค่าเอง:

```python
# settings/base.py — ถ้าใช้ Jinja2
TEMPLATES = [
    {
        'BACKEND': 'django.template.backends.jinja2.Jinja2',
        'DIRS': [BASE_DIR / 'jinja2'],
        'OPTIONS': {
            'environment': 'myproject.jinja2.environment',
            # *** ต้องตั้ง autoescape=True เองเมื่อใช้ Jinja2 ***
        },
    },
]

# myproject/jinja2.py
from jinja2 import Environment

def environment(**options):
    options['autoescape'] = True  # ← สำคัญมาก
    env = Environment(**options)
    return env
```

---

## ขั้นตอนที่ 805: mark_safe() และ |safe Filter — API Reference

### mark_safe()

`django.utils.safestring.mark_safe()` เป็นฟังก์ชันที่บอก Django ว่า string นี้
ปลอดภัยแล้ว ไม่ต้อง escape อีก Django จะ render ค่านั้นตรง ๆ โดยไม่แปลง HTML

```python
# django/utils/safestring.py (source reference)
def mark_safe(s):
    """
    Explicitly mark a string as safe for (HTML) output purposes. The returned
    object can be used everywhere a string is appropriate.
    
    If used on a method as a decorator, mark the returned data as safe.
    
    Can be used in conjunction with string interpolation, as all the string
    methods and operators used on the resulting object will retain the safe
    status of the original argument. For example:
    
        mark_safe("Hello, {0}").format("world")
        
    will return a SafeString.
    """
    ...
```

### ใช้ mark_safe() อย่างถูกต้อง

```python
# ✓ ถูกต้อง: ใช้กับ HTML ที่เราสร้างเอง ไม่ใช่ user input
from django.utils.html import mark_safe, format_html, escape

class PostAdmin(admin.ModelAdmin):
    def thumbnail_preview(self, obj):
        if obj.thumbnail:
            # ✓ URL ที่เชื่อถือได้ (มาจาก model field ที่ validated)
            # ใช้ format_html แทน mark_safe เพื่อ escape parameter อัตโนมัติ
            return format_html(
                '<img src="{}" style="width: 50px; height: 50px;" />',
                obj.thumbnail.url
            )
        return '-'
    thumbnail_preview.short_description = 'Preview'
```

### format_html() — วิธีที่ปลอดภัยกว่า mark_safe()

`django.utils.html.format_html()` คือ HTML-safe version ของ `str.format()`
ที่ escape arguments อัตโนมัติ:

```python
from django.utils.html import format_html, format_html_join
from django.utils.safestring import mark_safe

# format_html: escape arguments, แต่ไม่ escape template string
result = format_html(
    '<a href="{}" class="btn">{}</a>',
    user_provided_url,    # ← escape อัตโนมัติ
    user_provided_text,   # ← escape อัตโนมัติ
)

# format_html_join: สำหรับ list of values
links = format_html_join(
    '\n',
    '<li><a href="{}">{}</a></li>',
    ((post.get_absolute_url(), post.title) for post in posts),
)

# conditional_escape: escape ถ้าไม่ใช่ SafeString
from django.utils.html import conditional_escape
safe_text = conditional_escape(user_input)
```

### API Reference: html utilities

| ฟังก์ชัน | Module | พฤติกรรม |
|---------|--------|---------|
| `mark_safe(s)` | `django.utils.safestring` | Mark string ว่าปลอดภัย (ไม่ escape) |
| `format_html(fmt, *args)` | `django.utils.html` | Safe string formatting (escape args) |
| `format_html_join(sep, fmt, args_generator)` | `django.utils.html` | Safe join ของหลาย format_html |
| `escape(s)` | `django.utils.html` | Force escape HTML |
| `conditional_escape(s)` | `django.utils.html` | Escape ถ้าไม่ใช่ SafeString |
| `strip_tags(value)` | `django.utils.html` | ลบ HTML tags ทั้งหมด |
| `linebreaks(value)` | `django.utils.html` | แปลง `\n` เป็น `<p>` / `<br>` |
| `urlize(text)` | `django.utils.html` | แปลง URLs ใน text เป็น `<a>` links |
| `json_script(value, element_id)` | `django.utils.html` | Render JSON ใน `<script>` อย่างปลอดภัย |

---

## ขั้นตอนที่ 806: |safe Filter และ {% autoescape %} Tag

### |safe Filter

```django
{# ❌ อย่าใช้กับ user input โดยตรง #}
{{ user.bio|safe }}

{# ✓ ใช้กับ content ที่ผ่าน sanitization แล้ว #}
{{ sanitized_content|safe }}

{# ✓ ใช้กับ HTML ที่สร้างจาก server-side #}
{{ form_field.as_widget|safe }}
```

### {% autoescape %} Tag

```django
{# ปิด autoescape สำหรับบล็อก (ใช้อย่างระวัง) #}
{% autoescape off %}
    {# ทุกตัวแปรในนี้จะไม่ถูก escape #}
    {{ trusted_html_content }}
{% endautoescape %}

{# เปิด autoescape อีกครั้ง (default) #}
{% autoescape on %}
    {{ user_input }}  {# escape ตามปกติ #}
{% endautoescape %}
```

### json_script สำหรับ JavaScript Data

`json_script` เป็น built-in filter ที่ปลอดภัยสำหรับส่งข้อมูลไปยัง JavaScript:

```django
{# Template #}
{{ user_data|json_script:"user-data-element" }}

{# Output (Django escape ค่า JSON อย่างปลอดภัย): #}
<script id="user-data-element" type="application/json">{"name": "Alice", "id": 1}</script>

{# อ่านใน JavaScript: #}
```

```javascript
// อ่านข้อมูลจาก json_script อย่างปลอดภัย
const userData = JSON.parse(
    document.getElementById('user-data-element').textContent
);
```

---

## ขั้นตอนที่ 807: SQL Injection Prevention — ORM Parameterization

### Django ORM ใช้ Parameterized Queries

Django's ORM ไม่เคย string-interpolate ค่าจาก Python ลงใน SQL โดยตรง
ทุก QuerySet method ที่รับค่าจาก user จะส่งเป็น database parameter แยกต่างหาก

```python
# ทุก ORM query เหล่านี้ใช้ parameterized queries อัตโนมัติ

# filter()
Post.objects.filter(title=user_input)
# SQL: SELECT * FROM blog_post WHERE title = %s   params: [user_input]

# get()
Post.objects.get(id=pk_from_url)
# SQL: SELECT * FROM blog_post WHERE id = %s   params: [pk_from_url]

# create()
Post.objects.create(title=user_title, content=user_content)
# SQL: INSERT INTO blog_post (title, content, ...) VALUES (%s, %s, ...)

# update()
Post.objects.filter(id=pk).update(title=new_title)
# SQL: UPDATE blog_post SET title = %s WHERE id = %s

# order_by() — Django validate field names ก่อนใช้
Post.objects.order_by('created_at')  # ✓
Post.objects.order_by('-created_at')  # ✓

# Q objects ก็ใช้ parameterization เช่นกัน
from django.db.models import Q
Post.objects.filter(
    Q(title__icontains=search_term) | Q(content__icontains=search_term)
)
```

### QuerySet Lookups และ Parameterization

Django lookup expressions ทั้งหมดใช้ parameterized queries:

```python
# Lookup types ที่ใช้ parameterization
Post.objects.filter(title__exact=val)
Post.objects.filter(title__iexact=val)
Post.objects.filter(title__contains=val)
Post.objects.filter(title__icontains=val)
Post.objects.filter(title__startswith=val)
Post.objects.filter(title__endswith=val)
Post.objects.filter(id__in=[1, 2, 3])
Post.objects.filter(id__range=(1, 100))
Post.objects.filter(created_at__date=date_val)
Post.objects.filter(created_at__year=2024)
```

---

## ขั้นตอนที่ 808: raw() และ extra() — Safe Usage Reference

### raw() — Raw SQL Queries

`Manager.raw()` ใช้เมื่อ ORM ไม่สามารถ express query ที่ต้องการได้
Django ยังคงใช้ parameterized queries สำหรับ `raw()`:

```python
# ✓ ถูกต้อง: ใช้ %s placeholder + params argument
from django.db import connection

# raw() บน Manager
posts = Post.objects.raw(
    'SELECT * FROM blog_post WHERE title = %s',
    [user_title]  # ← params list
)

# ✓ ถูกต้อง: named parameters
posts = Post.objects.raw(
    'SELECT * FROM blog_post WHERE id = %(post_id)s',
    {'post_id': pk}  # ← params dict
)

# raw() รองรับ translations:
from django.db import models
posts = Post.objects.raw(
    '''
    SELECT p.id, p.title, COUNT(c.id) as comment_count
    FROM blog_post p
    LEFT JOIN blog_comment c ON c.post_id = p.id
    WHERE p.is_published = %s
    GROUP BY p.id
    ORDER BY comment_count DESC
    ''',
    [True]
)
for post in posts:
    print(post.title, post.comment_count)
```

### cursor.execute() — Raw Database Cursor

```python
from django.db import connection

def get_top_authors(limit):
    with connection.cursor() as cursor:
        # ✓ ถูกต้อง: parameterized query
        cursor.execute(
            '''
            SELECT u.username, COUNT(p.id) as post_count
            FROM auth_user u
            JOIN blog_post p ON p.author_id = u.id
            GROUP BY u.id, u.username
            ORDER BY post_count DESC
            LIMIT %s
            ''',
            [limit]  # ← parameter
        )
        return cursor.fetchall()

# ✓ Named parameters (PostgreSQL-style)
def search_posts(search_term, min_date):
    with connection.cursor() as cursor:
        cursor.execute(
            '''
            SELECT id, title, created_at
            FROM blog_post
            WHERE title ILIKE %(term)s
            AND created_at >= %(since)s
            ''',
            {'term': f'%{search_term}%', 'since': min_date}
            # *** Django escape '%' และ '_' ใน LIKE patterns อัตโนมัติ? ไม่!
            # ต้อง escape เอง ถ้าต้องการ literal '%' ใน LIKE
        )
        columns = [col[0] for col in cursor.description]
        return [dict(zip(columns, row)) for row in cursor.fetchall()]
```

### extra() — Legacy QuerySet Method

`extra()` เป็น legacy method ที่ Django ยังไม่ได้เอาออก
แนะนำให้ใช้ `annotate()` + `RawSQL()` แทน แต่ถ้าจำเป็นต้องใช้:

```python
# ✓ ถูกต้อง: ใช้ params argument
Post.objects.extra(
    select={'comment_count': 'SELECT COUNT(*) FROM blog_comment WHERE post_id = blog_post.id'},
    where=['created_at > %s'],
    params=[cutoff_date]  # ← params
)

# ✓ ทางเลือกที่ดีกว่า: ใช้ annotate() กับ RawSQL (Django 1.8+)
from django.db.models.expressions import RawSQL

Post.objects.annotate(
    comment_count=RawSQL(
        'SELECT COUNT(*) FROM blog_comment WHERE post_id = blog_post.id',
        ()  # empty params tuple
    )
).filter(created_at__gt=cutoff_date)
```

---

## ขั้นตอนที่ 809: ModelForm Fields — Input Validation Reference

### ModelForm Validation Chain

```
HTTP Request → ModelForm(data=request.POST) → is_valid()
                                                    │
                                         ┌──────────┴──────────┐
                                         ↓                     ↓
                                   Field.clean()         Form.clean()
                                         │
                                ┌────────┴────────┐
                                ↓                 ↓
                         to_python()        validate()
                         (type cast)        (validators)
```

### Field Types และ Built-in Validation

| Field Type | ตรวจสอบอะไร | Clean Output Type |
|-----------|------------|------------------|
| `CharField` | max_length, min_length | `str` |
| `EmailField` | email format (RFC 5321) | `str` |
| `URLField` | URL format, scheme whitelist | `str` |
| `IntegerField` | numeric, min_value, max_value | `int` |
| `DecimalField` | numeric, max_digits, decimal_places | `Decimal` |
| `DateField` | date format (input_formats) | `datetime.date` |
| `DateTimeField` | datetime format | `datetime.datetime` |
| `BooleanField` | True/False | `bool` |
| `ChoiceField` | value ต้องอยู่ใน choices | `str` |
| `TypedChoiceField` | choices + type coercion | type specified |
| `FileField` | file upload | `UploadedFile` |
| `ImageField` | valid image (requires Pillow) | `UploadedFile` |
| `SlugField` | slug format (`[a-zA-Z0-9_-]+`) | `str` |
| `UUIDField` | valid UUID | `uuid.UUID` |
| `GenericIPAddressField` | valid IPv4/IPv6 | `str` |

### Validators Reference

```python
from django.core.validators import (
    MinLengthValidator,
    MaxLengthValidator,
    MinValueValidator,
    MaxValueValidator,
    RegexValidator,
    EmailValidator,
    URLValidator,
    FileExtensionValidator,
    validate_email,
    validate_slug,
    validate_unicode_slug,
    validate_ipv4_address,
    validate_ipv6_address,
    validate_ipv46_address,
    int_list_validator,
)
from django import forms

class PostForm(forms.ModelForm):
    title = forms.CharField(
        min_length=5,
        max_length=200,
        validators=[
            RegexValidator(
                regex=r'^[\w\s\-\.:,!?]+$',
                message='กรุณาใช้เฉพาะตัวอักษร, ตัวเลข, และ !?.,:-',
                code='invalid_title',
            ),
        ]
    )
    
    content = forms.CharField(
        widget=forms.Textarea,
        min_length=50,
        max_length=50000,
    )
    
    class Meta:
        model = Post
        fields = ['title', 'content', 'category', 'tags', 'is_published']
    
    def clean_title(self):
        title = self.cleaned_data['title']
        # cleaned_data['title'] ผ่าน field validation แล้ว (type, length, regex)
        # ทำ business logic validation เพิ่มได้
        if Post.objects.filter(title__iexact=title).exclude(pk=self.instance.pk).exists():
            raise forms.ValidationError('บทความชื่อนี้มีอยู่แล้ว')
        return title.strip()
    
    def clean(self):
        cleaned_data = super().clean()
        # Cross-field validation
        title = cleaned_data.get('title', '')
        content = cleaned_data.get('content', '')
        if title and content and title.lower() in content.lower():
            # business rule: content ต้องไม่เริ่มด้วย title
            pass
        return cleaned_data
```

### File Upload Validation

```python
from django.core.validators import FileExtensionValidator
from django.core.exceptions import ValidationError

class ProfileImageForm(forms.Form):
    avatar = forms.ImageField(
        validators=[
            FileExtensionValidator(
                allowed_extensions=['jpg', 'jpeg', 'png', 'webp'],
                message='รองรับเฉพาะไฟล์ JPG, PNG, WebP',
            ),
        ]
    )
    
    def clean_avatar(self):
        avatar = self.cleaned_data.get('avatar')
        if avatar:
            # ตรวจสอบ file size
            if avatar.size > 5 * 1024 * 1024:  # 5 MB
                raise ValidationError('ไฟล์ต้องมีขนาดไม่เกิน 5 MB')
            
            # ตรวจสอบ MIME type จริง (ไม่ใช่แค่ extension)
            import imghdr
            content_type = imghdr.what(avatar)
            if content_type not in ['jpeg', 'png', 'webp']:
                raise ValidationError('ไฟล์ไม่ใช่รูปภาพที่ถูกต้อง')
        
        return avatar
```

---

## ขั้นตอนที่ 810: Capstone — Security Configuration สำหรับ Blog Application

### สรุป Security Configuration ทั้งหมดในที่เดียว

```python
# settings/production.py — Security Section (complete)

# ══════════════════════════════════════════════════════════
# CSRF Configuration
# ══════════════════════════════════════════════════════════
CSRF_COOKIE_SECURE = True
CSRF_COOKIE_HTTPONLY = False      # ต้อง False สำหรับ Fetch API
CSRF_COOKIE_SAMESITE = 'Lax'
CSRF_COOKIE_AGE = 31449600        # 1 ปี

# Trusted origins สำหรับ cross-origin requests
CSRF_TRUSTED_ORIGINS = [
    f'https://{host}' for host in ALLOWED_HOSTS
]

# ══════════════════════════════════════════════════════════
# Session Configuration
# ══════════════════════════════════════════════════════════
SESSION_COOKIE_SECURE = True
SESSION_COOKIE_HTTPONLY = True
SESSION_COOKIE_SAMESITE = 'Lax'
SESSION_COOKIE_AGE = 1209600      # 2 สัปดาห์
SESSION_ENGINE = 'django.contrib.sessions.backends.db'

# ══════════════════════════════════════════════════════════
# Template Autoescape (ตั้งแต่ Django ใช้ Django templates)
# ══════════════════════════════════════════════════════════
# ค่าเริ่มต้นเปิด autoescape — ไม่จำเป็นต้องตั้งค่าเพิ่ม
# ถ้าใช้ Jinja2 → ต้องตั้ง autoescape=True ใน Environment

# ══════════════════════════════════════════════════════════
# Content Security Policy (ใช้ django-csp)
# ══════════════════════════════════════════════════════════
CONTENT_SECURITY_POLICY = {
    "DIRECTIVES": {
        "default-src": ["'self'"],
        "script-src": [
            "'self'",
            "cdn.jsdelivr.net",
            "unpkg.com",
        ],
        "style-src": [
            "'self'",
            "'unsafe-inline'",              # สำหรับ Bootstrap inline styles
            "fonts.googleapis.com",
        ],
        "font-src": [
            "'self'",
            "fonts.gstatic.com",
        ],
        "img-src": [
            "'self'",
            "data:",
            "*.example.com",
        ],
        "connect-src": [
            "'self'",
            "wss://ws.example.com",         # สำหรับ WebSockets
        ],
        "frame-ancestors": ["'none'"],      # แทน X-Frame-Options: DENY
        "form-action": ["'self'"],          # Form ต้อง submit ไปยัง same origin เท่านั้น
        "base-uri": ["'self'"],
        "object-src": ["'none'"],
    }
}
```

### Custom CSRF Failure View

```python
# views/errors.py
from django.views.decorators.csrf import requires_csrf_token
from django.shortcuts import render

@requires_csrf_token
def csrf_failure(request, reason=''):
    """Custom CSRF failure page"""
    return render(
        request,
        'errors/403_csrf.html',
        {
            'reason': reason,
            'title': 'Session หมดอายุ',
            'message': 'กรุณาโหลดหน้าเว็บใหม่และลองอีกครั้ง',
        },
        status=403
    )

# settings/base.py
CSRF_FAILURE_VIEW = 'blog.views.errors.csrf_failure'
```

```django
{# templates/errors/403_csrf.html #}
{% extends "base.html" %}

{% block title %}Session หมดอายุ{% endblock %}

{% block content %}
<div class="container py-5 text-center">
    <h1>⚠️ Session หมดอายุ</h1>
    <p class="lead">กรุณาโหลดหน้าเว็บใหม่แล้วลองอีกครั้ง</p>
    <a href="{{ request.path }}" class="btn btn-primary">
        โหลดหน้าเว็บใหม่
    </a>
</div>
{% endblock %}
```

### Security Test Suite

```python
# tests/test_security.py
from django.test import TestCase, Client
from django.urls import reverse
from django.contrib.auth import get_user_model
from blog.models import Post

User = get_user_model()


class CSRFProtectionTests(TestCase):
    def setUp(self):
        self.client = Client(enforce_csrf_checks=True)
        self.user = User.objects.create_user(
            username='testuser',
            password='testpass123!'
        )
        self.client.login(username='testuser', password='testpass123!')
    
    def test_post_without_csrf_returns_403(self):
        """POST request ที่ไม่มี CSRF token ต้องได้รับ 403"""
        response = self.client.post(
            reverse('blog:post-create'),
            {'title': 'Test', 'content': 'Content'},
        )
        self.assertEqual(response.status_code, 403)
    
    def test_get_includes_csrf_token(self):
        """GET request ต้อง set CSRF cookie"""
        response = self.client.get(reverse('blog:post-create'))
        self.assertIn('csrftoken', response.cookies)


class AutoescapeTests(TestCase):
    def setUp(self):
        self.user = User.objects.create_user(
            username='testuser',
            password='testpass123!',
        )
        self.post = Post.objects.create(
            title='<script>alert("xss")</script>',
            content='Normal content',
            author=self.user,
            is_published=True,
        )
    
    def test_html_in_title_is_escaped_in_list(self):
        """HTML ใน title ต้องถูก escape เมื่อแสดงผล"""
        response = self.client.get(reverse('blog:post-list'))
        self.assertNotContains(response, '<script>alert')
        self.assertContains(response, '&lt;script&gt;')
    
    def test_html_in_title_is_escaped_in_detail(self):
        """HTML ใน title ต้องถูก escape ในหน้า detail"""
        response = self.client.get(
            reverse('blog:post-detail', kwargs={'pk': self.post.pk})
        )
        self.assertNotContains(response, '<script>alert')


class SQLInjectionTests(TestCase):
    def setUp(self):
        self.user = User.objects.create_user(
            username='testuser',
            password='testpass123!',
        )
        Post.objects.create(
            title='Normal Post',
            content='Content',
            author=self.user,
            is_published=True,
        )
    
    def test_search_with_sql_metacharacters(self):
        """Search ด้วย SQL metacharacters ต้องทำงานได้โดยไม่ error"""
        # Django ORM parameterize queries อัตโนมัติ
        # ค่าเหล่านี้จะถูก escape โดย database driver
        test_inputs = [
            "'; DROP TABLE blog_post; --",  # SQL injection attempt string
            "' OR '1'='1",
            "' UNION SELECT * FROM auth_user --",
            '%" AND 1=1 --',
        ]
        for search_input in test_inputs:
            with self.subTest(search=search_input):
                # ต้องไม่เกิด DatabaseError
                results = list(Post.objects.filter(
                    title__icontains=search_input
                ))
                # results อาจว่าง แต่ต้องไม่ throw exception
                self.assertIsInstance(results, list)
    
    def test_search_view_with_special_characters(self):
        """Search view ต้องจัดการ special characters ได้"""
        test_inputs = [
            "'; DROP TABLE blog_post; --",
            '<script>',
            '%27 OR 1=1',
        ]
        for search_input in test_inputs:
            with self.subTest(search=search_input):
                response = self.client.get(
                    reverse('blog:post-list'),
                    {'q': search_input}
                )
                # Response ต้องไม่ใช่ 500
                self.assertNotEqual(response.status_code, 500)
```

### Secure View Pattern — Blog Post Create

```python
# views/post.py
from django.contrib.auth.mixins import LoginRequiredMixin
from django.views.generic.edit import CreateView
from django.urls import reverse_lazy
from django.utils.html import strip_tags
from blog.forms import PostForm
from blog.models import Post


class PostCreateView(LoginRequiredMixin, CreateView):
    model = Post
    form_class = PostForm
    template_name = 'blog/post_form.html'
    success_url = reverse_lazy('blog:post-list')
    
    def form_valid(self, form):
        # ผูก post กับ user ที่ login อยู่
        # ไม่รับ author จาก form fields เพื่อป้องกัน mass assignment
        form.instance.author = self.request.user
        return super().form_valid(form)
```

```django
{# templates/blog/post_form.html #}
{% extends "base.html" %}

{% block content %}
<form method="post" novalidate>
    {% csrf_token %}
    
    {% for field in form %}
        <div class="mb-3">
            <label for="{{ field.id_for_label }}" class="form-label">
                {{ field.label }}
            </label>
            {{ field }}
            {% if field.errors %}
                <div class="invalid-feedback d-block">
                    {% for error in field.errors %}
                        {{ error }}
                    {% endfor %}
                </div>
            {% endif %}
            {% if field.help_text %}
                <div class="form-text">{{ field.help_text }}</div>
            {% endif %}
        </div>
    {% endfor %}
    
    <button type="submit" class="btn btn-primary">บันทึก</button>
</form>
{% endblock %}
```

---

## สรุป Part 081: Configuration Reference

### CSRF — Settings ที่ต้องตั้งใน Production

| Setting | ค่า Production |
|---------|--------------|
| `CSRF_COOKIE_SECURE` | `True` |
| `CSRF_COOKIE_SAMESITE` | `'Lax'` |
| `CSRF_TRUSTED_ORIGINS` | list of `https://` origins |
| `CSRF_FAILURE_VIEW` | custom 403 view (optional) |
| `CsrfViewMiddleware` | ต้องอยู่ใน MIDDLEWARE |

### XSS — Configuration Points

| Component | ค่า/การกระทำ |
|-----------|------------|
| Django Template Engine | autoescape = True (default, ไม่ต้องตั้ง) |
| Jinja2 Backend | ต้องตั้ง `autoescape=True` เอง |
| `mark_safe()` | ใช้เฉพาะ HTML ที่สร้างจาก server |
| `format_html()` | ใช้แทน mark_safe() + string formatting |
| `|safe` filter | ใช้เฉพาะ content ที่ sanitized แล้ว |
| `{% autoescape off %}` | หลีกเลี่ยงหรือใช้อย่างระวัง |

### SQL Injection — ORM Behavior Reference

| Method | Parameterized | หมายเหตุ |
|--------|--------------|----------|
| `filter()`, `get()`, `exclude()` | ✓ อัตโนมัติ | ทุก lookup |
| `create()`, `update()`, `bulk_create()` | ✓ อัตโนมัติ | ทุก field |
| `raw(sql, params=[])` | ✓ เมื่อใช้ params | ต้องระบุ params แยก |
| `cursor.execute(sql, params)` | ✓ เมื่อใช้ params | ต้องระบุ params แยก |
| `extra(where=[], params=[])` | ✓ เมื่อใช้ params | legacy API |
| `RawSQL(sql, params)` | ✓ เมื่อใช้ params | ใช้ใน annotate() |
| `order_by(field_name)` | ✓ validate field names | ไม่รับ raw SQL |

ใน **Part 082** เราจะดู Security Headers และ HTTPS ซึ่งครอบคลุม
`SecurityMiddleware` settings, HSTS, Content Security Policy, และ
Let's Encrypt certificate setup ที่เราใช้ใน Production จริง

---

*หลักสูตร Django ฉบับสมบูรณ์ — Part 081/100*