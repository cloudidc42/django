# หลักสูตร Django ฉบับสมบูรณ์: จากศูนย์สู่ระดับมืออาชีพระดับโลก

หลักสูตรนี้พาผู้เรียนเดินทางตั้งแต่ **ขั้นตอนที่ 1 ถึงขั้นตอนที่ 1000** แบ่งเป็น **100 Part**
(Part ละ 10 ขั้นตอน) ครอบคลุมตั้งแต่พื้นฐานการเขียนโปรแกรมด้วย Python และ Django
ไปจนถึงสถาปัตยกรรมระดับ Enterprise, Microservices, DevOps, Security และแนวทางอาชีพ
Django Developer มืออาชีพระดับโลก

เนื้อหาทุก Part เขียนให้ **ใช้งานได้จริง 100%** มีโค้ดตัวอย่างที่รันได้จริง อธิบายทีละขั้นตอน
พร้อมแบบฝึกหัดและคำแนะนำเชิงลึกแบบมืออาชีพ

> เวอร์ชันอ้างอิงที่ใช้ตลอดหลักสูตร: **Python 3.12+**, **Django 5.x**, **PostgreSQL 16**,
> **Django REST Framework 3.15+**, **Redis 7**, **Docker 26+**

## โครงสร้างหลักสูตร

ไฟล์เนื้อหาทั้งหมดอยู่ในโฟลเดอร์ [`course/`](./course) ตั้งชื่อไฟล์ตามรูปแบบ
`part-XXX-slug.md` เรียงตามลำดับ Part 001 ถึง Part 100

สถานะการเขียน: ดูคอลัมน์ **สถานะ** ในตารางด้านล่าง (✅ = เขียนเสร็จแล้ว, ⏳ = อยู่ระหว่างเขียน,
◻️ = ยังไม่เริ่ม) ไฟล์จะถูกทยอยเพิ่มเข้ามาเรื่อย ๆ จนครบหลักสูตร

---

## Phase 1: รากฐาน Python & Django (Part 1-10 | Step 1-100)

| Part | ชื่อตอน | ขั้นตอน | สถานะ |
|---|---|---|---|
| 001 | บทนำสู่ Django, Web Development และการเตรียมเครื่องมือ | 1-10 | ✅ |
| 002 | ทบทวน Python ที่จำเป็นสำหรับ Django Developer | 11-20 | ✅ |
| 003 | Virtual Environment, pip และการจัดการแพ็กเกจ | 21-30 | ✅ |
| 004 | สร้างโปรเจกต์ Django แรกและทำความเข้าใจโครงสร้าง | 31-40 | ✅ |
| 005 | Django Apps และการจัดระเบียบโค้ด | 41-50 | ✅ |
| 006 | URL Routing และ URLconf เบื้องต้น | 51-60 | ✅ |
| 007 | Views แบบ Function-Based เบื้องต้น | 61-70 | ✅ |
| 008 | Django Template Language เบื้องต้น | 71-80 | ✅ |
| 009 | Static Files และ Media Files เบื้องต้น | 81-90 | ✅ |
| 010 | Django Settings และ Environment Configuration | 91-100 | ✅ |

## Phase 2: Models, ORM และ Admin (Part 11-20 | Step 101-200)

| Part | ชื่อตอน | ขั้นตอน | สถานะ |
|---|---|---|---|
| 011 | Django Models เบื้องต้น: Fields และ Migrations | 101-110 | ✅ |
| 012 | ความสัมพันธ์ระหว่างโมเดล: ForeignKey, OneToOne, ManyToMany | 111-120 | ✅ |
| 013 | Django ORM QuerySet ขั้นสูง | 121-130 | ✅ |
| 014 | Aggregation, Annotation และ Q/F Expressions | 131-140 | ✅ |
| 015 | Model Meta Options, Managers และ Custom QuerySets | 141-150 | ✅ |
| 016 | Database Migrations ขั้นสูงและการจัดการ Schema | 151-160 | ✅ |
| 017 | Django Admin เบื้องต้น: ModelAdmin | 161-170 | ✅ |
| 018 | Django Admin ขั้นสูง: Customization และ Actions | 171-180 | ✅ |
| 019 | Signals และ Django Lifecycle Hooks | 181-190 | ✅ |
| 020 | Multiple Databases และ Database Routing | 191-200 | ✅ |

## Phase 3: Views, Templates, Forms และ CBV (Part 21-30 | Step 201-300)

| Part | ชื่อตอน | ขั้นตอน | สถานะ |
|---|---|---|---|
| 021 | Class-Based Views เบื้องต้น | 201-210 | ✅ |
| 022 | Generic Class-Based Views: ListView, DetailView | 211-220 | ✅ |
| 023 | Generic CBV ขั้นสูง: CreateView, UpdateView, DeleteView | 221-230 | ✅ |
| 024 | Mixins และการสร้าง CBV แบบกำหนดเอง | 231-240 | ✅ |
| 025 | Django Forms เบื้องต้น | 241-250 | ✅ |
| 026 | ModelForms และ Formsets | 251-260 | ✅ |
| 027 | Form Validation ขั้นสูงและ Custom Widgets | 261-270 | ✅ |
| 028 | Template Inheritance และ Template Tags | 271-280 | ✅ |
| 029 | Custom Template Tags และ Filters | 281-290 | ✅ |
| 030 | Context Processors และ Template Best Practices | 291-300 | ✅ |

## Phase 4: Authentication, Users และ Permissions (Part 31-38 | Step 301-380)

| Part | ชื่อตอน | ขั้นตอน | สถานะ |
|---|---|---|---|
| 031 | Django Authentication System เบื้องต้น | 301-310 | ✅ |
| 032 | Custom User Model | 311-320 | ✅ |
| 033 | Permissions และ Groups | 321-330 | ✅ |
| 034 | Django Sessions และ Cookies | 331-340 | ✅ |
| 035 | Password Management, Reset และ Security | 341-350 | ✅ |
| 036 | Social Authentication (OAuth, django-allauth) | 351-360 | ✅ |
| 037 | Two-Factor Authentication | 361-370 | ✅ |
| 038 | Row-Level Permissions และ Object-Level Permission | 371-380 | ✅ |

## Phase 5: Django REST Framework และ API (Part 39-50 | Step 381-500)

| Part | ชื่อตอน | ขั้นตอน | สถานะ |
|---|---|---|---|
| 039 | บทนำสู่ Django REST Framework | 381-390 | ✅ |
| 040 | Serializers เบื้องต้น | 391-400 | ✅ |
| 041 | ModelSerializer และ Nested Serializers | 401-410 | ✅ |
| 042 | API Views: Function-Based และ APIView | 411-420 | ✅ |
| 043 | Generic API Views และ Mixins | 421-430 | ✅ |
| 044 | ViewSets และ Routers | 431-440 | ✅ |
| 045 | DRF Permissions และ Authentication | 441-450 | ✅ |
| 046 | Authentication ขั้นสูง: JWT, Token, OAuth2 | 451-460 | ✅ |
| 047 | Filtering, Searching, Pagination ใน DRF | 461-470 | ◻️ |
| 048 | API Versioning และ Throttling | 471-480 | ◻️ |
| 049 | API Documentation: drf-spectacular, Swagger, OpenAPI | 481-490 | ◻️ |
| 050 | Testing REST APIs | 491-500 | ◻️ |

## Phase 6: Frontend Integration (Part 51-58 | Step 501-580)

| Part | ชื่อตอน | ขั้นตอน | สถานะ |
|---|---|---|---|
| 051 | Django กับ Bootstrap และ CSS Framework | 501-510 | ◻️ |
| 052 | Django กับ JavaScript และ Fetch API | 511-520 | ◻️ |
| 053 | HTMX กับ Django สำหรับ Interactive UI | 521-530 | ◻️ |
| 054 | Django กับ Alpine.js | 531-540 | ◻️ |
| 055 | Django กับ React (Django เป็น API Backend) | 541-550 | ◻️ |
| 056 | Django กับ Vue.js Integration | 551-560 | ◻️ |
| 057 | WebSockets เบื้องต้นด้วย Django Channels | 561-570 | ◻️ |
| 058 | File Upload, Image Processing และ Media Handling | 571-580 | ◻️ |

## Phase 7: Testing & Quality Assurance (Part 59-65 | Step 581-650)

| Part | ชื่อตอน | ขั้นตอน | สถานะ |
|---|---|---|---|
| 059 | Django Testing เบื้องต้น: unittest | 581-590 | ◻️ |
| 060 | Testing Views, Models และ Forms | 591-600 | ◻️ |
| 061 | pytest-django และ Fixtures | 601-610 | ◻️ |
| 062 | Test Coverage และ Mocking | 611-620 | ◻️ |
| 063 | Factory Boy และ Test Data Generation | 621-630 | ◻️ |
| 064 | Integration Testing และ Selenium | 631-640 | ◻️ |
| 065 | Continuous Testing และ Code Quality Tools | 641-650 | ◻️ |

## Phase 8: Performance & Caching (Part 66-72 | Step 651-720)

| Part | ชื่อตอน | ขั้นตอน | สถานะ |
|---|---|---|---|
| 066 | Django Performance Profiling | 651-660 | ◻️ |
| 067 | Query Optimization: select_related, prefetch_related | 661-670 | ◻️ |
| 068 | Django Caching Framework เบื้องต้น | 671-680 | ◻️ |
| 069 | Redis Caching ขั้นสูง | 681-690 | ◻️ |
| 070 | Database Indexing และ Query Analysis | 691-700 | ◻️ |
| 071 | Pagination และ Large Dataset Handling | 701-710 | ◻️ |
| 072 | Load Testing และ Scalability Testing | 711-720 | ◻️ |

## Phase 9: Async, Celery และ Channels (Part 73-79 | Step 721-790)

| Part | ชื่อตอน | ขั้นตอน | สถานะ |
|---|---|---|---|
| 073 | Async Views และ ASGI เบื้องต้น | 721-730 | ◻️ |
| 074 | Django Channels ขั้นสูง: Consumers และ Groups | 731-740 | ◻️ |
| 075 | Celery เบื้องต้น: Background Tasks | 741-750 | ◻️ |
| 076 | Celery ขั้นสูง: Periodic Tasks, Chains, Chords | 751-760 | ◻️ |
| 077 | Message Queue ด้วย RabbitMQ/Redis | 761-770 | ◻️ |
| 078 | Real-time Notification System | 771-780 | ◻️ |
| 079 | Background Job Monitoring: Flower, Django-RQ | 781-790 | ◻️ |

## Phase 10: Security (Part 80-85 | Step 791-850)

| Part | ชื่อตอน | ขั้นตอน | สถานะ |
|---|---|---|---|
| 080 | Django Security Best Practices เบื้องต้น | 791-800 | ◻️ |
| 081 | CSRF, XSS และ SQL Injection Prevention | 801-810 | ◻️ |
| 082 | Security Headers และ HTTPS | 811-820 | ◻️ |
| 083 | Rate Limiting และ Brute Force Protection | 821-830 | ◻️ |
| 084 | Secrets Management และ Environment Variables | 831-840 | ◻️ |
| 085 | Security Auditing และ Penetration Testing เบื้องต้น | 841-850 | ◻️ |

## Phase 11: DevOps, Docker และ CI/CD (Part 86-93 | Step 851-930)

| Part | ชื่อตอน | ขั้นตอน | สถานะ |
|---|---|---|---|
| 086 | Docker เบื้องต้นสำหรับ Django | 851-860 | ◻️ |
| 087 | Docker Compose: Django + PostgreSQL + Redis | 861-870 | ◻️ |
| 088 | CI/CD ด้วย GitHub Actions | 871-880 | ◻️ |
| 089 | Deployment บน VPS ด้วย Gunicorn และ Nginx | 881-890 | ◻️ |
| 090 | Deployment บน Cloud: AWS/GCP/Azure | 891-900 | ◻️ |
| 091 | Deployment บน Heroku, Railway, Render | 901-910 | ◻️ |
| 092 | Kubernetes เบื้องต้นสำหรับ Django | 911-920 | ◻️ |
| 093 | Monitoring และ Logging: Sentry, ELK Stack | 921-930 | ◻️ |

## Phase 12: Scaling, Enterprise, โปรเจกต์จริง และเส้นทางอาชีพ (Part 94-100 | Step 931-1000)

| Part | ชื่อตอน | ขั้นตอน | สถานะ |
|---|---|---|---|
| 094 | Microservices Architecture กับ Django | 931-940 | ◻️ |
| 095 | Multi-tenant Django Applications | 941-950 | ◻️ |
| 096 | GraphQL กับ Django (Graphene) | 951-960 | ◻️ |
| 097 | โปรเจกต์จริง: ระบบ E-Commerce แบบครบวงจร | 961-970 | ◻️ |
| 098 | โปรเจกต์จริง: ระบบ Social Media Platform | 971-980 | ◻️ |
| 099 | Django Design Patterns และ Clean Architecture | 981-990 | ◻️ |
| 100 | เส้นทางอาชีพ Django Developer และ Open Source | 991-1000 | ◻️ |

---

## วิธีใช้หลักสูตรนี้

1. เรียงตามลำดับ Part 001 → 100 ห้ามข้าม เพราะแต่ละตอนต่อยอดจากตอนก่อนหน้า
2. ลงมือโค้ดตามทุกตัวอย่างจริงในเครื่องของคุณ ห้ามอ่านเฉย ๆ
3. ทำแบบฝึกหัดท้ายบทก่อนไป Part ถัดไป
4. ตั้งแต่ Phase 5 เป็นต้นไป ควรมี "โปรเจกต์คู่ขนาน" ของตัวเองที่ประยุกต์ใช้ความรู้แต่ละ Phase
5. Phase 11-12 คือระดับที่ทำให้คุณพร้อมสำหรับงานจริงและสัมภาษณ์งานระดับ Senior/Staff Engineer

## License

เนื้อหาในหลักสูตรนี้เปิดให้ใช้เพื่อการศึกษาโดยเสรี
