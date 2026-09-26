# Part 004: Operators และ Control Flow

> ภาคที่ 1: พื้นฐานภาษา Go (Fundamentals) — ตอนที่ 4 จาก 15

## สารบัญของบทนี้

1. Arithmetic Operators (ตัวดำเนินการทางคณิตศาสตร์)
2. Comparison Operators (ตัวดำเนินการเปรียบเทียบ)
3. Logical Operators (ตัวดำเนินการทางตรรกะ)
4. Bitwise Operators (ตัวดำเนินการระดับบิต)
5. Assignment Operators และ Increment/Decrement
6. ตารางลำดับความสำคัญของ Operator (Precedence)
7. `if`/`else` และรูปแบบ init-statement
8. `switch`: expression switch, tagless switch, fallthrough
9. Type Switch เบื้องต้น (preview เชื่อมโยง Part 014)
10. Labeled break/continue
11. สรุปสิ่งที่ได้เรียนในบทนี้
12. แบบฝึกหัดท้ายบท

---

## 1. Arithmetic Operators (ตัวดำเนินการทางคณิตศาสตร์)

| Operator | ความหมาย | ตัวอย่าง |
|---|---|---|
| `+` | บวก | `a + b` |
| `-` | ลบ | `a - b` |
| `*` | คูณ | `a * b` |
| `/` | หาร | `a / b` |
| `%` | หารเอาเศษ (modulo) | `a % b` |

ข้อควรระวังสำคัญ: **การหารระหว่างจำนวนเต็มจะได้ผลลัพธ์เป็นจำนวนเต็มเสมอ (ปัดเศษทิ้ง ไม่ใช่ปัดเศษ)** ถ้าต้องการผลลัพธ์ทศนิยม ต้องแปลงชนิดข้อมูลเป็น float ก่อน (ตามกฎ explicit conversion ที่เรียนใน Part 003):

```go
a, b := 17, 5
fmt.Println(a+b, a-b, a*b, a/b, a%b)

x, y := 17.0, 5.0
fmt.Println(x / y)
```

ผลลัพธ์:

```
22 12 85 3 2
3.4
```

สังเกตว่า `17 / 5` (integer) ได้ `3` (ตัดเศษทิ้ง) ในขณะที่ `17.0 / 5.0` (float64) ได้ `3.4` ตามที่คาดหวัง — ถ้าเผลอเขียน `a / b` โดยตั้งใจจะได้ทศนิยมแต่ `a`, `b` เป็น `int` ทั้งคู่ จะเป็นบั๊กเงียบที่พบบ่อยมาก

`%` (modulo) ใช้ได้กับจำนวนเต็มเท่านั้น (`int` ทุกขนาด) **ใช้กับ float ไม่ได้** — ถ้าต้องการหาเศษของทศนิยม ต้องใช้ `math.Mod()` แทน

---

## 2. Comparison Operators (ตัวดำเนินการเปรียบเทียบ)

| Operator | ความหมาย |
|---|---|
| `==` | เท่ากับ |
| `!=` | ไม่เท่ากับ |
| `>` | มากกว่า |
| `<` | น้อยกว่า |
| `>=` | มากกว่าหรือเท่ากับ |
| `<=` | น้อยกว่าหรือเท่ากับ |

ผลลัพธ์ของ operator กลุ่มนี้เป็น `bool` เสมอ (`true`/`false`):

```go
a, b := 17, 5
fmt.Println(a == b, a != b, a > b, a < b, a >= b, a <= b)
```

ผลลัพธ์:

```
false true true false true false
```

**ข้อจำกัดสำคัญ**: `==` และ `!=` ใช้เปรียบเทียบได้เฉพาะค่าที่ **comparable** เท่านั้น ตัวเลข, string, bool, pointer, struct (ที่ทุก field comparable), array เปรียบเทียบได้ตรงไปตรงมา แต่ `slice`, `map`, `func` **เปรียบเทียบด้วย `==` ไม่ได้เลย** (ยกเว้นเทียบกับ `nil`) เพราะเป็น reference-like type ที่ compiler ไม่รู้ว่าจะนิยาม "เท่ากัน" อย่างไร — จะเห็นรายละเอียดนี้ชัดเจนขึ้นใน Part 006 (Slices) และ Part 007 (Maps)

---

## 3. Logical Operators (ตัวดำเนินการทางตรรกะ)

| Operator | ความหมาย |
|---|---|
| `&&` | AND (และ) |
| `\|\|` | OR (หรือ) |
| `!` | NOT (ปฏิเสธ) |

```go
isAdult := true
hasLicense := false
fmt.Println(isAdult && hasLicense, isAdult || hasLicense, !isAdult)
```

ผลลัพธ์:

```
false true false
```

Go ใช้ **short-circuit evaluation** เหมือนภาษาส่วนใหญ่: `&&` จะไม่ประเมินฝั่งขวาถ้าฝั่งซ้ายเป็น `false` อยู่แล้ว (รู้ผลแน่นอนว่า `false`) และ `||` จะไม่ประเมินฝั่งขวาถ้าฝั่งซ้ายเป็น `true` อยู่แล้ว พฤติกรรมนี้สำคัญมากในทางปฏิบัติ เพราะทำให้เขียนโค้ดแบบตรวจ `nil` ก่อนเข้าถึงค่าได้อย่างปลอดภัย:

```go
var m map[string]int // nil map
if m != nil && m["key"] > 0 {
	// ปลอดภัย: ถ้า m เป็น nil ฝั่งขวาจะไม่ถูกประเมินเลย
}
```

---

## 4. Bitwise Operators (ตัวดำเนินการระดับบิต)

Go มี bitwise operator ครบตามแบบภาษาตระกูล C ใช้ได้เฉพาะกับจำนวนเต็มเท่านั้น:

| Operator | ความหมาย | ตัวอย่าง |
|---|---|---|
| `&` | Bitwise AND | `p & q` |
| `\|` | Bitwise OR | `p \| q` |
| `^` | Bitwise XOR (เมื่อใช้แบบ binary operator) | `p ^ q` |
| `^` | Bitwise NOT / complement (เมื่อใช้แบบ unary operator นำหน้าตัวเดียว) | `^p` |
| `&^` | AND NOT (bit clear) — เป็น operator เฉพาะของ Go ที่ภาษาอื่นไม่ค่อยมี | `p &^ q` |
| `<<` | Shift left (เลื่อนบิตไปทางซ้าย = คูณด้วย 2 ยกกำลัง n) | `p << 2` |
| `>>` | Shift right (เลื่อนบิตไปทางขวา = หารด้วย 2 ยกกำลัง n) | `p >> 2` |

```go
p, q := 12, 10 // เลขฐาน 2: 1100, 1010
fmt.Println(p&q, p|q, p^q, p&^q, p<<2, p>>2)
```

ผลลัพธ์:

```
8 14 6 4 48 3
```

อธิบายทีละตัว (12 = `1100`, 10 = `1010`):

- `p & q` = `1000` = `8` (AND: บิตที่เป็น 1 ทั้งคู่)
- `p | q` = `1110` = `14` (OR: บิตที่เป็น 1 อย่างน้อยฝั่งใดฝั่งหนึ่ง)
- `p ^ q` = `0110` = `6` (XOR: บิตที่ต่างกัน)
- `p &^ q` = `0100` = `4` (AND NOT: เอาบิตของ `p` แต่ลบบิตที่ `q` เป็น 1 ออก — อ่านว่า "p bit-clear q")
- `p << 2` = `110000` = `48` (เลื่อนซ้าย 2 ตำแหน่ง เทียบเท่า `12 * 2^2`)
- `p >> 2` = `11` = `3` (เลื่อนขวา 2 ตำแหน่ง เทียบเท่า `12 / 2^2` ปัดเศษทิ้ง)

`&^` (AND NOT) เป็น operator ที่มีเฉพาะใน Go มีประโยชน์มากตอนต้องการ "ปิด" บาง bit flag โดยไม่กระทบ flag อื่น เช่น `flags = flags &^ FlagReadOnly` (ปิด flag `FlagReadOnly` โดยไม่แตะ flag อื่น) — เราเห็น `<<` ไปแล้วในรูปแบบ `iota` ของ Part 003 (`KB = 1 << (10 * iota)`) ซึ่งเป็นการใช้ bit shift ในทางปฏิบัติจริง

---

## 5. Assignment Operators และ Increment/Decrement

Go มี compound assignment operator ที่รวมการคำนวณกับการ assign ในตัวเดียว:

| Operator | เทียบเท่ากับ |
|---|---|
| `+=` | `x = x + y` |
| `-=` | `x = x - y` |
| `*=` | `x = x * y` |
| `/=` | `x = x / y` |
| `%=` | `x = x % y` |
| `&=` | `x = x & y` |
| `\|=` | `x = x \| y` |
| `^=` | `x = x ^ y` |
| `<<=` | `x = x << y` |
| `>>=` | `x = x >> y` |

```go
n := 10
n += 5  // 15
n -= 3  // 12
n *= 2  // 24
n /= 4  // 6
n %= 5  // 1
fmt.Println(n)  // 1
```

Go มี `++` และ `--` แต่มีข้อจำกัดที่ต่างจาก C/Java อย่างชัดเจน: **`++`/`--` เป็น statement เท่านั้น ไม่ใช่ expression** พูดง่ายๆ คือใช้เป็นคำสั่งเดี่ยวๆ ได้ (`c++`) แต่ **ใช้ในนิพจน์ไม่ได้เลย**:

```go
c := 0
c++
c++
c--
fmt.Println(c) // 1
```

```go
// x := c++       // ❌ compile error: c++ ไม่ใช่ expression ใช้เป็นค่าไม่ได้
// arr[c++] = 1   // ❌ ก็ผิดเช่นกัน
```

นอกจากนี้ Go มีแค่ `++`/`--` แบบ **postfix เท่านั้น** (`c++`) **ไม่มี prefix แบบ `++c`** เลย — นี่คือหนึ่งในการตัดสินใจ "ลดความกำกวม" ตามปรัชญา simplicity ของ Go ที่กล่าวถึงใน Part 001 (ภาษาอย่าง C ที่มีทั้ง `++c` และ `c++` ซึ่งพฤติกรรมต่างกันเมื่อใช้ในนิพจน์ซับซ้อน เป็นแหล่งบั๊กที่พบบ่อย Go จึงตัดปัญหานี้ทิ้งไปเลยด้วยการไม่ให้เป็น expression)

---

## 6. ตารางลำดับความสำคัญของ Operator (Precedence)

เมื่อมี operator หลายตัวในนิพจน์เดียว Go มีลำดับความสำคัญ (จากสูงไปต่ำ) ดังนี้:

| ระดับ | Operator | หมวดหมู่ |
|---|---|---|
| 5 (สูงสุด) | `*` `/` `%` `<<` `>>` `&` `&^` | คูณ/หาร/shift/bitwise AND |
| 4 | `+` `-` `\|` `^` | บวก/ลบ/bitwise OR/XOR |
| 3 | `==` `!=` `<` `<=` `>` `>=` | เปรียบเทียบ |
| 2 | `&&` | logical AND |
| 1 (ต่ำสุด) | `\|\|` | logical OR |

operator ในระดับเดียวกันจะประมวลผลจากซ้ายไปขวา (left-to-right associativity) ตัวอย่าง:

```go
result := 2 + 3*4    // * มาก่อน + ตามตาราง
result2 := (2 + 3) * 4  // วงเล็บบังคับลำดับใหม่
fmt.Println(result, result2)
```

ผลลัพธ์:

```
14 20
```

> **แนวทางปฏิบัติที่ดี**: แม้จะรู้ลำดับ precedence แม่นแค่ไหน การใส่วงเล็บ `()` เพื่อความชัดเจนในนิพจน์ที่ซับซ้อน (โดยเฉพาะผสม bitwise กับ arithmetic) เป็นสิ่งที่ทีมงานมืออาชีพทำเสมอ เพราะช่วยให้คนอ่านโค้ดไม่ต้องท่องตารางนี้ในหัว

---

## 7. `if`/`else` และรูปแบบ init-statement

โครงสร้าง `if` ใน Go พื้นฐานคล้ายภาษาอื่น แต่มีจุดต่างสำคัญ 2 อย่าง: **ไม่ต้องใส่วงเล็บรอบเงื่อนไข** และ **ต้องใส่ปีกกา `{}` เสมอ** (ต่างจาก C/Java ที่ถ้ามีคำสั่งเดียวจะละ `{}` ได้):

```go
score := 72
if score >= 60 {
	fmt.Println("ผ่าน")
} else {
	fmt.Println("ไม่ผ่าน")
}
```

### รูปแบบ init-statement (จุดเด่นเฉพาะของ Go)

Go อนุญาตให้เขียน **statement เริ่มต้น** ก่อนเงื่อนไขจริง คั่นด้วย `;` ในบรรทัดเดียวกับ `if` ได้ ตัวแปรที่ประกาศในส่วนนี้จะมี **scope จำกัดอยู่แค่ใน `if`/`else if`/`else` block เท่านั้น**:

```go
if v := computeScore(); v > 50 {
	fmt.Println("ผ่าน:", v)
} else {
	fmt.Println("ไม่ผ่าน:", v)
}
// v ไม่สามารถเข้าถึงได้ตรงนี้แล้ว เพราะ scope หมดตั้งแต่ } ปิด if/else
```

รูปแบบนี้เป็นสำนวนที่พบบ่อยที่สุดรูปแบบหนึ่งในโค้ด Go จริง โดยเฉพาะคู่กับการเรียกฟังก์ชันที่คืนค่าพร้อม error (จะเห็นเต็มรูปแบบใน Part 015):

```go
if val, err := someFunction(); err != nil {
	// จัดการ error
} else {
	// ใช้ val
}
```

ข้อดีของรูปแบบนี้คือ **จำกัด scope ของตัวแปรชั่วคราวให้แคบที่สุดเท่าที่จำเป็น** ป้องกันไม่ให้ตัวแปรที่ใช้ครั้งเดียวรั่วไหลไปปนกับโค้ดส่วนอื่นในฟังก์ชัน ซึ่งช่วยลดปัญหา shadowing ที่กล่าวถึงใน Part 003 ได้ทางหนึ่งด้วย

### if-else if-else แบบลูกโซ่

```go
func gradeOf(score int) string {
	if score >= 80 {
		return "A"
	} else if score >= 70 {
		return "B"
	} else if score >= 60 {
		return "C"
	} else {
		return "F"
	}
}
```

รูปแบบนี้ใช้ได้ แต่ Go มี `switch` ที่มักจะอ่านง่ายกว่าเมื่อมีเงื่อนไขหลายชั้น ตามที่จะเห็นในหัวข้อถัดไป

---

## 8. `switch`: expression switch, tagless switch, fallthrough

`switch` ใน Go ยืดหยุ่นกว่าภาษาตระกูล C มาก และเป็นที่นิยมใช้แทน if-else if ยาวๆ

### Expression Switch (แบบมาตรฐาน)

```go
day := 3
switch day {
case 1:
	fmt.Println("จันทร์")
case 2:
	fmt.Println("อังคาร")
case 3:
	fmt.Println("พุธ")
default:
	fmt.Println("วันอื่น")
}
```

จุดต่างสำคัญจาก C/Java: **Go ไม่ fall through ให้อัตโนมัติ** แต่ละ `case` จะ `break` ให้เองโดยปริยายทันทีที่ทำเสร็จ (ไม่ต้องเขียน `break` เอง) นี่คือการตัดสินใจออกแบบที่ลดบั๊ก "ลืมเขียน `break`" ที่เกิดบ่อยมากในภาษาอื่น

`case` หนึ่งอันจับได้หลายค่าพร้อมกันด้วยการคั่นด้วย comma:

```go
switch day {
case 6, 7:
	fmt.Println("วันหยุดสุดสัปดาห์")
default:
	fmt.Println("วันทำงาน")
}
```

### Tagless Switch (switch ไม่มี tag — ทดแทน if-else if ลูกโซ่)

ถ้าไม่ใส่ค่าหลัง `switch` เลย แต่ละ `case` จะเป็นนิพจน์ boolean แบบเดียวกับ `if` — รูปแบบนี้อ่านง่ายกว่า if-else if ยาวๆ มาก:

```go
func gradeOf(score int) string {
	switch {
	case score >= 80:
		return "A"
	case score >= 70:
		return "B"
	case score >= 60:
		return "C"
	default:
		return "F"
	}
}
```

```go
num := 15
switch {
case num < 10:
	fmt.Println("น้อยกว่า 10")
case num < 20:
	fmt.Println("อยู่ระหว่าง 10-19")
default:
	fmt.Println("20 ขึ้นไป")
}
```

ผลลัพธ์:

```
อยู่ระหว่าง 10-19
```

`switch` ก็รองรับ init-statement แบบเดียวกับ `if` ได้เช่นกัน: `switch x := f(); x { ... }`

### `fallthrough`

เนื่องจาก Go ไม่ fall through ให้อัตโนมัติ ถ้าต้องการพฤติกรรมแบบ C (ทำ case ถัดไปต่อ) ต้องระบุ keyword `fallthrough` ที่ท้าย case นั้นอย่างชัดเจน:

```go
switch 2 {
case 1:
	fmt.Println("one")
	fallthrough
case 2:
	fmt.Println("two")
	fallthrough
case 3:
	fmt.Println("three")
case 4:
	fmt.Println("four")
}
```

ผลลัพธ์:

```
two
three
```

สังเกตว่าเริ่มจับที่ `case 2` (ตรงกับค่า `2`) พิมพ์ `"two"` แล้วเจอ `fallthrough` จึง**ทำ case 3 ต่อทันทีโดยไม่เช็คเงื่อนไขของ case 3 เลย** พิมพ์ `"three"` แล้วหยุด (ไม่มี `fallthrough` ต่อจึงไม่ไป `case 4`)

> **ข้อควรระวัง**: `fallthrough` ต้องอยู่เป็นบรรทัดสุดท้ายของ case block เท่านั้น และมันจะกระโดดไปทำ case ถัดไป **โดยไม่สนใจว่าเงื่อนไขของ case ถัดไปจะเป็นจริงหรือไม่** ต่างจาก C ที่ fallthrough เกิดจากการไม่มี `break` (ซึ่งเป็น default behavior) แต่ Go ต้องระบุ `fallthrough` เอง (เป็น opt-in) — การออกแบบนี้ทำให้โค้ด Go ปลอดภัยกว่าเพราะพฤติกรรม fallthrough ต้องตั้งใจเขียนเท่านั้น

---

## 9. Type Switch เบื้องต้น (preview เชื่อมโยง Part 014)

Go มี `switch` อีกรูปแบบหนึ่งที่ใช้ตรวจสอบ **ชนิดข้อมูลจริง (dynamic type)** ของค่าที่เก็บอยู่ใน `interface{}` (หรือ `any` ตั้งแต่ Go 1.18) เรียกว่า **type switch** syntax คือ `switch v := x.(type)`:

```go
func describe(v interface{}) {
	switch val := v.(type) {
	case int:
		fmt.Println("เป็น int ค่า:", val)
	case string:
		fmt.Println("เป็น string ค่า:", val)
	default:
		fmt.Println("ชนิดอื่น ค่า:", val)
	}
}

func main() {
	describe(42)
	describe("hello")
	describe(3.14)
}
```

ผลลัพธ์:

```
เป็น int ค่า: 42
เป็น string ค่า: hello
ชนิดอื่น ค่า: 3.14
```

สังเกตว่าในแต่ละ `case`, ตัวแปร `val` จะถูกมองเป็นชนิดข้อมูลที่ตรงกับ `case` นั้นโดยอัตโนมัติ (ใน `case int` ตัว `val` มีชนิดเป็น `int` จริงๆ ไม่ใช่ `interface{}` อีกต่อไป) ทำให้เรียกใช้งานตาม operation ของชนิดนั้นได้เลยโดยไม่ต้อง cast เพิ่ม

บทนี้แค่แนะนำให้รู้จักรูปแบบผิวเผินเพื่อให้อ่านโค้ดคนอื่นเข้าใจ — เนื้อหาเต็มเรื่อง `interface{}`/`any`, type assertion (`x.(T)`), และ type switch แบบเจาะลึกจะอยู่ใน **Part 013 (Interfaces พื้นฐาน)** และ **Part 014 (Interfaces ขั้นสูง: type assertion, type switch, empty interface)**

---

## 10. Labeled break/continue

Go รองรับ **label** สำหรับ `for`, `switch`, และ `select` ทำให้ `break`/`continue` ควบคุมลูปที่ซ้อนกันหลายชั้นได้อย่างแม่นยำ (ต่างจากภาษาที่ไม่มี label ซึ่ง `break` จะออกจากลูปชั้นในสุดเสมอ)

Label ประกาศด้วยชื่อตามด้วย `:` วางไว้ก่อนคำสั่ง `for` ที่ต้องการอ้างอิงถึง:

```go
func labeledLoopDemo() {
outer:
	for i := 0; i < 3; i++ {
		for j := 0; j < 3; j++ {
			if j == 1 {
				continue outer
			}
			if i == 2 {
				break outer
			}
			fmt.Println("i =", i, "j =", j)
		}
	}
}
```

ผลลัพธ์:

```
i = 0 j = 0
i = 1 j = 0
```

ไล่ตามลอจิก:

1. `i=0, j=0`: ไม่เข้าเงื่อนไขไหน → print `i = 0 j = 0`
2. `i=0, j=1`: เข้าเงื่อนไข `j == 1` → `continue outer` → **ข้ามไปทำ loop `i` รอบถัดไปทันที** (ข้าม `j=2` ของ `i=0` ไปเลย)
3. `i=1, j=0`: print `i = 1 j = 0`
4. `i=1, j=1`: `continue outer` อีกครั้ง → ข้ามไป `i=2`
5. `i=2, j=0`: เข้าเงื่อนไข `i == 2` → `break outer` → **ออกจาก loop ทั้งสองชั้นทันที** ไม่ print อะไรเพิ่ม

ถ้าไม่มี label (`continue`/`break` เฉยๆ) คำสั่งเหล่านี้จะมีผลต่อ**ลูปชั้นในสุดที่ล้อมรอบมันเท่านั้น** เทียบให้เห็นชัด — ถ้าเปลี่ยนจาก `continue outer`/`break outer` เป็น `continue`/`break` เฉยๆ ในตัวอย่างเดียวกัน ผลลัพธ์จะกลายเป็นการข้าม/ออกแค่ loop `j` เท่านั้น ทำให้ `i` เดินหน้าต่อไปครบทั้ง 3 รอบแทน

Label ใช้ชื่อเดียวกับการอ้าง goto ไม่ได้ (ไม่ชนกัน) และตามธรรมเนียม Go นิยมตั้งชื่อ label เป็นตัวพิมพ์เล็กสั้นๆ สื่อความหมาย เช่น `outer:`, `searchLoop:` — เราจะกลับมาใช้ pattern นี้อีกครั้งตอนเขียนลูปซ้อนกันจริงจังใน **Part 005 (Loops)**

---

## สรุปสิ่งที่ได้เรียนในบทนี้

- Arithmetic operator มาตรฐาน (`+ - * / %`) — การหารจำนวนเต็มตัดเศษทิ้งเสมอ ต้อง convert เป็น float ถ้าต้องการผลทศนิยม
- Comparison operator (`== != > < >= <=`) คืนค่า `bool` เสมอ ใช้กับ slice/map/func ตรงๆ ไม่ได้
- Logical operator (`&& || !`) มี short-circuit evaluation ช่วยเขียนโค้ดตรวจ `nil` ก่อนใช้งานได้ปลอดภัย
- Bitwise operator ครบชุด (`& | ^ &^ << >>`) — `&^` (AND NOT) เป็นจุดเด่นเฉพาะของ Go
- Compound assignment (`+= -= *=` ฯลฯ) และ `++`/`--` เป็น **statement เท่านั้น** ไม่ใช่ expression และมีแค่ postfix เท่านั้น
- ตาราง precedence: `* / % << >> & &^` > `+ - | ^` > comparison > `&&` > `||`
- `if`/`else` ไม่ต้องมีวงเล็บรอบเงื่อนไขแต่ต้องมี `{}` เสมอ และรองรับ init-statement เพื่อจำกัด scope ตัวแปรชั่วคราว
- `switch` ไม่ fall through อัตโนมัติ (ต่างจาก C) ต้องใช้ `fallthrough` แบบ opt-in ถ้าต้องการ, มีทั้ง expression switch และ tagless switch (ทดแทน if-else if ลูกโซ่ได้ดี)
- Type switch (`switch v := x.(type)`) ใช้ตรวจ dynamic type ของ `interface{}`/`any` — จะเจาะลึกเต็มรูปแบบใน Part 013-014
- Labeled break/continue ควบคุมลูปซ้อนกันหลายชั้นได้แม่นยำ ต่างจาก break/continue เฉยๆ ที่มีผลแค่ลูปชั้นในสุด

## แบบฝึกหัดท้ายบท

1. เขียนโปรแกรมรับตัวเลขสองจำนวนแล้วพิมพ์ผลลัพธ์จาก operator ทางคณิตศาสตร์ทุกตัว (`+ - * / %`) พร้อมทดสอบด้วยเลขที่หารลงตัวและหารไม่ลงตัว
2. เขียนฟังก์ชันที่ใช้ bitwise operator ตรวจสอบว่าตัวเลขที่รับเข้ามาเป็นเลขคู่หรือเลขคี่ โดยใช้ `&` แทนการใช้ `%2` (คำใบ้: `n & 1`)
3. เขียนฟังก์ชัน `classify(score int) string` โดยใช้ tagless switch แบ่งเกรดเป็น A/B/C/D/F ตามช่วงคะแนน แล้วทดสอบด้วยคะแนนหลายค่ารวมถึงค่าขอบเขต (เช่น พอดี 80, 70, 60)
4. เขียน `switch` ที่มี `fallthrough` อย่างน้อย 2 ระดับ แล้ววาดผังหรืออธิบายเป็นข้อความว่าโปรแกรมจะพิมพ์อะไรออกมาบ้างก่อนรันจริงเพื่อตรวจคำตอบ
5. เขียนลูปซ้อนกัน 3 ชั้นพร้อม label แล้วใช้ `break` และ `continue` แบบมี label เพื่อควบคุมการทำงานของลูปชั้นนอกสุดจากลูปชั้นในสุด
6. เขียนฟังก์ชันที่รับ `interface{}` (หรือ `any`) แล้วใช้ type switch แยกพฤติกรรมสำหรับ `int`, `float64`, `bool`, และ `string` อย่างน้อยชนิดละ 1 กรณี

---

**ต่อไป**: [Part 005 — Loops: `for` ทุกรูปแบบ](./005-loops.md)
