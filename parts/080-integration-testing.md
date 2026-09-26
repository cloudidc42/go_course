# Part 080: Integration Testing

> ภาคที่ 7: Testing, Tooling, Performance — ตอนที่ 2 จาก 9 (Part 79–87)

## สารบัญของบทนี้

1. Test Pyramid: Unit vs Integration vs End-to-End Test
2. Integration Test คืออะไร และเมื่อไหร่ควรเขียน
3. แยก Test เร็ว/ช้าด้วย Build Tags: `//go:build integration`
4. เตรียมสภาพแวดล้อม: ตรวจสอบ Docker และฐานข้อมูลที่มีอยู่ในเครื่อง
5. ทดสอบกับฐานข้อมูลจริงด้วย `testcontainers-go`
6. ตัวอย่างเต็ม: `UserStore` ที่ทดสอบกับ Postgres จริงในคอนเทนเนอร์
7. รันจริง: แยก Unit Test เร็วออกจาก Integration Test ช้า
8. Setup/Teardown: Migration ก่อนรัน และ Truncate ระหว่าง Test
9. ทางเลือกเมื่อไม่มี Docker: SQLite และฐานข้อมูล In-Memory
10. รัน Integration Test บน CI (ปูทางสู่ Part 098)
11. สรุปสิ่งที่ได้เรียนในบทนี้
12. แบบฝึกหัดท้ายบท

---

## 1. Test Pyramid: Unit vs Integration vs End-to-End Test

**Part 079** พาไปลึกกับ unit test แล้วในหลายมิติ — fixture, temp dir, parallel test, fuzzing, dependency injection ของเวลา แต่ทุกเทคนิคในบทนั้นมีจุดร่วมกันหนึ่งอย่าง: **โค้ดที่ถูกทดสอบไม่เคยแตะสิ่งภายนอกจริงเลย** ไม่มีการเชื่อมต่อฐานข้อมูลจริง ไม่มีการเรียก HTTP ไปยัง service อื่น ไม่มี filesystem จริงนอกจาก `t.TempDir()` ที่ควบคุมได้เต็มที่

แนวคิดที่ช่วยจัดหมวดหมู่ระดับของการทดสอบทั้งหมดที่มีในวงการซอฟต์แวร์ เรียกว่า **Test Pyramid** ซึ่งมักถูกวาดเป็นรูปสามเหลี่ยม 3 ชั้น:

```
        /\
       /  \        End-to-End (E2E) Tests
      /----\        - จำนวนน้อยที่สุด, ช้าที่สุด, เปราะบางที่สุด
     /      \        - ทดสอบทั้งระบบทำงานร่วมกันจริง (browser, API จริง, DB จริง)
    /--------\
   /          \     Integration Tests
  /            \     - จำนวนปานกลาง, ทดสอบว่าหลาย component ทำงานร่วมกันถูกต้อง
 /--------------\    - เช่น โค้ด + ฐานข้อมูลจริง, โค้ด + message queue จริง
/                \
/------------------\ Unit Tests
                      - จำนวนมากที่สุด, เร็วที่สุด, เสถียรที่สุด
                      - ทดสอบฟังก์ชัน/struct เดียวแบบแยกโดด (isolated)
```

| มิติ | Unit Test | Integration Test | End-to-End Test |
|---|---|---|---|
| **ขอบเขตที่ทดสอบ** | ฟังก์ชัน/struct เดียว แยกจากภายนอกทั้งหมด | หลาย component ทำงานร่วมกัน (โค้ด + DB จริง, โค้ด + external service) | ทั้งระบบตั้งแต่ต้นจนจบ เหมือนผู้ใช้จริงใช้งาน |
| **ความเร็ว** | เร็วมาก (microsecond–millisecond) | ปานกลาง–ช้า (มี network/disk I/O จริง) | ช้าที่สุด (วินาที–นาที) |
| **ความเสถียร** | เสถียรมาก ไม่ flaky | เสถียรน้อยลง (ขึ้นกับ container, network) | เปราะบางที่สุด (ขึ้นกับหลายระบบพร้อมกัน) |
| **จำนวนที่ควรมี** | เยอะที่สุด (หลักร้อย–หลักพัน) | ปานกลาง (หลักสิบ–หลักร้อย) | น้อยที่สุด (หลักสิบ) |
| **สิ่งที่จับบั๊กได้ดี** | logic ผิดพลาดในฟังก์ชันเดี่ยว | SQL ผิด, schema ไม่ตรง, การ serialize/deserialize ผิด, transaction ผิดพลาด | ปัญหาที่เกิดจาก component หลายตัวรวมกันจริง เช่น config ผิด, network ปิดกั้น |

หลักการของ Test Pyramid คือ: **ยิ่งอยู่ชั้นบนยิ่งควรมีจำนวนน้อย** เพราะยิ่งช้าและยิ่งเปราะบาง — ถ้าพึ่งพา end-to-end test เป็นหลักในการจับบั๊ก การรัน test suite ทั้งหมดอาจใช้เวลาเป็นชั่วโมงและ fail แบบสุ่มบ่อยจนทีมเริ่มไม่เชื่อผลลัพธ์ของมันอีกต่อไป (ปรากฏการณ์ที่เรียกว่า **flaky test fatigue**) ในทางกลับกัน การมีแต่ unit test อย่างเดียวก็เสี่ยงพลาดบั๊กที่เกิดจาก "ส่วนต่อประสาน" ระหว่าง component เช่น query SQL ที่ compile ผ่านแต่ syntax ผิดกับฐานข้อมูลจริง หรือ mock ที่ไม่ตรงกับพฤติกรรมจริงของฐานข้อมูล (ปัญหานี้เจาะลึกไปแล้วบางส่วนใน **Part 035** เรื่อง test doubles)

บทนี้โฟกัสที่ **ชั้นกลางของปิรามิด — Integration Test** ซึ่งเป็นชั้นที่โปรเจกต์ที่ใช้ฐานข้อมูลจริง (ทบทวนจาก **ภาคที่ 6: Part 071-078**) แทบทุกโปรเจกต์ขาดไม่ได้

---

## 2. Integration Test คืออะไร และเมื่อไหร่ควรเขียน

**Integration Test** คือการทดสอบที่ปล่อยให้โค้ดของเราคุยกับ **dependency ภายนอกจริง** อย่างน้อยหนึ่งตัว แทนที่จะ mock/fake มันแบบที่ทำใน unit test — dependency ที่ว่านี้อาจเป็นฐานข้อมูล (Postgres, MySQL, Redis จาก **Part 072-077**), message queue, หรือแม้แต่ HTTP service ของทีมอื่น

**เมื่อไหร่ควรเขียน integration test แทนที่จะพยายาม mock ทุกอย่าง:**

1. **เมื่อความถูกต้องของ SQL/query เป็นความเสี่ยงหลัก** — mock ของ `database/sql` (ทบทวนจาก **Part 035**) บอกได้แค่ว่าโค้ด Go เรียก method ถูกลำดับหรือไม่ แต่บอกไม่ได้เลยว่า query string ที่ส่งไปจะ syntax ถูกต้องกับฐานข้อมูลจริงหรือเปล่า หรือ constraint ต่างๆ (UNIQUE, FOREIGN KEY) จะทำงานตามที่คาดหวังไหม
2. **เมื่อพฤติกรรมของ dependency ภายนอกซับซ้อนเกินกว่าจะ fake ได้แม่นยำ** — เช่น transaction isolation level, การ lock แถวข้อมูล, หรือพฤติกรรมการ retry ของ driver
3. **เมื่อเคยมีบั๊กที่ unit test (ที่ mock ทุกอย่าง) จับไม่ได้แต่เกิดขึ้นจริงตอน production** — สัญญาณที่บอกว่าทีมต้อง "เพิ่มชั้น" การทดสอบตรงจุดนั้น

**สิ่งที่ integration test ไม่ควรทำ**: integration test **ไม่ใช่**ที่สำหรับทดสอบทุก edge case ของ business logic (นั่นคือหน้าที่ของ unit test ที่เร็วกว่ามาก) — integration test ควรโฟกัสที่ "จุดต่อ" ระหว่างโค้ดกับ dependency ภายนอกเป็นหลัก เช่น "insert แล้ว query กลับมาได้ค่าที่ถูกต้องไหม", "constraint ทำงานไหม", "transaction rollback ถูกต้องไหม" ส่วน edge case ของการคำนวณ, การ validate, หรือ business rule ต่างๆ ควรแยกออกไปทดสอบเป็น unit test ที่ไม่ต้องพึ่งฐานข้อมูลเลย (ดูตัวอย่างการแยกใน หัวข้อ 6)

---

## 3. แยก Test เร็ว/ช้าด้วย Build Tags: `//go:build integration`

ปัญหาสำคัญของ integration test คือ **มันช้ากว่า unit test มาก** (ต้องรอ container เริ่มทำงาน, เชื่อมต่อ network จริง) ถ้าปล่อยให้ `go test ./...` รันทั้ง unit test และ integration test ปนกันเสมอ นักพัฒนาที่แค่อยากเช็คว่าฟังก์ชันที่เพิ่งแก้ยังทำงานถูกต้องจะต้องรอ container ทุกครั้ง ทำให้ "loop การพัฒนา" (แก้โค้ด → รัน test → แก้ต่อ) ช้าลงมากโดยไม่จำเป็น

Go มีกลไกที่ทำเรื่องนี้ได้โดยไม่ต้องพึ่ง library เสริมเลย เรียกว่า **build constraints** (หรือเรียกกันติดปากว่า **build tags**) — เขียนเป็น comment พิเศษที่บรรทัดบนสุดของไฟล์ (ต้องมีบรรทัดว่างคั่นระหว่าง comment นี้กับ `package` เสมอ):

```go
//go:build integration

package store
```

ไฟล์ที่มี `//go:build integration` กำกับไว้จะ**ไม่ถูกคอมไพล์เข้าไปด้วยเลย**เมื่อรัน `go build` หรือ `go test` ตามปกติ ต้องระบุ flag `-tags=integration` เพื่อสั่งให้ compiler รวมไฟล์เหล่านี้เข้ามาด้วยเท่านั้น:

```bash
# รันเฉพาะ test ที่ไม่มี build tag (unit test ที่เร็ว) — เป็นค่า default
go test ./...

# รันทุกอย่างรวมถึงไฟล์ที่มี //go:build integration ด้วย
go test -tags=integration ./...
```

**กฎสำคัญของ syntax `//go:build`:**

1. ต้องอยู่ **บรรทัดบนสุดของไฟล์** ก่อน `package` declaration เท่านั้น (อนุญาตให้มี comment อื่นก่อนหน้าได้ แต่ `//go:build` ต้องมาก่อน `package`)
2. **ต้องมีบรรทัดว่างคั่น** ระหว่าง `//go:build` กับ `package` เสมอ ไม่งั้น Go จะมองว่าเป็นแค่ comment ธรรมดา ไม่ใช่ build constraint
3. รองรับ expression แบบ boolean เต็มรูปแบบ: `//go:build integration && linux`, `//go:build integration || e2e`, `//go:build !integration`
4. ชื่อ tag (เช่น `integration`) เป็นแค่ string ที่เรากำหนดเองได้อิสระ ไม่ใช่ keyword ที่ Go รู้จักพิเศษ — จะตั้งชื่อเป็นอะไรก็ได้ แต่ `integration` เป็นชื่อที่ใช้กันเป็นธรรมเนียมทั่วทั้งวงการ Go

> **หมายเหตุประวัติศาสตร์**: ก่อน Go 1.17 build tag เขียนด้วย syntax เก่า `// +build integration` (มี `+` และไม่มี `go:`) ปัจจุบัน `gofmt` จะช่วยแปลง syntax เก่าให้เป็น `//go:build` ให้อัตโนมัติเมื่อรันกับไฟล์เก่า แต่โค้ดใหม่ควรใช้ `//go:build` เสมอ

ด้วยกลไกนี้ ทีมสามารถกำหนด **นโยบายการรันสองระดับ** ได้ชัดเจน:

- **ระหว่างพัฒนา (local development)**: รัน `go test ./...` เฉยๆ ได้ผลเร็วภายในไม่กี่วินาที
- **ก่อน merge หรือใน CI**: รัน `go test -tags=integration ./...` เพื่อให้แน่ใจว่าทั้ง unit และ integration test ผ่านหมดก่อนขึ้น production

---

## 4. เตรียมสภาพแวดล้อม: ตรวจสอบ Docker และฐานข้อมูลที่มีอยู่ในเครื่อง

ก่อนเขียน integration test กับฐานข้อมูลจริง ต้องมีฐานข้อมูลให้ทดสอบด้วยเสียก่อน วิธีที่นิยมที่สุดในปัจจุบันคือใช้ **Docker** รันฐานข้อมูลขึ้นมาชั่วคราวเฉพาะตอนทดสอบ ตรวจสอบว่าเครื่องมี Docker พร้อมใช้งานหรือไม่ด้วยคำสั่ง:

```bash
which docker
docker info
```

ถ้า `docker info` แสดงข้อมูล server ออกมาได้ (ไม่ error) แปลว่า Docker daemon กำลังทำงานอยู่และพร้อมใช้:

```
Client: Docker Engine - Community
 Version:    29.3.1
Server:
 Containers: 3
  Running: 3
 Server Version: 29.3.1
```

ทดสอบว่า pull image และรัน container ได้จริงด้วย:

```bash
docker run --rm hello-world
```

> **บทเรียนนี้ตรวจสอบแล้วจริง**: เครื่องที่ใช้เขียนบทเรียนนี้มี Docker daemon ทำงานอยู่และรัน container ได้สำเร็จ ดังนั้นตัวอย่างในหัวข้อ 5-6 ทั้งหมดเป็นโค้ดที่ **รันจริงกับ Postgres 16 ในคอนเทนเนอร์จริง** ไม่ใช่แค่ตัวอย่างอ้างอิงเฉยๆ — ยืนยันด้วยผลการรัน `go test -tags=integration -v` ที่แนบไว้ในหัวข้อ 7

ถ้าเครื่องของใครไม่มี Docker หรือ Docker daemon ไม่ทำงาน (พบบ่อยใน CI runner บางประเภท หรือ sandbox ที่จำกัดสิทธิ์) ให้ข้ามไปดูหัวข้อ 9 ที่แนะนำทางเลือกอื่นที่ไม่ต้องพึ่ง Docker เลย

---

## 5. ทดสอบกับฐานข้อมูลจริงด้วย `testcontainers-go`

**testcontainers-go** เป็น library แยกต่างหาก (ไม่ใช่ standard library) ที่ได้รับความนิยมสูงมากในระบบนิเวศ Go สำหรับงาน integration test เพราะทำสิ่งที่สำคัญมาก: **สั่งให้ Docker รันคอนเทนเนอร์ของฐานข้อมูล (หรือ service ใดๆ) ขึ้นมาชั่วคราวจากภายในโค้ด Go โดยตรง** แล้วลบทิ้งให้อัตโนมัติเมื่อ test จบ — ไม่ต้องเขียน `docker-compose.yml` แยกต่างหาก ไม่ต้องสั่ง `docker run` เองก่อนรัน test และที่สำคัญที่สุดคือ **แต่ละนักพัฒนาหรือแต่ละ CI job ได้ฐานข้อมูลที่สะอาดของตัวเอง** ไม่ชนกับใคร

ติดตั้ง dependency ที่จำเป็น:

```bash
go get github.com/testcontainers/testcontainers-go/modules/postgres@latest
go get github.com/jackc/pgx/v5/stdlib@latest
```

> ตัวอย่างในบทเรียนนี้ทดสอบด้วย `testcontainers-go v0.44.0` และ `github.com/jackc/pgx/v5 v5.11.0` (เวอร์ชันจะอัปเดตไปเรื่อยๆ ตามเวลา `go get @latest` จะดึงเวอร์ชันล่าสุด ณ ตอนที่รันเสมอ)

แนวคิดหลักของ `testcontainers-go` module `postgres` (มี module ย่อยสำหรับฐานข้อมูล/service ยอดนิยมอื่นๆ ให้ด้วย เช่น `mysql`, `redis`, `mongodb`, `kafka`):

```go
pgContainer, err := postgres.Run(ctx,
    "postgres:16-alpine",              // Docker image ที่จะใช้
    postgres.WithDatabase("godemo_test"),
    postgres.WithUsername("postgres"),
    postgres.WithPassword("postgres"),
    postgres.BasicWaitStrategies(),    // รอจน Postgres พร้อมรับการเชื่อมต่อจริงก่อนคืนค่ากลับมา
)
```

`postgres.Run` ทำงานเบื้องหลังทั้งหมดนี้ให้:

1. ดึง (pull) Docker image `postgres:16-alpine` มาถ้ายังไม่มีในเครื่อง
2. สร้างและ start container ด้วย environment variable ที่ตั้งค่า username/password/database ให้ตรงกับที่ระบุ
3. **รอ (wait strategy)** จนกว่า Postgres จะพร้อมรับการเชื่อมต่อจริง — จุดนี้สำคัญมาก เพราะ container "start" แล้วไม่ได้แปลว่า Postgres process ข้างในพร้อมรับ connection ทันที (Postgres ต้องใช้เวลา initialize schema ภายในก่อน) `postgres.BasicWaitStrategies()` จะรอจน log message "database system is ready to accept connections" ปรากฏ **และ** port เปิดจริง ก่อนคืนค่ากลับมา — ถ้าไม่มี wait strategy ที่ถูกต้อง test อาจพยายามเชื่อมต่อเร็วเกินไปจนล้มเหลวแบบสุ่ม (flaky)
4. คืน connection string ที่พร้อมใช้งานผ่าน `pgContainer.ConnectionString(ctx, "sslmode=disable")`

---

## 6. ตัวอย่างเต็ม: `UserStore` ที่ทดสอบกับ Postgres จริงในคอนเทนเนอร์

มาดูตัวอย่างที่สมบูรณ์: package `store` ที่มี `UserStore` สำหรับจัดการตาราง `users` ในฐานข้อมูล Postgres จริง (ทบทวน `database/sql` และ `pgx` จาก **Part 071-072**)

```go
// store.go
package store

import (
	"context"
	"database/sql"
	"errors"
	"fmt"
	"strings"
)

// ErrUserNotFound คือ sentinel error เมื่อหา user ตาม id ไม่เจอ
var ErrUserNotFound = errors.New("store: user not found")

// User คือ record ผู้ใช้ในตาราง users
type User struct {
	ID    int64
	Name  string
	Email string
}

// UserStore ห่อหุ้ม *sql.DB สำหรับดำเนินการกับตาราง users
type UserStore struct {
	db *sql.DB
}

// NewUserStore สร้าง UserStore จาก connection pool ที่เปิดไว้แล้ว
func NewUserStore(db *sql.DB) *UserStore {
	return &UserStore{db: db}
}

const schemaSQL = `
CREATE TABLE IF NOT EXISTS users (
	id BIGSERIAL PRIMARY KEY,
	name TEXT NOT NULL,
	email TEXT NOT NULL UNIQUE
);
`

// Migrate สร้างตารางที่จำเป็นทั้งหมด (migration แบบง่ายสุดสำหรับตัวอย่างนี้
// โปรเจกต์จริงมักใช้ library เฉพาะทาง เช่น golang-migrate หรือ Atlas)
func Migrate(ctx context.Context, db *sql.DB) error {
	_, err := db.ExecContext(ctx, schemaSQL)
	if err != nil {
		return fmt.Errorf("store: migrate: %w", err)
	}
	return nil
}

// ValidateEmail ตรวจสอบรูปแบบอีเมลแบบง่ายๆ (ไม่ต้องพึ่งฐานข้อมูลเลย
// จึงเป็นตัวอย่างที่เหมาะกับ "unit test" ล้วนๆ ต่างจาก method อื่นใน store.go
// ที่ต้องพึ่งฐานข้อมูลจริงและควรอยู่ใน integration test)
func ValidateEmail(email string) bool {
	at := strings.IndexByte(email, '@')
	return at > 0 && at < len(email)-1 && strings.Contains(email[at+1:], ".")
}

// Create เพิ่ม user ใหม่ ต้องพึ่งฐานข้อมูลจริงเสมอ
func (s *UserStore) Create(ctx context.Context, name, email string) (*User, error) {
	if name == "" {
		return nil, fmt.Errorf("store: name is required")
	}
	if !ValidateEmail(email) {
		return nil, fmt.Errorf("store: invalid email %q", email)
	}

	var id int64
	err := s.db.QueryRowContext(ctx,
		`INSERT INTO users (name, email) VALUES ($1, $2) RETURNING id`,
		name, email,
	).Scan(&id)
	if err != nil {
		return nil, fmt.Errorf("store: create user: %w", err)
	}
	return &User{ID: id, Name: name, Email: email}, nil
}

// GetByID ดึง user ตาม id คืน ErrUserNotFound ถ้าไม่เจอ
func (s *UserStore) GetByID(ctx context.Context, id int64) (*User, error) {
	var u User
	err := s.db.QueryRowContext(ctx,
		`SELECT id, name, email FROM users WHERE id = $1`, id,
	).Scan(&u.ID, &u.Name, &u.Email)
	if errors.Is(err, sql.ErrNoRows) {
		return nil, ErrUserNotFound
	}
	if err != nil {
		return nil, fmt.Errorf("store: get user: %w", err)
	}
	return &u, nil
}

// Count นับจำนวน user ทั้งหมดในตาราง
func (s *UserStore) Count(ctx context.Context) (int, error) {
	var n int
	err := s.db.QueryRowContext(ctx, `SELECT COUNT(*) FROM users`).Scan(&n)
	if err != nil {
		return 0, fmt.Errorf("store: count users: %w", err)
	}
	return n, nil
}

// TruncateAll ล้างข้อมูลทั้งหมดในตาราง users และรีเซ็ต id ให้เริ่มนับใหม่จาก 1
// ใช้เป็น "teardown" ระหว่าง test แต่ละตัว เพื่อให้แต่ละ test เริ่มต้นด้วยตารางว่างเสมอ
func (s *UserStore) TruncateAll(ctx context.Context) error {
	_, err := s.db.ExecContext(ctx, `TRUNCATE TABLE users RESTART IDENTITY`)
	if err != nil {
		return fmt.Errorf("store: truncate users: %w", err)
	}
	return nil
}
```

สังเกตว่า `ValidateEmail` ไม่แตะฐานข้อมูลเลย จึงแยกไปทดสอบเป็น **unit test ธรรมดา** (ไม่มี build tag) ที่รันเร็วมาก:

```go
// validate_unit_test.go — ไม่มี //go:build tag จึงรันทุกครั้งที่สั่ง `go test ./...` ธรรมดา
package store

import "testing"

func TestValidateEmail(t *testing.T) {
	tests := []struct {
		name  string
		email string
		want  bool
	}{
		{"อีเมลปกติ", "alice@example.com", true},
		{"ไม่มี @", "aliceexample.com", false},
		{"ไม่มี domain", "alice@", false},
		{"ไม่มี . ใน domain", "alice@example", false},
		{"@ อยู่ตำแหน่งแรก", "@example.com", false},
		{"สตริงว่าง", "", false},
	}

	for _, tt := range tests {
		t.Run(tt.name, func(t *testing.T) {
			if got := ValidateEmail(tt.email); got != tt.want {
				t.Errorf("ValidateEmail(%q) = %v, want %v", tt.email, got, tt.want)
			}
		})
	}
}
```

ส่วน method อื่นๆ ที่ต้องพึ่งฐานข้อมูลจริง (`Create`, `GetByID`, `Count`) ถูกทดสอบใน **integration test** ที่มี build tag กำกับไว้ชัดเจน:

```go
// store_integration_test.go
//go:build integration

package store

import (
	"context"
	"database/sql"
	"errors"
	"os"
	"testing"
	"time"

	_ "github.com/jackc/pgx/v5/stdlib"
	"github.com/testcontainers/testcontainers-go/modules/postgres"
)

// testDB คือ connection pool ที่ใช้ร่วมกันทุก integration test ในไฟล์นี้
// เปิดครั้งเดียวใน TestMain แล้วใช้ซ้ำ เพราะการสร้าง container ใหม่ทุก test
// จะทำให้ integration test suite ช้าลงมาก (เทียบกับ unit test ที่ควรรันได้ใน millisecond)
var testDB *sql.DB

// TestMain เป็น fixture ระดับ package (ทบทวนจาก Part 079): สั่ง testcontainers-go
// ให้ดึงและรัน Postgres 16 ใน container จริงหนึ่งตัว รัน migration ครั้งเดียว
// แล้วแชร์ connection pool เดียวกันให้ทุก TestXxx ในไฟล์นี้ใช้งาน
func TestMain(m *testing.M) {
	ctx := context.Background()

	pgContainer, err := postgres.Run(ctx,
		"postgres:16-alpine",
		postgres.WithDatabase("godemo_test"),
		postgres.WithUsername("postgres"),
		postgres.WithPassword("postgres"),
		postgres.BasicWaitStrategies(),
	)
	if err != nil {
		panic("start postgres container: " + err.Error())
	}

	connStr, err := pgContainer.ConnectionString(ctx, "sslmode=disable")
	if err != nil {
		panic("connection string: " + err.Error())
	}

	db, err := sql.Open("pgx", connStr)
	if err != nil {
		panic("open db: " + err.Error())
	}

	pingCtx, cancel := context.WithTimeout(ctx, 10*time.Second)
	defer cancel()
	if err := db.PingContext(pingCtx); err != nil {
		panic("ping db: " + err.Error())
	}

	// รัน migration ครั้งเดียวตอน setup ทั้ง suite — ทุก test ที่ตามมาใช้ schema เดียวกัน
	if err := Migrate(ctx, db); err != nil {
		panic("migrate: " + err.Error())
	}

	testDB = db

	code := m.Run()

	db.Close()
	if err := pgContainer.Terminate(context.Background()); err != nil {
		println("warning: failed to terminate postgres container:", err.Error())
	}

	os.Exit(code)
}

// newCleanStore คือ test helper (ทบทวน t.Helper() จาก Part 033) ที่ truncate ตาราง
// ก่อนคืน UserStore กลับไป เพื่อการันตีว่าทุก test เริ่มต้นด้วยตารางว่างเสมอ
// ไม่ว่า test ก่อนหน้าจะทิ้งข้อมูลอะไรไว้ก็ตาม — นี่คือรูปแบบ "ล้างข้อมูลระหว่าง test"
// ที่ใช้กันทั่วไปแทนการสร้าง/ลบ container ใหม่ทุก test (ซึ่งช้ากว่ามาก)
func newCleanStore(t *testing.T) *UserStore {
	t.Helper()
	s := NewUserStore(testDB)
	if err := s.TruncateAll(context.Background()); err != nil {
		t.Fatalf("truncate before test: %v", err)
	}
	return s
}

func TestUserStore_CreateAndGet(t *testing.T) {
	s := newCleanStore(t)
	ctx := context.Background()

	created, err := s.Create(ctx, "Alice", "alice@example.com")
	if err != nil {
		t.Fatalf("Create() error = %v", err)
	}
	if created.ID == 0 {
		t.Fatal("expected non-zero ID after Create")
	}

	got, err := s.GetByID(ctx, created.ID)
	if err != nil {
		t.Fatalf("GetByID() error = %v", err)
	}
	if got.Name != "Alice" || got.Email != "alice@example.com" {
		t.Errorf("GetByID() = %+v, want Name=Alice Email=alice@example.com", got)
	}
}

func TestUserStore_GetByID_NotFound(t *testing.T) {
	s := newCleanStore(t)

	_, err := s.GetByID(context.Background(), 999999)
	if !errors.Is(err, ErrUserNotFound) {
		t.Errorf("GetByID() error = %v, want ErrUserNotFound", err)
	}
}

func TestUserStore_Count(t *testing.T) {
	s := newCleanStore(t)
	ctx := context.Background()

	if n, err := s.Count(ctx); err != nil || n != 0 {
		t.Fatalf("Count() before insert = %d, %v; want 0, nil", n, err)
	}

	for _, name := range []string{"Alice", "Bob", "Carol"} {
		if _, err := s.Create(ctx, name, name+"@example.com"); err != nil {
			t.Fatalf("Create(%s) error = %v", name, err)
		}
	}

	if n, err := s.Count(ctx); err != nil || n != 3 {
		t.Fatalf("Count() after insert = %d, %v; want 3, nil", n, err)
	}
}

func TestUserStore_Create_DuplicateEmail(t *testing.T) {
	s := newCleanStore(t)
	ctx := context.Background()

	if _, err := s.Create(ctx, "Alice", "dup@example.com"); err != nil {
		t.Fatalf("first Create() error = %v", err)
	}

	// Postgres มี UNIQUE constraint บนคอลัมน์ email (ดู schemaSQL) ดังนั้นการ insert
	// อีเมลซ้ำต้องคืน error กลับมาเสมอ — นี่คือสิ่งที่ mock/fake ฐานข้อมูลจำลองได้ยาก
	// หรือจำลองได้ไม่ตรงกับพฤติกรรมจริงร้อยเปอร์เซ็นต์ ต้องพึ่ง Postgres ตัวจริงเท่านั้น
	if _, err := s.Create(ctx, "Alice2", "dup@example.com"); err == nil {
		t.Error("expected error on duplicate email, got nil")
	}
}
```

สังเกตกรณีทดสอบสุดท้าย `TestUserStore_Create_DuplicateEmail` เป็นตัวอย่างชัดเจนของ**สิ่งที่ integration test ทำได้แต่ unit test ที่ mock ฐานข้อมูลทำได้ยาก**: การพิสูจน์ว่า UNIQUE constraint ระดับฐานข้อมูลทำงานจริง

---

## 7. รันจริง: แยก Unit Test เร็วออกจาก Integration Test ช้า

รันแบบปกติ (ไม่มี `-tags=integration`) — ได้แค่ `TestValidateEmail` ที่รันเร็วมาก ไม่มีการแตะ Docker เลย:

```bash
go test -v ./...
```

```
=== RUN   TestValidateEmail
=== RUN   TestValidateEmail/อีเมลปกติ
=== RUN   TestValidateEmail/ไม่มี_@
=== RUN   TestValidateEmail/ไม่มี_domain
=== RUN   TestValidateEmail/ไม่มี_._ใน_domain
=== RUN   TestValidateEmail/@_อยู่ตำแหน่งแรก
=== RUN   TestValidateEmail/สตริงว่าง
--- PASS: TestValidateEmail (0.00s)
PASS
ok  	part080demo/store	0.002s
```

**0.002 วินาที** — ไม่มี container ใดถูกสร้างขึ้นเลย เพราะไฟล์ `store_integration_test.go` ทั้งไฟล์ไม่ถูกคอมไพล์เข้ามาด้วยซ้ำ (ไม่ใช่แค่ "ข้าม" การรัน — compiler ไม่เห็นไฟล์นี้เลย)

รันด้วย `-tags=integration` — คราวนี้ Docker ถูกเรียกใช้งานจริง:

```bash
go test -tags=integration -v ./...
```

```
2026/09/26 05:58:00 github.com/testcontainers/testcontainers-go - Connected to docker:
  Server Version: 29.3.1
  ...
2026/09/26 05:58:00 🐳 Creating container for image postgres:16-alpine
2026/09/26 05:58:00 ✅ Container created: 1ccd71af72b6
2026/09/26 05:58:00 🐳 Starting container: 1ccd71af72b6
2026/09/26 05:58:00 ⏳ Waiting for container id 1ccd71af72b6 image: postgres:16-alpine. Waiting for: all of: [log message "database system is ready to accept connections" (occurrence: 2), port 5432/tcp to be listening]
2026/09/26 05:58:02 🔔 Container is ready: 1ccd71af72b6
=== RUN   TestUserStore_CreateAndGet
--- PASS: TestUserStore_CreateAndGet (0.01s)
=== RUN   TestUserStore_GetByID_NotFound
--- PASS: TestUserStore_GetByID_NotFound (0.00s)
=== RUN   TestUserStore_Count
--- PASS: TestUserStore_Count (0.00s)
=== RUN   TestUserStore_Create_DuplicateEmail
--- PASS: TestUserStore_Create_DuplicateEmail (0.00s)
=== RUN   TestValidateEmail
... (unit test เดิมรันด้วยเช่นกัน เพราะไม่มี build tag จึงรวมอยู่เสมอ)
--- PASS: TestValidateEmail (0.00s)
PASS
2026/09/26 05:58:02 🐳 Terminating container: 1ccd71af72b6
2026/09/26 05:58:02 🚫 Container terminated: 1ccd71af72b6
ok  	part080demo/store	2.269s
```

ทั้งชุดทดสอบ (รวมเปิด Postgres จริง, รัน migration, รัน 4 integration test + 1 unit test, ปิด container) ใช้เวลา **2.269 วินาที** — เห็นความต่างชัดเจนกับ 0.002 วินาทีตอนไม่มี integration test เลย นี่คือเหตุผลที่การแยก build tag สำคัญมากในทางปฏิบัติ: ระหว่างเขียนโค้ดวันละหลายสิบครั้ง นักพัฒนาต้องการ feedback loop ที่เร็วที่สุดเท่าที่จะทำได้ ส่วน integration test ที่สมบูรณ์ค่อยรันตอน commit หรือใน CI

---

## 8. Setup/Teardown: Migration ก่อนรัน และ Truncate ระหว่าง Test

ตัวอย่างในหัวข้อ 6 สาธิตรูปแบบ setup/teardown มาตรฐานสำหรับ integration test กับฐานข้อมูล ซึ่งมี 2 ระดับที่ต้องแยกความแตกต่างให้ชัด:

### ระดับ Suite (ทำครั้งเดียว): Migration ใน `TestMain`

**Migration** (การสร้าง/ปรับ schema ของฐานข้อมูล) ควรทำ**ครั้งเดียว**ตอนเริ่มต้นทั้ง suite ไม่ใช่ทำซ้ำทุก test เพราะเป็นการดำเนินการที่มีต้นทุนสูงและไม่จำเป็นต้องทำซ้ำถ้า schema ไม่เปลี่ยน:

```go
if err := Migrate(ctx, db); err != nil {
    panic("migrate: " + err.Error())
}
```

ในโปรเจกต์จริงขนาดใหญ่ที่มี migration หลายไฟล์ (แบบที่เรียนใน **Part 075** เรื่อง GORM migrations) มักใช้ library เฉพาะทางอย่าง [`golang-migrate/migrate`](https://github.com/golang-migrate/migrate) รัน migration file ทั้งหมดตามลำดับเวอร์ชันแทนการเขียน `CREATE TABLE` ตรงๆ แบบในตัวอย่างนี้ (ซึ่งย่อให้ง่ายเพื่อการสาธิต)

### ระดับ Test (ทำทุกครั้ง): Truncate ข้อมูลก่อนแต่ละ Test

จุดที่มือใหม่มักพลาดคือ **ลืมล้างข้อมูลระหว่าง test แต่ละตัว** ทำให้ test ตัวหนึ่ง "เห็น" ข้อมูลที่ test ก่อนหน้าทิ้งไว้ กลายเป็นบั๊กที่แปลกประหลาดมาก: test ผ่านเมื่อรันตัวเดียว แต่ fail เมื่อรันพร้อมกับ test อื่น (หรือในลำดับที่ต่างกัน) ปัญหานี้เรียกว่า **test pollution** หรือ **test ที่ไม่ isolated จากกัน**

วิธีแก้ที่ใช้ในตัวอย่างนี้คือ helper `newCleanStore(t)` ที่ `TRUNCATE` ตารางก่อนคืน store กลับไปให้ test ใช้งาน:

```go
func newCleanStore(t *testing.T) *UserStore {
	t.Helper()
	s := NewUserStore(testDB)
	if err := s.TruncateAll(context.Background()); err != nil {
		t.Fatalf("truncate before test: %v", err)
	}
	return s
}
```

`TRUNCATE TABLE users RESTART IDENTITY` ไม่ได้แค่ลบข้อมูลทั้งหมด แต่ยัง**รีเซ็ต auto-increment counter** ของ `id` กลับไปเริ่มที่ 1 ด้วย ทำให้ทุก test คาดเดา `id` ที่จะได้ล่วงหน้าได้แน่นอน (deterministic) ไม่ต้องพึ่งค่า `id` ที่เปลี่ยนไปเรื่อยๆ ตามลำดับการรัน

**ทางเลือกอื่นนอกจาก TRUNCATE ที่ควรรู้จัก:**

| วิธี | ข้อดี | ข้อเสีย |
|---|---|---|
| **TRUNCATE ก่อนแต่ละ test** (ตัวอย่างนี้) | เขียนง่าย, เร็วพอสมควร | ถ้ารัน test แบบ parallel (`t.Parallel()` จาก Part 079) จะชนกันเพราะใช้ตารางเดียวกัน |
| **Transaction + Rollback** | เร็วมาก, ปลอดภัยกับ parallel เพราะแต่ละ test มี transaction ของตัวเอง | ต้องเขียนโค้ด production ให้รับ `*sql.Tx` แทน `*sql.DB` ได้ (เพิ่มความซับซ้อน) |
| **Database แยกต่อ test/worker** | isolate สมบูรณ์แบบ รองรับ parallel เต็มที่ | ช้าที่สุด (สร้าง/migrate database ใหม่ทุกครั้ง) |
| **Container ใหม่ต่อ test** | isolate สมบูรณ์แบบที่สุด | ช้ามาก ไม่เหมาะกับ suite ที่มี test จำนวนมาก |

สำหรับ integration test ส่วนใหญ่ที่ไม่ได้รันแบบ parallel กัน **TRUNCATE ก่อนแต่ละ test** อย่างที่สาธิตไปเป็นจุดสมดุลที่ดีที่สุดระหว่างความง่ายกับความเร็ว

---

## 9. ทางเลือกเมื่อไม่มี Docker: SQLite และฐานข้อมูล In-Memory

ถ้าเครื่องพัฒนาหรือ CI runner ไม่มี Docker ให้ใช้ (พบได้ในบาง sandbox ที่จำกัดสิทธิ์รันคอนเทนเนอร์ หรือเครื่องพัฒนาที่ยังไม่ได้ติดตั้ง Docker) มีทางเลือกที่ยังคงให้คุณค่าของ integration test ได้บางส่วนโดยไม่ต้องพึ่ง container เลย:

### ใช้ SQLite แทน Postgres/MySQL ในโหมด In-Memory

Driver อย่าง [`modernc.org/sqlite`](https://pkg.go.dev/modernc.org/sqlite) (pure-Go ไม่ต้องพึ่ง cgo) หรือ `mattn/go-sqlite3` เปิดฐานข้อมูล SQLite ในหน่วยความจำล้วนๆ ได้ทันทีด้วย DSN พิเศษ `:memory:` โดยไม่ต้องมี process ฐานข้อมูลแยกต่างหากเลย:

```go
db, err := sql.Open("sqlite", ":memory:")
```

ข้อดีคือเร็วมาก (ไม่มี network overhead) และไม่ต้องพึ่ง Docker แต่ข้อเสียที่ต้องตระหนักคือ **SQLite ไม่ใช่ Postgres** — syntax SQL บางส่วนต่างกัน (เช่น `RETURNING`, การจัดการ type, window function บางตัว) และพฤติกรรม concurrency ก็ต่างกันมาก (SQLite ล็อกทั้งไฟล์ตอนเขียน ต่างจาก Postgres ที่ล็อกระดับแถว) ดังนั้นถ้าโปรเจกต์ production ใช้ Postgres จริง integration test ที่รันกับ SQLite แทนจะ**ไม่ครอบคลุมพฤติกรรมเฉพาะของ Postgres**ได้ — เหมาะเป็นแค่ **safety net ระดับกลาง** (ดีกว่า mock ล้วนๆ แต่ยังไม่เท่าการทดสอบกับฐานข้อมูลจริงที่ใช้ใน production)

### หลักปฏิบัติที่แนะนำ

> **ถ้าเป็นไปได้ ให้ทดสอบกับฐานข้อมูลตัวเดียวกับที่ใช้ใน production เสมอ** — testcontainers-go (หัวข้อ 5-6) คือวิธีที่ทำสิ่งนี้ได้สะดวกที่สุดในปัจจุบัน เพราะทุกคนในทีมและทุก CI job ได้ Postgres เวอร์ชันเดียวกันเป๊ะโดยไม่ต้องติดตั้งอะไรถาวรในเครื่อง ใช้ SQLite in-memory เป็นทางเลือกสำรองเฉพาะเมื่อสภาพแวดล้อมไม่รองรับ Docker จริงๆ เท่านั้น

---

## 10. รัน Integration Test บน CI (ปูทางสู่ Part 098)

ใน CI pipeline (จะเรียนเจาะลึกด้วย GitHub Actions ใน **Part 098**) รูปแบบที่พบบ่อยที่สุดสำหรับโปรเจกต์ที่ใช้ `testcontainers-go` คือ:

1. **CI runner ต้องมี Docker ใช้งานได้** — GitHub Actions runner มาตรฐาน (`ubuntu-latest`) มี Docker daemon ติดตั้งมาให้แล้วโดย default จึงใช้ `testcontainers-go` ได้ทันทีโดยไม่ต้องตั้งค่าเพิ่ม
2. **แยก job หรือ step สำหรับ unit test กับ integration test** เพื่อให้เห็นผลแยกกันชัดเจนใน CI report และเพื่อให้ unit test ที่เร็วรายงานผลได้ก่อน (fail-fast):

```yaml
# ตัวอย่างคร่าวๆ (รายละเอียดเต็มรูปแบบอยู่ใน Part 098)
jobs:
  unit-test:
    steps:
      - run: go test ./...

  integration-test:
    needs: unit-test
    steps:
      - run: go test -tags=integration ./...
```

3. **ตั้ง timeout ให้เหมาะสม** — integration test ที่ดึง Docker image ครั้งแรกอาจใช้เวลานานกว่าปกติ ควรตั้ง timeout ของ CI job ให้กว้างพอ (เช่น 10-15 นาที) และพิจารณา cache Docker image ระหว่าง run ถ้า CI provider รองรับ
4. **ทำความสะอาด container เสมอแม้ test fail** — จุดนี้ `t.Cleanup()`/`TestMain` ที่เรียก `pgContainer.Terminate()` เสมอ (ตามที่สาธิตในหัวข้อ 6) สำคัญมาก ไม่งั้น CI runner อาจสะสม container ค้างจนเต็มพื้นที่ดิสก์ในระยะยาว (testcontainers-go มีกลไกเสริมชื่อ **Ryuk** ที่คอยลบ container กำพร้าทิ้งอัตโนมัติแม้ process หลักจะถูกฆ่ากะทันหันด้วย)

รายละเอียดเรื่อง workflow YAML แบบเต็ม, matrix testing หลายเวอร์ชัน Go, และการ cache dependency จะอยู่ใน **Part 098: CI/CD Pipeline ด้วย GitHub Actions**

---

## สรุปสิ่งที่ได้เรียนในบทนี้

- **Test Pyramid** แบ่งการทดสอบเป็น 3 ชั้น: unit (เยอะ, เร็ว), integration (ปานกลาง), end-to-end (น้อย, ช้า) — ยิ่งอยู่ชั้นบนควรยิ่งมีจำนวนน้อยเพื่อรักษาความเร็วและความเสถียรของ test suite โดยรวม
- **Integration test** คือการปล่อยให้โค้ดคุยกับ dependency ภายนอกจริง (ฐานข้อมูล, message queue) แทนการ mock เพื่อจับบั๊กที่ mock จำลองไม่ได้ เช่น SQL ผิด หรือ constraint ไม่ทำงานตามคาด
- **`//go:build integration`** เป็น build tag ที่แยกไฟล์ integration test ออกจาก unit test ทำให้ `go test ./...` ปกติเร็วมาก และต้องระบุ `-tags=integration` เพื่อรันไฟล์เหล่านี้เพิ่ม
- **`testcontainers-go`** สั่งให้ Docker รัน Postgres (หรือฐานข้อมูล/service อื่น) ขึ้นมาชั่วคราวจากภายในโค้ด Go เอง — ตัวอย่างในบทนี้ **รันจริงและยืนยันแล้ว** ด้วย Postgres 16 ในคอนเทนเนอร์จริงบนเครื่องที่ใช้เขียนบทเรียนนี้
- **Setup 2 ระดับ**: migration ทำครั้งเดียวใน `TestMain` ระดับ suite, TRUNCATE ตารางก่อนแต่ละ test เพื่อป้องกัน test pollution ระหว่าง test ด้วยกัน
- เมื่อไม่มี Docker ให้ใช้ SQLite in-memory เป็นทางเลือกสำรอง แต่ต้องตระหนักว่าพฤติกรรมต่างจากฐานข้อมูล production จริง (เช่น Postgres) ในหลายจุด
- Integration test บน CI ต้องมี Docker พร้อมใช้ในสภาพแวดล้อม runner, แยก job จาก unit test, ตั้ง timeout ให้เหมาะสม และการันตีว่า container ถูกลบทิ้งเสมอแม้ test fail (รายละเอียดเต็มใน Part 098)

## แบบฝึกหัดท้ายบท

1. เขียน package `store` ตามตัวอย่างในบทนี้ด้วยตัวเอง (หรือคัดลอกจากบทเรียน) แล้วรัน `go test ./...` เทียบกับ `go test -tags=integration ./...` สังเกตความต่างของเวลาที่ใช้และอธิบายด้วยคำพูดตัวเองว่าทำไมถึงต่างกัน
2. เพิ่ม method `UpdateEmail(ctx, id, newEmail string) error` ใน `UserStore` แล้วเขียน integration test ทดสอบทั้งกรณีอัปเดตสำเร็จ และกรณีอัปเดตเป็นอีเมลที่ซ้ำกับ user อื่น (ควร error เพราะ UNIQUE constraint)
3. ทดลองลบการเรียก `s.TruncateAll(...)` ออกจาก `newCleanStore` แล้วรัน integration test ทั้งหมดซ้ำๆ กันหลายรอบติดต่อกัน (`go test -tags=integration -count=3 ./...`) สังเกตว่า test ไหนเริ่ม fail และอธิบายว่าทำไม (นี่คือตัวอย่างจริงของ test pollution)
4. ค้นคว้าเพิ่มเติมเกี่ยวกับ module `testcontainers-go/modules/mysql` หรือ `testcontainers-go/modules/redis` แล้วลองเขียนโครงร่าง integration test สำหรับ Redis (ทบทวนจาก **Part 077**) ที่ทดสอบว่า `SET` แล้ว `GET` กลับมาได้ค่าเดิม
5. เขียน build tag ที่ซับซ้อนขึ้น เช่น `//go:build integration && linux` แล้วทดลองรันบนเครื่องของตัวเองเพื่อดูว่าเงื่อนไข `&&` ทำงานตามที่คาดหวังหรือไม่
6. ลองเปลี่ยนวิธี teardown จาก TRUNCATE เป็น "สร้าง `*sql.Tx` ตอนเริ่ม test แล้ว rollback ตอนจบ" (ใช้ `db.BeginTx` และ `t.Cleanup(func() { tx.Rollback() })`) แล้วเปรียบเทียบว่าต้องแก้ `UserStore` อย่างไรให้รับทั้ง `*sql.DB` และ `*sql.Tx` ได้ (คำใบ้: ทั้งสองชนิดสอดคล้องกับ interface เดียวกันที่มี method `QueryRowContext`/`ExecContext`)

---

**ต่อไป**: [Part 081 — `httptest`: ทดสอบ HTTP Handler](./081-httptest.md)
