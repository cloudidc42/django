# Part 041: ModelSerializer และ Nested Serializers

> **ขั้นตอนที่ 401-410 ของหลักสูตร** | Phase 5: Django REST Framework
>
> เป้าหมายของ Part นี้: เปลี่ยนจาก `serializers.Serializer` แบบเขียนมือทุก field
> (ที่คุณฝึกไว้ใน Part 040) มาใช้ **`ModelSerializer`** ที่ generate field ให้อัตโนมัติ
> จาก `Model` แล้วต่อยอดไปสู่เรื่องที่มือใหม่ DRF แทบทุกคนเคยงงอย่างน้อยหนึ่งครั้ง:
> **Nested Serializer** — การซ้อน serializer หนึ่งไว้ในอีกอันหนึ่ง ทั้งแบบอ่านอย่างเดียว
> และแบบเขียนได้จริง คุณจะเข้าใจ `PrimaryKeyRelatedField`, `StringRelatedField`,
> `HyperlinkedRelatedField`, `SlugRelatedField`, `depth`, `source=`, และ `extra_kwargs`
> อย่างครบถ้วน แล้วปิดท้ายด้วยการประกอบ `PostSerializer` เวอร์ชันสมบูรณ์ที่มี Category
> ซ้อนแบบเขียนได้, tags แบบ Many-to-Many ผ่าน `through` model, และ comment count —
> ครบทุกเทคนิคที่ใช้งานจริงในระบบ API ระดับมืออาชีพ

---

## สารบัญของ Part นี้

- ขั้นตอนที่ 401: `ModelSerializer` เบื้องต้น — `class Meta`, `fields`/`exclude`, การ generate field อัตโนมัติจาก Model field type
- ขั้นตอนที่ 402: Nested Serializer — แสดง `CategorySerializer` ซ้อนอยู่ใน `PostSerializer` (read-only ก่อน)
- ขั้นตอนที่ 403: Writable Nested Serializer — เขียน `create()`/`update()` เองเพื่อรองรับการสร้าง/แก้ไขข้อมูลซ้อนกัน
- ขั้นตอนที่ 404: `PrimaryKeyRelatedField` vs `StringRelatedField` vs `HyperlinkedRelatedField` — ตารางเปรียบเทียบ
- ขั้นตอนที่ 405: `SlugRelatedField` — อ้างอิงด้วย slug แทน pk
- ขั้นตอนที่ 406: `depth` attribute ใน `Meta` สำหรับ auto-nested representation (ข้อจำกัดของมัน)
- ขั้นตอนที่ 407: จัดการ ManyToMany relation (`Post.tags`) ใน Serializer
- ขั้นตอนที่ 408: `source=` argument สำหรับ remap ชื่อ field ให้ต่างจาก model field
- ขั้นตอนที่ 409: `extra_kwargs` ใน `Meta` สำหรับปรับแต่ง field โดยไม่ต้องประกาศใหม่ทั้งหมด
- ขั้นตอนที่ 410: สรุปและแบบฝึกหัด — สร้าง `PostSerializer` เต็มรูปแบบที่มี nested Category, tags, และ comment count

---

## ขั้นตอนที่ 401: `ModelSerializer` เบื้องต้น — `class Meta`, `fields`/`exclude`, การ generate field อัตโนมัติจาก Model field type

### 401.1 ทบทวนสถานะโปรเจกต์ก่อนเริ่ม Part นี้

ใน Part 039-040 คุณได้ติดตั้ง Django REST Framework และหัดเขียน `serializers.Serializer`
แบบพื้นฐานสำหรับโมเดล `Post` ที่มาจาก Part 012 (ซึ่งมี `category` เป็น `ForeignKey`,
`tags` เป็น `ManyToManyField` ผ่าน `through='PostTag'`, และมี `comments` เป็น reverse
relation จาก `Comment.post`) โค้ดที่คุณน่าจะมีอยู่ตอนนี้หน้าตาประมาณนี้:

```python
# blog/serializers.py (เวอร์ชันจาก Part 040 — เขียนมือทุก field)
from rest_framework import serializers
from .models import Post


class PostSerializer(serializers.Serializer):
    id = serializers.IntegerField(read_only=True)
    title = serializers.CharField(max_length=200)
    slug = serializers.SlugField(max_length=220, read_only=True)
    content = serializers.CharField()
    category = serializers.PrimaryKeyRelatedField(read_only=True)
    is_published = serializers.BooleanField(default=False)
    created_at = serializers.DateTimeField(read_only=True)
    updated_at = serializers.DateTimeField(read_only=True)

    def create(self, validated_data):
        return Post.objects.create(**validated_data)

    def update(self, instance, validated_data):
        for attr, value in validated_data.items():
            setattr(instance, attr, value)
        instance.save()
        return instance
```

โค้ดนี้ **ใช้งานได้จริง** แต่มีปัญหาที่เห็นชัดเจนขึ้นเรื่อย ๆ เมื่อโปรเจกต์โต:

1. **ซ้ำซ้อนกับ `models.py`** — `max_length=200` ของ `title` ถูกกำหนดไว้แล้วใน
   `Post.title = models.CharField(max_length=200)` แต่ต้องมาพิมพ์ซ้ำอีกครั้งใน
   serializer ถ้าวันหนึ่งเปลี่ยน `max_length` ใน model แล้วลืมแก้ serializer
   จะเกิดพฤติกรรมไม่ตรงกันแบบเงียบ ๆ (silent bug)
2. **ต้องเขียน `create()`/`update()` เองทุกครั้ง** ทั้งที่ส่วนใหญ่เป็นโค้ด boilerplate
   แบบเดียวกันเป๊ะ ๆ ในทุก serializer
3. **ลืม field ง่าย** — ถ้า `Post` มี field ใหม่เพิ่มเข้ามา (เช่น `view_count`)
   ต้องไม่ลืมมาเพิ่มใน serializer ด้วยมือทุกครั้ง

Django แก้ปัญหานี้ด้วยหลักการเดียวกับที่คุณเคยเห็นใน Django Admin (Part 017) และ
`ModelForm` (Part 025): **"บอก Django ว่าจะใช้ Model ไหน แล้วให้ Django สร้าง field
ที่เหมาะสมให้อัตโนมัติ"** สำหรับ Serializer คือคลาส **`ModelSerializer`**

### 401.2 `ModelSerializer` คืออะไร

`ModelSerializer` คือ subclass ของ `Serializer` ที่ Django REST Framework เตรียมไว้ให้
โดยมันจะ:

- **Generate field อัตโนมัติ** จาก field ของ Model ที่ระบุ (แปลง `models.CharField`
  เป็น `serializers.CharField`, แปลง `models.ForeignKey` เป็น
  `serializers.PrimaryKeyRelatedField` ฯลฯ)
- **สร้าง validator อัตโนมัติ** ตาม constraint ของ model (เช่น `unique=True`,
  `max_length`)
- **สร้าง `create()` และ `update()` เริ่มต้นให้แล้ว** โดยไม่ต้องเขียนเอง (ยกเว้นกรณี
  พิเศษที่เราจะเรียนใน ขั้นตอนที่ 403)

เขียน `PostSerializer` ใหม่ด้วย `ModelSerializer`:

```python
# blog/serializers.py
from rest_framework import serializers
from .models import Category, Post, Tag


class CategorySerializer(serializers.ModelSerializer):
    class Meta:
        model = Category
        fields = ['id', 'name', 'slug', 'description']


class TagSerializer(serializers.ModelSerializer):
    class Meta:
        model = Tag
        fields = ['id', 'name', 'slug']


class PostSerializer(serializers.ModelSerializer):
    class Meta:
        model = Post
        fields = [
            'id', 'title', 'slug', 'content', 'category', 'tags',
            'is_published', 'created_at', 'updated_at',
        ]
        read_only_fields = ['slug', 'created_at', 'updated_at']
```

โค้ดสั้นลงจาก 20 บรรทัดเหลือไม่ถึง 10 บรรทัด และไม่ต้องเขียน `create()`/`update()`
เองเลย — `ModelSerializer` เตรียมให้ครบแล้ว

### 401.3 `class Meta` — หัวใจของ `ModelSerializer`

เช่นเดียวกับ `Model.Meta` และ `ModelForm.Meta` ที่คุณคุ้นเคยจาก Part 015 และ Part 025
`ModelSerializer.Meta` คือจุดที่บอก DRF ว่าจะทำงานกับ model ไหนและ field อะไรบ้าง:

| Attribute ใน `Meta` | หน้าที่ | บังคับหรือไม่ |
|---|---|---|
| `model` | ระบุ Model class ที่ serializer นี้อ้างอิง | ✅ บังคับเสมอ |
| `fields` | ระบุรายชื่อ field ที่จะรวมเข้ามา (list หรือ `'__all__'`) | ต้องมีอย่างใดอย่างหนึ่งระหว่าง `fields` กับ `exclude` |
| `exclude` | ระบุรายชื่อ field ที่จะ**ไม่**เอาเข้ามา (เอาที่เหลือทั้งหมด) | ใช้แทน `fields` ได้ (แต่ไม่ใช้พร้อมกัน) |
| `read_only_fields` | รายชื่อ field ที่ยังแสดงตอนอ่าน แต่ห้ามเขียนตอน POST/PUT | ไม่บังคับ |
| `depth` | ความลึกของ nested representation อัตโนมัติ (ขั้นตอนที่ 406) | ไม่บังคับ (default `0`) |
| `extra_kwargs` | ปรับแต่ง keyword argument ของ field ที่ generate อัตโนมัติ (ขั้นตอนที่ 409) | ไม่บังคับ |

### 401.4 `fields = [...]` เจาะจงรายชื่อ vs `fields = '__all__'`

```python
class PostSerializer(serializers.ModelSerializer):
    class Meta:
        model = Post
        fields = '__all__'   # เอาทุก field ของ Post มาทั้งหมด
```

`fields = '__all__'` สะดวกตอนต้นแบบ (prototype) แต่ **หลักสูตรนี้ไม่แนะนำให้ใช้ใน
production** เพราะ:

- ถ้า field ใหม่ถูกเพิ่มเข้า model ในอนาคต (เช่น `internal_notes` ที่ไม่ควรออกสู่
  ภายนอก) มันจะโผล่ใน API ทันทีโดยไม่ตั้งใจ — เป็นช่องโหว่ด้าน security ที่พบบ่อยมาก
  ในระบบจริง (เรียกว่า **over-exposure**)
- ทำให้ผู้อ่านโค้ดไม่เห็นภาพชัดว่า API ส่งอะไรออกไปบ้างโดยไม่ต้องไปเปิด `models.py`
  ควบคู่กัน

**กฎของหลักสูตรนี้: ระบุ `fields` เป็น list เจาะจงเสมอในโค้ด production เว้นแต่จะเป็น
โปรเจกต์ทดลองหรือ internal tool ที่ไม่มีความเสี่ยงด้านข้อมูลรั่วไหล**

### 401.5 `exclude = [...]` ทางเลือกตรงข้ามกับ `fields`

```python
class PostSerializer(serializers.ModelSerializer):
    class Meta:
        model = Post
        exclude = ['updated_at']   # เอาทุก field ยกเว้น updated_at
```

`exclude` มีประโยชน์เมื่อ model มี field เยอะมาก และคุณต้องการเอาออกแค่ 1-2 ตัว
แต่มีข้อเสียตรงกันข้ามกับ `'__all__'` เป๊ะ: **field ใหม่ที่เพิ่มเข้า model จะโผล่ใน API
โดยอัตโนมัติเสมอ** ไม่มีการเซ็นเซอร์ล่วงหน้า จึงมีความเสี่ยงแบบเดียวกับ `'__all__'`

> **ข้อสำคัญ**: DRF ไม่อนุญาตให้ประกาศ `fields` และ `exclude` พร้อมกันใน `Meta`
> เดียวกัน — ถ้าใส่ทั้งคู่ Django จะ raise `AssertionError` ทันทีตอนโหลด serializer

### 401.6 การ generate field อัตโนมัติจาก Model field type

นี่คือส่วนสำคัญที่สุดของขั้นตอนนี้: `ModelSerializer` มีตาราง mapping ภายในที่แปลง
Model field type เป็น Serializer field type โดยอัตโนมัติ:

| Model Field | Serializer Field ที่ Generate อัตโนมัติ | หมายเหตุ |
|---|---|---|
| `CharField` | `CharField` | คัดลอก `max_length` มาเป็น validator |
| `TextField` | `CharField` (พร้อม `style={'base_template': 'textarea.html'}`) | ใช้ตอนแสดงผลใน Browsable API |
| `SlugField` | `SlugField` | ตรวจสอบรูปแบบ slug ให้อัตโนมัติ |
| `IntegerField` | `IntegerField` | คัดลอก min/max จาก `validators` ของ model |
| `DecimalField` | `DecimalField` | คัดลอก `max_digits`, `decimal_places` |
| `BooleanField` | `BooleanField` | คัดลอก `default` |
| `DateField` | `DateField` | — |
| `DateTimeField` | `DateTimeField` | ถ้า `auto_now_add=True` หรือ `auto_now=True` จะตั้ง `read_only=True` ให้อัตโนมัติ |
| `EmailField` | `EmailField` | มี validator ตรวจรูปแบบอีเมลติดมาด้วย |
| `URLField` | `URLField` | มี validator ตรวจรูปแบบ URL |
| `ImageField` / `FileField` | `ImageField` / `FileField` | ต้องส่งเป็น `multipart/form-data` |
| `ForeignKey` | `PrimaryKeyRelatedField` | อัตโนมัติใส่ `queryset=<ModelปลายทางRelated>.objects.all()` ถ้าเขียนได้ |
| `OneToOneField` | `PrimaryKeyRelatedField` | เหมือน `ForeignKey` แต่มักตั้ง `read_only=True` เมื่อเป็น reverse |
| `ManyToManyField` | `PrimaryKeyRelatedField(many=True)` | คืนเป็น list ของ pk |
| primary key (`id`) | `IntegerField(read_only=True)` | Django ตั้งเป็น read-only เสมอ เพราะ client ไม่ควรกำหนด pk เอง |

ค่า `unique=True` ที่ level model ก็จะถูกแปลงเป็น `UniqueValidator` อัตโนมัติด้วย
เช่นกัน — นี่คือเหตุผลที่ `Category.slug = models.SlugField(unique=True)` (จาก
Part 012) จะทำให้ `CategorySerializer` validate ความซ้ำของ slug ให้โดยไม่ต้องเขียน
validator เพิ่มเอง

### 401.7 ดู field ที่ Generate จริงด้วยตัวเองผ่าน `repr()`

DRF มีเทคนิคยอดนิยมสำหรับ debug ว่า `ModelSerializer` generate field อะไรออกมาบ้าง:
เปิด `python manage.py shell` แล้ว `print()` serializer เปล่า ๆ ออกมาดู

```bash
python manage.py shell
```

```python
>>> from blog.serializers import PostSerializer
>>> print(repr(PostSerializer()))
PostSerializer():
    id = IntegerField(label='ID', read_only=True)
    title = CharField(max_length=200)
    slug = CharField(max_length=220, read_only=True)
    content = CharField(style={'base_template': 'textarea.html'})
    category = PrimaryKeyRelatedField(allow_null=True, queryset=Category.objects.all(), required=False)
    tags = PrimaryKeyRelatedField(allow_empty=True, many=True, queryset=Tag.objects.all(), required=False)
    is_published = BooleanField(required=False)
    created_at = DateTimeField(read_only=True)
    updated_at = DateTimeField(read_only=True)
```

เทคนิคนี้มีประโยชน์มากในการทำงานจริง เพราะทำให้คุณเห็น **สิ่งที่ DRF ตัดสินใจให้เอง**
ก่อนที่จะไปแก้ไขอะไรเพิ่ม — สังเกตว่า `category` และ `tags` กลายเป็น
`PrimaryKeyRelatedField` โดยอัตโนมัติ ซึ่งเราจะเจาะลึกและปรับแต่งใน ขั้นตอนที่ 402-405

### 401.8 ทดสอบ `PostSerializer` เบื้องต้นใน Shell

```python
>>> from blog.models import Category, Post
>>> from blog.serializers import PostSerializer
>>> category = Category.objects.create(name='เทคโนโลยี')
>>> post = Post.objects.create(
...     title='แนะนำ ModelSerializer',
...     content='เนื้อหาเกี่ยวกับ ModelSerializer ของ DRF',
...     category=category,
...     is_published=True,
... )
>>> serializer = PostSerializer(post)
>>> serializer.data
{'id': 1, 'title': 'แนะนำ ModelSerializer', 'slug': 'แนะนำ-modelserializer',
 'content': 'เนื้อหาเกี่ยวกับ ModelSerializer ของ DRF', 'category': 1, 'tags': [],
 'is_published': True, 'created_at': '2026-01-15T10:30:00Z',
 'updated_at': '2026-01-15T10:30:00Z'}
```

สังเกตว่า `category` คืนค่าเป็นแค่ **เลข pk (`1`)** ไม่ใช่ข้อมูลของ `Category` เต็มรูปแบบ
— ถ้า frontend ต้องการชื่อหมวดหมู่ด้วย จะต้องยิง request แยกไปหา `/api/categories/1/`
อีกครั้ง (เป็นปัญหา N+1 ระดับ HTTP request ที่แย่กว่า N+1 query เสียอีก) นี่คือปัญหาที่
**Nested Serializer** ใน ขั้นตอนที่ 402 จะเข้ามาแก้ไข

---

## ขั้นตอนที่ 402: Nested Serializer — แสดง `CategorySerializer` ซ้อนอยู่ใน `PostSerializer` (read-only ก่อน)

### 402.1 ปัญหาที่ต้องแก้: Client อยากได้ข้อมูล Category เต็มรูปแบบในคำตอบเดียว

ต่อจากขั้นตอนที่แล้ว ทีม frontend (หรือทีม mobile app) มักขอ API แบบนี้เสมอ:

```json
{
    "id": 1,
    "title": "แนะนำ ModelSerializer",
    "category": {
        "id": 1,
        "name": "เทคโนโลยี",
        "slug": "เทคโนโลยี",
        "description": ""
    }
}
```

แทนที่ `category` จะเป็นแค่เลข `1` พวกเขาต้องการ **object เต็มรูปแบบของ Category** ซ้อน
อยู่ข้างในเลย เพื่อไม่ต้องยิง request ที่สองไปดึงชื่อหมวดหมู่ — นี่คือแนวคิดของ
**Nested Serializer**: ใช้ serializer หนึ่งเป็น field ของอีก serializer หนึ่ง

### 402.2 ประกาศ `CategorySerializer` เป็น field ของ `PostSerializer`

```python
# blog/serializers.py
from rest_framework import serializers
from .models import Category, Post, Tag


class CategorySerializer(serializers.ModelSerializer):
    class Meta:
        model = Category
        fields = ['id', 'name', 'slug', 'description']


class PostSerializer(serializers.ModelSerializer):
    category = CategorySerializer(read_only=True)

    class Meta:
        model = Post
        fields = [
            'id', 'title', 'slug', 'content', 'category', 'tags',
            'is_published', 'created_at', 'updated_at',
        ]
        read_only_fields = ['slug', 'created_at', 'updated_at']
```

สิ่งที่เปลี่ยนไปมีจุดเดียว: เราประกาศ `category = CategorySerializer(read_only=True)`
เป็น attribute ของ class **เหนือ** `class Meta` — เมื่อ DRF เจอ field ที่ประกาศไว้แบบนี้
มันจะ **ใช้ตัวที่เราประกาศเอง แทนที่จะ generate `PrimaryKeyRelatedField` ให้อัตโนมัติ**
(field ที่ประกาศ explicit บน class เสมอมีความสำคัญเหนือกว่าสิ่งที่ `ModelSerializer`
generate อัตโนมัติ)

### 402.3 ทดสอบผลลัพธ์ใน Shell

```python
>>> from blog.serializers import PostSerializer
>>> from blog.models import Post
>>> post = Post.objects.select_related('category').get(pk=1)
>>> serializer = PostSerializer(post)
>>> serializer.data
{'id': 1, 'title': 'แนะนำ ModelSerializer', 'slug': 'แนะนำ-modelserializer',
 'content': 'เนื้อหาเกี่ยวกับ ModelSerializer ของ DRF',
 'category': {'id': 1, 'name': 'เทคโนโลยี', 'slug': 'เทคโนโลยี', 'description': ''},
 'tags': [], 'is_published': True,
 'created_at': '2026-01-15T10:30:00Z', 'updated_at': '2026-01-15T10:30:00Z'}
```

ตอนนี้ `category` กลายเป็น **object เต็มรูปแบบ** แล้ว ไม่ใช่แค่เลข pk อีกต่อไป —
นี่คือพลังของ Nested Serializer

### 402.4 Nested Serializer สำหรับ list: `many=True`

หลักการเดียวกันนี้ใช้กับความสัมพันธ์แบบ "มีได้หลายรายการ" ได้เช่นกัน โดยเพิ่ม
`many=True` เข้าไปในตอนประกาศ ลองซ้อน `comments` (reverse relation จาก `Comment.post`
ที่สร้างไว้ใน Part 012 ขั้นตอนที่ 115) เข้าไปใน `PostSerializer`:

```python
# blog/serializers.py
from .models import Category, Comment, Post, Tag


class CommentSerializer(serializers.ModelSerializer):
    class Meta:
        model = Comment
        fields = ['id', 'author', 'content', 'created_at']


class PostSerializer(serializers.ModelSerializer):
    category = CategorySerializer(read_only=True)
    comments = CommentSerializer(many=True, read_only=True)

    class Meta:
        model = Post
        fields = [
            'id', 'title', 'slug', 'content', 'category', 'tags',
            'comments', 'is_published', 'created_at', 'updated_at',
        ]
        read_only_fields = ['slug', 'created_at', 'updated_at']
```

```python
>>> serializer = PostSerializer(post)
>>> serializer.data['comments']
[{'id': 1, 'author': 'สมชาย', 'content': 'บทความดีมากครับ', 'created_at': '2026-01-15T11:00:00Z'},
 {'id': 2, 'author': 'วิภา', 'content': 'ขอบคุณสำหรับข้อมูลครับ', 'created_at': '2026-01-15T11:15:00Z'}]
```

`comments` มาจาก `related_name='comments'` ที่ตั้งไว้บน `Comment.post` ตั้งแต่
Part 012 — DRF มองหา attribute ชื่อ `comments` บน instance ของ `Post` โดยอัตโนมัติ
(เหมือนที่คุณเคยเรียก `post.comments.all()` ใน shell) ตราบใดที่ชื่อ field ใน
serializer ตรงกับชื่อ attribute หรือ `related_name` บน model

### 402.5 ทำไม Nested Serializer เริ่มต้นจึงเป็น `read_only=True` เสมอ

สังเกตว่าทั้ง `category` และ `comments` เราใส่ `read_only=True` กำกับไว้ชัดเจน —
เพราะโดยธรรมชาติ `ModelSerializer.create()`/`update()` เริ่มต้น **ไม่รู้วิธีจัดการ
ข้อมูลซ้อน (nested data)** เลย ถ้าคุณไม่ใส่ `read_only=True` แล้วพยายาม POST ข้อมูลที่
มี `category` เป็น dict เข้ามา DRF จะ raise error ทันที — นี่คือสิ่งที่เราจะแก้ไขใน
ขั้นตอนที่ 403 ซึ่งเป็นจุดที่มือใหม่เกือบทุกคนเคยงงมาแล้ว

### 402.6 ข้อควรระวังเรื่อง Performance: N+1 Query กลับมาแล้ว

Nested Serializer เพิ่มความเสี่ยงเรื่อง **N+1 Query** (ที่คุณเรียนไปแล้วใน Part 012
ขั้นตอนที่ 117) ให้รุนแรงขึ้นไปอีก เพราะทุกครั้งที่ serialize `Post` หนึ่งตัว DRF ต้อง
เข้าถึง `post.category` (ทำให้เกิด query แยก ถ้าไม่ได้ `select_related`) และ
`post.comments.all()` (ทำให้เกิด query แยกอีกอันสำหรับแต่ละ post ถ้าไม่ได้
`prefetch_related`) — เมื่อ serialize `Post` 50 รายการพร้อมกัน (เช่นตอนแสดง list)
โดยไม่ optimize เลย จะเกิด query มากถึง **101 queries** (1 สำหรับ post list, 50
สำหรับ category, 50 สำหรับ comments)

```python
# ❌ อันตราย: ไม่ optimize เลย
posts = Post.objects.all()

# ✅ ถูกต้อง: ใช้ select_related สำหรับ ForeignKey/OneToOne, prefetch_related สำหรับ M2M/reverse FK
posts = Post.objects.select_related('category').prefetch_related('tags', 'comments')
```

**กฎของหลักสูตรนี้: ทุกครั้งที่ serializer มี nested field ต้องตรวจสอบ queryset
ที่ส่งเข้าไปว่ามี `select_related()`/`prefetch_related()` ครบถ้วนเสมอ** เราจะเรียน
วิธีผูก optimization นี้เข้ากับ View โดยอัตโนมัติใน Part 043 (Generic API Views)

---

## ขั้นตอนที่ 403: Writable Nested Serializer — เขียน `create()`/`update()` เองเพื่อรองรับการสร้าง/แก้ไขข้อมูลซ้อนกัน

### 403.1 สิ่งที่เกิดขึ้นถ้าลบ `read_only=True` ออกตรง ๆ

มาดูว่าถ้าเราอยากให้ client ส่งข้อมูล `category` มาพร้อมกับการสร้าง `Post` ในคำขอ
เดียว (ไม่ต้องสร้าง Category แยกต่างหากก่อน) จะเกิดอะไรขึ้นถ้าลบ `read_only=True`
ออกตรง ๆ

```python
class PostSerializer(serializers.ModelSerializer):
    category = CategorySerializer()   # ลบ read_only=True ออก

    class Meta:
        model = Post
        fields = ['id', 'title', 'slug', 'content', 'category', 'is_published']
        read_only_fields = ['slug']
```

```python
>>> data = {
...     'title': 'บทความใหม่',
...     'content': 'เนื้อหา',
...     'category': {'name': 'สุขภาพ', 'slug': 'สุขภาพ', 'description': ''},
...     'is_published': True,
... }
>>> serializer = PostSerializer(data=data)
>>> serializer.is_valid()
True
>>> serializer.save()
Traceback (most recent call last):
    ...
TypeError: Got a `TypeError` when calling `Post.objects.create()`. This may be
because you have a writable nested field on the serializer. If you meant to
have `.create()` be triggered on the related instance, remove the ForeignKey
field from `fields` and set `read_only=True`, or set `category` to be
writable and add a custom `.create()` method on `PostSerializer` that
correctly handles the nested field.
```

`serializer.is_valid()` ผ่านสบาย ๆ เพราะ validation ของแต่ละ field ทำงานถูกต้อง
แต่พอเรียก `.save()` มันไปเรียก `Post.objects.create(**validated_data)` ที่ Django
ORM มาตรฐาน ซึ่ง `validated_data['category']` ตอนนี้เป็น **`OrderedDict`**
(ผลลัพธ์จาก validate ของ `CategorySerializer`) ไม่ใช่ `Category` instance — Django
ORM ไม่รู้จะเอา dict ไปใส่ใน `ForeignKey` column ยังไง จึง raise `TypeError`
ข้อความ error ของ DRF บอกตรง ๆ เลยว่าต้องทำอย่างไร: **เขียน `create()` เอง**

### 403.2 Override `create()` เพื่อรองรับ Nested Category

```python
# blog/serializers.py
from rest_framework import serializers
from .models import Category, Post


class CategorySerializer(serializers.ModelSerializer):
    class Meta:
        model = Category
        fields = ['id', 'name', 'slug', 'description']


class PostSerializer(serializers.ModelSerializer):
    category = CategorySerializer()

    class Meta:
        model = Post
        fields = ['id', 'title', 'slug', 'content', 'category', 'is_published']
        read_only_fields = ['slug']

    def create(self, validated_data):
        category_data = validated_data.pop('category')
        category, _created = Category.objects.get_or_create(
            name=category_data['name'],
            defaults=category_data,
        )
        post = Post.objects.create(category=category, **validated_data)
        return post
```

หลักการสำคัญ 3 ขั้นตอนในการเขียน `create()` แบบ writable nested:

1. **`pop()` ข้อมูล nested ออกจาก `validated_data` ก่อนเสมอ** เพราะ `Post.objects
   .create()` รับแค่ field ของ `Post` เท่านั้น ไม่รับ dict ของ `Category`
2. **จัดการสร้าง/ค้นหา related object เอง** — ในที่นี้ใช้ `get_or_create()` เพื่อไม่
   สร้าง `Category` ซ้ำถ้ามีชื่อเดียวกันอยู่แล้ว (ทางเลือกอื่นคือ `Category.objects
   .create(**category_data)` ตรง ๆ ถ้าต้องการสร้างใหม่เสมอ)
3. **ส่ง object จริงที่ได้กลับมา (ไม่ใช่ dict) เข้าไปใน `Post.objects.create()`**

```python
>>> serializer = PostSerializer(data=data)
>>> serializer.is_valid()
True
>>> post = serializer.save()
>>> post.category.name
'สุขภาพ'
```

### 403.3 Override `update()` เพื่อรองรับการแก้ไข Nested Category

`update()` ซับซ้อนกว่า `create()` เล็กน้อย เพราะต้องรองรับทั้งกรณีที่ client **ส่ง
`category` มาด้วย** และ **ไม่ส่งมา** (เช่นตอนใช้ `PATCH` แก้แค่ `title` อย่างเดียว):

```python
class PostSerializer(serializers.ModelSerializer):
    category = CategorySerializer()

    class Meta:
        model = Post
        fields = ['id', 'title', 'slug', 'content', 'category', 'is_published']
        read_only_fields = ['slug']

    def create(self, validated_data):
        category_data = validated_data.pop('category')
        category, _created = Category.objects.get_or_create(
            name=category_data['name'],
            defaults=category_data,
        )
        return Post.objects.create(category=category, **validated_data)

    def update(self, instance, validated_data):
        category_data = validated_data.pop('category', None)
        if category_data is not None:
            category, _created = Category.objects.get_or_create(
                name=category_data['name'],
                defaults=category_data,
            )
            instance.category = category

        for attr, value in validated_data.items():
            setattr(instance, attr, value)
        instance.save()
        return instance
```

จุดสำคัญคือ `validated_data.pop('category', None)` — ใส่ค่า default เป็น `None`
เพราะถ้า client ทำ `PATCH` (partial update) โดยไม่ส่ง `category` มา
`validated_data` จะไม่มี key `'category'` เลย การเรียก `.pop('category')` แบบไม่มี
default จะ raise `KeyError` ทันที **นี่คือจุดที่มือใหม่งงบ่อยที่สุด**: ลืมใส่ค่า
default ใน `.pop()` ของเมธอด `update()` (ต่างจาก `create()` ที่ field ที่จำเป็นมัก
ถูกบังคับส่งมาอยู่แล้วผ่าน `required=True`)

### 403.4 ห่อด้วย `transaction.atomic` เพื่อความปลอดภัยของข้อมูล

ในการทำงานจริง การสร้าง/แก้ไขข้อมูลหลายตารางพร้อมกัน (เช่น `Category` แล้วต่อด้วย
`Post`) ควรอยู่ใน **database transaction เดียวกัน** เพื่อไม่ให้เกิดสถานะครึ่ง ๆ กลาง ๆ
ถ้าขั้นตอนใดขั้นตอนหนึ่งล้มเหลวกลางคัน (เช่น สร้าง `Category` สำเร็จ แต่สร้าง `Post`
ล้มเหลวเพราะ validation error ที่ระดับ database):

```python
from django.db import transaction


class PostSerializer(serializers.ModelSerializer):
    category = CategorySerializer()

    class Meta:
        model = Post
        fields = ['id', 'title', 'slug', 'content', 'category', 'is_published']
        read_only_fields = ['slug']

    @transaction.atomic
    def create(self, validated_data):
        category_data = validated_data.pop('category')
        category, _created = Category.objects.get_or_create(
            name=category_data['name'],
            defaults=category_data,
        )
        return Post.objects.create(category=category, **validated_data)

    @transaction.atomic
    def update(self, instance, validated_data):
        category_data = validated_data.pop('category', None)
        if category_data is not None:
            category, _created = Category.objects.get_or_create(
                name=category_data['name'],
                defaults=category_data,
            )
            instance.category = category
        for attr, value in validated_data.items():
            setattr(instance, attr, value)
        instance.save()
        return instance
```

`@transaction.atomic` ทำให้ทุกคำสั่งฐานข้อมูลใน method นั้น **สำเร็จทั้งหมดหรือ
ล้มเหลวทั้งหมด (all-or-nothing)** — ถ้าเกิด exception ระหว่างทาง Django จะ
`ROLLBACK` การเปลี่ยนแปลงทั้งหมดที่ทำไปแล้วในเมธอดนั้นโดยอัตโนมัติ (เราจะเรียนเรื่อง
transaction แบบเจาะลึกใน Phase Database Optimization ช่วงหลังของหลักสูตร)

### 403.5 สรุปรูปแบบ (Pattern) ของ Writable Nested Serializer

| ขั้นตอน | ทำอะไร | ทำไมสำคัญ |
|---|---|---|
| 1 | `pop()` nested data ออกจาก `validated_data` | ORM ไม่รู้จักการ create/update ด้วย dict ซ้อน |
| 2 | สร้างหรือค้นหา related object ด้วยตัวเอง | ต้องได้ instance จริง ไม่ใช่ dict |
| 3 | ใน `update()` ต้องใส่ default ให้ `.pop()` เสมอ | รองรับ partial update (`PATCH`) ที่ไม่ส่ง nested field มา |
| 4 | ห่อด้วย `@transaction.atomic` เมื่อแก้หลายตาราง | ป้องกันข้อมูลครึ่ง ๆ กลาง ๆ เมื่อเกิด error กลางคัน |
| 5 | เขียน test ครอบคลุมทั้ง create, update แบบส่ง nested และไม่ส่ง | Nested logic เป็นจุดที่ regression เกิดง่ายที่สุด |

> **ทางเลือกระดับมืออาชีพ**: สำหรับ nested write ที่ซับซ้อนมาก (หลายชั้น, หลาย M2M)
> ทีมงานจำนวนมากเลือกใช้ library สำเร็จรูปอย่าง **`drf-writable-nested`** แทนการเขียน
> `create()`/`update()` เองทุกครั้ง แต่หลักสูตรนี้แนะนำให้เข้าใจการเขียนมือก่อนเสมอ
> เพราะเมื่อ debug ปัญหาจริงในโปรเจกต์ที่ใช้ library เหล่านี้ คุณจะต้องเข้าใจกลไก
> เบื้องหลังอยู่ดี

---

## ขั้นตอนที่ 404: `PrimaryKeyRelatedField` vs `StringRelatedField` vs `HyperlinkedRelatedField` — ตารางเปรียบเทียบ

### 404.1 ทางเลือกอื่นนอกจาก Nested Serializer เต็มรูปแบบ

บางครั้งการซ้อน `CategorySerializer` เต็มรูปแบบเข้าไปใน `PostSerializer`
(ขั้นตอนที่ 402) ก็ **มากเกินความจำเป็น** เช่น ถ้า frontend ต้องการแค่ "ชื่อ" ของ
หมวดหมู่ ไม่ต้องการ `id`, `slug`, `description` ครบทุกตัว DRF มี relational field
สำเร็จรูปหลายแบบให้เลือกใช้แทน โดยไม่ต้องสร้าง serializer แยก

### 404.2 `PrimaryKeyRelatedField` — ค่า Default ที่ Generate อัตโนมัติ

```python
class PostSerializer(serializers.ModelSerializer):
    category = serializers.PrimaryKeyRelatedField(queryset=Category.objects.all())

    class Meta:
        model = Post
        fields = ['id', 'title', 'category']
```

```python
>>> serializer.data['category']
1
```

คืนแค่เลข pk — เร็วที่สุด (ไม่มี query เพิ่มเพื่อดึงข้อมูล related object) และ
**เขียนได้ (writable) โดยไม่ต้องเขียน `create()`/`update()` เอง** เพราะ DRF รู้วิธี
แปลงเลข pk กลับเป็น instance ผ่าน `queryset` ที่ระบุไว้ได้อัตโนมัติ — นี่คือเหตุผลที่
`ForeignKey` ถูก generate เป็น field ชนิดนี้โดย default

### 404.3 `StringRelatedField` — แสดงผลด้วย `__str__()`

```python
class PostSerializer(serializers.ModelSerializer):
    category = serializers.StringRelatedField()

    class Meta:
        model = Post
        fields = ['id', 'title', 'category']
```

```python
>>> serializer.data['category']
'เทคโนโลยี'
```

เรียก `str(category_instance)` ซึ่งก็คือผลลัพธ์จาก `Category.__str__()` ที่เรา
เขียนไว้ตั้งแต่ Part 012 (`return self.name`) — **read-only เท่านั้น** ใช้ไม่ได้เมื่อ
ต้องการเขียนข้อมูลกลับ เหมาะสำหรับ endpoint ที่เน้นการแสดงผล (เช่น หน้ารายงาน หรือ
Admin dashboard ภายใน) มากกว่า API ที่ต้อง create/update

### 404.4 `HyperlinkedRelatedField` — ลิงก์ไปยัง Endpoint อื่น

```python
class PostSerializer(serializers.ModelSerializer):
    category = serializers.HyperlinkedRelatedField(
        view_name='category-detail',
        queryset=Category.objects.all(),
    )

    class Meta:
        model = Post
        fields = ['id', 'title', 'category']
```

```python
>>> serializer.data['category']
'http://api.example.com/categories/1/'
```

คืนเป็น **URL เต็มรูปแบบ** ที่ชี้ไปยัง detail endpoint ของ `Category` แทนที่จะเป็น
เลข pk — สไตล์นี้เรียกว่า **HATEOAS** (Hypermedia as the Engine of Application
State) ซึ่งเป็นแนวคิด REST แบบเข้มข้น (client "เดินตามลิงก์" แทนการประกอบ URL เอง)
`view_name='category-detail'` ต้องตรงกับชื่อ URL pattern ที่สร้างโดย DRF Router
(เรียนเต็มรูปแบบใน Part 042-043) และฟิลด์นี้**ต้องการ `request` อยู่ใน context ของ
serializer เสมอ** เพื่อสร้าง URL แบบเต็ม (absolute URL) ไม่ใช่แค่ path เฉย ๆ:

```python
serializer = PostSerializer(post, context={'request': request})
```

### 404.5 ตารางเปรียบเทียบสรุป

| Field | ผลลัพธ์ที่ได้ | เขียนได้ไหม (Writable) | ต้องการ `queryset` | ต้องการ `request` ใน context | เหมาะกับ |
|---|---|---|---|---|---|
| `PrimaryKeyRelatedField` | เลข pk (`1`) | ✅ ได้ | ✅ ต้องมี (ถ้า writable) | ❌ ไม่ต้อง | API ทั่วไปที่เน้นความเร็ว, mobile app |
| `StringRelatedField` | ข้อความจาก `__str__()` | ❌ Read-only เท่านั้น | ❌ ไม่ต้อง | ❌ ไม่ต้อง | แสดงผลอย่างเดียว, รายงาน, Admin ภายใน |
| `HyperlinkedRelatedField` | URL เต็มรูปแบบ | ✅ ได้ | ✅ ต้องมี (ถ้า writable) | ✅ ต้องมี | REST API สไตล์ HATEOAS, public API ที่เน้น discoverability |
| `SlugRelatedField` (ขั้นตอนที่ 405) | ค่าจาก field ที่ระบุ (เช่น slug) | ✅ ได้ | ✅ ต้องมี (ถ้า writable) | ❌ ไม่ต้อง | URL/ID ที่มนุษย์อ่านง่าย (human-readable) |
| Nested Serializer (ขั้นตอนที่ 402) | Object เต็มรูปแบบ | ⚠️ ต้องเขียน `create()`/`update()` เอง | ไม่เกี่ยวข้องโดยตรง | ไม่บังคับ | ต้องการข้อมูล related แบบเต็ม ลดจำนวน request |

---

## ขั้นตอนที่ 405: `SlugRelatedField` — อ้างอิงด้วย slug แทน pk

### 405.1 ปัญหาของการอ้างอิงด้วยเลข pk

เลข pk อย่าง `1`, `2`, `3` ใช้งานได้ดีในเชิงเทคนิค แต่ **ไม่มีความหมายสำหรับมนุษย์**
และเปิดเผยข้อมูลภายในระบบโดยไม่จำเป็น (เช่น เดา pk ได้ว่ามีข้อมูลกี่แถวในระบบ) หลาย
ทีมเลือกให้ client ส่ง/รับ **slug** แทน pk เพื่อให้ URL และ payload อ่านง่ายขึ้น เช่น
`"category": "เทคโนโลยี"` แทนที่จะเป็น `"category": 1`

### 405.2 ใช้ `SlugRelatedField` แทน `PrimaryKeyRelatedField`

```python
# blog/serializers.py
class PostSerializer(serializers.ModelSerializer):
    category = serializers.SlugRelatedField(
        slug_field='slug',
        queryset=Category.objects.all(),
    )

    class Meta:
        model = Post
        fields = ['id', 'title', 'category']
```

```python
>>> serializer = PostSerializer(post)
>>> serializer.data['category']
'เทคโนโลยี'
```

`slug_field='slug'` บอก DRF ว่าให้ใช้ค่าจาก attribute ชื่อ `slug` ของ `Category`
(ไม่จำเป็นต้องเป็น `SlugField` ของ Django เสมอไป — จะใช้ `slug_field='name'`,
`slug_field='email'` หรือ attribute อื่นที่ **unique** ก็ได้ทั้งนั้น ชื่อ `Slug` ในที่นี้
หมายถึงแนวคิด "human-readable identifier" ไม่ใช่แค่ Django `SlugField` โดยเฉพาะ)

### 405.3 ทดสอบการเขียน (Writable) ผ่าน `SlugRelatedField`

```python
>>> data = {'title': 'บทความทดสอบ', 'category': 'เทคโนโลยี'}
>>> serializer = PostSerializer(data=data)
>>> serializer.is_valid()
True
>>> post = serializer.save()
>>> post.category.name
'เทคโนโลยี'
```

DRF จะไปค้นหา `Category.objects.get(slug='เทคโนโลยี')` จาก `queryset` ที่ระบุไว้
ให้อัตโนมัติ ไม่ต้องเขียน `create()`/`update()` เองเลย เพราะ `SlugRelatedField` ก็
เป็น relational field แบบเดียวกับ `PrimaryKeyRelatedField` เพียงแค่เปลี่ยนวิธีค้นหา
object เท่านั้น

### 405.4 สิ่งที่เกิดขึ้นเมื่อ slug ไม่มีอยู่จริง

```python
>>> data = {'title': 'บทความทดสอบ', 'category': 'ไม่มีหมวดนี้'}
>>> serializer = PostSerializer(data=data)
>>> serializer.is_valid()
False
>>> serializer.errors
{'category': [ErrorDetail(string='Object with slug=ไม่มีหมวดนี้ does not exist.',
 code='does_not_exist')]}
```

DRF validate ให้อัตโนมัติว่า slug ที่ส่งมาต้องมีอยู่จริงใน `queryset` — ถ้าไม่เจอจะ
ได้ validation error ที่มีข้อความชัดเจน อ่านง่ายกว่าการเจอ `Category.DoesNotExist`
แบบ raw exception มาก

### 405.5 ข้อกำหนดสำคัญ: field ที่ใช้เป็น `slug_field` ควรเป็น `unique=True`

เพราะ `SlugRelatedField` ใช้ `.get(**{slug_field: data})` ภายใน ถ้า field นั้นไม่
`unique` และมีข้อมูลซ้ำกันมากกว่า 1 แถว จะเกิด
`MultipleObjectsReturned` ทันที — ในโปรเจกต์นี้ `Category.slug` มี `unique=True`
ติดตัวมาแล้วตั้งแต่ Part 012 จึงปลอดภัย แต่ถ้าคุณจะใช้ `SlugRelatedField` กับ field
อื่นที่ยังไม่ `unique` ต้องไปเพิ่ม constraint ที่ model ก่อนเสมอ

---

## ขั้นตอนที่ 406: `depth` attribute ใน `Meta` สำหรับ auto-nested representation (ข้อจำกัดของมัน)

### 406.1 ทางลัดสำหรับ Nested แบบไม่ต้องเขียน Serializer แยก

DRF มีทางลัดให้ทำ nested representation แบบอัตโนมัติทั้งหมดโดยไม่ต้องประกาศ
`CategorySerializer` หรือ `CommentSerializer` แยกเลย ผ่าน attribute **`depth`** ใน
`Meta`:

```python
class PostSerializer(serializers.ModelSerializer):
    class Meta:
        model = Post
        fields = ['id', 'title', 'category', 'tags']
        depth = 1
```

```python
>>> serializer = PostSerializer(post)
>>> serializer.data
{'id': 1, 'title': 'แนะนำ ModelSerializer',
 'category': {'id': 1, 'name': 'เทคโนโลยี', 'slug': 'เทคโนโลยี', 'description': ''},
 'tags': [{'id': 1, 'name': 'python', 'slug': 'python'},
          {'id': 2, 'name': 'django', 'slug': 'django'}]}
```

เพียงตั้ง `depth = 1` ทุก `ForeignKey` และ `ManyToManyField` ที่อยู่ใน `fields`
ก็จะถูกขยายเป็น object เต็มรูปแบบให้อัตโนมัติทันที — สะดวกมากสำหรับ prototype หรือ
internal tool ที่ไม่ต้องการ control ละเอียด

### 406.2 `depth` มากกว่า 1 ระดับ

```python
class PostSerializer(serializers.ModelSerializer):
    class Meta:
        model = Post
        fields = ['id', 'title', 'comments']
        depth = 2
```

`depth = 2` จะขยายซ้อนลึกลงไปอีกชั้น เช่น ถ้า `Comment` มี `ForeignKey` ไปยัง `User`
ก็จะขยาย `comments[].author` (ถ้าเป็น FK ไปยัง User) เป็น object ของ `User` ด้วย
— ยิ่งค่า `depth` สูง ยิ่งมีความเสี่ยงเรื่อง N+1 query และ payload ขนาดใหญ่เกินจำเป็น
เพิ่มขึ้นเป็นทวีคูณ

### 406.3 ข้อจำกัดของ `depth` ที่ต้องรู้ก่อนใช้ในงานจริง

| ข้อจำกัด | รายละเอียด |
|---|---|
| **Read-only เสมอ** | field ที่ถูกขยายด้วย `depth` จะเป็น read-only โดยอัตโนมัติทั้งหมด ใช้สร้าง/แก้ไขข้อมูลซ้อนไม่ได้เลย (ต้องเขียน nested serializer เองตามขั้นตอนที่ 403 ถ้าต้องการ writable) |
| **ควบคุม field ที่แสดงไม่ได้** | จะได้ field **ทั้งหมด** ของ model ปลายทางเสมอ (ยกเว้น field ที่เป็น relation ต่ออีกชั้นเกิน `depth` ที่กำหนด) ไม่สามารถเลือกเฉพาะบาง field แบบที่ nested serializer ทำได้ |
| **ขัดกับ field ที่ประกาศ explicit** | ถ้า field ไหนถูกประกาศ explicit ไว้แล้ว (เช่น `category = CategorySerializer()`) `depth` จะไม่มีผลกับ field นั้น เพราะ field ที่ประกาศเองมีความสำคัญเหนือกว่าเสมอ |
| **เสี่ยง N+1 สูงมาก** | ยิ่ง field/relation ในโมเดลเยอะ ยิ่งเกิด query อัตโนมัติแบบควบคุมไม่ได้ ยากต่อการ optimize ด้วย `select_related`/`prefetch_related` อย่างแม่นยำ |
| **ไม่เหมาะกับ API สาธารณะ** | เสี่ยง over-exposure สูงมาก เพราะแสดงทุก field ของ related model รวมถึง field ที่อาจไม่ควรเปิดเผย |

### 406.4 คำแนะนำระดับมืออาชีพ: เมื่อไหร่ควรใช้ `depth`

**หลักสูตรนี้แนะนำให้ใช้ `depth` เฉพาะกรณีต่อไปนี้เท่านั้น:**

- Prototype หรือ proof-of-concept ที่ต้องการความเร็วในการพัฒนามากกว่าความละเอียด
- Internal tool หรือ admin API ที่ผู้ใช้ทุกคนเชื่อถือได้ (trusted users)
- Debug endpoint ชั่วคราวที่ไม่ได้ deploy จริง

**สำหรับ API ที่ใช้งานจริงใน production ควรใช้ Nested Serializer ที่เขียนเอง
(ขั้นตอนที่ 402-403) เสมอ** เพราะควบคุม field ที่แสดง, จัดการ writable nested,
และ optimize N+1 query ได้แม่นยำกว่ามาก — นี่คือหนึ่งในความแตกต่างที่ชัดเจนที่สุด
ระหว่างโค้ดของมือใหม่กับมืออาชีพเมื่อดูโค้ด serializer

---

## ขั้นตอนที่ 407: จัดการ ManyToMany relation (`Post.tags`) ใน Serializer

### 407.1 ทบทวน: `Post.tags` ใช้ `through='PostTag'` แบบกำหนดเอง

ย้อนกลับไป Part 012 ขั้นตอนที่ 114 คุณได้เปลี่ยน `Post.tags` จาก M2M อัตโนมัติ
ให้ใช้ **`through='PostTag'`** ที่มี field เพิ่มเติมคือ `added_at` และ `added_by`:

```python
# blog/models.py (ทบทวนจาก Part 012)
class PostTag(models.Model):
    post = models.ForeignKey(Post, on_delete=models.CASCADE)
    tag = models.ForeignKey(Tag, on_delete=models.CASCADE)
    added_at = models.DateTimeField(auto_now_add=True)
    added_by = models.CharField(max_length=100, blank=True)

    class Meta:
        constraints = [
            models.UniqueConstraint(fields=['post', 'tag'], name='unique_post_tag'),
        ]


class Post(models.Model):
    # ... field อื่น ๆ ...
    tags = models.ManyToManyField(Tag, through='PostTag', related_name='posts', blank=True)
```

การมี `through` แบบกำหนดเองส่งผลกระทบโดยตรงต่อ `ModelSerializer` — เมื่อ
`ModelSerializer.create()`/`update()` มาตรฐานเจอ M2M field มันจะพยายามเรียก
`instance.tags.set(value)` ตรง ๆ หลังจากสร้าง instance หลักเสร็จ แต่ตามที่เรียนไว้ใน
Part 012 ขั้นตอนที่ 114.4 **`.set()` แบบไม่ระบุ `through_defaults` จะใช้ไม่ได้เต็ม
รูปแบบกับ M2M ที่มี custom `through` model** — จึงต้อง override `create()`/
`update()` ของ `PostSerializer` เองเพื่อจัดการ `tags` ให้ถูกต้อง

### 407.2 อ่านค่า Tags แบบ Nested (Read-only)

เริ่มจากฝั่งอ่านก่อน — ซ้อน `TagSerializer` แบบ `many=True`:

```python
class TagSerializer(serializers.ModelSerializer):
    class Meta:
        model = Tag
        fields = ['id', 'name', 'slug']


class PostSerializer(serializers.ModelSerializer):
    category = CategorySerializer(read_only=True)
    tags = TagSerializer(many=True, read_only=True)

    class Meta:
        model = Post
        fields = ['id', 'title', 'category', 'tags', 'is_published']
```

```python
>>> serializer.data['tags']
[{'id': 1, 'name': 'python', 'slug': 'python'}, {'id': 2, 'name': 'django', 'slug': 'django'}]
```

### 407.3 เขียน Tags ผ่าน `create()`/`update()` โดยรับเป็น list ของชื่อ (Writable)

วิธีที่ใช้งานสะดวกที่สุดในทางปฏิบัติคือให้ client ส่ง `tags` เป็น **list ของชื่อ
ธรรมดา** (`["python", "django"]`) แล้วให้ serializer จัดการ `get_or_create` แท็ก
และผูกความสัมพันธ์ผ่าน `PostTag` ให้เอง:

```python
from django.db import transaction
from rest_framework import serializers
from .models import Category, Post, PostTag, Tag


class PostSerializer(serializers.ModelSerializer):
    category = CategorySerializer()
    tags = serializers.ListField(
        child=serializers.CharField(max_length=50),
        write_only=True,
        required=False,
    )
    tag_list = TagSerializer(many=True, read_only=True, source='tags')

    class Meta:
        model = Post
        fields = [
            'id', 'title', 'slug', 'content', 'category',
            'tags', 'tag_list', 'is_published',
        ]
        read_only_fields = ['slug']

    @transaction.atomic
    def create(self, validated_data):
        tag_names = validated_data.pop('tags', [])
        category_data = validated_data.pop('category')
        category, _created = Category.objects.get_or_create(
            name=category_data['name'], defaults=category_data,
        )
        post = Post.objects.create(category=category, **validated_data)
        self._sync_tags(post, tag_names)
        return post

    @transaction.atomic
    def update(self, instance, validated_data):
        tag_names = validated_data.pop('tags', None)
        category_data = validated_data.pop('category', None)
        if category_data is not None:
            category, _created = Category.objects.get_or_create(
                name=category_data['name'], defaults=category_data,
            )
            instance.category = category
        for attr, value in validated_data.items():
            setattr(instance, attr, value)
        instance.save()
        if tag_names is not None:
            self._sync_tags(instance, tag_names)
        return instance

    def _sync_tags(self, post, tag_names):
        post.posttag_set.all().delete()
        for name in tag_names:
            tag, _created = Tag.objects.get_or_create(name=name)
            PostTag.objects.create(post=post, tag=tag, added_by='api')
```

จุดสำคัญที่ต้องอธิบาย:

- เราแยก field ออกเป็น **สองตัว**: `tags` (สำหรับรับข้อมูลตอนเขียน, `write_only=True`)
  กับ `tag_list` (สำหรับแสดงข้อมูลตอนอ่าน, `read_only=True`, ใช้ `source='tags'`
  ชี้กลับไปที่ M2M ตัวจริงบน model — เทคนิค `source=` นี้จะอธิบายเจาะลึกใน
  ขั้นตอนที่ 408) การแยกแบบนี้แก้ปัญหาที่ field เดียวไม่สามารถมีทั้งรูปแบบรับ
  (list ของชื่อ) และรูปแบบส่ง (list ของ object) ที่ต่างกันได้ในตัวเดียว
- `_sync_tags()` คือ helper method ที่ลบความสัมพันธ์เดิมทั้งหมดออกจาก `PostTag`
  แล้วสร้างใหม่ทั้งหมดตาม list ที่ส่งมา (แนวคิดเดียวกับ `.set()` แต่ทำเองเพราะ
  ต้องผ่าน `through` model) และระบุ `added_by='api'` ให้ครบตาม field ที่ `PostTag`
  ต้องการ (อ้างอิงจาก Part 012 ขั้นตอนที่ 114.4 เรื่อง `through_defaults`)
- ทั้ง `create()` และ `update()` ห่อด้วย `@transaction.atomic` เพราะมีการเขียน
  หลายตาราง (`Category`, `Post`, `Tag`, `PostTag`) พร้อมกัน

### 407.4 ทางเลือกที่ง่ายกว่า: `SlugRelatedField(many=True)` (เฉพาะกรณีไม่ใช้ custom `through`)

ถ้าโปรเจกต์ของคุณ**ไม่ได้**ใช้ custom `through` model กับ M2M (คือใช้ตารางกลาง
อัตโนมัติแบบ Part 012 ขั้นตอนที่ 113 ธรรมดา) DRF มีทางลัดที่สั้นกว่ามากโดยไม่ต้อง
เขียน `create()`/`update()` เอง:

```python
class PostSerializer(serializers.ModelSerializer):
    tags = serializers.SlugRelatedField(
        slug_field='name',
        queryset=Tag.objects.all(),
        many=True,
    )

    class Meta:
        model = Post
        fields = ['id', 'title', 'tags']
```

```python
>>> data = {'title': 'บทความใหม่', 'tags': ['python', 'django']}
>>> serializer = PostSerializer(data=data)
>>> serializer.is_valid()
True
>>> post = serializer.save()
>>> list(post.tags.values_list('name', flat=True))
['python', 'django']
```

**ข้อจำกัดสำคัญ**: วิธีนี้ต้องการให้ `Tag` แต่ละชื่อ**มีอยู่แล้วในระบบ** (เพราะใช้
`queryset.get(name=...)` ภายใน ไม่ได้สร้างใหม่ให้อัตโนมัติ) ถ้าส่งชื่อแท็กที่ไม่มีอยู่
จะได้ validation error — ต่างจากวิธีในขั้นตอนที่ 407.3 ที่เราเขียน `get_or_create`
เองจึงสร้างแท็กใหม่ได้ทันทีถ้ายังไม่มี และวิธีนี้ยังใช้ไม่ได้เต็มรูปแบบกับ M2M ที่มี
custom `through` model ที่มี field บังคับเพิ่มเติมอยู่ดี (DRF จะพยายามเรียก
`.set()` ตรง ๆ ซึ่งย้อนกลับไปเจอปัญหาเดิมจาก 407.1)

### 407.5 ตารางสรุปวิธีจัดการ M2M ใน Serializer

| วิธี | Writable | รองรับ custom `through` | สร้าง related object ใหม่อัตโนมัติ | ความซับซ้อนของโค้ด |
|---|---|---|---|---|
| `PrimaryKeyRelatedField(many=True)` (default) | ✅ | ❌ ต้อง override `create()`/`update()` | ❌ (ต้องมี pk อยู่แล้ว) | ต่ำ |
| `TagSerializer(many=True, read_only=True)` | ❌ อ่านอย่างเดียว | ไม่เกี่ยวข้อง | ไม่เกี่ยวข้อง | ต่ำ |
| `SlugRelatedField(many=True)` | ✅ | ❌ ไม่รองรับเต็มรูปแบบ | ❌ | ต่ำ-ปานกลาง |
| `ListField(child=CharField())` + `create()`/`update()` เอง | ✅ | ✅ รองรับเต็มรูปแบบ | ✅ (`get_or_create`) | สูง แต่ควบคุมได้เต็มที่ |

---

## ขั้นตอนที่ 408: `source=` argument สำหรับ remap ชื่อ field ให้ต่างจาก model field

### 408.1 `source=` คืออะไร

ทุก field ใน DRF Serializer รับ argument `source=` ได้เสมอ เพื่อบอกว่า **ค่าจริง
ของ field นี้มาจาก attribute ไหนของ instance** — โดย default ถ้าไม่ระบุ `source`
DRF จะใช้**ชื่อ field เดียวกันกับชื่อ attribute** แต่บางครั้งเราต้องการให้ชื่อใน API
ต่างจากชื่อใน model (เช่น เพื่อความสื่อความหมายที่ดีกว่าสำหรับ client)

### 408.2 Remap ชื่อ field ง่าย ๆ

```python
class PostSerializer(serializers.ModelSerializer):
    headline = serializers.CharField(source='title', max_length=200)
    body = serializers.CharField(source='content')

    class Meta:
        model = Post
        fields = ['id', 'headline', 'body']
```

```python
>>> serializer.data
{'id': 1, 'headline': 'แนะนำ ModelSerializer', 'body': 'เนื้อหา...'}
```

`title` ใน model กลายเป็น `headline` ใน API response โดยไม่ต้องแตะ `models.py`
เลย — มีประโยชน์มากเมื่อชื่อ field ภายในระบบ (ที่ตั้งไว้นานแล้ว) ไม่ตรงกับคำศัพท์
ที่ทีม frontend หรือ business ใช้เรียกกัน

### 408.3 `source=` แบบ Dot Notation เพื่อดึงค่าจาก Related Object

`source` รองรับการ "เดินทาง" ข้าม relation ด้วย dot notation ได้เช่นกัน (คล้ายกับ
double underscore ใน ORM query ที่เรียนไปใน Part 012 ขั้นตอนที่ 116 แต่ syntax
ต่างกันเพราะเป็นคนละ layer):

```python
class PostSerializer(serializers.ModelSerializer):
    category_name = serializers.CharField(source='category.name', read_only=True)
    comment_count = serializers.IntegerField(source='comments.count', read_only=True)

    class Meta:
        model = Post
        fields = ['id', 'title', 'category_name', 'comment_count']
```

```python
>>> serializer.data
{'id': 1, 'title': 'แนะนำ ModelSerializer', 'category_name': 'เทคโนโลยี', 'comment_count': 3}
```

`source='category.name'` หมายถึง "ไปที่ `post.category` แล้วอ่าน attribute
`.name`" ส่วน `source='comments.count'` หมายถึง "ไปที่ `post.comments` (ซึ่งเป็น
`RelatedManager`) แล้วเรียก `.count()`" (DRF ฉลาดพอที่จะเรียก method ที่ไม่มี
argument ให้อัตโนมัติถ้า attribute นั้นเป็น callable)

> **ข้อควรระวังเรื่อง Performance**: `source='comments.count'` จะยิง
> `SELECT COUNT(*)` แยกหนึ่งครั้ง**ต่อ** post ที่ serialize ถ้า serialize post
> หลายรายการพร้อมกันโดยไม่ optimize จะเกิด N+1 query ทันที วิธีที่ดีกว่าในการทำงาน
> จริงคือใช้ `annotate(comment_count=Count('comments'))` ที่ระดับ QuerySet (เรียนไป
> แล้วใน Part 012 และจะเจาะลึกอีกครั้งใน Phase Optimization) แล้วอ่านค่าที่
> annotate ไว้ผ่าน `source='comment_count'` แทน — เราจะใช้เทคนิคนี้ใน ขั้นตอนที่ 410

### 408.4 `source='*'` — ส่ง Instance ทั้งตัวเข้า Field

มีกรณีพิเศษที่ต้องการให้ field เข้าถึง **instance ทั้งตัว** ไม่ใช่แค่ attribute
เดียว โดยใช้ `source='*'` ร่วมกับ `SerializerMethodField` หรือ field ที่เขียนเอง
(custom field) — ปกติแล้วสำหรับกรณีทั่วไปเราใช้ `SerializerMethodField` ซึ่งเข้าถึง
instance ทั้งตัวผ่าน parameter `obj` อยู่แล้วโดยไม่ต้องพึ่ง `source='*'`:

```python
class PostSerializer(serializers.ModelSerializer):
    summary = serializers.SerializerMethodField()

    class Meta:
        model = Post
        fields = ['id', 'title', 'summary']

    def get_summary(self, obj):
        return f'{obj.title} ({obj.category.name if obj.category else "ไม่มีหมวดหมู่"})'
```

`SerializerMethodField` เป็น **read-only เสมอ** และเรียก method ชื่อ
`get_<ชื่อ field>` ให้อัตโนมัติ เหมาะสำหรับ field ที่คำนวณจากหลาย attribute
ร่วมกัน หรือมี logic เงื่อนไขที่ `source=` แบบ dot notation ธรรมดาทำไม่ได้

### 408.5 ตารางสรุปการใช้ `source=`

| รูปแบบ | ตัวอย่าง | ผลลัพธ์ |
|---|---|---|
| Remap ชื่อ field ตรง ๆ | `source='title'` | ใช้ค่าจาก `instance.title` |
| Dot notation ข้าม relation | `source='category.name'` | ใช้ค่าจาก `instance.category.name` |
| เรียก method/property ไม่มี argument | `source='comments.count'` | เรียก `instance.comments.count()` |
| ไม่ระบุ `source` เลย | (ค่า default) | ใช้ attribute ที่ชื่อตรงกับชื่อ field |
| `SerializerMethodField` | `get_<field_name>(self, obj)` | เขียน logic เองแบบอิสระ เข้าถึง instance เต็มรูปแบบ |

---

## ขั้นตอนที่ 409: `extra_kwargs` ใน `Meta` สำหรับปรับแต่ง field โดยไม่ต้องประกาศใหม่ทั้งหมด

### 409.1 ปัญหา: อยากปรับแค่ 1 attribute แต่ต้องประกาศ field ใหม่ทั้งตัว

สมมติคุณอยากให้ `is_published` มีค่า default เป็น `False` อย่างชัดเจน (แม้ Django
model จะมี default อยู่แล้ว) และอยากให้ `content` แสดงเป็น textarea ใน Browsable
API — ปกติแล้วต้องประกาศ field ใหม่ทั้งตัว:

```python
class PostSerializer(serializers.ModelSerializer):
    is_published = serializers.BooleanField(default=False)
    content = serializers.CharField(style={'base_template': 'textarea.html'})

    class Meta:
        model = Post
        fields = ['id', 'title', 'content', 'is_published']
```

วิธีนี้ใช้ได้ แต่ทำให้ DRF **หยุด generate field เหล่านี้อัตโนมัติทั้งหมด** — ถ้า
`content` มี validator อื่นที่ generate มาจาก model (เช่น `blank=False` ที่แปลงเป็น
`required=True`) คุณต้องมาระบุซ้ำเองทั้งหมด เสี่ยงต่อการลืมใส่ constraint บางตัว

### 409.2 `extra_kwargs` — ปรับแต่งโดยไม่ต้องประกาศ field ใหม่

```python
class PostSerializer(serializers.ModelSerializer):
    class Meta:
        model = Post
        fields = ['id', 'title', 'slug', 'content', 'is_published']
        extra_kwargs = {
            'slug': {'read_only': True},
            'is_published': {'default': False},
            'content': {'style': {'base_template': 'textarea.html'}},
        }
```

`extra_kwargs` เป็น dict ที่ **key คือชื่อ field** และ **value คือ dict ของ
keyword argument** ที่จะถูก "ผสาน" (merge) เข้ากับ field ที่ `ModelSerializer`
generate อัตโนมัติ — field ยังคง generate ตามปกติทุกอย่าง (รวม validator จาก
model) เพียงแค่เพิ่ม/แก้ไข attribute ที่ระบุใน `extra_kwargs` เท่านั้น

### 409.3 ตัวอย่างการใช้งานที่พบบ่อยในงานจริง

```python
class PostSerializer(serializers.ModelSerializer):
    class Meta:
        model = Post
        fields = ['id', 'title', 'slug', 'content', 'category', 'is_published']
        extra_kwargs = {
            'title': {
                'min_length': 5,
                'error_messages': {
                    'min_length': 'หัวข้อบทความต้องมีอย่างน้อย 5 ตัวอักษร',
                    'required': 'กรุณากรอกหัวข้อบทความ',
                },
            },
            'slug': {'read_only': True},
            'category': {
                'required': False,
                'allow_null': True,
            },
        }
```

- `title` เพิ่ม `min_length=5` และ custom error message ภาษาไทย โดยไม่ต้องประกาศ
  field ใหม่ทั้งตัว
- `slug` ตั้ง `read_only=True` แบบเจาะจงเฉพาะ field เดียว (ทางเลือกเดียวกับใส่ใน
  `read_only_fields` ของ `Meta` — ถ้ามีแค่ `read_only=True` ตัวเดียว แนะนำใช้
  `read_only_fields` เพราะสั้นและอ่านง่ายกว่า `extra_kwargs`)
- `category` ปรับ `required=False` และ `allow_null=True` เพื่อให้สร้าง `Post`
  โดยไม่ระบุหมวดหมู่ได้ (สอดคล้องกับ `on_delete=models.SET_NULL, null=True,
  blank=True` ที่ตั้งไว้บน model ตั้งแต่ Part 012)

### 409.4 `extra_kwargs` vs การประกาศ field ใหม่ทั้งตัว — เลือกใช้เมื่อไหร่

| สถานการณ์ | แนะนำให้ใช้ |
|---|---|
| ปรับแค่ 1-2 attribute (เช่น `read_only`, `required`, `min_length`) | `extra_kwargs` |
| ต้องการเปลี่ยนชนิด field ทั้งหมด (เช่น จาก `PrimaryKeyRelatedField` เป็น nested serializer) | ประกาศ field ใหม่บน class |
| ต้องการ custom validation logic ที่ซับซ้อน | ประกาศ field ใหม่ พร้อมเขียน `validate_<field_name>()` |
| ต้องการ remap ชื่อ field (`source=`) | ประกาศ field ใหม่บน class (ขั้นตอนที่ 408) — `extra_kwargs` **ใช้กับ field ที่ไม่มีอยู่ใน `fields` ไม่ได้** |

> **ข้อจำกัดสำคัญ**: `extra_kwargs` ใช้ได้เฉพาะกับ field ที่ `ModelSerializer`
> generate อัตโนมัติเท่านั้น (คือ field ที่มีอยู่จริงใน `Meta.fields` และตรงกับชื่อ
> field ของ model) ถ้าพยายามใส่ key ที่ไม่ตรงกับ field ใด ๆ ใน `fields` เลย DRF จะ
> raise `AssertionError` ตอน initialize serializer ทันที

---

## ขั้นตอนที่ 410: สรุปและแบบฝึกหัด — สร้าง `PostSerializer` เต็มรูปแบบที่มี nested Category, tags, และ comment count

### 410.1 ประกอบทุกเทคนิคเข้าด้วยกัน: `PostSerializer` เวอร์ชันสมบูรณ์

นี่คือ `blog/serializers.py` เวอร์ชันสมบูรณ์ที่รวมทุกเทคนิคจาก Part นี้เข้าด้วยกัน
— nested `CategorySerializer` แบบเขียนได้, `tags` แบบรับเป็น list ชื่อและจัดการ
`through` model เอง, `comment_count` ที่ใช้ `annotate()` เพื่อเลี่ยง N+1, และ
`extra_kwargs` สำหรับปรับแต่ง validation:

```python
# blog/serializers.py
from django.db import transaction
from rest_framework import serializers

from .models import Category, Comment, Post, PostTag, Tag


class CategorySerializer(serializers.ModelSerializer):
    class Meta:
        model = Category
        fields = ['id', 'name', 'slug', 'description']
        extra_kwargs = {
            'slug': {'read_only': True},
        }


class TagSerializer(serializers.ModelSerializer):
    class Meta:
        model = Tag
        fields = ['id', 'name', 'slug']
        extra_kwargs = {
            'slug': {'read_only': True},
        }


class CommentSerializer(serializers.ModelSerializer):
    class Meta:
        model = Comment
        fields = ['id', 'author', 'content', 'created_at']
        read_only_fields = ['created_at']


class PostSerializer(serializers.ModelSerializer):
    # --- Nested read: แสดงข้อมูลเต็มรูปแบบตอนอ่าน ---
    category = CategorySerializer()
    comments = CommentSerializer(many=True, read_only=True)
    tag_list = TagSerializer(many=True, read_only=True, source='tags')

    # --- Writable input: รับ tags เป็น list ของชื่อธรรมดา ---
    tags = serializers.ListField(
        child=serializers.CharField(max_length=50),
        write_only=True,
        required=False,
    )

    # --- Computed field: ใช้ annotate() จาก View เพื่อเลี่ยง N+1 (ดู 410.2) ---
    comment_count = serializers.IntegerField(read_only=True, default=0)

    class Meta:
        model = Post
        fields = [
            'id', 'title', 'slug', 'content', 'category',
            'tags', 'tag_list', 'comments', 'comment_count',
            'is_published', 'created_at', 'updated_at',
        ]
        read_only_fields = ['slug', 'created_at', 'updated_at']
        extra_kwargs = {
            'title': {
                'min_length': 5,
                'error_messages': {
                    'min_length': 'หัวข้อบทความต้องมีอย่างน้อย 5 ตัวอักษร',
                },
            },
        }

    @transaction.atomic
    def create(self, validated_data):
        tag_names = validated_data.pop('tags', [])
        category_data = validated_data.pop('category')
        category, _created = Category.objects.get_or_create(
            name=category_data['name'],
            defaults=category_data,
        )
        post = Post.objects.create(category=category, **validated_data)
        self._sync_tags(post, tag_names)
        return post

    @transaction.atomic
    def update(self, instance, validated_data):
        tag_names = validated_data.pop('tags', None)
        category_data = validated_data.pop('category', None)
        if category_data is not None:
            category, _created = Category.objects.get_or_create(
                name=category_data['name'],
                defaults=category_data,
            )
            instance.category = category
        for attr, value in validated_data.items():
            setattr(instance, attr, value)
        instance.save()
        if tag_names is not None:
            self._sync_tags(instance, tag_names)
        return instance

    def _sync_tags(self, post, tag_names):
        post.posttag_set.all().delete()
        for name in tag_names:
            tag, _created = Tag.objects.get_or_create(name=name)
            PostTag.objects.create(post=post, tag=tag, added_by='api')
```

และตัวอย่างการใช้ `annotate()` ที่ระดับ View เพื่อเติมค่า `comment_count` โดยไม่
เกิด N+1 query (เชื่อมโยงกับ ขั้นตอนที่ 408.3 และ Part 012 ขั้นตอนที่ 117):

```python
# blog/views.py (ตัวอย่างเบื้องต้น — เจาะลึก APIView เต็มรูปแบบใน Part 042)
from django.db.models import Count
from rest_framework.decorators import api_view
from rest_framework.response import Response

from .models import Post
from .serializers import PostSerializer


@api_view(['GET'])
def post_list(request):
    posts = (
        Post.objects
        .select_related('category')
        .prefetch_related('tags', 'comments')
        .annotate(comment_count=Count('comments'))
        .all()
    )
    serializer = PostSerializer(posts, many=True)
    return Response(serializer.data)
```

`annotate(comment_count=Count('comments'))` คำนวณจำนวนคอมเมนต์ให้ครบทุก post
**ใน query เดียว** (ไม่ใช่ query แยกทีละ post) แล้ว `comment_count` ที่ประกาศไว้ใน
`PostSerializer` จะอ่านค่าที่ annotate ไว้นี้โดยตรง เพราะชื่อ field ตรงกับชื่อ
attribute ที่ Django เติมเข้าไปใน instance หลัง `annotate()` พอดี (ไม่ต้องใช้
`source=` เพิ่มเพราะชื่อตรงกันอยู่แล้ว)

### 410.2 ทดสอบ Serializer เต็มรูปแบบใน Shell

```bash
python manage.py shell
```

```python
>>> from blog.serializers import PostSerializer
>>> data = {
...     'title': 'สอน Nested Serializer แบบเข้าใจง่าย',
...     'content': 'บทความสอน DRF nested serializer',
...     'category': {'name': 'เทคโนโลยี', 'slug': '', 'description': ''},
...     'tags': ['python', 'django', 'drf'],
...     'is_published': True,
... }
>>> serializer = PostSerializer(data=data)
>>> serializer.is_valid()
True
>>> post = serializer.save()
>>> PostSerializer(post).data['tag_list']
[{'id': 1, 'name': 'python', 'slug': 'python'},
 {'id': 2, 'name': 'django', 'slug': 'django'},
 {'id': 3, 'name': 'drf', 'slug': 'drf'}]
>>> post.category.name
'เทคโนโลยี'
```

### 410.3 สรุปสิ่งที่ได้เรียนรู้ใน Part นี้

- ✅ เข้าใจว่า `ModelSerializer` generate field อัตโนมัติจาก Model field type
  พร้อมตาราง mapping ที่ครอบคลุมทุกชนิด field หลัก
- ✅ เข้าใจความแตกต่างระหว่าง `fields = [...]`, `fields = '__all__'`, และ `exclude`
  พร้อมความเสี่ยงด้าน over-exposure ของแต่ละแบบ
- ✅ สร้าง Nested Serializer แบบ read-only ได้ ทั้งแบบ object เดียว
  (`CategorySerializer`) และแบบ list (`CommentSerializer(many=True)`)
- ✅ เขียน Writable Nested Serializer ได้ โดย override `create()`/`update()`
  พร้อมเข้าใจ error message ที่ DRF แจ้งเมื่อลืมทำ
- ✅ เข้าใจความแตกต่างของ `PrimaryKeyRelatedField`, `StringRelatedField`,
  `HyperlinkedRelatedField`, และ `SlugRelatedField` พร้อมรู้ว่าเมื่อไหร่ควรใช้แบบไหน
- ✅ เข้าใจข้อจำกัดของ `depth` และรู้ว่าทำไมโค้ด production ควรหลีกเลี่ยงมัน
- ✅ จัดการ M2M relation (`Post.tags`) ที่มี custom `through` model ผ่าน
  serializer ได้อย่างถูกต้องและปลอดภัย
- ✅ ใช้ `source=` เพื่อ remap ชื่อ field และดึงค่าข้าม relation ได้
- ✅ ใช้ `extra_kwargs` เพื่อปรับแต่ง field ที่ generate อัตโนมัติโดยไม่ต้อง
  ประกาศใหม่ทั้งตัว
- ✅ ประกอบทุกเทคนิคเข้าด้วยกันเป็น `PostSerializer` เวอร์ชันสมบูรณ์ระดับที่ใช้งาน
  จริงได้

### 410.4 Checklist ก่อนไป Part ถัดไป

- [ ] เขียน `CategorySerializer`, `TagSerializer`, `CommentSerializer` ด้วย
  `ModelSerializer` และทดสอบ `repr()` เพื่อดู field ที่ generate อัตโนมัติแล้ว
- [ ] สร้าง `PostSerializer` ที่มี `category` เป็น nested read-only สำเร็จ
- [ ] แก้ `PostSerializer` ให้ `category` เขียนได้ (writable) โดย override
  `create()`/`update()` และทดสอบสร้าง `Post` พร้อม `Category` ใหม่ในคำขอเดียว
- [ ] ทดสอบ `PrimaryKeyRelatedField`, `StringRelatedField`, `SlugRelatedField`
  ทั้งสามแบบกับ field `category` แล้วเปรียบเทียบผลลัพธ์ JSON ด้วยตัวเอง
- [ ] เขียน logic จัดการ `tags` ที่รองรับ custom `through` model (`PostTag`)
  ได้ทั้ง create และ update
- [ ] ใช้ `annotate(comment_count=Count('comments'))` ร่วมกับ serializer field
  `comment_count` แล้วตรวจสอบด้วย Django Debug Toolbar หรือ `connection.queries`
  ว่าไม่มี N+1 query เกิดขึ้น
- [ ] อธิบายให้เพื่อนร่วมทีม (หรือเขียนใส่ `notes.md`) ได้ว่าทำไม `depth` ไม่เหมาะ
  กับ API ที่ใช้งานจริงใน production

### 410.5 แบบฝึกหัดท้ายบท

**แบบฝึกหัดที่ 1**: เพิ่ม field `author_name` ใน `PostSerializer` โดยใช้
`SerializerMethodField` ที่คืนค่าชื่อผู้เขียนจาก `post.category.name` ถ้ามีหมวดหมู่
หรือคืนค่า `"ไม่ระบุหมวดหมู่"` ถ้า `category` เป็น `None` (ทบทวนเทคนิคจาก
ขั้นตอนที่ 408.4)

**แบบฝึกหัดที่ 2**: สร้าง `ProfileSerializer` สำหรับ `accounts.Profile` (จาก
Part 012 ขั้นตอนที่ 112) ที่มี field `username` ซ้อนมาจาก `profile.user.username`
โดยใช้ `source='user.username'` และทำให้เป็น read-only เท่านั้น

**แบบฝึกหัดที่ 3**: แก้ `PostSerializer._sync_tags()` ให้รับ `added_by` จาก
`self.context['request'].user.username` แทนการ hardcode เป็น `'api'` (คำใบ้:
ต้องส่ง `context={'request': request}` เข้าไปตอนสร้าง serializer instance ในฝั่ง
view — เราจะเรียนเรื่อง `context` ของ serializer อย่างเป็นทางการใน Part 042)

**แบบฝึกหัดที่ 4 (ขั้นสูง)**: เขียนเทียบประสิทธิภาพระหว่างการใช้ `depth = 2`
กับการเขียน Nested Serializer เองแบบเต็มรูปแบบ (nested `category`, `tags`, และ
`comments` พร้อมกันทั้งหมด) โดยนับจำนวน query จริงที่เกิดขึ้นด้วย
`django.db.connection.queries` ตอน serialize `Post` 20 รายการพร้อมกัน แล้วสรุปผล
เปรียบเทียบเป็นตาราง

### 410.6 คำถามที่พบบ่อย (FAQ)

**Q: ทำไม `ModelSerializer` ไม่ generate `create()`/`update()` ที่รองรับ nested
data ให้อัตโนมัติเลย ทั้งที่รู้ว่า field ไหนเป็น relation?**
A: เพราะ DRF **ไม่รู้เจตนาทางธุรกิจ (business logic)** ของคุณ เช่น เมื่อเจอ
`category` เป็น dict ใหม่ DRF ไม่รู้ว่าคุณต้องการ "สร้าง Category ใหม่เสมอ"
หรือ "ค้นหา Category ที่มีอยู่แล้วก่อน" (`get_or_create`) หรือ "ห้ามสร้างใหม่
เด็ดขาด ต้องมีอยู่แล้วเท่านั้น" — การตัดสินใจนี้เป็นเรื่องเฉพาะของแต่ละโปรเจกต์
DRF จึงเลือกที่จะ raise error ชัดเจนแทนการเดาเจตนาให้ผิด ๆ

**Q: ควรใช้ Nested Serializer หรือ `PrimaryKeyRelatedField` ธรรมดา?**
A: ขึ้นกับว่า client ต้องการข้อมูล related แบบเต็มรูปแบบในคำตอบเดียวหรือไม่ ถ้า
frontend จะแสดงแค่ ID หรือยิง request แยกไปดึงรายละเอียดอยู่แล้ว ใช้
`PrimaryKeyRelatedField` เพราะเร็วกว่าและโค้ดง่ายกว่ามาก ถ้าต้องการลดจำนวน
round-trip ระหว่าง client-server (เช่น mobile app ที่อยากประหยัด network call)
ใช้ Nested Serializer แต่ต้องระวังเรื่อง N+1 query และขนาด payload ให้ดี

**Q: ทำไมไม่ใช้ `depth = 1` ไปเลย ง่ายกว่าเขียน Nested Serializer เยอะ?**
A: `depth` สะดวกจริงสำหรับ prototype แต่ในงาน production เกือบทุกครั้งคุณต้องการ
ควบคุมว่า field ไหนแสดง field ไหนไม่แสดง (เช่น ไม่อยากให้ `Category.description`
ยาว ๆ ติดมาด้วยทุกครั้งที่ดึง `Post` list) และต้องการให้เขียนข้อมูลซ้อนได้ ซึ่ง
`depth` ทำไม่ได้เลยทั้งสองอย่าง — ในระยะยาว Nested Serializer ที่เขียนเองจะ
maintain ง่ายกว่าและปลอดภัยกว่ามาก

**Q: `SlugRelatedField` กับ `slug_field='slug'` ต่างจากการใช้ Django `SlugField`
อย่างไร?**
A: เป็นคนละเรื่องกันโดยสิ้นเชิง `SlugField` (ตัวพิมพ์ใหญ่ทั้งคำ) เป็น **Model
field** ของ Django ที่เก็บข้อความรูปแบบ slug (เช่น `technology-news`) ในฐานข้อมูล
ส่วน `SlugRelatedField` เป็น **Serializer field** ของ DRF ที่ใช้อ้างอิงไปยัง
related object ผ่าน attribute ใดก็ได้ที่ unique (ไม่จำเป็นต้องเป็น `SlugField`
เสมอไป — จะใช้กับ `slug_field='email'` หรือ `slug_field='name'` ก็ได้ ถ้า field
นั้น unique)

**Q: จำเป็นต้องใช้ `@transaction.atomic` ทุกครั้งที่เขียน `create()`/`update()`
แบบ nested หรือไม่?**
A: จำเป็นเมื่อการดำเนินการกระทบมากกว่า 1 ตาราง (เช่นตัวอย่างใน Part นี้ที่แตะทั้ง
`Category`, `Post`, `Tag`, `PostTag`) เพราะถ้าไม่ห่อ transaction แล้วเกิด error
กลางคัน (เช่น `Post.objects.create()` ล้มเหลวหลังจากสร้าง `Category` ไปแล้ว)
ฐานข้อมูลจะเหลือ `Category` ที่ไม่มี `Post` เชื่อมอยู่ ซึ่งอาจไม่ใช่สถานะที่ถูกต้อง
ตามธุรกิจ ถ้า `create()`/`update()` ของคุณแตะแค่ตารางเดียว ไม่จำเป็นต้องใช้
`@transaction.atomic` เพิ่มก็ได้ (Django wrap คำสั่งเดียวใน transaction ของตัวมัน
เองอยู่แล้วโดย default)

---

## เตรียมตัวสำหรับ Part ถัดไป

**Part 042: API Views: Function-Based และ APIView** จะพา `PostSerializer` เต็ม
รูปแบบที่คุณสร้างไว้ใน Part นี้ไปใช้งานจริงผ่าน View — คุณจะได้เรียนความแตกต่าง
ระหว่าง `@api_view` แบบ function-based กับ `APIView` แบบ class-based, การจัดการ
`request.data`, การคืนค่า `Response` พร้อม HTTP status code ที่ถูกต้อง, การส่ง
`context={'request': request}` เข้า serializer (ที่ค้างไว้จากแบบฝึกหัดที่ 3 ของ
Part นี้), และการจัดการ error response ให้เป็นมาตรฐานเดียวกันทั้งระบบ เตรียม
`PostSerializer`, `CategorySerializer`, และ `TagSerializer` ของคุณให้พร้อม
เพราะเราจะเอามาต่อกับ View จริงในบทถัดไปทันที
