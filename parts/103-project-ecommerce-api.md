# Part 103: โปรเจกต์ E-Commerce REST API แบบเต็มรูปแบบ

> ภาคที่ 10: มืออาชีพและระดับโลก (Professional & World-Class) — ตอนที่ 4 จาก 11 (Part 100–110)

## สารบัญของบทนี้

1. ภาพรวมโปรเจกต์และการนำ Clean Architecture มาใช้จริง
2. โครงสร้างโปรเจกต์และการติดตั้ง dependency
3. ชั้น Domain: Entity และ Repository Interface
4. ชั้น Repository: GORM + SQLite Adapter
5. หัวใจของธุรกรรม: `PlaceOrder` แบบ Atomic
6. ชั้น Usecase: Business Logic ของสินค้า
7. ชั้น Usecase: Authentication ด้วย bcrypt + JWT
8. ชั้น Usecase: การสั่งซื้อสินค้า
9. ชั้น Handler: JSON Envelope, Validation, Error Mapping
10. ชั้น Handler: Auth Middleware และ Role-Based Access Control
11. ชั้น Handler: Product, Order Handler และ Router ด้วย chi
12. `cmd/api/main.go`: ประกอบร่างทั้งหมดและ Graceful Shutdown
13. รันจริงและทดสอบด้วย `curl` แบบ End-to-End
14. Integration Test ด้วย `httptest`
15. สิ่งที่ตั้งใจไม่ทำในบทนี้ และจะไปต่อที่ไหน
16. วิธีรันโปรเจกต์นี้ด้วยตัวเอง
17. สรุปสิ่งที่ได้เรียนในบทนี้
18. แบบฝึกหัดท้ายบท

---

## 1. ภาพรวมโปรเจกต์และการนำ Clean Architecture มาใช้จริง

ตลอด 102 part ที่ผ่านมา เราเรียนเครื่องมือแยกกันเป็นชิ้นๆ: HTTP server (Part 47, 56-58), REST design (Part 59), JSON (Part 60), JWT (Part 67), GORM (Part 74-75), Transaction (Part 78), httptest (Part 81) และล่าสุดคือ Clean Architecture (Part 100) บทนี้คือจุดที่เอาทุกชิ้นมา**ประกอบเป็นระบบจริงหนึ่งระบบ** — ไม่ใช่ตัวอย่างสาธิตแยกไฟล์เดียวแบบที่ผ่านมา แต่เป็นโปรเจกต์หลายแพ็กเกจที่แบ่งชั้นตามความรับผิดชอบอย่างเป็นระบบ

โปรเจกต์คือ **E-Commerce REST API** ที่มีความสามารถครบวงจรระดับที่ใช้สาธิตหลักการได้จริง:

- ดูรายการสินค้า ค้นหา กรองตามหมวดหมู่ พร้อม pagination
- สมัครสมาชิกและ login ด้วย JWT
- สั่งซื้อสินค้า (ต้อง login) โดยระบบต้องลด stock และบันทึกคำสั่งซื้อ**แบบ atomic** — ถ้า stock ไม่พอต้อง rollback ทั้งหมด ห้ามขายเกิน stock เด็ดขาด
- ดูประวัติคำสั่งซื้อของตัวเอง
- Role-based access control: เฉพาะ `admin` เท่านั้นที่สร้างสินค้า/หมวดหมู่ใหม่ได้

โค้ดทั้งหมดในบทนี้**ถูกเขียน คอมไพล์ รันจริง และทดสอบจริง** ด้วย Go 1.24.7 ก่อนนำมาลงในบทเรียน ผลลัพธ์ที่แสดงทุกจุด (JSON response, `go test` output, error message) คือผลลัพธ์จริงจากการรันโค้ดชุดนี้ ไม่ใช่โค้ดที่เขียนแล้วเดาผลลัพธ์เอา

### สถาปัตยกรรมที่ใช้: Clean Architecture 4 ชั้น (ตาม Part 100)

```
┌─────────────────────────────────────────────────────────┐
│  internal/handler   (Delivery: HTTP, chi, JSON, JWT)     │
│  ┌─────────────────────────────────────────────────┐    │
│  │  internal/usecase   (Business Logic)             │    │
│  │  ┌─────────────────────────────────────────┐    │    │
│  │  │  internal/domain   (Entity + Interface)  │    │    │
│  │  └─────────────────────────────────────────┘    │    │
│  └─────────────────────────────────────────────────┘    │
│                         ▲                                 │
│                         │ implements                       │
│  internal/repository/sqlite  (GORM + SQLite Adapter)      │
└─────────────────────────────────────────────────────────┘
```

กฎการ import ที่ยึดถือเคร่งครัดตลอดทั้งโปรเจกต์ (ตรวจสอบได้จริงด้วย `go vet` และการอ่านโค้ดทุกไฟล์):

| ชั้น | Import จาก | ห้าม import จาก |
|---|---|---|
| `internal/domain` | เฉพาะ standard library (`context`, `time`, `errors`) | ทุกอย่างในโปรเจกต์เดียวกัน |
| `internal/usecase` | `internal/domain` เท่านั้น | `internal/handler`, `internal/repository`, GORM, chi |
| `internal/repository/sqlite` | `internal/domain`, GORM, SQLite driver | `internal/usecase`, `internal/handler` |
| `internal/handler` | `internal/usecase`, `internal/domain`, chi, validator | `internal/repository` (!) |

ข้อสังเกตสำคัญที่สุด: **`internal/handler` ไม่ import `internal/repository` เลย** — handler รู้จักแค่ `*usecase.ProductService`, `*usecase.AuthService`, `*usecase.OrderService` เท่านั้น การเชื่อมทุกอย่างเข้าด้วยกัน (การสร้าง `sqlite.ProductRepository` แล้วส่งเข้า `usecase.NewProductService`) เกิดขึ้นที่จุดเดียวในทั้งโปรเจกต์คือ `cmd/api/main.go` — เทคนิคนี้เรียกว่า **dependency injection ด้วยมือ (manual wiring)** ซึ่งเป็นแนวทางที่ Part 100 แนะนำสำหรับโปรเจกต์ขนาดนี้ (ไม่ต้องพึ่ง DI framework ภายนอกให้ซับซ้อนเกินความจำเป็น)

---

## 2. โครงสร้างโปรเจกต์และการติดตั้ง dependency

```bash
mkdir ecommerce-api && cd ecommerce-api
go mod init ecommerce-api
```

โครงสร้างโฟลเดอร์เต็ม (ยึดตาม golang-standards/project-layout ที่แนะนำไว้ตั้งแต่ **Part 001**):

```
ecommerce-api/
├── cmd/
│   └── api/
│       └── main.go
├── internal/
│   ├── domain/
│   │   ├── entity.go
│   │   ├── errors.go
│   │   └── repository.go
│   ├── usecase/
│   │   ├── product_service.go
│   │   ├── auth_service.go
│   │   └── order_service.go
│   ├── repository/
│   │   └── sqlite/
│   │       ├── models.go
│   │       ├── db.go
│   │       ├── errors.go
│   │       ├── category_repo.go
│   │       ├── product_repo.go
│   │       ├── user_repo.go
│   │       └── order_repo.go
│   └── handler/
│       ├── response.go
│       ├── middleware.go
│       ├── auth_handler.go
│       ├── product_handler.go
│       ├── order_handler.go
│       ├── router.go
│       └── handler_test.go
├── go.mod
└── go.sum
```

ติดตั้ง dependency ทั้งหมดที่ใช้ในบทนี้ (แต่ละตัวคือของที่เรียนไปแล้วในบทก่อนหน้าทั้งสิ้น):

```bash
go get github.com/go-chi/chi/v5              # router — Part 058
go get gorm.io/gorm                          # ORM — Part 074-075
go get gorm.io/driver/sqlite                 # GORM driver สำหรับ SQLite — Part 074
go get github.com/golang-jwt/jwt/v5          # JWT — Part 067
go get golang.org/x/crypto/bcrypt            # bcrypt password hashing — Part 051
go get github.com/go-playground/validator/v10 # struct-tag validation — Part 060
```

`go.mod` ที่ได้จริงหลังรันคำสั่งข้างต้น (ทดสอบด้วย Go 1.24.7):

```
module ecommerce-api

go 1.24.7

require (
	github.com/go-chi/chi/v5 v5.3.2
	github.com/go-playground/validator/v10 v10.22.1
	github.com/golang-jwt/jwt/v5 v5.3.1
	golang.org/x/crypto v0.31.0
	gorm.io/driver/sqlite v1.6.0
	gorm.io/gorm v1.31.2
)

require (
	github.com/gabriel-vasile/mimetype v1.4.3 // indirect
	github.com/go-playground/locales v0.14.1 // indirect
	github.com/go-playground/universal-translator v0.18.1 // indirect
	github.com/jinzhu/inflection v1.0.0 // indirect
	github.com/jinzhu/now v1.1.5 // indirect
	github.com/leodido/go-urn v1.4.0 // indirect
	github.com/mattn/go-sqlite3 v1.14.22 // indirect
	golang.org/x/net v0.21.0 // indirect
	golang.org/x/sys v0.28.0 // indirect
	golang.org/x/text v0.21.0 // indirect
)
```

> **หมายเหตุเรื่องเวอร์ชัน**: ตอนรัน `go get` ครั้งแรกในสภาพแวดล้อมที่ใช้เขียนบทนี้ `go get` เลือกเวอร์ชันล่าสุดของ `golang.org/x/crypto` และ `validator/v10` ที่ต้องการ Go 1.26 ขึ้นไป (ใหม่กว่า toolchain 1.24.7 ที่ใช้ตรวจสอบบทนี้) จึงต้องระบุเวอร์ชันที่รองรับ Go 1.24 อย่างชัดเจน (`go get golang.org/x/crypto@v0.31.0` และ `go get github.com/go-playground/validator/v10@v10.22.1`) — นี่คือสถานการณ์จริงที่พบได้บ่อยเมื่อ dependency อัปเดตเร็วกว่า toolchain ที่ทีมใช้งาน จะเรียนวิธีจัดการเรื่องนี้อย่างเป็นระบบใน **Part 108: Go Modules Best Practices**

---

## 3. ชั้น Domain: Entity และ Repository Interface

ชั้น `internal/domain` คือ "ชั้นในสุด" ตาม Clean Architecture — ไม่ import อะไรจากแพ็กเกจอื่นในโปรเจกต์เลยแม้แต่ตัวเดียว ประกอบด้วย 3 ไฟล์

### `internal/domain/entity.go`

```go
// Package domain เก็บ entity หลักและ repository interface ของระบบ
// เป็น "ชั้นในสุด" ตามแนวคิด Clean Architecture (Part 100) — ไม่ import
// อะไรจาก internal/repository, internal/usecase หรือ internal/handler เลย
// เพื่อให้ business rule ไม่ขึ้นกับกรอบงาน (framework), ฐานข้อมูล หรือ HTTP
package domain

import "time"

// Category คือหมวดหมู่สินค้า
type Category struct {
	ID   uint   `json:"id"`
	Name string `json:"name"`
	Slug string `json:"slug"`
}

// Product คือสินค้าในร้าน เก็บราคาเป็นหน่วยสตางค์ (PriceCents) แบบจำนวนเต็ม
// แทนที่จะใช้ float64 — เป็นแนวปฏิบัติมาตรฐานสำหรับเงินเพื่อเลี่ยงปัญหา
// floating-point rounding error (เช่น 0.1 + 0.2 != 0.3 ในเลขฐานสอง)
type Product struct {
	ID          uint      `json:"id"`
	Name        string    `json:"name"`
	Description string    `json:"description"`
	PriceCents  int64     `json:"price_cents"`
	Stock       int       `json:"stock"`
	CategoryID  uint      `json:"category_id"`
	Category    *Category `json:"category,omitempty"`
	CreatedAt   time.Time `json:"created_at"`
	UpdatedAt   time.Time `json:"updated_at"`
}

// User คือผู้ใช้งานระบบ ไม่มี field JSON สำหรับ PasswordHash เพราะห้าม
// ส่งกลับให้ client เด็ดขาด (ตามแนวคิด Part 051)
type User struct {
	ID           uint      `json:"id"`
	Name         string    `json:"name"`
	Email        string    `json:"email"`
	PasswordHash string    `json:"-"`
	Role         string    `json:"role"`
	CreatedAt    time.Time `json:"created_at"`
}

const (
	RoleCustomer = "customer"
	RoleAdmin    = "admin"
)

// OrderStatus คือสถานะของคำสั่งซื้อ
type OrderStatus string

const (
	OrderStatusPaid      OrderStatus = "paid"
	OrderStatusCancelled OrderStatus = "cancelled"
)

// OrderItem คือสินค้าหนึ่งรายการในคำสั่งซื้อ เก็บ "snapshot" ของชื่อสินค้า
// และราคา ณ เวลาที่สั่งซื้อไว้ตรงๆ (denormalization โดยตั้งใจ) เพราะถ้าเก็บ
// แค่ ProductID แล้วราคาสินค้าถูกแก้ไขภายหลัง ประวัติการสั่งซื้อเก่าจะผิดเพี้ยน
type OrderItem struct {
	ID             uint   `json:"id"`
	OrderID        uint   `json:"order_id"`
	ProductID      uint   `json:"product_id"`
	ProductName    string `json:"product_name"`
	UnitPriceCents int64  `json:"unit_price_cents"`
	Quantity       int    `json:"quantity"`
}

// Order คือคำสั่งซื้อของผู้ใช้หนึ่งคน
type Order struct {
	ID         uint        `json:"id"`
	UserID     uint        `json:"user_id"`
	Status     OrderStatus `json:"status"`
	TotalCents int64       `json:"total_cents"`
	Items      []OrderItem `json:"items"`
	CreatedAt  time.Time   `json:"created_at"`
}
```

จุดที่ควรสังเกต:

- **`PriceCents int64`** — เก็บเงินเป็นหน่วยสตางค์แบบจำนวนเต็มเสมอ ไม่ใช้ `float64` เพื่อเลี่ยงปัญหาการปัดเศษของ floating point ที่เรียนไปใน **Part 003**
- **`PasswordHash` มี `json:"-"`** — รับประกันว่าไม่ว่าจะ serialize struct `User` จากจุดไหนในโปรแกรม (พลาดลืมสร้าง response DTO แยกก็ตาม) ค่านี้จะไม่รั่วไหลออกไปใน JSON เด็ดขาด เป็นการป้องกันสองชั้นต่อจากที่ handler จะสร้าง `userResponse` แยกไว้อีกที
- **`OrderItem` เก็บ `ProductName`/`UnitPriceCents` เป็น snapshot** — ไม่ join ไปดึงจากตาราง `products` สดๆ ทุกครั้งที่ดูประวัติ เพราะราคาสินค้าอาจถูกแก้ไขได้ในอนาคต แต่ใบเสร็จเก่าต้องแสดงราคา ณ วันที่ซื้อเสมอ

### `internal/domain/errors.go`

```go
package domain

import "errors"

// Sentinel error ของชั้น domain (แนวคิดจาก Part 016: errors.Is/As) —
// ชั้น usecase และ handler ตรวจสอบ error เหล่านี้ด้วย errors.Is เพื่อ
// ตัดสินใจว่าจะตอบ HTTP status code อะไรกลับไป โดยไม่ต้อง import
// อะไรจาก database driver หรือ HTTP package เข้ามาปนใน domain เลย
var (
	// ErrNotFound ใช้เมื่อค้นหา entity ใดๆ ไม่เจอ (Product, User, Order, Category)
	ErrNotFound = errors.New("resource not found")

	// ErrEmailTaken ใช้ตอนสมัครสมาชิกด้วยอีเมลที่มีอยู่แล้ว
	ErrEmailTaken = errors.New("email already registered")

	// ErrInvalidCredentials ใช้ตอน login ด้วย email/password ที่ไม่ถูกต้อง
	// (ตั้งใจใช้ข้อความเดียวกันไม่ว่าจะ "ไม่พบ user" หรือ "รหัสผ่านผิด"
	// เพื่อป้องกัน user enumeration ตามที่อธิบายไว้ใน Part 067)
	ErrInvalidCredentials = errors.New("invalid email or password")

	// ErrInsufficientStock ใช้ตอนสั่งซื้อสินค้าที่ stock ไม่พอ
	ErrInsufficientStock = errors.New("insufficient stock")

	// ErrValidation ใช้สำหรับ input ที่ผิดกฎทางธุรกิจ (business rule)
	// ที่ตรวจพบในชั้น usecase (ต่างจาก struct-tag validation ในชั้น handler)
	ErrValidation = errors.New("validation failed")
)
```

การรวม sentinel error ไว้ที่ชั้น domain แล้วให้ทั้ง usecase และ handler ใช้ `errors.Is` ตรวจสอบร่วมกัน คือกลไกที่ทำให้ **การแปลง "ความหมายทางธุรกิจ" เป็น "HTTP status code" เกิดขึ้นที่จุดเดียว** (ในไฟล์ `handler/response.go` หัวข้อ 9) แทนที่จะกระจัดกระจายไปทั่วทุก handler

### `internal/domain/repository.go`

```go
package domain

import "context"

// ProductFilter รวมเงื่อนไขการค้นหา/แบ่งหน้าสินค้าไว้ในที่เดียว
// (แนวคิด pagination + filtering จาก Part 059 หัวข้อ 6-7)
type ProductFilter struct {
	Search     string // ค้นหาจากชื่อสินค้าแบบ partial match
	CategoryID uint   // 0 หมายถึงไม่กรองตามหมวดหมู่
	Page       int    // เริ่มที่ 1
	PageSize   int
}

// ProductRepository คือ "port" ที่ชั้น usecase เรียกใช้ ส่วนชั้น
// repository/sqlite คือ "adapter" ที่ implement มันจริง — นี่คือหัวใจของ
// Dependency Inversion Principle ใน Clean Architecture (Part 100):
// usecase ไม่รู้จัก GORM หรือ SQLite เลยแม้แต่น้อย รู้จักแค่ interface นี้
type ProductRepository interface {
	Create(ctx context.Context, p *Product) error
	GetByID(ctx context.Context, id uint) (*Product, error)
	List(ctx context.Context, filter ProductFilter) ([]Product, int64, error)
}

// CategoryRepository จัดการหมวดหมู่สินค้า
type CategoryRepository interface {
	Create(ctx context.Context, c *Category) error
	List(ctx context.Context) ([]Category, error)
	GetByID(ctx context.Context, id uint) (*Category, error)
}

// UserRepository จัดการผู้ใช้งาน
type UserRepository interface {
	Create(ctx context.Context, u *User) error
	GetByEmail(ctx context.Context, email string) (*User, error)
	GetByID(ctx context.Context, id uint) (*User, error)
}

// OrderRepository จัดการคำสั่งซื้อ
type OrderRepository interface {
	// PlaceOrder ต้อง implement แบบ atomic: การลดจำนวน stock ของสินค้า
	// ทุกชิ้น กับการสร้างคำสั่งซื้อ+รายการสินค้า ต้องสำเร็จหรือล้มเหลว
	// พร้อมกันทั้งหมดในหนึ่ง database transaction เดียว (ดู Part 078)
	// ฟังก์ชันนี้เติมค่า ID, TotalCents, CreatedAt กลับเข้าไปใน order ที่รับเข้ามา
	PlaceOrder(ctx context.Context, order *Order) error

	ListByUser(ctx context.Context, userID uint) ([]Order, error)
	GetByID(ctx context.Context, id uint) (*Order, error)
}
```

interface สี่ตัวนี้คือ "สัญญา" ทั้งหมดที่ชั้น usecase ต้องพึ่งพา — ไม่มากไม่น้อยไปกว่านี้ ทุก method ถูกออกแบบจากมุมมอง "usecase ต้องการอะไร" ไม่ใช่ "GORM ทำอะไรได้บ้าง" ซึ่งเป็นทิศทางการออกแบบที่ถูกต้องตามหลัก Dependency Inversion: **นามธรรม (interface) ต้องไม่ขึ้นกับรายละเอียด (implementation) แต่รายละเอียดต้องขึ้นกับนามธรรม**

---

## 4. ชั้น Repository: GORM + SQLite Adapter

ชั้นนี้คือ "adapter" ที่ implement interface ของ `domain` ด้วย GORM (Part 074-075) เชื่อมกับ SQLite เหตุผลที่เลือก SQLite สำหรับโปรเจกต์นี้เหมือนกับที่อธิบายไว้ใน Part 074: **ไม่ต้องมีเซิร์ฟเวอร์ฐานข้อมูลแยกต่างหาก** ฐานข้อมูลทั้งก้อนเป็นไฟล์เดียว ทำให้ทุกคนที่โคลนโค้ดไปรันได้ทันทีโดยไม่ต้องติดตั้งอะไรเพิ่ม

### `internal/repository/sqlite/models.go`

```go
// Package sqlite คือ adapter ที่ implement repository interface ของ
// internal/domain โดยใช้ GORM (Part 074-075) กับ SQLite เป็นฐานข้อมูล
//
// เหตุผลที่ใช้ SQLite ในโปรเจกต์นี้: ไม่ต้องมีเซิร์ฟเวอร์ฐานข้อมูลแยก
// ต่างหาก ฐานข้อมูลทั้งก้อนเป็นไฟล์เดียว ทำให้ทุกคนที่เรียนโคลนโค้ดไปแล้ว
// รันได้ทันทีโดยไม่ต้องติดตั้งอะไรเพิ่ม เหมาะกับการสาธิตและพัฒนา
//
// สลับไปใช้ PostgreSQL จริงตาม Part 072 ทำได้ง่ายมาก เพราะโค้ดในแพ็กเกจนี้
// ใช้ API ของ GORM ล้วนๆ (db.Create, db.Where, db.Transaction, ...) ไม่มี
// SQL เฉพาะทางของ SQLite เจือปนเลยแม้แต่บรรทัดเดียว จุดที่ต้องแก้มีแค่
// ฟังก์ชัน Open ในไฟล์ db.go: เปลี่ยน `sqlite.Open(dsn)` เป็น
// `postgres.Open(dsn)` จาก `gorm.io/driver/postgres` เท่านั้น โค้ดที่เหลือ
// ทั้งหมดในไฟล์นี้และไฟล์ *_repo.go ไม่ต้องแตะเลยแม้แต่บรรทัดเดียว
package sqlite

import "time"

// categoryModel คือ GORM model ของตาราง categories — สังเกตว่าแยกออกจาก
// domain.Category โดยตั้งใจ: struct tag ของ GORM (`gorm:"..."`) เป็นรายละเอียด
// ของ "adapter" ชั้นนอก ไม่ควรรั่วไหลเข้าไปปนกับ entity ในชั้น domain ที่ต้อง
// เป็นกลางไม่ขึ้นกับเทคโนโลยีใดๆ ตามหลัก Clean Architecture (Part 100)
type categoryModel struct {
	ID   uint   `gorm:"primaryKey"`
	Name string `gorm:"size:150;not null"`
	Slug string `gorm:"size:150;uniqueIndex;not null"`
}

func (categoryModel) TableName() string { return "categories" }

// productModel คือ GORM model ของตาราง products
type productModel struct {
	ID          uint `gorm:"primaryKey"`
	Name        string
	Description string `gorm:"type:text"`
	PriceCents  int64
	Stock       int
	CategoryID  uint
	Category    categoryModel `gorm:"foreignKey:CategoryID;references:ID"`
	CreatedAt   time.Time
	UpdatedAt   time.Time
}

func (productModel) TableName() string { return "products" }

// userModel คือ GORM model ของตาราง users
type userModel struct {
	ID           uint `gorm:"primaryKey"`
	Name         string
	Email        string `gorm:"size:190;uniqueIndex;not null"`
	PasswordHash string
	Role         string `gorm:"size:30;not null;default:customer"`
	CreatedAt    time.Time
}

func (userModel) TableName() string { return "users" }

// orderModel คือ GORM model ของตาราง orders พร้อม has-many ไปยัง orderItemModel
type orderModel struct {
	ID         uint `gorm:"primaryKey"`
	UserID     uint `gorm:"index;not null"`
	Status     string
	TotalCents int64
	Items      []orderItemModel `gorm:"foreignKey:OrderID;references:ID"`
	CreatedAt  time.Time
}

func (orderModel) TableName() string { return "orders" }

// orderItemModel คือ GORM model ของตาราง order_items
type orderItemModel struct {
	ID             uint `gorm:"primaryKey"`
	OrderID        uint `gorm:"index;not null"`
	ProductID      uint `gorm:"not null"`
	ProductName    string
	UnitPriceCents int64
	Quantity       int
}

func (orderItemModel) TableName() string { return "order_items" }
```

**ทำไมต้องแยก `productModel` กับ `domain.Product` เป็นคนละ struct?** นี่คือคำถามที่พบบ่อยที่สุดเมื่อเริ่มใช้ Clean Architecture จริง คำตอบคือ struct tag ของ GORM (`gorm:"primaryKey"`, `gorm:"uniqueIndex"`) เป็นรายละเอียดของฐานข้อมูลที่เลือกใช้ ถ้าปนเข้าไปใน `domain.Product` โดยตรง เท่ากับว่าชั้น domain "รู้จัก" GORM ทันที ซึ่งขัดกับกฎ Dependency Inversion ที่วางไว้ — ถ้าวันหนึ่งเปลี่ยนจาก GORM ไปเป็น `sqlx` หรือ raw `database/sql` (Part 071) โค้ด `domain` จะไม่กระทบเลยแม้แต่บรรทัดเดียว มีแต่ไฟล์ `_repo.go` และ `models.go` ในแพ็กเกจนี้เท่านั้นที่ต้องเขียนใหม่

### `internal/repository/sqlite/db.go`

```go
package sqlite

import (
	"fmt"

	"gorm.io/driver/sqlite"
	"gorm.io/gorm"
	"gorm.io/gorm/logger"
)

// Open เชื่อมต่อฐานข้อมูล SQLite ตาม dsn ที่ให้มา แล้ว AutoMigrate ตาราง
// ทั้งหมดให้พร้อมใช้งาน (แนวคิด AutoMigrate จาก Part 075)
//
// dsn ทั่วไปสำหรับใช้งานจริงคือชื่อไฟล์ เช่น "ecommerce.db" ส่วนตอนเทส
// ในไฟล์ _test.go ของ Part นี้จะใช้ไฟล์ชั่วคราวจาก t.TempDir() แทน
func Open(dsn string) (*gorm.DB, error) {
	db, err := gorm.Open(sqlite.Open(dsn), &gorm.Config{
		Logger: logger.Default.LogMode(logger.Silent),
	})
	if err != nil {
		return nil, fmt.Errorf("sqlite: open %q: %w", dsn, err)
	}

	// SQLite ผ่าน driver mattn/go-sqlite3 (ที่ gorm.io/driver/sqlite ใช้ภายใน)
	// รองรับการเขียนพร้อมกันได้แค่ connection เดียวจริงๆ ในแต่ละครั้ง
	// (ตัวไฟล์เดียวไม่มี concurrent writer แบบ PostgreSQL/MySQL) การจำกัด
	// MaxOpenConns ไว้ที่ 1 ช่วยเลี่ยง error "database is locked" ที่พบบ่อย
	// เมื่อมีหลาย goroutine เขียนพร้อมกัน — เป็นข้อจำกัดเฉพาะของ SQLite ที่
	// PostgreSQL/MySQL (Part 072/073) ไม่มีปัญหานี้เพราะรองรับ connection
	// pool ขนาดใหญ่กว่ามากตามที่เรียนใน Part 078
	sqlDB, err := db.DB()
	if err != nil {
		return nil, fmt.Errorf("sqlite: get *sql.DB: %w", err)
	}
	sqlDB.SetMaxOpenConns(1)

	if err := db.AutoMigrate(
		&categoryModel{},
		&productModel{},
		&userModel{},
		&orderModel{},
		&orderItemModel{},
	); err != nil {
		return nil, fmt.Errorf("sqlite: automigrate: %w", err)
	}

	return db, nil
}
```

### `internal/repository/sqlite/errors.go`

```go
package sqlite

import "strings"

// isUniqueConstraintErr ตรวจว่า error ที่ได้จาก GORM/driver เกิดจากการ
// ละเมิด UNIQUE constraint หรือไม่ ใช้การเช็คข้อความ error แทนการ import
// github.com/mattn/go-sqlite3 ตรงๆ (ซึ่งเป็นแพ็กเกจ C-cgo) เพื่อไม่ให้
// แพ็กเกจนี้ผูกแน่นกับ driver มากเกินไป — ถ้าย้ายไป PostgreSQL/MySQL
// (Part 072/073) ในอนาคต จุดนี้คือจุดเดียวที่ต้องเพิ่มเงื่อนไขตรวจ error
// message ของ driver ใหม่เข้าไป (เช่น "duplicate key value" ของ Postgres
// หรือ error 1062 ของ MySQL) โดยไม่กระทบโค้ดส่วนอื่นเลย
func isUniqueConstraintErr(err error) bool {
	if err == nil {
		return false
	}
	return strings.Contains(err.Error(), "UNIQUE constraint failed")
}
```

### `internal/repository/sqlite/category_repo.go` และ `product_repo.go`

```go
package sqlite

import (
	"context"
	"errors"

	"gorm.io/gorm"

	"ecommerce-api/internal/domain"
)

// CategoryRepository implement domain.CategoryRepository ด้วย GORM
type CategoryRepository struct {
	db *gorm.DB
}

func NewCategoryRepository(db *gorm.DB) *CategoryRepository {
	return &CategoryRepository{db: db}
}

func toDomainCategory(m categoryModel) domain.Category {
	return domain.Category{ID: m.ID, Name: m.Name, Slug: m.Slug}
}

func (r *CategoryRepository) Create(ctx context.Context, c *domain.Category) error {
	m := categoryModel{Name: c.Name, Slug: c.Slug}
	if err := r.db.WithContext(ctx).Create(&m).Error; err != nil {
		return err
	}
	c.ID = m.ID
	return nil
}

func (r *CategoryRepository) List(ctx context.Context) ([]domain.Category, error) {
	var rows []categoryModel
	if err := r.db.WithContext(ctx).Order("id").Find(&rows).Error; err != nil {
		return nil, err
	}
	result := make([]domain.Category, 0, len(rows))
	for _, m := range rows {
		result = append(result, toDomainCategory(m))
	}
	return result, nil
}

// GetByID ใช้ตรวจสอบว่า CategoryID ที่ผูกกับสินค้ามีอยู่จริงไหมตอนสร้าง
// สินค้าใหม่ (เรียกจาก usecase.ProductService.Create)
func (r *CategoryRepository) GetByID(ctx context.Context, id uint) (*domain.Category, error) {
	var m categoryModel
	err := r.db.WithContext(ctx).First(&m, id).Error
	if errors.Is(err, gorm.ErrRecordNotFound) {
		return nil, domain.ErrNotFound
	}
	if err != nil {
		return nil, err
	}
	c := toDomainCategory(m)
	return &c, nil
}
```

```go
package sqlite

import (
	"context"
	"errors"

	"gorm.io/gorm"

	"ecommerce-api/internal/domain"
)

// ProductRepository implement domain.ProductRepository ด้วย GORM
type ProductRepository struct {
	db *gorm.DB
}

func NewProductRepository(db *gorm.DB) *ProductRepository {
	return &ProductRepository{db: db}
}

func toDomainProduct(m productModel) domain.Product {
	p := domain.Product{
		ID:          m.ID,
		Name:        m.Name,
		Description: m.Description,
		PriceCents:  m.PriceCents,
		Stock:       m.Stock,
		CategoryID:  m.CategoryID,
		CreatedAt:   m.CreatedAt,
		UpdatedAt:   m.UpdatedAt,
	}
	if m.Category.ID != 0 {
		cat := toDomainCategory(m.Category)
		p.Category = &cat
	}
	return p
}

func (r *ProductRepository) Create(ctx context.Context, p *domain.Product) error {
	m := productModel{
		Name:        p.Name,
		Description: p.Description,
		PriceCents:  p.PriceCents,
		Stock:       p.Stock,
		CategoryID:  p.CategoryID,
	}
	if err := r.db.WithContext(ctx).Create(&m).Error; err != nil {
		return err
	}
	p.ID = m.ID
	p.CreatedAt = m.CreatedAt
	p.UpdatedAt = m.UpdatedAt
	return nil
}

func (r *ProductRepository) GetByID(ctx context.Context, id uint) (*domain.Product, error) {
	var m productModel
	err := r.db.WithContext(ctx).Preload("Category").First(&m, id).Error
	if errors.Is(err, gorm.ErrRecordNotFound) {
		return nil, domain.ErrNotFound
	}
	if err != nil {
		return nil, err
	}
	p := toDomainProduct(m)
	return &p, nil
}

// List คืนสินค้าตามเงื่อนไขใน filter พร้อมจำนวนทั้งหมดที่ตรงเงื่อนไข (ไม่
// นับการแบ่งหน้า) สำหรับ client เอาไปคำนวณจำนวนหน้าทั้งหมด — รูปแบบ
// offset-based pagination ตามที่ออกแบบไว้ใน Part 059 หัวข้อ 6
func (r *ProductRepository) List(ctx context.Context, filter domain.ProductFilter) ([]domain.Product, int64, error) {
	query := r.db.WithContext(ctx).Model(&productModel{})

	if filter.Search != "" {
		query = query.Where("name LIKE ?", "%"+filter.Search+"%")
	}
	if filter.CategoryID != 0 {
		query = query.Where("category_id = ?", filter.CategoryID)
	}

	var total int64
	if err := query.Count(&total).Error; err != nil {
		return nil, 0, err
	}

	page, pageSize := filter.Page, filter.PageSize
	if page < 1 {
		page = 1
	}
	if pageSize < 1 {
		pageSize = 20
	}
	offset := (page - 1) * pageSize

	var rows []productModel
	err := query.Preload("Category").
		Order("id").
		Limit(pageSize).
		Offset(offset).
		Find(&rows).Error
	if err != nil {
		return nil, 0, err
	}

	result := make([]domain.Product, 0, len(rows))
	for _, m := range rows {
		result = append(result, toDomainProduct(m))
	}
	return result, total, nil
}
```

### `internal/repository/sqlite/user_repo.go`

```go
package sqlite

import (
	"context"
	"errors"

	"gorm.io/gorm"

	"ecommerce-api/internal/domain"
)

// UserRepository implement domain.UserRepository ด้วย GORM
type UserRepository struct {
	db *gorm.DB
}

func NewUserRepository(db *gorm.DB) *UserRepository {
	return &UserRepository{db: db}
}

func toDomainUser(m userModel) domain.User {
	return domain.User{
		ID:           m.ID,
		Name:         m.Name,
		Email:        m.Email,
		PasswordHash: m.PasswordHash,
		Role:         m.Role,
		CreatedAt:    m.CreatedAt,
	}
}

func (r *UserRepository) Create(ctx context.Context, u *domain.User) error {
	m := userModel{
		Name:         u.Name,
		Email:        u.Email,
		PasswordHash: u.PasswordHash,
		Role:         u.Role,
	}
	if err := r.db.WithContext(ctx).Create(&m).Error; err != nil {
		// GORM ส่ง error ดิบของ driver กลับมาตรงๆ เมื่อ unique constraint
		// ที่ตาราง users.email ถูกละเมิด (เทียบกับ Part 072/073 ที่แต่ละ
		// driver มีรหัส error ต่างกัน) ในระดับ repository เราแปลงให้เป็น
		// domain.ErrEmailTaken ที่ชั้น usecase/handler ตรวจสอบได้ง่ายๆ
		// ด้วย errors.Is โดยไม่ต้องรู้จัก driver ใดๆ เลย
		if isUniqueConstraintErr(err) {
			return domain.ErrEmailTaken
		}
		return err
	}
	u.ID = m.ID
	u.CreatedAt = m.CreatedAt
	return nil
}

func (r *UserRepository) GetByEmail(ctx context.Context, email string) (*domain.User, error) {
	var m userModel
	err := r.db.WithContext(ctx).Where("email = ?", email).First(&m).Error
	if errors.Is(err, gorm.ErrRecordNotFound) {
		return nil, domain.ErrNotFound
	}
	if err != nil {
		return nil, err
	}
	u := toDomainUser(m)
	return &u, nil
}

func (r *UserRepository) GetByID(ctx context.Context, id uint) (*domain.User, error) {
	var m userModel
	err := r.db.WithContext(ctx).First(&m, id).Error
	if errors.Is(err, gorm.ErrRecordNotFound) {
		return nil, domain.ErrNotFound
	}
	if err != nil {
		return nil, err
	}
	u := toDomainUser(m)
	return &u, nil
}
```

---

## 5. หัวใจของธุรกรรม: `PlaceOrder` แบบ Atomic

นี่คือส่วนสำคัญที่สุดของทั้งโปรเจกต์ในแง่ความถูกต้อง (correctness) — โจทย์ที่โปรเจกต์ระดับ Part 078 (Database Transactions) เตรียมเราไว้โดยตรง: **การลด stock ของสินค้าทุกชิ้น กับการสร้างคำสั่งซื้อ ต้องสำเร็จพร้อมกันทั้งหมด หรือไม่สำเร็จเลยสักรายการ**

### `internal/repository/sqlite/order_repo.go`

```go
package sqlite

import (
	"context"
	"errors"

	"gorm.io/gorm"

	"ecommerce-api/internal/domain"
)

// OrderRepository implement domain.OrderRepository ด้วย GORM
type OrderRepository struct {
	db *gorm.DB
}

func NewOrderRepository(db *gorm.DB) *OrderRepository {
	return &OrderRepository{db: db}
}

func toDomainOrder(m orderModel) domain.Order {
	items := make([]domain.OrderItem, 0, len(m.Items))
	for _, it := range m.Items {
		items = append(items, domain.OrderItem{
			ID:             it.ID,
			OrderID:        it.OrderID,
			ProductID:      it.ProductID,
			ProductName:    it.ProductName,
			UnitPriceCents: it.UnitPriceCents,
			Quantity:       it.Quantity,
		})
	}
	return domain.Order{
		ID:         m.ID,
		UserID:     m.UserID,
		Status:     domain.OrderStatus(m.Status),
		TotalCents: m.TotalCents,
		Items:      items,
		CreatedAt:  m.CreatedAt,
	}
}

// PlaceOrder คือหัวใจของทั้งโปรเจกต์ในแง่ database transaction (Part 078):
// สำหรับสินค้าทุกชิ้นในคำสั่งซื้อ ต้อง "ลด stock" และ "บันทึกคำสั่งซื้อ"
// สำเร็จพร้อมกันทั้งหมด หรือไม่สำเร็จเลยสักรายการ (all-or-nothing ตาม
// หลัก Atomicity ของ ACID) — ใช้ db.Transaction ของ GORM ซึ่งภายใน
// ทำหน้าที่เหมือน BEGIN/COMMIT/ROLLBACK ที่เขียนด้วยมือใน Part 078
// หัวข้อ 2 ทุกประการ แต่ GORM จัดการ defer rollback ให้อัตโนมัติเมื่อ
// callback คืน error ใดๆ ก็ตาม
func (r *OrderRepository) PlaceOrder(ctx context.Context, order *domain.Order) error {
	return r.db.WithContext(ctx).Transaction(func(tx *gorm.DB) error {
		items := make([]orderItemModel, 0, len(order.Items))
		var total int64

		for _, it := range order.Items {
			// เงื่อนไข `stock >= ?` ใน WHERE ทำให้การลด stock กับการเช็คว่า
			// stock พอหรือไม่ เกิดขึ้นเป็นคำสั่ง SQL เดียวแบบ atomic ระดับแถว
			// (ป้องกัน race condition แบบ "อ่านค่า stock แล้วค่อยเขียนทีหลัง"
			// ซึ่งสอง request ที่มาพร้อมกันอาจอ่านค่าเดิมเห็นตรงกันทั้งคู่
			// แล้วต่างฝ่ายต่างคิดว่า stock พอ ทำให้ขายเกิน stock จริงได้)
			res := tx.Model(&productModel{}).
				Where("id = ? AND stock >= ?", it.ProductID, it.Quantity).
				UpdateColumn("stock", gorm.Expr("stock - ?", it.Quantity))
			if res.Error != nil {
				return res.Error
			}
			if res.RowsAffected == 0 {
				// ไม่มีแถวไหนถูกอัปเดตเลย แปลว่า id ไม่มีอยู่จริง หรือ
				// stock ไม่พอ — คืน error แล้วปล่อยให้ GORM rollback
				// ธุรกรรมทั้งหมดให้อัตโนมัติ (รวมรายการที่ลด stock ไปแล้ว
				// ก่อนหน้านี้ในลูปเดียวกันด้วย)
				return domain.ErrInsufficientStock
			}

			var pm productModel
			if err := tx.First(&pm, it.ProductID).Error; err != nil {
				return err
			}

			subtotal := pm.PriceCents * int64(it.Quantity)
			total += subtotal
			items = append(items, orderItemModel{
				ProductID:      pm.ID,
				ProductName:    pm.Name,
				UnitPriceCents: pm.PriceCents,
				Quantity:       it.Quantity,
			})
		}

		om := orderModel{
			UserID:     order.UserID,
			Status:     string(order.Status),
			TotalCents: total,
			Items:      items,
		}
		if err := tx.Create(&om).Error; err != nil {
			return err
		}

		// เติมค่าที่ database สร้างให้กลับเข้าไปใน order ที่ caller ส่งเข้ามา
		// (แนวคิดเดียวกับ RETURNING ที่เห็นใน Part 072/074)
		*order = toDomainOrder(om)
		return nil
	})
}

func (r *OrderRepository) ListByUser(ctx context.Context, userID uint) ([]domain.Order, error) {
	var rows []orderModel
	err := r.db.WithContext(ctx).
		Preload("Items").
		Where("user_id = ?", userID).
		Order("id DESC").
		Find(&rows).Error
	if err != nil {
		return nil, err
	}
	result := make([]domain.Order, 0, len(rows))
	for _, m := range rows {
		result = append(result, toDomainOrder(m))
	}
	return result, nil
}

func (r *OrderRepository) GetByID(ctx context.Context, id uint) (*domain.Order, error) {
	var m orderModel
	err := r.db.WithContext(ctx).Preload("Items").First(&m, id).Error
	if errors.Is(err, gorm.ErrRecordNotFound) {
		return nil, domain.ErrNotFound
	}
	if err != nil {
		return nil, err
	}
	o := toDomainOrder(m)
	return &o, nil
}
```

### วิเคราะห์กลไก atomic ทีละขั้น

1. **`tx.Model(&productModel{}).Where("id = ? AND stock >= ?", ...).UpdateColumn(...)`** คือหัวใจของความปลอดภัยทั้งหมด — เงื่อนไข `stock >= quantity` อยู่ใน `WHERE` clause เดียวกับคำสั่ง `UPDATE` ทำให้ "เช็คว่า stock พอไหม" กับ "ลด stock" เป็นปฏิบัติการเดียวที่ฐานข้อมูล lock แถวนั้นไว้ตลอด ไม่มีช่องให้ request อื่นแทรกเข้ามาอ่านค่าเก่าระหว่างกลาง (เทียบกับการเขียนแบบผิดที่พบบ่อย: `SELECT stock` ก่อน แล้วเช็คด้วย `if` ใน Go แล้วค่อย `UPDATE` — วิธีนั้นมี race window ระหว่าง SELECT กับ UPDATE ที่สอง request พร้อมกันอาจขายสินค้าชิ้นสุดท้ายซ้ำกันทั้งคู่ได้)
2. **`res.RowsAffected == 0`** คือสัญญาณเดียวที่บอกว่า "ลดไม่สำเร็จ" — อาจเพราะ `id` ไม่มีอยู่จริง หรือ `stock` ไม่พอ (แยกสองกรณีนี้ออกจากกันในระดับนี้ไม่จำเป็น เพราะ `usecase.OrderService.PlaceOrder` เช็คว่าสินค้ามีอยู่จริงไปแล้วชั้นหนึ่งก่อนเรียกมาถึงตรงนี้)
3. **`return domain.ErrInsufficientStock`** ภายใน callback ของ `Transaction` คือกลไกที่ทำให้ GORM สั่ง `ROLLBACK` ให้อัตโนมัติทันที — รวมถึงรายการสินค้าอื่นในลูปเดียวกันที่ลด stock สำเร็จไปแล้วก่อนหน้าด้วย (สมมติสั่งซื้อ 3 รายการ รายการที่ 3 stock ไม่พอ รายการที่ 1-2 ที่ลด stock ไปแล้วก็ต้องถูก rollback กลับคืนด้วย ไม่ใช่ถูกลดค้างไว้)
4. **`*order = toDomainOrder(om)`** เติมค่าที่ database สร้างให้ (ID, CreatedAt) กลับเข้า pointer ที่ caller ส่งเข้ามา — รูปแบบเดียวกับที่ `db.Create` ของ GORM เติม field กลับให้ตามที่อธิบายไว้ใน Part 074

ทดสอบพฤติกรรมนี้จริงในหัวข้อที่ 13-14 ด้านล่าง จะเห็นว่าเมื่อสั่งซื้อเกิน stock ที่มีจริง คำสั่งซื้อจะถูกปฏิเสธ **และ stock จะไม่ถูกแตะต้องเลยแม้แต่หน่วยเดียว**

---

## 6. ชั้น Usecase: Business Logic ของสินค้า

ชั้น `internal/usecase` พึ่งพาแค่ interface ใน `internal/domain` เท่านั้น ไม่รู้จัก GORM, chi หรือ `net/http` เลย — ทำให้เทสได้ง่ายด้วย fake repository (Part 035) ในอนาคต และสลับ implementation ของ repository ได้โดยไม่กระทบโค้ดในแพ็กเกจนี้แม้แต่บรรทัดเดียว

### `internal/usecase/product_service.go`

```go
// Package usecase คือชั้น business logic ตาม Clean Architecture (Part 100)
// — พึ่งพาแค่ interface ใน internal/domain เท่านั้น ไม่รู้จัก GORM, chi,
// หรือ net/http เลย ทำให้เทสได้ง่ายด้วย fake/mock repository (Part 035)
// และสลับ implementation ของ repository (เช่นจาก SQLite เป็น PostgreSQL)
// โดยไม่กระทบโค้ดในแพ็กเกจนี้แม้แต่บรรทัดเดียว
package usecase

import (
	"context"
	"errors"
	"fmt"

	"ecommerce-api/internal/domain"
)

// ProductService รวม business logic ที่เกี่ยวกับสินค้าและหมวดหมู่
type ProductService struct {
	products   domain.ProductRepository
	categories domain.CategoryRepository
}

func NewProductService(products domain.ProductRepository, categories domain.CategoryRepository) *ProductService {
	return &ProductService{products: products, categories: categories}
}

const (
	defaultPageSize = 20
	maxPageSize     = 100
)

// List ค้นหาสินค้าตามเงื่อนไข พร้อม normalize ค่า pagination ให้อยู่ในช่วง
// ที่ปลอดภัยเสมอ (ป้องกัน client ส่ง page_size=100000 มาทำให้ query หนักเกินไป)
func (s *ProductService) List(ctx context.Context, filter domain.ProductFilter) ([]domain.Product, int64, error) {
	if filter.Page < 1 {
		filter.Page = 1
	}
	if filter.PageSize < 1 {
		filter.PageSize = defaultPageSize
	}
	if filter.PageSize > maxPageSize {
		filter.PageSize = maxPageSize
	}
	return s.products.List(ctx, filter)
}

// Get คืนสินค้าชิ้นเดียวตาม ID
func (s *ProductService) Get(ctx context.Context, id uint) (*domain.Product, error) {
	return s.products.GetByID(ctx, id)
}

// Create สร้างสินค้าใหม่ (เรียกจาก handler ที่ผ่าน role "admin" มาแล้วเท่านั้น)
func (s *ProductService) Create(ctx context.Context, p *domain.Product) error {
	if p.Name == "" {
		return fmt.Errorf("%w: name is required", domain.ErrValidation)
	}
	if p.PriceCents <= 0 {
		return fmt.Errorf("%w: price_cents must be greater than zero", domain.ErrValidation)
	}
	if p.Stock < 0 {
		return fmt.Errorf("%w: stock must not be negative", domain.ErrValidation)
	}
	if p.CategoryID != 0 {
		if _, err := s.categories.GetByID(ctx, p.CategoryID); err != nil {
			if errors.Is(err, domain.ErrNotFound) {
				return fmt.Errorf("%w: category_id %d does not exist", domain.ErrValidation, p.CategoryID)
			}
			return err
		}
	}
	return s.products.Create(ctx, p)
}

// ListCategories คืนหมวดหมู่ทั้งหมด
func (s *ProductService) ListCategories(ctx context.Context) ([]domain.Category, error) {
	return s.categories.List(ctx)
}

// CreateCategory สร้างหมวดหมู่ใหม่
func (s *ProductService) CreateCategory(ctx context.Context, c *domain.Category) error {
	if c.Name == "" || c.Slug == "" {
		return fmt.Errorf("%w: name and slug are required", domain.ErrValidation)
	}
	return s.categories.Create(ctx, c)
}
```

สังเกตว่า `fmt.Errorf("%w: ...", domain.ErrValidation)` (Part 016) ทำให้ error ที่คืนออกไปยัง**ทั้งเป็น** `domain.ErrValidation` (ตรวจด้วย `errors.Is` ได้) **และมี**ข้อความอธิบายเฉพาะเจาะจง (`"category_id 5 does not exist"`) ในเวลาเดียวกัน — handler ใช้ `errors.Is` ตัดสินใจ HTTP status code (422) ส่วน client เห็นข้อความเต็มที่อธิบายปัญหาจริง

---

## 7. ชั้น Usecase: Authentication ด้วย bcrypt + JWT

### `internal/usecase/auth_service.go`

```go
package usecase

import (
	"context"
	"errors"
	"fmt"
	"time"

	"github.com/golang-jwt/jwt/v5"
	"golang.org/x/crypto/bcrypt"

	"ecommerce-api/internal/domain"
)

// Claims คือ custom JWT claims ของแอปนี้ ผูก UserID และ Role เข้ากับ
// jwt.RegisteredClaims มาตรฐาน (แนวคิดจาก Part 067 หัวข้อ 5 — struct
// embedding เพื่อรวม registered claims กับ custom claims เป็นก้อนเดียว)
type Claims struct {
	UserID uint   `json:"uid"`
	Role   string `json:"role"`
	jwt.RegisteredClaims
}

// AuthService รวม business logic การสมัครสมาชิกและ login
type AuthService struct {
	users     domain.UserRepository
	jwtSecret []byte
	tokenTTL  time.Duration
}

func NewAuthService(users domain.UserRepository, jwtSecret []byte, tokenTTL time.Duration) *AuthService {
	return &AuthService{users: users, jwtSecret: jwtSecret, tokenTTL: tokenTTL}
}

// TokenTTL คืนอายุของ access token ที่ตั้งค่าไว้ ใช้สำหรับ handler ที่
// ต้องรายงานค่า expires_in กลับไปให้ client (ดู handler/auth_handler.go)
func (s *AuthService) TokenTTL() time.Duration {
	return s.tokenTTL
}

// Register สร้างผู้ใช้ใหม่ hash รหัสผ่านด้วย bcrypt ก่อนเก็บลงฐานข้อมูล
// เสมอ (ห้ามเก็บ plaintext เด็ดขาด ตามหลักการจาก Part 051)
func (s *AuthService) Register(ctx context.Context, name, email, password string) (*domain.User, error) {
	_, err := s.users.GetByEmail(ctx, email)
	if err == nil {
		return nil, domain.ErrEmailTaken
	}
	if !errors.Is(err, domain.ErrNotFound) {
		return nil, err
	}

	hash, err := bcrypt.GenerateFromPassword([]byte(password), bcrypt.DefaultCost)
	if err != nil {
		return nil, fmt.Errorf("hash password: %w", err)
	}

	u := &domain.User{
		Name:         name,
		Email:        email,
		PasswordHash: string(hash),
		Role:         domain.RoleCustomer,
	}
	if err := s.users.Create(ctx, u); err != nil {
		return nil, err
	}
	return u, nil
}

// Login ตรวจสอบ email/password แล้วออก JWT access token กลับไป
func (s *AuthService) Login(ctx context.Context, email, password string) (string, *domain.User, error) {
	u, err := s.users.GetByEmail(ctx, email)
	if err != nil {
		if errors.Is(err, domain.ErrNotFound) {
			// ตอบข้อความเดียวกันไม่ว่า "ไม่พบผู้ใช้" หรือ "รหัสผ่านผิด"
			// เพื่อป้องกัน user enumeration (Part 067 หัวข้อ 8)
			return "", nil, domain.ErrInvalidCredentials
		}
		return "", nil, err
	}

	if err := bcrypt.CompareHashAndPassword([]byte(u.PasswordHash), []byte(password)); err != nil {
		return "", nil, domain.ErrInvalidCredentials
	}

	token, err := s.generateToken(u)
	if err != nil {
		return "", nil, err
	}
	return token, u, nil
}

func (s *AuthService) generateToken(u *domain.User) (string, error) {
	now := time.Now()
	claims := Claims{
		UserID: u.ID,
		Role:   u.Role,
		RegisteredClaims: jwt.RegisteredClaims{
			Subject:   u.Email,
			IssuedAt:  jwt.NewNumericDate(now),
			ExpiresAt: jwt.NewNumericDate(now.Add(s.tokenTTL)),
			Issuer:    "ecommerce-api",
		},
	}
	token := jwt.NewWithClaims(jwt.SigningMethodHS256, claims)
	return token.SignedString(s.jwtSecret)
}

// ParseToken ตรวจสอบและถอดรหัส access token กลับเป็น Claims — ใช้จาก
// auth middleware ในชั้น handler (Part 067 หัวข้อ 6-7)
func (s *AuthService) ParseToken(tokenString string) (*Claims, error) {
	claims := &Claims{}
	token, err := jwt.ParseWithClaims(tokenString, claims, func(t *jwt.Token) (interface{}, error) {
		// ต้องตรวจสอบ signing method ก่อนคืน secret เสมอ เพื่อป้องกัน
		// algorithm confusion attack (Part 067 หัวข้อ 6)
		if _, ok := t.Method.(*jwt.SigningMethodHMAC); !ok {
			return nil, fmt.Errorf("unexpected signing method: %v", t.Header["alg"])
		}
		return s.jwtSecret, nil
	})
	if err != nil {
		return nil, err
	}
	if !token.Valid {
		return nil, errors.New("invalid token")
	}
	return claims, nil
}
```

ทุกอย่างในไฟล์นี้คือแนวคิดที่เรียนไปแล้วใน Part 051 และ Part 067 ทั้งสิ้น สิ่งใหม่คือการ**จัดวางให้เป็น struct ที่ inject repository เข้ามา** (`domain.UserRepository`) แทนการเขียนแบบ global function อย่างในบทเดี่ยวๆ ก่อนหน้า ทำให้ `AuthService` ทดสอบได้ง่ายด้วย fake repository โดยไม่ต้องมีฐานข้อมูลจริงเลย

---

## 8. ชั้น Usecase: การสั่งซื้อสินค้า

### `internal/usecase/order_service.go`

```go
package usecase

import (
	"context"
	"fmt"

	"ecommerce-api/internal/domain"
)

// PlaceOrderItem คือ input ดิบจาก client หนึ่งรายการสินค้า (ยังไม่รู้ราคา
// หรือชื่อสินค้า — usecase/repository จะไปดึงมาเองจากฐานข้อมูลเสมอ
// ไม่เชื่อราคาที่ client ส่งมาเด็ดขาด เพื่อป้องกันการปลอมแปลงราคา)
type PlaceOrderItem struct {
	ProductID uint
	Quantity  int
}

// OrderService รวม business logic การสั่งซื้อสินค้า
type OrderService struct {
	orders   domain.OrderRepository
	products domain.ProductRepository
}

func NewOrderService(orders domain.OrderRepository, products domain.ProductRepository) *OrderService {
	return &OrderService{orders: orders, products: products}
}

// PlaceOrder ตรวจสอบ input เบื้องต้นแล้วส่งต่อให้ repository ทำหน้าที่
// atomic transaction (ลด stock + สร้าง order พร้อมกัน — ดูรายละเอียดเต็ม
// ใน internal/repository/sqlite/order_repo.go ที่ผูกกับ Part 078)
func (s *OrderService) PlaceOrder(ctx context.Context, userID uint, items []PlaceOrderItem) (*domain.Order, error) {
	if len(items) == 0 {
		return nil, fmt.Errorf("%w: order must contain at least one item", domain.ErrValidation)
	}

	order := &domain.Order{UserID: userID, Status: domain.OrderStatusPaid}
	for _, it := range items {
		if it.Quantity <= 0 {
			return nil, fmt.Errorf("%w: quantity for product %d must be greater than zero", domain.ErrValidation, it.ProductID)
		}
		// ยืนยันว่าสินค้ามีอยู่จริงก่อนส่งเข้า transaction — ให้ error
		// message ที่ชัดเจนกว่าปล่อยให้ repository ล้มเหลวเงียบๆ ตอน
		// UpdateColumn คืน RowsAffected == 0 (ซึ่งอาจเป็นเพราะ id ไม่มีจริง
		// หรือ stock ไม่พอก็ได้ แยกกันไม่ออกถ้าไม่เช็คตรงนี้ก่อน)
		if _, err := s.products.GetByID(ctx, it.ProductID); err != nil {
			return nil, err
		}
		order.Items = append(order.Items, domain.OrderItem{
			ProductID: it.ProductID,
			Quantity:  it.Quantity,
		})
	}

	if err := s.orders.PlaceOrder(ctx, order); err != nil {
		return nil, err
	}
	return order, nil
}

// History คืนประวัติคำสั่งซื้อทั้งหมดของผู้ใช้คนหนึ่ง เรียงจากใหม่ไปเก่า
func (s *OrderService) History(ctx context.Context, userID uint) ([]domain.Order, error) {
	return s.orders.ListByUser(ctx, userID)
}

// Get คืนคำสั่งซื้อเดียวตาม ID — handler เป็นคนตรวจสอบว่าผู้เรียกมีสิทธิ์
// ดูคำสั่งซื้อนี้หรือไม่ (เจ้าของ order หรือ admin เท่านั้น)
func (s *OrderService) Get(ctx context.Context, id uint) (*domain.Order, error) {
	return s.orders.GetByID(ctx, id)
}
```

จุดที่สำคัญมากด้านความปลอดภัย: **`userID` มาจาก parameter ที่ handler ส่งเข้ามา (ซึ่งดึงจาก JWT claims) ไม่เคยมาจาก field ใน request body เลย** ถ้าออกแบบผิดให้ client ส่ง `user_id` มาในตัว JSON body ของคำสั่งซื้อ ผู้ใช้ A จะสามารถปลอมตัวสั่งซื้อในนามผู้ใช้ B ได้ทันทีแค่เปลี่ยนตัวเลขใน body — เป็นช่องโหว่คลาสสิกที่เรียกว่า **Broken Object Level Authorization (BOLA)** ซึ่งเป็นหนึ่งใน OWASP API Security Top 10

---

## 9. ชั้น Handler: JSON Envelope, Validation, Error Mapping

ชั้น `internal/handler` คือ "delivery layer" — แปลง HTTP request เป็นการเรียก usecase แล้วแปลงผลลัพธ์กลับเป็น HTTP response เท่านั้น ไม่มี business logic ในนี้เลย

### `internal/handler/response.go`

```go
// Package handler คือชั้น delivery (Part 100) — แปลง HTTP request เป็น
// เรียก usecase แล้วแปลงผลลัพธ์กลับเป็น HTTP response เท่านั้น ไม่มี
// business logic อยู่ในนี้เลย (business logic ทั้งหมดอยู่ใน internal/usecase)
package handler

import (
	"encoding/json"
	"errors"
	"io"
	"log"
	"net/http"
	"strings"

	"github.com/go-playground/validator/v10"

	"ecommerce-api/internal/domain"
)

// ErrorDetail/ErrorResponse คือ error envelope มาตรฐานของทั้ง API
// (ออกแบบตามหลักการจาก Part 059 หัวข้อ 9 และใช้ซ้ำแบบเดียวกับ Part 060)
type ErrorDetail struct {
	Message string            `json:"message"`
	Fields  map[string]string `json:"fields,omitempty"`
}

type ErrorResponse struct {
	Error ErrorDetail `json:"error"`
}

// Meta คือข้อมูลประกอบการแบ่งหน้า แนบไปกับ response ที่เป็น list เสมอ
// (แนวคิด offset-based pagination จาก Part 059 หัวข้อ 6)
type Meta struct {
	Page     int   `json:"page"`
	PageSize int   `json:"page_size"`
	Total    int64 `json:"total"`
}

// ListResponse คือ envelope มาตรฐานสำหรับ response ที่เป็นรายการ (list)
type ListResponse struct {
	Data any  `json:"data"`
	Meta Meta `json:"meta"`
}

// respondJSON เขียน response กลับเป็น JSON พร้อม status code ที่กำหนด
// เป็น helper กลางเดียวที่ handler ทุกตัวเรียก — ไม่มี handler ไหนแตะ
// http.ResponseWriter ตรงๆ อีกเลย ตามแนวทางจาก Part 060 หัวข้อ 2
func respondJSON(w http.ResponseWriter, status int, v any) {
	w.Header().Set("Content-Type", "application/json; charset=utf-8")
	w.WriteHeader(status)
	if v == nil {
		return
	}
	if err := json.NewEncoder(w).Encode(v); err != nil {
		log.Printf("handler: respondJSON encode error: %v", err)
	}
}

func respondError(w http.ResponseWriter, status int, message string, fields map[string]string) {
	respondJSON(w, status, ErrorResponse{Error: ErrorDetail{Message: message, Fields: fields}})
}

// respondDomainError แปลง error จากชั้น usecase/domain เป็น HTTP status
// code ที่เหมาะสม โดยใช้ errors.Is ตรวจสอบ sentinel error (Part 016)
// เป็นจุดเดียวในทั้งแอปที่ผูก "domain error" เข้ากับ "HTTP status code"
// — เหตุผลที่ยัดไว้ที่ handler ไม่ใช่ usecase เพราะ usecase ไม่ควรรู้จัก
// HTTP status code เลยตามหลัก Clean Architecture (Part 100)
func respondDomainError(w http.ResponseWriter, err error) {
	switch {
	case errors.Is(err, domain.ErrNotFound):
		respondError(w, http.StatusNotFound, err.Error(), nil)
	case errors.Is(err, domain.ErrEmailTaken):
		respondError(w, http.StatusConflict, err.Error(), nil)
	case errors.Is(err, domain.ErrInvalidCredentials):
		respondError(w, http.StatusUnauthorized, err.Error(), nil)
	case errors.Is(err, domain.ErrInsufficientStock):
		respondError(w, http.StatusConflict, err.Error(), nil)
	case errors.Is(err, domain.ErrValidation):
		respondError(w, http.StatusUnprocessableEntity, err.Error(), nil)
	default:
		log.Printf("handler: unexpected error: %v", err)
		respondError(w, http.StatusInternalServerError, "internal server error", nil)
	}
}

// decodeJSON อ่าน JSON body ลงใน dst พร้อมจับ error หลายแบบให้ข้อความ
// อ่านง่าย (เทคนิคเดียวกับ Part 060 หัวข้อ 4) และปฏิเสธ field แปลกปลอม
// ด้วย DisallowUnknownFields เพื่อจับ typo ของ client ตั้งแต่ต้น
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
	if errors.Is(err, io.EOF) {
		return errors.New("request body must not be empty")
	}
	return err
}

// validationFields แปลง validator.ValidationErrors (Part 060 หัวข้อ 6-7)
// เป็น map[field]message ที่อ่านง่าย สำหรับใส่ใน ErrorResponse.Fields
func validationFields(err error) map[string]string {
	fields := map[string]string{}
	var verrs validator.ValidationErrors
	if errors.As(err, &verrs) {
		for _, fe := range verrs {
			key := strings.ToLower(fe.Field())
			switch fe.Tag() {
			case "required":
				fields[key] = "field is required"
			case "min":
				fields[key] = "must be at least " + fe.Param()
			case "max":
				fields[key] = "must be at most " + fe.Param()
			case "email":
				fields[key] = "must be a valid email address"
			default:
				fields[key] = "failed on \"" + fe.Tag() + "\" validation"
			}
		}
	}
	return fields
}
```

ตารางสรุปการแม็ป domain error → HTTP status code ที่ `respondDomainError` ทำ:

| Domain Error | HTTP Status | ความหมาย |
|---|---|---|
| `domain.ErrNotFound` | 404 Not Found | ไม่พบ resource |
| `domain.ErrEmailTaken` | 409 Conflict | อีเมลถูกใช้ไปแล้ว |
| `domain.ErrInvalidCredentials` | 401 Unauthorized | login ผิด |
| `domain.ErrInsufficientStock` | 409 Conflict | stock ไม่พอ (ขัดแย้งกับสถานะปัจจุบันของระบบ) |
| `domain.ErrValidation` | 422 Unprocessable Entity | input ผิดกฎธุรกิจ |
| อื่นๆ ที่ไม่รู้จัก | 500 Internal Server Error | บั๊กที่ไม่คาดคิด — log ไว้เสมอ |

การเลือก 409 Conflict สำหรับทั้ง "อีเมลซ้ำ" และ "stock ไม่พอ" (แทนที่จะใช้ 400 ไปหมด) ตรงตามหลักการที่วางไว้ใน **Part 059 หัวข้อ 5**: 409 คือ status ที่ถูกต้องเมื่อ request ที่ส่งมาถูกต้องตามรูปแบบ แต่ **ขัดแย้งกับสถานะปัจจุบันของทรัพยากรในระบบ** ต่างจาก 400/422 ที่หมายถึง request มีปัญหาในตัวมันเอง

---

## 10. ชั้น Handler: Auth Middleware และ Role-Based Access Control

### `internal/handler/middleware.go`

```go
package handler

import (
	"context"
	"net/http"
	"strings"

	"ecommerce-api/internal/usecase"
)

type ctxKey string

const claimsContextKey ctxKey = "claims"

// AuthMiddleware ตรวจสอบ header "Authorization: Bearer <token>" แล้วฝัง
// claims ที่ถอดรหัสได้ไว้ใน context ของ request (แนวคิดจาก Part 067
// หัวข้อ 7 ผสม Part 032 เรื่อง context) endpoint ไหนอยากบังคับ login
// ก็ครอบ handler ของตัวเองด้วย middleware ตัวนี้
func AuthMiddleware(auth *usecase.AuthService) func(http.Handler) http.Handler {
	return func(next http.Handler) http.Handler {
		return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
			authHeader := r.Header.Get("Authorization")
			const prefix = "Bearer "
			if authHeader == "" || !strings.HasPrefix(authHeader, prefix) {
				respondError(w, http.StatusUnauthorized, "missing or malformed Authorization header", nil)
				return
			}
			tokenString := strings.TrimPrefix(authHeader, prefix)

			claims, err := auth.ParseToken(tokenString)
			if err != nil {
				respondError(w, http.StatusUnauthorized, "invalid or expired token", nil)
				return
			}

			ctx := context.WithValue(r.Context(), claimsContextKey, claims)
			next.ServeHTTP(w, r.WithContext(ctx))
		})
	}
}

// claimsFromContext ดึง claims ที่ AuthMiddleware ฝังไว้กลับออกมา
func claimsFromContext(r *http.Request) (*usecase.Claims, bool) {
	claims, ok := r.Context().Value(claimsContextKey).(*usecase.Claims)
	return claims, ok
}

// RequireRole คือ middleware เสริมที่ต้องต่อจาก AuthMiddleware เท่านั้น
// (ต้องมี claims ในบริบทอยู่ก่อนแล้ว) ใช้จำกัดว่า endpoint นี้อนุญาตเฉพาะ
// role ที่ระบุ — รับ list ของ role ที่อนุญาตแบบ variadic ตามแนวทางที่
// แนะนำไว้ในแบบฝึกหัดของ Part 067 (requireRole("admin", "editor"))
func RequireRole(roles ...string) func(http.Handler) http.Handler {
	allowed := make(map[string]bool, len(roles))
	for _, r := range roles {
		allowed[r] = true
	}
	return func(next http.Handler) http.Handler {
		return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
			claims, ok := claimsFromContext(r)
			if !ok {
				respondError(w, http.StatusUnauthorized, "authentication required", nil)
				return
			}
			if !allowed[claims.Role] {
				respondError(w, http.StatusForbidden, "you do not have permission to perform this action", nil)
				return
			}
			next.ServeHTTP(w, r)
		})
	}
}
```

การแยก `AuthMiddleware` (ตรวจว่า "เป็นใคร" — authentication) ออกจาก `RequireRole` (ตรวจว่า "ทำอะไรได้บ้าง" — authorization) เป็นคนละ middleware กัน ทำให้ผสมกันได้อย่างยืดหยุ่น: บาง endpoint ต้อง login แต่ไม่จำกัด role (`/orders`), บาง endpoint ต้อง login **และ** ต้องเป็น admin (`POST /products`) — ตรงกับหลักการที่เน้นย้ำใน **Part 067 หัวข้อ 10 ข้อ 7**: authentication กับ authorization เป็นคนละเรื่องกัน ต้องแยกตรวจสอบเสมอ

---

## 11. ชั้น Handler: Product, Order Handler และ Router ด้วย chi

### `internal/handler/product_handler.go`

```go
package handler

import (
	"net/http"
	"strconv"

	"github.com/go-chi/chi/v5"
	"github.com/go-playground/validator/v10"

	"ecommerce-api/internal/domain"
	"ecommerce-api/internal/usecase"
)

// ProductHandler ผูก HTTP endpoint ของสินค้า/หมวดหมู่เข้ากับ ProductService
type ProductHandler struct {
	products *usecase.ProductService
	validate *validator.Validate
}

func NewProductHandler(products *usecase.ProductService, validate *validator.Validate) *ProductHandler {
	return &ProductHandler{products: products, validate: validate}
}

// atoiDefault แปลง query parameter เป็น int โดยคืนค่า default ถ้าแปลงไม่ได้
// หรือไม่มีค่าส่งมา (ไม่ตอบ 400 เพราะ pagination parameter ที่แปลงไม่ได้
// ไม่ควรทำให้ทั้ง request ล้มเหลว แค่ fallback กลับไปใช้ค่าเริ่มต้นก็พอ)
func atoiDefault(s string, def int) int {
	if s == "" {
		return def
	}
	n, err := strconv.Atoi(s)
	if err != nil {
		return def
	}
	return n
}

// List จัดการ GET /products?search=...&category_id=...&page=...&page_size=...
func (h *ProductHandler) List(w http.ResponseWriter, r *http.Request) {
	q := r.URL.Query()

	filter := domain.ProductFilter{
		Search:   q.Get("search"),
		Page:     atoiDefault(q.Get("page"), 1),
		PageSize: atoiDefault(q.Get("page_size"), 20),
	}
	if catID := atoiDefault(q.Get("category_id"), 0); catID > 0 {
		filter.CategoryID = uint(catID)
	}

	products, total, err := h.products.List(r.Context(), filter)
	if err != nil {
		respondDomainError(w, err)
		return
	}

	respondJSON(w, http.StatusOK, ListResponse{
		Data: products,
		Meta: Meta{Page: filter.Page, PageSize: filter.PageSize, Total: total},
	})
}

// Get จัดการ GET /products/{id}
func (h *ProductHandler) Get(w http.ResponseWriter, r *http.Request) {
	id, err := strconv.ParseUint(chi.URLParam(r, "id"), 10, 64)
	if err != nil {
		respondError(w, http.StatusBadRequest, "invalid product id", nil)
		return
	}

	p, err := h.products.Get(r.Context(), uint(id))
	if err != nil {
		respondDomainError(w, err)
		return
	}
	respondJSON(w, http.StatusOK, p)
}

type createProductRequest struct {
	Name        string `json:"name" validate:"required,min=1,max=200"`
	Description string `json:"description" validate:"max=2000"`
	PriceCents  int64  `json:"price_cents" validate:"required,gt=0"`
	Stock       int    `json:"stock" validate:"gte=0"`
	CategoryID  uint   `json:"category_id"`
}

// Create จัดการ POST /products (ต้องมี role "admin" — บังคับผ่าน
// RequireRole middleware ที่ router.go ผูกไว้ก่อนถึง handler นี้)
func (h *ProductHandler) Create(w http.ResponseWriter, r *http.Request) {
	var req createProductRequest
	if err := decodeJSON(r, &req); err != nil {
		respondError(w, http.StatusBadRequest, err.Error(), nil)
		return
	}
	if err := h.validate.Struct(req); err != nil {
		respondError(w, http.StatusUnprocessableEntity, "validation failed", validationFields(err))
		return
	}

	p := &domain.Product{
		Name:        req.Name,
		Description: req.Description,
		PriceCents:  req.PriceCents,
		Stock:       req.Stock,
		CategoryID:  req.CategoryID,
	}
	if err := h.products.Create(r.Context(), p); err != nil {
		respondDomainError(w, err)
		return
	}
	respondJSON(w, http.StatusCreated, p)
}

// ListCategories จัดการ GET /categories
func (h *ProductHandler) ListCategories(w http.ResponseWriter, r *http.Request) {
	categories, err := h.products.ListCategories(r.Context())
	if err != nil {
		respondDomainError(w, err)
		return
	}
	respondJSON(w, http.StatusOK, categories)
}

type createCategoryRequest struct {
	Name string `json:"name" validate:"required,min=1,max=150"`
	Slug string `json:"slug" validate:"required,min=1,max=150"`
}

// CreateCategory จัดการ POST /categories (ต้องมี role "admin")
func (h *ProductHandler) CreateCategory(w http.ResponseWriter, r *http.Request) {
	var req createCategoryRequest
	if err := decodeJSON(r, &req); err != nil {
		respondError(w, http.StatusBadRequest, err.Error(), nil)
		return
	}
	if err := h.validate.Struct(req); err != nil {
		respondError(w, http.StatusUnprocessableEntity, "validation failed", validationFields(err))
		return
	}

	c := &domain.Category{Name: req.Name, Slug: req.Slug}
	if err := h.products.CreateCategory(r.Context(), c); err != nil {
		respondDomainError(w, err)
		return
	}
	respondJSON(w, http.StatusCreated, c)
}
```

### `internal/handler/auth_handler.go`

```go
package handler

import (
	"errors"
	"net/http"

	"github.com/go-playground/validator/v10"

	"ecommerce-api/internal/domain"
	"ecommerce-api/internal/usecase"
)

// AuthHandler ผูก HTTP endpoint ของการสมัครสมาชิก/login เข้ากับ AuthService
type AuthHandler struct {
	auth     *usecase.AuthService
	validate *validator.Validate
}

func NewAuthHandler(auth *usecase.AuthService, validate *validator.Validate) *AuthHandler {
	return &AuthHandler{auth: auth, validate: validate}
}

type registerRequest struct {
	Name     string `json:"name" validate:"required,min=1,max=150"`
	Email    string `json:"email" validate:"required,email"`
	Password string `json:"password" validate:"required,min=8,max=72"`
}

type userResponse struct {
	ID    uint   `json:"id"`
	Name  string `json:"name"`
	Email string `json:"email"`
	Role  string `json:"role"`
}

func toUserResponse(u *domain.User) userResponse {
	return userResponse{ID: u.ID, Name: u.Name, Email: u.Email, Role: u.Role}
}

// Register จัดการ POST /auth/register
func (h *AuthHandler) Register(w http.ResponseWriter, r *http.Request) {
	var req registerRequest
	if err := decodeJSON(r, &req); err != nil {
		respondError(w, http.StatusBadRequest, err.Error(), nil)
		return
	}
	if err := h.validate.Struct(req); err != nil {
		respondError(w, http.StatusUnprocessableEntity, "validation failed", validationFields(err))
		return
	}

	u, err := h.auth.Register(r.Context(), req.Name, req.Email, req.Password)
	if err != nil {
		respondDomainError(w, err)
		return
	}
	respondJSON(w, http.StatusCreated, toUserResponse(u))
}

type loginRequest struct {
	Email    string `json:"email" validate:"required,email"`
	Password string `json:"password" validate:"required"`
}

type loginResponse struct {
	AccessToken string       `json:"access_token"`
	TokenType   string       `json:"token_type"`
	ExpiresIn   int          `json:"expires_in"`
	User        userResponse `json:"user"`
}

// Login จัดการ POST /auth/login
func (h *AuthHandler) Login(w http.ResponseWriter, r *http.Request) {
	var req loginRequest
	if err := decodeJSON(r, &req); err != nil {
		respondError(w, http.StatusBadRequest, err.Error(), nil)
		return
	}
	if err := h.validate.Struct(req); err != nil {
		respondError(w, http.StatusUnprocessableEntity, "validation failed", validationFields(err))
		return
	}

	token, u, err := h.auth.Login(r.Context(), req.Email, req.Password)
	if err != nil {
		if errors.Is(err, domain.ErrInvalidCredentials) {
			respondError(w, http.StatusUnauthorized, err.Error(), nil)
			return
		}
		respondDomainError(w, err)
		return
	}

	respondJSON(w, http.StatusOK, loginResponse{
		AccessToken: token,
		TokenType:   "Bearer",
		ExpiresIn:   int(h.auth.TokenTTL().Seconds()),
		User:        toUserResponse(u),
	})
}
```

### `internal/handler/order_handler.go`

```go
package handler

import (
	"net/http"
	"strconv"

	"github.com/go-chi/chi/v5"
	"github.com/go-playground/validator/v10"

	"ecommerce-api/internal/domain"
	"ecommerce-api/internal/usecase"
)

// OrderHandler ผูก HTTP endpoint ของคำสั่งซื้อเข้ากับ OrderService — ทุก
// endpoint ในไฟล์นี้ต้องผ่าน AuthMiddleware มาก่อนเสมอ (ดู router.go)
// เพราะการสั่งซื้อ/ดูประวัติต้องรู้ว่า "ใคร" เป็นคนเรียก
type OrderHandler struct {
	orders   *usecase.OrderService
	validate *validator.Validate
}

func NewOrderHandler(orders *usecase.OrderService, validate *validator.Validate) *OrderHandler {
	return &OrderHandler{orders: orders, validate: validate}
}

type placeOrderItemRequest struct {
	ProductID uint `json:"product_id" validate:"required"`
	Quantity  int  `json:"quantity" validate:"required,gt=0"`
}

type placeOrderRequest struct {
	Items []placeOrderItemRequest `json:"items" validate:"required,min=1,dive"`
}

// Create จัดการ POST /orders — ผู้ใช้ที่ login แล้วเท่านั้นที่สั่งซื้อได้
// user id ของผู้สั่งซื้อ **ไม่เคย** อ่านจาก request body แต่อ่านจาก JWT
// claims เสมอ (ป้องกัน client ปลอมตัวสั่งซื้อในนามคนอื่น)
func (h *OrderHandler) Create(w http.ResponseWriter, r *http.Request) {
	claims, ok := claimsFromContext(r)
	if !ok {
		respondError(w, http.StatusUnauthorized, "authentication required", nil)
		return
	}

	var req placeOrderRequest
	if err := decodeJSON(r, &req); err != nil {
		respondError(w, http.StatusBadRequest, err.Error(), nil)
		return
	}
	if err := h.validate.Struct(req); err != nil {
		respondError(w, http.StatusUnprocessableEntity, "validation failed", validationFields(err))
		return
	}

	items := make([]usecase.PlaceOrderItem, 0, len(req.Items))
	for _, it := range req.Items {
		items = append(items, usecase.PlaceOrderItem{ProductID: it.ProductID, Quantity: it.Quantity})
	}

	order, err := h.orders.PlaceOrder(r.Context(), claims.UserID, items)
	if err != nil {
		respondDomainError(w, err)
		return
	}
	respondJSON(w, http.StatusCreated, order)
}

// History จัดการ GET /orders — คืนประวัติคำสั่งซื้อของผู้ใช้ที่ login อยู่เท่านั้น
func (h *OrderHandler) History(w http.ResponseWriter, r *http.Request) {
	claims, ok := claimsFromContext(r)
	if !ok {
		respondError(w, http.StatusUnauthorized, "authentication required", nil)
		return
	}

	orders, err := h.orders.History(r.Context(), claims.UserID)
	if err != nil {
		respondDomainError(w, err)
		return
	}
	respondJSON(w, http.StatusOK, ListResponse{
		Data: orders,
		Meta: Meta{Page: 1, PageSize: len(orders), Total: int64(len(orders))},
	})
}

// Get จัดการ GET /orders/{id} — เจ้าของคำสั่งซื้อหรือ admin เท่านั้นที่ดูได้
// (authorization ระดับ record ตรวจใน handler เพราะต้องรู้ทั้ง claims และ
// ตัว resource จริงก่อนตัดสินใจได้ — คนละเรื่องกับ RequireRole ที่ตรวจ
// จาก role อย่างเดียวโดยไม่ต้องรู้จัก resource เลย)
func (h *OrderHandler) Get(w http.ResponseWriter, r *http.Request) {
	claims, ok := claimsFromContext(r)
	if !ok {
		respondError(w, http.StatusUnauthorized, "authentication required", nil)
		return
	}

	id, err := strconv.ParseUint(chi.URLParam(r, "id"), 10, 64)
	if err != nil {
		respondError(w, http.StatusBadRequest, "invalid order id", nil)
		return
	}

	order, err := h.orders.Get(r.Context(), uint(id))
	if err != nil {
		respondDomainError(w, err)
		return
	}
	if order.UserID != claims.UserID && claims.Role != domain.RoleAdmin {
		respondError(w, http.StatusForbidden, "you do not have permission to view this order", nil)
		return
	}
	respondJSON(w, http.StatusOK, order)
}
```

### `internal/handler/router.go`

ใช้ **chi** (Part 058) แทน stdlib `ServeMux` เปล่าๆ เพราะโปรเจกต์นี้มี middleware เฉพาะทางหลายชั้น (auth, role-based) ที่ต้องผูกเฉพาะบางกลุ่ม route เท่านั้น (เช่น `/orders` ทั้งหมดต้อง auth แต่ `/products` ดูได้แบบไม่ต้อง login ยกเว้น `POST` ที่ต้อง auth และ admin) — `r.Route`/`r.Group` ของ chi จัดการ nested middleware กลุ่มนี้ได้สะอาดกว่าการเขียน `if` เช็ค path เองใน stdlib มาก ตามเหตุผลเปรียบเทียบเต็มๆ ใน **Part 058 หัวข้อ 11**

```go
package handler

import (
	"net/http"

	"github.com/go-chi/chi/v5"
	"github.com/go-chi/chi/v5/middleware"
	"github.com/go-playground/validator/v10"

	"ecommerce-api/internal/domain"
	"ecommerce-api/internal/usecase"
)

// NewRouter ประกอบทุก handler เข้าเป็น chi.Router เดียว
func NewRouter(
	auth *usecase.AuthService,
	products *usecase.ProductService,
	orders *usecase.OrderService,
) http.Handler {
	validate := validator.New()

	authH := NewAuthHandler(auth, validate)
	productH := NewProductHandler(products, validate)
	orderH := NewOrderHandler(orders, validate)

	r := chi.NewRouter()
	r.Use(middleware.Logger)
	r.Use(middleware.Recoverer)

	r.Get("/healthz", func(w http.ResponseWriter, r *http.Request) {
		respondJSON(w, http.StatusOK, map[string]string{"status": "ok"})
	})

	r.Route("/auth", func(r chi.Router) {
		r.Post("/register", authH.Register)
		r.Post("/login", authH.Login)
	})

	r.Route("/products", func(r chi.Router) {
		r.Get("/", productH.List)
		r.Get("/{id}", productH.Get)

		// สร้างสินค้าใหม่ต้อง login ก่อน (AuthMiddleware) และต้องเป็น
		// admin เท่านั้น (RequireRole) — ต่อ middleware สองชั้นด้วย .Use()
		r.Group(func(r chi.Router) {
			r.Use(AuthMiddleware(auth))
			r.Use(RequireRole(domain.RoleAdmin))
			r.Post("/", productH.Create)
		})
	})

	r.Route("/categories", func(r chi.Router) {
		r.Get("/", productH.ListCategories)
		r.Group(func(r chi.Router) {
			r.Use(AuthMiddleware(auth))
			r.Use(RequireRole(domain.RoleAdmin))
			r.Post("/", productH.CreateCategory)
		})
	})

	r.Route("/orders", func(r chi.Router) {
		// ทุก endpoint ใต้ /orders ต้อง login เสมอ (ไม่มีใครดูคำสั่งซื้อ
		// ของตัวเองหรือสั่งซื้อได้โดยไม่ยืนยันตัวตน)
		r.Use(AuthMiddleware(auth))
		r.Post("/", orderH.Create)
		r.Get("/", orderH.History)
		r.Get("/{id}", orderH.Get)
	})

	return r
}
```

ตาราง endpoint ทั้งหมดของ API:

| Method | Path | ต้อง login | ต้องเป็น admin | คำอธิบาย |
|---|---|---|---|---|
| GET | `/healthz` | ไม่ | ไม่ | health check |
| POST | `/auth/register` | ไม่ | ไม่ | สมัครสมาชิก |
| POST | `/auth/login` | ไม่ | ไม่ | เข้าสู่ระบบ รับ JWT |
| GET | `/products` | ไม่ | ไม่ | ค้นหา/ดูรายการสินค้า |
| GET | `/products/{id}` | ไม่ | ไม่ | ดูสินค้าชิ้นเดียว |
| POST | `/products` | ต้อง | ต้อง | สร้างสินค้าใหม่ |
| GET | `/categories` | ไม่ | ไม่ | ดูหมวดหมู่ทั้งหมด |
| POST | `/categories` | ต้อง | ต้อง | สร้างหมวดหมู่ใหม่ |
| POST | `/orders` | ต้อง | ไม่ | สั่งซื้อสินค้า |
| GET | `/orders` | ต้อง | ไม่ | ดูประวัติคำสั่งซื้อของตัวเอง |
| GET | `/orders/{id}` | ต้อง | เฉพาะเจ้าของ/admin | ดูคำสั่งซื้อเดียว |

---

## 12. `cmd/api/main.go`: ประกอบร่างทั้งหมดและ Graceful Shutdown

`main.go` มีหน้าที่เดียว: **ประกอบร่าง (wiring)** ทุกชั้นเข้าด้วยกัน — สร้าง repository จริง (`sqlite.*`) แล้วส่งเข้า usecase แล้วส่ง usecase เข้า handler ไม่มี business logic ปนอยู่เลยแม้แต่บรรทัดเดียว

```go
// Command api คือ entry point ของ E-Commerce REST API (Part 103)
//
// โครงสร้างโปรเจกต์นี้ยึดตาม golang-standards/project-layout ที่แนะนำไว้
// ตั้งแต่ Part 001 และปรับใช้ Clean Architecture (Part 100) อย่างเคร่งครัด:
//
//	cmd/api          -> จุดเริ่มโปรแกรม (main package), มีหน้าที่ "ประกอบ
//	                     ร่าง" (wiring) ทุกชั้นเข้าด้วยกันเท่านั้น
//	internal/domain     -> entity + repository interface (ชั้นในสุด)
//	internal/usecase    -> business logic (ขึ้นกับ domain เท่านั้น)
//	internal/repository -> GORM/SQLite adapter (implement domain interface)
//	internal/handler    -> HTTP delivery ด้วย chi (ขึ้นกับ usecase เท่านั้น)
//
// ทิศทางการ import ไหลเข้าหา domain เสมอ ไม่มีทางย้อนกลับ — usecase ไม่
// import handler, domain ไม่ import อะไรจากที่อื่นในโปรเจกต์เลย
package main

import (
	"context"
	"errors"
	"log"
	"net/http"
	"os"
	"os/signal"
	"syscall"
	"time"

	"ecommerce-api/internal/domain"
	"ecommerce-api/internal/handler"
	"ecommerce-api/internal/repository/sqlite"
	"ecommerce-api/internal/usecase"
)

func main() {
	dbPath := getEnv("DB_PATH", "ecommerce.db")
	addr := ":" + getEnv("PORT", "8080")
	jwtSecret := []byte(getEnv("JWT_SECRET", "dev-secret-change-me-in-production"))

	db, err := sqlite.Open(dbPath)
	if err != nil {
		log.Fatalf("open database: %v", err)
	}

	categoryRepo := sqlite.NewCategoryRepository(db)
	productRepo := sqlite.NewProductRepository(db)
	userRepo := sqlite.NewUserRepository(db)
	orderRepo := sqlite.NewOrderRepository(db)

	authService := usecase.NewAuthService(userRepo, jwtSecret, 24*time.Hour)
	productService := usecase.NewProductService(productRepo, categoryRepo)
	orderService := usecase.NewOrderService(orderRepo, productRepo)

	if err := seedDemoData(productService); err != nil {
		log.Fatalf("seed demo data: %v", err)
	}

	router := handler.NewRouter(authService, productService, orderService)

	srv := &http.Server{
		Addr:              addr,
		Handler:           router,
		ReadHeaderTimeout: 5 * time.Second,
	}

	go func() {
		log.Printf("ecommerce-api listening on %s (db: %s)", addr, dbPath)
		if err := srv.ListenAndServe(); err != nil && !errors.Is(err, http.ErrServerClosed) {
			log.Fatalf("server error: %v", err)
		}
	}()

	// รอ signal เพื่อทำ graceful shutdown แทนการ kill ทันที (สำคัญเพราะ
	// request ที่กำลังเขียน transaction อยู่ไม่ควรถูกตัดตอนกลางคัน)
	stop := make(chan os.Signal, 1)
	signal.Notify(stop, os.Interrupt, syscall.SIGTERM)
	<-stop

	log.Println("shutting down gracefully...")
	ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
	defer cancel()
	if err := srv.Shutdown(ctx); err != nil {
		log.Printf("shutdown error: %v", err)
	}
}

func getEnv(key, fallback string) string {
	if v := os.Getenv(key); v != "" {
		return v
	}
	return fallback
}

// seedDemoData ใส่หมวดหมู่และสินค้าตัวอย่างไว้ตอนเริ่มระบบ ถ้ายังไม่มี
// หมวดหมู่ใดๆ ในฐานข้อมูลเลย (idempotent — รันซ้ำกี่ครั้งก็ไม่สร้างข้อมูล
// ซ้ำ) เพื่อให้ทดลองยิง curl ตามหัวข้อ "วิธีรันด้วยตัวเอง" ได้ทันที
func seedDemoData(products *usecase.ProductService) error {
	ctx := context.Background()
	existing, err := products.ListCategories(ctx)
	if err != nil {
		return err
	}
	if len(existing) > 0 {
		return nil
	}

	electronics := &domain.Category{Name: "Electronics", Slug: "electronics"}
	books := &domain.Category{Name: "Books", Slug: "books"}
	if err := products.CreateCategory(ctx, electronics); err != nil {
		return err
	}
	if err := products.CreateCategory(ctx, books); err != nil {
		return err
	}

	demo := []domain.Product{
		{Name: "Mechanical Keyboard", Description: "Hot-swappable 75% keyboard", PriceCents: 289900, Stock: 15, CategoryID: electronics.ID},
		{Name: "Wireless Mouse", Description: "Ergonomic wireless mouse", PriceCents: 79900, Stock: 40, CategoryID: electronics.ID},
		{Name: "The Go Programming Language", Description: "Donovan & Kernighan", PriceCents: 129900, Stock: 25, CategoryID: books.ID},
	}
	for i := range demo {
		if err := products.Create(ctx, &demo[i]); err != nil {
			return err
		}
	}
	return nil
}
```

การรอ `os.Interrupt`/`syscall.SIGTERM` แล้วเรียก `srv.Shutdown(ctx)` แทนการปล่อยให้โปรแกรมถูก kill ทันที คือ **graceful shutdown** — สำคัญมากในระบบที่มี transaction เพราะถ้า process ถูกฆ่ากลางคันขณะ `PlaceOrder` กำลังรันอยู่ (ระหว่าง `BEGIN` กับ `COMMIT`) SQLite/PostgreSQL จะ rollback transaction ที่ค้างอยู่ให้เองตามกลไก ACID ที่เรียนใน Part 078 อยู่แล้ว แต่การให้เวลา request ที่กำลังทำงานอยู่จบตามปกติก่อนปิดเซิร์ฟเวอร์ (`srv.Shutdown` รอ request ที่ยังไม่เสร็จ) ยังคงเป็นแนวทางที่ปลอดภัยและสุภาพกว่าเสมอ

---

## 13. รันจริงและทดสอบด้วย `curl` แบบ End-to-End

รันเซิร์ฟเวอร์:

```bash
go build ./...
go vet ./...
PORT=8099 go run ./cmd/api
```

ผลลัพธ์จริง:

```
ecommerce-api listening on :8099 (db: ecommerce.db)
```

### Health check และดูสินค้าที่ seed มาให้

```bash
curl -s http://localhost:8099/healthz
```
```json
{"status":"ok"}
```

```bash
curl -s http://localhost:8099/categories
```
```json
[{"id":1,"name":"Electronics","slug":"electronics"},{"id":2,"name":"Books","slug":"books"}]
```

```bash
curl -s "http://localhost:8099/products?page=1&page_size=10"
```
```json
{"data":[{"id":1,"name":"Mechanical Keyboard","description":"Hot-swappable 75% keyboard","price_cents":289900,"stock":15,"category_id":1,"category":{"id":1,"name":"Electronics","slug":"electronics"},"created_at":"2026-09-26T06:38:02.564797219Z","updated_at":"2026-09-26T06:38:02.564797219Z"},{"id":2,"name":"Wireless Mouse","description":"Ergonomic wireless mouse","price_cents":79900,"stock":40,"category_id":1,"category":{"id":1,"name":"Electronics","slug":"electronics"},"created_at":"2026-09-26T06:38:02.565741459Z","updated_at":"2026-09-26T06:38:02.565741459Z"},{"id":3,"name":"The Go Programming Language","description":"Donovan & Kernighan","price_cents":129900,"stock":25,"category_id":2,"category":{"id":2,"name":"Books","slug":"books"},"created_at":"2026-09-26T06:38:02.566573652Z","updated_at":"2026-09-26T06:38:02.566573652Z"}],"meta":{"page":1,"page_size":10,"total":3}}
```

### สมัครสมาชิกและ login

```bash
curl -s -X POST http://localhost:8099/auth/register \
  -H "Content-Type: application/json" \
  -d '{"name":"Nattapong","email":"nattapong@example.com","password":"s3cretpass"}'
```
```json
{"id":1,"name":"Nattapong","email":"nattapong@example.com","role":"customer"}
```

```bash
curl -s -X POST http://localhost:8099/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email":"nattapong@example.com","password":"s3cretpass"}'
```
```json
{"access_token":"eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1aWQiOjEsInJvbGUiOiJjdXN0b21lciIsImlzcyI6ImVjb21tZXJjZS1hcGkiLCJzdWIiOiJuYXR0YXBvbmdAZXhhbXBsZS5jb20iLCJleHAiOjE3OTA0OTEwOTMsImlhdCI6MTc5MDQwNDY5M30.P1xtO69NwGFRkdS5TMvmTSjbmoJEkWVdnmSSIEFgOhY","token_type":"Bearer","expires_in":86400,"user":{"id":1,"name":"Nattapong","email":"nattapong@example.com","role":"customer"}}
```

### สั่งซื้อสินค้าและตรวจสอบว่า stock ลดลงจริง

ยิง `/orders` โดยไม่มี token ก่อน เพื่อยืนยันว่า middleware ทำงาน:

```bash
curl -s -o /dev/null -w "%{http_code}\n" -X POST http://localhost:8099/orders \
  -H "Content-Type: application/json" \
  -d '{"items":[{"product_id":1,"quantity":2}]}'
```
```
401
```

สั่งซื้อพร้อม token ที่ถูกต้อง (2 ชิ้นของ Mechanical Keyboard + 1 ชิ้นของ Wireless Mouse):

```bash
TOKEN="eyJhbGci..."   # จาก response ตอน login ด้านบน

curl -s -X POST http://localhost:8099/orders \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $TOKEN" \
  -d '{"items":[{"product_id":1,"quantity":2},{"product_id":2,"quantity":1}]}'
```
```json
{"id":1,"user_id":1,"status":"paid","total_cents":659700,"items":[{"id":1,"order_id":1,"product_id":1,"product_name":"Mechanical Keyboard","unit_price_cents":289900,"quantity":2},{"id":2,"order_id":1,"product_id":2,"product_name":"Wireless Mouse","unit_price_cents":79900,"quantity":1}],"created_at":"2026-09-26T06:38:21.239523671Z"}
```

`total_cents` คำนวณถูกต้อง: `289900*2 + 79900*1 = 659700` ตรวจสอบ stock หลังสั่งซื้อ (เริ่มที่ 15 สั่งไป 2 ต้องเหลือ 13):

```bash
curl -s http://localhost:8099/products/1
```
```json
{"id":1,"name":"Mechanical Keyboard","description":"Hot-swappable 75% keyboard","price_cents":289900,"stock":13,"category_id":1,"category":{"id":1,"name":"Electronics","slug":"electronics"},"created_at":"2026-09-26T06:38:02.564797219Z","updated_at":"2026-09-26T06:38:02.564797219Z"}
```

ดูประวัติคำสั่งซื้อ:

```bash
curl -s http://localhost:8099/orders -H "Authorization: Bearer $TOKEN"
```
```json
{"data":[{"id":1,"user_id":1,"status":"paid","total_cents":659700,"items":[{"id":1,"order_id":1,"product_id":1,"product_name":"Mechanical Keyboard","unit_price_cents":289900,"quantity":2},{"id":2,"order_id":1,"product_id":2,"product_name":"Wireless Mouse","unit_price_cents":79900,"quantity":1}],"created_at":"2026-09-26T06:38:21.239523671Z"}],"meta":{"page":1,"page_size":1,"total":1}}
```

### พิสูจน์ atomicity: สั่งซื้อเกิน stock ต้องถูกปฏิเสธทั้งหมด

```bash
curl -s -w "\nHTTP:%{http_code}\n" -X POST http://localhost:8099/orders \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $TOKEN" \
  -d '{"items":[{"product_id":2,"quantity":999999}]}'
```
```
{"error":{"message":"insufficient stock"}}
HTTP:409
```

### RBAC: ผู้ใช้ทั่วไปสร้างสินค้าไม่ได้

```bash
curl -s -w "\nHTTP:%{http_code}\n" -X POST http://localhost:8099/products \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $TOKEN" \
  -d '{"name":"Test","price_cents":1000,"stock":5}'
```
```
{"error":{"message":"you do not have permission to perform this action"}}
HTTP:403
```

ผลลัพธ์ทุกจุดข้างต้นคือ output จริงจากการรันเซิร์ฟเวอร์และยิง `curl` จริงระหว่างเขียนบทนี้

---

## 14. Integration Test ด้วย `httptest`

เทสจริงที่ครอบคลุมทุก flow หลักของระบบ ใช้เทคนิคจาก **Part 081**: สร้างแอปทั้งก้อน (repository จริงบน SQLite ไฟล์ชั่วคราว + usecase จริง + router จริง) แล้วห่อด้วย `httptest.NewServer` เพื่อทดสอบผ่าน HTTP จริง ไม่ mock อะไรเลยนอกจากใช้ไฟล์ฐานข้อมูลชั่วคราวแทนไฟล์ถาวร

### `internal/handler/handler_test.go`

```go
package handler_test

import (
	"bytes"
	"encoding/json"
	"net/http"
	"net/http/httptest"
	"path/filepath"
	"testing"
	"time"

	"ecommerce-api/internal/domain"
	"ecommerce-api/internal/handler"
	"ecommerce-api/internal/repository/sqlite"
	"ecommerce-api/internal/usecase"
)

// newTestServer สร้างแอปทั้งก้อน (repository จริงบน SQLite ไฟล์ชั่วคราว +
// usecase จริง + router จริง) แล้วห่อด้วย httptest.NewServer เพื่อทดสอบ
// แบบ end-to-end ผ่าน HTTP จริงๆ (แนวคิดจาก Part 081 หัวข้อ 4 และ 6)
// ใช้ไฟล์ชั่วคราวแทน ":memory:" เพราะ driver mattn/go-sqlite3 ผูก
// in-memory database ไว้กับ connection เดียว ถ้าใช้ connection pool ปกติ
// (แม้จะจำกัดไว้ที่ 1 ใน db.go) การเปิดไฟล์จริงด้วย t.TempDir() ให้ผล
// ที่แน่นอนกว่าและใกล้เคียงการใช้งานจริงมากกว่า
func newTestServer(t *testing.T) (*httptest.Server, *usecase.ProductService) {
	t.Helper()

	dbPath := filepath.Join(t.TempDir(), "test.db")
	db, err := sqlite.Open(dbPath)
	if err != nil {
		t.Fatalf("open test db: %v", err)
	}

	categoryRepo := sqlite.NewCategoryRepository(db)
	productRepo := sqlite.NewProductRepository(db)
	userRepo := sqlite.NewUserRepository(db)
	orderRepo := sqlite.NewOrderRepository(db)

	authService := usecase.NewAuthService(userRepo, []byte("test-secret"), time.Hour)
	productService := usecase.NewProductService(productRepo, categoryRepo)
	orderService := usecase.NewOrderService(orderRepo, productRepo)

	router := handler.NewRouter(authService, productService, orderService)
	srv := httptest.NewServer(router)
	t.Cleanup(srv.Close)

	return srv, productService
}

// seedProduct สร้างสินค้าตัวอย่างตรงๆ ผ่าน usecase (ไม่ผ่าน HTTP) เพื่อ
// เตรียมข้อมูลให้เทสอื่นใช้ โดยไม่ต้องพึ่ง endpoint POST /products
// (ซึ่งต้องใช้สิทธิ์ admin — เทสเรื่อง RBAC แยกไว้ต่างหากด้านล่าง)
func seedProduct(t *testing.T, products *usecase.ProductService, stock int, priceCents int64) *domain.Product {
	t.Helper()
	p := &domain.Product{Name: "Test Widget", PriceCents: priceCents, Stock: stock}
	if err := products.Create(t.Context(), p); err != nil {
		t.Fatalf("seed product: %v", err)
	}
	return p
}

func doJSON(t *testing.T, method, url string, token string, body any) *http.Response {
	t.Helper()
	var buf bytes.Buffer
	if body != nil {
		if err := json.NewEncoder(&buf).Encode(body); err != nil {
			t.Fatalf("encode body: %v", err)
		}
	}
	req, err := http.NewRequest(method, url, &buf)
	if err != nil {
		t.Fatalf("new request: %v", err)
	}
	req.Header.Set("Content-Type", "application/json")
	if token != "" {
		req.Header.Set("Authorization", "Bearer "+token)
	}
	resp, err := http.DefaultClient.Do(req)
	if err != nil {
		t.Fatalf("do request: %v", err)
	}
	return resp
}

func decodeBody(t *testing.T, resp *http.Response, dst any) {
	t.Helper()
	defer resp.Body.Close()
	if err := json.NewDecoder(resp.Body).Decode(dst); err != nil {
		t.Fatalf("decode response body: %v", err)
	}
}

func TestHealthz(t *testing.T) {
	srv, _ := newTestServer(t)
	resp, err := http.Get(srv.URL + "/healthz")
	if err != nil {
		t.Fatalf("GET /healthz: %v", err)
	}
	defer resp.Body.Close()
	if resp.StatusCode != http.StatusOK {
		t.Fatalf("expected 200, got %d", resp.StatusCode)
	}
}

func TestRegisterAndLogin(t *testing.T) {
	srv, _ := newTestServer(t)

	registerBody := map[string]string{
		"name":     "Somsri",
		"email":    "somsri@example.com",
		"password": "s3cretpass",
	}
	resp := doJSON(t, http.MethodPost, srv.URL+"/auth/register", "", registerBody)
	if resp.StatusCode != http.StatusCreated {
		t.Fatalf("register: expected 201, got %d", resp.StatusCode)
	}
	var user map[string]any
	decodeBody(t, resp, &user)
	if user["email"] != "somsri@example.com" {
		t.Fatalf("unexpected user in response: %+v", user)
	}
	if _, leaked := user["password"]; leaked {
		t.Fatalf("password must never be present in the response: %+v", user)
	}

	// สมัครซ้ำด้วยอีเมลเดิมต้องได้ 409 Conflict (domain.ErrEmailTaken)
	resp2 := doJSON(t, http.MethodPost, srv.URL+"/auth/register", "", registerBody)
	if resp2.StatusCode != http.StatusConflict {
		t.Fatalf("duplicate register: expected 409, got %d", resp2.StatusCode)
	}
	resp2.Body.Close()

	// login ด้วยรหัสผ่านผิดต้องได้ 401
	wrongLogin := map[string]string{"email": "somsri@example.com", "password": "wrong-password"}
	resp3 := doJSON(t, http.MethodPost, srv.URL+"/auth/login", "", wrongLogin)
	if resp3.StatusCode != http.StatusUnauthorized {
		t.Fatalf("wrong password: expected 401, got %d", resp3.StatusCode)
	}
	resp3.Body.Close()

	// login ถูกต้องต้องได้ access_token กลับมา
	correctLogin := map[string]string{"email": "somsri@example.com", "password": "s3cretpass"}
	resp4 := doJSON(t, http.MethodPost, srv.URL+"/auth/login", "", correctLogin)
	if resp4.StatusCode != http.StatusOK {
		t.Fatalf("login: expected 200, got %d", resp4.StatusCode)
	}
	var loginResp map[string]any
	decodeBody(t, resp4, &loginResp)
	if loginResp["access_token"] == "" || loginResp["access_token"] == nil {
		t.Fatalf("expected access_token in login response, got %+v", loginResp)
	}
}

func TestRegisterValidation(t *testing.T) {
	srv, _ := newTestServer(t)

	// email ผิดรูปแบบ + password สั้นเกินไป ต้องได้ 422 พร้อม field errors
	resp := doJSON(t, http.MethodPost, srv.URL+"/auth/register", "", map[string]string{
		"name":     "Bad Input",
		"email":    "not-an-email",
		"password": "123",
	})
	if resp.StatusCode != http.StatusUnprocessableEntity {
		t.Fatalf("expected 422, got %d", resp.StatusCode)
	}
	var errResp handler.ErrorResponse
	decodeBody(t, resp, &errResp)
	if _, ok := errResp.Error.Fields["email"]; !ok {
		t.Errorf("expected field error for email, got %+v", errResp.Error.Fields)
	}
	if _, ok := errResp.Error.Fields["password"]; !ok {
		t.Errorf("expected field error for password, got %+v", errResp.Error.Fields)
	}
}

func TestListAndGetProducts(t *testing.T) {
	srv, products := newTestServer(t)
	p := seedProduct(t, products, 10, 5000)

	resp := doJSON(t, http.MethodGet, srv.URL+"/products", "", nil)
	if resp.StatusCode != http.StatusOK {
		t.Fatalf("list products: expected 200, got %d", resp.StatusCode)
	}
	var listResp handler.ListResponse
	decodeBody(t, resp, &listResp)
	if listResp.Meta.Total != 1 {
		t.Fatalf("expected total=1, got %d", listResp.Meta.Total)
	}

	resp2 := doJSON(t, http.MethodGet, srv.URL+"/products/1", "", nil)
	if resp2.StatusCode != http.StatusOK {
		t.Fatalf("get product: expected 200, got %d", resp2.StatusCode)
	}
	var got domain.Product
	decodeBody(t, resp2, &got)
	if got.ID != p.ID || got.Name != p.Name {
		t.Fatalf("unexpected product: %+v", got)
	}

	// สินค้าที่ไม่มีอยู่จริงต้องได้ 404
	resp3 := doJSON(t, http.MethodGet, srv.URL+"/products/999", "", nil)
	if resp3.StatusCode != http.StatusNotFound {
		t.Fatalf("get missing product: expected 404, got %d", resp3.StatusCode)
	}
	resp3.Body.Close()
}

// registerAndLoginHelper สมัครสมาชิกแล้ว login ทันที คืน access token
// สำหรับใช้เป็น Authorization header ในเทสที่ต้องยืนยันตัวตน
func registerAndLoginHelper(t *testing.T, srv *httptest.Server, email string) string {
	t.Helper()
	body := map[string]string{"name": "Test User", "email": email, "password": "s3cretpass"}
	resp := doJSON(t, http.MethodPost, srv.URL+"/auth/register", "", body)
	if resp.StatusCode != http.StatusCreated {
		t.Fatalf("register: expected 201, got %d", resp.StatusCode)
	}
	resp.Body.Close()

	loginResp := doJSON(t, http.MethodPost, srv.URL+"/auth/login", "", map[string]string{
		"email": email, "password": "s3cretpass",
	})
	var login map[string]any
	decodeBody(t, loginResp, &login)
	token, _ := login["access_token"].(string)
	if token == "" {
		t.Fatalf("expected access token, got %+v", login)
	}
	return token
}

func TestPlaceOrderRequiresAuth(t *testing.T) {
	srv, products := newTestServer(t)
	seedProduct(t, products, 10, 1000)

	resp := doJSON(t, http.MethodPost, srv.URL+"/orders", "", map[string]any{
		"items": []map[string]any{{"product_id": 1, "quantity": 1}},
	})
	if resp.StatusCode != http.StatusUnauthorized {
		t.Fatalf("expected 401 without token, got %d", resp.StatusCode)
	}
	resp.Body.Close()
}

func TestPlaceOrderDecrementsStockAtomically(t *testing.T) {
	srv, products := newTestServer(t)
	p := seedProduct(t, products, 5, 10000) // stock = 5, price = 100.00

	token := registerAndLoginHelper(t, srv, "buyer@example.com")

	// สั่งซื้อ 3 ชิ้น ต้องสำเร็จและ stock ต้องลดลงเหลือ 2
	resp := doJSON(t, http.MethodPost, srv.URL+"/orders", token, map[string]any{
		"items": []map[string]any{{"product_id": p.ID, "quantity": 3}},
	})
	if resp.StatusCode != http.StatusCreated {
		t.Fatalf("place order: expected 201, got %d", resp.StatusCode)
	}
	var order domain.Order
	decodeBody(t, resp, &order)
	if order.TotalCents != 30000 {
		t.Errorf("expected total_cents=30000, got %d", order.TotalCents)
	}
	if len(order.Items) != 1 || order.Items[0].Quantity != 3 {
		t.Fatalf("unexpected order items: %+v", order.Items)
	}

	got, err := products.Get(t.Context(), p.ID)
	if err != nil {
		t.Fatalf("get product after order: %v", err)
	}
	if got.Stock != 2 {
		t.Fatalf("expected stock=2 after ordering 3 of 5, got %d", got.Stock)
	}

	// สั่งซื้อเกิน stock ที่เหลือ (ขอ 10 ทั้งที่เหลือ 2) ต้องถูกปฏิเสธด้วย
	// 409 Conflict และ stock ต้อง "ไม่" เปลี่ยนแปลงเลย (all-or-nothing —
	// พิสูจน์ atomicity ของ transaction ตาม Part 078)
	resp2 := doJSON(t, http.MethodPost, srv.URL+"/orders", token, map[string]any{
		"items": []map[string]any{{"product_id": p.ID, "quantity": 10}},
	})
	if resp2.StatusCode != http.StatusConflict {
		t.Fatalf("overselling order: expected 409, got %d", resp2.StatusCode)
	}
	resp2.Body.Close()

	stillGot, err := products.Get(t.Context(), p.ID)
	if err != nil {
		t.Fatalf("get product after failed order: %v", err)
	}
	if stillGot.Stock != 2 {
		t.Fatalf("stock must be unchanged after a failed order, expected 2, got %d", stillGot.Stock)
	}
}

func TestOrderHistoryOnlyShowsOwnOrders(t *testing.T) {
	srv, products := newTestServer(t)
	p := seedProduct(t, products, 10, 5000)

	aliceToken := registerAndLoginHelper(t, srv, "alice@example.com")
	bobToken := registerAndLoginHelper(t, srv, "bob@example.com")

	resp := doJSON(t, http.MethodPost, srv.URL+"/orders", aliceToken, map[string]any{
		"items": []map[string]any{{"product_id": p.ID, "quantity": 1}},
	})
	if resp.StatusCode != http.StatusCreated {
		t.Fatalf("alice place order: expected 201, got %d", resp.StatusCode)
	}
	resp.Body.Close()

	aliceHistory := doJSON(t, http.MethodGet, srv.URL+"/orders", aliceToken, nil)
	var aliceList handler.ListResponse
	decodeBody(t, aliceHistory, &aliceList)
	if aliceList.Meta.Total != 1 {
		t.Fatalf("alice: expected 1 order, got %d", aliceList.Meta.Total)
	}

	bobHistory := doJSON(t, http.MethodGet, srv.URL+"/orders", bobToken, nil)
	var bobList handler.ListResponse
	decodeBody(t, bobHistory, &bobList)
	if bobList.Meta.Total != 0 {
		t.Fatalf("bob: expected 0 orders, got %d", bobList.Meta.Total)
	}
}

func TestCreateProductRequiresAdminRole(t *testing.T) {
	srv, _ := newTestServer(t)
	token := registerAndLoginHelper(t, srv, "customer@example.com")

	resp := doJSON(t, http.MethodPost, srv.URL+"/products", token, map[string]any{
		"name":        "Unauthorized Product",
		"price_cents": 1000,
		"stock":       5,
	})
	if resp.StatusCode != http.StatusForbidden {
		t.Fatalf("expected 403 for non-admin creating product, got %d", resp.StatusCode)
	}
	resp.Body.Close()
}

func TestInvalidTokenRejected(t *testing.T) {
	srv, products := newTestServer(t)
	seedProduct(t, products, 10, 1000)

	resp := doJSON(t, http.MethodGet, srv.URL+"/orders", "this-is-not-a-valid-jwt", nil)
	if resp.StatusCode != http.StatusUnauthorized {
		t.Fatalf("expected 401 for malformed token, got %d", resp.StatusCode)
	}
	resp.Body.Close()
}
```

### รันเทสจริง

```bash
go test ./... -v
```

ผลลัพธ์จริง (ทุกเทสผ่านหมด):

```
?   	ecommerce-api/cmd/api	[no test files]
?   	ecommerce-api/internal/domain	[no test files]
=== RUN   TestHealthz
--- PASS: TestHealthz (0.06s)
=== RUN   TestRegisterAndLogin
--- PASS: TestRegisterAndLogin (0.23s)
=== RUN   TestRegisterValidation
--- PASS: TestRegisterValidation (0.01s)
=== RUN   TestListAndGetProducts
--- PASS: TestListAndGetProducts (0.01s)
=== RUN   TestPlaceOrderRequiresAuth
--- PASS: TestPlaceOrderRequiresAuth (0.01s)
=== RUN   TestPlaceOrderDecrementsStockAtomically
--- PASS: TestPlaceOrderDecrementsStockAtomically (0.16s)
=== RUN   TestOrderHistoryOnlyShowsOwnOrders
--- PASS: TestOrderHistoryOnlyShowsOwnOrders (0.32s)
=== RUN   TestCreateProductRequiresAdminRole
--- PASS: TestCreateProductRequiresAdminRole (0.16s)
=== RUN   TestInvalidTokenRejected
--- PASS: TestInvalidTokenRejected (0.02s)
PASS
ok  	ecommerce-api/internal/handler	0.996s
?   	ecommerce-api/internal/repository/sqlite	[no test files]
?   	ecommerce-api/internal/usecase	[no test files]
```

ทดสอบเพิ่มเติมด้วย **race detector** (Part 044) เพื่อยืนยันว่าไม่มี data race ในการเข้าถึง `*gorm.DB` แบบพร้อมกันจากหลาย goroutine ของ `httptest`:

```bash
go test ./... -race
```
```
?   	ecommerce-api/cmd/api	[no test files]
?   	ecommerce-api/internal/domain	[no test files]
ok  	ecommerce-api/internal/handler	10.904s
?   	ecommerce-api/internal/repository/sqlite	[no test files]
?   	ecommerce-api/internal/usecase	[no test files]
```

เทสที่สำคัญที่สุดในไฟล์นี้คือ `TestPlaceOrderDecrementsStockAtomically` — มันไม่ได้แค่เช็คว่า status code ถูกต้อง แต่**เช็คสถานะจริงของฐานข้อมูลหลังจากนั้น** ทั้งกรณีสำเร็จ (stock ลดลงตามจำนวนที่สั่งจริง) และกรณีล้มเหลว (stock ไม่เปลี่ยนแปลงแม้แต่หน่วยเดียว) — นี่คือมาตรฐานของ integration test ที่ดีตาม **Part 080**: ทดสอบ "ผลลัพธ์ที่สังเกตได้จริง" ไม่ใช่แค่ "response ที่ handler ส่งกลับมา"

---

## 15. สิ่งที่ตั้งใจไม่ทำในบทนี้ และจะไปต่อที่ไหน

โปรเจกต์นี้ตั้งใจโฟกัสที่การประกอบสถาปัตยกรรมและ transaction ที่ถูกต้อง ไม่ใช่การสร้างระบบ e-commerce ที่ครบทุกฟีเจอร์ สิ่งต่อไปนี้ถูก**ตัดออกโดยตั้งใจ** เพื่อรักษาขอบเขตบทเรียนให้จัดการได้:

| ตัดออก | เหตุผล | ไปเรียนต่อที่ |
|---|---|---|
| การชำระเงินจริง (payment gateway) | ต้องพึ่ง third-party API ภายนอก (Stripe, Omise) ซึ่งไม่ใช่หัวข้อของหลักสูตร Go | ปกติทำผ่าน webhook + HTTP client (Part 046) |
| การจอง stock ชั่วคราว (inventory reservation / cart hold) | ระบบจริงมักจอง stock ไว้ชั่วคราวตอนใส่ตะกร้า ก่อนจะยืนยันตัดจริงตอน checkout ซึ่งซับซ้อนกว่าการตัดทันทีแบบบทนี้มาก | ต่อยอดด้วย Redis (Part 077) เก็บ reservation ที่มี TTL |
| Refresh token / token revocation | บทนี้ใช้ access token อายุยาว (24 ชั่วโมง) เพื่อความง่าย ระบบจริงควรใช้ access token อายุสั้น + refresh token | Part 067 หัวข้อ 9 อธิบาย pattern นี้ไว้แล้ว |
| Admin panel (หน้าเว็บจัดการหลังบ้าน) | บทนี้มีแค่ REST endpoint สำหรับ admin ยังไม่มี UI | Part 065 (`html/template`) หรือ frontend framework แยกต่างหาก |
| Rate limiting / API key สำหรับ public API | ป้องกัน abuse เป็นเรื่องสำคัญของ production แต่ไม่ใช่แก่นของบทนี้ | Part 094 (Circuit Breaker/Retry) และ middleware เพิ่มเติมแบบ Part 057 |
| Search แบบ full-text/fuzzy | ตอนนี้ใช้ `LIKE '%...%'` แบบง่ายที่สุด ซึ่งช้าเมื่อข้อมูลเยอะ | ต่อยอดด้วย PostgreSQL full-text search (Part 072) หรือ Elasticsearch |
| Caching สินค้าที่ถูกดูบ่อย | ทุก request ยิงเข้าฐานข้อมูลตรงๆ | Part 077 (Redis) เพิ่ม cache-aside pattern ได้ |
| Observability (metrics, tracing) | ยังไม่มี logging โครงสร้าง, metrics, tracing เชื่อมกับระบบ monitoring | Part 099 (Prometheus, Grafana, OpenTelemetry) |
| Deploy เป็น container | โปรเจกต์นี้รันด้วย `go run`/`go build` ตรงๆ | Part 095-096 (Docker, Docker Compose) |

การรู้ว่า "ตัดอะไรออกด้วยเหตุผลอะไร" สำคัญพอๆ กับการรู้ว่าเขียนอะไรเข้าไป — เป็นทักษะของวิศวกรมืออาชีพที่ต้องประเมิน scope ของงานให้เหมาะกับเวลาและเป้าหมายที่มีอยู่เสมอ

---

## 16. วิธีรันโปรเจกต์นี้ด้วยตัวเอง

```bash
# 1. สร้างโปรเจกต์และติดตั้ง dependency
mkdir ecommerce-api && cd ecommerce-api
go mod init ecommerce-api
go get github.com/go-chi/chi/v5
go get gorm.io/gorm
go get gorm.io/driver/sqlite
go get github.com/golang-jwt/jwt/v5
go get golang.org/x/crypto@v0.31.0
go get github.com/go-playground/validator/v10@v10.22.1

# 2. สร้างโฟลเดอร์ตามโครงสร้างในหัวข้อ 2 แล้วคัดลอกโค้ดจากบทนี้ลงแต่ละไฟล์

# 3. ตรวจสอบว่าคอมไพล์ผ่านและไม่มี logic error ที่ go vet จับได้
go build ./...
go vet ./...

# 4. รัน integration test ทั้งหมด (ควรผ่านทุกเทส)
go test ./... -v
go test ./... -race

# 5. รันเซิร์ฟเวอร์จริง (ค่า default: PORT=8080, DB_PATH=ecommerce.db)
go run ./cmd/api

# หรือกำหนดค่าเองผ่าน environment variable
PORT=9000 DB_PATH=mystore.db JWT_SECRET=change-me-please go run ./cmd/api

# 6. build เป็น binary จริงสำหรับ deploy
go build -o bin/ecommerce-api ./cmd/api
./bin/ecommerce-api
```

ตัวแปรแวดล้อมที่ปรับได้:

| ตัวแปร | ค่า default | ความหมาย |
|---|---|---|
| `PORT` | `8080` | พอร์ตที่เซิร์ฟเวอร์ฟัง |
| `DB_PATH` | `ecommerce.db` | ไฟล์ฐานข้อมูล SQLite |
| `JWT_SECRET` | `dev-secret-change-me-in-production` | secret key สำหรับเซ็น JWT — **ต้องเปลี่ยนก่อนใช้งานจริงเสมอ** ตามข้อควรระวังใน Part 067 หัวข้อ 10 |

เมื่อรันครั้งแรก ระบบจะ seed หมวดหมู่และสินค้าตัวอย่างให้อัตโนมัติ (ดูฟังก์ชัน `seedDemoData` ในหัวข้อ 12) ทำให้ลองยิง `curl` ตามหัวข้อ 13 ได้ทันทีโดยไม่ต้องสร้างข้อมูลเอง หากต้องการล้างข้อมูลแล้วเริ่มใหม่ ให้ลบไฟล์ `ecommerce.db` (หรือไฟล์ตาม `DB_PATH` ที่ตั้งไว้) แล้วรันใหม่

---

## สรุปสิ่งที่ได้เรียนในบทนี้

- โปรเจกต์นี้คือการนำ **Clean Architecture (Part 100)** มาใช้จริงกับ 4 ชั้น: `domain` (entity + repository interface), `usecase` (business logic), `repository/sqlite` (GORM adapter), `handler` (HTTP delivery ด้วย chi) โดยทิศทางการ import ไหลเข้าหา `domain` เสมอ ไม่มีทางย้อนกลับ
- การแยก GORM model (`productModel` ฯลฯ) ออกจาก domain entity (`domain.Product`) ทำให้ business logic ไม่ผูกติดกับรายละเอียดของฐานข้อมูล และสลับจาก SQLite ไป PostgreSQL (Part 072) ได้โดยแก้แค่ฟังก์ชัน `Open` เพียงจุดเดียว
- **`OrderRepository.PlaceOrder`** คือหัวใจของบทนี้ในแง่ transaction (Part 078): ใช้ `db.Transaction` ร่วมกับ `UPDATE ... WHERE stock >= ?` เพื่อลด stock แบบ atomic ระดับแถว ป้องกัน race condition ที่ทำให้ขายเกิน stock ได้ และ rollback ทั้งหมดถ้ามีรายการใดรายการหนึ่งล้มเหลว
- ระบบ authentication ใช้ **bcrypt (Part 051)** hash รหัสผ่านและ **JWT (Part 067)** ออก access token โดยแยก `AuthMiddleware` (ยืนยันตัวตน) กับ `RequireRole` (ตรวจสิทธิ์) เป็นคนละ middleware เพื่อผสมใช้ตามความต้องการของแต่ละ endpoint
- `user_id` ของคำสั่งซื้อ**อ่านจาก JWT claims เสมอ ไม่เคยอ่านจาก request body** เพื่อป้องกันช่องโหว่ Broken Object Level Authorization (BOLA)
- ทุก response ใช้ JSON envelope ที่สม่ำเสมอ (`ErrorResponse`, `ListResponse` พร้อม `Meta` สำหรับ pagination) ตามหลักการจาก **Part 059-060** และ error จากชั้น domain ถูกแม็ปเป็น HTTP status code ที่จุดเดียว (`respondDomainError`) ไม่กระจัดกระจาย
- Router ใช้ **chi (Part 058)** เพราะจัดการ nested middleware (auth + role) ตามกลุ่ม route ได้สะอาดกว่า stdlib `ServeMux` เปล่าๆ
- Integration test ด้วย **`httptest.NewServer` (Part 081)** ทดสอบทั้งแอปจริงผ่าน HTTP รวมถึงพิสูจน์ atomicity ของ transaction (สั่งซื้อเกิน stock ต้องไม่กระทบ stock เดิมแม้แต่หน่วยเดียว) — ทุกเทสผ่านทั้งแบบปกติและแบบ `-race`
- การกำหนดขอบเขตให้ชัดเจน (สิ่งที่ตัดออก + เหตุผล + จะไปเรียนต่อที่ไหน) เป็นทักษะสำคัญของวิศวกรมืออาชีพพอๆ กับการเขียนโค้ด

## แบบฝึกหัดท้ายบท

1. เพิ่ม query parameter `sort` ให้ `GET /products` รองรับการเรียงลำดับตามราคา (`sort=price_asc`, `sort=price_desc`) และตามชื่อ (`sort=name`) โดยยึดหลักการ filtering/sorting จาก Part 059 หัวข้อ 7
2. เพิ่ม endpoint `GET /orders/{id}/cancel` ที่อนุญาตให้เจ้าของคำสั่งซื้อยกเลิกคำสั่งซื้อที่ยังไม่ได้จัดส่ง โดยต้อง**คืน stock กลับเข้าไปในสินค้าแต่ละชิ้นแบบ atomic** เช่นเดียวกับตอนสั่งซื้อ (เขียน transaction ใหม่ใน `order_repo.go`)
3. สลับฐานข้อมูลจาก SQLite เป็น PostgreSQL ตามที่เรียนใน **Part 072**: เพิ่ม `gorm.io/driver/postgres`, แก้ฟังก์ชัน `Open` ใน `internal/repository/sqlite/db.go` (หรือคัดลอกเป็นแพ็กเกจ `postgres` ใหม่) แล้วรัน integration test เดิมซ้ำเพื่อยืนยันว่าโค้ดชั้น `usecase`/`handler` ไม่ต้องแก้อะไรเลย
4. เพิ่ม refresh token pattern ตามแนวทางใน **Part 067 หัวข้อ 9**: endpoint `POST /auth/refresh` ที่ออก access token อายุสั้น (15 นาที) ใหม่จาก refresh token อายุยาว โดยเก็บ refresh token ที่ยังไม่ถูกเพิกถอนไว้ในตารางใหม่
5. เพิ่ม field `Category` filter หลายค่าพร้อมกัน (เช่น `?category_id=1,2,3`) ให้ `ProductFilter` และแก้ `ProductRepository.List` ให้ใช้ `WHERE category_id IN (...)` แทน `=`
6. เขียน benchmark (Part 034/084) เปรียบเทียบความเร็วของ `PlaceOrder` ระหว่างคำสั่งซื้อที่มี 1 รายการ กับคำสั่งซื้อที่มี 50 รายการ แล้ววิเคราะห์ว่า bottleneck อยู่ตรงไหน (คำใบ้: ลองนับจำนวนคำสั่ง SQL ที่เกิดขึ้นต่อหนึ่งรายการสินค้าในลูปของ `PlaceOrder`)

---

**ต่อไป**: [Part 104 — โปรเจกต์ Real-time Chat Application](./104-project-realtime-chat.md)
