# Part 028: Generics พื้นฐาน (Go 1.18+)

> ภาคที่ 2: ระดับกลาง (Intermediate) — ตอนที่ 13 จาก 20 (Part 16–35)

## สารบัญของบทนี้

1. ปัญหาที่ Generics แก้: โค้ดซ้ำซ้อนข้าม type
2. ทางแก้แบบเดิมก่อน Go 1.18 และข้อจำกัดของมัน
3. Generics คืออะไร: Type Parameter ตัวแรกของคุณ
4. Syntax: `func Foo[T any](x T) T`
5. Type Constraints: `any` และ `comparable`
6. เขียนฟังก์ชัน `Max`/`Min` แบบ Generic
7. Type Instantiation และ Type Inference
8. Type Parameter หลายตัวในฟังก์ชันเดียว: เขียน `Map` แบบ Generic
9. Generic Type บน Struct: `type Stack[T any] struct`
10. ทำไม Generics ของ Go ถูกออกแบบให้จำกัดกว่า C++ Templates โดยตั้งใจ
11. สรุปสิ่งที่ได้เรียนในบทนี้
12. แบบฝึกหัดท้ายบท

---

## 1. ปัญหาที่ Generics แก้: โค้ดซ้ำซ้อนข้าม type

ลองจินตนาการว่าต้องเขียนฟังก์ชันหาค่ามากสุดระหว่างตัวเลขสองตัว ถ้าต้องรองรับทั้ง `int` และ `float64` ด้วย Go ก่อนเวอร์ชัน 1.18 (เปิดตัวปี 2022) เราต้อง**เขียนฟังก์ชันแยกกันสำหรับแต่ละ type** เพราะ Go เป็นภาษา static type ที่ไม่ยอมให้ฟังก์ชันเดียวรับหลาย type ที่ไม่เกี่ยวข้องกันได้โดยตรง:

```go
func MaxInt(a, b int) int {
	if a > b {
		return a
	}
	return b
}

func MaxFloat64(a, b float64) float64 {
	if a > b {
		return a
	}
	return b
}
```

logic ข้างในทั้งสองฟังก์ชัน**เหมือนกันทุกตัวอักษร** ต่างกันแค่ type ของ parameter เท่านั้น ถ้าต้องรองรับ `int32`, `int64`, `uint`, `string` เพิ่มอีก ก็ต้อง copy-paste logic เดิมซ้ำไปเรื่อยๆ — นี่คือปัญหาคลาสสิกที่เรียกว่า **code duplication across types** ซึ่งขัดกับปรัชญา "less is more" ที่เรียนใน **Part 001** โดยตรง เพราะยิ่งโค้ดซ้ำมาก ยิ่งแก้ bug ยาก (ต้องแก้ทุกที่ที่ copy ไป) และยิ่ง maintain ยากขึ้นแบบทวีคูณ

---

## 2. ทางแก้แบบเดิมก่อน Go 1.18 และข้อจำกัดของมัน

ก่อนที่ Go จะมี generics นักพัฒนาแก้ปัญหานี้ด้วยสองวิธีหลัก ซึ่งทั้งคู่มีข้อเสียชัดเจน:

### วิธีที่ 1: ใช้ `interface{}` (หรือ `any`) + Type Assertion

```go
func MaxAny(a, b interface{}) interface{} {
	switch x := a.(type) {
	case int:
		y := b.(int)
		if x > y {
			return x
		}
		return y
	case float64:
		y := b.(float64)
		if x > y {
			return x
		}
		return y
	default:
		panic("unsupported type")
	}
}
```

วิธีนี้ (ทบทวน `interface{}`/`any` และ type switch จาก **Part 014**) ใช้งานได้จริง แต่มีข้อเสียร้ายแรง:

- **เสีย type safety ตอน compile time** — ถ้าเรียก `MaxAny(3, "hello")` โค้ดจะ compile ผ่านสบายๆ แต่ไป **panic ตอน runtime** เพราะ type assertion ล้มเหลว ซึ่งขัดกับจุดแข็งสำคัญที่สุดของ Go คือการจับ error ตั้งแต่ compile time
- **ต้องเขียน `case` ใหม่ทุกครั้งที่เพิ่ม type ที่รองรับ** — โค้ดยังคงซ้ำซ้อนอยู่ดี เพียงแค่ย้ายมาซ้ำใน `switch` แทน
- **เสีย performance** — การใช้ `interface{}` ต้องมีการ "boxing" ค่าเข้า interface (จะอธิบายเพิ่มใน **Part 029**) ซึ่งมี overhead มากกว่าการทำงานกับ concrete type ตรงๆ

### วิธีที่ 2: Code Generation

อีกวิธีที่ทีมใหญ่ๆ ใช้กันคือเขียนเครื่องมือ **generate โค้ด** สำหรับแต่ละ type โดยอัตโนมัติ (เช่นเครื่องมือชื่อ `genny` หรือเขียน script เอง) วิธีนี้รักษา type safety ไว้ได้เพราะโค้ดที่ generate ออกมาเป็น concrete type จริงๆ แต่มีข้อเสียคือ **เพิ่มความซับซ้อนของ build pipeline** ต้องรัน generator ก่อน compile ทุกครั้ง และโค้ดที่ generate ออกมาก็ยังคง "ซ้ำ" อยู่ในไฟล์จริง (แค่ไม่ต้องพิมพ์เองด้วยมือ)

ทั้งสองวิธีนี้คือเหตุผลที่ทีม Go ใช้เวลาคิดออกแบบ generics นานถึง 10 ปีก่อนจะเปิดตัวใน **Go 1.18 (ปี 2022)** ตามที่กล่าวถึงใน **Part 001** — ต้องหา syntax ที่เรียบง่ายพอจะไม่ทำลายปรัชญาของภาษา แต่ทรงพลังพอจะแก้ปัญหานี้ได้จริง

---

## 3. Generics คืออะไร: Type Parameter ตัวแรกของคุณ

**Generics** คือความสามารถในการเขียนฟังก์ชันหรือ type ที่ทำงานกับ**หลาย type ได้โดยยังคง type safety ไว้ครบถ้วนตอน compile time** กลไกหลักที่ทำให้เป็นไปได้คือ **type parameter** — คล้ายกับที่ function parameter ทั่วไปรับ**ค่า**เป็น argument, type parameter จะรับ**ชนิดข้อมูล**เป็น argument แทน

```go
func Max[T int | float64 | string](a, b T) T {
	if a > b {
		return a
	}
	return b
}
```

เมื่อเรียก `Max(3, 7)` compiler จะรู้ว่า `T` คือ `int` เพราะ argument ที่ส่งเข้ามาเป็น `int` และจะ**ตรวจสอบ type ให้ครบถ้วนตอน compile time เหมือนฟังก์ชันธรรมดาทุกประการ** — ถ้าเรียก `Max(3, "hello")` จะได้ **compile error ทันที** ไม่ต้องรอไป panic ตอน runtime แบบวิธี `interface{}` ในหัวข้อก่อนหน้า

---

## 4. Syntax: `func Foo[T any](x T) T`

Syntax ของ generic function เพิ่มส่วนใหม่เข้ามาหนึ่งส่วนคือ **type parameter list** ที่อยู่ในเครื่องหมาย `[...]` ต่อจากชื่อฟังก์ชันทันที ก่อนถึง parameter list ปกติ:

```go
func Foo[T any](x T) T {
	return x
}
```

อ่านได้ดังนี้:

- `[T any]` คือ **type parameter list** — ประกาศว่าฟังก์ชันนี้มี type parameter ชื่อ `T` และ `T` ต้องเป็น type ที่ตรงกับ constraint `any` (จะอธิบายในหัวข้อถัดไป)
- `(x T)` คือ parameter list ปกติ แต่ตอนนี้ `x` มี type เป็น `T` แทนที่จะเป็น type ตายตัวแบบ `int` หรือ `string`
- `T` (return type) หมายความว่าฟังก์ชันนี้คืนค่าเป็น type เดียวกับที่ `x` เป็น

ชื่อ `T` ไม่ใช่ keyword พิเศษ — เป็นแค่ชื่อตัวแปร type ที่ตั้งเองได้ (นิยมใช้ตัวอักษรเดี่ยวตัวใหญ่ เช่น `T`, `K`, `V`, `E` ตามธรรมเนียมที่มาจากภาษาอื่นๆ ที่มี generics เช่น Java, C#) สามารถมี type parameter หลายตัวในฟังก์ชันเดียวได้ เช่น `func Map[T, U any](s []T, f func(T) U) []U`

---

## 5. Type Constraints: `any` และ `comparable`

**Type constraint** คือการกำหนดว่า type parameter ยอมรับ type ไหนได้บ้าง เขียนอยู่ในตำแหน่งที่ปกติภาษาอื่นเรียก "extends" หรือ "bound" — ใน Go เขียนต่อจากชื่อ type parameter ทันที (`[T any]`, `[T comparable]`)

Go มี constraint สำเร็จรูปสองตัวที่ใช้บ่อยที่สุด:

| Constraint | ความหมาย |
|---|---|
| `any` | ยอมรับ**ทุก type** ไม่มีข้อจำกัดเลย เป็นแค่ alias ของ `interface{}` (เรียนไปแล้วใน **Part 014** และ **Part 025**) เมื่อใช้เป็น constraint |
| `comparable` | ยอมรับเฉพาะ type ที่ใช้ operator `==` และ `!=` เปรียบเทียบกันได้ (ตัวเลข, string, bool, pointer, struct ที่ field ทุกตัว comparable, array ที่ element comparable) — **ไม่รวม** slice, map, function เพราะ type เหล่านี้เปรียบเทียบด้วย `==` ไม่ได้ (ทบทวนกฎนี้จาก **Part 006** และ **Part 007**) |

```go
package main

import "fmt"

// Print ใช้ constraint any เพราะแค่รับค่ามาพิมพ์ ไม่ต้องเปรียบเทียบอะไรเลย
func Print[T any](v T) {
	fmt.Printf("value=%v type=%T\n", v, v)
}

// Contains ใช้ comparable เพราะต้องใช้ == เปรียบเทียบ element กับ target
func Contains[T comparable](slice []T, target T) bool {
	for _, v := range slice {
		if v == target {
			return true
		}
	}
	return false
}

func main() {
	Print(42)
	Print("hello")

	fmt.Println(Contains([]int{1, 2, 3}, 2))
	fmt.Println(Contains([]string{"a", "b"}, "z"))
}
```

ผลลัพธ์:

```
value=42 type=int
value=hello type=string
true
false
```

หลักการเลือก constraint: **ถ้าฟังก์ชันแค่รับค่าเข้ามาแล้วส่งต่อ/พิมพ์/เก็บ โดยไม่เปรียบเทียบหรือคำนวณอะไร ใช้ `any` พอ** แต่ถ้าต้องใช้ operator ใดๆ กับค่านั้น (`==`, `<`, `+` เป็นต้น) ต้องเลือก constraint ที่รองรับ operator นั้นโดยเฉพาะ — `comparable` สำหรับ `==`/`!=` และ constraint กำหนดเอง (จะเรียนใน **Part 029**) สำหรับ operator อื่น เช่น `<` หรือ `+`

---

## 6. เขียนฟังก์ชัน `Max`/`Min` แบบ Generic

มาดูตัวอย่างที่ครบถ้วนที่รวมทุกอย่างที่เรียนมา — ฟังก์ชัน `Max`/`Min` ที่ใช้ constraint แบบ **union of types** (การระบุ type ที่ยอมรับได้หลายตัวคั่นด้วย `|` ซึ่งจะอธิบายเจาะลึกเรื่องการนิยาม constraint เองใน **Part 029**):

```go
package main

import "fmt"

func Max[T int | float64 | string](a, b T) T {
	if a > b {
		return a
	}
	return b
}

func Min[T int | float64 | string](a, b T) T {
	if a < b {
		return a
	}
	return b
}

func main() {
	fmt.Println(Max(3, 7))              // T ถูก infer เป็น int อัตโนมัติ
	fmt.Println(Max(3.5, 1.2))          // T = float64
	fmt.Println(Max("banana", "apple")) // T = string

	fmt.Println(Min(3, 7))

	// ระบุ type argument ตรงๆ ก็ได้ (ปกติไม่จำเป็นเพราะ compiler infer ให้)
	fmt.Println(Max[int](10, 20))
}
```

ผลลัพธ์:

```
7
3.5
banana
3
20
```

สังเกตว่า `Max`/`Min` ใช้ operator `>` และ `<` ได้โดยตรงกับ `T` เพราะ constraint `int | float64 | string` จำกัดให้ `T` เป็นได้แค่สาม type นี้เท่านั้น ซึ่งทั้งสามรองรับการเปรียบเทียบด้วย `>`/`<` — ถ้าลองเขียน `func Max[T any](a, b T) T` แล้วใช้ `a > b` ข้างใน จะได้ **compile error ทันที** เพราะ `any` ไม่รับประกันว่า `T` จะรองรับ operator `>` เลย (บาง type เช่น struct หรือ slice ไม่รองรับ `>`)

> **หมายเหตุ**: ตั้งแต่ Go 1.21 เป็นต้นมา ภาษามีฟังก์ชัน built-in ชื่อ `min`/`max` ให้ใช้ได้ทันทีโดยไม่ต้อง import อะไรเลย (เช่น `max(3, 7)`) ซึ่งทำงานคล้ายกับตัวอย่างข้างบนนี้ในเชิงแนวคิด แต่ตัวอย่างนี้ยังมีประโยชน์มากในการทำความเข้าใจว่า generics ทำงานอย่างไรเบื้องหลัง และเป็นพื้นฐานสำหรับเขียนฟังก์ชัน generic อื่นๆ ที่ built-in ไม่มีให้

---

## 7. Type Instantiation และ Type Inference

การนำ generic function/type ไปใช้งานจริงเรียกว่า **instantiation** — คือการระบุ type argument ที่ชัดเจนให้กับ type parameter ทำได้สองแบบ:

### แบบระบุชัดเจน (Explicit Instantiation)

```go
Max[int](10, 20)
```

ระบุ `[int]` ต่อท้ายชื่อฟังก์ชันตรงๆ บอก compiler ชัดเจนว่า `T` คือ `int`

### แบบให้ Compiler อนุมานเอง (Type Inference)

```go
Max(10, 20)
```

ในกรณีส่วนใหญ่ Go compiler **อนุมาน (infer)** type argument ได้เองจาก argument ที่ส่งเข้ามา โดยไม่ต้องระบุ `[int]` เลย เพราะเห็นว่า `10` และ `20` เป็น `int` compiler จึงรู้ทันทีว่า `T = int`

Type inference ทำงานได้ในกรณีทั่วไปส่วนใหญ่ แต่มีบางสถานการณ์ที่ compiler อนุมานไม่ได้ (เช่น type parameter ที่ไม่ปรากฏใน parameter ใดๆ เลย มีแต่ใน return type) ซึ่งกรณีนั้นต้องระบุแบบ explicit เสมอ

```go
package main

import "fmt"

func Zero[T any]() T {
	var zero T
	return zero
}

func main() {
	// Zero() ไม่มี argument ให้ compiler อนุมาน T ได้เลย ต้องระบุเอง
	fmt.Println(Zero[int]())
	fmt.Println(Zero[string]())
}
```

**หลักปฏิบัติ**: เขียนโค้ดแบบให้ compiler infer เองเป็นค่าเริ่มต้นเสมอ (สั้นและอ่านง่ายกว่า) แล้วค่อยระบุ type argument แบบ explicit เฉพาะตอนที่ compiler แจ้ง error ว่าอนุมานไม่ได้เท่านั้น

---

## 8. Type Parameter หลายตัวในฟังก์ชันเดียว: เขียน `Map` แบบ Generic

ฟังก์ชัน `Max`/`Min` ในหัวข้อก่อนใช้ type parameter ตัวเดียว (`T`) ที่ทั้ง argument และ return value เป็น type เดียวกัน แต่ generics ของ Go รองรับ **type parameter หลายตัวในฟังก์ชันเดียวกัน** ได้ตามธรรมชาติ ซึ่งเปิดทางให้เขียนฟังก์ชันที่ "แปลง type" ระหว่างทางได้ เช่นฟังก์ชัน `Map` ที่คุ้นเคยกันดีจากภาษาที่รองรับ functional programming:

```go
package main

import "fmt"

// Map มี type parameter สองตัว: T (type ต้นทาง) และ U (type ปลายทาง)
// แสดงให้เห็นว่า type parameter ไม่จำเป็นต้องเป็นตัวเดียวกันทั้งฟังก์ชัน
func Map[T, U any](s []T, f func(T) U) []U {
	result := make([]U, len(s))
	for i, v := range s {
		result[i] = f(v)
	}
	return result
}

func main() {
	nums := []int{1, 2, 3, 4}

	// T = int, U = int (แปลงเลขเป็นเลขยกกำลังสอง)
	squared := Map(nums, func(n int) int {
		return n * n
	})
	fmt.Println(squared)

	// T = int, U = string (แปลงเลขเป็นข้อความ)
	labels := Map(nums, func(n int) string {
		return fmt.Sprintf("เลข-%d", n)
	})
	fmt.Println(labels)
}
```

ผลลัพธ์:

```
[1 4 9 16]
[เลข-1 เลข-2 เลข-3 เลข-4]
```

จุดที่ควรสังเกต:

- `[T, U any]` ประกาศ type parameter สองตัวที่ใช้ constraint เดียวกัน (`any`) คั่นด้วย comma — ถ้า `T` กับ `U` ต้องการ constraint ต่างกัน ก็เขียนแยกเป็น `[T any, U comparable]` ได้ตามปกติ
- Compiler อนุมาน (infer) ทั้ง `T` และ `U` ให้เองจาก argument ที่ส่งเข้ามา: `T` มาจาก type ของ element ใน `nums` ([]int) ส่วน `U` มาจาก **return type ของ closure** ที่ส่งเป็น argument ที่สอง — เป็นตัวอย่างที่ดีของ type inference ที่ไม่ได้ดูแค่ parameter ตัวแรกเท่านั้น แต่ดูจาก parameter อื่นที่เกี่ยวข้องกันด้วย
- แพทเทิร์นนี้คือหัวใจของฟังก์ชัน `Map`/`Filter`/`Reduce` ที่เป็นที่นิยมในภาษาที่เน้น functional programming — ตั้งแต่ Go 1.21 เป็นต้นมา standard library มี package ชื่อ `slices` ที่มีฟังก์ชันลักษณะนี้ให้ใช้สำเร็จรูปอยู่แล้ว (เช่น `slices.Sort`, `slices.Contains`) แต่การเข้าใจว่าฟังก์ชันเหล่านี้เขียนขึ้นจากหลักการ generics แบบที่เรียนในบทนี้ ช่วยให้อ่านและต่อยอด source code ของ package เหล่านั้นได้อย่างมั่นใจ

---

## 9. Generic Type บน Struct: `type Stack[T any] struct`

Generics ใช้ได้ไม่เฉพาะกับฟังก์ชัน แต่ใช้กับ **type declaration** ได้ด้วย ทำให้สร้าง data structure ที่ใช้ได้กับหลาย type ได้โดยไม่ต้องเขียนซ้ำ (เช่น `IntStack`, `StringStack` แยกกัน):

```go
package main

import "fmt"

// Stack[T] คือ generic struct - เก็บข้อมูลชนิดใดก็ได้ที่กำหนดตอนใช้งาน
type Stack[T any] struct {
	items []T
}

func (s *Stack[T]) Push(v T) {
	s.items = append(s.items, v)
}

func (s *Stack[T]) Pop() (T, bool) {
	var zero T
	if len(s.items) == 0 {
		return zero, false
	}
	last := s.items[len(s.items)-1]
	s.items = s.items[:len(s.items)-1]
	return last, true
}

func (s *Stack[T]) Len() int {
	return len(s.items)
}

func main() {
	// instantiation แบบระบุ type ชัดเจน
	var intStack Stack[int]
	intStack.Push(1)
	intStack.Push(2)
	intStack.Push(3)

	for intStack.Len() > 0 {
		v, _ := intStack.Pop()
		fmt.Println("popped:", v)
	}

	// ใช้กับ string ได้เหมือนกันโดยไม่ต้องเขียน StringStack แยก
	stringStack := Stack[string]{}
	stringStack.Push("go")
	stringStack.Push("rust")
	v, ok := stringStack.Pop()
	fmt.Println(v, ok)
}
```

ผลลัพธ์:

```
popped: 3
popped: 2
popped: 1
rust true
```

จุดสำคัญที่ต้องสังเกต:

- ประกาศ type parameter ต่อท้ายชื่อ type ทันที: `type Stack[T any] struct { items []T }`
- เวลาประกาศ **method** ของ generic type ต้องระบุ type parameter ซ้ำใน receiver เสมอ: `func (s *Stack[T]) Push(v T)` — สังเกตว่า `[T]` ต้องอยู่หลังชื่อ type ใน receiver แต่**ไม่ต้องมี constraint ซ้ำ** (ไม่ต้องเขียน `[T any]` ในบรรทัด method เพราะ constraint ถูกกำหนดไว้แล้วตอนประกาศ type)
- `var zero T` ในฟังก์ชัน `Pop` คือการสร้าง **zero value ของ type parameter** (ทบทวนแนวคิด zero value จาก **Part 003**) ใช้เป็นค่า default เมื่อ stack ว่างเปล่า ไม่ว่า `T` จะเป็น type อะไรก็ตาม (`0` สำหรับตัวเลข, `""` สำหรับ string, `nil` สำหรับ pointer เป็นต้น)
- การ instantiate ทำได้สองแบบ: ประกาศตัวแปรแบบ `var intStack Stack[int]` หรือสร้างแบบ composite literal `Stack[string]{}` — ทั้งสองแบบใช้ได้เหมือน struct ธรรมดาทุกประการ

---

## 10. ทำไม Generics ของ Go ถูกออกแบบให้จำกัดกว่า C++ Templates โดยตั้งใจ

หลายคนที่มีพื้นฐาน C++ มาก่อนอาจคุ้นเคยกับ **templates** ซึ่งเป็นกลไก generics ที่ทรงพลังมาก แต่ Go เลือก**จำกัดความสามารถของ generics โดยตั้งใจ** ด้วยเหตุผลที่สอดคล้องกับปรัชญา "less is more" จาก **Part 001**:

| ประเด็น | C++ Templates | Go Generics |
|---|---|---|
| การตรวจสอบ constraint | ตรวจสอบตอน**ใช้งาน template** (instantiation time) ทำให้ error message มักยาวและอ่านยากมาก (เรียกว่า "template error hell") | ตรวจสอบตอน**ประกาศฟังก์ชัน generic เอง** ผ่าน constraint ที่ชัดเจน ทำให้ error message สั้นและชี้จุดผิดตรงๆ |
| Specialization | รองรับ template specialization ที่ซับซ้อนมาก (เขียน logic ต่างกันสำหรับแต่ละ type ได้ในระดับ compile time) | ไม่รองรับ — โค้ดใน generic function/type ต้องเป็น logic เดียวกันสำหรับทุก type ที่ constraint อนุญาต |
| Metaprogramming | ใช้ template ทำ compile-time computation ที่ซับซ้อนได้ (template metaprogramming) จนเกือบเป็นภาษาโปรแกรมมิ่งแยกต่างหาก | ไม่มีแนวคิดนี้เลย — Go ตั้งใจไม่ให้ generics กลายเป็นเครื่องมือทำ metaprogramming |
| Compile time | Template ขยายโค้ดซ้ำสำหรับทุก type ที่ใช้ ทำให้ compile ช้าลงมากในโปรเจกต์ใหญ่ | ออกแบบให้กระทบเวลา compile น้อยที่สุดเท่าที่จะทำได้ (ดู "compile เร็ว" เป็นจุดเด่นจาก **Part 001**) |
| Syntax | ซับซ้อน มี syntax พิเศษจำนวนมาก (`template<typename T>`, SFINAE, concepts) | เรียบง่าย มี syntax เดียว (`[T constraint]`) ใช้ได้ทั้งฟังก์ชันและ type |

หัวใจของการออกแบบคือ: **Go ต้องการให้ generics แก้ปัญหา "โค้ดซ้ำซ้อนข้าม type" ให้ได้ดีที่สุด โดยไม่แลกมาด้วยความซับซ้อนของภาษาที่เพิ่มขึ้นแบบที่ C++ เจอ** นี่คือเหตุผลที่ต้องใช้เวลาคิด 10 ปีก่อนเปิดตัว — เพื่อหาจุดสมดุลระหว่างพลังของ generics กับความเรียบง่ายที่เป็นเอกลักษณ์ของภาษา Go

ในทางปฏิบัติ ข้อจำกัดเหล่านี้แทบไม่กระทบการใช้งานจริงเลย เพราะ 90% ของปัญหาที่ต้องการ generics (data structure ทั่วไป, ฟังก์ชันเปรียบเทียบ/แปลงข้อมูล) ไม่ต้องการความสามารถระดับ template metaprogramming ของ C++ อยู่แล้ว

---

## สรุปสิ่งที่ได้เรียนในบทนี้

- Generics แก้ปัญหาโค้ดซ้ำซ้อนข้าม type โดยไม่เสีย type safety ตอน compile time ต่างจากวิธีเดิม (`interface{}` + type assertion ที่ error ตอน runtime, หรือ code generation ที่ซับซ้อนขึ้น)
- Syntax พื้นฐาน: `func Foo[T constraint](x T) T` — `[T constraint]` คือ type parameter list ที่กำหนดว่า `T` ต้องเป็น type แบบไหนได้บ้าง
- `any` เป็น constraint ที่ยอมรับทุก type (alias ของ `interface{}`), `comparable` ยอมรับเฉพาะ type ที่ใช้ `==`/`!=` ได้
- Union of types (`int | float64 | string`) ใช้กำหนด constraint ที่รองรับ operator เฉพาะ เช่น `>`, `<`
- Type instantiation ทำได้ทั้งแบบ explicit (`Max[int](10, 20)`) และให้ compiler infer เอง (`Max(10, 20)`) — ควรปล่อยให้ infer เป็นค่าเริ่มต้นเสมอ
- Generic struct ประกาศ type parameter ต่อท้ายชื่อ type (`type Stack[T any] struct`) และ method ต้องระบุ `[T]` ซ้ำที่ receiver เสมอ
- Go generics ถูกออกแบบให้จำกัดกว่า C++ templates โดยตั้งใจ เพื่อรักษาความเรียบง่ายของภาษาและ error message ที่อ่านง่าย ตามปรัชญา "less is more"

## แบบฝึกหัดท้ายบท

1. เขียนฟังก์ชัน generic `Sum[T int | float64](nums []T) T` ที่รวมค่าทุกตัวใน slice แล้วทดสอบกับทั้ง `[]int` และ `[]float64`
2. เขียนฟังก์ชัน generic `Reverse[T any](s []T) []T` ที่คืน slice ที่กลับลำดับ แล้วทดสอบกับ `[]string` และ `[]int`
3. เขียน generic struct `Pair[K comparable, V any] struct { Key K; Value V }` พร้อม method `String()` ที่คืนค่ารูปแบบ `"key=value"` แล้วสร้าง instance ที่มี `K` เป็น `string` และ `V` เป็น `int`
4. เขียนฟังก์ชัน generic `Filter[T any](s []T, keep func(T) bool) []T` ที่คืน slice เฉพาะ element ที่ `keep` คืนค่า `true` แล้วทดสอบกรองเลขคู่จาก `[]int`
5. ขยาย `Stack[T any]` จากบทเรียนให้มี method `Peek() (T, bool)` ที่ดูค่าบนสุดโดยไม่เอาออกจาก stack แล้วเขียนโปรแกรมทดสอบทุก method (`Push`, `Pop`, `Peek`, `Len`)
6. ลองเขียนฟังก์ชัน `func BadMax[T any](a, b T) T { return a }` แล้วพยายามใช้ `a > b` ข้างในดูว่า compiler แจ้ง error อะไร อธิบายว่าทำไม `any` ถึงไม่พอสำหรับกรณีนี้ (เตรียมคำตอบไว้ เราจะแก้ปัญหานี้ด้วย constraint กำหนดเองใน **Part 029**)

---

**ต่อไป**: [Part 029 — Generics ขั้นสูง: constraints, generic data structures](./029-generics-advanced.md)
