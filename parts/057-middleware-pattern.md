# Part 057: Middleware Pattern

> ภาคที่ 5: Web Development — ตอนที่ 2 จาก 15 (Part 56–70)

## สารบัญของบทนี้

1. Middleware คืออะไร และทำไมต้องมี
2. Middleware ในฐานะ Decorator Pattern: `func(http.Handler) http.Handler`
3. Logging Middleware: บันทึกทุก request ที่เข้ามา
4. Recovery Middleware: กัน panic ไม่ให้ทั้งเซิร์ฟเวอร์ล่ม
5. Auth Middleware (โครงร่าง): ตรวจสอบสิทธิ์เบื้องต้น
6. รวม Middleware หลายตัวด้วย `Chain`
7. ลำดับของ Middleware มีผลจริง: พิสูจน์ด้วยโค้ด
8. ใช้ Middleware กับบาง route เทียบกับใช้ทั้งแอป (Global)
9. ตัวอย่างเต็มรูปแบบและผลการทดสอบ
10. แนวทางปฏิบัติที่ดีและข้อผิดพลาดที่พบบ่อย
11. สรุปสิ่งที่ได้เรียนในบทนี้
12. แบบฝึกหัดท้ายบท

---

## 1. Middleware คืออะไร และทำไมต้องมี

ลองนึกภาพ handler แต่ละตัวใน Notes API จาก **Part 056** — ทุก handler ต้องการสิ่งที่เหมือนกันซ้ำๆ: อยากบันทึก log ว่า request ไหนเข้ามาบ้าง, อยากป้องกันไม่ให้ panic ทำทั้งเซิร์ฟเวอร์ล่ม, อยากเช็คว่า user login แล้วหรือยัง ถ้าเขียนโค้ดเหล่านี้ซ้ำในทุก handler จะเกิดปัญหาสองอย่าง:

1. **โค้ดซ้ำซ้อน (duplication)** — แก้ logic การ log ทีต้องไปแก้ทุก handler
2. **Handler ทำหน้าที่เกินตัว** — handler ควรรับผิดชอบแค่ "ตอบ request นี้ยังไง" ไม่ใช่ "ต้อง log, ต้องกัน panic, ต้องเช็ค auth ด้วย"

**Middleware** คือคำตอบของปัญหานี้: มันคือโค้ดที่ **"ห่อ" (wrap)** อยู่รอบๆ handler จริง ทำงานก่อนและ/หรือหลัง handler จริงทำงาน โดยที่ handler จริงไม่ต้องรู้เลยว่ามันถูกห่ออยู่ นี่คือรูปแบบที่ใช้กันเป็นมาตรฐานในเกือบทุกเว็บเฟรมเวิร์กของทุกภาษา ไม่ใช่แค่ Go

---

## 2. Middleware ในฐานะ Decorator Pattern: `func(http.Handler) http.Handler`

**Part 048** พูดถึงแนวคิดการ compose `io.Reader`/`io.Writer` เข้าด้วยกัน เช่น เอา `io.Reader` ธรรมดามาห่อด้วย `gzip.Reader` เพื่อเพิ่มความสามารถ "แตกไฟล์ gzip" โดยไม่ต้องเขียน type ใหม่ทั้งหมด — นี่คือแก่นของ **decorator pattern**: รับ object ตัวหนึ่งมา คืน object ใหม่ที่มีพฤติกรรมเดิมบวกความสามารถเพิ่มเติม

Middleware ใน `net/http` ใช้แนวคิดเดียวกันเป๊ะ เพียงแต่ประยุกต์กับ `http.Handler` แทน `io.Reader`:

```go
type Middleware func(http.Handler) http.Handler
```

อ่าน type นี้ว่า: **"Middleware คือฟังก์ชันที่รับ Handler ตัวหนึ่งมา แล้วคืน Handler ตัวใหม่ออกไป"** ตัว Handler ใหม่ที่คืนออกไปจะทำงานอะไรก็ได้ก่อน/หลังเรียก handler เดิมที่รับเข้ามา (มักเรียกว่า `next`)

รูปแบบเขียนโค้ด middleware ทุกตัวจะมีโครงเดียวกันเสมอ:

```go
func MyMiddleware(next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		// (1) ทำอะไรบางอย่าง "ก่อน" เรียก handler จริง
		next.ServeHTTP(w, r) // (2) ส่งต่อให้ handler ชั้นในทำงาน
		// (3) ทำอะไรบางอย่าง "หลัง" handler จริงทำงานเสร็จ
	})
}
```

ข้อสังเกตสำคัญ: เพราะ `Middleware` รับและคืน `http.Handler` เหมือนกัน จึงเอา middleware หลายตัวมา **ต่อกันเป็นชั้นๆ (chain)** ได้ไม่จำกัด เหมือนหัวหอมที่ปอกทีละชั้น — เดี๋ยวหัวข้อ 6 จะเขียน helper สำหรับต่อชั้นเหล่านี้ให้อัตโนมัติ

---

## 3. Logging Middleware: บันทึกทุก request ที่เข้ามา

Middleware ตัวแรกที่แทบทุกโปรเจกต์ต้องมี: log ว่า request ไหนเข้ามา ตอบด้วย status อะไร ใช้เวลาเท่าไหร่ ปัญหาคือ `http.ResponseWriter` ไม่มีทางอ่านค่า status code ที่ตัวเองเพิ่งเขียนไปย้อนหลังได้ (มันออกแบบมาให้ "เขียนไปข้างหน้า" อย่างเดียว) จึงต้องสร้าง wrapper ที่ดัก `WriteHeader` ไว้:

```go
// responseRecorder ดักจับ status code ที่ handler ชั้นในเขียนออกมา เพื่อเอาไป log
type responseRecorder struct {
	http.ResponseWriter
	status int
}

func (r *responseRecorder) WriteHeader(code int) {
	r.status = code
	r.ResponseWriter.WriteHeader(code)
}

func Logging(next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		start := time.Now()
		rec := &responseRecorder{ResponseWriter: w, status: http.StatusOK}
		next.ServeHTTP(rec, r)
		log.Printf("%s %s -> %d (%s)", r.Method, r.URL.Path, rec.status, time.Since(start))
	})
}
```

เทคนิคที่ใช้ตรงนี้คือ **struct embedding** (ทบทวนได้จาก **Part 030**): `responseRecorder` embed `http.ResponseWriter` เข้าไป ทำให้มันมี method `Header()` และ `Write()` ครบตาม interface โดยอัตโนมัติ (ผ่าน embedding) เราแค่ override เฉพาะ `WriteHeader` ตัวเดียวเพื่อแอบดูค่า status ที่ผ่านมา

สังเกตว่าถ้า handler ไม่เคยเรียก `WriteHeader` เลย (เช่น เรียก `w.Write()` ตรงๆ) Go จะถือว่า status เป็น `200 OK` โดยอัตโนมัติ — เราจึงตั้งค่า default `status: http.StatusOK` ไว้ตั้งแต่แรกเพื่อให้ log ถูกต้องในกรณีนั้นด้วย

---

## 4. Recovery Middleware: กัน panic ไม่ให้ทั้งเซิร์ฟเวอร์ล่ม

**Part 017** อธิบายไว้ว่า panic ที่ไม่ถูก recover จะทำให้ **ทั้ง goroutine** ที่มันเกิดขึ้นต้อง crash และถ้า goroutine นั้นคือ goroutine หลักของโปรแกรม โปรแกรมทั้งตัวจะล่มไปด้วย ข่าวดีคือ `net/http` รัน handler ของแต่ละ request บน goroutine แยกกัน ดังนั้น panic ใน handler เดียวจะไม่ทำให้ request อื่นล่มตามไปด้วยเสมอ **แต่** ถ้าไม่ recover เอง goroutine นั้นจะ crash และ client ฝั่งนั้นจะได้ connection ที่ถูกตัดกลางคัน (ไม่ได้ response ที่มีความหมายอะไรเลย)

Recovery middleware แก้ปัญหานี้โดยวาง `defer` + `recover()` ไว้ที่ **"ขอบเขตของ API" (API boundary)** ตามหลักการที่ Part 017 สอนไว้ในหัวข้อที่ 5 — recover panic ตรงนี้เป็นจุดเดียวที่เหมาะสมที่สุด เพราะทุก request ผ่านจุดนี้แน่นอน:

```go
func Recovery(next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		defer func() {
			if err := recover(); err != nil {
				log.Printf("panic recovered: %v", err)
				w.Header().Set("Content-Type", "application/json")
				w.WriteHeader(http.StatusInternalServerError)
				w.Write([]byte(`{"error":"internal server error"}`))
			}
		}()
		next.ServeHTTP(w, r)
	})
}
```

ข้อควรระวังที่สำคัญมาก: ถ้า handler ชั้นในเขียน header หรือ body ออกไปแล้วบางส่วนก่อนเกิด panic (เช่นเขียน `w.WriteHeader(200)` ไปแล้วค่อย panic) การเรียก `w.WriteHeader(500)` ใน recovery จะไม่มีผลอะไร (Go จะ log warning ว่า `superfluous response.WriteHeader call`) เพราะ HTTP header ส่งออกไปแล้วครั้งเดียวเปลี่ยนไม่ได้ — นี่คือเหตุผลที่ **Recovery middleware ควรอยู่เป็นชั้นนอกสุดหรือเกือบนอกสุดเสมอ** (ดูหัวข้อ 7 เรื่องลำดับ)

---

## 5. Auth Middleware (โครงร่าง): ตรวจสอบสิทธิ์เบื้องต้น

การตรวจสอบ JWT แบบเต็มรูปแบบ (verify signature, ตรวจ expiry, อ่าน claims) จะเรียนอย่างละเอียดใน **Part 067: Authentication ด้วย JWT** บทนี้จะโชว์แค่ **โครงร่าง (skeleton)** ของ auth middleware เพื่อให้เห็นรูปแบบการทำงานร่วมกับ `context` (ทบทวนได้จาก **Part 032**):

```go
type ctxKey string

const userCtxKey ctxKey = "user"

// Auth เป็นแค่โครงร่างง่ายๆ: เช็คว่ามี header Authorization: Bearer <token>
// ไหม แล้วฝัง "user" ปลอมลงใน context เพื่อให้ handler ชั้นในดึงไปใช้ได้
func Auth(next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		header := r.Header.Get("Authorization")
		if !strings.HasPrefix(header, "Bearer ") {
			w.Header().Set("Content-Type", "application/json")
			w.WriteHeader(http.StatusUnauthorized)
			w.Write([]byte(`{"error":"missing or invalid authorization header"}`))
			return
		}
		token := strings.TrimPrefix(header, "Bearer ")
		// ในระบบจริงตรงนี้จะ verify signature และ decode claims ของ JWT
		if token == "" {
			w.WriteHeader(http.StatusUnauthorized)
			return
		}
		ctx := context.WithValue(r.Context(), userCtxKey, "user-"+token)
		next.ServeHTTP(w, r.WithContext(ctx))
	})
}
```

จุดสำคัญที่ต้องสังเกต:

- ถ้าตรวจสอบไม่ผ่าน middleware จะ **ไม่เรียก `next.ServeHTTP`** เลย — นี่คือกลไกที่ทำให้ middleware "ตัดวงจร" ได้ เมื่อไม่เรียก `next` handler ชั้นในจะไม่ทำงานเลย
- ถ้าผ่าน จะฝังข้อมูล user ลงใน `context` ด้วย `context.WithValue` แล้วส่งต่อด้วย `r.WithContext(ctx)` (ไม่ใช่ `r` ตัวเดิม!) เพื่อให้ handler ชั้นในดึงข้อมูลนี้กลับมาใช้ได้ผ่าน `r.Context().Value(userCtxKey)`
- การใช้ `ctxKey` เป็น type เฉพาะ (ไม่ใช่ `string` ตรงๆ) เป็น best practice ที่ `context` package แนะนำ เพื่อป้องกัน key ชนกันข้าม package (ทบทวนรายละเอียดได้จาก Part 032)

---

## 6. รวม Middleware หลายตัวด้วย `Chain`

ถ้าต้องเขียน `Logging(Recovery(Auth(handler)))` ทุกครั้งที่มี route ใหม่ จะดูรกและอ่านลำดับยาก แนวทางที่นิยมคือเขียน helper `Chain` ที่รับ handler ตั้งต้นและ middleware เป็น slice แล้วห่อให้อัตโนมัติ:

```go
// Chain รวม middleware หลายตัวเข้าด้วยกัน โดยตัวแรกสุดในรายการจะเป็นชั้นนอกสุด
// (ทำงานก่อนสุด และเป็นคนสุดท้ายที่เห็น response กลับออกมา)
func Chain(h http.Handler, mws ...Middleware) http.Handler {
	// ห่อจากขวาไปซ้าย เพื่อให้ mws[0] กลายเป็นชั้นนอกสุด
	for i := len(mws) - 1; i >= 0; i-- {
		h = mws[i](h)
	}
	return h
}
```

ใช้งานแบบนี้:

```go
protected := Chain(http.HandlerFunc(profileHandler), Auth)
mux.Handle("GET /profile", protected)
```

อ่านง่ายขึ้นมาก — เห็นทันทีว่า `profileHandler` ถูกห่อด้วย `Auth` ชั้นเดียว โดยไม่ต้องนับวงเล็บซ้อนกันเอง หลายเฟรมเวิร์ก (รวมถึง `chi` ที่จะเรียนใน Part 058) มี helper แบบนี้ในตัวชื่อ `Use()` ซึ่งทำงานบนหลักการเดียวกันเป๊ะ

---

## 7. ลำดับของ Middleware มีผลจริง: พิสูจน์ด้วยโค้ด

เพราะ middleware ห่อกันเป็นชั้นๆ **ลำดับที่ส่งเข้า `Chain` มีผลโดยตรง**ต่อพฤติกรรมจริง ลองพิสูจน์ด้วยโค้ดง่ายๆ ที่พิมพ์ข้อความก่อน/หลังเรียก handler ชั้นใน:

```go
func A(next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		fmt.Println("A: before")
		next.ServeHTTP(w, r)
		fmt.Println("A: after")
	})
}

func B(next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		fmt.Println("B: before")
		next.ServeHTTP(w, r)
		fmt.Println("B: after")
	})
}

func final(w http.ResponseWriter, r *http.Request) {
	fmt.Println("handler")
}
```

รันด้วย `Chain(http.HandlerFunc(final), A, B)` เทียบกับ `Chain(http.HandlerFunc(final), B, A)` ได้ผลลัพธ์จริงดังนี้:

```
--- Chain(handler, A, B) ---
A: before
B: before
handler
B: after
A: after

--- Chain(handler, B, A) ---
B: before
A: before
handler
A: after
B: after
```

สังเกตรูปแบบ **"เข้าตามลำดับ ออกย้อนกลับ" (LIFO — last in, first out)**: middleware ตัวแรกสุดในรายการ (`A` ในกรณีแรก) เป็นชั้นนอกสุด มันจึง "before" ก่อนใคร และ "after" หลังใครสุด — เหมือนการเข้าคิวปอกหัวหอม ชั้นนอกสุดคือชั้นที่ถูกปอก (เห็น) เป็นอันดับแรกและเป็นชั้นสุดท้ายที่ประกอบกลับเข้าไป

ทำไมเรื่องนี้ถึงสำคัญกับตัวอย่างในหัวข้อ 3-5:

- **Recovery ควรอยู่นอกสุด (หรือเกือบนอกสุด)** เพราะถ้า panic เกิดขึ้นใน middleware ชั้นในกว่า (เช่นใน Auth หรือใน handler เอง) Recovery ต้องอยู่ในตำแหน่งที่ "ครอบ" มันได้ทัน ถ้า Recovery อยู่ชั้นในสุด panic จาก middleware ชั้นนอกกว่าจะหลุดรอดไปได้
- **Logging มักอยู่นอกสุดหรือเกือบนอกสุดเช่นกัน** เพื่อให้ log ครอบคลุมทุก request แม้แต่ request ที่ถูก Auth ปฏิเสธไปตั้งแต่ต้น (ถ้า Logging อยู่ชั้นในกว่า Auth request ที่ถูกปฏิเสธจะไม่ถูก log เลย)
- **Auth มักอยู่ชั้นในสุด (ใกล้ handler จริงที่สุด)** เพราะมันควรทำงานเฉพาะกับ route ที่ต้องการจริงๆ ไม่ใช่ทุก route

ลำดับที่แนะนำโดยทั่วไป: **Recovery → Logging → Auth (ถ้ามี) → Handler จริง**

---

## 8. ใช้ Middleware กับบาง route เทียบกับใช้ทั้งแอป (Global)

Middleware ไม่จำเป็นต้องใช้กับทุก route เหมือนกันหมด — บางตัว (เช่น Recovery, Logging) ควรใช้แบบ **global** (ครอบทั้ง mux) เพราะทุก route ต้องการมันเหมือนกัน ในขณะที่บางตัว (เช่น Auth) ควรใช้แบบ **เฉพาะบาง route** เท่านั้น

```go
mux := http.NewServeMux()
mux.HandleFunc("GET /hello", helloHandler)
mux.HandleFunc("GET /boom", boomHandler)

// เฉพาะเส้นทาง /profile เท่านั้นที่ต้องผ่าน Auth ก่อน — ห่อเฉพาะ handler
// นี้ตัวเดียว ไม่กระทบเส้นทางอื่นอย่าง /hello เลย
protected := Chain(http.HandlerFunc(profileHandler), Auth)
mux.Handle("GET /profile", protected)

// ห่อทั้ง mux (ทุกเส้นทาง รวมถึง /profile ที่ห่อ Auth ไว้ชั้นในแล้ว)
// ด้วย Logging + Recovery แบบ global — ทุก request จะถูก log และป้องกัน
// panic เหมือนกันหมด ไม่ว่าจะเป็นเส้นทางไหน
globalHandler := Chain(mux, Logging, Recovery)
```

รูปแบบนี้ทำให้ `/hello` และ `/boom` ไม่ต้องผ่าน `Auth` เลย ในขณะที่ `/profile` ต้องผ่านทั้ง `Auth` (ชั้นใน) และ `Logging`+`Recovery` (ชั้นนอกสุด) — เป็นแนวทางที่ยืดหยุ่นและตรงกับความต้องการจริงของแต่ละ route

---

## 9. ตัวอย่างเต็มรูปแบบและผลการทดสอบ

รวมทุกอย่างเป็นโปรแกรมเดียว:

```go
package main

import (
	"context"
	"log"
	"net/http"
	"strings"
	"time"
)

type Middleware func(http.Handler) http.Handler

type responseRecorder struct {
	http.ResponseWriter
	status int
}

func (r *responseRecorder) WriteHeader(code int) {
	r.status = code
	r.ResponseWriter.WriteHeader(code)
}

func Logging(next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		start := time.Now()
		rec := &responseRecorder{ResponseWriter: w, status: http.StatusOK}
		next.ServeHTTP(rec, r)
		log.Printf("%s %s -> %d (%s)", r.Method, r.URL.Path, rec.status, time.Since(start))
	})
}

func Recovery(next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		defer func() {
			if err := recover(); err != nil {
				log.Printf("panic recovered: %v", err)
				w.Header().Set("Content-Type", "application/json")
				w.WriteHeader(http.StatusInternalServerError)
				w.Write([]byte(`{"error":"internal server error"}`))
			}
		}()
		next.ServeHTTP(w, r)
	})
}

type ctxKey string

const userCtxKey ctxKey = "user"

func Auth(next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		header := r.Header.Get("Authorization")
		if !strings.HasPrefix(header, "Bearer ") {
			w.Header().Set("Content-Type", "application/json")
			w.WriteHeader(http.StatusUnauthorized)
			w.Write([]byte(`{"error":"missing or invalid authorization header"}`))
			return
		}
		token := strings.TrimPrefix(header, "Bearer ")
		if token == "" {
			w.WriteHeader(http.StatusUnauthorized)
			return
		}
		ctx := context.WithValue(r.Context(), userCtxKey, "user-"+token)
		next.ServeHTTP(w, r.WithContext(ctx))
	})
}

func Chain(h http.Handler, mws ...Middleware) http.Handler {
	for i := len(mws) - 1; i >= 0; i-- {
		h = mws[i](h)
	}
	return h
}

func helloHandler(w http.ResponseWriter, r *http.Request) {
	w.Write([]byte("hello, public\n"))
}

func profileHandler(w http.ResponseWriter, r *http.Request) {
	user, _ := r.Context().Value(userCtxKey).(string)
	w.Write([]byte("profile of " + user + "\n"))
}

func boomHandler(w http.ResponseWriter, r *http.Request) {
	var items []int
	_ = items[5] // จงใจทำให้ panic: index out of range
}

func main() {
	mux := http.NewServeMux()
	mux.HandleFunc("GET /hello", helloHandler)
	mux.HandleFunc("GET /boom", boomHandler)

	protected := Chain(http.HandlerFunc(profileHandler), Auth)
	mux.Handle("GET /profile", protected)

	globalHandler := Chain(mux, Logging, Recovery)

	log.Println("listening on :8090")
	log.Fatal(http.ListenAndServe(":8090", globalHandler))
}
```

ทดสอบด้วย `curl` (ผลลัพธ์จริงจากการรันโค้ดข้างต้น):

```bash
curl -s http://localhost:8090/hello
```
```
hello, public
```

```bash
curl -s -o /dev/null -w "%{http_code}\n" http://localhost:8090/profile
```
```
401
```

```bash
curl -s -H "Authorization: Bearer abc123" http://localhost:8090/profile
```
```
profile of user-abc123
```

```bash
curl -s -o /dev/null -w "%{http_code}\n" http://localhost:8090/boom
```
```
500
```

```bash
# server ยังทำงานปกติหลังเกิด panic — ไม่ crash ทั้งโปรเซส
curl -s http://localhost:8090/hello
```
```
hello, public
```

log ฝั่ง server ระหว่างทดสอบทั้งหมด:

```
listening on :8090
GET /hello -> 200 (11.251µs)
GET /profile -> 401 (21.177µs)
GET /profile -> 200 (13.694µs)
panic recovered: runtime error: index out of range [5] with length 0
GET /boom -> 500 (138.708µs)
GET /hello -> 200 (7.994µs)
```

สังเกตว่า request ที่ถูก `Auth` ปฏิเสธ (401) และ request ที่ panic (500) ก็ยังถูก `Logging` บันทึกอย่างถูกต้อง เพราะ `Logging` อยู่ชั้นนอกสุด — นี่คือผลโดยตรงจากลำดับ middleware ที่อธิบายไว้ในหัวข้อ 7 และเซิร์ฟเวอร์ยังรับ request ถัดไป (`/hello` ครั้งที่สอง) ได้ตามปกติ พิสูจน์ว่า panic ใน `boomHandler` ไม่ได้ทำทั้งโปรเซสล่ม เพราะ `Recovery` ดักไว้ได้ทัน

---

## 10. แนวทางปฏิบัติที่ดีและข้อผิดพลาดที่พบบ่อย

- **อย่าลืมเรียก `next.ServeHTTP`** — ถ้าลืม (บั๊กที่พบบ่อยตอนเขียน middleware ใหม่ๆ) request จะค้างไม่มี response ตอบกลับเลย เพราะ handler ชั้นในไม่เคยถูกเรียก
- **Recovery middleware ควรอยู่ชั้นนอกสุดเสมอ** เพื่อให้ครอบคลุม panic จากทุก middleware ชั้นในทั้งหมด รวมถึง handler จริง
- **อย่าใช้ `context.WithValue` พร่ำเพรื่อ** — ใช้เฉพาะข้อมูลที่เกี่ยวกับ request/response scope จริงๆ เช่น user ที่ login, request ID ไม่ใช่เอาไว้ส่ง dependency ทั่วไปแทน constructor (Part 032 อธิบายเหตุผลไว้ละเอียด)
- **ระวังการเขียน header ซ้ำ** — ถ้า middleware ชั้นในเขียน `WriteHeader` ไปแล้ว middleware ชั้นนอกเขียนซ้ำอีกจะเจอ warning `superfluous response.WriteHeader call` เสมอเช็คให้ดีว่าใครควรเป็นคนตัดสินใจ status code สุดท้าย
- **Middleware ที่ต้องแก้ไข response (เช่นแปลง 404 เป็น JSON ใน Part 056) ต้องดักด้วย response wrapper เท่านั้น** — เขียนโดยตรงไม่ได้เพราะ `net/http` ไม่มีทาง "ยกเลิก" การเขียนที่เกิดไปแล้ว

### เทียบกับแนวทางในไลบรารีที่ได้รับความนิยม

รูปแบบ `func(http.Handler) http.Handler` ที่เขียนเองในบทนี้ไม่ใช่สิ่งประดิษฐ์เฉพาะของบทเรียนนี้ — มันคือรูปแบบมาตรฐานที่ทั้ง standard library เอง (ผ่าน `net/http`) และไลบรารีชื่อดังอย่าง `justinas/alice`, `gorilla/handlers` และ `chi/middleware` (ที่จะเรียนใน **Part 058**) ใช้เหมือนกันหมด ต่างกันแค่ syntax ของ helper สำหรับต่อ chain เท่านั้น:

| ไลบรารี | Middleware type | Helper ต่อ chain |
|---|---|---|
| เขียนเอง (บทนี้) | `func(http.Handler) http.Handler` | `Chain(h, mws...)` |
| `justinas/alice` | `func(http.Handler) http.Handler` | `alice.New(mws...).Then(h)` |
| `chi/middleware` | `func(http.Handler) http.Handler` | `r.Use(mws...)` |
| `gorilla/mux` | `func(http.Handler) http.Handler` | `router.Use(mws...)` |

เพราะ type ตรงกันพอดี middleware ที่เขียนเองในบทนี้ (`Logging`, `Recovery`, `Auth`) จึงเอาไปใช้กับ `chi` หรือ `gorilla/mux` ได้ทันทีโดยไม่ต้องแก้โค้ดเลยแม้แต่บรรทัดเดียว — นี่คือพลังของการที่ Go ออกแบบ `http.Handler` เป็น interface เดียวที่เรียบง่ายมาตั้งแต่ต้น (ทบทวนปรัชญา "less is more" ได้จาก **Part 001**)

---

## สรุปสิ่งที่ได้เรียนในบทนี้

- Middleware คือฟังก์ชัน `func(http.Handler) http.Handler` ที่ห่อ handler เดิมไว้ เป็นการประยุกต์ decorator pattern (แนวคิดเดียวกับ Part 048) เข้ากับ HTTP
- Logging middleware ต้องดักจับ status code ผ่าน wrapper ของ `http.ResponseWriter` เพราะอ่านค่าย้อนหลังไม่ได้
- Recovery middleware ใช้ `defer` + `recover()` ที่ API boundary ตามหลักการจาก Part 017 เพื่อป้องกัน panic ทำทั้งเซิร์ฟเวอร์ล่ม
- Auth middleware โครงร่างใช้ `context.WithValue` ฝังข้อมูล user ให้ handler ชั้นในดึงไปใช้ (รายละเอียด JWT เต็มรูปแบบอยู่ใน Part 067)
- `Chain(h, mws...)` ช่วยรวม middleware หลายตัวให้อ่านง่ายกว่าการซ้อนวงเล็บเอง
- ลำดับ middleware มีผลจริง: เข้าตามลำดับ ออกย้อนกลับ (LIFO) — Recovery และ Logging ควรอยู่ชั้นนอกสุด ส่วน Auth มักอยู่ชั้นในใกล้ handler จริง
- middleware บางตัวใช้แบบ global (ครอบทั้ง mux) บางตัวใช้เฉพาะบาง route ได้ตามความเหมาะสม

## แบบฝึกหัดท้ายบท

1. เขียน middleware ชื่อ `RequestID` ที่สร้างค่าสุ่มไม่ซ้ำกัน (ใช้ `crypto/rand` หรือ counter ง่ายๆ) ใส่ลงใน `context` แล้วให้ `Logging` middleware ดึงค่านี้มาพิมพ์ต่อท้าย log ด้วย
2. ทดลองสลับลำดับให้ `Auth` อยู่ชั้นนอกกว่า `Logging` (เช่น `Chain(handler, Auth, Logging)`) แล้วสังเกตว่า request ที่ถูกปฏิเสธ 401 จะถูก log หรือไม่ อธิบายว่าทำไม
3. เพิ่ม middleware ชื่อ `Timeout` ที่ใช้ `context.WithTimeout` (ทบทวนจาก Part 043) ห่อ request แล้วถ้า handler ทำงานเกิน 2 วินาที ให้ตอบ `503 Service Unavailable` แทน
4. เขียนเทสสำหรับ `Recovery` middleware โดยใช้ `httptest.NewRecorder()` และ handler ที่จงใจ panic แล้วตรวจสอบว่า response ที่ได้เป็น 500 JSON ตามที่ออกแบบไว้จริง
5. ปรับ `Chain` ให้เป็น method `Use(mws ...Middleware)` ของ struct `Router` ที่หุ้ม `*http.ServeMux` ไว้ข้างใน แล้วให้ route ที่ลงทะเบียนหลัง `Use` ทุกตัวถูกห่อด้วย middleware ที่สะสมไว้อัตโนมัติ
6. อธิบาย (เตรียมคำตอบไว้) ว่าทำไม middleware แบบ auth ไม่ควรอยู่เป็นชั้นนอกสุดกว่า logging ในระบบที่ต้องการ audit log ครบทุก request แม้แต่ request ที่ไม่ผ่านการยืนยันตัวตน

---

**ต่อไป**: [Part 058 — Router ยอดนิยม: `gorilla/mux`, `chi`](./058-routers-mux-chi.md)
