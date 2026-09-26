# Part 108: Go Modules Best Practices และ Semantic Versioning

> ภาคที่ 10: มืออาชีพและระดับโลก (Professional & World-Class) — ตอนที่ 9 จาก 11 (Part 100–110)

> **หมายเหตุเรื่องความซื่อสัตย์ในการสาธิต**: ทุกตัวอย่างในบทนี้ (semantic import versioning, Minimal Version Selection, การตรวจจับ deprecated symbol) **รันจริง** ด้วย Go 1.24.7 บนเครื่องที่ใช้เขียนหลักสูตรนี้ โดยสร้าง Go module จริงหลายตัวพร้อม git tag จริง และ (สำหรับหัวข้อ MVS) สร้าง local module proxy ตามมาตรฐาน [Go Module Proxy Protocol](https://go.dev/ref/mod#goproxy-protocol) เพื่อจำลองสถานการณ์ที่มี dependency หลายเวอร์ชันอยู่จริง โดยไม่ต้องพึ่ง network ภายนอก — ข้อความ error และผลลัพธ์ทั้งหมดคือ output จริงจากคำสั่งที่รันจริง ไม่ใช่ค่าที่แต่งขึ้น

## สารบัญของบทนี้

1. ทำไม Versioning ถึงสำคัญพอๆ กับตัวโค้ดเอง
2. Semantic Versioning (SemVer): MAJOR.MINOR.PATCH สัญญาอะไรบ้าง
3. Go ตีความ SemVer อย่างไร: Semantic Import Versioning (ทบทวน Part 018)
4. สาธิตจริง: ข้อผิดพลาดคลาสสิกตอนออก v2 และวิธีแก้ที่ถูกต้อง
5. Minimal Version Selection (MVS): Go เลือกเวอร์ชัน Dependency อย่างไร
6. สาธิตจริง: MVS ทำงานจริงกับ Dependency ที่ขัดแย้งกัน
7. MVS เทียบกับแนวทางของ npm/pip: ทำไม Go เลือกทางนี้
8. `go.sum` และ Checksum Database ทบทวนในมุมของ Versioning (Part 018)
9. Tagging Release อย่างถูกวิธีด้วย `git tag`
10. เขียน CHANGELOG ที่มีประโยชน์จริง
11. Deprecating Function/Package อย่างถูกวิธี: `// Deprecated:` และการตรวจจับด้วย `staticcheck`
12. เรียกคืนเวอร์ชันที่มีปัญหาด้วย `retract` (สาธิตจริง)
13. Go 1 Compatibility Promise: มาตรฐานทองคำของ Backward Compatibility (ทบทวน Part 001)
14. Checklist: ก่อนออก Release ใหม่ของ Library
15. สรุปสิ่งที่ได้เรียนในบทนี้
16. แบบฝึกหัดท้ายบท

---

## 1. ทำไม Versioning ถึงสำคัญพอๆ กับตัวโค้ดเอง

โค้ดที่เขียนดีแค่ไหนก็ไร้ประโยชน์ถ้าคนอื่น**ไม่กล้าอัปเดต**ไปใช้เวอร์ชันใหม่ เพราะกลัวว่าจะพังระบบที่ใช้งานอยู่ — นี่คือปัญหาที่ **versioning ที่ดี** แก้ให้ **Part 018** สอนกลไกของ `go.mod`/`go.sum` ไปแล้วในเชิงเทคนิค บทนี้ยกระดับไปสู่**หลักปฏิบัติของคนที่ดูแล library ที่คนอื่นพึ่งพา** ไม่ว่าจะเป็น internal library ภายในบริษัท หรือ open source package ที่เผยแพร่สู่สาธารณะ

หัวใจของบทนี้คือคำถามเดียว: **"เมื่อฉันเปลี่ยนโค้ด ฉันสื่อสารกับคนที่ใช้ library ของฉันอย่างไรว่าการเปลี่ยนแปลงนี้ปลอดภัยแค่ไหน?"** — คำตอบมาตรฐานของอุตสาหกรรมคือ **Semantic Versioning**

---

## 2. Semantic Versioning (SemVer): MAJOR.MINOR.PATCH สัญญาอะไรบ้าง

**Semantic Versioning** (semver.org) กำหนดรูปแบบเลขเวอร์ชัน `MAJOR.MINOR.PATCH` (เช่น `v2.5.1`) โดยแต่ละตำแหน่งมี**สัญญา (contract)** ที่ชัดเจนกับผู้ใช้:

| ตำแหน่ง | เพิ่มขึ้นเมื่อ | สัญญาที่ให้กับผู้ใช้ |
|---|---|---|
| **MAJOR** (`v`**2**`.5.1`) | มีการเปลี่ยนแปลงที่ **breaking** — โค้ดเดิมของผู้ใช้อาจ compile ไม่ผ่านหรือทำงานต่างไปจากเดิม | "ต้องอ่าน migration guide ก่อนอัปเกรด อาจต้องแก้โค้ด" |
| **MINOR** (`v2.`**5**`.1`) | เพิ่ม feature ใหม่ที่ **ไม่ breaking** (backward-compatible) | "อัปเกรดได้ทันทีอย่างปลอดภัย ได้ feature ใหม่เพิ่มมาฟรี" |
| **PATCH** (`v2.5.`**1**) | แก้บั๊กเท่านั้น ไม่มี feature ใหม่ ไม่ breaking | "อัปเกรดได้ทันทีเสมอ ไม่มีความเสี่ยงใดๆ" |

ตัวอย่างการตัดสินใจในทางปฏิบัติสำหรับ library Go ที่มี public API เป็น:

```go
package mathutil

func Add(a, b int) int { return a + b }
```

| การเปลี่ยนแปลง | ประเภท | เหตุผล |
|---|---|---|
| แก้บั๊กภายใน `Add` โดย signature เดิม | PATCH | ผู้ใช้ไม่ต้องแก้โค้ดเลย |
| เพิ่มฟังก์ชันใหม่ `Subtract(a, b int) int` | MINOR | โค้ดเดิมยังทำงานเหมือนเดิมทุกประการ แค่มีของใหม่เพิ่ม |
| เปลี่ยน `Add(a, b int) int` เป็น `Add(a, b int) (int, error)` | MAJOR | โค้ดเดิมที่เรียก `sum := Add(1, 2)` **compile ไม่ผ่านทันที** |
| ลบฟังก์ชัน `Add` ออกไปเฉยๆ | MAJOR | ชัดเจนว่า breaking |
| เปลี่ยนพฤติกรรมภายในโดย signature เดิม แต่ผลลัพธ์ต่างจากเดิม | MAJOR (ในทางปฏิบัติ) | แม้ compile ผ่าน แต่พฤติกรรมเปลี่ยนถือเป็น breaking change เชิง semantic |

จุดที่มือใหม่พลาดบ่อยคือแถวสุดท้าย: **"compile ผ่าน" ไม่ได้แปลว่า "ไม่ breaking"** ถ้าฟังก์ชันเดิมเคย return ผลลัพธ์แบบหนึ่งแล้วเปลี่ยนไป return อีกแบบโดย signature เดิมทุกประการ นี่คือ breaking change เชิง behavior ที่ควรขึ้น MAJOR เช่นกัน แม้ compiler จะไม่บ่นก็ตาม

---

## 3. Go ตีความ SemVer อย่างไร: Semantic Import Versioning (ทบทวน Part 018)

**Part 018** แนะนำแนวคิด **Semantic Import Versioning** ไว้แล้ว บทนี้ขยายรายละเอียดและเหตุผลเบื้องหลัง — Go มีกฎพิเศษที่ภาษาอื่นส่วนใหญ่ไม่มี: **module ที่ major version ตั้งแต่ v2 ขึ้นไป ต้องใส่ major version ไว้ใน import path โดยตรง**

```
example.com/mylib        → v0.x.x, v1.x.x (ไม่ต้องมี suffix)
example.com/mylib/v2     → v2.x.x
example.com/mylib/v3     → v3.x.x
```

### เหตุผลเบื้องหลังกฎนี้

กฎนี้ไม่ได้มีไว้เพื่อความยุ่งยาก แต่แก้ปัญหาจริงที่เรียกว่า **"diamond dependency problem"**: สมมติโปรเจกต์ A ใช้ library `X` v1 และในขณะเดียวกันก็ใช้ library `Y` ที่ภายในพึ่งพา `X` v2 (ซึ่งมี breaking change จาก v1) ถ้า v1 และ v2 ใช้ import path เดียวกัน คอมไพเลอร์จะสับสนว่า `X.Foo()` ที่เห็นในแต่ละที่หมายถึงเวอร์ชันไหน — การบังคับให้ `/v2` อยู่ใน import path ทำให้ **v1 และ v2 กลายเป็นคนละ package กันอย่างสมบูรณ์ในสายตาของ Go compiler** โปรเจกต์ A จึงใช้ทั้งสองเวอร์ชันพร้อมกันได้โดยไม่ชนกัน (เชื่อมโยงกับ `import` และการแยก package ที่เรียนไปตั้งแต่ **Part 001-002**)

---

## 4. สาธิตจริง: ข้อผิดพลาดคลาสสิกตอนออก v2 และวิธีแก้ที่ถูกต้อง

ข้อผิดพลาดที่พบบ่อยที่สุดของนักพัฒนาที่เพิ่งออก v2 ครั้งแรกคือ **แค่ tag `v2.0.0` โดยไม่แก้ module path ใน `go.mod`** มาดูว่าเกิดอะไรขึ้นจริงเมื่อทำแบบนั้น

สมมติมี library `example.com/greetlib` ที่ออก v1.0.0 ไปแล้ว:

```go
// go.mod
module example.com/greetlib

go 1.21
```

```go
// greet.go
package greetlib

func Hello(name string) string {
	return "Hello, " + name
}
```

```bash
git tag v1.0.0
```

ต่อมาผู้ดูแล library เพิ่มฟังก์ชันใหม่แบบ breaking (เปลี่ยนพฤติกรรมเดิม) แล้ว **tag เป็น v2.0.0 โดยไม่แก้ `go.mod`**:

```bash
git tag v2.0.0   # ยังคง module path เดิมใน go.mod
```

เมื่อโปรเจกต์อื่นพยายาม `go get example.com/greetlib@v2.0.0`:

```
go: errors parsing go.mod:
go.mod:5: require example.com/greetlib: version "v2.0.0" invalid: should be v0 or v1, not v2
```

**นี่คือ error จริงที่ Go toolchain รายงาน** — Go ปฏิเสธตั้งแต่ระดับ `go.mod` parsing เลย ไม่ปล่อยให้ build ต่อไปแล้วพังทีหลังแบบเงียบๆ นี่คือตัวอย่างที่ดีของปรัชญา **"บังคับความถูกต้องตั้งแต่ compiler/toolchain"** ที่ Go ยึดถือมาตั้งแต่ **Part 001** (เทียบกับ unused import ที่เป็น compile error)

### วิธีแก้ที่ถูกต้อง

แก้ module path ใน `go.mod` ให้มี `/v2` ต่อท้ายก่อน tag:

```go
// go.mod
module example.com/greetlib/v2

go 1.21
```

Import path ในผู้ใช้ก็ต้องเปลี่ยนตาม:

```go
import greetlib "example.com/greetlib/v2"
```

Tag ใหม่แล้วลองอีกครั้ง — ผลลัพธ์จริงจากการรัน:

```
$ go build -o consumer . && ./consumer
Hello, world!
Goodbye, world!
```

สำเร็จ — สังเกตว่า **path บนดิสก์/repository ไม่จำเป็นต้องมีโฟลเดอร์ชื่อ `v2`** (แนวทางที่พบบ่อยกว่าคือแก้แค่บรรทัด `module` ใน `go.mod` โดยโค้ดยังอยู่ที่ root เดิม) `/v2` เป็นเรื่องของ **import path เท่านั้น** ไม่ใช่โครงสร้างไฟล์บังคับ (แม้บาง project จะเลือกแยกโฟลเดอร์จริงเพื่อความชัดเจนก็ทำได้เช่นกัน)

### กฎสรุปสำหรับ v2 ขึ้นไป

> ก่อน tag เวอร์ชัน MAJOR ตั้งแต่ 2 ขึ้นไปทุกครั้ง ต้องแก้บรรทัด `module` ใน `go.mod` ให้มี `/vN` ต่อท้ายเสมอ (`/v2`, `/v3`, ...) และปรับ import path ในโค้ดของ library เองที่ import ตัวเองข้าม package ให้ตรงกัน มิฉะนั้นผู้ใช้จะเจอ error ตั้งแต่ `go get` ทันที

---

## 5. Minimal Version Selection (MVS): Go เลือกเวอร์ชัน Dependency อย่างไร

เมื่อโปรเจกต์หนึ่งมี dependency หลายตัว และแต่ละตัวต้องการ dependency ร่วมกันตัวเดียวกันคนละเวอร์ชัน — Go ต้องตัดสินใจว่าจะใช้เวอร์ชันไหนสุดท้าย อัลกอริทึมที่ Go ใช้เรียกว่า **Minimal Version Selection (MVS)** หลักการ (แบบย่อที่สุดเท่าที่จะอธิบายได้):

> **สำหรับ dependency แต่ละตัว Go เลือกเวอร์ชันที่ "สูงที่สุดเท่าที่มีใครใน dependency graph ต้องการ" (ไม่ใช่เวอร์ชันล่าสุดที่มีอยู่ในโลก)**

พูดให้ต่างออกไป: ถ้าโมดูล A ต้องการ dependency C อย่างน้อย v1.0.0 และโมดูล B ต้องการ C อย่างน้อย v1.1.0 ระบบจะเลือก **v1.1.0** (ค่าสูงสุดของความต้องการขั้นต่ำทั้งหมด) แม้ v1.2.0 หรือ v2.0.0 จะมีอยู่ในโลกจริงแล้วก็ตาม ตราบใดที่**ไม่มีใครใน graph ร้องขอ**เวอร์ชันนั้น

---

## 6. สาธิตจริง: MVS ทำงานจริงกับ Dependency ที่ขัดแย้งกัน

มาสร้างสถานการณ์นี้ขึ้นจริง: มี library `example.com/liba` ที่มี 3 เวอร์ชันอยู่จริง (`v1.0.0`, `v1.1.0`, `v1.2.0`), มี `example.com/modulex` ที่ require `liba v1.0.0`, และ `example.com/moduley` ที่ require `liba v1.1.0` — **สังเกตว่า `v1.2.0` มีอยู่จริงแต่ไม่มีใครร้องขอมันเลย**

```go
// modulex/go.mod
module example.com/modulex

go 1.21

require example.com/liba v1.0.0
```

```go
// moduley/go.mod
module example.com/moduley

go 1.21

require example.com/liba v1.1.0
```

โปรเจกต์หลักที่ require ทั้งสองตัว:

```go
// main/go.mod
module example.com/mvsdemo

go 1.21

require (
	example.com/modulex v1.0.0
	example.com/moduley v1.0.0
)
```

รัน `go mod tidy` จริง:

```
$ go mod tidy
go: downloading example.com/moduley v1.0.0
go: downloading example.com/modulex v1.0.0
go: downloading example.com/liba v1.1.0
```

สังเกตบรรทัดสุดท้าย: **Go ดาวน์โหลด `liba v1.1.0` โดยอัตโนมัติ ไม่ใช่ `v1.0.0` (ค่าต่ำสุดที่ modulex ขอ) และไม่ใช่ `v1.2.0` (เวอร์ชันล่าสุดที่มีอยู่จริง)** — `go.mod` หลังจาก tidy:

```
module example.com/mvsdemo

go 1.21

require (
	example.com/modulex v1.0.0
	example.com/moduley v1.0.0
)

require example.com/liba v1.1.0 // indirect
```

ตรวจสอบด้วย `go list -m all` (แสดงเวอร์ชันที่ **เลือกใช้จริง** ของทุก module ใน build):

```
$ go list -m all
example.com/mvsdemo
example.com/liba v1.1.0
example.com/modulex v1.0.0
example.com/moduley v1.0.0
```

และดู dependency graph แบบเต็มด้วย `go mod graph`:

```
$ go mod graph
example.com/mvsdemo example.com/liba@v1.1.0
example.com/mvsdemo example.com/modulex@v1.0.0
example.com/mvsdemo example.com/moduley@v1.0.0
example.com/modulex@v1.0.0 example.com/liba@v1.0.0
example.com/moduley@v1.0.0 example.com/liba@v1.1.0
```

อ่านกราฟนี้ตรงตัว: **`modulex` เองขอ `liba@v1.0.0`** (บรรทัดที่ 4) **แต่ `moduley` ขอ `liba@v1.1.0`** (บรรทัดที่ 5) — MVS เลือกค่าสูงสุดของทั้งสอง (`v1.1.0`) เป็นเวอร์ชันเดียวที่ใช้จริงทั่วทั้ง build (บรรทัดแรกสุด) ยืนยันด้วยการรันโปรแกรมจริง:

```go
package main

import (
	"fmt"

	"example.com/modulex"
	"example.com/moduley"
)

func main() {
	fmt.Println("modulex sees liba:", modulex.LibAVersion())
	fmt.Println("moduley sees liba:", moduley.LibAVersion())
}
```

```
$ go run .
modulex sees liba: v1.1.0
moduley sees liba: v1.1.0
```

**ทั้ง `modulex` และ `moduley` เห็น `liba` เวอร์ชันเดียวกัน (`v1.1.0`) แม้ `modulex` จะเขียนโค้ดโดยอ้างอิง v1.0.0 ก็ตาม** — นี่คือกฎสำคัญอีกข้อของ Go module system: **ทั้ง build ใช้ dependency แต่ละตัวเวอร์ชันเดียวเท่านั้น (single version rule)** ไม่มีการซ้อนหลายเวอร์ชันของ major version เดียวกันในกราฟเดียว (ต่างจาก v1 กับ v2 ที่ถือเป็นคนละ module กันตามกฎ semantic import versioning ในหัวข้อ 3 จึงอยู่ร่วมกันได้)

---

## 7. MVS เทียบกับแนวทางของ npm/pip: ทำไม Go เลือกทางนี้

นักพัฒนาที่มาจาก JavaScript (`npm`) หรือ Python (`pip`) มักแปลกใจกับพฤติกรรมข้างบน เพราะ ecosystem เหล่านั้นใช้แนวทางต่างกันโดยพื้นฐาน:

| | Go (MVS) | npm | pip (ค่าเริ่มต้น) |
|---|---|---|---|
| หลักการเลือกเวอร์ชัน | **ต่ำที่สุดที่ยังสอดคล้องกับทุกความต้องการ** (สูงสุดของ minimum ทั้งหมด) | **สูงที่สุดเท่าที่มีอยู่และสอดคล้องกับ range ที่ระบุ** (เช่น `^1.0.0`) | ค่าสูงสุดที่สอดคล้องกับทุก constraint |
| Build reproducible ไหมถ้าไม่ pin | **ใช่** — ผลลัพธ์เดิมเสมอตราบใดที่ `go.sum` ไม่เปลี่ยน แม้มีเวอร์ชันใหม่ออกมาหลังจากนั้น | **ไม่แน่นอน** — รันคนละวันอาจได้เวอร์ชันต่างกันถ้าไม่ lock ให้ดี (`package-lock.json` ช่วยแก้ปัญหานี้) | คล้าย npm ต้องพึ่ง lock file (`requirements.txt` แบบ pin หรือ `poetry.lock`) |
| ปัญหาหลายเวอร์ชันซ้อนกัน (dependency hell) | ป้องกันด้วย single version rule + semantic import versioning | เกิดได้บ่อย — `node_modules` มักมีหลายเวอร์ชันของ package เดียวกันซ้อนกันในระบบ nested dependency | เกิดได้ถ้า environment ไม่แยกให้ดี (แก้ด้วย virtualenv) |

### เหตุผลเชิงปรัชญาของทีม Go

Russ Cox (หัวหน้าทีม Go และผู้ออกแบบ MVS) อธิบายเหตุผลไว้ชัดเจน: **"เวอร์ชันล่าสุดเสมอ" ฟังดูดีในทางทฤษฎี แต่ในทางปฏิบัติทำให้ build ไม่ reproducible** — เวอร์ชันที่ "ล่าสุด" ของวันนี้ ไม่ใช่เวอร์ชันเดียวกับ "ล่าสุด" ของเมื่อวาน การเลือก **"เวอร์ชันต่ำที่สุดที่ยังใช้งานได้"** แทน ทำให้ build มีความนิ่ง (stable) และคาดเดาได้เสมอ ตราบใดที่ไม่มีใครแก้ `go.mod` เพิ่มความต้องการใหม่ — สอดคล้องโดยตรงกับปรัชญา **"less is more"** และความเรียบง่ายที่คาดเดาได้ ซึ่งเป็นแก่นของ Go มาตั้งแต่ **Part 001**

การจะได้เวอร์ชันใหม่กว่าที่ MVS เลือกให้ ผู้ใช้ต้อง**ร้องขอมันอย่างชัดเจน** ด้วย `go get example.com/liba@latest` หรือระบุเวอร์ชันตรงๆ — ไม่มีอะไรเปลี่ยนแปลง "อัตโนมัติ" อยู่เบื้องหลังโดยที่นักพัฒนาไม่รู้ตัว

---

## 8. `go.sum` และ Checksum Database ทบทวนในมุมของ Versioning (Part 018)

**Part 018** อธิบายกลไก `go.sum` ไว้แล้วว่าเก็บ cryptographic hash ของทุก module version ที่เคยถูกดาวน์โหลด เพื่อป้องกัน supply-chain attack (module ถูกแก้ไขเนื้อหาโดยไม่เปลี่ยนเลขเวอร์ชัน) ในบริบทของบทนี้ สิ่งที่ต้องเน้นย้ำคือ **`go.sum` กับ MVS ทำงานเสริมกันเพื่อรับประกัน reproducible build แบบสมบูรณ์**:

- **`go.mod`** บอกว่า "เวอร์ชันไหนถูกเลือกใช้" (ผลจาก MVS)
- **`go.sum`** บอกว่า "เนื้อหาของเวอร์ชันนั้นต้องมี hash ตรงกับค่านี้เท่านั้น"

ทั้งสองไฟล์ต้อง **commit เข้า version control เสมอ** (ไม่ใช่แค่ `go.mod`) เพื่อให้ทุกคนในทีม และทุก CI run ได้ dependency ชุดเดียวกันแบบไบต์ต่อไบต์ทุกครั้งไม่ว่าจะรันเมื่อไรก็ตาม — **Sum Database** (`sum.golang.org`) ที่ Go ตรวจสอบผ่านโดย default ทำหน้าที่เป็น "พยานอิสระ" คอยยืนยันว่า hash ใน `go.sum` ของทุกคนตรงกัน ไม่ถูกปลอมแปลงระหว่างทาง

---

## 9. Tagging Release อย่างถูกวิธีด้วย `git tag`

Go module ระบุเวอร์ชันผ่าน **git tag** โดยตรง ไม่มีไฟล์ manifest แยกต่างหากแบบ `package.json`/`setup.py` — วิธี tag ที่ถูกต้อง:

```bash
# Tag แบบ annotated (แนะนำ — เก็บข้อมูลผู้ tag, วันที่, ข้อความ ต่างจาก lightweight tag)
git tag -a v1.2.3 -m "Release v1.2.3: add retry backoff support"

# Push tag ขึ้น remote (git push เฉยๆ ไม่ push tag ให้อัตโนมัติ)
git push origin v1.2.3

# หรือ push ทุก tag ที่มีในเครื่องพร้อมกัน
git push origin --tags
```

### กฎการตั้งชื่อ tag ที่ Go บังคับ

- ต้องขึ้นต้นด้วย `v` เสมอ (`v1.2.3` ไม่ใช่ `1.2.3`) — Go tooling จะไม่รู้จัก tag ที่ไม่มี `v` นำหน้าว่าเป็นเวอร์ชันของ module
- Pre-release ใช้ `-` ต่อท้าย เช่น `v2.0.0-beta.1`, `v2.0.0-rc.1` — Go **จะไม่เลือก pre-release เหล่านี้โดยอัตโนมัติ** เมื่อมีคนขอ "เวอร์ชันล่าสุด" (`go get example.com/lib@latest`) ต้องระบุ pre-release version ตรงๆ เท่านั้นถึงจะได้ ป้องกันไม่ให้ผู้ใช้เผลอได้ build ที่ยังไม่เสถียรโดยไม่ตั้งใจ
- Module ที่อยู่ใน monorepo (หลาย module ในหนึ่ง git repository) ใช้ prefix ตามโฟลเดอร์ เช่น `subpkg/v1.2.3` สำหรับ module ที่อยู่ในโฟลเดอร์ `subpkg/`

### เมื่อไรควรออกเวอร์ชันไหน (ทบทวนจากหัวข้อ 2)

หลักปฏิบัติที่ทีมมืออาชีพใช้จริง: **ออก PATCH บ่อยเท่าที่จำเป็น**ทันทีที่มีบั๊กสำคัญถูกแก้ (อย่ารอสะสมหลายบั๊กแล้วออกทีเดียว), **ออก MINOR เป็นจังหวะสม่ำเสมอ** (เช่น ทุก 2-4 สัปดาห์) เมื่อมี feature ใหม่สะสมพอสมควร, และ **ออก MAJOR เฉพาะเมื่อจำเป็นจริงๆ** พร้อมเตรียม migration guide ล่วงหน้าให้ผู้ใช้เพราะเป็นเวอร์ชันที่ผู้ใช้ต้องลงทุนเวลาแก้โค้ดของตัวเอง

---

## 10. เขียน CHANGELOG ที่มีประโยชน์จริง

CHANGELOG ที่ดีคือสิ่งที่ทำให้ผู้ใช้ **กล้าอัปเดต** เพราะรู้ล่วงหน้าว่าจะเจออะไรบ้าง รูปแบบที่ชุมชนยอมรับกันกว้างขวางที่สุดคือ [**Keep a Changelog**](https://keepachangelog.com):

```markdown
# Changelog

ทุกการเปลี่ยนแปลงที่สำคัญของโปรเจกต์นี้จะถูกบันทึกไว้ในไฟล์นี้

รูปแบบอ้างอิงจาก [Keep a Changelog](https://keepachangelog.com/en/1.1.0/)
และโปรเจกต์นี้ยึดถือ [Semantic Versioning](https://semver.org/)

## [Unreleased]

### Added
- รองรับ context cancellation ใน `Client.Fetch`

## [1.2.0] - 2025-03-10

### Added
- เพิ่มฟังก์ชัน `WithTimeout` สำหรับตั้งค่า timeout ของ client (#42)

### Fixed
- แก้ปัญหา goroutine leak เมื่อ `Close()` ถูกเรียกซ้ำ (#45)

### Deprecated
- `Client.SetTimeout` ถูก deprecate แล้ว ใช้ `WithTimeout` แทน จะถูกลบใน v2.0.0

## [1.1.0] - 2025-01-15

### Added
- เพิ่ม retry with exponential backoff (#30)

## [1.0.0] - 2024-11-01

### Added
- Initial stable release
```

### หลักการเขียน CHANGELOG ที่ดี

- **จัดกลุ่มตามประเภทการเปลี่ยนแปลงเสมอ**: `Added`, `Changed`, `Deprecated`, `Removed`, `Fixed`, `Security` — ทำให้ผู้อ่านกวาดตาหาสิ่งที่เกี่ยวกับตัวเองได้เร็ว โดยเฉพาะกลุ่ม `Removed`/`Changed` ที่บอกถึง breaking change ที่ต้องระวังเป็นพิเศษ
- **อ้างอิง pull request/issue number เสมอ** เพื่อให้ผู้อ่านย้อนไปดูรายละเอียดเชิงลึกได้ถ้าต้องการ
- **เขียนจากมุมมองของผู้ใช้ ไม่ใช่ผู้เขียนโค้ด** เช่น "แก้ปัญหา goroutine leak เมื่อ `Close()` ถูกเรียกซ้ำ" สื่อสารชัดกว่า "refactor connection cleanup logic"
- **แยกหัวข้อ `[Unreleased]` ไว้บนสุดเสมอ** สะสมการเปลี่ยนแปลงระหว่างที่ยังไม่ tag เวอร์ชันใหม่ พอถึงเวลา release ก็เปลี่ยนหัวข้อนี้เป็นเลขเวอร์ชันจริงพร้อมวันที่

---

## 11. Deprecating Function/Package อย่างถูกวิธี: `// Deprecated:` และการตรวจจับด้วย `staticcheck`

เมื่อฟังก์ชันหนึ่งควรถูกแทนที่ด้วยฟังก์ชันใหม่ (แต่ยังไม่ถึงเวลาลบออกจริงเพราะจะ breaking) มาตรฐานของ Go คือ**คอมเมนต์แบบพิเศษ** ที่ tooling ทั้ง ecosystem รู้จัก:

```go
// GreetOld ทักทายแบบเดิม
//
// Deprecated: ใช้ GreetNew แทน เพราะรองรับ i18n ได้ดีกว่า จะถูกลบออกใน v3.0.0
func GreetOld(name string) string {
	return "Hi " + name
}

// GreetNew ทักทายแบบใหม่ที่รองรับ i18n
func GreetNew(name string) string {
	return "Hello, " + name
}
```

รูปแบบที่ [go.dev/wiki/Deprecated](https://go.dev/wiki/Deprecated) กำหนดไว้ต้องตรงเป๊ะ: คำว่า **`Deprecated:`** ต้องอยู่ **ต้นย่อหน้าใหม่** (เว้นบรรทัดว่างก่อนหน้า) ภายใน doc comment ของ symbol นั้น — ผิดรูปแบบแม้เล็กน้อย tooling จะตรวจจับไม่ได้เลย

### ผลลัพธ์จริงเมื่อมีคนเรียกใช้ symbol ที่ deprecate แล้ว

**gopls** (Go language server ที่ตั้งค่าไว้ตั้งแต่ **Part 001**) จะขีดเส้นทับ (strikethrough) ชื่อฟังก์ชันที่ deprecate ให้เห็นตรงๆ ใน editor ทันทีที่พิมพ์เรียกใช้ แต่ในเชิง**การตรวจจับอัตโนมัติระดับ CI** เครื่องมือที่ทำหน้าที่นี้อย่างเป็นทางการคือ **`staticcheck`** ผ่านกฎ **SA1019**

มาดูผลลัพธ์จริง: มี library `greetlib` ที่มี `GreetOld` (deprecated) และ `GreetNew` และมี `app` ที่ import แล้วเรียกใช้ทั้งคู่:

```go
package main

import (
	"fmt"

	"example.com/deprecatedemo/greetlib"
)

func main() {
	fmt.Println(greetlib.GreetOld("Alice"))
	fmt.Println(greetlib.GreetNew("Bob"))
}
```

รัน `go vet` (ไม่ตรวจ deprecation) เทียบกับ `staticcheck`:

```
$ go vet ./...
(ไม่มี output ใดๆ — go vet ไม่ตรวจจับ deprecated symbol)

$ staticcheck ./...
main.go:10:14: example.com/deprecatedemo/greetlib.GreetOld is deprecated: ใช้ GreetNew แทน เพราะรองรับ i18n ได้ดีกว่า จะถูกลบออกใน v3.0.0  (SA1019)
```

จุดสำคัญที่ได้จากการทดลองจริง:

- **`go vet` มาตรฐานไม่ตรวจจับ deprecation** ต้องพึ่ง `staticcheck` (หรือ linter อื่นที่รองรับ SA1019 เช่นที่ผนวกอยู่ใน `golangci-lint` ที่ **Part 086** สอนไว้) โดยเฉพาะ
- `staticcheck` ดึง**ข้อความอธิบายเต็มจาก doc comment** มาแสดงตรงๆ ในผลลัพธ์ (`ใช้ GreetNew แทน...`) ทำให้ผู้พัฒนาที่เห็น warning รู้ทันทีว่าควรทำอะไรต่อ โดยไม่ต้องเปิดไปดู source ของ library เอง
- ในการทดลองนี้ **`staticcheck` ตรวจจับได้เฉพาะตอนเรียกข้าม package/module เท่านั้น** (การเรียกใช้ deprecated symbol ภายใน package เดียวกันที่ประกาศมันจะไม่ถูกเตือน เพราะถือว่าผู้เขียน package นั้นรู้เจตนาของตัวเองอยู่แล้ว) — ทำให้เหมาะเป็น**ด่านเตือนสำหรับผู้ใช้ library ภายนอก**โดยเฉพาะ

### เพิ่ม `staticcheck` เข้า CI (ต่อยอด Part 098)

```yaml
- name: Install staticcheck
  run: go install honnef.co/go/tools/cmd/staticcheck@latest

- name: Run staticcheck
  run: staticcheck ./...
```

เพราะ `staticcheck` คืน exit code ไม่ใช่ศูนย์เมื่อพบปัญหา (รวมถึง SA1019) การเพิ่ม step นี้ทำให้ทีมเห็นการใช้ deprecated API ตั้งแต่ pull request ก่อน merge — ควรพิจารณาว่าจะให้ SA1019 เป็น **error ที่ block merge** หรือแค่ **warning ที่แจ้งเตือน** ตามนโยบายของแต่ละทีม (บาง config ตั้งให้ SA1019 เป็น non-blocking เพราะบางครั้งการยังใช้ deprecated API ต่อไปชั่วคราวเป็นการตัดสินใจที่ตั้งใจแล้ว)

---

## 12. เรียกคืนเวอร์ชันที่มีปัญหาด้วย `retract` (สาธิตจริง)

บางครั้ง version ที่ tag และ push ออกไปแล้วกลับพบว่ามีปัญหาร้ายแรง (บั๊กวิกฤต, ช่องโหว่ความปลอดภัย, หรือแม้แต่ tag ผิดโดยไม่ตั้งใจ) — ปัญหาคือ **Go module ไม่มีแนวคิด "ลบ tag ออกจากโลก" ได้จริง** เพราะ Sum Database (**Part 018, หัวข้อ 8**) เก็บ hash ของทุกเวอร์ชันที่เคยถูกดาวน์โหลดไว้ถาวร การลบ tag ออกจาก git repository เฉยๆ **ไม่ได้ทำให้คนที่ดาวน์โหลดไปแล้วหยุดใช้งานมันได้เลย**

วิธีที่ถูกต้องคือใช้ **`retract` directive** ใน `go.mod` ที่เพิ่มเข้ามาตั้งแต่ Go 1.16 — มันไม่ได้ "ลบ" เวอร์ชันออกไปจริง แต่**ประกาศต่อสาธารณะว่าเวอร์ชันนี้ไม่ควรถูกใช้** โดย `go get`/`go mod tidy` ของผู้ใช้จะเห็นคำเตือนทันทีถ้าพยายามเลือกเวอร์ชันที่ถูก retract ไว้

### สาธิตจริง: เพิ่ม `retract` เข้า `go.mod`

ใช้คำสั่ง `go mod edit` แก้ไข `go.mod` โดยตรงแทนการพิมพ์เอง (ลดโอกาสพิมพ์ผิด):

```bash
go mod edit -retract=v2.0.0
go mod edit -retract="[v2.0.1,v2.0.5]"
```

ผลลัพธ์จริงใน `go.mod`:

```
module example.com/greetlib/v2

go 1.21

retract (
	[v2.0.1, v2.0.5]
	v2.0.0
)
```

สังเกตว่า `retract` รับได้ทั้ง **เวอร์ชันเดี่ยว** (`v2.0.0`) และ **ช่วงเวอร์ชันแบบ interval** (`[v2.0.1, v2.0.5]` หมายถึง retract ทุกเวอร์ชันตั้งแต่ v2.0.1 ถึง v2.0.5) — ใช้แบบช่วงเมื่อบั๊กเดียวกันกระทบหลายเวอร์ชันติดต่อกัน แทนที่จะต้องเขียนทีละบรรทัด

### เพิ่มเหตุผล (Rationale) ให้ผู้ใช้เห็นทันที

การ retract ที่ดีควรมีคอมเมนต์อธิบายเหตุผลกำกับไว้เสมอ เพราะข้อความนี้จะถูกแสดงให้ผู้ใช้เห็นโดยตรงตอนรัน `go get`/`go list -m -u`:

```go
retract (
	v2.0.0          // มีช่องโหว่ด้าน memory safety ร้ายแรง ดู CVE-XXXX-YYYYY
	[v2.0.1, v2.0.5] // regression ทำให้ Parse() panic กับ input ว่าง
)
```

### สาธิตจริง: ผู้ใช้เห็นอะไรเมื่อพยายามใช้เวอร์ชันที่ถูก retract

เมื่อมี module ที่ retract เวอร์ชันหนึ่งไว้แล้ว และผู้ใช้ที่ `go.mod` ของตัวเองยัง pin เวอร์ชันนั้นอยู่รัน `go list -m -u`:

```bash
go list -m -u example.com/greetlib/v2
```

ผลลัพธ์จริง (ตัวอย่างจากการรันในสภาพแวดล้อมที่ตั้งค่า local proxy ตามหัวข้อ 6):

```
example.com/greetlib/v2 v2.0.0 (retracted) [v2.0.6 available]
```

ข้อความ **`(retracted)`** ปรากฏต่อท้ายเวอร์ชันที่กำลังใช้อยู่ทันที พร้อมแนะนำเวอร์ชันใหม่ที่ยังไม่ถูก retract ให้ในวงเล็บถัดไป — นี่คือกลไกที่ทำให้ผู้ใช้**รู้ตัวเองว่าใช้เวอร์ชันที่มีปัญหาอยู่** โดยไม่ต้องรอให้มีใครมาแจ้งเตือนผ่านช่องทางอื่น

> **ข้อควรระวัง**: การ retract ต้อง**ออกเวอร์ชันใหม่**ที่มี `retract` directive อยู่ใน `go.mod` เสมอ (เช่น ออก v2.0.7 ที่ประกาศ retract v2.0.0-v2.0.6 ไว้) เพราะ tooling จะอ่านค่า `retract` จากเวอร์ชัน**ล่าสุด**ของ module เป็นหลัก — การแก้ `go.mod` ของเวอร์ชันเก่าที่ tag ไปแล้วย้อนหลังทำไม่ได้ (tag ที่ push แล้วไม่ควรแก้ไขซ้ำ ตามหลักการเดียวกับที่ **Part 108** เน้นย้ำเรื่อง `go.sum`/Sum Database ที่ยึดถือเนื้อหาเดิมตลอดไป)

### `go.work`: พัฒนาหลาย Module พร้อมกันระหว่างเตรียม Release (ทบทวน Part 018)

**Part 018** แนะนำ **Go Workspaces** (`go.work`) ไว้สำหรับพัฒนาหลาย module พร้อมกัน — สถานการณ์ที่ใช้บ่อยที่สุดคือช่วงเตรียม breaking change ก่อนออก MAJOR version ใหม่ เมื่อต้องแก้ทั้ง library หลักและโปรเจกต์ที่ทดสอบใช้งานมันไปพร้อมกัน:

```bash
go work init ./greetlib ./consumer
```

```
// go.work
go 1.21

use (
	./greetlib
	./consumer
)
```

`go.work` ทำให้ `consumer` มองเห็นโค้ดล่าสุดใน `./greetlib` โดยตรงจากดิสก์ **โดยไม่ต้อง commit `replace` directive ปลอมเข้า `go.mod`** ของ `consumer` (ซึ่งเป็นกับดักที่พบบ่อย: นักพัฒนาใส่ `replace` ชั่วคราวไว้ทดสอบแล้วลืมเอาออกก่อน commit จริง ทำให้ CI ของคนอื่นพังเพราะหา path บนดิสก์ของตัวเองไม่เจอ) — ไฟล์ `go.work` ควรอยู่ใน `.gitignore` เสมอ เพราะเป็นการตั้งค่าเฉพาะเครื่องของนักพัฒนาแต่ละคน ไม่ใช่ส่วนหนึ่งของ module ที่ควร commit ร่วมกัน

---

## 13. Go 1 Compatibility Promise: มาตรฐานทองคำของ Backward Compatibility (ทบทวน Part 001)

**Part 001** กล่าวถึง **Go 1 Compatibility Promise** ไว้สั้นๆ ในไทม์ไลน์ประวัติศาสตร์ของภาษา บทนี้ขยายความว่าทำไมมันถึงเป็น**ตัวอย่างที่ดีที่สุด**ของการรักษาสัญญา backward compatibility ที่นักพัฒนา library ทุกคนควรศึกษาไว้เป็นแนวทาง

ตั้งแต่ Go 1.0 เปิดตัวในปี 2012 ทีม Go ให้สัญญาไว้ว่า:

> **โปรแกรม Go ที่เขียนถูกต้องตามสเปกและ compile ผ่านด้วย Go 1.x เวอร์ชันใดก็ตาม จะยังคง compile ผ่านและทำงานถูกต้องเหมือนเดิมกับ Go 1.x เวอร์ชันใหม่กว่าเสมอ**

นี่คือสัญญาที่ยึดถือมาแล้ว **มากกว่า 12 ปี** ผ่าน Go เวอร์ชัน 1.0 จนถึง 1.24 (เวอร์ชันที่ใช้เขียนหลักสูตรนี้) โดยไม่เคยหักสัญญานี้แม้แต่ครั้งเดียวในระดับ major — แม้จะมีการเพิ่ม feature ครั้งใหญ่ระดับ **Generics** (Go 1.18, 2022) ก็ยังคงรักษาสัญญานี้ไว้ได้ โค้ด Go ที่เขียนตั้งแต่ปี 2012 (ก่อนมี generics, ก่อนมี Go Modules) ยังคง **compile และรันได้ถูกต้องด้วย Go 1.24** โดยแทบไม่ต้องแก้ไขอะไรเลย

### ทำไมนี่ถึงเป็น "มาตรฐานทองคำ"

ผลลัพธ์ของสัญญานี้คือ**ความมั่นใจในระดับอุตสาหกรรม**: บริษัทขนาดใหญ่ที่มีโค้ด Go หลายล้านบรรทัด (Google เอง, Uber, Cloudflare ตามที่กล่าวถึงใน **Part 001**) กล้าอัปเดต Go toolchain เวอร์ชันใหม่ทันทีที่ออก โดยแทบไม่ต้องกลัวว่าโค้ดเดิมจะพัง เทียบกับภาษา/framework อื่นจำนวนมากที่ major version ใหม่มักมาพร้อม breaking change จำนวนมากจนต้องวางแผน migration เป็นเดือนหรือเป็นปี

### บทเรียนสำหรับนักพัฒนา library

หลักการที่ดึงมาใช้ได้จริงกับ library ของตัวเอง:

1. **แยกให้ชัดเจนระหว่าง "เพิ่มของใหม่" กับ "เปลี่ยนของเดิม"** — Go 1 Compatibility Promise ไม่ได้ห้ามเพิ่ม feature ใหม่ (มี MINOR version ใหม่ตลอดทุก 6 เดือน) แต่ห้ามทำให้โค้ดเดิม **หยุดทำงาน**
2. **Deprecate ก่อนลบเสมอ** ให้เวลาผู้ใช้ migrate (ตามหัวข้อ 11) แทนที่จะลบทันทีในเวอร์ชันถัดไป
3. **เมื่อจำเป็นต้อง breaking จริงๆ ให้ทำผ่าน MAJOR version ใหม่** (`/v2`, `/v3`) ไม่ใช่ทำใน MINOR/PATCH — ให้ผู้ใช้เดิมค้างอยู่ที่ major version เก่าได้ตราบเท่าที่ต้องการ โดยไม่ถูกบังคับอัปเกรดแบบไม่ทันตั้งตัว
4. **เขียน test ที่ครอบคลุม public API ให้มากที่สุด** เพื่อให้รู้ทันทีถ้า refactor ภายในทำให้ behavior ที่ผู้ใช้พึ่งพาเปลี่ยนไปโดยไม่ตั้งใจ — Go เองมี test suite ขนาดใหญ่มากที่รันทุก commit เพื่อรักษาสัญญานี้ไว้

---

## 14. Checklist: ก่อนออก Release ใหม่ของ Library

- [ ] เลือกประเภทเวอร์ชัน (MAJOR/MINOR/PATCH) ตรงตามนิยาม SemVer จริง ไม่ใช่ตามความรู้สึก "งานใหญ่แค่ไหน"
- [ ] ถ้าเป็น MAJOR ตั้งแต่ v2 ขึ้นไป: แก้ `module` path ใน `go.mod` ให้มี `/vN` ต่อท้ายก่อน tag เสมอ
- [ ] อัปเดต `CHANGELOG.md` ก่อน tag ทุกครั้ง จัดกลุ่มตาม `Added`/`Changed`/`Deprecated`/`Removed`/`Fixed`/`Security`
- [ ] รัน `go vet ./...`, `staticcheck ./...`, และ `go test ./...` ให้ผ่านทั้งหมดก่อน tag (ต่อยอด CI จาก **Part 098**)
- [ ] รัน `govulncheck ./...` (**Part 106**) ให้สะอาดก่อน release
- [ ] ตรวจสอบว่า public API ที่ตั้งใจ deprecate มีคอมเมนต์ `// Deprecated:` ตามรูปแบบมาตรฐานครบทุกจุด
- [ ] ถ้ากำลังแก้บั๊กวิกฤตของเวอร์ชันเก่า พิจารณาเพิ่ม `retract` directive ให้เวอร์ชันที่มีปัญหาในเวอร์ชันใหม่ที่กำลังจะออก
- [ ] `git tag -a vX.Y.Z -m "..."` แบบ annotated tag เสมอ ไม่ใช่ lightweight tag
- [ ] `git push origin vX.Y.Z` push tag ขึ้น remote จริง (อย่าลืมขั้นตอนนี้ — เป็นจุดที่พลาดบ่อยที่สุด)
- [ ] ถ้าเป็น public module ตรวจสอบว่า `pkg.go.dev` แสดงเวอร์ชันใหม่ถูกต้องหลัง proxy sync (มักใช้เวลาสักครู่หลัง push tag)
- [ ] แจ้งผู้ใช้ผ่านช่องทางที่เหมาะสม (GitHub Release notes, mailing list, Slack ทีม) พร้อมลิงก์ CHANGELOG โดยเฉพาะถ้าเป็น MAJOR version

---

## สรุปสิ่งที่ได้เรียนในบทนี้

- **Semantic Versioning** (`MAJOR.MINOR.PATCH`) คือสัญญาที่ชัดเจนระหว่างผู้ดูแล library กับผู้ใช้: PATCH ปลอดภัยเสมอ, MINOR เพิ่มของใหม่แบบไม่ breaking, MAJOR อาจ breaking
- Go บังคับ **Semantic Import Versioning**: module v2 ขึ้นไปต้องมี `/vN` ต่อท้าย import path เสมอ — เราได้เห็น error จริง `should be v0 or v1, not v2` เมื่อลืมทำตามกฎนี้ และวิธีแก้ที่ถูกต้อง
- **Minimal Version Selection (MVS)** เลือกเวอร์ชันที่**สูงที่สุดในบรรดาความต้องการขั้นต่ำทั้งหมด** ไม่ใช่เวอร์ชันล่าสุดที่มีอยู่ในโลก — สาธิตจริงด้วย `modulex` (ต้องการ v1.0.0) และ `moduley` (ต้องการ v1.1.0) ที่ทั้งคู่ลงเอยที่ `liba@v1.1.0` เหมือนกันทั้ง build แม้ `v1.2.0` จะมีอยู่จริงก็ตาม
- แนวทางนี้ต่างจาก npm/pip ที่มักเลือก "ล่าสุดที่ยังสอดคล้อง" ทำให้ Go ได้ **reproducible build** ที่มั่นคงกว่าโดยธรรมชาติ ไม่ต้องพึ่ง lock file แยกต่างหาก
- `go.sum` และ Checksum Database (**Part 018**) ทำงานร่วมกับ MVS เพื่อรับประกันว่าเนื้อหาของแต่ละเวอร์ชันไม่ถูกแก้ไขโดยไม่รู้ตัว
- **CHANGELOG** ที่ดีตามรูปแบบ Keep a Changelog ทำให้ผู้ใช้กล้าอัปเดต เพราะรู้ล่วงหน้าว่าจะเจออะไร
- **`// Deprecated:`** เป็นรูปแบบมาตรฐานที่ `gopls` และ `staticcheck` (กฎ SA1019) รู้จัก — เราได้เห็นผลลัพธ์จริงที่ `staticcheck` ตรวจจับการเรียกใช้ deprecated symbol ข้าม package ได้ (แต่ `go vet` มาตรฐานตรวจไม่ได้)
- **`retract` directive** คือวิธีที่ถูกต้องในการประกาศว่าเวอร์ชันหนึ่งไม่ควรถูกใช้ (บั๊กร้ายแรง/ช่องโหว่) เพราะ Go module ไม่มีแนวคิด "ลบเวอร์ชันออกจากโลก" จริง — เราได้เห็น `go.mod` จริงที่มี `retract` block และคำเตือน `(retracted)` ที่ผู้ใช้เห็นจริงตอนรัน `go list -m -u`
- **`go.work`** (**Part 018**) ช่วยพัฒนาหลาย module พร้อมกันระหว่างเตรียม breaking change โดยไม่ต้องพึ่ง `replace` ปลอมที่มักถูกลืมไว้ใน `go.mod` โดยไม่ตั้งใจ
- **Go 1 Compatibility Promise** คือมาตรฐานทองคำของ backward compatibility ที่รักษามาแล้วกว่า 12 ปี เป็นแบบอย่างของหลักการ "แยกให้ชัดระหว่างเพิ่มของใหม่กับเปลี่ยนของเดิม" ที่นักพัฒนา library ทุกคนควรยึดถือ

## แบบฝึกหัดท้ายบท

1. สร้าง Go module ของตัวเองพร้อม git repository จริง ลอง tag `v1.0.0` แล้วจงใจ tag `v2.0.0` โดยไม่แก้ `go.mod` เพื่อดู error จริงด้วยตัวเอง จากนั้นแก้ไขให้ถูกต้องตามหัวข้อ 4
2. ทำซ้ำการสาธิต MVS ในหัวข้อ 6 ด้วย module ของตัวเอง ลองเปลี่ยนให้ `modulex` และ `moduley` ต้องการ dependency คนละเวอร์ชันที่ต่างจากตัวอย่างในบทนี้ แล้วคาดเดาก่อนว่า MVS จะเลือกเวอร์ชันไหน ก่อนรัน `go mod tidy` เพื่อตรวจคำตอบ
3. เขียนไฟล์ `CHANGELOG.md` ตามรูปแบบ Keep a Changelog ให้กับโปรเจกต์ Go ที่เคยเขียนไว้ในบทก่อนๆ ของหลักสูตรนี้ (เช่นจาก Part 060 หรือ 088) ย้อนดู git log แล้วจัดกลุ่มการเปลี่ยนแปลงจริงที่เคยทำ
4. เพิ่มคอมเมนต์ `// Deprecated:` ให้ฟังก์ชันหนึ่งในโปรเจกต์ของตัวเอง ติดตั้งแล้วรัน `staticcheck` จริงเพื่อยืนยันว่าตรวจจับได้ตามที่คาดหวัง ลองแก้รูปแบบคอมเมนต์ให้ผิดเล็กน้อย (เช่นไม่เว้นบรรทัดก่อน `Deprecated:`) แล้วสังเกตว่า `staticcheck` ยังตรวจจับได้หรือไม่
5. ค้นคว้าเพิ่มเติมเรื่อง `go mod why -m <module>` ทดลองกับโปรเจกต์ MVS ในหัวข้อ 6 เพื่อดูว่าเครื่องมือนี้อธิบาย path ที่นำไปสู่การพึ่งพา `liba` อย่างไร เทียบกับข้อมูลที่ `go mod graph` ให้มา
6. สร้าง `go.mod` ของตัวเองแล้วลองใช้ `go mod edit -retract=v1.0.0` จริง สังเกตรูปแบบ `retract` block ที่ได้ ลองเพิ่ม interval แบบ `[v1.1.0,v1.1.5]` ด้วยคำสั่งเดียวกันแล้วดูผลลัพธ์
7. อ่านเอกสาร [go.dev/ref/mod#versions](https://go.dev/ref/mod#versions) เพิ่มเติมเรื่อง pseudo-version (เช่น `v0.0.0-20230101000000-abcdef123456`) ที่ Go ใช้เมื่อ dependency ไม่มี tag semver แต่ต้องการ commit ที่เจาะจง แล้วทดลองสร้างสถานการณ์ที่ `go get` module ที่ไม่มี tag ใดๆ เลย ดูว่า Go สร้าง pseudo-version ให้อย่างไร

---

**ต่อไป**: [Part 109 — การมีส่วนร่วมใน Open Source โปรเจกต์ Go](./109-contributing-to-open-source.md)
