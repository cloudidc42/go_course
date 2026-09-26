# Part 021: การจัดรูปแบบด้วย `fmt` เจาะลึก

> ภาคที่ 2: ระดับกลาง (Intermediate) — ตอนที่ 6 จาก 20 (Part 16–35)

## สารบัญของบทนี้

1. ทบทวน: ทำไม `fmt` ถึงสำคัญขนาดนี้
2. ตระกูลฟังก์ชัน Print: `Print`/`Println`/`Printf`
3. ตระกูลฟังก์ชัน Sprint: สร้าง string แทนการพิมพ์
4. ตระกูลฟังก์ชัน Fprint: เขียนไปยัง `io.Writer` ปลายทางใดก็ได้
5. ตาราง Verb ทั้งหมดที่ต้องรู้
6. Width และ Precision: ควบคุมความกว้างและทศนิยม
7. Stringer Interface: ทำให้ type ของเรา "รู้จักพิมพ์ตัวเอง"
8. การอ่าน Input ด้วย Scan/Scanf/Scanln
9. ข้อผิดพลาดที่พบบ่อยเวลาใช้ `fmt`
10. สรุปสิ่งที่ได้เรียนในบทนี้
11. แบบฝึกหัดท้ายบท

---

## 1. ทบทวน: ทำไม `fmt` ถึงสำคัญขนาดนี้

ตั้งแต่ **Part 001** เราใช้ `fmt.Println("Hello, World!")` มาโดยตลอด และตลอด 20 part ที่ผ่านมาก็แทบไม่มีไฟล์ไหนเลยที่ไม่ import `fmt` สักครั้ง — นี่ไม่ใช่เรื่องบังเอิญ เพราะ `fmt` คือ package ที่ทำหน้าที่ **formatted I/O** (input/output แบบมีรูปแบบ) ซึ่งเป็นสิ่งที่โปรแกรมแทบทุกโปรแกรมต้องใช้ ไม่ว่าจะเป็นการพิมพ์ผลลัพธ์ให้ผู้ใช้ดู, การสร้าง log, การ debug, หรือการอ่าน input จากผู้ใช้

ปัญหาคือที่ผ่านมาเราใช้แค่ `fmt.Println` และ `fmt.Printf` แบบผิวเผิน บทนี้จะพาไปเจาะลึกทุกซอกทุกมุมของ package นี้ ตั้งแต่ verb การจัดรูปแบบที่ครบถ้วน ไปจนถึงกลไกเบื้องหลังที่ทำให้ `fmt` "ฉลาด" พอจะพิมพ์ struct, pointer, slice หรือแม้แต่ type ที่เราสร้างเองได้อย่างสวยงาม

### ภาพรวมฟังก์ชันทั้งหมดใน `fmt`

`fmt` แบ่งฟังก์ชันออกเป็น 3 กลุ่มใหญ่ตามปลายทางที่พิมพ์ไป และแต่ละกลุ่มมี 3 รูปแบบตามวิธีจัดรูปแบบ รวมเป็น 9 ฟังก์ชันหลักที่ใช้บ่อยที่สุด:

| ปลายทาง \ รูปแบบ | ไม่มี format string (`Print`) | เว้นบรรทัดท้าย (`Println`) | ใช้ format verb (`Printf`) |
|---|---|---|---|
| **stdout (หน้าจอ)** | `fmt.Print` | `fmt.Println` | `fmt.Printf` |
| **string (คืนค่ากลับ)** | `fmt.Sprint` | `fmt.Sprintln` | `fmt.Sprintf` |
| **`io.Writer` ใดก็ได้** | `fmt.Fprint` | `fmt.Fprintln` | `fmt.Fprintf` |

นอกจากนี้ยังมีฝั่ง "อ่าน" ที่เป็นกระจกสะท้อนกัน คือ `Scan`, `Scanln`, `Scanf` (อ่านจาก stdin), `Sscan`, `Sscanln`, `Sscanf` (อ่านจาก string), และ `Fscan`, `Fscanln`, `Fscanf` (อ่านจาก `io.Reader` ใดก็ได้) — เราจะพูดถึงฝั่งอ่านในหัวข้อที่ 8

---

## 2. ตระกูลฟังก์ชัน Print: `Print`/`Println`/`Printf`

### `fmt.Print` — พิมพ์แบบไม่มี format string

```go
fmt.Print("Hello", "World")    // HelloWorld (ไม่มีช่องว่างเพราะทั้งคู่เป็น string)
fmt.Print(1, 2, 3)             // 1 2 3 (มีช่องว่างเพราะเป็นตัวเลข ไม่ใช่ string ติดกัน)
```

กฎของ `Print`: **จะเติมช่องว่างระหว่าง argument โดยอัตโนมัติ เมื่อ argument ทั้งสองข้างไม่ใช่ string** ถ้าเป็น string ติดกันจะไม่เติมช่องว่างให้ นี่เป็นจุดที่มือใหม่มักสับสน เพราะพฤติกรรมดูไม่สม่ำเสมอ

### `fmt.Println` — พิมพ์แล้วขึ้นบรรทัดใหม่

```go
fmt.Println("Hello", "World")  // Hello World (มีช่องว่างเสมอ ไม่ว่าจะเป็น string หรือไม่)
fmt.Println(1, 2, 3)           // 1 2 3
```

`Println` ต่างจาก `Print` ตรงที่ **เติมช่องว่างระหว่างทุก argument เสมอ** และ **ขึ้นบรรทัดใหม่ (`\n`) ต่อท้ายเสมอ** — ฟังก์ชันนี้จึงเป็นตัวที่ใช้บ่อยที่สุดในการพิมพ์ debug output แบบเร็วๆ

### `fmt.Printf` — พิมพ์แบบมี format verb (ควบคุมรูปแบบเต็มที่)

```go
name := "Gopher"
age := 15
fmt.Printf("ชื่อ %s อายุ %d ปี\n", name, age)
// ชื่อ Gopher อายุ 15 ปี
```

`Printf` คือฟังก์ชันที่ทรงพลังที่สุดในกลุ่มนี้ เพราะให้เราควบคุมได้ว่า argument แต่ละตัวจะถูกแปลงเป็น string อย่างไร ผ่านสัญลักษณ์ที่เรียกว่า **verb** (ขึ้นต้นด้วย `%`) สังเกตว่า `Printf` **ไม่เติมขึ้นบรรทัดใหม่ให้อัตโนมัติ** ต้องใส่ `\n` เองถ้าต้องการ

---

## 3. ตระกูลฟังก์ชัน Sprint: สร้าง string แทนการพิมพ์

บางครั้งเราไม่ต้องการพิมพ์ทันที แต่ต้องการ "สร้างข้อความ" เก็บไว้ใช้ต่อ เช่น เก็บใน variable, ส่งต่อเป็น error message, หรือเขียนลงไฟล์ — กลุ่ม `Sprint*` ทำหน้าที่นี้ โดยมี logic การจัดรูปแบบเหมือน `Print*` ทุกประการ เพียงแต่ **คืนค่าเป็น `string` แทนที่จะพิมพ์ออกจอ**

```go
msg := fmt.Sprintf("ผู้ใช้ %s ทำรายการสำเร็จ เวลา %d วินาที", "somchai", 3)
fmt.Println(msg) // ใช้ต่อได้ตามปกติ

s1 := fmt.Sprint("a", "b", 1, 2)   // "ab1 2"
s2 := fmt.Sprintln("a", "b")       // "a b\n"
```

`fmt.Sprintf` เป็นฟังก์ชันที่ใช้บ่อยที่สุดในกลุ่มนี้ เพราะมักใช้สร้างข้อความ error แบบมีรายละเอียด (ย้อนไปดู `fmt.Errorf` ใน **Part 016** ซึ่งใช้กลไก verb เดียวกันกับ `Sprintf` ทุกประการ เพียงแต่คืนค่าเป็น `error` แทน `string`)

---

## 4. ตระกูลฟังก์ชัน Fprint: เขียนไปยัง `io.Writer` ปลายทางใดก็ได้

กลุ่มสุดท้ายคือ `Fprint`, `Fprintln`, `Fprintf` ซึ่งรับ argument แรกเป็น `io.Writer` — interface ที่เป็นนามธรรมของ "ที่ที่เขียนข้อมูลออกไปได้" (เราจะเรียนเรื่อง `io.Writer` แบบเจาะลึกใน **Part 048** แต่ตอนนี้ขอให้รู้ว่ามันคือ interface ที่มี method เดียวคือ `Write([]byte) (int, error)`)

```go
package main

import (
	"fmt"
	"os"
)

func main() {
	// เขียนไปที่ stdout ตรงๆ (fmt.Println ก็คือ fmt.Fprintln(os.Stdout, ...) ภายใน)
	fmt.Fprintln(os.Stdout, "พิมพ์ไปที่ stdout")

	// เขียนไปที่ stderr แยกจาก stdout — สำคัญมากสำหรับ log/error message
	fmt.Fprintln(os.Stderr, "นี่คือข้อความ error")

	// เขียนไปที่ strings.Builder (มาจาก Part 019) แล้วดึงผลลัพธ์ทีหลัง
}
```

**ข้อเท็จจริงที่น่าสนใจ**: `fmt.Println(x)` ภายใน implementation จริงๆ แล้วก็คือการเรียก `fmt.Fprintln(os.Stdout, x)` — ทุกฟังก์ชันในกลุ่ม `Print*` เป็นเพียง wrapper ที่ fix ปลายทางเป็น `os.Stdout` ให้เท่านั้น การเข้าใจโครงสร้างนี้ช่วยให้เห็นภาพรวมว่าทำไม `fmt` ถึงออกแบบมาเป็น 3x3 แบบนี้

การใช้ `Fprintln(os.Stderr, ...)` แทน `Println` ธรรมดา เป็นแนวทางที่ดีสำหรับข้อความ error หรือ diagnostic เพราะเมื่อโปรแกรมถูกรันแบบ `myprogram > output.txt` ข้อความที่ไปที่ `stderr` จะยังคงแสดงบนหน้าจอ ไม่ปะปนไปกับผลลัพธ์ที่ redirect ลงไฟล์

---

## 5. ตาราง Verb ทั้งหมดที่ต้องรู้

**Verb** คือสัญลักษณ์ที่ขึ้นต้นด้วย `%` ใน format string ที่บอกว่า argument ตัวถัดไปควรถูกแปลงเป็น string อย่างไร นี่คือตารางสรุปทุก verb ที่ใช้งานจริงบ่อยที่สุด:

### Verb สำหรับใช้ทั่วไป (ทุก type)

| Verb | ความหมาย | ตัวอย่าง |
|---|---|---|
| `%v` | ค่าในรูปแบบ default ตาม type | `{3 4}` (สำหรับ struct) |
| `%+v` | เหมือน `%v` แต่เพิ่มชื่อ field ให้ struct | `{X:3 Y:4}` |
| `%#v` | Go-syntax representation — พิมพ์แบบที่เขียนเป็นโค้ด Go ได้เลย | `main.Point{X:3, Y:4}` |
| `%T` | พิมพ์ชื่อ type ของค่านั้น | `main.Point` |
| `%%` | พิมพ์เครื่องหมาย `%` ตัวจริง (ไม่ใช่ verb) | `%` |

### Verb สำหรับตัวเลข

| Verb | ความหมาย | ตัวอย่าง (input) | ผลลัพธ์ |
|---|---|---|---|
| `%d` | เลขฐาน 10 (integer) | `42` | `42` |
| `%b` | เลขฐาน 2 (binary) | `5` | `101` |
| `%o` | เลขฐาน 8 (octal) | `8` | `10` |
| `%x` | เลขฐาน 16 ตัวพิมพ์เล็ก | `255` | `ff` |
| `%X` | เลขฐาน 16 ตัวพิมพ์ใหญ่ | `255` | `FF` |
| `%f` | ทศนิยม (fixed-point) | `3.14159` | `3.141590` |
| `%e` | เลขทางวิทยาศาสตร์ (ตัว e เล็ก) | `123456.789` | `1.234568e+05` |
| `%g` | เลือกรูปแบบที่กระชับที่สุดระหว่าง `%e`/`%f` อัตโนมัติ | `123456.789` | `123456.789` |
| `%+d` | แสดงเครื่องหมาย `+`/`-` เสมอ | `42` | `+42` |

### Verb สำหรับ string และ byte

| Verb | ความหมาย | ตัวอย่าง (input) | ผลลัพธ์ |
|---|---|---|---|
| `%s` | string ตรงๆ | `"hello"` | `hello` |
| `%q` | string แบบมี quote และ escape (double-quoted Go string) | `"hello"` | `"hello"` |
| `%x` | (กับ string/[]byte) แปลงเป็น hex | `"AB"` | `4142` |
| `%c` | แปลงตัวเลขเป็น character ตาม Unicode code point | `65` | `A` |
| `%U` | Unicode code point แบบ `U+XXXX` | `65` | `U+0041` |

### Verb อื่นๆ ที่ควรรู้

| Verb | ความหมาย |
|---|---|
| `%t` | boolean (`true`/`false`) |
| `%p` | pointer address (แสดงเป็น hex เช่น `0xc000098040`) |

ทดลองรันโค้ดต่อไปนี้เพื่อดูผลลัพธ์จริงของแต่ละ verb:

```go
package main

import "fmt"

type Point struct {
	X, Y int
}

func main() {
	p := Point{X: 3, Y: 4}
	var i int = 42
	var f float64 = 3.14159
	var s string = "hello"
	var b bool = true

	fmt.Printf("%%v  -> %v\n", p)
	fmt.Printf("%%+v -> %+v\n", p)
	fmt.Printf("%%#v -> %#v\n", p)
	fmt.Printf("%%T  -> %T\n", p)
	fmt.Printf("%%d  -> %d\n", i)
	fmt.Printf("%%f  -> %f\n", f)
	fmt.Printf("%%.2f-> %.2f\n", f)
	fmt.Printf("%%s  -> %s\n", s)
	fmt.Printf("%%q  -> %q\n", s)
	fmt.Printf("%%x  -> %x\n", 255)
	fmt.Printf("%%X  -> %X\n", 255)
	fmt.Printf("%%o  -> %o\n", 8)
	fmt.Printf("%%b  -> %b\n", 5)
	fmt.Printf("%%t  -> %t\n", b)
	fmt.Printf("%%c  -> %c\n", 65)
	fmt.Printf("%%p  -> %p\n", &p)
	fmt.Printf("%%e  -> %e\n", 123456.789)
	fmt.Printf("%%g  -> %g\n", 123456.789)
}
```

ผลลัพธ์ (ค่า pointer address ในเครื่องคุณจะไม่เหมือนกันทุกครั้งที่รัน เพราะเป็นตำแหน่งจริงใน memory):

```
%v  -> {3 4}
%+v -> {X:3 Y:4}
%#v -> main.Point{X:3, Y:4}
%T  -> main.Point
%d  -> 42
%f  -> 3.141590
%.2f-> 3.14
%s  -> hello
%q  -> "hello"
%x  -> ff
%X  -> FF
%o  -> 10
%b  -> 101
%t  -> true
%c  -> A
%p  -> 0xc000098040
%e  -> 1.234568e+05
%g  -> 123456.789
```

### `%v` vs `%+v` vs `%#v` บน struct — สรุปให้ชัด

นี่คือจุดที่มือใหม่สับสนบ่อยที่สุด เรียงจาก "ข้อมูลน้อยสุด" ไป "ข้อมูลมากสุด":

1. **`%v`** — พิมพ์แค่ค่า ไม่มีชื่อ field: `{3 4}`
2. **`%+v`** — พิมพ์ค่าพร้อมชื่อ field กำกับ: `{X:3 Y:4}` — เหมาะกับการ debug เพราะอ่านง่ายกว่ามาก
3. **`%#v`** — พิมพ์แบบ Go syntax เต็มรูปแบบ รวมชื่อ type: `main.Point{X:3, Y:4}` — เอาไปแปะเป็นโค้ด Go ได้เลย เหมาะกับตอน debug ที่ต้องการรู้ type ชัดเจนแบบ 100%

กฎง่ายๆ ในการเลือกใช้: **debug ทั่วไปใช้ `%+v`, ต้องการรู้ type แบบชัดเจนสุดใช้ `%#v`, แสดงผลให้ user ทั่วไปดูใช้ `%v` หรือกำหนด `String()` เอง** (ดูหัวข้อ 7)

---

## 6. Width และ Precision: ควบคุมความกว้างและทศนิยม

Verb ทุกตัวสามารถใส่ **width** (ความกว้างขั้นต่ำของ output) และ **precision** (จำนวนทศนิยม หรือจำนวนตัวอักษรสูงสุดสำหรับ string) เพิ่มเข้าไปได้ ด้วย syntax:

```
%[flag][width][.precision]verb
```

### ตัวอย่าง Width กับ string และตัวเลข

```go
fmt.Printf("[%10s]\n", "go")     // จัดชิดขวา เติมช่องว่างซ้ายให้ครบ 10 ตัวอักษร
fmt.Printf("[%-10s]\n", "go")    // เครื่องหมาย - แปลว่าจัดชิดซ้าย
fmt.Printf("[%05d]\n", 42)       // เติม 0 ข้างหน้าแทนช่องว่าง จนครบ 5 หลัก
fmt.Printf("[%+d]\n", 42)        // บังคับให้แสดงเครื่องหมาย + เสมอ
```

### ตัวอย่าง Precision กับทศนิยมและ string

```go
fmt.Printf("[%6.2f]\n", 3.14159)   // width 6, ทศนิยม 2 ตำแหน่ง
fmt.Printf("[%-6.2f]\n", 3.14159)  // เหมือนกันแต่จัดชิดซ้าย
fmt.Printf("[%8.3s]\n", "golang")  // width 8, ตัด string เหลือแค่ 3 ตัวอักษรแรก
```

รันจริงได้ผลลัพธ์ดังนี้ (สังเกตช่องว่างในวงเล็บ `[]` ให้ดี):

```
[        go]
[go        ]
[  3.14]
[3.14  ]
[00042]
[+42]
[     gol]
```

**สรุปสัญลักษณ์สำคัญ**:

| สัญลักษณ์ | ความหมาย |
|---|---|
| `%6.2f` | width อย่างน้อย 6 ตัวอักษร, ทศนิยม 2 ตำแหน่ง |
| `%-10s` | width 10, จัดชิดซ้าย (ปกติ default จัดชิดขวา) |
| `%05d` | width 5, เติม `0` แทนช่องว่างข้างหน้า |
| `%+d` | แสดงเครื่องหมาย `+`/`-` เสมอแม้เป็นค่าบวก |
| `%.3s` | ตัด string ให้เหลือความยาวสูงสุด 3 ตัวอักษร |

Width/precision นี้มีประโยชน์มากเวลาต้องพิมพ์ตารางข้อมูลให้คอลัมน์ตรงกัน เช่น รายงานราคาสินค้า หรือ log ที่ต้องอ่านง่าย

```go
items := []struct {
	Name  string
	Price float64
}{
	{"กาแฟ", 45},
	{"ชาเขียว", 40.5},
	{"โกโก้", 55.25},
}

for _, item := range items {
	fmt.Printf("%-10s %8.2f บาท\n", item.Name, item.Price)
}
```

---

## 7. Stringer Interface: ทำให้ type ของเรา "รู้จักพิมพ์ตัวเอง"

จาก **Part 013** และ **Part 014** เรารู้จัก interface กันแล้ว และหนึ่งใน interface มาตรฐานที่สำคัญที่สุดของ Go คือ `fmt.Stringer`:

```go
type Stringer interface {
	String() string
}
```

**กฎสำคัญ**: เมื่อไรก็ตามที่ `fmt` ต้องพิมพ์ค่าด้วย verb `%v`, `%s`, หรือแม้แต่ผ่าน `Print`/`Println` โดยตรง — ถ้า type ของค่านั้น **implement method `String() string`** (นั่นคือ satisfy interface `Stringer`) `fmt` จะ**เรียก `String()` โดยอัตโนมัติ** แล้วใช้ผลลัพธ์ที่ได้แทนการพิมพ์แบบ default

นี่คือกลไกเดียวกับที่ทำให้ `error` (จาก **Part 015**) พิมพ์ข้อความที่อ่านรู้เรื่องได้ทันทีเวลาเราทำ `fmt.Println(err)` — เพราะ `error` interface กำหนดให้มี method `Error() string` และ `fmt` ก็เช็ค interface `error` ด้วยเช่นกัน (ลำดับการเช็คคือ: `Formatter` → `error` (ถ้าต้อง handle error) → `Stringer`)

### ตัวอย่าง: Enum-like type ที่พิมพ์ชื่อแทนตัวเลข

```go
package main

import "fmt"

type Color int

const (
	Red Color = iota
	Green
	Blue
)

func (c Color) String() string {
	switch c {
	case Red:
		return "Red"
	case Green:
		return "Green"
	case Blue:
		return "Blue"
	default:
		return "Unknown"
	}
}

func main() {
	c := Green
	fmt.Println(c)                     // เรียก String() อัตโนมัติ
	fmt.Printf("สี: %v\n", c)           // %v ก็เรียก String() เช่นกัน
	fmt.Printf("สี: %s\n", c)           // %s ก็เรียก String()
	fmt.Printf("สี (raw int ด้วย %%d): %d\n", c) // %d บังคับพิมพ์เป็นตัวเลขดิบ ไม่เรียก String()
}
```

ผลลัพธ์:

```
Green
สี: Green
สี: Green
สี (raw int ด้วย %d): 1
```

สังเกตว่า `Color` มี underlying type เป็น `int` (เรียนเรื่อง custom type จาก underlying type นี้ตั้งแต่ **Part 003**) แต่พอมี `String()` แล้ว `fmt` จะแสดงชื่อสีแทนตัวเลข **ยกเว้น** เมื่อเราบังคับ verb เป็น `%d` ตรงๆ ซึ่งจะข้าม `Stringer` ไปแสดงค่าตัวเลขดิบเสมอ — นี่เป็นเทคนิคที่ใช้กันแพร่หลายมากในการสร้าง enum ที่อ่านง่ายใน Go (เพราะ Go ไม่มี enum type แบบภาษาอื่นโดยตรง)

### ตัวอย่าง: struct ที่กำหนดการแสดงผลของตัวเอง

```go
type Money struct {
	Amount   float64
	Currency string
}

func (m Money) String() string {
	return fmt.Sprintf("%.2f %s", m.Amount, m.Currency)
}

func main() {
	price := Money{Amount: 199.5, Currency: "THB"}
	fmt.Println("ราคา:", price) // ราคา: 199.50 THB
	fmt.Printf("%v\n", price)   // 199.50 THB
	fmt.Printf("%+v\n", price)  // 199.50 THB (ยังเรียก String() เหมือนเดิม!)
}
```

**ข้อควรระวังสำคัญ**: เมื่อ type มี `String()` แล้ว **`%+v` และ `%v` จะเรียก `String()` เหมือนกัน** ไม่ได้แยกไปแสดงชื่อ field ให้แบบ struct ปกติ ถ้าต้องการดูโครงสร้าง field จริงๆ เพื่อ debug ต้องใช้ `%#v` แทน (ซึ่งจะไม่เรียก `String()` แต่พิมพ์ Go-syntax ของ struct ตรงๆ) หรือใช้ `%+v` กับตัวแปรที่ **ไม่ผ่าน interface** เช่น cast กลับไปเป็น struct เปล่า

**ข้อควรระวังอีกจุด**: การเขียน `String()` ที่เรียก `fmt.Sprintf` โดยใช้ `%v` กับตัวมันเองข้างในจะทำให้เกิด **infinite recursion** เพราะ `%v` จะเรียก `String()` วนไปเรื่อยๆ จนกว่า stack จะล้น (stack overflow) — ถ้าจำเป็นต้องอ้างถึงค่า field ดิบข้างใน `String()` ให้ระบุ field ตรงๆ อย่าใช้ `%v` กับ receiver ทั้งก้อน

---

## 8. การอ่าน Input ด้วย Scan/Scanf/Scanln

`fmt` ไม่ได้มีไว้แค่พิมพ์ แต่ยังใช้ **อ่าน** input ได้ด้วย โดยมีฟังก์ชันคู่ขนานกับฝั่งพิมพ์ครบทั้ง 3 ปลายทาง:

| ปลายทางอ่าน | ไม่สนใจรูปแบบ | อ่านทีละบรรทัด | ใช้ format verb |
|---|---|---|---|
| **stdin** | `fmt.Scan` | `fmt.Scanln` | `fmt.Scanf` |
| **string** | `fmt.Sscan` | `fmt.Sscanln` | `fmt.Sscanf` |
| **`io.Reader`** | `fmt.Fscan` | `fmt.Fscanln` | `fmt.Fscanf` |

ทุกฟังก์ชันในกลุ่มนี้รับ **pointer** ไปยัง variable ที่จะเก็บค่า (เหตุผลเดียวกับที่เรียนใน **Part 010**: ฟังก์ชันต้องแก้ไขค่าต้นทางได้ จึงต้องรับ address ผ่าน pointer) และคืนค่าเป็น `(n int, err error)` โดย `n` คือจำนวนค่าที่อ่านสำเร็จ

### `fmt.Scan` — อ่านค่าคั่นด้วยช่องว่าง

```go
package main

import "fmt"

func main() {
	var name string
	var age int
	fmt.Print("กรุณาใส่ชื่อและอายุ: ")
	fmt.Scan(&name, &age)
	fmt.Printf("สวัสดีคุณ %s อายุ %d ปี\n", name, age)
}
```

ถ้ารันโปรแกรมแล้วพิมพ์ `สมชาย 25` แล้วกด Enter จะได้ผลลัพธ์:

```
กรุณาใส่ชื่อและอายุ: สวัสดีคุณ สมชาย อายุ 25 ปี
```

`fmt.Scan` จะอ่านค่าโดยแบ่งด้วย **whitespace** (ช่องว่าง, tab, newline) โดยไม่สนใจว่าจะอยู่บรรทัดเดียวกันหรือคนละบรรทัด ส่วน `fmt.Scanln` จะทำงานคล้ายกันแต่ **หยุดอ่านทันทีที่เจอ newline** — ถ้า argument ยังไม่ครบตอนเจอ newline จะเกิด error

### `fmt.Scanf` — อ่านตาม format string ที่กำหนด

```go
var day, month, year int
fmt.Print("ใส่วันที่ (DD/MM/YYYY): ")
fmt.Scanf("%d/%d/%d", &day, &month, &year)
fmt.Printf("วันที่: %02d เดือน %02d ปี %d\n", day, month, year)
```

`Scanf` มีประโยชน์เมื่อ input มีรูปแบบตายตัว เช่น ต้องมีเครื่องหมาย `/` คั่นระหว่างตัวเลข — ตัวคั่นที่ไม่ใช่ whitespace ใน format string (เช่น `/`) จะถูกใช้ match กับ input ตรงตัว

### `fmt.Sscan`/`fmt.Sscanf` — อ่านจาก string ที่มีอยู่แล้ว (ใช้บ่อยกว่าที่คิด)

ในทางปฏิบัติ เรามักไม่ค่อยอ่านจาก stdin ตรงๆ ในโปรแกรมจริง (เช่น web server) แต่บ่อยครั้งต้อง "แกะ" ข้อมูลจาก string ที่ได้มา (เช่นบรรทัด log, ค่าจาก config) ซึ่งใช้ `Sscan`/`Sscanf` ได้สะดวกมาก:

```go
package main

import "fmt"

func main() {
	var name string
	var age int
	fmt.Sscanf("สมชาย 25", "%s %d", &name, &age)
	fmt.Println("ชื่อ:", name, "อายุ:", age)

	var a, b int
	fmt.Sscan("10 20", &a, &b)
	fmt.Println("ผลรวม:", a+b)
}
```

ผลลัพธ์:

```
ชื่อ: สมชาย อายุ: 25
ผลรวม: 30
```

**ข้อควรจำ**: ทุกฟังก์ชันตระกูล Scan คืนค่า `error` เป็นค่าที่สอง ต้องเช็คเสมอในโค้ด production (ตามหลักการจาก **Part 015**) เพราะ input จากผู้ใช้หรือจากภายนอกไม่มีอะไรรับประกันว่าจะถูกรูปแบบเสมอไป:

```go
n, err := fmt.Scan(&name, &age)
if err != nil {
	fmt.Println("อ่าน input ไม่สำเร็จ:", err)
	return
}
if n != 2 {
	fmt.Println("ใส่ข้อมูลไม่ครบ")
	return
}
```

สำหรับการอ่านทีละบรรทัดแบบยืดหยุ่นกว่านี้ (เช่น อ่านทั้งบรรทัดรวมช่องว่าง หรืออ่านจากไฟล์) เราจะใช้ `bufio.Scanner` ซึ่งจะเรียนแบบเจาะลึกใน **Part 024**

---

## 9. ข้อผิดพลาดที่พบบ่อยเวลาใช้ `fmt`

### ผิดพลาดที่ 1: จำนวน argument ไม่ตรงกับจำนวน verb

```go
fmt.Printf("ชื่อ: %s\n", "สมชาย", "extra")
fmt.Printf("อายุ: %d\n", "ยี่สิบ")
fmt.Printf("%s กับ %s\n", "หนึ่ง")
```

ผลลัพธ์จริง (ไม่ panic แต่พิมพ์ข้อความบอก error แทรกอยู่ใน output เลย):

```
ชื่อ: สมชาย
%!(EXTRA string=extra)อายุ: %!d(string=ยี่สิบ)
หนึ่ง กับ %!s(MISSING)
```

สิ่งที่เกิดขึ้น:

- **argument เกิน**: ได้ `%!(EXTRA type=value)` ต่อท้าย
- **verb ผิด type**: ได้ `%!verb(type=value)` เช่น `%!d(string=ยี่สิบ)` เพราะ `%d` ใช้กับตัวเลขเท่านั้น แต่ส่ง string เข้าไป
- **argument ขาด**: ได้ `%!s(MISSING)` แทนตำแหน่งที่ขาด verb

**จุดสำคัญที่ต้องเข้าใจ**: `fmt.Printf` **ไม่ panic และไม่คืน compile error** เมื่อ argument ไม่ตรงกับ verb (ต่างจากภาษาที่มี format string checking แบบ static เช่น Rust) เพราะ Go เช็คสิ่งเหล่านี้ตอน **runtime** เท่านั้น อย่างไรก็ตาม `go vet` (เครื่องมือที่แนะนำใน **Part 001**) สามารถตรวจจับความผิดพลาดแบบนี้ได้ **ตอน compile time** ถ้ารันด้วย argument ที่เป็น literal ตรงๆ ในโค้ด จึงควรรัน `go vet ./...` เป็นประจำ

```bash
go vet ./...
```

### ผิดพลาดที่ 2: สับสนระหว่าง `%v`, `%+v`, `%#v`

ดังที่อธิบายในหัวข้อ 5 — การใช้ `%v` เวลา debug struct ทำให้มองไม่เห็นว่าค่าไหนคือ field ไหน โดยเฉพาะ struct ที่มี field ประเภทเดียวกันหลายตัว ควรใช้ `%+v` เป็นค่าเริ่มต้นเวลา debug

### ผิดพลาดที่ 3: ลืมว่า `Stringer` เปลี่ยนพฤติกรรม `%v`/`%+v`

ถ้า type มี `String()` แล้วอยากดู field ดิบจริงๆ ต้องใช้ `%#v` เท่านั้น (ตามที่อธิบายไว้ในหัวข้อ 7)

### ผิดพลาดที่ 4: ใช้ `Print`/`Sprint` ผสม string กับค่าที่ไม่ใช่ string โดยไม่ตั้งใจ

```go
fmt.Print("ผลลัพธ์คือ", 42, "หน่วย") // ผลลัพธ์คือ42 หน่วย — เว้นวรรคไม่สม่ำเสมอ!
```

เพราะ `Print` เติมช่องว่างเฉพาะระหว่าง argument ที่ **ทั้งคู่ไม่ใช่ string** พอ `"ผลลัพธ์คือ"` (string) ชนกับ `42` (int) เลยไม่เติมช่องว่าง แต่ `42` ชนกับ `"หน่วย"` (string) ก็ไม่เติมเช่นกัน... ผลที่ได้คือช่องว่างที่ไม่ตรงกับที่คาดหวัง ทางแก้คือใช้ `Printf` กับ format string ที่ชัดเจนแทนเสมอเมื่อผสม string กับตัวเลข:

```go
fmt.Printf("ผลลัพธ์คือ %d หน่วย\n", 42) // ผลลัพธ์คือ 42 หน่วย — ชัดเจน ควบคุมได้เต็มที่
```

### ผิดพลาดที่ 5: ลืมเช็ค error จากฟังก์ชันตระกูล Scan

ตามที่กล่าวในหัวข้อ 8 — การไม่เช็ค error จาก `fmt.Scan` เป็นสาเหตุของ bug ที่ตามหายากในโปรแกรมที่รับ input จากผู้ใช้ เพราะโปรแกรมจะทำงานต่อด้วยค่าที่อาจเป็นค่า zero value (จาก **Part 003**) โดยไม่รู้ตัวว่า input ผิดรูปแบบ

---

## สรุปสิ่งที่ได้เรียนในบทนี้

- `fmt` มีฟังก์ชันพิมพ์ 3 กลุ่มตามปลายทาง (`Print*` → stdout, `Sprint*` → string, `Fprint*` → `io.Writer`) และแต่ละกลุ่มมี 3 รูปแบบ (ไม่มี format / เว้นบรรทัด / มี verb)
- Verb หลักที่ต้องจำ: `%v %+v %#v %T` สำหรับค่าทั่วไป, `%d %f %x %b %o` สำหรับตัวเลข, `%s %q` สำหรับ string, `%t` สำหรับ boolean, `%p` สำหรับ pointer
- Width/precision ควบคุมด้วย `%6.2f`, `%-10s`, `%05d` ฯลฯ ช่วยจัดตารางข้อมูลให้อ่านง่าย
- `fmt.Stringer` (method `String() string`) ทำให้ type ของเรากำหนดวิธีแสดงผลผ่าน `%v`/`%s`/`Println` เองได้ เป็นเทคนิคยอดนิยมสำหรับทำ enum-like type ที่อ่านง่าย
- ระวัง infinite recursion ถ้าเรียก `%v` กับตัวเองข้างใน `String()`
- อ่าน input ได้ด้วย `Scan`/`Scanln`/`Scanf` (จาก stdin) และ `Sscan`/`Sscanf` (จาก string) — ต้องรับ pointer และควรเช็ค error เสมอ
- ข้อผิดพลาดจาก verb ไม่ตรงกับ argument ไม่ทำให้ compile error แต่จะเห็นข้อความเช่น `%!d(string=...)` หรือ `%!(EXTRA ...)` ปนอยู่ใน output ตอน runtime — ใช้ `go vet` ช่วยตรวจจับได้บางส่วน

## แบบฝึกหัดท้ายบท

1. เขียนโปรแกรมที่ประกาศ struct `Student` ที่มี field `Name string`, `Score float64` แล้วลองพิมพ์ด้วย `%v`, `%+v`, `%#v` เปรียบเทียบผลลัพธ์ทั้งสามแบบ พร้อมอธิบายว่าแต่ละแบบเหมาะกับสถานการณ์ไหน
2. เขียน type `Status int` ที่มีค่า `Pending`, `Active`, `Done` (ใช้ `iota` จาก **Part 003**) แล้ว implement `String() string` ให้มันเพื่อพิมพ์ชื่อสถานะแทนตัวเลข ทดสอบด้วย `fmt.Println` และ `fmt.Printf("%d", ...)` ว่าให้ผลลัพธ์ต่างกันอย่างไร
3. เขียนฟังก์ชันที่รับ slice ของราคาสินค้า (`[]float64`) แล้วพิมพ์เป็นตารางโดยจัด column ให้ตรงกันด้วย width/precision (เช่น ชื่อสินค้าชิดซ้าย 15 ตัวอักษร ตามด้วยราคาชิดขวา 10 ตัวอักษร ทศนิยม 2 ตำแหน่ง)
4. ทดลองพิมพ์ `fmt.Printf("%d\n", "hello")` แล้วสังเกตข้อความ error ที่ได้ อธิบายว่าทำไม Go ถึงเลือกจะพิมพ์ข้อความ error แทรกไว้ใน output แทนที่จะ panic หรือ compile error ทันที
5. เขียนโปรแกรมที่ใช้ `fmt.Sscanf` แกะข้อมูลจาก log line รูปแบบ `"[2024-01-15] ERROR: connection timeout"` เพื่อดึงวันที่, ระดับ log, และข้อความ error ออกมาเป็น variable แยกกัน
6. ลองเขียน `String()` ที่มี bug คือเรียก `fmt.Sprintf("%v", m)` โดยที่ `m` คือ receiver ของตัวเอง แล้วสังเกตว่าเกิดอะไรขึ้นเมื่อรัน (เตือน: โปรแกรมอาจค้างหรือ crash เพราะ stack overflow ควรตั้ง timeout หรือเตรียม kill โปรแกรมไว้)

---

**ต่อไป**: [Part 022 — เวลาและแพ็กเกจ `time`](./022-time-package.md)
