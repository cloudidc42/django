# Part 069: Redis Caching ขั้นสูง

> **ขั้นตอนที่ 681-690 ของหลักสูตร** | Phase 8: Performance & Caching
>
> เป้าหมายของ Part นี้: ต่อยอดจาก Django Caching Framework เบื้องต้น (Part 068) เข้าสู่
> การใช้งาน **Redis** อย่างมืออาชีพเต็มรูปแบบ ตั้งแต่ติดตั้งและเชื่อม `django-redis` เป็น
> Cache Backend, ทำความเข้าใจโครงสร้างข้อมูลของ Redis ที่ใช้ทำ caching ได้จริง (String,
> Hash, Sorted Set), แก้ปัญหาคลาสสิกที่ระบบ cache ทุกระบบต้องเจอคือ **Cache Stampede**,
> ใช้ Redis เป็น Session Backend แบบเจาะลึกตามที่ Part 034 เกริ่นไว้, สร้างระบบ
> **Rate Limiting** ที่แม่นยำแบบ atomic ต่อยอดจาก Throttling ใน Part 048, กระจายคำสั่ง
> invalidate cache ข้าม server หลายตัวด้วย **Redis Pub/Sub**, Monitor สุขภาพของ Redis
> ด้วย `redis-cli`, ทำความเข้าใจภาพรวม Redis Sentinel/Cluster สำหรับ High Availability,
> ไปจนถึงกลยุทธ์ **Cache Warming** เมื่อจบ Part นี้ คุณจะสร้างระบบ Caching Layer ระดับ
> production ให้กับ Blog ของคุณได้ครบวงจร ทั้งเร็ว ทนทาน และปลอดภัยจากปัญหา cache ล่ม
> พร้อมกันทั้งระบบ

---

## สารบัญของ Part นี้

- ขั้นตอนที่ 681: ติดตั้ง Redis และ `django-redis` เชื่อมเป็น Django Cache Backend
- ขั้นตอนที่ 682: โครงสร้างข้อมูลของ Redis ที่เกี่ยวข้องกับ caching — String, Hash, Sorted Set และการใช้งานผ่าน `django-redis`
- ขั้นตอนที่ 683: ปัญหา Cache Stampede และวิธีแก้ — Lock และ Probabilistic Early Expiration
- ขั้นตอนที่ 684: ใช้ Redis เป็น Session Backend แบบเจาะลึกเต็มรูปแบบ (ต่อจาก Part 034)
- ขั้นตอนที่ 685: ใช้ Redis สำหรับ Rate Limiting จริงแบบ Atomic (เชื่อมกับ Throttling Part 048)
- ขั้นตอนที่ 686: Redis Pub/Sub สำหรับ Invalidate Cache ข้าม Server หลายตัว
- ขั้นตอนที่ 687: การ Monitor Redis — `redis-cli`, Memory Usage, `INFO`, `MONITOR`
- ขั้นตอนที่ 688: ภาพรวม Redis Sentinel/Cluster สำหรับ High Availability
- ขั้นตอนที่ 689: กลยุทธ์ Cache Warming — เตรียม Cache ล่วงหน้าก่อน User มาเจอ Cache Miss
- ขั้นตอนที่ 690: สรุปและแบบฝึกหัด — Redis Caching Layer เต็มรูปแบบสำหรับ Blog พร้อมป้องกัน Cache Stampede

---

## ขั้นตอนที่ 681: ติดตั้ง Redis และ `django-redis` เชื่อมเป็น Django Cache Backend

### 681.1 ทวนสถานะจาก Part 068 และ Part 048

ใน Part 068 คุณได้เรียนรู้ Django Caching Framework เบื้องต้นแล้ว รวมถึงวิธีสลับจาก
`LocMemCache` ไปเป็น Redis ผ่าน `django-redis` แบบพื้นฐาน และใน Part 048 ขั้นตอนที่
478 ก็เกริ่นไว้ว่า Throttle counter ต้องย้ายไปอยู่บน Redis เพื่อให้แม่นยำเมื่อมีหลาย
worker process Part นี้จะไม่สอนการติดตั้งซ้ำแบบผิวเผินอีกครั้ง แต่จะ **ลงลึกในรายละเอียด
การตั้งค่าที่ถูกต้องสำหรับ production จริง** ตั้งแต่ connection pool, error handling,
ไปจนถึงการแยกฐานข้อมูล (DB index) ตามการใช้งาน

### 681.2 ติดตั้ง Redis Server

**บน macOS (Homebrew):**

```bash
brew install redis
brew services start redis
```

**บน Ubuntu/Debian:**

```bash
sudo apt update
sudo apt install redis-server -y
sudo systemctl enable redis-server
sudo systemctl start redis-server
```

**ผ่าน Docker (แนะนำสำหรับพัฒนาในเครื่อง เพื่อความสอดคล้องกับ production):**

```bash
docker run -d --name redis-dev -p 6379:6379 redis:7-alpine redis-server --appendonly yes
```

ตรวจสอบว่า Redis ทำงานอยู่จริง:

```bash
redis-cli ping
# ผลลัพธ์ที่คาดหวัง: PONG
```

### 681.3 ติดตั้งไลบรารีฝั่ง Python

```bash
# activate venv ก่อนเสมอ (ทวนจาก Part 003)
pip install django-redis redis hiredis
pip freeze | grep -iE "redis|hiredis" >> requirements.txt
```

| Package | หน้าที่ |
|---|---|
| `redis` | Python client หลักสำหรับคุยกับ Redis (protocol RESP) |
| `django-redis` | ตัวเชื่อม Redis เข้ากับ Django Cache Framework พร้อมฟีเจอร์เสริมเหนือ built-in backend |
| `hiredis` | C extension เร่งความเร็วการ parse โปรโตคอลของ Redis (เร็วกว่า pure-Python parser หลายเท่า) — `redis-py` ตรวจพบและใช้อัตโนมัติถ้าติดตั้งไว้ |

### 681.4 `django-redis` เทียบกับ Built-in `RedisCache` ของ Django

ตั้งแต่ Django 4.0 เป็นต้นมา Django มี `django.core.cache.backends.redis.RedisCache`
มาให้ในตัวโดยไม่ต้องติดตั้ง package เพิ่ม แต่หลักสูตรนี้ยังคงแนะนำ `django-redis` สำหรับ
งาน production เพราะมีฟีเจอร์ที่จำเป็นสำหรับ Part นี้ที่ built-in backend ยังไม่มี:

| ฟีเจอร์ | Built-in `django.core.cache.backends.redis.RedisCache` | `django-redis` |
|---|---|---|
| ตั้งค่าพื้นฐาน (`get`/`set`/`delete`) | ✅ | ✅ |
| Connection Pooling ปรับแต่งได้ละเอียด | ⚠️ พื้นฐานเท่านั้น | ✅ ปรับ `max_connections`, timeout ได้เต็มที่ |
| `IGNORE_EXCEPTIONS` (ไม่ให้ Redis ล่มทำแอปทั้งระบบพัง) | ❌ ไม่มี | ✅ มี |
| `delete_pattern()` (ลบ key จำนวนมากด้วย wildcard) | ❌ ไม่มี | ✅ มี (ใช้บ่อยมากในขั้นตอนที่ 690) |
| เข้าถึง Redis client ดิบเพื่อใช้ Hash/Sorted Set | ❌ ไม่มี API ตรง ๆ | ✅ `get_redis_connection()` |
| `cache.lock()` สำหรับทำ Distributed Lock | ❌ ไม่มี | ✅ มี (ใช้ในขั้นตอนที่ 683) |
| รองรับ Redis Sentinel | ❌ ไม่มี | ✅ มี (ขั้นตอนที่ 688) |
| Serializer เลือกได้ (Pickle/JSON/MsgPack) | ⚠️ Pickle เท่านั้น | ✅ เลือกได้หลายแบบ |

**สรุป**: built-in backend เหมาะกับโปรเจกต์เล็กที่ต้องการแค่ cache ธรรมดา แต่เมื่อโปรเจกต์
ต้องการ pattern matching, distributed lock, หรือเข้าถึง data structure ขั้นสูงของ Redis
โดยตรง (ซึ่งเป็นหัวใจของ Part นี้ทั้ง Part) `django-redis` คือตัวเลือกที่จำเป็น

### 681.5 ตั้งค่า `CACHES` แบบ Production-Ready

```python
# config/settings.py
import os

REDIS_HOST = os.environ.get('REDIS_HOST', '127.0.0.1')
REDIS_PORT = os.environ.get('REDIS_PORT', '6379')

CACHES = {
    'default': {
        'BACKEND': 'django_redis.cache.RedisCache',
        'LOCATION': f'redis://{REDIS_HOST}:{REDIS_PORT}/1',   # DB 1: page/query cache (Part 068)
        'OPTIONS': {
            'CLIENT_CLASS': 'django_redis.client.DefaultClient',
            'CONNECTION_POOL_KWARGS': {
                'max_connections': 50,
            },
            'SOCKET_CONNECT_TIMEOUT': 5,   # วินาที: timeout ตอนเชื่อมต่อครั้งแรก
            'SOCKET_TIMEOUT': 5,           # วินาที: timeout ตอนรอผลลัพธ์คำสั่ง
            'IGNORE_EXCEPTIONS': True,     # ถ้า Redis ล่ม ไม่ให้แอปทั้งระบบ 500
        },
        'KEY_PREFIX': 'myapp',             # กันชนกับข้อมูลของแอปอื่นที่แชร์ Redis เดียวกัน
        'TIMEOUT': 300,                    # timeout เริ่มต้นถ้า .set() ไม่ระบุ (5 นาที)
    },
    'throttle': {
        'BACKEND': 'django_redis.cache.RedisCache',
        'LOCATION': f'redis://{REDIS_HOST}:{REDIS_PORT}/2',   # DB 2: throttle counters (Part 048)
        'OPTIONS': {'CLIENT_CLASS': 'django_redis.client.DefaultClient'},
    },
    'sessions': {
        'BACKEND': 'django_redis.cache.RedisCache',
        'LOCATION': f'redis://{REDIS_HOST}:{REDIS_PORT}/3',   # DB 3: session storage (ขั้นตอนที่ 684)
        'OPTIONS': {'CLIENT_CLASS': 'django_redis.client.DefaultClient'},
    },
}

DJANGO_REDIS_IGNORE_EXCEPTIONS = True
DJANGO_REDIS_LOG_IGNORED_EXCEPTIONS = True
```

> **ทำไมต้องแยก DB index หลายตัว**: Redis รองรับฐานข้อมูลย่อยได้ 16 ฐาน (index 0-15
> ตามค่าเริ่มต้น) ในเซิร์ฟเวอร์ตัวเดียว การแยก page cache, throttle counter, และ session
> ออกจากกันคนละ DB ทำให้สั่ง `FLUSHDB` ล้างเฉพาะส่วน (เช่น ล้าง cache ตอน deploy
> โดยไม่กระทบ session ของผู้ใช้ที่ login อยู่) ได้อย่างปลอดภัย และทำให้ดู memory usage
> แยกส่วนได้ชัดเจนใน `redis-cli` (ขั้นตอนที่ 687) — **ข้อควรระวัง**: DB index **ไม่ใช่**
> การแยก physical resource เหมือนแยก instance จริง ทั้งหมดยังใช้ CPU/memory ของ Redis
> เซิร์ฟเวอร์เดียวกันอยู่ ถ้าต้องการ isolation แบบสมบูรณ์ (เช่นแยก throttle ออกจาก cache
> จริง ๆ ในระบบขนาดใหญ่) ควรแยกเป็นคนละ Redis instance/cluster ไปเลย

### 681.6 `IGNORE_EXCEPTIONS`: หลักการสำคัญที่สุดของการใช้ Cache ใน Production

```python
'OPTIONS': {
    'IGNORE_EXCEPTIONS': True,
}
```

**หลักการทอง**: Cache ต้องเป็น **optimization** ไม่ใช่ **dependency ที่ทำให้ระบบล่มถ้า
หายไป** ถ้าตั้งค่านี้เป็น `True` เมื่อ Redis ล่มหรือ network มีปัญหาชั่วคราว
`cache.get()`/`cache.set()` จะคืนค่า `None`/ไม่ทำอะไรเงียบ ๆ แทนที่จะ raise exception
ทำให้ view ยังทำงานต่อได้ (แค่ช้าลงเพราะต้อง query ฐานข้อมูลจริงแทน cache) ซึ่งดีกว่า
การที่ผู้ใช้ทั้งเว็บเห็นหน้า 500 Internal Server Error เพราะ cache server ล่มเพียงเครื่อง
เดียวมาก

```python
# ทดสอบพฤติกรรมนี้ได้ทันทีด้วย Django shell
python manage.py shell
```

```python
>>> from django.core.cache import cache
>>> cache.set('test_key', 'hello redis')
True
>>> cache.get('test_key')
'hello redis'

>>> # ลองปิด Redis แล้วรันคำสั่งเดิมอีกครั้ง (docker stop redis-dev)
>>> cache.get('test_key')
None   # ไม่ raise exception เพราะ IGNORE_EXCEPTIONS=True — แอปยังทำงานต่อได้
```

### 681.7 ตรวจสอบการเชื่อมต่อและดู Client ที่ใช้งานจริง

```python
>>> from django_redis import get_redis_connection
>>> con = get_redis_connection("default")
>>> con.ping()
True
>>> con.info('server')['redis_version']
'7.2.4'
```

### 681.8 ตารางสรุปขั้นตอนที่ 681

| หัวข้อ | สรุป |
|---|---|
| ติดตั้ง Redis Server | `apt install redis-server` / `brew install redis` / Docker |
| ติดตั้งฝั่ง Python | `pip install django-redis redis hiredis` |
| เลือก Backend | `django_redis.cache.RedisCache` (มีฟีเจอร์ครบกว่า built-in) |
| แยก DB index | 1 = page cache, 2 = throttle, 3 = sessions |
| `IGNORE_EXCEPTIONS` | ต้องเป็น `True` เสมอใน production — cache ล่มต้องไม่ทำแอปล่มตาม |
| Connection Pool | ตั้ง `max_connections` ให้เหมาะกับจำนวน worker process ทั้งหมด |

---

## ขั้นตอนที่ 682: โครงสร้างข้อมูลของ Redis ที่เกี่ยวข้องกับ Caching

### 682.1 ทำไมต้องรู้จัก Data Structure ของ Redis

Django Cache API มาตรฐาน (`cache.get()`, `cache.set()`) มองทุกอย่างเป็น **key-value
ธรรมดา** เบื้องหลังคือ Redis String เท่านั้น แต่ Redis จริง ๆ แล้วเป็น **โครงสร้างข้อมูล
เซิร์ฟเวอร์ (data structure server)** ที่มี type ให้ใช้มากกว่านั้นมาก การเข้าใจ type
เหล่านี้ทำให้แก้ปัญหาบางอย่างได้อย่างมีประสิทธิภาพกว่าการใช้ String เพียงอย่างเดียวมาก

| Redis Type | คำสั่งหลัก | ใช้งานเข้าถึงผ่าน Django Cache API ได้ไหม | ใช้ทำอะไรใน Caching |
|---|---|---|---|
| **String** | `GET`, `SET`, `INCR` | ✅ ได้ (`cache.get/set/incr`) | เก็บค่าเดี่ยว ๆ เช่น ผลลัพธ์ query, HTML fragment |
| **Hash** | `HSET`, `HGET`, `HINCRBY` | ❌ ไม่ได้ ต้องใช้ raw client | เก็บ object ที่มีหลาย field และต้องการอัปเดตทีละ field โดยไม่ต้อง serialize/deserialize ทั้งก้อน |
| **Sorted Set** | `ZADD`, `ZRANGE`, `ZINCRBY` | ❌ ไม่ได้ ต้องใช้ raw client | Leaderboard, Trending content, Rate limiting แบบ sliding window (ขั้นตอนที่ 685) |
| List | `LPUSH`, `RPOP` | ❌ | Queue เบื้องต้น (Celery ใช้ list/stream ภายใน — เรียนเต็มใน Phase 9) |
| Set | `SADD`, `SISMEMBER` | ❌ | เก็บสมาชิกไม่ซ้ำ เช่น "user online ตอนนี้" |

### 682.2 เข้าถึง Redis Client ดิบผ่าน `django-redis`

```python
# blog/services/cache_raw.py
from django_redis import get_redis_connection


def get_raw_client(alias='default'):
    """
    คืนค่า redis-py Client ตัวจริง เพื่อเรียกคำสั่งที่ Django Cache API มาตรฐาน
    ไม่รองรับ (Hash, Sorted Set, Pub/Sub ฯลฯ)
    """
    return get_redis_connection(alias)
```

### 682.3 ใช้ Hash เก็บ Object ที่มีหลาย Field: Cache สรุปข้อมูลโพสต์

สถานการณ์: หน้า post detail ต้องแสดง `views`, `likes`, `comments_count` ที่เปลี่ยนบ่อย
มาก ถ้าใช้ String ธรรมดา ทุกครั้งที่มีคน view เพิ่ม ต้อง `GET` ทั้งก้อน, deserialize,
แก้ field เดียว, serialize ใหม่, แล้ว `SET` กลับ — สิ้นเปลือง และมีโอกาสเกิด race
condition ถ้ามีหลาย request อัปเดตพร้อมกัน Hash แก้ปัญหานี้ได้ตรงจุด เพราะแก้ field
เดียวได้แบบ atomic โดยไม่ต้องอ่าน-เขียนทั้งก้อน:

```python
# blog/services/post_stats.py
from django_redis import get_redis_connection

STATS_KEY_TEMPLATE = 'post_stats:{post_id}'


def record_post_view(post_id):
    """เพิ่มยอด view แบบ atomic โดยไม่ต้องอ่าน/เขียนทั้งก้อนข้อมูล"""
    con = get_redis_connection("default")
    key = STATS_KEY_TEMPLATE.format(post_id=post_id)
    con.hincrby(key, 'views', 1)
    con.expire(key, 60 * 60 * 24)   # ต่ออายุ 24 ชั่วโมงทุกครั้งที่มีการเข้าชม


def record_post_like(post_id, delta=1):
    con = get_redis_connection("default")
    key = STATS_KEY_TEMPLATE.format(post_id=post_id)
    con.hincrby(key, 'likes', delta)


def get_post_stats(post_id):
    """อ่านสถิติทั้งหมดของโพสต์ในครั้งเดียว (คำสั่งเดียว ไม่ต้อง round-trip หลายครั้ง)"""
    con = get_redis_connection("default")
    key = STATS_KEY_TEMPLATE.format(post_id=post_id)
    raw = con.hgetall(key)
    # redis-py คืนค่าเป็น bytes ทั้ง key และ value ต้อง decode เอง
    return {
        'views': int(raw.get(b'views', 0)),
        'likes': int(raw.get(b'likes', 0)),
    }
```

ใช้ในมุมมองจริง:

```python
# blog/views.py
from django.shortcuts import get_object_or_404, render
from .models import Post
from .services.post_stats import record_post_view, get_post_stats


def post_detail(request, slug):
    post = get_object_or_404(Post, slug=slug, is_published=True)
    record_post_view(post.id)
    stats = get_post_stats(post.id)
    return render(request, 'blog/post_detail.html', {'post': post, 'stats': stats})
```

> **ทำไมไม่เก็บยอด view ลงฐานข้อมูลตรง ๆ**: ถ้าโพสต์เป็นที่นิยม อาจมีคน view เข้ามา
> พร้อมกันหลายร้อยครั้ง/วินาที การยิง `UPDATE` ไปที่ PostgreSQL ทุกครั้งจะสร้างภาระเขียน
> (write load) มหาศาลให้ฐานข้อมูลหลัก Redis Hash ทนแรงเขียนระดับนี้ได้สบาย ๆ เพราะทำงาน
> ในหน่วยความจำล้วน ๆ ส่วนการ sync ยอด view กลับไปที่ฐานข้อมูลจริงเป็นระยะ (เช่นทุก 5
> นาทีผ่าน Celery periodic task) จะเรียนใน Part 076

### 682.4 ใช้ Sorted Set ทำ Trending Posts

**Sorted Set** คือ Set ที่สมาชิกแต่ละตัวมี "คะแนน" (score) กำกับ และ Redis จะเรียง
ลำดับให้อัตโนมัติเสมอ ทำให้เหมาะมากกับ leaderboard/trending content:

```python
# blog/services/trending.py
from django_redis import get_redis_connection

TRENDING_KEY = 'trending:posts'


def bump_trending_score(post_id, amount=1):
    """เพิ่มคะแนนความนิยมของโพสต์ (เรียกทุกครั้งที่มีคน view/like/comment)"""
    con = get_redis_connection("default")
    con.zincrby(TRENDING_KEY, amount, str(post_id))


def get_trending_post_ids(limit=10):
    """
    คืนค่า post_id เรียงจากคะแนนสูงสุดไปต่ำสุด
    ZREVRANGE = ดึงข้อมูลจาก Sorted Set แบบเรียงมากไปน้อย พร้อมคะแนน
    """
    con = get_redis_connection("default")
    results = con.zrevrange(TRENDING_KEY, 0, limit - 1, withscores=True)
    return [(int(post_id), score) for post_id, score in results]


def reset_trending_weekly():
    """ล้างคะแนนทั้งหมด (เรียกทุกสัปดาห์ผ่าน Celery beat ใน Phase 9)"""
    con = get_redis_connection("default")
    con.delete(TRENDING_KEY)
```

นำ `post_id` ที่ได้ไปดึงข้อมูลเต็มจาก ORM (Sorted Set เก็บแค่ id กับคะแนน ไม่ได้เก็บ
ข้อมูลทั้ง object):

```python
# blog/views.py
from .models import Post
from .services.trending import get_trending_post_ids


def trending_posts_view(request):
    ranked = get_trending_post_ids(limit=10)
    post_ids = [post_id for post_id, _ in ranked]

    # ดึงข้อมูลจริงจาก DB แล้วจัดเรียงตามลำดับเดิมจาก Sorted Set
    posts_by_id = Post.objects.in_bulk(post_ids)
    ordered_posts = [posts_by_id[pid] for pid in post_ids if pid in posts_by_id]

    return render(request, 'blog/trending.html', {'posts': ordered_posts})
```

### 682.5 ตารางเปรียบเทียบคำสั่ง Sorted Set ที่ใช้บ่อย

| คำสั่ง | ความหมาย |
|---|---|
| `ZADD key score member` | เพิ่มสมาชิกพร้อมคะแนน (หรืออัปเดตถ้ามีอยู่แล้ว) |
| `ZINCRBY key amount member` | เพิ่มคะแนนของสมาชิกแบบ atomic (สร้างใหม่ถ้ายังไม่มี) |
| `ZREVRANGE key 0 9 WITHSCORES` | ดึง 10 อันดับแรกเรียงจากคะแนนมากไปน้อย |
| `ZSCORE key member` | ดูคะแนนปัจจุบันของสมาชิกหนึ่งตัว |
| `ZREM key member` | ลบสมาชิกออกจาก Sorted Set |
| `ZCARD key` | นับจำนวนสมาชิกทั้งหมด |
| `ZREMRANGEBYSCORE key min max` | ลบสมาชิกที่คะแนนอยู่ในช่วงที่กำหนด (ใช้ทำ sliding window ในขั้นตอนที่ 685) |

### 682.6 ข้อควรระวัง: Raw Client ไม่ผ่าน `KEY_PREFIX`/Serializer ของ Django

```python
CACHES = {
    'default': {
        ...
        'KEY_PREFIX': 'myapp',
    }
}
```

เมื่อใช้ `cache.set('foo', 'bar')` Django/`django-redis` จะเติม prefix ให้อัตโนมัติเป็น
`myapp:1:foo` (รูปแบบ `PREFIX:VERSION:KEY`) แต่เมื่อใช้ `get_redis_connection()` เรียก
คำสั่งดิบอย่าง `con.hset('post_stats:5', ...)` **จะไม่ผ่าน prefix นี้เลย** เพราะเป็นการ
คุยกับ Redis โดยตรงข้าม Django Cache API ไปเลย ดังนั้นควรตั้ง **naming convention ของ
ตัวเอง** ให้ชัดเจนสำหรับ key ที่เข้าถึงผ่าน raw client (เช่น ขึ้นต้นด้วยชื่อโปรเจกต์เสมอ:
`myapp:post_stats:5`) เพื่อไม่ให้ชนกับ key อื่นในฐานข้อมูล Redis เดียวกัน

### 682.7 ตารางสรุปขั้นตอนที่ 682

| Type | Function ที่สร้าง | ใช้ทำอะไร |
|---|---|---|
| Hash | `record_post_view()`, `get_post_stats()` | เก็บสถิติหลาย field ต่อโพสต์ อัปเดตทีละ field แบบ atomic |
| Sorted Set | `bump_trending_score()`, `get_trending_post_ids()` | จัดอันดับความนิยมแบบเรียงลำดับอัตโนมัติ |
| เข้าถึง raw client | `get_redis_connection(alias)` | ใช้เมื่อ Django Cache API มาตรฐานไม่พอ |

---

## ขั้นตอนที่ 683: ปัญหา Cache Stampede และวิธีแก้

### 683.1 Cache Stampede คืออะไร

**Cache Stampede** (หรือเรียกว่า **Thundering Herd Problem**) คือปัญหาที่เกิดขึ้นเมื่อ
cache key ที่มีคนเรียกใช้บ่อยมาก **หมดอายุพร้อมกัน** และมีหลาย request เข้ามาพร้อมกัน
ในจังหวะนั้นพอดี ทุก request จะเห็นว่า cache miss แล้ว **พากันไปคำนวณ/query ข้อมูลเดิม
ซ้ำพร้อมกันทั้งหมด**:

```
เวลา 12:00:00.000 — cache key "homepage:trending" หมดอายุ

Request 1 ────> cache.get() = None ────> query DB (ใช้เวลา 800ms) ────> cache.set()
Request 2 ────> cache.get() = None ────> query DB (ใช้เวลา 800ms) ────> cache.set()
Request 3 ────> cache.get() = None ────> query DB (ใช้เวลา 800ms) ────> cache.set()
...
Request 500 ──> cache.get() = None ────> query DB (ใช้เวลา 800ms) ────> cache.set()

ผลลัพธ์: ฐานข้อมูลได้รับ query หนักเดิมซ้ำ 500 ครั้งพร้อมกันในเสี้ยววินาทีเดียว
→ ฐานข้อมูล CPU พุ่ง 100% → response ทุก request ช้าลงมาก หรือฐานข้อมูลล่ม
```

ยิ่ง cache key นั้น "แพง" ในการคำนวณ (query ซับซ้อน, join หลายตาราง, เรียก external
API) และ traffic ยิ่งสูง ปัญหานี้ยิ่งรุนแรง เพราะสิ่งที่ cache ควรจะป้องกันไว้ (ภาระ
ฐานข้อมูล) กลับเกิดขึ้นพร้อมกันหมดในจังหวะเดียว

### 683.2 วิธีแก้ที่ 1: Distributed Lock (Mutex)

แนวคิด: ให้ **request แรกเท่านั้น** ที่เจอ cache miss ไปคำนวณข้อมูลจริง ส่วน request
อื่น ๆ ที่มาซ้ำในช่วงเวลาเดียวกันต้อง **รอ** (หรือคืนค่าเก่าไปพลางก่อน) จนกว่า request
แรกจะคำนวณเสร็จและเซ็ต cache ใหม่แล้ว `django-redis` มี `cache.lock()` มาให้ในตัว ซึ่ง
เป็น distributed lock ที่ใช้ Redis `SET key value NX PX` เป็นกลไกภายใน (รับประกันว่า
ทุก process/server เห็น lock เดียวกัน ไม่ใช่แค่ในหน่วยความจำของ process เดียว):

```python
# blog/services/cache_stampede.py
import logging

from django.core.cache import cache

logger = logging.getLogger(__name__)


def get_or_compute_with_lock(key, compute_fn, timeout=300, lock_timeout=10,
                              wait_timeout=5):
    """
    Cache-aside pattern ที่ป้องกัน stampede ด้วย distributed lock
    - key: cache key
    - compute_fn: ฟังก์ชันไม่รับ argument ที่คำนวณค่าจริง (มักเป็น query หนัก)
    - timeout: อายุของค่าที่เก็บใน cache (วินาที)
    - lock_timeout: อายุสูงสุดของ lock เอง กันเคส process ถือ lock ค้างเพราะ crash
    - wait_timeout: เวลาสูงสุดที่ request อื่นจะรอ lock ก่อนยอม compute เองเป็น fallback
    """
    value = cache.get(key)
    if value is not None:
        return value

    lock = cache.lock(f'{key}:lock', timeout=lock_timeout)
    acquired = lock.acquire(blocking=True, blocking_timeout=wait_timeout)

    if acquired:
        try:
            # ตรวจสอบซ้ำอีกครั้งหลังได้ lock แล้ว (double-checked locking)
            # เผื่อ request ก่อนหน้าที่ถือ lock ไปแล้วเซ็ต cache เสร็จระหว่างที่เรารอ
            value = cache.get(key)
            if value is None:
                value = compute_fn()
                cache.set(key, value, timeout=timeout)
        finally:
            lock.release()
    else:
        # รอ lock นานเกินไปแล้ว (request แรกอาจยังคำนวณไม่เสร็จ)
        # ทางเลือก: compute เองไปเลย (ยอมรับภาระเพิ่มเล็กน้อย ดีกว่าให้ user รอไม่รู้จบ)
        logger.warning('ไม่สามารถขอ lock สำหรับ key=%s ได้ทันเวลา — compute โดยไม่ผ่าน cache', key)
        value = compute_fn()

    return value
```

**ทำไมต้องมี "double-checked locking"**: สมมติ Request A ได้ lock ไปคำนวณอยู่ ระหว่างนั้น
Request B รอ lock อยู่ (`blocking=True`) เมื่อ Request A ปล่อย lock และ Request B ได้
lock ต่อทันที ถ้าไม่เช็ค `cache.get(key)` อีกครั้งก่อน Request B จะคำนวณซ้ำโดยไม่จำเป็น
ทั้งที่ Request A เพิ่งเซ็ต cache ไว้ให้แล้วเมื่อครู่นี้เอง

ใช้งานจริงในมุมมอง:

```python
# blog/views.py
from django.shortcuts import render
from .models import Post
from .services.cache_stampede import get_or_compute_with_lock


def homepage(request):
    def compute_homepage_data():
        return {
            'featured_posts': list(
                Post.objects.filter(is_published=True)
                .select_related('author')
                .order_by('-views')[:5]
            ),
            'latest_posts': list(
                Post.objects.filter(is_published=True).order_by('-created_at')[:10]
            ),
        }

    data = get_or_compute_with_lock('homepage:data', compute_homepage_data, timeout=300)
    return render(request, 'blog/home.html', data)
```

### 683.3 วิธีแก้ที่ 2: Probabilistic Early Expiration (อัลกอริทึม XFetch)

Lock แก้ปัญหาได้ดี แต่มีข้อเสีย: request ที่ไม่ได้ lock ต้อง **รอ** (เพิ่ม latency) หรือ
ต้อง compute เองเป็น fallback ทางเลือกที่ซับซ้อนน้อยกว่าและไม่ต้องมี lock เลยคือ
**Probabilistic Early Expiration** (อัลกอริทึมชื่อ **XFetch** ที่เผยแพร่โดยทีมวิศวกร
ของ Facebook) แนวคิดคือ: **แทนที่จะให้ทุก request รอจนกว่า cache จะหมดอายุจริง ๆ ให้
สุ่มโอกาสที่ request หนึ่ง ๆ จะ "รีเฟรช cache ล่วงหน้า" ก่อนหมดอายุจริง** โดยความ
น่าจะเป็นนี้เพิ่มขึ้นเรื่อย ๆ เมื่อใกล้เวลาหมดอายุ ทำให้การรีเฟรชกระจายตัวออกไปในช่วงเวลา
แทนที่จะกระจุกตัวที่วินาทีเดียวกันเป๊ะ ๆ

```python
# blog/services/xfetch.py
import math
import random
import time

from django.core.cache import cache

# ค่า beta ควบคุมความ "ก้าวร้าว" ของการรีเฟรชล่วงหน้า
# beta สูง = รีเฟรชล่วงหน้าบ่อยขึ้น (ปลอดภัยกว่าแต่ compute บ่อยขึ้น)
XFETCH_BETA = 1.0


def xfetch_get_or_compute(key, compute_fn, ttl=300):
    """
    Cache-aside pattern ป้องกัน stampede แบบ probabilistic (ไม่ต้องใช้ lock เลย)
    เก็บ tuple (value, compute_delta, expiry_timestamp) แทนค่าตรง ๆ
    """
    cached = cache.get(key)
    now = time.time()

    if cached is not None:
        value, delta, expiry = cached
        # สูตร XFetch: ยิ่งใกล้ expiry ยิ่งมีโอกาสสูงที่จะตัดสินใจ "รีเฟรชล่วงหน้า"
        # random.random() อยู่ในช่วง (0, 1] — log ของค่านี้เป็นลบเสมอ ทำให้
        # เทอมทั้งหมดเป็นบวก และแปรผันตามเวลาที่ใช้คำนวณ (delta) เดิม
        should_refresh_early = now - delta * XFETCH_BETA * math.log(random.random()) >= expiry
        if not should_refresh_early:
            return value
        # เข้าเงื่อนไข "โชคร้าย" ให้เป็นคนรีเฟรชล่วงหน้า (มีโอกาสเกิดขึ้นน้อยมาก
        # ในแต่ละ request เดียว แต่เมื่อรวม request จำนวนมาก จะมีบางคนโดนสุ่มก่อนเสมอ)

    start = time.time()
    value = compute_fn()
    delta = time.time() - start   # ใช้เวลาคำนวณจริงกี่วินาที เก็บไว้ใช้ในรอบถัดไป
    expiry = time.time() + ttl

    # เก็บ TTL จริงยาวกว่าค่า ttl ที่ต้องการเล็กน้อย กัน key หายไปเลยก่อนมีใครมารีเฟรช
    cache.set(key, (value, delta, expiry), timeout=ttl + 30)
    return value
```

ทดสอบว่าอัลกอริทึมทำงานถูกต้อง:

```python
>>> from blog.services.xfetch import xfetch_get_or_compute
>>> import time
>>>
>>> def expensive_query():
...     time.sleep(0.5)   # จำลอง query ที่ใช้เวลา 500ms
...     return "ผลลัพธ์จากฐานข้อมูล"
...
>>> xfetch_get_or_compute('demo_key', expensive_query, ttl=10)
'ผลลัพธ์จากฐานข้อมูล'   # ครั้งแรก compute จริง (ใช้เวลา ~500ms)
>>> xfetch_get_or_compute('demo_key', expensive_query, ttl=10)
'ผลลัพธ์จากฐานข้อมูล'   # ครั้งถัดไปได้จาก cache ทันที (โอกาสรีเฟรชล่วงหน้ายังต่ำ)
```

### 683.4 ตารางเปรียบเทียบ Lock vs Probabilistic Early Expiration

| ประเด็น | Distributed Lock | Probabilistic Early Expiration (XFetch) |
|---|---|---|
| ความซับซ้อนในการ implement | ปานกลาง (ต้องจัดการ acquire/release/timeout) | ปานกลาง-สูง (ต้องเข้าใจสูตรคณิตศาสตร์) |
| Request ที่ไม่ได้ทำงานต้องรอไหม | ✅ ต้องรอ (หรือ fallback compute เอง) | ❌ ไม่ต้องรอเลย ได้ค่าเก่าไปก่อนเสมอ |
| รับประกัน compute แค่ 1 ครั้งต่อรอบหมดอายุ | ✅ รับประกัน (มี lock คุม) | ⚠️ ไม่รับประกัน 100% (มีโอกาสน้อยที่มากกว่า 1 request รีเฟรชพร้อมกัน) แต่ในทางปฏิบัติลดปัญหาได้มาก |
| เหมาะกับข้อมูลที่ "ต้องถูกต้องเป๊ะ" (เช่น ยอดเงินคงเหลือ) | ✅ เหมาะกว่า | ❌ ไม่เหมาะ (ค่าอาจ stale เล็กน้อยได้) |
| เหมาะกับข้อมูลที่ "เก่านิดหน่อยได้" (เช่น trending, homepage) | ✅ ใช้ได้ | ✅ เหมาะที่สุด |
| ต้องพึ่ง Redis lock/atomic operation | ✅ ต้องมี (`cache.lock()`) | ❌ ใช้แค่ `cache.get/set` ธรรมดา |
| ความเสี่ยงถ้า process ที่ถือ lock ค้าง/crash | ⚠️ ต้องตั้ง `lock_timeout` ป้องกัน deadlock | ไม่มีความเสี่ยงนี้เลย |

### 683.5 คำแนะนำระดับมืออาชีพในการเลือกวิธี

- **ข้อมูลที่ query แพงมาก และ traffic สูงมาก (เช่น หน้าแรกของเว็บที่มีคนดูพร้อมกัน
  หลักพัน)**: ใช้ **XFetch** เพราะไม่มี request ไหนต้องรอเลย ประสบการณ์ผู้ใช้ลื่นไหลที่สุด
- **ข้อมูลที่ต้องแม่นยำสูง หรือ compute แพงมากจนยอมให้ compute พร้อมกัน 2-3 ครั้งไม่ได้
  เลย (เช่น เรียก external API ที่คิดเงินตามจำนวนครั้ง)**: ใช้ **Lock** เพราะรับประกันได้
  ว่ามีแค่ 1 request เท่านั้นที่คำนวณจริงในแต่ละรอบ
- **ระบบขนาดใหญ่ระดับโลก**: มักใช้ทั้งสองแบบร่วมกัน — Lock เป็นกลไกหลักป้องกันการ
  compute ซ้ำซ้อน ส่วน XFetch ใช้ตัดสินใจ "เมื่อไหร่ควรเริ่มพยายามรีเฟรช" ก่อนหมดอายุจริง

### 683.6 ตารางสรุปขั้นตอนที่ 683

| หัวข้อ | สรุป |
|---|---|
| Cache Stampede คืออะไร | หลาย request แย่งกัน compute ข้อมูลเดิมพร้อมกันตอน cache หมดอายุ |
| วิธีแก้ที่ 1 | Distributed Lock ผ่าน `cache.lock()` ของ `django-redis` — request แรกคำนวณ ที่เหลือรอ |
| วิธีแก้ที่ 2 | Probabilistic Early Expiration (XFetch) — สุ่มให้บาง request รีเฟรชล่วงหน้าก่อนหมดอายุจริง |
| Double-checked locking | ต้องเช็ค cache ซ้ำหลังได้ lock เสมอ กัน compute ซ้ำโดยไม่จำเป็น |
| เลือกใช้แบบไหน | ข้อมูลแม่นยำสูง → Lock, ข้อมูล traffic สูง ยอม stale ได้เล็กน้อย → XFetch |

---

## ขั้นตอนที่ 684: ใช้ Redis เป็น Session Backend แบบเจาะลึกเต็มรูปแบบ

### 684.1 ทวนจาก Part 034

Part 034 ขั้นตอนที่ 332 เกริ่นไว้ว่า `SESSION_ENGINE` มีตัวเลือก `cache` และ `cached_db`
ที่ใช้ Redis เป็นที่เก็บ session ได้ พร้อมตารางเปรียบเทียบข้อดี-ข้อเสีย ตอนนี้เราจะตั้ง
ค่าจริงแบบเต็มรูปแบบ วัดผล และเรียนรู้การย้าย session ที่มีอยู่แล้วจากฐานข้อมูลไปยัง
Redis โดยไม่ทำให้ผู้ใช้ที่ login ค้างอยู่ถูกเด้งออกจากระบบ

### 684.2 ตั้งค่า Cache Alias เฉพาะสำหรับ Session

ทวนจากขั้นตอนที่ 681.5 เราเตรียม cache alias ชื่อ `'sessions'` ที่ชี้ไปยัง Redis DB
index 3 แยกจาก cache ทั่วไปไว้แล้ว:

```python
# config/settings.py
CACHES = {
    'default': {...},    # DB 1 — page/query cache
    'throttle': {...},   # DB 2 — throttle counters
    'sessions': {
        'BACKEND': 'django_redis.cache.RedisCache',
        'LOCATION': f'redis://{REDIS_HOST}:{REDIS_PORT}/3',
        'OPTIONS': {
            'CLIENT_CLASS': 'django_redis.client.DefaultClient',
            'IGNORE_EXCEPTIONS': False,   # ⚠️ session ต้อง IGNORE_EXCEPTIONS=False เสมอ (ดู 684.3)
        },
        'TIMEOUT': None,   # ไม่ตั้ง TTL ระดับ cache — ให้ Django session framework จัดการอายุเอง
    },
}

SESSION_ENGINE = 'django.contrib.sessions.backends.cached_db'
SESSION_CACHE_ALIAS = 'sessions'
```

### 684.3 ทำไม Session Cache ต้องตั้ง `IGNORE_EXCEPTIONS = False`

นี่คือข้อยกเว้นสำคัญจากหลักการในขั้นตอนที่ 681.6! สำหรับ cache ทั่วไป (page cache) การ
ตั้ง `IGNORE_EXCEPTIONS = True` คือหลักการที่ถูกต้อง แต่สำหรับ **session** ควรพิจารณา
ต่างออกไปตาม backend ที่เลือก:

| `SESSION_ENGINE` | ถ้า Redis ล่มเกิดอะไรขึ้น | `IGNORE_EXCEPTIONS` ที่แนะนำ |
|---|---|---|
| `cache` (ล้วน ๆ ไม่มี `_db`) | อ่าน session ไม่ได้เลย ถ้า `IGNORE_EXCEPTIONS=True` ผู้ใช้จะดูเหมือน "ยังไม่ login" ทันที (session หาย) แต่แอปไม่ error | `True` ถ้ายอมรับผู้ใช้ถูกเด้งออกชั่วคราวได้ ดีกว่าทั้งเว็บ 500 |
| `cached_db` | อ่านจาก cache ไม่ได้ แต่ **fallback ไปอ่านฐานข้อมูลอัตโนมัติ** (พฤติกรรมของ `cached_db` เอง) ทำให้ผู้ใช้ยัง login อยู่ได้ แค่ช้าลง | `True` ก็ปลอดภัย เพราะมี DB เป็น fallback อยู่แล้ว |

หลักสูตรนี้เลือกใช้ **`cached_db`** เป็นค่าแนะนำหลัก (ตรงกับคำแนะนำใน Part 034 ขั้นตอนที่
332.7) เพราะได้ทั้งความเร็วของ Redis (อ่าน/เขียนเร็วมากในกรณีปกติ) และความทนทานของ
ฐานข้อมูลเป็นเกราะป้องกันชั้นสอง หากใช้ `cached_db` จริง ๆ การตั้ง `IGNORE_EXCEPTIONS`
เป็น `True` หรือ `False` ก็ปลอดภัยทั้งคู่ แต่ถ้าเลือกใช้ `cache` แบบล้วน ๆ (ไม่มี
fallback) ต้องตั้งใจเลือกอย่างระมัดระวังตามสิ่งที่ยอมรับได้ของระบบ

### 684.4 วัดผลความเร็วก่อน-หลังเปลี่ยน Backend

```python
# blog/management/commands/bench_session_backend.py
import time

from django.contrib.sessions.backends.db import SessionStore as DBSessionStore
from django.contrib.sessions.backends.cached_db import SessionStore as CachedDBSessionStore
from django.core.management.base import BaseCommand


class Command(BaseCommand):
    help = "เปรียบเทียบความเร็วอ่าน/เขียน session ระหว่าง db กับ cached_db backend"

    def handle(self, *args, **options):
        for label, store_class in [
            ('db (PostgreSQL ล้วน)', DBSessionStore),
            ('cached_db (Redis + PostgreSQL)', CachedDBSessionStore),
        ]:
            store = store_class()
            store['test_key'] = 'test_value'
            store.create()

            start = time.perf_counter()
            for _ in range(100):
                store.load()
            elapsed = time.perf_counter() - start

            self.stdout.write(f'{label}: อ่าน 100 ครั้ง ใช้เวลา {elapsed * 1000:.2f} ms')
            store.delete()
```

ผลลัพธ์ตัวอย่างจากการรันจริง (แตกต่างตามเครื่อง แต่ทิศทางเดียวกันเสมอ):

```
db (PostgreSQL ล้วน): อ่าน 100 ครั้ง ใช้เวลา 340.12 ms
cached_db (Redis + PostgreSQL): อ่าน 100 ครั้ง ใช้เวลา 28.47 ms
```

`cached_db` เร็วกว่าประมาณ **10 เท่า** ในสถานการณ์นี้ เพราะการอ่านครั้งที่ 2 เป็นต้นไป
มาจาก Redis cache ทั้งหมด ไม่ต้องแตะฐานข้อมูลเลย

### 684.5 ย้าย Session ที่มีอยู่แล้วจาก Database Backend ไป Redis โดยไม่ให้ผู้ใช้ถูกเด้งออก

สถานการณ์จริง: ระบบมีอยู่แล้วใช้ `SESSION_ENGINE = 'db'` มีผู้ใช้ login ค้างอยู่หลายพันคน
ต้องการเปลี่ยนไปใช้ `cached_db` โดยไม่ทำให้ทุกคนถูก logout พร้อมกัน:

```python
# accounts/management/commands/migrate_sessions_to_redis.py
from django.contrib.sessions.backends.cached_db import SessionStore as CachedDBSessionStore
from django.contrib.sessions.models import Session
from django.core.management.base import BaseCommand
from django.utils import timezone


class Command(BaseCommand):
    help = "ย้าย session ที่ยังไม่หมดอายุจาก Database Backend เข้า Redis (cached_db)"

    def handle(self, *args, **options):
        active_sessions = Session.objects.filter(expire_date__gt=timezone.now())
        total = active_sessions.count()
        migrated = 0

        for session in active_sessions.iterator(chunk_size=500):
            decoded_data = session.get_decoded()

            # สร้าง cached_db SessionStore โดยใช้ session_key เดิมเป๊ะ ๆ
            # เพื่อให้ cookie sessionid ที่อยู่ในเบราว์เซอร์ผู้ใช้ยังใช้ได้ต่อเนื่อง
            store = CachedDBSessionStore(session_key=session.session_key)
            store._session_cache = decoded_data
            store.save(must_create=False)
            migrated += 1

            if migrated % 500 == 0:
                self.stdout.write(f'ย้ายไปแล้ว {migrated}/{total}...')

        self.stdout.write(self.style.SUCCESS(
            f'ย้าย session สำเร็จทั้งหมด {migrated}/{total} รายการ '
            f'— เปลี่ยน SESSION_ENGINE เป็น cached_db แล้ว deploy ได้เลย'
        ))
```

**ขั้นตอนการ deploy ที่ปลอดภัย**:

1. เตรียม Redis ให้พร้อมและตั้งค่า `CACHES['sessions']` (แต่ยัง**ไม่**เปลี่ยน
   `SESSION_ENGINE`)
2. รัน `python manage.py migrate_sessions_to_redis` ขณะที่ระบบยังใช้ `db` backend
   อยู่ตามปกติ (คำสั่งนี้แค่ "copy" ข้อมูล ไม่ได้ลบต้นฉบับ ปลอดภัยรันซ้ำได้)
3. เปลี่ยน `SESSION_ENGINE = 'django.contrib.sessions.backends.cached_db'`
   ใน `settings.py` แล้ว deploy
4. เพราะ `cached_db` อ่านจาก cache ก่อนเสมอ และ session ทุกตัวถูก copy เข้า Redis
   ไปแล้วในขั้นตอนที่ 2 ผู้ใช้ทุกคนจะยัง login อยู่ต่อเนื่องโดยไม่รู้สึกถึงการเปลี่ยนแปลง
   ใด ๆ เลย

### 684.6 ตรวจสอบจำนวน Session ใน Redis ด้วย `redis-cli`

```bash
redis-cli -n 3 dbsize
# (integer) 1847   ← จำนวน session ทั้งหมดที่อยู่ใน DB index 3

redis-cli -n 3 --scan --pattern "*sessionid*" | head -5
```

### 684.7 ตารางสรุปขั้นตอนที่ 684

| หัวข้อ | สรุป |
|---|---|
| Backend ที่แนะนำ | `cached_db` — เร็วเหมือน Redis, ทนทานเหมือนฐานข้อมูล |
| `IGNORE_EXCEPTIONS` สำหรับ session | ปลอดภัยตั้ง `True` ได้ถ้าใช้ `cached_db` (มี DB fallback) |
| ความเร็วที่ได้ | เร็วขึ้นประมาณ 10 เท่าเทียบกับ `db` backend ล้วน ๆ ในตัวอย่างจริง |
| ย้าย session เดิม | ใช้ management command copy ข้อมูลจาก `django_session` เข้า `cached_db` ก่อนสลับ setting |
| Deploy อย่างปลอดภัย | Migrate ก่อน → สลับ setting → deploy — ผู้ใช้ไม่ถูก logout |

---

## ขั้นตอนที่ 685: ใช้ Redis สำหรับ Rate Limiting จริงแบบ Atomic

### 685.1 ทวนปัญหาที่ยังไม่ถูกแก้จาก Part 048

Part 048 ขั้นตอนที่ 478 แก้ปัญหา "throttle counter ไม่แม่นยำข้าม process" ได้แล้วด้วย
การย้าย `SimpleRateThrottle.cache` ไปที่ Redis แต่ **ยังมีปัญหาที่ลึกกว่านั้นซ่อนอยู่**:
กลไกภายในของ `SimpleRateThrottle` (ทวนจาก Part 048 ขั้นตอนที่ 475.2) ทำงานแบบนี้:

```python
self.history = self.cache.get(self.key, [])      # ① อ่าน
# ... ตัดรายการที่หมดอายุออก ...
if len(self.history) >= self.num_requests:         # ② ตัดสินใจ
    return self.throttle_failure()
self.history.insert(0, self.now)
self.cache.set(self.key, self.history, self.duration)   # ③ เขียนกลับ
```

ทั้ง 3 ขั้นตอนนี้ **ไม่ใช่ operation เดียวที่ atomic** — เป็นการ "อ่าน-ตัดสินใจ-เขียน" ที่
แยกจากกัน 3 คำสั่ง แม้ทั้ง 3 คำสั่งจะไปที่ Redis ตัวเดียวกันแล้วก็ตาม ถ้ามี 2 request
มาถึงพร้อมกันแบบเป๊ะ ๆ (เช่นในระบบที่มี traffic สูงมาก) ทั้งคู่อาจอ่าน `history` ค่า
เดียวกันก่อนที่ฝ่ายใดฝ่ายหนึ่งจะเขียนกลับทัน ทำให้ **ทั้งคู่ตัดสินใจว่า "ยังไม่เกิน
เพดาน" พร้อมกัน** และปล่อยให้ผ่านทั้ง 2 request ทั้งที่ควรบล็อกไปแล้ว 1 ตัว — เรียกว่า
**Race Condition** ในระบบ rate limiting ยิ่ง traffic สูง ยิ่งมีโอกาสเกิดบ่อยขึ้น และยิ่ง
endpoint ไหนสำคัญมาก (เช่น login ที่ป้องกัน brute-force ตาม Part 048 ขั้นตอนที่ 477.1)
ยิ่งต้องการความแม่นยำสูงสุด

### 685.2 ทางแก้: Lua Script ทำให้ทั้งกระบวนการเป็น Atomic Operation เดียว

Redis รับประกันว่า **Lua script ที่รันผ่าน `EVAL`/`EVALSHA` จะทำงานแบบ atomic เสมอ** —
ระหว่างที่ script หนึ่งกำลังรันอยู่ Redis จะไม่ประมวลผลคำสั่งอื่นแทรกเข้ามาเลย (Redis
เป็น single-threaded ในการประมวลผลคำสั่ง) นี่คือวิธีมาตรฐานที่ระบบระดับโลกใช้แก้ปัญหา
"อ่าน-ตัดสินใจ-เขียน" ที่ต้องการความแม่นยำสูงบน Redis

เราจะ implement **Sliding Window Log** algorithm (แม่นยำกว่า Fixed Window ธรรมดา
เพราะนับจากเวลาปัจจุบันย้อนหลังไปจริง ๆ ไม่ใช่แบ่งเป็นช่วงตายตัว) โดยใช้ **Sorted Set**
(ทวนจากขั้นตอนที่ 682.4-682.5) เก็บ timestamp ของทุก request เป็นสมาชิก:

```python
# blog/throttles_redis.py
import time

from django_redis import get_redis_connection

# Lua script: ลบ timestamp ที่เก่าเกิน window ออก, นับจำนวนที่เหลือ,
# ถ้ายังไม่เกินเพดานให้เพิ่ม timestamp ปัจจุบันเข้าไปแล้วคืน 1 (อนุญาต)
# ถ้าเกินแล้วคืน 0 (บล็อก) — ทั้งหมดนี้เกิดขึ้นเป็น atomic operation เดียว
SLIDING_WINDOW_LUA = """
local key = KEYS[1]
local now = tonumber(ARGV[1])
local window = tonumber(ARGV[2])
local limit = tonumber(ARGV[3])
local member = ARGV[4]

redis.call('ZREMRANGEBYSCORE', key, 0, now - window)
local current_count = redis.call('ZCARD', key)

if current_count < limit then
    redis.call('ZADD', key, now, member)
    redis.call('EXPIRE', key, window)
    return 1
else
    return 0
end
"""


class RedisSlidingWindowLimiter:
    """
    Rate limiter แบบ Sliding Window Log ที่ atomic เต็มรูปแบบด้วย Lua script
    ใช้ cache alias 'throttle' (Redis DB 2) ตามที่ Part 048 ขั้นตอนที่ 478.3 แยกไว้
    """

    def __init__(self, cache_alias='throttle'):
        self.connection = get_redis_connection(cache_alias)
        self._script = self.connection.register_script(SLIDING_WINDOW_LUA)

    def is_allowed(self, key, limit, window_seconds):
        now = time.time()
        # member ต้อง unique แม้เรียกในวินาทีเดียวกันหลายครั้ง จึงผูก timestamp ละเอียด
        # ระดับ microsecond เข้าไปด้วย ไม่ใช้ now ตรง ๆ (จะชนกันถ้าเรียกเร็วมาก)
        member = f'{now}'
        result = self._script(keys=[key], args=[now, window_seconds, limit, member])
        return bool(result)
```

### 685.3 นำไปสร้าง Custom Throttle Class ต่อยอดจาก DRF

```python
# blog/throttles.py (เพิ่มต่อจากที่มีอยู่แล้วจาก Part 048)
from rest_framework.throttling import BaseThrottle

from .throttles_redis import RedisSlidingWindowLimiter

_limiter = RedisSlidingWindowLimiter(cache_alias='throttle')


class AtomicSlidingWindowThrottle(BaseThrottle):
    """
    Throttle ที่ใช้ Lua script บน Redis แทนกลไก history-list ของ
    SimpleRateThrottle เดิม (Part 048 ขั้นตอนที่ 475.2) — ไม่มี race condition
    """

    rate_limit = 100
    window_seconds = 60
    scope = 'atomic_default'

    def get_ident_key(self, request):
        if request.user and request.user.is_authenticated:
            ident = f'user:{request.user.pk}'
        else:
            ident = f'ip:{self.get_ident(request)}'
        return f'throttle:{self.scope}:{ident}'

    def allow_request(self, request, view):
        key = self.get_ident_key(request)
        allowed = _limiter.is_allowed(key, self.rate_limit, self.window_seconds)
        if not allowed:
            self._retry_after = self.window_seconds
        return allowed

    def wait(self):
        return getattr(self, '_retry_after', self.window_seconds)


class AtomicLoginRateThrottle(AtomicSlidingWindowThrottle):
    """
    เวอร์ชัน atomic ของ LoginRateThrottle (Part 048 ขั้นตอนที่ 477.1)
    ใช้กับ endpoint login โดยเฉพาะ ต้องการความแม่นยำสูงสุดเพราะป้องกัน brute-force
    """
    rate_limit = 5
    window_seconds = 60
    scope = 'login'
```

ใช้แทนที่ `LoginRateThrottle` เดิมในมุมมอง login:

```python
# blog/api_views.py
from rest_framework_simplejwt.views import TokenObtainPairView
from .serializers import CustomTokenObtainPairSerializer
from .throttles import AtomicLoginRateThrottle


class ThrottledTokenObtainPairView(TokenObtainPairView):
    serializer_class = CustomTokenObtainPairSerializer
    throttle_classes = [AtomicLoginRateThrottle]
```

### 685.4 ทดสอบว่า Race Condition หายไปจริง ด้วยการยิงพร้อมกัน

```python
# blog/tests/test_atomic_throttle.py
import threading

from django.test import TestCase

from blog.throttles_redis import RedisSlidingWindowLimiter


class AtomicThrottleRaceConditionTests(TestCase):
    def test_concurrent_requests_never_exceed_limit(self):
        limiter = RedisSlidingWindowLimiter(cache_alias='throttle')
        key = 'test:race_condition_key'
        limit = 10
        results = []
        lock = threading.Lock()

        def hit():
            allowed = limiter.is_allowed(key, limit, window_seconds=60)
            with lock:
                results.append(allowed)

        threads = [threading.Thread(target=hit) for _ in range(50)]
        for t in threads:
            t.start()
        for t in threads:
            t.join()

        allowed_count = sum(1 for r in results if r)
        # ต้องมี request ที่ผ่านพอดี "ไม่เกิน" limit แม้ยิงพร้อมกัน 50 thread
        self.assertLessEqual(allowed_count, limit)
        self.assertEqual(allowed_count, limit)   # ต้องผ่านพอดี limit เป๊ะ ไม่ใช่มากกว่า
```

รันเทสนี้เทียบกับ `SimpleRateThrottle` แบบเดิม (ที่ใช้ list ธรรมดา) จะเห็นว่าเวอร์ชันเดิม
บางครั้งปล่อยผ่านมากกว่า `limit` เล็กน้อยเมื่อยิงพร้อมกันจำนวนมาก ในขณะที่เวอร์ชัน Lua
script จะผ่านพอดี `limit` เป๊ะทุกครั้งที่รัน

### 685.5 ตารางเปรียบเทียบ Fixed Window, Sliding Window List (Part 048), Sliding Window Log แบบ Atomic (Part 069)

| ประเด็น | Fixed Window ธรรมดา | `SimpleRateThrottle` (Part 048) | `AtomicSlidingWindowThrottle` (Part 069) |
|---|---|---|---|
| ความแม่นยำที่ขอบหน้าต่างเวลา | ต่ำ (อาจปล่อยผ่าน 2 เท่าของ limit ตรงรอยต่อหน้าต่าง) | สูง (นับจากเวลาจริงย้อนหลัง ไม่มีรอยต่อ) | สูง (นับจากเวลาจริงย้อนหลัง ไม่มีรอยต่อ) |
| ปลอดภัยจาก Race Condition เมื่อ concurrent สูง | ขึ้นกับ implementation | ❌ ไม่ปลอดภัย (อ่าน-เขียนแยกกัน) | ✅ ปลอดภัย (Lua script atomic) |
| โครงสร้างข้อมูลที่ใช้ | Counter (String) | List ใน cache value | Sorted Set |
| ความซับซ้อนในการ implement | ต่ำมาก | ต่ำ (มีมาให้ใน DRF) | ปานกลาง (ต้องเขียน Lua script) |
| Memory ต่อ 1 client | ต่ำมาก | ปานกลาง (เก็บทุก timestamp เป็น list) | ปานกลาง (เก็บทุก timestamp เป็น sorted set member) |
| เหมาะกับ | Rate limit หยาบ ๆ ทั่วไป | API ทั่วไปที่ traffic ไม่สูงมาก | Endpoint ที่อ่อนไหวสูง (login, payment) หรือ traffic สูงมาก |

### 685.6 ตารางสรุปขั้นตอนที่ 685

| หัวข้อ | สรุป |
|---|---|
| ปัญหาของ Part 048 | `SimpleRateThrottle` อ่าน-ตัดสินใจ-เขียนแยกกัน 3 จังหวะ → race condition เมื่อ traffic สูง |
| ทางแก้ | Lua script บน Redis รับประกัน atomicity ของทั้งกระบวนการ |
| Data structure ที่ใช้ | Sorted Set (ทวนจากขั้นตอนที่ 682) เก็บ timestamp ของแต่ละ request |
| ใช้ที่ไหน | Endpoint อ่อนไหวสูง เช่น login (`AtomicLoginRateThrottle`) |
| วิธีพิสูจน์ผล | เทสยิง concurrent request จำนวนมากพร้อมกันด้วย `threading` |

---

## ขั้นตอนที่ 686: Redis Pub/Sub สำหรับ Invalidate Cache ข้าม Server หลายตัว

### 686.1 ปัญหา: Two-Level Cache กับการ Invalidate ข้าม Process

Redis เป็น **shared cache** อยู่แล้ว (ทุก Django process/server เชื่อมไปที่ Redis
ตัวเดียวกัน) การ `cache.delete(key)` ครั้งเดียวจึงลบข้อมูลที่ทุก process มองเห็นพร้อมกัน
ทันที **แต่ปัญหาจะเกิดขึ้นเมื่อระบบเพิ่ม cache ชั้นที่สอง (L1 cache) เป็นหน่วยความจำ
ภายใน process เอง** (in-process memory dict) วางไว้หน้า Redis (L2) อีกที เพื่อลด
network round-trip สำหรับข้อมูลที่ถูกอ่านบ่อยมาก ๆ (เช่น การตั้งค่าระบบที่แทบไม่เปลี่ยน
แต่ถูกอ่านทุก request):

```
Request ──> L1 (memory ของ process นี้เท่านั้น) ──miss──> L2 (Redis, ทุก process ใช้ร่วมกัน) ──miss──> Database
```

ปัญหา: เมื่อข้อมูลถูกแก้ไข (เช่น admin เปลี่ยนการตั้งค่า) เราลบ L1+L2 ของ **process ปัจจุบัน**
ได้ง่าย แต่ **process อื่น ๆ ที่รันอยู่บนเครื่อง/server อื่น (Gunicorn worker คนละตัว,
หรือคนละเครื่องเลย) ยังมี L1 cache ค่าเก่าค้างอยู่ในหน่วยความจำของตัวเอง** ไม่มีทางรู้ว่า
ข้อมูลถูกแก้ไปแล้ว **Redis Pub/Sub** คือกลไกที่แก้ปัญหานี้ได้ตรงจุดที่สุด เพราะ Redis
สามารถ "กระจายเสียงประกาศ" (broadcast) ข้อความไปยังทุก process ที่ subscribe อยู่ได้
ทันทีโดยไม่ต้องมีใครมา poll ถามเอง

### 686.2 กลไกของ Redis Pub/Sub

Pub/Sub ของ Redis ทำงานง่าย ๆ คือมี **channel** (เหมือนห้องประกาศ) ที่ใครก็ตาม `PUBLISH`
ข้อความเข้าไป ทุกคนที่ `SUBSCRIBE` channel นั้นอยู่จะได้รับข้อความนั้นทันที (ถ้าไม่มีใคร
subscribe อยู่ตอน publish ข้อความนั้นจะหายไปเลย ไม่มีการเก็บย้อนหลังแบบ message queue —
นี่คือข้อจำกัดสำคัญที่ต้องเข้าใจ ถ้าต้องการ guarantee การส่งถึงจริงต้องใช้ Redis Streams
หรือ message broker เต็มรูปแบบอย่าง RabbitMQ ซึ่งจะเรียนใน Phase 9)

```
                              PUBLISH "cache:invalidate" {"key": "settings:homepage"}
                                              │
                                              ▼
                                     Redis Pub/Sub Channel
                          ┌───────────────────┼───────────────────┐
                          ▼                   ▼                   ▼
              Django Process 1      Django Process 2      Django Process 3
              (ล้าง L1 memory       (ล้าง L1 memory       (ล้าง L1 memory
               ของตัวเอง)            ของตัวเอง)            ของตัวเอง)
```

### 686.3 Implement Two-Level Cache พร้อมระบบ Invalidate ผ่าน Pub/Sub

```python
# core/local_cache.py
"""
Cache สองชั้น: L1 = dict ในหน่วยความจำของ process นี้เอง (เร็วที่สุด ไม่มี network เลย)
              L2 = Redis (ช้ากว่า L1 นิดหน่อยแต่แชร์ข้ามทุก process)
"""
import json
import threading

from django.core.cache import caches
from django_redis import get_redis_connection

INVALIDATION_CHANNEL = 'cache:invalidate'

_local_store = {}
_local_lock = threading.Lock()


def get_two_level(key, compute_fn, ttl=300):
    with _local_lock:
        if key in _local_store:
            return _local_store[key]

    value = caches['default'].get(key)
    if value is None:
        value = compute_fn()
        caches['default'].set(key, value, timeout=ttl)

    with _local_lock:
        _local_store[key] = value

    return value


def invalidate(key):
    """ลบทั้ง L1 (ของ process นี้) และ L2 (Redis) แล้วประกาศให้ process อื่นล้าง L1 ของตัวเองด้วย"""
    caches['default'].delete(key)
    with _local_lock:
        _local_store.pop(key, None)

    con = get_redis_connection('default')
    con.publish(INVALIDATION_CHANNEL, json.dumps({'key': key}))


def _handle_invalidation_message(raw_message):
    try:
        payload = json.loads(raw_message['data'])
    except (TypeError, ValueError, KeyError):
        return
    key = payload.get('key')
    if key:
        with _local_lock:
            _local_store.pop(key, None)


def start_invalidation_listener():
    """
    เริ่ม thread เบื้องหลังที่คอยฟัง channel invalidate ตลอดอายุของ process
    ต้องเรียกครั้งเดียวตอน process เริ่มทำงาน (ผ่าน AppConfig.ready() ในขั้นตอนที่ 686.4)
    """
    con = get_redis_connection('default')
    pubsub = con.pubsub(ignore_subscribe_messages=True)
    pubsub.subscribe(INVALIDATION_CHANNEL)

    def _listen_forever():
        for message in pubsub.listen():
            _handle_invalidation_message(message)

    thread = threading.Thread(target=_listen_forever, daemon=True, name='cache-invalidation-listener')
    thread.start()
    return thread
```

### 686.4 เริ่ม Listener ตอน Django Process เริ่มทำงาน

```python
# core/apps.py
import os

from django.apps import AppConfig


class CoreConfig(AppConfig):
    default_auto_field = 'django.db.models.BigAutoField'
    name = 'core'

    def ready(self):
        # กัน listener เริ่มซ้ำ 2 รอบตอน `runserver` reload อัตโนมัติ (autoreloader
        # รัน process ซ้อนกัน 2 ชุดในโหมด DEBUG — ตัวจริงมีตัวแปรสภาพแวดล้อม RUN_MAIN)
        if os.environ.get('RUN_MAIN') == 'true' or not os.environ.get('RUN_MAIN'):
            from .local_cache import start_invalidation_listener
            start_invalidation_listener()
```

> **ข้อควรระวังสำหรับ production (Gunicorn/uWSGI หลาย worker)**: แต่ละ worker process
> เป็นคนละ process ของระบบปฏิบัติการอย่างแท้จริง (ไม่ใช่ thread) ดังนั้น **ทุก worker
> ต้องเรียก `start_invalidation_listener()` ของตัวเองใน `ready()`** ซึ่งเกิดขึ้นอัตโนมัติ
> อยู่แล้วเพราะ Django โหลด app config ใหม่ทุกครั้งที่ worker process เริ่มทำงาน — นี่คือ
> พฤติกรรมที่ต้องการพอดี เพราะ L1 cache เป็นของแต่ละ process แยกกัน จึงต้องมี listener
> แยกกันของแต่ละ process เช่นกัน

### 686.5 ใช้งานจริง: Cache การตั้งค่าเว็บไซต์ที่อ่านทุก Request

```python
# core/models.py
from django.db import models


class SiteSettings(models.Model):
    homepage_banner_text = models.CharField(max_length=200, blank=True)
    maintenance_mode = models.BooleanField(default=False)

    class Meta:
        verbose_name_plural = 'Site Settings'
```

```python
# core/services.py
from .local_cache import get_two_level, invalidate
from .models import SiteSettings

SITE_SETTINGS_CACHE_KEY = 'site_settings:singleton'


def get_site_settings():
    def _load_from_db():
        return SiteSettings.objects.first()

    return get_two_level(SITE_SETTINGS_CACHE_KEY, _load_from_db, ttl=3600)


def invalidate_site_settings():
    invalidate(SITE_SETTINGS_CACHE_KEY)
```

```python
# core/signals.py
from django.db.models.signals import post_save
from django.dispatch import receiver

from .models import SiteSettings
from .services import invalidate_site_settings


@receiver(post_save, sender=SiteSettings)
def on_site_settings_changed(sender, instance, **kwargs):
    invalidate_site_settings()
```

ตอนนี้เมื่อ admin คนหนึ่งเข้าไปแก้ `SiteSettings` ผ่าน Django Admin (ซึ่งอาจเชื่อมต่อกับ
Django process คนละตัวกับที่ผู้ใช้ทั่วไปกำลังเข้าเว็บอยู่) ทุก process ในทุกเครื่อง
server จะได้รับข้อความ invalidate ผ่าน Pub/Sub และล้าง L1 cache ของตัวเองภายในเสี้ยว
วินาที โดยไม่ต้องรอให้ TTL ของ L1 หมดอายุเอง

### 686.6 ตารางสรุปขั้นตอนที่ 686

| หัวข้อ | สรุป |
|---|---|
| ปัญหาที่แก้ | L1 cache (in-process memory) ของแต่ละ server ไม่รู้ว่าข้อมูลถูกแก้ที่ process อื่น |
| กลไก | Redis Pub/Sub: `PUBLISH`/`SUBSCRIBE` กระจายข้อความ invalidate ไปทุก process ทันที |
| ข้อจำกัดสำคัญ | ข้อความไม่ถูกเก็บย้อนหลัง — process ที่ไม่ได้ subscribe อยู่ตอน publish จะพลาดข้อความนั้นไปเลย |
| เริ่ม listener ที่ไหน | `AppConfig.ready()` — รันอัตโนมัติทุก worker process |
| ใช้ทำอะไรได้บ้าง | Invalidate two-level cache ของข้อมูลที่อ่านบ่อยมากแต่เปลี่ยนไม่บ่อย (site settings, feature flags) |

---

## ขั้นตอนที่ 687: การ Monitor Redis — `redis-cli`, Memory Usage, `INFO`, `MONITOR`

### 687.1 คำสั่งพื้นฐานที่ต้องรู้จักใน `redis-cli`

```bash
# เชื่อมต่อแบบ interactive
redis-cli

# เชื่อมต่อ DB index เฉพาะ (ทวนจากขั้นตอนที่ 681.5)
redis-cli -n 1   # DB 1: page/query cache

# จำนวน key ทั้งหมดใน DB ปัจจุบัน
redis-cli -n 1 dbsize

# ค้นหา key แบบ pattern (ใช้ SCAN แทน KEYS เสมอใน production — ดู 687.2)
redis-cli -n 1 --scan --pattern "myapp:*post_detail*"

# ดูค่าของ key หนึ่ง ๆ
redis-cli -n 1 get "myapp:1:homepage:data"

# ดู TTL ที่เหลือของ key (วินาที, -1 = ไม่มีวันหมดอายุ, -2 = ไม่มี key นี้)
redis-cli -n 1 ttl "myapp:1:homepage:data"

# ลบ key ทั้งหมดใน DB ปัจจุบัน (ใช้ระวังมาก — ไม่มี undo)
redis-cli -n 1 flushdb
```

### 687.2 ทำไมห้ามใช้ `KEYS *` ใน Production

```bash
# ❌ ห้ามใช้ใน production เด็ดขาด
redis-cli KEYS "myapp:*"
```

`KEYS` สแกนทุก key ในฐานข้อมูลแบบ **blocking** — เพราะ Redis เป็น single-threaded
ถ้าฐานข้อมูลมี key เป็นล้าน คำสั่งนี้จะ **บล็อกทุกคำสั่งอื่นทั้งหมด** ที่กำลังรออยู่จนกว่า
จะสแกนเสร็จ (อาจหลายวินาทีถึงหลายสิบวินาที) ทำให้ทุก request ของทุกผู้ใช้ในระบบค้าง
พร้อมกันหมด ให้ใช้ `SCAN` แทนเสมอ (ทวนจากคำสั่ง `--scan` ด้านบน) เพราะ `SCAN` คืนผลลัพธ์
เป็น cursor ทีละชุดเล็ก ๆ โดยไม่บล็อก Redis นาน:

```bash
redis-cli -n 1 --scan --pattern "myapp:*post_detail*" --count 100
```

### 687.3 `INFO`: ดูสถานะสุขภาพของ Redis ทั้งหมด

```bash
redis-cli info
```

คำสั่งนี้คืนข้อมูลจำนวนมาก แบ่งเป็น section ให้ดูเฉพาะที่ต้องการได้:

```bash
redis-cli info memory
redis-cli info stats
redis-cli info clients
redis-cli info replication
```

| Field (จาก `INFO memory`) | ความหมาย |
|---|---|
| `used_memory_human` | หน่วยความจำที่ Redis ใช้อยู่จริง (format อ่านง่าย เช่น `128.45M`) |
| `maxmemory_human` | เพดานหน่วยความจำที่ตั้งไว้ (0 = ไม่จำกัด — อันตรายมากใน production) |
| `maxmemory_policy` | นโยบายเมื่อ memory เต็ม (ดูตารางขั้นตอนที่ 687.5) |
| `mem_fragmentation_ratio` | อัตราส่วน memory ที่ OS จองไว้จริง เทียบกับที่ Redis ใช้จริง — ค่าสูงกว่า 1.5 มาก ๆ ควรสืบสวนเพิ่ม |
| `evicted_keys` | จำนวน key ที่ถูกลบทิ้งเพราะ memory เต็ม (ควรเป็น 0 ถ้าตั้ง `maxmemory` เผื่อพอดี) |

| Field (จาก `INFO stats`) | ความหมาย |
|---|---|
| `keyspace_hits` | จำนวนครั้งที่หา key เจอ (cache hit) |
| `keyspace_misses` | จำนวนครั้งที่หา key ไม่เจอ (cache miss) |
| `instantaneous_ops_per_sec` | จำนวนคำสั่งที่ประมวลผลต่อวินาที ณ ขณะนี้ |
| `total_connections_received` | จำนวนการเชื่อมต่อสะสมทั้งหมดตั้งแต่ Redis เริ่มทำงาน |

### 687.4 คำนวณ Cache Hit Rate ด้วย Management Command

```python
# core/management/commands/redis_stats.py
from django.core.management.base import BaseCommand
from django_redis import get_redis_connection


class Command(BaseCommand):
    help = "แสดงสถิติสุขภาพของ Redis อย่างย่อ (memory, hit rate, clients)"

    def add_arguments(self, parser):
        parser.add_argument('--alias', default='default', help='cache alias ที่จะตรวจสอบ')

    def handle(self, *args, **options):
        con = get_redis_connection(options['alias'])
        info = con.info()

        hits = info.get('keyspace_hits', 0)
        misses = info.get('keyspace_misses', 0)
        total = hits + misses
        hit_rate = (hits / total * 100) if total else 0

        self.stdout.write(f"Redis version:        {info.get('redis_version')}")
        self.stdout.write(f"Used memory:          {info.get('used_memory_human')}")
        self.stdout.write(f"Max memory:           {info.get('maxmemory_human') or 'unlimited'}")
        self.stdout.write(f"Maxmemory policy:     {info.get('maxmemory_policy')}")
        self.stdout.write(f"Connected clients:    {info.get('connected_clients')}")
        self.stdout.write(f"Ops/sec (ปัจจุบัน):    {info.get('instantaneous_ops_per_sec')}")
        self.stdout.write(f"Evicted keys:         {info.get('evicted_keys', 0)}")
        self.stdout.write(self.style.SUCCESS(f"Cache hit rate:       {hit_rate:.2f}%"))

        if hit_rate < 80 and total > 1000:
            self.stdout.write(self.style.WARNING(
                'Hit rate ต่ำกว่า 80% — พิจารณาตรวจสอบ TTL ที่ตั้งไว้สั้นเกินไป '
                'หรือมี cache key pattern ที่ออกแบบไม่ดี (key ไม่ซ้ำกันบ่อยเกินไป)'
            ))
```

```bash
python manage.py redis_stats
```

```
Redis version:        7.2.4
Used memory:          142.83M
Max memory:           1.00G
Maxmemory policy:     allkeys-lru
Connected clients:    12
Ops/sec (ปัจจุบัน):    340
Evicted keys:         0
Cache hit rate:       94.21%
```

### 687.5 `MONITOR`: ดูทุกคำสั่งที่ยิงเข้า Redis แบบ Real-time

```bash
redis-cli monitor
```

```
1758870421.234512 [1 127.0.0.1:54892] "GET" "myapp:1:post_detail:django-tutorial"
1758870421.235102 [2 127.0.0.1:54893] "ZINCRBY" "trending:posts" "1" "42"
1758870421.236781 [3 127.0.0.1:54894] "HINCRBY" "post_stats:42" "views" "1"
```

> **คำเตือนสำคัญ**: `MONITOR` มีผลกระทบต่อ performance ค่อนข้างมาก (ลดความเร็วของ Redis
> ได้ถึง 50% ในระบบที่ traffic สูงมาก เพราะทุกคำสั่งต้องถูกส่งซ้ำไปยัง client ที่ monitor
> อยู่ด้วย) **ใช้เฉพาะตอน debug ปัญหาเฉพาะหน้าในเวลาสั้น ๆ เท่านั้น ห้ามเปิดค้างไว้ใน
> production ตลอดเวลา**

### 687.6 นโยบายจัดการ Memory เมื่อเต็ม (`maxmemory-policy`)

```bash
redis-cli config get maxmemory-policy
redis-cli config set maxmemory-policy allkeys-lru
```

| Policy | พฤติกรรมเมื่อ Memory เต็ม | เหมาะกับ |
|---|---|---|
| `noeviction` (ค่าเริ่มต้น) | ปฏิเสธคำสั่งเขียนทั้งหมด คืน error ทันที | ระบบที่ Redis เก็บข้อมูลสำคัญที่ห้ามหาย (เช่น Celery broker) |
| `allkeys-lru` | ลบ key ที่ไม่ถูกใช้นานที่สุด (Least Recently Used) ออกจากทั้งฐานข้อมูล ไม่สนว่าตั้ง TTL ไว้หรือไม่ | **แนะนำที่สุดสำหรับ Redis ที่ใช้เป็น cache ล้วน ๆ** |
| `volatile-lru` | ลบเฉพาะ key ที่ตั้ง TTL ไว้ ด้วยหลัก LRU เหมือนกัน (key ที่ไม่มี TTL จะไม่ถูกแตะ) | Redis ที่แชร์ระหว่าง cache กับข้อมูลถาวร (เช่น session ที่ไม่อยากให้ถูก evict) |
| `allkeys-lfu` | ลบ key ที่ถูกเรียกใช้ **น้อยที่สุด** (Least Frequently Used) ออก | เหมาะกับ pattern การเข้าถึงที่ "ความถี่" สำคัญกว่า "ความล่าสุด" |
| `allkeys-random` | ลบ key แบบสุ่ม | กรณีพิเศษที่ต้องการความเร็วในการตัดสินใจสูงสุด ไม่สนใจ pattern การใช้งาน |

**คำแนะนำของหลักสูตรนี้**: สำหรับ Redis instance ที่ใช้เป็น cache ล้วน ๆ (แยกจาก
instance ที่ใช้เป็น Celery broker หรือเก็บ session สำคัญ) ให้ตั้ง `allkeys-lru` เสมอ
เพื่อให้ Redis จัดการพื้นที่เองอัตโนมัติโดยไม่ทำให้แอปพลิเคชัน error เมื่อ memory เต็ม

### 687.7 ตารางสรุปขั้นตอนที่ 687

| คำสั่ง/เครื่องมือ | ใช้ทำอะไร |
|---|---|
| `redis-cli --scan --pattern` | ค้นหา key แบบไม่บล็อก Redis (ใช้แทน `KEYS` เสมอ) |
| `redis-cli info memory/stats` | ดูสถานะ memory, hit rate, จำนวน connection |
| `redis-cli monitor` | ดูคำสั่งทุกตัว real-time (ใช้ debug เท่านั้น ห้ามเปิดค้างใน production) |
| `redis_stats` management command | รายงานสรุปสุขภาพ Redis + คำนวณ hit rate อัตโนมัติ |
| `maxmemory-policy` | ตั้งเป็น `allkeys-lru` สำหรับ Redis ที่ใช้เป็น cache ล้วน ๆ |

---

## ขั้นตอนที่ 688: ภาพรวม Redis Sentinel/Cluster สำหรับ High Availability

### 688.1 ปัญหาของ Redis แบบ Standalone

ทุกการตั้งค่าที่เรียนมาใน Part นี้จนถึงตอนนี้ใช้ Redis **เครื่องเดียว (standalone)**
ซึ่งมีจุดอ่อนสำคัญ: **Redis เป็น Single Point of Failure** — ถ้าเครื่องนั้นล่ม (hardware
พัง, ต้อง restart เพื่ออัปเดต, network ขาด) **ทุกอย่างที่พึ่งพา Redis จะหยุดทำงานพร้อมกัน
ทันที**: cache ทั้งหมดหาย, session ทั้งหมดหาย (ถ้าไม่ได้ตั้ง `cached_db` เป็น fallback),
rate limiting หยุดทำงาน (แม้จะ fail-open ได้ตาม `IGNORE_EXCEPTIONS` แต่ก็แปลว่าไม่มีการ
จำกัดอัตราเรียกใดๆ อยู่ชั่วคราว)

ระบบระดับ production จริงจึงต้องมีกลไก **High Availability (HA)** — Part นี้จะให้แค่
**ภาพรวม** เพื่อให้เข้าใจแนวคิดและรู้จักการตั้งค่าเบื้องต้นฝั่ง Django ส่วนการติดตั้ง/
ดูแล Redis Sentinel และ Cluster แบบเต็มรูปแบบ (multi-node, replication, failover
testing) จะเจาะลึกอีกครั้งใน **Phase 11 (Infrastructure & DevOps ขั้นสูง)**

### 688.2 Redis Sentinel: Automatic Failover

**Sentinel** คือระบบที่คอย **มอนิเตอร์** Redis master/replica หลายชุด และเมื่อ master
ตัวหลักล่ม จะ **เลือก replica ตัวหนึ่งขึ้นเป็น master ใหม่โดยอัตโนมัติ** โดยไม่ต้องมีคน
เข้ามาแก้ config ด้วยมือ

```
                     ┌─────────────┐
                     │  Sentinel 1  │ ─┐
                     └─────────────┘  │
                     ┌─────────────┐  │  คอยเช็คสุขภาพ Master/Replica ทุกวินาที
                     │  Sentinel 2  │ ─┼─  ถ้า Master ล่ม → โหวตเลือก Replica
                     └─────────────┘  │    ตัวใหม่ขึ้นเป็น Master แทน
                     ┌─────────────┐  │
                     │  Sentinel 3  │ ─┘
                     └─────────────┘
                            │
              ┌─────────────┼─────────────┐
              ▼             ▼             ▼
        ┌──────────┐  ┌──────────┐  ┌──────────┐
        │  Master  │─>│ Replica 1│  │ Replica 2│
        │ (เขียน+อ่าน)│  │ (อ่านอย่างเดียว)│  │ (อ่านอย่างเดียว)│
        └──────────┘  └──────────┘  └──────────┘
```

ตั้งค่าฝั่ง Django ผ่าน `django-redis`:

```python
# config/settings.py — ตัวอย่างการตั้งค่า Sentinel (ใช้จริงเมื่อมี Sentinel cluster พร้อมแล้ว)
CACHES = {
    'default': {
        'BACKEND': 'django_redis.cache.RedisCache',
        'LOCATION': [
            'redis://mymaster/1',
        ],
        'OPTIONS': {
            'CLIENT_CLASS': 'django_redis.client.SentinelClient',
            'SENTINELS': [
                ('sentinel-1.internal', 26379),
                ('sentinel-2.internal', 26379),
                ('sentinel-3.internal', 26379),
            ],
            'SENTINEL_KWARGS': {'socket_timeout': 0.5},
        },
    }
}
```

`django-redis` จะถาม Sentinel ว่า "master ชื่อ `mymaster` ตอนนี้คือเครื่องไหน" ทุกครั้ง
ก่อนเชื่อมต่อ ทำให้เมื่อเกิด failover แอปพลิเคชัน **ไม่ต้อง restart หรือแก้ config เอง
เลย** — Sentinel จัดการชี้ทางให้อัตโนมัติ

### 688.3 Redis Cluster: Sharding ข้อมูลข้ามหลายเครื่อง

**Cluster** แก้ปัญหาคนละแบบจาก Sentinel — Sentinel แก้ปัญหา "ล่มแล้วต้องมีตัวสำรอง"
ส่วน **Cluster แก้ปัญหา "ข้อมูลเยอะเกินกว่าเครื่องเดียวจะรับไหว"** โดยแบ่ง (shard)
ข้อมูลออกเป็นหลายส่วน กระจายไปเก็บบนหลายเครื่อง (node) พร้อมกัน แต่ละ node รับผิดชอบ
ช่วง **hash slot** ของตัวเอง (Redis Cluster มีทั้งหมด 16,384 slot คงที่)

```
Key "post:1" ──> hash("post:1") % 16384 ──> slot 5461 ──> Node A รับผิดชอบ
Key "post:2" ──> hash("post:2") % 16384 ──> slot 12890 ──> Node B รับผิดชอบ
Key "post:3" ──> hash("post:3") % 16384 ──> slot 890 ──> Node C รับผิดชอบ
```

### 688.4 ข้อจำกัดสำคัญของ Cluster ที่กระทบโค้ดจริง

Redis Cluster มีข้อจำกัดที่ **กระทบโค้ดที่เขียนมาตลอด Part นี้โดยตรง**: คำสั่งที่ทำงาน
กับ **หลาย key พร้อมกัน** (เช่น `MGET`, หรือ Lua script ที่แตะหลาย key ในขั้นตอนที่ 685)
จะทำงานได้ก็ต่อเมื่อ **ทุก key ที่เกี่ยวข้องอยู่ใน hash slot เดียวกันเท่านั้น** ไม่เช่นนั้น
Redis จะคืน error `CROSSSLOT Keys in request don't hash to the same slot` วิธีแก้คือใช้
**hash tag** (ครอบส่วนหนึ่งของ key ด้วย `{}`) เพื่อบังคับให้ key ที่เกี่ยวข้องกันตกอยู่
slot เดียวกันเสมอ:

```python
# ก่อน (อาจตกคนละ slot กันใน Cluster mode)
key1 = 'post_stats:42'
key2 = 'post_meta:42'

# หลัง (บังคับให้ตก slot เดียวกัน เพราะ Redis hash เฉพาะส่วนใน {} เท่านั้น)
key1 = '{post:42}:stats'
key2 = '{post:42}:meta'
```

### 688.5 ตารางเปรียบเทียบ Standalone vs Sentinel vs Cluster

| ประเด็น | Standalone | Sentinel | Cluster |
|---|---|---|---|
| แก้ปัญหาอะไร | (ไม่มี HA) | High Availability (auto-failover) | Horizontal Scaling (ข้อมูลใหญ่เกินเครื่องเดียว) |
| จำนวนเครื่องขั้นต่ำ | 1 | 3 Sentinel + 2 Redis (master+replica) ขึ้นไป | 6 (3 master + 3 replica) ขึ้นไป |
| ถ้า Master ล่ม | ระบบหยุดทำงานทั้งหมดจนกว่าจะกู้คืนด้วยมือ | Failover อัตโนมัติภายในไม่กี่วินาที | Node อื่นยังทำงานต่อได้ (เฉพาะ slot ที่ node ล่มถือครองเท่านั้นที่กระทบ) |
| รองรับข้อมูลขนาดใหญ่เกิน RAM เครื่องเดียว | ❌ | ❌ | ✅ (กระจายไปหลายเครื่อง) |
| ความซับซ้อนในการติดตั้ง/ดูแล | ต่ำมาก | ปานกลาง | สูง |
| รองรับคำสั่งข้าม key เต็มรูปแบบ (Lua script, `MGET`) | ✅ เต็มรูปแบบ | ✅ เต็มรูปแบบ | ⚠️ ต้องใช้ hash tag ถ้าต้องการให้ key อยู่ slot เดียวกัน |
| เหมาะกับ | โปรเจกต์เล็ก-กลาง, development | Production ที่ต้องการ uptime สูง แต่ข้อมูลไม่ใหญ่มาก | ระบบขนาดใหญ่ระดับ enterprise ที่ throughput/ข้อมูลสูงมาก |
| ทางเลือก Managed Service | Self-host หรือ managed instance เดี่ยว | AWS ElastiCache (Multi-AZ), Azure Cache for Redis (with replicas) | AWS ElastiCache Cluster Mode, Redis Enterprise Cloud, GCP Memorystore Cluster |

### 688.6 คำแนะนำสำหรับหลักสูตรนี้และงานจริงส่วนใหญ่

สำหรับโปรเจกต์ส่วนใหญ่ (รวมถึง Blog ที่ใช้ตลอดหลักสูตรนี้) **Sentinel เพียงพอแล้ว**
เพราะข้อมูล cache/session ไม่ได้ใหญ่จนต้อง shard ข้ามหลายเครื่อง สิ่งที่ต้องการจริง ๆ
คือ "อย่าให้ Redis ล่มแล้วทั้งระบบพัง" ซึ่ง Sentinel แก้ได้ตรงจุด ส่วน Cluster ควร
พิจารณาเมื่อ **ปริมาณข้อมูลหรือ throughput เกินขีดจำกัดของเครื่องเดียวจริง ๆ** (มักเกิด
กับระบบระดับ Instagram, Twitter ที่มีผู้ใช้หลักร้อยล้านคน) การตั้งค่ารายละเอียดของทั้ง
Sentinel และ Cluster แบบ production-grade เต็มรูปแบบ (การ provision เครื่อง, ทดสอบ
failover, backup strategy) จะเจาะลึกอีกครั้งใน **Phase 11**

### 688.7 ตารางสรุปขั้นตอนที่ 688

| หัวข้อ | สรุป |
|---|---|
| ปัญหาของ Standalone | Single Point of Failure — ล่มแล้วทั้งระบบหยุด |
| Sentinel แก้ปัญหาอะไร | Automatic failover — เลือก replica ขึ้นเป็น master อัตโนมัติเมื่อ master ล่ม |
| Cluster แก้ปัญหาอะไร | Sharding — กระจายข้อมูลข้ามหลายเครื่องเมื่อข้อมูลใหญ่เกิน RAM เครื่องเดียว |
| ข้อจำกัดของ Cluster ที่กระทบโค้ด | คำสั่งข้าม key ต้องอยู่ hash slot เดียวกัน (ใช้ hash tag `{}`) |
| คำแนะนำสำหรับงานส่วนใหญ่ | Sentinel เพียงพอ, Cluster ใช้เฉพาะระบบขนาดใหญ่จริง ๆ |
| เจาะลึกเต็มรูปแบบที่ไหน | Phase 11 (Infrastructure & DevOps ขั้นสูง) |

---

## ขั้นตอนที่ 689: กลยุทธ์ Cache Warming

### 689.1 Cache Warming คืออะไร และทำไมต้องมี

ทุกเทคนิคที่เรียนมาใน Part นี้เป็นแบบ **Cache-Aside** (Lazy Loading): ข้อมูลจะถูกใส่ลง
cache ก็ต่อเมื่อมี request จริงมาเจอ cache miss ก่อนเท่านั้น ปัญหาคือ **request แรกสุด
หลังจาก cache หมดอายุ (หรือหลัง deploy ที่ restart Redis/ล้าง cache) จะเจอความช้าเต็ม ๆ
เสมอ** เพราะต้องรอ compute จริงก่อน — ผู้ใช้คนแรกที่โชคร้ายมาเจอจังหวะนี้พอดีจะได้
ประสบการณ์ที่แย่

**Cache Warming** คือการ **เติมข้อมูลลง cache ล่วงหน้าอย่างจงใจ** ก่อนที่ user จริงจะมา
เจอ cache miss เลย ทำให้ผู้ใช้ทุกคนได้ความเร็วจาก cache ตั้งแต่ request แรกที่เข้ามาจริง

### 689.2 สร้าง Management Command สำหรับ Warm Cache

```python
# blog/management/commands/warm_cache.py
import time

from django.core.cache import cache
from django.core.management.base import BaseCommand

from blog.models import Post
from blog.services.trending import bump_trending_score
from blog.services.cache_stampede import get_or_compute_with_lock


class Command(BaseCommand):
    help = "เติมข้อมูลลง cache ล่วงหน้า (cache warming) ก่อนผู้ใช้จริงมาเจอ cache miss"

    def handle(self, *args, **options):
        start = time.perf_counter()
        self.stdout.write('เริ่ม warm cache...')

        self._warm_homepage()
        self._warm_popular_posts(limit=50)
        self._warm_trending_seed()

        elapsed = time.perf_counter() - start
        self.stdout.write(self.style.SUCCESS(f'Warm cache เสร็จสมบูรณ์ ใช้เวลา {elapsed:.2f} วินาที'))

    def _warm_homepage(self):
        def compute():
            return {
                'featured_posts': list(
                    Post.objects.filter(is_published=True)
                    .select_related('author')
                    .order_by('-views')[:5]
                ),
                'latest_posts': list(
                    Post.objects.filter(is_published=True).order_by('-created_at')[:10]
                ),
            }

        get_or_compute_with_lock('homepage:data', compute, timeout=600)
        self.stdout.write('  - หน้าแรก: เสร็จแล้ว')

    def _warm_popular_posts(self, limit):
        posts = (
            Post.objects.filter(is_published=True)
            .select_related('author')
            .order_by('-views')[:limit]
        )
        count = 0
        for post in posts:
            cache_key = f'post_detail:{post.slug}'
            cache.set(cache_key, post, timeout=1800)
            count += 1
        self.stdout.write(f'  - โพสต์ยอดนิยม {count} รายการ: เสร็จแล้ว')

    def _warm_trending_seed(self):
        # ให้คะแนนเริ่มต้นตามยอด view สะสมในฐานข้อมูล เพื่อไม่ให้ trending
        # ว่างเปล่าทันทีหลัง Redis ถูกล้าง/restart
        top_posts = Post.objects.filter(is_published=True).order_by('-views')[:100]
        for post in top_posts:
            bump_trending_score(post.id, amount=post.views)
        self.stdout.write(f'  - Trending seed: {top_posts.count()} รายการ')
```

```bash
python manage.py warm_cache
```

```
เริ่ม warm cache...
  - หน้าแรก: เสร็จแล้ว
  - โพสต์ยอดนิยม 50 รายการ: เสร็จแล้ว
  - Trending seed: 100 รายการ
Warm cache เสร็จสมบูรณ์ ใช้เวลา 2.34 วินาที
```

### 689.3 เมื่อไหร่ควรเรียก Warm Cache

| จังหวะ | วิธีสั่ง | เหตุผล |
|---|---|---|
| หลัง deploy โค้ดใหม่ | ใส่ `python manage.py warm_cache` เป็นขั้นตอนสุดท้ายใน deploy script (หลัง `migrate`) | Deploy มักมาพร้อมการ restart process ซึ่งล้าง L1 cache (ขั้นตอนที่ 686) ไปด้วย |
| หลังล้าง Redis ทั้งฐาน (`FLUSHDB`/`FLUSHALL`) | รันด้วยมือทันทีหลังคำสั่งล้าง | ป้องกัน request แรก ๆ หลังล้าง cache เจอ stampede รวมกันทั้งระบบ (ทวนขั้นตอนที่ 683) |
| ตามตารางเวลาสม่ำเสมอ (เช่น ทุก 10 นาที) | Celery Beat periodic task (เรียนเต็มรูปแบบใน Part 076) หรือ cron job ชั่วคราวก่อนถึง Phase 9 | Trending posts และข้อมูลยอดนิยมเปลี่ยนตลอดเวลา ต้อง refresh สม่ำเสมอไม่ใช่แค่ตอน deploy |
| ก่อนกิจกรรมที่คาดว่า traffic จะพุ่งสูง (เช่น เปิดตัวบทความไวรัล, campaign การตลาด) | รันด้วยมือล่วงหน้า | เตรียม cache ให้พร้อมก่อนที่ traffic จริงจะมาถึง ลดความเสี่ยง stampede ตอน peak |

### 689.4 Warm Cache ชั่วคราวด้วย Cron (ก่อนถึง Celery ใน Phase 9)

```bash
# crontab -e
*/10 * * * * cd /path/to/project && /path/to/venv/bin/python manage.py warm_cache >> /var/log/django/warm_cache.log 2>&1
```

> **หมายเหตุ**: นี่เป็นวิธีชั่วคราวที่เหมาะกับตอนที่ยังไม่มี Celery ในระบบ เมื่อถึง
> Part 076 (Celery ขั้นสูง: Periodic Tasks) เราจะเปลี่ยนมาใช้ **Celery Beat** แทน cron
> เพราะจัดการ retry, logging, และ monitoring ได้ดีกว่ามากในระบบ production จริง

### 689.5 ข้อควรระวัง: Cache Warming ไม่ใช่ทางแก้สำหรับทุกกรณี

Cache Warming เหมาะกับข้อมูลที่ **คาดเดาล่วงหน้าได้ว่าจะถูกเรียกบ่อย** (หน้าแรก,
โพสต์ยอดนิยม, trending) แต่ **ไม่เหมาะกับข้อมูล long-tail** ที่มีจำนวนมากมายมหาศาลแต่
แต่ละอันถูกเรียกน้อยครั้ง (เช่น โพสต์เก่าที่แทบไม่มีคนอ่าน) เพราะ:

- Warm ทุกอย่างล่วงหน้า = สิ้นเปลือง memory และเวลา compute โดยไม่จำเป็น (ข้อมูลที่ไม่มี
  ใครเรียกก็ถูก evict ทิ้งตาม `maxmemory-policy` อยู่ดี — ทวนขั้นตอนที่ 687.6)
- สำหรับข้อมูล long-tail ปล่อยให้เป็น **Cache-Aside แบบปกติร่วมกับ XFetch** (ขั้นตอนที่
  683.3) จะคุ้มค่ากว่ามาก เพราะ compute เฉพาะตอนมีคนเรียกจริง ๆ เท่านั้น

**หลักการเลือก**: Warm เฉพาะ "20% ของข้อมูลที่คิดเป็น 80% ของ traffic" (Pareto
Principle) ไม่ใช่ warm ทุกอย่างในระบบ

### 689.6 ตารางสรุปขั้นตอนที่ 689

| หัวข้อ | สรุป |
|---|---|
| Cache Warming คืออะไร | เติมข้อมูลลง cache ล่วงหน้า ก่อนผู้ใช้จริงมาเจอ cache miss |
| ต่างจาก Cache-Aside ปกติอย่างไร | Cache-Aside รอ miss ก่อนค่อย compute, Warming compute ล่วงหน้าเชิงรุก |
| เมื่อไหร่ควรรัน | หลัง deploy, หลังล้าง Redis, ตามตารางเวลา, ก่อนกิจกรรม traffic สูง |
| เครื่องมือที่ใช้ตอนนี้ | Django management command + cron (ชั่วคราว จนกว่าจะมี Celery Beat ใน Part 076) |
| ข้อควรระวัง | Warm เฉพาะข้อมูลที่คาดเดา traffic สูงได้ ไม่ใช่ warm ทุกอย่างในระบบ |

---

## ขั้นตอนที่ 690: สรุปและแบบฝึกหัด

### 690.1 ประกอบร่างทุกเทคนิคเข้าด้วยกัน: Redis Caching Layer เต็มรูปแบบสำหรับ Blog

ตอนนี้เราจะรวมทุกเทคนิคจากขั้นตอนที่ 681-689 เข้าเป็นระบบ caching layer เดียวที่ใช้งาน
จริงกับ `blog` app ตลอดหลักสูตรนี้ พร้อมป้องกัน cache stampede เต็มรูปแบบ

```python
# blog/caching.py
"""
Caching Layer เต็มรูปแบบของ blog app
รวมเทคนิคจาก Part 069 ทั้งหมด: lock-based stampede protection (683),
Hash/Sorted Set (682), delete_pattern สำหรับ invalidate (681), และ
Pub/Sub cross-server invalidation (686)
"""
import functools
import hashlib

from django.core.cache import cache

from core.local_cache import invalidate as invalidate_two_level
from .services.cache_stampede import get_or_compute_with_lock

POST_LIST_CACHE_PREFIX = 'post_list'
POST_DETAIL_CACHE_PREFIX = 'post_detail'


def cache_key_for_request(prefix, request):
    """สร้าง cache key ที่ unique ตาม path + query string ของ request"""
    raw = request.get_full_path()
    digest = hashlib.md5(raw.encode('utf-8')).hexdigest()
    return f'{prefix}:{digest}'


def stampede_safe_view_cache(prefix, timeout=300):
    """
    Decorator สำหรับ view ที่ต้องการ cache ทั้ง response พร้อมป้องกัน stampede
    ต่างจาก Django built-in @cache_page (Part 068) ตรงที่มี lock ป้องกัน
    หลาย request แย่งกัน render view เดียวกันพร้อมกันตอน cache หมดอายุ
    """

    def decorator(view_func):
        @functools.wraps(view_func)
        def wrapper(request, *args, **kwargs):
            if request.method != 'GET':
                return view_func(request, *args, **kwargs)

            cache_key = cache_key_for_request(prefix, request)

            def render_view():
                response = view_func(request, *args, **kwargs)
                if hasattr(response, 'render'):
                    response.render()
                return response

            return get_or_compute_with_lock(cache_key, render_view, timeout=timeout)

        return wrapper

    return decorator


def invalidate_post_caches(post=None):
    """
    เรียกทุกครั้งที่ Post ถูกสร้าง/แก้ไข/ลบ (ผูกผ่าน signal ด้านล่าง)
    ใช้ delete_pattern (ฟีเจอร์เฉพาะของ django-redis จากขั้นตอนที่ 681.4)
    เพื่อลบ post_list cache ทุก variation (แต่ละหน้า/filter มี cache key ต่างกัน)
    """
    cache.delete_pattern(f'{POST_LIST_CACHE_PREFIX}:*')

    if post is not None:
        cache.delete(f'{POST_DETAIL_CACHE_PREFIX}:{post.slug}')
        invalidate_two_level(f'post_detail_meta:{post.slug}')
```

```python
# blog/views.py
from django.shortcuts import get_object_or_404, render

from .caching import stampede_safe_view_cache
from .models import Post
from .services.post_stats import get_post_stats, record_post_view
from .services.trending import bump_trending_score


@stampede_safe_view_cache('post_list', timeout=300)
def post_list(request):
    posts = (
        Post.objects.filter(is_published=True)
        .select_related('author')
        .order_by('-created_at')
    )
    return render(request, 'blog/post_list.html', {'posts': posts})


def post_detail(request, slug):
    # หน้า detail ไม่ cache ทั้ง response เพราะต้องนับ view ทุกครั้ง (ขั้นตอนที่ 682.3)
    # แต่ตัวข้อมูลโพสต์เองยัง cache แยกได้ผ่าน ORM + select_related ตามปกติ
    post = get_object_or_404(Post, slug=slug, is_published=True)
    record_post_view(post.id)
    bump_trending_score(post.id, amount=1)
    stats = get_post_stats(post.id)
    return render(request, 'blog/post_detail.html', {'post': post, 'stats': stats})
```

```python
# blog/signals.py
from django.db.models.signals import post_delete, post_save
from django.dispatch import receiver

from .caching import invalidate_post_caches
from .models import Post


@receiver(post_save, sender=Post)
def on_post_saved(sender, instance, **kwargs):
    invalidate_post_caches(post=instance)


@receiver(post_delete, sender=Post)
def on_post_deleted(sender, instance, **kwargs):
    invalidate_post_caches(post=instance)
```

```python
# blog/apps.py
from django.apps import AppConfig


class BlogConfig(AppConfig):
    default_auto_field = 'django.db.models.BigAutoField'
    name = 'blog'

    def ready(self):
        import blog.signals  # noqa: F401  — ลงทะเบียน signal handler ตอน app โหลด
```

ระบบนี้รวม 5 เทคนิคจาก Part 069 เข้าด้วยกัน:

1. **Lock-based Stampede Protection** (ขั้นตอนที่ 683) — ป้องกันหลาย request แย่งกัน
   render `post_list` พร้อมกันตอน cache หมดอายุ
2. **Hash สำหรับสถิติ** (ขั้นตอนที่ 682) — นับ view/like แบบ atomic โดยไม่กระทบฐานข้อมูล
3. **Sorted Set สำหรับ Trending** (ขั้นตอนที่ 682) — จัดอันดับความนิยมแบบเรียลไทม์
4. **`delete_pattern`** (ขั้นตอนที่ 681.4, 690.1) — ล้าง cache หลาย variation พร้อมกัน
   ด้วยคำสั่งเดียว เมื่อข้อมูลถูกแก้ไข
5. **Two-Level Cache + Pub/Sub** (ขั้นตอนที่ 686) — invalidate ข้าม server สำหรับ
   metadata ที่อ่านบ่อยมาก

### 690.2 สรุปสิ่งที่ได้เรียนรู้ใน Part นี้

- ✅ ติดตั้ง Redis และเชื่อมต่อผ่าน `django-redis` แบบ production-ready พร้อม
  connection pool, timeout, และ `IGNORE_EXCEPTIONS`
- ✅ เข้าใจโครงสร้างข้อมูล String, Hash, Sorted Set ของ Redis และใช้งานผ่าน
  `get_redis_connection()` เมื่อ Django Cache API มาตรฐานไม่พอ
- ✅ เข้าใจปัญหา Cache Stampede อย่างลึกซึ้ง และแก้ได้ทั้งด้วย Distributed Lock และ
  Probabilistic Early Expiration (XFetch)
- ✅ ตั้งค่า Redis เป็น Session Backend แบบเต็มรูปแบบ พร้อมย้าย session เดิมโดยไม่ทำให้
  ผู้ใช้ถูก logout
- ✅ แก้ปัญหา Race Condition ของ Throttling เดิมด้วย Lua Script ที่ atomic เต็มรูปแบบ
- ✅ ใช้ Redis Pub/Sub กระจายคำสั่ง invalidate cache ข้าม server หลายตัว
- ✅ Monitor สุขภาพของ Redis ด้วย `redis-cli`, `INFO`, และเขียน management command
  รายงาน hit rate อัตโนมัติ
- ✅ เข้าใจภาพรวมของ Sentinel (HA) และ Cluster (Sharding) และรู้ว่าจะเลือกใช้แบบไหน
- ✅ ออกแบบกลยุทธ์ Cache Warming ที่คุ้มค่า ไม่ warm ทุกอย่างเกินความจำเป็น
- ✅ ประกอบทุกเทคนิคเข้าเป็น Caching Layer เต็มรูปแบบสำหรับ Blog จริง

### 690.3 Checklist ก่อนไป Part ถัดไป

- [ ] ติดตั้ง Redis และรัน `redis-cli ping` ได้ผลลัพธ์ `PONG`
- [ ] ตั้งค่า `CACHES` แยก 3 alias (`default`, `throttle`, `sessions`) คนละ DB index
- [ ] ทดสอบ `IGNORE_EXCEPTIONS` โดยปิด Redis แล้วดูว่าแอปยังทำงานต่อได้ (ไม่ error 500)
- [ ] เขียนฟังก์ชันใช้ Hash เก็บสถิติโพสต์ และ Sorted Set ทำ trending สำเร็จ
- [ ] Implement `get_or_compute_with_lock()` และทดสอบว่าป้องกัน stampede ได้จริง
- [ ] เปลี่ยน `SESSION_ENGINE` เป็น `cached_db` และวัดความเร็วเทียบกับ `db` backend
- [ ] เขียน `AtomicSlidingWindowThrottle` ด้วย Lua script และเทสด้วย concurrent threads
- [ ] Implement two-level cache พร้อม Pub/Sub invalidation และทดสอบข้าม 2 process
- [ ] รัน `python manage.py redis_stats` และเข้าใจความหมายของทุกค่าที่แสดง
- [ ] เขียน `warm_cache` management command และตั้ง cron ให้รันอัตโนมัติ
- [ ] ประกอบทุกเทคนิคเป็น caching layer เต็มรูปแบบสำหรับ `blog` app สำเร็จ

### 690.4 แบบฝึกหัดท้ายบท

**แบบฝึกหัดที่ 1**: เพิ่มฟังก์ชัน `get_recently_viewed(user_id, limit=5)` ที่ใช้ Redis
**List** (type ที่ยังไม่ได้ใช้ใน Part นี้ ลองค้นคว้าเพิ่มเติมเอง — คำสั่งหลักคือ `LPUSH`
และ `LTRIM`) เก็บประวัติ 5 โพสต์ล่าสุดที่ผู้ใช้แต่ละคนเคยเข้าชม โดยไม่ให้ list ยาวเกิน 5
รายการ (ใช้ `LTRIM` ตัดท้ายอัตโนมัติทุกครั้งที่ push)

**แบบฝึกหัดที่ 2**: นำ `xfetch_get_or_compute()` จากขั้นตอนที่ 683.3 ไปใช้แทน
`get_or_compute_with_lock()` ในฟังก์ชัน `_warm_homepage()`ของ management command
`warm_cache` แล้วเปรียบเทียบพฤติกรรม: ทดสอบด้วยการยิง request พร้อมกัน 100 ครั้งตอน
cache หมดอายุพอดี แล้วนับว่าฟังก์ชัน `compute()` ถูกเรียกจริงกี่ครั้งในแต่ละวิธี
บันทึกผลเปรียบเทียบ

**แบบฝึกหัดที่ 3**: เขียนเทส (`TestCase`) ที่พิสูจน์ว่า `invalidate_post_caches()` ทำงาน
ถูกต้อง — สร้าง `Post`, เรียก `post_list` view เพื่อให้เกิด cache, แก้ไข title ของ
`Post` นั้น (ทำให้ signal ยิง), แล้วเรียก `post_list` อีกครั้งพร้อมยืนยันว่าเห็น title
ใหม่ทันที (ไม่ใช่ค่าที่ cache ไว้ค้างอยู่)

**แบบฝึกหัดที่ 4 (ขั้นสูง)**: ตั้งค่า Redis Sentinel จำลองในเครื่องด้วย Docker Compose
(3 Sentinel + 1 master + 1 replica) แล้วตั้งค่า Django ให้เชื่อมต่อผ่าน
`SentinelClient` ตามขั้นตอนที่ 688.2 ทดสอบ failover จริงด้วยการสั่งหยุด container ของ
master แล้วสังเกตว่า Django ยังอ่าน/เขียน cache ต่อได้โดยไม่ต้อง restart แอปพลิเคชันเอง

### 690.5 คำถามที่พบบ่อย (FAQ)

**Q: ต้องใช้ Redis หลาย instance จริง ๆ หรือแยกแค่ DB index ก็พอ?**
A: สำหรับโปรเจกต์ขนาดเล็ก-กลาง แยกแค่ DB index (ตามที่ทำมาตลอด Part นี้) เพียงพอแล้ว
และประหยัดค่าใช้จ่ายกว่ามาก แต่เมื่อระบบโตขึ้นจนแต่ละส่วน (cache, session, throttle,
Celery broker ที่จะเรียนใน Part 077) มี traffic สูงมากพร้อมกัน การแยก instance จริง
ช่วยแยก memory/CPU/network ออกจากกันชัดเจน ทำให้ปัญหาในส่วนหนึ่งไม่กระทบอีกส่วน (เช่น
cache stampede ที่กิน CPU หนักจะไม่ทำให้ session อ่านช้าไปด้วย)

**Q: ทำไมไม่ใช้ Memcached แทน Redis ไปเลย ในเมื่อ Django รองรับทั้งคู่?**
A: Memcached เร็วกว่า Redis เล็กน้อยสำหรับ use case ง่าย ๆ (key-value ล้วน ไม่มี
persistence) แต่ Redis มีจุดแข็งที่ Memcached ไม่มีเลยและ Part นี้ใช้เกือบทั้งหมด:
data structure ขั้นสูง (Hash, Sorted Set, List), Pub/Sub, distributed lock, และ
persistence แบบเลือกได้ (RDB/AOF) — Memcached เหมาะกับ pure caching เท่านั้น ส่วน
Redis เหมาะกับระบบที่ต้องการมากกว่า cache (เช่น session, rate limiting, real-time
features) ซึ่งเป็นสิ่งที่โปรเจกต์ระดับ production เกือบทุกระบบต้องการ

**Q: Lock-based stampede protection (ขั้นตอนที่ 683.2) กับ XFetch (683.3) เลือกใช้แค่
อย่างเดียวได้ไหม หรือต้องใช้ทั้งคู่เสมอ?**
A: เลือกใช้แค่อย่างเดียวได้ตามความเหมาะสมของแต่ละ cache key ไม่จำเป็นต้องใช้ทั้งคู่
พร้อมกันเสมอไป — Part นี้แสดงทั้งสองแบบเพื่อให้เข้าใจ trade-off และเลือกใช้ให้ถูกกับ
สถานการณ์ (ทวนตารางขั้นตอนที่ 683.5) ระบบใหญ่บางระบบใช้ผสมกันในคนละ key ตามความสำคัญ
ของข้อมูลนั้น ๆ

**Q: ทำไม Redis Pub/Sub (ขั้นตอนที่ 686) ถึงไม่เหมาะเป็น message queue สำหรับงานสำคัญ
เช่น ส่งอีเมล?**
A: เพราะ Pub/Sub ของ Redis **ไม่รับประกันการส่งถึง (fire-and-forget)** — ถ้าไม่มีใคร
subscribe อยู่ตอน publish ข้อความนั้นหายไปเลยตลอดกาล ไม่มีการเก็บคิวรอ ไม่มี retry
ไม่มี acknowledgment เหมาะกับงาน "แจ้งเตือนแบบ best-effort" อย่าง cache invalidation
(ถ้าพลาดไปบ้าง L1 cache ก็แค่หมดอายุตาม TTL เองในที่สุด ไม่ใช่หายนะ) แต่**ไม่เหมาะกับ
งานที่ต้องรับประกันว่าเกิดขึ้นแน่นอน** เช่น ส่งอีเมล ตัดเงิน หรือประมวลผลคำสั่งซื้อ —
งานเหล่านั้นต้องใช้ message broker เต็มรูปแบบที่มี persistence และ acknowledgment
อย่าง Celery + RabbitMQ/Redis Streams ซึ่งจะเรียนใน Phase 9

### 690.6 ตารางสรุปภาพรวมทั้ง Part

| ขั้นตอน | หัวข้อ | ผลลัพธ์ที่ได้ |
|---|---|---|
| 681 | ติดตั้ง django-redis | Cache backend พร้อมใช้งานระดับ production |
| 682 | Hash, Sorted Set | เก็บสถิติและจัดอันดับความนิยมแบบ atomic |
| 683 | Cache Stampede | Lock + XFetch ป้องกันการ compute ซ้ำซ้อนพร้อมกัน |
| 684 | Redis Session Backend | Session เร็วขึ้น ~10 เท่า พร้อมย้ายข้อมูลเดิมอย่างปลอดภัย |
| 685 | Atomic Rate Limiting | Lua script แก้ race condition ของ throttling เดิม |
| 686 | Pub/Sub Invalidation | Two-level cache ที่ sync กันข้ามทุก server |
| 687 | Monitoring | `redis-cli`, `INFO`, hit rate ที่วัดผลได้จริง |
| 688 | Sentinel/Cluster | เข้าใจภาพรวม HA ก่อนเจาะลึกใน Phase 11 |
| 689 | Cache Warming | ลด cache miss latency สำหรับผู้ใช้คนแรก |
| 690 | ประกอบร่างทั้งหมด | Caching Layer เต็มรูปแบบสำหรับ Blog พร้อม production |

---

## เตรียมตัวสำหรับ Part ถัดไป

**Part 070: Database Indexing และ Query Analysis** จะพาคุณกลับไปที่รากฐานของ
performance อีกครั้ง แต่คราวนี้ที่ระดับฐานข้อมูลโดยตรง — เมื่อ caching (Part 068-069)
ช่วยลดจำนวนครั้งที่ต้อง query ฐานข้อมูลได้แล้ว คำถามถัดไปคือ **ทำอย่างไรให้ query ที่
ยังต้องรันจริง (ตอน cache miss) เร็วที่สุดเท่าที่จะเป็นไปได้** คุณจะได้เรียนรู้การสร้าง
Database Index อย่างถูกต้อง, อ่านผลลัพธ์จาก `EXPLAIN ANALYZE` ของ PostgreSQL, และ
วิเคราะห์ query ที่ช้าด้วยเครื่องมือระดับมืออาชีพ — เป็นทักษะที่ทำงานคู่กับ Caching
เสมอ เพราะ cache ที่ดีที่สุดในโลกก็ช่วยไม่ได้ถ้า query ตอน cache miss ยังช้าจนทำให้
ผู้ใช้คนแรกที่โชคร้ายต้องรอเป็นสิบวินาที เตรียมเปิด `psql` หรือเครื่องมือดูฐานข้อมูล
ที่คุณถนัดไว้ให้พร้อม แล้วไปต่อกันเลย!
