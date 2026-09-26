# Part 007: Maps เจาะลึก

> ภาคที่ 1: พื้นฐานภาษา Go (Fundamentals) — ตอนที่ 7 จาก 15

## สารบัญของบทนี้

1. Map คืออะไร และประกาศ map ได้กี่แบบ
2. Zero value ของ map คือ `nil` — และ nil map ทำอะไรได้/ไม่ได้บ้าง
3. Key ของ map ต้องเป็น comparable type
4. อ่านค่าจาก key ที่ไม่มีอยู่ ได้ zero value เสมอ
5. Comma ok idiom — แยกแยะ "ไม่มี key" กับ "ค่าเป็น zero value"
6. ลบ key ด้วย `delete`
7. ลำดับการวน iterate ถูกสุ่มโดยเจตนา
8. Map มีพฤติกรรมแบบ reference — ไม่ copy ตอน assign หรือ pass เข้าฟังก์ชัน
9. Map เป็น Set ด้วย `map[T]struct{}`
10. Nested maps (map ซ้อน map)
11. สรุปสิ่งที่ได้เรียนในบทนี้
12. แบบฝึกหัดท้ายบท

---

## 1. Map คืออะไร และประกาศ map ได้กี่แบบ

**Map** คือโครงสร้างข้อมูลแบบ key-value ที่เก็บคู่ของ "กุญแจ" (key) กับ "ค่า" (value) โดยแต่ละ key ต้องไม่ซ้ำกันในหนึ่ง map เดียว ถ้าเปรียบเทียบกับภาษาอื่น Map ใน Go ก็คือสิ่งที่หลายภาษาเรียกว่า `dict` (Python), `HashMap` (Java), หรือ `object`/`Map` (JavaScript)

ภายในกลไกของ Go implement map ด้วย **hash table** ทำให้การอ่าน/เขียน/ค้นหา key มีความเร็วเฉลี่ยระดับ O(1) ไม่ขึ้นกับจำนวนข้อมูลใน map (ต่างจาก slice ที่การค้นหาค่าต้องวนลูป O(n) ตามที่กล่าวไปใน Part 006)

Type ของ map เขียนในรูปแบบ `map[KeyType]ValueType` เช่น `map[string]int` คือ map ที่ key เป็น string และ value เป็น int

มี 3 วิธีหลักในการสร้าง map:

```go
package main

import "fmt"

func main() {
	// วิธีที่ 1: ประกาศตัวแปรเฉยๆ (จะได้ nil map — พูดถึงในหัวข้อถัดไป)
	var m1 map[string]int

	// วิธีที่ 2: ใช้ make() — สร้าง map ที่พร้อมใช้งานทันที (แนะนำเมื่อยังไม่รู้ค่าตอนสร้าง)
	m2 := make(map[string]int)
	m2["a"] = 1

	// วิธีที่ 3: map literal — กำหนดค่าเริ่มต้นไปพร้อมกับสร้าง
	m3 := map[string]int{
		"x": 1,
		"y": 2,
	}

	fmt.Println(m1, m2, m3)
}
```

ผลลัพธ์:

```
map[] map[a:1] map[x:1 y:2]
```

สังเกตว่า `fmt.Println` พิมพ์ map ออกมาในรูปแบบ `map[key:value key:value ...]` โดย Go เรียง key ตามลำดับ (ใช้ sort) เฉพาะตอน **print** เท่านั้น เพื่อให้ผลลัพธ์ที่พิมพ์ออกมาอ่านง่ายและ reproducible — แต่การเรียงนี้**ไม่เกี่ยวกับลำดับการ iterate จริง** ซึ่งจะอธิบายในหัวข้อที่ 7

`make()` ยังรับ argument ตัวที่สองเป็น "hint" ของขนาดเริ่มต้นได้ เพื่อลดจำนวนครั้งที่ Go ต้องขยาย hash table ภายในเมื่อรู้ล่วงหน้าว่าจะใส่ข้อมูลประมาณเท่าไร (คล้าย capacity ของ slice ที่เรียนไปใน Part 006):

```go
m := make(map[string]int, 100) // hint ว่าจะมีประมาณ 100 entries
```

การใส่/แก้ไขค่าใน map ทำผ่าน syntax `m[key] = value` เหมือนกันทั้งการเพิ่ม key ใหม่และการแก้ไข key ที่มีอยู่แล้ว (ไม่มีการแยก insert/update แบบภาษาอื่น):

```go
m := make(map[string]int)
m["score"] = 10 // เพิ่มใหม่
m["score"] = 20 // แก้ไขค่าเดิม (key เดิม)
fmt.Println(m["score"]) // 20
```

---

## 2. Zero value ของ map คือ `nil` — และ nil map ทำอะไรได้/ไม่ได้บ้าง

ตามหลักการ **zero value** ที่เรียนไปใน Part 003 ว่าทุก type ใน Go มีค่าเริ่มต้นเสมอ สำหรับ map ค่า zero value คือ **`nil`** ไม่ใช่ map ว่างเปล่าแบบ `map[string]int{}`

```go
package main

import "fmt"

func main() {
	var m map[string]int
	fmt.Println(m)          // map[]
	fmt.Println(m == nil)   // true
	fmt.Println(len(m))     // 0
}
```

จุดสำคัญที่สุดที่ทำให้มือใหม่หลายคน panic โดยไม่รู้สาเหตุคือ: **nil map อ่านได้ปกติ แต่เขียนไม่ได้**

```go
var m map[string]int
fmt.Println(m["anything"]) // อ่านได้ ได้ 0 (zero value ของ int) ไม่ panic
```

แต่ถ้าพยายามเขียนค่าลงใน nil map:

```go
var m map[string]int
m["a"] = 1 // panic: assignment to entry in nil map
```

รันแล้วจะได้:

```
panic: assignment to entry in nil map

goroutine 1 [running]:
main.main()
	/path/to/main.go:5 +0x28
exit status 2
```

เหตุผลเชิง design คือ nil map ไม่มี hash table ข้างในให้เขียนลงไปเลย (ตัวแปรมีแค่ pointer ที่ชี้ไป `nil`) ในขณะที่การ**อ่าน** ค่าจาก key ที่ไม่มีอยู่จริงถูกออกแบบให้คืน zero value เสมอ (อธิบายในหัวข้อที่ 4) ไม่ว่า map นั้นจะเป็น nil หรือไม่ก็ตาม ดังนั้นการอ่านจาก nil map จึงปลอดภัยเสมอ

**กฎจำง่ายๆ**: ถ้าจะเขียนค่าลง map ต้องสร้างด้วย `make()` หรือ literal ก่อนเสมอ ห้ามพึ่งพา zero value ของตัวแปร map แบบที่ทำได้กับ slice บางกรณี (เช่น `append` กับ nil slice ทำงานได้ตามที่เรียนใน Part 006 — แต่ map ไม่มีพฤติกรรมแบบนั้น นี่คือความแตกต่างสำคัญระหว่าง slice กับ map ที่ต้องจำให้แม่น)

ตารางเปรียบเทียบ nil map:

| การกระทำ | ทำได้ไหมกับ nil map |
|---|---|
| อ่านค่าด้วย `m[key]` | ได้ — คืน zero value |
| `len(m)` | ได้ — คืน 0 |
| `range m` | ได้ — วนลูปศูนย์รอบ (ไม่ error) |
| เขียนค่าด้วย `m[key] = v` | **panic** |
| `delete(m, key)` | ได้ — ไม่ error แม้ key ไม่มีอยู่หรือ map เป็น nil |

---

## 3. Key ของ map ต้องเป็น comparable type

Go บังคับว่า key ของ map ต้องเป็น type ที่**เปรียบเทียบกันได้ด้วย `==`** (comparable) เพราะกลไกภายในของ hash table ต้องใช้การเปรียบเทียบ key เพื่อตรวจสอบว่าซ้ำกันหรือไม่

Type ที่ใช้เป็น key ได้: ทุก primitive type (`string`, `int`, `float64`, `bool`, ...), pointer, struct ที่ทุก field เป็น comparable, array (แต่ **ไม่ใช่** slice)

Type ที่ใช้เป็น key **ไม่ได้**: `slice`, `map`, `function` — เพราะ type เหล่านี้ไม่มี `==` ให้ใช้ (compiler จะ error ทันที)

```go
package main

func main() {
	// compile error: invalid map key type []int
	m := map[[]int]string{}
	_ = m
}
```

ถ้าลอง compile โค้ดข้างบนจะได้ error:

```
invalid map key type []int
```

แต่ struct ใช้เป็น key ได้ตราบใดที่ทุก field ภายในเป็น comparable — เทคนิคนี้มีประโยชน์มากเวลาต้องการ key แบบผสมหลายค่า (composite key):

```go
package main

import "fmt"

type Point struct {
	X, Y int
}

func main() {
	visited := map[Point]bool{}
	visited[Point{0, 0}] = true
	visited[Point{1, 2}] = true

	fmt.Println(visited[Point{1, 2}]) // true
	fmt.Println(visited[Point{9, 9}]) // false (ยังไม่เคยเยี่ยมชม)
}
```

Pattern นี้ใช้บ่อยมากในโจทย์ประเภทกริด/พิกัด เช่น เกม, การเดินทางบนแผนที่ 2 มิติ, หรือ memoization ของฟังก์ชันที่รับ argument หลายตัว

---

## 4. อ่านค่าจาก key ที่ไม่มีอยู่ ได้ zero value เสมอ

เมื่อเข้าถึง key ที่ไม่มีอยู่จริงใน map Go จะ**ไม่ error และไม่ panic** แต่คืนค่า **zero value ของ value type** นั้นกลับมาเสมอ

```go
package main

import "fmt"

func main() {
	scores := map[string]int{"alice": 90}

	fmt.Println(scores["alice"])   // 90 — มีอยู่จริง
	fmt.Println(scores["bob"])     // 0  — ไม่มี key "bob" แต่ไม่ error ได้ zero value ของ int
}
```

พฤติกรรมนี้สะดวกมากในกรณีนับจำนวน (counting pattern) เพราะไม่ต้องเช็คก่อนว่า key มีอยู่หรือยัง:

```go
package main

import "fmt"

func main() {
	text := []string{"go", "is", "fun", "go", "is", "great"}
	count := make(map[string]int)

	for _, word := range text {
		count[word]++ // ถ้ายังไม่มี key นี้ จะเริ่มจาก 0 แล้วบวก 1 ให้อัตโนมัติ
	}

	fmt.Println(count)
}
```

ผลลัพธ์:

```
map[fun:1 go:2 great:1 is:2]
```

แต่พฤติกรรมนี้ก็เป็นดาบสองคม: ถ้า value type เป็น `bool` แล้วต้องการแยกแยะระหว่าง "key ไม่มีอยู่เลย" กับ "key มีอยู่แต่ค่าเป็น `false`" การอ่านแบบธรรมดา `v := m[key]` จะแยกแยะไม่ได้ ต้องใช้ comma ok idiom ในหัวข้อถัดไป

---

## 5. Comma ok idiom — แยกแยะ "ไม่มี key" กับ "ค่าเป็น zero value"

Go มี syntax พิเศษสำหรับอ่านค่าจาก map ที่คืนค่าสองตัวพร้อมกัน เรียกว่า **comma ok idiom**:

```go
value, ok := m[key]
```

- `value` คือค่าที่อ่านได้ (หรือ zero value ถ้าไม่มี key นั้น)
- `ok` เป็น `bool` บอกว่า key นั้น**มีอยู่จริงใน map หรือไม่** (`true` = มี, `false` = ไม่มี)

```go
package main

import "fmt"

func main() {
	m := map[string]int{"x": 1, "y": 2}

	v, ok := m["x"]
	fmt.Println(v, ok) // 1 true

	v2, ok2 := m["z"]
	fmt.Println(v2, ok2) // 0 false
}
```

Pattern นี้เจอบ่อยมากในการเขียน Go ทุกระดับ ไม่ใช่แค่กับ map — เรายังใช้ comma ok idiom กับ type assertion (Part 014) และการรับค่าจาก channel (Part 037) รูปแบบภาษาเดียวกันนี้ถูกนำมาใช้ซ้ำๆ ตลอดทั้งภาษา ซึ่งสะท้อนปรัชญา "less is more" ที่กล่าวไปใน Part 001

รูปแบบการใช้งานที่พบบ่อยที่สุดคือใน `if` แบบมี initialization statement (ตามที่เรียนไปใน Part 004):

```go
package main

import "fmt"

func main() {
	config := map[string]string{"env": "production"}

	if env, ok := config["env"]; ok {
		fmt.Println("Environment:", env)
	} else {
		fmt.Println("Environment not set, using default")
	}
}
```

รูปแบบนี้อ่านง่าย ปลอดภัย และจำกัดขอบเขตตัวแปร `env`/`ok` ให้อยู่แค่ใน `if`/`else` block เท่านั้น — เป็นสไตล์ idiomatic Go ที่ควรใช้เป็นค่าเริ่มต้นเมื่อต้องเช็คว่า key มีอยู่หรือไม่

**สำคัญ**: ถ้าเขียน `v, ok := m[key]` แล้วใช้แค่ `v` โดยไม่สนใจ `ok` เลย ก็ยังทำงานได้ปกติ (ไม่มี error) แต่จะไม่สามารถแยกแยะได้ว่า `v` เป็น zero value เพราะไม่มี key จริงๆ หรือเพราะ value ที่เก็บไว้บังเอิญเป็น zero value เอง — ดังนั้นเมื่อความแตกต่างนี้สำคัญกับ logic ของโปรแกรม ให้ใช้ `ok` เสมอ

---

## 6. ลบ key ด้วย `delete`

Go มี built-in function ชื่อ `delete` สำหรับลบ key ออกจาก map:

```go
delete(m, key)
```

```go
package main

import "fmt"

func main() {
	m := map[string]int{"a": 1, "b": 2, "c": 3}

	delete(m, "b")
	fmt.Println(m) // map[a:1 c:3]

	delete(m, "z") // key "z" ไม่มีอยู่จริง — ไม่ error ไม่ panic
	fmt.Println(m) // map[a:1 c:3]
}
```

จุดที่ควรจำ: `delete` เป็น **no-op ที่ปลอดภัยเสมอ** ไม่ว่า key นั้นจะมีอยู่จริงหรือไม่ก็ตาม และเรียก `delete` บน nil map ก็ไม่ panic เช่นกัน (เพราะไม่มีอะไรให้ลบอยู่แล้ว) ต่างจากการเขียนค่าลง nil map ที่ panic ตามที่อธิบายไปในหัวข้อที่ 2

`delete` ไม่มีค่า return ใดๆ — ถ้าต้องการรู้ว่า key ที่จะลบมีอยู่จริงหรือไม่ก่อนลบ ให้เช็คด้วย comma ok idiom ก่อน:

```go
if _, ok := m["b"]; ok {
	delete(m, "b")
	fmt.Println("deleted")
}
```

---

## 7. ลำดับการวน iterate ถูกสุ่มโดยเจตนา

นี่คือพฤติกรรมที่ทำให้โปรแกรมเมอร์จากภาษาอื่น (โดยเฉพาะ Python ที่ dict คงลำดับการ insert ตั้งแต่ 3.7) แปลกใจบ่อยที่สุด: **การใช้ `for range` วนลูปบน map ใน Go จะได้ลำดับที่แตกต่างกันในแต่ละครั้งที่รัน โดยเจตนา**

```go
package main

import "fmt"

func main() {
	m := map[string]int{"a": 1, "b": 2, "c": 3, "d": 4, "e": 5}
	for k, v := range m {
		fmt.Printf("%s=%d ", k, v)
	}
	fmt.Println()
}
```

รันซ้ำหลายครั้งจะได้ผลลัพธ์ที่ลำดับต่างกัน เช่น:

```
a=1 b=2 c=3 d=4 e=5
d=4 e=5 a=1 b=2 c=3
a=1 b=2 c=3 d=4 e=5
```

### ทำไม Go ถึงจงใจสุ่มลำดับ?

ทีมออกแบบ Go ตัดสินใจแบบนี้ด้วยเหตุผลหลัก 2 ข้อ:

1. **ป้องกันไม่ให้โปรแกรมเมอร์เผลอพึ่งพาลำดับที่ไม่ได้ถูกรับประกัน** — ถ้า Go คงลำดับแบบใดแบบหนึ่งไว้ (เช่นตามลำดับ insert หรือตามค่า hash) โปรแกรมเมอร์จำนวนมากจะเผลอเขียนโค้ดที่พึ่งพาลำดับนั้นโดยไม่รู้ตัว แล้วพอ Go เปลี่ยน implementation ภายในของ hash table ในเวอร์ชันอนาคต (ซึ่งทีม Go สงวนสิทธิ์ทำได้เสมอ) โค้ดที่เคย "บังเอิญ" ทำงานถูกต้องก็จะพังโดยไม่มีการเปลี่ยน syntax ใดๆ เลย การสุ่มลำดับตั้งแต่แรกบังคับให้ทุกคนเขียนโค้ดที่ถูกต้องตั้งแต่ต้น คือ**ห้ามพึ่งพาลำดับของ map**
2. **ความปลอดภัย** — ในช่วงแรกที่ Go ยังไม่สุ่มลำดับ (ก่อน Go 1) พบว่าโปรแกรมที่รับ input จากผู้ใช้แล้วเก็บลง map (เช่น HTTP header, form field) อาจถูกโจมตีแบบ hash flooding เพื่อคาดเดาลำดับการประมวลผลได้ Go จึงเพิ่ม randomization เข้าไปที่ทั้ง hash seed และลำดับการเริ่มวน iterate

### ถ้าต้องการลำดับที่แน่นอน ต้องทำอย่างไร?

ถ้าต้องการผลลัพธ์ที่มีลำดับคงที่ (เช่นจะพิมพ์รายงาน หรือต้องการผลลัพธ์ deterministic สำหรับ test) ต้อง**ดึง key ออกมาใส่ slice แล้ว sort เอง**:

```go
package main

import (
	"fmt"
	"sort"
)

func main() {
	m := map[string]int{"banana": 2, "apple": 1, "cherry": 3}

	keys := make([]string, 0, len(m))
	for k := range m {
		keys = append(keys, k)
	}
	sort.Strings(keys)

	for _, k := range keys {
		fmt.Printf("%s: %d\n", k, m[k])
	}
}
```

ผลลัพธ์ (ลำดับนี้จะเหมือนเดิมทุกครั้งที่รัน เพราะเรา sort เอง):

```
apple: 1
banana: 2
cherry: 3
```

Pattern "ดึง key มา sort ก่อน iterate" นี้เป็นสิ่งที่ต้องทำเป็นประจำเมื่อทำงานกับ map ใน Go โดยเฉพาะเวลาต้องการผลลัพธ์ที่ reproducible เช่นใน unit test เราจะกลับมาใช้แพ็กเกจ `sort` แบบเจาะลึกใน Part 027

> หมายเหตุ: การที่ `fmt.Println(m)` พิมพ์ map ออกมาแบบเรียง key แล้ว (ตามที่เห็นในหัวข้อที่ 1) เป็นความสะดวกที่ `fmt` package ทำให้เฉพาะตอน**แสดงผล**เท่านั้น ไม่ได้แปลว่า iteration order ของ map เปลี่ยนไปด้วย

---

## 8. Map มีพฤติกรรมแบบ reference — ไม่ copy ตอน assign หรือ pass เข้าฟังก์ชัน

ตามที่เรียนไปใน Part 006 ว่า slice เป็น "reference-like type" (มี header ที่ชี้ไปยัง underlying array ร่วมกัน) **map ก็มีพฤติกรรมแบบเดียวกัน** ตัวแปร map ภายในคือ pointer ที่ชี้ไปยังโครงสร้าง hash table เดียวกัน

ผลที่ตามมาคือ: เมื่อ assign map ตัวหนึ่งไปยังตัวแปรใหม่ หรือส่ง map เป็น argument ให้ฟังก์ชัน **จะไม่มีการ copy ข้อมูลภายในเกิดขึ้น** ทั้งสองตัวแปรยังคงชี้ไปที่ hash table เดียวกัน การแก้ไขผ่านตัวแปรใดตัวหนึ่งจะกระทบอีกตัวแปรหนึ่งทันที

```go
package main

import "fmt"

func modify(m map[string]int) {
	m["a"] = 99 // แก้ไข map ต้นฉบับโดยตรง แม้จะรับผ่าน parameter แบบไม่มี pointer
}

func main() {
	orig := map[string]int{"a": 1}
	modify(orig)
	fmt.Println("after modify:", orig) // after modify: map[a:99]

	cp := orig   // ไม่ copy ข้อมูล แค่ copy "ตัวชี้" ไปที่ hash table เดียวกัน
	cp["b"] = 2
	fmt.Println("orig after cp write:", orig) // orig after cp write: map[a:99 b:2]
}
```

ผลลัพธ์:

```
after modify: map[a:99]
orig after cp write: map[a:99 b:2]
```

นี่คือความแตกต่างสำคัญเมื่อเทียบกับ struct หรือ array ที่เป็น **value type แท้ๆ** (copy ทั้งก้อนเมื่อ assign — จะเรียนเจาะลึกใน Part 011) การส่ง map เข้าฟังก์ชันจึงไม่จำเป็นต้องใช้ pointer (`*map[K]V`) เพื่อให้ฟังก์ชันแก้ไขค่าภายในได้ ต่างจาก struct ที่ถ้าอยากให้ฟังก์ชันแก้ไขค่าต้นฉบับได้ ต้องส่ง pointer ไปเท่านั้น (รายละเอียดใน Part 010)

ข้อควรระวัง: แม้ฟังก์ชันจะแก้ไข**เนื้อหา**ใน map ต้นฉบับได้ แต่ถ้าฟังก์ชันนั้นเปลี่ยนตัวแปร parameter ให้ชี้ไปยัง map ก้อนใหม่ทั้งหมด (เช่น `m = make(map[string]int)` หรือ `m = nil`) การเปลี่ยนแปลงนั้นจะเกิดแค่ใน local copy ของตัวแปร (ตัวชี้) เท่านั้น ไม่กระทบตัวแปรต้นฉบับที่ผู้เรียกถืออยู่ — เพราะตัวชี้เองก็ถูกส่งเข้ามาแบบ by value เช่นกัน:

```go
package main

import "fmt"

func replace(m map[string]int) {
	m = map[string]int{"z": 100} // แค่เปลี่ยนให้ local variable m ชี้ไปที่ map ใหม่ ไม่กระทบของเดิม
}

func main() {
	orig := map[string]int{"a": 1}
	replace(orig)
	fmt.Println(orig) // map[a:1] — ไม่เปลี่ยนแปลง
}
```

สรุปหลักการ: **แก้ไข entry ภายใน map ที่มีอยู่แล้ว กระทบต้นฉบับเสมอ แต่การเปลี่ยนให้ตัวแปรชี้ไป map ก้อนใหม่ ไม่กระทบต้นฉบับ** หลักการเดียวกันนี้จะเจอซ้ำอีกครั้งตอนเรียนเรื่อง slice กับฟังก์ชันใน Part 006 และ pointer ใน Part 010

---

## 9. Map เป็น Set ด้วย `map[T]struct{}`

Go **ไม่มี built-in Set type** (ต่างจาก Python ที่มี `set()` หรือ Java ที่มี `HashSet`) แต่โปรแกรมเมอร์ Go นิยมจำลอง Set ด้วย map ที่ value เป็น `bool` หรือ (แบบ idiomatic กว่า) `struct{}`

### แบบใช้ `bool`

```go
package main

import "fmt"

func main() {
	set := map[string]bool{}
	set["apple"] = true
	set["banana"] = true

	if set["apple"] {
		fmt.Println("apple is in the set")
	}
	if !set["cherry"] {
		fmt.Println("cherry is NOT in the set")
	}
}
```

วิธีนี้เข้าใจง่าย แต่มีจุดสับสน: ค่า `false` เกิดได้สองทาง คือ "ไม่มี key นี้เลย" กับ "มี key นี้แต่ตั้งใจใส่ `false`" ซึ่งปกติแล้วโปรแกรมเมอร์ไม่ค่อยตั้งใจใส่ `false` แต่ก็เป็นช่องให้เกิด bug ได้

### แบบใช้ `struct{}` (แนะนำ — idiomatic กว่า)

```go
package main

import "fmt"

func main() {
	set := map[string]struct{}{}
	set["apple"] = struct{}{}
	set["banana"] = struct{}{}

	_, exists := set["apple"]
	fmt.Println("apple exists:", exists) // true

	_, exists2 := set["cherry"]
	fmt.Println("cherry exists:", exists2) // false
}
```

เหตุผลที่ `struct{}` (empty struct) เป็นตัวเลือกที่ดีกว่าคือ **empty struct ไม่กินพื้นที่ memory เลย** (ขนาด 0 byte) ต่างจาก `bool` ที่ยังกิน 1 byte ต่อ entry เมื่อ set มีขนาดใหญ่มาก (หลักล้าน entries) ความแตกต่างนี้มีผลต่อการใช้ memory จริง และการใช้ `struct{}` ยังสื่อความหมายชัดเจนกว่าด้วยว่า "เราสนใจแค่ว่า key นี้มีอยู่หรือไม่ ไม่สนใจ value เลย"

Pattern การเช็ค set: ใช้ comma ok idiom เสมอ (`_, exists := set[key]`) แทนการอ่านค่าตรงๆ เพราะ value ของ `struct{}` ไม่มีความหมายอะไรให้ใช้งานได้อยู่แล้ว

### ตัวอย่างการใช้งานจริง: หา element ที่ซ้ำกัน

```go
package main

import "fmt"

func hasDuplicate(items []string) bool {
	seen := make(map[string]struct{})
	for _, item := range items {
		if _, ok := seen[item]; ok {
			return true
		}
		seen[item] = struct{}{}
	}
	return false
}

func main() {
	fmt.Println(hasDuplicate([]string{"a", "b", "c"}))      // false
	fmt.Println(hasDuplicate([]string{"a", "b", "a"}))      // true
}
```

---

## 10. Nested maps (map ซ้อน map)

Value ของ map เป็น type อะไรก็ได้ รวมถึงเป็น map ตัวอื่นซ้อนกันเข้าไปอีกชั้น — เรียกว่า **nested map** ใช้บ่อยเวลาต้องการจัดกลุ่มข้อมูลแบบสองมิติหรือมากกว่า (คล้าย JSON object ซ้อนกัน)

```go
package main

import "fmt"

func main() {
	nested := map[string]map[string]int{}

	// ต้องสร้าง inner map ก่อนเสมอ ก่อนจะเขียนค่าลงไป
	nested["fruit"] = map[string]int{"apple": 5}
	nested["fruit"]["banana"] = 3

	fmt.Println(nested) // map[fruit:map[apple:5 banana:3]]
}
```

ข้อควรระวังสำคัญ: ถ้า outer map ยังไม่มี key นั้นอยู่ (เช่น `nested["vegetable"]`) การเข้าถึง `nested["vegetable"]` จะได้ **nil map** กลับมา (zero value ของ `map[string]int`) แล้วถ้าพยายามเขียนต่อทันทีจะ panic:

```go
package main

func main() {
	nested := map[string]map[string]int{}
	nested["vegetable"]["carrot"] = 10 // panic: assignment to entry in nil map
}
```

เพราะ `nested["vegetable"]` คืนค่า nil map (ไม่มี key "vegetable" อยู่จริง) แล้วเราพยายามเขียนค่าลง nil map นั้นทันที ตามกฎในหัวข้อที่ 2 วิธีแก้คือต้องเช็คและสร้าง inner map ก่อนเขียนค่าเสมอ:

```go
package main

import "fmt"

func addPrice(catalog map[string]map[string]int, category, item string, price int) {
	if catalog[category] == nil {
		catalog[category] = make(map[string]int)
	}
	catalog[category][item] = price
}

func main() {
	catalog := map[string]map[string]int{}

	addPrice(catalog, "fruit", "apple", 30)
	addPrice(catalog, "fruit", "banana", 15)
	addPrice(catalog, "vegetable", "carrot", 20)

	fmt.Println(catalog)
}
```

รูปแบบ `if catalog[category] == nil { catalog[category] = make(...) }` เป็น pattern มาตรฐานที่ต้องใช้ทุกครั้งเมื่อทำงานกับ nested map — เรียกกันว่า "lazy initialization" ของ inner map

สำหรับข้อมูลที่ซับซ้อนกว่านี้ (เช่นโครงสร้างที่มี field หลายแบบผสมกัน) มักจะเลือกใช้ **struct ซ้อนกันแทน map ซ้อนกัน** เพราะได้ type safety ที่ดีกว่าและ compiler ช่วยตรวจสอบ field ผิดพลาดให้ (จะเรียนเรื่อง struct เจาะลึกใน Part 011) โดยทั่วไป nested map เหมาะกับกรณีที่ key เป็น dynamic (ไม่รู้ล่วงหน้าว่ามี key อะไรบ้าง) เช่น การแปลง JSON ที่โครงสร้างไม่ตายตัว

---

## สรุปสิ่งที่ได้เรียนในบทนี้

- Map คือโครงสร้างข้อมูล key-value ที่ implement ด้วย hash table สร้างได้ด้วย `make()` หรือ map literal
- Zero value ของ map คือ `nil` — nil map **อ่านได้** (ได้ zero value) แต่**เขียนไม่ได้** (panic: assignment to entry in nil map)
- Key ของ map ต้องเป็น comparable type (ห้ามใช้ slice, map, function เป็น key) แต่ struct ที่ทุก field comparable ใช้เป็น composite key ได้
- อ่าน key ที่ไม่มีอยู่จริงจะได้ zero value เสมอ ไม่ error ไม่ panic — มีประโยชน์มากในการทำ counting pattern
- **Comma ok idiom** (`v, ok := m[key]`) ใช้แยกแยะ "key ไม่มีอยู่" กับ "ค่าเป็น zero value จริงๆ"
- `delete(m, key)` ลบ key ออกจาก map ได้อย่างปลอดภัยเสมอ แม้ key ไม่มีอยู่หรือ map เป็น nil
- ลำดับการ `range` บน map ถูกสุ่มโดยเจตนา เพื่อป้องกันไม่ให้โค้ดพึ่งพาลำดับที่ไม่ได้รับประกัน — ถ้าต้องการลำดับคงที่ ต้อง sort key เอง
- Map มีพฤติกรรมแบบ reference: assign หรือ pass เข้าฟังก์ชันไม่ copy ข้อมูล ทั้งสองตัวแปรยังชี้ไปที่ hash table เดียวกัน
- ใช้ `map[T]struct{}` เพื่อจำลอง Set ใน Go เพราะ `struct{}` ไม่กิน memory และสื่อความหมายชัดเจนกว่า `bool`
- Nested map ต้องสร้าง inner map ก่อนเสมอด้วย lazy initialization ก่อนเขียนค่าลงไป มิฉะนั้นจะเจอ nil map panic

## แบบฝึกหัดท้ายบท

1. เขียนโปรแกรมนับความถี่ของตัวอักษรในประโยคหนึ่งประโยค (ไม่นับช่องว่าง) แล้วพิมพ์ผลลัพธ์แบบเรียงตามตัวอักษร (a-z) โดยใช้เทคนิคดึง key มา `sort` ก่อน
2. เขียนฟังก์ชัน `intersect(a, b []string) []string` ที่คืนค่า element ที่ปรากฏอยู่ใน slice ทั้งสองตัว โดยใช้ map เป็นตัวช่วยค้นหา (ห้ามใช้ nested loop O(n²))
3. ทดลองเขียนโค้ดที่เขียนค่าลง nil map แล้วจับ panic ด้วยตัวเอง สังเกตข้อความ error ที่ได้ และอธิบายด้วยคำพูดตัวเองว่าทำไมถึง panic
4. เขียนโปรแกรมที่มี nested map แบบ `map[string]map[string]bool` แทนการเก็บว่านักเรียนแต่ละคน (key ชั้นนอก) ลงทะเบียนวิชาอะไรบ้าง (key ชั้นใน) โดยต้องจัดการ lazy initialization ให้ถูกต้อง
5. เขียนฟังก์ชันสองตัว: ตัวหนึ่งรับ `map[string]int` แล้วแก้ไข entry ที่มีอยู่ อีกตัวรับ `map[string]int` แล้วลองใช้ `m = make(map[string]int)` ทับตัวแปร แล้วพิสูจน์ด้วยการรันจริงว่าฟังก์ชันไหนกระทบ map ต้นฉบับที่ผู้เรียกถืออยู่ และฟังก์ชันไหนไม่กระทบ พร้อมอธิบายเหตุผล
6. ลองสร้าง map ที่ key เป็น struct (composite key) เพื่อจำลองกระดานหมากรุกขนาด 8x8 ที่เก็บว่าช่องไหนมีตัวหมากอยู่บ้าง (`map[Position]string` โดย `Position` มี field `Row`, `Col`)

---

**ต่อไป**: [Part 008 — Functions พื้นฐาน, multiple return values](./008-functions-basics.md)
