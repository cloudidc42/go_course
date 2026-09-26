# Part 018: การจัดการ Package และ Module ขั้นสูง

> ภาคที่ 2: ระดับกลาง (Intermediate) — ตอนที่ 3 จาก 20 (Part 16–35)

## สารบัญของบทนี้

1. ทบทวน `go.mod` และเจาะลึก `go.sum`
2. `go.sum` กับความปลอดภัยของ supply chain (hash verification)
3. Directive ใน `go.mod`: `require`, `replace`, `exclude`
4. ใช้ `replace` สำหรับพัฒนากับ module ที่ fork หรือแก้ไขในเครื่อง
5. Semantic Import Versioning: module `v2` ขึ้นไป
6. Private Module: `GOPRIVATE`, `GONOPROXY`, `GONOSUMDB` และเรื่องของ `GONOSUMCHECK`
7. Vendoring ด้วย `go mod vendor`
8. Debug dependency ด้วย `go mod graph` และ `go mod why`
9. Go Workspaces (`go.work`) สำหรับพัฒนาหลาย module พร้อมกัน
10. สรุปสิ่งที่ได้เรียนในบทนี้
11. แบบฝึกหัดท้ายบท

---

## 1. ทบทวน `go.mod` และเจาะลึก `go.sum`

จาก Part 002 เรารู้จัก `go.mod` มาแล้วในฐานะไฟล์ที่ประกาศตัวตนของ module (`module` directive), เวอร์ชัน Go ที่ใช้ (`go` directive), และรายการ dependency (`require` directive) รูปแบบพื้นฐานหน้าตาแบบนี้:

```
module example.com/myapp

go 1.22

require (
	rsc.io/quote v1.5.2
)
```

แต่เมื่อโปรเจกต์เริ่มมี dependency มากขึ้น เราจะเห็นไฟล์อีกไฟล์หนึ่งที่ Go สร้างขึ้นมาคู่กันเสมอคือ **`go.sum`** ซึ่งเป็นหัวข้อที่ Part 002 ยังไม่ได้ลงรายละเอียด — บทนี้จะอธิบายว่าไฟล์นี้ทำหน้าที่อะไร และสำคัญขนาดไหนต่อความปลอดภัยของโปรเจกต์

---

## 2. `go.sum` กับความปลอดภัยของ supply chain (hash verification)

ลองสร้างโปรเจกต์ที่ดึง dependency จริงมาดูว่า `go.sum` หน้าตาเป็นอย่างไร:

```bash
mkdir gosumdemo && cd gosumdemo
go mod init example.com/gosumdemo
go get rsc.io/quote@v1.5.2
```

หลังจากรันคำสั่งนี้ นอกจาก `go.mod` จะถูกอัปเดต จะมีไฟล์ `go.sum` ถูกสร้างขึ้นมาด้วย มีเนื้อหาประมาณนี้:

```
golang.org/x/text v0.0.0-20170915032832-14c0d48ead0c h1:qgOY6WgZOaTkIIMiVjBQcw93ERBE4m30iBm00nkL0i8=
golang.org/x/text v0.0.0-20170915032832-14c0d48ead0c/go.mod h1:NqM8EUOU14njkJ3fqMW+pc6Ldnwhi/IjpwHt7yyuwOQ=
rsc.io/quote v1.5.2 h1:w5fcysjrx7yqtD/aO+QwRjYZOKnaM9Uh2b40tElTs3Y=
rsc.io/quote v1.5.2/go.mod h1:LzX7hefJvL54yjefDEDHNONDjII0t9xZLPXsUe+TKr0=
rsc.io/sampler v1.3.0 h1:7uVkIFmeBqHfdjD+gZwtXXI+RODJ2Wc4O7MPEh/QiW4=
rsc.io/sampler v1.3.0/go.mod h1:T1hPZKmBbMNahiBKFy5HrXp6adAjACjK9JXDnKaTXpA=
```

### อ่านโครงสร้างของแต่ละบรรทัด

แต่ละบรรทัดมีรูปแบบ `<module path> <version>[/go.mod] <hash>`:

- **`module path` และ `version`**: ระบุ module และเวอร์ชันที่แน่นอน
- **บรรทัดที่ไม่มี `/go.mod` ต่อท้าย** (`h1:...`): เป็น hash ของ**เนื้อหาไฟล์ทั้งหมดของ module version นั้น** (ทุกไฟล์ source code ที่ดาวน์โหลดมา)
- **บรรทัดที่มี `/go.mod` ต่อท้าย**: เป็น hash ของไฟล์ `go.mod` ของ module นั้นเพียงไฟล์เดียว (Go ต้องอ่าน `go.mod` ของทุก dependency แม้บาง version จะไม่ได้ถูกใช้จริงในที่สุด เพื่อคำนวณ dependency graph จึงต้องมี hash แยกไว้ตรวจสอบด้วย)
- **`h1:` คือชื่ออัลกอริทึม hash** (เวอร์ชัน 1 ของรูปแบบ hash ที่ Go module ใช้ อ้างอิงจาก SHA-256)

### ทำไม `go.sum` ถึงสำคัญต่อความปลอดภัย

`go.mod` บอกแค่ว่า "ต้องการ dependency ตัวไหน เวอร์ชันอะไร" แต่ **ไม่ได้รับประกันว่าตอน build ครั้งถัดไป โค้ดที่ดาวน์โหลดมาจะเหมือนเดิมทุกประการ** — สมมติมีคนแฮ็กเข้าไปแก้ไขโค้ดใน repository ต้นทางของ dependency นั้น (หรือเข้าไปแก้ที่ proxy ระหว่างทาง) โดยที่เลขเวอร์ชันยังเป็นตัวเดิม ถ้าไม่มีกลไกตรวจสอบใดๆ โปรแกรมของเราก็จะดึงโค้ดที่ถูกแก้ไข (ซึ่งอาจแฝง malicious code) มาใช้โดยไม่รู้ตัว

`go.sum` แก้ปัญหานี้ด้วยการ**บันทึก cryptographic hash ของเนื้อหาที่ดาวน์โหลดมาไว้ตอนแรก** ทุกครั้งที่ `go build`/`go test`/`go run` ถูกเรียกในภายหลัง Go toolchain จะ**คำนวณ hash ของไฟล์ที่ดาวน์โหลดมาใหม่แล้วเทียบกับค่าที่บันทึกไว้ใน `go.sum`** ถ้าไม่ตรงกัน จะ**ปฏิเสธการ build ทันที** พร้อม error แจ้งว่า checksum ไม่ตรงกัน นี่คือหัวใจของสิ่งที่เรียกว่า **supply chain integrity** — รับประกันว่าโค้ดที่ใช้ compile วันนี้ พรุ่งนี้ หรือบนเครื่องอื่น จะเป็นโค้ดชุดเดียวกันเป๊ะทุกครั้ง

### `go.sum` ทำงานร่วมกับ Checksum Database (`sum.golang.org`)

นอกจากเทียบกับค่าที่มีอยู่แล้วใน `go.sum` เมื่อดึง dependency **ตัวใหม่** ที่ยังไม่เคยมี hash บันทึกไว้ Go จะตรวจสอบ hash นั้นกับ **checksum database กลางที่เชื่อถือได้** (`sum.golang.org` ตาม default ผ่านตัวแปร `GOSUMDB`) ก่อนบันทึกลง `go.sum` เพื่อป้องกันไม่ให้ผู้ให้บริการ proxy ตัวใดตัวหนึ่งส่งโค้ดปลอมมาให้ตั้งแต่ครั้งแรกที่ดาวน์โหลด (attack แบบนี้เรียกว่า trust-on-first-use ที่ไม่ปลอดภัยพอ) กลไกนี้เรียกรวมกันว่า **module authentication**

### คำสั่งที่เกี่ยวข้องกับ `go.sum`

```bash
go mod verify    # ตรวจสอบว่า module ทั้งหมดใน cache local ตรงกับ go.sum หรือไม่
go mod download  # ดาวน์โหลด dependency ตาม go.mod/go.sum โดยไม่ build
```

ทดลองรัน `go mod verify` หลังจากดึง dependency ข้างต้นมาแล้ว จะได้ผลลัพธ์:

```
all modules verified
```

**กฎสำคัญ: `go.sum` ต้อง commit เข้า version control เสมอ** เหมือน `go.mod` ห้ามใส่ไว้ใน `.gitignore` เด็ดขาด เพราะเป็นส่วนสำคัญของการรับประกันความปลอดภัยของทั้งทีม — ทุกคนที่ clone โปรเจกต์ไปต้อง build ด้วยโค้ด dependency ชุดเดียวกันเป๊ะ

---

## 3. Directive ใน `go.mod`: `require`, `replace`, `exclude`

ไฟล์ `go.mod` รองรับ directive มากกว่าที่ Part 002 กล่าวถึง มาดูทีละตัว

### `require`

Directive ที่คุ้นเคยที่สุด ใช้ประกาศว่าโปรเจกต์ต้องการ module ใดที่เวอร์ชันเท่าไหร่:

```
require (
	rsc.io/quote v1.5.2
	golang.org/x/text v0.3.0 // indirect
)
```

คอมเมนต์ `// indirect` หมายถึง module นี้ไม่ได้ถูก import ตรงๆ ในโค้ดของเรา แต่เป็น dependency ของ dependency อีกที (transitive dependency) — Go เติมคอมเมนต์นี้ให้อัตโนมัติเพื่อความชัดเจน ไม่ต้องแก้ไขเอง

### `exclude`

ใช้บอกว่า**ห้าม**ใช้ module version ใดเวอร์ชันหนึ่งเด็ดขาด แม้ dependency graph จะเรียกร้องเวอร์ชันนั้นก็ตาม (Go จะเลือกเวอร์ชันถัดไปที่ยังตรงเงื่อนไข semantic versioning แทน) มักใช้เมื่อรู้ว่า version นั้นมีบั๊กร้ายแรงหรือช่องโหว่ความปลอดภัย:

```
exclude rsc.io/sampler v1.99.99
```

`exclude` ใช้ได้เฉพาะใน **module หลักที่กำลัง build** เท่านั้น — ถ้า dependency ของเราเอง (ไม่ใช่ module ของเรา) มี `exclude` ไว้ใน `go.mod` ของมัน กฎนั้นจะไม่ถูกนำมาใช้กับเรา (`exclude` ไม่ใช่ transitive)

### `replace`

Directive ที่ทรงพลังและใช้บ่อยที่สุดในบรรดาสามตัวนี้ ใช้**เปลี่ยนแหล่งที่มา**ของ module หนึ่ง ไปเป็นแหล่งอื่น (path ในเครื่อง, repository อื่น, หรือ version อื่น) โดยไม่ต้องแก้ import path ในโค้ดเลยแม้แต่บรรทัดเดียว เราจะอธิบายละเอียดพร้อมตัวอย่างที่รันได้จริงในหัวข้อถัดไป

---

## 4. ใช้ `replace` สำหรับพัฒนากับ module ที่ fork หรือแก้ไขในเครื่อง

สถานการณ์ที่พบบ่อยมากในการพัฒนาจริง: เรากำลังพัฒนา module `A` ที่ต้องพึ่งพา module `B` แต่เจอบั๊กใน `B` หรือต้องการทดลองแก้ไข `B` เพื่อดูว่าแก้ปัญหาได้จริงไหม **ก่อนที่จะ publish เวอร์ชันใหม่ของ `B` ออกไปจริง** — `replace` directive แก้ปัญหานี้ได้โดยชี้ dependency ไปยัง path ในเครื่องแทน

### ตัวอย่างที่รันได้จริง: local module สองตัว

สร้างโครงสร้างสองโฟลเดอร์ — `greeting` (module ที่จะถูก "แก้ไขในเครื่อง") และ `app` (module หลักที่ใช้งาน `greeting`)

ไฟล์ `greeting/go.mod`:

```
module example.com/greeting

go 1.22
```

ไฟล์ `greeting/greeting.go`:

```go
package greeting

func Hello(name string) string {
	return "Hello, " + name + "! (จาก local module)"
}
```

ไฟล์ `app/go.mod`:

```
module example.com/app

go 1.22

require example.com/greeting v0.0.0

replace example.com/greeting => ../greeting
```

ไฟล์ `app/main.go`:

```go
package main

import (
	"fmt"

	"example.com/greeting"
)

func main() {
	fmt.Println(greeting.Hello("โลก"))
}
```

รันจากโฟลเดอร์ `app`:

```bash
go run main.go
```

ผลลัพธ์:

```
Hello, โลก! (จาก local module)
```

### อ่านบรรทัด `replace` ให้เข้าใจ

```
replace example.com/greeting => ../greeting
```

หมายความว่า **"เมื่อไหร่ก็ตามที่ dependency graph ต้องการ `example.com/greeting` ไม่ว่า version ไหนที่ระบุใน `require` ให้ไปอ่านโค้ดจาก path `../greeting` แทน"** สังเกตว่า `require example.com/greeting v0.0.0` ยังต้องมีอยู่ (บอกว่ามี dependency ตัวนี้) แต่ตัวเลขเวอร์ชันไม่มีผลจริงเมื่อมี `replace` มาทับ และ**ไม่ต้องมี `go.sum` entry สำหรับ module ที่ถูก replace ด้วย local path** เพราะไม่มีการดาวน์โหลดจากที่ไหนเลย เป็นแค่การอ่านไฟล์จาก disk ตรงๆ

### `replace` ใช้กับ fork บน Git ก็ได้

นอกจาก local path แล้ว `replace` ยังชี้ไปยัง repository อื่นหรือ version อื่นได้ เช่น กรณีที่เรา fork library แล้วแก้ไขบั๊กในบัญชี GitHub ของตัวเอง ระหว่างรอ pull request ต้นทางถูก merge:

```
replace github.com/original/library => github.com/myaccount/library v1.2.3-patched
```

### ข้อควรระวังเรื่อง `replace`

**`replace` มีผลเฉพาะใน module หลักที่กำลัง build เท่านั้น** — ถ้าเราสร้าง library ที่คนอื่นจะ import ไปใช้ `replace` ที่เราใส่ไว้ใน `go.mod` ของเรา**จะไม่ถูกนำไปใช้กับ module ของคนที่ import เราไปอีกที** (นี่คือความตั้งใจของผู้ออกแบบ Go Modules — ป้องกันไม่ให้ library หนึ่งบังคับ version ของ dependency ให้ผู้ใช้งานทั้งระบบ) เพราะฉะนั้น `replace` เหมาะสำหรับใช้ระหว่างพัฒนาเท่านั้น ก่อน publish เวอร์ชันจริงต้องลบ `replace` ออก แล้วรอให้ dependency ตัวจริงถูกแก้ไขและ publish เวอร์ชันใหม่แทน

---

## 5. Semantic Import Versioning: module `v2` ขึ้นไป

Go Modules ใช้ **Semantic Versioning** (semver: `MAJOR.MINOR.PATCH`) เป็นมาตรฐานในการกำหนดเวอร์ชัน และมีกฎพิเศษที่เข้มงวดมากสำหรับ **MAJOR version ตั้งแต่ 2 ขึ้นไป** ที่เรียกว่า **Semantic Import Versioning**

> **กฎ**: เมื่อ module ออกเวอร์ชัน major ใหม่ (v2, v3, ...) ที่มี breaking change **module path ต้องเปลี่ยนตามด้วย** โดยเติม `/v2`, `/v3`, ... ต่อท้าย path เดิม

เหตุผลของกฎนี้คือ Go Modules ออกแบบมาให้ **สอง major version ของ module เดียวกัน สามารถถูก import พร้อมกันในโปรแกรมเดียวได้โดยไม่ชนกัน** เพราะสำหรับ Go compiler แล้ว `example.com/mathlib` และ `example.com/mathlib/v2` คือคนละ package กันโดยสิ้นเชิง (import path ต่างกัน) ทำให้โปรเจกต์ที่ค่อยๆ ทยอย migrate จาก v1 ไป v2 สามารถใช้ทั้งสองเวอร์ชันคู่ขนานกันได้ระหว่างการเปลี่ยนผ่าน

### ตัวอย่างที่รันได้จริง: module v2

ไฟล์ `mathlib/go.mod` (สังเกต `module` directive ต้องมี `/v2` ต่อท้าย):

```
module example.com/mathlib/v2

go 1.22
```

ไฟล์ `mathlib/mathlib.go`:

```go
package mathlib

// Add ในเวอร์ชัน v2 เปลี่ยน behavior จาก v1 แบบ breaking change
// (v1 อาจจะ return int เฉยๆ แต่ v2 เปลี่ยน signature หรือพฤติกรรม)
func Add(a, b int) int {
	return a + b
}
```

ไฟล์ `consumer/go.mod`:

```
module example.com/consumer

go 1.22

require example.com/mathlib/v2 v2.0.0

replace example.com/mathlib/v2 => ../mathlib
```

ไฟล์ `consumer/main.go`:

```go
package main

import (
	"fmt"

	mathlib "example.com/mathlib/v2"
)

func main() {
	fmt.Println(mathlib.Add(2, 3))
}
```

รันจากโฟลเดอร์ `consumer`:

```bash
go run main.go
```

ผลลัพธ์:

```
5
```

สังเกตสามจุดสำคัญ:

1. **`package mathlib` (ชื่อ package ในตัวไฟล์ `.go`) ไม่ต้องมี `v2` ต่อท้าย** — `/v2` เป็นส่วนหนึ่งของ **module path / import path** เท่านั้น ไม่ใช่ชื่อ package
2. **`import "example.com/mathlib/v2"`** ใน `consumer/main.go` ต้องมี `/v2` ต่อท้ายเสมอ ตรงกับที่ `module` directive ใน `mathlib/go.mod` ประกาศไว้
3. เราตั้งชื่อเรียกตอน import ว่า `mathlib` (ผ่าน `mathlib "example.com/mathlib/v2"`) เพื่อความสะดวก — โดย default ถ้าไม่ตั้งชื่อ Go จะใช้ชื่อจาก `package` declaration ในไฟล์ต้นทาง (คือ `mathlib`) ไม่ใช่ค่าตัวสุดท้ายของ path ซึ่งจะเป็น `v2` เฉยๆ ถ้าไม่มีการตั้งชื่อกำกับ ผลลัพธ์คือชื่อที่เรียกใช้ในโค้ดจะเป็น `mathlib.Add(...)` ตามชื่อ package จริงอยู่ดี แต่การระบุชื่อกำกับไว้ชัดๆ (โดยเฉพาะเมื่อ path ลงท้ายด้วยตัวเลขเวอร์ชัน) ช่วยให้ผู้อ่านโค้ดเข้าใจง่ายขึ้นว่ากำลัง import อะไรจากที่ไหน

### ทำไม v0 และ v1 ไม่ต้องมี suffix

Semantic Import Versioning มีผลเฉพาะ **major version ตั้งแต่ 2 ขึ้นไป** เท่านั้น `v0.x.x` และ `v1.x.x` ใช้ module path เดิมโดยไม่ต้องเติมอะไร (`v0` ถือเป็นช่วง "ยังไม่เสถียร ไม่มีสัญญา backward compatibility ใดๆ" ตามหลัก semver) เราจะกลับมาพูดถึงการออกแบบ versioning เชิงลึกอีกครั้งใน **Part 108 (Go Modules Best Practices และ Semantic Versioning)**

---

## 6. Private Module: `GOPRIVATE`, `GONOPROXY`, `GONOSUMDB` และเรื่องของ `GONOSUMCHECK`

ในองค์กรที่มี private repository (เช่น GitHub Enterprise หรือ GitLab ภายในบริษัท) เราไม่ต้องการให้ Go toolchain พยายามดาวน์โหลด module เหล่านั้นผ่าน public proxy (`proxy.golang.org`) หรือตรวจสอบผ่าน public checksum database (`sum.golang.org`) เพราะ (ก) repository เหล่านั้นเข้าถึงไม่ได้จากอินเทอร์เน็ตสาธารณะอยู่แล้ว และ (ข) ไม่ต้องการให้ path ของ private module รั่วไหลไปยัง service ภายนอกโดยไม่จำเป็น

### `GOPRIVATE`: ตัวแปรหลักที่ควรตั้งค่า

```bash
go env -w GOPRIVATE=github.com/mycompany/*
```

`GOPRIVATE` รับ pattern แบบ comma-separated ของ module path prefix (ใช้ syntax เดียวกับ `path.Match`) — module ใดก็ตามที่ path ตรงกับ pattern นี้ Go จะ:

1. **ไม่ดึงผ่าน `GOPROXY` สาธารณะ** ดึงตรงจาก repository เลย (เทียบเท่ากับตั้งค่า `GONOPROXY` ให้ครอบคลุม pattern เดียวกัน)
2. **ไม่ตรวจสอบผ่าน checksum database สาธารณะ** (เทียบเท่ากับตั้งค่า `GONOSUMDB` ให้ครอบคลุม pattern เดียวกัน)

พูดง่ายๆ **`GOPRIVATE` คือทางลัดที่ตั้งค่าทั้ง `GONOPROXY` และ `GONOSUMDB` พร้อมกันในคำสั่งเดียว** ถ้าต้องการควบคุมแยกกันอย่างละเอียด (เช่น ให้ผ่าน proxy ภายในบริษัทได้ แต่ไม่ต้องเช็ค checksum) ก็ตั้งค่าตัวแปรย่อยแยกกันได้:

```bash
go env -w GONOPROXY=github.com/mycompany/*
go env -w GONOSUMDB=github.com/mycompany/*
```

### เรื่องของ `GONOSUMCHECK` (ตัวแปรเก่าที่เลิกใช้แล้ว)

หลายคนที่อ่านเอกสารเก่าหรือ blog เก่าอาจเจอตัวแปรชื่อ `GONOSUMCHECK` — **ตัวแปรนี้เป็นของยุคก่อน Go Modules (สมัย GOPATH)** ใช้สำหรับปิดการตรวจสอบ checksum ในเครื่องมือรุ่นเก่าที่ทำงานคู่กับ GOPATH เท่านั้น **ในยุค Go Modules ปัจจุบัน `GONOSUMCHECK` ไม่มีผลใดๆ อีกต่อไป** ตัวแปรที่มาแทนที่ในบทบาทเดียวกันคือ:

- **`GONOSUMDB`** (หรือใช้ผ่าน `GOPRIVATE`) — ยกเว้น pattern ที่กำหนดจากการตรวจสอบ checksum database
- **`GOSUMDB=off`** — ปิดการตรวจสอบ checksum database **ทั้งหมด** ทุก module (ใช้อย่างระมัดระวังมาก เพราะเสียการป้องกัน supply chain attack ไปทั้งระบบ ไม่ใช่แค่ private module)
- **`GOFLAGS=-insecure`** ร่วมกับ `GOINSECURE` — สำหรับกรณีพิเศษที่ต้องดึง module ผ่าน connection ที่ไม่มี TLS หรือ certificate ไม่ถูกต้อง (แยกเรื่องจาก checksum โดยสิ้นเชิง)

สรุปคือถ้าเจอคำแนะนำให้ตั้ง `GONOSUMCHECK` จากแหล่งข้อมูลใดก็ตาม ให้เข้าใจว่านั่นคือคำแนะนำที่ล้าสมัยแล้ว ควรใช้ `GOPRIVATE` แทนในโปรเจกต์ที่ใช้ Go Modules (ซึ่งคือมาตรฐานเดียวที่ใช้กันตั้งแต่ Go 1.16 เป็นต้นมา)

### การตั้งค่า authentication สำหรับ private repository

`GOPRIVATE` บอก Go ว่า**อย่าไปพึ่ง proxy/checksum สาธารณะ** แต่ไม่ได้จัดการเรื่อง**สิทธิ์การเข้าถึง** (authentication) เอง — สำหรับ private Git repository เราต้องตั้งค่าเพิ่มเติมผ่าน git config ให้ใช้ token แทน HTTPS ธรรมดา:

```bash
git config --global url."https://<token>@github.com/".insteadOf "https://github.com/"
```

หรือใช้ SSH แทน HTTPS โดยตั้งค่าคู่กับ `GOPRIVATE` ก็ได้เช่นกัน รายละเอียดเรื่อง environment ของ container/CI ที่ต้องเข้าถึง private repository จะกล่าวถึงลึกกว่านี้ใน **Part 098 (CI/CD Pipeline)**

---

## 7. Vendoring ด้วย `go mod vendor`

**Vendoring** คือการ**คัดลอกโค้ดของ dependency ทั้งหมดมาเก็บไว้ในโฟลเดอร์ `vendor/` ภายในโปรเจกต์เอง** แทนที่จะพึ่งพา module cache (`$GOPATH/pkg/mod`) หรือดาวน์โหลดใหม่ทุกครั้งที่ build

```bash
go mod vendor
```

คำสั่งนี้จะสร้างโฟลเดอร์ `vendor/` ที่มีโครงสร้างประมาณนี้:

```
vendor/
├── golang.org/
│   └── x/text/...
├── rsc.io/
│   ├── quote/...
│   └── sampler/...
└── modules.txt
```

ไฟล์ `vendor/modules.txt` คือ manifest ที่บอกว่า module ไหน version อะไรถูก vendor ไว้บ้าง:

```
# golang.org/x/text v0.0.0-20170915032832-14c0d48ead0c
## explicit
golang.org/x/text/internal/tag
golang.org/x/text/language
# rsc.io/quote v1.5.2
## explicit
rsc.io/quote
# rsc.io/sampler v1.3.0
## explicit
rsc.io/sampler
```

เมื่อมีโฟลเดอร์ `vendor/` อยู่ในโปรเจกต์ Go จะ**ใช้โค้ดจาก `vendor/` โดยอัตโนมัติ** (ตั้งแต่ Go 1.14 เป็นต้นไป ถ้ามี `vendor/modules.txt` ที่สอดคล้องกับ `go.mod`) หรือระบุให้ชัดเจนด้วย flag:

```bash
go build -mod=vendor ./...
```

### ทำไมยังต้องใช้ vendoring ในยุคที่มี module proxy แล้ว

แม้ Go Modules + `GOPROXY` จะแก้ปัญหาการดาวน์โหลด dependency ได้ดีอยู่แล้ว แต่ vendoring ยังมีประโยชน์ในบางสถานการณ์:

1. **Build ในสภาพแวดล้อมที่ไม่มี internet access เลย** (air-gapped environment) เช่นในองค์กรที่มีนโยบายความปลอดภัยเข้มงวด
2. **ต้องการให้ทุกไฟล์ที่ใช้ compile อยู่ใน version control repository เดียวกันแบบสมบูรณ์ 100%** ไม่ต้องพึ่ง network หรือ proxy ใดๆ เลยแม้แต่ตอน build ครั้งแรกบนเครื่องใหม่
3. **บาง CI/CD pipeline** ต้องการความเร็วในการ build ที่คงที่ ไม่ผันผวนตาม network latency ของการดาวน์โหลด module

ข้อเสียคือขนาด repository จะใหญ่ขึ้นมาก และต้องรัน `go mod vendor` ใหม่ทุกครั้งที่ dependency เปลี่ยน (ลืมรันแล้ว build ด้วย `-mod=vendor` จะได้โค้ดเก่าโดยไม่รู้ตัว) ในทางปฏิบัติปัจจุบัน โปรเจกต์ส่วนใหญ่ที่ไม่มีข้อจำกัดพิเศษ **มักไม่ vendor** และพึ่งพา module proxy + `go.sum` แทน

---

## 8. Debug dependency ด้วย `go mod graph` และ `go mod why`

เมื่อโปรเจกต์มี dependency ซับซ้อนขึ้น บางครั้งเราต้องการรู้ว่า "module ตัวนี้ที่ปรากฏใน `go.mod` มาจากไหน ใครดึงมันเข้ามา" หรือ "dependency graph ทั้งหมดหน้าตาเป็นอย่างไร" — Go มีเครื่องมือสำหรับ debug สองตัวนี้ในตัว

### `go mod graph`: แสดง dependency graph ทั้งหมด

```bash
go mod graph
```

ผลลัพธ์เป็นรายการคู่ `<module ที่ require> <module ที่ถูก require>` แบบเรียงต่อกันเป็น edge ของกราฟ:

```
example.com/gosumdemo go@1.24.7
example.com/gosumdemo golang.org/x/text@v0.0.0-20170915032832-14c0d48ead0c
example.com/gosumdemo rsc.io/quote@v1.5.2
example.com/gosumdemo rsc.io/sampler@v1.3.0
go@1.24.7 toolchain@go1.24.7
rsc.io/quote@v1.5.2 rsc.io/sampler@v1.3.0
rsc.io/sampler@v1.3.0 golang.org/x/text@v0.0.0-20170915032832-14c0d48ead0c
```

จากผลลัพธ์นี้อ่านได้ว่า `example.com/gosumdemo` (module หลักของเรา) require `rsc.io/quote` ตรงๆ ส่วน `rsc.io/quote` เองก็ require `rsc.io/sampler` ต่อไปอีกที และ `rsc.io/sampler` ก็ require `golang.org/x/text` ต่อไปอีกชั้น — เห็นภาพรวมทั้ง chain ได้ในคำสั่งเดียว บ่อยครั้งใช้ต่อกับ `grep` เพื่อกรองดูเฉพาะ module ที่สนใจ

### `go mod why`: ตอบคำถามว่า "ทำไมโปรเจกต์ถึงต้องพึ่งพา module นี้"

```bash
go mod why rsc.io/sampler
```

ผลลัพธ์:

```
# rsc.io/sampler
example.com/gosumdemo
rsc.io/quote
rsc.io/sampler
```

คำตอบนี้อ่านจากบนลงล่างเป็น **เส้นทางการ import สั้นที่สุด**: `example.com/gosumdemo` (โปรเจกต์เรา) import `rsc.io/quote` ซึ่ง import `rsc.io/sampler` — ทำให้เข้าใจทันทีว่า `rsc.io/sampler` ไม่ได้ถูกเราเรียกใช้ตรงๆ แต่ติดมาจาก `rsc.io/quote` อีกที คำสั่งนี้มีประโยชน์มากเวลาเจอ module แปลกๆ โผล่มาใน `go.sum` ที่เราไม่เคยเห็นชื่อมาก่อน แล้วอยากรู้ว่ามันมาจากไหน

เปรียบเทียบการใช้งานสองคำสั่งนี้:

| คำสั่ง | ใช้เมื่อ |
|---|---|
| `go mod graph` | ต้องการเห็นภาพรวมทั้งกราฟ dependency ทั้งหมด เพื่อวิเคราะห์เชิงลึกหรือส่งต่อให้เครื่องมืออื่นประมวลผล |
| `go mod why <module>` | ต้องการรู้เจาะจงว่า module ตัวหนึ่งถูกดึงเข้ามาเพราะอะไร เส้นทางสั้นที่สุดคืออะไร |

---

## 9. Go Workspaces (`go.work`) สำหรับพัฒนาหลาย module พร้อมกัน

`replace` ที่เรียนในหัวข้อ 4 แก้ปัญหา "ใช้โค้ด local แทน dependency จริง" ได้ แต่มีข้อเสียคือ **ต้องแก้ไข `go.mod` ของ module หลัก** (เพิ่ม `replace` เข้าไป) ซึ่งถ้าลืมลบก่อน commit อาจหลุดเข้า production โดยไม่ตั้งใจ — Go 1.18 แก้ปัญหานี้ด้วย **Go Workspaces** ผ่านไฟล์ `go.work` ที่อยู่**นอก** `go.mod` ของแต่ละ module โดยสิ้นเชิง (และปกติไม่ควร commit เข้า version control ของโปรเจกต์ share กับทีม เพราะเป็น local development setup ของแต่ละคน)

### ตัวอย่างที่รันได้จริง: ใช้ `go.work` แทน `replace`

สมมติโครงสร้างเดิมสองโฟลเดอร์ `app` และ `greeting` (ไม่มี `replace` ใน `go.mod` ของ `app` เลย):

ไฟล์ `greeting/go.mod`:

```
module example.com/greeting

go 1.22
```

ไฟล์ `greeting/greeting.go`:

```go
package greeting

func Hello(name string) string {
	return "สวัสดี " + name + " จาก workspace module"
}
```

ไฟล์ `app/go.mod` (สังเกตว่า**ไม่มี** `replace` เลย):

```
module example.com/app

go 1.22

require example.com/greeting v0.0.0
```

ไฟล์ `app/main.go`:

```go
package main

import (
	"fmt"

	"example.com/greeting"
)

func main() {
	fmt.Println(greeting.Hello("คุณ"))
}
```

จากนั้นที่ระดับบนสุด (โฟลเดอร์ที่ครอบทั้ง `app` และ `greeting`) สร้าง workspace:

```bash
go work init ./app ./greeting
```

คำสั่งนี้สร้างไฟล์ `go.work`:

```
go 1.24.7

use (
	./app
	./greeting
)
```

จากนั้นรันโปรแกรมจากโฟลเดอร์ `app` ได้เลยโดยไม่ต้องแก้ `go.mod` แม้แต่บรรทัดเดียว:

```bash
cd app
go run main.go
```

ผลลัพธ์:

```
สวัสดี คุณ จาก workspace module
```

### `go.work` ทำงานอย่างไร

เมื่อมี `go.work` อยู่ในโฟลเดอร์ปัจจุบันหรือโฟลเดอร์แม่ (Go ไล่หาขึ้นไปเรื่อยๆ เหมือนหา `go.mod`) Go toolchain จะ**อ่าน module ทุกตัวที่ระบุใน `use` เป็น "local module" ก่อนเสมอ** แทนที่จะไปดึงจาก `require` ตามปกติ — พูดง่ายๆ **`go.work` มีผลเหมือนใส่ `replace` ให้ทุก module ที่อยู่ใน `use` โดยอัตโนมัติ แต่ไม่ต้องแก้ `go.mod` ของ module ไหนเลย**

### คำสั่งจัดการ `go.work` ที่ใช้บ่อย

```bash
go work init ./app ./greeting   # สร้าง go.work และเพิ่ม module เริ่มต้น
go work use ./another-module    # เพิ่ม module เข้า workspace ภายหลัง
go work edit -dropuse=./old     # เอา module ออกจาก workspace
go work sync                    # sync เวอร์ชันจาก go.work ไปยัง go.mod ของแต่ละ module ที่เกี่ยวข้อง
```

### เทียบ `replace` กับ `go.work`

| | `replace` ใน `go.mod` | `go.work` |
|---|---|---|
| ตำแหน่งไฟล์ | อยู่ใน `go.mod` ของ module หลัก | ไฟล์แยกต่างหาก ไม่ปนกับ `go.mod` |
| เสี่ยงหลุดไป production | สูง (ถ้าลืมลบก่อน commit/push) | ต่ำมาก (ปกติไม่ commit `go.work` เข้า repo หลัก) |
| เหมาะกับ | Fix ชั่วคราวสำหรับ CI หรือกรณีเฉพาะที่ต้องแน่ใจว่าถูกใช้เสมอ | งานพัฒนาประจำวันที่ทำงานกับหลาย module ในเครื่องพร้อมกัน (เช่น monorepo หลาย module หรือแยก library ออกจาก app) |
| จำนวน module ที่จัดการง่าย | ทีละคู่ (module ต่อ module) | จัดการหลาย module พร้อมกันได้สะดวกกว่ามาก |

ในทางปฏิบัติ ทีมที่พัฒนาหลาย module พร้อมกันบ่อยๆ (เช่นแยก microservice หลายตัวเป็นคนละ module ใน monorepo เดียวกัน) มักใช้ `go.work` เป็นค่าเริ่มต้นสำหรับพัฒนาในเครื่อง แล้วปล่อยให้แต่ละ module มี `go.mod` ที่สะอาด ไม่มี `replace` เมื่อถูก build แยกจริงใน production

---

## สรุปสิ่งที่ได้เรียนในบทนี้

- **`go.sum`** เก็บ cryptographic hash ของทุก dependency ที่ดาวน์โหลดมา เพื่อรับประกัน **supply chain integrity** — ตรวจสอบว่าโค้ดที่ build ไม่ถูกแก้ไขระหว่างทาง ต้อง commit เข้า version control เสมอเหมือน `go.mod`
- Directive ใน `go.mod`: `require` (ประกาศ dependency), `exclude` (ห้ามใช้ version ที่กำหนด), `replace` (เปลี่ยนแหล่งที่มาของ dependency)
- **`replace`** ใช้พัฒนากับ module ที่ fork หรือแก้ไขในเครื่องได้โดยไม่ต้องแก้ import path แต่มีผลเฉพาะ module หลักที่ build เท่านั้น ไม่ส่งต่อไปยัง module ที่ import เราอีกที
- **Semantic Import Versioning**: module ที่ major version ตั้งแต่ v2 ขึ้นไป ต้องเติม `/v2`, `/v3`, ... ต่อท้าย module path เพื่อให้หลาย major version อยู่ร่วมกันในโปรแกรมเดียวได้โดยไม่ชนกัน
- **`GOPRIVATE`** ตั้งค่า pattern ของ private module เพื่อข้าม public proxy และ checksum database พร้อมกัน (เทียบเท่า `GONOPROXY` + `GONOSUMDB`) — **`GONOSUMCHECK` เป็นตัวแปรเก่าจากยุค GOPATH ที่เลิกใช้แล้ว**
- **`go mod vendor`** คัดลอก dependency ทั้งหมดมาเก็บใน `vendor/` เหมาะกับ environment ที่ไม่มี internet access หรือต้องการความเสถียรของ build สูงสุด
- **`go mod graph`** แสดง dependency graph ทั้งหมด, **`go mod why <module>`** ตอบว่าโปรเจกต์พึ่งพา module นั้นเพราะเส้นทางไหน
- **`go.work`** (Go 1.18+) จัดการหลาย module ในเครื่องพร้อมกันโดยไม่ต้องแก้ `go.mod` ของแต่ละ module เลย เหมาะกับงานพัฒนาประจำวันมากกว่า `replace`

## แบบฝึกหัดท้ายบท

1. สร้างโปรเจกต์ใหม่ แล้ว `go get` package ภายนอกสักตัว (เช่น `rsc.io/quote`) สังเกตเนื้อหาไฟล์ `go.sum` ที่ถูกสร้างขึ้น อธิบายความหมายของแต่ละส่วนในหนึ่งบรรทัด แล้วลองรัน `go mod verify` ดูผลลัพธ์
2. สร้าง local module สองตัวตามตัวอย่างในหัวข้อ 4 (`greeting` และ `app`) แล้วใช้ `replace` เชื่อมสองโมดูลเข้าด้วยกัน ทดลองแก้ไขโค้ดใน `greeting` แล้วรัน `app` ใหม่ ดูว่าการเปลี่ยนแปลงมีผลทันทีโดยไม่ต้อง publish version ใหม่
3. สร้าง module ที่ตั้งชื่อ `module path` ลงท้ายด้วย `/v2` ตามตัวอย่างในหัวข้อ 5 แล้วเขียนโปรแกรมที่ import ทั้ง v1 (สมมติสร้างโฟลเดอร์แยกไว้) และ v2 พร้อมกันในโปรแกรมเดียว พิสูจน์ว่าทั้งสองใช้งานพร้อมกันได้จริงโดยไม่ชนกัน
4. ทดลองรัน `go mod vendor` ในโปรเจกต์ที่มี dependency แล้วตรวจสอบไฟล์ `vendor/modules.txt` จากนั้นลองสั่ง `go build -mod=vendor` เทียบกับ `go build` ปกติ อธิบายว่าเห็นความต่างอะไรบ้าง (หรือไม่เห็นความต่างเพราะอะไร)
5. ในโปรเจกต์ที่มี dependency มากกว่าหนึ่งชั้น (transitive dependency) ใช้ `go mod graph` และ `go mod why` เพื่อหาว่า dependency ที่ลึกที่สุดถูกดึงเข้ามาผ่านเส้นทางไหน
6. สร้าง `go.work` ที่รวมสอง module (ใช้โครงสร้างเดียวกับข้อ 2 แต่คราวนี้ห้ามใส่ `replace` ใน `go.mod` เลย) แล้วพิสูจน์ว่าโปรแกรมยังรันได้ปกติผ่าน workspace เปรียบเทียบข้อดีข้อเสียกับวิธี `replace` ในข้อ 2

---

**ต่อไป**: [Part 019 — String และแพ็กเกจ `strings`](./019-strings-package.md)
