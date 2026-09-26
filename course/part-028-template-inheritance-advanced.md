# Part 028: Template Inheritance และ Template Tags ขั้นสูง

> **ขั้นตอนที่ 271-280 ของหลักสูตร** | Phase 3: Views, Templates, Forms และ CBV
>
> Part นี้ **ต่อยอดจาก Part 008** โดยตรง Part 008 พาคุณเรียนรู้ Template Inheritance
> พื้นฐาน (`{% extends %}`, `{% block %}`), `{% include %}`, filters ที่ใช้บ่อย,
> `{% csrf_token %}` และ Context Processors เบื้องต้นไปแล้ว — Part นี้**จะไม่สอนซ้ำ**
> เนื้อหาเหล่านั้น แต่จะพาคุณลงลึกไปอีกขั้นสู่สิ่งที่โปรเจกต์ระดับ production จริงต้องใช้:
> การออกแบบ Template Inheritance หลายชั้น (multi-level) อย่างเป็นระบบ, กลไกเบื้องหลังของ
> `{{ block.super }}` และ nested blocks, กลไกการทำงานจริงของ `{% load %}` ที่ Django ใช้
> ค้นหา template tag library (ปูทางสู่ Part 029 ที่จะสอนเขียน custom tag เต็มรูปแบบ),
> built-in tags ที่ยังไม่เคยพูดถึง (`spaceless`, `cycle`, `regroup`, `widthratio`,
> `firstof`, `verbatim`, `templatetag`), การสร้าง Email Template แบบ multipart
> HTML/plain-text, การจัดโครงสร้าง Template Tree ขนาดใหญ่แบบมืออาชีพ, ข้อควรระวังด้าน
> Performance ของ Template, และการรู้จัก Jinja2 เป็นทางเลือก engine เมื่อไหร่ควรพิจารณา
> เมื่อจบ Part นี้ คุณจะสามารถออกแบบโครงสร้าง Template ของโปรเจกต์ขนาดใหญ่ที่มีหลายแอป
> ได้อย่างเป็นระบบ ไม่ใช่แค่เขียน `extends`/`block` แบบสองชั้นตื้น ๆ อีกต่อไป

---

## สารบัญของ Part นี้

- ขั้นตอนที่ 271: โครงสร้าง Template Inheritance หลายชั้น (multi-level) แบบมืออาชีพ: `base.html` → `blog/base_blog.html` → `blog/post_detail.html`
- ขั้นตอนที่ 272: `{{ block.super }}` เจาะลึก, การ override เฉพาะบางส่วนของ block, nested blocks
- ขั้นตอนที่ 273: กลไกการทำงานของ `{% load %}` — Django ค้นหา template tag library จากไหน (เกริ่นก่อนเข้า Part 029)
- ขั้นตอนที่ 274: Built-in tags ที่ยังไม่ได้พูดถึงใน Part 008: `spaceless`, `cycle`, `regroup`, `widthratio`, `firstof`
- ขั้นตอนที่ 275: `{% verbatim %}` (ผสมกับ Vue.js/Alpine.js), `{% now %}` แบบเจาะลึก, `{% templatetag %}`
- ขั้นตอนที่ 276: Template สำหรับอีเมล — Multipart HTML/Plain-text ด้วย `EmailMultiAlternatives`
- ขั้นตอนที่ 277: การจัดโครงสร้าง Template Tree ขนาดใหญ่แบบมืออาชีพ (`templates/base/`, `templates/partials/`, `templates/emails/`)
- ขั้นตอนที่ 278: ข้อควรระวังด้าน Performance ของ Template (เกริ่น Template Caching ที่จะเจาะลึกใน Part 068)
- ขั้นตอนที่ 279: การใช้ Template Engine อื่นร่วมกับ Django (Jinja2 Backend) — เมื่อไหร่ควรพิจารณา
- ขั้นตอนที่ 280: สรุปและแบบฝึกหัด

---

## ขั้นตอนที่ 271: โครงสร้าง Template Inheritance หลายชั้น (Multi-Level) แบบมืออาชีพ

### 271.1 ทบทวนสั้น ๆ จาก Part 008 (ข้อ 75.6) — และสิ่งที่ Part นี้จะเพิ่มเข้าไป

Part 008 แนะนำแนวคิด multi-level inheritance แบบผิวเผินไว้แล้วด้วยตัวอย่าง 3 ชั้น:

```
templates/base.html               (ชั้น 1: ทั้งเว็บไซต์)
    ↓ extends
blog/templates/blog/base_blog.html   (ชั้น 2: โซนบล็อกโดยเฉพาะ)
    ↓ extends
blog/templates/blog/detail.html      (ชั้น 3: หน้าจริง)
```

สิ่งที่ Part 008 **ไม่ได้พูดถึง** คือคำถามที่สำคัญกว่านั้นมากในงานจริง:

- เมื่อไหร่ที่ควร**เพิ่มชั้น**เข้าไป และเมื่อไหร่ที่ควร**หยุดที่ 2 ชั้นพอ**?
- ถ้าโซนบล็อกมีทั้ง "หน้ารายการ", "หน้ารายละเอียด", "หน้าอาร์ไคฟ์ตามเดือน",
  "หน้าตามหมวดหมู่" ที่ทุกหน้าต้องมี sidebar + แถบแท็บ (tab) ร่วมกัน จะออกแบบ block
  อย่างไรให้ยืดหยุ่นแต่ไม่ซับซ้อนเกินไป?
- ทำอย่างไรให้คนอื่นในทีม (หรือตัวคุณเองอีก 6 เดือนข้างหน้า) รู้ว่า `base_blog.html`
  มี block อะไรให้ override บ้าง โดยไม่ต้องเปิดไฟล์อ่านทั้งหมด?
- เมื่อไหร่ที่ multi-level inheritance กลายเป็น "inheritance hell" ที่ควรเปลี่ยนไปใช้
  `{% include %}` แทน?

ขั้นตอนนี้จะตอบคำถามเหล่านี้ด้วยตัวอย่างที่สมจริงกว่าเดิม

### 271.2 กฎการตัดสินใจ: เมื่อไหร่ควรเพิ่มชั้น Inheritance

| สัญญาณ | ควรทำอย่างไร |
|---|---|
| มี ≥ 3 หน้าในแอปเดียวกันที่ต้องแชร์ layout ย่อยเหมือนกัน (เช่น sidebar, แถบแท็บ) ที่ไม่มีในหน้าอื่นของเว็บไซต์ | เพิ่ม base ระดับแอป (`base_<app>.html`) — คุ้มค่าที่จะทำ |
| มีแค่ 1-2 หน้าที่ต้องการ layout พิเศษ | **ไม่คุ้ม** ที่จะเพิ่มชั้น — override block จาก `base.html` ตรง ๆ พอ |
| ส่วนที่แชร์กันเป็น "ชิ้นส่วน" ที่ไม่มีช่องให้เติมเนื้อหาเพิ่ม (เช่น การ์ดสินค้า, แถบแจ้งเตือน) | ใช้ `{% include %}` ไม่ใช่ inheritance ชั้นใหม่ — ทบทวนความแตกต่างจาก Part 008 ข้อ 76.1 |
| Layout ย่อยต้องใช้ในหลายแอปพร้อมกัน (เช่น ทั้งแอป `blog` และแอป `shop` ต้องการ sidebar คล้ายกัน) | พิจารณาทำเป็น `templates/base/base_with_sidebar.html` ระดับโปรเจกต์แทนที่จะผูกกับแอปเดียว |
| จำนวนชั้นเกิน 3-4 ชั้น | เตือนตัวเอง — มีความเสี่ยงสูงที่จะเป็น "inheritance hell" ดูข้อ 271.6 |

### 271.3 ตัวอย่างจริง: โซนบล็อกที่มี Sidebar และแถบแท็บร่วมกัน 4 หน้า

สมมติแอป `blog` เติบโตขึ้นจาก Part 008 จนตอนนี้มี 4 หน้า: รายการบทความ, รายละเอียด
บทความ, อาร์ไคฟ์ตามเดือน, และหน้าตามหมวดหมู่ — ทั้ง 4 หน้าต้องมี sidebar เดียวกัน
(หมวดหมู่ยอดนิยม + บทความล่าสุด) และแถบแท็บด้านบนเหมือนกัน (บทความทั้งหมด / อาร์ไคฟ์)
นี่คือสัญญาณตามตารางข้อ 271.2 ว่าคุ้มค่าที่จะสร้าง `base_blog.html`

```html
<!-- blog/templates/blog/base_blog.html -->
{% extends 'base.html' %}
{% load static %}

{#
  สัญญาของ block ในไฟล์นี้ (Block Contract) — เขียนเป็นคอมเมนต์ไว้บนสุดเสมอ
  เพื่อให้คนที่มา extends ไฟล์นี้รู้ทันทีว่ามี block อะไรให้ override บ้าง:

  - blog_title      : ชื่อหัวข้อที่แสดงเหนือแถบแท็บ (ค่าเริ่มต้น: "บล็อก")
  - blog_tabs_extra  : ปุ่ม/ลิงก์เพิ่มเติมในแถบแท็บ (ค่าเริ่มต้น: ว่าง)
  - blog_content     : เนื้อหาหลักของโซนบล็อก (บังคับ override เสมอ)
  - blog_sidebar_extra : เนื้อหาเสริมท้าย sidebar เฉพาะบางหน้า (ค่าเริ่มต้น: ว่าง)
#}

{% block extra_head %}
    {{ block.super }}
    <link rel="stylesheet" href="{% static 'blog/css/blog-zone.css' %}">
{% endblock %}

{% block content %}
<div class="blog-layout">
    <div class="blog-layout__main">
        <header class="blog-layout__header">
            <h1>{% block blog_title %}บล็อก{% endblock %}</h1>
            <nav class="blog-layout__tabs">
                <a href="{% url 'blog:list' %}"
                   class="tab{% if request.resolver_match.url_name == 'list' %} tab--active{% endif %}">
                    บทความทั้งหมด
                </a>
                <a href="{% url 'blog:archive' %}"
                   class="tab{% if request.resolver_match.url_name == 'archive' %} tab--active{% endif %}">
                    อาร์ไคฟ์
                </a>
                {% block blog_tabs_extra %}{% endblock %}
            </nav>
        </header>

        <div class="blog-layout__body">
            {% block blog_content %}{% endblock %}
        </div>
    </div>

    <aside class="blog-layout__sidebar">
        <section class="sidebar-widget">
            <h3>หมวดหมู่ยอดนิยม</h3>
            <ul>
                {% for category in popular_categories %}
                    <li><a href="{% url 'blog:by_category' category.slug %}">{{ category.name }}</a></li>
                {% endfor %}
            </ul>
        </section>
        <section class="sidebar-widget">
            <h3>บทความล่าสุด</h3>
            <ul>
                {% for recent in recent_posts %}
                    <li><a href="{% url 'blog:detail' recent.slug %}">{{ recent.title }}</a></li>
                {% endfor %}
            </ul>
        </section>
        {% block blog_sidebar_extra %}{% endblock %}
    </aside>
</div>
{% endblock %}
```

> **หมายเหตุเรื่อง `popular_categories` และ `recent_posts`**: ตัวแปรทั้งสองนี้ควรมา
> จาก **context processor เฉพาะแอป** (ไม่ใช่ตัวแปร global เหมือนข้อ 79.4 ของ Part 008
> เพราะมันเกี่ยวข้องเฉพาะโซนบล็อกเท่านั้น) หรือจะส่งผ่าน mixin ของ Class-Based View
> ที่ทุก view ในโซนบล็อก inherit ร่วมกัน (ทบทวน Mixin จาก Part 024) ก็ได้เช่นกัน — ทั้ง
> สองวิธีนี้ช่วยไม่ให้ทุก view ต้อง query ข้อมูล sidebar ซ้ำ ๆ ด้วยตัวเอง

### 271.4 หน้าจริงแต่ละหน้า Extends จาก `base_blog.html`

```html
<!-- blog/templates/blog/post_detail.html -->
{% extends 'blog/base_blog.html' %}

{% block title %}{{ post.title }} | Django Mastery Blog{% endblock %}

{% block blog_title %}{{ post.title }}{% endblock %}

{% block blog_content %}
<article class="post-detail">
    <p class="post-detail__meta">
        เผยแพร่ {{ post.created_at|date:"d F Y" }}
        {% if post.updated_at != post.created_at %}
            (แก้ไขล่าสุด {{ post.updated_at|date:"d F Y" }})
        {% endif %}
    </p>
    {{ post.content|linebreaks }}
</article>
{% endblock %}

{% block blog_sidebar_extra %}
<section class="sidebar-widget">
    <h3>แท็กของบทความนี้</h3>
    <ul class="tag-list">
        {% for tag in post.tags.all %}
            <li><a href="{% url 'blog:by_tag' tag.slug %}">#{{ tag.name }}</a></li>
        {% endfor %}
    </ul>
</section>
{% endblock %}
```

```html
<!-- blog/templates/blog/archive.html -->
{% extends 'blog/base_blog.html' %}

{% block title %}อาร์ไคฟ์บทความ | Django Mastery Blog{% endblock %}
{% block blog_title %}อาร์ไคฟ์บทความ{% endblock %}

{% block blog_tabs_extra %}
    <span class="tab tab--static">{{ current_year }}</span>
{% endblock %}

{% block blog_content %}
<ul class="archive-list">
    {% for post in posts %}
        <li>{{ post.created_at|date:"d M" }} — <a href="{% url 'blog:detail' post.slug %}">{{ post.title }}</a></li>
    {% endfor %}
</ul>
{% endblock %}
```

สังเกตว่า `post_detail.html` override เพียง 3 block (`title`, `blog_title`,
`blog_content`, `blog_sidebar_extra`) ในขณะที่ `archive.html` override คนละชุด
(`title`, `blog_title`, `blog_tabs_extra`, `blog_content`) — นี่คือประโยชน์หลักของ
การออกแบบ block ให้ **ละเอียดพอ** (granular) ตั้งแต่ชั้นกลาง: แต่ละหน้าปลายทาง
override เฉพาะสิ่งที่ตัวเองต้องการจริง ๆ โดยไม่ต้องเขียนโครง HTML ของ sidebar/tabs ซ้ำเลย

### 271.5 ขยายไปยังหลายโซนพร้อมกัน: เมื่อโปรเจกต์มีมากกว่า 1 แอปที่ต้องการ Layout เฉพาะ

เมื่อโปรเจกต์เติบโต (คุณจะเห็นภาพนี้ชัดขึ้นเมื่อถึง Phase 4 ที่เพิ่มระบบ Auth และบัญชี
ผู้ใช้) โครงสร้าง multi-level มักขยายเป็นแบบนี้:

```
templates/
└── base.html                          ← ชั้น 1: ทั้งเว็บไซต์

blog/templates/blog/
└── base_blog.html                     ← ชั้น 2: โซนบล็อก (extends base.html)
    ├── list.html
    ├── post_detail.html
    ├── archive.html
    └── by_category.html

accounts/templates/accounts/
└── base_account.html                  ← ชั้น 2: โซนบัญชีผู้ใช้ (extends base.html)
    ├── profile.html
    ├── settings.html
    └── change_password.html
```

**ข้อสังเกตสำคัญ**: `base_blog.html` และ `base_account.html` **ต่างก็ extends จาก
`base.html` โดยตรง** (ไม่ใช่ extends ต่อกันเอง) เพราะทั้งสองโซนไม่มีความสัมพันธ์กัน
— นี่คือรูปแบบ **"แผนภูมิต้นไม้กว้าง ตื้น" (wide-and-shallow tree)** ซึ่งเป็นรูปแบบที่
โปรเจกต์ระดับมืออาชีพส่วนใหญ่ใช้ ต่างจาก **"โซ่ลึก" (deep chain)** ที่แต่ละชั้น extends
ต่อกันไปเรื่อย ๆ (`base.html` → `base_blog.html` → `base_blog_admin.html` →
`base_blog_admin_reports.html` → หน้าจริง) ซึ่งควรหลีกเลี่ยงถ้าเป็นไปได้

### 271.6 ข้อควรระวัง: เมื่อ Multi-Level Inheritance กลายเป็น "Inheritance Hell"

| สัญญาณเตือน | ปัญหาที่ตามมา | ทางแก้ |
|---|---|---|
| ชั้น inheritance ลึกเกิน 3-4 ชั้น | ต้องเปิดไฟล์หลายไฟล์พร้อมกันเพื่อเข้าใจว่า block หนึ่ง ๆ จะ render อะไรออกมาจริง | รวมชั้นกลางที่ไม่มีประโยชน์เข้าด้วยกัน หรือเปลี่ยนไปใช้ `{% include %}` สำหรับส่วนที่ไม่มีช่องให้เติม |
| ชั้นกลางมี block มากกว่า 10 block | ยากต่อการจำว่า block ไหนบังคับต้อง override, block ไหนมีค่าเริ่มต้นที่ใช้ได้เลย | เขียน Block Contract เป็นคอมเมนต์ (ดูข้อ 271.3) เสมอ และพิจารณาแยกเป็นหลายไฟล์ที่ใช้ `include` ประกอบกัน |
| แต่ละหน้าปลายทาง override block เกิน 80% ของทั้งหมด | ชั้นกลางแทบไม่ได้ให้ประโยชน์อะไรเลย เพราะทุกหน้าต้องเขียนใหม่เกือบหมดอยู่ดี | พิจารณาลบชั้นกลางทิ้ง แล้วให้ทุกหน้า extends จาก `base.html` ตรง ๆ |
| ต้องใช้ `{{ block.super }}` มากกว่า 2 ชั้นซ้อนกันเป็นประจำ | โค้ดอ่านยาก ต้องไล่ตามหลายไฟล์เพื่อรู้ว่าเนื้อหาสุดท้ายประกอบมาจากที่ไหนบ้าง | พิจารณาปรับโครงสร้าง block ใหม่ให้ตรงจุดกว่านี้ (ดูข้อ 272 ต่อไป) |

หลักการทองคำ: **Multi-level inheritance มีไว้แก้ปัญหาการซ้ำซ้อนของ "โครงหน้า"
(layout) เท่านั้น** ไม่ใช่เครื่องมือสารพัดประโยชน์ ถ้ารู้สึกว่ากำลังฝืนออกแบบ block
ให้ซับซ้อนขึ้นเรื่อย ๆ เพื่อให้ครอบคลุมทุกกรณี มักเป็นสัญญาณว่าถึงเวลาต้องหยุดคิดใหม่

---

## ขั้นตอนที่ 272: `{{ block.super }}` เจาะลึก, Override บางส่วน, Nested Blocks

### 272.1 ทบทวนกลไกพื้นฐานจาก Part 008 ข้อ 75.5

`{{ block.super }}` ดึงเนื้อหาต้นฉบับของ block เดียวกันจาก parent template กลับมา
ใช้ได้เฉพาะ**ภายใน block ที่ถูก override เท่านั้น** — Part นี้จะอธิบายกลไกที่ลึกกว่านั้น:
มันทำงานอย่างไรเมื่อมีมากกว่า 2 ชั้น และมันมีข้อจำกัดอะไรบ้างที่มือใหม่มักพลาด

### 272.2 `block.super` ไล่ขึ้นไปกี่ชั้นก็ได้ ไม่ใช่แค่ชั้นติดกัน

ประเด็นที่มือใหม่มักเข้าใจผิด: `{{ block.super }}` **ไม่ได้ดึงจาก parent โดยตรง
เท่านั้น** แต่ดึงจาก **เนื้อหาของ block นั้นตามที่ resolve ได้จนถึงจุดปัจจุบันในสาย
inheritance** ถ้าชั้นกลางก็ใช้ `block.super` ของตัวเองไว้แล้ว การไล่ขึ้นจะต่อกันเป็น
ทอด ๆ ครบทุกชั้น:

```html
<!-- templates/base.html -->
{% block extra_head %}
<meta name="theme-color" content="#0b5fff">
{% endblock %}
```

```html
<!-- blog/templates/blog/base_blog.html -->
{% extends 'base.html' %}

{% block extra_head %}
    {{ block.super }}
    <link rel="stylesheet" href="{% static 'blog/css/blog-zone.css' %}">
{% endblock %}
```

```html
<!-- blog/templates/blog/post_detail.html -->
{% extends 'blog/base_blog.html' %}

{% block extra_head %}
    {{ block.super }}
    <meta property="og:title" content="{{ post.title }}">
    <meta property="og:type" content="article">
{% endblock %}
```

ผลลัพธ์สุดท้ายที่ render ออกมาในหน้า `post_detail.html` คือเนื้อหาจาก **ทั้ง 3 ชั้น
เรียงต่อกัน**:

```html
<meta name="theme-color" content="#0b5fff">
<link rel="stylesheet" href="/static/blog/css/blog-zone.css">
<meta property="og:title" content="Django คืออะไร">
<meta property="og:type" content="article">
```

นี่คือรูปแบบที่ใช้บ่อยที่สุดของ `block.super` ในโปรเจกต์จริง: **แต่ละชั้นเพิ่มของ
ตัวเองต่อท้ายของชั้นก่อนหน้า** ทำให้ `extra_head` สะสม CSS/meta tag จากทุกชั้นในสาย
inheritance โดยไม่มีชั้นไหนต้องรู้จักเนื้อหาของอีกชั้นเลย

### 272.3 Nested Blocks: Block ที่อยู่ข้างในอีก Block หนึ่ง

DTL อนุญาตให้ประกาศ `{% block %}` ซ้อนกันได้ (nested blocks) — เนื้อหาเริ่มต้นของ
block นอกสามารถมี block ในซ้อนอยู่ข้างในได้:

```html
<!-- blog/templates/blog/base_blog.html (ส่วนที่เกี่ยวข้อง) -->
{% block blog_content %}
    <div class="blog-content-wrapper">
        {% block blog_content_header %}
            <h2>เนื้อหา</h2>
        {% endblock %}
        {% block blog_content_body %}
            <p>ยังไม่มีเนื้อหา</p>
        {% endblock %}
    </div>
{% endblock %}
```

**กฎสำคัญของ Nested Blocks**: ถ้าเทมเพลตลูก override **เฉพาะ block นอก**
(`blog_content`) ทั้งก้อนโดยไม่เขียน `{% block blog_content_header %}` หรือ
`{% block blog_content_body %}` ไว้ข้างในเลย **ค่าเริ่มต้นของ block ในทั้งสองจะ
หายไปทันที** เพราะเทมเพลตลูกได้แทนที่เนื้อหาทั้งหมดของ block นอกไปแล้ว รวมถึง block
ในที่ซ้อนอยู่ด้วย:

```html
<!-- ❌ ตัวอย่างที่ผิดพลาดบ่อย -->
{% extends 'blog/base_blog.html' %}

{% block blog_content %}
    <article>{{ post.content|linebreaks }}</article>
    {# block blog_content_header และ blog_content_body หายไปทั้งคู่ #}
    {# หน้านี้จะไม่มีโอกาสให้เทมเพลตลูกของลูก (ถ้ามี) override สอง block นี้ได้อีก #}
{% endblock %}
```

ถ้าต้องการ**คงความสามารถในการ override block ในไว้** สำหรับเทมเพลตที่จะ extends
ต่อไปอีกชั้น ต้องเขียน block ในไว้ข้างในด้วยเสมอ (แม้จะใส่เนื้อหาใหม่ทั้งหมดก็ตาม):

```html
<!-- ✅ ถูกต้อง: คง nested block ไว้เพื่อให้ override ต่อได้อีกชั้น -->
{% extends 'blog/base_blog.html' %}

{% block blog_content %}
    <div class="blog-content-wrapper">
        {% block blog_content_header %}
            <h2>{{ post.title }}</h2>
        {% endblock %}
        {% block blog_content_body %}
            <article>{{ post.content|linebreaks }}</article>
        {% endblock %}
    </div>
{% endblock %}
```

### 272.4 Override เฉพาะ Block ในโดยไม่แตะ Block นอกเลย

ในทางกลับกัน ถ้าเทมเพลตลูกต้องการ override **เฉพาะ block ในเท่านั้น** โดยใช้โครง
ของ block นอกตามเดิมทั้งหมด สามารถทำได้โดยไม่ต้องเขียน block นอกซ้ำเลย — DTL ค้นหา
block ตามชื่อทั่วทั้งสายการสืบทอด ไม่จำเป็นต้อง "ห่อ" ด้วยชื่อ block พ่อแม่ก่อนเสมอไป:

```html
<!-- blog/templates/blog/post_detail.html -->
{% extends 'blog/base_blog.html' %}

{# override เฉพาะ block ในสุด — โครง .blog-content-wrapper และ header
   ยังคงมาจาก base_blog.html ตามเดิมทั้งหมด #}
{% block blog_content_body %}
    <article class="post-detail">
        {{ post.content|linebreaks }}
    </article>
{% endblock %}
```

นี่คือประโยชน์หลักของการออกแบบ nested blocks: ทำให้เทมเพลตลูกเลือก override ได้
**ละเอียดถึงระดับที่ต้องการจริง ๆ** โดยไม่ต้องคัดลอกโครง HTML ส่วนที่ไม่เปลี่ยนแปลงมา
เขียนซ้ำ

### 272.5 ตารางสรุป: รูปแบบการ Override ที่เป็นไปได้ทั้งหมด

| ต้องการทำอะไร | เขียนอย่างไร |
|---|---|
| แทนที่เนื้อหาเดิมทั้งหมด | `{% block name %}เนื้อหาใหม่{% endblock %}` (ไม่ใช้ `block.super`) |
| เพิ่มเนื้อหาต่อท้ายของเดิม | `{% block name %}{{ block.super }}เนื้อหาใหม่{% endblock %}` |
| เพิ่มเนื้อหาไว้ก่อนของเดิม | `{% block name %}เนื้อหาใหม่{{ block.super }}{% endblock %}` |
| override เฉพาะส่วนย่อยข้างใน (nested) | ประกาศ block ย่อยไว้ล่วงหน้าในชั้นกลาง แล้ว override เฉพาะชื่อ block ย่อยนั้นในชั้นลูก |
| คงความสามารถ override ต่อได้อีกชั้น | ต้องเขียน `{% block %}` ของ nested block ซ้ำไว้เสมอ แม้จะใส่เนื้อหาใหม่ทั้งหมด |
| ยกเลิกเนื้อหา block ไปเลย (ว่างเปล่า) | `{% block name %}{% endblock %}` (เว้นว่างไว้ตรง ๆ) |

### 272.6 ข้อจำกัด: `block.super` ใช้ไม่ได้นอก Block ที่ Override

```html
{# ❌ Error: block.super ใช้ไม่ได้ตรงนี้ — ไม่ได้อยู่ใน context ของ block ใด #}
<p>{{ block.super }}</p>
```

`block.super` เป็นตัวแปรพิเศษที่ Django เตรียมไว้ให้เฉพาะภายในขอบเขต (scope) ของ
`{% block %}...{% endblock %}` ที่กำลังถูก override เท่านั้น ถ้าเขียนนอก block จะได้
ค่าว่างเปล่าตามพฤติกรรมมาตรฐานของตัวแปรที่ไม่มีอยู่จริง (ทบทวน Part 008 ข้อ 72.6) —
ไม่ raise error แต่จะไม่แสดงอะไรเลย

---

## ขั้นตอนที่ 273: กลไกการทำงานของ `{% load %}` เบื้องหลัง

> **หมายเหตุ**: ขั้นตอนนี้อธิบาย**กลไกการค้นหาและโหลด** template tag library เท่านั้น
> ยังไม่สอนวิธี**เขียน**custom tag/filter ของตัวเอง (เรื่องนั้นเจาะลึกเต็มรูปแบบใน
> **Part 029: Custom Template Tags และ Filters**) เป้าหมายตรงนี้คือให้คุณเข้าใจว่า
> เมื่อพิมพ์ `{% load static %}` แล้ว Django ไปหาไฟล์ไหน มาจากไหน ก่อนที่จะไปเขียน
> library ของตัวเองใน Part ถัดไป

### 273.1 Template Tag Library คืออะไร

**Template Tag Library** คือไฟล์ Python ที่รวบรวม custom tag และ filter ไว้เป็นกลุ่ม
เพื่อไม่ให้ Django ต้องโหลด tag ทุกตัวจากทุกแอปเข้าไปในทุกเทมเพลตโดยไม่จำเป็น (ทั้งเรื่อง
performance และการป้องกันชื่อ tag ชนกันโดยไม่ตั้งใจ) — `{% load static %}` ที่คุณใช้
มาตลอดใน Part 008 คือการโหลด library ชื่อ `static` ซึ่งเป็นของ
`django.contrib.staticfiles` นั่นเอง

### 273.2 Django ค้นหา Library จากไหนเมื่อเจอ `{% load libname %}`

เมื่อ Django parse เจอ `{% load libname %}` มันจะทำตามลำดับนี้:

```
{% load libname %}
        │
        ▼
1. Django ไล่ดูทุกแอปใน INSTALLED_APPS ตามลำดับที่ประกาศ
        │
        ▼
2. สำหรับแต่ละแอป มองหาโฟลเดอร์ชื่อ templatetags/ ข้างในแอปนั้น
        │
        ▼
3. มองหาไฟล์ .py ชื่อ "libname.py" ข้างในโฟลเดอร์ templatetags/ นั้น
        │
        ▼
4. เจอไฟล์แรกที่ตรงชื่อ → โหลด object ชื่อ "register" (instance ของ
   template.Library) จากไฟล์นั้นเข้ามาใช้งาน
        │
        ▼
5. ทุก tag/filter ที่ลงทะเบียนไว้ใน register นั้น ใช้งานได้ทันทีในเทมเพลตนี้
```

นี่คือเหตุผลที่ `{% load static %}` หา `static` เจอ: เพราะแพ็กเกจ
`django.contrib.staticfiles` (ซึ่งอยู่ใน `INSTALLED_APPS` มาตั้งแต่ `startproject`)
มีไฟล์ `django/contrib/staticfiles/templatetags/static.py` อยู่ข้างในจริง ๆ

### 273.3 โครงสร้างโฟลเดอร์ `templatetags/` ที่จำเป็น

ทุกแอปที่ต้องการมี custom tag library ของตัวเอง ต้องมีโครงสร้างไฟล์แบบนี้:

```
blog/
├── models.py
├── views.py
└── templatetags/
    ├── __init__.py          ← บังคับต้องมี ทำให้ Python มองเป็น package
    └── blog_extras.py       ← ชื่อไฟล์นี้ = ชื่อที่ใช้ตอน {% load blog_extras %}
```

```python
# blog/templatetags/blog_extras.py (โครงขั้นต่ำสุด — รายละเอียดเต็มอยู่ใน Part 029)
from django import template

register = template.Library()   # ← Django มองหา object ชื่อ "register" นี้เสมอ
```

> **ทำไมต้องมี `__init__.py`**: โฟลเดอร์ `templatetags/` ต้องเป็น **Python package**
> ที่ import ได้ (เหมือน `migrations/` ที่คุณสร้างมาตั้งแต่ Part 011) ถ้าลืมไฟล์นี้
> Django จะหา library ไม่เจอเลย แม้ไฟล์ `.py` จะอยู่ถูกที่ก็ตาม — เป็นข้อผิดพลาดที่พบ
> บ่อยที่สุดเมื่อสร้าง custom tag library ครั้งแรก

### 273.4 ลำดับการค้นหาเมื่อมีชื่อ Library ซ้ำกันหลายแอป

ถ้าสองแอปต่างมีไฟล์ `templatetags/format_helpers.py` เหมือนกัน Django จะใช้ไฟล์ของ
แอปที่มาก่อนใน `INSTALLED_APPS` เสมอ (กฎทองข้อเดิมจาก Part 006 และ Part 008 ข้อ
71.6: **ลำดับสำคัญเสมอ**) — ต่างจากการค้นหา template ตรงที่**ไม่มีแนวคิด `DIRS`
ระดับโปรเจกต์**สำหรับ template tag library เลย มันผูกกับ Python import path ของ
แอปใน `INSTALLED_APPS` เท่านั้น ไม่เกี่ยวกับโฟลเดอร์ `templates/` หรือค่า `DIRS` ใด ๆ

| ประเด็น | การค้นหา Template (Part 008 ข้อ 71.6) | การค้นหา Tag Library |
|---|---|---|
| แหล่งค้นหา | `TEMPLATES.DIRS` + โฟลเดอร์ `templates/` ของแต่ละแอป | เฉพาะโฟลเดอร์ `templatetags/` ของแต่ละแอปใน `INSTALLED_APPS` |
| มีระดับโปรเจกต์ไหม | มี (ผ่าน `DIRS`) | **ไม่มี** — ต้องอยู่ในแอปเสมอ |
| ตัวตัดสินเมื่อชื่อซ้ำ | ลำดับใน `DIRS` มาก่อน แล้วตามด้วยลำดับ `INSTALLED_APPS` | ลำดับใน `INSTALLED_APPS` เท่านั้น |
| namespace ป้องกันชื่อชน | ซ้อนชื่อแอปในโฟลเดอร์ (`blog/templates/blog/...`) | **ไม่มีการ namespace อัตโนมัติ** — ต้องตั้งชื่อไฟล์ให้ไม่ซ้ำกันเองข้ามแอป |

> **ข้อควรระวังเชิงปฏิบัติ**: เพราะไม่มีการ namespace อัตโนมัติเหมือน template ควร
> ตั้งชื่อไฟล์ library ให้มีคำนำหน้าเฉพาะแอปเสมอ เช่น `blog_extras.py`,
> `shop_extras.py` แทนที่จะตั้งชื่อกลาง ๆ อย่าง `extras.py` หรือ `helpers.py` ที่เสี่ยง
> ชนกับแอปอื่นในอนาคต — ธรรมเนียมนี้เราจะใช้ตลอดเมื่อเขียน custom tag จริงใน Part 029

### 273.5 `{% load %}` โหลดได้หลาย Library พร้อมกัน และโหลดเฉพาะบาง Tag ได้

```html
{% load static blog_extras humanize %}
```

โหลดได้หลาย library ในบรรทัดเดียวโดยคั่นด้วยช่องว่าง หรือถ้าต้องการโหลดเฉพาะบาง
tag/filter จาก library หนึ่ง (ไม่เอาทั้งหมด) ใช้ไวยากรณ์ `from`:

```html
{% load reading_time from blog_extras %}
```

### 273.6 Built-in Tags/Filters ที่ไม่ต้อง `{% load %}` เลย เทียบกับที่ต้องโหลด

| กลุ่ม | ตัวอย่าง | ต้อง `{% load %}` ไหม |
|---|---|---|
| **Built-in tags หลัก** (อยู่ใน `django.template.defaulttags`) | `if`, `for`, `block`, `extends`, `include`, `with`, `now`, `comment`, `spaceless`, `cycle`, `regroup`, `firstof`, `widthratio`, `verbatim`, `templatetag`, `autoescape`, `filter`, `ifchanged`, `lorem`, `debug`, `csrf_token`, `url` | **ไม่ต้อง** — โหลดอัตโนมัติเข้าทุกเทมเพลตเสมอ |
| **Built-in filters หลัก** (อยู่ใน `django.template.defaultfilters`) | `date`, `default`, `length`, `truncatewords`, `safe`, `linebreaks`, `join`, `yesno` ฯลฯ | **ไม่ต้อง** — เช่นเดียวกับ built-in tags |
| **Static files** | `static`, `get_static_prefix` | ต้อง `{% load static %}` |
| **Internationalization** | `trans`, `blocktrans`, `language` | ต้อง `{% load i18n %}` (จะเรียนใน Phase 10) |
| **Timezone** | `timezone`, `localtime` | ต้อง `{% load tz %}` |
| **django.contrib.humanize** | `naturaltime`, `intcomma`, `apnumber` | ต้อง `{% load humanize %}` และเพิ่ม `'django.contrib.humanize'` ใน `INSTALLED_APPS` |
| **Custom tag library ของแอปคุณเอง** | เช่น `reading_time` ในตัวอย่างข้อ 273.3 | ต้อง `{% load <ชื่อไฟล์> %}` เสมอ |

นี่คือคำตอบที่แท้จริงว่าทำไม `{% if %}`, `{% for %}`, `{% extends %}`,
`{% csrf_token %}` ที่ใช้มาตลอด Part 008 ถึงไม่เคยต้อง `{% load %}` เลย — เพราะมันคือ
built-in tag ที่อยู่ใน "default library" ที่ Django ผูกเข้ากับทุก `Template` object
โดยอัตโนมัติตั้งแต่ต้น ต่างจาก `static`, `i18n`, `humanize` หรือ library ของแอปคุณเอง
ที่เป็น library แยกต่างหากซึ่งต้องขอโหลดก่อนเสมอ

---

## ขั้นตอนที่ 274: Built-in Tags ที่ยังไม่ได้พูดถึงใน Part 008

Part 008 ครอบคลุม `if`, `for`, `with`, `include`, `extends`, `block`, `csrf_token`,
`static`, `now`, `comment`, `autoescape` ไปแล้ว ขั้นตอนนี้จะพาไปรู้จัก built-in tag
ที่เหลือซึ่งมีประโยชน์มากในสถานการณ์เฉพาะทาง

### 274.1 `{% spaceless %}`: ลบช่องว่างระหว่าง HTML Tag

```html
{% spaceless %}
    <ul>
        <li>บทความ 1</li>
        <li>บทความ 2</li>
    </ul>
{% endspaceless %}
```

`{% spaceless %}` ลบช่องว่าง (whitespace: space, tab, newline) ที่อยู่**ระหว่าง HTML
tag สองตัว** เท่านั้น (คือระหว่าง `>` กับ `<` ที่ตามมา) ไม่แตะข้อความที่อยู่ข้างในแท็ก
ผลลัพธ์ที่ได้จากตัวอย่างข้างบนคือ:

```html
<ul><li>บทความ 1</li><li>บทความ 2</li></ul>
```

**ประโยชน์**: ลดขนาดไฟล์ HTML เล็กน้อย และแก้ปัญหา CSS บางกรณีที่ `display: inline`
หรือ `inline-block` ทำให้เกิดช่องว่างที่มองเห็นได้ระหว่างอิลิเมนต์ (เช่นปุ่มติดกันแต่มี
ช่องว่างแปลก ๆ คั่นเพราะ whitespace ในซอร์ส HTML) ในทางปฏิบัติปัจจุบันมักแก้ปัญหานี้
ด้วย CSS `gap` หรือ Flexbox/Grid แทน แต่ `spaceless` ยังพบได้ในโค้ดเก่าและมีประโยชน์
เมื่อทำงานกับ email template (ขั้นตอนที่ 276) ที่ mail client บางตัวไวต่อ whitespace

### 274.2 `{% cycle %}`: สลับค่าไปเรื่อย ๆ ตามรอบการวนซ้ำ

`{% cycle %}` ใช้สลับค่าระหว่างตัวเลือกที่กำหนด ทุกครั้งที่ tag นี้ถูกเรียกซ้ำ
(โดยทั่วไปคือภายใน `{% for %}`) เหมาะมากสำหรับลายทางสลับสี (zebra striping) ของตาราง:

```html
<table>
{% for post in posts %}
    <tr class="{% cycle 'row-light' 'row-dark' %}">
        <td>{{ post.title }}</td>
        <td>{{ post.created_at|date:"d/m/Y" }}</td>
    </tr>
{% endfor %}
</table>
```

ผลลัพธ์: แถวที่ 1, 3, 5 ได้ class `row-light` ส่วนแถวที่ 2, 4, 6 ได้ `row-dark`
สลับกันไปเรื่อย ๆ ตามจำนวนรอบของ `{% for %}`

**ตั้งชื่อเก็บค่าไว้ใช้ซ้ำที่อื่นด้วย `as`**:

```html
{% for post in posts %}
    <div class="{% cycle 'bg-white' 'bg-gray' as rowcolor %}">
        <span class="badge {{ rowcolor }}">{{ post.title }}</span>
    </div>
{% endfor %}
```

เมื่อใช้ `as ชื่อ` ครั้งแรก ค่านั้นจะถูกเก็บไว้เป็นตัวแปร (`rowcolor` ในตัวอย่างนี้)
เรียกซ้ำที่ไหนก็ได้ในเทมเพลตต่อจากนั้นโดยไม่ต้องเขียน `{% cycle %}` ซ้ำ (การเรียกซ้ำ
แบบนี้จะ**ไม่**ขยับไปยังค่าถัดไป เพียงแค่คืนค่าปัจจุบันที่ตั้งไว้ล่าสุดเท่านั้น)

### 274.3 `{% regroup %}`: จัดกลุ่ม List แบบแบนให้เป็นกลุ่มตาม Attribute

`{% regroup %}` แก้ปัญหาที่ DTL เจตนาไม่มีความสามารถประมวลผลข้อมูลซับซ้อน (ทบทวน
Part 008 ข้อ 71.7: "template ไม่ควรมี business logic") — ในกรณีที่ต้องการ**จัดกลุ่ม
ข้อมูลง่าย ๆ เพื่อการแสดงผลล้วน ๆ** (ไม่ใช่ business logic จริงจัง) `regroup` ช่วยได้
โดยไม่ต้องเขียน logic ซับซ้อนใน view

ตัวอย่าง: มี queryset ของบทความที่ **เรียงตามปีเผยแพร่ไว้แล้ว** (สำคัญมาก — `regroup`
ต้องการข้อมูลที่เรียงลำดับตาม field ที่จะ group มาก่อน ไม่เช่นนั้นผลลัพธ์จะผิด) และ
ต้องการแสดงเป็นหัวข้อปีคั่นระหว่างกลุ่ม:

```python
# blog/views.py
def post_archive(request):
    posts = Post.objects.filter(is_published=True).order_by('-created_at')
    return render(request, 'blog/archive.html', {'posts': posts})
```

```html
<!-- blog/templates/blog/archive.html -->
{% regroup posts by created_at.year as posts_by_year %}

{% for year_group in posts_by_year %}
    <h2>{{ year_group.grouper }}</h2>
    <ul>
        {% for post in year_group.list %}
            <li>{{ post.created_at|date:"d M" }} — {{ post.title }}</li>
        {% endfor %}
    </ul>
{% endfor %}
```

ไวยากรณ์ `{% regroup queryset by field as varname %}` สร้างตัวแปรใหม่ (`posts_by_year`)
ที่เป็น list ของ object ที่มี 2 attribute เสมอ:

| Attribute | ความหมาย |
|---|---|
| `grouper` | ค่าที่ใช้เป็นตัวจัดกลุ่ม (ในตัวอย่างนี้คือปี เช่น `2026`, `2025`) |
| `list` | list ของ item ทั้งหมดในกลุ่มนั้น (เรียงตามลำดับเดิมของ queryset) |

> **ข้อจำกัดสำคัญที่พลาดกันบ่อย**: `regroup` **จัดกลุ่มเฉพาะรายการที่ติดกันเท่านั้น**
> ถ้าข้อมูลไม่ได้เรียงลำดับตาม field ที่ group มาก่อน (เช่น queryset เรียงตามชื่อเรื่อง
> แทนที่จะเรียงตามปี) ผลลัพธ์จะมีกลุ่มปีเดียวกันแยกเป็นหลายก้อนไม่ติดกัน **เสมอต้อง
> `order_by()` field เดียวกับที่จะ `regroup by` มาก่อนใน view เสมอ**

### 274.4 `{% widthratio %}`: คำนวณสัดส่วน (มักใช้ทำ Progress Bar)

`{% widthratio value max_value max_width %}` คำนวณสูตร `(value / max_value) * max_width`
แล้วปัดเศษเป็นจำนวนเต็ม — ตัวอย่างการใช้ทำแถบความคืบหน้า (progress bar):

```html
<!-- สมมติ context มี survey.answered_count = 45, survey.total_count = 60 -->
<div class="progress-bar">
    <div class="progress-bar__fill"
         style="width: {% widthratio survey.answered_count survey.total_count 100 %}%;">
    </div>
</div>
<p>ตอบแล้ว {{ survey.answered_count }}/{{ survey.total_count }} ข้อ
   ({% widthratio survey.answered_count survey.total_count 100 %}%)</p>
```

จากตัวอย่าง: `(45 / 60) * 100 = 75` ดังนั้นได้ `width: 75%;` และข้อความ `75%` — tag นี้
มีประโยชน์เฉพาะกรณีคำนวณสัดส่วนง่าย ๆ เพื่อการแสดงผลเท่านั้น ถ้าต้องคำนวณซับซ้อนกว่านี้
(เช่นปัดเศษทศนิยม, มีเงื่อนไขพิเศษ) ควรคำนวณใน View แล้วส่งค่าสำเร็จรูปเข้า context แทน

### 274.5 `{% firstof %}`: แสดงค่าแรกที่เป็น Truthy

`{% firstof %}` รับตัวแปรหลายตัว แล้วแสดง**ตัวแรกที่เป็นค่า truthy** (ไม่ใช่ `""`,
`None`, `0`, `False`, list/dict ว่าง) ถ้าไม่มีตัวไหนเป็น truthy เลย จะไม่แสดงอะไร
เว้นแต่จะใส่ค่าสตริงสำรองไว้เป็นตัวสุดท้าย:

```html
{% firstof post.subtitle post.excerpt post.title "ไม่มีชื่อบทความ" %}
```

เทียบเท่ากับการเขียนแบบยาวด้วย `{% if %}`:

```html
{% if post.subtitle %}{{ post.subtitle }}
{% elif post.excerpt %}{{ post.excerpt }}
{% elif post.title %}{{ post.title }}
{% else %}ไม่มีชื่อบทความ
{% endif %}
```

`firstof` กระชับกว่ามากเมื่อมี fallback หลายชั้น — ต่างจาก filter `default` (Part 008
ข้อ 74.2) ตรงที่ `default` เช็คแค่ตัวแปรเดียว ในขณะที่ `firstof` เช็คหลายตัวแปรพร้อมกัน
ตามลำดับความสำคัญ

### 274.6 ตารางสรุป Built-in Tags ในขั้นตอนนี้

| Tag | หน้าที่หลัก | ใช้บ่อยกับ |
|---|---|---|
| `{% spaceless %}` | ลบ whitespace ระหว่าง HTML tag | Email template, CSS inline spacing |
| `{% cycle %}` | สลับค่าไปเรื่อย ๆ ตามรอบ | Zebra striping ตาราง, สลับสี card |
| `{% regroup %}` | จัดกลุ่ม list แบนให้เป็นกลุ่มตาม attribute | หน้าอาร์ไคฟ์ตามปี/เดือน, จัดหมวดหมู่สินค้า |
| `{% widthratio %}` | คำนวณสัดส่วนเชิงเส้น | Progress bar, แถบเปอร์เซ็นต์ |
| `{% firstof %}` | แสดงค่าแรกที่เป็น truthy | Fallback หลายชั้นสำหรับ title/description |

---

## ขั้นตอนที่ 275: `{% verbatim %}`, `{% now %}` แบบเจาะลึก, `{% templatetag %}`

### 275.1 ปัญหา: DTL กับ JavaScript Framework ใช้ `{{ }}` เหมือนกัน

Vue.js, Alpine.js (จะเรียนเต็มใน Phase 6) และ framework แนวเดียวกันใช้ `{{ }}` เป็น
syntax สำหรับ **interpolation ฝั่ง JavaScript** เช่นเดียวกับ DTL ที่ใช้ `{{ }}` สำหรับ
**ตัวแปรฝั่ง Django** — ถ้าเขียนโค้ด Vue ไว้ในไฟล์ template ของ Django ตรง ๆ โดยไม่ทำ
อะไรเลย Django จะพยายาม parse `{{ message }}` ของ Vue เป็นตัวแปร Django (แล้วมักจะ
ได้ค่าว่างเปล่า เพราะ `message` ไม่มีอยู่ใน context ของ Django จริง ๆ)

```html
<!-- ❌ ปัญหา: Django จะพยายาม render {{ message }} เป็นตัวแปร Django ก่อนส่งให้ Vue -->
<div id="app">
    <p>{{ message }}</p>
</div>
```

### 275.2 `{% verbatim %}`: บอก Django ว่า "อย่ายุ่งกับส่วนนี้เลย"

`{% verbatim %}...{% endverbatim %}` บอก Django ให้ปล่อยเนื้อหาข้างในผ่านไปตรง ๆ
โดยไม่ประมวลผล DTL syntax ใด ๆ เลย (ทั้ง `{{ }}` และ `{% %}`) ทำให้ syntax ของ Vue.js
รอดพ้นจากการถูก Django parse:

```html
<!-- blog/templates/blog/comment_widget.html -->
{% load static %}

<div id="comment-app">
    {% verbatim %}
    <div v-if="loading">กำลังโหลดความคิดเห็น...</div>
    <ul v-else>
        <li v-for="comment in comments" :key="comment.id">
            <strong>{{ comment.author }}</strong>: {{ comment.body }}
        </li>
    </ul>
    <button @click="loadMore">โหลดเพิ่มเติม</button>
    {% endverbatim %}
</div>

<script src="{% static 'blog/js/vendor/vue.global.js' %}"></script>
<script>
    const { createApp } = Vue;
    createApp({
        data() {
            return { loading: true, comments: [] };
        },
        // ตัวแปร apiUrl นี้มาจาก Django จริง ๆ (นอก verbatim จึง render ปกติ)
        computed: {
            apiUrl() {
                return "{% url 'blog:api_comments' post.id %}";
            }
        },
        methods: {
            async loadMore() {
                const res = await fetch(this.apiUrl);
                this.comments = await res.json();
                this.loading = false;
            }
        }
    }).mount('#comment-app');
</script>
```

สังเกตรูปแบบสำคัญ: **ส่วนที่เป็น Vue template syntax (`{{ comment.author }}`,
`v-if`, `v-for`, `@click`) ต้องอยู่ใน `{% verbatim %}`** ในขณะที่ **ส่วนที่เป็น Django
syntax จริง ๆ (`{% url %}`, `{% static %}`)** ต้องอยู่**นอก** `{% verbatim %}` เสมอ
เพราะข้างใน `verbatim` Django จะไม่ประมวลผลอะไรเลย รวมถึง tag ของ Django เองด้วย

> **ทางเลือกสมัยใหม่**: ถ้าโปรเจกต์ใช้ Vue 3 คุณสามารถตั้งค่า **custom delimiter**
> ของ Vue เป็นสัญลักษณ์อื่นแทน `{{ }}` (เช่น `[[ ]]`) เพื่อไม่ให้ชนกับ DTL เลยตั้งแต่
> ต้น ซึ่งบางทีมเลือกทำแทนการใช้ `verbatim` ทุกจุด — แต่ `verbatim` ยังจำเป็นเสมอเมื่อ
> ทำงานกับ Vue component ของบุคคลที่สามที่กำหนด delimiter ตายตัวมาแล้ว หรือเมื่อผสม
> กับ Alpine.js ที่ไม่รองรับการเปลี่ยน delimiter

### 275.3 `{% now %}` แบบเจาะลึก: ตัวอักษร Format เพิ่มเติมและ Format Constant

Part 008 (ข้อ 76.2) แนะนำ `{% now "Y" %}` ไปแล้วสำหรับปีลิขสิทธิ์ท้าย footer
ขั้นตอนนี้ขยายความสามารถของ `{% now %}` ให้ครบขึ้น:

```html
{% now "d/m/Y" %}                 <!-- 26/09/2026 -->
{% now "D, d M Y H:i" %}          <!-- Sat, 26 Sep 2026 14:30 -->
{% now "jS F Y" %}                <!-- 26th September 2026 -->
{% now "SHORT_DATE_FORMAT" %}     <!-- ใช้ format ที่ตั้งไว้ใน settings (locale-aware) -->
{% now "DATETIME_FORMAT" %}       <!-- ใช้ format วันที่+เวลาเต็มรูปแบบจาก settings -->
```

**ใช้ format constant จาก settings แทน string ตายตัว**: ถ้าใส่ชื่อ format ที่ตรงกับ
ตัวแปรใน `settings.py` เป๊ะ ๆ (เช่น `SHORT_DATE_FORMAT`, `DATE_FORMAT`,
`DATETIME_FORMAT`, `TIME_FORMAT`) Django จะดึงรูปแบบจาก settings แทนที่จะตีความเป็น
ตัวอักษร format ตรง ๆ ประโยชน์คือถ้าเปลี่ยนรูปแบบวันที่มาตรฐานของทั้งเว็บไซต์ทีเดียวใน
`settings.py` ทุกจุดที่ใช้ `{% now "SHORT_DATE_FORMAT" %}` จะเปลี่ยนตามโดยอัตโนมัติ
โดยไม่ต้องไล่แก้ทุกเทมเพลต:

```python
# config/settings.py
USE_I18N = True
DATE_FORMAT = 'j F Y'          # ตัวอย่างการกำหนดรูปแบบวันที่มาตรฐานของทั้งเว็บไซต์
SHORT_DATE_FORMAT = 'd/m/Y'
```

**เก็บค่าไว้ใช้ซ้ำด้วย `as`** (เหมือน `cycle` ในข้อ 274.2):

```html
{% now "Y" as current_year %}
<p>&copy; {{ current_year }} Django Mastery Course</p>
<meta name="year" content="{{ current_year }}">
```

**ข้อควรระวังเรื่อง Timezone**: `{% now %}` ใช้เวลาปัจจุบันตาม `TIME_ZONE` ที่ตั้งไว้ใน
`settings.py` (ทบทวนแนวคิด timezone-aware datetime จาก Part 011) ถ้า `USE_TZ = True`
(ค่าเริ่มต้นของโปรเจกต์ใหม่ตั้งแต่ Django 5.0) เวลาที่แสดงจะถูกแปลงเป็น local time
ตาม `TIME_ZONE` โดยอัตโนมัติ ไม่ต้องแปลงเองในเทมเพลต

### 275.4 `{% templatetag %}`: แสดงอักขระ DTL แบบตัวอักษรจริง

ปัญหา: ถ้าต้องการแสดงข้อความที่**มีสัญลักษณ์ `{%` หรือ `{{` ปรากฏจริง ๆ บนหน้าเว็บ**
(เช่น หน้าเอกสารสอน DTL แบบที่คุณกำลังอ่านอยู่นี้!) จะเขียนยังไงในเมื่อ Django จะพยายาม
ตีความสัญลักษณ์เหล่านั้นเป็น tag/variable ทันที

```html
{# ❌ ผิด: Django จะพยายาม parse {% block content %} เป็น tag จริง แล้ว error #}
<code>อธิบาย: {% block content %}...{% endblock %}</code>
```

`{% templatetag %}` แก้ปัญหานี้โดยรับชื่อ argument ที่กำหนดไว้ตายตัว แล้วคืนอักขระ
ตัวอักษรล้วน ๆ:

| Argument | ผลลัพธ์ที่ได้ |
|---|---|
| `openblock` | `{%` |
| `closeblock` | `%}` |
| `openvariable` | `{{` |
| `closevariable` | `}}` |
| `openbrace` | `{` |
| `closebrace` | `}` |
| `opencomment` | `{#` |
| `closecomment` | `#}` |

```html
<code>
    {% templatetag openblock %} block content {% templatetag closeblock %}
    ...
    {% templatetag openblock %} endblock {% templatetag closeblock %}
</code>
```

ผลลัพธ์ที่ render ออกมาบนหน้าเว็บจะเป็นข้อความ `{% block content %} ...
{% endblock %}` ตัวอักษรล้วน ๆ ที่เบราว์เซอร์แสดงตรง ๆ โดยไม่ทำให้ Django สับสน
— tag นี้ใช้บ่อยที่สุดในหน้าเอกสาร/บทเรียนที่ต้องยกตัวอย่าง DTL syntax ให้ผู้อ่านเห็น
(หมายเหตุ: ในทางปฏิบัติจริง วิธีที่นิยมกว่าคือใช้ `{% verbatim %}` ครอบทั้งก้อนแทน
ถ้าต้องแสดงหลายบรรทัดต่อเนื่อง แต่ `templatetag` ยังจำเป็นเมื่อเขียนแทรกกลางประโยค
ปกติที่ไม่ต้องการครอบทั้ง block)

---

## ขั้นตอนที่ 276: Template สำหรับอีเมล — Multipart HTML/Plain-Text

### 276.1 ทำไมอีเมลต้องมีทั้งเวอร์ชัน HTML และ Plain-Text

อีเมลที่ส่งจากระบบจริง (ยืนยันคำสั่งซื้อ, รีเซ็ตรหัสผ่าน, แจ้งเตือน) ควรส่งแบบ
**multipart/alternative** คือมีทั้งเวอร์ชัน HTML (สวยงาม มีสไตล์) และเวอร์ชัน
plain-text (ข้อความล้วน) พร้อมกันในอีเมลเดียว ด้วยเหตุผล:

| เหตุผล | รายละเอียด |
|---|---|
| **Mail client บางตัวไม่แสดง HTML** | Mail client เก่าหรือที่ตั้งค่าปิด HTML จะแสดง plain-text แทนโดยอัตโนมัติ |
| **Spam filter** | อีเมลที่มีแค่ HTML อย่างเดียว (ไม่มี plain-text) มีความเสี่ยงถูกจัดเป็นสแปมสูงกว่า |
| **Accessibility** | Screen reader และผู้ใช้บาง tool อ่าน plain-text ได้ตรงไปตรงมากว่า |
| **ผู้ใช้เลือกปิด HTML เอง** | ผู้ใช้จำนวนหนึ่งตั้งค่าไคลเอนต์อีเมลให้แสดงเฉพาะ plain-text ด้วยเหตุผลความเป็นส่วนตัว/ความปลอดภัย |

### 276.2 `EmailMultiAlternatives` เทียบกับ `send_mail`

Django มีทั้ง `send_mail()` (shortcut ง่าย ๆ ส่งได้แค่ plain-text) และ
`EmailMultiAlternatives` (ยืดหยุ่นกว่า รองรับหลาย content type ในอีเมลเดียว):

| | `send_mail()` | `EmailMultiAlternatives` |
|---|---|---|
| รองรับ HTML + plain-text พร้อมกัน | ❌ (ส่งได้แบบเดียว) | ✅ |
| รองรับไฟล์แนบ (attachment) | ❌ | ✅ |
| ความซับซ้อนในการเขียน | ต่ำ | ปานกลาง |
| เหมาะกับ | อีเมลแจ้งเตือนภายในทีม, log สั้น ๆ | อีเมลที่ส่งถึงผู้ใช้จริง (ยืนยัน, ต้อนรับ, ใบเสร็จ) |

### 276.3 เตรียม Template สำหรับอีเมล: HTML และ Plain-Text แยกไฟล์

```html
<!-- templates/emails/base_email.html -->
<!DOCTYPE html>
<html lang="th">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>{% block email_title %}Django Mastery Blog{% endblock %}</title>
</head>
<body style="margin:0; padding:0; background-color:#f4f4f5; font-family: Arial, Helvetica, sans-serif;">
    {% spaceless %}
    <table role="presentation" width="100%" cellpadding="0" cellspacing="0" style="background-color:#f4f4f5;">
        <tr>
            <td align="center" style="padding: 24px 16px;">
                <table role="presentation" width="480" cellpadding="0" cellspacing="0"
                       style="background-color:#ffffff; border-radius:8px; overflow:hidden;">
                    <tr>
                        <td style="background-color:#0b5fff; padding:20px 24px;">
                            <span style="color:#ffffff; font-size:20px; font-weight:bold;">
                                Django Mastery Blog
                            </span>
                        </td>
                    </tr>
                    <tr>
                        <td style="padding:24px;">
                            {% block email_content %}{% endblock %}
                        </td>
                    </tr>
                    <tr>
                        <td style="padding:16px 24px; background-color:#f4f4f5; font-size:12px; color:#6b7280;">
                            &copy; {% now "Y" %} Django Mastery Course
                        </td>
                    </tr>
                </table>
            </td>
        </tr>
    </table>
    {% endspaceless %}
</body>
</html>
```

> **ทำไมต้องแยก `base_email.html` ต่างหาก ไม่ extends จาก `base.html` ของเว็บไซต์**:
> อีเมล HTML ต้องเขียนด้วย `<table>` layout และ inline style เป็นหลัก (mail client
> จำนวนมากไม่รองรับ CSS ภายนอกหรือ Flexbox/Grid สมัยใหม่) ต่างจาก HTML หน้าเว็บทั่วไป
> โดยสิ้นเชิง การพยายาม extends จาก `base.html` ของเว็บไซต์จะทำให้ได้โครง `<nav>`,
> `<footer>` ที่ไม่เหมาะกับอีเมลติดมาด้วย จึงควรมี **สาย inheritance ของตัวเองแยก
> ต่างหากสำหรับโซนอีเมลโดยเฉพาะ** — สังเกตว่านี่คือหลักการเดียวกับข้อ 271.5
> (wide-and-shallow tree) เพียงแต่ `base_email.html` เป็นอีกต้นไม้ที่แยกจาก
> `base.html` ของเว็บไซต์โดยสิ้นเชิง ไม่ใช่กิ่งเดียวกัน

```html
<!-- templates/emails/order_confirmation.html -->
{% extends 'emails/base_email.html' %}

{% block email_title %}ยืนยันคำสั่งซื้อ #{{ order.id }}{% endblock %}

{% block email_content %}
<h1 style="font-size:18px; color:#1a1a1a; margin-top:0;">
    สวัสดีคุณ {{ order.customer_name }}
</h1>
<p style="font-size:14px; color:#374151; line-height:1.6;">
    ขอบคุณสำหรับคำสั่งซื้อหมายเลข <strong>#{{ order.id }}</strong>
    ยอดรวมทั้งสิ้น <strong>{{ order.total_amount }} บาท</strong>
</p>
<table role="presentation" width="100%" cellpadding="6" cellspacing="0"
       style="border-collapse:collapse; margin:16px 0; font-size:14px;">
    {% for item in order.items.all %}
    <tr style="border-bottom:1px solid #e5e7eb;">
        <td>{{ item.product_name }} x{{ item.quantity }}</td>
        <td align="right">{{ item.subtotal }} บาท</td>
    </tr>
    {% endfor %}
</table>
<p style="text-align:center; margin:24px 0;">
    <a href="{{ order_detail_url }}"
       style="background-color:#0b5fff; color:#ffffff; padding:12px 24px;
              text-decoration:none; border-radius:6px; display:inline-block;">
        ดูรายละเอียดคำสั่งซื้อ
    </a>
</p>
{% endblock %}
```

```text
{# templates/emails/order_confirmation.txt #}
สวัสดีคุณ {{ order.customer_name }}

ขอบคุณสำหรับคำสั่งซื้อหมายเลข #{{ order.id }}
ยอดรวมทั้งสิ้น {{ order.total_amount }} บาท

รายการสินค้า:
{% for item in order.items.all %}- {{ item.product_name }} x{{ item.quantity }} — {{ item.subtotal }} บาท
{% endfor %}
ดูรายละเอียดคำสั่งซื้อได้ที่: {{ order_detail_url }}

ขอบคุณที่ใช้บริการ Django Mastery Blog
```

> **หมายเหตุ**: เทมเพลต `.txt` ก็ใช้ DTL syntax ได้ปกติทุกประการ (Django ไม่สนใจ
> นามสกุลไฟล์เวลา render) — `{% for %}`, `{{ }}` ทำงานเหมือนกันทุกอย่าง เพียงแต่ไม่มี
> HTML tag ล้อมรอบเท่านั้น

### 276.4 ฟังก์ชันส่งอีเมลที่ Render ทั้งสองเทมเพลตพร้อมกัน

```python
# orders/emails.py
from django.core.mail import EmailMultiAlternatives
from django.template.loader import render_to_string
from django.conf import settings


def send_order_confirmation_email(order, request=None):
    """ส่งอีเมลยืนยันคำสั่งซื้อแบบ multipart (HTML + plain-text)"""
    context = {
        'order': order,
        # ต้องใช้ absolute URL เสมอในอีเมล เพราะไม่มี "หน้าเว็บปัจจุบัน" ให้ resolve
        # relative URL ได้เหมือนตอน render หน้าเว็บปกติ (ทบทวน build_absolute_uri
        # จาก Part 006 เรื่อง reverse())
        'order_detail_url': (
            request.build_absolute_uri(order.get_absolute_url())
            if request
            else f'https://djangomastery.example.com{order.get_absolute_url()}'
        ),
    }

    subject = f'ยืนยันคำสั่งซื้อ #{order.id} — Django Mastery Blog'
    text_body = render_to_string('emails/order_confirmation.txt', context)
    html_body = render_to_string('emails/order_confirmation.html', context)

    email = EmailMultiAlternatives(
        subject=subject,
        body=text_body,                       # เนื้อหาหลัก (plain-text) — บังคับ
        from_email=settings.DEFAULT_FROM_EMAIL,
        to=[order.customer_email],
    )
    email.attach_alternative(html_body, 'text/html')   # เนื้อหาทางเลือก (HTML)
    email.send(fail_silently=False)
```

```python
# orders/views.py (ตัวอย่างการเรียกใช้จาก view จริง)
from django.shortcuts import redirect, get_object_or_404
from .models import Order
from .emails import send_order_confirmation_email


def confirm_order(request, order_id):
    order = get_object_or_404(Order, pk=order_id)
    order.status = 'confirmed'
    order.save()
    send_order_confirmation_email(order, request=request)
    return redirect('orders:detail', pk=order.id)
```

### 276.5 ตั้งค่า Email Backend สำหรับพัฒนา (Console Backend)

ระหว่างพัฒนา ยังไม่ต้องต่อกับ SMTP server จริง Django มี **console backend** ที่พิมพ์
เนื้อหาอีเมลออกทาง terminal แทนการส่งจริง — สะดวกมากสำหรับทดสอบ:

```python
# config/settings.py

# ระหว่างพัฒนา: พิมพ์อีเมลออกทาง terminal แทนการส่งจริง
EMAIL_BACKEND = 'django.core.mail.backends.console.EmailBackend'
DEFAULT_FROM_EMAIL = 'no-reply@djangomastery.example.com'

# production จริงจะเปลี่ยนเป็น SMTP backend (รายละเอียดเต็มเรื่อง deployment
# และตัวแปร environment จะอยู่ใน Phase 9)
# EMAIL_BACKEND = 'django.core.mail.backends.smtp.EmailBackend'
# EMAIL_HOST = 'smtp.sendgrid.net'
# EMAIL_PORT = 587
# EMAIL_USE_TLS = True
# EMAIL_HOST_USER = os.environ.get('EMAIL_HOST_USER')
# EMAIL_HOST_PASSWORD = os.environ.get('EMAIL_HOST_PASSWORD')
```

### 276.6 ข้อควรระวังเมื่อเขียน Email Template

| ข้อควรระวัง | เหตุผล |
|---|---|
| ใช้ inline style (`style="..."`) แทน `<link>` หรือ `<style>` block เสมอ | Mail client จำนวนมาก (โดยเฉพาะ Gmail บนมือถือ) ตัด `<style>` block ทิ้งหรือไม่รองรับ CSS ภายนอกเลย |
| ใช้ `<table>` layout แทน Flexbox/Grid | Outlook desktop ยังใช้ Word rendering engine ซึ่งไม่รองรับ CSS layout สมัยใหม่ |
| URL ทุกอันต้องเป็น absolute URL เสมอ | ไม่มี "หน้าปัจจุบัน" ให้ resolve relative path ได้เหมือนหน้าเว็บ — ใช้ `request.build_absolute_uri()` หรือกำหนด domain เต็มไว้ใน settings |
| ห้ามใช้ `{% static %}` ตรง ๆ โดยไม่แปลงเป็น absolute URL | ไฟล์ static ในอีเมลต้องเป็น URL เต็มที่เข้าถึงได้จากอินเทอร์เน็ตจริง ไม่ใช่ path สัมพัทธ์ของเว็บไซต์ |
| ทดสอบใน mail client หลายตัวก่อน deploy จริง | HTML/CSS ที่แสดงถูกใน Gmail อาจเพี้ยนใน Outlook — เครื่องมืออย่าง Litmus หรือ Email on Acid ช่วยตรวจสอบได้ (แนะนำเมื่อโปรเจกต์เข้าสู่ระดับ production) |

---

## ขั้นตอนที่ 277: การจัดโครงสร้าง Template Tree ขนาดใหญ่แบบมืออาชีพ

### 277.1 ปัญหาของโครงสร้างแบบ Part 008 เมื่อโปรเจกต์โตขึ้น

โครงสร้างจาก Part 008 (`templates/base.html` + `templates/partials/`) ใช้ได้ดีตอน
โปรเจกต์มีแค่แอปเดียว แต่เมื่อโปรเจกต์ขยายเป็นหลายสิบแอป (ตามที่จะเกิดขึ้นตลอด
หลักสูตรนี้) โฟลเดอร์ `templates/` ระดับโปรเจกต์จำเป็นต้องมีการจัดหมวดหมู่ที่ชัดเจนขึ้น
ไม่เช่นนั้นจะกลายเป็นโฟลเดอร์รวมไฟล์สิบ ๆ ไฟล์ที่ไม่มีระเบียบ

### 277.2 โครงสร้างมาตรฐานที่แนะนำสำหรับโปรเจกต์ขนาดใหญ่

```
templates/                          ← ระดับโปรเจกต์ (ตั้งค่าใน DIRS ตาม Part 008 ข้อ 71.5)
├── base/
│   ├── base.html                   ← โครงหลักทั้งเว็บไซต์ (เดิมคือ base.html เฉย ๆ)
│   ├── base_auth.html              ← โครงสำหรับหน้า login/register (ไม่มี navbar เต็ม)
│   └── base_dashboard.html         ← โครงสำหรับหน้า dashboard ภายใน (มี sidebar เมนู)
├── partials/
│   ├── navbar.html
│   ├── footer.html
│   ├── pagination.html             ← ใช้ร่วมกันได้ทุกหน้าที่มี pagination (Part 022)
│   ├── messages.html               ← แสดง Django messages framework แบบมาตรฐาน
│   └── seo_meta.html               ← meta tag พื้นฐาน (og:*, twitter:*) ให้ include ได้ทุกหน้า
├── emails/
│   ├── base_email.html
│   ├── base_email.txt
│   ├── order_confirmation.html
│   └── order_confirmation.txt
└── errors/
    ├── 404.html                    ← ทบทวนจาก Part 006 ข้อ 58
    └── 500.html

blog/templates/blog/                ← ระดับแอป (เฉพาะของแอป blog เท่านั้น)
├── base_blog.html
├── list.html
├── post_detail.html
├── archive.html
└── _post_card.html                 ← partial เฉพาะแอป blog (ใช้ underscore ตาม Part 008 ข้อ 76.5)
```

### 277.3 ตารางหลักการแบ่งโฟลเดอร์: อะไรควรอยู่ระดับโปรเจกต์ vs ระดับแอป

| โฟลเดอร์ | เก็บอะไร | ตัวอย่าง | อยู่ระดับ |
|---|---|---|---|
| `templates/base/` | โครง "แม่" ที่มากกว่า 1 แอปอาจ extends ร่วมกัน หรือเป็นโครงของทั้งเว็บไซต์ | `base.html`, `base_dashboard.html` | โปรเจกต์ |
| `templates/partials/` | ชิ้นส่วนที่ใช้ซ้ำข้ามหลายแอป | navbar, footer, pagination, messages | โปรเจกต์ |
| `templates/emails/` | ทุกเทมเพลตอีเมลของทั้งระบบ (มักไม่ผูกกับแอปใดแอปหนึ่งเพราะอีเมลเกี่ยวข้องกับหลายแอป) | order confirmation, welcome email, password reset | โปรเจกต์ |
| `templates/errors/` | หน้า error กำหนดเอง | `404.html`, `500.html`, `403.html` | โปรเจกต์ |
| `<app>/templates/<app>/base_<app>.html` | โครงเฉพาะโซนของแอปเดียว | `blog/base_blog.html` | แอป |
| `<app>/templates/<app>/_xxx.html` | Partial ที่ใช้เฉพาะภายในแอปนั้น (ไม่มีแอปอื่นเรียกใช้) | `blog/_post_card.html` | แอป |
| `<app>/templates/<app>/*.html` (ไม่มี underscore) | หน้าเพจจริงที่ผูกกับ URL/View โดยตรง | `list.html`, `post_detail.html` | แอป |

**หลักการตัดสินใจแบบง่าย**: ถามตัวเองว่า *"ถ้าลบแอปนี้ทิ้งไปพรุ่งนี้ ไฟล์เทมเพลตนี้
ควรหายไปด้วยไหม?"* ถ้าคำตอบคือ "ใช่" → อยู่ในแอป ถ้าคำตอบคือ "ไม่ ไฟล์นี้ยังมี
ประโยชน์กับแอปอื่นด้วย" → อยู่ระดับโปรเจกต์

### 277.4 ตั้งค่า `DIRS` ให้รองรับโครงสร้างย่อยนี้

ข่าวดี: **ไม่ต้องแก้ `TEMPLATES.DIRS` เพิ่มเลย** จากที่ตั้งไว้ใน Part 008 ข้อ 71.5
เพราะ `DIRS` ชี้ไปที่โฟลเดอร์ `templates/` ทั้งก้อนอยู่แล้ว โฟลเดอร์ย่อยข้างในอย่าง
`base/`, `partials/`, `emails/`, `errors/` เป็นเพียงการจัดกลุ่มด้วย path ภายใน ไม่ใช่
การเพิ่ม search path ใหม่ — สิ่งที่เปลี่ยนไปมีแค่ path ที่อ้างอิงตอนเรียก `extends`/
`include`/`render_to_string` เท่านั้น:

```python
# config/settings.py (เหมือนเดิมจาก Part 008 ข้อ 71.5 — ไม่ต้องแก้)
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
            ],
        },
    },
]
```

```html
<!-- blog/templates/blog/base_blog.html — path อ้างอิงเปลี่ยนตามโครงสร้างใหม่ -->
{% extends 'base/base.html' %}
```

```python
# ตัวอย่างการเรียก error handler ใน urls.py ระดับโปรเจกต์ (ทบทวน Part 006 ข้อ 58)
# config/urls.py
handler404 = 'django.views.defaults.page_not_found'
```

```python
# config/views.py (ตัวอย่าง view ที่ระบุ path เต็มของ error template ใหม่)
def custom_404(request, exception):
    return render(request, 'errors/404.html', status=404)
```

### 277.5 Block Contract ที่ต้นไฟล์: ธรรมเนียมที่ควรทำกับทุก Base Template

ทบทวนจากข้อ 271.3 — ทุกไฟล์ที่อยู่ในโฟลเดอร์ `base/` ควรมีคอมเมนต์บอก "สัญญาของ
block" ไว้บนสุดเสมอ เพื่อให้เทมเพลตลูกไม่ต้องเดา:

```html
<!-- templates/base/base_dashboard.html -->
{#
  Block Contract:
  - dashboard_title  : ชื่อหัวข้อในแถบด้านบนของ dashboard (ค่าเริ่มต้น: "Dashboard")
  - dashboard_actions: ปุ่ม action มุมขวาบน เช่น "เพิ่มใหม่" (ค่าเริ่มต้น: ว่าง)
  - dashboard_content: เนื้อหาหลัก (บังคับ override เสมอ)
#}
{% extends 'base/base.html' %}
{# ... เนื้อหาของ dashboard layout ... #}
```

การเขียน contract แบบนี้อาจดูเหมือนงานเพิ่ม แต่ในทีมที่มีนักพัฒนาหลายคน (หรือแม้แต่
ทำคนเดียวในโปรเจกต์ระยะยาว) มันช่วยประหยัดเวลาการอ่านโค้ดมหาศาลเมื่อกลับมาแก้ไขใน
อีกหลายเดือนข้างหน้า

---

## ขั้นตอนที่ 278: ข้อควรระวังด้าน Performance ของ Template

### 278.1 Template ไม่ควรมี Logic หนัก — ทบทวนหลักการจาก Part 008 ข้อ 71.7

Part 008 ย้ำหลักการ **"Template ไม่ควรมี business logic"** ไว้แล้วในเชิงการออกแบบโค้ด
ขั้นตอนนี้จะเจาะลึกในมุม **performance** ว่าทำไมการฝ่าฝืนหลักการนี้ถึงทำให้เว็บไซต์ช้าลง
จริง ๆ ไม่ใช่แค่เรื่องความสวยงามของโค้ดเท่านั้น

### 278.2 อันตรายที่พบบ่อยที่สุด: N+1 Query ที่ซ่อนอยู่ใน Template

```html
<!-- ❌ อันตราย: ทริกเกอร์ query ใหม่ทุกครั้งที่วนลูป -->
{% for post in posts %}
    <div class="post-card">
        <h2>{{ post.title }}</h2>
        <p>โดย {{ post.author.get_full_name }}</p>          {# query ที่ 1 ต่อโพสต์ #}
        <p>หมวดหมู่: {{ post.category.name }}</p>            {# query ที่ 2 ต่อโพสต์ #}
        <p>ความคิดเห็น {{ post.comments.count }} รายการ</p>  {# query ที่ 3 ต่อโพสต์ #}
    </div>
{% endfor %}
```

โค้ด template ด้านบนดู "บริสุทธิ์" มาก — ไม่มี logic ซับซ้อนเลยแม้แต่น้อย แต่ทุกครั้ง
ที่ dot-lookup ไปถึง `post.author`, `post.category`, `post.comments.count` (ทบทวน
กลไก dot-lookup จาก Part 008 ข้อ 72.3) ถ้า queryset ต้นทางไม่ได้เตรียม
`select_related`/`prefetch_related` ไว้ล่วงหน้า (ทบทวน Part 013-014) แต่ละ dot-lookup
นี้จะยิง query ใหม่ไปยังฐานข้อมูล **แยกกันทุกแถว** — ถ้ามี 50 โพสต์ หน้านี้อาจยิง query
ถึง 150+ ครั้งโดยที่ template เองไม่มีคำสั่งอะไรที่ "ดูเหมือน" จะ query เลยแม้แต่บรรทัด
เดียว

**ทางแก้อยู่ที่ View ไม่ใช่ Template**:

```python
# ✅ แก้ที่ view — เตรียมข้อมูลที่ template จะใช้ไว้ล่วงหน้าในคำสั่งเดียว
def post_list(request):
    posts = (
        Post.objects.filter(is_published=True)
        .select_related('author', 'category')   # JOIN ครั้งเดียว แทน query แยกทุกแถว
        .prefetch_related('comments')             # query แยก 1 ครั้ง แทนที่จะแยกทุกแถว
        .annotate(comments_count=Count('comments'))  # คำนวณจำนวนล่วงหน้าในฐานข้อมูล
    )
    return render(request, 'blog/list.html', {'posts': posts})
```

```html
<!-- ✅ template เดิม แทบไม่ต้องแก้อะไรเลย แค่เปลี่ยน comments.count เป็น comments_count -->
{% for post in posts %}
    <div class="post-card">
        <h2>{{ post.title }}</h2>
        <p>โดย {{ post.author.get_full_name }}</p>
        <p>หมวดหมู่: {{ post.category.name }}</p>
        <p>ความคิดเห็น {{ post.comments_count }} รายการ</p>
    </div>
{% endfor %}
```

หลักการสำคัญ: **Template ไม่มีทางรู้ล่วงหน้าว่า dot-lookup แต่ละจุดจะทำให้เกิด query
ใหม่หรือไม่ — หน้าที่ป้องกันปัญหานี้จึงตกอยู่ที่ View เสมอ** ต้องคิดล่วงหน้าตั้งแต่ตอน
เตรียม queryset ว่า template จะเข้าถึง related field อะไรบ้าง

### 278.3 ตารางสรุป Pattern ที่ควรหลีกเลี่ยงในการเขียน Template

| Pattern ที่ควรหลีกเลี่ยง | ปัญหา | ทางแก้ |
|---|---|---|
| เรียก related field/method ที่ trigger query ซ้ำใน `{% for %}` | N+1 query (ข้อ 278.2) | `select_related`/`prefetch_related`/`annotate` ใน View |
| Custom template tag ที่ query ฐานข้อมูลทุกครั้งที่ render | ทุกหน้าที่ใช้ tag นั้นช้าลงพร้อมกันหมด (คล้ายปัญหา context processor จาก Part 008 ข้อ 79.5) | Cache ผลลัพธ์ใน tag เอง หรือย้าย logic ไป View/context processor ที่ควบคุม scope ได้ชัดเจนกว่า |
| Filter chain ที่ยาวและซับซ้อนมาก (`{{ x\|f1\|f2\|f3\|f4\|f5 }}`) | อ่านยาก, debug ยาก, และ recompute ทุกครั้งที่ render แม้ค่าจะไม่เปลี่ยน | คำนวณผลลัพธ์สุดท้ายเป็น property/method บน model หรือคำนวณใน View ครั้งเดียว |
| `{% include %}` ไฟล์เดียวกันซ้ำหลายร้อยครั้งในหน้าเดียว (ทบทวน Part 008 ข้อ 76.6) | ต้อง compile/render ไฟล์นั้นซ้ำทุกครั้ง | พิจารณา cached template loader (Part 068) หรือลดจำนวนรายการต่อหน้าด้วย pagination (Part 022) |
| Inheritance chain ลึกเกินไป (ทบทวนข้อ 271.6) | ทุกชั้นต้องถูก parse และ resolve block ตามลำดับ ยิ่งลึกยิ่งมีค่าใช้จ่ายสะสม (แม้จะเล็กน้อยต่อชั้น) และยิ่งยาก debug | จำกัดความลึกไม่เกิน 3-4 ชั้น ตามคำแนะนำในข้อ 271.6 |
| `{% load %}` library ที่ไม่ได้ใช้จริงในไฟล์นั้น | เพิ่มเวลา parse/import โดยไม่จำเป็นเล็กน้อย และทำให้ไม่ชัดเจนว่าไฟล์นี้พึ่งพา tag อะไรจริง ๆ | โหลดเฉพาะ library ที่ใช้จริงในแต่ละไฟล์เท่านั้น |

### 278.4 เกริ่น Template Caching: จะเจาะลึกเต็มรูปแบบใน Part 068

ทางแก้ปัญหา performance ของ template ที่ทรงพลังที่สุดอย่างหนึ่งคือ **Template
Fragment Caching** — เก็บผลลัพธ์ที่ render แล้วของ block บางส่วนไว้ใน cache backend
(เช่น Redis) แล้วนำมาใช้ซ้ำโดยไม่ต้อง render ใหม่ทุกครั้ง เพียงแค่ล้อมด้วย
`{% load cache %}` และ `{% cache %}`:

```html
{% load cache %}
{% cache 500 sidebar_popular_categories %}
    {# เนื้อหาที่ render ยากหรือ query หนัก จะถูก cache ไว้ 500 วินาที #}
    ...
{% endcache %}
```

เราจะ**ยังไม่ใช้** `{% cache %}` ในหลักสูตรตอนนี้ เพราะต้องเข้าใจ **Django Caching
Framework** ทั้งระบบก่อน (การเลือก cache backend, cache key strategy, cache
invalidation) ซึ่งเป็นหัวข้อใหญ่ที่จะเจาะลึกเต็มรูปแบบใน **Part 068: Django Caching
Framework เบื้องต้น** (Phase 8) — ตอนนี้ขอให้จำไว้แค่ว่า**ทางแก้ปัญหา query หนักที่
ถูกต้องที่สุดคือแก้ที่ View ก่อนเสมอ (select_related/prefetch_related/annotate)
ส่วน caching คือชั้นป้องกันเพิ่มเติมสำหรับกรณีที่ query ได้ optimize สุดแล้วแต่ยังหนัก
อยู่ดี** (เช่น query ที่ aggregate ข้อมูลจำนวนมากจริง ๆ)

---

## ขั้นตอนที่ 279: การใช้ Template Engine อื่นร่วมกับ Django (Jinja2 Backend)

### 279.1 ทบทวนจาก Part 008 ข้อ 71.7 — และคำถามที่ยังไม่ได้ตอบ

Part 008 เกริ่นไว้สั้น ๆ ว่า Django รองรับ Jinja2 เป็นทางเลือกแทน DTL ได้ และให้เหตุผล
คร่าว ๆ ว่าทำไมหลักสูตรนี้เลือกใช้ DTL ตลอด ขั้นตอนนี้จะตอบคำถามที่ Part 008 ยังไม่ได้
ลงรายละเอียด: **ถ้าจะใช้ Jinja2 จริง ๆ ต้องตั้งค่าอย่างไร และมันอยู่ร่วมกับ DTL ในโปรเจกต์
เดียวกันได้อย่างไร**

### 279.2 ติดตั้งและตั้งค่า Jinja2 Backend

```bash
pip install Jinja2
```

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
            ],
        },
    },
    {
        'BACKEND': 'django.template.backends.jinja2.Jinja2',
        'DIRS': [BASE_DIR / 'jinja2'],
        'APP_DIRS': True,
        'OPTIONS': {
            'environment': 'config.jinja2_env.environment',
        },
    },
]
```

```python
# config/jinja2_env.py
from django.templatetags.static import static
from django.urls import reverse
from jinja2 import Environment


def environment(**options):
    env = Environment(**options)
    # Jinja2 ไม่มี context processor แบบ DTL — ต้องเพิ่มฟังก์ชัน global เอง
    env.globals.update({
        'static': static,
        'url': reverse,
    })
    return env
```

### 279.3 กลไกสำคัญ: Jinja2 กับ DTL แยกโฟลเดอร์ค้นหากันโดยสิ้นเชิง

ประเด็นที่สำคัญที่สุดเมื่อใช้สอง engine พร้อมกัน: เมื่อ `APP_DIRS = True` **DTL มองหา
โฟลเดอร์ `templates/` ในแต่ละแอป แต่ Jinja2 backend มองหาโฟลเดอร์ `jinja2/` แทน**
(ไม่ใช่ `templates/` เหมือนกัน) — นี่เป็นการออกแบบที่จงใจของ Django เพื่อไม่ให้ไฟล์
ของสอง engine ปะปนกันจนสับสนว่าไฟล์ไหนเป็นของ engine ไหน:

```
blog/
├── templates/blog/list.html      ← DTL จะมองหาที่นี่
└── jinja2/blog/list.html         ← Jinja2 จะมองหาที่นี่ (คนละโฟลเดอร์กันเลย)
```

เมื่อ view เรียก `render(request, 'blog/list.html', context)` Django จะไล่ลองทีละ
engine ตามลำดับที่ประกาศใน `TEMPLATES` (list) — ถ้า engine แรก (DTL) หาไฟล์ชื่อนี้
เจอในโฟลเดอร์ `templates/` ก็ใช้ engine นั้น render ถ้าหาไม่เจอเลยจะลองไล่ engine ถัดไป
(Jinja2) ต่อในโฟลเดอร์ `jinja2/` ของมัน — เพราะฉะนั้นถ้าตั้งชื่อไฟล์ path เดียวกันในทั้ง
สองโฟลเดอร์ **engine ที่ประกาศไว้ก่อนใน list จะถูกใช้ก่อนเสมอ**

### 279.4 ความแตกต่างเชิงไวยากรณ์ที่สำคัญระหว่าง DTL กับ Jinja2

| ประเด็น | Django Template Language (DTL) | Jinja2 |
|---|---|---|
| Python expression เต็มรูปแบบ | ❌ ทำไม่ได้ (จงใจจำกัดตามปรัชญา Part 008 ข้อ 71.7) | ✅ `{{ 1 + 1 }}`, `{{ items[0:5] }}` ทำได้ |
| Macro (ฟังก์ชันใน template) | ❌ ไม่มี — ต้องใช้ inclusion tag แทน (Part 029) | ✅ `{% macro %}...{% endmacro %}` |
| Auto-escaping | เปิดเป็นค่าเริ่มต้านเสมอ | เปิดเป็นค่าเริ่มต้าน **เฉพาะเมื่อผ่าน Django's Jinja2 backend** (Jinja2 เดี่ยว ๆ นอก Django ปิดโดย default) |
| Context Processors | ✅ ใช้ได้ตรง ๆ ตาม Part 008 ข้อ 79 | ❌ ใช้ไม่ได้ — ต้องเพิ่มผ่าน `environment()` function แบบข้อ 279.2 แทน |
| Custom filter/tag ของ 3rd-party app | ส่วนใหญ่เขียนมาให้ DTL โดยตรง ใช้ได้ทันที | มักใช้ไม่ได้ตรง ๆ ต้องเขียน adapter เอง |
| `{% url %}`, `{% static %}` | มีมาให้พร้อมใช้ | ต้อง register เป็น global function เองตามข้อ 279.2 |
| Whitespace control | ควบคุมผ่าน `{% spaceless %}` เท่านั้น | มี `{%-` และ `-%}` ควบคุมละเอียดกว่า |
| ความเร็วในการ render | ปานกลาง | เร็วกว่า (compile เป็น Python bytecode โดยตรง) |

### 279.5 เมื่อไหร่ควรพิจารณาใช้ Jinja2 จริง ๆ

| สถานการณ์ | ควรใช้ Jinja2 ไหม |
|---|---|
| โปรเจกต์ทั่วไป ทีมคุ้นเคย Django มาตลอด | ❌ ใช้ DTL — ความเข้ากันได้กับ ecosystem สำคัญกว่า |
| ต้องการ render จำนวนหน้ามหาศาลต่อวินาที และวัดแล้วว่า template rendering เป็นคอขวดจริง | ✅ พิจารณาได้ — วัดผลก่อนตัดสินใจเสมอ (อย่าเปลี่ยนเพราะ "เขาว่าเร็วกว่า" โดยไม่มีตัวเลขจริง) |
| ทีมย้ายมาจาก Flask และมีเทมเพลต Jinja2 จำนวนมากอยู่แล้ว | ✅ พิจารณาได้ — ลดต้นทุนการเขียนใหม่ |
| ต้องการ macro ที่ทรงพลังกว่า inclusion tag ของ DTL | ✅ พิจารณาได้ แต่ก่อนอื่นควรลองดู custom template tag เต็มรูปแบบใน Part 029 ก่อน เพราะมักตอบโจทย์ได้เพียงพอ |
| ใช้ 3rd-party Django package จำนวนมากที่มี template tag ของตัวเอง | ❌ อยู่กับ DTL — package ส่วนใหญ่ในระบบนิเวศ Django เขียนมาให้ DTL โดยเฉพาะ |

**สรุปจุดยืนของหลักสูตรนี้**: เราจะ**ใช้ DTL ตลอดทั้งหลักสูตร** เหมือนที่ Part 008
ประกาศไว้ ขั้นตอนนี้มีไว้เพื่อให้คุณ**รู้จักและตั้งค่าเป็น** เผื่อวันหนึ่งไปเจอโปรเจกต์
จริงที่ทีมเลือกใช้ Jinja2 — ไม่ใช่เพื่อให้เปลี่ยนมาใช้ในหลักสูตรนี้

---

## ขั้นตอนที่ 280: สรุปและแบบฝึกหัด

### 280.1 สรุปสิ่งที่ได้เรียนรู้ใน Part นี้

- ✅ ออกแบบ Multi-Level Template Inheritance อย่างเป็นระบบ พร้อมกฎการตัดสินใจว่าเมื่อไหร่
  ควรเพิ่มชั้น และเมื่อไหร่จะกลายเป็น "inheritance hell"
- ✅ เข้าใจกลไกเบื้องลึกของ `{{ block.super }}` การไล่ขึ้นหลายชั้น และ nested blocks
  พร้อมกฎการ override บางส่วน
- ✅ เข้าใจกลไกจริงเบื้องหลัง `{% load %}` ว่า Django ค้นหา template tag library
  จากโฟลเดอร์ `templatetags/` ของแอปใน `INSTALLED_APPS` อย่างไร (เตรียมพร้อมสำหรับ
  Part 029)
- ✅ รู้จัก built-in tags เพิ่มเติม: `spaceless`, `cycle`, `regroup`, `widthratio`,
  `firstof`
- ✅ ใช้ `{% verbatim %}` แก้ปัญหา syntax ชนกันกับ Vue.js/Alpine.js, ใช้ `{% now %}`
  แบบเจาะลึกกับ format constant, และใช้ `{% templatetag %}` แสดงอักขระ DTL ตรง ๆ
- ✅ สร้าง Email Template แบบ multipart HTML/plain-text ด้วย `EmailMultiAlternatives`
- ✅ จัดโครงสร้าง Template Tree ขนาดใหญ่แบบมืออาชีพด้วย `base/`, `partials/`,
  `emails/`, `errors/`
- ✅ รู้จักข้อควรระวังด้าน performance ของ template โดยเฉพาะปัญหา N+1 query ที่ซ่อนอยู่
  ใน dot-lookup
- ✅ รู้จักและตั้งค่า Jinja2 backend ได้ พร้อมเข้าใจว่าเมื่อไหร่ควรพิจารณาใช้จริง

### 280.2 Checklist ก่อนไป Part ถัดไป

- [ ] สร้าง `base_blog.html` ที่ extends จาก `base.html` พร้อม sidebar และแถบแท็บ
      ตามตัวอย่างข้อ 271.3 ได้จริง
- [ ] เขียน Block Contract เป็นคอมเมนต์ไว้บนสุดของ base template อย่างน้อย 1 ไฟล์
- [ ] ทดลองใช้ `{{ block.super }}` ไล่ขึ้น 3 ชั้นจริง แล้วดูผลลัพธ์ HTML ที่ได้ด้วย
      "View Page Source"
- [ ] สร้างโฟลเดอร์ `templatetags/` พร้อม `__init__.py` และไฟล์ library เปล่า
      (ยังไม่ต้องเขียน tag จริง — แค่ให้ `{% load %}` ไม่ error)
- [ ] ลองใช้ `{% regroup %}` จัดกลุ่มบทความตามปีจริงอย่างน้อย 1 หน้า
- [ ] สร้าง Email Template คู่ HTML/plain-text อย่างน้อย 1 ชุด และส่งทดสอบผ่าน
      console backend สำเร็จ
- [ ] จัดโครงสร้าง `templates/` ของโปรเจกต์ให้มี `base/`, `partials/`, `emails/`
      ตามข้อ 277.2
- [ ] ตรวจสอบ queryset ของหน้า `list.html` ว่ามี N+1 query หรือไม่ ด้วยการเปิด
      Django Debug Toolbar (หรือนับ query ผ่าน `django.db.connection.queries`)

### 280.3 แบบฝึกหัดท้ายบท

**แบบฝึกหัดที่ 1**: จากแอป `blog` ที่มีอยู่ ให้เพิ่มหน้าใหม่ `blog/by_category.html`
ที่ extends จาก `base_blog.html` (ตามตัวอย่างข้อ 271.3-271.4) แสดงบทความทั้งหมดใน
หมวดหมู่หนึ่ง พร้อม override block `blog_title` ให้แสดงชื่อหมวดหมู่ และ override
`blog_tabs_extra` ให้แสดงจำนวนบทความในหมวดหมู่นั้น

**แบบฝึกหัดที่ 2**: เขียน `templates/base/base_dashboard.html` ขึ้นใหม่ทั้งหมด
(สมมติว่าจะใช้กับ Phase 4 ที่จะมีหน้าจัดการบัญชีผู้ใช้) ที่มี nested blocks อย่างน้อย
2 ชั้น (เช่น `dashboard_content` ที่มี `dashboard_content_header` และ
`dashboard_content_body` ซ้อนอยู่ข้างใน) แล้วสร้างหน้าเดโม 2 หน้าที่ override
คนละ block กัน เพื่อพิสูจน์ว่า nested blocks ทำงานตามที่อธิบายในข้อ 272.3-272.4

**แบบฝึกหัดที่ 3**: สร้างฟังก์ชันส่งอีเมล `send_welcome_email(user)` ที่ใช้
`EmailMultiAlternatives` ส่งอีเมลต้อนรับสมาชิกใหม่ พร้อมเทมเพลตคู่ HTML/plain-text
ของตัวเอง (ไม่ใช่ copy จากตัวอย่าง order confirmation) ทดสอบด้วย console backend
แล้วตรวจสอบว่าเนื้อหาทั้งสองเวอร์ชันตรงกันในเชิงความหมาย (ไม่ใช่แค่ HTML ล้วน ๆ)

**แบบฝึกหัดที่ 4 (ขั้นสูง)**: เปิด Django shell (`python manage.py shell`) แล้วรัน
`django.db.reset_queries()` ตามด้วยการเรียก view `post_list` ผ่าน test client
(`from django.test import Client; c = Client(); c.get('/blog/')`) จากนั้นตรวจสอบ
`len(django.db.connection.queries)` ก่อนและหลังเพิ่ม `select_related`/
`prefetch_related` ตามตัวอย่างข้อ 278.2 บันทึกจำนวน query ที่ลดลงเป็นตัวเลขจริง

### 280.4 คำถามที่พบบ่อย (FAQ)

**Q: ถ้าใส่ `{% block %}` ไว้ในเทมเพลตที่ไม่มี `{% extends %}` เลย จะเกิดอะไรขึ้น?**
A: ใช้งานได้ตามปกติ — `{% block %}` ที่ไม่ได้ถูก extends จากที่ไหนจะแสดงแค่**เนื้อหา
เริ่มต้น**ของมันเองตรง ๆ เหมือนไม่มี block เลย มีประโยชน์เผื่ออนาคตถ้าอยากให้ไฟล์นี้
กลายเป็น parent ของไฟล์อื่นในภายหลัง

**Q: `{{ block.super }}` ใช้ได้กับ `{% include %}` ไหม?**
A: ไม่ได้ — `block.super` เป็นกลไกเฉพาะของ `{% extends %}`/`{% block %}` เท่านั้น
`{% include %}` ไม่มีแนวคิด "parent/child" แบบเดียวกัน มันแค่ฝังเนื้อหาไฟล์อื่นเข้ามา
ตรง ๆ ตามที่อธิบายความแตกต่างไว้ใน Part 008 ข้อ 76.1

**Q: จำเป็นต้องมี `templatetags/__init__.py` เสมอไหม ถ้าใช้ Python เวอร์ชันใหม่ที่
รองรับ namespace package?**
A: Django ยังคงต้องการให้ `templatetags/` เป็น **regular package** (มี
`__init__.py`) เสมอ แม้ Python จะรองรับ namespace package (ไม่มี `__init__.py`)
มาตั้งแต่ Python 3.3 ก็ตาม เพราะกลไกการค้นหา library ของ Django ใช้
`importlib.import_module()` แบบที่คาดหวัง regular package — ใส่ `__init__.py`
เปล่า ๆ ไว้เสมอเพื่อความชัวร์

**Q: ควรใช้ `{% cycle %}` หรือ `forloop.counter0|divisibleby:2` (filter ที่ยังไม่ได้
พูดถึงในหลักสูตรนี้) ในการทำ zebra striping ตาราง?**
A: `{% cycle %}` อ่านง่ายกว่าและตรงประเด็นกว่าสำหรับ 2 ค่า ส่วน
`divisibleby` เหมาะกับกรณีที่ต้องผสมกับเงื่อนไขอื่นใน `{% if %}` ร่วมด้วย
ทั้งสองวิธีให้ผลลัพธ์เดียวกันสำหรับ zebra striping ธรรมดา — เลือกใช้ตัวที่อ่านง่ายกว่า
ในบริบทนั้น ๆ

**Q: ทำไม Email Template ในข้อ 276 ถึงใช้ `{% spaceless %}` ครอบทั้งก้อน?**
A: เพราะ mail client บางตัว (โดยเฉพาะ Outlook desktop ที่ใช้ Word rendering engine)
ไวต่อ whitespace ระหว่าง `<table>`/`<tr>`/`<td>` มากกว่าเบราว์เซอร์ทั่วไปมาก บางครั้ง
ทำให้เกิดช่องว่างแปลก ๆ ระหว่างแถวหรือคอลัมน์ที่ไม่ได้ตั้งใจ `{% spaceless %}` ช่วยลด
ปัญหานี้ได้ในหลายกรณี แม้จะไม่ใช่ทางแก้ปัญหาการแสดงผลอีเมลที่เพี้ยนทั้งหมดก็ตาม

---

## เตรียมตัวสำหรับ Part ถัดไป

**Part 029: Custom Template Tags และ Filters** จะพาคุณกลับไปที่โฟลเดอร์
`templatetags/` ที่เพิ่งรู้จักกลไกเบื้องหลังในขั้นตอนที่ 273 ของ Part นี้ แล้วลงมือ
เขียนจริง: `simple_tag` ที่รับ argument, custom filter ด้วย `@register.filter`,
`inclusion_tag` ที่ render sub-template ของตัวเอง, เทคนิค `takes_context=True`
สำหรับเข้าถึง context ทั้งหมด, ไปจนถึงการเขียน custom `Node` class ขั้นสูงสำหรับ tag
ที่ต้องมี opening/closing tag ของตัวเอง (แบบเดียวกับ `{% block %}...{% endblock %}`
ที่คุณเพิ่งเจาะลึกไปในขั้นตอนที่ 272) พร้อมทั้งเรียนรู้ข้อควรระวังด้านความปลอดภัยเมื่อ
เขียน tag/filter เอง และวิธีเขียนเทสต์ให้ครอบคลุม

เตรียมโฟลเดอร์ `templatetags/` ที่สร้างไว้ในขั้นตอนที่ 273 ให้พร้อม แล้วไปเขียน tag
ตัวแรกของคุณกันใน Part ถัดไป!
