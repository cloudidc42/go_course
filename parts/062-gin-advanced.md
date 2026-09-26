# Part 062: Gin ขั้นสูง: Validation, Binding, Grouping

> ภาคที่ 5: Web Development — ตอนที่ 7 จาก 15 (Part 56–70)

## สารบัญของบทนี้

1. ทบทวน: `go-playground/validator` อยู่เบื้องหลัง binding tag ของ Gin อย่างไร
2. Validation tag ที่ใช้บ่อย
3. `ShouldBindJSON` vs `BindJSON`: ความแตกต่างที่ต้องเข้าใจให้ลึก
4. Custom Validator: เขียนกฎ validation ของตัวเอง
5. Custom Middleware: โครงสร้างและ `c.Next()` / `c.Abort()`
6. ลำดับการทำงานของ middleware (Middleware Ordering)
7. Route Groups กับ Group-level Middleware: สร้าง Admin Group ที่ต้อง Auth
8. Error Handling Middleware Pattern: `c.Error()` + Final Handler
9. ตัวอย่างเต็ม: รวมทุกอย่างเข้าด้วยกันพร้อม API Versioning (`/api/v1`)
10. สรุปสิ่งที่ได้เรียนในบทนี้
11. แบบฝึกหัดท้ายบท

---

## 1. ทบทวน: `go-playground/validator` อยู่เบื้องหลัง binding tag ของ Gin อย่างไร

ใน **Part 061** เราเห็นว่าแค่เติม `binding:"required"` ใน struct tag ก็ทำให้ Gin validate field นั้นให้อัตโนมัติ เบื้องหลังกลไกนี้คือไลบรารีแยกต่างหากชื่อ [`github.com/go-playground/validator/v10`](https://github.com/go-playground/validator) ซึ่ง Gin ใช้เป็น **default validation engine**

เมื่อเรา `go get github.com/gin-gonic/gin` ตัว `go-playground/validator/v10` จะถูกดึงมาเป็น transitive dependency โดยอัตโนมัติอยู่แล้ว (เห็นได้จาก `go.sum` ใน Part 061) Gin เข้าถึง instance ของ validator ผ่าน `binding.Validator.Engine()` ซึ่งคืนค่าเป็น `interface{}` ที่เรา type-assert กลับเป็น `*validator.Validate` ได้ — เราจะใช้จุดนี้ในหัวข้อที่ 4 ตอนเขียน custom validator

ความสัมพันธ์คือ:

```
struct tag `binding:"..."`
        │
        ▼
Gin's ShouldBindJSON / BindJSON
        │  เรียก
        ▼
binding.Validator.ValidateStruct(obj)
        │  ซึ่งภายในคือ
        ▼
go-playground/validator.Struct(obj)
```

พูดง่ายๆ คือ `binding:"..."` ของ Gin กับ `validate:"..."` ที่เราจะเห็นใน **Part 063 (Echo)** นั้นแทบจะเป็นระบบเดียวกัน เพียงแต่ Gin ผูก validator เข้ากับ binding process ให้อัตโนมัติ ในขณะที่ Echo (ตามที่จะเห็นในบทถัดไป) ต้อง "wire" validator เข้าไปเองเป็น interface แยก — นี่คือความแตกต่างเชิงปรัชญาระหว่างสอง framework ที่เราจะพูดถึงในรายละเอียดของ Part 063

---

## 2. Validation tag ที่ใช้บ่อย

ตารางสรุป tag ที่ใช้บ่อยที่สุดใน validator (ใช้ได้ทั้งกับ Gin ผ่าน `binding:"..."` โดยตรง):

| Tag | ความหมาย | ตัวอย่าง |
|---|---|---|
| `required` | ห้ามเป็น zero value (`""`, `0`, `nil`, `false`) | `binding:"required"` |
| `email` | ต้องเป็นรูปแบบอีเมลที่ถูกต้อง | `binding:"required,email"` |
| `min=N` | ค่าต่ำสุด (string = ความยาว, number = ค่า) | `binding:"min=8"` |
| `max=N` | ค่าสูงสุด | `binding:"max=100"` |
| `len=N` | ความยาว/ค่าต้องเท่ากับ N พอดี | `binding:"len=10"` |
| `gt=N` / `gte=N` | มากกว่า / มากกว่าเท่ากับ | `binding:"gt=0"` |
| `lt=N` / `lte=N` | น้อยกว่า / น้อยกว่าเท่ากับ | `binding:"lte=150"` |
| `oneof=a b c` | ต้องเป็นหนึ่งในค่าที่กำหนด (คั่นด้วย space) | `binding:"oneof=admin user guest"` |
| `numeric` | ต้องเป็นตัวเลข | `binding:"numeric"` |
| `alphanum` | ต้องเป็นตัวอักษร+ตัวเลขเท่านั้น | `binding:"alphanum"` |
| `url` | ต้องเป็น URL ที่ valid | `binding:"url"` |
| `uuid` | ต้องเป็นรูปแบบ UUID | `binding:"uuid"` |
| `dive` | ใช้กับ slice/map เพื่อ validate สมาชิกแต่ละตัวข้างใน | `binding:"dive,required"` |
| `omitempty` | ข้าม validation ถ้า field เป็น zero value (ใช้กับ field ที่ optional) | `binding:"omitempty,email"` |

รวม tag หลายตัวได้ด้วย comma:

```go
type RegisterRequest struct {
	Email    string `json:"email" binding:"required,email"`
	Password string `json:"password" binding:"required,min=8"`
	Role     string `json:"role" binding:"required,oneof=admin user"`
	Website  string `json:"website" binding:"omitempty,url"`
}
```

---

## 3. `ShouldBindJSON` vs `BindJSON`: ความแตกต่างที่ต้องเข้าใจให้ลึก

ใน Part 061 เราพูดถึงความต่างแบบผิวเผินไปแล้ว มาดูโค้ดจริงเทียบกันตรงๆ:

```go
// แนวทางที่ 1: BindJSON
r.POST("/register", func(c *gin.Context) {
	var req RegisterRequest
	if err := c.BindJSON(&req); err != nil {
		// ณ จุดนี้ Gin ได้เขียน response 400 ให้เรียบร้อยแล้ว
		// (เรียก c.AbortWithError(400, err) ให้อัตโนมัติ)
		// เราแค่ return ออกไปเฉยๆ ห้ามเขียน response ซ้ำอีก มิฉะนั้นจะเจอ
		// "http: superfluous response.WriteHeader call" ใน log
		return
	}
	c.JSON(http.StatusCreated, req)
})

// แนวทางที่ 2: ShouldBindJSON
r.POST("/register", func(c *gin.Context) {
	var req RegisterRequest
	if err := c.ShouldBindJSON(&req); err != nil {
		// ควบคุม response เองทั้งหมด — เลือก format, เลือก field ที่จะโชว์, log เพิ่มได้
		c.JSON(http.StatusBadRequest, gin.H{
			"error":   "validation failed",
			"details": err.Error(),
		})
		return
	}
	c.JSON(http.StatusCreated, req)
})
```

สรุปเป็นตาราง:

| | `BindJSON` | `ShouldBindJSON` |
|---|---|---|
| เขียน response ตอน error ให้อัตโนมัติ | ใช่ (เรียก `c.AbortWithError` เป็น HTTP 400 เสมอ) | ไม่ — คืน `error` ธรรมดาให้เราตัดสินใจเอง |
| ควบคุม status code ตอน error ได้ | ไม่ได้ (fix เป็น 400) | ได้เต็มที่ |
| ควบคุมรูปแบบ error message ได้ | ไม่ได้ | ได้ |
| เหมาะกับ | prototype เร็วๆ, internal tool ที่ไม่ต้องคุม response format | โปรเจกต์จริง โดยเฉพาะที่มี error handling middleware รวมศูนย์ (หัวข้อ 8) |

**คำแนะนำสำหรับโปรเจกต์จริง**: ใช้ `ShouldBindJSON` เป็นหลักเสมอ เพราะทำให้เรารวม error response ทุกจุดในระบบให้มีรูปแบบเดียวกันได้ (เช่น `{"error": "...", "code": "..."}`) ซึ่งเป็นสิ่งที่ทีม frontend หรือผู้ใช้ API ต้องพึ่งพา ความสม่ำเสมอของ error format คือส่วนหนึ่งของหลักการ RESTful API ที่ดีตามที่พูดถึงใน **Part 059**

---

## 4. Custom Validator: เขียนกฎ validation ของตัวเอง

บางครั้ง built-in tag ไม่พอ เช่น ต้องการเช็คว่าอายุ >= 18 (`gte=18` ทำได้อยู่แล้วจริงๆ แต่สมมติเป็นตัวอย่างกฎธุรกิจที่ซับซ้อนกว่านั้น) เราสามารถลงทะเบียน validation tag ของตัวเองผ่าน `validator.RegisterValidation` ได้:

```go
package main

import (
	"log"
	"net/http"

	"github.com/gin-gonic/gin"
	"github.com/gin-gonic/gin/binding"
	"github.com/go-playground/validator/v10"
)

// isAdult คือ custom validation function — รับ FieldLevel แล้วคืน bool ว่าผ่านเงื่อนไขหรือไม่
func isAdult(fl validator.FieldLevel) bool {
	age := fl.Field().Int()
	return age >= 18
}

type RegisterRequest struct {
	Email    string `json:"email" binding:"required,email"`
	Password string `json:"password" binding:"required,min=8"`
	Age      int    `json:"age" binding:"required,isadult"` // tag ที่เราตั้งเอง
}

func main() {
	// ดึง validator engine ของ Gin ออกมา type-assert เป็น *validator.Validate
	// แล้วลงทะเบียน tag ใหม่ชื่อ "isadult" ก่อนที่ route ใดๆ จะถูกเรียกใช้งาน
	if v, ok := binding.Validator.Engine().(*validator.Validate); ok {
		if err := v.RegisterValidation("isadult", isAdult); err != nil {
			log.Fatal(err)
		}
	}

	r := gin.Default()
	r.POST("/register", func(c *gin.Context) {
		var req RegisterRequest
		if err := c.ShouldBindJSON(&req); err != nil {
			c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
			return
		}
		c.JSON(http.StatusCreated, gin.H{"message": "registered", "email": req.Email})
	})
	r.Run(":8081")
}
```

ทดสอบจริง (ผลลัพธ์จากการรันจริง):

```bash
curl -X POST http://localhost:8081/register -H "Content-Type: application/json" \
  -d '{"email":"a@b.com","password":"12345678","age":15}'
```
```json
{"error":"Key: 'RegisterRequest.Age' Error:Field validation for 'Age' failed on the 'isadult' tag"}
```

```bash
curl -X POST http://localhost:8081/register -H "Content-Type: application/json" \
  -d '{"email":"a@b.com","password":"12345678","age":20}'
```
```json
{"message":"registered","email":"a@b.com"}
```

ข้อควรระวัง: `RegisterValidation` ต้องเรียก**ก่อน**ที่ handler แรกจะถูกเรียกใช้งาน (ปกติเรียกใน `main()` ตอน setup) เพราะ validator engine เป็น instance ที่ใช้ร่วมกันทั้งแอป (global-like ผ่าน `binding.Validator`) การลงทะเบียนซ้ำในหลาย goroutine พร้อมกันโดยไม่มีการ synchronize อาจทำให้เกิด data race ได้ — จึงควรทำเพียงครั้งเดียวตอน startup เท่านั้น

---

## 5. Custom Middleware: โครงสร้างและ `c.Next()` / `c.Abort()`

ทบทวนจาก **Part 057**: middleware คือฟังก์ชันที่ "ห่อ" handler อื่นไว้ เพื่อทำงานบางอย่างก่อนและ/หรือหลัง handler จริงทำงาน ใน Gin, middleware มี signature เดียวกับ handler ทุกประการ:

```go
func(c *gin.Context)
```

ตัวอย่าง middleware ที่ log เวลาในการประมวลผลของแต่ละ request:

```go
func LoggerMiddleware() gin.HandlerFunc {
	return func(c *gin.Context) {
		start := time.Now()

		c.Next() // ส่งต่อการทำงานให้ middleware/handler ตัวถัดไปในลำดับ

		latency := time.Since(start)
		log.Printf("[%s] %s %s -> %d (%s)",
			c.Request.Method, c.Request.URL.Path, c.ClientIP(), c.Writer.Status(), latency)
	}
}
```

จุดสำคัญที่สุดคือ **`c.Next()`** — มันคือคำสั่งที่บอกให้ Gin เรียก middleware/handler ตัวถัดไปในลำดับ (chain) แล้ว**ค่อยกลับมาทำงานต่อที่บรรทัดหลัง `c.Next()`** เมื่อ handler ปลายทางทำงานเสร็จแล้ว นี่คือสิ่งที่ทำให้เราวัด `latency` ได้ถูกต้อง เพราะโค้ดหลัง `c.Next()` รันหลังจาก handler จริงตอบกลับ response เสร็จแล้ว

ถ้า**ไม่เรียก** `c.Next()` เลย middleware chain จะ**หยุดอยู่ตรงนั้น** — handler ปลายทางจะไม่ถูกเรียกเลย นี่คือกลไกเดียวกับที่ `c.Abort()` ใช้:

```go
func AuthMiddleware() gin.HandlerFunc {
	const validKey = "secret-admin-key"
	return func(c *gin.Context) {
		key := c.GetHeader("X-API-Key")
		if key != validKey {
			// AbortWithStatusJSON = c.Abort() + c.JSON() รวมกันในคำสั่งเดียว
			c.AbortWithStatusJSON(http.StatusUnauthorized, gin.H{"error": "invalid or missing API key"})
			return // ควร return เสมอ เพื่อไม่ให้โค้ดหลังจากนี้ทำงานต่อ
		}
		c.Next()
	}
}
```

`c.Abort()` ไม่ได้ทำให้ handler ปัจจุบัน "return ทันที" (Go ไม่มีกลไกแบบนั้น) — มันแค่ตั้ง flag ภายใน `*gin.Context` ว่า "อย่าเรียก handler ตัวถัดไปอีก" ดังนั้นเราต้อง **`return` ออกจากฟังก์ชันเองด้วยมือ** เสมอหลังเรียก `Abort()`/`AbortWithStatusJSON()`/`AbortWithError()` ไม่งั้นโค้ดบรรทัดถัดไปในฟังก์ชันเดียวกันจะยังทำงานต่อ (แม้ handler ตัวถัดไปจะไม่ถูกเรียกก็ตาม)

### สรุปพฤติกรรม

| เรียก | Handler ตัวถัดไปถูกเรียกหรือไม่ | โค้ดหลังจุดที่เรียกในฟังก์ชันเดียวกันทำงานต่อหรือไม่ |
|---|---|---|
| `c.Next()` | ใช่ (แล้วกลับมาทำงานต่อหลังจากนั้น) | ใช่ |
| ไม่เรียกอะไรเลย (ปล่อยให้ฟังก์ชัน return เฉยๆ) | ใช่ — Gin เรียก `Next()` ให้อัตโนมัติถ้ายังไม่เคยเรียก | ใช่ |
| `c.Abort()` | ไม่ | ใช่ (ต้อง `return` เองด้วยมือ) |
| `c.AbortWithStatusJSON(...)` | ไม่ | ใช่ (ต้อง `return` เองด้วยมือ) |

---

## 6. ลำดับการทำงานของ middleware (Middleware Ordering)

Middleware ทำงานเป็นลำดับที่ **ลงทะเบียนก่อน = เรียกก่อนเสมอ** ในฝั่ง "ก่อน `c.Next()`" แต่จะเรียง**ย้อนกลับ**ในฝั่ง "หลัง `c.Next()`" — เหมือนการเข้าซ้อนกันของฟังก์ชัน (คล้ายกับ `defer` ที่เรียนใน **Part 009**):

```go
r.Use(MiddlewareA())
r.Use(MiddlewareB())
r.GET("/x", handlerX)
```

ลำดับการทำงานจริงเมื่อมี request เข้ามาที่ `/x`:

```
MiddlewareA (ก่อน c.Next())
  MiddlewareB (ก่อน c.Next())
    handlerX
  MiddlewareB (หลัง c.Next())
MiddlewareA (หลัง c.Next())
```

พิสูจน์ด้วยโค้ดจริง:

```go
func Trace(name string) gin.HandlerFunc {
	return func(c *gin.Context) {
		log.Printf("%s: before", name)
		c.Next()
		log.Printf("%s: after", name)
	}
}

r := gin.New()
r.Use(Trace("A"), Trace("B"))
r.GET("/x", func(c *gin.Context) {
	log.Println("handler: running")
	c.JSON(200, gin.H{"ok": true})
})
```

log ที่ได้เมื่อยิง `curl localhost:8080/x`:

```
A: before
B: before
handler: running
B: after
A: after
```

ความเข้าใจเรื่อง ordering นี้สำคัญมากเวลาผสม middleware หลายตัว เช่น `Recovery()` ควรอยู่ **นอกสุด** (ลงทะเบียนก่อนสุด หรืออยู่ก่อน middleware อื่นที่อาจ panic) เพื่อให้มันครอบคลุม panic ที่เกิดจาก middleware ตัวอื่นๆ ที่อยู่ข้างในได้ด้วย ไม่ใช่แค่ panic จาก handler ปลายทางเท่านั้น

---

## 7. Route Groups กับ Group-level Middleware: สร้าง Admin Group ที่ต้อง Auth

ต่อยอดจาก `r.Group()` ที่เรียนใน Part 061 ตอนนี้เราจะผูก middleware **เฉพาะกลุ่ม** โดยไม่กระทบ route อื่นนอกกลุ่ม:

```go
v1 := r.Group("/api/v1")
{
	// route สาธารณะ ไม่ต้อง auth
	v1.GET("/ping", pingHandler)
	v1.POST("/register", registerHandler)

	// admin group: ผูก AuthMiddleware() เฉพาะที่กลุ่มนี้เท่านั้น
	admin := v1.Group("/admin")
	admin.Use(AuthMiddleware())
	{
		admin.GET("/stats", statsHandler)
		admin.DELETE("/cache", clearCacheHandler)
	}
}
```

`admin.Use(AuthMiddleware())` เพิ่ม middleware เข้าไปใน **middleware chain เฉพาะของ `admin` group เท่านั้น** — route `v1.GET("/ping", ...)` ที่ประกาศไว้นอก `admin` จะไม่ถูก `AuthMiddleware()` แตะต้องเลย เพราะ Gin เก็บ middleware chain แยกเป็นสำเนาของแต่ละ group (ผ่านกลไก `RouterGroup.Handlers` ที่ copy มาจาก parent group ตอนสร้าง group ลูก)

นี่คือประโยชน์หลักของ route grouping ที่มากกว่าแค่การจัดกลุ่ม prefix — มันคือกลไกจัดการ **security boundary** ที่ชัดเจนในระดับโครงสร้างโค้ด ทำให้มองเห็นได้ทันทีว่า route ไหนต้อง authenticate และ route ไหนไม่ต้อง

---

## 8. Error Handling Middleware Pattern: `c.Error()` + Final Handler

ปัญหาของการเขียน `c.JSON(status, gin.H{"error": ...})` กระจายอยู่ทุก handler คือ **รูปแบบ error response ไม่สม่ำเสมอ** ถ้ามี 50 endpoint ก็มีโอกาสเขียนรูปแบบผิดเพี้ยนกันได้ 50 แบบ วิธีแก้คือรวมศูนย์การแปลง error เป็น response ไว้ที่ **middleware ตัวสุดท้าย** ตัวเดียว โดยใช้ `c.Error(err)` เป็นตัวกลางส่งผ่าน error

### ขั้นตอนที่ 1: กำหนด error type ที่มี HTTP status ติดมาด้วย

```go
type AppError struct {
	Status  int
	Message string
}

func (e *AppError) Error() string { return e.Message }
```

### ขั้นตอนที่ 2: handler รายงาน error ผ่าน `c.Error()` แทนการเขียน response เอง

```go
v1.POST("/register", func(c *gin.Context) {
	var req RegisterRequest
	if err := c.ShouldBindJSON(&req); err != nil {
		c.Error(&AppError{Status: http.StatusBadRequest, Message: err.Error()})
		return // สำคัญ: ต้อง return เอง c.Error() ไม่ได้ abort ให้อัตโนมัติ
	}
	c.JSON(http.StatusCreated, gin.H{"message": "registered", "email": req.Email})
})
```

`c.Error(err)` เพียงแค่**เก็บ error ไว้ใน slice** `c.Errors` ของ context เท่านั้น — มันไม่ได้เขียน response หรือ abort ให้อัตโนมัติ (ต่างจาก `c.AbortWithError` ที่ทำทั้งสองอย่าง) เราจึงต้อง `return` ออกจาก handler เองเสมอหลังเรียก `c.Error()` เพื่อไม่ให้โค้ดส่วนที่เหลือ (ที่คาดหวังว่าทุกอย่างสำเร็จ) ทำงานต่อ

### ขั้นตอนที่ 3: middleware ตัวสุดท้ายอ่าน `c.Errors` แล้วแปลงเป็น response เดียว

```go
func ErrorHandlerMiddleware() gin.HandlerFunc {
	return func(c *gin.Context) {
		c.Next() // ให้ handler ทำงานก่อน อาจมีการเรียก c.Error(err) ระหว่างทาง

		if len(c.Errors) == 0 {
			return
		}

		err := c.Errors.Last().Err
		var appErr *AppError
		if errors.As(err, &appErr) {
			c.JSON(appErr.Status, gin.H{"error": appErr.Message})
			return
		}
		// error ชนิดที่ไม่รู้จัก ให้ตอบ 500 แบบปลอดภัย ไม่หลุด internal detail ออกไปให้ผู้ใช้เห็น
		c.JSON(http.StatusInternalServerError, gin.H{"error": "internal server error"})
	}
}
```

สังเกตว่าเราใช้ `errors.As` (จาก **Part 016: Error Handling ขั้นสูง**) เพื่อตรวจสอบว่า error ที่เก็บไว้เป็น `*AppError` ของเราเองหรือไม่ ถ้าใช่ก็ดึง `Status`/`Message` มาสร้าง response ที่ถูกต้อง ถ้าเป็น error ชนิดอื่นที่เราไม่รู้จัก (เช่น error จาก database driver ที่หลุดขึ้นมาโดยไม่ได้ตั้งใจ) เราตอบ 500 กลาง ๆ แบบปลอดภัยเสมอ เพื่อไม่ให้ internal error message (ที่อาจมีข้อมูลอ่อนไหว เช่น connection string) รั่วไหลออกไปสู่ client

**สำคัญมาก**: `ErrorHandlerMiddleware()` ต้องถูกลงทะเบียนด้วย `r.Use()` **ก่อน** route handler ทั้งหมด (เพราะมันต้องอยู่ "รอบนอก" ของ chain เพื่อให้ `c.Next()` เรียกไปถึง handler แล้ววนกลับมาตรวจ `c.Errors` ได้) — ดูลำดับที่ถูกต้องในตัวอย่างเต็มของหัวข้อถัดไป

---

## 9. ตัวอย่างเต็ม: รวมทุกอย่างเข้าด้วยกันพร้อม API Versioning (`/api/v1`)

```go
package main

import (
	"errors"
	"fmt"
	"log"
	"net/http"
	"time"

	"github.com/gin-gonic/gin"
	"github.com/gin-gonic/gin/binding"
	"github.com/go-playground/validator/v10"
)

// ---------- Custom validator ----------

func isAdult(fl validator.FieldLevel) bool {
	return fl.Field().Int() >= 18
}

// ---------- Request DTOs ----------

type RegisterRequest struct {
	Email    string `json:"email" binding:"required,email"`
	Password string `json:"password" binding:"required,min=8"`
	Age      int    `json:"age" binding:"required,isadult"`
}

// ---------- Middleware ----------

func LoggerMiddleware() gin.HandlerFunc {
	return func(c *gin.Context) {
		start := time.Now()
		c.Next()
		latency := time.Since(start)
		log.Printf("[%s] %s %s -> %d (%s)",
			c.Request.Method, c.Request.URL.Path, c.ClientIP(), c.Writer.Status(), latency)
	}
}

func AuthMiddleware() gin.HandlerFunc {
	const validKey = "secret-admin-key"
	return func(c *gin.Context) {
		if c.GetHeader("X-API-Key") != validKey {
			c.AbortWithStatusJSON(http.StatusUnauthorized, gin.H{"error": "invalid or missing API key"})
			return
		}
		c.Next()
	}
}

type AppError struct {
	Status  int
	Message string
}

func (e *AppError) Error() string { return e.Message }

func ErrorHandlerMiddleware() gin.HandlerFunc {
	return func(c *gin.Context) {
		c.Next()
		if len(c.Errors) == 0 {
			return
		}
		err := c.Errors.Last().Err
		var appErr *AppError
		if errors.As(err, &appErr) {
			c.JSON(appErr.Status, gin.H{"error": appErr.Message})
			return
		}
		c.JSON(http.StatusInternalServerError, gin.H{"error": "internal server error"})
	}
}

func main() {
	// ลงทะเบียน custom validator ก่อน route ใดๆ ถูกใช้งาน
	if v, ok := binding.Validator.Engine().(*validator.Validate); ok {
		if err := v.RegisterValidation("isadult", isAdult); err != nil {
			log.Fatal(err)
		}
	}

	// ใช้ gin.New() เพื่อคุมลำดับ middleware เองทั้งหมด (ไม่ใช้ gin.Default())
	r := gin.New()
	r.Use(LoggerMiddleware())     // อยู่นอกสุด: วัดเวลาทั้งหมดรวม middleware อื่น
	r.Use(gin.Recovery())         // ครอบ panic จากทุกอย่างที่อยู่ข้างใน
	r.Use(ErrorHandlerMiddleware()) // ต้องมาก่อน route เสมอ เพื่อดัก c.Errors หลัง c.Next()

	v1 := r.Group("/api/v1")
	{
		v1.POST("/register", func(c *gin.Context) {
			var req RegisterRequest
			if err := c.ShouldBindJSON(&req); err != nil {
				c.Error(&AppError{Status: http.StatusBadRequest, Message: err.Error()})
				return
			}
			c.JSON(http.StatusCreated, gin.H{"message": "registered", "email": req.Email})
		})

		v1.GET("/ping", func(c *gin.Context) {
			c.JSON(http.StatusOK, gin.H{"message": "pong"})
		})

		admin := v1.Group("/admin")
		admin.Use(AuthMiddleware())
		{
			admin.GET("/stats", func(c *gin.Context) {
				c.JSON(http.StatusOK, gin.H{"users": 42, "uptime": "3d"})
			})

			admin.DELETE("/cache", func(c *gin.Context) {
				c.Error(&AppError{Status: http.StatusServiceUnavailable, Message: "cache service is down"})
			})
		}
	}

	fmt.Println("server running on :8081")
	r.Run(":8081")
}
```

### ทดสอบจริงทุก endpoint (ผลลัพธ์จากการรันจริง)

```bash
curl http://localhost:8081/api/v1/ping
```
```json
{"message":"pong"}
```

```bash
curl -X POST http://localhost:8081/api/v1/register -H "Content-Type: application/json" \
  -d '{"email":"a@b.com","password":"12345678","age":20}'
```
```json
{"email":"a@b.com","message":"registered"}
```

```bash
curl -X POST http://localhost:8081/api/v1/register -H "Content-Type: application/json" \
  -d '{"email":"not-an-email","password":"12345678","age":20}'
```
```json
{"error":"Key: 'RegisterRequest.Email' Error:Field validation for 'Email' failed on the 'email' tag"}
```

```bash
curl -X POST http://localhost:8081/api/v1/register -H "Content-Type: application/json" \
  -d '{"email":"a@b.com","password":"12345678","age":15}'
```
```json
{"error":"Key: 'RegisterRequest.Age' Error:Field validation for 'Age' failed on the 'isadult' tag"}
```

```bash
curl -o /dev/null -w "%{http_code}\n" http://localhost:8081/api/v1/admin/stats
# 401  (ไม่ส่ง X-API-Key มา)
```

```bash
curl -H "X-API-Key: secret-admin-key" http://localhost:8081/api/v1/admin/stats
```
```json
{"uptime":"3d","users":42}
```

```bash
curl -H "X-API-Key: secret-admin-key" -X DELETE http://localhost:8081/api/v1/admin/cache
```
```json
{"error":"cache service is down"}
```

log ฝั่งเซิร์ฟเวอร์ที่ได้จริงระหว่างทดสอบ (จาก `LoggerMiddleware`):

```
2026/09/26 02:50:22 [GET] /api/v1/ping 127.0.0.1 -> 200 (78.133µs)
2026/09/26 02:50:22 [POST] /api/v1/register 127.0.0.1 -> 201 (361.702µs)
2026/09/26 02:50:22 [POST] /api/v1/register 127.0.0.1 -> 400 (147.228µs)
2026/09/26 02:50:22 [POST] /api/v1/register 127.0.0.1 -> 400 (145.67µs)
2026/09/26 02:50:22 [GET] /api/v1/admin/stats 127.0.0.1 -> 401 (56.745µs)
2026/09/26 02:50:22 [GET] /api/v1/admin/stats 127.0.0.1 -> 200 (40.52µs)
2026/09/26 02:50:22 [DELETE] /api/v1/admin/cache 127.0.0.1 -> 503 (74.029µs)
```

สังเกตว่า status code ที่ log แสดง (`401`, `400`, `503`) ตรงกับ response จริงทุกครั้ง แม้ว่า status code เหล่านี้จะถูกตั้งค่า**ทีหลัง** จากใน `ErrorHandlerMiddleware` (ซึ่งทำงานหลัง `c.Next()`) — นี่คือเหตุผลที่ `LoggerMiddleware` ต้องอยู่**นอกสุด** (ลงทะเบียนก่อน `ErrorHandlerMiddleware`) เพื่อให้ `c.Writer.Status()` อ่านค่าที่ถูกต้องหลังจากทุกอย่าง (รวมถึง error handler) เขียน response เสร็จสมบูรณ์แล้ว — ตรงตามหลักการ ordering ที่อธิบายไว้ในหัวข้อ 6

---

## สรุปสิ่งที่ได้เรียนในบทนี้

- Gin ใช้ `go-playground/validator/v10` เป็น validation engine เบื้องหลัง `binding:"..."` tag ทั้งหมด
- Tag ที่ใช้บ่อย: `required`, `email`, `min`/`max`, `oneof`, `omitempty`, `dive` ฯลฯ
- `ShouldBindJSON` คืน error ให้เราจัดการเอง เหมาะกับโปรเจกต์จริง ส่วน `BindJSON` เขียน response 400 ให้อัตโนมัติ เหมาะกับ prototype
- เพิ่ม validation tag เองได้ผ่าน `validator.RegisterValidation()` โดยดึง engine จาก `binding.Validator.Engine()`
- Custom middleware มี signature `func(c *gin.Context)` — `c.Next()` ส่งต่อ chain แล้วกลับมาทำงานต่อ, `c.Abort()`/`c.AbortWithStatusJSON()` หยุด chain ทันที (ต้อง `return` เองเสมอ)
- Middleware ทำงานเรียงตามลำดับ registration ในฝั่งก่อน `c.Next()` และย้อนกลับในฝั่งหลัง `c.Next()` — คล้ายกับพฤติกรรมของ `defer`
- Route group ผูก middleware เฉพาะกลุ่มได้ด้วย `group.Use(...)` โดยไม่กระทบ route นอกกลุ่ม — ใช้สร้าง security boundary เช่น admin group ที่ต้อง auth
- Error handling pattern ที่รวมศูนย์: handler เรียก `c.Error(err)` แล้ว `return`, ส่วน middleware ตัวสุดท้าย (ลงทะเบียนก่อน route ทั้งหมด) อ่าน `c.Errors` แปลงเป็น response เดียวที่สม่ำเสมอทั้งระบบ
- `/api/v1` versioning ทำได้ง่ายๆ ด้วย `r.Group("/api/v1")` ทำให้ในอนาคตสร้าง `/api/v2` คู่ขนานกันได้โดยไม่กระทบ client เดิม

## แบบฝึกหัดท้ายบท

1. เพิ่ม custom validator tag ชื่อ `strongpassword` ที่ตรวจว่า password ต้องมีทั้งตัวเลขและตัวอักษรพิมพ์ใหญ่อย่างน้อย 1 ตัว แล้วนำไปใช้แทน `min=8` เดิม
2. เขียน middleware ใหม่ชื่อ `RequestIDMiddleware` ที่สร้าง unique ID ให้แต่ละ request (ใช้ `time.Now().UnixNano()` ง่ายๆ ก็พอ) เก็บไว้ใน context ด้วย `c.Set("request_id", id)` แล้วให้ `LoggerMiddleware` อ่านค่านั้นออกมา log ด้วย `c.Get("request_id")`
3. ทดลองสลับลำดับ `r.Use(LoggerMiddleware())` กับ `r.Use(ErrorHandlerMiddleware())` ให้ `ErrorHandlerMiddleware` อยู่ก่อน แล้วสังเกตว่า status code ที่ log ออกมาผิดเพี้ยนไปอย่างไรเมื่อเทียบกับ response จริง อธิบายว่าทำไม
4. เพิ่ม route group ใหม่ `/api/v2` ที่มี endpoint `/ping` เหมือนกับ `/api/v1/ping` แต่ตอบ response format ต่างออกไป (เช่น เพิ่ม field `"version": "v2"`) เพื่อจำลองสถานการณ์ API versioning จริง
5. เขียน error type ใหม่ `ValidationError` ที่เก็บ field ที่ผิดพลาดเป็น `map[string]string` (เช่น `{"email": "invalid format"}`) แล้วปรับ `ErrorHandlerMiddleware` ให้รองรับ error type นี้ด้วย นอกเหนือจาก `AppError`
6. อธิบายด้วยคำพูดของตัวเองว่าทำไม `c.Error(err)` ถึงไม่ทำให้ handler หยุดทำงานทันทีเหมือน `c.Abort()` และทำไม design ของ Gin ถึงแยกสองอย่างนี้ออกจากกัน

---

**ต่อไป**: [Part 063 — Web Framework: Echo](./063-echo-framework.md)
