# Part 060: Request/Response และ JSON API

> ภาคที่ 5: Web Development — ตอนที่ 5 จาก 15 (Part 56–70)

## สารบัญของบทนี้

1. ภาพรวม: รวมทุกอย่างจาก Part 056-059 เป็นชั้น JSON API เดียว
2. `respondJSON` / `respondError`: helper คู่ที่ใช้ซ้ำทุก handler
3. ตรวจสอบ Content-Type ก่อนแตะ body (Content Negotiation เบื้องต้น)
4. Decode Request Body อย่างปลอดภัย และจับ Error ให้อ่านง่าย
5. Validate แบบ Manual: เช็ค Required Field ด้วยมือ
6. Validate ด้วย Struct Tag ผ่าน `go-playground/validator`
7. แปลง Validation Error เป็น Field-level Message
8. ตัวอย่างเต็มรูปแบบ: Todo API ที่รวมทุกอย่างเข้าด้วยกัน
9. ทดสอบทุก Path (สำเร็จและ Error) ด้วย `curl`
10. แนวทางปฏิบัติที่ดีและข้อผิดพลาดที่พบบ่อย
11. สรุปสิ่งที่ได้เรียนในบทนี้
12. แบบฝึกหัดท้ายบท

---

## 1. ภาพรวม: รวมทุกอย่างจาก Part 056-059 เป็นชั้น JSON API เดียว

ตลอด 4 บทที่ผ่านมาเราเรียนแยกส่วนกัน: routing และ handler (**Part 056**), middleware (**Part 057**), router ทางเลือก (**Part 058**), และหลักการออกแบบ REST (**Part 059**) บทนี้คือจุดที่ทุกอย่างมาบรรจบกัน — เราจะสร้าง **"ชั้น JSON API" (JSON API layer)** ที่เป็นมาตรฐานเดียวกันสำหรับทุก handler ในแอป ไม่ว่า handler นั้นจะจัดการ resource อะไรก็ตาม

ชั้นนี้ประกอบด้วย 3 ส่วนหลัก:

1. **ฝั่งตอบกลับ (response)**: helper ที่รับประกันว่าทุก response (สำเร็จหรือ error) มีรูปแบบเดียวกัน ตาม error envelope ที่ออกแบบไว้ใน Part 059
2. **ฝั่งรับข้อมูล (request)**: การ decode JSON ที่จับ error ได้ครบทุกแบบ พร้อมข้อความที่มีประโยชน์
3. **การ validate**: ทั้งแบบ manual (เช็คเอง) และแบบใช้ library ภายนอกที่เป็นมาตรฐานอุตสาหกรรม

---

## 2. `respondJSON` / `respondError`: helper คู่ที่ใช้ซ้ำทุก handler

ปัญหาที่พบบ่อยเมื่อ API โตขึ้น: handler แต่ละตัวเขียน `w.Header().Set(...)`, `w.WriteHeader(...)`, `json.NewEncoder(w).Encode(...)` ซ้ำๆ กันเอง ทำให้เผลอลืม header หรือลืมเช็ค encode error ได้ง่าย แนวทางที่ดีคือรวมทุกอย่างไว้ใน helper คู่เดียวที่ใช้ทุก handler เรียก:

```go
func respondJSON(w http.ResponseWriter, status int, v any) {
	w.Header().Set("Content-Type", "application/json; charset=utf-8")
	w.WriteHeader(status)
	if v == nil {
		return
	}
	if err := json.NewEncoder(w).Encode(v); err != nil {
		// ถึงจุดนี้ header ถูกส่งไปแล้ว ทำได้แค่ log เอาไว้เพราะแก้ response ไม่ได้อีก
		log.Printf("respondJSON: encode error: %v", err)
	}
}

func respondError(w http.ResponseWriter, status int, message string, fields map[string]string) {
	respondJSON(w, status, ErrorResponse{Error: ErrorDetail{Message: message, Fields: fields}})
}
```

โดยที่ `ErrorResponse`/`ErrorDetail` คือ error envelope ที่ออกแบบไว้ใน **Part 059**:

```go
type ErrorDetail struct {
	Message string            `json:"message"`
	Fields  map[string]string `json:"fields,omitempty"`
}

type ErrorResponse struct {
	Error ErrorDetail `json:"error"`
}
```

ข้อสังเกตสำคัญ: การเรียก `w.WriteHeader(status)` ต้องเกิด**ก่อน** `Encode` เสมอ เพราะเมื่อเริ่มเขียน body ไปแล้ว (ผ่าน `Write` หรือ `Encode`) จะเปลี่ยน status code ย้อนหลังไม่ได้อีก (Go จะ log warning ถ้าพยายามเรียก `WriteHeader` ซ้ำ — ปัญหาเดียวกับที่เจอตอนเขียน middleware ใน Part 057)

จากนี้ไป **ทุก handler ในบทนี้จะเรียกแค่ `respondJSON`/`respondError` เท่านั้น ไม่แตะ `http.ResponseWriter` ตรงๆ อีกเลย** — นี่คือการรับประกันความสม่ำเสมอของ response ทั้งแอป

---

## 3. ตรวจสอบ Content-Type ก่อนแตะ body (Content Negotiation เบื้องต้น)

**Content negotiation** ในความหมายเต็มรูปแบบคือการที่ client กับ server ตกลงกันว่าจะส่งข้อมูลกันในรูปแบบไหน (ผ่าน header `Accept`/`Content-Type`) เพราะบทนี้ใช้ JSON เป็นรูปแบบเดียวตลอดทั้ง API เราจึงทำแค่ส่วนที่จำเป็นที่สุด: **ปฏิเสธ request ที่ไม่ได้ส่ง `Content-Type: application/json` มาตั้งแต่แรก** ก่อนที่จะเสียเวลาพยายาม decode body ที่อาจไม่ใช่ JSON เลย

```go
func requireJSONContentType(next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		if r.Method == http.MethodPost || r.Method == http.MethodPut || r.Method == http.MethodPatch {
			ct := r.Header.Get("Content-Type")
			if !strings.HasPrefix(ct, "application/json") {
				respondError(w, http.StatusUnsupportedMediaType,
					"Content-Type must be application/json", nil)
				return
			}
		}
		next.ServeHTTP(w, r)
	})
}
```

สังเกตว่านี่คือ middleware ตัวหนึ่ง — เขียนในรูปแบบ `func(http.Handler) http.Handler` เหมือนทุกตัวใน **Part 057** เช็คเฉพาะ method ที่มี body (`POST`, `PUT`, `PATCH`) เพราะ `GET`/`DELETE` ปกติไม่มี body ให้ตรวจสอบ และใช้ `strings.HasPrefix` แทนการเทียบเท่ากันตรงๆ เพราะ header จริงมักมี charset ต่อท้าย เช่น `application/json; charset=utf-8`

status code ที่ใช้คือ **415 Unsupported Media Type** ซึ่งเป็น status ที่ถูกต้องตามความหมายของ HTTP spec สำหรับกรณีที่ server ไม่รองรับรูปแบบข้อมูลที่ client ส่งมา (ต่างจาก 400 ที่หมายถึงข้อมูลผิดรูปแบบ)

---

## 4. Decode Request Body อย่างปลอดภัย และจับ Error ให้อ่านง่าย

**Part 025** สอนพื้นฐานของ `encoding/json` ไว้แล้ว บทนี้จะเจาะลึกกรณี error ที่เกิดขึ้นได้ตอน decode และแปลงให้เป็นข้อความที่มีประโยชน์สำหรับ client แทนที่จะโยน error ดิบของ Go กลับไปตรงๆ

```go
// decodeJSON อ่าน body เป็น JSON ลงใน dst โดยจับ error หลายแบบที่
// encoding/json คืนมา แล้วแปลงเป็นข้อความที่ client เอาไปแสดงต่อ user ได้จริง
func decodeJSON(r *http.Request, dst any) error {
	if r.Body == nil {
		return errors.New("request body is empty")
	}
	dec := json.NewDecoder(r.Body)
	dec.DisallowUnknownFields() // เจอ field แปลกปลอมใน JSON ให้ error ทันที

	err := dec.Decode(dst)
	if err == nil {
		return nil
	}

	var syntaxErr *json.SyntaxError
	var typeErr *json.UnmarshalTypeError
	var unmarshalErr *json.InvalidUnmarshalError

	switch {
	case errors.Is(err, io.EOF):
		return errors.New("request body must not be empty")
	case errors.Is(err, io.ErrUnexpectedEOF):
		return errors.New("malformed json: unexpected end of input")
	case errors.As(err, &syntaxErr):
		return fmt.Errorf("malformed json at byte offset %d", syntaxErr.Offset)
	case errors.As(err, &typeErr):
		return fmt.Errorf("invalid value for field %q: expected type %s", typeErr.Field, typeErr.Type)
	case errors.As(err, &unmarshalErr):
		return errors.New("json decode target must be a non-nil pointer")
	case strings.Contains(err.Error(), "unknown field"):
		return fmt.Errorf("request body contains an unknown field: %s", strings.TrimPrefix(err.Error(), "json: unknown field "))
	default:
		return err
	}
}
```

แกะทีละ error case ที่ครอบคลุม (ใช้ `errors.Is`/`errors.As` ตามหลักการจาก **Part 016**):

- **`io.EOF`** — body ว่างเปล่าตั้งแต่แรก (ไม่มีข้อมูลส่งมาเลย)
- **`io.ErrUnexpectedEOF`** — JSON เขียนมาไม่ครบ (เช่น `{"title":` ที่ถูกตัดกลางคัน)
- **`json.SyntaxError`** — โครงสร้าง JSON ผิดไวยากรณ์ (เช่นลืมใส่ quote รอบ key) `Offset` บอกตำแหน่ง byte ที่เจอปัญหา ช่วย debug ได้ตรงจุด
- **`json.UnmarshalTypeError`** — ค่าที่ส่งมาผิด type จาก struct ที่กำหนดไว้ (เช่นส่ง `"title": 123` ทั้งที่ struct กำหนดเป็น `string`) — บอกชื่อ field และ type ที่คาดหวังไว้ตรงๆ
- **`json.InvalidUnmarshalError`** — เกิดจากโค้ดฝั่งเราเองส่ง pointer ที่เป็น `nil` หรือไม่ใช่ pointer ให้ `Decode` (ไม่ใช่ความผิดของ client แต่เป็น safety net กันบั๊กของโปรแกรมเมอร์)
- **`unknown field`** — ใช้ `dec.DisallowUnknownFields()` แล้วเจอ key ที่ไม่มีอยู่ใน struct เลย ช่วยจับ typo ของ client ได้ตั้งแต่ต้น (เช่น client พิมพ์ `"titel"` แทน `"title"`)

การจับ error ครบทุกแบบแบบนี้ทำให้ client (หรือทีม frontend/mobile ที่เรียก API) ได้ข้อความ error ที่**อธิบายปัญหาจริง** แทนที่จะเจอแค่ "bad request" ลอยๆ ซึ่งลด รอบการถาม-ตอบระหว่างทีมได้มาก

---

## 5. Validate แบบ Manual: เช็ค Required Field ด้วยมือ

ก่อนจะพึ่ง library ภายนอก ควรเข้าใจก่อนว่าการ validate ด้วยมือทำได้อย่างไร เพราะบาง case ง่ายๆ ไม่จำเป็นต้องพึ่ง library เลย:

```go
type createTodoInputManual struct {
	Title    string `json:"title"`
	Priority int    `json:"priority"`
}

func validateCreateTodoManual(in createTodoInputManual) map[string]string {
	fields := map[string]string{}
	if strings.TrimSpace(in.Title) == "" {
		fields["title"] = "field is required"
	}
	if len(in.Title) > 200 {
		fields["title"] = "must be at most 200 characters"
	}
	if in.Priority != 0 && (in.Priority < 1 || in.Priority > 5) {
		fields["priority"] = "must be between 1 and 5"
	}
	return fields
}
```

วิธีนี้ตรงไปตรงมา อ่านเข้าใจง่ายทันทีโดยไม่ต้องรู้จัก syntax พิเศษใดๆ แต่มีข้อเสียชัดเจนเมื่อ struct มี field เยอะขึ้น: โค้ด validate จะยาวขึ้นเรื่อยๆ และซ้ำซากกับ struct ที่มี pattern คล้ายกัน (เช่น "ต้องไม่ว่าง", "ต้องอยู่ในช่วง") — นี่คือจุดที่ library ภายนอกเข้ามาช่วยได้มาก

---

## 6. Validate ด้วย Struct Tag ผ่าน `go-playground/validator`

[`go-playground/validator`](https://github.com/go-playground/validator) เป็น library validate struct ที่ได้รับความนิยมที่สุดใน ecosystem ของ Go ใช้แนวคิด **declarative validation** — ประกาศกฎผ่าน struct tag แล้วให้ library ตรวจสอบให้อัตโนมัติ แทนการเขียน `if` ทีละเงื่อนไขเอง

ติดตั้ง:

```bash
go get github.com/go-playground/validator/v10
```

ประกาศกฎผ่าน tag `validate`:

```go
// createTodoInput คือ input struct เฉพาะตอนสร้าง มี struct tag `validate:"..."`
// จาก go-playground/validator — ไลบรารีข้างนอกที่นิยมใช้กันมากที่สุดสำหรับ
// validate struct ด้วย tag แบบประกาศ (declarative) แทนการเช็คทีละ field เอง
type createTodoInput struct {
	Title    string `json:"title" validate:"required,min=1,max=200"`
	Priority int    `json:"priority" validate:"omitempty,min=1,max=5"`
}
```

เรียกใช้งาน:

```go
validate := validator.New()

var in createTodoInput
// (สมมติว่า decode และเช็ค content-type ผ่านมาแล้ว)
if err := validate.Struct(in); err != nil {
	// err เป็น validator.ValidationErrors — ดูหัวข้อถัดไปว่าแปลงให้อ่านง่ายยังไง
}
```

tag ที่ใช้บ่อยที่สุดของ `validator`:

| Tag | ความหมาย |
|---|---|
| `required` | ต้องมีค่า (ไม่ใช่ zero value ของ type นั้น) |
| `omitempty` | ถ้าเป็น zero value ให้ข้ามการ validate เงื่อนไขถัดไป |
| `min=N` / `max=N` | ค่าต่ำสุด/สูงสุด (ใช้ได้ทั้งตัวเลขและความยาว string) |
| `email` | ต้องเป็นรูปแบบ email ที่ถูกต้อง |
| `url` | ต้องเป็นรูปแบบ URL ที่ถูกต้อง |
| `oneof=a b c` | ค่าต้องเป็นหนึ่งในตัวเลือกที่กำหนด |
| `gt=N` / `lt=N` | มากกว่า/น้อยกว่าค่าที่กำหนด (ไม่รวมค่าเท่ากับ) |
| `dive` | ใช้กับ slice/map เพื่อ validate สมาชิกแต่ละตัวข้างในด้วย |

ข้อดีของแนวทางนี้คือ **กฎการ validate อยู่ติดกับ struct definition โดยตรง** อ่านครั้งเดียวเห็นครบทุกเงื่อนไขของแต่ละ field ไม่ต้องไล่หา logic แยกอีกไฟล์

---

## 7. แปลง Validation Error เป็น Field-level Message

`validator.Struct()` เมื่อ error จะคืน type `validator.ValidationErrors` ซึ่งเป็น slice ของ error รายตัวสำหรับแต่ละ field ที่ไม่ผ่าน เราแปลงมันให้เข้ากับ `ErrorResponse.Fields` ที่ออกแบบไว้ใน Part 059 ได้ดังนี้:

```go
// validationFields แปลง validator.ValidationErrors ให้เป็น map[field]message
// ที่อ่านง่าย สำหรับใส่ใน ErrorResponse.Fields
func (h *TodoHandler) validationFields(err error) map[string]string {
	fields := map[string]string{}
	var verrs validator.ValidationErrors
	if errors.As(err, &verrs) {
		for _, fe := range verrs {
			key := strings.ToLower(fe.Field())
			switch fe.Tag() {
			case "required":
				fields[key] = "field is required"
			case "min":
				fields[key] = fmt.Sprintf("must be at least %s", fe.Param())
			case "max":
				fields[key] = fmt.Sprintf("must be at most %s", fe.Param())
			default:
				fields[key] = fmt.Sprintf("failed on %q validation", fe.Tag())
			}
		}
	}
	return fields
}
```

`fe.Field()` คืนชื่อ Go field (เช่น `"Title"`) — แปลงเป็นตัวพิมพ์เล็กด้วย `strings.ToLower` ให้ตรงกับชื่อ JSON key ทั่วไป, `fe.Tag()` คือชื่อกฎที่ไม่ผ่าน (`"required"`, `"min"`, ฯลฯ), และ `fe.Param()` คือค่าพารามิเตอร์ของกฎนั้น (เช่น ถ้า tag คือ `min=1` ค่า `Param()` จะเป็น `"1"`)

---

## 8. ตัวอย่างเต็มรูปแบบ: Todo API ที่รวมทุกอย่างเข้าด้วยกัน

รวมทุกอย่างจากหัวข้อ 2-7 เป็น API เดียวที่สมบูรณ์ — Todo API ที่มี `GET/POST/PATCH/DELETE` ครบ พร้อม validate และ error handling เต็มรูปแบบ:

```go
package main

import (
	"encoding/json"
	"errors"
	"fmt"
	"io"
	"log"
	"net/http"
	"strconv"
	"strings"
	"sync"
	"time"

	"github.com/go-playground/validator/v10"
)

type ErrorDetail struct {
	Message string            `json:"message"`
	Fields  map[string]string `json:"fields,omitempty"`
}

type ErrorResponse struct {
	Error ErrorDetail `json:"error"`
}

func respondJSON(w http.ResponseWriter, status int, v any) {
	w.Header().Set("Content-Type", "application/json; charset=utf-8")
	w.WriteHeader(status)
	if v == nil {
		return
	}
	if err := json.NewEncoder(w).Encode(v); err != nil {
		log.Printf("respondJSON: encode error: %v", err)
	}
}

func respondError(w http.ResponseWriter, status int, message string, fields map[string]string) {
	respondJSON(w, status, ErrorResponse{Error: ErrorDetail{Message: message, Fields: fields}})
}

func decodeJSON(r *http.Request, dst any) error {
	if r.Body == nil {
		return errors.New("request body is empty")
	}
	dec := json.NewDecoder(r.Body)
	dec.DisallowUnknownFields()

	err := dec.Decode(dst)
	if err == nil {
		return nil
	}

	var syntaxErr *json.SyntaxError
	var typeErr *json.UnmarshalTypeError
	var unmarshalErr *json.InvalidUnmarshalError

	switch {
	case errors.Is(err, io.EOF):
		return errors.New("request body must not be empty")
	case errors.Is(err, io.ErrUnexpectedEOF):
		return errors.New("malformed json: unexpected end of input")
	case errors.As(err, &syntaxErr):
		return fmt.Errorf("malformed json at byte offset %d", syntaxErr.Offset)
	case errors.As(err, &typeErr):
		return fmt.Errorf("invalid value for field %q: expected type %s", typeErr.Field, typeErr.Type)
	case errors.As(err, &unmarshalErr):
		return errors.New("json decode target must be a non-nil pointer")
	case strings.Contains(err.Error(), "unknown field"):
		return fmt.Errorf("request body contains an unknown field: %s", strings.TrimPrefix(err.Error(), "json: unknown field "))
	default:
		return err
	}
}

func requireJSONContentType(next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		if r.Method == http.MethodPost || r.Method == http.MethodPut || r.Method == http.MethodPatch {
			ct := r.Header.Get("Content-Type")
			if !strings.HasPrefix(ct, "application/json") {
				respondError(w, http.StatusUnsupportedMediaType,
					"Content-Type must be application/json", nil)
				return
			}
		}
		next.ServeHTTP(w, r)
	})
}

type Todo struct {
	ID        int       `json:"id"`
	Title     string    `json:"title"`
	Done      bool      `json:"done"`
	Priority  int       `json:"priority"`
	CreatedAt time.Time `json:"created_at"`
}

type createTodoInput struct {
	Title    string `json:"title" validate:"required,min=1,max=200"`
	Priority int    `json:"priority" validate:"omitempty,min=1,max=5"`
}

// updateTodoInput สำหรับแก้ไข: Done เป็น *bool เพื่อแยกแยะ "ไม่ได้ส่งมา"
// ออกจาก "ส่งมาเป็น false" ได้ (pointer trick ที่เจอได้ทั่วไปใน JSON API)
type updateTodoInput struct {
	Title    *string `json:"title" validate:"omitempty,min=1,max=200"`
	Done     *bool   `json:"done"`
	Priority *int    `json:"priority" validate:"omitempty,min=1,max=5"`
}

type TodoStore struct {
	mu     sync.Mutex
	todos  map[int]Todo
	nextID int
}

func NewTodoStore() *TodoStore {
	return &TodoStore{todos: make(map[int]Todo), nextID: 1}
}

var ErrTodoNotFound = errors.New("todo not found")

func (s *TodoStore) List() []Todo {
	s.mu.Lock()
	defer s.mu.Unlock()
	result := make([]Todo, 0, len(s.todos))
	for _, t := range s.todos {
		result = append(result, t)
	}
	return result
}

func (s *TodoStore) Get(id int) (Todo, error) {
	s.mu.Lock()
	defer s.mu.Unlock()
	t, ok := s.todos[id]
	if !ok {
		return Todo{}, ErrTodoNotFound
	}
	return t, nil
}

func (s *TodoStore) Create(in createTodoInput) Todo {
	s.mu.Lock()
	defer s.mu.Unlock()
	priority := in.Priority
	if priority == 0 {
		priority = 3
	}
	t := Todo{ID: s.nextID, Title: in.Title, Priority: priority, CreatedAt: time.Now()}
	s.todos[t.ID] = t
	s.nextID++
	return t
}

func (s *TodoStore) Update(id int, in updateTodoInput) (Todo, error) {
	s.mu.Lock()
	defer s.mu.Unlock()
	t, ok := s.todos[id]
	if !ok {
		return Todo{}, ErrTodoNotFound
	}
	if in.Title != nil {
		t.Title = *in.Title
	}
	if in.Done != nil {
		t.Done = *in.Done
	}
	if in.Priority != nil {
		t.Priority = *in.Priority
	}
	s.todos[id] = t
	return t, nil
}

func (s *TodoStore) Delete(id int) error {
	s.mu.Lock()
	defer s.mu.Unlock()
	if _, ok := s.todos[id]; !ok {
		return ErrTodoNotFound
	}
	delete(s.todos, id)
	return nil
}

type TodoHandler struct {
	store    *TodoStore
	validate *validator.Validate
}

func (h *TodoHandler) validationFields(err error) map[string]string {
	fields := map[string]string{}
	var verrs validator.ValidationErrors
	if errors.As(err, &verrs) {
		for _, fe := range verrs {
			key := strings.ToLower(fe.Field())
			switch fe.Tag() {
			case "required":
				fields[key] = "field is required"
			case "min":
				fields[key] = fmt.Sprintf("must be at least %s", fe.Param())
			case "max":
				fields[key] = fmt.Sprintf("must be at most %s", fe.Param())
			default:
				fields[key] = fmt.Sprintf("failed on %q validation", fe.Tag())
			}
		}
	}
	return fields
}

func (h *TodoHandler) list(w http.ResponseWriter, r *http.Request) {
	respondJSON(w, http.StatusOK, h.store.List())
}

func (h *TodoHandler) create(w http.ResponseWriter, r *http.Request) {
	var in createTodoInput
	if err := decodeJSON(r, &in); err != nil {
		respondError(w, http.StatusBadRequest, err.Error(), nil)
		return
	}
	if err := h.validate.Struct(in); err != nil {
		respondError(w, http.StatusUnprocessableEntity, "validation failed", h.validationFields(err))
		return
	}
	t := h.store.Create(in)
	respondJSON(w, http.StatusCreated, t)
}

func (h *TodoHandler) get(w http.ResponseWriter, r *http.Request) {
	id, err := strconv.Atoi(r.PathValue("id"))
	if err != nil {
		respondError(w, http.StatusBadRequest, "invalid id", nil)
		return
	}
	t, err := h.store.Get(id)
	if err != nil {
		respondError(w, http.StatusNotFound, "todo not found", nil)
		return
	}
	respondJSON(w, http.StatusOK, t)
}

func (h *TodoHandler) update(w http.ResponseWriter, r *http.Request) {
	id, err := strconv.Atoi(r.PathValue("id"))
	if err != nil {
		respondError(w, http.StatusBadRequest, "invalid id", nil)
		return
	}
	var in updateTodoInput
	if err := decodeJSON(r, &in); err != nil {
		respondError(w, http.StatusBadRequest, err.Error(), nil)
		return
	}
	if err := h.validate.Struct(in); err != nil {
		respondError(w, http.StatusUnprocessableEntity, "validation failed", h.validationFields(err))
		return
	}
	t, err := h.store.Update(id, in)
	if err != nil {
		respondError(w, http.StatusNotFound, "todo not found", nil)
		return
	}
	respondJSON(w, http.StatusOK, t)
}

func (h *TodoHandler) delete(w http.ResponseWriter, r *http.Request) {
	id, err := strconv.Atoi(r.PathValue("id"))
	if err != nil {
		respondError(w, http.StatusBadRequest, "invalid id", nil)
		return
	}
	if err := h.store.Delete(id); err != nil {
		respondError(w, http.StatusNotFound, "todo not found", nil)
		return
	}
	respondJSON(w, http.StatusNoContent, nil)
}

func main() {
	h := &TodoHandler{store: NewTodoStore(), validate: validator.New()}

	mux := http.NewServeMux()
	mux.HandleFunc("GET /todos", h.list)
	mux.HandleFunc("POST /todos", h.create)
	mux.HandleFunc("GET /todos/{id}", h.get)
	mux.HandleFunc("PATCH /todos/{id}", h.update)
	mux.HandleFunc("DELETE /todos/{id}", h.delete)

	handler := requireJSONContentType(mux)

	log.Println("listening on :8092")
	log.Fatal(http.ListenAndServe(":8092", handler))
}
```

---

## 9. ทดสอบทุก Path (สำเร็จและ Error) ด้วย `curl`

ผลลัพธ์จริงทั้งหมดจากการรันโค้ดข้างต้น:

```bash
curl -s http://localhost:8092/todos
```
```json
[]
```

```bash
curl -s -X POST http://localhost:8092/todos \
  -H "Content-Type: application/json" \
  -d '{"title":"Buy milk","priority":2}'
```
```json
{"id":1,"title":"Buy milk","done":false,"priority":2,"created_at":"2026-09-26T02:55:27.487503562Z"}
```

```bash
# ขาด title (422 — validation error)
curl -s -X POST http://localhost:8092/todos \
  -H "Content-Type: application/json" -d '{"priority":2}'
```
```
HTTP status: 422
{"error":{"message":"validation failed","fields":{"title":"field is required"}}}
```

```bash
# priority เกินขอบเขต (422)
curl -s -X POST http://localhost:8092/todos \
  -H "Content-Type: application/json" -d '{"title":"x","priority":9}'
```
```
HTTP status: 422
{"error":{"message":"validation failed","fields":{"priority":"must be at most 5"}}}
```

```bash
# JSON เขียนไม่ครบ (400)
curl -s -X POST http://localhost:8092/todos \
  -H "Content-Type: application/json" -d '{"title":'
```
```
HTTP status: 400
{"error":{"message":"malformed json: unexpected end of input"}}
```

```bash
# genuine syntax error: key ไม่มี quote (400)
curl -s -X POST http://localhost:8092/todos \
  -H "Content-Type: application/json" -d '{title: "x"}'
```
```
HTTP status: 400
{"error":{"message":"malformed json at byte offset 2"}}
```

```bash
# ส่ง type ผิด: title เป็นตัวเลขแทน string (400)
curl -s -X POST http://localhost:8092/todos \
  -H "Content-Type: application/json" -d '{"title":123}'
```
```
HTTP status: 400
{"error":{"message":"invalid value for field \"title\": expected type string"}}
```

```bash
# field แปลกปลอมที่ไม่มีใน struct (400)
curl -s -X POST http://localhost:8092/todos \
  -H "Content-Type: application/json" -d '{"title":"x","foo":"bar"}'
```
```
HTTP status: 400
{"error":{"message":"request body contains an unknown field: \"foo\""}}
```

```bash
# ไม่ได้ส่ง Content-Type มาเลย (415)
curl -s -X POST http://localhost:8092/todos -d '{"title":"x"}'
```
```
HTTP status: 415
{"error":{"message":"Content-Type must be application/json"}}
```

```bash
curl -s -X PATCH http://localhost:8092/todos/1 \
  -H "Content-Type: application/json" -d '{"done":true}'
```
```json
{"id":1,"title":"Buy milk","done":true,"priority":2,"created_at":"2026-09-26T02:55:27.487503562Z"}
```

```bash
curl -s -o /dev/null -w "%{http_code}\n" http://localhost:8092/todos/999
```
```
404
```

```bash
curl -s -o /dev/null -w "%{http_code}\n" -X DELETE http://localhost:8092/todos/1
```
```
204
```

ทุก error case ตอบด้วย envelope เดียวกัน (`{"error": {"message": ..., "fields": ...}}`) และทุก status code ตรงกับความหมายที่ออกแบบไว้ใน **Part 059** — นี่คือผลลัพธ์ที่ได้จากการรวมทุกเทคนิคในบทนี้เข้าด้วยกัน

---

## 10. แนวทางปฏิบัติที่ดีและข้อผิดพลาดที่พบบ่อย

- **แยก input struct จาก domain struct เสมอ** — `createTodoInput`/`updateTodoInput` ไม่ใช่ `Todo` ตรงๆ เพราะ field ที่รับตอนสร้าง/แก้ไขมักไม่เหมือนกับ field ที่ตอบกลับ (เช่น `ID`, `CreatedAt` ไม่ควรให้ client กำหนดเองได้)
- **ใช้ pointer สำหรับ field ที่ต้องแยก "ไม่ได้ส่งมา" ออกจาก "ส่งมาเป็นค่าว่าง"** — เห็นได้จาก `updateTodoInput.Done *bool` ถ้าใช้ `bool` ธรรมดา จะแยกไม่ออกว่า client ตั้งใจส่ง `false` มา หรือแค่ไม่ได้ส่ง field นี้มาเลย (ปัญหา zero value ที่เจอบ่อยมากตอนทำ `PATCH`)
- **`dec.DisallowUnknownFields()` ไม่ใช่ default** — ต้องเรียกเองเสมอถ้าต้องการให้ client รู้ทันทีว่าพิมพ์ชื่อ field ผิด ไม่เช่นนั้น `encoding/json` จะ "เงียบๆ ข้าม" field ที่ไม่รู้จักไปเฉยๆ โดยไม่แจ้งอะไรเลย
- **Validate หลัง decode เสมอ ไม่ใช่ก่อน** — ต้อง decode สำเร็จก่อน (ได้ struct ที่มี type ถูกต้อง) ถึงจะ validate ค่าข้างในได้อย่างมีความหมาย
- **อย่าใส่ raw error message จาก internal system ลงใน response โดยตรง** เช่น database connection string หรือ stack trace เพราะเป็นการรั่วไหลข้อมูลภายในที่อาจเป็นช่องโหว่ความปลอดภัย ควร log error เต็มไว้ฝั่ง server (ทบทวน **Part 054**) แล้วส่งแค่ข้อความสรุปที่ปลอดภัยกลับไปให้ client
- **`go-playground/validator` ไม่ validate ข้าม field โดย default** — ถ้าต้องการกฎแบบ "ถ้า field A มีค่า field B ต้องมีค่าด้วย" ต้องใช้ tag พิเศษเช่น `required_with=A` หรือเขียน custom validation function เพิ่มเอง

---

## สรุปสิ่งที่ได้เรียนในบทนี้

- `respondJSON`/`respondError` เป็น helper คู่ที่รับประกันว่าทุก response (สำเร็จหรือ error) มีรูปแบบเดียวกันทั้งแอป ใช้ error envelope ที่ออกแบบไว้ใน Part 059
- middleware `requireJSONContentType` ปฏิเสธ request ที่ไม่ได้ส่ง `Content-Type: application/json` มาตั้งแต่ต้น ด้วย status 415
- `decodeJSON` จับ error ทุกแบบที่ `encoding/json` คืนมา (`io.EOF`, `io.ErrUnexpectedEOF`, `json.SyntaxError`, `json.UnmarshalTypeError`, unknown field) แล้วแปลงเป็นข้อความที่มีประโยชน์
- Validate แบบ manual เหมาะกับกฎง่ายๆ ไม่กี่ field ส่วน `go-playground/validator` เหมาะกับ struct ที่มีกฎซับซ้อนหลาย field ผ่าน struct tag แบบ declarative
- `validator.ValidationErrors` แปลงเป็น field-level message ได้ผ่าน `fe.Field()`, `fe.Tag()`, `fe.Param()` แล้วใส่ใน `ErrorResponse.Fields`
- ใช้ pointer field (`*bool`, `*string`) ใน update struct เพื่อแยกแยะ "ไม่ได้ส่งมา" กับ "ส่งมาเป็นค่า zero value"
- ตัวอย่าง Todo API ที่รวมทุกเทคนิคเข้าด้วยกันแสดงให้เห็นว่า handler แต่ละตัวยังคงอ่านง่าย สั้น กระชับ แม้จะมี validation และ error handling ครบถ้วน

## แบบฝึกหัดท้ายบท

1. เพิ่ม custom validation ให้ `createTodoInput.Title` ห้ามขึ้นต้นด้วยช่องว่าง โดยเขียน custom validation function จดทะเบียนกับ `validate.RegisterValidation`
2. เพิ่ม endpoint `GET /todos?done=true` ที่กรองเฉพาะ todo ที่ทำเสร็จแล้ว โดยใช้แนวคิด query parameter จาก Part 059
3. เขียนเทสสำหรับ `decodeJSON` โดยใช้ `httptest.NewRequest` ป้อน body หลายแบบ (JSON ถูกต้อง, syntax error, type ผิด, unknown field) แล้วตรวจสอบว่า error message ตรงกับที่ออกแบบไว้ทุกกรณี
4. ปรับ `respondError` ให้รับ error code แบบ string เพิ่มเข้าไปใน `ErrorDetail` (เช่น `"TODO_NOT_FOUND"`) เพื่อให้ client แยกแยะ error โดยไม่ต้อง parse ข้อความภาษาธรรมชาติ
5. ทดลองสร้าง `updateTodoInput` เวอร์ชันที่ไม่ใช้ pointer แล้วลองอธิบาย (หรือเขียนเทสพิสูจน์) ว่าเกิดปัญหาอะไรขึ้นเมื่อ client ส่ง `{"done": false}` มาเพื่อตั้งใจปิด todo ที่เคยเสร็จแล้ว
6. รวม Todo API นี้เข้ากับ `chi` router จาก **Part 058** แทน stdlib `ServeMux` โดยยังคงพฤติกรรม error handling ทั้งหมดไว้เหมือนเดิม

---

**ต่อไป**: [Part 061 — Web Framework: Gin เบื้องต้น](./061-gin-basics.md)
