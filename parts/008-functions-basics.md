# Part 008: Functions พื้นฐาน, multiple return values

> ภาคที่ 1: พื้นฐานภาษา Go (Fundamentals) — ตอนที่ 8 จาก 15

## สารบัญของบทนี้

1. Function declaration syntax
2. Parameters: การจัดกลุ่ม type และการส่ง argument
3. Multiple return values
4. Named return values และ naked return
5. Functions เป็น first-class citizens และ function types
6. ส่งฟังก์ชันเป็น argument (higher-order functions)
7. Function ที่ return ฟังก์ชันแบบไม่มี state (selector pattern)
8. Convention: `error` เป็นค่า return ตัวสุดท้าย
9. Doc comment: การเขียนเอกสารกำกับฟังก์ชันตามแนวทาง godoc
10. สรุปสิ่งที่ได้เรียนในบทนี้
11. แบบฝึกหัดท้ายบท

---

## 1. Function declaration syntax

ฟังก์ชันคือหน่วยพื้นฐานที่สุดในการจัดระเบียบโค้ด Go ตามที่เห็นมาตั้งแต่ `main()` ใน Part 001 โครงสร้างเต็มของการประกาศฟังก์ชันคือ:

```go
func ชื่อฟังก์ชัน(parameter1 type1, parameter2 type2) returnType {
    // function body
}
```

ตัวอย่างฟังก์ชันง่ายๆ:

```go
package main

import "fmt"

func add(a int, b int) int {
	return a + b
}

func main() {
	fmt.Println(add(2, 3)) // 5
}
```

ส่วนประกอบของ syntax:

- `func` คือ keyword บังคับสำหรับประกาศฟังก์ชันเสมอ
- ชื่อฟังก์ชันตามกฎ export ที่เรียนไปใน Part 001 (ขึ้นต้นตัวใหญ่ = export, ตัวเล็ก = private เฉพาะ package)
- รายการ parameter ในวงเล็บ แต่ละตัวมี `ชื่อ type` กำกับ
- ชนิดข้อมูลของค่าที่ return ต่อท้ายวงเล็บ parameter (ถ้าไม่ return อะไรเลย ก็ไม่ต้องเขียน type ใดๆ)
- body ของฟังก์ชันอยู่ใน `{ }`

ฟังก์ชันที่ไม่ return ค่าใดๆ:

```go
func greet(name string, times int) {
	for i := 0; i < times; i++ {
		fmt.Println("Hello,", name)
	}
}
```

---

## 2. Parameters: การจัดกลุ่ม type และการส่ง argument

### รวม parameter ที่มี type เดียวกันเข้าด้วยกัน

เมื่อ parameter หลายตัวติดกันมี type เดียวกัน สามารถเขียน type ครั้งเดียวท้ายกลุ่มได้ ไม่ต้องเขียนซ้ำ:

```go
package main

import "fmt"

// แบบเขียน type ซ้ำทุกตัว
func add(a int, b int) int {
	return a + b
}

// แบบย่อ (idiomatic กว่า) — a และ b เป็น int ทั้งคู่
func multiply(a, b int) int {
	return a * b
}

func main() {
	fmt.Println(add(2, 3))      // 5
	fmt.Println(multiply(4, 5)) // 20
}
```

ทั้งสองแบบเทียบเท่ากันทุกประการ Go community นิยมใช้แบบย่อเมื่อ parameter ติดกันมี type เดียวกัน เพราะสั้นและอ่านง่ายกว่า

### Go ไม่มี named arguments ตอนเรียกฟังก์ชัน

ต่างจากภาษาอย่าง Python หรือ Kotlin ที่เรียกฟังก์ชันแบบระบุชื่อ parameter ได้ (เช่น `f(name="Go", times=3)`) **Go ไม่รองรับ named arguments เลย** การเรียกฟังก์ชันต้องส่ง argument ตามลำดับตำแหน่ง (positional) เท่านั้น:

```go
greet("Go", 2) // ต้องเรียงตามลำดับ: name ก่อน, times ทีหลัง เสมอ
```

ถ้าฟังก์ชันมี parameter หลายตัวที่ type เดียวกันจนอาจสลับลำดับกันได้ง่าย (เช่น `createUser(firstName, lastName string)`) แนวทางที่ community แนะนำคือใช้ **struct เป็น parameter** แทน เพื่อให้ผู้เรียกต้องระบุชื่อ field อย่างชัดเจน (จะเห็นรูปแบบนี้บ่อยขึ้นหลังเรียน struct ใน Part 011):

```go
type CreateUserParams struct {
	FirstName string
	LastName  string
}

func createUser(p CreateUserParams) {
	fmt.Println(p.FirstName, p.LastName)
}

// เรียกใช้แบบระบุชื่อ field ชัดเจน ไม่มีทางสลับผิด
createUser(CreateUserParams{FirstName: "Ada", LastName: "Lovelace"})
```

### การส่ง argument เป็นแบบ pass by value เสมอ

ทุก argument ที่ส่งเข้าฟังก์ชันใน Go ถูก **copy ค่า** เข้าไปเสมอ (pass by value) ไม่ว่าจะเป็น primitive type, struct, array — ฟังก์ชันจะได้รับ "สำเนา" ของค่าต้นฉบับ การแก้ไขค่า parameter ภายในฟังก์ชันจะไม่กระทบตัวแปรต้นฉบับที่ผู้เรียกส่งเข้ามา (ยกเว้น type ที่มีพฤติกรรมแบบ reference-like เช่น slice และ map ตามที่เรียนไปใน Part 006 และ Part 007 ซึ่งตัว header/pointer ถูก copy แต่ยังชี้ไปยังข้อมูลชุดเดียวกัน) เราจะเจาะลึกเรื่องนี้อีกครั้งพร้อมกับ pointer ใน Part 010

---

## 3. Multiple return values

หนึ่งในจุดเด่นที่ทำให้ Go แตกต่างจากภาษาอื่นจำนวนมาก (เช่น Java, C ที่ต้อง return ค่าเดียว หรือใช้ output parameter/wrapper object) คือ **ฟังก์ชันใน Go return ค่าได้หลายค่าพร้อมกัน** โดยไม่ต้องสร้าง struct หรือ wrapper ใดๆ

```go
package main

import (
	"errors"
	"fmt"
)

func divide(a, b float64) (float64, error) {
	if b == 0 {
		return 0, errors.New("cannot divide by zero")
	}
	return a / b, nil
}

func main() {
	result, err := divide(10, 2)
	fmt.Println(result, err) // 5 <nil>

	result2, err2 := divide(10, 0)
	fmt.Println(result2, err2) // 0 cannot divide by zero
}
```

รูปแบบ `(float64, error)` คือการประกาศ return type หลายตัวโดยครอบด้วยวงเล็บ คั่นด้วย comma และ `return` ก็ระบุค่าคั่นด้วย comma ตามลำดับเดียวกัน

รูปแบบนี้คือรากฐานของ **error handling ทั้งหมดใน Go** ที่จะเรียนเจาะลึกใน Part 015 — แทนที่จะใช้ exception/try-catch แบบภาษาอื่น Go ให้ฟังก์ชัน return ค่า error ออกมาเป็นค่าปกติคู่กับผลลัพธ์ ทำให้ผู้เรียกต้องจัดการ error อย่างชัดเจนทุกครั้งที่เรียกใช้

### รับค่าได้มากกว่า 2 ค่า

ฟังก์ชันจะ return กี่ค่าก็ได้ตามต้องการ:

```go
func minMax(nums []int) (int, int) {
	min, max := nums[0], nums[0]
	for _, n := range nums[1:] {
		if n < min {
			min = n
		}
		if n > max {
			max = n
		}
	}
	return min, max
}

lo, hi := minMax([]int{5, 3, 9, 1, 7})
fmt.Println(lo, hi) // 1 9
```

### การละทิ้งค่า return ที่ไม่ต้องการด้วย `_`

ถ้าฟังก์ชัน return หลายค่า แต่เราสนใจแค่บางค่า ใช้ **blank identifier** `_` (ตามที่เรียนไปใน Part 004 กับ `for range`) เพื่อทิ้งค่าที่ไม่ใช้:

```go
_, err := divide(8, 4)
fmt.Println(err) // <nil>
```

Go **บังคับ**ให้รับค่า return ให้ครบทุกตำแหน่งเสมอ (หรือใช้ `_` แทน) จะรับแค่บางตัวโดยไม่ระบุตัวอื่นเลยไม่ได้ — ต่างจากภาษาที่ยอมให้ทิ้งผลลัพธ์ทั้งหมดไปเฉยๆ ได้

---

## 4. Named return values และ naked return

Go มี feature พิเศษที่เรียกว่า **named return values** — สามารถตั้งชื่อให้กับค่าที่จะ return ไว้ในวงเล็บ signature ได้เลย เมื่อตั้งชื่อไว้แล้ว ตัวแปรเหล่านั้นจะถูกประกาศ (พร้อม zero value) โดยอัตโนมัติตั้งแต่ต้นฟังก์ชัน และใช้งานได้เหมือนตัวแปรปกติภายใน body

```go
package main

import "fmt"

func divmod(a, b int) (quotient, remainder int) {
	quotient = a / b
	remainder = a % b
	return // naked return - คืนค่า quotient, remainder ตามชื่อที่ประกาศไว้โดยอัตโนมัติ
}

func main() {
	q, r := divmod(17, 5)
	fmt.Println(q, r) // 3 2
}
```

`return` ที่ไม่ระบุค่าใดๆ เลยแบบนี้เรียกว่า **naked return** — Go จะ return ค่าปัจจุบันของตัวแปรที่ตั้งชื่อไว้ในวงเล็บ (`quotient` และ `remainder`) โดยอัตโนมัติ

Named return values ยังใช้ `return` แบบระบุค่าตามปกติได้เช่นกัน (ไม่บังคับว่าต้องใช้ naked return เสมอไป):

```go
func splitName(full string) (first, last string) {
	for i := 0; i < len(full); i++ {
		if full[i] == ' ' {
			return full[:i], full[i+1:] // return แบบระบุค่าปกติ แม้จะมีชื่อไว้แล้วก็ทำได้
		}
	}
	return full, "" // ไม่เจอช่องว่าง — ถือว่าเป็นชื่อเดียว
}

f, l := splitName("Ada Lovelace")
fmt.Println(f, l) // Ada Lovelace
```

### ทำไม named return values ถึงมีประโยชน์

1. **ใช้เป็นเอกสารในตัว (self-documenting)**: signature `func divmod(a, b int) (quotient, remainder int)` บอกความหมายของค่าที่ return ได้ชัดเจนกว่า `func divmod(a, b int) (int, int)` มาก โดยเฉพาะเมื่อ return type ซ้ำกันหลายตัวจนแยกความหมายไม่ออกจาก type อย่างเดียว
2. **แก้ไขค่า return ได้ใน `defer`**: เพราะตัวแปร named return เป็นตัวแปรจริงที่มีอยู่ตลอดการทำงานของฟังก์ชัน โค้ดใน `defer` (จะเรียนเจาะลึกใน Part 009) จึงแก้ไขค่าที่กำลังจะ return ได้ ก่อนที่ฟังก์ชันจะจบจริงๆ — เทคนิคนี้ใช้บ่อยมากในการจัดการ error แบบรวมศูนย์

```go
package main

import "fmt"

func increment() (result int) {
	defer func() {
		result++ // แก้ไขค่า return ผ่าน defer ได้ เพราะ result เป็นตัวแปรจริง
	}()
	result = 10
	return
}

func main() {
	fmt.Println(increment()) // 11
}
```

### ข้อควรระวัง: naked return ทำให้โค้ดอ่านยากในฟังก์ชันยาว

แม้ naked return จะเขียนสั้นและดูสะดวก แต่ **ทีม Go เองก็แนะนำให้ใช้อย่างระมัดระวัง** โดยเฉพาะในฟังก์ชันที่ body ยาวหรือมี `return` หลายจุด เพราะผู้อ่านโค้ดต้องเลื่อนกลับไปดูวงเล็บ signature เพื่อจำว่าค่าที่กำลังจะถูก return คืออะไร ต่างจาก `return value1, value2` ที่เห็นค่าตรงหน้าทันที

**แนวทางที่ idiomatic**: ใช้ named return values เพื่อประโยชน์ด้านเอกสารและกรณี `defer` ที่ต้องแก้ไขค่า return แต่เมื่อจะ `return` จริง ให้พยายามระบุค่าอย่างชัดเจน (`return quotient, remainder`) แทน naked return เปล่าๆ ยกเว้นในฟังก์ชันสั้นมากๆ ที่มี `return` จุดเดียวตอนท้าย ซึ่งความกำกวมแทบไม่เกิดขึ้น

---

## 5. Functions เป็น first-class citizens และ function types

ใน Go ฟังก์ชันคือ **ค่า (value)** เหมือน `int` หรือ `string` — สามารถเก็บลงตัวแปร ส่งเป็น argument ให้ฟังก์ชันอื่น หรือ return ออกจากฟังก์ชันได้ คุณสมบัตินี้เรียกว่า **first-class functions** และเป็นรากฐานสำคัญของการเขียนโปรแกรมเชิงฟังก์ชัน (functional-style) ใน Go

```go
package main

import "fmt"

func square(n int) int {
	return n * n
}

func cube(n int) int {
	return n * n * n
}

func main() {
	// function เป็น value เก็บลงตัวแปรได้
	var op func(int) int
	op = square
	fmt.Println(op(5)) // 25

	op = cube
	fmt.Println(op(3)) // 27

	// anonymous function เก็บลงตัวแปรตรงๆ
	double := func(n int) int {
		return n * 2
	}
	fmt.Println(double(21)) // 42
}
```

`var op func(int) int` คือการประกาศตัวแปรที่มี **function type**: "รับ `int` หนึ่งตัว คืนค่า `int`" — type ของฟังก์ชันประกอบด้วย signature ของ parameter และ return value เท่านั้น ไม่สนใจชื่อฟังก์ชัน ทำให้ `square` และ `cube` (ที่มี signature เหมือนกัน) assign ให้ตัวแปร `op` ตัวเดียวกันได้

### ตั้งชื่อ function type ด้วย `type`

เหมือนกับที่เคยตั้งชื่อ type อื่นๆ ใน Part 003 เราสามารถตั้งชื่อให้ function type เพื่อให้โค้ดอ่านง่ายขึ้นได้เช่นกัน:

```go
type transformer func(int) int

var t transformer = square
fmt.Println(t(6)) // 36
```

การตั้งชื่อ function type แบบนี้มีประโยชน์มากเมื่อ signature ซับซ้อนและถูกใช้ซ้ำหลายที่ในโปรแกรม (เช่น callback ของ HTTP handler ที่จะเจอใน Part 047)

---

## 6. ส่งฟังก์ชันเป็น argument (higher-order functions)

เพราะฟังก์ชันเป็น value ธรรมดา จึงส่งฟังก์ชันหนึ่งเข้าไปเป็น **parameter** ของอีกฟังก์ชันหนึ่งได้ ฟังก์ชันที่รับหรือ return ฟังก์ชันอื่นเรียกว่า **higher-order function** — pattern นี้ใช้บ่อยมากในโค้ด Go จริง โดยเฉพาะเวลาต้องการแยก "logic การวนลูป/จัดการโครงสร้างข้อมูล" ออกจาก "logic การประมวลผลแต่ละ element"

```go
package main

import "fmt"

// higher-order function: รับ function เป็น parameter
func applyToAll(nums []int, f func(int) int) []int {
	result := make([]int, len(nums))
	for i, n := range nums {
		result[i] = f(n)
	}
	return result
}

func filter(nums []int, keep func(int) bool) []int {
	var result []int
	for _, n := range nums {
		if keep(n) {
			result = append(result, n)
		}
	}
	return result
}

func main() {
	nums := []int{1, 2, 3, 4, 5}

	doubled := applyToAll(nums, func(n int) int {
		return n * 2
	})
	fmt.Println(doubled) // [2 4 6 8 10]

	evens := filter(nums, func(n int) bool {
		return n%2 == 0
	})
	fmt.Println(evens) // [2 4]
}
```

สังเกตว่าเราส่ง **anonymous function** (ฟังก์ชันที่ไม่มีชื่อ) เข้าไปตรงๆ ตอนเรียก `applyToAll` และ `filter` โดยไม่ต้องประกาศฟังก์ชันแยกไว้ก่อนล่วงหน้า ซึ่งสะดวกมากเมื่อ logic นั้นใช้ครั้งเดียวและสั้น รูปแบบนี้จะกลับมาอีกครั้งแบบเจาะลึกเรื่อง **closures** ใน Part 009 ที่ anonymous function จะ "จำ" ตัวแปรจาก scope ภายนอกได้ด้วย

ไลบรารีมาตรฐานของ Go เองก็ใช้ pattern higher-order function อยู่ทั่วไป เช่น `sort.Slice(data, less func(i, j int) bool)` ที่จะเรียนใน Part 027 หรือ `http.HandleFunc(pattern string, handler func(...))` ที่จะเรียนใน Part 047

---

## 7. Function ที่ return ฟังก์ชันแบบไม่มี state (selector pattern)

หัวข้อก่อนหน้าแสดงให้เห็นว่าฟังก์ชัน**รับ**ฟังก์ชันอื่นเป็น parameter ได้ ในทางกลับกันฟังก์ชันก็**return ฟังก์ชันอื่นออกมา**ได้เช่นกัน เพราะ function เป็น value ธรรมดา (ตามที่เรียนไปในหัวข้อที่ 5) กรณีที่ง่ายที่สุดคือการ return ฟังก์ชันที่มีอยู่แล้วตัวใดตัวหนึ่ง โดยไม่มีการ "จดจำ" ตัวแปรใดๆ เพิ่มเติม — ต่างจาก **closure** (ที่จะเรียนเจาะลึกใน Part 009) ซึ่ง return ฟังก์ชันที่จดจำตัวแปรจาก scope ภายนอกได้ด้วย

```go
package main

import "fmt"

func add(a, b int) int      { return a + b }
func subtract(a, b int) int { return a - b }
func multiply(a, b int) int { return a * b }

// operation คืนฟังก์ชันที่ตรงกับชื่อ op ที่ระบุ (ไม่มีการ capture ตัวแปรใดๆ - ไม่ใช่ closure)
func operation(op string) func(int, int) int {
	switch op {
	case "+":
		return add
	case "-":
		return subtract
	case "*":
		return multiply
	default:
		return nil
	}
}

func main() {
	calc := operation("+")
	fmt.Println(calc(3, 4)) // 7

	calc = operation("*")
	fmt.Println(calc(3, 4)) // 12

	calc = operation("?")
	fmt.Println(calc == nil) // true - ฟังก์ชันเปรียบเทียบกับ nil ได้ (แต่เทียบกันเองไม่ได้)
}
```

Pattern นี้เรียกว่า **selector pattern** หรือ **function dispatch table** เหมาะกับกรณีที่ต้องเลือกพฤติกรรมจากชื่อหรือค่าที่กำหนดตอน runtime (เช่น อ่านชื่อ operation จาก config หรือ user input) แทนที่จะเขียน `if/else` หรือ `switch` ยาวๆ ทุกครั้งที่ต้องเรียกใช้ ให้เลือกฟังก์ชันที่ถูกต้องมาเก็บไว้ในตัวแปรครั้งเดียว แล้วเรียกใช้ซ้ำได้เรื่อยๆ

ข้อสังเกตที่สำคัญ: **function value เปรียบเทียบกับ `nil` ได้เท่านั้น** (`calc == nil`) แต่ **เปรียบเทียบฟังก์ชันสองตัวเข้าด้วยกันโดยตรงไม่ได้** (`add == subtract` จะ compile error ทันที เพราะ function ไม่ใช่ comparable type ตามที่เรียนไปใน Part 007) การเช็คว่าได้รับฟังก์ชันที่ถูกต้องหรือไม่ในทางปฏิบัติจึงทำได้แค่เช็คว่าเป็น `nil` หรือไม่เท่านั้น

---

## 8. Convention: `error` เป็นค่า return ตัวสุดท้าย

ใน Go มีธรรมเนียมปฏิบัติที่ยึดถือกันอย่างเคร่งครัดทั่วทั้งภาษาและ standard library: **ถ้าฟังก์ชันอาจเกิดข้อผิดพลาดได้ ให้ return ค่า type `error` เป็นค่าสุดท้ายเสมอ**

```go
package main

import (
	"fmt"
	"strconv"
)

func parseAge(s string) (int, error) {
	age, err := strconv.Atoi(s)
	if err != nil {
		return 0, fmt.Errorf("invalid age %q: %w", s, err)
	}
	return age, nil
}

func main() {
	age, err := parseAge("25")
	if err != nil {
		fmt.Println("error:", err)
	} else {
		fmt.Println("age:", age) // age: 25
	}

	_, err = parseAge("abc")
	if err != nil {
		fmt.Println("error:", err)
		// error: invalid age "abc": strconv.Atoi: parsing "abc": invalid syntax
	}
}
```

รูปแบบการเรียกใช้ที่เจอแทบทุกที่ในโค้ด Go คือ:

```go
result, err := someFunction()
if err != nil {
    // จัดการ error ทันที
    return err // หรือ handle ตามที่เหมาะสม
}
// ใช้งาน result ต่อได้อย่างมั่นใจว่าไม่มี error
```

เมื่อไม่มี error เกิดขึ้น ค่าที่ return ในตำแหน่ง `error` จะเป็น `nil` (ซึ่งเป็น zero value ของ interface type — จะอธิบายเรื่อง interface เจาะลึกใน Part 013) การเช็ค `if err != nil` จึงเป็นรูปแบบมาตรฐานที่ต้องเจอซ้ำๆ นับพันครั้งตลอดการเขียน Go

นี่เป็นเพียงการแนะนำแบบสั้นๆ ให้คุ้นเคยกับรูปแบบและ convention เท่านั้น — เนื้อหาเรื่อง `error` แบบเต็มรูปแบบ ทั้งการสร้าง custom error, การ wrap/unwrap, และแนวทางออกแบบ error handling ที่ดี จะอยู่ใน **Part 015** และเจาะลึกยิ่งขึ้นใน **Part 016**

---

## 9. Doc comment: การเขียนเอกสารกำกับฟังก์ชันตามแนวทาง godoc

Go มีธรรมเนียมมาตรฐานในการเขียนคอมเมนต์อธิบายฟังก์ชัน เรียกว่า **doc comment** ซึ่งเครื่องมือ `go doc` (ที่แนะนำไปใน Part 001) และเว็บไซต์ https://pkg.go.dev จะดึงคอมเมนต์นี้มาแสดงเป็นเอกสารประกอบ package โดยอัตโนมัติ กฎเขียนง่ายมากแค่ 2 ข้อ:

1. วางคอมเมนต์ไว้**ติดกับ**บรรทัด `func` โดยไม่มีบรรทัดว่างคั่น
2. ประโยคแรกของคอมเมนต์ควรขึ้นต้นด้วย**ชื่อฟังก์ชัน** ตามด้วยคำอธิบายว่าฟังก์ชันนั้นทำอะไร

```go
package main

import "fmt"

// Area คำนวณพื้นที่สี่เหลี่ยมผืนผ้าจากความกว้างและความสูงที่กำหนด
// ถ้า width หรือ height เป็นค่าลบ จะคืนค่า 0
func Area(width, height float64) float64 {
	if width < 0 || height < 0 {
		return 0
	}
	return width * height
}

func main() {
	fmt.Println(Area(3, 4))  // 12
	fmt.Println(Area(-1, 4)) // 0
}
```

เมื่อเขียนคอมเมนต์ตามรูปแบบนี้ คำสั่ง `go doc Area` (รันในโฟลเดอร์ที่มีไฟล์นี้) จะแสดงผลลัพธ์:

```
func Area(width, height float64) float64
    Area คำนวณพื้นที่สี่เหลี่ยมผืนผ้าจากความกว้างและความสูงที่กำหนด ถ้า width หรือ
    height เป็นค่าลบ จะคืนค่า 0
```

ธรรมเนียมนี้สำคัญมากสำหรับฟังก์ชันที่ **export** (ขึ้นต้นตัวใหญ่ ตามกฎที่เรียนไปใน Part 001) เพราะฟังก์ชันเหล่านี้คือ "หน้าตา" ของ package ที่คนอื่นจะนำไปใช้งาน — package มาตรฐานของ Go เองทุกตัว (`fmt`, `strings`, `strconv` ที่จะเรียนใน Part 019-020) ล้วนเขียน doc comment ตามรูปแบบนี้อย่างเคร่งครัด การฝึกเขียน doc comment ให้เป็นนิสัยตั้งแต่ต้นจะทำให้โค้ดที่เขียนออกมาดูเป็นมืออาชีพและใช้งานร่วมกับทีมอื่นได้ง่ายขึ้นมาก

---

## สรุปสิ่งที่ได้เรียนในบทนี้

- ฟังก์ชันประกาศด้วย `func ชื่อ(parameters) returnType { ... }` และ parameter ที่ type เดียวกันติดกันเขียน type ครั้งเดียวได้ (`func add(a, b int) int`)
- Go ไม่มี named arguments ตอนเรียกฟังก์ชัน — argument ทุกตัวส่งตามลำดับตำแหน่ง (positional) เท่านั้น และทุก argument ถูก pass by value เสมอ
- ฟังก์ชันใน Go return ค่าได้หลายค่าพร้อมกัน โดยประกาศ return type หลายตัวในวงเล็บ คั่นด้วย comma — นี่คือรากฐานของ error handling แบบ Go
- ใช้ `_` เพื่อทิ้งค่า return ที่ไม่ต้องการ แต่ต้องรับให้ครบทุกตำแหน่งเสมอ
- Named return values ตั้งชื่อ return value ไว้ในวงเล็บ signature ใช้เป็นเอกสารในตัวและแก้ไขค่าผ่าน `defer` ได้ ส่วน naked return (`return` เปล่า) ควรใช้อย่างระมัดระวังในฟังก์ชันสั้นๆ เท่านั้น เพื่อไม่ให้โค้ดอ่านยาก
- ฟังก์ชันเป็น first-class citizens: เก็บลงตัวแปร, ส่งเป็น argument, หรือ return ออกจากฟังก์ชันอื่นได้ — function type ประกอบด้วย signature ของ parameter และ return value เท่านั้น
- Higher-order function (ฟังก์ชันที่รับ/return ฟังก์ชันอื่น) เป็น pattern สำคัญที่ใช้แยก logic การวนลูปออกจาก logic การประมวลผล
- Selector pattern (return ฟังก์ชันที่มีอยู่แล้วตามเงื่อนไข) ต่างจาก closure ตรงที่ไม่มีการ capture ตัวแปรใดๆ — function value เทียบกับ `nil` ได้ แต่เทียบกับฟังก์ชันอื่นโดยตรงไม่ได้
- Convention มาตรฐานของ Go คือให้ `error` เป็นค่า return ตัวสุดท้ายเสมอเมื่อฟังก์ชันอาจล้มเหลวได้ (รายละเอียดเต็มใน Part 015)
- Doc comment ที่วางติดกับ `func` และขึ้นต้นด้วยชื่อฟังก์ชัน จะถูก `go doc` และ pkg.go.dev ดึงไปแสดงเป็นเอกสารอัตโนมัติ — ควรเขียนกำกับทุกฟังก์ชันที่ export

## แบบฝึกหัดท้ายบท

1. เขียนฟังก์ชัน `isPrime(n int) bool` ที่ตรวจสอบว่าจำนวนที่รับเข้ามาเป็นจำนวนเฉพาะหรือไม่ แล้วเขียนอีกฟังก์ชันหนึ่งที่ใช้ `isPrime` เป็น higher-order function argument เพื่อกรองจำนวนเฉพาะออกจาก slice ของ int
2. เขียนฟังก์ชัน `stats(nums []int) (sum, avg, max int)` ที่ return ค่าผลรวม ค่าเฉลี่ย และค่ามากที่สุดพร้อมกันโดยใช้ named return values แล้วทดลองเขียนทั้งแบบ naked return และแบบ return ระบุค่าชัดเจน เปรียบเทียบว่าแบบไหนอ่านง่ายกว่ากัน
3. เขียนฟังก์ชัน `safeSqrt(x float64) (float64, error)` ที่ return error เมื่อ `x` เป็นค่าลบ (เพราะรากที่สองของจำนวนลบไม่นิยามในจำนวนจริง) แล้วเขียน `main` ที่เรียกใช้และจัดการ error ตาม convention `if err != nil`
4. เขียนฟังก์ชันที่รับ `func(int) int` เป็น parameter ชื่อ `compose(f, g func(int) int) func(int) int` ที่ return ฟังก์ชันใหม่ซึ่งเทียบเท่ากับการเรียก `f(g(x))` แล้วทดลองใช้กับ `square` และ `double`
5. อธิบายด้วยคำพูดตัวเองว่าทำไม Go ถึงไม่มี named arguments ตอนเรียกฟังก์ชัน และยกตัวอย่างว่าเมื่อไรควรใช้ struct parameter แทนเพื่อความชัดเจน
6. เขียนฟังก์ชัน `operation(op string) func(int, int) int` แบบ selector pattern ที่รองรับ `"+"`, `"-"`, `"*"`, `"/"` (กรณี `"/"` ต้องคิดว่าจะจัดการหารด้วยศูนย์อย่างไร) แล้วเขียน doc comment กำกับฟังก์ชันนี้ตามแนวทาง godoc ที่เรียนในบทนี้

---

**ต่อไป**: [Part 009 — Functions ขั้นสูง: variadic, closures, `defer`](./009-functions-advanced.md)
