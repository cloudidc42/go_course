# Part 010: Pointers เจาะลึก

> ภาคที่ 1: พื้นฐานภาษา Go (Fundamentals) — ตอนที่ 10 จาก 15

## สารบัญของบทนี้

1. Pointer คืออะไร และทำไม Go ถึงมี pointer โดยไม่มี pointer arithmetic
2. Operator `&` และ `*`
3. `new()` เทียบกับ `&T{}`
4. Pass by value กับ pass by pointer — สาธิตการแก้ไขค่า
5. Pointer ไปยัง field ของ struct
6. เมื่อไรควรใช้ pointer receiver เทียบกับ value receiver (preview ของ Part 012)
7. Pointer กับ `nil`
8. Pitfall ที่พบบ่อย: คืน pointer ไปยังตัวแปร local ปลอดภัยใน Go เพราะ escape analysis
9. สรุปสิ่งที่ได้เรียนในบทนี้
10. แบบฝึกหัดท้ายบท

---

## 1. Pointer คืออะไร และทำไม Go ถึงมี pointer โดยไม่มี pointer arithmetic

ทุกตัวแปรในโปรแกรมถูกเก็บอยู่ในหน่วยความจำ (memory) ณ ตำแหน่งใดตำแหน่งหนึ่งที่มี **ที่อยู่ (address)** เฉพาะของตัวเอง **Pointer** คือตัวแปรชนิดพิเศษที่ไม่ได้เก็บ "ค่า" โดยตรง แต่เก็บ **ที่อยู่ในหน่วยความจำของตัวแปรอีกตัวหนึ่ง**

พูดง่ายๆ ถ้าตัวแปรปกติเปรียบเหมือนกล่องที่เก็บของไว้ข้างใน pointer ก็เปรียบเหมือนกระดาษโน้ตที่เขียนว่า "ของที่ต้องการอยู่ที่กล่องหมายเลขนี้" — pointer ไม่ได้เก็บของ แต่บอกตำแหน่งของกล่องที่เก็บของนั้นอยู่

### ทำไม Go ถึงต้องมี pointer?

จากที่เรียนมาตลอดหลักสูตรนี้ (Part 006 เรื่อง slice, Part 007 เรื่อง map, Part 008 เรื่อง function parameter) เราเห็นแล้วว่า **ค่าทุกอย่างที่ส่งเข้าฟังก์ชันใน Go ถูก copy เสมอ (pass by value)** ถ้าไม่มี pointer เลย การเขียนฟังก์ชันที่ต้องแก้ไขค่าตัวแปรต้นฉบับของผู้เรียก หรือฟังก์ชันที่ต้องทำงานกับข้อมูลก้อนใหญ่โดยไม่อยาก copy ทั้งก้อนทุกครั้ง จะทำไม่ได้เลยหรือทำได้ยากมาก

Pointer จึงเป็นเครื่องมือให้ Go:

1. **แก้ไขค่าต้นฉบับผ่านฟังก์ชันได้** โดยส่ง "ที่อยู่" ของตัวแปรเข้าไปแทนที่จะ copy ค่าทั้งก้อน
2. **หลีกเลี่ยงการ copy ข้อมูลขนาดใหญ่โดยไม่จำเป็น** เช่น struct ขนาดใหญ่ (จะเรียนเจาะลึกเรื่อง struct ใน Part 011) ที่ copy ทั้งก้อนทุกครั้งจะเปลืองทั้งเวลาและหน่วยความจำ
3. **แชร์ข้อมูลชุดเดียวกันระหว่างหลายส่วนของโปรแกรม** โดยไม่ต้องพึ่งตัวแปร global

### ทำไม Go ไม่มี pointer arithmetic เหมือน C?

ในภาษา C สามารถ "บวก/ลบ" ค่าของ pointer เพื่อเลื่อนตำแหน่งไปยัง memory address ข้างเคียงได้โดยตรง (เช่น `p + 1` เพื่อขยับไปยัง element ถัดไปของ array) ซึ่งเป็นความสามารถที่ทรงพลังแต่ก็อันตรายมาก เพราะการคำนวณ address ผิดพลาดแม้เพียงเล็กน้อยอาจทำให้โปรแกรมไปอ่าน/เขียนหน่วยความจำที่ไม่ได้เป็นของตัวเอง เกิด undefined behavior, ข้อมูลเสียหาย, หรือช่องโหว่ด้านความปลอดภัยร้ายแรง (buffer overflow) ซึ่งเป็นสาเหตุของบั๊กและช่องโหว่จำนวนมหาศาลในซอฟต์แวร์ที่เขียนด้วย C/C++ ตลอดหลายสิบปีที่ผ่านมา

Go เลือกตัดความสามารถนี้ออกไปโดยเจตนา ตามปรัชญา "less is more" ที่กล่าวไปใน Part 001 — **pointer ใน Go ทำได้แค่ "ชี้ไปยังตัวแปรหนึ่งตัว" และ "อ่าน/เขียนค่า ณ ตำแหน่งนั้น" เท่านั้น ไม่มีการคำนวณ address เองได้เลย** ทำให้ยังคงได้ประโยชน์หลักของ pointer (แก้ไขค่าต้นฉบับ, หลีกเลี่ยงการ copy ข้อมูลใหญ่) โดยตัดความเสี่ยงเรื่อง memory corruption ที่มาจากการคำนวณ address ผิดพลาดออกไปเกือบทั้งหมด ประกอบกับ garbage collector ที่ Go มีในตัว (ตามที่กล่าวไปใน Part 001) ทำให้โปรแกรมเมอร์ Go แทบไม่ต้องกังวลเรื่อง memory safety แบบที่โปรแกรมเมอร์ C ต้องเผชิญเลย

---

## 2. Operator `&` และ `*`

Go ใช้สัญลักษณ์สองตัวคู่กันในการทำงานกับ pointer:

- **`&`** (address-of operator): นำหน้าตัวแปร เพื่อขอ "ที่อยู่" ของตัวแปรนั้น ผลลัพธ์คือ pointer
- **`*`** (dereference operator): นำหน้า pointer เพื่อ "ตาม" ไปอ่าน/เขียนค่า ณ ที่อยู่ที่ pointer นั้นชี้ไป

`*` ยังใช้เป็นส่วนหนึ่งของ **type declaration** ของ pointer ด้วย เช่น `*int` หมายถึง "pointer ที่ชี้ไปยังค่า type `int`"

```go
package main

import "fmt"

func main() {
	x := 42
	p := &x // p คือ pointer ที่เก็บ "ที่อยู่" ของ x, type ของ p คือ *int

	fmt.Println("value of x:", x)
	fmt.Println("address of x:", p)
	fmt.Println("value pointed to by p:", *p) // dereference - อ่านค่า ณ ที่อยู่ที่ p ชี้ไป

	*p = 100 // แก้ไขค่า ณ ที่อยู่ที่ p ชี้ไป (ก็คือแก้ไข x นั่นเอง)
	fmt.Println("x after *p = 100:", x)

	fmt.Printf("type of p: %T\n", p)
}
```

ผลลัพธ์ (ค่า address จะแตกต่างกันไปในแต่ละครั้งที่รัน เพราะขึ้นกับตำแหน่งจริงในหน่วยความจำของเครื่อง ณ ขณะนั้น):

```
value of x: 42
address of x: 0xc000102040
value pointed to by p: 42
x after *p = 100: 100
type of p: *int
```

จุดสำคัญที่ต้องจำ: `p` และ `x` เป็นคนละตัวแปรกัน แต่ `p` "ชี้ไปหา" `x` ดังนั้นการแก้ไขผ่าน `*p = 100` จึงเปลี่ยนค่าของ `x` โดยตรง เพราะทั้งคู่อ้างอิงไปยังตำแหน่งหน่วยความจำเดียวกัน

หากพิมพ์ pointer ด้วย `fmt.Println(p)` โดยไม่ dereference จะได้ค่า address (ในรูปแบบ hexadecimal) แสดงออกมาตรงๆ ซึ่งไม่มีประโยชน์มากนักในการ debug ทั่วไป ปกติเราจะ dereference ด้วย `*p` เพื่อดูค่าจริงที่ pointer นั้นชี้ไปแทน

---

## 3. `new()` เทียบกับ `&T{}`

Go มี built-in function ชื่อ `new()` สำหรับจอง memory ให้กับ type ใดๆ แล้วคืน pointer ไปยัง **zero value** ของ type นั้น (zero value ตามที่เรียนไปใน Part 003)

```go
package main

import "fmt"

type Point struct {
	X, Y int
}

func main() {
	// new(T) จอง memory ให้ zero value ของ T แล้วคืน pointer ไปยังมัน
	p1 := new(Point)
	fmt.Println(p1, *p1) // &{0 0} {0 0}
	p1.X = 10
	fmt.Println(*p1) // {10 0}

	// &T{} สร้าง value พร้อมกำหนดค่าฟิลด์ แล้วคืน pointer ไปยัง value นั้นทันที
	p2 := &Point{X: 5, Y: 7}
	fmt.Println(*p2) // {5 7}

	// new(int) ก็ใช้ได้กับ primitive type เช่นกัน
	n := new(int)
	*n = 99
	fmt.Println(*n) // 99
}
```

ความแตกต่างระหว่างสองแนวทาง:

| | `new(Point)` | `&Point{X: 5, Y: 7}` |
|---|---|---|
| ค่าเริ่มต้น | zero value เสมอ (`{0 0}`) | กำหนดค่าฟิลด์ได้ทันที |
| การใช้งานทั่วไป | พบไม่บ่อยนักในโค้ดจริง | นิยมใช้กันอย่างแพร่หลาย |
| ใช้กับ primitive type | ได้ (`new(int)`, `new(string)`) | ใช้ไม่ได้ (`&int{}` ไม่มี syntax แบบนี้) |

ในทางปฏิบัติ โค้ด Go ส่วนใหญ่นิยมใช้ **`&T{...}`** มากกว่า `new(T)` เพราะสามารถกำหนดค่าเริ่มต้นของ field ไปพร้อมกับสร้าง pointer ได้ในบรรทัดเดียว อ่านง่ายและกระชับกว่า `new()` มักถูกใช้เฉพาะกับ primitive type ที่ต้องการ pointer เปล่าๆ ไปยัง zero value (เช่น `new(int)`) ซึ่งก็ยังพบไม่บ่อยนักเมื่อเทียบกับการประกาศตัวแปรแล้วใช้ `&` ตรงๆ

หมายเหตุ: `p1 := &Point{}` ก็ให้ผลลัพธ์เทียบเท่ากับ `p1 := new(Point)` ทุกประการ (ทั้งคู่ได้ pointer ไปยัง zero value ของ `Point`) เพียงแต่ `&T{}` เป็น syntax ที่ใช้ได้กว้างกว่าเพราะรองรับการกำหนดค่า field ได้ในตัว

---

## 4. Pass by value กับ pass by pointer — สาธิตการแก้ไขค่า

ตามที่กล่าวไปในหัวข้อที่ 1 การส่ง argument เข้าฟังก์ชันใน Go เป็น pass by value เสมอ มาดูตัวอย่างเปรียบเทียบชัดๆ ระหว่างการส่งค่าธรรมดา กับการส่ง pointer:

```go
package main

import "fmt"

func incByValue(n int) {
	n++ // แก้ไข copy ในเครื่อง ไม่กระทบต้นฉบับ
}

func incByPointer(n *int) {
	*n++ // แก้ไขผ่าน pointer กระทบต้นฉบับจริง
}

func main() {
	x := 10
	incByValue(x)
	fmt.Println("after incByValue:", x) // 10 - ไม่เปลี่ยน

	incByPointer(&x)
	fmt.Println("after incByPointer:", x) // 11 - เปลี่ยนแล้ว
}
```

`incByValue(x)` ส่ง**ค่า** ของ `x` เข้าไป ฟังก์ชันได้รับ copy อิสระ การ `n++` ภายในจึงแก้ไขแค่ copy นั้น ไม่กระทบ `x` ต้นฉบับเลย

`incByPointer(&x)` ส่ง**ที่อยู่**ของ `x` เข้าไป ฟังก์ชันได้รับ pointer ที่ชี้กลับไปหา `x` ตัวเดิม การ `*n++` (dereference แล้วบวกเพิ่ม) จึงแก้ไข `x` ต้นฉบับโดยตรง

นี่คือรูปแบบพื้นฐานที่สุดที่ทำให้ Go เขียนฟังก์ชันที่ "แก้ไขค่าต้นฉบับ" ได้ แม้จะไม่มี concept แบบ reference parameter ของภาษาอื่น (เช่น `ref` ใน C#) — Go ใช้ pointer แทนสำหรับ use case นี้เพียงอย่างเดียว

---

## 5. Pointer ไปยัง field ของ struct

Pointer ไม่ได้ชี้ไปหาแค่ตัวแปรทั้งตัวเท่านั้น แต่ยังชี้ไปยัง **field ของ struct โดยตรง**ได้ด้วย (struct จะเรียนเจาะลึกใน Part 011 แต่ในที่นี้ใช้แค่พื้นฐานเพื่อสาธิตการทำงานร่วมกับ pointer)

```go
package main

import "fmt"

type Rectangle struct {
	Width, Height int
}

func scale(r *Rectangle, factor int) {
	r.Width *= factor
	r.Height *= factor
}

func main() {
	rect := Rectangle{Width: 10, Height: 20}
	scale(&rect, 2)
	fmt.Println(rect) // {20 40}

	// pointer ไปยัง field ของ struct โดยตรง
	p := &rect.Width
	*p = 999
	fmt.Println(rect) // {999 40}
}
```

สังเกตว่าภายในฟังก์ชัน `scale` เราเขียน `r.Width` ตรงๆ ไม่ต้องเขียน `(*r).Width` ทั้งที่ `r` เป็น pointer (`*Rectangle`) ไม่ใช่ `Rectangle` โดยตรง — นี่เป็นเพราะ Go มี **automatic dereferencing** สำหรับการเข้าถึง field ผ่าน pointer: เขียน `r.Width` ก็เพียงพอแล้ว compiler จะแปลงเป็น `(*r).Width` ให้เองโดยอัตโนมัติ ทำให้โค้ดที่ทำงานกับ pointer ไปยัง struct อ่านและเขียนได้สะดวกเหมือนทำงานกับ struct ตรงๆ

`p := &rect.Width` แสดงให้เห็นว่า pointer ใน Go ชี้ไปยังส่วนย่อยของโครงสร้างข้อมูลได้ ไม่จำเป็นต้องชี้ไปทั้งก้อนเสมอไป การแก้ไขผ่าน `p` จึงกระทบ field `Width` ของ `rect` โดยตรง

---

## 6. เมื่อไรควรใช้ pointer receiver เทียบกับ value receiver (preview ของ Part 012)

เมื่อเรียนเรื่อง **method** ใน Part 012 จะพบว่า Go ให้เลือกได้ว่าจะประกาศ method ของ struct ด้วย **value receiver** (`func (c Counter) Method()`) หรือ **pointer receiver** (`func (c *Counter) Method()`) หลักการโดยสรุปที่ควรรู้ไว้ล่วงหน้า (รายละเอียดเต็มอยู่ใน Part 012):

```go
package main

import "fmt"

type Counter struct {
	value int
}

// pointer receiver - แก้ไข state ของ struct ต้นฉบับได้จริง
func (c *Counter) Increment() {
	c.value++
}

// value receiver - ทำงานกับ copy เท่านั้น ไม่กระทบต้นฉบับ
func (c Counter) Value() int {
	return c.value
}

func main() {
	c := Counter{}
	c.Increment()
	c.Increment()
	c.Increment()
	fmt.Println(c.Value()) // 3
}
```

หลักการเบื้องต้น: ถ้า method ต้อง**แก้ไขค่าภายใน struct** ให้ใช้ pointer receiver เสมอ (เหมือนหลักการ pass by pointer ในหัวข้อที่ 4 ทุกประการ — receiver ก็คือ parameter ตัวหนึ่งที่ซ่อนอยู่นั่นเอง) ส่วน method ที่แค่**อ่าน**ค่าโดยไม่แก้ไขอะไร จะใช้ value receiver หรือ pointer receiver ก็ได้ แต่มีข้อพิจารณาเพิ่มเติมเรื่องความสม่ำเสมอและประสิทธิภาพที่จะอธิบายแบบละเอียดพร้อมตัวอย่างเปรียบเทียบครบถ้วนใน **Part 012**

---

## 7. Pointer กับ `nil`

Zero value ของ pointer type ใดๆ คือ **`nil`** (หลักการ zero value เดียวกับที่เรียนไปใน Part 003 และเจออีกครั้งกับ slice ใน Part 006 และ map ใน Part 007) `nil` pointer หมายถึง "pointer ที่ยังไม่ได้ชี้ไปยังอะไรเลย"

```go
package main

import "fmt"

func main() {
	var p *int
	fmt.Println(p)        // <nil>
	fmt.Println(p == nil) // true

	if p != nil {
		fmt.Println(*p)
	} else {
		fmt.Println("p is nil, skip dereference")
	}
}
```

การ **dereference nil pointer จะทำให้เกิด panic ทันที** เพราะไม่มีตำแหน่งในหน่วยความจำจริงให้ตามไปอ่าน/เขียนค่า:

```go
package main

func main() {
	var p *int
	_ = *p // panic: runtime error: invalid memory address or nil pointer dereference
}
```

รันแล้วจะได้:

```
panic: runtime error: invalid memory address or nil pointer dereference
[signal SIGSEGV: segmentation violation code=0x1 addr=0x0 pc=0x...]
```

นี่คือ panic ที่พบบ่อยที่สุดอันดับต้นๆ ในโปรแกรม Go จริง โดยเฉพาะเมื่อทำงานกับ pointer ที่ได้มาจากฟังก์ชันที่อาจ return `nil` ได้ (เช่นค้นหาข้อมูลไม่เจอ) **กฎความปลอดภัยพื้นฐาน**: ก่อน dereference pointer ที่ไม่แน่ใจว่าเป็น `nil` หรือไม่ ให้เช็ค `if p != nil` เสมอ โดยเฉพาะ pointer ที่มาจาก:

- ผลลัพธ์ของฟังก์ชันที่อาจไม่พบข้อมูล (เช่นค้นหาใน slice/map แล้วไม่เจอ)
- field ของ struct ที่เป็น pointer type และยังไม่ได้ถูกกำหนดค่า
- parameter ของฟังก์ชันที่ผู้เรียกอาจส่ง `nil` เข้ามา

เราจะเห็นรูปแบบการเช็ค `nil` แบบนี้อีกครั้งเมื่อเรียนเรื่อง interface ใน Part 013-014 และเรื่อง error handling ใน Part 015-016 เพราะหลักการเดียวกันนี้ถูกใช้ซ้ำตลอดทั้งภาษา

---

## 8. Pitfall ที่พบบ่อย: คืน pointer ไปยังตัวแปร local ปลอดภัยใน Go เพราะ escape analysis

โปรแกรมเมอร์ที่มาจากภาษา C มักเข้าใจผิดหรือกังวลตอนเริ่มเขียน Go ว่า **การคืน pointer ไปยังตัวแปร local ของฟังก์ชันเป็นเรื่องอันตราย** เพราะใน C การทำแบบนี้เป็น **undefined behavior** ที่รู้จักกันดีในชื่อ "dangling pointer": เมื่อฟังก์ชันจบการทำงาน ตัวแปร local ที่อยู่บน **stack** จะถูกเคลียร์ทิ้งทันที pointer ที่ยังชี้ไปยังตำแหน่งนั้นจึงกลายเป็น pointer ที่ชี้ไปยังหน่วยความจำที่ไม่ได้เป็นของตัวเองอีกต่อไป การ dereference pointer นั้นในภายหลังจึงให้ผลลัพธ์ที่คาดเดาไม่ได้เลย

**แต่ใน Go การคืน pointer ไปยังตัวแปร local เป็นเรื่องปกติและปลอดภัยอย่างสมบูรณ์**:

```go
package main

import "fmt"

// คืนค่า pointer ไปยังตัวแปร local ปลอดภัยใน Go เพราะ escape analysis
func newCounter() *int {
	count := 0    // ตัวแปร local
	return &count // คืน pointer ไปยังตัวแปร local นี้ - ปลอดภัยเต็มที่ใน Go
}

func main() {
	c1 := newCounter()
	c2 := newCounter()

	*c1++
	*c1++
	*c2++

	fmt.Println(*c1, *c2) // 2 1 - คนละหน่วยความจำ ไม่ปะปนกัน
}
```

### ทำไมถึงปลอดภัย: escape analysis

Go compiler มีขั้นตอนที่เรียกว่า **escape analysis** ซึ่งทำงานตอน compile time โดยจะวิเคราะห์ทุกตัวแปรในโปรแกรมว่า **"ตัวแปรนี้จำเป็นต้องมีชีวิตอยู่นานกว่าฟังก์ชันที่สร้างมันหรือไม่"** ถ้าคำตอบคือใช่ (เช่น มีการคืน pointer ของตัวแปรนั้นออกจากฟังก์ชัน หรือมีการเก็บ pointer ของมันไว้ในโครงสร้างข้อมูลที่มีอายุยืนกว่า) compiler จะตัดสินใจ **จัดสรรตัวแปรนั้นไว้บน heap แทนที่จะเป็น stack** โดยอัตโนมัติ พูดกันในภาษาทั่วไปว่าตัวแปรนั้น "escape ไปที่ heap"

ในตัวอย่างข้างบน ตัวแปร `count` ถูก `&count` ส่งออกจากฟังก์ชัน `newCounter` ไป escape analysis จึงตัดสินใจให้ `count` อยู่บน heap แทนที่จะเป็น stack ของฟังก์ชัน `newCounter` ดังนั้นเมื่อ `newCounter` ทำงานจบและ stack frame ของมันถูกเคลียร์ ตัวแปร `count` บน heap ยังคงอยู่ต่อไปตราบใดที่ยังมี pointer อ้างอิงถึงมันอยู่ (ในที่นี้คือ `c1` และ `c2` ใน `main`) และ **garbage collector** ของ Go (ตามที่กล่าวไปใน Part 001) จะคอยเก็บกวาดหน่วยความจำนั้นทิ้งเองโดยอัตโนมัติเมื่อไม่มี pointer ใดอ้างอิงถึงมันอีกต่อไป

### เปรียบเทียบกับ C

| | C | Go |
|---|---|---|
| ตัวแปร local อยู่ที่ไหน | เสมอบน stack | stack หรือ heap แล้วแต่ escape analysis ตัดสินใจ |
| คืน pointer ไปยังตัวแปร local | **Undefined behavior** (dangling pointer) | ปลอดภัย — compiler ย้ายไป heap ให้อัตโนมัติ |
| ใครจัดการปล่อยหน่วยความจำ | โปรแกรมเมอร์ต้องจัดการเอง (หรือปล่อยรั่ว) | Garbage collector จัดการให้อัตโนมัติ |
| ผู้เขียนโปรแกรมต้องคิดเรื่อง stack/heap ไหม | ต้องคิดเองเสมอ | ปกติไม่ต้องคิด — compiler ตัดสินใจให้ |

จุดสำคัญคือ **ในฐานะโปรแกรมเมอร์ Go เราแทบไม่จำเป็นต้องคิดเรื่อง stack กับ heap เองเลยในการเขียนโปรแกรมทั่วไป** — เขียนโค้ดตามความหมายทางตรรกะที่ต้องการได้เลย (คืน pointer เมื่อจำเป็นต้องคืน) แล้วปล่อยให้ compiler ตัดสินใจเรื่องการจัดสรรหน่วยความจำที่ถูกต้องและปลอดภัยให้เอง นี่คือหนึ่งในเหตุผลสำคัญที่ทำให้ Go เขียนโปรแกรมได้เร็วและปลอดภัยกว่า C มาก โดยไม่ต้องแลกด้วย garbage collector ที่หนักเกินไป (ตรงข้ามกับภาษาที่มี GC จำนวนมาก Go ออกแบบ GC ให้มี latency ต่ำมาก เหมาะกับงาน backend/infrastructure ที่ต้องการ response time สม่ำเสมอ — จะเรียนเจาะลึกเรื่องนี้ใน Part 055)

**ข้อควรรู้เพิ่มเติม (ไม่ต้องกังวลตอนนี้)**: การที่ตัวแปรจำนวนมาก "escape" ไป heap มีผลด้าน performance เพราะการจัดสรรและเก็บกวาดหน่วยความจำบน heap มีค่าใช้จ่ายสูงกว่าการใช้ stack ล้วนๆ ในโปรแกรมที่ต้องการ performance สูงสุด นักพัฒนาระดับสูงอาจใช้คำสั่ง `go build -gcflags="-m"` เพื่อดูว่าตัวแปรใดบ้าง escape ไป heap และพยายามลดจำนวนนั้นลง — เนื้อหานี้จะกลับมาเจาะลึกอีกครั้งในภาคที่ 7 เรื่อง Performance (โดยเฉพาะ Part 085)

---

## สรุปสิ่งที่ได้เรียนในบทนี้

- Pointer คือตัวแปรที่เก็บ "ที่อยู่ในหน่วยความจำ" ของตัวแปรอีกตัวหนึ่ง ทำให้แก้ไขค่าต้นฉบับผ่านฟังก์ชันได้ และหลีกเลี่ยงการ copy ข้อมูลขนาดใหญ่โดยไม่จำเป็น
- Go มี pointer แต่**ไม่มี pointer arithmetic** เหมือน C เพื่อตัดความเสี่ยงเรื่อง memory corruption โดยยังคงประโยชน์หลักของ pointer ไว้
- `&x` คือการขอที่อยู่ของ `x` (ได้ pointer) ส่วน `*p` คือการ dereference เพื่ออ่าน/เขียนค่า ณ ที่อยู่ที่ `p` ชี้ไป
- `new(T)` จองหน่วยความจำและคืน pointer ไปยัง zero value ของ `T` ส่วน `&T{...}` สร้าง value พร้อมกำหนดค่าฟิลด์แล้วคืน pointer ทันที — `&T{...}` เป็นที่นิยมมากกว่าในทางปฏิบัติ
- การส่งค่าธรรมดาเข้าฟังก์ชันแก้ไขแค่ copy ไม่กระทบต้นฉบับ ในขณะที่การส่ง pointer ทำให้ฟังก์ชันแก้ไขค่าต้นฉบับได้โดยตรง
- Pointer ชี้ไปยัง field ของ struct ได้โดยตรง และ Go มี automatic dereferencing ทำให้เข้าถึง field ผ่าน pointer เขียนเหมือนเข้าถึงผ่าน struct ตรงๆ
- Method ที่ต้องแก้ไขค่าภายใน struct ควรใช้ pointer receiver (รายละเอียดเต็มใน Part 012)
- Zero value ของ pointer คือ `nil` — dereference nil pointer ทำให้ panic ทันที ต้องเช็ค `!= nil` ก่อนเสมอเมื่อไม่แน่ใจ
- การคืน pointer ไปยังตัวแปร local เป็นเรื่องปลอดภัยอย่างสมบูรณ์ใน Go เพราะ **escape analysis** จะย้ายตัวแปรนั้นไปอยู่บน heap โดยอัตโนมัติเมื่อจำเป็น ต่างจาก C ที่เป็น undefined behavior (dangling pointer) โดยสิ้นเชิง

## แบบฝึกหัดท้ายบท

1. เขียนฟังก์ชัน `swap(a, b *int)` ที่สลับค่าของตัวแปรสองตัวผ่าน pointer แล้วทดสอบว่าค่าต้นฉบับใน `main` ถูกสลับจริงหรือไม่
2. เขียน struct `BankAccount` ที่มี field `Balance float64` แล้วเขียนฟังก์ชัน `deposit(acc *BankAccount, amount float64)` และ `withdraw(acc *BankAccount, amount float64) error` (คืน error ถ้าถอนเกินยอดคงเหลือ) โดยใช้ pointer เพื่อแก้ไข `Balance` ต้นฉบับ
3. เขียนฟังก์ชันที่รับ `*int` เป็น parameter และมีการเช็ค `nil` ก่อน dereference เสมอ แล้วทดลองเรียกทั้งแบบส่ง pointer จริงและส่ง `nil` เข้าไป พิสูจน์ว่าโปรแกรมไม่ panic
4. เขียนฟังก์ชัน `makePair() (*int, *int)` ที่คืน pointer ไปยังตัวแปร local สองตัวที่ไม่เกี่ยวข้องกัน แล้วพิสูจน์ด้วยการรันจริงว่าทั้งสอง pointer เป็นอิสระต่อกัน ไม่แชร์หน่วยความจำเดียวกัน
5. อธิบายด้วยคำพูดตัวเองว่า escape analysis คืออะไร และทำไมการคืน pointer ไปยังตัวแปร local ถึงปลอดภัยใน Go แต่เป็นอันตรายใน C (เตรียมคำตอบไว้ให้ชัดเจน เพราะเป็นคำถามสัมภาษณ์งาน Go ที่พบบ่อยมาก)
6. ลองรันคำสั่ง `go build -gcflags="-m" .` กับโค้ดในแบบฝึกหัดข้อ 4 แล้วสังเกตข้อความที่ compiler รายงาน เช่น `moved to heap: count` ซึ่งยืนยันว่า escape analysis ตัดสินใจย้ายตัวแปรนั้นไปไว้บน heap จริง

---

**ต่อไป**: [Part 011 — Structs เจาะลึก](./011-structs.md)
