# Part 054: แพ็กเกจ `log` และแนวทาง Logging ที่ดี

> ภาคที่ 4: Standard Library เชิงลึก — ตอนที่ 9 จาก 10 (Part 46–55)

## สารบัญของบทนี้

1. ทำไม `fmt.Println` ไม่ใช่ Logging
2. แพ็กเกจ `log` มาตรฐาน: `Println`, `Printf`
3. `log.Fatal` vs `log.Panic` vs การ return error ปกติ
4. สร้าง Logger กำหนดเองด้วย `log.New`
5. ข้อจำกัดของ `log` แบบดั้งเดิม
6. `log/slog`: Structured Logging มาตรฐานของ Go (1.21+)
7. Log Levels ใน `slog`: Debug, Info, Warn, Error
8. `slog.NewTextHandler` vs `slog.NewJSONHandler`
9. ตั้งค่า Global Default Logger
10. Contextual Logging ด้วย `slog.Group` และ `context.Context`
11. ทำไม Structured JSON Logs ถึงสำคัญกับระบบ Production
12. สรุปสิ่งที่ได้เรียนในบทนี้
13. แบบฝึกหัดท้ายบท

---

## 1. ทำไม `fmt.Println` ไม่ใช่ Logging

นักพัฒนา Go มือใหม่จำนวนมากใช้ `fmt.Println` หรือ `fmt.Printf` เป็นเครื่องมือ debug ตลอดการพัฒนาโปรแกรม ซึ่งไม่ผิดในขั้นตอนการทดลองโค้ดสั้นๆ แต่ **`fmt` ไม่ใช่เครื่องมือ logging** และไม่ควรใช้แทน logging ในโปรแกรมระดับ production ด้วยเหตุผลหลายข้อ:

| ปัญหาของ `fmt.Println` | ทำไมถึงเป็นปัญหา |
|---|---|
| ไม่มี timestamp ในตัว | ไม่รู้ว่า event เกิดขึ้นตอนไหน เวลาไล่ดู log ย้อนหลัง |
| ไม่มีแนวคิดเรื่อง log level | แยกไม่ออกระหว่างข้อความ debug ธรรมดากับ error ร้ายแรง |
| เขียนไป stdout เสมอ | ควบคุมปลายทาง (ไฟล์, syslog, log aggregator) ไม่ได้โดยตรง |
| ไม่มีโครงสร้างข้อมูล (unstructured) | ข้อความเป็น string อิสระ ไม่มี key-value ที่ระบบภายนอกอ่านต่อได้ง่าย |
| ไม่ thread-safe ในเชิง "การเขียนเป็นบรรทัดที่สมบูรณ์" | ถ้าหลาย goroutine เขียนพร้อมกัน ข้อความอาจปนกันได้ในบางกรณี |

`fmt` ยังคงเป็นเครื่องมือที่ยอดเยี่ยมสำหรับการจัดรูปแบบข้อความ (ตามที่เรียนเจาะลึกไปในบทที่ 021) แต่หน้าที่ "การบันทึกเหตุการณ์ของระบบเพื่อการตรวจสอบภายหลัง" ต้องใช้เครื่องมือที่ออกแบบมาสำหรับงานนี้โดยเฉพาะ — นั่นคือ package `log` และ `log/slog` ที่จะเรียนในบทนี้

---

## 2. แพ็กเกจ `log` มาตรฐาน: `Println`, `Printf`

Go มี package `log` ในมาตรฐานมาตั้งแต่เวอร์ชันแรกๆ ให้ฟังก์ชันระดับ package สำหรับ log ข้อความแบบง่ายที่สุด:

```go
package main

import "log"

func main() {
	log.Println("this is a log message with default flags")
	log.Printf("user=%s action=%s\n", "alice", "login")
}
```

ผลลัพธ์:

```
2026/09/26 02:49:11 this is a log message with default flags
2026/09/26 02:49:11 user=alice action=login
```

สังเกตว่า **ต่างจาก `fmt.Println` ตรงที่ `log.Println` แนบ timestamp มาให้อัตโนมัติ** (`2026/09/26 02:49:11`) นี่คือความแตกต่างพื้นฐานที่สุดระหว่าง `log` กับ `fmt` — `log` ถูกออกแบบมาให้เหมาะกับการบันทึกเหตุการณ์ตั้งแต่ต้น โดย default จะเขียนไปที่ **`os.Stderr`** (ไม่ใช่ `os.Stdout` แบบ `fmt.Println`) ซึ่งเป็นธรรมเนียมที่ถูกต้องสำหรับ log message — เพราะ stdout ควรสงวนไว้สำหรับ "ผลลัพธ์จริง" ของโปรแกรม ส่วน stderr ไว้สำหรับข้อความ diagnostic/log (เชื่อมโยงกับที่เราแยก `cmd.Stdout`/`cmd.Stderr` ในบทที่ 052)

---

## 3. `log.Fatal` vs `log.Panic` vs การ return error ปกติ

`log` package มีฟังก์ชันตระกูล `Fatal*` และ `Panic*` ที่ทำงานคล้าย `Print*` (จัดรูปแบบและพิมพ์ข้อความ log) แต่มีพฤติกรรมเพิ่มเติมที่ **ต่างกันโดยสิ้นเชิง** และมือใหม่มักสับสน:

| ฟังก์ชัน | พฤติกรรมหลังพิมพ์ log | เทียบเท่ากับ |
|---|---|---|
| `log.Println`/`log.Printf` | ทำงานต่อตามปกติ | `fmt.Println` + timestamp |
| `log.Fatal`/`log.Fatalf`/`log.Fatalln` | เรียก **`os.Exit(1)`** ทันที | จบโปรแกรมทันที **โดยไม่รัน `defer` ใดๆ เลย** |
| `log.Panic`/`log.Panicf`/`log.Panicln` | เรียก **`panic(...)`** | เหมือน `panic` ปกติ — `defer` ยังทำงาน และ `recover()` ดักจับได้ |

```go
package main

import "log"

func riskyOperation() error {
	return nil // สมมติว่าสำเร็จ
}

func main() {
	defer log.Println("this defer will NOT run if log.Fatal is called above it")

	if err := riskyOperation(); err != nil {
		log.Fatalf("operation failed: %v", err) // จบโปรแกรมทันที os.Exit(1)
	}

	log.Println("operation succeeded")
}
```

### กฎในการเลือกใช้

> **`log.Fatal` เหมาะกับข้อผิดพลาดที่ทำให้โปรแกรม "ทำงานต่อไม่ได้อย่างมีความหมาย" ตั้งแต่ตอน startup** เช่น อ่านไฟล์ config ไม่ได้, เชื่อมต่อ database ไม่ได้ตอนเริ่มโปรแกรม — เพราะ `os.Exit(1)` ข้าม `defer` ทั้งหมด จึงไม่เหมาะใช้กลางโปรแกรมที่มี resource ต้อง cleanup (เช่น connection, file handle ที่เปิดค้างไว้) เนื่องจาก `defer` ที่ควรจะปิด resource เหล่านั้นจะไม่ถูกเรียกเลย
>
> **`log.Panic` เหมาะกับสถานการณ์ที่ต้องการให้ `defer`/`recover` ทำงานตามปกติ** แต่ยังต้อง log ข้อความไว้ก่อน panic — ใช้น้อยกว่า `log.Fatal` มากในทางปฏิบัติ
>
> **ในโค้ดส่วนใหญ่ของโปรแกรม ควร `return error` ตามปกติ** (ตามหลักการ error handling ที่เรียนไปในบทที่ 015-016) แล้วให้ผู้เรียกตัดสินใจว่าจะจัดการอย่างไร — `log.Fatal`/`log.Panic` ควรสงวนไว้ใช้เฉพาะที่จุดเริ่มต้นของโปรแกรม (`main()` หรือ `init()`) เท่านั้น ไม่ควรใช้กระจัดกระจายอยู่ใน business logic เพราะทำให้ทดสอบยากและควบคุม flow การจัดการ error ไม่ได้

---

## 4. สร้าง Logger กำหนดเองด้วย `log.New`

ฟังก์ชันระดับ package อย่าง `log.Println` ใช้ **logger เริ่มต้น (default logger)** ของ package `log` ซึ่งเขียนไป `os.Stderr` ด้วย flag มาตรฐาน แต่บ่อยครั้งเราต้องการ logger ที่กำหนดปลายทาง, prefix, และรูปแบบ timestamp เอง — ทำได้ผ่าน `log.New`:

```go
func New(out io.Writer, prefix string, flag int) *Logger
```

```go
package main

import (
	"log"
	"os"
)

func main() {
	// สร้าง logger เขียนไป stdout พร้อม prefix และ flag กำหนดเอง
	customLogger := log.New(os.Stdout, "[APP] ", log.Ldate|log.Ltime|log.Lshortfile)
	customLogger.Println("custom logger message")

	warnLogger := log.New(os.Stdout, "WARN: ", log.LstdFlags)
	warnLogger.Println("disk usage above 80%")
}
```

ผลลัพธ์:

```
[APP] 2026/09/26 02:49:11 log_demo.go:13: custom logger message
WARN: 2026/09/26 02:49:11 disk usage above 80%
```

### Flag ที่ใช้บ่อยของ package `log`

| Flag | ความหมาย |
|---|---|
| `log.Ldate` | แสดงวันที่ (`2026/09/26`) |
| `log.Ltime` | แสดงเวลา (`02:49:11`) |
| `log.Lmicroseconds` | แสดงเวลาละเอียดถึงไมโครวินาที |
| `log.Lshortfile` | แสดงชื่อไฟล์+เลขบรรทัดแบบสั้น (`log_demo.go:13`) |
| `log.Llongfile` | แสดง path เต็มของไฟล์+เลขบรรทัด |
| `log.LstdFlags` | รวม `Ldate \| Ltime` (ค่า default ของ logger มาตรฐาน) |

การสร้าง logger แยกตัวแบบนี้มีประโยชน์เมื่อต้องการแยก log แต่ละหมวดหมู่ (เช่น logger สำหรับ audit log แยกจาก logger สำหรับ debug log) หรือต้องการส่ง log ไปยังไฟล์เฉพาะ (ส่ง `*os.File` ที่เปิดด้วย `os.OpenFile` เป็น `out` แทน `os.Stdout`)

### ปรับ default logger ของ package `log` เองได้เช่นกัน

นอกจากสร้าง logger ใหม่ด้วย `log.New` แล้ว เรายังปรับพฤติกรรมของ **default logger** (ตัวที่ `log.Println`/`log.Printf` เรียกใช้อยู่เบื้องหลัง) ได้โดยตรงผ่าน `log.SetOutput`, `log.SetPrefix`, และ `log.SetFlags`:

```go
package main

import (
	"bytes"
	"fmt"
	"log"
	"os"
)

func main() {
	var buf bytes.Buffer

	log.SetOutput(&buf) // เปลี่ยนปลายทางของ default logger จาก stderr ไปที่ buffer
	log.SetPrefix("[GLOBAL] ")
	log.SetFlags(log.Lshortfile)

	log.Println("this goes into buf, not stderr")
	fmt.Print("captured: ", buf.String())

	// คืนค่ากลับไปเป็นค่ามาตรฐานเมื่อใช้เสร็จ
	log.SetOutput(os.Stderr)
	log.SetFlags(log.LstdFlags)
	log.SetPrefix("")
	log.Println("back to normal stderr output")
}
```

ผลลัพธ์:

```
captured: [GLOBAL] log_setoutput_demo.go:17: this goes into buf, not stderr
2026/09/26 03:12:53 back to normal stderr output
```

ฟังก์ชันกลุ่มนี้มีประโยชน์มากเวลาเขียน **unit test** ที่ต้องการตรวจสอบว่าโค้ดที่เรียก `log.Println` ภายในเขียนข้อความอะไรออกมาบ้าง (redirect ไปที่ `bytes.Buffer` แล้วตรวจสอบเนื้อหาที่ capture ได้) แทนที่จะปล่อยให้ log ไหลไปที่ stderr จริงระหว่างรัน test ตามที่จะเรียนเทคนิคการทดสอบเจาะลึกกว่านี้ในภาคที่ 7 (Testing ขั้นสูง)

---

## 5. ข้อจำกัดของ `log` แบบดั้งเดิม

แม้ `log` package จะดีกว่า `fmt.Println` มาก แต่ยังมีข้อจำกัดสำคัญที่ทำให้ไม่เหมาะกับระบบ production สมัยใหม่ที่ต้องการวิเคราะห์ log จำนวนมหาศาลด้วยเครื่องมืออัตโนมัติ:

1. **ไม่มีแนวคิดเรื่อง log level ในตัว** — ไม่มี `Debug`/`Info`/`Warn`/`Error` แยกกัน ต้องคิด prefix เองแบบ manual (เหมือนตัวอย่าง `WARN:` ข้างบน) ทำให้ไม่มีมาตรฐานร่วมกันและกรอง log ตามระดับความสำคัญไม่ได้ง่ายๆ
2. **ข้อความเป็น string อิสระ (unstructured)** — ข้อความอย่าง `"user=alice action=login"` เป็นแค่ text ธรรมดา ระบบภายนอกที่ต้องการค้นหา/กรอง log ตาม field เช่น `user` หรือ `action` ต้องมานั่ง parse string เอาเอง ซึ่งเปราะบางและช้า
3. **ไม่มี key-value pair ที่เป็นมาตรฐาน** — แต่ละคนเขียน format ข้อความ log ต่างกันไปเรื่อยๆ ทำให้ log จากส่วนต่างๆ ของระบบไม่มีรูปแบบเดียวกัน

ปัญหาเหล่านี้นำไปสู่การเพิ่ม package ใหม่เข้ามาใน standard library ตั้งแต่ Go 1.21 — **`log/slog`** — ที่แก้ปัญหาทั้งหมดนี้อย่างเป็นทางการ

---

## 6. `log/slog`: Structured Logging มาตรฐานของ Go (1.21+)

**`log/slog`** (structured log) คือ package logging รุ่นใหม่ที่เพิ่มเข้ามาใน Go 1.21 (ปี 2023) และถือเป็น**แนวทางที่แนะนำในปัจจุบัน**สำหรับโปรเจกต์ที่เริ่มต้นใหม่ แนวคิดหลักคือ **log ควรมีโครงสร้างเป็น key-value pair** แทนที่จะเป็น string อิสระ ทำให้เครื่องมือภายนอกอ่านและประมวลผลได้ง่ายและแม่นยำ

```go
package main

import (
	"log/slog"
)

func main() {
	slog.Info("server starting", "port", 8080, "env", "development")
	slog.Warn("cache miss", "key", "user:42")
	slog.Error("failed to connect to db", "error", "connection refused", "retry", 1)
}
```

ผลลัพธ์ (default handler ของ `slog` คือ text format เขียนไป stderr):

```
time=2026-09-26T02:49:04.360Z level=INFO msg="server starting" port=8080 env=development
time=2026-09-26T02:49:04.360Z level=WARN msg="cache miss" key=user:42
time=2026-09-26T02:49:04.360Z level=ERROR msg="failed to connect to db" error="connection refused" retry=1
```

สังเกตความแตกต่างจาก `log.Println` อย่างชัดเจน: แต่ละบรรทัดมี **`time`, `level`, `msg`** เป็นมาตรฐานเสมอ ตามด้วย **key-value pair** ที่เราส่งเข้าไป (`port=8080`, `env=development` ฯลฯ) — รูปแบบนี้ทำให้เครื่องมือ log aggregator วิเคราะห์และค้นหาได้ง่ายกว่า string อิสระมาก

### วิธีส่ง key-value: แบบ variadic กับแบบ `slog.Attr`

ตัวอย่างข้างบนใช้รูปแบบ variadic ธรรมดา (`"port", 8080`) ซึ่งใช้งานง่ายแต่ compiler ตรวจสอบ type ไม่ได้ (เป็น `...any`) `slog` ยังมีรูปแบบที่ type-safe กว่าโดยใช้ฟังก์ชันสร้าง attribute เช่น `slog.String`, `slog.Int`, `slog.Bool`:

```go
slog.Info("handling request",
	slog.String("method", "GET"),
	slog.String("path", "/api/orders"),
	slog.Int("status", 200),
)
```

รูปแบบนี้แนะนำสำหรับโค้ดที่ต้องการความชัดเจนและปลอดภัยด้าน type มากกว่า โดยเฉพาะใน library หรือโค้ดที่เรียกใช้บ่อยมาก (performance ดีกว่าเล็กน้อยเพราะไม่ต้องผ่าน reflection ตรวจ type ตอน runtime)

---

## 7. Log Levels ใน `slog`: Debug, Info, Warn, Error

`slog` มี log level มาตรฐาน 4 ระดับเรียงจากน้อยไปมาก:

| Level | ใช้เมื่อไร |
|---|---|
| `slog.LevelDebug` | ข้อมูลละเอียดสำหรับ debug เท่านั้น ไม่ควรเปิดใน production ปกติ |
| `slog.LevelInfo` | เหตุการณ์ปกติที่น่าสนใจ (server start, request handled) — ค่า default |
| `slog.LevelWarn` | สถานการณ์ผิดปกติแต่โปรแกรมยังทำงานต่อได้ |
| `slog.LevelError` | ข้อผิดพลาดที่ต้องให้ความสนใจ |

แต่ละ level มีฟังก์ชันเรียกตรงคู่กัน: `slog.Debug`, `slog.Info`, `slog.Warn`, `slog.Error` — เมื่อกำหนด minimum level ไว้ที่ handler (เช่น `Info`) ข้อความระดับต่ำกว่านั้น (`Debug`) จะถูกกรองทิ้งไปโดยอัตโนมัติ **ไม่ถูกประมวลผลหรือเขียนออกไปเลย** ทำให้ควบคุมปริมาณ log และ performance ได้ดีในระบบ production:

```go
package main

import (
	"log/slog"
	"os"
)

func main() {
	logger := slog.New(slog.NewTextHandler(os.Stdout, &slog.HandlerOptions{
		Level: slog.LevelWarn, // แสดงเฉพาะ Warn ขึ้นไป
	}))

	logger.Debug("this will NOT be printed")
	logger.Info("this will NOT be printed either")
	logger.Warn("this WILL be printed")
	logger.Error("this WILL be printed")
}
```

---

## 8. `slog.NewTextHandler` vs `slog.NewJSONHandler`

`slog` แยก "การสร้างข้อความ log" ออกจาก "การจัดรูปแบบผลลัพธ์" โดยใช้แนวคิด **Handler** — เราสร้าง `*slog.Logger` โดยส่ง handler ที่ต้องการเข้าไป มี handler มาตรฐานสองแบบให้ใช้ทันที:

```go
package main

import (
	"log/slog"
	"os"
)

func main() {
	// Text format: อ่านง่ายด้วยตา เหมาะกับตอน develop ในเครื่อง
	textLogger := slog.New(slog.NewTextHandler(os.Stdout, &slog.HandlerOptions{Level: slog.LevelDebug}))
	textLogger.Info("server starting", "port", 8080, "env", "development")

	// JSON format: เหมาะกับ production ที่ต้องส่ง log เข้าระบบ aggregator
	jsonLogger := slog.New(slog.NewJSONHandler(os.Stdout, nil))
	jsonLogger.Info("server starting", "port", 8080, "env", "development")
}
```

ผลลัพธ์:

```
time=2026-09-26T02:49:04.360Z level=INFO msg="server starting" port=8080 env=development
{"time":"2026-09-26T02:49:04.360837351Z","level":"INFO","msg":"server starting","port":8080,"env":"development"}
```

### เลือกใช้แบบไหนเมื่อไร

| Handler | เหมาะกับ |
|---|---|
| `slog.NewTextHandler` | Development ในเครื่อง — อ่านง่ายด้วยตาเปล่าบน terminal |
| `slog.NewJSONHandler` | Production — ส่งเข้าระบบ log aggregator เช่น Elasticsearch, Loki, Datadog ที่ parse JSON ได้โดยตรงและ query ตาม field ได้ทันที |

`slog.HandlerOptions` ยังปรับแต่งได้อีกหลายอย่าง เช่น `AddSource: true` เพื่อแนบตำแหน่งไฟล์/บรรทัดที่เรียก log เข้าไปด้วย หรือ `ReplaceAttr` สำหรับปรับแต่ง/ปกปิด field บางตัวก่อนเขียนออกจริง (เช่น ปกปิดรหัสผ่านหรือ token ไม่ให้หลุดไปอยู่ใน log)

---

## 9. ตั้งค่า Global Default Logger

เช่นเดียวกับ package `log` ที่มี default logger ระดับ package ให้เรียกตรงๆ (`log.Println`) `slog` ก็มีแนวคิดเดียวกันผ่าน `slog.SetDefault` — เมื่อเรียก `slog.SetDefault(logger)` แล้ว ฟังก์ชันระดับ package อย่าง `slog.Info`, `slog.Error` ทั้งหมดจะใช้ logger ตัวนี้แทน logger เริ่มต้นของระบบ

```go
package main

import (
	"log/slog"
	"os"
)

func main() {
	jsonLogger := slog.New(slog.NewJSONHandler(os.Stdout, nil))
	slog.SetDefault(jsonLogger)

	// หลังจากนี้ ฟังก์ชันระดับ package ทั้งหมดจะใช้ jsonLogger โดยอัตโนมัติ
	slog.Info("using default logger now", "component", "main")
}
```

ผลลัพธ์:

```
{"time":"2026-09-26T02:49:04.360837351Z","level":"INFO","msg":"using default logger now","component":"main"}
```

รูปแบบทั่วไปในโปรเจกต์จริงคือเรียก `slog.SetDefault(...)` หนึ่งครั้งตอนเริ่มโปรแกรมใน `main()` โดยตั้งค่า handler ตาม environment (เช่น text handler ตอน develop, JSON handler ตอน production ผ่านการเช็ค environment variable) จากนั้นทุกส่วนของโค้ดเรียก `slog.Info`/`slog.Error` ได้เลยโดยไม่ต้องส่ง logger ผ่านทุก function

---

## 10. Contextual Logging ด้วย `slog.Group` และ `context.Context`

### จัดกลุ่ม field ที่เกี่ยวข้องกันด้วย `slog.Group`

เมื่อ log message มี field จำนวนมากที่แบ่งเป็นหมวดหมู่ได้ (เช่น field ของ request แยกจาก field ของ user) `slog.Group` ช่วยจัดกลุ่มให้ชัดเจนขึ้นทั้งในแง่การอ่านและโครงสร้าง JSON ที่ได้:

```go
slog.Info("handling request",
	slog.String("user_id", "user-42"),
	slog.Group("request",
		slog.String("method", "GET"),
		slog.String("path", "/api/orders"),
	),
)
```

ผลลัพธ์แบบ JSON จะซ้อน field ภายใต้ `request` เป็น object ย่อย:

```json
{"time":"...","level":"INFO","msg":"handling request","user_id":"user-42","request":{"method":"GET","path":"/api/orders"}}
```

### ผูก Logger เข้ากับ `context.Context`

ในระบบ web/API จริง เรามักต้องการให้ log ทุกบรรทัดที่เกิดขึ้นระหว่างจัดการ request เดียวกัน มี field ร่วมกัน เช่น `request_id` เพื่อให้ไล่ดู log ของ request นั้นทั้งหมดได้ง่าย — เทคนิคที่นิยมคือผูก logger (ที่มี field ติดตัวไว้แล้วผ่าน `.With(...)`) เข้ากับ `context.Context` (จากบทที่ 032) แล้วส่งต่อ context ไปตาม function call chain:

```go
package main

import (
	"context"
	"log/slog"
	"os"
)

type ctxKey string

const loggerKey ctxKey = "logger"

// withLogger ผูก logger เข้ากับ context เพื่อส่งต่อไปตาม call chain
func withLogger(ctx context.Context, logger *slog.Logger) context.Context {
	return context.WithValue(ctx, loggerKey, logger)
}

// loggerFromContext ดึง logger กลับออกมาจาก context
// ถ้าไม่เจอ (เช่น context ไม่ได้ผูก logger มา) จะ fallback ไปใช้ default logger แทน
func loggerFromContext(ctx context.Context) *slog.Logger {
	if l, ok := ctx.Value(loggerKey).(*slog.Logger); ok {
		return l
	}
	return slog.Default()
}

func handleRequest(ctx context.Context, userID string) {
	logger := loggerFromContext(ctx)
	logger.Info("handling request",
		slog.String("user_id", userID),
		slog.Group("request",
			slog.String("method", "GET"),
			slog.String("path", "/api/orders"),
		),
	)
}

func main() {
	jsonLogger := slog.New(slog.NewJSONHandler(os.Stdout, nil))

	// .With(...) คืน logger ตัวใหม่ที่แนบ field นี้ติดไปทุกข้อความ log ที่เรียกต่อจากนี้
	requestLogger := jsonLogger.With(slog.String("request_id", "req-123"))

	ctx := withLogger(context.Background(), requestLogger)
	handleRequest(ctx, "user-42")
}
```

ผลลัพธ์:

```json
{"time":"...","level":"INFO","msg":"handling request","request_id":"req-123","user_id":"user-42","request":{"method":"GET","path":"/api/orders"}}
```

สังเกตว่า `request_id` ติดมาโดยอัตโนมัติแม้จะไม่ได้เขียนซ้ำใน `handleRequest` เพราะถูกแนบไว้ล่วงหน้าผ่าน `.With(...)` ก่อนใส่ลงใน context — เทคนิคนี้ทำให้ทุกฟังก์ชันที่รับ `context.Context` เข้าไป (ซึ่งเป็นธรรมเนียมมาตรฐานของ Go ตามที่เรียนไปในบทที่ 032) สามารถดึง logger ที่มี field เชื่อมโยงกับ request นั้นๆ มาใช้ได้ทันที โดยไม่ต้องส่ง logger เป็น parameter แยกต่างหากทุกครั้ง

---

## 11. ทำไม Structured JSON Logs ถึงสำคัญกับระบบ Production

ในระบบขนาดเล็กที่รันบนเครื่องเดียว การอ่าน log จากไฟล์ text ธรรมดาด้วยตาเปล่าอาจเพียงพอ แต่ระบบ production จริงในปัจจุบันมักประกอบด้วย **หลาย service หลาย instance** ที่รันพร้อมกันเป็นจำนวนมาก (โดยเฉพาะระบบ microservices ที่รันบน Kubernetes) การไล่ดู log แบบ text ทีละไฟล์ทีละเครื่องเป็นไปไม่ได้ในทางปฏิบัติ

นี่คือเหตุผลที่ระบบ production สมัยใหม่ใช้ **log aggregator** (เช่น Elasticsearch/Loki/Datadog/CloudWatch Logs) รวบรวม log จากทุก service เข้าไว้ที่เดียว แล้วให้ทีมค้นหา/กรอง/สร้างกราฟจาก log เหล่านั้น — เครื่องมือเหล่านี้ทำงานได้ดีที่สุดเมื่อ log มาในรูปแบบ **JSON ที่มีโครงสร้างชัดเจน** เพราะสามารถ:

- **ค้นหาตาม field ได้แม่นยำ** เช่น `level:ERROR AND user_id:"user-42"` แทนการค้นแบบ text ธรรมดาที่เดายาก
- **สร้างกราฟ/dashboard อัตโนมัติ** เช่น จำนวน error ต่อนาที แยกตาม service, response time เฉลี่ยแยกตาม endpoint
- **เชื่อมโยง log ข้าม service** ด้วย field ร่วม เช่น `request_id`/`trace_id` เพื่อไล่ดู request เดียวที่เดินทางผ่านหลาย microservice
- **ตั้ง alert อัตโนมัติ** เมื่อ pattern บางอย่างเกิดขึ้น เช่น error rate เกิน threshold ที่กำหนด

การออกแบบ log ให้เป็น structured JSON ตั้งแต่ต้น (ด้วย `slog.NewJSONHandler`) จึงไม่ใช่แค่เรื่องความสวยงาม แต่เป็น**การเตรียมความพร้อมให้ระบบ observability ทำงานได้เต็มประสิทธิภาพ** — เนื้อหาเรื่อง observability แบบเต็มรูปแบบ รวมถึงการเชื่อมต่อกับ Prometheus, Grafana และ OpenTelemetry จะเรียนแบบเจาะลึกใน **ภาคที่ 9 (Part 099: Monitoring และ Observability)**

---

## สรุปสิ่งที่ได้เรียนในบทนี้

- `fmt.Println` ไม่ใช่เครื่องมือ logging เพราะไม่มี timestamp, log level, หรือโครงสร้างข้อมูลที่เป็นมาตรฐาน
- `log.Println`/`log.Printf` แนบ timestamp มาให้อัตโนมัติและเขียนไป `os.Stderr` โดย default
- `log.Fatal` เรียก `os.Exit(1)` ทันที **ไม่รัน `defer`** ส่วน `log.Panic` เรียก `panic()` ที่ `defer`/`recover` ยังทำงานตามปกติ — ทั้งสองควรใช้เฉพาะจุดเริ่มต้นโปรแกรม ไม่ใช่กระจายอยู่ทั่ว business logic
- `log.New(out, prefix, flag)` สร้าง logger กำหนดเองได้ ทั้งปลายทาง, prefix, และ flag ควบคุมรูปแบบ (`Ldate`, `Ltime`, `Lshortfile` ฯลฯ)
- **`log/slog`** (Go 1.21+) คือ structured logging มาตรฐานที่แนะนำสำหรับโปรเจกต์ใหม่ ให้ log เป็น key-value pair แทน string อิสระ
- `slog` มี 4 log level (`Debug`, `Info`, `Warn`, `Error`) กรองข้อความที่ต่ำกว่า minimum level ที่กำหนดได้อัตโนมัติ
- `slog.NewTextHandler` เหมาะกับ development, `slog.NewJSONHandler` เหมาะกับ production ที่ต้องส่งเข้า log aggregator
- `slog.SetDefault(logger)` ตั้ง global default logger ให้ฟังก์ชันระดับ package (`slog.Info` ฯลฯ) ใช้ได้ทันที
- `slog.Group` จัดกลุ่ม field ที่เกี่ยวข้องกัน และการผูก logger เข้ากับ `context.Context` ทำให้ส่งต่อ field ร่วม (เช่น `request_id`) ไปตาม call chain ได้โดยไม่ต้องส่ง logger เป็น parameter แยก
- Structured JSON logs สำคัญมากกับระบบ production ที่มีหลาย service เพราะทำให้ log aggregator ค้นหา/วิเคราะห์/ตั้ง alert ได้แม่นยำ — จะเรียนเจาะลึกต่อในภาคที่ 9 เรื่อง Observability

## แบบฝึกหัดท้ายบท

1. เขียนโปรแกรมที่สร้าง logger กำหนดเองด้วย `log.New` เขียน log ลงไฟล์ (`app.log`) แทนหน้าจอ พร้อม prefix `[MYAPP] ` และ flag `log.LstdFlags | log.Lshortfile`
2. เขียนฟังก์ชัน `divide(a, b int) (int, error)` ที่ return error แทนการใช้ `log.Fatal` เมื่อ `b == 0` แล้วเขียน `main()` ที่เรียกใช้และ log ผลลัพธ์ด้วย `slog.Error` เมื่อเกิด error — อธิบายว่าทำไมวิธีนี้ดีกว่าการใช้ `log.Fatal` ใน `divide` โดยตรง
3. เขียนโปรแกรมที่สร้าง `*slog.Logger` สองตัว: ตัวหนึ่งใช้ `slog.NewTextHandler` อีกตัวใช้ `slog.NewJSONHandler` ทั้งคู่ตั้ง `Level: slog.LevelDebug` แล้ว log ข้อความระดับต่างๆ (`Debug`, `Info`, `Warn`, `Error`) ด้วยทั้งสอง logger เปรียบเทียบผลลัพธ์ที่ได้
4. สร้างฟังก์ชัน middleware สมมติ `withRequestID(ctx context.Context) context.Context` ที่สุ่ม request ID (ใช้ `crypto/rand` จากบทที่ 051) แล้วผูก logger ที่มี field `request_id` นั้นเข้ากับ context ตามรูปแบบในหัวข้อที่ 10 ทดสอบเรียกฟังก์ชันจำลองสองสามชั้นที่รับ context ต่อกันไป และดึง logger ออกมาใช้ log ในแต่ละชั้น
5. ใช้ `slog.HandlerOptions{ReplaceAttr: ...}` เขียนโปรแกรมที่ปกปิดค่าของ field ชื่อ `password` ไม่ให้ปรากฏใน log จริง (แทนที่ด้วย `"***"`) แล้วทดสอบ log ข้อมูลที่มี field นี้ปนอยู่
6. ค้นคว้าเพิ่มเติมเกี่ยวกับการเขียน custom `slog.Handler` ของตัวเอง (implement interface `slog.Handler`) เพื่อส่ง log ไปยังปลายทางพิเศษ เช่น ส่งเข้า message queue หรือ third-party logging service โดยตรง

---

**ต่อไป**: [Part 055 — แพ็กเกจ `runtime` และ Garbage Collector](./055-runtime-and-gc.md)
