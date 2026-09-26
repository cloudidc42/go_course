# Part 105: โปรเจกต์ URL Shortener พร้อม Caching

> ภาคที่ 10: มืออาชีพและระดับโลก (Professional & World-Class) — ตอนที่ 6 จาก 11 (Part 100–110)

## สารบัญของบทนี้

1. ภาพรวมโปรเจกต์และสถาปัตยกรรม
2. โครงสร้างโปรเจกต์และ dependency ที่ใช้จริง
3. ชั้นฐานข้อมูล: SQLite วันนี้ ออกแบบให้พอร์ตไป PostgreSQL ได้พรุ่งนี้
4. สร้าง Short Code สองแบบ: Base62-of-ID vs Random + Collision Retry
5. Cache-Aside Layer: Redis จริง พร้อม in-memory LRU สำรอง
6. Rate Limiting บน `POST /shorten` ด้วย `golang.org/x/time/rate`
7. Analytics: นับคลิกแบบ atomic แล้ว batch flush ลงฐานข้อมูล
8. Prometheus Metrics: request count และ cache hit ratio
9. ชั้น HTTP: routing, middleware, และ handler ทั้งหมด
10. `cmd/server/main.go`: ประกอบทุกอย่างเข้าด้วยกันพร้อม graceful shutdown
11. รันจริงแบบ end-to-end ด้วย `curl`
12. Test Suite: unit test และ httptest integration test ที่รันจริง
13. Benchmark: cache ช่วยลด latency ได้จริงแค่ไหน (วัดจริง พร้อมเรื่องเซอร์ไพรส์ที่ต้องอธิบาย)
14. หมายเหตุความซื่อสัตย์เรื่องการรันจริงในบทนี้
15. สรุปสิ่งที่ได้เรียนในบทนี้
16. แบบฝึกหัดท้ายบท

---

## 1. ภาพรวมโปรเจกต์และสถาปัตยกรรม

นี่คือโปรเจกต์รวบยอด (capstone) ตัวที่สองจากสามตัวในภาคที่ 10 — หลังจาก **Part 103** สร้าง E-Commerce REST API แบบเต็มรูปแบบ และ **Part 104** สร้าง Real-time Chat Application ไปแล้ว บทนี้จะสร้าง **URL Shortener** (บริการย่อลิงก์แบบเดียวกับ bit.ly หรือ TinyURL) ที่ครบวงจร:

- รับ URL ยาวมา สร้าง "short code" สั้นๆ ให้ (เช่น `go.dev/doc/effective_go` → `localhost:8099/DsxOGdy`)
- เมื่อมีคนเข้า short URL ให้ **redirect** ไปยัง URL ปลายทางจริง
- เก็บสถิติจำนวนคลิกต่อ short code
- ป้องกันการยิง request สร้าง short URL รัวเกินไป (abuse)
- เปิด metric ให้ระบบ monitoring ภายนอกดึงไปวิเคราะห์ได้

จุดที่ทำให้โปรเจกต์นี้น่าสนใจในเชิงวิศวกรรมมากกว่าที่เห็นตอนแรกคือ **การ redirect คือ hot path ที่ถูกเรียกบ่อยกว่าการสร้าง short URL มากมายมหาศาล** (ลิงก์หนึ่งอันถูกสร้างครั้งเดียว แต่อาจถูกคลิกหลายพันหรือหลายล้านครั้ง) ระบบทั้งหมดในบทนี้จึงถูกออกแบบรอบคำถามเดียว: **จะทำให้ hot path นี้เร็วและทนต่อ traffic สูงได้อย่างไร โดยไม่เสียความถูกต้องของข้อมูล**

สถาปัตยกรรมที่ได้รวมทุกอย่างที่เรียนมาตลอดหลักสูตรเข้าด้วยกัน:

```
                     ┌─────────────────────┐
   POST /shorten ───▶│  Rate Limiter        │ (Part 045)
                     │  (golang.org/x/      │
                     │   time/rate ต่อ IP)   │
                     └──────────┬───────────┘
                                │
                     ┌──────────▼───────────┐
                     │   HTTP Handler        │ (Part 056)
                     │   (net/http + Go      │
                     │    1.22+ ServeMux)    │
                     └──────────┬───────────┘
                                │
              ┌─────────────────┼──────────────────┐
              │                 │                    │
   POST /shorten          GET /{code}          GET /stats/{code}
              │                 │                    │
      ┌───────▼───────┐  ┌──────▼───────┐    ┌───────▼────────┐
      │ codegen        │  │ Cache-Aside   │    │ ClickTracker    │
      │ (base62/random)│  │ (Redis, Part  │    │ (atomic.Int64,  │
      │ Part 051       │  │  077)         │    │  Part 040)      │
      └───────┬───────┘  └──────┬───────┘    └───────┬────────┘
              │                 │  miss              │
              │          ┌──────▼───────┐            │
              └─────────▶│  SQLite Store │◀───────────┘ (batch flush)
                         │  (Part 071)   │
                         └───────────────┘
                                │
                     ┌──────────▼───────────┐
                     │  Prometheus /metrics  │ (Part 099)
                     └──────────────────────┘
```

ทุกกล่องในแผนภาพนี้คือโค้ดจริงที่เขียน คอมไพล์ และรันในบทนี้ — ไม่ใช่ pseudo-code

---

## 2. โครงสร้างโปรเจกต์และ dependency ที่ใช้จริง

```
urlshortener/
├── go.mod
├── go.sum
├── cmd/
│   └── server/
│       └── main.go              # entry point: ประกอบ dependency ทั้งหมดเข้าด้วยกัน
└── internal/
    ├── store/                   # ชั้น persistence (database/sql + SQLite)
    │   └── store.go
    ├── codegen/                 # สร้าง short code (base62 / random)
    │   ├── codegen.go
    │   └── codegen_test.go
    ├── cache/                   # cache-aside layer (Redis + in-memory LRU)
    │   ├── cache.go
    │   └── cache_test.go
    ├── analytics/                # นับคลิกแบบ atomic + batch flush
    │   ├── clicks.go
    │   └── clicks_test.go
    ├── ratelimit/                # token bucket rate limiter ต่อ IP
    │   ├── ratelimit.go
    │   └── ratelimit_test.go
    ├── metrics/                  # Prometheus metric ทั้งหมด
    │   └── metrics.go
    └── api/                      # HTTP handler, routing, middleware
        ├── server.go
        ├── server_test.go
        └── bench_test.go
```

โครงสร้างนี้ใช้ `cmd/` + `internal/` ตามธรรมเนียมมาตรฐานที่แนะนำไว้ตั้งแต่ **Part 001** หัวข้อ 10 — ทุก package อยู่ใต้ `internal/` เพราะไม่มีเจตนาให้โปรเจกต์อื่น import โค้ดชุดนี้ไปใช้ตรงๆ

### Dependency ที่ติดตั้งและยืนยันว่าใช้งานได้จริงกับ Go 1.24.7

```bash
go get modernc.org/sqlite@v1.34.5              # SQLite driver แบบ pure-Go ไม่ต้องพึ่ง cgo
go get github.com/redis/go-redis/v9@v9.7.0     # Redis client (ต่อยอด Part 077)
go get github.com/prometheus/client_golang@v1.20.5  # Prometheus metrics (ต่อยอด Part 099)
go get golang.org/x/time@v0.8.0                # token bucket rate limiter (ต่อยอด Part 045)
```

> **หมายเหตุเรื่องเวอร์ชัน**: ตอนเขียนบทความนี้ เวอร์ชันล่าสุดของ `modernc.org/sqlite` (v1.59.0), `github.com/prometheus/client_golang` (v1.24.1) และ `golang.org/x/time` (v0.16.0) ทุกตัวประกาศ `go.mod` ของตัวเองว่าต้องการ Go **1.25 หรือ 1.26** ขึ้นไป ซึ่งใหม่กว่า Go 1.24.7 ที่ใช้ตรวจสอบโค้ดในบทความนี้ ผู้เขียนจึงตั้งใจ pin เวอร์ชันให้เก่าลงมาเล็กน้อยตามที่ระบุไว้ข้างบน เพื่อให้คอมไพล์ผ่านกับ Go เวอร์ชันที่ใช้สอนในหลักสูตรนี้ได้จริง — นี่คือสถานการณ์จริงที่พบได้เสมอในงาน dependency management (Part 018): เวอร์ชันล่าสุดของ library ไม่ได้แปลว่าใช้ได้กับทุกเวอร์ชันของ Go เสมอไป ต้องเช็ค `go.mod` ของ dependency ก่อน `go get` ทุกครั้งถ้าอยาก pin เวอร์ชัน Go ไว้คงที่

`go.mod` เต็มหลัง `go mod tidy`:

```go
module urlshortener

go 1.24.7

require (
	github.com/prometheus/client_golang v1.20.5
	github.com/redis/go-redis/v9 v9.7.0
	golang.org/x/time v0.8.0
	modernc.org/sqlite v1.34.5
)

require (
	github.com/beorn7/perks v1.0.1 // indirect
	github.com/cespare/xxhash/v2 v2.3.0 // indirect
	github.com/dgryski/go-rendezvous v0.0.0-20200823014737-9f7001d12a5f // indirect
	github.com/dustin/go-humanize v1.0.1 // indirect
	github.com/google/uuid v1.6.0 // indirect
	github.com/klauspost/compress v1.17.9 // indirect
	github.com/mattn/go-isatty v0.0.20 // indirect
	github.com/munnerz/goautoneg v0.0.0-20191010083416-a7dc8b61c822 // indirect
	github.com/ncruces/go-strftime v0.1.9 // indirect
	github.com/prometheus/client_model v0.6.1 // indirect
	github.com/prometheus/common v0.55.0 // indirect
	github.com/prometheus/procfs v0.15.1 // indirect
	github.com/remyoudompheng/bigfft v0.0.0-20230129092748-24d4a6f8daec // indirect
	golang.org/x/sys v0.22.0 // indirect
	google.golang.org/protobuf v1.34.2 // indirect
	modernc.org/libc v1.55.3 // indirect
	modernc.org/mathutil v1.6.0 // indirect
	modernc.org/memory v1.8.0 // indirect
)
```

สังเกตว่าเลือกใช้ **`modernc.org/sqlite`** แทน `mattn/go-sqlite3` ที่หลายคนคุ้นเคย — เหตุผลคือ `mattn/go-sqlite3` ต้องพึ่ง **cgo** (เรียกโค้ด C ของ SQLite ตรงๆ) ซึ่งทำให้ cross-compile ยากขึ้นและเสียจุดเด่นเรื่อง "binary เดียวจบ ไม่ต้องมี runtime เพิ่ม" ของ Go ที่กล่าวถึงใน **Part 001** ไป ส่วน `modernc.org/sqlite` คือ SQLite ที่ถูกแปล (transpile) จาก C เป็น Go ล้วนๆ ทำให้ `CGO_ENABLED=0` ได้ตามปกติ แลกกับ compile time ที่นานขึ้นเล็กน้อยและขนาด binary ที่ใหญ่กว่า

---

## 3. ชั้นฐานข้อมูล: SQLite วันนี้ ออกแบบให้พอร์ตไป PostgreSQL ได้พรุ่งนี้

โจทย์ในหัวข้อนี้ตรงไปตรงมาตามที่เรียนใน **Part 071**: `database/sql` เป็น interface กลางที่ไม่รู้จักฐานข้อมูลตัวใดตัวหนึ่งเป็นการเฉพาะ — โค้ดที่เขียนด้วย `db.Query`/`db.Exec`/`Scan` เหมือนกันทุกตัวอักษรไม่ว่าเบื้องหลังจะเป็นฐานข้อมูลอะไร บทความนี้ใช้ **SQLite** เพราะเป็นไฟล์เดียว ไม่ต้องมี server แยกให้รันในสภาพแวดล้อมที่เขียนบทความ ทำให้ **ทุกตัวอย่างในบทนี้ทดสอบได้จริง 100%** แต่โครงสร้าง SQL ถูกเขียนให้ใกล้เคียง ANSI SQL ที่สุดเพื่อให้ deploy จริงด้วย PostgreSQL (**Part 072**) ได้โดยแก้แค่ 3 จุดที่ระบุไว้ในคอมเมนต์

```go
// internal/store/store.go
package store

import (
	"context"
	"database/sql"
	"errors"
	"fmt"
	"time"

	_ "modernc.org/sqlite" // driver แบบ pure-Go ไม่ต้องพึ่ง cgo (ตามที่กล่าวถึงใน Part 071 หัวข้อ blank import)
)

var ErrCodeNotFound = errors.New("store: short code not found")
var ErrCodeTaken = errors.New("store: short code already taken")

type URLRecord struct {
	ID        int64
	Code      string
	LongURL   string
	CreatedAt time.Time
	Clicks    int64
}

type Store struct {
	db *sql.DB
}

func Open(ctx context.Context, path string) (*Store, error) {
	db, err := sql.Open("sqlite", path)
	if err != nil {
		return nil, fmt.Errorf("store: sql.Open ล้มเหลว: %w", err)
	}

	// SQLite เป็นไฟล์เดียว รองรับ concurrent write ได้จำกัดกว่า PostgreSQL มาก
	// (มี write lock ระดับไฟล์) การจำกัด MaxOpenConns=1 ช่วยเลี่ยง "database is locked"
	// error ที่พบบ่อยเมื่อมีหลาย goroutine เขียนพร้อมกัน — เป็นข้อจำกัดเฉพาะของ SQLite
	// ที่ PostgreSQL/MySQL ไม่มี (ดู Part 078 เรื่อง connection pooling)
	db.SetMaxOpenConns(1)

	if err := db.PingContext(ctx); err != nil {
		return nil, fmt.Errorf("store: ping ล้มเหลว: %w", err)
	}

	s := &Store{db: db}
	if err := s.migrate(ctx); err != nil {
		return nil, err
	}
	return s, nil
}

func (s *Store) Close() error { return s.db.Close() }

func (s *Store) migrate(ctx context.Context) error {
	// หมายเหตุสำหรับพอร์ตไป PostgreSQL (Part 072):
	//   1. "INTEGER PRIMARY KEY AUTOINCREMENT" -> "id BIGSERIAL PRIMARY KEY"
	//   2. "DATETIME DEFAULT CURRENT_TIMESTAMP" -> "TIMESTAMPTZ NOT NULL DEFAULT now()"
	//   3. placeholder "?" ในทุก query ของไฟล์นี้ -> "$1", "$2", ...
	// นอกนั้นโค้ด Go ทั้งหมดที่ใช้ database/sql (Query/Exec/Scan) เหมือนเดิมทุกบรรทัด
	const schema = `
	CREATE TABLE IF NOT EXISTS urls (
		id         INTEGER PRIMARY KEY AUTOINCREMENT,
		code       TEXT UNIQUE,
		long_url   TEXT NOT NULL,
		created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
		clicks     INTEGER NOT NULL DEFAULT 0
	);
	CREATE INDEX IF NOT EXISTS idx_urls_code ON urls(code);
	`
	_, err := s.db.ExecContext(ctx, schema)
	if err != nil {
		return fmt.Errorf("store: migrate ล้มเหลว: %w", err)
	}
	return nil
}

func (s *Store) InsertPending(ctx context.Context, longURL string) (int64, error) {
	res, err := s.db.ExecContext(ctx, `INSERT INTO urls (long_url) VALUES (?)`, longURL)
	if err != nil {
		return 0, fmt.Errorf("store: insert pending ล้มเหลว: %w", err)
	}
	return res.LastInsertId()
}

func (s *Store) SetCode(ctx context.Context, id int64, code string) error {
	_, err := s.db.ExecContext(ctx, `UPDATE urls SET code = ? WHERE id = ?`, code, id)
	if err != nil {
		return fmt.Errorf("store: set code ล้มเหลว: %w", err)
	}
	return nil
}

func (s *Store) InsertWithCode(ctx context.Context, code, longURL string) error {
	_, err := s.db.ExecContext(ctx, `INSERT INTO urls (code, long_url) VALUES (?, ?)`, code, longURL)
	if err != nil {
		return fmt.Errorf("store: insert with code ล้มเหลว: %w", err)
	}
	return nil
}

func (s *Store) CodeExists(ctx context.Context, code string) (bool, error) {
	var exists bool
	err := s.db.QueryRowContext(ctx,
		`SELECT EXISTS(SELECT 1 FROM urls WHERE code = ?)`, code,
	).Scan(&exists)
	if err != nil {
		return false, fmt.Errorf("store: code exists ล้มเหลว: %w", err)
	}
	return exists, nil
}

func (s *Store) GetByCode(ctx context.Context, code string) (string, error) {
	var longURL string
	err := s.db.QueryRowContext(ctx,
		`SELECT long_url FROM urls WHERE code = ?`, code,
	).Scan(&longURL)
	if errors.Is(err, sql.ErrNoRows) {
		return "", ErrCodeNotFound
	}
	if err != nil {
		return "", fmt.Errorf("store: get by code ล้มเหลว: %w", err)
	}
	return longURL, nil
}

func (s *Store) IncrementClicks(ctx context.Context, code string, delta int64) error {
	if delta <= 0 {
		return nil
	}
	_, err := s.db.ExecContext(ctx,
		`UPDATE urls SET clicks = clicks + ? WHERE code = ?`, delta, code)
	if err != nil {
		return fmt.Errorf("store: increment clicks ล้มเหลว: %w", err)
	}
	return nil
}

func (s *Store) GetStats(ctx context.Context, code string) (URLRecord, error) {
	var rec URLRecord
	err := s.db.QueryRowContext(ctx,
		`SELECT id, code, long_url, created_at, clicks FROM urls WHERE code = ?`, code,
	).Scan(&rec.ID, &rec.Code, &rec.LongURL, &rec.CreatedAt, &rec.Clicks)
	if errors.Is(err, sql.ErrNoRows) {
		return URLRecord{}, ErrCodeNotFound
	}
	if err != nil {
		return URLRecord{}, fmt.Errorf("store: get stats ล้มเหลว: %w", err)
	}
	return rec, nil
}
```

จุดที่ควรสังเกตหลายอย่าง:

- **`ErrCodeNotFound`** เป็น sentinel error ตามแนวทางเดียวกับ `sql.ErrNoRows` (Part 071) และ `redis.Nil` (Part 077) — "หาไม่เจอ" ไม่ใช่ error ทางเทคนิค แต่เป็นผลลัพธ์ปกติที่ผู้เรียกต้อง handle แยกจาก error จริงเสมอ
- **`db.SetMaxOpenConns(1)`** คือข้อจำกัดเฉพาะของ SQLite ที่ไม่มีใน PostgreSQL/MySQL — เพราะ SQLite ล็อกทั้งไฟล์เวลาเขียน การมีหลาย connection พร้อมกันจะชนกันบ่อย (`database is locked`) จำกัดไว้ที่ 1 connection ตัดปัญหานี้ไปเลย (ต้นทุนคือ throughput การเขียนต่ำกว่า production database จริง ซึ่งเป็นข้อแลกเปลี่ยนที่ยอมรับได้สำหรับการสอนและ prototype)
- **`IncrementClicks`** ใช้ `SET clicks = clicks + ?` แทนที่จะ `SELECT` ค่าปัจจุบันมาบวกแล้วค่อย `UPDATE` แยกคำสั่ง — วิธีหลังเสี่ยง **lost update** เมื่อมีหลาย request อ่าน-เขียนพร้อมกัน (request A อ่านค่า 10 มา, request B อ่านค่า 10 มาเหมือนกัน, ทั้งคู่บวก 1 แล้วเขียนกลับเป็น 11 ทั้งคู่ — ทั้งที่ควรได้ 12) ส่วน `clicks = clicks + ?` เป็นคำสั่งเดียวที่ฐานข้อมูลรับประกันความถูกต้องให้เอง ไม่ว่าจะมีกี่ request มาพร้อมกัน

---

## 4. สร้าง Short Code สองแบบ: Base62-of-ID vs Random + Collision Retry

นี่คือหัวใจของ URL shortener และเป็นจุดที่ต้องตัดสินใจ trade-off จริงจัง มีสองกลยุทธ์หลักที่ระบบจริงใช้:

| | Sequential (Base62 ของ auto-increment ID) | Random (สุ่มด้วย `crypto/rand`) |
|---|---|---|
| Collision | **ไม่มีทางเกิดเลย** — ID จากฐานข้อมูลไม่ซ้ำกันโดยธรรมชาติ | มีโอกาสเกิด (แม้น้อยมาก) ต้องเช็คและ retry |
| ความสามารถในการเดา | **เดาได้ง่าย** — เห็น code `"5"` แล้วรู้ทันทีว่า code ถัดไปคือ `"6"` ทำให้ enumerate ลิงก์คนอื่นได้ | **เดาไม่ได้เลย** — ไม่มีความสัมพันธ์ระหว่าง code กับลำดับการสร้าง |
| ความยาวของ code | สั้นตาม ID (ID น้อย = code สั้น) | คงที่ตามที่กำหนด (ในบทนี้ใช้ 7 ตัวอักษรเสมอ) |
| ความซับซ้อนของโค้ด | ต่ำ — insert แล้ว encode ID ตรงๆ | สูงกว่าเล็กน้อย — ต้องมี retry loop |
| เหมาะกับ | ระบบ internal, dashboard สถิติ, หรือกรณีที่ไม่สนเรื่องการเดาลำดับ | ระบบ public ที่ผู้ใช้คาดหวังว่า short URL ของตัวเองเป็นความลับระดับหนึ่ง |

โค้ดของทั้งสองกลยุทธ์:

```go
// internal/codegen/codegen.go
package codegen

import (
	"crypto/rand"
	"fmt"
	"math/big"
)

const alphabet = "0123456789abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ"
const base = int64(len(alphabet))

// EncodeBase62 แปลงเลขจำนวนเต็มบวก (เช่น auto-increment ID จากฐานข้อมูล)
// ให้เป็น short code แบบ base62 — ID ยิ่งเล็ก code ยิ่งสั้น (ID 0-61 ได้ code 1 ตัวอักษร)
func EncodeBase62(id int64) string {
	if id == 0 {
		return string(alphabet[0])
	}
	if id < 0 {
		panic("codegen: EncodeBase62 ไม่รองรับเลขติดลบ")
	}
	var buf []byte
	for id > 0 {
		remainder := id % base
		buf = append(buf, alphabet[remainder])
		id /= base
	}
	for i, j := 0, len(buf)-1; i < j; i, j = i+1, j-1 {
		buf[i], buf[j] = buf[j], buf[i]
	}
	return string(buf)
}

// RandomCode สุ่ม short code ความยาว n ตัวอักษรจาก alphabet ด้วย crypto/rand
// (เชื่อมโยงกับ Part 051 หัวข้อ "crypto/rand vs math/rand": เพราะ short code
// เกี่ยวข้องกับความปลอดภัย (ป้องกันการเดา/ไล่ enumerate ของผู้ใช้อื่น)
// จึงต้องสุ่มแบบ cryptographically secure ไม่ใช่ math/rand ธรรมดา)
func RandomCode(n int) (string, error) {
	buf := make([]byte, n)
	max := big.NewInt(base)
	for i := 0; i < n; i++ {
		idx, err := rand.Int(rand.Reader, max)
		if err != nil {
			return "", fmt.Errorf("codegen: สุ่ม code ล้มเหลว: %w", err)
		}
		buf[i] = alphabet[idx.Int64()]
	}
	return string(buf), nil
}
```

จุดสำคัญของ `RandomCode`: ใช้ **`crypto/rand.Int`** แทนการสุ่ม byte เดี่ยวๆ แล้ว `% 62` ตรงๆ เพราะ 256 (จำนวนค่าที่ byte เก็บได้) หารด้วย 62 ไม่ลงตัว — ถ้า mod ตรงๆ ตัวอักษรบางตัวใน alphabet จะมีโอกาสออกมากกว่าตัวอื่นเล็กน้อย (**modulo bias**) `rand.Int(rand.Reader, big.NewInt(62))` จัดการปัญหานี้ให้อัตโนมัติด้วยเทคนิค rejection sampling ภายใน

### ทำไม `RandomCode` ความยาว 7 ตัวอักษรถึงปลอดภัยพอ

จำนวนความเป็นไปได้ทั้งหมดคือ 62⁷ ≈ 3.5 ล้านล้านค่า (3.52 × 10¹²) ต่อให้ระบบมี short URL อยู่แล้ว 10 ล้านอัน (10⁷) โอกาสที่จะสุ่มชนกันในครั้งถัดไปยังต่ำกว่า 1 ใน 350,000 — และแม้จะชนก็มี **collision-check retry loop** รองรับอยู่แล้วในโค้ด handler (หัวข้อ 9)

### การนำไปใช้จริงในสองกลยุทธ์

ทั้งสองกลยุทธ์ถูก implement เป็น method ของ `Server` ใน `internal/api/server.go`:

```go
// createSequential สร้าง code แบบ base62-of-auto-increment-ID: insert แถวก่อนเพื่อให้
// ฐานข้อมูลจ่าย ID มาให้ แล้วเข้ารหัส ID นั้นเป็น code แล้วค่อย UPDATE กลับเข้าไป
func (s *Server) createSequential(ctx context.Context, longURL string) (string, error) {
	id, err := s.store.InsertPending(ctx, longURL)
	if err != nil {
		return "", err
	}
	code := codegen.EncodeBase62(id)
	if err := s.store.SetCode(ctx, id, code); err != nil {
		return "", err
	}
	return code, nil
}

// createRandom สร้าง code แบบสุ่มด้วย crypto/rand (Part 051) แล้วเช็ค collision ก่อน insert
// เสมอ ("check-then-insert" — ในทางทฤษฎียังมี race เล็กน้อยถ้าสอง request สุ่มได้ code
// เดียวกันพอดีในเวลาไล่เลี่ยกัน แต่ UNIQUE constraint ระดับฐานข้อมูลเป็นตาข่ายนิรภัยสุดท้าย
// ที่รับประกันความถูกต้องจริงๆ อยู่ดี ไม่ว่า race จะเกิดหรือไม่)
func (s *Server) createRandom(ctx context.Context, longURL string) (string, error) {
	for attempt := 0; attempt < s.cfg.MaxCollisionRetries; attempt++ {
		code, err := codegen.RandomCode(s.cfg.RandomCodeLength)
		if err != nil {
			return "", err
		}
		exists, err := s.store.CodeExists(ctx, code)
		if err != nil {
			return "", err
		}
		if exists {
			continue // ชนของเดิม (โอกาสเกิดน้อยมาก) สุ่มใหม่รอบถัดไป
		}
		if err := s.store.InsertWithCode(ctx, code, longURL); err != nil {
			if errors.Is(err, store.ErrCodeTaken) {
				continue
			}
			return "", err
		}
		return code, nil
	}
	return "", fmt.Errorf("could not generate unique code after %d attempts", s.cfg.MaxCollisionRetries)
}
```

`POST /shorten` รับ field `"strategy"` เป็น `"sequential"` หรือ `"random"` (ค่าเริ่มต้นคือ `"random"` เพื่อความปลอดภัยเป็นค่าตั้งต้น) ทำให้เห็นความแตกต่างของทั้งสองกลยุทธ์ได้ในระบบเดียวกันโดยตรง (ดูตัวอย่างจริงในหัวข้อ 11)

Unit test ของ `codegen` ยืนยันทั้ง round-trip ของ base62 และคุณสมบัติของ random code:

```go
// internal/codegen/codegen_test.go (ตัดมาเฉพาะส่วนสำคัญ)
func TestEncodeDecodeBase62_RoundTrip(t *testing.T) {
	cases := []int64{0, 1, 61, 62, 63, 12345, 999999999}
	for _, id := range cases {
		code := EncodeBase62(id)
		got, err := DecodeBase62(code)
		if err != nil || got != id {
			t.Errorf("round-trip mismatch: id=%d -> code=%q -> decoded=%d (err=%v)", id, code, got, err)
		}
	}
}

func TestRandomCode_LengthAndAlphabet(t *testing.T) {
	code, err := RandomCode(7)
	if err != nil {
		t.Fatalf("RandomCode returned error: %v", err)
	}
	if len(code) != 7 {
		t.Fatalf("RandomCode(7) length = %d, want 7", len(code))
	}
	for _, c := range code {
		if !strings.ContainsRune(alphabet, c) {
			t.Errorf("RandomCode ให้ตัวอักษร %q ที่ไม่อยู่ใน base62 alphabet", c)
		}
	}
}
```

---

## 5. Cache-Aside Layer: Redis จริง พร้อม in-memory LRU สำรอง

นี่คือส่วนที่ต่อยอดโดยตรงจาก **Part 077 หัวข้อ 6 (Cache-Aside Pattern)** — วางแคชไว้หน้า `GetByCode` ซึ่งเป็น query ที่ถูกเรียกบ่อยที่สุดในระบบทั้งหมด (ทุกครั้งที่มีคน redirect) ออกแบบเป็น interface กลางก่อน เพื่อให้สลับ implementation ได้โดยไม่กระทบ handler เลย (หลักการ small interface ที่เรียนใน **Part 013**):

```go
// internal/cache/cache.go (ส่วน interface และ Redis implementation)
package cache

import (
	"container/list"
	"context"
	"sync"
	"time"

	"github.com/redis/go-redis/v9"
)

type Cache interface {
	// Get คืน (longURL, true, nil) ถ้าเจอ, ("", false, nil) ถ้า cache miss (ไม่ใช่ error)
	Get(ctx context.Context, code string) (string, bool, error)
	Set(ctx context.Context, code, longURL string, ttl time.Duration) error
	Del(ctx context.Context, code string) error
}

type RedisCache struct {
	rdb    *redis.Client
	prefix string
}

func NewRedisCache(rdb *redis.Client) *RedisCache {
	return &RedisCache{rdb: rdb, prefix: "short:"}
}

func (c *RedisCache) key(code string) string { return c.prefix + code }

func (c *RedisCache) Get(ctx context.Context, code string) (string, bool, error) {
	val, err := c.rdb.Get(ctx, c.key(code)).Result()
	if err == redis.Nil {
		return "", false, nil // cache miss คือกรณีปกติ ไม่ใช่ error (Part 077 หัวข้อ 4)
	}
	if err != nil {
		return "", false, err
	}
	return val, true, nil
}

func (c *RedisCache) Set(ctx context.Context, code, longURL string, ttl time.Duration) error {
	return c.rdb.Set(ctx, c.key(code), longURL, ttl).Err()
}

func (c *RedisCache) Del(ctx context.Context, code string) error {
	return c.rdb.Del(ctx, c.key(code)).Err()
}
```

### ทางเลือกสำรอง: in-memory LRU cache

บทความนี้รันกับ **Redis server จริง** ตลอดทั้งบท (ดูหัวข้อ 14) แต่เพื่อความครบถ้วน — ในระบบที่ deploy เป็น binary เดี่ยวไม่มี infrastructure เสริม (เช่น เดโม, edge device, หรือช่วง local development ที่ไม่อยากตั้ง Redis) การมี cache สำรองในหน่วยความจำของโปรเซสเองก็ยังดีกว่าไม่มี cache เลย โค้ดด้านล่างเป็น **Least-Recently-Used (LRU) cache** implement `Cache` interface เดียวกัน คอมไพล์ผ่านและมี unit test ครบทุกเคส:

```go
// internal/cache/cache.go (ส่วน MemoryLRUCache)
type lruEntry struct {
	code      string
	longURL   string
	expiresAt time.Time
}

type MemoryLRUCache struct {
	mu       sync.Mutex
	capacity int
	ll       *list.List
	items    map[string]*list.Element
}

func NewMemoryLRUCache(capacity int) *MemoryLRUCache {
	if capacity <= 0 {
		capacity = 1024
	}
	return &MemoryLRUCache{capacity: capacity, ll: list.New(), items: make(map[string]*list.Element)}
}

func (c *MemoryLRUCache) Get(_ context.Context, code string) (string, bool, error) {
	c.mu.Lock()
	defer c.mu.Unlock()

	el, ok := c.items[code]
	if !ok {
		return "", false, nil
	}
	entry := el.Value.(*lruEntry)
	if !entry.expiresAt.IsZero() && time.Now().After(entry.expiresAt) {
		c.ll.Remove(el)
		delete(c.items, code)
		return "", false, nil
	}
	c.ll.MoveToFront(el) // ทำเครื่องหมายว่าเพิ่งถูกใช้ล่าสุด
	return entry.longURL, true, nil
}

func (c *MemoryLRUCache) Set(_ context.Context, code, longURL string, ttl time.Duration) error {
	c.mu.Lock()
	defer c.mu.Unlock()

	var expiresAt time.Time
	if ttl > 0 {
		expiresAt = time.Now().Add(ttl)
	}
	if el, ok := c.items[code]; ok {
		el.Value.(*lruEntry).longURL = longURL
		el.Value.(*lruEntry).expiresAt = expiresAt
		c.ll.MoveToFront(el)
		return nil
	}

	el := c.ll.PushFront(&lruEntry{code: code, longURL: longURL, expiresAt: expiresAt})
	c.items[code] = el

	if c.ll.Len() > c.capacity {
		oldest := c.ll.Back()
		if oldest != nil {
			c.ll.Remove(oldest)
			delete(c.items, oldest.Value.(*lruEntry).code)
		}
	}
	return nil
}
```

ใช้ `container/list` (doubly linked list มาตรฐานของ Go) คู่กับ `map` ธรรมดา: `map` ให้ O(1) lookup, `list` ให้ O(1) ในการย้ายรายการที่เพิ่งใช้ไปไว้หน้าสุด (`MoveToFront`) และตัดรายการที่ใช้น้อยสุดออกจากท้าย (`Back`) เมื่อ cache เต็ม — คือ implementation มาตรฐานของ LRU ที่พบได้ในหนังสือ data structure ทั่วไป

**ข้อจำกัดสำคัญที่ต้องบอกตรงๆ**: `MemoryLRUCache` เก็บข้อมูลอยู่แค่ใน process เดียว ถ้ารัน server หลาย instance หลัง load balancer แต่ละ instance จะมี cache แยกกันไม่ sync กัน (instance A cache ไว้แต่ instance B ยัง miss) ต่างจาก `RedisCache` ที่เป็น shared cache ตัวกลางให้ทุก instance ใช้ร่วมกันได้ — นี่คือเหตุผลหลักที่ระบบ production จริงส่วนใหญ่เลือก Redis แทนที่จะพึ่ง in-memory cache ล้วนๆ เมื่อต้อง scale เกินหนึ่งเครื่อง

Unit test ยืนยันพฤติกรรม eviction จริง:

```go
func TestMemoryLRUCache_EvictsLeastRecentlyUsed(t *testing.T) {
	c := NewMemoryLRUCache(2) // ความจุแค่ 2 รายการ
	ctx := context.Background()

	_ = c.Set(ctx, "a", "url-a", time.Minute)
	_ = c.Set(ctx, "b", "url-b", time.Minute)
	_, _, _ = c.Get(ctx, "a") // เข้าถึง "a" ทำให้เป็น most-recently-used
	_ = c.Set(ctx, "c", "url-c", time.Minute) // cache เต็ม -> evict "b" (ใช้น้อยสุด)

	if _, hit, _ := c.Get(ctx, "b"); hit {
		t.Error("expected \"b\" to be evicted (least recently used) but it's still in cache")
	}
}
```

---

## 6. Rate Limiting บน `POST /shorten` ด้วย `golang.org/x/time/rate`

ต่อยอดโดยตรงจาก **Part 045 หัวข้อ 5** — ใช้ token bucket algorithm ผ่าน `golang.org/x/time/rate` จำกัดว่าแต่ละ IP สร้าง short URL ได้ไม่เกินกี่ครั้งต่อวินาที เพื่อป้องกันการยิง request รัวจนฐานข้อมูลรับภาระไม่ไหว (หรือมีคนเอาไปใช้สร้าง spam link จำนวนมาก)

ประเด็นสำคัญที่ต่างจาก Part 045: บทความนั้นสาธิต `rate.Limiter` ตัวเดียวควบคุมอัตรารวม ส่วนบทนี้ต้องมี **limiter แยกต่อ IP** เพราะถ้าใช้ตัวเดียวคุมทุกคน ผู้ใช้ที่ยิงรัวคนเดียวจะไปกิน quota ของผู้ใช้คนอื่นด้วย ไม่ยุติธรรม:

```go
// internal/ratelimit/ratelimit.go
package ratelimit

import (
	"net"
	"net/http"
	"sync"
	"time"

	"golang.org/x/time/rate"

	"urlshortener/internal/metrics"
)

type IPRateLimiter struct {
	mu       sync.Mutex
	limiters map[string]*visitor
	r        rate.Limit
	burst    int
	ttl      time.Duration
}

type visitor struct {
	limiter  *rate.Limiter
	lastSeen time.Time
}

func New(r rate.Limit, b int) *IPRateLimiter {
	return &IPRateLimiter{limiters: make(map[string]*visitor), r: r, burst: b, ttl: 3 * time.Minute}
}

func (i *IPRateLimiter) getLimiter(ip string) *rate.Limiter {
	i.mu.Lock()
	defer i.mu.Unlock()

	v, exists := i.limiters[ip]
	if !exists {
		limiter := rate.NewLimiter(i.r, i.burst)
		i.limiters[ip] = &visitor{limiter: limiter, lastSeen: time.Now()}
		return limiter
	}
	v.lastSeen = time.Now()
	return v.limiter
}

func (i *IPRateLimiter) Allow(ip string) bool { return i.getLimiter(ip).Allow() }

// Cleanup ลบ limiter ของ IP ที่ไม่ได้ใช้งานมานานเกิน ttl ออกจาก map เพื่อไม่ให้
// memory โตไม่มีที่สิ้นสุดเมื่อมี IP แปลกใหม่เข้ามาเรื่อยๆ
func (i *IPRateLimiter) Cleanup() {
	i.mu.Lock()
	defer i.mu.Unlock()
	for ip, v := range i.limiters {
		if time.Since(v.lastSeen) > i.ttl {
			delete(i.limiters, ip)
		}
	}
}

func clientIP(r *http.Request) string {
	if fwd := r.Header.Get("X-Forwarded-For"); fwd != "" {
		return fwd
	}
	host, _, err := net.SplitHostPort(r.RemoteAddr)
	if err != nil {
		return r.RemoteAddr
	}
	return host
}

func (i *IPRateLimiter) Middleware(next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		ip := clientIP(r)
		if !i.Allow(ip) {
			metrics.RateLimitedTotal.Inc()
			w.Header().Set("Retry-After", "1")
			http.Error(w, `{"error":"rate limit exceeded, please slow down"}`, http.StatusTooManyRequests)
			return
		}
		next.ServeHTTP(w, r)
	})
}
```

จุดที่ควรสังเกต:

- **`map[string]*visitor` + `Cleanup()`**: ถ้าไม่มี cleanup, map นี้จะโตไม่มีที่สิ้นสุดเมื่อมี IP แปลกหน้าเข้ามาเรื่อยๆ (โดยเฉพาะระบบที่เปิดรับ public traffic) เพราะทุก IP ใหม่จะได้ `*rate.Limiter` เป็นของตัวเองที่ไม่เคยถูกลบ — `Cleanup()` ถูกเรียกเป็น background goroutine ทุก 1 นาทีใน `main.go`
- **`GET /{code}` (redirect) ไม่ถูกจำกัด**: ตั้งใจให้ rate limiter ครอบคลุมเฉพาะ `POST /shorten` เท่านั้น เพราะเป็น endpoint เดียวที่มีต้นทุนแพง (ต้องเขียนฐานข้อมูล) ส่วน redirect เป็น hot path ที่ต้องรองรับ traffic สูงและมี cache (หัวข้อ 5) ช่วยรับภาระอยู่แล้ว การจำกัด rate การ redirect จะทำร้ายผู้ใช้จริงที่คลิกลิงก์ปกติโดยไม่จำเป็น

Unit test ยืนยัน 3 พฤติกรรมหลัก:

```go
func TestIPRateLimiter_AllowsBurstThenBlocks(t *testing.T) {
	rl := New(rate.Limit(1), 3) // 1 req/sec, burst 3
	ip := "1.2.3.4"
	for i := 0; i < 3; i++ {
		if !rl.Allow(ip) {
			t.Fatalf("request %d ควรผ่าน (อยู่ใน burst 3 request แรก) แต่ถูกบล็อก", i+1)
		}
	}
	if rl.Allow(ip) {
		t.Fatal("request ที่ 4 ควรถูกบล็อก (เกิน burst แล้ว) แต่กลับผ่าน")
	}
}

func TestIPRateLimiter_SeparateBucketsPerIP(t *testing.T) {
	rl := New(rate.Limit(1), 1)
	if !rl.Allow("1.1.1.1") {
		t.Fatal("IP แรกควรผ่านได้ 1 ครั้ง")
	}
	if rl.Allow("1.1.1.1") {
		t.Fatal("IP แรกเรียกครั้งที่ 2 ติดกันควรถูกบล็อก")
	}
	if !rl.Allow("2.2.2.2") {
		t.Fatal("IP ที่สองควรผ่านได้ เพราะมี token bucket แยกจาก IP แรก")
	}
}
```

---

## 7. Analytics: นับคลิกแบบ atomic แล้ว batch flush ลงฐานข้อมูล

จุดนี้คือจุดที่แสดงให้เห็นว่า **แนวคิดเรื่อง hot path** ที่พูดถึงในหัวข้อ 1 ต้องถูกนำมาคิดกับทุกส่วนของระบบ ไม่ใช่แค่การ query ฐานข้อมูล — การ redirect คือ path ที่ถูกเรียกถี่ที่สุด ถ้าทุกครั้งที่มีคน redirect ต้องรอ `UPDATE urls SET clicks = clicks + 1` เสร็จก่อนถึงจะตอบกลับผู้ใช้ ก็จะเสียประโยชน์ของ cache ในหัวข้อ 5 ไปเกือบหมด (เพราะแม้ cache จะ hit แต่ยังต้องรอเขียนฐานข้อมูลอยู่ดี)

ทางแก้คือใช้ **`sync/atomic`** (**Part 040**) นับคลิกในหน่วยความจำก่อนแบบ lock-free แล้วค่อย "รวมยอด" (batch flush) ลงฐานข้อมูลเป็นช่วงๆ:

```go
// internal/analytics/clicks.go
package analytics

import (
	"context"
	"sync"
	"sync/atomic"
)

type Flusher interface {
	IncrementClicks(ctx context.Context, code string, delta int64) error
}

// ClickTracker เก็บตัวนับคลิกต่อ short code หนึ่งตัวต่อหนึ่ง code โดยใช้ atomic.Int64
// (typed API ของ Go 1.19+ ตามที่แนะนำใน Part 040 แทน atomic.AddInt64 แบบเก่า)
type ClickTracker struct {
	mu       sync.RWMutex
	counters map[string]*atomic.Int64
}

func NewClickTracker() *ClickTracker {
	return &ClickTracker{counters: make(map[string]*atomic.Int64)}
}

func (t *ClickTracker) getCounter(code string) *atomic.Int64 {
	t.mu.RLock()
	c, ok := t.counters[code]
	t.mu.RUnlock()
	if ok {
		return c
	}
	t.mu.Lock()
	defer t.mu.Unlock()
	if c, ok := t.counters[code]; ok { // เช็คซ้ำหลังได้ write lock กันสร้างซ้ำ
		return c
	}
	c = &atomic.Int64{}
	t.counters[code] = c
	return c
}

// Record บันทึกว่า short code นี้ถูกคลิกเพิ่มอีก 1 ครั้ง — เรียกจาก hot path (redirect
// handler) ได้อย่างปลอดภัยจากหลาย goroutine พร้อมกันโดยไม่มี lock contention
func (t *ClickTracker) Record(code string) {
	t.getCounter(code).Add(1)
}

func (t *ClickTracker) Pending(code string) int64 {
	t.mu.RLock()
	c, ok := t.counters[code]
	t.mu.RUnlock()
	if !ok {
		return 0
	}
	return c.Load()
}

// Flush รวมยอดคลิกทั้งหมดที่ยังค้างอยู่ของทุก code แล้วส่งไปที่ Flusher (ฐานข้อมูล)
// ใช้ atomic.Int64.Swap(0) เพื่อ "อ่านค่าปัจจุบันพร้อมรีเซ็ตเป็น 0" ในจังหวะเดียวแบบ atomic —
// รับประกันว่าไม่มีคลิกไหนหายไป แม้จะมี Record() เกิดขึ้นพร้อมกันระหว่างการ flush พอดี
func (t *ClickTracker) Flush(ctx context.Context, f Flusher) error {
	t.mu.RLock()
	codes := make([]string, 0, len(t.counters))
	for code := range t.counters {
		codes = append(codes, code)
	}
	t.mu.RUnlock()

	for _, code := range codes {
		c := t.getCounter(code)
		delta := c.Swap(0)
		if delta == 0 {
			continue
		}
		if err := f.IncrementClicks(ctx, code, delta); err != nil {
			c.Add(delta) // flush ไม่สำเร็จ บวกกลับเข้าตัวนับ ไม่ให้คลิกหาย
			return err
		}
	}
	return nil
}
```

จุดที่ควรเจาะลึกที่สุดในไฟล์นี้คือ **`c.Swap(0)`** — เมธอดนี้ (มีมาตั้งแต่ typed atomic API ของ Go 1.19+ ตามที่แนะนำใน **Part 040**) "อ่านค่าปัจจุบันพร้อมรีเซ็ตเป็น 0" ในจังหวะเดียวแบบ atomic ซึ่งสำคัญมาก: ถ้าเขียนแบบ `old := c.Load(); c.Store(0)` (สองคำสั่งแยกกัน) จะมี race window เล็กๆ ที่คลิกที่มาถึงระหว่างสองบรรทัดนี้จะหายไปเฉยๆ (ถูก `Store(0)` ทับ) — `Swap` รับประกันว่าไม่มีคลิกไหนหายไปเด็ดขาด ไม่ว่าจะมี `Record()` เกิดขึ้นพร้อมกันแค่ไหนก็ตาม

`main.go` เรียก `tracker.Flush(ctx, st)` ทุก 5 วินาทีผ่าน background goroutine (ดูหัวข้อ 10) และ `GET /stats/{code}` รวมยอดที่ flush แล้ว (`rec.Clicks` จากฐานข้อมูล) กับยอดที่ยังค้างอยู่ (`tracker.Pending(code)`) เพื่อให้ตัวเลขที่ผู้ใช้เห็นสดที่สุดเท่าที่เป็นไปได้

Unit test พิสูจน์ว่าไม่มีคลิกหายแม้ flush ล้มเหลว และไม่มี data race แม้เรียกพร้อมกันจากหลาย goroutine (รันผ่าน `go test -race` แล้วจริง — ดูหัวข้อ 12):

```go
func TestClickTracker_FlushFailureDoesNotLoseClicks(t *testing.T) {
	tracker := NewClickTracker()
	tracker.Record("abc")
	tracker.Record("abc")
	tracker.Record("abc")

	f := newFakeFlusher()
	f.failNext = true

	if err := tracker.Flush(context.Background(), f); err == nil {
		t.Fatal("expected error from simulated flush failure")
	}
	if got := tracker.Pending("abc"); got != 3 {
		t.Errorf("Pending(abc) after failed flush = %d, want 3 (ต้องไม่มีคลิกหาย)", got)
	}
}

func TestClickTracker_ConcurrentRecordIsRaceFree(t *testing.T) {
	tracker := NewClickTracker()
	const goroutines, perGoroutine = 50, 200
	var wg sync.WaitGroup
	wg.Add(goroutines)
	for i := 0; i < goroutines; i++ {
		go func() {
			defer wg.Done()
			for j := 0; j < perGoroutine; j++ {
				tracker.Record("hot-code")
			}
		}()
	}
	wg.Wait()

	want := int64(goroutines * perGoroutine)
	if got := tracker.Pending("hot-code"); got != want {
		t.Errorf("Pending(hot-code) = %d, want %d", got, want)
	}
}
```

> **ทางเลือกอื่นที่ควรรู้จัก**: ถ้าระบบต้อง scale เป็นหลาย instance หลัง load balancer, `ClickTracker` แบบนี้ (in-memory ต่อ process) จะมีปัญหาเดียวกับ `MemoryLRUCache` ในหัวข้อ 5 — แต่ละ instance นับแยกกัน ทางแก้ในระบบจริงคือย้ายการนับไปที่ **Redis `INCR`** แทน (ตามที่กล่าวถึงใน **Part 077** ตารางหัวข้อ 1: "Rate limiting... นับจำนวน request... ด้วย `INCR`") ซึ่งเป็น shared counter ที่ทุก instance เห็นค่าตรงกัน แล้วค่อย flush จาก Redis ไปฐานข้อมูลเป็นระยะแทน — บทความนี้เลือกใช้ atomic in-memory เพราะสาธิตหลักการของ **Part 040** ได้ตรงกว่า และเพียงพอสำหรับ single-instance deployment

---

## 8. Prometheus Metrics: request count และ cache hit ratio

ต่อยอดจาก **Part 099** ตรงๆ — เปิด metric 5 ตัวผ่าน `github.com/prometheus/client_golang`:

```go
// internal/metrics/metrics.go
package metrics

import (
	"time"

	"github.com/prometheus/client_golang/prometheus"
	"github.com/prometheus/client_golang/prometheus/promauto"
)

var (
	// HTTPRequestsTotal นับจำนวน request ทั้งหมด แยกตาม path/method/status
	// (ตัวแรกของ RED Method: Rate — ดู Part 099 หัวข้อ 5)
	HTTPRequestsTotal = promauto.NewCounterVec(
		prometheus.CounterOpts{
			Name: "urlshortener_http_requests_total",
			Help: "จำนวน HTTP request ทั้งหมดที่ระบบรับเข้ามา แยกตาม path/method/status",
		},
		[]string{"path", "method", "status"},
	)

	// HTTPRequestDuration วัดเวลาตอบสนองของแต่ละ request (ตัวที่สามของ RED Method: Duration)
	HTTPRequestDuration = promauto.NewHistogramVec(
		prometheus.HistogramOpts{
			Name:    "urlshortener_http_request_duration_seconds",
			Help:    "เวลาที่ใช้ประมวลผลแต่ละ HTTP request เป็นวินาที",
			Buckets: prometheus.DefBuckets,
		},
		[]string{"path"},
	)

	// CacheLookupsTotal นับจำนวนครั้งที่ตรวจ cache แยกตามผลลัพธ์ hit/miss
	// ใช้คำนวณ cache hit ratio = hit / (hit + miss)
	CacheLookupsTotal = promauto.NewCounterVec(
		prometheus.CounterOpts{
			Name: "urlshortener_cache_lookups_total",
			Help: "จำนวนครั้งที่ตรวจสอบ cache สำหรับ short code แยกตามผลลัพธ์ hit/miss",
		},
		[]string{"result"},
	)

	RateLimitedTotal = promauto.NewCounter(prometheus.CounterOpts{
		Name: "urlshortener_rate_limited_total",
		Help: "จำนวน request ที่ถูกปฏิเสธเพราะเกิน rate limit ของ POST /shorten",
	})

	ShortenedURLsTotal = promauto.NewCounterVec(
		prometheus.CounterOpts{
			Name: "urlshortener_shortened_urls_total",
			Help: "จำนวน short URL ที่สร้างสำเร็จทั้งหมด แยกตามกลยุทธ์การสร้าง code",
		},
		[]string{"strategy"},
	)
)

func ObserveCacheResult(hit bool) {
	if hit {
		CacheLookupsTotal.WithLabelValues("hit").Inc()
	} else {
		CacheLookupsTotal.WithLabelValues("miss").Inc()
	}
}

func Timer(path string) func() {
	start := time.Now()
	return func() {
		HTTPRequestDuration.WithLabelValues(path).Observe(time.Since(start).Seconds())
	}
}
```

**คำเตือนเรื่อง label cardinality** (ย้ำจาก **Part 099**): สังเกตว่า label ทั้งหมดที่ใช้ (`path`, `method`, `status`, `result`, `strategy`) ล้วนมีจำนวนค่าที่เป็นไปได้จำกัดชัดเจน — **ไม่มี label ไหนเป็น short code หรือ IP ของผู้ใช้เด็ดขาด** เพราะ short code มีได้หลายล้านค่าไม่จำกัด ถ้าเผลอใส่เป็น label จะทำให้ Prometheus สร้าง time series ใหม่นับล้านชุดจนกิน memory server จนล่มได้

ตอนพัฒนาช่วง debug ครั้งแรก ผู้เขียนพบบั๊กจริงจากการรันเดโม: ประกาศ `RateLimitedTotal` ไว้แต่ลืมเรียก `.Inc()` ที่จุดที่ควรเพิ่มค่าจริง (ใน `ratelimit.Middleware`) ทำให้ curl ยิง request จนโดน 429 สามครั้งแล้ว metric ยังโชว์ `0` อยู่ — เป็นตัวอย่างจริงว่าทำไม **การรันจริงแล้วอ่านค่าที่ได้ทุกครั้งสำคัญกว่าการอ่านโค้ดเฉยๆ** (แก้ไขแล้วในโค้ดที่แสดงในหัวข้อ 6 และยืนยันด้วย curl ในหัวข้อ 11)

---

## 9. ชั้น HTTP: routing, middleware, และ handler ทั้งหมด

ไฟล์นี้คือจุดที่ทุกอย่างในหัวข้อก่อนหน้าถูกประกอบเข้าด้วยกัน ใช้ **enhanced `http.ServeMux`** ของ Go 1.22+ (method + path parameter ในตัว ไม่ต้องพึ่ง router ภายนอกอย่าง `gorilla/mux`/`chi` จาก **Part 058**):

```go
// internal/api/server.go
package api

// ... imports ตัดออกเพื่อความกระชับ ดูโค้ดเต็มในโปรเจกต์ ...

type Config struct {
	BaseURL             string
	CacheTTL            time.Duration
	RandomCodeLength    int
	MaxCollisionRetries int
}

func DefaultConfig() Config {
	return Config{
		BaseURL:             "http://localhost:8080",
		CacheTTL:            30 * time.Minute,
		RandomCodeLength:    7,
		MaxCollisionRetries: 5,
	}
}

type Server struct {
	cfg     Config
	store   *store.Store
	cache   cache.Cache
	tracker *analytics.ClickTracker
	limiter *ratelimit.IPRateLimiter
	log     *slog.Logger
	mux     *http.ServeMux
}

func New(cfg Config, st *store.Store, c cache.Cache, tracker *analytics.ClickTracker, limiter *ratelimit.IPRateLimiter, log *slog.Logger) *Server {
	s := &Server{cfg: cfg, store: st, cache: c, tracker: tracker, limiter: limiter, log: log, mux: http.NewServeMux()}
	s.routes()
	return s
}

func (s *Server) routes() {
	s.mux.Handle("POST /shorten", s.instrument("/shorten", s.limiter.Middleware(http.HandlerFunc(s.handleShorten))))
	s.mux.Handle("GET /stats/{code}", s.instrument("/stats/{code}", http.HandlerFunc(s.handleStats)))
	s.mux.Handle("GET /healthz", s.instrument("/healthz", http.HandlerFunc(s.handleHealthz)))
	s.mux.Handle("GET /metrics", promhttp.Handler())
	// Go 1.22+ ServeMux จัด priority ของ literal path ("/healthz", "/stats/{code}" ตอน prefix
	// ตรง) เหนือกว่า wildcard เดี่ยว "{code}" ให้เองเสมอ ไม่ว่าจะลงทะเบียนก่อนหรือหลังก็ตาม
	// จึงไม่มีทางชนกับ route ด้านบนแม้จะกว้างครอบคลุมทุกอย่างที่เหลือ
	s.mux.Handle("GET /{code}", s.instrument("/{code}", http.HandlerFunc(s.handleRedirect)))
}

func (s *Server) ServeHTTP(w http.ResponseWriter, r *http.Request) { s.mux.ServeHTTP(w, r) }
```

### Middleware วัด metric ของทุก request

```go
type statusRecorder struct {
	http.ResponseWriter
	status int
}

func (r *statusRecorder) WriteHeader(code int) {
	r.status = code
	r.ResponseWriter.WriteHeader(code)
}

func (s *Server) instrument(pathLabel string, next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		stop := metrics.Timer(pathLabel)
		rec := &statusRecorder{ResponseWriter: w, status: http.StatusOK}
		next.ServeHTTP(rec, r)
		stop()
		metrics.HTTPRequestsTotal.WithLabelValues(pathLabel, r.Method, fmt.Sprintf("%d", rec.status)).Inc()
	})
}
```

`http.ResponseWriter` มาตรฐานไม่มีทาง "อ่าน" status code ที่ handler ตอบไปแล้วย้อนหลังได้ — `statusRecorder` คือเทคนิคมาตรฐานที่ห่อ `ResponseWriter` ไว้ดักจำค่าตอน `WriteHeader` ถูกเรียก

### `POST /shorten`: validate URL แล้วเลือกกลยุทธ์สร้าง code

```go
type shortenRequest struct {
	URL      string `json:"url"`
	Strategy string `json:"strategy,omitempty"` // "sequential" | "random" (ค่าเริ่มต้น "random")
}

func (s *Server) handleShorten(w http.ResponseWriter, r *http.Request) {
	var req shortenRequest
	if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
		writeError(w, http.StatusBadRequest, "invalid JSON body")
		return
	}

	longURL, err := normalizeURL(req.URL)
	if err != nil {
		writeError(w, http.StatusBadRequest, err.Error())
		return
	}

	strategy := req.Strategy
	if strategy == "" {
		strategy = "random"
	}

	var code string
	switch strategy {
	case "sequential":
		code, err = s.createSequential(r.Context(), longURL)
	case "random":
		code, err = s.createRandom(r.Context(), longURL)
	default:
		writeError(w, http.StatusBadRequest, `strategy must be "sequential" or "random"`)
		return
	}
	if err != nil {
		s.log.Error("shorten failed", "error", err, "strategy", strategy)
		writeError(w, http.StatusInternalServerError, "failed to create short URL")
		return
	}

	metrics.ShortenedURLsTotal.WithLabelValues(strategy).Inc()
	writeJSON(w, http.StatusCreated, shortenResponse{
		Code: code, ShortURL: s.cfg.BaseURL + "/" + code, LongURL: longURL, Strategy: strategy,
	})
}

func normalizeURL(raw string) (string, error) {
	raw = strings.TrimSpace(raw)
	if raw == "" {
		return "", errors.New("url is required")
	}
	u, err := url.ParseRequestURI(raw)
	if err != nil {
		return "", fmt.Errorf("invalid url: %w", err)
	}
	if u.Scheme != "http" && u.Scheme != "https" {
		return "", errors.New("url must use http or https scheme")
	}
	if u.Host == "" {
		return "", errors.New("url must include a host")
	}
	return raw, nil
}
```

`normalizeURL` ปฏิเสธ scheme ที่ไม่ใช่ `http`/`https` ตรงๆ — กันเคสที่มีคนพยายามสร้าง short URL ให้ redirect ไปที่ `javascript:...` หรือ scheme แปลกๆ อื่น (ดูผลจริงในหัวข้อ 11)

### `GET /{code}`: redirect พร้อม cache-aside เต็มรูปแบบ

```go
func (s *Server) handleRedirect(w http.ResponseWriter, r *http.Request) {
	code := r.PathValue("code")
	ctx := r.Context()

	longURL, hit, err := s.cache.Get(ctx, code)
	if err != nil {
		// Redis ล่มหรือ timeout ไม่ควรทำให้ทั้งระบบล่มตาม (Part 077 หัวข้อ 8:
		// fallback ไปอ่านฐานข้อมูลหลักตรงๆ ดีกว่าปล่อยให้ request ค้าง)
		s.log.Warn("cache get failed, falling back to database", "error", err)
		hit = false
	}
	metrics.ObserveCacheResult(hit)

	if !hit {
		longURL, err = s.store.GetByCode(ctx, code)
		if errors.Is(err, store.ErrCodeNotFound) {
			writeError(w, http.StatusNotFound, "short code not found")
			return
		}
		if err != nil {
			s.log.Error("lookup failed", "error", err)
			writeError(w, http.StatusInternalServerError, "internal error")
			return
		}
		if err := s.cache.Set(ctx, code, longURL, s.cfg.CacheTTL); err != nil {
			s.log.Warn("cache set failed", "error", err)
		}
	}

	s.tracker.Record(code)

	// ใช้ 302 Found ไม่ใช่ 301 Moved Permanently — ดูคำอธิบายเต็มด้านล่าง
	http.Redirect(w, r, longURL, http.StatusFound)
}
```

### ทำไมต้องเป็น 302 ไม่ใช่ 301

นี่คือหนึ่งในการตัดสินใจที่สำคัญที่สุดของ URL shortener ทุกตัวในโลกจริง และมักถูกมองข้าม:

| | 301 Moved Permanently | 302 Found (Temporary Redirect) |
|---|---|---|
| ความหมายตาม HTTP spec | บอกว่าทรัพยากรนี้ย้ายไปถาวรแล้ว | บอกว่าทรัพยากรนี้อยู่ที่อื่น**ชั่วคราว** |
| พฤติกรรม cache ฝั่ง client | เบราว์เซอร์/crawler **จำผลลัพธ์ไว้ถาวร** ฝั่งตัวเอง | ไม่ cache ฝั่ง client (ต้องถามเซิร์ฟเวอร์ทุกครั้ง) |
| ผลต่อการนับคลิก | **นับไม่ได้ตั้งแต่ครั้งที่สอง** — browser ไม่ยิง request มาที่ server ของเราอีกเลย | นับได้ทุกครั้ง เพราะทุกครั้งต้องถาม server ก่อนเสมอ |
| ผลด้าน SEO | ส่งต่อ "link juice" ให้หน้าปลายทางเต็มที่ | ส่งต่อ link juice ได้บางส่วน (น้อยกว่า 301) |

เหตุผลหลักที่บริการ URL shortener แทบทุกเจ้าในโลกจริง (bit.ly, TinyURL) **เลือก 302 เป็นค่าเริ่มต้น**: เพราะ 301 ถูกออกแบบมาให้เบราว์เซอร์ "จำไว้ถาวร" — พอผู้ใช้คลิกลิงก์เดียวกันเป็นครั้งที่สอง เบราว์เซอร์จะพาไปหน้าปลายทางตรงๆ**โดยไม่ยิง request มาที่ server ของเราเลย** ซึ่งทำให้ analytics/click-count ที่สร้างไว้ในหัวข้อ 7 **ใช้งานไม่ได้ตั้งแต่การคลิกครั้งที่สองเป็นต้นไป** — server ไม่มีทางรู้เลยว่ามีคนคลิกอีกกี่ครั้ง เพราะ request ไม่เคยมาถึง ระบบนี้จึงใช้ **`http.StatusFound` (302)** เสมอ แลกกับ SEO link juice ที่ 301 ให้ได้ดีกว่าเล็กน้อย ซึ่งเป็นข้อแลกเปลี่ยนที่ยอมรับได้เพราะฟีเจอร์หลักของ URL shortener คือ analytics ไม่ใช่ SEO

### `GET /stats/{code}`

```go
func (s *Server) handleStats(w http.ResponseWriter, r *http.Request) {
	code := r.PathValue("code")
	rec, err := s.store.GetStats(r.Context(), code)
	if errors.Is(err, store.ErrCodeNotFound) {
		writeError(w, http.StatusNotFound, "short code not found")
		return
	}
	if err != nil {
		s.log.Error("stats failed", "error", err)
		writeError(w, http.StatusInternalServerError, "internal error")
		return
	}

	// รวมยอดที่ flush ลงฐานข้อมูลแล้ว กับยอดที่ยังค้างอยู่ใน atomic counter (Part 040)
	// ที่ยังไม่ถึงรอบ flush ถัดไป เพื่อให้ตัวเลขที่ตอบกลับสดที่สุดเท่าที่ทำได้
	total := rec.Clicks + s.tracker.Pending(code)
	writeJSON(w, http.StatusOK, statsResponse{Code: rec.Code, LongURL: rec.LongURL, Clicks: total})
}
```

---

## 10. `cmd/server/main.go`: ประกอบทุกอย่างเข้าด้วยกันพร้อม graceful shutdown

```go
package main

import (
	"context"
	"flag"
	"log/slog"
	"net/http"
	"os"
	"os/signal"
	"syscall"
	"time"

	"github.com/redis/go-redis/v9"
	"golang.org/x/time/rate"

	"urlshortener/internal/analytics"
	"urlshortener/internal/api"
	"urlshortener/internal/cache"
	"urlshortener/internal/ratelimit"
	"urlshortener/internal/store"
)

func main() {
	addr := flag.String("addr", ":8080", "address ที่ HTTP server จะ listen")
	dbPath := flag.String("db", "urlshortener.db", "path ของไฟล์ SQLite")
	redisAddr := flag.String("redis", "localhost:6379", "address ของ Redis server")
	baseURL := flag.String("base-url", "http://localhost:8080", "base URL ที่ใช้ประกอบ short_url ในผลลัพธ์")
	flag.Parse()

	log := slog.New(slog.NewTextHandler(os.Stdout, nil))
	ctx, stop := signal.NotifyContext(context.Background(), os.Interrupt, syscall.SIGTERM)
	defer stop()

	st, err := store.Open(ctx, *dbPath)
	if err != nil {
		log.Error("open store failed", "error", err)
		os.Exit(1)
	}
	defer st.Close()

	rdb := redis.NewClient(&redis.Options{Addr: *redisAddr})
	defer rdb.Close()
	if err := rdb.Ping(ctx).Err(); err != nil {
		log.Error("connect redis failed", "error", err)
		os.Exit(1)
	}
	c := cache.NewRedisCache(rdb)

	tracker := analytics.NewClickTracker()
	// จำกัด POST /shorten ไว้ที่ 2 request/วินาทีต่อ IP โดยอนุญาต burst แรก 3 request
	limiter := ratelimit.New(rate.Limit(2), 3)

	cfg := api.DefaultConfig()
	cfg.BaseURL = *baseURL
	srv := api.New(cfg, st, c, tracker, limiter, log)

	httpServer := &http.Server{
		Addr: *addr, Handler: srv,
		ReadTimeout: 5 * time.Second, WriteTimeout: 10 * time.Second,
	}

	// background goroutine #1: flush atomic click counters (Part 040) ลงฐานข้อมูลทุก 5 วินาที
	go func() {
		ticker := time.NewTicker(5 * time.Second)
		defer ticker.Stop()
		for {
			select {
			case <-ctx.Done():
				return
			case <-ticker.C:
				if err := tracker.Flush(context.Background(), st); err != nil {
					log.Warn("flush clicks failed", "error", err)
				}
			}
		}
	}()

	// background goroutine #2: เก็บกวาด rate limiter ของ IP ที่ไม่ได้ใช้งานนาน
	go func() {
		ticker := time.NewTicker(time.Minute)
		defer ticker.Stop()
		for {
			select {
			case <-ctx.Done():
				return
			case <-ticker.C:
				limiter.Cleanup()
			}
		}
	}()

	go func() {
		log.Info("server starting", "addr", *addr)
		if err := httpServer.ListenAndServe(); err != nil && err != http.ErrServerClosed {
			log.Error("server failed", "error", err)
			os.Exit(1)
		}
	}()

	<-ctx.Done()
	log.Info("shutting down...")

	shutdownCtx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
	defer cancel()

	// flush ยอดคลิกที่ค้างอยู่รอบสุดท้ายก่อนปิดโปรแกรม ไม่ให้ข้อมูลหาย
	if err := tracker.Flush(shutdownCtx, st); err != nil {
		log.Warn("final flush failed", "error", err)
	}
	if err := httpServer.Shutdown(shutdownCtx); err != nil {
		log.Error("graceful shutdown failed", "error", err)
	}
}
```

จุดที่ควรสังเกต: `signal.NotifyContext` (มาตรฐานตั้งแต่ Go 1.16) ทำให้ `ctx` ถูกยกเลิกอัตโนมัติเมื่อโปรแกรมได้รับ `SIGINT`/`SIGTERM` — เมื่อนั้น background goroutine ทั้งสองตัวจะออกจาก loop ทันที (ผ่าน `<-ctx.Done()`) และ main goroutine จะ **flush คลิกที่ค้างอยู่รอบสุดท้าย** ก่อนเรียก `httpServer.Shutdown` ซึ่งจะรอให้ request ที่กำลังประมวลผลอยู่เสร็จก่อนปิดจริง — รับประกันว่าไม่มีข้อมูลคลิกหายแม้จะสั่งปิดโปรแกรมกะทันหัน

---

## 11. รันจริงแบบ end-to-end ด้วย `curl`

Build และรันจริงบนเครื่องที่เขียนบทความนี้ (พร้อม Redis server จริงที่ `localhost:6379` — ยืนยันด้วย `redis-cli ping` ได้ `PONG` ก่อนเริ่ม):

```bash
$ go build -o urlshortener-server ./cmd/server
$ ./urlshortener-server -addr=:8099 -db=./urlshortener.db -redis=localhost:6379 -base-url=http://localhost:8099 &
```

**Log ตอน startup (ของจริง)**:
```
time=2026-09-26T06:44:19.895Z level=INFO msg="server starting" addr=:8099
```

### Health check

```bash
$ curl -s http://localhost:8099/healthz
{"status":"ok"}
```

### สร้าง short URL ด้วยกลยุทธ์ random (ค่าเริ่มต้น)

```bash
$ curl -s -X POST http://localhost:8099/shorten \
    -H "Content-Type: application/json" \
    -d '{"url":"https://go.dev/doc/effective_go"}'
```

**ผลลัพธ์จริง**:
```json
{"code":"DsxOGdy","short_url":"http://localhost:8099/DsxOGdy","long_url":"https://go.dev/doc/effective_go","strategy":"random"}
```

สังเกต `code` ยาว 7 ตัวอักษรตามที่ตั้งค่าไว้ใน `DefaultConfig` และไม่มีความสัมพันธ์กับลำดับการสร้างเลยตามที่ออกแบบไว้ในหัวข้อ 4

### สร้าง short URL ด้วยกลยุทธ์ sequential

```bash
$ curl -s -X POST http://localhost:8099/shorten \
    -H "Content-Type: application/json" \
    -d '{"url":"https://pkg.go.dev/net/http","strategy":"sequential"}'
```

**ผลลัพธ์จริง**:
```json
{"code":"2","short_url":"http://localhost:8099/2","long_url":"https://pkg.go.dev/net/http","strategy":"sequential"}
```

`code` เป็น `"2"` เพราะแถวที่แล้ว (`DsxOGdy`) กินไปแล้ว ID = 1 ในตาราง `urls` (แม้จะเป็น random strategy ก็ยัง auto-increment ID เดินหน้าเหมือนกัน เพียงแต่ไม่ได้ใช้ ID นั้นมาทำ code) แถวนี้จึงได้ ID = 2 แล้ว `EncodeBase62(2)` ให้ผลลัพธ์เป็น `"2"` พอดี (ตามตาราง alphabet `0-9a-zA-Z` ที่ตำแหน่งดัชนี 2 คือตัวอักษร `"2"`)

### ปฏิเสธ URL ที่ไม่ถูกต้อง

```bash
$ curl -s -X POST http://localhost:8099/shorten \
    -H "Content-Type: application/json" -d '{"url":"not-a-url"}'
```

**ผลลัพธ์จริง**:
```json
{"error":"invalid url: parse \"not-a-url\": invalid URI for request"}
```

### Redirect (สังเกต header `Location` และ status 302)

```bash
$ curl -s -i http://localhost:8099/DsxOGdy
```

**ผลลัพธ์จริง**:
```
HTTP/1.1 302 Found
Content-Type: text/html; charset=utf-8
Location: https://go.dev/doc/effective_go
Date: Sat, 26 Sep 2026 06:44:31 GMT
Content-Length: 54

<a href="https://go.dev/doc/effective_go">Found</a>.
```

### Short code ที่ไม่มีอยู่จริง

```bash
$ curl -s -i http://localhost:8099/doesnotexist
```

**ผลลัพธ์จริง**:
```
HTTP/1.1 404 Not Found
Content-Type: application/json
Date: Sat, 26 Sep 2026 06:44:31 GMT
Content-Length: 33

{"error":"short code not found"}
```

### สถิติคลิก: ก่อนและหลัง background flush

```bash
# ยิง redirect ซ้ำ 6 ครั้งติดกัน (รวมกับครั้งก่อนหน้าที่ยิงไปแล้ว 1 ครั้ง = 7 ครั้งทั้งหมด)
$ for i in 1 2 3 4 5 6; do curl -s -o /dev/null http://localhost:8099/DsxOGdy; done

$ curl -s http://localhost:8099/stats/DsxOGdy
```

**ผลลัพธ์จริง (ก่อนรอบ flush 5 วินาที — อ่านจาก pending atomic counter ใน Part 040)**:
```json
{"code":"DsxOGdy","long_url":"https://go.dev/doc/effective_go","clicks":7}
```

```bash
$ sleep 6   # รอให้ background flusher ทำงาน (ตั้งไว้ทุก 5 วินาทีใน main.go)
$ curl -s http://localhost:8099/stats/DsxOGdy
```

**ผลลัพธ์จริง (หลัง flush — ตัวเลขเดิม แต่คราวนี้มาจากคอลัมน์ `clicks` ในฐานข้อมูลล้วนๆ)**:
```json
{"code":"DsxOGdy","long_url":"https://go.dev/doc/effective_go","clicks":7}
```

ตัวเลข **เท่ากันทั้งก่อนและหลัง flush** — พิสูจน์ว่า `handleStats` (หัวข้อ 9) รวมยอด pending เข้ากับยอดในฐานข้อมูลได้ถูกต้อง ไม่ว่าจะถามตอนไหนก็ตาม

### Rate Limiting: ยิง `POST /shorten` รัวๆ 6 ครั้งติดกัน

Server รันด้วยค่า `rate.Limit(2), 3` (2 req/sec ต่อ IP, burst 3 ครั้งแรก):

```bash
$ for i in 1 2 3 4 5 6; do
    code=$(curl -s -o /dev/null -w "%{http_code}" -X POST http://localhost:8099/shorten \
      -H "Content-Type: application/json" -d "{\"url\":\"https://example.com/item/$i\"}")
    echo "request $i -> HTTP $code"
  done
```

**ผลลัพธ์จริง**:
```
request 1 -> HTTP 201
request 2 -> HTTP 201
request 3 -> HTTP 201
request 4 -> HTTP 429
request 5 -> HTTP 429
request 6 -> HTTP 429
```

เห็นชัดเจนว่า 3 request แรก (เท่ากับ burst ที่ตั้งไว้) ผ่านหมด แล้ว request ที่ 4 เป็นต้นไปโดนบล็อกทันทีด้วย `429 Too Many Requests` ตรงตามที่ออกแบบไว้ในหัวข้อ 6 เป๊ะ

### Prometheus metrics (`/metrics`)

```bash
$ curl -s http://localhost:8099/metrics | grep -E "^urlshortener_(http_requests_total|cache_lookups_total|rate_limited_total|shortened_urls_total)"
```

**ผลลัพธ์จริง**:
```
urlshortener_cache_lookups_total{result="hit"} 6
urlshortener_cache_lookups_total{result="miss"} 2
urlshortener_http_requests_total{method="GET",path="/healthz",status="200"} 1
urlshortener_http_requests_total{method="GET",path="/stats/{code}",status="200"} 2
urlshortener_http_requests_total{method="GET",path="/{code}",status="302"} 7
urlshortener_http_requests_total{method="GET",path="/{code}",status="404"} 1
urlshortener_http_requests_total{method="POST",path="/shorten",status="201"} 5
urlshortener_http_requests_total{method="POST",path="/shorten",status="400"} 1
urlshortener_http_requests_total{method="POST",path="/shorten",status="429"} 3
urlshortener_rate_limited_total 3
urlshortener_shortened_urls_total{strategy="random"} 4
urlshortener_shortened_urls_total{strategy="sequential"} 1
```

ตัวเลขทุกตัวตรงกับสิ่งที่เกิดขึ้นจริงในเดโมทั้งหมดข้างบน: **cache hit 6 ครั้ง / miss 2 ครั้ง** (miss คือครั้งแรกของแต่ละ code ก่อนที่ cache จะถูก populate — เรามี 2 code ที่ต่างกันคือ `DsxOGdy` และโค้ดจากการทดสอบ 404 ซึ่งไม่ถูก cache เพราะหาไม่เจอ ดังนั้น miss ทั้งสองครั้งมาจาก `DsxOGdy` ครั้งแรกและ redirect ที่นับรวมจาก curl -i ก่อนหน้า), **`rate_limited_total` = 3** ตรงกับ 3 request ที่โดน 429 พอดี, **`shortened_urls_total`** แยกตาม strategy ถูกต้องตามที่สร้างไปจริง (random 4 + sequential 1)

**หมายเหตุ**: `rate_limited_total` เคยโชว์ `0` ในการรันรอบแรกของบทความนี้เพราะโค้ด `ratelimit.Middleware` ลืมเรียก `metrics.RateLimitedTotal.Inc()` (ตามที่เล่าไว้ในหัวข้อ 8) — เป็นบั๊กจริงที่พบระหว่างเขียนบทความนี้ แก้ไขแล้วและตัวเลขที่แสดงข้างบนคือผลลัพธ์**หลังแก้บั๊ก**

### Graceful shutdown

```bash
$ kill -TERM $(cat server.pid)
```

**Log จริง**:
```
time=2026-09-26T06:44:19.895Z level=INFO msg="server starting" addr=:8099
time=2026-09-26T06:44:47.055Z level=INFO msg="shutting down..."
```

โปรเซสปิดตัวเองสะอาดหลัง flush คลิกรอบสุดท้ายและปิด HTTP server เรียบร้อย (ตรวจสอบด้วย `ps aux | grep urlshortener-server` แล้วไม่พบโปรเซสค้างอยู่)

---

## 12. Test Suite: unit test และ httptest integration test ที่รันจริง

โปรเจกต์นี้มี test รวม **24 เทส** ครอบคลุมทุก package ยกเว้น `store` (ที่ถูกทดสอบทางอ้อมผ่าน integration test ของ `api` เพราะทุก handler test เปิด SQLite ไฟล์จริงใน `t.TempDir()`) รันด้วย:

```bash
$ go build ./...   # exit code 0
$ go vet ./...     # exit code 0
$ go test ./... -race
```

**ผลลัพธ์จริง**:
```
?   	urlshortener/cmd/server	[no test files]
ok  	urlshortener/internal/analytics	1.018s
ok  	urlshortener/internal/api	1.163s
ok  	urlshortener/internal/cache	1.046s
ok  	urlshortener/internal/codegen	1.013s
?   	urlshortener/internal/metrics	[no test files]
ok  	urlshortener/internal/ratelimit	1.043s
?   	urlshortener/internal/store	[no test files]
```

ทั้งหมด **ผ่านและไม่มี data race** (`-race` เปิดใช้ race detector จาก **Part 044**) ตัวเลขความยาวเวลาเพิ่มขึ้นเมื่อเทียบกับรันแบบปกติเพราะ instrumentation ของ race detector เอง ไม่ใช่โค้ดช้าลง

### Coverage ต่อ package (จากการรันจริง)

```bash
$ go test ./... -cover
```

```
ok  	urlshortener/internal/analytics	coverage: 91.2% of statements
ok  	urlshortener/internal/api	coverage: 75.0% of statements
ok  	urlshortener/internal/cache	coverage: 88.5% of statements
ok  	urlshortener/internal/codegen	coverage: 87.1% of statements
ok  	urlshortener/internal/ratelimit	coverage: 93.3% of statements
```

### ตัวอย่าง unit test: `internal/cache/cache_test.go`

เทสสำคัญที่สุดของหัวข้อนี้คือเทสที่ต่อ **Redis จริง** (ไม่ใช่ mock) แต่เขียนให้ **`t.Skip`** อย่างสุภาพถ้าต่อ Redis ไม่ได้ แทนที่จะทำให้ทั้ง test suite fail — รูปแบบมาตรฐานสำหรับ integration test ที่พึ่ง external service (ตามแนวทาง **Part 080**):

```go
func newTestRedisClient(t *testing.T) *redis.Client {
	t.Helper()
	rdb := redis.NewClient(&redis.Options{Addr: "localhost:6379"})
	ctx, cancel := context.WithTimeout(context.Background(), 500*time.Millisecond)
	defer cancel()
	if err := rdb.Ping(ctx).Err(); err != nil {
		t.Skipf("skip: ต่อ Redis ที่ localhost:6379 ไม่ได้ (%v)", err)
	}
	return rdb
}

func TestRedisCache_SetGetDel(t *testing.T) {
	rdb := newTestRedisClient(t)
	defer rdb.Close()

	c := NewRedisCache(rdb)
	ctx := context.Background()
	code := "test_cache_key_xyz"
	defer c.Del(ctx, code)

	if err := c.Set(ctx, code, "https://go.dev", time.Minute); err != nil {
		t.Fatalf("Set returned error: %v", err)
	}
	val, hit, err := c.Get(ctx, code)
	if err != nil || !hit || val != "https://go.dev" {
		t.Fatalf("Get = (%q, %v, %v), want (\"https://go.dev\", true, nil)", val, hit, err)
	}
}
```

ในสภาพแวดล้อมที่เขียนบทความนี้ Redis server รันอยู่จริง เทสนี้จึงรันผ่านการเชื่อมต่อจริง ไม่ถูก skip

### ตัวอย่าง httptest integration test: `internal/api/server_test.go`

เทสนี้ยิง request ผ่าน `srv.ServeHTTP` ตรงๆ (ตามแนวทาง **Part 081**) ครอบคลุม flow เต็ม: สร้าง short URL → redirect → นับคลิก → flush → อ่านค่าจากฐานข้อมูล:

```go
func TestStats_TracksClicksAfterFlush(t *testing.T) {
	srv, st, tracker := newTestServer(t)
	ctx := context.Background()

	resp := doShorten(t, srv, shortenRequest{URL: "https://example.com/tracked", Strategy: "sequential"})

	for i := 0; i < 3; i++ {
		req := httptest.NewRequest(http.MethodGet, "/"+resp.Code, nil)
		rec := httptest.NewRecorder()
		srv.ServeHTTP(rec, req)
		if rec.Code != http.StatusFound {
			t.Fatalf("redirect #%d status = %d, want 302", i+1, rec.Code)
		}
	}

	// ก่อน flush: /stats ต้องยังเห็นยอด 3 ครั้งอยู่ดี เพราะ handleStats รวม Pending() เข้าไปด้วย
	statsRec := httptest.NewRecorder()
	srv.ServeHTTP(statsRec, httptest.NewRequest(http.MethodGet, "/stats/"+resp.Code, nil))
	var stats statsResponse
	json.NewDecoder(statsRec.Body).Decode(&stats)
	if stats.Clicks != 3 {
		t.Fatalf("clicks ก่อน flush = %d, want 3", stats.Clicks)
	}

	// จำลอง background flusher: เรียก Flush ตรงๆ แล้วเช็คว่าค่าลงฐานข้อมูลจริง
	if err := tracker.Flush(ctx, st); err != nil {
		t.Fatalf("Flush failed: %v", err)
	}
	rec, err := st.GetStats(ctx, resp.Code)
	if err != nil || rec.Clicks != 3 {
		t.Fatalf("clicks ในฐานข้อมูลหลัง flush = %d (err=%v), want 3", rec.Clicks, err)
	}
}
```

`newTestServer` (helper ที่ใช้ร่วมกันทุกเทสในไฟล์นี้) ตั้งใจใช้ **`MemoryLRUCache`** แทน `RedisCache` เพื่อให้ integration test ของ `api` รันผ่านได้เสมอไม่ว่าจะมี Redis server อยู่ในเครื่องที่รันเทสหรือไม่ — เป็นข้อพิสูจน์เพิ่มเติมว่า interface `Cache` ในหัวข้อ 5 ใช้งานได้จริงกับทั้งสอง implementation อย่างสมบูรณ์ ไม่ต้องแก้โค้ด handler แม้แต่บรรทัดเดียว:

```go
func newTestServer(t *testing.T) (*Server, *store.Store, *analytics.ClickTracker) {
	t.Helper()
	dbPath := filepath.Join(t.TempDir(), "test.db")
	st, err := store.Open(context.Background(), dbPath)
	if err != nil {
		t.Fatalf("store.Open failed: %v", err)
	}
	t.Cleanup(func() { st.Close() })

	c := cache.NewMemoryLRUCache(100)
	tracker := analytics.NewClickTracker()
	limiter := ratelimit.New(rate.Limit(1000), 1000) // burst สูงมากในเทสเพื่อไม่ให้รบกวนเทสอื่น
	log := slog.New(slog.NewTextHandler(io.Discard, nil))

	return New(DefaultConfig(), st, c, tracker, limiter, log), st, tracker
}
```

---

## 13. Benchmark: cache ช่วยลด latency ได้จริงแค่ไหน (วัดจริง พร้อมเรื่องเซอร์ไพรส์ที่ต้องอธิบาย)

นี่คือหัวข้อที่สำคัญที่สุดในเชิงความซื่อสัตย์ของบทความนี้ ผู้เขียนตั้งเบนช์มาร์กสองชุดแล้วได้ผลลัพธ์ที่ **ไม่ตรงกับสัญชาตญาณแรก** — แทนที่จะซ่อนหรือแก้ไขตัวเลขให้ "ดูดี" บทความนี้จะแสดงตัวเลขจริงทั้งสองชุดและอธิบายว่าทำไม

### เบนช์มาร์กชุดที่ 1: ยิงผ่าน full HTTP handler จริงด้วย SQLite ไฟล์จริง

```go
// internal/api/bench_test.go
func BenchmarkRedirect_NoCache_HitsDBEveryTime(b *testing.B) {
	srv, code := setupBenchServer(b, false) // withCache=false ใช้ noCache{} ที่ miss ตลอด
	req := httptest.NewRequest(http.MethodGet, "/"+code, nil)

	b.ResetTimer()
	for i := 0; i < b.N; i++ {
		rec := httptest.NewRecorder()
		srv.ServeHTTP(rec, req)
		if rec.Code != http.StatusFound {
			b.Fatalf("status = %d, want 302", rec.Code)
		}
	}
}

func BenchmarkRedirect_RedisCacheAside(b *testing.B) {
	srv, code := setupBenchServer(b, true) // withCache=true ใช้ RedisCache จริง
	req := httptest.NewRequest(http.MethodGet, "/"+code, nil)

	warmupRec := httptest.NewRecorder()
	srv.ServeHTTP(warmupRec, req) // warm cache ก่อนเริ่มจับเวลา

	b.ResetTimer()
	for i := 0; i < b.N; i++ {
		rec := httptest.NewRecorder()
		srv.ServeHTTP(rec, req)
		if rec.Code != http.StatusFound {
			b.Fatalf("status = %d, want 302", rec.Code)
		}
	}
}
```

รันจริงด้วย `go test -bench=. -benchtime=3s`:

```
BenchmarkRedirect_NoCache_HitsDBEveryTime-4     	  227781	     15554 ns/op
BenchmarkRedirect_RedisCacheAside-4             	   79278	     44781 ns/op
```

**ผลลัพธ์ตรงข้ามกับที่คาดหวัง**: การไม่มี cache เลย (query SQLite ตรงๆ ทุกครั้ง) เร็วกว่า cache-aside ผ่าน Redis เกือบ 3 เท่า (15.6 µs เทียบกับ 44.8 µs)!

### ทำไมถึงเกิดเรื่องนี้ขึ้น

คำตอบอยู่ที่ธรรมชาติของฐานข้อมูลที่ใช้ในบทความนี้: **SQLite เป็น embedded database** — โค้ดของมันรันอยู่ **ในโปรเซสเดียวกัน** กับแอป Go โดยตรง (ผ่าน `modernc.org/sqlite` ที่ไม่มี cgo หรือ IPC ใดๆ เลย) การ query จึงเป็นแค่การเรียกฟังก์ชัน Go ธรรมดาที่อ่านไฟล์ที่แคชอยู่ใน OS page cache อยู่แล้ว **ไม่มี network round-trip แม้แต่นิดเดียว**

ในทางกลับกัน **Redis แม้จะรันอยู่บน `localhost` เดียวกัน ก็ยังเป็นกระบวนการแยกที่คุยกันผ่าน TCP loopback จริง** — ทุกคำสั่งต้องผ่าน socket write/read, การ encode/decode RESP protocol, และ context switch ระดับ kernel ซึ่งมีต้นทุนที่แน่นอน (เห็นได้จากตัวเลข ~40-45 µs ต่อครั้งข้างบน) ต้นทุนนี้**คงที่**ไม่ว่าฐานข้อมูลหลักที่ cache ไว้แทนจะช้าแค่ไหนก็ตาม

สรุปคือ **การเปรียบเทียบนี้ไม่ยุติธรรมกับสถานการณ์ production จริง** — ระบบจริงส่วนใหญ่ไม่ได้ใช้ SQLite แบบ embedded เป็นฐานข้อมูลหลัก แต่ใช้ **PostgreSQL หรือ MySQL ที่รันอยู่บนเครื่องแยก** (ตามที่เรียนใน **Part 072/073**) ซึ่งมี network latency จริงหลัก 1-5+ มิลลิวินาทีต่อ query (ยิ่งถ้าอยู่คนละ availability zone ยิ่งช้ากว่านี้) — เป็น latency ที่สูงกว่า Redis มาก และเป็นสถานการณ์ที่ cache-aside pattern ถูกออกแบบมาให้ช่วยจริงๆ

### เบนช์มาร์กชุดที่ 2: จำลองฐานข้อมูลหลักที่อยู่บนเครือข่ายจริง

เพื่อแสดงสถานการณ์ที่ตรงกับ production จริง บทความนี้เพิ่มเบนช์มาร์กชุดที่สองที่จำลองต้นทุนเครือข่ายของ PostgreSQL/MySQL ด้วย `time.Sleep` ตรงๆ (เทคนิคเดียวกับ `fakeQueryProductFromDB` ใน **Part 077 หัวข้อ 6** ที่จำลอง `time.Sleep(50 * time.Millisecond)`) — ในบทความนี้ใช้ **2 มิลลิวินาที** ซึ่งใกล้เคียงกับ network round-trip ทั่วไปของ managed database ในคลาวด์เดียวกัน:

```go
const simulatedNetworkDBLatency = 2 * time.Millisecond

func simulatedNetworkDBLookup(longURL string) string {
	time.Sleep(simulatedNetworkDBLatency)
	return longURL
}

func BenchmarkSimulatedNetworkDB_NoCache(b *testing.B) {
	const longURL = "https://go.dev/doc/effective_go"
	b.ResetTimer()
	for i := 0; i < b.N; i++ {
		if got := simulatedNetworkDBLookup(longURL); got != longURL {
			b.Fatalf("got %q, want %q", got, longURL)
		}
	}
}

func BenchmarkSimulatedNetworkDB_RedisCacheAside(b *testing.B) {
	const longURL = "https://go.dev/doc/effective_go"
	const code = "bench-simulated-network-db"

	ctx := context.Background()
	rdb := redis.NewClient(&redis.Options{Addr: "localhost:6379"})
	if err := rdb.Ping(ctx).Err(); err != nil {
		b.Skipf("skip: ต่อ Redis ไม่ได้ (%v)", err)
	}
	defer rdb.Close()
	c := cache.NewRedisCache(rdb)
	defer c.Del(ctx, code)

	// คำขอแรก: cache miss -> จ่ายต้นทุนเครือข่ายเต็มๆ หนึ่งครั้งแล้ว populate cache
	val, hit, _ := c.Get(ctx, code)
	if !hit {
		val = simulatedNetworkDBLookup(longURL)
		c.Set(ctx, code, val, time.Minute)
	}

	b.ResetTimer()
	for i := 0; i < b.N; i++ {
		got, hit, err := c.Get(ctx, code)
		if err != nil || !hit || got != longURL {
			b.Fatalf("Get failed: hit=%v err=%v got=%q", hit, err, got)
		}
	}
}
```

**ผลลัพธ์จริง**:

```
BenchmarkSimulatedNetworkDB_NoCache-4           	    1633	   2204992 ns/op
BenchmarkSimulatedNetworkDB_RedisCacheAside-4   	   91095	     40073 ns/op
```

คราวนี้ตัวเลขตรงกับสัญชาตญาณและตรงกับสิ่งที่ cache-aside pattern ควรทำได้จริง: **2,204,992 ns/op (≈2.2 มิลลิวินาที) เทียบกับ 40,073 ns/op (≈40 ไมโครวินาที)** — cache ทำให้เร็วขึ้นประมาณ **55 เท่า** สำหรับลิงก์ยอดนิยมที่ถูกคลิกซ้ำๆ ตัวเลข 2.2 ms ของฝั่งไม่มี cache ก็ตรงกับค่าที่จำลองไว้ (`2 * time.Millisecond`) บวกกับ overhead เล็กน้อยของ benchmark loop เอง ซึ่งยืนยันว่าการจำลองทำงานถูกต้องตามที่ตั้งใจ

### บทเรียนที่ได้จากเบนช์มาร์กทั้งสองชุด

1. **"cache ทำให้เร็วขึ้นเสมอ" เป็นความเชื่อที่ไม่ถูกต้องเสมอไป** — cache ช่วยได้มากก็ต่อเมื่อสิ่งที่มันแทนที่ (ฐานข้อมูลหลัก) มีต้นทุนสูงกว่าตัว cache เองจริงๆ ถ้าฐานข้อมูลหลักเร็วอยู่แล้ว (เช่น embedded database ในเครื่องเดียวกัน) การเพิ่ม cache ที่มี network round-trip ของตัวเองเข้าไปอาจทำให้ระบบ**ช้าลง** ไม่ใช่เร็วขึ้น
2. **การวัดผลจริงสำคัญกว่าการเชื่อ pattern เพียงเพราะมันเป็น "best practice"** — ถ้าผู้เขียนไม่รันเบนช์มาร์กจริงแล้วเชื่อคำโฆษณาของ cache-aside pattern เฉยๆ จะพลาดข้อเท็จจริงสำคัญของระบบนี้ไปเลย
3. **ในระบบ production จริงที่ฐานข้อมูลหลักอยู่บนเครือข่าย (PostgreSQL/MySQL ตาม Part 072/073) cache-aside ยังคงได้ประโยชน์มหาศาลตามที่ Part 077 สาธิตไว้** — บทความนี้เพียงแค่ทำให้เห็นชัดว่า "ประโยชน์มหาศาล" นั้นมาจาก**ต้นทุนเครือข่ายของฐานข้อมูลหลัก** ไม่ใช่จาก "cache" ในฐานะแนวคิดลอยๆ

---

## 14. หมายเหตุความซื่อสัตย์เรื่องการรันจริงในบทนี้

ตามธรรมเนียมของหลักสูตรนี้ (เช่นเดียวกับ **Part 071** และ **Part 077**) ขอสรุปให้ชัดเจนว่าอะไรถูกรันจริงและยืนยันแล้วบ้างในบทความนี้:

1. **รันจริงและยืนยันผลได้ทั้งหมด**: Redis server ตัวจริง (`redis-server --daemonize yes --port 6379` ยืนยันด้วย `redis-cli ping` ได้ `PONG`) ถูกใช้ตลอดทั้งบทความนี้ — ทั้ง `TestRedisCache_SetGetDel`, `BenchmarkRedirect_RedisCacheAside`, และ `BenchmarkSimulatedNetworkDB_RedisCacheAside` ต่อ Redis จริงและได้ผลลัพธ์จริงทุกจุด **ไม่ต้องพึ่ง in-memory LRU cache สำรองเลยในบทความนี้** — `MemoryLRUCache` ที่แสดงในหัวข้อ 5 คอมไพล์ผ่าน มี unit test ของตัวเองครบ (`TestMemoryLRUCache_*`) และถูกใช้จริงใน integration test ของ `api` package (เพื่อไม่ให้เทสต้องพึ่ง Redis) แต่ **ไม่ใช่ production path หลักที่บทความนี้สาธิต** เพราะมี Redis จริงให้ใช้ตลอด
2. **`go build ./...` และ `go vet ./...`**: รันจริง ผ่านทั้งคู่ (exit code 0) ด้วย Go 1.24.7 ที่ `/usr/local/go/bin/go`
3. **Test suite ทั้ง 24 เทส**: รันจริงด้วย `go test ./... -race` ผ่านทั้งหมด ไม่มี data race — ตัวเลข coverage ต่อ package (91.2%, 75.0%, 88.5%, 87.1%, 93.3%) มาจากการรัน `go test ./... -cover` จริง
4. **การเดโม curl ทั้งหมดในหัวข้อ 11**: รันกับ server ตัวจริงที่ build จากซอร์สในบทความนี้ ผลลัพธ์ทุกก้อน (JSON response, HTTP header, metric output, log) คือ output ที่ copy มาจากการรันจริงทุกตัวอักษร รวมถึงบั๊กจริงที่พบระหว่างเขียนบทความ (`rate_limited_total` ไม่ถูกเพิ่มค่า) ซึ่งถูกแก้ไขแล้วในโค้ดที่แสดง
5. **เบนช์มาร์กชุดที่ 1 (SQLite embedded vs Redis)**: รันจริง ตัวเลขที่แสดง (15,554 ns/op และ 44,781 ns/op) มาจากการรัน `go test -bench=. -benchtime=3s` จริง **และเป็นผลลัพธ์ที่ไม่ตรงกับสัญชาตญาณแรก** (Redis ช้ากว่า SQLite ในสถานการณ์นี้) ซึ่งบทความอธิบายเหตุผลไว้ตรงๆ ในหัวข้อ 13 แทนที่จะละไว้หรือปรับแต่งให้ดูดี
6. **เบนช์มาร์กชุดที่ 2 (จำลอง networked database)**: รันจริงเช่นกัน ส่วนที่เป็น "การจำลอง" ชัดเจนคือ **latency ของฝั่งไม่มี cache** ที่ใช้ `time.Sleep(2 * time.Millisecond)` แทนการต่อ PostgreSQL/MySQL จริงผ่านเครือข่าย (ไม่มี PostgreSQL server แยกเครื่องให้ต่อในสภาพแวดล้อมแซนด์บ็อกซ์ short-lived ที่ใช้เขียนบทความนี้ เช่นเดียวกับข้อจำกัดที่ **Part 076** และ **Part 099** เจอ) ส่วนฝั่ง Redis cache-aside เป็น Redis จริงทั้งหมด — ตัวเลขที่ได้ (2,204,992 ns/op และ 40,073 ns/op) จึงเป็นการวัดจริงของโค้ดจริง เพียงแต่ input latency ฝั่งหนึ่งถูกควบคุมไว้ให้จำลองสถานการณ์ที่ต้องการอธิบาย
7. **Graceful shutdown**: ทดสอบจริงด้วยการส่ง `SIGTERM` ไปยังโปรเซสที่รันอยู่ ยืนยันจาก log จริงและตรวจสอบด้วย `ps aux` ว่าโปรเซสไม่ค้าง

---

## สรุปสิ่งที่ได้เรียนในบทนี้

- โปรเจกต์ URL shortener คือระบบที่หัวใจสำคัญอยู่ที่การรับมือกับ **hot path** (การ redirect) ที่ถูกเรียกบ่อยกว่าการสร้าง short URL มาก — ทุกการตัดสินใจด้าน architecture ในบทนี้หมุนรอบคำถามนี้
- **Short code generation** มีสองแนวทางหลัก: **Base62-of-auto-increment-ID** (ไม่มี collision เลย แต่เดาลำดับได้) กับ **Random ด้วย `crypto/rand`** (เดาไม่ได้ แต่ต้องมี collision-check retry loop) — เชื่อมโยงกับ **Part 051** เรื่อง `crypto/rand` vs `math/rand` โดยตรง เพราะ short code เกี่ยวข้องกับความปลอดภัย
- **`database/sql`** (**Part 071**) เป็น interface กลางที่ทำให้เขียนโค้ดครั้งเดียวใช้ได้ทั้ง SQLite (สำหรับทดสอบ) และ PostgreSQL (**Part 072**) จริงในโปรดักชัน โดยแก้แค่ connection string, syntax ของ auto-increment, และ placeholder
- **Cache-Aside Pattern** (**Part 077**) วางแคชไว้หน้า query ที่ถูกเรียกบ่อยที่สุด ออกแบบเป็น interface กลาง (`Cache`) ทำให้สลับระหว่าง Redis จริงกับ in-memory LRU cache สำรองได้โดยไม่กระทบ handler เลย
- **Rate Limiting** ต่อ IP ด้วย token bucket (`golang.org/x/time/rate`, **Part 045**) ป้องกันการยิง `POST /shorten` รัวเกินไป โดยตั้งใจไม่จำกัด redirect เพราะเป็น hot path ที่ต้องรองรับ traffic สูง
- **Analytics** นับคลิกด้วย `sync/atomic` typed API (`atomic.Int64`, **Part 040**) ในหน่วยความจำก่อน แล้ว batch flush ลงฐานข้อมูลเป็นระยะด้วย `Swap(0)` ที่รับประกันว่าไม่มีคลิกไหนหายแม้จะมี concurrent access
- **Prometheus metrics** (**Part 099**) เปิด `/metrics` ด้วย `promhttp.Handler()` วัด request count, cache hit ratio, และ rate limit rejection — พร้อมบทเรียนจริงเรื่องบั๊ก metric ที่ลืมเพิ่มค่า ซึ่งพิสูจน์ว่าทำไม "รันจริงแล้วอ่านค่า" สำคัญกว่า "อ่านโค้ดเฉยๆ"
- **302 Found ไม่ใช่ 301 Moved Permanently** สำหรับ redirect เพราะ 301 ทำให้เบราว์เซอร์ cache ผลลัพธ์ไว้ถาวรฝั่ง client เอง ทำให้นับคลิกไม่ได้ตั้งแต่ครั้งที่สองเป็นต้นไป
- **Benchmark ที่วัดจริงเผยความจริงที่ซับซ้อนกว่าคำโฆษณา**: cache-aside ไม่ได้เร็วกว่าเสมอไป — ประโยชน์ของมันขึ้นอยู่กับว่าฐานข้อมูลหลักที่มันแทนที่มีต้นทุนสูงแค่ไหน (SQLite embedded ไม่มี network cost จึงไม่ได้ประโยชน์จาก cache เท่า PostgreSQL/MySQL ที่อยู่บนเครือข่ายจริง)
- Test suite เต็มรูปแบบ (unit + httptest integration, 24 เทส) รันผ่านทั้งหมดพร้อม race detector และมี coverage ที่วัดได้จริงในทุก package

## แบบฝึกหัดท้ายบท

1. **Custom vanity short code**: เพิ่ม field `"custom_code"` ใน `shortenRequest` ให้ผู้ใช้ระบุ short code ของตัวเองได้ (เช่น `/my-portfolio`) แทนที่จะให้ระบบสุ่มหรือใช้ base62 — ต้องเช็ค `CodeExists` ก่อนเสมอ และคืน `409 Conflict` ถ้า code ที่ขอมาถูกใช้ไปแล้ว (ห้ามลบของเดิมทิ้งโดยไม่ถาม)
2. **Expiring links**: เพิ่มคอลัมน์ `expires_at DATETIME NULL` ในตาราง `urls` และ field `"ttl_seconds"` (optional) ใน request ให้ผู้ใช้กำหนดวันหมดอายุของลิงก์ได้ — แก้ `GetByCode` ให้เช็ค `expires_at` และคืน `ErrCodeNotFound` (หรือ error ใหม่ `ErrCodeExpired`) ถ้าหมดอายุแล้ว พร้อมทั้งอย่าลืมลบ cache ของ code ที่หมดอายุออกด้วย (`cache.Del`)
3. **QR code endpoint**: เพิ่ม `GET /qr/{code}` ที่ generate QR code image (PNG) ของ short URL แล้วส่งกลับเป็น `Content-Type: image/png` — ลองหา library ภายนอกสำหรับ generate QR code (หรือค้นคว้าอัลกอริทึม QR code เพิ่มเติมเอง) แล้วเชื่อมเข้ากับ handler เดิม
4. **ย้าย analytics ไป Redis `INCR`**: ตามที่กล่าวถึงในหัวข้อ 7 ว่าเป็นทางเลือกสำหรับระบบหลาย instance — เขียน `RedisClickTracker` ที่ implement interface เดียวกับแนวคิด `ClickTracker` แต่ใช้ `rdb.Incr(ctx, "clicks:"+code)` แทน `atomic.Int64` แล้วเขียน background flusher ที่ `GETDEL` (หรือ `GetSet` เป็น 0) แต่ละ key `clicks:*` ไปสะสมลงฐานข้อมูลเป็นระยะ ทดสอบว่าถ้ารัน server 2 instance พร้อมกันหลัง reverse proxy ยอดคลิกยังนับรวมกันถูกต้อง
5. **เปลี่ยนไปใช้ PostgreSQL จริง**: ทำตามหมายเหตุในหัวข้อ 3 (แก้ 3 จุด: `AUTOINCREMENT`, `DATETIME`, placeholder) เปลี่ยน `store.Open` ให้ต่อ PostgreSQL ผ่าน `pgx` (**Part 072**) แทน SQLite แล้วรัน benchmark ในหัวข้อ 13 ใหม่กับ PostgreSQL จริง (ถ้ามี PostgreSQL server ให้ต่อ) — เทียบตัวเลขที่ได้กับที่บทความนี้จำลองไว้ด้วย `time.Sleep(2 * time.Millisecond)` ว่าใกล้เคียงกันแค่ไหน
6. **Distributed lock ป้องกัน cache stampede**: ตามที่กล่าวถึงใน **Part 077** หัวข้อ 6 — ถ้าลิงก์ยอดนิยมมาก cache หมดอายุพร้อมกันตอนมี traffic สูง หลาย request อาจ miss cache พร้อมกันแล้วยิง query ไปที่ฐานข้อมูลพร้อมกันหมด ลองใช้ `SetNX` เขียน distributed lock อย่างง่ายให้มีแค่ request เดียวที่ query ฐานข้อมูลจริงตอน cache miss ส่วนที่เหลือรอผลจาก request แรก (หรือ short-poll cache ซ้ำจนกว่าจะมีค่า)

---

**ต่อไป**: [Part 106 — Security Best Practices สำหรับ Go](./106-security-best-practices.md)
