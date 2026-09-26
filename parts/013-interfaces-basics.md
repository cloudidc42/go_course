# Part 013: Interfaces พื้นฐาน

> ภาคที่ 1: พื้นฐานภาษา Go (Fundamentals) — ตอนที่ 13 จาก 15

## สารบัญของบทนี้

1. Interface คืออะไร
2. Implicit Satisfaction: ไม่มี keyword `implements`
3. ทำไม Implicit Satisfaction ถึงทำให้โค้ด Decouple กัน
4. ตัวอย่างเต็ม: `Shape` Interface กับ `Rectangle`, `Circle`
5. Interface Value คือคู่ (Type, Value) ไม่ใช่แค่ตัวชี้เฉยๆ
6. กับดักคลาสสิก: nil Interface vs Interface ที่หุ้ม nil Pointer
7. Small Interfaces Idiom: `io.Reader`, `fmt.Stringer`
8. ทำไม Go เลือก Interface เล็กๆ จำนวนมาก แทน Interface ใหญ่ตัวเดียว
9. สรุปสิ่งที่ได้เรียนในบทนี้
10. แบบฝึกหัดท้ายบท

---

## 1. Interface คืออะไร

**Interface** ใน Go คือ **ชุดของ method signature** (ชื่อ method, parameter, return type) โดยไม่มี implementation ใดๆ อยู่ในตัวมันเอง พูดง่ายๆ interface คือ "สัญญา" ที่บอกว่า "ใครก็ตามที่มี method ตามรายการนี้ครบ ถือว่าเป็น type นี้ได้"

```go
type Shape interface {
	Area() float64
	Perimeter() float64
}
```

`Shape` ในตัวอย่างนี้ไม่ได้บอกว่า struct หน้าตาเป็นอย่างไร ไม่มี field ใดๆ เลย มันบอกแค่ว่า **"อะไรก็ตามที่มี method `Area() float64` และ `Perimeter() float64` ถือว่าเป็น `Shape`"**

นี่คือความต่างสำคัญจาก `struct` ที่เรียนใน Part 011: `struct` อธิบาย "ข้อมูล" (state) ว่ามี field อะไรบ้าง ส่วน `interface` อธิบาย "พฤติกรรม" (behavior) ว่าทำอะไรได้บ้าง โดยไม่สนใจว่าข้างในเก็บข้อมูลแบบไหน

Interface ตอบโจทย์ปรัชญา "less is more" ที่กล่าวถึงใน Part 001 ได้ตรงจุด: Go ไม่มี class, ไม่มี inheritance แต่ interface ทำให้เราเขียนโค้ดแบบ polymorphism (ให้ฟังก์ชันหนึ่งทำงานกับหลาย type ได้) โดยไม่ต้องมีกลไกซับซ้อนแบบภาษา OOP ดั้งเดิมเลย

---

## 2. Implicit Satisfaction: ไม่มี keyword `implements`

นี่คือจุดที่ทำให้ interface ของ Go **ต่างจากภาษา OOP ส่วนใหญ่อย่างสิ้นเชิง** ในภาษาอย่าง Java หรือ C# เวลาจะบอกว่า class หนึ่ง implement interface ไหน ต้องเขียนประกาศชัดเจน:

```java
// Java: ต้องประกาศ implements ชัดเจน
class Rectangle implements Shape {
    // ...
}
```

แต่ Go **ไม่มี keyword `implements` เลย** Type ใดๆ ก็ตามจะ "implement" interface โดยอัตโนมัติ ทันทีที่มัน**มี method ครบตามที่ interface กำหนด** — ไม่ต้องประกาศความสัมพันธ์นี้ที่ไหนทั้งสิ้น เรียกแนวคิดนี้ว่า **structural typing** (บางครั้งเรียก "duck typing แบบ static type")

> "If it walks like a duck and quacks like a duck, it's a duck." — ถ้ามันเดินเหมือนเป็ดและร้องเหมือนเป็ด ก็ถือว่าเป็นเป็ด ไม่สำคัญว่ามันประกาศตัวเองว่าเป็นเป็ดหรือเปล่า

```go
package main

import "fmt"

type Shape interface {
	Area() float64
}

type Rectangle struct {
	Width, Height float64
}

// Rectangle ไม่ได้เขียนอะไรบอกว่า "implements Shape" เลย
// แต่แค่มี method Area() float64 ตรงตามที่ Shape ต้องการ ก็ถือว่าเป็น Shape ได้ทันที
func (r Rectangle) Area() float64 {
	return r.Width * r.Height
}

func main() {
	var s Shape = Rectangle{Width: 3, Height: 4}
	fmt.Println(s.Area()) // 12
}
```

`Rectangle` implement `Shape` โดยอัตโนมัติ ทันทีที่มัน method `Area() float64` ตรงตาม signature — compiler เช็คให้เองตอน compile time (จึงยังปลอดภัยเท่า static typing เต็มรูปแบบ ไม่ใช่ dynamic typing แบบ Python/JavaScript)

---

## 3. ทำไม Implicit Satisfaction ถึงทำให้โค้ด Decouple กัน

ผลลัพธ์ที่ทรงพลังที่สุดของ implicit satisfaction คือ **เราสามารถประกาศ interface ไว้ฝั่งผู้ใช้งาน (consumer) โดยไม่ต้องขอความร่วมมือจากฝั่งที่เขียน struct เลย**

ลองนึกภาพสถานการณ์จริง: สมมติเรามี package `payment` ที่มี struct `CreditCard`, `PayPal`, `BankTransfer` ซึ่งแต่ละตัวก็มี method `Pay(amount float64) error` ของตัวเอง โดยที่คนเขียน struct เหล่านี้**ไม่เคยรู้จัก** และไม่เคยเห็น interface ของเราเลยด้วยซ้ำ ทีมของเรา (ผู้ใช้งาน) สามารถประกาศ interface ขึ้นมาเองในโค้ดของเราได้เลย:

```go
package main

import "fmt"

// เราประกาศ interface นี้เอง ในฝั่งของเรา โดยไม่ต้องแก้โค้ดของ struct ต้นทางแม้แต่บรรทัดเดียว
type PaymentMethod interface {
	Pay(amount float64) error
}

type CreditCard struct{ Number string }

func (c CreditCard) Pay(amount float64) error {
	fmt.Printf("จ่าย %.2f บาท ผ่านบัตร %s\n", amount, c.Number)
	return nil
}

type PayPal struct{ Email string }

func (p PayPal) Pay(amount float64) error {
	fmt.Printf("จ่าย %.2f บาท ผ่าน PayPal (%s)\n", amount, p.Email)
	return nil
}

// Checkout ทำงานกับ PaymentMethod ตัวไหนก็ได้ โดยไม่ต้องรู้จัก concrete type เลย
func Checkout(method PaymentMethod, amount float64) error {
	return method.Pay(amount)
}

func main() {
	Checkout(CreditCard{Number: "1234-5678"}, 500)
	Checkout(PayPal{Email: "user@example.com"}, 300)
}
```

จุดที่ควรสังเกตคือ `Checkout` ไม่รู้จัก `CreditCard` หรือ `PayPal` เลยด้วยซ้ำ มันรู้จักแค่ "สิ่งที่จ่ายเงินได้" (`PaymentMethod`) เท่านั้น ทำให้:

- เพิ่ม payment method ใหม่ในอนาคต (เช่น `Crypto`) ได้โดย**ไม่ต้องแก้ `Checkout` เลย**
- เขียน test ได้ง่ายมาก เพราะสร้าง mock struct ปลอมที่มี method `Pay` ขึ้นมาแทนของจริงได้ทันที โดยไม่ต้องมี framework พิเศษ
- โค้ดสองฝั่ง (ฝั่งสร้าง struct กับฝั่งใช้ interface) **แทบไม่ผูกติดกัน (decoupled)** เลย ต่างจากภาษาที่ต้องประกาศ `implements` ตรงๆ ซึ่งบังคับให้ทั้งสองฝั่งต้อง "รู้จัก" กันตั้งแต่ตอนเขียนโค้ด

นี่คือเหตุผลที่วงการ Go พูดกันบ่อยว่า **"interface ควรถูกประกาศฝั่งผู้ใช้งาน (consumer), ไม่ใช่ฝั่งผู้ให้บริการ (producer)"** เพราะ implicit satisfaction เปิดโอกาสให้ทำแบบนั้นได้อย่างเป็นธรรมชาติ

---

## 4. ตัวอย่างเต็ม: `Shape` Interface กับ `Rectangle`, `Circle`

มาดูตัวอย่างที่สมบูรณ์กว่าเดิม โดยมี interface หนึ่งตัวและ struct สอง type ที่ implement มันคนละแบบ:

```go
package main

import (
	"fmt"
	"math"
)

type Shape interface {
	Area() float64
	Perimeter() float64
}

type Rectangle struct {
	Width, Height float64
}

func (r Rectangle) Area() float64      { return r.Width * r.Height }
func (r Rectangle) Perimeter() float64 { return 2 * (r.Width + r.Height) }

type Circle struct {
	Radius float64
}

func (c Circle) Area() float64      { return math.Pi * c.Radius * c.Radius }
func (c Circle) Perimeter() float64 { return 2 * math.Pi * c.Radius }

// describe รับ Shape ตัวไหนก็ได้ ไม่ว่าจะเป็น Rectangle, Circle หรือ type อื่นที่จะเพิ่มในอนาคต
func describe(s Shape) {
	fmt.Printf("Area = %.2f, Perimeter = %.2f\n", s.Area(), s.Perimeter())
}

func main() {
	shapes := []Shape{
		Rectangle{Width: 4, Height: 5},
		Circle{Radius: 3},
	}

	for _, s := range shapes {
		describe(s)
	}
}
```

ผลลัพธ์:

```
Area = 20.00, Perimeter = 18.00
Area = 28.27, Perimeter = 18.85
```

สังเกตจุดสำคัญ:

- `[]Shape{...}` คือ slice ที่เก็บ **type ต่างกัน** (`Rectangle` กับ `Circle`) ไว้ด้วยกันได้ เพราะทั้งคู่ implement `Shape` interface เหมือนกัน — นี่คือ **polymorphism** แบบ Go
- ฟังก์ชัน `describe` เขียนครั้งเดียว ใช้ได้กับทุก `Shape` ในอนาคต ถ้าเพิ่ม `Triangle` เข้ามาแล้วมันมี `Area()` กับ `Perimeter()` ครบ ก็ใช้กับ `describe` ได้ทันทีโดยไม่ต้องแก้อะไรเลย
- `Rectangle` และ `Circle` ไม่รู้จักกันเลย ไม่ได้สืบทอดจากอะไรร่วมกัน (ไม่มี base class) เชื่อมกันแค่ผ่าน "รูปร่าง" ของ method ที่มีตรงกันเท่านั้น

หลักการเดียวกันนี้จะกลับมาใช้ซ้ำอีกหลายครั้งตลอดหลักสูตร โดยเฉพาะตอนเรียน `sort.Interface` ใน Part 027, `io.Reader`/`io.Writer` ใน Part 048 และการออกแบบ Clean Architecture ใน Part 100

---

## 5. Interface Value คือคู่ (Type, Value) ไม่ใช่แค่ตัวชี้เฉยๆ

ก่อนจะเข้าใจกับดักเรื่อง nil ในหัวข้อถัดไป ต้องเข้าใจก่อนว่า **ตัวแปร interface ไม่ได้เก็บแค่ "ค่า" อย่างที่ตาเห็น** แต่เก็บเป็น**คู่ข้อมูล 2 ส่วนภายใน**เสมอ:

1. **Type** — concrete type จริงๆ ที่ถูกเก็บอยู่ (เช่น `Rectangle`, `*Item`)
2. **Value** — ค่าจริงของ type นั้น (หรือ pointer ไปยังค่านั้น)

เขียนเป็นภาพคือ interface value = `(type, value)` เสมอ ลองดูโค้ดสาธิตที่พิมพ์ทั้งสองส่วนออกมาให้เห็นชัดๆ ด้วย `%T` (พิมพ์ type) และ `%v` (พิมพ์ value):

```go
package main

import "fmt"

type Animal interface {
	Sound() string
}

type Dog struct{}

func (Dog) Sound() string { return "Woof" }

type Cat struct{}

func (Cat) Sound() string { return "Meow" }

func inspect(a Animal) {
	fmt.Printf("type = %T, value = %v, Sound() = %s\n", a, a, a.Sound())
}

func main() {
	var a Animal // nil interface: (type=nil, value=nil)
	fmt.Printf("ก่อนกำหนดค่า: type = %T, value = %v, a == nil: %v\n", a, a, a == nil)

	a = Dog{}
	inspect(a)

	a = Cat{}
	inspect(a)
}
```

ผลลัพธ์:

```
ก่อนกำหนดค่า: type = <nil>, value = <nil>, a == nil: true
type = main.Dog, value = {}, Sound() = Woof
type = main.Cat, value = {}, Sound() = Meow
```

จะเห็นว่าตัวแปร `a` ตัวเดียวกัน เปลี่ยน "คู่ (type, value)" ข้างในไปเรื่อยๆ เมื่อ assign ค่าใหม่ที่เป็นคนละ concrete type กัน นี่คือกลไกเบื้องหลังที่ทำให้ **type assertion** และ **type switch** (Part 014) ทำงานได้ — มันคือการ "เปิดดู" ว่า concrete type ที่ซ่อนอยู่ข้างในตอนนี้คืออะไร

**Interface value จะเป็น `nil` ก็ต่อเมื่อทั้ง type และ value เป็น `nil` พร้อมกันเท่านั้น** — นี่คือกุญแจสำคัญของกับดักในหัวข้อถัดไป

---

## 6. กับดักคลาสสิก: nil Interface vs Interface ที่หุ้ม nil Pointer

นี่คือหนึ่งในเรื่องที่ทำให้มือใหม่ Go (และบางทีก็มือเก่า) งงและเจอบั๊กที่หาสาเหตุยากที่สุดเรื่องหนึ่งในภาษา สมควรอ่านหัวข้อนี้ช้าๆ และรันโค้ดตามจริง

### ปัญหาคืออะไร

จากหัวข้อที่แล้ว เรารู้ว่า interface value คือคู่ `(type, value)` ทีนี้ลองนึกภาพสถานการณ์นี้:

- เรามีตัวแปร pointer ชนิด `*MyError` ที่มีค่าเป็น `nil` (คือ `(type=*MyError, value=nil)`)
- เราเอาตัวแปรนี้ไปใส่ในตัวแปร interface (เช่น `error`)
- ตัวแปร interface นั้นจะได้คู่ `(type=*MyError, value=nil)` เก็บไว้ — **type ไม่ใช่ nil!** มันคือ `*MyError` มีแค่ value ข้างในเท่านั้นที่เป็น nil
- ผลคือ **interface นั้นจะไม่เท่ากับ `nil`** ทั้งๆ ที่ตาเรามองแล้วรู้สึกว่า "มันก็แค่ nil ไม่ใช่เหรอ"

มาดูโค้ดสาธิตแบบเต็มที่รันได้จริง:

```go
package main

import "fmt"

type MyError struct {
	Code int
}

func (e *MyError) Error() string {
	if e == nil {
		return "<nil MyError>"
	}
	return fmt.Sprintf("error code %d", e.Code)
}

// doSomething คืนค่า *MyError ที่เป็น nil เสมอในตัวอย่างนี้ (จำลองว่าไม่มี error เกิดขึ้น)
func doSomething() *MyError {
	return nil
}

// wrapper คืนค่าเป็น interface error โดยรับค่าจาก doSomething มาตรงๆ
func wrapper() error {
	var err *MyError = doSomething()
	return err // อันตราย! err ที่เป็น *MyError(nil) ถูกห่อเข้า interface แล้วไม่เท่ากับ nil อีกต่อไป
}

func main() {
	// กรณีที่ 1: nil interface ตรงๆ
	var err1 error
	fmt.Println("err1 == nil:", err1 == nil) // true

	// กรณีที่ 2: interface ที่ห่อ typed nil pointer เอาไว้
	err2 := wrapper()
	fmt.Println("err2 == nil:", err2 == nil) // false! นี่คือกับดักชื่อดังของ Go
	fmt.Printf("err2 type = %T, value = %v\n", err2, err2)

	if err2 != nil {
		fmt.Println("โปรแกรมคิดว่ามี error ทั้งที่จริงๆ ค่าเป็น nil ข้างใน")
	}
}
```

ผลลัพธ์:

```
err1 == nil: true
err2 == nil: false
err2 type = *main.MyError, value = <nil MyError>
โปรแกรมคิดว่ามี error ทั้งที่จริงๆ ค่าเป็น nil ข้างใน
```

### ทำไมถึงเกิดแบบนี้

ในฟังก์ชัน `wrapper()`:

```go
var err *MyError = doSomething() // err คือ (*MyError)(nil) -- pointer ชนิด *MyError ที่ชี้ไปที่ nil
return err                        // ตอน return ค่านี้ในฐานะ error (interface) มันถูกห่อเป็น (type=*MyError, value=nil)
```

ค่าที่ `return` ออกไปมี **type ระบุชัดเจนว่าเป็น `*MyError`** (ไม่ใช่ "ไม่มี type" แบบ nil interface ที่แท้จริง) มีแค่ value ข้างในเท่านั้นที่เป็น nil ด้วยกฎที่ว่า "interface จะ nil ก็ต่อเมื่อทั้ง type และ value เป็น nil พร้อมกัน" การเทียบ `err2 == nil` จึงได้ `false` เพราะ type ส่วน ไม่ใช่ nil

### สถานการณ์จริงที่พังเพราะกับดักนี้

รูปแบบที่พบบ่อยที่สุดในโค้ดจริงคือฟังก์ชันที่ประกาศตัวแปร error แบบ concrete pointer type ไว้ก่อน แล้ว return มันออกไปเป็น `error` interface ตรงๆ:

```go
func process() error {
	var err *MyError // ประกาศไว้เฉยๆ ยังไม่ได้ใส่ค่า -> เป็น nil

	// ... โค้ดที่ (ในกรณีนี้) ไม่เคยเซ็ตค่าให้ err เลย เพราะไม่มีปัญหาอะไรเกิดขึ้น

	return err // BUG: คืนค่า (*MyError)(nil) ห่อเป็น error interface -> err != nil เมื่อเช็คฝั่งผู้เรียก!
}

func main() {
	if err := process(); err != nil {
		fmt.Println("มี error:", err) // โค้ดส่วนนี้จะทำงานเสมอ ทั้งที่ไม่ควรมี error!
	}
}
```

### วิธีป้องกันกับดักนี้

**กฎทองคำ**: อย่า return ตัวแปร concrete pointer type ที่อาจเป็น nil ไปเป็น interface ตรงๆ ถ้าตั้งใจจะสื่อว่า "ไม่มี error" ให้ `return nil` แบบ literal ตรงๆ เสมอ

```go
func process() error {
	var myErr *MyError // สมมติว่ามีค่าเป็น nil อยู่แบบนี้

	if myErr != nil {
		return myErr // คืน error จริงๆ เฉพาะตอนที่มันไม่ใช่ nil เท่านั้น
	}
	return nil // คืน nil literal ตรงๆ -> ได้ nil interface (type=nil, value=nil) จริงๆ
}
```

หลักการนี้สรุปเป็นประโยคที่ควรจำขึ้นใจ: **ฟังก์ชันที่คืนค่าเป็น `error` interface ควร `return nil` แบบตรงๆ เมื่อไม่มี error เท่านั้น ห้าม return ตัวแปร concrete pointer type ที่อาจเป็น nil ออกไปเป็น interface โดยไม่เช็คก่อน**

เรื่องนี้สำคัญมากพอที่ทีม Go เองยังเคยเตือนไว้ใน FAQ อย่างเป็นทางการของภาษา และเป็นคำถามสัมภาษณ์งาน Go Developer ที่พบบ่อยที่สุดเรื่องหนึ่ง

### Interface กับการเปรียบเทียบด้วย `==`

จากที่รู้ว่า interface value คือคู่ `(type, value)` การเปรียบเทียบ interface สองตัวด้วย `==` จะเป็น `true` ก็ต่อเมื่อ **ทั้ง type และ value ตรงกันทั้งคู่**:

```go
package main

import "fmt"

type Point struct{ X, Y int }

func main() {
	var a, b any
	a = Point{X: 1, Y: 2}
	b = Point{X: 1, Y: 2}
	fmt.Println(a == b) // true: type เดียวกัน (Point) และ field ค่าตรงกันหมด

	b = Point{X: 9, Y: 9}
	fmt.Println(a == b) // false: type เดียวกัน แต่ value ต่างกัน

	var c any = "hello"
	fmt.Println(a == c) // false: type ต่างกันเลย (Point vs string)
}
```

ต้องระวังกรณีพิเศษ: **ถ้า concrete type ที่ซ่อนอยู่ข้างในเป็น type ที่เปรียบเทียบกันไม่ได้** (เช่น slice, map, function) การเปรียบเทียบ interface สองตัวด้วย `==` จะทำให้เกิด **panic ตอน runtime** ทันที แม้ตัวแปรทั้งสองจะประกาศเป็น `any`/interface ก็ตาม เพราะ compiler เช็คให้ตอน compile time ไม่ได้ว่า concrete type ข้างในจะเปรียบเทียบได้หรือเปล่า:

```go
var x any = []int{1, 2, 3}
var y any = []int{1, 2, 3}
fmt.Println(x == y) // panic: runtime error: comparing uncomparable type []int
```

ข้อควรระวังนี้เป็นอีกเหตุผลหนึ่งที่ควรใช้ `any`/empty interface อย่างระมัดระวัง (จะเรียนเรื่องนี้ต่อใน Part 014) และเป็นเหตุผลที่ map ที่ใช้ interface เป็น key ต้องมั่นใจว่า concrete type ที่ใส่เข้าไปเปรียบเทียบกันได้เสมอ

---

## 7. Small Interfaces Idiom: `io.Reader`, `fmt.Stringer`

แนวทางการออกแบบ interface ที่เป็นเอกลักษณ์ของ Go คือการสร้าง **interface ขนาดเล็กมาก** มักมีแค่ 1 method เท่านั้น standard library เต็มไปด้วยตัวอย่างแบบนี้:

```go
// io.Reader -- มีแค่ method เดียว
type Reader interface {
	Read(p []byte) (n int, err error)
}

// fmt.Stringer -- มีแค่ method เดียวเช่นกัน
type Stringer interface {
	String() string
}
```

`io.Reader` คือ interface ที่ทรงพลังที่สุดตัวหนึ่งในทั้งภาษา มันเป็นตัวแทนของ "อะไรก็ตามที่อ่านข้อมูลเป็น byte ได้" ไม่ว่าจะเป็นไฟล์ (`os.File`), การเชื่อมต่อ network (`net.Conn`), ข้อมูลใน memory (`strings.Reader`, `bytes.Buffer`) หรือแม้แต่ผลลัพธ์จากการบีบอัดข้อมูล (`gzip.Reader`) — ทั้งหมดนี้ implement `Reader` เดียวกัน จึงใช้กับฟังก์ชันตัวเดียวกันได้หมด

```go
package main

import (
	"fmt"
	"strings"
)

func main() {
	var r = strings.NewReader("hello Go")
	buf := make([]byte, 4)

	for {
		n, err := r.Read(buf) // Read คือ method เดียวใน io.Reader interface
		if n > 0 {
			fmt.Printf("อ่านได้ %d bytes: %q\n", n, buf[:n])
		}
		if err != nil {
			fmt.Println("อ่านจบแล้ว:", err)
			break
		}
	}
}
```

ผลลัพธ์:

```
อ่านได้ 4 bytes: "hell"
อ่านได้ 4 bytes: "o Go"
อ่านจบแล้ว: EOF
```

ส่วน `fmt.Stringer` คือ interface ที่ `fmt.Println`, `fmt.Printf` ใช้เช็คโดยอัตโนมัติ — ถ้า type ไหนมี method `String() string` เมื่อไหร่ที่เอาไปพิมพ์ด้วย `fmt` มันจะเรียก `String()` ให้เองแทนการพิมพ์ค่าดิบๆ (ตามที่เคยเห็นตัวอย่าง `Celsius` ใน Part 012)

เราจะเรียน `io.Reader`/`io.Writer` แบบเจาะลึกเต็มๆ ใน Part 048 บทนี้ขอให้เห็นแค่ภาพรวมว่ามันคือตัวอย่างของ small interfaces idiom ที่ชัดเจนที่สุดในภาษา

---

## 8. ทำไม Go เลือก Interface เล็กๆ จำนวนมาก แทน Interface ใหญ่ตัวเดียว

ในภาษา OOP หลายภาษา มักเจอ interface ขนาดใหญ่ที่มี method เป็นสิบๆ ตัว (เช่น `IRepository` ที่มีทั้ง `Create`, `Read`, `Update`, `Delete`, `List`, `Count`, ...) แต่แนวทางที่ Go และชุมชนแนะนำคือตรงกันข้าม — มีคำพูดที่ยึดถือกันอย่างกว้างขวางในวงการ Go:

> **"The bigger the interface, the weaker the abstraction."** — interface ยิ่งใหญ่ ยิ่งเป็นนามธรรมที่อ่อนแอ

เหตุผลเบื้องหลังแนวคิดนี้:

1. **Interface เล็กใช้ซ้ำได้ในหลายบริบทมากกว่า** — `io.Reader` ที่มี method เดียวใช้ได้กับทุกอย่างที่ "อ่านได้" แต่ถ้า interface มี 10 method ก็แทบไม่มี type ไหนที่จะบังเอิญมีครบทั้ง 10 method พอดี ทำให้ต้อง implement ปลอมๆ ทิ้งขว้างหลาย method ที่ไม่เกี่ยวข้อง
2. **Implement ง่ายกว่ามาก** — ผู้เขียน struct ใหม่ (หรือ mock สำหรับ test) implement interface เล็กได้ง่ายกว่าเยอะ เทียบกับต้องเขียน method ปลอมๆ อีก 9 ตัวที่ไม่ได้ใช้จริงเพื่อให้ผ่าน interface ใหญ่
3. **Compose interface ใหญ่จาก interface เล็กได้เสมอเมื่อจำเป็นจริงๆ** — ผ่าน interface embedding (จะเรียนใน Part 014) เช่น `io.ReadWriter` ก็คือเอา `io.Reader` กับ `io.Writer` มาประกอบกัน เราจึง**เริ่มจากเล็กแล้วค่อยประกอบให้ใหญ่ขึ้นทีหลังได้เสมอ** แต่ทำย้อนกลับ (แยก interface ใหญ่ให้เล็กลงทีหลัง) ยากกว่ามากเพราะโค้ดจำนวนมากอาจผูกกับ interface ใหญ่ไปแล้ว
4. **ตรงกับหลักการ Interface Segregation Principle** (ตัว "I" ใน SOLID) ที่บอกว่า "ไม่มี client คนไหนควรถูกบังคับให้พึ่งพา method ที่ตัวเองไม่ได้ใช้" — Go ทำให้หลักการนี้เป็นเรื่องธรรมชาติ ไม่ต้องพยายามฝืนออกแบบเอง

แนวทางปฏิบัติที่แนะนำคือ: **เริ่มออกแบบ interface ด้วย method แค่ 1 ตัวเสมอถ้าเป็นไปได้** แล้วค่อยขยายเมื่อมีความจำเป็นจริงๆ ที่พิสูจน์แล้วจากการใช้งานจริง (ไม่ใช่คาดเดาล่วงหน้า) — เรื่องการออกแบบ interface และข้อผิดพลาดจากการ over-abstract จะพูดถึงอย่างละเอียดใน Part 014

---

## สรุปสิ่งที่ได้เรียนในบทนี้

- **Interface** คือชุดของ method signature ล้วนๆ ไม่มี implementation ใดๆ อยู่ในตัวเอง เป็นตัวแทนของ "พฤติกรรม" ไม่ใช่ "ข้อมูล"
- Go ใช้ **implicit satisfaction (structural typing)** — ไม่มี keyword `implements` ใดๆ type ใดก็ตามที่มี method ครบตามที่ interface กำหนด จะ implement interface นั้นโดยอัตโนมัติ
- Implicit satisfaction ทำให้โค้ด **decouple** กันสูงมาก เพราะฝั่งผู้ใช้งาน (consumer) ประกาศ interface ของตัวเองได้โดยไม่ต้องขอความร่วมมือจากฝั่งที่เขียน struct เลย
- ตัวอย่าง `Shape` interface กับ `Rectangle`/`Circle` แสดงให้เห็น polymorphism แบบ Go: เก็บหลาย concrete type ไว้ใน slice เดียวกันได้ผ่าน interface ร่วม
- **Interface value คือคู่ `(type, value)` เสมอ** — interface จะ `nil` ก็ต่อเมื่อทั้งสองส่วนเป็น `nil` พร้อมกันเท่านั้น
- **กับดัก nil interface vs interface ที่หุ้ม typed nil pointer**: ถ้า return ตัวแปร concrete pointer type ที่เป็น nil ออกไปเป็น interface ตรงๆ ผลลัพธ์จะไม่เท่ากับ `nil` ทั้งๆ ที่ค่าข้างในเป็น nil — วิธีป้องกันคือ `return nil` แบบ literal เสมอเมื่อไม่มี error
- **Small interfaces idiom**: `io.Reader`, `fmt.Stringer` คือตัวอย่างของ interface ที่มีแค่ 1 method แต่ใช้งานได้กว้างขวางมากในทั้ง standard library
- Go เลือก **interface เล็กจำนวนมาก แทน interface ใหญ่ตัวเดียว** เพราะ implement ง่ายกว่า ใช้ซ้ำได้มากกว่า และ compose ให้ใหญ่ขึ้นทีหลังได้เสมอ

## แบบฝึกหัดท้ายบท

1. ออกแบบ interface `Employee` ที่มี method `CalculateSalary() float64` แล้วสร้าง struct `FullTimeEmployee` และ `Contractor` ที่มีวิธีคำนวณเงินเดือนต่างกัน implement interface นี้โดยไม่ต้องเขียน `implements` ใดๆ (เพราะ Go ไม่มี) ทดสอบด้วยฟังก์ชันที่รับ `[]Employee` แล้ววนพิมพ์เงินเดือนทุกคน
2. รันโค้ดตัวอย่างกับดัก nil interface ในหัวข้อ 6 ด้วยตัวเอง แล้วลองแก้ `wrapper()` ให้ถูกต้องตามหลักการ "กฎทองคำ" ทดสอบว่า `err2 == nil` กลับมาเป็น `true` แล้ว
3. เขียนฟังก์ชัน `Sum(numbers []int) int` แล้วเปรียบเทียบกับการออกแบบแบบใช้ interface ที่มี method `Value() int` — อภิปรายว่ากรณีไหนควรใช้ interface กรณีไหนใช้ type ธรรมดาพอ (คำใบ้: ย้อนกลับไปอ่านหัวข้อ 8)
4. สร้าง type ของตัวเองที่ implement `fmt.Stringer` (มี method `String() string`) แล้วลองพิมพ์ค่านั้นด้วย `fmt.Println` สังเกตว่า Go เรียก `String()` ให้อัตโนมัติ
5. เขียน interface เล็กๆ ของตัวเองชื่อ `Closer` ที่มี method `Close() error` เพียงตัวเดียว (คล้าย `io.Closer` ใน standard library) แล้วสร้าง struct 2 ตัวที่ implement มันคนละแบบ อธิบายว่าทำไม interface นี้ถึงนำไปใช้ซ้ำได้กับหลายสถานการณ์
6. ค้นคว้าเพิ่มเติม: เปิดดู source code ของ `io.Reader` ใน Go standard library (หรือใช้ `go doc io.Reader`) แล้วเขียนสรุปสั้นๆ ว่าทำไมมันถึงถูกออกแบบให้มีแค่ method เดียว

---

**ต่อไป**: [Part 014 — Interfaces ขั้นสูง: type assertion, type switch, empty interface](./014-interfaces-advanced.md)
