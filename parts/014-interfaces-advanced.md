# Part 014: Interfaces ขั้นสูง: type assertion, type switch, empty interface

> ภาคที่ 1: พื้นฐานภาษา Go (Fundamentals) — ตอนที่ 14 จาก 15

## สารบัญของบทนี้

1. ทบทวน: ทำไมต้อง "เปิดดู" concrete type ข้างใน interface
2. Type Assertion แบบปลอดภัย: `v, ok := x.(T)`
3. Type Assertion แบบ Panic: `v := x.(T)`
4. Type Switch: `switch v := x.(type)`
5. Empty Interface: `interface{}` และ `any`
6. เมื่อไหร่ควรใช้ Empty Interface เมื่อไหร่คือ Anti-Pattern
7. Interface Embedding: ประกอบ Interface ใหญ่จาก Interface เล็ก
8. หลักการ "Accept Interfaces, Return Structs"
9. ข้อผิดพลาดที่พบบ่อย: Over-Abstraction ด้วย Interface เร็วเกินไป
10. สรุปสิ่งที่ได้เรียนในบทนี้
11. แบบฝึกหัดท้ายบท

---

## 1. ทบทวน: ทำไมต้อง "เปิดดู" concrete type ข้างใน interface

จาก Part 013 เราเรียนไปแล้วว่า interface value คือคู่ `(type, value)` เสมอ ตัวแปร interface ทำให้เราเขียนโค้ดที่ทำงานกับหลาย concrete type ได้อย่างเป็นนามธรรม — แต่บางครั้งเราก็ต้องการ **"เปิดดู"** ว่า concrete type ที่ซ่อนอยู่ข้างในตอนนั้นคืออะไรกันแน่ เพื่อเข้าถึง field หรือ method เฉพาะของ type นั้นที่ไม่ได้อยู่ใน interface

บทนี้จะพูดถึงกลไก 3 อย่างที่ Go มีให้สำหรับทำเรื่องนี้: **type assertion**, **type switch**, และแนวคิดเรื่อง **empty interface** ที่เก็บอะไรก็ได้ รวมถึงหลักการออกแบบ interface ที่ดีในหัวข้อท้ายบท

---

## 2. Type Assertion แบบปลอดภัย: `v, ok := x.(T)`

**Type assertion** คือ syntax สำหรับ "ถาม" ว่าค่าที่เก็บอยู่ใน interface มี concrete type ตรงกับ `T` ที่เราสงสัยหรือไม่ รูปแบบที่ปลอดภัยที่สุด (แนะนำให้ใช้เป็นหลัก) คือรูปแบบ **comma-ok**:

```go
v, ok := x.(T)
```

- ถ้า `x` เก็บค่าที่มี concrete type ตรงกับ `T` จริง → `v` จะได้ค่านั้น (แปลงเป็น type `T`) และ `ok` เป็น `true`
- ถ้าไม่ตรง → `v` จะได้ **zero value ของ `T`** และ `ok` เป็น `false` **โดยไม่ panic**

```go
package main

import "fmt"

type Shape interface {
	Area() float64
}

type Circle struct{ Radius float64 }

func (c Circle) Area() float64 { return 3.14159 * c.Radius * c.Radius }

type Square struct{ Side float64 }

func (s Square) Area() float64 { return s.Side * s.Side }

func main() {
	var s Shape = Circle{Radius: 5}

	sq, ok := s.(Square) // comma-ok form: ไม่ panic ต่อให้ผิดชนิด
	fmt.Println("ok =", ok, ", sq =", sq)

	c, ok := s.(Circle) // ชนิดตรง จะได้ ok = true
	fmt.Println("ok =", ok, ", c =", c)
}
```

ผลลัพธ์:

```
ok = false , sq = {0}
ok = true , c = {5}
```

สังเกตว่าตอน `ok` เป็น `false` ค่า `sq` ที่ได้คือ `{0}` ซึ่งคือ zero value ของ `Square` (ไม่ใช่ค่าขยะหรือ error ใดๆ) — Go ออกแบบให้ปลอดภัยเสมอในรูปแบบนี้

### รูปแบบที่ใช้บ่อยที่สุดในโค้ดจริง: เช็ค optional interface

รูปแบบที่พบบ่อยมากคือการเช็คว่า concrete type หนึ่ง "มี" ความสามารถเสริม (implement interface อีกตัวหนึ่งเพิ่มเติม) หรือไม่ ก่อนจะเรียกใช้ความสามารถนั้น:

```go
type Colorer interface {
	Color() string
}

func describe(s Shape) {
	fmt.Printf("Area = %.2f", s.Area())

	// เช็คว่า s มี method Color() เพิ่มเติมหรือไม่ (อาจมีหรือไม่มีก็ได้)
	if c, ok := s.(Colorer); ok {
		fmt.Printf(", Color = %s", c.Color())
	}
	fmt.Println()
}
```

รูปแบบนี้ใช้บ่อยมากใน standard library เช่นตอนที่ `encoding/json` เช็คว่า type หนึ่งมี method `MarshalJSON()` (implement `json.Marshaler`) เป็นพิเศษหรือไม่ ก่อนจะเลือกวิธี encode ข้อมูล

---

## 3. Type Assertion แบบ Panic: `v := x.(T)`

ถ้าเขียน type assertion โดย**ไม่รับค่า `ok`** ตัวที่สอง จะได้ syntax แบบที่เรียกว่า **panicking form**:

```go
v := x.(T)
```

รูปแบบนี้ถ้า concrete type ไม่ตรงกับ `T` โปรแกรมจะ **panic ทันที** (เรื่อง panic แบบเต็มจะเรียนใน Part 017 บทนี้ขอพูดแค่ผลลัพธ์ที่เกิดขึ้น):

```go
package main

import "fmt"

type Shape interface {
	Area() float64
}

type Circle struct{ Radius float64 }

func (c Circle) Area() float64 { return 3.14159 * c.Radius * c.Radius }

type Square struct{ Side float64 }

func (s Square) Area() float64 { return s.Side * s.Side }

func main() {
	var s Shape = Circle{Radius: 5}
	sq := s.(Square) // panicking form: ชนิดไม่ตรง -> panic ทันที
	fmt.Println(sq)
}
```

รันแล้วจะได้:

```
panic: interface conversion: main.Shape is main.Circle, not main.Square

goroutine 1 [running]:
main.main()
	/path/to/main.go:19 +0x28
exit status 2
```

**ควรใช้รูปแบบ panicking form เมื่อไหร่?** เมื่อเรา**มั่นใจ 100%** จาก logic ของโปรแกรมว่า type ต้องตรงแน่ๆ และถ้ามันไม่ตรงจริงๆ ถือว่าเป็นบั๊กร้ายแรงในโปรแกรมที่ควรทำให้โปรแกรม crash ทันทีเพื่อให้เจอบั๊กเร็วที่สุด (fail fast) มากกว่าจะปล่อยให้โปรแกรมทำงานต่อด้วยสภาพที่ผิดเพี้ยน แต่ **ในกรณีทั่วไป ให้ใช้ comma-ok form เป็นค่าเริ่มต้นเสมอ** โดยเฉพาะเมื่อ type ของค่าที่ได้รับมาไม่แน่นอน (เช่น รับมาจากภายนอกโปรแกรม จาก user input หรือจาก package อื่น)

---

## 4. Type Switch: `switch v := x.(type)`

เมื่อต้องเช็คกับหลาย type พร้อมกัน การเขียน `if`-`else` ต่อกันด้วย type assertion หลายรอบจะดูรกและอ่านยาก Go จึงมี syntax พิเศษเรียกว่า **type switch**:

```go
switch v := x.(type) {
case TypeA:
    // v มี type เป็น TypeA แล้วในบล็อกนี้
case TypeB:
    // v มี type เป็น TypeB แล้วในบล็อกนี้
default:
    // ไม่ตรงกับ case ไหนเลย v ยังเป็น type เดิม (interface type)
}
```

สังเกต keyword พิเศษ `.(type)` — ใช้ได้เฉพาะในโครงสร้าง `switch` เท่านั้น ไม่ใช่ type assertion ปกติ

```go
package main

import "fmt"

type Shape interface {
	Area() float64
}

type Circle struct{ Radius float64 }

func (c Circle) Area() float64 { return 3.14159 * c.Radius * c.Radius }

type Square struct{ Side float64 }

func (s Square) Area() float64 { return s.Side * s.Side }

func classify(s Shape) string {
	switch v := s.(type) {
	case Circle:
		return fmt.Sprintf("Circle รัศมี %.1f", v.Radius)
	case Square:
		return fmt.Sprintf("Square ด้าน %.1f", v.Side)
	case nil:
		return "ไม่มีค่า (nil interface)"
	default:
		return fmt.Sprintf("ไม่รู้จักชนิด %T", v)
	}
}

func main() {
	fmt.Println(classify(Circle{Radius: 2}))
	fmt.Println(classify(Square{Side: 3}))
	fmt.Println(classify(nil))
}
```

ผลลัพธ์:

```
Circle รัศมี 2.0
Square ด้าน 3.0
ไม่มีค่า (nil interface)
```

ข้อสังเกตสำคัญเกี่ยวกับ type switch:

- แต่ละ `case` สามารถระบุได้หลาย type พร้อมกันด้วย comma เช่น `case Circle, Square:` — แต่ในกรณีนั้น `v` ภายในบล็อกจะยังคงเป็น interface type เดิม (ไม่ได้ narrow ลงมาเป็น type เดียว) เพราะ Go ไม่รู้ว่าเป็นตัวไหนกันแน่ในบรรดาหลาย type ที่ระบุ
- `case nil:` เอาไว้ดักกรณีที่ interface เป็น nil จริงๆ ตามที่เรียนไปใน Part 013
- `default` ทำงานเมื่อไม่ตรงกับ case ไหนเลย ในบล็อกนี้ `v` จะยังมี type เป็น interface type ตั้งต้น (`Shape` ในตัวอย่างนี้)
- Type switch ไม่มีรูปแบบ panic เหมือน type assertion เดี่ยวๆ — ถ้าไม่ตรงกับ case ไหนเลยและไม่มี `default` ก็แค่ไม่ทำอะไรเลย ไม่ panic

---

## 5. Empty Interface: `interface{}` และ `any`

**Empty interface** คือ interface ที่**ไม่มี method ใดๆ เลย**:

```go
var x interface{}
```

เนื่องจากไม่มี method สักตัว **ทุก type ในภาษา Go จึง implement empty interface โดยอัตโนมัติเสมอ** (เพราะเงื่อนไข "มี method ครบตามที่กำหนด" เป็นจริงเสมอเมื่อไม่มี method ให้ต้องมี) ทำให้ตัวแปร `interface{}` เก็บค่าอะไรก็ได้ในโลก

ตั้งแต่ **Go 1.18** เป็นต้นมา มีการเพิ่ม alias ชื่อ **`any`** ซึ่งเป็นแค่ชื่อเรียกอื่นของ `interface{}` เป๊ะๆ (ประกาศไว้ใน builtin ว่า `type any = interface{}`) เพื่อให้โค้ดอ่านง่ายขึ้น ปัจจุบันโค้ด Go สมัยใหม่นิยมเขียน `any` มากกว่า `interface{}` เกือบทั้งหมด

```go
package main

import "fmt"

// PrintAny รับค่าอะไรก็ได้ผ่าน empty interface (any คือ alias ของ interface{} ตั้งแต่ Go 1.18)
func PrintAny(v any) {
	switch x := v.(type) {
	case int:
		fmt.Println("เป็น int:", x*2)
	case string:
		fmt.Println("เป็น string:", x+x)
	case bool:
		fmt.Println("เป็น bool:", !x)
	default:
		fmt.Printf("ไม่รู้จักชนิด %T: %v\n", x, x)
	}
}

func main() {
	PrintAny(10)
	PrintAny("go")
	PrintAny(true)
	PrintAny(3.14)
}
```

ผลลัพธ์:

```
เป็น int: 20
เป็น string: gogo
เป็น bool: false
ไม่รู้จักชนิด float64: 3.14
```

ตัวอย่างที่คุ้นเคยที่สุดของ empty interface ในการเขียนโปรแกรม Go จริงๆ คือ `fmt.Println(args ...any)` — นี่คือเหตุผลที่เราส่งค่าอะไรก็ได้เข้า `fmt.Println` มาตั้งแต่บทแรกของหลักสูตรนี้ได้เลย

---

## 6. เมื่อไหร่ควรใช้ Empty Interface เมื่อไหร่คือ Anti-Pattern

Empty interface ทรงพลังมากแต่ก็ **สูญเสีย type safety ไปเกือบทั้งหมด** — compiler ไม่สามารถเช็คอะไรให้เราได้เลยตอน compile time ว่าค่าข้างในเป็น type ที่ถูกต้องหรือไม่ ต้องมาเช็คเอาตอน runtime ด้วย type assertion/type switch เท่านั้น ซึ่งถ้าเดาผิดก็คือ panic หรือ logic ผิดพลาดตอนโปรแกรมรันจริง

### กรณีที่เหมาะสมกับการใช้ `any`

- ฟังก์ชันที่ต้องรับค่าได้จริงๆ ทุกชนิดโดยธรรมชาติของงาน เช่น `fmt.Println`, การ encode/decode ข้อมูลแบบ `encoding/json` ที่ต้องแปลง Go value ไปมากับ JSON ซึ่งไม่รู้ structure ล่วงหน้า
- โครงสร้างข้อมูลทั่วไปที่ต้องเก็บอะไรก็ได้ เช่น cache แบบ generic ที่เก็บค่าได้ทุกชนิด, หรือ `context.Context` ที่เก็บค่าประกอบ (values) แบบ key-value ใน Part 032
- ตอนเขียนโค้ดที่ทำงานร่วมกับ reflection (Part 031)

### กรณีที่เป็น Anti-Pattern

- ใช้ `any` เป็น parameter หรือ return type เพียงเพราะ **ขี้เกียจกำหนด type ให้ชัดเจน** ทั้งที่จริงๆ รู้อยู่แล้วว่ารับ/คืนอะไร เช่น:

```go
// แย่: ทำไมต้องรับ any ทั้งที่รู้อยู่แล้วว่าอยากได้แค่ int กับ int
func Add(a, b any) any {
	x := a.(int)
	y := b.(int)
	return x + y
}

// ดีกว่ามาก: ระบุ type ตรงๆ ให้ compiler ช่วยเช็ค
func Add(a, b int) int {
	return a + b
}
```

- ใช้ `map[string]any` แทนที่จะประกาศ `struct` ที่มี field ชัดเจน ทั้งที่ shape ของข้อมูลรู้อยู่แล้วล่วงหน้า ทำให้เสีย type safety และ autocomplete ของ editor ไปฟรีๆ
- ตั้งแต่ **Go 1.18 เป็นต้นมา มี Generics** (จะเรียนเต็มใน Part 028-029) ซึ่งในหลายกรณีที่เคยต้องพึ่ง `any` มาก่อน (เช่น container ที่เก็บข้อมูลชนิดเดียวกันจำนวนมาก อย่าง stack, queue, linked list) ตอนนี้ **ควรใช้ Generics แทน** เพราะได้ type safety กลับคืนมาเต็มรูปแบบโดยไม่ต้องแลกกับความยืดหยุ่น

หลักตัดสินใจง่ายๆ: **ถ้ารู้ type ล่วงหน้าได้ ให้ระบุ type นั้นตรงๆ เสมอ ใช้ `any` เฉพาะตอนที่ธรรมชาติของปัญหาจริงๆ ต้องการรับ/เก็บค่าได้ทุกชนิดเท่านั้น**

---

## 7. Interface Embedding: ประกอบ Interface ใหญ่จาก Interface เล็ก

ต่อยอดจากหัวข้อ "small interfaces idiom" ใน Part 013 — Go อนุญาตให้เรา **embed interface หนึ่งไว้ใน interface อีกตัว** เพื่อประกอบ interface ที่ใหญ่ขึ้นจาก interface เล็กๆ หลายตัว วิธีนี้คือที่มาของ `io.ReadWriter` ใน standard library

```go
package main

import (
	"bytes"
	"fmt"
	"io"
)

// Reader และ Writer เป็น interface เล็กๆ
type Reader interface {
	Read(p []byte) (n int, err error)
}

type Writer interface {
	Write(p []byte) (n int, err error)
}

// ReadWriter ประกอบจาก interface เล็กสองตัว (interface embedding)
type ReadWriter interface {
	Reader
	Writer
}

func useReadWriter(rw ReadWriter) {
	rw.Write([]byte("hello"))
	buf := make([]byte, 5)
	rw.Read(buf)
	fmt.Println(string(buf))
}

func main() {
	var buf bytes.Buffer // bytes.Buffer implement ทั้ง Read และ Write อยู่แล้ว
	useReadWriter(&buf)

	// io.ReadWriter ใน standard library ก็ประกอบแบบเดียวกันนี้เป๊ะ
	var _ io.ReadWriter = &bytes.Buffer{}
}
```

ผลลัพธ์:

```
hello
```

Type ที่จะ implement `ReadWriter` ได้ต้องมี method ครบทั้งจาก `Reader` และ `Writer` (คือ `Read` และ `Write`) — เพราะ interface embedding แค่ "รวม" รายการ method จากทุก interface ที่ embed เข้าด้วยกันเป็น interface เดียว ไม่มีอะไรซับซ้อนไปกว่านั้น

หลักการนี้สอดคล้องกับ struct embedding ที่จะเรียนแบบเต็มใน Part 030 — Go ใช้แนวคิด **composition** (การประกอบเข้าด้วยกัน) แทน **inheritance** (การสืบทอด) อย่างสม่ำเสมอทั้งกับ struct และ interface นี่คือเหตุผลที่ standard library มี interface หลายระดับความใหญ่ ตั้งแต่ `io.Reader`, `io.Writer` (เล็กสุด) ไปจนถึง `io.ReadWriteCloser` (ประกอบจาก 3 interface เล็ก) โดยที่ทุกระดับยังคงใช้งานร่วมกันได้อย่างลงตัว

---

## 8. หลักการ "Accept Interfaces, Return Structs"

นี่คือหนึ่งใน idiom การออกแบบ API ที่สำคัญที่สุดของ Go สรุปสั้นๆ คือ:

> **ฟังก์ชัน/method ควรรับ parameter เป็น interface (แคบที่สุดเท่าที่จำเป็น) แต่ควรคืนค่าเป็น concrete struct (หรือ pointer ไปยัง struct)**

```go
package main

import "fmt"

// Logger เป็น interface เล็กๆ ฝั่งรับ (accept interfaces)
type Logger interface {
	Log(msg string)
}

type ConsoleLogger struct{ Prefix string }

func (c ConsoleLogger) Log(msg string) {
	fmt.Println(c.Prefix + msg)
}

// Service รับ Logger แบบ interface เข้ามา (ยืดหยุ่น สลับ implementation ได้ ทดสอบง่าย)
// แต่คืนค่าเป็น *Service ซึ่งเป็น concrete struct (return structs)
// ผู้เรียกได้ field และ method ทั้งหมดของ Service ไม่ต้องเดาว่า interface ที่ได้มามีอะไรบ้าง
type Service struct {
	logger    Logger
	callCount int
}

func NewService(l Logger) *Service {
	return &Service{logger: l}
}

func (s *Service) DoWork() {
	s.callCount++
	s.logger.Log(fmt.Sprintf("working (call #%d)...", s.callCount))
}

func main() {
	svc := NewService(ConsoleLogger{Prefix: "[app] "})
	svc.DoWork()
	svc.DoWork()
	fmt.Println("total calls:", svc.callCount) // เข้าถึง field ตรงๆ ได้ เพราะ svc เป็น concrete struct
}
```

ผลลัพธ์:

```
[app] working (call #1)...
[app] working (call #2)...
total calls: 2
```

### ทำไมต้องรับเป็น interface

ฝั่ง parameter รับเป็น interface (`Logger`) ทำให้ `NewService` ยืดหยุ่นมาก — ผู้เรียกส่ง `ConsoleLogger`, `FileLogger`, หรือ mock logger ตอนเขียน test เข้ามาแทนกันได้หมด โดยไม่ต้องแก้โค้ดของ `Service` เลย ตรงตามหลักการ decoupling ที่เรียนไปใน Part 013

### ทำไมต้องคืนเป็น struct

ฝั่ง return ค่าคืนเป็น `*Service` (concrete struct) แทนที่จะคืนเป็น interface ทำให้:

- ผู้เรียกได้ field และ method **ทั้งหมด** ของ `Service` ไม่ถูกจำกัดไว้แค่ method ที่ interface ประกาศไว้
- เพิ่ม method หรือ field ใหม่ให้ `Service` ในอนาคตได้อย่างอิสระ โดยไม่ต้องแก้ interface (ถ้าคืนเป็น interface อยู่แล้ว การเพิ่ม method ใหม่จะกลายเป็น breaking change ทันที เพราะต้องแก้ interface definition ด้วย)
- ผู้เรียกเห็นชัดเจนว่ากำลังทำงานกับอะไรอยู่จริงๆ ไม่ต้องเดาจาก interface ว่าเบื้องหลังคือ type ไหน

สรุปสั้นที่สุด: **"be liberal in what you accept, be specific in what you return"** — รับให้กว้าง (interface) แต่คืนให้ชัด (concrete struct)

---

## 9. ข้อผิดพลาดที่พบบ่อย: Over-Abstraction ด้วย Interface เร็วเกินไป

แม้ interface จะเป็นเครื่องมือทรงพลัง แต่ก็เป็นเครื่องมือที่ **ถูกใช้ผิดบ่อยที่สุด** โดยเฉพาะโดยโปรแกรมเมอร์ที่ย้ายมาจากภาษาที่เน้น OOP หนักๆ อย่าง Java หรือ C# ซึ่งเคยชินกับการสร้าง interface ให้ทุกอย่างตั้งแต่ต้น

### อาการของปัญหา

```go
// แย่: สร้าง interface ทั้งที่มี implementation เดียวในระบบทั้งหมด
// และไม่มีแผนจะเพิ่ม implementation ตัวที่สองเลย
type UserRepository interface {
	FindByID(id int) (*User, error)
	FindByEmail(email string) (*User, error)
	Save(u *User) error
	Delete(id int) error
}

type postgresUserRepository struct {
	// ...
}

func (r *postgresUserRepository) FindByID(id int) (*User, error)      { /* ... */ }
func (r *postgresUserRepository) FindByEmail(email string) (*User, error) { /* ... */ }
func (r *postgresUserRepository) Save(u *User) error                  { /* ... */ }
func (r *postgresUserRepository) Delete(id int) error                 { /* ... */ }
```

ถ้าทั้งระบบมี implementation ของ `UserRepository` แค่ตัวเดียว (`postgresUserRepository`) และไม่มีแผนจะเปลี่ยนฐานข้อมูลหรือเพิ่ม mock ในเร็วๆ นี้ การสร้าง interface ตั้งแต่แรกแบบนี้เป็นการเพิ่มความซับซ้อนโดยไม่ได้ประโยชน์อะไรกลับมาเลย — ผู้อ่านโค้ดต้องกระโดดไปมาระหว่าง interface กับ struct ที่ implement มันอยู่เสมอ ทั้งที่จริงๆ มีแค่ทางเดียวให้ไป

### แนวทางที่ดีกว่า

**เขียน concrete struct ตรงๆ ไปก่อน** แล้วค่อยสกัด (extract) interface ออกมาทีหลัง **เมื่อมีเหตุผลที่พิสูจน์แล้วจริงๆ** เช่น:

- ต้องการเขียน unit test โดย mock dependency ตัวนั้นออกไป (มักสกัดเฉพาะ method ที่ test ต้องการเท่านั้น ไม่ใช่ทั้งหมด)
- มี implementation ตัวที่สองเกิดขึ้นจริงแล้ว (เช่น เพิ่มการรองรับ MySQL นอกจาก PostgreSQL)
- ต้องการ decouple module สองตัวออกจากกันตาม Clean Architecture (Part 100) ซึ่งจะเห็นเหตุผลชัดเจนตอนนั้น

```go
// เริ่มแบบนี้ก่อน: struct ตรงๆ ไม่มี interface
type PostgresUserRepository struct {
	// ...
}

func (r *PostgresUserRepository) FindByID(id int) (*User, error) { /* ... */ }
// ...

// ค่อยสกัด interface ออกมาทีหลัง เมื่อมี Service ที่ต้องการแค่บาง method เท่านั้น (ตาม small interfaces idiom)
type UserFinder interface {
	FindByID(id int) (*User, error)
}
```

Rob Pike เคยให้แนวทางที่สรุปเรื่องนี้ไว้กระชับมาก:

> "A little copying is better than a little dependency." — ก็อปปี้โค้ดนิดหน่อยยังดีกว่าสร้าง dependency (ผ่าน interface ที่ไม่จำเป็น) นิดหน่อย

และอีกคำแนะนำที่ยึดถือกันในชุมชน Go:

> "Don't design with interfaces, discover them." — อย่าออกแบบด้วย interface ตั้งแต่แรก แต่ให้ **ค้นพบ** มันจากการใช้งานจริง

หลักตัดสินใจสั้นๆ ที่ใช้ได้จริง: **ถ้าไม่มีเหตุผลที่จับต้องได้ตอนนี้ว่าทำไมต้องมี interface (เช่น มีแค่ implementation เดียว ไม่มีแผน mock ตอน test) ให้เขียน concrete struct ตรงๆ ไปก่อน แล้วสกัด interface ออกมาทีหลังเมื่อความจำเป็นปรากฏชัดเจน**

---

## สรุปสิ่งที่ได้เรียนในบทนี้

- **Type assertion แบบ comma-ok** (`v, ok := x.(T)`) เป็นวิธีปลอดภัยในการเช็คและดึง concrete type ออกจาก interface โดยไม่ panic ถ้าไม่ตรง
- **Type assertion แบบ panicking form** (`v := x.(T)`) ใช้เมื่อมั่นใจว่า type ต้องตรงแน่ๆ ถ้าผิดถือเป็นบั๊กที่ควร fail fast
- **Type switch** (`switch v := x.(type)`) ใช้เมื่อต้องเช็คกับหลาย type พร้อมกัน อ่านง่ายกว่าเขียน `if`-`else` ต่อกันหลายรอบ
- **Empty interface** (`interface{}` หรือ `any` ตั้งแต่ Go 1.18) เก็บค่าอะไรก็ได้ เพราะทุก type implement มันโดยอัตโนมัติ แต่แลกมาด้วยการเสีย type safety เกือบทั้งหมด
- ควรใช้ `any` เฉพาะตอนธรรมชาติของปัญหาต้องการรับ/เก็บค่าได้ทุกชนิดจริงๆ ไม่ใช่ใช้แทนการระบุ type ที่รู้อยู่แล้ว — และตั้งแต่มี **Generics** ควรพิจารณา Generics แทน `any` ในหลายกรณี
- **Interface embedding** ประกอบ interface ใหญ่จาก interface เล็กหลายตัวได้ (เช่น `io.ReadWriter` จาก `io.Reader` + `io.Writer`) สอดคล้องกับหลัก composition ของ Go
- หลักการ **"accept interfaces, return structs"** — รับ parameter เป็น interface เพื่อความยืดหยุ่น แต่คืนค่าเป็น concrete struct เพื่อให้ผู้เรียกได้ความสามารถเต็มรูปแบบ
- ข้อผิดพลาดที่พบบ่อยที่สุดคือ **over-abstraction**: สร้าง interface ตั้งแต่ยังไม่มีเหตุผลชัดเจน ควรเขียน concrete struct ไปก่อนแล้วค่อยสกัด interface ออกมาทีหลังเมื่อจำเป็นจริง

## แบบฝึกหัดท้ายบท

1. เขียนฟังก์ชัน `sumInts(items []any) int` ที่รับ slice ของ `any` แล้วบวกเฉพาะค่าที่เป็น `int` เข้าด้วยกัน (ใช้ comma-ok type assertion เช็คแต่ละตัวก่อนบวก ข้ามค่าที่ไม่ใช่ int ไป)
2. เขียน type switch ที่รับ `any` แล้วแยกแยะระหว่าง `int`, `float64`, `string`, `[]int`, และ `nil` พิมพ์ข้อความอธิบายที่ต่างกันในแต่ละกรณี
3. ประกอบ interface `ReadWriteCloser` ขึ้นเองจาก `Reader`, `Writer`, และ `Closer` (interface ละ 1 method) ด้วย interface embedding แล้วสร้าง struct จำลองหนึ่งตัวที่ implement ครบทั้ง 3 method
4. ทบทวนโค้ดเก่าของตัวเอง (หรือโค้ดตัวอย่างจาก Part ก่อนหน้า) หา 1 จุดที่ใช้ `any` หรือ `map[string]any` อยู่ทั้งที่รู้ shape ของข้อมูลล่วงหน้าอยู่แล้ว แล้วลองแก้ให้ใช้ `struct` แทน
5. เขียนฟังก์ชันที่รับ interface `Logger` (มี method `Log(string)`) แต่คืนค่าเป็น `*Worker` (concrete struct ที่มี field `TaskCount int`) ตามหลัก "accept interfaces, return structs" แล้วอธิบายว่าทำไมไม่คืนเป็น interface
6. ยกตัวอย่างสถานการณ์ในงานจริง (หรือสมมติ) ที่การสร้าง interface ตั้งแต่แรกเป็น over-abstraction แล้วอธิบายว่าจะรอสัญญาณอะไรก่อนถึงจะสกัด interface ออกมา

---

**ต่อไป**: [Part 015 — Error Handling พื้นฐาน](./015-error-handling-basics.md)
