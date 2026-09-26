# Part 011: Structs เจาะลึก

> ภาคที่ 1: พื้นฐานภาษา Go (Fundamentals) — ตอนที่ 11 จาก 15

## สารบัญของบทนี้

1. Struct declaration และการใช้งานพื้นฐาน
2. Field tags: preview เรื่อง `json` tag (เจาะลึกเต็มรูปแบบใน Part 025)
3. Anonymous structs
4. เปรียบเทียบ struct ด้วย `==` — ข้อกำหนดเรื่อง comparable field
5. Struct literals: keyed vs positional และทำไม keyed ถึงเป็นที่แนะนำ
6. Nested structs (struct ซ้อน struct)
7. Array และ slice ของ struct
8. Struct copying semantics — struct เป็น value type แท้ๆ
9. สรุปสิ่งที่ได้เรียนในบทนี้
10. แบบฝึกหัดท้ายบท

---

## 1. Struct declaration และการใช้งานพื้นฐาน

**Struct** คือโครงสร้างข้อมูลที่รวม field หลายตัว (อาจมี type ต่างกัน) เข้าไว้ในหน่วยเดียวกัน คล้ายกับ `class` (ที่ไม่มี method) ในภาษาอื่น หรือ `record`/`struct` ใน C — struct คือวิธีหลักที่ Go ใช้จัดกลุ่มข้อมูลที่เกี่ยวข้องกันให้อยู่ในที่เดียว ตามที่กล่าวไปใน Part 001 ว่า Go ไม่มี class/inheritance แต่ใช้ `struct` ร่วมกับ `interface` และ composition แทน — struct คือจุดเริ่มต้นของแนวคิดนี้

ประกาศ struct type ด้วย `type ชื่อ struct { ... }`:

```go
package main

import "fmt"

type Person struct {
	Name string
	Age  int
	City string
}

func main() {
	var p Person // zero value: field ทุกตัวเป็น zero value ของ type ตัวเอง
	fmt.Println(p) // { 0 }

	p2 := Person{Name: "Alice", Age: 30, City: "Bangkok"}
	fmt.Println(p2) // {Alice 30 Bangkok}

	fmt.Println(p2.Name, p2.Age, p2.City) // Alice 30 Bangkok

	p2.Age = 31
	fmt.Println(p2.Age) // 31
}
```

จุดสำคัญที่ควรจำ:

- **zero value ของ struct** คือ struct ที่ทุก field เป็น zero value ของ type ตัวเอง (string ว่าง, int เป็น 0, bool เป็น false ฯลฯ) ตามหลักการ zero value ที่เรียนไปตั้งแต่ Part 003 — Go ไม่มี concept "null object" แบบภาษาอื่น struct ที่ยังไม่ได้กำหนดค่าจะพร้อมใช้งานได้ทันทีเสมอ ไม่ panic
- เข้าถึง field ด้วย dot notation (`p2.Name`) และแก้ไขค่าได้โดยตรงถ้าตัวแปรนั้นแก้ไขได้ (ไม่ใช่ constant)
- ชื่อ field ที่ขึ้นต้นตัวใหญ่จะถูก export ออกจาก package ได้ (ตามกฎเดียวกับ Part 001) ชื่อตัวเล็กเข้าถึงได้แค่ภายใน package เดียวกัน — กฎนี้ใช้กับทุก field ของ struct เช่นกัน

---

## 2. Field tags: preview เรื่อง `json` tag

Go อนุญาตให้แนบ **string metadata** ไว้กับแต่ละ field ของ struct เรียกว่า **struct tag** เขียนต่อท้าย type ของ field ด้วย backtick (`` ` ``) tag ที่พบบ่อยที่สุดคือ `json` tag ซึ่งบอกว่า field นั้นควรถูกแปลงเป็น/จาก key อะไรเมื่อทำงานกับ JSON ผ่านแพ็กเกจ `encoding/json`

```go
package main

import (
	"encoding/json"
	"fmt"
)

type User struct {
	Name  string `json:"name"`
	Email string `json:"email"`
	Age   int    `json:"age,omitempty"`
}

func main() {
	u := User{Name: "Bob", Email: "bob@example.com"}
	data, _ := json.Marshal(u)
	fmt.Println(string(data))
}
```

ผลลัพธ์:

```
{"name":"Bob","email":"bob@example.com"}
```

สังเกตว่า field `Age` ที่มีค่า `0` (zero value) หายไปจากผลลัพธ์ JSON เพราะ tag `omitempty` บอกให้ข้าม field นั้นถ้าค่าเป็น zero value — นี่เป็นเพียงตัวอย่างสั้นๆ ให้คุ้นเคยกับรูปแบบ struct tag เท่านั้น รายละเอียดเต็มรูปแบบของการทำงานกับ JSON ทั้งหมด (การ marshal/unmarshal, การจัดการ nested JSON, tag option ต่างๆ) จะอยู่ใน **Part 025**

Struct tag ไม่ได้ใช้แค่กับ JSON เท่านั้น ไลบรารีอื่นๆ จำนวนมาก (เช่น การ validate ข้อมูล, การ map กับฐานข้อมูลผ่าน ORM ที่จะเรียนในภาคที่ 6) ก็ใช้กลไก struct tag แบบเดียวกันนี้เพื่ออ่าน metadata ของแต่ละ field ผ่านกลไก reflection (จะเรียนเจาะลึกใน Part 031)

---

## 3. Anonymous structs

**Anonymous struct** คือ struct ที่ **ไม่ได้ตั้งชื่อ type ไว้ล่วงหน้าด้วย `type`** ใช้เมื่อต้องการโครงสร้างข้อมูลชั่วคราวที่ใช้ครั้งเดียวแล้วทิ้ง ไม่คุ้มค่าที่จะตั้งชื่อ type แยกต่างหาก

```go
package main

import "fmt"

func main() {
	// anonymous struct: ประกาศและกำหนดค่าพร้อมกันในบรรทัดเดียว
	point := struct {
		X, Y int
	}{X: 3, Y: 4}

	fmt.Println(point) // {3 4}

	// นิยมใช้กับข้อมูลชั่วคราว เช่น รายการ test case
	tests := []struct {
		Input    int
		Expected int
	}{
		{Input: 2, Expected: 4},
		{Input: 3, Expected: 9},
	}

	for _, tt := range tests {
		result := tt.Input * tt.Input
		fmt.Println(result == tt.Expected) // true, true
	}
}
```

Pattern การใช้ anonymous struct ที่พบบ่อยที่สุดในโค้ด Go จริงคือการนิยาม **table-driven test** (slice ของ anonymous struct ที่เก็บ input กับ expected output) ซึ่งจะเรียนเจาะลึกใน **Part 034** — การใช้ anonymous struct ในกรณีนี้ช่วยให้ไม่ต้องสร้าง type แยกที่ไม่ได้ใช้ที่อื่นเลยในโปรแกรม ทำให้โค้ดกระชับและอยู่ในที่เดียวกับจุดที่ใช้งานจริง

---

## 4. เปรียบเทียบ struct ด้วย `==` — ข้อกำหนดเรื่อง comparable field

Struct สองตัวเปรียบเทียบกันด้วย `==` ได้โดยตรง **ถ้าทุก field ภายในเป็น comparable type** (comparable type ตามที่เรียนไปใน Part 007 เรื่อง map key) การเปรียบเทียบจะดูค่าของทุก field ทีละตัว ถ้าเท่ากันหมดทุก field จึงจะถือว่า struct ทั้งสองเท่ากัน

```go
package main

import "fmt"

type Point struct {
	X, Y int
}

func main() {
	p1 := Point{1, 2}
	p2 := Point{1, 2}
	p3 := Point{3, 4}

	fmt.Println(p1 == p2) // true - ทุก field เท่ากัน
	fmt.Println(p1 == p3) // false
}
```

แต่ถ้า struct มี field ที่เป็น type ที่ไม่ comparable (เช่น `slice`, `map`, `function` ตามที่เรียนไปใน Part 007) การเปรียบเทียบด้วย `==` จะทำให้เกิด **compile error** ทันที:

```go
package main

type WithSlice struct {
	Items []int
}

func main() {
	a := WithSlice{Items: []int{1, 2}}
	b := WithSlice{Items: []int{1, 2}}
	_ = a == b // compile error
}
```

รันแล้วจะได้:

```
invalid operation: a == b (struct containing []int cannot be compared)
```

ถ้าต้องการเปรียบเทียบเนื้อหาของ struct ที่มี field แบบ slice/map จริงๆ ต้องเขียนฟังก์ชันเปรียบเทียบเอง (วนเทียบทีละ field/element) หรือใช้แพ็กเกจ `reflect.DeepEqual` (จะเรียนใน Part 031) แทนการใช้ `==` โดยตรง

ความสามารถในการเปรียบเทียบ struct ด้วย `==` ยังทำให้ **struct ที่ทุก field comparable ใช้เป็น map key ได้** เช่นเดียวกับที่เห็นตัวอย่าง `Point` เป็น key ของ map ไปแล้วใน Part 007

---

## 5. Struct literals: keyed vs positional และทำไม keyed ถึงเป็นที่แนะนำ

Go มีสอง syntax ในการสร้าง struct literal (กำหนดค่า field ตอนสร้าง struct):

```go
package main

import "fmt"

type Point struct {
	X, Y int
}

func main() {
	// keyed literal - ระบุชื่อ field ชัดเจน (แนะนำ)
	p1 := Point{X: 1, Y: 2}

	// positional literal - เรียงตามลำดับ field ที่ประกาศไว้ (ต้องครบทุก field)
	p2 := Point{1, 2}

	fmt.Println(p1, p2) // {1 2} {1 2}

	// keyed literal ข้าม field ที่ไม่ระบุได้ (จะได้ zero value)
	p3 := Point{X: 5}
	fmt.Println(p3) // {5 0}
}
```

- **Keyed literal** (`Point{X: 1, Y: 2}`): ระบุชื่อ field คู่กับค่าอย่างชัดเจน ไม่สนใจลำดับ และสามารถข้าม field ที่ไม่ต้องการระบุได้ (จะได้ zero value ของ field นั้นแทน)
- **Positional literal** (`Point{1, 2}`): กำหนดค่าตามลำดับ field ที่ประกาศไว้ใน `type` เท่านั้น ต้องระบุค่าให้ครบทุก field เสมอ ห้ามข้าม

### ทำไม keyed literal ถึงเป็นที่แนะนำ

1. **อ่านง่ายกว่ามาก** โดยเฉพาะเมื่อ struct มีหลาย field ที่ type เดียวกัน (เช่น `int` หลายตัว) — `Point{1, 2}` บอกไม่ได้เลยว่าตัวไหนคือ X ตัวไหนคือ Y ถ้าไม่เปิดดู definition ในขณะที่ `Point{X: 1, Y: 2}` ชัดเจนในตัวเอง
2. **ปลอดภัยต่อการเปลี่ยนแปลงโครงสร้างในอนาคต** — ถ้า struct มีการเพิ่ม field ใหม่ หรือสลับลำดับ field ที่ประกาศไว้ (ซึ่งเป็นเรื่องปกติมากเมื่อโค้ดถูกพัฒนาต่อ) positional literal ที่เขียนไว้ที่อื่นในโปรแกรมจะ**ยังคง compile ผ่านได้แต่ได้ผลลัพธ์ผิดพลาดแบบเงียบๆ** เพราะค่าจะถูกกำหนดผิดตำแหน่งไปตามลำดับใหม่ทันที ในขณะที่ keyed literal จะยังคงถูกต้องเสมอไม่ว่าจะสลับลำดับ field อย่างไร (ยกเว้นกรณีเปลี่ยนชื่อ field ซึ่งจะ compile error ทันทีแทนที่จะ silent bug)
3. **`go vet` เตือนโดยอัตโนมัติเมื่อใช้ positional literal ข้าม package** — Go มีเครื่องมือ `go vet` (ตามที่แนะนำใน Part 001) ที่จะแจ้งเตือนทันทีถ้าพบการสร้าง positional literal ของ struct ที่มาจาก package อื่น เพราะความเสี่ยงข้อ 2 นั้นรุนแรงเป็นพิเศษเมื่อ struct type ถูกกำหนดไว้ใน package หนึ่ง แต่ถูกใช้สร้าง literal ใน package อื่นที่ไม่เห็น definition ตรงหน้า

ตัวอย่างการเตือนของ `go vet` เมื่อสร้าง struct จาก package อื่นแบบ positional:

```go
// ไฟล์ mathutil/point.go (package mathutil)
package mathutil

type Point struct {
	X, Y int
}
```

```go
// ไฟล์ main.go (package main)
package main

import (
	"fmt"
	"vettest/mathutil"
)

func main() {
	p := mathutil.Point{1, 2} // positional literal ข้าม package
	fmt.Println(p)
}
```

รันคำสั่ง `go vet ./...` จะได้คำเตือน:

```
./main.go:9:7: vettest/mathutil.Point struct literal uses unkeyed fields
```

โปรแกรมยังคง compile และรันได้ปกติ (ไม่ error, ไม่ panic) แต่ `go vet` แจ้งเตือนล่วงหน้าว่านี่คือแนวทางที่เสี่ยงต่อการเกิด bug เมื่อ `mathutil.Point` ถูกแก้ไขโครงสร้างในอนาคต **แนวทางปฏิบัติที่ดี**: ใช้ keyed literal เป็นค่าเริ่มต้นเสมอ โดยเฉพาะกับ struct ที่มาจาก package ภายนอก ยกเว้นกรณี struct ง่ายมากๆ ที่ทุกคนรู้ความหมายของลำดับ field อยู่แล้วชัดเจน (เช่น `image.Point{x, y}` ที่ทั้งวงการเข้าใจตรงกันว่า x มาก่อน y เสมอ) ซึ่งก็ยังพบไม่บ่อยนักในทางปฏิบัติ

---

## 6. Nested structs (struct ซ้อน struct)

Field ของ struct เป็น type อะไรก็ได้ รวมถึงเป็น struct อีกตัวหนึ่งซ้อนกันเข้าไป เรียกว่า **nested struct** ใช้จัดกลุ่มข้อมูลที่มีความสัมพันธ์เชิงลำดับชั้น (hierarchical)

```go
package main

import "fmt"

type Address struct {
	Street string
	City   string
}

type Employee struct {
	Name    string
	Address Address
}

func main() {
	e := Employee{
		Name: "Carol",
		Address: Address{
			Street: "123 Main St",
			City:   "Chiang Mai",
		},
	}

	fmt.Println(e) // {Carol {123 Main St Chiang Mai}}
	fmt.Println(e.Address.City) // เข้าถึง field ซ้อนกันด้วย . ต่อกัน — Chiang Mai

	e.Address.City = "Phuket"
	fmt.Println(e.Address.City) // Phuket
}
```

การเข้าถึงและแก้ไข field ที่ซ้อนกันหลายชั้นทำได้ด้วยการต่อ `.` ตามลำดับชั้น (`e.Address.City`) เหมือนการเข้าถึง field ปกติทุกประการ

**หมายเหตุสำคัญ**: `Employee` ในตัวอย่างนี้มี field ชื่อ `Address` ที่เป็น type `Address` — นี่คือ **named field ที่มี struct เป็น type** (ต้องระบุชื่อ field เวลาสร้าง literal) ซึ่งแตกต่างจาก **struct embedding** ที่จะเรียนเจาะลึกใน **Part 030** ซึ่งเป็นการฝัง struct type เข้าไปโดยไม่ตั้งชื่อ field แล้วเข้าถึง field ของ struct ที่ฝังอยู่ได้โดยตรงราวกับเป็น field ของตัวเอง — nested struct แบบธรรมดาที่เห็นในบทนี้ยังคงต้องระบุชื่อ field (`e.Address.City`) เสมอ

---

## 7. Array และ slice ของ struct

Struct เป็น type ธรรมดาที่ใช้เป็น element ของ array หรือ slice ได้เหมือน type อื่นๆ ทุกประการ (array และ slice ตามที่เรียนไปใน Part 006)

```go
package main

import "fmt"

type Product struct {
	Name  string
	Price float64
}

func main() {
	// array ของ struct
	var inventory [3]Product
	inventory[0] = Product{Name: "Pen", Price: 10.5}
	fmt.Println(inventory) // [{Pen 10.5} { 0} { 0}]

	// slice ของ struct (พบบ่อยกว่า array มากในโค้ดจริง)
	products := []Product{
		{Name: "Book", Price: 120},
		{Name: "Pen", Price: 10.5},
		{Name: "Notebook", Price: 45},
	}

	total := 0.0
	for _, p := range products {
		total += p.Price
	}
	fmt.Println("total:", total) // total: 175.5

	// แก้ไข field ของ struct ใน slice ต้องเข้าถึงผ่าน index โดยตรง
	products[0].Price = 150
	fmt.Println(products[0]) // {Book 150}
}
```

สังเกตว่าเวลาสร้าง slice ของ struct ด้วย literal (`[]Product{{...}, {...}, {...}}`) ไม่ต้องเขียนชื่อ type `Product` ซ้ำในแต่ละ element เพราะ compiler รู้อยู่แล้วว่า element ของ slice นี้ต้องเป็น `Product` — เขียนแค่ `{Name: "Book", Price: 120}` ก็เพียงพอ

### กับดักสำคัญ: `range` คืนค่า copy ของ struct มาให้เสมอ

จุดที่ต้องระวังอย่างมากเมื่อวนลูป struct ด้วย `range`: ตัวแปรที่ได้จาก `range` เป็น **copy** ของ struct นั้น ไม่ใช่ตัวจริงใน slice การแก้ไขค่าผ่านตัวแปรนั้นจะ**ไม่กระทบ**ข้อมูลต้นฉบับใน slice เลย

```go
// ระวัง: range คืนค่า copy ของ struct มาให้ ไม่ใช่ตัวจริงใน slice
for _, p := range products {
	p.Price = 0 // แก้ไขแค่ copy ใน loop เท่านั้น ไม่กระทบ slice ต้นฉบับ
}
fmt.Println(products[1]) // ยังคงเป็นราคาเดิม ไม่ถูกเปลี่ยนเป็น 0
```

ถ้าต้องการแก้ไขข้อมูลจริงใน slice ต้องเข้าถึงผ่าน **index โดยตรง** (`products[i].Price = 0`) หรือใช้ `range` แบบที่ได้ index มาแล้ว index เข้า slice ตรงๆ:

```go
for i := range products {
	products[i].Price *= 1.1 // แก้ไขค่าจริงใน slice ได้ เพราะเข้าถึงผ่าน index
}
```

กับดักนี้เป็นข้อผิดพลาดที่พบบ่อยมากสำหรับมือใหม่ Go และเชื่อมโยงโดยตรงกับหัวข้อถัดไปเรื่อง struct copying semantics — เหตุผลที่ `range` คืน copy มาให้ก็เพราะ **struct เป็น value type** ตามที่จะอธิบายในหัวข้อสุดท้ายของบทนี้

---

## 8. Struct copying semantics — struct เป็น value type แท้ๆ

นี่คือความแตกต่างที่สำคัญที่สุดระหว่าง struct กับ slice/map ที่เรียนไปใน Part 006 และ Part 007: **struct เป็น value type แท้ๆ** การ assign struct ตัวหนึ่งไปยังตัวแปรใหม่ หรือส่งเข้าฟังก์ชัน จะเป็นการ **copy ค่าของทุก field ทั้งก้อนทันที** ไม่ใช่การ share ข้อมูลเดียวกันแบบ slice หรือ map

```go
package main

import "fmt"

type Point struct {
	X, Y int
}

func modify(p Point) {
	p.X = 999 // แก้ไข copy เท่านั้น ไม่กระทบต้นฉบับ
}

func main() {
	original := Point{X: 1, Y: 2}

	// struct เป็น value type แท้ๆ - assign แล้ว copy ทั้งก้อนทันที
	copy1 := original
	copy1.X = 100

	fmt.Println("original:", original) // {1 2} - ไม่เปลี่ยน
	fmt.Println("copy1:", copy1)       // {100 2}

	modify(original)
	fmt.Println("after modify:", original) // {1 2} - ไม่เปลี่ยน เพราะ pass by value
}
```

ผลลัพธ์:

```
original: {1 2}
copy1: {100 2}
after modify: {1 2}
```

ทั้ง `copy1 := original` และการเรียก `modify(original)` ล้วนสร้าง **copy อิสระ** ของ struct ต้นฉบับ การแก้ไขค่าใน copy จึงไม่กระทบตัวต้นฉบับเลยไม่ว่ากรณีใด — พฤติกรรมนี้แตกต่างโดยสิ้นเชิงจาก map ที่เรียนไปใน Part 007 ที่การ assign หรือ pass เข้าฟังก์ชันไม่ copy ข้อมูลภายในเลย (แค่ copy ตัวชี้ไปยังข้อมูลเดียวกัน)

### ตารางเปรียบเทียบสรุป

| Type | Assign / ส่งเข้าฟังก์ชัน | ผลของการแก้ไขผ่าน copy |
|---|---|---|
| `struct` | copy ทุก field ทั้งก้อน | ไม่กระทบต้นฉบับ |
| `array` | copy ทุก element ทั้งก้อน (เหมือน struct — ตามที่เรียนไปใน Part 006) | ไม่กระทบต้นฉบับ |
| `slice` | copy แค่ header (ptr, len, cap) ยังชี้ไปที่ underlying array เดียวกัน | **กระทบ**ต้นฉบับถ้าแก้ไข element ที่มีอยู่แล้ว |
| `map` | copy แค่ตัวชี้ไปยัง hash table เดียวกัน | **กระทบ**ต้นฉบับเสมอ |

ถ้าต้องการให้ฟังก์ชันแก้ไข struct ต้นฉบับได้โดยตรง (โดยไม่ผ่านการ copy) ต้องส่ง **pointer ไปยัง struct** แทน (`*Point`) ตามหลักการที่เรียนไปใน Part 010 — นี่คือเหตุผลสำคัญที่ทำให้หลายฟังก์ชัน/method ใน Go เลือกรับ struct เป็น pointer เมื่อจำเป็นต้องแก้ไขค่าภายใน ซึ่งจะเห็นรูปแบบนี้ชัดเจนขึ้นเมื่อเรียนเรื่อง pointer receiver ใน **Part 012**

**ข้อพิจารณาด้าน performance**: เพราะ struct ถูก copy ทั้งก้อนทุกครั้งที่ assign หรือส่งเข้าฟังก์ชัน struct ที่มีขนาดใหญ่มาก (field จำนวนมาก หรือมี array ขนาดใหญ่ฝังอยู่) ควรพิจารณาส่งผ่าน pointer แทนการส่งค่าตรงๆ เพื่อหลีกเลี่ยงค่าใช้จ่ายในการ copy ข้อมูลจำนวนมากซ้ำๆ โดยเฉพาะในฟังก์ชันที่ถูกเรียกบ่อยมาก — รายละเอียดเรื่องนี้จะกลับมาเจาะลึกอีกครั้งในภาคที่ 7 (Performance)

---

## สรุปสิ่งที่ได้เรียนในบทนี้

- Struct รวม field หลายตัวเข้าเป็นหน่วยเดียว ประกาศด้วย `type ชื่อ struct { field type }` — zero value คือ struct ที่ทุก field เป็น zero value ของตัวเอง
- Struct tag (เช่น `` `json:"name"` ``) แนบ metadata ไว้กับ field ใช้บอกวิธีแปลงข้อมูล เช่นกับ `encoding/json` — รายละเอียดเต็มอยู่ใน Part 025
- Anonymous struct คือ struct ที่ไม่ตั้งชื่อ type ไว้ล่วงหน้า เหมาะกับข้อมูลชั่วคราวที่ใช้ครั้งเดียว เช่น table-driven test
- Struct สองตัวเปรียบเทียบด้วย `==` ได้ถ้าทุก field เป็น comparable type เท่านั้น ถ้ามี field แบบ slice/map จะ compile error ทันที
- Keyed literal (`Point{X: 1, Y: 2}`) ควรใช้เป็นค่าเริ่มต้นแทน positional literal (`Point{1, 2}`) เพราะอ่านง่ายกว่าและปลอดภัยต่อการเปลี่ยนโครงสร้างในอนาคต — `go vet` จะเตือนอัตโนมัติเมื่อใช้ positional literal ข้าม package
- Nested struct คือ struct ที่มี field เป็น struct อีกตัวหนึ่ง เข้าถึง field ที่ซ้อนกันด้วยการต่อ `.` ตามลำดับชั้น
- Struct ใช้เป็น element ของ array/slice ได้ปกติ แต่ต้องระวังว่า `range` คืนค่า copy ของ struct มาให้เสมอ การแก้ไขต้องทำผ่าน index โดยตรงจึงจะกระทบข้อมูลต้นฉบับ
- Struct เป็น **value type แท้ๆ**: assign หรือส่งเข้าฟังก์ชันจะ copy ทุก field ทั้งก้อนทันที ต่างจาก slice/map ที่มีพฤติกรรมแบบ reference-like — ถ้าต้องการแก้ไขต้นฉบับต้องส่งผ่าน pointer (Part 010)

## แบบฝึกหัดท้ายบท

1. ประกาศ struct `Book` ที่มี field `Title`, `Author`, `Price`, `Stock` แล้วเขียนฟังก์ชัน `totalValue(books []Book) float64` ที่คำนวณมูลค่ารวมของสต็อกทั้งหมด (`Price * Stock` รวมทุกเล่ม)
2. เขียน struct สองตัว `Address` และ `Company` โดย `Company` มี field เป็น `Address` (nested struct) แล้วเขียนฟังก์ชันที่รับ `*Company` เพื่อแก้ไข field ที่ซ้อนอยู่ข้างในให้ได้
3. ทดลองสร้าง struct ที่มี field เป็น `map[string]int` แล้วพยายามเปรียบเทียบสอง instance ด้วย `==` สังเกต error message ที่ compiler แจ้ง แล้วอธิบายว่าทำไมถึง error
4. เขียนโปรแกรมที่มี slice ของ struct `Student{Name string, Score int}` แล้วทดลองเขียนฟังก์ชันที่พยายามเพิ่มคะแนนให้ทุกคนผ่าน `range` ตรงๆ (จะไม่ได้ผล) เทียบกับการแก้ไขผ่าน index (`for i := range`) แล้วพิสูจน์ด้วยการรันจริงว่าแบบไหนได้ผลลัพธ์ที่ถูกต้อง
5. สร้าง package แยกที่มี struct หนึ่งตัว แล้วในอีก package หนึ่งลองสร้าง struct literal แบบ positional เพื่อดูคำเตือนจาก `go vet ./...` ด้วยตัวเอง แล้วแก้เป็น keyed literal เพื่อให้คำเตือนหายไป
6. เขียนฟังก์ชันสองตัวที่รับ struct ขนาดใหญ่ (มีหลาย field) ตัวหนึ่งรับแบบ value ตัวหนึ่งรับแบบ pointer แล้วทั้งคู่พยายามแก้ไข field ภายใน พิสูจน์ด้วยการรันจริงว่าตัวไหนกระทบ struct ต้นฉบับ และอธิบายเหตุผลด้วยหลักการ value type ที่เรียนในบทนี้

---

**ต่อไป**: [Part 012 — Methods และ Receiver (value vs pointer)](./012-methods.md)
