# Part 030: Struct Embedding และ Interface Embedding

> ภาคที่ 2: ระดับกลาง (Intermediate) — ตอนที่ 15 จาก 20 (Part 16–35)

## สารบัญของบทนี้

1. ทบทวน: ทำไม Go ไม่มี Class/Inheritance
2. Struct Embedding: Anonymous Field
3. Field Promotion และ Method Promotion
4. การพิสูจน์สำคัญ: Embedding ไม่ใช่ Inheritance (ไม่มี Polymorphism)
5. Embedding คือ Composition: "Has-A ผ่านการ Promote" ไม่ใช่ "Is-A"
6. Name Conflict และการแก้ความกำกวมแบบ Explicit
7. ทบทวน Interface Embedding จาก Part 014
8. Embedding เพื่อ Satisfy Interface บางส่วน: ห่อ `io.Reader`
9. เมื่อไรควรใช้ Embedding และเมื่อไรไม่ควร
10. สรุปสิ่งที่ได้เรียนในบทนี้
11. แบบฝึกหัดท้ายบท

---

## 1. ทบทวน: ทำไม Go ไม่มี Class/Inheritance

จำได้ไหมว่าใน **Part 001** เราพูดถึงปรัชญาการออกแบบของ Go ข้อหนึ่งคือ **"ไม่มี class / inheritance — ใช้ `struct` + `interface` + `composition` แทน"** บทนี้คือจุดที่เราจะเห็นกลไกที่ทำให้ประโยคนั้นเป็นจริงในทางปฏิบัติอย่างชัดเจนที่สุด

ในภาษาที่มี OOP แบบคลาสสิก (Java, C++, Python) เมื่อ class `Dog` **inherit** จาก class `Animal` จะได้สองสิ่งมาพร้อมกันเสมอ:

1. **Field/method ของ `Animal` ถูกนำมาใช้ใน `Dog` ได้** (code reuse)
2. **Polymorphism ผ่าน virtual dispatch** — ถ้า `Animal` มี method ที่เรียก method อื่นของตัวเองภายใน และ `Dog` override method นั้น การเรียกจะ**ย้อนไปเรียก method ของ `Dog`** แม้ว่าโค้ดที่เรียกจะอยู่ใน `Animal` ก็ตาม

Go มี **struct embedding** ที่ให้ข้อ 1 (code reuse ผ่าน field/method promotion) แต่**ไม่ให้ข้อ 2 เลย** — นี่คือความแตกต่างที่สำคัญที่สุดที่ทำให้ embedding ไม่ใช่ inheritance และเป็นสิ่งที่ผู้ที่มาจากภาษา OOP คลาสสิกมักเข้าใจผิดบ่อยที่สุด บทนี้จะพิสูจน์ให้เห็นด้วยโค้ดจริงในหัวข้อที่ 4

---

## 2. Struct Embedding: Anonymous Field

**Struct embedding** ทำได้โดยประกาศ field ใน struct โดย**ไม่ตั้งชื่อ field** — ใช้แค่ชื่อ type ตรงๆ เรียกว่า **anonymous field**:

```go
package main

import "fmt"

type Engine struct {
	Horsepower int
}

func (e Engine) Start() string {
	return fmt.Sprintf("เครื่องยนต์สตาร์ท (%d แรงม้า)", e.Horsepower)
}

// Car embed Engine โดยไม่ตั้งชื่อ field (anonymous field)
type Car struct {
	Engine // anonymous field - field/method ของ Engine ถูก "promote" ขึ้นมาที่ Car
	Brand  string
}

func main() {
	c := Car{
		Engine: Engine{Horsepower: 150},
		Brand:  "Toyota",
	}

	// เข้าถึง field ที่ promote มาได้ตรงๆ เหมือนเป็น field ของ Car เอง
	fmt.Println("horsepower:", c.Horsepower)
	// เข้าถึงแบบเต็มผ่านชื่อ type ที่ embed ก็ยังทำได้เสมอ
	fmt.Println("horsepower (full path):", c.Engine.Horsepower)

	// method ก็ถูก promote เช่นกัน
	fmt.Println(c.Start())
	fmt.Println(c.Engine.Start())
}
```

ผลลัพธ์:

```
horsepower: 150
horsepower (full path): 150
เครื่องยนต์สตาร์ท (150 แรงม้า)
เครื่องยนต์สตาร์ท (150 แรงม้า)
```

สังเกตความแตกต่างของ syntax ระหว่าง field ปกติกับ anonymous field:

```go
type Car struct {
	Engine        // anonymous field: ใช้แค่ชื่อ type
	Brand  string // field ปกติ: ต้องมีทั้งชื่อ field และ type
}
```

**ชื่อของ anonymous field คือชื่อ type นั้นเอง** (ในที่นี้คือ `Engine`) นี่คือเหตุผลที่เข้าถึงแบบเต็มได้ผ่าน `c.Engine.Horsepower` — ไม่ใช่ว่า field นี้ "ไม่มีชื่อ" จริงๆ แต่เป็นการ**ใช้ชื่อ type แทนการตั้งชื่อเอง**

---

## 3. Field Promotion และ Method Promotion

จากตัวอย่างข้างบน จะเห็นว่า `c.Horsepower` ใช้งานได้ทั้งที่ `Horsepower` เป็น field ของ `Engine` ไม่ใช่ของ `Car` โดยตรง ปรากฏการณ์นี้เรียกว่า **field promotion** — Go "เลื่อนขั้น" field และ method ของ struct ที่ถูก embed ให้เข้าถึงได้ราวกับเป็นของ struct แม่โดยตรง

กฎการทำงานของ promotion:

- **Field promotion**: field ทุกตัวของ struct ที่ถูก embed จะเข้าถึงได้ผ่าน struct แม่โดยตรง โดยไม่ต้องระบุชื่อ type ของ embedded struct
- **Method promotion**: method ทุกตัว (ทั้ง value receiver และ pointer receiver ตามกฎที่เรียนใน **Part 012**) ของ struct ที่ถูก embed จะถูก promote ขึ้นมาเช่นกัน — เป็นกลไกเดียวกับที่เราเห็นใน `ThaiDate` ที่ embed `time.Time` ใน **Part 025** ทำให้เรียก `.Format()` ของ `time.Time` ผ่าน `ThaiDate` ได้โดยตรง
- Promotion ทำงานได้**หลายชั้น** — ถ้า struct `A` embed `B` และ `B` embed `C`, field/method ของ `C` จะถูก promote ขึ้นมาถึง `A` ด้วย (ผ่าน `B` เป็นตัวกลาง)
- การเข้าถึงแบบเต็ม (`c.Engine.Horsepower`) **ใช้ได้เสมอ** ไม่ว่าจะ promote ได้หรือไม่ก็ตาม — นี่คือทางออกสำคัญเมื่อเกิด name conflict ที่จะเห็นในหัวข้อที่ 6

จุดสำคัญที่ต้องเข้าใจให้ชัด: **promotion เป็นแค่ทางลัดของ syntax (syntactic sugar)** ที่ compiler แปลง `c.Horsepower` ให้เป็น `c.Engine.Horsepower` ให้อัตโนมัติเบื้องหลัง — มันไม่ได้เปลี่ยนแปลงโครงสร้างข้อมูลจริงเลย `Car` ยังคงมี field ชื่อ `Engine` อยู่ข้างในเหมือนเดิม ไม่ได้ "flatten" ข้อมูลของ `Engine` เข้ามารวมกับ `Car` แต่อย่างใด

---

## 4. การพิสูจน์สำคัญ: Embedding ไม่ใช่ Inheritance (ไม่มี Polymorphism)

นี่คือหัวใจสำคัญที่สุดของบทนี้ มาดูตัวอย่างที่พิสูจน์ให้เห็นชัดเจนว่า **embedding ไม่มี virtual dispatch แบบ OOP inheritance**:

```go
package main

import "fmt"

type Animal struct {
	Name string
}

func (a Animal) Speak() string {
	return a.Name + " ส่งเสียงร้องทั่วไป"
}

// Describe เรียก a.Speak() ภายใน - นี่คือ method ของ Animal เอง
// ไม่รู้จักและไม่สนใจว่าใครมา embed ตัวเองไปใช้
func (a Animal) Describe() string {
	return "รายงาน: " + a.Speak()
}

type Dog struct {
	Animal // embed Animal
}

// Dog "override" Speak - แต่จริงๆ แล้วนี่คือการประกาศ method Speak ใหม่ของ Dog
// ไม่ใช่การ override แบบ OOP virtual method
func (d Dog) Speak() string {
	return d.Name + " เห่า โฮ่งๆ"
}

func main() {
	d := Dog{Animal: Animal{Name: "โปโป้"}}

	// เมื่อเรียกผ่าน Dog โดยตรง Go เลือก method ของ Dog เอง (ใกล้ที่สุดใน struct) ถูกต้องตามคาด
	fmt.Println(d.Speak()) // โปโป้ เห่า โฮ่งๆ

	// แต่ Describe() ถูกนิยามอยู่บน Animal และภายในมันเรียก a.Speak()
	// ซึ่ง ณ ตอนที่ Describe ถูก compile นั้น a มี static type เป็น Animal เท่านั้น
	// Go ไม่มีกลไก "virtual dispatch" แบบ OOP inheritance ที่จะย้อนกลับไปเรียก Dog.Speak()
	// ผลคือ Describe() ยังคงเรียก Animal.Speak() เดิม ไม่ใช่ Dog.Speak() ที่ override ไว้
	fmt.Println(d.Describe()) // รายงาน: โปโป้ ส่งเสียงร้องทั่วไป (ไม่ใช่ "เห่า โฮ่งๆ")
}
```

ผลลัพธ์:

```
โปโป้ เห่า โฮ่งๆ
รายงาน: โปโป้ ส่งเสียงร้องทั่วไป
```

**นี่คือจุดที่ทำให้คนที่มาจาก Java/C++/Python "งง" มากที่สุดตอนเริ่มเขียน Go** ถ้าเป็นภาษา OOP แบบคลาสสิก การเรียก `d.Describe()` ควรจะพิมพ์ `"รายงาน: โปโป้ เห่า โฮ่งๆ"` เพราะ `Dog` override `Speak()` ไว้แล้ว การเรียก `Speak()` จากภายใน `Describe()` ควรจะ"มองเห็น" การ override นั้นผ่านกลไก virtual dispatch

แต่ใน Go **ไม่มีสิ่งนั้นเลย** เหตุผลเชิงเทคนิคคือ:

- Method `Describe()` ถูกประกาศบน type `Animal` และภายใน method นั้น ตัวแปร receiver `a` มี **static type เป็น `Animal` ตลอดไป** ไม่เปลี่ยนเป็น `Dog` แม้ว่าตอนเรียกจริงจะเรียกผ่าน `Dog` ก็ตาม
- เมื่อ `Describe()` เรียก `a.Speak()` ข้างใน Go compiler ผูก (bind) การเรียกนี้เข้ากับ `Animal.Speak()` **ตั้งแต่ตอน compile** เพราะ `a` คือ `Animal` เสมอในสายตาของ compiler ไม่มีขั้นตอนใดที่จะ "มองย้อนกลับ" ไปหา type ภายนอกที่ embed ตัวเองอยู่ได้เลย
- การที่ `Dog` มี method ชื่อ `Speak()` ของตัวเอง**ไม่ใช่การ override** ในความหมายของ OOP แต่คือการ**ประกาศ method ใหม่ที่บังเอิญชื่อซ้ำกัน** และเพราะ method ของ `Dog` เอง "ใกล้ตัว" กว่า method ที่ promote มาจาก `Animal` เวลาเรียก `d.Speak()` ตรงๆ Go จึงเลือก method ของ `Dog` ก่อนเสมอ (กฎการเลือก method ที่ใกล้ที่สุด) — แต่กฎนี้ใช้ได้แค่ตอนเรียกผ่าน `Dog` โดยตรงเท่านั้น ไม่ส่งผลย้อนกลับไปยัง method ที่ประกาศบน `Animal` เลย

---

## 5. Embedding คือ Composition: "Has-A ผ่านการ Promote" ไม่ใช่ "Is-A"

จากการพิสูจน์ในหัวข้อก่อน สรุปหลักการสำคัญได้ว่า: **struct embedding คือเครื่องมือสำหรับ composition ("has-a" relationship ที่มี syntax สะดวกขึ้นผ่าน promotion) ไม่ใช่เครื่องมือสำหรับสร้างความสัมพันธ์แบบ "is-a" เหมือน class inheritance**

เปรียบเทียบมุมมองทั้งสองแบบ:

| มุมมอง | ความหมาย | ตัวอย่างในหัวข้อ 4 |
|---|---|---|
| "is-a" (Inheritance แบบ OOP) | `Dog` **เป็น** `Animal` ชนิดหนึ่ง — ทุกที่ที่ใช้ `Animal` ได้ ใช้ `Dog` แทนได้ (Liskov Substitution) พร้อม behavior ที่ override ไว้ทำงานถูกต้องเสมอ | ไม่เป็นจริงใน Go — `Describe()` ที่เรียกผ่าน `Dog` ยังคงพฤติกรรมของ `Animal` เดิม |
| "has-a" (Composition แบบ Go) | `Dog` **มี** `Animal` อยู่ข้างใน (เป็น field หนึ่ง) และได้ field/method ของ `Animal` มาใช้แบบสะดวก (ผ่าน promotion) แต่ทั้งสองยังเป็นคนละ type กันอย่างชัดเจน | `Dog` ไม่ใช่ `Animal` — ส่ง `Dog` ไปยังฟังก์ชันที่รับ parameter type `Animal` ตรงๆ ไม่ได้เลย ต้องแปลงหรือส่ง `d.Animal` แทน |

ปรัชญาเบื้องหลังคือคำพูดที่มีชื่อเสียงในวงการออกแบบซอฟต์แวร์: **"Favor composition over inheritance"** (เลือก composition มากกว่า inheritance) — Go ผลักดันแนวคิดนี้ไปอีกขั้นด้วยการ**ไม่มี inheritance ให้เลือกใช้เลยตั้งแต่แรก** บังคับให้นักพัฒนาคิดแบบ composition มาตั้งแต่ต้น ซึ่งในระยะยาวมักนำไปสู่โค้ดที่ยืดหยุ่นกว่า เพราะไม่มีปัญหาคลาสสิกของ inheritance เช่น "fragile base class problem" (การแก้ class แม่กระทบ class ลูกที่คาดไม่ถึง) หรือ deep inheritance hierarchy ที่ตามรอย behavior ยาก

---

## 6. Name Conflict และการแก้ความกำกวมแบบ Explicit

เมื่อ struct หนึ่ง embed มากกว่าหนึ่ง type ที่บังเอิญมี field หรือ method ชื่อเดียวกัน จะเกิด **name conflict** — Go จัดการเรื่องนี้ด้วยการ**บังคับให้ระบุ path แบบเต็มเสมอ** ไม่ยอมเดาให้เองว่าจะหมายถึงตัวไหน:

```go
package main

import "fmt"

type Base struct {
	ID int
}

func (b Base) Describe() string {
	return fmt.Sprintf("Base#%d", b.ID)
}

type Logger struct {
	ID int // ชื่อ field ชนกับ Base.ID
}

func (l Logger) Describe() string {
	return fmt.Sprintf("Logger#%d", l.ID)
}

// Combined embed ทั้ง Base และ Logger ซึ่งมี field/method ชื่อชนกัน
type Combined struct {
	Base
	Logger
}

func main() {
	c := Combined{Base: Base{ID: 1}, Logger: Logger{ID: 2}}

	// c.ID และ c.Describe() แบบสั้นจะ "ambiguous selector" เพราะ Go หาไม่ได้ว่าจะเอาของใคร
	// (บรรทัดด้านล่างถ้าเปิด comment จะ compile error: ambiguous selector c.ID)
	// fmt.Println(c.ID)
	// fmt.Println(c.Describe())

	// ต้องระบุ path เต็มเพื่อแก้ความกำกวมเสมอ
	fmt.Println(c.Base.ID, c.Logger.ID)
	fmt.Println(c.Base.Describe())
	fmt.Println(c.Logger.Describe())
}
```

ผลลัพธ์:

```
1 2
Base#1
Logger#2
```

ถ้าลองเปิด comment บรรทัด `fmt.Println(c.ID)` จะได้ error:

```
./main.go:15:16: ambiguous selector c.ID
```

**กฎการเลือก field/method เมื่อมี embedding หลายชั้น** (สำคัญเพราะไม่ใช่ทุก conflict จะ error): Go มีกฎเรื่อง **"depth"** (ความลึกของการ embed) — ถ้า field/method ที่ชื่อชนกันอยู่ **คนละ depth** กัน (เช่น field ตรงของ struct แม่ ปะทะกับ field ที่ promote มาจาก embedded struct) ตัวที่ depth ตื้นกว่า (ใกล้ตัว struct แม่มากกว่า) จะถูกเลือกโดยอัตโนมัติโดยไม่ error แต่ถ้าชื่อชนกันที่ **depth เดียวกันพอดี** (แบบตัวอย่างข้างบนที่ `Base` และ `Logger` อยู่ระดับเดียวกันใน `Combined`) จะเป็น ambiguous selector ทันที บังคับให้ต้องระบุ path เต็มเสมอ — พฤติกรรมนี้เหมือนกับกฎการเลือก method ที่ใกล้ที่สุดที่เห็นในหัวข้อที่ 4 (`Dog.Speak()` ที่ depth 0 ชนะ `Animal.Speak()` ที่ depth 1 โดยอัตโนมัติ)

---

## 7. ทบทวน Interface Embedding จาก Part 014

ใน **Part 014** เรากล่าวถึง interface embedding ไว้สั้นๆ ตอนนี้มาดูรายละเอียดให้ครบถ้วน **Interface embedding** คือการนำ interface อื่นมาประกาศไว้ข้างในนิยามของ interface ใหม่ เพื่อรวม method set เข้าด้วยกัน:

```go
package main

import "fmt"

type Reader interface {
	Read() string
}

type Writer interface {
	Write(s string)
}

// ReadWriter embed interface อื่นเข้ามาเป็น "interface embedding"
// type ใดจะ implement ReadWriter ได้ต้องมีทั้ง method Read() และ Write() ครบ
type ReadWriter interface {
	Reader
	Writer
}

type MemoryBuffer struct {
	data string
}

func (m *MemoryBuffer) Read() string {
	return m.data
}

func (m *MemoryBuffer) Write(s string) {
	m.data += s
}

func main() {
	var rw ReadWriter = &MemoryBuffer{}
	rw.Write("hello ")
	rw.Write("world")
	fmt.Println(rw.Read())
}
```

ผลลัพธ์:

```
hello world
```

`ReadWriter` ไม่มี method ของตัวเองประกาศเพิ่มเลย มีแค่ `Reader` กับ `Writer` ซ้อนอยู่ข้างใน — ผลคือ `ReadWriter` มี **method set เท่ากับผลรวมของทั้งสอง interface** (`Read()` และ `Write()`) type ใดก็ตามที่ implement ทั้งสอง method นี้ครบจะ satisfy `ReadWriter` โดยอัตโนมัติ (ทบทวนหลักการ implicit interface satisfaction จาก **Part 013**)

**นี่คือแพทเทิร์นเดียวกับที่ standard library ของ Go ใช้จริงเป็นประจำ** — ตัวอย่างที่โด่งดังที่สุดคือใน package `io` (จะเรียนเจาะลึกใน **Part 048**):

```go
type ReadWriter interface {
	Reader
	Writer
}
```

`io.ReadWriter`, `io.ReadCloser`, `io.WriteCloser` ล้วนสร้างจากการ embed `io.Reader`, `io.Writer`, `io.Closer` เข้าด้วยกันในรูปแบบต่างๆ ทั้งสิ้น — interface embedding คือเครื่องมือสำคัญที่ทำให้ Go ออกแบบ interface เล็กๆ (small interfaces) หลายตัวแล้วประกอบ (compose) เป็น interface ใหญ่ขึ้นได้ตามต้องการ แทนที่จะต้องเขียน interface ใหญ่ตายตัวไว้ล่วงหน้า

---

## 8. Embedding เพื่อ Satisfy Interface บางส่วน: ห่อ `io.Reader`

เทคนิคที่ทรงพลังอย่างหนึ่งที่ใช้ struct embedding คือการ **embed interface เข้าไปใน struct โดยตรง** (ไม่ใช่ embed struct ธรรมดา) เพื่อ "ยืม" การ implement interface นั้นมาใช้ฟรีๆ แล้วค่อย override เฉพาะ method ที่ต้องการเปลี่ยนพฤติกรรม — เทคนิคนี้เรียกกันทั่วไปว่า **decorator pattern** หรือ **wrapper pattern**

```go
package main

import (
	"bufio"
	"fmt"
	"io"
	"strings"
)

// CountingReader embed io.Reader (interface) เข้ามาโดยตรง
// ทำให้ CountingReader ได้ implement io.Reader "ฟรี" ผ่านการ promote method Read
// จากนั้นเรา override เฉพาะ Read เพื่อนับจำนวน byte ที่อ่านไปโดยไม่ต้อง implement ส่วนอื่นเพิ่ม
type CountingReader struct {
	io.Reader // embed interface - ไม่ใช่ struct
	BytesRead int
}

func (c *CountingReader) Read(p []byte) (int, error) {
	n, err := c.Reader.Read(p) // เรียก method ของ io.Reader ตัวจริงที่ห่ออยู่ข้างใน
	c.BytesRead += n
	return n, err
}

func main() {
	src := strings.NewReader("สวัสดีชาวโลก Go เจ๋งมาก")
	cr := &CountingReader{Reader: src}

	scanner := bufio.NewScanner(cr)
	scanner.Split(bufio.ScanWords)
	for scanner.Scan() {
		fmt.Println("word:", scanner.Text())
	}
	fmt.Println("total bytes read:", cr.BytesRead)

	// เพราะ CountingReader "เป็น" io.Reader (ผ่านการห่อ embedding)
	// จึงส่งเข้าฟังก์ชันที่ต้องการ io.Reader เช่น io.ReadAll ได้โดยตรง
	src2 := strings.NewReader("more data")
	cr2 := &CountingReader{Reader: src2}
	all, _ := io.ReadAll(cr2)
	fmt.Println(string(all), "| bytes:", cr2.BytesRead)
}
```

ผลลัพธ์:

```
word: สวัสดีชาวโลก
word: Go
word: เจ๋งมาก
total bytes read: 61
more data | bytes: 9
```

จุดสำคัญที่ต้องเข้าใจในเทคนิคนี้:

- `io.Reader` ที่ถูก embed เป็น**ค่า interface** ไม่ใช่ struct — ตอนสร้าง `CountingReader{Reader: src}` เรากำหนดค่า field `Reader` (ชื่อ field คือชื่อ interface ตามกฎ anonymous field เดียวกับหัวข้อที่ 2) ให้เป็น `strings.NewReader(...)` ที่ implement `io.Reader` อยู่แล้ว
- เพราะ `io.Reader` มี method `Read([]byte) (int, error)` ถูก embed เข้ามา `CountingReader` จึงได้ method `Read` แบบ promoted มาโดยอัตโนมัติ ทำให้ **`CountingReader` satisfy interface `io.Reader` ได้ทันที** โดยที่เรายังไม่ต้องเขียนอะไรเพิ่มเลย
- แต่เราต้องการเปลี่ยนพฤติกรรม `Read` ให้นับ byte ด้วย จึงประกาศ method `Read` ของ `CountingReader` เอง**ทับ**ตัวที่ promote มา (ตามกฎ "method ใกล้ตัวชนะ" จากหัวข้อที่ 4 และ 6) แล้วเรียก `c.Reader.Read(p)` (ของจริงที่ห่ออยู่ข้างใน) ต่อไปข้างในเพื่อยังทำงานตามปกติ พร้อมเพิ่ม logic นับ byte เข้าไป
- ผลลัพธ์คือ `CountingReader` **ใช้แทน `io.Reader` ตัวไหนก็ได้ในระบบ** (ส่งเข้า `bufio.NewScanner`, `io.ReadAll` หรือฟังก์ชันอื่นที่รับ `io.Reader` เป็น parameter) โดยเพิ่มความสามารถนับ byte เข้าไปแบบโปร่งใส (transparent) — นี่คือรูปแบบการ **satisfy interface บางส่วนแล้ว override เฉพาะจุดที่ต้องการ** ที่ใช้กันมากใน Go โดยเฉพาะใน middleware pattern ของ web server (จะเรียนใน **Part 057**)

หลักการนี้ต่างจากตัวอย่าง `Dog`/`Animal` ตรงที่ **ในกรณีนี้เราตั้งใจ override เพื่อเปลี่ยนพฤติกรรมของ method นั้นเองโดยตรง ไม่ได้หวังให้ method อื่นที่เรียกผ่าน interface เดิม "มองเห็น" การ override แบบ virtual dispatch** — ผู้เรียก `CountingReader.Read()` ตรงๆ (หรือผ่านตัวแปร type `io.Reader` ที่ชี้ไปที่ `CountingReader`) จะได้ method ที่ override แล้วเสมอ เพราะเรียกผ่าน **dynamic dispatch ของ interface เอง** (ทบทวนจาก **Part 013**) ซึ่งเป็นกลไกคนละอย่างกับ field/method promotion ของ struct embedding ที่ไม่มี dynamic dispatch เลยตามที่พิสูจน์ในหัวข้อที่ 4

---

## 9. เมื่อไรควรใช้ Embedding และเมื่อไรไม่ควร

| สถานการณ์ | ควรใช้ Embedding หรือไม่ |
|---|---|
| ต้องการ "ยืม" พฤติกรรมพื้นฐานของ type อื่นมาใช้ แล้วเพิ่มความสามารถใหม่ (เช่น `ThaiDate` embed `time.Time` จาก **Part 025**) | **ควรใช้** — เป็นการใช้งานหลักที่ embedding ถูกออกแบบมาเพื่อสิ่งนี้ |
| ต้องการ implement interface บางส่วนโดยห่อ (wrap) ของเดิม แล้ว override เฉพาะบาง method (decorator pattern) | **ควรใช้** — ตามตัวอย่าง `CountingReader` ในหัวข้อ 8 |
| ต้องการรวม method set ของหลาย interface เป็น interface ใหม่ | **ควรใช้** — interface embedding เป็นวิธีมาตรฐานตามที่ standard library ทำ (`io.ReadWriter` เป็นต้น) |
| ต้องการ "inheritance" แบบ OOP ที่ method ของ type แม่เรียก method ที่ type ลูก override ไว้ (แบบ `Describe()` เรียก `Speak()` ที่ override) | **ห้ามใช้ embedding เพื่อจุดประสงค์นี้** เพราะทำไม่ได้ตามที่พิสูจน์ในหัวข้อ 4 — ถ้าต้องการ behavior แบบนี้จริงๆ ให้ออกแบบด้วย interface + dependency injection แทน (ส่ง behavior ที่ต้องการเป็น field type interface แล้วเรียกผ่าน field นั้น) |
| Struct มี field จำนวนมากที่ไม่เกี่ยวข้องกันเลย แค่อยากลดความยาวชื่อ field | **ไม่ควรใช้** — embedding ควรสื่อความสัมพันธ์ "has-a" ที่มีความหมายจริง ไม่ใช่ใช้เป็นแค่ทางลัดพิมพ์โค้ดสั้นลง เพราะจะทำให้โครงสร้างข้อมูลอ่านยากขึ้นในระยะยาว |

หลักคิดสรุป: **ใช้ embedding เมื่อความสัมพันธ์เป็น "has-a" ที่ต้องการ code reuse ผ่าน promotion จริงๆ (โดยเฉพาะเพื่อ satisfy หรือห่อ interface) แต่อย่าคาดหวังพฤติกรรมแบบ polymorphism/virtual dispatch ของ OOP inheritance จากมันเด็ดขาด** ถ้าต้องการ dynamic behavior ที่เปลี่ยนไปตาม concrete type ที่ใช้งานจริง เครื่องมือที่ถูกต้องคือ **interface** ไม่ใช่ struct embedding

---

## สรุปสิ่งที่ได้เรียนในบทนี้

- Struct embedding ทำโดยประกาศ anonymous field (ใช้ชื่อ type แทนชื่อ field) ทำให้ field/method ของ type ที่ embed ถูก **promote** ขึ้นมาเข้าถึงได้โดยตรงจาก struct แม่
- **Embedding ไม่ใช่ inheritance** — พิสูจน์ด้วยตัวอย่าง `Animal`/`Dog`: method ของ `Animal` ที่เรียก method อื่นของตัวเองภายใน จะไม่มีวัน "เห็น" การ override ที่ `Dog` ทำไว้ เพราะ Go ไม่มี virtual dispatch สำหรับ struct embedding
- Embedding คือเครื่องมือสำหรับ **composition ("has-a" ผ่าน promotion)** ไม่ใช่ "is-a" — สอดคล้องกับปรัชญา "favor composition over inheritance" ที่ Go ยึดถือมาตั้งแต่ **Part 001**
- เมื่อ embed หลาย type ที่มีชื่อ field/method ชนกันที่ depth เดียวกัน จะเกิด **ambiguous selector** บังคับให้ระบุ path เต็มเสมอ (`c.Base.ID` แทน `c.ID`)
- Interface embedding รวม method set ของหลาย interface เป็น interface ใหม่ (`io.ReadWriter` = `io.Reader` + `io.Writer`) เป็นแพทเทิร์นมาตรฐานของ standard library
- Embed **interface** (ไม่ใช่ struct) เข้าไปใน struct ทำให้ satisfy interface นั้นได้ทันทีแบบ "ยืม" implementation มา แล้ว override เฉพาะ method ที่ต้องการเปลี่ยนพฤติกรรม (decorator/wrapper pattern) ตามตัวอย่าง `CountingReader`
- ใช้ embedding เมื่อต้องการ code reuse แบบ has-a หรือ wrap interface บางส่วน แต่ใช้ interface เมื่อต้องการ dynamic behavior ที่เปลี่ยนไปตาม concrete type จริง

## แบบฝึกหัดท้ายบท

1. สร้าง struct `Person` ที่มี `Name string` และ `Age int` แล้วสร้าง struct `Employee` ที่ embed `Person` เพิ่ม field `Salary int` ทดสอบว่าเข้าถึง `emp.Name` ได้โดยตรงหรือไม่
2. ทำซ้ำตัวอย่าง `Animal`/`Dog` ในหัวข้อที่ 4 ด้วยชื่อ type และ method ของตัวเอง (เช่น `Shape`/`Circle` ที่มี method `Area()` และ `Info()` ที่เรียก `Area()` ภายใน) แล้วยืนยันด้วยตัวเองว่าผลลัพธ์เป็นไปตามที่บทเรียนอธิบายไว้
3. สร้าง struct สองตัวที่มี field ชื่อ `Status string` เหมือนกัน แล้ว embed ทั้งคู่ไว้ใน struct ที่สาม ลองเข้าถึง `x.Status` แบบสั้นดูว่า error อะไร แล้วแก้ด้วยการระบุ path เต็ม
4. เขียน interface `Shape` (มี `Area() float64`) และ `Named` (มี `Name() string`) แล้วสร้าง interface `NamedShape` ที่ embed ทั้งสอง จากนั้นเขียน struct ที่ implement `NamedShape` ให้ครบ
5. เขียน struct `LoggingWriter` ที่ embed `io.Writer` แล้ว override method `Write` ให้พิมพ์ log ทุกครั้งที่มีการเขียนข้อมูล (เช่น `fmt.Println("writing", len(p), "bytes")`) ก่อนเรียก `Write` ของตัวจริงข้างใน ทดสอบโดยส่งเข้า `fmt.Fprintln`
6. อธิบายด้วยคำพูดของตัวเอง (เขียนเป็น comment ในโค้ดหรือกระดาษ) ว่าทำไม Go ถึงเลือกไม่ให้ struct embedding มี virtual dispatch ทั้งที่จะทำให้เขียนโค้ดสไตล์ OOP ได้คุ้นเคยกว่า เชื่อมโยงกับปรัชญา "less is more" จาก **Part 001**

---

**ต่อไป**: [Part 031 — Reflection พื้นฐานด้วย `reflect`](./031-reflection-basics.md)
