# Part 100: Clean Architecture ใน Go

> ภาคที่ 10: มืออาชีพและระดับโลก (Professional & World-Class) — ตอนที่ 1 จาก 11 (Part 100–110)

ยินดีต้อนรับสู่ **ภาคที่ 10** — ภาคสุดท้ายของหลักสูตรนี้ ตลอด 99 part ที่ผ่านมา เราเรียนรู้ทุกกลไกของภาษา Go ตั้งแต่ syntax พื้นฐาน, concurrency, standard library, web development, database, testing/tooling, ไปจนถึง microservices และ DevOps ภาคนี้จะไม่สอนกลไกใหม่ของภาษาอีกต่อไป แต่จะสอน **วิธีจัดระเบียบทุกสิ่งที่เรียนมาแล้ว** ให้กลายเป็นระบบที่ดูแลรักษาได้ในระยะยาว — ทักษะที่แยกวิศวกรมืออาชีพออกจากคนที่แค่ "เขียนโค้ดให้รันได้"

บทแรกของภาคนี้ว่าด้วย **Clean Architecture** — แนวคิดการจัดโครงสร้างโค้ดที่ Robert C. Martin ("Uncle Bob") เผยแพร่ในปี 2012 ซึ่งกลายเป็นมาตรฐานอุตสาหกรรมสำหรับระบบขนาดกลาง-ใหญ่ในหลายภาษา และเข้ากันได้ดีเป็นพิเศษกับปรัชญาการออกแบบของ Go

## สารบัญของบทนี้

1. ปัญหาที่ Clean Architecture แก้ไข: โค้ดที่ผูกติดกับ Framework/Database
2. Dependency Rule: กฎเหล็กข้อเดียวที่ต้องจำ
3. ชั้นทั้งสี่แบบ Go: Domain, Usecase, Repository (Interface), Delivery
4. ทำไม Implicit Interface Satisfaction ของ Go ถึงทำให้ Pattern นี้เป็นธรรมชาติ
5. โครงสร้างโปรเจกต์: จัดวางไฟล์อย่างไรให้สื่อความหมาย
6. ตัวอย่างเต็ม: Product Service แบบ Clean Architecture (รันได้จริง)
7. ทดสอบ Business Logic โดยไม่ต้องมีฐานข้อมูลจริง
8. สลับ Repository จาก In-Memory เป็น PostgreSQL โดยไม่แตะ Business Logic
9. ข้อควรระวังตรงไปตรงมา: Clean Architecture ไม่ใช่ยาวิเศษ
10. เมื่อไรควรใช้ และเมื่อไรไม่ควรใช้
11. สรุปสิ่งที่ได้เรียนในบทนี้
12. แบบฝึกหัดท้ายบท

---

## 1. ปัญหาที่ Clean Architecture แก้ไข: โค้ดที่ผูกติดกับ Framework/Database

ลองจินตนาการโค้ดแบบที่มือใหม่ (และแม้แต่มือกลางจำนวนไม่น้อย) มักเขียนกัน — handler ของ HTTP endpoint เดียวทำทุกอย่างพร้อมกัน:

```go
func handleCreateOrder(w http.ResponseWriter, r *http.Request) {
	var req struct {
		CustomerID string  `json:"customer_id"`
		ProductID  string  `json:"product_id"`
		Quantity   int     `json:"quantity"`
	}
	json.NewDecoder(r.Body).Decode(&req)

	// business logic ปนกับ HTTP handling ปนกับ SQL ทั้งหมดในฟังก์ชันเดียว
	if req.Quantity <= 0 {
		http.Error(w, "invalid quantity", 400)
		return
	}
	row := db.QueryRow("SELECT price, stock FROM products WHERE id = $1", req.ProductID)
	var price float64
	var stock int
	row.Scan(&price, &stock)
	if stock < req.Quantity {
		http.Error(w, "insufficient stock", 422)
		return
	}
	db.Exec("UPDATE products SET stock = stock - $1 WHERE id = $2", req.Quantity, req.ProductID)
	db.Exec("INSERT INTO orders (customer_id, product_id, quantity, total) VALUES ($1, $2, $3, $4)",
		req.CustomerID, req.ProductID, req.Quantity, price*float64(req.Quantity))

	w.WriteHeader(201)
}
```

โค้ดแบบนี้รันได้ และสำหรับ prototype เล็กๆ ก็อาจไม่มีปัญหาอะไรเลย แต่เมื่อระบบโตขึ้น ปัญหาจะค่อยๆ ปรากฏ:

- **ทดสอบ business logic ไม่ได้โดยไม่มีฐานข้อมูลจริง** — อยากทดสอบว่า "ถ้าสต็อกไม่พอต้อง reject" ก็ต้องมี PostgreSQL รันอยู่จริงเสมอ ทำให้ test ช้าและเปราะบาง (ทบทวนความแตกต่างระหว่าง unit test กับ integration test จาก **Part 079–080**)
- **เปลี่ยนจาก REST เป็น gRPC (Part 089) ต้องเขียน business logic ใหม่ทั้งหมด** เพราะมันฝังอยู่ใน HTTP handler โดยตรง
- **เปลี่ยนจาก PostgreSQL เป็น MongoDB (Part 076) ต้องไล่หา SQL string ที่กระจัดกระจายอยู่ทั่วทั้ง handler** ไม่มีจุดศูนย์กลางที่รู้จัก "การเข้าถึงข้อมูล" ทั้งหมด
- **กฎทางธุรกิจซ้ำซ้อนกันในหลาย handler** — ถ้ามีอีก endpoint ที่ต้องเช็ค "สต็อกพอไหม" ก็ต้อง copy logic เดิมไปวางซ้ำ

**Clean Architecture** แก้ปัญหานี้ด้วยการ**แยกความรับผิดชอบออกเป็นชั้นๆ (layers)** อย่างมีวินัย โดยมีกฎเหล็กข้อเดียวที่ควบคุมทุกอย่าง — กฎที่จะอธิบายในหัวข้อถัดไป

---

## 2. Dependency Rule: กฎเหล็กข้อเดียวที่ต้องจำ

หัวใจทั้งหมดของ Clean Architecture สรุปได้เป็นประโยคเดียว:

> **"Source code dependencies must point only inward, toward higher-level policies."**
> ทิศทางของ "การรู้จัก" (import) ต้องชี้เข้าหาศูนย์กลางเสมอ ไม่มีทางชี้ออกจากศูนย์กลาง

ลองนึกภาพวงกลมซ้อนกันหลายชั้น:

```
┌─────────────────────────────────────────────────┐
│  Delivery (HTTP handler, gRPC server, CLI)         │
│  ┌─────────────────────────────────────────────┐  │
│  │  Infrastructure (Postgres repo, Redis client)  │  │
│  │  ┌───────────────────────────────────────┐  │  │
│  │  │  Usecase (business logic / service)      │  │  │
│  │  │  ┌─────────────────────────────────┐  │  │  │
│  │  │  │  Domain (entity, business rule)    │  │  │  │
│  │  │  └─────────────────────────────────┘  │  │  │
│  │  └───────────────────────────────────────┘  │  │
│  └─────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────┘
        ลูกศร import ทั้งหมดชี้เข้าหาศูนย์กลางเท่านั้น
```

**ชั้นในสุด (Domain)** ไม่รู้จักอะไรเลยนอกจากตัวมันเอง — ไม่ import `net/http`, ไม่ import `database/sql`, ไม่รู้จัก framework ตัวไหนทั้งสิ้น มันคือ **กฎทางธุรกิจที่แท้จริง** ที่จะยังคงถูกต้องไม่ว่าจะรันบน web, mobile app, CLI, หรือแม้แต่กระดาษ

**ชั้นถัดออกมา (Usecase)** รู้จัก Domain แต่ไม่รู้จัก Infrastructure หรือ Delivery — มันประสานงาน entity ต่างๆ เพื่อทำ business flow แต่ไม่สนใจว่าข้อมูลจะถูกเก็บที่ไหนหรือถูกเรียกผ่านช่องทางอะไร

**ชั้นนอก (Infrastructure, Delivery)** รู้จักทุกอย่างที่อยู่ข้างใน แต่ชั้นในไม่มีวันรู้จักชั้นนอกกลับ — นี่คือทิศทางเดียวที่ยอมให้เกิดขึ้น

ผลที่ตามมาที่สำคัญที่สุด: **ธุรกิจ (business logic) ไม่ควรขึ้นอยู่กับรายละเอียดทางเทคนิค (framework, database, UI) เลย** — ควรเป็นตรงกันข้าม รายละเอียดทางเทคนิคต่างหากที่ควรขึ้นอยู่กับธุรกิจ Uncle Bob เรียกหลักการนี้ว่า **"business logic should not know or care about the delivery mechanism"** — ระบบเดียวกันควรเสิร์ฟผ่าน REST, gRPC, หรือ CLI ก็ได้โดยไม่ต้องเขียน business logic ใหม่แม้แต่บรรทัดเดียว

---

## 3. ชั้นทั้งสี่แบบ Go: Domain, Usecase, Repository (Interface), Delivery

ในการนำ Clean Architecture มาใช้จริงกับ Go ชุมชนมักย่อรูปแบบ 4 ชั้นดั้งเดิมของ Uncle Bob (Entities, Use Cases, Interface Adapters, Frameworks & Drivers) ให้เป็นโครงสร้างที่ใช้งานได้จริงกว่า:

| ชั้น | ชื่อ package ที่นิยม | หน้าที่ | ตัวอย่าง |
|---|---|---|---|
| **Domain** | `domain`, `entity` | entity, value object, กฎทางธุรกิจแท้ๆ, **นิยาม interface ของ repository** | `type Product struct{...}`, `type ProductRepository interface{...}` |
| **Usecase** | `usecase`, `service` | orchestrate business flow, เรียก repository ผ่าน interface | `type ProductService struct{ repo ProductRepository }` |
| **Repository (impl)** | `repository/postgres`, `repository/memory` | **implement** interface ที่ domain กำหนด ด้วยเทคโนโลยีจริง | `type ProductRepository struct{ db *sql.DB }` |
| **Delivery** | `delivery/http`, `delivery/grpc` | แปลง request จากช่องทางภายนอกเป็นการเรียก usecase | `func (h *Handler) CreateProduct(w, r)` |

จุดที่มักทำให้สับสน: **repository interface อยู่ที่ไหน?** สองแนวทางที่พบในโลกจริง ทั้งคู่ถูกต้องทั้งคู่:

1. **นิยามที่ domain** (แนวทางที่บทนี้ใช้สาธิต): เพราะ entity กับกฎว่า "ต้องเก็บ/ดึงอย่างไร" มักผูกกันแน่นในทางความหมาย
2. **นิยามที่ usecase** (แนวทางที่ยึดหลัก "interface ควรอยู่ฝั่งผู้ใช้งาน" จาก **Part 013** อย่างเคร่งครัดที่สุด): เพราะ usecase คือฝ่ายที่ "ใช้" repository จริงๆ

ทั้งสองแนวทางรักษา dependency rule ไว้เหมือนกันทุกประการ — สิ่งที่ **ห้ามเกิดขึ้นเด็ดขาด** คือนิยาม interface นี้ไว้ที่ฝั่ง `repository/postgres` (ชั้นนอก) เพราะจะทำให้ domain/usecase ต้อง import ชั้นนอกกลับเข้ามา ซึ่งขัดกับ dependency rule ทันที

---

## 4. ทำไม Implicit Interface Satisfaction ของ Go ถึงทำให้ Pattern นี้เป็นธรรมชาติ

นี่คือจุดที่ Go มีข้อได้เปรียบชัดเจนเหนือภาษาอย่าง Java หรือ C# ในการทำ Clean Architecture ทบทวนจาก **Part 013**: Go ไม่มี keyword `implements` — type ใดก็ตามที่มี method ครบตามที่ interface กำหนด จะ implement interface นั้นโดยอัตโนมัติ โดยไม่ต้องประกาศความสัมพันธ์ล่วงหน้า

ผลที่ตามมาสำหรับ Clean Architecture คือมหาศาล:

```go
// package domain — ประกาศ interface นี้ได้เลย โดยที่ยังไม่มี concrete
// implementation ตัวไหนอยู่ในโปรเจกต์แม้แต่ตัวเดียว
type ProductRepository interface {
	Create(ctx context.Context, p *Product) error
	FindByID(ctx context.Context, id string) (*Product, error)
}
```

```go
// package repository/postgres — เขียนทีหลัง import "productapp/internal/domain"
// เพียงอย่างเดียว ไม่ต้องรู้จัก usecase เลยด้วยซ้ำ
type ProductRepository struct{ db *sql.DB }

func (r *ProductRepository) Create(ctx context.Context, p *domain.Product) error { /* ... */ }
func (r *ProductRepository) FindByID(ctx context.Context, id string) (*domain.Product, error) { /* ... */ }
// ไม่มีบรรทัดไหนเขียนว่า "implements domain.ProductRepository" เลย
// แต่มันก็เป็น domain.ProductRepository ตัวหนึ่งแล้วโดยอัตโนมัติ
```

ในภาษาที่ต้องประกาศ `implements` ชัดเจน (Java, C#, TypeScript) การจะทำแบบนี้ต้องมี "DI framework" (Spring, .NET Core DI, NestJS) ที่ scan class ทั้งหมด อ่าน annotation แล้ว "ประกอบ" (wire) ให้อัตโนมัติผ่าน reflection ในเวลา runtime — เกิด magic ที่มองไม่เห็นด้วยตาเปล่า debug ยาก และมี runtime overhead

ใน Go เราแค่**เขียนโค้ดธรรมดา** ที่ `main.go` — สร้าง concrete struct ตรงๆ แล้วส่งเข้า constructor ที่รับ interface:

```go
repo := postgres.New(db)              // concrete type ที่ implement domain.ProductRepository
svc := usecase.NewProductService(repo) // รับ interface — ไม่สนใจว่า repo เป็น concrete type ไหน
```

ไม่มี reflection ไม่มี annotation ไม่มี container ที่ซ่อนความสัมพันธ์ไว้ที่ไหนสักแห่ง — ทุกอย่างคือ **plain Go code ที่ compiler เช็คให้ตั้งแต่ compile time** จุดที่โค้ดจะ import อะไรบ้างมองเห็นได้ตรงๆ จากบรรทัด `import` เอง เรื่องนี้คือเหตุผลสำคัญที่หลายคนบอกว่า **Go ทำ dependency injection แบบ "manual" ได้เนียนกว่าและปลอดภัยกว่าภาษาที่มี DI framework เต็มรูปแบบเสียอีก**

---

## 5. โครงสร้างโปรเจกต์: จัดวางไฟล์อย่างไรให้สื่อความหมาย

ทบทวนโครงสร้างโปรเจกต์มาตรฐานจาก **Part 001** ที่กล่าวถึง `cmd/`, `internal/`, `pkg/` — Clean Architecture ใน Go มักจัดวางไฟล์แบบนี้:

```
productapp/
├── go.mod
├── cmd/
│   └── productapp/
│       └── main.go              # composition root: ประกอบทุกชั้นเข้าด้วยกันที่นี่ที่เดียว
└── internal/
    ├── domain/
    │   └── product.go            # entity + repository interface
    ├── usecase/
    │   └── product_service.go    # business logic
    ├── repository/
    │   ├── memory/
    │   │   └── memory.go         # implementation สำหรับ test/demo
    │   └── postgres/
    │       └── postgres.go       # implementation จริงสำหรับ production
    └── delivery/
        └── http/
            └── handler.go        # HTTP handler
```

จุดสำคัญ: ทุกอย่างอยู่ใต้ `internal/` เพื่อใช้กฎพิเศษของ Go compiler (ทบทวนจาก **Part 001**) ที่ห้ามโปรเจกต์อื่น import โค้ดใต้ `internal/` ได้ — บังคับ encapsulation ระดับโปรเจกต์ ส่วน `cmd/productapp/main.go` คือจุดเดียวในทั้งระบบที่ **รู้จักทุกชั้นพร้อมกัน** (เรียกว่า **composition root**) เพราะมันมีหน้าที่ "ประกอบ" ทุกอย่างเข้าด้วยกัน

---

## 6. ตัวอย่างเต็ม: Product Service แบบ Clean Architecture (รันได้จริง)

มาสร้างระบบจัดการสินค้าเล็กๆ ที่ใช้ครบทั้ง 4 ชั้น ตัวอย่างนี้ถูกสร้างและรันจริงเพื่อยืนยันว่า compile และทำงานถูกต้องทุกประการ

### ชั้นที่ 1: Domain

```go
// internal/domain/product.go
//
// Package domain คือชั้นในสุดของ Clean Architecture — เก็บ entity และกฎทางธุรกิจ
// ที่เป็นแก่นแท้ที่สุดของแอปพลิเคชัน package นี้ไม่ import อะไรจาก usecase,
// repository หรือ delivery เลยแม้แต่น้อย (dependency rule: ลูกศรความรู้จักชี้เข้าหาศูนย์กลางเสมอ)
package domain

import (
	"context"
	"errors"
)

// Product คือ entity หลักของระบบ
//
// หมายเหตุ: การใส่ struct tag `json:"..."` ตรงนี้เป็นการยอมทำผิดหลัก "dependency
// rule" อย่างเคร่งครัดเล็กน้อยเพื่อความสะดวก (encoding/json ทางเทคนิคเป็นเรื่องของ
// delivery layer) โปรเจกต์ที่เข้มงวดมากจะแยก DTO (Data Transfer Object) ต่างหาก
// ในชั้น delivery แล้วแปลง entity ไป-กลับเอง แต่สำหรับสเกลขนาดเล็ก-กลาง การแปะ
// json tag ไว้ที่ entity ตรงๆ เป็นการประนีประนอมที่ทีม Go จำนวนมากยอมรับได้
// (ทบทวนเรื่อง struct tag จาก Part 025)
type Product struct {
	ID    string  `json:"id"`
	Name  string  `json:"name"`
	Price float64 `json:"price"`
	Stock int     `json:"stock"`
}

// sentinel errors ของ domain — ทบทวนแนวคิดจาก Part 016
var (
	ErrProductNotFound    = errors.New("product: not found")
	ErrInvalidName        = errors.New("product: name must not be empty")
	ErrInvalidPrice       = errors.New("product: price must be greater than zero")
	ErrInsufficientStock  = errors.New("product: insufficient stock")
	ErrInvalidPurchaseQty = errors.New("product: purchase quantity must be greater than zero")
)

// ProductRepository คือ "port" ของ Clean Architecture — สัญญาที่ชั้น domain
// เป็นคนกำหนดขึ้นเอง โดยไม่รู้จักและไม่สนใจเลยว่าใครจะเป็นคน implement
// (จะเป็น in-memory map, PostgreSQL, MongoDB หรือไฟล์ text ก็ได้ทั้งนั้น)
//
// นี่คือจุดที่ implicit interface satisfaction ของ Go (Part 013) ทรงพลังที่สุด:
// เราประกาศ interface นี้ไว้ตรงนี้ได้เลย โดยที่ยังไม่มี concrete type ใดๆ
// implement มันอยู่แม้แต่ตัวเดียวในตอนที่เขียนบรรทัดนี้
type ProductRepository interface {
	Create(ctx context.Context, p *Product) error
	FindByID(ctx context.Context, id string) (*Product, error)
	List(ctx context.Context) ([]*Product, error)
	UpdateStock(ctx context.Context, id string, delta int) error
}
```

### ชั้นที่ 2: Usecase

```go
// internal/usecase/product_service.go
//
// Package usecase คือชั้น "business logic" ของแอปพลิเคชัน — ประสานงานระหว่าง
// domain entity กับ repository เพื่อทำ operation ที่มีความหมายทางธุรกิจ
//
// สังเกตว่า package นี้ import แค่ "productapp/internal/domain" เท่านั้น
// ไม่รู้จัก net/http, database/sql หรือ package repository ตัวใดเลย —
// นี่คือหัวใจของ dependency rule: usecase ไม่ขึ้นกับ framework หรือ database ใดๆ
package usecase

import (
	"context"
	"fmt"

	"productapp/internal/domain"
)

// ProductService ถือ ProductRepository ไว้ในรูปแบบ "interface" ไม่ใช่ concrete type
// ทำให้ service ตัวนี้ทำงานได้กับ repository ตัวไหนก็ได้ที่ implement
// domain.ProductRepository — ไม่ว่าจะเป็น in-memory (สำหรับ test) หรือ PostgreSQL จริง (production)
type ProductService struct {
	repo domain.ProductRepository
}

// NewProductService รับ dependency ผ่าน constructor (dependency injection แบบ explicit
// ไม่มี framework วิเศษใดๆ มาช่วย — แค่ส่ง interface เข้ามาตรงๆ)
func NewProductService(repo domain.ProductRepository) *ProductService {
	return &ProductService{repo: repo}
}

// CreateProduct สร้างสินค้าใหม่ พร้อม validate กฎทางธุรกิจก่อนบันทึกเสมอ
// นี่คือตัวอย่างว่าทำไม business logic ควรอยู่ที่ usecase ไม่ใช่กระจัดกระจาย
// อยู่ใน handler หรือ repository
func (s *ProductService) CreateProduct(ctx context.Context, id, name string, price float64, stock int) (*domain.Product, error) {
	if name == "" {
		return nil, domain.ErrInvalidName
	}
	if price <= 0 {
		return nil, domain.ErrInvalidPrice
	}
	if stock < 0 {
		return nil, fmt.Errorf("product: stock must not be negative")
	}

	p := &domain.Product{ID: id, Name: name, Price: price, Stock: stock}
	if err := s.repo.Create(ctx, p); err != nil {
		return nil, fmt.Errorf("create product: %w", err)
	}
	return p, nil
}

// GetProduct ดึงสินค้าตาม id
func (s *ProductService) GetProduct(ctx context.Context, id string) (*domain.Product, error) {
	p, err := s.repo.FindByID(ctx, id)
	if err != nil {
		return nil, fmt.Errorf("get product: %w", err)
	}
	return p, nil
}

// ListProducts คืนสินค้าทั้งหมดในระบบ
func (s *ProductService) ListProducts(ctx context.Context) ([]*domain.Product, error) {
	products, err := s.repo.List(ctx)
	if err != nil {
		return nil, fmt.Errorf("list products: %w", err)
	}
	return products, nil
}

// PurchaseProduct คือตัวอย่าง business rule ที่แท้จริง: ต้องเช็คว่าสต็อกพอก่อนเสมอ
// ก่อนจะสั่งลด — กฎนี้อยู่ที่ usecase ไม่ใช่ที่ repository เพราะเป็นเรื่องของ
// "ธุรกิจ" ไม่ใช่เรื่องของ "การเก็บข้อมูล"
func (s *ProductService) PurchaseProduct(ctx context.Context, id string, qty int) error {
	if qty <= 0 {
		return domain.ErrInvalidPurchaseQty
	}

	p, err := s.repo.FindByID(ctx, id)
	if err != nil {
		return fmt.Errorf("purchase product: %w", err)
	}
	if p.Stock < qty {
		return domain.ErrInsufficientStock
	}

	if err := s.repo.UpdateStock(ctx, id, -qty); err != nil {
		return fmt.Errorf("purchase product: %w", err)
	}
	return nil
}
```

สังเกตว่า `PurchaseProduct` คือหัวใจของบทเรียนนี้: มันเช็ค `p.Stock < qty` **ก่อน**ที่จะสั่งลดสต็อก — ถ้ากฎนี้เขียนอยู่ใน SQL handler โดยตรงแบบตัวอย่างในหัวข้อ 1 ก็จะต้อง copy ไปวางซ้ำทุกที่ที่ต้องเช็คสต็อก แต่ตอนนี้มันอยู่ที่เดียว เรียกซ้ำได้จากทุกช่องทาง (HTTP, gRPC, CLI, cron job) โดยไม่ต้องเขียนซ้ำเลย

### ชั้นที่ 3a: Repository — In-Memory (สำหรับ Demo และ Test)

```go
// internal/repository/memory/memory.go
//
// Package memory คือ concrete implementation ของ domain.ProductRepository ตัวหนึ่ง
// เก็บข้อมูลไว้ใน memory ล้วนๆ ด้วย map — เหมาะกับ unit test และตัวอย่างสาธิต
// ที่ไม่ต้องพึ่งฐานข้อมูลจริง
package memory

import (
	"context"
	"fmt"
	"sync"

	"productapp/internal/domain"
)

// ProductRepository implement domain.ProductRepository แบบ implicit —
// ไม่มีบรรทัดไหนเขียนว่า "implements domain.ProductRepository" เลยสักตัวอักษร
// (ทบทวน Part 013: implicit satisfaction) compiler จะเช็คให้เองตอน compile time
// ผ่านการ assign ที่ท้ายไฟล์นี้
type ProductRepository struct {
	mu     sync.Mutex
	data   map[string]*domain.Product
	nextID int
}

func New() *ProductRepository {
	return &ProductRepository{data: make(map[string]*domain.Product)}
}

func (r *ProductRepository) Create(_ context.Context, p *domain.Product) error {
	r.mu.Lock()
	defer r.mu.Unlock()

	if p.ID == "" {
		r.nextID++
		p.ID = fmt.Sprintf("prod-%d", r.nextID)
	}
	cp := *p // เก็บสำเนา ป้องกันไม่ให้ผู้เรียกแก้ struct ต้นฉบับแล้วกระทบข้อมูลใน repository โดยไม่ตั้งใจ
	r.data[p.ID] = &cp
	return nil
}

func (r *ProductRepository) FindByID(_ context.Context, id string) (*domain.Product, error) {
	r.mu.Lock()
	defer r.mu.Unlock()

	p, ok := r.data[id]
	if !ok {
		return nil, domain.ErrProductNotFound
	}
	cp := *p
	return &cp, nil
}

func (r *ProductRepository) List(_ context.Context) ([]*domain.Product, error) {
	r.mu.Lock()
	defer r.mu.Unlock()

	result := make([]*domain.Product, 0, len(r.data))
	for _, p := range r.data {
		cp := *p
		result = append(result, &cp)
	}
	return result, nil
}

func (r *ProductRepository) UpdateStock(_ context.Context, id string, delta int) error {
	r.mu.Lock()
	defer r.mu.Unlock()

	p, ok := r.data[id]
	if !ok {
		return domain.ErrProductNotFound
	}
	if p.Stock+delta < 0 {
		return domain.ErrInsufficientStock
	}
	p.Stock += delta
	return nil
}

// บรรทัดนี้คือ "compile-time interface check" idiom ที่เจอมาแล้วใน Part 048:
// ถ้า *ProductRepository ไม่มี method ครบตามที่ domain.ProductRepository ต้องการ
// โปรแกรมจะ compile ไม่ผ่านทันที ณ บรรทัดนี้ (ไม่ต้องรอไปเจอ error ตอนรันจริง)
var _ domain.ProductRepository = (*ProductRepository)(nil)
```

### ชั้นที่ 4: Delivery (HTTP)

```go
// internal/delivery/http/handler.go
//
// Package http คือชั้น "delivery" (บางตำราเรียก "interface adapter" หรือ
// "presentation layer") — หน้าที่เดียวของมันคือแปลง HTTP request/response
// ให้เป็นการเรียก usecase และแปลงผลลัพธ์กลับเป็น JSON เท่านั้น
// ไม่มี business logic ใดๆ ปนอยู่ในไฟล์นี้เลย (validate ราคา/สต็อกเป็นหน้าที่ของ usecase)
package http

import (
	"encoding/json"
	"errors"
	"net/http"

	"productapp/internal/domain"
	"productapp/internal/usecase"
)

// ProductHandler พึ่งพา *usecase.ProductService เท่านั้น ไม่รู้จัก repository
// ตัวใดเลยไม่ว่าจะเป็น memory หรือ postgres — สอดคล้องกับ dependency rule เต็มรูปแบบ
type ProductHandler struct {
	svc *usecase.ProductService
}

func NewProductHandler(svc *usecase.ProductService) *ProductHandler {
	return &ProductHandler{svc: svc}
}

// RegisterRoutes ผูก route ทั้งหมดเข้ากับ mux ที่ส่งเข้ามา
// ใช้ pattern แบบ Go 1.22+ ServeMux ที่รองรับ method + path parameter ในตัว
// (ทบทวนจาก Part 056)
func (h *ProductHandler) RegisterRoutes(mux *http.ServeMux) {
	mux.HandleFunc("POST /products", h.create)
	mux.HandleFunc("GET /products", h.list)
	mux.HandleFunc("GET /products/{id}", h.get)
	mux.HandleFunc("POST /products/{id}/purchase", h.purchase)
}

type createProductRequest struct {
	Name  string  `json:"name"`
	Price float64 `json:"price"`
	Stock int     `json:"stock"`
}

type errorResponse struct {
	Error string `json:"error"`
}

func writeJSON(w http.ResponseWriter, status int, v any) {
	w.Header().Set("Content-Type", "application/json")
	w.WriteHeader(status)
	json.NewEncoder(w).Encode(v)
}

func writeError(w http.ResponseWriter, status int, err error) {
	writeJSON(w, status, errorResponse{Error: err.Error()})
}

func (h *ProductHandler) create(w http.ResponseWriter, r *http.Request) {
	var req createProductRequest
	if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
		writeError(w, http.StatusBadRequest, err)
		return
	}

	p, err := h.svc.CreateProduct(r.Context(), "", req.Name, req.Price, req.Stock)
	if err != nil {
		// validation error จาก usecase ถือเป็น 422 (ทบทวนหลักการ 400 vs 422 จาก Part 059)
		writeError(w, http.StatusUnprocessableEntity, err)
		return
	}
	writeJSON(w, http.StatusCreated, p)
}

func (h *ProductHandler) list(w http.ResponseWriter, r *http.Request) {
	products, err := h.svc.ListProducts(r.Context())
	if err != nil {
		writeError(w, http.StatusInternalServerError, err)
		return
	}
	writeJSON(w, http.StatusOK, products)
}

func (h *ProductHandler) get(w http.ResponseWriter, r *http.Request) {
	id := r.PathValue("id")
	p, err := h.svc.GetProduct(r.Context(), id)
	if errors.Is(err, domain.ErrProductNotFound) {
		writeError(w, http.StatusNotFound, err)
		return
	}
	if err != nil {
		writeError(w, http.StatusInternalServerError, err)
		return
	}
	writeJSON(w, http.StatusOK, p)
}

type purchaseRequest struct {
	Quantity int `json:"quantity"`
}

func (h *ProductHandler) purchase(w http.ResponseWriter, r *http.Request) {
	id := r.PathValue("id")
	var req purchaseRequest
	if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
		writeError(w, http.StatusBadRequest, err)
		return
	}

	err := h.svc.PurchaseProduct(r.Context(), id, req.Quantity)
	switch {
	case errors.Is(err, domain.ErrProductNotFound):
		writeError(w, http.StatusNotFound, err)
	case errors.Is(err, domain.ErrInsufficientStock), errors.Is(err, domain.ErrInvalidPurchaseQty):
		writeError(w, http.StatusUnprocessableEntity, err)
	case err != nil:
		writeError(w, http.StatusInternalServerError, err)
	default:
		w.WriteHeader(http.StatusNoContent)
	}
}
```

### Composition Root: `main.go`

```go
// cmd/productapp/main.go
//
// main.go คือจุดเดียวในทั้งโปรเจกต์ที่ "รู้จัก" ทุกชั้นพร้อมกัน — มันคือที่ที่เรา
// ตัดสินใจว่าจะใช้ repository ตัวไหนจริงๆ (memory หรือ postgres) แล้ว "ประกอบ"
// (wire) ทุกชั้นเข้าด้วยกันแบบ explicit ล้วนๆ ไม่มี framework DI วิเศษใดๆ มาช่วย
//
// สังเกตว่าการจะสลับจาก memory ไป postgres ทำได้แค่เปลี่ยน 1 บรรทัดตรงนี้
// (เปลี่ยนจาก memory.New() เป็น postgres.New(db)) โดยไม่ต้องแก้โค้ดใน usecase
// หรือ delivery แม้แต่บรรทัดเดียว — นี่คือประโยชน์ที่จับต้องได้ของ Clean Architecture
package main

import (
	"bytes"
	"encoding/json"
	"fmt"
	"net/http"
	"net/http/httptest"

	productHTTP "productapp/internal/delivery/http"
	"productapp/internal/repository/memory"
	"productapp/internal/usecase"
)

func main() {
	// ----- Composition Root: ประกอบทุกชั้นเข้าด้วยกันตรงนี้ที่เดียว -----
	repo := memory.New() // เปลี่ยนเป็น postgres.New(db) ได้ทันทีถ้ามี PostgreSQL จริง
	svc := usecase.NewProductService(repo)
	handler := productHTTP.NewProductHandler(svc)

	mux := http.NewServeMux()
	handler.RegisterRoutes(mux)

	// ใช้ httptest.NewServer เพื่อสาธิตการทำงานของทั้งระบบแบบ end-to-end
	// ผ่าน HTTP จริง (ทบทวน httptest จาก Part 081) โดยไม่ต้องผูก port ค้างไว้
	server := httptest.NewServer(mux)
	defer server.Close()

	fmt.Println("=== 1. สร้างสินค้าใหม่ (POST /products) ===")
	created := createProduct(server.URL, "Mechanical Keyboard", 2500, 10)
	fmt.Printf("สร้างสำเร็จ: %+v\n\n", created)

	fmt.Println("=== 2. ดึงสินค้าตาม id (GET /products/{id}) ===")
	fetched := getProduct(server.URL, created["id"].(string))
	fmt.Printf("พบสินค้า: %+v\n\n", fetched)

	fmt.Println("=== 3. สั่งซื้อสินค้า 3 ชิ้น (POST /products/{id}/purchase) ===")
	status := purchaseProduct(server.URL, created["id"].(string), 3)
	fmt.Printf("HTTP status หลังสั่งซื้อ: %d\n\n", status)

	fmt.Println("=== 4. ดึงสินค้าอีกครั้ง เพื่อยืนยันว่าสต็อกลดลง ===")
	afterPurchase := getProduct(server.URL, created["id"].(string))
	fmt.Printf("สต็อกคงเหลือ: %v\n\n", afterPurchase["stock"])

	fmt.Println("=== 5. พยายามสั่งซื้อเกินสต็อก (คาดหวัง 422) ===")
	overStatus := purchaseProduct(server.URL, created["id"].(string), 999)
	fmt.Printf("HTTP status: %d (422 = Unprocessable Entity ตามที่ตั้งใจ)\n\n", overStatus)

	fmt.Println("=== 6. สร้างสินค้าอีกตัว แล้ว List ทั้งหมด ===")
	createProduct(server.URL, "USB-C Hub", 690, 25)
	all := listProducts(server.URL)
	fmt.Printf("จำนวนสินค้าทั้งหมดในระบบ: %d รายการ\n", len(all))
}

func createProduct(baseURL, name string, price float64, stock int) map[string]any {
	body, _ := json.Marshal(map[string]any{"name": name, "price": price, "stock": stock})
	resp, err := http.Post(baseURL+"/products", "application/json", bytes.NewReader(body))
	if err != nil {
		panic(err)
	}
	defer resp.Body.Close()
	var result map[string]any
	json.NewDecoder(resp.Body).Decode(&result)
	return result
}

func getProduct(baseURL, id string) map[string]any {
	resp, err := http.Get(baseURL + "/products/" + id)
	if err != nil {
		panic(err)
	}
	defer resp.Body.Close()
	var result map[string]any
	json.NewDecoder(resp.Body).Decode(&result)
	return result
}

func purchaseProduct(baseURL, id string, qty int) int {
	body, _ := json.Marshal(map[string]any{"quantity": qty})
	resp, err := http.Post(baseURL+"/products/"+id+"/purchase", "application/json", bytes.NewReader(body))
	if err != nil {
		panic(err)
	}
	defer resp.Body.Close()
	return resp.StatusCode
}

func listProducts(baseURL string) []map[string]any {
	resp, err := http.Get(baseURL + "/products")
	if err != nil {
		panic(err)
	}
	defer resp.Body.Close()
	var result []map[string]any
	json.NewDecoder(resp.Body).Decode(&result)
	return result
}
```

รันด้วย `go run ./cmd/productapp` ผลลัพธ์จริง:

```
=== 1. สร้างสินค้าใหม่ (POST /products) ===
สร้างสำเร็จ: map[id:prod-1 name:Mechanical Keyboard price:2500 stock:10]

=== 2. ดึงสินค้าตาม id (GET /products/{id}) ===
พบสินค้า: map[id:prod-1 name:Mechanical Keyboard price:2500 stock:10]

=== 3. สั่งซื้อสินค้า 3 ชิ้น (POST /products/{id}/purchase) ===
HTTP status หลังสั่งซื้อ: 204

=== 4. ดึงสินค้าอีกครั้ง เพื่อยืนยันว่าสต็อกลดลง ===
สต็อกคงเหลือ: 7

=== 5. พยายามสั่งซื้อเกินสต็อก (คาดหวัง 422) ===
HTTP status: 422 (422 = Unprocessable Entity ตามที่ตั้งใจ)

=== 6. สร้างสินค้าอีกตัว แล้ว List ทั้งหมด ===
จำนวนสินค้าทั้งหมดในระบบ: 2 รายการ
```

โปรเจกต์ทั้งชุดนี้ถูก build, vet, run และ format-check จริงด้วย `go build ./...`, `go vet ./...`, `go run ./cmd/productapp` และ `gofmt -l .` — ทุกคำสั่งผ่านสะอาดและ output ที่แสดงคือ output จริงจากการรัน

---

## 7. ทดสอบ Business Logic โดยไม่ต้องมีฐานข้อมูลจริง

นี่คือประโยชน์ที่จับต้องได้ที่สุดของ Clean Architecture — เพราะ `ProductService` รับ `domain.ProductRepository` เป็น **interface** เราจึงส่ง `memory.New()` เข้าไปแทน PostgreSQL จริงได้ทันทีตอนเขียน unit test ทำให้ test รันเร็วมาก (ไม่มี network I/O เลย) และไม่ flaky จากปัญหาเชื่อมต่อฐานข้อมูล:

```go
// internal/usecase/product_service_test.go
package usecase_test

import (
	"context"
	"errors"
	"testing"

	"productapp/internal/domain"
	"productapp/internal/repository/memory"
	"productapp/internal/usecase"
)

// จุดสำคัญที่ควรสังเกตในไฟล์ test นี้: เราทดสอบ business logic ของ ProductService
// ได้เต็มรูปแบบโดยไม่ต้องมีฐานข้อมูลจริงสักตัว เพราะ ProductService รับ
// domain.ProductRepository เป็น interface — เราจึงส่ง memory.New() (ซึ่งเร็วมาก
// และไม่ต้องพึ่ง network) เข้าไปแทนได้เลย นี่คือประโยชน์โดยตรงของ dependency
// inversion ที่ interface ทำให้เกิดขึ้นแบบธรรมชาติใน Go (ทบทวน Part 035: Mocking)
func TestPurchaseProduct_InsufficientStock(t *testing.T) {
	repo := memory.New()
	svc := usecase.NewProductService(repo)
	ctx := context.Background()

	p, err := svc.CreateProduct(ctx, "", "Mouse", 590, 2)
	if err != nil {
		t.Fatalf("CreateProduct: %v", err)
	}

	err = svc.PurchaseProduct(ctx, p.ID, 5)
	if !errors.Is(err, domain.ErrInsufficientStock) {
		t.Fatalf("want ErrInsufficientStock, got %v", err)
	}
}

func TestPurchaseProduct_Success(t *testing.T) {
	repo := memory.New()
	svc := usecase.NewProductService(repo)
	ctx := context.Background()

	p, err := svc.CreateProduct(ctx, "", "Mouse", 590, 10)
	if err != nil {
		t.Fatalf("CreateProduct: %v", err)
	}

	if err := svc.PurchaseProduct(ctx, p.ID, 4); err != nil {
		t.Fatalf("PurchaseProduct: %v", err)
	}

	got, err := svc.GetProduct(ctx, p.ID)
	if err != nil {
		t.Fatalf("GetProduct: %v", err)
	}
	if got.Stock != 6 {
		t.Fatalf("want stock 6, got %d", got.Stock)
	}
}

func TestCreateProduct_InvalidPrice(t *testing.T) {
	svc := usecase.NewProductService(memory.New())
	_, err := svc.CreateProduct(context.Background(), "", "Broken Item", 0, 1)
	if !errors.Is(err, domain.ErrInvalidPrice) {
		t.Fatalf("want ErrInvalidPrice, got %v", err)
	}
}
```

รันด้วย `go test ./...` — ผลลัพธ์จริง:

```
ok  	productapp/internal/usecase	0.002s
```

test ทั้งสามตัวผ่านในเวลา 2 มิลลิวินาที เพราะไม่มีการเชื่อมต่อเครือข่ายใดๆ เกิดขึ้นเลย เทียบกับถ้าต้องมี PostgreSQL จริงรันอยู่ (ทบทวนความแตกต่างของ unit test vs integration test จาก **Part 079–080**) — นี่คือเหตุผลที่ทีมวิศวกรรมจำนวนมากยืนกรานให้ business logic ทดสอบได้โดยไม่ต้องพึ่งฐานข้อมูลจริงเสมอ

---

## 8. สลับ Repository จาก In-Memory เป็น PostgreSQL โดยไม่แตะ Business Logic

มาดูกันว่า concrete implementation ตัวที่สองที่คุยกับ PostgreSQL จริงหน้าตาเป็นอย่างไร — ใช้ `database/sql` ตามที่เรียนเจาะลึกใน **Part 071**:

```go
// internal/repository/postgres/postgres.go
//
// Package postgres คือ concrete implementation อีกตัวหนึ่งของ domain.ProductRepository
// ตัวนี้คุยกับ PostgreSQL จริงผ่าน database/sql (ทบทวนเจาะลึกจาก Part 071)
//
// หมายเหตุสำหรับผู้เรียน: ไฟล์นี้ใช้ "database/sql" ล้วนๆ (ไม่ import driver
// เช่น github.com/lib/pq ตรงๆ ในไฟล์นี้) — ในโปรเจกต์จริง main.go จะเป็นคน
// blank-import driver (`_ "github.com/lib/pq"`) แล้วส่ง *sql.DB ที่เชื่อมต่อ
// เสร็จแล้วเข้ามาที่ New() ตรงนี้ ตัว repository เองไม่จำเป็นต้องรู้จักชื่อ driver เลย
package postgres

import (
	"context"
	"database/sql"
	"fmt"

	"productapp/internal/domain"
)

// ProductRepository implement domain.ProductRepository เหมือนกับ memory.ProductRepository
// เป๊ะทุกประการในแง่ "หน้าตาสัญญา" (signature) แต่ข้างในคุยกับฐานข้อมูลจริงแทน map
type ProductRepository struct {
	db *sql.DB
}

// New รับ *sql.DB ที่เชื่อมต่อ (และ Ping สำเร็จแล้ว) เข้ามาจากภายนอกเสมอ —
// repository ไม่รับผิดชอบเรื่องเปิด/ปิด connection pool เอง (แยกความรับผิดชอบ
// ตามหลักการจาก Part 071 หัวข้อ 3)
func New(db *sql.DB) *ProductRepository {
	return &ProductRepository{db: db}
}

func (r *ProductRepository) Create(ctx context.Context, p *domain.Product) error {
	const q = `INSERT INTO products (id, name, price, stock) VALUES ($1, $2, $3, $4)`
	if _, err := r.db.ExecContext(ctx, q, p.ID, p.Name, p.Price, p.Stock); err != nil {
		return fmt.Errorf("postgres: create product: %w", err)
	}
	return nil
}

func (r *ProductRepository) FindByID(ctx context.Context, id string) (*domain.Product, error) {
	const q = `SELECT id, name, price, stock FROM products WHERE id = $1`
	var p domain.Product
	err := r.db.QueryRowContext(ctx, q, id).Scan(&p.ID, &p.Name, &p.Price, &p.Stock)
	if err == sql.ErrNoRows {
		return nil, domain.ErrProductNotFound
	}
	if err != nil {
		return nil, fmt.Errorf("postgres: find product: %w", err)
	}
	return &p, nil
}

func (r *ProductRepository) List(ctx context.Context) ([]*domain.Product, error) {
	const q = `SELECT id, name, price, stock FROM products ORDER BY id`
	rows, err := r.db.QueryContext(ctx, q)
	if err != nil {
		return nil, fmt.Errorf("postgres: list products: %w", err)
	}
	defer rows.Close()

	var products []*domain.Product
	for rows.Next() {
		var p domain.Product
		if err := rows.Scan(&p.ID, &p.Name, &p.Price, &p.Stock); err != nil {
			return nil, fmt.Errorf("postgres: scan product: %w", err)
		}
		products = append(products, &p)
	}
	if err := rows.Err(); err != nil {
		return nil, fmt.Errorf("postgres: rows error: %w", err)
	}
	return products, nil
}

func (r *ProductRepository) UpdateStock(ctx context.Context, id string, delta int) error {
	const q = `UPDATE products SET stock = stock + $1 WHERE id = $2 AND stock + $1 >= 0`
	res, err := r.db.ExecContext(ctx, q, delta, id)
	if err != nil {
		return fmt.Errorf("postgres: update stock: %w", err)
	}
	affected, err := res.RowsAffected()
	if err != nil {
		return fmt.Errorf("postgres: update stock: %w", err)
	}
	if affected == 0 {
		// ไม่มีแถวถูกแก้เลย: อาจเพราะไม่พบ id หรือสต็อกไม่พอ (เงื่อนไข stock + $1 >= 0 ไม่ผ่าน)
		return domain.ErrInsufficientStock
	}
	return nil
}

var _ domain.ProductRepository = (*ProductRepository)(nil)
```

สังเกต `UpdateStock` ที่ฉลาดกว่าเวอร์ชัน memory เล็กน้อย: มันใช้เงื่อนไข `AND stock + $1 >= 0` ใน `WHERE` clause โดยตรง เพื่อป้องกัน **race condition** ระหว่างการเช็คสต็อกกับการลดสต็อก — ถ้ามีสอง request มาซื้อสินค้าตัวสุดท้ายพร้อมกัน ฐานข้อมูลจะรับประกันให้แค่ request เดียวเท่านั้นที่ `UPDATE` สำเร็จ (แถวถูกกระทบ 0 แถวสำหรับอีก request) โดยไม่ต้องพึ่ง application-level lock เลย นี่คือตัวอย่างการดึงจุดแข็งของฐานข้อมูลมาใช้แก้ปัญหา concurrency แทนที่จะแก้ที่ฝั่ง Go (ทบทวน transaction/locking เจาะลึกจาก **Part 078**)

การจะสลับใช้งานจริง แค่เปลี่ยน 1 บรรทัดใน `main.go`:

```go
// จาก
repo := memory.New()

// เป็น
db, err := sql.Open("postgres", dsn)
// ... db.Ping(), defer db.Close() ...
repo := postgres.New(db)
```

`usecase.ProductService`, `internal/delivery/http/handler.go` และ test ทั้งหมดที่เขียนไว้ **ไม่ต้องแก้แม้แต่บรรทัดเดียว** — นี่คือคำมั่นสัญญาหลักของ Clean Architecture ที่พิสูจน์ได้จริงด้วยโค้ด ไม่ใช่แค่ทฤษฎีบนกระดาษ

---

## 9. ข้อควรระวังตรงไปตรงมา: Clean Architecture ไม่ใช่ยาวิเศษ

หลังจากเห็นประโยชน์ทั้งหมดแล้ว มาพูดกันตรงไปตรงมาถึงต้นทุนที่ต้องจ่าย:

### 9.1 Boilerplate และ Indirection ที่เพิ่มขึ้นจริง

จากตัวอย่างข้างบน กว่าจะได้ endpoint `POST /products/{id}/purchase` ที่ทำงานได้หนึ่งตัว ต้องเขียนไฟล์อย่างน้อย 4 ไฟล์ (domain, usecase, repository, delivery) เทียบกับการเขียน handler เดียวจบแบบหัวข้อ 1 ที่ใช้แค่ไฟล์เดียวฟังก์ชันเดียว สำหรับ CRUD ธรรมดาที่ไม่มี business rule ซับซ้อนอะไรเลย การแยกชั้นขนาดนี้**อาจเป็นการเพิ่มความซับซ้อนโดยไม่จำเป็น**

### 9.2 ต้องมี "การแปลข้อมูลไป-มา" ระหว่างชั้น

ทุกครั้งที่ข้อมูลเดินทางข้ามชั้น (domain → usecase → delivery) มักต้องมีการแปลงรูปแบบ (แม้จะเล็กน้อย เช่น entity → JSON) ยิ่งเข้มงวดตามตำรามากเท่าไหร่ (แยก DTO ทุกชั้นอย่างเคร่งครัด) ยิ่งมีโค้ดแปลงข้อมูลเพิ่มขึ้นเท่านั้น

### 9.3 นักพัฒนาใหม่ต้องใช้เวลาทำความเข้าใจโครงสร้าง

โปรเจกต์ CRUD ธรรมดาที่จัดเป็น 4 ชั้นแบบนี้ อาจทำให้นักพัฒนาที่เพิ่งเข้าทีมงงว่า "ทำไมแค่จะเพิ่ม field หนึ่งตัวต้องแก้ไฟล์ตั้ง 4 ไฟล์" — นี่เป็นคำถามที่สมเหตุสมผล และคำตอบคือ **ต้นทุนนี้จะคุ้มค่าก็ต่อเมื่อระบบมีความซับซ้อนทางธุรกิจสูงพอ หรือมีแนวโน้มต้องเปลี่ยนเทคโนโลยีเบื้องหลังบ่อย**

---

## 10. เมื่อไรควรใช้ และเมื่อไรไม่ควรใช้

ทบทวนแนวคิด **"Monolith First"** จาก **Part 088** ที่บอกว่าไม่ควรเริ่มระบบใหม่ด้วย microservices ตั้งแต่วันแรก — หลักการเดียวกันนี้ใช้ได้กับ Clean Architecture:

**ควรใช้เมื่อ:**
- ระบบมี business logic ที่ซับซ้อนจริง มีกฎทางธุรกิจจำนวนมากที่ต้องทดสอบอย่างละเอียด
- คาดว่าจะต้องเปลี่ยนหรือรองรับหลายช่องทาง (REST + gRPC + CLI) หรือหลายฐานข้อมูล
- ทีมมีขนาดใหญ่พอที่ประโยชน์จาก "แบ่งงานตามชั้นที่ชัดเจน" คุ้มกับต้นทุนการประสานงาน
- โปรเจกต์คาดว่าจะมีอายุการใช้งานยาวนาน (หลายปี) ที่การดูแลรักษาระยะยาวสำคัญกว่าความเร็วในการเริ่มต้น

**ไม่ควรใช้ (หรือใช้แบบเบาลง) เมื่อ:**
- โปรเจกต์เป็น prototype, MVP, หรือ script ขนาดเล็กที่ไม่รู้ด้วยซ้ำว่าจะถูกใช้งานต่อในระยะยาวหรือไม่
- CRUD ล้วนๆ ที่แทบไม่มี business rule ที่ซับซ้อน (บันทึกข้อมูลลงตารางตรงๆ) — การแยกชั้นอาจเป็นแค่พิธีกรรมที่ไม่มีประโยชน์จริง
- ทีมมีขนาดเล็กมาก (1-2 คน) ที่ overhead การประสานงานระหว่างชั้นไม่คุ้มกับที่ได้มา

แนวทางที่นักพัฒนาอาวุโสจำนวนมากแนะนำคือ: **เริ่มต้นด้วยโครงสร้างที่เรียบง่ายกว่านี้ก่อน** (เช่น แบ่งแค่ `handler` → `service` → `store` แบบหลวมๆ ไม่ต้องเคร่งครัดตามตำราทุกกระเบียดนิ้ว) แล้ว**ค่อยๆ เพิ่มการแยกชั้นเมื่อความซับซ้อนของระบบเรียกร้องจริงๆ** — เหมือนกับที่ Part 088 แนะนำให้เริ่มจาก monolith ก่อนแล้วค่อยแตกเป็น microservices เมื่อจำเป็นจริง ไม่ใช่ทำทุกอย่างให้ "ถูกต้องตามตำรา" ตั้งแต่บรรทัดแรกโดยไม่ดูบริบทของงานตรงหน้า

---

## สรุปสิ่งที่ได้เรียนในบทนี้

- **Clean Architecture** แก้ปัญหาโค้ดที่ผูก business logic ติดกับ framework/database โดยแยกความรับผิดชอบเป็นชั้นๆ
- **Dependency Rule** คือกฎเดียวที่ต้องจำ: ทิศทาง import ต้องชี้เข้าหาศูนย์กลาง (domain) เสมอ ไม่มีทางชี้ออก
- ชั้นทั่วไปแบบ Go: **Domain** (entity + repository interface) → **Usecase** (business logic) → **Repository implementation** (infrastructure จริง) → **Delivery** (HTTP/gRPC handler)
- **Implicit interface satisfaction** ของ Go (Part 013) ทำให้ dependency injection เกิดขึ้นแบบ explicit ล้วนๆ ผ่านโค้ดธรรมดาที่ `main.go` โดยไม่ต้องพึ่ง DI framework ที่ใช้ reflection
- ตัวอย่าง `ProductService` แสดงให้เห็นว่า business logic (เช่น เช็คสต็อกก่อนขาย) ทดสอบได้เต็มรูปแบบด้วย in-memory repository โดยไม่ต้องมีฐานข้อมูลจริง แล้วสลับไปใช้ PostgreSQL จริงได้โดยไม่แตะ usecase/delivery เลย
- **ข้อควรระวังที่ตรงไปตรงมา**: Clean Architecture เพิ่ม boilerplate และ indirection จริง ไม่คุ้มค่าเสมอไปสำหรับ CRUD ธรรมดาหรือ prototype ขนาดเล็ก
- หลักการ **"Monolith First"** จาก Part 088 ใช้ได้กับที่นี่เช่นกัน: เริ่มเรียบง่ายก่อน แล้วค่อยเพิ่มการแยกชั้นเมื่อความซับซ้อนเรียกร้องจริง

## แบบฝึกหัดท้ายบท

1. ทำตามตัวอย่างในบทนี้ให้ครบทุกไฟล์ แล้วรัน `go build ./...`, `go vet ./...`, `go test ./...` และ `go run ./cmd/productapp` ด้วยตัวเอง ยืนยันว่าได้ output ตรงกับที่แสดงในบทเรียน
2. เพิ่ม method `DeleteProduct(ctx, id) error` เข้าไปใน `domain.ProductRepository`, `usecase.ProductService`, `memory.ProductRepository`, `postgres.ProductRepository` และเพิ่ม route `DELETE /products/{id}` ใน delivery layer — สังเกตว่าต้องแก้ไฟล์กี่ไฟล์ และคิดว่าคุ้มค่ากับความซับซ้อนที่เพิ่มขึ้นหรือไม่สำหรับ operation ง่ายๆ แบบนี้
3. เขียน `internal/repository/mongo/mongo.go` ที่ implement `domain.ProductRepository` ด้วยแนวคิดจาก **Part 076 (MongoDB)** (ไม่ต้องเชื่อมต่อ MongoDB จริงก็ได้ แค่ให้ compile ผ่านและมี method ครบ) เพื่อยืนยันว่าสามารถมี repository ได้มากกว่า 2 ตัวพร้อมกันในโปรเจกต์เดียว
4. เพิ่ม business rule ใหม่ใน `ProductService`: ห้ามสร้างสินค้าที่มีชื่อซ้ำกับสินค้าที่มีอยู่แล้ว (ต้องเรียก `repo.List` มาเช็คก่อน) เขียน unit test ยืนยันว่ากฎนี้ทำงานถูกต้องโดยใช้ `memory.New()` เท่านั้น ไม่ต้องมีฐานข้อมูลจริง
5. ลองจงใจ**ละเมิด** dependency rule โดยเพิ่ม `import "net/http"` เข้าไปใน `internal/domain/product.go` แล้วลองเขียนโค้ดที่ใช้ `http.Request` ใน entity ตรงๆ — อภิปรายว่าทำไมสิ่งนี้ถึงเป็นสัญญาณอันตราย แม้ว่าโค้ดจะยัง compile ผ่านอยู่ก็ตาม (compiler ไม่ได้บังคับ dependency rule ให้เราโดยอัตโนมัติ — วินัยของทีมต่างหากที่บังคับ)
6. เขียนโปรเจกต์เล็กๆ ของตัวเอง (เช่น ระบบจอง "ห้องประชุม") แบบ **ไม่ใช้ Clean Architecture เลย** (handler เดียวทำทุกอย่างแบบหัวข้อ 1) แล้วลอง refactor เป็น 4 ชั้นทีละขั้น สังเกตว่าจุดไหนที่การแยกชั้นช่วยจริง และจุดไหนที่รู้สึกว่าเป็นภาระเกินจำเป็น

---

**ต่อไป**: [Part 101 — Domain-Driven Design (DDD) ใน Go](./101-domain-driven-design.md)
