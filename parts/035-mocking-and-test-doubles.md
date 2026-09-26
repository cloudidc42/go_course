# Part 035: Mocking และ Test Doubles

> ภาคที่ 2: ระดับกลาง (Intermediate) — ตอนที่ 20 จาก 20 (Part 16–35)

## สารบัญของบทนี้

1. ทำไม Small Interfaces ถึงทำให้ Go เทสได้ง่ายเป็นพิเศษ
2. อนุกรมวิธานของ Test Double: Dummy, Stub, Fake, Spy, Mock
3. Consumer-Defined Interfaces: ประกาศ Interface ที่จุดใช้งาน
4. ตัวอย่างเต็ม: Service ที่พึ่งพา Interface แทน Concrete Type
5. เขียน Fake และ Spy ด้วยมือ
6. ทดสอบ Service ด้วยการ Inject Fake/Spy
7. ทางเลือกแบบ Library: แนะนำ `testify/mock` และ `gomock`
8. Preview: `httptest.Server` สำหรับทดสอบ HTTP Client
9. ข้อควรระวัง: อย่า Over-Mock
10. สรุปสิ่งที่ได้เรียนในบทนี้
11. แบบฝึกหัดท้ายบท

---

## 1. ทำไม Small Interfaces ถึงทำให้ Go เทสได้ง่ายเป็นพิเศษ

ทบทวนจาก **Part 013**: Go เลือกใช้ **implicit satisfaction (structural typing)** — type ใดก็ตามที่มี method ครบตามที่ interface กำหนด จะ implement interface นั้นโดยอัตโนมัติ โดยไม่ต้องประกาศความสัมพันธ์ใดๆ ล่วงหน้า และ Go นิยม **interface ขนาดเล็ก** (มักมีแค่ 1-2 method) มากกว่า interface ขนาดใหญ่

คุณสมบัติสองข้อนี้รวมกันแล้ว **ทำให้การเขียน test double (ตัวปลอมสำหรับใช้แทน dependency จริงตอนทดสอบ) ใน Go ง่ายกว่าภาษาที่มี interface ใหญ่ๆ หรือต้องประกาศ `implements` ชัดเจนอย่างมาก** เพราะ:

- ไม่ต้องมี mocking framework ที่ทำ bytecode manipulation หรือ dynamic proxy ซับซ้อนแบบที่บางภาษาต้องใช้ (เช่น Mockito ของ Java ที่ต้องสร้าง proxy class ที่ runtime)
- แค่เขียน `struct` ธรรมดาที่มี method ตรงตามชื่อและ signature ที่ interface ต้องการ ก็ถือว่า "เป็น" interface นั้นได้ทันที ไม่ต้องประกาศอะไรเพิ่ม
- เมื่อ interface มีแค่ 1-2 method การเขียน struct ปลอมขึ้นมาแทนของจริงใช้เวลาไม่กี่บรรทัด ไม่ต้อง implement method ที่ไม่เกี่ยวข้องทิ้งขว้างเหมือน interface ใหญ่

บทนี้จะสอนวิธีออกแบบโค้ดให้ **testable (ทดสอบได้ง่าย)** ตั้งแต่แรก โดยใช้ interface เป็นจุดแยก (seam) ระหว่าง business logic กับ dependency ภายนอก (database, HTTP call, filesystem) แล้วสาธิตวิธีเขียน test double ทั้งแบบมือเปล่าและแบบใช้ library ช่วย

---

## 2. อนุกรมวิธานของ Test Double: Dummy, Stub, Fake, Spy, Mock

คำว่า **"mock"** มักถูกใช้แบบเหมารวมในภาษาพูด แต่จริงๆ แล้วมีคำศัพท์เฉพาะที่แยกแยะบทบาทของ **test double** (คำรวมที่หมายถึง "ตัวปลอมทุกชนิดที่ใช้แทนของจริงในการทดสอบ" — มาจากคำว่า "stunt double" ในวงการภาพยนตร์) แต่ละแบบไว้ชัดเจน ควรรู้จักไว้เพื่อสื่อสารกับทีมได้ตรงกัน:

| ชนิด | นิยาม | ตัวอย่าง |
|---|---|---|
| **Dummy** | ส่งเข้าไปแค่ให้ signature ครบ ไม่เคยถูกใช้งานจริงในเส้นทางที่ test สนใจ | ส่ง `nil` หรือ struct เปล่าเป็น parameter ที่ test ไม่ได้เกี่ยวข้องด้วย |
| **Stub** | คืนค่าตายตัวตามที่ตั้งไว้ล่วงหน้า ไม่มี logic ซับซ้อน | ฟังก์ชันที่คืน `"Bangkok", nil` เสมอไม่ว่าจะเรียกด้วย input อะไร |
| **Fake** | มี implementation ทำงานได้จริง แต่ง่ายกว่าของจริงมาก (เช่น ใช้ memory แทน database) | `map[int]User` แทน PostgreSQL จริง |
| **Spy** | เหมือน stub/fake แต่ **บันทึกไว้ด้วยว่าถูกเรียกอย่างไรบ้าง** เพื่อให้ test ตรวจสอบทีหลังได้ (ถูกเรียกกี่ครั้ง, ด้วย argument อะไร) | struct ที่เก็บ `calls []string` ทุกครั้งที่ method ถูกเรียก |
| **Mock** | เหมือน spy แต่ **ตั้งความคาดหวังไว้ล่วงหน้า** (เช่น "ต้องถูกเรียกด้วย argument นี้เท่านั้น") แล้ว **fail ทันทีถ้าไม่ตรง** | object จาก library อย่าง `testify/mock` ที่เรียก `.On(...)` ตั้งกฎไว้ก่อน |

ในทางปฏิบัติ Go community มักไม่พิถีพิถันกับศัพท์เหล่านี้เท่าภาษาอื่น (มักเรียกรวมๆ ว่า "fake" หรือ "test double") แต่การเข้าใจความต่างช่วยให้เลือกเครื่องมือที่เหมาะสมกับสถานการณ์ได้ดีขึ้น — บทนี้จะเน้นที่ **Fake** และ **Spy** ที่เขียนเองด้วยมือเป็นหลัก เพราะเป็นแนวทางที่ idiomatic ที่สุดใน Go แล้วค่อยแนะนำ **Mock** จาก library ในหัวข้อ 7

---

## 3. Consumer-Defined Interfaces: ประกาศ Interface ที่จุดใช้งาน

นี่คือหลักการออกแบบที่สำคัญที่สุดของบทนี้ ต่อยอดโดยตรงจากหัวข้อ 3 ใน **Part 013** ที่กล่าวว่า **"interface ควรถูกประกาศฝั่งผู้ใช้งาน (consumer) ไม่ใช่ฝั่งผู้ให้บริการ (producer)"**

ในทางปฏิบัติหมายความว่า: ถ้ามี package `service` ที่ต้องใช้ database ในการทำงาน **ให้ประกาศ interface ที่ระบุว่า `service` ต้องการ method อะไรบ้างไว้ใน package `service` เอง** (ไม่ใช่ import interface จาก package ของ database driver มาใช้ตรงๆ) โดยระบุ **เฉพาะ method ที่ `service` ใช้จริงเท่านั้น** แม้ว่า concrete type ของ database จริงจะมี method อื่นอีกเยอะก็ตาม

```go
// ในไฟล์ userservice.go (package ที่ "ใช้งาน" database)

// UserStore คือ interface ที่ Service "ต้องการ" จากฐานข้อมูล -- ประกาศฝั่งผู้ใช้งาน (consumer)
// มีแค่ method ที่ Service ใช้จริงเท่านั้น ไม่ใช่ทุก method ที่ database มี
type UserStore interface {
	SaveUser(u User) error
	GetUser(id int) (User, error)
}
```

ทำไม pattern นี้ถึงสำคัญมากต่อการทดสอบ:

1. **`service` ไม่รู้จักและไม่สนใจว่า concrete type ที่แท้จริงข้างหลังคืออะไร** — จะเป็น PostgreSQL, MySQL, MongoDB หรือแม้แต่ struct ปลอมในหน่วยความจำ ก็ implement `UserStore` ได้เหมือนกันหมด ตราบใดที่มี method ครบตามที่ interface กำหนด (implicit satisfaction จาก Part 013)
2. **ทดสอบได้โดยไม่ต้องมี database จริงเลย** — แค่เขียน struct ปลอมเล็กๆ ที่มี method `SaveUser`/`GetUser` ตามนี้ ก็ใช้แทนของจริงในการทดสอบได้ทันที
3. **Interface มีขนาดพอดีกับความต้องการจริง** — ถ้า database driver จริงมี method อีก 20 ตัว (`BeginTx`, `Ping`, `Close`, ...) เราไม่จำเป็นต้อง implement ปลอมทั้งหมดนั้นเลย implement แค่ 2 ตัวที่ `Service` ใช้จริงก็พอ

รูปแบบนี้เรียกกันในวงการว่า **Dependency Injection ผ่าน Interface** — `Service` ไม่ได้สร้าง dependency ของตัวเองขึ้นมาโดยตรง (เช่น เรียก `sql.Open(...)` เอง) แต่**รับ dependency เข้ามาจากภายนอก**ผ่าน constructor หรือ field ในรูปแบบ interface แนวคิดนี้จะกลับมาเจออีกครั้งอย่างเป็นระบบใน **Part 100 (Clean Architecture ใน Go)**

---

## 4. ตัวอย่างเต็ม: Service ที่พึ่งพา Interface แทน Concrete Type

มาออกแบบตัวอย่างที่ใกล้เคียงกับงานจริง: `Service` ที่รับสมัครผู้ใช้ใหม่ โดยต้องบันทึกลงฐานข้อมูลและส่งอีเมลต้อนรับ — มี dependency สองตัวที่อยากทดสอบแยกจากของจริง:

```go
// userservice.go
package userservice

import "fmt"

type User struct {
	ID    int
	Name  string
	Email string
}

// UserStore คือ interface ที่ Service ต้องการจากฐานข้อมูล
type UserStore interface {
	SaveUser(u User) error
	GetUser(id int) (User, error)
}

// Notifier คือ interface สำหรับส่งการแจ้งเตือน อาจเป็นอีเมล, SMS หรือ push notification จริงก็ได้
type Notifier interface {
	Notify(to, message string) error
}

// Service คือ business logic หลัก ไม่รู้จักและไม่สนใจว่า UserStore/Notifier ตัวจริงข้างหลังคืออะไร
type Service struct {
	store    UserStore
	notifier Notifier
}

// NewService สร้าง Service โดย inject dependency ทั้งสองเข้ามาจากภายนอก (dependency injection)
func NewService(store UserStore, notifier Notifier) *Service {
	return &Service{store: store, notifier: notifier}
}

// Register บันทึกผู้ใช้ใหม่แล้วส่งอีเมลต้อนรับ
// ถ้าบันทึกไม่สำเร็จ จะไม่ส่งการแจ้งเตือนเลย
func (s *Service) Register(u User) error {
	if err := s.store.SaveUser(u); err != nil {
		return fmt.Errorf("บันทึกผู้ใช้ไม่สำเร็จ: %w", err)
	}
	message := fmt.Sprintf("ยินดีต้อนรับ %s", u.Name)
	if err := s.notifier.Notify(u.Email, message); err != nil {
		return fmt.Errorf("ส่งการแจ้งเตือนไม่สำเร็จ: %w", err)
	}
	return nil
}
```

สังเกตว่า `Service` ไม่มีบรรทัดไหนเลยที่อ้างอิงถึง database หรือ SMTP server จริงๆ — มันรู้จักแค่ `UserStore` และ `Notifier` ในฐานะ interface เท่านั้น นี่คือจุดที่ทำให้เราสามารถสลับใส่ตัวปลอมเข้าไปแทนได้อย่างสมบูรณ์ในโค้ด test โดยไม่ต้องแก้โค้ดของ `Service` แม้แต่บรรทัดเดียว

---

## 5. เขียน Fake และ Spy ด้วยมือ

ทีนี้มาเขียน test double สำหรับทั้งสอง interface — `fakeStore` (Fake ที่ทำงานได้จริงในหน่วยความจำ) และ `spyNotifier` (Spy ที่บันทึกการเรียกไว้ให้ตรวจสอบทีหลัง):

```go
// userservice_test.go
package userservice

import (
	"errors"
	"testing"
)

// fakeStore คือ fake (test double ที่มี implementation จริงแบบง่าย ทำงานได้จริงในหน่วยความจำ)
// ใช้แทนฐานข้อมูลจริงตอนทดสอบ ไม่ต้องต่อ database ใดๆ เลย
type fakeStore struct {
	users     map[int]User
	saveError error // ตั้งค่านี้เพื่อจำลองว่าบันทึกล้มเหลว
}

func newFakeStore() *fakeStore {
	return &fakeStore{users: make(map[int]User)}
}

func (f *fakeStore) SaveUser(u User) error {
	if f.saveError != nil {
		return f.saveError
	}
	f.users[u.ID] = u
	return nil
}

func (f *fakeStore) GetUser(id int) (User, error) {
	u, ok := f.users[id]
	if !ok {
		return User{}, errors.New("ไม่พบผู้ใช้")
	}
	return u, nil
}

// spyNotifier คือ spy (test double ที่บันทึกไว้ว่าถูกเรียกอย่างไรบ้าง เพื่อให้ test ตรวจสอบทีหลังได้)
type spyNotifier struct {
	calls       []string // เก็บข้อความทุกครั้งที่ Notify ถูกเรียก
	notifyError error
}

func (s *spyNotifier) Notify(to, message string) error {
	if s.notifyError != nil {
		return s.notifyError
	}
	s.calls = append(s.calls, to+": "+message)
	return nil
}
```

จุดสำคัญที่ต้องสังเกต:

- `fakeStore` และ `spyNotifier` **ไม่ได้ประกาศว่า "implements `UserStore`" หรือ "implements `Notifier`" ที่ไหนเลย** — เพราะ Go ไม่มี keyword แบบนั้น (ทบทวนจาก **Part 013**) มันเป็น `UserStore`/`Notifier` โดยอัตโนมัติทันทีที่มี method ครบตรงตาม signature
- `fakeStore` มี field `saveError` ที่ปกติเป็น `nil` แต่ test สามารถตั้งค่าให้ไม่เป็น `nil` เพื่อ**จำลองสถานการณ์ error** ได้ตามต้องการ โดยไม่ต้องพึ่งเทคนิคซับซ้อนใดๆ — นี่คือข้อดีของการเขียนเอง: ควบคุมพฤติกรรมได้อย่างอิสระเต็มที่
- `spyNotifier` เก็บ `calls []string` ไว้ ทำให้ test ตรวจสอบได้ภายหลังว่า `Notify` ถูกเรียกกี่ครั้ง ด้วยข้อมูลอะไรบ้าง

---

## 6. ทดสอบ Service ด้วยการ Inject Fake/Spy

มาเขียน test ที่ครอบคลุมสถานการณ์สำคัญ 3 แบบ: กรณีสำเร็จ, กรณีบันทึกฐานข้อมูลล้มเหลว, และกรณีส่งแจ้งเตือนล้มเหลว:

```go
func TestService_Register_Success(t *testing.T) {
	store := newFakeStore()
	notifier := &spyNotifier{}
	svc := NewService(store, notifier)

	u := User{ID: 1, Name: "สมชาย", Email: "somchai@example.com"}
	if err := svc.Register(u); err != nil {
		t.Fatalf("Register() error = %v, want nil", err)
	}

	saved, err := store.GetUser(1)
	if err != nil {
		t.Fatalf("คาดว่าจะพบผู้ใช้ที่บันทึกไว้ แต่ error: %v", err)
	}
	if saved != u {
		t.Errorf("ข้อมูลผู้ใช้ที่บันทึกไม่ตรง: got %+v, want %+v", saved, u)
	}

	if len(notifier.calls) != 1 {
		t.Fatalf("คาดว่า Notify ถูกเรียก 1 ครั้ง แต่ถูกเรียก %d ครั้ง", len(notifier.calls))
	}
	want := "somchai@example.com: ยินดีต้อนรับ สมชาย"
	if notifier.calls[0] != want {
		t.Errorf("Notify call = %q, want %q", notifier.calls[0], want)
	}
}

func TestService_Register_StoreFails(t *testing.T) {
	store := newFakeStore()
	store.saveError = errors.New("database ล่ม")
	notifier := &spyNotifier{}
	svc := NewService(store, notifier)

	err := svc.Register(User{ID: 2, Name: "สมหญิง"})
	if err == nil {
		t.Fatal("คาดว่าจะ error แต่ไม่ error")
	}

	// ต้องไม่มีการส่งแจ้งเตือนเลย ถ้าบันทึกฐานข้อมูลไม่สำเร็จ
	if len(notifier.calls) != 0 {
		t.Errorf("ไม่ควรมีการเรียก Notify เลย แต่ถูกเรียก %d ครั้ง", len(notifier.calls))
	}
}

func TestService_Register_NotifyFails(t *testing.T) {
	store := newFakeStore()
	notifier := &spyNotifier{notifyError: errors.New("SMTP server ล่ม")}
	svc := NewService(store, notifier)

	err := svc.Register(User{ID: 3, Name: "วิชัย", Email: "wichai@example.com"})
	if err == nil {
		t.Fatal("คาดว่าจะ error แต่ไม่ error")
	}

	// ผู้ใช้ควรถูกบันทึกไปแล้ว แม้การแจ้งเตือนจะล้มเหลวทีหลัง
	if _, err := store.GetUser(3); err != nil {
		t.Errorf("คาดว่าผู้ใช้ถูกบันทึกไว้แล้วก่อนที่ notify จะล้มเหลว: %v", err)
	}
}
```

รันทดสอบ:

```bash
go test -v ./userservice/...
```

ผลลัพธ์:

```
=== RUN   TestService_Register_Success
--- PASS: TestService_Register_Success (0.00s)
=== RUN   TestService_Register_StoreFails
--- PASS: TestService_Register_StoreFails (0.00s)
=== RUN   TestService_Register_NotifyFails
--- PASS: TestService_Register_NotifyFails (0.00s)
PASS
ok  	userservice	0.003s
```

สังเกตว่าเราทดสอบ **business logic ของ `Register` ได้ครบทุกเส้นทาง** (สำเร็จ, database ล้มเหลว, notification ล้มเหลว) **โดยไม่เคยต้องต่อฐานข้อมูลจริงหรือส่งอีเมลจริงแม้แต่ครั้งเดียว** — test รันเร็วมาก (เสี้ยววินาที) และรันซ้ำได้ไม่จำกัดโดยไม่มีผลข้างเคียงใดๆ ต่อระบบภายนอก นี่คือคุณค่าหลักของการออกแบบโค้ดด้วย consumer-defined interface

---

## 7. ทางเลือกแบบ Library: แนะนำ `testify/mock` และ `gomock`

การเขียน fake/spy ด้วยมือเหมาะกับกรณีส่วนใหญ่และเป็นแนวทาง idiomatic ที่สุดของ Go แต่เมื่อโปรเจกต์มี interface จำนวนมากที่ต้องทำ mock ซ้ำๆ หรือต้องการตรวจสอบเงื่อนไขการเรียกที่ซับซ้อน (เช่น "ต้องถูกเรียกด้วย argument ที่มี pattern ตรงนี้เท่านั้น" หรือ "ต้องถูกเรียกแน่นอน 3 ครั้ง") การเขียนด้วยมือทุกครั้งเริ่มไม่คุ้มค่าเวลา จึงมี library ช่วยสองตัวที่นิยมใช้กันมากในระบบนิเวศ Go:

### `testify/mock`

[`testify`](https://github.com/stretchr/testify) เป็น library ที่นิยมมากที่สุดตัวหนึ่งสำหรับงาน testing ใน Go (มี submodule `assert` สำหรับ assertion สั้นๆ ด้วย) ติดตั้งด้วย:

```bash
go get github.com/stretchr/testify
```

วิธีใช้คือสร้าง struct ที่ embed `mock.Mock` แล้ว implement interface โดยเรียก `m.Called(...)` ส่งต่อให้ testify จัดการเบื้องหลัง:

```go
package testifyexample

import (
	"testing"

	"github.com/stretchr/testify/mock"
)

type Notifier interface {
	Notify(to, message string) error
}

// MockNotifier คือ mock ที่สร้างจาก testify/mock -- ไม่ต้องเขียน struct fake ด้วยมือเอง
type MockNotifier struct {
	mock.Mock
}

func (m *MockNotifier) Notify(to, message string) error {
	args := m.Called(to, message)
	return args.Error(0)
}

func TestWithTestifyMock(t *testing.T) {
	mockNotifier := new(MockNotifier)

	// ตั้งความคาดหวังไว้ล่วงหน้า: ถ้าถูกเรียกด้วย argument นี้ ให้คืนค่า nil (ไม่ error)
	mockNotifier.On("Notify", "a@example.com", "สวัสดี").Return(nil)

	err := mockNotifier.Notify("a@example.com", "สวัสดี")
	if err != nil {
		t.Fatalf("ไม่ควร error: %v", err)
	}

	// ตรวจสอบว่า Notify ถูกเรียกตามที่คาดไว้จริง (จำนวนครั้ง, argument ที่ถูกต้อง)
	mockNotifier.AssertExpectations(t)
	mockNotifier.AssertCalled(t, "Notify", "a@example.com", "สวัสดี")
}
```

ผลลัพธ์:

```
=== RUN   TestWithTestifyMock
--- PASS: TestWithTestifyMock (0.00s)
PASS
```

`mockNotifier.On(...)` ตั้งความคาดหวังไว้ล่วงหน้าว่าถ้า `Notify` ถูกเรียกด้วย argument ที่ระบุ ให้คืนค่าอะไร ถ้าถูกเรียกด้วย argument ที่ไม่ตรงกับที่ตั้งไว้เลย จะ panic ทันทีพร้อมข้อความบอกว่าไม่มี expectation ที่ match — และ `AssertExpectations(t)` ช่วยตรวจสอบว่าทุก expectation ที่ตั้งไว้ถูกเรียกจริงครบถ้วน (ไม่ใช่แค่ไม่ error แต่ยังไม่เคยถูกเรียกเลย)

### `gomock`

อีกทางเลือกที่นิยมคือ [`gomock`](https://github.com/uber-go/mock) (เดิมพัฒนาโดยทีม Google ปัจจุบันดูแลต่อโดย Uber) ซึ่งต่างจาก `testify/mock` ตรงที่ **ใช้ code generation** — มีเครื่องมือ `mockgen` ที่อ่าน interface ที่มีอยู่แล้วสร้างไฟล์ mock ให้อัตโนมัติ แทนที่จะเขียน struct ด้วยมือ:

```bash
# ติดตั้งเครื่องมือ mockgen (ทำครั้งเดียว)
go install go.uber.org/mock/mockgen@latest

# สร้างไฟล์ mock จาก interface Notifier ใน package userservice
mockgen -source=userservice.go -destination=mocks/notifier_mock.go -package=mocks
```

ไฟล์ที่ถูก generate ออกมาจะมีหน้าตาประมาณนี้ (ย่อให้เห็นโครงสร้างหลัก):

```go
// mocks/notifier_mock.go (ไฟล์นี้ถูก generate อัตโนมัติ ไม่ควรแก้ไขด้วยมือ)
type MockNotifier struct {
	ctrl     *gomock.Controller
	recorder *MockNotifierMockRecorder
}

func (m *MockNotifier) Notify(to, message string) error {
	ret := m.ctrl.Call(m, "Notify", to, message)
	return ret[0].(error) // แปลงกลับเป็น error type ที่ถูกต้อง
}

func (mr *MockNotifierMockRecorder) Notify(to, message any) *gomock.Call {
	return mr.ctrl.RecordCallWithMethodType(mr.mock, "Notify", ...)
}
```

แล้วใช้งานใน test:

```go
func TestWithGomock(t *testing.T) {
	ctrl := gomock.NewController(t)
	mockNotifier := mocks.NewMockNotifier(ctrl)

	mockNotifier.EXPECT().
		Notify("a@example.com", "สวัสดี").
		Return(nil).
		Times(1) // ต้องถูกเรียกพอดี 1 ครั้งเท่านั้น ไม่งั้น test fail อัตโนมัติตอนจบ

	svc := userservice.NewService(fakeStore, mockNotifier)
	svc.Register(userservice.User{Email: "a@example.com", Name: "..."})
}
```

**ข้อแตกต่างหลักระหว่างสองแนวทาง**: `testify/mock` เขียน mock struct ด้วยมือเอง (แค่ไม่กี่บรรทัดต่อ interface) เหมาะกับ interface ที่มีไม่กี่ตัว ส่วน `gomock` ใช้ code generation ทำให้ mock ที่ได้ type-safe กว่า (ตรวจสอบตอน compile time มากกว่า) และสะดวกเมื่อ project มี interface จำนวนมากที่ต้องทำ mock ซ้ำๆ แต่ต้องมีขั้นตอน generate โค้ดเพิ่มเข้ามาในกระบวนการ build

> **หมายเหตุ**: ตัวอย่าง `gomock` ข้างบนเป็นโค้ดสาธิตเพื่อให้เห็นรูปแบบการใช้งาน ไม่ได้ถูก generate และรันจริงในบทนี้ (ต่างจากตัวอย่าง `testify/mock` ที่รันผ่านจริงแล้วข้างบน) เพราะต้องติดตั้งเครื่องมือ `mockgen` และรัน code generation ก่อนเสมอ — ถ้าสนใจใช้งานจริงแนะนำให้ลองทำตามแบบฝึกหัดท้ายบท

---

## 8. Preview: `httptest.Server` สำหรับทดสอบ HTTP Client

เราได้เห็น `httptest.NewServer` มาบ้างแล้วใน **Part 032** ตอนทดสอบ context timeout กับ HTTP request — นี่คือเครื่องมือที่สำคัญมากสำหรับการทดสอบโค้ดที่เรียก HTTP API ภายนอก และคุ้มค่าที่จะแนะนำเพิ่มเติมในบทนี้ เพราะมันคือ**ทางเลือกที่ดีกว่าการ mock `http.Client` ในหลายกรณี** (รายละเอียดเชิงลึกเต็มรูปแบบจะอยู่ใน **Part 081: `httptest` — ทดสอบ HTTP Handler**)

```go
// weatherclient.go
package weatherclient

import (
	"encoding/json"
	"fmt"
	"net/http"
)

type Weather struct {
	City        string  `json:"city"`
	Temperature float64 `json:"temperature"`
}

type Client struct {
	BaseURL    string
	HTTPClient *http.Client
}

func NewClient(baseURL string) *Client {
	return &Client{BaseURL: baseURL, HTTPClient: http.DefaultClient}
}

func (c *Client) GetWeather(city string) (*Weather, error) {
	url := fmt.Sprintf("%s/weather?city=%s", c.BaseURL, city)
	resp, err := c.HTTPClient.Get(url)
	if err != nil {
		return nil, fmt.Errorf("เรียก API ไม่สำเร็จ: %w", err)
	}
	defer resp.Body.Close()

	if resp.StatusCode != http.StatusOK {
		return nil, fmt.Errorf("API ตอบสถานะ %d", resp.StatusCode)
	}

	var w Weather
	if err := json.NewDecoder(resp.Body).Decode(&w); err != nil {
		return nil, fmt.Errorf("decode JSON ไม่สำเร็จ: %w", err)
	}
	return &w, nil
}
```

แทนที่จะ mock `http.Client` (ซึ่งทำได้ยุ่งยากเพราะ `http.Client` เป็น concrete struct ไม่ใช่ interface เล็กๆ) เราใช้ **`httptest.NewServer`** สร้าง HTTP server จริงๆ ขึ้นมาชั่วคราวในเครื่องระหว่างทดสอบ แล้วให้ `Client` ของเรายิง request ไปหาเซิร์ฟเวอร์จำลองนี้จริงๆ ผ่าน network stack ของเครื่อง (แค่เป็น `localhost` เท่านั้น):

```go
// weatherclient_test.go
package weatherclient

import (
	"encoding/json"
	"net/http"
	"net/http/httptest"
	"testing"
)

func TestGetWeather(t *testing.T) {
	server := httptest.NewServer(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		if r.URL.Query().Get("city") != "Bangkok" {
			t.Errorf("คาดว่า city=Bangkok แต่ได้ %q", r.URL.Query().Get("city"))
		}
		w.Header().Set("Content-Type", "application/json")
		json.NewEncoder(w).Encode(Weather{City: "Bangkok", Temperature: 33.5})
	}))
	defer server.Close()

	client := NewClient(server.URL)
	weather, err := client.GetWeather("Bangkok")
	if err != nil {
		t.Fatalf("GetWeather() error = %v", err)
	}

	if weather.City != "Bangkok" || weather.Temperature != 33.5 {
		t.Errorf("got %+v, want {Bangkok 33.5}", weather)
	}
}

func TestGetWeather_ServerError(t *testing.T) {
	server := httptest.NewServer(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		w.WriteHeader(http.StatusInternalServerError)
	}))
	defer server.Close()

	client := NewClient(server.URL)
	_, err := client.GetWeather("Bangkok")
	if err == nil {
		t.Fatal("คาดว่าจะ error เมื่อ server ตอบ 500 แต่ไม่ error")
	}
}
```

ผลลัพธ์:

```
=== RUN   TestGetWeather
--- PASS: TestGetWeather (0.00s)
=== RUN   TestGetWeather_ServerError
--- PASS: TestGetWeather_ServerError (0.00s)
PASS
```

ทำไมวิธีนี้ถึงดีกว่าการ mock `http.Client` ในหลายกรณี:

- **ทดสอบ code path ทั้งหมดจริงๆ** ตั้งแต่การสร้าง URL, การส่ง HTTP request, ไปจนถึงการ decode response — ไม่ใช่แค่ทดสอบว่าโค้ดของเรา "เรียก" `http.Client` ถูกวิธีเฉยๆ
- **ไม่ต้องแตะโครงสร้างของ `Client`** เพื่อทำให้ mock ได้ (ถ้าใช้วิธี mock `http.Client` ตรงๆ มักต้องแยก interface พิเศษออกมาห่อ `http.Client.Do` ซึ่งเพิ่มความซับซ้อนโดยไม่จำเป็น)
- **จำลอง response ที่ซับซ้อนได้ง่าย** — กำหนด status code, header, body ตามที่ต้องการได้อย่างอิสระผ่าน `http.HandlerFunc` ธรรมดา ทบทวนความรู้เรื่อง HTTP handler ที่จะเจาะลึกใน **Part 047**

หัวข้อนี้เป็นเพียงตัวอย่างเบื้องต้น — `httptest` ยังมีเครื่องมืออีกมาก เช่น `httptest.NewRecorder()` สำหรับทดสอบ HTTP handler โดยตรง (ไม่ใช่ client) ซึ่งจะเรียนอย่างละเอียดเต็มรูปแบบใน **Part 081**

---

## 9. ข้อควรระวัง: อย่า Over-Mock

หลังเรียนเทคนิคการสร้าง test double มามากมาย มีกับดักสำคัญที่ต้องระวังคือ **"over-mocking"** — การ mock ทุกอย่างที่ขวางหน้าโดยไม่จำเป็น ทำให้ test เปราะบางและดูแลรักษายากกว่าประโยชน์ที่ได้

หลักการตัดสินใจที่ควรยึดถือคือ:

> **Mock/Fake เฉพาะ dependency ที่ "แพง" หรือ "ไม่แน่นอน" เท่านั้น — dependency ที่ทำงานได้เร็ว, กำหนดผลลัพธ์ได้แน่นอน และไม่มีผลข้างเคียงต่อระบบภายนอก ควรใช้ของจริงตรงๆ ในการทดสอบ**

ตัวอย่างของ dependency ที่**ควร**ทำ fake/mock:

- **Database จริง** — ช้า, ต้องมี network, ต้องเตรียม/ล้างข้อมูลก่อน-หลัง test, ผลลัพธ์อาจไม่แน่นอนถ้ามีข้อมูลเก่าค้างอยู่
- **HTTP call ไปยัง API ภายนอก** — ช้า, ต้องพึ่ง network, อาจมี rate limit, และไม่ควรให้ test ยิง request จริงไปยัง service ของบริษัทอื่นโดยไม่ตั้งใจ
- **ระบบส่งอีเมล/SMS จริง** — ไม่มีใครอยากได้อีเมลจริงส่งไปหาลูกค้าทุกครั้งที่รัน test!
- **นาฬิกาของระบบ (`time.Now()`)** — ถ้า business logic ขึ้นกับเวลาปัจจุบัน ควร inject เป็น interface หรือ function เพื่อควบคุมเวลาให้แน่นอนตอนทดสอบ

ตัวอย่างของ dependency ที่**ไม่ควร**ทำ fake/mock (ควรใช้ของจริง):

- **`map` หรือ slice ในหน่วยความจำ** — เร็วอยู่แล้ว ไม่มีผลข้างเคียง ไม่มีเหตุผลต้อง mock
- **ฟังก์ชันคำนวณล้วนๆ (pure function)** เช่น `mathutil.Add`, `strutil.Initials` จาก Part 033-034 — เรียกของจริงตรงๆ ในทุก test ได้เลย ไม่มีต้นทุนอะไรเลย
- **struct ธรรมดาที่ไม่มีการติดต่อกับโลกภายนอก** — สร้าง instance จริงใช้ในการทดสอบตรงๆ ดีกว่าเสมอ เพราะทดสอบพฤติกรรมจริงได้แม่นยำกว่า mock ที่อาจเขียนพฤติกรรมผิดเพี้ยนไปจากของจริงโดยไม่รู้ตัว

**สัญญาณเตือนว่ากำลัง over-mock**:

1. Test ผ่านหมด แต่แก้ implementation จริงนิดเดียวก็ทำให้ test พังจำนวนมาก (mock ผูกติดกับรายละเอียดภายในมากเกินไป แทนที่จะทดสอบพฤติกรรมที่สังเกตได้จากภายนอก)
2. เขียน mock ซับซ้อนกว่า logic จริงที่กำลังทดสอบ (เสียเวลาดูแล mock มากกว่าประโยชน์ที่ได้)
3. Mock ทุก dependency จนไม่เหลือโค้ดจริงให้ทดสอบเลย — test แค่ยืนยันว่า "เรียก mock ถูกต้อง" ไม่ได้พิสูจน์ว่า business logic ทำงานถูกต้องจริง

หลักการทองคำสรุปสั้นๆ: **mock เฉพาะขอบเขต (boundary) ระหว่างระบบของเรากับโลกภายนอกที่ควบคุมไม่ได้ ส่วนโค้ดภายในระบบของเราเองที่เร็วและไม่มีผลข้างเคียง ให้ทดสอบด้วยของจริงเสมอ**

---

## สรุปสิ่งที่ได้เรียนในบทนี้

- Go เทสง่ายเป็นพิเศษเพราะ **implicit interface satisfaction** (Part 013) ทำให้เขียน struct ปลอมขึ้นมาแทน dependency จริงได้โดยไม่ต้องประกาศ `implements` และ **small interfaces idiom** ทำให้ implement ปลอมมีแค่ไม่กี่ method ที่จำเป็นจริงๆ
- **Test double** แบ่งเป็นหลายชนิด: Dummy, Stub, Fake (ทำงานได้จริงแบบง่าย), Spy (บันทึกการเรียก), Mock (ตั้งความคาดหวังล่วงหน้าและ fail ถ้าไม่ตรง)
- **Consumer-defined interfaces**: ประกาศ interface ที่ package ผู้ใช้งาน โดยระบุเฉพาะ method ที่ต้องใช้จริงเท่านั้น ทำให้ swap ระหว่าง dependency จริงกับตัวปลอมได้อย่างอิสระ
- ออกแบบ `Service` ให้รับ dependency (`UserStore`, `Notifier`) ผ่าน constructor ในรูปแบบ interface (dependency injection) แล้วทดสอบด้วยการ inject fake/spy ที่เขียนเองแทนของจริง — ทดสอบครบทุก code path ได้เร็วโดยไม่ต้องมี database/SMTP จริง
- `testify/mock` คือ library ยอดนิยมสำหรับสร้าง mock แบบเขียนเอง (`.On(...)`, `.AssertExpectations(t)`) ส่วน `gomock` ใช้ code generation ผ่าน `mockgen` เหมาะกับโปรเจกต์ที่มี interface จำนวนมาก
- **`httptest.Server`** เป็นทางเลือกที่ดีกว่าการ mock `http.Client` สำหรับทดสอบโค้ดที่เรียก HTTP API — สร้าง server จำลองจริงในเครื่องแล้วให้ client ยิง request จริงเข้าไป ทดสอบ code path ได้ครบถ้วนกว่า (รายละเอียดเต็มใน Part 081)
- **อย่า over-mock**: mock เฉพาะ dependency ที่ช้า/ไม่แน่นอน/มีผลข้างเคียงต่อโลกภายนอกเท่านั้น ส่วน dependency ที่เร็วและไม่มีผลข้างเคียง (in-memory data structure, pure function) ควรใช้ของจริงตรงๆ ในการทดสอบเสมอ

## แบบฝึกหัดท้ายบท

1. ออกแบบ interface `PaymentGateway` ที่มี method `Charge(amount float64) error` เพียงตัวเดียว แล้วเขียน `OrderService` ที่พึ่งพา interface นี้ (แทน concrete payment provider จริง) พร้อมเขียน fake สำหรับทดสอบทั้งกรณีจ่ายสำเร็จและจ่ายล้มเหลว
2. เพิ่ม spy เข้าไปใน fake ของแบบฝึกหัดข้อ 1 เพื่อตรวจสอบว่า `Charge` ถูกเรียกด้วยจำนวนเงินที่ถูกต้องพอดี ไม่มากไม่น้อยกว่าที่ควร
3. ติดตั้ง `testify` (`go get github.com/stretchr/testify`) แล้วเขียน `MockPaymentGateway` ด้วย `mock.Mock` แทนที่ fake ที่เขียนเองในข้อ 1 เปรียบเทียบว่าโค้ดต่างกันอย่างไร แบบไหนอ่านง่ายกว่าในกรณีนี้
4. เขียน HTTP client ตัวใหม่ที่เรียก API สมมติ (เช่น ดึงรายชื่อสินค้า) แล้วทดสอบด้วย `httptest.NewServer` ให้ครอบคลุมทั้งกรณี status 200 พร้อมข้อมูลถูกต้อง, กรณี status 404, และกรณี response body เป็น JSON ที่ผิดรูปแบบ (invalid JSON)
5. ทบทวนโค้ดจากแบบฝึกหัดก่อนหน้าทั้งหมด (Part 033-034) แล้วระบุว่ามี dependency ไหนบ้างที่ "ควร" ทำ fake/mock และไหนบ้างที่ "ไม่ควร" ตามหลักการในหัวข้อ 9 พร้อมอธิบายเหตุผล
6. ค้นคว้าเพิ่มเติม: อ่านเกี่ยวกับแนวคิด **"test pyramid"** (Unit Test เยอะที่สุด, Integration Test ปานกลาง, End-to-End Test น้อยที่สุด) แล้วอธิบายว่าเทคนิคจากบทนี้ (fake, spy, mock) เหมาะกับ test ระดับไหนของพีระมิดนี้มากที่สุด และทำไม — เตรียมคำตอบไว้ก่อนเข้า **Part 080 (Integration Testing)**

---

**ต่อไป**: [Part 036 — Goroutines พื้นฐาน](./036-goroutines.md)
