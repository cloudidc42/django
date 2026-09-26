# Part 098: โปรเจกต์จริง: ระบบ Social Media Platform

> **ขั้นตอนที่ 971-980** | Phase 12: Scaling, Enterprise, โปรเจกต์จริง และเส้นทางอาชีพ

---

## ขั้นตอนที่ 971: สถาปัตยกรรม Social Media Platform

### 971.1 ฟีเจอร์หลัก

โปรเจกต์นี้สร้าง Twitter-like platform ครอบคลุม:
- โพสต์สถานะ (Post/Tweet) พร้อมรูปภาพหรือวิดีโอ
- Follow/Unfollow ระหว่าง users
- Like และ Repost
- Comment (nested replies)
- Real-time notifications (Django Channels)
- Newsfeed (Timeline ของคนที่ติดตาม)
- Search (users + posts)
- Hashtags
- Direct Messages (DM)

### 971.2 โครงสร้างโปรเจกต์

```
social/
├── config/
├── apps/
│   ├── accounts/     # User profiles, followers
│   ├── posts/        # Posts, likes, reposts, hashtags
│   ├── comments/     # Comments + replies
│   ├── notifications/# Real-time notifications
│   ├── messages/     # Direct messages
│   └── feed/         # Newsfeed algorithm
├── templates/
├── static/
└── manage.py
```

---

## ขั้นตอนที่ 972: User Profile และ Follow System

### 972.1 Custom User Model

```python
# apps/accounts/models.py
from django.contrib.auth.models import AbstractUser
from django.db import models


class User(AbstractUser):
    bio = models.TextField(max_length=500, blank=True)
    avatar = models.ImageField(upload_to='avatars/', blank=True)
    header_image = models.ImageField(upload_to='headers/', blank=True)
    website = models.URLField(blank=True)
    location = models.CharField(max_length=100, blank=True)
    is_verified = models.BooleanField(default=False)

    @property
    def followers_count(self):
        return self.followers.count()

    @property
    def following_count(self):
        return self.following.count()

    def follow(self, user):
        Follow.objects.get_or_create(follower=self, following=user)

    def unfollow(self, user):
        Follow.objects.filter(follower=self, following=user).delete()

    def is_following(self, user):
        return self.following.filter(following=user).exists()


class Follow(models.Model):
    follower = models.ForeignKey(User, related_name='following', on_delete=models.CASCADE)
    following = models.ForeignKey(User, related_name='followers', on_delete=models.CASCADE)
    created_at = models.DateTimeField(auto_now_add=True)

    class Meta:
        unique_together = ('follower', 'following')
        constraints = [
            models.CheckConstraint(
                check=~models.Q(follower=models.F('following')),
                name='no_self_follow'
            )
        ]
```

---

## ขั้นตอนที่ 973: Posts App

### 973.1 Post Models

```python
# apps/posts/models.py
from django.db import models
from django.contrib.auth import get_user_model

User = get_user_model()


class Post(models.Model):
    author = models.ForeignKey(User, related_name='posts', on_delete=models.CASCADE)
    content = models.TextField(max_length=280)
    parent = models.ForeignKey(
        'self', null=True, blank=True, related_name='replies', on_delete=models.CASCADE
    )  # สำหรับ reply
    repost_of = models.ForeignKey(
        'self', null=True, blank=True, related_name='reposts', on_delete=models.CASCADE
    )  # สำหรับ repost
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)

    class Meta:
        ordering = ['-created_at']
        indexes = [
            models.Index(fields=['author', '-created_at']),
        ]

    @property
    def likes_count(self):
        return self.likes.count()

    @property
    def reposts_count(self):
        return Post.objects.filter(repost_of=self).count()

    @property
    def replies_count(self):
        return self.replies.count()

    def extract_hashtags(self):
        import re
        return re.findall(r'#(\w+)', self.content)

    def save(self, *args, **kwargs):
        super().save(*args, **kwargs)
        # บันทึก hashtags
        for tag_name in self.extract_hashtags():
            tag, _ = Hashtag.objects.get_or_create(name=tag_name.lower())
            PostHashtag.objects.get_or_create(post=self, hashtag=tag)


class PostMedia(models.Model):
    class MediaType(models.TextChoices):
        IMAGE = 'image', 'รูปภาพ'
        VIDEO = 'video', 'วิดีโอ'

    post = models.ForeignKey(Post, related_name='media', on_delete=models.CASCADE)
    file = models.FileField(upload_to='posts/media/')
    media_type = models.CharField(max_length=10, choices=MediaType.choices)
    order = models.PositiveSmallIntegerField(default=0)


class Like(models.Model):
    post = models.ForeignKey(Post, related_name='likes', on_delete=models.CASCADE)
    user = models.ForeignKey(User, on_delete=models.CASCADE)
    created_at = models.DateTimeField(auto_now_add=True)

    class Meta:
        unique_together = ('post', 'user')


class Hashtag(models.Model):
    name = models.CharField(max_length=100, unique=True)
    created_at = models.DateTimeField(auto_now_add=True)


class PostHashtag(models.Model):
    post = models.ForeignKey(Post, on_delete=models.CASCADE)
    hashtag = models.ForeignKey(Hashtag, related_name='posts', on_delete=models.CASCADE)

    class Meta:
        unique_together = ('post', 'hashtag')
```

---

## ขั้นตอนที่ 974: Newsfeed Algorithm

### 974.1 Fan-out on Write vs Fan-out on Read

```python
# apps/feed/models.py
from django.db import models
from django.contrib.auth import get_user_model

User = get_user_model()


class FeedItem(models.Model):
    """Fan-out on Write: pre-computed feed สำหรับแต่ละ user"""
    user = models.ForeignKey(User, related_name='feed_items', on_delete=models.CASCADE)
    post = models.ForeignKey('posts.Post', on_delete=models.CASCADE)
    created_at = models.DateTimeField(auto_now_add=True)

    class Meta:
        unique_together = ('user', 'post')
        ordering = ['-created_at']
        indexes = [
            models.Index(fields=['user', '-created_at']),
        ]
```

```python
# apps/posts/tasks.py
from celery import shared_task


@shared_task
def distribute_post_to_feeds(post_id):
    """
    Fan-out on Write: เมื่อ user โพสต์ กระจาย post ไปยัง feed ของทุก follower
    เหมาะสำหรับ user ที่มี followers ไม่มากเกินไป
    """
    from apps.posts.models import Post
    from apps.accounts.models import Follow
    from apps.feed.models import FeedItem

    post = Post.objects.get(id=post_id)
    followers = Follow.objects.filter(following=post.author).values_list('follower_id', flat=True)

    feed_items = [
        FeedItem(user_id=follower_id, post=post)
        for follower_id in followers
    ]
    FeedItem.objects.bulk_create(feed_items, ignore_conflicts=True)

    # รักษา feed ไม่เกิน 1000 items ต่อ user
    for follower_id in followers:
        old_items = FeedItem.objects.filter(user_id=follower_id).order_by('-created_at')[1000:]
        if old_items.exists():
            FeedItem.objects.filter(id__in=old_items.values('id')).delete()
```

---

## ขั้นตอนที่ 975: Real-time Notifications

### 975.1 Notification Models

```python
# apps/notifications/models.py
from django.db import models
from django.contrib.auth import get_user_model
from django.contrib.contenttypes.fields import GenericForeignKey
from django.contrib.contenttypes.models import ContentType

User = get_user_model()


class Notification(models.Model):
    class Type(models.TextChoices):
        LIKE = 'like', 'ถูกใจโพสต์'
        REPOST = 'repost', 'รีโพสต์'
        REPLY = 'reply', 'ตอบกลับ'
        FOLLOW = 'follow', 'มีผู้ติดตามใหม่'
        MENTION = 'mention', 'ถูกกล่าวถึง'

    recipient = models.ForeignKey(User, related_name='notifications', on_delete=models.CASCADE)
    sender = models.ForeignKey(User, on_delete=models.CASCADE)
    notification_type = models.CharField(max_length=20, choices=Type.choices)

    # Generic FK — ชี้ไปที่ Post, Follow, ฯลฯ
    content_type = models.ForeignKey(ContentType, on_delete=models.CASCADE)
    object_id = models.PositiveIntegerField()
    content_object = GenericForeignKey('content_type', 'object_id')

    is_read = models.BooleanField(default=False)
    created_at = models.DateTimeField(auto_now_add=True)

    class Meta:
        ordering = ['-created_at']
```

### 975.2 WebSocket Consumer

```python
# apps/notifications/consumers.py
import json
from channels.generic.websocket import AsyncWebsocketConsumer
from channels.db import database_sync_to_async


class NotificationConsumer(AsyncWebsocketConsumer):
    async def connect(self):
        user = self.scope['user']
        if not user.is_authenticated:
            await self.close()
            return

        self.group_name = f'notifications_{user.id}'
        await self.channel_layer.group_add(self.group_name, self.channel_name)
        await self.accept()

        # ส่ง unread count ทันที
        unread_count = await self.get_unread_count(user)
        await self.send(json.dumps({'type': 'unread_count', 'count': unread_count}))

    async def disconnect(self, close_code):
        await self.channel_layer.group_discard(self.group_name, self.channel_name)

    async def notification_message(self, event):
        """รับ message จาก group และส่งให้ WebSocket client"""
        await self.send(json.dumps(event['data']))

    @database_sync_to_async
    def get_unread_count(self, user):
        from .models import Notification
        return Notification.objects.filter(recipient=user, is_read=False).count()
```

```python
# apps/notifications/tasks.py
from celery import shared_task
from asgiref.sync import async_to_sync
from channels.layers import get_channel_layer


@shared_task
def send_notification(recipient_id, sender_id, notification_type, object_id, content_type_id):
    """สร้าง Notification และส่ง WebSocket event"""
    from django.contrib.auth import get_user_model
    from django.contrib.contenttypes.models import ContentType
    from .models import Notification

    User = get_user_model()
    recipient = User.objects.get(id=recipient_id)
    sender = User.objects.get(id=sender_id)

    if recipient == sender:
        return  # ไม่แจ้งเตือนตัวเอง

    notif = Notification.objects.create(
        recipient=recipient,
        sender=sender,
        notification_type=notification_type,
        object_id=object_id,
        content_type_id=content_type_id,
    )

    # ส่ง WebSocket event
    channel_layer = get_channel_layer()
    async_to_sync(channel_layer.group_send)(
        f'notifications_{recipient_id}',
        {
            'type': 'notification.message',
            'data': {
                'id': notif.id,
                'type': notification_type,
                'sender': sender.username,
                'sender_avatar': sender.avatar.url if sender.avatar else None,
                'created_at': notif.created_at.isoformat(),
            },
        }
    )
```

---

## ขั้นตอนที่ 976: Direct Messages

### 976.1 DM Models

```python
# apps/messages/models.py
from django.db import models
from django.contrib.auth import get_user_model

User = get_user_model()


class Conversation(models.Model):
    """การสนทนาระหว่าง 2 users"""
    participants = models.ManyToManyField(User, related_name='conversations')
    created_at = models.DateTimeField(auto_now_add=True)
    last_message_at = models.DateTimeField(null=True, blank=True)

    class Meta:
        ordering = ['-last_message_at']

    @classmethod
    def get_or_create_between(cls, user1, user2):
        """หรือสร้าง conversation ระหว่าง 2 users"""
        existing = cls.objects.filter(
            participants=user1
        ).filter(
            participants=user2
        ).first()
        if existing:
            return existing, False
        conversation = cls.objects.create()
        conversation.participants.add(user1, user2)
        return conversation, True


class DirectMessage(models.Model):
    conversation = models.ForeignKey(Conversation, related_name='messages', on_delete=models.CASCADE)
    sender = models.ForeignKey(User, on_delete=models.CASCADE)
    content = models.TextField(max_length=1000)
    is_read = models.BooleanField(default=False)
    created_at = models.DateTimeField(auto_now_add=True)

    class Meta:
        ordering = ['created_at']
```

---

## ขั้นตอนที่ 977: REST API สำหรับ Mobile App

### 977.1 Post API

```python
# apps/posts/serializers.py
from rest_framework import serializers
from .models import Post, Like


class PostSerializer(serializers.ModelSerializer):
    author = serializers.SerializerMethodField()
    likes_count = serializers.IntegerField(read_only=True)
    is_liked = serializers.SerializerMethodField()
    media = serializers.SerializerMethodField()

    class Meta:
        model = Post
        fields = ['id', 'content', 'author', 'likes_count', 'is_liked', 'media', 'created_at']
        read_only_fields = ['author', 'likes_count', 'created_at']

    def get_author(self, obj):
        return {
            'id': obj.author.id,
            'username': obj.author.username,
            'avatar': obj.author.avatar.url if obj.author.avatar else None,
            'is_verified': obj.author.is_verified,
        }

    def get_is_liked(self, obj):
        request = self.context.get('request')
        if request and request.user.is_authenticated:
            return Like.objects.filter(post=obj, user=request.user).exists()
        return False

    def get_media(self, obj):
        return [
            {'url': m.file.url, 'type': m.media_type}
            for m in obj.media.all()
        ]


class PostCreateSerializer(serializers.ModelSerializer):
    class Meta:
        model = Post
        fields = ['content', 'parent']

    def create(self, validated_data):
        validated_data['author'] = self.context['request'].user
        return super().create(validated_data)
```

---

## ขั้นตอนที่ 978: Search

### 978.1 Full-Text Search

```python
# apps/posts/views.py (search endpoint)
from django.db.models import Q
from rest_framework.decorators import api_view, permission_classes
from rest_framework.permissions import AllowAny
from rest_framework.response import Response
from .models import Post
from .serializers import PostSerializer
from apps.accounts.models import User


@api_view(['GET'])
@permission_classes([AllowAny])
def search(request):
    q = request.query_params.get('q', '').strip()
    search_type = request.query_params.get('type', 'posts')  # posts / users

    if not q:
        return Response({'results': []})

    if search_type == 'users':
        results = User.objects.filter(
            Q(username__icontains=q) | Q(first_name__icontains=q)
        )[:20]
        data = [
            {
                'id': u.id,
                'username': u.username,
                'avatar': u.avatar.url if u.avatar else None,
                'followers_count': u.followers_count,
            }
            for u in results
        ]
    else:
        # Full-text search ด้วย PostgreSQL
        from django.contrib.postgres.search import SearchVector, SearchQuery, SearchRank

        vector = SearchVector('content', config='thai')
        query = SearchQuery(q, config='thai')

        posts = Post.objects.annotate(
            rank=SearchRank(vector, query)
        ).filter(rank__gte=0.01).order_by('-rank')[:20]

        data = PostSerializer(posts, many=True, context={'request': request}).data

    return Response({'results': data, 'query': q, 'type': search_type})
```

---

## ขั้นตอนที่ 979: Performance Optimization

### 979.1 Caching Newsfeed

```python
# apps/feed/views.py
from django.core.cache import cache
from rest_framework.views import APIView
from rest_framework.permissions import IsAuthenticated
from rest_framework.response import Response
from apps.posts.models import Post
from apps.posts.serializers import PostSerializer


class FeedView(APIView):
    permission_classes = [IsAuthenticated]

    def get(self, request):
        page = int(request.query_params.get('page', 1))
        page_size = 20
        cache_key = f'feed:{request.user.id}:page:{page}'

        cached = cache.get(cache_key)
        if cached:
            return Response(cached)

        from apps.feed.models import FeedItem
        feed_items = FeedItem.objects.filter(
            user=request.user
        ).select_related(
            'post__author', 'post__author__profile'
        ).prefetch_related(
            'post__media', 'post__likes'
        )[(page - 1) * page_size:page * page_size]

        posts = [item.post for item in feed_items]
        data = PostSerializer(posts, many=True, context={'request': request}).data

        cache.set(cache_key, {'results': data}, timeout=60)  # cache 1 นาที
        return Response({'results': data})
```

---

## ขั้นตอนที่ 980: สรุปและแบบฝึกหัด Social Media

### 980.1 Checklist โปรเจกต์

- [ ] User: registration, login, profile edit, avatar upload
- [ ] Follow/Unfollow + followers/following lists
- [ ] Post: create, delete, media upload
- [ ] Like/Unlike (toggle)
- [ ] Repost
- [ ] Reply (nested)
- [ ] Newsfeed (fan-out on write)
- [ ] Notifications (real-time WebSocket)
- [ ] Direct Messages (WebSocket)
- [ ] Hashtag browsing
- [ ] Search (users + posts)
- [ ] API (REST) พร้อม JWT auth

### 980.2 แบบฝึกหัดขยายโปรเจกต์

**แบบฝึกหัดที่ 1**: เพิ่ม Trending Hashtags
คำนวณ hashtag ที่ถูกใช้มากที่สุดใน 24 ชั่วโมงที่ผ่านมา
cache ผลลัพธ์ด้วย Redis (อัปเดตทุก 15 นาที ด้วย Celery periodic task)

**แบบฝึกหัดที่ 2**: เพิ่ม Bookmark
User บันทึก post ไว้ดูทีหลัง model: Bookmark(user, post, created_at)
แสดงใน "Bookmarks" tab ของ user

**แบบฝึกหัดที่ 3**: เพิ่ม Block User
User block user อื่น model: Block(blocker, blocked)
เมื่อ block แล้ว: ซ่อน post จาก feed, ซ่อน notification, ป้องกัน DM

**แบบฝึกหัดที่ 4 (ขั้นสูง)**: เพิ่ม Polls ในโพสต์
Post มี poll (หัวข้อ + ตัวเลือก + กำหนดเวลา) user โหวตได้ครั้งเดียว
แสดงผลเป็น % real-time ผ่าน WebSocket

### 980.3 เตรียมตัวสำหรับ Part ถัดไป

**Part 099: Django Design Patterns และ Clean Architecture** จะสรุป patterns สำคัญที่ทำให้ code
maintainable และ testable ในระยะยาว
