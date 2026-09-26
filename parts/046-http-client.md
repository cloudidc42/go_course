# Part 046: `net/http` — HTTP Client เจาะลึก

> ภาคที่ 4: Standard Library เชิงลึก — ตอนที่ 1 จาก 10 (Part 46–55)

## สารบัญของบทนี้

1. ภาพรวม: `net/http` แบ่งเป็นฝั่ง Client และฝั่ง Server
2. `http.Get`/`http.Post`: ใช้งานง่ายแต่มีข้อจำกัด
3. สร้าง Request เองด้วย `http.NewRequest`/`http.NewRequestWithContext`
4. การตั้งค่า Header
5. `http.Client` struct และทำไม Default Client ถึงอันตรายสำหรับ Production
6. อ่านและปิด Response Body อย่างถูกต้อง พร้อมตรวจสอบ Status Code
7. `http.Transport` และ Connection Pooling / Keep-Alive
8. Retry พร้อม Backoff (เขียนเอง)
9. ส่ง JSON Request Body
10. สรุปสิ่งที่ได้เรียนในบทนี้
11. แบบฝึกหัดท้ายบท

---

## 1. ภาพรวม: `net/http` แบ่งเป็นฝั่ง Client และฝั่ง Server

`net/http` เป็น package มาตรฐานที่ทรงพลังที่สุดตัวหนึ่งของ Go — แข็งแกร่งพอที่จะสร้าง production web server และ HTTP client ได้โดย**ไม่ต้องพึ่ง library ภายนอกเลย** ต่างจากภาษาอื่นหลายภาษาที่ต้องพึ่ง framework หรือ library แยกต่างหากสำหรับงานพื้นฐานนี้ ซึ่งสอดคล้องกับจุดเด่นที่กล่าวถึงตั้งแต่ **Part 001**: "Standard Library แข็งแกร่ง"

บทนี้และ **Part 047** จะแบ่งเนื้อหาออกเป็นสองฝั่งตามธรรมชาติของการสื่อสาร HTTP:

- **Part 046 (บทนี้)**: ฝั่ง **Client** — โปรแกรม Go ของเราเป็นผู้เรียก (request) ไปหา server อื่น เช่น เรียก REST API ภายนอก, ดึงข้อมูลจาก service อื่น
- **Part 047**: ฝั่ง **Server** — โปรแกรม Go ของเราเป็นผู้รับ (handle) request จาก client อื่น เช่น สร้าง API ของตัวเอง

ทั้งสองฝั่งใช้ type ร่วมกันจำนวนมาก เช่น `http.Request`, `http.Header`, `http.Client` (ฝั่ง server ก็ใช้ `http.Client` เวลาต้องเรียกไปยัง service อื่นต่อ) ทำให้ความรู้จากบทนี้เป็นพื้นฐานสำคัญของบทถัดไปด้วย

ก่อนเริ่ม เราจะใช้ `net/http/httptest` (package ย่อยของ `net/http` สำหรับการทดสอบ ที่จะเรียนเจาะลึกใน **Part 081**) เพื่อจำลอง HTTP server ขึ้นมาในโปรแกรมเดียวกัน แทนที่จะพึ่ง network ภายนอกที่ควบคุมไม่ได้ ทำให้ตัวอย่างในบทนี้รันซ้ำได้ผลลัพธ์เดิมทุกครั้งและไม่ต้องพึ่ง internet

---

## 2. `http.Get`/`http.Post`: ใช้งานง่ายแต่มีข้อจำกัด

`net/http` มีฟังก์ชันสำเร็จรูประดับ package ให้ใช้งานได้ทันทีโดยไม่ต้องสร้างอะไรเพิ่ม:

```go
package main

import (
	"fmt"
	"io"
	"net/http"
	"net/http/httptest"
	"strings"
)

func main() {
	// จำลอง server จริงด้วย httptest (จะเรียนเจาะลึกใน Part 081) เพื่อไม่ต้องพึ่ง network ภายนอก
	server := httptest.NewServer(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		switch r.URL.Path {
		case "/hello":
			fmt.Fprintln(w, "สวัสดีจาก server")
		case "/echo":
			body, _ := io.ReadAll(r.Body)
			fmt.Fprintf(w, "server ได้รับ: %s", body)
		}
	}))
	defer server.Close()

	// http.Get - ฟังก์ชันสำเร็จรูปสำหรับ GET request แบบง่ายที่สุด
	resp, err := http.Get(server.URL + "/hello")
	if err != nil {
		fmt.Println("get error:", err)
		return
	}
	body, _ := io.ReadAll(resp.Body)
	resp.Body.Close()
	fmt.Print("http.Get ได้ผลลัพธ์: ", string(body))

	// http.Post - ฟังก์ชันสำเร็จรูปสำหรับ POST request แบบง่ายที่สุด
	resp2, err := http.Post(server.URL+"/echo", "text/plain", strings.NewReader("ข้อมูลทดสอบ"))
	if err != nil {
		fmt.Println("post error:", err)
		return
	}
	body2, _ := io.ReadAll(resp2.Body)
	resp2.Body.Close()
	fmt.Println("http.Post ได้ผลลัพธ์:", string(body2))
}
```

ผลลัพธ์:

```
http.Get ได้ผลลัพธ์: สวัสดีจาก server
http.Post ได้ผลลัพธ์: server ได้รับ: ข้อมูลทดสอบ
```

### เจาะลึกสิ่งที่เกิดขึ้น

- `http.Get(url)` คืนค่า `(*http.Response, error)` — เป็น shortcut ของ `http.DefaultClient.Get(url)`
- `http.Post(url, contentType, body)` รับ `io.Reader` เป็น body (ในที่นี้คือ `strings.NewReader(...)` — ทบทวนแนวคิด `io.Reader` จาก **Part 024**) พร้อมระบุ Content-Type ตรงๆ เป็น argument
- ทั้งคู่**ไม่ต้องสร้าง `http.Client` เอง** — ทำงานผ่าน `http.DefaultClient` ที่เป็นตัวแปร global ของ package โดยอัตโนมัติ

### ข้อจำกัดของฟังก์ชันสำเร็จรูปเหล่านี้

ฟังก์ชัน `http.Get`/`http.Post`/`http.Head`/`http.PostForm` สะดวกมากสำหรับสคริปต์เล็กๆ หรือการทดลอง แต่มีข้อจำกัดที่ทำให้**ไม่เหมาะกับโค้ด production**:

1. **ตั้งค่า header เองไม่ได้** — ไม่มีทางส่ง `Authorization`, custom header ใดๆ เข้าไปได้เลย ต้องใช้ค่า default ที่ Go กำหนดให้เท่านั้น
2. **ผูกกับ `context.Context` ไม่ได้** — ไม่มีทาง cancel หรือกำหนด timeout เฉพาะ request นี้ได้ (ทบทวนความสำคัญของ context จาก **Part 032**)
3. **ใช้ `http.DefaultClient` เสมอ** — ซึ่งอย่างที่จะเห็นในหัวข้อ 5 คือ client ที่**ไม่มี timeout ใดๆ ทั้งสิ้น** อันตรายมากถ้านำไปใช้ตรงๆ ใน production
4. **ปรับแต่ง redirect policy, cookie jar, transport ไม่ได้**

สรุปคือ: **ใช้ได้สำหรับ script/demo/CLI เล็กๆ ที่ไม่ซีเรียส** แต่โค้ด production ควรสร้าง `*http.Request` และ `*http.Client` เองเสมอ ซึ่งเป็นเนื้อหาของหัวข้อถัดไป

---

## 3. สร้าง Request เองด้วย `http.NewRequest`/`http.NewRequestWithContext`

การควบคุม request แบบเต็มรูปแบบเริ่มจากการสร้าง `*http.Request` เองก่อนส่ง แล้วค่อยส่งผ่าน `client.Do(req)`:

```go
package main

import (
	"context"
	"fmt"
	"io"
	"net/http"
	"net/http/httptest"
	"time"
)

func main() {
	server := httptest.NewServer(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		fmt.Fprintf(w, "Authorization ที่ได้รับ: %s\n", r.Header.Get("Authorization"))
		fmt.Fprintf(w, "User-Agent ที่ได้รับ: %s\n", r.Header.Get("User-Agent"))
		fmt.Fprintf(w, "Query param name: %s\n", r.URL.Query().Get("name"))
	}))
	defer server.Close()

	// http.NewRequest สร้าง *http.Request แบบควบคุมได้เต็มที่ ก่อนส่งจริงด้วย client.Do
	req, err := http.NewRequest(http.MethodGet, server.URL+"/greet?name=Gopher", nil)
	if err != nil {
		fmt.Println("สร้าง request ไม่สำเร็จ:", err)
		return
	}

	// ตั้งค่า header ก่อนส่ง — Set เขียนทับค่าเดิมทั้งหมด, Add เพิ่มค่าใหม่ต่อท้าย (มีได้หลายค่าต่อ header เดียว)
	req.Header.Set("Authorization", "Bearer sometoken123")
	req.Header.Set("User-Agent", "go-course-client/1.0")

	client := &http.Client{Timeout: 5 * time.Second}
	resp, err := client.Do(req)
	if err != nil {
		fmt.Println("ส่ง request ไม่สำเร็จ:", err)
		return
	}
	defer resp.Body.Close()

	body, _ := io.ReadAll(resp.Body)
	fmt.Print(string(body))

	fmt.Println("---")

	// http.NewRequestWithContext ผูก context เข้ากับ request ตั้งแต่ตอนสร้าง (ทบทวนจาก Part 032)
	// เมื่อ ctx ถูกยกเลิกหรือหมดเวลา การเรียก client.Do(req) จะยกเลิก connection ทันที
	ctx, cancel := context.WithTimeout(context.Background(), 3*time.Second)
	defer cancel()

	req2, err := http.NewRequestWithContext(ctx, http.MethodGet, server.URL+"/greet?name=Context", nil)
	if err != nil {
		fmt.Println("สร้าง request ไม่สำเร็จ:", err)
		return
	}

	resp2, err := client.Do(req2)
	if err != nil {
		fmt.Println("ส่ง request ไม่สำเร็จ:", err)
		return
	}
	defer resp2.Body.Close()

	body2, _ := io.ReadAll(resp2.Body)
	fmt.Print(string(body2))
}
```

ผลลัพธ์:

```
Authorization ที่ได้รับ: Bearer sometoken123
User-Agent ที่ได้รับ: go-course-client/1.0
Query param name: Gopher
---
Authorization ที่ได้รับ: 
User-Agent ที่ได้รับ: Go-http-client/1.1
Query param name: Context
```

### `http.NewRequest` vs `http.NewRequestWithContext`

- `http.NewRequest(method, url, body)` — สร้าง request แบบพื้นฐาน ผูกกับ `context.Background()` ให้อัตโนมัติ (ไม่มีการยกเลิกใดๆ)
- `http.NewRequestWithContext(ctx, method, url, body)` — เหมือนกันทุกอย่าง แต่รับ `context.Context` เป็นพารามิเตอร์ตัวแรกตาม convention ที่เรียนใน **Part 032** ข้อ 9 — ทำให้ผูก cancellation/timeout/deadline เข้ากับ request เดียวได้อย่างละเอียด

สังเกตว่าในตัวอย่างที่สอง เราไม่ได้ตั้งค่า header ใดๆ ให้ `req2` เลย ผลลัพธ์จึงแสดง `User-Agent` เป็นค่า default ของ Go เอง (`Go-http-client/1.1`) และ `Authorization` เป็นค่าว่าง — นี่คือพฤติกรรม default ที่ทุกคนควรรู้ไว้ เพราะ API ภายนอกหลายเจ้าตรวจสอบ `User-Agent` และอาจปฏิเสธ request ที่มาจาก client ที่ระบุตัวตนไม่ชัดเจน

**คำแนะนำในทางปฏิบัติ**: ใน production ให้ใช้ `http.NewRequestWithContext` เป็นค่าเริ่มต้นเสมอ แม้ว่าจะยังไม่ได้ต้องการ cancel ทันที เพราะทำให้โค้ดพร้อมรองรับ context ที่ไหลมาจาก caller (เช่น จาก HTTP handler ของ server ตัวเองใน **Part 047**) ได้ทันทีโดยไม่ต้องแก้ signature ทีหลัง

---

## 4. การตั้งค่า Header

`req.Header` มี type เป็น `http.Header` ซึ่งจริงๆ แล้วคือ `map[string][]string` (header หนึ่งตัวมีได้หลายค่า เช่น `Set-Cookie` หลายอัน) มี method หลักที่ใช้บ่อย:

```go
req.Header.Set("Content-Type", "application/json") // เขียนทับค่าเดิมทั้งหมด (ใช้บ่อยที่สุด)
req.Header.Add("X-Custom-Tag", "a")                 // เพิ่มค่าใหม่ต่อท้าย ไม่ลบของเดิม
req.Header.Add("X-Custom-Tag", "b")                 // ตอนนี้ X-Custom-Tag มีสองค่า: "a" กับ "b"
value := req.Header.Get("Content-Type")             // อ่านค่าแรกของ header นั้น (case-insensitive)
req.Header.Del("X-Custom-Tag")                      // ลบ header ทิ้งทั้งหมด
```

Header ที่พบบ่อยที่สุดในการเขียน HTTP client:

| Header | ใช้ทำอะไร |
|---|---|
| `Content-Type` | บอกชนิดข้อมูลใน request body เช่น `application/json`, `application/x-www-form-urlencoded` |
| `Authorization` | ส่ง credential เช่น `Bearer <token>` หรือ `Basic <base64>` — จะเจาะลึกใน **Part 067-068** |
| `Accept` | บอก server ว่า client รับ response แบบไหนได้บ้าง เช่น `application/json` |
| `User-Agent` | ระบุตัวตนของ client — บาง API บังคับให้ตั้งเป็นค่าเฉพาะ ไม่งั้นปฏิเสธ request |
| `X-Request-ID` | ตัวอย่าง custom header สำหรับ tracing request ข้าม service (พบบ่อยใน microservices ที่จะเรียนใน **ภาคที่ 8**) |

**ข้อควรระวัง**: `req.Header.Get(...)` เป็น case-insensitive ตามมาตรฐาน HTTP (เช่น `Content-Type` กับ `content-type` ถือเป็น header เดียวกัน) แต่ถ้าเข้าถึง `req.Header` ในฐานะ map ตรงๆ (เช่น `req.Header["Content-Type"]`) จะเป็น case-sensitive ดังนั้นควรใช้ method `.Get()`/`.Set()`/`.Add()` เสมอ ไม่ควรเข้าถึง map ตรงๆ

---

## 5. `http.Client` struct และทำไม Default Client ถึงอันตรายสำหรับ Production

`http.Client` คือ struct หลักที่ควบคุมพฤติกรรมการส่ง request ทั้งหมด:

```go
type Client struct {
	Transport     RoundTripper                             // ควบคุมระดับ connection/transport (หัวข้อ 7)
	CheckRedirect func(req *Request, via []*Request) error  // ควบคุมว่าจะติดตาม redirect ยังไง
	Jar           CookieJar                                 // จัดการ cookie อัตโนมัติข้าม request
	Timeout       time.Duration                             // เวลาสูงสุดของ "ทั้ง request" รวมกัน
}
```

ทุก field เป็น**ค่าว่าง (zero value) ได้หมด** — `&http.Client{}` ใช้งานได้ทันทีโดยใช้ค่า default ของทุกอย่าง แต่ตรงนี้เองคือจุดที่นักพัฒนามือใหม่จำนวนมากตกหลุมพราง

### ปัญหา: `http.DefaultClient` ไม่มี Timeout เลย

`http.DefaultClient` (ตัวแปร global ที่ `http.Get`/`http.Post` ใช้อยู่เบื้องหลัง) มีค่า `Timeout` เป็น **`0`** ซึ่งใน Go หมายถึง **"ไม่มีการจำกัดเวลาเลย"** — ถ้า server ปลายทางไม่ตอบกลับ (เครือข่ายมีปัญหา, server ค้าง, service ตายกลางคัน) โปรแกรมของเราจะ**ค้างรอตลอดไปโดยไม่มีวันหลุดออกมาเอง**

มาพิสูจน์ให้เห็นภาพจริง:

```go
package main

import (
	"fmt"
	"net/http"
	"net/http/httptest"
)

func main() {
	// server จำลองที่ไม่ตอบกลับเลย (ค้างตลอดไปโดยเจตนา) เพื่อพิสูจน์ว่า http.DefaultClient
	// (หรือ client ที่ไม่ได้ตั้งค่า Timeout) จะรอ "ตลอดไป" โดยไม่มีการยกเลิกเองเลย
	server := httptest.NewServer(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		select {} // บล็อกตลอดไป จำลอง server ที่ค้าง/ตายกลางคัน
	}))
	defer server.Close()

	fmt.Println("กำลังส่ง request ด้วย http.DefaultClient (ไม่มี timeout)...")
	// อันตราย: ไม่มีการจำกัดเวลาใดๆ เลย ถ้า server ไม่ตอบ โปรแกรมนี้จะค้างตลอดไป
	resp, err := http.DefaultClient.Get(server.URL)
	if err != nil {
		fmt.Println("error:", err)
		return
	}
	defer resp.Body.Close()
	fmt.Println("ได้รับ response แล้ว (บรรทัดนี้จะไม่มีวันถูกพิมพ์)")
}
```

ถ้ารันโปรแกรมนี้ตรงๆ ด้วย `go run main.go` โปรแกรมจะ**ค้างไปตลอดกาล** (ต้องกด Ctrl+C เพื่อฆ่าเอง) มายืนยันด้วยการจำกัดเวลารันจากภายนอกด้วยคำสั่ง `timeout` ของ Unix:

```bash
timeout 3 go run main.go
echo "exit code: $?"
```

ผลลัพธ์:

```
กำลังส่ง request ด้วย http.DefaultClient (ไม่มี timeout)...
exit code: 124
```

`exit code: 124` คือรหัสที่คำสั่ง `timeout` ใช้บอกว่า**มันต้องบังคับฆ่าโปรแกรมทิ้งเอง**เพราะโปรแกรมไม่จบภายในเวลาที่กำหนด และสังเกตว่าบรรทัด `"ได้รับ response แล้ว"` ไม่ถูกพิมพ์ออกมาเลย — เป็นหลักฐานชัดเจนว่า `client.Get()` ค้างอยู่ตรงนั้นจริงๆ ไม่มีวันคืนค่าเองถ้าไม่มีใครมาช่วยตัดจบจากภายนอก

ในระบบ production จริง สถานการณ์แบบนี้เกิดขึ้นบ่อยกว่าที่คิด: service ปลายทางที่เรียกอาจ deploy ผิดพลาดจนค้าง, load balancer routing ผิดไปยัง instance ที่ตายแล้วแต่ TCP connection ยังไม่ถูกตัด, หรือ network partition ทำให้ packet หายไปเงียบๆ — ถ้า client ของเราไม่มี timeout สิ่งที่จะเกิดขึ้นคือ **goroutine ที่รอ request นี้ค้างอยู่ตลอดไป** สะสมไปเรื่อยๆ จนกระทั่ง memory หมดและทั้งระบบล่มตามไปด้วย (goroutine leak ที่ทบทวนแนวคิดมาจาก **ภาคที่ 3**)

### วิธีแก้: ตั้งค่า `Timeout` เสมอ

```go
package main

import (
	"fmt"
	"net/http"
	"net/http/httptest"
	"time"
)

func main() {
	server := httptest.NewServer(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		select {} // จำลอง server ที่ค้างตลอดไปเหมือนเดิม
	}))
	// หมายเหตุ: ไม่เรียก server.Close() ในตัวอย่างนี้ เพราะ handler ค้างอยู่ใน select{}
	// ตลอดไปโดยเจตนา ทำให้ Close() ต้องรอ handler goroutine จบก่อนซึ่งไม่มีวันเกิดขึ้น
	// (ในโปรแกรมจริงที่ยิง request ออกไปยัง server ภายนอก จะไม่เจอปัญหานี้)

	// แก้ปัญหา: สร้าง client ของตัวเองเสมอ พร้อมกำหนด Timeout ให้ชัดเจน
	// Timeout นี้ครอบคลุมทั้ง connection + ส่ง request + รอ response + อ่าน body ทั้งหมด "รวมกัน"
	client := &http.Client{
		Timeout: 2 * time.Second,
	}

	fmt.Println("กำลังส่ง request ด้วย client ที่ตั้ง Timeout ไว้ 2 วินาที...")
	start := time.Now()
	resp, err := client.Get(server.URL)
	elapsed := time.Since(start)

	if err != nil {
		fmt.Printf("ยกเลิกหลังจากรอ %v: %v\n", elapsed.Round(time.Millisecond), err)
		return
	}
	defer resp.Body.Close()
	fmt.Println("ได้รับ response แล้ว")
}
```

ผลลัพธ์:

```
กำลังส่ง request ด้วย client ที่ตั้ง Timeout ไว้ 2 วินาที...
ยกเลิกหลังจากรอ 2.002s: Get "http://127.0.0.1:xxxxx": context deadline exceeded (Client.Timeout exceeded while awaiting headers)
```

คราวนี้โปรแกรมจบการทำงานได้เองอย่างถูกต้องหลังจากรอครบ 2 วินาทีตามที่กำหนด แทนที่จะค้างตลอดไป

### `Client.Timeout` vs `context.WithTimeout` — ใช้ตัวไหนดี

ทั้งสองวิธีคุมเวลาได้เหมือนกัน แต่มีจุดต่างที่สำคัญ:

| | `Client.Timeout` | `context.WithTimeout` ที่ผูกกับ request |
|---|---|---|
| ขอบเขต | ครอบคลุม**ทุก request** ที่ยิงผ่าน client ตัวนี้เท่ากันหมด | กำหนดแยกเป็น**รายครั้ง**ต่อ request ได้ |
| ครอบคลุมอะไรบ้าง | ทั้ง connect + ส่ง header + รอ response + อ่าน body ทั้งหมด | เหมือนกัน แต่ยกเลิกได้จากภายนอกด้วย (เช่น user กดปิดหน้าเว็บ) |
| เหมาะกับ | เป็น **safety net ขั้นต่ำสุด** ที่ต้องมีเสมอไม่ว่าอะไรจะเกิดขึ้น | ควบคุมละเอียดตาม business logic ของแต่ละ request |

**แนวทางที่แนะนำในทางปฏิบัติ**: ตั้งค่า `Client.Timeout` ไว้เป็น**เพดานบนสุดที่ไม่ควรเกินไม่ว่ากรณีใด** (เช่น 30 วินาที) เสมอ เป็นตาข่ายนิรภัยชั้นสุดท้าย แล้วใช้ `context.WithTimeout` ควบคุมเวลาที่ละเอียดกว่านั้นในระดับ request แต่ละครั้งซ้อนทับเข้าไปอีกชั้น — สองอย่างนี้**ไม่ขัดแย้งกัน** ใครถึงกำหนดก่อนก็ยกเลิกก่อน

> **กฎทอง**: **ห้ามใช้ `http.DefaultClient`, `http.Get`, `http.Post` ตรงๆ ในโค้ด production เด็ดขาด** ให้สร้าง `*http.Client` ของตัวเองพร้อม `Timeout` ที่เหมาะสมเสมอ แม้จะเป็นแค่ script เล็กๆ ก็ควรทำเป็นนิสัย

---

## 6. อ่านและปิด Response Body อย่างถูกต้อง พร้อมตรวจสอบ Status Code

`*http.Response` ที่ได้จาก `client.Do(req)` มี field `Body io.ReadCloser` ซึ่งเป็นทั้ง `io.Reader` (อ่านข้อมูลได้) และ `io.Closer` (ปิดได้) — จะเจาะลึก interface ประกอบร่างแบบนี้ใน **Part 048** แต่ตอนนี้สิ่งที่ต้องรู้คือกฎการใช้งานที่ถูกต้อง:

```go
package main

import (
	"fmt"
	"io"
	"net/http"
	"net/http/httptest"
	"time"
)

func fetch(client *http.Client, url string) (string, error) {
	resp, err := client.Get(url)
	if err != nil {
		// สำคัญ: ถ้า err != nil ห้ามแตะ resp เลย (resp อาจเป็น nil) และไม่ต้อง Close ใดๆ
		return "", fmt.Errorf("เรียก request ไม่สำเร็จ: %w", err)
	}
	// วางไว้ทันทีหลังเช็ค err ว่าไม่ใช่ nil เพื่อรับประกันว่า connection จะถูกคืนกลับ pool เสมอ
	defer resp.Body.Close()

	// ต้องเช็ค status code เอง — client.Do/Get จะคืน err เป็น nil แม้ status จะเป็น 4xx/5xx ก็ตาม!
	// (err ที่ไม่ใช่ nil หมายถึงปัญหาระดับเครือข่าย เช่น เชื่อมต่อไม่ได้ หรือ timeout เท่านั้น)
	if resp.StatusCode < 200 || resp.StatusCode >= 300 {
		io.Copy(io.Discard, resp.Body) // อ่านทิ้งให้หมดก่อน close เพื่อให้ connection นำกลับมาใช้ซ้ำได้ (หัวข้อถัดไป)
		return "", fmt.Errorf("server ตอบสถานะผิดพลาด: %s", resp.Status)
	}

	body, err := io.ReadAll(resp.Body)
	if err != nil {
		return "", fmt.Errorf("อ่าน body ไม่สำเร็จ: %w", err)
	}
	return string(body), nil
}

func main() {
	server := httptest.NewServer(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		switch r.URL.Path {
		case "/ok":
			fmt.Fprintln(w, "ทุกอย่างเรียบร้อย")
		case "/notfound":
			w.WriteHeader(http.StatusNotFound)
			fmt.Fprintln(w, "ไม่พบข้อมูลที่ต้องการ")
		case "/servererror":
			w.WriteHeader(http.StatusInternalServerError)
			fmt.Fprintln(w, "server พังภายใน")
		}
	}))
	defer server.Close()

	client := &http.Client{Timeout: 5 * time.Second}

	for _, path := range []string{"/ok", "/notfound", "/servererror"} {
		result, err := fetch(client, server.URL+path)
		if err != nil {
			fmt.Println("error:", err)
			continue
		}
		fmt.Print("สำเร็จ: ", result)
	}
}
```

ผลลัพธ์:

```
สำเร็จ: ทุกอย่างเรียบร้อย
error: server ตอบสถานะผิดพลาด: 404 Not Found
error: server ตอบสถานะผิดพลาด: 500 Internal Server Error
```

### กฎ 3 ข้อที่ต้องจำให้ขึ้นใจ

1. **`err != nil` ไม่ได้แปลว่า HTTP status ผิดพลาด** — `client.Do`/`Get`/`Post` จะคืน `err` เป็น non-nil ก็ต่อเมื่อเกิดปัญหา**ระดับเครือข่าย** เท่านั้น (เชื่อมต่อไม่ได้, DNS resolve ไม่ได้, timeout) ส่วน HTTP status 404 หรือ 500 นั้น Go มองว่า **"เชื่อมต่อสำเร็จและได้ response กลับมาแล้ว"** จึงคืน `err` เป็น `nil` เสมอ — เราต้อง**เช็ค `resp.StatusCode` เอง**ทุกครั้ง
2. **ปิด `resp.Body` เสมอด้วย `defer` ทันทีหลังเช็ค `err`** — ถ้าลืมปิด จะทำให้ connection ค้างอยู่และไม่ถูกคืนกลับเข้า pool (อธิบายเจาะลึกในหัวข้อถัดไป) สะสมนานเข้าจะทำให้ระบบ "หมด file descriptor" ได้ในที่สุด
3. **อ่าน body ให้หมดก่อน close ถ้าต้องการนำ connection กลับมาใช้ซ้ำ** — แม้ในกรณีที่ status code ผิดพลาดและเราไม่สนใจเนื้อหา body เลย ควร `io.Copy(io.Discard, resp.Body)` เพื่ออ่านข้อมูลที่เหลือทิ้งไปให้หมดก่อน `Close()` มิฉะนั้น Go จะไม่สามารถนำ TCP connection นั้นไปใช้ซ้ำกับ request ถัดไปได้ (ต้องเปิด connection ใหม่ทุกครั้งแทน ซึ่งช้ากว่ามาก)

---

## 7. `http.Transport` และ Connection Pooling / Keep-Alive

`http.Client` ไม่ได้เป็นผู้จัดการ TCP connection โดยตรง — หน้าที่นั้นเป็นของ **`http.Transport`** (ค่า default คือ `http.DefaultTransport`) ซึ่งทำหน้าที่สำคัญมากที่มักถูกมองข้าม: **เก็บ connection ที่ใช้เสร็จแล้วไว้ใน pool เพื่อนำมาใช้ซ้ำกับ request ถัดไปที่ไปยัง host เดียวกัน** (HTTP Keep-Alive) แทนที่จะต้องเปิด TCP connection ใหม่ทุกครั้ง ซึ่งมีค่าใช้จ่ายสูง (ต้องทำ TCP handshake ใหม่ และถ้าเป็น HTTPS ต้องทำ TLS handshake ใหม่ด้วย — ทั้งสองอย่างมีค่า latency ที่สูงกว่าการส่งข้อมูลจริงเสียอีก)

มาดูพฤติกรรมการ reuse connection แบบเห็นภาพจริงด้วย `net/http/httptrace`:

```go
package main

import (
	"context"
	"fmt"
	"io"
	"net/http"
	"net/http/httptest"
	"net/http/httptrace"
	"time"
)

// requestAndReportReuse ยิง request หนึ่งครั้งผ่าน client ที่ให้มา แล้วรายงานว่า
// connection ที่ใช้เป็น connection เก่าที่ถูกนำมาใช้ซ้ำ (reused) หรือเปิดใหม่
func requestAndReportReuse(client *http.Client, url string, label string) {
	var reused bool
	trace := &httptrace.ClientTrace{
		GotConn: func(info httptrace.GotConnInfo) {
			reused = info.Reused
		},
	}
	ctx := httptrace.WithClientTrace(context.Background(), trace)

	req, _ := http.NewRequestWithContext(ctx, http.MethodGet, url, nil)
	resp, err := client.Do(req)
	if err != nil {
		fmt.Println(label, "error:", err)
		return
	}
	io.Copy(io.Discard, resp.Body) // ต้องอ่านให้หมดก่อน Close เพื่อให้ connection นำกลับไปใช้ซ้ำได้
	resp.Body.Close()

	fmt.Printf("%s: connection ถูกนำมาใช้ซ้ำ (reused) = %v\n", label, reused)
}

func main() {
	server := httptest.NewServer(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		fmt.Fprintln(w, "pong")
	}))
	defer server.Close()

	fmt.Println("=== ใช้ client ตัวเดียวกันซ้ำหลายครั้ง (ค่า default เปิด keep-alive) ===")
	sharedClient := &http.Client{}
	requestAndReportReuse(sharedClient, server.URL, "ครั้งที่ 1")
	requestAndReportReuse(sharedClient, server.URL, "ครั้งที่ 2")
	requestAndReportReuse(sharedClient, server.URL, "ครั้งที่ 3")

	fmt.Println()
	fmt.Println("=== สร้าง http.Client{} ใหม่ทุกครั้ง (แต่ไม่ได้กำหนด Transport เอง) ===")
	requestAndReportReuse(&http.Client{}, server.URL, "client ใหม่ 1")
	requestAndReportReuse(&http.Client{}, server.URL, "client ใหม่ 2")

	fmt.Println()
	fmt.Println("=== สร้าง Transport แยกกันจริงๆ ทุกครั้ง (pool ไม่ถูกแชร์ระหว่างกันเลย) ===")
	requestAndReportReuse(&http.Client{Transport: &http.Transport{}}, server.URL, "transport แยก 1")
	requestAndReportReuse(&http.Client{Transport: &http.Transport{}}, server.URL, "transport แยก 2")

	fmt.Println()
	fmt.Println("=== ปรับแต่ง Transport เอง: จำกัด MaxIdleConnsPerHost ===")
	customClient := &http.Client{
		Transport: &http.Transport{
			MaxIdleConns:        100,
			MaxIdleConnsPerHost: 2,                // ปกติ default คือ 2 อยู่แล้ว แต่ระบุชัดเจนเพื่อสาธิต
			IdleConnTimeout:     30 * time.Second, // ปิด connection ที่ไม่ได้ใช้งานเกิน 30 วินาทีทิ้ง
		},
	}
	requestAndReportReuse(customClient, server.URL, "custom transport 1")
	requestAndReportReuse(customClient, server.URL, "custom transport 2")
}
```

ผลลัพธ์:

```
=== ใช้ client ตัวเดียวกันซ้ำหลายครั้ง (ค่า default เปิด keep-alive) ===
ครั้งที่ 1: connection ถูกนำมาใช้ซ้ำ (reused) = false
ครั้งที่ 2: connection ถูกนำมาใช้ซ้ำ (reused) = true
ครั้งที่ 3: connection ถูกนำมาใช้ซ้ำ (reused) = true

=== สร้าง http.Client{} ใหม่ทุกครั้ง (แต่ไม่ได้กำหนด Transport เอง) ===
client ใหม่ 1: connection ถูกนำมาใช้ซ้ำ (reused) = true
client ใหม่ 2: connection ถูกนำมาใช้ซ้ำ (reused) = true

=== สร้าง Transport แยกกันจริงๆ ทุกครั้ง (pool ไม่ถูกแชร์ระหว่างกันเลย) ===
transport แยก 1: connection ถูกนำมาใช้ซ้ำ (reused) = false
transport แยก 2: connection ถูกนำมาใช้ซ้ำ (reused) = false

=== ปรับแต่ง Transport เอง: จำกัด MaxIdleConnsPerHost ===
custom transport 1: connection ถูกนำมาใช้ซ้ำ (reused) = false
custom transport 2: connection ถูกนำมาใช้ซ้ำ (reused) = true
```

### ข้อสังเกตสำคัญที่พลิกความเข้าใจ

ผลลัพธ์ช่วงที่สองอาจทำให้แปลกใจ: แม้เราจะสร้าง **`&http.Client{}` ใหม่ทุกครั้ง** connection ก็ยัง**ถูกนำมาใช้ซ้ำได้** นี่คือรายละเอียดสำคัญที่หลายคนเข้าใจผิด:

> **connection pool อยู่ที่ระดับ `Transport` ไม่ใช่ระดับ `Client`** เมื่อสร้าง `http.Client{}` โดยไม่ได้ระบุ field `Transport` เอง Go จะใช้ **`http.DefaultTransport`** ซึ่งเป็น**ตัวแปร global ตัวเดียวที่ถูกแชร์กันทั้งโปรแกรม** ให้อัตโนมัติ ดังนั้นแม้จะสร้าง `Client` ใหม่กี่ตัวก็ตาม ถ้าไม่ตั้ง `Transport` เอง ทุกตัวจะแชร์ connection pool เดียวกันอยู่ดี

ในทางกลับกัน เมื่อเราสร้าง **`&http.Transport{}` ใหม่จริงๆ** ให้แต่ละ client (ช่วงที่สาม) แต่ละตัวจะมี pool ของตัวเองแยกขาดจากกันโดยสมบูรณ์ ทำให้ reuse ไม่เกิดขึ้นข้ามกันเลย (`reused = false` ทั้งคู่)

**ข้อสรุปเชิงปฏิบัติ**: ในโปรแกรมจริง ควร**สร้าง `http.Client` เพียงตัวเดียว (หรือน้อยตัว) แล้วใช้ซ้ำตลอดอายุโปรแกรม** (เก็บไว้เป็น package-level variable หรือ field ของ struct ที่ inject เข้าไป) แทนที่จะสร้าง `&http.Client{...}` ใหม่ทุกครั้งที่ต้องการยิง request — แม้จะไม่ทำให้เกิดปัญหาทันทีถ้าใช้ `Transport` default ร่วมกัน แต่ถ้ามีการปรับแต่ง `Transport` เอง (ซึ่งพบบ่อยในระบบจริงที่ต้องคุม timeout ระดับ connection อย่างละเอียด) การสร้าง `Transport` ใหม่ทุกครั้งจะทำลาย connection pool ทิ้งอย่างสิ้นเชิง เสียประโยชน์ของ keep-alive ไปโดยไม่รู้ตัว

### Field สำคัญของ `http.Transport`

| Field | ความหมาย |
|---|---|
| `MaxIdleConns` | จำนวน idle connection สูงสุดที่เก็บไว้ใน pool ทั้งหมด (ทุก host รวมกัน) |
| `MaxIdleConnsPerHost` | จำนวน idle connection สูงสุดต่อ host เดียว (default คือ 2 — ค่อนข้างต่ำ ถ้ายิง request จำนวนมากไปยัง host เดียวพร้อมกันควรเพิ่มค่านี้) |
| `IdleConnTimeout` | ปิด idle connection ที่ไม่ได้ใช้งานเกินเวลานี้ทิ้งไปเลย |
| `MaxConnsPerHost` | จำกัดจำนวน connection ทั้งหมด (ทั้ง idle และ active) ต่อ host ไม่ให้เกินค่านี้ |
| `DisableKeepAlives` | ตั้งเป็น `true` เพื่อปิดการ reuse connection ทั้งหมด (ปกติไม่ควรปิด ยกเว้นมีเหตุผลเฉพาะทาง) |

---

## 8. Retry พร้อม Backoff (เขียนเอง)

ในระบบจริง การเรียก HTTP ไปยัง service ภายนอกอาจล้มเหลวแบบชั่วคราว (transient failure) เช่น server กำลัง restart, load balancer เพิ่งสลับ instance ได้ไม่นาน — การ **retry** พร้อมหน่วงเวลาแบบ **exponential backoff** (หน่วงเวลาเพิ่มขึ้นเป็นเท่าตัวทุกครั้งที่ล้มเหลว) เป็นแนวทางมาตรฐานที่ช่วยให้ระบบ "รอด" จากปัญหาชั่วคราวโดยไม่ต้อง fail ทันที (หลักการ retry แบบเต็มรูปแบบพร้อม circuit breaker จะเรียนเจาะลึกใน **Part 094**)

```go
package main

import (
	"fmt"
	"io"
	"net/http"
	"net/http/httptest"
	"sync/atomic"
	"time"
)

// doWithRetry ยิง request ซ้ำสูงสุด maxAttempts ครั้ง ด้วย exponential backoff
// สำคัญ: สร้าง *http.Request "ใหม่" ทุกครั้งที่ retry ผ่าน newReq เพราะ Body ของ
// http.Request เป็น io.ReadCloser ที่ถูก "อ่านจนหมด" ไปแล้วหลัง request แรกล้มเหลว
// นำ Request เดิมมาส่งซ้ำตรงๆ ไม่ได้ (Body จะว่างเปล่าในการส่งครั้งที่ 2 เป็นต้นไป)
func doWithRetry(client *http.Client, newReq func() (*http.Request, error), maxAttempts int) (*http.Response, error) {
	var lastErr error
	for attempt := 1; attempt <= maxAttempts; attempt++ {
		req, err := newReq()
		if err != nil {
			return nil, fmt.Errorf("สร้าง request ไม่สำเร็จ: %w", err)
		}

		resp, err := client.Do(req)
		if err == nil && resp.StatusCode < 500 {
			// สำเร็จ หรือ error ฝั่ง client (4xx) ที่ retry ไปก็ไม่มีประโยชน์ — คืนค่าทันที
			return resp, nil
		}

		if err != nil {
			lastErr = err
		} else {
			lastErr = fmt.Errorf("server ตอบสถานะ %s", resp.Status)
			io.Copy(io.Discard, resp.Body)
			resp.Body.Close()
		}

		if attempt == maxAttempts {
			break
		}

		// exponential backoff: 100ms, 200ms, 400ms, ... เพิ่มเป็นสองเท่าทุกครั้งที่ retry
		backoff := 100 * time.Millisecond * time.Duration(1<<(attempt-1))
		fmt.Printf("ครั้งที่ %d ล้มเหลว (%v) รอ %v แล้วลองใหม่\n", attempt, lastErr, backoff)
		time.Sleep(backoff)
	}
	return nil, fmt.Errorf("ลองครบ %d ครั้งแล้วยังไม่สำเร็จ: %w", maxAttempts, lastErr)
}

func main() {
	var failCount int32
	server := httptest.NewServer(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		// จำลอง server ที่ล้มเหลว 2 ครั้งแรก แล้วสำเร็จในครั้งที่ 3
		n := atomic.AddInt32(&failCount, 1)
		if n <= 2 {
			w.WriteHeader(http.StatusServiceUnavailable)
			return
		}
		fmt.Fprintln(w, "สำเร็จแล้ว!")
	}))
	defer server.Close()

	client := &http.Client{Timeout: 5 * time.Second}

	newReq := func() (*http.Request, error) {
		return http.NewRequest(http.MethodGet, server.URL, nil)
	}

	resp, err := doWithRetry(client, newReq, 5)
	if err != nil {
		fmt.Println("สุดท้ายล้มเหลว:", err)
		return
	}
	defer resp.Body.Close()

	body, _ := io.ReadAll(resp.Body)
	fmt.Print("ผลลัพธ์สุดท้าย: ", string(body))
}
```

ผลลัพธ์:

```
ครั้งที่ 1 ล้มเหลว (server ตอบสถานะ 503 Service Unavailable) รอ 100ms แล้วลองใหม่
ครั้งที่ 2 ล้มเหลว (server ตอบสถานะ 503 Service Unavailable) รอ 200ms แล้วลองใหม่
ผลลัพธ์สุดท้าย: สำเร็จแล้ว!
```

### จุดที่ต้องระวังที่สุดของการเขียน retry เอง

**ทำไมต้องรับ `newReq func() (*http.Request, error)` แทนที่จะรับ `*http.Request` ตรงๆ**: field `Body` ของ `http.Request` เป็น `io.ReadCloser` ซึ่งเมื่อ Go ส่ง request ออกไปแล้ว จะ**อ่าน stream นั้นจนหมด** เพื่อเขียนลง connection — ถ้านำ `*http.Request` ตัวเดิมมาส่งซ้ำในการ retry ครั้งถัดไป **`Body` จะว่างเปล่า** (อ่านไปครั้งเดียวจบ ไม่สามารถ "rewind" กลับไปอ่านใหม่ได้เอง) ทำให้ POST/PUT request ที่มี body จะส่งข้อมูลว่างเปล่าไปในการ retry ครั้งที่ 2 เป็นต้นไปโดยไม่มี error ใดๆ เตือนเลย — เป็นบั๊กที่พบบ่อยและตรวจจับยากมากในโค้ด production จริง วิธีแก้ที่ปลอดภัยที่สุดคือสร้าง `*http.Request` ใหม่ทุกครั้งที่ retry ผ่านฟังก์ชัน factory แบบในตัวอย่างนี้

**หมายเหตุ**: ถ้า request มาจาก source ที่ "rewind" ได้ (เช่น `bytes.Reader`, `strings.Reader`) `http.NewRequest` จะตั้งค่า field `req.GetBody` ให้อัตโนมัติ ซึ่งเป็นฟังก์ชันสำหรับสร้าง `Body` ใหม่โดยไม่ต้องสร้าง `*http.Request` ใหม่ทั้งก้อน — แต่การใช้ factory function แบบในตัวอย่างเป็นวิธีที่ตรงไปตรงมาและปลอดภัยกว่าสำหรับ retry logic ที่เขียนเอง

**เลือก retry เฉพาะ error ที่ควร retry**: โค้ดข้างต้นจงใจ **ไม่ retry เมื่อ status code เป็น 4xx** (เช่น 400 Bad Request, 404 Not Found) เพราะ error เหล่านี้เป็นปัญหาที่ฝั่ง client ส่งข้อมูลผิดเอง ต่อให้ retry กี่ครั้งก็ได้ผลลัพธ์เดิมเสมอ (เสียเวลาและทรัพยากรฟรีๆ) ควร retry เฉพาะ error ระดับเครือข่าย (`err != nil`) หรือ 5xx (ปัญหาฝั่ง server ที่อาจเป็นแค่ชั่วคราว) เท่านั้น

---

## 9. ส่ง JSON Request Body

งานที่พบบ่อยที่สุดของ HTTP client ในโลกจริงคือการเรียก REST API ที่รับ-ส่งข้อมูลเป็น JSON มาผสานความรู้จาก **Part 025 (JSON)** เข้ากับสิ่งที่เรียนมาทั้งบทนี้:

```go
package main

import (
	"bytes"
	"encoding/json"
	"fmt"
	"net/http"
	"net/http/httptest"
	"time"
)

// CreateUserRequest คือ struct ที่จะถูกแปลงเป็น JSON ก่อนส่งไปเป็น request body
// ใช้ struct tag `json:"..."` ตามที่เรียนไปใน Part 025
type CreateUserRequest struct {
	Name  string `json:"name"`
	Email string `json:"email"`
}

// CreateUserResponse คือรูปแบบ JSON ที่คาดว่า server จะตอบกลับมา
type CreateUserResponse struct {
	ID      int    `json:"id"`
	Name    string `json:"name"`
	Created bool   `json:"created"`
}

func main() {
	server := httptest.NewServer(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		var reqBody CreateUserRequest
		if err := json.NewDecoder(r.Body).Decode(&reqBody); err != nil {
			http.Error(w, "invalid json", http.StatusBadRequest)
			return
		}

		w.Header().Set("Content-Type", "application/json")
		w.WriteHeader(http.StatusCreated)
		json.NewEncoder(w).Encode(CreateUserResponse{
			ID:      42,
			Name:    reqBody.Name,
			Created: true,
		})
	}))
	defer server.Close()

	// 1. แปลง struct เป็น JSON ด้วย json.Marshal (ทบทวนจาก Part 025)
	payload := CreateUserRequest{Name: "สมชาย", Email: "somchai@example.com"}
	jsonBytes, err := json.Marshal(payload)
	if err != nil {
		fmt.Println("marshal error:", err)
		return
	}

	// 2. ห่อ []byte ด้วย bytes.NewReader (จะเรียนเจาะลึก package bytes ใน Part 050)
	//    เพื่อให้ได้ io.Reader ตามที่ http.NewRequest ต้องการเป็น body
	req, err := http.NewRequest(http.MethodPost, server.URL+"/users", bytes.NewReader(jsonBytes))
	if err != nil {
		fmt.Println("สร้าง request ไม่สำเร็จ:", err)
		return
	}
	// 3. ต้องตั้ง Content-Type เอง — Go ไม่เดาให้อัตโนมัติว่า body เป็น JSON
	req.Header.Set("Content-Type", "application/json")

	client := &http.Client{Timeout: 5 * time.Second}
	resp, err := client.Do(req)
	if err != nil {
		fmt.Println("ส่ง request ไม่สำเร็จ:", err)
		return
	}
	defer resp.Body.Close()

	// 4. อ่าน response body กลับมาเป็น struct โดยตรงด้วย json.NewDecoder (streaming decode)
	var result CreateUserResponse
	if err := json.NewDecoder(resp.Body).Decode(&result); err != nil {
		fmt.Println("decode error:", err)
		return
	}

	fmt.Printf("สร้างผู้ใช้สำเร็จ: ID=%d ชื่อ=%s สถานะ=%v (HTTP %d)\n",
		result.ID, result.Name, result.Created, resp.StatusCode)
}
```

ผลลัพธ์:

```
สร้างผู้ใช้สำเร็จ: ID=42 ชื่อ=สมชาย สถานะ=true (HTTP 201)
```

### สรุป pattern มาตรฐานของการยิง JSON API

1. **Marshal struct → `[]byte`** ด้วย `json.Marshal` (หรือใช้ `json.NewEncoder` เขียนตรงลง `bytes.Buffer` ถ้าต้องการ streaming ก็ได้)
2. **ห่อ `[]byte` เป็น `io.Reader`** ด้วย `bytes.NewReader(...)` เพื่อส่งเป็น request body (จะเจาะลึกเรื่อง `bytes.Buffer`/`bytes.Reader` ใน **Part 050**)
3. **ตั้ง `Content-Type: application/json` เสมอ** — Go ไม่มีทางรู้เองว่า body ที่ส่งเป็น JSON ถ้าไม่บอก server หลายเจ้าจะปฏิเสธ request ทันทีถ้า header นี้ขาดหรือผิด
4. **Decode response กลับด้วย `json.NewDecoder(resp.Body).Decode(&result)`** แทนที่จะ `io.ReadAll` แล้วค่อย `json.Unmarshal` แยกสองขั้นตอน — วิธีนี้ **stream ข้อมูลตรงจาก connection เข้า decoder โดยไม่ต้องพักข้อมูลทั้งก้อนไว้ใน memory ก่อน** ประหยัดกว่าเมื่อ response มีขนาดใหญ่ (หลักการเดียวกับที่เรียนใน **Part 025 หัวข้อ 8**)

---

## สรุปสิ่งที่ได้เรียนในบทนี้

- `http.Get`/`http.Post` สะดวกแต่มีข้อจำกัดมาก (ตั้ง header ไม่ได้, ผูก context ไม่ได้, ใช้ `http.DefaultClient` ที่ไม่มี timeout) — เหมาะกับ script เล็กๆ เท่านั้น ไม่ควรใช้ใน production
- `http.NewRequest`/`http.NewRequestWithContext` สร้าง request แบบควบคุมได้เต็มที่ ควรใช้ `WithContext` เป็นค่าเริ่มต้นเสมอเพื่อรองรับ cancellation/timeout ตาม convention จาก **Part 032**
- ตั้งค่า header ผ่าน `req.Header.Set`/`.Add`/`.Get`/`.Del` — `Set` เขียนทับ, `Add` เพิ่มค่าใหม่ต่อท้าย
- **`http.DefaultClient` ไม่มี `Timeout` เลย** — เป็นอันตรายร้ายแรงที่พิสูจน์ได้จริงด้วยการจำลอง server ที่ค้าง (`select {}`) แล้วรัน `timeout N go run main.go` ดู exit code — ต้องสร้าง `&http.Client{Timeout: ...}` ของตัวเองเสมอ
- ต้องเช็ค `resp.StatusCode` เองเสมอ (err เป็น nil ไม่ได้แปลว่า status สำเร็จ) และปิด `resp.Body` ด้วย `defer` ทันทีหลังเช็ค err
- **Connection pool อยู่ที่ระดับ `http.Transport` ไม่ใช่ `http.Client`** — ควรใช้ client (และ transport) ตัวเดิมซ้ำตลอดอายุโปรแกรม เพื่อให้ keep-alive ทำงานเต็มประสิทธิภาพ อ่าน body ให้หมดก่อน close เพื่อให้ connection กลับเข้า pool ได้
- Retry ต้องสร้าง `*http.Request` ใหม่ทุกครั้ง (ผ่าน factory function) เพราะ `Body` อ่านซ้ำไม่ได้ และควร retry เฉพาะ error เครือข่ายหรือ 5xx เท่านั้น ไม่ retry 4xx
- ส่ง JSON body ด้วย pattern: `json.Marshal` → `bytes.NewReader` → ตั้ง `Content-Type` → `json.NewDecoder(resp.Body).Decode(...)` สำหรับอ่านกลับ

## แบบฝึกหัดท้ายบท

1. เขียนฟังก์ชัน `fetchJSON[T any](client *http.Client, url string) (T, error)` โดยใช้ generics (ทบทวนจาก **Part 028-029**) ที่ยิง GET request แล้ว decode JSON response กลับมาเป็น type `T` ที่ระบุ พร้อมเช็ค status code และปิด body อย่างถูกต้อง
2. ปรับปรุงฟังก์ชัน `doWithRetry` ในหัวข้อ 8 ให้รับ `context.Context` เพิ่ม แล้วให้หยุด retry ทันทีถ้า `ctx` ถูกยกเลิกระหว่างรอ backoff (ใบ้: ใช้ `select` ระหว่าง `time.After(backoff)` กับ `ctx.Done()` แทน `time.Sleep` ตรงๆ)
3. เขียนโปรแกรมที่สร้าง `http.Client` สองตัว — ตัวหนึ่งไม่ตั้ง `Timeout` เลย อีกตัวตั้ง `Timeout` ไว้ 1 วินาที ทั้งคู่ยิงไปยัง `httptest.Server` ที่ตอบช้า 3 วินาที แล้วเปรียบเทียบผลลัพธ์และเวลาที่ใช้จริง
4. เพิ่ม header `X-Retry-Count` ให้กับ request ในตัวอย่างหัวข้อ 8 โดยใส่ค่าเป็นจำนวนครั้งที่พยายามไปแล้ว (attempt) เพื่อให้ server (หรือ log) รู้ว่านี่คือการ retry ครั้งที่เท่าไร
5. ใช้ `net/http/httptrace` เพิ่มเติมจากหัวข้อ 7 เพื่อวัดเวลาที่ใช้ในแต่ละ phase ของ request (DNS lookup, connect, TLS handshake, ส่ง request แรก) ด้วย callback อื่นๆ ใน `httptrace.ClientTrace` เช่น `DNSStart`/`DNSDone`, `ConnectStart`/`ConnectDone`
6. เขียนฟังก์ชันที่ส่งไฟล์ (multipart form) ไปยัง server จำลองด้วย `httptest` โดยค้นคว้าเพิ่มเติมเรื่อง `multipart.Writer` จาก standard library (ใบ้: เกี่ยวข้องกับ `mime/multipart` ซึ่งเราจะเจอการใช้งานฝั่ง server ใน **Part 066**)

---

**ต่อไป**: [Part 047 — `net/http` — HTTP Server เจาะลึก](./047-http-server.md)
