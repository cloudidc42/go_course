# Part 061: Web Framework: Gin เบื้องต้น

> ภาคที่ 5: Web Development — ตอนที่ 6 จาก 15 (Part 56–70)

## สารบัญของบทนี้

1. ทำไมต้องมี Web Framework ทั้งที่ Go มี `net/http` อยู่แล้ว
2. Gin คืออะไร ทำไมถึงเป็น framework ที่นิยมที่สุดของ Go
3. ติดตั้ง Gin
4. Hello Gin: โปรแกรมแรก
5. `gin.Default()` vs `gin.New()`
6. การประกาศ Route: `GET`, `POST`, `PUT`, `DELETE`, `PATCH`
7. `gin.Context` เจาะลึก: `Param`, `Query`, `JSON`, `BindJSON`
8. Route Groups (`r.Group`)
9. ตัวอย่างเต็ม: Todo CRUD API ด้วย Gin
10. เทียบปริมาณโค้ด: Gin vs `net/http` ดิบ
11. สรุปสิ่งที่ได้เรียนในบทนี้
12. แบบฝึกหัดท้ายบท

---

## 1. ทำไมต้องมี Web Framework ทั้งที่ Go มี `net/http` อยู่แล้ว

ใน **Part 056** เราเขียน HTTP server ด้วย `net/http` ล้วนๆ และใน **Part 057** เราสร้าง middleware pattern ขึ้นมาเองด้วยมือ ทั้งสองบทนั้นทำให้เห็นว่า `net/http` ของ Go นั้น "ครบ" ในความหมายที่ว่ามันมีทุกอย่างที่จำเป็นต่อการสร้างเว็บเซิร์ฟเวอร์ แต่ก็ "ดิบ" ในความหมายที่ว่าทุกอย่างที่เกินพื้นฐานเราต้องเขียนเอง เช่น:

- **Path parameter** (`/users/:id`) — `net/http` มาตรฐาน (ก่อน Go 1.22) ไม่รองรับโดยตรง ต้อง parse `r.URL.Path` เอง หรือใช้ router ภายนอกอย่าง `gorilla/mux`/`chi` ที่เรียนไปใน **Part 058**
- **Route grouping / versioning** — ต้องเขียน wrapper เอง
- **Binding JSON request body เข้า struct พร้อม validation** — ต้อง `json.NewDecoder` เอง แล้วเขียน validation logic เอง
- **Middleware chain ที่ประกอบกันง่าย** — ทำได้ แต่ต้องเขียน pattern decorator เอง (ตามที่ทำใน Part 057)
- **Error handling ที่รวมศูนย์** — ต้องออกแบบเอง

Web framework อย่าง **Gin**, **Echo** (Part 063), และ **Fiber** (Part 064) ถูกสร้างขึ้นมาเพื่อลด boilerplate เหล่านี้ โดยมี 3 สิ่งหลักที่ framework มอบให้เรา:

1. **Routing tree ที่มีประสิทธิภาพสูง** — ใช้โครงสร้างข้อมูลแบบ [radix tree](https://en.wikipedia.org/wiki/Radix_tree) ทำให้ match route (รวมถึง path parameter และ wildcard) ได้เร็วมาก แม้มีหลายพันเส้นทาง
2. **Context object ที่รวมทุกอย่างไว้ที่เดียว** — ไม่ต้องสลับไปมาระหว่าง `http.ResponseWriter` และ `*http.Request` แยกกัน
3. **Ecosystem ของ middleware สำเร็จรูป** — CORS, rate limiting, JWT, logging, request ID ฯลฯ ที่มีคนเขียนไว้ให้แล้วและใช้งานร่วมกันได้เป็นมาตรฐาน

สิ่งสำคัญที่ต้องเข้าใจคือ **framework เหล่านี้ไม่ได้แทนที่ `net/http`** — Gin, Echo และ chi (แต่ไม่ใช่ Fiber ซึ่งเราจะพูดถึงข้อยกเว้นนี้ใน Part 064) ล้วนสร้างอยู่**บนฐานของ** `net/http` ทั้งสิ้น `*gin.Engine` เองก็ implement interface `http.Handler` (มี method `ServeHTTP`) ทำให้เอาไปใช้กับ `http.Server` มาตรฐานได้โดยตรง สิ่งที่ framework ทำคือห่อ (wrap) ความสามารถของ `net/http` ให้ใช้งานสะดวกขึ้น ไม่ใช่เขียน HTTP stack ใหม่ทั้งหมด (ต่างจาก Fiber ที่เลือกใช้ `fasthttp` แทน)

---

## 2. Gin คืออะไร ทำไมถึงเป็น framework ที่นิยมที่สุดของ Go

[Gin](https://github.com/gin-gonic/gin) เป็น HTTP web framework ที่เขียนด้วย Go เปิดตัวปี 2014 ปัจจุบันมีดาวบน GitHub มากกว่า 80,000 ดวง และเป็น framework ที่ถูกใช้งานมากที่สุดในระบบนิเวศ Go เหตุผลหลักที่ทำให้ Gin ได้รับความนิยม:

- **Performance สูง**: ใช้ [httprouter](https://github.com/julienschmidt/httprouter) เป็นแรงบันดาลใจในการออกแบบ routing แบบ radix tree ทำให้ routing เร็วกว่า `net/http` เปล่าๆ ที่ใช้ `ServeMux` แบบ linear/pattern matching (ในเวอร์ชันเก่าก่อน Go 1.22)
- **API เรียบง่าย อ่านง่าย**: `gin.Context` รวมทุกอย่างที่ handler ต้องใช้ไว้ในที่เดียว
- **Middleware ecosystem ใหญ่**: เพราะได้รับความนิยมมานาน จึงมี middleware สำเร็จรูปให้เลือกใช้เยอะมาก (JWT, CORS, rate limit, Swagger, Prometheus ฯลฯ)
- **Built-in validation** ผ่าน struct tag โดยใช้ [go-playground/validator](https://github.com/go-playground/validator) อยู่ข้างใต้ (จะเรียนเจาะลึกใน Part 062)
- **เอกสารและตัวอย่างเยอะ**: เพราะได้รับความนิยมมานาน หา resource เรียนรู้และ community support ได้ง่าย

Gin เหมาะกับทีมที่ต้องการสร้าง REST API ได้เร็ว มี ecosystem รองรับครบ และไม่ต้องกังวลเรื่อง compatibility กับ middleware ของ `net/http` (เพราะ Gin ใช้ `net/http` เป็นฐานอยู่แล้ว)

---

## 3. ติดตั้ง Gin

เริ่มจากสร้างโปรเจกต์ใหม่และติดตั้ง Gin ผ่าน `go get`:

```bash
mkdir gin-todo-api
cd gin-todo-api
go mod init gin-todo-api
go get github.com/gin-gonic/gin
```

ผลลัพธ์จริงที่ได้เมื่อรันคำสั่งข้างต้น (Go 1.24, ตัดบางบรรทัดออกเพื่อความกระชับ):

```
go: creating new go.mod: module gin-todo-api
go: added github.com/bytedance/sonic v1.15.0
go: added github.com/gin-gonic/gin v1.12.0
go: added github.com/go-playground/validator/v10 v10.30.1
go: added github.com/goccy/go-json v0.10.5
go: added github.com/pelletier/go-toml/v2 v2.2.4
go: added github.com/ugorji/go/codec v1.3.1
go: added golang.org/x/net v0.51.0
... (dependency อื่นๆ ที่ Gin ใช้ภายใน)
```

สังเกตว่า Gin ดึง dependency มาค่อนข้างเยอะ เพราะภายในมันรวม JSON encoder ประสิทธิภาพสูงหลายตัว (`sonic`, `go-json`) ให้เลือกใช้ตาม build tag, ตัว validator (`go-playground/validator`) สำหรับ struct tag validation, และ `go-toml` สำหรับ config binding — ทั้งหมดนี้คือสิ่งที่เราจะได้ "ฟรี" เมื่อใช้ Gin แทนที่จะต้องเลือกและประกอบเองทีละตัวเหมือนตอนใช้ `net/http` ล้วนๆ

ตรวจสอบว่าติดตั้งสำเร็จได้จาก `go.mod`:

```
module gin-todo-api

go 1.24.0

require github.com/gin-gonic/gin v1.12.0
```

---

## 4. Hello Gin: โปรแกรมแรก

```go
package main

import (
	"net/http"

	"github.com/gin-gonic/gin"
)

func main() {
	r := gin.Default()

	r.GET("/ping", func(c *gin.Context) {
		c.JSON(http.StatusOK, gin.H{
			"message": "pong",
		})
	})

	r.Run(":8080") // เทียบเท่า http.ListenAndServe(":8080", r)
}
```

รันด้วย:

```bash
go run main.go
```

ผลลัพธ์ตอน start (ของจริงจากการรัน):

```
[GIN-debug] [WARNING] Creating an Engine instance with the Logger and Recovery middleware already attached.

[GIN-debug] [WARNING] Running in "debug" mode. Switch to "release" mode in production.
 - using env:	export GIN_MODE=release
 - using code:	gin.SetMode(gin.ReleaseMode)

[GIN-debug] GET    /ping                     --> main.main.func1 (3 handlers)
[GIN-debug] [WARNING] You trusted all proxies, this is NOT safe. We recommend you to set a value.
Please check https://github.com/gin-gonic/gin/blob/master/docs/doc.md#dont-trust-all-proxies for details.
[GIN-debug] Listening and serving HTTP on :8080
```

ทดสอบด้วย `curl` อีก terminal หนึ่ง:

```bash
curl http://localhost:8080/ping
```

```json
{"message":"pong"}
```

จุดที่ควรสังเกตจากบรรทัด log:

- **`[GIN-debug]`**: Gin มี debug logger ในตัวที่ log ทุก route ที่ลงทะเบียนไว้ตอน start และทุก request ที่เข้ามา (คล้าย middleware logging ที่เราเขียนเองใน Part 057 แต่ Gin ให้มาเป็นค่า default)
- **คำเตือนเรื่อง "release" mode**: ใน production ควรตั้ง `gin.SetMode(gin.ReleaseMode)` หรือ environment variable `GIN_MODE=release` เพื่อปิด debug log ที่มี overhead และเปิดเผยข้อมูล route มากเกินไป
- **คำเตือนเรื่อง trusted proxies**: เกี่ยวกับการอ่าน header `X-Forwarded-For` เพื่อหา IP จริงของ client เมื่ออยู่หลัง reverse proxy/load balancer — ควรตั้งค่า `r.SetTrustedProxies([]string{...})` ให้ตรงกับ IP ของ proxy จริงใน production
- **`gin.H`**: เป็นแค่ `type H map[string]interface{}` — เป็น shortcut ที่ Gin สร้างไว้ให้สร้าง JSON response แบบ ad-hoc ได้กระชับ ไม่ต้องประกาศ struct ทุกครั้ง

---

## 5. `gin.Default()` vs `gin.New()`

Gin มีสอง constructor สำหรับสร้าง engine:

| | `gin.Default()` | `gin.New()` |
|---|---|---|
| Middleware ที่ผูกมาให้อัตโนมัติ | `Logger()` + `Recovery()` | ไม่มีเลย (เปล่าทั้งหมด) |
| เหมาะกับ | เริ่มโปรเจกต์ใหม่ทั่วไป, prototype, งานที่ต้องการ log + panic recovery ทันที | ต้องการควบคุม middleware chain เองทั้งหมด (เช่น ใช้ structured logger ของตัวเอง แทน `gin.Logger()`) |

โค้ดจริงของ `gin.Default()` (จากซอร์สของ Gin) มีค่าเทียบเท่ากับ:

```go
func Default() *Engine {
	engine := New()
	engine.Use(Logger(), Recovery())
	return engine
}
```

`Recovery()` middleware สำคัญมาก — มันคือตัวที่ `recover()` panic ที่เกิดขึ้นใน handler แล้วแปลงเป็น HTTP 500 แทนที่จะทำให้ทั้ง process ล่ม (เชื่อมโยงกับสิ่งที่เรียนใน **Part 017: Panic, Recover**) ถ้าใช้ `gin.New()` แล้วลืมใส่ `Recovery()` เอง หนึ่ง panic ใน handler เดียวจะทำให้ **ทั้งเซิร์ฟเวอร์ล่มทันที** ไม่ใช่แค่ request นั้น request เดียว

ตัวอย่างการใช้ `gin.New()` แล้วประกอบ middleware เอง (รูปแบบที่นิยมใน production เพื่อคุมลำดับ middleware อย่างชัดเจน):

```go
r := gin.New()
r.Use(gin.Logger())
r.Use(gin.Recovery())
// ต่อด้วย middleware ของเราเอง เช่น request ID, auth ฯลฯ (จะเรียนเจาะลึกใน Part 062)
```

รูปแบบนี้จะกลับมาใช้จริงจังใน **Part 062** ตอนที่เราเขียน custom middleware เอง (`LoggerMiddleware`, `AuthMiddleware`, `ErrorHandlerMiddleware`) และต้องคุมลำดับการ `Use()` อย่างระมัดระวัง

---

## 6. การประกาศ Route: `GET`, `POST`, `PUT`, `DELETE`, `PATCH`

`*gin.Engine` (และ `*gin.RouterGroup` ที่จะเรียนในหัวข้อถัดไป) มี method ตรงตัวสำหรับแต่ละ HTTP method:

```go
r.GET("/users", listUsers)
r.POST("/users", createUser)
r.GET("/users/:id", getUser)
r.PUT("/users/:id", updateUser)
r.PATCH("/users/:id", patchUser)
r.DELETE("/users/:id", deleteUser)
r.HEAD("/users", headUsers)
r.OPTIONS("/users", optionsUsers)

// รับได้ทุก method (ใช้น้อย มักใช้กับ webhook หรือ catch-all)
r.Any("/webhook", handleWebhook)
```

### Path parameter: `:id`

`:id` คือ **named parameter** — ส่วนของ path ที่เป็น dynamic segment เดียว (จับคู่ path segment เดียวเท่านั้น ไม่ใช่หลาย segment) เช่น `/users/:id` จะ match `/users/42` และ `/users/abc` แต่ไม่ match `/users/42/edit`

### Wildcard parameter: `*action`

ถ้าต้องการจับ path หลาย segment ต่อท้าย ใช้ wildcard ที่ขึ้นต้นด้วย `*`:

```go
r.GET("/files/*filepath", func(c *gin.Context) {
	filepath := c.Param("filepath") // ได้ "/images/logo.png" รวม / ข้างหน้า
	c.String(http.StatusOK, "requested file: %s", filepath)
})
```

> **ข้อควรระวังเรื่อง route conflict**: เวอร์ชันเก่าของ Gin (ที่ใช้ httprouter tree แบบดั้งเดิม) ห้ามลงทะเบียน static route กับ named parameter ที่ path เดียวกันแบบขัดแย้งกัน เช่น `/users/new` กับ `/users/:id` พร้อมกันจะ panic ตอน start เพราะ router ตัดสินใจไม่ได้ว่า `/users/new` ควรไปทาง static หรือ parameter — Gin เวอร์ชันปัจจุบัน (1.9+) รองรับกรณีนี้ได้ดีขึ้นมากด้วยการจัดลำดับความสำคัญให้ static route มาก่อนเสมอ แต่การออกแบบ route ให้ชัดเจนไม่กำกวมตั้งแต่ต้นก็ยังเป็นแนวทางที่ดีกว่า

---

## 7. `gin.Context` เจาะลึก: `Param`, `Query`, `JSON`, `BindJSON`

`*gin.Context` คือหัวใจของ Gin ทุก handler จะได้รับ `*gin.Context` เป็น parameter เดียว และใช้มันทำทุกอย่าง: อ่าน request, เขียน response, ส่งต่อค่าให้ middleware อื่น ฯลฯ

### `c.Param(key string) string` — อ่าน path parameter

```go
r.GET("/users/:id", func(c *gin.Context) {
	id := c.Param("id") // string เสมอ ต้องแปลงเองถ้าต้องการ int
	c.String(http.StatusOK, "user id = %s", id)
})
```

### `c.Query(key string) string` — อ่าน query string (`?key=value`)

```go
r.GET("/search", func(c *gin.Context) {
	q := c.Query("q")                       // "" ถ้าไม่มี
	page := c.DefaultQuery("page", "1")     // ให้ default value ถ้าไม่มี
	c.JSON(http.StatusOK, gin.H{"query": q, "page": page})
})
```

```bash
curl "http://localhost:8080/search?q=golang"
# {"page":"1","query":"golang"}
```

### `c.JSON(code int, obj any)` — เขียน JSON response

```go
c.JSON(http.StatusOK, gin.H{"message": "ok"})
// เทียบเท่า net/http แบบดิบ:
//   w.Header().Set("Content-Type", "application/json")
//   w.WriteHeader(http.StatusOK)
//   json.NewEncoder(w).Encode(map[string]string{"message": "ok"})
```

`c.JSON` จัดการ `Content-Type` header, การ set status code, และการ marshal เป็น JSON ให้ในบรรทัดเดียว — เทียบกับที่เราต้องเขียนแยก 3 บรรทัดตอนใช้ `net/http` ใน Part 060

### `c.BindJSON` vs `c.ShouldBindJSON`

ทั้งคู่ทำหน้าที่ **decode JSON body เข้า struct ที่กำหนด** แต่มีพฤติกรรมตอน error ต่างกัน:

```go
type CreateUserRequest struct {
	Name  string `json:"name" binding:"required"`
	Email string `json:"email" binding:"required,email"`
}

r.POST("/users", func(c *gin.Context) {
	var req CreateUserRequest
	if err := c.BindJSON(&req); err != nil {
		// BindJSON เขียน response 400 กับ error message ให้อัตโนมัติแล้ว!
		// เราไม่จำเป็นต้อง c.JSON เองอีก — แค่ return ออกจาก handler พอ
		return
	}
	c.JSON(http.StatusCreated, req)
})
```

`c.BindJSON` เป็น shortcut ของ `c.MustBindWith(obj, binding.JSON)` — ถ้า bind ไม่ผ่าน มันจะเรียก `c.AbortWithError(400, err)` ให้อัตโนมัติทันที ซึ่งสะดวกสำหรับกรณีทั่วไป แต่ทำให้เรา**ควบคุม response format ตอน error เองไม่ได้**

ส่วน `c.ShouldBindJSON` จะ **คืน error กลับมาให้เราจัดการเอง** ไม่เขียน response ให้อัตโนมัติ — เราจะเห็นความแตกต่างนี้ชัดเจนขึ้นและเหตุผลว่าทำไมโปรเจกต์จริงมักเลือกใช้ `ShouldBindJSON` ใน **Part 062** ตอนพูดถึง error handling middleware แบบรวมศูนย์

---

## 8. Route Groups (`r.Group`)

เมื่อ API มีหลาย endpoint ที่ขึ้นต้นด้วย prefix เดียวกัน (เช่น `/api/v1/...`) หรือต้องการ middleware เฉพาะกลุ่ม (เช่น เฉพาะ `/admin/...` ต้องผ่าน auth) เราใช้ `Group()`:

```go
v1 := r.Group("/api/v1")
{
	v1.GET("/todos", listTodos)
	v1.POST("/todos", createTodo)

	admin := v1.Group("/admin")
	admin.Use(AuthMiddleware()) // middleware เฉพาะ group นี้เท่านั้น
	{
		admin.GET("/stats", getStats)
	}
}
```

การใช้ `{ }` ครอบ block ไม่มีผลทาง syntax ใดๆ กับ Go (มันไม่ใช่ scope พิเศษ) แต่เป็น**ธรรมเนียมการเขียนโค้ด** (convention) ที่ชุมชน Gin ใช้กันเพื่อให้เห็นด้วยตาว่า route กลุ่มไหนอยู่ใน group ไหนเวลาอ่านโค้ด เราจะใช้ pattern นี้ตลอดทั้งบทและใน Part 062 ที่จะเพิ่ม group-level middleware สำหรับ authentication

`Group()` คืนค่าเป็น `*gin.RouterGroup` ซึ่งมี method `GET`, `POST`, `Group`, `Use` เหมือนกับ `*gin.Engine` ทุกประการ (เพราะ `*gin.Engine` embed `RouterGroup` อยู่ข้างใน) ทำให้ **group ซ้อน group ได้ไม่จำกัดชั้น** ตามตัวอย่างข้างบนที่มี `admin` ซ้อนอยู่ใน `v1` อีกที

---

## 9. ตัวอย่างเต็ม: Todo CRUD API ด้วย Gin

ตอนนี้มาสร้าง API เดียวกับที่ทำใน **Part 060** (Todo CRUD ด้วย JSON) ใหม่ทั้งหมดด้วย Gin เพื่อเทียบกันตรงๆ

โครงสร้างโปรเจกต์:

```
gin-todo-api/
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

	"github.com/gin-gonic/gin"
)

// ---------- Model ----------

type Todo struct {
	ID    int    `json:"id"`
	Title string `json:"title" binding:"required"`
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

// ---------- main ----------

func main() {
	store := NewTodoStore()
	store.Create(Todo{Title: "เรียน Go", Done: false})
	store.Create(Todo{Title: "เขียน Gin CRUD", Done: false})

	r := gin.Default()

	api := r.Group("/api/v1")
	{
		todos := api.Group("/todos")
		{
			todos.GET("", func(c *gin.Context) {
				c.JSON(http.StatusOK, store.List())
			})

			todos.GET("/:id", func(c *gin.Context) {
				id, err := strconv.Atoi(c.Param("id"))
				if err != nil {
					c.JSON(http.StatusBadRequest, gin.H{"error": "invalid id"})
					return
				}
				t, err := store.Get(id)
				if err != nil {
					c.JSON(http.StatusNotFound, gin.H{"error": err.Error()})
					return
				}
				c.JSON(http.StatusOK, t)
			})

			todos.POST("", func(c *gin.Context) {
				var t Todo
				if err := c.ShouldBindJSON(&t); err != nil {
					c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
					return
				}
				created := store.Create(t)
				c.JSON(http.StatusCreated, created)
			})

			todos.PUT("/:id", func(c *gin.Context) {
				id, err := strconv.Atoi(c.Param("id"))
				if err != nil {
					c.JSON(http.StatusBadRequest, gin.H{"error": "invalid id"})
					return
				}
				var t Todo
				if err := c.ShouldBindJSON(&t); err != nil {
					c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
					return
				}
				updated, err := store.Update(id, t)
				if err != nil {
					c.JSON(http.StatusNotFound, gin.H{"error": err.Error()})
					return
				}
				c.JSON(http.StatusOK, updated)
			})

			todos.DELETE("/:id", func(c *gin.Context) {
				id, err := strconv.Atoi(c.Param("id"))
				if err != nil {
					c.JSON(http.StatusBadRequest, gin.H{"error": "invalid id"})
					return
				}
				if err := store.Delete(id); err != nil {
					c.JSON(http.StatusNotFound, gin.H{"error": err.Error()})
					return
				}
				c.Status(http.StatusNoContent)
			})
		}
	}

	r.GET("/search", func(c *gin.Context) {
		q := c.Query("q")
		c.JSON(http.StatusOK, gin.H{"query": q})
	})

	r.Run(":8080")
}
```

### ทดสอบจริงด้วย `curl` (ผลลัพธ์ของจริงจากการรัน)

```bash
curl http://localhost:8080/api/v1/todos
```
```json
[{"id":1,"title":"เรียน Go","done":false},{"id":2,"title":"เขียน Gin CRUD","done":false}]
```

```bash
curl -X POST http://localhost:8080/api/v1/todos \
  -H "Content-Type: application/json" \
  -d '{"title":"ทดสอบ Gin","done":false}'
```
```json
{"id":3,"title":"ทดสอบ Gin","done":false}
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

ทดสอบกรณี validation ล้มเหลว (ไม่ส่ง `title` มา ซึ่งมี `binding:"required"`):

```bash
curl -X POST http://localhost:8080/api/v1/todos \
  -H "Content-Type: application/json" -d '{"done":false}'
```
```json
{"error":"Key: 'Todo.Title' Error:Field validation for 'Title' failed on the 'required' tag"}
```

สังเกตว่าแค่เพิ่ม struct tag `binding:"required"` บรรทัดเดียว เราก็ได้ validation แบบ 400 Bad Request พร้อม error message ที่อธิบายได้ว่า field ไหนผิดเงื่อนไขอะไร — โดยไม่ต้องเขียน `if req.Title == "" { ... }` เอง เราจะเรียนเรื่อง validation tag แบบเจาะลึกกว่านี้มากใน **Part 062**

---

## 10. เทียบปริมาณโค้ด: Gin vs `net/http` ดิบ

ลองเทียบเฉพาะส่วน routing + handler ของ endpoint เดียว — "ดึง todo ตาม id" — ระหว่างสองแนวทาง:

**`net/http` ล้วนๆ (แนวทางจาก Part 056/060)**

```go
mux.HandleFunc("GET /api/v1/todos/{id}", func(w http.ResponseWriter, r *http.Request) {
	idStr := r.PathValue("id") // ต้องใช้ Go 1.22+ ถึงมี PathValue จาก ServeMux
	id, err := strconv.Atoi(idStr)
	if err != nil {
		w.Header().Set("Content-Type", "application/json")
		w.WriteHeader(http.StatusBadRequest)
		json.NewEncoder(w).Encode(map[string]string{"error": "invalid id"})
		return
	}
	t, err := store.Get(id)
	if err != nil {
		w.Header().Set("Content-Type", "application/json")
		w.WriteHeader(http.StatusNotFound)
		json.NewEncoder(w).Encode(map[string]string{"error": err.Error()})
		return
	}
	w.Header().Set("Content-Type", "application/json")
	w.WriteHeader(http.StatusOK)
	json.NewEncoder(w).Encode(t)
})
```

**Gin**

```go
todos.GET("/:id", func(c *gin.Context) {
	id, err := strconv.Atoi(c.Param("id"))
	if err != nil {
		c.JSON(http.StatusBadRequest, gin.H{"error": "invalid id"})
		return
	}
	t, err := store.Get(id)
	if err != nil {
		c.JSON(http.StatusNotFound, gin.H{"error": err.Error()})
		return
	}
	c.JSON(http.StatusOK, t)
})
```

| หัวข้อเปรียบเทียบ | `net/http` ดิบ | Gin |
|---|---|---|
| set `Content-Type` + `WriteHeader` + `Encode` แยก 3 บรรทัดทุกครั้ง | ใช่ ต้องทำเอง | ไม่ต้อง — `c.JSON()` บรรทัดเดียว |
| อ่าน path parameter | ต้องมี Go 1.22+ ถึงมี `PathValue`, หรือใช้ router ภายนอก | `c.Param()` ใช้ได้เสมอ ไม่ขึ้นกับ Go version |
| Route grouping / versioning | ต้องเขียน wrapper เอง หรือใช้ `http.StripPrefix` | `r.Group("/api/v1")` มีให้ในตัว |
| Binding + validation JSON body | เขียน decode + validate logic เอง | `binding:"required"` tag บรรทัดเดียว |
| จำนวนบรรทัดของ handler เดียวกัน | ~16 บรรทัด | ~9 บรรทัด |

จำนวนบรรทัดต่อ handler ลดลงประมาณ **40-45%** และที่สำคัญกว่าจำนวนบรรทัดคือ **ความสม่ำเสมอ** — ทุก handler ใน Gin เขียนรูปแบบเดียวกันหมด (`c.JSON(status, data)`) ทำให้โค้ด base ขนาดใหญ่ที่มีหลายสิบ endpoint อ่านและ maintain ง่ายกว่ามาก และยิ่งเห็นชัดเจนขึ้นเมื่อโปรเจกต์โตขึ้นและต้องมี middleware, validation, grouping จำนวนมาก — ซึ่งคือสิ่งที่ `net/http` ดิบไม่ได้ให้มาฟรีๆ ต้องประกอบเองทั้งหมดแบบที่ทำใน Part 057-058

อย่างไรก็ตาม การแลกเปลี่ยนคือ **dependency ภายนอกเพิ่มขึ้น** (`go.sum` ยาวขึ้นมาก), **binary size ใหญ่ขึ้น**, และ **ต้องเรียนรู้ API เฉพาะของ framework** เพิ่มจาก `net/http` มาตรฐาน — สำหรับโปรเจกต์เล็กมากๆ หรืองานที่ต้องการ dependency น้อยที่สุด (เช่น CLI tool ที่มี HTTP server เสริม) `net/http` ล้วนๆ ยังคงเป็นตัวเลือกที่สมเหตุสมผล

---

## สรุปสิ่งที่ได้เรียนในบทนี้

- Web framework อย่าง Gin ไม่ได้แทนที่ `net/http` แต่สร้างอยู่บนฐานของมัน เพื่อลด boilerplate เรื่อง routing, binding, grouping
- ติดตั้งด้วย `go get github.com/gin-gonic/gin`
- `gin.Default()` ผูก `Logger()` + `Recovery()` มาให้อัตโนมัติ ส่วน `gin.New()` เริ่มจากศูนย์ให้เราคุมเอง
- Route ประกาศด้วย method ตรงตัว (`r.GET`, `r.POST`, ...) รองรับ path parameter (`:id`) และ wildcard (`*filepath`)
- `gin.Context` คือหัวใจของ Gin: `c.Param()` อ่าน path parameter, `c.Query()` อ่าน query string, `c.JSON()` เขียน JSON response, `c.BindJSON`/`c.ShouldBindJSON` decode request body พร้อม validation
- `c.BindJSON` เขียน error response ให้อัตโนมัติเมื่อ bind ล้มเหลว ส่วน `c.ShouldBindJSON` คืน error ให้เราจัดการเอง — รายละเอียดเพิ่มเติมใน Part 062
- `r.Group()` ใช้จัดกลุ่ม route ตาม prefix และผูก middleware เฉพาะกลุ่มได้ ซ้อนกันได้หลายชั้น
- เทียบกับ `net/http` ดิบแล้ว Gin ลดจำนวนโค้ดต่อ handler ได้มาก โดยเฉพาะเรื่อง JSON response และ validation แลกกับ dependency และการเรียนรู้ API เฉพาะทาง

## แบบฝึกหัดท้ายบท

1. สร้างโปรเจกต์ Gin ใหม่ ติดตั้งและรันตัวอย่าง `/ping` ให้สำเร็จ แล้วลองเปลี่ยนเป็น `gin.New()` พร้อมเพิ่ม `gin.Logger()` และ `gin.Recovery()` ด้วยตัวเอง สังเกตว่า log ที่ได้เหมือนกับ `gin.Default()` หรือไม่
2. เพิ่ม endpoint `GET /api/v1/todos/:id/toggle` ที่สลับค่า `Done` ของ todo ตาม id ที่ระบุ (ถ้าเป็น `true` ให้เป็น `false` และในทางกลับกัน) โดยไม่ต้องรับ body
3. ลองลบ `binding:"required"` ออกจาก field `Title` แล้วยิง POST โดยไม่ส่ง `title` มา สังเกตว่า Gin ยอมรับ request นั้นหรือไม่ และ response ที่ได้ต่างจากเดิมอย่างไร
4. เขียน endpoint ใหม่ `GET /api/v1/todos/filter?done=true` ที่ใช้ `c.Query("done")` กรองเฉพาะ todo ที่ done ตรงกับค่าที่ระบุ (ใบ้: ต้องแปลง string เป็น bool ด้วย `strconv.ParseBool`)
5. ลองสร้าง route ที่ conflict กันโดยตั้งใจ เช่น `r.GET("/users/new", ...)` และ `r.GET("/users/:id", ...)` พร้อมกัน รันดูว่าเกิดอะไรขึ้น แล้วอธิบายว่าทำไม
6. เขียนโปรแกรม `net/http` ล้วนๆ (ไม่ใช้ Gin) ที่ทำ endpoint เดียวกับข้อ 2 แล้วเทียบจำนวนบรรทัดโค้ดกับเวอร์ชัน Gin ด้วยตัวเอง

---

**ต่อไป**: [Part 062 — Gin ขั้นสูง: Validation, Binding, Grouping](./062-gin-advanced.md)
