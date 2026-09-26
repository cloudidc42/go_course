# Part 056: HTTP Server ด้วย `net/http`: Routing, Handler

> ภาคที่ 5: Web Development — ตอนที่ 1 จาก 15 (Part 56–70)

## สารบัญของบทนี้

1. ทบทวนสิ่งที่ต้องรู้มาก่อนจาก Part 047
2. ปัญหาของการ Routing แบบเดิมก่อน Go 1.22
3. ServeMux ยุคใหม่ (Go 1.22+): Pattern แบบ Method + Path + Wildcard
4. ดึงค่าจาก Path ด้วย `r.PathValue`
5. ออกแบบ Handler แบบจัดกลุ่มตาม Resource
6. Handler แบบ Struct-based ที่ผูกกับ Dependency (Store/Service)
7. จัดการ 404 Not Found ให้ตอบกลับในรูปแบบที่ต้องการ
8. จัดการ 405 Method Not Allowed ให้ตอบกลับในรูปแบบที่ต้องการ
9. ตัวอย่างเต็มรูปแบบ: Notes API (In-Memory)
10. ทดสอบ API ด้วย `curl` ทีละ endpoint
11. ข้อควรระวังและแนวทางปฏิบัติที่ดี
12. สรุปสิ่งที่ได้เรียนในบทนี้
13. แบบฝึกหัดท้ายบท

---

## 1. ทบทวนสิ่งที่ต้องรู้มาก่อนจาก Part 047

**Part 047** ได้วางรากฐานของการสร้าง HTTP server ด้วย `net/http` ไว้แล้ว ทั้งเรื่อง:

- **`http.Handler` interface** — อะไรก็ตามที่มี method `ServeHTTP(w http.ResponseWriter, r *http.Request)` ถือว่าเป็น Handler ได้
- **`http.HandlerFunc`** — adapter ที่แปลง function ธรรมดาให้กลายเป็น `http.Handler`
- **`http.ServeMux`** — ตัว router พื้นฐานที่มากับ standard library เอง
- **Graceful shutdown** — การปิด server อย่างสุภาพด้วย `srv.Shutdown(ctx)` แทนการตัดการเชื่อมต่อทันที

บทนี้จะ**ไม่สอนเรื่องเหล่านี้ซ้ำ** แต่จะต่อยอดจากรากฐานนั้น เพื่อสร้างแอปพลิเคชันจริงที่มีหลาย resource, หลาย route และต้องจัดการ HTTP method, error case ต่างๆ อย่างเป็นระบบ — พูดง่ายๆ คือ Part 047 สอน "ประกอบ server ยังไง" ส่วนบทนี้สอน "จัดระเบียบ route และ handler ของ API จริงยังไง"

ถ้ายังไม่คุ้นกับ `http.Handler`, `http.HandlerFunc`, หรือวิธีเปิด/ปิด server อย่างสุภาพ แนะนำให้กลับไปอ่าน Part 047 ก่อน

---

## 2. ปัญหาของการ Routing แบบเดิมก่อน Go 1.22

ก่อน Go 1.22 (ออกกุมภาพันธ์ 2024) `http.ServeMux` มีข้อจำกัดสำคัญที่ทำให้นักพัฒนา Go จำนวนมากหันไปพึ่ง third-party router (ซึ่งจะเรียนใน **Part 058**) ข้อจำกัดหลักคือ:

1. **Match ได้แค่ path เท่านั้น ไม่สนใจ HTTP method** — ถ้าลงทะเบียน `mux.HandleFunc("/notes", handler)` ทั้ง `GET /notes` และ `POST /notes` และ `DELETE /notes` จะไปเข้า handler เดียวกันหมด ต้องเช็ค `r.Method` เองข้างในทุกครั้ง
2. **ไม่มี wildcard/path parameter ในตัว** — ต้องการ endpoint แบบ `/notes/123` ต้อง parse string เอาเลข id ออกมาเองด้วย `strings.TrimPrefix` หรือ `regexp` ซึ่งเขียนซ้ำๆ ทุก handler
3. **ไม่มี automatic 405** — เมื่อ path ตรงแต่ method ไม่ถูกต้อง ก็ไม่มีกลไกตอบ `405 Method Not Allowed` ให้อัตโนมัติ ต้องเขียนเองทุกจุด

โค้ดแบบเก่าก่อน Go 1.22 หน้าตาประมาณนี้ (ตัวอย่างเพื่อให้เห็นปัญหา ไม่ใช่โค้ดที่แนะนำให้ใช้ต่อไป):

```go
mux.HandleFunc("/notes/", func(w http.ResponseWriter, r *http.Request) {
	idStr := strings.TrimPrefix(r.URL.Path, "/notes/")
	id, err := strconv.Atoi(idStr)
	if err != nil {
		http.Error(w, "invalid id", http.StatusBadRequest)
		return
	}
	switch r.Method {
	case http.MethodGet:
		// ... GET logic
	case http.MethodDelete:
		// ... DELETE logic
	default:
		http.Error(w, "method not allowed", http.StatusMethodNotAllowed)
	}
})
```

จะเห็นว่า logic การแยก method และ parse path ปนอยู่กับ business logic ทำให้อ่านยากขึ้นเรื่อยๆ เมื่อ endpoint เยอะขึ้น นี่คือเหตุผลที่ **gorilla/mux** และ **chi** (Part 058) ได้รับความนิยมมากในอดีต — Go ทีมงานเองก็รับรู้ปัญหานี้ จึงปรับปรุง `ServeMux` ครั้งใหญ่ใน Go 1.22

---

## 3. ServeMux ยุคใหม่ (Go 1.22+): Pattern แบบ Method + Path + Wildcard

ตั้งแต่ **Go 1.22** เป็นต้นไป `http.ServeMux` รองรับ pattern ที่ทรงพลังขึ้นมาก โดยไม่ต้องพึ่ง library ภายนอกเลย ไวยากรณ์ pattern มีรูปแบบ:

```
[METHOD ][HOST]/path/{wildcard}
```

ตัวอย่างการลงทะเบียน route ที่ระบุ method ตรงๆ:

```go
mux := http.NewServeMux()
mux.HandleFunc("GET /notes", listNotes)
mux.HandleFunc("POST /notes", createNote)
mux.HandleFunc("GET /notes/{id}", getNote)
mux.HandleFunc("DELETE /notes/{id}", deleteNote)
```

สังเกตสิ่งที่เปลี่ยนไปจากเดิม:

- **ระบุ HTTP method นำหน้า path ได้เลย** (`"GET /notes"`) — mux จะ match เฉพาะ request ที่ทั้ง method และ path ตรงกันเท่านั้น
- **`{id}` คือ wildcard segment** — จับคู่กับ path segment ใดก็ได้หนึ่งช่วง (ไม่รวม `/`) แล้วให้เราดึงค่าออกมาทีหลังด้วย `r.PathValue("id")`
- **ไม่ระบุ method** (เช่น `"/notes/"`) ยังใช้ได้เหมือนเดิม จะ match ทุก method

### กฎการเลือก pattern ที่ specific ที่สุด

ถ้ามีหลาย pattern ที่ match กับ request เดียวกัน `ServeMux` จะเลือก pattern ที่ **specific ที่สุด** เสมอ เช่น ถ้าลงทะเบียนทั้ง `"GET /notes/{id}"` และ `"GET /notes/latest"` ไว้ request `GET /notes/latest` จะจับคู่กับ pattern หลัง (ที่ literal ตรงกว่า) ไม่ใช่ pattern ที่มี wildcard

### wildcard พิเศษ: `{$}` และ `{path...}`

- `"GET /notes/{$}"` — จับคู่เฉพาะ path ที่ลงท้ายด้วย `/notes/` พอดี (ไม่ match `/notes/extra`)
- `"GET /files/{path...}"` — wildcard แบบ "กินรวด" จับคู่ทุกอย่างที่เหลือของ path รวม `/` ด้วย เหมาะกับการเสิร์ฟไฟล์แบบ nested path (จะกลับมาเรียกละเอียดใน **Part 066: Static Files และ File Upload**)

บทนี้จะเน้นที่ wildcard แบบ segment เดียว (`{id}`) เป็นหลัก เพราะเป็นรูปแบบที่ใช้บ่อยที่สุดในการทำ REST API

---

## 4. ดึงค่าจาก Path ด้วย `r.PathValue`

เมื่อประกาศ pattern ที่มี wildcard เช่น `{id}` แล้ว เราดึงค่าที่ match ออกมาได้ผ่าน method `r.PathValue(name string) string` ของ `*http.Request`:

```go
func getNote(w http.ResponseWriter, r *http.Request) {
	idStr := r.PathValue("id") // ดึงค่าจาก segment {id}
	id, err := strconv.Atoi(idStr)
	if err != nil {
		http.Error(w, "invalid id", http.StatusBadRequest)
		return
	}
	// ... ใช้ id ต่อ
}
```

ข้อสังเกตสำคัญ: `r.PathValue` คืนค่าเป็น `string` เสมอ (เพราะ URL path เป็น string โดยธรรมชาติ) ถ้าต้องการเป็นตัวเลขต้องแปลงเองด้วย `strconv` (ทบทวนได้ที่ **Part 020**) และต้องเช็ค error เสมอ เพราะ client อาจส่ง path แปลกๆ เช่น `/notes/abc` มาได้

> **ข้อควรระวัง**: `r.PathValue` ทำงานได้ก็ต่อเมื่อ request ถูก route ผ่าน `http.ServeMux` ตัวเดียวกับที่ประกาศ pattern ไว้เท่านั้น ถ้าคุณเขียน middleware ที่ดัก request ก่อนแล้วเรียก handler ภายในตรงๆ (ไม่ผ่าน `mux.ServeHTTP`) ค่าที่ได้จาก `PathValue` อาจว่างเปล่า เดี๋ยวหัวข้อที่ 7 จะเจอประเด็นนี้จริงเวลาทำ custom 404/405

---

## 5. ออกแบบ Handler แบบจัดกลุ่มตาม Resource

เมื่อ API มีหลาย resource (notes, users, tags, ...) การเขียนทุก handler เป็น function ลอยๆ ในไฟล์เดียวจะเริ่มอ่านยากอย่างรวดเร็ว แนวทางที่นิยมคือ **จัดกลุ่ม handler ของแต่ละ resource ไว้ด้วยกัน** โดยตั้งชื่อ pattern ให้สอดคล้องกับ resource นั้นๆ:

```go
func registerNoteRoutes(mux *http.ServeMux, h *NoteHandler) {
	mux.HandleFunc("GET /notes", h.list)
	mux.HandleFunc("POST /notes", h.create)
	mux.HandleFunc("GET /notes/{id}", h.get)
	mux.HandleFunc("DELETE /notes/{id}", h.delete)
}

func registerTagRoutes(mux *http.ServeMux, h *TagHandler) {
	mux.HandleFunc("GET /tags", h.list)
	mux.HandleFunc("POST /tags", h.create)
}

func main() {
	mux := http.NewServeMux()
	registerNoteRoutes(mux, noteHandler)
	registerTagRoutes(mux, tagHandler)
	// ...
}
```

รูปแบบนี้ทำให้:

- อ่าน `main` แล้วเห็นภาพรวมของ route ทั้งหมดในที่เดียว
- เพิ่ม resource ใหม่โดยไม่ต้องแตะโค้ดของ resource เดิม
- ทดสอบแต่ละกลุ่ม route แยกกันได้ง่ายขึ้น (จะเรียนเรื่อง `httptest` เจาะลึกใน **Part 081**)

---

## 6. Handler แบบ Struct-based ที่ผูกกับ Dependency (Store/Service)

Handler ในโลกจริงแทบไม่เคย standalone — มันต้องคุยกับฐานข้อมูล, cache, หรืออย่างน้อยที่สุดก็ต้องมี "ที่เก็บข้อมูล" บางอย่าง แทนที่จะใช้ package-level variable (global state ซึ่งทดสอบยากและแก้ race condition ยาก ตามที่เรียนใน **Part 039/044**) แนวทางที่ดีกว่าคือผูก dependency เข้ากับ struct แล้วให้ method ของ struct นั้นเป็น handler:

```go
type NoteHandler struct {
	store *NoteStore // dependency ถูกฉีดเข้ามาตอนสร้าง ไม่ใช่ global variable
}

func (h *NoteHandler) list(w http.ResponseWriter, r *http.Request) {
	notes := h.store.List()
	writeJSON(w, http.StatusOK, notes)
}
```

เพราะ method ที่มี receiver แบบ `func (h *NoteHandler) list(w http.ResponseWriter, r *http.Request)` มี signature ตรงกับ `func(http.ResponseWriter, *http.Request)` พอดี จึงส่งให้ `mux.HandleFunc` ได้โดยตรงโดยไม่ต้องแปลงอะไรเพิ่ม (ทบทวนเรื่อง method values ได้จาก **Part 012**)

ข้อดีของแนวทางนี้ชัดเจนที่สุดตอนเขียนเทส: สร้าง `NoteStore` ปลอมๆ (fake) แล้วส่งเข้า `NoteHandler{store: fakeStore}` ทดสอบ handler แยกจาก store จริงได้ทันที ไม่ต้องมี database จริงรันอยู่

---

## 7. จัดการ 404 Not Found ให้ตอบกลับในรูปแบบที่ต้องการ

`http.ServeMux` มี behavior default อยู่แล้วเมื่อไม่เจอ pattern ที่ match: ตอบ `404 Not Found` พร้อม body เป็น plain text ว่า `404 page not found` แต่ใน REST API เราอยากให้ **ทุก response เป็น JSON เหมือนกันหมด** รวมถึง error response ด้วย จึงต้องเขียน wrapper เพื่อแปลง 404 default ให้เป็น JSON

จุดที่ต้องระวังคือ: `mux.Handler(r)` (method ที่คืน handler และ pattern ที่ match) **ไม่ได้เซ็ตค่า path wildcard ให้ request** — การเซ็ตค่านั้นเกิดขึ้นเฉพาะตอนเรียกผ่าน `mux.ServeHTTP(w, r)` เท่านั้น ดังนั้นถ้าเขียน wrapper ผิดวิธีโดยเรียก handler ที่ได้จาก `mux.Handler` ตรงๆ จะทำให้ `r.PathValue("id")` คืนค่าว่างเปล่าทุกครั้ง (ผู้เขียนบทความนี้ก็เจอบั๊กนี้จริงระหว่างทดสอบตัวอย่างในบทนี้ — เป็นข้อผิดพลาดที่พบได้บ่อยมาก)

วิธีที่ถูกต้อง คือ ใช้ `mux.Handler(r)` แค่ **"แอบดู" ว่ามี pattern ตรงกันไหม** แล้วถ้ามี ให้เรียก `mux.ServeHTTP(w, r)` ใหม่อีกครั้งเพื่อให้ mux เซ็ต path value ให้ถูกต้อง:

```go
func withJSONErrors(mux *http.ServeMux) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		h, pattern := mux.Handler(r)
		if pattern != "" {
			// เจอ pattern ตรงกันจริง: เรียกผ่าน mux.ServeHTTP (ไม่ใช่ h.ServeHTTP
			// ตรงๆ) เพราะ mux ต้องเป็นคนเซ็ต r.PathValue ให้เองระหว่าง
			// ประมวลผล request นี้อีกครั้ง มิเช่นนั้น PathValue จะว่างเปล่า
			mux.ServeHTTP(w, r)
			return
		}
		// pattern == "" คือไม่เจอ route ที่ตรงกันเลย (หรือ method ไม่ตรง — ดูหัวข้อ 8)
		writeError(w, http.StatusNotFound, "resource not found")
	})
}
```

---

## 8. จัดการ 405 Method Not Allowed ให้ตอบกลับในรูปแบบที่ต้องการ

ข่าวดีคือ `ServeMux` ยุคใหม่ (Go 1.22+) มี**automatic 405 ในตัว**แล้ว ถ้าคุณลงทะเบียน `GET /notes/{id}` กับ `DELETE /notes/{id}` แต่ไม่มี pattern สำหรับ `PUT` request `PUT /notes/1` จะได้ response `405 Method Not Allowed` พร้อม header `Allow: DELETE, GET, HEAD` บอกด้วยว่า method ไหนใช้ได้บ้าง โดยไม่ต้องเขียนโค้ดเพิ่มเลย — เดี๋ยวหัวข้อ 10 จะแสดงผลจริงให้ดู

แต่ปัญหาคือ body ของ 405 default ก็เป็น plain text เหมือนกัน (`Method Not Allowed`) ไม่ใช่ JSON การจะแปลงให้เป็น JSON โดยยังรักษา automatic detection ของ 405 ไว้ ต้องอาศัยเทคนิคดัก status code ที่ ServeMux "ตั้งใจ" จะตอบ ก่อนที่มันจะถูกเขียนออกไปจริง:

```go
// statusInterceptor แอบดักสถานะ (status code) ที่ handler ภายในของ
// ServeMux ตั้งใจจะตอบกลับ โดยไม่ปล่อยให้ body เดิม (text ธรรมดา) ถูกเขียนจริง
type statusInterceptor struct {
	http.ResponseWriter
	status int
}

func (w *statusInterceptor) WriteHeader(code int) { w.status = code }
func (w *statusInterceptor) Write(b []byte) (int, error) { return len(b), nil }
```

แล้วรวมเข้ากับ `withJSONErrors` จากหัวข้อก่อนหน้า:

```go
func withJSONErrors(mux *http.ServeMux) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		h, pattern := mux.Handler(r)
		if pattern != "" {
			mux.ServeHTTP(w, r)
			return
		}
		// pattern == "" หมายถึง ServeMux จะตอบด้วย internal handler
		// (NotFoundHandler หรือ MethodNotAllowedHandler) เราดักดูว่า
		// มันตั้งใจตอบ status อะไร แล้วค่อยเขียน body เป็น JSON เอง
		rec := &statusInterceptor{ResponseWriter: w, status: http.StatusOK}
		h.ServeHTTP(rec, r)
		if rec.status == http.StatusMethodNotAllowed {
			writeError(w, http.StatusMethodNotAllowed, "method not allowed")
			return
		}
		writeError(w, http.StatusNotFound, "resource not found")
	})
}
```

เทคนิคนี้อาศัยความจริงที่ว่า `mux.Handler(r)` คืน handler ที่ **ถูกต้อง** เสมอ (แค่ pattern เป็นค่าว่างในกรณี 404/405) ดังนั้นเมื่อเราเรียก `h.ServeHTTP(rec, r)` กับ interceptor ปลอมๆ เราจะเห็น status code จริงที่ handler ภายในตั้งใจตอบ โดยไม่ต้องพึ่ง regular expression เดา หรือ hardcode logic ใดๆ เอง

> หมายเหตุ: เฉพาะ "path" ที่ handler ภายในตอบ 404/405 เท่านั้นที่ต้องผ่านการดักแบบนี้ — ถ้า pattern ตรงกันแล้ว (`pattern != ""`) เราปล่อยให้ `mux.ServeHTTP` ทำงานตามปกติ ไม่มี overhead อะไรเพิ่มเลยสำหรับ request ที่ปกติ

---

## 9. ตัวอย่างเต็มรูปแบบ: Notes API (In-Memory)

มารวมทุกอย่างที่เรียนมาในบทนี้เป็นแอปเดียว: **Notes API** ที่เก็บข้อมูลใน memory (ยังไม่ต่อฐานข้อมูลจริง — เรื่องนั้นเริ่มที่ **ภาคที่ 6**) รองรับ `GET /notes`, `POST /notes`, `GET /notes/{id}`, `DELETE /notes/{id}`

สร้างโปรเจกต์ใหม่:

```bash
mkdir notesapi && cd notesapi
go mod init notesapi
```

ไฟล์ `main.go`:

```go
package main

import (
	"encoding/json"
	"errors"
	"log"
	"net/http"
	"strconv"
	"sync"
)

type Note struct {
	ID    int    `json:"id"`
	Title string `json:"title"`
	Body  string `json:"body"`
}

// NoteStore เก็บ note ทั้งหมดไว้ใน map พร้อม mutex ป้องกัน race condition
// เมื่อมีหลาย goroutine (หลาย request พร้อมกัน) เข้าถึงพร้อมกัน (ทบทวน
// sync.Mutex ได้จาก Part 039)
type NoteStore struct {
	mu     sync.Mutex
	notes  map[int]Note
	nextID int
}

func NewNoteStore() *NoteStore {
	return &NoteStore{notes: make(map[int]Note), nextID: 1}
}

var ErrNotFound = errors.New("note not found")

func (s *NoteStore) List() []Note {
	s.mu.Lock()
	defer s.mu.Unlock()
	result := make([]Note, 0, len(s.notes))
	for _, n := range s.notes {
		result = append(result, n)
	}
	return result
}

func (s *NoteStore) Get(id int) (Note, error) {
	s.mu.Lock()
	defer s.mu.Unlock()
	n, ok := s.notes[id]
	if !ok {
		return Note{}, ErrNotFound
	}
	return n, nil
}

func (s *NoteStore) Create(title, body string) Note {
	s.mu.Lock()
	defer s.mu.Unlock()
	n := Note{ID: s.nextID, Title: title, Body: body}
	s.notes[n.ID] = n
	s.nextID++
	return n
}

func (s *NoteStore) Delete(id int) error {
	s.mu.Lock()
	defer s.mu.Unlock()
	if _, ok := s.notes[id]; !ok {
		return ErrNotFound
	}
	delete(s.notes, id)
	return nil
}

type NoteHandler struct {
	store *NoteStore
}

func (h *NoteHandler) list(w http.ResponseWriter, r *http.Request) {
	notes := h.store.List()
	writeJSON(w, http.StatusOK, notes)
}

func (h *NoteHandler) create(w http.ResponseWriter, r *http.Request) {
	var input struct {
		Title string `json:"title"`
		Body  string `json:"body"`
	}
	if err := json.NewDecoder(r.Body).Decode(&input); err != nil {
		writeError(w, http.StatusBadRequest, "invalid json body")
		return
	}
	if input.Title == "" {
		writeError(w, http.StatusUnprocessableEntity, "title is required")
		return
	}
	n := h.store.Create(input.Title, input.Body)
	writeJSON(w, http.StatusCreated, n)
}

func (h *NoteHandler) get(w http.ResponseWriter, r *http.Request) {
	id, err := strconv.Atoi(r.PathValue("id"))
	if err != nil {
		writeError(w, http.StatusBadRequest, "invalid id")
		return
	}
	n, err := h.store.Get(id)
	if err != nil {
		writeError(w, http.StatusNotFound, "note not found")
		return
	}
	writeJSON(w, http.StatusOK, n)
}

func (h *NoteHandler) delete(w http.ResponseWriter, r *http.Request) {
	id, err := strconv.Atoi(r.PathValue("id"))
	if err != nil {
		writeError(w, http.StatusBadRequest, "invalid id")
		return
	}
	if err := h.store.Delete(id); err != nil {
		writeError(w, http.StatusNotFound, "note not found")
		return
	}
	w.WriteHeader(http.StatusNoContent)
}

func writeJSON(w http.ResponseWriter, status int, v any) {
	w.Header().Set("Content-Type", "application/json")
	w.WriteHeader(status)
	json.NewEncoder(w).Encode(v)
}

func writeError(w http.ResponseWriter, status int, msg string) {
	writeJSON(w, status, map[string]string{"error": msg})
}

type statusInterceptor struct {
	http.ResponseWriter
	status int
}

func (w *statusInterceptor) WriteHeader(code int) { w.status = code }
func (w *statusInterceptor) Write(b []byte) (int, error) { return len(b), nil }

func withJSONErrors(mux *http.ServeMux) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		h, pattern := mux.Handler(r)
		if pattern != "" {
			mux.ServeHTTP(w, r)
			return
		}
		rec := &statusInterceptor{ResponseWriter: w, status: http.StatusOK}
		h.ServeHTTP(rec, r)
		if rec.status == http.StatusMethodNotAllowed {
			writeError(w, http.StatusMethodNotAllowed, "method not allowed")
			return
		}
		writeError(w, http.StatusNotFound, "resource not found")
	})
}

func main() {
	store := NewNoteStore()
	store.Create("Go", "learn go")
	h := &NoteHandler{store: store}

	mux := http.NewServeMux()
	mux.HandleFunc("GET /notes", h.list)
	mux.HandleFunc("POST /notes", h.create)
	mux.HandleFunc("GET /notes/{id}", h.get)
	mux.HandleFunc("DELETE /notes/{id}", h.delete)

	log.Println("listening on :8080")
	log.Fatal(http.ListenAndServe(":8080", withJSONErrors(mux)))
}
```

รันด้วย:

```bash
go run main.go
```

จะเห็น `listening on :8080` ปรากฏขึ้นมา (การ shutdown อย่างสุภาพด้วย `context` ได้เรียนไปแล้วใน Part 047 ในตัวอย่างนี้ตัดออกเพื่อให้โฟกัสกับเรื่อง routing เป็นหลัก)

---

## 10. ทดสอบ API ด้วย `curl` ทีละ endpoint

เปิด terminal อีกหน้าต่างแล้วยิง request ทดสอบทีละเส้นทาง (ผลลัพธ์ด้านล่างนี้คือผลจริงจากการรันโค้ดข้างต้น):

### GET /notes — ดู note ทั้งหมด

```bash
curl -s http://localhost:8080/notes
```

```json
[{"id":1,"title":"Go","body":"learn go"}]
```

### POST /notes — สร้าง note ใหม่

```bash
curl -s -X POST http://localhost:8080/notes -d '{"title":"Buy milk","body":"2 liters"}'
```

```json
{"id":2,"title":"Buy milk","body":"2 liters"}
```

### POST /notes ที่ขาด title (422)

```bash
curl -s -X POST http://localhost:8080/notes -d '{"body":"no title"}'
```

```
HTTP status: 422
{"error":"title is required"}
```

### GET /notes/{id} — ดู note ตัวเดียว

```bash
curl -s http://localhost:8080/notes/2
```

```json
{"id":2,"title":"Buy milk","body":"2 liters"}
```

### GET /notes/{id} ที่ไม่มีอยู่จริง (404)

```bash
curl -s http://localhost:8080/notes/999
```

```
HTTP status: 404
{"error":"note not found"}
```

### PUT /notes/{id} — method ที่ไม่ได้ลงทะเบียนไว้ (405)

```bash
curl -s -D - -X PUT http://localhost:8080/notes/1
```

```
HTTP/1.1 405 Method Not Allowed
Allow: DELETE, GET, HEAD
Content-Type: application/json

{"error":"method not allowed"}
```

สังเกตว่า header `Allow: DELETE, GET, HEAD` มาจาก `ServeMux` เองอัตโนมัติ (บอกเราตรงๆ ว่า path นี้รองรับ method อะไรบ้าง) — เราแค่แปลง body ให้เป็น JSON เท่านั้น ไม่ต้องคำนวณ `Allow` เอง

### DELETE /notes/{id} — ลบ note

```bash
curl -s -o /dev/null -w "%{http_code}\n" -X DELETE http://localhost:8080/notes/1
```

```
204
```

### เส้นทางที่ไม่มีอยู่เลย (404)

```bash
curl -s http://localhost:8080/foobar
```

```
HTTP status: 404
{"error":"resource not found"}
```

ทั้งหมดนี้ตรงกับพฤติกรรมที่ออกแบบไว้ทุกจุด: 404 สำหรับ path ที่ไม่รู้จัก, 404 สำหรับ id ที่ไม่มีอยู่จริง, 405 สำหรับ method ที่ path รู้จักแต่ไม่รองรับ, และ 422 สำหรับ input ที่ผิด validation — ครบทุก error case ที่ REST API ที่ดีควรมี

---

## 11. ข้อควรระวังและแนวทางปฏิบัติที่ดี

- **อย่าลืม `defer s.mu.Unlock()`** ทุกครั้งที่ล็อก mutex ใน store — ถ้าลืมจะเกิด deadlock ทันทีตั้งแต่ request ที่สองเป็นต้นไป
- **`r.Body` เป็น stream ที่อ่านได้ครั้งเดียว** — ถ้าต้องอ่านซ้ำ (เช่น log ดู raw body ก่อน decode) ต้องอ่านใส่ `[]byte` ก่อนแล้ว wrap ด้วย `bytes.NewReader` ทีหลัง (ทบทวน `io.Reader` ได้จาก Part 048)
- **`json.NewDecoder(r.Body).Decode(&v)` ไม่ปิด `r.Body` ให้เอง** — ปกติไม่ต้องปิดเองเพราะ `net/http` จัดการปิด body ให้หลัง handler คืนค่าเสมอ แต่ถ้าอ่านแบบ manual ด้วย `io.ReadAll` ก็ควรปิดตามหลัก resource management ปกติ
- **แยก pattern ให้ specific พอ** — ถ้า pattern คลุมเครือ (เช่น ทั้ง `/notes/{id}` และ `/notes/export` ที่อาจตีความ `export` เป็นค่า `{id}` ได้) ให้จำไว้ว่า `ServeMux` เลือก pattern ที่ literal มากกว่าเสมอ แต่การตั้งชื่อ endpoint ที่ชัดเจนตั้งแต่แรกช่วยลดความสับสนได้มากกว่าพึ่งกฎนี้
- **Handler ควรบางที่สุดเท่าที่ทำได้** — logic การตรวจสอบ, คำนวณ ควรอยู่ใน store/service layer ไม่ใช่ในตัว handler เอง เพื่อให้ทดสอบ business logic แยกจาก HTTP layer ได้ (แนวคิดนี้จะขยายเต็มรูปแบบใน **Part 100: Clean Architecture**)

---

## สรุปสิ่งที่ได้เรียนในบทนี้

- Go 1.22+ อัปเกรด `http.ServeMux` ให้รองรับ pattern แบบ `"METHOD /path/{id}"` ในตัว ไม่ต้องพึ่ง third-party router สำหรับความต้องการพื้นฐาน
- `r.PathValue("id")` ดึงค่าจาก wildcard segment ออกมาเป็น `string` เสมอ ต้องแปลงเองด้วย `strconv`
- จัดกลุ่ม handler ตาม resource ผ่านฟังก์ชัน `registerXxxRoutes` ทำให้ `main` อ่านง่ายและขยายง่าย
- Handler แบบ struct-based ที่ผูก dependency (store/service) เข้าไปใน field ทำให้ทดสอบและแยกส่วนได้ดีกว่า global variable
- `ServeMux` มี automatic 405 ในตัวเมื่อ path ตรงแต่ method ไม่ตรง — เราแค่ต้องแปลง body ให้เป็น JSON โดยใช้เทคนิค status interceptor
- ต้องเรียก `mux.ServeHTTP` (ไม่ใช่ `h.ServeHTTP` ตรงๆ) เมื่อ pattern ตรงกันแล้ว มิเช่นนั้น `r.PathValue` จะว่างเปล่า — นี่คือกับดักที่พบได้บ่อยเวลาทำ custom error handling รอบ mux

## แบบฝึกหัดท้ายบท

1. เพิ่ม endpoint `PUT /notes/{id}` สำหรับแก้ไข title/body ของ note ที่มีอยู่แล้ว พร้อมตอบ 404 ถ้าไม่พบ id นั้น
2. เพิ่ม field `Done bool` ให้ `Note` แล้วทำ endpoint `GET /notes?done=true` กรองเฉพาะ note ที่เสร็จแล้ว (ใช้ `r.URL.Query()`)
3. ลองลบ `withJSONErrors` ออก แล้วสังเกตความแตกต่างของ response เมื่อยิง path ที่ไม่มีอยู่จริงและ method ที่ไม่รองรับ — เทียบ header และ body ให้เห็นชัดว่าต่างกันตรงไหน
4. เขียนฟังก์ชัน `registerNoteRoutes(mux *http.ServeMux, h *NoteHandler)` แยกออกจาก `main` ตามแนวทางหัวข้อที่ 5 แล้วลองเพิ่ม resource ใหม่ชื่อ `tags` (`GET /tags`, `POST /tags`) แบบเดียวกัน
5. ทดลองใช้ wildcard แบบ `{id...}` (กินรวด path ที่เหลือ) กับ endpoint `GET /files/{path...}` แล้วสังเกตว่า `r.PathValue("path")` คืนค่าอะไรเมื่อยิง `GET /files/a/b/c`
6. อธิบายด้วยคำพูดตัวเอง (เตรียมคำตอบไว้) ว่าทำไม `mux.Handler(r)` ถึงไม่เซ็ต path value ให้ ทั้งที่มันรู้ pattern ที่ match แล้ว — ลองเดาเหตุผลเชิงออกแบบก่อนเปิดดูซอร์สโค้ดจริงของ `net/http`

---

**ต่อไป**: [Part 057 — Middleware Pattern](./057-middleware-pattern.md)
