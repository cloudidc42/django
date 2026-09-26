# Part 095: Multi-tenant Django Applications

> **ขั้นตอนที่ 941-950** | Phase 12: Scaling, Enterprise, โปรเจกต์จริง และเส้นทางอาชีพ

---

## ขั้นตอนที่ 941: Multi-tenancy คืออะไร

### 941.1 ทำความเข้าใจ Multi-tenancy

**Multi-tenant** = ระบบเดียวให้บริการหลาย "tenant" (organization/customer)
โดยข้อมูลของแต่ละ tenant แยกกันอย่างสมบูรณ์

```
SaaS Examples:
- Slack: บริษัท A และบริษัท B ใช้ app เดียวกัน ข้อมูลแยกกัน
- Shopify: ร้านค้า A และ B ใช้ platform เดียวกัน
- GitHub: org A และ B ใช้ platform เดียวกัน
```

### 941.2 3 รูปแบบหลัก

| รูปแบบ | คำอธิบาย | เหมาะกับ | ความซับซ้อน |
|---|---|---|---|
| **Shared Schema** | tenant_id column ในทุก table | small-medium SaaS | ต่ำ |
| **Separate Schema** | แต่ละ tenant มี PostgreSQL schema ของตัวเอง | medium SaaS | ปานกลาง |
| **Database per Tenant** | แต่ละ tenant มี database แยก | Enterprise, compliance | สูง |

---

## ขั้นตอนที่ 942: Shared Schema Approach

### 942.1 ออกแบบ Models

```python
# core/models.py

class Tenant(models.Model):
    """Tenant = Organization/Customer ที่สมัครใช้บริการ"""
    name = models.CharField(max_length=200)
    slug = models.SlugField(unique=True)    # tenant identifier ใน URL/subdomain
    domain = models.CharField(max_length=200, unique=True, blank=True)
    is_active = models.BooleanField(default=True)
    plan = models.CharField(max_length=50, choices=[
        ('free', 'Free'), ('starter', 'Starter'), ('pro', 'Pro'), ('enterprise', 'Enterprise')
    ])
    created_at = models.DateTimeField(auto_now_add=True)

    def __str__(self):
        return self.name


class TenantAwareModel(models.Model):
    """Abstract base model สำหรับทุก model ที่ต้องแยก tenant"""
    tenant = models.ForeignKey(
        Tenant,
        on_delete=models.CASCADE,
        related_name='%(class)s_set',
    )

    class Meta:
        abstract = True


# blog/models.py
from core.models import TenantAwareModel

class Post(TenantAwareModel):
    title = models.CharField(max_length=200)
    content = models.TextField()
    author = models.ForeignKey('auth.User', on_delete=models.CASCADE)
    created_at = models.DateTimeField(auto_now_add=True)

    class Meta:
        # Compound index เพื่อ query เร็ว
        indexes = [
            models.Index(fields=['tenant', 'created_at']),
        ]
```

### 942.2 TenantManager — กรอง tenant อัตโนมัติ

```python
# core/managers.py

from threading import local

_thread_local = local()


def get_current_tenant():
    """อ่าน tenant ปัจจุบันจาก thread-local storage"""
    return getattr(_thread_local, 'tenant', None)


def set_current_tenant(tenant):
    """ตั้ง tenant ปัจจุบันสำหรับ request นี้"""
    _thread_local.tenant = tenant


class TenantManager(models.Manager):
    """Manager ที่กรอง queryset ตาม current tenant อัตโนมัติ"""

    def get_queryset(self):
        queryset = super().get_queryset()
        tenant = get_current_tenant()
        if tenant is not None:
            return queryset.filter(tenant=tenant)
        return queryset


class TenantAwareModel(models.Model):
    tenant = models.ForeignKey('core.Tenant', on_delete=models.CASCADE)

    # Manager ที่กรอง tenant อัตโนมัติ
    objects = TenantManager()
    # Manager ที่ไม่กรอง (สำหรับ admin/migration)
    all_objects = models.Manager()

    class Meta:
        abstract = True
```

### 942.3 TenantMiddleware

```python
# core/middleware.py

from .models import Tenant
from .managers import set_current_tenant


class TenantMiddleware:
    """กำหนด current tenant จาก subdomain หรือ request header"""

    def __init__(self, get_response):
        self.get_response = get_response

    def __call__(self, request):
        tenant = self._resolve_tenant(request)
        set_current_tenant(tenant)
        request.tenant = tenant

        response = self.get_response(request)

        # Reset หลัง request เสร็จ (สำคัญ!)
        set_current_tenant(None)
        return response

    def _resolve_tenant(self, request):
        """ระบุ tenant จาก subdomain: acme.myblog.com → tenant slug = acme"""
        host = request.get_host().lower()
        # ตัด port ออก
        host = host.split(':')[0]

        # Subdomain-based routing
        parts = host.split('.')
        if len(parts) >= 3:
            subdomain = parts[0]
            try:
                return Tenant.objects.get(slug=subdomain, is_active=True)
            except Tenant.DoesNotExist:
                pass

        # Custom domain routing
        try:
            return Tenant.objects.get(domain=host, is_active=True)
        except Tenant.DoesNotExist:
            return None
```

```python
# config/settings/base.py
MIDDLEWARE = [
    'core.middleware.TenantMiddleware',
    # ... rest of middleware
]
```

### 942.4 Views ที่ใช้ Tenant Context

```python
# blog/views.py

from rest_framework import generics
from .models import Post
from .serializers import PostSerializer


class PostListCreateView(generics.ListCreateAPIView):
    serializer_class = PostSerializer

    def get_queryset(self):
        # TenantManager กรอง tenant อัตโนมัติ!
        return Post.objects.all()

    def perform_create(self, serializer):
        # ต้องระบุ tenant เพราะ auto-create ต้องการ
        serializer.save(
            tenant=self.request.tenant,
            author=self.request.user,
        )
```

---

## ขั้นตอนที่ 943: Separate Schema Approach ด้วย django-tenants

### 943.1 ติดตั้ง django-tenants

```bash
pip install django-tenants
```

### 943.2 ตั้งค่า settings.py

```python
# config/settings.py

DATABASE_ROUTERS = (
    'django_tenants.routers.TenantSyncRouter',
)

DATABASES = {
    'default': {
        'ENGINE': 'django_tenants.postgresql_backend',
        # ... other settings
    }
}

TENANT_MODEL = 'tenants.Client'
TENANT_DOMAIN_MODEL = 'tenants.Domain'

# Apps ที่ share ข้ามทุก tenant (public schema)
SHARED_APPS = [
    'django_tenants',
    'django.contrib.contenttypes',
    'django.contrib.auth',
    'tenants',      # Tenant management
]

# Apps ที่แยกสำหรับแต่ละ tenant
TENANT_APPS = [
    'blog',
    'orders',
    'accounts',
]

INSTALLED_APPS = list(SHARED_APPS) + [app for app in TENANT_APPS if app not in SHARED_APPS]
```

### 943.3 Models สำหรับ django-tenants

```python
# tenants/models.py
from django_tenants.models import TenantMixin, DomainMixin


class Client(TenantMixin):
    name = models.CharField(max_length=200)
    created_at = models.DateTimeField(auto_now_add=True)
    plan = models.CharField(max_length=50)

    # auto_create_schema=True จะสร้าง PostgreSQL schema ใหม่อัตโนมัติ
    auto_create_schema = True


class Domain(DomainMixin):
    pass
```

### 943.4 Migrations สำหรับ Multi-schema

```bash
# สร้าง migration สำหรับ shared apps
python manage.py makemigrations

# migrate shared tables ไป public schema
python manage.py migrate_schemas --shared

# สร้าง tenant ใหม่ → สร้าง schema อัตโนมัติ
python manage.py shell
>>> from tenants.models import Client, Domain
>>> tenant = Client(schema_name='acme', name='ACME Corp', plan='pro')
>>> tenant.save()  # สร้าง schema 'acme' ใน PostgreSQL
>>> domain = Domain(domain='acme.myblog.com', tenant=tenant, is_primary=True)
>>> domain.save()

# migrate แต่ละ tenant
python manage.py migrate_schemas  # migrate ทุก tenant schema
python manage.py migrate_schemas --schema=acme  # migrate เฉพาะ tenant acme
```

---

## ขั้นตอนที่ 944: Tenant-aware Authentication

### 944.1 User ผูกกับ Tenant

```python
# accounts/models.py

class TenantUser(models.Model):
    """เชื่อม User กับ Tenant (many-to-many) พร้อม role"""
    user = models.ForeignKey(
        settings.AUTH_USER_MODEL,
        on_delete=models.CASCADE,
        related_name='tenant_memberships',
    )
    tenant = models.ForeignKey(
        'core.Tenant',
        on_delete=models.CASCADE,
        related_name='members',
    )
    role = models.CharField(max_length=50, choices=[
        ('owner', 'Owner'),
        ('admin', 'Admin'),
        ('member', 'Member'),
        ('viewer', 'Viewer'),
    ])
    is_active = models.BooleanField(default=True)
    joined_at = models.DateTimeField(auto_now_add=True)

    class Meta:
        unique_together = ('user', 'tenant')
        indexes = [models.Index(fields=['user', 'tenant'])]
```

### 944.2 Permission Check ตาม Tenant Role

```python
# core/permissions.py
from rest_framework import permissions


class TenantMemberPermission(permissions.BasePermission):
    """ตรวจสอบว่า user เป็น member ของ tenant ปัจจุบัน"""

    def has_permission(self, request, view):
        if not request.user.is_authenticated:
            return False

        if not request.tenant:
            return False

        return request.user.tenant_memberships.filter(
            tenant=request.tenant,
            is_active=True,
        ).exists()


class TenantAdminPermission(TenantMemberPermission):
    """ต้องเป็น admin หรือ owner"""

    def has_permission(self, request, view):
        if not super().has_permission(request, view):
            return False

        return request.user.tenant_memberships.filter(
            tenant=request.tenant,
            role__in=['owner', 'admin'],
            is_active=True,
        ).exists()
```

---

## ขั้นตอนที่ 945: Tenant Onboarding Flow

### 945.1 Registration Endpoint

```python
# tenants/views.py

from rest_framework.decorators import api_view, permission_classes
from rest_framework.permissions import AllowAny
from rest_framework.response import Response
from django.db import transaction


@api_view(['POST'])
@permission_classes([AllowAny])
def register_tenant(request):
    """สมัครใช้งาน — สร้าง tenant + owner user ในครั้งเดียว"""
    serializer = TenantRegistrationSerializer(data=request.data)
    serializer.is_valid(raise_exception=True)

    with transaction.atomic():
        # 1. สร้าง Tenant
        tenant = Tenant.objects.create(
            name=serializer.validated_data['company_name'],
            slug=serializer.validated_data['subdomain'],
            plan='free',
        )

        # 2. สร้าง Owner User
        user = User.objects.create_user(
            username=serializer.validated_data['email'],
            email=serializer.validated_data['email'],
            password=serializer.validated_data['password'],
        )

        # 3. เชื่อม User กับ Tenant ในฐานะ owner
        TenantUser.objects.create(
            user=user,
            tenant=tenant,
            role='owner',
        )

        # 4. ส่ง welcome email
        send_welcome_email.delay(user.id, tenant.id)

    return Response({
        'message': 'ลงทะเบียนสำเร็จ',
        'tenant_url': f'https://{tenant.slug}.myblog.com',
    }, status=201)
```

---

## ขั้นตอนที่ 946: Billing และ Plan Limits

### 946.1 Plan-based Feature Gating

```python
# core/plans.py

PLAN_LIMITS = {
    'free': {
        'max_users': 5,
        'max_posts': 100,
        'max_storage_mb': 100,
        'api_rate_limit': 100,
        'features': ['basic_analytics'],
    },
    'starter': {
        'max_users': 25,
        'max_posts': 1000,
        'max_storage_mb': 1000,
        'api_rate_limit': 1000,
        'features': ['basic_analytics', 'custom_domain', 'email_support'],
    },
    'pro': {
        'max_users': 100,
        'max_posts': 10000,
        'max_storage_mb': 10000,
        'api_rate_limit': 10000,
        'features': ['basic_analytics', 'advanced_analytics', 'custom_domain',
                     'priority_support', 'api_access'],
    },
}


def check_plan_limit(tenant, resource: str) -> bool:
    """ตรวจสอบว่า tenant ยังไม่เกิน limit ของ plan"""
    limits = PLAN_LIMITS.get(tenant.plan, {})
    limit_key = f'max_{resource}'

    if limit_key not in limits:
        return True  # ไม่มี limit สำหรับ resource นี้

    current_count = get_current_usage(tenant, resource)
    return current_count < limits[limit_key]


# views.py
def create_post(request):
    if not check_plan_limit(request.tenant, 'posts'):
        return Response(
            {'error': 'คุณเข้าถึง limit การสร้าง Post ของ plan ปัจจุบันแล้ว กรุณา upgrade'},
            status=403
        )
    # ... create post
```

---

## ขั้นตอนที่ 947: Multi-tenant Admin

### 947.1 Django Admin แบบ Multi-tenant

```python
# core/admin.py
from django.contrib import admin


class TenantAwareModelAdmin(admin.ModelAdmin):
    """Base ModelAdmin ที่แสดงเฉพาะ records ของ selected tenant"""

    def get_queryset(self, request):
        qs = super().get_queryset(request)
        if hasattr(request, 'tenant') and request.tenant:
            return qs.filter(tenant=request.tenant)
        return qs

    def save_model(self, request, obj, form, change):
        if not obj.pk and hasattr(request, 'tenant'):
            obj.tenant = request.tenant
        super().save_model(request, obj, form, change)


@admin.register(Tenant)
class TenantAdmin(admin.ModelAdmin):
    list_display = ['name', 'slug', 'plan', 'is_active', 'created_at']
    list_filter = ['plan', 'is_active']
    search_fields = ['name', 'slug']
```

---

## ขั้นตอนที่ 948: Performance สำหรับ Multi-tenant

### 948.1 Database Indexes ที่สำคัญ

```python
# blog/models.py

class Post(TenantAwareModel):
    title = models.CharField(max_length=200)
    content = models.TextField()
    author = models.ForeignKey('auth.User', on_delete=models.CASCADE)
    status = models.CharField(max_length=20, default='draft')
    created_at = models.DateTimeField(auto_now_add=True)

    class Meta:
        indexes = [
            # Compound index: tenant + status + created_at (สำหรับ query บ่อยที่สุด)
            models.Index(fields=['tenant', 'status', '-created_at']),
            # Tenant + author
            models.Index(fields=['tenant', 'author']),
        ]
```

### 948.2 Row-Level Security ใน PostgreSQL

```sql
-- เปิด Row Level Security สำหรับ table posts
ALTER TABLE blog_post ENABLE ROW LEVEL SECURITY;

-- Policy: แต่ละ connection เห็นแค่ records ของ tenant ตัวเอง
CREATE POLICY tenant_isolation ON blog_post
    USING (tenant_id = current_setting('app.tenant_id')::INTEGER);

-- ตั้ง tenant_id ก่อน query (ใน Django middleware)
-- SET LOCAL app.tenant_id = 123;
```

```python
# core/middleware.py — ตั้ง PostgreSQL session variable
from django.db import connection

class TenantMiddleware:
    def __call__(self, request):
        tenant = self._resolve_tenant(request)
        if tenant:
            with connection.cursor() as cursor:
                cursor.execute("SET LOCAL app.tenant_id = %s", [tenant.id])
        # ...
```

---

## ขั้นตอนที่ 949: Testing Multi-tenant Apps

### 949.1 Test Fixtures สำหรับ Multi-tenant

```python
# tests/conftest.py
import pytest
from core.models import Tenant
from accounts.models import TenantUser
from core.managers import set_current_tenant


@pytest.fixture
def tenant_a(db):
    return Tenant.objects.create(name='Tenant A', slug='tenant-a', plan='pro')


@pytest.fixture
def tenant_b(db):
    return Tenant.objects.create(name='Tenant B', slug='tenant-b', plan='free')


@pytest.fixture
def tenant_a_context(tenant_a):
    """Context manager ที่ตั้ง current tenant เป็น tenant A"""
    set_current_tenant(tenant_a)
    yield tenant_a
    set_current_tenant(None)


# tests/test_tenant_isolation.py
def test_tenant_data_isolation(tenant_a_context, tenant_b, user_factory):
    """ทดสอบว่า tenant A ไม่เห็นข้อมูลของ tenant B"""
    # สร้าง post ใน tenant A (context ปัจจุบัน)
    post_a = Post.objects.create(
        tenant=tenant_a_context,
        title='Post ของ Tenant A',
        author=user_factory(tenant=tenant_a_context),
    )

    # สร้าง post ใน tenant B โดยตรง (bypass manager)
    post_b = Post.all_objects.create(
        tenant=tenant_b,
        title='Post ของ Tenant B',
        author=user_factory(tenant=tenant_b),
    )

    # Query ผ่าน TenantManager (ควรเห็นแค่ของ tenant A)
    posts = Post.objects.all()
    assert posts.count() == 1
    assert posts.first().title == 'Post ของ Tenant A'

    # ตรวจสอบว่า post B ยังอยู่ (ผ่าน all_objects)
    assert Post.all_objects.filter(tenant=tenant_b).count() == 1
```

---

## ขั้นตอนที่ 950: สรุปและแบบฝึกหัด

### 950.1 เปรียบเทียบ Approaches

| | Shared Schema | Separate Schema | DB per Tenant |
|---|---|---|---|
| Complexity | ต่ำ | ปานกลาง | สูง |
| Data isolation | Row-level (query filter) | Schema-level | Full |
| Performance | ดีกับ indexes ถูกต้อง | ดี | ดีที่สุด |
| Max tenants | ไม่จำกัด | ~10,000+ | ~1,000 |
| Backup complexity | ยาก (ต้อง filter) | ปานกลาง | ง่าย (dump by DB) |
| เหมาะกับ | Small/Medium SaaS | Medium SaaS | Enterprise |

### 950.2 แบบฝึกหัดท้ายบท

**แบบฝึกหัดที่ 1**: แปลง blog project เป็น multi-tenant โดยใช้ shared schema approach
เพิ่ม `TenantMiddleware`, `TenantAwareModel`, และ `TenantManager` แล้วทดสอบว่า
tenant isolation ทำงานถูกต้องด้วย test จาก 949.1

**แบบฝึกหัดที่ 2**: สร้าง tenant registration API endpoint ตามขั้นตอนที่ 945.1 และ
ทดสอบ flow ครบตั้งแต่ register จนถึง login ด้วย tenant context

**แบบฝึกหัดที่ 3**: ใช้ PLAN_LIMITS จากขั้นตอนที่ 946.1 เพิ่ม `check_plan_limit`
ใน Post create endpoint และทดสอบว่า free plan ไม่สามารถสร้าง post เกิน 100 ได้

**แบบฝึกหัดที่ 4 (ขั้นสูง)**: ติดตั้ง `django-tenants` และสร้าง project ใหม่แบบ
separate schema ตั้งค่า subdomain routing แล้วทดสอบว่า `acme.localhost` และ
`beta.localhost` มีข้อมูลแยกกัน schema-level

### 950.3 เตรียมตัวสำหรับ Part ถัดไป

**Part 096: GraphQL กับ Django (Graphene)** จะพาไปเรียนรู้ alternative ของ REST API
ที่ให้ client ขอข้อมูลที่ต้องการได้ยืดหยุ่นกว่า — เรียนรู้ Graphene-Django ตั้งแต่
Schema, Types, Queries, Mutations, Subscriptions จนถึง N+1 problem solving และ
DataLoader pattern
