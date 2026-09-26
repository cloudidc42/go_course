# Part 016: Error Handling ขั้นสูง — Custom Error, Wrapping, `errors.Is`/`As`/`Join`

> ภาคที่ 2: ระดับกลาง (Intermediate) — ตอนที่ 1 จาก 20 (Part 16–35)

## สารบัญของบทนี้

1. ทบทวนสิ่งที่รู้แล้วจาก Part 015 และข้อจำกัดที่ต้องแก้
2. Custom Error Types: สร้างชนิดข้อมูลของตัวเองที่ implement `error`
3. Sentinel Errors: เมื่อไหร่ควรประกาศ error ระดับ package
4. Error Wrapping ด้วย `fmt.Errorf` และ `%w`
5. `errors.Unwrap`: กลไกเบื้องหลังการห่อ error
6. `errors.Is`: เทียบ sentinel error ผ่าน chain ที่ถูกห่อ
7. `errors.As`: ดึง custom error type ออกจาก chain
8. `errors.Join` (Go 1.20+): รวมหลาย error เข้าด้วยกัน
9. การออกแบบ error message ที่ดี
10. รูปแบบที่พบบ่อยและข้อผิดพลาดที่ควรเลี่ยง
11. สรุปสิ่งที่ได้เรียนในบทนี้
12. แบบฝึกหัดท้ายบท

---

## 1. ทบทวนสิ่งที่รู้แล้วจาก Part 015 และข้อจำกัดที่ต้องแก้

ใน Part 015 เราเรียนรู้รากฐานของการจัดการ error ใน Go ไปแล้ว:

- `error` เป็นเพียง **interface** ที่มี method เดียวคือ `Error() string`
- สร้าง error พื้นฐานด้วย `errors.New("ข้อความ")` หรือ `fmt.Errorf("ข้อความ")`
- รูปแบบมาตรฐานของการส่งต่อ error คือ `if err != nil { return err }`
- Go มี sentinel error สำเร็จรูปให้ใช้ เช่น `io.EOF` ที่เทียบด้วย `==` ตรงๆ ได้

รูปแบบพื้นฐานเหล่านี้เพียงพอสำหรับโปรแกรมเล็กๆ แต่เมื่อระบบใหญ่ขึ้น จะเจอปัญหาสามข้อที่ error แบบพื้นฐานตอบโจทย์ไม่ได้:

**ปัญหาที่ 1 — error ไม่มีข้อมูลประกอบ**: ถ้า validation ล้มเหลว เราอยากรู้ว่า field ไหนผิด ค่าอะไรที่ส่งมา ไม่ใช่แค่ข้อความ string เฉยๆ

**ปัญหาที่ 2 — เมื่อ error ถูกห่อหลายชั้น (เพราะแต่ละ layer เติม context เข้าไป) เราจะเช็คได้อย่างไรว่า "ราก" ของ error คือ error ตัวไหน?** ถ้าใช้ `==` ตรงๆ กับ error ที่ผ่านการห่อด้วย `fmt.Errorf` มาแล้ว จะเทียบไม่ตรงอีกต่อไป เพราะ string ข้อความเปลี่ยนไป

**ปัญหาที่ 3 — เมื่อมี error หลายตัวเกิดพร้อมกัน** (เช่น validate form ที่มีหลาย field ผิดพร้อมกัน หรืองาน cleanup หลายอย่างที่ fail พร้อมกัน) เราจะรวม error เหล่านั้นเป็นค่าเดียวโดยไม่ทำข้อมูลหายได้อย่างไร?

บทนี้จะแก้ทั้งสามปัญหาด้วยเครื่องมือจาก package `errors` และ `fmt` ที่ Go มีให้ในตัว โดยไม่ต้องพึ่ง library ภายนอกเลย

---

## 2. Custom Error Types: สร้างชนิดข้อมูลของตัวเองที่ implement `error`

จาก Part 013 เราเรียนรู้แล้วว่า **interface ใน Go ถูก implement แบบ implicit** — แค่ struct มี method ที่ตรง signature ก็ถือว่า implement interface นั้นโดยอัตโนมัติ `error` ก็เป็น interface ธรรมดาที่ประกาศไว้ใน package `builtin`:

```go
type error interface {
	Error() string
}
```

นั่นแปลว่า **struct ใดๆ ก็ตามที่มี method `Error() string` ก็คือ `error` ได้ทันที** เราจึงสามารถออกแบบ struct ที่เก็บข้อมูลประกอบเพิ่มเติมได้ตามต้องการ ไม่ได้ถูกจำกัดอยู่แค่ข้อความ string เดียวเหมือน `errors.New`

### ตัวอย่าง: `ValidationError` ที่เก็บข้อมูล field

```go
package main

import (
	"errors"
	"fmt"
)

type ValidationError struct {
	Field string
	Value interface{}
	Msg   string
}

func (e *ValidationError) Error() string {
	return fmt.Sprintf("validation failed on field %q (value: %v): %s", e.Field, e.Value, e.Msg)
}

func validateAge(age int) error {
	if age < 0 || age > 150 {
		return &ValidationError{Field: "age", Value: age, Msg: "must be between 0 and 150"}
	}
	return nil
}

func main() {
	err := validateAge(-5)
	if err != nil {
		fmt.Println("เกิดข้อผิดพลาด:", err)
	}

	var ve *ValidationError
	if errors.As(err, &ve) {
		fmt.Println("field ที่ผิดพลาด:", ve.Field)
		fmt.Println("ค่าที่ส่งมา:", ve.Value)
	}
}
```

ผลลัพธ์:

```
เกิดข้อผิดพลาด: validation failed on field "age" (value: -5): must be between 0 and 150
field ที่ผิดพลาด: age
ค่าที่ส่งมา: -5
```

จุดสำคัญที่ต้องสังเกต:

1. **Method `Error()` รับ pointer receiver (`*ValidationError`)** ไม่ใช่ value receiver — นี่คือธรรมเนียมมาตรฐานสำหรับ custom error type เกือบทั้งหมดในวงการ Go เหตุผลคือ (ก) เลี่ยงการ copy struct ที่อาจมีขนาดใหญ่โดยไม่จำเป็นทุกครั้งที่ error ถูกส่งต่อ และ (ข) ทำให้ `errors.Is`/`errors.As` เทียบ identity ของ pointer ได้ตรงไปตรงมากว่า เราจึง `return &ValidationError{...}` ไม่ใช่ `return ValidationError{...}`
2. **`fmt.Sprintf` ใช้สร้างข้อความ error ที่อ่านง่าย** โดยฝัง field ทั้งหมดเข้าไปในข้อความด้วย
3. เราได้ใช้ `errors.As` แล้วในตัวอย่างนี้ (จะอธิบายละเอียดในหัวข้อที่ 6) เพื่อ**ดึง struct ต้นฉบับกลับออกมา** จากค่า `error` (interface) — สิ่งที่ทำไม่ได้เลยถ้าใช้ `errors.New` เพราะ `errors.New` คืน error ชนิดพื้นฐานที่ไม่มี field ให้ดึง

### เมื่อไหร่ควรสร้าง custom error type

ไม่ใช่ทุก error ต้องมี custom type — ถ้าแค่ต้องการบอกว่า "เกิดอะไรขึ้น" เป็นข้อความ ใช้ `errors.New`/`fmt.Errorf` ก็เพียงพอ แต่ควรสร้าง custom error type เมื่อ:

- **Caller ต้องการข้อมูลเชิงโครงสร้างเพื่อนำไปใช้ต่อ** เช่น HTTP handler ต้องรู้ status code ที่ควรตอบกลับจาก error, หรือ retry logic ต้องรู้ว่า error นี้ retry ได้ไหม
- **ต้องการแยกแยะ error หลายแบบที่มาจากจุดเดียวกัน** ผ่าน type switch หรือ `errors.As` แทนที่จะเทียบข้อความ string (ซึ่งเปราะบางมาก เพราะข้อความอาจเปลี่ยนได้ทุกเมื่อ)
- **ต้องการแนบ error ต้นตอ (underlying cause)** ไว้ในโครงสร้างเดียวกัน (ดูหัวข้อ Wrapping ด้านล่าง)

### ตัวอย่าง: custom error ที่บอกได้ว่า retry ได้หรือไม่

```go
type APIError struct {
	StatusCode int
	Message    string
	Retryable  bool
}

func (e *APIError) Error() string {
	return fmt.Sprintf("API error %d: %s", e.StatusCode, e.Message)
}
```

Caller สามารถเขียนโค้ดแบบนี้ได้ทันที:

```go
var apiErr *APIError
if errors.As(err, &apiErr) && apiErr.Retryable {
	// สั่ง retry ได้อย่างมั่นใจ เพราะรู้ชัดเจนจากข้อมูลใน error เอง
}
```

นี่คือพลังของ custom error type — เปลี่ยนจาก "เดาความหมายจากข้อความ" เป็น "อ่านข้อมูลจากโครงสร้างที่ชัดเจน"

---

## 3. Sentinel Errors: เมื่อไหร่ควรประกาศ error ระดับ package

**Sentinel error** คือ error ค่าคงที่ที่ประกาศไว้เป็นตัวแปรระดับ package (package-level variable) เพื่อให้ทั้ง package ตัวเองและ caller ภายนอกใช้เทียบค่าได้ ตัวอย่างที่เราคุ้นเคยแล้วจาก standard library คือ `io.EOF`:

```go
var EOF = errors.New("EOF")
```

รูปแบบการประกาศ sentinel error ของเราเองก็เขียนเหมือนกันทุกประการ:

```go
package store

import "errors"

var ErrNotFound = errors.New("record not found")
var ErrPermissionDenied = errors.New("permission denied")
```

### กฎการตั้งชื่อ

ชุมชน Go มีธรรมเนียมตั้งชื่อ sentinel error ที่ยึดถือกันแทบทุกโปรเจกต์:

- ขึ้นต้นด้วย `Err` เสมอ (เช่น `ErrNotFound`, `ErrTimeout`, `ErrInvalidInput`)
- ถ้าต้องการ export ให้ package อื่น import ไปใช้ ต้องขึ้นต้นด้วยตัวใหญ่ตามกฎ Part 001 (`ErrNotFound` ไม่ใช่ `errNotFound`)
- ถ้า error นั้นใช้ภายใน package เดียวเท่านั้น ให้ขึ้นต้นด้วยตัวเล็กได้ (`errInternal`)

### เมื่อไหร่ควรใช้ sentinel error

Sentinel error เหมาะกับกรณีที่:

1. **Caller ต้องการเช็คว่า "เกิด error ประเภทนี้หรือไม่"** โดยไม่สนใจรายละเอียดเพิ่มเติม เช่น "ไม่พบข้อมูลหรือเปล่า" (`ErrNotFound`) โดยไม่สนใจว่าค้นหาด้วย key อะไร
2. **Error นั้นไม่มีข้อมูลประกอบที่จำเป็นต้องแนบ** ถ้าต้องการแนบข้อมูล (เช่น field ที่ผิด, ID ที่หาไม่เจอ) ให้ใช้ custom error type ตามหัวข้อ 2 แทน หรือใช้ทั้งคู่ร่วมกัน (ดูหัวข้อถัดไป)
3. **เป็นสัญญา (contract) ระหว่าง package กับผู้ใช้งาน** — เมื่อ export sentinel error ออกไป ถือเป็นส่วนหนึ่งของ public API ที่ต้องรักษาความเข้ากันได้ย้อนหลัง (backward compatibility) เหมือน function signature

### ตัวอย่างการใช้งานร่วมกับ custom error type

รูปแบบที่ทรงพลังที่สุดในทางปฏิบัติคือ**ผสมทั้งสองแบบเข้าด้วยกัน**: ใช้ sentinel error เป็น "หมวดหมู่" และ custom error type เป็น "รายละเอียด"

```go
package store

import (
	"errors"
	"fmt"
)

var ErrNotFound = errors.New("record not found")

type NotFoundError struct {
	Collection string
	ID         string
}

func (e *NotFoundError) Error() string {
	return fmt.Sprintf("%s with id %q not found", e.Collection, e.ID)
}

// Unwrap ทำให้ errors.Is(err, ErrNotFound) ใช้งานได้
// แม้ err จะเป็น *NotFoundError ก็ตาม (อธิบายละเอียดในหัวข้อ 5)
func (e *NotFoundError) Unwrap() error {
	return ErrNotFound
}
```

ด้วยรูปแบบนี้ caller ที่สนใจแค่ "หาไม่เจอไหม" เขียน `errors.Is(err, store.ErrNotFound)` ได้ทันที ส่วน caller ที่อยากรู้รายละเอียดก็ `errors.As(err, &notFoundErr)` เพื่อดึง `Collection`/`ID` ออกมาได้ — ได้ประโยชน์ทั้งสองแบบพร้อมกัน

---

## 4. Error Wrapping ด้วย `fmt.Errorf` และ `%w`

ปัญหาที่พบบ่อยมากในระบบจริงคือ: error เกิดขึ้นที่ layer ล่างสุด (เช่น database) แล้วต้องส่งผ่าน layer หลายชั้นขึ้นมา (repository → service → handler) แต่ละ layer อยากจะ**เติม context** เข้าไปว่า "กำลังทำอะไรอยู่ตอนที่ error เกิด" โดยไม่ทำให้ error ต้นฉบับ (root cause) หายไป

ก่อน Go 1.13 วิธีเดียวที่ทำได้คือสร้าง error ข้อความใหม่ทับไปเลย ซึ่งทำให้ error ต้นฉบับ (เช่น `sql.ErrNoRows`) หายไปโดยสิ้นเชิง ไม่สามารถเช็คย้อนกลับได้อีก

Go 1.13 แก้ปัญหานี้ด้วย **verb พิเศษ `%w`** ใน `fmt.Errorf`:

```go
package main

import (
	"errors"
	"fmt"
)

var ErrNotFound = errors.New("record not found")

func findUserInDB(id int) error {
	if id != 1 {
		return ErrNotFound
	}
	return nil
}

func findUser(id int) error {
	if err := findUserInDB(id); err != nil {
		return fmt.Errorf("findUser(id=%d): %w", id, err)
	}
	return nil
}

func main() {
	err := findUser(42)
	fmt.Println("error message:", err)

	if errors.Is(err, ErrNotFound) {
		fmt.Println("นี่คือ ErrNotFound ที่ถูกห่อไว้")
	}

	fmt.Println("unwrap แล้วได้:", errors.Unwrap(err))
}
```

ผลลัพธ์:

```
error message: findUser(id=42): record not found
นี่คือ ErrNotFound ที่ถูกห่อไว้
unwrap แล้วได้: record not found
```

### `%w` ต่างจาก `%v` อย่างไร

ทั้งคู่แสดงข้อความเหมือนกันทุกประการเมื่อ `Println`/`Printf` — ความต่างอยู่ที่ **โครงสร้างข้อมูลภายใน**:

| Verb | ผลลัพธ์ข้อความ | ผลต่อ chain |
|---|---|---|
| `%v` | เหมือนกัน | error ที่ถูกฝังเข้าไปจะ**หายไป** ไม่สามารถ `errors.Is`/`errors.As` เจอได้อีก — กลายเป็นข้อความ string ธรรมดา |
| `%w` | เหมือนกัน | error ที่ถูกฝังเข้าไปยัง**เก็บ reference ไว้** ทำให้ `errors.Is`/`errors.As`/`errors.Unwrap` เดินย้อนไปเจอได้ |

กฎการใช้: **ใช้ `%w` เมื่อ error ที่ห่อนั้นเป็นข้อมูลที่ caller อาจต้องการเช็คย้อนหลัง** (เกือบทุกกรณีในทางปฏิบัติ) ใช้ `%v` เฉพาะกรณีที่ตั้งใจจะ**ปิดบัง** error ต้นฉบับไม่ให้ caller ภายนอกรู้รายละเอียด (เช่น ไม่อยากให้ error จาก internal database หลุดออกไปเป็น public API contract)

### ห่อได้กี่ชั้น และห่อพร้อมกันหลายตัว

`fmt.Errorf` ห่อ error กี่ชั้นก็ได้ เพราะแต่ละชั้นก็แค่สร้าง error ใหม่ที่มี `Unwrap()` ชี้ไปยัง error ก่อนหน้า ตั้งแต่ Go 1.20 เป็นต้นมา ยังใช้ `%w` **ได้มากกว่าหนึ่งตัวใน `Errorf` เดียวกัน**:

```go
err := fmt.Errorf("failed both: %w and %w", err1, err2)
```

ซึ่งภายในจะใช้กลไกเดียวกับ `errors.Join` ที่จะอธิบายในหัวข้อ 8

---

## 5. `errors.Unwrap`: กลไกเบื้องหลังการห่อ error

`errors.Is` และ `errors.As` ที่เราจะเรียนต่อไปนั้น **ทำงานโดยอาศัย method `Unwrap()`** เป็นแกนกลาง ทุก error ที่ต้องการให้เดินย้อน chain ได้ ต้อง implement interface นี้ (ไม่มีชื่อเป็นทางการ แต่เอกสารมักเรียกว่า `Wrapper`):

```go
type interface {
	Unwrap() error
}
```

เวลาเราเขียน `fmt.Errorf("...: %w", err)` ภายใน Go จะสร้าง struct ที่ซ่อนอยู่ (unexported) ซึ่งมี method `Unwrap()` คืนค่า `err` ตัวที่ถูกห่อไว้ให้อัตโนมัติ — เราไม่ต้องเขียนเองเวลาใช้ `fmt.Errorf`

แต่ถ้าเรา**สร้าง custom error type เอง** และต้องการให้มันเข้าร่วม chain ได้ (เหมือนตัวอย่าง `NotFoundError` ในหัวข้อ 3) เราต้อง implement `Unwrap()` เอง:

```go
func (e *NotFoundError) Unwrap() error {
	return ErrNotFound
}
```

`errors.Unwrap(err)` คือฟังก์ชันที่**เรียก `Unwrap()` ให้เราแค่ชั้นเดียว** ถ้า `err` ไม่มี method `Unwrap()` จะคืนค่า `nil`:

```go
fmt.Println(errors.Unwrap(err)) // ได้ error ชั้นถัดไป หรือ nil ถ้าไม่มี
```

ในทางปฏิบัติเราแทบไม่เรียก `errors.Unwrap` ตรงๆ บ่อยนัก เพราะ `errors.Is` และ `errors.As` จะเรียกมันวนซ้ำให้เราอัตโนมัติจนกว่าจะเจอ (หรือจนกว่า chain จะจบที่ `nil`)

---

## 6. `errors.Is`: เทียบ sentinel error ผ่าน chain ที่ถูกห่อ

`errors.Is(err, target)` ตอบคำถามว่า **"ใน chain ของ `err` (คือตัว `err` เอง หรือ error ใดๆ ที่ถูก `Unwrap()` ไล่ไปเจอ) มี error ที่เท่ากับ `target` หรือไม่"**

การเทียบ "เท่ากับ" ในที่นี้หมายถึง **`==` แบบเดียวกับที่เราใช้เทียบ `io.EOF` มาตั้งแต่ Part 015** (สำหรับ error ที่เป็น comparable value เช่นที่สร้างจาก `errors.New`) หรือถ้า error นั้น implement method `Is(error) bool` เอง `errors.Is` จะเรียก method นั้นแทน (ใช้สำหรับกรณีพิเศษที่ error เดียวกันในทางความหมายอาจไม่เท่ากันแบบ `==` เช่นเทียบด้วย field บางตัว)

```go
if errors.Is(err, ErrNotFound) {
	// พบ ErrNotFound อยู่ที่ไหนสักแห่งใน chain
}
```

ทำไมต้องใช้ `errors.Is` แทน `==` ตรงๆ? เพราะเมื่อ error ผ่านการห่อด้วย `%w` มาแล้ว **ตัว `err` ที่ได้ไม่ใช่ `ErrNotFound` อีกต่อไป** (มันคือ error ใหม่ที่สร้างโดย `fmt.Errorf` ซึ่งข้างในเก็บ reference ไปยัง `ErrNotFound`) การเทียบด้วย `err == ErrNotFound` จึงได้ `false` เสมอไม่ว่าจะห่อกี่ชั้นก็ตาม `errors.Is` แก้ปัญหานี้ด้วยการ**ไล่เดิน chain ทั้งหมดให้อัตโนมัติ**

### ตัวอย่างเปรียบเทียบ `==` กับ `errors.Is`

```go
err := fmt.Errorf("wrapped: %w", ErrNotFound)

fmt.Println(err == ErrNotFound)          // false! ข้อความไม่เท่ากัน เป็นคนละ value
fmt.Println(errors.Is(err, ErrNotFound)) // true! ไล่ chain ไปเจอ ErrNotFound
```

นี่คือเหตุผลที่ **ตั้งแต่ Go 1.13 เป็นต้นมา ควรใช้ `errors.Is` แทน `==` เสมอเมื่อเทียบกับ sentinel error** เว้นแต่มั่นใจ 100% ว่า error นั้นไม่มีทางถูกห่อ (ซึ่งในระบบใหญ่แทบไม่มีทางรู้ล่วงหน้าได้เลย)

---

## 7. `errors.As`: ดึง custom error type ออกจาก chain

ถ้า `errors.Is` คือ "เช็คว่าเจอ error **ค่านี้** หรือไม่" `errors.As` ก็คือ "เช็คว่าเจอ error **ชนิดนี้** หรือไม่ ถ้าเจอให้ดึงออกมาใส่ตัวแปรให้ด้วย" ใช้กับ custom error type ที่เราต้องการอ่าน field ข้างในต่อ

```go
type ValidationError struct {
	Field string
	Msg   string
}

func (e *ValidationError) Error() string {
	return fmt.Sprintf("field %q: %s", e.Field, e.Msg)
}

func processForm(age int) error {
	if age < 0 {
		return &ValidationError{Field: "age", Msg: "ต้องไม่ติดลบ"}
	}
	return nil
}

func handleRequest(age int) error {
	if err := processForm(age); err != nil {
		return fmt.Errorf("handleRequest: %w", err)
	}
	return nil
}

func main() {
	err := handleRequest(-1)
	fmt.Println(err)

	var ve *ValidationError
	if errors.As(err, &ve) {
		fmt.Printf("ดึง ValidationError ออกมาได้: field=%s\n", ve.Field)
	} else {
		fmt.Println("ไม่พบ ValidationError ใน chain")
	}
}
```

ผลลัพธ์:

```
handleRequest: field "age": ต้องไม่ติดลบ
ดึง ValidationError ออกมาได้: field=age
```

### กฎการใช้ `errors.As` ที่ต้องจำให้แม่น

1. **Argument ตัวที่สองต้องเป็น pointer เสมอ** (`&ve` ไม่ใช่ `ve`) เพราะ `errors.As` ต้องเขียนค่ากลับเข้าไปในตัวแปรนั้นถ้าเจอ
2. **ชนิดของตัวแปรที่ประกาศไว้ต้องตรงกับชนิดที่ error คืนมาจริงๆ** ถ้า error คืนเป็น `*ValidationError` ตัวแปรที่ประกาศต้องเป็น `*ValidationError` เช่นกัน (ไม่ใช่ `ValidationError` เฉยๆ) — ผิดพลาดตรงนี้บ่อยมากสำหรับมือใหม่
3. `errors.As` จะไล่เดิน chain ทั้งหมดเหมือน `errors.Is` — เจอที่ layer ไหนก็ดึงออกมาให้ทันที ไม่ต้องสนใจว่าห่อลึกแค่ไหน

### `errors.Is` vs `errors.As`: เลือกใช้เมื่อไหร่

| ต้องการทำอะไร | ใช้ |
|---|---|
| เช็คว่า error นี้คือ sentinel error **ตัวที่รู้จักอยู่แล้ว** หรือไม่ (ไม่สนใจข้อมูลเพิ่มเติม) | `errors.Is(err, ErrNotFound)` |
| ดึงข้อมูลจาก custom error type เพื่อนำไปใช้ต่อ | `errors.As(err, &myErr)` |
| เช็คว่าเกิด error จาก package/layer ไหน แล้วอ่าน field ประกอบการตัดสินใจ | `errors.As` |

---

## 8. `errors.Join` (Go 1.20+): รวมหลาย error เข้าด้วยกัน

บางสถานการณ์ error ไม่ได้เกิดทีละตัว แต่เกิด**พร้อมกันหลายตัว** เช่น validate form ที่มีหลาย field ผิดพร้อมกัน หรือปิด resource หลายตัวใน `defer` ที่แต่ละตัวอาจ fail อิสระจากกัน ก่อน Go 1.20 เราต้องเลือกคืนแค่ error ตัวแรก (ทำให้ข้อมูลของตัวอื่นหายไป) หรือประดิษฐ์ custom type สำหรับเก็บ error หลายตัวเอง

Go 1.20 เพิ่ม `errors.Join` เข้ามาแก้ปัญหานี้โดยตรง:

```go
package main

import (
	"errors"
	"fmt"
)

var (
	ErrDiskFull    = errors.New("disk full")
	ErrNetworkDown = errors.New("network timeout")
)

func saveAndSync() error {
	var errDisk, errNet error
	errDisk = ErrDiskFull
	errNet = ErrNetworkDown

	return errors.Join(errDisk, errNet)
}

func main() {
	err := saveAndSync()
	fmt.Println("joined error:")
	fmt.Println(err)

	fmt.Println("is ErrDiskFull?", errors.Is(err, ErrDiskFull))
	fmt.Println("is ErrNetworkDown?", errors.Is(err, ErrNetworkDown))

	var okErr error
	joined2 := errors.Join(nil, okErr)
	fmt.Println("join ของ nil ทั้งหมด:", joined2)
}
```

ผลลัพธ์:

```
joined error:
disk full
network timeout
is ErrDiskFull? true
is ErrNetworkDown? true
join ของ nil ทั้งหมด: <nil>
```

จุดสำคัญของ `errors.Join`:

1. **`err.Error()` ของผลลัพธ์คือข้อความของ error ทุกตัวต่อกันด้วยขึ้นบรรทัดใหม่** — อ่านง่ายเมื่อ log ออกมาตรงๆ
2. **`errors.Is`/`errors.As` เดิน chain เข้าไปเช็ค error ทุกตัวที่ถูก join** ไม่ใช่แค่ตัวแรก — ต่างจาก `%w` แบบเดี่ยวที่มี chain เดียว ที่นี่ error ที่ join กลายเป็นเหมือน "ต้นไม้" ที่แตกกิ่งได้หลายทาง
3. **ถ้าทุก argument เป็น `nil` ผลลัพธ์คือ `nil`** — ทำให้เขียนโค้ดสะสม error จากหลาย operation ได้อย่างปลอดภัยโดยไม่ต้องเช็คทีละตัวก่อน join

### รูปแบบที่ใช้บ่อย: สะสม error จากหลาย operation ในลูป

```go
func closeAll(closers []io.Closer) error {
	var errs []error
	for _, c := range closers {
		if err := c.Close(); err != nil {
			errs = append(errs, err)
		}
	}
	return errors.Join(errs...) // ถ้า errs ว่างเปล่า จะได้ nil อัตโนมัติ
}
```

รูปแบบนี้พบบ่อยมากในโค้ดที่ต้อง cleanup resource หลายตัวพร้อมกัน (เช่นปิดหลาย connection หรือ validate หลาย field) เพราะไม่ต้องเขียน `if len(errs) == 0 { return nil }` เอง — `errors.Join` เช็คให้แล้ว

---

## 9. การออกแบบ error message ที่ดี

Error message เป็นสิ่งที่ทั้งโปรแกรมเมอร์และบางครั้งผู้ใช้จะเห็นตอน debug ปัญหา ข้อความที่ออกแบบไม่ดีทำให้เสียเวลา debug เพิ่มขึ้นมาก แนวทางที่ชุมชน Go ยึดถือ (อ้างอิงจาก Go style guide และ standard library):

### กฎที่ 1: ใช้ตัวพิมพ์เล็กขึ้นต้น ไม่ใส่เครื่องหมาย `.` ท้ายข้อความ

```go
// ไม่ดี
return errors.New("Failed to open file.")

// ดี
return errors.New("failed to open file")
```

เหตุผล: error message ของเรามักถูกนำไปต่อท้ายด้วย error message อื่นเวลาห่อ (`fmt.Errorf("doSomething: %w", err)`) ถ้าขึ้นต้นด้วยตัวใหญ่หรือมี `.` คั่นกลาง จะทำให้ข้อความที่รวมกันอ่านแปลกๆ เช่น `"doSomething: Failed to open file."` เทียบกับ `"doSomething: failed to open file"` ที่ลื่นไหลกว่า

### กฎที่ 2: ใส่ context ว่า "กำลังทำอะไรอยู่" ไม่ใช่แค่ "อะไรผิด"

```go
// บอกแค่ผลลัพธ์ ไม่รู้ว่าเกิดตอนไหน
return fmt.Errorf("%w", err)

// บอกว่ากำลังทำอะไรอยู่ตอนที่ error เกิด — debug ง่ายกว่ามาก
return fmt.Errorf("loading config from %s: %w", path, err)
```

เมื่อ error ถูกห่อหลายชั้นจนถึง top level ข้อความที่ได้จะกลายเป็น "breadcrumb trail" ที่บอก call stack แบบย่อได้เลย เช่น:

```
startServer: loading config from /etc/app/config.yaml: open /etc/app/config.yaml: no such file or directory
```

อ่านแล้วรู้ทันทีว่าปัญหาเกิดตอนไหน ไม่ต้องเปิด stack trace

### กฎที่ 3: อย่าใส่ error ซ้ำคำเดิม

```go
// ซ้ำซ้อน คำว่า "error" ปรากฏ 3 ครั้ง
return fmt.Errorf("error: failed with error: %w", err)

// กระชับ ชัดเจน
return fmt.Errorf("connect to database: %w", err)
```

### กฎที่ 4: ไม่ต้องบอกว่าเป็น error ซ้ำ เพราะ caller รู้อยู่แล้วจาก type

ฟังก์ชันที่ return `error` ทุกคนที่เรียกใช้ก็รู้อยู่แล้วว่านี่คือ error ไม่จำเป็นต้องเขียนคำว่า "error occurred" ซ้ำในข้อความ

### สรุปสูตรที่ใช้ได้ในเกือบทุกกรณี

```go
fmt.Errorf("<กำลังทำอะไร>: %w", err)
```

เช่น `"parse config"`, `"connect to database"`, `"read file %s"`, `"call user service"` — สั้น กระชับ ตรงประเด็น ต่อกันหลายชั้นแล้วยังอ่านลื่น

---

## 10. รูปแบบที่พบบ่อยและข้อผิดพลาดที่ควรเลี่ยง

### ข้อผิดพลาดที่ 1: เทียบ error ด้วย string

```go
// อย่าทำแบบนี้ — เปราะบางมาก ข้อความเปลี่ยนเมื่อไหร่โค้ดพัง
if err.Error() == "record not found" {
	// ...
}

// ทำแบบนี้แทน
if errors.Is(err, ErrNotFound) {
	// ...
}
```

### ข้อผิดพลาดที่ 2: ใช้ `%v` ตอนที่ตั้งใจจะห่อ error

```go
// ผิดเจตนา -- errors.Is/As จะหา err เดิมไม่เจอ เพราะ chain ถูกตัดขาด
return fmt.Errorf("save failed: %v", err)

// ถูกต้อง
return fmt.Errorf("save failed: %w", err)
```

### ข้อผิดพลาดที่ 3: ห่อ error โดยไม่ใส่ context อะไรเลย

```go
// ไม่มีประโยชน์ — เหมือนแค่ pass err ผ่านเฉยๆ แต่เสีย stack frame ในการ debug
return fmt.Errorf("%w", err)

// ถ้าจะแค่ pass ผ่าน ให้ return err ตรงๆ ไปเลย จะชัดเจนกว่า
return err
```

### ข้อผิดพลาดที่ 4: ลืมว่า custom error type ที่ implement ด้วย pointer receiver ต้องใช้ pointer ตอนเทียบด้วย `errors.As`

```go
type MyError struct{ Msg string }
func (e *MyError) Error() string { return e.Msg }

var target MyError    // ผิด! ต้องเป็น *MyError
errors.As(err, &target)
```

```go
var target *MyError   // ถูกต้อง
errors.As(err, &target)
```

### ข้อผิดพลาดที่ 5: export sentinel error โดยไม่คิดถึงผลกระทบระยะยาว

Sentinel error ที่ export ออกไปแล้ว **ถือเป็นสัญญาถาวรกับผู้ใช้ package** เหมือน public function — เปลี่ยนพฤติกรรม (เช่น เลิกคืน error ตัวนี้ในบางกรณี) อาจทำให้โค้ดที่พึ่งพา `errors.Is` ของผู้ใช้ package เสียหายได้โดยไม่มี compile error เตือน ต้องออกแบบอย่างรอบคอบตั้งแต่แรก

---

## สรุปสิ่งที่ได้เรียนในบทนี้

- **Custom error type**: สร้าง struct ที่มี method `Error() string` เพื่อแนบข้อมูลประกอบเข้ากับ error ได้ (เช่น `ValidationError` ที่เก็บ `Field`, `Value`, `Msg`) ใช้ pointer receiver เป็นธรรมเนียมมาตรฐาน
- **Sentinel error**: ตัวแปร `error` ระดับ package (`var ErrNotFound = errors.New(...)`) ใช้เมื่อ caller แค่ต้องการเช็คว่าเกิด error ประเภทนี้หรือไม่ ตั้งชื่อขึ้นต้นด้วย `Err` เป็นมาตรฐาน
- **Error wrapping**: `fmt.Errorf("...: %w", err)` ห่อ error พร้อมเติม context โดยไม่ทำลาย error ต้นฉบับ ต่างจาก `%v` ที่ตัดขาด chain ทันที
- **`errors.Unwrap(err)`**: ดึง error ชั้นถัดไปจาก chain หนึ่งชั้น เป็นกลไกพื้นฐานที่ `Is`/`As` ใช้ภายใน
- **`errors.Is(err, target)`**: เช็คว่า chain ของ `err` มี error ที่เท่ากับ `target` (sentinel) หรือไม่ ใช้แทน `==` เสมอเมื่อ error อาจถูกห่อ
- **`errors.As(err, &target)`**: ดึง error ที่มีชนิดตรงกับ `target` ออกจาก chain มาเก็บไว้ในตัวแปร ใช้เมื่อต้องการอ่าน field ของ custom error type
- **`errors.Join(errs...)`** (Go 1.20+): รวมหลาย error เป็นค่าเดียว โดยที่ `errors.Is`/`errors.As` ยังไล่เช็ค error ทุกตัวที่ถูกรวมได้ ถ้าทุกตัวเป็น `nil` จะได้ผลลัพธ์ `nil`
- การออกแบบ error message ที่ดี: ตัวเล็กขึ้นต้น ไม่มี `.` ท้าย ใส่ context ว่ากำลังทำอะไร ไม่ซ้ำคำว่า "error"

## แบบฝึกหัดท้ายบท

1. สร้าง custom error type ชื่อ `OutOfStockError` ที่เก็บ field `ProductID string` และ `Requested int` แล้วเขียนฟังก์ชัน `Order(productID string, qty int) error` ที่คืน error นี้เมื่อ `qty > 10` จากนั้นใช้ `errors.As` ดึงข้อมูลออกมาพิมพ์
2. ประกาศ sentinel error สองตัวคือ `ErrInsufficientFunds` และ `ErrAccountLocked` เขียนฟังก์ชัน `Withdraw(balance float64, amount float64, locked bool) error` ที่ห่อ sentinel error เหล่านี้ด้วย context ที่เหมาะสมผ่าน `fmt.Errorf` แล้วทดสอบด้วย `errors.Is`
3. เขียนฟังก์ชัน `ValidateForm` ที่ตรวจสอบ 3 field (name ห้ามว่าง, age ต้องมากกว่า 0, email ต้องมี `@`) โดยสะสม error ของแต่ละ field ที่ผิดด้วย `errors.Join` แล้วคืนค่าเดียว ทดลองส่ง input ที่ผิดพร้อมกันหลาย field แล้วดูข้อความที่ได้
4. อธิบายด้วยคำพูดของตัวเอง (เขียนเป็น comment ในโค้ด) ว่าทำไม `err == ErrNotFound` ถึงคืนค่า `false` หลังจากผ่าน `fmt.Errorf("...: %w", err)` มาแล้ว ทั้งที่ `errors.Is(err, ErrNotFound)` คืนค่า `true`
5. สร้าง custom error type ที่ implement ทั้ง `Error() string` และ `Unwrap() error` เพื่อเชื่อมกับ sentinel error ของตัวเอง (ตามรูปแบบ `NotFoundError` ในหัวข้อ 3) แล้วพิสูจน์ด้วยโค้ดว่าทั้ง `errors.Is` (เช็ค sentinel) และ `errors.As` (ดึง field) ใช้งานพร้อมกันได้จริง
6. ลองเขียนฟังก์ชันที่จงใจใช้ `%v` แทน `%w` ในการห่อ error แล้วพิสูจน์ด้วย `errors.Is` ว่าทำไมมันถึงหา sentinel error ต้นฉบับไม่เจอ — เปรียบเทียบผลลัพธ์กับตอนใช้ `%w`

---

**ต่อไป**: [Part 017 — Panic, Recover และการจัดการข้อผิดพลาดร้ายแรง](./017-panic-and-recover.md)
