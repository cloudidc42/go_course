# Part 064: Web Framework: Fiber

> ภาคที่ 5: Web Development — ตอนที่ 9 จาก 15 (Part 56–70)

## สารบัญของบทนี้

1. ทบทวน: Fiber ต่างจาก Gin และ Echo ตรงไหนตั้งแต่รากฐาน
2. `fasthttp` คืออะไร ทำไม Fiber ถึงเลือกใช้แทน `net/http`
3. ติดตั้ง Fiber
4. Hello Fiber: โปรแกรมแรกและ `fiber.New()`
5. Routing และ Context API: `c.Params`, `c.Query`, `c.BodyParser`, `c.JSON`
6. Built-in Middleware: `logger`, `recover`, `cors`
7. บั๊กยอดฮิต: Pooled `*fiber.Ctx` และการอ้างอิงข้ามอายุของ request
8. สาธิตบั๊กจริง: โปรแกรม crash เพราะ retain context ข้าม goroutine
9. วิธีแก้ที่ถูกต้อง: Copy ค่าออกมาทันที และ `fiber.Config{Immutable: true}`
10. ตัวอย่างเต็ม: Todo CRUD API ด้วย Fiber
11. เมื่อไหร่ควรใช้ Fiber เมื่อไหร่ควรใช้ Gin/Echo/chi
12. สรุปสิ่งที่ได้เรียนในบทนี้
13. แบบฝึกหัดท้ายบท

---

## 1. ทบทวน: Fiber ต่างจาก Gin และ Echo ตรงไหนตั้งแต่รากฐาน

ใน **Part 063** ตารางสรุปท้ายบทได้ทิ้งปมสำคัญไว้หนึ่งจุด: **Gin และ Echo สร้างอยู่บน `net/http` เหมือนกัน แต่ [Fiber](https://gofiber.io/) เลือกสร้างอยู่บน `fasthttp` แทน** — นี่ไม่ใช่ความต่างเล็กๆ น้อยๆ ระดับ API แต่เป็นความต่างตั้งแต่**รากฐานของ HTTP stack ทั้งหมด** ที่ส่งผลกระทบต่อวิธีเขียนโค้ดของเราโดยตรง โดยเฉพาะเรื่องที่จะเป็นหัวใจของบทนี้: **context object ที่ถูกนำกลับมาใช้ซ้ำ (pooled)**

บทนี้จะพาไปดูทั้งข้อดี (ความเร็ว) และข้อเสีย (ความเข้ากันไม่ได้กับ `net/http` และบั๊กที่มาจาก pooling) ของการตัดสินใจออกแบบนี้อย่างตรงไปตรงมา เพื่อให้ตัดสินใจได้เองว่าโปรเจกต์แบบไหนควรเลือก Fiber และแบบไหนไม่ควร

---

## 2. `fasthttp` คืออะไร ทำไม Fiber ถึงเลือกใช้แทน `net/http`

[`fasthttp`](https://github.com/valyala/fasthttp) เป็น HTTP server/client implementation ที่เขียนขึ้นมาใหม่ทั้งหมดโดยไม่ใช้ `net/http` ของ standard library เลย เป้าหมายเดียวของมันคือ **performance สูงสุดสำหรับกรณีใช้งานเฉพาะ** (high-load API server) โดยแลกกับความเข้ากันได้ในวงกว้าง

### เทคนิคหลักที่ทำให้ `fasthttp` เร็วกว่า `net/http`

1. **Object pooling อย่างหนัก**: `net/http` สร้าง `*http.Request` และ `http.ResponseWriter` ใหม่ทุกครั้งที่มี request เข้ามา (ต้อง allocate memory ใหม่ทุกรอบ ให้ Garbage Collector จัดการทีหลัง) ส่วน `fasthttp` ใช้ `sync.Pool` เก็บ `*fasthttp.RequestCtx` ไว้ใช้ซ้ำ — เมื่อ request หนึ่งจบลง object เดิมจะถูกล้างค่า (reset) แล้วนำไปให้ request ถัดไปใช้ต่อ **แทนที่จะสร้างใหม่** ลด GC pressure ได้มากในสถานการณ์ throughput สูง
2. **ลด memory allocation ในการ parse header/body**: ใช้ byte slice และ unsafe string conversion (`unsafe.Pointer`) เพื่อเลี่ยงการ copy ข้อมูลโดยไม่จำเป็น
3. **Connection handling ที่ optimize เฉพาะทาง**: ออกแบบ event loop และ buffer management ใหม่ทั้งหมดโดยไม่ผูกกับ interface ทั่วไปแบบ `io.Reader`/`io.Writer` ของ `net/http`

ผลลัพธ์คือ benchmark ที่ทีม `fasthttp` และ Fiber เผยแพร่มักแสดงตัวเลข throughput สูงกว่า `net/http` ธรรมดาหลายเท่าตัวในสถานการณ์ที่มี concurrent connection จำนวนมากและ payload เล็ก (เช่น echo server, JSON API ง่ายๆ)

### ราคาที่ต้องจ่าย: ความเข้ากันไม่ได้กับ `net/http`

นี่คือจุดที่สำคัญที่สุดที่ต้องเข้าใจก่อนตัดสินใจใช้ Fiber ในโปรเจกต์จริง:

> **`fasthttp` ไม่ implement interface ของ `net/http`** — `fasthttp.RequestCtx` ไม่ใช่ `http.ResponseWriter` และไม่มี `*http.Request` ให้ใช้ตรงๆ

ผลกระทบที่ตามมาโดยตรง:

- **Middleware ของ `net/http` ใช้กับ Fiber ไม่ได้** — middleware ทุกตัวที่เขียนในรูปแบบ `func(http.Handler) http.Handler` (มาตรฐานของระบบนิเวศ Go ทั้งหมด รวมถึงที่ใช้กับ `gorilla/mux`/`chi` ใน **Part 058**) ใช้กับ Fiber ไม่ได้เลยโดยตรง ต้องหา middleware เวอร์ชันที่เขียนสำหรับ Fiber โดยเฉพาะ หรือเขียนเอง
- **Library ที่รับ `http.Handler`/`*http.Request` ใช้ตรงๆ ไม่ได้** — เช่น `net/http/pprof`, บาง instrumentation library ของ observability tools, หรือ `httptest.NewRequest` ที่คุ้นเคย (แม้ Fiber จะมี `app.Test()` เป็นทางเลือกของตัวเอง)
- **Ecosystem middleware ต้องเป็นของ Fiber โดยเฉพาะ** — โชคดีที่ Fiber มี middleware ในตัวให้ครบสำหรับงานทั่วไป (หัวข้อ 6) และมี third-party มากพอสมควร แต่เทียบกับจำนวน middleware ที่เขียนสำหรับ `net/http` (ซึ่งใช้ได้กับทั้ง Gin, Echo, chi, gorilla/mux) แล้วยังน้อยกว่ามาก

พูดสั้นๆ: **Fiber ไม่ได้ "ห่อ" `net/http` แบบ Gin/Echo แต่ "แทนที่" มันทั้งหมด** การเลือก Fiber คือการเลือกออกจากระบบนิเวศ `net/http` ทั้งระบบเพื่อแลกกับความเร็ว

---

## 3. ติดตั้ง Fiber

```bash
mkdir fiber-todo-api
cd fiber-todo-api
go mod init fiber-todo-api
go get github.com/gofiber/fiber/v2
```

ผลลัพธ์จริงจากการรัน (Go 1.24.7):

```
go: added github.com/andybalholm/brotli v1.1.0
go: added github.com/gofiber/fiber/v2 v2.52.15
go: added github.com/google/uuid v1.6.0
go: added github.com/klauspost/compress v1.17.9
go: added github.com/mattn/go-colorable v0.1.13
go: added github.com/mattn/go-isatty v0.0.20
go: added github.com/mattn/go-runewidth v0.0.16
go: added github.com/rivo/uniseg v0.2.0
go: added github.com/valyala/bytebufferpool v1.0.0
go: added github.com/valyala/fasthttp v1.51.0
go: added github.com/valyala/tcplisten v1.0.0
go: added golang.org/x/sys v0.28.0
```

สังเกตบรรทัด `github.com/valyala/fasthttp v1.51.0` — นี่คือหลักฐานตรงๆ ว่า Fiber ดึง `fasthttp` มาเป็น dependency หลัก (ไม่ใช่ `net/http`) ต่างจากตอนติดตั้ง Gin (Part 061) หรือ Echo (Part 063) ที่ไม่มี dependency ตัวไหนมาแทนที่ `net/http` เลย

`go.mod` ที่ได้ยังคงอยู่ที่ Go 1.24.7 ตามเดิม — Fiber v2 เข้ากันได้กับ Go เวอร์ชันค่อนข้างกว้าง (ต่างจาก Echo/validator เวอร์ชันล่าสุดที่เจอใน Part 063 ที่ต้องการ Go 1.25+)

---

## 4. Hello Fiber: โปรแกรมแรกและ `fiber.New()`

```go
package main

import "github.com/gofiber/fiber/v2"

func main() {
	app := fiber.New()

	app.Get("/ping", func(c *fiber.Ctx) error {
		return c.JSON(fiber.Map{
			"message": "pong",
		})
	})

	app.Listen(":8080")
}
```

จุดที่ควรสังเกต:

- **`fiber.New()`** สร้าง `*fiber.App` — ไม่มี middleware ติดมาอัตโนมัติเหมือนกับ `echo.New()` ใน Part 063
- **`app.Get(...)`** ใช้ตัวพิมพ์เล็ก (`Get`, `Post`) ต่างจาก Gin/Echo ที่ใช้ตัวพิมพ์ใหญ่ทั้งหมด (`GET`, `POST`) — เป็นรายละเอียดเล็กๆ ที่สร้างความสับสนให้คนย้ายมาจาก Gin/Echo บ่อยที่สุด
- **Handler signature `func(c *fiber.Ctx) error`** — เหมือนกับ Echo เป๊ะ (`echo.Context` เป็น interface ส่วน `*fiber.Ctx` เป็น concrete pointer แบบเดียวกับ Gin) บังคับ return error เหมือนกัน
- **`c.JSON(obj)`** — สังเกตว่า **ไม่มี status code เป็น argument แรก** ต่างจาก Gin (`c.JSON(code, obj)`) และ Echo (`c.JSON(code, obj)`) ถ้าต้องการกำหนด status code ต้องเรียก `c.Status(code)` ต่อท้ายก่อน (ดูหัวข้อ 5)
- **`app.Listen(":8080")`** เทียบเท่ากับ `r.Run(":8080")` ของ Gin และ `e.Start(":8080")` ของ Echo

รันด้วย `go run main.go` ผลลัพธ์ตอน start (ของจริงจากการรัน):

```
 ┌───────────────────────────────────────────────────┐
 │                  Fiber v2.52.15                   │
 │               http://127.0.0.1:8080               │
 │       (bound on host 0.0.0.0 and port 8080)       │
 │                                                   │
 │ Handlers ............ 2  Processes ........... 1 │
 │ Prefork ....... Disabled  PID ............. 11175 │
 └───────────────────────────────────────────────────┘
```

Fiber มี startup banner ที่สวยงามเป็นกล่องกรอบ พร้อมแสดงจำนวน handler ที่ลงทะเบียนไว้ทั้งหมด และสถานะ `Prefork` (ฟีเจอร์ขั้นสูงสำหรับ scale การใช้ CPU หลาย core ที่ไม่ได้ครอบคลุมในบทนี้) ทดสอบด้วย curl:

```bash
curl http://localhost:8080/ping
```
```json
{"message":"pong"}
```

---

## 5. Routing และ Context API: `c.Params`, `c.Query`, `c.BodyParser`, `c.JSON`

`*fiber.Ctx` มี method ที่หน้าตาคล้าย Gin/Echo มาก แต่มีจุดต่างในรายละเอียดที่ต้องระวัง:

| งาน | Gin | Echo | Fiber |
|---|---|---|---|
| อ่าน path parameter | `c.Param("id")` | `c.Param("id")` | `c.Params("id")` (**มี s ต่อท้าย**) |
| อ่าน query string | `c.Query("q")` | `c.QueryParam("q")` | `c.Query("q")` |
| Decode JSON body | `c.ShouldBindJSON(&obj)` | `c.Bind(&obj)` | `c.BodyParser(&obj)` |
| เขียน JSON response | `c.JSON(code, obj)` | `c.JSON(code, obj)` | `c.JSON(obj)` (ไม่มี code — ใช้ `c.Status(code).JSON(obj)` แทน) |
| กำหนด status code แยก | `c.Status(code)` | `c.NoContent(code)` | `c.Status(code)` (คืน `*fiber.Ctx` ให้ chain ต่อได้) |
| อ่าน header | `c.GetHeader("X-Key")` | `c.Request().Header.Get("X-Key")` | `c.Get("X-Key")` |

ตัวอย่างการอ่าน path parameter และ query string:

```go
app.Get("/users/:id", func(c *fiber.Ctx) error {
	id := c.Params("id")               // string เสมอ (path parameter)
	return c.JSON(fiber.Map{"id": id})
})

app.Get("/search", func(c *fiber.Ctx) error {
	q := c.Query("q")                  // "" ถ้าไม่มี
	page := c.Query("page", "1")       // argument ตัวที่สองคือ default value (ในตัวเดียวกัน)
	return c.JSON(fiber.Map{"query": q, "page": page})
})
```

สังเกตว่า **`c.Query("page", "1")` มี default value ในตัว** เป็น variadic argument ตัวที่สอง — สะดวกกว่า Gin ที่ต้องเรียก `c.DefaultQuery(...)` แยกชื่อ method แต่ผลลัพธ์เหมือนกัน

### `c.BodyParser` — decode request body

```go
type CreateUserRequest struct {
	Name  string `json:"name"`
	Email string `json:"email"`
}

app.Post("/users", func(c *fiber.Ctx) error {
	var req CreateUserRequest
	if err := c.BodyParser(&req); err != nil {
		return c.Status(fiber.StatusBadRequest).JSON(fiber.Map{"error": err.Error()})
	}
	return c.Status(fiber.StatusCreated).JSON(req)
})
```

`c.BodyParser` คล้าย `c.Bind` ของ Echo — auto-detect `Content-Type` แล้วเลือก decoder ให้เอง (รองรับ JSON, XML, form) **แต่ไม่มี validation ใดๆ ในตัวเลย** เช่นเดียวกับ Echo ต้อง validate เองแยกต่างหาก (Fiber ไม่มี struct tag แบบ `binding:"..."` ของ Gin หรือ interface `Validator` แบบ Echo มาให้ — ถ้าต้องการ validation แบบมี tag ต้องติดตั้ง `go-playground/validator` เองแล้วเรียก `validator.New().Struct(req)` ตรงๆ ในทุก handler หรือเขียน helper function ห่อเอาไว้)

---

## 6. Built-in Middleware: `logger`, `recover`, `cors`

Fiber จัด middleware ไว้เป็น sub-package ย่อยตามชื่อ คล้ายแนวทางของ Echo แต่ชื่อ import เป็นตัวพิมพ์เล็กทั้งหมด:

```go
import (
	"github.com/gofiber/fiber/v2"
	"github.com/gofiber/fiber/v2/middleware/cors"
	"github.com/gofiber/fiber/v2/middleware/logger"
	"github.com/gofiber/fiber/v2/middleware/recover"
)

func main() {
	app := fiber.New()

	app.Use(logger.New())  // เทียบเท่า middleware.Logger() ของ Echo
	app.Use(recover.New()) // เทียบเท่า middleware.Recover() ของ Echo — ดัก panic
	app.Use(cors.New())    // เทียบเท่า middleware.CORS() ของ Echo

	// ... routes
}
```

| Middleware package | หน้าที่ | เทียบเท่าใน Echo |
|---|---|---|
| `middleware/logger` | log ทุก request แบบ plain text (ค่า default) | `middleware.Logger()` |
| `middleware/recover` | ดัก panic แปลงเป็น HTTP 500 | `middleware.Recover()` |
| `middleware/cors` | ใส่ CORS header | `middleware.CORS()` |

ตัวอย่าง log จริงจาก `logger.New()` (รูปแบบ plain text ต่อบรรทัด ต่างจาก Echo ที่เป็น JSON โดย default):

```
03:16:55 | 200 |      96.632µs | 127.0.0.1 | GET | /api/v1/todos | -
03:16:55 | 201 |      50.893µs | 127.0.0.1 | POST | /api/v1/todos | -
03:16:55 | 400 |      28.159µs | 127.0.0.1 | POST | /api/v1/todos | -
```

**ข้อควรระวังที่สำคัญที่สุด**: `recover.New()` ของ Fiber ดัก panic ได้เฉพาะใน **goroutine เดียวกับที่ Fiber เรียก handler เท่านั้น** — ถ้า handler เปิด goroutine ใหม่แล้ว panic เกิดขึ้นข้างในนั้น `recover.New()` จะดักไม่ได้เลย และจะทำให้ **ทั้งโปรเซสล่มทันที** (ข้อจำกัดนี้จริงๆ แล้วใช้กับ `gin.Recovery()` และ `middleware.Recover()` ของ Echo เหมือนกันทุกตัว เพราะ `recover()` ของภาษา Go ทำงานเฉพาะใน goroutine ที่เกิด panic เท่านั้น ตามที่เรียนไปใน **Part 017**) นี่เป็นสาเหตุหนึ่งที่การเปิด goroutine ใน handler ต้องระวังเป็นพิเศษ — และเป็นจุดเชื่อมโยงไปสู่หัวข้อถัดไปที่เป็นบั๊กจริงจังกว่านั้นอีกขั้น

---

## 7. บั๊กยอดฮิต: Pooled `*fiber.Ctx` และการอ้างอิงข้ามอายุของ request

นี่คือหัวใจสำคัญที่สุดของบทนี้ ย้อนกลับไปหัวข้อ 2: `fasthttp` ใช้ `sync.Pool` เก็บ `*fasthttp.RequestCtx` ไว้ใช้ซ้ำข้าม request เพื่อลด memory allocation — และ **`*fiber.Ctx` ก็ถูก pool ในลักษณะเดียวกัน** เมื่อ handler ทำงานเสร็จและ Fiber ส่ง response กลับไปแล้ว `*fiber.Ctx` ตัวนั้น (พร้อมข้อมูลข้างในทั้งหมด: path parameters, query string, body ที่ parse แล้ว ฯลฯ) จะถูก **reset แล้วนำไปใช้กับ request ถัดไปที่เข้ามา** ซึ่งอาจเป็น request จาก client คนละคนโดยสิ้นเชิง

นี่หมายความว่า **`*fiber.Ctx` ไม่ใช่ข้อมูลที่ "เป็นของ" request นั้นตลอดไป — มันแค่ยืมมาใช้ชั่วคราวระหว่างที่ handler กำลังทำงานเท่านั้น**

### กฎเหล็กที่ต้องจำ

> **ห้ามเก็บอ้างอิง (reference) ถึง `*fiber.Ctx` หรือค่าที่ได้จากมัน (เช่น จาก `c.Params()`, `c.Query()`, `c.Body()`) ไว้ใช้ "หลังจาก" handler return แล้วเด็ดขาด** — ไม่ว่าจะเป็นการส่งเข้า goroutine ที่ทำงานต่อ, เก็บใน channel, หรือเก็บใน struct ที่มีอายุยืนกว่า request

ปัญหานี้**ไม่เกิดกับ Gin หรือ Echo** เพราะทั้งคู่สร้างอยู่บน `net/http` ซึ่ง `*http.Request` เป็น object ใหม่ทุกครั้ง ไม่ได้ถูก pool กลับมาใช้ซ้ำ (`sync.Pool` ของ `net/http` เก็บแค่ internal buffer บางส่วน ไม่ใช่ตัว `*http.Request`/`ResponseWriter` ทั้งก้อนที่ handler เห็น) นี่คือหนึ่งใน**ผลข้างเคียงโดยตรง**ของการเลือกใช้ `fasthttp` เพื่อความเร็ว ที่นักพัฒนาที่ย้ายจาก Gin/Echo มา Fiber มักไม่รู้ตัวจนกว่าจะเจอบั๊กที่ debug ยากมากใน production (เพราะบั๊กนี้**ไม่เกิดทุกครั้ง** ขึ้นอยู่กับจังหวะที่ pool นำ `Ctx` ตัวเดิมไปใช้ซ้ำ ทำให้ทดสอบตอน dev บนเครื่องเดียวมักไม่เจอ แต่ไปเจอตอนมี traffic สูงใน production)

รูปแบบโค้ดที่นำไปสู่บั๊กนี้บ่อยที่สุดในโค้ดจริง:

```go
app.Post("/orders/:id", func(c *fiber.Ctx) error {
	go func() {
		// อันตราย: sendConfirmationEmail ทำงานหลัง handler return ไปแล้ว
		// ตอนนั้น c อาจถูกนำไปใช้กับ request อื่นแล้ว
		sendConfirmationEmail(c.Params("id"))
	}()
	return c.SendString("order received")
})
```

โค้ดแบบนี้ **compile ผ่านปกติ ไม่มี error หรือ warning ใดๆ** และในเครื่อง dev ที่ทดสอบทีละ request มักทำงานถูกต้องเสมอ เพราะกว่า goroutine จะเริ่มทำงาน `Ctx` ตัวเดิมก็ยังไม่ทันถูกเอาไปใช้ซ้ำ — แต่ในสถานการณ์ที่มี concurrent request จำนวนมาก (ตรงกับสถานการณ์ที่คนเลือก Fiber เพราะต้องการ throughput สูงพอดี) โอกาสที่ `Ctx` จะถูกรีไซเคิลก่อน goroutine จะอ่านค่ามันจะสูงขึ้นมาก

---

## 8. สาธิตบั๊กจริง: โปรแกรม crash เพราะ retain context ข้าม goroutine

มาดูโค้ดที่สาธิตบั๊กนี้ให้เกิดขึ้นจริง (ไม่ใช่แค่ทฤษฎี) โดยจำลองสถานการณ์ concurrent request ด้วย `app.Test()` (เครื่องมือทดสอบในตัวของ Fiber ที่ยิง request ผ่าน `*fiber.App` เดียวกันโดยไม่ต้องเปิด real TCP listener — ใช้ pooled `Ctx` ชุดเดียวกับตอนรันจริงทุกประการ):

```go
package main

import (
	"fmt"
	"io"
	"net/http"
	"net/http/httptest"
	"sync"
	"time"

	"github.com/gofiber/fiber/v2"
)

func main() {
	app := fiber.New()

	var mu sync.Mutex
	var results []string
	var wg sync.WaitGroup

	app.Get("/order/:id", func(c *fiber.Ctx) error {
		wg.Add(1)
		go func() {
			defer wg.Done()
			time.Sleep(50 * time.Millisecond) // จำลอง async work: ส่งอีเมล, เขียน audit log ฯลฯ
			// บั๊ก: c คือ *fiber.Ctx ที่ถูก pool ไว้ใช้ซ้ำ พอ goroutine นี้ตื่นมาทำงาน
			// เซิร์ฟเวอร์อาจ reset/นำ c ไปใช้กับ request อื่นที่ไม่เกี่ยวข้องไปแล้ว
			id := c.Params("id")
			mu.Lock()
			results = append(results, fmt.Sprintf("async goroutine read id=%q", id))
			mu.Unlock()
		}()
		return c.SendString("queued " + c.Params("id"))
	})

	fmt.Println("--- request #1: id=ORIGINAL (ctx ของมันถูก leak เข้า goroutine) ---")
	req1 := httptest.NewRequest(http.MethodGet, "/order/ORIGINAL", nil)
	resp1, _ := app.Test(req1, -1)
	body1, _ := io.ReadAll(resp1.Body)
	fmt.Println("response:", string(body1))

	fmt.Println("--- ระหว่างนั้น มี 30 request อื่นเข้ามาใช้ ctx pool ชุดเดียวกัน ---")
	for i := 2; i <= 30; i++ {
		req := httptest.NewRequest(http.MethodGet, fmt.Sprintf("/order/OTHER-%d", i), nil)
		resp, _ := app.Test(req, -1)
		io.Copy(io.Discard, resp.Body)
	}

	wg.Wait()
	fmt.Println("--- สิ่งที่ goroutine ที่ leak เห็นจริงๆ ---")
	mu.Lock()
	for _, r := range results {
		fmt.Println(r)
	}
	mu.Unlock()
}
```

รันด้วย `go run buggy.go` — ผลลัพธ์จริงจากการรัน:

```
--- request #1: id=ORIGINAL (ctx ของมันถูก leak เข้า goroutine) ---
response: queued ORIGINAL
--- ระหว่างนั้น มี 30 request อื่นเข้ามาใช้ ctx pool ชุดเดียวกัน ---
panic: runtime error: invalid memory address or nil pointer dereference
[signal SIGSEGV: segmentation violation code=0x1 addr=0x98 pc=0x5df538]

goroutine 11 [running]:
github.com/gofiber/fiber/v2.(*Ctx).Params(0xc000170008, {0x69b051?, 0x0?}, {0x0, 0x0, 0x0?})
	/root/go/pkg/mod/github.com/gofiber/fiber/v2@v2.52.15/ctx.go:1072 +0x78
main.main.func1.1()
	/tmp/.../buggy.go:29 +0x9b
created by main.main.func1 in goroutine 10
	/tmp/.../buggy.go:23 +0xae
exit status 2
```

**โปรแกรม crash จริง — ทั้งโปรเซสล่มทันที** ไม่ใช่แค่ request เดียวที่ error สาเหตุคือพอ 30 request ที่ตามมาถูกประมวลผลไปเรื่อยๆ `*fiber.Ctx` ตัวเดิมที่ request แรก (`ORIGINAL`) เคยใช้ ถูก reset แล้วเอาไปใช้กับ route อื่นซ้ำแล้วซ้ำอีก จนถึงจุดหนึ่ง field ภายใน (`c.route`) ที่ `c.Params()` ต้องใช้กลายเป็น `nil` พอดีในจังหวะที่ goroutine ตื่นขึ้นมาอ่านค่า ทำให้เกิด nil pointer dereference — **ยิ่งเป็นหลักฐานที่ชัดเจนว่า `*fiber.Ctx` ไม่ใช่ของที่ปลอดภัยจะถือไว้ข้ามขอบเขตของ handler เลย**

> **ทำไม `recover.New()` ในหัวข้อ 6 ไม่ช่วยตรงนี้**: เพราะ panic เกิดใน goroutine ที่แยกออกไปจาก goroutine ที่ Fiber เรียก handler โดยตรง (ตามที่อธิบายไว้ท้ายหัวข้อ 6) — `recover()` middleware ดักได้เฉพาะ panic ที่เกิดในเส้นทางการเรียก handler ปกติเท่านั้น

---

## 9. วิธีแก้ที่ถูกต้อง: Copy ค่าออกมาทันที และ `fiber.Config{Immutable: true}`

### ความเข้าใจผิดที่พบบ่อย: "แค่ copy ค่าลง string ตัวแปรก็พอ"

หลายคนคิดว่าการแก้บั๊กนี้ทำได้ง่ายๆ แค่ดึงค่าออกมาเก็บใน local variable **ก่อน** เปิด goroutine:

```go
app.Get("/order/:id", func(c *fiber.Ctx) error {
	id := c.Params("id") // ดูเหมือนจะปลอดภัยแล้ว...

	go func() {
		time.Sleep(50 * time.Millisecond)
		// ใช้ id (string ธรรมดา) แทน c.Params("id")
		fmt.Println(id)
	}()
	return c.SendString("queued " + id)
})
```

**นี่ยังไม่ปลอดภัยพอ!** เมื่อทดสอบโค้ดนี้จริงด้วยสถานการณ์เดียวกับหัวข้อ 8 (ไม่ crash แล้ว แต่ผลลัพธ์ยังผิด):

```
async goroutine read id="OTHER-30"
async goroutine read id="OTHER-3"
async goroutine read id="OTHER-3"
async goroutine read id="OTHER-30"
...
```

ไม่มีค่า `"ORIGINAL"` ปรากฏเลยแม้แต่ครั้งเดียว! สาเหตุคือ `fasthttp`/Fiber ใช้เทคนิค **unsafe string conversion** เพื่อเลี่ยงการ allocate memory: string ที่ `c.Params()` คืนกลับมาไม่ได้ชี้ไปยัง memory ของตัวเองที่ปลอดภัย แต่ชี้ไปยัง**byte buffer ภายในของ `Ctx` ที่ยังคงถูกเขียนทับต่อไปเรื่อยๆ เมื่อ `Ctx` ถูกนำไปใช้กับ request ถัดไป** การเขียน `id := c.Params("id")` เป็นแค่การ copy "string header" (ตัวชี้ + ความยาว) ไม่ได้ copy เนื้อข้อมูลจริงๆ ข้างใน — ซึ่งขัดกับสัญชาตญาณปกติที่ string ใน Go ควรจะ immutable เสมอ (เพราะโดยปกติมันเป็นแบบนั้นจริง ยกเว้นกรณีที่ library ใช้ `unsafe` แบบนี้เพื่อความเร็ว)

### วิธีแก้ที่ถูกต้อง #1: Deep copy ด้วย `utils.CopyString` / `utils.CopyBytes`

Fiber ให้ helper function มาสำหรับ **บังคับ copy เนื้อข้อมูลจริงๆ** ไปไว้ใน memory ใหม่ที่เป็นของตัวเองล้วนๆ ผ่าน package `github.com/gofiber/fiber/v2/utils`:

```go
import "github.com/gofiber/fiber/v2/utils"

app.Get("/order/:id", func(c *fiber.Ctx) error {
	// FIX: utils.CopyString allocate byte array ใหม่แล้ว copy ข้อมูลจริงลงไป
	// เป็น deep copy ที่แท้จริง ไม่ใช่แค่ copy string header เหมือนก่อนหน้า
	id := utils.CopyString(c.Params("id"))

	go func() {
		time.Sleep(50 * time.Millisecond)
		fmt.Println(id) // ปลอดภัยแล้ว: id ไม่ได้อ้างอิง memory ของ Ctx อีกต่อไป
	}()
	return c.SendString("queued " + id)
})
```

ทดสอบด้วยสถานการณ์เดียวกันทุกประการ (30 request แทรกเข้ามาระหว่างที่ goroutine หลับอยู่) — ผลลัพธ์จริงจากการรัน:

```
--- สิ่งที่ goroutine เห็นจริงๆ (หลังแก้) ---
async goroutine read id="OTHER-2"
async goroutine read id="OTHER-5"
async goroutine read id="OTHER-3"
...
async goroutine read id="ORIGINAL"
...
async goroutine read id="OTHER-23"
```

ครบ 30 ค่า ตรงกับ request ที่ยิงเข้ามาทุกตัวแบบไม่มีตกหล่นหรือสลับกัน — รวมถึง `"ORIGINAL"` ที่ปรากฏถูกต้องด้วย พิสูจน์ว่า `utils.CopyString` แก้ปัญหาได้จริง

สำหรับ `[]byte` (เช่นจาก `c.Body()`) ใช้ `utils.CopyBytes` ในรูปแบบเดียวกัน:

```go
bodyCopy := utils.CopyBytes(c.Body())
go processAsync(bodyCopy)
```

### วิธีแก้ที่ถูกต้อง #2: `fiber.Config{Immutable: true}` — ปลอดภัยทั้งแอปในบรรทัดเดียว

ถ้าแอปทั้งตัวมีจุดที่ต้อง retain ค่าข้ามอายุ request บ่อยๆ (async job, WebSocket broadcast, logging queue ฯลฯ) การไล่ใส่ `utils.CopyString`/`utils.CopyBytes` ทุกจุดเสี่ยงพลาดได้ Fiber จึงมี config ระดับแอปที่บังคับให้ **ทุก method ที่คืน string/[]byte จาก `Ctx` (เช่น `Params`, `Query`, `Body`) ทำการ copy ให้อัตโนมัติเสมอ**:

```go
app := fiber.New(fiber.Config{Immutable: true})
```

ทดสอบสถานการณ์เดียวกันอีกครั้ง โดยกลับไปใช้โค้ดแบบง่ายที่สุด (`id := c.Params("id")` ธรรมดา ไม่เรียก `utils.CopyString` เอง) แต่เปิด `Immutable: true` ไว้ที่ระดับแอป — ผลลัพธ์จริง:

```
async goroutine read id="OTHER-10"
async goroutine read id="OTHER-5"
...
async goroutine read id="ORIGINAL"
...
async goroutine read id="OTHER-25"
```

ถูกต้องครบทุกค่าเช่นเดียวกัน โดยไม่ต้องแก้โค้ด handler แม้แต่บรรทัดเดียว **ข้อแลกเปลี่ยน**: `Immutable: true` ทำให้ทุก request เสีย allocation เพิ่มขึ้นเล็กน้อยเพื่อ copy ข้อมูล (แม้ใน handler ส่วนใหญ่ที่ไม่ได้ retain ค่าข้ามอายุ request เลยก็ตาม) จึงลดข้อได้เปรียบด้าน performance ของ Fiber ลงไปบ้าง — เอกสารทางการของ Fiber แนะนำให้เปิด flag นี้เฉพาะเมื่อจำเป็นจริงๆ (เช่น แอปที่มี async pattern กระจายอยู่ทั่วไป) มากกว่าเปิดเป็นค่าเริ่มต้นของทุกโปรเจกต์

### สรุปเป็นกฎจำง่าย

| สถานการณ์ | ทำอย่างไร |
|---|---|
| ใช้ค่าจาก `Ctx` แค่ภายใน handler เอง (ไม่ retain ข้ามไปที่ไหน) | ใช้ได้ปกติ ไม่มีปัญหา |
| ต้องส่งค่าจาก `Ctx` เข้า goroutine/channel/struct ที่มีอายุยืนกว่า handler | ต้อง `utils.CopyString`/`utils.CopyBytes` เสมอ หรือแปลงเป็น type อื่นที่ copy ค่าจริง (เช่น `strconv.Atoi` แปลง string เป็น int ก็ปลอดภัยเพราะ int ไม่อ้างอิง memory เดิม) |
| แอปทั้งตัวมี pattern แบบ retain ค่าอยู่ทั่วไป | พิจารณาเปิด `fiber.Config{Immutable: true}` ทั้งแอป แลกกับ allocation ที่เพิ่มขึ้นเล็กน้อย |
| งาน async ต้องทำจริงจัง ไม่ใช่แค่ log สั้นๆ | พิจารณา copy ค่าที่ต้องใช้ทั้งหมดลงใน struct ของตัวเองตั้งแต่ต้น แล้วส่ง struct นั้น (ไม่ใช่ `Ctx`) เข้า queue/goroutine — เป็นแนวทางที่ปลอดภัยและอ่านง่ายที่สุด |

---

## 10. ตัวอย่างเต็ม: Todo CRUD API ด้วย Fiber

มาสร้าง Todo CRUD API เดียวกับ Part 061 (Gin) และ Part 063 (Echo) ด้วย Fiber โครงสร้างโปรเจกต์:

```
fiber-todo-api/
├── go.mod
├── go.sum
└── main.go
```

```go
package main

import (
	"errors"
	"strconv"
	"sync"

	"github.com/gofiber/fiber/v2"
	"github.com/gofiber/fiber/v2/middleware/cors"
	"github.com/gofiber/fiber/v2/middleware/logger"
	"github.com/gofiber/fiber/v2/middleware/recover"
)

// ---------- Model ----------

type Todo struct {
	ID    int    `json:"id"`
	Title string `json:"title"`
	Done  bool   `json:"done"`
}

// ---------- In-memory store ----------

type TodoStore struct {
	mu     sync.Mutex
	nextID int
	items  map[int]Todo
}

func NewTodoStore() *TodoStore {
	return &TodoStore{nextID: 1, items: make(map[int]Todo)}
}

var ErrNotFound = errors.New("todo not found")

func (s *TodoStore) List() []Todo {
	s.mu.Lock()
	defer s.mu.Unlock()
	list := make([]Todo, 0, len(s.items))
	for _, t := range s.items {
		list = append(list, t)
	}
	return list
}

func (s *TodoStore) Get(id int) (Todo, error) {
	s.mu.Lock()
	defer s.mu.Unlock()
	t, ok := s.items[id]
	if !ok {
		return Todo{}, ErrNotFound
	}
	return t, nil
}

func (s *TodoStore) Create(t Todo) Todo {
	s.mu.Lock()
	defer s.mu.Unlock()
	t.ID = s.nextID
	s.nextID++
	s.items[t.ID] = t
	return t
}

func (s *TodoStore) Update(id int, t Todo) (Todo, error) {
	s.mu.Lock()
	defer s.mu.Unlock()
	if _, ok := s.items[id]; !ok {
		return Todo{}, ErrNotFound
	}
	t.ID = id
	s.items[id] = t
	return t, nil
}

func (s *TodoStore) Delete(id int) error {
	s.mu.Lock()
	defer s.mu.Unlock()
	if _, ok := s.items[id]; !ok {
		return ErrNotFound
	}
	delete(s.items, id)
	return nil
}

func main() {
	app := fiber.New()

	app.Use(logger.New())
	app.Use(recover.New())
	app.Use(cors.New())

	store := NewTodoStore()
	store.Create(Todo{Title: "เรียน Go", Done: false})
	store.Create(Todo{Title: "เขียน Fiber CRUD", Done: false})

	api := app.Group("/api/v1")
	todos := api.Group("/todos")

	todos.Get("", func(c *fiber.Ctx) error {
		return c.JSON(store.List())
	})

	todos.Get("/:id", func(c *fiber.Ctx) error {
		id, err := strconv.Atoi(c.Params("id"))
		if err != nil {
			return c.Status(fiber.StatusBadRequest).JSON(fiber.Map{"error": "invalid id"})
		}
		t, err := store.Get(id)
		if err != nil {
			return c.Status(fiber.StatusNotFound).JSON(fiber.Map{"error": err.Error()})
		}
		return c.JSON(t)
	})

	todos.Post("", func(c *fiber.Ctx) error {
		var t Todo
		if err := c.BodyParser(&t); err != nil {
			return c.Status(fiber.StatusBadRequest).JSON(fiber.Map{"error": err.Error()})
		}
		if t.Title == "" {
			return c.Status(fiber.StatusBadRequest).JSON(fiber.Map{"error": "title is required"})
		}
		created := store.Create(t)
		return c.Status(fiber.StatusCreated).JSON(created)
	})

	todos.Put("/:id", func(c *fiber.Ctx) error {
		id, err := strconv.Atoi(c.Params("id"))
		if err != nil {
			return c.Status(fiber.StatusBadRequest).JSON(fiber.Map{"error": "invalid id"})
		}
		var t Todo
		if err := c.BodyParser(&t); err != nil {
			return c.Status(fiber.StatusBadRequest).JSON(fiber.Map{"error": err.Error()})
		}
		updated, err := store.Update(id, t)
		if err != nil {
			return c.Status(fiber.StatusNotFound).JSON(fiber.Map{"error": err.Error()})
		}
		return c.JSON(updated)
	})

	todos.Delete("/:id", func(c *fiber.Ctx) error {
		id, err := strconv.Atoi(c.Params("id"))
		if err != nil {
			return c.Status(fiber.StatusBadRequest).JSON(fiber.Map{"error": "invalid id"})
		}
		if err := store.Delete(id); err != nil {
			return c.Status(fiber.StatusNotFound).JSON(fiber.Map{"error": err.Error()})
		}
		return c.SendStatus(fiber.StatusNoContent)
	})

	app.Get("/search", func(c *fiber.Ctx) error {
		q := c.Query("q")
		return c.JSON(fiber.Map{"query": q})
	})

	app.Listen(":8080")
}
```

### ทดสอบจริงทุก endpoint ด้วย `curl` (ผลลัพธ์ของจริงจากการรัน)

```bash
curl http://localhost:8080/api/v1/todos
```
```json
[{"id":1,"title":"เรียน Go","done":false},{"id":2,"title":"เขียน Fiber CRUD","done":false}]
```

```bash
curl -X POST http://localhost:8080/api/v1/todos \
  -H "Content-Type: application/json" \
  -d '{"title":"ทดสอบ Fiber","done":false}'
```
```json
{"id":3,"title":"ทดสอบ Fiber","done":false}
```

```bash
curl http://localhost:8080/api/v1/todos/1
```
```json
{"id":1,"title":"เรียน Go","done":false}
```

```bash
curl -X PUT http://localhost:8080/api/v1/todos/1 \
  -H "Content-Type: application/json" \
  -d '{"title":"เรียน Go (แก้ไข)","done":true}'
```
```json
{"id":1,"title":"เรียน Go (แก้ไข)","done":true}
```

```bash
curl -X DELETE http://localhost:8080/api/v1/todos/2 -o /dev/null -w "%{http_code}\n"
# 204
```

```bash
curl "http://localhost:8080/search?q=go"
```
```json
{"query":"go"}
```

ทดสอบกรณีไม่ส่ง `title` มา (ไม่มี struct tag validation แบบ Gin/Echo — ต้องเช็คเองด้วยมือดังที่เขียนไว้ใน handler):

```bash
curl -X POST http://localhost:8080/api/v1/todos \
  -H "Content-Type: application/json" -d '{"done":false}'
```
```json
{"error":"title is required"}
```

log ฝั่งเซิร์ฟเวอร์จาก `logger.New()` (plain text):

```
03:16:55 | 200 |      96.632µs | 127.0.0.1 | GET | /api/v1/todos | -
03:16:55 | 201 |      50.893µs | 127.0.0.1 | POST | /api/v1/todos | -
03:16:55 | 200 |      15.918µs | 127.0.0.1 | GET | /api/v1/todos/1 | -
03:16:55 | 200 |      38.554µs | 127.0.0.1 | PUT | /api/v1/todos/1 | -
03:16:55 | 204 |      10.277µs | 127.0.0.1 | DELETE | /api/v1/todos/2 | -
03:16:55 | 200 |      29.233µs | 127.0.0.1 | GET | /search | -
03:16:55 | 400 |      28.159µs | 127.0.0.1 | POST | /api/v1/todos | -
```

พฤติกรรมของ API ต่อผู้ใช้เหมือนกับเวอร์ชัน Gin (Part 061) และ Echo (Part 063) ทุกประการ — สิ่งที่ต่างคือรากฐานเบื้องหลัง (`fasthttp` แทน `net/http`) และวิธีเขียนโค้ด ไม่ใช่ผลลัพธ์ที่ client เห็น

---

## 11. เมื่อไหร่ควรใช้ Fiber เมื่อไหร่ควรใช้ Gin/Echo/chi

หลังเห็นทั้งข้อดี (ความเร็ว) และข้อเสีย (ความเข้ากันไม่ได้กับ `net/http`, บั๊กเรื่อง pooled context) มาสรุปแนวทางตัดสินใจแบบตรงไปตรงมา:

### เลือก Fiber เมื่อ...

- **Throughput คือปัจจัยตัดสินอันดับหนึ่งจริงๆ** — เช่น API gateway, service ที่ต้องรับ request หลักหมื่นถึงแสน request ต่อวินาที, edge service ที่ latency ทุก microsecond มีผลต่อ SLA
- **ทีมมีความเข้าใจเรื่อง `fasthttp`/pooled context อย่างถ่องแท้** และมีวินัยในการ code review ที่จับบั๊กแบบหัวข้อ 7-9 ได้ก่อนขึ้น production
- **ไม่ได้พึ่งพา middleware/library ของ `net/http` ecosystem มากนัก** — ระบบเขียนขึ้นใหม่ทั้งหมด ไม่ต้อง integrate กับของเดิมที่ผูกกับ `net/http`
- ต้องการ **syntax ที่คุ้นเคยกับ Express.js** (Fiber ได้แรงบันดาลใจการออกแบบ API มาจาก Express.js ของ Node.js โดยตรง เหมาะกับทีมที่มีพื้นฐาน Node.js มาก่อน)

### เลือก Gin, Echo หรือ chi (Part 058) เมื่อ...

- **ต้องการความเข้ากันได้กับ `net/http` ecosystem** — ใช้ middleware มาตรฐาน, instrumentation library, `net/http/pprof`, หรือ integrate กับโค้ดเดิมที่เขียนด้วย `net/http` ได้ทันที
- **ทีมส่วนใหญ่ยังไม่คุ้นเคยกับข้อจำกัดของ pooled context** — ความเสี่ยงเรื่องบั๊กที่ debug ยาก (หัวข้อ 7-9) มีน้ำหนักมากกว่าประโยชน์ด้าน throughput ที่ได้มา โดยเฉพาะกับทีมขนาดเล็กหรือ mid-level ที่หมุนเวียนคนบ่อย
- **Bottleneck จริงของระบบไม่ได้อยู่ที่ routing layer** — งานส่วนใหญ่ (มากกว่า 90% ของโปรเจกต์ CRUD API ทั่วไป) คอขวดจริงอยู่ที่ database query, external API call, หรือ business logic ไม่ใช่ที่ HTTP router เร็วหรือช้า ทำให้ความต่างของ throughput ระหว่าง Gin/Echo กับ Fiber แทบไม่ปรากฏผลในการวัดจริง
- **ต้องการ ecosystem ที่ใหญ่และเสถียรที่สุด** — จำนวน middleware, ตัวอย่างโค้ด, และ Stack Overflow answer ของ Gin ยังคงมากกว่า Fiber อย่างมีนัยสำคัญ

### คำแนะนำสำหรับคนส่วนใหญ่

สำหรับ**ทีมส่วนใหญ่และโปรเจกต์ส่วนใหญ่** — โดยเฉพาะทีมที่เพิ่งเริ่มต้นกับ Go หรือทีมขนาดกลางที่ต้องการความเสถียรและง่ายต่อการ maintain ระยะยาว — **Gin หรือ Echo คือตัวเลือกที่ปลอดภัยกว่า** เพราะอยู่ในโลกของ `net/http` ที่คุ้นเคยและมี safety net มากกว่า Fiber เหมาะกับสถานการณ์เฉพาะทางที่วัดผลมาแล้วจริงๆ ว่า throughput ของ routing layer เป็นคอขวดจริง ไม่ใช่ทางเลือกเริ่มต้นสำหรับทุกโปรเจกต์

---

## สรุปสิ่งที่ได้เรียนในบทนี้

- Fiber สร้างอยู่บน `fasthttp` ไม่ใช่ `net/http` — ทำให้เร็วกว่าในหลาย benchmark แต่แลกกับความเข้ากันไม่ได้กับ middleware/library ของ `net/http` ecosystem ทั้งหมด
- ติดตั้งด้วย `go get github.com/gofiber/fiber/v2` — สังเกต dependency `valyala/fasthttp` เป็นหลักฐานของความต่างเชิงรากฐาน
- Context API ของ Fiber (`c.Params`, `c.Query`, `c.BodyParser`, `c.JSON`) คล้าย Gin/Echo แต่มีรายละเอียดปลีกย่อยต่างกัน (`Params` มี s, `JSON` ไม่มี status code parameter)
- Built-in middleware: `logger.New()`, `recover.New()`, `cors.New()` — import จาก sub-package `middleware/*` เฉพาะของ Fiber เท่านั้น ใช้กับ middleware ของ `net/http` ไม่ได้
- **บั๊กสำคัญที่สุดของบทนี้**: `*fiber.Ctx` ถูก pool ไว้ใช้ซ้ำข้าม request ห้าม retain มันหรือค่าที่ได้จากมันไว้ใช้หลัง handler return แล้วเด็ดขาด (เช่น ส่งเข้า goroutine) — พิสูจน์ด้วยโปรแกรมจริงที่ panic เพราะ nil pointer dereference
- แค่ copy string header (`id := c.Params("id")`) **ไม่พอ** เพราะ Fiber ใช้ unsafe string ที่อ้างอิง buffer ที่ถูกเขียนทับได้ — ต้อง deep copy ด้วย `utils.CopyString`/`utils.CopyBytes` จริงๆ หรือเปิด `fiber.Config{Immutable: true}` ทั้งแอป
- Fiber เหมาะกับงานที่ต้องการ throughput สูงสุดจริงๆ และทีมที่เข้าใจข้อจำกัดของมันดี ส่วนงานทั่วไปที่ bottleneck อยู่ที่ database/business logic Gin หรือ Echo ยังคงเป็นตัวเลือกที่ปลอดภัยและ maintain ง่ายกว่า

## แบบฝึกหัดท้ายบท

1. สร้างโปรเจกต์ Fiber ใหม่ ติดตั้งและรันตัวอย่าง `/ping` ให้สำเร็จ แล้วลองเปลี่ยน `c.JSON(fiber.Map{...})` เป็น `c.Status(fiber.StatusCreated).JSON(fiber.Map{...})` สังเกต status code ที่เปลี่ยนไปด้วย `curl -i`
2. คัดลอกโค้ดสาธิตบั๊กในหัวข้อ 8 มารันเองบนเครื่อง (`go run buggy.go`) แล้วลองปรับจำนวน "request อื่น" จาก 30 เป็น 5 หรือ 100 สังเกตว่าบั๊ก (panic) ยังเกิดขึ้นเสมอหรือไม่ อธิบายว่าทำไมจำนวน request ที่แทรกเข้ามาถึงมีผลต่อโอกาสเกิดบั๊ก
3. แก้โค้ดในข้อ 2 ด้วยวิธี `utils.CopyString` ตามหัวข้อ 9 แล้วรันซ้ำ ยืนยันว่าไม่ crash และค่าที่ goroutine อ่านได้ถูกต้องครบทุกตัว
4. เพิ่ม validation ให้ Todo CRUD ในหัวข้อ 10 ด้วย `go-playground/validator` เอง (ติดตั้งเพิ่ม แล้วเรียก `validator.New().Struct(&t)` ใน handler `POST`/`PUT`) เพื่อจำลองว่าถ้าอยากได้ validation แบบมี tag ใน Fiber ต้องประกอบเองอย่างไร
5. เขียน endpoint `GET /api/v1/todos/:id/toggle` ที่สลับค่า `Done` เหมือนแบบฝึกหัดข้อ 2 ของ Part 061 แต่ทำด้วย Fiber แทน
6. อธิบายด้วยคำพูดของตัวเองว่าทำไมการเลือก Fiber ถึงเป็น "การเลือกออกจากระบบนิเวศ `net/http` ทั้งระบบ" ไม่ใช่แค่ "เปลี่ยน framework" ธรรมดา และยกตัวอย่างสถานการณ์จริงหนึ่งกรณีที่คุณคิดว่าคุ้มค่าที่จะแลก กับอีกหนึ่งกรณีที่ไม่คุ้มค่า

---

**ต่อไป**: [Part 065 — Templates ด้วย `html/template`](./065-html-template.md)
