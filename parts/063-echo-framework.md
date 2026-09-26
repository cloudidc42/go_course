# Part 063: Web Framework: Echo

> ภาคที่ 5: Web Development — ตอนที่ 8 จาก 15 (Part 56–70)

## สารบัญของบทนี้

1. ทบทวน: Echo อยู่ตรงไหนในระบบนิเวศ Web Framework ของ Go
2. Echo คืออะไร ปรัชญาการออกแบบ
3. ติดตั้ง Echo
4. Hello Echo: โปรแกรมแรกและ `echo.New()`
5. Route Handler Signature: `func(c echo.Context) error` ต่างจาก Gin อย่างไร
6. Built-in Middleware: `Logger`, `Recover`, `CORS`
7. `echo.Context` เจาะลึก: `Param`, `QueryParam`, `Bind`, `JSON`
8. Request Validation: Pluggable Validator Interface (จุดต่างสำคัญจาก Gin)
9. Route Groups ใน Echo
10. ตัวอย่างเต็ม: Todo CRUD API ด้วย Echo
11. เทียบ Gin vs Echo vs Fiber
12. สรุปสิ่งที่ได้เรียนในบทนี้
13. แบบฝึกหัดท้ายบท

---

## 1. ทบทวน: Echo อยู่ตรงไหนในระบบนิเวศ Web Framework ของ Go

ใน **Part 061-062** เราใช้เวลาสองบทเต็มกับ **Gin** ทั้ง routing, binding, validation, middleware และ error handling pattern — Gin เป็นตัวแทนที่ดีของ "framework ที่ครบเครื่องและได้รับความนิยมสูงสุด" ในระบบนิเวศ Go แต่ Gin ไม่ใช่ตัวเลือกเดียว บทนี้เราจะมาดู **[Echo](https://echo.labstack.com/)** ซึ่งเป็นอีกหนึ่ง framework ยอดนิยม (ดาวบน GitHub มากกว่า 30,000 ดวง) ที่มีปรัชญาการออกแบบคล้ายกับ Gin ในหลายจุด แต่ก็มีความต่างที่สำคัญพอจะเปลี่ยนวิธีที่เราเขียนโค้ดในระดับหนึ่ง

จุดร่วมของ Echo กับ Gin ที่ควรรู้ไว้ก่อน:

- ทั้งคู่สร้างอยู่**บนฐานของ** `net/http` เหมือนกัน (ต่างจาก Fiber ที่จะเรียนใน **Part 064**) — `*echo.Echo` implement `http.Handler` เช่นเดียวกับ `*gin.Engine`
- ทั้งคู่ใช้ routing แบบ radix tree เพื่อความเร็วในการ match path
- ทั้งคู่มี context object ตัวเดียวที่รวมทุกอย่างที่ handler ต้องใช้
- ทั้งคู่มี route grouping และ middleware chain

ความต่างที่สำคัญที่สุดที่บทนี้จะเน้น คือ **Echo ออกแบบ handler ให้ return `error` เสมอ** และ **Echo ไม่ผูก validation เข้ากับ binding process ให้อัตโนมัติแบบ Gin** — สองจุดนี้สะท้อนปรัชญาการออกแบบที่ต่างกันของทั้งสอง framework ซึ่งเราจะเจาะลึกในหัวข้อที่ 5 และ 8

---

## 2. Echo คืออะไร ปรัชญาการออกแบบ

Echo เปิดตัวโดยทีม LabStack ประมาณปี 2015 โดยวางตัวเองเป็น **"high performance, minimalist Go web framework"** — คำโฆษณานี้ไม่ได้เกินจริง เพราะมันสะท้อนการตัดสินใจออกแบบจริงสองข้อ:

1. **Performance-focused**: Echo ใช้ routing tree ของตัวเองที่ออกแบบมาเพื่อลด memory allocation ต่อ request ให้น้อยที่สุด และมี benchmark ที่ทีมงานเผยแพร่เทียบกับ framework อื่นอย่างสม่ำเสมอ (แม้ในทางปฏิบัติ ความต่างของ throughput ระหว่าง Gin กับ Echo สำหรับ REST API ทั่วไปมักจะเล็กน้อยมากจนไม่มีนัยสำคัญ เพราะ bottleneck จริงมักอยู่ที่ database หรือ business logic ไม่ใช่ routing)

2. **Minimalist core + Middleware ที่ถอดประกอบได้ (pluggable)**: Core ของ Echo ตั้งใจให้เล็กและมี opinion น้อยที่สุด ทุกความสามารถเสริม (logging, CORS, JWT, rate limiting, compression ฯลฯ) ถูกแยกเป็น middleware package ต่างหากในโฟลเดอร์ `middleware/` ที่เราเลือกใช้เฉพาะที่ต้องการ ไม่ต้องแบกของที่ไม่ได้ใช้ แนวคิดนี้คล้ายกับปรัชญา "ไม่มี feature ที่ไม่จำเป็น" ของภาษา Go เอง ที่เราเรียนไปตั้งแต่ **Part 001**

ผลลัพธ์ของปรัชญานี้คือ **Echo ให้เราต้อง "ประกอบ" ความสามารถเองมากกว่า Gin เล็กน้อย** แลกกับความยืดหยุ่นและความชัดเจนว่าความสามารถแต่ละอย่างมาจากไหน จุดที่เห็นชัดที่สุดคือเรื่อง validation ที่เราจะเจาะลึกในหัวข้อที่ 8

### เทียบสรุปเชิงปรัชญา

| ด้าน | Gin | Echo |
|---|---|---|
| Validation | ผูกเข้ากับ binding อัตโนมัติผ่าน struct tag `binding:"..."` | ไม่ผูกให้อัตโนมัติ ต้อง wire `Validator` interface เอง (หัวข้อ 8) |
| Handler signature | `func(c *gin.Context)` ไม่มี return value | `func(c echo.Context) error` บังคับ return error |
| Middleware | มาพร้อม `gin.Default()` (`Logger`+`Recovery`) แบบ built-in สองตัว | แยก package `middleware/` ทั้งหมด ไม่มีตัวไหนติดมาอัตโนมัติ |
| เอกสาร/ชุมชน | ใหญ่ที่สุดในระบบนิเวศ Go | ใหญ่รองลงมา เอกสารทางการค่อนข้างครบ |

---

## 3. ติดตั้ง Echo

```bash
mkdir echo-todo-api
cd echo-todo-api
go mod init echo-todo-api
go get github.com/labstack/echo/v4
```

ผลลัพธ์จริงจากการรัน (Go 1.24.7):

```
go: creating new go.mod: module echo-todo-api
go: added github.com/labstack/echo/v4 v4.13.4
go: added github.com/labstack/gommon v0.4.2
go: added github.com/mattn/go-colorable v0.1.14
go: added github.com/mattn/go-isatty v0.0.20
go: added github.com/valyala/bytebufferpool v1.0.0
go: added github.com/valyala/fasttemplate v1.2.2
go: added golang.org/x/crypto v0.38.0
go: added golang.org/x/net v0.40.0
go: added golang.org/x/sys v0.33.0
go: added golang.org/x/text v0.25.0
```

สังเกตว่า Echo ดึง dependency มาน้อยกว่า Gin อย่างชัดเจน (เทียบกับที่เห็นใน Part 061) — ไม่มี JSON encoder ทางเลือก ไม่มี validator ติดมาด้วย เพราะ Echo core ตั้งใจให้เล็กที่สุดตามปรัชญาข้อ 2

เนื่องจากบทนี้จะสาธิตเรื่อง validation ด้วย เราจึงติดตั้ง `go-playground/validator/v10` เพิ่มเอง (ตัวเดียวกับที่ Gin ใช้อยู่ข้างใน แต่ Echo ไม่ได้ติดมาให้ฟรีๆ):

```bash
go get github.com/go-playground/validator/v10
```

```
go: added github.com/gabriel-vasile/mimetype v1.4.3
go: added github.com/go-playground/locales v0.14.1
go: added github.com/go-playground/universal-translator v0.18.1
go: added github.com/go-playground/validator/v10 v10.22.1
go: added github.com/leodido/go-urn v1.4.0
```

> **หมายเหตุเรื่องเวอร์ชัน**: ณ ตอนเขียนบทนี้ Echo เวอร์ชันล่าสุด (v4.15.x) และ validator เวอร์ชันล่าสุด (v10.30.x) ต้องการ Go 1.25+ ขึ้นไป ถ้าเครื่องคุณใช้ Go 1.24.x เหมือนหลักสูตรนี้ ให้ระบุเวอร์ชันตรงๆ อย่างในตัวอย่าง (`go get github.com/labstack/echo/v4@v4.13.4` และ `go get github.com/go-playground/validator/v10@v10.22.1`) หรืออัปเดต Go เป็นเวอร์ชันใหม่กว่า — `go get` จะแจ้งเตือนชัดเจนถ้า dependency ต้องการ Go เวอร์ชันสูงกว่าที่มี

---

## 4. Hello Echo: โปรแกรมแรกและ `echo.New()`

```go
package main

import (
	"net/http"

	"github.com/labstack/echo/v4"
)

func main() {
	e := echo.New()

	e.GET("/ping", func(c echo.Context) error {
		return c.JSON(http.StatusOK, echo.Map{
			"message": "pong",
		})
	})

	e.Logger.Fatal(e.Start(":8080"))
}
```

จุดที่ต่างจาก Gin ให้สังเกตทันที:

- **`echo.New()`** มีตัวเดียว ไม่มี `Default()` แยกแบบ Gin — Echo ไม่ผูก middleware ใดๆ มาให้อัตโนมัติเลยแม้แต่ตัวเดียว (ต้องเพิ่ม `Logger()`/`Recover()` เองเสมอ ตามหัวข้อ 6)
- **`e.Start(":8080")`** เทียบเท่ากับ `r.Run(":8080")` ของ Gin — ภายในคือ `http.ListenAndServe` เช่นกัน
- **`e.Logger.Fatal(...)`**: Echo มี logger ในตัวชื่อ `e.Logger` ใช้ครอบการเรียก `Start()` เพื่อ log error แล้ว exit โปรแกรมถ้า server เริ่มไม่ได้ (เช่น port ถูกใช้งานอยู่แล้ว)
- **`echo.Map`**: เหมือน `gin.H` เป๊ะ — เป็น shortcut ของ `type Map map[string]interface{}` สำหรับสร้าง JSON แบบ ad-hoc

รันด้วย `go run main.go` ผลลัพธ์ตอน start (ของจริงจากการรัน):

```
   ____    __
  / __/___/ /  ___
 / _// __/ _ \/ _ \
/___/\__/_//_/\___/ v4.13.4
High performance, minimalist Go web framework
https://echo.labstack.com
____________________________________O/_______
                                    O\
⇨ http server started on 0.0.0.0:8080
```

Echo มี ASCII art banner ตอน start (ปิดได้ด้วย `e.HideBanner = true`) ต่างจาก Gin ที่มีแค่ debug log ธรรมดา ทดสอบด้วย curl:

```bash
curl http://localhost:8080/ping
```
```json
{"message":"pong"}
```

---

## 5. Route Handler Signature: `func(c echo.Context) error` ต่างจาก Gin อย่างไร

นี่คือความต่างเชิงโครงสร้างที่สำคัญที่สุดระหว่างสอง framework:

```go
// Gin (Part 061-062)
func(c *gin.Context)

// Echo
func(c echo.Context) error
```

สังเกตสองจุด:

1. **`echo.Context` เป็น interface ไม่ใช่ struct pointer** — Gin ใช้ `*gin.Context` ซึ่งเป็น concrete struct ตรงๆ ส่วน Echo กำหนด `Context` เป็น interface (ดูได้จาก `context.go` ใน source ของ Echo) ทำให้เรา mock หรือ implement `echo.Context` เวอร์ชันของตัวเองได้ในการเขียน test ซึ่งยืดหยุ่นกว่าเล็กน้อยในเชิง design แต่ในการใช้งานประจำวันแทบไม่ต่างกัน
2. **Handler ต้อง `return error` เสมอ** — Gin ปล่อยให้ handler ไม่ return อะไรเลย (จบด้วย `c.JSON(...)` เฉยๆ) ส่วน Echo บังคับให้ทุก handler จบด้วยการ `return` ค่าที่เป็น `error` (หรือ `nil` ถ้าสำเร็จ)

การบังคับ return `error` นี้ทำให้เขียนโค้ดแบบนี้ได้:

```go
todos.GET("/:id", func(c echo.Context) error {
	id, err := strconv.Atoi(c.Param("id"))
	if err != nil {
		return c.JSON(http.StatusBadRequest, echo.Map{"error": "invalid id"})
	}
	t, err := store.Get(id)
	if err != nil {
		return c.JSON(http.StatusNotFound, echo.Map{"error": err.Error()})
	}
	return c.JSON(http.StatusOK, t)
})
```

สังเกตว่า **`c.JSON(...)` เองก็ return ค่าเป็น `error`** อยู่แล้ว (เป็น `error` ที่เกิดจากการเขียน response ล้มเหลว เช่น connection ถูกตัดกลางทาง ซึ่งปกติจะเป็น `nil`) เราจึงเขียน `return c.JSON(...)` ได้ตรงๆ โดยไม่ต้องมี `return` แยกบรรทัด ทำให้โค้ดกระชับกว่า Gin ในบาง pattern (Gin ต้องเขียน `c.JSON(...)` แล้ว `return` เปล่าอีกบรรทัดถ้าต้องการหยุด handler กลางทาง)

จุดที่ได้ประโยชน์ชัดเจนกว่าคือเมื่อใช้ **centralized error handling** — เพราะทุก handler คืน `error` ได้ Echo จึงมีกลไก `HTTPErrorHandler` ที่ดัก error ทุกตัวที่ handler return ออกมาไว้ที่จุดเดียวได้ทันทีโดยไม่ต้องประกอบ pattern แบบ `c.Error()` + middleware ตัวสุดท้ายเหมือนที่เราทำใน Gin (**Part 062 หัวข้อ 8**):

```go
e.HTTPErrorHandler = func(err error, c echo.Context) {
	code := http.StatusInternalServerError
	message := "internal server error"

	var he *echo.HTTPError
	if errors.As(err, &he) {
		code = he.Code
		message = fmt.Sprintf("%v", he.Message)
	}

	if !c.Response().Committed {
		c.JSON(code, echo.Map{"error": message})
	}
}
```

`echo.NewHTTPError(code, message)` คือ error type มาตรฐานของ Echo ที่พก HTTP status code ติดตัวมาด้วย (คล้ายกับ `AppError` ที่เราสร้างเองใน Part 062) — เมื่อ handler `return echo.NewHTTPError(http.StatusBadRequest, "invalid input")` แล้ว `HTTPErrorHandler` ที่ตั้งไว้จะถูกเรียกอัตโนมัติเสมอ ทำให้ error handling รวมศูนย์ได้ **โดยไม่ต้องพึ่ง middleware เพิ่มเติมเหมือน Gin** — นี่คือข้อดีโดยตรงของการบังคับ return `error` ในทุก handler

---

## 6. Built-in Middleware: `Logger`, `Recover`, `CORS`

ตามที่บอกไว้ในหัวข้อ 2 — Echo ไม่ผูก middleware ใดๆ ให้อัตโนมัติ ต้อง import package `middleware` แยกแล้วเพิ่มเองทุกตัว:

```go
import (
	"github.com/labstack/echo/v4"
	"github.com/labstack/echo/v4/middleware"
)

func main() {
	e := echo.New()

	e.Use(middleware.Logger())  // เทียบเท่า gin.Logger()
	e.Use(middleware.Recover()) // เทียบเท่า gin.Recovery() — ดัก panic แปลงเป็น HTTP 500
	e.Use(middleware.CORS())    // เปิด CORS แบบ default (อนุญาตทุก origin — ใช้เฉพาะตอน dev)

	// ... routes
}
```

สาม middleware นี้คือชุดพื้นฐานที่สุดที่โปรเจกต์จริงแทบทุกตัวต้องมี:

| Middleware | หน้าที่ | เทียบเท่าใน Gin |
|---|---|---|
| `middleware.Logger()` | log ทุก request เป็น JSON structured log (method, path, status, latency) | `gin.Logger()` |
| `middleware.Recover()` | `recover()` panic ในเชนแล้วแปลงเป็น HTTP 500 ป้องกัน process ล่ม | `gin.Recovery()` |
| `middleware.CORS()` | ใส่ header `Access-Control-Allow-*` ให้ request จาก origin อื่น | ต้องติดตั้ง `gin-contrib/cors` แยกต่างหาก |

ข้อสังเกต: **Gin ไม่มี CORS middleware ติดตัวมาเลย** ต้องติดตั้ง `github.com/gin-contrib/cors` เพิ่ม ในขณะที่ Echo core มี `middleware.CORS()` ให้ใช้ได้ทันทีโดยไม่ต้องติดตั้งอะไรเพิ่ม เพราะ package `middleware` ของ Echo (แม้จะแยกจาก core logic) ก็ยังถูก maintain อยู่ใน repo เดียวกันและติดมาพร้อมกับ `go get github.com/labstack/echo/v4` อยู่แล้ว

ตัวอย่าง log จริงจาก `middleware.Logger()` (รูปแบบ JSON, ต่างจาก Gin ที่เป็น plain text):

```json
{"time":"2026-09-26T03:16:05Z","id":"","remote_ip":"127.0.0.1","host":"localhost:8080","method":"GET","uri":"/api/v1/todos","user_agent":"curl/8.5.0","status":200,"error":"","latency":56302,"latency_human":"56.302µs","bytes_in":0,"bytes_out":111}
```

ปรับแต่ง format ของ `Logger()` ได้ผ่าน `middleware.LoggerWithConfig(...)` ถ้าต้องการ field เพิ่มเติมหรือรูปแบบอื่น (เช่น plain text แทน JSON) — เราจะไม่ลงรายละเอียดในบทนี้ แต่รูปแบบการปรับแต่งนี้เป็นแพทเทิร์นเดียวกันกับ middleware อื่นๆ เกือบทุกตัวของ Echo: มี constructor เปล่า (`X()`) กับเวอร์ชันที่รับ config (`XWithConfig(cfg)`)

---

## 7. `echo.Context` เจาะลึก: `Param`, `QueryParam`, `Bind`, `JSON`

เทียบ method ของ `echo.Context` กับ `*gin.Context` ที่เรียนไปแล้วใน Part 061 ตรงๆ:

| งาน | Gin | Echo |
|---|---|---|
| อ่าน path parameter | `c.Param("id")` | `c.Param("id")` (ชื่อเหมือนกันเป๊ะ) |
| อ่าน query string | `c.Query("q")` | `c.QueryParam("q")` |
| อ่าน query พร้อม default | `c.DefaultQuery("page", "1")` | `c.QueryParam("page")` แล้วเช็ค `""` เอง (ไม่มี default ในตัว) |
| เขียน JSON response | `c.JSON(code, obj)` | `c.JSON(code, obj)` (เหมือนกัน) |
| Decode JSON body เข้า struct | `c.ShouldBindJSON(&obj)` | `c.Bind(&obj)` |
| อ่าน header | `c.GetHeader("X-Key")` | `c.Request().Header.Get("X-Key")` |

จุดที่ควรสังเกตเป็นพิเศษคือ **`c.Bind(&obj)` ของ Echo ไม่ได้ผูกเฉพาะ JSON** — มันตรวจ `Content-Type` header ของ request แล้วเลือก decoder ให้เองอัตโนมัติ (JSON, XML, form-urlencoded, หรือ query string) ในขณะที่ `c.ShouldBindJSON` ของ Gin เจาะจงว่าต้องเป็น JSON เท่านั้น (Gin มี `c.ShouldBind` แบบ auto-detect เหมือนกัน แต่ที่นิยมใช้ในตัวอย่างทั่วไปมักเป็น `ShouldBindJSON` แบบเจาะจง)

```go
todos.POST("", func(c echo.Context) error {
	var t Todo
	if err := c.Bind(&t); err != nil {
		return c.JSON(http.StatusBadRequest, echo.Map{"error": err.Error()})
	}
	// ...
})
```

**สำคัญมาก**: `c.Bind()` ทำแค่ **decode ข้อมูลเข้า struct เท่านั้น** — มันไม่ตรวจ validation rule ใดๆ เลย (ต่างจาก Gin ที่ `ShouldBindJSON` จะ validate struct tag `binding:"..."` ให้ทันทีในตัวเดียวกัน) ถ้าอยากได้ validation ต้องเรียกขั้นตอนแยกต่างหาก ซึ่งคือหัวข้อถัดไป

---

## 8. Request Validation: Pluggable Validator Interface (จุดต่างสำคัญจาก Gin)

นี่คือความต่างเชิงปรัชญาที่สำคัญที่สุดของบทนี้ ทวนอีกครั้งจาก **Part 062 หัวข้อ 1**:

> Gin ผูก validator เข้ากับ binding process ให้อัตโนมัติ ในขณะที่ Echo ต้อง "wire" validator เข้าไปเองเป็น interface แยก

ลองเขียนแบบ Gin-style ใน Echo ดูก่อน (จะพบว่า**ไม่ทำงาน**):

```go
type Todo struct {
	Title string `json:"title" validate:"required"` // ใส่ tag ไว้เฉยๆ
}

todos.POST("", func(c echo.Context) error {
	var t Todo
	if err := c.Bind(&t); err != nil {
		return c.JSON(http.StatusBadRequest, echo.Map{"error": err.Error()})
	}
	// ตรงนี้ t.Title ว่างก็ผ่านมาได้ปกติ! ไม่มีอะไร validate ให้อัตโนมัติ
	return c.JSON(http.StatusCreated, t)
})
```

Echo **ไม่รู้จัก** struct tag `validate:"..."` เลยจนกว่าเราจะบอกมันว่าใช้ validation engine ตัวไหน ผ่าน field `e.Validator` ซึ่งต้อง implement interface นี้ (นิยามอยู่ใน `echo.Validator`):

```go
type Validator interface {
	Validate(i interface{}) error
}
```

เราต้องเขียน struct ของตัวเองที่ implement interface นี้ โดยข้างในห่อ `go-playground/validator` (ตัวเดียวกับที่ Gin ใช้อยู่ข้างใต้ตามที่บอกไว้ใน Part 062):

```go
import "github.com/go-playground/validator/v10"

type CustomValidator struct {
	validator *validator.Validate
}

func (cv *CustomValidator) Validate(i interface{}) error {
	if err := cv.validator.Struct(i); err != nil {
		// ห่อ error เป็น echo.HTTPError เพื่อให้ HTTPErrorHandler (หัวข้อ 5) จัดการต่อได้ถูกต้อง
		return echo.NewHTTPError(http.StatusBadRequest, err.Error())
	}
	return nil
}

func main() {
	e := echo.New()
	e.Validator = &CustomValidator{validator: validator.New()} // wire เข้ากับ Echo

	// ...
}
```

จากนั้นในทุก handler ต้องเรียก `c.Validate(&obj)` **แยกจาก** `c.Bind(&obj)` เสมอ — สองขั้นตอนที่ Gin ทำให้ในบรรทัดเดียว (`ShouldBindJSON`) ตอนนี้กลายเป็นสองบรรทัดใน Echo:

```go
todos.POST("", func(c echo.Context) error {
	var t Todo
	if err := c.Bind(&t); err != nil {          // ขั้นที่ 1: decode
		return c.JSON(http.StatusBadRequest, echo.Map{"error": err.Error()})
	}
	if err := c.Validate(&t); err != nil {      // ขั้นที่ 2: validate (ต้อง wire e.Validator ไว้ก่อน)
		return err // เป็น *echo.HTTPError อยู่แล้ว ส่งต่อให้ HTTPErrorHandler จัดการ
	}
	created := store.Create(t)
	return c.JSON(http.StatusCreated, created)
})
```

### ทำไม Echo ถึงออกแบบแบบนี้

การแยก validation ออกจาก binding ให้เป็น interface ต่างหากมีเหตุผลเชิงสถาปัตยกรรม: **Echo ไม่อยากผูก dependency กับ `go-playground/validator` โดยตรงในตัว core** เพื่อให้ทีมที่ใช้ validation engine อื่น (เช่น `ozzo-validation`, หรือเขียนกฎ validate เองแบบ manual) เสียบเข้ามาแทนได้โดยไม่ต้องแก้โค้ด framework core เลย นี่คือหลักการ **dependency inversion** ที่เราเรียนแนวคิดไปแล้วใน **Part 013-014 (Interfaces)** — Echo core รู้จักแค่ interface `Validator` ไม่รู้จักและไม่สนใจว่า implementation จริงข้างในเป็นอะไร

ข้อดี: ยืดหยุ่นกว่า เปลี่ยน validation library ได้โดยไม่กระทบโค้ด route handler แม้แต่บรรทัดเดียว (แก้แค่ `CustomValidator`)

ข้อเสีย: ต้องเขียน boilerplate เพิ่ม (การ wire `e.Validator` และเรียก `c.Validate()` แยกทุกที่) และ**ลืมเรียก `c.Validate()`** เป็นบั๊กที่พบบ่อยมากสำหรับคนที่เพิ่งย้ายจาก Gin มา Echo เพราะ `c.Bind()` เพียงอย่างเดียว **ไม่ error แม้ field ที่จำเป็นจะว่างเปล่า**

ทดสอบจริง — เมื่อ POST โดยไม่ส่ง `title`:

```bash
curl -X POST http://localhost:8080/api/v1/todos \
  -H "Content-Type: application/json" -d '{"done":false}'
```
```json
{"message":"Key: 'Todo.Title' Error:Field validation for 'Title' failed on the 'required' tag"}
```

สังเกตว่า error message มีหน้าตาเหมือนกับที่เราเห็นตอนใช้ Gin ทุกประการ เพราะเบื้องหลังคือ `go-playground/validator` ตัวเดียวกัน — ต่างกันแค่ "ใครเรียกมันตอนไหน"

---

## 9. Route Groups ใน Echo

`e.Group(prefix)` ทำงานคล้าย `r.Group(prefix)` ของ Gin ทุกประการ:

```go
api := e.Group("/api/v1")
todos := api.Group("/todos")

todos.GET("", listTodosHandler)
todos.POST("", createTodoHandler)

admin := api.Group("/admin")
admin.Use(AuthMiddleware()) // middleware เฉพาะกลุ่มนี้เท่านั้น
admin.GET("/stats", statsHandler)
```

`Group()` คืนค่าเป็น `*echo.Group` ซึ่งมี method `GET`, `POST`, `Use`, `Group` เหมือนกับ `*echo.Echo` — ซ้อน group กันได้หลายชั้นเหมือน Gin เป๊ะ ความแตกต่างเดียวที่เจอคือชื่อ type (`*echo.Group` แทน `*gin.RouterGroup`) ซึ่งไม่กระทบวิธีใช้งานเลย

---

## 10. ตัวอย่างเต็ม: Todo CRUD API ด้วย Echo

มาสร้าง Todo CRUD API เดียวกับที่ทำใน **Part 061** (ด้วย Gin) ใหม่ทั้งหมดด้วย Echo เพื่อเทียบกันตรงๆ

โครงสร้างโปรเจกต์:

```
echo-todo-api/
├── go.mod
├── go.sum
└── main.go
```

```go
package main

import (
	"errors"
	"net/http"
	"strconv"
	"sync"

	"github.com/go-playground/validator/v10"
	"github.com/labstack/echo/v4"
	"github.com/labstack/echo/v4/middleware"
)

// ---------- Model ----------

type Todo struct {
	ID    int    `json:"id"`
	Title string `json:"title" validate:"required"`
	Done  bool   `json:"done"`
}

// ---------- In-memory store (thread-safe ด้วย sync.Mutex ตามที่เรียนใน Part 039) ----------

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

// ---------- Validator wiring (Echo ไม่ทำให้อัตโนมัติ ต้องต่อเอง) ----------

type CustomValidator struct {
	validator *validator.Validate
}

func (cv *CustomValidator) Validate(i interface{}) error {
	if err := cv.validator.Struct(i); err != nil {
		return echo.NewHTTPError(http.StatusBadRequest, err.Error())
	}
	return nil
}

// ---------- main ----------

func main() {
	e := echo.New()
	e.Validator = &CustomValidator{validator: validator.New()}

	e.Use(middleware.Logger())
	e.Use(middleware.Recover())
	e.Use(middleware.CORS())

	store := NewTodoStore()
	store.Create(Todo{Title: "เรียน Go", Done: false})
	store.Create(Todo{Title: "เขียน Echo CRUD", Done: false})

	api := e.Group("/api/v1")
	todos := api.Group("/todos")

	todos.GET("", func(c echo.Context) error {
		return c.JSON(http.StatusOK, store.List())
	})

	todos.GET("/:id", func(c echo.Context) error {
		id, err := strconv.Atoi(c.Param("id"))
		if err != nil {
			return c.JSON(http.StatusBadRequest, echo.Map{"error": "invalid id"})
		}
		t, err := store.Get(id)
		if err != nil {
			return c.JSON(http.StatusNotFound, echo.Map{"error": err.Error()})
		}
		return c.JSON(http.StatusOK, t)
	})

	todos.POST("", func(c echo.Context) error {
		var t Todo
		if err := c.Bind(&t); err != nil {
			return c.JSON(http.StatusBadRequest, echo.Map{"error": err.Error()})
		}
		if err := c.Validate(&t); err != nil {
			return err
		}
		created := store.Create(t)
		return c.JSON(http.StatusCreated, created)
	})

	todos.PUT("/:id", func(c echo.Context) error {
		id, err := strconv.Atoi(c.Param("id"))
		if err != nil {
			return c.JSON(http.StatusBadRequest, echo.Map{"error": "invalid id"})
		}
		var t Todo
		if err := c.Bind(&t); err != nil {
			return c.JSON(http.StatusBadRequest, echo.Map{"error": err.Error()})
		}
		if err := c.Validate(&t); err != nil {
			return err
		}
		updated, err := store.Update(id, t)
		if err != nil {
			return c.JSON(http.StatusNotFound, echo.Map{"error": err.Error()})
		}
		return c.JSON(http.StatusOK, updated)
	})

	todos.DELETE("/:id", func(c echo.Context) error {
		id, err := strconv.Atoi(c.Param("id"))
		if err != nil {
			return c.JSON(http.StatusBadRequest, echo.Map{"error": "invalid id"})
		}
		if err := store.Delete(id); err != nil {
			return c.JSON(http.StatusNotFound, echo.Map{"error": err.Error()})
		}
		return c.NoContent(http.StatusNoContent)
	})

	e.GET("/search", func(c echo.Context) error {
		q := c.QueryParam("q")
		return c.JSON(http.StatusOK, echo.Map{"query": q})
	})

	e.Logger.Fatal(e.Start(":8080"))
}
```

### ทดสอบจริงทุก endpoint ด้วย `curl` (ผลลัพธ์ของจริงจากการรัน)

```bash
curl http://localhost:8080/api/v1/todos
```
```json
[{"id":1,"title":"เรียน Go","done":false},{"id":2,"title":"เขียน Echo CRUD","done":false}]
```

```bash
curl -X POST http://localhost:8080/api/v1/todos \
  -H "Content-Type: application/json" \
  -d '{"title":"ทดสอบ Echo","done":false}'
```
```json
{"id":3,"title":"ทดสอบ Echo","done":false}
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

ทดสอบกรณี validation ล้มเหลว (ไม่ส่ง `title` มา):

```bash
curl -X POST http://localhost:8080/api/v1/todos \
  -H "Content-Type: application/json" -d '{"done":false}'
```
```json
{"message":"Key: 'Todo.Title' Error:Field validation for 'Title' failed on the 'required' tag"}
```

log ฝั่งเซิร์ฟเวอร์จาก `middleware.Logger()` (รูปแบบ JSON ต่อบรรทัด — ตัดมาบางส่วน):

```
{"time":"2026-09-26T03:16:05Z",...,"method":"GET","uri":"/api/v1/todos","status":200,...}
{"time":"2026-09-26T03:16:05Z",...,"method":"POST","uri":"/api/v1/todos","status":201,...}
{"time":"2026-09-26T03:16:05Z",...,"method":"DELETE","uri":"/api/v1/todos/2","status":204,...}
{"time":"2026-09-26T03:16:05Z",...,"method":"POST","uri":"/api/v1/todos","status":400,"error":"code=400, message=Key: 'Todo.Title' Error:Field validation for 'Title' failed on the 'required' tag",...}
```

ทุก endpoint ทำงานเหมือนเวอร์ชัน Gin ใน Part 061 ทุกประการ — สิ่งที่ต่างคือ **วิธีเขียนโค้ดข้างใน** (return error, Bind+Validate แยกกัน) ไม่ใช่พฤติกรรมที่ผู้ใช้ API มองเห็น

---

## 11. เทียบ Gin vs Echo vs Fiber

ก่อนไป **Part 064** ที่จะแนะนำ Fiber มาสรุปภาพรวมของทั้งสาม framework ที่นิยมที่สุดในระบบนิเวศ Go ไว้ในตารางเดียว เพื่อให้เห็นตำแหน่งของแต่ละตัวชัดเจน:

| หัวข้อเปรียบเทียบ | Gin | Echo | Fiber |
|---|---|---|---|
| ฐานที่สร้างอยู่บน | `net/http` | `net/http` | `fasthttp` (**ไม่ใช่** `net/http`) |
| Performance philosophy | เร็ว เน้น routing tree ประสิทธิภาพสูง | เร็วใกล้เคียง Gin เน้น core เล็ก + allocation ต่ำ | เร็วที่สุดในกลุ่มนี้จาก benchmark เพราะข้าม `net/http` ไปเลย แลกกับ compatibility (Part 064) |
| Validation | ผูกกับ binding อัตโนมัติ (`binding:"..."` tag) | ต้อง wire `Validator` interface เอง (บทนี้) | ไม่มี binding tag ในตัว ต้อง validate เอง (จะเห็นใน Part 064) |
| Handler signature | `func(c *gin.Context)` | `func(c echo.Context) error` | `func(c *fiber.Ctx) error` (คล้าย Echo) |
| CORS ในตัว | ไม่มี ต้องติดตั้ง `gin-contrib/cors` | มี `middleware.CORS()` ในตัว | มี `cors.New()` ในตัว |
| Ecosystem / ความนิยม | ใหญ่ที่สุด เอกสารและตัวอย่างเยอะที่สุด | ใหญ่รองลงมา เอกสารทางการครบ | ใหญ่และโตเร็ว แต่ middleware ต้องเป็นของ Fiber โดยเฉพาะ ใช้ของ `net/http` ร่วมกันไม่ได้ |
| Compatibility กับ `net/http` middleware | ได้ (ผ่าน `http.HandlerFunc` wrapper) | ได้ | **ไม่ได้เลย** — เป็นข้อจำกัดสำคัญที่สุดของ Fiber |

จุดสำคัญที่สุดที่ต้องจำจากตารางนี้คือแถวสุดท้าย: **Gin และ Echo ยังอยู่ในโลกของ `net/http`** ทำให้ middleware, library หรือเครื่องมือ debug ใดๆ ที่เขียนมาสำหรับ `net/http` (เช่น `net/http/pprof`, OpenTelemetry instrumentation มาตรฐาน, `httptest`) ใช้ร่วมกันได้โดยตรงหรือแทบไม่ต้องดัดแปลง ส่วน **Fiber เลือกออกจากโลกนั้นไปทั้งหมด** เพื่อแลกกับความเร็ว — รายละเอียดของ tradeoff นี้และผลกระทบที่ตามมา (โดยเฉพาะเรื่อง pooled context ที่เป็นบั๊กยอดฮิต) คือหัวข้อหลักของ **Part 064**

---

## สรุปสิ่งที่ได้เรียนในบทนี้

- Echo เป็น web framework ที่สร้างอยู่บน `net/http` เหมือน Gin แต่เน้นปรัชญา "minimalist core + pluggable middleware" ทำให้ core เล็กกว่าและ dependency น้อยกว่าตอนติดตั้ง
- ติดตั้งด้วย `go get github.com/labstack/echo/v4` — ระวังเรื่องเวอร์ชันของ Echo/validator ที่อาจต้องการ Go เวอร์ชันใหม่กว่าที่มี
- Handler signature ของ Echo คือ `func(c echo.Context) error` ต่างจาก Gin ที่ไม่ return อะไรเลย — การบังคับ return error ทำให้ centralized error handling ผ่าน `e.HTTPErrorHandler` ทำได้ง่ายกว่า Gin โดยไม่ต้องพึ่ง middleware เพิ่ม
- Built-in middleware หลัก: `middleware.Logger()`, `middleware.Recover()`, `middleware.CORS()` — ไม่มีตัวไหนติดมาอัตโนมัติ ต้อง `e.Use()` เองทุกตัว
- `echo.Context` มี method คล้าย `gin.Context` มาก (`Param`, `QueryParam`, `JSON`) แต่ decode body ใช้ `c.Bind()` ตัวเดียวที่ auto-detect content type
- ความต่างสำคัญที่สุด: **Echo ไม่ผูก validation เข้ากับ binding อัตโนมัติ** ต้อง implement interface `echo.Validator` เอง แล้วเรียก `c.Validate()` แยกจาก `c.Bind()` เสมอ — ลืมเรียกคือบั๊กยอดฮิตของคนย้ายจาก Gin มา Echo
- Route groups ของ Echo (`e.Group()`) ทำงานเหมือน Gin ทุกประการ
- Gin, Echo อยู่บนฐาน `net/http` เหมือนกัน ส่วน Fiber ใช้ `fasthttp` ซึ่งทำให้ไม่ compatible กับ middleware ของ `net/http` เลย — รายละเอียดใน Part 064

## แบบฝึกหัดท้ายบท

1. สร้างโปรเจกต์ Echo ใหม่ ติดตั้งและรันตัวอย่าง `/ping` ให้สำเร็จ แล้วลองปิด banner ด้วย `e.HideBanner = true` สังเกตความต่างของ output ตอน start
2. เขียน endpoint `POST /api/v1/todos` แบบไม่เรียก `c.Validate()` เลย (ลืมโดยตั้งใจ) แล้วยิง request โดยไม่ส่ง `title` มา สังเกตว่า response ที่ได้ต่างจากตอนเรียก `c.Validate()` อย่างไร และอธิบายว่าทำไม
3. เพิ่ม `e.HTTPErrorHandler` แบบ custom ตามหัวข้อ 5 แล้วปรับทุก handler ใน Todo CRUD ให้ `return echo.NewHTTPError(...)` แทนการเขียน `c.JSON(status, echo.Map{"error": ...})` เอง ทดสอบว่า response ที่ได้ยังเหมือนเดิมหรือไม่
4. เขียน middleware ของตัวเอง (ไม่ใช้ `middleware.Logger()`) ที่ log แค่ `method`, `path`, และเวลาที่ใช้ (latency) เป็น plain text บรรทัดเดียว โดยใช้โครงสร้าง `func(next echo.HandlerFunc) echo.HandlerFunc` (ดู pattern จาก source ของ Echo middleware เป็นตัวอย่าง)
5. ลองเปลี่ยน validation tag จาก `validate:"required"` เป็น `validate:"required,min=3"` สำหรับ field `Title` แล้วทดสอบว่า title ที่สั้นกว่า 3 ตัวอักษรถูกปฏิเสธหรือไม่
6. เขียนตารางเปรียบเทียบของตัวเอง (ไม่ต้องเหมือนหัวข้อ 11 เป๊ะ) ระหว่างโค้ด Gin จาก Part 061 กับโค้ด Echo ในบทนี้ โดยเลือกมา 3 endpoint แล้วนับจำนวนบรรทัดของแต่ละเวอร์ชัน สรุปว่าอันไหนกระชับกว่าในกรณีไหน

---

**ต่อไป**: [Part 064 — Web Framework: Fiber](./064-fiber-framework.md)
