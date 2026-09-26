# Part 089: gRPC เบื้องต้นด้วย Protocol Buffers

> ภาคที่ 8: Microservices, gRPC, Message Queue — ตอนที่ 2 จาก 7 (Part 88–94)

## สารบัญของบทนี้

1. gRPC คืออะไร และแก้ปัญหาอะไรที่ REST ทำได้ไม่ดี
2. รากฐานของ gRPC: HTTP/2 และ Protocol Buffers
3. REST+JSON เทียบกับ gRPC+Protobuf แบบข้อมูลจริง
4. ติดตั้ง `protoc` (Protocol Buffer Compiler)
5. ติดตั้งปลั๊กอิน Go: `protoc-gen-go` และ `protoc-gen-go-grpc`
6. เขียนไฟล์ `.proto` ตัวแรก: GreeterService
7. Generate โค้ด Go จาก `.proto`
8. สำรวจโค้ดที่ถูก generate ออกมา
9. Implement gRPC Server
10. เขียน gRPC Client
11. รันจริง: พิสูจน์ว่า RPC ทำงานผ่าน network จริง
12. โครงสร้างภายใน `.proto` ที่ควรรู้เพิ่มเติม
13. สรุปสิ่งที่ได้เรียนในบทนี้
14. แบบฝึกหัดท้ายบท

---

## 1. gRPC คืออะไร และแก้ปัญหาอะไรที่ REST ทำได้ไม่ดี

ใน Part 059 เราเรียนหลักการออกแบบ RESTful API ไปแล้วอย่างละเอียด — REST เหมาะมากสำหรับ API ที่ client ภายนอก (เว็บ เบราว์เซอร์ มือถือ นักพัฒนาภายนอก) จะเรียกใช้ เพราะใช้ HTTP/JSON ที่เข้าใจง่าย เครื่องมือรองรับทุกที่ (เบราว์เซอร์เรียกได้ตรงๆ, `curl` ทดสอบได้ทันที)

แต่เมื่อระบบแตกเป็น Microservices ตามที่เรียนใน Part 088 จะเกิดการสื่อสารแบบใหม่ขึ้นมาก: **service เรียก service** เช่น `order-service` เรียก `inventory-service` เพื่อเช็คสต็อกก่อนยืนยันคำสั่งซื้อ — การสื่อสารแบบนี้เกิดขึ้น**บ่อยมาก บ่อยกว่า** การเรียกจาก client ภายนอกหลายเท่า และไม่มีเบราว์เซอร์หรือมนุษย์มาเกี่ยวข้องเลย ความต้องการจึงเปลี่ยนจาก "อ่านง่าย เข้าถึงง่าย" ไปเป็น **"เร็วที่สุด ประหยัด bandwidth ที่สุด ปลอดภัยต่อ type ที่สุด"**

**gRPC** (เดิมย่อมาจาก "gRPC Remote Procedure Call" พัฒนาโดย Google และปัจจุบันเป็นโปรเจกต์ของ CNCF เช่นเดียวกับ Kubernetes) คือ framework สำหรับสื่อสารระหว่าง service ที่ออกแบบมาเพื่อแก้จุดอ่อนของ REST+JSON ในบริบทนี้โดยเฉพาะ:

- **Binary protocol แทน text (JSON)** — ข้อมูลถูกเข้ารหัสด้วย Protocol Buffers (Protobuf) ซึ่งเป็น binary format ที่เล็กและ parse เร็วกว่า JSON มาก
- **Strongly-typed contract** — service ทุกตัวต้องนิยาม schema ไว้ในไฟล์ `.proto` ก่อนใช้งาน ทำให้ client และ server รู้ตรงกันเป๊ะว่าข้อมูลหน้าตาเป็นอย่างไร ผิด type ตั้งแต่ตอน compile ไม่ต้องรอ error ตอน runtime แบบ JSON ที่เป็น dynamic
- **Code generation อัตโนมัติ** — เขียน `.proto` ครั้งเดียว generate เป็นโค้ด client/server ได้หลายภาษา (Go, Python, Java, C++ ฯลฯ) พร้อมกัน ไม่ต้องเขียน struct/model มือซ้ำในแต่ละภาษา
- **สร้างบน HTTP/2** — ได้ประโยชน์จาก multiplexing, header compression และรองรับ **streaming** ในตัว (ซึ่งเราจะเรียนเจาะลึกใน Part 090)

จุดสำคัญที่ต้องเข้าใจให้ชัด: **gRPC ไม่ได้มาแทนที่ REST** ทั้งสองมีที่ทางของตัวเอง REST ยังคงเหมาะที่สุดสำหรับ public API ที่ client ภายนอกหลากหลายต้องเรียกใช้ ส่วน gRPC เหมาะที่สุดสำหรับการสื่อสาร **ภายในระบบ (service-to-service)** ที่ทั้งสองฝั่งเป็นโค้ดที่เราควบคุมเอง และต้องการความเร็ว/ความปลอดภัยด้าน type สูงสุด

---

## 2. รากฐานของ gRPC: HTTP/2 และ Protocol Buffers

### HTTP/2

ใน Part 047 เราสร้าง HTTP server ด้วย `net/http` ซึ่งโดย default เป็น HTTP/1.1 gRPC สร้างอยู่บน **HTTP/2** ซึ่งมีข้อได้เปรียบสำคัญเหนือ HTTP/1.1:

- **Multiplexing**: ส่งหลาย request/response พร้อมกันบน connection (TCP) เดียว โดยไม่ต้องรอกัน (HTTP/1.1 ต้องเปิดหลาย connection หรือรอ response ก่อนส่ง request ถัดไปในบาง mode)
- **Header compression (HPACK)**: ลดขนาด header ที่ซ้ำๆ กันในแต่ละ request
- **Binary framing**: HTTP/2 เองก็เป็น binary protocol ไม่ใช่ text แบบ HTTP/1.1 ทำให้ parse เร็วกว่า
- **รองรับ full-duplex streaming โดยธรรมชาติ**: เปิด stream เดียวแล้วทั้งสองฝั่งส่งข้อมูลสวนทางกันได้ตลอดเวลา (พื้นฐานของ bidirectional streaming ใน Part 090)

### Protocol Buffers (Protobuf)

**Protobuf** คือ **Interface Definition Language (IDL)** และ binary serialization format ที่ Google พัฒนาขึ้น แนวคิดคือ:

1. นิยามโครงสร้างข้อมูลและ service ในไฟล์ `.proto` (ภาษาที่เป็นกลาง ไม่ผูกกับภาษาโปรแกรมมิ่งใดภาษาหนึ่ง)
2. ใช้ตัว compile `protoc` แปลงไฟล์ `.proto` เป็นโค้ดในภาษาเป้าหมาย (Go, Python, Java, ...)
3. โค้ดที่ได้มีทั้ง struct/class สำหรับข้อมูล และฟังก์ชันสำหรับ serialize/deserialize เป็น binary

ข้อได้เปรียบของ Protobuf เหนือ JSON คือ **ไม่ต้องส่งชื่อ field ไปด้วยทุกครั้ง** — JSON ส่ง `{"name": "John", "age": 30}` ที่มีชื่อ field ปนไปกับข้อมูลทุกครั้ง แต่ Protobuf เข้ารหัสด้วย **field number** (ตัวเลขที่กำหนดไว้ใน `.proto`) แทนชื่อ ทำให้ payload มีขนาดเล็กกว่ามาก และ parse เร็วกว่าเพราะไม่ต้อง parse ข้อความ (text parsing) แบบ JSON

---

## 3. REST+JSON เทียบกับ gRPC+Protobuf แบบข้อมูลจริง

| ประเด็น | REST + JSON | gRPC + Protobuf |
|---|---|---|
| Format ข้อมูล | Text (JSON) มนุษย์อ่านได้ | Binary อ่านไม่ได้ตรงๆ ต้อง decode |
| ขนาด payload | ใหญ่กว่า (ส่งชื่อ field ทุกครั้ง) | เล็กกว่ามาก (field number แทนชื่อ) |
| ความเร็วในการ parse | ช้ากว่า (ต้อง parse text) | เร็วกว่ามาก (binary decode ตรงๆ) |
| Contract ระหว่าง client/server | ไม่บังคับ (OpenAPI เป็นแค่เอกสาร ไม่ enforce ตอน compile) | บังคับด้วย `.proto` — ผิด type คือ compile error |
| HTTP version | ส่วนใหญ่ HTTP/1.1 | HTTP/2 เท่านั้น |
| Streaming | ทำได้ยาก (SSE, WebSocket แยกต่างหาก) | รองรับในตัว 4 รูปแบบ (Part 090) |
| เรียกจากเบราว์เซอร์โดยตรง | ได้เลย (`fetch`, `axios`) | ทำไม่ได้ตรงๆ ต้องผ่าน gRPC-Web หรือ gRPC-Gateway |
| อ่าน/debug ด้วยตาเปล่า | ง่าย (`curl` เห็น JSON ตรงๆ) | ยากกว่า (ต้องใช้เครื่องมือเช่น `grpcurl`) |
| เหมาะกับ | Public API, client หลากหลาย | Service-to-service ภายในระบบ |

ตัวเลขที่มักถูกอ้างอิงกันในวงการ (จาก benchmark ของทีม gRPC และรายงานหลายแหล่งที่ทดสอบ payload ขนาดใกล้เคียงกัน) คือ Protobuf ให้ payload เล็กกว่า JSON ประมาณ **3-10 เท่า** และเร็วกว่าในการ serialize/deserialize ประมาณ **5-10 เท่า** ขึ้นอยู่กับโครงสร้างข้อมูลและภาษาที่ใช้ทดสอบ ตัวเลขที่แน่นอนแตกต่างกันไปตามลักษณะข้อมูล แต่ทิศทางเดียวกันเสมอ: **binary protocol ที่มี schema ตายตัวเร็วกว่า text protocol ที่ต้อง parse แบบ dynamic เสมอ**

---

## 4. ติดตั้ง `protoc` (Protocol Buffer Compiler)

`protoc` คือตัวคอมไพเลอร์หลักที่แปลงไฟล์ `.proto` เป็นโค้ด ต้องติดตั้งแยกจาก Go toolchain เพราะมันเป็นเครื่องมือกลางที่ใช้ได้กับหลายภาษา ไม่ใช่ Go tool

> **หมายเหตุความโปร่งใส**: เนื้อหาในบทนี้ทั้งหมด ตั้งแต่การติดตั้ง `protoc`, generate โค้ด, ไปจนถึงรัน server/client จริง **ถูกรันจริงบนเครื่องที่ใช้เขียนบทเรียนนี้** ทุกคำสั่งและผลลัพธ์ที่แสดงคือของจริง ไม่ใช่การจำลอง

### macOS

```bash
brew install protobuf
```

### Ubuntu/Debian

```bash
sudo apt-get update
sudo apt-get install -y protobuf-compiler
```

นี่คือวิธีที่ใช้ติดตั้งจริงในเครื่องที่เขียนบทเรียนนี้ (Ubuntu 24.04) ผลลัพธ์การติดตั้ง:

```
Setting up protobuf-compiler (3.21.12-8.2ubuntu0.3) ...
Processing triggers for libc-bin (2.39-0ubuntu8.7) ...
```

### ติดตั้งจาก binary release โดยตรง (ทุก OS)

ถ้า package manager ของระบบให้เวอร์ชันเก่าเกินไป หรือไม่มี `apt`/`brew` สามารถดาวน์โหลด binary จาก [GitHub releases ของ protocolbuffers/protobuf](https://github.com/protocolbuffers/protobuf/releases) โดยตรง:

```bash
# ตัวอย่างสำหรับ Linux x86_64 (เปลี่ยนเลขเวอร์ชันตามที่ต้องการ)
curl -LO https://github.com/protocolbuffers/protobuf/releases/download/v28.2/protoc-28.2-linux-x86_64.zip
unzip protoc-28.2-linux-x86_64.zip -d $HOME/.local
export PATH="$HOME/.local/bin:$PATH"
```

### ตรวจสอบการติดตั้ง

```bash
protoc --version
```

ผลลัพธ์จริงในเครื่องที่ใช้เขียนบทเรียนนี้:

```
libprotoc 3.21.12
```

---

## 5. ติดตั้งปลั๊กอิน Go: `protoc-gen-go` และ `protoc-gen-go-grpc`

`protoc` เองไม่รู้จักภาษา Go โดยตรง มันทำงานผ่านระบบ **plugin**: เมื่อสั่ง `protoc --go_out=...` มันจะไปเรียก binary ชื่อ `protoc-gen-go` ที่ต้องอยู่ใน `PATH` ให้ทำหน้าที่ generate โค้ด Go จริงๆ เราต้องการปลั๊กอิน 2 ตัว:

- **`protoc-gen-go`**: generate struct ของ message (เช่น `HelloRequest`, `HelloResponse`) พร้อมฟังก์ชัน serialize/deserialize
- **`protoc-gen-go-grpc`**: generate client stub และ server interface ของ service (เช่น `GreeterServiceClient`, `GreeterServiceServer`)

ติดตั้งทั้งสองตัวด้วย `go install` (คำสั่งที่เรียนไปแล้วใน Part 001):

```bash
go install google.golang.org/protobuf/cmd/protoc-gen-go@latest
go install google.golang.org/grpc/cmd/protoc-gen-go-grpc@latest
```

`go install` จะวาง binary ไว้ที่ `$GOBIN` (ปกติคือ `$GOPATH/bin` ตามที่เรียนใน Part 001 หัวข้อ 5) ต้องแน่ใจว่า path นี้อยู่ใน `$PATH` ของ shell:

```bash
export PATH="$PATH:$(go env GOPATH)/bin"
```

ตรวจสอบว่าติดตั้งสำเร็จ — ผลลัพธ์จริงจากเครื่องที่เขียนบทเรียนนี้:

```bash
protoc-gen-go --version
# protoc-gen-go v1.36.12

protoc-gen-go-grpc --version
# protoc-gen-go-grpc 1.6.2
```

> **ข้อสังเกตเรื่องเวอร์ชัน Go**: ตอนรัน `go install` ทั้งสองคำสั่งข้างต้น พบว่า `protoc-gen-go-grpc` เวอร์ชันล่าสุดต้องการ Go >= 1.25 ทำให้ Go toolchain ที่มีกลไก **auto-toolchain switching** (ตั้งแต่ Go 1.21 เป็นต้นมา) ดาวน์โหลด Go เวอร์ชันใหม่กว่ามาติดตั้งชั่วคราวเพื่อ build ตัวปลั๊กอินนี้โดยเฉพาะ — นี่เป็นเรื่องปกติและเกิดเฉพาะตอน **build ตัวเครื่องมือ (tool) เอง** เท่านั้น ส่วนโค้ด Go ของ service ที่เราจะเขียนในบทนี้ทั้งหมดยังคง build และรันด้วย **Go 1.24.7** ตามที่ระบุใน `go.mod` ของแต่ละโปรเจกต์ (ตรวจสอบได้ด้วย `go version` ก่อน build เสมอ) ถ้าต้องการบังคับไม่ให้ดาวน์โหลด toolchain ใหม่ ให้ตั้ง `GOTOOLCHAIN=local` ก่อนรันคำสั่ง

---

## 6. เขียนไฟล์ `.proto` ตัวแรก: GreeterService

สร้างโปรเจกต์ใหม่:

```bash
mkdir grpc-basics && cd grpc-basics
go mod init grpc-basics
```

สร้างไฟล์ `greeter.proto`:

```protobuf
syntax = "proto3";

package greeter;

option go_package = "grpc-basics/greeterpb";

// GreeterService ให้บริการทักทายผู้ใช้
service GreeterService {
  // SayHello รับชื่อ แล้วตอบกลับด้วยข้อความทักทาย (unary RPC)
  rpc SayHello (HelloRequest) returns (HelloResponse);
}

message HelloRequest {
  string name = 1;
}

message HelloResponse {
  string message = 1;
}
```

เจาะลึกทีละส่วน:

- `syntax = "proto3";` — ระบุเวอร์ชันของภาษา Protobuf ที่ใช้ (`proto3` เป็นเวอร์ชันมาตรฐานปัจจุบัน เรียบง่ายกว่า `proto2` รุ่นเก่า)
- `package greeter;` — namespace ระดับ Protobuf (แยกจาก Go package) ป้องกันชื่อ message ชนกันเมื่อรวมหลาย `.proto` เข้าด้วยกัน
- `option go_package = "grpc-basics/greeterpb";` — บอก `protoc-gen-go` ว่าจะ generate โค้ด Go ไปไว้ที่ import path ไหน (สำคัญมาก ถ้าลืมใส่จะ generate ไม่ได้)
- `service GreeterService { ... }` — นิยาม RPC service พร้อมเมธอด `SayHello` ที่รับ `HelloRequest` และคืนค่า `HelloResponse` — รูปแบบนี้เรียกว่า **unary RPC** (request เดียว, response เดียว เหมือน function call ปกติ)
- `message HelloRequest { string name = 1; }` — นิยามโครงสร้างข้อมูล field `name` เป็น `string` และมี **field number = 1**

### field number สำคัญกว่าที่คิด

ตัวเลขหลัง `=` ในแต่ละ field (เช่น `= 1`) **ไม่ใช่ค่าเริ่มต้นหรือ index ธรรมดา** แต่คือ **หมายเลขที่ใช้แทนชื่อ field ตอนเข้ารหัสเป็น binary** — Protobuf ไม่ส่งคำว่า `"name"` ไปใน binary เลย ส่งแค่ `1: <ค่า>` เท่านั้น ทำให้ payload เล็กลงมาก เราจะพูดถึงกฎการใช้ field number แบบเจาะลึกในหัวข้อ 12

---

## 7. Generate โค้ด Go จาก `.proto`

รันคำสั่ง `protoc` เพื่อ generate โค้ด:

```bash
protoc --go_out=. --go_opt=paths=source_relative \
  --go-grpc_out=. --go-grpc_opt=paths=source_relative \
  greeter.proto
```

อธิบายแต่ละ flag:

- `--go_out=.` — บอกให้เรียก plugin `protoc-gen-go` และเขียนไฟล์ผลลัพธ์ลงโฟลเดอร์ปัจจุบัน
- `--go_opt=paths=source_relative` — จัดวางไฟล์ผลลัพธ์ตามตำแหน่งของไฟล์ `.proto` ต้นฉบับ (ไม่ใช้ path เต็มตาม `go_package`) ทำให้ควบคุมตำแหน่งไฟล์ได้ง่ายในโปรเจกต์เล็กๆ
- `--go-grpc_out=.` และ `--go-grpc_opt=paths=source_relative` — เหมือนกันแต่สำหรับ plugin `protoc-gen-go-grpc` ที่ generate ส่วน service/client/server

รันจริงในเครื่องที่เขียนบทเรียนนี้ คำสั่งจบโดยไม่มี error (`exit code 0`) และได้ไฟล์ใหม่ 2 ไฟล์:

```
greeter.pb.go        # struct ของ message + serialize/deserialize
greeter_grpc.pb.go    # client stub + server interface
```

จัดไฟล์เข้าโฟลเดอร์ `greeterpb/` ตามที่ระบุใน `go_package`:

```bash
mkdir -p greeterpb
mv greeter.pb.go greeter_grpc.pb.go greeterpb/
```

> **ทางลัดสำหรับโปรเจกต์จริง**: เมื่อมีหลายไฟล์ `.proto` การพิมพ์คำสั่ง `protoc` ยาวๆ ซ้ำทุกครั้งไม่สะดวก โปรเจกต์จริงมักเขียนคำสั่งนี้ไว้ใน `Makefile` หรือใช้ comment แบบ `//go:generate protoc ...` แล้วเรียกด้วย `go generate` (คำสั่งที่เรียนไปแล้วใน Part 001 หัวข้อ 6)

---

## 8. สำรวจโค้ดที่ถูก generate ออกมา

เข้าใจสิ่งที่ `protoc` generate ให้ จะช่วยให้ debug และเข้าใจ gRPC ได้ลึกขึ้นมาก มาดูส่วนสำคัญจาก `greeter.pb.go` (ตัดมาบางส่วน):

```go
// Code generated by protoc-gen-go. DO NOT EDIT.
// versions:
// 	protoc-gen-go v1.36.12
// 	protoc        v3.21.12
// source: greeter.proto

package greeterpb

type HelloRequest struct {
	state         protoimpl.MessageState
	Name          string `protobuf:"bytes,1,opt,name=name,proto3" json:"name,omitempty"`
	unknownFields protoimpl.UnknownFields
	sizeCache     protoimpl.SizeCache
}

func (x *HelloRequest) GetName() string {
	if x != nil {
		return x.Name
	}
	return ""
}
```

สังเกตสิ่งสำคัญ:

- **คอมเมนต์ `DO NOT EDIT`** — ไฟล์นี้ generate อัตโนมัติทุกครั้งที่รัน `protoc` ห้ามแก้มือเด็ดขาด เพราะจะถูกเขียนทับ ถ้าต้องการ logic เพิ่มเติมให้เขียนแยกไฟล์ต่างหากใน package เดียวกัน
- **struct tag `protobuf:"bytes,1,opt,name=name,proto3"`** — เก็บข้อมูล field number (`1`) และ wire type ไว้สำหรับตอน encode/decode
- **getter method `GetName()`** — Protobuf generate getter ให้ทุก field เสมอ (แม้ Go ปกติไม่นิยม getter สไตล์นี้) เหตุผลคือ getter จะ return zero value อย่างปลอดภัยแม้เรียกบน pointer ที่เป็น `nil` — ควรใช้ `req.GetName()` แทน `req.Name` ตรงๆ เมื่อไม่แน่ใจว่า pointer เป็น nil หรือไม่

และจาก `greeter_grpc.pb.go`:

```go
// GreeterServiceClient is the client API for GreeterService service.
type GreeterServiceClient interface {
	SayHello(ctx context.Context, in *HelloRequest, opts ...grpc.CallOption) (*HelloResponse, error)
}

// GreeterServiceServer is the server API for GreeterService service.
type GreeterServiceServer interface {
	SayHello(context.Context, *HelloRequest) (*HelloResponse, error)
	mustEmbedUnimplementedGreeterServiceServer()
}

type UnimplementedGreeterServiceServer struct{}
```

จุดสำคัญ: **`GreeterServiceServer` เป็น interface** ที่เราต้อง implement เอง — และมันบังคับให้ struct ของเรา embed `UnimplementedGreeterServiceServer` เข้าไปด้วย (ผ่าน `mustEmbedUnimplementedGreeterServiceServer()`) เหตุผลคือ **forward compatibility**: ถ้าวันหนึ่งเพิ่มเมธอดใหม่ใน `.proto` แล้ว regenerate โค้ด, server เดิมที่ embed `UnimplementedGreeterServiceServer` ไว้จะยัง compile ผ่าน (เพราะ struct ที่ embed ไว้มี default implementation ของเมธอดใหม่ที่คืน error "unimplemented" ให้อัตโนมัติ) แทนที่จะ compile error ทันที

---

## 9. Implement gRPC Server

เขียน `server/main.go`:

```go
package main

import (
	"context"
	"fmt"
	"log"
	"net"

	"google.golang.org/grpc"

	pb "grpc-basics/greeterpb"
)

// server implements pb.GreeterServiceServer
type server struct {
	pb.UnimplementedGreeterServiceServer
}

func (s *server) SayHello(ctx context.Context, req *pb.HelloRequest) (*pb.HelloResponse, error) {
	log.Printf("received request from: %s", req.GetName())
	msg := fmt.Sprintf("สวัสดี, %s! ยินดีต้อนรับสู่ gRPC", req.GetName())
	return &pb.HelloResponse{Message: msg}, nil
}

func main() {
	lis, err := net.Listen("tcp", ":50051")
	if err != nil {
		log.Fatalf("failed to listen: %v", err)
	}

	grpcServer := grpc.NewServer()
	pb.RegisterGreeterServiceServer(grpcServer, &server{})

	log.Println("gRPC server listening on :50051")
	if err := grpcServer.Serve(lis); err != nil {
		log.Fatalf("failed to serve: %v", err)
	}
}
```

เจาะลึกทีละส่วน:

- `type server struct { pb.UnimplementedGreeterServiceServer }` — embed struct ที่ generate มาให้ (เหตุผลตามหัวข้อ 8) แล้ว implement เมธอด `SayHello` ทับ (override) ด้วย logic จริงของเรา
- `net.Listen("tcp", ":50051")` — เปิด TCP listener แบบเดียวกับที่ `net/http` ใช้ภายในตอนเรียก `http.ListenAndServe` (ตามที่เรียนใน Part 047) เพียงแต่ gRPC ต้องจัดการ `net.Listener` เองแล้วส่งให้ `grpcServer.Serve(lis)`
- `grpc.NewServer()` — สร้าง gRPC server instance เปล่าๆ (ยังไม่ผูก service ใดๆ)
- `pb.RegisterGreeterServiceServer(grpcServer, &server{})` — ฟังก์ชันที่ generate มาให้ ใช้ผูก implementation ของเรา (`&server{}`) เข้ากับ gRPC server
- `grpcServer.Serve(lis)` — เริ่มรับ connection แบบ blocking (เหมือน `http.ListenAndServe`) จะไม่ return จนกว่า server จะหยุดทำงานหรือเกิด error

---

## 10. เขียน gRPC Client

เขียน `client/main.go`:

```go
package main

import (
	"context"
	"log"
	"time"

	"google.golang.org/grpc"
	"google.golang.org/grpc/credentials/insecure"

	pb "grpc-basics/greeterpb"
)

func main() {
	conn, err := grpc.NewClient("localhost:50051", grpc.WithTransportCredentials(insecure.NewCredentials()))
	if err != nil {
		log.Fatalf("did not connect: %v", err)
	}
	defer conn.Close()

	client := pb.NewGreeterServiceClient(conn)

	ctx, cancel := context.WithTimeout(context.Background(), 3*time.Second)
	defer cancel()

	resp, err := client.SayHello(ctx, &pb.HelloRequest{Name: "สมชาย"})
	if err != nil {
		log.Fatalf("could not greet: %v", err)
	}
	log.Printf("response from server: %s", resp.GetMessage())
}
```

เจาะลึกทีละส่วน:

- `grpc.NewClient(...)` — สร้าง connection ไปยัง server (API นี้มาแทน `grpc.Dial` รุ่นเก่าที่ยังใช้ได้แต่ไม่แนะนำแล้วในเวอร์ชันปัจจุบันของไลบรารี)
- `grpc.WithTransportCredentials(insecure.NewCredentials())` — เพราะเรายังไม่ได้ตั้งค่า TLS (เรื่อง TLS/security จะอยู่นอกขอบเขตบทนี้ แต่ในระบบ production จริงต้องใช้ credential ที่มี TLS เสมอ ไม่ควรใช้ `insecure` ข้ามเครือข่ายจริง)
- `pb.NewGreeterServiceClient(conn)` — สร้าง client stub จาก connection — นี่คือจุดที่ทำให้การเรียก gRPC "รู้สึกเหมือนเรียกฟังก์ชันในโปรเซสเดียวกัน" ทั้งที่จริงข้อมูลถูกส่งผ่าน network
- `context.WithTimeout(...)` — ตามที่เรียนใน Part 032 ทุกการเรียก gRPC ควรมี context ที่มี deadline เสมอ ป้องกันการรอค้างตลอดไปถ้า server ไม่ตอบ (เราจะเห็นว่า deadline นี้ถูกส่งผ่าน wire จริงๆ ไม่ใช่แค่ timeout ฝั่ง client เฉยๆ ใน Part 090)
- `client.SayHello(ctx, &pb.HelloRequest{Name: "สมชาย"})` — เรียก RPC เหมือนเรียกฟังก์ชันปกติ ส่ง `context` เป็น argument แรกตาม convention ที่เรียนใน Part 032

---

## 11. รันจริง: พิสูจน์ว่า RPC ทำงานผ่าน network จริง

จัดโครงสร้างไฟล์ให้ครบ:

```
grpc-basics/
├── go.mod
├── greeter.proto
├── greeterpb/
│   ├── greeter.pb.go
│   └── greeter_grpc.pb.go
├── server/
│   └── main.go
└── client/
    └── main.go
```

ติดตั้ง dependency:

```bash
go get google.golang.org/grpc@v1.68.1 google.golang.org/protobuf@v1.35.2
go mod tidy
```

> **หมายเหตุเรื่องเวอร์ชัน**: บทเรียนนี้ pin เวอร์ชัน `google.golang.org/grpc@v1.68.1` ไว้โดยตั้งใจ เพราะเวอร์ชันล่าสุด ณ ตอนเขียนบทเรียนต้องการ Go >= 1.25 ในขณะที่หลักสูตรนี้อ้างอิง Go 1.24.7 การ pin เวอร์ชันแบบนี้ทำให้ `go build` ไม่ต้อง auto-switch toolchain และยังคง build/run ด้วย Go 1.24.7 ได้ตรงตามที่ตั้งใจ — เมื่ออ่านบทเรียนนี้ในอนาคต สามารถใช้ `go get google.golang.org/grpc@latest` แทนได้ตามปกติถ้า Go ที่ติดตั้งเป็นเวอร์ชันใหม่กว่านี้

Build ตรวจสอบว่าไม่มี error:

```bash
go build ./...
go version
```

ผลลัพธ์จริง:

```
go version go1.24.7 linux/amd64
```

(build สำเร็จโดยไม่มี error ใดๆ)

รัน server ก่อนในเทอร์มินัลหนึ่ง:

```bash
go run ./server
```

ผลลัพธ์:

```
2026/09/26 03:40:59 gRPC server listening on :50051
```

จากนั้นรัน client ในอีกเทอร์มินัลหนึ่ง (หรือรอ background process):

```bash
go run ./client
```

ผลลัพธ์จริงที่ได้ฝั่ง client:

```
2026/09/26 03:41:00 response from server: สวัสดี, สมชาย! ยินดีต้อนรับสู่ gRPC
```

และฝั่ง server แสดง log ว่าได้รับ request เข้ามาจริง:

```
2026/09/26 03:40:59 gRPC server listening on :50051
2026/09/26 03:41:00 received request from: สมชาย
```

นี่คือการยืนยันแบบ end-to-end ว่า: client ส่ง `HelloRequest{Name: "สมชาย"}` ผ่าน TCP connection ไปยัง server จริง, server decode ข้อมูลจาก binary format กลับมาเป็น Go struct, ประมวลผล logic, แล้ว encode `HelloResponse` ส่งกลับมาผ่าน network เดิม — ทั้งหมดนี้เกิดขึ้นในเวลาไม่ถึงวินาที โดยที่โค้ดฝั่ง client และ server เขียนราวกับกำลังเรียกฟังก์ชันธรรมดาในโปรแกรมเดียวกัน นี่คือคุณค่าหลักที่ gRPC มอบให้: **ซ่อนความซับซ้อนของ network ไว้เบื้องหลัง type-safe function call**

---

## 12. โครงสร้างภายใน `.proto` ที่ควรรู้เพิ่มเติม

### กฎการใช้ field number

- **1-15**: ใช้พื้นที่ 1 byte ในการเข้ารหัส ควรสงวนไว้ให้ field ที่ใช้บ่อยที่สุด (เช่น `id`, `name`)
- **16-2047**: ใช้พื้นที่ 2 byte
- **ห้ามใช้ 19000-19999**: สงวนไว้สำหรับ Protobuf internal
- **field number ห้ามเปลี่ยนหลัง deploy ไปแล้ว** — เพราะฝั่ง client เก่าที่ยังไม่ได้ update จะตีความข้อมูลผิดทันที (นี่คือกฎ backward compatibility ที่สำคัญที่สุดของ Protobuf)

### `reserved` ป้องกันการใช้ field number ซ้ำโดยไม่ตั้งใจ

```protobuf
message User {
  reserved 2, 15, 9 to 11;
  reserved "old_field_name";

  string name = 1;
  string email = 3;
}
```

เมื่อลบ field ออกจาก message (เช่นเลิกใช้ field number 2 แล้ว) ควรประกาศ `reserved` ไว้เสมอ เพื่อป้องกันไม่ให้มีใครนำ field number หรือชื่อนั้นกลับมาใช้ใหม่โดยไม่ตั้งใจในอนาคต ซึ่งจะทำให้ client เก่าที่ cache schema เดิมไว้ตีความข้อมูลผิดพลาด

### `enum` และ `repeated`

```protobuf
enum OrderStatus {
  ORDER_STATUS_UNSPECIFIED = 0; // ค่าแรกของ enum ต้องเป็น 0 เสมอใน proto3
  ORDER_STATUS_PENDING = 1;
  ORDER_STATUS_CONFIRMED = 2;
  ORDER_STATUS_CANCELLED = 3;
}

message Order {
  string id = 1;
  OrderStatus status = 2;
  repeated string item_ids = 3; // repeated = array/slice ใน Go
}
```

`repeated` คือ keyword ที่ทำให้ field กลายเป็น slice ใน Go (`[]string` สำหรับตัวอย่างข้างต้น) ส่วน enum ค่าแรก (`= 0`) ควรตั้งชื่อแบบ `_UNSPECIFIED` เสมอ เพราะ proto3 ไม่แยกความแตกต่างระหว่าง "ไม่ได้ตั้งค่า" กับ "ตั้งค่าเป็น 0" การมีค่า UNSPECIFIED ชัดเจนช่วยให้โค้ดฝั่งรับรู้ได้ว่าฟิลด์นี้ผู้ส่งไม่ได้ตั้งใจส่งค่ามาจริงๆ

### `message` ซ้อนกันและ `import`

`.proto` ไฟล์หนึ่งสามารถ `import` message จากไฟล์อื่นได้ ทำให้แชร์ type ร่วมกันข้าม service ได้ (เช่น message `Money` หรือ `Timestamp` ที่ใช้ร่วมกันหลาย service) — Google เองมี **well-known types** สำเร็จรูปให้ import ใช้ เช่น `google/protobuf/timestamp.proto` สำหรับแทนค่าเวลาแบบมาตรฐาน แทนที่จะประดิษฐ์ type เวลาขึ้นมาเอง

---

## สรุปสิ่งที่ได้เรียนในบทนี้

- **gRPC** คือ framework สำหรับสื่อสารระหว่าง service ที่เน้นความเร็วและ type safety สร้างอยู่บน **HTTP/2** และ **Protocol Buffers** เหมาะกับการสื่อสารภายในระบบ (service-to-service) มากกว่า public API ที่ REST ยังคงเหมาะสมกว่า
- Protobuf เป็น binary format ที่เข้ารหัสด้วย **field number** แทนชื่อ field ทำให้ payload เล็กและ parse เร็วกว่า JSON
- ติดตั้งเครื่องมือ 3 ตัวที่จำเป็น: `protoc` (ผ่าน `apt`/`brew`/binary release), `protoc-gen-go`, `protoc-gen-go-grpc` (ทั้งสองผ่าน `go install`) — ทั้งหมดติดตั้งและใช้งานได้จริงตามที่สาธิตในบทนี้
- เขียนไฟล์ `.proto` นิยาม `service` และ `message` แล้ว generate โค้ด Go ด้วยคำสั่ง `protoc --go_out=... --go-grpc_out=...`
- โค้ดที่ generate มาให้ทั้ง struct ของ message พร้อม getter, และ interface ของ client/server ที่ embed `Unimplemented*Server` เพื่อ forward compatibility
- Implement server ด้วยการ embed `UnimplementedGreeterServiceServer` แล้ว override เมธอดที่ต้องการ, สร้าง client ด้วย `grpc.NewClient` แล้วเรียก RPC เหมือนเรียกฟังก์ชันปกติ
- ทดสอบรันจริง server และ client คุยกันผ่าน TCP port 50051 ได้สำเร็จ ยืนยันด้วย log ทั้งสองฝั่ง
- กฎสำคัญของ `.proto`: field number ห้ามเปลี่ยนหลัง deploy, ใช้ `reserved` ป้องกันการใช้ซ้ำ, enum ค่าแรกควรเป็น `_UNSPECIFIED = 0` เสมอ

## แบบฝึกหัดท้ายบท

1. ติดตั้ง `protoc`, `protoc-gen-go`, และ `protoc-gen-go-grpc` บนเครื่องของตัวเอง แล้วรัน `protoc --version`, `protoc-gen-go --version`, `protoc-gen-go-grpc --version` ยืนยันว่าติดตั้งสำเร็จ
2. ทำตามบทเรียนนี้ทีละขั้นตอน สร้าง `GreeterService` ของตัวเอง generate โค้ด รัน server/client แล้วยืนยันว่าได้ response กลับมาจริง
3. เพิ่มเมธอกใหม่ชื่อ `SayGoodbye` ใน `.proto` เดิม (รับ `HelloRequest` คืน `HelloResponse` เหมือนกัน) regenerate โค้ด แล้ว implement ฝั่ง server และเรียกจาก client
4. ลองแก้ field number ของ `HelloRequest.name` จาก `1` เป็น `2` แล้ว regenerate โค้ดแค่ฝั่ง server (ไม่แตะ client) จำลองสถานการณ์ client เก่า/server ใหม่ไม่ตรงกัน สังเกตว่าเกิดอะไรขึ้น (คำใบ้: ลองส่ง raw bytes ที่ encode ด้วย field number เก่า)
5. เพิ่ม message `Address` และใช้ `repeated Address addresses = 2;` ใน message `HelloRequest` ลอง generate แล้วดูว่าโค้ด Go ที่ได้ให้ type อะไรสำหรับ field นี้
6. เปรียบเทียบขนาดของ payload ระหว่าง `HelloRequest{Name: "test"}` ที่ encode เป็น Protobuf กับ JSON object `{"name": "test"}` ด้วยการเขียนโค้ดเล็กๆ วัดความยาว byte ทั้งสองแบบ (ใช้ `proto.Marshal` เทียบกับ `json.Marshal` ตามที่เรียนใน Part 025)

---

**ต่อไป**: [Part 090 — gRPC ขั้นสูง: Streaming, Interceptors](./090-grpc-advanced.md)
