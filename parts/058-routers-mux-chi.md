# Part 058: Router ยอดนิยม: `gorilla/mux`, `chi`

> ภาคที่ 5: Web Development — ตอนที่ 3 จาก 15 (Part 56–70)

## สารบัญของบทนี้

1. ทำไม third-party router ถึงเคยเป็นที่นิยมมาก
2. รู้จัก `chi`: router ที่ยังคง active และเป็นที่นิยมที่สุดในปัจจุบัน
3. ติดตั้งและเริ่มต้นใช้งาน `chi`
4. Route Group และ Subrouter ด้วย `r.Route`
5. ดึงค่า URL Parameter ด้วย `chi.URLParam`
6. Middleware ในตัวของ chi: `middleware.Logger`, `middleware.Recoverer`
7. ตัวอย่างเต็มรูปแบบ: Book + Review API ด้วย chi
8. ทดสอบด้วย `curl`
9. รู้จัก `gorilla/mux` และสถานะ maintenance mode ในปัจจุบัน
10. ตารางเปรียบเทียบ: stdlib ServeMux vs chi vs gorilla/mux
11. เมื่อไหร่ที่ third-party router ยังคุ้มค่าอยู่
12. สรุปสิ่งที่ได้เรียนในบทนี้
13. แบบฝึกหัดท้ายบท

---

## 1. ทำไม third-party router ถึงเคยเป็นที่นิยมมาก

**Part 056** แสดงให้เห็นว่า Go 1.22+ ทำให้ `http.ServeMux` มาตรฐานรองรับ pattern แบบ `"GET /notes/{id}"` ได้ในตัวแล้ว แต่ก่อนหน้า Go 1.22 (กุมภาพันธ์ 2024) ความสามารถนี้**ไม่มีอยู่เลย** — นักพัฒนา Go ต้องพึ่ง third-party router เพื่อให้ได้ฟีเจอร์พื้นฐานที่ REST API สมัยใหม่ต้องการ:

- **Route parameter** เช่น `/users/{id}` ที่ดึงค่าออกมาได้ตรงๆ
- **Match ตาม HTTP method** ในตัว ไม่ต้องเช็ค `r.Method` เอง
- **Regex constraint** เช่น จำกัดว่า `{id}` ต้องเป็นตัวเลขเท่านั้น `/users/{id:[0-9]+}`
- **Subrouter / route grouping** เพื่อจัดกลุ่ม route ที่มี prefix หรือ middleware ร่วมกัน
- **Middleware chaining ในตัว** ผ่าน method อย่าง `.Use()`

router สองตัวที่ได้รับความนิยมสูงสุดในยุคนั้นคือ **`gorilla/mux`** (เก่าแก่ที่สุด เปิดตัวปี 2012) และ **`chi`** (เปิดตัวปี 2016 เน้นความเบาและเข้ากับ `net/http` โดยตรง) แม้ปัจจุบัน stdlib จะตามทันในเรื่อง routing พื้นฐานแล้ว แต่ third-party router ก็ยังมีที่ทางของมันอยู่ — บทนี้จะพาไปทำความรู้จักทั้งสองตัว โดยเน้นตัวอย่างจริงที่ `chi` เพราะเป็นตัวที่ยัง active และแนะนำให้ใช้มากที่สุดในปัจจุบัน

---

## 2. รู้จัก `chi`: router ที่ยังคง active และเป็นที่นิยมที่สุดในปัจจุบัน

[`chi`](https://github.com/go-chi/chi) เป็น router ที่ออกแบบตามปรัชญา **"100% compatible กับ `net/http`"** อย่างเคร่งครัด นั่นคือ `chi.Router` ก็คือ `http.Handler` ตัวหนึ่ง (ผ่าน interface embedding) และ middleware ของ chi ก็คือ `func(http.Handler) http.Handler` แบบเดียวกับที่เขียนเองใน **Part 057** ทุกประการ — ทำให้ผสมโค้ดที่เขียนด้วย stdlib ล้วนๆ กับโค้ดที่ใช้ chi เข้าด้วยกันได้อย่างไร้รอยต่อ

จุดเด่นของ chi เมื่อเทียบกับ router อื่น:

- **น้ำหนักเบามาก** — ไม่มี dependency ภายนอกเลยแม้แต่ตัวเดียว (pure stdlib)
- **Active maintenance** — ยังมีการอัปเดตสม่ำเสมอ รองรับ Go เวอร์ชันใหม่ๆ ต่อเนื่อง
- **Middleware มาตรฐานให้พร้อมใช้** ในแพ็กเกจย่อย `chi/middleware`
- **Subrouter/route grouping ที่ยืดหยุ่นมาก** เหมาะกับ API ที่มี nested resource เยอะ

---

## 3. ติดตั้งและเริ่มต้นใช้งาน `chi`

สร้างโปรเจกต์ใหม่และติดตั้ง chi:

```bash
mkdir chidemo && cd chidemo
go mod init chidemo
go get github.com/go-chi/chi/v5@latest
```

คำสั่งข้างต้นจะได้ผลประมาณนี้ (เวอร์ชันจริงอาจใหม่กว่านี้เมื่อคุณรันเอง):

```
go: downloading github.com/go-chi/chi/v5 v5.3.2
go: added github.com/go-chi/chi/v5 v5.3.2
```

> **ข้อสังเกต**: import path คือ `github.com/go-chi/chi/v5` (มี `/v5` ต่อท้าย) เพราะ chi เดิมมี major version 1 (`github.com/go-chi/chi`) แล้วยกเครื่องใหม่เป็น v5 ตามกฎ **semantic import versioning** ของ Go (major version ตั้งแต่ 2 ขึ้นไปต้องอยู่ใน import path) — เรื่องนี้จะเรียนเจาะลึกใน **Part 108: Go Modules Best Practices และ Semantic Versioning**

การสร้าง router ตัวแรกเทียบกับ `http.ServeMux`:

```go
package main

import (
	"log"
	"net/http"

	"github.com/go-chi/chi/v5"
)

func main() {
	r := chi.NewRouter() // เทียบเท่า http.NewServeMux()
	r.Get("/hello", func(w http.ResponseWriter, r *http.Request) {
		w.Write([]byte("hello from chi"))
	})
	log.Fatal(http.ListenAndServe(":8091", r)) // r คือ http.Handler ธรรมดา
}
```

สังเกตว่าแทนที่จะเขียน `mux.HandleFunc("GET /hello", ...)` แบบ stdlib chi มี method แยกตาม HTTP method ให้ตรงๆ: `r.Get`, `r.Post`, `r.Put`, `r.Delete`, `r.Patch` ฯลฯ

---

## 4. Route Group และ Subrouter ด้วย `r.Route`

จุดแข็งที่สุดของ chi คือการจัดกลุ่ม route ผ่าน `r.Route(pattern, func(r chi.Router))` ซึ่งสร้าง **subrouter** ที่ผูกกับ prefix นั้นโดยอัตโนมัติ ซ้อนกันได้หลายชั้นตามความลึกของ resource:

```go
r.Route("/books", func(r chi.Router) {
	r.Get("/", listBooks)   // GET /books
	r.Post("/", createBook) // POST /books

	r.Route("/{bookID}", func(r chi.Router) {
		r.Get("/", getBook) // GET /books/{bookID}

		// Subrouter ซ้อนอีกชั้น: /books/{bookID}/reviews
		r.Route("/reviews", func(r chi.Router) {
			r.Get("/", listBookReviews) // GET /books/{bookID}/reviews
		})
	})
})
```

รูปแบบนี้สะท้อนโครงสร้าง URL ของ nested resource ได้ตรงกับที่ตาเห็นเป๊ะ — อ่านโค้ดแล้วเห็นภาพ tree ของ endpoint ทั้งหมดทันที ต่างจาก stdlib `ServeMux` ที่ต้องเขียน pattern เต็มซ้ำทุกบรรทัด (`"GET /books/{bookID}/reviews"`) แม้ prefix จะซ้ำกันก็ตาม

---

## 5. ดึงค่า URL Parameter ด้วย `chi.URLParam`

เทียบเท่ากับ `r.PathValue("id")` ของ stdlib ที่เรียนใน Part 056 chi มีฟังก์ชันของตัวเองชื่อ `chi.URLParam(r *http.Request, key string) string`:

```go
func getBook(w http.ResponseWriter, r *http.Request) {
	// chi.URLParam ดึงค่าจาก route pattern ที่ประกาศด้วย {bookID}
	id, err := strconv.Atoi(chi.URLParam(r, "bookID"))
	if err != nil {
		writeError(w, http.StatusBadRequest, "invalid id")
		return
	}
	// ...
}
```

ข้อควรระวังเดียวกับ stdlib: ชื่อ parameter ที่ใช้ใน `chi.URLParam` ต้องตรงกับชื่อที่ประกาศใน pattern เป๊ะ (`{bookID}` ในตัวอย่างข้างบน) ถ้าพิมพ์ชื่อผิดหรือใช้ชื่อไม่ตรงกัน `chi.URLParam` จะคืนค่าเป็น string ว่างเปล่าเงียบๆ โดยไม่มี compile error หรือ panic ใดๆ เตือน (เป็นข้อผิดพลาดที่พบได้บ่อยเวลา refactor เปลี่ยนชื่อ parameter)

---

## 6. Middleware ในตัวของ chi: `middleware.Logger`, `middleware.Recoverer`

แพ็กเกจย่อย `github.com/go-chi/chi/v5/middleware` มี middleware มาตรฐานให้ใช้ทันทีโดยไม่ต้องเขียนเอง (เทียบกับที่เขียนมือใน **Part 057**):

```go
import "github.com/go-chi/chi/v5/middleware"

r := chi.NewRouter()
r.Use(middleware.Logger)    // เทียบเท่า Logging middleware ที่เขียนเองใน Part 057
r.Use(middleware.Recoverer) // เทียบเท่า Recovery middleware ที่เขียนเองใน Part 057
```

middleware ที่มีให้ใช้บ่อยๆ ใน `chi/middleware` ได้แก่:

| Middleware | หน้าที่ |
|---|---|
| `middleware.Logger` | log ทุก request พร้อม status, ขนาด response, เวลาที่ใช้ |
| `middleware.Recoverer` | recover panic แล้วตอบ 500 อัตโนมัติ |
| `middleware.RequestID` | สร้าง unique ID ต่อ request ใส่ใน context |
| `middleware.RealIP` | อ่าน IP จริงของ client จาก header เช่น `X-Forwarded-For` |
| `middleware.Timeout` | ยกเลิก request ที่ทำงานนานเกินกำหนดผ่าน context timeout |
| `middleware.Compress` | บีบอัด response ด้วย gzip อัตโนมัติ |

เพราะ middleware ของ chi ก็คือ `func(http.Handler) http.Handler` ธรรมดา middleware ที่เขียนเองใน Part 057 (เช่น `Auth`) จึงเอามาใช้กับ `r.Use()` ของ chi ได้ทันทีโดยไม่ต้องแก้อะไรเลย

---

## 7. ตัวอย่างเต็มรูปแบบ: Book + Review API ด้วย chi

มาสร้าง API ที่มี nested resource จริง: `Book` และ `Review` ที่ผูกกับ book แต่ละเล่ม เพื่อโชว์พลังของ subrouter:

```go
package main

import (
	"encoding/json"
	"log"
	"net/http"
	"strconv"
	"sync"

	"github.com/go-chi/chi/v5"
	"github.com/go-chi/chi/v5/middleware"
)

type Book struct {
	ID     int    `json:"id"`
	Title  string `json:"title"`
	Author string `json:"author"`
}

type Review struct {
	ID     int    `json:"id"`
	BookID int    `json:"book_id"`
	Score  int    `json:"score"`
	Text   string `json:"text"`
}

var (
	mu      sync.Mutex
	books   = map[int]Book{1: {1, "The Go Programming Language", "Donovan & Kernighan"}}
	reviews = map[int]Review{1: {1, 1, 5, "great book"}}
	nextID  = 2
)

func writeJSON(w http.ResponseWriter, status int, v any) {
	w.Header().Set("Content-Type", "application/json")
	w.WriteHeader(status)
	json.NewEncoder(w).Encode(v)
}

func writeError(w http.ResponseWriter, status int, msg string) {
	writeJSON(w, status, map[string]string{"error": msg})
}

func listBooks(w http.ResponseWriter, r *http.Request) {
	mu.Lock()
	defer mu.Unlock()
	result := make([]Book, 0, len(books))
	for _, b := range books {
		result = append(result, b)
	}
	writeJSON(w, http.StatusOK, result)
}

func getBook(w http.ResponseWriter, r *http.Request) {
	id, err := strconv.Atoi(chi.URLParam(r, "bookID"))
	if err != nil {
		writeError(w, http.StatusBadRequest, "invalid id")
		return
	}
	mu.Lock()
	b, ok := books[id]
	mu.Unlock()
	if !ok {
		writeError(w, http.StatusNotFound, "book not found")
		return
	}
	writeJSON(w, http.StatusOK, b)
}

func createBook(w http.ResponseWriter, r *http.Request) {
	var input struct {
		Title  string `json:"title"`
		Author string `json:"author"`
	}
	if err := json.NewDecoder(r.Body).Decode(&input); err != nil {
		writeError(w, http.StatusBadRequest, "invalid json body")
		return
	}
	mu.Lock()
	defer mu.Unlock()
	b := Book{ID: nextID, Title: input.Title, Author: input.Author}
	books[b.ID] = b
	nextID++
	writeJSON(w, http.StatusCreated, b)
}

func listBookReviews(w http.ResponseWriter, r *http.Request) {
	bookID, err := strconv.Atoi(chi.URLParam(r, "bookID"))
	if err != nil {
		writeError(w, http.StatusBadRequest, "invalid book id")
		return
	}
	mu.Lock()
	defer mu.Unlock()
	result := make([]Review, 0)
	for _, rv := range reviews {
		if rv.BookID == bookID {
			result = append(result, rv)
		}
	}
	writeJSON(w, http.StatusOK, result)
}

func main() {
	r := chi.NewRouter()
	r.Use(middleware.Logger)
	r.Use(middleware.Recoverer)

	r.Route("/books", func(r chi.Router) {
		r.Get("/", listBooks)
		r.Post("/", createBook)
		r.Route("/{bookID}", func(r chi.Router) {
			r.Get("/", getBook)
			r.Route("/reviews", func(r chi.Router) {
				r.Get("/", listBookReviews)
			})
		})
	})

	log.Println("listening on :8091")
	log.Fatal(http.ListenAndServe(":8091", r))
}
```

---

## 8. ทดสอบด้วย `curl`

ผลลัพธ์จริงจากการรันโค้ดข้างบน:

```bash
curl -s http://localhost:8091/books
```
```json
[{"id":1,"title":"The Go Programming Language","author":"Donovan & Kernighan"}]
```

```bash
curl -s -X POST http://localhost:8091/books -d '{"title":"Learning Go","author":"Bodner"}'
```
```json
{"id":2,"title":"Learning Go","author":"Bodner"}
```

```bash
curl -s http://localhost:8091/books/1
```
```json
{"id":1,"title":"The Go Programming Language","author":"Donovan & Kernighan"}
```

```bash
curl -s http://localhost:8091/books/1/reviews
```
```json
[{"id":1,"book_id":1,"score":5,"text":"great book"}]
```

```bash
curl -s -o /dev/null -w "%{http_code}\n" http://localhost:8091/books/999
```
```
404
```

log ฝั่ง server จาก `middleware.Logger` (สังเกตว่าให้รายละเอียดมากกว่า logging middleware ที่เขียนเองใน Part 057 โดยไม่ต้องเขียนอะไรเพิ่มเลย):

```
listening on :8091
"GET http://localhost:8091/books HTTP/1.1" from 127.0.0.1:53518 - 200 85B in 90.821µs
"POST http://localhost:8091/books HTTP/1.1" from 127.0.0.1:53524 - 201 49B in 129.779µs
"GET http://localhost:8091/books/1 HTTP/1.1" from 127.0.0.1:53534 - 200 83B in 35.102µs
"GET http://localhost:8091/books/1/reviews HTTP/1.1" from 127.0.0.1:53538 - 200 53B in 86.997µs
"GET http://localhost:8091/books/999 HTTP/1.1" from 127.0.0.1:53542 - 404 27B in 42.619µs
```

---

## 9. รู้จัก `gorilla/mux` และสถานะ maintenance mode ในปัจจุบัน

[`gorilla/mux`](https://github.com/gorilla/mux) เป็นหนึ่งใน router ที่เก่าแก่และมีอิทธิพลที่สุดในวงการ Go เปิดตัวมาตั้งแต่ปี 2012 เป็นส่วนหนึ่งของชุด **Gorilla Web Toolkit** ที่มีทั้ง `gorilla/mux`, `gorilla/websocket`, `gorilla/sessions` ฯลฯ ไวยากรณ์ของมันมีเอกลักษณ์เฉพาะตัว โดยเฉพาะการรองรับ **regex constraint** ในตัว pattern โดยตรง:

```go
package main

import (
	"log"
	"net/http"

	"github.com/gorilla/mux"
)

func getBook(w http.ResponseWriter, r *http.Request) {
	// gorilla/mux ดึงค่า path parameter ผ่าน mux.Vars(r) ซึ่งคืนเป็น map[string]string
	// (คนละแบบกับ chi.URLParam ของ chi หรือ r.PathValue ของ stdlib ที่รับชื่อ key ตรงๆ)
	vars := mux.Vars(r)
	w.Write([]byte("book id: " + vars["id"]))
}

func createBook(w http.ResponseWriter, r *http.Request) {
	w.Write([]byte("created"))
}

func statsHandler(w http.ResponseWriter, r *http.Request) {
	w.Write([]byte("stats"))
}

func main() {
	r := mux.NewRouter()
	// {id:[0-9]+} คือ regex constraint: เส้นทางนี้จะ match ก็ต่อเมื่อ id เป็น
	// ตัวเลขล้วนเท่านั้น — ถ้ายิง /books/abc จะไม่ match เลย (ตกไปเป็น 404)
	r.HandleFunc("/books/{id:[0-9]+}", getBook).Methods("GET")
	r.HandleFunc("/books", createBook).Methods("POST")

	sub := r.PathPrefix("/admin").Subrouter()
	sub.HandleFunc("/stats", statsHandler)

	log.Fatal(http.ListenAndServe(":8095", r))
}
```

ทดสอบจริง (ผลลัพธ์จากการรันโค้ดข้างต้น):

```bash
curl -s http://localhost:8095/books/1
```
```
book id: 1
```

```bash
curl -s -o /dev/null -w "%{http_code}\n" http://localhost:8095/books/abc
```
```
404
```

```bash
curl -s http://localhost:8095/admin/stats
```
```
stats
```

สังเกตว่า `/books/abc` ได้ 404 ทันทีโดยไม่ต้องเขียนโค้ด validate เองเลย เพราะ regex constraint ปฏิเสธมันตั้งแต่ชั้น routing — นี่คือจุดที่ `gorilla/mux` ทำได้ตรงๆ ในตัว pattern ในขณะที่ `chi` หรือ stdlib ต้อง validate เองใน handler (เหมือนที่ทำใน `strconv.Atoi` + เช็ค error ตลอดทั้งบท)

จุดเด่นเฉพาะตัวของ `gorilla/mux` ที่ `chi` ไม่มีให้ตรงๆ:

- **Regex constraint ในตัว pattern** — `{id:[0-9]+}` บังคับว่า `id` ต้องเป็นตัวเลขล้วนเท่านั้น ไม่ต้อง validate เองใน handler
- **Match ตาม header, query string, หรือ scheme** ได้ในตัว เช่น `r.Headers("X-Requested-With", "XMLHttpRequest")`
- **URL building แบบย้อนกลับ** — สร้าง URL จากชื่อ route ที่ตั้งไว้ (`r.Name("book").URL(...)`)

**อย่างไรก็ตาม** ตั้งแต่ปี 2022 เป็นต้นมา ทีมงาน Gorilla ได้ประกาศอย่างเป็นทางการว่าโปรเจกต์เข้าสู่ **"maintenance mode"** — หมายความว่าจะยังคง merge patch ด้าน security และรับ pull request เล็กๆ น้อยๆ แต่**จะไม่มีการพัฒนาฟีเจอร์ใหม่อีกต่อไป** นี่คือเหตุผลสำคัญที่ทำให้โปรเจกต์ใหม่ๆ ส่วนใหญ่หันไปใช้ `chi` แทน แม้ `gorilla/mux` จะยังใช้งานได้ปกติและปลอดภัยสำหรับโปรเจกต์ที่มีอยู่แล้ว (legacy) ก็ตาม

---

## 10. ตารางเปรียบเทียบ: stdlib ServeMux vs chi vs gorilla/mux

| คุณสมบัติ | `net/http.ServeMux` (Go 1.22+) | `chi` | `gorilla/mux` |
|---|---|---|---|
| ต้องติดตั้ง dependency เพิ่ม | ไม่ต้อง | ต้อง (แต่ dependency-free เอง) | ต้อง |
| Match ตาม HTTP method | ได้ (ในตัว) | ได้ | ได้ (ผ่าน `.Methods()`) |
| Path parameter (`{id}`) | ได้ (ในตัว) | ได้ | ได้ |
| Regex constraint บน parameter | ไม่ได้ | ไม่ได้โดยตรง (ต้องเขียน validation เอง) | ได้ (`{id:[0-9]+}`) |
| Route grouping / subrouter | ไม่มี (ต้องเขียนเอง) | มี (`r.Route`) แข็งแกร่งมาก | มี (`.Subrouter()`) |
| Middleware chaining ในตัว | ไม่มี (ต้องเขียนเองแบบ Part 057) | มี (`.Use()` + `chi/middleware`) | มี (`.Use()`) |
| Automatic 405 | มี | มี | ไม่มีในตัว (ต้องตั้งค่าเพิ่ม) |
| สถานะการพัฒนา (ปี 2026) | active (เป็นส่วนหนึ่งของ Go เอง) | active | maintenance mode |
| เหมาะกับ | API เล็ก-กลาง ที่ไม่ต้องการ regex/nested route ซับซ้อน | API ทุกขนาด โดยเฉพาะที่มี nested resource เยอะ | โปรเจกต์เดิมที่ใช้อยู่แล้ว |

---

## 11. เมื่อไหร่ที่ third-party router ยังคุ้มค่าอยู่

แม้ stdlib จะครอบคลุมความต้องการพื้นฐานได้แล้ว แต่ยังมีสถานการณ์ที่ third-party router (โดยเฉพาะ `chi`) ยังคุ้มค่าที่จะใช้:

1. **API มี nested resource ลึกหลายชั้น** — `r.Route` ของ chi จัดการเรื่องนี้ได้สะอาดกว่าการเขียน pattern เต็มซ้ำๆ ใน stdlib มาก
2. **ต้องการ middleware มาตรฐานพร้อมใช้จำนวนมาก** — `chi/middleware` มี middleware คุณภาพดีให้ใช้ทันที (Compress, Timeout, RequestID, RealIP) โดยไม่ต้องเขียนเองหรือหา library แยก
3. **ทีมคุ้นเคยกับ ecosystem รอบๆ chi อยู่แล้ว** — เช่น `render` (สำหรับ response), `chi/docgen` (สร้างเอกสาร route อัตโนมัติ) ที่ออกแบบมาให้ทำงานร่วมกับ chi ได้ลื่นไหล
4. **ต้องการ regex constraint บน path parameter โดยตรง** — กรณีนี้ `gorilla/mux` ยังเป็นตัวเลือกที่ตรงจุดที่สุด แม้จะอยู่ใน maintenance mode
5. **โปรเจกต์เดิมใช้ router ตัวใดตัวหนึ่งอยู่แล้ว** — ไม่มีเหตุผลต้อง migrate ออกจาก `gorilla/mux` หรือ `chi` ที่ทำงานได้ดีอยู่แล้ว เพียงเพราะ stdlib อัปเกรดขึ้นมา

ในทางกลับกัน ถ้าเริ่มโปรเจกต์ใหม่และความต้องการ routing ไม่ซับซ้อน (ไม่มี regex constraint, ไม่มี nested resource ลึกมาก) **stdlib `ServeMux` ของ Go 1.22+ เพียงพอแล้ว** และมีข้อดีคือไม่เพิ่ม dependency ให้โปรเจกต์เลย ตรงกับปรัชญา "standard library แข็งแกร่ง" ที่กล่าวถึงใน Part 001

---

## สรุปสิ่งที่ได้เรียนในบทนี้

- ก่อน Go 1.22 stdlib `ServeMux` ไม่รองรับ path parameter หรือ method matching ในตัว ทำให้ third-party router อย่าง `gorilla/mux` และ `chi` ได้รับความนิยมมาก
- `chi` เป็น router ที่เบา ไม่มี dependency ภายนอก และยัง active — เป็นตัวเลือกที่แนะนำที่สุดในปัจจุบันเมื่อจำเป็นต้องใช้ third-party router
- `r.Route()` ของ chi สร้าง subrouter ที่จัดกลุ่ม nested resource ได้อ่านง่ายกว่าการเขียน pattern เต็มซ้ำใน stdlib
- `chi.URLParam(r, "name")` เทียบเท่ากับ `r.PathValue("name")` ของ stdlib
- `chi/middleware` มี middleware มาตรฐาน (`Logger`, `Recoverer`, `RequestID`, `Timeout`, `Compress`) พร้อมใช้ทันที และเข้ากันได้กับ middleware ที่เขียนเองแบบ Part 057 เพราะ type ตรงกัน
- `gorilla/mux` มีจุดเด่นเรื่อง regex constraint บน parameter แต่เข้าสู่ maintenance mode แล้วตั้งแต่ปี 2022
- ตัดสินใจเลือก router จากความซับซ้อนของ routing ที่ต้องการจริง ไม่ใช่ตามความเคยชินหรือกระแส — stdlib เพียงพอสำหรับ API ส่วนใหญ่ในปัจจุบัน

## แบบฝึกหัดท้ายบท

1. ติดตั้ง `chi` ในโปรเจกต์ใหม่ แล้วแปลง Notes API จาก **Part 056** ให้ใช้ `chi` แทน stdlib `ServeMux` ทั้งหมด
2. เพิ่ม `middleware.Timeout(5 * time.Second)` เข้าไปใน router แล้วทดสอบด้วย handler ที่จงใจ `time.Sleep(10 * time.Second)` ว่า client ได้ response อะไรกลับมา
3. เขียน subrouter สำหรับ resource ใหม่ชื่อ `authors` ที่มี nested resource `/authors/{authorID}/books` แสดงหนังสือทั้งหมดของนักเขียนคนนั้น
4. ลองติดตั้ง `gorilla/mux` แล้วเขียน route ที่ใช้ regex constraint `{id:[0-9]+}` ทดสอบว่ายิง `/books/abc` (ที่ไม่ใช่ตัวเลข) ได้ผลลัพธ์อย่างไรเทียบกับ `/books/1`
5. อ่านเอกสาร `chi/middleware` เพิ่มเติมแล้วเลือกมาหนึ่งตัวที่ยังไม่ได้พูดถึงในบทนี้ (เช่น `middleware.AllowContentType` หรือ `middleware.Heartbeat`) ทดลองใช้งานจริงพร้อมอธิบายว่าใช้แก้ปัญหาอะไร
6. เขียนตารางเปรียบเทียบของตัวเอง (นอกเหนือจากหัวข้อ 10) โดยเพิ่มมิติ "ความง่ายในการเทส handler แยกจาก router" แล้วให้เหตุผลประกอบ

---

**ต่อไป**: [Part 059 — RESTful API Design หลักการ](./059-restful-api-design.md)
