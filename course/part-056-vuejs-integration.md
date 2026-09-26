# Part 056: Django กับ Vue.js Integration

> **ขั้นตอนที่ 551-560 ของหลักสูตร** | Phase 6: Frontend Integration
>
> เป้าหมายของ Part นี้: ตั้งค่าโปรเจกต์ **Vue 3 + Vite** ด้วย Composition API
> ให้เป็น frontend แยกต่างหากที่คุยกับ Blog REST API ของ Django (ที่สร้างไว้ใน
> Part 044-046) ผ่าน HTTP ล้วน ๆ คุณจะเรียนรู้ `Vue Router` สำหรับ navigation,
> `Pinia` สำหรับ state management, การผสาน JWT Authentication แบบเดียวกับที่
> Part 055 ทำกับ React แต่คราวนี้ implement ด้วย Vue, แนวคิด Composition API
> (`ref`, `reactive`, `computed`, composable functions), การออกแบบ Reusable
> Component (`PostCard.vue`, `CommentList.vue`) และปิดท้ายด้วยการเปรียบเทียบ
> Vue กับ React อย่างตรงไปตรงมา รวมถึงข้อควรพิจารณาตอน deploy Vue build จริง
> เมื่อจบ Part นี้คุณจะมี Vue 3 frontend ที่สมบูรณ์ ทำงานคู่กับ Blog API ของ
> Django ได้ครบทั้ง list, detail, comment, และ login/logout ด้วย JWT

---

## สารบัญของ Part นี้

- ขั้นตอนที่ 551: ตั้งค่าโปรเจกต์ Vue 3 ด้วย Vite (Composition API)
- ขั้นตอนที่ 552: ดึงข้อมูลจาก Blog Django API มาแสดงใน Vue Component
- ขั้นตอนที่ 553: Vue Router สำหรับ navigation
- ขั้นตอนที่ 554: Pinia สำหรับ State Management
- ขั้นตอนที่ 555: การผสาน Vue กับ Django Authentication (JWT)
- ขั้นตอนที่ 556: Composition API patterns (`ref`, `reactive`, `computed`, composables)
- ขั้นตอนที่ 557: สร้าง Reusable Component (`PostCard.vue`, `CommentList.vue`)
- ขั้นตอนที่ 558: ตารางเปรียบเทียบ Vue vs React สำหรับใช้กับ Django Backend
- ขั้นตอนที่ 559: ข้อควรพิจารณาการ Deploy Vue build
- ขั้นตอนที่ 560: สรุปและแบบฝึกหัด — สร้าง Vue frontend ที่สมบูรณ์สำหรับ Blog API

---

## ขั้นตอนที่ 551: ตั้งค่าโปรเจกต์ Vue 3 ด้วย Vite (Composition API)

### 551.1 ทบทวนสถาปัตยกรรม "Decoupled Frontend" จาก Part 055

ใน Part 055 คุณได้เปลี่ยนวิธีคิดจากการให้ Django render HTML ผ่าน Template
(Part 001-054) มาเป็นการให้ Django ทำหน้าที่ **API Backend ล้วน ๆ** แล้วให้
React เป็นแอปแยกต่างหากที่ยิงคำขอ HTTP มาคุยกับ Django ผ่าน Blog REST API ที่
สร้างไว้ตั้งแต่ Part 044 (`ModelViewSet` + `Router`) Part นี้ทำสิ่งเดียวกัน
ทุกประการ เพียงแค่เปลี่ยนฝั่ง frontend จาก React เป็น **Vue 3** เท่านั้น
สถาปัตยกรรมภาพรวมจึงเหมือนเดิมเป๊ะ:

```
┌───────────────────────┐        HTTP (JSON)        ┌───────────────────────┐
│   Vue 3 App (Vite)     │ ─────────────────────────> │   Django REST API     │
│   http://localhost:5173│ <───────────────────────── │  http://localhost:8000│
└───────────────────────┘                             └───────────────────────┘
        │                                                       │
        │ npm run build → dist/                                 │ manage.py runserver
        ▼                                                       ▼
   Static hosting (Nginx/Vercel/                         PostgreSQL + Django Admin
   Whitenoise) — ขั้นตอนที่ 559
```

Django **ไม่รู้จัก** Vue เลยแม้แต่น้อย มันแค่ตอบ JSON กลับไปให้ client ตัวไหนก็ได้
ที่ยิง request มาถูก endpoint พร้อม header ที่ถูกต้อง — นี่คือประโยชน์ของสถาปัตยกรรม
แบบแยกส่วน (decoupled) ที่ Part 055 วางรากฐานไว้แล้ว

### 551.2 ทำไมต้องใช้ Vite แทน Vue CLI แบบเก่า

Vue เคยมีเครื่องมือสร้างโปรเจกต์ชื่อ **Vue CLI** (ใช้ Webpack) แต่ตั้งแต่ปี 2023
เป็นต้นมา ทีม Vue เปลี่ยนมาแนะนำ **Vite** (สร้างโดย Evan You คนเดียวกับที่สร้าง
Vue) เป็นเครื่องมือมาตรฐานแทนอย่างเป็นทางการ เพราะ:

| คุณสมบัติ | Vue CLI (Webpack) | Vite |
|---|---|---|
| Dev server เริ่มทำงาน | ช้า (bundle ทั้งแอปก่อน) | เร็วมาก (ใช้ ES Modules ดิบ ไม่ bundle ตอน dev) |
| Hot Module Replacement (HMR) | ช้าลงเมื่อโปรเจกต์ใหญ่ขึ้น | เร็วคงที่ไม่ว่าโปรเจกต์จะใหญ่แค่ไหน |
| Production build | Webpack | Rollup (เร็วและ output เล็กกว่า) |
| การดูแลรักษา (Maintenance) | อยู่ในโหมด maintenance เท่านั้น | พัฒนาต่อเนื่องอย่างจริงจัง |
| สถานะปัจจุบัน (2026) | ไม่แนะนำสำหรับโปรเจกต์ใหม่ | มาตรฐานที่ทีม Vue แนะนำอย่างเป็นทางการ |

### 551.3 สร้างโปรเจกต์ Vue 3 ด้วย Vite

รันคำสั่งต่อไปนี้ **แยกโฟลเดอร์ออกจาก Django project เดิมโดยสิ้นเชิง** เหมือนที่
Part 055 ทำกับ React (แนะนำให้วางไว้ข้าง ๆ กันในโฟลเดอร์ workspace เดียวกัน):

```bash
# อยู่นอกโฟลเดอร์ django-mastery-course (โฟลเดอร์ Django จาก Part 001)
npm create vite@latest blog-frontend-vue -- --template vue

cd blog-frontend-vue
npm install

# ติดตั้ง library ที่จะใช้ตลอด Part นี้
npm install axios vue-router@4 pinia
```

`--template vue` บอก Vite ให้สร้างโปรเจกต์ Vue แบบ **Composition API + `<script setup>`**
เป็นค่าเริ่มต้น (ถ้าต้องการ TypeScript ให้ใช้ `--template vue-ts` แทน แต่หลักสูตรนี้
จะใช้ JavaScript ล้วนเพื่อโฟกัสที่แนวคิดของ Vue เอง)

### 551.4 โครงสร้างโฟลเดอร์ที่ได้

```
blog-frontend-vue/
├── index.html
├── package.json
├── vite.config.js
├── .env                      ← สร้างเพิ่มเอง (ขั้นตอนที่ 551.6)
├── public/
└── src/
    ├── main.js
    ├── App.vue
    ├── assets/
    ├── router/               ← สร้างเพิ่มใน ขั้นตอนที่ 553
    │   └── index.js
    ├── stores/               ← สร้างเพิ่มใน ขั้นตอนที่ 554
    │   ├── auth.js
    │   └── posts.js
    ├── composables/          ← สร้างเพิ่มใน ขั้นตอนที่ 556
    │   └── useApi.js
    ├── api/
    │   └── client.js         ← สร้างเพิ่มใน ขั้นตอนที่ 552
    ├── components/           ← สร้างเพิ่มใน ขั้นตอนที่ 557
    │   ├── PostCard.vue
    │   └── CommentList.vue
    └── views/                ← สร้างเพิ่มใน ขั้นตอนที่ 553
        ├── PostListView.vue
        ├── PostDetailView.vue
        └── LoginView.vue
```

โครงสร้างนี้แยกความรับผิดชอบชัดเจน: `api/` คุยกับ Django, `stores/` เก็บ state
ที่แชร์ข้าม component, `composables/` เก็บ logic ที่ใช้ซ้ำได้ (Part นี้จะเจาะลึก
ในขั้นตอนที่ 556), `views/` คือหน้าที่ผูกกับ route, `components/` คือชิ้นส่วน UI
ที่ใช้ซ้ำได้ในหลายหน้า

### 551.5 ตั้งค่า Vite Dev Server ให้ Proxy ไปยัง Django

เพื่อหลีกเลี่ยงปัญหา CORS ตอน develop (Django รันที่ port 8000, Vue รันที่ port
5173) เราตั้งให้ Vite **proxy** คำขอที่ขึ้นต้นด้วย `/api` ไปยัง Django โดยตรง
วิธีนี้ทำให้ browser มองว่า request ทั้งหมดมาจาก origin เดียวกัน (`localhost:5173`)
ไม่ต้องยุ่งกับ CORS header เลยตอน develop:

```javascript
// vite.config.js
import { defineConfig } from 'vite'
import vue from '@vitejs/plugin-vue'

export default defineConfig({
  plugins: [vue()],
  server: {
    port: 5173,
    proxy: {
      '/api': {
        target: 'http://127.0.0.1:8000',
        changeOrigin: true,
      },
    },
  },
})
```

**ข้อควรระวัง**: proxy นี้ทำงาน **เฉพาะตอน `npm run dev`** เท่านั้น เมื่อ build
เป็น production (`npm run build`) ไฟล์ static ที่ได้จะไม่มี dev server คอยทำ proxy
ให้อีกต่อไป จึงยังคงต้องตั้งค่า `django-cors-headers` ที่ฝั่ง Django ให้ถูกต้อง
เหมือนที่ Part 055 ทำไว้ (ทบทวนสั้น ๆ ในขั้นตอนถัดไป) เพื่อรองรับกรณีที่ Vue build
ถูก serve จากคนละ origin กับ Django จริงบน production

### 551.6 ตัวแปรสภาพแวดล้อม (`.env`) สำหรับ Base URL ของ API

```bash
# .env (อยู่ที่ root ของ blog-frontend-vue)
VITE_API_BASE_URL=/api
```

```bash
# .env.production (ใช้ตอน build production ถ้า deploy แยก origin จาก Django จริง)
VITE_API_BASE_URL=https://api.myblog.com/api
```

Vite กำหนดกฎว่า environment variable ที่จะถูก expose เข้าไปในโค้ด client ต้อง
ขึ้นต้นด้วย **`VITE_`** เท่านั้น (ตัวแปรอื่นที่ไม่มี prefix นี้จะไม่ถูกฝังเข้าไป
ใน bundle ด้วยเหตุผลด้านความปลอดภัย — ป้องกันไม่ให้ secret ที่ตั้งใจใช้แค่ฝั่ง
build เล็ดลอดเข้าไปใน JavaScript ที่ browser โหลดได้) เข้าถึงค่านี้ในโค้ดผ่าน
`import.meta.env.VITE_API_BASE_URL`

### 551.7 ทบทวน CORS ฝั่ง Django ให้รองรับ Vue

Django ฝั่งเดิมจาก Part 055 ตั้งค่า `django-cors-headers` ไว้รองรับ React ที่ port
`5173` (Vite ใช้ port เดียวกับที่ React+Vite ใช้ ถ้า React project ของ Part 055
สร้างด้วย Vite เช่นกัน) หรือ `3000` (ถ้าใช้ Create React App) ตรวจสอบและเพิ่ม
origin ของ Vue ให้ครบใน `settings.py`:

```python
# config/settings.py
INSTALLED_APPS = [
    # ...
    'corsheaders',
]

MIDDLEWARE = [
    'corsheaders.middleware.CorsMiddleware',   # ต้องอยู่บนสุดของ MIDDLEWARE
    'django.middleware.common.CommonMiddleware',
    # ...
]

CORS_ALLOWED_ORIGINS = [
    'http://localhost:5173',    # Vue 3 + Vite dev server (Part นี้)
    'http://localhost:5174',    # เผื่อรัน React (Part 055) พร้อมกันคนละ port
]
```

หากรัน dev server ของทั้ง React (Part 055) และ Vue (Part นี้) พร้อมกันในเครื่อง
เดียวกัน Vite จะขยับไปใช้ port ถัดไปอัตโนมัติเมื่อ port เดิมถูกใช้แล้ว (5174, 5175, ...)
ควรเช็ค terminal ตอนรัน `npm run dev` เสมอว่าได้ port อะไร แล้วอัปเดต
`CORS_ALLOWED_ORIGINS` ให้ตรงกัน

---

## ขั้นตอนที่ 552: ดึงข้อมูลจาก Blog Django API มาแสดงใน Vue Component

### 552.1 สร้าง API Client กลางด้วย Axios

เหมือนกับที่ Part 055 สร้าง Axios instance กลางไว้ใช้ทั่วทั้งแอป React เราทำ
สิ่งเดียวกันกับ Vue:

```javascript
// src/api/client.js
import axios from 'axios'

const apiClient = axios.create({
  baseURL: import.meta.env.VITE_API_BASE_URL,
  headers: {
    'Content-Type': 'application/json',
  },
})

export default apiClient
```

การรวม config ไว้ที่จุดเดียวแบบนี้ทำให้ภายหลังเมื่อต้องเพิ่ม JWT interceptor
(ขั้นตอนที่ 555) เราแก้ที่ไฟล์เดียว ไม่ต้องไล่แก้ทุก component ที่เรียก API

### 552.2 เขียนฟังก์ชันเรียก Endpoint ของ Blog API แยกตามโดเมน

```javascript
// src/api/posts.js
import apiClient from './client'

export function fetchPosts(params = {}) {
  return apiClient.get('/posts/', { params })
}

export function fetchPostBySlug(slug) {
  return apiClient.get(`/posts/${slug}/`)
}

export function fetchCommentsForPost(postId) {
  return apiClient.get(`/posts/${postId}/comments/`)
}

export function createComment(postId, content) {
  return apiClient.post(`/posts/${postId}/comments/`, { content })
}
```

```javascript
// src/api/categories.js
import apiClient from './client'

export function fetchCategories() {
  return apiClient.get('/categories/')
}
```

การแยกไฟล์ตามโดเมน (`posts.js`, `categories.js`) แทนที่จะยัดทุกอย่างไว้ใน
`client.js` ไฟล์เดียว ช่วยให้โค้ดอ่านง่ายขึ้นเมื่อ API มีหลาย resource เหมือนที่
Blog API มี `posts`, `categories`, `comments` จาก Part 044

### 552.3 Component แรก: `PostListView.vue` ด้วย Composition API

```vue
<!-- src/views/PostListView.vue -->
<script setup>
import { ref, onMounted } from 'vue'
import { fetchPosts } from '../api/posts'

const posts = ref([])
const isLoading = ref(true)
const error = ref(null)

async function loadPosts() {
  isLoading.value = true
  error.value = null
  try {
    const response = await fetchPosts({ is_published: true })
    posts.value = response.data.results ?? response.data
  } catch (err) {
    error.value = 'ไม่สามารถโหลดรายการบทความได้ กรุณาลองใหม่อีกครั้ง'
    console.error(err)
  } finally {
    isLoading.value = false
  }
}

// onMounted คือ Lifecycle Hook ที่ทำงานหลัง component ถูก render ลง DOM
// ครั้งแรก — เทียบเท่ากับ useEffect(() => {...}, []) ใน React ที่เรียนใน Part 055
onMounted(loadPosts)
</script>

<template>
  <div class="post-list">
    <h1>บทความทั้งหมด</h1>

    <p v-if="isLoading">กำลังโหลด...</p>
    <p v-else-if="error" class="error">{{ error }}</p>

    <ul v-else>
      <li v-for="post in posts" :key="post.id">
        <router-link :to="`/posts/${post.slug}`">{{ post.title }}</router-link>
        <span class="category">{{ post.category_name }}</span>
      </li>
    </ul>

    <p v-if="!isLoading && !error && posts.length === 0">ยังไม่มีบทความที่เผยแพร่</p>
  </div>
</template>

<style scoped>
.post-list ul { list-style: none; padding: 0; }
.post-list li { display: flex; justify-content: space-between; padding: 0.5rem 0; border-bottom: 1px solid #eee; }
.category { color: #888; font-size: 0.85rem; }
.error { color: #c0392b; }
</style>
```

### 552.4 กลไก Reactivity เบื้องหลัง `ref()`

`ref()` สร้าง **reactive reference** — object ห่อค่าดั้งเดิม (primitive) ไว้ใน
`{ value: ... }` เพื่อให้ Vue ตรวจจับการเปลี่ยนแปลงได้ (JavaScript primitive
ธรรมดาอย่าง string/number ไม่มีกลไกให้ track การเปลี่ยนแปลงได้ด้วยตัวเอง) เมื่อ
เข้าถึงค่าจาก `<script setup>` ต้องเติม `.value` เสมอ (`posts.value = [...]`)
แต่ใน `<template>` Vue จะ **unwrap ให้อัตโนมัติ** จึงเขียน `{{ post.title }}`
ตรง ๆ ได้โดยไม่ต้องเขียน `.value`

| การกระทำ | ใน `<script setup>` | ใน `<template>` |
|---|---|---|
| อ่านค่า | `posts.value` | `posts` (unwrap อัตโนมัติ) |
| เขียนค่าใหม่ | `posts.value = [...]` | ไม่สามารถเขียนตรง ๆ ใน template ได้ |
| เข้าถึง property ของ array/object ข้างใน | `posts.value[0].title` | `posts[0].title` |

### 552.5 `v-if`/`v-else`/`v-for` เทียบกับ JSX Conditional ของ React

ผู้ที่มาจาก Part 055 (React) จะสังเกตความต่างที่ชัดเจนที่สุดตรงนี้: React ใช้
JavaScript expression ล้วน (`{condition && <div>...</div>}`) ผสมอยู่ใน JSX ส่วน
Vue ใช้ **directive พิเศษที่เขียนเป็น HTML attribute** (`v-if`, `v-for`, `v-show`)
ซึ่งเป็นปรัชญาการออกแบบที่ต่างกันคนละแนว (เจาะลึกในขั้นตอนที่ 558):

```
React (Part 055)                          Vue (Part นี้)
─────────────────────                     ─────────────────────
{isLoading && <p>กำลังโหลด...</p>}         <p v-if="isLoading">กำลังโหลด...</p>
{posts.map(p => <li key={p.id}>...)}      <li v-for="p in posts" :key="p.id">...
{error ? <p>{error}</p> : null}           <p v-else-if="error">{{ error }}</p>
```

---

## ขั้นตอนที่ 553: Vue Router สำหรับ navigation

### 553.1 ทำไมต้องมี Router แยกต่างหาก

Vue core ไม่มีระบบ routing มาให้ในตัว (ต่างจาก Django ที่มี `urls.py` เป็นส่วน
หนึ่งของ framework) ทีม Vue จึงแยก **Vue Router** เป็น official library ต่างหาก
(คล้ายกับ `react-router-dom` ที่ Part 055 ใช้กับ React) หน้าที่ของมันคือแปลง URL
path ในเบราว์เซอร์ให้ตรงกับ component ที่ควรแสดง โดยไม่ต้อง reload หน้าทั้งหมด
(Single Page Application)

### 553.2 นิยาม Route ทั้งหมดของแอป

```javascript
// src/router/index.js
import { createRouter, createWebHistory } from 'vue-router'
import PostListView from '../views/PostListView.vue'
import PostDetailView from '../views/PostDetailView.vue'
import LoginView from '../views/LoginView.vue'
import { useAuthStore } from '../stores/auth'

const routes = [
  { path: '/', name: 'post-list', component: PostListView },
  { path: '/posts/:slug', name: 'post-detail', component: PostDetailView, props: true },
  { path: '/login', name: 'login', component: LoginView },
  {
    path: '/dashboard',
    name: 'dashboard',
    component: () => import('../views/DashboardView.vue'), // lazy-loaded
    meta: { requiresAuth: true },
  },
]

const router = createRouter({
  history: createWebHistory(),
  routes,
})

// Navigation Guard: ทำงานก่อนทุกครั้งที่มีการเปลี่ยน route
router.beforeEach((to, from, next) => {
  const authStore = useAuthStore()
  if (to.meta.requiresAuth && !authStore.isAuthenticated) {
    next({ name: 'login', query: { redirect: to.fullPath } })
  } else {
    next()
  }
})

export default router
```

```javascript
// src/main.js
import { createApp } from 'vue'
import { createPinia } from 'pinia'
import App from './App.vue'
import router from './router'

const app = createApp(App)

app.use(createPinia())   // ต้องติดตั้งก่อน router เพราะ router guard เรียกใช้ store
app.use(router)
app.mount('#app')
```

### 553.3 `<router-view>` และ `<router-link>` ใน `App.vue`

```vue
<!-- src/App.vue -->
<script setup>
import { useAuthStore } from './stores/auth'

const authStore = useAuthStore()
</script>

<template>
  <nav>
    <router-link to="/">หน้าแรก</router-link>
    <router-link v-if="!authStore.isAuthenticated" to="/login">เข้าสู่ระบบ</router-link>
    <button v-else @click="authStore.logout">ออกจากระบบ ({{ authStore.username }})</button>
  </nav>

  <!-- router-view คือจุดที่ Vue Router จะเรนเดอร์ component ตาม route ปัจจุบัน -->
  <router-view />
</template>
```

`<router-view>` ทำหน้าที่เหมือน `<Outlet />` ของ `react-router-dom` ที่ Part 055
ใช้ ส่วน `<router-link>` เทียบเท่ากับ `<Link>` ของ React Router — ทั้งคู่ render
เป็น `<a>` tag แต่ดัก `click` event เพื่อเปลี่ยนหน้าโดยไม่ reload browser จริง

### 553.4 อ่าน Route Parameter ด้วย `useRoute()` และเปลี่ยนหน้าด้วย `useRouter()`

```vue
<!-- src/views/PostDetailView.vue -->
<script setup>
import { ref, onMounted, watch } from 'vue'
import { useRoute, useRouter } from 'vue-router'
import { fetchPostBySlug } from '../api/posts'

const props = defineProps({ slug: String })  // มาจาก routes props: true (553.2)
const route = useRoute()
const router = useRouter()

const post = ref(null)
const notFound = ref(false)

async function loadPost(slug) {
  notFound.value = false
  try {
    const response = await fetchPostBySlug(slug)
    post.value = response.data
  } catch (err) {
    if (err.response?.status === 404) notFound.value = true
  }
}

onMounted(() => loadPost(props.slug))

// watch เฝ้าดู route param slug เผื่อผู้ใช้กดลิงก์ไปโพสต์อื่นจากหน้าเดิม
// (component ถูก reuse ไม่ถูก unmount/mount ใหม่ — จุดที่มือใหม่ Vue มักพลาด)
watch(() => route.params.slug, (newSlug) => {
  if (newSlug) loadPost(newSlug)
})

function goBack() {
  router.push({ name: 'post-list' })
}
</script>

<template>
  <div v-if="notFound">ไม่พบบทความนี้</div>
  <article v-else-if="post">
    <h1>{{ post.title }}</h1>
    <p>{{ post.content }}</p>
    <button @click="goBack">กลับไปหน้ารายการ</button>
  </article>
</template>
```

### 553.5 ตารางเทียบ API หลักของ Vue Router กับ React Router (Part 055)

| ความต้องการ | Vue Router 4 | React Router 6 (Part 055) |
|---|---|---|
| จุด render ตาม route | `<router-view />` | `<Outlet />` |
| ลิงก์เปลี่ยนหน้า | `<router-link to="...">` | `<Link to="...">` |
| อ่าน param จาก URL | `useRoute().params` | `useParams()` |
| เปลี่ยนหน้าด้วยโค้ด | `useRouter().push(...)` | `useNavigate()()` |
| ป้องกัน route ที่ต้อง login | `router.beforeEach()` (global guard) | Wrapper component ที่เช็คแล้ว `<Navigate>` |
| Lazy-loaded route | `component: () => import('...')` | `React.lazy(() => import('...'))` |

---

## ขั้นตอนที่ 554: Pinia สำหรับ State Management

### 554.1 ทำไมต้องมี State Management แยกจาก Component State

`ref()`/`reactive()` ใน component เดียวใช้ได้ดีสำหรับ state ที่อยู่แค่ในหน้านั้น
แต่ state บางอย่าง เช่น **ข้อมูล user ที่ login อยู่** หรือ **รายการบทความที่โหลด
มาแล้ว** จำเป็นต้องใช้ร่วมกันข้ามหลาย component ที่ไม่มีความสัมพันธ์ parent-child
โดยตรง (เช่น `Navbar` ต้องรู้ว่า user login อยู่ไหม พร้อมกับ `Dashboard` ที่ต้องรู้
เหมือนกัน) **Pinia** คือ library จัดการ state กลางที่เป็นทางการของ Vue (สืบทอด
ตำแหน่งจาก Vuex ที่เคยเป็นมาตรฐานเดิม) บทบาทเทียบเท่ากับ Redux Toolkit หรือ
Zustand ที่มักใช้คู่กับ React

### 554.2 สร้าง Store แรก: `posts.js`

```javascript
// src/stores/posts.js
import { defineStore } from 'pinia'
import { ref } from 'vue'
import { fetchPosts as apiFetchPosts } from '../api/posts'

export const usePostsStore = defineStore('posts', () => {
  // นี่คือ "Setup Store" syntax — เขียนเหมือน composable function
  // ref() ที่ return ออกไปกลายเป็น state, function ที่ return ออกไปกลายเป็น action
  const posts = ref([])
  const isLoading = ref(false)
  const lastFetchedAt = ref(null)

  async function fetchAll(params = {}) {
    isLoading.value = true
    try {
      const response = await apiFetchPosts(params)
      posts.value = response.data.results ?? response.data
      lastFetchedAt.value = new Date()
    } finally {
      isLoading.value = false
    }
  }

  function upsertPost(updatedPost) {
    const index = posts.value.findIndex((p) => p.id === updatedPost.id)
    if (index !== -1) posts.value[index] = updatedPost
    else posts.value.unshift(updatedPost)
  }

  return { posts, isLoading, lastFetchedAt, fetchAll, upsertPost }
})
```

### 554.3 สร้าง Store สำหรับ Authentication: `auth.js`

```javascript
// src/stores/auth.js
import { defineStore } from 'pinia'
import { ref, computed } from 'vue'
import apiClient from '../api/client'

export const useAuthStore = defineStore('auth', () => {
  const accessToken = ref(localStorage.getItem('access_token'))
  const refreshToken = ref(localStorage.getItem('refresh_token'))
  const username = ref(localStorage.getItem('username'))

  const isAuthenticated = computed(() => !!accessToken.value)

  async function login(usernameInput, password) {
    const response = await apiClient.post('/token/', {
      username: usernameInput,
      password,
    })
    accessToken.value = response.data.access
    refreshToken.value = response.data.refresh
    username.value = usernameInput

    localStorage.setItem('access_token', accessToken.value)
    localStorage.setItem('refresh_token', refreshToken.value)
    localStorage.setItem('username', usernameInput)
  }

  function logout() {
    accessToken.value = null
    refreshToken.value = null
    username.value = null
    localStorage.removeItem('access_token')
    localStorage.removeItem('refresh_token')
    localStorage.removeItem('username')
  }

  function setAccessToken(newToken) {
    accessToken.value = newToken
    localStorage.setItem('access_token', newToken)
  }

  return {
    accessToken, refreshToken, username,
    isAuthenticated, login, logout, setAccessToken,
  }
})
```

### 554.4 ใช้ Store ใน Component

```vue
<!-- src/views/PostListView.vue (ปรับให้ใช้ store แทนการ fetch เองใน component) -->
<script setup>
import { onMounted } from 'vue'
import { usePostsStore } from '../stores/posts'
import { storeToRefs } from 'pinia'

const postsStore = usePostsStore()
// storeToRefs รักษาความเป็น reactive ไว้ตอน destructure — ถ้า destructure
// ตรง ๆ (const { posts } = postsStore) จะได้ค่า snapshot ที่ไม่ reactive อีกต่อไป
const { posts, isLoading } = storeToRefs(postsStore)

onMounted(() => postsStore.fetchAll({ is_published: true }))
</script>

<template>
  <p v-if="isLoading">กำลังโหลด...</p>
  <ul v-else>
    <li v-for="post in posts" :key="post.id">{{ post.title }}</li>
  </ul>
</template>
```

**ข้อควรระวังที่สำคัญที่สุดของ Pinia**: การ `const { posts, isLoading } = postsStore`
ตรง ๆ โดยไม่ผ่าน `storeToRefs()` จะทำให้ตัวแปรที่ได้ **หลุด reactivity** ทันที
เพราะเป็นการ copy ค่า primitive ออกมา ณ เวลานั้น ไม่ใช่การอ้างอิงถึง reactive
reference อีกต่อไป — ฟังก์ชัน `postsStore.fetchAll(...)` (actions) ไม่ต้องผ่าน
`storeToRefs()` เพราะ function ไม่มีปัญหาเรื่อง reactivity นี้

### 554.5 ตารางเทียบ Pinia กับแนวทาง State Management ของ React (Part 055)

| ความต้องการ | Pinia (Vue) | React (Part 055 มักใช้) |
|---|---|---|
| นิยาม store | `defineStore('name', () => {...})` | `createSlice()` (Redux Toolkit) หรือ `create()` (Zustand) |
| อ่าน state ใน component | `storeToRefs(store)` | `useSelector()` หรือเรียก hook ตรง ๆ |
| เปลี่ยน state | เรียก action function ตรง ๆ | `dispatch(action())` หรือเรียก setter ตรง ๆ |
| DevTools | Vue DevTools (in ตัว) | Redux DevTools Extension |
| Boilerplate | น้อยมาก (คล้าย composable) | ปานกลาง-มาก (Redux) / น้อย (Zustand) |

---

## ขั้นตอนที่ 555: การผสาน Vue กับ Django Authentication (JWT)

### 555.1 ทบทวน JWT Flow จาก Part 046 และ Part 055

Part 046 สร้าง endpoint `/api/token/` (รับ username/password คืน access + refresh
token) และ `/api/token/refresh/` (แลก refresh token เป็น access token ใหม่) ไว้ที่
ฝั่ง Django แล้ว Part 055 implement flow นี้ด้วย React (เก็บ token ผ่าน
`localStorage` + Axios interceptor) Part นี้ทำ flow **เดียวกันทุกประการ** เพียง
เปลี่ยนมาเขียนด้วย Vue + Pinia (store `auth.js` จากขั้นตอนที่ 554.3 สร้าง
`login()`/`logout()` ไว้แล้ว) สิ่งที่เหลือคือการต่อ interceptor ให้แนบ token
อัตโนมัติทุก request และ refresh อัตโนมัติเมื่อ token หมดอายุ

```
┌────────┐  1. POST /api/token/ (username+password)  ┌──────────────┐
│ Vue App│ ───────────────────────────────────────>  │ Django + JWT │
│(Pinia) │  <───────────────────────────────────────  │ (Part 046)   │
└────────┘  2. { access, refresh }                    └──────────────┘
     │
     │ 3. เก็บ access+refresh ใน localStorage (auth.js store)
     ▼
┌────────┐  4. GET /api/posts/mine/                    ┌──────────────┐
│ Vue App│    Authorization: Bearer <access>       ──> │ Django       │
│(Axios  │  <───────────────────────────────────────   │              │
│interceptor)│ 5a. 200 OK (ถ้า token ยังไม่หมดอายุ)     └──────────────┘
└────────┘  5b. 401 (ถ้า token หมดอายุ) → ไป step 6

     6. POST /api/token/refresh/ { refresh }  → ได้ access ใหม่ → ยิง request เดิมซ้ำ
```

### 555.2 Axios Interceptor: แนบ Token อัตโนมัติทุก Request

```javascript
// src/api/client.js (อัปเดตจากขั้นตอนที่ 552.1)
import axios from 'axios'
import { useAuthStore } from '../stores/auth'

const apiClient = axios.create({
  baseURL: import.meta.env.VITE_API_BASE_URL,
  headers: { 'Content-Type': 'application/json' },
})

apiClient.interceptors.request.use((config) => {
  const authStore = useAuthStore()
  if (authStore.accessToken) {
    config.headers.Authorization = `Bearer ${authStore.accessToken}`
  }
  return config
})

export default apiClient
```

**ข้อควรระวัง**: `useAuthStore()` เรียกได้ตรงนี้เพราะ Pinia ถูก `app.use(createPinia())`
ไปแล้วก่อนที่ component แรกจะ mount (ขั้นตอนที่ 553.2) แต่ต้อง **เรียกภายใน
callback ของ interceptor เท่านั้น** ห้ามเรียกที่ระดับบนสุดของไฟล์ (module scope)
เพราะตอนไฟล์นี้ถูก import ครั้งแรก Pinia อาจยังไม่ถูกติดตั้งบน app instance

### 555.3 Axios Interceptor: Refresh Token อัตโนมัติเมื่อเจอ 401

```javascript
// src/api/client.js (ต่อจาก 555.2)
let isRefreshing = false
let pendingRequests = []

function resolvePendingRequests(newToken) {
  pendingRequests.forEach((callback) => callback(newToken))
  pendingRequests = []
}

apiClient.interceptors.response.use(
  (response) => response,
  async (error) => {
    const authStore = useAuthStore()
    const originalRequest = error.config

    if (error.response?.status !== 401 || originalRequest._retry) {
      return Promise.reject(error)
    }

    if (!authStore.refreshToken) {
      authStore.logout()
      return Promise.reject(error)
    }

    if (isRefreshing) {
      // ถ้ามี request อื่นกำลัง refresh อยู่แล้ว ให้รอคิว แทนที่จะยิง
      // /api/token/refresh/ ซ้ำซ้อนหลายครั้งพร้อมกัน
      return new Promise((resolve) => {
        pendingRequests.push((newToken) => {
          originalRequest.headers.Authorization = `Bearer ${newToken}`
          resolve(apiClient(originalRequest))
        })
      })
    }

    originalRequest._retry = true
    isRefreshing = true
    try {
      const response = await axios.post(
        `${import.meta.env.VITE_API_BASE_URL}/token/refresh/`,
        { refresh: authStore.refreshToken },
      )
      authStore.setAccessToken(response.data.access)
      resolvePendingRequests(response.data.access)
      originalRequest.headers.Authorization = `Bearer ${response.data.access}`
      return apiClient(originalRequest)
    } catch (refreshError) {
      authStore.logout()
      return Promise.reject(refreshError)
    } finally {
      isRefreshing = false
    }
  },
)

export default apiClient
```

โครงสร้างนี้เหมือนกับ interceptor ที่ Part 055 เขียนไว้สำหรับ React **แทบทุก
บรรทัด** เพราะ Axios เป็น library เดียวกัน ไม่เกี่ยวกับ framework ฝั่ง UI เลย —
นี่คือตัวอย่างที่ชัดเจนว่า "การคุยกับ API" กับ "การ render UI" เป็นคนละชั้นกัน
โดยสิ้นเชิงในสถาปัตยกรรมแบบ decoupled

### 555.4 `LoginView.vue`

```vue
<!-- src/views/LoginView.vue -->
<script setup>
import { ref } from 'vue'
import { useRouter, useRoute } from 'vue-router'
import { useAuthStore } from '../stores/auth'

const username = ref('')
const password = ref('')
const errorMessage = ref('')
const isSubmitting = ref(false)

const authStore = useAuthStore()
const router = useRouter()
const route = useRoute()

async function handleSubmit() {
  errorMessage.value = ''
  isSubmitting.value = true
  try {
    await authStore.login(username.value, password.value)
    const redirectTo = route.query.redirect || '/dashboard'
    router.push(redirectTo)
  } catch (err) {
    errorMessage.value = 'Username หรือ Password ไม่ถูกต้อง'
  } finally {
    isSubmitting.value = false
  }
}
</script>

<template>
  <form @submit.prevent="handleSubmit">
    <h1>เข้าสู่ระบบ</h1>
    <p v-if="errorMessage" class="error">{{ errorMessage }}</p>

    <label>
      Username
      <input v-model="username" type="text" required autocomplete="username" />
    </label>

    <label>
      Password
      <input v-model="password" type="password" required autocomplete="current-password" />
    </label>

    <button type="submit" :disabled="isSubmitting">
      {{ isSubmitting ? 'กำลังเข้าสู่ระบบ...' : 'เข้าสู่ระบบ' }}
    </button>
  </form>
</template>
```

`v-model="username"` คือ **two-way binding** — Vue จัดการ `:value` +
`@input` ให้อัตโนมัติในบรรทัดเดียว เทียบเท่ากับการเขียน `value={username}
onChange={(e) => setUsername(e.target.value)}` ด้วยมือใน React (Part 055)

### 555.5 ตารางเทียบการจัดเก็บ Token: localStorage vs httpOnly Cookie

Part 055 น่าจะพูดถึงข้อถกเถียงนี้ไปแล้วสำหรับ React ประเด็นเดียวกันนี้ใช้ได้กับ
Vue **ทุกประการ** เพราะเป็นเรื่องของ browser security ไม่ใช่เรื่องของ framework:

| แนวทาง | ข้อดี | ข้อเสีย |
|---|---|---|
| `localStorage` (ตัวอย่างใน Part นี้) | เขียนง่าย, เข้าถึงจาก JS ได้ตรง ๆ, เหมาะกับ demo/เรียนรู้ | เสี่ยงต่อ XSS (ถ้ามี script แปลกปลอมรันได้ จะขโมย token ไปได้ทันที) |
| httpOnly Cookie (Django ตั้งให้) | JavaScript อ่านค่าไม่ได้เลย ป้องกัน XSS ขโมย token โดยตรง | ต้องระวัง CSRF เพิ่ม (ต้องใช้ `SameSite` + CSRF token คู่กัน), Django ต้องเปลี่ยนมาตั้ง cookie เอง แทนที่จะคืน JSON |

**คำแนะนำระดับมืออาชีพ**: สำหรับโปรเจกต์ที่ต้องการความปลอดภัยสูงสุด ให้ Django
ตั้ง refresh token เป็น httpOnly cookie (`Set-Cookie` พร้อม `HttpOnly`,
`Secure`, `SameSite=Strict`) แล้วเก็บแค่ access token ที่อายุสั้นไว้ใน memory
(ตัวแปร JavaScript ธรรมดา ไม่ใช่ `localStorage`) ฝั่ง client — Part นี้ใช้
`localStorage` เพื่อความง่ายในการสอน แต่ในงานจริงควรพิจารณาแนวทาง httpOnly
cookie อย่างจริงจัง โดยเฉพาะแอปที่จัดการข้อมูลอ่อนไหว

---

## ขั้นตอนที่ 556: Composition API patterns (`ref`, `reactive`, `computed`, composables)

### 556.1 `ref()` vs `reactive()`: เลือกใช้เมื่อไหร่

Vue 3 มี API สองแบบสำหรับสร้าง reactive state ที่มือใหม่มักสับสนว่าต่างกันตรงไหน:

```javascript
import { ref, reactive } from 'vue'

// ref() — ห่อค่าเดี่ยว (primitive หรือ object ก็ได้) ไว้ใน { value: ... }
const count = ref(0)
count.value++

// reactive() — ทำงานกับ object/array โดยตรง ไม่ต้องมี .value
const state = reactive({ count: 0, name: 'เอิร์ธ' })
state.count++
```

| คุณสมบัติ | `ref()` | `reactive()` |
|---|---|---|
| ใช้กับ primitive (string, number, boolean) ได้ไหม | ✅ ได้ | ❌ ไม่ได้ (primitive ไม่ใช่ reference type) |
| ต้องเติม `.value` ใน `<script>` | ✅ ต้องเติม | ❌ ไม่ต้อง |
| Destructure แล้วยังคง reactive ไหม | ✅ ได้ (เพราะ `.value` คือ property เดียว) | ❌ เสีย reactivity ทันที (ต้องใช้ `toRefs()` ช่วย) |
| แทนที่ทั้ง object/array ได้ตรง ๆ ไหม | ✅ ได้ (`arr.value = newArr`) | ❌ ไม่ได้ (ต้อง mutate เช่น `Object.assign()`) |
| คำแนะนำของทีม Vue | ใช้เป็นค่าเริ่มต้นสำหรับทุกกรณี | ใช้เฉพาะเมื่อรู้ชัดว่า object นั้นจะไม่ถูก destructure หรือแทนที่ทั้งก้อน |

**คำแนะนำระดับมืออาชีพ (ตรงกับแนวทางที่ทีม Vue core แนะนำในเอกสารทางการ)**: ใช้
`ref()` เป็นค่าเริ่มต้นเกือบทุกกรณี เพราะปลอดภัยกว่าเรื่อง destructure และใช้ได้
กับทุกชนิดข้อมูล ส่วน `reactive()` เก็บไว้ใช้เฉพาะกรณีที่แน่ใจว่า state นั้นเป็น
object ก้อนใหญ่ที่จะไม่ถูกแยกเป็นตัวแปรย่อย

### 556.2 `computed()`: ค่าที่คำนวณจาก state อื่นและ cache อัตโนมัติ

```vue
<script setup>
import { ref, computed } from 'vue'

const posts = ref([
  { title: 'A', isPublished: true },
  { title: 'B', isPublished: false },
])

// computed() คืน ComputedRef ที่คำนวณใหม่เฉพาะเมื่อ dependency (posts) เปลี่ยนจริง
// ต่างจากการเขียนเป็น function ธรรมดาที่คำนวณใหม่ทุกครั้งที่ template re-render
const publishedCount = computed(() =>
  posts.value.filter((p) => p.isPublished).length,
)

const summaryText = computed(() =>
  `เผยแพร่แล้ว ${publishedCount.value} จากทั้งหมด ${posts.value.length} บทความ`,
)
</script>

<template>
  <p>{{ summaryText }}</p>
</template>
```

`computed()` เทียบเท่ากับ `useMemo()` ของ React (Part 055) แต่ syntax กระชับกว่า
เพราะไม่ต้องระบุ dependency array เอง — Vue ตรวจจับ dependency ให้อัตโนมัติจาก
reactive property ที่ถูกเข้าถึงภายใน getter function

### 556.3 Composable: ดึง Logic ที่ใช้ซ้ำออกมาเป็นฟังก์ชัน

**Composable** คือฟังก์ชัน JavaScript ธรรมดาที่ใช้ Composition API ข้างในและคืน
reactive state ออกมา — เทียบเท่ากับ **Custom Hook** ของ React (`useSomething()`)
ที่ Part 055 แนะนำไปแล้ว หลักการเดียวกันทุกประการ: ดึง logic ที่ซ้ำกันในหลาย
component ออกมาเป็นฟังก์ชันเดียว

```javascript
// src/composables/useApi.js
import { ref } from 'vue'

/**
 * Composable ทั่วไปสำหรับเรียก API async function ใด ๆ พร้อมจัดการ
 * loading/error state ให้อัตโนมัติ — ใช้ซ้ำได้กับทุก endpoint
 */
export function useApi(apiFunction) {
  const data = ref(null)
  const error = ref(null)
  const isLoading = ref(false)

  async function execute(...args) {
    isLoading.value = true
    error.value = null
    try {
      const response = await apiFunction(...args)
      data.value = response.data
      return response.data
    } catch (err) {
      error.value = err
      throw err
    } finally {
      isLoading.value = false
    }
  }

  return { data, error, isLoading, execute }
}
```

```javascript
// src/composables/usePosts.js
import { fetchPosts, fetchPostBySlug } from '../api/posts'
import { useApi } from './useApi'

export function usePostList() {
  const { data: posts, error, isLoading, execute } = useApi(fetchPosts)
  return { posts, error, isLoading, loadPosts: execute }
}

export function usePostDetail() {
  const { data: post, error, isLoading, execute } = useApi(fetchPostBySlug)
  return { post, error, isLoading, loadPost: execute }
}
```

### 556.4 ใช้ Composable ใน Component: โค้ดสั้นลงอย่างเห็นได้ชัด

```vue
<!-- src/views/PostListView.vue (เวอร์ชันที่ใช้ composable แทนการเขียน ref เอง) -->
<script setup>
import { onMounted } from 'vue'
import { usePostList } from '../composables/usePosts'

const { posts, isLoading, error, loadPosts } = usePostList()

onMounted(() => loadPosts({ is_published: true }))
</script>

<template>
  <p v-if="isLoading">กำลังโหลด...</p>
  <p v-else-if="error" class="error">โหลดข้อมูลไม่สำเร็จ</p>
  <ul v-else>
    <li v-for="post in posts" :key="post.id">{{ post.title }}</li>
  </ul>
</template>
```

เทียบกับขั้นตอนที่ 552.3 ที่เขียน `ref()`, `try/catch`, `finally` เองทั้งหมดใน
component โดยตรง เวอร์ชันนี้ดึง logic ซ้ำซ้อนออกไปไว้ที่ composable แล้วนำกลับมา
ใช้ได้กับทุกหน้าที่ต้องเรียก API แบบเดียวกัน — หลักการ **DRY** เดียวกับที่ Django
ใช้กับ Mixin ใน Generic Views (Part 043) และ ViewSet (Part 044)

### 556.5 ตารางเทียบ Composable (Vue) กับ Custom Hook (React)

| แนวคิด | Vue Composable | React Custom Hook |
|---|---|---|
| ชื่อฟังก์ชัน (convention) | `useXxx()` | `useXxx()` |
| เรียกใช้ reactive state ภายใน | `ref()`, `reactive()`, `computed()` | `useState()`, `useMemo()`, `useRef()` |
| ข้อจำกัดเรื่องลำดับการเรียก | ไม่มี (เรียกใน `if` ได้ ตราบใดที่อยู่ใน `setup()`) | ต้องเรียกตามลำดับเดิมทุก render (Rules of Hooks) |
| คืนค่าแบบไหน | Object ของ ref/computed/function | Array (`useState`) หรือ Object แล้วแต่ออกแบบ |
| แชร์ instance เดียวข้าม component ได้ไหม | ไม่ได้โดยตรง (แต่ละ component ได้ instance แยกกัน เว้นแต่ใช้ store) | ไม่ได้เช่นกัน (ต้องใช้ Context/store เหมือนกัน) |

---

## ขั้นตอนที่ 557: สร้าง Reusable Component (`PostCard.vue`, `CommentList.vue`)

### 557.1 `PostCard.vue`: Component รับ `props` และส่ง `emit` กลับ

```vue
<!-- src/components/PostCard.vue -->
<script setup>
defineProps({
  post: {
    type: Object,
    required: true,
  },
})

// defineEmits ประกาศ event ที่ component นี้จะยิงออกไปให้ parent ฟัง
// เทียบเท่ากับการรับ callback prop เช่น onPublishToggle เข้ามาใน React
const emit = defineEmits(['togglePublish'])

function handleToggle() {
  emit('togglePublish', undefined)
}
</script>

<template>
  <article class="post-card">
    <header>
      <h2>
        <router-link :to="`/posts/${post.slug}`">{{ post.title }}</router-link>
      </h2>
      <span class="badge" :class="{ published: post.is_published }">
        {{ post.is_published ? 'เผยแพร่แล้ว' : 'ร่าง' }}
      </span>
    </header>

    <p class="excerpt">{{ post.content.slice(0, 120) }}...</p>

    <footer>
      <span>โดย {{ post.author }}</span>
      <span v-if="post.category_name"> · {{ post.category_name }}</span>
      <!-- slot ชื่อ actions ให้ parent ใส่ปุ่มเพิ่มเติมเองได้ตามบริบท -->
      <slot name="actions">
        <button @click="handleToggle">
          {{ post.is_published ? 'ยกเลิกเผยแพร่' : 'เผยแพร่' }}
        </button>
      </slot>
    </footer>
  </article>
</template>

<style scoped>
.post-card { border: 1px solid #ddd; border-radius: 8px; padding: 1rem; margin-bottom: 1rem; }
.badge { font-size: 0.75rem; padding: 0.2rem 0.5rem; border-radius: 4px; background: #eee; }
.badge.published { background: #d4edda; color: #155724; }
</style>
```

### 557.2 ใช้ `PostCard.vue` พร้อม Custom Slot

```vue
<!-- src/views/PostListView.vue (ใช้ PostCard แทนการเขียน <li> เอง) -->
<script setup>
import { onMounted } from 'vue'
import { usePostsStore } from '../stores/posts'
import { storeToRefs } from 'pinia'
import PostCard from '../components/PostCard.vue'

const postsStore = usePostsStore()
const { posts, isLoading } = storeToRefs(postsStore)

onMounted(() => postsStore.fetchAll())

function handlePublishToggle(post) {
  console.log('toggle publish for', post.slug)
  // เรียก API publish/unpublish จาก Part 044 ที่นี่ แล้ว postsStore.upsertPost(...)
}
</script>

<template>
  <p v-if="isLoading">กำลังโหลด...</p>
  <PostCard
    v-for="post in posts"
    :key="post.id"
    :post="post"
    @toggle-publish="handlePublishToggle(post)"
  />
</template>
```

สังเกตว่า event ที่ประกาศเป็น `togglePublish` (camelCase) ใน `defineEmits`
ตอนใช้งานใน template ของ parent เขียนเป็น `@toggle-publish` (kebab-case) — นี่คือ
convention มาตรฐานของ Vue สำหรับชื่อ event ใน template เสมอ

### 557.3 `CommentList.vue`: Component ที่มีทั้ง `props`, `emit`, และ Form ภายในตัวเอง

```vue
<!-- src/components/CommentList.vue -->
<script setup>
import { ref } from 'vue'
import { fetchCommentsForPost, createComment } from '../api/posts'
import { onMounted } from 'vue'

const props = defineProps({
  postId: { type: Number, required: true },
})

const comments = ref([])
const newCommentText = ref('')
const isSubmitting = ref(false)

async function loadComments() {
  const response = await fetchCommentsForPost(props.postId)
  comments.value = response.data.results ?? response.data
}

async function handleSubmit() {
  if (!newCommentText.value.trim()) return
  isSubmitting.value = true
  try {
    const response = await createComment(props.postId, newCommentText.value)
    comments.value.push(response.data)
    newCommentText.value = ''
  } finally {
    isSubmitting.value = false
  }
}

onMounted(loadComments)
</script>

<template>
  <section class="comments">
    <h3>ความคิดเห็น ({{ comments.length }})</h3>

    <ul>
      <li v-for="comment in comments" :key="comment.id">
        <strong>{{ comment.author }}</strong>: {{ comment.content }}
      </li>
    </ul>

    <form @submit.prevent="handleSubmit">
      <textarea
        v-model="newCommentText"
        placeholder="แสดงความคิดเห็น..."
        rows="3"
      ></textarea>
      <button type="submit" :disabled="isSubmitting">
        {{ isSubmitting ? 'กำลังส่ง...' : 'ส่งความคิดเห็น' }}
      </button>
    </form>
  </section>
</template>
```

### 557.4 ประกอบ `CommentList.vue` เข้ากับ `PostDetailView.vue`

```vue
<!-- src/views/PostDetailView.vue (เพิ่ม CommentList จากขั้นตอนที่ 553.4) -->
<script setup>
import { ref, onMounted } from 'vue'
import { fetchPostBySlug } from '../api/posts'
import CommentList from '../components/CommentList.vue'

const props = defineProps({ slug: String })
const post = ref(null)

onMounted(async () => {
  const response = await fetchPostBySlug(props.slug)
  post.value = response.data
})
</script>

<template>
  <article v-if="post">
    <h1>{{ post.title }}</h1>
    <p>{{ post.content }}</p>

    <!-- post.id มาจาก PostSerializer ของ Part 044 ใช้เป็น postId
         สำหรับ nested route /api/posts/{post_pk}/comments/ -->
    <CommentList :post-id="post.id" />
  </article>
</template>
```

### 557.5 หลักการออกแบบ Reusable Component ที่ควรยึดถือ

| หลักการ | เหตุผล |
|---|---|
| Component รับข้อมูลผ่าน `props` เท่านั้น ไม่ดึง API เองถ้าไม่จำเป็น | ทำให้ component ทดสอบง่าย และใช้ซ้ำได้กับ data source ต่างกัน |
| ใช้ `emit` แจ้ง parent แทนการแก้ state ของ parent ตรง ๆ | รักษาทิศทางข้อมูลแบบ "ลงล่างทางเดียว" (props down, events up) เหมือนหลักการของ React |
| ตั้งชื่อ prop/emit ให้สื่อความหมาย ไม่ผูกกับ implementation ภายใน | เปลี่ยน logic ภายใน component ได้โดยไม่กระทบ parent ที่เรียกใช้ |
| ใช้ `slot` เมื่อ parent ต้องการ customize บางส่วนของ UI | ยืดหยุ่นกว่าการเพิ่ม prop ใหม่ทุกครั้งที่ต้องการ variant ใหม่ |
| แยก component ที่ "ดึงข้อมูลเอง" (`CommentList`) กับที่ "รับข้อมูลมาแสดง" (`PostCard`) ให้ชัดเจน | ง่ายต่อการเทส และรู้ทันทีว่า component ไหนมี side effect |

---

## ขั้นตอนที่ 558: ตารางเปรียบเทียบ Vue vs React สำหรับใช้กับ Django Backend

### 558.1 ตารางเปรียบเทียบภาพรวม

| หัวข้อ | Vue 3 | React (Part 055) |
|---|---|---|
| รูปแบบเขียน UI | Single-File Component (`.vue`: template + script + style ในไฟล์เดียว) | JSX (HTML ผสม JavaScript ในไฟล์ `.jsx`) |
| Reactivity | Proxy-based (`ref`/`reactive`) ตรวจจับการเปลี่ยนแปลงอัตโนมัติ | ต้องเรียก `setState`/`useState` setter เองทุกครั้งเพื่อ trigger re-render |
| Two-way binding | มีในตัว (`v-model`) | ไม่มี ต้องเขียน `value` + `onChange` เอง |
| State management ทางการ | Pinia (official) | ไม่มี "ทางการ" ชัดเจน — ชุมชนใช้ Redux Toolkit, Zustand, Jotai หลากหลาย |
| Routing ทางการ | Vue Router (official) | React Router เป็นที่นิยมสูงสุดแต่ไม่ใช่ "official" ของทีม React |
| Template syntax | HTML-based พร้อม directive (`v-if`, `v-for`) | JavaScript expression ล้วนใน JSX |
| Learning Curve | ต่ำกว่าเล็กน้อยสำหรับคนที่มีพื้นฐาน HTML/CSS ชัดเจน | ต้องคุ้นกับแนวคิด JavaScript-first ของ JSX ก่อน |
| Ecosystem/ตลาดงาน | ใหญ่ (โดยเฉพาะเอเชีย/จีน/ยุโรป) | ใหญ่ที่สุดในโลก (โดยเฉพาะสหรัฐฯ) |
| Performance (runtime) | ดีมาก (Proxy-based reactivity + compiler optimization) | ดีมาก (Virtual DOM + Fiber, ต้อง `memo`/`useMemo` เองในบางเคส) |
| Bundle size (runtime) | เล็กกว่า React เล็กน้อย | ใหญ่กว่า Vue เล็กน้อย |
| Backed by | ทีม/ชุมชน Open Source (Evan You + core team) | Meta (Facebook) |
| ความนิยมในสาย Enterprise ตะวันตก | ปานกลาง | สูงมาก |

### 558.2 ความเห็นเชิง Pros/Cons: เมื่อไหร่ควรเลือก Vue คู่กับ Django

**จุดแข็งของ Vue เมื่อใช้กับ Django**

- Syntax ของ Single-File Component (`<template>` เป็น HTML จริง ๆ) ใกล้เคียงกับ
  Django Template Language ที่คุณคุ้นเคยมาตั้งแต่ Part 001 ทำให้ทีมที่มาจากสาย
  Django/Backend เรียนรู้ Vue ได้เร็วกว่า React ในหลายกรณี เพราะไม่ต้องปรับตัว
  เข้ากับ JSX ที่ผสม logic กับ markup แบบสุดขั้ว
- `v-model` และ two-way binding ในตัว ลดโค้ด boilerplate สำหรับฟอร์มได้มาก
  ซึ่งงาน CRUD ของ Django project ส่วนใหญ่เต็มไปด้วยฟอร์ม
- Pinia + Vue Router เป็น "ทางเลือกทางการ" ที่ชัดเจน ลดเวลาที่ทีมต้องถกเถียงกัน
  ว่าจะเลือก library ตัวไหน (ต่างจาก React ที่ต้องเลือกเองระหว่าง Redux/Zustand
  หลายตัว)

**จุดแข็งของ React เมื่อใช้กับ Django (ทบทวนจาก Part 055)**

- Ecosystem และตลาดแรงงานใหญ่กว่ามาก โดยเฉพาะถ้าต้องจ้างทีมเพิ่มหรือหา
  library เฉพาะทาง (เช่น chart, drag-and-drop, rich text editor) มักมีตัวเลือก
  สำหรับ React มากกว่า
- React Native ใช้โค้ด/แนวคิดเดียวกันสำหรับ mobile app ถ้าโปรเจกต์มีแผนขยายไป
  mobile ในอนาคต การเริ่มด้วย React ทำให้ reuse ความรู้ทีมได้มากกว่า
- Meta ผลักดันและดูแลอย่างต่อเนื่องในระดับองค์กรใหญ่ มีความมั่นใจเรื่อง
  long-term support สูง

**สรุปเชิงความเห็น**: ทั้งสองตัวคุยกับ Django REST API ได้ดีเท่ากันทุกประการ
เพราะ Django ไม่สนใจว่าใครเป็นคนยิง HTTP request มา การเลือกจึงขึ้นอยู่กับ
**ทีมและบริบทของโปรเจกต์** มากกว่าความสามารถทางเทคนิค: ถ้าทีมมาจากสาย Django/
Backend เป็นหลักและอยากได้ syntax ที่ใกล้เคียง Template เดิม Vue มักเรียนรู้ได้
เร็วกว่า แต่ถ้าต้องการ ecosystem ใหญ่ที่สุด หรือมีแผนทำ React Native ควบคู่ไปด้วย
React มักเป็นตัวเลือกที่ปลอดภัยกว่าในระยะยาว

### 558.3 ตารางเปรียบเทียบไวยากรณ์คู่ขนาน (Syntax Side-by-Side)

| งาน | Vue 3 (Composition API) | React (Hooks) |
|---|---|---|
| State พื้นฐาน | `const count = ref(0)` | `const [count, setCount] = useState(0)` |
| อัปเดต state | `count.value++` | `setCount(count + 1)` |
| Side effect ตอน mount | `onMounted(() => {...})` | `useEffect(() => {...}, [])` |
| ค่าคำนวณจาก state อื่น | `computed(() => ...)` | `useMemo(() => ..., [deps])` |
| Conditional render | `<p v-if="cond">...</p>` | `{cond && <p>...</p>}` |
| Loop render | `<li v-for="x in list" :key="x.id">` | `{list.map(x => <li key={x.id}>...)}` |
| ผูก input สองทาง | `<input v-model="text" />` | `<input value={text} onChange={e => setText(e.target.value)} />` |

---

## ขั้นตอนที่ 559: ข้อควรพิจารณาการ Deploy Vue build

### 559.1 ผลลัพธ์ของ `npm run build`

```bash
npm run build
```

Vite จะ compile ทุก `.vue` file, bundle JavaScript, ทำ tree-shaking และ
minify แล้วสร้างโฟลเดอร์ `dist/` ขึ้นมา:

```
dist/
├── index.html
└── assets/
    ├── index-a1b2c3d4.js     ← hash เปลี่ยนทุกครั้งที่โค้ดเปลี่ยน (cache busting)
    ├── index-e5f6g7h8.css
    └── logo-i9j0k1l2.svg
```

ไฟล์ทุกไฟล์ใน `assets/` มี **content hash** ต่อท้ายชื่อไฟล์ ทำให้ตั้งค่า
`Cache-Control: max-age=31536000, immutable` (cache ยาว 1 ปี) กับไฟล์เหล่านี้ได้
อย่างปลอดภัย เพราะถ้าเนื้อหาไฟล์เปลี่ยน ชื่อไฟล์ (และ URL) จะเปลี่ยนตามไปด้วย
เสมอ — browser จะไม่มีวันได้ไฟล์เก่าค้าง cache

### 559.2 ตัวเลือกที่ 1: Serve `dist/` แยกจาก Django โดยสิ้นเชิง

เหมือนที่ Part 055 แนะนำสำหรับ React แนวทางที่ตรงไปตรงมาที่สุดคือ deploy
`dist/` ขึ้น static hosting โดยเฉพาะ (Vercel, Netlify, Cloudflare Pages, AWS S3
+ CloudFront) แล้ว Django API รันแยกต่างหากคนละ domain/subdomain:

```
https://myblog.com          → Vue build (static hosting)
https://api.myblog.com      → Django REST API (Gunicorn + Nginx)
```

ข้อดีคือ scale และ deploy อิสระต่อกัน แต่ต้องตั้งค่า `CORS_ALLOWED_ORIGINS` และ
`ALLOWED_HOSTS` ให้ครบทั้งสอง domain และต้องตั้ง `.env.production` ของ Vue ให้ชี้
`VITE_API_BASE_URL` ไปที่ `https://api.myblog.com/api` (แทนที่จะเป็น path
สัมพัทธ์ `/api` แบบตอน develop)

### 559.3 ตัวเลือกที่ 2: ให้ Django Serve `dist/` ผ่าน WhiteNoise (Single Origin)

หากต้องการ deploy เป็นก้อนเดียว (เหมาะกับโปรเจกต์ขนาดเล็ก-กลางที่ไม่อยากดูแล
infrastructure สองชุด) ให้ก็อปปี้ผลลัพธ์ `dist/` เข้าไปเป็นส่วนหนึ่งของ Django
static files:

```python
# config/settings.py
STATICFILES_DIRS = [
    BASE_DIR / 'vue-dist',   # ก็อปปี้ dist/assets/* มาไว้ที่นี่ตอน build
]
```

```python
# config/urls.py
from django.views.generic import TemplateView

urlpatterns = [
    path('api/', include('blog.api_urls')),
    path('admin/', admin.site.urls),
    # catch-all: ทุก path ที่ไม่ตรง /api/ หรือ /admin/ ให้ Vue Router
    # (client-side routing) เป็นคนจัดการต่อในฝั่ง browser
    re_path(r'^(?!api/|admin/).*$', TemplateView.as_view(template_name='index.html')),
]
```

```bash
# สคริปต์ deploy (ตัวอย่างแนวคิด)
cd blog-frontend-vue && npm run build
cp -r dist/assets/* ../django-mastery-course/vue-dist/
cp dist/index.html ../django-mastery-course/templates/index.html
cd ../django-mastery-course && python manage.py collectstatic --noinput
```

**ข้อควรระวังสำคัญ**: `index.html` ที่ Vite สร้างอ้างอิงไฟล์ asset ด้วย path
สัมบูรณ์ (`/assets/index-a1b2c3d4.js`) ต้องตรวจสอบให้ `STATIC_URL` ของ Django
สอดคล้องกัน หรือตั้ง `base` ใน `vite.config.js` ให้ตรงกับ path ที่ Django จะ serve
ไฟล์เหล่านี้จริง มิฉะนั้นหน้าเว็บจะโหลดขึ้นมาแต่ CSS/JS ไม่ทำงานเพราะหา asset
ไม่เจอ (404 เงียบ ๆ ที่ debug ยากมากสำหรับมือใหม่)

### 559.4 ตัวเลือกที่ 3: Docker Multi-Stage Build

```dockerfile
# Dockerfile (แนวทาง multi-stage: build Vue ด้วย Node แล้วส่งต่อให้ Django image)
FROM node:20-alpine AS vue-build
WORKDIR /app/frontend
COPY blog-frontend-vue/package*.json ./
RUN npm ci
COPY blog-frontend-vue/ ./
RUN npm run build

FROM python:3.12-slim AS django-app
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY . .
# คัดลอกผลลัพธ์จาก stage แรกเข้ามาเป็น static files ของ Django
COPY --from=vue-build /app/frontend/dist/assets ./vue-dist
COPY --from=vue-build /app/frontend/dist/index.html ./templates/index.html
RUN python manage.py collectstatic --noinput
CMD ["gunicorn", "config.wsgi:application", "--bind", "0.0.0.0:8000"]
```

แนวทางนี้ทำให้ CI/CD pipeline (Part 088) build ทั้ง frontend และ backend ในคำสั่ง
`docker build` เดียว โดยไม่ต้องมี Node.js ติดตั้งอยู่บน production server จริง
(Node ถูกใช้แค่ตอน build stage แรกเท่านั้น image สุดท้ายมีแค่ Python + ไฟล์
static ที่ build เสร็จแล้ว)

### 559.5 ตารางสรุปตัวเลือกการ Deploy

| ตัวเลือก | ความซับซ้อนของ Infra | เหมาะกับ | ข้อควรระวังหลัก |
|---|---|---|---|
| แยก Static Hosting + Django API คนละ origin | ปานกลาง (ดูแล 2 ระบบ) | ทีมที่มี frontend/backend แยกกันชัดเจน, ต้องการ scale อิสระ | ตั้ง CORS ให้ครบ, จัดการ env var `VITE_API_BASE_URL` ต่างกันตาม environment |
| Django + WhiteNoise serve `dist/` | ต่ำ (ระบบเดียว) | โปรเจกต์เล็ก-กลาง, ทีมเดียวดูแลทั้งหมด | ต้อง sync ขั้นตอน build Vue เข้ากับ deploy script ของ Django เอง |
| Docker Multi-Stage | ปานกลาง-สูง (ต้องรู้ Docker) | ทีมที่มี CI/CD อยู่แล้ว ต้องการ image เดียวจบ | ต้องดูแล Dockerfile ให้ทันสมัยเมื่อ dependency เปลี่ยน |

### 559.6 Environment Variable ตอน Build vs ตอน Runtime — กับดักที่พบบ่อย

จุดที่มือใหม่มักพลาดตอน deploy Vue (ไม่ต่างจาก React ใน Part 055): ตัวแปร
`VITE_*` ทั้งหมดถูก **ฝังเข้าไปในไฟล์ JavaScript ตอน `npm run build`แล้ว** ไม่ใช่
ค่าที่อ่านจาก environment ตอน container รันจริงเหมือนฝั่ง Django (`os.environ`)
ดังนั้นถ้าต้อง deploy image เดียวกันไปหลาย environment (staging, production)
ที่มี API base URL ต่างกัน จะ **ต้อง build แยกกันสำหรับแต่ละ environment**
(หรือใช้เทคนิคขั้นสูงกว่า เช่น inject config ผ่านไฟล์ `config.js` แยกที่โหลด
runtime แทนการ build เข้าไปตรง ๆ) — นี่คือความแตกต่างพื้นฐานที่ต้องเข้าใจก่อน
วางแผน pipeline การ deploy จริง

---

## ขั้นตอนที่ 560: สรุปและแบบฝึกหัด — สร้าง Vue frontend ที่สมบูรณ์สำหรับ Blog API

### 560.1 ประกอบทุกอย่างเข้าด้วยกัน: โครงสร้างไฟล์สุดท้ายของ `blog-frontend-vue`

```
blog-frontend-vue/
├── .env
├── .env.production
├── vite.config.js
├── package.json
└── src/
    ├── main.js
    ├── App.vue
    ├── api/
    │   ├── client.js          # Axios + JWT interceptor (555.2-555.3)
    │   ├── posts.js           # (552.2)
    │   └── categories.js      # (552.2)
    ├── stores/
    │   ├── auth.js            # Pinia auth store (554.3)
    │   └── posts.js           # Pinia posts store (554.2)
    ├── composables/
    │   ├── useApi.js          # (556.3)
    │   └── usePosts.js        # (556.3)
    ├── router/
    │   └── index.js           # (553.2)
    ├── components/
    │   ├── PostCard.vue       # (557.1)
    │   └── CommentList.vue    # (557.3)
    └── views/
        ├── PostListView.vue   # (556.4, 557.2)
        ├── PostDetailView.vue # (557.4)
        ├── LoginView.vue      # (555.4)
        └── DashboardView.vue  # route ที่ต้อง login (553.2)
```

Vue frontend ทั้งระบบตอนนี้คุยกับ Blog API ของ Django (Part 044-046) ได้ครบ
วงจร: list/detail บทความ, comment, login/logout ด้วย JWT พร้อม auto-refresh
token, และมี route ที่ป้องกันด้วย navigation guard — โครงสร้างเดียวกันทุกประการ
กับที่ Part 055 ทำด้วย React เพียงเปลี่ยนภาษาที่ใช้เขียน UI เท่านั้น

### 560.2 สรุปสิ่งที่ได้เรียนรู้ใน Part นี้

- ✅ ตั้งค่าโปรเจกต์ Vue 3 ด้วย Vite พร้อม dev server proxy ไปยัง Django และเข้าใจ
  ว่าทำไม Vite มาแทน Vue CLI
- ✅ เรียก Blog Django API ด้วย Axios มาแสดงผลใน Component ด้วย `ref()`,
  `onMounted()` และเข้าใจกลไก reactivity เบื้องหลัง
- ✅ ตั้งค่า Vue Router สำหรับ navigation พร้อม navigation guard ป้องกัน route
  ที่ต้อง login
- ✅ ใช้ Pinia จัดการ state ที่แชร์ข้าม component (`auth`, `posts` store) และรู้จัก
  `storeToRefs()` เพื่อรักษา reactivity ตอน destructure
- ✅ ผสาน Vue กับ Django JWT Authentication ครบวงจร ทั้ง login, เก็บ token,
  แนบ token อัตโนมัติผ่าน Axios interceptor, และ refresh token อัตโนมัติเมื่อหมดอายุ
- ✅ เข้าใจแนวคิด Composition API เชิงลึก: `ref()` vs `reactive()`, `computed()`,
  และเขียน composable function ที่ใช้ซ้ำได้ (เทียบเท่า Custom Hook ของ React)
- ✅ สร้าง Reusable Component (`PostCard.vue`, `CommentList.vue`) ด้วย `props`,
  `emit`, และ `slot`
- ✅ เปรียบเทียบ Vue กับ React อย่างตรงไปตรงมาทั้งเชิง syntax และเชิงกลยุทธ์การ
  เลือกใช้กับทีม/โปรเจกต์จริง
- ✅ เข้าใจตัวเลือกการ deploy Vue build ทั้ง 3 แบบ และกับดักเรื่อง environment
  variable ที่ถูกฝังเข้า bundle ตอน build

### 560.3 Checklist ก่อนไป Part ถัดไป

- [ ] สร้างโปรเจกต์ Vue 3 + Vite และตั้งค่า proxy ให้คุยกับ Django ได้โดยไม่ติด CORS
- [ ] ดึงรายการบทความจาก `/api/posts/` มาแสดงด้วย `ref()` + `onMounted()` ได้เอง
- [ ] ตั้งค่า Vue Router ให้มีอย่างน้อย 3 route พร้อม navigation guard 1 เส้นทาง
- [ ] สร้าง Pinia store อย่างน้อย 1 ตัว และใช้ `storeToRefs()` ถูกต้องใน component
- [ ] Login ผ่าน `/api/token/`, เก็บ token, และเห็น Authorization header ถูกแนบ
      อัตโนมัติในทุก request ที่ยิงผ่าน Axios instance กลาง
- [ ] อธิบายความแตกต่างระหว่าง `ref()` กับ `reactive()` ได้โดยไม่ต้องเปิดเอกสาร
- [ ] เขียน composable function อย่างน้อย 1 ตัวที่ใช้ซ้ำได้ในหลาย component
- [ ] สร้าง Reusable Component ที่รับ `props` และยิง `emit` กลับไปยัง parent ได้เอง
- [ ] อธิบายได้ว่าทำไม environment variable ของ Vite ต้องขึ้นต้นด้วย `VITE_`
      และทำไมต้อง build แยกกันสำหรับแต่ละ environment

### 560.4 แบบฝึกหัดท้ายบท

**แบบฝึกหัดที่ 1 (พื้นฐาน)**: สร้างหน้า `CategoryListView.vue` ที่ดึงข้อมูลจาก
`/api/categories/` (endpoint จาก Part 044) มาแสดงเป็นรายการ พร้อมเพิ่ม route
`/categories` ใน `router/index.js` และลิงก์ไปหน้านี้จาก navbar ใน `App.vue`

**แบบฝึกหัดที่ 2 (ประยุกต์)**: เพิ่ม custom action `publish`/`unpublish` (จาก
`PostViewSet` ของ Part 044 ขั้นตอนที่ 434) เข้าไปใน `src/api/posts.js` แล้วต่อ
ปุ่ม "เผยแพร่/ยกเลิกเผยแพร่" ใน `PostCard.vue` (ที่วางโครงไว้แล้วในขั้นตอนที่
557.1) ให้ยิง API จริงและอัปเดต state ใน Pinia store ผ่าน `postsStore.upsertPost()`
โดยไม่ต้อง reload หน้าใหม่

**แบบฝึกหัดที่ 3 (State Management)**: สร้าง Pinia store ใหม่ชื่อ `categories.js`
ที่ cache รายการ category ไว้ (ไม่เรียก API ซ้ำถ้าเคยโหลดมาแล้วในรอบ session
เดียวกัน) แล้วเขียน composable `useCategoryOptions()` ที่คืนรายการ category
สำหรับใช้เป็น `<select>` option ในฟอร์มสร้าง/แก้ไขบทความ

**แบบฝึกหัดที่ 4 (ขั้นสูง — เปรียบเทียบกับ Part 055)**: สร้างหน้า `DashboardView.vue`
(ที่ผูกกับ route `requiresAuth: true` จากขั้นตอนที่ 553.2) แสดงเฉพาะบทความของ
ผู้ใช้ที่ login อยู่ (เรียก `@action(detail=False)` ชื่อ `mine` จาก `PostViewSet`
ของ Part 044 ขั้นตอนที่ 440.1) จากนั้นเขียนสรุปเปรียบเทียบ 5-10 บรรทัดว่าถ้า
สร้างหน้าเดียวกันนี้ด้วย React (ตามแนวทาง Part 055) โค้ดส่วนไหนจะสั้นกว่า/ยาวกว่า
กัน และเพราะเหตุใด

### 560.5 คำถามที่พบบ่อย (FAQ)

**Q: ควรเรียนทั้ง Vue และ React ให้ครบทั้งคู่ไหม หรือเลือกทางใดทางหนึ่งพอ?**
A: สำหรับการทำงานจริง เลือกทางใดทางหนึ่งให้ลึกก็เพียงพอแล้ว เพราะหลักการพื้นฐาน
(component, state, props, routing) เหมือนกันเกือบทั้งหมดในทั้งสอง framework
Part 055 และ Part นี้จงใจให้เห็นโค้ดคู่ขนานกัน เพื่อให้คุณเห็นว่าความรู้เรื่อง
Django REST API ที่เรียนมาตั้งแต่ Part 042-050 นั้น **ใช้ได้กับ frontend
framework ตัวไหนก็ได้** — นี่คือคุณค่าที่แท้จริงของสถาปัตยกรรมแบบแยกส่วน
(decoupled architecture)

**Q: `<script setup>` คืออะไร ต่างจาก Options API แบบเดิมของ Vue 2 อย่างไร?**
A: `<script setup>` คือ syntax sugar ของ Composition API ที่ compiler ของ Vue
แปลงให้อัตโนมัติ ทำให้ไม่ต้องเขียน `export default { setup() { return {...} } }`
เอง ตัวแปรและฟังก์ชันที่ประกาศใน `<script setup>` เข้าถึงได้จาก `<template>`
โดยตรงโดยไม่ต้อง `return` ออกมาเอง ส่วน Options API (`data()`, `methods: {}`,
`computed: {}`) ยังใช้งานได้อยู่ใน Vue 3 เพื่อ backward compatibility กับโค้ด
Vue 2 เดิม แต่โปรเจกต์ใหม่ทีม Vue core แนะนำ Composition API + `<script setup>`
เป็นมาตรฐานเริ่มต้น

**Q: จำเป็นต้องใช้ Pinia เสมอไหม หรือใช้ `provide`/`inject` แทนได้?**
A: สำหรับ state ขนาดเล็กที่แชร์แค่ระหว่าง parent กับ descendant ไม่กี่ชั้น
`provide`/`inject` (กลไก dependency injection ในตัวของ Vue) เพียงพอและเบากว่า
Pinia แต่สำหรับ state ระดับแอป (เช่น ข้อมูล user ที่ login, cart, notification)
ที่ต้องเข้าถึงจากหลายจุดที่ไม่มีความสัมพันธ์ parent-child ชัดเจน Pinia เหมาะสม
กว่ามาก เพราะมี DevTools ในตัว, รองรับ SSR, และมีโครงสร้างที่ทีมใหญ่ดูแลร่วมกัน
ได้ง่ายกว่า

**Q: ทำไม Comment endpoint ใน `CommentList.vue` ใช้ `postId` (ตัวเลข) แทน `slug`?**
A: เพราะ nested router ของ `drf-nested-routers` ที่ตั้งไว้ใน Part 044 ขั้นตอนที่
435.4 ใช้ `post_pk` (primary key เชิงตัวเลข) เป็นค่าเริ่มต้นในการสร้าง URL แม้ว่า
`PostViewSet` เองจะตั้ง `lookup_field = 'slug'` สำหรับ endpoint หลักของ `Post`
ก็ตาม — ทั้งสองระบบ lookup ทำงานเป็นอิสระต่อกัน จึงต้องส่ง `post.id` (ไม่ใช่
`post.slug`) ไปให้ endpoint comment เสมอ ตามที่ `PostSerializer` คืนค่า `id`
มาให้อยู่แล้ว

---

## เตรียมตัวสำหรับ Part ถัดไป

**Part 057: WebSockets เบื้องต้นด้วย Django Channels** จะพาก้าวข้ามข้อจำกัดของ
HTTP request-response แบบเดิมที่ Part 055-056 ใช้มาตลอด (client ต้อง fetch เอง
ทุกครั้ง, ไม่มีทาง server ส่งข้อมูลใหม่ไปหา client แบบ real-time ได้) คุณจะได้
เรียนรู้ `Django Channels`, `ASGI`, `Consumer`, `Channel Layer` (ผ่าน Redis)
เพื่อสร้างฟีเจอร์ real-time เช่น แจ้งเตือนเมื่อมี comment ใหม่ทันทีโดยไม่ต้อง
refresh หน้า — และจะได้เห็นวิธีเชื่อมต่อ WebSocket จากทั้งฝั่ง Vue (ต่อยอดจาก
Part นี้) ผ่าน native `WebSocket` API ของเบราว์เซอร์

เตรียมเปิดทั้งโปรเจกต์ Django (`django-mastery-course`) และ Vue
(`blog-frontend-vue`) ของคุณไว้พร้อมกัน เพราะ Part 057 จะแก้ไขทั้งสองฝั่งควบคู่
กันไปตลอดทั้ง Part!
