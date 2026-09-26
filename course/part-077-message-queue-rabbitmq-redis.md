# Part 077: Message Queue ด้วย RabbitMQ/Redis

> **ขั้นตอนที่ 761-770 ของหลักสูตร** | Phase 9: Async, Celery และ Channels
>
> Part 075 และ 076 สอนให้คุณใช้ Celery เป็นเครื่องมือได้อย่างคล่องแคล่วแล้ว —
> เขียน Task, ตั้ง Retry, ทำ Chain/Group/Chord, และแยกงานไปคนละ Queue ตาม
> ความสำคัญ แต่ตลอดสองบทที่ผ่านมา คุณใช้ **Redis** เป็น Broker โดยไม่เคยถามว่า
> "ทำไมต้องเป็น Broker" หรือ "ข้างใน Broker เกิดอะไรขึ้นบ้าง" กันแน่ — Part นี้
> คือ Part ที่เปิดฝาเครื่องยนต์ดูเบื้องหลัง เราจะทำความเข้าใจแนวคิด **Message
> Queue** ในภาพกว้างกว่า Celery, เจาะลึก **RabbitMQ** ในฐานะ Message Broker
> แท้ ๆ ที่พูดภาษา **AMQP** เปรียบเทียบกับ Redis อย่างละเอียดทั้งด้าน
> reliability, feature และความซับซ้อนในการ deploy (ตามที่ Part 076 ขั้นตอนที่
> 756.4 ทิ้งคำถามค้างไว้เรื่อง priority queue), ติดตั้ง RabbitMQ ใช้งานจริง,
> เข้าใจ Exchange/Queue/Binding, สร้างรูปแบบ **Dead Letter Queue** สำหรับงาน
> ที่ล้มเหลวซ้ำ ๆ, ทำความเข้าใจ **Message Acknowledgment** อย่างลึกซึ้งว่า
> at-least-once กับ at-most-once ต่างกันอย่างไรและกระทบระบบจริงยังไง, สเกล
> ด้วยหลาย Worker แบบ queue-based load balancing, Monitor queue depth เพื่อ
> รู้สุขภาพระบบก่อนที่ผู้ใช้จะร้องเรียน และปิดท้ายด้วยกรอบการตัดสินใจเลือก
> Broker สำหรับ production พร้อมลงมือย้าย broker ของ Blog project จาก Redis
> ไปเป็น RabbitMQ จริงโดยไม่กระทบ business logic ของ Task แม้แต่บรรทัดเดียว

---

## สารบัญของ Part นี้

- ขั้นตอนที่ 761: แนวคิด Message Queue โดยรวม — ทำไม Celery ต้องมี Broker
- ขั้นตอนที่ 762: RabbitMQ vs Redis เป็น Broker — ตารางเปรียบเทียบ (reliability, feature, ความซับซ้อนในการ deploy)
- ขั้นตอนที่ 763: ติดตั้งและตั้งค่า RabbitMQ สำหรับใช้กับ Celery
- ขั้นตอนที่ 764: แนวคิด Exchange/Queue/Binding ของ RabbitMQ (AMQP protocol)
- ขั้นตอนที่ 765: รูปแบบ Dead Letter Queue สำหรับ message ที่ประมวลผลไม่สำเร็จ
- ขั้นตอนที่ 766: Message Acknowledgment และการรับประกันความน่าเชื่อถือ (at-least-once vs at-most-once delivery)
- ขั้นตอนที่ 767: หลาย Worker และ Load Balancing แบบ queue-based
- ขั้นตอนที่ 768: การ Monitor ความลึกของ Queue (queue depth) และสุขภาพของระบบ
- ขั้นตอนที่ 769: กรอบการตัดสินใจเลือก Broker สำหรับ Production จริง
- ขั้นตอนที่ 770: สรุปและแบบฝึกหัด — ย้าย broker ของ blog project จาก Redis ไปเป็น RabbitMQ

---

## ขั้นตอนที่ 761: แนวคิด Message Queue โดยรวม — ทำไม Celery ต้องมี Broker

### 761.1 ทวนสถาปัตยกรรม Celery จาก Part 075

ทวนแผนภาพจาก Part 075 ขั้นตอนที่ 742.1: Django View (Producer) ไม่เคยคุยกับ
Celery Worker (Consumer) โดยตรงเลยแม้แต่ครั้งเดียว — ทุกครั้งที่เรียก
`.delay()` งานจะถูกส่งผ่าน **Broker** ตรงกลางเสมอ:

```
Django View ──.delay()──> Broker (คิวกลาง) ──> Worker หยิบไปทำ
 (Producer)                                      (Consumer)
```

คำถามของ Part นี้คือ: **ทำไมต้องมีตัวกลางนี้ด้วย? ทำไม View เรียก Worker
ตรง ๆ ไม่ได้เลย?**

### 761.2 ถ้าไม่มี Broker: ปัญหาที่เกิดขึ้นทันที

ลองจินตนาการว่า Django View เรียก Worker ตรง ๆ ผ่าน network (เช่น HTTP call
ไปยัง Worker service):

```python
# สมมติฐาน (ไม่ใช่โค้ดจริง) — Django เรียก Worker ตรง ๆ แบบ synchronous HTTP
def signup_view(request):
    user = form.save()
    requests.post('http://worker-service:9000/send-email', json={
        'email': user.email, 'username': user.username,
    })  # ❌ ต้องรอ Worker ตอบกลับ — กลับไปเจอปัญหาเดิมจาก Part 075 ขั้นตอนที่ 741.2!
    return redirect('success')
```

ปัญหาที่เกิดขึ้นทันทีมี 3 ข้อ:

1. **View ต้องรอ Worker ตอบกลับอีกครั้ง** — ย้อนกลับไปที่ปัญหาบล็อก request
   cycle ที่ Part 075 พยายามแก้ตั้งแต่ต้น
2. **ถ้า Worker ไม่ว่าง (กำลังทำงานอื่นอยู่) หรือ service ล่มชั่วคราว งานนั้น
   หายไปเลย** — ไม่มีที่เก็บงานไว้รอ
3. **View ต้องรู้ว่า Worker ตัวไหนว่าง** — ต้องทำ load balancing เอง, ต้องรู้
   IP/hostname ของ Worker ทุกตัว, เมื่อเพิ่ม/ลด Worker ต้องแก้โค้ด View ด้วย

### 761.3 Message Queue แก้ปัญหาทั้ง 3 ข้อพร้อมกันด้วยหลักการเดียว: Decoupling

**Message Queue** คือคิวกลางที่ทำหน้าที่ **แยก (decouple)** ฝั่งที่สร้างงาน
(Producer) ออกจากฝั่งที่ทำงาน (Consumer) โดยสิ้นเชิง — ทั้งสองฝั่งไม่จำเป็น
ต้องรู้จักกันเลยแม้แต่น้อย รู้จักแค่ "คิวกลาง" ตัวเดียว:

```
┌─────────────┐                ┌──────────────┐                ┌─────────────┐
│  Producer   │  ส่งงานเข้าคิว  │    Message   │  ดึงงานไปทำ    │  Consumer   │
│ (Django View)│ ──────────────>│    Queue     │<───────────────│(Celery Worker)│
└─────────────┘                │  (Broker)    │                └─────────────┘
                                └──────────────┘
      Producer ไม่รู้จัก Consumer เลย รู้แค่ว่า "ส่งเข้าคิวนี้"
      Consumer ไม่รู้จัก Producer เลย รู้แค่ว่า "ดึงจากคิวนี้"
```

| ปัญหาจากขั้นตอนที่ 761.2 | Message Queue แก้อย่างไร |
|---|---|
| View ต้องรอ Worker ตอบกลับ | Producer ส่งเข้าคิวแล้ว **return ทันที** ไม่ต้องรอ Consumer เลย |
| งานหายถ้า Worker ไม่ว่าง/ล่ม | คิวเก็บงานไว้ (buffer) จนกว่าจะมี Consumer มาหยิบ — งานไม่หายแม้ Worker จะล่มชั่วคราว |
| View ต้องรู้จัก Worker ทุกตัว | View รู้จักแค่ Broker ตัวเดียว เพิ่ม/ลด Worker กี่ตัวก็ได้โดยไม่แก้โค้ด View เลย (ขยายความในขั้นตอนที่ 767) |

หลักการนี้เรียกว่า **Producer-Consumer Pattern** ผสมกับแนวคิด **Buffering /
Load Leveling** — คิวทำหน้าที่เป็น "กันชน" (buffer) ระหว่างจังหวะที่ Producer
สร้างงานเร็วกว่าที่ Consumer ทำงานทัน (เช่น มีคนสมัครสมาชิกพร้อมกัน 1,000 คน
ในเสี้ยววินาที) — งานจะไม่หายหรือทำให้ระบบล่ม แต่จะถูกเก็บคิวไว้ให้ Worker
ทยอยทำไปเรื่อย ๆ ตามความสามารถของมันเอง

### 761.4 Message Queue ไม่ใช่แนวคิดที่ผูกกับ Celery เท่านั้น

Celery คือ **framework ระดับสูง** ที่สร้างอยู่**บน**แนวคิด Message Queue อีกที
— ตัว Message Queue เองเป็นแนวคิดสถาปัตยกรรมที่ใช้กันทั่วทั้งอุตสาหกรรม
ไม่ผูกกับ Python หรือ Django เลย:

| ระบบ | ใช้ทำอะไร | ใช้ที่ไหนในโลกจริง |
|---|---|---|
| **RabbitMQ** | Message Broker ทั่วไป รองรับ AMQP | ระบบ enterprise, microservices, e-commerce order processing |
| **Redis** (เมื่อใช้เป็น queue) | In-memory store ที่ใช้ List/Pub-Sub จำลอง queue ได้ | Celery broker, งานที่ต้องการความเร็วสูงและ throughput มาก |
| **Apache Kafka** | Distributed Event Streaming Platform (ไม่ใช่ queue ล้วน ๆ แต่ใช้แทนกันได้ในหลายกรณี) | LinkedIn, Netflix, Uber — งาน event-driven ขนาดใหญ่ระดับ streaming |
| **Amazon SQS** | Managed Message Queue บน AWS | ระบบที่ไม่อยากดูแล infrastructure เอง |
| **Google Cloud Pub/Sub** | Managed Messaging บน GCP | ระบบ event-driven บน Google Cloud |

**สิ่งที่หลักสูตรนี้จะโฟกัส**: RabbitMQ และ Redis เพราะเป็นสองตัวที่ Celery
รองรับเป็นทางการและนิยมใช้คู่กับ Django มากที่สุด แต่หลักการ Decoupling ที่
เรียนใน Part นี้ใช้ได้กับทุกระบบ message queue ในตารางข้างบน

### 761.5 คุณสมบัติหลักที่ Message Queue ทุกตัวควรมี

| คุณสมบัติ | ความหมาย | ทำไมสำคัญ |
|---|---|---|
| **Persistence (ความคงทน)** | งานที่ยังไม่ถูกประมวลผลไม่หายไปแม้ broker restart | ป้องกันข้อมูลสูญหายเมื่อระบบมีปัญหา |
| **Ordering (การเรียงลำดับ)** | งานถูกประมวลผลตามลำดับที่เข้าคิว (FIFO เป็นส่วนใหญ่) | บางงานต้องทำตามลำดับ (เช่น อัปเดตยอดเงินในบัญชี) |
| **Delivery Guarantee** | รับประกันว่างานจะถูกส่งถึง Consumer อย่างน้อยหนึ่งครั้ง (หรือมากกว่า) | ความน่าเชื่อถือของระบบ (เจาะลึกในขั้นตอนที่ 766) |
| **Routing** | ส่งงานไปยังคิวที่ถูกต้องตามเงื่อนไข | แยกงานสำคัญ/ไม่สำคัญ (ทวนจาก Part 076 ขั้นตอนที่ 756) |
| **Scalability** | เพิ่ม Producer/Consumer ได้โดยไม่กระทบกัน | รองรับการเติบโตของระบบ (ขั้นตอนที่ 767) |

RabbitMQ และ Redis ทำได้ทุกข้อนี้ในระดับที่ต่างกัน — นี่คือสิ่งที่ขั้นตอนที่
762 จะเจาะลึกทีละข้อ

### 761.6 ตารางสรุปขั้นตอนที่ 761

| หัวข้อ | สรุป |
|---|---|
| ทำไมต้องมี Broker | เพื่อ **decouple** Producer (Django View) ออกจาก Consumer (Celery Worker) โดยสิ้นเชิง |
| ปัญหาที่แก้ได้ | View ไม่ต้องรอ Worker, งานไม่หายเมื่อ Worker ไม่ว่าง, View ไม่ต้องรู้จัก Worker แต่ละตัว |
| แนวคิดหลัก | Producer-Consumer Pattern + Buffering/Load Leveling |
| Message Queue ผูกกับ Celery ไหม | ไม่ผูก — เป็นแนวคิดสถาปัตยกรรมกว้าง ใช้ได้กับ RabbitMQ, Redis, Kafka, SQS ฯลฯ |
| คุณสมบัติหลักที่ต้องมี | Persistence, Ordering, Delivery Guarantee, Routing, Scalability |

---

## ขั้นตอนที่ 762: RabbitMQ vs Redis เป็น Broker

### 762.1 ประวัติและปรัชญาการออกแบบที่ต่างกันตั้งแต่ต้น

| | RabbitMQ | Redis |
|---|---|---|
| เปิดตัวปี | 2007 | 2009 |
| ออกแบบมาเพื่อ | เป็น **Message Broker โดยเฉพาะ** ตั้งแต่วันแรก | เป็น **In-memory Key-Value Data Store** (ใช้ทำ cache, session ตามที่เรียนใน Part 069) |
| ภาษาที่เขียน | Erlang (ออกแบบมาเพื่อระบบ concurrent/distributed ที่ทนทานสูง — ภาษาเดียวกับที่ใช้สร้างระบบโทรศัพท์ของ Ericsson) | C |
| Protocol มาตรฐาน | **AMQP** (Advanced Message Queuing Protocol) — มาตรฐานเปิดที่มีสเปกชัดเจน | ไม่มี protocol มาตรฐานสำหรับ "queue" — ใช้ data structure ทั่วไป (List, Pub/Sub, Streams) จำลองพฤติกรรมคิวเอาเอง |

**ประเด็นสำคัญที่สุดของขั้นตอนนี้**: RabbitMQ ถูกสร้างมา**เพื่อเป็น message
broker** โดยตรง ทุกฟีเจอร์ (routing, acknowledgment, durability) ถูกออกแบบมา
รองรับ use case นี้ตั้งแต่สถาปัตยกรรมแรก ส่วน Redis ถูกสร้างมาเป็น **data
store อเนกประสงค์** แล้ว Celery (ผ่าน library `kombu`) เขียนโค้ดจำลองพฤติกรรม
queue ขึ้นมาจาก Redis List อีกที — นี่คือรากเหง้าของความแตกต่างทุกข้อที่จะ
เห็นในตารางต่อไปนี้

### 762.2 ตารางเปรียบเทียบด้าน Reliability (ความน่าเชื่อถือ)

| ประเด็น | RabbitMQ | Redis |
|---|---|---|
| **Message Durability** (message ไม่หายแม้ broker restart) | ✅ รองรับเต็มรูปแบบ — เขียนลง disk ได้ (`durable=True` ของ queue + `delivery_mode=2` ของ message) | ⚠️ ขึ้นกับการตั้งค่า persistence ของ Redis เอง (RDB snapshot / AOF) — ถ้าไม่ตั้งค่าอาจเสียงานที่ค้างอยู่ใน memory เมื่อ Redis restart กะทันหัน |
| **Message Acknowledgment** | ✅ Protocol ระดับ AMQP รองรับ ack/nack/reject อย่างเป็นทางการ, redeliver อัตโนมัติเมื่อ consumer หลุดการเชื่อมต่อ | ⚠️ Kombu จำลอง ack ด้วยกลไก "unacked list" + `visibility_timeout` เอง ไม่ใช่ความสามารถแท้ของ Redis (เจาะลึกในขั้นตอนที่ 766.5) |
| **Delivery Guarantee** | ✅ At-least-once ที่แน่นอนเมื่อ config ถูกต้อง (`publisher confirms` + durable queue) | ⚠️ At-least-once ได้เช่นกัน แต่ต้องพึ่งการตั้งค่า `visibility_timeout` ให้เหมาะสม เสี่ยง duplicate/loss มากกว่าถ้าตั้งค่าผิด |
| **Clustering สำหรับ High Availability** | ✅ รองรับ Quorum Queue / Mirrored Queue ข้ามหลายเครื่องอย่างเป็นทางการ | ⚠️ Redis Cluster/Sentinel ทำ HA ได้ แต่ไม่ได้ออกแบบมาเพื่อ "รับประกัน message ไม่หาย" โดยเฉพาะเหมือน RabbitMQ |
| **Flow Control** (ป้องกัน consumer รับงานล้น) | ✅ มีในตัว (TCP backpressure ระดับ protocol) | ❌ ไม่มี — ต้องพึ่ง `worker_prefetch_multiplier` ฝั่ง Celery ควบคุมเอาเอง |

### 762.3 ตารางเปรียบเทียบด้าน Feature

| ประเด็น | RabbitMQ | Redis |
|---|---|---|
| **Routing ซับซ้อน** (ส่งงานไปหลายคิวตามเงื่อนไข) | ✅ เต็มรูปแบบผ่าน Exchange 4 ประเภท (direct, topic, fanout, headers — เจาะลึกในขั้นตอนที่ 764) | ⚠️ ทำได้แค่ระดับพื้นฐาน (แยกคิวตามชื่อ) ไม่มี exchange/binding แบบ AMQP |
| **Message Priority** | ✅ รองรับเต็มรูปแบบผ่าน `x-max-priority` — ตั้ง priority ได้ละเอียดหลายระดับในคิวเดียว (ตอบคำถามค้างจาก Part 076 ขั้นตอนที่ 756.4) | ⚠️ รองรับแบบจำกัดมาก ผ่าน `priority` field ของ kombu แต่ไม่แม่นยำเท่า และ Celery/หลักสูตรนี้แนะนำให้แยกเป็นคนละคิวแทนเสมอ |
| **Dead Letter Queue แบบ native** | ✅ มีในตัวผ่าน `x-dead-letter-exchange` (เจาะลึกในขั้นตอนที่ 765) | ❌ ไม่มี native DLQ — ต้องสร้างเองระดับ application |
| **Message TTL / Queue Length Limit** | ✅ ตั้งได้ตรง ๆ ที่ระดับ queue (`x-message-ttl`, `x-max-length`) | ⚠️ ทำได้ผ่าน `CELERY_RESULT_EXPIRES` (เฉพาะ result) แต่ไม่มีกลไก TTL ระดับ message ในคิวงานเอง |
| **Management UI พร้อมใช้** | ✅ Web UI ในตัว (`rabbitmq_management` plugin) ดู queue, exchange, consumer แบบ real-time | ❌ ไม่มี UI เฉพาะสำหรับ queue — ต้องใช้ `redis-cli` หรือเครื่องมือ third-party |
| **ใช้เป็น Cache ได้ไหมในตัวเดียวกัน** | ❌ ไม่เหมาะ — RabbitMQ ไม่ใช่ data store ทั่วไป | ✅ ได้ — โปรเจกต์นี้ใช้ Redis ตัวเดียวกันทำทั้ง cache, session, throttle, broker (ทวนจาก Part 069) |

### 762.4 ตารางเปรียบเทียบด้านความซับซ้อนในการ Deploy

| ประเด็น | RabbitMQ | Redis |
|---|---|---|
| ติดตั้งครั้งแรก | ต้องติดตั้ง service แยกใหม่ทั้งหมด (ขั้นตอนที่ 763) | ถ้ามี Redis จาก Part 069 อยู่แล้ว **ไม่ต้องติดตั้งอะไรเพิ่มเลย** |
| จำนวน Service ที่ต้องดูแลใน production | เพิ่มอีก 1 service (RabbitMQ) นอกเหนือจาก PostgreSQL, Redis, Django | คงเดิม — ใช้ Redis ตัวเดียวกันกับที่มีอยู่แล้วทำหลายหน้าที่ |
| การตั้งค่า User/Permission/Vhost | ต้องตั้งค่าเพิ่มเติม (ไม่ควรใช้ `guest/guest` ใน production — ขั้นตอนที่ 763.5) | Redis ใช้ password เดียวกับที่ตั้งไว้แล้วสำหรับ cache |
| Resource ที่ใช้ (RAM/CPU เพิ่มเติม) | ใช้ RAM/CPU เพิ่มต่างหากจาก Redis ที่มีอยู่ | ไม่มี resource เพิ่มถ้าใช้ Redis instance เดิม (แต่ต้องระวัง memory pressure ถ้างานเยอะมาก — ทวนจาก Part 069 เรื่อง `maxmemory-policy`) |
| Learning Curve | สูงกว่า — ต้องเข้าใจ AMQP, Exchange, Binding | ต่ำกว่า — ถ้าคุ้นเคย Redis จาก Part 069 อยู่แล้วแทบไม่ต้องเรียนรู้อะไรใหม่ |
| Monitoring/Alerting Ecosystem | เครื่องมือ mature และครบครัน (Management UI, Prometheus exporter อย่างเป็นทางการ) | ต้องพึ่งเครื่องมือทั่วไปของ Redis (`redis-cli`, RedisInsight) ซึ่งไม่ได้ออกแบบมาเพื่อดู "สุขภาพคิวงาน" โดยเฉพาะ |

### 762.5 ตารางสรุปภาพรวม: เมื่อไหร่ควรเลือกอันไหน

| สถานการณ์ | ตัวเลือกที่เหมาะกว่า |
|---|---|
| โปรเจกต์เล็ก-กลาง มี Redis อยู่แล้ว ต้องการเริ่ม Celery ให้เร็วที่สุด | **Redis** (เหมือนที่ Part 075 ทำ) |
| ต้องการ routing ซับซ้อนหลายเงื่อนไข, priority หลายระดับจริงจัง | **RabbitMQ** |
| ต้องการ Dead Letter Queue แบบ native โดยไม่อยากเขียนเอง | **RabbitMQ** |
| ทีมมีประสบการณ์ RabbitMQ/AMQP อยู่แล้วจากงานก่อนหน้า | **RabbitMQ** |
| งานส่วนใหญ่เป็น "fire and forget" ไม่ซับซ้อน อัตรางานสูงมาก (throughput สำคัญกว่า feature) | **Redis** |
| ระบบต้องการความทนทานสูงสุดระดับ enterprise (banking, payment processing) | **RabbitMQ** |
| ทีมเล็ก ไม่มีคนดูแล infrastructure เพิ่ม อยากลด operational overhead | **Redis** |

เราจะกลับมาที่ตารางนี้อีกครั้งพร้อมกรอบการตัดสินใจแบบเป็นระบบมากขึ้นใน
ขั้นตอนที่ 769 หลังจากได้ลงมือใช้ RabbitMQ จริงแล้วในขั้นตอนถัดไป

### 762.6 ตารางสรุปขั้นตอนที่ 762

| หัวข้อ | สรุป |
|---|---|
| ต้นกำเนิด | RabbitMQ ออกแบบมาเป็น message broker โดยเฉพาะ (AMQP), Redis เป็น data store ทั่วไปที่ถูกดัดแปลงมาใช้เป็น broker |
| Reliability | RabbitMQ แข็งแรงกว่าในเรื่อง durability, acknowledgment protocol, flow control |
| Feature | RabbitMQ มี routing/priority/DLQ แบบ native ที่ Redis ทำไม่ได้หรือทำได้จำกัด |
| Deploy | Redis เร็วกว่ามากถ้ามีอยู่แล้ว, RabbitMQ ต้องติดตั้ง service ใหม่ทั้งหมด |
| สรุปสั้น ๆ | Redis = เริ่มเร็ว เรียบง่าย, RabbitMQ = แข็งแรงกว่า ฟีเจอร์ครบกว่า แต่ซับซ้อนกว่า |

---

## ขั้นตอนที่ 763: ติดตั้งและตั้งค่า RabbitMQ สำหรับใช้กับ Celery

### 763.1 ติดตั้งบน Ubuntu/Debian

```bash
# ติดตั้ง RabbitMQ server ผ่าน apt (Ubuntu 22.04+/Debian 12+)
sudo apt update
sudo apt install rabbitmq-server -y

# ตรวจสอบว่า service ทำงานอยู่
sudo systemctl status rabbitmq-server
sudo systemctl enable rabbitmq-server   # ให้เริ่มทำงานอัตโนมัติทุกครั้งที่บูตเครื่อง
```

### 763.2 ติดตั้งบน macOS

```bash
# ติดตั้งผ่าน Homebrew (ทวนวิธีใช้ brew จาก Part 001 ขั้นตอนที่ 4.3)
brew install rabbitmq

# เริ่มทำงานเป็น background service
brew services start rabbitmq

# ตรวจสอบสถานะ
brew services list | grep rabbitmq
```

### 763.3 ติดตั้งด้วย Docker (แนะนำสำหรับเครื่องพัฒนา)

วิธีที่สะดวกที่สุดและตรงกับสภาพแวดล้อม production มากที่สุด (เพราะโปรเจกต์
จริงมักรัน RabbitMQ ผ่าน container อยู่แล้ว):

```bash
# image "management" มาพร้อม Web UI ในตัว (ขั้นตอนที่ 763.4) ไม่ต้องติดตั้งเพิ่ม
docker run -d \
  --name rabbitmq \
  -p 5672:5672 \
  -p 15672:15672 \
  -e RABBITMQ_DEFAULT_USER=guest \
  -e RABBITMQ_DEFAULT_PASS=guest \
  rabbitmq:3.13-management

# ตรวจสอบว่า container ทำงานอยู่
docker ps | grep rabbitmq
```

| Port | ใช้ทำอะไร |
|---|---|
| `5672` | AMQP protocol — พอร์ตหลักที่ Celery/Django เชื่อมต่อมาส่ง-รับ message |
| `15672` | Management UI (HTTP) — เข้าผ่านเบราว์เซอร์ที่ `http://localhost:15672` |

### 763.4 เปิดใช้งาน Management Plugin (ถ้าติดตั้งแบบไม่ใช้ Docker image `-management`)

```bash
sudo rabbitmq-plugins enable rabbitmq_management
# หรือบน macOS
rabbitmq-plugins enable rabbitmq_management
```

เข้า `http://localhost:15672` แล้ว login ด้วย user เริ่มต้น `guest`/`guest`
(ใช้ได้เฉพาะตอนเชื่อมต่อจาก `localhost` เท่านั้น — RabbitMQ บล็อก `guest`
ไม่ให้เชื่อมต่อจากเครื่องอื่นโดยอัตโนมัติเพื่อความปลอดภัย)

### 763.5 สร้าง vhost, User และ Permission เฉพาะโปรเจกต์

**ห้ามใช้ `guest/guest` ใน production เด็ดขาด** — สร้าง user และ **vhost**
(virtual host คือพื้นที่แยก namespace ของ exchange/queue ภายใน RabbitMQ
instance เดียว คล้ายกับแนวคิด "database" แยกกันของ PostgreSQL) เฉพาะสำหรับ
โปรเจกต์นี้:

```bash
# สร้าง vhost ชื่อ blog_vhost แยกจาก vhost อื่นที่อาจใช้ RabbitMQ instance เดียวกัน
sudo rabbitmqctl add_vhost blog_vhost

# สร้าง user ใหม่พร้อมรหัสผ่าน (เปลี่ยนรหัสผ่านนี้ก่อนใช้งานจริงเสมอ)
sudo rabbitmqctl add_user django_blog 'change-this-strong-password'

# ให้สิทธิ์ user นี้เข้าถึง vhost ที่สร้างไว้แบบเต็ม (configure, write, read)
sudo rabbitmqctl set_permissions -p blog_vhost django_blog ".*" ".*" ".*"

# ตรวจสอบรายการ user และ vhost ทั้งหมด
sudo rabbitmqctl list_users
sudo rabbitmqctl list_vhosts
sudo rabbitmqctl list_permissions -p blog_vhost
```

| Pattern ทั้ง 3 ค่าใน `set_permissions` | ความหมาย |
|---|---|
| `configure` (`.*`) | สิทธิ์สร้าง/ลบ queue, exchange |
| `write` (`.*`) | สิทธิ์ส่ง (publish) message เข้า exchange |
| `read` (`.*`) | สิทธิ์อ่าน (consume) message จาก queue |

`.*` หมายถึง "ทุกชื่อ queue/exchange" — ในระบบที่ต้องการความละเอียดสูงกว่านี้
สามารถจำกัด pattern ให้ user นี้เข้าถึงได้เฉพาะ queue ที่ขึ้นต้นด้วยชื่อ
โปรเจกต์เท่านั้น

### 763.6 ตั้งค่า `CELERY_BROKER_URL` ให้ชี้ไป RabbitMQ

```python
# config/settings.py
import os

# ── RabbitMQ Configuration ────────────────────────────────────────
RABBITMQ_HOST = os.environ.get('RABBITMQ_HOST', '127.0.0.1')
RABBITMQ_PORT = os.environ.get('RABBITMQ_PORT', '5672')
RABBITMQ_USER = os.environ.get('RABBITMQ_USER', 'django_blog')
RABBITMQ_PASSWORD = os.environ.get('RABBITMQ_PASSWORD', 'change-this-strong-password')
RABBITMQ_VHOST = os.environ.get('RABBITMQ_VHOST', 'blog_vhost')

# รูปแบบ URL มาตรฐานของ AMQP: amqp://user:password@host:port/vhost
CELERY_BROKER_URL = (
    f'amqp://{RABBITMQ_USER}:{RABBITMQ_PASSWORD}'
    f'@{RABBITMQ_HOST}:{RABBITMQ_PORT}/{RABBITMQ_VHOST}'
)

# Result Backend ยังคงเป็น Redis เหมือนเดิมจาก Part 075 ขั้นตอนที่ 742.6
# (เหตุผลว่าทำไมไม่ใช้ RabbitMQ เป็น result backend ด้วย อยู่ในขั้นตอนที่ 763.7)
REDIS_HOST = os.environ.get('REDIS_HOST', '127.0.0.1')
REDIS_PORT = os.environ.get('REDIS_PORT', '6379')
CELERY_RESULT_BACKEND = f'redis://{REDIS_HOST}:{REDIS_PORT}/5'
```

**สังเกตสิ่งสำคัญ**: เราเปลี่ยนแค่ `CELERY_BROKER_URL` จาก `redis://...`
เป็น `amqp://...` เท่านั้น — `CELERY_RESULT_BACKEND` ยังคงเป็น Redis เหมือน
เดิมทุกประการ เพราะ **Broker กับ Result Backend เป็นคนละหน้าที่กัน** (ทวนจาก
Part 075 ขั้นตอนที่ 745.1) และสามารถใช้เทคโนโลยีต่างกันได้อย่างอิสระ — นี่คือ
เหตุผลที่ Celery แยก config สองตัวนี้ออกจากกันตั้งแต่แรก

### 763.7 ทำไมไม่ใช้ RabbitMQ เป็น Result Backend ด้วยเลย

Celery รองรับการใช้ `rpc://` (ผ่าน RabbitMQ) เป็น result backend ได้ในทาง
เทคนิค แต่เอกสารทางการของ Celery เองแนะนำว่า**ไม่ควรทำในระบบที่มี task
จำนวนมาก**:

| ประเด็น | RabbitMQ เป็น Result Backend | Redis เป็น Result Backend |
|---|---|---|
| ธรรมชาติของข้อมูล | RabbitMQ ออกแบบมาเก็บ "message ที่รอถูกบริโภคครั้งเดียวแล้วหายไป" ไม่ใช่ "ค่าที่ query ซ้ำได้" | Redis เป็น key-value store โดยธรรมชาติ — เหมาะกับ "เก็บผลลัพธ์ไว้ให้ query ซ้ำได้ตาม task_id" พอดี |
| Query ผลลัพธ์ซ้ำหลายครั้ง (`AsyncResult(task_id).status` เรียกซ้ำได้) | ⚠️ ทำได้ยากกว่า/มีข้อจำกัด | ✅ ทำได้ตรงไปตรงมา |
| ผลกระทบต่อ RabbitMQ เมื่อมี task จำนวนมาก | เพิ่ม queue จำนวนมากสำหรับเก็บผลลัพธ์ กระทบ performance ของ broker เอง | ไม่กระทบ broker เลยเพราะแยก instance/DB index กัน |

**สรุปการตั้งค่าที่แนะนำเสมอ**: ใช้ RabbitMQ เป็น **Broker เท่านั้น** และใช้
Redis เป็น **Result Backend เท่านั้น** — นี่คือรูปแบบ (pattern) ที่นิยมที่สุด
ในระบบ production จริงเมื่อเลือกใช้ RabbitMQ

### 763.8 ทดสอบว่า Celery เชื่อมต่อ RabbitMQ ได้จริง

```bash
# ติดตั้ง library สำหรับคุยโปรโตคอล AMQP (kombu ใช้ตัวนี้เป็น transport)
pip install librabbitmq  # ถ้าติดตั้งไม่ได้ (เช่นบน Windows) ใช้ pyamqp แทนได้เลย ไม่ต้องตั้งค่าเพิ่ม
pip freeze | grep -iE "celery|amqp|kombu" >> requirements.txt
```

```bash
python manage.py shell
```

```python
>>> from config.celery import app
>>> conn = app.connection()
>>> conn.connect()
>>> conn.connected
True   # เชื่อมต่อ RabbitMQ สำเร็จ
>>> conn.release()
```

ตรวจสอบผ่าน `rabbitmqctl` ว่า connection ถูกสร้างจริง:

```bash
sudo rabbitmqctl list_connections
# แสดง connection ที่กำลังเปิดอยู่ รวมถึงจาก Django shell ที่เพิ่งทดสอบ
```

### 763.9 ตารางสรุปขั้นตอนที่ 763

| หัวข้อ | สรุป |
|---|---|
| ติดตั้ง | `apt install rabbitmq-server` / `brew install rabbitmq` / Docker image `rabbitmq:3.13-management` |
| Management UI | เปิดใช้ผ่าน `rabbitmq_management` plugin, เข้าที่ `http://localhost:15672` |
| ความปลอดภัย | ห้ามใช้ `guest/guest` ใน production — สร้าง vhost + user + permission เฉพาะโปรเจกต์เสมอ |
| `CELERY_BROKER_URL` | เปลี่ยนจาก `redis://` เป็น `amqp://user:password@host:port/vhost` |
| `CELERY_RESULT_BACKEND` | **ยังคงเป็น Redis เหมือนเดิม** — ไม่แนะนำให้ RabbitMQ ทำหน้าที่ result backend ด้วย |

---

## ขั้นตอนที่ 764: แนวคิด Exchange/Queue/Binding ของ RabbitMQ (AMQP Protocol)

### 764.1 สามองค์ประกอบหลักที่ทำให้ RabbitMQ ต่างจาก Redis List

Redis List (ที่ Celery ใช้จำลอง queue ในขั้นตอนก่อนหน้า) มีแค่แนวคิดเดียว:
"push เข้าคิว, pop ออกจากคิว" ตรงไปตรงมา แต่ AMQP (โปรโตคอลที่ RabbitMQ พูด)
มีองค์ประกอบ 3 ส่วนที่ทำงานร่วมกัน:

```
┌───────────┐  publish   ┌──────────────┐  binding   ┌───────────┐  consume  ┌──────────┐
│  Producer │ ──────────>│   Exchange   │───────────>│   Queue   │──────────>│ Consumer │
└───────────┘            └──────────────┘  (routing  └───────────┘           └──────────┘
                                             key จับคู่)
```

| องค์ประกอบ | หน้าที่ |
|---|---|
| **Exchange** | จุดแรกที่ message เข้ามาจาก Producer — **ไม่เก็บ message ไว้เอง** มีหน้าที่แค่ "ตัดสินใจว่าจะส่งต่อไปคิวไหน" ตามกฎ routing |
| **Queue** | ที่เก็บ message จริง ๆ รอให้ Consumer มาหยิบไปทำ (เทียบเท่ากับ Redis List ที่ Celery ใช้ในขั้นตอนก่อนหน้า) |
| **Binding** | กฎที่เชื่อม Exchange เข้ากับ Queue พร้อม **Routing Key** — บอกว่า message แบบไหนควรถูกส่งไปคิวไหน |

**ประเด็นสำคัญที่สุด**: Producer **ไม่เคยส่ง message ตรงไปที่ Queue เลย** —
ส่งไปที่ Exchange เสมอ แล้ว Exchange เป็นคนตัดสินใจว่าจะส่งต่อไปคิวไหนบ้าง
ตาม Binding ที่ตั้งไว้ — นี่คือจุดที่ทำให้ RabbitMQ ทำ routing ที่ซับซ้อนได้
มากกว่า Redis List ธรรมดา

### 764.2 ประเภทของ Exchange ทั้ง 4 แบบ

```
1. Direct Exchange              2. Fanout Exchange
   routing_key ตรงกันเป๊ะ           ส่งไปทุก Queue ที่ bind ไว้ (ไม่สนใจ routing key)

   Producer──>[Exchange]──"high"──>[Queue A]        Producer──>[Exchange]──>[Queue A]
                       └──"low"───>[Queue B]                            └──>[Queue B]
                                                                          └──>[Queue C]

3. Topic Exchange                4. Headers Exchange
   routing_key แบบ wildcard          จับคู่ตาม header attribute แทน routing_key
   (*, #)                            (ใช้น้อยที่สุดในทางปฏิบัติ)

   "order.created.*" ──> [Queue A]
   "order.#"         ──> [Queue B]
```

| ประเภท Exchange | ใช้เมื่อไหร่ | ตัวอย่างการใช้งาน |
|---|---|---|
| **Direct** | ต้องการ routing key ตรงเป๊ะ 1 ต่อ 1 หรือ 1 ต่อหลาย queue | **นี่คือแบบที่ Celery ใช้เป็นค่าเริ่มต้นเสมอ** (ทวนจาก Part 076 ขั้นตอนที่ 756.3 ที่ตั้ง `high_priority`/`low_priority`) |
| **Fanout** | ต้องการ broadcast message เดียวไปทุกคิวที่สนใจ | ระบบแจ้งเตือนที่ต้องส่งให้หลาย service พร้อมกัน (log service, notification service, analytics service) |
| **Topic** | ต้องการ routing แบบ pattern matching ที่ยืดหยุ่นกว่า direct | ระบบ event-driven ที่มี event type หลากหลาย เช่น `order.created`, `order.cancelled`, `order.*` |
| **Headers** | ต้องการ routing ตามหลาย attribute พร้อมกันโดยไม่พึ่ง routing key เดียว | Use case พิเศษที่ routing key เดียวไม่พอสื่อความหมาย (พบน้อยในทางปฏิบัติ) |

### 764.3 Celery ใช้ Exchange/Queue/Binding อย่างไรเบื้องหลัง

ทวนโค้ดจาก Part 076 ขั้นตอนที่ 756.3 ที่คุณเขียนไปแล้ว (ตอนนั้นยังใช้ Redis
broker) — จริง ๆ แล้วโค้ดชุดนี้กำลังประกาศ Exchange/Queue/Binding แบบ AMQP
อยู่เบื้องหลังทั้งหมด แม้ตอนนั้นจะใช้ Redis (ซึ่ง kombu จำลอง concept นี้ให้
บางส่วน) พอเปลี่ยนมาใช้ RabbitMQ จริง โค้ดชุดเดิมนี้จะทำงานตรงตามสเปก AMQP
เป๊ะ ๆ ทันที:

```python
# config/settings.py
from kombu import Exchange, Queue

CELERY_TASK_QUEUES = (
    Queue('high_priority', Exchange('high_priority'), routing_key='high_priority'),
    Queue('low_priority', Exchange('low_priority'), routing_key='low_priority'),
    Queue('celery', Exchange('celery'), routing_key='celery'),  # คิวเริ่มต้น
)

CELERY_TASK_ROUTES = {
    'accounts.tasks.send_password_reset_email': {'queue': 'high_priority'},
    'accounts.tasks.send_login_otp': {'queue': 'high_priority'},
    'blog.tasks.gather_report_data': {'queue': 'low_priority'},
    'blog.tasks.render_report_pdf': {'queue': 'low_priority'},
}

CELERY_TASK_DEFAULT_QUEUE = 'celery'
CELERY_TASK_DEFAULT_EXCHANGE = 'celery'
CELERY_TASK_DEFAULT_ROUTING_KEY = 'celery'
```

เมื่อรันคำสั่งนี้กับ RabbitMQ จริง ตรวจสอบผ่าน `rabbitmqctl` ได้ว่า Exchange
และ Binding ถูกสร้างขึ้นจริงตามสเปก AMQP:

```bash
sudo rabbitmqctl list_exchanges -p blog_vhost
# แสดง: celery, high_priority, low_priority (พร้อม type = direct)

sudo rabbitmqctl list_queues -p blog_vhost name messages consumers
# แสดง: high_priority, low_priority, celery พร้อมจำนวน message ที่ค้างอยู่

sudo rabbitmqctl list_bindings -p blog_vhost
# แสดงว่า exchange "high_priority" ผูกกับ queue "high_priority"
# ด้วย routing_key "high_priority" (Direct Exchange — routing key ตรงกันเป๊ะ)
```

### 764.4 ประกาศ Queue พร้อมค่าตั้งต้นขั้นสูงด้วย `queue_arguments`

`kombu.Queue` รองรับการส่ง argument ระดับ AMQP เพิ่มเติมผ่าน `queue_arguments`
— นี่คือประตูสู่ฟีเจอร์ขั้นสูงของ RabbitMQ ที่ Redis ทำไม่ได้เลย เช่น
Dead Letter Exchange (เจาะลึกเต็มรูปแบบในขั้นตอนที่ 765) และ message TTL:

```python
# config/settings.py
from kombu import Exchange, Queue

CELERY_TASK_QUEUES = (
    Queue(
        'low_priority',
        Exchange('low_priority'),
        routing_key='low_priority',
        queue_arguments={
            # ป้องกันงานค้างในคิวนานเกินไปโดยไม่มีใครหยิบไปทำ (ขยายความในขั้นตอนที่ 765.7)
            'x-message-ttl': 3600000,   # 1 ชั่วโมง (หน่วยเป็นมิลลิวินาที)
            'x-dead-letter-exchange': 'dlx',
            'x-dead-letter-routing-key': 'dead_letter',
        },
    ),
)
```

### 764.5 เปรียบเทียบโมเดล AMQP กับโมเดล List ธรรมดาของ Redis

| ประเด็น | AMQP (RabbitMQ) | Redis List |
|---|---|---|
| จุดที่ Producer ส่งงานเข้าไป | Exchange (ตัวกลาง ไม่เก็บ message) | Queue (List) โดยตรง — ไม่มี "ตัวกลาง" แยกออกมา |
| ความยืดหยุ่นของ routing | สูงมาก (4 ประเภท exchange, wildcard matching) | ต่ำ — Producer ต้องรู้ชื่อ queue ปลายทางเป๊ะ ๆ เอง |
| Broadcast งานเดียวไปหลายคิว | ✅ ทำได้ทันทีด้วย Fanout Exchange | ❌ ต้อง push ซ้ำเข้าหลาย List เอง |
| ความซับซ้อนในการเข้าใจ | สูงกว่า (ต้องเข้าใจ 3 concept ประกอบกัน) | ต่ำกว่ามาก (push/pop ตรงไปตรงมา) |

### 764.6 ตารางสรุปขั้นตอนที่ 764

| หัวข้อ | สรุป |
|---|---|
| องค์ประกอบหลักของ AMQP | Exchange (ตัดสินใจ routing) + Queue (เก็บ message) + Binding (กฎเชื่อมสองอย่างแรก) |
| ประเภท Exchange | Direct (ตรงเป๊ะ), Fanout (broadcast), Topic (wildcard), Headers (attribute) |
| Exchange ที่ Celery ใช้เป็นค่าเริ่มต้น | Direct Exchange เสมอ |
| `queue_arguments` | ประตูสู่ฟีเจอร์ระดับ AMQP เช่น TTL, Dead Letter Exchange, max length |
| ข้อได้เปรียบเหนือ Redis | Routing ยืดหยุ่นกว่ามาก โดยเฉพาะเมื่อต้องการ broadcast หรือ pattern matching |

---

## ขั้นตอนที่ 765: รูปแบบ Dead Letter Queue สำหรับ Message ที่ประมวลผลไม่สำเร็จ

### 765.1 ปัญหา: Task ที่ล้มเหลวหลัง Retry ครบแล้วหายไปไหน

ทวนจาก Part 075 ขั้นตอนที่ 746 — เมื่อ Task ตั้ง `max_retries=5` และล้มเหลว
ครบ 5 ครั้ง สถานะจะกลายเป็น `FAILURE` ถาวร และ `on_failure` hook (ทวนจาก
Part 075 ขั้นตอนที่ 749.2) จะถูกเรียก — แต่คำถามคือ: **message ต้นฉบับที่อยู่
ใน Broker หายไปไหน?** คำตอบคือ Celery **ack (ยืนยันรับงานแล้ว) message นั้น
ไปเรียบร้อยตั้งแต่ตอนที่ task เริ่มทำงานหรือจบการทำงาน** (ขึ้นกับ
`task_acks_late` — เจาะลึกในขั้นตอนที่ 766) — message หายไปจาก broker โดย
สมบูรณ์ ไม่มีการเก็บสำเนาไว้ที่ไหนเลยนอกจาก log ที่คุณเขียนเองใน `on_failure`

นี่คือปัญหาจริงในระบบ production: **ถ้าไม่ได้ตั้งใจเก็บ payload ของงานที่
ล้มเหลวไว้ที่ไหนสักแห่ง คุณจะไม่มีทาง "ลองทำใหม่" งานนั้นได้อีกเลย** แม้จะ
แก้ต้นเหตุของปัญหาแล้วก็ตาม (เช่น SMTP server กลับมาทำงานปกติแล้ว แต่งานที่
เคยล้มเหลวไม่มีทางกลับมาอีก)

### 765.2 แนวคิด Dead Lettering ใน AMQP

**Dead Letter Queue (DLQ)** คือคิวพิเศษที่เก็บ message ที่ **"ตายแล้ว"**
(dead) ในความหมายที่ว่า Consumer ปกติไม่สามารถ/ไม่ควรประมวลผลมันได้อีกต่อไป
แต่แทนที่จะทิ้งไปเฉย ๆ ระบบจะย้าย message นั้นไปเก็บไว้ที่คิวสำรองแทน เพื่อ
ให้มนุษย์ (หรือระบบอื่น) มาตรวจสอบทีหลังได้:

```
คิวปกติ (low_priority)              Dead Letter Exchange (dlx)         DLQ (dead_letter_queue)
┌─────────────────┐                 ┌──────────────────┐               ┌──────────────────┐
│ message ถูก      │  dead-letter    │                   │  route ต่อ     │  message รอ       │
│ reject/nack/     │ ──────────────> │  dlx (exchange)  │──────────────>│  ให้ทีมมาตรวจสอบ  │
│ TTL หมดอายุ      │                 │                   │               │  ด้วยตนเอง        │
└─────────────────┘                 └──────────────────┘               └──────────────────┘
```

### 765.3 สิ่งที่ทำให้ Message ถูก Dead-Letter โดยอัตโนมัติใน RabbitMQ

RabbitMQ ย้าย message ไปยัง Dead Letter Exchange **โดยอัตโนมัติ**เมื่อเกิด
เหตุการณ์ใดเหตุการณ์หนึ่งต่อไปนี้กับคิวที่ตั้งค่า `x-dead-letter-exchange`
ไว้ (ทวนจากขั้นตอนที่ 764.4):

| เหตุการณ์ | อธิบาย |
|---|---|
| **Message ถูก reject/nack แบบ `requeue=False`** | Consumer บอก broker ว่า "ฉันไม่เอา message นี้แล้ว และไม่ต้องส่งกลับมาให้ใครใหม่" |
| **Message หมดอายุตาม TTL** (`x-message-ttl`) | ไม่มี consumer มาหยิบไปทำภายในเวลาที่กำหนด (ทวนจากขั้นตอนที่ 764.4) |
| **Queue เต็มตาม `x-max-length`** | Message เก่าสุดในคิวถูกดันออกเพื่อให้ที่ว่างสำหรับ message ใหม่ |
| **Queue ถูกลบทั้งที่ยังมี message ค้างอยู่** | (พบน้อยในทางปฏิบัติ) |

**ข้อสังเกตสำคัญ**: ทั้ง 4 กรณีนี้เป็นกลไกระดับ **queue/broker** ล้วน ๆ — ไม่
เกี่ยวกับว่า "ผลลัพธ์ทาง business logic ของ task สำเร็จหรือล้มเหลว" เลย
กรณีที่ 1 (reject/nack) ใกล้เคียงกับสิ่งที่เราต้องการมากที่สุด แต่ Celery
เองไม่ได้ nack message โดยอัตโนมัติเมื่อ task ล้มเหลวถาวร — นำไปสู่หัวข้อ
ถัดไป

### 765.4 ทำไม Celery ไม่ Dead-Letter ให้อัตโนมัติเมื่อ Retry หมด

Celery ออกแบบมาให้ **"retry หมด" กับ "message หายไปจาก broker" เป็นเหตุการณ์
เดียวกัน** — เมื่อ task ทำงาน (ไม่ว่าจะสำเร็จหรือ raise exception จนกลาย
เป็น `FAILURE` ถาวร) Celery จะ **ack** message นั้นตามปกติ (การ "ทำงานจบ"
กับ "ทำงานสำเร็จ" เป็นคนละแนวคิดกันในมุมของ broker) เพราะ Celery มองว่า
"งานนี้ถูกจัดการแล้วโดยระบบ retry ของตัวเอง จบสมบูรณ์แล้วไม่ว่าผลจะเป็น
อย่างไร" — จึงไม่ได้ nack message เพื่อส่งเข้า DLQ ระดับ broker ให้อัตโนมัติ

**ทางออกที่ใช้กันจริงในทางปฏิบัติ**: สร้าง **Application-level Dead Letter
Queue** เอง โดยใช้จุดที่ Celery มีให้พอดีคือ `on_failure` hook (ทวนจาก Part
075 ขั้นตอนที่ 749.2) — แทนที่จะแค่ log อย่างเดียว ให้ส่ง payload ของ task
ที่ล้มเหลวไปเก็บไว้ในคิวพิเศษต่างหากด้วย

### 765.5 สร้าง Application-level DLQ ด้วย `on_failure` Hook

```python
# core/celery_tasks.py (ต่อยอดจาก LoggingTask ของ Part 075 ขั้นตอนที่ 749.2)
import logging

from celery import Task

logger = logging.getLogger(__name__)


class DeadLetterTask(Task):
    """
    Base Task ที่นอกจาก log แล้ว ยังส่ง payload ของ task ที่ล้มเหลวถาวร
    (หลัง retry ครบตาม max_retries แล้ว) ไปเก็บไว้ใน Dead Letter Queue
    เพื่อให้ทีมมาตรวจสอบและสั่งรันซ้ำได้ในภายหลัง แก้ปัญหาที่ขั้นตอนที่ 765.1
    อธิบายไว้ — message หายไปเฉย ๆ โดยไม่มีทางกู้คืน
    """

    def on_failure(self, exc, task_id, args, kwargs, einfo):
        logger.error(
            'Task ล้มเหลวถาวร กำลังส่งเข้า Dead Letter Queue: '
            'name=%s task_id=%s args=%s kwargs=%s error=%s',
            self.name, task_id, args, kwargs, exc, exc_info=einfo,
        )
        # ส่ง task ใหม่ไปที่คิว 'dead_letter' โดยเฉพาะ พร้อมข้อมูลครบถ้วน
        # เพื่อให้สามารถสั่งรันงานเดิมซ้ำได้ทุกเมื่อในอนาคต
        record_dead_letter_task.apply_async(
            kwargs={
                'original_task_name': self.name,
                'original_task_id': task_id,
                'original_args': args,
                'original_kwargs': kwargs,
                'error_message': str(exc),
            },
            queue='dead_letter',
        )
        super().on_failure(exc, task_id, args, kwargs, einfo)
```

```python
# core/tasks.py
import logging

from celery import shared_task
from django.utils import timezone

logger = logging.getLogger(__name__)


@shared_task(ignore_result=True)
def record_dead_letter_task(
    original_task_name, original_task_id, original_args, original_kwargs, error_message
):
    """
    Task ที่ทำหน้าที่เพียงอย่างเดียว: บันทึกงานที่ล้มเหลวถาวรลงฐานข้อมูล
    เพื่อให้ทีมดูรายการทั้งหมดผ่าน Django Admin และตัดสินใจว่าจะสั่งรันซ้ำ
    (reprocess) งานไหนบ้าง — รันบน queue แยก 'dead_letter' เสมอ
    เพื่อไม่ให้แย่ง worker กับงานปกติ
    """
    from core.models import DeadLetterRecord

    DeadLetterRecord.objects.create(
        task_name=original_task_name,
        task_id=original_task_id,
        args=original_args,
        kwargs=original_kwargs,
        error_message=error_message,
        failed_at=timezone.now(),
    )
    logger.warning('บันทึก Dead Letter สำหรับ task %s (%s) แล้ว', original_task_name, original_task_id)
```

```python
# core/models.py
from django.db import models


class DeadLetterRecord(models.Model):
    """เก็บประวัติ task ที่ล้มเหลวถาวร เพื่อให้ทีมตรวจสอบและสั่งรันซ้ำได้ภายหลัง"""

    task_name = models.CharField(max_length=255)
    task_id = models.CharField(max_length=255)
    args = models.JSONField(default=list)
    kwargs = models.JSONField(default=dict)
    error_message = models.TextField()
    failed_at = models.DateTimeField()
    reprocessed_at = models.DateTimeField(null=True, blank=True)

    class Meta:
        ordering = ['-failed_at']

    def __str__(self):
        return f'{self.task_name} ({self.task_id}) — {self.failed_at:%Y-%m-%d %H:%M}'
```

### 765.6 ตรวจสอบและ Reprocess งานใน DLQ ผ่าน Django Admin

```python
# core/admin.py
from django.contrib import admin
from django.utils import timezone

from .models import DeadLetterRecord
from celery import current_app


@admin.register(DeadLetterRecord)
class DeadLetterRecordAdmin(admin.ModelAdmin):
    list_display = ('task_name', 'task_id', 'failed_at', 'reprocessed_at')
    list_filter = ('task_name', 'reprocessed_at')
    actions = ['reprocess_selected']

    @admin.action(description='สั่งรันงานที่เลือกซ้ำอีกครั้ง (Reprocess)')
    def reprocess_selected(self, request, queryset):
        count = 0
        for record in queryset.filter(reprocessed_at__isnull=True):
            # ใช้ send_task แทนการ import task โดยตรง เพราะไม่รู้ล่วงหน้าว่า
            # task_name ที่เก็บไว้จะเป็นของ app ไหน — send_task หาให้อัตโนมัติ
            # จากชื่อที่ลงทะเบียนไว้ตอน autodiscover_tasks()
            current_app.send_task(
                record.task_name, args=record.args, kwargs=record.kwargs
            )
            record.reprocessed_at = timezone.now()
            record.save(update_fields=['reprocessed_at'])
            count += 1
        self.message_user(request, f'สั่งรันงานซ้ำแล้ว {count} รายการ')
```

Flow ที่สมบูรณ์ตอนนี้: Task ล้มเหลว → retry ครบ → `on_failure` บันทึกเข้า
DLQ → ทีมเปิด Django Admin เห็นรายการ → กด action "Reprocess" → งานถูกส่ง
กลับเข้าคิวปกติอีกครั้งด้วย argument ชุดเดิมเป๊ะ

### 765.7 DLQ แบบ Native เสริมอีกชั้น: ป้องกันงานค้างในคิวหลักนานเกินไป

Application-level DLQ ในขั้นตอนที่ 765.5 จัดการกรณี "Celery retry ครบแล้ว
ล้มเหลว" แต่ยังมีอีกกรณีที่ native DLQ ของ RabbitMQ (ทวนจากขั้นตอนที่ 764.4)
จัดการได้ดีกว่า: **งานที่ไม่มี Worker มาหยิบไปทำเลยเป็นเวลานาน** (เช่น
Worker ทุกตัวของ queue นั้นล่มพร้อมกันหมด) — กรณีนี้ Celery ไม่รู้ตัวด้วยซ้ำ
ว่ามีปัญหา เพราะ task ยังไม่เคยเริ่มทำงานเลย:

```python
# config/settings.py — เพิ่ม Dead Letter Exchange/Queue แบบ native ของ RabbitMQ
from kombu import Exchange, Queue

dlx_exchange = Exchange('dlx', type='direct')

CELERY_TASK_QUEUES = (
    Queue(
        'low_priority',
        Exchange('low_priority'),
        routing_key='low_priority',
        queue_arguments={
            'x-message-ttl': 3600000,               # ค้างเกิน 1 ชั่วโมง = dead-letter
            'x-dead-letter-exchange': 'dlx',
            'x-dead-letter-routing-key': 'expired',
        },
    ),
    # คิวปลายทางสำหรับ message ที่หมดอายุ (แยกจาก DLQ ระดับ application ในขั้นตอนที่ 765.5)
    Queue('expired_low_priority', dlx_exchange, routing_key='expired'),
)
```

**สองชั้นของ DLQ ในระบบนี้ทำหน้าที่ต่างกัน**: ชั้น native (ระดับ broker)
ป้องกันงานที่ไม่มีใครหยิบไปทำเลย ส่วนชั้น application (`DeadLetterRecord`)
ป้องกันงานที่ Worker หยิบไปทำแล้วแต่ล้มเหลวถาวรทาง business logic — ทั้งสอง
ชั้นเสริมกันเพื่อให้ไม่มี message ไหนหายไปแบบไร้ร่องรอยอีกเลย

### 765.8 Redis ไม่มี DLQ มาให้ในตัว

ถ้ายังใช้ Redis เป็น broker (ทวนจาก Part 075) การจำลอง DLQ ต้องทำในระดับ
application ทั้งหมด (เหมือนขั้นตอนที่ 765.5) เพราะ Redis List ไม่มีแนวคิด
`x-dead-letter-exchange` หรือ TTL ระดับ message ให้ใช้เลย — นี่คือหนึ่งใน
ข้อได้เปรียบที่ชัดเจนที่สุดของ RabbitMQ เหนือ Redis ในแง่ฟีเจอร์ (ทวนจาก
ขั้นตอนที่ 762.3)

### 765.9 ตารางสรุปขั้นตอนที่ 765

| หัวข้อ | สรุป |
|---|---|
| ปัญหาที่แก้ | Task ที่ล้มเหลวถาวรหลัง retry ครบ ไม่มีทางกู้คืน/สั่งรันซ้ำได้เลยถ้าไม่เก็บไว้ |
| ทำไม Celery ไม่ dead-letter อัตโนมัติ | Celery มองว่า "retry ครบแล้ว" คือ task จบสมบูรณ์แล้ว จึง ack message ตามปกติ |
| วิธีแก้ (Application-level) | สร้าง `DeadLetterTask` base class ที่ส่ง payload เข้าคิว `dead_letter` ผ่าน `on_failure` |
| วิธีแก้เสริม (Native RabbitMQ) | `x-dead-letter-exchange` + `x-message-ttl` ป้องกันงานที่ไม่มี Worker มาหยิบเลย |
| Redis | ไม่มี native DLQ ต้องทำ application-level เท่านั้น |

---

## ขั้นตอนที่ 766: Message Acknowledgment และการรับประกันความน่าเชื่อถือ

### 766.1 Acknowledgment (Ack) คืออะไร และทำไมสำคัญที่สุดข้อหนึ่งของ Broker

**Acknowledgment (ack)** คือสัญญาณที่ Consumer ส่งกลับไปบอก Broker ว่า
"ฉันได้รับ/ประมวลผล message นี้เรียบร้อยแล้ว ลบมันออกจากคิวได้เลย" —
จังหวะเวลาที่ Worker ส่ง ack กลับไปนี่เองที่เป็นตัวกำหนดว่าระบบของคุณมี
พฤติกรรมแบบ **at-most-once** หรือ **at-least-once**

### 766.2 `task_acks_late`: Early Ack เทียบกับ Late Ack

```python
# config/settings.py
CELERY_TASK_ACKS_LATE = True  # ค่าเริ่มต้นคือ False (early ack)
```

```
Early Ack (ค่าเริ่มต้น, task_acks_late=False):
Broker ส่งงาน ──> Worker "รับทราบทันที" (ack) ──> Worker เริ่มทำงานจริง ──> (สำเร็จ/ล้มเหลว)
                        ▲
                  ack เกิดขึ้น ณ จุดนี้ — ก่อนที่งานจะเริ่มทำจริงด้วยซ้ำ

Late Ack (task_acks_late=True):
Broker ส่งงาน ──> Worker เริ่มทำงานจริง ──> ทำงานเสร็จ (สำเร็จ) ──> Worker "รับทราบ" (ack)
                                                                          ▲
                                                                    ack เกิดขึ้น ณ จุดนี้
```

| สถานการณ์ | Early Ack (`False`) | Late Ack (`True`) |
|---|---|---|
| Worker process ถูกฆ่ากะทันหัน (`kill -9`) **ระหว่าง**ทำงาน | Message ถูก ack ไปแล้วก่อนเริ่มทำงาน → **หายไปตลอดกาล** ไม่มีใครมาทำงานนี้อีกเลย | Message ยังไม่ถูก ack → Broker **ส่งงานนี้ไปให้ Worker ตัวอื่นทำใหม่โดยอัตโนมัติ** |
| Worker deploy โค้ดใหม่/restart แบบ graceful | ไม่กระทบ (ack ไปแล้วตั้งแต่ต้น) | Worker ที่ตั้งค่า graceful shutdown จะรอทำงานปัจจุบันให้เสร็จก่อน ack (ทวนแนวคิดจาก Part 075 ขั้นตอนที่ 741.5) |
| ความเสี่ยงเรื่อง Duplicate Processing | ต่ำมาก (งานถูกทำแค่ครั้งเดียวเสมอ แม้จะเสี่ยงหายก็ตาม) | มีความเสี่ยง — ถ้า Worker ทำงานเสร็จ**เกือบ**จะ ack แต่ crash พอดีก่อน ack สำเร็จ Broker จะส่งงานเดิมไปให้ Worker ตัวอื่นทำ**ซ้ำอีกครั้ง** |
| ชื่อเรียกทางเทคนิค | **At-most-once** (ทำได้สูงสุด 1 ครั้ง อาจ 0 ครั้งถ้าหาย) | **At-least-once** (ทำได้อย่างน้อย 1 ครั้ง อาจมากกว่านั้นถ้าเกิด duplicate) |

### 766.3 ตารางสรุป At-most-once vs At-least-once vs Exactly-once

| ระดับการรับประกัน | ความหมาย | Celery รองรับไหม |
|---|---|---|
| **At-most-once** | งานถูกทำ 0 หรือ 1 ครั้ง (อาจไม่ถูกทำเลยถ้าเกิดปัญหา แต่ไม่มีทางถูกทำซ้ำ) | ✅ (ค่าเริ่มต้น `task_acks_late=False`) |
| **At-least-once** | งานถูกทำอย่างน้อย 1 ครั้งเสมอ (อาจถูกทำซ้ำมากกว่า 1 ครั้งได้ในบางสถานการณ์) | ✅ (`task_acks_late=True`) |
| **Exactly-once** | งานถูกทำเป๊ะ ๆ 1 ครั้งเท่านั้น ไม่มากไม่น้อย | ❌ **Celery ไม่รองรับโดยตรง** — ในทางทฤษฎีการรับประกัน exactly-once แบบสมบูรณ์ในระบบ distributed ทำได้ยากมาก (ต้องพึ่ง idempotency ที่ระดับ business logic เข้าช่วยเสมอ) |

**คำแนะนำมาตรฐานของหลักสูตรนี้**: ตั้ง `CELERY_TASK_ACKS_LATE = True` เสมอ
สำหรับ task ที่สำคัญ (ส่งอีเมล, บันทึกธุรกรรม) เพื่อไม่ให้งานหายเมื่อ Worker
ล่มกลางทาง แต่ **ต้องออกแบบ task ให้ทำซ้ำได้อย่างปลอดภัย (idempotent)**
เสมอเป็นเงื่อนไขคู่กัน — ทวนแนวคิด Idempotency เต็มรูปแบบจาก Part 076
ขั้นตอนที่ 758

### 766.4 `task_reject_on_worker_lost`: ปิดช่องโหว่ของ Late Ack

```python
# config/settings.py
CELERY_TASK_ACKS_LATE = True
CELERY_TASK_REJECT_ON_WORKER_LOST = True
```

มีสถานการณ์พิเศษที่ `task_acks_late=True` เพียงอย่างเดียวยังไม่ครอบคลุม:
เมื่อ **child process ของ Worker ถูกฆ่ากะทันหันโดยระบบปฏิบัติการ** (เช่น
Linux OOM Killer ฆ่า process เพราะเครื่องขาด memory) โดยไม่มีโอกาสส่ง signal
อะไรกลับไปที่ broker เลย ค่าเริ่มต้นของ Celery ในกรณีนี้คือ**ถือว่า task
สำเร็จแล้ว** (เพราะไม่รู้ว่าเกิดอะไรขึ้นจริง ๆ) ซึ่งอันตรายมาก —
`task_reject_on_worker_lost=True` เปลี่ยนพฤติกรรมนี้ให้ **reject message
กลับไปที่ broker ทันทีที่ตรวจพบว่า worker หายไปโดยไม่ทราบสาเหตุ** ทำให้
message ถูกส่งไปให้ worker ตัวอื่นทำใหม่แทนที่จะถูกทิ้งเป็น "สำเร็จ" อย่าง
ผิด ๆ

### 766.5 กลไก Ack จำลองของ Redis Broker: `visibility_timeout`

Redis ไม่มี ack protocol แท้แบบ AMQP (ทวนจากขั้นตอนที่ 762.2) — kombu จำลอง
พฤติกรรมนี้ด้วยเทคนิคที่เรียกว่า **visibility timeout**:

```python
# config/settings.py — ใช้ได้เฉพาะตอน broker เป็น Redis เท่านั้น
CELERY_BROKER_TRANSPORT_OPTIONS = {
    'visibility_timeout': 3600,  # หน่วยวินาที (1 ชั่วโมง)
}
```

กลไกเบื้องหลัง: เมื่อ Worker หยิบ message จาก Redis List ไปทำ kombu จะย้าย
message นั้นไปเก็บไว้ใน **"unacked" sorted set** ชั่วคราวพร้อม timestamp
ถ้า Worker ack สำเร็จ (ทำงานเสร็จ) message จะถูกลบออกจาก unacked set ถาวร
แต่ถ้าผ่านไปนานเกิน `visibility_timeout` วินาทีแล้ว**ยังไม่มีการ ack**
(เช่น Worker crash) kombu จะเอา message นั้น**กลับเข้าคิวหลักอีกครั้ง**ให้
Worker ตัวอื่นมาหยิบไปทำใหม่

| ประเด็น | RabbitMQ (AMQP native) | Redis (จำลองผ่าน visibility_timeout) |
|---|---|---|
| ตรวจจับ worker ล่มได้เร็วแค่ไหน | เกือบทันที (TCP connection หลุดทันทีที่ worker ตาย) | ต้องรอจนครบ `visibility_timeout` เต็มจำนวนก่อนถึงจะ requeue |
| ความเสี่ยง | ต่ำ — Broker รู้สถานะ consumer ตลอดเวลาผ่าน connection | ถ้าตั้ง `visibility_timeout` สั้นเกินไปสำหรับ task ที่ใช้เวลานาน อาจถูก requeue **ทั้งที่ยังทำงานอยู่** เกิด duplicate execution ทันที |
| คำแนะนำ | ใช้ค่าเริ่มต้นได้เลย | ต้องตั้ง `visibility_timeout` ให้**มากกว่า**เวลาที่ task ที่ใช้เวลานานที่สุดในระบบใช้จริงเสมอ (ทวนแนวคิด `CELERY_TASK_TIME_LIMIT` จาก Part 075 ขั้นตอนที่ 742.6 ประกอบการตั้งค่านี้) |

นี่คือตัวอย่างที่ชัดเจนอีกข้อว่าทำไม RabbitMQ ถึงมีความน่าเชื่อถือ
(reliability) สูงกว่า Redis ในฐานะ broker (ทวนตารางเปรียบเทียบจากขั้นตอนที่
762.2) — Redis ทำได้ แต่ต้องตั้งค่าให้ถูกต้องด้วยตัวเอง ไม่มี "ความปลอดภัย
โดยธรรมชาติ" เหมือน RabbitMQ

### 766.6 ทำไมต้องออกแบบ Task ให้ Idempotent เสมอ ไม่ว่าจะใช้ Broker ไหน

ไม่ว่าจะเลือก RabbitMQ หรือ Redis, ไม่ว่าจะตั้ง `task_acks_late` หรือไม่ —
**ตราบใดที่ระบบให้การรับประกันแค่ระดับ at-least-once (ไม่ใช่ exactly-once)
มีโอกาสเสมอที่ task หนึ่งจะถูกรันมากกว่า 1 ครั้ง** ด้วยเหตุผลต่าง ๆ (worker
crash จังหวะคาบเกี่ยว, network partition, ฯลฯ) — task ที่เขียนดีต้อง**ทำซ้ำ
ได้โดยไม่เกิดผลข้างเคียงซ้ำ** (ทวนหลักการเต็มรูปแบบจาก Part 076 ขั้นตอนที่
758):

```python
# ตัวอย่าง: Idempotent ด้วยการเช็คสถานะก่อนทำงานซ้ำ
@shared_task(bind=True)
def charge_customer_task(self, order_id):
    order = Order.objects.select_for_update().get(id=order_id)
    if order.payment_status == 'PAID':
        # ถ้างานนี้เคยถูกรันไปแล้วสำเร็จ (อาจเพราะ ack ช้าจนถูก requeue)
        # ให้จบการทำงานทันทีโดยไม่เก็บเงินซ้ำ
        return
    # ... เก็บเงินจริง แล้วอัปเดตสถานะ ...
    order.payment_status = 'PAID'
    order.save(update_fields=['payment_status'])
```

### 766.7 ตารางสรุปการตั้งค่าที่แนะนำสำหรับโปรเจกต์นี้

```python
# config/settings.py — ค่าตั้งต้นด้าน Reliability ที่แนะนำสำหรับ Part นี้เป็นต้นไป
CELERY_TASK_ACKS_LATE = True
CELERY_TASK_REJECT_ON_WORKER_LOST = True
CELERY_BROKER_TRANSPORT_OPTIONS = {'visibility_timeout': 3600}  # เผื่อกรณีสลับกลับไปใช้ Redis
```

| หัวข้อ | สรุป |
|---|---|
| `task_acks_late=False` (ค่าเริ่มต้น) | At-most-once — เสี่ยงงานหายเมื่อ worker ล่มกลางทาง แต่ไม่มี duplicate |
| `task_acks_late=True` | At-least-once — งานไม่หาย แต่ต้องออกแบบ task ให้ idempotent เสมอ |
| `task_reject_on_worker_lost=True` | ปิดช่องโหว่กรณี worker ถูกฆ่ากะทันหันโดยไม่มี signal (เช่น OOM Killer) |
| RabbitMQ ack | native ผ่าน AMQP protocol โดยตรง แม่นยำและเร็ว |
| Redis ack | จำลองผ่าน `visibility_timeout` — ต้องตั้งค่าให้เหมาะสมกับเวลาที่ task ใช้จริงเอง |
| กฎทอง | ไม่ว่าใช้ broker ไหน ออกแบบ task ให้ idempotent เสมอ เพราะไม่มี broker ไหนรับประกัน exactly-once ได้จริง |

---

## ขั้นตอนที่ 767: หลาย Worker และ Load Balancing แบบ Queue-based

### 767.1 Competing Consumers Pattern

เมื่อรัน Celery Worker หลายตัวโดยชี้ไปที่**คิวชื่อเดียวกัน** พฤติกรรมที่เกิด
ขึ้นเรียกว่า **Competing Consumers Pattern** — Broker จะแจกจ่าย message
แต่ละชิ้นให้ Worker แค่**ตัวเดียว**เท่านั้น (ไม่ใช่ทุกตัว เหมือน Pub/Sub)
โดยอัตโนมัติ ไม่ต้องเขียนโค้ด load balancing เองเลยแม้แต่บรรทัดเดียว:

```
                    ┌──────────────┐
                    │    Queue     │
                    │ [msg1, msg2, │
                    │  msg3, msg4] │
                    └──────┬───────┘
                           │  Broker แจกจ่ายแบบ round-robin
              ┌────────────┼────────────┐
              ▼            ▼            ▼
        ┌───────────┐┌───────────┐┌───────────┐
        │ Worker A  ││ Worker B  ││ Worker C  │
        │ (msg1)    ││ (msg2)    ││ (msg3)    │
        └───────────┘└───────────┘└───────────┘
```

```bash
# รัน Worker 3 ตัว (คนละ terminal หรือคนละเครื่อง) ชี้ไปคิวเดียวกัน "celery"
# hostname ต้องไม่ซ้ำกัน เพื่อให้แยกแยะได้ตอน monitor
celery -A config worker --loglevel=info --hostname=worker1@%h
celery -A config worker --loglevel=info --hostname=worker2@%h
celery -A config worker --loglevel=info --hostname=worker3@%h
```

**นี่คือคำตอบของคำถามที่ค้างจากขั้นตอนที่ 761.2**: View ไม่ต้องรู้เลยว่ามี
Worker กี่ตัว หรือ Worker ไหนว่าง — Broker จัดการการกระจายงานให้ทั้งหมด
เพิ่ม Worker เมื่อไหร่ก็ได้ระหว่างระบบทำงานอยู่ (แม้ไม่ต้อง restart Django
เลย) และระบบจะเริ่มใช้ Worker ตัวใหม่ทันทีที่มันเชื่อมต่อกับ Broker สำเร็จ

### 767.2 `worker_prefetch_multiplier`: ควบคุมความ "แฟร์" ของการแจกจ่ายงาน

```python
# config/settings.py
CELERY_WORKER_PREFETCH_MULTIPLIER = 1
```

ค่าเริ่มต้นของ Celery คือ `4` หมายความว่า **แต่ละ child process จะดึงงาน
ล่วงหน้า (prefetch) มาเก็บไว้ล่วงหน้าถึง 4 งาน** ก่อนที่จะเริ่มทำงานจริง —
ตั้งใจไว้เพื่อลด network round-trip ระหว่าง Worker กับ Broker แต่มีข้อเสีย
ชัดเจนเมื่อ task แต่ละงานใช้เวลาต่างกันมาก:

```
สถานการณ์ปัญหา (prefetch=4, มี Worker 2 ตัว, มีงาน 8 ชิ้นเข้าคิวพร้อมกัน):

Worker A: ดึงไป 4 งาน (msg1-4) ── กำลังทำงานแรกที่ใช้เวลานานมาก (10 นาที)
Worker B: ดึงไป 4 งาน (msg5-8) ── ทำงานเสร็จเร็วทุกงาน (2 วินาทีต่องาน)

ผลลัพธ์: Worker B ทำงานเสร็จหมดใน 8 วินาที แต่ msg1-4 ที่ค้างอยู่ที่ Worker A
ต้องรอ Worker A ทำทีละงานจนครบ แม้ Worker B จะว่างอยู่เฉย ๆ ก็ตาม!
```

| ค่า `worker_prefetch_multiplier` | พฤติกรรม | เหมาะกับ |
|---|---|---|
| `4` (ค่าเริ่มต้น) | ดึงงานล่วงหน้าเยอะ ลด network overhead | Task ที่ใช้เวลาใกล้เคียงกันทุกงาน (เช่น ส่งอีเมลทั่วไป) |
| `1` | ดึงทีละงาน ไม่ prefetch ล่วงหน้าเลย | **Task ที่ใช้เวลาต่างกันมาก** (บางงาน 1 วินาที บางงาน 10 นาที) — กระจายงานได้แฟร์กว่ามาก |

**คำแนะนำ**: ตั้ง `CELERY_WORKER_PREFETCH_MULTIPLIER = 1` เมื่อ Task ในระบบ
มีเวลาทำงานต่างกันมาก (เช่นในโปรเจกต์นี้ที่มีทั้ง `high_priority` เร็ว ๆ
และ `low_priority` ที่อาจเป็นงาน render PDF ที่ใช้เวลานาน — ทวนจาก Part 076
ขั้นตอนที่ 756)

### 767.3 Scale แนวนอน: เพิ่ม Worker กี่ตัวก็ได้โดยไม่แก้โค้ด

ทวนจากขั้นตอนที่ 767.1 — เพราะ Producer (Django View) ไม่รู้จัก Worker
โดยตรงเลย (หลักการ Decoupling จากขั้นตอนที่ 761.3) การเพิ่ม Worker จึงทำได้
โดยไม่ต้องแก้โค้ดฝั่ง Django แม้แต่บรรทัดเดียว — แค่รันคำสั่งเดิมซ้ำบนเครื่อง
ใหม่ หรือเพิ่ม container:

```bash
# เพิ่ม Worker บนเครื่องอื่น (เชื่อมต่อ broker ตัวเดียวกันผ่าน network)
# ทำได้แม้ระบบ production กำลังรันอยู่ ไม่ต้อง downtime เลย
ssh worker-server-2
source venv/bin/activate
celery -A config worker --loglevel=info --hostname=worker-server-2@%h
```

### 767.4 `--autoscale`: ปรับจำนวน Child Process อัตโนมัติตามปริมาณงาน

```bash
# --autoscale=max,min — ขยายสูงสุด 10 process เมื่องานเยอะ
# หดเหลือ 3 process เมื่องานน้อย (ประหยัด resource ตอนกลางคืนที่งานน้อย)
celery -A config worker --loglevel=info --autoscale=10,3
```

Celery จะเพิ่ม child process อัตโนมัติเมื่อคิวเริ่มมีงานค้างเยอะ และลดจำนวน
ลงเมื่องานเบาบางเพื่อประหยัด memory/CPU — เหมาะกับ Worker ที่รันบนเครื่อง
เดียวกับ service อื่นที่ต้องแบ่ง resource กัน (ทวนจาก Part 075 ขั้นตอนที่
744.3 เรื่อง `--concurrency` แบบคงที่)

### 767.5 ผสมกับ Task Routing: Worker เฉพาะทางต่อ Queue (ทวนจาก Part 076)

Competing Consumers ทำงานร่วมกับ Task Routing (Part 076 ขั้นตอนที่ 756) ได้
อย่างลงตัว — สามารถมี **หลาย Worker ต่อหนึ่งคิวเฉพาะทาง** เพื่อ scale แต่ละ
ประเภทงานแยกกันอย่างอิสระ:

```bash
# 3 worker สำหรับงานด่วน (ต้องการ throughput สูง response ไว)
celery -A config worker -Q high_priority --concurrency=8 --hostname=urgent-1@%h &
celery -A config worker -Q high_priority --concurrency=8 --hostname=urgent-2@%h &
celery -A config worker -Q high_priority --concurrency=8 --hostname=urgent-3@%h &

# 1 worker สำหรับงานหนักที่ไม่รีบ (ประหยัด resource, จำกัด concurrency ต่ำ)
celery -A config worker -Q low_priority --concurrency=2 --hostname=batch-1@%h &
```

ตารางสรุปการจัดสรร Worker ตาม Queue:

| Queue | จำนวน Worker | Concurrency ต่อตัว | เหตุผล |
|---|---|---|---|
| `high_priority` | 3 | 8 | งานด่วนต้องการ throughput สูงและ latency ต่ำ — เพิ่ม worker เผื่อ traffic พุ่ง |
| `low_priority` | 1 | 2 | งานหนัก (render PDF) ไม่รีบ จำกัด resource ไม่ให้แย่ง CPU จาก worker ด่วน |

### 767.6 ตัวอย่าง `docker-compose.yml` Scale หลาย Worker

```yaml
# docker-compose.yml (ส่วนที่เกี่ยวข้องกับ worker)
services:
  worker_urgent:
    build: .
    command: celery -A config worker -Q high_priority --concurrency=8 --loglevel=info
    deploy:
      replicas: 3   # รัน container นี้พร้อมกัน 3 ชุด — แต่ละชุดคือ Worker แยก 1 ตัว
    depends_on:
      - rabbitmq
      - redis

  worker_batch:
    build: .
    command: celery -A config worker -Q low_priority --concurrency=2 --loglevel=info
    deploy:
      replicas: 1
    depends_on:
      - rabbitmq
      - redis
```

```bash
# หรือสั่ง scale ทันทีแบบไม่ต้องแก้ไฟล์ (docker-compose แบบไม่ใช้ deploy.replicas)
docker compose up -d --scale worker_urgent=5
```

### 767.7 ตารางสรุปขั้นตอนที่ 767

| หัวข้อ | สรุป |
|---|---|
| Competing Consumers | Worker หลายตัวชี้คิวเดียวกัน = Broker แจกจ่ายงานให้อัตโนมัติแบบ round-robin |
| `worker_prefetch_multiplier=1` | แนะนำเมื่อ task มีเวลาทำงานต่างกันมาก ป้องกัน worker ว่างทั้งที่ยังมีงานค้าง |
| Scale แนวนอน | เพิ่ม Worker กี่ตัว/เครื่องไหนก็ได้ โดยไม่แก้โค้ด Django เลย |
| `--autoscale=max,min` | ปรับจำนวน child process อัตโนมัติตามปริมาณงาน ประหยัด resource ช่วงงานน้อย |
| ผสมกับ Task Routing | จัดสรรจำนวน Worker/concurrency ต่างกันตามความสำคัญของแต่ละ queue |

---

## ขั้นตอนที่ 768: การ Monitor ความลึกของ Queue (Queue Depth) และสุขภาพของระบบ

### 768.1 ทำไม Queue Depth คือสัญญาณสุขภาพอันดับหนึ่งของระบบ Message Queue

**Queue Depth** คือจำนวน message ที่ค้างอยู่ในคิว ยังไม่มี Consumer มาหยิบ
ไปทำ — นี่คือตัวชี้วัดที่สำคัญที่สุดตัวเดียวที่บอกได้ว่า **"Worker ตามงาน
ทันหรือไม่"**:

```
Queue Depth คงที่หรือลดลง         Queue Depth เพิ่มขึ้นเรื่อย ๆ ไม่หยุด
= ระบบปกติ ทุกอย่างสมดุลดี        = สัญญาณอันตราย! Worker ทำงานไม่ทัน
                                     ปริมาณงานเข้า > ความสามารถ Worker ที่มี
```

ถ้าปล่อยไว้โดยไม่สังเกต Queue Depth ที่พุ่งสูงขึ้นเรื่อย ๆ จะนำไปสู่ปัญหา
ลูกโซ่: ผู้ใช้รอนานขึ้นเรื่อย ๆ กว่าจะได้รับอีเมล/ผลลัพธ์ (แม้ View จะ
response เร็วตามปกติ แต่ผลลัพธ์จริงมาช้า), Memory ของ Broker บวมขึ้นเรื่อย ๆ
จนอาจถึงขั้น broker ล่มทั้งระบบได้ในที่สุด

### 768.2 ตรวจสอบ Queue Depth ผ่าน `rabbitmqctl`

```bash
# แสดงชื่อ queue, จำนวน message ทั้งหมด, จำนวน consumer ที่กำลังฟังอยู่
sudo rabbitmqctl list_queues -p blog_vhost name messages consumers

# ผลลัพธ์ตัวอย่าง
# name              messages    consumers
# high_priority     3           3
# low_priority      847         1        <── น่าเป็นห่วง! งานค้างเยอะ consumer มีแค่ 1
# celery            0           2
# dead_letter       12          1
```

```bash
# แยกดูรายละเอียดเพิ่มเติม: messages_ready (รอทำ) vs messages_unacknowledged (กำลังทำอยู่)
sudo rabbitmqctl list_queues -p blog_vhost name messages_ready messages_unacknowledged
```

| คอลัมน์ | ความหมาย |
|---|---|
| `messages` | จำนวน message ทั้งหมดในคิว (ready + unacknowledged) |
| `messages_ready` | message ที่**รอ**ให้ consumer มาหยิบ (ยังไม่มีใครทำ) |
| `messages_unacknowledged` | message ที่ consumer หยิบไปแล้ว **กำลังทำอยู่** ยังไม่ ack |
| `consumers` | จำนวน Worker ที่กำลังเชื่อมต่อฟังคิวนี้อยู่ ณ ขณะนี้ |

**สัญญาณเตือนที่ควรระวัง**: `messages_ready` สูงมากแต่ `consumers` เป็น `0`
หมายความว่า**ไม่มี Worker ตัวไหนฟังคิวนี้อยู่เลย** — งานทั้งหมดจะค้างตลอด
กาลจนกว่าจะมีคนสตาร์ท Worker ขึ้นมาใหม่ (สาเหตุคลาสสิกที่สุด: Worker crash
แล้วไม่มีระบบ auto-restart)

### 768.3 ตรวจสอบผ่าน RabbitMQ Management UI

เปิด `http://localhost:15672` (ทวนจากขั้นตอนที่ 763.3) แล้วไปที่แท็บ
**"Queues"** — จะเห็นกราฟ Queue Depth แบบ real-time ของทุกคิว พร้อม
Message Rate (in/out ต่อวินาที) ทำให้เห็นแนวโน้มได้ง่ายกว่าดูตัวเลขนิ่ง ๆ
จาก command line มาก — เหมาะสำหรับ debug ระหว่างพัฒนา แต่สำหรับ production
ที่ต้องการ dashboard รวมศูนย์และ alert อัตโนมัติ เราจะติดตั้ง **Flower**
(เครื่องมือ monitor เฉพาะของ Celery) ใน **Part 079**

### 768.4 ตรวจสอบ Queue Depth ผ่าน `redis-cli` (ถ้ายังใช้ Redis เป็น Broker)

```bash
# ทวนคำสั่งจาก Part 075 ขั้นตอนที่ 743.6 — llen นับจำนวนสมาชิกใน List
redis-cli -n 4 llen celery           # คิวเริ่มต้น
redis-cli -n 4 llen high_priority    # คิวสำหรับงานด่วน (ทวนจาก Part 076)
redis-cli -n 4 llen low_priority     # คิวสำหรับงานหนัก

# ดูรายการ key ทั้งหมดที่เกี่ยวกับ queue ใน DB index 4
redis-cli -n 4 keys '*'
```

### 768.5 เขียน Django Management Command ตรวจสุขภาพ Queue อัตโนมัติ

```python
# core/management/commands/check_queue_health.py
import logging

from django.conf import settings
from django.core.management.base import BaseCommand

from celery import current_app

logger = logging.getLogger(__name__)

# กำหนด threshold ที่ยอมรับได้ต่อคิว — ปรับตามลักษณะงานจริงของแต่ละคิว
QUEUE_DEPTH_THRESHOLDS = {
    'high_priority': 50,     # งานด่วนไม่ควรค้างเกิน 50 ชิ้น
    'low_priority': 500,     # งานหนักยอมให้ค้างได้มากกว่า เพราะไม่เร่งด่วน
    'dead_letter': 1,        # มี dead letter แม้แต่ 1 ชิ้นก็ควรแจ้งเตือนทันที
}


class Command(BaseCommand):
    help = 'ตรวจสอบความลึกของแต่ละ Queue เทียบกับ threshold ที่ตั้งไว้ แจ้งเตือนถ้าเกิน'

    def handle(self, *args, **options):
        with current_app.connection_or_acquire() as connection:
            for queue_name, threshold in QUEUE_DEPTH_THRESHOLDS.items():
                depth = self._get_queue_depth(connection, queue_name)
                status = '🔴 เกิน threshold!' if depth > threshold else '🟢 ปกติ'
                self.stdout.write(f'{queue_name}: {depth} message(s) [{status}]')

                if depth > threshold:
                    logger.error(
                        'Queue "%s" มี %d message ค้างอยู่ เกิน threshold (%d)',
                        queue_name, depth, threshold,
                    )
                    self._send_alert(queue_name, depth, threshold)

    def _get_queue_depth(self, connection, queue_name):
        """ใช้ kombu queue.queue_declare(passive=True) เพื่อถามจำนวน message
        โดยไม่สร้างคิวใหม่ถ้ายังไม่มี (ทำงานได้ทั้งกับ RabbitMQ และ Redis
        เพราะ kombu เป็นชั้นนามธรรมที่คุยกับทั้งสองแบบเหมือนกัน)"""
        from kombu import Queue
        queue = Queue(queue_name, channel=connection.channel())
        _, message_count, _ = queue.queue_declare(passive=True)
        return message_count

    def _send_alert(self, queue_name, depth, threshold):
        """จุดเชื่อมต่อระบบแจ้งเตือนจริง (Slack webhook, PagerDuty, อีเมลทีม)
        รายละเอียดการเชื่อมต่อระบบ monitoring แบบเต็มรูปแบบเรียนใน Part 079"""
        # ตัวอย่างแบบง่าย: log ระดับ critical ให้ระบบ log aggregation ดักจับ
        logger.critical(
            'ALERT: Queue "%s" ต้องการความสนใจด่วน — %d/%d message',
            queue_name, depth, threshold,
        )
```

```bash
# รันด้วยมือระหว่างพัฒนา
python manage.py check_queue_health

# ในโปรเจกต์จริง มักตั้งเป็น Celery Beat periodic task (ทวนจาก Part 076 ขั้นตอนที่ 751)
# ให้รันทุก 5 นาทีอัตโนมัติ แทนที่จะรันด้วยมือ
```

### 768.6 ตัวชี้วัดอื่นที่ควรดูควบคู่กับ Queue Depth

| ตัวชี้วัด | ความหมาย | สัญญาณอันตราย |
|---|---|---|
| **Message Rate (in/out)** | จำนวน message ที่เข้า/ออกคิวต่อวินาที | Rate เข้า > Rate ออก ต่อเนื่อง = คิวจะบวมขึ้นเรื่อย ๆ |
| **Consumer Count** | จำนวน Worker ที่ฟังคิวอยู่จริง | ลดลงกะทันหันโดยไม่ทราบสาเหตุ = worker ล่ม |
| **Oldest Message Age** | Message ที่ค้างนานที่สุดในคิวค้างมานานแค่ไหน | นานเกินกว่าที่ธุรกิจยอมรับได้ (เช่น อีเมล OTP ค้างเกิน 1 นาที) |
| **Worker Memory/CPU Usage** | resource ที่แต่ละ worker process ใช้ | สูงผิดปกติอาจบ่งบอก memory leak ใน task |
| **Task Success/Failure Rate** | สัดส่วน task ที่สำเร็จเทียบกับล้มเหลว | Failure rate พุ่งสูงกะทันหัน = มีปัญหาที่ dependency ภายนอก (ทวนจาก Part 075 ขั้นตอนที่ 749) |

### 768.7 ตารางสรุปขั้นตอนที่ 768

| หัวข้อ | สรุป |
|---|---|
| Queue Depth คืออะไร | จำนวน message ที่ค้างรอ Worker มาหยิบไปทำ |
| ทำไมสำคัญ | เพิ่มขึ้นเรื่อย ๆ = Worker ทำงานไม่ทันปริมาณงานที่เข้ามา |
| RabbitMQ | `rabbitmqctl list_queues` หรือ Management UI (`http://localhost:15672`) |
| Redis | `redis-cli -n 4 llen <queue_name>` |
| Automation | เขียน Django management command ตรวจ threshold อัตโนมัติ ผูกกับ Celery Beat |
| ขั้นต่อไป | Dashboard รวมศูนย์และ real-time alert เต็มรูปแบบด้วย **Flower** ใน Part 079 |

---

## ขั้นตอนที่ 769: กรอบการตัดสินใจเลือก Broker สำหรับ Production จริง

### 769.1 คำถาม 6 ข้อที่ต้องถามก่อนตัดสินใจ

หลังจากเห็นทั้ง Redis (Part 075-076) และ RabbitMQ (ขั้นตอนที่ 762-768) ใน
Part นี้แล้ว มาสรุปเป็นกรอบการตัดสินใจที่ใช้ได้จริงในหน้างาน:

1. **มี Redis ใช้งานอยู่แล้วในระบบหรือไม่?** (สำหรับ cache/session ตามที่
   Part 069 ทำไว้)
2. **ต้องการ routing ที่ซับซ้อนกว่าแค่ "แยกคิวตามความสำคัญ" หรือไม่?**
   (เช่น broadcast, pattern matching หลายเงื่อนไข)
3. **ต้องการ priority queue แบบละเอียดหลายระดับจริงจังหรือไม่?** (ไม่ใช่
   แค่ 2-3 ระดับที่แยกเป็นคนละคิวได้)
4. **ทีมมีความสามารถ/เวลาดูแล infrastructure เพิ่มอีก 1 service หรือไม่?**
5. **ปริมาณงาน (throughput) สูงแค่ไหน?** เป็นหลักสิบ, หลักพัน หรือหลักแสน
   message ต่อวินาที?
6. **ระบบต้องการการรับประกันความน่าเชื่อถือระดับสูงสุดหรือไม่?** (เช่น
   ธุรกรรมทางการเงิน ที่ message หายแม้แต่ชิ้นเดียวก็ยอมรับไม่ได้)

### 769.2 Decision Tree

```
เริ่มต้น
   │
   ▼
มี Redis อยู่แล้วในระบบไหม? ──ไม่มี──> ต้องการ routing/priority ซับซ้อนไหม? ──ใช่──> RabbitMQ
   │                                          │
  มี                                         ไม่ใช่
   │                                          │
   ▼                                          ▼
ต้องการ routing/priority              เลือกตามความถนัดของทีม
ซับซ้อน หรือ DLQ แบบ native ไหม?      (Redis เรียบง่ายกว่าเล็กน้อย)
   │
  ┌┴──────────────┐
 ใช่              ไม่ใช่
  │                │
  ▼                ▼
ทีมพร้อมดูแล      Redis
service เพิ่ม     (ต่อยอดจากที่มีอยู่แล้ว
ไหม?              ไม่ต้องติดตั้งใหม่)
  │
 ┌┴────────┐
ใช่        ไม่ใช่
 │          │
 ▼          ▼
RabbitMQ   Redis (ยอมรับข้อจำกัด
           เรื่อง routing/priority)
```

### 769.3 ตารางสรุปเชิงสถานการณ์ (ต่อยอดจากขั้นตอนที่ 762.5)

| สถานการณ์จริง | คำแนะนำ | เหตุผล |
|---|---|---|
| Startup ขนาดเล็ก เพิ่งเริ่มใช้ Celery | Redis | ลด operational overhead ให้เหลือน้อยที่สุด โฟกัสที่ product |
| SaaS ที่มี tier ราคาต่างกัน ต้อง priority งานตาม tier ลูกค้าอย่างละเอียด | RabbitMQ | Priority queue แบบ native (`x-max-priority`) รองรับได้ตรงจุด |
| ระบบ e-commerce ที่ต้อง broadcast event เดียวไปหลาย service (inventory, notification, analytics) | RabbitMQ | Fanout Exchange ทำได้ในคำสั่งเดียว (ทวนจากขั้นตอนที่ 764.2) |
| ทีม DevOps เล็ก ไม่มีคนเชี่ยวชาญ AMQP | Redis | Learning curve ต่ำกว่า ใช้ความรู้ Redis ที่มีอยู่แล้วจาก Part 069 |
| ระบบ Fintech ที่ message หายแม้ชิ้นเดียวก็เป็นปัญหาใหญ่ | RabbitMQ | Durability + Acknowledgment protocol ที่แข็งแรงกว่า (ทวนจากขั้นตอนที่ 762.2, 766.5) |
| ระบบที่ throughput สูงมาก (หลักแสน message/วินาที) เน้นความเร็วดิบเป็นหลัก | ต้องวัดจริงทั้งคู่ (benchmark) | ทั้งสองตัวปรับจูนได้ถึงระดับสูงมาก ขึ้นกับ hardware/config มากกว่าตัวซอฟต์แวร์เอง |

### 769.4 แนวทางไฮบริด: ใช้ทั้งสองตัวพร้อมกันในระบบเดียว

ในระบบขนาดใหญ่ที่ซับซ้อน การใช้ **ทั้ง RabbitMQ และ Redis พร้อมกัน** ในคน
ละบทบาทเป็นเรื่องปกติมาก:

```
┌─────────────────────────────────────────────────────────────┐
│                        Django Project                        │
├───────────────────┬───────────────────┬─────────────────────┤
│  Celery Broker     │  Result Backend   │  Cache/Session/     │
│  (RabbitMQ)        │  (Redis DB 5)     │  Throttle (Redis)   │
│  งาน routing ซับซ้อน│  เก็บสถานะ task    │  ทวนจาก Part 069     │
│  + DLQ native      │  query ได้เร็ว     │                      │
└───────────────────┴───────────────────┴─────────────────────┘
```

นี่คือรูปแบบที่โปรเจกต์ Blog ของหลักสูตรนี้กำลังจะกลายเป็นในขั้นตอนที่ 770
— **RabbitMQ ทำหน้าที่ broker ที่ต้องการความน่าเชื่อถือและ routing สูง
ในขณะที่ Redis ยังทำหน้าที่เดิมทั้งหมดที่มันถนัด** (cache, session, throttle,
result backend) — ไม่มีกฎตายตัวว่าต้องเลือกอย่างใดอย่างหนึ่งเท่านั้น

### 769.5 คำแนะนำสำหรับ Blog Project ของหลักสูตรนี้

ในบริบทของโปรเจกต์ Blog ที่คุณพัฒนาต่อเนื่องมาตั้งแต่ Part 001 มีเหตุผล
สนับสนุนการย้ายไป RabbitMQ ที่ชัดเจนพอสมควร:

- มี Task หลายความสำคัญ (`high_priority`/`low_priority` จาก Part 076) —
  ได้ประโยชน์จาก Direct Exchange + Binding ของ AMQP เต็มที่
- ต้องการ Dead Letter Queue ที่แข็งแรง (ขั้นตอนที่ 765) — RabbitMQ ให้ native
  DLQ เสริมชั้น application-level ได้
- โปรเจกต์กำลังเติบโตเข้าสู่ Phase Production (Phase 10 เป็นต้นไปว่าด้วย
  Security, Phase 11 ว่าด้วย Deployment) — เป็นจังหวะที่เหมาะสมที่จะลงทุน
  เรียนรู้เครื่องมือระดับ enterprise ก่อนเข้าสู่ช่วง production จริง

ขั้นตอนที่ 770 จะพาคุณลงมือทำการย้ายนี้แบบครบวงจร

---

## ขั้นตอนที่ 770: สรุปและแบบฝึกหัด — ย้าย Broker ของ Blog Project จาก Redis ไปเป็น RabbitMQ

### 770.1 แผนการย้าย Broker ทีละขั้น (Migration Plan)

```
ขั้นที่ 1: ติดตั้ง RabbitMQ (ขั้นตอนที่ 763)
ขั้นที่ 2: สร้าง vhost/user/permission เฉพาะโปรเจกต์
ขั้นที่ 3: เปลี่ยน CELERY_BROKER_URL เป็น amqp://
ขั้นที่ 4: ย้าย Queue/Exchange/Routing เดิมมาใช้ kombu Queue/Exchange แบบ explicit
ขั้นที่ 5: เพิ่ม Dead Letter Queue (native + application-level)
ขั้นที่ 6: ตั้งค่า Reliability (task_acks_late, task_reject_on_worker_lost)
ขั้นที่ 7: ทดสอบทุก Task เดิมว่ายังทำงานถูกต้อง — business logic ไม่ต้องแก้เลย
ขั้นที่ 8: Monitor queue depth เพื่อยืนยันว่าระบบเสถียร
```

**ประเด็นสำคัญที่สุดของ Migration นี้**: ทวนจาก Part 075-076 — Task ทุกตัว
เขียนด้วย `@shared_task` (ทวนขั้นตอนที่ 743.1) ที่ไม่ผูกกับ broker ตัวไหน
เจาะจงเลย ดังนั้น**การเปลี่ยน broker ครั้งนี้จะไม่ต้องแก้โค้ด business logic
ของ task แม้แต่บรรทัดเดียว** — แก้แค่ configuration ในไฟล์ `settings.py`
เท่านั้น นี่คือประโยชน์ของการออกแบบระบบให้ Producer/Consumer ไม่รู้จัก
รายละเอียดของ Broker ตั้งแต่แรก (หลักการ Decoupling จากขั้นตอนที่ 761.3)

### 770.2 `docker-compose.yml` ฉบับสมบูรณ์ (Redis + RabbitMQ อยู่ด้วยกัน)

```yaml
# docker-compose.yml
services:
  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    volumes:
      - redis_data:/data

  rabbitmq:
    image: rabbitmq:3.13-management
    ports:
      - "5672:5672"
      - "15672:15672"
    environment:
      RABBITMQ_DEFAULT_VHOST: blog_vhost
      RABBITMQ_DEFAULT_USER: django_blog
      RABBITMQ_DEFAULT_PASS: change-this-strong-password
    volumes:
      - rabbitmq_data:/var/lib/rabbitmq

  worker_urgent:
    build: .
    command: celery -A config worker -Q high_priority --concurrency=8 --loglevel=info
    depends_on:
      - rabbitmq
      - redis
    env_file: .env

  worker_batch:
    build: .
    command: celery -A config worker -Q low_priority,dead_letter --concurrency=2 --loglevel=info
    depends_on:
      - rabbitmq
      - redis
    env_file: .env

  beat:
    build: .
    command: celery -A config beat --loglevel=info
    depends_on:
      - rabbitmq
      - redis
    env_file: .env

volumes:
  redis_data:
  rabbitmq_data:
```

### 770.3 `config/settings.py` ฉบับสมบูรณ์ (รวมทุกขั้นตอนของ Part นี้)

```python
# config/settings.py
import os

from kombu import Exchange, Queue

# ── Redis (ทวนจาก Part 069/075 — คงบทบาทเดิมทั้งหมด) ────────────────
REDIS_HOST = os.environ.get('REDIS_HOST', '127.0.0.1')
REDIS_PORT = os.environ.get('REDIS_PORT', '6379')

# ── RabbitMQ (ใหม่ใน Part นี้) ───────────────────────────────────────
RABBITMQ_HOST = os.environ.get('RABBITMQ_HOST', '127.0.0.1')
RABBITMQ_PORT = os.environ.get('RABBITMQ_PORT', '5672')
RABBITMQ_USER = os.environ.get('RABBITMQ_USER', 'django_blog')
RABBITMQ_PASSWORD = os.environ.get('RABBITMQ_PASSWORD', 'change-this-strong-password')
RABBITMQ_VHOST = os.environ.get('RABBITMQ_VHOST', 'blog_vhost')

# ── Celery: Broker เปลี่ยนเป็น RabbitMQ, Result Backend ยังเป็น Redis ──
CELERY_BROKER_URL = (
    f'amqp://{RABBITMQ_USER}:{RABBITMQ_PASSWORD}'
    f'@{RABBITMQ_HOST}:{RABBITMQ_PORT}/{RABBITMQ_VHOST}'
)
CELERY_RESULT_BACKEND = f'redis://{REDIS_HOST}:{REDIS_PORT}/5'

CELERY_ACCEPT_CONTENT = ['json']
CELERY_TASK_SERIALIZER = 'json'
CELERY_RESULT_SERIALIZER = 'json'
CELERY_TIMEZONE = TIME_ZONE
CELERY_ENABLE_UTC = True
CELERY_TASK_TRACK_STARTED = True
CELERY_RESULT_EXPIRES = 3600
CELERY_TASK_SOFT_TIME_LIMIT = 300
CELERY_TASK_TIME_LIMIT = 360

# ── Reliability: at-least-once delivery (ทวนจากขั้นตอนที่ 766) ─────────
CELERY_TASK_ACKS_LATE = True
CELERY_TASK_REJECT_ON_WORKER_LOST = True
CELERY_WORKER_PREFETCH_MULTIPLIER = 1

# ── Exchange/Queue/Routing/DLQ (ทวนจากขั้นตอนที่ 764-765) ──────────────
dlx_exchange = Exchange('dlx', type='direct')

CELERY_TASK_QUEUES = (
    Queue(
        'high_priority', Exchange('high_priority'), routing_key='high_priority',
        queue_arguments={
            'x-dead-letter-exchange': 'dlx',
            'x-dead-letter-routing-key': 'expired_high_priority',
        },
    ),
    Queue(
        'low_priority', Exchange('low_priority'), routing_key='low_priority',
        queue_arguments={
            'x-message-ttl': 3600000,
            'x-dead-letter-exchange': 'dlx',
            'x-dead-letter-routing-key': 'expired_low_priority',
        },
    ),
    Queue('celery', Exchange('celery'), routing_key='celery'),
    Queue('dead_letter', Exchange('dead_letter'), routing_key='dead_letter'),
    Queue('expired_high_priority', dlx_exchange, routing_key='expired_high_priority'),
    Queue('expired_low_priority', dlx_exchange, routing_key='expired_low_priority'),
)

CELERY_TASK_ROUTES = {
    'accounts.tasks.send_password_reset_email': {'queue': 'high_priority'},
    'accounts.tasks.send_login_otp': {'queue': 'high_priority'},
    'blog.tasks.gather_report_data': {'queue': 'low_priority'},
    'blog.tasks.render_report_pdf': {'queue': 'low_priority'},
    'core.tasks.record_dead_letter_task': {'queue': 'dead_letter'},
}

CELERY_TASK_DEFAULT_QUEUE = 'celery'
CELERY_TASK_DEFAULT_EXCHANGE = 'celery'
CELERY_TASK_DEFAULT_ROUTING_KEY = 'celery'
```

### 770.4 คำสั่งรันระบบทั้งหมดสำหรับพัฒนา (ฉบับสมบูรณ์)

```bash
# Terminal 1: Django development server
source venv/bin/activate
python manage.py runserver

# Terminal 2: Worker สำหรับงานด่วน
source venv/bin/activate
celery -A config worker -Q high_priority --concurrency=8 --loglevel=info --hostname=urgent@%h

# Terminal 3: Worker สำหรับงานหนัก + dead letter
source venv/bin/activate
celery -A config worker -Q low_priority,dead_letter --concurrency=2 --loglevel=info --hostname=batch@%h

# Terminal 4: Celery Beat (ทวนจาก Part 076 ขั้นตอนที่ 751)
source venv/bin/activate
celery -A config beat --loglevel=info

# Terminal 5 (ตรวจสอบ): Queue depth แบบ real-time
watch -n 2 'sudo rabbitmqctl list_queues -p blog_vhost name messages consumers'
```

### 770.5 ทดสอบว่าระบบทำงานเหมือนเดิมทุกประการหลังย้าย Broker

```bash
python manage.py shell
```

```python
>>> from accounts.tasks import send_password_reset_email_task
>>> result = send_password_reset_email_task.delay(1, 'https', 'localhost:8000')
>>> result.id
'a1b2c3d4-...'   # ทำงานเหมือนเดิมทุกประการ แม้ broker จะเปลี่ยนไปแล้ว

>>> result.status
'PENDING'
# รอสักครู่ให้ worker_urgent (ฟังคิว high_priority) หยิบไปทำ
>>> result.status
'SUCCESS'
```

```bash
# ยืนยันว่า message ไหลผ่าน exchange/queue ที่ตั้งไว้จริง
sudo rabbitmqctl list_queues -p blog_vhost name messages
# high_priority ควรเห็นค่า messages เพิ่มขึ้นชั่วครู่แล้วกลับเป็น 0
# หลัง worker_urgent หยิบไปทำเสร็จ
```

### 770.6 สรุปสิ่งที่ได้เรียนรู้ใน Part นี้

- ✅ เข้าใจแนวคิด Message Queue ในภาพกว้าง (Producer-Consumer,
  Decoupling, Buffering) ที่อยู่เบื้องหลัง Celery มาตลอด
- ✅ เปรียบเทียบ RabbitMQ กับ Redis อย่างละเอียดทั้ง 3 มิติ: reliability,
  feature (routing, priority, DLQ), และความซับซ้อนในการ deploy
- ✅ ติดตั้งและตั้งค่า RabbitMQ พร้อม vhost/user/permission เฉพาะโปรเจกต์
  อย่างปลอดภัย
- ✅ เข้าใจ AMQP protocol ผ่าน 3 องค์ประกอบหลัก: Exchange, Queue, Binding
  และประเภท Exchange ทั้ง 4 แบบ
- ✅ สร้าง Dead Letter Queue ทั้งระดับ native (RabbitMQ) และระดับ
  application (`on_failure` hook + `DeadLetterRecord` model)
- ✅ เข้าใจ Message Acknowledgment อย่างลึกซึ้ง: `task_acks_late`,
  at-most-once vs at-least-once vs exactly-once, และทำไม idempotency
  ยังจำเป็นเสมอไม่ว่าจะเลือก broker ไหน
- ✅ ใช้ Competing Consumers Pattern สเกลด้วยหลาย Worker แบบ queue-based
  load balancing ได้โดยไม่ต้องแก้โค้ด Django
- ✅ Monitor queue depth ผ่าน `rabbitmqctl`, Management UI, และเขียน
  Django management command ตรวจสุขภาพอัตโนมัติ
- ✅ มีกรอบการตัดสินใจที่เป็นระบบสำหรับเลือก broker ในโปรเจกต์ production
  จริงในอนาคต
- ✅ ย้าย broker ของ Blog project จาก Redis เป็น RabbitMQ สำเร็จ โดยไม่แก้
  business logic ของ task แม้แต่บรรทัดเดียว

### 770.7 Checklist ก่อนไป Part ถัดไป

- [ ] ติดตั้ง RabbitMQ สำเร็จ (ผ่าน apt/brew/Docker) และเข้า Management UI
      ที่ `http://localhost:15672` ได้
- [ ] สร้าง vhost, user, permission เฉพาะโปรเจกต์แล้ว (ไม่ใช้ `guest/guest`)
- [ ] เปลี่ยน `CELERY_BROKER_URL` เป็น `amqp://` สำเร็จ โดย
      `CELERY_RESULT_BACKEND` ยังคงเป็น Redis
- [ ] ประกาศ `CELERY_TASK_QUEUES` ด้วย `kombu.Exchange`/`Queue` ครบทุกคิว
      ที่ใช้ในโปรเจกต์
- [ ] ตั้งค่า Dead Letter Exchange/Queue ทั้งระดับ native และสร้าง
      `DeadLetterTask` + `DeadLetterRecord` model สำเร็จ
- [ ] ตั้งค่า `CELERY_TASK_ACKS_LATE = True` และ
      `CELERY_TASK_REJECT_ON_WORKER_LOST = True`
- [ ] รัน Worker หลายตัวชี้คิวเดียวกัน แล้วยืนยันว่า Broker แจกจ่ายงานแบบ
      round-robin จริง
- [ ] ตรวจสอบ queue depth ผ่าน `rabbitmqctl list_queues` ได้สำเร็จ
- [ ] ทดสอบ task เดิมทั้งหมดจาก Part 075-076 ว่ายังทำงานถูกต้องหลังย้าย
      broker

### 770.8 แบบฝึกหัดท้ายบท

**แบบฝึกหัดที่ 1**: ติดตั้ง RabbitMQ ด้วย Docker ตามขั้นตอนที่ 763.3 แล้ว
สร้าง vhost, user และ permission เฉพาะโปรเจกต์ของคุณเอง (ห้ามใช้
`guest/guest`) จากนั้นเชื่อมต่อจาก Django shell ตามขั้นตอนที่ 763.8 และ
ถ่ายภาพหน้าจอ Management UI ที่แสดงว่า connection สำเร็จ

**แบบฝึกหัดที่ 2**: สร้าง Dead Letter Queue แบบ application-level ตาม
ขั้นตอนที่ 765.5 ให้ครบทั้ง `DeadLetterTask`, `record_dead_letter_task`,
และ `DeadLetterRecord` model จากนั้นจงใจทำให้ task หนึ่งล้มเหลวถาวร (เช่น
ตั้ง `EMAIL_HOST` ผิดแล้วปล่อยให้ retry ครบทุกครั้งจนกลายเป็น `FAILURE`)
แล้วตรวจสอบว่า record ถูกบันทึกลง Django Admin จริง และลองกด action
"Reprocess" ดูว่างานถูกส่งกลับเข้าคิวสำเร็จหรือไม่

**แบบฝึกหัดที่ 3**: เปรียบเทียบพฤติกรรมจริงระหว่าง `task_acks_late=True`
กับ `task_acks_late=False` — เขียน task ที่ใช้เวลาทำงาน 30 วินาที (ใช้
`time.sleep(30)` จำลอง) แล้วสั่ง `.delay()` จากนั้น **ระหว่างที่ task กำลัง
ทำงานอยู่** ให้สั่ง `kill -9` ที่ process ของ Worker โดยตรง สังเกตว่า
message กลับเข้าคิวให้ worker ตัวอื่นทำใหม่หรือไม่ในแต่ละกรณี บันทึกผลลัพธ์
เปรียบเทียบทั้งสองค่าไว้ในไฟล์ `notes.md`

**แบบฝึกหัดที่ 4 (ขั้นสูง)**: เขียน Django management command
`check_queue_health` ตามขั้นตอนที่ 768.5 ให้ครบ แล้วตั้งเป็น Celery Beat
periodic task (ทวนจาก Part 076 ขั้นตอนที่ 751) ให้รันทุก 5 นาที จากนั้น
จำลองสถานการณ์ที่ queue depth เกิน threshold (เช่น หยุด Worker ของ
`low_priority` แล้วยิงงานเข้าคิวจำนวนมาก) และยืนยันว่าเห็น log ระดับ
`critical` ปรากฏขึ้นจริงตามที่คาดหวัง

### 770.9 คำถามที่พบบ่อย (FAQ)

**Q: ต้องย้ายทุกโปรเจกต์ไปใช้ RabbitMQ หลังเรียน Part นี้เลยไหม?**
A: ไม่จำเป็นเลย — Redis ยังคงเป็นตัวเลือกที่ดีมากสำหรับโปรเจกต์ส่วนใหญ่
โดยเฉพาะที่มี Redis อยู่แล้ว (ทวนกรอบการตัดสินใจจากขั้นตอนที่ 769) Part นี้
มีเป้าหมายให้คุณ**เข้าใจทั้งสองตัวอย่างลึกซึ้งพอที่จะเลือกได้ถูกต้อง** ไม่ใช่
บังคับให้เปลี่ยนไปใช้ RabbitMQ เสมอ

**Q: ถ้าเปลี่ยน broker แล้ว Task ที่เคยเขียนด้วย `@shared_task` ต้องแก้อะไร
ไหม?**
A: ไม่ต้องแก้เลยแม้แต่บรรทัดเดียว (ทวนจากขั้นตอนที่ 770.1) — นี่คือจุดแข็ง
ของสถาปัตยกรรม Celery ที่แยกชั้น business logic ของ task ออกจากรายละเอียด
ของ broker อย่างสมบูรณ์ผ่าน `@shared_task` ตั้งแต่ Part 075 ขั้นตอนที่
743.1 แล้ว

**Q: RabbitMQ ใช้ RAM มากกว่า Redis ไหมในสถานการณ์ทั่วไป?**
A: ขึ้นกับปริมาณงานและการตั้งค่า — RabbitMQ เก็บ message ที่ยังไม่ ack ไว้
ใน RAM เช่นกัน (เว้นแต่ตั้งเป็น lazy queue ที่เขียนลง disk แทน) โดยทั่วไป
ถ้าปริมาณงานใกล้เคียงกัน การใช้ RAM ของทั้งสองไม่ต่างกันมากนัก ความแตกต่าง
ที่ชัดเจนกว่าคือ **RabbitMQ ใช้ CPU/overhead มากกว่าเล็กน้อย** จากการรักษา
guarantee ต่าง ๆ ตาม AMQP protocol (durability, acknowledgment tracking)
ซึ่งเป็นการแลกเปลี่ยนที่คุ้มค่าถ้าต้องการ reliability ระดับสูงกว่า

**Q: ทำไมยังต้องใช้ Redis เป็น Result Backend ทั้งที่เปลี่ยนมาใช้ RabbitMQ
เป็น broker แล้ว ทำไมไม่ใช้ RabbitMQ ให้หมดไปเลย?**
A: ทวนเหตุผลจากขั้นตอนที่ 763.7 — Broker กับ Result Backend มีธรรมชาติของ
งานต่างกันโดยสิ้นเชิง (message ที่ใช้ครั้งเดียวแล้วหาย เทียบกับค่าที่ต้อง
query ซ้ำได้) Redis เหมาะกับงานเก็บผลลัพธ์มากกว่าโดยธรรมชาติ การผสมทั้งสอง
เทคโนโลยีเข้าด้วยกันตามจุดแข็งของแต่ละตัว (แนวทางไฮบริดจากขั้นตอนที่ 769.4)
คือรูปแบบที่ระบบ production จริงจำนวนมากใช้กัน ไม่ใช่ข้อบกพร่องของการ
ออกแบบแต่อย่างใด

---

## เตรียมตัวสำหรับ Part ถัดไป

**Part 078: Real-time Notification System** จะพาคุณผสมผสานทุกอย่างที่เรียน
มาจาก Phase 9 เข้าด้วยกัน — Django Channels (Part 074), Celery
Chain/Group/Chord (Part 076), และ Message Queue ที่แข็งแรงขึ้นจาก Part นี้
— เพื่อสร้างระบบแจ้งเตือนแบบ real-time เต็มรูปแบบ: เมื่อ Celery Task ทำงาน
เสร็จในเบื้องหลัง (เช่น export รายงานเสร็จ, มีคอมเมนต์ใหม่) ระบบจะส่ง
notification ผ่าน WebSocket ไปแสดงที่หน้าเว็บของผู้ใช้ทันทีโดยไม่ต้อง
refresh หรือ poll เองเลย ซึ่งเป็นแพทเทิร์นที่ Part 076 ขั้นตอนที่ 759 (FAQ)
เกริ่นไว้แล้วว่าจะเจาะลึกเต็มรูปแบบ

เตรียม RabbitMQ, Redis, และ Celery Worker/Beat ที่ทำงานได้ครบจาก Part นี้
ไว้ให้พร้อม เพราะ Part ถัดไปจะต่อยอดจากโครงสร้างเดิมทั้งหมดทันที!
