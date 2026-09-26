# Part 019: String และแพ็กเกจ `strings`

> ภาคที่ 2: ระดับกลาง (Intermediate) — ตอนที่ 4 จาก 20 (Part 16–35)

## สารบัญของบทนี้

1. String ใน Go คืออะไรกันแน่: immutable byte sequence แบบ UTF-8
2. `len(s)` นับ byte ไม่ใช่ตัวอักษร: ปัญหาที่พบบ่อยกับข้อความไทย
3. วนลูป string ด้วย `range`: ได้ rune ไม่ใช่ byte
4. `strings.Builder`: ทำไม `+` ในลูปถึงช้า และควรใช้อะไรแทน
5. ทัวร์ฟังก์ชันยอดนิยมในแพ็กเกจ `strings`
6. พื้นฐาน `unicode/utf8`: เครื่องมือทำงานกับ rune โดยตรง
7. แนวทางปฏิบัติและข้อผิดพลาดที่พบบ่อย
8. สรุปสิ่งที่ได้เรียนในบทนี้
9. แบบฝึกหัดท้ายบท

---

## 1. String ใน Go คืออะไรกันแน่: immutable byte sequence แบบ UTF-8

ตั้งแต่ Part 003 เราใช้ชนิดข้อมูล `string` มาโดยตลอด แต่ยังไม่เคยเจาะลึกว่าจริงๆ แล้ว string ใน Go มีโครงสร้างภายในเป็นอย่างไร ซึ่งเป็นความรู้ที่**จำเป็นมาก**ก่อนจะเข้าใจพฤติกรรมของแพ็กเกจ `strings` ได้อย่างถูกต้อง

> **ในทางเทคนิค `string` ใน Go คือลำดับของ byte (สิ่งที่แน่นอนคือ read-only slice of bytes) ที่ตามธรรมเนียม (ไม่ใช่ข้อบังคับทางภาษา) มักเก็บข้อความในรูปแบบการเข้ารหัส UTF-8**

สองคุณสมบัติที่สำคัญที่สุดของ string ใน Go:

**1. Immutable (เปลี่ยนแปลงไม่ได้)** — เมื่อสร้าง string ขึ้นมาแล้ว ไม่มีทางแก้ไข byte ใดๆ ภายในได้โดยตรง ทุกครั้งที่เราเขียนโค้ดที่ "ดูเหมือน" จะแก้ไข string (เช่น การต่อ string ด้วย `+`) จริงๆ แล้ว Go จะสร้าง string **ใหม่** ขึ้นมาทั้งก้อนเสมอ ไม่มีการแก้ไขของเดิม

```go
s := "hello"
// s[0] = 'H' // compile error! cannot assign to s[0] (value of type byte)
```

**2. UTF-8 คือ encoding มาตรฐานที่ Go source code และ standard library ใช้** — เมื่อเขียน string literal ในโค้ด เช่น `"สวัสดี"` ตัว compiler จะแปลงเป็นลำดับ byte ตามการเข้ารหัส UTF-8 โดยอัตโนมัติ ซึ่งเป็นสาเหตุที่ทำให้ **ตัวอักษร 1 ตัว อาจใช้พื้นที่มากกว่า 1 byte** โดยเฉพาะตัวอักษรที่ไม่ใช่ ASCII เช่น ภาษาไทย, ภาษาญี่ปุ่น, หรือ emoji

ความเข้าใจสองข้อนี้คือกุญแจสำคัญที่จะอธิบายพฤติกรรมแปลกๆ ที่มือใหม่มักเจอตอนทำงานกับข้อความภาษาไทยใน Go — มาดูรายละเอียดในหัวข้อถัดไป

---

## 2. `len(s)` นับ byte ไม่ใช่ตัวอักษร: ปัญหาที่พบบ่อยกับข้อความไทย

จาก Part 006 เรารู้จัก `len()` ในฐานะฟังก์ชันที่คืนความยาวของ slice/array/map แล้ว สำหรับ string ก็เช่นกัน — **`len(s)` คืนจำนวน byte ที่ string นั้นใช้เก็บข้อมูล ไม่ใช่จำนวนตัวอักษร (character/rune)**

```go
package main

import (
	"fmt"
	"unicode/utf8"
)

func main() {
	s := "สวัสดี"
	fmt.Println("len(s) [จำนวน byte]:", len(s))
	fmt.Println("utf8.RuneCountInString(s) [จำนวน rune]:", utf8.RuneCountInString(s))

	s2 := "Hi 👋"
	fmt.Println()
	fmt.Println("len(s2):", len(s2))
	fmt.Println("RuneCountInString(s2):", utf8.RuneCountInString(s2))
}
```

ผลลัพธ์:

```
len(s) [จำนวน byte]: 18
utf8.RuneCountInString(s) [จำนวน rune]: 6

len(s2): 7
RuneCountInString(s2): 4
```

`"สวัสดี"` มี 6 ตัวอักษรไทย (ส-ว-ั-ส-ด-ี ในทาง Unicode แต่ละตัวคือ 1 code point/rune) แต่ `len(s)` กลับได้ 18 เพราะ**ตัวอักษรไทยแต่ละตัวใน UTF-8 ใช้พื้นที่ 3 byte** (18 ÷ 6 = 3 พอดี) เช่นเดียวกับ `"Hi 👋"` ที่มี 4 ตัวอักษร (`H`, `i`, ` `, `👋`) แต่ `len()` ได้ 7 byte เพราะตัวอักษร ASCII (`H`, `i`, ` `) ใช้ 1 byte ต่อตัว ส่วน emoji `👋` ใช้ถึง 4 byte

### ตารางจำนวน byte ต่อตัวอักษรใน UTF-8

| ช่วงตัวอักษร | จำนวน byte ต่อ 1 rune |
|---|---|
| ASCII (ภาษาอังกฤษ, ตัวเลข, สัญลักษณ์พื้นฐาน) | 1 byte |
| ภาษาไทย, ภาษาละตินเสริม, อักษรพิเศษส่วนใหญ่ | 2-3 byte (ภาษาไทยส่วนใหญ่ใช้ 3 byte) |
| อักษรจีน/ญี่ปุ่น/เกาหลี (CJK) | 3 byte |
| Emoji และ symbol พิเศษบางกลุ่ม | 4 byte |

### ทำไมเรื่องนี้ถึงสำคัญมากสำหรับโปรแกรมเมอร์ไทย

ถ้าเขียนโค้ดที่ต้อง "จำกัดความยาวข้อความไม่เกิน N ตัวอักษร" (เช่น validate ความยาวชื่อผู้ใช้) แล้วใช้ `len(s)` ตรงๆ **ผลลัพธ์จะผิดทันทีเมื่อ input เป็นภาษาไทย** เพราะ `len()` นับ byte ไม่ใช่ตัวอักษรที่ผู้ใช้มองเห็น ต้องใช้ `utf8.RuneCountInString(s)` แทนเสมอเมื่อทำงานกับข้อความที่อาจมีตัวอักษรนอก ASCII

```go
// ผิด! ถ้า s เป็นภาษาไทย จะจำกัดความยาวผิดพลาด (จำกัดตาม byte ไม่ใช่ตัวอักษร)
if len(name) > 20 {
	return errors.New("ชื่อยาวเกินไป")
}

// ถูกต้อง! นับจำนวนตัวอักษรจริง
if utf8.RuneCountInString(name) > 20 {
	return errors.New("ชื่อยาวเกินไป")
}
```

---

## 3. วนลูป string ด้วย `range`: ได้ rune ไม่ใช่ byte

จาก Part 005 เราเรียนรู้การใช้ `for range` กับ slice และ map แล้ว สำหรับ string, `for range` มีพฤติกรรมพิเศษที่ต้องเข้าใจให้แม่นยำ:

> **`for i, r := range s` จะถอดรหัส UTF-8 ให้อัตโนมัติทีละตัวอักษร โดย `r` ที่ได้เป็นชนิด `rune` (คือ `int32` ที่แทน Unicode code point หนึ่งตัว) ส่วน `i` คือตำแหน่ง byte ที่ตัวอักษรนั้นเริ่มต้น (ไม่ใช่ลำดับที่ 0, 1, 2, ... ของตัวอักษร)**

```go
package main

import "fmt"

func main() {
	s := "Aก😀"

	fmt.Println("วนด้วย range (ได้ rune, index เป็นตำแหน่ง byte เริ่มต้น):")
	for i, r := range s {
		fmt.Printf("  index=%d rune=%q (code point U+%04X)\n", i, r, r)
	}

	fmt.Println("\nวนด้วย index ตรงๆ s[i] (ได้ byte ดิบ ไม่ใช่ rune):")
	for i := 0; i < len(s); i++ {
		fmt.Printf("  byte[%d] = 0x%02X\n", i, s[i])
	}
}
```

ผลลัพธ์:

```
วนด้วย range (ได้ rune, index เป็นตำแหน่ง byte เริ่มต้น):
  index=0 rune='A' (code point U+0041)
  index=1 rune='ก' (code point U+0E01)
  index=4 rune='😀' (code point U+1F600)

วนด้วย index ตรงๆ s[i] (ได้ byte ดิบ ไม่ใช่ rune):
  byte[0] = 0x41
  byte[1] = 0xE0
  byte[2] = 0xB8
  byte[3] = 0x81
  byte[4] = 0xF0
  byte[5] = 0x9F
  byte[6] = 0x98
  byte[7] = 0x80
```

สังเกตสิ่งสำคัญมากจากผลลัพธ์นี้:

1. **`index` ที่ `range` ให้มา คือ `0, 1, 4`** ไม่ใช่ `0, 1, 2` — เพราะ `'A'` ใช้ 1 byte (ตำแหน่งถัดไปคือ 1), `'ก'` ใช้ 3 byte (ตำแหน่งถัดไปคือ 1+3=4), ดังนั้นตัวถัดไปจึงเริ่มที่ index 4 พอดี ตำแหน่งนี้คือ**ตำแหน่ง byte เริ่มต้นของตัวอักษรนั้นในข้อความ** ซึ่งมีประโยชน์เวลาต้อง slice string ต่อ (`s[i:]`) แต่**ไม่ใช่ลำดับที่ของตัวอักษร**
2. **การเข้าถึงด้วย `s[i]` ตรงๆ (ไม่ผ่าน `range`) ได้ค่าเป็น `byte` (ชนิด `uint8`) ดิบๆ ไม่ใช่ rune** ถ้า `s[i]` บังเอิญตกอยู่กลางตัวอักษรหลาย byte (เช่น `s[2]` หรือ `s[3]` ในตัวอย่างนี้) จะได้ byte ที่เป็นเพียง "ส่วนหนึ่ง" ของตัวอักษรนั้น ไม่ใช่ตัวอักษรที่สมบูรณ์ — นี่คือกับดักคลาสสิกที่ทำให้โค้ดที่เขียนโดยคิดว่า string เป็น array ของตัวอักษร (เหมือนใน Python หรือ JavaScript) พังทันทีเมื่อเจอข้อความภาษาไทย

### กฎจำง่ายๆ

> **ถ้าต้องการวนดูทีละ "ตัวอักษร" ที่ผู้ใช้มองเห็นจริง ให้ใช้ `for _, r := range s` เสมอ อย่าวนด้วย `for i := 0; i < len(s); i++` แล้วเข้าถึง `s[i]` ตรงๆ เว้นแต่ตั้งใจทำงานกับ byte ดิบจริงๆ (เช่น งานด้าน network protocol หรือ binary data)**

---

## 4. `strings.Builder`: ทำไม `+` ในลูปถึงช้า และควรใช้อะไรแทน

เพราะ string เป็น **immutable** (หัวข้อ 1) ทุกครั้งที่เราเขียน `s = s + "x"` Go จะต้อง**สร้าง string ก้อนใหม่ทั้งหมด** ที่มีความยาวเท่ากับ `s` เดิมบวกกับ `"x"` แล้ว copy เนื้อหาทั้งหมดของ `s` เดิมไปยังก้อนใหม่นี้ ก่อนจะเพิ่ม `"x"` ต่อท้าย — ถ้าทำแบบนี้ในลูปที่รันหลายพันหรือหลายหมื่นรอบ **ต้นทุนรวมจะกลายเป็น O(n²)** เพราะยิ่งลูปวนไปนาน ก้อน string ที่ต้อง copy ซ้ำก็ยิ่งใหญ่ขึ้นเรื่อยๆ

```go
package main

import (
	"fmt"
	"strings"
	"time"
)

func concatWithPlus(n int) string {
	s := ""
	for i := 0; i < n; i++ {
		s += "x" // ทุกครั้งที่ += ต้อง allocate string ใหม่ทั้งก้อน -> O(n^2)
	}
	return s
}

func concatWithBuilder(n int) string {
	var b strings.Builder
	b.Grow(n) // จองความจุล่วงหน้า ลด allocation ซ้ำ (ไม่บังคับ แต่ช่วยได้)
	for i := 0; i < n; i++ {
		b.WriteString("x")
	}
	return b.String()
}

func main() {
	const n = 50000

	start := time.Now()
	r1 := concatWithPlus(n)
	elapsedPlus := time.Since(start)

	start = time.Now()
	r2 := concatWithBuilder(n)
	elapsedBuilder := time.Since(start)

	fmt.Println("ความยาวผลลัพธ์เท่ากัน:", len(r1) == len(r2))
	fmt.Println("เวลาที่ใช้กับ + ในลูป:      ", elapsedPlus)
	fmt.Println("เวลาที่ใช้กับ strings.Builder:", elapsedBuilder)
}
```

ผลลัพธ์ตัวอย่าง (ตัวเลขจริงอาจต่างกันไปตามเครื่อง แต่ลำดับความต่างจะเห็นชัดเสมอ):

```
ความยาวผลลัพธ์เท่ากัน: true
เวลาที่ใช้กับ + ในลูป:       450.049459ms
เวลาที่ใช้กับ strings.Builder: 77.006µs
```

จากผลลัพธ์นี้จะเห็นว่า **`strings.Builder` เร็วกว่า `+=` ในลูปถึงประมาณ 5,800 เท่า** สำหรับการต่อ string 50,000 ครั้ง! และยิ่ง `n` มากขึ้นเท่าไหร่ ความต่างก็จะยิ่งถ่างมากขึ้นเรื่อยๆ ตามลักษณะของ O(n²) เทียบกับ O(n)

### `strings.Builder` แก้ปัญหานี้อย่างไร

`strings.Builder` เก็บข้อมูลภายในเป็น **`[]byte` (byte slice) ที่ขยายขนาดแบบ amortized (เหมือนกลไก append ของ slice ที่เรียนใน Part 006)** แทนที่จะสร้าง string ใหม่ทุกครั้ง มันแค่**เขียนต่อท้าย buffer เดิม** และขยายความจุเป็นบางครั้งเท่านั้น (ไม่ใช่ทุกครั้ง) ทำให้ต้นทุนรวมของการต่อ string ทั้งหมดกลายเป็น **O(n)** แทนที่จะเป็น O(n²)

Method หลักที่ใช้บ่อยของ `strings.Builder`:

```go
var b strings.Builder
b.WriteString("hello")   // เขียน string
b.WriteByte(' ')          // เขียน byte เดียว
b.WriteRune('ก')          // เขียน rune เดียว (รองรับ UTF-8 หลาย byte ให้อัตโนมัติ)
b.Grow(1000)              // จองความจุล่วงหน้า ถ้ารู้ขนาดผลลัพธ์คร่าวๆ (optional แต่ช่วยลด allocation)
result := b.String()      // ดึงผลลัพธ์สุดท้ายออกมาเป็น string
```

### เมื่อไหร่ควรใช้ `strings.Builder` เมื่อไหร่ใช้ `+` ธรรมดาได้

- **ต่อ string จำนวนน้อยครั้ง (ไม่กี่ครั้งในโค้ด ไม่ใช่ในลูป)**: ใช้ `+` หรือ `fmt.Sprintf` ได้ตามปกติ อ่านง่ายกว่า และ compiler ยุคใหม่บางกรณีก็ optimize การต่อ string literal ให้อยู่แล้ว
- **ต่อ string ในลูปที่จำนวนรอบไม่แน่นอนหรืออาจมาก**: ใช้ `strings.Builder` เสมอ เป็นธรรมเนียมมาตรฐานของโค้ด Go คุณภาพดี
- **ต้องการต่อ string จาก slice ที่มีอยู่แล้วด้วย separator**: ใช้ `strings.Join` (หัวข้อ 5) จะสะดวกและอ่านง่ายกว่าเขียนลูปเอง

---

## 5. ทัวร์ฟังก์ชันยอดนิยมในแพ็กเกจ `strings`

แพ็กเกจ `strings` มีฟังก์ชันมากมาย แต่ในทางปฏิบัติมีชุดฟังก์ชันหลักๆ ที่ใช้บ่อยมากในงานประจำวัน มาดูตัวอย่างการใช้งานพร้อมกันในโปรแกรมเดียว

```go
package main

import (
	"fmt"
	"strings"
)

func main() {
	s := "  Hello, Go Programming  "

	fmt.Println("Contains:", strings.Contains(s, "Go"))
	fmt.Println("TrimSpace:", strings.TrimSpace(s))
	fmt.Println("ToUpper:", strings.ToUpper(s))
	fmt.Println("ToLower:", strings.ToLower(s))
	fmt.Println("Replace (n=1):", strings.Replace(s, "o", "0", 1))
	fmt.Println("ReplaceAll:", strings.ReplaceAll(s, "o", "0"))
	fmt.Println("Split by ', ':", strings.Split("a, b, c", ", "))
	fmt.Println("Fields (แยกด้วย whitespace):", strings.Fields("  a   b  c "))
	fmt.Println("Join:", strings.Join([]string{"go", "is", "fun"}, "-"))
	fmt.Println("HasPrefix:", strings.HasPrefix("golang.org", "go"))
	fmt.Println("HasSuffix:", strings.HasSuffix("main.go", ".go"))
	fmt.Println("Index:", strings.Index("chicken", "ken"))
	fmt.Println("Count:", strings.Count("banana", "a"))
	fmt.Println("Repeat:", strings.Repeat("ab", 3))
	fmt.Println("EqualFold:", strings.EqualFold("Go", "GO"))
}
```

ผลลัพธ์:

```
Contains: true
TrimSpace: Hello, Go Programming
ToUpper:   HELLO, GO PROGRAMMING  
ToLower:   hello, go programming  
Replace (n=1):   Hell0, Go Programming  
ReplaceAll:   Hell0, G0 Pr0gramming  
Split by ', ': [a b c]
Fields (แยกด้วย whitespace): [a b c]
Join: go-is-fun
HasPrefix: true
HasSuffix: true
Index: 4
Count: 3
Repeat: ababab
EqualFold: true
```

### สรุปฟังก์ชันแยกตามหมวดหมู่

**ค้นหาและเช็คเงื่อนไข:**

| ฟังก์ชัน | ทำอะไร |
|---|---|
| `strings.Contains(s, substr)` | มี substring นี้อยู่ใน `s` หรือไม่ |
| `strings.HasPrefix(s, prefix)` | `s` ขึ้นต้นด้วย `prefix` หรือไม่ |
| `strings.HasSuffix(s, suffix)` | `s` ลงท้ายด้วย `suffix` หรือไม่ |
| `strings.Index(s, substr)` | ตำแหน่ง byte แรกที่พบ `substr` (คืน `-1` ถ้าไม่พบ) |
| `strings.Count(s, substr)` | นับจำนวนครั้งที่ `substr` ปรากฏใน `s` (ไม่ทับซ้อนกัน) |
| `strings.EqualFold(s1, s2)` | เทียบ string แบบไม่สนตัวพิมพ์ใหญ่เล็ก (case-insensitive) |

**แปลงรูปแบบ:**

| ฟังก์ชัน | ทำอะไร |
|---|---|
| `strings.ToUpper(s)` / `ToLower(s)` | แปลงเป็นตัวใหญ่/เล็กทั้งหมด |
| `strings.TrimSpace(s)` | ตัด whitespace (เว้นวรรค, tab, newline) หัวท้ายออก |
| `strings.Trim(s, cutset)` | ตัดตัวอักษรใน `cutset` ออกจากหัวท้าย |
| `strings.Replace(s, old, new, n)` | แทนที่ `old` ด้วย `new` จำนวน `n` ครั้งแรก (ใส่ `-1` แทนที่ทั้งหมด) |
| `strings.ReplaceAll(s, old, new)` | แทนที่ทั้งหมด (เทียบเท่า `Replace` ที่ `n = -1`) |
| `strings.Repeat(s, count)` | ทำซ้ำ `s` จำนวน `count` ครั้งต่อกัน |

**แยกและรวม:**

| ฟังก์ชัน | ทำอะไร |
|---|---|
| `strings.Split(s, sep)` | แยก `s` ด้วย separator `sep` คืนเป็น `[]string` |
| `strings.Fields(s)` | แยกด้วย whitespace ใดๆ ก็ได้ (เว้นวรรค, tab, newline ต่อเนื่องกี่ตัวก็นับเป็นตัวคั่นเดียว) — สะดวกกว่า `Split` มากเมื่อไม่รู้จำนวนช่องว่างที่แน่นอน |
| `strings.Join(elems, sep)` | รวม `[]string` เข้าด้วยกันด้วย separator `sep` |

### `Split` เทียบกับ `Fields`

ความต่างที่สำคัญระหว่างสองฟังก์ชันนี้ควรจำให้แม่น: `strings.Split("  a   b  c ", " ")` จะได้ผลลัพธ์ที่มี string ว่างปนอยู่มากมาย (เพราะช่องว่างติดกันหลายตัวถูกมองเป็นตัวคั่นหลายตัว) ในขณะที่ `strings.Fields("  a   b  c ")` จะได้ `["a", "b", "c"]` ที่สะอาดกว่ามาก เพราะออกแบบมาเฉพาะสำหรับแยกคำด้วย whitespace โดยเฉพาะ

### ฟังก์ชันที่ทันสมัยกว่า: `strings.Cut`, `strings.Map`, `strings.NewReader`

นอกจากฟังก์ชันคลาสสิกด้านบน แพ็กเกจ `strings` ยังมีเครื่องมือที่มีประโยชน์มากในสถานการณ์เฉพาะทาง:

```go
package main

import (
	"fmt"
	"strings"
)

func main() {
	before, after, found := strings.Cut("user@example.com", "@")
	fmt.Println(before, after, found)

	_, _, found2 := strings.Cut("no-at-sign", "@")
	fmt.Println(found2)

	upper := strings.Map(func(r rune) rune {
		if r >= 'a' && r <= 'z' {
			return r - 32
		}
		return r
	}, "hello world")
	fmt.Println(upper)

	r := strings.NewReader("streaming data")
	buf := make([]byte, 4)
	n, _ := r.Read(buf)
	fmt.Println(n, string(buf[:n]))
}
```

ผลลัพธ์:

```
user example.com true
false
HELLO WORLD
4 stre
```

- **`strings.Cut(s, sep)`** (เพิ่มใน Go 1.18) แยก string ออกเป็นสองส่วนที่ตัวคั่นตัวแรกที่เจอ คืนค่าสามตัว: ส่วนก่อนตัวคั่น, ส่วนหลังตัวคั่น, และ `bool` บอกว่าเจอตัวคั่นหรือไม่ — เหมาะกับกรณีแยกแค่ครั้งเดียว เช่นแยก `key=value` หรือ `user@domain` อ่านง่ายและเร็วกว่า `strings.Split` ที่ต้อง allocate slice เต็มรูปแบบทั้งที่ต้องการแค่สองส่วน
- **`strings.Map(mapping, s)`** แปลงแต่ละ rune ของ `s` ผ่านฟังก์ชันที่กำหนด มีประโยชน์เมื่อ logic การแปลงตัวอักษรซับซ้อนกว่าที่ `ToUpper`/`ToLower` รองรับ (เช่น แปลงเฉพาะบางช่วงตัวอักษร หรือกรองตัวอักษรบางตัวออกโดยคืนค่า `-1` จาก mapping function เพื่อลบตัวอักษรนั้นทิ้ง)
- **`strings.NewReader(s)`** สร้าง `*strings.Reader` ที่ implement interface `io.Reader` (จะเรียนแบบเต็มใน Part 048) ทำให้ส่ง string เข้าไปในฟังก์ชันที่ต้องการ `io.Reader` ได้ทันที เช่นทดสอบโค้ดที่ปกติอ่านจากไฟล์หรือ network โดยไม่ต้องสร้างไฟล์จริง

---

## 6. พื้นฐาน `unicode/utf8`: เครื่องมือทำงานกับ rune โดยตรง

เมื่อจำเป็นต้องควบคุมการทำงานกับ rune อย่างละเอียด (มากกว่าที่ `range` ให้มา) แพ็กเกจ `unicode/utf8` มีฟังก์ชันระดับล่างที่ใช้บ่อย:

```go
package main

import (
	"fmt"
	"unicode/utf8"
)

func main() {
	s := "Go🚀"

	fmt.Println("ValidString:", utf8.ValidString(s))
	fmt.Println("RuneCountInString:", utf8.RuneCountInString(s))

	r, size := utf8.DecodeRuneInString(s[2:])
	fmt.Printf("DecodeRuneInString ที่ byte offset 2: rune=%q size=%d bytes\n", r, size)

	fmt.Println("RuneLen ของ 'ก':", utf8.RuneLen('ก'))
	fmt.Println("RuneLen ของ 'A':", utf8.RuneLen('A'))
}
```

ผลลัพธ์:

```
ValidString: true
RuneCountInString: 3
DecodeRuneInString ที่ byte offset 2: rune='🚀' size=4 bytes
RuneLen ของ 'ก': 3
RuneLen ของ 'A': 1
```

ฟังก์ชันหลักที่ควรรู้จัก:

| ฟังก์ชัน | ทำอะไร |
|---|---|
| `utf8.RuneCountInString(s)` | นับจำนวน rune (ตัวอักษรจริง) ใน string — คือคำตอบที่ถูกต้องสำหรับ "string นี้มีกี่ตัวอักษร" |
| `utf8.ValidString(s)` | เช็คว่า string นี้เป็น UTF-8 ที่ถูกต้องตามรูปแบบหรือไม่ (มีประโยชน์เมื่อรับข้อมูลจากภายนอกที่ไม่น่าเชื่อถือ เช่น ไฟล์ binary ที่อ้างว่าเป็น text) |
| `utf8.DecodeRuneInString(s)` | ถอดรหัส rune **ตัวแรก** ของ string คืนทั้ง rune และจำนวน byte ที่ใช้ — เป็นกลไกเดียวกับที่ `for range` ใช้ภายใน |
| `utf8.RuneLen(r)` | คืนจำนวน byte ที่ rune ตัวนี้จะใช้เมื่อเข้ารหัสเป็น UTF-8 |

ในทางปฏิบัติ เราแทบไม่ค่อยเรียก `utf8.DecodeRuneInString` ตรงๆ บ่อยนัก เพราะ `for range` (หัวข้อ 3) จัดการเรื่องนี้ให้อัตโนมัติอยู่แล้วในกรณีส่วนใหญ่ ฟังก์ชันที่ใช้บ่อยที่สุดในชีวิตจริงคือ **`utf8.RuneCountInString`** สำหรับนับความยาวข้อความที่แสดงผลจริง และ **`utf8.ValidString`** สำหรับตรวจสอบความถูกต้องของ input จากภายนอก

---

## 7. แนวทางปฏิบัติและข้อผิดพลาดที่พบบ่อย

### ข้อผิดพลาดที่ 1: ใช้ `len(s)` เพื่อนับตัวอักษรของข้อความที่อาจมีภาษาไทยหรือ emoji

```go
// ผิด
if len(username) < 3 {
	return errors.New("ชื่อผู้ใช้สั้นเกินไป")
}

// ถูกต้อง
if utf8.RuneCountInString(username) < 3 {
	return errors.New("ชื่อผู้ใช้สั้นเกินไป")
}
```

### ข้อผิดพลาดที่ 2: ตัด string ด้วย index ตรงๆ โดยไม่คำนึงถึงขอบเขตของตัวอักษร

```go
s := "สวัสดี"
// อันตราย! ถ้า n ไม่ตรงกับขอบเขตของ rune พอดี จะได้ string ที่ผิดรูปแบบ UTF-8
truncated := s[:5] // ตัดกลาง byte ของตัวอักษร ผลลัพธ์อ่านไม่ได้ตามที่คาดหวัง
```

การตัด string ให้ปลอดภัยต้องแปลงเป็น `[]rune` ก่อน แล้วค่อยตัดตามจำนวนตัวอักษร:

```go
r := []rune(s)
if len(r) > 3 {
	truncated := string(r[:3]) // ตัดตามจำนวนตัวอักษรจริง ปลอดภัย
	fmt.Println(truncated)
}
```

### ข้อผิดพลาดที่ 3: ต่อ string ในลูปด้วย `+=` โดยไม่รู้ต้นทุน

```go
// ช้ามากถ้า n ใหญ่ (O(n^2))
result := ""
for _, item := range items {
	result += item + ", "
}

// เร็วกว่ามาก (O(n))
var b strings.Builder
for _, item := range items {
	b.WriteString(item)
	b.WriteString(", ")
}
result := b.String()

// หรือถ้าแค่ต้องการรวมด้วย separator ใช้ Join จะสั้นและชัดเจนที่สุด
result := strings.Join(items, ", ")
```

### ข้อผิดพลาดที่ 4: ลืมว่า `strings.Split` กับตัวคั่นที่เป็นช่องว่างซ้ำๆ ให้ผลลัพธ์ไม่สะอาด

```go
parts := strings.Split("a,  b,c", ", ") // แยกด้วย ", " (comma+space) ตรงๆ อาจไม่ครอบคลุมทุกกรณี
```

ถ้าข้อมูลมีช่องว่างไม่สม่ำเสมอ ควร `strings.TrimSpace` แต่ละส่วนหลัง `Split` หรือใช้ package `regexp` (จะเรียนใน Part 023) สำหรับ pattern ที่ซับซ้อนกว่า

### แนวทางปฏิบัติที่ดี — สรุปสั้นๆ

- นับความยาวข้อความที่แสดงผลจริงด้วย `utf8.RuneCountInString` ไม่ใช่ `len()`
- วนดูตัวอักษรด้วย `for _, r := range s` เสมอ ไม่ใช้ index ตรงๆ เว้นแต่ทำงานกับ byte จริงๆ
- ต่อ string จำนวนมากในลูปด้วย `strings.Builder` หรือ `strings.Join`
- ใช้ `strings.Fields` แทน `strings.Split` เมื่อต้องการแยกคำด้วย whitespace ที่ไม่สม่ำเสมอ
- ตรวจสอบ input จากภายนอกด้วย `utf8.ValidString` ก่อนประมวลผลต่อ ถ้าข้อมูลนั้นไม่น่าเชื่อถือ 100%

---

## สรุปสิ่งที่ได้เรียนในบทนี้

- **String ใน Go คือ immutable byte sequence** ที่ตามธรรมเนียมเก็บข้อความแบบ UTF-8 — ทุกการ "แก้ไข" string จริงๆ คือการสร้าง string ใหม่ทั้งก้อนเสมอ
- **`len(s)` นับจำนวน byte ไม่ใช่จำนวนตัวอักษร** ตัวอักษรไทยส่วนใหญ่ใช้ 3 byte ต่อตัว emoji บางตัวใช้ถึง 4 byte ต้องใช้ `utf8.RuneCountInString(s)` เพื่อนับตัวอักษรจริง
- **`for i, r := range s` ถอดรหัส UTF-8 ให้อัตโนมัติ** คืน `rune` และตำแหน่ง byte เริ่มต้น ต่างจากการเข้าถึง `s[i]` ตรงๆ ที่ได้ `byte` ดิบซึ่งอาจเป็นแค่ส่วนหนึ่งของตัวอักษรหลาย byte
- **`strings.Builder`** แก้ปัญหาการต่อ string ในลูปที่ช้าแบบ O(n²) เมื่อใช้ `+=` ให้กลายเป็น O(n) ด้วยการใช้ buffer ภายในที่ขยายแบบ amortized
- ทัวร์ฟังก์ชันสำคัญของ `strings`: `Contains`, `Split`, `Join`, `TrimSpace`, `Replace`/`ReplaceAll`, `ToUpper`/`ToLower`, `Fields`, `HasPrefix`/`HasSuffix`, `Index`, `Count`, `EqualFold`
- **`unicode/utf8`** มีฟังก์ชันระดับล่างสำหรับทำงานกับ rune โดยตรง เช่น `RuneCountInString`, `ValidString`, `DecodeRuneInString`, `RuneLen`

## แบบฝึกหัดท้ายบท

1. เขียนฟังก์ชัน `CharCount(s string) int` ที่คืนจำนวนตัวอักษรที่แสดงผลจริง (ไม่ใช่จำนวน byte) แล้วทดสอบกับข้อความภาษาไทย ภาษาอังกฤษ และข้อความที่มี emoji ปนกัน
2. เขียนฟังก์ชัน `TruncateRunes(s string, maxRunes int) string` ที่ตัดข้อความให้เหลือไม่เกิน `maxRunes` ตัวอักษรอย่างปลอดภัย (ไม่ตัดกลาง byte ของตัวอักษรหลาย byte) โดยแปลงเป็น `[]rune` ก่อนตัด
3. เขียนโปรแกรมที่ต่อ string จำนวน 100,000 ครั้งด้วยสองวิธี (`+=` กับ `strings.Builder`) วัดเวลาด้วย `time.Now()`/`time.Since()` เหมือนตัวอย่างในหัวข้อ 4 แล้วเปรียบเทียบผลลัพธ์บนเครื่องของตัวเอง
4. เขียนฟังก์ชัน `WordCount(s string) int` ที่นับจำนวนคำในประโยคภาษาอังกฤษ (คั่นด้วย whitespace ไม่สม่ำเสมอกี่ตัวก็ได้) โดยใช้ `strings.Fields` แล้วทดสอบกับข้อความที่มีช่องว่างซ้อนกันหลายตัว
5. เขียนฟังก์ชัน `IsValidEmail(s string) bool` แบบง่าย (เช็คว่ามี `@` และมีข้อความอยู่ทั้งก่อนและหลัง `@`) โดยใช้ฟังก์ชันจากแพ็กเกจ `strings` เท่านั้น (`Contains`, `Split`, `Index` เป็นต้น) ไม่ใช้ `regexp`
6. ทดลองเขียนโปรแกรมที่รับ string จากผู้ใช้ แล้วใช้ `utf8.ValidString` ตรวจสอบว่าเป็น UTF-8 ที่ถูกต้องหรือไม่ ก่อนจะประมวลผลต่อด้วย `utf8.RuneCountInString` — อธิบายว่าทำไมการเช็คนี้สำคัญเมื่อรับข้อมูลจากแหล่งภายนอกที่ไม่น่าเชื่อถือ (เช่น ไฟล์ upload หรือ network input)

---

**ต่อไป**: [Part 020 — แพ็กเกจ `strconv`](./020-strconv-package.md)
