# Part 003: ตัวแปร, ชนิดข้อมูล, ค่าคงที่, `iota`

> ภาคที่ 1: พื้นฐานภาษา Go (Fundamentals) — ตอนที่ 3 จาก 15

## สารบัญของบทนี้

1. การประกาศตัวแปรด้วย `var` ทุกรูปแบบ
2. Short Variable Declaration ด้วย `:=`
3. กฎ scope ของตัวแปรและปัญหา shadowing
4. Zero Value: ค่าเริ่มต้นของทุกชนิดข้อมูล
5. ชนิดข้อมูลตัวเลขทั้งหมดใน Go
6. string, rune, และ byte
7. Type Conversion แบบ explicit เท่านั้น (ไม่มี implicit coercion)
8. ค่าคงที่ด้วย `const`
9. `iota` และการสร้าง enum แบบ Go
10. Untyped Constants: จุดเด่นที่ภาษาอื่นไม่มี
11. สรุปสิ่งที่ได้เรียนในบทนี้
12. แบบฝึกหัดท้ายบท

---

## 1. การประกาศตัวแปรด้วย `var` ทุกรูปแบบ

Go เป็นภาษา **static typed** หมายความว่าทุกตัวแปรมีชนิดข้อมูลตายตัวตั้งแต่ compile time แต่ Go ก็มี **type inference** ช่วยให้ไม่ต้องเขียนชนิดข้อมูลซ้ำซากทุกครั้ง keyword หลักสำหรับประกาศตัวแปรคือ `var`

### รูปแบบที่ 1: ระบุชนิดข้อมูล ไม่กำหนดค่าเริ่มต้น

```go
var age int
```

ตัวแปรจะได้ **zero value** ของชนิดนั้นทันที (เรื่อง zero value จะเจาะลึกในหัวข้อที่ 4)

### รูปแบบที่ 2: ระบุทั้งชนิดข้อมูลและค่าเริ่มต้น

```go
var name string = "Somchai"
```

### รูปแบบที่ 3: ไม่ระบุชนิดข้อมูล ให้ Go infer จากค่าที่กำหนด

```go
var score = 95.5   // Go infer ว่าเป็น float64 โดยอัตโนมัติ
```

### รูปแบบที่ 4: ประกาศหลายตัวแปรในบรรทัดเดียว

```go
var width, height int = 10, 20
```

### รูปแบบที่ 5: กลุ่มตัวแปรด้วยวงเล็บ (แนะนำเมื่อมีหลายตัวแปรระดับ package หรือต้นฟังก์ชัน)

```go
var (
	isActive bool
	count    int
	title    string = "หัวข้อ"
)
```

รูปแบบนี้ทำให้จัดกลุ่มตัวแปรที่เกี่ยวข้องกันให้อ่านง่าย และเป็นรูปแบบที่เห็นบ่อยที่สุดตอนประกาศตัวแปรระดับ package (นอกฟังก์ชัน) ทดสอบรวมทุกรูปแบบ:

```go
package main

import "fmt"

func main() {
	var age int
	var name string = "Somchai"
	var score = 95.5
	var width, height int = 10, 20

	var (
		isActive bool
		count    int
		title    string = "หัวข้อ"
	)

	fmt.Println(age, name, score, width, height, isActive, count, title)
}
```

ผลลัพธ์:

```
0 Somchai 95.5 10 20 false 0 หัวข้อ
```

สังเกตว่า `age`, `isActive`, `count` ไม่ได้กำหนดค่า แต่กลับมีค่าแสดงออกมา (`0`, `false`, `0`) — นี่คือ **zero value** ซึ่งเป็นหนึ่งในการออกแบบที่สำคัญของ Go: **ไม่มีตัวแปรที่เป็น "ค่าว่าง/undefined" แบบ uninitialized ใน C** ทุกตัวแปรที่ประกาศแล้วมีค่าเสมอ

---

## 2. Short Variable Declaration ด้วย `:=`

Go มี syntax ลัดที่ใช้บ่อยที่สุดในทางปฏิบัติ คือ **short variable declaration** ด้วยเครื่องหมาย `:=` ซึ่งประกาศตัวแปรพร้อม infer ชนิดข้อมูลในขั้นตอนเดียว:

```go
city := "Bangkok"
population := 10_000_000  // underscore คั่นหลักได้เพื่ออ่านง่าย ไม่มีผลต่อค่าจริง
fmt.Println(city, population)
```

`:=` เทียบเท่ากับการเขียน `var city = "Bangkok"` แบบสั้น — **แต่มีข้อจำกัดสำคัญที่ `var` ไม่มี**:

### ข้อจำกัดที่ 1: ใช้ได้เฉพาะภายในฟังก์ชันเท่านั้น

`:=` **ใช้ไม่ได้**ที่ระดับ package (นอกฟังก์ชัน) ต้องใช้ `var` เท่านั้นสำหรับตัวแปรระดับ package:

```go
package main

// count := 0   // ❌ compile error: syntax error ถ้าเขียนแบบนี้นอกฟังก์ชัน
var count = 0   // ✅ ใช้ var แทน

func main() {
	x := 10   // ✅ ใช้ได้ในฟังก์ชัน
	_ = x
}
```

### ข้อจำกัดที่ 2: ต้องมีตัวแปรใหม่อย่างน้อย 1 ตัวทางซ้าย

ถ้าประกาศตัวแปรชื่อเดิมซ้ำด้วย `:=` โดยไม่มีตัวแปรใหม่เลย จะเป็น compile error:

```go
x := 5
x := 10   // ❌ error
fmt.Println(x)
```

ผลลัพธ์จาก `go run`:

```
./main.go:7:4: no new variables on left side of :=
```

แต่ถ้ามีตัวแปรใหม่ปนอยู่ด้วยอย่างน้อย 1 ตัว จะใช้ได้ (ตัวแปรเดิมจะถูก**กำหนดค่าใหม่** ไม่ใช่ประกาศซ้ำ):

```go
x := 5
x, y := 10, 20   // ✅ x ถูก assign ค่าใหม่, y ถูกประกาศใหม่
fmt.Println(x, y) // 10 20
```

### เมื่อไหร่ใช้ `var`, เมื่อไหร่ใช้ `:=`

| สถานการณ์ | ใช้ |
|---|---|
| ตัวแปรระดับ package (นอกฟังก์ชัน) | `var` เท่านั้น |
| ต้องการ zero value ชัดเจน ไม่กำหนดค่าเริ่มต้น | `var x Type` |
| ต้องการชนิดข้อมูลต่างจากที่ infer อัตโนมัติ (เช่น `float64` แทน `int`) | `var x float64 = 5` |
| ประกาศ+กำหนดค่าเริ่มต้นในฟังก์ชัน แบบสั้นกระชับ | `:=` (นิยมที่สุดในโค้ด Go จริง) |

---

## 3. กฎ scope ของตัวแปรและปัญหา shadowing

Go ใช้ **block scope**: ตัวแปรมีชีวิตอยู่ภายใน `{ }` ที่มันถูกประกาศเท่านั้น เมื่อออกจาก block ตัวแปรนั้นจะไม่สามารถเข้าถึงได้อีก

ปัญหาที่พบบ่อยที่สุดสำหรับมือใหม่คือ **variable shadowing** — การประกาศตัวแปรชื่อเดิมซ้ำใน scope ที่อยู่ลึกกว่า ทำให้ตัวแปรชั้นในบัง (shadow) ตัวแปรชั้นนอกชั่วคราว:

```go
x := 10
{
	x := 20   // ตัวแปรใหม่ scope ในวงเล็บนี้เท่านั้น ไม่ใช่ตัวเดียวกับ x ข้างนอก
	fmt.Println("inner x:", x)
}
fmt.Println("outer x:", x)
```

ผลลัพธ์:

```
inner x: 20
outer x: 10
```

สังเกตว่าค่า `x` วงนอกไม่ถูกเปลี่ยนเลย เพราะ `x := 20` ข้างในสร้างตัวแปรใหม่คนละตัวที่บังเฉยๆ ไม่ใช่การ assign ทับตัวเดิม

### กับดักยอดฮิต: shadowing ใน `if`/`err`

Shadowing เป็นบั๊กที่พบบ่อยมากในโค้ด Go จริง โดยเฉพาะกับรูปแบบตรวจ error (จะเรียนเจาะลึกเรื่อง `error` ใน Part 015) ตัวอย่างรูปแบบที่ทำให้เกิดบั๊กโดยไม่รู้ตัว:

```go
func doSomething() (int, error) {
	return 42, nil
}

func problematic() error {
	result := 0
	if val, err := doSomething(); err == nil {
		result = val
		// err ที่ใช้ตรงนี้เป็นตัวใหม่ใน scope ของ if เท่านั้น
	}
	// ตัวแปร err ด้านนอก (ถ้ามี) จะไม่ถูกกระทบเลย เพราะมันคนละตัวกัน
	_ = result
	return nil
}
```

บทเรียนสำคัญ: **ระวังการใช้ `:=` ซ้ำในหลาย scope ที่ซ้อนกัน** ถ้าตั้งใจจะแก้ไขตัวแปรเดิม ให้ใช้ `=` (assignment เฉยๆ ไม่ประกาศใหม่) ไม่ใช่ `:=`

---

## 4. Zero Value: ค่าเริ่มต้นของทุกชนิดข้อมูล

หนึ่งในการออกแบบที่สำคัญของ Go คือ **ทุกตัวแปรที่ประกาศแล้วแต่ยังไม่ได้กำหนดค่า จะมี "zero value" เสมอ ไม่มีสถานะ uninitialized/garbage แบบ C** ตารางค่าเริ่มต้นของแต่ละชนิด:

| ชนิดข้อมูล | Zero Value |
|---|---|
| ตัวเลขทุกชนิด (`int`, `float64`, ฯลฯ) | `0` |
| `bool` | `false` |
| `string` | `""` (string ว่าง ไม่ใช่ `nil`) |
| `slice` | `nil` |
| `map` | `nil` |
| `pointer` | `nil` |
| `func` | `nil` |
| `channel` | `nil` |
| `interface` | `nil` |
| `struct` | struct ที่ทุก field เป็น zero value ของตัวเอง (เรียนเจาะลึกใน Part 011) |

ทดสอบยืนยันจริง:

```go
var zInt int
var zFloat float64
var zBool bool
var zString string
var zSlice []int
var zMap map[string]int
var zPtr *int
var zFunc func()
var zChan chan int
var zInterface interface{}

fmt.Println(zInt, zFloat, zBool, zString == "", zSlice == nil,
	zMap == nil, zPtr == nil, zFunc == nil, zChan == nil, zInterface == nil)
```

ผลลัพธ์:

```
0 0 false true true true true true true true
```

> **ข้อควรระวัง**: `zString == ""` เป็น `true` เพราะ zero value ของ string คือ string ว่าง **ไม่ใช่** `nil` (ต่างจากภาษาอย่าง Java ที่ String ที่ไม่ initialize จะเป็น `null`) ส่วน `slice`, `map`, `pointer`, `func`, `channel`, `interface` เป็นกลุ่ม "reference-like type" ที่ zero value คือ `nil` เหมือนกันหมด — เรื่อง `nil` slice vs empty slice จะเจาะลึกใน Part 006

---

## 5. ชนิดข้อมูลตัวเลขทั้งหมดใน Go

Go มีชนิดข้อมูลตัวเลขที่ระบุขนาด (bit width) ชัดเจน ต่างจากภาษาสคริปต์ที่มักมี "number" ชนิดเดียว

### จำนวนเต็มมีเครื่องหมาย (signed integer)

| ชนิด | ขนาด | ช่วงค่า |
|---|---|---|
| `int8` | 8 bit | -128 ถึง 127 |
| `int16` | 16 bit | -32,768 ถึง 32,767 |
| `int32` (นามแฝง: `rune`) | 32 bit | -2,147,483,648 ถึง 2,147,483,647 |
| `int64` | 64 bit | -9,223,372,036,854,775,808 ถึง 9,223,372,036,854,775,807 |
| `int` | 32 หรือ 64 bit **ขึ้นกับ platform** (ปัจจุบันแทบทุกเครื่อง 64 bit) | เท่ากับ `int64` บนเครื่องส่วนใหญ่ในปัจจุบัน |

### จำนวนเต็มไม่มีเครื่องหมาย (unsigned integer)

| ชนิด | ขนาด | ช่วงค่า |
|---|---|---|
| `uint8` (นามแฝง: `byte`) | 8 bit | 0 ถึง 255 |
| `uint16` | 16 bit | 0 ถึง 65,535 |
| `uint32` | 32 bit | 0 ถึง 4,294,967,295 |
| `uint64` | 64 bit | 0 ถึง 18,446,744,073,709,551,615 |
| `uint` | 32 หรือ 64 bit ขึ้นกับ platform | เท่ากับ `uint64` บนเครื่องส่วนใหญ่ |
| `uintptr` | ขนาดพอเก็บ memory address ได้ | ใช้เฉพาะงาน low-level เช่น `unsafe` |

### จำนวนทศนิยม (floating point)

| ชนิด | ความละเอียด |
|---|---|
| `float32` | ความละเอียดประมาณ 6-7 หลักทศนิยม (IEEE-754 single precision) |
| `float64` | ความละเอียดประมาณ 15-17 หลักทศนิยม (IEEE-754 double precision) — **เป็นค่า default เมื่อ infer จาก literal ทศนิยม** |

### จำนวนเชิงซ้อน (complex number)

| ชนิด | ประกอบด้วย |
|---|---|
| `complex64` | ส่วนจริง+จินตภาพเป็น `float32` อย่างละตัว |
| `complex128` | ส่วนจริง+จินตภาพเป็น `float64` อย่างละตัว — เป็นค่า default เมื่อ infer |

ทดสอบประกาศและใช้งานครบทุกชนิด:

```go
var i8 int8 = 127
var u8 uint8 = 255
var i16 int16 = 32000
var u16 uint16 = 65000
var i32 int32 = 2147483647
var u32 uint32 = 4294967295
var i64 int64 = 9223372036854775807
var u64 uint64 = 18446744073709551615
var f32 float32 = 3.14
var f64 float64 = 3.141592653589793
var c64 complex64 = 3 + 4i
var c128 complex128 = complex(3, 4)

fmt.Println(i8, u8, i16, u16, i32, u32, i64, u64, f32, f64, c64, c128)
fmt.Println("abs of c128:", cmplx.Abs(c128))
```

ผลลัพธ์:

```
127 255 32000 65000 2147483647 4294967295 9223372036854775807 18446744073709551615 3.14 3.141592653589793 (3+4i) (3+4i)
abs of c128: 5
```

(ต้อง `import "math/cmplx"` เพื่อใช้ `cmplx.Abs` คำนวณ magnitude ของจำนวนเชิงซ้อน)

### เลือกใช้ชนิดไหนดี

แนวทางปฏิบัติจริงที่ทีม Go ส่วนใหญ่ยึดถือ:

- **ใช้ `int` เป็นค่า default** สำหรับตัวเลขจำนวนเต็มทั่วไป (นับจำนวน, index, loop counter) ยกเว้นมีเหตุผลเฉพาะ เช่น ต้องการควบคุมขนาด memory ที่แน่นอน (โครงสร้างข้อมูลขนาดใหญ่จำนวนมาก) หรือต้อง interoperate กับ binary format/protocol ที่กำหนดขนาด bit ตายตัว (เช่น อ่านไฟล์ binary, network protocol)
- **ใช้ `float64` เป็นค่า default** สำหรับทศนิยม ยกเว้นมีข้อจำกัดด้าน memory ชัดเจนจริงๆ ถึงจะใช้ `float32`
- หลีกเลี่ยงการผสมชนิดตัวเลขต่างกันในนิพจน์เดียว (ดูหัวข้อที่ 7 เรื่อง type conversion)

### ตัวเลข literal แบบพิเศษ

Go รองรับการเขียนตัวเลขหลายฐานและใส่ underscore คั่นหลักเพื่อความอ่านง่าย:

```go
decimal := 1_000_000        // ฐาน 10 พร้อม underscore คั่นหลัก
hex := 0xFF                 // ฐาน 16 (hexadecimal) = 255
octal := 0o17                // ฐาน 8 (octal) = 15
binary := 0b1010            // ฐาน 2 (binary) = 10
scientific := 1.5e3          // scientific notation = 1500
```

---

## 6. string, rune, และ byte

### string

`string` ใน Go คือ**ลำดับของ byte ที่ immutable** (แก้ไขค่าตรงตำแหน่งไม่ได้โดยตรง) โดย convention เข้ารหัสเป็น **UTF-8**

```go
greeting := "สวัสดี"
fmt.Println(len(greeting))  // ความยาวเป็น "จำนวน byte" ไม่ใช่จำนวนตัวอักษร!
```

เพราะภาษาไทยใช้ UTF-8 หลาย byte ต่อ 1 ตัวอักษร `len("สวัสดี")` จะไม่เท่ากับจำนวนตัวอักษรที่เห็น — เรื่องนี้จะเจาะลึกพร้อมวิธีนับตัวอักษรที่ถูกต้องด้วย `utf8.RuneCountInString` ใน **Part 019 (String และแพ็กเกจ strings)**

### byte

`byte` คือ **นามแฝง (alias)** ของ `uint8` ใช้แทนข้อมูลดิบ 1 byte (0-255) เวลา loop ผ่าน string ด้วย index ธรรมดา (`s[i]`) จะได้ค่าเป็น `byte` ทีละตัว (ทีละ 1 byte ไม่ใช่ทีละตัวอักษร)

### rune

`rune` คือ **นามแฝงของ `int32`** ใช้แทน 1 Unicode code point (ตัวอักษร 1 ตัวไม่ว่าจะใช้กี่ byte เก็บก็ตาม) นี่คือชนิดข้อมูลที่ Go ใช้แทน "ตัวอักษร" อย่างถูกต้องตามหลัก Unicode

```go
var b byte = 'A'      // 'A' คือ rune literal แต่ assign ให้ byte ได้เพราะค่าอยู่ในช่วง 0-255
var r rune = '๙'      // เลขไทย ๙ เป็น 1 code point แต่ใช้มากกว่า 1 byte ใน UTF-8
fmt.Println(b, r, string(r))
```

ผลลัพธ์:

```
65 3673 ๙
```

สังเกตว่า `fmt.Println(b, r, ...)` พิมพ์ `b` และ `r` ออกมาเป็น**ตัวเลข** (`65` และ `3673`) ไม่ใช่ตัวอักษร เพราะ `byte`/`rune` ที่แท้จริงแล้วก็คือตัวเลขจำนวนเต็ม การจะพิมพ์เป็นตัวอักษรต้องแปลงเป็น `string` อย่างชัดเจนด้วย `string(r)` ก่อน (`๙` มีค่า Unicode code point เท่ากับ `3673` ในเลขฐาน 10 และ `'A'` มีค่า ASCII เท่ากับ `65`)

### เปรียบเทียบ byte vs rune

| | `byte` (`uint8`) | `rune` (`int32`) |
|---|---|---|
| แทนอะไร | 1 byte ดิบ | 1 Unicode code point (ตัวอักษร 1 ตัว) |
| ขนาด | 8 bit | 32 bit |
| ใช้เมื่อไหร่ | ประมวลผลข้อมูล binary ดิบ, ASCII ล้วน | ประมวลผลข้อความที่มีตัวอักษรนอก ASCII (ภาษาไทย, อิโมจิ, ภาษาอื่นๆ) |
| การ loop ผ่าน string | `for i := 0; i < len(s); i++` ได้ `s[i]` เป็น byte | `for _, r := range s` ได้ `r` เป็น rune (ถูกต้องตาม Unicode) |

เรื่องการ `range` ผ่าน string อย่างถูกต้อง (ได้ rune ไม่ใช่ byte) จะเจาะลึกใน **Part 005 (Loops)**

---

## 7. Type Conversion แบบ explicit เท่านั้น (ไม่มี implicit coercion)

นี่คือกฎที่สำคัญที่สุดข้อหนึ่งของระบบชนิดข้อมูลใน Go และเป็นจุดที่ผู้เขียนโปรแกรมจากภาษาอื่น (JavaScript, Python, C) มักตกใจ:

> **Go ไม่มีการแปลงชนิดข้อมูลให้อัตโนมัติ (no implicit type coercion) แม้ระหว่างชนิดตัวเลขที่ดู "ใกล้เคียงกัน" ก็ตาม ต้อง cast/convert ด้วยตัวเองเสมอ**

ลองดู error จริงเมื่อพยายามบวกเลขต่างชนิดโดยไม่แปลง:

```go
var x int32 = 10
var y int64 = 20
z := x + y   // ❌ compile error
```

ผลลัพธ์:

```
./main.go:6:7: invalid operation: x + y (mismatched types int32 and int64)
```

แม้ `int32` กับ `int64` จะเป็น "จำนวนเต็ม" เหมือนกัน Go ก็ถือว่าเป็นคนละชนิดกันเด็ดขาด ต้อง convert ให้ตรงกันก่อนเสมอ:

```go
var x int32 = 10
var y int64 = 20
z := int64(x) + y   // ✅ แปลง x เป็น int64 ก่อนบวก
fmt.Println(z)
```

### Syntax การแปลงชนิดข้อมูล

รูปแบบทั่วไปคือ `TargetType(value)`:

```go
var n int = 42
var f float64 = float64(n)   // int → float64
var n2 int = int(f)          // float64 → int (ตัดทศนิยมทิ้ง ไม่ปัดเศษ)
fmt.Println(n, f, n2)        // 42 42 42
```

```go
var iv int32 = 65
var s string = string(iv)    // int32 → string: แปลง code point เป็นตัวอักษร
fmt.Println(s)               // "A" (เพราะ 65 คือรหัส ASCII ของ 'A')
```

> **ข้อควรระวัง**: `string(iv)` เมื่อ `iv` เป็นชนิดตัวเลข จะตีความค่าตัวเลขนั้นเป็น **Unicode code point** แล้วแปลงเป็นตัวอักษรที่สอดคล้องกัน **ไม่ใช่**การแปลง "42" เป็น string "42" แบบที่หลายคนคาดหวัง (ถ้าต้องการแปลงตัวเลขเป็น string ของตัวเลขนั้นจริงๆ ต้องใช้ `strconv.Itoa(n)` ซึ่งจะเรียนใน **Part 020**)

### ผลลัพธ์เมื่อแปลงแบบ "เสียข้อมูล" (narrowing conversion)

การแปลงจากชนิดที่มีช่วงค่ากว้างไปชนิดที่แคบกว่า Go **ไม่ error แต่ตัดบิตทิ้งเงียบๆ** ตามกฎ modular arithmetic:

```go
var big int32 = 300
var small int8 = int8(big)
fmt.Println(small)   // ไม่ error แต่ได้ 44 (300 mod 256 แล้วตีความเป็นค่าติดลบ/บวกตาม two's complement)
```

นี่คือจุดที่ต้อง**ระวังด้วยตัวเอง** เพราะ compiler จะไม่เตือนตอน runtime (ต่างจากตอน constant ที่ compiler ตรวจให้ตามหัวข้อถัดไป) ทีมงานที่รอบคอบมักเขียนฟังก์ชันตรวจสอบขอบเขตก่อนแปลงเองในจุดที่มีความเสี่ยง

---

## 8. ค่าคงที่ด้วย `const`

ค่าคงที่ประกาศด้วย `const` — ค่าต้องรู้ตอน compile time เท่านั้น (คำนวณจาก literal หรือ const ตัวอื่น ไม่ใช่ผลจากการเรียกฟังก์ชันตอน runtime):

```go
const Pi = 3.14159
```

ประกาศเป็นกลุ่มด้วยวงเล็บได้เช่นเดียวกับ `var`:

```go
const (
	StatusActive   = "active"
	StatusInactive = "inactive"
	StatusDeleted  = "deleted"
)
```

ข้อแตกต่างสำคัญจาก `var`:

1. **ค่าคงที่แก้ไขไม่ได้ตลอดโปรแกรม** พยายาม assign ค่าใหม่จะเป็น compile error ทันที
2. **ต้องกำหนดค่าตอนประกาศเสมอ** ไม่มี "zero value const" (ยกเว้นใช้ `iota` ตามหัวข้อถัดไป)
3. **ค่าต้องคำนวณได้ตอน compile time** เช่น `const x = someFunction()` จะ error ถ้า `someFunction()` เป็น function ธรรมดาที่ทำงานตอน runtime

---

## 9. `iota` และการสร้าง enum แบบ Go

Go **ไม่มี keyword `enum` แบบภาษาอื่น** แต่ใช้ `const` ร่วมกับ `iota` แทน — `iota` เป็นตัวนับพิเศษที่**รีเซ็ตเป็น 0 ทุกครั้งที่ขึ้น `const (...)` บล็อกใหม่ และเพิ่มค่าอัตโนมัติทีละ 1 ในแต่ละบรรทัดถัดไปของบล็อกเดียวกัน**

### ตัวอย่างพื้นฐานที่สุด: enum วันในสัปดาห์

```go
type Weekday int

const (
	Sunday Weekday = iota // 0
	Monday                // 1 (สืบทอด expression เดิมจากบรรทัดบน)
	Tuesday                // 2
	Wednesday              // 3
	Thursday               // 4
	Friday                 // 5
	Saturday               // 6
)

func (d Weekday) String() string {
	names := [...]string{"อาทิตย์", "จันทร์", "อังคาร", "พุธ", "พฤหัสบดี", "ศุกร์", "เสาร์"}
	return names[d]
}

func main() {
	fmt.Println(Monday, Friday)
}
```

ผลลัพธ์:

```
จันทร์ ศุกร์
```

(เมธอด `String()` ทำให้ `fmt.Println` เรียกใช้อัตโนมัติเพื่อแปลงเป็นข้อความที่อ่านง่าย — กลไกนี้เรียกว่า `Stringer` interface จะเจาะลึกใน Part 013)

จุดสำคัญ: บรรทัด `Monday` ไม่ได้เขียน `= iota` ซ้ำ แต่ **สืบทอด expression จากบรรทัดก่อนหน้าโดยอัตโนมัติ** (กฎของ Go: ถ้าไม่เขียน expression ในบรรทัดถัดไปของ `const` block เดียวกัน จะใช้ expression เดียวกับบรรทัดก่อนหน้า แต่ `iota` จะขยับค่าไปตามตำแหน่งบรรทัดเสมอ)

### ตัวอย่างขั้นสูง: หน่วยความจำด้วย bit shift

รูปแบบยอดนิยมอีกแบบคือใช้ `iota` ร่วมกับ bit shift operator (`<<` จะเจาะลึกใน Part 004) เพื่อสร้างค่าคูณ 2 ยกกำลัง:

```go
const (
	_  = iota                   // ข้ามค่า 0 (บรรทัดแรกทิ้งไปเลยด้วย _)
	KB = 1 << (10 * iota)       // iota = 1 → 1 << 10 = 1024
	MB                          // iota = 2 → 1 << 20 = 1,048,576
	GB                          // iota = 3 → 1 << 30 = 1,073,741,824
	TB                          // iota = 4 → 1 << 40 = 1,099,511,627,776
)

func main() {
	fmt.Println(KB, MB, GB, TB)
}
```

ผลลัพธ์:

```
1024 1048576 1073741824 1099511627776
```

เทคนิค `_ = iota` ในบรรทัดแรกเป็นสำนวนยอดนิยมที่ใช้ **ข้ามค่า 0** เพราะในตัวอย่างนี้เราไม่ต้องการให้มีค่าคงที่ที่แทน "0 byte" (ใช้ `_` ซึ่งเป็น blank identifier ทิ้งค่าที่ไม่ต้องการ — เรียนเจาะลึกเรื่อง `_` ใน Part 008)

### กฎสำคัญของ `iota` ที่ต้องจำ

1. `iota` เริ่มที่ `0` เสมอในทุก `const (...)` บล็อกใหม่
2. เพิ่มค่าทีละ 1 **ทุกบรรทัด** ในบล็อกเดียวกัน แม้บรรทัดนั้นจะเป็น `_` (บรรทัดว่าง/ข้าม) ก็ยังนับ
3. ถ้าบรรทัดไม่มี expression ระบุไว้ จะใช้ expression เดียวกับบรรทัดล่าสุดที่มีการระบุไว้ (แต่ `iota` เปลี่ยนค่าตามตำแหน่งบรรทัดเสมอ)
4. `iota` ใช้ได้เฉพาะภายใน `const` block เท่านั้น

---

## 10. Untyped Constants: จุดเด่นที่ภาษาอื่นไม่มี

ค่าคงที่ใน Go มีความสามารถพิเศษที่เรียกว่า **untyped constant** — ถ้าประกาศ `const` โดยไม่ระบุชนิดข้อมูล ค่านั้นจะยังไม่ถูกผูกกับชนิดข้อมูลใดตายตัว และสามารถนำไปใช้กับหลายชนิดข้อมูลที่เข้ากันได้โดยไม่ต้อง convert:

```go
const Pi = 3.14159   // untyped constant

var radius1 float32 = 2.0
var radius2 float64 = 2.0

area1 := Pi * radius1 * radius1   // Pi ถูกตีความเป็น float32 อัตโนมัติในบริบทนี้
area2 := Pi * radius2 * radius2   // Pi ถูกตีความเป็น float64 อัตโนมัติในบริบทนี้
```

นี่คือข้อยกเว้นเดียวที่ดู "เหมือน" implicit conversion แต่จริงๆ แล้ว **ไม่ใช่** — เพราะ `Pi` ไม่มีชนิดข้อมูลตายตัวตั้งแต่แรก มันจะ "รับชนิด" จากบริบทที่ถูกใช้งานในแต่ละจุดแยกกัน ต่างจากตัวแปรที่ประกาศด้วย `var`/`:=` ซึ่งมีชนิดตายตัวทันทีที่ประกาศ

เทียบให้เห็นชัด — ถ้าเปลี่ยน `Pi` เป็น typed constant จะใช้กับ `float32` ไม่ได้ทันที:

```go
const PiFloat64 float64 = 3.14159   // typed constant — ผูกเป็น float64 ตายตัว

var radius1 float32 = 2.0
// area1 := PiFloat64 * radius1 * radius1  // ❌ mismatched types float64 and float32
```

### Constant overflow ถูกตรวจตอน compile time

จุดแข็งอีกอย่างของ typed/untyped constant คือ Go ตรวจสอบ **overflow ตอน compile time** ได้เลย ไม่ต้องรอ error ตอน runtime:

```go
var x int8 = 200
```

ผลลัพธ์:

```
./main.go:4:15: cannot use 200 (untyped int constant) as int8 value in variable declaration (overflows)
```

สังเกตว่า error message บอกตรงๆ ว่า `200` เป็น "untyped int constant" — Go รู้ตั้งแต่ compile time ว่าค่า `200` ใส่ใน `int8` (max 127) ไม่ได้ จึง reject ทันที นี่คือประโยชน์ของระบบ constant ที่ผูกกับ static type checking อย่างเข้มงวด ป้องกันบั๊กประเภท silent overflow ได้ตั้งแต่ก่อนรันโปรแกรมจริงด้วยซ้ำ (ต่างจากตอน runtime conversion ในหัวข้อที่ 7 ที่ overflow แบบเงียบๆ ได้ เพราะตอนนั้นเป็นการแปลงค่าตัวแปร ไม่ใช่ constant)

---

## สรุปสิ่งที่ได้เรียนในบทนี้

- `var` มีหลายรูปแบบ: ระบุ/ไม่ระบุชนิด, กำหนด/ไม่กำหนดค่าเริ่มต้น, ประกาศเดี่ยว/กลุ่ม
- `:=` ใช้ได้เฉพาะในฟังก์ชัน ต้องมีตัวแปรใหม่อย่างน้อย 1 ตัวทางซ้ายเสมอ
- Go ใช้ block scope และมีปัญหา **shadowing** ที่ต้องระวัง โดยเฉพาะใน `if`/error-checking pattern
- ทุกตัวแปรมี **zero value** เสมอ ไม่มีสถานะ uninitialized — ตัวเลขเป็น 0, bool เป็น false, string เป็น "", ส่วน slice/map/pointer/func/channel/interface เป็น nil
- Go มีชนิดตัวเลขครบทุกขนาด: `int8`-`int64`, `uint8`-`uint64`, `float32`/`float64`, `complex64`/`complex128` และ `int`/`uint` ที่ขนาดขึ้นกับ platform
- `byte` คือ alias ของ `uint8` (1 byte ดิบ), `rune` คือ alias ของ `int32` (1 Unicode code point)
- Go **ไม่มี implicit type conversion** แม้ระหว่างชนิดตัวเลขที่ดูใกล้เคียงกัน ต้อง convert ด้วย `TargetType(value)` เสมอ
- Narrowing conversion (แปลงไปชนิดที่แคบกว่า) ไม่ error แต่ตัดบิตทิ้งแบบ modular arithmetic ต้องระวังเอง
- `const` ใช้สร้างค่าคงที่ที่รู้ค่าตอน compile time เท่านั้น
- `iota` คือตัวนับพิเศษใน `const` block เริ่มที่ 0 เพิ่มทีละ 1 ทุกบรรทัด ใช้สร้าง enum-like pattern ได้ทรงพลัง (รวมถึงเทคนิค bit shift สำหรับหน่วยความจำ)
- **Untyped constant** ไม่มีชนิดตายตัว รับชนิดจากบริบทที่ใช้งาน และ Go ตรวจ overflow ของ constant ได้ตั้งแต่ compile time

## แบบฝึกหัดท้ายบท

1. เขียนโปรแกรมประกาศตัวแปรด้วยทั้ง 5 รูปแบบของ `var` ที่กล่าวถึงในบทนี้ พร้อม comment อธิบายแต่ละรูปแบบ
2. เขียนฟังก์ชันที่มีตัวแปรชื่อ `total` ประกาศนอก `if` แล้วมี `if` ที่ใช้ `:=` ประกาศ `total` ซ้ำข้างในโดยไม่ตั้งใจ (shadowing) รันดูผลลัพธ์ที่ผิดเพี้ยนไปจากที่ตั้งใจ แล้วแก้ไขให้ถูกต้องด้วยการใช้ `=` แทน
3. เขียนโปรแกรมที่ประกาศตัวแปรทุกชนิดตัวเลขที่เรียนในบทนี้ (`int8` ถึง `complex128`) แล้ว `fmt.Println` ค่า zero value ของแต่ละตัว (ไม่กำหนดค่าเริ่มต้นเลย)
4. ลองเขียนโค้ดที่บวกตัวแปรชนิด `int32` กับ `int64` ตรงๆ โดยไม่แปลงชนิด สังเกต error message แล้วแก้ไขให้ compile ผ่านด้วยการ convert ที่ถูกต้อง
5. สร้าง enum ด้วย `iota` สำหรับสถานะคำสั่งซื้อ (`OrderPending`, `OrderPaid`, `OrderShipped`, `OrderDelivered`, `OrderCancelled`) พร้อมเขียนเมธอด `String()` ให้แสดงชื่อสถานะเป็นภาษาไทยเมื่อ print
6. ทดลองสร้าง `const` แบบ untyped และแบบ typed อย่างละตัว แล้วพิสูจน์ด้วยโค้ดว่า untyped constant ใช้ได้กับทั้ง `float32` และ `float64` แต่ typed constant ใช้ได้กับชนิดที่ระบุไว้เท่านั้น

---

**ต่อไป**: [Part 004 — Operators และ Control Flow](./004-operators-and-control-flow.md)
