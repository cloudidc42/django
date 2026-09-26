# Part 053: HTMX กับ Django สำหรับ Interactive UI

> **ขั้นตอนที่ 521-530 ของหลักสูตร** | Phase 6: Frontend Integration
>
> Part 052 พาคุณไปทางสาย **Fetch API + JSON**: เขียน endpoint ที่คืน JSON,
> เขียน JavaScript ฝั่ง client ดึงข้อมูลมาเอง แล้วประกอบ DOM เองทีละบรรทัด —
> วิธีนี้ใช้ได้จริง แต่ต้อง**เขียนโครงสร้างหน้าเว็บซ้ำสองที่**เสมอ: หนึ่งครั้งใน
> Django Template (สำหรับโหลดหน้าแรก) และอีกครั้งใน JavaScript (สำหรับอัปเดต
> บางส่วนภายหลัง) Part นี้จะแนะนำแนวทางที่ตรงข้ามกันโดยสิ้นเชิงคือ **HTMX** —
> ไลบรารีขนาดเล็กที่ทำให้ HTML ธรรมดาสามารถส่ง request แบบ AJAX ได้ผ่าน
> attribute พิเศษ (`hx-get`, `hx-post`, ...) โดย **server ยังคงคืน HTML กลับมา
> เหมือนเดิมทุกประการ** ไม่ต้องแปลงเป็น JSON เลย นี่คือแนวคิดที่เรียกว่า
> **"server-driven UI"** ซึ่งเข้ากับปรัชญาของ Django ที่เน้น Template
> rendering ฝั่ง server อยู่แล้วอย่างลงตัวที่สุด คุณจะได้เรียนรู้ตั้งแต่
> การติดตั้ง HTMX, attribute หลักทั้งหมด (`hx-get`, `hx-post`, `hx-trigger`,
> `hx-target`, `hx-swap`), การออกแบบ View ที่คืน template fragment โดยเฉพาะ,
> การทำฟอร์ม validation แบบ inline ไม่ reload หน้า, `hx-boost` ที่เปลี่ยน
> ทั้งเว็บให้ลื่นไหลแบบ SPA, Infinite Scroll, การผสาน HTMX กับ Django
> Messages Framework และ package `django-htmx` ที่ทำให้ชีวิตง่ายขึ้นไปอีกขั้น
> เมื่อจบ Part นี้ คุณจะแปลงระบบคอมเมนต์ของ blog ทั้งระบบให้เป็น HTMX เต็ม
> รูปแบบ — โพสต์และลบคอมเมนต์ได้โดยไม่ต้อง reload หน้าเว็บแม้แต่ครั้งเดียว

---

## สารบัญของ Part นี้

- ขั้นตอนที่ 521: HTMX คืออะไร — แนวคิด "server-driven UI" ทำไมเข้ากับ Django ได้ดีมาก
- ขั้นตอนที่ 522: ติดตั้ง HTMX, ใช้ `hx-get`/`hx-post` พื้นฐาน และการจัดการ CSRF Token
- ขั้นตอนที่ 523: `hx-trigger`, `hx-target`, `hx-swap` เจาะลึก
- ขั้นตอนที่ 524: Partial Template Rendering — View ที่คืน template fragment สำหรับ HTMX โดยเฉพาะ
- ขั้นตอนที่ 525: HTMX ร่วมกับ Django Forms — Inline Validation แบบไม่ Reload หน้า
- ขั้นตอนที่ 526: `hx-boost` — เปลี่ยนทั้งเว็บให้เป็น SPA-like โดยไม่ต้องเขียน JS Framework
- ขั้นตอนที่ 527: Infinite Scroll / Lazy Loading ด้วย HTMX (`hx-trigger="revealed"`)
- ขั้นตอนที่ 528: ผสาน HTMX กับ Django Messages Framework
- ขั้นตอนที่ 529: แนะนำ Package `django-htmx` (`request.htmx` Helper Property)
- ขั้นตอนที่ 530: สรุปและแบบฝึกหัด — แปลงระบบคอมเมนต์ของ Blog ให้เป็น HTMX เต็มรูปแบบ

---

## ขั้นตอนที่ 521: HTMX คืออะไร — แนวคิด "Server-Driven UI"

### 521.1 ทบทวนปัญหาของแนวทาง Fetch API + JSON จาก Part 052

ใน Part 052 คุณเขียน endpoint ที่คืน JSON แล้วใช้ `fetch()` ฝั่ง JavaScript
ดึงข้อมูลมาประกอบเป็น HTML เอง เช่น การกดปุ่ม "โหลดคอมเมนต์เพิ่ม" ต้องผ่าน
ขั้นตอนประมาณนี้:

```javascript
// แนวทางแบบ Part 052: ต้องเขียนโครง HTML ซ้ำใน JavaScript
async function loadComments(postId) {
    const res = await fetch(`/api/posts/${postId}/comments/`);
    const comments = await res.json();
    const container = document.getElementById('comment-list');
    container.innerHTML = comments.map(c => `
        <div class="comment-item">
            <strong>${escapeHtml(c.author)}</strong>
            <p>${escapeHtml(c.text)}</p>
        </div>
    `).join('');
}
```

สังเกตปัญหาที่ตามมา:

1. **เขียนโครง HTML ของคอมเมนต์ 2 ที่**: ครั้งแรกใน Django Template
   (`comment_list.html` ตอนโหลดหน้าแรก) และครั้งที่สองในสตริง JavaScript
   ข้างบน (ตอนโหลดเพิ่มทีหลัง) — ถ้าวันหนึ่งอยากเปลี่ยนดีไซน์การ์ดคอมเมนต์
   ต้องแก้ทั้งสองที่ให้ตรงกันเป๊ะ ไม่เช่นนั้นหน้าเว็บจะดูไม่เหมือนกันระหว่าง
   ตอนโหลดครั้งแรกกับตอนโหลดเพิ่ม
2. **ต้องป้องกัน XSS เอง**: Django Template escape ค่าให้อัตโนมัติผ่าน
   auto-escaping (ทบทวน Part 008) แต่ JavaScript ที่ต่อสตริง HTML เองต้อง
   เขียนฟังก์ชัน `escapeHtml()` เองทุกจุดที่แทรกข้อมูลจากผู้ใช้ ถ้าลืมแม้แต่
   จุดเดียวคือช่องโหว่ความปลอดภัยทันที
3. **Logic การแสดงผลกระจายอยู่ 2 ภาษา**: เงื่อนไขอย่าง "ถ้าไม่มีคอมเมนต์เลย
   ให้แสดงข้อความว่าง ๆ" ต้องเขียนทั้งใน `{% empty %}` ของ Django Template
   และใน `if (comments.length === 0)` ของ JavaScript แยกกัน

### 521.2 แนวคิด "HTML over the Wire" / Server-Driven UI

**HTMX** (https://htmx.org) แก้ปัญหานี้ด้วยแนวคิดที่ตรงข้ามกับ SPA/Fetch+JSON
โดยสิ้นเชิง: แทนที่จะให้ server คืน **JSON** แล้วให้ JavaScript ฝั่ง client
แปลงเป็น HTML เอง — HTMX ให้ server คืน **HTML fragment สำเร็จรูป** กลับมา
ตรง ๆ แล้ว HTMX แค่เอา fragment นั้นไปแทนที่ส่วนหนึ่งของหน้าเว็บที่มีอยู่แล้ว
เท่านั้น ไม่มีการแปลงข้อมูลไปมาระหว่างรูปแบบใด ๆ เลย

```
แนวทาง Fetch + JSON (Part 052)              แนวทาง HTMX (Part นี้)
─────────────────────────────              ──────────────────────
Browser → GET /api/comments/  (JSON)        Browser → GET /comments/  (HTML)
Server  → [{"author":"...", ...}]           Server  → <div class="comment-item">...</div>
JS      → แปลง JSON เป็น HTML เอง            HTMX    → แทรก HTML ที่ได้ลงหน้าเว็บทันที
        → ประกอบ DOM เอง                             → ไม่มีขั้นตอนแปลงข้อมูลเลย
```

แนวคิดนี้มีชื่อเรียกในวงการว่า **"HTML over the Wire"** หรือ
**"Server-Driven UI"** — ตรรกะการแสดงผลทั้งหมดยังอยู่ที่ server (ที่เดียว)
เหมือนเว็บแบบดั้งเดิม เพียงแต่เปลี่ยน "หน่วยของการอัปเดต" จาก **ทั้งหน้า**
(full page reload) มาเป็น **บางส่วนของหน้า** (partial fragment) เท่านั้น

### 521.3 ประวัติโดยย่อของ HTMX

HTMX พัฒนาโดย **Carson Gross** ต่อยอดมาจากไลบรารีรุ่นก่อนหน้าชื่อ
**intercooler.js** ที่เขาเขียนไว้ตั้งแต่ปี 2013 แนวคิดหลักคือ "HTML ควรมี
ความสามารถมากกว่าที่มันมีมาแต่เดิม" — ทำไม `<a>` และ `<form>` เท่านั้นที่ยิง
request ได้ ทำไม request ต้องเปลี่ยนทั้งหน้าเสมอ ทำไมต้อง trigger จากการคลิก
เท่านั้น HTMX ตอบคำถามเหล่านี้ด้วย attribute ชุดใหม่ที่ขยายความสามารถของ HTML
ให้ **element ไหนก็ได้** ยิง request แบบไหนก็ได้ (GET/POST/PUT/PATCH/DELETE)
จาก event ไหนก็ได้ (click, keyup, scroll, ...) แล้วเอาผลลัพธ์ไปแทรกที่ไหน
ของหน้าก็ได้ — ทั้งหมดนี้**โดยไม่ต้องเขียน JavaScript เองสักบรรทัดเดียว**

### 521.4 ทำไม Django เข้ากับ HTMX ได้ดีเป็นพิเศษ

| ปัจจัย | ทำไมเข้ากันดี |
|---|---|
| Django Template Engine render HTML อยู่แล้ว | HTMX ต้องการ HTML fragment กลับมา — Django ทำสิ่งนี้เป็นงานหลักอยู่แล้วตั้งแต่ Part 008 ไม่ต้องเพิ่ม library ใหม่ |
| ไม่ต้องเขียน Serializer แยก | สาย Fetch/SPA/React (Part 052, 055) ต้องมี DRF Serializer แปลง Model เป็น JSON เสมอ (Part 041) — สาย HTMX ใช้ Template ธรรมดาที่มีอยู่แล้วได้เลย |
| Auto-escaping ป้องกัน XSS ให้ฟรี | ต่างจากการต่อสตริง HTML ใน JavaScript เอง (ข้อ 521.1) Django Template escape ค่าให้อัตโนมัติเสมอ |
| Form + `ModelForm` re-render ซ้ำได้ทันที | Part 025-026 สอน `ModelForm` ที่ render error กลับมาเป็น HTML อยู่แล้ว — HTMX แค่เอาผลลัพธ์นั้นไปแทรกในหน้าโดยไม่ reload (ขั้นตอนที่ 525) |
| ทีมเล็กเขียนโค้ดน้อยกว่ามาก | ไม่ต้องดูแลทั้ง Python (backend) และ JavaScript framework แยกกันสองชุด ลดพื้นที่ผิวของบั๊กและจำนวนไฟล์ที่ต้อง maintain |

### 521.5 ตารางเปรียบเทียบ 3 แนวทางการทำ Interactive UI ที่หลักสูตรนี้ครอบคลุม

| ประเด็น | Vanilla JS + Fetch API (Part 052) | HTMX (Part นี้) | React/Vue (Part 055-056) |
|---|---|---|---|
| รูปแบบข้อมูลที่ server คืน | JSON | **HTML fragment** | JSON |
| ต้องเขียน Serializer (DRF) ไหม | ต้อง | **ไม่ต้อง** | ต้อง |
| Logic การแสดงผลอยู่ที่ไหน | กระจาย: Template + JavaScript | **รวมที่ Template ฝั่งเดียว** | กระจาย: Template (โหลดแรก) + Component |
| ต้องเรียนภาษา/syntax ใหม่มากแค่ไหน | JavaScript พื้นฐาน-กลาง | HTML attribute ใหม่ ~15-20 ตัว | ภาษา/build tool ใหม่ทั้งชุด |
| เหมาะกับทีมขนาดไหน | ทีมที่มี frontend dev แยก | **ทีมเล็ก-กลาง ที่ backend dev ทำ UI เองได้** | ทีมที่มี frontend dev เฉพาะทางชัดเจน |
| SEO / initial load | ต้องรอ JS รันก่อนเห็นเนื้อหา (ถ้าไม่ SSR) | **HTML เต็มตั้งแต่ request แรกเสมอ** | ต้องทำ SSR เพิ่มถ้าต้องการ SEO ดี |
| ตัวอย่างเว็บที่ใช้จริง | เว็บทั่วไปที่มี JS เสริมเล็กน้อย | GitHub (บางส่วน), Basecamp/HEY (แนวคิดเดียวกัน) | Instagram, Facebook, Airbnb |

> **จุดยืนของหลักสูตรนี้**: ทั้ง 3 แนวทางมีที่ใช้งานจริงในอุตสาหกรรม ไม่มีแนวทาง
> ไหน "ดีที่สุด" ตายตัว — HTMX เหมาะที่สุดเมื่อแอปพลิเคชันส่วนใหญ่ยังคงเป็น
> "หน้าเว็บที่มี interactive บางจุด" (เช่น comment, like, filter, inline edit)
> ในขณะที่ React/Vue เหมาะกับแอปที่มี state ฝั่ง client ซับซ้อนมาก ๆ
> (เช่น dashboard แบบ real-time, editor ที่ซับซ้อน) เราจะเรียนรู้ทั้งสามแนวทาง
> เพื่อให้คุณเลือกเครื่องมือที่เหมาะกับงานแต่ละชิ้นได้อย่างมืออาชีพ

---

## ขั้นตอนที่ 522: ติดตั้ง HTMX, ใช้ `hx-get`/`hx-post` พื้นฐาน

### 522.1 ติดตั้ง HTMX แบบ Self-Hosted (แนะนำสำหรับหลักสูตรนี้)

HTMX เป็นไฟล์ JavaScript **ไฟล์เดียว ไม่มี dependency** ไม่ต้องผ่าน npm/webpack
เลยก็ได้ วิธีที่แนะนำสำหรับ production คือดาวน์โหลดมาเก็บไว้เป็น static file
ของโปรเจกต์เอง (ไม่พึ่ง CDN ภายนอกที่อาจล่มหรือถูกบล็อกในบางเครือข่าย):

```bash
mkdir -p static/vendor/htmx
curl -L https://unpkg.com/htmx.org@2.0.3/dist/htmx.min.js \
    -o static/vendor/htmx/htmx.min.js
```

จากนั้นโหลดในเทมเพลตหลัก (ต่อยอดจาก `templates/base.html` ที่มีมาตั้งแต่
Part 008 และถูกจัดโครงสร้างใหม่ใน Part 028 ข้อ 277):

```html
<!-- templates/base.html -->
{% load static %}
<!DOCTYPE html>
<html lang="th">
<head>
    <meta charset="UTF-8">
    <title>{% block title %}Django Mastery Blog{% endblock %}</title>
    {% block extra_head %}{% endblock %}
</head>
<body{% block body_attrs %}{% endblock %}>
    <script src="{% static 'vendor/htmx/htmx.min.js' %}" defer></script>
    {% block content %}{% endblock %}
</body>
</html>
```

> **ทางเลือกแบบ CDN (เร็วสำหรับทดลองเล่น)**: `<script src="https://unpkg.com/htmx.org@2.0.3"></script>`
> ใช้ได้ดีตอนทดลองในเครื่อง แต่หลักสูตรนี้แนะนำ self-hosted เสมอสำหรับโค้ด
> ที่จะ deploy จริง เพราะควบคุมเวอร์ชันได้แน่นอนและไม่มีความเสี่ยงเรื่อง
> Content Security Policy (จะเจาะลึก CSP ใน Phase 9)

### 522.2 การจัดการ CSRF Token กับ HTMX — ขั้นตอนที่ห้ามลืมเด็ดขาด

ก่อนลองใช้ `hx-post` ต้องแก้ปัญหาสำคัญก่อน: Django บังคับให้ทุก POST/PUT/
PATCH/DELETE request แนบ **CSRF token** มาด้วยเสมอ (ทบทวนจาก Part 001 ข้อ
121 ที่กล่าวถึงระบบป้องกัน CSRF ในตัว) — แต่ HTMX ไม่รู้จัก `{% csrf_token %}`
โดยอัตโนมัติ ถ้าไม่ตั้งค่าอะไรเลย ทุก `hx-post` จะได้ error **403 Forbidden**
ทันที

วิธีที่สะอาดที่สุดคือติดตั้ง **event listener ตัวเดียว** ที่แนบ CSRF token
เข้าไปกับ**ทุก request ของ HTMX โดยอัตโนมัติ** ผ่าน event `htmx:configRequest`
ที่ HTMX ยิงออกมาก่อนส่ง request ทุกครั้ง:

```html
<!-- templates/base.html (เพิ่มก่อนปิด </body>) -->
<script src="{% static 'vendor/htmx/htmx.min.js' %}" defer></script>
<script>
    document.body.addEventListener('htmx:configRequest', (event) => {
        event.detail.headers['X-CSRFToken'] = getCookie('csrftoken');
    });

    // ฟังก์ชันมาตรฐานของ Django สำหรับอ่านค่าจาก cookie
    // (ตัวเดียวกับที่เอกสารทางการ Django แนะนำสำหรับ AJAX ทุกชนิด)
    function getCookie(name) {
        let cookieValue = null;
        if (document.cookie && document.cookie !== '') {
            const cookies = document.cookie.split(';');
            for (let cookie of cookies) {
                cookie = cookie.trim();
                if (cookie.startsWith(name + '=')) {
                    cookieValue = decodeURIComponent(cookie.substring(name.length + 1));
                    break;
                }
            }
        }
        return cookieValue;
    }
</script>
```

โค้ดนี้ทำงานตลอดทั้งหลักสูตรนับจากนี้ — เขียนไว้ครั้งเดียวใน `base.html`
ทุก `hx-post`/`hx-put`/`hx-delete` ที่จะเจอต่อจากนี้จะแนบ CSRF token ให้
อัตโนมัติโดยไม่ต้องเติมอะไรเพิ่มในแต่ละ element เลย

> **ข้อกำหนดสำคัญ**: `CSRF_COOKIE_HTTPONLY` ใน `settings.py` **ต้องเป็น
> `False`** (ค่าเริ่มต้นของ Django อยู่แล้ว) เพราะโค้ดข้างบนต้องอ่านค่า
> cookie ผ่าน `document.cookie` จาก JavaScript ได้ ถ้าโปรเจกต์เคยตั้งค่านี้
> เป็น `True` ไว้ด้วยเหตุผลด้านความปลอดภัยอื่น ต้องเปลี่ยนกลับ หรือใช้วิธี
> ฝัง token ผ่าน `{% csrf_token %}` ในหน้าแทนตามที่จะแสดงในขั้นตอนที่ 525

### 522.3 `hx-get` แรกของคุณ: ปุ่มโหลดเนื้อหาโดยไม่ Reload หน้า

ตัวอย่างที่ง่ายที่สุด: ปุ่มที่กดแล้วโหลด "บทความยอดนิยม" มาแสดงในหน้าโดยไม่
reload

```python
# blog/views.py
from django.shortcuts import render
from .models import Post


def post_list_view(request):
    posts = Post.objects.filter(is_published=True).order_by('-created_at')
    return render(request, 'blog/post_list.html', {'posts': posts})


def popular_posts_partial(request):
    """View นี้คืน HTML fragment เล็ก ๆ สำหรับ HTMX โดยเฉพาะ (เจาะลึกในขั้นตอนที่ 524)"""
    popular = Post.objects.filter(is_published=True).order_by('-like_count')[:5]
    return render(request, 'blog/partials/popular_posts.html', {'posts': popular})
```

```python
# blog/urls.py
from django.urls import path
from . import views

app_name = 'blog'

urlpatterns = [
    path('', views.post_list_view, name='list'),
    path('popular/', views.popular_posts_partial, name='popular_partial'),
]
```

```html
<!-- blog/templates/blog/partials/popular_posts.html -->
<ul class="popular-posts">
    {% for post in posts %}
        <li><a href="{% url 'blog:detail' post.slug %}">{{ post.title }}</a>
            ({{ post.like_count }} ถูกใจ)</li>
    {% empty %}
        <li>ยังไม่มีบทความยอดนิยม</li>
    {% endfor %}
</ul>
```

```html
<!-- blog/templates/blog/post_list.html -->
{% extends 'base.html' %}

{% block content %}
<h1>บทความทั้งหมด</h1>

<button hx-get="{% url 'blog:popular_partial' %}"
        hx-target="#popular-box"
        hx-swap="innerHTML">
    แสดงบทความยอดนิยม
</button>

<div id="popular-box"><!-- ผลลัพธ์จาก HTMX จะถูกแทรกตรงนี้ --></div>

<ul>
    {% for post in posts %}
        <li>{{ post.title }}</li>
    {% endfor %}
</ul>
{% endblock %}
```

อธิบาย attribute ทั้ง 3 ตัวที่ใช้:

| Attribute | หน้าที่ |
|---|---|
| `hx-get="{% url ... %}"` | เมื่อ trigger (ค่าเริ่มต้นของปุ่มคือ `click`) ให้ยิง `GET` request ไปยัง URL นี้ |
| `hx-target="#popular-box"` | เอาผลลัพธ์ (HTML) ที่ได้ไปแทรกใน element ที่มี `id="popular-box"` (ไม่ใช่ตัวปุ่มเอง) |
| `hx-swap="innerHTML"` | วิธีแทรก: แทนที่**เนื้อหาข้างใน**ของ target ทั้งหมด (ค่าเริ่มต้นอยู่แล้ว แต่เขียนชัดเจนไว้เพื่อความอ่านง่าย) |

เมื่อกดปุ่ม HTMX จะยิง `GET /blog/popular/` เบื้องหลัง (คล้าย `fetch()` ที่
เขียนเองใน Part 052 ทุกประการ) แต่ **View คืน HTML ไม่ใช่ JSON** และ HTMX
จัดการแทรกผลลัพธ์ให้อัตโนมัติ โดยไม่ต้องเขียน JavaScript เองสักบรรทัดเดียว

### 522.4 `hx-post` พื้นฐาน: ปุ่มถูกใจที่อัปเดตจำนวนโดยไม่ Reload

ขยาย Model `Post` จาก Part 026 เพิ่ม field `like_count`:

```python
# blog/models.py (เพิ่มเข้าไปใน class Post ที่มีอยู่แล้ว)
class Post(models.Model):
    # ... field เดิมทั้งหมดจาก Part 026 ...
    like_count = models.PositiveIntegerField(default=0)
```

```bash
python manage.py makemigrations blog
python manage.py migrate
```

```python
# blog/views.py
from django.shortcuts import get_object_or_404, render
from django.views.decorators.http import require_POST
from .models import Post


@require_POST
def post_like_view(request, slug):
    post = get_object_or_404(Post, slug=slug)
    post.like_count += 1
    post.save(update_fields=['like_count'])
    return render(request, 'blog/partials/like_button.html', {'post': post})
```

```python
# blog/urls.py (เพิ่มบรรทัดนี้)
urlpatterns += [
    path('<slug:slug>/like/', views.post_like_view, name='like'),
]
```

```html
<!-- blog/templates/blog/partials/like_button.html -->
<button hx-post="{% url 'blog:like' post.slug %}"
        hx-target="this"
        hx-swap="outerHTML"
        class="like-button">
    ❤️ ถูกใจ ({{ post.like_count }})
</button>
```

```html
<!-- blog/templates/blog/post_detail.html (ส่วนที่เกี่ยวข้อง) -->
{% include 'blog/partials/like_button.html' %}
```

จุดที่น่าสังเกตคือ `hx-target="this"` กับ `hx-swap="outerHTML"` ที่ต่างจาก
ตัวอย่างก่อนหน้า:

- `hx-target="this"` หมายถึง "เอาผลลัพธ์ไปแทรกที่**ตัวเอง**" (ปุ่มเป็น target
  ของตัวมันเอง ไม่ใช่ element อื่น)
- `hx-swap="outerHTML"` หมายถึง "แทนที่**ทั้ง element**" (ทั้ง `<button>...`)
  ไม่ใช่แค่เนื้อหาข้างใน — เพราะ View คืน `<button>` ตัวใหม่ทั้งก้อนกลับมา
  (พร้อมตัวเลขล่าสุดและ `hx-post` attribute เดิมที่ยังกดซ้ำได้เรื่อย ๆ)

ผลลัพธ์ที่ได้คือปุ่มถูกใจที่กดได้เรื่อย ๆ ตัวเลขอัปเดตทันทีโดยไม่ reload หน้า
เว็บแม้แต่ครั้งเดียว — เขียนโค้ด Python + HTML รวมกันไม่ถึง 20 บรรทัด เทียบกับ
แนวทาง Fetch + JSON ที่ต้องมี Serializer, endpoint คืน JSON, และ JavaScript
อัปเดต DOM เอง

---

## ขั้นตอนที่ 523: `hx-trigger`, `hx-target`, `hx-swap` เจาะลึก

### 523.1 `hx-trigger`: ควบคุมว่า "เมื่อไหร่" ที่จะยิง Request

ค่าเริ่มต้นของ `hx-trigger` ขึ้นกับชนิด element: `<button>`/`<a>` ใช้ `click`,
ส่วน `<form>` ใช้ `submit`, `<input>`/`<select>`/`<textarea>` ใช้ `change`
เสมอ — แต่ระบุเองได้อย่างละเอียดผ่าน **modifier** ต่าง ๆ:

| Modifier | ความหมาย | ตัวอย่างการใช้ |
|---|---|---|
| `changed` | ยิงเฉพาะเมื่อค่าที่ได้เปลี่ยนไปจริง (ไม่ยิงถ้าพิมพ์แล้วลบกลับมาค่าเดิม) | ค้นหาแบบพิมพ์สด |
| `delay:500ms` | รอ 500ms หลัง event ก่อนยิง ถ้ามี event ใหม่มาก่อนครบเวลา จะรีเซ็ตนับใหม่ (debounce) | ลดจำนวน request ตอนพิมพ์ค้นหา |
| `throttle:1s` | จำกัดไม่ให้ยิงถี่กว่า 1 วินาทีต่อครั้ง (ต่างจาก `delay` ตรงที่ยิงครั้งแรกทันที) | ป้องกัน spam คลิก |
| `once` | ยิงแค่ครั้งเดียวตลอดอายุของ element นั้น | โหลดข้อมูลเริ่มต้นครั้งเดียว |
| `from:<selector>` | ฟัง event จาก element อื่น ไม่ใช่ตัวเอง | ปุ่มนอก form สั่ง submit form |
| `load` | ยิงทันทีที่ element ถูกโหลดเข้าหน้า (แทน event ของผู้ใช้) | โหลดเนื้อหาเริ่มต้นแบบ lazy |
| `revealed` | ยิงเมื่อ element เลื่อนเข้ามาในหน้าจอที่มองเห็นได้ (scroll เข้ามา) | Infinite scroll (ขั้นตอนที่ 527) |
| `intersect` | คล้าย `revealed` แต่ควบคุม threshold ได้ละเอียดกว่าผ่าน Intersection Observer | Lazy loading รูปภาพ |

**ตัวอย่าง: ค้นหาบทความแบบพิมพ์สด (live search)**

```python
# blog/views.py
def post_search_partial(request):
    query = request.GET.get('q', '').strip()
    posts = Post.objects.filter(is_published=True)
    if query:
        posts = posts.filter(title__icontains=query)
    return render(request, 'blog/partials/post_search_results.html', {
        'posts': posts[:20],
        'query': query,
    })
```

```python
# blog/urls.py
path('search/', views.post_search_partial, name='search'),
```

```html
<!-- blog/templates/blog/post_list.html (เพิ่มเข้าไป) -->
<input type="search"
       name="q"
       placeholder="ค้นหาบทความ..."
       hx-get="{% url 'blog:search' %}"
       hx-trigger="keyup changed delay:400ms, search"
       hx-target="#search-results"
       hx-swap="innerHTML">

<div id="search-results"></div>
```

```html
<!-- blog/templates/blog/partials/post_search_results.html -->
{% if query %}
<ul>
    {% for post in posts %}
        <li><a href="{% url 'blog:detail' post.slug %}">{{ post.title }}</a></li>
    {% empty %}
        <li>ไม่พบบทความที่ตรงกับ "{{ query }}"</li>
    {% endfor %}
</ul>
{% endif %}
```

`hx-trigger="keyup changed delay:400ms, search"` ระบุ trigger **สองแบบคั่น
ด้วยคอมมา**: (1) `keyup changed delay:400ms` — รอ 400ms หลังพิมพ์ค่าที่
เปลี่ยนจริง และ (2) `search` — event พิเศษที่ browser ยิงให้เองเมื่อผู้ใช้กด
ปุ่ม "X" ล้างช่องค้นหาของ `type="search"` (ทำให้ผลลัพธ์เคลียร์ทันทีโดยไม่
ต้องรอ 400ms) การรวม trigger หลายแบบแบบนี้พบได้บ่อยมากในงานจริง

### 523.2 `hx-target`: ควบคุมว่า "ผลลัพธ์ไปอยู่ที่ไหน"

| ค่า | ความหมาย |
|---|---|
| (ไม่ระบุ) | ค่าเริ่มต้น = element ที่มี attribute `hx-get`/`hx-post` เอง |
| `this` | ตัวเอง (เขียนชัดเจนแทนค่าเริ่มต้น) |
| `#some-id` | element ที่มี `id="some-id"` (CSS selector ทั่วไปก็ใช้ได้ เช่น `.some-class`) |
| `closest <selector>` | element ที่ใกล้ที่สุดที่ตรงกับ selector โดยไล่ขึ้นจาก element ปัจจุบัน (รวมตัวเองด้วยถ้าตรง) |
| `next <selector>` / `previous <selector>` | element ถัดไป/ก่อนหน้าในระดับเดียวกัน (sibling) ที่ตรงกับ selector |
| `find <selector>` | element ลูกตัวแรกที่ตรงกับ selector ภายในตัวเอง |

**ตัวอย่างที่ใช้บ่อยที่สุดในงานจริง**: ปุ่มลบที่อยู่ข้างในการ์ด ต้องการลบ
การ์ดทั้งใบที่ครอบตัวเองอยู่ (ไม่ใช่แค่ปุ่ม) — ใช้ `closest`:

```html
<div class="comment-item" id="comment-42">
    <p>ความคิดเห็นตัวอย่าง</p>
    <button hx-delete="{% url 'blog:comment_delete' 42 %}"
            hx-target="closest .comment-item"
            hx-swap="outerHTML"
            hx-confirm="ยืนยันการลบความคิดเห็นนี้?">
        ลบ
    </button>
</div>
```

`hx-confirm` เป็น attribute เสริมที่สั่งให้ browser แสดง dialog ยืนยันก่อน
ยิง request จริง (ใช้ `window.confirm()` มาตรฐานของ browser เบื้องหลัง) —
ถ้าผู้ใช้กด "ยกเลิก" HTMX จะไม่ยิง request เลย

### 523.3 `hx-swap`: ควบคุมว่า "แทรกผลลัพธ์อย่างไร"

| ค่า | ความหมาย |
|---|---|
| `innerHTML` (ค่าเริ่มต้น) | แทนที่เนื้อหาข้างในของ target ทั้งหมด |
| `outerHTML` | แทนที่ทั้ง element ของ target เอง (รวม tag เปิด-ปิดของมันด้วย) |
| `beforebegin` | แทรกเป็น sibling ก่อนหน้า target (นอก target) |
| `afterbegin` | แทรกเป็นลูกคนแรกข้างในสุดของ target |
| `beforeend` | แทรกเป็นลูกคนสุดท้ายข้างในสุดของ target (ใช้บ่อยกับ "เพิ่มรายการต่อท้ายลิสต์") |
| `afterend` | แทรกเป็น sibling ถัดไปหลัง target (นอก target) |
| `delete` | ลบ target ทิ้งไปเลย ไม่สนใจเนื้อหา response ที่ได้กลับมา |
| `none` | ไม่แทรกอะไรเลย (ใช้เมื่อ request มีผลข้างเคียงอื่นที่ต้องการ เช่น out-of-band swap ในขั้นตอนที่ 528) |

**ตัวอย่าง**: เพิ่ม `hx-swap` modifier `swap:200ms` เพื่อหน่วงเวลาก่อนแทรก
จริง (ให้ CSS transition เล่นก่อน) และ `settle:100ms` (หน่วงก่อนลบ class
`htmx-swapping`/`htmx-settling` ที่ HTMX ใส่ให้อัตโนมัติระหว่าง transition):

```html
<button hx-delete="{% url 'blog:comment_delete' comment.id %}"
        hx-target="closest .comment-item"
        hx-swap="outerHTML swap:200ms"
        hx-confirm="ยืนยันการลบ?">
    ลบ
</button>
```

```css
/* static/blog/css/htmx-transitions.css */
.comment-item.htmx-swapping {
    opacity: 0;
    transition: opacity 200ms ease-out;
}
```

HTMX ใส่ class `htmx-swapping` ให้ target โดยอัตโนมัติทันทีที่เริ่มกระบวนการ
แทนที่ (ก่อนแทนที่จริง 200ms ตามที่ระบุ) ทำให้เขียน CSS transition ธรรมดา
(ไม่ต้องพึ่ง JavaScript animation library ใด ๆ) ให้การ์ดคอมเมนต์ค่อย ๆ จางหาย
ก่อนถูกลบออกจาก DOM จริง

### 523.4 ตัวอย่างผสมทั้ง 3 Attribute: ปุ่ม Toggle เผยแพร่/ซ่อนบทความ (สำหรับ Staff)

```python
# blog/views.py
from django.contrib.auth.decorators import login_required, user_passes_test


@login_required
@user_passes_test(lambda u: u.is_staff)
@require_POST
def post_toggle_publish_view(request, slug):
    post = get_object_or_404(Post, slug=slug)
    post.is_published = not post.is_published
    post.save(update_fields=['is_published'])
    return render(request, 'blog/partials/publish_toggle.html', {'post': post})
```

```html
<!-- blog/templates/blog/partials/publish_toggle.html -->
<button hx-post="{% url 'blog:toggle_publish' post.slug %}"
        hx-target="this"
        hx-swap="outerHTML"
        class="badge {% if post.is_published %}badge--published{% else %}badge--draft{% endif %}">
    {% if post.is_published %}เผยแพร่แล้ว (คลิกเพื่อซ่อน){% else %}ฉบับร่าง (คลิกเพื่อเผยแพร่){% endif %}
</button>
```

รูปแบบนี้ ("ปุ่มที่ส่งตัวเองเป็น target แล้วแทนที่ตัวเองด้วยเวอร์ชันใหม่")
เป็น pattern ที่พบบ่อยที่สุดของ HTMX สำหรับ toggle/state สั้น ๆ — View ไม่
ต้องรู้เรื่อง "หน้าที่เรียกมันมาจากไหน" เลย มันแค่รับ action, อัปเดตข้อมูล,
แล้ว render ปุ่มเวอร์ชันล่าสุดกลับไปเสมอ

---

## ขั้นตอนที่ 524: Partial Template Rendering — View ที่คืน Fragment โดยเฉพาะ

### 524.1 ปัญหา: View เดียวต้องรับใช้ทั้ง "โหลดหน้าเต็ม" และ "โหลดจาก HTMX"

ตัวอย่างในขั้นตอนที่ 522-523 ทั้งหมดเป็น View ที่**สร้างมาเพื่อ HTMX โดยเฉพาะ**
(คืน fragment เสมอ) แต่ในงานจริงมักเจอสถานการณ์ที่ **View เดียวกัน** ต้อง
ทำงานได้ทั้งสองแบบ:

- ถ้าผู้ใช้เข้า URL `/blog/?q=django` ตรง ๆ ทาง browser (พิมพ์ URL เอง หรือ
  bookmark ไว้) → ต้องได้ **หน้าเต็ม** (พร้อม `<html>`, navbar, footer ทุกอย่าง)
- ถ้า HTMX ยิง request เดียวกันมาจาก `hx-get` (จากช่องค้นหาในหน้า) → ต้อง
  ได้ **แค่ fragment** ของผลการค้นหา ไม่เอาทั้งหน้า (ไม่เช่นนั้นจะกลายเป็น
  หน้าเว็บซ้อนหน้าเว็บ)

### 524.2 วิธีตรวจจับ: HTMX ส่ง Header `HX-Request: true` มาด้วยเสมอ

ทุก request ที่ HTMX ยิงออกไปจะแนบ HTTP header `HX-Request: true` มาด้วย
เสมอโดยอัตโนมัติ (ไม่ต้องตั้งค่าอะไรเพิ่ม) — View จึงตรวจสอบ header นี้เพื่อ
เลือกว่าจะ render template ไหน:

```python
# blog/views.py
def post_list_view(request):
    query = request.GET.get('q', '').strip()
    posts = Post.objects.filter(is_published=True).order_by('-created_at')
    if query:
        posts = posts.filter(title__icontains=query)

    context = {'posts': posts, 'query': query}

    # ตรวจสอบ header ที่ HTMX แนบมาเองทุกครั้ง (วิธีตรวจแบบ manual —
    # ขั้นตอนที่ 529 จะแนะนำวิธีที่สะอาดกว่านี้ผ่าน django-htmx)
    if request.headers.get('HX-Request') == 'true':
        return render(request, 'blog/partials/post_search_results.html', context)

    return render(request, 'blog/post_list.html', context)
```

```html
<!-- blog/templates/blog/post_list.html -->
{% extends 'base.html' %}

{% block content %}
<h1>บทความทั้งหมด</h1>

<input type="search" name="q" value="{{ query }}"
       hx-get="{% url 'blog:list' %}"
       hx-trigger="keyup changed delay:400ms, search"
       hx-target="#post-results"
       hx-swap="innerHTML"
       hx-push-url="true">

<div id="post-results">
    {% include 'blog/partials/post_search_results.html' %}
</div>
{% endblock %}
```

สังเกตว่า `post_list.html` (หน้าเต็ม) กับ View ใช้ `{% include %}` (ทบทวน
Part 008/028) ดึง `post_search_results.html` มาแสดงตอนโหลดหน้าแรก **ไฟล์
เดียวกันเป๊ะ**กับที่ View คืนให้ตอน HTMX ยิงมา — นี่คือการแก้ปัญหาข้อ 521.1
(เขียนโครง HTML ซ้ำ 2 ที่) ได้อย่างสมบูรณ์: มี partial template ไฟล์เดียว
ที่ทั้ง full-page render และ HTMX fragment render ใช้ร่วมกัน

`hx-push-url="true"` สั่งให้ HTMX อัปเดต URL ในแถบ address bar ของ browser
ให้ตรงกับ URL ที่เพิ่งยิงไป (เช่น `/blog/?q=django`) ทำให้กด "ย้อนกลับ"
(back button) หรือ refresh หน้าแล้วยังเห็นผลค้นหาเดิมได้ถูกต้อง

### 524.3 จัดโครงสร้างโฟลเดอร์ Partial ตามธรรมเนียมจาก Part 028

ต่อยอดจากโครงสร้าง Template Tree ที่แนะนำใน Part 028 ข้อ 277 (`base/`,
`partials/`, `emails/`) — โฟลเดอร์ `partials/` เดิมที่มีไว้สำหรับ
`{% include %}` ทั่วไป ตอนนี้จะกลายเป็นที่เก็บ **HTMX fragment view
templates** ไปพร้อมกันด้วย เพราะโดยธรรมชาติแล้วมันคือสิ่งเดียวกัน:

```
blog/templates/blog/
├── post_list.html              ← หน้าเต็ม
├── post_detail.html            ← หน้าเต็ม
└── partials/
    ├── popular_posts.html       ← ทั้ง {% include %} และ HTMX fragment
    ├── post_search_results.html ← ทั้ง {% include %} และ HTMX fragment
    ├── like_button.html         ← HTMX fragment เท่านั้น (ไม่มีที่ include แบบ static)
    ├── publish_toggle.html      ← HTMX fragment เท่านั้น
    ├── comment_list.html        ← HTMX fragment (ขั้นตอนที่ 530)
    └── comment_row.html         ← HTMX fragment (ขั้นตอนที่ 530)
```

### 524.4 Helper Function เพื่อลดโค้ดซ้ำในการเช็ค `HX-Request` ทุก View

เมื่อ pattern "เช็ค header แล้วเลือก template" เกิดขึ้นซ้ำหลาย View ควรดึง
ออกมาเป็นฟังก์ชันช่วยกลาง:

```python
# blog/utils.py
def render_htmx_aware(request, full_template, partial_template, context):
    """เลือก template ให้อัตโนมัติตามว่า request มาจาก HTMX หรือไม่"""
    from django.shortcuts import render
    template = partial_template if request.headers.get('HX-Request') == 'true' else full_template
    return render(request, template, context)
```

```python
# blog/views.py
from .utils import render_htmx_aware


def post_list_view(request):
    query = request.GET.get('q', '').strip()
    posts = Post.objects.filter(is_published=True).order_by('-created_at')
    if query:
        posts = posts.filter(title__icontains=query)

    return render_htmx_aware(
        request,
        full_template='blog/post_list.html',
        partial_template='blog/partials/post_search_results.html',
        context={'posts': posts, 'query': query},
    )
```

ฟังก์ชันช่วยนี้เป็นต้นแบบง่าย ๆ ของสิ่งที่ package `django-htmx` มอบให้แบบ
สำเร็จรูปในขั้นตอนที่ 529 — เขียนเองก่อนเพื่อให้เข้าใจกลไกเบื้องหลังอย่าง
ถ่องแท้ ก่อนไปใช้เครื่องมือที่ทำให้สะดวกขึ้น

---

## ขั้นตอนที่ 525: HTMX ร่วมกับ Django Forms — Inline Validation

### 525.1 เป้าหมาย: ฟอร์มคอมเมนต์ที่ Validate โดยไม่ Reload หน้า

ใช้ `CommentForm` (`ModelForm`) จาก Part 026 ข้อ 251.3 ตรง ๆ — สิ่งที่ต้อง
เพิ่มมีแค่ View กับ Template ที่ทำงานร่วมกับ HTMX เท่านั้น ไม่ต้องแก้ Form
class เลยแม้แต่บรรทัดเดียว:

```python
# blog/forms.py (จาก Part 026 — ใช้ต่อได้ทันที ไม่ต้องแก้ไข)
from django import forms
from .models import Comment


class CommentForm(forms.ModelForm):
    class Meta:
        model = Comment
        fields = ['author', 'text']
```

### 525.2 หลักการสำคัญ: HTMX แทรกผลลัพธ์เมื่อ Response เป็น 2xx เท่านั้น (โดยค่าเริ่มต้น)

ค่าเริ่มต้นของ HTMX คือ **swap เนื้อหาเฉพาะเมื่อ HTTP status เป็น 2xx หรือ
3xx เท่านั้น** — ถ้า View คืน 4xx/5xx แล้วไม่ได้ตั้งค่าเพิ่มเติม HTMX จะ
**ไม่แทรก HTML ที่ได้ลงหน้าเว็บเลย** (เพียงยิง event `htmx:responseError`
ออกมาให้ดักจับเองถ้าต้องการ)

ด้วยเหตุนี้ pattern มาตรฐานสำหรับฟอร์มที่ต้องแสดง error กลับมาในหน้าเดิมคือ
**คืน HTTP 200 เสมอ ไม่ว่าฟอร์มจะ valid หรือไม่** แล้ว render ฟอร์มพร้อม
error กลับไปให้ HTMX แทรกแทน — Django Form object ที่มี error อยู่แล้ว
(`form.errors`) render ออกมาเป็น HTML ที่แสดงข้อความ error ให้เองอัตโนมัติ
ตามที่เรียนมาตั้งแต่ Part 025:

```python
# blog/views.py
from django.shortcuts import get_object_or_404, render
from django.views.decorators.http import require_http_methods
from .models import Post
from .forms import CommentForm


@require_http_methods(['GET', 'POST'])
def comment_create_view(request, slug):
    post = get_object_or_404(Post, slug=slug)

    if request.method == 'POST':
        form = CommentForm(request.POST)
        if form.is_valid():
            comment = form.save(commit=False)
            comment.post = post
            comment.save()
            # สำเร็จ: คืนแถวคอมเมนต์ใหม่ + ฟอร์มเปล่าสำหรับพิมพ์ต่อ (ดู 525.3)
            return render(request, 'blog/partials/comment_created.html', {
                'comment': comment,
                'form': CommentForm(),   # ฟอร์มเปล่าใหม่ ไม่ใช่ฟอร์มเดิมที่มีค่าเก่าค้างอยู่
                'post': post,
            })
        # ไม่ valid: คืน HTTP 200 พร้อมฟอร์มเดิมที่มี error ติดมาด้วย
        return render(request, 'blog/partials/comment_form.html', {
            'form': form,
            'post': post,
        }, status=200)

    return render(request, 'blog/partials/comment_form.html', {
        'form': CommentForm(),
        'post': post,
    })
```

### 525.3 Template ของฟอร์มที่แสดง Error ในตำแหน่งเดิมทุกครั้ง

```html
<!-- blog/templates/blog/partials/comment_form.html -->
<form id="comment-form"
      hx-post="{% url 'blog:comment_create' post.slug %}"
      hx-target="#comment-form-wrapper"
      hx-swap="innerHTML">
    {{ form.non_field_errors }}
    <div class="form-group">
        {{ form.author.label_tag }}
        {{ form.author }}
        {{ form.author.errors }}
    </div>
    <div class="form-group">
        {{ form.text.label_tag }}
        {{ form.text }}
        {{ form.text.errors }}
    </div>
    <button type="submit">ส่งความคิดเห็น</button>
</form>
```

```html
<!-- blog/templates/blog/partials/comment_created.html -->
<div id="comment-form-wrapper" hx-swap-oob="innerHTML">
    {% include 'blog/partials/comment_form.html' %}
</div>
```

จุดสำคัญที่ทำให้ pattern นี้ทำงานถูกต้อง:

1. `hx-target="#comment-form-wrapper"` และ `hx-swap="innerHTML"` บนตัวฟอร์ม
   หมายความว่า **ไม่ว่า response จะเป็นฟอร์ม (มี error) หรือฟอร์มเปล่าใหม่
   (สำเร็จ) ก็ตาม ผลลัพธ์จะถูกแทรกกลับเข้าตำแหน่งเดิมเสมอ** — เพราะทั้งสอง
   กรณี Django คืนโครงสร้างที่ครอบด้วย `id="comment-form-wrapper"` เหมือนกัน
   ทุกประการ (ทั้งจาก template `comment_form.html` โดยตรง หรือผ่าน
   `comment_created.html` ที่ include มันเข้าไปอีกที)
2. เมื่อ validation ล้มเหลว ฟอร์มที่ render กลับมาคือ **ฟอร์มเดิมที่มีค่า
   ที่ผู้ใช้กรอกไว้ครบ** (เพราะ `form = CommentForm(request.POST)` bound
   ด้วยข้อมูลเดิม) ผู้ใช้จึงไม่ต้องพิมพ์ใหม่ทั้งหมด เห็นเฉพาะข้อความ error
   ใต้ field ที่ผิดพลาดเท่านั้น — ประสบการณ์เดียวกับฟอร์มทั่วไปที่เรียนมา
   ตั้งแต่ Part 025 เพียงแต่ไม่มีการ reload หน้าเว็บเกิดขึ้นเลย

### 525.4 แสดงคอมเมนต์ใหม่ในลิสต์ทันทีด้วย `hx-swap-oob` (Out-of-Band Swap)

สังเกตว่ากรณีสำเร็จ เราต้องการทำ **2 อย่างพร้อมกัน**: (1) เคลียร์ฟอร์มกลับ
เป็นค่าว่างในตำแหน่งเดิม และ (2) เพิ่มการ์ดคอมเมนต์ใหม่เข้าไปในลิสต์ที่อยู่
คนละตำแหน่งของหน้า — HTMX รองรับการอัปเดต "หลายจุดพร้อมกันจาก response
เดียว" ผ่านเทคนิคที่เรียกว่า **Out-of-Band (OOB) Swap**:

```html
<!-- blog/templates/blog/partials/comment_created.html (เวอร์ชันสมบูรณ์) -->
<!-- ส่วนที่ 1: เคลียร์ฟอร์ม (เป็นผลลัพธ์หลักที่ hx-target ของฟอร์มรับไป) -->
{% include 'blog/partials/comment_form.html' %}

<!-- ส่วนที่ 2: การ์ดคอมเมนต์ใหม่ ส่งไปแทรกที่ #comment-list แบบ out-of-band -->
<div id="comment-list" hx-swap-oob="afterbegin">
    {% include 'blog/partials/comment_row.html' with comment=comment %}
</div>
```

`hx-swap-oob="afterbegin"` บน `<div id="comment-list">` บอก HTMX ว่า:
"ไม่ว่า `hx-target` ของ request นี้จะชี้ไปที่ไหน ให้เอา element ที่มี
`id="comment-list"` ในตัว response นี้ไปหา element ที่มี `id` เดียวกันใน
หน้าเว็บปัจจุบัน แล้วแทรกเนื้อหาข้างในเข้าไปที่ตำแหน่ง `afterbegin`
(ด้านบนสุดของลิสต์) ต่างหาก โดยไม่เกี่ยวกับ `hx-target` หลักเลย"

นี่คือเหตุผลที่ response เดียวสามารถ "เคลียร์ฟอร์ม" และ "เพิ่มคอมเมนต์เข้า
ลิสต์" พร้อมกันได้ในคราวเดียว — เทคนิคนี้จะกลับมาใช้อีกครั้งอย่างเข้มข้นใน
ขั้นตอนที่ 528 (Messages Framework) และขั้นตอนที่ 530 (Capstone)

---

## ขั้นตอนที่ 526: `hx-boost` — เปลี่ยนทั้งเว็บให้เป็น SPA-Like

### 526.1 ปัญหาที่ `hx-boost` แก้: ลิงก์และฟอร์มธรรมดายัง Reload ทั้งหน้าอยู่

ทุกตัวอย่างที่ผ่านมาต้องเติม `hx-get`/`hx-post` ให้ element ทีละตัว — แต่
ลิงก์ (`<a href="...">`) และฟอร์ม (`<form action="...">`) ธรรมดาที่ไม่มี
attribute ของ HTMX เลยยังคง**reload ทั้งหน้าตามพฤติกรรมมาตรฐานของ HTML**
เหมือนเดิมทุกประการ — ถ้าอยากให้ **ทั้งเว็บไซต์** รู้สึกลื่นไหลแบบ SPA
(ไม่มีการกระพริบขาวตอนเปลี่ยนหน้า) โดยไม่ต้องเติม `hx-get` ให้ลิงก์ทุกอันทั่ว
เว็บ ให้ใช้ **`hx-boost`**

### 526.2 วิธีใช้: เติม Attribute เดียวที่ `<body>`

```html
<!-- templates/base.html -->
<body hx-boost="true">
    ...
</body>
```

เมื่อใส่ `hx-boost="true"` ที่ element ใด HTMX จะ **"ครอบ" (progressively
enhance)** ทุก `<a>` และ `<form>` ที่อยู่ข้างในโดยอัตโนมัติ (รวมถึง element
ที่ถูกเพิ่มเข้ามาทีหลังผ่าน HTMX swap อื่นด้วย) ให้ทำงานแบบนี้แทนพฤติกรรม
ปกติ:

1. คลิกลิงก์ → HTMX ยิง `GET` แบบ AJAX ไปที่ `href` แทนการ navigate เต็มรูป
2. Submit ฟอร์ม → HTMX ยิง `GET`/`POST` แบบ AJAX ไปที่ `action` แทน
3. เมื่อได้ response กลับมา (เป็นหน้า HTML เต็มตามปกติ ไม่ต้องเปลี่ยนอะไร
   ฝั่ง View เลย) HTMX จะดึงเฉพาะเนื้อหาใน `<body>` ออกมาแทนที่ `<body>`
   เดิมของหน้าปัจจุบัน (ค่าเริ่มต้น `hx-target` ของ boost คือ `body`,
   `hx-swap` คือ `innerHTML` ของ body)
4. อัปเดต URL ใน address bar และ browser history ให้อัตโนมัติ (เทียบเท่า
   `hx-push-url="true"` ที่ตั้งค่าไว้ให้เป็นค่าเริ่มต้นเมื่อใช้ boost)

**สิ่งสำคัญที่สุด**: View ฝั่ง Django **ไม่ต้องแก้ไขอะไรเลยแม้แต่บรรทัดเดียว**
— render หน้าเต็มปกติเหมือนที่เขียนมาตั้งแต่ Part 001 ทุกประการ HTMX เป็น
ฝ่ายจัดการตัดเฉพาะส่วน `<body>` ออกมาเองฝั่ง client

### 526.3 ปรับ Target ให้ Boost แทนที่เฉพาะบางส่วน (ไม่ใช่ทั้ง Body)

ถ้าต้องการให้ boost แทนที่แค่ส่วน "เนื้อหาหลัก" ไม่แตะ navbar/sidebar ที่
ควรอยู่นิ่ง (ทำให้ลื่นไหลกว่าการแทนที่ทั้ง `<body>` ทุกครั้ง) ให้ใช้
`hx-target` และ `hx-select` ร่วมกัน:

```html
<!-- templates/base.html -->
<body hx-boost="true" hx-target="#main-content" hx-select="#main-content" hx-swap="innerHTML">
    <nav>...</nav>
    <main id="main-content">
        {% block content %}{% endblock %}
    </main>
    <footer>...</footer>
</body>
```

`hx-select="#main-content"` บอก HTMX ว่า "แม้ response ที่ได้กลับมาจะเป็น
หน้า HTML เต็ม (มี `<html>`, `<nav>`, `<footer>` ครบ) ให้**เลือกเฉพาะ**
element ที่มี `id="main-content"` จากในนั้นมาใช้เท่านั้น" ผลลัพธ์คือ navbar
และ footer จะไม่กระพริบหรือถูกสร้างใหม่ทุกครั้งที่เปลี่ยนหน้าเลย

### 526.4 ยกเว้นบาง Element ออกจาก Boost

บางลิงก์ (เช่น ลิงก์ไปหน้า external, ลิงก์ดาวน์โหลดไฟล์, ลิงก์ logout ที่
ต้องการ full page reload จริง ๆ เพื่อเคลียร์ state ฝั่ง client ทั้งหมด)
ไม่ควรถูก boost ใช้ `hx-boost="false"` เจาะจงเป็นราย element:

```html
<a href="https://github.com" hx-boost="false">GitHub (เปิดแบบปกติ)</a>
<a href="{% url 'media_download' file.id %}" hx-boost="false" download>ดาวน์โหลดไฟล์</a>
<form action="{% url 'logout' %}" method="post" hx-boost="false">
    {% csrf_token %}
    <button type="submit">ออกจากระบบ</button>
</form>
```

### 526.5 ตารางสรุป: `hx-boost` เทียบกับการเขียน `hx-get`/`hx-post` เอง

| ประเด็น | เขียน `hx-get`/`hx-post` เอง (ขั้นตอนที่ 522-525) | `hx-boost="true"` |
|---|---|---|
| ต้องเติม attribute กี่จุด | ทุกลิงก์/ฟอร์มที่ต้องการ | **จุดเดียว** (เช่นที่ `<body>`) ครอบทั้งเว็บ |
| View ต้องคืน fragment ไหม | ส่วนใหญ่ต้อง (เพื่อประสิทธิภาพ) | **ไม่ต้อง** — คืนหน้าเต็มปกติได้เลย |
| เหมาะกับ | ปฏิสัมพันธ์เฉพาะจุด (like, comment, search) | การนำทางทั่วเว็บ (เปลี่ยนหน้า, submit ฟอร์มปกติ) |
| ควบคุมละเอียดแค่ไหน | ละเอียดมาก (เลือก target/swap/trigger เองทุกจุด) | หยาบกว่า (ใช้ค่าเริ่มต้นเดียวกันทั้งเว็บ เว้นแต่ override) |

**คำแนะนำของหลักสูตรนี้**: ใช้ **`hx-boost` เป็นค่าเริ่มต้นของทั้งเว็บ**
เพื่อให้การเปลี่ยนหน้าทั่วไปลื่นไหลโดยแทบไม่ต้องเขียนอะไรเพิ่ม แล้วค่อยเติม
`hx-get`/`hx-post`/`hx-trigger`/`hx-target` แบบละเอียดเฉพาะจุดที่ต้องการ
พฤติกรรมพิเศษกว่าปกติ (เช่น like button, live search, infinite scroll)
— นี่คือแนวทางที่โปรเจกต์ระดับ production ที่ใช้ HTMX ส่วนใหญ่เลือกใช้

---

## ขั้นตอนที่ 527: Infinite Scroll ด้วย `hx-trigger="revealed"`

### 527.1 แนวคิด: โหลดหน้าถัดไปเมื่อผู้ใช้เลื่อนมาถึงท้ายลิสต์

**Infinite Scroll** คือรูปแบบการแบ่งหน้า (pagination) ที่โหลดข้อมูลชุดถัดไป
โดยอัตโนมัติเมื่อผู้ใช้เลื่อนหน้าจอมาถึงจุดสุดท้ายของลิสต์ปัจจุบัน แทนที่จะ
ให้กดปุ่ม "หน้าถัดไป" เอง — HTMX ทำสิ่งนี้ได้ด้วย trigger พิเศษชื่อ
**`revealed`** ที่จะยิง request ทันทีที่ element นั้นเลื่อนเข้ามาอยู่ใน
viewport (มองเห็นได้บนหน้าจอ) เป็นครั้งแรก

### 527.2 View ที่รองรับ Pagination แบบหน้าแยก

```python
# blog/views.py
from django.core.paginator import Paginator


def post_list_infinite_view(request):
    all_posts = Post.objects.filter(is_published=True).order_by('-created_at')
    paginator = Paginator(all_posts, 10)   # หน้าละ 10 บทความ
    page_number = request.GET.get('page', 1)
    page_obj = paginator.get_page(page_number)

    if request.headers.get('HX-Request') == 'true':
        return render(request, 'blog/partials/post_page.html', {'page_obj': page_obj})
    return render(request, 'blog/post_list_infinite.html', {'page_obj': page_obj})
```

```python
# blog/urls.py
path('infinite/', views.post_list_infinite_view, name='list_infinite'),
```

### 527.3 Template: Sentinel Element ที่ Trigger การโหลดหน้าถัดไป

```html
<!-- blog/templates/blog/post_list_infinite.html -->
{% extends 'base.html' %}

{% block content %}
<h1>บทความทั้งหมด (Infinite Scroll)</h1>
<div id="post-container">
    {% include 'blog/partials/post_page.html' %}
</div>
{% endblock %}
```

```html
<!-- blog/templates/blog/partials/post_page.html -->
{% for post in page_obj %}
    <article class="post-card">
        <h2><a href="{% url 'blog:detail' post.slug %}">{{ post.title }}</a></h2>
        <p>{{ post.content|truncatewords:30 }}</p>
    </article>
{% endfor %}

{% if page_obj.has_next %}
    <div hx-get="{% url 'blog:list_infinite' %}?page={{ page_obj.next_page_number }}"
         hx-trigger="revealed"
         hx-swap="outerHTML"
         hx-indicator="#loading-spinner">
        <div id="loading-spinner" class="htmx-indicator">กำลังโหลดเพิ่มเติม...</div>
    </div>
{% else %}
    <p class="end-of-list">— แสดงบทความครบทุกรายการแล้ว —</p>
{% endif %}
```

หลักการทำงาน:

1. `<div>` สุดท้ายในแต่ละหน้าทำหน้าที่เป็น **sentinel** (ตัวตรวจจับ) — ยัง
   ไม่มีอะไรอยู่ในนั้นตอนแรกนอกจาก loading indicator
2. เมื่อผู้ใช้เลื่อนหน้าจอลงมาจนกระทั่ง sentinel นี้ **เลื่อนเข้ามาปรากฏใน
   viewport** (`revealed`) HTMX จะยิง `hx-get` ไปขอหน้าถัดไปทันที
3. `hx-swap="outerHTML"` แทนที่ sentinel เดิมทั้งก้อนด้วยผลลัพธ์ใหม่ที่ได้
   — ผลลัพธ์ใหม่ก็คือ `post_page.html` fragment ของหน้าถัดไป ซึ่ง**มี
   sentinel ตัวใหม่ของตัวเองอยู่ท้ายสุดอีกครั้ง** (ถ้ายังมีหน้าถัดไปต่อ) —
   ทำให้กระบวนการนี้เกิดซ้ำไปเรื่อย ๆ จนกว่าจะถึงหน้าสุดท้าย (`has_next`
   เป็น `False` แล้วจึงแสดงข้อความ "แสดงครบทุกรายการแล้ว" แทน sentinel)

### 527.4 `hx-indicator`: แสดง Loading State ระหว่างรอ Response

```css
/* static/blog/css/htmx-indicator.css */
.htmx-indicator {
    display: none;
}
.htmx-request .htmx-indicator {
    display: block;
}
/* กรณี indicator เป็น element เดียวกับที่มี class htmx-request เอง */
.htmx-request.htmx-indicator {
    display: block;
}
```

`hx-indicator="#loading-spinner"` บอก HTMX ว่าระหว่างที่ request กำลังรอ
response อยู่ ให้เติม class **`htmx-request`** ให้กับ element ที่ระบุ
โดยอัตโนมัติ (และลบออกทันทีที่ response กลับมา) — CSS ด้านบนใช้ประโยชน์
จาก class นี้ทำให้ spinner **ซ่อนอยู่ตามปกติ และโผล่ขึ้นมาเฉพาะช่วงที่
กำลังโหลดเท่านั้น** โดยไม่ต้องเขียน JavaScript แสดง/ซ่อนเอง

---

## ขั้นตอนที่ 528: ผสาน HTMX กับ Django Messages Framework

### 528.1 ทบทวนปัญหา: Messages Framework ออกแบบมาสำหรับ Full-Page Redirect

**Django Messages Framework** (`django.contrib.messages`) ทำงานผ่านรูปแบบ
`redirect` → `render` แบบดั้งเดิม: View เพิ่มข้อความผ่าน `messages.success()`
แล้ว `redirect()` ไปหน้าอื่น หน้าที่ redirect ไปถึงจะดึงข้อความออกมาแสดงผ่าน
`{% for message in messages %}` ในเทมเพลต — ปัญหาคือ **request ที่มาจาก
HTMX ส่วนใหญ่ไม่ redirect** (มันแค่ swap fragment เข้าที่เดิม) ทำให้ข้อความ
messages ที่เพิ่มไว้ไม่มีโอกาสถูกแสดงเลยถ้าไม่ทำอะไรเพิ่ม

### 528.2 วิธีแก้: ส่ง Messages Fragment กลับมาด้วย Out-of-Band Swap

ใช้เทคนิคเดียวกับข้อ 525.4 — เตรียม container สำหรับ messages ไว้ใน
`base.html` ที่มี `id` ตายตัว แล้วให้ทุก View ที่เพิ่ม message ส่ง fragment
ของ container นั้นกลับมาแบบ `hx-swap-oob` ควบคู่กับผลลัพธ์หลัก:

```html
<!-- templates/base.html -->
<body hx-boost="true">
    <div id="messages-container">
        {% include 'partials/messages.html' %}
    </div>
    {% block content %}{% endblock %}
</body>
```

```html
<!-- templates/partials/messages.html -->
{% for message in messages %}
    <div class="alert alert--{{ message.tags }}" role="alert">
        {{ message }}
        <button class="alert__close" onclick="this.parentElement.remove()">×</button>
    </div>
{% endfor %}
```

```python
# blog/views.py
from django.contrib import messages
from django.template.loader import render_to_string


@require_POST
def comment_delete_view(request, pk):
    comment = get_object_or_404(Comment, pk=pk)
    if not request.user.is_staff:
        return HttpResponseForbidden('ไม่มีสิทธิ์ลบความคิดเห็นนี้')

    comment.delete()
    messages.success(request, 'ลบความคิดเห็นเรียบร้อยแล้ว')

    # fragment หลัก: ลบการ์ดคอมเมนต์ออกจาก DOM (hx-swap="outerHTML" ฝั่ง client จะรับค่านี้)
    # fragment เสริม (OOB): แสดง message ที่เพิ่งเพิ่มไว้
    messages_html = render_to_string('partials/messages.html', {}, request=request)
    return HttpResponse(
        f'<div id="messages-container" hx-swap-oob="innerHTML">{messages_html}</div>'
    )
```

```html
<!-- blog/templates/blog/partials/comment_row.html -->
<div class="comment-item" id="comment-{{ comment.pk }}">
    <strong>{{ comment.author }}</strong>
    <p>{{ comment.text }}</p>
    {% if request.user.is_staff %}
    <button hx-delete="{% url 'blog:comment_delete' comment.pk %}"
            hx-target="closest .comment-item"
            hx-swap="outerHTML swap:200ms"
            hx-confirm="ยืนยันการลบความคิดเห็นนี้?">
        ลบ
    </button>
    {% endif %}
</div>
```

เมื่อผู้ใช้กด "ลบ": HTMX ยิง `DELETE` request → View ลบข้อมูลจริง เพิ่ม
message → คืน response ที่มีแค่ `<div id="messages-container" hx-swap-oob=...>`
(ไม่มีเนื้อหาการ์ดคอมเมนต์เหลืออยู่เลย เพราะไม่ต้องการให้มัน "แทนที่" ตัว
`hx-target` หลัก) — HTMX เห็นว่า response ว่างเปล่าสำหรับ `hx-target` หลัก
จึงลบการ์ดคอมเมนต์เดิมออก (ตาม `hx-swap="outerHTML"` ที่แทนที่ด้วยค่าว่าง)
พร้อมกับแทรก message OOB เข้า `#messages-container` ไปพร้อมกันในการ swap
เดียว

### 528.3 ตารางสรุป: Pattern การผสาน Messages กับ HTMX

| สถานการณ์ | วิธีจัดการ |
|---|---|
| View HTMX ทำสำเร็จ ต้องการแสดง success message | render `messages.html` แล้วครอบด้วย `hx-swap-oob` ส่งกลับไปพร้อม fragment หลัก |
| View HTMX ทำสำเร็จ และลบ element ออกจากหน้าด้วย | คืน response ว่างเปล่าสำหรับ `hx-target` (ทำให้ `outerHTML` แทนที่ด้วยความว่าง = ลบทิ้ง) + แนบ OOB message |
| View ปกติ (ไม่ใช่ HTMX) ทำสำเร็จ | ใช้ `redirect()` ตามปกติแบบดั้งเดิมที่เรียนมาตั้งแต่ Part 010 — ไม่ต้องเปลี่ยนอะไร |
| ต้องการ auto-dismiss message หลัง 3 วินาที | เพิ่ม `hx-trigger="load delay:3s"` กับ `hx-swap="outerHTML"` ที่คืน string ว่างบน element ของ message เอง (ไม่ต้องใช้ JavaScript setTimeout) |

---

## ขั้นตอนที่ 529: แนะนำ Package `django-htmx`

### 529.1 ทำไมควรใช้ Package แทนการเช็ค Header เอง

ขั้นตอนที่ 524 สอนให้เช็ค `request.headers.get('HX-Request') == 'true'` เอง
ซึ่งใช้งานได้ดี แต่ HTMX ยังมี header อื่นอีกหลายตัวที่มีประโยชน์
(`HX-Boosted`, `HX-Target`, `HX-Trigger`, `HX-Trigger-Name`, `HX-Current-URL`)
ถ้าต้องเช็คทุกตัวด้วยมือ โค้ดจะรกและเสี่ยงพิมพ์ชื่อ header ผิด — package
**`django-htmx`** (https://django-htmx.readthedocs.io) ห่อ header เหล่านี้
ไว้เป็น property ที่ใช้งานสะดวกผ่าน `request.htmx`

### 529.2 ติดตั้งและตั้งค่า

```bash
pip install django-htmx
pip freeze > requirements.txt
```

```python
# config/settings.py
INSTALLED_APPS = [
    # ...
    'django_htmx',
]

MIDDLEWARE = [
    # ...
    'django.contrib.sessions.middleware.SessionMiddleware',
    'django.middleware.common.CommonMiddleware',
    'django_htmx.middleware.HtmxMiddleware',   # เพิ่มบรรทัดนี้
    # ...
]
```

Middleware ตัวนี้เติม attribute `request.htmx` ให้กับทุก request ที่เข้ามา
(เป็น object ชนิด `HtmxDetails`) — ถ้า request ไม่ได้มาจาก HTMX,
`bool(request.htmx)` จะเป็น `False` และ property อื่น ๆ ทั้งหมดจะคืนค่า
`None`/`False` ตามความเหมาะสม

### 529.3 ตารางเปรียบเทียบ Header ดิบ vs `request.htmx`

| Header ดิบ | `request.htmx` property | ความหมาย |
|---|---|---|
| `HX-Request: true` | `request.htmx` (ใช้เป็น bool ได้ตรง ๆ) | request นี้มาจาก HTMX หรือไม่ |
| `HX-Boosted: true` | `request.htmx.boosted` | request นี้มาจากลิงก์/ฟอร์มที่ถูก `hx-boost` หรือไม่ |
| `HX-Target: some-id` | `request.htmx.target` | ค่า `id` ของ target ที่ request นี้จะไปแทรก |
| `HX-Trigger: btn-id` | `request.htmx.trigger` | ค่า `id` ของ element ที่ trigger request นี้ |
| `HX-Trigger-Name: field-name` | `request.htmx.trigger_name` | ค่า `name` ของ element ที่ trigger (ถ้ามี) |
| `HX-Current-URL: /blog/` | `request.htmx.current_url` | URL ของหน้าที่ผู้ใช้อยู่ตอนยิง request นี้ |
| `HX-Prompt: ...` | `request.htmx.prompt` | ค่าที่ผู้ใช้พิมพ์ตอบ `hx-prompt` (ถ้ามีการใช้) |

### 529.4 เขียนใหม่ด้วย `django-htmx`: เทียบกับขั้นตอนที่ 524.4

```python
# blog/views.py — เวอร์ชันใช้ django-htmx (แทนที่ render_htmx_aware แบบเขียนเอง)
def post_list_view(request):
    query = request.GET.get('q', '').strip()
    posts = Post.objects.filter(is_published=True).order_by('-created_at')
    if query:
        posts = posts.filter(title__icontains=query)

    context = {'posts': posts, 'query': query}
    template = 'blog/partials/post_search_results.html' if request.htmx else 'blog/post_list.html'
    return render(request, template, context)
```

เทียบกับข้อ 524.2 ที่เขียน `request.headers.get('HX-Request') == 'true'`
โค้ดสั้นลง อ่านง่ายขึ้น และสื่อความหมายชัดเจนกว่ามาก — `django-htmx` ยังมี
`HttpResponseClientRedirect` และ `HttpResponseClientRefresh` ที่มีประโยชน์
สำหรับสั่งให้ browser ทำ full navigation/refresh จาก response ของ HTMX
request โดยตรง (ใช้ header `HX-Redirect`/`HX-Refresh` ของ HTMX เบื้องหลัง):

```python
# blog/views.py
from django_htmx.http import HttpResponseClientRedirect


@require_POST
def post_delete_view(request, slug):
    post = get_object_or_404(Post, slug=slug, author=request.user)
    post.delete()
    # สั่งให้ browser ทำ full-page redirect จริง ๆ (ไม่ใช่แค่ swap fragment)
    # มีประโยชน์เมื่อลบ object ที่หน้าปัจจุบันแสดงอยู่ ไม่มีที่ให้ swap ต่อแล้ว
    return HttpResponseClientRedirect(reverse('blog:list'))
```

### 529.5 ตัวอย่าง: ใช้ `request.htmx.target` เพื่อ Debug หรือปรับพฤติกรรมตาม Target

```python
def comment_create_view(request, slug):
    post = get_object_or_404(Post, slug=slug)
    # เผื่อฟอร์มเดียวกันถูกฝังไว้หลายตำแหน่งในหน้า (เช่น modal กับ inline)
    # และต้องปรับ template ที่คืนให้ต่างกันตาม target ที่เรียกมา
    if request.htmx.target == 'modal-comment-form':
        template_prefix = 'blog/partials/modal/'
    else:
        template_prefix = 'blog/partials/'
    # ... logic ที่เหลือเหมือนขั้นตอนที่ 525.2
```

> **คำแนะนำของหลักสูตรนี้**: ตั้งแต่จุดนี้เป็นต้นไป (รวมถึง Capstone ใน
> ขั้นตอนที่ 530) เราจะใช้ `request.htmx` แทนการเช็ค header ดิบเสมอ เพราะ
> เป็นวิธีที่โปรเจกต์ระดับ production ส่วนใหญ่ที่ใช้ HTMX กับ Django เลือกใช้
> จริง — การเขียนแบบ manual ในขั้นตอนที่ 522-528 มีไว้เพื่อให้เข้าใจกลไก
> เบื้องหลังก่อนเท่านั้น

---

## ขั้นตอนที่ 530: สรุปและแบบฝึกหัด

### 530.1 Capstone: แปลงระบบคอมเมนต์ทั้งระบบให้เป็น HTMX เต็มรูปแบบ

นำทุกเทคนิคจากขั้นตอนที่ 521-529 มาประกอบกันเป็นระบบคอมเมนต์ที่ **โพสต์และ
ลบคอมเมนต์ได้โดยไม่ต้อง reload หน้าเว็บแม้แต่ครั้งเดียว** ใช้ Model `Post`
และ `Comment` จาก Part 026 ตรง ๆ

**Model (ทบทวนจาก Part 026 — ไม่มีการแก้ไข):**

```python
# blog/models.py
class Comment(models.Model):
    post = models.ForeignKey(Post, on_delete=models.CASCADE, related_name='comments')
    author = models.CharField(max_length=100)
    text = models.TextField()
    created_at = models.DateTimeField(auto_now_add=True)

    class Meta:
        ordering = ['-created_at']

    def __str__(self):
        return f'ความคิดเห็นโดย {self.author} บน {self.post.title}'
```

**Form (ทบทวนจาก Part 026 ข้อ 251.3 — ไม่มีการแก้ไข):**

```python
# blog/forms.py
from django import forms
from .models import Comment


class CommentForm(forms.ModelForm):
    class Meta:
        model = Comment
        fields = ['author', 'text']
```

**Views ทั้งหมดของระบบคอมเมนต์ (ใช้ `django-htmx` ตามที่แนะนำในขั้นตอนที่ 529):**

```python
# blog/views.py
from django.contrib.auth.decorators import login_required
from django.shortcuts import get_object_or_404, render
from django.views.decorators.http import require_POST, require_http_methods
from django.http import HttpResponse
from .models import Post, Comment
from .forms import CommentForm


def post_detail_view(request, slug):
    post = get_object_or_404(Post, slug=slug)
    comments = post.comments.all()
    return render(request, 'blog/post_detail.html', {
        'post': post,
        'comments': comments,
        'form': CommentForm(),
    })


@require_http_methods(['POST'])
def comment_create_view(request, slug):
    post = get_object_or_404(Post, slug=slug)
    form = CommentForm(request.POST)

    if form.is_valid():
        comment = form.save(commit=False)
        comment.post = post
        comment.save()
        # OOB: เพิ่มการ์ดคอมเมนต์ใหม่เข้าลิสต์ + อัปเดตตัวนับ + เคลียร์ฟอร์ม (target หลัก)
        return render(request, 'blog/partials/comment_created.html', {
            'comment': comment,
            'post': post,
            'form': CommentForm(),
        })

    # ไม่ valid: คืน HTTP 200 พร้อมฟอร์มเดิม (ยังมีค่าที่ผู้ใช้กรอกไว้) และ error
    return render(request, 'blog/partials/comment_form.html', {
        'form': form,
        'post': post,
    })


@login_required
@require_POST
def comment_delete_view(request, pk):
    comment = get_object_or_404(Comment, pk=pk)

    if not request.user.is_staff:
        # ไม่มีสิทธิ์: คืนข้อความ error ผ่าน messages แบบ OOB โดยไม่ลบอะไรออกจากหน้า
        from django.contrib import messages
        from django.template.loader import render_to_string
        messages.error(request, 'คุณไม่มีสิทธิ์ลบความคิดเห็นนี้')
        messages_html = render_to_string('partials/messages.html', {}, request=request)
        return HttpResponse(
            f'<div id="messages-container" hx-swap-oob="innerHTML">{messages_html}</div>',
            status=200,
        )

    post = comment.post
    comment.delete()
    remaining_count = post.comments.count()

    # OOB: อัปเดตตัวนับจำนวนคอมเมนต์ + แสดง success message
    # target หลัก (การ์ดคอมเมนต์ที่ hx-delete ถูกยิงมา) คืนค่าว่าง = ถูกลบออกจาก DOM
    from django.contrib import messages
    from django.template.loader import render_to_string
    messages.success(request, 'ลบความคิดเห็นเรียบร้อยแล้ว')
    messages_html = render_to_string('partials/messages.html', {}, request=request)
    return HttpResponse(
        f'<div id="messages-container" hx-swap-oob="innerHTML">{messages_html}</div>'
        f'<span id="comment-count" hx-swap-oob="innerHTML">{remaining_count}</span>'
    )
```

**URLs:**

```python
# blog/urls.py
from django.urls import path
from . import views

app_name = 'blog'

urlpatterns = [
    path('<slug:slug>/', views.post_detail_view, name='detail'),
    path('<slug:slug>/comments/create/', views.comment_create_view, name='comment_create'),
    path('comments/<int:pk>/delete/', views.comment_delete_view, name='comment_delete'),
]
```

**Templates:**

```html
<!-- blog/templates/blog/post_detail.html -->
{% extends 'base.html' %}

{% block content %}
<article>
    <h1>{{ post.title }}</h1>
    {{ post.content|linebreaks }}
</article>

<section class="comments-section">
    <h2>ความคิดเห็น (<span id="comment-count">{{ comments.count }}</span>)</h2>

    <div id="comment-list">
        {% for comment in comments %}
            {% include 'blog/partials/comment_row.html' %}
        {% endfor %}
    </div>

    <div id="comment-form-wrapper">
        {% include 'blog/partials/comment_form.html' %}
    </div>
</section>
{% endblock %}
```

```html
<!-- blog/templates/blog/partials/comment_row.html -->
<div class="comment-item" id="comment-{{ comment.pk }}">
    <strong>{{ comment.author }}</strong>
    <time>{{ comment.created_at|date:"d M Y H:i" }}</time>
    <p>{{ comment.text }}</p>
    {% if request.user.is_staff %}
    <button hx-delete="{% url 'blog:comment_delete' comment.pk %}"
            hx-target="closest .comment-item"
            hx-swap="outerHTML swap:200ms"
            hx-confirm="ยืนยันการลบความคิดเห็นนี้?">
        ลบ
    </button>
    {% endif %}
</div>
```

```html
<!-- blog/templates/blog/partials/comment_form.html -->
<form hx-post="{% url 'blog:comment_create' post.slug %}"
      hx-target="#comment-form-wrapper"
      hx-swap="innerHTML"
      hx-on::after-request="if(event.detail.successful) this.reset()">
    {{ form.non_field_errors }}
    <div class="form-group">
        {{ form.author.label_tag }}
        {{ form.author }}
        {{ form.author.errors }}
    </div>
    <div class="form-group">
        {{ form.text.label_tag }}
        {{ form.text }}
        {{ form.text.errors }}
    </div>
    <button type="submit">ส่งความคิดเห็น</button>
</form>
```

```html
<!-- blog/templates/blog/partials/comment_created.html -->
<div id="comment-list" hx-swap-oob="afterbegin">
    {% include 'blog/partials/comment_row.html' with comment=comment %}
</div>
<span id="comment-count" hx-swap-oob="innerHTML">{{ post.comments.count }}</span>
{% include 'blog/partials/comment_form.html' %}
```

**หมายเหตุเรื่อง `hx-on::after-request`**: นี่คือ syntax ของ HTMX ที่ให้
ผูก JavaScript สั้น ๆ เข้ากับ event ของ HTMX โดยตรงในแอตทริบิวต์ (คล้าย
`onclick` มาตรฐานของ HTML) — `event.detail.successful` เป็น `true` เมื่อ
response เป็น 2xx เท่านั้น โค้ดนี้จึงรีเซ็ตฟอร์มกลับเป็นค่าว่างเฉพาะตอนที่
ส่งสำเร็จจริง ๆ (ไม่รีเซ็ตถ้ามี validation error ที่ต้องให้ผู้ใช้เห็นค่าเดิม
ตามที่อธิบายในขั้นตอนที่ 525.3) — เป็นทางเลือกเสริมนอกเหนือจากการที่ View
คืนฟอร์มเปล่าใหม่ผ่าน context `'form': CommentForm()` อยู่แล้วในกรณีสำเร็จ

ผลลัพธ์สุดท้าย: ผู้ใช้พิมพ์คอมเมนต์ กดส่ง → เห็นคอมเมนต์ใหม่ปรากฏบนสุดของ
ลิสต์ทันที พร้อมตัวนับอัปเดต และฟอร์มเคลียร์ตัวเองพร้อมพิมพ์ต่อ — ทั้งหมดนี้
**ไม่มี JavaScript framework ใด ๆ เข้ามาเกี่ยวข้องเลย** นอกจากไฟล์ HTMX
ไฟล์เดียวที่โหลดไว้ตั้งแต่ขั้นตอนที่ 522

### 530.2 ตารางสรุป: จุดที่ HTMX เข้ามาแทนที่โค้ด JavaScript ของ Part 052

| งานที่ต้องทำ | แนวทาง Part 052 (Fetch + JSON) | แนวทาง HTMX (Part นี้) |
|---|---|---|
| โหลดคอมเมนต์เพิ่มโดยไม่ reload | เขียน `fetch()` + แปลง JSON เป็น HTML เอง | `hx-get` + View คืน HTML fragment ตรง ๆ |
| Validate ฟอร์มแบบ inline | เขียน `fetch()` POST + จัดการ error response เอง + อัปเดต DOM เอง | `hx-post` + คืน `ModelForm` ที่มี error กลับไป render ปกติ |
| ลบรายการโดยไม่ reload | เขียน `fetch()` DELETE + หา element ใน DOM มาลบเอง | `hx-delete` + `hx-target="closest ..."` + `hx-swap="outerHTML"` |
| อัปเดตหลายจุดพร้อมกัน (ลิสต์ + ตัวนับ) | จัดการ DOM หลายจุดด้วยมือใน callback เดียว | `hx-swap-oob` หลายจุดในการ swap เดียว |
| ป้องกัน CSRF | แนบ header เองทุกจุดที่เรียก `fetch()` | ตั้งค่า `htmx:configRequest` ครั้งเดียว ใช้ได้ทุก request |

### 530.3 สรุปสิ่งที่ได้เรียนรู้ใน Part นี้

- ✅ เข้าใจแนวคิด "server-driven UI" / "HTML over the wire" และทำไม Django
  เข้ากับ HTMX ได้ดีเป็นพิเศษเมื่อเทียบกับแนวทาง Fetch + JSON (Part 052)
  และ React/Vue (Part 055-056)
- ✅ ติดตั้ง HTMX แบบ self-hosted และตั้งค่าการแนบ CSRF token อัตโนมัติผ่าน
  `htmx:configRequest` ให้ทุก request
- ✅ ใช้ `hx-get`/`hx-post` พื้นฐาน และเข้าใจ attribute หลักทั้งสามตัวอย่าง
  ละเอียด: `hx-trigger` (เมื่อไหร่), `hx-target` (ที่ไหน), `hx-swap`
  (อย่างไร) พร้อม modifier สำคัญของแต่ละตัว
- ✅ ออกแบบ View ที่คืน Partial Template โดยเฉพาะสำหรับ HTMX แยกจาก View
  ที่คืนหน้าเต็ม โดยใช้ `{% include %}` ไฟล์เดียวกันร่วมกันทั้งสองกรณี
  เพื่อไม่ต้องเขียนโครง HTML ซ้ำ
- ✅ ทำฟอร์ม `ModelForm` ให้ validate แบบ inline ไม่ reload หน้า พร้อมเข้าใจ
  ว่าทำไมต้องคืน HTTP 200 เสมอแม้ validation ล้มเหลว และใช้
  `hx-swap-oob` อัปเดตหลายจุดของหน้าพร้อมกันจาก response เดียว
- ✅ ใช้ `hx-boost` เปลี่ยนการนำทางทั่วทั้งเว็บให้ลื่นไหลแบบ SPA โดยไม่ต้อง
  แก้ View แม้แต่บรรทัดเดียว พร้อมควบคุมด้วย `hx-select`/`hx-target`
- ✅ สร้าง Infinite Scroll ด้วย `hx-trigger="revealed"` และแสดง loading
  state ด้วย `hx-indicator`
- ✅ ผสาน Django Messages Framework เข้ากับ HTMX ผ่านเทคนิค Out-of-Band Swap
- ✅ ติดตั้งและใช้ `django-htmx` เพื่อเข้าถึง `request.htmx` แทนการเช็ค
  HTTP header ดิบเอง พร้อมรู้จัก `HttpResponseClientRedirect`
- ✅ แปลงระบบคอมเมนต์ของ blog ทั้งระบบให้เป็น HTMX เต็มรูปแบบ: โพสต์และลบ
  คอมเมนต์ได้โดยไม่ reload หน้าเว็บแม้แต่ครั้งเดียว

### 530.4 Checklist ก่อนไป Part ถัดไป

- [ ] ติดตั้ง HTMX แบบ self-hosted ในโปรเจกต์ และตั้งค่า CSRF header ผ่าน
      `htmx:configRequest` สำเร็จ (ทดสอบด้วย `hx-post` แล้วไม่เจอ 403)
- [ ] เขียนปุ่ม `hx-get`/`hx-post` อย่างน้อย 1 ตัวที่ทำงานได้จริง พร้อมระบุ
      `hx-target`/`hx-swap` เอง (ไม่ใช้ค่าเริ่มต้นล้วน ๆ)
- [ ] อธิบายความแตกต่างระหว่าง `innerHTML`, `outerHTML`, `beforeend` ของ
      `hx-swap` ได้ด้วยคำพูดตัวเอง พร้อมยกตัวอย่างว่าแต่ละแบบเหมาะกับ
      สถานการณ์ไหน
- [ ] เขียน View ที่ตรวจ `HX-Request` header (หรือ `request.htmx`) แล้ว
      เลือก template คืนกลับต่างกันได้สำเร็จ
- [ ] ทำฟอร์มคอมเมนต์แบบ inline validation ได้จริง (ทดสอบด้วยการส่งข้อมูล
      ว่างเปล่า แล้วเห็น error โดยไม่ reload หน้า)
- [ ] เปิดใช้ `hx-boost="true"` ที่ `<body>` แล้วทดสอบคลิกลิงก์ทั่วเว็บ
      สังเกตว่า URL เปลี่ยนแต่หน้าไม่กระพริบขาว
- [ ] สร้าง Infinite Scroll อย่างน้อย 1 หน้า ด้วย `hx-trigger="revealed"`
- [ ] ติดตั้ง `django-htmx` และแก้ View อย่างน้อย 1 ตัวให้ใช้
      `request.htmx` แทนการเช็ค header ดิบ
- [ ] แปลงระบบคอมเมนต์ของ blog เป็น HTMX เต็มรูปแบบตามข้อ 530.1 และทดสอบ
      ผ่านเบราว์เซอร์จริงว่าโพสต์/ลบคอมเมนต์ได้โดยไม่ reload

### 530.5 แบบฝึกหัดท้ายบท

**แบบฝึกหัดที่ 1**: จากตัวอย่างปุ่มถูกใจในขั้นตอนที่ 522.4 ให้ปรับปรุงให้
รองรับการ **ยกเลิกถูกใจ** ได้ด้วย (toggle แทนที่จะเพิ่มค่าเรื่อย ๆ) โดยเพิ่ม
Model `PostLike` ที่เก็บว่า user คนไหนถูกใจโพสต์ไหนบ้าง (`ForeignKey` ไปยัง
`User` และ `Post` พร้อม `unique_together`) แล้วปรับ View และปุ่มให้แสดงผล
ต่างกันระหว่างสถานะ "ยังไม่ถูกใจ" กับ "ถูกใจแล้ว" ด้วย `hx-target="this"`
และ `hx-swap="outerHTML"` เหมือนเดิม

**แบบฝึกหัดที่ 2**: สร้างหน้า "จัดการหมวดหมู่" (`CategoryListView`) ที่แสดง
รายการหมวดหมู่ทั้งหมดพร้อมปุ่ม "แก้ไข" ข้างแต่ละแถว เมื่อกด "แก้ไข" ให้
แถวนั้น**เปลี่ยนเป็นฟอร์มแก้ไข inline** ทันที (ไม่เปิดหน้าใหม่ ไม่มี modal)
โดยใช้ `hx-get` ดึงฟอร์มมาแทนที่แถวเดิมด้วย `hx-swap="outerHTML"` แล้วเมื่อ
submit ฟอร์มสำเร็จ ให้กลับไปแสดงเป็นแถวปกติที่มีข้อมูลใหม่ (pattern นี้
เรียกว่า **"Click to Edit"** เป็นหนึ่งใน pattern ยอดนิยมที่สุดของ HTMX)

**แบบฝึกหัดที่ 3**: เพิ่มฟีเจอร์ **ตอบกลับคอมเมนต์ (nested reply)** เข้าไป
ในระบบคอมเมนต์จากขั้นตอนที่ 530.1 โดยเพิ่ม field `parent` (`ForeignKey`
ไปยัง `Comment` ตัวเอง, `null=True`, `blank=True`) แสดงปุ่ม "ตอบกลับ" ใต้
แต่ละคอมเมนต์ที่กดแล้วเปิดฟอร์มย่อยขึ้นมาด้วย `hx-get`/`hx-target="next
.reply-form-slot"` และเมื่อส่งสำเร็จให้แสดงคอมเมนต์ตอบกลับแบบเยื้องเข้ามา
(indent) ใต้คอมเมนต์แม่

**แบบฝึกหัดที่ 4 (ขั้นสูง)**: เขียน Django test (`django.test.TestCase` +
`Client`) สำหรับ View `comment_create_view` และ `comment_delete_view` จาก
ขั้นตอนที่ 530.1 อย่างน้อย 5 เคส ครอบคลุม: (1) POST ข้อมูลถูกต้อง → ได้
HTTP 200 พร้อมเนื้อหาคอมเมนต์ใหม่ปรากฏใน response, (2) POST ข้อมูลว่างเปล่า
→ ได้ HTTP 200 เช่นกัน (ไม่ใช่ 400) พร้อมข้อความ error ปรากฏใน response,
(3) DELETE โดย user ที่ไม่ใช่ staff → ถูกปฏิเสธ, (4) DELETE โดย staff →
คอมเมนต์ถูกลบจริงจากฐานข้อมูล, (5) ตรวจสอบว่า response มี header
`HX-Request` ถูกจำลองถูกต้องด้วย `self.client.post(url, data, HTTP_HX_REQUEST='true')`
(วิธีมาตรฐานในการจำลอง custom header ผ่าน Django test client)

### 530.6 คำถามที่พบบ่อย (FAQ)

**Q: HTMX แทนที่ JavaScript ได้ทั้งหมดเลยหรือไม่ ไม่ต้องเขียน JavaScript
อีกแล้ว?**
A: ไม่ทั้งหมด — HTMX เก่งเรื่อง "ปฏิสัมพันธ์ระหว่าง client กับ server"
(โหลดข้อมูล, ส่งฟอร์ม, อัปเดตบางส่วนของหน้า) แต่ **ไม่ได้ออกแบบมาสำหรับ
state ฝั่ง client ล้วน ๆ ที่ไม่ต้องคุยกับ server** เช่น เปิด/ปิด dropdown,
สลับ tab, drag-and-drop ที่ยังไม่ต้องบันทึกจนกว่าจะปล่อยเมาส์ — งานเหล่านี้
เหมาะกับไลบรารีเสริมอย่าง **Alpine.js** ที่จะเรียนใน **Part 054** มากกว่า
HTMX กับ Alpine.js มักถูกใช้คู่กันเสมอในโปรเจกต์จริง (HTMX คุยกับ server,
Alpine.js จัดการ UI state เล็ก ๆ ฝั่ง client)

**Q: ทำไมตัวอย่างในบทนี้ถึงคืน HTTP 200 เสมอแม้ตอน validation ล้มเหลว
ทั้งที่ปกติ REST API ควรคืน 400 Bad Request?**
A: เพราะพฤติกรรมเริ่มต้นของ HTMX คือ**ไม่แทรก HTML ที่ได้กลับมาเลยถ้า
status ไม่ใช่ 2xx/3xx** (ข้อ 525.2) ในบริบทของ "ฟอร์ม HTML ที่ render error
กลับมาให้ผู้ใช้เห็น" เราต้องการให้ HTML นั้นถูกแทรกเสมอ จึงคืน 200 เพื่อให้
HTMX ยอมรับ อย่างไรก็ตาม ถ้าต้องการรักษา semantic ของ HTTP status code ไว้
ตามหลัก REST HTMX ก็รองรับการตั้งค่า `htmx.config.responseHandling` เพื่อ
กำหนดเองว่า status code ไหนควร swap หรือไม่ — แต่สำหรับหลักสูตรนี้ การคืน
200 เสมอสำหรับ endpoint ที่ให้บริการ HTMX โดยเฉพาะเป็นวิธีที่ตรงไปตรงมาและ
เข้าใจง่ายที่สุด

**Q: ควรใช้ `hx-boost` กับทุกโปรเจกต์เสมอไปหรือไม่?**
A: ส่วนใหญ่แนะนำให้เปิดไว้เป็นค่าเริ่มต้น เพราะแทบไม่มีข้อเสีย (View ไม่ต้อง
แก้ไขอะไรเลย) แต่มีข้อควรระวัง 2 อย่าง: (1) หน้าที่ต้องใช้ JavaScript ของ
บุคคลที่สามที่รัน `<script>` inline ตอนโหลดหน้า (เช่น payment widget บาง
เจ้า) อาจทำงานผิดปกติเพราะ boost ไม่ re-execute inline script เหมือนการ
โหลดหน้าเต็มปกติเสมอไป ต้องทดสอบเป็นกรณี ๆ ไป และ (2) หน้าที่มี
`<form enctype="multipart/form-data">` สำหรับอัปโหลดไฟล์ขนาดใหญ่ควรพิจารณา
ปิด boost เฉพาะฟอร์มนั้นด้วย `hx-boost="false"` เพื่อให้เห็น progress bar
ของ browser แบบมาตรฐานระหว่างอัปโหลด

**Q: `hx-swap-oob` ปลอดภัยแค่ไหน มีความเสี่ยงอะไรที่ต้องระวัง?**
A: `hx-swap-oob` เป็นแค่กลไกฝั่ง client ในการเลือกว่าจะแทรก HTML ที่ server
render มาให้ (ผ่าน Django Template ที่ auto-escape ค่าอยู่แล้วตามปกติ) ไป
ไว้ตรงไหนของหน้า **ไม่ใช่ช่องโหว่ความปลอดภัยในตัวมันเอง** ตราบใดที่ยังใช้
Django Template render เนื้อหาตามปกติ (ไม่ใช่ต่อสตริง HTML เองแบบไม่ escape)
สิ่งที่ต้องระวังคือ **ตรรกะ**: ต้องแน่ใจว่า `id` ที่ใช้ตรงกับ target
ที่ตั้งใจจริง ๆ เท่านั้น เพราะถ้ามี `id` ซ้ำกันโดยไม่ตั้งใจในหน้าเว็บ (เช่น
คอมเมนต์การ์ดสองใบใช้ `id="comment-list"` ซ้ำกันเพราะลืมใส่ pk) HTMX จะ
สับสนว่าจะแทรกเข้าจุดไหน

---

## เตรียมตัวสำหรับ Part ถัดไป

**Part 054: Django กับ Alpine.js** จะพาคุณไปรู้จักไลบรารี JavaScript ขนาดเล็ก
อีกตัวที่มักถูกใช้คู่กับ HTMX เสมอในโปรเจกต์จริง — Alpine.js เก่งเรื่อง
**state ฝั่ง client ล้วน ๆ** ที่ไม่ต้องคุยกับ server (เปิด/ปิด dropdown,
สลับ tab, แสดง/ซ่อน modal, นับจำนวนตัวอักษรที่พิมพ์แบบ real-time) ซึ่งเป็น
สิ่งที่ HTMX **ไม่ได้ออกแบบมาให้ทำ** ตามที่กล่าวถึงใน FAQ ข้อแรกของ Part นี้
คุณจะได้เรียนรู้ `x-data`, `x-show`, `x-if`, `x-model`, `x-on`, และเทคนิค
การผสาน Alpine.js เข้ากับ HTMX ในหน้าเดียวกันอย่างเป็นระบบ (เช่น ปุ่มที่ใช้
Alpine.js เปิด modal ก่อน แล้วในนั้นมีฟอร์มที่ใช้ HTMX ส่งข้อมูลจริง) เตรียม
ระบบคอมเมนต์ที่แปลงเป็น HTMX เสร็จแล้วจาก Part นี้ไว้ให้พร้อม เพราะเราจะ
เพิ่ม UI enhancement เล็ก ๆ น้อย ๆ ด้วย Alpine.js เข้าไปต่อยอดกันต่อ!
