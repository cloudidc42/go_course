# Part 001: แนะนำภาษา Go, ปรัชญาการออกแบบ, การติดตั้ง และเครื่องมือ

> ภาคที่ 1: พื้นฐานภาษา Go (Fundamentals) — ตอนที่ 1 จาก 15

## สารบัญของบทนี้

1. Go คืออะไร และทำไมต้องเรียน
2. ประวัติและปรัชญาการออกแบบภาษา
3. ใครใช้ Go บ้างในโลกจริง
4. การติดตั้ง Go บน Windows, macOS, Linux
5. โครงสร้างการติดตั้งและตัวแปรแวดล้อม (Environment Variables)
6. เครื่องมือ Go Toolchain ทั้งหมดที่ต้องรู้
7. การตั้งค่า Editor/IDE (VS Code, GoLand, Vim/Neovim)
8. โปรแกรมแรก: Hello, World! แบบเจาะลึกทุกบรรทัด
9. Go Playground และการทดลองโค้ดออนไลน์
10. โครงสร้างโปรเจกต์ Go มาตรฐาน
11. แบบฝึกหัดท้ายบท

---

## 1. Go คืออะไร และทำไมต้องเรียน

**Go** (หรือเรียกอีกชื่อว่า **Golang**) เป็นภาษาโปรแกรมมิ่งที่พัฒนาโดย Google เปิดตัวสู่สาธารณะในปี 2009 โดยทีมวิศวกรที่มีชื่อเสียงระดับตำนาน 3 คน:

- **Robert Griesemer** — หนึ่งในผู้ร่วมพัฒนา V8 JavaScript Engine และ Java HotSpot VM
- **Rob Pike** — หนึ่งในผู้บุกเบิก Unix ที่ Bell Labs และผู้ร่วมสร้างภาษา UTF-8
- **Ken Thompson** — ผู้สร้าง Unix และภาษา B (ต้นกำเนิดของภาษา C)

Go ถูกออกแบบมาเพื่อแก้ปัญหาที่ทีมวิศวกรของ Google เจอในการพัฒนาซอฟต์แวร์ขนาดใหญ่:

- **Compile ช้า**: โปรเจกต์ C++ ขนาดใหญ่ใช้เวลา compile นานเป็นชั่วโมง
- **Dependency management ยุ่งเหยิง**: การจัดการ header files และ library ซับซ้อน
- **การเขียนโค้ด concurrent ยาก**: Thread และ lock ใน C++/Java เขียนยากและเกิด bug ง่าย
- **โค้ดอ่านยาก**: แต่ละทีมมีสไตล์การเขียนโค้ดต่างกัน ทำให้ maintain ยาก

### จุดเด่นของ Go

| คุณสมบัติ | รายละเอียด |
|---|---|
| **Compile เร็ว** | คอมไพล์เป็น native binary ได้เร็วมาก แม้โปรเจกต์ขนาดใหญ่ |
| **Static typing + Type inference** | ปลอดภัยแบบ static type แต่เขียนสั้นกระชับด้วย `:=` |
| **Garbage Collected** | ไม่ต้องจัดการ memory เองแบบ C/C++ |
| **Concurrency ในตัว** | Goroutines และ Channels ทำให้เขียนโปรแกรม concurrent ง่ายกว่าภาษาอื่นมาก |
| **Standard Library แข็งแกร่ง** | มี `net/http`, `encoding/json`, `crypto` ครบ ไม่ต้องพึ่ง library ภายนอกมากสำหรับงานพื้นฐาน |
| **Binary เดียวจบ** | คอมไพล์แล้วได้ไฟล์ executable เดียว ไม่ต้องติดตั้ง runtime หรือ dependency เพิ่ม |
| **Cross-compilation ง่าย** | คอมไพล์จาก Mac ให้รันบน Linux/Windows ได้โดยไม่ต้องมีเครื่องเป้าหมาย |
| **Syntax เรียบง่าย** | มี keyword เพียง 25 คำ เรียนรู้ได้เร็ว อ่านโค้ดคนอื่นเข้าใจง่าย |
| **Tooling ในตัวครบ** | `go fmt`, `go vet`, `go test`, `go doc` มาพร้อม ไม่ต้องหา tool เสริม |

Go ไม่ได้พยายามเป็นภาษาที่ "มีทุกอย่าง" แบบ C++ หรือ Scala แต่เลือกที่จะ **เรียบง่ายโดยเจตนา (simplicity by design)** ซึ่งเป็นปรัชญาหลักที่จะเจอซ้ำๆ ตลอดหลักสูตรนี้

---

## 2. ประวัติและปรัชญาการออกแบบภาษา

### Timeline สำคัญ

- **2007**: เริ่มออกแบบภายใน Google โดย Griesemer, Pike, Thompson
- **2009**: เปิดตัวเป็น Open Source
- **2012**: Go 1.0 เสถียร พร้อม **Go 1 Compatibility Promise** — สัญญาว่าโค้ดที่เขียนด้วย Go 1.x จะ compile ได้เสมอในเวอร์ชันถัดไป
- **2015**: Go 1.5 — compiler เขียนใหม่ทั้งหมดด้วยภาษา Go เอง (จากเดิมที่เขียนด้วย C)
- **2018**: Go Modules เปิดตัว (Go 1.11) แก้ปัญหา dependency management แบบเก่า (GOPATH)
- **2022**: Go 1.18 — เพิ่ม **Generics** ครั้งใหญ่ที่สุดในประวัติศาสตร์ภาษา
- **ปัจจุบัน**: Go ออกเวอร์ชันใหม่ทุก 6 เดือน (กุมภาพันธ์ และ สิงหาคม) พร้อมปรับปรุง performance และ tooling อย่างต่อเนื่อง

### ปรัชญา "Less is More"

Rob Pike เคยกล่าวไว้ว่า:

> "Less is exponentially more."

หมายความว่า ยิ่งภาษามี feature น้อยและเรียบง่าย ยิ่งทำให้ระบบที่ใหญ่ขึ้นนั้น**จัดการได้ง่ายขึ้นแบบทวีคูณ** ตัวอย่างการตัดสินใจออกแบบที่สะท้อนปรัชญานี้:

1. **ไม่มี class / inheritance** — ใช้ `struct` + `interface` + `composition` แทน
2. **ไม่มี exception** — ใช้ค่า return แบบ `error` explicit แทน `try/catch`
3. **ไม่มี generic ตอนเปิดตัว** (เพิ่มทีหลังปี 2022 หลังคิดมา 10 ปี เพื่อให้ได้ syntax ที่เรียบง่ายที่สุด)
4. **มี loop แบบเดียว**: `for` (ไม่มี `while`, `do-while`)
5. **Formatting มาตรฐานเดียว**: `gofmt` บังคับให้โค้ดทุกที่หน้าตาเหมือนกัน ไม่มีการเถียงเรื่อง tab vs space
6. **Unused import / unused variable คือ compile error** — บังคับให้โค้ด clean ตั้งแต่ระดับ compiler

### หลักการ "Do not communicate by sharing memory; share memory by communicating"

นี่คือหัวใจสำคัญของแนวคิด concurrency ใน Go แทนที่จะใช้ shared memory + lock แบบภาษาอื่น Go สนับสนุนให้ใช้ **channel** ในการส่งข้อมูลระหว่าง goroutine ซึ่งเราจะเรียนเจาะลึกใน ภาคที่ 3 (Concurrency)

---

## 3. ใครใช้ Go บ้างในโลกจริง

Go ไม่ใช่แค่ภาษาทดลอง แต่เป็นภาษาที่ขับเคลื่อนโครงสร้างพื้นฐานของอินเทอร์เน็ตยุคปัจจุบัน:

- **Google**: ใช้ในหลายระบบ internal, YouTube บางส่วน
- **Docker**: เขียนด้วย Go ทั้งหมด (Docker Engine, Docker CLI)
- **Kubernetes**: เขียนด้วย Go ทั้งหมด — มาตรฐานอุตสาหกรรม container orchestration
- **Uber**: ใช้ Go สำหรับ microservices จำนวนมาก
- **Netflix**: ใช้ Go สำหรับ tools และบาง service
- **Twitch**: ใช้ Go สำหรับ chat system ที่รองรับผู้ใช้หลายล้านคนพร้อมกัน
- **Cloudflare**: ใช้ Go ในหลาย edge service
- **HashiCorp**: Terraform, Vault, Consul, Nomad — เขียนด้วย Go ทั้งหมด
- **PayPal, Dropbox, SendGrid**: ใช้ Go ใน production ระดับ scale ใหญ่

จุดร่วมคือ: งานที่ต้องการ **performance สูง**, **concurrency สูง**, และ **deploy ง่าย** เช่น backend API, microservices, CLI tools, DevOps tools, cloud infrastructure — ล้วนเป็นจุดแข็งของ Go ทั้งสิ้น

---

## 4. การติดตั้ง Go บน Windows, macOS, Linux

### ตรวจสอบเวอร์ชันล่าสุด

ก่อนติดตั้งเสมอ ให้ตรวจสอบเวอร์ชันล่าสุดที่ https://go.dev/dl/ — หลักสูตรนี้อ้างอิงจาก **Go 1.22+**

### macOS

**วิธีที่ 1: ใช้ Homebrew (แนะนำ)**

```bash
brew install go
```

**วิธีที่ 2: ดาวน์โหลด .pkg installer**

ดาวน์โหลดจาก https://go.dev/dl/ แล้วติดตั้งแบบ double-click ตามปกติ

### Linux (Ubuntu/Debian)

```bash
# ดาวน์โหลดไฟล์ tarball (เปลี่ยนเลขเวอร์ชันตามที่ต้องการ)
wget https://go.dev/dl/go1.22.5.linux-amd64.tar.gz

# ลบเวอร์ชันเก่า (ถ้ามี) และแตกไฟล์ไปที่ /usr/local
sudo rm -rf /usr/local/go
sudo tar -C /usr/local -xzf go1.22.5.linux-amd64.tar.gz

# เพิ่ม PATH ใน ~/.bashrc หรือ ~/.zshrc
echo 'export PATH=$PATH:/usr/local/go/bin' >> ~/.bashrc
source ~/.bashrc
```

### Windows

1. ดาวน์โหลด `.msi` installer จาก https://go.dev/dl/
2. รัน installer แล้วกด Next ตามปกติ (ค่า default ติดตั้งที่ `C:\Go`)
3. เปิด Command Prompt ใหม่แล้วตรวจสอบด้วยคำสั่งด้านล่าง

### ตรวจสอบการติดตั้ง

```bash
go version
```

ผลลัพธ์ควรได้ประมาณ:

```
go version go1.22.5 linux/amd64
```

---

## 5. โครงสร้างการติดตั้งและตัวแปรแวดล้อม (Environment Variables)

Go มีตัวแปรแวดล้อมสำคัญหลายตัวที่ควบคุมพฤติกรรมของ toolchain ดูค่าปัจจุบันทั้งหมดด้วยคำสั่ง:

```bash
go env
```

ตัวแปรที่สำคัญที่สุด:

| ตัวแปร | ความหมาย | ค่า default |
|---|---|---|
| `GOROOT` | ตำแหน่งที่ติดตั้ง Go toolchain เอง | `/usr/local/go` (Linux/Mac) |
| `GOPATH` | ตำแหน่งเก็บ package ที่ดาวน์โหลด (สมัยก่อน Go Modules) และ `go install` binaries | `$HOME/go` |
| `GOBIN` | ตำแหน่งที่ `go install` วางไฟล์ executable | `$GOPATH/bin` |
| `GOOS` | ระบบปฏิบัติการเป้าหมายตอน compile | ตาม OS ปัจจุบัน |
| `GOARCH` | สถาปัตยกรรม CPU เป้าหมาย | ตามเครื่องปัจจุบัน |
| `GOPROXY` | Proxy server สำหรับดาวน์โหลด module | `https://proxy.golang.org,direct` |
| `GOPRIVATE` | Pattern ของ module ที่ไม่ควรผ่าน proxy สาธารณะ (ใช้กับ private repo บริษัท) | ว่าง |
| `GO111MODULE` | เปิด/ปิด Go Modules (ปัจจุบันเปิดเสมอโดย default) | `on` |
| `CGO_ENABLED` | เปิด/ปิดการใช้ C code ผ่าน cgo | `1` |

ตั้งค่าตัวแปรแวดล้อมแบบถาวรด้วยคำสั่ง (ตัวอย่างการปิด cgo เพื่อ build binary แบบ static):

```bash
go env -w CGO_ENABLED=0
```

คำสั่ง `go env -w` จะเขียนค่าไปเก็บไว้ในไฟล์ config ของ Go (ถาวรข้าม terminal session) ต่างจากการ `export` ผ่าน shell ที่จะหายเมื่อปิด terminal

### สำคัญ: `GOPATH` ในยุค Go Modules

ก่อนปี 2018 ทุกโปรเจกต์ Go **ต้อง**อยู่ใน `$GOPATH/src` เท่านั้น ทำให้จัดการ dependency ยากมาก ปัจจุบันเราใช้ **Go Modules** ทำให้โปรเจกต์อยู่ที่ไหนก็ได้ในเครื่อง — `GOPATH` ปัจจุบันใช้เก็บแค่ cache ของ module ที่ดาวน์โหลดมา (`$GOPATH/pkg/mod`) เราจะเรียนเรื่อง Go Modules แบบเจาะลึกใน **Part 002**

---

## 6. เครื่องมือ Go Toolchain ทั้งหมดที่ต้องรู้

คำสั่ง `go` เป็นเหมือนศูนย์รวมเครื่องมือทั้งหมด (คล้าย `git` ที่มี subcommand เยอะ) มาดูคำสั่งที่ใช้บ่อยที่สุด:

```bash
go build      # คอมไพล์โปรเจกต์เป็น executable (ไม่รัน)
go run        # คอมไพล์และรันทันที (ไม่เก็บไฟล์ binary ไว้)
go test       # รัน unit tests
go fmt        # จัดรูปแบบโค้ดตามมาตรฐาน Go อัตโนมัติ
go vet        # ตรวจจับข้อผิดพลาดเชิง logic ที่ compiler ตรวจไม่เจอ
go doc        # แสดง documentation ของ package/function
go get        # เพิ่ม/อัปเดต dependency ในโปรเจกต์
go install    # คอมไพล์และติดตั้ง binary ไปที่ GOBIN
go mod        # จัดการ Go Modules (init, tidy, download, verify)
go clean      # ลบไฟล์ cache/build artifact
go env        # แสดง/ตั้งค่าตัวแปรแวดล้อมของ Go
go list       # แสดงข้อมูล package
go generate   # รันโค้ด generator ที่ระบุด้วย comment พิเศษ
gofmt         # ตัว formatter ดิบ (go fmt คือ wrapper ของตัวนี้)
```

ลองคำสั่งพื้นฐานที่สุดตอนนี้:

```bash
go version   # เช็คเวอร์ชัน
go help      # ดูรายการคำสั่งทั้งหมด
go help build  # ดูรายละเอียดคำสั่ง build
```

เราจะใช้คำสั่งเหล่านี้ซ้ำๆ ตลอดหลักสูตร โดยเฉพาะ `go run`, `go build`, `go test`, `go mod` และ `go fmt`

---

## 7. การตั้งค่า Editor/IDE

### Visual Studio Code (แนะนำสำหรับผู้เริ่มต้น — ฟรีและเบา)

1. ติดตั้ง [VS Code](https://code.visualstudio.com/)
2. ติดตั้ง extension ชื่อ **Go** (publisher: Go Team at Google)
3. เปิดไฟล์ `.go` ใดๆ ครั้งแรก VS Code จะถามให้ติดตั้ง tools เพิ่ม (เช่น `gopls`, `dlv`, `staticcheck`) กด **Install All**
4. `gopls` คือ Go Language Server ที่ทำให้มี autocomplete, go-to-definition, real-time error checking

ฟีเจอร์สำคัญที่จะได้หลังติดตั้ง:
- Auto-format เมื่อ save (`gofmt` ทำงานอัตโนมัติ)
- Auto import package ที่ใช้
- Inline error/warning จาก `go vet`
- Debugging ด้วย `dlv` (Delve) — จะเรียนใน Part 087

### GoLand (จ่ายเงิน — เหมาะสำหรับทีมมืออาชีพ)

IDE เฉพาะทางจาก JetBrains มี refactoring tools ที่ทรงพลังมาก เหมาะกับโปรเจกต์ขนาดใหญ่ระดับองค์กร

### Vim/Neovim

ใช้ plugin `vim-go` หรือตั้งค่า `gopls` ผ่าน LSP client เช่น `nvim-lspconfig` สำหรับผู้ที่คุ้นเคยกับ terminal-based workflow

> **คำแนะนำ**: หลักสูตรนี้จะใช้ VS Code เป็นหลักในการสาธิต แต่ตัวอย่างโค้ดทั้งหมดรันได้จาก terminal โดยตรงด้วย `go run` โดยไม่จำเป็นต้องใช้ IDE ใดเป็นพิเศษ

---

## 8. โปรแกรมแรก: Hello, World! แบบเจาะลึกทุกบรรทัด

สร้างโฟลเดอร์โปรเจกต์และไฟล์แรก:

```bash
mkdir hello-go
cd hello-go
go mod init hello-go
```

สร้างไฟล์ `main.go`:

```go
package main

import "fmt"

func main() {
	fmt.Println("Hello, World!")
}
```

รันโปรแกรม:

```bash
go run main.go
```

ผลลัพธ์:

```
Hello, World!
```

### เจาะลึกทุกบรรทัด

```go
package main
```

- ทุกไฟล์ Go ต้องอยู่ใน **package** โดยไฟล์ที่จะ compile เป็น **executable program** ต้องอยู่ใน package ชื่อ `main` เท่านั้น
- ถ้าไฟล์อยู่ใน package อื่น (เช่น `package mathutil`) ไฟล์นั้นจะกลายเป็น **library package** ที่ import ไปใช้ในโปรเจกต์อื่นได้ แต่รันตรงๆ ไม่ได้

```go
import "fmt"
```

- `import` ใช้นำเข้า package อื่นมาใช้งาน
- `fmt` (format) เป็น package มาตรฐานของ Go สำหรับจัดการ input/output แบบมีรูปแบบ (formatted I/O) เช่น พิมพ์ข้อความ, อ่าน input, สร้าง string
- Go **บังคับ**ว่า import ที่ไม่ได้ใช้งานจะทำให้ compile error ทันที — นี่คือการบังคับความสะอาดของโค้ดตั้งแต่ compiler

```go
func main() {
	fmt.Println("Hello, World!")
}
```

- `func` คือ keyword สำหรับประกาศฟังก์ชัน
- `main()` เป็นฟังก์ชันพิเศษ: เมื่ออยู่ใน `package main` ฟังก์ชันนี้คือ**จุดเริ่มต้นการทำงาน (entry point)** ของโปรแกรม โปรแกรมจะเริ่มรันจากบรรทัดแรกใน `main()` เสมอ
- `fmt.Println(...)` คือการเรียกใช้ function `Println` จาก package `fmt` เพื่อพิมพ์ข้อความออกทาง standard output พร้อมขึ้นบรรทัดใหม่ท้ายข้อความอัตโนมัติ

### สังเกตเรื่อง Capitalization ที่สำคัญมาก

`Println` ขึ้นต้นด้วยตัวใหญ่ (`P`) — นี่**ไม่ใช่**เรื่องสไตล์การเขียนโค้ดธรรมดา แต่เป็นกฎภาษา Go โดยตรง:

> **ชื่อ (identifier) ที่ขึ้นต้นด้วยตัวพิมพ์ใหญ่ จะถูก export ออกจาก package (เข้าถึงได้จาก package อื่น) ส่วนชื่อที่ขึ้นต้นด้วยตัวพิมพ์เล็ก จะเข้าถึงได้เฉพาะภายใน package เดียวกันเท่านั้น**

นี่คือกลไก **access control** ของ Go แทนที่จะมี keyword `public`/`private` แบบ Java หรือ `pub` แบบ Rust กฎนี้ใช้กับทุกอย่าง: function, struct, field ของ struct, method, constant, variable เราจะเจอกฎนี้ซ้ำไปซ้ำมาตลอดหลักสูตร

### go build vs go run

```bash
go build main.go   # สร้างไฟล์ executable ชื่อ main (หรือ main.exe บน Windows)
./main              # รัน executable ที่ได้
```

เทียบกับ:

```bash
go run main.go      # compile + รัน ในคำสั่งเดียว ไม่เก็บไฟล์ binary ไว้ (จริงๆ compile ไปที่ temp directory)
```

ระหว่างพัฒนา ใช้ `go run` เพื่อความเร็ว แต่ตอน deploy จริงต้องใช้ `go build` เพื่อได้ไฟล์ binary ไปรันบน server

---

## 9. Go Playground และการทดลองโค้ดออนไลน์

หากยังไม่สะดวกติดตั้งเครื่อง หรือต้องการทดลองโค้ดสั้นๆ อย่างรวดเร็ว สามารถใช้ **Go Playground** ได้ที่ https://go.dev/play/

ข้อจำกัดของ Playground:
- ไม่มี network access (เชื่อมต่อ internet ไม่ได้)
- ไม่มี filesystem จริง
- จำกัดเวลาและ memory การรัน

เหมาะสำหรับทดลอง syntax, algorithm สั้นๆ หรือแชร์โค้ดตัวอย่างให้คนอื่นดู (แชร์เป็นลิงก์ได้) แต่ตัวอย่างในหลักสูตรนี้ตั้งแต่ part ที่เกี่ยวกับ file I/O, network, database เป็นต้นไป จำเป็นต้องรันบนเครื่องจริง

---

## 10. โครงสร้างโปรเจกต์ Go มาตรฐาน

แม้ Go จะไม่บังคับโครงสร้างโฟลเดอร์แบบเป็นทางการ แต่ชุมชนมีธรรมเนียมที่ยึดถือกันอย่างกว้างขวาง เรียกว่า [`golang-standards/project-layout`](https://github.com/golang-standards/project-layout) ตัวอย่างโครงสร้างสำหรับโปรเจกต์ขนาดกลาง-ใหญ่:

```
myproject/
├── cmd/                  # จุดเริ่มต้นโปรแกรม (main package) แยกตาม binary
│   └── myapp/
│       └── main.go
├── internal/             # โค้ดภายในที่ห้าม import จากนอกโปรเจกต์
│   ├── handler/
│   ├── service/
│   └── repository/
├── pkg/                  # โค้ดที่ตั้งใจให้โปรเจกต์อื่น import ไปใช้ได้
├── api/                  # API spec เช่น OpenAPI, Protobuf definitions
├── configs/              # ไฟล์ config
├── scripts/              # สคริปต์ build/deploy
├── test/                 # integration test เพิ่มเติม
├── go.mod
├── go.sum
└── README.md
```

จุดที่สำคัญที่สุดตอนนี้คือโฟลเดอร์ **`internal/`** — Go มีกฎพิเศษระดับ compiler ว่า package ใดๆ ที่อยู่ใต้โฟลเดอร์ชื่อ `internal` จะ import ได้เฉพาะจากโค้ดที่อยู่ **ภายใต้โฟลเดอร์แม่ของ `internal` เดียวกันเท่านั้น** ทำให้เป็นเครื่องมือบังคับ encapsulation ระดับโปรเจกต์ที่ทรงพลังมาก เราจะกลับมาใช้โครงสร้างนี้จริงจังใน**ภาคที่ 10 (โปรเจกต์จริง)**

สำหรับตอนนี้ (เพิ่งเริ่มต้น) โปรเจกต์เล็กๆ แค่มี `main.go` กับ `go.mod` ที่ root ก็เพียงพอแล้ว — เราจะค่อยๆ ขยายโครงสร้างตามความซับซ้อนที่เพิ่มขึ้นตลอดหลักสูตร

---

## สรุปสิ่งที่ได้เรียนในบทนี้

- Go คือภาษาที่เน้นความเรียบง่าย compile เร็ว และรองรับ concurrency ในตัว สร้างโดยทีมงานระดับตำนานที่ Google
- ปรัชญาหลักคือ "less is more" — feature น้อยแต่ compose กันได้ทรงพลัง
- ติดตั้ง Go ได้ทั้ง Windows/macOS/Linux และตรวจสอบด้วย `go version`
- `go env` ควบคุมพฤติกรรม toolchain ทั้งหมด โดยเฉพาะ `GOPATH`, `GOROOT`, `GOPROXY`
- คำสั่งที่ใช้บ่อยที่สุด: `go run`, `go build`, `go test`, `go mod`, `go fmt`
- ชื่อขึ้นต้นตัวใหญ่ = export, ตัวเล็ก = private — กฎนี้สำคัญมากและจะเจอตลอดหลักสูตร
- โครงสร้างโปรเจกต์มาตรฐานใช้ `cmd/`, `internal/`, `pkg/` เป็นหลัก

## แบบฝึกหัดท้ายบท

1. ติดตั้ง Go บนเครื่องของตัวเอง แล้วรัน `go version` และ `go env` ถ่ายภาพหน้าจอผลลัพธ์เก็บไว้
2. เขียนโปรแกรม `main.go` ที่พิมพ์ชื่อและอายุของตัวเองออกทางหน้าจอ คนละบรรทัด โดยใช้ `fmt.Println` สองครั้ง
3. ลองเปลี่ยน `fmt.Println` เป็น `fmt.println` (ตัว p เล็ก) แล้วรันดูว่า error อะไรเกิดขึ้น — อธิบายว่าทำไม (เชื่อมโยงกับกฎ export ที่เรียนไปในบทนี้)
4. ลองลบบรรทัด `import "fmt"` ออกแล้วรันดูว่า error คืออะไร
5. ติดตั้ง VS Code พร้อม Go extension แล้วลองพิมพ์โค้ด Hello World ใหม่ สังเกตว่า autocomplete และ auto-format ทำงานอย่างไรเมื่อกด save
6. ค้นหาข้อมูลเพิ่มเติม: ทำไม Go ถึงเลือกใช้ Garbage Collector แทนที่จะให้ผู้เขียนโปรแกรมจัดการ memory เองแบบ C/C++? มีข้อดีข้อเสียอย่างไร (เตรียมคำตอบไว้ เราจะพูดถึงเรื่องนี้อีกครั้งในภาคที่ Performance)

---

**ต่อไป**: [Part 002 — โครงสร้างโปรแกรม, package, Go Module, `go.mod`](./002-packages-and-modules.md)
