# Part 092: Message Queue: Kafka กับ Go

> ภาคที่ 8: Microservices, gRPC, Message Queue — ตอนที่ 5 จาก 7

## สารบัญของบทนี้

1. ทบทวน: จาก RabbitMQ (Part 091) สู่ Kafka
2. Kafka คืออะไร — mental model ที่ต่างจาก message broker ทั่วไป
3. สถาปัตยกรรมของ Kafka: broker, topic, partition, offset, replica
4. Kafka vs RabbitMQ: เปรียบเทียบให้เห็นภาพชัด ใช้อะไรเมื่อไหร่
5. Consumer Group: กลไกสเกลการอ่านและการ replay
6. Client library สำหรับ Go: `kafka-go` vs `confluent-kafka-go`
7. เตรียมสภาพแวดล้อม (สิ่งที่รันจริงในบทนี้)
8. เขียน Producer ด้วย kafka-go
9. เขียน Consumer แบบธรรมดา (ไม่มี group)
10. Consumer Group และ Partitioning: ตัวอย่างที่รันจริงครบวงจร
11. Replay-ability: ความสามารถที่ RabbitMQ ทำได้ยากกว่า
12. แนวทางออกแบบ topic/partition ในโปรดักชัน
13. สรุปสิ่งที่ได้เรียนในบทนี้
14. แบบฝึกหัดท้ายบท

---

## 1. ทบทวน: จาก RabbitMQ (Part 091) สู่ Kafka

ใน Part 091 เราเรียนเรื่อง RabbitMQ ไปแล้ว — message broker แบบดั้งเดิมที่ยึดโมเดล **queue**: producer ส่งข้อความเข้า exchange, exchange routing ไปที่ queue, consumer ดึงข้อความออกจาก queue แล้วข้อความนั้นก็ถูกลบทิ้ง (acknowledge แล้วหายไป) RabbitMQ เก่งเรื่อง routing ที่ซับซ้อน (direct, topic, fanout exchange), priority queue, และ delivery guarantee ที่ยืดหยุ่น เหมาะกับงานประเภท **task queue** — เช่น ส่งงานให้ worker ไปประมวลผลแล้วจบ

Kafka เป็นระบบ **message queue** เหมือนกันในความหมายกว้างๆ (ส่งข้อมูลจากจุดหนึ่งไปอีกจุดหนึ่งแบบ asynchronous ที่เราคุ้นเคยจาก Part 091) แต่ **โมเดลข้างในต่างจาก RabbitMQ อย่างสิ้นเชิง** มันไม่ใช่ queue แต่เป็น **distributed append-only log** การเข้าใจความต่างนี้คือหัวใจของบทนี้ เพราะมันกำหนดว่า Kafka เหมาะกับงานแบบไหน และทำไมบริษัทระดับ LinkedIn (ผู้สร้าง Kafka), Uber, Netflix, Twitter ถึงเลือกใช้ Kafka เป็นกระดูกสันหลังของระบบ event-driven ขนาดใหญ่

---

## 2. Kafka คืออะไร — mental model ที่ต่างจาก message broker ทั่วไป

### ลืมคำว่า "queue" ไปก่อน

วิธีที่เข้าใจ Kafka ได้ง่ายที่สุดคือคิดว่ามันเป็น **ไฟล์ log ที่เขียนต่อท้ายได้อย่างเดียว (append-only)** และกระจายอยู่บนหลายเครื่อง:

```
offset:   0    1    2    3    4    5    6    7    8  ...
          ┌────┬────┬────┬────┬────┬────┬────┬────┬────┐
partition │ m0 │ m1 │ m2 │ m3 │ m4 │ m5 │ m6 │ m7 │ m8 │ ...
          └────┴────┴────┴────┴────┴────┴────┴────┴────┘
                          ▲                    ▲
                    consumer A อ่านถึง    consumer B อ่านถึง
                    offset 3              offset 7
```

ความต่างที่สำคัญที่สุดจาก RabbitMQ:

| | RabbitMQ (queue) | Kafka (log) |
|---|---|---|
| เมื่อ consumer อ่านข้อความแล้ว | ข้อความหายไปจาก queue (หลัง ack) | ข้อความยัง**อยู่ที่เดิม**ใน log |
| ตำแหน่งการอ่าน | broker เป็นคนจำว่า consumer ไหน ack ไปถึงไหน | **consumer เป็นคนจำ offset ของตัวเอง** broker ไม่สนใจ |
| อ่านซ้ำ (replay) | ทำยากมาก ต้องมี dead-letter/redelivery logic เอง | อ่านซ้ำได้ทันที แค่ตั้ง offset ย้อนกลับ |
| หลาย consumer อ่านข้อความเดียวกัน | ต้องแยก queue ต่อ consumer เอง (fanout exchange) | หลาย consumer group อ่าน log เดียวกันได้อิสระ โดยธรรมชาติ |
| ลำดับข้อความ | รับประกันลำดับต่อ queue เดียว | รับประกันลำดับเฉพาะ**ภายใน partition เดียว**เท่านั้น |

พูดสั้นๆ: RabbitMQ คือ**ที่ส่งจดหมาย** (ส่งแล้วจบ, บุรุษไปรษณีย์ไม่เก็บสำเนา) ส่วน Kafka คือ**สมุดบันทึกสาธารณะที่เขียนต่อท้ายได้อย่างเดียว** (ใครจะมาอ่านตอนไหน อ่านซ้ำกี่รอบก็ได้ ตราบใดที่ข้อมูลยังไม่ถูกลบตาม retention policy)

---

## 3. สถาปัตยกรรมของ Kafka: broker, topic, partition, offset, replica

### องค์ประกอบหลัก

- **Broker**: เครื่อง (หรือ process) หนึ่งตัวในคลัสเตอร์ Kafka ทำหน้าที่เก็บข้อมูลและตอบ request จาก producer/consumer โปรดักชันจริงมักมีหลาย broker ต่อคลัสเตอร์
- **Topic**: ชื่อ "ช่องทาง" ของข้อมูล เทียบได้กับ queue ใน RabbitMQ แต่ topic ไม่ใช่ log เดี่ยว — มันถูกแบ่งย่อยเป็น partition
- **Partition**: หน่วยเก็บข้อมูลจริงที่เป็น append-only log แต่ละ topic มีได้หลาย partition (กระจายอยู่บนหลาย broker ได้) การมีหลาย partition คือกลไกที่ทำให้ Kafka **สเกลการเขียนและการอ่านแบบขนานได้**
- **Offset**: เลขลำดับของข้อความภายใน partition หนึ่งๆ (เริ่มจาก 0 เพิ่มขึ้นเรื่อยๆ) offset มีความหมายเฉพาะภายใน partition เดียวเท่านั้น (partition คนละอันมี offset 0 ของตัวเองแยกกัน)
- **Replica**: สำเนาของ partition ที่กระจายไปเก็บบน broker อื่นเพื่อความทนทาน (ถ้า broker ที่เป็น leader ของ partition ล่ม จะมี replica อื่นขึ้นมาเป็น leader แทน) ในบทนี้เราจะรันแบบ broker เดียว (`ReplicationFactor: 1`) เพื่อความง่าย แต่โปรดักชันจริงควรตั้งอย่างน้อย 3

### ทำไมต้องมีหลาย partition

Partition คือหน่วยของ**ความขนาน (parallelism)** ทั้งฝั่งเขียนและฝั่งอ่าน:

- **ฝั่งเขียน**: producer หลายตัวเขียนไปคนละ partition พร้อมกันได้โดยไม่ชนกัน
- **ฝั่งอ่าน**: consumer หลายตัวใน consumer group เดียวกันแบ่งกันอ่านคนละ partition ได้พร้อมกัน (รายละเอียดในหัวข้อ 5)

ข้อแลกเปลี่ยน (trade-off) ที่สำคัญ: Kafka รับประกันลำดับ (ordering) เฉพาะ**ภายใน partition เดียว**เท่านั้น ถ้าต้องการให้ข้อความที่เกี่ยวข้องกัน (เช่น event ทั้งหมดของ user คนเดียวกัน) เรียงลำดับกันเสมอ ต้องส่งโดยใช้ **key เดียวกัน** เพราะ Kafka จะ hash key แล้ว map ไปยัง partition เดิมเสมอ (เราจะเห็นโค้ดจริงในหัวข้อ 10)

### Retention: ข้อมูลอยู่ได้นานแค่ไหน

ต่างจาก RabbitMQ ที่ข้อความหายทันทีหลัง ack, Kafka เก็บข้อมูลไว้ตาม **retention policy** ที่ตั้งไว้ระดับ topic เช่น "เก็บไว้ 7 วัน" หรือ "เก็บไว้ไม่เกิน 100GB ต่อ partition" ไม่ว่าจะมี consumer อ่านไปแล้วหรือยัง ข้อมูลก็ยังอยู่จนกว่าจะหมดอายุตาม policy นี้เอง ทำให้ consumer ใหม่ที่เพิ่ง deploy วันนี้ยังย้อนไปอ่าน event ของเมื่อวานได้ ถ้า retention ยังไม่หมด

---

## 4. Kafka vs RabbitMQ: เปรียบเทียบให้เห็นภาพชัด ใช้อะไรเมื่อไหร่

ตารางเปรียบเทียบเชิงการใช้งานจริง ต่อยอดจากที่วางรากฐาน sync-vs-async ไว้ใน Part 088 และ RabbitMQ ใน Part 091:

| ปัจจัย | RabbitMQ เหมาะกว่า | Kafka เหมาะกว่า |
|---|---|---|
| รูปแบบงาน | Task queue: งานหนึ่งชิ้นควรถูกประมวลผล**ครั้งเดียว**แล้วจบ (เช่น ส่งอีเมล, ประมวลผลรูปภาพ) | Event streaming: เหตุการณ์ที่**หลายระบบอิสระ**ต้องการรับรู้ (เช่น "order ถูกสร้างแล้ว" ที่ทั้งระบบ billing, ระบบ inventory, ระบบ analytics ต้องรู้) |
| Throughput | หลักหมื่น-แสนข้อความ/วินาที (ขึ้นกับ routing ที่ซับซ้อนแค่ไหน) | หลักแสน-ล้านข้อความ/วินาที เพราะออกแบบมาเพื่อ sequential disk I/O และ zero-copy |
| การ replay ข้อมูลย้อนหลัง | ทำยาก ต้องออกแบบเพิ่มเอง | ทำได้ในตัว — reset consumer offset แล้วอ่านใหม่ตั้งแต่ต้น |
| Consumer หลายกลุ่มอ่านสตรีมเดียวกัน | ต้อง fanout exchange + สร้าง queue แยกต่อ consumer | Consumer group แต่ละกลุ่มอ่าน topic เดียวกันได้อิสระ ไม่ต้องตั้งค่าเพิ่ม |
| Routing ที่ซับซ้อน (priority, TTL ต่อข้อความ, delayed message) | แข็งแกร่งกว่ามาก มี exchange type หลายแบบ | ทำได้จำกัดกว่า ต้องออกแบบเอง |
| ความซับซ้อนในการ operate | ติดตั้ง/ดูแลง่ายกว่า | ต้องดูแล partition, replication, ZooKeeper/KRaft — ซับซ้อนกว่า |
| Message ordering | รับประกันตาม queue | รับประกันเฉพาะภายใน partition |

### กฎง่ายๆ ในการเลือก

- ถ้าโจทย์คือ **"กระจายงานให้ worker หลายตัวช่วยกันทำ แล้วงานนั้นควรถูกทำครั้งเดียว"** → RabbitMQ (หรือ queue ทั่วไป)
- ถ้าโจทย์คือ **"มีเหตุการณ์เกิดขึ้น แล้วมีหลายระบบอิสระต้องรับรู้เหตุการณ์นั้น และอาจต้องย้อนดูประวัติได้"** → Kafka
- ถ้า throughput สูงมากระดับ event สตรีมของทั้งบริษัท (clickstream, log aggregation, metrics) → Kafka แทบจะเป็นค่ามาตรฐานอุตสาหกรรมไปแล้ว

หลายองค์กรจริงๆ ใช้**ทั้งคู่**พร้อมกัน: RabbitMQ สำหรับ internal task queue ของแต่ละ service, Kafka เป็น "backbone" ของ event ที่ทั้งองค์กรต้องแชร์กัน

---

## 5. Consumer Group: กลไกสเกลการอ่านและการ replay

**Consumer Group** คือกลุ่มของ consumer ที่ประกาศตัวว่าอยู่ใน "group ID" เดียวกัน Kafka broker (ผ่าน **group coordinator**) จะแบ่ง partition ของ topic ให้แต่ละ consumer ในกลุ่มรับผิดชอบแบบ**ไม่ทับซ้อนกัน**:

```
Topic "orders" มี 3 partitions          Consumer Group "order-processors"

partition 0  ──────────────────────►    consumer-A
partition 1  ──────────────────────►    consumer-A
partition 2  ──────────────────────►    consumer-B
```

กฎสำคัญที่ต้องจำ:

1. **partition หนึ่งอันถูกอ่านโดย consumer ได้แค่ตัวเดียวในกลุ่มเดียวกัน ณ เวลาหนึ่ง** — นี่คือสิ่งที่ป้องกันไม่ให้ข้อความเดียวถูกประมวลผลซ้ำโดยสมาชิกคนละคนในกลุ่มเดียวกัน
2. **ถ้าจำนวน consumer มากกว่าจำนวน partition consumer ส่วนเกินจะว่างงาน** (ไม่มี partition ให้อ่าน) เพราะฉะนั้นจำนวน partition คือ**เพดานบนของการสเกลแนวนอน**ของการอ่าน (ต้องคิดเรื่องนี้ตั้งแต่ตอนออกแบบ topic — ดูหัวข้อ 12)
3. **consumer group คนละกลุ่มอ่าน topic เดียวกันได้อย่างอิสระต่อกัน** แต่ละกลุ่มมี offset ของตัวเอง — นี่คือสิ่งที่ทำให้ทีม billing, ทีม inventory, ทีม analytics อ่าน event "order created" ตัวเดียวกันได้พร้อมกันโดยไม่ยุ่งกัน (สร้าง group ID คนละชื่อ)
4. **Rebalance**: เมื่อ consumer ใหม่เข้ากลุ่มหรือ consumer เดิมหลุดออกจากกลุ่ม (crash, deploy ใหม่) broker จะจัด partition ใหม่ให้สมาชิกที่เหลือโดยอัตโนมัติ — ระหว่าง rebalance การอ่านจะหยุดชั่วคราว

### Replay: reset offset แล้วอ่านใหม่

เพราะ offset ถูกเก็บแยกจากตัวข้อมูล (เก็บเป็น per-group state) การ "ย้อนเวลา" ทำได้ง่ายมาก — แค่สร้าง consumer group ใหม่ที่ยังไม่เคย commit offset มาก่อน (หรือ reset offset ของกลุ่มเดิมกลับไปที่ offset เก่า/`FirstOffset`) แล้วมันจะอ่าน log ตั้งแต่ต้นใหม่ทั้งหมด นี่คือความสามารถที่ RabbitMQ ทำไม่ได้ในตัว (เพราะข้อความถูกลบไปแล้วหลัง ack)

---

## 6. Client library สำหรับ Go: `kafka-go` vs `confluent-kafka-go`

มี Go client หลักสองตัวที่ใช้กันในวงกว้าง:

### `github.com/segmentio/kafka-go` — pure Go, ไม่ต้องใช้ cgo

- เขียนด้วย Go ล้วน ไม่ต้องพึ่งไลบรารี C ภายนอก
- Cross-compile ง่าย (ตรงกับปรัชญา "binary เดียวจบ" ของ Go ที่เราคุยกันตั้งแต่ Part 001) — build แล้วได้ binary เดียว ไม่ต้องกังวลเรื่อง `CGO_ENABLED` หรือหา `.so`/`.dll` ของ librdkafka มาแปะด้วย
- API เขียนง่าย เข้าใจง่าย เหมาะกับการเรียนรู้และงานส่วนใหญ่
- Performance ดีในระดับที่งานทั่วไปเพียงพอ แต่ไม่ใช่ตัวเร็วที่สุดในตลาด
- **นี่คือ client ที่บทนี้จะใช้สาธิตทั้งหมด**

### `github.com/confluentinc/confluent-kafka-go` — wrap librdkafka ผ่าน cgo

- เป็น wrapper รอบ **librdkafka** (ไลบรารี C ที่ทีม Confluent ดูแล ใช้เป็น reference implementation ของ Kafka client มายาวนาน)
- **ต้องเปิด cgo** (`CGO_ENABLED=1`) และต้องมี librdkafka ติดตั้งอยู่บนเครื่อง build/runtime (หรือ build แบบ static ที่รวม librdkafka เข้าไปในไบนารี ซึ่งทำให้ build ซับซ้อนขึ้นมาก)
- แลกกับความซับซ้อนนี้ คือ **performance ที่สูงกว่า** และ feature ที่ตรงกับ Kafka protocol เวอร์ชันใหม่ๆ เร็วกว่า (librdkafka มักได้ feature ใหม่ก่อน pure-Go client)
- เหมาะกับงานที่ throughput สูงมากระดับ production ใหญ่จริงๆ และทีมที่ยอมรับความซับซ้อนของ cgo build ได้ (Docker multi-stage build ที่มี librdkafka-dev, cross-compile ที่ยุ่งยากขึ้น)

### กฎการเลือกอย่างย่อ

> เริ่มต้นด้วย `kafka-go` เสมอ เพราะ deploy ง่าย build ง่าย ตรงกับข้อดีของ Go มากที่สุด — สลับไป `confluent-kafka-go` เฉพาะเมื่อวัด (benchmark, Part 084) แล้วพบว่า throughput ไม่พอจริงๆ และทีมพร้อมรับภาระ operational ของ cgo

บทนี้เลือกใช้ `kafka-go` เพราะไม่ต้องพึ่ง cgo หรือไลบรารีระบบภายนอก ทำให้ทุกตัวอย่างในบทนี้ build และรันได้ตรงไปตรงมาด้วย `go build`/`go run` เหมือนทุก part ก่อนหน้า

---

## 7. เตรียมสภาพแวดล้อม (สิ่งที่รันจริงในบทนี้)

> **หมายเหตุความโปร่งใส**: โค้ดทุกชิ้นในบทนี้ (การสร้าง topic, producer, consumer, consumer group) **ถูกรันจริง**ในสภาพแวดล้อมที่เขียนบทนี้ โดยใช้ Docker รัน Kafka broker จริง แล้วรันโปรแกรม Go เชื่อมต่อเข้าไปจริง ผลลัพธ์ที่แสดงในหัวข้อ 10 คือผลลัพธ์ที่ได้จากการรันจริง ไม่ใช่ผลลัพธ์ที่แต่งขึ้น

รัน Kafka broker เดี่ยว (KRaft mode — Kafka เวอร์ชันใหม่ไม่ต้องพึ่ง ZooKeeper แล้ว) ด้วย image ทางการของ Apache Kafka:

```bash
docker run -d --name kafka-course -p 9092:9092 apache/kafka:3.7.0
```

รอสัก 5-10 วินาทีให้ broker start เสร็จ (ดู log ด้วย `docker logs kafka-course` จนเห็นบรรทัด `Kafka Server started`) แล้วสร้างโปรเจกต์ Go:

```bash
mkdir kafka-demo && cd kafka-demo
go mod init kafka-demo
go get github.com/segmentio/kafka-go@latest
```

> ถ้าไม่มี Docker หรือไม่สะดวกรัน Kafka เอง โค้ดในหัวข้อถัดไปยังอ่านและทำความเข้าใจได้ตามปกติ เพียงแต่รันจริงไม่ได้จนกว่าจะมี broker ให้เชื่อมต่อ — วิธีตั้งค่าที่ครบถ้วนกว่านี้ (multi-broker, replication) จะกลับมาพูดถึงอีกครั้งใน**ภาคที่ 9 (Docker Compose, Part 096)**

---

## 8. เขียน Producer ด้วย kafka-go

### สร้าง topic ก่อน

Kafka สร้าง topic อัตโนมัติได้ถ้า broker ตั้งค่า `auto.create.topics.enable=true` (ค่า default ของ image ที่ใช้ในบทนี้) แต่ในโปรดักชันควรสร้าง topic เองอย่างชัดเจนเพื่อควบคุมจำนวน partition และ replication factor:

```go
package main

import (
	"fmt"
	"log"

	"github.com/segmentio/kafka-go"
)

func createTopic(brokerAddr, topic string, numPartitions int) error {
	conn, err := kafka.Dial("tcp", brokerAddr)
	if err != nil {
		return fmt.Errorf("dial broker: %w", err)
	}
	defer conn.Close()

	// admin operation (create topic) ต้องคุยกับ controller broker เท่านั้น
	controller, err := conn.Controller()
	if err != nil {
		return fmt.Errorf("find controller: %w", err)
	}
	controllerConn, err := kafka.Dial("tcp", fmt.Sprintf("%s:%d", controller.Host, controller.Port))
	if err != nil {
		return fmt.Errorf("dial controller: %w", err)
	}
	defer controllerConn.Close()

	return controllerConn.CreateTopics(kafka.TopicConfig{
		Topic:             topic,
		NumPartitions:     numPartitions,
		ReplicationFactor: 1, // โปรดักชันจริงควรตั้งอย่างน้อย 3
	})
}

func main() {
	if err := createTopic("127.0.0.1:9092", "orders", 3); err != nil {
		log.Fatal(err)
	}
	fmt.Println("topic \"orders\" created with 3 partitions")
}
```

โค้ดชิ้นนี้ต้องคุยกับ **controller broker** โดยเฉพาะ (ไม่ใช่ broker ตัวไหนก็ได้) เพราะการสร้าง/ลบ topic เป็น metadata operation ที่ controller เป็นผู้จัดการ — รายละเอียดนี้ `kafka-go` จัดการให้ผ่าน `conn.Controller()`

### เขียนข้อความด้วย `kafka.Writer`

```go
package main

import (
	"context"
	"fmt"
	"log"
	"time"

	"github.com/segmentio/kafka-go"
)

func main() {
	writer := &kafka.Writer{
		Addr:         kafka.TCP("127.0.0.1:9092"),
		Topic:        "orders",
		Balancer:     &kafka.Hash{}, // ข้อความที่มี key เดียวกัน ไปลง partition เดียวกันเสมอ
		RequiredAcks: kafka.RequireAll,
	}
	defer writer.Close()

	ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
	defer cancel()

	msg := kafka.Message{
		Key:   []byte("user-42"),
		Value: []byte(`{"order_id": 1001, "user_id": 42, "amount": 599.00}`),
	}
	if err := writer.WriteMessages(ctx, msg); err != nil {
		log.Fatalf("write message: %v", err)
	}
	fmt.Println("message produced")
}
```

จุดที่ควรสังเกต:

- **`Balancer`** กำหนดว่าข้อความที่ไม่ระบุ partition ชัดเจนจะถูกส่งไปลง partition ไหน `&kafka.Hash{}` คือการ hash `Key` แล้ว map ไปยัง partition — ทำให้ข้อความที่มี key เดียวกัน (เช่น user คนเดียวกัน) ไปอยู่ partition เดียวกันเสมอ จึงรักษาลำดับของ event ของ user คนนั้นได้ ทางเลือกอื่นคือ `&kafka.RoundRobin{}` (กระจายเท่าๆ กันโดยไม่สนใจ key) หรือ `&kafka.LeastBytes{}` (ส่งไปยัง partition ที่มี buffer น้อยที่สุดตอนนั้น)
- **`RequiredAcks: kafka.RequireAll`** คือ durability guarantee ระดับสูงสุด — writer จะรอจนกว่า replica ทั้งหมดยืนยันว่าเขียนสำเร็จก่อนถือว่า `WriteMessages` เสร็จ (ตรงข้ามกับ `RequireNone` ที่ไม่รอ ack เลย เร็วแต่เสี่ยงข้อมูลหายถ้า broker ล่มก่อนเขียนจริง)
- `writer.WriteMessages` รับ `context.Context` (Part 032) เพื่อควบคุม timeout ของการเขียนแต่ละครั้ง — เป็น pattern เดียวกับที่เราใช้ตลอดหลักสูตรกับ database (Part 078) และ HTTP client (Part 046)

---

## 9. เขียน Consumer แบบธรรมดา (ไม่มี group)

ก่อนพูดถึง consumer group มาดู consumer แบบพื้นฐานที่สุดก่อน — อ่านจาก partition เดียว ไม่ใช้กลไก group ใดๆ:

```go
package main

import (
	"context"
	"fmt"
	"log"

	"github.com/segmentio/kafka-go"
)

func main() {
	reader := kafka.NewReader(kafka.ReaderConfig{
		Brokers:   []string{"127.0.0.1:9092"},
		Topic:     "orders",
		Partition: 0, // อ่านเฉพาะ partition 0 เท่านั้น ไม่มี group
		MinBytes:  1,
		MaxBytes:  10e6,
	})
	defer reader.Close()

	for {
		m, err := reader.ReadMessage(context.Background())
		if err != nil {
			log.Fatalf("read message: %v", err)
		}
		fmt.Printf("partition=%d offset=%d key=%s value=%s\n",
			m.Partition, m.Offset, m.Key, m.Value)
	}
}
```

โหมดนี้เหมาะกับงานที่ต้องการควบคุม partition assignment เอง (เช่น เขียน tool สำหรับ debug ดูข้อมูลดิบใน partition ใดๆ) แต่**ไม่สเกล**และ**ไม่มี consumer group coordination** ให้อัตโนมัติ — งานจริงส่วนใหญ่จึงใช้ consumer group แทน (หัวขัดถัดไป)

---

## 10. Consumer Group และ Partitioning: ตัวอย่างที่รันจริงครบวงจร

นี่คือตัวอย่างที่สาธิตทุกแนวคิดสำคัญของบทนี้ในโปรแกรมเดียว: สร้าง topic 3 partitions, เริ่ม consumer 2 ตัวใน **consumer group เดียวกัน** ก่อนที่จะ produce ข้อความ (เพื่อให้เห็นการแบ่ง partition ชัดเจน), แล้วผลิตข้อความ 24 ข้อความด้วย key 6 แบบ

```go
package main

import (
	"context"
	"fmt"
	"log"
	"sync"
	"time"

	"github.com/segmentio/kafka-go"
)

func main() {
	brokerAddr := "127.0.0.1:9092"
	topic := "orders-demo"

	// 1. สร้าง topic ที่มี 3 partitions
	conn, err := kafka.Dial("tcp", brokerAddr)
	if err != nil {
		log.Fatalf("dial: %v", err)
	}
	controller, err := conn.Controller()
	if err != nil {
		log.Fatalf("controller: %v", err)
	}
	controllerConn, err := kafka.Dial("tcp", fmt.Sprintf("%s:%d", controller.Host, controller.Port))
	if err != nil {
		log.Fatalf("dial controller: %v", err)
	}
	if err := controllerConn.CreateTopics(kafka.TopicConfig{
		Topic:             topic,
		NumPartitions:     3,
		ReplicationFactor: 1,
	}); err != nil {
		log.Fatalf("create topic: %v", err)
	}
	controllerConn.Close()
	conn.Close()
	fmt.Println("topic created with 3 partitions")

	ctx, cancel := context.WithTimeout(context.Background(), 60*time.Second)
	defer cancel()

	// 2. เริ่ม consumer 2 ตัวใน consumer group เดียวกัน "order-processors"
	//    Kafka จะแบ่ง 3 partitions ให้สมาชิกทั้งสองอัตโนมัติ
	groupID := "order-processors"
	const totalMessages = 24

	var (
		mu             sync.Mutex
		consumed       = map[string]int{}
		partitionsSeen = map[string]map[int]bool{}
		total          int
	)

	var wg sync.WaitGroup
	runConsumer := func(name string) {
		defer wg.Done()
		r := kafka.NewReader(kafka.ReaderConfig{
			Brokers:  []string{brokerAddr},
			Topic:    topic,
			GroupID:  groupID, // ← นี่คือสิ่งที่ทำให้มันเป็น "consumer group"
			MinBytes: 1,
			MaxBytes: 10e6,
		})
		defer r.Close()

		rctx, rcancel := context.WithTimeout(context.Background(), 30*time.Second)
		defer rcancel()

		seen := map[int]bool{}
		for {
			mu.Lock()
			done := total >= totalMessages
			mu.Unlock()
			if done {
				break
			}
			m, err := r.ReadMessage(rctx)
			if err != nil {
				break // deadline หมดเวลาแปลว่าไม่มีข้อความมาเพิ่มแล้ว
			}
			seen[m.Partition] = true
			mu.Lock()
			consumed[name]++
			total++
			mu.Unlock()
		}
		mu.Lock()
		partitionsSeen[name] = seen
		mu.Unlock()
	}

	wg.Add(2)
	go runConsumer("consumer-A")
	go runConsumer("consumer-B")

	// รอให้ทั้งคู่ join group และได้รับ partition assignment ก่อน
	time.Sleep(5 * time.Second)

	// 3. produce 24 ข้อความ กระจาย key ให้ครบทั้ง 3 partitions
	writer := &kafka.Writer{
		Addr:         kafka.TCP(brokerAddr),
		Topic:        topic,
		Balancer:     &kafka.Hash{},
		RequiredAcks: kafka.RequireAll,
	}
	defer writer.Close()

	keys := []string{"user-1", "user-3", "user-7", "user-2", "user-5", "user-9"}
	for i := 0; i < totalMessages; i++ {
		k := keys[i%len(keys)]
		msg := kafka.Message{
			Key:   []byte(k),
			Value: []byte(fmt.Sprintf("order-event-%d", i)),
		}
		if err := writer.WriteMessages(ctx, msg); err != nil {
			log.Fatalf("write message %d: %v", i, err)
		}
	}
	fmt.Printf("produced %d messages across keys %v\n", totalMessages, keys)

	wg.Wait()

	fmt.Println("---- consumer group result ----")
	for _, name := range []string{"consumer-A", "consumer-B"} {
		partitions := make([]int, 0)
		for p := range partitionsSeen[name] {
			partitions = append(partitions, p)
		}
		fmt.Printf("%s consumed %d messages, from partitions %v\n", name, consumed[name], partitions)
	}
	fmt.Printf("total consumed by group %q: %d/%d\n", groupID, total, totalMessages)
}
```

### ผลลัพธ์จากการรันจริง

```
topic created with 3 partitions
produced 24 messages across keys [user-1 user-3 user-7 user-2 user-5 user-9]
---- consumer group result ----
consumer-A consumed 16 messages, from partitions [0 2]
consumer-B consumed 8 messages, from partitions [1]
total consumed by group "order-processors": 24/24
```

สังเกตสิ่งที่เกิดขึ้นจริง:

- Broker แบ่ง 3 partitions ให้ `consumer-A` รับผิดชอบ 2 partitions (0, 2) และ `consumer-B` รับผิดชอบ 1 partition (1) — การแบ่งไม่จำเป็นต้องเท่ากันเป๊ะเมื่อจำนวน partition หารด้วยจำนวน consumer ไม่ลงตัว (3 หาร 2 ไม่ลงตัว)
- รวมกันแล้วครบ **24/24 ข้อความ ไม่มีข้อความไหนถูกอ่านซ้ำโดยทั้งสอง consumer** — นี่คือ guarantee หลักของ consumer group ภายในกลุ่มเดียวกัน
- เพราะ key ทั้ง 6 ตัวถูกเลือกให้ hash กระจายไปทั้ง 3 partition (ตรวจสอบได้ด้วยการเรียก `(&kafka.Hash{}).Balance(...)` ตรงๆ) ผลลัพธ์จำนวนข้อความต่อ partition จึงไม่เท่ากันเป๊ะ 8/8/8 — ในโปรดักชันจริงเมื่อ key มีความหลากหลายมากพอ (เช่น user ID หลักพัน) การกระจายจะสม่ำเสมอกว่านี้มาก

### ทำไมต้อง sleep 5 วินาทีก่อน produce

ถ้า produce ข้อความทั้งหมดไปก่อนที่ consumer จะ join group เสร็จ สิ่งที่จะเกิดขึ้นคือ consumer ตัวแรกที่ join สำเร็จก่อนอาจถูก assign partition ทั้งหมดชั่วคราว และอ่านข้อความที่ผลิตไว้ก่อนหน้าไปจนหมดก่อนที่ consumer ตัวที่สองจะ join ทัน ทำให้เห็นภาพ "แบ่งงานกัน" ไม่ชัดเจน (ตัวหนึ่งได้ทุกอย่าง อีกตัวไม่ได้อะไรเลย) — โค้ดข้างต้นจึงเริ่ม consumer ก่อน แล้วรอให้ **join + rebalance** เสร็จก่อนค่อย produce เพื่อให้เห็นการแบ่ง partition ชัดเจนตามที่ควรจะเป็น ในโปรดักชันจริงลำดับการ start ไม่ค่อยสำคัญขนาดนี้ เพราะ Kafka รับประกันว่าข้อความที่ค้างอยู่ใน topic (ยังไม่หมด retention) จะถูกอ่านในที่สุดไม่ว่า consumer จะ join ก่อนหรือหลัง เพียงแต่ตัวอย่างเพื่อการสอนนี้ต้องการให้เห็นการแบ่งงานชัดๆ ในรันเดียว

---

## 11. Replay-ability: ความสามารถที่ RabbitMQ ทำได้ยากกว่า

เพราะ Kafka ไม่ลบข้อความหลังอ่าน (จนกว่าจะหมด retention) การ "ย้อนอ่านใหม่" ทำได้ง่ายมาก — สร้าง group ใหม่ที่ตั้ง `StartOffset: kafka.FirstOffset` แล้วมันจะอ่านตั้งแต่ข้อความแรกสุดที่ยังอยู่ใน log:

```go
reader := kafka.NewReader(kafka.ReaderConfig{
	Brokers:     []string{"127.0.0.1:9092"},
	Topic:       "orders",
	GroupID:     "analytics-backfill-2024", // group ID ใหม่ที่ไม่เคย commit offset มาก่อน
	StartOffset: kafka.FirstOffset,          // อ่านตั้งแต่ข้อความแรกสุดที่ยังไม่ถูก retention ลบ
})
```

สถานการณ์ที่ใช้บ่อยในงานจริง:

- ทีม analytics เพิ่งสร้าง pipeline ใหม่ อยากได้ event ย้อนหลัง 7 วันเพื่อ backfill ข้อมูล → สร้าง consumer group ใหม่ อ่านจากต้น
- เกิด bug ใน consumer เดิม ทำให้ประมวลผลข้อมูลผิดไป 2 ชั่วโมงที่ผ่านมา → reset offset ของ group นั้นย้อนกลับไป 2 ชั่วโมงก่อน แล้วให้มันประมวลผลใหม่ (ข้อความที่ RabbitMQ ทำแบบนี้ไม่ได้เพราะข้อความหายไปแล้วตั้งแต่ ack ครั้งแรก)
- ต้องการรัน integration test ที่ประมวลผล event ชุดเดิมซ้ำหลายรอบ → ใช้ group ID คนละชื่อทุกครั้งที่รันเทสต์

นี่คือเหตุผลสำคัญที่ทำให้ Kafka ถูกเลือกเป็น **source of truth ของ event** ในสถาปัตยกรรมแบบ event sourcing/CQRS มากกว่า RabbitMQ

---

## 12. แนวทางออกแบบ topic/partition ในโปรดักชัน

ข้อควรพิจารณาเมื่อออกแบบ topic จริง (ไม่ใช่แค่ในตัวอย่างสาธิต):

1. **จำนวน partition ต้อง "คิดล่วงหน้า"** — เพิ่ม partition ทีหลังทำได้ แต่**ลดไม่ได้** และการเพิ่ม partition กลางทางจะทำให้ key เดิมที่เคย hash ไป partition หนึ่ง อาจ hash ไปอีก partition หนึ่งแทน (ทำลาย ordering guarantee ที่เคยมี) ควรประเมิน throughput สูงสุดที่ต้องการและจำนวน consumer สูงสุดที่จะสเกลไปถึง แล้วตั้ง partition ให้เผื่อไว้ตั้งแต่ต้น
2. **เลือก key ให้เหมาะกับ ordering ที่ต้องการจริงๆ** — ถ้าไม่มี requirement เรื่องลำดับเลย ปล่อยให้ไม่มี key (round-robin) จะกระจายโหลดได้สม่ำเสมอที่สุด ถ้าต้องการลำดับต่อ entity (เช่น user, order) ให้ใช้ entity ID เป็น key เสมอ
3. **ReplicationFactor อย่างน้อย 3 ในโปรดักชัน** เพื่อทนต่อ broker ล่มได้อย่างน้อย 1-2 ตัวโดยไม่เสียข้อมูล (ตัวอย่างในบทนี้ใช้ 1 เพราะรันบน broker เดียวเพื่อความง่ายในการสาธิตเท่านั้น)
4. **ตั้ง retention ให้เหมาะกับงาน** — event ที่ต้องใช้ replay/audit นานอาจตั้งเป็นสัปดาห์หรือถาวร (`log.retention.ms=-1` หรือใช้ topic แบบ **compacted** ที่เก็บเฉพาะค่าล่าสุดต่อ key แทนการลบตามเวลา) ส่วน event ที่ใช้ชั่วคราว (เช่น cache invalidation signal) ตั้งสั้นได้เพื่อประหยัด disk
5. **แยก topic ตามโดเมนของ event ไม่ใช่ตามทีม** — เช่น `order.created`, `order.cancelled` แยกกัน แทนที่จะยัดทุกอย่างของทีม order รวมกันเป็น topic เดียว ทำให้ consumer เลือก subscribe เฉพาะ event ที่สนใจได้ (concept นี้ต่อยอดจากหลักการออกแบบ event ที่วางไว้ใน Part 088)

---

## สรุปสิ่งที่ได้เรียนในบทนี้

- Kafka ไม่ใช่ queue แบบ RabbitMQ แต่เป็น **distributed append-only log** — ข้อความไม่หายหลังอ่าน ผู้ที่จำ offset ของตัวเองคือ consumer ไม่ใช่ broker
- โครงสร้างหลัก: **topic** แบ่งเป็นหลาย **partition**, แต่ละ partition มี **offset** เรียงลำดับของตัวเอง, ordering รับประกันได้เฉพาะภายใน partition เดียว
- Kafka เหมาะกับ **event streaming ที่หลายระบบอิสระต้องรับรู้เหตุการณ์เดียวกัน** และต้องการ **replay-ability** ส่วน RabbitMQ เหมาะกับ **task queue** ที่งานควรถูกทำครั้งเดียวจบ
- **Consumer group** คือกลไกสเกลการอ่าน — Kafka แบ่ง partition ให้สมาชิกในกลุ่มโดยอัตโนมัติ ไม่มี partition ไหนถูกอ่านซ้ำโดยสมาชิกสองคนในกลุ่มเดียวกัน
- `github.com/segmentio/kafka-go` คือ client แบบ pure Go ไม่ต้องพึ่ง cgo เหมาะเป็นตัวเลือกแรกเสมอ ส่วน `confluent-kafka-go` (wrap librdkafka ผ่าน cgo) ให้ performance สูงกว่าแต่ deploy ซับซ้อนกว่า
- โค้ดในบทนี้ (สร้าง topic, producer, consumer, consumer group แบ่ง partition) **รันจริงกับ Kafka broker จริงผ่าน Docker** และผลลัพธ์ที่แสดงคือผลลัพธ์จากการรันจริง
- การออกแบบ topic/partition ในโปรดักชันต้องคิดล่วงหน้าเรื่องจำนวน partition, การเลือก key, replication factor และ retention

## แบบฝึกหัดท้ายบท

1. รัน Kafka broker ด้วย Docker ตามหัวข้อ 7 แล้วรันตัวอย่างในหัวข้อ 10 ด้วยตัวเอง ลองเปลี่ยนจำนวน partition จาก 3 เป็น 6 และเพิ่ม consumer ในกลุ่มเป็น 3 ตัว สังเกตว่า partition ถูกแบ่งอย่างไร
2. เขียนโปรแกรมที่สร้าง consumer group ใหม่ (group ID ที่ไม่เคยใช้มาก่อน) แล้วตั้ง `StartOffset: kafka.FirstOffset` เพื่ออ่าน topic เดิมตั้งแต่ต้น เทียบกับการอ่านด้วย group ID เดิมที่เคย commit offset ไปแล้ว ผลต่างคืออะไร
3. ทดลองสร้าง producer ที่ส่งข้อความโดยไม่ใส่ `Key` เลย (key เป็น `nil`) แล้วสังเกตว่า `&kafka.RoundRobin{}` กับ `&kafka.Hash{}` กระจายข้อความไปยัง partition ต่างกันอย่างไรเมื่อไม่มี key
4. อธิบายด้วยคำพูดของตัวเอง (ไม่ต้องเขียนโค้ด): ทำไมการเพิ่มจำนวน partition ของ topic ที่มีข้อมูลอยู่แล้วถึงเสี่ยงทำลาย ordering guarantee ที่เคยมีอยู่
5. ออกแบบ (เขียนเป็นเอกสารสั้นๆ ไม่ต้องเขียนโค้ด) ว่าถ้าต้องเลือกใช้ RabbitMQ หรือ Kafka สำหรับระบบ "แจ้งเตือนเมื่อสินค้าใกล้หมดสต๊อก" ที่ทีม inventory, ทีม purchasing, และทีม analytics ต้องรับรู้เหตุการณ์เดียวกันทั้งหมด จะเลือกอะไร เพราะอะไร
6. (ขั้นสูง) ลองติดตั้ง `confluent-kafka-go` ในเครื่องที่มี librdkafka (หรืออ่าน README ของโปรเจกต์) แล้วเปรียบเทียบขั้นตอนการติดตั้ง/build กับ `kafka-go` — สรุปว่าความซับซ้อนที่เพิ่มขึ้นคุ้มกับ performance ที่ได้หรือไม่ในบริบทงานที่คุณคุ้นเคย

---

**ต่อไป**: [Part 093 — Service Discovery และ API Gateway](./093-service-discovery-and-api-gateway.md)
