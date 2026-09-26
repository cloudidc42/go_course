# Part 012: Methods และ Receiver (value vs pointer)

> ภาคที่ 1: พื้นฐานภาษา Go (Fundamentals) — ตอนที่ 12 จาก 15

## สารบัญของบทนี้

1. Method คืออะไร ต่างจาก function ธรรมดาอย่างไร
2. Syntax การประกาศ Method: Receiver
3. Value Receiver
4. Pointer Receiver
5. เมื่อไหร่ต้องใช้ Pointer Receiver
6. Method Set คืออะไร
7. Method Set กับการ Implement Interface (จุดที่พลาดกันบ่อยที่สุด)
8. กฎการเลือก Receiver ให้สม่ำเสมอทั้ง Type
9. Method บน Named Type ที่ไม่ใช่ Struct
10. Method Value และ Method Expression
11. สรุปสิ่งที่ได้เรียนในบทนี้
12. แบบฝึกหัดท้ายบท

---

## 1. Method คืออะไร ต่างจาก function ธรรมดาอย่างไร

ตามที่กล่าวไปใน Part 001 ว่า Go **ไม่มี class** แบบภาษา OOP ทั่วไป แต่ Go ให้เราผูกฟังก์ชันเข้ากับ type ที่เราประกาศเองได้ (โดยเฉพาะ `struct` จาก Part 011) เรียกฟังก์ชันแบบนี้ว่า **method**

พูดง่ายๆ **method คือ function ที่มี "receiver" ต่อท้ายจาก keyword `func`** ทำให้เราเรียกมันผ่าน syntax แบบจุด (`ตัวแปร.MethodName()`) ได้ เหมือนที่เราเคยเห็น `fmt.Println` หรือ `strings.ToUpper` — จริงๆ แล้วของพวกนี้ก็คือ function ของ package ไม่ใช่ method แต่ style การเรียกแบบจุดที่เราจะเขียนเองในบทนี้ ทำให้โค้ดอ่านเป็นธรรมชาติคล้ายกัน

เทียบให้เห็นภาพชัดๆ:

```go
package main

import "fmt"

type Rectangle struct {
	Width, Height float64
}

// function ธรรมดา: ต้องส่ง Rectangle เข้าไปเป็น argument ตรงๆ
func AreaFunc(r Rectangle) float64 {
	return r.Width * r.Height
}

// method: ผูกกับ type Rectangle ผ่าน receiver "(r Rectangle)"
func (r Rectangle) Area() float64 {
	return r.Width * r.Height
}

func main() {
	rect := Rectangle{Width: 10, Height: 5}

	fmt.Println(AreaFunc(rect)) // เรียกแบบ function ธรรมดา
	fmt.Println(rect.Area())    // เรียกแบบ method ผ่านตัวแปร rect
}
```

ทั้งสองแบบทำงานเหมือนกันทุกประการในทางเทคนิค — คอมไพเลอร์แปลง `rect.Area()` เป็นการเรียกฟังก์ชันที่รับ `rect` เป็น argument ตัวแรกอยู่ดี (concept นี้จะสำคัญมากตอนพูดถึง **method expression** ในหัวข้อท้ายบท) แต่ประโยชน์ของการเขียนเป็น method คือ:

- โค้ดอ่านเป็นธรรมชาติ: `rect.Area()` อ่านว่า "rect หา area ของตัวเอง"
- เป็นกลไกเดียวที่ทำให้ type ของเรา **implement interface ได้** (จะเรียนใน Part 013)
- จัดกลุ่มพฤติกรรม (behavior) ให้อยู่ติดกับ struct ที่มันเกี่ยวข้อง คล้ายแนวคิด encapsulation ใน OOP แต่ Go ทำผ่าน composition ไม่ใช่ inheritance

---

## 2. Syntax การประกาศ Method: Receiver

โครงสร้าง syntax ของ method คือ:

```go
func (receiverName ReceiverType) MethodName(params) returnType {
    // body
}
```

ส่วนที่ต่างจาก function ธรรมดาคือวงเล็บก้อนแรกหลัง `func` เรียกว่า **receiver** ประกอบด้วยชื่อตัวแปร (เหมือนชื่อ parameter) และ type ที่ method นี้ผูกอยู่

```go
func (r Rectangle) Area() float64 {
	return r.Width * r.Height
}
```

อ่านว่า: "ประกาศ method ชื่อ `Area` ที่มี receiver ชื่อ `r` ชนิด `Rectangle`" เมื่อเรียก `rect.Area()` ตัวแปร `r` ภายใน method จะเป็นสำเนา (หรือ pointer แล้วแต่กรณี) ของ `rect`

**ข้อจำกัดสำคัญ**: receiver type ต้องประกาศอยู่ใน **package เดียวกัน** กับที่ประกาศ method เท่านั้น เราไม่สามารถเพิ่ม method ให้ type จาก package อื่นได้ (เช่น เพิ่ม method ให้ `string` หรือ `time.Time` ตรงๆ ไม่ได้) หากต้องการพฤติกรรมแบบนั้น ต้องสร้าง named type ใหม่ครอบไว้ก่อน (จะเห็นตัวอย่างในหัวข้อที่ 9)

ชื่อ receiver ตามธรรมเนียมของ Go นิยมใช้ตัวย่อสั้นๆ 1-2 ตัวอักษรจากชื่อ type (เช่น `r` สำหรับ `Rectangle`, `c` สำหรับ `Circle`) **ไม่นิยมใช้ `this` หรือ `self`** แบบภาษาอื่น เพราะ Go มองว่า receiver ก็เป็นแค่ parameter ตัวหนึ่งไม่ได้พิเศษกว่าตัวอื่น

---

## 3. Value Receiver

**Value receiver** คือ receiver ที่ไม่มีเครื่องหมาย `*` นำหน้า type เช่น `(r Rectangle)` เมื่อเรียก method แบบนี้ Go จะ**ก็อปปี้ค่าทั้งหมดของ struct** ส่งเข้าไปใน method เหมือนการส่ง argument แบบ pass-by-value ที่เคยเรียนใน Part 008-009

```go
package main

import "fmt"

type Rectangle struct {
	Width, Height float64
}

func (r Rectangle) Area() float64 {
	return r.Width * r.Height
}

func main() {
	rect := Rectangle{Width: 10, Height: 5}
	fmt.Println("Area:", rect.Area()) // 50
}
```

จุดสำคัญคือ **method ที่มี value receiver จะไม่สามารถแก้ไขค่าต้นฉบับได้** เพราะสิ่งที่ทำงานอยู่ข้างในคือสำเนา แก้เท่าไหร่ก็ไม่มีผลกับตัวแปรจริงข้างนอก เช่นเดียวกับที่เคยเห็นตอนส่ง struct เข้า function ธรรมดาใน Part 011

```go
func (r Rectangle) TryScale(factor float64) {
	r.Width *= factor  // แก้แค่สำเนา r ภายใน method นี้เท่านั้น
	r.Height *= factor
}

func main() {
	rect := Rectangle{Width: 10, Height: 5}
	rect.TryScale(2)
	fmt.Println(rect) // ยังเป็น {10 5} เหมือนเดิม ไม่เปลี่ยน!
}
```

---

## 4. Pointer Receiver

**Pointer receiver** คือ receiver ที่มี `*` นำหน้า type เช่น `(r *Rectangle)` เมื่อเรียก method แบบนี้ Go จะส่ง **pointer ที่ชี้ไปยังตัวแปรต้นฉบับ** เข้าไปใน method ทำให้การแก้ไขค่าผ่าน receiver มีผลต่อตัวแปรจริงข้างนอกด้วย เหมือนหลักการ pointer ที่เรียนละเอียดใน Part 010

```go
package main

import "fmt"

type Rectangle struct {
	Width, Height float64
}

func (r *Rectangle) Scale(factor float64) {
	r.Width *= factor
	r.Height *= factor
}

func main() {
	rect := Rectangle{Width: 10, Height: 5}
	rect.Scale(2) // Go แปลงให้เป็น (&rect).Scale(2) อัตโนมัติ
	fmt.Println("After scale:", rect) // {20 10}
}
```

สังเกตว่าตอนเรียก `rect.Scale(2)` เราไม่ได้เขียน `(&rect).Scale(2)` เอง — **Go เป็นคนแปลงให้อัตโนมัติ** เพราะ `rect` เป็นตัวแปรที่ **addressable** (มีที่อยู่ในหน่วยความจำที่หยิบมาได้ด้วย `&`) กฎนี้ทำงานทั้งสองทิศทาง:

- เรียก pointer-receiver method ผ่านตัวแปรธรรมดา → Go เติม `&` ให้อัตโนมัติ (ถ้า addressable)
- เรียก value-receiver method ผ่านตัวแปร pointer → Go เติม `*` (dereference) ให้อัตโนมัติ

```go
p := &Rectangle{Width: 3, Height: 4}
fmt.Println(p.Area()) // Go แปลงเป็น (*p).Area() ให้อัตโนมัติ แม้ Area จะมี value receiver
```

ความสะดวกนี้ทำให้เราแทบไม่ต้องกังวลเรื่อง `&` หรือ `*` เองตอนเรียก method — **ยกเว้นกรณีตัวแปรไม่ addressable** ซึ่งจะเห็นในหัวข้อที่ 7

---

## 5. เมื่อไหร่ต้องใช้ Pointer Receiver

สรุปเป็นหลักการที่ใช้ตัดสินใจได้จริงในงานเขียนโค้ด:

| สถานการณ์ | ควรใช้ |
|---|---|
| Method ต้อง**แก้ไขค่า field** ของ struct | **Pointer receiver** (บังคับ — ไม่งั้นแก้ไม่ได้จริง) |
| Struct มีขนาดใหญ่ (หลาย field หรือมี array ขนาดใหญ่) | **Pointer receiver** เพื่อเลี่ยงการก็อปปี้ข้อมูลจำนวนมากทุกครั้งที่เรียก |
| Struct มี field ที่เป็น mutex, หรือห้าม copy โดยเด็ดขาด | **Pointer receiver** เสมอ |
| Struct เล็กมาก (เช่น `struct{ X, Y int }`) และไม่ต้องแก้ไขค่า | Value receiver ก็ได้ (แต่ดูหัวข้อ 8 เรื่องความสม่ำเสมอ) |
| Type เป็น slice, map, channel, function, หรือ primitive (int, string) ที่ไม่ต้องแก้ไข | Value receiver มักเพียงพอ |

ตัวอย่างที่เห็นภาพชัดว่าทำไม value receiver ใช้ไม่ได้กับกรณีที่ต้อง mutate:

```go
type Counter struct {
	count int
}

// ผิด: value receiver ทำให้ Increment ไม่มีผลอะไรกับ c ต้นฉบับเลย
func (c Counter) IncrementWrong() {
	c.count++
}

// ถูก: pointer receiver ทำให้แก้ไขค่าจริงได้
func (c *Counter) Increment() {
	c.count++
}

func main() {
	c := Counter{}
	c.IncrementWrong()
	c.IncrementWrong()
	fmt.Println(c.count) // 0 -- งงเพราะดูเหมือนเรียก 2 ครั้งแต่ไม่เพิ่มเลย

	c.Increment()
	c.Increment()
	fmt.Println(c.count) // 2 -- ทำงานถูกต้อง
}
```

บั๊กแบบ `IncrementWrong` นี้เป็นบั๊กคลาสสิกของมือใหม่ Go มาก — โค้ด **compile ผ่านสบายๆ ไม่มี error หรือ warning ใดๆ** เพราะไวยากรณ์ถูกต้องทุกอย่าง แต่ผลลัพธ์ทาง logic ผิดเงียบๆ จึงต้องจำกฎให้ขึ้นใจ: **ถ้า method ต้องเปลี่ยนแปลงค่า field ของ struct ต้นฉบับ ต้องใช้ pointer receiver เท่านั้น**

### ทำไมขนาดของ struct ถึงมีผลต่อการเลือก Receiver

เหตุผลเรื่อง "ประสิทธิภาพ" ที่กล่าวถึงข้างต้นมาจากข้อเท็จจริงพื้นฐานเรื่อง pass-by-value ที่เรียนใน Part 008-010: ทุกครั้งที่เรียก method ด้วย value receiver ข้อมูลทั้งหมดของ struct จะถูก**ก็อปปี้** เข้าไปในหน่วยความจำใหม่ ถ้า struct มีขนาดเล็ก (เช่น 2 field เป็น `float64`) การก็อปปี้แทบไม่มีต้นทุนอะไรเลย แต่ถ้า struct มีขนาดใหญ่ (เช่น เก็บ array ขนาดใหญ่ หรือมีหลายสิบ field) การก็อปปี้ทุกครั้งที่เรียก method จะกลายเป็นภาระด้าน performance ที่สะสมได้จริงเมื่อเรียกบ่อยๆ

```go
type SmallPoint struct {
	X, Y int // ขนาดรวมประมาณ 16 bytes บนเครื่อง 64-bit -- ก็อปปี้ถูกมาก
}

type LargeBuffer struct {
	Data [1024]byte // ขนาด 1024 bytes -- ก็อปปี้ทุกครั้งที่เรียก value-receiver method แพงกว่ามาก
	Name string
}

func (p SmallPoint) Sum() int { return p.X + p.Y } // value receiver โอเค เพราะเล็กมาก

func (b *LargeBuffer) Checksum() byte { // ควรใช้ pointer receiver เพื่อเลี่ยงก็อปปี้ 1024 bytes ทุกครั้ง
	var sum byte
	for _, v := range b.Data {
		sum += v
	}
	return sum
}
```

หลักคร่าวๆ ที่ใช้อ้างอิงได้ (ไม่ใช่กฎตายตัว): ถ้า struct มีขนาดใหญ่กว่าสัก 2-3 คำ (word) ของระบบ (บนเครื่อง 64-bit ก็คือใหญ่กว่าประมาณ 24 bytes) มักคุ้มค่ากว่าที่จะใช้ pointer receiver แม้ method นั้นจะไม่ได้ต้อง mutate อะไรเลยก็ตาม

### กรณีพิเศษ: ตัวแปรที่ไม่ addressable ก็เรียก pointer-receiver method ไม่ได้

ค่าที่ไม่ addressable เช่น ค่าที่อยู่ใน `map` (map value ไม่ addressable) จะเรียก pointer-receiver method ตรงๆ ไม่ได้ ถึงแม้ค่านั้นจะเป็น struct ก็ตาม:

```go
type Item struct {
	Name string
}

func (i *Item) Describe() string {
	return "Item: " + i.Name
}

func main() {
	m := map[string]Item{"a": {Name: "x"}}
	fmt.Println(m["a"].Describe()) // compile error!
}
```

รันแล้วจะได้ error:

```
./main.go:15:21: cannot call pointer method Describe on Item
```

เหตุผลคือ `m["a"]` ไม่ได้เป็นตัวแปรจริงที่มีที่อยู่ในหน่วยความจำแบบตรงไปตรงมา (map ภายในอาจ rehash ย้ายตำแหน่งข้อมูลได้ตลอดเวลา) Go จึงห้าม `&m["a"]` เพื่อความปลอดภัย วิธีแก้คือดึงค่าออกมาเป็นตัวแปรก่อน แล้วค่อยเซ็ตกลับเข้า map:

```go
item := m["a"]
item.Describe() // ใช้ได้ เพราะ item เป็นตัวแปรจริง addressable
m["a"] = item   // ถ้ามีการแก้ไขค่า ต้องเซ็ตกลับเข้า map เอง
```

หรือเปลี่ยน map ให้เก็บ `*Item` แทน `Item` ตั้งแต่แรกก็เป็นอีกทางออกที่นิยมใช้ในโค้ดจริง

---

## 6. Method Set คืออะไร

**Method set** ของ type คือ "ชุดของ method ทั้งหมดที่สามารถเรียกได้บนค่าของ type นั้น" นี่เป็นแนวคิดที่ดูเป็นนามธรรม แต่สำคัญมากเพราะมันคือกฎที่ตัดสินว่า type ไหน implement interface ไหนได้บ้าง (Part 013)

กฎของ method set มีอยู่แค่นี้:

- **Method set ของ type `T`** (value type) ประกอบด้วย method ที่มี **value receiver** เท่านั้น
- **Method set ของ type `*T`** (pointer type) ประกอบด้วย method ที่มี**ทั้ง value receiver และ pointer receiver**

พูดสั้นๆ ที่จำง่าย: **`*T` มี method set ที่กว้างกว่า `T` เสมอ** เพราะ pointer เข้าถึง method แบบ value receiver ได้ด้วย (ผ่านการ auto-dereference ที่เห็นในหัวข้อ 4) แต่ value เข้าถึง method แบบ pointer receiver ไม่ได้ (เพราะ value เดี่ยวๆ อาจไม่ addressable)

```go
type Rectangle struct{ Width, Height float64 }

func (r Rectangle) Area() float64       { return r.Width * r.Height } // value receiver
func (r *Rectangle) Scale(f float64)    { r.Width *= f; r.Height *= f } // pointer receiver
```

| Type | Method Set |
|---|---|
| `Rectangle` | `Area()` เท่านั้น |
| `*Rectangle` | `Area()` และ `Scale()` |

ตราบใดที่เรียก method ผ่านตัวแปรที่ addressable ตรงๆ (เช่น `rect.Scale(2)` ที่เห็นในหัวข้อ 4) Go จะช่วยแปลงให้อัตโนมัติจนแทบไม่รู้สึกถึงกฎนี้ — **แต่พอเมื่อไหร่ที่ type ถูกเก็บเป็น interface value กฎนี้จะเข้มงวดขึ้นทันที** และนี่คือประเด็นที่สำคัญที่สุดของบทนี้

---

## 7. Method Set กับการ Implement Interface (จุดที่พลาดกันบ่อยที่สุด)

Go ตัดสินว่า type หนึ่งจะ "implement" interface ได้หรือไม่ โดยเช็คว่า **method set ของ type นั้นครอบคลุม method signature ทั้งหมดที่ interface กำหนดหรือเปล่า** (จะเรียนเรื่อง interface แบบเต็มใน Part 013 บทนี้ขอพูดเฉพาะผลกระทบจาก method set)

ปัญหาคือ: ถ้า method ตัวใดตัวหนึ่งของ type ประกาศด้วย **pointer receiver** ค่าที่เป็น **value ล้วนๆ** (ไม่ใช่ pointer) จะไม่มี method นั้นอยู่ใน method set ของมัน จึง**ไม่ผ่านการ implement interface**

ลองดูตัวอย่างที่ **compile ไม่ผ่าน** เพื่อให้เห็นภาพชัดเจนที่สุด:

```go
package main

import "fmt"

type Describer interface {
	Describe() string
}

type Item struct {
	Name string
}

// Describe ประกาศด้วย pointer receiver
func (i *Item) Describe() string {
	return "Item: " + i.Name
}

func main() {
	var d Describer = Item{Name: "Book"} // ผิด!
	fmt.Println(d.Describe())
}
```

พยายาม compile จะได้ error ทันที:

```
./main.go:19:20: cannot use Item{…} (value of struct type Item) as Describer value
in variable declaration: Item does not implement Describer (method Describe has pointer receiver)
```

ข้อความ error บอกตรงๆ เลยว่า `Item does not implement Describer (method Describe has pointer receiver)` — เพราะ `Describe()` มี pointer receiver มันจึงอยู่ใน method set ของ `*Item` เท่านั้น **ไม่ได้อยู่ใน method set ของ `Item`** (value type) เมื่อเรา assign ค่า `Item{...}` (value ล้วน) ให้ตัวแปร interface `Describer` Go จึงมองว่า `Item` ไม่ได้ implement interface นี้

วิธีแก้มีสองทาง:

**ทางที่ 1**: assign เป็น pointer แทน (`&Item{...}`) ซึ่งมี method set ที่กว้างกว่า

```go
func main() {
	var d Describer = &Item{Name: "Book"} // ใช้ pointer -> compile ผ่าน
	fmt.Println(d.Describe())             // Item: Book
}
```

**ทางที่ 2**: เปลี่ยน `Describe` ให้เป็น value receiver ถ้า method นั้นไม่จำเป็นต้องแก้ไขค่าอะไร (ดูหัวข้อ 8 เรื่องความสม่ำเสมอประกอบ)

```go
func (i Item) Describe() string { // เปลี่ยนเป็น value receiver
	return "Item: " + i.Name
}

func main() {
	var d Describer = Item{Name: "Book"} // ตอนนี้ compile ผ่านแล้ว เพราะ value receiver อยู่ใน method set ของทั้ง Item และ *Item
	fmt.Println(d.Describe())
}
```

### สรุปเป็นตารางเปรียบเทียบที่ต้องจำ

| Type ของตัวแปร interface เก็บ | Method มี value receiver | Method มี pointer receiver |
|---|---|---|
| `T` (value) | ✅ ใช้ได้ — อยู่ใน method set ของ `T` | ❌ ไม่ได้ — ไม่ได้อยู่ใน method set ของ `T` |
| `*T` (pointer) | ✅ ใช้ได้ — pointer เข้าถึง value-receiver method ได้เสมอ | ✅ ใช้ได้ — อยู่ใน method set ของ `*T` โดยตรง |

**กฎจำง่ายที่สุด**: ถ้า type ของเรามี pointer receiver method แม้แต่ตัวเดียว ให้ผ่าน pointer (`&value`) เข้า interface เสมอ อย่าผ่าน value ตรงๆ เพราะจะพลาดแบบในตัวอย่างข้างต้นทันที นี่คือเหตุผลที่โค้ด Go ในโลกจริงจำนวนมาก เลือก**ใช้ pointer receiver ให้กับ method ทุกตัวของ struct ตัวเดียวกัน** เพื่อไม่ต้องมานั่งจำว่า method ไหนใช้ value ได้ method ไหนต้องใช้ pointer

---

## 8. กฎการเลือก Receiver ให้สม่ำเสมอทั้ง Type

จากปัญหาในหัวข้อที่แล้ว แนวปฏิบัติที่ยอมรับกันในวงกว้างของชุมชน Go (อ้างอิงจาก Go Code Review Comments อย่างเป็นทางการ) คือ:

> **ทุก method ของ struct เดียวกัน ควรใช้ receiver แบบเดียวกันทั้งหมด** ไม่ว่าจะเป็น value ล้วน หรือ pointer ล้วน ห้ามผสมกันโดยไม่มีเหตุผลชัดเจน

หลักในการเลือกว่าจะใช้แบบไหนสำหรับ type หนึ่งๆ:

1. ถ้ามี method ใด method หนึ่งที่ **ต้อง**ใช้ pointer receiver (เพราะต้อง mutate หรือ struct มีขนาดใหญ่) → **ใช้ pointer receiver กับทุก method ของ type นั้น** เพื่อความสม่ำเสมอ แม้บาง method จะไม่ได้แก้ไขค่าอะไรเลยก็ตาม
2. ถ้า struct เล็กมาก ไม่มี method ไหนต้อง mutate เลย → ใช้ value receiver กับทุก method ได้ ทำให้ type นี้ implement interface ได้ทั้งจาก value และ pointer อย่างยืดหยุ่น

```go
// แนวทางที่แนะนำ: ใช้ pointer receiver ให้สม่ำเสมอทั้ง type เพราะมี Scale ที่ต้อง mutate
type Rectangle struct {
	Width, Height float64
}

func (r *Rectangle) Area() float64 {
	return r.Width * r.Height
}

func (r *Rectangle) Scale(factor float64) {
	r.Width *= factor
	r.Height *= factor
}
```

ข้อดีของความสม่ำเสมอนี้คือทีมงานไม่ต้องมานั่งเดาว่า "method นี้ทำไมใช้ value แต่ตัวนั้นใช้ pointer" และลดโอกาสเจอบั๊กแบบ method set ที่เพิ่งเห็นไปเมื่อครู่

---

## 9. Method บน Named Type ที่ไม่ใช่ Struct

Method ไม่จำเป็นต้องผูกกับ `struct` เท่านั้น — เราสามารถประกาศ method บน **named type ใดๆ ก็ได้** ที่เราสร้างขึ้นเอง แม้จะมีฐานเป็น primitive type อย่าง `float64`, `int`, หรือ `string` ก็ตาม

```go
package main

import "fmt"

// Celsius คือ named type ที่มีฐานเป็น float64 ไม่ใช่ struct
type Celsius float64

func (c Celsius) ToFahrenheit() float64 {
	return float64(c)*9/5 + 32
}

func (c Celsius) String() string {
	return fmt.Sprintf("%.1f°C", float64(c))
}

func main() {
	body := Celsius(37.0)
	fmt.Println(body, "=", body.ToFahrenheit(), "F")
}
```

ผลลัพธ์:

```
37.0°C = 98.6 F
```

รูปแบบนี้มีประโยชน์มากในทางปฏิบัติ: แทนที่จะส่ง `float64` เปล่าๆ ไปมาในโปรแกรมแล้วเสี่ยงสับสนหน่วย (Celsius หรือ Fahrenheit?) เราสร้าง named type ขึ้นมาห่อไว้ แล้วผูก method ที่เกี่ยวข้องเข้ากับมันโดยตรง ทำให้ type system ช่วยป้องกันความผิดพลาดได้ตั้งแต่ compile time (เช่น ถ้ามี `type Fahrenheit float64` แยกอีกตัว จะส่ง `Celsius` ไปที่ parameter ที่รับ `Fahrenheit` ไม่ได้เลยถ้าไม่ cast ตรงๆ)

method `String()` ที่เห็นในตัวอย่างนี้ไม่ใช่เรื่องบังเอิญ — มันคือการ implement **`fmt.Stringer` interface** ที่ standard library กำหนดไว้ ทำให้เมื่อเราส่งค่า `Celsius` เข้า `fmt.Println` โดยตรง มันจะเรียก `String()` ให้อัตโนมัติแทนการพิมพ์ตัวเลขดิบๆ (สังเกตผลลัพธ์ `37.0°C` ไม่ใช่ `37`) เรื่องนี้จะเห็นภาพเต็มใน Part 013 เมื่อพูดถึง interface เล็กๆ ที่ทรงพลัง

**หมายเหตุสำคัญ**: กฎ "receiver type ต้องอยู่ package เดียวกัน" ที่กล่าวไปในหัวข้อ 2 หมายความว่าเราเพิ่ม method ให้ `float64` ตรงๆ ไม่ได้ (เพราะ `float64` เป็น built-in type ของภาษา ไม่ได้อยู่ใน package ของเรา) แต่พอเราสร้าง named type ของตัวเองอย่าง `Celsius` ขึ้นมาครอบไว้ก่อน เราก็เพิ่ม method ให้มันได้อย่างอิสระ นี่คือเทคนิคมาตรฐานที่ใช้แก้ข้อจำกัดนี้

---

## 10. Method Value และ Method Expression

Go มีอีกสองรูปแบบการใช้งาน method ที่พบไม่บ่อยเท่าการเรียกตรงๆ แต่มีประโยชน์เฉพาะทาง

### Method Value

**Method value** คือการหยิบ method ของตัวแปรตัวหนึ่งมาเก็บเป็นตัวแปร function เฉยๆ โดย **ผูก (bind) receiver เข้ากับค่าตอนนั้นทันที** ส่วน **method expression** คือการเขียนในรูปแบบ `Type.MethodName` (ไม่ผ่านตัวแปรใดๆ) ผลลัพธ์ที่ได้คือ function ที่ **รับ receiver เป็น parameter ตัวแรกอย่างชัดเจน** — สะท้อนให้เห็นตรงๆ เลยว่าจริงๆ แล้ว method ก็คือ function ที่มี receiver เป็น parameter พิเศษตามที่กล่าวไว้ในหัวข้อ 1 นั่นเอง

```go
package main

import "fmt"

type Rectangle struct {
	Width, Height float64
}

func (r Rectangle) Area() float64 {
	return r.Width * r.Height
}

func main() {
	rect := Rectangle{Width: 3, Height: 4}

	// Method value: ผูก receiver (rect) เข้ากับ method ไว้ล่วงหน้า
	// ได้ค่าที่เป็น function ธรรมดา ที่ "จำ" ค่า rect ตอนสร้างไว้
	areaFunc := rect.Area
	rect.Width = 100 // เปลี่ยนค่า rect หลังจากนี้ไม่มีผลกับ areaFunc เพราะ value receiver ก็อปปี้ไปแล้ว
	fmt.Println("Method value:", areaFunc())

	// Method expression: เขียนในรูปแบบ Type.Method
	// ได้ function ที่รับ receiver เป็น argument ตัวแรกอย่างชัดเจน
	areaExpr := Rectangle.Area
	fmt.Println("Method expression:", areaExpr(Rectangle{Width: 5, Height: 6}))
}
```

ผลลัพธ์:

```
Method value: 12
Method expression: 30
```

`areaFunc` มี type เป็น `func() float64` ธรรมดา ใช้งานเหมือน function ทั่วไปได้เลย เช่น ส่งเป็น callback หรือเก็บใน slice ของ function ได้ จุดสำคัญคือค่า `rect` ณ ตอนที่สร้าง `areaFunc` **ถูกก็อปปี้ (เพราะเป็น value receiver) แล้วผูกไว้ถาวร** การแก้ไข `rect` ในภายหลังจึงไม่กระทบ `areaFunc` อีกต่อไป (ผลลัพธ์เป็น `12` มาจากขนาดตอนสร้าง ไม่ใช่ `400` ที่ควรได้ถ้าคำนวณจาก `rect.Width = 100` ใหม่)

ส่วน `areaExpr` มี type เป็น `func(Rectangle) float64` — ต้องส่ง `Rectangle` เข้าไปเป็น argument ตัวแรกเองตรงๆ ตอนเรียก ต่างจาก method value ที่ผูก receiver ไว้ล่วงหน้าแล้ว

Method expression ใช้บ่อยในโค้ดที่เขียน generic-style function ที่ต้องรับ "การกระทำ" (method) ของหลายๆ type มาประมวลผลแบบเดียวกัน แต่ในงานเขียนโปรแกรมทั่วไป method value และ method expression พบไม่บ่อยเท่า method call ปกติ — รู้จักไว้เพื่อให้เข้าใจโค้ดขั้นสูงที่อาจเจอในไลบรารีบางตัวก็เพียงพอ

---

## สรุปสิ่งที่ได้เรียนในบทนี้

- **Method** คือ function ที่มี **receiver** ต่อท้าย `func` ทำให้เรียกผ่าน syntax แบบจุดได้ และเป็นกลไกเดียวที่ทำให้ type ใน Go implement interface ได้
- **Value receiver** `(r T)` ได้สำเนาของค่าไปทำงาน แก้ไขค่าต้นฉบับไม่ได้ ใช้กับ struct เล็กๆ ที่ไม่ต้อง mutate
- **Pointer receiver** `(r *T)` ได้ pointer ไปยังค่าต้นฉบับ แก้ไขค่าได้จริง จำเป็นเมื่อ method ต้อง mutate หรือ struct มีขนาดใหญ่
- Go แปลง `&`/`*` ให้อัตโนมัติเมื่อเรียก method ผ่านตัวแปรที่ addressable แต่ **ค่าที่ไม่ addressable (เช่น map value) เรียก pointer-receiver method ตรงๆ ไม่ได้**
- **Method set** ของ `T` มีแค่ value-receiver method ส่วน **method set ของ `*T` มีทั้ง value และ pointer-receiver method** — `*T` มี method set กว้างกว่าเสมอ
- ผลที่ตามมาคือ: type ที่มี **pointer-receiver method แม้แต่ตัวเดียว** จะไม่ implement interface นั้นถ้า assign เป็น **value** ต้อง assign เป็น **pointer** เท่านั้น นี่คือกับดักที่พบบ่อยที่สุดเรื่องหนึ่งของมือใหม่ Go
- แนวทางที่ดีคือ **เลือก receiver แบบเดียวกันให้สม่ำเสมอทั้ง type** ไม่ผสม value/pointer โดยไม่มีเหตุผล
- Method ประกาศบน **named type ที่ไม่ใช่ struct** ได้ด้วย เช่น `type Celsius float64` ซึ่งเป็นเทคนิคมาตรฐานสำหรับเพิ่มพฤติกรรมให้ primitive type ที่เพิ่มไม่ได้ตรงๆ
- **Method value** ผูก receiver ไว้ล่วงหน้าแล้วได้ function เปล่าๆ ส่วน **method expression** (`Type.Method`) ได้ function ที่รับ receiver เป็น parameter ตัวแรกอย่างชัดเจน

## แบบฝึกหัดท้ายบท

1. สร้าง struct `BankAccount` ที่มี field `Balance float64` เขียน method `Deposit(amount float64)` และ `Withdraw(amount float64) error` (คืน error ถ้าเงินไม่พอ) โดยเลือก receiver ให้ถูกต้องเพื่อให้ยอดเงินเปลี่ยนแปลงได้จริง อธิบายว่าทำไมต้องเลือก receiver แบบนั้น
2. เขียนโค้ดที่จงใจทำผิดพลาดแบบในหัวข้อ 5 (ใช้ value receiver กับ method ที่ต้อง mutate) แล้วสังเกตว่าทำไม compile ผ่านแต่ผลลัพธ์ผิด บันทึกคำอธิบายของตัวเองไว้
3. สร้าง interface `Notifier` ที่มี method `Notify() string` หนึ่งตัว สร้าง struct `EmailNotifier` ที่ implement method นี้ด้วย **pointer receiver** แล้วลอง assign `EmailNotifier{}` (value) ให้ตัวแปร `Notifier` ดูว่า error อะไรเกิดขึ้น จากนั้นแก้โดยใช้ `&EmailNotifier{}` แทน
4. สร้าง named type `type Meter float64` และ `type Feet float64` เขียน method `ToFeet()` ให้ `Meter` และ method `ToMeter()` ให้ `Feet` ทดลองเรียกใช้งานทั้งสองทิศทาง
5. เขียน struct `Point{ X, Y int }` แล้วสร้าง method value และ method expression ของ method `Add(other Point) Point` ทดลองพิมพ์ type ของตัวแปรที่ได้ด้วย `%T` เพื่อดูความต่างระหว่างสองรูปแบบ
6. อธิบายด้วยคำพูดของตัวเอง (เขียนเป็นคอมเมนต์ในไฟล์ .go ก็ได้) ว่าทำไม method set ของ `*T` ถึงกว้างกว่า `T` เสมอ และทำไมกฎนี้ถึงสำคัญมากตอนใช้งาน interface

---

**ต่อไป**: [Part 013 — Interfaces พื้นฐาน](./013-interfaces-basics.md)
