# Part 017: Panic, Recover และการจัดการข้อผิดพลาดร้ายแรง

> ภาคที่ 2: ระดับกลาง (Intermediate) — ตอนที่ 2 จาก 20 (Part 16–35)

## สารบัญของบทนี้

1. Panic คืออะไร และไม่ใช่ `error` ตัวเดิมที่เรารู้จัก
2. อะไรทำให้เกิด panic บ้าง
3. Panic ทำให้ stack "unwind" อย่างไร และ `defer` ทำงานตอนนั้นยังไง
4. `recover()`: ทำไมต้องอยู่ใน deferred function เท่านั้น
5. แปลง panic เป็น error ที่ขอบเขตของ API (API boundary)
6. panic/recover ไม่ใช่กลไก exception handling — ควรใช้เมื่อไหร่จริงๆ
7. Panic ใน goroutine: ทำไมถึงทำทั้งโปรแกรมล่มได้
8. ตัวอย่างจริง: recover ใน HTTP middleware
9. แนวทางปฏิบัติและข้อผิดพลาดที่พบบ่อย
10. สรุปสิ่งที่ได้เรียนในบทนี้
11. แบบฝึกหัดท้ายบท

---

## 1. Panic คืออะไร และไม่ใช่ `error` ตัวเดิมที่เรารู้จัก

ตลอด Part 015–016 เราเรียนรู้ว่า Go จัดการความผิดพลาด "ที่คาดการณ์ได้" (expected errors) ผ่านค่า `error` ที่ return ออกมาแบบ explicit — ไฟล์เปิดไม่ได้, เชื่อมต่อ database ไม่สำเร็จ, input ไม่ผ่าน validation ล้วนเป็นสถานการณ์ปกติที่**โปรแกรมควรรับมือได้** และเขียนโค้ดจัดการต่อได้ตามปกติ

แต่มีความผิดพลาดอีกประเภทหนึ่งที่ต่างออกไปโดยสิ้นเชิง คือความผิดพลาดที่**ทำให้โปรแกรมอยู่ในสถานะที่ไม่สามารถทำงานต่อได้อย่างปลอดภัยอีกต่อไป** เช่น เข้าถึง index ที่ไม่มีอยู่จริงใน slice, หารด้วยศูนย์, หรือเรียก method ผ่าน pointer ที่เป็น `nil` — สถานการณ์เหล่านี้มักเป็นสัญญาณของ**บั๊กในตัวโปรแกรมเมอร์เอง** ไม่ใช่สถานการณ์ที่ควรถูกคาดหวังให้เกิดขึ้นและจัดการอย่างสุภาพเหมือน error ปกติ

Go จัดการความผิดพลาดประเภทนี้ด้วยกลไกที่เรียกว่า **panic** ซึ่งทำงานต่างจาก `error` อย่างสิ้นเชิง:

| | `error` | `panic` |
|---|---|---|
| การส่งต่อ | ต้อง return และเช็คด้วยมือทุกจุด (`if err != nil`) | ลอยขึ้นไปเองอัตโนมัติผ่าน call stack |
| ใช้เมื่อไหร่ | สถานการณ์ที่คาดการณ์ได้ ควรจัดการได้ตามปกติ | สถานการณ์ผิดปกติร้ายแรง ที่โปรแกรมไม่ควรทำงานต่อ |
| ผลลัพธ์ถ้าไม่จัดการ | โปรแกรมยังทำงานต่อไปได้ (แค่ error path ไม่ถูกจัดการ) | โปรแกรม**ทั้งตัว**จะ crash ทันที |
| วิธีจัดการ | `if err != nil` | `recover()` ภายใน deferred function เท่านั้น |

สิ่งสำคัญที่สุดที่ต้องจำไว้ตั้งแต่ต้นบทนี้: **panic ไม่ใช่ "exception" แบบ Java/Python ที่ควรใช้เป็น control flow ปกติ** เราจะอธิบายเหตุผลอย่างละเอียดในหัวข้อ 6

---

## 2. อะไรทำให้เกิด panic บ้าง

Panic เกิดขึ้นได้จากสองทางหลัก คือ **runtime panic** (ที่ Go runtime สร้างขึ้นเองโดยอัตโนมัติเมื่อเจอสถานการณ์ผิดปกติ) และ **explicit panic** (ที่โปรแกรมเมอร์เรียกเองด้วย `panic()`)

### Runtime panic ที่พบบ่อยที่สุด

```go
// 1. Index out of range
var s []int
fmt.Println(s[5]) // panic: runtime error: index out of range [5] with length 0

// 2. Nil pointer dereference
var p *int
fmt.Println(*p) // panic: runtime error: invalid memory address or nil pointer dereference

// 3. หารด้วยศูนย์ (เฉพาะจำนวนเต็ม — float หารด้วยศูนย์ไม่ panic แต่ได้ +Inf/NaN)
a, b := 10, 0
fmt.Println(a / b) // panic: runtime error: integer divide by zero

// 4. Type assertion ที่ผิดพลาด (ไม่ใช้ comma-ok form ที่เรียนใน Part 014)
var i interface{} = "hello"
n := i.(int) // panic: interface conversion: interface {} is string, not int

// 5. เข้าถึง map ที่เป็น nil เพื่อเขียนข้อมูล (อ่านได้ปกติ แต่เขียนไม่ได้)
var m map[string]int
m["key"] = 1 // panic: assignment to entry in nil map

// 6. ปิด channel ที่ถูกปิดไปแล้ว หรือปิด channel ที่เป็น nil
ch := make(chan int)
close(ch)
close(ch) // panic: close of closed channel
```

### Explicit panic

นอกจาก runtime panic ที่เกิดขึ้นเองแล้ว โปรแกรมเมอร์สามารถเรียก `panic()` ได้ด้วยตัวเองเมื่อต้องการส่งสัญญาณว่า "เกิดสถานการณ์ที่ไม่ควรเกิดขึ้นเลย":

```go
func mustPositive(n int) int {
	if n < 0 {
		panic(fmt.Sprintf("mustPositive: n ต้องไม่ติดลบ แต่ได้รับ %d", n))
	}
	return n
}
```

`panic()` รับ argument เป็น `interface{}` (หรือ `any` ตั้งแต่ Go 1.18) ชนิดใดก็ได้ — นิยมส่งเป็น `string` หรือ `error` เพื่อให้ข้อความอ่านง่ายเมื่อโปรแกรม crash

### สาธิต: จับ panic ด้วย `recover()` เพื่อดูข้อความ

```go
package main

import "fmt"

func main() {
	defer func() {
		if r := recover(); r != nil {
			fmt.Println("recovered:", r)
		}
	}()

	var s []int
	fmt.Println(s[5]) // index out of range จะทำให้เกิด panic
}
```

ผลลัพธ์:

```
recovered: runtime error: index out of range [5] with length 0
```

(เราจะอธิบาย `recover()` และ `defer` ในหัวข้อถัดไปอย่างละเอียด)

---

## 3. Panic ทำให้ stack "unwind" อย่างไร และ `defer` ทำงานตอนนั้นยังไง

จาก Part 009 เรารู้จัก `defer` แล้วว่าเป็นการเลื่อนการทำงานของฟังก์ชันไปจนกว่าฟังก์ชันปัจจุบันจะ return สิ่งที่สำคัญมากคือ **`defer` ทำงานแม้ฟังก์ชันจะจบเพราะ panic ก็ตาม** ไม่ใช่แค่ตอน return แบบปกติเท่านั้น

เมื่อเกิด panic ขึ้นในฟังก์ชันใดก็ตาม กระบวนการที่เรียกว่า **stack unwinding** จะเริ่มทำงาน:

1. ฟังก์ชันปัจจุบันหยุดทำงานทันที (บรรทัดถัดจากจุด panic จะไม่ถูกรันอีก)
2. ฟังก์ชันนั้นรัน **deferred function ทั้งหมดของตัวเอง** ตามลำดับ LIFO (Last In, First Out) เหมือนตอน return ปกติ
3. หลังจากนั้น panic จะ "ลอย" ขึ้นไปยังฟังก์ชันที่เรียกมัน (caller) — ฟังก์ชันนั้นก็จะหยุดทำงานทันทีเช่นกัน และรัน deferred function ของตัวเองต่อ
4. กระบวนการนี้ทำซ้ำไปเรื่อยๆ ขึ้นไปตาม call stack จนกว่าจะถึง `main()` หรือจนกว่าจะมี deferred function ใดสักตัวเรียก `recover()` สำเร็จ (อธิบายในหัวข้อ 4)
5. ถ้าไม่มีใคร `recover()` เลยจนถึงบนสุดของ stack โปรแกรมจะพิมพ์ panic message พร้อม stack trace แล้วจบการทำงานด้วย exit code ที่ไม่ใช่ 0

### สาธิต stack unwinding

```go
package main

import "fmt"

func step3() {
	fmt.Println("step3: กำลังจะ panic")
	panic("boom")
}

func step2() {
	defer fmt.Println("step2: deferred ทำงาน (ระหว่าง unwind)")
	step3()
	fmt.Println("step2: บรรทัดนี้จะไม่ถูกรัน")
}

func step1() {
	defer fmt.Println("step1: deferred ทำงาน (ระหว่าง unwind)")
	step2()
}

func main() {
	defer func() {
		if r := recover(); r != nil {
			fmt.Println("main: recovered จาก:", r)
		}
	}()
	defer fmt.Println("main: deferred ทำงาน (ระหว่าง unwind)")
	step1()
	fmt.Println("main: บรรทัดนี้จะไม่ถูกรัน")
}
```

ผลลัพธ์:

```
step3: กำลังจะ panic
step2: deferred ทำงาน (ระหว่าง unwind)
step1: deferred ทำงาน (ระหว่าง unwind)
main: deferred ทำงาน (ระหว่าง unwind)
main: recovered จาก: boom
```

สังเกตลำดับการทำงานอย่างละเอียด:

1. `step3` panic ทันทีที่เรียก `panic("boom")` — บรรทัดหลังจากนั้นใน `step3` (ถ้ามี) จะไม่ถูกรัน
2. `step2` ไม่มีโอกาสรันบรรทัด `"step2: บรรทัดนี้จะไม่ถูกรัน"` เลย เพราะ panic ทำให้ฟังก์ชันหยุดทันที **แต่ deferred function ของ `step2` ยังคงถูกรัน**
3. เช่นเดียวกัน `step1` ไม่รันบรรทัดหลัง `step2()` (ซึ่งไม่มีอยู่ในตัวอย่างนี้ แต่ deferred ของมันก็ยังทำงาน)
4. panic ลอยขึ้นไปถึง `main` ซึ่งมี deferred function สองตัว รันตามลำดับ LIFO: ตัวที่ประกาศทีหลัง (`fmt.Println`) รันก่อน ตามด้วยตัวที่ประกาศก่อน (anonymous function ที่มี `recover()`)
5. เมื่อ `recover()` ถูกเรียกสำเร็จใน deferred function ของ `main` **การ unwind หยุดทันที** โปรแกรมไม่ crash และทำงานต่อได้ตามปกติหลังจาก `main()` (ในที่นี้คือจบการทำงานแบบปกติเพราะไม่มีอะไรเหลือใน `main` แล้ว)

นี่คือเหตุผลที่ `defer` และ panic/recover ผูกกันแน่นมาก — **`defer` คือกลไกเดียวที่รับประกันว่า cleanup code (ปิดไฟล์, ปลด lock, ปิด connection) จะถูกรันแม้เกิด panic ขึ้นระหว่างทาง**

---

## 4. `recover()`: ทำไมต้องอยู่ใน deferred function เท่านั้น

`recover()` เป็น built-in function ที่ใช้ "หยุด" การ panic ที่กำลังเกิดขึ้น เมื่อเรียกสำเร็จ มันจะคืนค่าที่ถูกส่งให้ `panic()` (ค่า `interface{}`) และหยุด stack unwinding ทันที — ฟังก์ชันที่เรียก `recover()` จะ return ตามปกติหลังจากนั้น (ไม่ panic ต่อ)

**กฎที่สำคัญที่สุดของ `recover()` คือ: มันมีผลก็ต่อเมื่อถูกเรียก "ตรงๆ" ภายใน deferred function เท่านั้น** ถ้าเรียกในสถานการณ์อื่น (เรียกตรงๆ ในโค้ดปกติ, เรียกใน goroutine แยก, หรือเรียกผ่านฟังก์ชันอีกชั้นหนึ่งภายใน defer) `recover()` จะคืนค่า `nil` เสมอ และไม่มีผลใดๆ ต่อ panic ที่กำลังเกิดขึ้น

### ทำไมต้องออกแบบแบบนี้

เหตุผลเชิงออกแบบคือ **panic ต้องมีโอกาสรัน defer ของทุก stack frame ระหว่างทางก่อนเสมอ** ถ้า `recover()` เรียกได้จากที่ไหนก็ได้แบบอิสระ จะทำให้ควบคุมยากว่า panic ควรถูกดักที่ระดับไหน — การบังคับว่าต้องอยู่ใน `defer` ทำให้ชัดเจนว่า **การ recover ต้องเกิดที่จุดที่ตั้งใจดักไว้ล่วงหน้าเท่านั้น** ไม่ใช่ดักได้แบบสุ่มจากทุกที่เหมือน `catch` ในภาษาอื่น

### สาธิต: `recover()` ที่เรียกผิดที่ ไม่มีผลใดๆ

```go
package main

import "fmt"

func wrongRecover() {
	// เรียก recover() ตรงๆ โดยไม่อยู่ใน defer -- จะไม่มีผลใดๆ เลย
	if r := recover(); r != nil {
		fmt.Println("จะไม่มีทางมาถึงบรรทัดนี้:", r)
	}
	panic("บูม จาก wrongRecover")
}

func main() {
	defer func() {
		if r := recover(); r != nil {
			fmt.Println("main recovered:", r)
		}
	}()
	wrongRecover()
}
```

ผลลัพธ์:

```
main recovered: บูม จาก wrongRecover
```

สังเกตว่า `recover()` ตัวแรกใน `wrongRecover()` **ไม่มีผลอะไรเลย** เพราะ ณ จุดนั้นยังไม่มี panic เกิดขึ้น (มันถูกเรียกก่อนบรรทัด `panic()` ด้วยซ้ำ และไม่ได้อยู่ใน `defer`) panic จริงๆ เกิดที่บรรทัด `panic("บูม จาก wrongRecover")` แล้วลอยขึ้นไปถูกจับที่ `main` แทน

รูปแบบการใช้ที่ถูกต้องเสมอคือ:

```go
defer func() {
	if r := recover(); r != nil {
		// จัดการ panic ที่นี่
	}
}()
```

ต้องเป็น **anonymous function ที่ห่อ `recover()` ไว้ข้างในโดยตรง** และฟังก์ชันนั้นต้องถูกเรียกผ่าน `defer` — เขียนแบบอื่นจะใช้งานไม่ได้

---

## 5. แปลง panic เป็น error ที่ขอบเขตของ API (API boundary)

รูปแบบที่ใช้บ่อยมากในโค้ด production คือ: ฟังก์ชันหรือ library บางตัวอาจมีความเสี่ยงเกิด panic ภายใน (เช่น เรียก library ภายนอกที่ panic เวลา input ผิดรูปแบบ) แต่เราไม่อยากให้ panic นั้น**หลุดออกไปทำให้ caller หรือทั้งโปรแกรม crash** จึงออกแบบให้ฟังก์ชันนั้น **recover panic ของตัวเอง แล้วแปลงเป็น `error` ธรรมดา** ก่อนส่งกลับให้ caller — caller จึงไม่ต้องรู้เลยว่าข้างในมีการ panic เกิดขึ้น เห็นแค่ `error` แบบที่คุ้นเคยจาก Part 015–016

```go
package main

import (
	"errors"
	"fmt"
)

// safeDivide คือฟังก์ชัน library ที่แปลง panic ภายในให้เป็น error
// เพื่อไม่ให้ caller ต้องกังวลเรื่อง panic หลุดออกไปนอก package
func safeDivide(a, b int) (result int, err error) {
	defer func() {
		if r := recover(); r != nil {
			err = fmt.Errorf("safeDivide panic: %v", r)
		}
	}()
	result = a / b // ถ้า b เป็น 0 จะ panic: integer divide by zero
	return result, nil
}

func main() {
	r, err := safeDivide(10, 2)
	fmt.Println(r, err)

	r, err = safeDivide(10, 0)
	if err != nil {
		fmt.Println("จัดการ error ได้ตามปกติ ไม่ crash:", err)
	}
	fmt.Println("โปรแกรมทำงานต่อไปได้:", r)

	fmt.Println("errors.New เทียบ:", errors.New("test") != nil)
}
```

ผลลัพธ์:

```
5 <nil>
จัดการ error ได้ตามปกติ ไม่ crash: safeDivide panic: runtime error: integer divide by zero
โปรแกรมทำงานต่อไปได้: 0
```

จุดเทคนิคสำคัญที่ทำให้ตัวอย่างนี้ทำงานได้ถูกต้อง:

1. **ฟังก์ชันต้องใช้ named return value** (`result int, err error`) ไม่ใช่ return แบบไม่มีชื่อ เพราะ deferred function ต้อง**แก้ไขค่าที่จะถูก return จริง** ผ่านชื่อตัวแปรเหล่านี้ ถ้าใช้ `func safeDivide(a, b int) (int, error)` แบบไม่มีชื่อ เราจะไม่มีทาง assign ค่า `err` จากภายใน `defer` ได้เลย
2. `recover()` คืนค่าเป็น `interface{}` (หรือ `any`) เราจึงแปลงมันเป็นข้อความด้วย `%v` ใน `fmt.Errorf` แล้วเก็บใน `err`
3. รูปแบบนี้เหมาะมากสำหรับ**การเขียน library ที่ต้องพึ่งพา third-party code ซึ่งไม่รู้ว่าจะ panic เมื่อไหร่** เช่น การ parse ข้อมูลด้วย regex ที่ผิดพลาด, หรือการเรียกใช้ reflection (Part 031) ที่ผิดชนิด

### ข้อควรระวัง: อย่าใช้รูปแบบนี้พร่ำเพรื่อ

การ recover panic แล้วแปลงเป็น error **ควรทำเฉพาะที่ "ขอบเขต" ที่ชัดเจน** เช่น จุดเริ่มต้นของ library function, HTTP handler, หรือ worker goroutine — ไม่ควรทำในทุกฟังก์ชันย่อยภายใน เพราะจะทำให้บั๊กจริงถูกกลบด้วย error message ทั่วไป จนหาสาเหตุที่แท้จริงยากขึ้น (จะอธิบายเพิ่มในหัวข้อถัดไป)

---

## 6. panic/recover ไม่ใช่กลไก exception handling — ควรใช้เมื่อไหร่จริงๆ

โปรแกรมเมอร์ที่มาจากภาษาอื่น (Java, Python, C#, JavaScript) มักเข้าใจผิดว่า `panic`/`recover` คือ `try`/`catch` เวอร์ชัน Go แล้วพยายามใช้มันเป็น control flow ทั่วไปสำหรับจัดการ error — **นี่คือความเข้าใจผิดที่อันตรายที่สุดอย่างหนึ่งสำหรับคนเริ่มเขียน Go**

### ทำไมถึงไม่ควรใช้ panic แทน error

1. **Panic ทำลายความชัดเจนของ control flow** — จุดแข็งที่สุดของโมเดล `error` แบบ explicit ใน Go คือทุกจุดที่อาจ fail จะเห็นชัดเจนจาก signature ของฟังก์ชัน (`func Foo() (T, error)`) ผู้อ่านโค้ดรู้ทันทีว่าต้องเช็ค error ตรงไหน แต่ panic ไม่ปรากฏใน signature เลย — ฟังก์ชันไหนจะ panic เมื่อไหร่ ผู้อ่านโค้ดต้องเปิดอ่าน implementation ทั้งหมดถึงจะรู้
2. **Panic ที่ recover ผิดที่ผิดทาง อาจซ่อนบั๊กจริงไว้** ถ้า recover ทุกอย่างแบบไม่เลือก อาจทำให้โปรแกรมทำงานต่อไปในสถานะที่เสียหายไปแล้ว (corrupted state) โดยไม่รู้ตัว ซึ่งอันตรายกว่าการปล่อยให้ crash เสียอีก
3. **Performance**: การ panic/recover มีต้นทุนสูงกว่าการ return error ธรรมดาอย่างมีนัยสำคัญ (ต้อง unwind stack, เก็บ stack trace) ไม่เหมาะกับ control flow ที่ทำงานถี่

### เมื่อไหร่ที่ใช้ panic ได้อย่างเหมาะสม

Panic (โดยเฉพาะ explicit `panic()`) ควรสงวนไว้สำหรับสถานการณ์ต่อไปนี้เท่านั้น:

**1. Programmer error ที่ไม่ควรเกิดขึ้นเลยถ้าโค้ดถูกต้อง** — เช่นการเรียกฟังก์ชันด้วย argument ที่ผิด invariant ที่รับประกันไว้แล้วตั้งแต่การออกแบบ:

```go
func NewRateLimiter(maxPerSecond int) *RateLimiter {
	if maxPerSecond <= 0 {
		panic("NewRateLimiter: maxPerSecond ต้องมากกว่า 0")
	}
	return &RateLimiter{max: maxPerSecond}
}
```

ในกรณีนี้ `maxPerSecond <= 0` ไม่ใช่ error ที่ผู้ใช้ปลายทางควรเจอตอน runtime แต่เป็นบั๊กของโปรแกรมเมอร์ที่เรียกใช้ผิด ควรถูกจับได้ทันทีตอน develop/test ไม่ใช่ปล่อยให้ silent fail

**2. ความผิดพลาดที่ไม่มีทางกู้คืนได้อย่างมีความหมาย (unrecoverable)** — เช่น โปรแกรมเริ่มทำงานไม่ได้เพราะไฟล์ config จำเป็นหายไป (มักใช้คู่กับ `log.Fatal` มากกว่า panic ตรงๆ):

```go
func init() {
	if _, err := os.Stat("/etc/myapp/required.conf"); err != nil {
		panic("ไม่พบไฟล์ config ที่จำเป็น โปรแกรมไม่สามารถเริ่มทำงานได้")
	}
}
```

**3. Invariant ภายใน package ที่ควรเป็นจริงเสมอ (assertion)** — เช่นใน switch statement ที่ครบทุกกรณีตาม type แล้วในทางทฤษฎี ไม่ควรมี default case แต่ใส่ไว้เพื่อดักบั๊กในอนาคต:

```go
switch status {
case StatusActive, StatusInactive, StatusPending:
	// จัดการตามปกติ
default:
	panic(fmt.Sprintf("สถานะที่ไม่รู้จัก: %v (นี่คือบั๊ก ไม่ควรเกิดขึ้น)", status))
}
```

### กฎง่ายๆ ในการตัดสินใจ

> **ถ้าสถานการณ์นั้นเป็นสิ่งที่ caller "ควรคาดหวังได้" ว่าอาจเกิดขึ้น (ไฟล์ไม่มี, network ล่ม, input ผิด) ให้ return `error`**
>
> **ถ้าสถานการณ์นั้นคือ "ถ้าเกิดขึ้นแปลว่ามีบั๊กในโค้ดแน่นอน" (invariant ถูกละเมิด, argument ผิดกฎที่ตกลงกันไว้ตั้งแต่ design) ให้ใช้ `panic`**

standard library ของ Go เองก็ยึดกฎนี้: `strconv.Atoi` return error เพราะ input ที่ parse ไม่ได้เป็นเรื่องปกติที่คาดเดาได้ ในขณะที่ `regexp.MustCompile` panic ทันทีถ้า pattern ผิด เพราะ pattern มักถูก hardcode ไว้ในโค้ด — ถ้าผิดก็คือบั๊กที่ควรถูกจับตั้งแต่ตอน develop ไม่ใช่ตอน production

---

## 7. Panic ใน goroutine: ทำไมถึงทำทั้งโปรแกรมล่มได้

เรื่องนี้สำคัญมากและเป็นกับดักที่นักพัฒนา Go มือใหม่เจอบ่อยที่สุดเรื่องหนึ่ง (เราจะเรียน goroutine อย่างเต็มรูปแบบใน Part 036 แต่ต้องรู้ประเด็นนี้ไว้ก่อนเพราะเกี่ยวข้องโดยตรงกับ panic/recover):

> **ถ้า panic เกิดขึ้นใน goroutine ใดๆ และไม่มีการ `recover()` ภายใน goroutine นั้นเอง panic จะทำให้ "ทั้งโปรแกรม" (ทุก goroutine ที่กำลังทำงานอยู่) หยุดทำงานทันที ไม่ใช่แค่ goroutine นั้นตัวเดียว**

นี่ต่างจากภาษาอื่นที่มักมี thread แยกกันโดยที่ exception ใน thread หนึ่งไม่กระทบ thread อื่น ใน Go **`recover()` ใช้ได้เฉพาะภายใน goroutine เดียวกับที่ panic เกิดขึ้นเท่านั้น** goroutine อื่น (รวมถึง `main`) ไม่มีทาง recover panic ที่เกิดในอีก goroutine หนึ่งได้เลย

### สาธิต: panic ใน goroutine ทำให้ทั้งโปรแกรม crash

```go
package main

import (
	"fmt"
	"time"
)

func riskyWorker() {
	fmt.Println("goroutine เริ่มทำงาน")
	panic("เกิดข้อผิดพลาดร้ายแรงใน goroutine")
}

func main() {
	fmt.Println("main เริ่มทำงาน")
	go riskyWorker()

	// รอสักครู่เพื่อให้ goroutine ทำงาน (ในโค้ดจริงไม่ควรใช้ sleep แบบนี้)
	time.Sleep(200 * time.Millisecond)

	// บรรทัดนี้จะไม่มีทางถูกพิมพ์ เพราะ panic ใน goroutine
	// ที่ไม่มี recover จะทำให้ "ทั้งโปรแกรม" crash ทันที ไม่ใช่แค่ goroutine นั้น
	fmt.Println("main จบการทำงานแบบปกติ")
}
```

เมื่อรันโปรแกรมนี้ด้วย `go run` จะได้ผลลัพธ์ประมาณนี้ พร้อม **exit status ที่ไม่ใช่ 0**:

```
main เริ่มทำงาน
goroutine เริ่มทำงาน
panic: เกิดข้อผิดพลาดร้ายแรงใน goroutine

goroutine 6 [running]:
main.riskyWorker()
	/path/to/file.go:10 +0x59
created by main.main in goroutine 1
	/path/to/file.go:15 +0x56
exit status 2
```

สังเกตว่า **บรรทัดสุดท้าย `"main จบการทำงานแบบปกติ"` ไม่ถูกพิมพ์เลย** ทั้งที่อยู่ใน `main()` เอง — เพราะ panic จาก goroutine ที่แยกออกไปทำให้ runtime สั่งจบโปรแกรมทันทีทั้งกระบวนการ (process)

### วิธีป้องกัน: recover ในทุก goroutine ที่เราสร้างเอง

กฎเหล็กในการเขียนโปรแกรม Go ที่ใช้ goroutine คือ **ทุก goroutine ที่รับผิดชอบงานที่อาจ panic ได้ ควรมี `defer`+`recover()` ของตัวเองเสมอ** ไม่มีใครมา recover แทนได้:

```go
func safeGo(fn func()) {
	go func() {
		defer func() {
			if r := recover(); r != nil {
				fmt.Println("goroutine recovered:", r)
			}
		}()
		fn()
	}()
}

func main() {
	fmt.Println("main เริ่มทำงาน")
	safeGo(riskyWorker)
	time.Sleep(200 * time.Millisecond)
	fmt.Println("main จบการทำงานแบบปกติ") // ตอนนี้จะถูกพิมพ์ เพราะ panic ถูก recover แล้ว
}
```

รูปแบบ `safeGo` แบบนี้เป็นที่นิยมมากในระบบ production เพื่อป้องกันไม่ให้ goroutine ตัวใดตัวหนึ่งที่ error โดยไม่คาดคิด ทำให้ทั้งระบบล่มตามไปด้วย เราจะกลับมาเจอรูปแบบนี้อีกครั้งแบบเจาะลึกใน **Part 036 (Goroutines พื้นฐาน)** และ **Part 041 (Worker Pool Pattern)**

---

## 8. ตัวอย่างจริง: recover ใน HTTP middleware

หนึ่งในการใช้งาน panic/recover ที่พบบ่อยที่สุดในโค้ด production คือใน **HTTP server** — ถ้า handler ใดๆ เกิด panic ขึ้นมา (เช่นเพราะ bug ที่ไม่คาดคิด, nil pointer, index out of range) เราไม่อยากให้ panic นั้นทำให้**ทั้ง server ล่ม** จนทุก request ที่กำลังทำงานอยู่หยุดชะงักไปด้วย เราจึงเขียน **middleware** (รูปแบบที่จะเรียนแบบเต็มใน Part 057) ที่ดัก panic ของแต่ละ request แยกจากกัน แล้วตอบกลับเป็น HTTP 500 แทนที่จะปล่อยให้ล่ม

```go
package main

import (
	"fmt"
	"net/http"
	"net/http/httptest"
)

// recoverMiddleware คือ middleware ที่ดัก panic จาก handler ใดๆ
// ไม่ให้ทั้ง HTTP server ล่ม (เราจะเรียนเรื่อง middleware แบบเต็มรูปแบบใน Part 057)
func recoverMiddleware(next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		defer func() {
			if rec := recover(); rec != nil {
				fmt.Println("[middleware] recovered from panic:", rec)
				w.WriteHeader(http.StatusInternalServerError)
				w.Write([]byte("internal server error"))
			}
		}()
		next.ServeHTTP(w, r)
	})
}

func riskyHandler(w http.ResponseWriter, r *http.Request) {
	panic("something went wrong inside the handler")
}

func okHandler(w http.ResponseWriter, r *http.Request) {
	w.Write([]byte("ok"))
}

func main() {
	mux := http.NewServeMux()
	mux.HandleFunc("/risky", riskyHandler)
	mux.HandleFunc("/ok", okHandler)

	server := httptest.NewServer(recoverMiddleware(mux))
	defer server.Close()

	resp, err := http.Get(server.URL + "/risky")
	if err != nil {
		fmt.Println("request error:", err)
		return
	}
	fmt.Println("GET /risky -> status:", resp.StatusCode)
	resp.Body.Close()

	resp2, err := http.Get(server.URL + "/ok")
	if err != nil {
		fmt.Println("request error:", err)
		return
	}
	fmt.Println("GET /ok -> status:", resp2.StatusCode)
	resp2.Body.Close()

	fmt.Println("server ยังทำงานอยู่ปกติ ไม่ crash แม้ handler จะ panic")
}
```

ผลลัพธ์:

```
[middleware] recovered from panic: something went wrong inside the handler
GET /risky -> status: 500
GET /ok -> status: 200
server ยังทำงานอยู่ปกติ ไม่ crash แม้ handler จะ panic
```

ตัวอย่างนี้ใช้ `net/http/httptest` เพื่อสร้าง HTTP server จำลองที่รันได้จริงในโปรแกรมเดียว (ไม่ต้องเปิด port ค้างไว้หรือรันแยก process) เหมาะสำหรับทดลองและทดสอบโค้ด — เราจะเรียน `httptest` แบบเต็มรูปแบบใน **Part 081**

จุดสำคัญที่ต้องสังเกต:

1. **แต่ละ HTTP request ใน Go ทำงานบน goroutine แยกกัน** (สร้างโดย `net/http` โดยอัตโนมัติ ไม่ใช่สิ่งที่เราต้องเขียนเอง) ตามหลักการในหัวข้อ 7 นั่นแปลว่า **ถ้าไม่มี middleware นี้ panic ใน `riskyHandler` จะทำให้ทั้ง server process ล่ม** ส่งผลกระทบต่อ request อื่นๆ ที่กำลังทำงานอยู่พร้อมกันด้วย
2. Middleware pattern คือการ "ห่อ" handler เดิมด้วยฟังก์ชันอีกชั้นหนึ่งที่ทำงานพิเศษก่อน/หลังเรียก handler จริง — เป็นรูปแบบที่ใช้กันแพร่หลายมากใน web framework ของ Go แทบทุกตัว
3. หลังจาก `/risky` panic และถูก recover แล้ว server **ยังคงทำงานต่อได้ตามปกติ** พิสูจน์ได้จากการที่ request ไปยัง `/ok` หลังจากนั้นยังได้ผลลัพธ์ถูกต้อง

รูปแบบ recover-in-middleware แบบนี้เป็นมาตรฐานที่แทบทุก web framework ของ Go (Gin, Echo, Fiber ที่จะเรียนใน Part 061–064) มีให้ในตัวอยู่แล้วเป็นค่า default โดยไม่ต้องเขียนเอง แต่การเข้าใจว่ามันทำงานอย่างไรภายในจะช่วยให้ debug ปัญหาได้ลึกซึ้งกว่ามาก

---

## 9. แนวทางปฏิบัติและข้อผิดพลาดที่พบบ่อย

### ข้อผิดพลาดที่ 1: ใช้ panic/recover แทน error สำหรับ control flow ปกติ

```go
// ผิดหลักการ Go โดยสิ้นเชิง
func findUser(id int) *User {
	user, ok := users[id]
	if !ok {
		panic("user not found")
	}
	return user
}

func main() {
	defer func() {
		if r := recover(); r != nil {
			fmt.Println("error:", r)
		}
	}()
	u := findUser(999)
	fmt.Println(u)
}
```

ควรเขียนแบบ return error ตามปกติแทน:

```go
func findUser(id int) (*User, error) {
	user, ok := users[id]
	if !ok {
		return nil, fmt.Errorf("findUser(%d): %w", id, ErrNotFound)
	}
	return user, nil
}
```

### ข้อผิดพลาดที่ 2: recover แล้วไม่ทำอะไรเลย (silent failure)

```go
defer func() {
	recover() // กลืน panic เงียบๆ ไม่ log ไม่ report อะไรเลย -- อันตรายมาก
}()
```

การ recover โดยไม่บันทึก log หรือแจ้งเตือนใดๆ เท่ากับ**ซ่อนบั๊กไว้ใต้พรม** ทำให้ปัญหาจริงไม่ถูกแก้ไข ควรอย่างน้อยที่สุด log ข้อความและ stack trace ไว้เสมอ (จะเรียนเรื่อง logging ที่ดีใน Part 054)

### ข้อผิดพลาดที่ 3: ลืมว่า `recover()` ต้องอยู่ใน deferred function โดยตรง

```go
func doRecover() interface{} {
	return recover() // ไม่มีผล! ฟังก์ชันนี้ไม่ได้ถูกเรียกผ่าน defer โดยตรง ณ จุดที่ panic เกิด
}

func risky() {
	defer doRecover() // การเรียกผ่านฟังก์ชันอื่นแบบนี้ใช้ไม่ได้
	panic("boom")
}
```

`recover()` ต้องถูกเรียก**โดยตรง**ภายใน function literal ที่ผ่าน `defer` เท่านั้น การซ่อนไว้หลังชั้นของฟังก์ชันอื่นแบบนี้จะทำให้ recover ไม่ทำงาน

### ข้อผิดพลาดที่ 4: ไม่ตรวจสอบว่า `recover()` คืนค่าอะไร ก่อนใช้งานต่อ

```go
defer func() {
	r := recover()
	fmt.Println("error:", r.(error)) // panic ถ้า recover() คืน nil (ไม่มี panic เกิดขึ้น) หรือคืนค่าที่ไม่ใช่ error
}()
```

ต้องเช็ค `if r != nil` ก่อนเสมอ และถ้าจะแปลงชนิด ควรใช้ comma-ok form ของ type assertion ตามที่เรียนใน Part 014 (`if err, ok := r.(error); ok { ... }`) แทนการ assert ตรงๆ

### แนวทางปฏิบัติที่ดี — สรุปสั้นๆ

- ใช้ `error` เป็นค่าเริ่มต้นเสมอสำหรับความผิดพลาดที่คาดการณ์ได้
- ใช้ `panic` เฉพาะ programmer error / invariant violation / สถานการณ์ที่กู้คืนไม่ได้จริงๆ
- ถ้าต้องเขียน library ที่เรียก third-party code ที่เสี่ยง panic ให้ recover แล้วแปลงเป็น error ที่ขอบเขต API เท่านั้น
- ทุก goroutine ที่สร้างขึ้นเองต้องมี `defer`+`recover()` ของตัวเอง ไม่มีใครมา recover แทนได้
- ทุกครั้งที่ recover ควร log ไว้เสมอ ไม่ควรกลืนความผิดพลาดแบบเงียบๆ

---

## สรุปสิ่งที่ได้เรียนในบทนี้

- **Panic** คือกลไกสำหรับความผิดพลาดร้ายแรงที่โปรแกรมไม่ควรทำงานต่อได้อย่างปลอดภัย ต่างจาก `error` ที่ใช้กับสถานการณ์ที่คาดการณ์ได้
- สาเหตุของ panic มีทั้งจาก runtime เอง (index out of range, nil pointer dereference, divide by zero, type assertion ผิด, เขียน nil map) และจาก explicit `panic()` ที่โปรแกรมเมอร์เรียกเอง
- เมื่อเกิด panic, stack จะ **unwind** ขึ้นไปเรื่อยๆ โดยรัน **deferred function ทุกตัว** ตามลำดับ LIFO ในแต่ละ stack frame ระหว่างทาง
- **`recover()`** หยุดการ unwind ได้ แต่ต้องถูกเรียกโดยตรงภายใน deferred function เท่านั้น เรียกที่อื่นจะคืนค่า `nil` เสมอและไม่มีผล
- แปลง panic เป็น `error` ที่ขอบเขตของ API ได้ด้วยการใช้ named return value ร่วมกับ `defer`+`recover()` — เหมาะสำหรับ library ที่ต้องพึ่งพา code ที่เสี่ยง panic
- **panic/recover ไม่ใช่ exception handling** ควรสงวนไว้สำหรับ programmer error หรือ invariant ที่ถูกละเมิด ไม่ใช่ control flow ปกติ
- **panic ใน goroutine ที่ไม่ถูก recover จะทำให้ "ทั้งโปรแกรม" ล่ม** ไม่ใช่แค่ goroutine นั้น — ทุก goroutine ที่สร้างเองต้องมี recover ของตัวเอง
- **HTTP middleware ที่ recover panic** เป็นรูปแบบมาตรฐานในการป้องกัน handler ตัวใดตัวหนึ่งที่ panic ไม่ให้ทำทั้ง server ล่ม (foreshadow Part 057)

## แบบฝึกหัดท้ายบท

1. เขียนโปรแกรมที่จงใจทำให้เกิด panic จากสาเหตุทั้ง 4 แบบ (index out of range, nil pointer dereference, integer divide by zero, type assertion ผิด) แยกเป็นฟังก์ชันละตัว แล้วดัก panic แต่ละตัวด้วย `recover()` พิมพ์ข้อความ error ที่ได้ออกมาเปรียบเทียบกัน
2. เขียนฟังก์ชัน `safeCall(fn func()) (err error)` ที่รับฟังก์ชันใดๆ มาเรียก แล้วถ้าฟังก์ชันนั้น panic ให้แปลงเป็น error คืนกลับมาแทนที่จะปล่อยให้โปรแกรม crash (ใช้รูปแบบ named return + defer + recover ตามหัวข้อ 5)
3. สร้างตัวอย่างที่แสดงลำดับการทำงานของ `defer` ระหว่าง stack unwinding อย่างน้อย 4 ชั้นฟังก์ชัน (คล้ายตัวอย่างในหัวข้อ 3) แล้ววาดลำดับ (comment หรือ diagram ง่ายๆ) ว่า deferred function แต่ละตัวถูกเรียกเมื่อไหร่
4. เขียนโปรแกรมที่สร้าง 3 goroutine พร้อมกัน โดย goroutine หนึ่งจงใจ panic โดยไม่มี recover — สังเกตว่าทั้งโปรแกรมล่มทันที จากนั้นแก้ไขให้ทุก goroutine มี `defer`+`recover()` ของตัวเอง แล้วพิสูจน์ว่าโปรแกรมทำงานต่อได้ครบทุก goroutine ที่เหลือ
5. ขยายตัวอย่าง HTTP middleware ในหัวข้อ 8 ให้ middleware log ทั้งข้อความ panic และเวลาที่เกิดขึ้น (ใช้ `time.Now()`) ก่อนตอบกลับ HTTP 500 แล้วทดสอบด้วย handler ที่ panic ด้วยสาเหตุต่างกัน (เช่น divide by zero กับ nil pointer)
6. อภิปราย (เขียนเป็น comment ในโค้ดหรือข้อความสั้นๆ): ถ้ามีฟังก์ชัน `Divide(a, b int) int` ที่ panic เมื่อ `b == 0` เทียบกับฟังก์ชัน `Divide(a, b int) (int, error)` ที่ return error เมื่อ `b == 0` — เลือกใช้แบบไหนดีกว่ากันในสถานการณ์ที่ `b` มาจาก input ของผู้ใช้ที่ควบคุมไม่ได้ เทียบกับสถานการณ์ที่ `b` เป็นค่าคงที่ที่ hardcode ไว้ในโค้ดเอง อธิบายเหตุผลโดยอ้างอิงกฎจากหัวข้อ 6

---

**ต่อไป**: [Part 018 — การจัดการ Package และ Module ขั้นสูง](./018-packages-and-modules-advanced.md)
