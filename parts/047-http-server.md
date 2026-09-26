# Part 047: `net/http` — HTTP Server เจาะลึก

> ภาคที่ 4: Standard Library เชิงลึก — ตอนที่ 2 จาก 10 (Part 46–55)

## สารบัญของบทนี้

1. ภาพรวม: `http.HandleFunc`, `http.Handle`, และ `http.ServeMux`
2. หัวใจของทุกอย่าง: interface `http.Handler` และ method `ServeHTTP`
3. `http.ResponseWriter` และ `*http.Request`: อ่าน query param และ request body
4. Status Code และ Header: ทำไมลำดับการเรียกถึงสำคัญมาก
5. `http.ListenAndServe`: รัน server จริง
6. Graceful Shutdown: `http.Server.Shutdown` + `os/signal` + `context.WithTimeout`
7. Go 1.22+ Enhanced `ServeMux`: Method Matching และ Wildcard Routing
8. สรุปสิ่งที่ได้เรียนในบทนี้
9. แบบฝึกหัดท้ายบท

---

## 1. ภาพรวม: `http.HandleFunc`, `http.Handle`, และ `http.ServeMux`

ต่อจาก **Part 046** ที่เราเรียนฝั่ง client กันไปแล้ว บทนี้เป็นฝั่งตรงข้าม: เขียนโปรแกรม Go ให้เป็น**ผู้รับ** HTTP request แทนที่จะเป็นผู้ส่ง หัวใจของการสร้าง HTTP server ด้วย `net/http` วนอยู่รอบ 3 อย่างนี้:

- **`http.ServeMux`** — router (multiplexer) ที่จับคู่ **URL pattern** เข้ากับ **handler** ที่จะประมวลผล request นั้น
- **`http.HandleFunc`/`mux.HandleFunc`** — ลงทะเบียนฟังก์ชันธรรมดา (มี signature `func(http.ResponseWriter, *http.Request)`) ให้ทำหน้าที่เป็น handler ของ pattern หนึ่ง
- **`http.Handle`/`mux.Handle`** — ลงทะเบียนค่าใดๆ ก็ได้ที่ implement interface `http.Handler` (มี method `ServeHTTP`) ให้เป็น handler

Go มี **`http.DefaultServeMux`** เป็น router ตัว default ให้ใช้แบบ global (เวลาเรียก `http.HandleFunc(...)` ตรงๆ โดยไม่ผ่าน mux ของตัวเอง คือการลงทะเบียนเข้า `DefaultServeMux` นี้) แต่ในโค้ด production **ควรสร้าง `http.NewServeMux()` ของตัวเองเสมอ** ด้วยเหตุผลเดียวกับที่ **Part 046** แนะนำให้เลี่ยง `http.DefaultClient`: `DefaultServeMux` เป็น **global mutable state** ที่ package ใดๆ ในโปรแกรม (รวมถึง third-party library ที่ import เข้ามา) สามารถแอบลงทะเบียน route ทับกันได้โดยไม่รู้ตัว ทำให้ debug ยากและทดสอบยาก การสร้าง mux ของตัวเองทำให้ควบคุมได้ชัดเจนว่า route ทั้งหมดของแอปมีอะไรบ้าง

---

## 2. หัวใจของทุกอย่าง: interface `http.Handler` และ method `ServeHTTP`

เบื้องหลัง `http.HandleFunc` และ `http.Handle` ทั้งหมดคือ interface ตัวเดียวที่เรียบง่ายมาก (ตามปรัชญา "less is more" จาก **Part 001** และรูปแบบเดียวกับ `io.Reader`/`io.Writer` ที่มี method เดียวจาก **Part 024**):

```go
type Handler interface {
	ServeHTTP(ResponseWriter, *Request)
}
```

**ทุกอย่างในระบบ HTTP server ของ Go ล้วนเป็น `http.Handler`** ไม่ว่าจะเป็น `http.ServeMux` เอง (mux ก็ implement `ServeHTTP` เพื่อทำหน้าที่หา handler ที่ตรงกับ pattern แล้วเรียกต่อ), middleware (จะเรียนเจาะลึกใน **Part 057**), หรือ handler ที่เราเขียนเอง — เพราะ interface นี้มี method เดียว การสร้าง type ใดๆ ที่มี method `ServeHTTP` ที่ signature ตรงกันก็ทำให้ type นั้น "เป็น" `http.Handler` ได้ทันที (implicit interface satisfaction ทบทวนจาก **Part 013**)

ส่วน **`http.HandlerFunc`** คือ "สะพาน" ที่ทำให้ฟังก์ชันธรรมดากลายเป็น `http.Handler` ได้โดยไม่ต้องเขียน struct เพิ่ม:

```go
type HandlerFunc func(ResponseWriter, *Request)

func (f HandlerFunc) ServeHTTP(w ResponseWriter, r *Request) {
	f(w, r)
}
```

นี่คือเทคนิคที่ชาญฉลาดมาก: `HandlerFunc` เป็น **function type** ที่มี method `ServeHTTP` ของตัวเอง (method บน function type ทำได้ใน Go) ทำให้ฟังก์ชันธรรมดาที่แปลงเป็น `HandlerFunc` แล้ว กลายเป็น `http.Handler` ที่ใช้งานได้ทันที — `http.HandleFunc(pattern, fn)` ภายในก็แค่ทำ `mux.Handle(pattern, http.HandlerFunc(fn))` ให้เราโดยอัตโนมัติเท่านั้นเอง

มาดูทั้งสองวิธีเทียบกันในโปรแกรมเดียว:

```go
package main

import (
	"fmt"
	"io"
	"net/http"
	"net/http/httptest"
)

func helloHandler(w http.ResponseWriter, r *http.Request) {
	fmt.Fprintln(w, "สวัสดีจาก /hello")
}

func aboutHandler(w http.ResponseWriter, r *http.Request) {
	fmt.Fprintln(w, "นี่คือหน้า /about")
}

// customHandler implement http.Handler interface เอง (ไม่ใช้ HandleFunc)
type customHandler struct {
	greeting string
}

// ServeHTTP คือ method เดียวที่ http.Handler ต้องการ - เพียงเท่านี้ type นี้ก็เป็น http.Handler แล้ว
func (h customHandler) ServeHTTP(w http.ResponseWriter, r *http.Request) {
	fmt.Fprintf(w, "%s (จาก custom Handler type)\n", h.greeting)
}

func main() {
	// http.NewServeMux สร้าง router (multiplexer) ของตัวเอง แทนที่จะใช้ http.DefaultServeMux ตรงๆ
	// (การใช้ mux ของตัวเองดีกว่าในโค้ดจริง เพราะ DefaultServeMux เป็น global state ที่ package อื่น
	// อาจแอบลงทะเบียน route ทับกันโดยไม่ตั้งใจ)
	mux := http.NewServeMux()

	// http.HandleFunc/mux.HandleFunc รับฟังก์ชันธรรมดาที่มี signature ตรงกับ http.HandlerFunc
	mux.HandleFunc("/hello", helloHandler)
	mux.HandleFunc("/about", aboutHandler)

	// http.Handle/mux.Handle รับค่าที่ implement interface http.Handler (มี method ServeHTTP)
	mux.Handle("/custom", customHandler{greeting: "ยินดีต้อนรับ"})

	// ทดสอบด้วย httptest.NewServer ที่รับ http.Handler ใดๆ ก็ได้ (mux เองก็ implement http.Handler)
	server := httptest.NewServer(mux)
	defer server.Close()

	for _, path := range []string{"/hello", "/about", "/custom"} {
		resp, err := http.Get(server.URL + path)
		if err != nil {
			fmt.Println("error:", err)
			continue
		}
		body, _ := io.ReadAll(resp.Body)
		resp.Body.Close()
		fmt.Printf("GET %s -> %s", path, string(body))
	}
}
```

ผลลัพธ์:

```
GET /hello -> สวัสดีจาก /hello
GET /about -> นี่คือหน้า /about
GET /custom -> ยินดีต้อนรับ (จาก custom Handler type)
```

**เมื่อไรควรใช้ `Handle` (custom type) แทน `HandleFunc` (ฟังก์ชันธรรมดา)**: ถ้า handler ต้องการ**เก็บ state หรือ dependency ของตัวเอง** (เช่น connection ไปยัง database, logger, config) การทำเป็น struct ที่ implement `ServeHTTP` แล้วเก็บ dependency เป็น field จะสะอาดกว่าการพึ่ง closure หรือตัวแปร global — pattern นี้จะกลับมาใช้บ่อยมากเมื่อสร้าง REST API จริงจังใน **ภาคที่ 5 (Web Development)**

`httptest.NewServer(handler http.Handler)` ในตัวอย่างข้างต้นรับ `http.Handler` เป็นพารามิเตอร์ — และเพราะ `mux` (ที่เป็น `*http.ServeMux`) implement interface นี้อยู่แล้ว จึงส่งเข้าไปได้ตรงๆ นี่คือพลังของ interface อีกครั้ง: โค้ดที่เขียนรับ `http.Handler` ใช้ได้กับทั้ง mux, custom handler, หรือ middleware ที่ห่อ handler อื่นซ้อนกันหลายชั้น โดยไม่ต้องรู้รายละเอียดภายในเลย

---

## 3. `http.ResponseWriter` และ `*http.Request`: อ่าน query param และ request body

ทุก handler รับพารามิเตอร์สองตัวเสมอ:

- **`w http.ResponseWriter`** — interface สำหรับ**เขียน**ข้อมูลตอบกลับไปยัง client (set header, set status, เขียน body)
- **`r *http.Request`** — struct ที่เก็บข้อมูลทั้งหมดของ request ที่เข้ามา (URL, header, body, method ฯลฯ)

```go
package main

import (
	"encoding/json"
	"fmt"
	"io"
	"net/http"
	"net/http/httptest"
	"strings"
)

type greetRequest struct {
	Name string `json:"name"`
}

func main() {
	mux := http.NewServeMux()

	// อ่าน query parameter ผ่าน r.URL.Query() ซึ่งคืนค่าเป็น url.Values (คล้าย map[string][]string)
	mux.HandleFunc("/greet", func(w http.ResponseWriter, r *http.Request) {
		name := r.URL.Query().Get("name")
		if name == "" {
			name = "ผู้มาเยือน"
		}
		fmt.Fprintf(w, "สวัสดี %s!\n", name)
	})

	// อ่าน request body แบบ JSON แล้วตอบกลับเป็น JSON เช่นกัน
	mux.HandleFunc("/api/greet", func(w http.ResponseWriter, r *http.Request) {
		var req greetRequest
		if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
			http.Error(w, "invalid json body", http.StatusBadRequest)
			return
		}
		w.Header().Set("Content-Type", "application/json")
		json.NewEncoder(w).Encode(map[string]string{
			"message": fmt.Sprintf("สวัสดี %s จาก API", req.Name),
		})
	})

	// อ่าน request body แบบ raw text ด้วย io.ReadAll ตรงๆ
	mux.HandleFunc("/echo", func(w http.ResponseWriter, r *http.Request) {
		body, err := io.ReadAll(r.Body)
		if err != nil {
			http.Error(w, "read body failed", http.StatusInternalServerError)
			return
		}
		fmt.Fprintf(w, "server ได้รับ %d bytes: %s\n", len(body), body)
	})

	server := httptest.NewServer(mux)
	defer server.Close()

	client := server.Client()

	resp1, _ := client.Get(server.URL + "/greet?name=Gopher")
	body1, _ := io.ReadAll(resp1.Body)
	resp1.Body.Close()
	fmt.Print(string(body1))

	resp2, _ := client.Get(server.URL + "/greet")
	body2, _ := io.ReadAll(resp2.Body)
	resp2.Body.Close()
	fmt.Print(string(body2))

	resp3, _ := client.Post(server.URL+"/api/greet", "application/json", strings.NewReader(`{"name":"สมหญิง"}`))
	body3, _ := io.ReadAll(resp3.Body)
	resp3.Body.Close()
	fmt.Println(string(body3))

	resp4, _ := client.Post(server.URL+"/echo", "text/plain", strings.NewReader("ทดสอบ echo"))
	body4, _ := io.ReadAll(resp4.Body)
	resp4.Body.Close()
	fmt.Print(string(body4))
}
```

ผลลัพธ์:

```
สวัสดี Gopher!
สวัสดี ผู้มาเยือน!
{"message":"สวัสดี สมหญิง จาก API"}

server ได้รับ 20 bytes: ทดสอบ echo
```

### ประเด็นสำคัญที่ต้องจำ

- `r.URL.Query()` คืนค่าเป็น `url.Values` (คือ `map[string][]string`) — ใช้ `.Get(key)` เพื่ออ่านค่าแรกสะดวกๆ (คืน string ว่างถ้าไม่มี key นั้น ไม่ panic)
- `r.Body` เป็น `io.ReadCloser` เหมือนกับ `resp.Body` ฝั่ง client ใน **Part 046** — อ่านได้ครั้งเดียว (เป็น stream) และฝั่ง server **ไม่จำเป็นต้องปิด `r.Body` เอง** เพราะ `net/http` จัดการปิดให้อัตโนมัติหลัง handler ทำงานเสร็จ (ต่างจากฝั่ง client ที่ต้อง `defer resp.Body.Close()` เองเสมอ)
- `json.NewDecoder(r.Body).Decode(&req)` เป็น pattern มาตรฐานสำหรับอ่าน JSON request body — stream ตรงจาก connection เข้า struct โดยไม่ต้องพักข้อมูลทั้งก้อนไว้ก่อน เหมือนที่เรียนใน **Part 025** และ **Part 046**
- `http.Error(w, message, statusCode)` เป็นฟังก์ชันสะดวกสำหรับตอบ error กลับไปพร้อมกัน — ตั้ง `Content-Type: text/plain`, เขียน status code, และเขียนข้อความ error ลง body ให้ในคำสั่งเดียว (ทำสิ่งเดียวกับที่หัวข้อถัดไปจะอธิบายเรื่องลำดับ `WriteHeader`/`Write` ให้ถูกต้องอัตโนมัติ)

---

## 4. Status Code และ Header: ทำไมลำดับการเรียกถึงสำคัญมาก

`http.ResponseWriter` มี 3 method หลัก:

```go
type ResponseWriter interface {
	Header() Header               // ดึง map ของ response header ออกมาแก้ไข
	Write([]byte) (int, error)    // เขียนข้อมูลลง response body
	WriteHeader(statusCode int)   // ส่ง status code (และ header ทั้งหมดที่ตั้งไว้ ณ ตอนนั้น) ออกไป
}
```

จุดที่สร้างความสับสนให้มือใหม่มากที่สุดคือ**ลำดับการเรียก 3 method นี้มีผลจริง** เพราะ HTTP response ถูกส่งออกไปเป็น stream ทีละส่วน: **status line และ header ต้องถูกส่งออกไปก่อน body เสมอ** (เป็นข้อกำหนดของ HTTP protocol เอง ไม่ใช่ข้อจำกัดของ Go) เมื่อ Go เจอการเรียก `Write()` เป็นครั้งแรกโดยที่ยังไม่มีใครเรียก `WriteHeader()` มาก่อน มันจะ**ส่ง header ออกไปด้วย status `200 OK` ให้อัตโนมัติทันที** ก่อนเขียน body — ทำให้การเรียก `WriteHeader()` หลังจากนั้น **ไม่มีผลอะไรเลย** เพราะ header ถูกส่งออกไปหมดแล้ว

มาดูบั๊กนี้แบบจับต้องได้:

```go
package main

import (
	"fmt"
	"net/http"
	"net/http/httptest"
)

func main() {
	mux := http.NewServeMux()

	// ผิด: เขียน body ก่อน แล้วค่อยพยายามตั้ง status code ทีหลัง
	mux.HandleFunc("/wrong", func(w http.ResponseWriter, r *http.Request) {
		fmt.Fprintln(w, "พยายามส่ง 404 แต่จริงๆ แล้วมันสายไปแล้ว")
		// Write() ด้านบนทำให้ Go ส่ง header "200 OK" ออกไปให้ client โดยอัตโนมัติทันที
		// (เพราะยังไม่มีใครเรียก WriteHeader มาก่อน) การเรียก WriteHeader ตอนนี้จึง "ไม่มีผลใดๆ"
		w.WriteHeader(http.StatusNotFound)
	})

	// ถูก: ตั้ง header และ status code ให้ครบก่อน แล้วค่อยเขียน body ทีหลังเสมอ
	mux.HandleFunc("/correct", func(w http.ResponseWriter, r *http.Request) {
		w.Header().Set("X-Custom-Header", "must-be-set-before-writeheader")
		w.WriteHeader(http.StatusNotFound)
		fmt.Fprintln(w, "ไม่พบข้อมูลที่ต้องการ (404 ถูกต้อง)")
	})

	server := httptest.NewServer(mux)
	defer server.Close()

	client := server.Client()

	resp1, _ := client.Get(server.URL + "/wrong")
	fmt.Println("/wrong -> status:", resp1.Status, "(คาดหวัง 404 แต่ได้ 200 เพราะเรียก WriteHeader สายไป)")
	resp1.Body.Close()

	resp2, _ := client.Get(server.URL + "/correct")
	fmt.Println("/correct -> status:", resp2.Status, "header:", resp2.Header.Get("X-Custom-Header"))
	resp2.Body.Close()
}
```

ผลลัพธ์:

```
2026/09/26 02:57:09 http: superfluous response.WriteHeader call from main.main.func1 (main.go:17)
/wrong -> status: 200 OK (คาดหวัง 404 แต่ได้ 200 เพราะเรียก WriteHeader สายไป)
/correct -> status: 404 Not Found header: must-be-set-before-writeheader
```

สังเกตบรรทัดแรก — Go **เตือนให้เองทาง log** ด้วยข้อความ `superfluous response.WriteHeader call` (แปลตรงตัวว่า "เรียก WriteHeader ที่ฟุ่มเฟือย/ไม่มีผล") พร้อมระบุไฟล์และบรรทัดที่ทำผิดพลาดให้ชัดเจน นี่เป็นสัญญาณที่มีประโยชน์มากเวลา debug ปัญหาว่า "ทำไม status code ที่ตั้งไว้ไม่มีผล"

### กฎการเรียงลำดับที่ต้องจำ

1. **`w.Header().Set(...)` ต้องมาก่อนเสมอ** — แก้ header ได้เฉพาะ**ก่อน**ที่จะเรียก `WriteHeader` หรือ `Write` เท่านั้น หลังจากนั้นแก้ไม่ได้อีกแล้ว (header ถูกส่งออกไปแล้วจริงๆ ทาง network)
2. **`w.WriteHeader(code)` เรียกได้เพียงครั้งเดียว** — ถ้าไม่เรียกเลย Go จะเรียกให้อัตโนมัติด้วย `200 OK` ตอนที่ `Write()` ถูกเรียกครั้งแรก
3. **`w.Write(...)` (หรือ `fmt.Fprintf(w, ...)`, `json.NewEncoder(w).Encode(...)` ที่เขียนผ่าน `w` ในฐานะ `io.Writer`) มาหลังสุดเสมอ**

ลำดับที่ถูกต้องคือ: **Header → WriteHeader → Write** เท่านั้น สลับหรือข้ามขั้นตอนไม่ได้เลย

---

## 5. `http.ListenAndServe`: รัน server จริง

ตัวอย่างก่อนหน้าทั้งหมดใช้ `httptest.NewServer` เพื่อทดสอบในโปรแกรมเดียว แต่ในโลกจริง server ต้อง**เปิด port รอรับ connection ตลอดเวลา** ด้วย `http.ListenAndServe`:

```go
mux := http.NewServeMux()
mux.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
	fmt.Fprintln(w, "Hello, World!")
})

log.Fatal(http.ListenAndServe(":8080", mux))
```

`http.ListenAndServe(addr, handler)` **บล็อกการทำงานตลอดไป** (จนกว่า server จะปิดหรือเกิด error) จึงมักเขียนคลุมด้วย `log.Fatal(...)` เพื่อพิมพ์ error แล้วจบโปรแกรมทันทีถ้า listen ไม่สำเร็จ (เช่น port ถูกใช้งานอยู่แล้ว) — ฟังก์ชันนี้เป็น shortcut ของการสร้าง `&http.Server{Addr: addr, Handler: handler}` แล้วเรียก `srv.ListenAndServe()` เองอีกที

รูปแบบที่ควบคุมได้มากกว่าคือประกาศ `http.Server` เอง:

```go
srv := &http.Server{
	Addr:         ":8080",
	Handler:      mux,
	ReadTimeout:  5 * time.Second,  // เวลาสูงสุดที่ยอมให้อ่าน request ทั้งหมด (header + body)
	WriteTimeout: 10 * time.Second, // เวลาสูงสุดที่ยอมให้เขียน response ทั้งหมด
	IdleTimeout:  120 * time.Second, // เวลาสูงสุดที่ยอมให้ connection แบบ keep-alive ว่างเปล่า
}
log.Fatal(srv.ListenAndServe())
```

**ทำไมต้องตั้ง `ReadTimeout`/`WriteTimeout` เอง**: เช่นเดียวกับ `http.Client.Timeout` ที่เรียนใน **Part 046** — `http.Server` ที่ไม่ตั้งค่า timeout เหล่านี้ (ค่า default เป็น `0` คือไม่จำกัด) **เสี่ยงต่อการถูกโจมตีแบบ "Slowloris"** ที่ client ส่ง request มาแบบช้าๆ ทีละไม่กี่ byte เพื่อยึด connection ค้างไว้จำนวนมาก จนทำให้ server ไม่มี resource เหลือรับ request จาก client ปกติได้เลย — **การตั้ง timeout ทั้งฝั่ง client และฝั่ง server จึงเป็นกฎทองที่ต้องทำเสมอทั้งคู่**

---

## 6. Graceful Shutdown: `http.Server.Shutdown` + `os/signal` + `context.WithTimeout`

การปิด server แบบ**ทันที** (เช่น กด Ctrl+C แล้วโปรแกรมตายทันที) มีปัญหา: request ที่**กำลังประมวลผลอยู่ ณ ขณะนั้น** จะถูกตัดจบกลางคัน client จะได้รับ connection reset แทนที่จะได้ response ที่ถูกต้อง — ใน production (โดยเฉพาะเวลา deploy เวอร์ชันใหม่ หรือ container orchestrator อย่าง Kubernetes สั่งปิด pod) เราต้องการ **"ปิดอย่างนุ่มนวล" (graceful shutdown)**: หยุดรับ connection ใหม่ทันที แต่**รอให้ request ที่กำลังทำงานอยู่เสร็จก่อน**แล้วค่อยปิดจริง

`http.Server` มี method `Shutdown(ctx context.Context)` ที่ทำสิ่งนี้ให้โดยตรง ร่วมกับการดักจับสัญญาณจากระบบปฏิบัติการผ่าน `os/signal` (ทบทวน `context.WithTimeout` จาก **Part 032**):

```go
package main

import (
	"context"
	"errors"
	"fmt"
	"log"
	"net"
	"net/http"
	"os"
	"os/signal"
	"syscall"
	"time"
)

func main() {
	mux := http.NewServeMux()
	mux.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
		fmt.Fprintln(w, "สวัสดีจาก graceful server")
	})
	// จำลอง handler ที่ทำงานนาน เพื่อพิสูจน์ว่า Shutdown รอ request ที่กำลังทำงานอยู่ให้เสร็จก่อน
	mux.HandleFunc("/slow", func(w http.ResponseWriter, r *http.Request) {
		time.Sleep(1 * time.Second)
		fmt.Fprintln(w, "งานที่ใช้เวลานานเสร็จแล้ว")
	})

	srv := &http.Server{
		Handler: mux,
	}

	// ใช้ net.Listen เอง (แทน http.ListenAndServe ตรงๆ) เพื่อผูกกับ port แบบสุ่ม (":0")
	// สำหรับการสาธิตในบทความนี้ ในโปรแกรมจริงมักระบุ port ตายตัว เช่น ":8080"
	listener, err := net.Listen("tcp", "127.0.0.1:0")
	if err != nil {
		log.Fatal("เปิด listener ไม่สำเร็จ:", err)
	}
	addr := listener.Addr().String()
	fmt.Println("server เริ่มทำงานที่:", addr)

	// รัน server ใน goroutine แยก เพราะ Serve() จะบล็อกจนกว่า server จะถูกปิด
	serverErr := make(chan error, 1)
	go func() {
		serverErr <- srv.Serve(listener)
	}()

	// ดักจับสัญญาณ SIGINT (Ctrl+C) และ SIGTERM (สัญญาณมาตรฐานที่ container orchestrator เช่น
	// Kubernetes ส่งมาตอนจะปิด pod) ผ่าน os/signal - นี่คือ pattern มาตรฐานสำหรับ graceful shutdown
	quit := make(chan os.Signal, 1)
	signal.Notify(quit, syscall.SIGINT, syscall.SIGTERM)

	// จำลองไคลเอนต์ยิง request เข้ามา แล้วส่งสัญญาณ SIGINT ให้ตัวเองเพื่อทดสอบ graceful shutdown
	// (ในโปรแกรมจริง สัญญาณนี้มาจาก OS ตอนผู้ใช้กด Ctrl+C หรือ container ถูกสั่งหยุด)
	go func() {
		time.Sleep(200 * time.Millisecond)
		resp, err := http.Get("http://" + addr + "/")
		if err == nil {
			resp.Body.Close()
			fmt.Println("client เรียก / สำเร็จก่อน shutdown")
		}

		// ยิง request ที่ทำงานนานเข้ามาแบบไม่รอผล เพื่อพิสูจน์ว่า Shutdown จะรอ request นี้ให้เสร็จก่อน
		go func() {
			resp, err := http.Get("http://" + addr + "/slow")
			if err == nil {
				resp.Body.Close()
				fmt.Println("client เรียก /slow สำเร็จ (แม้ shutdown จะเริ่มไปแล้วระหว่างที่รออยู่)")
			}
		}()

		time.Sleep(100 * time.Millisecond)
		fmt.Println("ส่งสัญญาณ SIGINT ให้ตัวเอง (จำลอง Ctrl+C)...")
		syscall.Kill(os.Getpid(), syscall.SIGINT)
	}()

	// บล็อกรอสัญญาณ shutdown
	<-quit
	fmt.Println("ได้รับสัญญาณ shutdown กำลังปิด server อย่างนุ่มนวล...")

	// context.WithTimeout กำหนดเวลาสูงสุดที่ยอมให้ Shutdown รอ request ที่ค้างอยู่ให้เสร็จ
	// ถ้าเกินเวลานี้ Shutdown จะ "ตัดจบ" request ที่เหลือทันทีแล้วคืน error กลับมา
	ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
	defer cancel()

	if err := srv.Shutdown(ctx); err != nil {
		fmt.Println("shutdown ไม่สมบูรณ์:", err)
	} else {
		fmt.Println("server ปิดตัวเรียบร้อยแล้ว (request ที่ค้างอยู่ทำงานจนเสร็จหมดแล้ว)")
	}

	// รอให้ goroutine ของ Serve() คืนค่ากลับมา (จะได้ http.ErrServerClosed เสมอเมื่อปิดผ่าน Shutdown)
	if err := <-serverErr; err != nil && !errors.Is(err, http.ErrServerClosed) {
		fmt.Println("server หยุดทำงานด้วย error ที่ไม่คาดคิด:", err)
	}
}
```

ผลลัพธ์:

```
server เริ่มทำงานที่: 127.0.0.1:36869
client เรียก / สำเร็จก่อน shutdown
ส่งสัญญาณ SIGINT ให้ตัวเอง (จำลอง Ctrl+C)...
ได้รับสัญญาณ shutdown กำลังปิด server อย่างนุ่มนวล...
client เรียก /slow สำเร็จ (แม้ shutdown จะเริ่มไปแล้วระหว่างที่รออยู่)
server ปิดตัวเรียบร้อยแล้ว (request ที่ค้างอยู่ทำงานจนเสร็จหมดแล้ว)
```

### เจาะลึกกลไกการทำงาน

- **`signal.Notify(quit, syscall.SIGINT, syscall.SIGTERM)`** — บอก Go ให้ส่งสัญญาณ OS ที่ระบุเข้า channel `quit` แทนที่จะให้ signal จบโปรแกรมทันทีตามพฤติกรรม default (ค่า default ของ `SIGINT`/`SIGTERM` คือฆ่าโปรแกรมทันที) เมื่อ "ดัก" สัญญาณไว้แบบนี้ เราจึงมีโอกาสทำ cleanup ก่อนที่โปรแกรมจะจบจริง
- **`<-quit`** บล็อกรอจนกว่าจะมีสัญญาณเข้ามา — นี่คือจุดที่โปรแกรมหลักรอเฉยๆ ระหว่างที่ server ทำงานให้บริการ request ตามปกติ
- **`srv.Shutdown(ctx)`** เมื่อถูกเรียก จะ**หยุดรับ connection ใหม่ทันที** (connection ใหม่ที่พยายามเข้ามาจะถูกปฏิเสธ) แต่**รอให้ handler ที่กำลังทำงานอยู่ ณ ขณะนั้นทำงานจนเสร็จก่อน** จะเห็นได้จาก log ว่าข้อความ `"client เรียก /slow สำเร็จ"` ปรากฏ**หลังจาก** `"ได้รับสัญญาณ shutdown"` แต่**ก่อน** `"server ปิดตัวเรียบร้อยแล้ว"` — พิสูจน์ชัดเจนว่า `Shutdown` รอ request ที่ sleep ค้างอยู่ 1 วินาทีให้เสร็จสมบูรณ์ก่อนจริงๆ ไม่ได้ตัดจบกลางคัน
- **`context.WithTimeout(...)` ที่ส่งให้ `Shutdown`** คือ "เพดานความอดทน" — ถ้า request ที่ค้างอยู่ใช้เวลานานเกินไป (เกิน 5 วินาทีในตัวอย่างนี้) `Shutdown` จะเลิกรอแล้วตัดจบ connection ที่เหลือทันที คืน error ที่ไม่ใช่ `nil` กลับมาแทน — ป้องกันไม่ให้กระบวนการปิดโปรแกรมค้างอยู่ตลอดไปเพราะ request ตัวใดตัวหนึ่งค้าง
- **`errors.Is(err, http.ErrServerClosed)`** (ทบทวนจาก **Part 016**) — `Serve()`/`ListenAndServe()` จะคืน error เสมอเมื่อ server หยุดทำงาน แม้จะปิดแบบตั้งใจผ่าน `Shutdown()` ก็ตาม โดยจะได้ `http.ErrServerClosed` เป็นค่าประจำ ซึ่ง**ไม่ใช่ error จริง** แต่เป็นสัญญาณยืนยันว่า "ปิดตามที่สั่งแล้ว" จึงต้องเช็คแยกจาก error ประเภทอื่นเสมอ ไม่งั้นจะรายงาน error ผิดๆ ทุกครั้งที่ shutdown ตามปกติ

รูปแบบนี้เป็น **มาตรฐานอุตสาหกรรม** สำหรับ Go HTTP server ทุกตัวที่รันบน container/Kubernetes — orchestrator จะส่ง `SIGTERM` มาก่อนเสมอเมื่อจะปิด pod (ให้เวลา "grace period" หนึ่งก่อนจะบังคับ `SIGKILL`) โค้ดแบบนี้ทำให้แอปมีโอกาสปิดตัวอย่างสุภาพโดยไม่ทำ request ของผู้ใช้ที่กำลังทำงานอยู่ให้เสียหาย

---

## 7. Go 1.22+ Enhanced `ServeMux`: Method Matching และ Wildcard Routing

ก่อน Go 1.22 `http.ServeMux` รองรับแค่การจับคู่ path แบบง่ายๆ (exact match หรือ prefix match ด้วย `/`) ทำให้การทำ REST API ที่ต้องแยกตาม HTTP method (`GET`/`POST`/`DELETE`) หรืออ่านค่าจาก path เช่น `/items/42` ต้องเขียนโค้ด parse เองหรือพึ่ง router จาก third-party อย่าง `gorilla/mux`/`chi` (ที่จะเรียนใน **Part 058**)

ตั้งแต่ **Go 1.22** เป็นต้นไป `http.ServeMux` รองรับ **pattern syntax ที่ทรงพลังขึ้นมาก** ในตัวเอง โดยไม่ต้องพึ่ง library ภายนอกอีกต่อไปสำหรับ routing พื้นฐาน:

```go
package main

import (
	"encoding/json"
	"fmt"
	"io"
	"net/http"
	"net/http/httptest"
)

type Item struct {
	ID   string `json:"id"`
	Name string `json:"name"`
}

var items = map[string]Item{
	"1": {ID: "1", Name: "แล็ปท็อป"},
	"2": {ID: "2", Name: "เมาส์"},
}

func main() {
	mux := http.NewServeMux()

	// Go 1.22+ : ระบุ HTTP method นำหน้า pattern ได้โดยตรง (คั่นด้วยช่องว่าง)
	// mux จะ dispatch ตาม method ให้อัตโนมัติ ไม่ต้องเช็ค r.Method เองในตัว handler อีกต่อไป
	mux.HandleFunc("GET /items/{id}", func(w http.ResponseWriter, r *http.Request) {
		// r.PathValue("id") ดึงค่าจาก wildcard segment {id} ใน pattern ออกมาโดยตรง
		id := r.PathValue("id")
		item, ok := items[id]
		if !ok {
			http.Error(w, "ไม่พบสินค้า", http.StatusNotFound)
			return
		}
		w.Header().Set("Content-Type", "application/json")
		json.NewEncoder(w).Encode(item)
	})

	mux.HandleFunc("POST /items/{id}", func(w http.ResponseWriter, r *http.Request) {
		id := r.PathValue("id")
		var item Item
		if err := json.NewDecoder(r.Body).Decode(&item); err != nil {
			http.Error(w, "invalid json", http.StatusBadRequest)
			return
		}
		item.ID = id
		items[id] = item
		w.WriteHeader(http.StatusCreated)
		json.NewEncoder(w).Encode(item)
	})

	mux.HandleFunc("DELETE /items/{id}", func(w http.ResponseWriter, r *http.Request) {
		id := r.PathValue("id")
		delete(items, id)
		w.WriteHeader(http.StatusNoContent)
	})

	// wildcard ท้าย pattern ด้วย {name...} จับ path ที่เหลือทั้งหมด (รวม "/" ข้างในด้วย)
	mux.HandleFunc("GET /files/{path...}", func(w http.ResponseWriter, r *http.Request) {
		fmt.Fprintf(w, "ขอไฟล์ที่ path: %s\n", r.PathValue("path"))
	})

	server := httptest.NewServer(mux)
	defer server.Close()
	client := server.Client()

	// GET /items/1 -> เจอสินค้า
	resp1, _ := client.Get(server.URL + "/items/1")
	body1, _ := io.ReadAll(resp1.Body)
	resp1.Body.Close()
	fmt.Println("GET /items/1 ->", resp1.StatusCode, string(body1))

	// GET /items/999 -> ไม่เจอ
	resp2, _ := client.Get(server.URL + "/items/999")
	resp2.Body.Close()
	fmt.Println("GET /items/999 ->", resp2.StatusCode)

	// DELETE /items/2 -> ลบสำเร็จ ต่างจาก method GET ที่ใช้ path เดียวกัน เพราะ mux แยกด้วย method
	req, _ := http.NewRequest(http.MethodDelete, server.URL+"/items/2", nil)
	resp3, _ := client.Do(req)
	fmt.Println("DELETE /items/2 ->", resp3.StatusCode)
	resp3.Body.Close()

	// PUT /items/1 -> ไม่มี handler ของ method นี้ลงทะเบียนไว้เลย แต่ path ตรงกับ pattern อื่น
	reqPut, _ := http.NewRequest(http.MethodPut, server.URL+"/items/1", nil)
	respPut, _ := client.Do(reqPut)
	fmt.Println("PUT /items/1 (ไม่มี handler ของ method นี้) ->", respPut.StatusCode, respPut.Header.Get("Allow"))
	respPut.Body.Close()

	// GET /files/... -> wildcard {path...} จับ path ที่เหลือทั้งหมดรวม "/" ข้างในด้วย
	resp4, _ := client.Get(server.URL + "/files/docs/2024/report.pdf")
	body4, _ := io.ReadAll(resp4.Body)
	resp4.Body.Close()
	fmt.Print(string(body4))
}
```

ผลลัพธ์:

```
GET /items/1 -> 200 {"id":"1","name":"แล็ปท็อป"}

GET /items/999 -> 404
DELETE /items/2 -> 204
PUT /items/1 (ไม่มี handler ของ method นี้) -> 405 DELETE, GET, HEAD, POST
ขอไฟล์ที่ path: docs/2024/report.pdf
```

### เจาะลึก pattern syntax ใหม่

1. **`"METHOD /path"`** — ระบุ HTTP method นำหน้า pattern ได้โดยตรง คั่นด้วยช่องว่างหนึ่งตัว ถ้าไม่ระบุ method (เช่น `"/greet"` แบบหัวข้อก่อนหน้า) จะ match ได้ทุก method เหมือนพฤติกรรมเดิม
2. **`{name}`** — wildcard segment เดียว จับได้แค่ 1 ส่วนของ path (ไม่ข้าม `/`) อ่านค่าออกมาด้วย `r.PathValue("name")`
3. **`{name...}`** — wildcard แบบ "จับส่วนที่เหลือทั้งหมด" (ต้องอยู่ท้าย pattern เท่านั้น) จับได้แม้มี `/` ซ้อนอยู่ข้างในหลายชั้น เหมาะกับงานเช่น serve ไฟล์ตาม path ที่ซ้อนกันลึกๆ
4. **405 Method Not Allowed อัตโนมัติ** — สังเกตผลลัพธ์ของ `PUT /items/1`: แม้เราไม่ได้เขียนโค้ดเช็คเองเลยว่า method ไหนรองรับบ้าง `ServeMux` ก็**ฉลาดพอที่จะรู้เองว่า path `/items/{id}` มี pattern ที่ลงทะเบียนไว้สำหรับ `GET`, `POST`, `DELETE`** (และ `HEAD` ที่ Go เพิ่มให้อัตโนมัติคู่กับทุก `GET`) จึงตอบ **`405 Method Not Allowed`** กลับมาพร้อม header **`Allow`** ที่บอกรายชื่อ method ที่ใช้ได้จริงให้ client รู้ — พฤติกรรมนี้เป็นไปตามมาตรฐาน HTTP ที่ถูกต้อง โดยที่เราไม่ต้องเขียน logic นี้เองเลยสักบรรทัด

### ความสำคัญของ "ความจำเพาะเจาะจง" (Specificity) ในการเลือก pattern

เมื่อมีหลาย pattern ที่อาจ match กับ path เดียวกันได้ (เช่น `/items/{id}` กับ `/items/all`) `ServeMux` จะเลือก pattern ที่**เจาะจงกว่าเสมอ** โดยไม่สนใจลำดับการลงทะเบียนก่อนหลัง — กฎนี้ทำให้เขียน route ที่ทับซ้อนกันได้อย่างปลอดภัยโดยไม่ต้องกังวลเรื่องลำดับ `mux.HandleFunc(...)` ก่อนหลังเหมือน router บางตัวในภาษาอื่น

**เมื่อไรควรใช้ `ServeMux` ของ standard library เอง เมื่อไรควรใช้ third-party router**: สำหรับ REST API ขนาดเล็ก-กลางที่ routing ไม่ซับซ้อนมาก `ServeMux` เวอร์ชันปัจจุบันเพียงพอและไม่ต้องเพิ่ม dependency ใดๆ เลย แต่ถ้าต้องการ feature ขั้นสูงกว่านี้ เช่น route grouping ที่สะดวก, middleware chaining ในตัว, regex pattern ที่ซับซ้อน — `gorilla/mux` หรือ `chi` (ที่จะเรียนใน **Part 058**) หรือ web framework เต็มรูปแบบอย่าง Gin/Echo (**Part 061-063**) จะช่วยลดโค้ด boilerplate ได้มากกว่า

---

## สรุปสิ่งที่ได้เรียนในบทนี้

- `http.ServeMux` คือ router ที่จับคู่ URL pattern กับ handler ควรสร้างด้วย `http.NewServeMux()` เอง แทนการใช้ `http.DefaultServeMux` ตรงๆ เพื่อเลี่ยงปัญหา global state
- ทุกอย่างในระบบ server ของ Go คือ `http.Handler` (interface ที่มี method เดียว `ServeHTTP`) — `http.HandlerFunc` คือสะพานที่แปลงฟังก์ชันธรรมดาให้กลายเป็น `http.Handler` ได้
- `http.ResponseWriter` ใช้เขียน response, `*http.Request` เก็บข้อมูล request ทั้งหมด — อ่าน query param ด้วย `r.URL.Query().Get(...)`, อ่าน body ด้วย `io.ReadAll(r.Body)` หรือ `json.NewDecoder(r.Body).Decode(...)` โดยไม่ต้องปิด `r.Body` เอง (Go จัดการให้)
- **ลำดับสำคัญมาก**: `Header().Set(...)` → `WriteHeader(code)` → `Write(...)` เท่านั้น เรียกผิดลำดับทำให้ status code ที่ตั้งไว้ไม่มีผล พร้อม log เตือน `superfluous response.WriteHeader call` ให้สังเกตได้
- `http.ListenAndServe` รัน server จริง ควรตั้ง `ReadTimeout`/`WriteTimeout`/`IdleTimeout` ผ่าน `http.Server` เสมอ เพื่อป้องกันการโจมตีแบบ Slowloris เช่นเดียวกับที่ **Part 046** เตือนเรื่อง `Client.Timeout`
- **Graceful shutdown** ใช้ `os/signal` ดักจับ `SIGINT`/`SIGTERM` แล้วเรียก `srv.Shutdown(ctx)` ที่มี `context.WithTimeout` กำกับ — `Shutdown` จะหยุดรับ connection ใหม่ทันทีแต่รอ request ที่กำลังทำงานอยู่ให้เสร็จก่อน และคืน `http.ErrServerClosed` เป็น error ปกติเมื่อปิดตามที่สั่ง (ต้องเช็คด้วย `errors.Is`)
- **Go 1.22+ ServeMux** รองรับ `"METHOD /path"` และ wildcard `{name}`/`{name...}` ในตัว พร้อม `r.PathValue(...)` อ่านค่า และตอบ `405 Method Not Allowed` พร้อม header `Allow` ให้อัตโนมัติเมื่อ path ตรงแต่ method ไม่ตรง — เพียงพอสำหรับ REST API ขนาดเล็ก-กลางโดยไม่ต้องพึ่ง router ภายนอก

## แบบฝึกหัดท้ายบท

1. เขียน custom `http.Handler` (ไม่ใช้ `HandleFunc`) ที่เก็บตัวนับจำนวนครั้งที่ถูกเรียกไว้เป็น field ของ struct (ต้องใช้ `sync.Mutex` หรือ `sync/atomic` ป้องกัน race ตามที่เรียนใน **Part 039-040** เพราะ handler อาจถูกเรียกจากหลาย goroutine พร้อมกัน) แล้วให้ตอบกลับจำนวนครั้งที่เคยถูกเรียกในแต่ละ response
2. จำลองสถานการณ์ handler ที่เขียน header สองครั้ง (`w.WriteHeader(200)` ตามด้วย `w.WriteHeader(404)`) แล้วสังเกต log เตือนที่ Go พิมพ์ออกมา อธิบายว่าทำไม Go อนุญาตให้ "เตือน" แทนที่จะ panic ทันที
3. เพิ่ม endpoint `PUT /items/{id}` ในตัวอย่างหัวข้อ 7 ที่รองรับการอัปเดตข้อมูลบางส่วน (partial update) ของ `Item` แล้วทดสอบว่า `405 Method Not Allowed` ที่เคยเกิดขึ้นกับ `PUT` หายไปและ header `Allow` เปลี่ยนไปอย่างไร
4. เขียน middleware อย่างง่าย (ฟังก์ชันที่รับ `http.Handler` แล้วคืน `http.Handler` ใหม่) ที่ log วิธี (`r.Method`) และ path (`r.URL.Path`) ของทุก request ก่อนส่งต่อไปยัง handler จริง โดยยังไม่ต้องดูรายละเอียดเชิงลึกของ middleware pattern (จะเรียนเต็มรูปแบบใน **Part 057**)
5. ปรับปรุงตัวอย่าง graceful shutdown ในหัวข้อ 6 ให้พิมพ์คำเตือนถ้า `Shutdown` ใช้เวลารอเกิน context timeout ที่กำหนด (ลองลด `WithTimeout` ให้สั้นกว่าเวลาที่ handler `/slow` ใช้ แล้วสังเกตว่า error ที่ได้จาก `Shutdown` คืออะไร)
6. เขียน handler ที่รับ multipart form (ไฟล์อัปโหลด) เบื้องต้นด้วย `r.ParseMultipartForm` และ `r.FormFile` เป็นการเตรียมพื้นฐานก่อนไปเรียนเจาะลึกเรื่อง static files และ file upload ใน **Part 066**

---

**ต่อไป**: [Part 048 — `io.Reader` / `io.Writer` และ Interface Composition](./048-io-reader-writer.md)
