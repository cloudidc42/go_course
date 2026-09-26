# Part 020: แพ็กเกจ `strconv`

> ภาคที่ 2: ระดับกลาง (Intermediate) — ตอนที่ 5 จาก 20 (Part 16–35)

## สารบัญของบทนี้

1. ทำไมต้องมีแพ็กเกจแยกสำหรับแปลงชนิดข้อมูลเป็น string
2. แปลง string เป็นตัวเลข: `Atoi`, `ParseInt`, `ParseFloat`, `ParseBool`
3. แปลงตัวเลขกลับเป็น string: `Itoa`, `FormatInt`, `FormatFloat`, `FormatBool`
4. `*strconv.NumError`: จัดการ error จาก input ที่ผิดรูปแบบ (เชื่อมกับ Part 016)
5. การแปลงฐานเลข: ฐาน 2, 8, 16 และการเดา base อัตโนมัติ
6. `Quote`/`Unquote`: escape และ unescape ข้อความ
7. ข้อผิดพลาดที่พบบ่อยที่สุด: ลืมเช็ค error จาก `Atoi`
8. สรุปสิ่งที่ได้เรียนในบทนี้
9. แบบฝึกหัดท้ายบท

---

## 1. ทำไมต้องมีแพ็กเกจแยกสำหรับแปลงชนิดข้อมูลเป็น string

Go เป็นภาษา **static typing ที่เข้มงวด** (strict static typing) ตามที่เรียนมาตั้งแต่ Part 001 — ต่างจากภาษาที่ type ยืดหยุ่นอย่าง JavaScript หรือ Python ที่ตัวเลขกับ string สามารถผสมกันได้ค่อนข้างอิสระ ใน Go **ไม่มีการแปลงชนิดข้อมูลแบบอัตโนมัติ (implicit conversion) ระหว่าง `string` กับตัวเลขเลย**

```go
n := 42
s := "จำนวน: " + n // compile error! mismatched types string and int
```

เมื่อต้องการรวมตัวเลขเข้ากับข้อความ หรือแปลงข้อความที่รับมาจากผู้ใช้ (เช่น จาก command line, HTTP form, ไฟล์ config) ให้กลายเป็นตัวเลขที่ใช้คำนวณได้ เราต้อง**แปลงชนิดข้อมูลอย่างชัดเจน (explicit conversion)** เท่านั้น — นี่คือหน้าที่หลักของแพ็กเกจ **`strconv`** (ย่อมาจาก **str**ing **conv**ersion) ซึ่งเป็นส่วนหนึ่งของ standard library ที่แทบทุกโปรแกรม Go ต้องใช้ไม่ทางใดก็ทางหนึ่ง

ข้อสังเกต: การแปลงระหว่าง `string` กับ `[]byte` หรือ `[]rune` ทำได้ด้วย type conversion ตรงๆ (`[]byte(s)`, `string(b)`) อยู่แล้วตามที่เรียนใน Part 019 — แต่การแปลงระหว่าง `string` กับ**ตัวเลข** (`int`, `float64`, `bool`) ต้องผ่านฟังก์ชันของ `strconv` เท่านั้น เพราะการแปลงนี้ **อาจล้มเหลวได้** (ถ้า string ไม่ใช่รูปแบบตัวเลขที่ถูกต้อง) จึงต้องมี error กำกับ ซึ่งเป็นเหตุผลที่มันไม่ใช่แค่ type conversion ธรรมดา

---

## 2. แปลง string เป็นตัวเลข: `Atoi`, `ParseInt`, `ParseFloat`, `ParseBool`

```go
package main

import (
	"fmt"
	"strconv"
)

func main() {
	n, err := strconv.Atoi("42")
	fmt.Println(n, err)

	n2, err := strconv.Atoi("abc")
	fmt.Println(n2, err)

	i64, err := strconv.ParseInt("-123", 10, 64)
	fmt.Println(i64, err)

	f, err := strconv.ParseFloat("3.14159", 64)
	fmt.Println(f, err)

	b, err := strconv.ParseBool("true")
	fmt.Println(b, err)

	b2, err := strconv.ParseBool("1")
	fmt.Println(b2, err)

	b3, err := strconv.ParseBool("ใช่")
	fmt.Println(b3, err)
}
```

ผลลัพธ์:

```
42 <nil>
0 strconv.Atoi: parsing "abc": invalid syntax
-123 <nil>
3.14159 <nil>
true <nil>
true <nil>
false strconv.ParseBool: parsing "ใช่": invalid syntax
```

### `strconv.Atoi(s string) (int, error)`

ฟังก์ชันที่ใช้บ่อยที่สุดในบรรดาทั้งหมด ชื่อย่อมาจาก **"ASCII to integer"** แปลง string เป็น `int` (ขนาดตามสถาปัตยกรรมเครื่อง คือ 32 หรือ 64 บิต) เหมาะสำหรับกรณีทั่วไปที่ไม่ต้องการควบคุมขนาด bit หรือฐานเลขเป็นพิเศษ

### `strconv.ParseInt(s string, base int, bitSize int) (int64, error)`

เวอร์ชันที่ยืดหยุ่นกว่า `Atoi` มาก — ควบคุมได้ทั้ง **ฐานเลข** (`base`) และ **ขนาดจำนวนเต็มที่ผลลัพธ์ต้องพอดี** (`bitSize`: 0, 8, 16, 32, หรือ 64):

- `base = 10` คือฐานสิบปกติ, `base = 0` ให้ Go เดาฐานจาก prefix ของ string เอง (จะอธิบายในหัวข้อ 5)
- `bitSize = 64` หมายถึงผลลัพธ์ต้องพอดีกับ `int64`, ถ้าใส่ `bitSize = 8` แต่ค่าที่ parse ได้เกินขอบเขตของ `int8` (-128 ถึง 127) จะได้ error แม้ syntax จะถูกต้อง

จริงๆ แล้ว **`strconv.Atoi(s)` คือ shortcut ของ `strconv.ParseInt(s, 10, 0)`** ที่แปลงผลลัพธ์เป็น `int` ให้อัตโนมัติ (ดูได้จาก source code ของ Go เอง) — ในทางปฏิบัติถ้าไม่ต้องการฐานพิเศษหรือควบคุม bit size ใช้ `Atoi` สั้นกว่าและอ่านง่ายกว่ามาก

### `strconv.ParseFloat(s string, bitSize int) (float64, error)`

แปลง string เป็นเลขทศนิยม `bitSize` เป็น 32 หรือ 64 (สำหรับ `float32`/`float64` ตามลำดับ) แต่ **ผลลัพธ์ที่คืนมาเป็น `float64` เสมอ** แม้ระบุ `bitSize = 32` ก็ตาม (ต้อง cast เป็น `float32` เองถ้าต้องการ) การระบุ `bitSize = 32` มีผลแค่**จำกัดค่าที่ยอมรับได้ไม่ให้เกินขอบเขตของ float32** เท่านั้น

### `strconv.ParseBool(s string) (bool, error)`

แปลง string เป็น `bool` โดยรับค่าที่หลากหลายกว่าที่คิด: `"1"`, `"t"`, `"T"`, `"TRUE"`, `"true"`, `"True"` ทั้งหมดแปลงเป็น `true` ได้ ส่วน `"0"`, `"f"`, `"F"`, `"FALSE"`, `"false"`, `"False"` แปลงเป็น `false` ได้ — นอกเหนือจากรูปแบบเหล่านี้ (เช่น `"ใช่"`, `"yes"`, `"y"`) จะได้ error ทันที

---

## 3. แปลงตัวเลขกลับเป็น string: `Itoa`, `FormatInt`, `FormatFloat`, `FormatBool`

ทิศทางตรงข้าม — แปลงตัวเลขเป็น string เพื่อนำไปแสดงผลหรือต่อกับข้อความอื่น:

```go
package main

import (
	"fmt"
	"strconv"
)

func main() {
	s := strconv.Itoa(42)
	fmt.Println(s, len(s))

	s2 := strconv.FormatInt(-123, 10)
	fmt.Println(s2)

	s3 := strconv.FormatFloat(3.14159, 'f', 2, 64)
	fmt.Println(s3)

	s4 := strconv.FormatFloat(1234.5678, 'e', 3, 64)
	fmt.Println(s4)

	s5 := strconv.FormatBool(true)
	fmt.Println(s5)
}
```

ผลลัพธ์:

```
42 2
-123
3.14
1.235e+03
```
```
true
```

### `strconv.Itoa(n int) string`

คู่ตรงข้ามของ `Atoi` ("**I**nteger **to** **a**SCII") แปลง `int` เป็น string ในฐานสิบ ใช้บ่อยที่สุดในบรรดาฟังก์ชัน format ทั้งหมด

### `strconv.FormatInt(n int64, base int) string`

เวอร์ชันที่ระบุฐานได้ (เหมือน `ParseInt` แต่ทำทิศทางกลับกัน) — `strconv.Itoa(n)` เทียบเท่ากับ `strconv.FormatInt(int64(n), 10)`

### `strconv.FormatFloat(f float64, fmt byte, prec int, bitSize int) string`

ฟังก์ชันที่ควบคุมรูปแบบการแสดงผลเลขทศนิยมได้ละเอียดที่สุด:

- **`fmt`** คือรูปแบบการแสดงผล: `'f'` (ทศนิยมปกติ เช่น `3.14`), `'e'` (scientific notation ตัวเล็ก เช่น `3.14e+00`), `'E'` (scientific notation ตัวใหญ่ เช่น `3.14E+00`), `'g'`/`'G'` (เลือกรูปแบบที่กระชับที่สุดโดยอัตโนมัติ)
- **`prec`** คือจำนวนตำแหน่งทศนิยม (ใส่ `-1` ให้ Go เลือกจำนวนตำแหน่งที่น้อยที่สุดที่ยังแปลงกลับเป็นค่าเดิมได้แม่นยำ)
- **`bitSize`** เหมือนกับใน `ParseFloat` คือ 32 หรือ 64

ในทางปฏิบัติ เมื่อแค่ต้องการ format ตัวเลขแบบง่ายๆ เพื่อ print ออกหน้าจอ นิยมใช้ `fmt.Sprintf("%.2f", f)` (ซึ่งเราจะเรียนละเอียดใน **Part 021**) มากกว่า `strconv.FormatFloat` เพราะเขียนสั้นกว่า — `strconv.FormatFloat` เหมาะกับกรณีที่ต้องการควบคุมรูปแบบเชิงโปรแกรม (เช่น เลือก `fmt` byte แบบไดนามิกตามเงื่อนไข) หรือต้องการประสิทธิภาพสูงสุดโดยไม่ผ่าน `fmt` package ที่มี overhead ของ reflection

### `strconv.FormatBool(b bool) string`

แปลง `bool` เป็น `"true"` หรือ `"false"` ตรงไปตรงมา

---

## 4. `*strconv.NumError`: จัดการ error จาก input ที่ผิดรูปแบบ (เชื่อมกับ Part 016)

จาก **Part 016** เราเรียนรู้ `errors.As` สำหรับดึง custom error type ออกจาก chain ของ error มาแล้ว — ฟังก์ชัน parse ทั้งหมดใน `strconv` (`Atoi`, `ParseInt`, `ParseFloat`, `ParseBool`) เมื่อเกิดความผิดพลาด **จะคืน error ที่มีชนิดเป็น `*strconv.NumError` เสมอ** ซึ่งเป็นตัวอย่างที่ดีมากของการนำความรู้เรื่อง custom error type ไปใช้งานจริง

```go
package main

import (
	"errors"
	"fmt"
	"strconv"
)

func parseAge(s string) (int, error) {
	age, err := strconv.Atoi(s)
	if err != nil {
		return 0, fmt.Errorf("parseAge(%q): %w", s, err)
	}
	return age, nil
}

func main() {
	_, err := parseAge("abc")
	fmt.Println("error message:", err)

	// strconv.Atoi/ParseInt คืนค่า error ชนิด *strconv.NumError เสมอ
	// ใช้ errors.As (จาก Part 016) ดึงรายละเอียดออกมาได้
	var numErr *strconv.NumError
	if errors.As(err, &numErr) {
		fmt.Println("Func:", numErr.Func)
		fmt.Println("Num (input ดิบ):", numErr.Num)
		fmt.Println("Err ภายใน:", numErr.Err)
		fmt.Println("เทียบกับ strconv.ErrSyntax:", errors.Is(numErr.Err, strconv.ErrSyntax))
	}

	_, err2 := strconv.ParseInt("99999999999999999999", 10, 64)
	var numErr2 *strconv.NumError
	if errors.As(err2, &numErr2) {
		fmt.Println("\nกรณีตัวเลขเกินขอบเขต:")
		fmt.Println("เทียบกับ strconv.ErrRange:", errors.Is(numErr2.Err, strconv.ErrRange))
	}
}
```

ผลลัพธ์:

```
error message: parseAge("abc"): strconv.Atoi: parsing "abc": invalid syntax
Func: Atoi
Num (input ดิบ): abc
Err ภายใน: invalid syntax
เทียบกับ strconv.ErrSyntax: true

กรณีตัวเลขเกินขอบเขต:
เทียบกับ strconv.ErrRange: true
```

### โครงสร้างของ `strconv.NumError`

```go
type NumError struct {
	Func string // ชื่อฟังก์ชันที่เกิด error เช่น "Atoi", "ParseInt"
	Num  string // string ต้นฉบับที่พยายาม parse
	Err  error  // สาเหตุที่แท้จริง เป็น sentinel error ตัวใดตัวหนึ่ง
}

func (e *NumError) Error() string {
	return "strconv." + e.Func + ": " + "parsing " + Quote(e.Num) + ": " + e.Err.Error()
}

func (e *NumError) Unwrap() error {
	return e.Err
}
```

สังเกตว่า `NumError` **implement ทั้ง `Error()` และ `Unwrap()`** ตรงตามรูปแบบที่เรียนใน Part 016 พอดี — field `Err` ภายในจะเป็นหนึ่งในสอง sentinel error ที่ `strconv` ประกาศไว้:

- **`strconv.ErrSyntax`** — string ไม่ใช่รูปแบบตัวเลขที่ถูกต้องเลย (เช่น `"abc"`, `""`, `"12.5"` เวลาพยายาม parse เป็น int)
- **`strconv.ErrRange`** — syntax ถูกต้อง แต่ค่าตัวเลขเกินขอบเขตของชนิดข้อมูลปลายทาง (เช่น parse `"99999999999999999999"` เป็น `int64` ซึ่งมีขอบเขตจำกัด)

การแยกสองกรณีนี้ออกจากกันมีประโยชน์มากในทางปฏิบัติ เช่น ถ้า error เป็น `ErrRange` อาจแนะนำผู้ใช้ให้กรอกตัวเลขที่เล็กลง แต่ถ้าเป็น `ErrSyntax` ต้องแจ้งว่ารูปแบบไม่ถูกต้องเลย — เขียนโค้ดแยกแยะได้ทันทีด้วย `errors.Is(numErr.Err, strconv.ErrRange)` ตามที่เรียนมาแล้ว

---

## 5. การแปลงฐานเลข: ฐาน 2, 8, 16 และการเดา base อัตโนมัติ

`strconv.ParseInt`/`FormatInt` รองรับฐานเลขได้ตั้งแต่ 2 ถึง 36 ทำให้แปลงเลขฐานสอง (binary), ฐานแปด (octal), และฐานสิบหก (hexadecimal) ได้ในฟังก์ชันเดียว

```go
package main

import (
	"fmt"
	"strconv"
)

func main() {
	// แปลงจาก string ฐานต่างๆ เป็น int64 (ฐาน 10)
	binVal, _ := strconv.ParseInt("1010", 2, 64)
	fmt.Println("เลขฐาน 2 \"1010\" ->", binVal)

	hexVal, _ := strconv.ParseInt("1F", 16, 64)
	fmt.Println("เลขฐาน 16 \"1F\" ->", hexVal)

	octVal, _ := strconv.ParseInt("17", 8, 64)
	fmt.Println("เลขฐาน 8 \"17\" ->", octVal)

	// ฐาน 0 ให้ strconv เดาจาก prefix เอง (0x, 0o, 0b, 0)
	autoVal, _ := strconv.ParseInt("0x1F", 0, 64)
	fmt.Println("auto-detect \"0x1F\" ->", autoVal)

	// แปลงกลับจาก int64 เป็น string ฐานต่างๆ
	fmt.Println("10 ในฐาน 2:", strconv.FormatInt(10, 2))
	fmt.Println("31 ในฐาน 16:", strconv.FormatInt(31, 16))
	fmt.Println("15 ในฐาน 8:", strconv.FormatInt(15, 8))
}
```

ผลลัพธ์:

```
เลขฐาน 2 "1010" -> 10
เลขฐาน 16 "1F" -> 31
เลขฐาน 8 "17" -> 15
auto-detect "0x1F" -> 31
10 ในฐาน 2: 1010
31 ในฐาน 16: 1f
15 ในฐาน 8: 17
```

### `base = 0`: การเดาฐานอัตโนมัติ

เมื่อระบุ `base = 0` ใน `strconv.ParseInt` Go จะดู **prefix** ของ string เพื่อเดาฐานให้อัตโนมัติ ตามกฎเดียวกับ literal ตัวเลขในโค้ด Go เอง (ที่เรียนใน Part 003):

| Prefix | ฐานที่เดา |
|---|---|
| `0x` หรือ `0X` | ฐาน 16 (hexadecimal) |
| `0o` หรือ `0O` | ฐาน 8 (octal) |
| `0b` หรือ `0B` | ฐาน 2 (binary) |
| `0` ตามด้วยตัวเลข (ไม่มีตัวอักษรคั่น) | ฐาน 8 (รูปแบบเก่าแบบ C) |
| ไม่มี prefix พิเศษ | ฐาน 10 |

การเดาฐานอัตโนมัติมีประโยชน์มากเมื่อรับ input จากผู้ใช้ที่อาจพิมพ์เลขมาในรูปแบบต่างๆ กัน (เช่น CLI tool ที่รับ argument เป็นเลขฐานสิบหกด้วย prefix `0x` ตามธรรมเนียมทั่วไป) โดยไม่ต้องเขียนโค้ดตรวจสอบ prefix เองก่อน

### `FormatInt` แสดงตัวอักษรฐาน 16 เป็นตัวพิมพ์เล็กเสมอ

สังเกตผลลัพธ์ `"1f"` (ตัวเล็ก) จาก `strconv.FormatInt(31, 16)` — ถ้าต้องการตัวพิมพ์ใหญ่ (`"1F"`) ต้องแปลงด้วย `strings.ToUpper` เพิ่มเติมเอง หรือใช้ `fmt.Sprintf("%X", 31)` แทน (verb ตัวใหญ่ `%X` ให้ผลลัพธ์เป็นตัวพิมพ์ใหญ่โดยตรง เราจะเรียนละเอียดใน Part 021)

---

## 6. `Quote`/`Unquote`: escape และ unescape ข้อความ

เมื่อต้องการแสดงผล string ที่อาจมีตัวอักษรพิเศษ (tab, newline, quote character) ในรูปแบบที่ปลอดภัยต่อการอ่านหรือฝังกลับเข้าไปใน source code/log file แพ็กเกจ `strconv` มีฟังก์ชัน `Quote`/`Unquote` ให้ใช้

```go
package main

import (
	"fmt"
	"strconv"
)

func main() {
	s := "Hello\tWorld\n\"Go\""
	quoted := strconv.Quote(s)
	fmt.Println("Quote:", quoted)

	unquoted, err := strconv.Unquote(quoted)
	fmt.Println("Unquote:", unquoted, err)
	fmt.Println("เท่ากับต้นฉบับ:", unquoted == s)

	_, err2 := strconv.Unquote("ไม่มี quote ครอบ")
	fmt.Println("Unquote string ที่ไม่ถูกต้อง:", err2)
}
```

ผลลัพธ์:

```
Quote: "Hello\tWorld\n\"Go\""
Unquote: Hello	World
"Go" <nil>
เท่ากับต้นฉบับ: true
Unquote string ที่ไม่ถูกต้อง: invalid syntax
```

### `strconv.Quote(s string) string`

แปลง string ธรรมดาให้อยู่ในรูปแบบ **Go string literal ที่ escape ตัวอักษรพิเศษแล้ว** ห่อด้วยเครื่องหมาย `"` ทั้งสองข้าง — `\t` (tab), `\n` (newline), `"` (quote ภายใน) จะถูกแปลงเป็นรูปแบบ escape sequence ที่ปลอดภัย มีประโยชน์มากเวลา log ข้อความที่อาจมีตัวอักษรที่มองไม่เห็น (invisible character) เพื่อ debug ว่าจริงๆ แล้ว string นั้นมีอะไรปนอยู่บ้าง

### `strconv.Unquote(s string) (string, error)`

ทิศทางย้อนกลับ — แปลง string ที่อยู่ในรูปแบบ Go string literal (มี `"` ครอบและมี escape sequence) กลับเป็น string ธรรมดา ถ้า input ไม่ได้อยู่ในรูปแบบที่ถูกต้อง (เช่น ไม่มี `"` ครอบ) จะได้ error กลับมา — ฟังก์ชันนี้มักใช้เวลาอ่านค่าที่ถูก quote ไว้จากไฟล์ config หรือ source code ที่ parse เอง

---

## 7. ข้อผิดพลาดที่พบบ่อยที่สุด: ลืมเช็ค error จาก `Atoi`

นี่คือข้อผิดพลาดที่พบบ่อยที่สุดในโค้ด Go ของมือใหม่เกี่ยวกับ `strconv` โดยเฉพาะ — การใช้ `_` ทิ้ง error จาก `Atoi` ไปเฉยๆ

```go
package main

import (
	"fmt"
	"strconv"
)

func badSum(inputs []string) int {
	total := 0
	for _, s := range inputs {
		n, _ := strconv.Atoi(s) // ไม่เช็ค error -- ถ้า parse ไม่ได้ n จะเป็น 0 เงียบๆ
		total += n
	}
	return total
}

func goodSum(inputs []string) (int, error) {
	total := 0
	for _, s := range inputs {
		n, err := strconv.Atoi(s)
		if err != nil {
			return 0, fmt.Errorf("goodSum: ค่า %q ไม่ใช่ตัวเลข: %w", s, err)
		}
		total += n
	}
	return total, nil
}

func main() {
	inputs := []string{"10", "20", "abc", "30"}

	fmt.Println("badSum (ไม่เช็ค error):", badSum(inputs))

	total, err := goodSum(inputs)
	if err != nil {
		fmt.Println("goodSum พบปัญหา:", err)
	} else {
		fmt.Println("goodSum:", total)
	}
}
```

ผลลัพธ์:

```
badSum (ไม่เช็ค error): 60
goodSum พบปัญหา: goodSum: ค่า "abc" ไม่ใช่ตัวเลข: strconv.Atoi: parsing "abc": invalid syntax
```

สังเกตว่า **`badSum` คืนค่า 60 (10+20+30) โดยไม่มีการแจ้งเตือนใดๆ ว่า `"abc"` ไม่ใช่ตัวเลขและถูกข้ามไปเงียบๆ** (เพราะเมื่อ `Atoi` ล้มเหลว มันคืนค่า `0` มาพร้อมกับ error — ถ้าเราไม่เช็ค error ก็จะบวก `0` เข้าไปในผลรวมโดยไม่รู้ตัวว่ามันคือค่าที่ parse ไม่สำเร็จ ไม่ใช่ `0` ที่ผู้ใช้ตั้งใจกรอกมาจริงๆ) นี่คือ **silent failure ที่อันตรายมาก** เพราะโปรแกรมยังทำงานต่อได้ปกติ แต่ผลลัพธ์ผิดโดยไม่มีใครรู้

เทียบกับ `goodSum` ที่เช็ค error ทุกครั้งตามหลักการจาก Part 015-016 ทำให้รู้ทันทีว่า input ตัวไหนมีปัญหา และหยุดการประมวลผลก่อนที่ผลลัพธ์ที่ผิดจะไหลต่อไปยังส่วนอื่นของระบบ

### กฎเหล็ก: ไม่มีข้อยกเว้นสำหรับการทิ้ง error จาก `strconv`

> **ทุกครั้งที่เรียกฟังก์ชัน parse ของ `strconv` ต้องเช็ค error เสมอ ไม่มีข้อยกเว้น** เพราะ input ที่นำมา parse มักมาจากแหล่งภายนอก (ผู้ใช้, ไฟล์, network, environment variable) ที่เราไม่สามารถควบคุมความถูกต้องได้ 100% แม้แต่กรณีที่ "ดูเหมือน" ปลอดภัย เช่น parse ค่าคงที่ที่ hardcode ไว้ในโค้ดเอง ก็ควรเช็ค error เพื่อความสอดคล้องและป้องกันบั๊กจากการแก้ไขโค้ดในอนาคต

ข้อยกเว้นเดียวที่พอยอมรับได้คือกรณีทดสอบเร็วๆ ใน Go Playground หรือสคริปต์ใช้แล้วทิ้งที่ไม่ใช่โค้ด production — แต่ในโค้ดจริงทุกกรณีต้องเช็ค error เสมอ

---

## สรุปสิ่งที่ได้เรียนในบทนี้

- Go ไม่มีการแปลงชนิดข้อมูลอัตโนมัติระหว่าง `string` กับตัวเลข ต้องใช้แพ็กเกจ **`strconv`** แปลงอย่างชัดเจนเสมอ
- แปลง string เป็นตัวเลข: **`Atoi`** (int ฐานสิบ, ใช้บ่อยที่สุด), **`ParseInt`** (ควบคุมฐานและ bit size), **`ParseFloat`** (เลขทศนิยม), **`ParseBool`** (รับหลายรูปแบบ เช่น `"1"`, `"true"`, `"T"`)
- แปลงตัวเลขกลับเป็น string: **`Itoa`** (คู่ตรงข้าม `Atoi`), **`FormatInt`** (ควบคุมฐาน), **`FormatFloat`** (ควบคุมรูปแบบทศนิยม/scientific notation), **`FormatBool`**
- ทุกฟังก์ชัน parse คืน error ชนิด **`*strconv.NumError`** เสมอเมื่อล้มเหลว ซึ่งมี field `Func`, `Num`, `Err` — ใช้ `errors.As` (จาก Part 016) ดึงรายละเอียดออกมาได้ และเทียบ `Err` กับ **`strconv.ErrSyntax`**/**`strconv.ErrRange`** ด้วย `errors.Is` เพื่อแยกแยะสาเหตุของความล้มเหลว
- แปลงฐานเลขได้ตั้งแต่ 2 ถึง 36 ผ่าน `ParseInt`/`FormatInt` และใช้ `base = 0` ให้ Go เดาฐานจาก prefix (`0x`, `0o`, `0b`) อัตโนมัติ
- **`Quote`/`Unquote`** แปลง string ให้อยู่ในรูปแบบ escape sequence ที่ปลอดภัยและย้อนกลับ มีประโยชน์เวลา debug ข้อความที่มีตัวอักษรพิเศษ
- ข้อผิดพลาดที่พบบ่อยที่สุดคือ**ลืมเช็ค error จาก `Atoi`** ทำให้ input ที่ parse ไม่ได้กลายเป็น `0` แบบเงียบๆ (silent failure) ต้องเช็ค error ทุกครั้งไม่มีข้อยกเว้น

## แบบฝึกหัดท้ายบท

1. เขียนโปรแกรมที่รับ string หลายแบบ (`"123"`, `"-45"`, `"3.14"`, `"abc"`, `""`) มาทดสอบผ่าน `strconv.Atoi` แล้วพิมพ์ผลลัพธ์พร้อม error ของแต่ละตัว สังเกตว่า `Atoi("3.14")` ให้ error อะไร (เพราะ `Atoi` รับเฉพาะจำนวนเต็ม)
2. เขียนฟังก์ชัน `ParsePercentage(s string) (float64, error)` ที่รับ string รูปแบบ `"75.5%"` (มีเครื่องหมาย `%` ต่อท้าย) แล้วคืนค่าทศนิยมที่ตัด `%` ออกแล้ว (ใช้ `strings.TrimSuffix` จาก Part 019 ร่วมกับ `strconv.ParseFloat`) พร้อมจัดการ error ให้ถูกต้อง
3. เขียนโปรแกรมที่ใช้ `errors.As` ดึง `*strconv.NumError` ออกมาจาก error ที่เกิดจาก `strconv.ParseInt` แล้วเขียนเงื่อนไขแยกว่าเป็น `strconv.ErrSyntax` หรือ `strconv.ErrRange` แล้วพิมพ์ข้อความแนะนำผู้ใช้ที่ต่างกันสำหรับแต่ละกรณี
4. เขียนฟังก์ชัน `ToBinaryString(n int64) string` และ `FromBinaryString(s string) (int64, error)` ที่แปลงจำนวนเต็มเป็น/จากสตริงฐานสอง โดยใช้ `strconv.FormatInt`/`ParseInt` กับ `base = 2`
5. เขียนโปรแกรมที่จำลองการอ่านค่า config จาก environment variable (ใช้ `os.Getenv` ซึ่งคืนค่าเป็น string เสมอ) แล้วแปลงเป็น `int` ด้วย `strconv.Atoi` พร้อมกำหนดค่า default หากตัวแปรนั้นไม่ได้ตั้งไว้หรือแปลงไม่สำเร็จ (ห้ามใช้ `_` ทิ้ง error)
6. เปรียบเทียบพฤติกรรมของ `badSum` และ `goodSum` ในหัวข้อ 7 ด้วยตัวเอง โดยเพิ่ม test case ที่ input มีค่าว่าง (`""`) ปนอยู่ อธิบายว่าทำไมการเช็ค error ทุกครั้งถึงสำคัญกว่าที่คิดในระบบที่รับข้อมูลจากผู้ใช้จริง

---

**ต่อไป**: [Part 021 — การจัดรูปแบบด้วย `fmt` เจาะลึก](./021-fmt-deep-dive.md)
