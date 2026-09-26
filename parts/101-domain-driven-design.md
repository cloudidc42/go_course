# Part 101: Domain-Driven Design (DDD) ใน Go

> ภาคที่ 10: มืออาชีพและระดับโลก (Professional & World-Class) — ตอนที่ 2 จาก 11 (Part 100–110)

Part 100 สอนเรื่อง**การจัดชั้นของโค้ด** (Clean Architecture) — บทนี้จะสอนเรื่องที่ลึกกว่านั้นหนึ่งขั้น: **การจัดระเบียบความคิดเกี่ยวกับตัวธุรกิจเอง** ก่อนจะเขียนโค้ดสักบรรทัด **Domain-Driven Design (DDD)** เป็นแนวคิดที่ Eric Evans เผยแพร่ในหนังสือ "Domain-Driven Design: Tackling Complexity in the Heart of Software" (2003) ซึ่งพูดถึงวิธีทำให้โค้ด**สะท้อนความเป็นจริงทางธุรกิจอย่างซื่อสัตย์** แทนที่จะเป็นแค่ตารางฐานข้อมูลที่ห่อด้วย API

บทนี้จะอธิบาย DDD แบบปฏิบัติได้จริง ไม่ใช่แบบวิชาการล้วนๆ — ทุก concept จะมาพร้อมโค้ด Go ที่รันได้จริง

## สารบัญของบทนี้

1. DDD คืออะไร และปัญหาที่มันแก้ (ไม่ใช่แค่ "ตั้งชื่อ class ให้เท่")
2. Ubiquitous Language: ภาษาเดียวกันระหว่างธุรกิจกับโค้ด
3. Entity vs Value Object: ความแตกต่างที่สำคัญที่สุดใน DDD
4. Value Object ตัวอย่าง: `Money` ที่ปลอดภัยและ Immutable
5. Aggregate และ Aggregate Root: ขอบเขตของความสอดคล้อง (Consistency Boundary)
6. ตัวอย่างเต็ม: `Order` Aggregate ที่ปกป้อง Invariant ของตัวเอง
7. Domain Events: สื่อสารการเปลี่ยนแปลงโดยไม่ผูกติดกัน
8. Repository แบบ DDD: หนึ่งตัวต่อหนึ่ง Aggregate Root เท่านั้น
9. Bounded Context: ทำไมคำว่า "Order" ในสองทีมถึงไม่ใช่สิ่งเดียวกัน
10. ตัวอย่างเต็มแบบรวม: Application Service ที่ประสาน Aggregate + Event Bus
11. ข้อควรระวังตรงไปตรงมา: DDD ไม่เหมาะกับ CRUD ธรรมดา
12. สรุปสิ่งที่ได้เรียนในบทนี้
13. แบบฝึกหัดท้ายบท

---

## 1. DDD คืออะไร และปัญหาที่มันแก้ (ไม่ใช่แค่ "ตั้งชื่อ class ให้เท่")

หลายคนเข้าใจผิดว่า DDD คือ "การตั้งชื่อ struct ให้ตรงกับคำศัพท์ทางธุรกิจ" ซึ่งเป็นแค่ผลลัพธ์ส่วนหนึ่งเท่านั้น หัวใจจริงๆ ของ DDD คือการแก้ปัญหาคลาสสิกที่เกิดกับระบบซอฟต์แวร์ที่ซับซ้อนทุกระบบ:

> **นักธุรกิจ (domain expert) อธิบายกฎทางธุรกิจด้วยคำหนึ่ง แต่นักพัฒนาเข้าใจและเขียนโค้ดออกมาอีกความหมายหนึ่ง**

ตัวอย่างที่พบบ่อยมาก: ทีมขายพูดว่า "ยกเลิก order ได้ตลอดเวลา" แต่ทีมคลังสินค้าหมายถึง "ยกเลิกได้ก่อนที่จะแพ็คของเท่านั้น" ถ้านักพัฒนาไม่รู้ความแตกต่างนี้ตั้งแต่แรก ก็จะเขียนโค้ด `CancelOrder()` ที่ยอมให้ยกเลิกได้เสมอ ทั้งที่ธุรกิจจริงมีเงื่อนไขซ่อนอยู่ — บั๊กแบบนี้ไม่ใช่บั๊กทางเทคนิค (compile ผ่าน รันได้) แต่เป็น **"บั๊กทางความเข้าใจธุรกิจ"** ซึ่งอันตรายกว่ามากเพราะตรวจจับด้วย test อัตโนมัติทั่วไปไม่ได้ถ้าคนเขียน test ก็เข้าใจผิดเหมือนกัน

DDD เสนอชุดแนวคิด (ทั้ง **strategic patterns** สำหรับมองภาพใหญ่ระดับองค์กร และ **tactical patterns** สำหรับออกแบบโค้ดระดับละเอียด) ที่ช่วยลดช่องว่างนี้ — บทนี้จะเน้น tactical patterns ที่นำไปเขียนโค้ด Go ได้ทันที: Entity, Value Object, Aggregate, Domain Event, Repository ส่วน strategic pattern (Bounded Context) จะพูดถึงในหัวข้อ 9

---

## 2. Ubiquitous Language: ภาษาเดียวกันระหว่างธุรกิจกับโค้ด

**Ubiquitous Language** (ภาษาสากลร่วม) คือหลักการที่ว่า **คำศัพท์ที่ domain expert ใช้พูดคุยกันในชีวิตประจำวัน ต้องเป็นคำเดียวกันกับที่ปรากฏในโค้ด** ทุกตัวอักษร ไม่ผ่านการแปลหรือย่อความใดๆ

ตัวอย่างเปรียบเทียบ:

| สิ่งที่ทีมขายพูด | โค้ดที่ **ไม่ทำตาม** Ubiquitous Language | โค้ดที่ **ทำตาม** Ubiquitous Language |
|---|---|---|
| "ยืนยันคำสั่งซื้อ" | `func (o *Order) SetStatus(2)` | `func (o *Order) Submit()` |
| "สินค้าหมดสต็อก" | `if p.Qty == 0` | `if p.IsOutOfStock()` |
| "ลูกค้า VIP" | `if u.Tier == 3` | `if c.IsVIP()` |

คอลัมน์ซ้ายไม่ผิดในแง่ compile — แต่มันบังคับให้ทุกคนที่อ่านโค้ดต้อง "แปล" ตัวเลข magic number กลับเป็นความหมายทางธุรกิจในหัวตัวเองทุกครั้ง ยิ่งระบบใหญ่ขึ้น การแปลไป-มาแบบนี้ยิ่งสะสมความเข้าใจผิดมากขึ้นเรื่อยๆ ในขณะที่คอลัมน์ขวา**อ่านออกเสียงแล้วตรงกับที่ domain expert พูดจริงๆ**

ผลในทางปฏิบัติของ Ubiquitous Language คือ: ชื่อ method, struct, field, error message ควรมาจาก**การพูดคุยกับคนที่เข้าใจธุรกิจจริง** ไม่ใช่มาจากจินตนาการของนักพัฒนาเอง และเมื่อธุรกิจเปลี่ยนคำศัพท์ (เช่น เปลี่ยนจาก "ยกเลิก" เป็น "ระงับชั่วคราว") โค้ดก็ควรถูก rename ตามทันที — นี่คือเหตุผลที่ DDD มักไปคู่กับการ refactor บ่อยๆ ตลอดอายุโปรเจกต์ ไม่ใช่ออกแบบครั้งเดียวจบ

---

## 3. Entity vs Value Object: ความแตกต่างที่สำคัญที่สุดใน DDD

นี่คือแนวคิดพื้นฐานที่สุดของ DDD tactical pattern และเป็นจุดที่มือใหม่สับสนบ่อยที่สุด — ความแตกต่างสรุปได้ในตารางเดียว:

| | **Entity** | **Value Object** |
|---|---|---|
| มี identity (ID) เป็นของตัวเองไหม | มี — สองตัวที่ field เหมือนกันทุกอย่างแต่ ID ต่างกัน ถือเป็นคนละตัว | ไม่มี — สองตัวที่ field เหมือนกันทุกอย่าง ถือว่า**เป็นตัวเดียวกัน** |
| เปรียบเทียบด้วยอะไร | ID | ค่าของทุก field รวมกัน |
| Mutable ไหม | มักจะ mutable (แก้ไข state ได้ตลอดอายุ) | ควร **immutable** เสมอ (สร้างใหม่แทนการแก้ไข) |
| ตัวอย่าง | `Order` (order #1001 ยังเป็น order #1001 เสมอ แม้จะแก้ไข status/lines) | `Money`, `Address`, `DateRange`, `Color` |

ตัวอย่างที่ทำให้เห็นภาพชัด: ธนบัตร 100 บาทสองใบในกระเป๋าเงินคุณ — ถ้ามองในแง่ **Value Object** สองใบนี้ "เหมือนกันทุกประการ" ไม่ต้องแยกแยะว่าใบไหนเป็นใบไหน สลับกันก็ไม่มีความหมายอะไรเปลี่ยนไป แต่ถ้าเป็น **Entity** เช่น "บัตรประชาชน" สองใบที่ข้อมูลหน้าตาเหมือนกันทุกอย่าง (ชื่อเดียวกัน วันเกิดเดียวกัน) ก็ยังถือเป็นคนละใบกันถ้าเลขบัตรประชาชนต่างกัน — **identity คือสิ่งที่ทำให้ entity เป็น entity**

ใน Go แนวคิดนี้แปลเป็นโค้ดตรงไปตรงมา:

```go
// Entity: มี field ID ที่ใช้เปรียบเทียบ "ความเป็นตัวเดียวกัน" (ทบทวน Part 011)
type Order struct {
	ID     string // identity
	Status OrderStatus
	// ... field อื่นๆ ที่แก้ไขได้ตลอดอายุของ order
}

// Value Object: ไม่มี ID เปรียบเทียบด้วยค่าล้วนๆ ผ่าน struct comparison ปกติ
// (ทบทวน Part 011 เรื่อง comparable struct) — Go ให้ "ฟรี" มาแล้วโดยธรรมชาติ
// ผ่าน struct ที่มีแต่ field ชนิด comparable
type Money struct {
	Amount   int64
	Currency string
}
```

จุดที่น่าสนใจคือ **ภาษา Go เอื้อต่อการทำ Value Object เป็นพิเศษ** เพราะ struct ที่ประกอบด้วย comparable field ล้วนๆ (string, int, bool, struct ที่ field เป็น comparable ทั้งหมด) จะเปรียบเทียบกันด้วย `==` ได้ทันทีโดยไม่ต้องเขียน `Equals()` เอง แตกต่างจากภาษาอย่าง Java ที่ต้อง override `equals()`/`hashCode()` เองทุกครั้ง — นี่คือหนึ่งในจุดที่ Go "ทำ DDD ได้เป็นธรรมชาติ" โดยไม่ต้องมี framework หรือ annotation ช่วย

---

## 4. Value Object ตัวอย่าง: `Money` ที่ปลอดภัยและ Immutable

`Money` คือตัวอย่างคลาสสิกที่สุดของ Value Object ในวงการ DDD — มาดูการ implement เต็มรูปแบบที่ verify แล้วว่า compile และรันถูกต้อง:

```go
package main

import (
	"errors"
	"fmt"
)

// Money คือตัวอย่างคลาสสิกของ "Value Object" ใน DDD:
//   - ไม่มี identity ของตัวเอง (ไม่มี ID) — เงิน 100 บาทสองก้อนถือว่า "เหมือนกันทุก
//     ประการ" ไม่ต้องแยกแยะว่าก้อนไหนเป็นก้อนไหน ต่างจาก Entity เช่น Order ที่ต่อให้
//     ข้อมูลข้างในเหมือนกันเป๊ะ ก็ยังถือเป็นคนละ order กันถ้า ID ต่างกัน
//   - Immutable: ทุก method ที่ "แก้ไข" ค่า จะคืน Money ตัวใหม่เสมอ ไม่แก้ struct เดิม
//   - เปรียบเทียบกันด้วย "ค่า" ล้วนๆ — เพราะ field ทั้งหมดเป็น comparable type
//     (string, int64) เราจึงใช้ `==` เปรียบเทียบ Money สองตัวได้ตรงๆ โดยไม่ต้องเขียน
//     Equals() เอง (ทบทวนเรื่อง struct comparison จาก Part 011)
//
// Amount เก็บเป็น "หน่วยที่เล็กที่สุด" ของสกุลเงิน (สตางค์/cent) เป็น int64 เสมอ
// ไม่ใช้ float64 เด็ดขาด — เพราะ floating point มีปัญหาเรื่อง rounding error
// สะสมที่ยอมรับไม่ได้เลยกับเงิน (เช่น 0.1 + 0.2 ไม่เท่ากับ 0.3 พอดีใน IEEE 754)
type Money struct {
	Amount   int64 // หน่วยสตางค์ เช่น 10000 = 100.00 บาท
	Currency string
}

var (
	ErrCurrencyMismatch = errors.New("money: currency mismatch")
	ErrNegativeAmount   = errors.New("money: amount must not be negative")
)

// NewMoney สร้าง Money จากจำนวนเงินหลักหน่วย (บาท/ดอลลาร์) แปลงเป็นสตางค์/เซนต์ให้เอง
func NewMoney(units float64, currency string) Money {
	return Money{Amount: int64(units*100 + 0.5), Currency: currency}
}

// Add คืน Money ตัวใหม่ (ไม่แก้ตัวเดิม) — ปฏิเสธการบวกข้ามสกุลเงินทันที
// นี่คือตัวอย่างของการ "enforce invariant" ตั้งแต่ระดับ value object เอง
// ไม่ปล่อยให้ผู้เรียกบวก 100 THB + 5 USD ได้แบบไม่มีความหมาย
func (m Money) Add(other Money) (Money, error) {
	if m.Currency != other.Currency {
		return Money{}, fmt.Errorf("%w: %s vs %s", ErrCurrencyMismatch, m.Currency, other.Currency)
	}
	return Money{Amount: m.Amount + other.Amount, Currency: m.Currency}, nil
}

// Multiply คูณด้วยจำนวนเต็ม (เช่น unit price * quantity)
func (m Money) Multiply(qty int) Money {
	return Money{Amount: m.Amount * int64(qty), Currency: m.Currency}
}

func (m Money) String() string {
	return fmt.Sprintf("%.2f %s", float64(m.Amount)/100, m.Currency)
}

// IsZero ใช้เช็คว่า Money นี้เป็นค่าว่าง/ศูนย์หรือไม่ — มักใช้แทนการเทียบกับ
// Money{} ตรงๆ เพื่อความอ่านง่าย
func (m Money) IsZero() bool {
	return m.Amount == 0
}
```

จุดสำคัญที่ควรสังเกต 3 อย่าง:

1. **ทุก method ที่ดู "เปลี่ยนค่า" (`Add`, `Multiply`) ใช้ **value receiver** (`m Money` ไม่ใช่ `m *Money`) และคืน `Money` ตัวใหม่เสมอ — ไม่มี method ไหนแก้ field ของ `m` เดิมเลย นี่คือการบังคับ immutability ด้วยรูปแบบ receiver เอง ทบทวนความแตกต่าง value vs pointer receiver จาก **Part 012**
2. **การเปรียบเทียบ `Money` สองตัวทำได้ตรงๆ ด้วย `==`** เพราะ field ทั้งคู่เป็น comparable type — ทดสอบได้ง่ายมาก: `NewMoney(99.99, "THB") == NewMoney(99.99, "THB")` ได้ `true` เสมอ โดยไม่ต้องเขียน `Equals()` เอง
3. **`Add` ปฏิเสธการบวกข้ามสกุลเงินทันที** — นี่คือตัวอย่างว่า value object ที่ดีไม่ใช่แค่ "ตัวห่อข้อมูล" (data holder) เฉยๆ แต่**บังคับกฎทางธุรกิจไว้ในตัวมันเอง** ทำให้ไม่มีทางเกิดสถานการณ์ "100 บาท + 5 ดอลลาร์ = 105" ที่ไม่มีความหมายใดๆ เลยในระบบ

---

## 5. Aggregate และ Aggregate Root: ขอบเขตของความสอดคล้อง (Consistency Boundary)

นี่คือแนวคิดที่ทรงพลังที่สุดและมักเข้าใจผิดมากที่สุดใน DDD tactical pattern

**Aggregate** คือกลุ่มของ entity และ value object ที่ **ต้องเปลี่ยนแปลงไปด้วยกันอย่างสอดคล้องกันเสมอ** (consistency boundary) — พูดง่ายๆ คือ "กลุ่มข้อมูลที่ต้องคิดเป็นก้อนเดียว ไม่แยกกันแก้ไขได้ตามใจ"

ตัวอย่าง: `Order` (คำสั่งซื้อ) กับ `OrderLine` (รายการสินค้าในคำสั่งซื้อ) ต้องเปลี่ยนแปลงไปด้วยกันเสมอ — ถ้าใครไปแก้ `OrderLine.Quantity` ตรงๆ โดยไม่ผ่าน `Order` เลย ก็อาจทำให้ `Order.Total()` ที่คำนวณไว้ก่อนหน้าไม่ตรงกับความเป็นจริงอีกต่อไป (invariant "total ต้องเท่ากับผลรวมของทุก line" ถูกละเมิด)

**Aggregate Root** คือ entity ตัวเดียวใน aggregate ที่ทำหน้าที่เป็น **"ประตูเดียว"** ที่โลกภายนอกใช้เข้าถึงและแก้ไขข้อมูลทั้งกลุ่มได้ — ในตัวอย่างข้างต้น `Order` คือ aggregate root ส่วน `OrderLine` เป็นแค่ส่วนประกอบภายในที่**ห้ามใครแก้ไขตรงๆ จากภายนอก**

กฎทองของ aggregate มี 3 ข้อที่ต้องจำ:

1. **โลกภายนอกแก้ไขได้เฉพาะผ่าน method ของ aggregate root เท่านั้น** ห้าม expose field ภายในให้แก้ไขตรงๆ
2. **Aggregate root รับผิดชอบรับประกันว่า invariant ของทั้งกลุ่มจะไม่มีวันถูกละเมิด** ไม่ว่าจะเรียก method ไหนก็ตาม
3. **Repository มีหนึ่งตัวต่อหนึ่ง aggregate root เท่านั้น** (จะอธิบายเจาะลึกในหัวข้อ 8) — ไม่มี repository แยกสำหรับ `OrderLine` เพราะมันไม่มีตัวตนที่มีความหมายถ้าแยกออกจาก `Order`

ในภาษา Go กฎข้อ 1 บังคับได้ตรงไปตรงมาด้วยกฎ export/unexported ที่เรียนมาตั้งแต่ **Part 001**: ทำให้ field ของ line เป็น **unexported** (`lines []OrderLine` ตัวเล็ก) แล้วให้เข้าถึงได้ผ่าน method สาธารณะเท่านั้น — compiler จะบังคับกฎนี้ให้เองโดยอัตโนมัติ ไม่มีทางที่โค้ดจาก package อื่นจะเขียน `order.lines = append(...)` ตรงๆ ได้เลย

---

## 6. ตัวอย่างเต็ม: `Order` Aggregate ที่ปกป้อง Invariant ของตัวเอง

มาดูตัวอย่างเต็มรูปแบบของ aggregate ที่ verify แล้วว่า compile, run และ test ผ่านจริง:

```go
package main

import (
	"errors"
	"time"
)

// OrderStatus คือ enum แบบ Go ที่ทบทวนจาก Part 003 (iota)
type OrderStatus int

const (
	StatusDraft OrderStatus = iota
	StatusSubmitted
	StatusCancelled
)

func (s OrderStatus) String() string {
	switch s {
	case StatusDraft:
		return "draft"
	case StatusSubmitted:
		return "submitted"
	case StatusCancelled:
		return "cancelled"
	default:
		return "unknown"
	}
}

// OrderLine เป็น "ส่วนประกอบ" ของ Order aggregate ไม่ใช่ entity ที่มีตัวตนอิสระ
// ในความหมายของ DDD — ไม่มีใครนอกเหนือจาก Order เข้าถึงหรือแก้ไข OrderLine
// โดยตรงได้เลย ต้องผ่าน method ของ Order เท่านั้น (นี่คือหัวใจของ "aggregate boundary")
type OrderLine struct {
	ProductID string
	Quantity  int
	UnitPrice Money
}

func (l OrderLine) Subtotal() Money {
	return l.UnitPrice.Multiply(l.Quantity)
}

var (
	ErrInvalidQuantity   = errors.New("order: quantity must be greater than zero")
	ErrOrderNotEditable  = errors.New("order: cannot modify an order that is not in draft status")
	ErrEmptyOrder        = errors.New("order: cannot submit an order with no lines")
	ErrLineNotFound      = errors.New("order: line not found")
	ErrCannotCancelOrder = errors.New("order: cannot cancel an order that is not draft or submitted")
)

// Order คือ Aggregate Root — เป็น "ประตูเดียว" ที่โลกภายนอกใช้แก้ไขข้อมูลภายใน
// aggregate (ในที่นี้คือ lines ทั้งหมด) ได้ ไม่มี method หรือ field ใดที่ยอมให้
// ใครมาแก้ไข o.lines ตรงๆ จากภายนอก package ได้เลย (สังเกตว่า field เป็นตัวเล็ก
// ทั้งหมด — ทบทวนกฎ export/unexported จาก Part 001)
//
// หน้าที่หลักของ aggregate root คือ "รับประกันว่า invariant (กฎที่ต้องเป็นจริง
// เสมอ) ของทั้งกลุ่มข้อมูลจะไม่มีวันถูกละเมิด ไม่ว่าจะเรียก method ไหนก็ตาม"
// เช่น "ห้ามมี line ที่จำนวนติดลบ", "ห้ามแก้ไข order หลัง submit ไปแล้ว"
type Order struct {
	ID         string
	CustomerID string
	Currency   string
	Status     OrderStatus

	lines  []OrderLine
	events []DomainEvent // uncommitted domain events ที่รอถูก publish
}

// NewOrder คือ "aggregate factory" — วิธีเดียวที่ถูกต้องในการสร้าง Order ใหม่
// เพื่อรับประกันว่า Order ทุกตัวที่หลุดออกจากฟังก์ชันนี้อยู่ในสถานะที่ถูกต้อง
// ตั้งแต่แรกเกิดเสมอ (ไม่มีทางได้ Order ที่ CustomerID ว่างเปล่าออกไปได้)
func NewOrder(id, customerID, currency string) (*Order, error) {
	if customerID == "" {
		return nil, errors.New("order: customerID must not be empty")
	}
	return &Order{
		ID:         id,
		CustomerID: customerID,
		Currency:   currency,
		Status:     StatusDraft,
	}, nil
}

// AddLine เพิ่มสินค้าเข้า order — enforce invariant สามข้อพร้อมกันตรงนี้ที่เดียว:
//  1. แก้ไขได้เฉพาะตอนยังเป็น draft เท่านั้น
//  2. จำนวนต้องมากกว่า 0 เสมอ
//  3. สกุลเงินของราคาต้องตรงกับสกุลเงินของ order เท่านั้น
//
// ถ้ามี ProductID เดิมอยู่แล้ว จะรวมจำนวนเข้าด้วยกันแทนที่จะสร้าง line ซ้ำ
func (o *Order) AddLine(productID string, quantity int, unitPrice Money) error {
	if o.Status != StatusDraft {
		return ErrOrderNotEditable
	}
	if quantity <= 0 {
		return ErrInvalidQuantity
	}
	if unitPrice.Currency != o.Currency {
		return ErrCurrencyMismatch
	}

	for i, line := range o.lines {
		if line.ProductID == productID {
			o.lines[i].Quantity += quantity
			return nil
		}
	}
	o.lines = append(o.lines, OrderLine{
		ProductID: productID,
		Quantity:  quantity,
		UnitPrice: unitPrice,
	})
	return nil
}

// RemoveLine ลบสินค้าออกจาก order — แก้ไขได้เฉพาะตอน draft เช่นกัน
func (o *Order) RemoveLine(productID string) error {
	if o.Status != StatusDraft {
		return ErrOrderNotEditable
	}
	for i, line := range o.lines {
		if line.ProductID == productID {
			o.lines = append(o.lines[:i], o.lines[i+1:]...)
			return nil
		}
	}
	return ErrLineNotFound
}

// Lines คืนสำเนาของ line ทั้งหมด (ไม่ใช่ slice ต้นฉบับ) เพื่อป้องกันไม่ให้ผู้เรียก
// แก้ไขข้อมูลภายใน aggregate ทางอ้อมผ่านการแก้ slice ที่ได้รับกลับไป —
// เทคนิคเดียวกับที่ memory.ProductRepository ใน Part 100 ใช้ป้องกันการรั่วไหลของ
// mutable state ออกนอก boundary
func (o *Order) Lines() []OrderLine {
	cp := make([]OrderLine, len(o.lines))
	copy(cp, o.lines)
	return cp
}

// Total คำนวณยอดรวมทั้ง order จากทุก line
func (o *Order) Total() Money {
	total := Money{Amount: 0, Currency: o.Currency}
	for _, line := range o.lines {
		total, _ = total.Add(line.Subtotal()) // ปลอดภัยเสมอ เพราะ AddLine บังคับสกุลเงินตรงกันไว้แล้ว
	}
	return total
}

// Submit ยืนยันคำสั่งซื้อ — เปลี่ยนสถานะจาก draft เป็น submitted และปล่อย
// domain event OrderSubmitted ออกมา หลังจาก submit แล้ว order จะแก้ไข line
// ใดๆ ไม่ได้อีกต่อไป (ทำได้แค่ cancel เท่านั้น)
func (o *Order) Submit() error {
	if o.Status != StatusDraft {
		return ErrOrderNotEditable
	}
	if len(o.lines) == 0 {
		return ErrEmptyOrder
	}
	o.Status = StatusSubmitted
	o.record(OrderSubmitted{
		baseEvent: baseEvent{occurredAt: time.Now()},
		OrderID:   o.ID,
		Total:     o.Total(),
	})
	return nil
}

// Cancel ยกเลิกคำสั่งซื้อ — ทำได้ทั้งตอน draft และ submitted แต่ทำซ้ำไม่ได้
func (o *Order) Cancel(reason string) error {
	if o.Status == StatusCancelled {
		return ErrCannotCancelOrder
	}
	o.Status = StatusCancelled
	o.record(OrderCancelled{
		baseEvent: baseEvent{occurredAt: time.Now()},
		OrderID:   o.ID,
		Reason:    reason,
	})
	return nil
}

// record เก็บ event ไว้ใน buffer ภายใน ไม่ publish ทันที — เป็น pattern มาตรฐาน
// ของ DDD ที่เรียกว่า "collect then dispatch": aggregate ทำหน้าที่แค่ "บันทึกว่า
// มีอะไรเกิดขึ้นบ้าง" ส่วนการ publish จริง (ไปหา event bus/message queue) เป็น
// หน้าที่ของ application layer หลังจาก repository บันทึก state ลงฐานข้อมูลสำเร็จแล้ว
// (เพื่อไม่ให้ publish event ไปแล้วแต่บันทึกฐานข้อมูลล้มเหลวทีหลัง)
func (o *Order) record(e DomainEvent) {
	o.events = append(o.events, e)
}

// PullEvents คืน event ที่สะสมไว้ทั้งหมด แล้วเคลียร์ buffer ภายใน — เรียกครั้งเดียว
// หลังบันทึก aggregate ลง repository สำเร็จ เพื่อนำไป publish ต่อ
func (o *Order) PullEvents() []DomainEvent {
	events := o.events
	o.events = nil
	return events
}
```

สังเกตจุดสำคัญ 4 อย่างในตัวอย่างนี้:

1. **`lines` และ `events` เป็น field ตัวเล็ก (unexported)** — ไม่มีทางที่โค้ดนอก package จะไปยุ่งกับมันตรงๆ ได้เลย ทุกการเปลี่ยนแปลงต้องผ่าน `AddLine`, `RemoveLine`, `Submit`, `Cancel` ที่ enforce invariant ไว้ให้เสมอ
2. **`AddLine` เช็คเงื่อนไข 3 อย่างพร้อมกัน** ก่อนจะยอมให้เพิ่ม line ได้เลย — ถ้าเงื่อนไขใดเงื่อนไขหนึ่งไม่ผ่าน จะไม่มีการแก้ไข state ใดๆ เกิดขึ้นเลย (all-or-nothing)
3. **`Submit()` เปลี่ยน `Status` และปล่อย event พร้อมกันในการเรียกเดียว** — ไม่มีทางที่ order จะ "submitted แล้วแต่ยังไม่มี event" หรือ "มี event แต่ status ยังเป็น draft" เพราะทั้งสองอย่างเกิดในฟังก์ชันเดียวกัน
4. **`Lines()` คืนสำเนา ไม่ใช่ slice ต้นฉบับ** — ถ้าคืน `o.lines` ตรงๆ ผู้เรียกจะสามารถ `order.Lines()[0].Quantity = -999` แล้วมันจะไปกระทบ slice ภายในจริงๆ (เพราะ slice คือ reference type ที่ชี้ไป array เดียวกัน — ทบทวนจาก **Part 006**) ทำลาย invariant ทั้งหมดที่ `AddLine` เพิ่งบังคับไว้

---

## 7. Domain Events: สื่อสารการเปลี่ยนแปลงโดยไม่ผูกติดกัน

**Domain Event** คือ "สิ่งที่เกิดขึ้นแล้วจริงๆ ในภาษาธุรกิจ" — สังเกตว่าตั้งชื่อเป็น**กริยารูปอดีตเสมอ** (`OrderSubmitted` ไม่ใช่ `SubmitOrder`) เพราะมันสื่อว่า "เหตุการณ์นี้เกิดขึ้นแล้ว เปลี่ยนแปลงไม่ได้อีก" ต่างจาก command ที่สื่อว่า "อยากให้เกิดอะไรบางอย่าง" (ซึ่งอาจถูกปฏิเสธได้ เช่น `Submit()` อาจ return error)

```go
package main

import "time"

// DomainEvent คือสิ่งที่ "เกิดขึ้นแล้วจริงๆ" ในภาษาธุรกิจ (ubiquitous language) —
// ตั้งชื่อเป็นกริยารูปอดีตเสมอ (OrderSubmitted ไม่ใช่ SubmitOrder) เพราะมันสื่อว่า
// "เหตุการณ์นี้เกิดขึ้นแล้ว เปลี่ยนแปลงไม่ได้อีกแล้ว" ต่างจาก command ที่สื่อว่า
// "อยากให้เกิดอะไรบางอย่าง" (ซึ่งอาจถูกปฏิเสธได้)
type DomainEvent interface {
	EventName() string
	OccurredAt() time.Time
}

// baseEvent เก็บ field ร่วมของทุก event ผ่าน struct embedding (ทบทวน Part 030)
// เพื่อไม่ต้องเขียน OccurredAt() ซ้ำในทุก event type
type baseEvent struct {
	occurredAt time.Time
}

func (b baseEvent) OccurredAt() time.Time { return b.occurredAt }

// OrderSubmitted คือ domain event ที่เกิดขึ้นเมื่อลูกค้ายืนยันคำสั่งซื้อสำเร็จ
type OrderSubmitted struct {
	baseEvent
	OrderID string
	Total   Money
}

func (e OrderSubmitted) EventName() string { return "order.submitted" }

// OrderCancelled คือ domain event ที่เกิดขึ้นเมื่อคำสั่งซื้อถูกยกเลิก
type OrderCancelled struct {
	baseEvent
	OrderID string
	Reason  string
}

func (e OrderCancelled) EventName() string { return "order.cancelled" }

// EventHandler คือ function type ธรรมดา (ทบทวนแนวคิด function-as-value จาก
// Part 008–009) — ใน Go ไม่จำเป็นต้องมี interface "IEventListener" แบบภาษา
// OOP ดั้งเดิม ฟังก์ชันตรงๆ ก็เพียงพอแล้ว
type EventHandler func(DomainEvent)

// EventBus คือ dispatcher ในหน่วยความจำล้วนๆ (in-process) — ไม่ใช่ message queue
// จริงแบบ Kafka/RabbitMQ ที่เรียนไปใน Part 091–092 แต่เป็นรูปแบบที่เพียงพอมาก
// สำหรับงานภายในแอปเดียว เช่น "ส่ง event ให้ module อื่นในโปรเซสเดียวกันรับรู้"
// โดยไม่ต้องผูก business logic หลักเข้ากับ side effect (ส่ง email, log, sync ข้อมูล)
type EventBus struct {
	handlers map[string][]EventHandler
}

func NewEventBus() *EventBus {
	return &EventBus{handlers: make(map[string][]EventHandler)}
}

func (b *EventBus) Subscribe(eventName string, handler EventHandler) {
	b.handlers[eventName] = append(b.handlers[eventName], handler)
}

// Publish ส่ง event ไปให้ทุก handler ที่ subscribe ไว้กับชื่อ event นี้
// (เวอร์ชัน production จริงมักส่งแบบ async ผ่าน channel/goroutine — ทบทวน Part 037 —
// หรือส่งออกไปยัง message queue จริงแทน แต่หลักการเรื่อง "แยก side effect ออกจาก
// aggregate" เหมือนกันทุกประการ)
func (b *EventBus) Publish(event DomainEvent) {
	for _, h := range b.handlers[event.EventName()] {
		h(event)
	}
}
```

ประโยชน์หลักของ domain event คือ **แยกกฎทางธุรกิจแกนกลาง (aggregate) ออกจาก side effect ที่ตามมา** ให้เห็นภาพชัด: `Order` aggregate เอง**ไม่รู้จัก**คำว่า "อีเมล" เลยแม้แต่น้อย มันแค่รู้ว่า "ตอนนี้มีคำสั่งซื้อถูก submit แล้ว" ส่วนใครจะทำอะไรต่อ (ส่งอีเมลยืนยัน, ตัดสต็อกใน service อื่น, sync ข้อมูลไป analytics) เป็นเรื่องของ subscriber แต่ละตัวที่ผูกไว้ทีหลัง — ต่อไปถ้าอยากเพิ่ม subscriber ใหม่ (เช่น ส่ง SMS แจ้งเตือน) ก็แค่เพิ่ม `bus.Subscribe(...)` อีกบรรทัดเดียว โดยไม่ต้องแก้โค้ดใน `Order` เลย

---

## 8. Repository แบบ DDD: หนึ่งตัวต่อหนึ่ง Aggregate Root เท่านั้น

ทบทวน pattern repository จาก **Part 100** — DDD เพิ่มกฎที่ชัดเจนกว่านั้นหนึ่งข้อ: **repository มีหนึ่งตัวต่อหนึ่ง aggregate root เท่านั้น ไม่ใช่หนึ่งตัวต่อหนึ่งตาราง**

```go
package main

import (
	"errors"
	"sync"
)

// OrderRepository คือ repository แบบ "DDD flavor" ของ pattern เดียวกันกับ
// Part 100: มันทำงานกับ Order ทั้งก้อนในฐานะ Aggregate Root เท่านั้น —
// ไม่มี method ระดับ "SaveLine" หรือ "FindLineByProductID" ให้เข้าถึง OrderLine
// แยกจาก Order เลย เพราะ OrderLine ไม่มีตัวตนที่มีความหมายอะไรถ้าแยกออกจาก
// Order ที่มันสังกัดอยู่ — นี่คือกฎทองของ DDD repository: "repository มีหนึ่งตัว
// ต่อหนึ่ง aggregate root เท่านั้น ไม่ใช่หนึ่งตัวต่อหนึ่งตาราง"
type OrderRepository interface {
	Save(order *Order) error
	FindByID(id string) (*Order, error)
}

var ErrOrderNotFound = errors.New("order: not found")

// InMemoryOrderRepository เป็น implementation ง่ายๆ สำหรับสาธิตและ test
// (ของจริงในโปรเจกต์ production จะเป็น PostgreSQL/MongoDB repository แบบเดียวกับ
// ที่โชว์ใน Part 100 — สลับกันได้ทันทีเพราะ OrderService ผูกกับ interface เท่านั้น)
type InMemoryOrderRepository struct {
	mu   sync.Mutex
	data map[string]*Order
}

func NewInMemoryOrderRepository() *InMemoryOrderRepository {
	return &InMemoryOrderRepository{data: make(map[string]*Order)}
}

func (r *InMemoryOrderRepository) Save(order *Order) error {
	r.mu.Lock()
	defer r.mu.Unlock()
	r.data[order.ID] = order
	return nil
}

func (r *InMemoryOrderRepository) FindByID(id string) (*Order, error) {
	r.mu.Lock()
	defer r.mu.Unlock()
	o, ok := r.data[id]
	if !ok {
		return nil, ErrOrderNotFound
	}
	return o, nil
}

var _ OrderRepository = (*InMemoryOrderRepository)(nil)
```

แม้ในฐานข้อมูลจริง `Order` กับ `OrderLine` มักถูกเก็บเป็นสองตารางแยกกัน (`orders` และ `order_lines` เชื่อมด้วย foreign key) แต่ในระดับ **domain model** สิ่งนี้ควรถูกซ่อนไว้เบื้องหลัง `OrderRepository` ทั้งหมด — เวลาเรียก `Save(order)` การ implementation จริงจะ `INSERT`/`UPDATE` ทั้งตาราง `orders` และลบ-แทรก `order_lines` ใหม่ทั้งหมดในหนึ่ง database transaction (ทบทวนแนวคิด transaction จาก **Part 078**) เพื่อให้การบันทึกทั้ง aggregate เกิดขึ้นแบบ atomic — ผู้เรียก `OrderRepository.Save()` ไม่จำเป็นต้องรู้รายละเอียดนี้เลย

---

## 9. Bounded Context: ทำไมคำว่า "Order" ในสองทีมถึงไม่ใช่สิ่งเดียวกัน

**Bounded Context** คือ **strategic pattern** ที่สำคัญที่สุดของ DDD — มันบอกว่า **คำศัพท์เดียวกันสามารถมีความหมายต่างกันได้ในแต่ละส่วนของระบบ** และนั่นคือเรื่องปกติที่ยอมรับได้ ไม่ใช่ความผิดพลาด

ตัวอย่างที่ชัดเจน: คำว่า **"Order"** ในบริบทของทีมต่างๆ:

- **ทีมขาย (Sales context)**: `Order` คือ รายการสินค้า + ราคา + ลูกค้า + สถานะการชำระเงิน
- **ทีมคลังสินค้า (Warehouse context)**: `Order` คือ รายการที่ต้องหยิบจากชั้นวาง + ตำแหน่งจัดเก็บ + สถานะการแพ็ค (ไม่สนใจราคาเลย)
- **ทีมขนส่ง (Shipping context)**: `Order` คือ พัสดุที่ต้องส่ง + ที่อยู่ปลายทาง + tracking number (ไม่สนใจว่ามีสินค้าอะไรอยู่ข้างในเลย)

ทั้งสามทีมพูดคำว่า "Order" เหมือนกัน แต่จริงๆ แล้วกำลังพูดถึง **model คนละตัวที่มีเหตุผลในบริบทของตัวเองครบถ้วน** — DDD บอกว่านี่คือเรื่องที่**ควรยอมรับ** ไม่ใช่พยายามสร้าง `Order` struct ตัวเดียวที่มี field ทุกอย่างของทั้งสามทีมรวมกัน (ซึ่งจะกลายเป็น "god object" ที่ทุกทีมแก้ไขกระทบกันหมด)

นี่คือเหตุผลที่ **Bounded Context เชื่อมโยงโดยตรงกับ Service Boundaries ที่เรียนไปใน Part 088** — เมื่อแบ่ง microservices ควรแบ่งตาม bounded context ไม่ใช่แบ่งตามตาราง database หรือแบ่งตาม CRUD operation ทีมขาย, ทีมคลังสินค้า, ทีมขนส่งควรมี `Order` model เป็นของตัวเอง อยู่คนละ service กัน สื่อสารกันผ่าน API หรือ event (เช่น `OrderSubmitted` ที่เรียนในหัวข้อ 7) แทนที่จะแชร์ตารางฐานข้อมูลเดียวกันตรงๆ — ทบทวนคำแนะนำเรื่อง "Service Boundaries: จะแบ่ง Service อย่างไรไม่ให้พัง" ใน **Part 088 หัวข้อ 5** ซึ่งพรีวิวแนวคิดนี้ไว้แล้ว

ในระดับโค้ด Go เพียงอย่างเดียว (ไม่ถึงกับแยก microservices) bounded context แปลเป็น: **แต่ละ context ควรมี Go module หรืออย่างน้อย package ของตัวเองที่แยกกันชัดเจน** เช่น `internal/sales/order.go` กับ `internal/warehouse/order.go` เป็นคนละ `Order` struct กันโดยสิ้นเชิง ไม่พยายามใช้ struct ร่วมกัน — แม้จะดู "ซ้ำซ้อน" ในสายตาแรก แต่มันป้องกันปัญหาที่ทีมหนึ่งแก้ field แล้วไปพังอีกทีมโดยไม่ตั้งใจ

---

## 10. ตัวอย่างเต็มแบบรวม: Application Service ที่ประสาน Aggregate + Event Bus

มาดูการนำทุกอย่างมารวมกัน — **Application Service** (เทียบเท่ากับ usecase layer ของ Clean Architecture ใน Part 100) ที่ทำหน้าที่ "orchestrate" ระหว่าง repository, aggregate, และ event bus:

```go
package main

import "fmt"

// OrderApplicationService คือชั้น "application service" ของ DDD (เทียบเท่ากับ
// usecase layer ของ Clean Architecture ใน Part 100) — หน้าที่ของมันคือ orchestrate
// ระหว่าง repository กับ aggregate กับ event bus โดยไม่มี business rule ของตัวเอง
// เลย (business rule ทั้งหมดอยู่ใน Order aggregate) มันแค่ "เรียกให้ถูกลำดับ"
type OrderApplicationService struct {
	repo OrderRepository
	bus  *EventBus
}

func NewOrderApplicationService(repo OrderRepository, bus *EventBus) *OrderApplicationService {
	return &OrderApplicationService{repo: repo, bus: bus}
}

// SubmitOrder ดึง aggregate ขึ้นมา เรียก method ทางธุรกิจของมัน บันทึกกลับ
// แล้วค่อย publish event ที่สะสมไว้ — นี่คือลำดับ "collect then dispatch" ที่
// พูดถึงใน order.go: publish event ทีหลังจากบันทึกสำเร็จเท่านั้น ไม่ใช่ก่อนหน้า
func (s *OrderApplicationService) SubmitOrder(orderID string) error {
	order, err := s.repo.FindByID(orderID)
	if err != nil {
		return fmt.Errorf("submit order: %w", err)
	}

	if err := order.Submit(); err != nil {
		return fmt.Errorf("submit order: %w", err)
	}

	if err := s.repo.Save(order); err != nil {
		return fmt.Errorf("submit order: %w", err)
	}

	for _, event := range order.PullEvents() {
		s.bus.Publish(event)
	}
	return nil
}

// CancelOrder ทำงานตามรูปแบบเดียวกันกับ SubmitOrder เป๊ะ
func (s *OrderApplicationService) CancelOrder(orderID, reason string) error {
	order, err := s.repo.FindByID(orderID)
	if err != nil {
		return fmt.Errorf("cancel order: %w", err)
	}

	if err := order.Cancel(reason); err != nil {
		return fmt.Errorf("cancel order: %w", err)
	}

	if err := s.repo.Save(order); err != nil {
		return fmt.Errorf("cancel order: %w", err)
	}

	for _, event := range order.PullEvents() {
		s.bus.Publish(event)
	}
	return nil
}
```

และนำทุกชิ้นมาประกอบกันใน `main()`:

```go
package main

import "fmt"

func main() {
	repo := NewInMemoryOrderRepository()
	bus := NewEventBus()

	// Subscriber จำลอง: เมื่อ order ถูก submit ให้ "ส่งอีเมลยืนยัน" (แค่ print
	// จำลอง) — จุดสำคัญคือ Order aggregate ไม่รู้จัก concept "อีเมล" เลยแม้แต่น้อย
	// business logic หลัก (invariant ของ order) กับ side effect (แจ้งเตือน) ถูก
	// แยกออกจากกันอย่างสมบูรณ์ผ่าน domain event
	bus.Subscribe("order.submitted", func(e DomainEvent) {
		evt := e.(OrderSubmitted)
		fmt.Printf("[email service] ส่งอีเมลยืนยันคำสั่งซื้อ %s ยอดรวม %s\n", evt.OrderID, evt.Total)
	})
	bus.Subscribe("order.submitted", func(e DomainEvent) {
		evt := e.(OrderSubmitted)
		fmt.Printf("[inventory service] ตัดสต็อกสำหรับคำสั่งซื้อ %s\n", evt.OrderID)
	})
	bus.Subscribe("order.cancelled", func(e DomainEvent) {
		evt := e.(OrderCancelled)
		fmt.Printf("[refund service] เริ่มกระบวนการคืนเงินสำหรับคำสั่งซื้อ %s (เหตุผล: %s)\n", evt.OrderID, evt.Reason)
	})

	svc := NewOrderApplicationService(repo, bus)

	fmt.Println("=== 1. สร้าง order ใหม่และเพิ่มสินค้า ===")
	order, err := NewOrder("order-1", "customer-42", "THB")
	if err != nil {
		panic(err)
	}
	if err := order.AddLine("sku-keyboard", 1, NewMoney(2500, "THB")); err != nil {
		panic(err)
	}
	if err := order.AddLine("sku-mouse", 2, NewMoney(590, "THB")); err != nil {
		panic(err)
	}
	repo.Save(order)
	fmt.Printf("ยอดรวมก่อน submit: %s (สถานะ: %s)\n\n", order.Total(), order.Status)

	fmt.Println("=== 2. พยายามเพิ่มสินค้าด้วยสกุลเงินผิด (คาดหวัง error) ===")
	err = order.AddLine("sku-cable", 1, NewMoney(199, "USD"))
	fmt.Printf("ผลลัพธ์: %v\n\n", err)

	fmt.Println("=== 3. พยายามเพิ่มสินค้าจำนวนติดลบ (คาดหวัง error) ===")
	err = order.AddLine("sku-cable", -3, NewMoney(199, "THB"))
	fmt.Printf("ผลลัพธ์: %v\n\n", err)

	fmt.Println("=== 4. Submit order (จะ publish domain event ออกไป) ===")
	if err := svc.SubmitOrder("order-1"); err != nil {
		panic(err)
	}
	fmt.Println()

	fmt.Println("=== 5. พยายามแก้ไข order หลัง submit ไปแล้ว (คาดหวัง error) ===")
	err = order.AddLine("sku-cable", 1, NewMoney(199, "THB"))
	fmt.Printf("ผลลัพธ์: %v\n\n", err)

	fmt.Println("=== 6. สร้าง order เปล่าแล้วพยายาม submit ทันที (คาดหวัง error) ===")
	emptyOrder, _ := NewOrder("order-2", "customer-7", "THB")
	repo.Save(emptyOrder)
	err = svc.SubmitOrder("order-2")
	fmt.Printf("ผลลัพธ์: %v\n\n", err)

	fmt.Println("=== 7. ยกเลิก order-2 แทน (จะ publish OrderCancelled) ===")
	if err := svc.CancelOrder("order-2", "ลูกค้าเปลี่ยนใจ"); err != nil {
		panic(err)
	}
}
```

รันด้วย `go run .` ผลลัพธ์จริง:

```
=== 1. สร้าง order ใหม่และเพิ่มสินค้า ===
ยอดรวมก่อน submit: 3680.00 THB (สถานะ: draft)

=== 2. พยายามเพิ่มสินค้าด้วยสกุลเงินผิด (คาดหวัง error) ===
ผลลัพธ์: money: currency mismatch

=== 3. พยายามเพิ่มสินค้าจำนวนติดลบ (คาดหวัง error) ===
ผลลัพธ์: order: quantity must be greater than zero

=== 4. Submit order (จะ publish domain event ออกไป) ===
[email service] ส่งอีเมลยืนยันคำสั่งซื้อ order-1 ยอดรวม 3680.00 THB
[inventory service] ตัดสต็อกสำหรับคำสั่งซื้อ order-1

=== 5. พยายามแก้ไข order หลัง submit ไปแล้ว (คาดหวัง error) ===
ผลลัพธ์: order: cannot modify an order that is not in draft status

=== 6. สร้าง order เปล่าแล้วพยายาม submit ทันที (คาดหวัง error) ===
ผลลัพธ์: submit order: order: cannot submit an order with no lines

=== 7. ยกเลิก order-2 แทน (จะ publish OrderCancelled) ===
[refund service] เริ่มกระบวนการคืนเงินสำหรับคำสั่งซื้อ order-2 (เหตุผล: ลูกค้าเปลี่ยนใจ)
```

สังเกตข้อ 4 และ 5: `AddLine` ที่เรียกตรงๆ กับ `order` (ไม่ผ่าน service) ในข้อ 5 ยังคงถูกปฏิเสธถูกต้อง แม้ `order` ตัวแปรใน `main()` จะเป็น pointer ตัวเดียวกับที่อยู่ใน repository ก็ตาม — เพราะ invariant ถูก enforce อยู่ **ข้างใน aggregate เอง** ไม่ใช่แค่ที่ชั้น application service เท่านั้น นี่คือจุดสำคัญที่สุดของแนวคิด aggregate: ไม่ว่าจะเรียกจากที่ไหน (ผ่าน service, ผ่าน test, ผ่าน code อื่นในอนาคต) กฎจะยังคงถูกบังคับใช้เสมอ

ตัวอย่างนี้ยังมี table-driven test ที่ยืนยัน invariant ทั้งหมดของ `Order` และ `Money` (ทบทวนรูปแบบจาก **Part 034**) โดยทดสอบผ่าน public method ล้วนๆ — verify แล้วว่าทุก test ผ่านด้วย `go test ./...`:

```
ok  	ddddemo	0.002s
```

---

## 11. ข้อควรระวังตรงไปตรงมา: DDD ไม่เหมาะกับ CRUD ธรรมดา

Eric Evans เองก็เคยเตือนไว้ในหนังสือต้นฉบับว่า DDD **ไม่ใช่เครื่องมือสารพัดประโยชน์ที่ควรใช้กับทุกโปรเจกต์** — มาดูข้อควรระวังที่ตรงไปตรงมา:

### 11.1 CRUD ธรรมดาไม่มี invariant ให้ปกป้อง

ถ้าระบบของคุณคือ "แบบฟอร์มบันทึกข้อมูลติดต่อลูกค้า" ที่แค่ create/read/update/delete ตรงๆ โดยไม่มีกฎทางธุรกิจซับซ้อนอะไรเลย (ไม่มีเงื่อนไขว่า "ทำได้เมื่อไหร่", ไม่มี state machine, ไม่มีการคำนวณที่ต้องสอดคล้องกันข้ามหลาย field) การสร้าง Aggregate, Domain Event, Value Object เต็มรูปแบบ**เป็นการเสียเวลาโดยไม่มีประโยชน์ใดๆ กลับมา** — CRUD แบบ Clean Architecture ธรรมดา (Part 100) ก็เพียงพอแล้ว หรือบางกรณีอาจไม่ต้องแยกชั้นอะไรเลยด้วยซ้ำ

### 11.2 Tactical Pattern มีต้นทุนสูงกว่าที่คิด

การเขียน `Order` aggregate แบบเต็มรูปแบบในบทนี้ใช้เวลามากกว่า struct ธรรมดาที่มี public field ล้วนๆ หลายเท่า — ต้นทุนนี้**คุ้มค่าก็ต่อเมื่อ** ความซับซ้อนทางธุรกิจสูงพอที่จะทำให้เกิดบั๊กร้ายแรงถ้าไม่มี invariant ป้องกัน (เช่น ระบบการเงิน, ระบบจอง, ระบบคำนวณค่าคอมมิชชั่นที่ซับซ้อน) สำหรับระบบที่ไม่มีความเสี่ยงแบบนี้ ต้นทุนนี้อาจไม่คุ้มเลย

### 11.3 ทีมต้องเข้าใจ Domain จริงๆ ก่อนเริ่มออกแบบ

DDD กำหนดให้นักพัฒนาต้อง**พูดคุยกับ domain expert อย่างต่อเนื่อง** เพื่อให้ Ubiquitous Language และ Aggregate boundary ถูกต้อง — ถ้าทีมไม่มีโอกาสเข้าถึง domain expert จริง (เช่น ทำ SaaS ทั่วไปที่ requirement มาจาก assumption ของทีม product เอง) การพยายามทำ DDD เต็มรูปแบบอาจกลายเป็นการสร้าง aggregate boundary ที่ผิดตั้งแต่ต้น ซึ่งแก้ไขทีหลังยากกว่าการไม่แบ่งอะไรเลยตั้งแต่แรก

### บทสรุปเชิงปฏิบัติ

แนวทางที่สมเหตุสมผลที่สุดคือ: **ใช้แนวคิด Ubiquitous Language และ Bounded Context กับแทบทุกโปรเจกต์** (เพราะต้นทุนต่ำ แค่ตั้งชื่อให้ตรงกับธุรกิจ และคิดเรื่องขอบเขต service ให้ดี) แต่ **สงวน tactical pattern หนักๆ (Aggregate, Domain Event แบบเต็มรูปแบบ) ไว้เฉพาะส่วนของระบบที่ธุรกิจซับซ้อนจริงๆ เท่านั้น** — ระบบใหญ่หนึ่งระบบอาจมีบางส่วนที่ควรทำ DDD เต็มรูปแบบ (เช่น โมดูลคำนวณราคา/ส่วนลด) ในขณะที่อีกหลายส่วน (เช่น โมดูลจัดการโปรไฟล์ผู้ใช้) ใช้ CRUD ธรรมดาก็เพียงพอแล้ว — นี่คือหลักการเดียวกับ "ไม่ต้องใช้ Clean Architecture ทุกที่" ที่กล่าวไปใน **Part 100 หัวข้อ 10**

---

## สรุปสิ่งที่ได้เรียนในบทนี้

- **DDD** แก้ปัญหาช่องว่างระหว่างความเข้าใจของ domain expert กับสิ่งที่นักพัฒนาเขียนเป็นโค้ด ไม่ใช่แค่เรื่องตั้งชื่อให้เท่
- **Ubiquitous Language**: คำศัพท์ในโค้ดต้องตรงกับคำที่ domain expert ใช้พูดจริง ทุกตัวอักษร
- **Entity** มี identity เปรียบเทียบด้วย ID ส่วน **Value Object** ไม่มี identity เปรียบเทียบด้วยค่า และควร immutable เสมอ — Go เอื้อต่อ Value Object เป็นพิเศษเพราะ struct comparison มาให้ฟรีผ่าน `==`
- **`Money`** คือตัวอย่างคลาสสิกของ Value Object: เก็บเป็นหน่วยเล็กสุด (int64) ไม่ใช้ float, ทุก method คืนค่าใหม่แทนการแก้ไขของเดิม, ปฏิเสธการดำเนินการข้ามสกุลเงินตั้งแต่ตัว value object เอง
- **Aggregate** คือกลุ่มข้อมูลที่ต้องเปลี่ยนแปลงสอดคล้องกันเสมอ **Aggregate Root** คือประตูเดียวที่แก้ไขข้อมูลทั้งกลุ่มได้ — field ภายในต้อง unexported เพื่อบังคับ boundary นี้ด้วยกฎ export ของ Go เอง
- ตัวอย่าง `Order` aggregate แสดง invariant 3 ชั้น (สกุลเงินตรงกัน, จำนวนเป็นบวก, แก้ไขได้เฉพาะตอน draft) ที่ enforce อยู่ข้างในตัว aggregate เอง ไม่ว่าจะเรียกจากที่ไหนก็ตาม
- **Domain Event** ตั้งชื่อเป็นกริยารูปอดีต (`OrderSubmitted`) แยก business logic แกนกลางออกจาก side effect ผ่าน pattern "collect then dispatch"
- **Repository แบบ DDD** มีหนึ่งตัวต่อหนึ่ง aggregate root เท่านั้น ไม่ใช่หนึ่งตัวต่อหนึ่งตาราง
- **Bounded Context**: คำศัพท์เดียวกัน (เช่น "Order") มีความหมายต่างกันได้ในแต่ละส่วนของระบบ เชื่อมโยงตรงกับ Service Boundaries ที่เรียนใน Part 088
- **ข้อควรระวังตรงไปตรงมา**: tactical pattern ของ DDD มีต้นทุนสูง คุ้มค่าเฉพาะกับ domain ที่ซับซ้อนจริง ไม่เหมาะกับ CRUD ธรรมดา

## แบบฝึกหัดท้ายบท

1. ทำตามตัวอย่างในบทนี้ให้ครบทุกไฟล์ แล้วรัน `go run .` และ `go test ./...` ด้วยตัวเอง ยืนยันว่าได้ output ตรงกับที่แสดงในบทเรียน
2. เพิ่ม value object ใหม่ชื่อ `Address` (มี field `Street`, `City`, `PostalCode` ทั้งหมดเป็น string) แล้วเพิ่ม field `ShippingAddress Address` เข้าไปใน `Order` — เขียน test ยืนยันว่า `Address` สองตัวที่ field เหมือนกันทุกอย่างเปรียบเทียบด้วย `==` แล้วได้ `true`
3. เพิ่ม invariant ใหม่ใน `Order`: ห้าม submit order ที่ยอดรวม (`Total()`) เกิน 100,000 บาท (สมมติเป็นกฎ "ต้องอนุมัติพิเศษก่อน") เพิ่ม sentinel error `ErrOrderExceedsLimit` และเขียน test ยืนยันว่ากฎนี้ทำงานถูกต้อง
4. เพิ่ม domain event ใหม่ชื่อ `OrderLineAdded` ที่ถูก publish ทุกครั้งที่เรียก `AddLine` สำเร็จ (ต้องปรับ `AddLine` ให้ return error ได้เหมือนเดิม แต่เพิ่มการ record event ด้วย) แล้ว subscribe handler ที่พิมพ์ log ทุกครั้งที่มีการเพิ่มสินค้า
5. ออกแบบ (เขียนเป็น comment หรือ diagram ง่ายๆ ไม่ต้องเขียนโค้ดเต็ม) ว่าถ้าจะแบ่ง bounded context สำหรับระบบ "แพลตฟอร์มจองที่พัก" (คล้าย Airbnb) ออกเป็นกี่ context อะไรบ้าง (เช่น Booking, Payment, Listing, Review) และคำว่า "Property" (ที่พัก) ในแต่ละ context ควรมีข้อมูลอะไรต่างกันบ้าง
6. อภิปราย (เตรียมคำตอบไว้): ยกตัวอย่างระบบที่คุณเคยทำงานด้วยหรือรู้จัก แล้ววิเคราะห์ว่าส่วนไหนของระบบนั้น "ควร" ทำ DDD tactical pattern เต็มรูปแบบ และส่วนไหน "ไม่ควร" พร้อมให้เหตุผลอ้างอิงจากหัวข้อ 11

---

**ต่อไป**: [Part 102 — Design Patterns ใน Go](./102-design-patterns.md)
