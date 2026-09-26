# Part 031: Reflection พื้นฐานด้วย `reflect`

> ภาคที่ 2: ระดับกลาง (Intermediate) — ตอนที่ 16 จาก 20 (Part 16–35)

## สารบัญของบทนี้

1. Reflection คืออะไร และทำไมแทบไม่จำเป็นในชีวิตประจำวัน
2. เทียบกับ Generics (Part 028-029): ปัญหาคล้ายกัน แต่แก้คนละแบบ
3. `reflect.TypeOf` และ `reflect.ValueOf` พื้นฐาน
4. `Kind()` vs `Type()`: ความแตกต่างที่มือใหม่สับสนบ่อยที่สุด
5. ตรวจสอบ Struct Fields และ Tags ที่ Runtime
6. เบื้องหลัง `encoding/json`: Reflection ทำงานตรงไหน
7. แก้ไขค่าผ่าน Reflection: `Elem()`, `CanSet()` และกฎ Addressability
8. สร้างค่าใหม่แบบไดนามิกด้วย `reflect.New`
9. ข้อควรระวัง: ทำไม Reflection ช้าและทำลาย Type Safety ตอน Compile Time
10. เมื่อไหร่ควรใช้ Reflection จริงๆ
11. สรุปสิ่งที่ได้เรียนในบทนี้
12. แบบฝึกหัดท้ายบท

---

## 1. Reflection คืออะไร และทำไมแทบไม่จำเป็นในชีวิตประจำวัน

**Reflection** คือความสามารถของโปรแกรมในการ**ตรวจสอบและจัดการโครงสร้างของตัวเองที่ runtime** — พูดง่ายๆ คือโค้ดที่สามารถ "มองย้อนกลับมาดูตัวเอง" ได้ว่าตัวแปรตัวหนึ่งมี type อะไร มี field อะไรบ้าง มี method อะไรบ้าง โดยที่ตอนเขียนโค้ด (compile time) เราไม่รู้ล่วงหน้าว่า value ตัวนั้นจะเป็น type อะไรแน่ชัด

ใน Go ความสามารถนี้อยู่ใน package มาตรฐาน **`reflect`** เราได้เห็นร่องรอยของมันมาแล้วโดยไม่รู้ตัวใน **Part 025** — ตอนที่เรียนเรื่อง `encoding/json` เราบอกไว้ว่า `json.Marshal`/`json.Unmarshal` เข้าถึง field ของ struct ได้เฉพาะ **exported field** เท่านั้น เพราะมันใช้กลไก `reflect` ในการอ่านค่า field เหล่านั้นที่ runtime นั่นเอง

คำถามที่ต้องตอบให้ชัดตั้งแต่ต้นบทคือ: **แล้วนักพัฒนา Go ทั่วไปต้องใช้ `reflect` บ่อยแค่ไหน?**

คำตอบคือ **แทบไม่ต้องใช้เลยในโค้ด business logic ทั่วไป** Go เป็นภาษา static typed ที่ตั้งใจออกแบบให้เขียนโปรแกรมได้โดยรู้ type ของทุกอย่างตั้งแต่ compile time (ตามปรัชญา "less is more" และ "static typing" ที่พูดถึงใน **Part 001**) `reflect` มีไว้สำหรับกรณีพิเศษที่หลีกเลี่ยงไม่ได้จริงๆ เท่านั้น เช่น การเขียน library ที่ต้องทำงานกับ type ใดๆ ก็ได้แบบไม่รู้จักล่วงหน้า (เช่น `encoding/json`, ORM, dependency injection framework) มีคำพูดที่โด่งดังของ Rob Pike หนึ่งในผู้ออกแบบภาษา Go ที่สรุปเรื่องนี้ได้ตรงที่สุด:

> "Clear is better than clever. Reflection is never clear." — Rob Pike

หรือแปลเป็นไทยประมาณว่า "โค้ดที่ชัดเจนดีกว่าโค้ดที่ฉลาดจัด และ reflection ไม่เคยเป็นโค้ดที่ชัดเจน" นั่นเอง เหตุผลคือ:

- โค้ดที่ใช้ `reflect` มักอ่านยากกว่าโค้ดปกติมาก เพราะ type ถูกซ่อนอยู่หลัง `interface{}`/`any` และ compiler ตรวจสอบไม่ได้ว่าโค้ดถูกต้องจริงหรือไม่จนกว่าจะรันจริง
- Error จาก reflection ส่วนใหญ่เป็น **panic ตอน runtime** ไม่ใช่ compile error ที่ตรวจจับได้ตั้งแต่เขียนโค้ด
- Performance ช้ากว่าโค้ดปกติมาก (จะพิสูจน์ให้เห็นด้วยตัวเลขจริงในหัวข้อที่ 8)

ดังนั้นบทนี้ไม่ได้สอนให้ "ใช้ reflection ให้บ่อยขึ้น" แต่สอนให้**เข้าใจกลไกเบื้องหลัง**ของหลาย library ที่เราใช้ทุกวัน (เช่น `encoding/json` ที่เรียนไปแล้ว และ GORM ที่จะเรียนใน **Part 074**) และรู้ว่าเมื่อไหร่การใช้ `reflect` เป็นทางเลือกที่สมเหตุสมผลจริงๆ

---

## 2. เทียบกับ Generics (Part 028-029): ปัญหาคล้ายกัน แต่แก้คนละแบบ

ถ้าเรียนหลักสูตรนี้มาตามลำดับ ตอนนี้ควรผ่าน **Generics** (Part 028-029) มาแล้ว ซึ่งเป็นวิธีที่ Go 1.18 เพิ่มเข้ามาเพื่อเขียนฟังก์ชัน/data structure ที่ทำงานกับหลาย type ได้โดยไม่ต้องเขียนโค้ดซ้ำ คำถามที่เกิดขึ้นตามธรรมชาติคือ **reflection กับ generics ต่างกันตรงไหน ในเมื่อทั้งคู่ดูเหมือนจะแก้ปัญหา "โค้ดที่ทำงานกับหลาย type" เหมือนกัน?**

คำตอบคือ ทั้งสองแก้ปัญหาคนละจุดในสเปกตรัมเดียวกัน:

| หัวข้อเปรียบเทียบ | Generics (Part 028-029) | Reflection (บทนี้) |
|---|---|---|
| **รู้ type ตอนไหน** | Compile time — compiler รู้ type ที่แน่นอนตอน instantiate | Runtime เท่านั้น — โปรแกรมต้องรันก่อนถึงจะรู้ type จริง |
| **Type safety** | ปลอดภัยเต็มรูปแบบ compiler ตรวจสอบให้ | ไม่มีการตรวจสอบตอน compile เลย ผิดพลาดกลายเป็น panic ตอนรัน |
| **Performance** | เท่ากับโค้ดที่เขียนเฉพาะ type นั้นๆ (แทบไม่มี overhead) | ช้ากว่ามาก เพราะต้องตรวจสอบและ dispatch ผ่าน type information ที่ runtime |
| **ต้องรู้ type ล่วงหน้าไหม** | ต้องรู้ (ระบุผ่าน type parameter `[T any]`) แม้จะ "ทั่วไป" แค่ไหนก็ยังมีขอบเขตจาก constraint | ไม่ต้องรู้เลย — รับ `any` แล้วค้นดูตอนรันว่าเป็น type อะไรจริงๆ |
| **ตัวอย่างการใช้** | `Map[T, U any](s []T, f func(T) U) []U`, generic stack/queue | `json.Marshal(x any)`, ORM ที่ map struct field ไป column database |

พูดให้ชัดที่สุด: **ถ้ารู้ว่าโค้ดต้องรองรับ type ไหนบ้างตั้งแต่ตอนเขียน (แม้จะหลาย type) ให้ใช้ generics เสมอ** เพราะได้ทั้งความปลอดภัยและ performance เต็มที่ ส่วน **reflection ใช้เฉพาะตอนที่โปรแกรมต้อง "ค้นพบ" โครงสร้างของ value ที่ไม่รู้จักล่วงหน้าจริงๆ ที่ runtime** เช่น เขียน library ที่ผู้ใช้ส่ง struct อะไรมาก็ได้ แล้ว library ต้องอ่าน field ทั้งหมดของมันเองโดยอัตโนมัติ — นี่คือสิ่งที่ generics ทำไม่ได้ เพราะ generics ยังต้องรู้ "รูปร่าง" ของ type ผ่าน constraint อยู่ดี ไม่สามารถ "เดิน" ผ่าน field ของ struct ที่ไม่รู้จักล่วงหน้าได้

หลาย library ในโลก Go จริงๆ ก็ค่อยๆ ย้ายจาก reflection ไปใช้ generics มากขึ้นเมื่อทำได้ (เพราะเร็วกว่าและปลอดภัยกว่า) แต่งานบางอย่าง เช่น การ serialize struct ทั่วไปที่ไม่รู้จักล่วงหน้า (`encoding/json`) หรือ ORM ที่ต้อง map struct field ไปยัง column ในฐานข้อมูล ยังคงต้องพึ่ง reflection อยู่ดี เพราะเป็นปัญหาที่ generics แก้ไม่ได้โดยธรรมชาติของมัน

---

## 3. `reflect.TypeOf` และ `reflect.ValueOf` พื้นฐาน

หัวใจของ package `reflect` มีสองฟังก์ชันหลักที่ต้องรู้จักก่อนอย่างอื่นทั้งหมด:

```go
func TypeOf(i any) Type   // คืนข้อมูล "type" ของค่า i
func ValueOf(i any) Value // คืน "ค่า" ของ i ในรูปแบบที่ reflect จัดการได้
```

ทั้งสองฟังก์ชันรับ parameter เป็น `any` (เทียบเท่า `interface{}` ที่เรียนใน **Part 014**) — นี่คือเหตุผลที่ reflection ทำงานได้กับทุก type: เพราะทุก type ใน Go แปลงเป็น `any` ได้เสมอ และเมื่อค่าถูกห่อเข้า interface มันจะพก **(type, value)** ติดไปด้วยเสมอ (ทบทวนเรื่อง interface value คือคู่ (type, value) จาก **Part 013**) — `reflect` ก็คือกลไกที่เปิดดูคู่ (type, value) ที่ซ่อนอยู่ในทุก interface นี้แหละ

```go
package main

import (
	"fmt"
	"reflect"
)

func main() {
	var x int = 42
	var s string = "hello"
	var f float64 = 3.14

	fmt.Println(reflect.TypeOf(x), reflect.ValueOf(x))
	fmt.Println(reflect.TypeOf(s), reflect.ValueOf(s))
	fmt.Println(reflect.TypeOf(f), reflect.ValueOf(f))

	t := reflect.TypeOf(x)
	v := reflect.ValueOf(x)
	fmt.Println("Type:", t)
	fmt.Println("Kind:", t.Kind())
	fmt.Println("Value:", v)
	fmt.Println("Value as int:", v.Int())
}
```

ผลลัพธ์:

```
int 42
string hello
float64 3.14
Type: int
Kind: int
Value: 42
Value as int: 42
```

จุดที่ควรสังเกต:

- `reflect.TypeOf(x)` คืนค่า type `reflect.Type` ที่บอกข้อมูลเกี่ยวกับ "ชนิด" ของ `x` (ในที่นี้คือ `int`)
- `reflect.ValueOf(x)` คืนค่า type `reflect.Value` ที่ห่อ "ค่าจริง" ของ `x` เอาไว้ พร้อม method สำหรับดึงค่ากลับออกมา เช่น `v.Int()` (ดึงเป็น `int64`), `v.String()` (ดึงเป็น `string`), `v.Float()` (ดึงเป็น `float64`)
- การเรียก method ดึงค่าผิดประเภท (เช่นเรียก `.Int()` กับค่าที่เป็น string) จะทำให้เกิด **panic ตอน runtime ทันที** — นี่คือตัวอย่างแรกที่เห็นได้ชัดว่า reflection ไม่มี compile-time safety เลย

---

## 4. `Kind()` vs `Type()`: ความแตกต่างที่มือใหม่สับสนบ่อยที่สุด

จุดที่คนเริ่มเรียน `reflect` สับสนบ่อยที่สุดคือความแตกต่างระหว่าง `Type()` กับ `Kind()`:

- **`Type`** คือ type ที่แท้จริงตามที่ประกาศไว้ในโค้ด (รวม named type ที่ผู้เขียนตั้งชื่อเอง เช่น `Celsius`, `User`)
- **`Kind`** คือ "หมวดหมู่พื้นฐาน" ที่ type นั้นสร้างขึ้นมาจากอะไร (เช่น `float64`, `struct`, `slice`, `ptr`, `map`) — Go มี Kind อยู่ประมาณ 26 แบบเท่านั้น กำหนดตายตัวไว้ใน package `reflect`

พูดง่ายๆ: **`Type` คือชื่อเฉพาะของ type นั้น ส่วน `Kind` คือ "โครงสร้างต่ำสุด" ที่มันถูกสร้างขึ้นมา** ทุก custom type ที่เราตั้งชื่อเองจะมี `Type` เป็นชื่อของมันเอง แต่ `Kind` จะเป็น type ดั้งเดิมที่มันอิงอยู่เสมอ:

```go
package main

import (
	"fmt"
	"reflect"
)

type Celsius float64

type User struct {
	Name  string
	Age   int
	Email string
}

func main() {
	var c Celsius = 36.5
	t := reflect.TypeOf(c)
	fmt.Println("Type:", t)        // main.Celsius
	fmt.Println("Kind:", t.Kind()) // float64

	u := User{Name: "A", Age: 1}
	tu := reflect.TypeOf(u)
	fmt.Println("Type:", tu)        // main.User
	fmt.Println("Kind:", tu.Kind()) // struct

	pu := &u
	tp := reflect.TypeOf(pu)
	fmt.Println("Type:", tp)        // *main.User
	fmt.Println("Kind:", tp.Kind()) // ptr
	fmt.Println("Elem Kind:", tp.Elem().Kind())
}
```

ผลลัพธ์:

```
Type: main.Celsius
Kind: float64
Type: main.User
Kind: struct
Type: *main.User
Kind: ptr
Elem Kind: struct
```

ทบทวนจาก **Part 012**: `Celsius` คือ named type ที่สร้างจาก `float64` ตอนเรียก `Type()` เราจะได้ชื่อเฉพาะ `main.Celsius` แต่ตอนเรียก `Kind()` มันจะบอกว่าโครงสร้างพื้นฐานคือ `float64` เสมอ

ทำไมความแตกต่างนี้ถึงสำคัญมากในทางปฏิบัติ? เพราะเวลาเขียนโค้ดที่ต้องจัดการกับ value แบบทั่วไป (เช่น เขียนฟังก์ชันที่ต้องแยกว่า value เป็น struct หรือ slice หรือ pointer) **เราแทบจะเช็คด้วย `Kind()` เสมอ ไม่ใช่ `Type()`** เพราะ `Kind()` มีจำนวนค่าคงที่ (ประมาณ 26 แบบ) จึงเขียน `switch` ครอบคลุมได้หมด ในขณะที่ `Type()` มีค่าที่เป็นไปได้ไม่จำกัด (เพราะใครๆ ก็สร้าง named type ใหม่ได้ตลอด) — นี่คือเหตุผลที่ฟังก์ชัน `PrintFields` ในหัวข้อถัดไปเช็คด้วย `v.Kind() == reflect.Ptr` และ `v.Kind() == reflect.Struct` แทนที่จะเช็ค `Type()`

---

## 5. ตรวจสอบ Struct Fields และ Tags ที่ Runtime

นี่คือการใช้งาน `reflect` ที่พบบ่อยที่สุดในโลกจริง: การเดินผ่าน field ทั้งหมดของ struct ที่ไม่รู้จักล่วงหน้า พร้อมอ่าน struct tag ของแต่ละ field (ทบทวน struct tag จาก **Part 025** ที่ใช้ `json:"..."` ควบคุมพฤติกรรมของ `encoding/json`)

มาลองเขียนฟังก์ชัน `PrintFields` ที่รับ struct อะไรก็ได้ (คล้ายกับที่ generics ทำไม่ได้ เพราะเราไม่รู้ล่วงหน้าว่า struct จะมี field อะไรบ้าง) แล้วพิมพ์ชื่อ, type, ค่า และ tag ของทุก field ออกมา ฟังก์ชันแบบนี้คือรากฐานเดียวกับที่ `encoding/json` ใช้ในการอ่าน struct tag `json:"..."` นั่นเอง:

```go
package main

import (
	"fmt"
	"reflect"
)

type Product struct {
	Name    string  `label:"ชื่อสินค้า"`
	Price   float64 `label:"ราคา"`
	InStock bool    `label:"มีสินค้า"`
}

// PrintFields รับ struct (หรือ pointer ไปยัง struct) ตัวไหนก็ได้ผ่าน any
// แล้ววนพิมพ์ทุก field พร้อมชื่อ, ค่า, type และ tag "label" ถ้ามี
func PrintFields(x any) {
	v := reflect.ValueOf(x)
	if v.Kind() == reflect.Ptr {
		v = v.Elem()
	}
	if v.Kind() != reflect.Struct {
		fmt.Println("ไม่ใช่ struct, ข้าม")
		return
	}
	t := v.Type()
	for i := 0; i < t.NumField(); i++ {
		field := t.Field(i)
		value := v.Field(i)
		label := field.Tag.Get("label")
		fmt.Printf("- %s (%s) = %v", field.Name, field.Type, value.Interface())
		if label != "" {
			fmt.Printf(" [label: %s]", label)
		}
		fmt.Println()
	}
}

func main() {
	p := Product{Name: "เมาส์", Price: 199.5, InStock: true}
	PrintFields(p)
	fmt.Println("---")
	PrintFields(&p)
	fmt.Println("---")
	PrintFields(42)
}
```

ผลลัพธ์:

```
- Name (string) = เมาส์ [label: ชื่อสินค้า]
- Price (float64) = 199.5 [label: ราคา]
- InStock (bool) = true [label: มีสินค้า]
---
- Name (string) = เมาส์ [label: ชื่อสินค้า]
- Price (float64) = 199.5 [label: ราคา]
- InStock (bool) = true [label: มีสินค้า]
---
ไม่ใช่ struct, ข้าม
```

เจาะรายละเอียดของฟังก์ชันนี้ทีละบรรทัด:

- `if v.Kind() == reflect.Ptr { v = v.Elem() }` — ถ้าค่าที่ส่งมาเป็น pointer (เช่น `&p`) เราต้อง "ปอกเปลือก" ด้วย `Elem()` ก่อนเพื่อเข้าถึง struct ที่มันชี้ไป ทำให้ฟังก์ชันรองรับได้ทั้งการส่ง struct ตรงๆ และการส่ง pointer
- `t.NumField()` คืนจำนวน field ทั้งหมดของ struct
- `t.Field(i)` คืน `reflect.StructField` ที่มีข้อมูล metadata ของ field ตัวที่ `i` เช่นชื่อ (`.Name`), type (`.Type`), และที่สำคัญคือ `.Tag` — struct tag แบบ raw string
- `v.Field(i)` คืน `reflect.Value` ของ field ตัวที่ `i` (ค่าจริง ไม่ใช่ metadata) — ต้องใช้ `.Interface()` เพื่อแปลงกลับเป็น `any` ก่อนนำไปพิมพ์หรือใช้งานต่อแบบปกติ
- `field.Tag.Get("label")` คือวิธีอ่านค่าจาก struct tag ด้วยชื่อ key — เป็น method เดียวกับที่ `encoding/json` เรียกภายในด้วย `field.Tag.Get("json")` เพื่ออ่าน tag `json:"..."` ที่เราคุ้นเคยจาก **Part 025**

---

## 6. เบื้องหลัง `encoding/json`: Reflection ทำงานตรงไหน

ตอนนี้เรามีเครื่องมือครบพอที่จะเข้าใจ**อย่างแท้จริง**ว่าทำไม `encoding/json` ถึงมีพฤติกรรมตามที่เรียนไปใน **Part 025** ทุกข้อ:

1. **"เฉพาะ exported field เท่านั้นที่ถูก marshal"** — เพราะ `reflect` มีข้อจำกัดระดับภาษาว่า `v.Field(i).Interface()` จะ **panic ทันที** ถ้า field นั้นเป็น unexported (ตัวพิมพ์เล็กนำหน้า) เมื่อเรียกจาก package ภายนอก นี่ไม่ใช่ทางเลือกของผู้เขียน `encoding/json` แต่เป็นกฎความปลอดภัยที่ฝังอยู่ใน runtime ของ Go เอง
2. **"struct tag `json:"..."` ควบคุมชื่อ key"** — เพราะ `encoding/json` เดินผ่าน field ทุกตัวด้วย `t.Field(i)` แบบเดียวกับ `PrintFields` ข้างบน แล้วอ่าน `field.Tag.Get("json")` เพื่อตัดสินใจว่าจะใช้ชื่อ key อะไรในการ encode
3. **"`json.Unmarshal` ต้องรับ pointer"** — เพราะการเขียนค่ากลับเข้า struct ต้องใช้ `Elem()` กับค่าที่ addressable ได้ ซึ่งเป็นกลไกเดียวกับที่จะเรียนในหัวข้อถัดไป

พูดให้ชัดเจน: **`encoding/json` ไม่ใช่เวทมนตร์อะไรเลย มันคือฟังก์ชัน `PrintFields`-ish ที่ซับซ้อนขึ้นอีกหน่อย** ที่เดินผ่าน field ของ struct ด้วย `reflect` แบบเดียวกับที่เราเพิ่งเขียนเอง เพียงแต่มันสร้าง JSON bytes ออกมาแทนที่จะพิมพ์ข้อความ หลักการเดียวกันนี้จะกลับมาอีกครั้งใน **Part 074 (GORM เบื้องต้น)** ที่ ORM ใช้ reflection ในการอ่าน struct tag เพื่อตัดสินใจว่า field ไหน map กับ column ไหนในฐานข้อมูล

---

## 7. แก้ไขค่าผ่าน Reflection: `Elem()`, `CanSet()` และกฎ Addressability

การ **อ่าน** ค่าผ่าน reflection ทำได้ง่าย แต่การ **แก้ไข** ค่าผ่าน reflection มีกฎที่เข้มงวดกว่ามาก เรียกว่ากฎ **addressability** — สรุปเป็นประโยคเดียวคือ:

> **จะแก้ไขค่าผ่าน `reflect.Value` ได้ ต้องส่ง pointer เข้าไปใน `reflect.ValueOf` แล้วเรียก `.Elem()` เพื่อเข้าถึงค่าที่ pointer นั้นชี้ไปเท่านั้น**

เหตุผลเบื้องหลังตรงกับหลักการ pointer ที่เรียนใน **Part 010**: ถ้าส่งค่า struct ตรงๆ (ไม่ใช่ pointer) เข้า `reflect.ValueOf` มันจะได้แค่ **สำเนา** ของค่านั้น การแก้ไขสำเนาไม่มีผลอะไรกับตัวแปรต้นฉบับเลย Go จึงห้ามไม่ให้แก้ไขค่าที่ได้จาก `reflect.ValueOf(x)` ตรงๆ (ที่ไม่ใช่ pointer) ไปเลยตั้งแต่ต้น เพื่อไม่ให้เขียนโค้ดที่ดูเหมือนได้ผลแต่จริงๆ ไม่มีผลอะไร

ทุก `reflect.Value` มี method `CanSet()` ที่บอกว่า **ค่านั้นแก้ไขได้หรือไม่** — ลองดูตัวอย่างที่เทียบทั้งสองกรณีให้เห็นชัด:

```go
package main

import (
	"fmt"
	"reflect"
)

type User struct {
	Name string
	Age  int
}

func main() {
	u := User{Name: "สมชาย", Age: 30}

	// กรณีที่ 1: ส่งค่าตรงๆ (ไม่ใช่ pointer) -> CanSet() = false เสมอ, แก้ค่าไม่ได้
	v1 := reflect.ValueOf(u)
	fmt.Println("v1 CanSet:", v1.Field(0).CanSet()) // false

	// กรณีที่ 2: ส่ง pointer แล้วเรียก Elem() เพื่อ "ปอกเปลือก" ไปยังค่าจริง
	v2 := reflect.ValueOf(&u).Elem()
	fmt.Println("v2 CanSet:", v2.Field(0).CanSet()) // true

	nameField := v2.FieldByName("Name")
	nameField.SetString("สมหญิง")

	ageField := v2.FieldByName("Age")
	ageField.SetInt(25)

	fmt.Printf("หลังแก้ไข: %+v\n", u)

	// ลอง panic กรณีพยายาม Set ค่าที่ CanSet() เป็น false
	defer func() {
		if r := recover(); r != nil {
			fmt.Println("recovered panic:", r)
		}
	}()
	v1.Field(0).SetString("จะพัง") // ต้อง panic
}
```

ผลลัพธ์:

```
v1 CanSet: false
v2 CanSet: true
หลังแก้ไข: {Name:สมหญิง Age:25}
recovered panic: reflect: reflect.Value.SetString using unaddressable value
```

จุดสำคัญที่ต้องจำ:

- `reflect.ValueOf(u)` — ส่งค่าตรงๆ ได้ `reflect.Value` ที่ **CanSet() เป็น false เสมอ** เพราะเป็นแค่สำเนา
- `reflect.ValueOf(&u).Elem()` — ส่ง pointer แล้วปอกเปลือกด้วย `Elem()` จะได้ `reflect.Value` ที่ **addressable** และ **CanSet() เป็น true** เพราะมันอ้างอิงกลับไปยัง `u` ตัวจริง
- `FieldByName("Name")` คือการค้นหา field ด้วยชื่อ (ทางเลือกแทน `Field(i)` ที่ใช้ index) สะดวกเมื่อรู้ชื่อ field ที่ต้องการแก้แน่นอน
- การเรียก `Set...()` (เช่น `SetString`, `SetInt`, `SetBool`, `SetFloat`) กับค่าที่ `CanSet()` เป็น `false` จะ **panic ทันที** — ควรเช็ค `CanSet()` ก่อนเรียกเสมอในโค้ด production เพื่อป้องกัน panic ที่ไม่คาดคิด
- panic ในตัวอย่างข้างบนถูก recover ด้วยเทคนิคจาก **Part 017** เพื่อให้เห็นข้อความ error ชัดๆ โดยโปรแกรมไม่ตาย — ในโค้ดจริงควรป้องกันด้วยการเช็ค `CanSet()` มากกว่าปล่อยให้ panic แล้วค่อย recover

**กฎทองที่ต้องจำจากหัวข้อนี้**: การแก้ไขค่าผ่าน `reflect` ต้อง "pointer → Elem() → CanSet() true → Set" เสมอ ถ้าขาดขั้นตอนไหนไปจะได้ `reflect.Value` ที่แก้ไขไม่ได้ทันที

---

## 8. สร้างค่าใหม่แบบไดนามิกด้วย `reflect.New`

นอกจากอ่านและแก้ไขค่าที่มีอยู่แล้ว บางสถานการณ์ต้อง **สร้าง value ใหม่ขึ้นมาทั้งก้อนโดยรู้แค่ `reflect.Type`** โดยไม่รู้จัก concrete type ที่แน่ชัดตอนเขียนโค้ดเลย — สถานการณ์แบบนี้พบได้บ่อยในโค้ดของ `encoding/json` ตอน `Unmarshal` เข้า slice ของ struct หรือใน ORM ตอนต้อง "สร้างแถวใหม่" จากผลลัพธ์ query โดยไม่รู้ชนิดของ struct ปลายทางล่วงหน้า

ฟังก์ชันที่ใช้คือ **`reflect.New(t Type) Value`** ทำงานคล้าย `new()` builtin ที่เรียนใน **Part 010** ทุกประการ แต่รับ `reflect.Type` แทนการระบุชื่อ type ตรงๆ ในโค้ด และคืนค่าเป็น `reflect.Value` ที่เป็น **pointer** ไปยัง value ใหม่ที่ถูกสร้างขึ้น (ค่าเริ่มต้นเป็น zero value ของ type นั้นเสมอ):

```go
package main

import (
	"fmt"
	"reflect"
)

type User struct {
	Name string
	Age  int
}

func main() {
	t := reflect.TypeOf(User{})
	newValue := reflect.New(t) // คืน reflect.Value ที่เป็น pointer ไปยัง User ตัวใหม่ (zero value)
	fmt.Println("Kind ของ newValue:", newValue.Kind())               // ptr
	fmt.Println("Kind ของ newValue.Elem():", newValue.Elem().Kind()) // struct

	elem := newValue.Elem()
	elem.FieldByName("Name").SetString("สมชาย")
	elem.FieldByName("Age").SetInt(28)

	created := newValue.Interface().(*User)
	fmt.Printf("สร้างค่าใหม่ได้: %+v\n", created)
}
```

ผลลัพธ์:

```
Kind ของ newValue: ptr
Kind ของ newValue.Elem(): struct
สร้างค่าใหม่ได้: &{Name:สมชาย Age:28}
```

จุดสำคัญที่ต้องสังเกต:

- `reflect.New(t)` คืนค่าเป็น **pointer เสมอ** (`Kind()` เป็น `reflect.Ptr`) เหมือนกับ `new()` ธรรมดา ดังนั้นค่าที่ได้จึง**addressable และ `CanSet()` เป็น `true` ทันที** หลังเรียก `.Elem()` โดยไม่ต้องผ่านขั้นตอนพิเศษเหมือนหัวข้อที่แล้ว
- `newValue.Interface().(*User)` คือขั้นตอนสุดท้ายที่แปลง `reflect.Value` กลับมาเป็นค่า Go ปกติ (`*User`) ผ่าน type assertion (ทบทวนจาก **Part 014**) เพื่อนำไปใช้งานต่อในโค้ดทั่วไปได้

เทคนิคนี้คือกลไกเบื้องหลังที่ทำให้ `json.Unmarshal` สามารถ `Unmarshal` JSON array เข้า `[]User` ได้ทั้งที่ไม่รู้จำนวนสมาชิกล่วงหน้า — มันใช้ `reflect.New` สร้าง `User` ตัวใหม่ขึ้นมาทีละตัวสำหรับแต่ละ element ใน JSON array แล้วเติมค่าเข้าไปด้วยกลไกเดียวกับที่เรียนในหัวข้อ 7

---

## 9. ข้อควรระวัง: ทำไม Reflection ช้าและทำลาย Type Safety ตอน Compile Time

มาพิสูจน์ด้วยตัวเลขจริงว่า reflection ช้ากว่าการเข้าถึง field ตรงๆ แค่ไหน (การวัดประสิทธิภาพแบบเป็นระบบด้วย `testing.B` จะเรียนเต็มรูปแบบใน **Part 034** บทนี้ขอใช้การจับเวลาแบบง่ายๆ ด้วย `time.Since` ให้เห็นภาพก่อน):

```go
package main

import (
	"fmt"
	"reflect"
	"time"
)

type Point struct {
	X, Y int
}

func directAccess(p *Point, n int) int {
	sum := 0
	for i := 0; i < n; i++ {
		sum += p.X + p.Y
	}
	return sum
}

func reflectAccess(p *Point, n int) int {
	v := reflect.ValueOf(p).Elem()
	sum := 0
	for i := 0; i < n; i++ {
		x := v.Field(0).Interface().(int)
		y := v.Field(1).Interface().(int)
		sum += x + y
	}
	return sum
}

func main() {
	p := &Point{X: 3, Y: 4}
	const n = 2_000_000

	start := time.Now()
	directAccess(p, n)
	directDur := time.Since(start)

	start = time.Now()
	reflectAccess(p, n)
	reflectDur := time.Since(start)

	fmt.Println("direct access:  ", directDur)
	fmt.Println("reflect access: ", reflectDur)
}
```

ผลลัพธ์ตัวอย่าง (ตัวเลขจริงจะต่างกันไปตามเครื่อง แต่สัดส่วนจะใกล้เคียงกันเสมอ):

```
direct access:   594.211µs
reflect access:  136.951919ms
```

ในเครื่องที่ทดสอบ **reflection ช้ากว่าการเข้าถึง field ตรงๆ ประมาณ 230 เท่า** นี่ไม่ใช่เรื่องบังเอิญ แต่เป็นผลจากธรรมชาติของ reflection เอง:

1. **ไม่มี inlining/optimization ระดับ compile time** — compiler ปกติจะ optimize การเข้าถึง `p.X` ให้เป็นแค่การอ่าน memory offset ตรงๆ แต่ reflection ต้องผ่านชั้น interface, type checking, และ dynamic dispatch หลายชั้นที่ runtime
2. **ต้องแปลงค่าไปมาระหว่าง `interface{}` กับ concrete type** — ทุกครั้งที่เรียก `.Interface()` หรือ `.Int()` มี overhead ของการ box/unbox ค่า
3. **Type checking เกิดที่ runtime แทน compile time** — ทุกการเรียก method อย่าง `SetString()`, `Field(i)` ต้องตรวจสอบ type ให้ถูกต้องก่อนทำงานจริงทุกครั้ง ต่างจากโค้ดปกติที่ compiler ตรวจสอบให้ครั้งเดียวตอน compile

นอกจาก performance แล้ว ข้อเสียที่สำคัญไม่แพ้กันคือ **การทำลาย type safety ตอน compile time**:

- โค้ดที่เขียนผิด เช่น เรียก `.Int()` กับ value ที่จริงๆ เป็น string จะ**ไม่มีทาง compile error** เลย compiler มองว่าโค้ดถูกต้องเสมอ (เพราะ signature เป็น `any`) แต่จะ **panic ทันทีตอนรัน** ซึ่งอาจไปเกิดตอน production แทนที่จะถูกจับได้ตั้งแต่ตอนพัฒนา
- Refactoring เสี่ยงกว่ามาก — ถ้าเปลี่ยนชื่อ field ของ struct โค้ดที่ใช้ `FieldByName("OldName")` (ที่เป็น string ธรรมดา) จะไม่มี compiler เตือนอะไรเลย ต่างจากโค้ดปกติที่ IDE และ compiler จะรายงาน error ทันทีถ้า field ไม่มีอยู่จริง
- โค้ดอ่านยากกว่ามาก เพราะ logic จริงถูกซ่อนอยู่หลัง string (ชื่อ field, ชื่อ tag) แทนที่จะเป็น identifier ที่ IDE ตามไปดูได้โดยตรง

**สรุปหลักการที่ต้องจำ**: ให้มองว่า reflection เป็นเครื่องมือ "ประตูหลังฉุกเฉิน" ของภาษา ไม่ใช่เครื่องมือที่ใช้ในโค้ด business logic ทั่วไป ถ้าเขียนโค้ดแล้วรู้สึกว่า "น่าจะใช้ reflect ตรงนี้สะดวกกว่า" ให้ถามตัวเองก่อนเสมอว่า generics (Part 028-029), interface (Part 013-014), หรือ code generation (`go generate` ที่เรียนใน Part 001) แก้ปัญหาเดียวกันได้โดยไม่ต้องใช้ reflect หรือไม่ — ส่วนใหญ่มักจะได้

---

## 10. เมื่อไหร่ควรใช้ Reflection จริงๆ

แม้จะมีข้อเสียเยอะ แต่ก็มีสถานการณ์ที่ reflection เป็นเครื่องมือที่ **เหมาะสมและจำเป็นจริงๆ** เพราะไม่มีทางเลือกอื่นที่ดีกว่า:

1. **Serialization library** — เช่น `encoding/json`, `encoding/xml` (Part 026) ที่ต้อง encode/decode struct **อะไรก็ได้**ที่ผู้ใช้ประกาศขึ้นมาเอง โดย library ไม่มีทางรู้ล่วงหน้าว่า struct จะหน้าตาเป็นอย่างไร — นี่คือกรณีคลาสสิกที่ generics ทำแทนไม่ได้ เพราะปัญหาคือ "ต้องเดินผ่าน field ของ struct ที่ไม่รู้จักล่วงหน้า" ไม่ใช่ "ต้องทำงานกับ type ที่รู้ล่วงหน้าหลายแบบ"
2. **ORM (Object-Relational Mapping)** — จะเรียนจริงจังใน **Part 074 (GORM เบื้องต้น)** และ **Part 075 (GORM ขั้นสูง)** ORM ต้อง map field ของ struct ไปยัง column ของตาราง database โดยอ่าน struct tag (เช่น `gorm:"column:user_name"`) แบบเดียวกับที่เราเพิ่งเขียนใน `PrintFields` ทุกประการ
3. **Dependency Injection framework** — บาง framework (เช่น Wire, Dig) ใช้ reflection ในการตรวจสอบ signature ของ constructor function เพื่อ "เดา" ว่าต้อง inject dependency อะไรเข้าไปบ้าง
4. **Validation library** — library ตรวจสอบความถูกต้องของข้อมูล (เช่น `go-playground/validator`) อ่าน struct tag อย่าง `validate:"required,min=3"` แล้วใช้ reflection ตรวจสอบค่าจริงของแต่ละ field ว่าผ่านเงื่อนไขหรือไม่
5. **Generic pretty-printer หรือ debugging tool** — เช่น `fmt.Printf("%+v", x)` เองก็ใช้ reflection ภายในเพื่อพิมพ์ค่าของ struct ที่ไม่รู้จักล่วงหน้าได้อย่างสวยงาม
6. **Deep equality** — `reflect.DeepEqual` ใช้เปรียบเทียบค่าของ struct/slice/map ที่ซับซ้อนแบบลึกทุกชั้น ซึ่งมีประโยชน์มากในการเขียน test (จะเจอการใช้งานจริงใน **Part 033-034**)

จุดร่วมของทุกกรณีข้างบนคือ: **มันคือการเขียน library หรือ framework ที่ต้องรองรับ type ของผู้ใช้แบบไม่รู้จักล่วงหน้า** ไม่ใช่การเขียน business logic ทั่วไปของแอปพลิเคชัน ถ้ากำลังเขียนโค้ดแอปพลิเคชันปกติ (handler, service, repository) แล้วคิดจะใช้ `reflect` ให้กลับไปพิจารณา interface หรือ generics ก่อนเสมอ

---

## สรุปสิ่งที่ได้เรียนในบทนี้

- **Reflection** คือความสามารถของโปรแกรมในการตรวจสอบ type และค่าของตัวเองที่ runtime ผ่าน package `reflect` — ใช้บ่อยในการเขียน library/framework แต่แทบไม่จำเป็นในโค้ด business logic ทั่วไป
- **Reflection ต่างจาก Generics (Part 028-029)**: generics รู้ type ตอน compile time (ปลอดภัยและเร็วกว่า) ส่วน reflection ค้นพบ type ตอน runtime (ยืดหยุ่นกว่าแต่ช้ากว่าและไม่ปลอดภัยเท่า) — ใช้ generics เมื่อรู้ type ล่วงหน้า ใช้ reflection เมื่อต้อง "เดิน" ผ่านโครงสร้างที่ไม่รู้จักล่วงหน้าจริงๆ เท่านั้น
- `reflect.TypeOf(x)` คืนข้อมูล type, `reflect.ValueOf(x)` คืนค่าจริงในรูปแบบที่จัดการได้ — ทั้งคู่ทำงานผ่านกลไก interface value คือคู่ (type, value) ที่เรียนจาก Part 013
- **`Kind()` vs `Type()`**: `Type()` คือชื่อเฉพาะของ type (รวม named type) ส่วน `Kind()` คือหมวดหมู่พื้นฐาน (struct, ptr, slice, float64, ...) — ในโค้ดจริงมักเช็คด้วย `Kind()` เสมอเพราะมีจำนวนค่าคงที่
- อ่าน struct field และ tag ที่ runtime ได้ผ่าน `t.NumField()`, `t.Field(i)`, `field.Tag.Get("...")` — นี่คือกลไกเดียวกับที่ `encoding/json` (Part 025) ใช้อ่าน tag `json:"..."` และที่ GORM (Part 074) จะใช้อ่าน tag `gorm:"..."`
- แก้ไขค่าผ่าน reflection ต้อง **pointer → `Elem()` → `CanSet()` เป็น true → `Set...()`** เสมอ ถ้าส่งค่าตรงๆ (ไม่ใช่ pointer) จะแก้ไขไม่ได้เลยเพราะเป็นแค่สำเนา
- **`reflect.New(t)`** สร้าง value ใหม่แบบไดนามิกจาก `reflect.Type` โดยไม่ต้องรู้ concrete type ล่วงหน้า คืนค่าเป็น pointer ที่ addressable ทันที — เป็นกลไกที่ `encoding/json` ใช้สร้าง element ใหม่ตอน `Unmarshal` เข้า slice ของ struct
- Reflection **ช้ากว่าการเข้าถึงค่าตรงๆ อย่างมาก** (วัดได้จริงเป็นร้อยเท่า) และ**ทำลาย type safety ตอน compile time** เพราะ error กลายเป็น panic ตอนรันแทนที่จะเป็น compile error
- ใช้ reflection เหมาะกับการเขียน serialization library, ORM, validation library, dependency injection และเครื่องมือ debugging เท่านั้น ไม่ใช่โค้ด business logic ทั่วไป

## แบบฝึกหัดท้ายบท

1. เขียนฟังก์ชัน `Describe(x any)` ที่พิมพ์ทั้ง `Type()` และ `Kind()` ของค่าที่ส่งเข้ามา ทดสอบกับ `int`, `string`, named type ของตัวเอง, struct, slice, map, และ pointer แล้วสังเกตความแตกต่างระหว่าง `Type()` กับ `Kind()` ในแต่ละกรณี
2. ขยายฟังก์ชัน `PrintFields` ในหัวข้อ 5 ให้รองรับ struct ที่มี struct ซ้อนอยู่ข้างใน (nested struct) โดยให้เดินเข้าไปพิมพ์ field ของ struct ที่ซ้อนอยู่ด้วย (ใบ้: เช็คว่า `value.Kind() == reflect.Struct` แล้วเรียกฟังก์ชันตัวเองซ้ำ)
3. เขียนฟังก์ชัน `SetFieldByTag(x any, tagValue string, newValue any) error` ที่รับ pointer ไปยัง struct, ค่าของ tag `label` ที่ต้องการค้นหา, และค่าใหม่ที่จะ set แล้วค้นหา field ที่มี tag ตรงกันเพื่อแก้ไขค่าให้ (ใช้ความรู้จากหัวข้อ 7)
4. เขียนโปรแกรมที่จับเวลาเปรียบเทียบการอ่านค่าด้วย reflection กับการอ่านค่าตรงๆ แบบในหัวข้อ 9 แต่เปลี่ยนจำนวนรอบ (`n`) เป็นค่าต่างๆ (100, 10,000, 1,000,000) แล้วสังเกตว่าสัดส่วนความช้าเปลี่ยนไปหรือไม่
5. ลองเขียนโค้ดที่ตั้งใจทำให้ reflection panic (เช่น เรียก `.Int()` กับค่าที่เป็น string, หรือเรียก `Set...()` กับค่าที่ `CanSet()` เป็น false) แล้วใช้ `recover()` จาก **Part 017** จับ panic เหล่านั้น พร้อมพิมพ์ข้อความ error ที่ได้ อธิบายว่าทำไมมันไม่ถูกจับเป็น compile error ตั้งแต่แรก
6. เขียนฟังก์ชัน `NewZeroValue(t reflect.Type) any` ที่ใช้ `reflect.New` (หัวข้อ 8) สร้าง value เปล่าของ type ใดๆ ที่ส่งเข้ามา แล้วทดสอบกับ `reflect.TypeOf(User{})`, `reflect.TypeOf(0)`, และ `reflect.TypeOf("")` พร้อมพิมพ์ผลลัพธ์ที่ได้ในแต่ละกรณี
7. ค้นคว้าเพิ่มเติม: อ่าน documentation ของ `reflect.DeepEqual` (`go doc reflect.DeepEqual`) แล้วลองเขียนโปรแกรมเปรียบเทียบ struct ที่มี slice และ map อยู่ข้างในด้วย `==` ธรรมดา (จะ compile error หรือ panic หรือไม่) เทียบกับการใช้ `reflect.DeepEqual` — อธิบายว่าทำไมถึงต่างกัน

---

**ต่อไป**: [Part 032 — แพ็กเกจ `context`](./032-context-package.md)
