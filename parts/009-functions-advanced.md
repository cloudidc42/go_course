# Part 009: Functions ขั้นสูง: variadic, closures, `defer`

> ภาคที่ 1: พื้นฐานภาษา Go (Fundamentals) — ตอนที่ 9 จาก 15

## สารบัญของบทนี้

1. Variadic parameters (`...T`)
2. Spread slice เข้า variadic ด้วย `...`
3. Closures — ฟังก์ชันที่จดจำตัวแปรภายนอกได้
4. Classic loop-closure gotcha และการเปลี่ยนแปลงใน Go 1.22
5. `defer` คืออะไร และลำดับการทำงานแบบ LIFO
6. เวลาที่ argument ของ `defer` ถูก evaluate
7. Use case ที่พบบ่อยของ `defer`
8. Gotcha: `defer` ใน loop
9. สรุปสิ่งที่ได้เรียนในบทนี้
10. แบบฝึกหัดท้ายบท

---

## 1. Variadic parameters (`...T`)

**Variadic parameter** คือ parameter ที่รับ argument ได้ **จำนวนไม่จำกัด** (ตั้งแต่ 0 ตัวขึ้นไป) โดยประกาศด้วย `...` นำหน้า type ภายในฟังก์ชัน parameter นี้จะถูกมองเป็น **slice** ของ type นั้นโดยอัตโนมัติ

```go
package main

import "fmt"

func sum(nums ...int) int {
	total := 0
	for _, n := range nums {
		total += n
	}
	return total
}

func main() {
	fmt.Println(sum())           // 0 - ไม่ส่ง argument เลยก็ได้
	fmt.Println(sum(1))          // 1
	fmt.Println(sum(1, 2, 3))    // 6
	fmt.Println(sum(1, 2, 3, 4)) // 10
}
```

ภายในฟังก์ชัน `sum` ตัวแปร `nums` มี type เป็น `[]int` ธรรมดา ใช้ `range`, `len`, index ได้ทุกอย่างเหมือน slice ปกติที่เรียนไปใน Part 006 ความแตกต่างอยู่ที่**ฝั่งผู้เรียก**เท่านั้น ที่สามารถส่ง argument แยกกันทีละตัว คั่นด้วย comma ได้โดยไม่ต้องสร้าง slice เอง

ฟังก์ชัน `fmt.Println` เองก็เป็นตัวอย่างที่ใช้ variadic parameter: signature จริงคือ `func Println(a ...any) (n int, err error)` นี่คือเหตุผลที่เราส่ง argument กี่ตัวก็ได้เข้า `fmt.Println` มาตั้งแต่ Part 001

### ข้อจำกัด: variadic parameter ต้องเป็นตัวสุดท้ายเท่านั้น

ฟังก์ชันหนึ่งมี variadic parameter ได้แค่ตัวเดียว และต้องอยู่ในตำแหน่งสุดท้ายของ parameter list เสมอ แต่ผสมกับ fixed parameter ตัวอื่นที่อยู่ก่อนหน้าได้:

```go
func joinWithPrefix(prefix string, items ...string) string {
	result := ""
	for _, item := range items {
		result += prefix + "-" + item + " "
	}
	return result
}

fmt.Println(joinWithPrefix("item", "a", "b", "c"))
// item-a item-b item-c
```

---

## 2. Spread slice เข้า variadic ด้วย `...`

ถ้าข้อมูลที่มีอยู่แล้วเป็น slice (ไม่ใช่ค่าแยกทีละตัว) แต่ต้องการส่งเข้าฟังก์ชันที่รับ variadic parameter สามารถ **"spread" slice นั้นออกเป็น argument แยกกัน** ได้โดยใส่ `...` ต่อท้ายชื่อ slice ตอนเรียกฟังก์ชัน

```go
package main

import "fmt"

func sum(nums ...int) int {
	total := 0
	for _, n := range nums {
		total += n
	}
	return total
}

func main() {
	nums := []int{10, 20, 30}
	fmt.Println(sum(nums...)) // 60 - spread slice ทั้งก้อนเป็น argument
}
```

`sum(nums...)` แตกต่างจาก `sum(nums)` โดยสิ้นเชิง — `sum(nums)` จะ compile error ทันทีเพราะพยายามส่ง `[]int` ทั้งก้อนเป็น argument ตัวเดียวให้ parameter ที่คาดหวัง `int` ทีละตัว ในขณะที่ `sum(nums...)` คือการบอก compiler ว่า "แตก slice นี้ออกเป็น argument หลายตัว"

รูปแบบนี้ใช้บ่อยมากเมื่อต้องการส่งต่อ (forward) ข้อมูลที่มีอยู่แล้วในรูป slice ให้ฟังก์ชัน variadic โดยไม่ต้องวนลูปส่งเองทีละตัว รวมถึงใช้ในกรณี "wrap" ฟังก์ชัน variadic ด้วยฟังก์ชันอื่นที่รับ slice เข้ามาแล้วส่งต่อทั้งก้อน:

```go
func sumAll(groups ...[]int) int {
	total := 0
	for _, group := range groups {
		total += sum(group...) // spread แต่ละ group เข้า sum
	}
	return total
}
```

---

## 3. Closures — ฟังก์ชันที่จดจำตัวแปรภายนอกได้

**Closure** คือฟังก์ชัน (โดยเฉพาะ anonymous function ตามที่แนะนำไปใน Part 008) ที่ "จดจำ" และเข้าถึงตัวแปรจาก scope ที่ห่อหุ้มมันอยู่ได้ แม้ฟังก์ชันภายนอกที่สร้าง closure นั้นจะทำงานจบไปแล้วก็ตาม

```go
package main

import "fmt"

func makeCounter() func() int {
	count := 0
	return func() int {
		count++ // closure นี้ "จำ" ตัวแปร count จากฟังก์ชัน makeCounter ได้
		return count
	}
}

func main() {
	counter1 := makeCounter()
	fmt.Println(counter1()) // 1
	fmt.Println(counter1()) // 2
	fmt.Println(counter1()) // 3

	counter2 := makeCounter() // ตัวใหม่ มี count เป็นของตัวเอง แยกจาก counter1 โดยสิ้นเชิง
	fmt.Println(counter2())   // 1
	fmt.Println(counter1())   // 4 - counter1 ไม่ถูกกระทบโดย counter2
}
```

สิ่งที่เกิดขึ้นเบื้องหลัง: ทุกครั้งที่เรียก `makeCounter()` Go จะสร้างตัวแปร `count` ขึ้นใหม่ในหน่วยความจำ (บน heap หากจำเป็น — จะอธิบายเรื่อง escape analysis ใน Part 010) แล้ว anonymous function ที่ return ออกไปจะถือ **reference** ไปยังตัวแปร `count` ตัวนั้นตลอดไป ไม่ใช่ copy ค่าออกไป ดังนั้นทุกครั้งที่เรียก closure ที่ return มา มันจะแก้ไขและอ่านค่า `count` ตัวเดิมนั้นซ้ำๆ ทำให้ `counter1` และ `counter2` ต่างก็มี `count` เป็นของตัวเอง ไม่ปะปนกัน

นี่คือหัวใจสำคัญของ closure: **capture ตัวแปรโดยอ้างอิง (by reference) ไม่ใช่โดยค่า (by value)** ทุก closure ที่ capture ตัวแปรเดียวกันจะเห็นการเปลี่ยนแปลงของตัวแปรนั้นร่วมกันเสมอ ซึ่งเป็นทั้งจุดแข็ง (ใช้ทำ state ที่ซ่อนอยู่ภายในได้ เช่น counter, cache, memoization) และเป็นจุดที่ทำให้เกิด bug บ่อยเมื่อใช้ใน loop ตามหัวข้อถัดไป

---

## 4. Classic loop-closure gotcha และการเปลี่ยนแปลงใน Go 1.22

ปัญหาคลาสสิกที่โปรแกรมเมอร์ Go แทบทุกคนเคยเจอคือการสร้าง closure จำนวนมากภายใน loop โดยหวังว่าแต่ละตัวจะ capture ค่าของตัวแปร loop ในรอบนั้นๆ ไว้

ตามที่กล่าวไปใน Part 005 **ตั้งแต่ Go 1.22 เป็นต้นไป ตัวแปร loop ใน `for` จะถูกสร้างเป็นตัวแปรใหม่ทุก iteration** ทำให้พฤติกรรมของ closure ในลูปเป็นไปตามสัญชาตญาณของโปรแกรมเมอร์ส่วนใหญ่:

```go
package main

import "fmt"

func main() {
	var funcs []func() int
	for i := 0; i < 3; i++ {
		funcs = append(funcs, func() int {
			return i
		})
	}

	for _, f := range funcs {
		fmt.Println(f())
	}
}
```

ผลลัพธ์บน Go 1.22 ขึ้นไป (เวอร์ชันที่หลักสูตรนี้ใช้):

```
0
1
2
```

closure แต่ละตัวใน slice `funcs` จับค่า `i` ของ iteration ตัวเอง เพราะ Go สร้างตัวแปร `i` ใหม่ทุกรอบ (คนละหน่วยความจำ) ทำให้แต่ละ closure ไม่ได้ share ตัวแปรเดียวกัน

### แล้ว gotcha คืออะไร?

**ก่อน Go 1.22** (เวอร์ชัน 1.21 ลงไป) ตัวแปร loop `i` จะถูกสร้างขึ้น**เพียงครั้งเดียว**แล้วนำมาใช้ซ้ำทุก iteration (แค่แก้ไขค่าในตำแหน่งความจำเดิม) ผลคือโค้ดแบบเดียวกันข้างบนบน Go เวอร์ชันเก่าจะพิมพ์ `3 3 3` แทน (ทุก closure capture ตัวแปรตัวเดียวกัน แล้วพอถึงเวลาเรียกจริง ค่าตัวแปรก็กลายเป็นค่าสุดท้ายหลัง loop จบไปแล้ว) นี่คือ "loop variable capture gotcha" ที่มีชื่อเสียงที่สุดอันหนึ่งของภาษา Go และเป็นสาเหตุของ bug จำนวนมากในโค้ด production ก่อนหน้านี้

เราสามารถจำลองพฤติกรรมแบบเดียวกันได้ในทุกเวอร์ชันของ Go เมื่อ closure หลายตัว **capture ตัวแปรร่วมกันโดยตั้งใจ** (ไม่ใช่ตัวแปร loop ของ `for` เอง แต่เป็นตัวแปรที่ประกาศไว้นอกลูปแล้วถูกเขียนทับซ้ำในทุกรอบ):

```go
package main

import "fmt"

func main() {
	var funcs []func() int
	shared := 0
	for i := 0; i < 3; i++ {
		shared = i // ตัวแปร shared ตัวเดียวถูกเขียนทับทุกรอบ
		funcs = append(funcs, func() int {
			return shared // closure ทุกตัว capture ตัวแปร shared ตัวเดียวกัน (by reference)
		})
	}

	for _, f := range funcs {
		fmt.Println(f())
	}
}
```

ผลลัพธ์:

```
2
2
2
```

เพราะ `shared` เป็นตัวแปรเดียวที่ถูก capture ร่วมกันโดยทุก closure และเมื่อเรียก `f()` ในภายหลัง (หลัง loop จบไปแล้ว) `shared` มีค่าสุดท้ายคือ `2` ทุก closure จึงเห็นค่าเดียวกันหมด ตัวอย่างนี้แสดงให้เห็นชัดเจนว่าสาเหตุที่แท้จริงของ gotcha ไม่ใช่เรื่อง `for` โดยตรง แต่เป็นเรื่อง **closure capture ตัวแปรโดยอ้างอิงเสมอ** — Go 1.22 แค่เปลี่ยนให้ตัวแปร loop เป็นตัวแปรใหม่ทุกรอบ ทำให้ปัญหานี้ไม่เกิดกับตัวแปร loop เองอีกต่อไป

**ข้อควรระวังในทางปฏิบัติ**: ถ้าโค้ดของคุณหรือของทีมต้อง compile ด้วย Go เวอร์ชันต่ำกว่า 1.22 (เช็คได้จาก `go` directive ใน `go.mod` ตามที่เรียนไปใน Part 002) หรือทำงานร่วมกับโค้ดเก่าที่เขียนไว้ก่อนการเปลี่ยนแปลงนี้ ควรรู้จัก pattern แก้ปัญหาแบบดั้งเดิมไว้ด้วย คือการสร้างตัวแปร local ใหม่ในลูปก่อน capture:

```go
for i := 0; i < 3; i++ {
	i := i // สร้างตัวแปรใหม่ในแต่ละ iteration ก่อน capture (แก้ปัญหาบน Go < 1.22)
	funcs = append(funcs, func() int {
		return i
	})
}
```

Pattern `i := i` นี้ (ตั้งชื่อตัวแปรใหม่ทับชื่อเดิมใน scope ที่แคบกว่า) เป็นสิ่งที่พบเห็นได้ทั่วไปในโค้ด Go ที่เขียนก่อนปี 2024 แม้ปัจจุบันจะไม่จำเป็นแล้วบน Go 1.22+ แต่การเข้าใจว่าทำไม pattern นี้เคยจำเป็นก็ช่วยให้เข้าใจกลไก closure ได้ลึกซึ้งขึ้น

---

## 5. `defer` คืออะไร และลำดับการทำงานแบบ LIFO

`defer` เป็น keyword พิเศษของ Go ใช้ **เลื่อนการทำงานของฟังก์ชันไปทำตอนใกล้จะออกจากฟังก์ชันปัจจุบัน** ไม่ว่าฟังก์ชันปัจจุบันจะจบแบบ `return` ปกติ หรือจบเพราะ panic (จะเรียนเจาะลึกใน Part 017) ก็ตาม

```go
func doSomething() {
	defer fmt.Println("this runs last")
	fmt.Println("this runs first")
}
```

เมื่อมี `defer` หลายตัวในฟังก์ชันเดียวกัน จะถูกเรียกทำงานตามลำดับ **LIFO (Last In, First Out)** คือตัวที่ `defer` ไว้ล่าสุด จะถูกเรียกทำงาน**ก่อน**:

```go
package main

import "fmt"

func main() {
	fmt.Println("start")
	defer fmt.Println("defer 1")
	defer fmt.Println("defer 2")
	defer fmt.Println("defer 3")
	fmt.Println("end")
}
```

ผลลัพธ์:

```
start
end
defer 3
defer 2
defer 1
```

สังเกตว่า `defer` ทั้งสามบรรทัดถูก "จอง" ไว้ทำงานทันทีที่เจอ statement แต่ตัว**การทำงานจริง**ถูกเลื่อนออกไปจนกว่าฟังก์ชัน `main` จะจบ (ถึงบรรทัดสุดท้าย) แล้วเรียงลำดับย้อนกลับจากตัวล่าสุดไปตัวแรกสุด เหมือนกับการวางจานซ้อนกันเป็น stack — จานที่วางล่าสุดจะถูกหยิบออกก่อน

พฤติกรรม LIFO นี้สำคัญมากเมื่อมีทรัพยากรที่ต้องปิด/ปลดล็อกตามลำดับย้อนกลับจากที่เปิด/ล็อกไว้ เช่น เปิด lock ชั้นนอกก่อนแล้วค่อยล็อกชั้นใน `defer` การปลดล็อกจะปลดชั้นในก่อน แล้วค่อยปลดชั้นนอก ซึ่งเป็นลำดับที่ถูกต้องเสมอ

---

## 6. เวลาที่ argument ของ `defer` ถูก evaluate

จุดที่มือใหม่มักสับสนคือ **argument ของฟังก์ชันที่ `defer` ไว้ จะถูก evaluate (คำนวณค่า) ทันทีตอนที่ `defer` statement นั้นทำงาน** ไม่ใช่ตอนที่ฟังก์ชันถูกเรียกจริงในภายหลัง

```go
package main

import "fmt"

func main() {
	i := 1
	defer fmt.Println("deferred i =", i) // i ถูก evaluate ณ บรรทัดนี้ทันที (ได้ค่า 1)
	i = 2
	fmt.Println("current i =", i)
}
```

ผลลัพธ์:

```
current i = 2
deferred i = 1
```

แม้ `i` จะถูกเปลี่ยนเป็น `2` หลังจากบรรทัด `defer` แล้ว แต่ค่าที่ `fmt.Println` (ที่ถูก defer ไว้) จะพิมพ์ออกมาคือ `1` เพราะค่านั้นถูก "จับภาพ" ไว้ตั้งแต่ตอนอ่าน argument ของ `defer` statement เอง — เปรียบเทียบง่ายๆ คือ Go คำนวณ argument ทั้งหมดของฟังก์ชันที่ defer ไว้ทันที แล้วเก็บผลลัพธ์นั้นรอไว้เรียกในภายหลังเท่านั้น ตัวฟังก์ชันเองยังไม่ถูกเรียกจนกว่าจะถึงเวลาที่กำหนด

### ถ้าต้องการอ่านค่าล่าสุด ณ เวลาที่ทำงานจริง ต้องใช้ closure

ถ้าต้องการให้ค่าที่พิมพ์เป็นค่า ณ เวลาที่ `defer` ทำงานจริง (ไม่ใช่ตอน `defer` statement ถูกอ่าน) ต้องห่อด้วย anonymous function (closure) แทน เพราะ **closure จะ evaluate ตัวแปรภายในตอนถูกเรียกจริง ไม่ใช่ตอนสร้าง**:

```go
package main

import "fmt"

func main() {
	i := 1
	defer func() {
		fmt.Println("deferred (closure) i =", i) // อ่านค่า i ตอนฟังก์ชันทำงานจริง
	}()
	i = 2
	fmt.Println("current i =", i)
}
```

ผลลัพธ์:

```
current i = 2
deferred (closure) i = 2
```

ความแตกต่างนี้สำคัญมากในทางปฏิบัติ: **ถ้า argument ของ `defer` เป็นค่าธรรมดา (เช่นตัวแปร, ผลลัพธ์จากฟังก์ชัน) จะถูก freeze ค่าไว้ทันที แต่ถ้าใช้ closure ห่อไว้ ค่าภายในจะอ้างอิงถึงตัวแปรจริงเสมอ** เทคนิคการใช้ closure ร่วมกับ `defer` นี้เป็นสิ่งที่จำเป็นสำหรับการแก้ไข named return values ตามที่เห็นไปแล้วใน Part 008 (ตัวอย่าง `increment()`) และสำหรับการทำ `recover()` ในหัวข้อถัดไป

---

## 7. Use case ที่พบบ่อยของ `defer`

### 7.1 ปิด resource เสมอ (file, connection)

use case ที่พบบ่อยที่สุดของ `defer` คือการรับประกันว่าทรัพยากรที่เปิดไว้ (ไฟล์, network connection, database connection) จะถูกปิดเสมอไม่ว่าฟังก์ชันจะจบแบบไหน:

```go
package main

import (
	"fmt"
	"os"
)

func readFile(path string) error {
	f, err := os.Open(path)
	if err != nil {
		return err
	}
	defer f.Close() // รับประกันว่าไฟล์จะถูกปิดก่อนออกจากฟังก์ชันเสมอ ไม่ว่าจะ return ตรงไหน

	buf := make([]byte, 32)
	_, err = f.Read(buf)
	if err != nil {
		return err // f.Close() จะยังถูกเรียกก่อน return นี้จริงๆ
	}
	fmt.Println("read ok")
	return nil
}

func main() {
	os.WriteFile("test.txt", []byte("hello world"), 0644)
	defer os.Remove("test.txt")

	if err := readFile("test.txt"); err != nil {
		fmt.Println("error:", err)
	}
}
```

จุดสำคัญคือการวาง `defer f.Close()` **ทันทีหลังจากเปิดไฟล์สำเร็จ** ไม่ใช่รอไปวางท้ายฟังก์ชัน เพราะถ้าฟังก์ชันมีจุด `return` หลายจุด (เช่นตอนเจอ error) การวาง `defer` ไว้ใกล้จุดที่ได้ resource มา ช่วยให้มั่นใจได้ว่าทุก code path จะปิด resource นั้นแน่นอน ไม่มีทางลืม เราจะเห็น pattern นี้อีกครั้งแบบเจาะลึกเรื่อง file I/O ใน Part 024

### 7.2 ปลดล็อก mutex (preview — เจาะลึกใน Part 039)

รูปแบบที่พบบ่อยไม่แพ้กันในโค้ด concurrent คือการล็อกแล้ว `defer` การปลดล็อกทันที:

```go
var mu sync.Mutex

func updateBalance() {
	mu.Lock()
	defer mu.Unlock() // ปลดล็อกเสมอ แม้จะ return กลางฟังก์ชันหรือ panic
	// ... แก้ไขข้อมูลที่ต้องป้องกันด้วย lock ...
}
```

รายละเอียดเรื่อง `sync.Mutex` และการทำงานพร้อมกันจะเรียนเต็มรูปแบบในภาคที่ 3 (Concurrency) โดยเฉพาะ Part 039

### 7.3 `recover()` จาก panic (preview — เจาะลึกใน Part 017)

`defer` เป็นกลไกเดียวที่ทำให้เรียก `recover()` เพื่อดัก panic ได้ (ต้องเรียกภายในฟังก์ชันที่ถูก `defer` ไว้เท่านั้น) นี่คือวิธีเดียวใน Go ที่ทำให้ฟังก์ชันหนึ่ง "รอด" จาก panic ที่เกิดขึ้นภายในได้:

```go
package main

import "fmt"

func safeDivide(a, b int) (result int, err error) {
	defer func() {
		if r := recover(); r != nil {
			err = fmt.Errorf("recovered from panic: %v", r)
		}
	}()

	result = a / b // ถ้า b == 0 จะ panic: integer divide by zero
	return result, nil
}

func main() {
	r, err := safeDivide(10, 2)
	fmt.Println(r, err) // 5 <nil>

	r2, err2 := safeDivide(10, 0)
	fmt.Println(r2, err2) // 0 recovered from panic: runtime error: integer divide by zero
}
```

ตัวอย่างนี้ผสมผสาน 3 concept ที่เรียนมาในสอง part นี้เข้าด้วยกัน: **named return values** (จาก Part 008, ทำให้ `err` แก้ไขได้จากใน `defer`), **closure** (ฟังก์ชันที่ `defer` ไว้เข้าถึงและแก้ไข `err` ของฟังก์ชันแม่ได้), และ **defer** เอง (รับประกันว่า `recover()` จะถูกเรียกตรวจสอบเสมอก่อนฟังก์ชันจบจริง) กลไก `panic`/`recover` แบบเต็มรูปแบบจะเรียนใน **Part 017**

---

## 8. Gotcha: `defer` ใน loop

ข้อผิดพลาดที่พบบ่อยอีกอย่างคือการใช้ `defer` ภายใน loop โดยคาดหวังว่าจะทำงาน**ทันทีเมื่อจบแต่ละรอบ** แต่ความจริงแล้ว **`defer` จะไม่ทำงานจนกว่าทั้งฟังก์ชันที่ล้อมรอบ loop นั้นจะจบ** ไม่ใช่จบแค่ในแต่ละ iteration

```go
package main

import (
	"fmt"
	"os"
)

func badExample() {
	for i := 0; i < 3; i++ {
		f, err := os.CreateTemp("", "example")
		if err != nil {
			continue
		}
		defer f.Close() // จะไม่ถูกเรียกจนกว่า badExample() ทั้งฟังก์ชันจะ return
		defer os.Remove(f.Name())
		fmt.Println("opened", f.Name())
	}
	fmt.Println("all files still open here - defer ยังไม่ทำงานสักตัว")
}
```

ถ้า loop นี้วนหลายพันรอบ (เช่น เปิดไฟล์หรือ connection จำนวนมากในลูปเดียว) จะเกิดปัญหา **resource leak ชั่วคราว**: ไฟล์/connection ทั้งหมดจะยังคงเปิดค้างอยู่จนกว่าฟังก์ชันทั้งหมดจะ return ซึ่งอาจทำให้ระบบชนขีดจำกัดจำนวน file descriptor หรือ connection ที่เปิดพร้อมกันได้ในกรณีที่ loop มีขนาดใหญ่

### วิธีแก้: ย้าย logic ที่มี resource + defer ไปไว้ในฟังก์ชันย่อย

```go
func goodExample() {
	for i := 0; i < 3; i++ {
		func() { // anonymous function ที่เรียกทันที (IIFE-style)
			f, err := os.CreateTemp("", "example")
			if err != nil {
				return
			}
			defer f.Close() // ทำงานทันทีเมื่อ anonymous function นี้จบในแต่ละรอบ loop
			defer os.Remove(f.Name())
			fmt.Println("opened and closed", f.Name())
		}()
	}
}
```

ด้วยการห่อ body ของ loop ไว้ในฟังก์ชันไม่มีชื่อที่เรียกทันที (immediately-invoked function) `defer` ที่อยู่ข้างในจะทำงานทันทีที่ anonymous function นั้นจบในแต่ละรอบ ไม่ต้องรอให้ loop วนจบทั้งหมด — อีกทางเลือกหนึ่งที่พบบ่อยไม่แพ้กันคือการแยก body ของ loop ออกเป็นฟังก์ชันแยกต่างหาก (named function) แล้วเรียกใช้ในลูปแทน ซึ่งให้ผลลัพธ์เดียวกันแต่โค้ดอ่านง่ายกว่าเมื่อ logic ซับซ้อนขึ้น

**กฎจำง่ายๆ**: `defer` ผูกกับ**การจบของฟังก์ชันที่มันอยู่** เสมอ ไม่ใช่ผูกกับ block หรือ loop ที่ล้อมรอบ ถ้าต้องการให้ทำงานทุกรอบของ loop ต้องสร้างขอบเขตฟังก์ชันใหม่ให้แต่ละรอบ

---

## สรุปสิ่งที่ได้เรียนในบทนี้

- Variadic parameter (`...T`) รับ argument ได้ไม่จำกัดจำนวน และถูกมองเป็น slice ภายในฟังก์ชัน ต้องอยู่เป็น parameter ตัวสุดท้ายเท่านั้น
- Spread slice ที่มีอยู่แล้วเข้า variadic parameter ได้ด้วย `slice...` ตอนเรียกฟังก์ชัน
- Closure คือฟังก์ชันที่ capture ตัวแปรจาก scope ภายนอกโดยอ้างอิง (by reference) ทำให้แก้ไขและอ่านค่าตัวแปรนั้นร่วมกันได้แม้ฟังก์ชันแม่จะจบไปแล้ว
- ตั้งแต่ Go 1.22 ตัวแปร loop เป็นตัวใหม่ทุก iteration ทำให้ classic loop-closure gotcha (ที่ทุก closure เห็นค่าตัวสุดท้ายเหมือนกันหมด) ไม่เกิดขึ้นกับตัวแปร loop อีกต่อไป แต่ยังเกิดได้ถ้า closure หลายตัว capture ตัวแปรร่วมกันโดยตั้งใจ
- `defer` เลื่อนการทำงานของฟังก์ชันไปจนใกล้ฟังก์ชันปัจจุบันจะจบ และทำงานตามลำดับ **LIFO** เมื่อมีหลายตัว
- Argument ของฟังก์ชันที่ `defer` ไว้ถูก evaluate ทันทีตอน `defer` statement ทำงาน ไม่ใช่ตอนเรียกจริง — ถ้าต้องการอ่านค่าล่าสุด ณ เวลาทำงานจริง ต้องห่อด้วย closure
- Use case หลักของ `defer`: ปิดไฟล์/connection, ปลดล็อก mutex, และเรียก `recover()` เพื่อดัก panic (preview ของ Part 017)
- `defer` ผูกกับการจบของฟังก์ชัน ไม่ใช่จบของ loop หรือ block — ถ้าใช้ใน loop โดยตรงจะสะสมรอจนฟังก์ชันทั้งหมดจบ ต้องห่อ body ของ loop ด้วยฟังก์ชันแยกถ้าต้องการให้ `defer` ทำงานทุกรอบ

## แบบฝึกหัดท้ายบท

1. เขียนฟังก์ชัน `average(nums ...float64) float64` ที่คำนวณค่าเฉลี่ยจาก argument จำนวนเท่าไรก็ได้ แล้วทดลองเรียกทั้งแบบส่งค่าตรงๆ และแบบ spread จาก `[]float64` ที่มีอยู่แล้ว
2. เขียนฟังก์ชัน `makeMultiplier(factor int) func(int) int` ที่ return closure สำหรับคูณตัวเลขด้วย `factor` ที่กำหนด แล้วสร้างตัวคูณ 2 และตัวคูณ 3 พิสูจน์ว่าทั้งสองตัวทำงานอิสระจากกัน
3. เขียนโค้ดที่สร้าง slice ของ closure จำนวน 5 ตัวใน loop โดยแต่ละตัว capture ค่า index ของตัวเอง (ทดสอบบน Go เวอร์ชันปัจจุบันที่เป็น 1.22+) แล้วอธิบายว่าถ้ารันโค้ดเดียวกันนี้บน Go 1.21 จะได้ผลลัพธ์อะไรและเพราะอะไร
4. เขียนฟังก์ชันที่มี `defer` 4 ตัวเรียงกัน โดยแต่ละตัวพิมพ์ตัวเลขต่างกัน แล้วคาดเดาลำดับผลลัพธ์ก่อนรัน แล้วรันจริงเพื่อตรวจคำตอบ
5. เขียนฟังก์ชันที่เปิดไฟล์ชั่วคราวหลายไฟล์ภายใน loop (ใช้ `os.CreateTemp`) ทั้งแบบที่ `defer` ตรงๆ ในลูป (ผิด) และแบบที่ห่อด้วย anonymous function (ถูก) แล้วสังเกตความแตกต่างของเวลาที่ไฟล์ถูกปิดจริง
6. เขียนฟังก์ชัน `safeCall(f func())` ที่รับฟังก์ชันเข้ามาแล้วเรียกมันภายใต้การป้องกันด้วย `recover()` เพื่อไม่ให้ panic จากภายใน `f` ทำให้โปรแกรมทั้งโปรแกรมล่ม แล้วทดลองส่งฟังก์ชันที่ panic เข้าไปทดสอบ

---

**ต่อไป**: [Part 010 — Pointers เจาะลึก](./010-pointers.md)
