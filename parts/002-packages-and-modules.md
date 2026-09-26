# Part 002: โครงสร้างโปรแกรม, package, Go Module, `go.mod`

> ภาคที่ 1: พื้นฐานภาษา Go (Fundamentals) — ตอนที่ 2 จาก 15

## สารบัญของบทนี้

1. ทบทวน: package คืออะไร
2. กฎการประกาศ package ในไฟล์ `.go`
3. หนึ่ง package หลายไฟล์ (multi-file package)
4. package name กับ import path ต่างกันอย่างไร
5. Go Module คืออะไร แก้ปัญหาอะไร
6. `go mod init` และการสร้างโมดูลแรก
7. ผ่าโครงสร้างไฟล์ `go.mod` ทีละบรรทัด
8. `go.sum` คืออะไร ทำไมต้องมี
9. การ resolve import path: standard library, module ของเรา, module ภายนอก
10. `go mod tidy` ใช้เมื่อไหร่ ทำอะไรบ้าง
11. คำสั่ง `go mod` อื่นๆ ที่ควรรู้
12. Semantic Import Versioning เบื้องต้น (v2+)
13. Visibility กับขอบเขตของ package (ทบทวนและขยายจาก Part 001)
14. โครงสร้างโปรเจกต์จริงที่ประกอบด้วยหลาย package
15. สรุปสิ่งที่ได้เรียนในบทนี้
16. แบบฝึกหัดท้ายบท

---

## 1. ทบทวน: package คืออะไร

ใน Part 001 เราเห็นแล้วว่าไฟล์ `main.go` เริ่มต้นด้วย `package main` เสมอ ในบทนี้เราจะเจาะลึกว่า **package** คืออะไรกันแน่ และมันสัมพันธ์กับ **Go Module** อย่างไร

พูดให้ชัดที่สุด:

> **package** คือหน่วยของการจัดระเบียบโค้ดใน Go — โค้ดทุกไฟล์ `.go` ต้องเป็นสมาชิกของ package ใด package หนึ่งเสมอ ไม่มีข้อยกเว้น

package ทำหน้าที่คล้าย "namespace" ใน Java/C# หรือ "module" ใน Python แต่ Go ผูก package เข้ากับ **โฟลเดอร์** อย่างเข้มงวด: ไฟล์ `.go` ทุกไฟล์ที่อยู่ใน**โฟลเดอร์เดียวกัน** ต้องประกาศ `package` เป็นชื่อเดียวกันทั้งหมด (ยกเว้นไฟล์ทดสอบที่ใช้ suffix `_test` ซึ่งจะเรียนใน Part 033)

package มี 2 ประเภทหลัก:

| ประเภท | ลักษณะ | ตัวอย่าง |
|---|---|---|
| **`package main`** | คอมไพล์เป็น **executable program** ต้องมีฟังก์ชัน `func main()` | โปรแกรม CLI, web server, batch job |
| **library package** (ชื่ออื่นที่ไม่ใช่ `main`) | คอมไพล์เป็น "ของที่ import ไปใช้" ไม่สามารถรันตรงๆ ได้ | `fmt`, `strings`, หรือ package ที่เราเขียนเอง เช่น `mathutil` |

---

## 2. กฎการประกาศ package ในไฟล์ `.go`

ทุกไฟล์ `.go` **ต้อง**มีบรรทัด `package <ชื่อ>` เป็นบรรทัดแรกที่ไม่ใช่ comment เสมอ:

```go
package mathutil
```

กฎสำคัญที่ต้องจำ:

1. **ชื่อ package ควรเป็นตัวพิมพ์เล็กล้วน** ไม่ใช้ underscore หรือ camelCase (เช่น `mathutil` ไม่ใช่ `MathUtil` หรือ `math_util`) — นี่คือ convention ที่ทั้งชุมชนยึดถือเหมือนกันหมด
2. **ชื่อ package ไม่จำเป็นต้องตรงกับชื่อโฟลเดอร์** แต่ตามธรรมเนียม (และเพื่อความไม่งง) ควรตั้งให้ตรงกันเสมอ ยกเว้นกรณีพิเศษเช่น `package main` ที่อยู่ในโฟลเดอร์ชื่อ `cmd/myapp/`
3. **ไฟล์ในโฟลเดอร์เดียวกันต้องประกาศ package ชื่อเดียวกันทั้งหมด** ถ้าไฟล์ใดในโฟลเดอร์เดียวกันประกาศ package ต่างชื่อ compiler จะ error ทันที
4. ชื่อ package ควรสั้น กระชับ สื่อความหมาย (เช่น `http`, `json`, `time`) — หลีกเลี่ยงชื่อที่ทั่วไปเกินไปเช่น `util`, `common`, `helper` เพราะทำให้ import แล้วอ่านไม่รู้เรื่องว่าใช้ทำอะไร

ลองสร้าง error จริงเพื่อดูว่า Go เข้มงวดแค่ไหน — ถ้ามีไฟล์สองไฟล์ในโฟลเดอร์เดียวกันประกาศ package ต่างกัน:

```
mypkg/
├── a.go   → package foo
└── b.go   → package bar
```

จะได้ error ทันทีตอน build:

```
found packages foo (a.go) and bar (b.go) in /path/to/mypkg
```

---

## 3. หนึ่ง package หลายไฟล์ (multi-file package)

ข้อดีอย่างหนึ่งของ Go คือ package หนึ่งสามารถแบ่งเป็นหลายไฟล์ได้ตามความเหมาะสม โดย**ทุกไฟล์ในโฟลเดอร์เดียวกันจะมองเห็นกันหมดโดยอัตโนมัติ** ไม่ต้อง import ข้ามไฟล์ภายใน package เดียวกัน

ตัวอย่าง: โปรเจกต์ `demo01` ที่มี `package main` แยกเป็น 2 ไฟล์

`main.go`:

```go
package main

import (
	"fmt"

	"github.com/google/uuid"

	"example.com/demo01/mathutil"
)

func main() {
	fmt.Println("ผลรวม:", sum(3, 4))
	fmt.Println("ผลคูณจาก mathutil:", mathutil.Multiply(6, 7))
	fmt.Println(greet("Go"))
	fmt.Println("Request ID ใหม่:", uuid.New())
}

func sum(a, b int) int {
	return a + b
}
```

`greet.go` (ไฟล์ที่สองใน**package main เดียวกัน**):

```go
package main

import "fmt"

// greet คืน string ทักทาย โดยไฟล์นี้อยู่ใน package main เดียวกันกับ main.go
func greet(name string) string {
	return fmt.Sprintf("สวัสดี, %s!", name)
}
```

สังเกตว่า `main.go` เรียก `sum(...)` และ `greet(...)` ได้เลยโดยไม่ต้อง import อะไรเพิ่ม เพราะทั้งสองไฟล์อยู่ใน package `main` เดียวกัน — Go compiler จะรวมทุกไฟล์ในโฟลเดอร์เดียวกันเข้าด้วยกันเป็น "หน่วยคอมไพล์" เดียวก่อนตรวจสอบ type และ compile จริง

การแบ่งไฟล์แบบนี้เป็นเรื่องของ**การจัดระเบียบเพื่อความอ่านง่าย** ล้วนๆ ไม่ได้มีผลต่อพฤติกรรมโปรแกรมเลย ทีมงานจริงมักแบ่งไฟล์ตามหน้าที่ เช่น `user.go`, `user_handler.go`, `user_repository.go` ทั้งหมดอาจอยู่ใน package `user` เดียวกัน

---

## 4. package name กับ import path ต่างกันอย่างไร

นี่เป็นจุดที่ผู้เริ่มต้นสับสนบ่อยที่สุด — **ชื่อ package (package name)** กับ **import path** เป็นคนละเรื่องกัน:

- **import path** คือ "ที่อยู่" แบบเต็มที่ใช้ใน `import "..."` — บอกว่า package นี้หาได้จากไหน (เช่น `example.com/demo01/mathutil`)
- **package name** คือชื่อสั้นๆ ที่ประกาศไว้ใน `package xxx` ของไฟล์นั้น และเป็นชื่อที่ใช้**อ้างอิงถึงสมาชิกใน code** (เช่น `mathutil.Multiply(...)`)

ปกติสองอย่างนี้จะเป็นชื่อสุดท้ายของ path เดียวกัน (`.../mathutil` → `package mathutil`) แต่**ไม่จำเป็นต้องตรงกันเสมอ** ตัวอย่างในโลกจริง: `import "gopkg.in/yaml.v3"` แต่ประกาศเป็น `package yaml` — เวลาโค้ดจะเรียกใช้ผ่านชื่อ `yaml.Marshal(...)` ไม่ใช่ `yaml_v3.Marshal(...)`

ถ้า package name กับส่วนท้ายของ path ไม่ตรงกัน เราสามารถตั้งชื่อเรียกเอง (import alias) ได้:

```go
import (
	realname "example.com/some/path-that-has-different-name"
)

func main() {
	realname.DoSomething()
}
```

alias ยังมีประโยชน์เมื่อ import สอง package ที่มีชื่อชนกัน เช่น:

```go
import (
	stdjson "encoding/json"
	jsoniter "github.com/json-iterator/go"
)
```

---

## 5. Go Module คืออะไร แก้ปัญหาอะไร

ตามที่กล่าวไปใน Part 001 ว่าก่อนปี 2018 Go ใช้ระบบ `GOPATH` บังคับให้โค้ดทุกโปรเจกต์ต้องอยู่ใต้ `$GOPATH/src` และไม่มีระบบ versioning ของ dependency ที่ดี ตั้งแต่ Go 1.11 เป็นต้นมา (และเป็นค่า default ตั้งแต่ Go 1.16) Go ใช้ **Go Modules** เป็นระบบจัดการ dependency มาตรฐาน

**Module** คือกลุ่มของ package ที่ **release และ version ร่วมกัน** เป็นหน่วยเดียว โดยมีไฟล์ `go.mod` อยู่ที่ root ของโปรเจกต์เป็นตัวกำหนดขอบเขต หนึ่ง repository มักมีหนึ่ง module (แต่ก็มี multi-module repository ได้ในกรณีพิเศษ)

ปัญหาที่ Go Module แก้ได้:

1. **โปรเจกต์อยู่ที่ไหนในเครื่องก็ได้** ไม่ต้องอยู่ใต้ `GOPATH/src` อีกต่อไป
2. **Versioning ที่ชัดเจน** — ระบุ dependency แต่ละตัวพร้อมเลขเวอร์ชันแบบ [Semantic Versioning](https://semver.org/) (`v1.2.3`)
3. **Reproducible build** — ไฟล์ `go.sum` เก็บ checksum ของทุก dependency เพื่อยืนยันว่าโค้ดที่ดาวน์โหลดมาไม่ถูกแก้ไข
4. **ทำงานแบบ offline ได้** หลัง cache dependency ไว้ในเครื่องแล้ว (`$GOPATH/pkg/mod`)

---

## 6. `go mod init` และการสร้างโมดูลแรก

สร้างโฟลเดอร์โปรเจกต์แล้วรัน `go mod init` พร้อมระบุ **module path**:

```bash
mkdir demo01
cd demo01
go mod init example.com/demo01
```

ผลลัพธ์:

```
go: creating new go.mod: module example.com/demo01
```

จะได้ไฟล์ `go.mod` หน้าตาประมาณนี้:

```
module example.com/demo01

go 1.22
```

### เลือก module path อย่างไร

- ถ้าโปรเจกต์จะถูก publish ให้คนอื่น import ผ่าน Git repository จริง (GitHub, GitLab) ให้ใช้ path ที่ตรงกับที่อยู่ repository เช่น `github.com/yourname/yourrepo` — เพราะ `go get` จะใช้ path นี้ในการดาวน์โหลดโค้ดจริงจาก Git
- ถ้าเป็นโปรเจกต์ภายใน ไม่ publish ที่ไหน จะตั้งเป็นอะไรก็ได้ตามธรรมเนียมทีม เช่น `example.com/myapp` หรือแม้แต่ชื่อสั้นๆ อย่าง `myapp` (แต่ไม่แนะนำถ้ามีแผนจะแตก sub-package หลายตัว เพราะ path จะดูไม่เป็นมาตรฐาน)

---

## 7. ผ่าโครงสร้างไฟล์ `go.mod` ทีละบรรทัด

มาดู `go.mod` ที่ซับซ้อนขึ้นอีกนิด (มี dependency ภายนอกแล้ว) ทีละบรรทัด:

```
module example.com/demo01

go 1.22

require github.com/google/uuid v1.6.0
```

| บรรทัด/directive | ความหมาย |
|---|---|
| `module example.com/demo01` | ประกาศ **module path** — นี่คือ "ราก" ของ import path ทั้งหมดในโปรเจกต์นี้ package ย่อยเช่น `example.com/demo01/mathutil` จะต้องขึ้นต้นด้วย path นี้เสมอ |
| `go 1.22` | ประกาศว่าโค้ดในโมดูลนี้เขียนโดยอ้างอิง **language version** ของ Go 1.22 เป็นอย่างต่ำ — มีผลต่อว่า compiler จะเปิดใช้ syntax/behavior ใหม่ของเวอร์ชันไหนบ้าง (เช่น `range over int` ที่จะเรียนใน Part 005 ต้องมี `go 1.22` ขึ้นไป) |
| `require github.com/google/uuid v1.6.0` | ระบุว่าโมดูลนี้พึ่งพา package `github.com/google/uuid` เวอร์ชัน `v1.6.0` โดยตรง |

เมื่อ dependency มีมากขึ้น `require` จะถูกจัดกลุ่มเป็นบล็อกวงเล็บ:

```
require (
	github.com/google/uuid v1.6.0
	github.com/gin-gonic/gin v1.10.0
)
```

### directive อื่นที่จะเจอเมื่อโปรเจกต์โตขึ้น (เรียนเจาะลึกใน Part 018)

| directive | ใช้ทำอะไร |
|---|---|
| `require ... // indirect` | บอกว่า dependency ตัวนี้ไม่ได้ถูก import ตรงจากโค้ดเรา แต่เป็น dependency ของ dependency อีกที |
| `replace` | สลับ path หรือเวอร์ชันของ dependency ชั่วคราว เช่น ใช้โค้ด local ระหว่างพัฒนาแทนเวอร์ชันบน GitHub |
| `exclude` | ห้ามใช้ dependency เวอร์ชันใดเวอร์ชันหนึ่งโดยเฉพาะ |
| `retract` | module author ประกาศถอนเวอร์ชันของตัวเองที่มีปัญหา (เช่นมี security bug) |

บทนี้ขอโฟกัสแค่ `module`, `go`, และ `require` พื้นฐานก่อน ส่วน `replace`/`exclude`/private module จะเรียนเจาะลึกใน **Part 018**

---

## 8. `go.sum` คืออะไร ทำไมต้องมี

เมื่อโปรเจกต์มี dependency ภายนอก Go จะสร้างไฟล์ `go.sum` คู่กับ `go.mod` เสมอ ลองดูตัวอย่างจริงจากการ `go get github.com/google/uuid`:

```
github.com/google/uuid v1.6.0 h1:NIvaJDMOsjHA8n1jAhLSgzrAzy1Hgr+hNrb57e+94F0=
github.com/google/uuid v1.6.0/go.mod h1:TIyPZe4MgqvfeYDBFedMoGGpEw/LqOeaOT+nhxU+yHo=
```

แต่ละบรรทัดคือ **cryptographic checksum (hash)** ของเนื้อหาไฟล์ package เวอร์ชันนั้นๆ จุดประสงค์คือ:

- **ความปลอดภัย**: ป้องกันไม่ให้ dependency ที่ดาวน์โหลดมาถูกแก้ไข/ปลอมแปลงระหว่างทาง (supply chain attack)
- **Reproducibility**: ทุกเครื่องที่ build โปรเจกต์เดียวกันจะได้ dependency เนื้อหาเดียวกันเป๊ะ ไม่ใช่แค่เลขเวอร์ชันตรงกัน

กฎสำคัญ: **`go.sum` ต้อง commit เข้า version control เสมอ** เช่นเดียวกับ `go.mod` (ต่างจาก `node_modules` หรือ cache directory อื่นๆ ที่มักถูก ignore) ห้าม `.gitignore` ไฟล์นี้เด็ดขาด

---

## 9. การ resolve import path: standard library, module ของเรา, module ภายนอก

เวลาเราเขียน `import "..."` ตัว Go toolchain จะ resolve path ตามลำดับกฎนี้:

1. **Standard library** — ถ้า path ตรงกับชื่อ package ในตัว Go เอง (เช่น `fmt`, `strings`, `net/http`) จะดึงจาก `GOROOT` ที่ติดตั้งไว้ทันที ไม่ต้องพึ่ง internet หรือ `go.mod`
2. **Package ภายใน module เดียวกัน** — ถ้า path ขึ้นต้นด้วย module path ของเราเอง (ตามที่ประกาศใน `go.mod`) เช่น module คือ `example.com/demo01` แล้ว import `example.com/demo01/mathutil` จะหาไฟล์จากโฟลเดอร์ `mathutil/` ในโปรเจกต์เราเอง
3. **Module ภายนอก** — ถ้า path ไม่ตรงกับสองข้อบน จะไปค้นหาใน `require` ของ `go.mod` แล้วดึงจาก cache ที่เครื่อง (`$GOPATH/pkg/mod`) หรือดาวน์โหลดผ่าน `GOPROXY` ถ้ายังไม่มี

ตัวอย่างการ import ครบทั้ง 3 แบบในไฟล์เดียว:

```go
package main

import (
	"fmt"                       // (1) standard library

	"github.com/google/uuid"    // (3) module ภายนอก ต้องมีใน go.mod

	"example.com/demo01/mathutil" // (2) package ภายใน module เดียวกัน
)
```

ธรรมเนียมการจัดกลุ่ม import (ที่ `gofmt`/`goimports` มักจัดให้อัตโนมัติ) คือแยกกลุ่มด้วยบรรทัดว่างตามลำดับ: standard library ก่อน แล้วตามด้วย third-party และ internal package ตามลำดับ (ทีมส่วนใหญ่ใช้ 2 กลุ่ม คือ standard library กับที่เหลือรวมกัน)

---

## 10. `go mod tidy` ใช้เมื่อไหร่ ทำอะไรบ้าง

`go mod tidy` เป็นคำสั่งที่ควรรันบ่อยที่สุดในบรรดาคำสั่ง `go mod` ทั้งหมด หน้าที่ของมันคือทำให้ `go.mod` และ `go.sum` **ตรงกับสิ่งที่โค้ดใช้จริง**:

- **เพิ่ม** entry ใน `require` สำหรับทุก package ที่โค้ดเรา import แต่ยังไม่มีใน `go.mod`
- **ลบ** entry ที่มีใน `go.mod` แต่ไม่มีโค้ดส่วนไหน import ใช้แล้ว (dependency ที่เลิกใช้)
- อัปเดต `go.sum` ให้ครบทุก checksum ที่จำเป็น

ทดลองจริง: เริ่มจากยังไม่มี `require` เลย แล้วเขียนโค้ดที่ import `github.com/google/uuid` ไปตรงๆ โดยยังไม่ได้รัน `go get`:

```bash
go run .
```

จะเจอ error ประมาณ:

```
main.go:7:2: no required module provides package github.com/google/uuid; to add it:
	go get github.com/google/uuid
```

Go **ไม่เดา**ให้เองว่าจะเอา dependency เวอร์ชันไหน ต้องสั่งอย่างใดอย่างหนึ่ง:

```bash
go get github.com/google/uuid   # เพิ่ม dependency ตัวใหม่ (หรืออัปเดตตัวเดิม) พร้อมแก้ go.mod/go.sum ให้
```

หรือถ้ามีหลาย import ที่ขาดหายพร้อมกัน (เช่น copy โค้ดจากที่อื่นมาทั้งไฟล์) ให้ใช้:

```bash
go mod tidy   # สแกนทั้งโปรเจกต์ แล้วจัดการ require ทั้งหมดให้ครบในทีเดียว
```

ก่อนและหลังรัน `go mod tidy` กับโปรเจกต์ demo01 (มี `uuid` เป็น `// indirect` เพราะได้มาจาก `go get` ตรงๆ แต่ยังไม่ถูกใช้จริงในโค้ด):

```
# ก่อน — ยังไม่มีโค้ดเรียกใช้ uuid จริง
require github.com/google/uuid v1.6.0 // indirect
```

```
# หลัง — โค้ดเรียก uuid.New() แล้ว go mod tidy ลบคำว่า indirect ออกให้อัตโนมัติ
require github.com/google/uuid v1.6.0
```

comment `// indirect` หมายความว่า Go **ตรวจไม่พบ**ว่ามีโค้ดส่วนไหน import package นี้โดยตรง (อาจจะเผลอ `go get` ไว้เฉยๆ หรือมันถูกดึงมาเป็น dependency ของอีก package หนึ่ง) พอเราเขียนโค้ดเรียกใช้งานจริงและรัน `go mod tidy` comment นี้จะหายไป

> **แนวทางปฏิบัติที่ดี**: รัน `go mod tidy` ทุกครั้งก่อน commit โค้ด เพื่อให้ `go.mod`/`go.sum` สะอาดและตรงกับโค้ดจริงเสมอ หลายทีมตั้งเป็นขั้นตอนบังคับใน CI (`go mod tidy && git diff --exit-code go.mod go.sum` เพื่อเช็คว่าไม่มีใครลืมรัน)

---

## 11. คำสั่ง `go mod` อื่นๆ ที่ควรรู้

| คำสั่ง | หน้าที่ |
|---|---|
| `go mod init <path>` | สร้าง `go.mod` ใหม่ |
| `go mod tidy` | จัดการ require ให้ตรงกับโค้ดจริง (เพิ่ม/ลบ) |
| `go mod download` | ดาวน์โหลด dependency ทั้งหมดตาม `go.mod` มาเก็บ cache โดยไม่แก้ไฟล์ |
| `go mod verify` | ตรวจสอบว่า dependency ใน cache ตรงกับ checksum ใน `go.sum` ไม่ถูกแก้ไข |
| `go mod graph` | แสดง dependency graph ทั้งหมดแบบ text |
| `go mod why <package>` | อธิบายว่าทำไมโปรเจกต์ถึงพึ่งพา package นี้ (ผ่าน chain ไหน) |
| `go mod vendor` | copy dependency ทั้งหมดมาเก็บไว้ในโฟลเดอร์ `vendor/` ภายในโปรเจกต์ (ใช้เมื่อต้องการ build แบบไม่พึ่ง network เลย) |
| `go list -m all` | แสดงรายการ module ทั้งหมดที่ใช้ พร้อมเวอร์ชัน |

ตัวอย่างการใช้ `go mod why` เพื่อสืบว่าทำไมถึงมี dependency ตัวหนึ่งโผล่มา (มีประโยชน์มากตอน debug dependency tree ที่ซับซ้อน):

```bash
go mod why github.com/google/uuid
```

```
# github.com/google/uuid
example.com/demo01
github.com/google/uuid
```

ผลลัพธ์อ่านจากบนลงล่างเป็น chain ว่าใครเรียกใคร — ในที่นี้คือโปรเจกต์เราเรียก `uuid` ตรงๆ

---

## 12. Semantic Import Versioning เบื้องต้น (v2+)

Go module ใช้ [Semantic Versioning](https://semver.org/) เต็มรูปแบบ: `vMAJOR.MINOR.PATCH` เช่น `v1.6.0`

- **PATCH** (`v1.6.0` → `v1.6.1`) แก้ bug อย่างเดียว ไม่เปลี่ยน API
- **MINOR** (`v1.6.0` → `v1.7.0`) เพิ่มความสามารถใหม่ แบบ backward-compatible
- **MAJOR** (`v1.x.x` → `v2.0.0`) มีการเปลี่ยนแปลงที่ **breaking change** ต่อ API เดิม

จุดที่ Go พิเศษกว่าภาษาอื่นคือกฎ **Semantic Import Versioning**: ตั้งแต่ major version `v2` ขึ้นไป **import path ต้องมีเลข major version ต่อท้ายด้วยเสมอ**

```go
import "github.com/some/pkg"       // v0 หรือ v1 — ไม่มีเลขต่อท้าย
import "github.com/some/pkg/v2"    // v2.x.x — ต้องมี /v2 ต่อท้าย path
import "github.com/some/pkg/v3"    // v3.x.x — ต้องมี /v3 ต่อท้าย path
```

เหตุผลของกฎนี้: การมี major version ต่างกันถือว่าเป็น "package คนละตัวกัน" ในทาง Go tooling — ทำให้โปรเจกต์เดียวสามารถ import `pkg/v1` และ `pkg/v2` **พร้อมกันในเวลาเดียวกันได้** (เช่น ระหว่างช่วง migrate จาก v1 ไป v2 แบบค่อยเป็นค่อยไป) โดยไม่ชนกัน เพราะ Go มองว่าเป็นคนละ import path กันเลย

เรื่องนี้จะเจอเจาะลึกอีกครั้งพร้อมวิธี publish module เวอร์ชัน major ใหม่ใน **Part 108 (Go Modules Best Practices และ Semantic Versioning)** ตอนนี้แค่จำหลักการไว้ก่อนว่า **"เจอ `/v2`, `/v3` ต่อท้าย import path ให้รู้ทันทีว่านี่คือ major version 2 ขึ้นไป"**

---

## 13. Visibility กับขอบเขตของ package (ทบทวนและขยายจาก Part 001)

ใน Part 001 เราเรียนกฎ export ไปแล้ว: **ตัวพิมพ์ใหญ่ = export, ตัวพิมพ์เล็ก = private** ตอนนี้เรามาดูว่ากฎนี้ทำงานร่วมกับ package อย่างไรบ้าง

จากตัวอย่าง `mathutil.go`:

```go
package mathutil

// Multiply คูณเลขจำนวนเต็มสองตัว (ขึ้นต้นตัวใหญ่ = export ใช้จากนอก package ได้)
func Multiply(a, b int) int {
	return a * b
}

// divideHelper เป็นฟังก์ชันภายใน เข้าถึงได้เฉพาะใน package mathutil เท่านั้น
func divideHelper(a, b int) int {
	if b == 0 {
		return 0
	}
	return a / b
}
```

- `Multiply` ขึ้นต้นด้วยตัวใหญ่ → **exported** → package อื่น import แล้วเรียก `mathutil.Multiply(...)` ได้
- `divideHelper` ขึ้นต้นด้วยตัวเล็ก → **unexported** → เรียกได้เฉพาะจากภายใน package `mathutil` เท่านั้น ถ้า `main.go` (คนละ package) พยายามเรียก `mathutil.divideHelper(...)` จะเจอ compile error ทันที:

```
mathutil.divideHelper undefined (cannot refer to unexported name mathutil.divideHelper)
```

ประเด็นสำคัญที่ขยายเพิ่มจาก Part 001 คือ **ขอบเขตของ "private" ใน Go คือระดับ package ไม่ใช่ระดับไฟล์** พูดอีกแบบ: ฟังก์ชัน `divideHelper` แม้จะ unexported แต่ก็ถูกเรียกได้จาก**ทุกไฟล์**ใน package `mathutil` (สมมติมีไฟล์ `mathutil_extra.go` ในโฟลเดอร์เดียวกัน ก็เรียก `divideHelper` ได้ทันทีโดยไม่ต้อง import อะไร) เพราะทุกไฟล์ในโฟลเดอร์เดียวกันถือเป็น "หน่วยเดียวกัน" ในสายตา compiler ตามที่อธิบายไปในหัวข้อที่ 3

กฎ export นี้ใช้เหมือนกันหมดไม่ว่ากับ function, struct, struct field, method, const, var — เราจะเห็นซ้ำไปเรื่อยๆ ตลอดหลักสูตร โดยเฉพาะตอนเรียน struct (Part 011) และ interface (Part 013)

### สรุปตารางเปรียบเทียบ private ใน Go กับภาษาอื่น

| ภาษา | ระดับการเข้าถึงที่มี | Go เทียบเท่าอะไร |
|---|---|---|
| Java/C# | `public`, `private`, `protected`, `package-private` | Go มีแค่ 2 ระดับ: exported (public) และ unexported (package-private) — **ไม่มี** `protected` เพราะไม่มี inheritance |
| Python | ไม่มีบังคับจริง (ใช้ `_` นำหน้าเป็น convention เท่านั้น) | Go บังคับจริงระดับ compiler ผ่านตัวพิมพ์ใหญ่/เล็ก |
| Go | exported / unexported | กำหนดจากตัวอักษรตัวแรกของชื่อเท่านั้น ไม่มี keyword พิเศษ |

---

## 14. โครงสร้างโปรเจกต์จริงที่ประกอบด้วยหลาย package

มาดูภาพรวมว่าเมื่อโปรเจกต์โตขึ้น การแบ่ง package หลายตัวภายใน module เดียวหน้าตาเป็นอย่างไร โดยต่อยอดจากโครงสร้างที่แนะนำใน Part 001:

```
demo01/                          ← root ของ module (มี go.mod ที่นี่)
├── go.mod                       ← module example.com/demo01
├── go.sum
├── main.go                      ← package main
├── greet.go                     ← package main (ไฟล์ที่ 2 ของ package เดียวกัน)
└── mathutil/                    ← โฟลเดอร์ = package ใหม่
    └── mathutil.go              ← package mathutil
```

กฎที่ยึดตลอด:

1. **หนึ่ง module มี `go.mod` เดียวที่ root** (ในกรณีทั่วไป — multi-module repo เป็นกรณีพิเศษที่ไม่ได้พูดถึงในบทนี้)
2. **หนึ่งโฟลเดอร์ = หนึ่ง package เสมอ** (ยกเว้นไม่มีไฟล์ `.go` เลย)
3. **import path ของ package ภายใน = module path + ที่อยู่โฟลเดอร์สัมพัทธ์จาก root** เช่น โฟลเดอร์ `mathutil/` ที่ root จะมี import path เป็น `example.com/demo01/mathutil`
4. ถ้ามีโฟลเดอร์ซ้อนลึกลงไปอีก เช่น `mathutil/stats/`, import path ก็จะเป็น `example.com/demo01/mathutil/stats` ตามลำดับ

เมื่อโปรเจกต์ใหญ่ขึ้นเรื่อยๆ เราจะกลับมาใช้โครงสร้างแบบ `cmd/`, `internal/`, `pkg/` ที่แนะนำไว้ใน Part 001 อย่างจริงจังในภาคที่ 10 (โปรเจกต์จริง) — ตอนนี้จำหลักการเรื่อง "โฟลเดอร์ = package" ให้แม่นเป็นพอ เพราะจะเป็นฐานของทุกอย่างที่ตามมา

---

## สรุปสิ่งที่ได้เรียนในบทนี้

- **package** คือหน่วยจัดระเบียบโค้ดพื้นฐานของ Go ผูกกับโฟลเดอร์อย่างเข้มงวด: ไฟล์ในโฟลเดอร์เดียวกันต้องอยู่ package เดียวกันเสมอ
- `package main` ใช้ compile เป็น executable (ต้องมี `func main()`) ส่วนชื่ออื่นคือ library package
- หนึ่ง package แบ่งเป็นหลายไฟล์ได้ ไฟล์ในโฟลเดอร์เดียวกันมองเห็นกันหมดโดยไม่ต้อง import
- **import path** (ที่อยู่เต็ม) กับ **package name** (ชื่อใช้เรียกในโค้ด) เป็นคนละเรื่องกัน ใช้ alias ได้เมื่อจำเป็น
- **Go Module** คือกลุ่ม package ที่ version ร่วมกัน สร้างด้วย `go mod init <module-path>` ได้ไฟล์ `go.mod`
- `go.mod` ประกอบด้วย `module` (ราก import path), `go` (language version ขั้นต่ำ), และ `require` (รายการ dependency)
- `go.sum` เก็บ checksum ของทุก dependency เพื่อความปลอดภัยและ reproducibility — ต้อง commit เสมอ
- Import path resolve ตามลำดับ: standard library → package ภายใน module ตัวเอง → module ภายนอกใน `require`
- `go mod tidy` คือคำสั่งที่ใช้บ่อยที่สุด เพิ่ม/ลบ `require` ให้ตรงกับโค้ดจริงเสมอ ควรรันก่อน commit ทุกครั้ง
- Semantic Import Versioning: module major version `v2` ขึ้นไป **ต้อง**มี `/v2`, `/v3` ต่อท้าย import path
- กฎ export (ตัวใหญ่/ตัวเล็ก) ทำงานที่ระดับ **package** ไม่ใช่ระดับไฟล์ — unexported เรียกได้จากทุกไฟล์ใน package เดียวกัน

## แบบฝึกหัดท้ายบท

1. สร้างโปรเจกต์ใหม่ด้วย `go mod init` ตั้งชื่อ module เอง แล้วเปิดดูเนื้อหาไฟล์ `go.mod` ที่ได้ อธิบายแต่ละบรรทัดด้วยคำพูดตัวเอง
2. สร้าง sub-package ของตัวเอง (เช่น `stringutil`) ที่มีฟังก์ชัน exported อย่างน้อย 1 ตัวและ unexported อย่างน้อย 1 ตัว แล้ว import มาใช้ใน `main.go` ลองเรียกฟังก์ชัน unexported จากภายนอกดูว่า error หน้าตาเป็นอย่างไร
3. แบ่ง `package main` ของโปรเจกต์ในข้อ 2 ออกเป็น 3 ไฟล์ (เช่น `main.go`, `handler.go`, `util.go`) แล้วยืนยันว่ายังคง `go run .` ได้ปกติโดยไม่ต้อง import อะไรเพิ่มระหว่างไฟล์
4. ลอง `go get` package ภายนอกสักตัว (เช่น `github.com/google/uuid`) โดยยังไม่เขียนโค้ดเรียกใช้ สังเกตว่า `go.mod` มี comment `// indirect` หรือไม่ แล้วเขียนโค้ดเรียกใช้งานจริง รัน `go mod tidy` แล้วดูว่า comment หายไปหรือไม่ อธิบายว่าทำไม
5. รันคำสั่ง `go mod why <package>` และ `go mod graph` กับโปรเจกต์ที่มี dependency ภายนอก แล้วอธิบายผลลัพธ์ที่ได้
6. ค้นหาใน GitHub หา open source Go package ที่มี import path ลงท้ายด้วย `/v2` หรือ `/v3` มาอย่างน้อย 1 ตัว แล้วอธิบายว่าทำไม path ต้องมีเลขต่อท้ายแบบนั้น (เชื่อมโยงกับ Semantic Import Versioning ที่เรียนในบทนี้)

---

**ต่อไป**: [Part 003 — ตัวแปร, ชนิดข้อมูล, ค่าคงที่, `iota`](./003-variables-types-constants.md)
