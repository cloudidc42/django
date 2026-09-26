# Part 099: Django Design Patterns และ Clean Architecture

> **ขั้นตอนที่ 981-990** | Phase 12: Scaling, Enterprise, โปรเจกต์จริง และเส้นทางอาชีพ

---

## ขั้นตอนที่ 981: Repository Pattern

### 981.1 ปัญหาที่แก้

**Fat View / Fat Model**: logic กระจายอยู่ใน views/models ทดสอบยาก
**Repository Pattern**: แยก data access logic ออกเป็นชั้นต่างหาก

```python
# apps/blog/repositories.py
from abc import ABC, abstractmethod
from typing import Optional
from .models import Post


class PostRepositoryBase(ABC):
    @abstractmethod
    def get_by_id(self, post_id: int) -> Optional[Post]:
        ...

    @abstractmethod
    def get_published(self) -> list[Post]:
        ...

    @abstractmethod
    def create(self, **kwargs) -> Post:
        ...


class DjangoPostRepository(PostRepositoryBase):
    """Implementation จริงที่ใช้ Django ORM"""

    def get_by_id(self, post_id: int) -> Optional[Post]:
        try:
            return Post.objects.select_related('author').get(id=post_id)
        except Post.DoesNotExist:
            return None

    def get_published(self) -> list[Post]:
        return list(
            Post.objects.filter(status='published')
            .select_related('author')
            .prefetch_related('tags')
            .order_by('-created_at')
        )

    def get_by_author(self, author_id: int) -> list[Post]:
        return list(Post.objects.filter(author_id=author_id, status='published'))

    def create(self, **kwargs) -> Post:
        return Post.objects.create(**kwargs)

    def update(self, post: Post, **kwargs) -> Post:
        for key, value in kwargs.items():
            setattr(post, key, value)
        post.save(update_fields=list(kwargs.keys()))
        return post


class InMemoryPostRepository(PostRepositoryBase):
    """Implementation สำหรับ testing — ไม่แตะ database"""

    def __init__(self):
        self._posts = {}
        self._counter = 1

    def get_by_id(self, post_id: int) -> Optional[Post]:
        return self._posts.get(post_id)

    def get_published(self) -> list[Post]:
        return [p for p in self._posts.values() if p.status == 'published']

    def create(self, **kwargs) -> Post:
        post = Post(id=self._counter, **kwargs)
        self._posts[self._counter] = post
        self._counter += 1
        return post
```

```python
# tests/test_post_service.py
def test_publish_post_without_db():
    """ทดสอบ service logic โดยไม่แตะ database"""
    repo = InMemoryPostRepository()
    service = PostService(repo)

    post = repo.create(title='ทดสอบ', content='...', author_id=1, status='draft')
    service.publish(post.id, publisher_id=1)

    updated = repo.get_by_id(post.id)
    assert updated.status == 'published'
```

---

## ขั้นตอนที่ 982: Service Layer Pattern

### 982.1 Business Logic ใน Service

```python
# apps/blog/services.py
from typing import Optional
from django.db import transaction
from django.core.exceptions import PermissionDenied
from .repositories import PostRepositoryBase
from .models import Post


class PostService:
    """Business logic ทั้งหมดอยู่ที่นี่ ไม่อยู่ใน views หรือ models"""

    def __init__(self, post_repository: PostRepositoryBase):
        self.repo = post_repository

    def publish(self, post_id: int, publisher_id: int) -> Post:
        """Publish post — ตรวจสอบ ownership และเงื่อนไขก่อน"""
        post = self.repo.get_by_id(post_id)
        if not post:
            raise ValueError(f"ไม่พบ Post id={post_id}")
        if post.author_id != publisher_id:
            raise PermissionDenied("ไม่มีสิทธิ์ publish post นี้")
        if post.status == 'published':
            raise ValueError("Post นี้ publish แล้ว")
        if not post.content.strip():
            raise ValueError("ไม่สามารถ publish post ที่ไม่มีเนื้อหา")

        return self.repo.update(post, status='published')

    @transaction.atomic
    def create_with_tags(self, author_id: int, title: str, content: str, tag_ids: list[int]) -> Post:
        """สร้าง post พร้อม tags ใน transaction"""
        post = self.repo.create(
            title=title, content=content, author_id=author_id, status='draft'
        )
        if tag_ids:
            post.tags.set(tag_ids)
        return post

    def get_feed_for_user(self, user_id: int) -> list[Post]:
        """Logic สำหรับ personalized feed"""
        return self.repo.get_published()
```

```python
# dependency injection ใน views
# apps/blog/dependencies.py
from functools import lru_cache
from .repositories import DjangoPostRepository
from .services import PostService


@lru_cache(maxsize=None)
def get_post_service() -> PostService:
    return PostService(DjangoPostRepository())


# apps/blog/views.py
from django.views import View
from .dependencies import get_post_service


class PostPublishView(View):
    def post(self, request, post_id):
        service = get_post_service()
        try:
            post = service.publish(post_id, request.user.id)
        except ValueError as e:
            return JsonResponse({'error': str(e)}, status=400)
        except PermissionDenied:
            return JsonResponse({'error': 'ไม่มีสิทธิ์'}, status=403)
        return JsonResponse({'id': post.id, 'status': post.status})
```

---

## ขั้นตอนที่ 983: Command Pattern

### 983.1 Command Objects

```python
# apps/orders/commands.py
from dataclasses import dataclass
from decimal import Decimal
from django.db import transaction


@dataclass
class PlaceOrderCommand:
    user_id: int
    cart_items: list[dict]
    shipping_address: dict
    coupon_code: str | None = None


@dataclass
class PlaceOrderResult:
    order_id: int
    order_number: str
    total: Decimal


class PlaceOrderHandler:
    """Command Handler — จัดการ PlaceOrderCommand ทั้งหมด"""

    @transaction.atomic
    def handle(self, command: PlaceOrderCommand) -> PlaceOrderResult:
        # 1. validate stock
        self._validate_stock(command.cart_items)

        # 2. คำนวณราคา
        subtotal = self._calculate_subtotal(command.cart_items)
        discount = self._apply_coupon(command.coupon_code, subtotal)
        shipping = self._calculate_shipping(subtotal - discount)
        total = subtotal - discount + shipping

        # 3. สร้าง order
        order = self._create_order(command, subtotal, discount, shipping, total)

        # 4. ลดสต็อก
        self._deduct_stock(command.cart_items)

        return PlaceOrderResult(
            order_id=order.id,
            order_number=order.order_number,
            total=order.total,
        )

    def _validate_stock(self, cart_items):
        from apps.catalog.models import Product
        for item in cart_items:
            product = Product.objects.select_for_update().get(id=item['product_id'])
            if product.stock < item['quantity']:
                raise ValueError(f"{product.name}: สต็อกไม่เพียงพอ (เหลือ {product.stock})")

    def _calculate_subtotal(self, cart_items):
        from apps.catalog.models import Product
        total = Decimal('0')
        for item in cart_items:
            product = Product.objects.get(id=item['product_id'])
            total += product.effective_price * item['quantity']
        return total

    def _apply_coupon(self, coupon_code, subtotal):
        if not coupon_code:
            return Decimal('0')
        from apps.orders.models import Coupon
        try:
            coupon = Coupon.objects.get(code=coupon_code, is_active=True)
            return coupon.calculate_discount(subtotal)
        except Coupon.DoesNotExist:
            raise ValueError("คูปองไม่ถูกต้องหรือหมดอายุ")
```

---

## ขั้นตอนที่ 984: Observer Pattern ด้วย Signals

### 984.1 Domain Events ผ่าน Signals

```python
# apps/orders/signals.py
from django.dispatch import Signal

# Custom domain events
order_placed = Signal()
order_confirmed = Signal()
order_shipped = Signal()
order_cancelled = Signal()
```

```python
# apps/orders/handlers.py
from django.dispatch import receiver
from .signals import order_placed, order_confirmed, order_shipped


@receiver(order_placed)
def handle_order_placed(sender, order, **kwargs):
    """เมื่อมีคำสั่งซื้อใหม่"""
    from .tasks import send_order_confirmation_email
    send_order_confirmation_email.delay(order.id)


@receiver(order_confirmed)
def handle_order_confirmed(sender, order, **kwargs):
    """เมื่อ order ได้รับการยืนยัน"""
    from apps.inventory.tasks import update_inventory
    update_inventory.delay(order.id)


@receiver(order_shipped)
def handle_order_shipped(sender, order, tracking_number, **kwargs):
    """เมื่อ order จัดส่งแล้ว"""
    from .tasks import send_shipping_notification
    send_shipping_notification.delay(order.id, tracking_number)
```

```python
# เรียกใช้ใน views
from .signals import order_placed

def confirm_order(order):
    order.status = 'confirmed'
    order.save()
    order_confirmed.send(sender=order.__class__, order=order)
```

---

## ขั้นตอนที่ 985: Strategy Pattern

### 985.1 Multiple Payment Strategies

```python
# apps/payments/strategies.py
from abc import ABC, abstractmethod
from decimal import Decimal
from dataclasses import dataclass


@dataclass
class PaymentResult:
    success: bool
    transaction_id: str
    message: str


class PaymentStrategy(ABC):
    @abstractmethod
    def charge(self, amount: Decimal, order_number: str, **kwargs) -> PaymentResult:
        ...


class StripeStrategy(PaymentStrategy):
    def charge(self, amount: Decimal, order_number: str, **kwargs) -> PaymentResult:
        import stripe
        try:
            intent = stripe.PaymentIntent.create(
                amount=int(amount * 100),
                currency='thb',
                confirm=True,
                payment_method=kwargs['payment_method_id'],
                metadata={'order_number': order_number},
            )
            return PaymentResult(True, intent.id, 'ชำระเงินสำเร็จ')
        except stripe.error.CardError as e:
            return PaymentResult(False, '', str(e.user_message))


class OmiseStrategy(PaymentStrategy):
    def charge(self, amount: Decimal, order_number: str, **kwargs) -> PaymentResult:
        import omise
        try:
            charge = omise.Charge.create(
                amount=int(amount * 100),
                currency='thb',
                card=kwargs['card_token'],
                description=f'Order {order_number}',
            )
            return PaymentResult(charge.status == 'successful', charge.id, charge.failure_message or 'สำเร็จ')
        except omise.errors.BaseError as e:
            return PaymentResult(False, '', str(e))


class PaymentProcessor:
    """Context ที่ใช้ Strategy"""
    def __init__(self, strategy: PaymentStrategy):
        self.strategy = strategy

    def process(self, amount: Decimal, order_number: str, **kwargs) -> PaymentResult:
        return self.strategy.charge(amount, order_number, **kwargs)


# ใช้งาน
processor = PaymentProcessor(StripeStrategy())
result = processor.process(Decimal('1500.00'), 'ORD1234567890', payment_method_id='pm_xxx')
```

---

## ขั้นตอนที่ 986: Clean Architecture ใน Django

### 986.1 โครงสร้างแบบ Clean Architecture

```
apps/
└── blog/
    ├── domain/           # Pure Python — ไม่ import Django
    │   ├── models.py     # Dataclasses, ไม่ใช่ Django models
    │   └── services.py   # Business logic บริสุทธิ์
    ├── infrastructure/   # Django-specific
    │   ├── models.py     # Django ORM models
    │   ├── repositories.py
    │   └── admin.py
    ├── interfaces/       # API/View layer
    │   ├── views.py
    │   ├── serializers.py
    │   └── urls.py
    └── application/      # Use cases
        └── use_cases.py
```

```python
# apps/blog/domain/services.py — Pure Python, ไม่ import Django
from dataclasses import dataclass
from datetime import datetime


@dataclass
class PostData:
    id: int | None
    title: str
    content: str
    author_id: int
    status: str = 'draft'
    created_at: datetime | None = None


class PostDomainService:
    """Business rules บริสุทธิ์ — testable โดยไม่ต้องมี Django"""

    @staticmethod
    def can_publish(post: PostData, requester_id: int) -> tuple[bool, str]:
        if post.author_id != requester_id:
            return False, "ไม่มีสิทธิ์"
        if post.status == 'published':
            return False, "Published แล้ว"
        if len(post.content.strip()) < 10:
            return False, "เนื้อหาสั้นเกินไป"
        return True, ""

    @staticmethod
    def calculate_reading_time(content: str) -> int:
        words = len(content.split())
        return max(1, round(words / 250))
```

---

## ขั้นตอนที่ 987: SOLID Principles ใน Django

### 987.1 Single Responsibility

```python
# ❌ Fat View ทำหลายอย่าง
class CreatePostView(View):
    def post(self, request):
        data = json.loads(request.body)
        # validate
        if not data.get('title'):
            return JsonResponse({'error': 'title required'}, status=400)
        # create
        post = Post.objects.create(**data, author=request.user)
        # send email
        send_mail('New post', f'{post.title} created', ...)
        # log
        logger.info(f'Post {post.id} created')
        return JsonResponse({'id': post.id})

# ✅ แต่ละ class ทำหน้าที่เดียว
class CreatePostView(View):
    def post(self, request):
        service = get_post_service()
        form = PostCreateForm(json.loads(request.body))
        if not form.is_valid():
            return JsonResponse(form.errors, status=400)
        post = service.create(author=request.user, **form.cleaned_data)
        return JsonResponse({'id': post.id})
```

### 987.2 Open/Closed Principle

```python
# ✅ เปิดรับ extension, ปิดรับ modification
class NotificationSender(ABC):
    @abstractmethod
    def send(self, user, message: str): ...


class EmailNotificationSender(NotificationSender):
    def send(self, user, message: str):
        send_mail(message, message, settings.DEFAULT_FROM_EMAIL, [user.email])


class SMSNotificationSender(NotificationSender):
    def send(self, user, message: str):
        # ส่ง SMS
        ...


class PushNotificationSender(NotificationSender):
    def send(self, user, message: str):
        # ส่ง Push notification
        ...


class MultiChannelNotifier:
    """เพิ่ม channel ใหม่โดยไม่แก้ class นี้"""
    def __init__(self, senders: list[NotificationSender]):
        self.senders = senders

    def notify(self, user, message: str):
        for sender in self.senders:
            sender.send(user, message)
```

---

## ขั้นตอนที่ 988: Testing Patterns

### 988.1 Test Pyramid

```python
# Unit Test (เร็วที่สุด, มากที่สุด)
def test_post_domain_publish_rule():
    service = PostDomainService()
    post = PostData(id=1, title='test', content='content here...', author_id=1)
    can, reason = service.can_publish(post, requester_id=1)
    assert can is True


# Integration Test (กลาง)
@pytest.mark.django_db
def test_post_service_publish(post_factory, user_factory):
    user = user_factory()
    post = post_factory(author=user, status='draft')
    service = PostService(DjangoPostRepository())
    result = service.publish(post.id, publisher_id=user.id)
    assert result.status == 'published'


# E2E Test (ช้าที่สุด, น้อยที่สุด)
def test_create_and_publish_post_via_api(client, authenticated_user):
    response = client.post('/api/posts/', {'title': 'test', 'content': 'content...'})
    assert response.status_code == 201
    post_id = response.json()['id']

    response = client.post(f'/api/posts/{post_id}/publish/')
    assert response.status_code == 200
    assert response.json()['status'] == 'published'
```

---

## ขั้นตอนที่ 989: Code Organization Best Practices

### 989.1 Fat Model, Thin View

```python
# apps/orders/models.py
class Order(models.Model):
    # ... fields ...

    def can_be_cancelled(self) -> bool:
        return self.status in [self.Status.PENDING, self.Status.CONFIRMED]

    def cancel(self) -> None:
        if not self.can_be_cancelled():
            raise ValueError(f"ไม่สามารถยกเลิก order ที่มีสถานะ {self.status}")
        self.status = self.Status.CANCELLED
        self.save(update_fields=['status', 'updated_at'])

    def calculate_refund_amount(self) -> Decimal:
        """Logic การคืนเงิน (ถ้าผ่านไปแล้ว 24 ชั่วโมง หักค่าธรรมเนียม 5%)"""
        from django.utils import timezone
        from datetime import timedelta
        if timezone.now() - self.created_at > timedelta(hours=24):
            return self.total * Decimal('0.95')
        return self.total
```

### 989.2 Separate settings ตาม environment

```python
# config/settings/base.py — settings ร่วม
# config/settings/development.py — debug, email backend
# config/settings/production.py — security, sentry, CDN
# config/settings/testing.py — fast hasher, in-memory cache

# pytest.ini
[pytest]
DJANGO_SETTINGS_MODULE = config.settings.testing
```

---

## ขั้นตอนที่ 990: สรุปและ Checklist

### 990.1 Design Patterns Summary

| Pattern | ใช้เมื่อ | ตัวอย่างใน Django |
|---------|---------|-------------------|
| Repository | แยก data access จาก business logic | PostRepository, UserRepository |
| Service Layer | รวม business logic ที่ข้าม models | OrderService, PaymentService |
| Command | แยก "สิ่งที่จะทำ" จาก "วิธีทำ" | PlaceOrderCommand, SendEmailCommand |
| Observer | react ต่อ events โดยไม่ coupling | Django Signals |
| Strategy | สลับ algorithm ได้ runtime | Payment gateway, Notification channel |
| Factory | สร้าง objects ที่ซับซ้อน | ModelFactory (factory_boy) |

### 990.2 Checklist Code Quality

- [ ] Views얇: validation และ HTTP handling เท่านั้น
- [ ] Models: business rules และ data integrity เท่านั้น
- [ ] Services: business logic หลัก, ไม่มี HTTP concerns
- [ ] Tests: unit > integration > e2e (pyramid)
- [ ] Dependency injection: inject repositories เข้า services
- [ ] DRY: ไม่ copy-paste, แต่ไม่ over-abstract
- [ ] YAGNI: ไม่เพิ่ม feature ที่ไม่ต้องการวันนี้
- [ ] Type hints ครบ: ใช้ mypy ตรวจ

### 990.3 เตรียมตัวสำหรับ Part สุดท้าย

**Part 100: เส้นทางอาชีพ Django Developer และ Open Source** คือ Part สุดท้ายของหลักสูตร
จะแนะนำเส้นทางอาชีพ, วิธีการสัมภาษณ์งาน, การ contribute open source,
และก้าวต่อไปหลังจบหลักสูตร
