# Part 074: Django Channels ขั้นสูง: Consumers และ Groups

> **ขั้นตอนที่ 731-740 ของหลักสูตร** | Phase 9: Async, Celery และ Channels
>
> Part 057 ปูพื้นฐาน Django Channels ไว้ครบวงจร: ติดตั้ง ASGI, เขียน Consumer
> ทั้งแบบ sync/async, ใช้ Channel Layer บน Redis กระจายข้อความข้าม connection,
> ผูก Signal เข้ากับ WebSocket เพื่อแจ้งเตือนแบบ real-time, ยืนยันตัวตนผู้ใช้
> และเขียนเทสต์ — แต่จงใจเก็บเรื่องที่ซับซ้อนกว่านั้นไว้ไม่แตะเลยตามที่ข้อ
> 567.5 และ 569.6 ของ Part 057 บอกไว้ว่า "จะเจาะลึกใน Part 074" นั่นคือ:
> การจัดการกลุ่มแบบไดนามิกเมื่อ Consumer ตัวเดียวต้องอยู่หลายกลุ่มพร้อมกัน,
> รูปแบบแชร์โค้ดระหว่าง Consumer หลายตัวด้วย Inheritance/Mixin, การตั้งค่า
> `channels_redis` ให้ทนโหลดจริงระดับ production, การจัดการ disconnect อย่าง
> สวยงามพร้อมกลยุทธ์ reconnect ฝั่ง client, ระบบ Presence Tracking, Rate
> Limiting ป้องกัน spam ผ่าน WebSocket, การ scale ข้ามหลายเครื่อง, และการ
> เปรียบเทียบ Channels กับทางเลือกอื่น ๆ ในตลาด Part นี้จะจบด้วยการประกอบ
> ทุกเทคนิคเข้าเป็นระบบแชทแบบ real-time เต็มรูปแบบพร้อม Presence Tracking
> ที่ใช้งานได้จริงในระดับมืออาชีพ

---

## สารบัญของ Part นี้

- ขั้นตอนที่ 731: ทบทวน Channels พื้นฐานสั้น ๆ แล้วเจาะลึก Group Management ขั้นสูง
- ขั้นตอนที่ 732: รูปแบบ Consumer Inheritance และ Mixin สำหรับแชร์ Logic ระหว่าง Consumer
- ขั้นตอนที่ 733: `channels_redis` ใน Production — Connection, Capacity, Expiry
- ขั้นตอนที่ 734: จัดการ Disconnect อย่างสวยงาม และกลยุทธ์ Reconnect ฝั่ง Client
- ขั้นตอนที่ 735: Presence Tracking — ระบบแสดงว่าใครออนไลน์อยู่บ้าง
- ขั้นตอนที่ 736: Rate Limiting ข้อความ WebSocket ป้องกัน Spam
- ขั้นตอนที่ 737: การ Scale Channels ข้ามหลาย Server ด้วย Shared Channel Layer
- ขั้นตอนที่ 738: เปรียบเทียบ Django Channels กับทางเลือกอื่น (Socket.IO, ASGI แยก)
- ขั้นตอนที่ 739: Deployment เจาะลึก — Process Model ของ Daphne/Uvicorn กับ Connection จำนวนมาก
- ขั้นตอนที่ 740: สรุปและแบบฝึกหัด — ระบบแชท Real-time เต็มรูปแบบพร้อม Presence Tracking

---

## ขั้นตอนที่ 731: ทบทวน Channels พื้นฐานสั้น ๆ แล้วเจาะลึก Group Management ขั้นสูง

### 731.1 ทบทวนกลไกหลักจาก Part 057 แบบรวบรัด

ก่อนเจาะลึก ขอทวนความจำ 3 กลไกหลักที่ Part 057 วางรากฐานไว้ เพราะทุกขั้นตอน
ใน Part นี้สร้างต่อยอดจากมันทั้งหมด:

| กลไก | หน้าที่ | สอนไว้ที่ |
|---|---|---|
| `Consumer` (`AsyncWebsocketConsumer`) | Object อายุยืนตลอด connection มี `connect()`/`receive()`/`disconnect()` | Part 057 ข้อ 563-564 |
| `Channel Layer` (Redis) | "กระดานประกาศ" กลางที่ Consumer หลายตัวคุยกันได้ผ่าน `group_add`/`group_send`/`group_discard` | Part 057 ข้อ 565 |
| `AuthMiddlewareStack` + `self.scope['user']` | เติมข้อมูลผู้ใช้ที่ล็อกอินให้ Consumer เข้าถึงได้ | Part 057 ข้อ 567 |

ตัวอย่างจาก Part 057 ทั้งหมด (Echo, ChatConsumer, NotificationConsumer) มี
ลักษณะร่วมกันอย่างหนึ่งที่ยังไม่ถูกท้าทาย: **แต่ละ Consumer เข้าร่วมแค่ 1
กลุ่มเท่านั้น** (`ChatConsumer` เข้าร่วม `chat_<room_name>` กลุ่มเดียว,
`NotificationConsumer` เข้าร่วม `notifications` กลุ่มเดียว) งานจริงแทบไม่มี
ทางง่ายขนาดนั้น — ผู้ใช้ 1 คนมักต้องรับทั้งข้อความห้องแชทที่ตัวเองอยู่ **และ**
การแจ้งเตือนส่วนตัว **และ** ประกาศทั่วทั้งระบบ พร้อมกันในเวลาเดียวกันผ่าน
WebSocket connection เส้นเดียว

### 731.2 ฟีเจอร์ที่ถูกมองข้าม: `groups` Class Attribute ของ Generic Consumer

Channels มีฟีเจอร์ในตัวที่ Part 057 ยังไม่ได้แนะนำ — คลาส
`WebsocketConsumer`/`AsyncWebsocketConsumer` มี attribute ชื่อ **`groups`**
(ค่า default เป็น `None`) ถ้ากำหนดเป็น list ของชื่อกลุ่ม Channels จะ
`group_add` ให้อัตโนมัติ**ก่อน**เรียก `connect()` ของเรา และ `group_discard`
ให้อัตโนมัติ**หลัง**เรียก `disconnect()` เสร็จ โดยไม่ต้องเขียน
`channel_layer.group_add()` เองเลย:

```python
# realtime/consumers.py
from channels.generic.websocket import AsyncWebsocketConsumer


class AnnouncementConsumer(AsyncWebsocketConsumer):
    """ตัวอย่างการใช้ groups แบบ static — ทุก connection เข้ากลุ่มเดียวกันเสมอ"""

    groups = ["announcements"]  # Channels จะ group_add/group_discard ให้อัตโนมัติ

    async def connect(self):
        await self.accept()

    async def announce(self, event):
        await self.send(text_data=event["message"])
```

โค้ดนี้ทำงานเหมือนกับที่ `NotificationConsumer` ของ Part 057 เขียนด้วยมือ
ทุกประการ แต่สั้นกว่า — เหมาะกับกรณีที่ **กลุ่มคงที่ รู้ชื่อล่วงหน้า และไม่
ต้องตรวจสิทธิ์ก่อนเข้าร่วม**

### 731.3 ข้อจำกัดสำคัญของ `groups`: ลำดับการทำงานที่ต้องระวัง

จุดที่ต้องเข้าใจให้ลึกก่อนนำไปใช้จริง คือ **ลำดับการทำงานภายในของ Channels**:
เมื่อ handshake เข้ามา Channels จะเรียก `group_add()` ให้ทุกกลุ่มใน
`self.groups` ก่อน แล้ว**ค่อย**เรียก `connect()` ของเรา นั่นหมายความว่า
**ถ้า `connect()` ปฏิเสธการเชื่อมต่อด้วย `self.close()` (แบบที่ Part 057
ข้อ 567.4 ทำกับ `NotificationConsumer`) connection นั้นได้เข้าร่วมกลุ่มไป
แล้วเรียบร้อยก่อนจะถูกปฏิเสธ!** แม้ `disconnect()` จะถูกเรียกตามมาเพื่อ
`group_discard` ออกให้ในที่สุด แต่ก็มีช่วงเวลาสั้น ๆ ที่ connection ที่ยัง
ไม่ผ่านการยืนยันตัวตนเป็นสมาชิกกลุ่มอยู่ ซึ่งเป็นความเสี่ยงที่ยอมรับไม่ได้
สำหรับกลุ่มที่มีข้อมูลอ่อนไหว

**กฎการเลือกใช้ของ Part นี้**:

| สถานการณ์ | ใช้ `groups` Class Attribute | ใช้ `channel_layer.group_add()` ด้วยมือ |
|---|---|---|
| กลุ่มสาธารณะ ไม่ต้องตรวจสิทธิ์ (เช่น "ประกาศระบบทั่วไป") | ✅ เหมาะมาก สั้นและปลอดภัย | ใช้ได้แต่เกินความจำเป็น |
| ต้องตรวจสิทธิ์ก่อนอนุญาตให้เข้ากลุ่ม (เช่น `NotificationConsumer` ของ Part 057) | ❌ อันตราย — เข้ากลุ่มก่อนตรวจสิทธิ์เสมอ | ✅ บังคับใช้แบบนี้เท่านั้น |
| ชื่อกลุ่มต้อง**คำนวณจาก URL/user** (ไดนามิก) | ได้ (ผ่าน `@property`, ข้อ 731.4) แต่ซับซ้อนกว่า | ✅ ตรงไปตรงมากว่า |
| Consumer เดียวต้องเข้าร่วม**หลายกลุ่มพร้อมกัน** | ได้ (คืน list หลายรายการ) | ✅ ยืดหยุ่นที่สุด โดยเฉพาะถ้าจำนวนกลุ่มไม่คงที่ |

### 731.4 ชื่อกลุ่มแบบไดนามิกด้วย `@property`

เพราะ `self.scope` ถูกตั้งค่าไว้ก่อนที่ Channels จะอ่านค่า `self.groups`
(scope ถูกกำหนดตอน dispatch ข้อความ `websocket.connect` เข้ามา) เราจึง
override `groups` เป็น **property** ที่คำนวณจาก URL kwargs ได้ ใช้ได้กับ
กรณีที่ไม่ต้องตรวจสิทธิ์ก่อน:

```python
# realtime/consumers.py
from channels.generic.websocket import AsyncWebsocketConsumer


class RoomBroadcastConsumer(AsyncWebsocketConsumer):
    """เข้าร่วมกลุ่มที่ชื่อคำนวณจาก URL โดยไม่ต้องเขียน connect() เอง"""

    @property
    def groups(self):
        room_name = self.scope["url_route"]["kwargs"]["room_name"]
        return [f"broadcast_{room_name}"]

    async def connect(self):
        await self.accept()

    async def room_message(self, event):
        await self.send(text_data=event["message"])
```

### 731.5 กรณีที่ซับซ้อนที่สุด: Consumer เดียวเข้าร่วมหลายกลุ่มแบบไดนามิก

นี่คือรูปแบบที่งานจริงต้องการบ่อยที่สุด และเป็นเหตุผลที่ Part 057 ตั้งใจ
เก็บไว้ให้ Part นี้: ผู้ใช้ที่เชื่อมต่อ WebSocket เส้นเดียว ต้องรับทั้ง (1)
ข้อความของห้องแชทที่ตนอยู่ (2) การแจ้งเตือนส่วนตัวที่ผูกกับ user ID ของตน
และ (3) ประกาศทั่วทั้งระบบ — สามกลุ่มที่**ไม่รู้จำนวนล่วงหน้า**และ**ต้อง
ตรวจสิทธิ์ก่อน**เข้าร่วม จึงต้องเขียนด้วยมือใน `connect()` ตามกฎข้อ 731.3:

```python
# realtime/consumers.py
import json

from channels.generic.websocket import AsyncWebsocketConsumer


class HubConsumer(AsyncWebsocketConsumer):
    """
    Consumer เดียวที่เป็น "ศูนย์กลาง" ของผู้ใช้คนหนึ่ง เข้าร่วมได้หลายกลุ่ม
    พร้อมกัน: กลุ่มส่วนตัว, กลุ่มห้องแชทที่เลือกเข้าระหว่างทาง, กลุ่มประกาศระบบ
    """

    async def connect(self):
        user = self.scope["user"]
        if not user.is_authenticated:
            await self.close(code=4001)
            return

        # เก็บชื่อกลุ่มทั้งหมดที่ instance นี้เข้าร่วมไว้ใน list ของตัวเอง
        # (ไม่ใช้ self.groups เพราะต้องตรวจสิทธิ์ก่อนตามกฎข้อ 731.3)
        self.joined_groups = set()

        # กลุ่มที่ 1: กลุ่มส่วนตัว ผูกกับ user.id เสมอ ไม่มีใครแอบเข้ามาฟังได้
        personal_group = f"user_{user.id}"
        await self.channel_layer.group_add(personal_group, self.channel_name)
        self.joined_groups.add(personal_group)

        # กลุ่มที่ 2: ประกาศทั่วทั้งระบบ — ทุกคนที่ authenticated เข้าได้เสมอ
        await self.channel_layer.group_add("system_announcements", self.channel_name)
        self.joined_groups.add("system_announcements")

        await self.accept()

    async def disconnect(self, close_code):
        # group_discard ทุกกลุ่มที่เคยเข้าร่วมไว้ ไม่ใช่แค่กลุ่มเดียว
        joined_groups = getattr(self, "joined_groups", set())
        for group_name in joined_groups:
            await self.channel_layer.group_discard(group_name, self.channel_name)

    async def receive(self, text_data=None, bytes_data=None):
        data = json.loads(text_data)
        action = data.get("action")

        if action == "join_room":
            room_name = data["room_name"]
            group_name = f"chat_{room_name}"
            if group_name not in self.joined_groups:
                await self.channel_layer.group_add(group_name, self.channel_name)
                self.joined_groups.add(group_name)
                await self.send(text_data=json.dumps({
                    "event": "joined_room", "room_name": room_name,
                }))

        elif action == "leave_room":
            room_name = data["room_name"]
            group_name = f"chat_{room_name}"
            if group_name in self.joined_groups:
                await self.channel_layer.group_discard(group_name, self.channel_name)
                self.joined_groups.discard(group_name)
                await self.send(text_data=json.dumps({
                    "event": "left_room", "room_name": room_name,
                }))

        elif action == "chat_message":
            room_name = data["room_name"]
            group_name = f"chat_{room_name}"
            if group_name not in self.joined_groups:
                return  # เพิกเฉยข้อความจากห้องที่ยังไม่ได้ join (ป้องกัน spoofing)
            await self.channel_layer.group_send(group_name, {
                "type": "chat_message",
                "room_name": room_name,
                "sender": user_display_name(self.scope["user"]),
                "message": data["message"],
            })

    async def chat_message(self, event):
        await self.send(text_data=json.dumps({
            "event": "chat_message",
            "room_name": event["room_name"],
            "sender": event["sender"],
            "message": event["message"],
        }))

    async def personal_notification(self, event):
        await self.send(text_data=json.dumps({
            "event": "notification", "payload": event["payload"],
        }))

    async def system_announcement(self, event):
        await self.send(text_data=json.dumps({
            "event": "announcement", "message": event["message"],
        }))


def user_display_name(user):
    return user.get_full_name() or user.username
```

จุดสำคัญของ `HubConsumer`: การเข้า/ออกห้องแชทเกิดขึ้น **ระหว่างที่ connection
เปิดค้างอยู่** ผ่านข้อความ `join_room`/`leave_room` ที่ client ส่งเข้ามา ไม่
ใช่แค่ตอน `connect()` ครั้งเดียวแบบ `ChatConsumer` ของ Part 057 — นี่คือ
ความแตกต่างสำคัญที่ทำให้รองรับ UI แบบ "สลับห้องแชทได้โดยไม่ต้องตัดการ
เชื่อมต่อ WebSocket ใหม่ทุกครั้ง" ได้จริง และ `self.joined_groups` ทำหน้าที่
เป็น "บัญชีรายชื่อ" ที่ทำให้ `disconnect()` รู้ว่าต้อง `group_discard` กี่กลุ่ม
โดยไม่ทิ้ง channel ที่ตายแล้วค้างอยู่ใน Redis (ทวนคำเตือนของ Part 057 ข้อ
565.4)

### 731.6 ข้อควรระวัง: Race Condition ตอนตรวจสอบ `joined_groups`

เพราะ `AsyncWebsocketConsumer` ประมวลผลข้อความ**ทีละข้อความตามลำดับที่มาถึง
เสมอ** (ไม่มีสอง `receive()` ทำงานพร้อมกันในอินสแตนซ์เดียวกัน) การเช็ค
`if group_name not in self.joined_groups` ในโค้ดข้างต้นจึงปลอดภัยจาก race
condition ภายใน instance เดียวกัน — แต่สิ่งที่ยังต้องระวังคือ **ชื่อกลุ่มชน
กันข้าม feature** เช่นถ้ามีคนตั้งชื่อห้องแชทว่า `announcements` จะไปชนกับ
`chat_announcements` โดยบังเอิญได้ถ้าลืม prefix — กฎเหล็กคือ **ตั้ง prefix
ให้ชื่อกลุ่มทุกประเภทต่างกันชัดเจนเสมอ** (`user_`, `chat_`, `system_` ตาม
ตัวอย่างข้างต้น) และควร validate รูปแบบชื่อห้อง (เช่น จำกัดเฉพาะตัวอักษร
ตัวเลข และขีดกลาง) ก่อนนำไปต่อ string เป็นชื่อกลุ่มเสมอ เพื่อป้องกันผู้ใช้
ตั้งชื่อห้องที่มีอักขระที่ Channel Layer ไม่รองรับ (ชื่อกลุ่มของ Channels
รองรับเฉพาะ ASCII ตัวอักษร ตัวเลข `-`, `.`, `_` เท่านั้น ตามสเปกของ ASGI)

---

## ขั้นตอนที่ 732: รูปแบบ Consumer Inheritance และ Mixin สำหรับแชร์ Logic ระหว่าง Consumer

### 732.1 ปัญหา: Logic ซ้ำกันระหว่าง Consumer หลายตัว

เมื่อโปรเจกต์มี Consumer มากขึ้นเรื่อย ๆ (`ChatConsumer`, `HubConsumer`,
`PresenceConsumer` ที่จะเขียนในขั้นตอนที่ 735) จะเริ่มเห็นโค้ดซ้ำ ๆ กันใน
ทุกตัว: การตรวจสอบ `user.is_authenticated`, การแปลง event เป็น JSON ก่อน
ส่ง, การจัดการ `joined_groups` เพื่อ `group_discard` ตอน `disconnect()` —
นี่คือสัญญาณคลาสสิกที่ควรดึงออกมาเป็น **Base Class** หรือ **Mixin** ตามหลัก
DRY ที่ Part 001 ข้อ 3.4 แนะนำไว้ตั้งแต่ต้นหลักสูตร

### 732.2 ใช้ `JsonWebsocketConsumer` ที่ Channels เตรียมไว้ให้แทนการ `json.loads`/`json.dumps` เอง

ก่อนเขียน base class ของตัวเอง ควรรู้จักคลาสสำเร็จรูปที่ Channels มีให้ก่อน:
`AsyncJsonWebsocketConsumer` ทำหน้าที่ encode/decode JSON ให้อัตโนมัติ ตัด
โค้ด `json.loads(text_data)`/`json.dumps(...)` ที่ซ้ำในทุก Consumer ของ
Part 057 และขั้นตอนที่ 731 ออกไปได้ทั้งหมด:

```python
# realtime/consumers.py
from channels.generic.websocket import AsyncJsonWebsocketConsumer


class EchoJsonConsumer(AsyncJsonWebsocketConsumer):
    """receive_json/send_json แทนที่ json.loads/json.dumps ที่ต้องเขียนเองทุกครั้ง"""

    async def connect(self):
        await self.accept()

    async def receive_json(self, content, **kwargs):
        # content คือ dict ที่ Channels decode ให้แล้ว ไม่ต้อง json.loads เอง
        await self.send_json({"echo": content})
```

### 732.3 สร้าง Base Class: `AuthenticatedGroupConsumer`

รวม logic ที่ซ้ำที่สุด 3 อย่างเข้าไว้ในคลาสเดียว: ตรวจสิทธิ์ก่อน `accept()`,
เก็บรายชื่อกลุ่มที่เข้าร่วม, และ `group_discard` ให้ครบตอน `disconnect()`
โดยลูกคลาสแค่ override hook method สั้น ๆ:

```python
# realtime/consumers.py
from channels.generic.websocket import AsyncJsonWebsocketConsumer


class AuthenticatedGroupConsumer(AsyncJsonWebsocketConsumer):
    """
    Base class กลางสำหรับ Consumer ที่ต้อง (1) authenticated ก่อนเสมอ
    (2) อาจเข้าร่วมได้หลายกลุ่มแบบไดนามิก (3) ต้อง group_discard ครบทุกกลุ่ม
    ตอนตัดการเชื่อมต่อ ลูกคลาสไม่ต้องเขียน boilerplate นี้ซ้ำอีกเลย
    """

    async def connect(self):
        self.joined_groups = set()
        user = self.scope["user"]

        if not user.is_authenticated:
            await self.close(code=4001)
            return

        allowed = await self.is_authorized(user)
        if not allowed:
            await self.close(code=4003)
            return

        await self.accept()
        await self.on_authorized_connect()

    async def disconnect(self, close_code):
        for group_name in getattr(self, "joined_groups", set()):
            await self.channel_layer.group_discard(group_name, self.channel_name)
        await self.on_disconnect(close_code)

    async def join_group(self, group_name):
        """Helper: ลูกคลาสเรียกอันนี้แทน channel_layer.group_add ตรง ๆ
        เพื่อให้ Base Class ติดตามกลุ่มที่เข้าร่วมไว้ให้อัตโนมัติ"""
        await self.channel_layer.group_add(group_name, self.channel_name)
        self.joined_groups.add(group_name)

    async def leave_group(self, group_name):
        await self.channel_layer.group_discard(group_name, self.channel_name)
        self.joined_groups.discard(group_name)

    # --- Hook methods: ลูกคลาส override เท่าที่จำเป็น ---

    async def is_authorized(self, user):
        """Override เพื่อเพิ่มเงื่อนไขสิทธิ์ (เช่น is_staff) ค่า default คือผ่านทุกคนที่ล็อกอิน"""
        return True

    async def on_authorized_connect(self):
        """Override เพื่อทำงานหลัง accept() สำเร็จ เช่น join_group เริ่มต้น"""

    async def on_disconnect(self, close_code):
        """Override เพื่อทำ cleanup เพิ่มเติมนอกเหนือจาก group_discard อัตโนมัติ"""
```

### 732.4 ใช้งาน Base Class จริงกับ Consumer 3 ตัวที่ต่างกัน

```python
# realtime/consumers.py (ต่อ)

class StaffNotificationConsumer(AuthenticatedGroupConsumer):
    """เทียบเท่า NotificationConsumer ของ Part 057 ข้อ 567.4 แต่สั้นลงมาก"""

    async def is_authorized(self, user):
        return user.is_staff

    async def on_authorized_connect(self):
        await self.join_group("notifications")

    async def notify(self, event):
        await self.send_json({key: value for key, value in event.items() if key != "type"})


class RoomChatConsumer(AuthenticatedGroupConsumer):
    """ห้องแชทที่ทุกคน authenticated เข้าได้ ต่อยอดจากขั้นตอนที่ 731"""

    async def on_authorized_connect(self):
        room_name = self.scope["url_route"]["kwargs"]["room_name"]
        self.room_group_name = f"chat_{room_name}"
        await self.join_group(self.room_group_name)

    async def receive_json(self, content, **kwargs):
        await self.channel_layer.group_send(self.room_group_name, {
            "type": "chat_message",
            "sender": self.scope["user"].username,
            "message": content["message"],
        })

    async def chat_message(self, event):
        await self.send_json({"sender": event["sender"], "message": event["message"]})


class OrgDashboardConsumer(AuthenticatedGroupConsumer):
    """แดชบอร์ดที่ต้องอยู่ทั้งกลุ่มองค์กรและกลุ่มส่วนตัวพร้อมกัน"""

    async def is_authorized(self, user):
        return hasattr(user, "profile") and user.profile.organization_id is not None

    async def on_authorized_connect(self):
        user = self.scope["user"]
        await self.join_group(f"org_{user.profile.organization_id}")
        await self.join_group(f"user_{user.id}")

    async def dashboard_update(self, event):
        await self.send_json({"event": "dashboard_update", "data": event["data"]})

    async def personal_notification(self, event):
        await self.send_json({"event": "notification", "data": event["data"]})
```

สังเกตว่าทั้งสามคลาสไม่มีการเขียน `if not user.is_authenticated`, การเก็บ
`joined_groups`, หรือ loop `group_discard` ซ้ำเลยแม้แต่บรรทัดเดียว — logic
ทั้งหมดอยู่ใน `AuthenticatedGroupConsumer` เพียงจุดเดียว ถ้าวันหนึ่งต้องแก้
กฎการตรวจสิทธิ์ (เช่น เพิ่ม logging ทุกครั้งที่ connection ถูกปฏิเสธ) แก้ที่
เดียวจบ ครบทุก Consumer ในระบบทันที — นี่คือหัวใจของ DRY ที่ Part 001 ข้อ
3.4 อธิบายไว้ นำมาใช้จริงกับ Consumer

### 732.5 Mixin แบบ Composition สำหรับ Logic ที่ไม่ได้ต้องการ Inheritance เชิงเดี่ยว

Inheritance (ข้อ 732.3-732.4) เหมาะกับ logic ที่ "ทุก Consumer ต้องมี" แต่
บาง logic เป็นแบบ **เลือกใช้เฉพาะบางตัว** (opt-in) เช่น Rate Limiting
(ขั้นตอนที่ 736) ที่ไม่ใช่ทุก Consumer ต้องการ กรณีนี้ **Mixin** เหมาะกว่า
เพราะรวมเข้ากับ base class อื่นได้อย่างอิสระผ่าน multiple inheritance ของ
Python:

```python
# realtime/mixins.py
from django.core.cache import cache


class RateLimitMixin:
    """Mixin แบบ opt-in — ผสมเข้ากับ Consumer ตัวไหนก็ได้ที่ต้องการจำกัดอัตราข้อความ
    ไม่ขึ้นกับ base class ใด ๆ เป็นพิเศษ (composition over inheritance)"""

    rate_limit_max_messages = 10
    rate_limit_window_seconds = 10

    async def is_rate_limited(self):
        from asgiref.sync import sync_to_async

        cache_key = f"ws_rate_limit:{self.scope['user'].id}:{self.__class__.__name__}"
        current_count = await sync_to_async(cache.get)(cache_key, 0)

        if current_count >= self.rate_limit_max_messages:
            return True

        await sync_to_async(cache.set)(
            cache_key, current_count + 1, timeout=self.rate_limit_window_seconds,
        )
        return False


class LoggingMixin:
    """Mixin อีกตัวที่เป็นอิสระจากกัน — บันทึก log ทุกครั้งที่ connect/disconnect"""

    async def log_connect(self):
        import logging
        logging.getLogger("realtime").info(
            "WebSocket connected: user=%s consumer=%s",
            self.scope["user"], self.__class__.__name__,
        )
```

```python
# realtime/consumers.py (ผสม Mixin เข้ากับ Base Class จาก 732.3)
class RoomChatConsumerV2(RateLimitMixin, LoggingMixin, AuthenticatedGroupConsumer):
    """ผสม 3 ความสามารถเข้าด้วยกัน: rate limit + logging + auth/group management
    โดยไม่มีคลาสไหนรู้จักกันโดยตรง — นี่คือพลังของ Mixin แบบ composition"""

    async def on_authorized_connect(self):
        await self.log_connect()
        room_name = self.scope["url_route"]["kwargs"]["room_name"]
        self.room_group_name = f"chat_{room_name}"
        await self.join_group(self.room_group_name)

    async def receive_json(self, content, **kwargs):
        if await self.is_rate_limited():
            await self.send_json({"error": "ส่งข้อความเร็วเกินไป กรุณารอสักครู่"})
            return
        await self.channel_layer.group_send(self.room_group_name, {
            "type": "chat_message",
            "sender": self.scope["user"].username,
            "message": content["message"],
        })

    async def chat_message(self, event):
        await self.send_json({"sender": event["sender"], "message": event["message"]})
```

### 732.6 ตารางเปรียบเทียบ: เมื่อไหร่ใช้ Inheritance vs Mixin

| ประเด็น | Base Class Inheritance (732.3) | Mixin แบบ Composition (732.5) |
|---|---|---|
| ใช้เมื่อ | Logic ที่ Consumer **ทุกตัวในกลุ่มเดียวกัน**ต้องมีเหมือนกันหมด | ความสามารถที่ **เลือกผสมเฉพาะบางตัว** ได้อิสระ |
| จำนวน parent class | เดี่ยว (single inheritance เป็นหลัก) | หลายตัวพร้อมกันได้ (multiple inheritance) |
| ความเสี่ยงเรื่อง MRO (Method Resolution Order) | ต่ำ (มักมี parent เดียว) | ต้องระวังลำดับที่ประกาศ (`class X(MixinA, MixinB, Base)`) เพราะ Python ค้นหา method ตามลำดับซ้ายไปขวา |
| ตัวอย่างในหลักสูตรนี้ | `AuthenticatedGroupConsumer` (ตรวจสิทธิ์ + จัดการกลุ่ม) | `RateLimitMixin`, `LoggingMixin` (opt-in ต่อ Consumer) |
| ข้อควรระวัง | ถ้า hierarchy ลึกเกินไป (Base → Sub → SubSub) จะตามโค้ดยาก | ถ้าผสม Mixin เยอะเกินไปจนอ่านไม่ออกว่า method ไหนมาจากไหน ให้พิจารณาแยกเป็น Base Class แทน |

**คำแนะนำระดับมืออาชีพ**: เริ่มจาก **ไม่มี** abstraction ใด ๆ เขียน
Consumer ตรง ๆ แบบ Part 057 ไปก่อน แล้วค่อยดึงเป็น Base Class/Mixin **เมื่อ
เห็นโค้ดซ้ำจริง ๆ ตั้งแต่ 3 ที่ขึ้นไป** (กฎ "Rule of Three" ที่ใช้กันทั่วไป
ในวงการ) การ abstraction ก่อนเวลาอันควร (premature abstraction) มักทำให้
โค้ดอ่านยากกว่าเดิมโดยไม่จำเป็น

---

## ขั้นตอนที่ 733: `channels_redis` ใน Production — Connection, Capacity, Expiry

### 733.1 ทวนการตั้งค่าเบื้องต้นจาก Part 057

Part 057 ข้อ 565.3 ตั้งค่า `CHANNEL_LAYERS` แบบง่ายที่สุดสำหรับ dev:

```python
# config/settings.py (เวอร์ชัน dev จาก Part 057)
CHANNEL_LAYERS = {
    "default": {
        "BACKEND": "channels_redis.core.RedisChannelLayer",
        "CONFIG": {"hosts": [("127.0.0.1", 6379)]},
    },
}
```

ค่านี้ใช้ได้ดีตอนพัฒนา แต่ production ที่มีผู้ใช้หลักพัน-หมื่นคนพร้อมกัน
ต้องปรับพารามิเตอร์เพิ่มเติมอีก 4 ตัวที่ Part 057 ยังไม่ได้พูดถึง

### 733.2 พารามิเตอร์ `capacity` — ป้องกัน Consumer ที่ "อ่านช้า" ทำ Redis บวม

`capacity` (ค่า default ของ `channels_redis` คือ **100**) คือจำนวนข้อความ
สูงสุดที่แต่ละ channel เดี่ยว ๆ (ไม่ใช่ group) เก็บสะสมรอส่งได้ก่อนที่ข้อความ
เก่าที่สุดจะถูกทิ้งไปเมื่อมีข้อความใหม่เข้ามาเกิน — ปัญหาที่เกิดถ้าไม่มีเพดาน
นี้: ถ้า Consumer ตัวหนึ่งประมวลผลช้า (เช่น ติด rate limit เยอะ หรือ event
loop คอขวด) แต่กลุ่มยังคง `group_send` เข้ามาเรื่อย ๆ ข้อความจะกอง
สะสมใน Redis list ของ channel นั้นแบบไม่จำกัด จนใช้ memory หมด

```python
# config/settings.py (เวอร์ชัน production)
CHANNEL_LAYERS = {
    "default": {
        "BACKEND": "channels_redis.core.RedisChannelLayer",
        "CONFIG": {
            "hosts": [{"address": "redis://redis-channels.internal:6379/0"}],
            "capacity": 1500,       # ปรับสูงขึ้นสำหรับกลุ่มที่มีการ broadcast ถี่ (เช่น ห้องแชทคนเยอะ)
            "channel_capacity": {
                "http.request": 200,
                r"^chat_": 3000,     # regex: ห้องแชทที่ traffic สูงได้ capacity มากกว่ากลุ่มทั่วไป
            },
        },
    },
}
```

`channel_capacity` คือ dict ที่ override ค่า `capacity` เป็นรายกลุ่ม/รายชื่อ
channel โดยใช้ regex เป็น key — มีประโยชน์มากเมื่อระบบมีทั้งกลุ่มที่ traffic
สูงมาก (ห้องแชทสาธารณะ) ปนกับกลุ่มที่ traffic ต่ำ (แจ้งเตือนส่วนตัว) และไม่
อยากตั้ง capacity สูงเท่ากันทั้งระบบ (สิ้นเปลือง memory โดยไม่จำเป็น)

### 733.3 พารามิเตอร์ `expiry` — TTL ของข้อความที่ยังไม่ถูกอ่าน

`expiry` (ค่า default **60 วินาที**) คือเวลาสูงสุดที่ข้อความหนึ่งจะรอให้
Consumer มาอ่านก่อนถูกทิ้งไปเงียบ ๆ โดยไม่มีการแจ้งเตือน — เหตุผลของ default
60 วินาทีคือ Channels ออกแบบมาโดยสมมติว่า Consumer ที่ยัง "มีชีวิตอยู่" ควร
ประมวลผลข้อความเร็วกว่านั้นมาก ถ้าข้อความค้างเกิน 1 นาทีแปลว่ามีบางอย่าง
ผิดปกติ (Consumer ค้าง, deadlock, หรือ process ตายไปแล้วแต่ Redis ยังไม่รู้)

```python
"CONFIG": {
    "hosts": [{"address": "redis://redis-channels.internal:6379/0"}],
    "capacity": 1500,
    "expiry": 10,  # งาน real-time ต้องการความสด ลด expiry ให้สั้นลงจาก default 60s
},
```

สำหรับงานที่ต้องการความสดมาก (เช่น แชท, live score) ควรลด `expiry` ให้สั้น
กว่า default เพราะ **ข้อความที่ค้างนานเกินไปแล้วเพิ่งมาถึงมักไม่มีประโยชน์
กับผู้ใช้แล้ว** (ข้อความแชทที่มาช้ากว่า 10 วินาทีสร้างความสับสนมากกว่าจะ
ไม่ส่งเลยด้วยซ้ำ) ตรงข้ามกับงานที่ยอมรับความหน่วงได้ (เช่น sync สถานะที่ไม่
เร่งด่วน) อาจเพิ่ม `expiry` ให้นานขึ้นได้

### 733.4 พารามิเตอร์ `group_expiry` — TTL ของสมาชิกภาพกลุ่ม

`group_expiry` (ค่า default **86400 วินาที = 1 วัน**) แยกจาก `expiry`
โดยสิ้นเชิง — นี่คือเวลาที่ **สมาชิกภาพของกลุ่ม** (ที่บันทึกไว้ตอน
`group_add`) จะหมดอายุถ้าไม่มีการ `group_add` ซ้ำ (Channels รีเฟรช TTL
อัตโนมัติทุกครั้งที่ `group_send` ไปที่กลุ่มนั้นสำเร็จ) ค่านี้เป็นตัวป้องกัน
ปัญหา "channel ผี" ที่ Part 057 ข้อ 565.4 เตือนไว้ว่าต้องเรียก
`group_discard` ใน `disconnect()` เสมอ — `group_expiry` คือ**เกราะป้องกัน
ชั้นที่สอง** สำหรับกรณีที่ `disconnect()` ไม่ได้ถูกเรียก (เช่น process
เซิร์ฟเวอร์ถูก kill กะทันหันโดยไม่มีโอกาส cleanup)

```python
"CONFIG": {
    ...
    "group_expiry": 3600,  # ลดจาก default 1 วัน เหลือ 1 ชั่วโมง สำหรับห้องแชทที่อายุสั้น
},
```

### 733.5 Connection และการกระจายโหลดข้าม Redis หลายตัว (Sharding)

`hosts` รับ **list ของ Redis instance ได้มากกว่า 1 ตัว** — `channels_redis`
จะใช้ consistent hashing กระจายชื่อ channel/group ไปยัง Redis แต่ละตัวโดย
อัตโนมัติ ทำให้ scale แนวนอนได้เมื่อ Redis instance เดียวเริ่มเป็นคอขวด:

```python
CHANNEL_LAYERS = {
    "default": {
        "BACKEND": "channels_redis.core.RedisChannelLayer",
        "CONFIG": {
            "hosts": [
                {"address": "redis://redis-channels-1.internal:6379/0"},
                {"address": "redis://redis-channels-2.internal:6379/0"},
                {"address": "redis://redis-channels-3.internal:6379/0"},
            ],
            "capacity": 1500,
            "expiry": 10,
        },
    },
}
```

**ข้อควรระวัง**: consistent hashing หมายความว่า "กลุ่มเดียวกันจะไปอยู่ที่
Redis instance เดียวกันเสมอ" — ถ้า Redis instance หนึ่งใน list ล่ม
**เฉพาะกลุ่มที่แม็ปไปตัวนั้น**เท่านั้นที่ได้รับผลกระทบ ไม่ใช่ทั้งระบบ (ต่างจาก
`CACHES` ที่ Part 069 อธิบายว่าถ้า Redis ตัวเดียวล่มกระทบทั้งระบบ) นี่คือ
เหตุผลที่การใช้หลาย Redis host ช่วยลด **blast radius** ได้ในระดับหนึ่ง แม้จะ
ไม่ได้ทดแทนการทำ High Availability (Redis Sentinel/Cluster) อย่างแท้จริง

### 733.6 `RedisChannelLayer` vs `RedisPubSubChannelLayer`

`channels_redis` ให้ backend สำรองอีกตัวที่เหมาะกับรูปแบบการใช้งานต่างกัน:

| ประเด็น | `channels_redis.core.RedisChannelLayer` (ค่า default) | `channels_redis.pubsub.RedisPubSubChannelLayer` |
|---|---|---|
| กลไกภายใน | Redis List + `BRPOP` (แต่ละ channel คือ list หนึ่งอัน) | Redis Pub/Sub (`SUBSCRIBE`/`PUBLISH`) |
| เหมาะกับจำนวนกลุ่มที่มีสมาชิกเยอะต่อกลุ่ม (fan-out สูง) | ปานกลาง — ยิ่งกลุ่มใหญ่ ยิ่งต้องเขียนข้อความลง list ของทุก channel สมาชิก | ดีกว่ามากสำหรับ fan-out สูง เพราะ Pub/Sub กระจายในตัว ไม่ต้อง loop เขียนทีละ channel |
| รองรับ `capacity`/`expiry` ต่อข้อความ | ✅ รองรับเต็มรูปแบบ | ⚠️ จำกัดกว่า (Pub/Sub ไม่มีแนวคิดเรื่องข้อความค้างในคิว) |
| Persistence กรณี Consumer หลุดชั่วคราว | ข้อความยังรออยู่ใน list จนกว่าจะ `expiry` | ข้อความที่พลาดตอนหลุดการเชื่อมต่อ **หายไปเลยทันที** (ธรรมชาติของ Pub/Sub) |
| ความนิยม/ความเสถียรของเอกสาร | เป็น default ที่ใช้กันแพร่หลายที่สุด | ใหม่กว่า เอกสารและตัวอย่างน้อยกว่า |

**คำแนะนำ**: เริ่มต้นด้วย `RedisChannelLayer` (default) เสมอสำหรับโปรเจกต์
ส่วนใหญ่รวมถึงหลักสูตรนี้ทั้งหมด และพิจารณาสลับไป `RedisPubSubChannelLayer`
ก็ต่อเมื่อวัดผลจริงแล้วพบว่ากลุ่มที่มีสมาชิกจำนวนมาก (หลักพันคนต่อกลุ่ม เช่น
live event ที่มีคนดูพร้อมกันเยอะ) เป็นคอขวดของระบบจริง ๆ — อย่าสลับไปใช้
ล่วงหน้าโดยไม่มีข้อมูลรองรับ (ทวนหลักการ "อย่า optimize ก่อนวัดผล" จาก
Part 072)

### 733.7 คำสั่งตรวจสุขภาพ Redis Channel Layer

```bash
# ดูจำนวน connection ที่ Redis กำลังให้บริการอยู่ (สังเกตว่าไม่เกิน maxclients)
redis-cli -h redis-channels.internal INFO clients

# ดูการใช้ memory โดยรวม — capacity ที่ตั้งสูงเกินไปจะเห็นผลตรงนี้ชัดเจน
redis-cli -h redis-channels.internal INFO memory

# นับจำนวน key ทั้งหมด (channel/group ที่ยัง active อยู่)
redis-cli -h redis-channels.internal DBSIZE

# หา key ที่ใหญ่ผิดปกติ (มักบ่งบอกว่ามี channel ที่ไม่มีคนอ่านสะสมข้อความ)
redis-cli -h redis-channels.internal --bigkeys
```

> **คำเตือน**: ห้ามรัน `redis-cli MONITOR` บน Redis instance ของ production
> ที่มี traffic สูง เพราะมันพิมพ์ทุกคำสั่งที่ Redis ประมวลผลออกมาแบบ real-time
> ซึ่งกิน CPU ของ Redis เองอย่างมีนัยสำคัญ ใช้เพื่อ debug ระยะสั้นบน staging
> เท่านั้น

---

## ขั้นตอนที่ 734: จัดการ Disconnect อย่างสวยงาม และกลยุทธ์ Reconnect ฝั่ง Client

### 734.1 `disconnect()` ถูกเรียกเมื่อไหร่บ้าง — ไม่ใช่แค่ตอน Client ปิดเอง

`disconnect()` ของทุก Consuemr ที่เขียนมาตั้งแต่ Part 057 ถูกเรียกใน**หลาย
สถานการณ์**ที่มือใหม่มักนึกไม่ถึงทั้งหมด:

| สถานการณ์ | `close_code` ที่ได้รับ | สาเหตุ |
|---|---|---|
| Client ปิดแท็บ/เบราว์เซอร์ตามปกติ | `1000` | ปิดแบบปกติ (normal closure) |
| Client เรียก `socket.close()` เอง | `1000` (หรือค่าที่ client กำหนดเอง) | ปิดแบบตั้งใจ |
| อินเทอร์เน็ตหลุดกะทันหัน (ไม่มีโอกาสส่ง close frame) | `1006` | Abnormal closure — พบบ่อยที่สุดในโลกจริงบนมือถือ |
| Consumer เรียก `self.close(code=...)` เอง (เช่น ปฏิเสธสิทธิ์) | ค่าที่กำหนดเอง (4001, 4003 ตามตัวอย่างขั้นตอนที่ 731-732) | Server ปฏิเสธ |
| Server รีสตาร์ท/deploy ใหม่ระหว่างที่ connection เปิดอยู่ | `1001` (Going Away) หรือ `1012`/`1013` ถ้า server ส่งก่อนปิด | Deployment (เจาะลึกในขั้นตอนที่ 739) |
| เกิด Exception ที่ไม่ได้ดักใน `receive()`/method ของกลุ่ม | `1011` (Internal Error) — Channels ปิดให้อัตโนมัติ | Bug ในโค้ด Consumer |

**กฎเหล็ก**: `disconnect()` ต้อง **ทำงานถูกต้องเหมือนกันทุกกรณีข้างต้น**
เพราะเราไม่มีทางรู้ล่วงหน้าว่า connection จะถูกปิดด้วยเหตุผลไหน — โค้ด
cleanup (`group_discard`, ปิด presence ในขั้นตอนที่ 735) ต้องไม่พึ่งพา
สมมติฐานว่า "client จะปิดแบบสุภาพเสมอ"

### 734.2 การดักจับ Exception ใน `receive()` โดยไม่ทำให้ Connection พังทั้งเส้น

จุดที่ Part 057 ยังไม่ได้พูดถึงคือ **ถ้า `receive()` เกิด exception ที่ไม่
ได้ดักไว้ Channels จะปิด connection ทันทีด้วย close code `1011`** ซึ่งมัก
รุนแรงเกินไปสำหรับ error เล็ก ๆ เช่น client ส่ง JSON ที่ format ผิด:

```python
# realtime/consumers.py
import json

from channels.generic.websocket import AsyncWebsocketConsumer


class RobustChatConsumer(AsyncWebsocketConsumer):
    async def connect(self):
        self.room_group_name = f"chat_{self.scope['url_route']['kwargs']['room_name']}"
        await self.channel_layer.group_add(self.room_group_name, self.channel_name)
        await self.accept()

    async def disconnect(self, close_code):
        await self.channel_layer.group_discard(self.room_group_name, self.channel_name)

    async def receive(self, text_data=None, bytes_data=None):
        try:
            data = json.loads(text_data)
        except (TypeError, json.JSONDecodeError):
            # ผิดพลาดจาก client ฝั่งเดียว ไม่ควรทำให้ connection ทั้งเส้นถูกตัด
            await self.send(text_data=json.dumps({"error": "รูปแบบข้อความไม่ถูกต้อง"}))
            return

        message = data.get("message", "").strip()
        if not message:
            await self.send(text_data=json.dumps({"error": "ข้อความห้ามว่างเปล่า"}))
            return

        await self.channel_layer.group_send(self.room_group_name, {
            "type": "chat_message", "message": message,
        })

    async def chat_message(self, event):
        await self.send(text_data=json.dumps({"message": event["message"]}))
```

หลักการคือ: **แยก error ที่เกิดจาก input ของ client ผิดพลาด (ควรตอบกลับ
ด้วยข้อความ error แล้วรอ input ถัดไปต่อ) ออกจาก error ที่เป็นบั๊กจริงของ
เซิร์ฟเวอร์ (ควรปล่อยให้ exception หลุดขึ้นไปเพื่อให้ logging/monitoring
เห็นและแจ้งเตือนทีมพัฒนา)** — การ `try/except` ครอบคลุมทุกอย่างแบบ
`except Exception: pass` เป็นแนวทางที่ผิด เพราะจะซ่อนบั๊กจริงไว้เงียบ ๆ

### 734.3 ปิด Connection แบบตั้งใจด้วย `StopConsumer`

สำหรับกรณีที่ต้องการหยุดการทำงานของ Consumer จาก**ภายนอก** method ปกติ
(เช่น จาก background task ที่ตรวจพบว่า token ของผู้ใช้หมดอายุระหว่างทาง)
Channels มี exception พิเศษชื่อ `StopConsumer`:

```python
# realtime/consumers.py
from channels.exceptions import StopConsumer


class TokenAwareConsumer(AuthenticatedGroupConsumer):
    async def force_logout(self, event):
        """ถูกเรียกผ่าน group_send เมื่อระบบต้องการบังคับตัดการเชื่อมต่อ (เช่น admin สั่ง ban)"""
        await self.send_json({"event": "force_logout", "reason": event.get("reason", "")})
        raise StopConsumer()
```

`raise StopConsumer()` สั่งให้ Channels ปิด connection และหยุด dispatch
loop อย่างสะอาด (ยังคงเรียก `disconnect()` ให้ cleanup ตามปกติ) — ต่างจาก
การเรียก `self.close()` ตรงตรงตรงที่ `StopConsumer` ใช้ได้แม้อยู่ใน handler
ของกลุ่ม (`force_logout` ที่ถูกเรียกจาก `group_send`) ซึ่งไม่มี `return` ที่
จะ "จบ" การเชื่อมต่อแบบ `connect()` ปกติ

### 734.4 กลยุทธ์ Reconnect ฝั่ง Client: Exponential Backoff + Jitter

Part 057 ข้อ 570.1 แนะนำ exponential backoff แบบง่าย (`reconnectDelay * 2`)
ไปแล้ว — Part นี้ปรับปรุงให้สมบูรณ์ขึ้นด้วย 3 เรื่องที่ยังขาด: **jitter**
(สุ่มเวลาเพิ่มเติมเล็กน้อยเพื่อไม่ให้ client หลายพันตัว reconnect
พร้อมกันเป๊ะจนถล่ม server หลัง deploy — ปรากฏการณ์ที่เรียกว่า **thundering
herd**), **เพดานจำนวนครั้งสูงสุด** ก่อนจะแจ้งผู้ใช้ว่ามีปัญหาจริง ๆ, และ
**การแยกแยะ close code** เพื่อไม่พยายาม reconnect กรณีที่ server ปฏิเสธ
สิทธิ์อย่างชัดเจน (reconnect ไปก็จะถูกปฏิเสธซ้ำเหมือนเดิม):

```javascript
// static/js/websocket-client.js
class ResilientWebSocket {
    constructor(url, { maxDelay = 30000, baseDelay = 1000, maxAttempts = null } = {}) {
        this.url = url;
        this.baseDelay = baseDelay;
        this.maxDelay = maxDelay;
        this.maxAttempts = maxAttempts;  // null = ไม่จำกัด
        this.attempt = 0;
        this.socket = null;
        this.intentionallyClosed = false;
        this.onmessage = () => {};
        this.onopen = () => {};
        this.onpermanentfailure = () => {};
    }

    connect() {
        this.socket = new WebSocket(this.url);

        this.socket.onopen = (event) => {
            this.attempt = 0;  // เชื่อมต่อสำเร็จ รีเซ็ตตัวนับความพยายาม
            this.onopen(event);
        };

        this.socket.onmessage = (event) => this.onmessage(event);

        this.socket.onclose = (event) => {
            if (this.intentionallyClosed) return;  // ผู้ใช้ตั้งใจปิดเอง ไม่ต้อง reconnect

            // close code ในช่วง 4000-4009 ถูกกำหนดไว้ในระบบนี้ว่าเป็น "ปฏิเสธถาวร"
            // (เช่น ไม่ authenticated, ไม่มีสิทธิ์) reconnect ไปก็ถูกปฏิเสธซ้ำเหมือนเดิม
            const PERMANENT_REJECTION_CODES = [4001, 4003];
            if (PERMANENT_REJECTION_CODES.includes(event.code)) {
                this.onpermanentfailure(event);
                return;
            }

            if (this.maxAttempts !== null && this.attempt >= this.maxAttempts) {
                this.onpermanentfailure(event);
                return;
            }

            this.attempt += 1;
            const exponential = Math.min(this.baseDelay * 2 ** this.attempt, this.maxDelay);
            // Jitter แบบ "full jitter": สุ่มค่าระหว่าง 0 ถึงเวลาที่คำนวณได้ทั้งหมด
            // ป้องกัน client หลายพันตัวยิง reconnect พร้อมกันเป๊ะหลัง server กลับมา
            const delay = Math.random() * exponential;

            setTimeout(() => this.connect(), delay);
        };
    }

    send(data) {
        if (this.socket && this.socket.readyState === WebSocket.OPEN) {
            this.socket.send(data);
        }
    }

    close() {
        this.intentionallyClosed = true;
        if (this.socket) this.socket.close(1000);
    }
}
```

ใช้งาน:

```javascript
const client = new ResilientWebSocket('wss://example.com/ws/notifications/', {
    maxAttempts: 10,
});

client.onopen = () => console.log('เชื่อมต่อสำเร็จ');
client.onmessage = (event) => console.log('ได้รับ:', JSON.parse(event.data));
client.onpermanentfailure = (event) => {
    console.error('ไม่สามารถเชื่อมต่อได้ (code:', event.code, ') หยุดพยายามแล้ว');
    // แสดง UI แจ้งผู้ใช้ให้รีเฟรชหน้าด้วยตัวเอง
};

client.connect();
```

### 734.5 Heartbeat/Ping-Pong: ตรวจจับ Connection ที่ "ค้างแบบเงียบ" (Half-Open)

ปัญหาที่ backoff อย่างเดียวแก้ไม่ได้: บาง network (โดยเฉพาะ proxy/firewall
บางประเภท หรือมือถือที่สลับเครือข่าย WiFi↔4G) ตัด TCP connection แบบเงียบ ๆ
โดยไม่ส่ง close frame ใด ๆ มาให้ทั้งสองฝั่งรู้เลย — เบราว์เซอร์ยังคิดว่า
`socket.readyState === WebSocket.OPEN` อยู่ ทั้งที่จริงข้อมูลส่งไม่ถึงกันแล้ว
เรียกสถานะนี้ว่า **half-open connection** ทางแก้คือ **Heartbeat**: ทั้งสอง
ฝั่งส่งข้อความ ping/pong เป็นระยะเพื่อยืนยันว่ายังมีชีวิตอยู่จริง

```python
# realtime/consumers.py
import asyncio
import json

from channels.generic.websocket import AsyncWebsocketConsumer


class HeartbeatConsumer(AsyncWebsocketConsumer):
    HEARTBEAT_INTERVAL = 30    # ส่ง ping ทุก 30 วินาที
    HEARTBEAT_TIMEOUT = 10     # รอ pong ไม่เกิน 10 วินาทีหลังส่ง ping

    async def connect(self):
        await self.accept()
        self._last_pong = asyncio.get_event_loop().time()
        self._heartbeat_task = asyncio.create_task(self._heartbeat_loop())

    async def disconnect(self, close_code):
        self._heartbeat_task.cancel()  # ต้องยกเลิก background task เสมอ ไม่เช่นนั้นค้างในหน่วยความจำ

    async def receive(self, text_data=None, bytes_data=None):
        data = json.loads(text_data)
        if data.get("type") == "pong":
            self._last_pong = asyncio.get_event_loop().time()
            return
        # ... จัดการข้อความประเภทอื่นตามปกติต่อจากนี้

    async def _heartbeat_loop(self):
        try:
            while True:
                await asyncio.sleep(self.HEARTBEAT_INTERVAL)
                await self.send(text_data=json.dumps({"type": "ping"}))

                await asyncio.sleep(self.HEARTBEAT_TIMEOUT)
                elapsed = asyncio.get_event_loop().time() - self._last_pong
                if elapsed > self.HEARTBEAT_INTERVAL + self.HEARTBEAT_TIMEOUT:
                    # ไม่ได้รับ pong ทันเวลา — connection น่าจะ half-open แล้ว ปิดทิ้งเพื่อบังคับ
                    # ให้ client reconnect ใหม่ (ซึ่งจะได้ TCP connection ที่ใช้งานได้จริง)
                    await self.close(code=1001)
                    return
        except asyncio.CancelledError:
            pass  # ถูกยกเลิกจาก disconnect() ตามปกติ ไม่ใช่ error
```

ฝั่ง client ตอบ `pong` กลับทุกครั้งที่ได้รับ `ping`:

```javascript
client.onmessage = (event) => {
    const data = JSON.parse(event.data);
    if (data.type === 'ping') {
        client.send(JSON.stringify({ type: 'pong' }));
        return;
    }
    // จัดการข้อความประเภทอื่นตามปกติ
};
```

> **หมายเหตุ**: WebSocket protocol มี ping/pong frame ระดับ protocol อยู่
> แล้ว (ไม่ใช่ text message) แต่ Daphne/เบราว์เซอร์จัดการเรื่องนี้ให้บาง
> ส่วนโดยอัตโนมัติในระดับ TCP keep-alive ซึ่งมักช้าเกินไปสำหรับตรวจจับ
> half-open ในงาน real-time ที่ต้องการรู้ผลไวภายในหลักสิบวินาที การทำ
> heartbeat ระดับ application (ข้อความ JSON แบบข้างต้น) จึงเป็นแนวทางที่
> ควบคุมได้แม่นยำกว่าและเป็นที่นิยมมากกว่าในทางปฏิบัติ

---

## ขั้นตอนที่ 735: Presence Tracking — ระบบแสดงว่าใครออนไลน์อยู่บ้าง

### 735.1 โจทย์: "ใครกำลังออนไลน์อยู่ในห้องนี้บ้าง"

Chat Consumer ทุกตัวที่เขียนมาจนถึงตอนนี้ (Part 057 และขั้นตอนที่ 731-734)
กระจาย**ข้อความ**ระหว่างสมาชิกในกลุ่มได้ดีแล้ว แต่ยังไม่มีใครรู้ **"ตอนนี้มี
ใครอยู่ในห้องบ้าง"** — Channel Layer ของ `channels_redis` **ไม่มี API ให้
ถามว่ากลุ่มหนึ่งมีสมาชิกกี่คน**โดยตรง (`group_add`/`group_send` เป็น
fire-and-forget ทั้งคู่) เราจึงต้องสร้างกลไก presence ของตัวเองแยกต่างหาก
โดยใช้ Redis ตรง ๆ ผ่าน `django-redis` ที่ Part 069 ติดตั้งไว้แล้ว

### 735.2 ทำไมใช้ Set ธรรมดาไม่พอ: ปัญหาผู้ใช้เปิดหลายแท็บ/หลายอุปกรณ์

วิธีคิดแรกที่มักผิดพลาด: เก็บ user ID ใน Redis Set (`SADD room:general
user_42`) แล้วนับด้วย `SCARD` — ปัญหาคือถ้าผู้ใช้คนเดียวเปิด 2 แท็บพร้อมกัน
(หรือเปิดทั้งมือถือและคอมพิวเตอร์) จะได้ 2 connection ที่แยกกันอิสระ แต่
`SADD` กับ user ID ตัวเดียวจะเพิ่ม element ซ้ำไม่ได้ (set ไม่มี duplicate)
ทำให้เมื่อปิดแท็บใดแท็บหนึ่ง (`SREM`) ระบบคิดว่าผู้ใช้ออฟไลน์ทั้งที่ยังเปิด
อีกแท็บอยู่ — ทางแก้คือใช้ **Redis Hash เก็บตัวนับจำนวน connection ต่อผู้ใช้**
แทน Set ตรง ๆ

### 735.3 ออกแบบโครงสร้างข้อมูลใน Redis

```
Key: presence:room:general           (Redis Hash)
  Field: user_42  →  Value: 2   (user_42 เปิดอยู่ 2 connection พร้อมกัน)
  Field: user_17  →  Value: 1

เมื่อ user_42 เปิดแท็บที่ 3: HINCRBY presence:room:general user_42 1  → กลายเป็น 3
เมื่อ user_42 ปิดแท็บหนึ่ง:   HINCRBY presence:room:general user_42 -1 → กลายเป็น 2
เมื่อค่ากลายเป็น 0:           HDEL presence:room:general user_42 (และประกาศว่าออฟไลน์)
```

### 735.4 เขียน `PresenceConsumer`

```python
# realtime/presence.py
from asgiref.sync import sync_to_async
from django_redis import get_redis_connection


def _redis():
    """ใช้ connection ของ django-redis ที่ตั้งค่าไว้แล้วใน CACHES ตั้งแต่ Part 069
    แทนที่จะเปิด connection ใหม่แยกต่างหาก เพื่อใช้ connection pool ร่วมกัน"""
    return get_redis_connection("default")


async def mark_online(room_name, user_id):
    """เพิ่มตัวนับ connection ของ user ในห้องนี้ คืนค่า True ถ้านี่คือ connection แรก
    (แปลว่าผู้ใช้เพิ่งออนไลน์ ควรประกาศให้คนอื่นในห้องรู้)"""
    key = f"presence:room:{room_name}"

    def _incr():
        conn = _redis()
        new_count = conn.hincrby(key, user_id, 1)
        conn.expire(key, 3600)  # เผื่อไว้กรณี process ตายไปทั้งกลุ่มโดยไม่ cleanup เลย
        return new_count

    new_count = await sync_to_async(_incr)()
    return new_count == 1  # True แปลว่าเพิ่ง "เข้ามาใหม่" ไม่ใช่แค่เปิดแท็บเพิ่ม


async def mark_offline(room_name, user_id):
    """ลดตัวนับ cืนค่า True ถ้า user ออกจากห้องนี้หมดทุก connection แล้วจริง ๆ"""
    key = f"presence:room:{room_name}"

    def _decr():
        conn = _redis()
        new_count = conn.hincrby(key, user_id, -1)
        if new_count <= 0:
            conn.hdel(key, user_id)
            return True
        return False

    return await sync_to_async(_decr)()


async def get_online_user_ids(room_name):
    """คืนรายชื่อ user ID ทั้งหมดที่ออนไลน์อยู่ในห้องนี้ ณ ขณะนี้ (สำหรับผู้ที่เพิ่งเข้าห้อง)"""
    key = f"presence:room:{room_name}"

    def _members():
        conn = _redis()
        return [key.decode() for key in conn.hkeys(key)]

    return await sync_to_async(_members)()
```

```python
# realtime/consumers.py
import json

from channels.generic.websocket import AsyncWebsocketConsumer

from . import presence


class PresenceChatConsumer(AsyncWebsocketConsumer):
    async def connect(self):
        user = self.scope["user"]
        if not user.is_authenticated:
            await self.close(code=4001)
            return

        self.room_name = self.scope["url_route"]["kwargs"]["room_name"]
        self.room_group_name = f"chat_{self.room_name}"
        self.user_id = str(user.id)

        await self.channel_layer.group_add(self.room_group_name, self.channel_name)
        await self.accept()

        # ส่งรายชื่อคนออนไลน์ปัจจุบันให้ผู้ที่เพิ่งเข้ามาก่อน (initial state)
        online_ids = await presence.get_online_user_ids(self.room_name)
        await self.send(text_data=json.dumps({
            "event": "presence_snapshot", "online_user_ids": online_ids,
        }))

        # เพิ่มตัวนับ และประกาศให้คนอื่นรู้ "เฉพาะตอนที่เป็น connection แรกของ user นี้"
        is_newly_online = await presence.mark_online(self.room_name, self.user_id)
        if is_newly_online:
            await self.channel_layer.group_send(self.room_group_name, {
                "type": "presence_update", "user_id": self.user_id, "status": "online",
            })

    async def disconnect(self, close_code):
        await self.channel_layer.group_discard(self.room_group_name, self.channel_name)

        went_offline = await presence.mark_offline(self.room_name, self.user_id)
        if went_offline:
            await self.channel_layer.group_send(self.room_group_name, {
                "type": "presence_update", "user_id": self.user_id, "status": "offline",
            })

    async def receive(self, text_data=None, bytes_data=None):
        data = json.loads(text_data)
        await self.channel_layer.group_send(self.room_group_name, {
            "type": "chat_message",
            "sender_id": self.user_id,
            "message": data["message"],
        })

    async def chat_message(self, event):
        await self.send(text_data=json.dumps({
            "event": "chat_message", "sender_id": event["sender_id"], "message": event["message"],
        }))

    async def presence_update(self, event):
        await self.send(text_data=json.dumps({
            "event": "presence_update", "user_id": event["user_id"], "status": event["status"],
        }))
```

### 735.5 เหตุใดต้อง `conn.expire(key, 3600)` ทุกครั้งที่มีคน `mark_online`

บรรทัด `conn.expire(key, 3600)` ในข้อ 735.4 คือเกราะป้องกันชั้นสุดท้าย
สำหรับสถานการณ์ที่ **ทั้ง process เซิร์ฟเวอร์ตายกะทันหันโดยไม่มีโอกาสเรียก
`disconnect()` เลยแม้แต่ตัวเดียว** (เช่น เซิร์ฟเวอร์ไฟดับ, OOM killer ฆ่า
process ทิ้ง) กรณีแบบนี้ตัวนับใน Redis Hash จะค้างเป็นเลขที่ผิดไปตลอดกาล
ถ้าไม่มี TTL คอยล้างให้ — การตั้ง TTL ใหม่ทุกครั้งที่มีการ `mark_online`
ทำให้ห้องที่ยังมีคน active อยู่จริงจะไม่มีวันหมดอายุ (TTL ถูกต่ออายุตลอด)
แต่ห้องที่ไม่มีใครใช้งานแล้วจริง ๆ (รวมถึงกรณี process ตายไปเงียบ ๆ) จะถูก
ล้างข้อมูลทิ้งไปเองภายใน 1 ชั่วโมง ไม่ค้างเป็นขยะถาวรใน Redis

### 735.6 ตารางสรุปการออกแบบ Presence Tracking

| ประเด็น | การออกแบบที่ผิด (Set ธรรมดา) | การออกแบบที่ถูก (Hash + Counter, ข้อ 735.3) |
|---|---|---|
| โครงสร้างข้อมูล | `SADD room:general user_42` | `HINCRBY presence:room:general user_42 1` |
| ผู้ใช้เปิดหลายแท็บ/อุปกรณ์ | ปิดแท็บเดียวถูกนับว่าออฟไลน์ทั้งที่ยังเปิดแท็บอื่นอยู่ | นับถูกต้อง — ออฟไลน์จริงเมื่อตัวนับเป็น 0 เท่านั้น |
| ป้องกันข้อมูลค้างถ้า Server ตายกะทันหัน | ไม่มี (ต้องเขียน cleanup job แยก) | มี TTL ต่ออายุอัตโนมัติทุกครั้งที่ active |
| Broadcast แจ้งเข้า/ออกห้อง | ทุกครั้งที่ connect/disconnect (spam ถ้าเปิดหลายแท็บ) | เฉพาะตอนตัวนับเปลี่ยนจาก 0→1 หรือ 1→0 เท่านั้น (แม่นยำ) |

---

## ขั้นตอนที่ 736: Rate Limiting ข้อความ WebSocket ป้องกัน Spam

### 736.1 ทำไม WebSocket เสี่ยงต่อ Spam มากกว่า HTTP ธรรมดา

Part 048 สอนเรื่อง API Throttling ของ DRF ไปแล้วสำหรับ HTTP request — แต่
WebSocket มีความเสี่ยงเพิ่มเติมที่ HTTP ไม่มี: **connection เดียวเปิดค้างไว้
สามารถส่งข้อความรัว ๆ นับพันข้อความต่อวินาทีได้โดยไม่ต้องเปิด connection
ใหม่เลยสักครั้ง** (ต่างจาก HTTP ที่แต่ละ request มีต้นทุนเปิด/ปิด TCP ในตัว
เอง ซึ่งเป็นแรงเสียดทานตามธรรมชาติอยู่แล้ว) ถ้าไม่มี rate limiting ผู้ใช้
คนเดียว (หรือ bot) ที่เขียนสคริปต์ยิงข้อความรัว ๆ เข้ามาที่ `receive()` จะ
ทำให้ CPU ของ event loop หมดไปกับการประมวลผล spam จนกระทบผู้ใช้คนอื่นทั้ง
ระบบ (เพราะ event loop เดียวรับผิดชอบหลาย connection พร้อมกันตามที่ Part
057 ข้อ 564.1 อธิบายไว้)

### 736.2 เปรียบเทียบ 3 อัลกอริทึม Rate Limiting

| อัลกอริทึม | หลักการ | ข้อดี | ข้อเสีย |
|---|---|---|---|
| **Fixed Window** | นับจำนวนข้อความในช่วงเวลาคงที่ (เช่น ทุกนาฬิกาวินาที่ 0-59) แล้วรีเซ็ตเป็น 0 | เขียนง่ายที่สุด (ใช้ `INCR`+`EXPIRE` ของ Redis ได้ตรง ๆ) | มีช่องโหว่ตรงขอบหน้าต่าง — ผู้ใช้ส่ง 2 เท่าของ limit ได้ถ้าจับจังหวะช่วงรอยต่อพอดี |
| **Sliding Window Log** | เก็บ timestamp ของทุกข้อความไว้ใน Sorted Set แล้วนับเฉพาะที่อยู่ใน N วินาทีล่าสุด | แม่นยำที่สุด ไม่มีช่องโหว่ตรงขอบ | ใช้ memory มากกว่า (เก็บ timestamp ทุกอัน) |
| **Token Bucket** | มี "โทเค็น" เติมเข้ากระเป๋าเป็นอัตราคงที่ ส่งข้อความ 1 ครั้งใช้ 1 โทเค็น หมดแล้วต้องรอเติม | รองรับ burst สั้น ๆ ได้ตามธรรมชาติ (มีโทเค็นสะสมไว้ก่อนได้) | ซับซ้อนกว่าในการ implement ให้ถูกต้อง |

Part นี้เลือกสอน **Sliding Window Log** เพราะแม่นยำที่สุดและ implement
ด้วย Redis Sorted Set ได้ไม่ซับซ้อนมาก เหมาะกับ WebSocket ที่ต้องการความ
แม่นยำสูง (spam 1-2 ข้อความเกินก็สร้างความรำคาญให้คนอื่นในห้องแชทได้จริง)

### 736.3 Implement Sliding Window ด้วย Redis Sorted Set

```python
# realtime/ratelimit.py
import time
import uuid

from asgiref.sync import sync_to_async
from django_redis import get_redis_connection


async def is_rate_limited(key, max_messages, window_seconds):
    """
    Sliding Window Log ด้วย Redis Sorted Set:
    - score ของแต่ละ member คือ timestamp ตอนที่ส่งข้อความ
    - ลบสมาชิกที่เก่ากว่า window_seconds ออกก่อนนับทุกครั้ง (ZREMRANGEBYSCORE)
    - ถ้านับได้ >= max_messages แปลว่าโดน rate limit แล้ว ไม่เพิ่มสมาชิกใหม่
    """

    def _check():
        conn = get_redis_connection("default")
        now = time.time()
        window_start = now - window_seconds

        pipe = conn.pipeline()
        pipe.zremrangebyscore(key, 0, window_start)   # ล้างข้อความเก่าที่พ้น window แล้ว
        pipe.zcard(key)                                 # นับจำนวนที่เหลือ (ยังอยู่ใน window)
        _, current_count = pipe.execute()

        if current_count >= max_messages:
            conn.expire(key, window_seconds)
            return True

        # unique member ด้วย uuid เพราะ Sorted Set ต้องการ member ที่ไม่ซ้ำกัน
        # (ถ้าส่ง 2 ข้อความใน timestamp วินาทีเดียวกันเป๊ะ ต้องนับแยกกัน ไม่ใช่ merge)
        conn.zadd(key, {f"{now}:{uuid.uuid4().hex}": now})
        conn.expire(key, window_seconds)
        return False

    return await sync_to_async(_check)()
```

### 736.4 ผสาน Rate Limiting เข้ากับ Consumer พร้อมนโยบายลงโทษที่เข้มขึ้นเรื่อย ๆ

```python
# realtime/consumers.py
import json

from channels.generic.websocket import AsyncWebsocketConsumer

from . import ratelimit


class ModeratedChatConsumer(AsyncWebsocketConsumer):
    MAX_MESSAGES = 10
    WINDOW_SECONDS = 10
    MAX_VIOLATIONS_BEFORE_KICK = 3

    async def connect(self):
        user = self.scope["user"]
        if not user.is_authenticated:
            await self.close(code=4001)
            return

        self.room_name = self.scope["url_route"]["kwargs"]["room_name"]
        self.room_group_name = f"chat_{self.room_name}"
        self.rate_limit_key = f"ratelimit:chat:{user.id}"
        self.violation_count = 0

        await self.channel_layer.group_add(self.room_group_name, self.channel_name)
        await self.accept()

    async def disconnect(self, close_code):
        await self.channel_layer.group_discard(self.room_group_name, self.channel_name)

    async def receive(self, text_data=None, bytes_data=None):
        limited = await ratelimit.is_rate_limited(
            self.rate_limit_key, self.MAX_MESSAGES, self.WINDOW_SECONDS,
        )

        if limited:
            self.violation_count += 1
            await self.send(text_data=json.dumps({
                "error": "ส่งข้อความเร็วเกินไป กรุณารอสักครู่",
                "violation_count": self.violation_count,
            }))

            # ผู้ใช้ที่โดน rate limit ซ้ำ ๆ หลายครั้งติดกัน มักเป็น bot/spam จริง
            # ไม่ใช่แค่พิมพ์เร็ว — ตัดการเชื่อมต่อทิ้งไปเลยเพื่อป้องกันห้องแชท
            if self.violation_count >= self.MAX_VIOLATIONS_BEFORE_KICK:
                await self.close(code=4029)  # 4029 กำหนดเองให้สื่อถึง "429 Too Many Requests"
            return

        data = json.loads(text_data)
        await self.channel_layer.group_send(self.room_group_name, {
            "type": "chat_message",
            "sender": self.scope["user"].username,
            "message": data["message"],
        })

    async def chat_message(self, event):
        await self.send(text_data=json.dumps({"sender": event["sender"], "message": event["message"]}))
```

### 736.5 ระวัง: Rate Limit ต้องผูกกับ User ไม่ใช่กับ Connection

ข้อผิดพลาดที่พบบ่อย: เก็บตัวนับ rate limit เป็น **instance attribute ของ
Consumer** (เช่น `self.message_count += 1` ธรรมดาไม่ผ่าน Redis) วิธีนี้ใช้
ไม่ได้ผลจริงเพราะ **ผู้ใช้เปิด connection ใหม่ (เช่น รีเฟรชหน้า หรือเปิด
แท็บใหม่) ก็จะได้ instance ใหม่ที่ตัวนับเริ่มจาก 0 ใหม่ทันที** ทำให้ rate
limit ที่ผูกกับ instance ถูกหลบเลี่ยงได้ง่าย ๆ แค่รีเฟรชหน้า — การเก็บใน
Redis ตามข้อ 736.3 ผูกกับ **user ID** (ไม่ใช่ connection/channel name)
ทำให้ตัวนับคงอยู่ข้าม connection ได้อย่างถูกต้อง และยังทำงานถูกต้องแม้
deploy ด้วยหลาย process/หลายเครื่องพร้อมกัน เพราะทุก process แชร์ Redis
instance เดียวกัน (ทวนหลักการเดียวกับ `CACHES` ที่ Part 069 อธิบายไว้)

---

## ขั้นตอนที่ 737: การ Scale Channels ข้ามหลาย Server ด้วย Shared Channel Layer

### 737.1 ทวนสิ่งที่ Part 057 ข้อ 569.4 เกริ่นไว้

Part 057 อธิบายสั้น ๆ ว่า "ทุก process ต้องชี้ไปที่ Redis instance เดียวกัน"
— Part นี้จะอธิบายว่า**ทำไมกลไกนี้ถึงทำให้ scale ข้ามหลายเครื่องได้โดยไม่
ต้องพึ่ง Sticky Session** ซึ่งเป็นข้อจำกัดที่ระบบ real-time แบบเดิม (ก่อนมี
message broker กลาง) มักเจอ

### 737.2 ทำไม "Sticky Session" ถึงไม่จำเป็นสำหรับ Channels

**Sticky Session** คือเทคนิคที่ Load Balancer บังคับให้ผู้ใช้คนเดียวกัน
เชื่อมต่อไปยัง server ตัวเดียวกันเสมอ (ผ่าน cookie หรือ IP hash) — ระบบที่
เก็บ state ไว้ใน memory ของ process เดียว (เช่น Socket.IO แบบ default โดย
ไม่มี Redis adapter) **จำเป็นต้องใช้ Sticky Session** เพราะถ้าผู้ใช้ถูกส่ง
ไปคนละ server กับที่เก็บ connection ของตัวเองไว้ ระบบจะหาผู้ใช้คนนั้นไม่เจอ
เลย

Django Channels **ไม่มีข้อจำกัดนี้** เพราะสถาปัตยกรรมของมันแยก 2 เรื่องออก
จากกันอย่างชัดเจนตั้งแต่ต้น:

```
                    ┌─────────────────────────────────────────┐
                    │         Redis (Channel Layer กลาง)         │
                    │   เก็บ: group membership + message queue   │
                    └─────────────────────────────────────────┘
                       ▲                    ▲                    ▲
                       │ group_add/         │ group_add/         │ group_send
                       │ group_send         │ group_send         │ (จาก process ไหนก็ได้)
                       │                    │                    │
              ┌────────┴───────┐   ┌────────┴───────┐   ┌────────┴───────┐
              │  Server A       │   │  Server B       │   │  Server C       │
              │  (Daphne #1)    │   │  (Daphne #2)    │   │  (Daphne #3)    │
              │  ผู้ใช้ X ต่ออยู่  │   │  ผู้ใช้ Y ต่ออยู่  │   │  ผู้ใช้ Z ต่ออยู่  │
              └────────┬────────┘   └────────┬────────┘   └────────┬────────┘
                       │                    │                    │
                   WebSocket             WebSocket             WebSocket
                       │                    │                    │
                  Browser ของ X        Browser ของ Y        Browser ของ Z
```

เมื่อ Server A ต้องการส่งข้อความหาผู้ใช้ Y (ที่ TCP connection จริงอยู่ที่
Server B) **Server A ไม่จำเป็นต้องรู้เลยว่า Y อยู่ที่ไหน** — มันแค่เรียก
`channel_layer.group_send("room_x", ...)` เข้า Redis กลาง แล้ว **Server B
เอง**ที่กำลัง `BRPOP` (ทวนกลไกจากข้อ 733.6) รอฟังข้อความของ channel ที่ตน
ดูแลอยู่ จะเป็นฝ่ายดึงข้อความนั้นออกมาส่งต่อให้ browser ของ Y เอง — **ไม่มี
process ไหนต้องรู้จัก process อื่นโดยตรงเลยแม้แต่น้อย** ทุกอย่างคุยผ่าน
Redis เป็นตัวกลางเท่านั้น นี่คือเหตุผลที่ Load Balancer จ่าย connection
ใหม่แบบ **round-robin ธรรมดา** ไปยัง server ไหนก็ได้โดยไม่ต้องสนใจว่าผู้ใช้
คนนั้นเคยต่อกับ server ไหนมาก่อน

### 737.3 ข้อแม้ที่สำคัญ: ต้องมี Redis ตัวเดียวกัน (หรือ Cluster เดียวกัน) เท่านั้น

สถาปัตยกรรมข้อ 737.2 ทำงานได้ก็ต่อเมื่อ **`CHANNEL_LAYERS["default"]["CONFIG"]["hosts"]`
ของทุก server ชี้ไปที่ Redis instance/cluster ชุดเดียวกันเป๊ะ** — ถ้าทีม
deploy ผิดพลาดตั้ง Redis คนละตัวให้ Server A กับ Server B (เช่น ลืมอัปเดต
environment variable ตอน deploy ใหม่บาง instance) ผลลัพธ์คือระบบจะทำงาน
"บางครั้งได้ บางครั้งไม่ได้" แบบที่ Part 057 ข้อ 570.5 (FAQ ข้อสุดท้าย)
เตือนไว้ — บั๊กประเภทนี้ตามหาสาเหตุยากมากเพราะไม่มี error message ชัดเจน
ออกมาเลย มีแค่ "การแจ้งเตือนบางอันไม่ถึงบางคน"

### 737.4 High Availability ด้วย Redis Sentinel

Redis instance เดียวเป็น **single point of failure** — ถ้า Redis ตัวนั้น
ล่ม ทั้งระบบ real-time ทั้งหมดหยุดทำงานทันที (แม้ Django server ทุกตัวจะยัง
ทำงานปกติ) การใช้งานจริงระดับ production จึงมักใช้ **Redis Sentinel** เพื่อ
สลับไป replica อัตโนมัติเมื่อ primary ล่ม:

```python
# config/settings.py
CHANNEL_LAYERS = {
    "default": {
        "BACKEND": "channels_redis.core.RedisChannelLayer",
        "CONFIG": {
            "hosts": [
                {
                    "sentinels": [
                        ("sentinel-1.internal", 26379),
                        ("sentinel-2.internal", 26379),
                        ("sentinel-3.internal", 26379),
                    ],
                    "master_name": "channels-master",
                },
            ],
            "capacity": 1500,
            "expiry": 10,
        },
    },
}
```

Sentinel คอยตรวจสุขภาพ Redis primary/replica อยู่เบื้องหลัง ถ้า primary
ล่ม จะเลือก replica ตัวหนึ่งเลื่อนขึ้นเป็น primary ใหม่อัตโนมัติ และ
`channels_redis` จะถาม Sentinel ทุกครั้งว่า "ตอนนี้ใครคือ master" แทนที่จะ
ต่อ Redis instance ตัวใดตัวหนึ่งตรง ๆ — เรื่อง Redis Sentinel/Cluster แบบ
เต็มรูปแบบสำหรับทั้งระบบ (ไม่ใช่แค่ Channel Layer) จะเจาะลึกใน Part 087
(Docker Compose + Infrastructure) และ Part 090+ (High Availability)

### 737.5 การวางแผนกำลังต่อ Server (Capacity Planning) เบื้องต้น

| ปัจจัย | ผลกระทบต่อจำนวน Connection ที่ 1 Server รับได้ |
|---|---|
| Memory ต่อ WebSocket connection ที่ idle | ประมาณ 20-50 KB ต่อ connection (ขึ้นกับขนาด scope, buffer) — เครื่อง 4 GB RAM รองรับได้หลักหมื่น connection idle ในทางทฤษฎี |
| จำนวน CPU core | มีผลต่อจำนวน **worker process** ที่รันพร้อมกันได้ ไม่ใช่จำนวน connection ต่อ process โดยตรง (รายละเอียดในขั้นตอนที่ 739) |
| Redis throughput | ถ้า `group_send` ถี่มาก (เช่น ห้องแชทคนเยอะพิมพ์พร้อมกันตลอดเวลา) Redis เองอาจกลายเป็นคอขวดก่อน Django server เสียอีก |
| ulimit (จำนวน file descriptor สูงสุดของ OS) | ค่า default ของ Linux มักต่ำเกินไป (1024) ต้องปรับเพิ่มก่อน scale ให้รองรับ connection จำนวนมาก (รายละเอียดขั้นตอนที่ 739.5) |

---

## ขั้นตอนที่ 738: เปรียบเทียบ Django Channels กับทางเลือกอื่น

### 738.1 ทำไมต้องรู้จักทางเลือกอื่น ทั้งที่หลักสูตรนี้เลือก Channels แล้ว

มืออาชีพต้องเลือกเครื่องมือให้เหมาะกับงาน ไม่ใช่ใช้เครื่องมือเดียวกับทุก
โปรเจกต์เพราะ "เคยเรียนมา" — Part นี้จะให้ภาพที่เป็นกลางว่า Channels เหมาะ
กับสถานการณ์ไหน และเมื่อไหร่ที่ทางเลือกอื่นอาจเหมาะกว่า

### 738.2 ตารางเปรียบเทียบ Django Channels vs Socket.IO vs ASGI แยกต่างหาก

| ประเด็น | **Django Channels** (ที่หลักสูตรนี้ใช้) | **python-socketio** (บน ASGI) | **ASGI Service แยกต่างหาก** (เช่น FastAPI/Starlette เฉพาะ WebSocket) |
|---|---|---|---|
| เข้าถึง Django ORM/Auth/Session ได้ตรง ๆ | ✅ ได้ทันที (`scope['user']`, `database_sync_to_async`) | ⚠️ ได้ แต่ต้อง mount ร่วมกับ Django เอง ไม่ integrate ให้อัตโนมัติ | ❌ ต้องเรียกผ่าน API/message queue แยก (เพิ่ม latency และความซับซ้อน) |
| Protocol ที่ใช้ | WebSocket ดิบ (ควบคุมได้เต็มที่) | Protocol ของตัวเอง (บน WebSocket + fallback HTTP long-polling อัตโนมัติ) | ขึ้นกับที่เลือก เช่น WebSocket ดิบเหมือนกัน |
| รองรับ Browser/Network ที่ WebSocket ใช้ไม่ได้ (firewall องค์กรบางที่บล็อก) | ❌ ไม่มี fallback ในตัว | ✅ Fallback เป็น HTTP long-polling อัตโนมัติ | ❌ ไม่มี (ต้องเขียนเอง) |
| Ecosystem ฝั่ง Frontend (client library พร้อมใช้) | ต้องเขียน `WebSocket` API ดิบเอง (หรือใช้ library เสริมอย่าง reconnecting-websocket) | Client library สมบูรณ์มาก (`socket.io-client`) ใช้กันแพร่หลายทั้ง Web/Mobile | ขึ้นกับที่เลือกเขียนเอง |
| ความซับซ้อนในการ deploy | ปานกลาง (ต้องมี ASGI server + Redis) | ใกล้เคียง Channels (ต้องมี Redis adapter ด้วยถ้า multi-process) | สูงกว่า — ต้อง deploy เป็น service แยก + จัดการการสื่อสารข้าม service |
| เหมาะกับทีมที่ | ทีมที่ใช้ Django อยู่แล้วทั้งระบบ ต้องการ integrate กับ ORM/Auth แน่นแฟ้น | ทีมที่ต้องรองรับ client หลากหลายแพลตฟอร์ม (มือถือ, IoT) ที่มี ecosystem ของ Socket.IO อยู่แล้ว | ทีมที่แยก realtime service เป็น microservice เฉพาะ ต้องการ scale อิสระจาก Django app หลัก |
| ประสิทธิภาพดิบสำหรับ connection จำนวนมหาศาล (แสนขึ้นไป) | ดี (จำกัดตาม ASGI server) | ใกล้เคียงกัน (ขึ้นกับ implementation) | อาจดีกว่าเล็กน้อยถ้าเขียนด้วย framework ที่เบากว่า Django (แต่ต้องแลกกับการสูญเสีย integration) |

### 738.3 กรณีศึกษา: เมื่อไหร่ควรแยก Realtime เป็น ASGI Service ต่างหาก

แม้หลักสูตรนี้จะสอนแบบ "Channels รวมอยู่ใน Django project เดียวกัน" (แบบที่
Part 057 ทำมาตลอด) แต่ในองค์กรขนาดใหญ่ที่ WebSocket traffic เติบโตเร็วกว่า
HTTP traffic มาก บางทีมเลือก **แยก WebSocket ออกเป็น service ต่างหาก**
ด้วยเหตุผล:

- **Scale อิสระจากกัน**: HTTP traffic (view, API) อาจต้องการ CPU สำหรับ
  ประมวลผล query หนัก ในขณะที่ WebSocket traffic ต้องการแค่ connection
  slot จำนวนมากแต่ CPU ต่ำ — การแยก service ทำให้ scale แต่ละส่วนตามความ
  ต้องการจริงได้ ไม่ต้อง scale ทั้งคู่พร้อมกันเสมอ
- **Deploy แยกกัน**: การ deploy โค้ด HTTP ใหม่ไม่กระทบ WebSocket connection
  ที่เปิดค้างอยู่ (และในทางกลับกัน)
- **ความเสี่ยงด้าน Blast Radius**: ถ้า WebSocket service มีบั๊กจน crash
  ทั้งหมด ระบบ HTTP หลักยังทำงานได้ปกติ ไม่ล่มไปด้วยกัน

**แต่ข้อเสียที่ต้องยอมรับ**: ต้องมีกลไกสื่อสารข้าม service (เช่น เขียน
event ผ่าน message queue อย่าง Celery/RabbitMQ ที่ Part 075-077 จะสอน แทน
การเรียก Django signal ตรง ๆ แบบ Part 057 ข้อ 566.2) ซึ่งเพิ่มความซับซ้อน
และ latency เล็กน้อย — **หลักสูตรนี้แนะนำให้เริ่มด้วยสถาปัตยกรรมรวมใน Django
project เดียว (แบบ Part 057) เสมอ** และพิจารณาแยก service ก็ต่อเมื่อวัดผล
แล้วพบคอขวดจริง ๆ (ทวนหลักการเดียวกับข้อ 733.6 — อย่า optimize ก่อนมีข้อมูล)

### 738.4 ตัวอย่าง: Mount ASGI App อื่นควบคู่กับ Channels ใน `ProtocolTypeRouter` เดียวกัน

สำหรับทีมที่อยากทดลองใช้ library ภายนอก (เช่น `python-socketio`) ควบคู่กับ
Django Channels ในโปรเจกต์เดียวกันโดยไม่ต้องแยก service เต็มรูปแบบ
`ProtocolTypeRouter` ของ Part 057 ข้อ 562.6 รองรับการผสมได้อยู่แล้วเพราะ
มันคือ ASGI router ธรรมดา ไม่ได้ผูกติดกับ Channels เท่านั้น:

```python
# config/asgi.py (ตัวอย่างแนวคิดการผสม — ไม่ใช่การตั้งค่าเริ่มต้นของหลักสูตรนี้)
import os

from django.core.asgi import get_asgi_application

os.environ.setdefault("DJANGO_SETTINGS_MODULE", "config.settings")
django_asgi_app = get_asgi_application()

from channels.auth import AuthMiddlewareStack               # noqa: E402
from channels.routing import ProtocolTypeRouter, URLRouter   # noqa: E402

import realtime.routing  # noqa: E402

application = ProtocolTypeRouter({
    "http": django_asgi_app,
    "websocket": AuthMiddlewareStack(
        URLRouter(realtime.routing.websocket_urlpatterns)
    ),
    # หมายเหตุ: ProtocolTypeRouter แยกเส้นทางตาม "ชนิด protocol" เท่านั้น
    # (http/websocket) ถ้าต้องการแยกเส้นทาง websocket บางส่วนไปอีก ASGI app
    # (เช่น python-socketio) ต้องเขียน URLRouter/routing ของตัวเองมาคั่นกลาง
    # ก่อนถึง Channels URLRouter อีกที ซึ่งเป็นรายละเอียดที่ซับซ้อนเกินขอบเขต
    # ของ Part นี้ — ประเด็นสำคัญคือ "ทำได้" เพราะทุกอย่างพูดภาษา ASGI เดียวกัน
})
```

---

## ขั้นตอนที่ 739: Deployment เจาะลึก — Process Model ของ Daphne/Uvicorn กับ Connection จำนวนมาก

### 739.1 ทวนพื้นฐานจาก Part 057 ข้อ 569 แล้วเจาะลึกกว่าเดิม

Part 057 สอนแค่ "ใช้ Daphne/Uvicorn แทน Gunicorn sync worker" — Part นี้
จะอธิบาย **ว่าทำไม 1 process ของ ASGI server ถึงรองรับ connection ค้างไว้
ได้เป็นพันเป็นหมื่นพร้อมกัน** ทั้งที่เครื่องมีแค่ไม่กี่ CPU core

### 739.2 Event Loop เดียว จัดการ Connection นับพันได้อย่างไร

หัวใจสำคัญที่ Part 057 ข้อ 564.1 เกริ่นไว้แล้วขยายความในที่นี้: **1 process
ของ Uvicorn/Daphne รันบน 1 event loop** (ของ `asyncio` หรือ Twisted reactor
สำหรับ Daphne) — event loop ทำงานแบบ **single-threaded** แต่จัดการหลาย
connection พร้อมกันได้เพราะ **connection ที่ idle (รอข้อมูลอยู่) ไม่ใช้
CPU เลย** มันแค่ "จอง slot" ไว้ในหน่วยความจำเฉย ๆ (`await` ที่ยังไม่มีอะไร
ให้ทำจะคืน control กลับให้ event loop ไปทำงานอื่นก่อนเสมอ) เมื่อมีข้อมูล
เข้ามาจริง (client ส่งข้อความ) event loop จะปลุก coroutine ที่เกี่ยวข้อง
ขึ้นมาประมวลผลแค่ตอนนั้น แล้วกลับไป idle ต่อ

```
Event Loop 1 ตัว (1 CPU core ใช้งานจริงตอนประมวลผล)
┌──────────────────────────────────────────────────────────────┐
│  Connection A (idle, รอ await)     ← ไม่ใช้ CPU ตอนนี้            │
│  Connection B (idle, รอ await)     ← ไม่ใช้ CPU ตอนนี้            │
│  Connection C (มีข้อความเข้า!)      ← event loop ประมวลผลตอนนี้    │
│  Connection D (idle, รอ await)     ← ไม่ใช้ CPU ตอนนี้            │
│  ... (อาจมีอีกหลายพัน connection ที่ idle พร้อมกัน)                │
└──────────────────────────────────────────────────────────────┘

ข้อจำกัดที่แท้จริงคือ "หน่วยความจำ" (แต่ละ connection กิน RAM เก็บ scope,
buffer) ไม่ใช่ "CPU" — นี่คือเหตุผลที่ WebSocket connection จำนวนมาก
"ไม่แพง" เท่า HTTP request จำนวนเท่ากันที่ต้องประมวลผลตลอดเวลา
```

### 739.3 Daphne (Twisted) เทียบกับ Uvicorn (asyncio/uvloop)

| ประเด็น | Daphne | Uvicorn |
|---|---|---|
| Event loop ที่ใช้ | Twisted reactor (library เก่าแก่ตั้งแต่ก่อนยุค `asyncio`) | `asyncio` มาตรฐาน หรือ `uvloop` (เร็วกว่า `asyncio` ปกติ เขียนด้วย Cython) |
| ผู้พัฒนา | ทีม Django เอง (ทีมเดียวกับที่ดูแล Channels) | ทีม Encode (ผู้สร้าง Starlette, HTTPX) |
| Multi-process ในตัว | ไม่มี — ต้องรันหลาย process ผ่านเครื่องมือภายนอก (systemd, supervisor) | มี flag `--workers N` ในตัว (แต่แนะนำใช้คู่กับ Gunicorn สำหรับ process management ที่แข็งแรงกว่าในงานจริง ตามที่ Part 057 ข้อ 569.3 แนะนำ) |
| ความเร็วดิบ (benchmark ทั่วไป) | ปานกลาง | เร็วกว่าอย่างมีนัยสำคัญในหลาย benchmark เพราะ `uvloop` เขียนด้วย Cython |
| ความเสถียร/ความนิยมในงาน Channels โดยเฉพาะ | เป็นตัวอ้างอิงทางการ เข้ากับ Channels เนียนที่สุด | ต้องพึ่ง `uvicorn-worker` ประกอบกับ Gunicorn สำหรับ Channels ให้ทำงานราบรื่นเต็มที่ |

### 739.4 กี่ Process ถึงจะพอ: สูตรเริ่มต้น

```bash
# สูตรเริ่มต้นที่ใช้กันทั่วไป: จำนวน worker process ≈ จำนวน CPU core
# (ต่างจาก Gunicorn sync worker ของ Part 089 ที่มักตั้ง 2×core+1 เพราะแต่ละ
# sync worker บล็อกขณะรอ I/O — แต่ ASGI worker ไม่บล็อกแบบนั้น จึงไม่ต้องคูณ 2)

# ตัวอย่างเครื่องที่มี 4 CPU core
gunicorn config.asgi:application \
    -k uvicorn_worker.UvicornWorker \
    --workers 4 \
    --bind 0.0.0.0:8001
```

แต่ละ worker process (แต่ละ event loop) รองรับ connection พร้อมกันได้ตาม
ข้อจำกัดของ **memory** และ **file descriptor** ไม่ใช่จำนวน CPU core — เครื่อง
ที่มี 4 core และ RAM 8 GB อาจรองรับได้รวมกันหลักหมื่น connection idle พร้อม
กันทั้ง 4 process รวมกัน (ตัวเลขจริงต้อง**วัดผลบนเครื่อง staging จริง**เสมอ
ตามหลักการ Load Testing ของ Part 072 ไม่ใช่เดาจากทฤษฎีอย่างเดียว)

### 739.5 ปรับ OS-Level Limits ก่อน Scale ไปสู่ Connection จำนวนมาก

```bash
# ตรวจสอบ ulimit ปัจจุบัน (จำนวน file descriptor สูงสุดต่อ process)
ulimit -n
# ค่า default ของ Linux distro ส่วนใหญ่คือ 1024 — ต่ำเกินไปมากสำหรับ
# WebSocket server ที่ต้องรองรับหลักพัน-หมื่น connection ต่อ process

# เพิ่มค่าแบบถาวรใน /etc/security/limits.conf
echo "www-data soft nofile 65536" | sudo tee -a /etc/security/limits.conf
echo "www-data hard nofile 65536" | sudo tee -a /etc/security/limits.conf

# ถ้ารันผ่าน systemd ต้องตั้งใน service unit ด้วย (limits.conf ไม่ครอบคลุม systemd)
```

```ini
# /etc/systemd/system/django-asgi.service
[Unit]
Description=Django ASGI Server (Channels)
After=network.target redis.service

[Service]
User=www-data
Group=www-data
WorkingDirectory=/opt/django-mastery-course
Environment="DJANGO_SETTINGS_MODULE=config.settings"
LimitNOFILE=65536
ExecStart=/opt/django-mastery-course/venv/bin/gunicorn \
    config.asgi:application \
    -k uvicorn_worker.UvicornWorker \
    --workers 4 \
    --bind 0.0.0.0:8001
Restart=on-failure
RestartSec=5
TimeoutStopSec=30

[Install]
WantedBy=multi-user.target
```

**ทำไม `TimeoutStopSec=30`**: เมื่อ systemd สั่งหยุด service (ตอน deploy
ใหม่) มันจะส่ง `SIGTERM` ให้ process ก่อน แล้วรอ `TimeoutStopSec` วินาที
ก่อนจะบังคับ `SIGKILL` — WebSocket connection ที่เปิดค้างอยู่ต้องการเวลา
มากกว่า HTTP request ธรรมดาในการปิดอย่างสุภาพ (graceful) เพราะ ASGI server
ต้องส่ง close frame ให้ทุก connection ที่เปิดอยู่ก่อนจะปิดตัวจริง ค่า
default ที่สั้นเกินไป (systemd default คือ 90 วินาทีจริง ๆ แต่หลายทีมลด
ลงโดยไม่รู้ผลกระทบ) จะทำให้ connection หลายพันเส้นถูกตัดแบบ abrupt (code
1006) พร้อมกันหมดตอน deploy ทุกครั้ง ซึ่งกระทบ user experience โดยไม่จำเป็น

### 739.6 Graceful Shutdown: แจ้ง Client ก่อนที่จะปิดจริง

```python
# realtime/consumers.py
import signal

from channels.generic.websocket import AsyncWebsocketConsumer


class GracefulShutdownMixin:
    """Mixin ที่ทำให้ Consumer ส่งข้อความเตือน client ก่อนถูกตัดตอน server กำลัง
    จะ restart/deploy แทนที่จะถูกตัดแบบไม่มีการเตือนล่วงหน้าเลย"""

    async def notify_server_restarting(self, event):
        await self.send(text_data='{"event": "server_restarting", "reconnect_in_ms": 3000}')
        await self.close(code=4009)  # 4009 = กำหนดเองให้สื่อถึง "server กำลังจะปิด"
```

ผูก `notify_server_restarting` เข้ากับกลุ่มกว้าง ๆ ที่ Consumer ทุกตัวเข้า
ร่วมอยู่แล้ว (เช่น `system_announcements` จากข้อ 731.5) แล้วให้ deployment
script เรียก management command ก่อน `systemctl restart` เพื่อ
`group_send` ข้อความนี้ไปหาทุก connection ล่วงหน้าสัก 2-3 วินาที — ฝั่ง
client ที่ implement `ResilientWebSocket` ตามข้อ 734.4 ไว้แล้วจะ reconnect
ให้อัตโนมัติ แต่การได้รับข้อความเตือนล่วงหน้าทำให้ UI แสดงสถานะ "กำลัง
เชื่อมต่อใหม่" ให้ผู้ใช้เห็นแทนที่จะรู้สึกเหมือนระบบพังกะทันหันโดยไม่มี
สาเหตุ

---

## ขั้นตอนที่ 740: สรุปและแบบฝึกหัด — ระบบแชท Real-time เต็มรูปแบบพร้อม Presence Tracking

### 740.1 Capstone: ประกอบทุกเทคนิคจากขั้นตอนที่ 731-739 เข้าด้วยกัน

โจทย์: ห้องแชทที่รองรับ (1) หลายห้องพร้อมกันต่อผู้ใช้หนึ่งคน (2) Presence
Tracking แสดงว่าใครออนไลน์ (3) Rate Limiting ป้องกัน spam (4) Reconnect
อัตโนมัติแบบ exponential backoff ฝั่ง client (5) โครงสร้าง Consumer ที่แชร์
logic ผ่าน Base Class

```python
# realtime/consumers.py — เวอร์ชันสมบูรณ์ที่ประกอบทุกขั้นตอนเข้าด้วยกัน
import json

from channels.generic.websocket import AsyncJsonWebsocketConsumer

from . import presence, ratelimit


class ProductionChatConsumer(AsyncJsonWebsocketConsumer):
    """
    รวมเทคนิคจากขั้นตอนที่ 731 (dynamic group), 734 (graceful disconnect),
    735 (presence), 736 (rate limit) เข้าเป็น Consumer เดียวที่ใช้งานได้จริง
    """

    MAX_MESSAGES = 15
    WINDOW_SECONDS = 10
    MAX_VIOLATIONS_BEFORE_KICK = 5

    async def connect(self):
        user = self.scope["user"]
        if not user.is_authenticated:
            await self.close(code=4001)
            return

        self.room_name = self.scope["url_route"]["kwargs"]["room_name"]
        self.room_group_name = f"chat_{self.room_name}"
        self.user_id = str(user.id)
        self.username = user.username
        self.violation_count = 0

        await self.channel_layer.group_add(self.room_group_name, self.channel_name)
        await self.accept()

        online_ids = await presence.get_online_user_ids(self.room_name)
        await self.send_json({"event": "presence_snapshot", "online_user_ids": online_ids})

        is_newly_online = await presence.mark_online(self.room_name, self.user_id)
        if is_newly_online:
            await self.channel_layer.group_send(self.room_group_name, {
                "type": "presence_update", "user_id": self.user_id,
                "username": self.username, "status": "online",
            })

    async def disconnect(self, close_code):
        await self.channel_layer.group_discard(self.room_group_name, self.channel_name)

        went_offline = await presence.mark_offline(self.room_name, self.user_id)
        if went_offline:
            await self.channel_layer.group_send(self.room_group_name, {
                "type": "presence_update", "user_id": self.user_id,
                "username": self.username, "status": "offline",
            })

    async def receive_json(self, content, **kwargs):
        message = content.get("message", "").strip()
        if not message:
            await self.send_json({"error": "ข้อความห้ามว่างเปล่า"})
            return

        limited = await ratelimit.is_rate_limited(
            f"ratelimit:chat:{self.user_id}", self.MAX_MESSAGES, self.WINDOW_SECONDS,
        )
        if limited:
            self.violation_count += 1
            await self.send_json({
                "error": "ส่งข้อความเร็วเกินไป กรุณารอสักครู่",
                "violation_count": self.violation_count,
            })
            if self.violation_count >= self.MAX_VIOLATIONS_BEFORE_KICK:
                await self.close(code=4029)
            return

        await self.channel_layer.group_send(self.room_group_name, {
            "type": "chat_message",
            "sender_id": self.user_id,
            "sender_username": self.username,
            "message": message[:2000],  # จำกัดความยาวป้องกัน payload ใหญ่ผิดปกติ
        })

    async def chat_message(self, event):
        await self.send_json({
            "event": "chat_message",
            "sender_id": event["sender_id"],
            "sender_username": event["sender_username"],
            "message": event["message"],
        })

    async def presence_update(self, event):
        await self.send_json({
            "event": "presence_update",
            "user_id": event["user_id"],
            "username": event["username"],
            "status": event["status"],
        })
```

```python
# realtime/routing.py
from django.urls import path

from . import consumers

websocket_urlpatterns = [
    path("ws/chat/<str:room_name>/", consumers.ProductionChatConsumer.as_asgi()),
]
```

Frontend ผสาน `ResilientWebSocket` จากข้อ 734.4 เข้ากับ UI แสดงรายชื่อ
ออนไลน์และห้องแชท:

```javascript
// static/js/chat-room.js
function initChatRoom(roomName, currentUsername) {
    const onlineUsers = new Set();
    const client = new ResilientWebSocket(
        `wss://${window.location.host}/ws/chat/${roomName}/`,
        { maxAttempts: null },  // ห้องแชทควรพยายาม reconnect ไม่จำกัดจำนวนครั้ง
    );

    function renderOnlineList() {
        document.getElementById('online-count').textContent = onlineUsers.size;
    }

    function appendMessage(sender, message) {
        const el = document.createElement('div');
        el.className = 'chat-message';
        el.innerHTML = `<strong>${sender}</strong>: ${message}`;
        document.getElementById('chat-messages').appendChild(el);
        el.scrollIntoView();
    }

    client.onmessage = (event) => {
        const data = JSON.parse(event.data);

        if (data.event === 'presence_snapshot') {
            data.online_user_ids.forEach((id) => onlineUsers.add(id));
            renderOnlineList();
        } else if (data.event === 'presence_update') {
            if (data.status === 'online') onlineUsers.add(data.user_id);
            else onlineUsers.delete(data.user_id);
            renderOnlineList();
        } else if (data.event === 'chat_message') {
            appendMessage(data.sender_username, data.message);
        } else if (data.error) {
            appendMessage('ระบบ', data.error);
        }
    };

    client.connect();

    document.getElementById('chat-form').addEventListener('submit', (e) => {
        e.preventDefault();
        const input = document.getElementById('chat-input');
        if (!input.value.trim()) return;
        client.send(JSON.stringify({ message: input.value }));
        input.value = '';
    });

    return client;
}
```

### 740.2 สรุปสิ่งที่ได้เรียนรู้ใน Part นี้

- ✅ เข้าใจฟีเจอร์ `groups` class attribute ของ Generic Consumer และรู้
  ข้อจำกัดสำคัญ (join กลุ่มก่อนตรวจสิทธิ์เสมอ) พร้อมเขียน Consumer ที่เข้า
  ร่วมหลายกลุ่มแบบไดนามิกด้วยมือได้อย่างปลอดภัย
- ✅ ออกแบบ Base Class (`AuthenticatedGroupConsumer`) และ Mixin
  (`RateLimitMixin`, `LoggingMixin`) เพื่อแชร์ logic ระหว่าง Consumer หลาย
  ตัวตามหลัก DRY โดยเข้าใจว่าเมื่อไหร่ควรใช้ Inheritance กับเมื่อไหร่ควรใช้
  Mixin แบบ composition
- ✅ ตั้งค่า `channels_redis` ระดับ production: `capacity`, `expiry`,
  `group_expiry`, sharding ข้ามหลาย Redis host, และเลือกระหว่าง
  `RedisChannelLayer` กับ `RedisPubSubChannelLayer` ได้ถูกต้อง
- ✅ จัดการ `disconnect()` ให้ครอบคลุมทุกสถานการณ์ที่เป็นไปได้ ดักจับ
  exception ใน `receive()` อย่างเหมาะสม ใช้ `StopConsumer` และเขียนกลยุทธ์
  reconnect ฝั่ง client ด้วย exponential backoff + jitter พร้อม heartbeat
  ตรวจจับ half-open connection
- ✅ สร้างระบบ Presence Tracking ด้วย Redis Hash + Counter ที่รองรับผู้ใช้
  เปิดหลายแท็บ/อุปกรณ์พร้อมกันได้อย่างถูกต้อง
- ✅ Implement Rate Limiting แบบ Sliding Window Log ด้วย Redis Sorted Set
  พร้อมนโยบายลงโทษที่เข้มขึ้นตามจำนวนครั้งที่ทำผิด
- ✅ เข้าใจว่าทำไม Channels ไม่ต้องพึ่ง Sticky Session เมื่อ scale ข้าม
  หลาย server และวางแผน High Availability ด้วย Redis Sentinel
- ✅ เปรียบเทียบ Django Channels กับ Socket.IO และ ASGI service แยกได้
  อย่างเป็นกลาง รู้ว่าเมื่อไหร่ควรแยก realtime ออกเป็น service ต่างหาก
- ✅ เข้าใจ Process Model ของ Daphne/Uvicorn ในระดับลึก (event loop เดียว
  จัดการ connection จำนวนมากผ่าน idle time), ปรับ OS-level limits
  (`ulimit`), และ implement graceful shutdown ที่แจ้งเตือน client ก่อน

### 740.3 Checklist ก่อนไป Part ถัดไป

- [ ] เขียน Consumer ที่เข้าร่วมหลายกลุ่มแบบไดนามิกได้ (กลุ่มส่วนตัว + กลุ่ม
      ห้อง + กลุ่มระบบ) พร้อม `group_discard` ครบทุกกลุ่มตอน disconnect
- [ ] สร้าง Base Class หรือ Mixin อย่างน้อย 1 ตัว และนำไปใช้กับ Consumer
      จริงอย่างน้อย 2 ตัวที่แชร์ logic ร่วมกัน
- [ ] ตั้งค่า `CHANNEL_LAYERS` ด้วย `capacity`/`expiry`/`group_expiry` ที่
      ปรับให้เหมาะกับงานของตัวเอง (ไม่ใช้ค่า default เปล่า ๆ)
- [ ] เขียน `disconnect()` ที่ทำงานถูกต้องไม่ว่าจะปิดด้วยเหตุผลไหน และ
      implement `ResilientWebSocket` ฝั่ง client พร้อม exponential backoff
- [ ] สร้างระบบ Presence Tracking ที่ทดสอบแล้วว่าเปิด 2 แท็บพร้อมกันไม่ทำ
      ให้สถานะออนไลน์ผิดพลาดตอนปิดแท็บใดแท็บหนึ่ง
- [ ] Implement Rate Limiting และทดสอบว่าผู้ใช้ที่ส่งข้อความรัวเกินโดน
      ปฏิเสธ/ตัดการเชื่อมต่อตามนโยบายที่ตั้งไว้จริง
- [ ] อธิบายได้ด้วยคำพูดตัวเองว่าทำไม Channels ไม่ต้องใช้ Sticky Session
      เมื่อ scale ข้ามหลาย server
- [ ] ตั้งค่า `ulimit`/systemd service file สำหรับรัน ASGI server ใน
      production พร้อม `TimeoutStopSec` ที่เหมาะสมกับ WebSocket

### 740.4 แบบฝึกหัดท้ายบท

**แบบฝึกหัดที่ 1**: ต่อยอด `HubConsumer` จากขั้นตอนที่ 731.5 ให้รองรับคำสั่ง
`leave_all_rooms` ที่ให้ผู้ใช้ออกจากทุกห้องแชทที่ join ไว้พร้อมกันในคำสั่ง
เดียว (แต่ยังคงอยู่ในกลุ่มส่วนตัวและกลุ่มระบบต่อไป ไม่ถูกตัดการเชื่อมต่อ)

**แบบฝึกหัดที่ 2**: เขียนเทสต์ (อ้างอิงรูปแบบจาก Part 057 ข้อ 568) สำหรับ
`ProductionChatConsumer` ของขั้นตอนที่ 740.1 อย่างน้อย 5 เคส: (1) ผู้ใช้ที่
ไม่ login ถูกปฏิเสธ (2) ข้อความกระจายถึงทุกคนในห้องเดียวกัน (3) Presence
แสดงผลถูกต้องเมื่อมี 2 connection ของผู้ใช้คนเดียวกัน (4) Rate Limit ทำงาน
เมื่อส่งเกิน `MAX_MESSAGES` ภายใน `WINDOW_SECONDS` (5) ผู้ใช้ที่ถูก rate
limit ซ้ำครบ `MAX_VIOLATIONS_BEFORE_KICK` ครั้งถูกตัดการเชื่อมต่อจริง

**แบบฝึกหัดที่ 3**: ปรับปรุง `ratelimit.is_rate_limited()` จากขั้อ 736.3 ให้
รองรับ **หลายระดับ limit พร้อมกัน** เช่น "ไม่เกิน 15 ข้อความใน 10 วินาที
**และ** ไม่เกิน 100 ข้อความใน 1 ชั่วโมง" (ป้องกันทั้ง burst สั้น ๆ และ
spam ต่อเนื่องระยะยาว) โดยเรียกฟังก์ชันเดิมสองครั้งด้วยพารามิเตอร์ต่างกัน
แล้ว rate limit ถ้าเงื่อนไขใดเงื่อนไขหนึ่งเกิน

**แบบฝึกหัดที่ 4 (ขั้นสูง — Capstone เต็มรูปแบบ)**: สร้างระบบแชทแบบ
real-time ที่สมบูรณ์ที่สุดโดยรวมทุกอย่างจาก Part นี้เข้าด้วยกัน: (1)
`ProductionChatConsumer` ตามขั้อ 740.1 พร้อม Presence และ Rate Limiting
(2) รองรับหลายห้องพร้อมกันต่อผู้ใช้แบบ `HubConsumer` (ขั้อ 731.5) (3)
Heartbeat ตรวจจับ half-open connection (ขั้อ 734.5) (4) Deploy จริงบน
เครื่อง staging ด้วย Gunicorn+Uvicorn worker หลาย process ที่ชี้ Redis
เดียวกัน (ขั้อ 737, 739) แล้วทดสอบว่าผู้ใช้ 2 คนที่ถูก Load Balancer ส่งไป
คนละ process ยังคุยกันได้ปกติ (5) ทำ Load Test ด้วยเครื่องมือจาก Part 072
จำลอง WebSocket connection พร้อมกันอย่างน้อย 500 connection แล้วบันทึกผล
memory/CPU ของแต่ละ worker process ก่อน-หลัง

### 740.5 คำถามที่พบบ่อย (FAQ)

**Q: ควรใช้ `groups` class attribute หรือ `channel_layer.group_add()` ด้วย
มือเป็นค่าเริ่มต้นของทุกโปรเจกต์?**
A: ใช้ `channel_layer.group_add()` ด้วยมือเป็นค่าเริ่มต้นเสมอ เพราะข้อ
731.3 แสดงให้เห็นแล้วว่า `groups` attribute มีความเสี่ยงด้านความปลอดภัยถ้า
Consumer ต้องตรวจสิทธิ์ก่อนอนุญาตให้เข้ากลุ่ม (ซึ่งเป็นกรณีส่วนใหญ่ในงาน
จริง) สงวน `groups` attribute ไว้เฉพาะกลุ่มสาธารณะแท้ ๆ ที่ไม่มีเงื่อนไข
สิทธิ์ใด ๆ เลยเท่านั้น

**Q: Presence Tracking จำเป็นต้องใช้ Redis เสมอไหม ใช้ PostgreSQL แทนได้
หรือไม่?**
A: ทำได้ในทางเทคนิค แต่ไม่แนะนำ เพราะ Presence เปลี่ยนแปลงถี่มาก (ทุกครั้ง
ที่ connect/disconnect) การเขียน PostgreSQL ถี่ขนาดนั้นสร้างภาระให้
ฐานข้อมูลหลักที่ควรสงวนไว้สำหรับข้อมูลที่ต้อง durable จริง ๆ (ทวนหลักการ
"เลือก storage ให้เหมาะกับลักษณะข้อมูล" จาก Part 069) Redis เหมาะกว่ามาก
เพราะ presence เป็นข้อมูลที่ "หายไปได้ถ้า Redis restart" โดยไม่กระทบความ
ถูกต้องของระบบระยะยาว (แค่ต้องรอให้ผู้ใช้ reconnect แล้ว mark_online ใหม่)

**Q: Rate Limiting ควรทำที่ฝั่ง Consumer (application level) หรือที่ระดับ
Nginx/Load Balancer เลย?**
A: ทำทั้งสองระดับเพื่อป้องกันคนละปัญหา — Nginx/Load Balancer (ทวน Part
057 ข้อ 569.5) เหมาะกับการจำกัด**จำนวน connection ใหม่**ต่อ IP ในเวลาสั้น ๆ
(ป้องกัน DDoS ระดับ connection) ในขณะที่ Rate Limiting ระดับ Consumer แบบ
ขั้นตอนที่ 736 เหมาะกับการจำกัด**จำนวนข้อความภายใน connection ที่เปิดอยู่
แล้ว**ต่อผู้ใช้ (ป้องกัน spam ระดับ application) ทั้งสองชั้นทำงานเสริมกัน
ไม่ใช่แทนกัน

**Q: ทำไม Part นี้ไม่สอนใช้ Django Channels ร่วมกับ Celery เลย ทั้งที่ทั้ง
คู่เกี่ยวกับงาน asynchronous เหมือนกัน?**
A: เพราะทั้งสองแก้ปัญหาคนละแบบที่ไม่ทับซ้อนกัน — Channels แก้ปัญหา "server
ต้องส่งข้อมูลเข้าหา client ได้เองแบบ real-time" (ทวนข้อ 561.1 ของ Part 057)
ส่วน Celery (Part 075-076 ถัดไป) แก้ปัญหา "งานหนักที่ไม่ต้องรอให้เสร็จ
ทันทีควรย้ายออกจาก request-response cycle" ทั้งสองมักถูก**ใช้ร่วมกัน**ใน
งานจริง เช่น Celery task ประมวลผลรายงานเสร็จแล้วเรียก
`channel_layer.group_send()` (แบบเดียวกับที่ signal ทำในข้อ 566.2 ของ Part
057) เพื่อแจ้งผลลัพธ์กลับไปหา client แบบ real-time — Part 078 (Real-time
Notification) จะสาธิตการผสานทั้งสองเรื่องเข้าด้วยกันโดยตรง

---

## เตรียมตัวสำหรับ Part ถัดไป

**Part 075: Celery พื้นฐาน — Background Task แรกของคุณ** จะเริ่มต้นเรื่อง
ใหม่ที่ต่างจาก WebSocket โดยสิ้นเชิง — แทนที่จะให้ server "ส่งข้อมูลเข้าหา
client ทันที" แบบที่ Part 057 และ Part นี้ทำ Celery จะช่วยให้ View "ส่งงาน
หนักไปทำเบื้องหลัง" โดยไม่ต้องให้ผู้ใช้รอ (ส่งอีเมล, ประมวลผลรูปภาพ, สร้าง
รายงาน PDF) คุณจะได้ติดตั้ง Celery Worker ตัวแรก เชื่อมกับ Redis ที่ตั้งค่า
ไว้แล้วตั้งแต่ Part 069 (และใช้เป็น Channel Layer ใน Part นี้) ในฐานะ
**Message Broker** ย้าย Signal ที่เคยเรียก `channel_layer.group_send()`
ตรง ๆ แบบ synchronous ในข้อ 566.2 ของ Part 057 ไปเป็น Celery Task
แบบ asynchronous แทน (ตอบคำถามที่ FAQ ข้อสุดท้ายของ Part 057 ทิ้งไว้เรื่อง
ผลกระทบต่อความเร็วของ View) และเริ่มเข้าใจความแตกต่างระหว่าง "งานที่ต้อง
ตอบกลับทันที" (Channels) กับ "งานที่ทำทีหลังได้" (Celery) อย่างชัดเจน —
เตรียม Redis instance ที่ใช้งานอยู่ให้พร้อม เพราะจะถูกใช้เป็นทั้ง Cache,
Channel Layer, และ Celery Broker ในเวลาเดียวกัน
