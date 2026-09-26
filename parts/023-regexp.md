# Part 023: Regular Expressions ด้วย `regexp`

> ภาคที่ 2: ระดับกลาง (Intermediate) — ตอนที่ 8 จาก 20 (Part 16–35)

## สารบัญของบทนี้

1. Regular Expression คืออะไร และเมื่อไรควรใช้
2. `regexp.MustCompile` vs `regexp.Compile`
3. ตรวจสอบการ match ด้วย `MatchString`
4. ค้นหาข้อความที่ match ด้วย `FindString`/`FindAllString`
5. Capture Groups ด้วย `FindStringSubmatch`
6. Named Capture Groups และ `SubexpNames`
7. แทนที่ข้อความด้วย `ReplaceAllString`
8. RE2 Engine: ทำไม Go regexp ถึงต่างจาก PCRE
9. ตัวอย่างจริง: ตรวจสอบรูปแบบอีเมลแบบคร่าวๆ และแกะข้อมูลจาก log line
10. เทคนิคเสริม: Case-insensitive, `Split`, และ `QuoteMeta`
11. ข้อควรระวังเรื่อง Performance
12. สรุปสิ่งที่ได้เรียนในบทนี้
13. แบบฝึกหัดท้ายบท

---

## 1. Regular Expression คืออะไร และเมื่อไรควรใช้

**Regular Expression (regex)** คือรูปแบบ (pattern) ที่ใช้อธิบายลักษณะของข้อความ เพื่อใช้ค้นหา ตรวจสอบ หรือแทนที่ข้อความที่ตรงกับรูปแบบนั้น ใน Go มี package มาตรฐานชื่อ `regexp` ที่ implement regex ตามมาตรฐาน **RE2** (จะอธิบายในหัวข้อ 8)

เราเรียนเรื่อง `strings` package ไปแล้วใน **Part 019** ซึ่งเหมาะกับงานค้นหา/แทนที่แบบตรงตัว (literal) เช่น `strings.Contains`, `strings.Replace` แต่เมื่อไรที่ต้องจับ **รูปแบบ** ที่ซับซ้อนกว่านั้น เช่น "ตัวเลข 3 หลักตามด้วยขีด" หรือ "คำที่ขึ้นต้นด้วยตัวพิมพ์ใหญ่" การใช้ `strings` เพียงอย่างเดียวจะทำได้ยากหรือทำไม่ได้เลย นี่คือจุดที่ `regexp` เข้ามาช่วย

**กฎทองในการเลือกใช้**: ถ้างานของคุณคือค้นหา/แทนที่ข้อความแบบตรงตัวชัดเจน ให้ใช้ `strings` เพราะเร็วกว่าและอ่านง่ายกว่า ใช้ `regexp` เฉพาะเมื่อต้องจับ**รูปแบบ**ที่มีความยืดหยุ่น เช่น validate รูปแบบข้อมูล, แกะข้อมูลจากข้อความที่มีโครงสร้างกึ่งตายตัว (semi-structured text) เช่น log file

---

## 2. `regexp.MustCompile` vs `regexp.Compile`

ก่อนใช้งาน regex ใดๆ ต้อง **compile** pattern ให้เป็น object `*regexp.Regexp` ก่อนเสมอ ซึ่งทำได้ 2 วิธี:

### `regexp.Compile` — คืน error เมื่อ pattern ผิดรูปแบบ

```go
re, err := regexp.Compile(`\d+`)
if err != nil {
	// จัดการ error ตามปกติ (หลักการจาก Part 015)
	log.Fatal(err)
}
```

### `regexp.MustCompile` — panic ทันทีถ้า pattern ผิด (ไม่คืน error)

```go
re := regexp.MustCompile(`\d+`)
```

`MustCompile` เป็นรูปแบบที่ใช้กันมากที่สุดในทางปฏิบัติ เพราะ pattern ส่วนใหญ่เป็นค่าคงที่ที่เขียนตายตัวในโค้ด (ไม่ได้มาจาก input ผู้ใช้) — ถ้า pattern ผิดพลาด นั่นคือ **bug ของโปรแกรมเมอร์** ไม่ใช่ error ที่ควรจัดการตอน runtime จึงเหมาะจะให้ panic ทันทีตอนโปรแกรมเริ่มทำงาน (fail fast) เพื่อให้เจอ bug เร็วที่สุดตอนพัฒนา

**กฎการเลือกใช้**: ถ้า pattern เป็นค่าคงที่ที่เขียนในโค้ดเอง → ใช้ `MustCompile` ถ้า pattern มาจากภายนอก (เช่น ผู้ใช้พิมพ์ pattern เอง, อ่านจาก config file) → ต้องใช้ `Compile` แล้วเช็ค `err` เสมอ เพราะเราไม่สามารถควบคุมได้ว่า input จะถูกต้องเสมอไป

```go
package main

import (
	"fmt"
	"regexp"
)

func main() {
	re := regexp.MustCompile(`\d+`)
	fmt.Println(re.MatchString("มีของ 42 ชิ้น"))
	fmt.Println(re.MatchString("ไม่มีตัวเลข"))

	fmt.Println(re.FindString("ราคา 150 บาท ลด 20 บาท"))
	fmt.Println(re.FindAllString("ราคา 150 บาท ลด 20 บาท", -1))

	re2, err := regexp.Compile(`[`) // pattern ผิดรูปแบบโดยตั้งใจ
	fmt.Println(re2, err)
}
```

ผลลัพธ์:

```
true
false
150
[150 20]
<nil> error parsing regexp: missing closing ]: `[`
```

**ข้อสังเกตสำคัญเรื่อง performance**: การ `Compile`/`MustCompile` มีต้นทุนในการประมวลผล pattern ค่อนข้างสูงเมื่อเทียบกับการ match ตัวมันเอง จึงควร **compile เพียงครั้งเดียว** แล้วนำ `*regexp.Regexp` ที่ได้ไปใช้ซ้ำ (reuse) — เรื่องนี้สำคัญมากพอจะพูดถึงอีกครั้งในหัวข้อ 10

---

## 3. ตรวจสอบการ match ด้วย `MatchString`

`MatchString` คือ method ที่ใช้บ่อยที่สุด — คืนค่า `bool` ว่า string นั้น**มีส่วนใดส่วนหนึ่งที่ตรงกับ pattern หรือไม่** (ไม่จำเป็นต้อง match ทั้ง string เว้นแต่จะใส่ `^` และ `$` กำกับหัวท้าย)

```go
re := regexp.MustCompile(`\d+`)
re.MatchString("มีของ 42 ชิ้น")   // true — มีตัวเลขอยู่ในข้อความ
re.MatchString("ไม่มีตัวเลข")     // false
```

ถ้าต้องการเช็คว่า **string ทั้งหมด** ตรงกับ pattern (ไม่ใช่แค่บางส่วน) ต้องใส่ `^` (จุดเริ่มต้น) และ `$` (จุดสิ้นสุด) ครอบ pattern ไว้ ดังจะเห็นในตัวอย่างการ validate email ในหัวข้อ 9

ยังมีฟังก์ชันระดับ package-level `regexp.MatchString(pattern, s string)` ที่ compile และ match ในคำสั่งเดียว แต่ **ไม่แนะนำให้ใช้ในโค้ดที่ทำงานซ้ำๆ** (เช่นใน loop) เพราะจะ compile pattern ใหม่ทุกครั้งที่เรียก สิ้นเปลืองโดยไม่จำเป็น

### สัญลักษณ์ regex พื้นฐานที่ใช้บ่อย

| สัญลักษณ์ | ความหมาย |
|---|---|
| `.` | ตัวอักษรใดก็ได้ (ยกเว้น newline) |
| `\d` | ตัวเลข (0-9) |
| `\w` | ตัวอักษร, ตัวเลข, หรือ underscore |
| `\s` | whitespace (ช่องว่าง, tab, newline) |
| `+` | หนึ่งตัวขึ้นไป |
| `*` | ศูนย์ตัวขึ้นไป |
| `?` | ศูนย์หรือหนึ่งตัว |
| `{n,m}` | ระหว่าง n ถึง m ตัว |
| `^` | จุดเริ่มต้นของ string (หรือบรรทัด ถ้าเปิด multi-line flag) |
| `$` | จุดสิ้นสุดของ string (หรือบรรทัด) |
| `[abc]` | ตัวอักษรตัวใดตัวหนึ่งใน `a`, `b`, `c` |
| `[^abc]` | ตัวอักษรใดก็ได้ที่ไม่ใช่ `a`, `b`, `c` |
| `(...)` | Capture group |
| `\|` | หรือ (alternation) |

---

## 4. ค้นหาข้อความที่ match ด้วย `FindString`/`FindAllString`

### `FindString` — หา match แรกที่เจอ

```go
re := regexp.MustCompile(`\d+`)
result := re.FindString("ราคา 150 บาท ลด 20 บาท")
fmt.Println(result) // "150" — เจอแค่ตัวแรก
```

ถ้าไม่เจอ match เลย `FindString` จะคืนค่า string ว่าง `""`

### `FindAllString` — หาทุก match ที่เจอ

```go
results := re.FindAllString("ราคา 150 บาท ลด 20 บาท", -1)
fmt.Println(results) // [150 20]
```

Argument ตัวที่สองของ `FindAllString` คือ **จำนวน match สูงสุด** ที่ต้องการ — ใส่ `-1` เพื่อบอกว่าต้องการทุก match ที่เจอ ไม่จำกัดจำนวน ถ้าใส่ `2` จะได้ผลลัพธ์แค่ 2 อันแรก แม้จะมีมากกว่านั้นในข้อความ

รันทั้งสองตัวอย่างรวมกัน:

```
true
false
150
[150 20]
```

**ตระกูลฟังก์ชัน Find ทั้งหมดที่ควรรู้จัก**: นอกจาก `FindString`/`FindAllString` ยังมี `Find` (คืน `[]byte` แทน `string`), `FindIndex` (คืนตำแหน่ง index แทนตัว string), `FindStringIndex`, `FindAllStringIndex` — ใช้เมื่อต้องการรู้**ตำแหน่ง**ของ match ไม่ใช่แค่เนื้อหา เช่น ใช้ตัด string ต่อเอง

---

## 5. Capture Groups ด้วย `FindStringSubmatch`

**Capture Group** คือการใส่วงเล็บ `(...)` ล้อมส่วนหนึ่งของ pattern ไว้ เพื่อ "ดึง" เฉพาะส่วนนั้นออกมาต่างหาก แทนที่จะได้ทั้งก้อนที่ match

```go
re := regexp.MustCompile(`(\d{4})-(\d{2})-(\d{2})`)
match := re.FindStringSubmatch("วันที่สร้าง: 2024-03-15 เวลา 10:00")
fmt.Println(match)
if len(match) == 4 {
	fmt.Println("ปี:", match[1], "เดือน:", match[2], "วัน:", match[3])
}
```

ผลลัพธ์:

```
[2024-03-15 2024 03 15]
ปี: 2024 เดือน: 03 วัน: 15
```

**สิ่งสำคัญที่ต้องจำ**: `FindStringSubmatch` คืนค่าเป็น `[]string` โดย **index 0 คือข้อความทั้งหมดที่ match** (เหมือนไม่มี capture group เลย) ส่วน **index 1 เป็นต้นไปคือค่าของแต่ละ capture group ตามลำดับวงเล็บเปิด** ถ้าไม่มี match เลยจะได้ `nil` กลับมา — ต้องเช็ค `nil` หรือความยาวก่อนเข้าถึง index เสมอ ไม่งั้นจะเกิด panic แบบ index out of range (ทบทวนเรื่อง slice จาก **Part 006**)

---

## 6. Named Capture Groups และ `SubexpNames`

การอ้างอิง capture group ด้วยตัวเลข index (`match[1]`, `match[2]`) มีปัญหาเรื่องการอ่านโค้ด เพราะไม่รู้ว่า index ไหนคือข้อมูลอะไรถ้าไม่ไปดู pattern ประกอบ Go จึงรองรับ **named capture group** ด้วย syntax `(?P<name>...)`:

```go
reNamed := regexp.MustCompile(`(?P<year>\d{4})-(?P<month>\d{2})-(?P<day>\d{2})`)
m := reNamed.FindStringSubmatch("2024-03-15")
names := reNamed.SubexpNames()
fmt.Println("names:", names)

result := make(map[string]string)
for i, name := range names {
	if i != 0 && name != "" {
		result[name] = m[i]
	}
}
fmt.Println(result)
```

ผลลัพธ์:

```
names: [ year month day]
map[day:15 month:03 year:2024]
```

`SubexpNames()` คืน `[]string` ที่มีความยาวเท่ากับจำนวน group ทั้งหมด (รวม index 0) — **index 0 จะเป็น string ว่างเสมอ** (เพราะ group ทั้งหมดของ string ไม่มีชื่อ) ส่วน group ที่ไม่ได้ตั้งชื่อ (ใช้ `(...)` ธรรมดา) ก็จะเป็น string ว่างเช่นกัน จึงต้องเช็ค `name != ""` ก่อนใช้งานเสมอ ตามที่แสดงในโค้ดด้านบน

รูปแบบนี้มีประโยชน์มากเวลาแกะข้อมูลจาก log line ที่มีหลาย field ดังจะเห็นในหัวข้อ 9

---

## 7. แทนที่ข้อความด้วย `ReplaceAllString`

`ReplaceAllString` ใช้แทนที่ **ทุกส่วนที่ match** ด้วยข้อความใหม่:

```go
reSpace := regexp.MustCompile(`\s+`)
cleaned := reSpace.ReplaceAllString("มี   ช่องว่าง    เยอะมาก", " ")
fmt.Println(cleaned) // "มี ช่องว่าง เยอะมาก"
```

ตัวอย่างนี้เป็นเทคนิคยอดนิยมสำหรับ **normalize ช่องว่างซ้ำให้เหลือช่องเดียว** — สังเกตว่า `\s+` (หนึ่งตัวขึ้นไป) จับกลุ่มช่องว่างติดกันทั้งหมดมาแทนที่ในครั้งเดียว

### ใช้ Capture Group ใน Replacement String (Backreference)

Replacement string สามารถอ้างอิงกลับไปยัง capture group ด้วย `$1`, `$2`, ... ได้:

```go
reDate := regexp.MustCompile(`(\d{4})-(\d{2})-(\d{2})`)
swapped := reDate.ReplaceAllString("2024-03-15", "$3/$2/$1")
fmt.Println(swapped) // "15/03/2024"
```

ผลลัพธ์ทั้งสองตัวอย่าง:

```
มี ช่องว่าง เยอะมาก
15/03/2024
```

ตัวอย่างนี้แปลงรูปแบบวันที่จาก `YYYY-MM-DD` เป็น `DD/MM/YYYY` ได้ในบรรทัดเดียว โดยใช้ `$1`, `$2`, `$3` อ้างอิงกลับไปยัง group ปี/เดือน/วันที่จับไว้ (**ข้อควรระวัง**: ถ้าตามหลัง `$1` เป็นตัวอักษรหรือตัวเลขที่อาจตีความปนกับชื่อ group เช่น `${1}x` ต้องใช้ `${name}` ครอบไว้ให้ชัดเจน)

ฟังก์ชันที่เกี่ยวข้องอื่นๆ: `ReplaceAll` (ทำงานกับ `[]byte`), `ReplaceAllLiteralString` (ไม่ตีความ `$1` เป็น backreference ใช้แทนที่แบบตรงตัว), `ReplaceAllStringFunc` (ใช้ฟังก์ชัน callback กำหนด logic การแทนที่แบบซับซ้อนได้เอง)

---

## 8. RE2 Engine: ทำไม Go regexp ถึงต่างจาก PCRE

หลายคนที่มาจากภาษาอื่น (Python, JavaScript, PHP) จะคุ้นเคยกับ regex แบบ **PCRE (Perl Compatible Regular Expressions)** ซึ่งรองรับ feature ขั้นสูงมากมาย เช่น **backreference ใน pattern เอง** (`\1` อ้างอิงกลับไปยัง group ที่ match ไปแล้วในตัว pattern), **lookahead/lookbehind** (`(?=...)`, `(?<=...)`)

Go's `regexp` package **ไม่รองรับ feature เหล่านี้** เพราะใช้ engine ที่ชื่อ **RE2** (พัฒนาโดย Russ Cox วิศวกร Go คนสำคัญ) ซึ่งออกแบบมาบนหลักการที่ต่างจาก PCRE โดยสิ้นเชิง:

### ปัญหาของ PCRE: Catastrophic Backtracking

Engine แบบ PCRE ใช้วิธี **backtracking** — เมื่อ pattern ซับซ้อน (เช่นมี `(a+)+b` ที่ nested repetition) และ string ที่ทดสอบไม่ match พอดี engine จะต้อง "ย้อนกลับไปลองใหม่" หลายครั้งซ้อนกัน ในกรณีที่เลวร้ายที่สุด เวลาที่ใช้ประมวลผลอาจเพิ่มขึ้นแบบ **exponential** ตามความยาวของ string จนโปรแกรมค้างไปเป็นวินาทีหรือเป็นนาทีจาก string สั้นๆ เพียงไม่กี่สิบตัวอักษร — ปัญหานี้ร้ายแรงพอจะมีชื่อเรียกเฉพาะว่า **ReDoS (Regular Expression Denial of Service)** และเป็นช่องโหว่ด้านความปลอดภัยที่แท้จริงในหลายระบบที่รับ pattern จากผู้ใช้

### RE2: รับประกัน Linear Time เสมอ

RE2 ถูกออกแบบมาเพื่อ**รับประกันว่าเวลาในการประมวลผลจะแปรผันเป็นเส้นตรง (linear time) ตามความยาวของ input เสมอ** ไม่ว่า pattern จะซับซ้อนแค่ไหน โดยใช้ทฤษฎี **finite automaton** (automata theory) แทนการ backtracking — engine จะติดตามสถานะที่เป็นไปได้ทั้งหมดพร้อมกันในการสแกนผ่าน string เพียงรอบเดียว แทนที่จะลองผิดลองถูกไปมา

ข้อแลกเปลี่ยน (trade-off) คือ RE2 **ไม่รองรับ backreference ในตัว pattern เอง** (เช่น `(\w+)\s\1` ที่ใช้จับคำซ้ำ) และ **ไม่รองรับ lookahead/lookbehind** เพราะ feature เหล่านี้ทำให้ไม่สามารถรับประกัน linear time ได้อีกต่อไป

**สรุปสั้นๆ**: Go เลือกความปลอดภัยและความสามารถคาดเดาประสิทธิภาพได้ (predictability) มากกว่าความสามารถขั้นสูงบางอย่าง ซึ่งสอดคล้องกับปรัชญา "less is more" ของภาษา Go ที่เรียนไปตั้งแต่ **Part 001** — นี่คือเหตุผลที่ทำให้ `regexp` ของ Go ปลอดภัยต่อการนำ pattern จากผู้ใช้ภายนอกมา compile และ run โดยไม่ต้องกลัวปัญหา ReDoS เหมือนภาษาที่ใช้ PCRE

---

## 9. ตัวอย่างจริง: ตรวจสอบรูปแบบอีเมลแบบคร่าวๆ และแกะข้อมูลจาก log line

### ตรวจสอบรูปแบบอีเมลแบบคร่าวๆ (loose validation)

**ข้อควรรู้ก่อน**: การ validate อีเมลด้วย regex ให้ถูกต้อง 100% ตามมาตรฐาน RFC 5322 นั้นซับซ้อนมากจนแทบไม่มีใครทำสมบูรณ์แบบจริง ในทางปฏิบัติเราจึงมักใช้ pattern แบบ **คร่าวๆ (loose)** ที่ครอบคลุมกรณีทั่วไปพอ แล้วค่อยยืนยันตัวตนจริงด้วยการส่งอีเมลยืนยัน (verification email) แทน

```go
package main

import (
	"fmt"
	"regexp"
)

var emailRe = regexp.MustCompile(`^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$`)

func isValidEmailLoose(email string) bool {
	return emailRe.MatchString(email)
}

func main() {
	emails := []string{
		"somchai@example.com",
		"invalid-email",
		"test.user+tag@sub.domain.co.th",
		"@missing-local.com",
		"missing-at.com",
	}
	for _, e := range emails {
		fmt.Printf("%-35s -> %v\n", e, isValidEmailLoose(e))
	}
}
```

ผลลัพธ์:

```
somchai@example.com                 -> true
invalid-email                       -> false
test.user+tag@sub.domain.co.th      -> true
@missing-local.com                  -> false
missing-at.com                      -> false
```

สังเกตว่าเราประกาศ `emailRe` เป็น **package-level variable** ที่ compile ครั้งเดียวตอนโปรแกรมเริ่มทำงาน แล้วนำมาใช้ซ้ำในฟังก์ชัน `isValidEmailLoose` — นี่คือ pattern ที่ถูกต้องตามหลักการ "compile once, reuse" ที่จะอธิบายเพิ่มในหัวข้อ 10

### แกะข้อมูลจาก Log Line ด้วย Named Capture Groups

```go
logLine := `[2024-03-15 10:23:45] ERROR user=somchai action=login message="invalid password"`

logRe := regexp.MustCompile(`^\[(?P<timestamp>[^\]]+)\]\s+(?P<level>\w+)\s+user=(?P<user>\S+)\s+action=(?P<action>\S+)\s+message="(?P<message>[^"]*)"`)

m := logRe.FindStringSubmatch(logLine)
if m == nil {
	fmt.Println("ไม่ match")
	return
}
for i, name := range logRe.SubexpNames() {
	if i != 0 && name != "" {
		fmt.Printf("%s = %s\n", name, m[i])
	}
}
```

ผลลัพธ์:

```
timestamp = 2024-03-15 10:23:45
level = ERROR
user = somchai
action = login
message = invalid password
```

ตัวอย่างนี้แสดงให้เห็นการใช้ regex กับงานจริงที่พบบ่อยมาก — การแกะ (parse) ข้อมูลจาก log ที่มีโครงสร้างกึ่งตายตัว โดยใช้ named group ทำให้โค้ดอ่านง่ายกว่าการอ้าง index ตัวเลขมาก และเมื่อต้อง maintain pattern ในอนาคต (เช่นเพิ่ม field ใหม่) ก็ไม่กระทบ index เดิมที่เคยอ้างอิงไว้

---

## 10. เทคนิคเสริม: Case-insensitive, `Split`, และ `QuoteMeta`

### Case-insensitive matching ด้วย `(?i)`

ใส่ flag `(?i)` ไว้หน้า pattern เพื่อให้ match แบบไม่สนตัวพิมพ์เล็ก-ใหญ่:

```go
reCI := regexp.MustCompile(`(?i)golang`)
fmt.Println(reCI.MatchString("I love GoLang")) // true
fmt.Println(reCI.MatchString("I love Python")) // false
```

### `Split` — แบ่ง string ด้วย pattern แทนตัวคั่นตายตัว

ถ้า `strings.Split` (จาก **Part 019**) ใช้ตัวคั่นตายตัวไม่พอ (เช่น ตัวคั่นมีช่องว่างปนมาไม่แน่นอน) `regexp` มี `Split` ให้ใช้เช่นกัน:

```go
reSplit := regexp.MustCompile(`\s*,\s*`)
parts := reSplit.Split("apple, banana ,cherry,   date", -1)
fmt.Println(parts) // [apple banana cherry date]
```

Pattern `\s*,\s*` จับ comma ที่อาจมีหรือไม่มีช่องว่างล้อมรอบ ทำให้ผลลัพธ์สะอาดโดยไม่ต้อง `strings.TrimSpace` แต่ละตัวเพิ่มเติม

### `regexp.QuoteMeta` — escape ตัวอักษรพิเศษก่อนนำ input ผู้ใช้ไปฝังใน pattern

ถ้าต้องนำข้อความจากผู้ใช้ไปใช้เป็นส่วนหนึ่งของ pattern โดยตรง (เช่น ค้นหาคำที่ผู้ใช้พิมพ์แบบตรงตัว) ต้อง escape ตัวอักษรที่มีความหมายพิเศษใน regex (เช่น `.`, `+`, `?`, `(`, `)`) ก่อนเสมอ ไม่งั้นตัวอักษรเหล่านั้นจะถูกตีความเป็นส่วนหนึ่งของ pattern แทนที่จะเป็นตัวอักษรตรงตัว:

```go
userInput := "1+1=2?"
escaped := regexp.QuoteMeta(userInput)
fmt.Println(escaped) // 1\+1=2\?

re := regexp.MustCompile(escaped)
fmt.Println(re.MatchString("คำถามคือ 1+1=2? ใช่ไหม")) // true
```

`QuoteMeta` เป็นฟังก์ชันสำคัญด้านความปลอดภัยเมื่อต้องรับ pattern บางส่วนจากผู้ใช้ — ถ้าไม่ escape ก่อน ผู้ใช้อาจส่ง input ที่ทำให้ pattern มีความหมายผิดเพี้ยนไปจากที่ตั้งใจ (แม้ RE2 จะไม่มีปัญหา ReDoS แต่ก็ยังมีปัญหาเรื่อง pattern ทำงานผิดจากที่ควรจะเป็นได้)

รันทั้งสามตัวอย่างรวมกันได้ผลลัพธ์:

```
true
false
[apple banana cherry date]
1\+1=2\?
true
```

---

## 11. ข้อควรระวังเรื่อง Performance

### กฎเหล็ก: Compile ครั้งเดียว ใช้ซ้ำเสมอ

```go
// ผิด — compile pattern ใหม่ทุกครั้งที่เรียกฟังก์ชัน สิ้นเปลืองมาก
func isNumberBad(s string) bool {
	re := regexp.MustCompile(`^\d+$`)
	return re.MatchString(s)
}

// ถูก — compile แค่ครั้งเดียวตอนโปรแกรมเริ่มทำงาน
var numberRe = regexp.MustCompile(`^\d+$`)

func isNumberGood(s string) bool {
	return numberRe.MatchString(s)
}
```

`*regexp.Regexp` ที่ compile แล้ว **safe สำหรับใช้งานพร้อมกันจากหลาย goroutine ได้ (concurrent-safe)** โดยไม่ต้องมี lock เพิ่มเติม (เราจะเรียนเรื่อง goroutine และ concurrency-safety แบบเจาะลึกใน **ภาคที่ 3**) ทำให้การประกาศเป็น package-level variable แล้วใช้ซ้ำเป็นแนวทางที่ถูกต้องและปลอดภัยเสมอ ไม่ต้อง compile ใหม่ในแต่ละ request หรือแต่ละ goroutine

### เมื่อไรที่ regex อาจไม่ใช่เครื่องมือที่เหมาะสม

แม้ RE2 จะรับประกัน linear time แต่ regex ก็ยังมีต้นทุนสูงกว่าการเปรียบเทียบ string ตรงๆ เสมอ ถ้างานเป็นการค้นหา/ตรวจสอบแบบตรงตัวไม่ซับซ้อน (เช่น `strings.HasPrefix`, `strings.Contains` จาก **Part 019**) ควรเลือกใช้ `strings` ก่อนเสมอ เพราะเร็วกว่าและสื่อความหมายชัดเจนกว่า เก็บ `regexp` ไว้สำหรับงานที่ต้องจับรูปแบบที่ซับซ้อนจริงๆ

---

## สรุปสิ่งที่ได้เรียนในบทนี้

- `regexp.MustCompile` ใช้กับ pattern คงที่ในโค้ด (panic ถ้าผิด — fail fast) ส่วน `regexp.Compile` ใช้เมื่อ pattern มาจากภายนอกและต้องเช็ค error
- `MatchString` เช็คว่ามี match หรือไม่ (`bool`), `FindString`/`FindAllString` ดึงข้อความที่ match ออกมา
- `FindStringSubmatch` ดึง capture group โดย index 0 คือทั้งก้อน ส่วน index 1+ คือแต่ละ group
- Named capture group (`(?P<name>...)`) ใช้คู่กับ `SubexpNames()` ทำให้โค้ดอ่านง่ายกว่าการอ้าง index ตัวเลข
- `ReplaceAllString` แทนที่ข้อความที่ match ทั้งหมด รองรับ backreference ผ่าน `$1`, `$2` ในการอ้างอิง capture group
- Go's `regexp` ใช้ **RE2 engine** ที่รับประกัน linear time เสมอ ไม่มี catastrophic backtracking แบบ PCRE แต่แลกมาด้วยการไม่รองรับ backreference-in-pattern และ lookahead/lookbehind
- ต้อง **compile pattern เพียงครั้งเดียว** แล้วนำ `*regexp.Regexp` ไปใช้ซ้ำเสมอ (ปลอดภัยสำหรับใช้พร้อมกันหลาย goroutine)
- ใช้ `regexp` เฉพาะเมื่อจำเป็นต้องจับรูปแบบที่ซับซ้อน งานค้นหาแบบตรงตัวควรใช้ `strings` ก่อนเสมอ

## แบบฝึกหัดท้ายบท

1. เขียนฟังก์ชัน `isValidPhoneNumber(s string) bool` ที่ตรวจสอบว่า string เป็นเบอร์โทรศัพท์ไทยรูปแบบ `0XX-XXX-XXXX` หรือไม่ (เช่น `081-234-5678`) โดยใช้ `regexp.MustCompile` เป็น package-level variable
2. เขียนโปรแกรมที่แกะข้อมูล username และ domain ออกจากอีเมล โดยใช้ named capture group `(?P<user>...)` และ `(?P<domain>...)`
3. เขียนฟังก์ชันที่รับข้อความยาวๆ แล้วใช้ `FindAllString` ดึง hashtag ทั้งหมด (คำที่ขึ้นต้นด้วย `#`) ออกมาเป็น `[]string`
4. ทดลองเขียน pattern ที่พยายามใช้ backreference แบบ `(\w+)\s\1` (จับคำซ้ำติดกัน เช่น "the the") แล้วสังเกตว่า `regexp.Compile` จะคืน error อะไร อธิบายว่าทำไม RE2 ถึงไม่รองรับ syntax นี้
5. เขียนโปรแกรมที่ใช้ `ReplaceAllString` แปลงรูปแบบวันที่จาก `DD/MM/YYYY` เป็น `YYYY-MM-DD` โดยใช้ capture group และ backreference
6. อธิบายด้วยคำพูดตัวเอง (เขียนเป็นคอมเมนต์) ว่า Catastrophic Backtracking คืออะไร และทำไมมันถึงเป็นช่องโหว่ความปลอดภัยที่แท้จริงถ้าระบบรับ regex pattern จากผู้ใช้โดยตรงในภาษาที่ใช้ PCRE

---

**ต่อไป**: [Part 024 — File I/O: `os`, `io`, `bufio`](./024-file-io.md)
