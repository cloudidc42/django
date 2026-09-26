# Part 054: Django กับ Alpine.js

> **ขั้นตอนที่ 531-540 ของหลักสูตร** | Phase 6: Frontend Integration
>
> เป้าหมายของ Part นี้: เข้าใจปรัชญาของ Alpine.js ในฐานะ "minimal JavaScript
> framework" ที่ไม่ต้องมี build step และเข้ากันได้ดีเยี่ยมกับ Django Template
> เรียนรู้ directive หลัก (`x-data`, `x-show`, `x-on`, `x-model`) สร้าง
> Component ที่ใช้งานจริง เช่น Dropdown, Modal, Toggle และ Live Character
> Counter เข้าใจรูปแบบ Stack ยอดนิยม "Django + HTMX + Alpine.js + Tailwind"
> ที่ชุมชนนิยมใช้แทน SPA เต็มรูปแบบ พร้อมทั้งรู้ขีดจำกัดของ Alpine.js ว่าเมื่อไหร่
> ควรก้าวไปใช้ React/Vue เต็มรูปแบบ เมื่อจบ Part นี้คุณจะสร้าง Modal ยืนยัน
> การลบโพสต์ที่ทำงานได้จริงด้วย Alpine.js ล้วน ๆ โดยไม่ต้องเขียน JavaScript
> แยกไฟล์แม้แต่บรรทัดเดียว

---

## สารบัญของ Part นี้

- ขั้นตอนที่ 531: Alpine.js คืออะไร และปรัชญา "Minimal JavaScript Framework"
- ขั้นตอนที่ 532: `x-data`, `x-show`, `x-on` Directive เบื้องต้น
- ขั้นตอนที่ 533: ผสาน Alpine.js เข้ากับ Django Template จริง (Toggle แสดง/ซ่อน)
- ขั้นตอนที่ 534: รูปแบบ Stack ยอดนิยม "Django + HTMX + Alpine.js + Tailwind"
- ขั้นตอนที่ 535: สร้าง Dropdown และ Modal Component ด้วย Alpine.js
- ขั้นตอนที่ 536: Alpine.js Store สำหรับแชร์ State ข้าม Component
- ขั้นตอนที่ 537: `x-model` Two-way Binding ผสานกับ Django Form Field
- ขั้นตอนที่ 538: ข้อควรพิจารณาด้าน Testing สำหรับหน้าที่มี Alpine.js เสริม
- ขั้นตอนที่ 539: ขีดจำกัดของ Alpine.js — เมื่อไหร่ควรย้ายไปใช้ React/Vue เต็มรูปแบบ
- ขั้นตอนที่ 540: สรุปและแบบฝึกหัด — สร้าง Modal ยืนยันการลบโพสต์

---

## ขั้นตอนที่ 531: Alpine.js คืออะไร และปรัชญา "Minimal JavaScript Framework"

### 531.1 ย้อนกลับไปดูภาพรวมที่เราเรียนมาแล้ว

ใน Part 053 คุณได้เรียนรู้ HTMX ซึ่งช่วยให้ Django ส่ง HTML fragment กลับไปอัปเดต
บางส่วนของหน้าเว็บได้โดยไม่ต้อง reload ทั้งหน้า — นี่คือรูปแบบ **"server-driven UI"**
ที่ตรรกะทั้งหมดยังอยู่ที่ฝั่งเซิร์ฟเวอร์ (Django) แต่ในโลกจริงมีบาง interaction ที่ไม่
จำเป็นต้องเดินทางไปเซิร์ฟเวอร์เลยด้วยซ้ำ เช่น:

- เปิด/ปิด dropdown menu
- เปิด/ปิด modal dialog
- toggle แสดง/ซ่อน password ในฟอร์ม
- นับจำนวนตัวอักษรที่พิมพ์ในกล่องข้อความแบบ real-time
- สลับ tab บนหน้าเดียวกัน
- แสดง loading spinner ระหว่างรอ HTMX request

Interaction เหล่านี้เป็นเรื่อง **"UI state ชั่วคราว"** (ephemeral UI state) ที่ไม่จำเป็น
ต้องรู้จักฐานข้อมูลหรือตรรกะทางธุรกิจใด ๆ เลย ถ้าจะเขียน `fetch()` ไป Django ทุกครั้ง
ที่ผู้ใช้กด hover เมนู จะทั้งช้าและสิ้นเปลืองโดยใช่เหตุ นี่คือช่องว่างที่ **Alpine.js**
เข้ามาเติมเต็มพอดี

### 531.2 Alpine.js คืออะไรกันแน่

**Alpine.js** คือ JavaScript library ขนาดเล็กมาก (~15KB gzip) ที่สร้างโดย
**Caleb Porzio** เปิดตัวปี 2019 ปรัชญาการออกแบบสรุปได้จากคำพูดที่โด่งดังของผู้สร้างเอง:

> "Alpine.js offers you the reactive and declarative nature of big frameworks
> like Vue or React at a much lower cost. You get to keep your DOM, and
> sprinkle in behavior as you see fit."
>
> (Alpine.js ให้ความสามารถ reactive และ declarative แบบ framework ใหญ่อย่าง
> Vue หรือ React ในต้นทุนที่ต่ำกว่ามาก คุณยังคงเก็บ DOM เดิมไว้ได้ และ "โรย"
> พฤติกรรมแบบ interactive ลงไปตามต้องการ)

พูดง่าย ๆ คือ Alpine.js ถูกเรียกโดยชุมชนว่าเป็น **"jQuery ของยุคใหม่"** หรือ
**"Tailwind CSS สำหรับ JavaScript"** — ไม่ได้มาแทนที่ Django Template แต่มาเสริม
ให้ HTML ธรรมดามีชีวิตชีวาขึ้น โดยเขียน directive ลงไปใน attribute ของ HTML
โดยตรง ไม่ต้องแยกไฟล์ `.js` ก็ได้ (แม้จะแยกได้ก็ตาม)

### 531.3 ปรัชญา "ไม่ต้องมี Build Step" สำคัญกับ Django อย่างไร

นี่คือจุดที่ทำให้ Alpine.js เหมาะกับ Django Developer เป็นพิเศษ Framework แบบ
React/Vue เต็มรูปแบบต้องมี **build pipeline** (Webpack, Vite, Babel, npm/yarn)
เพื่อ compile JSX/SFC ให้เป็น JavaScript ธรรมดาก่อนที่เบราว์เซอร์จะเข้าใจ ซึ่งหมายความว่า:

- ต้องรัน `npm install`, `npm run build` แยกต่างหากจากขั้นตอน deploy ของ Django
- ต้องมี Node.js บนเครื่อง dev และมักต้องมีบน CI/CD pipeline ด้วย
- ต้องคิดเรื่อง proxy ระหว่าง dev server ของ Django กับ dev server ของ frontend
- Deployment ซับซ้อนขึ้น (ต้อง build assets ก่อน collectstatic)

Alpine.js **ไม่ต้องมีสิ่งเหล่านี้เลย** เพียงแค่โหลดผ่าน `<script>` tag เดียว (จาก CDN
หรือไฟล์ static ที่ดาวน์โหลดมาเก็บเอง) แล้วเขียน directive ลงใน HTML/Django
Template ได้ทันที เหมือนที่เราเคยใช้ jQuery กันมาในอดีต — เข้ากับ **Django
Template Language (DTL)** ได้อย่างไม่มีรอยต่อ เพราะทั้งคู่ทำงานอยู่บน HTML
ตัวเดียวกัน

```
┌─────────────────────────────────────────────────────────────┐
│                     ไฟล์ template.html                      │
│                                                               │
│   {% for post in posts %}          ← Django render ตอน       │
│       <div x-data="{ open: false }">   ที่เซิร์ฟเวอร์ (SSR)   │
│           <h2>{{ post.title }}</h2>                          │
│           <button @click="open = !open">อ่านเพิ่ม</button>    │
│           <p x-show="open">{{ post.content }}</p>  ← Alpine   │
│       </div>                              ทำงานตอนที่          │
│   {% endfor %}                            เบราว์เซอร์ (CSR)   │
└─────────────────────────────────────────────────────────────┘
```

สังเกตว่า `{{ post.title }}` (Django) และ `x-data`, `@click`, `x-show` (Alpine)
อยู่ในไฟล์เดียวกันได้อย่างสงบสุข เพราะ Django render ที่ฝั่งเซิร์ฟเวอร์ **ก่อน**
ส่ง HTML ไปให้เบราว์เซอร์ ส่วน Alpine ทำงาน **หลังจาก** เบราว์เซอร์ได้รับ HTML
แล้วเท่านั้น ไม่มีการชนกันของ syntax แบบที่ต้องระวังใน Vue (ซึ่งใช้ `{{ }}`
เหมือนกันจนต้องมี escape พิเศษ)

### 531.4 ติดตั้ง Alpine.js ในโปรเจกต์ Django

วิธีที่ง่ายที่สุดคือโหลดผ่าน CDN ในไฟล์ `base.html`:

```html
<!-- templates/base.html -->
<!DOCTYPE html>
<html lang="th">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>{% block title %}My Django Site{% endblock %}</title>
    {% load static %}
    <link rel="stylesheet" href="{% static 'css/main.css' %}">

    <!-- Alpine.js Plugins (โหลดก่อนตัว core เสมอ) -->
    <script defer src="https://cdn.jsdelivr.net/npm/@alpinejs/persist@3.x.x/dist/cdn.min.js"></script>
    <script defer src="https://cdn.jsdelivr.net/npm/@alpinejs/focus@3.x.x/dist/cdn.min.js"></script>

    <!-- Alpine.js Core (โหลดตัวนี้เป็นตัวสุดท้ายเสมอ) -->
    <script defer src="https://cdn.jsdelivr.net/npm/alpinejs@3.x.x/dist/cdn.min.js"></script>
</head>
<body>
    {% block content %}{% endblock %}
</body>
</html>
```

**ข้อควรระวังสำคัญ**: attribute `defer` จำเป็นมาก เพราะต้องให้ browser parse
HTML ทั้งหมดก่อน แล้วค่อยรัน Alpine.js เพื่อให้ Alpine หา element ที่มี `x-data`
เจอครบทุกตัว และถ้าใช้ปลั๊กอิน (เช่น `persist`, `focus`, `collapse`) ต้องโหลด
**ก่อน** ไฟล์ core เสมอ มิฉะนั้นปลั๊กอินจะทำงานไม่ได้

สำหรับ production ที่ไม่ต้องการพึ่งพา CDN ภายนอก (control ทุกอย่างเอง หรือ
ต้องการทำงานแบบ offline-first) ให้ดาวน์โหลดไฟล์มาเก็บเป็น static file แทน:

```bash
mkdir -p static/vendor/alpinejs
curl -o static/vendor/alpinejs/alpine.min.js \
    https://cdn.jsdelivr.net/npm/alpinejs@3.14.9/dist/cdn.min.js
```

```html
{% load static %}
<script defer src="{% static 'vendor/alpinejs/alpine.min.js' %}"></script>
```

### 531.5 เปรียบเทียบ Alpine.js กับเครื่องมือ Frontend อื่น ๆ ที่เรียนมา/จะเรียน

| คุณสมบัติ | Alpine.js | HTMX (Part 053) | React (Part 055) | Vue.js (Part 056) |
|---|---|---|---|---|
| ขนาดไฟล์ (gzip) | ~15KB | ~14KB | ~45KB (+ ReactDOM) | ~34KB |
| ต้องมี Build Step | ❌ ไม่ต้อง | ❌ ไม่ต้อง | ✅ ต้องมี (Webpack/Vite) | ⚠️ แนะนำให้มี (SFC) |
| แก้ปัญหาอะไร | UI state ฝั่ง client (toggle, modal) | การสื่อสารกับ server แบบไม่ reload หน้า | สร้าง SPA เต็มรูปแบบ | สร้าง SPA เต็มรูปแบบ |
| ทำงานร่วมกับ Django Template | ดีเยี่ยม (attribute-based) | ดีเยี่ยม (attribute-based) | ต้องแยก API (DRF) | ต้องแยก API หรือฝัง SFC |
| Learning Curve | ต่ำมาก (คล้าย Vue ย่อส่วน) | ต่ำ | สูง | ปานกลาง-สูง |
| เหมาะกับ | โรย interactivity เล็ก ๆ น้อย ๆ | เปลี่ยนส่วนของหน้าโดยให้ server render | แอปที่ UI ซับซ้อนมาก, มี state เยอะ | แอปขนาดกลาง-ใหญ่ |
| Component Reusability | จำกัด (ไม่มี built-in component system เต็มรูปแบบ) | ไม่มี concept component | ดีเยี่ยม | ดีเยี่ยม |
| State Management ข้ามหน้า | `Alpine.store` (เบา) | ไม่มี (เป็น server state) | Redux/Zustand/Context | Pinia/Vuex |

**ข้อสรุป**: Alpine.js ไม่ได้แข่งกับ HTMX แต่ **ทำงานคู่กัน** — HTMX จัดการการ
สื่อสารกับ Django (fetch data, submit form, swap HTML) ส่วน Alpine.js จัดการ
พฤติกรรมเล็ก ๆ น้อย ๆ ที่ไม่จำเป็นต้องคุยกับ server เลย เราจะเห็นภาพชัดเจนขึ้นใน
ขั้นตอนที่ 534

### 531.6 ทำไม Community ถึงชอบ Alpine.js คู่กับ Django

1. **Progressive Enhancement โดยธรรมชาติ**: หน้าเว็บที่ Django render ออกมา
   ยังคงเป็น HTML ที่สมบูรณ์แม้ JavaScript จะโหลดไม่ทัน (ต่างจาก SPA ที่ถ้า JS
   พัง หน้าจะว่างเปล่าทันที)
2. **ไม่ทำลาย mental model ของ Django Developer**: ไม่ต้องเรียนรู้ JSX,
   Virtual DOM, component lifecycle ที่ซับซ้อน แค่รู้ HTML + directive ไม่กี่ตัว
3. **Deploy ง่ายเหมือนเดิมทุกประการ**: `python manage.py collectstatic` ครั้ง
   เดียวจบ ไม่ต้องมี pipeline แยก
4. **SEO friendly**: เนื้อหาหลักอยู่ใน HTML ที่ Django render ตั้งแต่ต้น ไม่ต้อง
   พึ่ง JavaScript rendering เหมือน SPA (ซึ่งบางครั้งมีปัญหากับ search engine
   crawler)
5. **เหมาะกับทีมขนาดเล็กถึงกลาง**: ทีมที่ไม่มี frontend specialist แยกต่างหาก
   สามารถให้ Django Developer คนเดียวดูแลทั้ง backend และ UI sprinkle ได้

---

## ขั้นตอนที่ 532: `x-data`, `x-show`, `x-on` Directive เบื้องต้น

### 532.1 แนวคิดหลัก: HTML คือ "Component"

Alpine.js ไม่มีไฟล์ `.vue` หรือ `.jsx` แยกต่างหาก — **HTML element ที่มี
`x-data` คือ component** ทุกอย่างที่อยู่ภายใน element นั้น (รวมลูกหลาน) จะ
"มองเห็น" ข้อมูลใน `x-data` ได้

```html
<div x-data="{ count: 0 }">
    <!-- ทุกอย่างในนี้เข้าถึง count ได้ -->
    <span x-text="count"></span>
    <button @click="count++">เพิ่ม</button>
</div>
```

### 532.2 ตาราง Directive หลักที่ต้องรู้จักก่อน

| Directive | หน้าที่ | ตัวอย่าง |
|---|---|---|
| `x-data` | ประกาศ scope และ state เริ่มต้นของ component | `x-data="{ open: false }"` |
| `x-init` | รันโค้ดทันทีตอน component ถูกสร้าง | `x-init="console.log('ready')"` |
| `x-show` | แสดง/ซ่อน element ด้วย `display: none` (element ยังอยู่ใน DOM) | `x-show="open"` |
| `x-if` | ใส่/เอา element ออกจาก DOM จริง ๆ (ต้องใช้กับ `<template>`) | `<template x-if="open">` |
| `x-text` | กำหนดข้อความภายใน element จาก expression | `x-text="count"` |
| `x-html` | กำหนด HTML ภายใน element (ระวัง XSS!) | `x-html="htmlContent"` |
| `x-bind` (`:`) | ผูกค่า attribute แบบ dynamic | `:class="{ 'active': open }"` |
| `x-on` (`@`) | ผูก event listener | `@click="open = !open"` |
| `x-model` | Two-way binding กับ form input | `x-model="search"` |
| `x-for` | วนลูปสร้าง element (ต้องใช้กับ `<template>`) | `<template x-for="item in items">` |
| `x-transition` | ใส่ animation ตอนแสดง/ซ่อน | `x-transition` |
| `x-cloak` | ซ่อน element จนกว่า Alpine จะโหลดเสร็จ (กัน flash of unstyled content) | `x-cloak` |
| `x-ref` / `$refs` | อ้างอิง element โดยตรง (คล้าย `document.getElementById`) | `x-ref="input"` |
| `$el` | อ้างอิง element ปัจจุบัน | `console.log($el)` |
| `$watch` | เฝ้าดูการเปลี่ยนแปลงของตัวแปร | `x-init="$watch('open', v => ...)"` |

### 532.3 `x-data`: หัวใจของทุก Component

`x-data` รับ JavaScript object literal เป็นค่าเริ่มต้นของ state สามารถใส่ทั้ง
ตัวแปรและฟังก์ชัน (method) ได้ในตัวเดียว:

```html
<div x-data="{
    count: 0,
    step: 1,
    increment() {
        this.count += this.step;
    },
    decrement() {
        this.count -= this.step;
    }
}">
    <button @click="decrement()">-</button>
    <span x-text="count"></span>
    <button @click="increment()">+</button>
</div>
```

Scope ของ `x-data` จะครอบคลุมทุก element ลูกหลาน แต่จะไม่ "รั่ว" ออกไปนอก
element ที่ประกาศไว้ — ถ้ามี `x-data` ซ้อนกัน (nested) ตัวลูกจะเข้าถึงตัวแม่ได้
แต่ตัวแม่เข้าถึงตัวลูกไม่ได้ (เหมือน scope ปกติของ JavaScript)

### 532.4 `x-show` vs `x-if`: ความแตกต่างที่สำคัญ

```html
<!-- x-show: element ยังอยู่ใน DOM เสมอ แค่ซ่อนด้วย CSS (display: none) -->
<div x-data="{ open: false }">
    <button @click="open = !open">Toggle</button>
    <p x-show="open">ข้อความนี้ถูกซ่อนด้วย CSS ไม่ใช่ถูกลบออกจาก DOM</p>
</div>

<!-- x-if: element ถูกสร้าง/ทำลายจริงใน DOM ต้องใช้คู่กับ <template> -->
<div x-data="{ open: false }">
    <button @click="open = !open">Toggle</button>
    <template x-if="open">
        <p>ข้อความนี้ถูกสร้างขึ้นใหม่ทุกครั้งที่ open เป็น true</p>
    </template>
</div>
```

**เลือกใช้เมื่อไหร่**:

| สถานการณ์ | ควรใช้ |
|---|---|
| Toggle บ่อย ๆ (dropdown, accordion) | `x-show` (เร็วกว่า เพราะไม่ต้องสร้าง/ทำลาย DOM ใหม่) |
| Element มี cost สูงตอนสร้าง (เช่น มี `<video>`, third-party widget) | `x-if` |
| ต้องการให้ element หายไปจาก DOM จริง ๆ (เช่น สำหรับ screen reader) | `x-if` |
| ต้องการ animation ตอนแสดง/ซ่อน | `x-show` (ใช้คู่กับ `x-transition` ได้ลื่นกว่า) |

### 532.5 `x-on` / `@`: จัดการ Event

`x-on:click` สามารถย่อเป็น `@click` ได้ (นิยมใช้แบบย่อ) รองรับ **modifier**
คล้าย Vue.js:

```html
<div x-data="{ message: '' }">
    <!-- .prevent = เรียก event.preventDefault() อัตโนมัติ -->
    <form @submit.prevent="alert('ส่งฟอร์มแล้ว: ' + message)">
        <input type="text" x-model="message">
        <button type="submit">ส่ง</button>
    </form>

    <!-- .stop = เรียก event.stopPropagation() -->
    <div @click="console.log('คลิกที่ div นอก')">
        <button @click.stop="console.log('คลิกปุ่มใน ไม่ลามไป div นอก')">
            คลิกที่นี่
        </button>
    </div>

    <!-- .outside = ทำงานเมื่อคลิก "นอก" element นี้ (มีประโยชน์มากกับ dropdown) -->
    <div x-data="{ open: true }" x-show="open" @click.outside="open = false">
        คลิกข้างนอกกล่องนี้เพื่อปิด
    </div>

    <!-- .window = ฟัง event จาก window object เช่น กด Escape ที่ไหนก็ได้ -->
    <div x-data="{ open: true }" x-show="open" @keydown.escape.window="open = false">
        กด Esc ปิดได้จากทุกที่ในหน้า
    </div>

    <!-- .debounce = หน่วงเวลาก่อนทำงาน มีประโยชน์กับ search input -->
    <input type="text" @input.debounce.500ms="console.log('ค้นหา...')">
</div>
```

### 532.6 ตัวอย่างสมบูรณ์: Counter และ Toggle แบบไม่พึ่ง Django เลย

ก่อนจะผสานกับ Django ในขั้นตอนถัดไป มาลองทดสอบไฟล์ HTML ล้วน ๆ เพื่อให้เห็น
ว่า Alpine.js ทำงานอย่างไรโดยไม่ต้องรัน Django server เลยด้วยซ้ำ:

```html
<!-- ทดสอบได้ทันทีโดยเซฟเป็น .html แล้วเปิดในเบราว์เซอร์ -->
<!DOCTYPE html>
<html lang="th">
<head>
    <meta charset="UTF-8">
    <title>ทดสอบ Alpine.js</title>
    <script defer src="https://cdn.jsdelivr.net/npm/alpinejs@3.x.x/dist/cdn.min.js"></script>
</head>
<body>
    <div x-data="{ count: 0, open: false }">
        <h2>Counter: <span x-text="count"></span></h2>
        <button @click="count++">+1</button>
        <button @click="count--">-1</button>
        <button @click="count = 0">Reset</button>

        <hr>

        <button @click="open = !open" x-text="open ? 'ซ่อน' : 'แสดง'"></button>
        <p x-show="open" x-transition>สวัสดี! นี่คือข้อความที่ถูกซ่อน/แสดงด้วย Alpine.js</p>
    </div>
</body>
</html>
```

ลองเปิดไฟล์นี้ในเบราว์เซอร์โดยไม่ต้องมี Django server เลย คุณจะเห็นว่า
Alpine.js ทำงานได้ทันที นี่คือข้อดีของการ "ไม่ต้องมี build step" ที่เราพูดถึง
ในขั้นตอนที่แล้ว

---

## ขั้นตอนที่ 533: ผสาน Alpine.js เข้ากับ Django Template จริง (Toggle แสดง/ซ่อน)

### 533.1 เตรียมแอป Django สำหรับตัวอย่างตลอด Part นี้

เราจะใช้แอป `blog` เป็นตัวอย่างหลักตลอด Part นี้ สมมติว่ามี Model และ View
พื้นฐานอยู่แล้ว (ทบทวนจาก Phase 2-3):

```python
# blog/models.py
from django.conf import settings
from django.db import models
from django.urls import reverse


class Post(models.Model):
    title = models.CharField(max_length=200)
    slug = models.SlugField(max_length=220, unique=True)
    excerpt = models.CharField(max_length=300)
    content = models.TextField()
    author = models.ForeignKey(
        settings.AUTH_USER_MODEL, on_delete=models.CASCADE, related_name="posts"
    )
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)

    class Meta:
        ordering = ["-created_at"]

    def __str__(self):
        return self.title

    def get_absolute_url(self):
        return reverse("blog:post_detail", kwargs={"slug": self.slug})
```

```python
# blog/views.py
from django.contrib.auth.decorators import login_required
from django.shortcuts import get_object_or_404, redirect, render
from django.views.decorators.http import require_POST

from .models import Post


def post_list(request):
    posts = Post.objects.select_related("author").all()
    return render(request, "blog/post_list.html", {"posts": posts})


def post_detail(request, slug):
    post = get_object_or_404(Post, slug=slug)
    return render(request, "blog/post_detail.html", {"post": post})
```

```python
# blog/urls.py
from django.urls import path

from . import views

app_name = "blog"

urlpatterns = [
    path("", views.post_list, name="post_list"),
    path("<slug:slug>/", views.post_detail, name="post_detail"),
]
```

### 533.2 ตัวอย่างแรก: Toggle แสดง/ซ่อนเนื้อหาแบบ "อ่านเพิ่มเติม"

หน้าลิสต์โพสต์แบบเดิม (ไม่มี Alpine.js) จะแสดงแค่ `excerpt` และต้องคลิกลิงก์
ไปหน้ารายละเอียดถึงจะเห็นเนื้อหาเต็ม แต่ด้วย Alpine.js เราสามารถให้ผู้ใช้กด
"อ่านเพิ่มเติม" แล้วขยายเนื้อหาในหน้าเดียวกันได้ทันที **โดยไม่ต้องมี round-trip
ไปเซิร์ฟเวอร์เลย** (ต่างจาก HTMX ที่ต้อง fetch ข้อมูลจากเซิร์ฟเวอร์):

```html
<!-- templates/blog/post_list.html -->
{% extends "base.html" %}

{% block content %}
<div class="post-list">
    <h1>บทความทั้งหมด</h1>

    {% for post in posts %}
        <article
            x-data="{ expanded: false }"
            class="post-card"
        >
            <h2>{{ post.title }}</h2>
            <p class="meta">
                โดย {{ post.author.get_full_name|default:post.author.username }}
                · {{ post.created_at|date:"d M Y" }}
            </p>

            <!-- แสดง excerpt เสมอ เมื่อยังไม่ expand -->
            <p x-show="!expanded">{{ post.excerpt }}</p>

            <!-- แสดงเนื้อหาเต็มเมื่อ expand เท่านั้น -->
            <div x-show="expanded" x-transition>
                {{ post.content|linebreaks }}
            </div>

            <button
                @click="expanded = !expanded"
                x-text="expanded ? 'ย่อกลับ ▲' : 'อ่านเพิ่มเติม ▼'"
                class="btn-link"
            ></button>
        </article>
    {% empty %}
        <p>ยังไม่มีบทความ</p>
    {% endfor %}
</div>
{% endblock %}
```

**จุดสำคัญที่ต้องเข้าใจ**: `{{ post.content|linebreaks }}` ถูก Django render
เป็น HTML จริง ๆ ตั้งแต่ตอนที่เซิร์ฟเวอร์ตอบกลับมา (View Source จะเห็นเนื้อหา
เต็มอยู่ใน HTML แม้จะยังไม่ได้กด "อ่านเพิ่มเติม") — Alpine.js แค่ **ซ่อน**
ด้วย CSS (`display: none`) เท่านั้น เนื้อหาไม่ได้ถูกโหลดทีหลัง! นี่คือข้อดีเรื่อง
SEO ที่พูดถึงในขั้นตอนที่ 531: search engine crawler เห็นเนื้อหาเต็มเสมอ
ไม่ว่า Alpine.js จะทำงานหรือไม่

### 533.3 ป้องกัน "Flash of Unstyled Content" ด้วย `x-cloak`

ปัญหาที่พบบ่อย: ก่อนที่ Alpine.js จะโหลดเสร็จ (ใช้เวลาเป็นมิลลิวินาที) เบราว์เซอร์
อาจแสดง element ที่ควรถูกซ่อนไว้ก่อนเป็นเสี้ยววินาที ทำให้เกิดการ "กระพริบ"
วิธีแก้คือใช้ `x-cloak`:

```html
<!-- ต้องเพิ่ม CSS นี้ใน base.html เสมอเมื่อใช้ x-cloak -->
<style>
    [x-cloak] { display: none !important; }
</style>
```

```html
<div x-data="{ expanded: false }">
    <div x-show="expanded" x-cloak x-transition>
        เนื้อหาที่จะไม่กระพริบให้เห็นตอนโหลดหน้าครั้งแรก
    </div>
</div>
```

`x-cloak` ทำงานคู่กับ CSS ด้านบน: ก่อน Alpine โหลดเสร็จ attribute `x-cloak`
ยังอยู่ครบ ทำให้ CSS ซ่อน element ไว้ก่อน พอ Alpine โหลดเสร็จ มันจะลบ
attribute `x-cloak` ออกจาก DOM โดยอัตโนมัติ แล้วให้ `x-show` เข้าควบคุมต่อ

### 533.4 ตัวอย่างที่สอง: Accordion แบบ FAQ (หลาย Panel ควบคุมแยกกัน)

```html
<!-- templates/blog/faq.html -->
{% extends "base.html" %}

{% block content %}
<div class="faq-section">
    <h1>คำถามที่พบบ่อย</h1>

    {% for faq in faqs %}
        <div x-data="{ open: false }" class="faq-item" x-cloak>
            <button
                @click="open = !open"
                class="faq-question"
                :aria-expanded="open"
            >
                {{ faq.question }}
                <span x-text="open ? '−' : '+'" class="faq-icon"></span>
            </button>
            <div x-show="open" x-transition.duration.200ms class="faq-answer">
                {{ faq.answer }}
            </div>
        </div>
    {% endfor %}
</div>
{% endblock %}
```

สังเกตว่าแต่ละ `<div x-data="{ open: false }">` มี scope ของตัวเอง — การเปิด
FAQ ข้อหนึ่งจะไม่กระทบข้ออื่น เพราะ Alpine.js สร้าง **instance ใหม่**
ทุกครั้งที่เจอ `x-data` (คนละ context กับตัวแปร JavaScript แบบ global)

### 533.5 การผสาน Alpine.js State กับข้อมูลเริ่มต้นจาก Django (`x-data` แบบ Dynamic)

บางครั้งค่าเริ่มต้นของ Alpine state ต้องมาจากข้อมูลที่ Django ส่งมา ไม่ใช่ค่า
คงที่ ทำได้โดยฝัง Django template variable ลงใน `x-data` string โดยตรง:

```html
<!-- สมมติว่า post.is_featured เป็น boolean จาก Django -->
<div x-data="{ bookmarked: {{ post.is_bookmarked_by_user|yesno:'true,false' }} }">
    <button
        @click="bookmarked = !bookmarked"
        :class="bookmarked ? 'btn-bookmarked' : 'btn-default'"
        x-text="bookmarked ? '★ บันทึกแล้ว' : '☆ บันทึก'"
    ></button>
</div>
```

`|yesno:'true,false'` เป็น Django template filter ที่แปลงค่า Python boolean
ให้เป็น string `"true"` หรือ `"false"` ซึ่งเป็น syntax ที่ JavaScript เข้าใจได้
พอดี — เทคนิคนี้สำคัญมากเพราะการพิมพ์ `{{ value }}` ตรง ๆ ถ้า `value` เป็น
Python `True`/`False` จะกลาย เป็นตัวอักษร `True`/`False` (ขึ้นต้นด้วยตัวใหญ่)
ซึ่ง **ไม่ใช่** JavaScript boolean ที่ถูกต้อง (จะทำให้เกิด `ReferenceError:
True is not defined`)

**คำเตือนด้านความปลอดภัย**: ถ้าค่าที่ฝังลงไปเป็น string ที่มาจากผู้ใช้ (user
input) ต้องระวังเรื่อง XSS เสมอ ใช้ `{{ value|escapejs }}` เมื่อฝังค่า string
ลงใน JavaScript context:

```html
<div x-data="{ username: '{{ request.user.username|escapejs }}' }">
    <p>สวัสดีคุณ <span x-text="username"></span></p>
</div>
```

### 533.6 เปรียบเทียบ 3 วิธีในการ Toggle: CSS-only, jQuery แบบเดิม, Alpine.js

| วิธี | โค้ดที่ต้องเขียน | ข้อดี | ข้อเสีย |
|---|---|---|---|
| CSS `:checked` + `<input type="checkbox" hidden>` | HTML + CSS ล้วน | ไม่ต้องมี JS เลย | จำกัดมาก ทำ logic ซับซ้อนไม่ได้ |
| jQuery (`$('.btn').click(...)`) | ต้องเขียน JS แยกไฟล์ ผูก selector เอง | คุ้นเคยกันมานาน | ต้องจัดการ DOM selector เอง เสี่ยง memory leak, ผูก id ซ้ำเมื่อมีหลาย element |
| Alpine.js (`x-data`, `x-show`) | Directive ใน HTML โดยตรง | Declarative, scope ชัดเจนต่อ element, ไม่ต้องผูก selector | ต้องเรียนรู้ syntax ใหม่ (แม้จะเรียนเร็ว) |

---

## ขั้นตอนที่ 534: รูปแบบ Stack ยอดนิยม "Django + HTMX + Alpine.js + Tailwind"

### 534.1 ทำไมชุมชนถึงเรียก Stack นี้ว่า "The Modern Monolith"

ในช่วงปี 2022-2026 เกิดกระแสที่เรียกว่า **"HTML-over-the-wire"** หรือ
**"Hypermedia-driven Applications"** ซึ่งเป็นแนวคิดที่ท้าทายความเชื่อเดิมที่ว่า
"เว็บสมัยใหม่ต้องเป็น SPA เท่านั้น" ผู้เสนอแนวคิดนี้ที่มีชื่อเสียงคือ Carson Gross
(ผู้สร้าง HTMX) ที่เขียนไว้ว่าการรวม 4 เครื่องมือนี้เข้าด้วยกันทำให้ทีมเล็ก ๆ
สร้างเว็บแอปที่ interactive เทียบเท่า SPA ได้โดยไม่ต้องแบก complexity ของ
JavaScript build tooling — เรียกกันในชุมชนว่า **"The Modern Monolith"**
หรือบางครั้งเรียกสั้น ๆ ว่า **"HTMAD" (HTMX + Alpine + Django)**

### 534.2 แบ่งหน้าที่ให้ชัดเจน: ใครทำอะไร

| เครื่องมือ | ทำหน้าที่อะไร | ตัวอย่างงาน |
|---|---|---|
| **Django** | Business logic, database, authentication, render HTML เริ่มต้นและ HTML fragment | Query ข้อมูล, validate ฟอร์ม, ตรวจสอบสิทธิ์ |
| **HTMX** | สื่อสารกับ Django แบบ AJAX แต่ยังคง "คิดแบบ HTML" (ไม่ใช่ JSON) | กด "โหลดเพิ่ม", submit form โดยไม่ reload, infinite scroll |
| **Alpine.js** | UI state ฝั่ง client ที่ไม่ต้องคุยกับ server | เปิด/ปิด modal, dropdown, tab, loading spinner, animation |
| **Tailwind CSS** | Styling แบบ utility-first ไม่ต้องเขียน custom CSS เยอะ | จัดวาง layout, สี, spacing, responsive design |

```
┌────────────────────────────────────────────────────────────────┐
│                         Browser (Client)                       │
│                                                                  │
│  ┌────────────────────────────────────────────────────────┐   │
│  │  HTML ที่ Django ส่งมา (ตกแต่งด้วย Tailwind class)         │   │
│  │                                                            │   │
│  │  ┌──────────────┐        ┌──────────────────────────┐   │   │
│  │  │  Alpine.js    │        │  HTMX                     │   │   │
│  │  │  จัดการ:      │        │  จัดการ:                  │   │   │
│  │  │  - Modal       │        │  - hx-get / hx-post       │   │   │
│  │  │  - Dropdown    │        │  - hx-trigger             │   │   │
│  │  │  - Tab         │        │  - hx-swap                │   │   │
│  │  │  - Local state │        │  - ไปคุยกับ Django          │   │   │
│  │  └──────────────┘        └────────────┬─────────────┘   │   │
│  └───────────────────────────────────────┼─────────────────┘   │
└──────────────────────────────────────────┼─────────────────────┘
                                            │ HTTP (fragment ไป-กลับ)
                                            ▼
                              ┌───────────────────────────┐
                              │      Django (Server)       │
                              │  Views + Models + Templates│
                              └───────────────────────────┘
```

### 534.3 ตัวอย่างจริง: HTMX กับ Alpine.js ทำงานร่วมกันในฟีเจอร์เดียว

สถานการณ์: ปุ่ม "กดถูกใจ" (Like) ที่ต้องอัปเดตจำนวนใน database จริง (ใช้ HTMX)
แต่ต้องการ animation "หัวใจเด้ง" ตอนกด (ใช้ Alpine.js) — งานสองอย่างนี้แยก
ความรับผิดชอบกันชัดเจน:

```html
<!-- templates/blog/_like_button.html -->
<div
    x-data="{ liked: {{ post.is_liked_by_user|yesno:'true,false' }}, animate: false }"
    class="like-widget"
>
    <button
        hx-post="{% url 'blog:toggle_like' post.slug %}"
        hx-swap="none"
        hx-headers='{"X-CSRFToken": "{{ csrf_token }}"}'
        @click="
            liked = !liked;
            animate = true;
            setTimeout(() => animate = false, 300)
        "
        @htmx:response-error="liked = !liked"
        :class="{ 'is-liked': liked, 'is-animating': animate }"
        class="like-button"
    >
        <span x-text="liked ? '❤️' : '🤍'"></span>
        <span x-text="liked ? 'ถูกใจแล้ว' : 'ถูกใจ'"></span>
    </button>
</div>
```

```python
# blog/views.py (เพิ่มเติม)
from django.http import HttpResponse
from django.views.decorators.http import require_POST


@login_required
@require_POST
def toggle_like(request, slug):
    post = get_object_or_404(Post, slug=slug)
    like, created = post.likes.get_or_create(user=request.user)
    if not created:
        like.delete()
    return HttpResponse(status=204)  # ไม่ต้องส่ง HTML กลับ เพราะ Alpine อัปเดต UI ไปแล้ว
```

จุดที่น่าสนใจคือ:

1. **Alpine.js อัปเดต UI ทันที** (`liked = !liked`) แบบ **optimistic update**
   โดยไม่รอ server ตอบกลับก่อน ทำให้รู้สึกเร็วมาก (instant feedback)
2. **HTMX ส่ง request จริงไป Django เบื้องหลัง** เพื่อบันทึกลงฐานข้อมูลจริง
3. ถ้า request ล้มเหลว (`@htmx:response-error`) Alpine.js จะ **rollback**
   สถานะกลับเป็นเหมือนเดิม — pattern นี้เรียกว่า **"optimistic UI with
   rollback"** ซึ่งเป็นรูปแบบมาตรฐานในแอประดับมืออาชีพ
4. `hx-swap="none"` บอก HTMX ว่าไม่ต้องเอา response มา swap ที่ไหนเลย เพราะ
   Alpine.js จัดการ UI เองหมดแล้ว

### 534.4 เปรียบเทียบกับ Stack แบบ SPA (React + DRF)

| ประเด็น | Django + HTMX + Alpine.js + Tailwind | React (SPA) + Django REST Framework |
|---|---|---|
| จำนวน repository | 1 (Django project เดียว) | มักแยก 2 (backend + frontend) |
| Build tooling | ไม่มี หรือมีแค่ Tailwind CLI เบา ๆ | ต้องมี Webpack/Vite เต็มรูปแบบ |
| การจัดการ state | ส่วนใหญ่อยู่ที่ server (Django), Alpine จัดการ ephemeral state เล็ก ๆ | ต้องจัดการ state ฝั่ง client อย่างเป็นระบบ (Redux/Zustand) |
| ทีมที่เหมาะสม | ทีมเล็ก-กลาง, Django Developer ทำ full-stack คนเดียวได้ | ทีมใหญ่ที่แยก backend/frontend developer ชัดเจน |
| Time-to-market | เร็วกว่าสำหรับ CRUD app ทั่วไป | ช้ากว่าในช่วงแรก แต่ scale ทีมได้ดีกว่าในระยะยาว |
| Mobile app ในอนาคต | ต้องเพิ่ม DRF API แยกทีหลัง | มี API (DRF) พร้อมอยู่แล้ว ใช้ต่อกับ mobile ได้ทันที |
| SEO | ดีมาก (SSR โดยธรรมชาติ) | ต้องทำ SSR/SSG เพิ่มเติม (Next.js) |
| ตัวอย่างบริษัทที่ใช้จริง | GitHub (บางส่วน), Basecamp (HEY), Deckstack | Instagram, Netflix, Facebook |

**ข้อคิดสำคัญสำหรับมืออาชีพ**: ไม่มี stack ไหนดีที่สุดในทุกสถานการณ์ การเลือก
ต้องดูจาก **ขนาดทีม, ความซับซ้อนของ UI, และความต้องการในอนาคต** (เช่น
ถ้าต้องมี mobile app ด้วย การมี DRF API ตั้งแต่ต้นอาจคุ้มค่ากว่า) เราจะเจาะลึก
การตัดสินใจนี้อีกครั้งในขั้นตอนที่ 539 และ Part 055-056

### 534.5 ติดตั้ง Tailwind CSS คู่กับ Django แบบย่อ (สำหรับให้ตัวอย่างสมบูรณ์)

```bash
# ติดตั้ง Tailwind CLI แบบ standalone (ไม่ต้องมี Node.js เลยด้วยซ้ำ!)
curl -sLO https://github.com/tailwindlabs/tailwindcss/releases/latest/download/tailwindcss-linux-x64
chmod +x tailwindcss-linux-x64
mv tailwindcss-linux-x64 tailwindcss
```

```bash
# สร้างไฟล์ input CSS
mkdir -p static/css
cat > static/css/input.css << 'EOF'
@import "tailwindcss";
EOF

# build เป็น CSS จริงที่ Django จะ serve (รันครั้งเดียวหรือใช้ --watch ตอน dev)
./tailwindcss -i ./static/css/input.css -o ./static/css/main.css --watch
```

จุดสำคัญคือ Tailwind CLI แบบ standalone binary **ไม่ต้องมี Node.js/npm เลย**
ซึ่งตอกย้ำปรัชญาเดียวกับ Alpine.js: ลดความซับซ้อนของ build tooling ให้เหลือ
น้อยที่สุดเท่าที่จะทำได้ในระบบนิเวศของ Django

---

## ขั้นตอนที่ 535: สร้าง Dropdown และ Modal Component ด้วย Alpine.js

### 535.1 Dropdown Menu แบบใช้งานจริง (User Account Menu)

```html
<!-- templates/partials/_navbar.html -->
<nav class="navbar">
    <a href="{% url 'blog:post_list' %}" class="navbar-brand">MyBlog</a>

    {% if request.user.is_authenticated %}
        <div x-data="{ open: false }" @click.outside="open = false" class="dropdown" x-cloak>
            <button
                @click="open = !open"
                :aria-expanded="open"
                class="dropdown-trigger"
            >
                {{ request.user.username }}
                <svg :class="{ 'rotate-180': open }" class="dropdown-arrow" width="12" height="12">
                    <path d="M2 4l4 4 4-4" stroke="currentColor" fill="none"/>
                </svg>
            </button>

            <div
                x-show="open"
                x-transition:enter="transition ease-out duration-150"
                x-transition:enter-start="opacity-0 scale-95"
                x-transition:enter-end="opacity-100 scale-100"
                x-transition:leave="transition ease-in duration-100"
                x-transition:leave-start="opacity-100 scale-100"
                x-transition:leave-end="opacity-0 scale-95"
                class="dropdown-menu"
                @keydown.escape.window="open = false"
            >
                <a href="{% url 'accounts:profile' %}" class="dropdown-item">โปรไฟล์ของฉัน</a>
                <a href="{% url 'blog:my_posts' %}" class="dropdown-item">บทความของฉัน</a>
                <hr>
                <form method="post" action="{% url 'accounts:logout' %}">
                    {% csrf_token %}
                    <button type="submit" class="dropdown-item dropdown-item-danger">
                        ออกจากระบบ
                    </button>
                </form>
            </div>
        </div>
    {% else %}
        <a href="{% url 'accounts:login' %}" class="btn-login">เข้าสู่ระบบ</a>
    {% endif %}
</nav>
```

จุดสำคัญของ Dropdown ที่ต้องมีครบเสมอในระดับมืออาชีพ:

1. **`@click.outside="open = false"`** — ปิดเมนูเมื่อคลิกนอกกล่อง (UX มาตรฐาน)
2. **`@keydown.escape.window="open = false"`** — ปิดเมนูเมื่อกด Esc (accessibility)
3. **`:aria-expanded="open"`** — บอก screen reader ว่าเมนูเปิดอยู่หรือไม่
4. **`x-transition`** พร้อม enter/leave states แยกกัน — ทำให้ animation ลื่นไหล
   ทั้งตอนเปิดและปิด ไม่ใช่แค่ปรากฏ/หายวับ

### 535.2 Modal Dialog แบบสมบูรณ์ (Reusable Component)

Modal เป็น component ที่ซับซ้อนกว่า dropdown เล็กน้อย เพราะต้องจัดการเรื่อง
**focus trap**, **scroll lock**, และ **backdrop click** ด้วย:

```html
<!-- templates/partials/_modal.html — ใช้เป็น "template" ที่ include ซ้ำได้ -->
{% comment %}
ใช้งานโดย include ไฟล์นี้แล้วครอบด้วย x-data ที่มี modalOpen: false
ตัวอย่าง: {% include "partials/_modal.html" with modal_title="ยืนยันการลบ" %}
{% endcomment %}

<template x-teleport="body">
    <div
        x-show="modalOpen"
        x-cloak
        class="modal-backdrop"
        @keydown.escape.window="modalOpen = false"
    >
        <div
            x-show="modalOpen"
            x-transition:enter="transition ease-out duration-200"
            x-transition:enter-start="opacity-0 scale-90"
            x-transition:enter-end="opacity-100 scale-100"
            x-trap="modalOpen"
            @click.outside="modalOpen = false"
            class="modal-box"
            role="dialog"
            aria-modal="true"
            :aria-labelledby="'modal-title'"
        >
            <div class="modal-header">
                <h3 id="modal-title">{{ modal_title|default:"หัวข้อ" }}</h3>
                <button @click="modalOpen = false" class="modal-close" aria-label="ปิด">
                    &times;
                </button>
            </div>
            <div class="modal-body">
                {{ modal_body|default:"" }}
            </div>
        </div>
    </div>
</template>
```

**หมายเหตุเรื่อง `x-teleport`**: directive นี้ย้าย element ไปวางไว้ที่ตำแหน่ง
อื่นใน DOM (ในที่นี้คือ `body`) ทำให้ modal ไม่ถูกจำกัดด้วย `overflow: hidden`
หรือ `z-index` ของ container แม่ ซึ่งเป็นปัญหาคลาสสิกของ modal ที่ฝังลึกใน
component อื่น ส่วน `x-trap` (ต้องโหลด `@alpinejs/focus` plugin) จะ "ขัง"
focus ของ keyboard ให้วนอยู่ใน modal เท่านั้น (กด Tab ไม่หลุดออกไปนอก modal)
— สำคัญมากสำหรับผู้ใช้ที่พึ่งพา keyboard navigation

```html
<!-- ตัวอย่างการใช้งาน modal ในหน้าจริง -->
{% extends "base.html" %}

{% block content %}
<div x-data="{ modalOpen: false }">
    <button @click="modalOpen = true" class="btn btn-primary">
        เปิดข้อมูลติดต่อ
    </button>

    <template x-teleport="body">
        <div x-show="modalOpen" x-cloak class="modal-backdrop" @keydown.escape.window="modalOpen = false">
            <div
                x-show="modalOpen"
                x-transition
                @click.outside="modalOpen = false"
                class="modal-box"
                role="dialog"
                aria-modal="true"
            >
                <h3>ข้อมูลติดต่อ</h3>
                <p>อีเมล: contact@myblog.example.com</p>
                <button @click="modalOpen = false" class="btn">ปิด</button>
            </div>
        </div>
    </template>
</div>
{% endblock %}
```

### 535.3 CSS ที่ต้องมีคู่กับ Modal (Tailwind และ Plain CSS)

```css
/* static/css/components.css — สำหรับกรณีไม่ได้ใช้ Tailwind */
.modal-backdrop {
    position: fixed;
    inset: 0;
    background: rgba(0, 0, 0, 0.5);
    display: flex;
    align-items: center;
    justify-content: center;
    z-index: 50;
}

.modal-box {
    background: white;
    border-radius: 0.5rem;
    padding: 1.5rem;
    max-width: 28rem;
    width: 90%;
    box-shadow: 0 20px 25px -5px rgba(0, 0, 0, 0.1);
}

.dropdown {
    position: relative;
    display: inline-block;
}

.dropdown-menu {
    position: absolute;
    right: 0;
    top: 100%;
    margin-top: 0.5rem;
    background: white;
    border-radius: 0.375rem;
    box-shadow: 0 10px 15px -3px rgba(0, 0, 0, 0.1);
    min-width: 12rem;
    z-index: 40;
}

[x-cloak] { display: none !important; }
```

หากใช้ Tailwind CSS ตามที่ตั้งค่าไว้ในขั้นตอนที่ 534 สามารถแทนที่ class ด้านบน
ด้วย utility class ได้ทันที เช่น `class="fixed inset-0 bg-black/50 flex
items-center justify-center z-50"` แทน `.modal-backdrop`

### 535.4 Tab Component ด้วย Alpine.js (โบนัส Pattern ที่ใช้บ่อย)

```html
<div x-data="{ activeTab: 'details' }" class="tabs-container">
    <div class="tabs-nav" role="tablist">
        <button
            @click="activeTab = 'details'"
            :class="{ 'tab-active': activeTab === 'details' }"
            role="tab"
            :aria-selected="activeTab === 'details'"
        >
            รายละเอียด
        </button>
        <button
            @click="activeTab = 'comments'"
            :class="{ 'tab-active': activeTab === 'comments' }"
            role="tab"
            :aria-selected="activeTab === 'comments'"
        >
            ความคิดเห็น ({{ post.comments.count }})
        </button>
        <button
            @click="activeTab = 'related'"
            :class="{ 'tab-active': activeTab === 'related' }"
            role="tab"
        >
            บทความที่เกี่ยวข้อง
        </button>
    </div>

    <div x-show="activeTab === 'details'" role="tabpanel">
        {{ post.content|linebreaks }}
    </div>
    <div x-show="activeTab === 'comments'" role="tabpanel" x-cloak>
        {% include "blog/_comments.html" %}
    </div>
    <div x-show="activeTab === 'related'" role="tabpanel" x-cloak>
        {% include "blog/_related_posts.html" %}
    </div>
</div>
```

สังเกตว่าทุก tab ถูก render โดย Django มาให้ครบตั้งแต่ต้น (รวมถึง comment
และ related post) — Alpine.js แค่สลับการแสดงผลด้วย CSS เท่านั้น เหมาะกับ
เนื้อหาที่ไม่ใหญ่มาก ถ้าเนื้อหาแต่ละ tab หนักมาก (เช่น ต้อง query ข้อมูลเยอะ)
ควรพิจารณาใช้ HTMX (`hx-get` เมื่อคลิก tab) แทนเพื่อ lazy-load แทนที่จะโหลด
มาทั้งหมดตั้งแต่แรก

---

## ขั้นตอนที่ 536: Alpine.js Store สำหรับแชร์ State ข้าม Component

### 536.1 ปัญหาที่ Store แก้: State ที่ต้องใช้ร่วมกันหลายจุดในหน้าเดียว

`x-data` มี scope จำกัดอยู่แค่ element และลูกหลานของมันเท่านั้น แต่บางครั้ง
เราต้องการ state ที่ **ใช้ร่วมกันได้ทั่วทั้งหน้า** (หรือทั่วทั้งเว็บไซต์) เช่น:

- จำนวนสินค้าใน "ตะกร้า" ที่ต้องแสดงทั้งใน navbar และในหน้ารายละเอียดสินค้า
- สถานะ Dark Mode ที่ต้องรู้ทั่วทั้งหน้า
- สถานะเปิด/ปิดของ sidebar ที่ควบคุมจาก navbar แต่ sidebar อยู่คนละที่ใน DOM

`Alpine.store()` คือคำตอบสำหรับปัญหานี้ — มันสร้าง **global reactive state**
ที่ทุก component เข้าถึงได้ผ่าน `$store`

### 536.2 ประกาศ Store

```html
<!-- templates/base.html -->
<script defer src="https://cdn.jsdelivr.net/npm/alpinejs@3.x.x/dist/cdn.min.js"></script>
<script>
    document.addEventListener('alpine:init', () => {
        Alpine.store('cart', {
            items: [],
            count: 0,

            add(productId, productName, price) {
                this.items.push({ id: productId, name: productName, price: price });
                this.count = this.items.length;
            },

            remove(productId) {
                this.items = this.items.filter(item => item.id !== productId);
                this.count = this.items.length;
            },

            get total() {
                return this.items.reduce((sum, item) => sum + item.price, 0);
            },
        });

        Alpine.store('theme', {
            dark: false,
            toggle() {
                this.dark = !this.dark;
                document.documentElement.classList.toggle('dark', this.dark);
            },
        });
    });
</script>
```

**สำคัญมาก**: ต้องประกาศ store ภายใน event listener `alpine:init` เสมอ และ
`<script>` นี้ต้องอยู่**ก่อน** `<script>` ของ Alpine.js core (เพราะ event
`alpine:init` จะยิงตอนที่ Alpine core โหลดเสร็จพอดี ถ้าประกาศหลังจากนั้น
event จะยิงไปแล้วและ listener จะไม่ทำงาน)

### 536.3 ใช้งาน Store จากหลาย Component

```html
<!-- ในตำแหน่ง navbar (บนสุดของหน้า) -->
<div class="navbar">
    <a href="{% url 'shop:cart_detail' %}" class="cart-icon">
        🛒 <span x-text="$store.cart.count"></span>
    </a>
</div>

<!-- ในหน้ารายละเอียดสินค้า (ห่างจาก navbar มากใน DOM) -->
<div x-data="{ product: { id: {{ product.id }}, name: '{{ product.name|escapejs }}', price: {{ product.price }} } }">
    <h1>{{ product.name }}</h1>
    <p>ราคา: {{ product.price }} บาท</p>
    <button @click="$store.cart.add(product.id, product.name, product.price)" class="btn btn-primary">
        เพิ่มลงตะกร้า
    </button>
</div>

<!-- ปุ่ม Dark Mode ที่ไหนก็ได้ในหน้า -->
<button @click="$store.theme.toggle()" x-text="$store.theme.dark ? '☀️ Light Mode' : '🌙 Dark Mode'"></button>
```

สังเกตว่า navbar และหน้ารายละเอียดสินค้าอยู่คนละ `x-data` scope กันโดย
สิ้นเชิง (คนละ element, ไม่มีความสัมพันธ์แบบ parent-child ใน DOM) แต่ยัง
สื่อสารกันได้ผ่าน `$store.cart` — นี่คือประโยชน์หลักของ Store

### 536.4 Persist State ด้วย `@alpinejs/persist` Plugin (บันทึกลง localStorage)

```html
<script>
    document.addEventListener('alpine:init', () => {
        Alpine.store('theme', {
            // Alpine.$persist ทำให้ค่าถูกบันทึกลง localStorage อัตโนมัติ
            // และโหลดกลับมาเมื่อผู้ใช้กลับมาที่เว็บไซต์ใหม่
            dark: Alpine.$persist(false).as('theme_dark_mode'),

            toggle() {
                this.dark = !this.dark;
                document.documentElement.classList.toggle('dark', this.dark);
            },
        });
    });
</script>
```

ด้วย plugin นี้ ถ้าผู้ใช้เปิด Dark Mode แล้วปิดเบราว์เซอร์ไป พอเปิดเว็บไซต์ใหม่
Dark Mode จะยังคงเปิดอยู่ (เพราะอ่านค่าจาก `localStorage` โดยอัตโนมัติ) —
นี่คือ **client-side persistence** ที่ไม่ต้องพึ่ง Django session หรือ
database เลย เหมาะกับ preference ที่ไม่จำเป็นต้อง sync ข้าม device

### 536.5 เปรียบเทียบ: เมื่อไหร่ควรใช้ Store vs เมื่อไหร่ควรใช้ Django Session

| สถานการณ์ | ควรใช้ |
|---|---|
| Dark mode preference (ไม่ต้อง sync ข้ามเครื่อง) | `Alpine.store` + `$persist` (localStorage) |
| ตะกร้าสินค้าของผู้ใช้ที่ยังไม่ login (guest cart) | `Alpine.store` + `$persist` ชั่วคราว แล้วค่อย sync เข้า Django Session/DB ตอน checkout |
| ตะกร้าสินค้าของผู้ใช้ที่ login แล้ว ต้องดูได้จากหลายอุปกรณ์ | Django Model (database) — ต้องดึงมาแสดงผ่าน context หรือ API |
| สถานะ sidebar เปิด/ปิด (UI preference ล้วน ๆ) | `Alpine.store` (ไม่จำเป็นต้อง persist ด้วยซ้ำ) |
| ข้อมูล user ที่ login (username, permissions) | Django Session/Auth — **ห้าม** เก็บข้อมูลสำคัญไว้ที่ client-side store เด็ดขาด |

**หลักการสำคัญ**: `Alpine.store` เหมาะกับ **UI state** และ **preference ที่ไม่
sensitive** เท่านั้น ข้อมูลที่เป็นความจริงของระบบ (source of truth) เช่น
ยอดเงินในตะกร้า, สิทธิ์การเข้าถึง, ข้อมูลผู้ใช้ ต้องยึด Django/Database เป็นหลัก
เสมอ — Alpine store เป็นเพียง "ภาพสะท้อน" ชั่วคราวของข้อมูลนั้นบนหน้าจอเท่านั้น

---

## ขั้นตอนที่ 537: `x-model` Two-way Binding ผสานกับ Django Form Field

### 537.1 `x-model` ทำงานอย่างไร

`x-model` ผูกค่าของ input เข้ากับตัวแปรใน Alpine state แบบ **two-way**: เมื่อ
ผู้ใช้พิมพ์ใน input ตัวแปรจะอัปเดตทันที และถ้าตัวแปรเปลี่ยนจากที่อื่น ค่าใน
input ก็จะเปลี่ยนตามไปด้วย

```html
<div x-data="{ search: '' }">
    <input type="text" x-model="search" placeholder="ค้นหา...">
    <p>คุณพิมพ์ว่า: <span x-text="search"></span></p>
</div>
```

### 537.2 ตัวอย่างจริง: Live Character Counter สำหรับ Django Form

สถานการณ์: ฟอร์มเขียนโพสต์มีช่อง "excerpt" (`max_length=300`) เราต้องการแสดง
จำนวนตัวอักษรที่เหลือแบบ real-time โดยยังคงใช้ Django Form validation ที่
ฝั่งเซิร์ฟเวอร์เป็นตัวตัดสินสุดท้ายเสมอ (client-side เป็นแค่ UX เสริม):

```python
# blog/forms.py
from django import forms

from .models import Post


class PostForm(forms.ModelForm):
    class Meta:
        model = Post
        fields = ["title", "excerpt", "content"]
        widgets = {
            "excerpt": forms.Textarea(attrs={"rows": 3, "maxlength": 300}),
            "content": forms.Textarea(attrs={"rows": 10}),
        }
```

```python
# blog/views.py (เพิ่มเติม)
from django.contrib.auth.decorators import login_required

from .forms import PostForm


@login_required
def post_create(request):
    if request.method == "POST":
        form = PostForm(request.POST)
        if form.is_valid():
            post = form.save(commit=False)
            post.author = request.user
            post.save()
            return redirect(post.get_absolute_url())
    else:
        form = PostForm()
    return render(request, "blog/post_form.html", {"form": form})
```

```html
<!-- templates/blog/post_form.html -->
{% extends "base.html" %}

{% block content %}
<form method="post" x-data="{
    excerpt: '{{ form.excerpt.value|default:''|escapejs }}',
    maxLength: 300,
    get remaining() {
        return this.maxLength - this.excerpt.length;
    },
    get isOverLimit() {
        return this.remaining < 0;
    }
}">
    {% csrf_token %}

    <div class="form-group">
        <label for="{{ form.title.id_for_label }}">หัวข้อ</label>
        {{ form.title }}
        {% if form.title.errors %}
            <p class="field-error">{{ form.title.errors.0 }}</p>
        {% endif %}
    </div>

    <div class="form-group">
        <label for="{{ form.excerpt.id_for_label }}">คำอธิบายสั้น</label>
        <textarea
            name="excerpt"
            id="{{ form.excerpt.id_for_label }}"
            x-model="excerpt"
            rows="3"
            class="form-control"
        ></textarea>
        <p
            class="char-counter"
            :class="{ 'char-counter-danger': isOverLimit }"
            x-text="remaining + ' ตัวอักษรที่เหลือ'"
        ></p>
        {% if form.excerpt.errors %}
            <p class="field-error">{{ form.excerpt.errors.0 }}</p>
        {% endif %}
    </div>

    <div class="form-group">
        <label for="{{ form.content.id_for_label }}">เนื้อหา</label>
        {{ form.content }}
        {% if form.content.errors %}
            <p class="field-error">{{ form.content.errors.0 }}</p>
        {% endif %}
    </div>

    <button type="submit" :disabled="isOverLimit" class="btn btn-primary">
        บันทึกบทความ
    </button>
</form>
{% endblock %}
```

**จุดสำคัญที่ต้องเข้าใจอย่างลึกซึ้ง**:

1. `x-model="excerpt"` ผูกกับ `<textarea name="excerpt">` — เมื่อฟอร์ม
   submit ค่าที่ Django ได้รับจะเป็นค่าล่าสุดที่ผู้ใช้พิมพ์ (เพราะ `x-model`
   sync กับค่าจริงของ input element เสมอ ไม่ใช่แค่ตัวแปรลอย ๆ)
2. `excerpt: '{{ form.excerpt.value|default:""|escapejs }}'` ทำให้ค่า
   เริ่มต้นของ Alpine state ตรงกับค่าที่มีอยู่แล้วในฟอร์ม (สำคัญมากตอนแก้ไข
   โพสต์เดิม ไม่ใช่แค่สร้างใหม่ — ถ้าไม่ทำแบบนี้ ตัวนับตัวอักษรจะเริ่มจาก 0
   ทั้งที่ในฟอร์มมีข้อความอยู่แล้ว)
3. `get remaining()` เป็น **computed property** ของ Alpine (getter ธรรมดา
   ของ JavaScript) — คำนวณใหม่อัตโนมัติทุกครั้งที่ `excerpt` เปลี่ยน โดยไม่
   ต้องเขียน `$watch` เอง
4. `:disabled="isOverLimit"` ปิดปุ่ม submit ฝั่ง client เมื่อพิมพ์เกิน — แต่นี่
   เป็นแค่ **UX convenience** เท่านั้น! Django `ModelForm` ที่มี
   `maxlength=300` จาก Model field (`excerpt = models.CharField(max_length=300)`)
   จะ validate ซ้ำที่ฝั่งเซิร์ฟเวอร์เสมอ — **ห้ามเชื่อ client-side validation
   เพียงอย่างเดียวเด็ดขาด** เพราะผู้ใช้สามารถแก้ไข HTML/JavaScript ผ่าน
   DevTools เพื่อ bypass การตรวจสอบฝั่ง client ได้เสมอ

### 537.3 ตัวอย่างที่สอง: ตรวจสอบรหัสผ่านตรงกัน (Client-side Preview เท่านั้น)

```html
<form method="post" x-data="{
    password1: '',
    password2: '',
    get passwordsMatch() {
        return this.password2 === '' || this.password1 === this.password2;
    }
}">
    {% csrf_token %}

    <div class="form-group">
        <label>รหัสผ่าน</label>
        <input type="password" name="password1" x-model="password1" class="form-control">
    </div>

    <div class="form-group">
        <label>ยืนยันรหัสผ่าน</label>
        <input type="password" name="password2" x-model="password2" class="form-control">
        <p x-show="!passwordsMatch" x-cloak class="field-error">
            รหัสผ่านไม่ตรงกัน
        </p>
    </div>

    <button type="submit" class="btn btn-primary">สมัครสมาชิก</button>
</form>
```

โค้ดฝั่ง Django ยังคงต้อง validate ว่ารหัสผ่านตรงกันจริงเสมอ (เช่นใน
`UserCreationForm.clean()` ที่ Django ให้มาอยู่แล้ว) — Alpine.js เพียงแค่
**บอกผู้ใช้ล่วงหน้าก่อนกด submit** เพื่อลด round-trip ที่ไม่จำเป็น ไม่ใช่มา
แทนที่การตรวจสอบฝั่งเซิร์ฟเวอร์

### 537.4 `x-model` กับ Checkbox และ Select

```html
<div x-data="{ categories: [], newsletter: false }">
    <!-- Checkbox เดี่ยว: ผูกกับ boolean -->
    <label>
        <input type="checkbox" name="newsletter" x-model="newsletter">
        สมัครรับจดหมายข่าว
    </label>

    <!-- Checkbox กลุ่ม: ผูกกับ array -->
    <label>
        <input type="checkbox" name="categories" value="tech" x-model="categories">
        เทคโนโลยี
    </label>
    <label>
        <input type="checkbox" name="categories" value="life" x-model="categories">
        ไลฟ์สไตล์
    </label>

    <!-- Select -->
    <select x-model="categories" multiple>
        <option value="tech">เทคโนโลยี</option>
        <option value="life">ไลฟ์สไตล์</option>
    </select>

    <p>เลือกไว้: <span x-text="categories.join(', ')"></span></p>
</div>
```

Alpine.js จัดการเรื่องนี้ให้อัตโนมัติ — ถ้า `x-model` ผูกกับ checkbox หลายตัวที่
ใช้ตัวแปรเดียวกัน (array) มันจะ push/remove ค่าให้เองตาม checked state โดย
ไม่ต้องเขียน logic เพิ่ม

### 537.5 Modifier ที่มีประโยชน์กับ `x-model`

| Modifier | หน้าที่ |
|---|---|
| `.lazy` | อัปเดตตัวแปรตอน `change` event แทน `input` (คือรอจน blur/enter แทนที่จะทุก keystroke) |
| `.number` | แปลงค่าเป็น number อัตโนมัติ (`parseFloat`) |
| `.debounce.500ms` | หน่วงเวลาก่อนอัปเดต มีประโยชน์กับ search-as-you-type |
| `.trim` | ตัดช่องว่างหน้า-หลังอัตโนมัติ |

```html
<input type="text" x-model.debounce.500ms="searchQuery" placeholder="ค้นหา...">
<input type="number" x-model.number="quantity">
<input type="text" x-model.trim="username">
```

---

## ขั้นตอนที่ 538: ข้อควรพิจารณาด้าน Testing สำหรับหน้าที่มี Alpine.js เสริม

### 538.1 หลักการสำคัญ: Django Test แทบไม่รู้จัก Alpine.js เลย

จุดที่ต้องเข้าใจให้ชัดเจนตั้งแต่ต้น: `django.test.TestCase` และ `Client`
ทำงานโดยการยิง HTTP request ไปที่ View แล้วตรวจสอบ HTML response ที่ได้กลับมา
**มันไม่รัน JavaScript เลย** เพราะฉะนั้น Alpine.js (ซึ่งทำงานหลังจากเบราว์เซอร์
ได้รับ HTML แล้วเท่านั้น) จะไม่ถูกทดสอบผ่าน Django TestCase แบบปกติ

```python
# blog/tests/test_views.py
from django.contrib.auth import get_user_model
from django.test import TestCase
from django.urls import reverse

from blog.models import Post

User = get_user_model()


class PostListViewTests(TestCase):
    def setUp(self):
        self.user = User.objects.create_user(username="tester", password="pass1234")
        self.post = Post.objects.create(
            title="ทดสอบ Alpine.js",
            slug="test-alpine",
            excerpt="สั้น ๆ",
            content="เนื้อหาแบบเต็ม" * 20,
            author=self.user,
        )

    def test_post_list_renders_alpine_directives_in_html(self):
        """
        เราไม่ได้ทดสอบว่า Alpine.js 'ทำงาน' จริง ๆ (เช่น กดปุ่มแล้วซ่อน/แสดง)
        แต่ทดสอบว่า Django render attribute ที่ถูกต้องออกมาใน HTML
        ซึ่งเป็นสิ่งที่ Django TestCase ทำได้และควรทำ
        """
        response = self.client.get(reverse("blog:post_list"))
        self.assertEqual(response.status_code, 200)
        self.assertContains(response, 'x-data="{ expanded: false }"')
        self.assertContains(response, self.post.excerpt)
        # เนื้อหาเต็มต้องอยู่ใน HTML ตั้งแต่แรก (สำหรับ SEO)
        # แม้ Alpine.js จะซ่อนมันไว้ด้วย x-show
        self.assertContains(response, self.post.content)

    def test_post_list_escapes_username_correctly_for_js_context(self):
        """
        ทดสอบว่าค่าที่ฝังลงใน x-data ผ่าน escapejs ถูก escape จริง
        ป้องกัน XSS ผ่าน JavaScript context
        """
        malicious_user = User.objects.create_user(
            username="test</script><script>alert(1)</script>",
            password="pass1234",
        )
        Post.objects.create(
            title="โพสต์อันตราย",
            slug="dangerous-post",
            excerpt="x",
            content="y",
            author=malicious_user,
        )
        response = self.client.get(reverse("blog:post_list"))
        self.assertNotContains(response, "<script>alert(1)</script>")
```

### 538.2 สิ่งที่ Django TestCase ควรทดสอบ vs ไม่ควรทดสอบ

| ควรทดสอบด้วย Django TestCase | ไม่ควร (ต้องใช้เครื่องมืออื่น) |
|---|---|
| HTML ที่ render ออกมามี attribute `x-data`, `x-show` ที่ถูกต้องครบถ้วน | การคลิกปุ่มแล้ว UI เปลี่ยนจริงหรือไม่ |
| ค่าที่ฝังใน `x-data` ถูก escape ป้องกัน XSS อย่างถูกต้อง | Animation/transition ทำงานลื่นไหลหรือไม่ |
| เนื้อหาที่ควรถูกซ่อนด้วย `x-show` ยังคงอยู่ครบใน HTML (สำหรับ SEO/no-JS) | localStorage persist ค่าได้จริงหรือไม่ |
| View, Form validation, permission ทำงานถูกต้อง (ไม่เกี่ยวกับ Alpine เลย) | `Alpine.store` sync ข้าม component ถูกต้องหรือไม่ |
| CSRF token ถูกส่งไปพร้อม HTMX/Alpine request อย่างถูกต้อง | Focus trap ใน modal ทำงานตาม accessibility spec หรือไม่ |

### 538.3 ทดสอบพฤติกรรมจริงของ Alpine.js ด้วย Browser Automation

การทดสอบว่า "กดปุ่มแล้ว modal เปิดจริง" ต้องใช้เครื่องมือที่รัน JavaScript
จริงในเบราว์เซอร์ เช่น **Playwright** หรือ **Selenium** (เราจะเรียนละเอียด
ใน Part 064: Integration Testing และ Selenium) ตัวอย่างคร่าว ๆ ด้วย
`django.test.LiveServerTestCase` ร่วมกับ Playwright:

```python
# blog/tests/test_alpine_behavior.py
from django.contrib.staticfiles.testing import StaticLiveServerTestCase
from playwright.sync_api import sync_playwright

from blog.models import Post
from django.contrib.auth import get_user_model

User = get_user_model()


class AlpineToggleBehaviorTests(StaticLiveServerTestCase):
    """
    ทดสอบพฤติกรรมจริงของ Alpine.js โดยรันเบราว์เซอร์จริง (headless)
    ใช้ LiveServerTestCase เพื่อให้มี server จริงที่ Playwright เข้าถึงได้
    """

    @classmethod
    def setUpClass(cls):
        super().setUpClass()
        cls.playwright = sync_playwright().start()
        cls.browser = cls.playwright.chromium.launch()

    @classmethod
    def tearDownClass(cls):
        cls.browser.close()
        cls.playwright.stop()
        super().tearDownClass()

    def setUp(self):
        self.user = User.objects.create_user(username="tester", password="pass1234")
        self.post = Post.objects.create(
            title="ทดสอบ Toggle",
            slug="test-toggle",
            excerpt="excerpt สั้น",
            content="เนื้อหาเต็มที่ยาวกว่า excerpt มาก",
            author=self.user,
        )

    def test_click_read_more_shows_full_content(self):
        page = self.browser.new_page()
        page.goto(f"{self.live_server_url}/blog/")

        # เนื้อหาเต็มต้องยังไม่แสดงตอนแรก (ถูก x-show ซ่อนไว้)
        full_content_locator = page.get_by_text(self.post.content)
        assert not full_content_locator.is_visible()

        # คลิกปุ่ม "อ่านเพิ่มเติม"
        page.get_by_text("อ่านเพิ่มเติม").click()

        # ตอนนี้เนื้อหาเต็มต้องแสดงแล้ว (Alpine.js ทำงาน)
        assert full_content_locator.is_visible()

        page.close()
```

### 538.4 หลักการ Progressive Enhancement: ทดสอบว่าหน้าใช้งานได้แม้ไม่มี JavaScript

เพราะ Alpine.js เป็นแค่ "ของเสริม" หน้าเว็บที่ดีควรยังพอใช้งานได้ (หรืออย่าง
น้อยไม่พังจนอ่านไม่ได้) แม้ JavaScript จะโหลดไม่สำเร็จ ข้อควรระวังที่พบบ่อย:

```html
<!-- ❌ ไม่ดี: ถ้า Alpine.js โหลดไม่สำเร็จ ปุ่มนี้จะกดไม่ได้เลย
     และเนื้อหาที่ควรซ่อนไว้ด้วย x-show จะ "ค้าง" อยู่ในสถานะเริ่มต้นตลอดไป -->
<div x-data="{ open: false }">
    <button @click="open = !open">ดูรายละเอียด</button>
    <div x-show="open">รายละเอียดสำคัญที่ผู้ใช้ต้องเห็น</div>
</div>

<!-- ✅ ดีกว่า: ถ้าเนื้อหาสำคัญมาก ให้แสดงไว้เป็นค่าเริ่มต้น (open: true)
     หรือใช้ <noscript> สำรอง -->
<div x-data="{ open: false }">
    <button @click="open = !open">ซ่อน/แสดงรายละเอียด</button>
    <div x-show="open" x-cloak>รายละเอียดเสริม (ไม่จำเป็นต่อการอ่านหลัก)</div>
    <noscript>
        <div>รายละเอียดสำคัญที่ผู้ใช้ต้องเห็น (แสดงเสมอถ้าไม่มี JavaScript)</div>
    </noscript>
</div>
```

**คำแนะนำระดับมืออาชีพ**: ใช้ Alpine.js สำหรับ interaction ที่เป็น
**"nice-to-have"** เท่านั้น (toggle, animation, dropdown ที่ไม่ใช่เนื้อหาหลัก)
ส่วนเนื้อหาที่จำเป็นต่อการทำความเข้าใจหน้าเว็บ ควรแสดงอยู่ใน HTML ตั้งแต่ต้น
เสมอ ไม่ว่า Alpine.js จะทำงานหรือไม่ก็ตาม

### 538.5 Checklist สำหรับ Code Review หน้าที่มี Alpine.js

- [ ] ทุก `x-data` ที่รับค่าจาก Django ผ่าน escape ที่ถูกต้อง (`escapejs`,
      `yesno`) เพื่อป้องกัน XSS และ syntax error
- [ ] Element ที่ใช้ `x-cloak` มี CSS `[x-cloak] { display: none !important; }`
      อยู่ใน `base.html` แล้ว
- [ ] Modal และ Dropdown มี `@click.outside` และ `@keydown.escape.window`
- [ ] Client-side validation (เช่น `x-model` + `:disabled`) มี server-side
      validation คู่กันเสมอใน Django Form/View
- [ ] เนื้อหาสำคัญไม่ได้ถูกซ่อนแบบที่ทำให้ใช้งานไม่ได้เมื่อ JavaScript พัง
- [ ] มี test อย่างน้อยระดับ "HTML ที่ render ออกมาถูกต้อง" (Django TestCase)
      แม้จะยังไม่มี browser automation test ครบทุกจุด

---

## ขั้นตอนที่ 539: ขีดจำกัดของ Alpine.js — เมื่อไหร่ควรย้ายไปใช้ React/Vue เต็มรูปแบบ

### 539.1 Alpine.js ไม่ได้ถูกออกแบบมาให้สร้างแอปทั้งแอป

สิ่งสำคัญที่สุดที่ต้องเข้าใจ: Alpine.js ออกแบบมาเพื่อ **"sprinkle"**
(โรยพฤติกรรม) ไม่ใช่เพื่อสร้าง Single Page Application เต็มรูปแบบ เมื่อ
โปรเจกต์เติบโตขึ้น มีสัญญาณหลายอย่างที่บอกว่าถึงเวลาต้องพิจารณา React หรือ
Vue.js เต็มรูปแบบแล้ว

### 539.2 สัญญาณเตือนว่า Alpine.js เริ่มไม่พอ

| สัญญาณ | อธิบาย |
|---|---|
| **State ซับซ้อนเกินไป** | ถ้า `x-data` object มี property มากกว่า 15-20 ตัว หรือมี nested logic ลึกหลายชั้น โค้ดจะอ่านยากมากเพราะ Alpine ไม่มีระบบจัดการ state ที่เป็นระบบเหมือน Redux/Pinia |
| **ต้องการ Component Reusability ข้ามหลายหน้า** | Alpine ไม่มี component system แบบ `.vue`/`.jsx` ที่ import/export กันได้ ถ้าต้องใช้ dropdown/modal เดียวกันในหลายสิบหน้าและต้องการ prop แบบซับซ้อน จะเริ่มรู้สึกอึดอัด (แม้ `Alpine.data()` จะช่วยได้ระดับหนึ่ง) |
| **UI ต้อง render รายการจำนวนมาก (พันรายการขึ้นไป)** | Alpine ไม่มี Virtual DOM diffing ที่มีประสิทธิภาพสูงแบบ React ทำให้ re-render list ใหญ่ ๆ บ่อย ๆ อาจช้า |
| **ต้องการ Client-side Routing** | เว็บแบบ SPA ที่เปลี่ยนหน้าโดยไม่ reload เลย (เช่น Dashboard ที่ซับซ้อน) ต้องมี router ซึ่ง Alpine ไม่มีให้ |
| **Real-time Collaboration ที่ซับซ้อน** | เช่น Google Docs-style editor ที่หลายคนแก้พร้อมกัน ต้องการ state management + reconciliation ที่ซับซ้อนเกินกว่า Alpine จะรองรับได้ดี |
| **ทีมมี Frontend Specialist แยกต่างหาก** | ถ้าทีมโตขึ้นจนมี frontend developer เฉพาะทาง การมี React/Vue ที่มี ecosystem, dev tools, type-checking (TypeScript) ที่แข็งแรงกว่าจะคุ้มค่ากว่าในระยะยาว |
| **ต้องการ Offline-first / PWA เต็มรูปแบบ** | React/Vue ผสาน Service Worker, IndexedDB ได้เป็นระบบมากกว่า |
| **ต้องการ Mobile App ด้วย React Native** | ถ้าเขียน React บน web อยู่แล้ว การแชร์ logic ไปยัง React Native ทำได้ง่ายกว่าการเขียนแอปแยกจาก Alpine.js โดยสิ้นเชิง |

### 539.3 ตารางตัดสินใจ: Alpine.js เพียงพอ vs ต้องการ React/Vue

| ลักษณะฟีเจอร์ | Alpine.js เพียงพอ | ควรใช้ React/Vue |
|---|---|---|
| Toggle/Dropdown/Modal/Tab | ✅ | ไม่จำเป็น |
| Live character counter, form preview | ✅ | ไม่จำเป็น |
| Shopping cart widget แบบง่าย (จำนวน, total) | ✅ | ทั้งสองแบบได้ |
| Rich text editor พร้อม toolbar ซับซ้อน | ⚠️ พอทำได้แต่เหนื่อย | ✅ (มี library สำเร็จรูปเยอะกว่า) |
| Drag-and-drop Kanban board (Trello-clone) | ⚠️ ทำได้แต่ต้องเขียนเยอะเอง | ✅ |
| Dashboard ที่มี real-time chart อัปเดตถี่มาก | ⚠️ | ✅ |
| Data table ที่ sort/filter/paginate ฝั่ง client หลายพันแถว | ❌ ช้า | ✅ |
| Multi-step wizard form ที่ state ซับซ้อนมาก | ⚠️ พอไหวถ้า step ไม่เกิน 4-5 | ✅ ถ้าซับซ้อนกว่านั้น |
| แอปที่ทีม frontend แยกจาก backend ชัดเจน | ❌ | ✅ (คู่กับ DRF) |

### 539.4 ทางสายกลาง: ใช้ทั้งสามอย่างในโปรเจกต์เดียวกันได้

ข้อดีอย่างหนึ่งของสถาปัตยกรรมนี้คือ **ไม่ต้องเลือกอย่างใดอย่างหนึ่งทั้งโปรเจกต์**
ทีมจำนวนมากใช้ Django + HTMX + Alpine.js เป็นค่าเริ่มต้นสำหรับหน้าส่วนใหญ่
(หน้า CRUD ทั่วไป, marketing page, admin-like page) แต่ฝัง React component
เป็น "island" เฉพาะจุดที่ซับซ้อนจริง ๆ เช่น rich text editor หรือ interactive
chart:

```html
<!-- ตัวอย่าง: หน้าส่วนใหญ่ใช้ Django + Alpine.js -->
{% extends "base.html" %}
{% block content %}
    <div x-data="{ open: false }">...</div>  <!-- Alpine.js sprinkle -->

    <!-- ยกเว้นส่วนนี้ที่ mount React component แยกเข้ามาเฉพาะจุด -->
    <div id="react-chart-widget" data-chart-data="{{ chart_data_json }}"></div>
    <script type="module" src="{% static 'js/chart-widget.js' %}"></script>
{% endblock %}
```

รูปแบบนี้เรียกว่า **"Islands Architecture"** — ส่วนใหญ่ของหน้าเว็บยังคงเป็น
server-rendered HTML ธรรมดา (เร็ว, SEO ดี) มีเพียง "เกาะ" เล็ก ๆ ที่เป็น
JavaScript framework เต็มรูปแบบสำหรับ UI ที่ซับซ้อนจริง ๆ เท่านั้น

### 539.5 เกริ่นสำหรับ Part ถัดไป: React และ Vue.js

ใน **Part 055** คุณจะได้เรียนรู้การใช้ Django เป็น **API Backend ล้วน ๆ**
(ผ่าน Django REST Framework ที่เรียนไปแล้วใน Phase 5) ควบคู่กับ React ที่
เป็น SPA แยกออกมาโดยสมบูรณ์ และใน **Part 056** จะเป็น Vue.js ซึ่งมีแนวทาง
การผสานที่ยืดหยุ่นกว่า (ทั้งแบบ SPA แยกเต็มรูปแบบ และแบบฝัง Vue component
เข้าไปใน Django Template คล้าย Alpine.js แต่มีความสามารถมากกว่า) — เมื่อ
เรียนจบทั้งสาม Part นี้ คุณจะสามารถเลือก **stack ที่เหมาะกับแต่ละโปรเจกต์
จริง** ได้อย่างมีเหตุผล ไม่ใช่เลือกตามกระแสเพียงอย่างเดียว

---

## ขั้นตอนที่ 540: สรุปและแบบฝึกหัด — สร้าง Modal ยืนยันการลบโพสต์

### 540.1 โจทย์: Modal ยืนยันการลบโพสต์แบบ Production-ready

มาประกอบทุกอย่างที่เรียนมาใน Part นี้เข้าด้วยกัน สร้างฟีเจอร์ที่ใช้งานจริงบ่อย
ที่สุดในทุกเว็บแอป: **ปุ่มลบที่ต้องมี Modal ยืนยันก่อนเสมอ** (ป้องกันการกด
ผิดโดยไม่ได้ตั้งใจ) โดยใช้ Alpine.js ล้วน ๆ สำหรับ modal และยังคง submit
ฟอร์มไปยัง Django ตามปกติเพื่อความปลอดภัย (ไม่ใช้ `DELETE` ผ่าน `fetch()`
JavaScript ที่ซับซ้อนเกินความจำเป็น)

```python
# blog/views.py (เพิ่มเติม)
from django.contrib.auth.mixins import LoginRequiredMixin, UserPassesTestMixin
from django.urls import reverse_lazy
from django.views.generic import DeleteView

from .models import Post


class PostDeleteView(LoginRequiredMixin, UserPassesTestMixin, DeleteView):
    model = Post
    success_url = reverse_lazy("blog:post_list")

    def test_func(self):
        # อนุญาตให้เฉพาะเจ้าของโพสต์เท่านั้นที่ลบได้
        post = self.get_object()
        return self.request.user == post.author

    def get_queryset(self):
        return Post.objects.select_related("author")
```

```python
# blog/urls.py (เพิ่มเติม)
from django.urls import path

from . import views

app_name = "blog"

urlpatterns = [
    path("", views.post_list, name="post_list"),
    path("new/", views.post_create, name="post_create"),
    path("<slug:slug>/", views.post_detail, name="post_detail"),
    path("<slug:slug>/delete/", views.PostDeleteView.as_view(), name="post_delete"),
]
```

```html
<!-- templates/blog/post_list.html (ปรับปรุงจากขั้นตอนที่ 533) -->
{% extends "base.html" %}

{% block content %}
<div class="post-list">
    <h1>บทความทั้งหมด</h1>

    {% for post in posts %}
        <article x-data="{ expanded: false, confirmDelete: false }" class="post-card">
            <h2>{{ post.title }}</h2>
            <p class="meta">
                โดย {{ post.author.get_full_name|default:post.author.username }}
                · {{ post.created_at|date:"d M Y" }}
            </p>

            <p x-show="!expanded">{{ post.excerpt }}</p>
            <div x-show="expanded" x-transition>{{ post.content|linebreaks }}</div>

            <div class="post-actions">
                <button
                    @click="expanded = !expanded"
                    x-text="expanded ? 'ย่อกลับ ▲' : 'อ่านเพิ่มเติม ▼'"
                    class="btn-link"
                ></button>

                {% if request.user == post.author %}
                    <a href="{% url 'blog:post_update' post.slug %}" class="btn-link">
                        แก้ไข
                    </a>

                    <!-- ปุ่มเปิด Modal ยืนยันการลบ -->
                    <button @click="confirmDelete = true" class="btn-link btn-link-danger">
                        ลบ
                    </button>

                    <!-- Modal ยืนยันการลบ (Alpine.js ล้วน ๆ) -->
                    <template x-teleport="body">
                        <div
                            x-show="confirmDelete"
                            x-cloak
                            class="modal-backdrop"
                            @keydown.escape.window="confirmDelete = false"
                        >
                            <div
                                x-show="confirmDelete"
                                x-transition:enter="transition ease-out duration-200"
                                x-transition:enter-start="opacity-0 scale-90"
                                x-transition:enter-end="opacity-100 scale-100"
                                @click.outside="confirmDelete = false"
                                class="modal-box"
                                role="alertdialog"
                                aria-modal="true"
                                aria-labelledby="delete-modal-title-{{ post.pk }}"
                            >
                                <h3 id="delete-modal-title-{{ post.pk }}">
                                    ยืนยันการลบบทความ
                                </h3>
                                <p>
                                    คุณแน่ใจหรือไม่ว่าต้องการลบบทความ
                                    <strong>"{{ post.title }}"</strong>?
                                    การกระทำนี้ไม่สามารถย้อนกลับได้
                                </p>
                                <div class="modal-footer">
                                    <button
                                        @click="confirmDelete = false"
                                        type="button"
                                        class="btn btn-secondary"
                                    >
                                        ยกเลิก
                                    </button>
                                    <form
                                        method="post"
                                        action="{% url 'blog:post_delete' post.slug %}"
                                    >
                                        {% csrf_token %}
                                        <button type="submit" class="btn btn-danger">
                                            ลบบทความนี้
                                        </button>
                                    </form>
                                </div>
                            </div>
                        </div>
                    </template>
                {% endif %}
            </div>
        </article>
    {% empty %}
        <p>ยังไม่มีบทความ</p>
    {% endfor %}
</div>
{% endblock %}
```

**เหตุผลของการออกแบบนี้ (สำคัญมากในเชิง Best Practice)**:

1. การลบยังคงใช้ **`<form method="post">` ธรรมดาไปยัง Django View** ไม่ได้
   ใช้ `fetch()` หรือ HTMX เพื่อให้เห็นชัดว่า Alpine.js **ไม่ได้แทนที่**
   กลไกความปลอดภัยของ Django (`{% csrf_token %}`, `LoginRequiredMixin`,
   `UserPassesTestMixin`) เลยแม้แต่น้อย — Alpine.js ทำหน้าที่แค่ "แสดง/ซ่อน
   modal" เท่านั้น
2. `UserPassesTestMixin` + `test_func()` ตรวจสอบที่ **ฝั่งเซิร์ฟเวอร์เสมอ**
   ว่าเป็นเจ้าของโพสต์จริงหรือไม่ แม้ว่า Alpine.js จะซ่อนปุ่มลบไว้แล้วสำหรับ
   คนที่ไม่ใช่เจ้าของ (`{% if request.user == post.author %}`) — เพราะ
   การซ่อนปุ่มด้วย template condition (ฝั่งเซิร์ฟเวอร์ ก่อนส่ง HTML ไปเลย)
   ไม่ใช่ Alpine.js ป้องกันอะไรได้จริง ถ้าไม่มี `UserPassesTestMixin` คนอื่น
   ก็สามารถยิง POST ตรงไปที่ URL ลบได้อยู่ดี
3. ปุ่ม "ลบ" (เปิด modal) และปุ่ม "ยกเลิก" (ปิด modal) ทำงานทันทีโดยไม่มี
   round-trip ไปเซิร์ฟเวอร์เลย — Server ถูกเรียกใช้ **เฉพาะตอนที่ผู้ใช้ยืนยัน
   จริง ๆ** เท่านั้น (submit form ปุ่ม "ลบบทความนี้")
4. Modal นี้ทำงานได้เต็มรูปแบบทั้ง keyboard (`Esc` ปิด), mouse (`click.outside`
   ปิด), และ screen reader (`role="alertdialog"`, `aria-labelledby`)

### 540.2 สรุปสิ่งที่ได้เรียนรู้ใน Part นี้

- ✅ เข้าใจปรัชญา "minimal JavaScript framework" ของ Alpine.js และทำไมมัน
  เข้ากับ Django Template ได้ดีเพราะไม่ต้องมี build step
- ✅ ใช้งาน directive หลัก: `x-data`, `x-show`, `x-if`, `x-on`/`@`, `x-text`,
  `x-bind`/`:`, `x-transition`, `x-cloak` ได้อย่างคล่องแคล่ว
- ✅ ผสาน Alpine.js เข้ากับ Django Template จริง รวมถึงเทคนิคการฝังค่าจาก
  Django context ลงใน `x-data` อย่างปลอดภัยด้วย `escapejs` และ `yesno`
- ✅ เข้าใจรูปแบบ Stack "Django + HTMX + Alpine.js + Tailwind" และวิธีแบ่ง
  หน้าที่ระหว่าง HTMX (server communication) กับ Alpine.js (client UI state)
- ✅ สร้าง Dropdown, Modal, และ Tab component ที่ครบเรื่อง accessibility
  (`@click.outside`, `@keydown.escape.window`, `aria-*`)
- ✅ ใช้ `Alpine.store()` แชร์ state ข้าม component และเข้าใจขอบเขตว่าอะไร
  ควรอยู่ใน store อะไรควรอยู่ใน Django Session/Database
- ✅ ใช้ `x-model` ทำ two-way binding ร่วมกับ Django Form พร้อมเข้าใจว่า
  client-side validation ต้องมี server-side validation คู่กันเสมอ
- ✅ เข้าใจขอบเขตการทดสอบ: Django TestCase ทดสอบ HTML output ได้ แต่ต้องใช้
  browser automation (Playwright/Selenium) สำหรับพฤติกรรมจริงของ Alpine.js
- ✅ รู้ขีดจำกัดของ Alpine.js และสัญญาณที่บ่งบอกว่าถึงเวลาต้องใช้ React/Vue
  เต็มรูปแบบ หรือใช้ "Islands Architecture" ผสมกัน
- ✅ สร้าง Modal ยืนยันการลบโพสต์แบบ production-ready ที่ยึดความปลอดภัยจาก
  Django เป็นหลักเสมอ

### 540.3 Checklist ก่อนไป Part ถัดไป

- [ ] เพิ่ม `<script defer src="...alpinejs.../cdn.min.js">` ใน `base.html`
      ของโปรเจกต์คุณเองสำเร็จ และทดสอบด้วย counter ง่าย ๆ
- [ ] เพิ่ม CSS `[x-cloak] { display: none !important; }` ใน stylesheet หลัก
- [ ] สร้าง toggle "อ่านเพิ่มเติม" บนหน้าลิสต์ข้อมูลจริงในโปรเจกต์ของคุณ
      (ไม่ว่าจะเป็นบล็อก, สินค้า, หรือ resource อื่น)
- [ ] สร้าง Dropdown menu ที่ปิดได้ทั้งจาก `@click.outside` และ `Esc`
- [ ] สร้าง Modal ที่ใช้ `x-teleport="body"` และมี `role`/`aria-*` ครบถ้วน
- [ ] ทดลองสร้าง `Alpine.store()` อย่างน้อย 1 ตัวที่แชร์ state ข้าม 2
      component ที่ไม่ใช่ parent-child กัน
- [ ] เขียน Django Form ที่มี `x-model` แสดงจำนวนตัวอักษรที่เหลือแบบ real-time
- [ ] เขียน Django Test อย่างน้อย 1 เคสที่ตรวจสอบว่า `x-data` ที่มีค่าจาก
      context ถูก escape อย่างถูกต้อง
- [ ] ทำ Modal ยืนยันการลบให้สมบูรณ์ตามตัวอย่างในขั้นตอนที่ 540.1 บน Model
      ของคุณเอง พร้อมตรวจสอบสิทธิ์ที่ฝั่งเซิร์ฟเวอร์

### 540.4 แบบฝึกหัดท้ายบท

**แบบฝึกหัดที่ 1 (พื้นฐาน)**: สร้างหน้า Django Template ใหม่ที่มี "Image
Gallery" แบบง่าย ๆ ใช้ Alpine.js สร้าง Lightbox: เมื่อคลิกรูปภาพขนาดเล็ก
(thumbnail) ให้เปิด modal แสดงรูปขนาดใหญ่ พร้อมปุ่ม "ก่อนหน้า"/"ถัดไป" ที่
เปลี่ยนรูปโดยใช้ index ใน Alpine state (ไม่ต้อง reload หน้าเลย) ใช้ `x-data`
เก็บ array ของ URL รูปภาพและ index ปัจจุบัน

**แบบฝึกหัดที่ 2 (ประยุกต์ HTMX + Alpine.js)**: สร้างฟีเจอร์ "Add to Cart"
ที่ใช้ HTMX ส่ง POST ไปบันทึกลงฐานข้อมูลจริง (สร้าง Model `CartItem` ง่าย ๆ)
พร้อมใช้ Alpine.js แสดง toast notification "เพิ่มสินค้าแล้ว!" ที่ปรากฏ 2 วินาที
แล้วหายไปเอง (ใช้ `x-show` ร่วมกับ `setTimeout` ใน `x-init` หรือ `$watch`)
เมื่อ HTMX ได้รับ response สำเร็จ (ฟัง event `htmx:afterRequest`)

**แบบฝึกหัดที่ 3 (Store ขั้นสูง)**: สร้าง `Alpine.store('notifications')`
ที่เก็บ array ของข้อความแจ้งเตือน มีเมธอด `add(message, type)` และ
`remove(id)` แสดงผลเป็น "notification stack" มุมขวาบนของหน้าเว็บที่ลอย
อยู่เหนือทุก component (ใช้ `x-teleport="body"`) ทดสอบเรียกใช้จากหลายจุดใน
หน้าเดียวกัน เช่น จากปุ่ม "บันทึกสำเร็จ" และปุ่ม "เกิดข้อผิดพลาด" คนละที่กัน

**แบบฝึกหัดที่ 4 (ขั้นสูง — Testing)**: เขียน Django TestCase ที่ตรวจสอบว่า
หน้า post list ที่ทำใน 540.1:
1. ผู้ใช้ที่ไม่ใช่เจ้าของโพสต์จะไม่เห็นปุ่ม "ลบ" ใน HTML เลย (ตรวจด้วย
   `assertNotContains`)
2. ถ้ามีคนพยายามยิง `POST` ตรงไปที่ URL `blog:post_delete` ของโพสต์ที่ตัวเอง
   ไม่ได้เป็นเจ้าของ (โดยไม่ผ่าน UI/Modal เลย) ต้องได้ response `403 Forbidden`
   ยืนยันว่าการป้องกันที่แท้จริงอยู่ที่ Django ไม่ใช่การซ่อนปุ่มด้วย
   Alpine.js/Template condition เพียงอย่างเดียว

### 540.5 คำถามที่พบบ่อย (FAQ)

**Q: Alpine.js กับ jQuery ต่างกันยังไง ในเมื่อทั้งคู่ก็ "โรย" JavaScript ลงใน
HTML เหมือนกัน?**

A: jQuery เป็นแนวทาง **imperative** (บอกทีละขั้นตอนว่าต้องทำอะไร เช่น
`$('.btn').click(function() { $('.box').toggle(); })`) ต้องผูก selector เอง
และจัดการ DOM state เอง ส่วน Alpine.js เป็นแนวทาง **declarative** (ประกาศว่า
"ผลลัพธ์ที่ต้องการคืออะไร" เช่น `x-show="open"` แล้วปล่อยให้ Alpine จัดการ
การอัปเดต DOM ให้) ทำให้โค้ด Alpine.js สั้นกว่า อ่านง่ายกว่า และ scope ของ
แต่ละ component ชัดเจนกว่ามาก ไม่มีปัญหาเรื่อง global selector ชนกัน

**Q: ใช้ Alpine.js กับ HTMX พร้อมกันในหน้าเดียวได้ไหม จะขัดแย้งกันหรือเปล่า?**

A: ใช้ร่วมกันได้ดีมาก และเป็นที่นิยมมากในชุมชน (ดูขั้นตอนที่ 534) เพราะทำงาน
คนละหน้าที่กันชัดเจน — HTMX จัดการ HTTP request/response กับ Django ส่วน
Alpine.js จัดการ UI state ที่ไม่ต้องคุยกับ server เลย ไม่มีการชนกันของ syntax
เพราะทั้งคู่ใช้ HTML attribute เป็นหลัก (HTMX ใช้ `hx-*`, Alpine ใช้ `x-*`)
Alpine.js ยังมี event พิเศษ (`@htmx:before-request`, `@htmx:after-request`,
`@htmx:response-error`) ที่ให้ดักฟัง lifecycle ของ HTMX ได้โดยตรงอีกด้วย

**Q: ถ้าโปรเจกต์เริ่มมี `x-data` ที่ซับซ้อนมาก ควรทำอย่างไรก่อนจะย้ายไป React
เลย?**

A: ก่อนกระโดดไป React ทั้งระบบ ลองใช้ `Alpine.data()` เพื่อแยก logic ของ
component ที่ซับซ้อนออกไปไว้ในไฟล์ JavaScript แยกต่างหาก (ยังคงไม่ต้องมี
build step):

```javascript
// static/js/components.js
document.addEventListener('alpine:init', () => {
    Alpine.data('postCard', (initialExpanded = false) => ({
        expanded: initialExpanded,
        confirmDelete: false,
        toggle() {
            this.expanded = !this.expanded;
        },
    }));
});
```

```html
<article x-data="postCard()">
    <button @click="toggle()">...</button>
</article>
```

วิธีนี้ช่วยแยก logic ที่ซับซ้อนออกจาก HTML โดยยังไม่ต้องเปลี่ยน stack ทั้งหมด
ถ้าทำแบบนี้แล้วยังรู้สึกว่าไม่พอ (ดูสัญญาณเตือนในขั้นตอนที่ 539.2) นั่นคือ
เวลาที่ควรพิจารณา React/Vue อย่างจริงจัง

**Q: ทำไมตัวอย่างในบทนี้ยังใช้ `<form method="post">` ธรรมดาสำหรับการลบ
แทนที่จะใช้ HTMX หรือ `fetch()` ให้ "ทันสมัย" กว่านี้?**

A: เพราะการลบข้อมูลเป็นการกระทำที่ "มีผลถาวรและย้อนกลับไม่ได้" หลักการ
Progressive Enhancement แนะนำให้ action ที่สำคัญมากใช้กลไกพื้นฐานที่สุด
(HTML form submission) ที่ทำงานได้แม้ JavaScript ล้มเหลว บวกกับ CSRF
protection ที่ Django จัดการให้อัตโนมัติผ่าน `{% csrf_token %}` ในฟอร์ม
ธรรมดา หากต้องการทำ UX ที่ลื่นไหลกว่านี้ (ลบโดยไม่ reload หน้าเลย) สามารถ
เพิ่ม `hx-post` และ `hx-confirm` ของ HTMX เข้าไปแทนที่ modal ของ Alpine.js
ได้เช่นกัน (เป็นอีกทางเลือกหนึ่งที่ควรลองทำเป็นแบบฝึกหัดเสริม) — ประเด็นสำคัญ
คือ Alpine.js กับ HTMX ต่างก็เป็น "ทางเลือก" ในการปรับปรุง UX ไม่ใช่สิ่งที่
ต้องบังคับใช้ทุกจุดเสมอไป

**Q: ต้องรู้ React หรือ Vue มาก่อนถึงจะเรียน Alpine.js ได้ไหม?**

A: ไม่จำเป็นเลย Alpine.js ถูกออกแบบมาให้เรียนรู้ได้เร็วที่สุดในบรรดา reactive
framework ทั้งหมด แม้ปรัชญาการเขียน (declarative, reactive) จะได้แรงบันดาลใจ
มาจาก Vue.js อย่างชัดเจน (ผู้สร้าง Alpine.js เคยบอกว่าตั้งใจทำให้เหมือนการ
"เขียน Vue โดยไม่ต้องมี build step") แต่คุณสามารถเรียน Alpine.js ก่อนแล้ว
ค่อยไปเรียน Vue.js ใน Part 056 ทีหลังได้สบาย ๆ และจะพบว่าหลายแนวคิด (reactive
data, `x-model`/`v-model`, `x-show`/`v-show`) คล้ายกันมากจนรู้สึกคุ้นเคยทันที

---

## เตรียมตัวสำหรับ Part ถัดไป

**Part 055: Django กับ React (Django เป็น API Backend)** จะพาคุณก้าวข้าม
"การโรย JavaScript" ไปสู่การสร้าง **Single Page Application เต็มรูปแบบ**
โดยให้ Django ทำหน้าที่เป็น **API Backend ล้วน ๆ** ผ่าน Django REST Framework
ที่คุณเรียนไปแล้วใน Phase 5 (Part 39-50) คุณจะได้เรียนรู้การตั้งค่า CORS,
การจัดการ JWT Authentication ระหว่าง React กับ Django, การจัดโครงสร้าง
โปรเจกต์แบบแยก repository (หรือ monorepo), และวิธีตัดสินใจว่าเมื่อไหร่ควร
ทิ้ง Server-rendered Template แล้วไปใช้แนวทาง API-first แบบเต็มรูปแบบ ตาม
สัญญาณเตือนที่เราพูดถึงในขั้นตอนที่ 539

เตรียมทบทวน Django REST Framework (Part 39-50) ให้แม่นอีกครั้ง โดยเฉพาะ
เรื่อง Serializers, ViewSets, และ JWT Authentication เพราะ Part 055 จะใช้
ความรู้เหล่านี้เป็นฐานตลอดทั้งบท
