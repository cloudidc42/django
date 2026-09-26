# Part 031: Django Authentication System เบื้องต้น

> **ขั้นตอนที่ 301-310 ของหลักสูตร** | Phase 4: Authentication, Users และ Permissions
>
> เป้าหมายของ Part นี้: เจาะลึกระบบ `django.contrib.auth` ที่มากับ Django ตั้งแต่ต้น
> เข้าใจ default `User` model, ตาราง auth ในฐานข้อมูล, `@login_required`/
> `LoginRequiredMixin` แบบเต็มรูปแบบตามที่ Part 023/024 ค้างไว้, กลไก
> `authenticate()`/`login()`/`logout()` เบื้องหลัง session, การสร้างหน้า signup ด้วย
> `UserCreationForm`, `request.user`/`AnonymousUser`, การตั้งค่า URL auth ทั้งระบบผ่าน
> `django.contrib.auth.urls`, ความเสี่ยง Open Redirect ตอน logout, การเช็คสถานะ login
> ใน template และภาพรวมสั้น ๆ ของ django-allauth เมื่อจบ Part นี้ คุณจะสามารถสร้างระบบ
> login/logout/signup ที่สมบูรณ์และปลอดภัยให้กับแอป `blog` ได้ พร้อมจำกัดว่าต้อง login
> ก่อนสร้าง/แก้ไขบทความ

---

## สารบัญของ Part นี้

- ขั้นตอนที่ 301: ภาพรวม `django.contrib.auth` — field ของ default `User` model และตาราง auth ที่ Django ติดตั้งให้อัตโนมัติ
- ขั้นตอนที่ 302: `@login_required` decorator และ `LoginRequiredMixin` แบบเจาะลึกเต็มรูปแบบ
- ขั้นตอนที่ 303: `authenticate()` และ `login()` เบื้องต้น เทียบกับ built-in `LoginView`
- ขั้นตอนที่ 304: `UserCreationForm` และหน้า signup สมัครสมาชิกจริง
- ขั้นตอนที่ 305: `request.user`, `AnonymousUser`, การเช็ค `is_authenticated`
- ขั้นตอนที่ 306: ตั้งค่า URL auth ทั้งระบบผ่าน `django.contrib.auth.urls` เทียบกับเขียนเอง
- ขั้นตอนที่ 307: `LogoutView`, พารามิเตอร์ `next` และความเสี่ยง Open Redirect
- ขั้นตอนที่ 308: เช็คสถานะ login ใน Template — `{% if user.is_authenticated %}`
- ขั้นตอนที่ 309: ภาพรวมสั้น ๆ ของ django-allauth เทียบกับ built-in auth
- ขั้นตอนที่ 310: สรุปและแบบฝึกหัด — ระบบ login/logout/signup สมบูรณ์สำหรับ blog

---

## ขั้นตอนที่ 301: ภาพรวม `django.contrib.auth` — field ของ default `User` model และตาราง auth ที่ Django ติดตั้งให้อัตโนมัติ

### 301.1 ทบทวน: `django.contrib.auth` คืออะไร

ตั้งแต่ Part 004 ที่คุณสร้างโปรเจกต์ Django แรก คุณอาจเคยเห็นบรรทัดนี้ใน
`settings.py` มาโดยไม่ได้สนใจรายละเอียดมากนัก:

```python
# config/settings.py
INSTALLED_APPS = [
    "django.contrib.admin",
    "django.contrib.auth",              # <- นี่คือแอปที่เราจะเจาะลึกใน Part นี้
    "django.contrib.contenttypes",      # <- auth ต้องพึ่งแอปนี้
    "django.contrib.sessions",          # <- auth ใช้ session เก็บสถานะ login
    "django.contrib.messages",
    "django.contrib.staticfiles",
    "blog",
]

MIDDLEWARE = [
    "django.middleware.security.SecurityMiddleware",
    "django.contrib.sessions.middleware.SessionMiddleware",   # <- ต้องมาก่อน AuthenticationMiddleware
    "django.middleware.common.CommonMiddleware",
    "django.middleware.csrf.CsrfViewMiddleware",
    "django.contrib.auth.middleware.AuthenticationMiddleware", # <- นี่คือตัวที่ทำให้ request.user ใช้ได้
    "django.contrib.messages.middleware.MessageMiddleware",
    "django.middleware.clickjacking.XFrameOptionsMiddleware",
]
```

`django.contrib.auth` คือ **แอปมาตรฐาน (built-in app)** ของ Django ที่ให้ระบบ
Authentication (ยืนยันตัวตนว่า "คุณคือใคร") และ Authorization (ตรวจสอบว่า
"คุณมีสิทธิ์ทำอะไรได้บ้าง") มาให้ครบตั้งแต่ `pip install django` โดยไม่ต้องติดตั้ง
package เพิ่มเติมเลย ซึ่งสอดคล้องกับปรัชญา "Batteries Included" ที่เรียนไปใน
Part 001 ขั้นตอนที่ 2

สองแอปที่ `django.contrib.auth` **ต้องพึ่งพา** เสมอคือ:

| แอปที่ต้องมี | ทำไมต้องมี |
|---|---|
| `django.contrib.contenttypes` | เก็บ metadata ของทุก Model ในระบบ ใช้เป็นกลไกให้ตาราง `auth_permission` รู้ว่า permission แต่ละตัวผูกกับ Model ไหน (จะเจาะลึกใน Part 033) |
| `django.contrib.sessions` | เก็บสถานะ "ใคร login อยู่" ไว้ในฝั่งเซิร์ฟเวอร์ ผูกกับ cookie `sessionid` ที่ส่งไปเก็บที่ browser (จะเจาะลึกใน Part 034) |

ถ้าเอา 2 แอปนี้ออกจาก `INSTALLED_APPS` ระบบ auth ทั้งหมดจะพังทันที เพราะ
`auth.User` มี field ที่อ้างอิงไปยัง `ContentType` (ผ่านตาราง permission) และ
login state ทั้งหมดถูกเก็บผ่านกลไก session

### 301.2 field ทั้งหมดของ default `User` model

Django มี Model ชื่อ `User` อยู่แล้วที่ `django.contrib.auth.models.User` พร้อมใช้งาน
ทันทีโดยไม่ต้องเขียน Model เอง (Part 032 จะสอนวิธีสร้าง **Custom User Model** ของ
ตัวเองเพื่อทดแทน `User` ตัวนี้ในกรณีที่ต้องการ field เพิ่มเติมตั้งแต่ต้นโปรเจกต์)

```python
# django/contrib/auth/models.py (โค้ดจริงของ Django ย่อเพื่อความเข้าใจ)
class User(AbstractUser):
    """
    Users within the Django authentication system are represented by this model.
    """
    class Meta(AbstractUser.Meta):
        swappable = "AUTH_USER_MODEL"
```

`User` สืบทอดมาจาก `AbstractUser` ซึ่งมี field มาตรฐานดังนี้:

| Field | ชนิดข้อมูล | คำอธิบาย |
|---|---|---|
| `id` | `AutoField` | Primary key อัตโนมัติ |
| `username` | `CharField(max_length=150, unique=True)` | ชื่อผู้ใช้ที่ใช้ login (unique ทั้งระบบ) |
| `first_name` | `CharField(max_length=150, blank=True)` | ชื่อจริง |
| `last_name` | `CharField(max_length=150, blank=True)` | นามสกุล |
| `email` | `EmailField(blank=True)` | อีเมล (**ไม่ unique โดยค่าเริ่มต้น!** เป็นกับดักที่มือใหม่มักไม่รู้) |
| `password` | `CharField(max_length=128)` | เก็บ **hash** ของรหัสผ่าน ไม่เคยเก็บ plaintext (จะเจาะลึกอัลกอริทึม hashing ใน Part 035) |
| `is_staff` | `BooleanField(default=False)` | ใช้กำหนดว่าเข้า Django Admin site ได้หรือไม่ |
| `is_active` | `BooleanField(default=True)` | ใช้ "ปิดการใช้งาน" บัญชีโดยไม่ต้องลบข้อมูล (soft-delete pattern) |
| `is_superuser` | `BooleanField(default=False)` | มีสิทธิ์ทุกอย่างในระบบโดยอัตโนมัติ ไม่ต้องเช็ค permission ทีละตัว |
| `last_login` | `DateTimeField(null=True, blank=True)` | อัปเดตอัตโนมัติทุกครั้งที่ login สำเร็จ |
| `date_joined` | `DateTimeField(default=timezone.now)` | วันที่สมัครสมาชิก |
| `groups` | `ManyToManyField(Group)` | กลุ่มผู้ใช้ที่ user สังกัดอยู่ (จะเจาะลึกใน Part 033) |
| `user_permissions` | `ManyToManyField(Permission)` | permission เฉพาะบุคคลที่ไม่ได้มาจาก group (จะเจาะลึกใน Part 033) |

> **ข้อควรระวังสำคัญ**: `email` ไม่ได้ตั้งเป็น `unique=True` โดยค่าเริ่มต้น หมายความว่า
> ผู้ใช้สองคนสามารถสมัครด้วยอีเมลเดียวกันได้ในระบบมาตรฐานของ Django! ถ้าธุรกิจของคุณ
> ต้องการให้อีเมล unique (ซึ่งเกือบทุกระบบจริงต้องการ) คุณต้องแก้ผ่าน Custom User
> Model ใน Part 032 หรือเพิ่ม validation เองในฟอร์มสมัครสมาชิก (จะสาธิตวิธีเพิ่ม
> validation ใน ขั้นตอนที่ 304)

### 301.3 `is_staff` vs `is_superuser` vs `is_active`: ตารางเปรียบเทียบที่ต้องแม่น

สามฟิลด์นี้เป็นฟิลด์ที่มือใหม่สับสนกันบ่อยที่สุดในระบบ auth ของ Django:

| Field | ควบคุมอะไร | ตัวอย่างการใช้งานจริง |
|---|---|---|
| `is_active=False` | ผู้ใช้ **login ไม่ได้เลย** ไม่ว่าจะมีสิทธิ์อะไรก็ตาม | ระงับบัญชีชั่วคราว (banned user), รอยืนยันอีเมล |
| `is_staff=True` | ผู้ใช้เข้า **Django Admin site** (`/admin/`) ได้ แต่ยังต้องมี permission เฉพาะทางถึงจะแก้ไขข้อมูลใน admin ได้จริง | พนักงานที่ต้องเข้า admin แต่ไม่ใช่ผู้ดูแลระบบเต็มรูปแบบ |
| `is_superuser=True` | ผู้ใช้ **มีทุก permission โดยอัตโนมัติ** ไม่ว่าจะกำหนด permission เฉพาะไว้หรือไม่ (bypass การเช็ค permission ทั้งหมด) | ผู้ดูแลระบบสูงสุด (คนที่รัน `createsuperuser`) |

สังเกตว่าทั้งสามค่านี้**เป็นอิสระจากกัน** — เป็นไปได้ที่ user จะมี `is_staff=True` แต่
`is_superuser=False` (staff ทั่วไปที่เข้า admin ได้แต่ทำได้จำกัด) หรือมี
`is_active=False` พร้อม `is_superuser=True` (ซูเปอร์ยูสเซอร์ที่ถูกระงับบัญชีชั่วคราว
ก็ login ไม่ได้เหมือนกัน) เราจะเจาะลึกเรื่อง permission และ group อย่างเต็มรูปแบบใน
Part 033

### 301.4 ตารางแสดง auth-related tables ที่ Django สร้างให้อัตโนมัติตอน `migrate`

เมื่อคุณรัน `python manage.py migrate` ครั้งแรกในโปรเจกต์ใหม่ (แม้ยังไม่ได้เขียน
Model ของตัวเองเลยสักตัว) Django จะสร้างตารางต่อไปนี้ในฐานข้อมูลให้อัตโนมัติ
เพราะ `django.contrib.auth`, `django.contrib.contenttypes`, และ
`django.contrib.sessions` อยู่ใน `INSTALLED_APPS` ตั้งแต่ต้น:

| ชื่อตาราง | มาจากแอป | เก็บอะไร |
|---|---|---|
| `auth_user` | `django.contrib.auth` | ข้อมูลผู้ใช้ทั้งหมดตาม field ในขั้นตอนที่ 301.2 |
| `auth_group` | `django.contrib.auth` | กลุ่มผู้ใช้ (เช่น "Editor", "Moderator") |
| `auth_permission` | `django.contrib.auth` | รายการ permission ทั้งหมดในระบบ (Django สร้างให้อัตโนมัติ 4 ตัวต่อ Model ทุกตัว: add, change, delete, view) |
| `auth_group_permissions` | `django.contrib.auth` | ตารางเชื่อม (junction table) ระหว่าง group กับ permission (ManyToMany) |
| `auth_user_groups` | `django.contrib.auth` | ตารางเชื่อมระหว่าง user กับ group (ManyToMany) |
| `auth_user_user_permissions` | `django.contrib.auth` | ตารางเชื่อมระหว่าง user กับ permission เฉพาะบุคคล (ManyToMany) |
| `django_content_type` | `django.contrib.contenttypes` | เก็บรายการ Model ทั้งหมดในโปรเจกต์ (app_label + model name) เพื่อให้ `auth_permission` อ้างอิงถึงได้ |
| `django_session` | `django.contrib.sessions` | เก็บข้อมูล session ของผู้ใช้ที่ login อยู่ (จะเจาะลึกใน Part 034) |

ตรวจสอบได้จริงด้วยตัวเองผ่าน `python manage.py shell`:

```python
from django.contrib.auth.models import User, Group, Permission

print(User.objects.count())        # 0 ถ้ายังไม่มีใครสมัคร
print(Group.objects.count())       # 0 ถ้ายังไม่สร้าง group
print(Permission.objects.count())  # จะเห็นตัวเลขไม่น้อย เพราะ Django สร้าง
                                    # permission (add/change/delete/view) ให้ทุก Model
                                    # อัตโนมัติแล้ว รวมถึง Model ของแอป blog เอง!

for p in Permission.objects.filter(content_type__app_label="blog")[:8]:
    print(p.content_type.model, "->", p.codename)
```

ผลลัพธ์ตัวอย่าง (สมมติแอป `blog` มี Model `Post`):

```
post -> add_post
post -> change_post
post -> delete_post
post -> view_post
```

นี่คือสิ่งที่เรียกว่า **default permissions** ที่ Django สร้างให้ **ทุก Model**
โดยอัตโนมัติทันทีที่รัน migration — เราจะนำ permission เหล่านี้มาใช้งานจริงร่วมกับ
`PermissionRequiredMixin` ใน Part 033

### 301.5 สร้างผู้ใช้แรกด้วย `createsuperuser`

ก่อนทดสอบระบบ login ในขั้นตอนถัดไป ต้องมีผู้ใช้อย่างน้อย 1 คนในระบบก่อน:

```bash
python manage.py createsuperuser
```

```
Username: admin
Email address: admin@example.com
Password: ********
Password (again): ********
Superuser created successfully.
```

คำสั่งนี้สร้าง `User` ที่มี `is_staff=True`, `is_superuser=True`, `is_active=True`
ให้อัตโนมัติ — สามารถ login เข้า `/admin/` ได้ทันที และเราจะใช้บัญชีนี้ทดสอบระบบ
login ที่กำลังจะสร้างใน Part นี้ด้วย

### 301.6 ตารางสรุปโมดูลย่อยภายใน `django.contrib.auth` ที่จะได้ใช้ตลอด Part นี้

| โมดูล/ฟังก์ชัน | Import จาก | ใช้ทำอะไร |
|---|---|---|
| `authenticate()` | `django.contrib.auth` | ตรวจสอบ username/password ว่าถูกต้องหรือไม่ |
| `login()` | `django.contrib.auth` | บันทึกสถานะ login ลง session |
| `logout()` | `django.contrib.auth` | ล้างสถานะ login ออกจาก session |
| `login_required` | `django.contrib.auth.decorators` | decorator บังคับให้ login ก่อนเข้า FBV |
| `LoginRequiredMixin` | `django.contrib.auth.mixins` | mixin บังคับให้ login ก่อนเข้า CBV |
| `AuthenticationForm` | `django.contrib.auth.forms` | ฟอร์ม login สำเร็จรูป |
| `UserCreationForm` | `django.contrib.auth.forms` | ฟอร์มสมัครสมาชิกสำเร็จรูป |
| `LoginView`, `LogoutView` | `django.contrib.auth.views` | CBV สำเร็จรูปสำหรับ login/logout |
| `AnonymousUser` | `django.contrib.auth.models` | ตัวแทนของ "ผู้ใช้ที่ยังไม่ login" |

---

## ขั้นตอนที่ 302: `@login_required` decorator และ `LoginRequiredMixin` แบบเจาะลึกเต็มรูปแบบ

### 302.1 ทบทวนสิ่งที่ Part 023/024 เกริ่นไว้

ใน Part 023 ขั้นตอนที่ 226 และ Part 024 ขั้นตอนที่ 231-234 คุณเคยเห็น
`LoginRequiredMixin` มาแล้วในบริบทของการป้องกัน `PostCreateView`/`PostUpdateView`/
`PostDeleteView` แต่ทั้งสอง Part นั้น **ตั้งใจเกริ่นแบบผิวเผิน** และบอกไว้ชัดเจนว่า
รายละเอียดเต็มรูปแบบจะมาที่ Part 031 (Part นี้เอง!) — เรามาดูกันว่าเบื้องหลังการ
ทำงานจริง ๆ เป็นอย่างไร

### 302.2 `@login_required` decorator สำหรับ Function-Based View

```python
# blog/views.py
from django.contrib.auth.decorators import login_required
from django.shortcuts import redirect, render

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

`login_required` คือ decorator ที่ห่อฟังก์ชัน view ไว้ เมื่อ request เข้ามาจาก
ผู้ใช้ที่ **ยังไม่ login** มันจะ redirect ไปหน้า login โดยอัตโนมัติ **ก่อน** ที่โค้ด
ข้างในฟังก์ชัน `post_create` จะได้ทำงานแม้แต่บรรทัดเดียว

โค้ดจริงของ Django (ย่อเพื่อความเข้าใจ) แสดงให้เห็นว่ามันคือ decorator ธรรมดาที่
เรียนไปใน Part 002 ขั้นตอนที่ 14 นั่นเอง:

```python
# django/contrib/auth/decorators.py (โค้ดจริงย่อเพื่อความเข้าใจ)
def login_required(function=None, redirect_field_name=REDIRECT_FIELD_NAME, login_url=None):
    actual_decorator = user_passes_test(
        lambda u: u.is_authenticated,
        login_url=login_url,
        redirect_field_name=redirect_field_name,
    )
    if function:
        return actual_decorator(function)
    return actual_decorator
```

สังเกตว่า `login_required` แท้จริงแล้วคือการเรียก `user_passes_test()` โดยส่ง
เงื่อนไข `lambda u: u.is_authenticated` เข้าไป — เป็นรูปแบบเดียวกับที่เราจะเห็นใน
`UserPassesTestMixin` ฝั่ง CBV (Part 024 ขั้นตอนที่ 234) เพียงแต่ฝั่ง FBV ใช้
decorator แทน mixin

### 302.3 กำหนด `login_url` และ `redirect_field_name` ให้ `@login_required`

```python
@login_required(login_url="/accounts/login/", redirect_field_name="next")
def post_create(request):
    ...
```

| พารามิเตอร์ | ค่าเริ่มต้น | ความหมาย |
|---|---|---|
| `login_url` | `None` (จะไปอ่านจาก `settings.LOGIN_URL` แทน ซึ่งค่าเริ่มต้นคือ `/accounts/login/`) | URL ที่จะ redirect ไปเมื่อยังไม่ login |
| `redirect_field_name` | `"next"` | ชื่อ query string parameter ที่แนบ URL เดิมไปด้วย เพื่อให้กลับมาหน้าเดิมได้หลัง login สำเร็จ |

เมื่อผู้ใช้ที่ยังไม่ login พยายามเข้า `/blog/new/` ระบบจะ redirect ไปที่:

```
/accounts/login/?next=/blog/new/
```

### 302.4 `LoginRequiredMixin` สำหรับ Class-Based View แบบเจาะลึก

```python
# blog/views.py
from django.contrib.auth.mixins import LoginRequiredMixin
from django.views.generic import CreateView

from .forms import PostForm
from .models import Post


class PostCreateView(LoginRequiredMixin, CreateView):
    model = Post
    form_class = PostForm
    login_url = "/accounts/login/"
    redirect_field_name = "next"

    def form_valid(self, form):
        form.instance.author = self.request.user
        return super().form_valid(form)
```

`LoginRequiredMixin` และ `login_required` ทำหน้าที่**เหมือนกันทุกประการ** เพียงแต่
คนละรูปแบบ (decorator vs mixin) ที่เหมาะกับคนละสไตล์ view — และทั้งคู่ทำงานผ่าน
กลไกเดียวกันคือ `AccessMixin` ที่ `LoginRequiredMixin` สืบทอดมา:

```python
# django/contrib/auth/mixins.py (โค้ดจริงย่อเพื่อความเข้าใจ)
class AccessMixin:
    login_url = None
    permission_denied_message = ""
    redirect_field_name = REDIRECT_FIELD_NAME
    raise_exception = False

    def get_login_url(self):
        return self.login_url or settings.LOGIN_URL

    def get_redirect_field_name(self):
        return self.redirect_field_name

    def handle_no_permission(self):
        if self.raise_exception or self.request.user.is_authenticated:
            raise PermissionDenied(self.get_permission_denied_message())
        return redirect_to_login(
            self.request.get_full_path(),
            self.get_login_url(),
            self.get_redirect_field_name(),
        )


class LoginRequiredMixin(AccessMixin):
    def dispatch(self, request, *args, **kwargs):
        if not request.user.is_authenticated:
            return self.handle_no_permission()
        return super().dispatch(request, *args, **kwargs)
```

### 302.5 ตารางเปรียบเทียบ attribute ที่ปรับแต่งได้ทั้งหมดของ `AccessMixin`

| Attribute/Method | ค่าเริ่มต้น | ความหมาย |
|---|---|---|
| `login_url` | `None` → fallback ไป `settings.LOGIN_URL` | URL ปลายทางที่จะ redirect เมื่อยังไม่ login |
| `redirect_field_name` | `"next"` | ชื่อ query string ที่บอกว่าจะกลับไปหน้าไหนหลัง login |
| `raise_exception` | `False` | `True` = ยิง `PermissionDenied` (403) ทันทีแทนที่จะ redirect ไป login |
| `permission_denied_message` | `""` | ข้อความที่แสดงเมื่อ raise `PermissionDenied` |
| `get_login_url()` | — | override ได้ถ้าต้องการ logic ซับซ้อนกว่าค่าคงที่ (เช่น login_url ต่างกันตาม request) |
| `get_redirect_field_name()` | — | override ได้เช่นกัน |
| `handle_no_permission()` | — | จุดที่ควบคุมพฤติกรรมทั้งหมดเมื่อเข้าถึงไม่ได้ ปรับแต่งได้อิสระ |

### 302.6 `LOGIN_URL` ใน `settings.py`

```python
# config/settings.py
LOGIN_URL = "login"          # ใช้ URL name ได้ (Django resolve ให้อัตโนมัติ)
# หรือ
LOGIN_URL = "/accounts/login/"   # ใช้ path ตรง ๆ ก็ได้
```

ถ้าไม่กำหนด `LOGIN_URL` เลย ค่าเริ่มต้นของ Django คือ `/accounts/login/` ซึ่งตรงกับ
URL pattern ที่ `django.contrib.auth.urls` เตรียมไว้ให้พอดี (จะอธิบายเต็มรูปแบบใน
ขั้นตอนที่ 306) — นี่คือเหตุผลที่หลายโปรเจกต์ Django ไม่เคยต้องตั้งค่า `LOGIN_URL`
เองเลย เพราะ path เริ่มต้นถูกออกแบบมาให้ตรงกับ URL ของระบบ auth มาตรฐานอยู่แล้ว

### 302.7 ตารางเปรียบเทียบ `@login_required` กับ `LoginRequiredMixin`

| ประเด็น | `@login_required` (FBV) | `LoginRequiredMixin` (CBV) |
|---|---|---|
| ใช้กับ | Function-Based View | Class-Based View |
| ตำแหน่งที่วาง | บนสุดของฟังก์ชัน (decorator) | ซ้ายสุดของ parent classes (Part 024 ขั้นตอนที่ 233) |
| กำหนด `login_url` | ผ่าน parameter ของ decorator | ผ่าน class attribute `login_url` |
| กำหนด `raise_exception` | ทำไม่ได้ตรง ๆ ต้องใช้ `user_passes_test` เอง | ทำได้ผ่าน class attribute `raise_exception` |
| กลไกภายใน | ห่อฟังก์ชันด้วย `user_passes_test()` | override `dispatch()` ผ่าน MRO |
| ทดสอบง่ายกว่า | ระดับ function เดี่ยว ๆ | ระดับ class รวมกับ mixin อื่นได้ (Part 024) |

### 302.8 `raise_exception=True`: แสดง 403 แทนการ redirect

บางกรณี (เช่น API endpoint ที่ผู้ใช้ AJAX เรียก) การ redirect ไปหน้า login อาจไม่ใช่
พฤติกรรมที่ต้องการ — ควรตอบ `403 Forbidden` ตรง ๆ แทน:

```python
class PostCreateView(LoginRequiredMixin, CreateView):
    model = Post
    form_class = PostForm
    raise_exception = True   # ยิง 403 ทันทีถ้ายังไม่ login แทนที่จะ redirect
```

---

## ขั้นตอนที่ 303: `authenticate()` และ `login()` เบื้องต้น เทียบกับ built-in `LoginView`

### 303.1 กลไกเบื้องหลัง Session-Based Authentication

ก่อนเขียนหน้า login เอง ต้องเข้าใจก่อนว่า Django auth แบบ default ทำงานผ่าน
**session** (จะเจาะลึกเต็มรูปแบบใน Part 034) โดยสรุปสั้น ๆ คือ:

```
1. ผู้ใช้กรอก username/password ส่งมาที่ view
2. authenticate() ตรวจสอบว่า username/password ถูกต้องหรือไม่ (เทียบ hash ของ password)
   → ถ้าถูกต้อง คืนค่าเป็น User object
   → ถ้าไม่ถูกต้อง คืนค่า None
3. login() บันทึกว่า "user คนนี้ login แล้ว" ลง session
   → Django สร้าง session key แล้วส่ง cookie ชื่อ sessionid กลับไปที่ browser
4. Request ถัดไปทุกครั้งที่ browser แนบ cookie sessionid มา
   AuthenticationMiddleware จะ resolve เป็น request.user ให้อัตโนมัติ
```

### 303.2 เขียน login view ด้วยตัวเองด้วย `authenticate()` + `login()`

```python
# accounts/views.py
from django.contrib.auth import authenticate, login
from django.shortcuts import redirect, render


def manual_login_view(request):
    if request.method == "POST":
        username = request.POST.get("username")
        password = request.POST.get("password")

        # authenticate() ตรวจสอบ credential กับฐานข้อมูล (ผ่าน authentication backend)
        # คืนค่า User object ถ้าถูกต้อง หรือ None ถ้าผิด
        user = authenticate(request, username=username, password=password)

        if user is not None:
            # login() บันทึกสถานะ login ลง session ของ request นี้
            login(request, user)
            return redirect("blog:post-list")
        else:
            return render(
                request,
                "registration/login.html",
                {"error": "Username หรือ Password ไม่ถูกต้อง"},
            )

    return render(request, "registration/login.html")
```

```html
<!-- templates/registration/login.html -->
{% extends "base.html" %}

{% block content %}
<h1>เข้าสู่ระบบ</h1>

{% if error %}
    <p style="color: red;">{{ error }}</p>
{% endif %}

<form method="post">
    {% csrf_token %}
    <label>Username: <input type="text" name="username" required></label>
    <label>Password: <input type="password" name="password" required></label>
    <button type="submit">เข้าสู่ระบบ</button>
</form>
{% endblock %}
```

### 303.3 อธิบาย `authenticate()` ลึกขึ้น: ทำไมต้องส่ง `request` เข้าไปด้วย

```python
user = authenticate(request, username=username, password=password)
```

พารามิเตอร์ `request` **ไม่ได้บังคับเสมอไป** ตาม signature ของฟังก์ชัน แต่ Django
**แนะนำให้ส่งเสมอ** เพราะ authentication backend บางตัว (เช่น backend ที่ทำ
throttling/rate-limiting การพยายาม login ผิดพลาด ซึ่งจะเจาะลึกใน Part 083) ต้องใช้
ข้อมูลจาก `request` เช่น IP address ประกอบการตัดสินใจ

`authenticate()` เบื้องหลังจะวนลูปผ่านค่าที่ตั้งไว้ใน `AUTHENTICATION_BACKENDS`
(ค่าเริ่มต้นมีแค่ `ModelBackend` ตัวเดียว ซึ่งเช็คกับตาราง `auth_user`) แล้วคืนค่า
`User` ตัวแรกที่ backend ใดก็ตามยืนยันว่าถูกต้อง — เป็นสถาปัตยกรรมที่ยืดหยุ่นมาก
เพราะรองรับการเพิ่ม backend อื่น ๆ ในอนาคต (เช่น login ผ่านอีเมลแทน username หรือ
ผ่าน LDAP ในระบบองค์กร)

### 303.4 `login()`: อะไรเกิดขึ้นเบื้องหลัง

```python
login(request, user)
```

`login()` ทำสิ่งสำคัญ 3 อย่าง:

1. เรียก `request.session.cycle_key()` เพื่อสร้าง **session key ใหม่** (ป้องกัน
   **Session Fixation Attack** — ถ้าไม่สร้างใหม่ ผู้โจมตีที่รู้ session key เดิม
   ก่อน login อาจสวมรอยได้)
2. บันทึก `user.pk`, `user.get_session_auth_hash()`, และ backend ที่ใช้ยืนยันตัวตน
   ลงใน session data
3. ส่ง signal `user_logged_in` (สามารถเขียน receiver ดักฟังได้ ตามที่เรียนใน
   Part 019 เรื่อง Signals)

### 303.5 เทียบกับการใช้ built-in `LoginView`

การเขียน login view เองแบบข้างต้นมีประโยชน์เพื่อความเข้าใจกลไก แต่ในงานจริง
Django มี **`LoginView`** สำเร็จรูปที่ครอบคลุม edge case สำคัญ ๆ ให้แล้ว
(CSRF protection, redirect หลัง login ที่ตรวจสอบ `next` อย่างปลอดภัย,
`AuthenticationForm` ที่มี validation ครบ):

```python
# accounts/urls.py
from django.contrib.auth import views as auth_views
from django.urls import path

urlpatterns = [
    path("login/", auth_views.LoginView.as_view(template_name="registration/login.html"), name="login"),
]
```

Template ที่ `LoginView` ต้องการมีโครงสร้างเพียงเล็กน้อย เพราะ `AuthenticationForm`
จัดการ field ให้หมดแล้ว:

```html
<!-- templates/registration/login.html -->
{% extends "base.html" %}

{% block content %}
<h1>เข้าสู่ระบบ</h1>

<form method="post">
    {% csrf_token %}
    {{ form.as_p }}
    <button type="submit">เข้าสู่ระบบ</button>
</form>
{% endblock %}
```

### 303.6 ตารางเปรียบเทียบ: เขียน login view เอง vs ใช้ `LoginView`

| ประเด็น | เขียนเอง (`authenticate()` + `login()`) | Built-in `LoginView` |
|---|---|---|
| จำนวนโค้ดที่ต้องเขียน | มาก (ต้องจัดการ GET/POST, error, redirect เอง) | น้อยมาก (แค่ 1 บรรทัดใน `urls.py`) |
| การจัดการ `next` parameter | ต้องเขียนเอง (และเสี่ยงพลาดเรื่อง Open Redirect — ขั้นตอนที่ 307) | จัดการให้อัตโนมัติและปลอดภัยอยู่แล้ว |
| Rate limiting / throttling | ต้องเพิ่มเอง | ไม่มีในตัว (ต้องเพิ่มเองเช่นกัน แต่โครงสร้างพร้อมกว่า) |
| Custom logic พิเศษ (เช่น ส่ง email แจ้งเตือนตอน login) | ทำได้อิสระเต็มที่ | ทำได้ผ่านการ override `form_valid()` ของ `LoginView` |
| แนะนำสำหรับ | เพื่อการศึกษา / กรณีต้องการ flow ที่ต่างจากมาตรฐานมาก | งานจริงเกือบทั้งหมด (ค่าเริ่มต้นของหลักสูตรนี้) |

### 303.7 Override `LoginView` เพื่อ custom logic โดยไม่ต้องเขียนใหม่ทั้งหมด

```python
# accounts/views.py
from django.contrib.auth import views as auth_views
from django.contrib import messages


class CustomLoginView(auth_views.LoginView):
    template_name = "registration/login.html"

    def form_valid(self, form):
        messages.success(self.request, f"ยินดีต้อนรับกลับมา, {form.get_user().username}!")
        return super().form_valid(form)
```

นี่คือตัวอย่างที่ดีของหลักการ "ใช้ของที่มีอยู่แล้วให้มากที่สุด แล้ว override เฉพาะ
จุดที่ต้องการเพิ่มเติม" ตามแนวทาง Mixin/CBV ที่เรียนไปใน Part 024

---

## ขั้นตอนที่ 304: `UserCreationForm` และหน้า signup สมัครสมาชิกจริง

### 304.1 `UserCreationForm` คืออะไร

Django มี `ModelForm` สำเร็จรูปชื่อ `UserCreationForm` ที่สร้างมาเพื่อ "สมัครสมาชิก
ใหม่" โดยเฉพาะ อยู่ที่ `django.contrib.auth.forms`:

```python
# django/contrib/auth/forms.py (โครงสร้างจริงย่อเพื่อความเข้าใจ)
class UserCreationForm(forms.ModelForm):
    password1 = forms.CharField(label="Password", widget=forms.PasswordInput)
    password2 = forms.CharField(label="Password confirmation", widget=forms.PasswordInput)

    class Meta:
        model = User
        fields = ("username",)

    def clean_password2(self):
        password1 = self.cleaned_data.get("password1")
        password2 = self.cleaned_data.get("password2")
        if password1 and password2 and password1 != password2:
            raise forms.ValidationError("รหัสผ่านทั้งสองช่องไม่ตรงกัน")
        return password2

    def _post_clean(self):
        super()._post_clean()
        password = self.cleaned_data.get("password2")
        if password:
            try:
                password_validation.validate_password(password, self.instance)
            except forms.ValidationError as error:
                self.add_error("password2", error)

    def save(self, commit=True):
        user = super().save(commit=False)
        user.set_password(self.cleaned_data["password1"])   # hash password ก่อนบันทึก
        if commit:
            user.save()
        return user
```

จุดสำคัญที่ต้องสังเกต:

- ฟอร์มนี้มี field `password1`/`password2` เพื่อให้ผู้ใช้ยืนยันรหัสผ่านซ้ำ (ป้องกัน
  พิมพ์ผิด) และมี `clean_password2()` เช็คว่าทั้งสองช่องตรงกันหรือไม่
- `_post_clean()` เรียก `password_validation.validate_password()` ซึ่งไปตรวจสอบ
  กับ `AUTH_PASSWORD_VALIDATORS` ใน `settings.py` (เช่น ความยาวขั้นต่ำ, ไม่ใช่
  รหัสผ่านที่พบบ่อย ฯลฯ — จะเจาะลึกเต็มรูปแบบใน Part 035)
- `save()` เรียก `user.set_password()` ที่ทำการ **hash** รหัสผ่านก่อนบันทึกเสมอ
  **ไม่เคยเก็บ plaintext ลงฐานข้อมูล**

### 304.2 สร้างหน้า signup ด้วย `CreateView` + `UserCreationForm`

```python
# accounts/views.py
from django.contrib.auth.forms import UserCreationForm
from django.urls import reverse_lazy
from django.views.generic import CreateView


class SignUpView(CreateView):
    form_class = UserCreationForm
    template_name = "registration/signup.html"
    success_url = reverse_lazy("login")
```

```python
# accounts/urls.py
from django.urls import path

from . import views

urlpatterns = [
    path("signup/", views.SignUpView.as_view(), name="signup"),
]
```

```html
<!-- templates/registration/signup.html -->
{% extends "base.html" %}

{% block content %}
<h1>สมัครสมาชิก</h1>

<form method="post">
    {% csrf_token %}
    {{ form.as_p }}
    <button type="submit">สมัครสมาชิก</button>
</form>

<p>มีบัญชีอยู่แล้ว? <a href="{% url 'login' %}">เข้าสู่ระบบ</a></p>
{% endblock %}
```

โค้ดข้างต้นทำงานได้ทันทีเพราะ `UserCreationForm` เป็น `ModelForm` ที่ผูกกับ
`django.contrib.auth.models.User` อยู่แล้ว เมื่อสมัครสำเร็จ Django จะสร้าง
`User` ใหม่ (พร้อม hash password ให้เอง) แล้ว redirect ไปหน้า login

### 304.3 เพิ่ม `email` เข้าไปในฟอร์ม (subclass `UserCreationForm`)

`UserCreationForm` ค่าเริ่มต้นมีแค่ `username`, `password1`, `password2` — ในงานจริง
เกือบทุกระบบต้องการเก็บ `email` ตั้งแต่ตอนสมัครด้วย ทำได้โดย subclass:

```python
# accounts/forms.py
from django import forms
from django.contrib.auth.forms import UserCreationForm
from django.contrib.auth.models import User


class CustomUserCreationForm(UserCreationForm):
    email = forms.EmailField(required=True, label="อีเมล")

    class Meta(UserCreationForm.Meta):
        model = User
        fields = ("username", "email")   # password1/password2 ถูกเพิ่มให้อัตโนมัติจาก parent

    def clean_email(self):
        email = self.cleaned_data["email"]
        # เนื่องจาก email ไม่ unique โดยค่าเริ่มต้น (ขั้นตอนที่ 301.2) ต้องเช็คเอง
        if User.objects.filter(email__iexact=email).exists():
            raise forms.ValidationError("อีเมลนี้ถูกใช้สมัครสมาชิกไปแล้ว")
        return email

    def save(self, commit=True):
        user = super().save(commit=False)
        user.email = self.cleaned_data["email"]
        if commit:
            user.save()
        return user
```

```python
# accounts/views.py
from django.urls import reverse_lazy
from django.views.generic import CreateView

from .forms import CustomUserCreationForm


class SignUpView(CreateView):
    form_class = CustomUserCreationForm
    template_name = "registration/signup.html"
    success_url = reverse_lazy("login")
```

> **ทำไมต้องเช็ค `email__iexact` เอง**: อย่างที่อธิบายในขั้นตอนที่ 301.2, field
> `email` ของ `User` ไม่ได้ตั้ง `unique=True` โดยค่าเริ่มต้น การเช็คซ้ำในระดับฟอร์ม
> (`clean_email()`) จึงเป็นวิธีเดียวที่ป้องกันอีเมลซ้ำได้ในตอนที่ยังใช้ default
> `User` model อยู่ — ถ้าต้องการบังคับ unique ที่ระดับฐานข้อมูลจริง ๆ ต้องใช้
> Custom User Model ตาม Part 032

### 304.4 Auto-login หลังสมัครสำเร็จ (UX ที่ดีกว่า)

ผู้ใช้ส่วนใหญ่คาดหวังว่าเมื่อสมัครสมาชิกเสร็จแล้วจะ **login ให้อัตโนมัติทันที**
โดยไม่ต้องกรอกฟอร์ม login อีกรอบ ทำได้โดย override `form_valid()`:

```python
# accounts/views.py
from django.contrib.auth import login
from django.urls import reverse_lazy
from django.views.generic import CreateView

from .forms import CustomUserCreationForm


class SignUpView(CreateView):
    form_class = CustomUserCreationForm
    template_name = "registration/signup.html"
    success_url = reverse_lazy("blog:post-list")

    def form_valid(self, form):
        response = super().form_valid(form)   # form.save() ถูกเรียก, self.object คือ user ใหม่
        login(self.request, self.object)      # login ให้อัตโนมัติทันที
        return response
```

สังเกตรูปแบบ "เรียก `super().form_valid(form)` ก่อนแล้วค่อยทำงานเพิ่มเติม" ซึ่งเป็น
pattern เดียวกับที่เรียนไปใน Part 023 ขั้นตอนที่ 225.3 (การ log หลัง save)

### 304.5 ตารางสรุปทางเลือกในการสร้างฟอร์มสมัครสมาชิก

| แนวทาง | เหมาะกับ |
|---|---|
| ใช้ `UserCreationForm` ตรง ๆ | Prototype เร็ว ๆ ไม่สนใจ email |
| Subclass เพิ่ม `email` | งานจริงเกือบทั้งหมดที่ยังใช้ default `User` model |
| Custom User Model (Part 032) + ฟอร์มของตัวเอง | ระบบที่ต้องการ field พิเศษตั้งแต่ต้น เช่น เบอร์โทร, avatar, role |
| django-allauth (ขั้นตอนที่ 309) | ต้องการ social login (Google, GitHub) ร่วมด้วย |

---

## ขั้นตอนที่ 305: `request.user`, `AnonymousUser`, การเช็ค `is_authenticated`

### 305.1 `request.user` มาจากไหน

`request.user` **ไม่ได้เป็นส่วนหนึ่งของ `HttpRequest` โดยธรรมชาติ** แต่ถูกเติมเข้าไป
โดย `AuthenticationMiddleware` ที่เห็นใน `MIDDLEWARE` ตั้งแต่ขั้นตอนที่ 301.1:

```python
# django/contrib/auth/middleware.py (โค้ดจริงย่อเพื่อความเข้าใจ)
class AuthenticationMiddleware(MiddlewareMixin):
    def process_request(self, request):
        request.user = SimpleLazyObject(lambda: get_user(request))
```

`SimpleLazyObject` หมายความว่า Django **ยังไม่ query ฐานข้อมูลทันที** ตอน
middleware ทำงาน แต่จะ query จริง ๆ ก็ต่อเมื่อโค้ดของคุณเข้าถึง `request.user`
เป็นครั้งแรก (เช่น `request.user.username`) — เป็นเทคนิค **lazy evaluation**
เพื่อประสิทธิภาพ (ถ้า view ไหนไม่เคยแตะ `request.user` เลย ก็ไม่ต้องเสีย query
ฐานข้อมูลโดยเปล่าประโยชน์)

### 305.2 `AnonymousUser`: ตัวแทนของ "ยังไม่ login"

ถ้าผู้ใช้ยังไม่ login (ไม่มี session ที่ถูกต้อง) `request.user` จะไม่ใช่ `None`
แต่เป็น instance ของ `django.contrib.auth.models.AnonymousUser` แทน — นี่คือ
การออกแบบที่ชาญฉลาดของ Django ที่ทำให้โค้ดของคุณ **ไม่ต้องเช็ค `None` ทุกที่**
ก่อนเรียกใช้ attribute ของ user

```python
# django/contrib/auth/models.py (โค้ดจริงย่อเพื่อความเข้าใจ)
class AnonymousUser:
    id = None
    pk = None
    username = ""
    is_staff = False
    is_active = False
    is_superuser = False

    def __str__(self):
        return "AnonymousUser"

    @property
    def is_authenticated(self):
        return False

    @property
    def is_anonymous(self):
        return True

    def save(self):
        raise NotImplementedError("Django doesn't provide a DB representation for AnonymousUser.")

    def has_perm(self, perm, obj=None):
        return False

    def has_perms(self, perm_list, obj=None):
        return False
```

### 305.3 ตารางเปรียบเทียบ `User` (login แล้ว) กับ `AnonymousUser` (ยังไม่ login)

| Attribute/Method | `User` (login แล้ว) | `AnonymousUser` (ยังไม่ login) |
|---|---|---|
| `is_authenticated` | `True` เสมอ | `False` เสมอ |
| `is_anonymous` | `False` เสมอ | `True` เสมอ |
| `username` | ค่าจริงจากฐานข้อมูล | `""` (string ว่าง) |
| `is_active` | ตามค่าจริงในฐานข้อมูล | `False` เสมอ |
| `pk` / `id` | ค่าจริงจากฐานข้อมูล | `None` |
| `has_perm(...)` | ตรวจสอบ permission จริง | คืน `False` เสมอ (ไม่มีสิทธิ์อะไรเลย) |
| `save()` | บันทึกลงฐานข้อมูลได้ปกติ | โยน `NotImplementedError` (ไม่มีอยู่จริงในฐานข้อมูล) |

> **กฎเหล็ก**: อย่าเช็คสถานะ login ด้วย `if request.user:` เพราะ `AnonymousUser`
> ก็เป็น object ที่ truthy เหมือนกัน (`bool(AnonymousUser())` เป็น `True`) เช็คแบบนี้
> จะให้ผลลัพธ์ `True` เสมอไม่ว่าจะ login หรือไม่! **ต้องเช็คด้วย
> `request.user.is_authenticated` เท่านั้น** ซึ่งเป็น property ที่ถูกออกแบบมาให้
> ตอบคำถามนี้โดยตรง ไม่กำกวม

### 305.4 การใช้งานจริงใน view

```python
# blog/views.py
from django.shortcuts import render


def homepage(request):
    if request.user.is_authenticated:
        greeting = f"ยินดีต้อนรับกลับมา, {request.user.username}!"
        my_draft_count = request.user.posts.filter(is_published=False).count()
    else:
        greeting = "ยินดีต้อนรับ! กรุณาเข้าสู่ระบบเพื่อเขียนบทความ"
        my_draft_count = 0

    return render(request, "blog/home.html", {
        "greeting": greeting,
        "my_draft_count": my_draft_count,
    })
```

สังเกตว่าโค้ดนี้เขียนได้อย่างปลอดภัยโดยไม่ต้องเช็ค `None` เลยแม้แต่ครั้งเดียว
เพราะไม่ว่า `request.user` จะเป็น `User` จริงหรือ `AnonymousUser` มันก็ตอบ
`.is_authenticated` ได้เสมอโดยไม่ error

### 305.5 การใช้งานใน queryset filtering (เตรียมพื้นฐานสำหรับ Part 038)

```python
class MyDraftsListView(ListView):
    template_name = "blog/my_drafts.html"
    context_object_name = "posts"

    def get_queryset(self):
        if not self.request.user.is_authenticated:
            return Post.objects.none()   # คืน queryset ว่างเปล่า ไม่ error
        return Post.objects.filter(author=self.request.user, is_published=False)
```

`Post.objects.none()` คืน `EmptyQuerySet` ที่ปลอดภัย ไม่มีการ query ฐานข้อมูลจริง
เป็นวิธีมาตรฐานในการจัดการกรณี "ยังไม่ login" โดยไม่ต้อง raise exception

---

## ขั้นตอนที่ 306: ตั้งค่า URL auth ทั้งระบบผ่าน `django.contrib.auth.urls` เทียบกับเขียนเอง

### 306.1 `django.contrib.auth.urls` คืออะไร

แทนที่จะเขียน `path("login/", ...)`, `path("logout/", ...)`, `path("password_change/", ...)`
ฯลฯ ทีละบรรทัดเอง Django เตรียม URLconf สำเร็จรูปที่ครอบคลุม auth-related URL
ทั้งหมดไว้ให้แล้วในโมดูล `django.contrib.auth.urls`

```python
# config/urls.py
from django.contrib import admin
from django.urls import include, path

urlpatterns = [
    path("admin/", admin.site.urls),
    path("accounts/", include("django.contrib.auth.urls")),
    path("accounts/signup/", views.SignUpView.as_view(), name="signup"),  # signup ต้องเพิ่มเอง
    path("blog/", include("blog.urls")),
]
```

### 306.2 ตาราง URL patterns ทั้งหมดที่ `django.contrib.auth.urls` ให้มา

```python
# django/contrib/auth/urls.py (โค้ดจริงทั้งหมด)
urlpatterns = [
    path("login/", LoginView.as_view(), name="login"),
    path("logout/", LogoutView.as_view(), name="logout"),
    path("password_change/", PasswordChangeView.as_view(), name="password_change"),
    path("password_change/done/", PasswordChangeDoneView.as_view(), name="password_change_done"),
    path("password_reset/", PasswordResetView.as_view(), name="password_reset"),
    path("password_reset/done/", PasswordResetDoneView.as_view(), name="password_reset_done"),
    path("reset/<uidb64>/<token>/", PasswordResetConfirmView.as_view(), name="password_reset_confirm"),
    path("reset/done/", PasswordResetCompleteView.as_view(), name="password_reset_complete"),
]
```

เมื่อรวมกับ `path("accounts/", include(...))` แล้ว จะได้ URL เต็มดังนี้:

| URL name | Path เต็ม | CBV ที่ใช้ | จะเจาะลึกใน Part |
|---|---|---|---|
| `login` | `/accounts/login/` | `LoginView` | Part นี้ (303, 306) |
| `logout` | `/accounts/logout/` | `LogoutView` | Part นี้ (307) |
| `password_change` | `/accounts/password_change/` | `PasswordChangeView` | Part 035 |
| `password_change_done` | `/accounts/password_change/done/` | `PasswordChangeDoneView` | Part 035 |
| `password_reset` | `/accounts/password_reset/` | `PasswordResetView` | Part 035 |
| `password_reset_done` | `/accounts/password_reset/done/` | `PasswordResetDoneView` | Part 035 |
| `password_reset_confirm` | `/accounts/reset/<uidb64>/<token>/` | `PasswordResetConfirmView` | Part 035 |
| `password_reset_complete` | `/accounts/reset/done/` | `PasswordResetCompleteView` | Part 035 |

### 306.3 Template ที่แต่ละ view ต้องการ (convention `registration/`)

CBV เหล่านี้ทั้งหมดมองหา template ในโฟลเดอร์ `registration/` โดยค่าเริ่มต้น:

| View | Template ที่ต้องสร้าง |
|---|---|
| `LoginView` | `registration/login.html` |
| `LoggedOutView` (จาก `LogoutView`) | `registration/logged_out.html` |
| `PasswordChangeView` | `registration/password_change_form.html` |
| `PasswordChangeDoneView` | `registration/password_change_done.html` |
| `PasswordResetView` | `registration/password_reset_form.html` |
| `PasswordResetDoneView` | `registration/password_reset_done.html` |
| `PasswordResetConfirmView` | `registration/password_reset_confirm.html` |
| `PasswordResetCompleteView` | `registration/password_reset_complete.html` |

ตั้งชื่อโฟลเดอร์ `registration/` เป็น convention ตายตัวของ Django (ไม่ใช่ตั้งชื่อ
ตามแอปของคุณ) ต้องวางไว้ในโฟลเดอร์ template ระดับโปรเจกต์ (ตามที่ตั้งค่า `DIRS`
ใน `TEMPLATES` — ทบทวนได้จาก Part 008):

```
myproject/
└── templates/
    └── registration/
        ├── login.html
        └── logged_out.html
```

### 306.4 `LOGIN_REDIRECT_URL`: ไปไหนหลัง login สำเร็จ (เมื่อไม่มี `next`)

```python
# config/settings.py
LOGIN_REDIRECT_URL = "blog:post-list"   # ใช้ URL name ได้
```

ถ้าไม่ตั้งค่านี้ ค่าเริ่มต้นของ Django คือ `/accounts/profile/` ซึ่งแทบไม่มีโปรเจกต์
ไหนใช้จริง (มักลืมตั้งค่านี้แล้วงงว่าทำไม login แล้วเจอ 404!) **ควรตั้งค่านี้เสมอ
ตั้งแต่ตอนเริ่มโปรเจกต์**

### 306.5 ตารางเปรียบเทียบ: `include("django.contrib.auth.urls")` vs เขียน URL เอง

| ประเด็น | `include("django.contrib.auth.urls")` | เขียน `path()` เองทั้งหมด |
|---|---|---|
| จำนวนบรรทัดโค้ด | 1 บรรทัด ได้ครบ 8 URL | ต้องเขียนเองทีละ URL |
| ความสอดคล้องกับ Django convention | สูง (ใช้ URL name มาตรฐานที่ package อื่น ๆ เช่น django-allauth คาดหวัง) | ขึ้นกับวินัยของทีม |
| ความยืดหยุ่นในการเปลี่ยน path | ทำได้ผ่าน `path("custom-prefix/", include(...))` | ยืดหยุ่นเต็มที่ |
| ความยืดหยุ่นในการเปลี่ยน View/Template | ต้องเขียน `urls.py` เองแทน include เพื่อ override เฉพาะบาง view | ยืดหยุ่นเต็มที่ |
| เหมาะกับ | โปรเจกต์ส่วนใหญ่ (จุดเริ่มต้นที่แนะนำ) | ระบบที่ auth flow ต่างจากมาตรฐานมาก (เช่น login ผ่าน OTP เท่านั้น) |

### 306.6 Override เฉพาะบาง URL โดยยังใช้ `include()` ส่วนที่เหลือ

เทคนิคที่ใช้บ่อยในงานจริง: ใช้ `django.contrib.auth.urls` เป็นฐาน แต่แทนที่บาง URL
ด้วย view ของตัวเอง โดยวาง `path()` ของเราไว้ **ก่อน** `include()` (ทบทวนหลักการ
"จากบนลงล่าง" จาก Part 006 ขั้นตอนที่ 52 — Django จับคู่ URL pattern แรกที่ตรงกัน
ก่อนเสมอ):

```python
# config/urls.py
from django.urls import include, path

from accounts.views import CustomLoginView, SignUpView

urlpatterns = [
    path("accounts/login/", CustomLoginView.as_view(), name="login"),  # override เฉพาะ login
    path("accounts/signup/", SignUpView.as_view(), name="signup"),
    path("accounts/", include("django.contrib.auth.urls")),  # ที่เหลือ (logout, password_reset ฯลฯ) ใช้ของ Django
]
```

---

## ขั้นตอนที่ 307: `LogoutView`, พารามิเตอร์ `next` และความเสี่ยง Open Redirect

### 307.1 `LogoutView` ทำงานอย่างไร

```python
# blog/urls.py หรือใช้จาก include("django.contrib.auth.urls") ก็ได้
from django.contrib.auth.views import LogoutView
from django.urls import path

urlpatterns = [
    path("logout/", LogoutView.as_view(), name="logout"),
]
```

`LogoutView` เรียก `django.contrib.auth.logout(request)` เบื้องหลัง ซึ่งทำสิ่ง
ตรงข้ามกับ `login()`:

```python
# django/contrib/auth/__init__.py (โค้ดจริงย่อเพื่อความเข้าใจ)
def logout(request):
    user = getattr(request, "user", None)
    if not getattr(user, "is_authenticated", True):
        user = None
    user_logged_out.send(sender=user.__class__, request=request, user=user)

    request.session.flush()   # ล้างข้อมูล session ทั้งหมด (ไม่ใช่แค่ auth-related keys)
    if hasattr(request, "user"):
        from django.contrib.auth.models import AnonymousUser
        request.user = AnonymousUser()
```

สังเกตว่า `logout()` เรียก `request.session.flush()` ซึ่งล้าง **session ทั้งหมด**
ไม่ใช่แค่ข้อมูล auth เท่านั้น — ถ้าคุณเก็บข้อมูลอื่นไว้ใน session (เช่น ตะกร้าสินค้า
ของผู้ใช้ที่ยังไม่ login) ข้อมูลนั้นจะหายไปด้วยเมื่อ logout (จะเจาะลึกเรื่อง session
เต็มรูปแบบใน Part 034)

### 307.2 ตั้งค่าปลายทางหลัง logout ด้วย `LOGOUT_REDIRECT_URL`

```python
# config/settings.py
LOGOUT_REDIRECT_URL = "blog:post-list"
```

ถ้าไม่ตั้งค่านี้ Django รุ่นปัจจุบันจะ render template `registration/logged_out.html`
แทนการ redirect (พฤติกรรมนี้ต่างจาก `login`/`LOGIN_REDIRECT_URL` ที่ redirect
เสมอ — เป็นรายละเอียดที่ Django เปลี่ยนมาหลายเวอร์ชันแล้วให้ยืดหยุ่นขึ้น)

```html
<!-- templates/registration/logged_out.html -->
{% extends "base.html" %}

{% block content %}
<h1>ออกจากระบบเรียบร้อยแล้ว</h1>
<p><a href="{% url 'login' %}">เข้าสู่ระบบอีกครั้ง</a></p>
{% endblock %}
```

### 307.3 พารามิเตอร์ `next` ใน `LogoutView`

`LogoutView` รองรับ query string `?next=` เพื่อระบุว่าหลัง logout แล้วให้ไปหน้าไหน
แทนที่จะใช้ `LOGOUT_REDIRECT_URL`:

```html
<a href="{% url 'logout' %}?next={% url 'blog:post-list' %}">ออกจากระบบ</a>
```

Django ในเวอร์ชันปัจจุบัน (5.x) **บังคับให้ logout ผ่าน `POST` เท่านั้น** ไม่รองรับ
`GET` อีกต่อไป (เปลี่ยนแปลงมาจาก Django รุ่นเก่า) ด้วยเหตุผลเดียวกับที่อธิบายไปใน
Part 023 ขั้นตอนที่ 223.4 เรื่อง `DeleteView`: ลิงก์ `GET` ธรรมดาเสี่ยงต่อการถูก
web crawler หรือ `<img>` แอบเรียกโดยไม่ตั้งใจ **ต้องใช้ฟอร์มเสมอ**:

```html
<!-- templates/base.html -->
<form method="post" action="{% url 'logout' %}" style="display: inline;">
    {% csrf_token %}
    <button type="submit">ออกจากระบบ</button>
</form>
```

### 307.4 ความเสี่ยง Open Redirect: ทำไม `next` ที่ไม่ตรวจสอบถึงอันตราย

**Open Redirect** คือช่องโหว่ที่เกิดเมื่อระบบยอม redirect ผู้ใช้ไปยัง URL
**ภายนอกโดเมนของตัวเอง** ตามค่าที่ผู้ใช้ (หรือผู้โจมตี) ควบคุมได้ผ่าน query string
โดยไม่ตรวจสอบก่อน ลองจินตนาการโค้ดที่ผิดพลาดแบบนี้:

```python
# ตัวอย่างโค้ดที่อันตราย! อย่าทำแบบนี้
def bad_logout_view(request):
    logout(request)
    next_url = request.GET.get("next", "/")
    return redirect(next_url)   # ไม่ตรวจสอบ next_url เลย!
```

ผู้โจมตีสามารถส่งลิงก์แบบนี้ให้เหยื่อคลิก:

```
https://yourdjango-site.com/accounts/logout/?next=https://evil-phishing-site.com/fake-login
```

เหยื่อเห็น URL เป็นโดเมนที่เชื่อถือได้ (`yourdjango-site.com`) จึงคลิกโดยไม่สงสัย
แต่หลัง logout ระบบจะ redirect ไปเว็บปลอมที่ทำหน้าตาเหมือน login จริงทุกประการ
เพื่อหลอกขโมย username/password — เป็นเทคนิคการโจมตีแบบ **Phishing ผสม Open
Redirect** ที่พบได้จริงในระบบที่ไม่ได้ป้องกัน

### 307.5 Django ป้องกันให้อัตโนมัติผ่าน `url_has_allowed_host_and_scheme()`

ข่าวดีคือ `LoginView`/`LogoutView` ที่มากับ Django **มีการป้องกันเรื่องนี้ในตัวอยู่
แล้ว** ผ่านฟังก์ชัน `url_has_allowed_host_and_scheme()`:

```python
# django/contrib/auth/views.py (โค้ดจริงย่อเพื่อความเข้าใจ)
from django.utils.http import url_has_allowed_host_and_scheme


class LoginView(RedirectURLMixin, FormView):
    def get_success_url(self):
        url = self.get_redirect_url()
        return url or resolve_url(settings.LOGIN_REDIRECT_URL)

    def get_redirect_url(self):
        redirect_to = self.request.POST.get(
            self.redirect_field_name, self.request.GET.get(self.redirect_field_name, "")
        )
        url_is_safe = url_has_allowed_host_and_scheme(
            url=redirect_to,
            allowed_hosts=self.get_success_url_allowed_hosts(),
            require_https=self.request.is_secure(),
        )
        return redirect_to if url_is_safe else ""
```

ถ้า `next` ชี้ไปโดเมนอื่นที่ไม่อยู่ใน allowed hosts ฟังก์ชันนี้จะคืนค่า `False`
และ `LoginView`/`LogoutView` จะ **เพิกเฉยค่า `next` นั้นทันที** แล้ว fallback ไปใช้
`LOGIN_REDIRECT_URL`/`LOGOUT_REDIRECT_URL` แทน — นี่คือเหตุผลสำคัญที่สุดที่ควร
**ใช้ built-in `LoginView`/`LogoutView` แทนการเขียน redirect เองเสมอ**

### 307.6 ถ้าจำเป็นต้องเขียน redirect-after-login เองจริง ๆ

ในกรณีที่คุณต้องเขียน view จัดการ `next` เอง (เช่นในขั้นตอนที่ 303.2) **ต้อง**
เรียกใช้ฟังก์ชันเดียวกันนี้เพื่อ validate เสมอ:

```python
# accounts/views.py
from django.contrib.auth import authenticate, login
from django.shortcuts import redirect, render
from django.utils.http import url_has_allowed_host_and_scheme


def manual_login_view(request):
    next_url = request.POST.get("next") or request.GET.get("next", "")

    if request.method == "POST":
        username = request.POST.get("username")
        password = request.POST.get("password")
        user = authenticate(request, username=username, password=password)

        if user is not None:
            login(request, user)

            # ต้อง validate next_url ก่อนใช้เสมอ ห้าม redirect(next_url) ตรง ๆ เด็ดขาด
            if next_url and url_has_allowed_host_and_scheme(
                url=next_url,
                allowed_hosts={request.get_host()},
                require_https=request.is_secure(),
            ):
                return redirect(next_url)
            return redirect("blog:post-list")

    return render(request, "registration/login.html", {"next": next_url})
```

### 307.7 ตารางสรุปกฎความปลอดภัยเรื่อง Open Redirect

| กฎ | เหตุผล |
|---|---|
| ห้าม `redirect(request.GET.get("next"))` ตรง ๆ เด็ดขาด | ผู้ใช้/ผู้โจมตีควบคุมค่านี้ได้เต็มที่ |
| ใช้ `url_has_allowed_host_and_scheme()` ตรวจสอบก่อนเสมอ | เป็นฟังก์ชันมาตรฐานของ Django ที่ผ่านการตรวจสอบความปลอดภัยมาแล้ว |
| ระบุ `allowed_hosts` ให้ตรงกับโดเมนจริงของระบบ | ป้องกัน redirect ไปโดเมนอื่นแม้จะเป็น URL ที่ "ดูปลอดภัย" |
| ใช้ built-in `LoginView`/`LogoutView` เมื่อทำได้ | ลดความเสี่ยงจากการเขียนเองแล้วลืมตรวจสอบ |
| logout ต้องใช้ `POST` ผ่านฟอร์ม ไม่ใช่ลิงก์ `GET` | ป้องกัน CSRF และการ logout โดยไม่ตั้งใจ (ขั้นตอนที่ 307.3) |

---

## ขั้นตอนที่ 308: เช็คสถานะ login ใน Template — `{% if user.is_authenticated %}`

### 308.1 `user` ใน template มาจากไหน

คุณอาจสงสัยว่าทำไมใน template สามารถเขียน `{{ user.username }}` ได้เลยโดยไม่ต้อง
ส่ง `request.user` เข้า context ทุกครั้ง — คำตอบคือ **context processor**
`django.contrib.auth.context_processors.auth` ที่มากับ `TEMPLATES` โดยค่าเริ่มต้น
(ทบทวนแนวคิด context processor แบบเต็มได้ที่ Part 030):

```python
# config/settings.py
TEMPLATES = [
    {
        "BACKEND": "django.template.backends.django.DjangoTemplates",
        "DIRS": [BASE_DIR / "templates"],
        "APP_DIRS": True,
        "OPTIONS": {
            "context_processors": [
                "django.template.context_processors.debug",
                "django.template.context_processors.request",
                "django.contrib.auth.context_processors.auth",  # <- ตัวนี้เติม `user` ให้ทุก template
                "django.contrib.messages.context_processors.messages",
            ],
        },
    },
]
```

Context processor นี้เติมตัวแปร `user` (เท่ากับ `request.user`) และ `perms`
(สำหรับเช็ค permission — จะเจาะลึกใน Part 033) เข้าไปใน context ของ **ทุก
template ที่ render ด้วย `render()`, `RequestContext`, หรือ CBV มาตรฐาน**
โดยอัตโนมัติ

> **ข้อควรระวัง**: ถ้าลบ `django.contrib.auth.context_processors.auth` ออกจาก
> `context_processors`, ตัวแปร `user` จะหายไปจากทุก template ทันที และ
> `{% if user.is_authenticated %}` จะ error หรือได้ผลลัพธ์ผิดเพราะ `user` ไม่มีค่า

### 308.2 ตัวอย่างการใช้งานจริงใน `base.html`

```html
<!-- templates/base.html -->
<!DOCTYPE html>
<html lang="th">
<head>
    <meta charset="UTF-8">
    <title>{% block title %}My Blog{% endblock %}</title>
</head>
<body>
    <nav>
        <a href="{% url 'blog:post-list' %}">หน้าแรก</a>

        {% if user.is_authenticated %}
            <span>สวัสดี, {{ user.username }}!</span>
            <a href="{% url 'blog:post-create' %}">เขียนบทความใหม่</a>
            <a href="{% url 'password_change' %}">เปลี่ยนรหัสผ่าน</a>
            <form method="post" action="{% url 'logout' %}" style="display: inline;">
                {% csrf_token %}
                <button type="submit">ออกจากระบบ</button>
            </form>
        {% else %}
            <a href="{% url 'login' %}">เข้าสู่ระบบ</a>
            <a href="{% url 'signup' %}">สมัครสมาชิก</a>
        {% endif %}
    </nav>

    {% block content %}{% endblock %}
</body>
</html>
```

### 308.3 เช็ค `is_staff`/`is_superuser` ใน template

```html
{% if user.is_authenticated %}
    <p>สวัสดี, {{ user.get_full_name|default:user.username }}</p>

    {% if user.is_staff %}
        <a href="{% url 'admin:index' %}">ไปที่ Admin Panel</a>
    {% endif %}

    {% if user.is_superuser %}
        <span class="badge">ผู้ดูแลระบบสูงสุด</span>
    {% endif %}
{% endif %}
```

### 308.4 ทำไมใน template ต้องเรียก `is_authenticated` โดย**ไม่ใส่วงเล็บ**

จุดที่มือใหม่มักสับสน: ใน Python ปกติ `is_authenticated` เป็น **property** ไม่ใช่
method (ตามที่เห็นในขั้นตอนที่ 305.2) ดังนั้นเวลาเรียกใน Python ต้องเขียน
`request.user.is_authenticated` (ไม่มีวงเล็บ) — และใน Django Template Language
(DTL) การเรียก attribute/method จะใช้ syntax เดียวกันหมดคือ `{{ obj.attr }}`
หรือ `{% if obj.attr %}` (ทบทวนจาก Part 008) ทำให้ `{% if user.is_authenticated %}`
เขียนได้ถูกต้องตามธรรมชาติโดยไม่ต้องกังวลเรื่องวงเล็บเหมือนใน Python เลย

### 308.5 ตารางสรุปตัวแปร auth ที่ใช้ได้ในทุก template

| ตัวแปรใน template | เทียบเท่ากับ | ใช้ทำอะไร |
|---|---|---|
| `user` | `request.user` | ตรวจสอบสถานะ/ข้อมูลผู้ใช้ปัจจุบัน |
| `user.is_authenticated` | — | `True`/`False` ว่า login อยู่หรือไม่ |
| `user.username` | — | ชื่อผู้ใช้ (ว่างเปล่าถ้ายังไม่ login) |
| `user.get_full_name` | — | ชื่อเต็ม (first_name + last_name) |
| `user.is_staff` / `user.is_superuser` | — | สิทธิ์ระดับ admin |
| `perms` | `request.user`'s permissions | เช็ค permission แบบ `{% if perms.blog.add_post %}` (จะเจาะลึกใน Part 033) |

---

## ขั้นตอนที่ 309: ภาพรวมสั้น ๆ ของ django-allauth เทียบกับ built-in auth

### 309.1 django-allauth คืออะไร

**django-allauth** คือ third-party package ที่ได้รับความนิยมสูงมากในระบบ Django
สำหรับจัดการ authentication แบบครบวงจร ครอบคลุมทั้ง:

- Local authentication (username/email + password เหมือน built-in auth)
- **Social Authentication**: login ผ่าน Google, GitHub, Facebook, Apple ฯลฯ
  (เรียกว่า OAuth/OAuth2 provider)
- Email verification workflow ที่ครบวงจรกว่า built-in
- Multi-factor authentication (บางส่วน หรือใช้ร่วมกับ django-allauth-mfa)

```bash
pip install django-allauth
```

### 309.2 ตารางเปรียบเทียบ Built-in `django.contrib.auth` กับ `django-allauth`

| คุณสมบัติ | Built-in `django.contrib.auth` | `django-allauth` |
|---|---|---|
| Username/Password login | ✅ มีในตัว | ✅ มีในตัว (ปรับแต่งได้มากกว่า) |
| Social Login (Google, GitHub, ...) | ❌ ไม่มี ต้องเขียน OAuth flow เอง | ✅ รองรับ provider นับร้อยตัวสำเร็จรูป |
| Email verification | ⚠️ ต้องเขียนเอง (มี `PasswordResetView` เป็นฐาน) | ✅ มีในตัว ครบวงจร |
| Login ด้วย email แทน username | ⚠️ ต้องปรับแต่ง authentication backend เอง | ✅ ตั้งค่าผ่าน setting ได้ทันที |
| Rate limiting การพยายาม login | ❌ ไม่มีในตัว | ✅ มีในตัว (ป้องกัน brute-force) |
| ความซับซ้อนในการติดตั้ง | ต่ำ (มีอยู่แล้วไม่ต้องติดตั้งเพิ่ม) | ปานกลาง-สูง (ต้องตั้งค่า provider, callback URL ฯลฯ) |
| ความยืดหยุ่นในการปรับแต่ง UI/Flow | สูง (ควบคุมทุกอย่างเอง) | ปานกลาง (ต้อง override template/adapter ของ allauth) |
| เหมาะกับ | โปรเจกต์ที่ต้องการแค่ username/password ธรรมดา | โปรเจกต์ที่ต้องการ "Sign in with Google" หรือ social login |

### 309.3 ตัวอย่างหน้าตาการตั้งค่า django-allauth (ภาพรวมเท่านั้น)

```python
# config/settings.py (ตัวอย่างโครงร่างคร่าว ๆ เท่านั้น — จะเจาะลึกจริงใน Part 036)
INSTALLED_APPS = [
    # ...
    "django.contrib.sites",
    "allauth",
    "allauth.account",
    "allauth.socialaccount",
    "allauth.socialaccount.providers.google",
]

AUTHENTICATION_BACKENDS = [
    "django.contrib.auth.backends.ModelBackend",
    "allauth.account.auth_backends.AuthenticationBackend",
]

SITE_ID = 1
```

โค้ดข้างต้นเป็นเพียง**ภาพรวมคร่าว ๆ** เพื่อให้เห็นว่า django-allauth ต้องมีการตั้งค่า
เพิ่มเติมพอสมควร (แอปใหม่, backend ใหม่, credential ของ OAuth provider แต่ละตัว)
เทียบกับ built-in auth ที่พร้อมใช้ทันทีตั้งแต่ `pip install django`

### 309.4 เมื่อไหร่ควรเลือกอะไร

| สถานการณ์ | คำแนะนำ |
|---|---|
| ระบบภายในองค์กร ไม่ต้องการ social login | ใช้ built-in auth พอเพียง (สิ่งที่เรียนใน Part นี้) |
| ต้องการ "Sign in with Google/GitHub" | ใช้ django-allauth |
| Blog/Portfolio ส่วนตัวขนาดเล็ก | built-in auth เพียงพอมาก |
| SaaS ที่ต้องการลด friction การสมัครสมาชิก | django-allauth (ผู้ใช้ไม่ต้องจำรหัสผ่านใหม่) |
| ระบบที่ต้องรองรับ 2FA/MFA เต็มรูปแบบ | ผสมทั้งสองแนวทาง (เจาะลึกใน Part 037) |

> **หมายเหตุสำคัญ**: Part นี้จงใจแนะนำ django-allauth แบบภาพรวมสั้น ๆ เท่านั้น
> เพื่อให้คุณรู้จักตัวเลือกที่มีในโลกจริง ส่วนการติดตั้ง ตั้งค่า OAuth credential
> จริงกับ Google/GitHub, การเขียน custom adapter, และการรวม Social Login เข้ากับ
> ระบบ user เดิม จะเจาะลึกแบบเต็มรูปแบบใน **Part 036: Social Authentication
> (OAuth, django-allauth)**

---

## ขั้นตอนที่ 310: สรุปและแบบฝึกหัด — ระบบ login/logout/signup สมบูรณ์สำหรับ blog

### 310.1 ประกอบทุกอย่างเข้าด้วยกัน: โครงสร้างไฟล์เต็มรูปแบบ

มาประกอบความรู้ทั้ง 9 ขั้นตอนที่ผ่านมาให้เป็นระบบ authentication ที่สมบูรณ์และใช้
งานได้จริงสำหรับแอป `blog` โดยสร้างแอปใหม่ชื่อ `accounts` เพื่อแยกความรับผิดชอบ
เรื่อง auth ออกจาก `blog` (ทบทวนหลักการแยกแอปตามความรับผิดชอบจาก Part 005):

```bash
python manage.py startapp accounts
```

```python
# config/settings.py
INSTALLED_APPS = [
    "django.contrib.admin",
    "django.contrib.auth",
    "django.contrib.contenttypes",
    "django.contrib.sessions",
    "django.contrib.messages",
    "django.contrib.staticfiles",
    "accounts",
    "blog",
]

LOGIN_URL = "login"
LOGIN_REDIRECT_URL = "blog:post-list"
LOGOUT_REDIRECT_URL = "blog:post-list"
```

### 310.2 `accounts/forms.py`: ฟอร์มสมัครสมาชิกพร้อม email

```python
# accounts/forms.py
from django import forms
from django.contrib.auth.forms import UserCreationForm
from django.contrib.auth.models import User


class CustomUserCreationForm(UserCreationForm):
    email = forms.EmailField(required=True, label="อีเมล")

    class Meta(UserCreationForm.Meta):
        model = User
        fields = ("username", "email")

    def clean_email(self):
        email = self.cleaned_data["email"]
        if User.objects.filter(email__iexact=email).exists():
            raise forms.ValidationError("อีเมลนี้ถูกใช้สมัครสมาชิกไปแล้ว")
        return email

    def save(self, commit=True):
        user = super().save(commit=False)
        user.email = self.cleaned_data["email"]
        if commit:
            user.save()
        return user
```

### 310.3 `accounts/views.py`: signup พร้อม auto-login

```python
# accounts/views.py
from django.contrib.auth import login
from django.contrib.auth import views as auth_views
from django.contrib import messages
from django.urls import reverse_lazy
from django.views.generic import CreateView

from .forms import CustomUserCreationForm


class SignUpView(CreateView):
    form_class = CustomUserCreationForm
    template_name = "registration/signup.html"
    success_url = reverse_lazy("blog:post-list")

    def form_valid(self, form):
        response = super().form_valid(form)
        login(self.request, self.object)
        messages.success(self.request, "สมัครสมาชิกสำเร็จ! ยินดีต้อนรับสู่บล็อกของเรา")
        return response


class CustomLoginView(auth_views.LoginView):
    template_name = "registration/login.html"

    def form_valid(self, form):
        messages.success(self.request, f"ยินดีต้อนรับกลับมา, {form.get_user().username}!")
        return super().form_valid(form)
```

### 310.4 `accounts/urls.py` และการ include เข้า `config/urls.py`

```python
# accounts/urls.py
from django.urls import path

from . import views

urlpatterns = [
    path("signup/", views.SignUpView.as_view(), name="signup"),
    path("login/", views.CustomLoginView.as_view(), name="login"),
]
```

```python
# config/urls.py
from django.contrib import admin
from django.urls import include, path

urlpatterns = [
    path("admin/", admin.site.urls),
    path("accounts/", include("accounts.urls")),          # login/signup ของเราเอง (ต้องมาก่อน!)
    path("accounts/", include("django.contrib.auth.urls")),  # logout, password_reset ฯลฯ ของ Django
    path("", include("blog.urls")),
]
```

> **ทบทวนหลักการลำดับ URL จาก Part 023 ขั้นตอนที่ 221.7**: `path("accounts/",
> include("accounts.urls"))` ต้องมาก่อน `path("accounts/",
> include("django.contrib.auth.urls"))` เพราะทั้งคู่มี prefix `accounts/`
> เหมือนกัน และเรามี `login/` ของตัวเองที่ต้องการ override — ถ้าสลับลำดับ
> Django จะจับคู่ `login/` เข้ากับ `LoginView` มาตรฐานของ Django ก่อน ทำให้
> `CustomLoginView` ของเราไม่เคยถูกเรียกเลย

### 310.5 `blog/models.py`: field `author` (ทบทวนจาก Part 023-024)

```python
# blog/models.py
from django.conf import settings
from django.db import models
from django.urls import reverse


class Post(models.Model):
    title = models.CharField(max_length=200)
    slug = models.SlugField(max_length=220, unique=True)
    content = models.TextField()
    author = models.ForeignKey(
        settings.AUTH_USER_MODEL,
        on_delete=models.CASCADE,
        related_name="posts",
    )
    is_published = models.BooleanField(default=False)
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)

    class Meta:
        ordering = ["-created_at"]

    def __str__(self):
        return self.title

    def get_absolute_url(self):
        return reverse("blog:post-detail", kwargs={"slug": self.slug})
```

### 310.6 `blog/views.py`: บังคับ login ก่อนสร้าง/แก้ไข/ลบ

```python
# blog/views.py
from django.contrib.auth.mixins import LoginRequiredMixin, UserPassesTestMixin
from django.urls import reverse_lazy
from django.views.generic import (
    CreateView, DeleteView, DetailView, ListView, UpdateView,
)

from .forms import PostForm
from .models import Post


class PostListView(ListView):
    model = Post
    template_name = "blog/post_list.html"
    context_object_name = "posts"
    paginate_by = 10

    def get_queryset(self):
        return Post.objects.filter(is_published=True).select_related("author")


class PostDetailView(DetailView):
    model = Post
    template_name = "blog/post_detail.html"
    context_object_name = "post"


class PostCreateView(LoginRequiredMixin, CreateView):
    model = Post
    form_class = PostForm
    template_name = "blog/post_form.html"
    login_url = "login"

    def form_valid(self, form):
        form.instance.author = self.request.user
        return super().form_valid(form)


class PostUpdateView(LoginRequiredMixin, UserPassesTestMixin, UpdateView):
    model = Post
    form_class = PostForm
    template_name = "blog/post_form.html"
    login_url = "login"

    def test_func(self):
        post = self.get_object()
        return post.author_id == self.request.user.id


class PostDeleteView(LoginRequiredMixin, UserPassesTestMixin, DeleteView):
    model = Post
    template_name = "blog/post_confirm_delete.html"
    success_url = reverse_lazy("blog:post-list")
    login_url = "login"

    def test_func(self):
        post = self.get_object()
        return post.author_id == self.request.user.id
```

> **หมายเหตุ**: `UserPassesTestMixin` สำหรับตรวจสอบความเป็นเจ้าของถูกเกริ่นไปแล้ว
> ใน Part 024 ขั้นตอนที่ 234 การใช้ในที่นี้เป็นการนำมาประกอบกับสิ่งที่เรียนใน
> Part นี้เท่านั้น ส่วนรายละเอียดเชิงลึกของระบบ Permission/Group แบบเต็มรูปแบบ
> จะอยู่ใน Part 033

### 310.7 `blog/urls.py`

```python
# blog/urls.py
from django.urls import path

from . import views

app_name = "blog"

urlpatterns = [
    path("", views.PostListView.as_view(), name="post-list"),
    path("new/", views.PostCreateView.as_view(), name="post-create"),
    path("<slug:slug>/", views.PostDetailView.as_view(), name="post-detail"),
    path("<slug:slug>/edit/", views.PostUpdateView.as_view(), name="post-update"),
    path("<slug:slug>/delete/", views.PostDeleteView.as_view(), name="post-delete"),
]
```

### 310.8 Templates สุดท้ายที่ต้องมี

```html
<!-- templates/registration/login.html -->
{% extends "base.html" %}

{% block content %}
<h1>เข้าสู่ระบบ</h1>
<form method="post">
    {% csrf_token %}
    {{ form.as_p }}
    <button type="submit">เข้าสู่ระบบ</button>
</form>
<p>ยังไม่มีบัญชี? <a href="{% url 'signup' %}">สมัครสมาชิก</a></p>
{% endblock %}
```

```html
<!-- templates/registration/signup.html -->
{% extends "base.html" %}

{% block content %}
<h1>สมัครสมาชิก</h1>
<form method="post">
    {% csrf_token %}
    {{ form.as_p }}
    <button type="submit">สมัครสมาชิก</button>
</form>
<p>มีบัญชีอยู่แล้ว? <a href="{% url 'login' %}">เข้าสู่ระบบ</a></p>
{% endblock %}
```

```html
<!-- templates/base.html -->
<!DOCTYPE html>
<html lang="th">
<head>
    <meta charset="UTF-8">
    <title>{% block title %}My Blog{% endblock %}</title>
</head>
<body>
    <nav>
        <a href="{% url 'blog:post-list' %}">หน้าแรก</a>
        {% if user.is_authenticated %}
            <span>สวัสดี, {{ user.username }}</span>
            <a href="{% url 'blog:post-create' %}">เขียนบทความ</a>
            <form method="post" action="{% url 'logout' %}" style="display: inline;">
                {% csrf_token %}
                <button type="submit">ออกจากระบบ</button>
            </form>
        {% else %}
            <a href="{% url 'login' %}">เข้าสู่ระบบ</a>
            <a href="{% url 'signup' %}">สมัครสมาชิก</a>
        {% endif %}
    </nav>

    {% if messages %}
        {% for message in messages %}
            <p>{{ message }}</p>
        {% endfor %}
    {% endif %}

    {% block content %}{% endblock %}
</body>
</html>
```

### 310.9 ทดสอบระบบทั้งหมดแบบ end-to-end

```bash
python manage.py makemigrations blog
python manage.py migrate
python manage.py runserver
```

1. เปิด `http://127.0.0.1:8000/accounts/signup/` → สมัครสมาชิกใหม่ → ควรถูก
   login อัตโนมัติและ redirect ไปหน้ารายการบทความพร้อมข้อความ "สมัครสมาชิกสำเร็จ"
2. เปิด `http://127.0.0.1:8000/new/` ในสถานะที่ยัง**ไม่ได้** login (ลอง logout
   ก่อน) → ควรถูก redirect ไป `/accounts/login/?next=/new/`
3. Login สำเร็จ → ควรถูกพากลับไปที่ `/new/` อัตโนมัติ (ทดสอบกลไก `next`)
4. สร้างบทความ → ควรเห็น `author` ถูกกำหนดเป็นผู้ใช้ปัจจุบันอัตโนมัติ
5. Login ด้วยผู้ใช้อีกคน แล้วลองเข้า `/<slug ของโพสต์คนแรก>/edit/` → ควรได้ 403
   Forbidden (เพราะ `UserPassesTestMixin.test_func()` คืน `False`)

### 310.10 สรุปสิ่งที่ได้เรียนรู้ใน Part นี้

- เข้าใจภาพรวม `django.contrib.auth`, field ของ default `User` model, และตาราง
  auth ทั้งหมดที่ Django สร้างให้อัตโนมัติตอน `migrate`
- เจาะลึก `@login_required` และ `LoginRequiredMixin` ครบทุก attribute
  (`login_url`, `redirect_field_name`, `raise_exception`) ตามที่ Part 023/024
  ค้างไว้
- เข้าใจกลไก `authenticate()`/`login()`/`logout()` เบื้องหลัง session และเหตุผล
  ที่ควรใช้ built-in `LoginView`/`LogoutView` แทนการเขียนเอง
- สร้างหน้า signup จริงด้วย `UserCreationForm` พร้อม subclass เพิ่ม `email`
- เข้าใจ `request.user`, `AnonymousUser`, และกฎเหล็กเรื่องการเช็ค
  `is_authenticated` แทนการเช็ค `None`
- ตั้งค่า URL auth ทั้งระบบผ่าน `django.contrib.auth.urls` และรู้วิธี override
  เฉพาะบาง URL
- เข้าใจความเสี่ยง Open Redirect ใน parameter `next` และวิธีป้องกันด้วย
  `url_has_allowed_host_and_scheme()`
- เช็คสถานะ login ใน template ผ่าน context processor `auth`
- รู้จักภาพรวมของ django-allauth เทียบกับ built-in auth และรู้ว่าเมื่อไหร่ควรเลือก
  อะไร
- ประกอบทุกอย่างเป็นระบบ login/logout/signup ที่สมบูรณ์ พร้อมจำกัดสิทธิ์การสร้าง/
  แก้ไข/ลบบทความตามความเป็นเจ้าของ

### 310.11 Checklist ก่อนไป Part ถัดไป

- [ ] อธิบายได้ว่า `auth_user`, `auth_group`, `auth_permission` แต่ละตารางเก็บอะไร
- [ ] เขียน view ที่ป้องกันด้วย `@login_required` และ `LoginRequiredMixin` ได้ทั้งคู่
- [ ] อธิบายความแตกต่างระหว่าง `authenticate()` กับ `login()` ได้
- [ ] สร้างหน้า signup ด้วย `UserCreationForm` (หรือ subclass ที่เพิ่ม email) ได้จริง
- [ ] อธิบายได้ว่าทำไมต้องเช็ค `is_authenticated` แทนการเช็ค `if request.user:`
- [ ] ตั้งค่า `LOGIN_URL`, `LOGIN_REDIRECT_URL`, `LOGOUT_REDIRECT_URL` ได้ถูกต้อง
- [ ] อธิบายความเสี่ยง Open Redirect และวิธีป้องกันด้วยคำพูดของตัวเองได้
- [ ] เขียน `{% if user.is_authenticated %}` ใน template ได้โดยไม่ต้องเปิดเอกสาร
- [ ] ทำระบบ login/logout/signup + จำกัดสิทธิ์แก้ไข/ลบ Post ตามเจ้าของสำเร็จจริง

### 310.12 แบบฝึกหัดท้ายบท

**แบบฝึกหัดที่ 1**: เพิ่ม field `bio` (TextField, blank=True) ให้กับหน้า Profile
ของผู้ใช้ โดยสร้าง view ชื่อ `ProfileView` ที่แสดงข้อมูล `username`, `email`,
`date_joined`, และจำนวนบทความทั้งหมดที่ผู้ใช้คนนั้นเขียน (ใช้ `LoginRequiredMixin`
ป้องกัน และดึงเฉพาะข้อมูลของ `request.user` เท่านั้น — ห้ามให้ดูโปรไฟล์คนอื่นผ่าน
URL ได้ในขั้นตอนนี้)

**แบบฝึกหัดที่ 2**: ทดลองสร้างสถานการณ์ Open Redirect ด้วยตัวเอง — เขียน view
ทดสอบที่ **จงใจไม่ validate** `next` parameter (เหมือนตัวอย่างในขั้นตอนที่ 307.4)
แล้วลองยิง request ด้วย `curl` หรือ browser ไปที่
`?next=https://example.com/evil` เพื่อดูว่า redirect ไปที่นั่นจริงหรือไม่ จากนั้น
แก้ไขให้ใช้ `url_has_allowed_host_and_scheme()` แล้วทดสอบซ้ำว่าถูกบล็อกแล้ว
บันทึกผลเปรียบเทียบทั้งสองกรณี

**แบบฝึกหัดที่ 3**: เพิ่ม custom validation ให้กับ `CustomUserCreationForm` ใน
ขั้นตอนที่ 304.3 โดยบังคับว่า `username` ต้องมีความยาวอย่างน้อย 4 ตัวอักษร และ
ห้ามมีช่องว่าง (เขียน `clean_username()` เอง) พร้อมเขียนข้อความ error ภาษาไทยที่
เข้าใจง่าย

**แบบฝึกหัดที่ 4 (ขั้นสูง)**: อ่านเอกสารทางการของ django-allauth
(https://docs.allauth.org/) แล้วเปรียบเทียบ URL patterns ที่ allauth เตรียมไว้ให้
(เช่น `/accounts/google/login/`) กับ URL patterns จาก `django.contrib.auth.urls`
ที่เรียนใน ขั้นตอนที่ 306 เขียนตารางเปรียบเทียบว่า URL ไหนของ allauth มาแทนที่
URL ไหนของ built-in auth บ้าง เพื่อเตรียมความพร้อมก่อนเข้า Part 036

### 310.13 คำถามที่พบบ่อย (FAQ)

**Q: ทำไม `LoginRequiredMixin` ถึงไม่ตรวจสอบว่า user เป็น staff หรือ superuser?**
A: `LoginRequiredMixin` ตรวจสอบแค่ "login แล้วหรือยัง" (`is_authenticated`)
เท่านั้น ถ้าต้องการจำกัดเฉพาะ staff ต้องใช้ `UserPassesTestMixin` ร่วมกัน
(ตามที่เกริ่นใน Part 024 ขั้นตอนที่ 234) หรือ `PermissionRequiredMixin` ซึ่งจะ
เจาะลึกเต็มรูปแบบใน Part 033

**Q: `authenticate()` คืนค่า `None` แปลว่ารหัสผ่านผิดเสมอไปหรือไม่?**
A: ไม่เสมอไป `authenticate()` คืน `None` ได้จากหลายสาเหตุ: username ไม่มีในระบบ,
รหัสผ่านผิด, หรือ `is_active=False` (บัญชีถูกระงับ) — ด้วยเหตุผลด้านความปลอดภัย
Django (ผ่าน `ModelBackend`) ตั้งใจไม่บอกรายละเอียดว่า "ผิดเพราะอะไร" เพื่อไม่ให้
ผู้โจมตีรู้ว่า username นั้นมีอยู่จริงในระบบหรือไม่ (ป้องกัน **User Enumeration
Attack**)

**Q: ถ้าอยากให้ผู้ใช้ login ด้วย email แทน username ต้องทำอย่างไร?**
A: มี 2 ทางเลือก: (1) เขียน custom authentication backend ที่ override
`authenticate()` ให้ค้นหาด้วย `email` แทน `username`, หรือ (2) ใช้ django-allauth
ที่รองรับ config นี้ผ่าน settings ได้ทันที (`ACCOUNT_AUTHENTICATION_METHOD =
"email"`) — วิธีที่ 1 จะสาธิตแบบเต็มรูปแบบใน Part 032 เมื่อเรียนเรื่อง Custom
User Model เพราะมักทำควบคู่กับการทำ `email` เป็น unique field

**Q: จำเป็นต้องใช้ `UserPassesTestMixin` ทุกครั้งที่ต้องเช็คความเป็นเจ้าของหรือไม่
ทำไมไม่ filter queryset เหมือนที่เรียนใน Part 023 ขั้นตอนที่ 222.3 ไปเลย?**
A: ทั้งสองวิธีใช้ได้และให้ผลลัพธ์ปลอดภัยเหมือนกัน แต่ต่างกันที่ HTTP status code
ที่ผู้ใช้เห็น: การ filter queryset (`Post.objects.filter(author=request.user)`)
จะให้ **404 Not Found** เมื่อพยายามแก้ไขโพสต์คนอื่น (ปกปิดว่ามี object นี้อยู่จริง)
ส่วน `UserPassesTestMixin` จะให้ **403 Forbidden** (ยอมรับว่ามี object อยู่ แต่ไม่มี
สิทธิ์) — เลือกใช้ตามนโยบายความปลอดภัยของระบบคุณ เราจะเจาะลึกข้อดี-ข้อเสียของ
ทั้งสองแนวทางอย่างละเอียดใน Part 038 (Object-Level Permission)

---

## เตรียมตัวสำหรับ Part ถัดไป

**Part 032: Custom User Model** จะพาคุณไปแก้ปัญหาที่ Part นี้เกริ่นไว้หลายจุด
โดยเฉพาะเรื่อง `email` ที่ไม่ unique โดยค่าเริ่มต้น (ขั้นตอนที่ 301.2 และ 304.3)
คุณจะได้เรียนรู้ความแตกต่างระหว่าง `AbstractUser` กับ `AbstractBaseUser`, วิธีสร้าง
Custom User Model ของตัวเองตั้งแต่ต้นโปรเจกต์ (และเหตุผลที่ **ต้องทำตั้งแต่
migration แรกเท่านั้น** ไม่สามารถเปลี่ยนกลางทางได้อย่างปลอดภัย), การตั้งค่า
`AUTH_USER_MODEL`, และการเขียน `CustomUserManager` ของตัวเองสำหรับ `create_user()`
และ `create_superuser()`

เตรียมทบทวนเนื้อหาเรื่อง Model และ Migration จาก Part 011 ไว้ให้พร้อม เพราะ Part
ถัดไปจะเจาะลึกเรื่องนี้อีกครั้งในบริบทของระบบ Authentication โดยเฉพาะ
