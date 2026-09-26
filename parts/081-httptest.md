# Part 081: `httptest` — ทดสอบ HTTP Handler

> ภาคที่ 7: Testing, Tooling, Performance — ตอนที่ 3 จาก 9 (Part 79–87)

## สารบัญของบทนี้

1. ทบทวนภาพรวม HTTP ใน Go และตำแหน่งของ `httptest`
2. `httptest.NewRecorder()`: ทดสอบ Handler โดยไม่มี Network จริง
3. Table-Driven Handler Test: รวม Part 034 เข้ากับ `httptest`
4. `httptest.NewServer()`: ทดสอบฝั่ง Client กับ Server จริง
5. ทดสอบ Middleware ด้วย `httptest` (ทบทวน Part 057)
6. ทดสอบ JSON API แบบ End-to-End เต็มรูปแบบ
7. ข้อควรระวังและ Best Practice
8. สรุปสิ่งที่ได้เรียนในบทนี้
9. แบบฝึกหัดท้ายบท

---

## 1. ทบทวนภาพรวม HTTP ใน Go และตำแหน่งของ `httptest`

**Part 046** สอน `net/http` ฝั่ง client เจาะลึก, **Part 047** สอนฝั่ง server, **Part 056-057** สอนการทำ routing และ middleware pattern บน `net/http` โดยตรง สิ่งที่ทุกบทเหล่านั้นยังไม่ได้ตอบคือ: **แล้วจะเขียน automated test (ทบทวนจาก Part 033) ให้กับ handler และ client เหล่านี้อย่างไร โดยไม่ต้องเปิด server จริงบน port จริงทุกครั้งที่รัน test?**

คำตอบคือ package **`net/http/httptest`** ซึ่งเป็นส่วนหนึ่งของ standard library (สอดคล้องกับปรัชญา "standard library แข็งแกร่ง" จาก **Part 001**) ให้เครื่องมือหลัก 2 ตัวที่ครอบคลุมสถานการณ์ทดสอบ HTTP แทบทั้งหมด:

| เครื่องมือ | ใช้ทดสอบอะไร | มี network จริงไหม |
|---|---|---|
| **`httptest.NewRecorder()`** | **Handler** ตัวเดียว (หรือทั้ง mux ที่ประกอบแล้ว) โดยตรง | ไม่มี — เรียก handler function ตรงๆ ในหน่วยความจำ |
| **`httptest.NewServer()`** | **Client** ที่ต้องยิง HTTP request จริงออกไปหา server | มี — เปิด TCP listener จริงบน `localhost` พอร์ตที่ระบบสุ่มให้ |

หลักการเลือกใช้ง่ายๆ คือ: **ถ้าโค้ดที่ทดสอบเป็นฝั่งที่ "รับ" request (handler)** ใช้ `NewRecorder` เพราะเร็วกว่าและไม่ต้องพึ่ง network stack จริงเลย **ถ้าโค้ดที่ทดสอบเป็นฝั่งที่ "ส่ง" request (client)** ต้องใช้ `NewServer` เพราะ client code เรียก `http.Client` ที่ต้องมี URL จริงให้ยิงไปหา จะส่ง `ResponseRecorder` เข้าไปแทนไม่ได้เลย

บทนี้จะพาไปใช้ทั้งสองเครื่องมือกับตัวอย่างเดียวกัน: JSON API เล็กๆ สำหรับจัดการ user (สร้าง + ดึงข้อมูล) พร้อม middleware ตรวจสอบ API key

---

## 2. `httptest.NewRecorder()`: ทดสอบ Handler โดยไม่มี Network จริง

`httptest.NewRecorder()` สร้างค่าที่ implement interface `http.ResponseWriter` (ทบทวนจาก **Part 047**) แต่แทนที่จะเขียนข้อมูลออกทาง network จริง มันแค่**บันทึก**ทุกอย่างที่ handler เขียนไว้ในหน่วยความจำ: status code, header, และ body — ให้เรา assert ค่าพวกนี้ได้หลัง handler ทำงานเสร็จ

คู่กับ `httptest.NewRequest(method, target string, body io.Reader)` ที่สร้าง `*http.Request` ปลอมขึ้นมาโดยไม่ต้องผ่าน network เช่นกัน (ต่างจาก `http.NewRequest` ปกติที่ต้องระบุ URL แบบเต็มรวม scheme/host แต่ `httptest.NewRequest` ยอมรับแค่ path ได้เลย เพราะไม่ได้ตั้งใจจะยิงออกไปไหนจริงๆ)

มาดูตัวอย่าง Server API สำหรับสร้างและดึงข้อมูล user:

```go
// server.go
package api

import (
	"encoding/json"
	"fmt"
	"net/http"
	"strconv"
	"strings"
	"sync"
)

// CreateUserRequest คือ payload ที่ client ส่งมาตอนสร้าง user ใหม่
type CreateUserRequest struct {
	Name  string `json:"name"`
	Email string `json:"email"`
}

// UserResponse คือ payload ที่ server ส่งกลับ (ทั้งตอนสร้างและตอนดึงข้อมูล)
type UserResponse struct {
	ID    int    `json:"id"`
	Name  string `json:"name"`
	Email string `json:"email"`
}

// Server คือ JSON API แบบง่ายที่เก็บ user ไว้ใน memory (เพื่อความกระชับของตัวอย่าง)
// ในโปรเจกต์จริงส่วนนี้จะเป็น service ที่คุยกับฐานข้อมูล (ดู Part 080)
type Server struct {
	mu     sync.Mutex
	users  map[int]UserResponse
	nextID int
}

// NewServer สร้าง Server พร้อม state เริ่มต้นว่างเปล่า
func NewServer() *Server {
	return &Server{users: map[int]UserResponse{}, nextID: 1}
}

// CreateUser คือ http.HandlerFunc สำหรับ POST /users
func (s *Server) CreateUser(w http.ResponseWriter, r *http.Request) {
	if r.Method != http.MethodPost {
		http.Error(w, "method not allowed", http.StatusMethodNotAllowed)
		return
	}

	var req CreateUserRequest
	if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
		http.Error(w, "invalid json body", http.StatusBadRequest)
		return
	}
	if req.Name == "" {
		http.Error(w, "name is required", http.StatusBadRequest)
		return
	}

	s.mu.Lock()
	id := s.nextID
	s.nextID++
	u := UserResponse{ID: id, Name: req.Name, Email: req.Email}
	s.users[id] = u
	s.mu.Unlock()

	w.Header().Set("Content-Type", "application/json")
	w.WriteHeader(http.StatusCreated)
	_ = json.NewEncoder(w).Encode(u)
}

// GetUser คือ http.HandlerFunc สำหรับ GET /users/{id}
func (s *Server) GetUser(w http.ResponseWriter, r *http.Request) {
	if r.Method != http.MethodGet {
		http.Error(w, "method not allowed", http.StatusMethodNotAllowed)
		return
	}

	idStr := strings.TrimPrefix(r.URL.Path, "/users/")
	id, err := strconv.Atoi(idStr)
	if err != nil {
		http.Error(w, "invalid user id", http.StatusBadRequest)
		return
	}

	s.mu.Lock()
	u, ok := s.users[id]
	s.mu.Unlock()
	if !ok {
		http.Error(w, "user not found", http.StatusNotFound)
		return
	}

	w.Header().Set("Content-Type", "application/json")
	_ = json.NewEncoder(w).Encode(u)
}

// Routes ประกอบ handler ทั้งหมดเข้าเป็น http.Handler เดียว (ทบทวนจาก Part 056)
func (s *Server) Routes() http.Handler {
	mux := http.NewServeMux()
	mux.HandleFunc("/users", s.CreateUser)
	mux.HandleFunc("/users/", s.GetUser)
	return mux
}
```

ทดสอบ `CreateUser` ด้วย `httptest.NewRecorder()` ตรงๆ โดยไม่ผ่าน mux เลย:

```go
// server_test.go
package api

import (
	"bytes"
	"encoding/json"
	"net/http"
	"net/http/httptest"
	"testing"
)

// TestCreateUser_Recorder สาธิตการใช้ httptest.NewRecorder() ทดสอบ Handler โดยตรง
// โดยไม่มี network listener จริงเกิดขึ้นเลย: เราสร้าง *http.Request ปลอมด้วยมือ
// (httptest.NewRequest) แล้วส่งเข้า handler พร้อม httptest.ResponseRecorder ที่ทำหน้าที่
// "จำลอง" http.ResponseWriter คอยบันทึกทุกอย่างที่ handler เขียนออกมา
func TestCreateUser_Recorder(t *testing.T) {
	s := NewServer()

	body := bytes.NewBufferString(`{"name":"Alice","email":"alice@example.com"}`)
	req := httptest.NewRequest(http.MethodPost, "/users", body)
	rec := httptest.NewRecorder()

	s.CreateUser(rec, req)

	if rec.Code != http.StatusCreated {
		t.Fatalf("status = %d, want %d; body=%s", rec.Code, http.StatusCreated, rec.Body.String())
	}

	var got UserResponse
	if err := json.NewDecoder(rec.Body).Decode(&got); err != nil {
		t.Fatalf("decode response: %v", err)
	}
	if got.Name != "Alice" || got.Email != "alice@example.com" || got.ID == 0 {
		t.Errorf("response = %+v, want Name=Alice Email=alice@example.com with non-zero ID", got)
	}
}
```

รันแล้วผ่านทันที:

```
=== RUN   TestCreateUser_Recorder
--- PASS: TestCreateUser_Recorder (0.00s)
```

สังเกตจุดสำคัญของรูปแบบนี้:

1. **เรียก `s.CreateUser(rec, req)` ตรงๆ** — ไม่ผ่าน `http.ListenAndServe`, ไม่มี TCP socket, ไม่มี network stack ของ OS เข้ามาเกี่ยวข้องเลย ทำให้ test รันเร็วมาก (เทียบเท่า unit test ปกติ)
2. **`rec.Code`** เก็บ status code ที่ handler เรียก `w.WriteHeader(...)` ไว้ (หรือ `200` โดย default ถ้าไม่เคยเรียกเลย)
3. **`rec.Body`** เป็น `*bytes.Buffer` เก็บทุกอย่างที่ handler เขียนผ่าน `w.Write(...)` หรือ `fmt.Fprint(w, ...)` — decode เป็น JSON กลับได้ตรงๆ เหมือนกับ `resp.Body` ของ HTTP response จริง
4. **`rec.Header()`** คืน `http.Header` ที่บันทึก header ทั้งหมดที่ handler ตั้งไว้ (ดูตัวอย่างการใช้ใน หัวข้อ 5)

---

## 3. Table-Driven Handler Test: รวม Part 034 เข้ากับ `httptest`

รูปแบบ `httptest.NewRecorder()` เข้ากันได้ดีมากกับ **table-driven test** ที่เรียนใน **Part 034** — สร้างตารางกรณีทดสอบ ระบุ request body กับ status code ที่คาดหวัง แล้ววนลูปยิงเข้า handler:

```go
// TestCreateUser_TableDriven รวม table-driven test (Part 034) เข้ากับ httptest.NewRecorder
// เพื่อทดสอบหลาย request body ในฟังก์ชันเดียว
func TestCreateUser_TableDriven(t *testing.T) {
	tests := []struct {
		name       string
		body       string
		wantStatus int
	}{
		{"สร้างสำเร็จ", `{"name":"Bob","email":"bob@example.com"}`, http.StatusCreated},
		{"ไม่มี name", `{"email":"noname@example.com"}`, http.StatusBadRequest},
		{"JSON ผิดรูปแบบ", `{invalid-json`, http.StatusBadRequest},
		{"body ว่าง", ``, http.StatusBadRequest},
	}

	for _, tt := range tests {
		t.Run(tt.name, func(t *testing.T) {
			s := NewServer() // สร้าง server ใหม่ทุก subtest ป้องกัน state รั่วไหลข้าม test
			req := httptest.NewRequest(http.MethodPost, "/users", bytes.NewBufferString(tt.body))
			rec := httptest.NewRecorder()

			s.CreateUser(rec, req)

			if rec.Code != tt.wantStatus {
				t.Errorf("status = %d, want %d; body=%s", rec.Code, tt.wantStatus, rec.Body.String())
			}
		})
	}
}
```

รันด้วย `-v` เห็นทุกกรณีแยกชัดเจน:

```
=== RUN   TestCreateUser_TableDriven
=== RUN   TestCreateUser_TableDriven/สร้างสำเร็จ
=== RUN   TestCreateUser_TableDriven/ไม่มี_name
=== RUN   TestCreateUser_TableDriven/JSON_ผิดรูปแบบ
=== RUN   TestCreateUser_TableDriven/body_ว่าง
--- PASS: TestCreateUser_TableDriven (0.00s)
    --- PASS: TestCreateUser_TableDriven/สร้างสำเร็จ (0.00s)
    --- PASS: TestCreateUser_TableDriven/ไม่มี_name (0.00s)
    --- PASS: TestCreateUser_TableDriven/JSON_ผิดรูปแบบ (0.00s)
    --- PASS: TestCreateUser_TableDriven/body_ว่าง (0.00s)
```

จุดสำคัญคือการสร้าง **`s := NewServer()` ใหม่ในทุก subtest** แทนที่จะใช้ตัวเดียวร่วมกันทั้งตาราง — ถ้าใช้ตัวเดียวกัน กรณี "สร้างสำเร็จ" ของ subtest แรกจะเพิ่ม user เข้าไปใน map แล้วส่งผลกระทบต่อ `nextID` ของ subtest ถัดไปโดยไม่ตั้งใจ (หลักการเดียวกับ **test isolation** ที่พูดถึงใน **Part 080** เรื่อง test pollution ระหว่าง integration test)

ทดสอบเส้นทาง `GetUser` ที่หา user ไม่เจอเพิ่มอีกกรณี และทดสอบทั้ง mux ที่ประกอบแล้วแบบ end-to-end ในหน่วยความจำ (สร้าง user ก่อน แล้วดึงกลับมาด้วย id ที่ได้):

```go
func TestGetUser_NotFound(t *testing.T) {
	s := NewServer()

	req := httptest.NewRequest(http.MethodGet, "/users/42", nil)
	rec := httptest.NewRecorder()

	s.GetUser(rec, req)

	if rec.Code != http.StatusNotFound {
		t.Errorf("status = %d, want %d", rec.Code, http.StatusNotFound)
	}
}

// TestRoutes_EndToEnd ทดสอบผ่าน s.Routes() (http.Handler ที่ประกอบ mux ทั้งหมดแล้ว)
// เพื่อยืนยันว่าเส้นทาง /users และ /users/{id} เชื่อมกันถูกต้อง: สร้าง user ก่อน
// แล้วดึงกลับมาด้วย id ที่ได้ ตรวจสอบทั้ง status code และ payload JSON ทั้งสองฝั่ง
func TestRoutes_EndToEnd(t *testing.T) {
	s := NewServer()
	handler := s.Routes()

	createReq := httptest.NewRequest(http.MethodPost, "/users",
		bytes.NewBufferString(`{"name":"Carol","email":"carol@example.com"}`))
	createRec := httptest.NewRecorder()
	handler.ServeHTTP(createRec, createReq)

	if createRec.Code != http.StatusCreated {
		t.Fatalf("create status = %d, want %d", createRec.Code, http.StatusCreated)
	}
	var created UserResponse
	if err := json.NewDecoder(createRec.Body).Decode(&created); err != nil {
		t.Fatalf("decode create response: %v", err)
	}

	getReq := httptest.NewRequest(http.MethodGet, "/users/1", nil)
	getRec := httptest.NewRecorder()
	handler.ServeHTTP(getRec, getReq)

	if getRec.Code != http.StatusOK {
		t.Fatalf("get status = %d, want %d; body=%s", getRec.Code, http.StatusOK, getRec.Body.String())
	}
	var fetched UserResponse
	if err := json.NewDecoder(getRec.Body).Decode(&fetched); err != nil {
		t.Fatalf("decode get response: %v", err)
	}
	if fetched != created {
		t.Errorf("fetched = %+v, want %+v", fetched, created)
	}
}
```

สังเกตว่า `handler.ServeHTTP(rec, req)` ถูกเรียกแทน `s.CreateUser(rec, req)` ตรงๆ — เพราะตอนนี้เราทดสอบผ่าน `http.Handler` ที่ประกอบ mux ไว้แล้ว (ผลลัพธ์จาก `s.Routes()`) ซึ่งเป็นสิ่งเดียวกับที่ `http.ListenAndServe` จะเรียกจริงตอน production ทุกประการ — เพียงแต่ตอน test เราข้ามขั้นตอนการเปิด network listener ไปเท่านั้น

---

## 4. `httptest.NewServer()`: ทดสอบฝั่ง Client กับ Server จริง

`httptest.NewRecorder()` ใช้ไม่ได้เมื่อโค้ดที่ต้องการทดสอบเป็น**ฝั่งเรียก** (client) เพราะ `http.Client` (ทบทวนจาก **Part 046**) ต้องการ URL จริงที่ยิง request ออกไปได้ทาง network — ตรงนี้เองที่ `httptest.NewServer()` เข้ามาเติมเต็ม: มันเปิด HTTP server จริงบน `localhost` ที่ port ว่างซึ่งระบบปฏิบัติการจัดสรรให้อัตโนมัติ (ไม่ชนกับ port อื่นที่ใช้งานอยู่แน่นอน)

มาดูฝั่ง client ที่จะทดสอบ — ฟังก์ชันที่ดึงข้อมูล user จาก API ภายนอก:

```go
// FetchUser คือ HTTP client ฝั่งผู้เรียกใช้ API (ทบทวน http.Client จาก Part 046)
// รับ *http.Client เข้ามาเป็นพารามิเตอร์ (แทนที่จะใช้ http.DefaultClient ตรงๆ)
// เพื่อให้ทดสอบง่าย: ตอน test เราจะส่ง client ของ httptest server เข้ามาแทน
func FetchUser(client *http.Client, baseURL string, id int) (*UserResponse, error) {
	url := fmt.Sprintf("%s/users/%d", baseURL, id)
	resp, err := client.Get(url)
	if err != nil {
		return nil, fmt.Errorf("fetch user: %w", err)
	}
	defer resp.Body.Close()

	if resp.StatusCode != http.StatusOK {
		return nil, fmt.Errorf("fetch user: unexpected status %d", resp.StatusCode)
	}

	var u UserResponse
	if err := json.NewDecoder(resp.Body).Decode(&u); err != nil {
		return nil, fmt.Errorf("fetch user: decode response: %w", err)
	}
	return &u, nil
}
```

สังเกตว่า `FetchUser` รับ `*http.Client` และ `baseURL` เป็นพารามิเตอร์ **แทนที่จะ hardcode `http.DefaultClient` และ URL จริงไว้ในฟังก์ชัน** — นี่คือหลักการ **dependency injection** เดียวกับที่เรียนไปกับ `Clock` interface ใน **Part 079**: การ inject dependency เข้ามาแทนการอ้างอิงตรงๆ ทำให้ตอน test สามารถสลับเป็นของปลอม (ในที่นี้คือ test server) ได้อย่างง่ายดาย โดยไม่ต้องแก้โค้ด production เลยสักบรรทัด

ทดสอบด้วย `httptest.NewServer()`:

```go
// client_test.go
package api

import (
	"bytes"
	"encoding/json"
	"net/http"
	"net/http/httptest"
	"testing"
)

// TestFetchUser_AgainstRealTestServer สาธิตการใช้ httptest.NewServer() ซึ่งต่างจาก
// httptest.NewRecorder() ตรงที่มันเปิด network listener จริง (บน localhost, port สุ่ม)
// ทำให้เหมาะกับการทดสอบฝั่ง "client" ที่ใช้ http.Client จริงยิงคำขอออกไปทาง network
// จริงๆ (ผ่าน TCP loopback) แทนที่จะเรียก handler ตรงๆ แบบ NewRecorder
func TestFetchUser_AgainstRealTestServer(t *testing.T) {
	s := NewServer()
	// สร้าง user ไว้ล่วงหน้าหนึ่งคนก่อนเปิด test server
	s.users[1] = UserResponse{ID: 1, Name: "Dave", Email: "dave@example.com"}
	s.nextID = 2

	// httptest.NewServer เปิด HTTP server จริงบน port ว่างที่ระบบจัดสรรให้อัตโนมัติ
	testServer := httptest.NewServer(s.Routes())
	defer testServer.Close() // ต้องปิดเสมอ ไม่งั้น listener จะค้างอยู่หลัง test จบ

	// testServer.Client() คืน *http.Client ที่ตั้งค่าให้เชื่อมต่อกับ testServer ได้ทันที
	got, err := FetchUser(testServer.Client(), testServer.URL, 1)
	if err != nil {
		t.Fatalf("FetchUser() error = %v", err)
	}
	if got.Name != "Dave" || got.Email != "dave@example.com" {
		t.Errorf("FetchUser() = %+v, want Name=Dave Email=dave@example.com", got)
	}
}

// TestFetchUser_NotFound ยืนยันว่า FetchUser คืน error ที่สื่อความหมายเมื่อ server ตอบ 404
func TestFetchUser_NotFound(t *testing.T) {
	s := NewServer()
	testServer := httptest.NewServer(s.Routes())
	defer testServer.Close()

	_, err := FetchUser(testServer.Client(), testServer.URL, 999)
	if err == nil {
		t.Fatal("expected error for non-existent user, got nil")
	}
}
```

รันแล้วผ่านทั้งสอง:

```
=== RUN   TestFetchUser_AgainstRealTestServer
--- PASS: TestFetchUser_AgainstRealTestServer (0.00s)
=== RUN   TestFetchUser_NotFound
--- PASS: TestFetchUser_NotFound (0.00s)
```

**สิ่งสำคัญที่ต้องจำ:**

1. **`testServer.URL`** เป็น string ของ URL เต็ม พร้อม scheme และ port แบบสุ่มที่ระบบจัดให้ เช่น `http://127.0.0.1:41823` — ใช้แทน URL จริงของ API ได้ทันที
2. **`testServer.Client()`** คืน `*http.Client` ที่ config ไว้ให้เหมาะกับการยิงไปหา test server นี้โดยเฉพาะแล้ว (จัดการเรื่อง TLS certificate กรณีใช้ `httptest.NewTLSServer` ด้วย) — ควรใช้ตัวนี้แทนการสร้าง `&http.Client{}` เปล่าๆ เอง
3. **ต้องเรียก `defer testServer.Close()` เสมอ** เพื่อปิด listener หลัง test จบ — ถ้าลืม จะทำให้มี goroutine และ socket ค้างอยู่เบื้องหลังสะสมไปเรื่อยๆ ตลอดการรัน test suite ทั้งหมด (โดยเฉพาะอันตรายถ้ามี test หลายร้อยตัวที่เปิด test server)
4. Network ที่ใช้จริงคือ **TCP ผ่าน loopback interface (`127.0.0.1`)** ไม่ใช่แค่ mock — request วิ่งผ่าน TCP/IP stack ของ OS จริง เพียงแต่ไม่ออกไปนอกเครื่องเท่านั้น ทำให้มั่นใจได้ว่าพฤติกรรมของ `http.Client` (เช่น connection pooling, timeout, redirect handling) ถูกทดสอบภายใต้สภาพแวดล้อมที่ใกล้เคียงของจริงมากกว่า `NewRecorder`

---

## 5. ทดสอบ Middleware ด้วย `httptest` (ทบทวน Part 057)

**Part 057** สอนรูปแบบ middleware pattern: ฟังก์ชันที่รับ `http.Handler` ตัวหนึ่งแล้วคืน `http.Handler` ตัวใหม่ที่ "ห่อ" พฤติกรรมเพิ่มเติมไว้รอบนอก การทดสอบ middleware มีเทคนิคเฉพาะที่ควรรู้: **สร้าง "dummy handler" ปลอมง่ายๆ ไว้ห่อ แทนที่จะใช้ handler จริงที่ซับซ้อน** เพื่อให้ test ของ middleware แยกอิสระจาก business logic ของ handler ปลายทางอย่างสมบูรณ์

```go
// middleware.go
package api

import "net/http"

// RequireAPIKey คือ middleware pattern (ทบทวนจาก Part 057): รับ handler ตัวถัดไป
// แล้วคืน http.Handler ตัวใหม่ที่ตรวจสอบ header "X-API-Key" ก่อนส่งต่อ request จริง
func RequireAPIKey(key string, next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		if r.Header.Get("X-API-Key") != key {
			http.Error(w, "unauthorized", http.StatusUnauthorized)
			return
		}
		next.ServeHTTP(w, r)
	})
}

// WithRequestID คือ middleware อีกตัวที่เติม header "X-Request-ID" เข้าไปใน response เสมอ
func WithRequestID(id string, next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		w.Header().Set("X-Request-ID", id)
		next.ServeHTTP(w, r)
	})
}
```

```go
// middleware_test.go
package api

import (
	"net/http"
	"net/http/httptest"
	"testing"
)

// dummyHandler คือ handler จำลองง่ายๆ ใช้ทดสอบว่า middleware ทำงานถูกต้อง
// โดยไม่ต้องพึ่ง handler จริงที่ซับซ้อนของ Server เลย — เทคนิคนี้ทำให้ test ของ
// middleware แยกอิสระจาก business logic ของ handler ปลายทางอย่างสมบูรณ์
func dummyHandler() http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		w.WriteHeader(http.StatusOK)
		_, _ = w.Write([]byte("reached inner handler"))
	})
}

func TestRequireAPIKey_TableDriven(t *testing.T) {
	handler := RequireAPIKey("secret-key", dummyHandler())

	tests := []struct {
		name       string
		apiKey     string
		wantStatus int
		wantBody   string
	}{
		{"key ถูกต้อง", "secret-key", http.StatusOK, "reached inner handler"},
		{"key ผิด", "wrong-key", http.StatusUnauthorized, "unauthorized\n"},
		{"ไม่มี key เลย", "", http.StatusUnauthorized, "unauthorized\n"},
	}

	for _, tt := range tests {
		t.Run(tt.name, func(t *testing.T) {
			req := httptest.NewRequest(http.MethodGet, "/anything", nil)
			if tt.apiKey != "" {
				req.Header.Set("X-API-Key", tt.apiKey)
			}
			rec := httptest.NewRecorder()

			handler.ServeHTTP(rec, req)

			if rec.Code != tt.wantStatus {
				t.Errorf("status = %d, want %d", rec.Code, tt.wantStatus)
			}
			if rec.Body.String() != tt.wantBody {
				t.Errorf("body = %q, want %q", rec.Body.String(), tt.wantBody)
			}
		})
	}
}

// TestWithRequestID ยืนยันว่า middleware เติม header ให้ response โดยที่ inner handler
// ไม่ต้องรู้เรื่อง request id เลยแม้แต่น้อย (separation of concerns ของ middleware pattern)
func TestWithRequestID(t *testing.T) {
	handler := WithRequestID("req-123", dummyHandler())

	req := httptest.NewRequest(http.MethodGet, "/anything", nil)
	rec := httptest.NewRecorder()

	handler.ServeHTTP(rec, req)

	if got := rec.Header().Get("X-Request-ID"); got != "req-123" {
		t.Errorf("X-Request-ID header = %q, want %q", got, "req-123")
	}
	if rec.Code != http.StatusOK {
		t.Errorf("status = %d, want %d", rec.Code, http.StatusOK)
	}
}

// TestMiddlewareChaining ทดสอบการซ้อน middleware หลายชั้นเข้าด้วยกัน
// (RequireAPIKey ห่อ WithRequestID ห่อ dummyHandler) เพื่อยืนยันว่าทั้ง chain ทำงานถูกต้อง
func TestMiddlewareChaining(t *testing.T) {
	chained := RequireAPIKey("secret-key", WithRequestID("req-999", dummyHandler()))

	req := httptest.NewRequest(http.MethodGet, "/anything", nil)
	req.Header.Set("X-API-Key", "secret-key")
	rec := httptest.NewRecorder()

	chained.ServeHTTP(rec, req)

	if rec.Code != http.StatusOK {
		t.Fatalf("status = %d, want %d", rec.Code, http.StatusOK)
	}
	if got := rec.Header().Get("X-Request-ID"); got != "req-999" {
		t.Errorf("X-Request-ID header = %q, want %q", got, "req-999")
	}
}
```

ผลการรันทั้ง 3 test:

```
=== RUN   TestRequireAPIKey_TableDriven
=== RUN   TestRequireAPIKey_TableDriven/key_ถูกต้อง
=== RUN   TestRequireAPIKey_TableDriven/key_ผิด
=== RUN   TestRequireAPIKey_TableDriven/ไม่มี_key_เลย
--- PASS: TestRequireAPIKey_TableDriven (0.00s)
    --- PASS: TestRequireAPIKey_TableDriven/key_ถูกต้อง (0.00s)
    --- PASS: TestRequireAPIKey_TableDriven/key_ผิด (0.00s)
    --- PASS: TestRequireAPIKey_TableDriven/ไม่มี_key_เลย (0.00s)
=== RUN   TestWithRequestID
--- PASS: TestWithRequestID (0.00s)
=== RUN   TestMiddlewareChaining
--- PASS: TestMiddlewareChaining (0.00s)
```

`TestMiddlewareChaining` สาธิตจุดที่สำคัญมากในทางปฏิบัติ: middleware แทบทุกโปรเจกต์จริงถูก**ซ้อนกันหลายชั้น** (logging, auth, request-id, recovery จาก panic ฯลฯ) การทดสอบว่าแต่ละชั้นทำงานถูกต้อง**เมื่อรวมกัน**สำคัญไม่แพ้การทดสอบแต่ละชั้นแยกกัน เพราะบางครั้งลำดับการห่อ (`A(B(C(handler)))` เทียบกับ `B(A(C(handler)))`) ส่งผลต่อพฤติกรรมจริงได้ เช่น ถ้า logging middleware อยู่ชั้นนอกสุดของ auth middleware มันจะ log แม้แต่ request ที่ถูกปฏิเสธเพราะ API key ผิดด้วย ต่างจากถ้าสลับลำดับกัน

---

## 6. ทดสอบ JSON API แบบ End-to-End เต็มรูปแบบ

มารวมทุกเทคนิคเข้าด้วยกันในตัวอย่างสุดท้าย: ทดสอบ JSON API แบบเต็มรูปแบบผ่าน `httptest.NewServer()` — เข้ารหัส request body เป็น JSON เอง ยิงผ่าน `http.Client` จริง แล้วถอดรหัส response กลับมาตรวจสอบทั้ง status code และ payload ทั้งสองฝั่ง เหมือนกับสิ่งที่ client ภายนอกจริงจะทำ:

```go
// TestFullJSONRoundTrip ทดสอบ JSON API แบบ end-to-end เต็มรูปแบบผ่าน httptest.NewServer:
// ยิง POST ด้วย http.Client จริง เข้ารหัส request body เป็น JSON เอง แล้วถอดรหัส
// response กลับมาตรวจสอบทั้ง status code และ payload — จำลองสิ่งที่ client ภายนอกจะทำจริง
func TestFullJSONRoundTrip(t *testing.T) {
	s := NewServer()
	testServer := httptest.NewServer(s.Routes())
	defer testServer.Close()

	reqBody, err := json.Marshal(CreateUserRequest{Name: "Eve", Email: "eve@example.com"})
	if err != nil {
		t.Fatalf("marshal request: %v", err)
	}

	resp, err := testServer.Client().Post(
		testServer.URL+"/users",
		"application/json",
		bytes.NewReader(reqBody),
	)
	if err != nil {
		t.Fatalf("POST /users: %v", err)
	}
	defer resp.Body.Close()

	if resp.StatusCode != http.StatusCreated {
		t.Fatalf("status = %d, want %d", resp.StatusCode, http.StatusCreated)
	}

	var created UserResponse
	if err := json.NewDecoder(resp.Body).Decode(&created); err != nil {
		t.Fatalf("decode response: %v", err)
	}
	if created.Name != "Eve" || created.Email != "eve@example.com" || created.ID == 0 {
		t.Errorf("created = %+v, want Name=Eve Email=eve@example.com with non-zero ID", created)
	}

	// ยืนยันด้วยการดึงกลับมาผ่าน FetchUser อีกครั้ง เพื่อพิสูจน์ว่าข้อมูลถูกบันทึกจริง
	fetched, err := FetchUser(testServer.Client(), testServer.URL, created.ID)
	if err != nil {
		t.Fatalf("FetchUser() error = %v", err)
	}
	if *fetched != created {
		t.Errorf("fetched = %+v, want %+v", fetched, created)
	}
}
```

รันทั้งไฟล์ทั้งหมดพร้อม `-race -cover` เพื่อความมั่นใจสูงสุด (ทบทวน race detector จาก **Part 044**):

```bash
go test -race -v -cover ./...
```

```
=== RUN   TestFetchUser_AgainstRealTestServer
--- PASS: TestFetchUser_AgainstRealTestServer (0.00s)
=== RUN   TestFetchUser_NotFound
--- PASS: TestFetchUser_NotFound (0.00s)
=== RUN   TestFullJSONRoundTrip
--- PASS: TestFullJSONRoundTrip (0.00s)
=== RUN   TestRequireAPIKey_TableDriven
    --- PASS: TestRequireAPIKey_TableDriven/key_ถูกต้อง (0.00s)
    --- PASS: TestRequireAPIKey_TableDriven/key_ผิด (0.00s)
    --- PASS: TestRequireAPIKey_TableDriven/ไม่มี_key_เลย (0.00s)
=== RUN   TestWithRequestID
--- PASS: TestWithRequestID (0.00s)
=== RUN   TestMiddlewareChaining
--- PASS: TestMiddlewareChaining (0.00s)
=== RUN   TestCreateUser_Recorder
--- PASS: TestCreateUser_Recorder (0.00s)
=== RUN   TestCreateUser_TableDriven
    --- PASS: TestCreateUser_TableDriven/สร้างสำเร็จ (0.00s)
    --- PASS: TestCreateUser_TableDriven/ไม่มี_name (0.00s)
    --- PASS: TestCreateUser_TableDriven/JSON_ผิดรูปแบบ (0.00s)
    --- PASS: TestCreateUser_TableDriven/body_ว่าง (0.00s)
--- PASS: TestGetUser_NotFound (0.00s)
--- PASS: TestRoutes_EndToEnd (0.00s)
PASS
coverage: 86.4% of statements
ok  	part081demo/api	1.024s
```

ทุก test ผ่านหมด ไม่มีการแจ้ง race condition ใดๆ (แม้ `Server` จะมี `sync.Mutex` ป้องกัน `map` ที่ใช้ร่วมกันไว้แล้วก็ตาม — `-race` ยืนยันว่าการล็อกนั้นถูกต้องจริง) และ coverage 86.4% — ตัวเลขนี้จะเจาะลึกความหมายเพิ่มเติมใน **Part 082**

---

## 7. ข้อควรระวังและ Best Practice

สรุปข้อควรระวังที่สำคัญที่สุดเมื่อเขียน HTTP test ด้วย `httptest`:

1. **อย่าลืม `defer testServer.Close()`** ทุกครั้งที่ใช้ `httptest.NewServer()` — เหมือนกับการลืมปิดไฟล์หรือ database connection มันจะรั่วไหลสะสมไปเรื่อยๆ ตลอด test suite
2. **สร้าง state ใหม่ (`NewServer()`) ทุก test/subtest ที่จำเป็น** เพื่อป้องกัน test pollution เช่นเดียวกับหลักการที่เรียนใน **Part 080**
3. **`httptest.NewRequest` ไม่ตรวจสอบความถูกต้องของ URL** เท่า `http.NewRequest` จริง เพราะไม่ได้ตั้งใจให้ยิงออกไปไหน — ถ้าพิมพ์ path ผิดรูปแบบมากๆ อาจไม่ error ตอนสร้าง request แต่จะพลาดตอน handler พยายาม parse
4. **เลือก `NewRecorder` ก่อนเสมอถ้าเป็นไปได้** เพราะเร็วกว่า `NewServer` มาก (ไม่มี TCP overhead) ใช้ `NewServer` เฉพาะตอนทดสอบโค้ดฝั่ง client จริงๆ เท่านั้น
5. **ปิด `resp.Body` เสมอ** เมื่อทดสอบผ่าน `testServer.Client()` (เหมือนกับการเขียนโค้ด client จริงตามที่เรียนใน **Part 046**) ไม่งั้นจะรั่วไหล connection ในการทดสอบที่มี HTTP call จำนวนมาก
6. **ใช้ `httptest.NewTLSServer()` แทน `NewServer()`** เมื่อโค้ด client ต้องทดสอบการเชื่อมต่อผ่าน HTTPS — มันจะสร้าง TLS certificate ชั่วคราวให้อัตโนมัติ และ `testServer.Client()` จะถูกตั้งค่าให้เชื่อถือ certificate นั้นโดยไม่ต้องตั้งค่า TLS เองเลย

---

## สรุปสิ่งที่ได้เรียนในบทนี้

- **`httptest.NewRecorder()`** จำลอง `http.ResponseWriter` ในหน่วยความจำ ใช้ทดสอบ **Handler** โดยตรง ไม่มี network จริงเกิดขึ้นเลย รันเร็วเทียบเท่า unit test ปกติ
- **`httptest.NewServer()`** เปิด HTTP server จริงบน `localhost` พอร์ตสุ่ม ใช้ทดสอบ **Client** ที่ต้องยิง request ผ่าน `http.Client` จริง — คู่กับ `testServer.Client()` และ `testServer.URL`
- **Table-driven handler test** รวม pattern จาก **Part 034** เข้ากับ `httptest.NewRecorder` ได้อย่างลงตัว: วนลูปสร้าง request/response ตามตาราง แล้วตรวจสอบ status code และ body
- **ทดสอบ middleware** (ทบทวน **Part 057**) ด้วยการห่อ "dummy handler" ง่ายๆ แทนที่จะใช้ handler จริง เพื่อแยก concern ของ middleware ออกจาก business logic โดยสมบูรณ์ — และควรทดสอบการซ้อน middleware หลายชั้นด้วยเมื่อโปรเจกต์จริงมีการ chain
- **ทดสอบ JSON API แบบ end-to-end** ผ่าน `httptest.NewServer` ครอบคลุมทั้งการเข้ารหัส request, ยิงผ่าน network จริง (loopback), และถอดรหัส response กลับมาตรวจสอบทั้ง status และ payload
- ต้องปิด `testServer.Close()` และ `resp.Body.Close()` เสมอเพื่อป้องกัน resource รั่วไหลระหว่าง test suite ขนาดใหญ่

## แบบฝึกหัดท้ายบท

1. เพิ่ม endpoint `DELETE /users/{id}` ใน `Server` แล้วเขียน table-driven test ด้วย `httptest.NewRecorder()` ครอบคลุมกรณีลบสำเร็จและลบ user ที่ไม่มีอยู่จริง
2. เขียน middleware `WithLogging(logger *log.Logger, next http.Handler) http.Handler` ที่บันทึก method, path, และ status code ของทุก request (ทบทวน `log` package จาก **Part 054**) แล้วเขียน test ยืนยันว่า log message มีข้อมูลครบตามที่คาดหวัง (คำใบ้: ใช้ `bytes.Buffer` เป็นปลายทางของ `log.Logger` แทนการเขียนลง stdout)
3. เขียนฟังก์ชัน client `CreateUserViaAPI(client *http.Client, baseURL, name, email string) (*UserResponse, error)` ที่ห่อการยิง POST request แล้วเขียน test ด้วย `httptest.NewServer()` ครอบคลุมทั้งกรณีสำเร็จและกรณี server ตอบ error
4. ลองแก้ `TestCreateUser_TableDriven` ให้ใช้ `Server` ตัวเดียวร่วมกันทุก subtest (ไม่สร้างใหม่) แล้วสังเกตว่ามี subtest ไหนเริ่ม fail หรือให้ผลลัพธ์ไม่คาดคิดหรือไม่ อธิบายว่าทำไม (เชื่อมโยงกับแนวคิด test isolation)
5. ทดลองใช้ `httptest.NewTLSServer()` แทน `httptest.NewServer()` ในตัวอย่างหัวข้อ 6 แล้วสังเกตว่าต้องแก้โค้ด test อะไรเพิ่มเติมหรือไม่ (คำใบ้: ลองไม่ใช้ `testServer.Client()` แล้วดูว่าเกิด error อะไร)
6. เขียน middleware `RecoverPanic(next http.Handler) http.Handler` ที่ดักจับ panic จาก handler ปลายทางด้วย `recover()` (ทบทวน **Part 017**) แล้วตอบ `500 Internal Server Error` แทนที่จะปล่อยให้ทั้ง process ล่ม เขียน test ที่ยืนยันพฤติกรรมนี้โดยใช้ dummy handler ที่ตั้งใจ `panic(...)`

---

**ต่อไป**: [Part 082 — Test Coverage](./082-test-coverage.md)
