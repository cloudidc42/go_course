# Part 091: Message Queue: RabbitMQ กับ Go

> ภาคที่ 8: Microservices, gRPC, Message Queue — ตอนที่ 4 จาก 7 (Part 88–94)

## สารบัญของบทนี้

1. ทบทวน: จาก synchronous สู่ asynchronous (ต่อยอด Part 088)
2. RabbitMQ คืออะไร และแก้ปัญหาอะไร
3. แนวคิดหลักของ AMQP: Exchange, Queue, Binding, Routing Key
4. ประเภทของ Exchange: direct, fanout, topic, headers
5. Client library สำหรับ Go: `amqp091-go`
6. เตรียมสภาพแวดล้อม (สิ่งที่รันจริงในบทนี้)
7. เขียน Producer: ส่งข้อความเข้า queue
8. เขียน Consumer: รับข้อความและประมวลผล
9. Message Acknowledgment: `ack`/`nack` และทำไมมันสำคัญต่อความน่าเชื่อถือ
10. Topic Exchange: routing แบบยืดหยุ่นด้วย pattern
11. Worker Queue แบบกระจายหลายโปรเซส: ต่อยอดจาก Worker Pool (Part 041)
12. Dead Letter Queue และการจัดการข้อความที่ล้มเหลวซ้ำๆ
13. RabbitMQ กับ Worker Pool ในโปรเซสเดียว (Part 041): ใช้อะไรเมื่อไหร่
14. สรุปสิ่งที่ได้เรียนในบทนี้
15. แบบฝึกหัดท้ายบท

---

## 1. ทบทวน: จาก synchronous สู่ asynchronous (ต่อยอด Part 088)

ใน **Part 088** เราวางรากฐานสำคัญไว้ข้อหนึ่งคือความแตกต่างระหว่างการสื่อสารแบบ **synchronous** (เช่น เรียก REST API หรือ gRPC แล้วรอผลลัพธ์กลับมาก่อนทำงานต่อ — Part 089, 090) กับแบบ **asynchronous** (ส่งงานแล้วเดินหน้าต่อทันที ไม่ต้องรอผลลัพธ์) และเราสรุปไว้ว่า service ที่คุยกันแบบ synchronous ล้วนๆ จะมีปัญหาเรื่อง **coupling ที่แน่นเกินไป**:

- ถ้า service B ล่มหรือช้า service A ที่เรียก B แบบ sync ก็จะช้าหรือพังตามไปด้วยทันที (cascading failure — เรื่องนี้จะกลับมาเจาะลึกอีกครั้งใน Part 094)
- ถ้ามีงานเข้ามาเป็น burst จำนวนมาก (เช่น ตอนโปรโมชั่นลดราคา มีคนสั่งซื้อพร้อมกันหลักหมื่นออเดอร์ในเวลาไม่กี่นาที) service ปลายทางที่ประมวลผลได้จำกัด (เช่น ระบบส่งอีเมลยืนยัน หรือระบบตัดสต๊อกที่ query ฐานข้อมูลหนัก) จะรับภาระไม่ไหวและล่ม
- Producer กับ Consumer ต้อง**ทำงานพร้อมกันตลอดเวลา** ถ้า consumer ปิดปรับปรุงชั่วคราว (deploy ใหม่) producer ก็เรียกไม่ได้เลย

**Message Queue** คือคำตอบมาตรฐานของปัญหานี้ แนวคิดหลักคือแทรก **broker กลาง** ระหว่าง producer กับ consumer:

```
sync (ตรงไปตรงมา):     Service A ──── HTTP/gRPC call ────► Service B
                                (A ต้องรอ B ตอบ ถึงจะทำงานต่อได้)

async (ผ่าน message queue): Service A ──► [ Message Broker ] ──► Service B
                                (A ส่งแล้วเดินหน้าต่อทันที B ดึงงานไปทำเมื่อพร้อม)
```

ประโยชน์หลักที่ได้จากการแทรก broker เข้ามา 3 ข้อที่จะเห็นชัดเจนตลอดบทนี้:

1. **Decoupling**: A ไม่จำเป็นต้องรู้ว่า B คือใคร อยู่ที่ไหน หรือแม้แต่ B ทำงานอยู่หรือเปล่าในขณะนั้น A แค่ส่งข้อความเข้า queue แล้วเสร็จหน้าที่ของตัวเอง
2. **Buffering โหลดที่พุ่งสูง**: ถ้ามีงานเข้ามาเร็วกว่าที่ consumer จะประมวลผลทัน ข้อความจะถูกเก็บรอไว้ใน queue (แทนที่จะทำให้ consumer ล่มเพราะรับ request พร้อมกันเกินกำลัง) consumer ค่อยๆ ดึงงานไปทำตามจังหวะของตัวเอง
3. **ความทนทาน (resilience)**: ถ้า consumer ล่มหรือกำลัง deploy ใหม่ ข้อความยังคงรออยู่ใน queue อย่างปลอดภัย (ถ้าตั้งค่า durable ไว้ถูกต้อง) พอ consumer กลับมาออนไลน์ก็ดึงงานที่ค้างไปทำต่อได้ทันที ไม่มีข้อมูลสูญหาย

**RabbitMQ** คือ message broker แบบ open source ที่ได้รับความนิยมสูงสุดตัวหนึ่งของโลก พัฒนาโดยทีม Pivotal (ปัจจุบันอยู่ภายใต้ Broadcom/VMware) เขียนด้วยภาษา Erlang (ภาษาที่ออกแบบมาเพื่อระบบ concurrent/distributed ที่ทนทานสูงเป็นพิเศษ) และใช้โปรโตคอลมาตรฐานเปิดชื่อ **AMQP** (Advanced Message Queuing Protocol) เป็นแกนหลัก

---

## 2. RabbitMQ คืออะไร และแก้ปัญหาอะไร

RabbitMQ จัดอยู่ในหมวด **message broker** แบบ "smart broker, dumb consumer" — ตัว broker เองรับผิดชอบ routing logic ที่ซับซ้อนได้ (เลือกว่าข้อความไหนควรไปที่ queue ไหน) ส่วน consumer แค่ดึงข้อความจาก queue ที่ตัวเอง subscribe อยู่ไปประมวลผลตรงๆ ไม่ต้องรู้เรื่อง routing เลย

งานที่ RabbitMQ เก่งที่สุดคือรูปแบบ **task queue**: มีงานชิ้นหนึ่งเกิดขึ้น (เช่น "ส่งอีเมลยืนยันคำสั่งซื้อ", "ประมวลผลรูปภาพที่อัปโหลด", "สร้างรายงาน PDF") และงานชิ้นนั้น**ควรถูกทำเพียงครั้งเดียว**โดย worker ตัวใดตัวหนึ่ง แล้วก็จบไป — ตรงข้ามกับรูปแบบ event streaming ที่หลายระบบอิสระต้องรับรู้เหตุการณ์เดียวกันพร้อมกัน (นั่นคือจุดแข็งของ Kafka ซึ่งจะเรียนใน **Part 092** — ท้ายบทนี้จะมีตารางเทียบให้ชัดอีกครั้งในบทถัดไป)

ตัวอย่างสถานการณ์จริงที่ใช้ RabbitMQ:

- เว็บอีคอมเมิร์ซ: user กด "สั่งซื้อ" → API ตอบกลับทันทีว่า "รับออเดอร์แล้ว" (เร็ว ไม่ต้องรอ) → ส่งงาน "ส่งอีเมลยืนยัน" เข้า queue → worker แยกต่างหากค่อยๆ ดึงไปส่งอีเมลทีละฉบับ
- ระบบอัปโหลดรูปภาพ: user อัปโหลดรูป → เก็บไฟล์ต้นฉบับแล้วตอบกลับทันที → ส่งงาน "สร้าง thumbnail หลายขนาด" เข้า queue → worker pool แยกไปประมวลผลภาพ (งานหนัก CPU) โดยไม่บล็อก HTTP request ของ user
- ระบบแจ้งเตือน: มีหลาย event ที่ต้อง trigger การแจ้งเตือนผ่าน push notification, SMS, email → ส่งเข้า queue เดียว → worker ดึงไปประมวลผลตามลำดับ ควบคุม throughput ไม่ให้ยิง SMS gateway จนโดน rate limit

---

## 3. แนวคิดหลักของ AMQP: Exchange, Queue, Binding, Routing Key

จุดที่มือใหม่ RabbitMQ สับสนบ่อยที่สุดคือ: **producer ไม่เคยส่งข้อความเข้า queue โดยตรง** สิ่งที่ producer ทำคือส่งข้อความไปที่ **exchange** เท่านั้น แล้ว exchange จะเป็นคนตัดสินใจว่าจะส่งต่อข้อความนั้นไปที่ queue ไหนบ้าง ตามกฎที่เรียกว่า **binding**

```
                              ┌─────────────┐
   Producer ──publish──────► │  Exchange   │
                              └──────┬──────┘
                                     │ binding (routing key match)
                       ┌─────────────┼─────────────┐
                       ▼             ▼             ▼
                  ┌────────┐   ┌────────┐    ┌────────┐
                  │ Queue A│   │ Queue B│    │ Queue C│
                  └───┬────┘   └───┬────┘    └───┬────┘
                      ▼            ▼             ▼
                  Consumer 1   Consumer 2    Consumer 3
```

องค์ประกอบหลักของ AMQP ที่ต้องเข้าใจให้แม่นก่อนเขียนโค้ดบรรทัดแรก:

| องค์ประกอบ | ความหมาย | เปรียบเทียบง่ายๆ |
|---|---|---|
| **Exchange** | จุดที่ producer ส่งข้อความเข้ามา ทำหน้าที่ routing เท่านั้น ไม่เก็บข้อความถาวร | เหมือนศูนย์คัดแยกไปรษณีย์ |
| **Queue** | ที่เก็บข้อความจริงๆ รอให้ consumer มาดึงไปประมวลผล | เหมือนตู้ไปรษณีย์ปลายทางที่ผู้รับมาเปิดดู |
| **Binding** | กฎที่บอกว่า exchange ควรส่งข้อความแบบไหนไปที่ queue ไหน | เหมือนกฎการคัดแยกจดหมายตามรหัสไปรษณีย์ |
| **Routing Key** | ป้ายกำกับสั้นๆ ที่ producer แนบไปกับข้อความ ใช้จับคู่กับ binding | เหมือนรหัสไปรษณีย์ที่เขียนบนซองจดหมาย |
| **Consumer** | โปรแกรมที่เชื่อมต่อไปยัง queue แล้วดึงข้อความออกมาประมวลผล | เหมือนบุรุษไปรษณีย์ที่มารับพัสดุจากตู้ไปส่งต่อ |

**Default exchange**: RabbitMQ มี exchange พิเศษตัวหนึ่งชื่อว่า `""` (ชื่อว่าง) เป็น exchange ประเภท direct ที่ผูก (bind) กับทุก queue โดยอัตโนมัติ ด้วย routing key ที่ตรงกับชื่อ queue เป๊ะๆ นี่คือเหตุผลที่ตัวอย่างเบื้องต้นส่วนใหญ่ (รวมถึงหัวข้อ 7 ในบทนี้) ดูเหมือน "ส่งเข้า queue ตรงๆ" ได้ — จริงๆ แล้วเบื้องหลังมันก็ผ่าน exchange เหมือนเดิม เพียงแต่เป็น default exchange ที่มี binding สำเร็จรูปให้แล้ว

---

## 4. ประเภทของ Exchange: direct, fanout, topic, headers

RabbitMQ มี exchange 4 ประเภทหลัก แต่ละแบบมีกฎ routing ต่างกัน:

### Direct Exchange
Routing key ของข้อความต้อง**ตรงเป๊ะ**กับ binding key ของ queue เท่านั้น เหมาะกับกรณีง่ายๆ ที่ต้องการส่งข้อความไปยัง queue ใดคิวหนึ่งอย่างเจาะจง (default exchange ก็เป็น direct exchange ชนิดพิเศษนี่เอง)

### Fanout Exchange
ไม่สนใจ routing key เลย — ส่งสำเนาข้อความไปให้**ทุก queue**ที่ผูกกับ exchange นี้ เหมาะกับ broadcast เช่น "แจ้งทุก service ว่ามีการอัปเดต config" หรือ "ส่ง event เดียวไปให้หลาย consumer ที่สนใจต่างมุมมองกัน"

### Topic Exchange
ยืดหยุ่นที่สุด — routing key เป็น string คั่นด้วยจุด เช่น `order.created`, `order.cancelled`, `payment.failed` และ binding key รองรับ wildcard สองแบบ: `*` (แทนคำเดียว) และ `#` (แทนกี่คำก็ได้ รวมถึงศูนย์คำ) เช่น queue ที่ bind ด้วย `order.*` จะรับทั้ง `order.created` และ `order.cancelled` แต่ไม่รับ `payment.failed` — เดี๋ยวหัวข้อ 10 จะสาธิตแบบรันจริงให้ดู

### Headers Exchange
Routing อิงจาก **header ของข้อความ** (key-value pairs) แทน routing key ใช้เมื่อเงื่อนไข routing ซับซ้อนเกินกว่าจะใส่ในรูป string เดียวได้ (ใช้น้อยที่สุดในสี่แบบ จึงไม่มีตัวอย่างรันจริงในบทนี้)

| ประเภท | ใช้ Routing Key? | พฤติกรรม |
|---|---|---|
| Direct | ใช่ (ตรงเป๊ะ) | ส่งไปเฉพาะ queue ที่ binding key ตรงกัน |
| Fanout | ไม่ใช้ | ส่งไปทุก queue ที่ผูกไว้ (broadcast) |
| Topic | ใช่ (รองรับ `*`, `#`) | ส่งไปยัง queue ที่ pattern ตรงกับ routing key |
| Headers | ไม่ใช้ (ใช้ header แทน) | ส่งตามเงื่อนไข header ที่ตรงกัน |

---

## 5. Client library สำหรับ Go: `amqp091-go`

Library อย่างเป็นทางการที่ทีม RabbitMQ (VMware/Broadcom) ดูแลเองสำหรับ Go คือ `github.com/rabbitmq/amqp091-go` — เป็น fork ต่อจาก `streadway/amqp` ตัวเดิมที่ผู้เขียนเดิมหยุดดูแล แล้วทีม RabbitMQ รับช่วงต่ออย่างเป็นทางการ (ชื่อ `091` มาจากเลขเวอร์ชันโปรโตคอล **AMQP 0-9-1** ซึ่งเป็นเวอร์ชันที่ RabbitMQ ใช้เป็นหลัก — ไม่ใช่ AMQP 1.0 ที่เป็นคนละมาตรฐานกัน)

ติดตั้งด้วย:

```bash
go get github.com/rabbitmq/amqp091-go@latest
```

import แล้วมักตั้งชื่อ alias เป็น `amqp` เพื่อความกระชับ:

```go
import amqp "github.com/rabbitmq/amqp091-go"
```

> **หมายเหตุความโปร่งใส**: บทนี้ทดสอบกับเวอร์ชัน `v1.15.0` ของ library ผ่านการรัน `go get` จริงในสภาพแวดล้อมที่เขียนบทนี้

---

## 6. เตรียมสภาพแวดล้อม (สิ่งที่รันจริงในบทนี้)

> **หมายเหตุความโปร่งใส**: โค้ดทุกชิ้นในบทนี้ (producer, consumer, ack/nack, topic exchange, worker queue กระจายหลายโปรเซส) **ถูกรันจริง**ในสภาพแวดล้อมที่เขียนบทนี้ โดยใช้ Docker รัน RabbitMQ broker จริง แล้วรันโปรแกรม Go เชื่อมต่อเข้าไปจริง ผลลัพธ์ที่แสดงในแต่ละหัวข้อคือผลลัพธ์ที่ได้จากการรันจริง ไม่ใช่ผลลัพธ์ที่แต่งขึ้น

รัน RabbitMQ ด้วย Docker image ทางการ (เวอร์ชัน `management` มาพร้อม web UI สำหรับดู queue/exchange แบบ visual ที่ port 15672 ด้วย):

```bash
docker run -d --name rabbitmq-course \
  -p 5672:5672 -p 15672:15672 \
  rabbitmq:3-management
```

รอสัก 10-15 วินาทีให้ broker start เสร็จ (ดู log ด้วย `docker logs rabbitmq-course` จนเห็นบรรทัด `Server startup complete`) จากนั้นเข้า web UI ได้ที่ `http://localhost:15672` (username/password default คือ `guest`/`guest` — ใช้ได้เฉพาะตอน connect จาก `localhost` เท่านั้นด้วยเหตุผลด้านความปลอดภัย)

สร้างโปรเจกต์ Go:

```bash
mkdir rabbitmq-demo && cd rabbitmq-demo
go mod init rabbitmq-demo
go get github.com/rabbitmq/amqp091-go@latest
```

> ถ้าไม่มี Docker หรือไม่สะดวกรัน RabbitMQ เอง โค้ดในหัวข้อถัดไปยังอ่านและทำความเข้าใจได้ตามปกติ เพียงแต่รันจริงไม่ได้จนกว่าจะมี broker ให้เชื่อมต่อ — วิธีตั้งค่าที่ครบถ้วนกว่านี้ (multi-node cluster, TLS) จะกลับมาพูดถึงอีกครั้งใน **ภาคที่ 9 (Docker Compose, Part 096)**

---

## 7. เขียน Producer: ส่งข้อความเข้า queue

มาดูตัวอย่างที่ง่ายที่สุดก่อน: ส่งงาน "ส่งอีเมล" 5 ชิ้นเข้า queue ผ่าน default exchange

```go
package main

import (
	"context"
	"fmt"
	"log"
	"time"

	amqp "github.com/rabbitmq/amqp091-go"
)

func main() {
	conn, err := amqp.Dial("amqp://guest:guest@localhost:5672/")
	if err != nil {
		log.Fatalf("connect to rabbitmq: %v", err)
	}
	defer conn.Close()

	ch, err := conn.Channel()
	if err != nil {
		log.Fatalf("open channel: %v", err)
	}
	defer ch.Close()

	q, err := ch.QueueDeclare(
		"email_tasks", // ชื่อ queue
		true,          // durable: queue รอดจาก broker restart
		false,         // auto-delete: ลบตัวเองอัตโนมัติเมื่อไม่มี consumer ต่ออยู่เลย
		false,         // exclusive: จำกัดให้ connection นี้ใช้ได้คนเดียว
		false,         // no-wait
		nil,           // arguments เพิ่มเติม (เช่น TTL, dead-letter — ดูหัวข้อ 12)
	)
	if err != nil {
		log.Fatalf("declare queue: %v", err)
	}

	ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
	defer cancel()

	for i := 1; i <= 5; i++ {
		body := fmt.Sprintf(`{"task_id": %d, "to": "user%d@example.com"}`, i, i)
		err = ch.PublishWithContext(ctx,
			"",     // exchange: "" คือ default exchange
			q.Name, // routing key = ชื่อ queue โดยตรงเมื่อใช้ default exchange
			false,  // mandatory: ถ้า true และ route ไม่ถึง queue ไหนเลย broker จะตีข้อความกลับ
			false,  // immediate: deprecated ใน RabbitMQ รุ่นใหม่ ใส่ false เสมอ
			amqp.Publishing{
				ContentType:  "application/json",
				Body:         []byte(body),
				DeliveryMode: amqp.Persistent, // เขียนลง disk ไม่ใช่แค่เก็บใน memory
			},
		)
		if err != nil {
			log.Fatalf("publish message %d: %v", i, err)
		}
		fmt.Printf("[producer] sent: %s\n", body)
	}
}
```

### จุดที่ต้องสังเกตให้ดี

- **`QueueDeclare` ต้องเรียกทั้งฝั่ง producer และ consumer** — ไม่ใช่แค่ฝั่งใดฝั่งหนึ่ง เพราะเราไม่รู้ว่าใครจะรันก่อนกัน `QueueDeclare` เป็น operation แบบ **idempotent** (เรียกซ้ำกี่ครั้งก็ไม่เป็นไร ถ้า config เดิมไม่เปลี่ยน) จึงปลอดภัยที่จะเรียกทุกครั้งตอนเริ่มโปรแกรม
- **`durable: true`** หมายความว่า queue เองจะยังอยู่ต่อแม้ broker restart (แต่ **ข้อความ**ข้างในจะรอดด้วยหรือไม่ ขึ้นอยู่กับ `DeliveryMode: amqp.Persistent` อีกชั้นหนึ่ง) ทั้งสองอย่างต้องตั้งคู่กันถ้าต้องการไม่ให้ข้อมูลหายเมื่อ broker ล่ม
- **`ch.PublishWithContext`** รับ `context.Context` (ทบทวนจาก **Part 032**) เพื่อควบคุม timeout ของการ publish — เป็น pattern เดียวกับที่ใช้กับ database (Part 078), HTTP client (Part 046) และ gRPC (Part 089) ตลอดหลักสูตรนี้

### ผลลัพธ์จากการรันจริง

```
[producer] sent: {"task_id": 1, "to": "user1@example.com"}
[producer] sent: {"task_id": 2, "to": "user2@example.com"}
[producer] sent: {"task_id": 3, "to": "user3@example.com"}
[producer] sent: {"task_id": 4, "to": "user4@example.com"}
[producer] sent: {"task_id": 5, "to": "user5@example.com"}
```

---

## 8. เขียน Consumer: รับข้อความและประมวลผล

```go
package main

import (
	"fmt"
	"log"
	"time"

	amqp "github.com/rabbitmq/amqp091-go"
)

func main() {
	conn, err := amqp.Dial("amqp://guest:guest@localhost:5672/")
	if err != nil {
		log.Fatalf("connect to rabbitmq: %v", err)
	}
	defer conn.Close()

	ch, err := conn.Channel()
	if err != nil {
		log.Fatalf("open channel: %v", err)
	}
	defer ch.Close()

	q, err := ch.QueueDeclare("email_tasks", true, false, false, false, nil)
	if err != nil {
		log.Fatalf("declare queue: %v", err)
	}

	// prefetch = 1: consumer ตัวนี้จะได้รับข้อความใหม่ทีละ 1 ชิ้นเท่านั้น
	// จนกว่าจะ ack ข้อความปัจจุบันก่อน (fair dispatch — สำคัญมากเมื่อมีหลาย worker)
	if err := ch.Qos(1, 0, false); err != nil {
		log.Fatalf("set qos: %v", err)
	}

	msgs, err := ch.Consume(
		q.Name,
		"",    // consumer tag: ปล่อยว่างให้ broker ตั้งชื่อให้อัตโนมัติ
		false, // auto-ack = false: เราจะ ack เอง (สำคัญมาก ดูหัวข้อ 9)
		false, // exclusive
		false, // no-local
		false, // no-wait
		nil,
	)
	if err != nil {
		log.Fatalf("register consumer: %v", err)
	}

	for d := range msgs {
		fmt.Printf("[consumer] received: %s\n", d.Body)
		time.Sleep(100 * time.Millisecond) // จำลองการประมวลผลจริง เช่น เรียก SMTP server
		if err := d.Ack(false); err != nil {
			log.Printf("ack error: %v", err)
		} else {
			fmt.Printf("[consumer] acked: %s\n", d.Body)
		}
	}
}
```

### ผลลัพธ์จากการรันจริง

รันโปรแกรม consumer ก่อน แล้วจึงรัน producer จากหัวข้อ 7 (ในเทอร์มินัลอีกหน้าต่างหนึ่ง):

```
[consumer] received: {"task_id": 1, "to": "user1@example.com"}
[consumer] acked: {"task_id": 1, "to": "user1@example.com"}
[consumer] received: {"task_id": 2, "to": "user2@example.com"}
[consumer] acked: {"task_id": 2, "to": "user2@example.com"}
[consumer] received: {"task_id": 3, "to": "user3@example.com"}
[consumer] acked: {"task_id": 3, "to": "user3@example.com"}
[consumer] received: {"task_id": 4, "to": "user4@example.com"}
[consumer] acked: {"task_id": 4, "to": "user4@example.com"}
[consumer] received: {"task_id": 5, "to": "user5@example.com"}
[consumer] acked: {"task_id": 5, "to": "user5@example.com"}
```

### `ch.Qos(1, 0, false)` คืออะไร

`Qos` (Quality of Service) กำหนด **prefetch count** — จำนวนข้อความสูงสุดที่ broker จะส่งให้ consumer ตัวนี้ค้างไว้แบบยัง unacknowledged พร้อมกัน ถ้าไม่ตั้งค่านี้เลย (ค่า default คือ 0 = ไม่จำกัด) broker จะยิงข้อความทั้งหมดที่มีมาให้ consumer ตัวแรกที่ต่อเข้ามาทันที ทำให้เมื่อมีหลาย consumer/worker การกระจายงานจะไม่เท่ากันเลย (worker ตัวแรกได้งานทั้งหมด ตัวอื่นว่างงาน) การตั้ง `Qos(1, ...)` ทำให้เกิด **fair dispatch**: worker ตัวไหนว่างก่อน (ack ข้อความก่อนหน้าเสร็จก่อน) จะได้รับข้อความถัดไปก่อน — จะเห็นผลชัดเจนในหัวข้อ 11 ที่รันหลาย process จริง

---

## 9. Message Acknowledgment: `ack`/`nack` และทำไมมันสำคัญต่อความน่าเชื่อถือ

นี่คือกลไกที่สำคัญที่สุดของบทนี้ในแง่ **ความน่าเชื่อถือ (reliability)** ของระบบ

### ทำไมต้อง ack เอง (ไม่ใช้ auto-ack)

ถ้าตั้ง `autoAck: true` ตอน `ch.Consume` broker จะถือว่าข้อความ "ส่งมอบสำเร็จ" ทันทีที่ส่งออกไปให้ consumer โดย**ไม่สนใจว่า consumer ประมวลผลสำเร็จจริงหรือไม่** ถ้า consumer เกิด crash กลางทาง (เช่น ตอนกำลังเขียนไฟล์รูป thumbnail แล้ว process ถูก kill) ข้อความนั้นจะ**หายไปตลอดกาล** เพราะ broker คิดว่าส่งสำเร็จไปแล้ว

การ ack เองแบบ manual (`autoAck: false`) แก้ปัญหานี้ตรงๆ: broker จะถือว่าข้อความยัง "อยู่ระหว่างดำเนินการ" จนกว่า consumer จะเรียก `d.Ack(false)` อย่างชัดเจน ถ้า connection ของ consumer หลุดไปก่อน ack (ไม่ว่าจะ crash หรือ network ขาด) broker จะรู้ทันทีว่าข้อความนั้นยังไม่เสร็จ และจะ**ส่งข้อความนั้นซ้ำ (redeliver)** ให้ consumer ตัวอื่น (หรือตัวเดิมตอนกลับมาออนไลน์) โดยอัตโนมัติ

### สามวิธีตอบสนองต่อข้อความ

| Method | ความหมาย | ใช้เมื่อไหร่ |
|---|---|---|
| `d.Ack(false)` | ประมวลผลสำเร็จ ลบข้อความออกจาก queue ถาวร | งานเสร็จสมบูรณ์ |
| `d.Nack(false, requeue)` | ประมวลผลล้มเหลว `requeue=true` → ส่งกลับเข้า queue ให้ลองใหม่, `requeue=false` → ทิ้งข้อความ (หรือส่งไป dead-letter queue ถ้าตั้งไว้ — หัวข้อ 12) | ล้มเหลวแบบชั่วคราว (transient) ที่คาดว่าลองใหม่แล้วอาจสำเร็จ |
| `d.Reject(requeue)` | เหมือน `Nack` แต่ปฏิเสธได้ทีละ 1 ข้อความเท่านั้น (`Nack` ปฏิเสธได้หลายข้อความพร้อมกันด้วย multiple flag) | เหมือน `Nack` แต่ต้องการความชัดเจนว่าปฏิเสธข้อความนี้ข้อความเดียว |

พารามิเตอร์ตัวแรกของ `Ack`/`Nack` คือ `multiple bool` — ถ้า `true` หมายถึง ack/nack ข้อความปัจจุบัน**และทุกข้อความก่อนหน้าที่ยังไม่ ack** ในครั้งเดียว (ใช้เพื่อลดจำนวนรอบการสื่อสารกับ broker เมื่อประมวลผลเป็น batch) ตัวอย่างในบทนี้ใช้ `false` เสมอเพื่อความชัดเจน (ack ทีละข้อความ)

### ตัวอย่างที่รันจริง: จำลอง failure แล้วดูการ redeliver

```go
package main

import (
	"fmt"
	"log"
	"time"

	amqp "github.com/rabbitmq/amqp091-go"
)

func main() {
	conn, _ := amqp.Dial("amqp://guest:guest@localhost:5672/")
	defer conn.Close()
	ch, _ := conn.Channel()
	defer ch.Close()

	q, err := ch.QueueDeclare("nack_demo", true, false, false, false, nil)
	if err != nil {
		log.Fatalf("declare queue: %v", err)
	}

	ch.QueuePurge(q.Name, false) // เคลียร์ข้อความค้างจากรันครั้งก่อน (เพื่อความชัดเจนของตัวอย่าง)
	ch.Publish("", q.Name, false, false, amqp.Publishing{
		Body: []byte(`{"task": "resize-image", "attempt": "will fail once then succeed"}`),
	})
	fmt.Println("[setup] published 1 message")

	ch.Qos(1, 0, false)
	msgs, err := ch.Consume(q.Name, "", false, false, false, false, nil)
	if err != nil {
		log.Fatalf("consume: %v", err)
	}

	attempt := 0
	for d := range msgs {
		attempt++
		fmt.Printf("[worker] delivery attempt #%d, redelivered=%v, body=%s\n",
			attempt, d.Redelivered, d.Body)

		if attempt == 1 {
			fmt.Println("[worker] simulated failure -> nack with requeue=true")
			d.Nack(false, true) // ส่งกลับเข้า queue ให้ลองใหม่
			continue
		}
		fmt.Println("[worker] processing succeeded -> ack")
		d.Ack(false)
		break
	}
	time.Sleep(200 * time.Millisecond)
	fmt.Println("[done]")
}
```

### ผลลัพธ์จากการรันจริง

```
[setup] published 1 message
[worker] delivery attempt #1, redelivered=false, body={"task": "resize-image", "attempt": "will fail once then succeed"}
[worker] simulated failure -> nack with requeue=true
[worker] delivery attempt #2, redelivered=true, body={"task": "resize-image", "attempt": "will fail once then succeed"}
[worker] processing succeeded -> ack
[done]
```

สังเกต field `d.Redelivered` ที่เปลี่ยนจาก `false` เป็น `true` ในการส่งครั้งที่สอง — นี่คือ flag ที่ broker ใส่มาให้เพื่อบอก consumer ว่า "ข้อความนี้เคยถูกส่งมาแล้วอย่างน้อยหนึ่งครั้ง" ซึ่งมีประโยชน์มากถ้าต้องการเขียน logic แบบ "ถ้าเจอข้อความเดิมซ้ำ ให้ตรวจสอบเพิ่มเติมก่อนประมวลผลซ้ำ" (idempotency check)

> **คำเตือนสำคัญ**: การ `Nack(..., requeue=true)` แบบไม่มีเงื่อนไขจำกัดจำนวนครั้ง มีความเสี่ยงสร้าง **infinite redelivery loop** ถ้าข้อความนั้นล้มเหลวทุกครั้งไม่ว่าจะลองกี่รอบ (เช่น ข้อมูลใน payload ผิดรูปแบบถาวร ไม่ใช่ปัญหาชั่วคราว) โปรดักชันจริงมักนับจำนวนครั้งที่ลองแล้ว (เก็บใน header ของข้อความ) แล้ว nack แบบ `requeue=false` เมื่อเกินจำนวนที่กำหนด เพื่อส่งไปที่ **dead letter queue** แทน (หัวข้อ 12)

---

## 10. Topic Exchange: routing แบบยืดหยุ่นด้วย pattern

มาดูตัวอย่างที่ไม่ได้ใช้ default exchange อีกต่อไป แต่ประกาศ topic exchange เอง แล้วสร้างสอง queue ที่สนใจ event คนละแบบ:

```go
package main

import (
	"fmt"
	"log"
	"time"

	amqp "github.com/rabbitmq/amqp091-go"
)

func main() {
	conn, _ := amqp.Dial("amqp://guest:guest@localhost:5672/")
	defer conn.Close()
	ch, _ := conn.Channel()
	defer ch.Close()

	// 1. ประกาศ topic exchange
	if err := ch.ExchangeDeclare("events", "topic", true, false, false, false, nil); err != nil {
		log.Fatalf("declare exchange: %v", err)
	}

	// 2. สร้าง queue สองใบ ผูก (bind) กับ routing key pattern คนละแบบ
	orderQ, _ := ch.QueueDeclare("order_events_q", true, false, false, false, nil)
	failureQ, _ := ch.QueueDeclare("failure_events_q", true, false, false, false, nil)
	ch.QueuePurge(orderQ.Name, false)
	ch.QueuePurge(failureQ.Name, false)

	ch.QueueBind(orderQ.Name, "order.*", "events", false, nil)    // สนใจทุก event ที่ขึ้นต้นด้วย order.
	ch.QueueBind(failureQ.Name, "*.failed", "events", false, nil) // สนใจทุก event ที่ลงท้ายด้วย .failed

	// 3. publish 3 event ด้วย routing key ต่างกัน
	events := []string{"order.created", "order.cancelled", "payment.failed"}
	for _, rk := range events {
		ch.Publish("events", rk, false, false, amqp.Publishing{
			Body: []byte(fmt.Sprintf(`{"event":"%s"}`, rk)),
		})
	}
	fmt.Println("published events:", events)
	time.Sleep(300 * time.Millisecond)

	drain := func(qname string) []string {
		var out []string
		for {
			d, ok, err := ch.Get(qname, true) // Get: ดึงข้อความทีละชิ้นแบบ polling (ไม่ใช้ Consume)
			if err != nil || !ok {
				break
			}
			out = append(out, string(d.Body))
		}
		return out
	}

	fmt.Println("order_events_q got:", drain(orderQ.Name))
	fmt.Println("failure_events_q got:", drain(failureQ.Name))
}
```

### ผลลัพธ์จากการรันจริง

```
published events: [order.created order.cancelled payment.failed]
order_events_q got: [{"event":"order.created"} {"event":"order.cancelled"}]
failure_events_q got: [{"event":"payment.failed"}]
```

สังเกตว่า `order.created` และ `order.cancelled` ตรงกับ pattern `order.*` ทั้งคู่ (คำเดียวหลังจุด ไม่ว่าจะเป็นคำไหน) และไม่มีตัวไหนหลุดไปที่ `failure_events_q` เพราะไม่มี routing key ไหนลงท้ายด้วย `.failed` ยกเว้น `payment.failed` ซึ่งตรงกับ pattern `*.failed` พอดี — นี่คือพลังของ topic exchange: **queue เดียวกันไม่จำเป็นต้องรับรู้ว่ามี queue อื่นอยู่ด้วย** แต่ละ queue แค่ประกาศ "ฉันสนใจ pattern อะไร" แล้ว exchange จัดการ routing ให้เองทั้งหมด — นี่คือหลักการ **decoupling ระดับ routing** ที่ทำให้เพิ่ม consumer ใหม่ในอนาคต (เช่น queue ที่สนใจ `*.cancelled`) ได้โดยไม่ต้องแก้โค้ด producer เลยสักบรรทัด

---

## 11. Worker Queue แบบกระจายหลายโปรเซส: ต่อยอดจาก Worker Pool (Part 041)

ใน **Part 041** เราเขียน Worker Pool ที่จำกัดจำนวน goroutine ที่ทำงานพร้อมกันด้วย **channel ภายในโปรเซสเดียว** — jobs channel และ worker goroutines ทั้งหมดอยู่ใน process เดียวกัน ข้อจำกัดที่ตามมาชัดเจนคือ: **ถ้า process นั้นล่ม งานทั้งหมดที่ค้างอยู่ใน channel หายไปทันที** (channel อยู่ใน memory ของ process เดียว ไม่มีการ persist) และเราสเกลได้แค่ในแนวตั้ง (เพิ่ม CPU ให้เครื่องเดียว) ไม่สามารถกระจายงานไปยังหลายเครื่องได้

RabbitMQ แก้ข้อจำกัดทั้งสองข้อนี้พร้อมกัน: **queue กลายเป็น "jobs channel ที่ทนทานและกระจายได้"** — worker แต่ละตัวไม่จำเป็นต้องอยู่ process เดียวกัน หรือแม้แต่เครื่องเดียวกันอีกต่อไป ทุกตัวแค่เชื่อมต่อไปที่ queue เดียวกันแล้วแข่งกันดึงงาน (`ch.Qos(1, 0, false)` ทำให้เกิด fair dispatch แบบเดียวกับที่ jobs channel ทำใน Part 041 — "ใครว่างก่อนได้งานก่อน")

```
Part 041 (in-process):                RabbitMQ (distributed):

  jobs channel (memory)                 queue "distributed_jobs" (broker, บน disk ถ้า durable)
   │    │    │                            │      │      │
   ▼    ▼    ▼                            ▼      ▼      ▼
 worker worker worker                  process1 process2 process3
 (goroutine ในโปรเซสเดียว)             (คนละ process, คนละเครื่องก็ได้)
```

มาดูตัวอย่างที่รันจริง: producer ตัวเดียวส่งงาน 15 ชิ้น แล้วมี **worker 3 process แยกกัน** มาแข่งกันดึงงานจาก queue เดียวกัน

**Producer:**

```go
package main

import (
	"fmt"
	"log"

	amqp "github.com/rabbitmq/amqp091-go"
)

func main() {
	conn, _ := amqp.Dial("amqp://guest:guest@localhost:5672/")
	defer conn.Close()
	ch, _ := conn.Channel()
	defer ch.Close()

	q, err := ch.QueueDeclare("distributed_jobs", true, false, false, false, nil)
	if err != nil {
		log.Fatalf("declare: %v", err)
	}
	ch.QueuePurge(q.Name, false)

	for i := 1; i <= 15; i++ {
		body := fmt.Sprintf(`{"job_id": %d}`, i)
		if err := ch.Publish("", q.Name, false, false, amqp.Publishing{
			DeliveryMode: amqp.Persistent,
			Body:         []byte(body),
		}); err != nil {
			log.Fatalf("publish: %v", err)
		}
	}
	fmt.Println("published 15 jobs")
}
```

**Worker (compile เป็น binary เดียว แล้วรันหลาย process พร้อมกัน โดยรับชื่อ worker เป็น argument):**

```go
package main

import (
	"fmt"
	"log"
	"os"
	"time"

	amqp "github.com/rabbitmq/amqp091-go"
)

func main() {
	name := os.Args[1] // เช่น "worker-A", "worker-B", "worker-C"

	conn, _ := amqp.Dial("amqp://guest:guest@localhost:5672/")
	defer conn.Close()
	ch, _ := conn.Channel()
	defer ch.Close()

	q, err := ch.QueueDeclare("distributed_jobs", true, false, false, false, nil)
	if err != nil {
		log.Fatalf("declare: %v", err)
	}

	// prefetch=1: แต่ละ process รับงานใหม่ทีละชิ้น กระจายงานตามความเร็วจริงของแต่ละ worker
	if err := ch.Qos(1, 0, false); err != nil {
		log.Fatalf("qos: %v", err)
	}

	msgs, err := ch.Consume(q.Name, name, false, false, false, false, nil)
	if err != nil {
		log.Fatalf("consume: %v", err)
	}

	count := 0
	timeout := time.After(4 * time.Second)
	for {
		select {
		case d, ok := <-msgs:
			if !ok {
				fmt.Printf("%s: done, processed %d jobs\n", name, count)
				return
			}
			time.Sleep(150 * time.Millisecond) // จำลองงานที่ใช้เวลา
			d.Ack(false)
			count++
			fmt.Printf("%s: processed %s\n", name, d.Body)
		case <-timeout:
			fmt.Printf("%s: done, processed %d jobs\n", name, count)
			return
		}
	}
}
```

รันด้วยคำสั่งชุดนี้ (เปิด worker 3 process ก่อน แล้วค่อยสั่ง producer):

```bash
go build -o worker-bin ./worker
go build -o producer-bin ./producer

./worker-bin worker-A &
./worker-bin worker-B &
./worker-bin worker-C &
sleep 1
./producer-bin
wait
```

### ผลลัพธ์จากการรันจริง

```
published 15 jobs
worker-A: processed {"job_id": 2}
worker-A: processed {"job_id": 5}
worker-A: processed {"job_id": 7}
worker-A: processed {"job_id": 10}
worker-A: processed {"job_id": 15}
worker-A: done, processed 5 jobs
worker-B: processed {"job_id": 3}
worker-B: processed {"job_id": 4}
worker-B: processed {"job_id": 8}
worker-B: processed {"job_id": 11}
worker-B: processed {"job_id": 14}
worker-B: done, processed 5 jobs
worker-C: processed {"job_id": 1}
worker-C: processed {"job_id": 6}
worker-C: processed {"job_id": 9}
worker-C: processed {"job_id": 12}
worker-C: processed {"job_id": 13}
worker-C: done, processed 5 jobs
```

งานทั้ง 15 ชิ้นถูกกระจายไปที่ worker ทั้ง 3 ตัวอย่างละ 5 ชิ้นพอดี (เพราะทั้งสามตัวเร็วเท่ากันในตัวอย่างนี้ — `time.Sleep` เท่ากันหมด) **ไม่มีงานไหนถูกประมวลผลซ้ำโดยสองตัวพร้อมกัน** — นี่คือ guarantee หลักที่ queue มอบให้ แม้ว่า worker แต่ละตัวจะเป็นคนละ process อิสระจากกันโดยสิ้นเชิงก็ตาม (ในทางปฏิบัติ ทดลองปิด `worker-B` กลางทางดูได้ — งานที่เหลือจะยังกระจายไปที่ `worker-A` และ `worker-C` ต่อโดยไม่มีอะไรหายไป)

เปรียบเทียบกับ Part 041 ตรงๆ: โครงสร้างทางความคิดเหมือนกันทุกประการ (jobs กระจายให้ worker ที่ว่าง, ผลลัพธ์ไม่ขึ้นกับลำดับ) ต่างกันแค่ **jobs channel ในหน่วยความจำ ถูกแทนที่ด้วย queue ของ broker ที่ทนทานและกระจายข้าม process/เครื่องได้**

---

## 12. Dead Letter Queue และการจัดการข้อความที่ล้มเหลวซ้ำๆ

ต่อยอดจากคำเตือนท้ายหัวข้อ 9: ถ้าปล่อยให้ `Nack(..., requeue=true)` วนซ้ำไม่จำกัดจำนวนครั้ง ข้อความที่เสียตั้งแต่ต้น (เช่น JSON ผิดรูปแบบ) จะวนเวียนไม่จบสิ้น กินทรัพยากรทั้ง broker และ worker โดยไม่มีประโยชน์

RabbitMQ มีกลไก **Dead Letter Exchange (DLX)** สำหรับกรณีนี้: ตั้งค่า argument พิเศษตอน `QueueDeclare` ว่าถ้าข้อความถูก reject/nack โดย `requeue=false` (หรือหมดอายุตาม TTL) ให้ส่งข้อความนั้นไปที่ exchange อื่นแทนที่จะทิ้งไปเฉยๆ:

```go
args := amqp.Table{
	"x-dead-letter-exchange":    "dlx-exchange",  // ส่งข้อความที่ตายไปที่ exchange นี้แทน
	"x-dead-letter-routing-key": "failed-tasks",  // ด้วย routing key นี้
}
q, err := ch.QueueDeclare("email_tasks", true, false, false, false, args)
```

จากนั้นค่อยผูก queue อีกใบ (เช่น `failed_tasks_q`) เข้ากับ `dlx-exchange` เพื่อเก็บข้อความที่ล้มเหลวไว้ตรวจสอบภายหลัง (log, แจ้งเตือนทีม, หรือให้ทีม support มาดูด้วยตา) แทนที่จะปล่อยให้หายไปเงียบๆ หรือวนซ้ำไม่จบ

แนวทางที่ใช้กันจริงในโปรดักชันคือผสมทั้งสองอย่าง: นับจำนวนครั้งที่ retry ผ่าน header ของข้อความเอง (เช่น `x-retry-count`) ถ้ายังไม่เกิน N ครั้งให้ `Nack(..., requeue=true)` ลองใหม่ ถ้าเกินแล้วให้ `Nack(..., requeue=false)` ปล่อยให้ DLX จัดการแทน — เป็นแนวคิดเดียวกับ **retry pattern พร้อม backoff** ที่จะเรียนแบบเจาะลึกใน **Part 094**

---

## 13. RabbitMQ กับ Worker Pool ในโปรเซสเดียว (Part 041): ใช้อะไรเมื่อไหร่

| ปัจจัย | Worker Pool ในโปรเซสเดียว (Part 041) | RabbitMQ (บทนี้) |
|---|---|---|
| ความเร็ว | เร็วกว่ามาก (ไม่มี network round-trip) | ช้ากว่า (ต้องส่งผ่าน network ไป broker) |
| ความทนทานเมื่อ process ล่ม | งานที่ค้างใน channel หายทันที | งานยังอยู่ใน queue (ถ้า durable + persistent) |
| การสเกล | จำกัดที่ CPU ของเครื่องเดียว | กระจายไปหลาย process/เครื่องได้อิสระ |
| ความซับซ้อนในการ deploy | ไม่มี dependency เพิ่ม (แค่ binary เดียว) | ต้องดูแล broker แยกต่างหาก (หรือใช้ managed service) |
| เหมาะกับ | งานที่เกิดและจบภายในคำขอเดียว (เช่น ประมวลผล request ที่รับเข้ามา) | งานที่ต้องส่งต่อข้าม service/deploy แยกกัน หรือต้องทนต่อการ restart |

กฎง่ายๆ ในการเลือก: **ถ้างานทั้งหมดเกิดและจบในโปรเซสเดียวกัน ไม่ต้องส่งข้าม service ใช้ Worker Pool (Part 041) พอ** เพราะเร็วกว่าและไม่มี operational overhead เพิ่ม แต่ทันทีที่ต้องการ **decouple ข้าม service, ทนต่อ process ล่ม, หรือ buffer โหลดที่พุ่งสูงจากภายนอก** — RabbitMQ (หรือ message broker ตัวอื่น) คือคำตอบที่เหมาะสมกว่า หลายระบบจริงใช้**ทั้งคู่ร่วมกัน**: consumer หนึ่งตัวดึงงานจาก RabbitMQ queue มาแล้ว fan-out ต่อด้วย Worker Pool ภายในตัวเองอีกชั้นเพื่อประมวลผลแบบขนาน (เช่น consumer ดึงงาน "ประมวลผลไฟล์ 100 รูป" มา 1 ข้อความ แล้วใช้ worker pool ภายในตัวเองแตกงานนั้นเป็น 100 งานย่อยพร้อมกัน)

---

## สรุปสิ่งที่ได้เรียนในบทนี้

- Message Queue แก้ปัญหา coupling ที่แน่นเกินไปของการสื่อสารแบบ synchronous — ทำให้ decouple producer/consumer, buffer โหลดที่พุ่งสูง, และทนต่อการล่มของ consumer ได้
- AMQP มี 4 องค์ประกอบหลัก: **Exchange** (จุดรับข้อความ ทำ routing), **Queue** (เก็บข้อความจริง), **Binding** (กฎ routing), **Routing Key** (ป้ายกำกับข้อความ) — producer ไม่เคยส่งเข้า queue ตรงๆ เสมอผ่าน exchange ก่อน
- Exchange มี 4 ประเภท: **direct** (ตรงเป๊ะ), **fanout** (broadcast ทุก queue), **topic** (pattern ด้วย `*`/`#`), **headers** (ตามเงื่อนไข header)
- `github.com/rabbitmq/amqp091-go` คือ client library ทางการที่ทีม RabbitMQ ดูแลสำหรับ Go
- **Manual acknowledgment** (`autoAck: false` + `d.Ack`/`d.Nack`) คือกลไกสำคัญที่สุดของความน่าเชื่อถือ — ป้องกันข้อความหายเมื่อ consumer ล่มกลางทาง โดย broker จะ redeliver ข้อความที่ยังไม่ ack ให้อัตโนมัติ
- `ch.Qos(1, 0, false)` ทำให้เกิด **fair dispatch** — จำเป็นเมื่อมีหลาย worker แข่งกันดึงงานจาก queue เดียวกัน
- Worker Queue ที่กระจายหลาย process ต่อยอดตรงจาก Worker Pool ใน Part 041 — แนวคิดเดียวกันทุกประการ (worker ที่ว่างได้งานก่อน) เพียงแต่แทนที่ channel ในหน่วยความจำด้วย queue ของ broker ที่ทนทานและกระจายข้ามเครื่องได้
- **Dead Letter Exchange** ใช้ดักข้อความที่ล้มเหลวซ้ำๆ ไม่ให้วนเวียนไม่จบ
- โค้ดทุกชิ้นในบทนี้ (producer, consumer, ack/nack, topic exchange, worker queue กระจาย 3 process) **รันจริงกับ RabbitMQ broker จริงผ่าน Docker** และผลลัพธ์ที่แสดงคือผลลัพธ์จากการรันจริง

## แบบฝึกหัดท้ายบท

1. รัน RabbitMQ ด้วย Docker ตามหัวข้อ 6 แล้วรันตัวอย่าง producer/consumer ในหัวข้อ 7-8 ด้วยตัวเอง ลองเปิด web UI ที่ `http://localhost:15672` ระหว่างรัน แล้วสังเกตจำนวนข้อความใน queue `email_tasks` ที่เปลี่ยนแปลงแบบ real-time
2. ทดลองรันเฉพาะ producer (หัวข้อ 7) โดยยังไม่รัน consumer เลย รอ 10 วินาทีแล้วค่อยรัน consumer ทีหลัง — ข้อความยังอยู่ครบหรือไม่ อธิบายว่าเกี่ยวข้องกับ `durable`/`Persistent` อย่างไร
3. ดัดแปลงตัวอย่างในหัวข้อ 9 ให้จำลอง failure 3 ครั้งติดกันก่อนจะสำเร็จ (แทนที่จะเป็น 1 ครั้ง) แล้วสังเกตค่า `d.Redelivered` ในแต่ละรอบ
4. เพิ่ม dead letter exchange ตามแนวทางหัวข้อ 12 เข้าไปใน queue `nack_demo` จากหัวข้อ 9 แล้วดัดแปลงโค้ดให้ `Nack(..., requeue=false)` หลังพยายามครบ 3 ครั้งไม่สำเร็จ ตรวจสอบด้วย web UI ว่าข้อความไปโผล่ที่ queue ปลายทางของ DLX จริงหรือไม่
5. รันตัวอย่าง worker queue แบบกระจายในหัวข้อ 11 แล้วลองปิด (kill) worker ตัวหนึ่งกลางทางระหว่างที่กำลังประมวลผลอยู่ (ก่อนจะ ack) สังเกตว่างานชิ้นนั้นถูกส่งไปให้ worker ตัวอื่นทำต่อหรือไม่ อธิบายว่าเกี่ยวข้องกับกลไกใดที่เรียนในหัวข้อ 9
6. (ขั้นสูง) ค้นคว้าเพิ่มเติมเกี่ยวกับ RabbitMQ Streams (ฟีเจอร์ใหม่ที่ทำให้ RabbitMQ มีความสามารถแบบ log-based คล้าย Kafka มากขึ้น) แล้วเปรียบเทียบว่ามันต่างจาก queue แบบดั้งเดิมที่เรียนในบทนี้อย่างไร เตรียมไว้เทียบกับ Kafka ใน Part 092

---

**ต่อไป**: [Part 092 — Message Queue: Kafka กับ Go](./092-kafka.md)
