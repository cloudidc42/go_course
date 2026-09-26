# Part 025: JSON ด้วย `encoding/json`

> ภาคที่ 2: ระดับกลาง (Intermediate) — ตอนที่ 10 จาก 20 (Part 16–35)

## สารบัญของบทนี้

1. JSON คืออะไร และทำไม Go ถึงจัดการมันได้ดีมาก
2. `json.Marshal`: แปลง struct เป็น JSON
3. Struct Field Tags: ควบคุมชื่อ field และพฤติกรรมด้วย `json:"..."`
4. กฎ Exported Fields Only: ผูกกับกฎ Capitalization จาก Part 001
5. `json.Unmarshal`: แปลง JSON กลับเป็น struct
6. JSON แบบไดนามิกด้วย `map[string]any`
7. Nested Struct และ Slice ของ Struct ใน JSON
8. Streaming ด้วย `json.NewEncoder`/`json.NewDecoder`
9. Custom Marshaling: `MarshalJSON`/`UnmarshalJSON`
10. จัดการ Field ที่ไม่รู้จัก (Unknown Fields)
11. สรุปสิ่งที่ได้เรียนในบทนี้
12. แบบฝึกหัดท้ายบท

---

## 1. JSON คืออะไร และทำไม Go ถึงจัดการมันได้ดีมาก

**JSON (JavaScript Object Notation)** คือรูปแบบข้อมูลที่ใช้แลกเปลี่ยนกันมากที่สุดในโลกซอฟต์แวร์ปัจจุบัน โดยเฉพาะใน REST API (ที่จะเรียนเจาะลึกใน **ภาคที่ 5: Web Development**) Go มี package มาตรฐานชื่อ `encoding/json` ที่ทำให้การแปลงข้อมูลไปมาระหว่าง Go struct กับ JSON เป็นเรื่องง่ายมาก โดยไม่ต้องพึ่ง library ภายนอกเลย — สอดคล้องกับจุดเด่นเรื่อง "Standard Library แข็งแกร่ง" ที่กล่าวถึงใน **Part 001**

กระบวนการหลักมีสองทิศทาง:

- **Marshal** (encode) — แปลงจาก Go value (struct, slice, map) **ไปเป็น** JSON bytes
- **Unmarshal** (decode) — แปลงจาก JSON bytes **กลับมาเป็น** Go value

---

## 2. `json.Marshal`: แปลง struct เป็น JSON

```go
package main

import (
	"encoding/json"
	"fmt"
)

type User struct {
	Name  string
	Age   int
	Email string
}

func main() {
	u := User{Name: "สมชาย", Age: 30, Email: "somchai@example.com"}
	data, err := json.Marshal(u)
	if err != nil {
		fmt.Println("marshal error:", err)
		return
	}
	fmt.Println(string(data))
}
```

`json.Marshal` คืนค่าเป็น `([]byte, error)` — ตามหลักการ error handling จาก **Part 015** ต้องเช็ค `err` เสมอ แม้ในทางปฏิบัติการ marshal struct ธรรมดาแทบไม่มีทาง error เลย (error มักเกิดจาก type ที่ marshal ไม่ได้ เช่น channel หรือ function type)

ถ้าต้องการผลลัพธ์ที่จัดรูปแบบสวยงามอ่านง่าย (เว้นบรรทัด/เยื้อง) ใช้ `json.MarshalIndent` แทน:

```go
pretty, _ := json.MarshalIndent(u, "", "  ")
fmt.Println(string(pretty))
```

---

## 3. Struct Field Tags: ควบคุมชื่อ field และพฤติกรรมด้วย `json:"..."`

ถ้าไม่ระบุ tag ใดๆ `encoding/json` จะใช้**ชื่อ field ตรงตัว**เป็น key ใน JSON (เช่น `FirstName` → `"FirstName"`) ซึ่งมักไม่ตรงกับ convention ของ JSON ที่นิยมใช้ `snake_case` หรือ `camelCase` ตัวเล็ก เราจึงกำหนดชื่อ key เองผ่าน **struct tag**:

```go
type User struct {
	Name     string `json:"name"`
	Age      int    `json:"age"`
	Email    string `json:"email,omitempty"`
	password string // unexported - จะไม่ถูก marshal
}
```

```go
u := User{Name: "สมชาย", Age: 30, password: "secret"}
data, _ := json.Marshal(u)
fmt.Println(string(data))

u2 := User{Name: "สมหญิง", Age: 25, Email: "somying@example.com"}
data2, _ := json.Marshal(u2)
fmt.Println(string(data2))

pretty, _ := json.MarshalIndent(u2, "", "  ")
fmt.Println(string(pretty))
```

ผลลัพธ์:

```
{"name":"สมชาย","age":30}
{"name":"สมหญิง","age":25,"email":"somying@example.com"}
{
  "name": "สมหญิง",
  "age": 25,
  "email": "somying@example.com"
}
```

สังเกตว่า `u` (ที่ไม่ได้ตั้งค่า `Email`) **ไม่มี key `email` ปรากฏใน JSON เลย** เพราะ tag `omitempty` — นี่คือ modifier สำคัญที่ต้องเข้าใจ

### ตัวเลือกใน struct tag ที่ใช้บ่อย

| Tag | ความหมาย |
|---|---|
| `json:"name"` | ใช้ `name` เป็น key แทนชื่อ field เดิม |
| `json:"name,omitempty"` | ใช้ชื่อ `name` และ**ข้าม field นี้ไปเลย**ถ้าค่าเป็น zero value (ทบทวน zero value จาก **Part 003**: `0`, `""`, `false`, `nil`, slice/map ว่าง) |
| `json:"-"` | **ไม่ marshal/unmarshal field นี้เลย** แม้จะเป็น exported field ก็ตาม |
| `json:",omitempty"` | ใช้ชื่อ field เดิม (ไม่เปลี่ยนชื่อ) แต่ข้ามถ้าเป็น zero value |
| `json:"name,string"` | บังคับให้ field ที่เป็นตัวเลขถูก encode เป็น JSON string (ใช้น้อย แต่มีประโยชน์เมื่อต้อง compat กับระบบที่ต้องการตัวเลขเป็น string) |

**ข้อควรระวังเรื่อง `omitempty`**: `omitempty` เช็คแค่ว่าเป็น **zero value ของ type นั้น** หรือไม่ ไม่ได้เช็คว่า field นั้น "ถูกตั้งค่ามาจากผู้ใช้หรือไม่" ดังนั้นถ้า field เป็น `int` ที่ค่าจริงคือ `0` โดยตั้งใจ (เช่น คะแนนเริ่มต้น 0 คะแนน) `omitempty` ก็จะข้าม field นั้นไปเหมือนกัน ทำให้แยกไม่ออกระหว่าง "ไม่ได้ส่งค่ามา" กับ "ส่งค่า 0 มาจริงๆ" — ถ้าต้องการแยกสองกรณีนี้ให้ชัดเจน ควรใช้ pointer type (เช่น `*int`) แทน `int` ธรรมดา เพราะ `nil` กับ `0` เป็นคนละค่ากัน

---

## 4. กฎ Exported Fields Only: ผูกกับกฎ Capitalization จาก Part 001

จำกฎ Capitalization ที่เรียนใน **Part 001** ได้ไหม: **ชื่อที่ขึ้นต้นด้วยตัวพิมพ์ใหญ่คือ exported (เข้าถึงได้จากภายนอก package) ชื่อที่ขึ้นต้นด้วยตัวพิมพ์เล็กคือ unexported (เข้าถึงได้เฉพาะใน package เดียวกัน)**

`encoding/json` ใช้กลไกนี้เป็นกฎเหล็กในการตัดสินใจว่า field ไหนจะถูก marshal/unmarshal: **เฉพาะ exported field เท่านั้นที่ package `encoding/json` จะประมวลผลได้** เพราะ `encoding/json` เข้าถึง field ของ struct ผ่านกลไก `reflect` (ที่จะเรียนเจาะลึกใน **Part 031**) ซึ่งมีข้อจำกัดในตัวภาษาเองว่า **ไม่สามารถอ่าน/เขียนค่าของ unexported field จาก package ภายนอกได้เลย** ไม่ว่าจะพยายามอย่างไรก็ตาม

ดังตัวอย่างในหัวข้อก่อนหน้า field `password` (ตัวพิมพ์เล็ก) **ไม่ปรากฏใน JSON เลย** แม้จะไม่มี tag `json:"-"` กำกับไว้ก็ตาม เพราะมันเป็น unexported field โดยธรรมชาติ

```go
type NoTag struct {
	FirstName string
	lastName  string // unexported
}

n := NoTag{FirstName: "สมชาย", lastName: "ใจดี"}
data, _ := json.Marshal(n)
fmt.Println(string(data))
```

ผลลัพธ์:

```
{"FirstName":"สมชาย"}
```

`lastName` หายไปโดยสิ้นเชิง และ `FirstName` (ที่ไม่มี tag) ก็ถูกใช้ชื่อ field ตรงตัวเป็น key — **นี่คือเหตุผลที่ทำให้ต้องใส่ struct tag เสมอในโค้ด production** เพื่อควบคุมชื่อ key ให้ตรงกับ convention ของ JSON API ที่ต้องการ (โดยทั่วไปคือ `snake_case`)

**สรุปกฎทอง**: ถ้าต้องการให้ field ปรากฏใน JSON ต้องตั้งชื่อ field ด้วยตัวพิมพ์ใหญ่นำหน้าเสมอ (exported) แล้วค่อยใช้ struct tag ควบคุมชื่อ key ที่จะแสดงใน JSON ให้เป็นรูปแบบที่ต้องการ

---

## 5. `json.Unmarshal`: แปลง JSON กลับเป็น struct

```go
jsonStr := `{"name":"วิชัย","age":40,"email":"wichai@example.com"}`
var u3 User
if err := json.Unmarshal([]byte(jsonStr), &u3); err != nil {
	fmt.Println("unmarshal error:", err)
	return
}
fmt.Printf("%+v\n", u3)
```

ผลลัพธ์:

```
{Name:วิชัย Age:40 Email:wichai@example.com password:}
```

**จุดสำคัญที่ต้องจำ**:

1. `json.Unmarshal` รับ **pointer** (`&u3`) เสมอ เพราะต้องแก้ไขค่าของ `u3` โดยตรง — เหตุผลเดียวกับที่เรียนเรื่อง pointer ใน **Part 010** และเหมือนกับกลไกของ `fmt.Scan` ใน **Part 021**
2. การ match ระหว่าง JSON key กับ struct field ใช้ **struct tag ก่อน** ถ้าไม่มี tag จะ match แบบ **case-insensitive** กับชื่อ field (เช่น JSON key `"name"` จะ match กับ field `Name` ได้แม้ตัวพิมพ์ต่างกัน)
3. Key ใน JSON ที่ **ไม่ตรงกับ field ไหนเลย** จะถูก**ข้ามไปเงียบๆ** โดย default (ไม่ error) — พฤติกรรมนี้จะพูดถึงวิธีเปลี่ยนในหัวขัด 10

---

## 6. JSON แบบไดนามิกด้วย `map[string]any`

บางครั้งเราไม่รู้โครงสร้างของ JSON ล่วงหน้า (เช่น รับข้อมูลจาก API ภายนอกที่โครงสร้างไม่แน่นอน หรือกำลังเขียนเครื่องมือสำรวจ JSON ทั่วไป) กรณีนี้สามารถ unmarshal เข้า `map[string]any` แทนการประกาศ struct ตายตัวได้ (`any` คือ alias ของ `interface{}` ที่เรียนใน **Part 014** เพิ่มเข้ามาตั้งแต่ Go 1.18)

```go
package main

import (
	"encoding/json"
	"fmt"
)

func main() {
	jsonStr := `{"name":"สมชาย","age":30,"active":true,"tags":["go","dev"],"address":{"city":"Bangkok"}}`

	var data map[string]any
	if err := json.Unmarshal([]byte(jsonStr), &data); err != nil {
		fmt.Println("error:", err)
		return
	}
	fmt.Println("name:", data["name"])
	fmt.Println("age:", data["age"]) // จะเป็น float64 ไม่ใช่ int!
	fmt.Printf("age type: %T\n", data["age"])
	fmt.Println("active:", data["active"])

	tags := data["tags"].([]any)
	for _, t := range tags {
		fmt.Println("tag:", t)
	}

	address := data["address"].(map[string]any)
	fmt.Println("city:", address["city"])
}
```

ผลลัพธ์:

```
name: สมชาย
age: 30
age type: float64
active: true
tag: go
tag: dev
city: Bangkok
```

### กับดักสำคัญ: ตัวเลขทั้งหมดกลายเป็น `float64`

**นี่คือจุดที่ต้องระวังมากที่สุด**: เมื่อ unmarshal JSON เข้า `map[string]any` (หรือ `any` เปล่าๆ) **ตัวเลข JSON ทุกตัวจะถูกแปลงเป็น `float64` เสมอ** ไม่ว่าค่าเดิมจะเป็นจำนวนเต็มหรือทศนิยมก็ตาม เพราะ JSON spec เองไม่ได้แยกประเภทตัวเลขระหว่าง int กับ float ไว้ชัดเจน `encoding/json` จึงเลือก `float64` เป็นค่ากลางที่ครอบคลุมทุกกรณี

ผลที่ตามมาคือถ้าต้องการใช้ค่าเป็น `int` จริงๆ ต้อง **type assertion เป็น `float64` ก่อน แล้วค่อยแปลงเป็น `int`** (ทบทวน type assertion จาก **Part 014**):

```go
ageFloat := data["age"].(float64)
age := int(ageFloat)
```

การ type assertion ที่เห็นในตัวอย่างข้างบน (`data["tags"].([]any)` และ `data["address"].(map[string]any)`) ก็ต้องระวังเช่นกัน — array ใน JSON กลายเป็น `[]any` และ object กลายเป็น `map[string]any` เสมอ ถ้า assertion ผิด type จะ panic ทันที ในโค้ด production ควรใช้ comma-ok idiom (`v, ok := data["tags"].([]any)`) เพื่อเช็คก่อนใช้งานเสมอ

---

## 7. Nested Struct และ Slice ของ Struct ใน JSON

ในทางปฏิบัติ ข้อมูลจริงมักซับซ้อนกว่า flat struct ธรรมดา — มี struct ซ้อน struct และ slice ของ struct ปนกัน `encoding/json` จัดการเรื่องนี้ได้อัตโนมัติโดยไม่ต้องเขียนโค้ดพิเศษเพิ่ม:

```go
package main

import (
	"encoding/json"
	"fmt"
)

type Address struct {
	City    string `json:"city"`
	ZipCode string `json:"zip_code"`
}

type Order struct {
	ID    int      `json:"id"`
	Items []string `json:"items"`
}

type Customer struct {
	Name    string  `json:"name"`
	Address Address `json:"address"`
	Orders  []Order `json:"orders"`
}

func main() {
	c := Customer{
		Name:    "สมชาย",
		Address: Address{City: "Bangkok", ZipCode: "10110"},
		Orders: []Order{
			{ID: 1, Items: []string{"apple", "banana"}},
			{ID: 2, Items: []string{"cherry"}},
		},
	}

	data, _ := json.MarshalIndent(c, "", "  ")
	fmt.Println(string(data))
}
```

ผลลัพธ์:

```
{
  "name": "สมชาย",
  "address": {
    "city": "Bangkok",
    "zip_code": "10110"
  },
  "orders": [
    {
      "id": 1,
      "items": [
        "apple",
        "banana"
      ]
    },
    {
      "id": 2,
      "items": [
        "cherry"
      ]
    }
  ]
}
```

การ `Unmarshal` กลับก็ทำงานในทิศทางตรงข้ามได้อย่างสมมาตร — แค่ประกาศ struct ที่มีโครงสร้างตรงกับ JSON (nested struct field และ slice field) แล้ว `json.Unmarshal(&customer)` ก็จะเติมค่าให้ครบทุกชั้นโดยอัตโนมัติ ไม่ว่าจะซ้อนกันกี่ระดับก็ตาม

---

## 8. Streaming ด้วย `json.NewEncoder`/`json.NewDecoder`

`json.Marshal`/`json.Unmarshal` ทำงานกับข้อมูลที่อยู่ใน memory ทั้งก้อน (`[]byte`) แต่ในสถานการณ์จริงหลายครั้งเราต้องการอ่าน/เขียน JSON โดยตรงจาก/ไปยัง **`io.Reader`/`io.Writer`** (ที่เรียนใน **Part 024**) เช่น เขียน JSON ลงไฟล์โดยตรง หรืออ่าน JSON จาก HTTP request body โดยไม่ต้องโหลดทั้งก้อนเข้า memory ก่อน — `json.NewEncoder` และ `json.NewDecoder` ถูกออกแบบมาเพื่อกรณีนี้โดยเฉพาะ

```go
package main

import (
	"bytes"
	"encoding/json"
	"fmt"
	"strings"
)

func main() {
	c := Customer{ /* ... เหมือนตัวอย่างก่อนหน้า ... */ }

	// Streaming Encoder -> io.Writer (ตัวอย่างนี้ใช้ bytes.Buffer แทนไฟล์จริง)
	var buf bytes.Buffer
	enc := json.NewEncoder(&buf)
	enc.SetIndent("", "  ")
	if err := enc.Encode(c); err != nil {
		fmt.Println("encode error:", err)
	}
	fmt.Print(buf.String())

	// Streaming Decoder <- io.Reader
	reader := strings.NewReader(buf.String())
	dec := json.NewDecoder(reader)
	var c2 Customer
	if err := dec.Decode(&c2); err != nil {
		fmt.Println("decode error:", err)
	}
	fmt.Printf("decoded: %+v\n", c2)
}
```

ผลลัพธ์ (ส่วนของ decode):

```
decoded: {Name:สมชาย Address:{City:Bangkok ZipCode:10110} Orders:[{ID:1 Items:[apple banana]} {ID:2 Items:[cherry]}]}
```

### เมื่อไรควรใช้ Encoder/Decoder แทน Marshal/Unmarshal

- **เขียน JSON ลงไฟล์โดยตรง**: `json.NewEncoder(f).Encode(v)` เขียนตรงไปยัง `*os.File` ได้เลย ไม่ต้อง `Marshal` แล้วค่อย `os.WriteFile` แยกสองขั้นตอน
- **อ่าน JSON จาก HTTP request/response body**: (จะเจอบ่อยมากใน **ภาคที่ 5**) `json.NewDecoder(r.Body).Decode(&v)` อ่านตรงจาก stream โดยไม่ต้องอ่านทั้งก้อนเข้า memory ก่อนด้วย `io.ReadAll`
- **ข้อมูลขนาดใหญ่มาก**: Decoder ประมวลผลแบบ stream ทำให้ไม่ต้องโหลดข้อมูลทั้งหมดเข้า memory พร้อมกัน ต่างจาก `Unmarshal` ที่ต้องมี `[]byte` ทั้งก้อนอยู่ใน memory ก่อนเริ่มแปลง

ในทางกลับกัน ถ้าข้อมูลมีขนาดเล็กและอยู่ใน memory อยู่แล้ว (เช่น string หรือ `[]byte` ที่มีอยู่แล้ว) การใช้ `Marshal`/`Unmarshal` ตรงๆ จะเรียบง่ายและอ่านง่ายกว่า

---

## 9. Custom Marshaling: `MarshalJSON`/`UnmarshalJSON`

บางครั้งรูปแบบ default ของ `encoding/json` ไม่ตรงกับที่ต้องการ เช่น การจัดการ `time.Time` — ค่า default จะ marshal เป็นรูปแบบ RFC3339 (`"2024-03-15T00:00:00Z"`) เสมอ แต่ถ้าต้องการรูปแบบอื่น (เช่น `"15/03/2024"` แบบไทย) ต้องเขียน **custom marshaling** เอง โดย implement interface สองตัวนี้:

```go
type Marshaler interface {
	MarshalJSON() ([]byte, error)
}

type Unmarshaler interface {
	UnmarshalJSON([]byte) error
}
```

เมื่อ type ใด implement interface เหล่านี้ (ทบทวนหลักการ interface satisfaction จาก **Part 013**) `encoding/json` จะเรียก method เหล่านี้แทนพฤติกรรม default โดยอัตโนมัติ — กลไกเดียวกับที่ `fmt` เรียก `String()` ของ `Stringer` โดยอัตโนมัติที่เรียนไปใน **Part 021**

```go
package main

import (
	"encoding/json"
	"fmt"
	"time"
)

type ThaiDate struct {
	time.Time
}

const thaiDateLayout = "02/01/2006"

func (d ThaiDate) MarshalJSON() ([]byte, error) {
	s := d.Time.Format(thaiDateLayout)
	return json.Marshal(s)
}

func (d *ThaiDate) UnmarshalJSON(data []byte) error {
	var s string
	if err := json.Unmarshal(data, &s); err != nil {
		return err
	}
	t, err := time.Parse(thaiDateLayout, s)
	if err != nil {
		return err
	}
	d.Time = t
	return nil
}

type Event struct {
	Name string   `json:"name"`
	Date ThaiDate `json:"date"`
}

func main() {
	e := Event{Name: "ประชุมทีม", Date: ThaiDate{time.Date(2024, 3, 15, 0, 0, 0, 0, time.UTC)}}
	data, _ := json.Marshal(e)
	fmt.Println(string(data))

	jsonStr := `{"name":"อบรม Go","date":"25/12/2024"}`
	var e2 Event
	if err := json.Unmarshal([]byte(jsonStr), &e2); err != nil {
		fmt.Println("unmarshal error:", err)
		return
	}
	fmt.Printf("name=%s date=%s\n", e2.Name, e2.Date.Format("2006-01-02"))
}
```

ผลลัพธ์:

```
{"name":"ประชุมทีม","date":"15/03/2024"}
name=อบรม Go date=2024-12-25
```

จุดที่ควรสังเกต:

- `ThaiDate` embed `time.Time` ไว้ (embedding จะเรียนเจาะลึกใน **Part 030**) ทำให้ยังใช้ method ของ `time.Time` เช่น `.Format()` ได้ตามปกติผ่าน `e2.Date.Format(...)`
- `MarshalJSON()` ใช้ **value receiver** ส่วน `UnmarshalJSON()` ใช้ **pointer receiver เสมอ** เพราะต้องแก้ไขค่าของ `d` โดยตรง (ทบทวนความแตกต่างของ value กับ pointer receiver จาก **Part 012**) — ถ้าใช้ value receiver กับ `UnmarshalJSON` การแก้ไขค่าจะไม่ส่งผลกลับไปยัง struct ต้นทางเลย
- ภายใน `MarshalJSON`/`UnmarshalJSON` เราเรียก `json.Marshal`/`json.Unmarshal` ซ้อนเข้าไปอีกชั้นเพื่อจัดการกับ string ธรรมดา (ใส่ quote ให้ถูกต้องตาม JSON string format) แทนที่จะสร้าง `[]byte` เองด้วยมือ ซึ่งเป็นแนวทางที่ปลอดภัยและแนะนำ

---

## 10. จัดการ Field ที่ไม่รู้จัก (Unknown Fields)

โดย default `json.Unmarshal` และ `json.Decoder.Decode` จะ **เพิกเฉยต่อ JSON key ที่ไม่ตรงกับ field ไหนใน struct เลยแบบเงียบๆ** ไม่มี error หรือคำเตือนใดๆ:

```go
type Strict struct {
	Name string `json:"name"`
}

var s Strict
json.Unmarshal([]byte(`{"name":"test","extra":"field"}`), &s) // ไม่ error แม้มี "extra" ที่ไม่รู้จัก
```

ในบางสถานการณ์ (เช่น ต้องการตรวจจับว่า client ส่ง field ที่สะกดผิดหรือ API version ไม่ตรงกัน) เราต้องการให้ **error ทันทีถ้าเจอ field ที่ไม่รู้จัก** ซึ่งทำได้ผ่าน `Decoder.DisallowUnknownFields()` (มีเฉพาะใน `Decoder` เท่านั้น ไม่มีใน `json.Unmarshal` ตรงๆ):

```go
dec := json.NewDecoder(strings.NewReader(`{"name":"test","extra":"field"}`))
dec.DisallowUnknownFields()
var s Strict
err := dec.Decode(&s)
fmt.Println("strict decode error:", err)
```

ผลลัพธ์:

```
strict decode error: json: unknown field "extra"
```

**แนวทางปฏิบัติ**: ถ้าเขียน API ที่ต้องการความเข้มงวดสูง (strict validation) เช่น รับ config file หรือ request body ที่ต้องตรวจสอบความถูกต้องอย่างเคร่งครัด ควรใช้ `Decoder` คู่กับ `DisallowUnknownFields()` แทนการใช้ `json.Unmarshal` ตรงๆ เพื่อจับข้อผิดพลาดจากการสะกดผิดหรือโครงสร้างข้อมูลที่ผิดคาดได้ตั้งแต่เนิ่นๆ

---

## สรุปสิ่งที่ได้เรียนในบทนี้

- `json.Marshal`/`json.MarshalIndent` แปลง Go value เป็น JSON bytes, `json.Unmarshal` แปลงกลับ (ต้องส่ง pointer เสมอ)
- Struct tag `json:"name,omitempty"` ควบคุมชื่อ key และการข้าม field ที่เป็น zero value, `json:"-"` ไม่ marshal field นั้นเลย
- **เฉพาะ exported field เท่านั้น** ที่ถูก marshal/unmarshal — ผูกกับกฎ Capitalization จาก **Part 001** โดยตรง เพราะ `encoding/json` ใช้ `reflect` ซึ่งอ่าน unexported field ข้าม package ไม่ได้
- `map[string]any` ใช้กับ JSON ที่ไม่รู้โครงสร้างล่วงหน้าได้ แต่**ตัวเลขทุกตัวจะกลายเป็น `float64` เสมอ** ต้อง type assertion และแปลงเองถ้าต้องการ `int`
- Nested struct และ slice ของ struct ทำงานได้อัตโนมัติทั้งสองทิศทางโดยไม่ต้องเขียนโค้ดเพิ่ม
- `json.NewEncoder`/`json.NewDecoder` ทำงานกับ `io.Writer`/`io.Reader` โดยตรง เหมาะกับไฟล์, HTTP body, หรือข้อมูลขนาดใหญ่ที่ไม่อยากโหลดเข้า memory ทั้งก้อน
- Custom marshaling ทำผ่าน interface `Marshaler`/`Unmarshaler` (method `MarshalJSON`/`UnmarshalJSON`) — `UnmarshalJSON` ต้องใช้ pointer receiver เสมอ
- Field ที่ไม่รู้จักถูกข้ามเงียบๆ โดย default ถ้าต้องการเข้มงวด ใช้ `Decoder.DisallowUnknownFields()`

## แบบฝึกหัดท้ายบท

1. ประกาศ struct `Product` ที่มี `Name`, `Price float64`, `InStock bool`, `Tags []string` พร้อม struct tag ที่เหมาะสม (ใช้ `snake_case`) แล้วลอง marshal เป็น JSON ทั้งแบบมีและไม่มี `omitempty`
2. เขียนโปรแกรมที่ unmarshal JSON array ของ object (เช่น `[{"name":"a"},{"name":"b"}]`) เข้า `[]Product`
3. เขียนโปรแกรมที่รับ JSON string ที่ไม่รู้โครงสร้างล่วงหน้า แล้ว unmarshal เข้า `map[string]any` จากนั้นเขียนฟังก์ชันที่ดึงค่าตัวเลขออกมาเป็น `int` อย่างปลอดภัย (เช็ค type assertion ด้วย comma-ok ก่อนแปลง)
4. เขียน custom type `Cents int` ที่มี `MarshalJSON` แปลงค่าเป็นรูปแบบเงินบาท เช่น `1050` (สตางค์) → `"10.50"` (string) และเขียน `UnmarshalJSON` แปลงกลับ
5. เขียนโปรแกรมที่มี struct ซ้อนกัน 3 ชั้น (เช่น `Company` มี `[]Department` และแต่ละ `Department` มี `[]Employee`) แล้ว marshal/unmarshal ทดสอบว่าทำงานถูกต้องทุกชั้น
6. เขียนโปรแกรมที่ใช้ `json.NewDecoder` อ่าน JSON จากไฟล์ (ใช้ความรู้ `os.Open` จาก **Part 024**) พร้อมเปิดใช้ `DisallowUnknownFields()` แล้วทดสอบว่าถ้าไฟล์มี field แปลกปลอมจะ error ตามที่คาดหวัง

---

**ต่อไป**: [Part 026 — XML และ YAML ใน Go](./026-xml-and-yaml.md)
