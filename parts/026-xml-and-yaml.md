# Part 026: XML และ YAML ใน Go

> ภาคที่ 2: ระดับกลาง (Intermediate) — ตอนที่ 11 จาก 20 (Part 16–35)

## สารบัญของบทนี้

1. ทำไมต้องรู้จัก XML และ YAML ทั้งที่มี JSON แล้ว
2. `encoding/xml`: `xml.Marshal` แปลง struct เป็น XML
3. Struct Field Tags ของ XML: `xml:"name,attr"` และรูปแบบอื่นๆ
4. Nested XML Elements และ Slice
5. `xml.Unmarshal`: แปลง XML กลับเป็น struct
6. เทียบ XML Tags กับ JSON Tags ที่เรียนใน Part 025
7. YAML คืออะไร และทำไมไม่อยู่ใน Standard Library
8. ติดตั้ง Third-Party Package: `gopkg.in/yaml.v3`
9. Struct Tags และ Marshal/Unmarshal ด้วย `yaml.v3`
10. เมื่อไรควรเลือก YAML แทน JSON ในทางปฏิบัติ
11. สรุปสิ่งที่ได้เรียนในบทนี้
12. แบบฝึกหัดท้ายบท

---

## 1. ทำไมต้องรู้จัก XML และ YAML ทั้งที่มี JSON แล้ว

ใน **Part 025** เราเรียน `encoding/json` ไปแล้วว่าเป็นรูปแบบข้อมูลที่นิยมที่สุดในโลก REST API ปัจจุบัน แต่โลกซอฟต์แวร์จริงยังมีระบบจำนวนมากที่ใช้รูปแบบข้อมูลอื่นอยู่ และ Go Developer มืออาชีพจำเป็นต้องอ่าน/เขียนข้อมูลเหล่านั้นได้:

- **XML** ยังคงเป็นมาตรฐานในระบบ enterprise เก่า, SOAP web service, ไฟล์ config ของหลาย framework (เช่น Maven `pom.xml`), และ RSS/Atom feed
- **YAML** กลายเป็นมาตรฐาน**โดยพฤตินัย**ของโลก DevOps สมัยใหม่ — Kubernetes manifests, Docker Compose, GitHub Actions workflow, Ansible playbook ล้วนเขียนเป็น YAML ทั้งสิ้น

ข่าวดีคือ Go มี `encoding/xml` อยู่ใน standard library พร้อมใช้งานทันที (สอดคล้องกับจุดเด่น "Standard Library แข็งแกร่ง" จาก **Part 001**) ส่วน YAML นั้น **ไม่มีอยู่ใน standard library** — นี่คือจุดที่บทนี้จะพาไปรู้จักการใช้ **third-party package** เป็นครั้งแรกในหลักสูตร ซึ่งเป็นทักษะสำคัญที่ Go Developer ทุกคนต้องมี เพราะระบบนิเวศ (ecosystem) ของ Go พึ่งพา community package คุณภาพสูงเป็นจำนวนมาก

---

## 2. `encoding/xml`: `xml.Marshal` แปลง struct เป็น XML

การใช้งานเบื้องต้นของ `encoding/xml` มีรูปแบบคล้ายกับ `encoding/json` มาก เพราะทั้งคู่ใช้หลักการเดียวกัน: อ่าน struct ผ่าน `reflect` (จะเรียนเจาะลึกใน **Part 031**) แล้วแปลงตาม struct tag

```go
package main

import (
	"encoding/xml"
	"fmt"
)

type Book struct {
	Title  string  `xml:"title"`
	Author string  `xml:"author"`
	Price  float64 `xml:"price"`
}

func main() {
	b := Book{Title: "Learning Go", Author: "สมชาย ใจดี", Price: 350.50}

	data, err := xml.MarshalIndent(b, "", "  ")
	if err != nil {
		fmt.Println("marshal error:", err)
		return
	}
	fmt.Println(string(data))
}
```

ผลลัพธ์:

```
<Book>
  <title>Learning Go</title>
  <author>สมชาย ใจดี</author>
  <price>350.5</price>
</Book>
```

สังเกตว่า **ชื่อ root element** (`<Book>`) มาจากชื่อ Go type ตรงๆ (ไม่มี tag กำกับ) ในขณะที่ field ภายในใช้ชื่อจาก tag ที่กำหนด — นี่คือความแตกต่างสำคัญข้อแรกจาก JSON ที่จะอธิบายเพิ่มในหัวข้อที่ 6

เหมือนกับ `json.Marshal`, ฟังก์ชัน `xml.Marshal`/`xml.MarshalIndent` คืนค่า `([]byte, error)` และควรเช็ค `err` เสมอตามหลักการจาก **Part 015**

---

## 3. Struct Field Tags ของ XML: `xml:"name,attr"` และรูปแบบอื่นๆ

XML มีแนวคิดที่ JSON ไม่มี นั่นคือ **attribute** (คุณสมบัติที่แนบอยู่ในแท็กเปิด เช่น `<book id="1">`) ต่างจาก **element** (แท็กลูกที่แยกออกมาต่างหาก เช่น `<title>...</title>`) การควบคุมว่า field ไหนจะเป็น attribute หรือ element ทำผ่าน struct tag ของ `encoding/xml`:

| Tag | ความหมาย |
|---|---|
| `xml:"name"` | ใช้ `name` เป็นชื่อ element (ค่า default ถ้าไม่มี tag คือใช้ชื่อ field ตรงตัว) |
| `xml:"name,attr"` | เก็บค่านี้เป็น **attribute** ในแท็กเปิดของ element แม่ ไม่ใช่ element ลูก |
| `xml:"-"` | ไม่ marshal/unmarshal field นี้เลย (เหมือน JSON) |
| `xml:",chardata"` | เก็บเนื้อหาข้อความที่อยู่ระหว่างแท็กเปิด-ปิดโดยตรง โดยไม่สร้าง element ลูก |
| `xml:"parent>child"` | สร้าง element ลูกซ้อนกัน (nested) โดยไม่ต้องประกาศ struct แยก |
| `xml:",omitempty"` | ข้าม field นี้ถ้าเป็น zero value เหมือนหลักการใน **Part 025** |

`XMLName xml.Name` เป็น field พิเศษ: ถ้าประกาศ field ชื่อนี้ไว้ใน struct จะใช้กำหนด**ชื่อ root element เอง** แทนที่จะปล่อยให้ Go ใช้ชื่อ type ตรงๆ

```go
package main

import (
	"encoding/xml"
	"fmt"
)

type Book struct {
	XMLName xml.Name `xml:"book"`
	ID      int      `xml:"id,attr"`
	Title   string   `xml:"title"`
	Author  string   `xml:"author"`
	Price   float64  `xml:"price"`
}

func main() {
	b := Book{ID: 1, Title: "Learning Go", Author: "สมชาย ใจดี", Price: 350.50}

	data, err := xml.MarshalIndent(b, "", "  ")
	if err != nil {
		fmt.Println("marshal error:", err)
		return
	}
	fmt.Println(xml.Header + string(data))
}
```

ผลลัพธ์:

```
<?xml version="1.0" encoding="UTF-8"?>
<book id="1">
  <title>Learning Go</title>
  <author>สมชาย ใจดี</author>
  <price>350.5</price>
</book>
```

จุดที่ควรสังเกต:

- `XMLName xml.Name` ทำให้ root element เปลี่ยนจาก `<Book>` เป็น `<book>` ตามที่ tag `xml:"book"` กำหนด
- `ID int` มี tag `,attr` จึงกลายเป็น attribute `id="1"` ในแท็กเปิด ไม่ใช่ element ลูก `<id>1</id>`
- `xml.Header` เป็นค่าคงที่ string ที่เก็บ `<?xml version="1.0" encoding="UTF-8"?>\n` ไว้ให้ใช้ต่อกับผลลัพธ์ marshal ได้ทันที เพราะ `xml.Marshal` เองไม่ได้ใส่ XML declaration บรรทัดนี้ให้อัตโนมัติ

---

## 4. Nested XML Elements และ Slice

XML ในโลกจริงมักมีโครงสร้างซ้อนกันหลายชั้น เช่นเดียวกับ JSON ที่เรียนใน **Part 025** `encoding/xml` รองรับทั้ง nested struct และ slice โดยอัตโนมัติ:

```go
package main

import (
	"encoding/xml"
	"fmt"
)

type Address struct {
	City string `xml:"city"`
	Zip  string `xml:"zip"`
}

type Order struct {
	XMLName  xml.Name `xml:"order"`
	Customer string   `xml:"customer"`
	Address  Address  `xml:"address"`
	Items    []string `xml:"items>item"`
}

func main() {
	o := Order{
		Customer: "สมหญิง",
		Address:  Address{City: "Chiang Mai", Zip: "50000"},
		Items:    []string{"apple", "banana", "cherry"},
	}

	data, _ := xml.MarshalIndent(o, "", "  ")
	fmt.Println(string(data))
}
```

ผลลัพธ์:

```
<order>
  <customer>สมหญิง</customer>
  <address>
    <city>Chiang Mai</city>
    <zip>50000</zip>
  </address>
  <items>
    <item>apple</item>
    <item>banana</item>
    <item>cherry</item>
  </items>
</order>
```

จุดที่น่าสนใจคือ tag `xml:"items>item"` — เครื่องหมาย `>` บอกให้ `encoding/xml` สร้าง element ครอบ `<items>` แล้วค่อยใส่แต่ละสมาชิกของ slice เป็น `<item>` ซ้อนอยู่ข้างใน โดยไม่ต้องประกาศ struct กลางแยกต่างหาก (ถ้าไม่ใส่ `>item` แต่ใส่แค่ `xml:"item"` เฉยๆ ผลลัพธ์จะได้ `<item>` เรียงกันตรงๆ โดยไม่มี `<items>` ครอบ)

Struct ซ้อนธรรมดาอย่าง `Address` ไม่ต้องทำอะไรพิเศษเลย — `encoding/xml` จะสร้าง element ลูกตามชื่อ tag ของ field `Address` โดยอัตโนมัติ และไปวนอ่าน field ภายใน `Address` ต่อตาม tag ของมันเอง

---

## 5. `xml.Unmarshal`: แปลง XML กลับเป็น struct

ทิศทางตรงข้ามทำงานแบบสมมาตรกับ `Marshal` เหมือนที่เคยเห็นใน `encoding/json`:

```go
package main

import (
	"encoding/xml"
	"fmt"
)

type Book struct {
	XMLName xml.Name `xml:"book"`
	ID      int      `xml:"id,attr"`
	Title   string   `xml:"title"`
	Author  string   `xml:"author"`
	Price   float64  `xml:"price"`
}

func main() {
	xmlStr := `<book id="2"><title>Go in Action</title><author>วิชัย มั่นคง</author><price>420.75</price></book>`

	var b Book
	if err := xml.Unmarshal([]byte(xmlStr), &b); err != nil {
		fmt.Println("unmarshal error:", err)
		return
	}
	fmt.Printf("%+v\n", b)
}
```

ผลลัพธ์:

```
{XMLName:{Space: Local:book} ID:2 Title:Go in Action Author:วิชัย มั่นคง Price:420.75}
```

จุดสำคัญที่ต้องจำเหมือนกับ `json.Unmarshal`:

1. `xml.Unmarshal` รับ **pointer** เสมอ (`&b`) เพราะต้องแก้ไขค่าของตัวแปรโดยตรง (ทบทวนจาก **Part 010**)
2. `xml.Name` มี field ย่อยคือ `Space` (XML namespace ถ้ามี) และ `Local` (ชื่อ element ไม่รวม namespace) — สำหรับ XML ทั่วไปที่ไม่มี namespace `Space` จะว่างเปล่า

ถ้าต้องการอ่าน XML จาก `io.Reader` โดยตรง (เช่นจากไฟล์หรือ HTTP response body) ก็มี `xml.NewDecoder(r).Decode(&v)` ให้ใช้ ทำงานแบบเดียวกับ `json.NewDecoder` ที่เรียนใน **Part 025** ข้อ 8 ทุกประการ

---

## 6. เทียบ XML Tags กับ JSON Tags ที่เรียนใน Part 025

เพราะทั้งสอง package ออกแบบตามหลักการเดียวกัน (struct tag + reflect) จึงมีความคล้ายคลึงกันมาก แต่ก็มีความต่างที่สำคัญ:

| หัวข้อ | `encoding/json` | `encoding/xml` |
|---|---|---|
| Tag key | `json:"name"` | `xml:"name"` |
| ข้ามการ marshal | `json:"-"` | `xml:"-"` |
| ข้าม zero value | `json:"name,omitempty"` | `xml:"name,omitempty"` |
| แนวคิด attribute | ไม่มี (JSON ไม่มี attribute) | `xml:"name,attr"` |
| ชื่อ root/element แม่ | ไม่ต้องกำหนด (JSON เป็น object แบน) | ต้องกำหนดผ่าน field `XMLName xml.Name` หรือใช้ชื่อ type |
| Nested field แบบย่อ | ต้องประกาศ struct แยกเสมอ | ใช้ `xml:"parent>child"` สร้าง nested element ได้โดยไม่ต้องแยก struct |
| Exported field only | ใช่ | ใช่ (กฎเดียวกันจาก **Part 001**) |
| การ marshal เนื้อหาข้อความล้วน | ไม่มีแนวคิดนี้ | `xml:",chardata"` |

กฎทองที่ยังใช้ได้เหมือนกันทั้งคู่: **ต้องเป็น exported field (ขึ้นต้นตัวพิมพ์ใหญ่) เท่านั้นถึงจะถูกประมวลผล** เพราะทั้งสอง package ใช้กลไก `reflect` แบบเดียวกันในการอ่าน/เขียนค่า field ของ struct

---

## 7. YAML คืออะไร และทำไมไม่อยู่ใน Standard Library

**YAML (YAML Ain't Markup Language)** คือรูปแบบข้อมูลที่เน้นความอ่านง่ายสำหรับมนุษย์ ใช้การเยื้องบรรทัด (indentation) แทนวงเล็บปีกกาหรือแท็ก ตัวอย่างข้อมูลเดียวกันในสามรูปแบบ:

```yaml
# YAML
name: my-service
debug: true
server:
  host: 0.0.0.0
  port: 8080
```

```json
// JSON
{"name": "my-service", "debug": true, "server": {"host": "0.0.0.0", "port": 8080}}
```

YAML อ่านง่ายกว่า JSON มากสำหรับไฟล์ config ที่มนุษย์ต้องแก้ไขด้วยมือบ่อยๆ เพราะไม่มี comma, ไม่มีวงเล็บปีกกาซ้อนกันหลายชั้นให้นับ และรองรับ **comment** (`#`) ซึ่ง JSON ไม่มีเลย

อย่างไรก็ตาม Go **ไม่ได้ใส่ YAML ไว้ใน standard library** เหตุผลหลักคือ YAML spec มีความซับซ้อนสูงกว่า JSON/XML มาก (มี anchor, alias, multi-document, type coercion หลายแบบ) ทีมงาน Go เลือกที่จะรักษาปรัชญา "less is more" จาก **Part 001** โดยให้ standard library มีเฉพาะ format ที่ใช้กันแพร่หลายที่สุดในการสื่อสารระหว่างระบบ (JSON) และรูปแบบมาตรฐานเก่าแก่ (XML) ส่วน YAML ปล่อยให้ชุมชนพัฒนา package เอง

---

## 8. ติดตั้ง Third-Party Package: `gopkg.in/yaml.v3`

Package มาตรฐานของชุมชน Go ที่ใช้กันแพร่หลายที่สุดสำหรับ YAML คือ **`gopkg.in/yaml.v3`** (พัฒนาโดย Canonical ผู้สร้าง Ubuntu) ซึ่งใช้กันเป็นค่าเริ่มต้นในโปรเจกต์ Go เกือบทุกโปรเจกต์ที่ต้องอ่าน YAML

การติดตั้ง third-party package ใน Go ทำผ่านคำสั่ง `go get` ที่เกริ่นไว้ตั้งแต่ **Part 001**:

```bash
mkdir yaml-demo
cd yaml-demo
go mod init yaml-demo
go get gopkg.in/yaml.v3
```

หลังรันคำสั่งนี้ Go จะ:

1. ดาวน์โหลด package จาก proxy (`GOPROXY` ที่เรียนใน **Part 001**) มาเก็บไว้ใน cache local
2. เพิ่มบรรทัด `require gopkg.in/yaml.v3 v3.0.1` ลงใน `go.mod`
3. สร้าง/อัปเดต `go.sum` เพื่อเก็บ checksum ยืนยันความถูกต้องของโค้ดที่ดาวน์โหลดมา (จะเรียนเจาะลึกเรื่อง `go.sum` ใน **Part 018**)

หลังจากนี้ก็ `import` มาใช้งานได้ตามปกติเหมือน package มาตรฐานทุกประการ — Go ไม่แยกความแตกต่างระหว่าง standard library กับ third-party package ในระดับภาษาเลย ต่างกันแค่ที่มาของโค้ดเท่านั้น

---

## 9. Struct Tags และ Marshal/Unmarshal ด้วย `yaml.v3`

หน้าตา API ของ `yaml.v3` ถูกออกแบบให้คุ้นเคยสำหรับใครก็ตามที่ใช้ `encoding/json` มาก่อน — ใช้ `yaml:"name"` เป็น struct tag และมีฟังก์ชัน `Marshal`/`Unmarshal` คู่กันเช่นเดียวกัน:

```go
package main

import (
	"fmt"

	"gopkg.in/yaml.v3"
)

type Server struct {
	Host string   `yaml:"host"`
	Port int      `yaml:"port"`
	Tags []string `yaml:"tags,omitempty"`
}

type Config struct {
	Name    string `yaml:"name"`
	Debug   bool   `yaml:"debug"`
	Server  Server `yaml:"server"`
	Workers int    `yaml:"workers,omitempty"`
}

func main() {
	cfg := Config{
		Name:  "my-service",
		Debug: true,
		Server: Server{
			Host: "0.0.0.0",
			Port: 8080,
			Tags: []string{"prod", "api"},
		},
	}

	out, err := yaml.Marshal(cfg)
	if err != nil {
		fmt.Println("marshal error:", err)
		return
	}
	fmt.Println(string(out))

	yamlStr := `
name: another-service
debug: false
server:
  host: 127.0.0.1
  port: 9090
  tags:
    - staging
workers: 4
`
	var cfg2 Config
	if err := yaml.Unmarshal([]byte(yamlStr), &cfg2); err != nil {
		fmt.Println("unmarshal error:", err)
		return
	}
	fmt.Printf("%+v\n", cfg2)
}
```

ผลลัพธ์:

```
name: my-service
debug: true
server:
    host: 0.0.0.0
    port: 8080
    tags:
        - prod
        - api

{Name:another-service Debug:false Server:{Host:127.0.0.1 Port:9090 Tags:[staging]} Workers:4}
```

จุดที่ควรสังเกตเทียบกับ JSON:

- `yaml.v3` ใช้ **4 ช่องว่าง** เป็นค่า default สำหรับการเยื้องบรรทัดตอน marshal (ต่างจาก `MarshalIndent` ของ JSON ที่เราเลือกเองได้ทุกครั้ง)
- `omitempty` ทำงานแบบเดียวกับใน `encoding/json` ที่เรียนใน **Part 025** — ข้าม field ถ้าเป็น zero value และมีข้อควรระวังเรื่องแยก "ไม่ได้ตั้งค่า" กับ "ตั้งค่าเป็นศูนย์" ไม่ได้เหมือนกัน แนะนำใช้ pointer type ในกรณีที่ต้องแยกให้ชัดเจนเหมือนกัน
- `yaml.Unmarshal` รับ pointer เสมอ เหมือนกับ `json.Unmarshal` และ `xml.Unmarshal`
- YAML รองรับ list แบบ `- item` (dash) ซึ่งแปลงเข้า Go slice ได้อัตโนมัติ เหมือนที่ JSON array แปลงเข้า Go slice

---

## 10. เมื่อไรควรเลือก YAML แทน JSON ในทางปฏิบัติ

ในฐานะ Go Developer มืออาชีพ ควรเลือกรูปแบบข้อมูลให้เหมาะกับสถานการณ์ ไม่ใช่ใช้ตัวเดียวกันตลอด:

| สถานการณ์ | รูปแบบที่แนะนำ | เหตุผล |
|---|---|---|
| REST API request/response | **JSON** | เบา, parse เร็ว, เป็นมาตรฐานของเว็บ, ไม่มี ambiguity เรื่อง indentation |
| ไฟล์ config ที่มนุษย์แก้ไขด้วยมือบ่อย | **YAML** | อ่านง่าย มี comment ได้ ไม่ต้องนับวงเล็บปีกกา |
| Kubernetes manifests (Deployment, Service, ConfigMap) | **YAML** | เป็นมาตรฐานบังคับของ Kubernetes API — จะเรียนเจาะลึกใน **Part 097** |
| Docker Compose (`docker-compose.yml`) | **YAML** | เป็นมาตรฐานของ Docker Compose — จะเรียนเจาะลึกใน **Part 096** |
| CI/CD pipeline (GitHub Actions, GitLab CI) | **YAML** | อ่านง่ายสำหรับ workflow ที่มีขั้นตอนซับซ้อนหลายชั้น |
| ข้อมูลที่ machine-to-machine ล้วนๆ ไม่มีคนอ่าน | **JSON** | เร็วกว่า YAML ในการ parse และมี ambiguity น้อยกว่า (YAML มีกับดักเรื่อง type coercion เช่น `no`/`yes`/`on`/`off` ที่อาจถูกตีความเป็น boolean โดยไม่ตั้งใจ) |
| SOAP web service, ระบบ enterprise เก่า | **XML** | เป็นมาตรฐานบังคับของ protocol เหล่านั้น |

หลักการสรุปสั้นๆ: **JSON เหมาะกับการสื่อสารระหว่างโปรแกรม (machine-readable เป็นหลัก), YAML เหมาะกับไฟล์ที่มนุษย์ต้องอ่านและแก้ไขบ่อย (human-editable เป็นหลัก)** ส่วน XML ยังมีที่ยืนในระบบ legacy และมาตรฐานอุตสาหกรรมบางกลุ่มที่ยังไม่เปลี่ยนมาใช้ JSON

หลักสูตรนี้จะกลับมาเจอ YAML อีกครั้งอย่างจริงจังใน **ภาคที่ 9 (DevOps และ Deployment)** โดยเฉพาะ **Part 096 (Docker Compose)** และ **Part 097 (Kubernetes)** ที่ทักษะการอ่าน/เขียน YAML ด้วย `yaml.v3` ที่เรียนในบทนี้จะถูกนำไปใช้จริงในการเขียนเครื่องมือจัดการ config อัตโนมัติ

---

## สรุปสิ่งที่ได้เรียนในบทนี้

- `encoding/xml` อยู่ใน standard library ใช้งานคล้าย `encoding/json` มาก: `xml.Marshal`/`xml.MarshalIndent` แปลง struct เป็น XML, `xml.Unmarshal` แปลงกลับ (รับ pointer เสมอ)
- Struct tag ของ XML: `xml:"name"` กำหนดชื่อ element, `xml:"name,attr"` ทำให้เป็น attribute แทน element, `xml:"parent>child"` สร้าง nested element แบบย่อ, field พิเศษ `XMLName xml.Name` กำหนดชื่อ root element
- กฎ exported field only ใช้เหมือนกับ JSON เพราะทั้งคู่ใช้ `reflect` แบบเดียวกัน
- YAML ไม่มีอยู่ใน standard library เพราะ spec ซับซ้อนกว่า JSON มาก — ต้องใช้ third-party package `gopkg.in/yaml.v3` ที่เป็นมาตรฐานโดยพฤตินัยของชุมชน
- ติดตั้ง third-party package ด้วย `go get <module-path>` ซึ่งอัปเดต `go.mod`/`go.sum` ให้อัตโนมัติ
- `yaml.v3` ใช้ struct tag `yaml:"name"` และมี `yaml.Marshal`/`yaml.Unmarshal` คู่กันเหมือน `encoding/json`
- เลือก JSON สำหรับ machine-to-machine communication (REST API), เลือก YAML สำหรับไฟล์ config ที่มนุษย์แก้ไขบ่อย (Kubernetes, Docker Compose, CI/CD) — จะกลับมาใช้จริงใน **ภาคที่ 9**

## แบบฝึกหัดท้ายบท

1. ประกาศ struct `Article` ที่มี `ID int` (เป็น XML attribute), `Title string`, `Content string` แล้วลอง marshal เป็น XML พร้อม `xml.Header` นำหน้า
2. เขียนโปรแกรมที่มี struct `Playlist` ที่มี field `Songs []string` โดยใช้ tag `xml:"songs>song"` แล้ว marshal ดูผลลัพธ์ว่า element ซ้อนกันถูกต้องหรือไม่
3. เขียนโปรแกรมที่ unmarshal ไฟล์ XML string ที่กำหนดให้กลับเป็น struct แล้วพิมพ์ค่าที่ได้ทั้งหมดออกมา
4. สร้างโปรเจกต์ใหม่ ติดตั้ง `gopkg.in/yaml.v3` ด้วยตัวเอง แล้วเขียน struct ที่แทนไฟล์ config ของแอปสมมติ (มี `port`, `debug`, `allowed_hosts []string`) จากนั้น marshal เป็น YAML และ unmarshal กลับ
5. เขียนโปรแกรมที่อ่านไฟล์ YAML จริงจากดิสก์ (ใช้ `os.ReadFile` จาก **Part 024**) แล้ว unmarshal เข้า struct ที่ออกแบบเอง
6. ลองเปรียบเทียบ: เขียนข้อมูลชุดเดียวกัน (เช่น config ของเว็บเซิร์ฟเวอร์) เป็นทั้ง JSON, XML, และ YAML แล้วเทียบว่ารูปแบบไหนอ่านง่ายที่สุดสำหรับไฟล์ config ที่ซับซ้อน 3-4 ชั้น

---

**ต่อไป**: [Part 027 — การเรียงลำดับด้วยแพ็กเกจ `sort`](./027-sort-package.md)
