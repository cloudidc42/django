# Part 096: GraphQL กับ Django (Graphene)

> **ขั้นตอนที่ 951-960** | Phase 12: Scaling, Enterprise, โปรเจกต์จริง และเส้นทางอาชีพ

---

## ขั้นตอนที่ 951: GraphQL vs REST

### 951.1 ทำความเข้าใจ GraphQL

**REST**: client ขอข้อมูลตาม endpoint ที่ server กำหนด
**GraphQL**: client ขอข้อมูลที่ต้องการ field-by-field ผ่าน endpoint เดียว

```graphql
# REST: ต้องเรียก 2-3 endpoints
GET /api/posts/1/          → { id, title, content, author_id, ... }
GET /api/users/42/         → { id, name, email, ... }
GET /api/posts/1/comments/ → [{ id, body, author_id }, ...]

# GraphQL: เรียกครั้งเดียว ได้แค่ field ที่ต้องการ
query {
  post(id: 1) {
    title
    author {
      name
    }
    comments {
      body
    }
  }
}
```

### 951.2 เมื่อไหร่ควรใช้ GraphQL

**ใช้ GraphQL เมื่อ**:
- Mobile app ต้องการ bandwidth น้อย (ขอแค่ field ที่จำเป็น)
- Data model ซับซ้อน มี relationships เยอะ
- Client หลายประเภท (web, mobile, partner API) ต้องการ data format ต่างกัน
- Rapid product iteration (ไม่ต้องออก API version ใหม่ทุกครั้ง)

**ยังคงใช้ REST เมื่อ**:
- API ง่าย ไม่ซับซ้อน
- Caching ต้องการ HTTP cache header
- File upload/download
- ทีมไม่คุ้นเคยกับ GraphQL

---

## ขั้นตอนที่ 952: ติดตั้ง Graphene-Django

### 952.1 ติดตั้ง

```bash
pip install graphene-django strawberry-graphql-django  # สองทางเลือก
# แนะนำ strawberry สำหรับ project ใหม่ (type-hint ดีกว่า)
```

### 952.2 ตั้งค่า (Strawberry)

```python
# config/settings.py
INSTALLED_APPS += ['strawberry_django']

# config/urls.py
from strawberry.django.views import AsyncGraphQLView
from .schema import schema

urlpatterns = [
    path('graphql/', AsyncGraphQLView.as_view(schema=schema)),
]
```

### 952.3 Schema พื้นฐานด้วย Strawberry

```python
# schema.py
import strawberry
import strawberry_django
from strawberry import auto
from blog.models import Post, Tag
from accounts.models import User


@strawberry_django.type(User)
class UserType:
    id: auto
    username: auto
    email: auto


@strawberry_django.type(Tag)
class TagType:
    id: auto
    name: auto
    slug: auto


@strawberry_django.type(Post)
class PostType:
    id: auto
    title: auto
    content: auto
    created_at: auto
    author: UserType
    tags: list[TagType]


@strawberry.type
class Query:
    @strawberry.field
    def posts(self) -> list[PostType]:
        return Post.objects.select_related('author').prefetch_related('tags').all()

    @strawberry.field
    def post(self, id: int) -> PostType | None:
        try:
            return Post.objects.select_related('author').get(id=id)
        except Post.DoesNotExist:
            return None


schema = strawberry.Schema(query=Query)
```

---

## ขั้นตอนที่ 953: Queries ขั้นสูง

### 953.1 Filtering และ Pagination

```python
# schema.py
from strawberry_django import filters, pagination
import strawberry_django


@strawberry_django.filter(Post)
class PostFilter:
    title: auto
    status: auto
    author: auto
    created_at: auto


@strawberry_django.order(Post)
class PostOrder:
    created_at: auto
    title: auto


@strawberry.type
class Query:
    posts: strawberry.django.ListConnectionWithTotalCount[PostType] = \
        strawberry_django.connection(
            filters=PostFilter,
            order=PostOrder,
            pagination=True,
        )
```

```graphql
# GraphQL query พร้อม filtering + pagination
query {
  posts(
    filters: { status: "published" }
    order: { createdAt: DESC }
    pagination: { limit: 10, offset: 0 }
  ) {
    totalCount
    edges {
      node {
        id
        title
        author {
          username
        }
      }
    }
  }
}
```

### 953.2 Custom Resolvers

```python
@strawberry_django.type(Post)
class PostType:
    id: auto
    title: auto
    content: auto

    @strawberry.field
    def excerpt(self) -> str:
        """ย่อ content เป็น 200 characters แรก"""
        return self.content[:200] + '...' if len(self.content) > 200 else self.content

    @strawberry.field
    def reading_time_minutes(self) -> int:
        """คำนวณเวลาอ่าน (250 words/minute)"""
        word_count = len(self.content.split())
        return max(1, round(word_count / 250))
```

---

## ขั้นตอนที่ 954: Mutations

### 954.1 CRUD Mutations

```python
# schema.py
import strawberry
from strawberry_django import mutations


@strawberry_django.input(Post)
class CreatePostInput:
    title: auto
    content: auto
    tags: list[int] = strawberry.field(default_factory=list)


@strawberry_django.input(Post, partial=True)
class UpdatePostInput:
    id: auto
    title: auto
    content: auto


@strawberry.type
class Mutation:
    @strawberry.mutation
    def create_post(self, info: strawberry.types.Info, input: CreatePostInput) -> PostType:
        user = info.context.request.user
        if not user.is_authenticated:
            raise ValueError("ต้อง login ก่อน")

        post = Post.objects.create(
            title=input.title,
            content=input.content,
            author=user,
        )

        if input.tags:
            post.tags.set(input.tags)

        return post

    @strawberry.mutation
    def update_post(self, info: strawberry.types.Info, input: UpdatePostInput) -> PostType:
        user = info.context.request.user
        post = Post.objects.get(id=input.id)

        if post.author != user:
            raise PermissionError("ไม่มีสิทธิ์แก้ไข post นี้")

        if input.title is not strawberry.UNSET:
            post.title = input.title
        if input.content is not strawberry.UNSET:
            post.content = input.content
        post.save()
        return post

    @strawberry.mutation
    def delete_post(self, info: strawberry.types.Info, id: int) -> bool:
        user = info.context.request.user
        post = Post.objects.get(id=id)
        if post.author != user:
            raise PermissionError("ไม่มีสิทธิ์ลบ post นี้")
        post.delete()
        return True


schema = strawberry.Schema(query=Query, mutation=Mutation)
```

```graphql
# GraphQL mutation
mutation {
  createPost(input: {
    title: "บทความใหม่"
    content: "เนื้อหาบทความ..."
    tags: [1, 3]
  }) {
    id
    title
    author {
      username
    }
  }
}
```

---

## ขั้นตอนที่ 955: N+1 Problem และ DataLoader

### 955.1 ปัญหา N+1

```python
# ❌ N+1 Problem: query posts = 1, query author ต่อ post = N
@strawberry.type
class Query:
    @strawberry.field
    def posts(self) -> list[PostType]:
        return Post.objects.all()  # 1 query
        # เมื่อ resolve author → N queries (ทีละ post!)
```

### 955.2 แก้ด้วย strawberry-django (auto select_related)

```python
# ✅ วิธีที่ 1: ใช้ prefetch_related ใน resolver
@strawberry.field
def posts(self) -> list[PostType]:
    return Post.objects.select_related('author').prefetch_related('tags').all()
```

### 955.3 DataLoader สำหรับ Complex Cases

```python
# dataloaders.py
from strawberry.dataloader import DataLoader
from accounts.models import User


async def load_users_by_ids(keys: list[int]) -> list[User]:
    """Load หลาย users ในครั้งเดียว"""
    users = {u.id: u for u in await User.objects.filter(id__in=keys).aall()}
    return [users.get(key) for key in keys]


user_loader = DataLoader(load_fn=load_users_by_ids)


# schema.py
@strawberry_django.type(Post)
class PostType:
    id: auto
    title: auto

    @strawberry.field
    async def author(self, info: strawberry.types.Info) -> UserType:
        # DataLoader batch load ทุก author requests เข้าด้วยกัน → 1 query
        return await info.context['user_loader'].load(self.author_id)
```

---

## ขั้นตอนที่ 956: Authentication ใน GraphQL

### 956.1 JWT Authentication

```python
# schema.py
from functools import wraps


def login_required(func):
    """Decorator ตรวจสอบว่า user ล็อกอินแล้ว"""
    @wraps(func)
    def wrapper(root, info: strawberry.types.Info, **kwargs):
        if not info.context.request.user.is_authenticated:
            raise strawberry.PermissionError("ต้องล็อกอินก่อน")
        return func(root, info, **kwargs)
    return wrapper


@strawberry.type
class Mutation:
    @strawberry.mutation
    def login(self, username: str, password: str) -> str:
        """Login และรับ JWT token"""
        from django.contrib.auth import authenticate
        from rest_framework_simplejwt.tokens import RefreshToken

        user = authenticate(username=username, password=password)
        if not user:
            raise ValueError("username หรือ password ไม่ถูกต้อง")

        refresh = RefreshToken.for_user(user)
        return str(refresh.access_token)

    @strawberry.mutation
    @login_required
    def create_post(self, info: strawberry.types.Info, title: str, content: str) -> PostType:
        return Post.objects.create(title=title, content=content, author=info.context.request.user)
```

---

## ขั้นตอนที่ 957: Subscriptions

### 957.1 Real-time Subscriptions ด้วย WebSocket

```python
# schema.py
import asyncio
import strawberry
from typing import AsyncGenerator


@strawberry.type
class Subscription:
    @strawberry.subscription
    async def post_created(self) -> AsyncGenerator[PostType, None]:
        """Subscribe เพื่อรับ notification เมื่อมี post ใหม่"""
        from channels.layers import get_channel_layer
        from asgiref.sync import async_to_sync
        import json

        channel_layer = get_channel_layer()
        channel_name = await channel_layer.new_channel()

        await channel_layer.group_add('post_created', channel_name)
        try:
            while True:
                message = await channel_layer.receive(channel_name)
                post = Post.objects.get(id=message['post_id'])
                yield post
        finally:
            await channel_layer.group_discard('post_created', channel_name)


schema = strawberry.Schema(query=Query, mutation=Mutation, subscription=Subscription)
```

```python
# blog/signals.py — ส่ง signal เมื่อ post ถูกสร้าง
from django.db.models.signals import post_save
from channels.layers import get_channel_layer
from asgiref.sync import async_to_sync


def notify_post_created(sender, instance, created, **kwargs):
    if created:
        channel_layer = get_channel_layer()
        async_to_sync(channel_layer.group_send)(
            'post_created',
            {'type': 'post.created', 'post_id': instance.id}
        )


post_save.connect(notify_post_created, sender=Post)
```

---

## ขั้นตอนที่ 958: Error Handling

### 958.1 Custom Error Types

```python
# errors.py
import strawberry
from typing import Annotated, Union


@strawberry.type
class PostNotFoundError:
    message: str = "ไม่พบ Post ที่ระบุ"


@strawberry.type
class PermissionDeniedError:
    message: str = "ไม่มีสิทธิ์ดำเนินการนี้"


@strawberry.type
class ValidationError:
    message: str
    field: str


# Union type สำหรับ mutation result
PostResult = Annotated[
    Union[PostType, PostNotFoundError, PermissionDeniedError, ValidationError],
    strawberry.annotated_type
]


@strawberry.type
class Mutation:
    @strawberry.mutation
    def update_post(self, info: strawberry.types.Info, id: int, title: str) -> PostResult:
        user = info.context.request.user

        try:
            post = Post.objects.get(id=id)
        except Post.DoesNotExist:
            return PostNotFoundError()

        if post.author != user:
            return PermissionDeniedError()

        if not title.strip():
            return ValidationError(message="หัวข้อต้องไม่ว่าง", field="title")

        post.title = title
        post.save()
        return post
```

---

## ขั้นตอนที่ 959: Testing GraphQL

### 959.1 Test GraphQL Queries

```python
# tests/test_graphql.py
import pytest
from strawberry.test import TestClient
from config.schema import schema


@pytest.fixture
def graphql_client():
    return TestClient(schema)


def test_posts_query(graphql_client, published_post):
    """ทดสอบ query posts"""
    result = graphql_client.execute("""
        query {
          posts {
            id
            title
            author {
              username
            }
          }
        }
    """)

    assert result.errors is None
    assert len(result.data['posts']) == 1
    assert result.data['posts'][0]['title'] == published_post.title


def test_create_post_mutation(graphql_client, authenticated_user):
    """ทดสอบ mutation createPost"""
    result = graphql_client.execute(
        """
        mutation CreatePost($title: String!, $content: String!) {
          createPost(input: { title: $title, content: $content }) {
            ... on PostType {
              id
              title
            }
            ... on ValidationError {
              message
              field
            }
          }
        }
        """,
        variable_values={"title": "ทดสอบ", "content": "เนื้อหาทดสอบ"},
        context_value={"request": mock_request(authenticated_user)},
    )

    assert result.errors is None
    assert result.data['createPost']['title'] == 'ทดสอบ'
```

---

## ขั้นตอนที่ 960: สรุปและแบบฝึกหัด

### 960.1 Schema สมบูรณ์

```python
# config/schema.py
import strawberry
from .query import Query
from .mutation import Mutation
from .subscription import Subscription

schema = strawberry.Schema(
    query=Query,
    mutation=Mutation,
    subscription=Subscription,
    extensions=[
        # Complexity limiting (ป้องกัน expensive queries)
        QueryDepthLimiter(max_depth=10),
        MaxAliasesLimiter(max_alias_count=15),
    ]
)
```

### 960.2 Checklist GraphQL

- [ ] Schema type definitions ครบถ้วน
- [ ] N+1 problem แก้ด้วย select_related/prefetch หรือ DataLoader
- [ ] Auth check ใน mutations ทุกตัว
- [ ] Custom error types แทน raise Exception
- [ ] Query depth limiting เปิดไว้
- [ ] Tests ครอบคลุม happy path + error cases
- [ ] GraphQL playground เปิดเฉพาะ development

### 960.3 แบบฝึกหัดท้ายบท

**แบบฝึกหัดที่ 1**: เพิ่ม GraphQL endpoint ให้กับ blog project โดยใช้ Strawberry
เปิดเผย queries สำหรับ posts, tags, users และ mutations สำหรับ create/update/delete post

**แบบฝึกหัดที่ 2**: แก้ N+1 problem ใน posts query (ให้ author และ tags load ใน
1 query ต่อ list) ใช้ Django debug toolbar ตรวจสอบจำนวน queries ก่อนและหลัง

**แบบฝึกหัดที่ 3**: เพิ่ม Subscription สำหรับ "post_created" event ทดสอบด้วย
GraphQL playground โดยเปิด subscription tab แล้วสร้าง post ผ่าน mutation และดูว่า
subscription ได้รับ notification

**แบบฝึกหัดที่ 4 (ขั้นสูง)**: เขียน contract tests สำหรับ GraphQL API ทดสอบทั้ง
valid queries, invalid auth, validation errors, และ not found errors

### 960.4 เตรียมตัวสำหรับ Part ถัดไป

**Part 097: โปรเจกต์จริง: ระบบ E-Commerce แบบครบวงจร** จะนำความรู้ทั้งหมดตั้งแต่
Part 001 จนถึงตอนนี้มาสร้างระบบ E-Commerce จริงที่ใช้งานได้จริง: product catalog, cart,
checkout, payment integration, order management, inventory, reviews, และ admin dashboard
