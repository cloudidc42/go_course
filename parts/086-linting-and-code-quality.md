# Part 086: Linting ด้วย `golangci-lint` และ Code Quality

> ภาคที่ 7: Testing, Tooling, Performance — ตอนที่ 8 จาก 9 (Part 79–87)

## สารบัญของบทนี้

1. ทำไม `go vet` และ `gofmt` ไม่เพียงพอ
2. ติดตั้ง `golangci-lint` จริงบนเครื่อง
3. ตัวอย่างโค้ดที่มีปัญหาโดยตั้งใจ และการรัน `golangci-lint run` จริง
4. อ่านผลลัพธ์ทีละปัญหา: errcheck, ineffassign, unused, staticcheck
5. เปิดใช้งาน `govet` แบบเจาะลึกขึ้นด้วยการตั้งค่า `shadow`
6. ตั้งค่า `.golangci.yml` สำหรับทีม
7. Linter ชุดมาตรฐานที่ทีมส่วนใหญ่ใช้ และชื่อ linter ที่เปลี่ยนไปใน golangci-lint v2
8. ผูก Lint เข้ากับ Pre-commit Hook และ CI (เกริ่นนำ Part 098)
9. ธรรมเนียมการ Code Review เฉพาะของ Go: Effective Go และ Go Code Review Comments
10. สรุปสิ่งที่ได้เรียนในบทนี้
11. แบบฝึกหัดท้ายบท

---

## 1. ทำไม `go vet` และ `gofmt` ไม่เพียงพอ

ย้อนกลับไปที่ **Part 001** เราเรียนไปแล้วว่า Go มาพร้อมเครื่องมือในตัวสองอย่างที่ช่วยเรื่องคุณภาพโค้ด:

- **`gofmt`** (เรียกผ่าน `go fmt`) จัดรูปแบบโค้ดให้เป็นมาตรฐานเดียวกันทั้งหมด (เว้นวรรค, tab, ตำแหน่งวงเล็บ) — แก้ปัญหาเรื่อง "สไตล์การเขียนโค้ด" ให้หมดไปโดยสิ้นเชิง
- **`go vet`** ตรวจจับข้อผิดพลาดเชิง logic ที่ compiler มองไม่เห็น เช่น `Printf` ที่ใส่ format verb ไม่ตรงกับ argument, การ copy struct ที่มี `sync.Mutex` อยู่ข้างใน, หรือการเปรียบเทียบ struct ด้วย `==` ที่มี field เปรียบเทียบไม่ได้

ทั้งสองตัวนี้ **มีประโยชน์มากและควรใช้เสมอ** แต่ยังมีช่องว่างสำคัญที่มันตรวจจับไม่ได้ ลองดูตัวอย่างโค้ดที่มีปัญหาชัดเจนหลายจุด:

```go
package main

import "fmt"

func Greet(name string) string {
	msg := "Hello, "
	msg = msg + name // ไม่เคยถูกอ่านก่อนถูกเขียนทับด้านล่าง
	msg = fmt.Sprintf("Hi, %s!", name)
	return msg
}

func main() {
	x := 10
	if x > 5 {
		x := 20 // ตัวแปรชื่อซ้ำในขอบเขตย่อย บัง x ตัวนอกโดยไม่ตั้งใจ
		fmt.Println("inner x:", x)
	}
	fmt.Println("outer x:", x)
	fmt.Println(Greet("Go"))
}
```

รัน `go vet` กับโค้ดนี้:

```bash
go vet ./...
```

ผลลัพธ์จริง:

```
(ไม่มีข้อความใดๆ ออกมาเลย — exit code 0)
```

**`go vet` ไม่รายงานอะไรเลย** ทั้งที่โค้ดนี้มีปัญหาชัดเจนอย่างน้อยสองจุด: บรรทัด `msg = msg + name` เขียนค่าทิ้งโดยไม่เคยอ่าน (**ineffectual assignment**) และตัวแปร `x` ตัวในถูกประกาศซ้ำบัง `x` ตัวนอกโดยไม่ตั้งใจ (**variable shadowing**) — ปัญหาแบบหลังนี้อันตรายมากในโค้ดจริง เพราะถ้าโปรแกรมเมอร์ตั้งใจจะแก้ไข `x` ตัวนอกแต่พิมพ์ `:=` แทน `=` โดยไม่ทันสังเกต การเปลี่ยนแปลงจะหายไปเงียบๆ โดยไม่มี error หรือ warning ใดๆ

### เหตุผลที่ `go vet` ไม่ครอบคลุมเรื่องเหล่านี้

`go vet` ถูกออกแบบมาให้ **ระมัดระวังมาก** (conservative) โดยเจตนา — ทีม Go เลือกให้ `go vet` ตรวจเฉพาะสิ่งที่**แทบไม่มี false positive เลย** เพื่อให้มันทำงานเป็นส่วนหนึ่งของ `go test` โดยอัตโนมัติได้โดยไม่รบกวนการพัฒนา (ทบทวนจาก Part 033: `go test` จะรัน `go vet` ให้อัตโนมัติก่อนรัน test เสมอ) การตรวจเรื่อง shadowing หรือ ineffectual assignment มีโอกาส false positive สูงกว่า (บางครั้งการ shadow ตัวแปรก็ตั้งใจทำจริงๆ) จึงถูกแยกออกไปเป็น **analyzer เสริม** ที่ไม่ได้เปิดใช้งานโดย default ใน `go vet`

นี่คือช่องว่างที่เครื่องมือ **linter รวมศูนย์ (meta-linter)** เข้ามาเติมเต็ม — และเครื่องมือที่เป็นมาตรฐานอุตสาหกรรมมากที่สุดในระบบนิเวศ Go ปัจจุบันคือ **`golangci-lint`**

---

## 2. ติดตั้ง `golangci-lint` จริงบนเครื่อง

`golangci-lint` คือตัวรวม **linter หลายสิบตัว** เข้าด้วยกัน (รวมถึง `go vet` เองด้วย) รันพร้อมกันแบบขนานอย่างมีประสิทธิภาพ แล้วรายงานผลลัพธ์แบบรวมศูนย์ในรูปแบบเดียว มีสองวิธีหลักในการติดตั้ง

### วิธีที่ 1: ติดตั้งผ่านสคริปต์ทางการ (แนะนำสำหรับใช้งานทั่วไป)

```bash
curl -sSfL https://raw.githubusercontent.com/golangci/golangci-lint/master/install.sh | sh -s -- -b $(go env GOPATH)/bin
```

สคริปต์นี้ดาวน์โหลด binary ที่ build ไว้ล่วงหน้าตรงกับ OS/architecture ของเครื่อง แล้ววางไว้ที่ `$(go env GOPATH)/bin` (ทบทวน `GOPATH`/`GOBIN` จาก **Part 001**)

### วิธีที่ 2: ติดตั้งผ่าน `go install` (ควบคุมเวอร์ชันได้ชัดเจนกว่า)

```bash
go install github.com/golangci/golangci-lint/v2/cmd/golangci-lint@latest
```

นี่คือคำสั่งที่ใช้ทดสอบจริงตอนเขียนบทเรียนนี้ ผลลัพธ์จริงที่ได้:

```
go: github.com/golangci/golangci-lint/v2@v2.14.0 requires go >= 1.26.0; switching to go1.26.8
```

> **หมายเหตุ**: เหมือนที่เจอกับ `benchstat` ใน **Part 084** — `go install` ดาวน์โหลด Go toolchain เวอร์ชันใหม่กว่ามาใช้ชั่วคราวโดยอัตโนมัติ (`GOTOOLCHAIN=auto`) เมื่อ dependency ของเครื่องมือต้องการเวอร์ชันใหม่กว่าที่ติดตั้งอยู่ ไม่กระทบ Go เวอร์ชันหลักที่ใช้ compile โปรเจกต์ของเราเอง

ตรวจสอบว่าติดตั้งสำเร็จ:

```bash
golangci-lint --version
```

```
golangci-lint has version 2.14.0 built with go1.26.8 from (unknown, modified: ?, mod sum: "h1:ot8QffRa4LzAAEgtvNYrVs9esxQ7xoTwoAv6uhp4ngA=") on (unknown)
```

> **สำคัญ**: เวอร์ชันของ `golangci-lint` ที่ใช้ในบทนี้คือ **v2** (เปิดตัวปี 2025) ซึ่งเปลี่ยนแปลงจาก v1 พอสมควร ทั้งรูปแบบไฟล์ config (ต้องมี `version: "2"` กำกับ) และการรวม linter บางตัวเข้าด้วยกัน (จะอธิบายในหัวข้อ 7) ถ้าเครื่องของใครติดตั้ง v1 ไว้เดิม ควรอัปเดตเป็น v2 ตามคำสั่งด้านบน เพราะไฟล์ `.golangci.yml` ในบทนี้เขียนสำหรับ v2 โดยเฉพาะ

---

## 3. ตัวอย่างโค้ดที่มีปัญหาโดยตั้งใจ และการรัน `golangci-lint run` จริง

มาสร้างไฟล์ตัวอย่างที่มีปัญหาโดยตั้งใจครบทุกประเภทที่พบบ่อยในโค้ด Go จริง แล้วรัน `golangci-lint` จริงเพื่อดูว่ามันตรวจจับอะไรได้บ้าง:

```go
// main.go
package main

import (
	"fmt"
	"os"
)

const maxRetries = 5 // ไม่มีใครเรียกใช้ constant นี้เลยในโค้ดทั้งไฟล์

// Greet returns a greeting for the given name.
func Greet(name string) string {
	msg := "Hello, "
	msg = msg + name // จะกลายเป็น ineffectual assignment เพราะถูกเขียนทับด้านล่างโดยไม่เคยถูกอ่านก่อน
	msg = fmt.Sprintf("Hi, %s!", name)
	return msg
}

func cleanup(path string) {
	os.Remove(path) // errcheck: ไม่เช็คค่า error ที่ return กลับมา
}

func unusedHelper() int {
	return 42
}

func isValid(ok bool) bool {
	if ok == true { // staticcheck (S1002): เทียบ bool กับ true ตรงๆ โดยไม่จำเป็น
		return true
	}
	return false
}

func main() {
	x := 10
	if x > 5 {
		x := 20 // shadow: ประกาศตัวแปรชื่อซ้ำในขอบเขตย่อย บัง x ตัวนอก
		fmt.Println("inner x:", x)
	}
	fmt.Println("outer x:", x)

	result := Greet("Go")
	fmt.Println(result)

	cleanup("/tmp/does-not-exist-demo.txt")
	fmt.Println("isValid:", isValid(true))
}
```

ยืนยันก่อนว่าโค้ดนี้ **compile และรันได้ปกติ** (ปัญหาทั้งหมดเป็นเรื่อง code quality ไม่ใช่ compile error):

```bash
go build ./... && go run .
```

```
inner x: 20
outer x: 10
Hi, Go!
isValid: true
```

และยืนยันว่า `go vet` ผ่านสนิท ไม่เจออะไรเลย ตามที่อธิบายไว้ในหัวข้อ 1:

```bash
go vet ./...
```

```
(exit code 0, ไม่มีข้อความใดๆ)
```

ทีนี้รัน `golangci-lint` โดยยังไม่มีไฟล์ config ใดๆ (ใช้ค่า default ทั้งหมด):

```bash
golangci-lint run ./...
```

ผลลัพธ์จริง:

```
main.go:19:11: Error return value of `os.Remove` is not checked (errcheck)
	os.Remove(path) // errcheck: ไม่เช็คค่า error ที่ return กลับมา
	         ^
main.go:13:2: ineffectual assignment to msg (ineffassign)
	msg = msg + name // จะกลายเป็น ineffectual assignment เพราะถูกเขียนทับด้านล่างโดยไม่เคยถูกอ่านก่อน
	^
main.go:42:5: S1002: should omit comparison to bool constant, can be simplified to ok (staticcheck)
	if ok == true { // staticcheck (S1002): เทียบ bool กับ true ตรงๆ โดยไม่จำเป็น
	   ^
main.go:8:7: const maxRetries is unused (unused)
const maxRetries = 5 // ไม่มีใครเรียกใช้ constant นี้เลยในโค้ดทั้งไฟล์
      ^
main.go:22:6: func unusedHelper is unused (unused)
func unusedHelper() int {
     ^
5 issues:
* errcheck: 1
* ineffassign: 1
* staticcheck: 1
* unused: 2
```

**5 ปัญหาจริง** ที่ `go vet` มองไม่เห็นเลยแม้แต่ปัญหาเดียว! นี่คือสิ่งที่ **`golangci-lint` เพิ่มเข้ามาเหนือ `go vet` ธรรมดา** โดยยังไม่ได้ตั้งค่าอะไรเป็นพิเศษเลยด้วยซ้ำ (ใช้ชุด linter ที่เรียกว่า **"standard" preset** ซึ่งเป็นค่า default ของ `golangci-lint run` เมื่อไม่มีไฟล์ config)

---

## 4. อ่านผลลัพธ์ทีละปัญหา: errcheck, ineffassign, unused, staticcheck

มาดูรายละเอียดของแต่ละ linter ที่เพิ่งเจอ เพื่อเข้าใจว่ามันตรวจจับอะไรและทำไมมันสำคัญ

### `errcheck`: ตรวจจับ error ที่ไม่ได้เช็ค

```
main.go:19:11: Error return value of `os.Remove` is not checked (errcheck)
```

ทบทวนจาก **Part 015**: Go บังคับให้ error เป็นค่า return ธรรมดา ไม่ใช่ exception ที่บังคับให้ต้องจัดการ — ข้อดีคือความชัดเจน แต่ข้อเสียคือ **ไม่มีอะไรบังคับให้โปรแกรมเมอร์เช็ค error จริงๆ** เขียน `os.Remove(path)` เฉยๆ โดยไม่รับค่า return ก็ compile ผ่านได้สบาย ทั้งที่การลบไฟล์อาจล้มเหลวได้หลายสาเหตุ (ไฟล์ไม่มีอยู่, ไม่มีสิทธิ์เข้าถึง) `errcheck` คือ linter ที่อุดช่องโหว่นี้ — เป็นหนึ่งใน linter ที่ **สำคัญที่สุด** สำหรับทีมที่จริงจังเรื่อง error handling

### `ineffassign`: ตรวจจับการเขียนทับค่าที่ไม่เคยถูกอ่าน

```
main.go:13:2: ineffectual assignment to msg (ineffassign)
```

`msg = msg + name` คำนวณค่าขึ้นมาแล้วเก็บไว้ในตัวแปร `msg` แต่บรรทัดถัดไป (`msg = fmt.Sprintf(...)`) เขียนทับค่านั้นทันทีโดยไม่เคยอ่านค่าเก่าเลย — การคำนวณทั้งหมดในบรรทัดแรกจึง **สูญเปล่าโดยสมบูรณ์** สถานการณ์แบบนี้มักเกิดจากการ refactor โค้ดที่ทำไม่สมบูรณ์ (ลืมลบ logic เก่าที่ไม่จำเป็นแล้วออก) `ineffassign` ช่วยจับ "โค้ดที่ตายแล้วแต่ยังไม่รู้ตัว" แบบนี้ได้ดีมาก

### `unused`: ตรวจจับ identifier ระดับ package ที่ไม่มีใครใช้

```
main.go:8:7: const maxRetries is unused (unused)
main.go:22:6: func unusedHelper is unused (unused)
```

ทบทวนจาก **Part 001**: ตัวแปร local ที่ไม่ได้ใช้เป็น **compile error** ทันทีอยู่แล้ว (บังคับความสะอาดของโค้ดตั้งแต่ระดับ compiler) แต่กฎนี้ **ใช้ไม่ได้กับ identifier ระดับ package** เช่น function, constant, type, หรือ struct field ที่ export หรือ unexported ก็ตาม — โค้ดข้างบนมี `const maxRetries` และ `func unusedHelper` ที่ไม่มีใครเรียกใช้เลย แต่ compile ผ่านสบายเพราะกฎ "unused = compile error" ครอบคลุมแค่ตัวแปร local เท่านั้น linter `unused` เข้ามาอุดช่องว่างนี้ในระดับ package — มีประโยชน์มากในการเก็บกวาดโค้ดที่ตายแล้ว (dead code) ที่ค้างมาจากการ refactor ในโปรเจกต์ระยะยาว

### `staticcheck`: ชุดกฎมาตรฐานคุณภาพสูงจากชุมชน

```
main.go:42:5: S1002: should omit comparison to bool constant, can be simplified to ok (staticcheck)
```

`staticcheck` เป็นเครื่องมือ static analysis อิสระที่ทรงพลังมาก (สามารถใช้แยกเดี่ยวได้โดยไม่ต้องผ่าน `golangci-lint` ด้วย) `golangci-lint` นำมันมารวมไว้เป็นหนึ่งใน linter ของตัวเอง กฎรหัส `S1002` ตรวจพบว่า `if ok == true` เป็นการเขียนที่ยืดยาวโดยไม่จำเป็น เพราะ `ok` เป็น `bool` อยู่แล้ว เขียน `if ok` ตรงๆ ก็ได้ความหมายเดียวกันและอ่านง่ายกว่า — เป็นตัวอย่างของกฎประเภท **"simplification"** ที่ไม่ใช่บั๊ก แต่ช่วยให้โค้ดกระชับและเป็นสำนวน Go มากขึ้น (idiomatic)

---

## 5. เปิดใช้งาน `govet` แบบเจาะลึกขึ้นด้วยการตั้งค่า `shadow`

สังเกตว่าผลลัพธ์ในหัวข้อ 3 **ยังไม่มีปัญหาเรื่อง variable shadowing** ที่เห็นในโค้ด (`x := 20` ที่บังตัวแปร `x` ตัวนอก) ทั้งที่ปัญหานี้อันตรายมาก — เหตุผลคือ **`govet`** (ตัวห่อ `go vet` ภายใน `golangci-lint`) ใช้ชุด analyzer เริ่มต้นของ `go vet` เอง ซึ่ง **ไม่รวม `shadow` analyzer ไว้โดย default** (ตามเหตุผลที่อธิบายในหัวข้อ 1) ต้องเปิดใช้งานเพิ่มเองผ่านการตั้งค่า

สร้างไฟล์ `.golangci.yml`:

```yaml
version: "2"

linters:
  enable:
    - govet
    - staticcheck
    - errcheck
    - unused
  settings:
    govet:
      enable:
        - shadow
```

รันอีกครั้งด้วย config นี้:

```bash
golangci-lint run ./...
```

ผลลัพธ์จริง:

```
main.go:19:11: Error return value of `os.Remove` is not checked (errcheck)
	os.Remove(path) // errcheck: ไม่เช็คค่า error ที่ return กลับมา
	         ^
main.go:29:3: shadow: declaration of "x" shadows declaration at line 27 (govet)
		x := 20 // shadow: ประกาศตัวแปรชื่อซ้ำในขอบเขตย่อย บัง x ตัวนอก
		^
main.go:13:2: ineffectual assignment to msg (ineffassign)
	msg = msg + name // จะกลายเป็น ineffectual assignment เพราะถูกเขียนทับด้านล่างโดยไม่เคยถูกอ่านก่อน
	^
main.go:42:5: S1002: should omit comparison to bool constant, can be simplified to ok (staticcheck)
	if ok == true { // staticcheck (S1002): เทียบ bool กับ true ตรงๆ โดยไม่จำเป็น
	   ^
main.go:8:7: const maxRetries is unused (unused)
const maxRetries = 5 // ไม่มีใครเรียกใช้ constant นี้เลยในโค้ดทั้งไฟล์
      ^
main.go:22:6: func unusedHelper is unused (unused)
func unusedHelper() int {
     ^
6 issues:
* errcheck: 1
* govet: 1
* ineffassign: 1
* staticcheck: 1
* unused: 2
```

ตอนนี้ **`shadow: declaration of "x" shadows declaration at line 27 (govet)` ปรากฏขึ้นมาแล้ว** — รวมเป็น 6 ปัญหาทั้งหมด ครอบคลุมทุกจุดที่ตั้งใจใส่ไว้ในตัวอย่าง นี่คือเหตุผลที่การตั้งค่า `.golangci.yml` ให้เหมาะกับทีมมีความสำคัญมาก: **ค่า default ให้จุดเริ่มต้นที่ดี แต่การตั้งค่าเพิ่มเติมช่วยจับปัญหาที่ลึกและอันตรายกว่าได้อีกมาก**

---

## 6. ตั้งค่า `.golangci.yml` สำหรับทีม

มาดูโครงสร้างไฟล์ config แบบเต็มที่เหมาะสำหรับทีมจริง พร้อมอธิบายแต่ละส่วน:

```yaml
version: "2"

run:
  timeout: 5m        # เวลาสูงสุดที่ยอมให้ lint ทำงาน (โปรเจกต์ใหญ่อาจต้องเพิ่มค่านี้)

linters:
  # เปิดใช้งาน linter เพิ่มเติมนอกเหนือจากชุด default ("standard" preset)
  enable:
    - govet          # ห่อ go vet พร้อมเปิด analyzer เสริมได้ (เช่น shadow)
    - staticcheck     # รวม staticcheck + gosimple + stylecheck เดิม (อธิบายในหัวข้อ 7)
    - errcheck        # บังคับเช็ค error ที่ return กลับมา
    - unused          # ตรวจ identifier ระดับ package ที่ไม่มีใครใช้
    - revive          # ทางเลือกแทน golint เดิม (ตรวจสไตล์และ convention เช่น comment ของ exported identifier)

  # ปิด linter บางตัวที่ default เปิดไว้ แต่ทีมตัดสินใจว่าเข้มงวดเกินไปสำหรับโปรเจกต์นี้
  disable:
    - gosec           # เปิดทีหลังเมื่อทีมพร้อมจัดการปัญหาด้าน security ที่มันมักเจอเยอะในโค้ดเก่า

  settings:
    govet:
      enable:
        - shadow      # เปิด shadow analyzer เพิ่มเติมจากที่อธิบายในหัวข้อ 5
    errcheck:
      # อนุญาตให้ไม่เช็ค error ของฟังก์ชันที่ไม่สำคัญ (เช่น การปิด response body ใน defer)
      exclude-functions:
        - (net/http.ResponseWriter).Write

  exclusions:
    rules:
      # ผ่อนปรน error check ในไฟล์ test เพราะมักมีการเรียกฟังก์ชันที่ error แทบไม่เคยเกิดจริง
      - path: _test\.go
        linters:
          - errcheck

formatters:
  enable:
    - gofmt           # จัดรูปแบบตามมาตรฐาน (เหมือน go fmt)
    - goimports       # จัดเรียง import ให้เป็นระเบียบและเติม import ที่ขาดอัตโนมัติ

issues:
  max-issues-per-linter: 0   # 0 = ไม่จำกัดจำนวน (default จะตัดโชว์แค่ 50 รายการต่อ linter)
  max-same-issues: 0
```

### ประเด็นสำคัญที่ต้องรู้เกี่ยวกับ config นี้

- **`version: "2"`** เป็น field บังคับสำหรับ golangci-lint v2 — ถ้าไม่ใส่หรือใส่ผิด golangci-lint จะปฏิเสธไม่ยอมรันเลย
- **`linters.enable`** vs **`linters.disable`**: ค่า default ของ golangci-lint คือชุด **"standard" preset** (คือ `govet`, `staticcheck`, `errcheck`, `unused`, `ineffassign` ตามที่เห็นในหัวข้อ 3) การใช้ `enable`/`disable` คือการ**ปรับแต่งจากชุด default** นี้ ไม่ใช่กำหนดใหม่ทั้งหมด
- **`exclusions.rules`** มีประโยชน์มากในทางปฏิบัติ — โค้ด test มักมี pattern ที่ไม่อยากให้ linter บางตัวเข้มงวดเท่าโค้ด production (เช่น ไม่เช็ค error ของการเปิดไฟล์ test fixture ที่รู้อยู่แล้วว่าต้องมีอยู่จริง)
- **`formatters`** เป็น section ใหม่ใน v2 (แยกออกจาก `linters` อย่างชัดเจน เพราะ formatter แก้โค้ดให้อัตโนมัติได้ ในขณะที่ linter แค่รายงานปัญหา) รันแยกด้วยคำสั่ง `golangci-lint fmt`

ตรวจสอบว่า config ถูกต้องตาม schema ก่อนใช้งานจริงด้วยคำสั่ง:

```bash
golangci-lint config verify
```

---

## 7. Linter ชุดมาตรฐานที่ทีมส่วนใหญ่ใช้ และชื่อ linter ที่เปลี่ยนไปใน golangci-lint v2

โจทย์ที่โจทย์ของบทนี้ระบุไว้คือชุด linter มาตรฐาน `govet`, `staticcheck`, `errcheck`, `unused`, `gosimple` — แต่ถ้าลองเปิดใช้ `gosimple` ใน golangci-lint v2 ตรงๆ จะพบว่า:

```bash
golangci-lint linters 2>&1 | grep -i gosimple
```

```
(ไม่มีผลลัพธ์ใดๆ — ไม่พบ linter ชื่อ gosimple ใน v2)
```

**เหตุผล**: golangci-lint v2 **รวม `gosimple` (กฎ `S*`) และ `stylecheck` (กฎ `ST*`) เข้าไปเป็นส่วนหนึ่งของ `staticcheck` โดยตรง** แล้ว ไม่แยกเป็น linter คนละตัวเหมือน v1 อีกต่อไป — นี่คือเหตุผลที่ตัวอย่างในหัวข้อ 4 เห็นกฎ `S1002` (ซึ่งเดิมเป็นกฎของ `gosimple`) รายงานออกมาภายใต้ป้ายกำกับ `(staticcheck)` แทนที่จะเป็น `(gosimple)`

สรุปชุด linter มาตรฐานที่แนะนำสำหรับทีมส่วนใหญ่ (เทียบเท่ากับที่โจทย์ตั้งต้นไว้ แต่ปรับชื่อให้ตรงกับ v2):

| Linter | หน้าที่ | อยู่ใน default preset ของ v2 หรือไม่ |
|---|---|---|
| `govet` | ห่อ `go vet` พร้อมเปิด analyzer เสริมได้ (เช่น `shadow`) | ใช่ |
| `staticcheck` | รวม static analysis, simplification (`gosimple` เดิม), และ style (`stylecheck` เดิม) | ใช่ |
| `errcheck` | บังคับเช็ค error ที่ return กลับมา | ใช่ |
| `unused` | ตรวจ identifier ระดับ package ที่ไม่มีใครใช้ | ใช่ |
| `ineffassign` | ตรวจการเขียนทับค่าที่ไม่เคยถูกอ่าน | ใช่ |

ชุดนี้ (ที่จริงคือค่า default ของ `golangci-lint run` เองโดยไม่ต้องตั้งค่าอะไรเพิ่มเลย) ถือเป็นจุดเริ่มต้นที่แข็งแรงมากสำหรับเกือบทุกโปรเจกต์ Go — สิ่งที่ทีมมักเพิ่มเข้าไปเมื่อโตขึ้นคือ:

- **`revive`**: ตรวจสไตล์และ convention ระดับละเอียด (ชื่อ identifier, comment ของ exported item) — ตัวสืบทอดที่นิยมใช้แทน `golint` เดิมที่เลิกพัฒนาไปแล้ว
- **`gosec`**: ตรวจปัญหาด้าน security เบื้องต้น (เช่น การใช้ `crypto/md5`, SQL string concatenation ที่เสี่ยง injection)
- **`gocritic`**: ชุดกฎ opinionated จำนวนมากเกี่ยวกับ performance และ style
- **`bodyclose`**: ตรวจว่า `http.Response.Body` ถูกปิดเสมอ (เกี่ยวข้องกับ `net/http` ที่เรียนใน Part 046)

> **คำแนะนำในการนำไปใช้จริง**: เริ่มจากชุด default (5 ตัวข้างบน) ก่อนเสมอ อย่าเปิดทุก linter ที่มีพร้อมกันตั้งแต่วันแรก เพราะในโปรเจกต์ที่มีโค้ดเก่าอยู่แล้วจะเจอปัญหาเป็นร้อยเป็นพันจุดจนทีมท้อและเลิกสนใจ lint ไปเลย ค่อยๆ เพิ่มทีละตัวเมื่อทีมพร้อมจัดการปัญหาที่มันเจอ

---

## 8. ผูก Lint เข้ากับ Pre-commit Hook และ CI (เกริ่นนำ Part 098)

Lint ที่ไม่มีใครรันจะไม่มีประโยชน์อะไรเลย — วิธีที่มีประสิทธิภาพที่สุดคือทำให้มันรัน**อัตโนมัติ** ในสองจุด: ก่อน commit (บนเครื่องนักพัฒนาแต่ละคน) และใน CI (เป็นด่านสุดท้ายที่บังคับทุกคนไม่ให้หลบเลี่ยงได้)

### Pre-commit Hook

Git มีกลไก **hook** ในตัวอยู่แล้ว — ไฟล์ script ที่ Git เรียกอัตโนมัติในจังหวะต่างๆ ของ workflow สร้างไฟล์ `.git/hooks/pre-commit`:

```bash
#!/bin/sh
# .git/hooks/pre-commit
echo "Running golangci-lint before commit..."
golangci-lint run ./...
if [ $? -ne 0 ]; then
	echo "Lint ไม่ผ่าน — commit ถูกยกเลิก แก้ปัญหาก่อนแล้วลองใหม่"
	exit 1
fi
```

```bash
chmod +x .git/hooks/pre-commit
```

จากนี้ทุกครั้งที่รัน `git commit` Git จะรัน script นี้ก่อนเสมอ ถ้า `golangci-lint run` คืนค่า exit code ที่ไม่ใช่ 0 (คือเจอปัญหา) การ commit จะถูกยกเลิกทันที

> **ข้อจำกัดของ `.git/hooks/pre-commit` แบบดิบ**: ไฟล์นี้อยู่ใน `.git/` ซึ่ง**ไม่ถูก track โดย Git** หมายความว่า**ไม่สามารถแชร์ให้เพื่อนร่วมทีมได้อัตโนมัติ**ผ่านการ clone repository ทีมที่ต้องการบังคับให้ทุกคนใช้ hook เดียวกันจริงๆ มักใช้เครื่องมือเสริมอย่าง [`pre-commit`](https://pre-commit.com/) (เครื่องมือ framework จัดการ git hook ข้ามภาษา) หรือ [`lefthook`](https://github.com/evilmartians/lefthook) ที่เก็บ config ไว้เป็นไฟล์ในโปรเจกต์ (เช่น `.pre-commit-config.yaml`) แล้ว install hook ให้ทุกคนอัตโนมัติตอน setup โปรเจกต์

### CI Pipeline (เกริ่นนำ Part 098)

Pre-commit hook ช่วยจับปัญหาได้เร็วบนเครื่องนักพัฒนา แต่ **hook เป็นสิ่งที่ข้ามได้เสมอ** (เช่น `git commit --no-verify`) ด่านที่บังคับได้จริงคือ **CI (Continuous Integration)** ตัวอย่าง workflow สำหรับ GitHub Actions (จะเรียนเจาะลึกเรื่อง CI/CD เต็มรูปแบบใน **Part 098**):

```yaml
# .github/workflows/lint.yml
name: Lint
on: [push, pull_request]

jobs:
  golangci-lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-go@v5
        with:
          go-version: '1.24'
      - name: golangci-lint
        uses: golangci/golangci-lint-action@v6
        with:
          version: v2.14.0
```

`golangci-lint-action` เป็น GitHub Action ทางการที่ทีม golangci-lint ดูแลเอง จัดการเรื่อง caching และการติดตั้งให้อัตโนมัติ ถ้า lint พบปัญหา job นี้จะ **fail** ทำให้ pull request แสดงสถานะ "checks failed" ชัดเจน และทีมส่วนใหญ่ตั้งค่าให้ **ห้าม merge PR ที่ check นี้ยังไม่ผ่าน** — นี่คือวิธีที่ทำให้มาตรฐาน code quality ถูกบังคับใช้จริงในทีม ไม่ใช่แค่ "ควรทำ" แต่เป็นสิ่งที่ **เป็นไปไม่ได้ที่จะข้าม**

---

## 9. ธรรมเนียมการ Code Review เฉพาะของ Go: Effective Go และ Go Code Review Comments

Linter จับปัญหาที่เป็นกฎตายตัวได้ (compile ผิด, ไม่เช็ค error, ตัวแปรไม่ได้ใช้) แต่มีอีกชั้นหนึ่งที่เป็น **ธรรมเนียมของชุมชน Go** ที่ไม่มี tool ไหนตรวจให้ได้ 100% ต้องอาศัยการ code review จากคนจริงๆ ทีม Go เองมีเอกสารสองฉบับที่เป็นรากฐานของธรรมเนียมนี้:

- **[Effective Go](https://go.dev/doc/effective_go)** — เอกสารทางการที่อธิบายสำนวนการเขียน Go ที่ถูกต้อง
- **[Go Code Review Comments](https://go.dev/wiki/CodeReviewComments)** — wiki ที่รวบรวมข้อคิดเห็นที่ reviewer ของทีม Go เองพิมพ์ซ้ำๆ บ่อยที่สุดตอน review โค้ดจริง

มาดูธรรมเนียมที่สำคัญที่สุดสามข้อที่ควรนำไปใช้ตอน review โค้ด Go ของทีม:

### ธรรมเนียมที่ 1: การตั้งชื่อ Receiver

```go
// ไม่ดี: ชื่อ receiver ยาวและไม่สอดคล้องกันระหว่าง method
func (this *UserService) GetUser(id int) (*User, error) { ... }
func (self *UserService) DeleteUser(id int) error { ... }
func (u *UserService) UpdateUser(id int) error { ... }

// ดี: ชื่อ receiver สั้น (1-2 ตัวอักษร มาจากตัวย่อของชื่อ type) และสอดคล้องกันทุก method
func (s *UserService) GetUser(id int) (*User, error) { ... }
func (s *UserService) DeleteUser(id int) error { ... }
func (s *UserService) UpdateUser(id int) error { ... }
```

Go Code Review Comments ระบุชัดเจนว่า **ห้ามใช้ `this` หรือ `self`** แบบภาษา OOP อื่น (Go ไม่ใช่ภาษาที่เน้น object-oriented แบบ class-based ตามปรัชญาที่กล่าวไว้ใน **Part 001**) และ **ชื่อ receiver ต้องสอดคล้องกันในทุก method ของ type เดียวกัน** — ไม่ใช่ตั้งชื่อไม่ซ้ำกันในแต่ละ method แบบตัวอย่างที่ไม่ดีข้างบน (`this`, `self`, `u` ปนกัน) ทบทวนจาก **Part 012**: ชื่อ receiver ที่นิยมมักเป็นตัวย่อ 1 ตัวอักษรจากชื่อ type (เช่น `s` สำหรับ `UserService`, `c` สำหรับ `Counter`)

### ธรรมเนียมที่ 2: ข้อความ Error String (ทบทวนจาก Part 016)

**Part 016** สอนกฎนี้ไปแล้วตอนเรียนเรื่อง custom error: ข้อความ error ต้องขึ้นต้นด้วยตัวพิมพ์เล็กและไม่มีเครื่องหมาย `.` ท้ายประโยค

```go
// ไม่ดี
return errors.New("Failed to connect to database.")

// ดี
return errors.New("failed to connect to database")
```

เหตุผล (ตามที่อธิบายไว้ใน Part 016): error message มักถูก wrap ต่อกันหลายชั้นด้วย `fmt.Errorf("...: %w", err)` การขึ้นต้นด้วยตัวพิมพ์ใหญ่หรือมี `.` ท้ายประโยคจะทำให้ error message ที่ประกอบกันหลายชั้นอ่านแปลกๆ (มีตัวพิมพ์ใหญ่และจุดกลางประโยค) — นี่เป็นตัวอย่างที่ดีว่าทำไม code review ต้องอาศัยคนจริงๆ ตรวจ เพราะแม้ `golangci-lint` จะมี linter อย่าง `stylecheck` (รวมอยู่ใน `staticcheck` ตามที่อธิบายในหัวข้อ 7) ที่ตรวจกฎนี้ได้บางส่วน แต่การตัดสินใจว่า error message สื่อความหมายชัดเจนพอหรือไม่ยังต้องใช้วิจารณญาณของคน

### ธรรมเนียมที่ 3: การตั้งชื่อ Package (ทบทวนจาก Part 001/002)

```go
// ไม่ดี: ชื่อ package เป็น camelCase, มีคำซ้ำกับสิ่งที่อยู่ข้างในเสมอ
package userManagement
package httpUtils

// ดี: ชื่อ package สั้น ตัวพิมพ์เล็กล้วน ไม่มี underscore
package user
package httputil
```

**Part 002** สอนไปแล้วว่าชื่อ package ควรเป็นตัวพิมพ์เล็กคำเดียว Go Code Review Comments ขยายความเพิ่มเติมว่า:

- **ห้ามตั้งชื่อ package ซ้ำกับสิ่งที่ import เข้าไปใช้เกือบทุกครั้ง** เช่น ฟังก์ชันใน package `user` ไม่ควรตั้งชื่อ `user.UserID`, `user.NewUser` (ซ้ำคำว่า user) ควรเป็น `user.ID`, `user.New` แทน เพราะตอนเรียกใช้จะเขียนเป็น `user.ID` อยู่แล้วซึ่งสื่อความหมายครบถ้วนโดยไม่ต้องซ้ำคำ
- **หลีกเลี่ยงชื่อ package ที่คลุมเครือเกินไป** เช่น `util`, `common`, `helpers` เพราะไม่บอกอะไรเลยว่าข้างในมีอะไร ยิ่งโปรเจกต์โตขึ้น package ประเภทนี้มักกลายเป็น "ที่ทิ้งทุกอย่างที่ไม่รู้จะเอาไปไว้ไหน" ทำให้ยากต่อการค้นหาและ maintain ในระยะยาว — ควรตั้งชื่อตามหน้าที่จริง (`validation`, `httpclient`, `timeutil`)

### เหตุผลที่ธรรมเนียมเหล่านี้สำคัญพอๆ กับ Linter

Linter ตรวจกฎที่เป็น **ขาว-ดำชัดเจน** ได้ (เช่น เช็ค error หรือไม่) แต่ธรรมเนียมข้างบนเป็นเรื่องของ **วิจารณญาณ** ที่ทีมต้องฝึกฝนร่วมกันผ่านการ code review จริง — โค้ดที่ผ่าน `golangci-lint` ทุกตัวไม่ได้แปลว่าเป็นโค้ด Go ที่ดีเสมอไป การผสมผสานทั้งสองอย่าง (**automated linting** + **human code review ที่ยึดธรรมเนียมของชุมชน**) คือแนวทางที่ทีม Go มืออาชีพใช้กันจริงในการรักษาคุณภาพโค้ดระยะยาว

---

## สรุปสิ่งที่ได้เรียนในบทนี้

- **`go vet` และ `gofmt` ยังไม่เพียงพอ** — `go vet` ถูกออกแบบให้ conservative มาก ไม่ครอบคลุมปัญหาอย่าง shadowing, ineffectual assignment, unused package-level identifier, หรือ error ที่ไม่ได้เช็ค
- **`golangci-lint`** คือ meta-linter มาตรฐานของวงการ Go ติดตั้งได้ด้วย `go install github.com/golangci/golangci-lint/v2/cmd/golangci-lint@latest` — ชุด default ("standard" preset) ครอบคลุม `govet`, `staticcheck`, `errcheck`, `unused`, `ineffassign` และจับปัญหาได้ทันทีโดยไม่ต้องตั้งค่าอะไรเพิ่ม
- **`shadow`** เป็น analyzer เสริมของ `govet` ที่ต้องเปิดเองผ่าน `.golangci.yml` (`settings.govet.enable: [shadow]`) เพราะไม่ได้อยู่ใน default
- **golangci-lint v2 รวม `gosimple` และ `stylecheck` เข้าไปในชื่อ `staticcheck`** เดียว — ต่างจาก v1 ที่แยกเป็นคนละ linter
- **`.golangci.yml`** ต้องมี `version: "2"` กำกับ ปรับแต่งชุด linter ด้วย `linters.enable`/`disable`, ตั้งค่าเจาะจงด้วย `linters.settings`, และยกเว้นบาง path ด้วย `linters.exclusions.rules`
- **Pre-commit hook** ช่วยจับปัญหาเร็วบนเครื่องนักพัฒนา แต่ **ข้ามได้เสมอ** — ด่านที่บังคับได้จริงคือ **CI** (จะเรียนเต็มรูปแบบใน Part 098) ที่ block การ merge PR เมื่อ lint ไม่ผ่าน
- **Effective Go และ Go Code Review Comments** เป็นรากฐานธรรมเนียมของชุมชน Go: ชื่อ receiver ต้องสั้นและสอดคล้องกัน (ห้าม `this`/`self`), error string ต้องขึ้นต้นตัวเล็กไม่มีจุดท้าย (ทบทวน Part 016), และชื่อ package ต้องสั้น ไม่ซ้ำคำกับสิ่งข้างใน และไม่คลุมเครือ

## แบบฝึกหัดท้ายบท

1. ติดตั้ง `golangci-lint` บนเครื่องของตัวเอง แล้วรัน `golangci-lint run ./...` กับโปรเจกต์ใดก็ได้จากแบบฝึกหัดของ Part ก่อนหน้าในหลักสูตรนี้ บันทึกว่าเจอปัญหาอะไรบ้าง
2. เขียนโค้ดที่มีปัญหาครบทั้ง 4 ประเภทจากหัวข้อ 4 ด้วยตัวเอง (ไม่ใช่ copy จากบทเรียน) แล้วรัน `golangci-lint run` ยืนยันว่าจับได้ครบ
3. สร้างไฟล์ `.golangci.yml` ของตัวเอง เปิด `revive` เพิ่มเติมจากชุด default แล้วสังเกตว่ามันเตือนอะไรเพิ่มขึ้นบ้าง (คำใบ้: มักเตือนเรื่อง comment ของ exported identifier ที่ต้องขึ้นต้นด้วยชื่อของ identifier นั้น)
4. เขียน `.git/hooks/pre-commit` ตามหัวข้อ 8 ลงในโปรเจกต์ทดสอบของตัวเอง แล้วลอง commit โค้ดที่มีปัญหา lint ยืนยันว่า commit ถูกบล็อกจริง
5. หาโค้ด Go open source ที่มีชื่อเสียง (เช่น จาก `golang/go` เอง หรือ `kubernetes/kubernetes`) มา 1 ไฟล์ แล้วไล่ตรวจสอบว่าธรรมเนียมทั้ง 3 ข้อในหัวข้อ 9 (receiver naming, error string, package naming) ถูกปฏิบัติตามหรือไม่
6. ค้นคว้าเพิ่มเติมเกี่ยวกับ `golangci-lint run --fix` ซึ่ง auto-fix ปัญหาบางประเภทได้อัตโนมัติ (เช่นจาก `gofmt`, `goimports`) ลองใช้กับโค้ดของตัวเองแล้วดูว่า linter ตัวไหนบ้างที่รองรับการ auto-fix (สังเกตจากป้าย `[auto-fix]` ตอนรัน `golangci-lint linters`)

---

**ต่อไป**: [Part 087 — Debugging ด้วย Delve](./087-debugging-with-delve.md)
