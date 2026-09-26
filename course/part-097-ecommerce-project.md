# Part 097: โปรเจกต์จริง: ระบบ E-Commerce แบบครบวงจร

> **ขั้นตอนที่ 961-970** | Phase 12: Scaling, Enterprise, โปรเจกต์จริง และเส้นทางอาชีพ

---

## ขั้นตอนที่ 961: สถาปัตยกรรมและโครงสร้างโปรเจกต์

### 961.1 ภาพรวมระบบ E-Commerce

โปรเจกต์นี้จะสร้างระบบ E-Commerce ที่ใช้งานได้จริง ครอบคลุม:
- Product Catalog (สินค้า, หมวดหมู่, รูปภาพ, ราคา, สต็อก)
- Shopping Cart (ตะกร้าสินค้า + session/database)
- Order Management (คำสั่งซื้อ, สถานะ, ประวัติ)
- Payment Integration (Stripe / Omise)
- User Dashboard (ประวัติคำสั่งซื้อ, รีวิว)
- Admin Dashboard (สถิติ, จัดการสินค้า, จัดการคำสั่งซื้อ)

### 961.2 โครงสร้างโปรเจกต์

```
ecommerce/
├── config/
│   ├── settings/
│   │   ├── base.py
│   │   ├── development.py
│   │   └── production.py
│   ├── urls.py
│   └── wsgi.py
├── apps/
│   ├── catalog/          # สินค้าและหมวดหมู่
│   │   ├── models.py
│   │   ├── views.py
│   │   ├── urls.py
│   │   └── admin.py
│   ├── cart/             # ตะกร้าสินค้า
│   ├── orders/           # คำสั่งซื้อ
│   ├── payments/         # ชำระเงิน
│   ├── accounts/         # user และ profile
│   └── reviews/          # รีวิวสินค้า
├── templates/
├── static/
├── media/
├── requirements/
│   ├── base.txt
│   ├── development.txt
│   └── production.txt
└── docker-compose.yml
```

---

## ขั้นตอนที่ 962: Catalog App — Models

### 962.1 Product Models

```python
# apps/catalog/models.py
from django.db import models
from django.utils.text import slugify
from django.urls import reverse
from django.contrib.auth import get_user_model


class Category(models.Model):
    name = models.CharField(max_length=200)
    slug = models.SlugField(max_length=200, unique=True)
    parent = models.ForeignKey(
        'self', null=True, blank=True, related_name='children', on_delete=models.CASCADE
    )
    image = models.ImageField(upload_to='categories/', blank=True)
    is_active = models.BooleanField(default=True)

    class Meta:
        verbose_name_plural = 'categories'
        ordering = ['name']

    def __str__(self):
        return self.name

    def save(self, *args, **kwargs):
        if not self.slug:
            self.slug = slugify(self.name)
        super().save(*args, **kwargs)


class Product(models.Model):
    class Status(models.TextChoices):
        DRAFT = 'draft', 'แบบร่าง'
        ACTIVE = 'active', 'เปิดขาย'
        ARCHIVED = 'archived', 'ปิดขาย'

    category = models.ForeignKey(Category, related_name='products', on_delete=models.PROTECT)
    name = models.CharField(max_length=300)
    slug = models.SlugField(max_length=300, unique=True)
    description = models.TextField()
    short_description = models.CharField(max_length=500, blank=True)
    price = models.DecimalField(max_digits=10, decimal_places=2)
    sale_price = models.DecimalField(max_digits=10, decimal_places=2, null=True, blank=True)
    stock = models.PositiveIntegerField(default=0)
    sku = models.CharField(max_length=50, unique=True)
    status = models.CharField(max_length=10, choices=Status.choices, default=Status.DRAFT)
    thumbnail = models.ImageField(upload_to='products/thumbnails/')
    weight = models.DecimalField(max_digits=6, decimal_places=2, null=True, blank=True)
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)

    class Meta:
        ordering = ['-created_at']
        indexes = [
            models.Index(fields=['slug']),
            models.Index(fields=['status', 'category']),
        ]

    def __str__(self):
        return self.name

    def get_absolute_url(self):
        return reverse('catalog:product_detail', kwargs={'slug': self.slug})

    @property
    def effective_price(self):
        return self.sale_price if self.sale_price else self.price

    @property
    def is_in_stock(self):
        return self.stock > 0

    @property
    def discount_percentage(self):
        if self.sale_price and self.sale_price < self.price:
            return round((1 - self.sale_price / self.price) * 100)
        return 0


class ProductImage(models.Model):
    product = models.ForeignKey(Product, related_name='images', on_delete=models.CASCADE)
    image = models.ImageField(upload_to='products/images/')
    alt_text = models.CharField(max_length=200, blank=True)
    is_primary = models.BooleanField(default=False)
    order = models.PositiveIntegerField(default=0)

    class Meta:
        ordering = ['order']
```

---

## ขั้นตอนที่ 963: Cart App

### 963.1 Cart Models และ Session

```python
# apps/cart/models.py
from django.db import models
from apps.catalog.models import Product


class Cart(models.Model):
    """ตะกร้าสินค้า สำหรับ user ที่ล็อกอิน"""
    user = models.OneToOneField('accounts.User', on_delete=models.CASCADE)
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)

    def get_total_price(self):
        return sum(item.get_subtotal() for item in self.items.all())

    def get_total_items(self):
        return sum(item.quantity for item in self.items.all())


class CartItem(models.Model):
    cart = models.ForeignKey(Cart, related_name='items', on_delete=models.CASCADE)
    product = models.ForeignKey(Product, on_delete=models.CASCADE)
    quantity = models.PositiveIntegerField(default=1)
    added_at = models.DateTimeField(auto_now_add=True)

    class Meta:
        unique_together = ('cart', 'product')

    def get_subtotal(self):
        return self.product.effective_price * self.quantity
```

```python
# apps/cart/cart.py — Cart สำหรับ session (ไม่ต้องล็อกอิน)
from decimal import Decimal
from apps.catalog.models import Product

CART_SESSION_KEY = 'cart'


class SessionCart:
    """Cart ที่เก็บใน session, ไม่ต้องล็อกอิน"""

    def __init__(self, request):
        self.session = request.session
        cart = self.session.get(CART_SESSION_KEY)
        if not cart:
            cart = self.session[CART_SESSION_KEY] = {}
        self.cart = cart

    def add(self, product, quantity=1, override_quantity=False):
        product_id = str(product.id)
        if product_id not in self.cart:
            self.cart[product_id] = {'quantity': 0, 'price': str(product.effective_price)}
        if override_quantity:
            self.cart[product_id]['quantity'] = quantity
        else:
            self.cart[product_id]['quantity'] += quantity
        self.save()

    def remove(self, product):
        product_id = str(product.id)
        if product_id in self.cart:
            del self.cart[product_id]
            self.save()

    def save(self):
        self.session.modified = True

    def __iter__(self):
        product_ids = self.cart.keys()
        products = Product.objects.filter(id__in=product_ids)
        cart = self.cart.copy()
        for product in products:
            cart[str(product.id)]['product'] = product
        for item in cart.values():
            item['price'] = Decimal(item['price'])
            item['subtotal'] = item['price'] * item['quantity']
            yield item

    def __len__(self):
        return sum(item['quantity'] for item in self.cart.values())

    def get_total_price(self):
        return sum(Decimal(item['price']) * item['quantity'] for item in self.cart.values())

    def clear(self):
        del self.session[CART_SESSION_KEY]
        self.save()
```

---

## ขั้นตอนที่ 964: Orders App — Models

### 964.1 Order Models

```python
# apps/orders/models.py
from django.db import models
from django.contrib.auth import get_user_model
from apps.catalog.models import Product
import uuid

User = get_user_model()


class Order(models.Model):
    class Status(models.TextChoices):
        PENDING = 'pending', 'รอชำระเงิน'
        CONFIRMED = 'confirmed', 'ยืนยันแล้ว'
        PROCESSING = 'processing', 'กำลังเตรียม'
        SHIPPED = 'shipped', 'จัดส่งแล้ว'
        DELIVERED = 'delivered', 'ได้รับแล้ว'
        CANCELLED = 'cancelled', 'ยกเลิกแล้ว'
        REFUNDED = 'refunded', 'คืนเงินแล้ว'

    order_number = models.CharField(max_length=20, unique=True)
    user = models.ForeignKey(User, on_delete=models.PROTECT, related_name='orders')
    status = models.CharField(max_length=20, choices=Status.choices, default=Status.PENDING)

    # ที่อยู่จัดส่ง (snapshot ณ เวลาสั่ง)
    shipping_first_name = models.CharField(max_length=100)
    shipping_last_name = models.CharField(max_length=100)
    shipping_address = models.TextField()
    shipping_city = models.CharField(max_length=100)
    shipping_province = models.CharField(max_length=100)
    shipping_postal_code = models.CharField(max_length=10)
    shipping_phone = models.CharField(max_length=20)

    subtotal = models.DecimalField(max_digits=10, decimal_places=2)
    shipping_cost = models.DecimalField(max_digits=8, decimal_places=2, default=0)
    discount_amount = models.DecimalField(max_digits=8, decimal_places=2, default=0)
    total = models.DecimalField(max_digits=10, decimal_places=2)

    notes = models.TextField(blank=True)
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)

    class Meta:
        ordering = ['-created_at']

    def save(self, *args, **kwargs):
        if not self.order_number:
            self.order_number = self._generate_order_number()
        super().save(*args, **kwargs)

    @staticmethod
    def _generate_order_number():
        import random, string
        prefix = 'ORD'
        suffix = ''.join(random.choices(string.digits, k=10))
        return f'{prefix}{suffix}'


class OrderItem(models.Model):
    order = models.ForeignKey(Order, related_name='items', on_delete=models.PROTECT)
    product = models.ForeignKey(Product, on_delete=models.PROTECT)
    product_name = models.CharField(max_length=300)   # snapshot ชื่อสินค้า
    product_sku = models.CharField(max_length=50)      # snapshot SKU
    unit_price = models.DecimalField(max_digits=10, decimal_places=2)
    quantity = models.PositiveIntegerField()
    subtotal = models.DecimalField(max_digits=10, decimal_places=2)

    def save(self, *args, **kwargs):
        self.subtotal = self.unit_price * self.quantity
        super().save(*args, **kwargs)
```

---

## ขั้นตอนที่ 965: Order Views และ Flow

### 965.1 Checkout View

```python
# apps/orders/views.py
from django.shortcuts import render, redirect, get_object_or_404
from django.contrib.auth.decorators import login_required
from django.db import transaction
from apps.cart.cart import SessionCart
from .models import Order, OrderItem
from .forms import CheckoutForm


@login_required
def checkout(request):
    """หน้า checkout: กรอกที่อยู่และยืนยันคำสั่งซื้อ"""
    cart = SessionCart(request)

    if len(cart) == 0:
        return redirect('cart:detail')

    if request.method == 'POST':
        form = CheckoutForm(request.POST)
        if form.is_valid():
            with transaction.atomic():
                order = _create_order_from_cart(request.user, form.cleaned_data, cart)
                cart.clear()
            return redirect('orders:payment', order_number=order.order_number)
    else:
        # pre-fill ด้วยข้อมูล profile
        initial = {}
        profile = getattr(request.user, 'profile', None)
        if profile:
            initial = {
                'first_name': request.user.first_name,
                'last_name': request.user.last_name,
                'phone': profile.phone,
                'address': profile.address,
            }
        form = CheckoutForm(initial=initial)

    return render(request, 'orders/checkout.html', {'cart': cart, 'form': form})


def _create_order_from_cart(user, cleaned_data, cart):
    """สร้าง Order จาก cart (ทำใน transaction)"""
    subtotal = cart.get_total_price()
    shipping_cost = _calculate_shipping(subtotal)
    total = subtotal + shipping_cost

    order = Order.objects.create(
        user=user,
        shipping_first_name=cleaned_data['first_name'],
        shipping_last_name=cleaned_data['last_name'],
        shipping_address=cleaned_data['address'],
        shipping_city=cleaned_data['city'],
        shipping_province=cleaned_data['province'],
        shipping_postal_code=cleaned_data['postal_code'],
        shipping_phone=cleaned_data['phone'],
        subtotal=subtotal,
        shipping_cost=shipping_cost,
        total=total,
        notes=cleaned_data.get('notes', ''),
    )

    for item in cart:
        product = item['product']
        if product.stock < item['quantity']:
            raise ValueError(f"สต็อก {product.name} ไม่เพียงพอ")
        OrderItem.objects.create(
            order=order,
            product=product,
            product_name=product.name,
            product_sku=product.sku,
            unit_price=item['price'],
            quantity=item['quantity'],
        )
        # ลดสต็อก
        product.stock -= item['quantity']
        product.save(update_fields=['stock'])

    return order


def _calculate_shipping(subtotal):
    from decimal import Decimal
    if subtotal >= 1000:
        return Decimal('0')  # ส่งฟรีเมื่อซื้อ >= 1000 บาท
    return Decimal('50')
```

---

## ขั้นตอนที่ 966: Payment Integration

### 966.1 Stripe Payment

```python
# apps/payments/views.py
import stripe
from django.conf import settings
from django.http import JsonResponse, HttpResponse
from django.views.decorators.csrf import csrf_exempt
from django.contrib.auth.decorators import login_required
from apps.orders.models import Order

stripe.api_key = settings.STRIPE_SECRET_KEY


@login_required
def create_payment_intent(request, order_number):
    """สร้าง Stripe PaymentIntent สำหรับ order"""
    order = Order.objects.get(
        order_number=order_number,
        user=request.user,
        status=Order.Status.PENDING,
    )

    # สร้าง PaymentIntent (หน่วยเป็น satang = สตางค์)
    intent = stripe.PaymentIntent.create(
        amount=int(order.total * 100),
        currency='thb',
        metadata={'order_number': order.order_number},
    )

    return JsonResponse({'client_secret': intent.client_secret})


@csrf_exempt
def stripe_webhook(request):
    """รับ Stripe Webhook events"""
    payload = request.body
    sig_header = request.META.get('HTTP_STRIPE_SIGNATURE')

    try:
        event = stripe.Webhook.construct_event(
            payload, sig_header, settings.STRIPE_WEBHOOK_SECRET
        )
    except stripe.error.SignatureVerificationError:
        return HttpResponse(status=400)

    if event['type'] == 'payment_intent.succeeded':
        payment_intent = event['data']['object']
        order_number = payment_intent['metadata']['order_number']
        _handle_payment_success(order_number, payment_intent['id'])

    elif event['type'] == 'payment_intent.payment_failed':
        payment_intent = event['data']['object']
        order_number = payment_intent['metadata']['order_number']
        _handle_payment_failure(order_number)

    return HttpResponse(status=200)


def _handle_payment_success(order_number, payment_id):
    """อัปเดต order เมื่อชำระเงินสำเร็จ"""
    from django.db import transaction
    from apps.orders.tasks import send_order_confirmation_email

    with transaction.atomic():
        order = Order.objects.select_for_update().get(order_number=order_number)
        order.status = Order.Status.CONFIRMED
        order.save(update_fields=['status'])

        Payment.objects.create(
            order=order,
            payment_id=payment_id,
            amount=order.total,
            status='completed',
        )

    send_order_confirmation_email.delay(order.id)
```

---

## ขั้นตอนที่ 967: Reviews และ Ratings

### 967.1 Review Models และ Views

```python
# apps/reviews/models.py
from django.db import models
from django.contrib.auth import get_user_model
from apps.catalog.models import Product

User = get_user_model()


class Review(models.Model):
    product = models.ForeignKey(Product, related_name='reviews', on_delete=models.CASCADE)
    user = models.ForeignKey(User, on_delete=models.CASCADE)
    rating = models.PositiveSmallIntegerField(
        choices=[(i, i) for i in range(1, 6)]
    )
    title = models.CharField(max_length=200)
    body = models.TextField()
    is_verified_purchase = models.BooleanField(default=False)
    is_approved = models.BooleanField(default=True)
    created_at = models.DateTimeField(auto_now_add=True)

    class Meta:
        unique_together = ('product', 'user')  # รีวิวได้ครั้งเดียวต่อสินค้า

    def save(self, *args, **kwargs):
        # ตรวจว่า user เคยซื้อสินค้านี้จริง
        from apps.orders.models import Order, OrderItem
        has_purchased = OrderItem.objects.filter(
            product=self.product,
            order__user=self.user,
            order__status__in=['confirmed', 'delivered'],
        ).exists()
        self.is_verified_purchase = has_purchased
        super().save(*args, **kwargs)
        # อัปเดต rating สรุปของสินค้า
        self._update_product_rating()

    def _update_product_rating(self):
        from django.db.models import Avg, Count
        stats = Review.objects.filter(product=self.product, is_approved=True).aggregate(
            avg=Avg('rating'), count=Count('id')
        )
        ProductRating.objects.update_or_create(
            product=self.product,
            defaults={'average': stats['avg'] or 0, 'count': stats['count']},
        )


class ProductRating(models.Model):
    """Denormalized rating summary เพื่อ performance"""
    product = models.OneToOneField(Product, related_name='rating_summary', on_delete=models.CASCADE)
    average = models.DecimalField(max_digits=3, decimal_places=2, default=0)
    count = models.PositiveIntegerField(default=0)
```

---

## ขั้นตอนที่ 968: Admin Dashboard

### 968.1 Custom Admin Views สำหรับสถิติ

```python
# apps/orders/admin.py
from django.contrib import admin
from django.db.models import Sum, Count
from django.urls import path
from django.shortcuts import render
from django.utils import timezone
from .models import Order, OrderItem


class OrderAdmin(admin.ModelAdmin):
    list_display = ['order_number', 'user', 'status', 'total', 'created_at']
    list_filter = ['status', 'created_at']
    search_fields = ['order_number', 'user__email']
    readonly_fields = ['order_number', 'created_at', 'updated_at']
    raw_id_fields = ['user']

    actions = ['mark_as_processing', 'mark_as_shipped']

    @admin.action(description='เปลี่ยนสถานะเป็น กำลังเตรียม')
    def mark_as_processing(self, request, queryset):
        updated = queryset.filter(status=Order.Status.CONFIRMED).update(
            status=Order.Status.PROCESSING
        )
        self.message_user(request, f'อัปเดต {updated} คำสั่งซื้อ')

    @admin.action(description='เปลี่ยนสถานะเป็น จัดส่งแล้ว')
    def mark_as_shipped(self, request, queryset):
        updated = queryset.filter(status=Order.Status.PROCESSING).update(
            status=Order.Status.SHIPPED
        )
        self.message_user(request, f'อัปเดต {updated} คำสั่งซื้อ')

    def get_urls(self):
        urls = super().get_urls()
        custom_urls = [
            path('dashboard/', self.admin_site.admin_view(self.dashboard_view), name='orders_dashboard'),
        ]
        return custom_urls + urls

    def dashboard_view(self, request):
        today = timezone.now().date()
        stats = {
            'today_orders': Order.objects.filter(created_at__date=today).count(),
            'today_revenue': Order.objects.filter(
                created_at__date=today, status=Order.Status.CONFIRMED
            ).aggregate(total=Sum('total'))['total'] or 0,
            'pending_orders': Order.objects.filter(status=Order.Status.PENDING).count(),
            'total_orders': Order.objects.count(),
        }
        return render(request, 'admin/orders/dashboard.html', stats)


admin.site.register(Order, OrderAdmin)
```

---

## ขั้นตอนที่ 969: Celery Tasks และ Email

### 969.1 Background Tasks

```python
# apps/orders/tasks.py
from celery import shared_task
from django.core.mail import send_mail
from django.template.loader import render_to_string
from django.conf import settings


@shared_task(bind=True, max_retries=3)
def send_order_confirmation_email(self, order_id):
    """ส่งอีเมลยืนยันคำสั่งซื้อ"""
    from .models import Order
    try:
        order = Order.objects.select_related('user').prefetch_related('items__product').get(id=order_id)

        subject = f'ยืนยันคำสั่งซื้อ #{order.order_number}'
        html_body = render_to_string('emails/order_confirmation.html', {'order': order})
        text_body = render_to_string('emails/order_confirmation.txt', {'order': order})

        send_mail(
            subject=subject,
            message=text_body,
            from_email=settings.DEFAULT_FROM_EMAIL,
            recipient_list=[order.user.email],
            html_message=html_body,
        )
    except Exception as exc:
        raise self.retry(exc=exc, countdown=60)


@shared_task
def check_low_stock():
    """ตรวจสต็อกสินค้าใกล้หมด — ทำงานทุกวัน 8 โมงเช้า"""
    from apps.catalog.models import Product
    low_stock = Product.objects.filter(
        status=Product.Status.ACTIVE,
        stock__lte=10,
        stock__gt=0,
    )
    if low_stock.exists():
        product_list = '\n'.join(f'- {p.name}: {p.stock} ชิ้น' for p in low_stock)
        send_mail(
            subject='แจ้งเตือน: สต็อกสินค้าใกล้หมด',
            message=f'สินค้าต่อไปนี้มีสต็อกน้อย:\n{product_list}',
            from_email=settings.DEFAULT_FROM_EMAIL,
            recipient_list=[settings.ADMIN_EMAIL],
        )
```

---

## ขั้นตอนที่ 970: สรุปและแบบฝึกหัด E-Commerce

### 970.1 Checklist โปรเจกต์

- [ ] Catalog: product listing + detail + category filter + search
- [ ] Cart: add/remove/update quantity (session + database)
- [ ] Checkout: address form + shipping calculation
- [ ] Payment: Stripe integration + webhook
- [ ] Order: history + status tracking + email confirmation
- [ ] Reviews: rating + verified purchase badge
- [ ] Admin: dashboard สถิติ + bulk actions
- [ ] Tests: coverage > 80% สำหรับ views และ models หลัก
- [ ] Security: CSRF, login required, ownership check ทุก order view

### 970.2 แบบฝึกหัดขยายโปรเจกต์

**แบบฝึกหัดที่ 1**: เพิ่มระบบ Coupon/Discount Code
สร้าง model Coupon (code, discount_type=percent/fixed, value, min_order, expiry_date, max_uses)
และ validate ใน checkout

**แบบฝึกหัดที่ 2**: เพิ่ม Product Variants
สร้าง ProductVariant model (color, size) พร้อม stock ต่าง variant และราคาต่างกัน
แก้ CartItem ให้เลือก variant ได้

**แบบฝึกหัดที่ 3**: เพิ่ม Wishlist
User เพิ่มสินค้าเข้า wishlist (model: Wishlist + WishlistItem)
แสดงใน user dashboard และ button "ย้ายไป cart"

**แบบฝึกหัดที่ 4 (ขั้นสูง)**: เพิ่ม Recommendation Engine
แสดง "สินค้าที่มักซื้อพร้อมกัน" โดย query orders ที่มีสินค้าเดียวกัน
(collaborative filtering แบบง่าย)

**แบบฝึกหัดที่ 5 (ขั้นสูง)**: เพิ่ม Inventory Management
สร้าง StockMovement model บันทึกทุกการเปลี่ยนแปลง stock (เติม, ขาย, ยกเลิก, คืน)
แสดง stock history และ low stock alert

### 970.3 เตรียมตัวสำหรับ Part ถัดไป

**Part 098: โปรเจกต์จริง: Social Media Platform** จะสร้าง Twitter-like platform
ด้วย Django, Channels (real-time), Celery (notifications), และ REST API
