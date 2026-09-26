# Part 088: หลักการ Microservices Architecture

> ภาคที่ 8: Microservices, gRPC, Message Queue — ตอนที่ 1 จาก 7 (Part 88–94)

## สารบัญของบทนี้

1. Monolith คืออะไร และทำไมโปรเจกต์ส่วนใหญ่เริ่มต้นที่นี่
2. Microservices คืออะไรกันแน่
3. Monolith vs Microservices: เปรียบเทียบแบบตรงไปตรงมา
4. "Monolith First": ทำไมผู้เชี่ยวชาญตัวจริงแนะนำให้เริ่มจาก Monolith
5. Service Boundaries: จะแบ่ง Service อย่างไรไม่ให้พัง (พรีวิว Domain-Driven Design)
6. การสื่อสารระหว่าง Service: Synchronous vs Asynchronous
7. ปัญหาเฉพาะของ Distributed Systems ที่ Monolith ไม่เคยเจอ
8. API Gateway: ประตูเดียวสู่ระบบทั้งหมด (พรีวิว Part 093)
9. ทำไม Go เหมาะกับ Microservices เป็นพิเศษ
10. แผนการเดินทางของภาคที่ 8
11. สรุปสิ่งที่ได้เรียนในบทนี้
12. แบบฝึกหัดท้ายบท

---

## 1. Monolith คืออะไร และทำไมโปรเจกต์ส่วนใหญ่เริ่มต้นที่นี่

ตลอด 87 part ที่ผ่านมาในหลักสูตรนี้ ตั้งแต่ Part 047 (HTTP Server) ไปจนถึง Part 078 (Transactions และ Connection Pooling) เราสร้างแอปพลิเคชันแบบ **Monolith** มาโดยตลอด โดยไม่รู้ตัว

**Monolith (โมโนลิธ)** คือสถาปัตยกรรมที่รวมทุก business logic ของระบบไว้ใน **codebase เดียว, build เดียว, deploy เป็นหน่วยเดียว** ตัวอย่างเช่น API server ตัวหนึ่งที่มีทั้ง endpoint สำหรับ user, order, payment, notification อยู่ใน binary เดียวกัน เชื่อมต่อฐานข้อมูลเดียวกัน และ scale ด้วยการรัน instance ของ binary นั้นซ้ำๆ หลายตัว

```
┌─────────────────────────────────────┐
│         Monolith Application          │
│                                         │
│  ┌──────┐ ┌───────┐ ┌───────┐ ┌─────┐ │
│  │ User │ │ Order │ │Payment│ │Notif│ │
│  └──────┘ └───────┘ └───────┘ └─────┘ │
│              │                         │
│         ┌────▼─────┐                   │
│         │ Database │                   │
│         └──────────┘                   │
└─────────────────────────────────────┘
```

Monolith ไม่ใช่ของล้าสมัยหรือ "ทำผิด" — บริษัทระดับโลกจำนวนมาก (Shopify, Basecamp, StackOverflow ในยุคที่ทำ traffic มหาศาลด้วยเซิร์ฟเวอร์ไม่กี่เครื่อง) ยังคงรัน Monolith เป็นแกนหลักของระบบจนถึงทุกวันนี้ ข้อดีของมันคือ:

- **พัฒนาเร็วในช่วงเริ่มต้น**: ไม่ต้องคิดเรื่อง network, serialization, service discovery — เรียกฟังก์ชันตรงๆ ในโปรเซสเดียวกัน
- **Debug ง่าย**: stack trace เดียว, log เดียว, ไม่ต้องไล่ตามข้าม service
- **Transaction ง่าย**: ใช้ database transaction ปกติ (ตามที่เรียนใน Part 078) ครอบคลุมทุก business logic ได้ในทีเดียว ไม่มีปัญหาเรื่อง distributed transaction
- **Deploy ง่าย**: build ครั้งเดียว, deploy ไฟล์เดียว (หรือ binary เดียวสำหรับ Go)
- **Performance ดีกว่าในหลายกรณี**: เรียกฟังก์ชันในโปรเซสเดียวกันเร็วกว่าเรียกผ่าน network มาก (นาโนวินาที เทียบกับมิลลิวินาที)

จุดอ่อนของ Monolith จะเริ่มปรากฏเมื่อ **ทีมและระบบโตขึ้น**:

- Codebase ใหญ่ขึ้นเรื่อยๆ จนหลายทีมแตะไฟล์เดียวกัน conflict กันบ่อย
- ต้อง deploy ทั้งระบบทุกครั้งแม้แก้แค่ฟีเจอร์เล็กๆ ส่วนเดียว
- Scale ต้อง scale ทั้งแอป แม้จะมีแค่บาง module ที่ต้องการ resource เยอะ (เช่น image processing ที่กิน CPU สูง แต่ต้อง scale ทั้ง monolith ตามไปด้วย)
- เทคโนโลยีถูกล็อกทั้งระบบ เปลี่ยนภาษา/framework ของ module เดียวทำไม่ได้ง่ายๆ

---

## 2. Microservices คืออะไรกันแน่

**Microservices** คือสถาปัตยกรรมที่แตกระบบใหญ่ออกเป็น **service ขนาดเล็กหลายตัว** แต่ละตัวมี:

- **Codebase และ deploy pipeline เป็นของตัวเอง** — deploy แยกอิสระจาก service อื่น
- **Database เป็นของตัวเอง** (หรืออย่างน้อย schema/ownership เป็นของตัวเอง) — ไม่มี service อื่นเข้ามาแก้ข้อมูลตรงๆ
- **รับผิดชอบ business capability หนึ่งอย่างชัดเจน** เช่น `user-service`, `order-service`, `payment-service`, `notification-service`
- **สื่อสารกับ service อื่นผ่าน network** ด้วย API ที่ชัดเจน (REST, gRPC, message queue)

```
┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐
│   User    │    │  Order   │    │ Payment  │    │  Notif   │
│  Service  │◄──►│ Service  │◄──►│ Service  │◄──►│ Service  │
└─────┬────┘    └─────┬────┘    └─────┬────┘    └─────┬────┘
      │               │               │               │
  ┌───▼───┐       ┌───▼───┐       ┌───▼───┐       ┌───▼───┐
  │  DB   │       │  DB   │       │  DB   │       │  DB   │
  └───────┘       └───────┘       └───────┘       └───────┘
```

สังเกตว่า **นี่ไม่ใช่แค่การแยกโฟลเดอร์ในโปรเจกต์เดียว** แบบที่เราทำใน `internal/handler`, `internal/service`, `internal/repository` (ตามโครงสร้างที่แนะนำใน Part 001) — นั่นยังเป็น Monolith ที่จัดระเบียบโค้ดดี (Modular Monolith) เท่านั้น Microservices ที่แท้จริงคือแต่ละกล่องในภาพด้านบน **รันเป็นโปรเซสแยกกัน อาจอยู่คนละเครื่อง คนละภาษาโปรแกรมมิ่งก็ได้** และสื่อสารกันผ่านเครือข่ายเท่านั้น

### สิ่งที่ Microservices แก้ได้จริง

ประเด็นสำคัญที่มักถูกเข้าใจผิดคือ Microservices **ไม่ได้ทำให้ระบบเร็วขึ้นหรือ performance ดีขึ้น** โดยตัวมันเอง (ในทางตรงกันข้าม มันช้าลงในหลายจุดเพราะต้องผ่าน network) สิ่งที่ Microservices แก้ได้จริงคือ **ปัญหาระดับองค์กร (organizational scaling)**:

- **Conway's Law**: "องค์กรจะออกแบบระบบที่มีโครงสร้างสื่อสารเหมือนโครงสร้างองค์กรของตัวเอง" — ถ้ามี 10 ทีมทำงานคนละส่วน การมี 10 service ที่แต่ละทีมเป็นเจ้าของเต็มตัว ทำให้ทีมทำงานอิสระจากกันได้จริง ไม่ต้องรอ merge/review ข้ามทีม
- **Independent deployability**: ทีม payment แก้บั๊กแล้ว deploy ได้ทันทีโดยไม่ต้องรอทีม notification เสร็จงานของตัวเองก่อน
- **Independent scaling**: scale เฉพาะ service ที่ต้องการจริงๆ เช่น `order-service` รับ traffic สูงช่วงเทศกาล ก็เพิ่ม instance เฉพาะตัวนั้น โดยไม่ต้อง scale `notification-service` ตามไปด้วย
- **Technology diversity**: ทีมหนึ่งอยากใช้ Go เพราะต้องการ throughput สูง อีกทีมอยากใช้ Python เพราะทำ ML model ก็ทำได้ในสถาปัตยกรรมเดียวกัน

---

## 3. Monolith vs Microservices: เปรียบเทียบแบบตรงไปตรงมา

หลายบทความออนไลน์นำเสนอ Microservices เป็น "ทางออกที่ดีกว่าเสมอ" ซึ่ง**ไม่เป็นความจริง** ทั้งสองสถาปัตยกรรมมี tradeoff ที่ชัดเจน:

| ประเด็น | Monolith | Microservices |
|---|---|---|
| ความเร็วในการเริ่มพัฒนา | เร็ว — setup โปรเจกต์เดียวจบ | ช้า — ต้อง setup infra, network, CI/CD หลายชุด |
| Deploy | ง่าย deploy ก้อนเดียว | ซับซ้อน ต้อง orchestrate หลาย service |
| Debugging | stack trace เดียว log เดียว หาง่าย | ต้องไล่ log ข้าม service (ต้องมี distributed tracing) |
| Transaction | ACID transaction ปกติ (Part 078) | ต้องใช้ Saga pattern หรือ eventual consistency |
| Scaling | scale ทั้งแอปพร้อมกัน | scale แยกตาม service ที่ต้องการจริง |
| Team autonomy | ทีมต้อง coordinate กันเยอะในโค้ดเดียวกัน | ทีมทำงานอิสระ deploy เองได้ |
| Failure mode | แอปทั้งก้อนล่มพร้อมกัน | บาง service ล่มได้ ส่วนอื่นยังทำงานต่อ (ถ้าออกแบบดี) แต่ก็เพิ่ม "partial failure" ที่ Monolith ไม่มี |
| Network overhead | ไม่มี (เรียกฟังก์ชันในโปรเซสเดียวกัน) | มีเสมอ (latency, serialization, retries) |
| ความซับซ้อนของ Operations | ต่ำ | สูงมาก ต้องมี service discovery, API gateway, monitoring, distributed logging |
| เหมาะกับทีมขนาด | เล็ก-กลาง (1 ทีมถึงไม่กี่ทีม) | ใหญ่ (หลายทีมอิสระ) |

ข้อความสำคัญที่สุดของบทนี้คือ: **Microservices ไม่ใช่ของฟรี** มันแลกความง่ายในการพัฒนาและ debug กับความสามารถในการ scale องค์กรและ deploy อิสระ ถ้าทีมของคุณมีแค่ 3-5 คน การแตกเป็น 10 microservices จะทำให้ทีมช้าลงอย่างมาก เพราะต้องแบกรับ operational overhead มหาศาลโดยไม่ได้ประโยชน์จากการแบ่งทีมที่ไม่มีอยู่จริง

---

## 4. "Monolith First": ทำไมผู้เชี่ยวชาญตัวจริงแนะนำให้เริ่มจาก Monolith

Martin Fowler (หนึ่งในผู้เชี่ยวชาญด้าน software architecture ที่มีอิทธิพลที่สุด) เขียนบทความชื่อดังชื่อ **"MonolithFirst"** เสนอแนวคิดที่สรุปได้ว่า:

> "แทบทุกเรื่องราวความสำเร็จของ Microservices เริ่มต้นจาก Monolith ที่โตเกินขนาดแล้วค่อยแตกออก ไม่มีเรื่องราวความสำเร็จที่เริ่มจาก Microservices ตั้งแต่วันแรก"

เหตุผลเชิงปฏิบัติ:

1. **คุณยังไม่รู้ service boundary ที่ถูกต้องตั้งแต่วันแรก** — ตอนเริ่มโปรเจกต์ใหม่ ความเข้าใจโดเมนธุรกิจยังไม่นิ่ง การแบ่ง service ผิดตั้งแต่แรกแล้วต้องมาสลับข้อมูลข้าม service ทีหลัง แก้ยากกว่าการ refactor ฟังก์ชันในโค้ดเดียวกันมาก
2. **ต้นทุนการแก้ไข boundary ผิดใน Monolith ต่ำกว่ามาก** — ย้ายโค้ดข้าม package/refactor ภายในโปรเจกต์เดียวใช้เวลาเป็นชั่วโมง แต่การย้าย boundary ข้าม microservices ที่ deploy แยกกันแล้วอาจใช้เวลาเป็นสัปดาห์หรือเดือน
3. **Operational overhead ของ Microservices สูงตั้งแต่ service แรก** — คุณต้องมี CI/CD หลายชุด, service discovery, monitoring แบบ distributed, API gateway ตั้งแต่ต้น ทั้งที่ยังไม่มี traffic หรือทีมมากพอจะได้ประโยชน์จากมัน

แนวทางที่แนะนำในทางปฏิบัติคือเขียน **Modular Monolith** ก่อน: จัดโครงสร้างโค้ดให้แบ่งเป็น module ที่ชัดเจนด้วย `internal/` package (ตามที่เรียนใน Part 001) โดยแต่ละ module สื่อสารกันผ่าน interface ที่ชัดเจน ไม่ import ข้าม module มั่วซั่ว เมื่อถึงจุดที่:

- ทีมโตจนต้อง deploy แยกกันจริงๆ
- module ใด module หนึ่งต้องการ scale ต่างจากส่วนอื่นอย่างชัดเจน
- boundary ของ domain นิ่งแล้ว ไม่เปลี่ยนบ่อย

จึงค่อยแตก module นั้นออกเป็น microservice จริง — ซึ่งถ้าโค้ดถูกจัดโครงสร้างดีตั้งแต่ต้น (module มี interface ชัดเจน ไม่ผูกกับ module อื่นแบบสุ่มสี่สุ่มห้า) การแตกออกจะทำได้ค่อนข้างราบรื่น

หลักสูตรนี้จึงสอน Microservices ในภาคที่ 8 **หลังจาก** ที่คุณได้เรียนรู้การสร้าง Monolith ที่ดีมาแล้วครบถ้วน (Web Development ภาคที่ 5, Database ภาคที่ 6) — เพราะ Microservices ที่ดีคือ Monolith ที่ถูกแตกออกอย่างมีเหตุผล ไม่ใช่การเริ่มต้นด้วยความซับซ้อนตั้งแต่บรรทัดแรก

### ตัวอย่าง: Modular Monolith ใน Go ที่พร้อมแตกเป็น Microservice ในอนาคต

หน้าตาของ Modular Monolith ที่ดีคือแต่ละ module สื่อสารกันผ่าน **interface** เท่านั้น ไม่แตะ struct หรือ database ของ module อื่นตรงๆ ตัวอย่างโครงสร้างสำหรับระบบ order/inventory:

```go
// internal/inventory/service.go
package inventory

type Service interface {
	ReserveStock(ctx context.Context, productID string, qty int) error
}

type service struct {
	repo Repository // เข้าถึง database ของ inventory เท่านั้น
}

func (s *service) ReserveStock(ctx context.Context, productID string, qty int) error {
	// business logic ของ inventory ทั้งหมดอยู่ที่นี่
	return s.repo.DecrementStock(ctx, productID, qty)
}
```

```go
// internal/order/service.go
package order

import "myapp/internal/inventory"

type Service struct {
	inventory inventory.Service // เรียกผ่าน interface เท่านั้น ไม่ import internal ของ inventory ตรงๆ
}

func (s *Service) PlaceOrder(ctx context.Context, productID string, qty int) error {
	if err := s.inventory.ReserveStock(ctx, productID, qty); err != nil {
		return fmt.Errorf("reserve stock: %w", err)
	}
	// สร้าง order record ต่อ...
	return nil
}
```

สังเกตว่า `order` เรียก `inventory` ผ่าน **interface** (`inventory.Service`) ไม่ใช่ผ่าน struct หรือ SQL ตรงๆ ของ inventory — วันหนึ่งถ้า `inventory` ต้องแยกเป็น microservice จริง สิ่งที่ต้องเปลี่ยนคือแค่ **implementation ของ interface** จาก `service{repo: ...}` (เรียกฟังก์ชันในโปรเซสเดียวกัน) เป็น `grpcClient{conn: ...}` (เรียกผ่าน gRPC ตามที่จะเรียนใน Part 089) โดยที่โค้ดฝั่ง `order` แทบไม่ต้องแก้อะไรเลย เพราะมันคุยกับ interface อยู่แล้วตั้งแต่ต้น — นี่คือเหตุผลที่การออกแบบ boundary ด้วย interface ตั้งแต่ตอนเป็น Monolith ช่วยให้การแตกเป็น Microservices ในอนาคต (ถ้าจำเป็นจริงๆ) ทำได้ราบรื่นกว่ามาก

### Strangler Fig Pattern: แตก Monolith ทีละส่วนแบบไม่หยุดระบบ

เมื่อถึงเวลาต้องแตก Monolith จริงๆ แนวทางที่ปลอดภัยที่สุดไม่ใช่การเขียนระบบใหม่ทั้งหมดตั้งแต่ต้น (**"Big Bang Rewrite"** ซึ่งมีประวัติล้มเหลวสูงมากในวงการ) แต่คือ **Strangler Fig Pattern** ตั้งชื่อตามต้นไม้ชนิดหนึ่งที่ขึ้นพันรอบต้นไม้เดิมจนแทนที่ทั้งหมดในที่สุด:

1. วาง API Gateway (หัวข้อ 8) ไว้หน้า Monolith เดิม
2. เลือก module หนึ่งที่ boundary ชัดเจนที่สุด (เช่น `notification`) แล้วสร้างเป็น microservice แยกต่างหาก
3. ปรับ Gateway ให้ route request ของ module นั้นไปที่ service ใหม่ ส่วน request อื่นยังไปที่ Monolith เดิมตามปกติ
4. ทำซ้ำทีละ module จนกว่า Monolith เดิมจะเหลือน้อยหรือหมดไป

ข้อดีคือระบบใช้งานได้ตลอดเวลาระหว่างการย้าย ไม่ต้อง "หยุดโลก" เพื่อ migrate ทั้งหมดพร้อมกัน และถ้า module ไหนย้ายแล้วมีปัญหา สามารถ route กลับไปที่ Monolith เดิมชั่วคราวได้

---

## 5. Service Boundaries: จะแบ่ง Service อย่างไรไม่ให้พัง

คำถามที่ยากที่สุดของ Microservices ไม่ใช่เรื่องเทคนิค (gRPC, message queue) แต่คือ **"จะตัดแบ่ง service ตรงไหน"** ตัดผิดจุดจะทำให้ service ต้องเรียกกันไปมาถี่มาก (chatty communication) จนได้ผลเสียมากกว่าผลดี

หลักการที่ใช้กันแพร่หลายที่สุดคือ **Domain-Driven Design (DDD)** ซึ่งเราจะเรียนแบบเจาะลึกใน **Part 101** แต่แนวคิดหลักที่ควรรู้ไว้ตั้งแต่ตอนนี้คือ:

- **Bounded Context**: แบ่งระบบตาม "ขอบเขตความหมาย" ของแต่ละโดเมนธุรกิจ เช่นคำว่า "Product" ใน context ของ `catalog-service` หมายถึงข้อมูลสินค้า (ชื่อ, รูป, คำอธิบาย) แต่คำว่า "Product" ใน context ของ `inventory-service` หมายถึงจำนวนสต็อกคงเหลือ — สอง context นี้ไม่ควรพยายามใช้ struct เดียวกันร่วมกัน แต่ควรแยกเป็นคนละ service ที่มีมุมมองข้อมูลของตัวเอง
- **High cohesion, low coupling**: ฟังก์ชันที่เปลี่ยนแปลงพร้อมกันบ่อยๆ ควรอยู่ใน service เดียวกัน (cohesion สูง) ส่วนบริการที่ต้องคุยกันน้อยที่สุดเท่าที่จะทำได้ควรอยู่คนละ service (coupling ต่ำ)
- **แบ่งตาม business capability ไม่ใช่ตาม technical layer**: ผิดถ้าจะแบ่งเป็น `database-service`, `validation-service`, `logic-service` (นี่คือ layer ทางเทคนิค ไม่ใช่ domain) ที่ถูกต้องคือแบ่งตามความสามารถทางธุรกิจ เช่น `order-service`, `shipping-service`, `billing-service` — แต่ละตัวมีทั้ง data, logic, validation ของตัวเองครบ

ตัวชี้วัดที่ใช้ตรวจสอบง่ายๆ ว่า boundary ที่แบ่งไว้ดีหรือไม่: **ถ้า deploy service A แล้วต้อง deploy service B พร้อมกันเสมอ** หรือ **ถ้า service A ต้องเรียก service B มากกว่า 1 ครั้งเพื่อทำงานหนึ่งอย่างให้เสร็จ (chatty)** นั่นเป็นสัญญาณว่า boundary อาจตัดผิดจุด — ควรพิจารณารวมสอง service กลับเป็นตัวเดียว หรือย้าย logic ให้ตรงกับข้อมูลที่มันต้องใช้บ่อยที่สุด

เราจะกลับมาเจาะลึกเรื่อง DDD, Aggregate, Bounded Context และ Ubiquitous Language แบบเต็มรูปแบบใน **Part 101 (Domain-Driven Design ใน Go)** ซึ่งเป็นส่วนหนึ่งของภาคที่ 10

---

## 6. การสื่อสารระหว่าง Service: Synchronous vs Asynchronous

เมื่อแตกระบบเป็นหลาย service แล้ว คำถามถัดไปคือ "แต่ละ service จะคุยกันอย่างไร" คำตอบแบ่งเป็นสองแนวทางหลัก ซึ่งเป็นแกนของเนื้อหาทั้งภาคที่ 8:

### Synchronous Communication (แบบซิงโครนัส)

Service A เรียก service B แล้ว **รอ (block)** จนกว่าจะได้รับคำตอบกลับมาก่อนจะทำงานต่อ เช่น REST API (ที่เรียนไปแล้วใน Part 059) หรือ gRPC (ที่จะเรียนใน **Part 089–090**)

```
Client → [Order Service] ──HTTP/gRPC──► [Payment Service]
              │                                │
              │◄───────── รอ response ─────────┘
              ▼
         ตอบกลับ Client
```

**ข้อดี**: เข้าใจง่าย เหมือนเรียกฟังก์ชันปกติ, ได้ผลลัพธ์ทันที, error handling ตรงไปตรงมา (รู้ทันทีว่าสำเร็จหรือไม่)

**ข้อเสีย**: ถ้า service ปลายทางช้าหรือล่ม service ต้นทางก็ต้องรอหรือ fail ตามไปด้วย (**cascading failure**) ยิ่งมี chain การเรียกยาว (A เรียก B เรียก C เรียก D) latency รวมยิ่งสูงและความเสี่ยง failure ยิ่งทวีคูณ

### Asynchronous Communication (แบบอะซิงโครนัส)

Service A ส่งข้อความ (message/event) ไปที่ **message broker** (เช่น RabbitMQ ที่จะเรียนใน **Part 091** หรือ Kafka ใน **Part 092**) แล้ว**ทำงานต่อทันทีโดยไม่รอ** service ปลายทางประมวลผลเสร็จ

```
Client → [Order Service] ──publish──► [Message Queue] ──consume──► [Payment Service]
              │
              ▼
       ตอบกลับ Client ทันที (ไม่รอ payment ประมวลผลเสร็จ)
```

**ข้อดี**: service ต้นทางไม่ถูกบล็อกแม้ปลายทางช้าหรือล่มชั่วคราว (message รอใน queue ได้), **buffer load spike** ได้ (ถ้ามีคำสั่งซื้อเข้ามาพร้อมกัน 10,000 รายการ queue จะรับไว้แล้วให้ consumer ค่อยๆ ประมวลผลตามกำลังของตัวเอง แทนที่จะให้ระบบล่มเพราะรับ traffic พร้อมกันหมด), **decouple** service ออกจากกันอย่างแท้จริง (service ต้นทางไม่จำเป็นต้องรู้ด้วยซ้ำว่ามีใครฟัง message นี้อยู่บ้าง)

**ข้อเสีย**: ซับซ้อนกว่า, ไม่ได้ผลลัพธ์ทันที (client ต้อง poll หรือรอ webhook/notification แยกต่างหากถ้าต้องการรู้ผลจริง), debug ยากกว่า (ต้องไล่ตาม message ข้าม queue), ต้องจัดการเรื่อง **eventual consistency** (ข้อมูลจะ "ถูกต้องในที่สุด" ไม่ใช่ "ถูกต้องทันที")

### เมื่อไหร่ควรใช้แบบไหน

| สถานการณ์ | แนวทางที่เหมาะสม |
|---|---|
| ต้องการผลลัพธ์ทันทีเพื่อแสดงให้ผู้ใช้เห็น (เช่น "ยืนยันการชำระเงินสำเร็จหรือไม่") | Synchronous (REST/gRPC) |
| งานที่ไม่ต้องรอผลทันที (เช่น "ส่งอีเมลยืนยัน", "อัปเดตรายงานสถิติ") | Asynchronous (Message Queue) |
| ต้องการ decouple service ให้ deploy/scale อิสระจากกันมากที่สุด | Asynchronous |
| Traffic มีลักษณะเป็น spike สูงๆ ต่ำๆ ไม่สม่ำเสมอ | Asynchronous (queue ทำหน้าที่ buffer) |
| Logic ต้องการ "รู้ผลแล้วตัดสินใจต่อทันที" ในขั้นตอนเดียวกัน | Synchronous |

ระบบจริงส่วนใหญ่ **ใช้ทั้งสองแบบผสมกัน**: เช่น `order-service` เรียก `inventory-service` แบบ synchronous (gRPC) เพื่อเช็คสต็อกทันทีก่อนยืนยันคำสั่งซื้อ แต่หลังจากยืนยันสำเร็จแล้ว จะ publish event `OrderCreated` แบบ asynchronous ไปที่ message queue ให้ `notification-service`, `analytics-service`, `shipping-service` แต่ละตัวไปทำงานของตัวเองอิสระโดยไม่ทำให้ผู้ใช้ต้องรอ

---

## 7. ปัญหาเฉพาะของ Distributed Systems ที่ Monolith ไม่เคยเจอ

เมื่อ logic กระจายออกไปหลายเครื่อง เชื่อมกันผ่าน network จะเกิดปัญหาประเภทใหม่ที่ Monolith ไม่มีทางเจอเลย เพราะการเรียกฟังก์ชันในโปรเซสเดียวกันแทบไม่มีทางล้มเหลวกลางทาง แต่การเรียกผ่าน network **ล้มเหลวได้เสมอ**

### Partial Failure (ความล้มเหลวบางส่วน)

ใน Monolith ถ้าโปรแกรม crash ทุกอย่างหยุดพร้อมกัน แต่ใน Microservices เป็นไปได้ที่ `order-service` ยังทำงานปกติ ในขณะที่ `payment-service` ล่มไปแล้ว ระบบต้องถูกออกแบบให้ **ทนต่อความล้มเหลวบางส่วน** ได้ ไม่ใช่ล่มยกระบบตามกันเป็นโดมิโน (cascading failure) — นี่คือที่มาของแนวคิดอย่าง **Circuit Breaker** ที่เราจะเรียนใน **Part 094**

### Network Unreliability (เครือข่ายไม่น่าเชื่อถือ 100%)

การเรียก API ข้าม network อาจ:

- **Timeout** — ไม่รู้ว่าฝั่งปลายทางทำงานสำเร็จหรือไม่ ก่อนที่ connection จะขาด
- **ส่งไปถึงแต่ response หายระหว่างทาง** — ปลายทางประมวลผลสำเร็จแล้ว แต่ต้นทางไม่รู้ (เข้าใจผิดว่าล้มเหลว) — ถ้า retry ซ้ำ อาจทำให้เกิด**การประมวลผลซ้ำ** (เช่น เก็บเงินซ้ำสองครั้ง) ทำให้ต้องออกแบบ endpoint ให้เป็น **idempotent** (เรียกซ้ำกี่ครั้งผลลัพธ์เหมือนเดิม)
- **Packet loss, latency spike** — สิ่งที่ Monolith (เรียกฟังก์ชันในหน่วยความจำเดียวกัน) ไม่มีทางเจอเลย

หลักการที่มีชื่อเสียงเรียกว่า **The Eight Fallacies of Distributed Computing** (ความเข้าใจผิดที่พบบ่อยของนักพัฒนาที่ไม่เคยทำระบบ distributed มาก่อน) ซึ่งทุกข้อคือสมมติฐานที่คนมักเผลอเชื่อ แต่เป็นเท็จเสมอในระบบจริง:

1. The network is reliable — เครือข่ายเชื่อถือได้ 100%
2. Latency is zero — การส่งข้อมูลไม่มี delay
3. Bandwidth is infinite — แบนด์วิดท์ไม่จำกัด
4. The network is secure — เครือข่ายปลอดภัยเสมอ
5. Topology doesn't change — โครงสร้างเครือข่ายไม่เปลี่ยนแปลง
6. There is one administrator — มีผู้ดูแลระบบคนเดียว
7. Transport cost is zero — การส่งข้อมูลไม่มีต้นทุน
8. The network is homogeneous — เครือข่ายเป็นแบบเดียวกันทั้งหมด

ทุกข้อในลิสต์นี้ล้วนเป็น "สมมติฐานที่เป็นจริงตลอดเวลา" เมื่อ logic ทั้งหมดรันในโปรเซสเดียวกันแบบ Monolith แต่กลายเป็น "เท็จเสมอ" ทันทีที่ต้องเรียกข้าม network — ทุกการเรียกข้าม service ต้องเตรียมรับมือกับความล้มเหลวไว้ล่วงหน้าเสมอ ไม่ว่าจะด้วย timeout, retry with backoff, หรือ circuit breaker (Part 094)

### Eventual Consistency (ความสอดคล้องที่มาช้า)

ใน Monolith เราใช้ database transaction เดียว (ACID) ครอบคลุมการเปลี่ยนแปลงข้อมูลทั้งหมดในคำสั่งเดียว (ตามที่เรียนใน Part 078) แต่เมื่อข้อมูลกระจายอยู่คนละฐานข้อมูล คนละ service **ไม่มีทางทำ transaction เดียวครอบคลุมทุก service ได้อีกต่อไป** (distributed transaction แบบ two-phase commit ทำได้ในทางทฤษฎีแต่ช้ามากและเปราะบางในทางปฏิบัติ แทบไม่มีใครใช้จริงในระบบสมัยใหม่)

ทางออกที่ใช้กันจริงคือยอมรับ **Eventual Consistency**: หลังจากเหตุการณ์เกิดขึ้น (เช่น สร้างคำสั่งซื้อสำเร็จ) service อื่นๆ จะได้รับ event แล้วอัปเดตข้อมูลของตัวเองตามมา "ในเวลาไม่นาน" แทนที่จะ "ทันทีเป๊ะ" เช่น หลังสั่งซื้อสำเร็จ อาจใช้เวลาไม่กี่วินาทีกว่า `inventory-service` จะลดสต็อกสินค้าจริง หรือกว่า `analytics-service` จะอัปเดตยอดขายในแดชบอร์ด — ระบบจึงต้องออกแบบ UX และ logic ให้ยอมรับความล่าช้าเล็กน้อยนี้ได้ และมักใช้ pattern เช่น **Saga Pattern** เพื่อจัดการ transaction ที่ครอบคลุมหลาย service ด้วยชุดของ compensating action แทนการ rollback แบบ database transaction ปกติ

---

## 8. API Gateway: ประตูเดียวสู่ระบบทั้งหมด

เมื่อมี service ย่อยหลายสิบตัว จะเกิดคำถามว่า **client ภายนอก (เว็บ, มือถือ) จะรู้ได้อย่างไรว่าต้องเรียก service ไหน ที่ address อะไร** ถ้าปล่อยให้ client เรียกตรงไปแต่ละ service เอง จะเกิดปัญหา:

- Client ต้องรู้จัก network topology ภายในทั้งหมด (ที่ควรเป็นความลับของระบบ)
- แต่ละ service ต้องทำ authentication/rate limiting/logging ซ้ำๆ กันเอง
- เปลี่ยนโครงสร้าง service ภายใน (เช่น รวม/แยก service) กระทบ client โดยตรงทันที

**API Gateway** คือ service ตัวหนึ่งที่ทำหน้าที่เป็น **จุดเข้าเดียว (single entry point)** ให้ client ภายนอกทั้งหมด รับ request เข้ามาแล้ว route ไปยัง service ภายในที่ถูกต้อง พร้อมทำหน้าที่ส่วนกลางที่ไม่ควรให้แต่ละ service ทำซ้ำ:

```
                        ┌─────────────────┐
  Client (Web/Mobile) ─►│   API Gateway    │
                        │  - Auth          │
                        │  - Rate Limit    │
                        │  - Routing       │
                        │  - Logging       │
                        └────────┬─────────┘
                    ┌────────────┼────────────┐
                    ▼            ▼            ▼
              [User Service] [Order Service] [Payment Service]
```

API Gateway ยังมักทำหน้าที่แปลงโปรโตคอลด้วย เช่น client ภายนอกคุยด้วย REST/JSON ตามปกติ แต่ภายในระบบ service คุยกันด้วย gRPC ที่เร็วกว่า (สิ่งที่เรียกว่า **gRPC-Gateway** ซึ่งเราจะพูดถึงสั้นๆ ใน Part 090 และเจาะลึกเรื่อง service discovery กับ API Gateway เต็มรูปแบบใน **Part 093**)

---

## 9. ทำไม Go เหมาะกับ Microservices เป็นพิเศษ

ย้อนกลับไปที่ Part 001 เราพูดถึงจุดเด่นของ Go ไว้หลายข้อ — ในบริบทของ Microservices จุดเด่นเหล่านั้นสำคัญกว่าที่เคยเป็นตอนเขียน Monolith เดี่ยวๆ มาก เพราะเมื่อมี service เป็นสิบเป็นร้อยตัว **ต้นทุนต่อ service คูณด้วยจำนวน service** ทำให้ทุกความไม่มีประสิทธิภาพขยายผลออกไปมหาศาล

### Binary เดียว ไม่มี runtime dependency

Go compile เป็น **static binary เดียวจบ** ไม่ต้องติดตั้ง runtime แยก (ต่างจาก Java ที่ต้องมี JVM, Python/Node ที่ต้องมี interpreter และ dependency ครบชุดตอน deploy) ทำให้ container image ของแต่ละ microservice **เล็กมาก** — ทดลองจริงในเครื่องที่ใช้เขียนบทเรียนนี้:

```bash
CGO_ENABLED=0 go build -ldflags="-s -w" -o app .
ls -lh app
```

ผลลัพธ์ที่วัดได้จริง:

```
-rwxr-xr-x 1 root root 1.4M Sep 26 03:43 app
```

โปรแกรม Go แบบง่ายๆ (แค่ `fmt.Println`) ได้ binary ขนาด **1.4 MB** เท่านั้น — เทียบกับ container image ของ Java/Spring Boot ที่มักเริ่มต้นที่หลักร้อย MB เพราะต้องรวม JVM เข้าไปด้วย เมื่อคูณด้วย service เป็นร้อยตัว ความต่างของ image size ส่งผลโดยตรงต่อเวลาที่ใช้ pull image, พื้นที่ storage ใน registry, และความเร็วในการ scale (container ที่ image เล็กสร้างและเริ่มทำงานได้เร็วกว่ามาก)

### Startup เร็วมาก

เพราะไม่มี runtime ให้ warm up (ไม่มี JIT compilation, ไม่มี class loading แบบ JVM) โปรแกรม Go เริ่มทำงานได้เกือบจะทันที — ทดลองรัน binary ตัวเดียวกัน 10 ครั้งติดกันในเครื่องเดียวกัน:

```bash
time (for i in $(seq 1 10); do ./app > /dev/null; done)
```

ผลลัพธ์ที่วัดได้จริง:

```
real    0m0.027s
```

รวม 10 ครั้ง (process spawn + run + exit) ใช้เวลาเพียง **27 มิลลิวินาที** เฉลี่ยแล้วต่ำกว่า 3 มิลลิวินาทีต่อครั้ง (ตัวเลขนี้เป็นตัวอย่างสาธิตด้วยโปรแกรมเล็กมากในเครื่อง sandbox ไม่ใช่ benchmark มาตรฐานอุตสาหกรรม แต่สะท้อนหลักการได้ชัดเจน) startup ที่เร็วขนาดนี้สำคัญมากสำหรับ:

- **Auto-scaling**: เมื่อ traffic พุ่งขึ้นกะทันหัน container ใหม่พร้อมรับ traffic ได้เกือบจะทันที ต่างจากบาง runtime ที่ต้อง warm up หลายวินาทีก่อนจะ serve ได้เต็มประสิทธิภาพ
- **Serverless/Function-as-a-Service**: แพลตฟอร์มอย่าง AWS Lambda คิดค่าใช้จ่ายรวม cold start time ด้วย ภาษาที่ startup เร็วจึงประหยัดกว่าโดยตรง
- **Rolling deployment**: restart instance ทีละตัวได้เร็ว ลด downtime ระหว่าง deploy

### หน่วยความจำต่ำ

Go ใช้ garbage collector ที่ออกแบบมาให้ pause time ต่ำและ overhead ของ runtime เองก็เล็กกว่า JVM/CLR มาก ทำให้รัน container จำนวนมากบนเครื่องเดียวกันได้หนาแน่นกว่า (higher density) ประหยัดค่าใช้จ่ายโครงสร้างพื้นฐานเมื่อคูณด้วยจำนวน service ที่มาก

### Concurrency ในตัวที่เหมาะกับงาน I/O-bound ของ Microservices

Microservice ส่วนใหญ่ใช้เวลาทำงานไปกับการ **รอ I/O** (รอ database, รอเรียก service อื่น, รอ message จาก queue) มากกว่าคำนวณหนักๆ Goroutines และ Channels (ที่เรียนในภาคที่ 3) ทำให้ Go จัดการ concurrent connection จำนวนมากพร้อมกันได้อย่างมีประสิทธิภาพ โดยใช้ syntax ที่เขียนง่ายกว่าภาษาอื่นมาก — เหมาะโดยตรงกับธรรมชาติของงานที่ service หนึ่งต้องรับ request จำนวนมากพร้อมกัน แล้วแต่ละ request ก็ไปเรียก service อื่นต่อ (ซึ่งเป็น I/O-bound ล้วนๆ)

### Cross-compilation และ deploy ง่าย

ตามที่เรียนใน Part 001 การ cross-compile ของ Go ทำให้ build image สำหรับ target platform ต่างๆ (Linux amd64, arm64 สำหรับ ARM-based cloud instance) ได้จากเครื่อง build เดียว โดยไม่ต้องมีเครื่องเป้าหมายจริง — สำคัญมากในระบบ CI/CD ที่ต้อง build image ของ service หลายสิบตัวโดยอัตโนมัติ

จุดร่วมของทุกข้อข้างต้นคือ: **Go ไม่ได้ทำให้คุณเขียน Microservices ได้ "ถูกต้อง" มากขึ้น** (เรื่อง service boundary, distributed transaction ยังคงเป็นปัญหาการออกแบบที่ต้องคิดเองเสมอ ไม่ว่าจะใช้ภาษาอะไร) แต่ Go ลด **operational cost ต่อ service** ลงอย่างมีนัยสำคัญ ซึ่งสำคัญมากเมื่อต้นทุนนั้นถูกคูณด้วยจำนวน service เป็นสิบเป็นร้อยตัวในระบบจริง — นี่คือเหตุผลที่บริษัทที่ขับเคลื่อนด้วย microservices จำนวนมาก (Uber, Google, Cloudflare, ตามที่กล่าวถึงใน Part 001) เลือกใช้ Go เป็นภาษาหลักสำหรับเขียน service เหล่านี้

---

## 10. แผนการเดินทางของภาคที่ 8

ก่อนไปต่อ สรุปภาพรวมว่าภาคที่ 8 จะพาไปที่ไหนบ้าง:

- **Part 089**: gRPC เบื้องต้น — โปรโตคอล synchronous ที่เร็วกว่า REST สำหรับสื่อสารระหว่าง service ภายใน
- **Part 090**: gRPC ขั้นสูง — Streaming และ Interceptor สำหรับ use case ที่ซับซ้อนขึ้น
- **Part 091**: RabbitMQ — message broker ยอดนิยมสำหรับ asynchronous communication แบบ queue-based
- **Part 092**: Kafka — message broker สำหรับ event streaming ขนาดใหญ่
- **Part 093**: Service Discovery และ API Gateway — วิธีที่ service หาเจอกันเองในระบบที่ instance เปลี่ยนแปลงตลอดเวลา
- **Part 094**: Circuit Breaker และ Retry Pattern — เทคนิครับมือกับ partial failure ที่พูดถึงในหัวข้อ 7

ทุก part ในภาคนี้ต่อยอดจากหลักการพื้นฐานที่วางไว้ในบทนี้ทั้งหมด

---

## สรุปสิ่งที่ได้เรียนในบทนี้

- **Monolith** รวมทุกอย่างไว้ใน codebase/deploy เดียว พัฒนาง่าย debug ง่าย แต่ scale องค์กรยาก เมื่อทีมโตขึ้น
- **Microservices** แตกระบบเป็น service อิสระ แก้ปัญหา **organizational scaling** และ **independent deployability** เป็นหลัก ไม่ใช่ตัวช่วยด้าน performance
- Microservices มีต้นทุนสูงกว่าเสมอในแง่ operational complexity — ไม่ใช่ทางเลือกที่ดีกว่าโดยอัตโนมัติ ทีมเล็กควรเริ่มที่ Monolith ก่อนตามแนวคิด **"Monolith First"** ของ Martin Fowler
- การแบ่ง **service boundary** ที่ดีอิงหลัก Domain-Driven Design (Bounded Context, high cohesion/low coupling) จะเจาะลึกใน Part 101
- การสื่อสารระหว่าง service แบ่งเป็น **Synchronous** (REST/gRPC — รอผลทันที) และ **Asynchronous** (Message Queue — decouple และ buffer load ได้) ระบบจริงมักใช้ผสมกันทั้งสองแบบ
- Distributed systems มีปัญหาเฉพาะที่ Monolith ไม่เจอ: **Partial failure**, **Network unreliability**, **Eventual consistency**
- **API Gateway** เป็นจุดเข้าเดียวของระบบ จัดการ routing, auth, rate limit ให้ client ภายนอกไม่ต้องรู้จัก topology ภายใน
- Go เหมาะกับ Microservices เพราะ static binary ขนาดเล็ก (วัดได้จริง ~1.4 MB), startup เร็วมาก (วัดได้จริง ~2.7ms ต่อครั้งสำหรับโปรแกรมสาธิต), หน่วยความจำต่ำ, และ concurrency model ที่เหมาะกับงาน I/O-bound ซึ่งเป็นธรรมชาติของ microservice ส่วนใหญ่

## แบบฝึกหัดท้ายบท

1. เลือกแอปพลิเคชันหนึ่งที่คุณเคยสร้างในหลักสูตรนี้ (เช่นโปรเจกต์จาก Part 059 หรือ Part 078) ลองระบุว่าถ้าจะแตกเป็น Microservices จะแบ่งเป็น service อะไรบ้าง และอธิบายเหตุผลตามหลัก Bounded Context จากหัวข้อ 5
2. อธิบายด้วยคำพูดตัวเองว่าทำไม "Microservices ทำให้ระบบเร็วขึ้น" เป็นความเข้าใจผิด พร้อมยกตัวอย่างสถานการณ์ที่ Microservices จะทำให้ระบบ**ช้าลง**เมื่อเทียบกับ Monolith
3. ยกตัวอย่าง use case จริงในระบบ e-commerce ที่ควรใช้ **synchronous communication** และอีก use case ที่ควรใช้ **asynchronous communication** พร้อมอธิบายเหตุผล
4. ค้นคว้าเพิ่มเติมเรื่อง **Saga Pattern** ว่าใช้แก้ปัญหา distributed transaction อย่างไรเมื่อไม่มี ACID transaction ครอบคลุมหลาย service ได้อีกต่อไป (เตรียมคำตอบไว้ เราจะไม่ลงรายละเอียดในภาคนี้ แต่เป็นแนวคิดสำคัญที่ต่อยอดจากหัวข้อ 7)
5. ลองรันคำสั่ง `go build -ldflags="-s -w"` กับโปรแกรม Go เล็กๆ ของตัวเอง แล้วเทียบขนาดไฟล์กับ `go build` แบบปกติ (ไม่ใส่ ldflags) อธิบายว่า flag `-s -w` ทำอะไร
6. จากตัวชี้วัด "ถ้า deploy service A ต้อง deploy service B พร้อมกันเสมอ = boundary ผิด" ในหัวข้อ 5 — ลองนึกถึงระบบหนึ่งที่คุณรู้จัก (จากที่ทำงานหรือใช้งานทั่วไป) แล้ววิเคราะห์ว่ามันเข้าข่ายปัญหานี้หรือไม่
7. เขียนโครงสร้าง Modular Monolith แบบง่ายๆ ด้วยตัวเอง (คล้ายตัวอย่างในหัวข้อ 4) ที่มี 2 module สื่อสารกันผ่าน interface เท่านั้น แล้วลองอธิบายว่าถ้าจะแตก module หนึ่งออกเป็น microservice ต้องแก้โค้ดตรงไหนบ้าง

---

**ต่อไป**: [Part 089 — gRPC เบื้องต้นด้วย Protocol Buffers](./089-grpc-basics.md)
