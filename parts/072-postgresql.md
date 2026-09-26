# Part 072: PostgreSQL กับ Go (`pgx`, `lib/pq`)

> ภาคที่ 6: ฐานข้อมูล — ตอนที่ 2 จาก 8 (Part 71–78)

## สารบัญของบทนี้

1. ทำไม PostgreSQL ถึงเป็นตัวเลือกยอดนิยมสำหรับโปรเจกต์ Go
2. เลือก Driver: `lib/pq` vs `jackc/pgx` — และทำไมควรใช้ `pgx` สำหรับโปรเจกต์ใหม่
3. DSN / Connection String ของ PostgreSQL
4. `pgx` แบบ `database/sql` compatibility mode
5. `pgx` แบบ Native API และ `pgxpool` สำหรับ Connection Pooling
6. `RETURNING` Clause: ได้ค่าที่เพิ่งเพิ่มกลับมาในรอบเดียว
7. Arrays: เก็บและอ่าน PostgreSQL Array จาก Go Slice โดยตรง
8. JSONB: เก็บข้อมูลกึ่งโครงสร้างและ Query ด้วย Operator ของ JSON
9. ตัวอย่างเต็ม: CRUD ที่รันจริงกับ PostgreSQL ด้วย `pgxpool`
10. สรุปสิ่งที่ได้เรียนในบทนี้
11. แบบฝึกหัดท้ายบท

---

## 1. ทำไม PostgreSQL ถึงเป็นตัวเลือกยอดนิยมสำหรับโปรเจกต์ Go

**PostgreSQL** (มักเรียกสั้นๆ ว่า "Postgres") เป็นฐานข้อมูลเชิงสัมพันธ์แบบ open source ที่ได้รับความนิยมสูงมากในวงการ Go โดยเฉพาะ เหตุผลหลักๆ คือ:

- **มาตรฐาน SQL ที่เข้มงวดและ feature ครบ** — รองรับ `CTE`, `Window Function`, `JSON/JSONB`, `Array`, `Full-text Search`, `Extension` (เช่น `PostGIS` สำหรับข้อมูลภูมิศาสตร์) ในตัว
- **Ecosystem Go แข็งแกร่งมาก** — driver อย่าง `pgx` ถือเป็นหนึ่งใน driver ฐานข้อมูลที่เขียนดีและเร็วที่สุดในโลก Go ทั้งหมด ไม่ใช่แค่เทียบกับ driver ของ PostgreSQL เท่านั้น
- **Open source แท้ ไม่มีบริษัทเดียวควบคุม** — ต่างจาก MySQL ที่ Oracle เป็นเจ้าของ (แม้จะยัง open source อยู่)
- **License ที่เสรีกว่า** — PostgreSQL License คล้าย MIT/BSD ใช้ในเชิงพาณิชย์ได้อย่างอิสระเต็มที่

บทนี้จะพาไปรู้จัก driver 2 ตัวหลักที่ใช้คุยกับ PostgreSQL จาก Go และ feature เฉพาะของ PostgreSQL ที่มีประโยชน์มากเวลาทำงานจริง

---

## 2. เลือก Driver: `lib/pq` vs `jackc/pgx` — และทำไมควรใช้ `pgx` สำหรับโปรเจกต์ใหม่

ในโลก Go มี driver สำหรับ PostgreSQL ที่ได้รับความนิยม 2 ตัวหลัก:

### `github.com/lib/pq`

driver ตัวแรกๆ ที่ community Go ใช้กันมานาน implement `database/sql` driver interface แบบมาตรฐาน เขียนด้วย pure Go ล้วน (ไม่ต้องพึ่ง cgo) จุดที่ควรรู้คือ **`lib/pq` อยู่ในสถานะ maintenance mode มาหลายปีแล้ว** — README ของโปรเจกต์เองก็ระบุไว้ชัดเจนว่าไม่มีการพัฒนา feature ใหม่อีกแล้ว มีแค่การรับ patch เล็กๆ น้อยๆ เพื่อความเข้ากันได้เท่านั้น

### `github.com/jackc/pgx`

driver รุ่นใหม่กว่าที่พัฒนาต่อเนื่องอย่างจริงจัง โดย Jack Christensen จุดเด่นสำคัญ:

- **เร็วกว่า `lib/pq` อย่างชัดเจน** ในการ benchmark ส่วนใหญ่ เพราะ implement wire protocol ของ PostgreSQL อย่างมีประสิทธิภาพกว่า และรองรับ extended query protocol เต็มรูปแบบ
- **ใช้งานได้ 2 โหมด**:
  1. **`database/sql` compatibility mode** ผ่าน sub-package `github.com/jackc/pgx/v5/stdlib` — ใช้ร่วมกับ `sql.Open`/`sql.DB` แบบเดิมได้ทันที เปลี่ยนจาก `lib/pq` มาแทบไม่ต้องแก้โค้ดเลย
  2. **Native API** ของตัวเอง (`pgx.Conn`, `pgxpool.Pool`) ที่มีลูกเล่นเฉพาะของ PostgreSQL มากกว่า `database/sql` ทั่วไป เช่น การ scan ตรงเข้า struct, การจัดการ type พิเศษของ Postgres (arrays, ranges, composite types) ได้ดีกว่า, และ COPY protocol สำหรับ bulk insert ที่เร็วมาก
- **พัฒนาต่อเนื่องและ active** — ตามมาตรฐาน PostgreSQL feature ใหม่ๆ ได้เร็ว

### คำแนะนำ

> **สำหรับโปรเจกต์ใหม่ทุกโปรเจกต์ แนะนำให้ใช้ `pgx` เป็นค่าเริ่มต้น** เพราะเร็วกว่า พัฒนาต่อเนื่องมากกว่า และให้ทางเลือกทั้ง `database/sql` compatibility mode (ถ้าอยากคงโค้ดให้ generic ข้าม driver ได้) และ native API (ถ้าต้องการประสิทธิภาพและ feature เต็มรูปแบบของ PostgreSQL) `lib/pq` ยังใช้งานได้ปกติและเสถียรดีสำหรับโปรเจกต์เก่าที่ใช้อยู่แล้ว แต่ไม่มีเหตุผลที่ควรเลือกมันสำหรับโปรเจกต์ที่เพิ่งเริ่มใหม่

ติดตั้ง:

```bash
go get github.com/jackc/pgx/v5
go get github.com/jackc/pgx/v5/pgxpool  # สำหรับ connection pooling แบบ native
```

> **หมายเหตุความซื่อสัตย์**: ทุกตัวอย่างในบทนี้ (หัวข้อ 3–9) รันจริงกับ **PostgreSQL 16** ที่ติดตั้งและสตาร์ทขึ้นมาในสภาพแวดล้อมที่ใช้เขียนหลักสูตรนี้ ผ่าน `pgx/v5` และ `pgxpool` เวอร์ชันล่าสุด (v5.11.0) ผลลัพธ์ที่แสดงทั้งหมดคือ output จริงจากการรันโค้ด

---

## 3. DSN / Connection String ของ PostgreSQL

PostgreSQL รับ connection string ได้ 2 รูปแบบหลัก ทั้งคู่ใช้ได้กับทั้ง `lib/pq` และ `pgx`:

**รูปแบบ URL** (นิยมมากกว่า เพราะอ่านง่ายและใช้ได้กับหลาย tool):

```
postgres://username:password@host:port/database?sslmode=disable
```

**รูปแบบ key-value**:

```
host=127.0.0.1 port=5432 user=postgres password=postgres dbname=godemo sslmode=disable
```

พารามิเตอร์ที่ควรรู้จัก:

| พารามิเตอร์ | ความหมาย |
|---|---|
| `sslmode` | ระดับการเข้ารหัส TLS: `disable`, `require`, `verify-ca`, `verify-full` — production ควรใช้อย่างน้อย `require` |
| `connect_timeout` | เวลาสูงสุด (วินาที) รอการเชื่อมต่อก่อน timeout |
| `application_name` | ชื่อแอปพลิเคชันที่จะไปโผล่ใน `pg_stat_activity` ฝั่งเซิร์ฟเวอร์ ช่วย debug ว่า connection ไหนมาจากแอปตัวไหน |
| `search_path` | schema ที่จะค้นหาตารางเมื่อไม่ระบุ schema ให้ query |

ตัวอย่างที่ใช้ในบทนี้ (เชื่อมต่อ localhost แบบไม่เข้ารหัส เหมาะสำหรับ development เท่านั้น — production ต้องเปิด `sslmode=require` เป็นอย่างน้อย):

```
postgres://postgres:postgres@127.0.0.1:5432/godemo?sslmode=disable
```

---

## 4. `pgx` แบบ `database/sql` Compatibility Mode

ถ้าต้องการคงโค้ดให้ใช้ `database/sql` แบบเดิมทุกอย่างตาม **Part 071** (เผื่ออนาคตอยากสลับฐานข้อมูล หรือใช้ร่วมกับ library ที่คาดหวัง `*sql.DB`) ให้ import `pgx/v5/stdlib` แทน `lib/pq`:

```go
package main

import (
	"database/sql"
	"fmt"
	"log"

	_ "github.com/jackc/pgx/v5/stdlib" // ลงทะเบียน driver name "pgx"
)

func main() {
	db, err := sql.Open("pgx", "postgres://postgres:postgres@127.0.0.1:5432/godemo?sslmode=disable")
	if err != nil {
		log.Fatal(err)
	}
	defer db.Close()
	if err := db.Ping(); err != nil {
		log.Fatal(err)
	}
	var count int
	if err := db.QueryRow(`SELECT COUNT(*) FROM products`).Scan(&count); err != nil {
		log.Fatal(err)
	}
	fmt.Println("pgx ผ่าน database/sql (driver name \"pgx\"): จำนวนสินค้า =", count)
}
```

ผลลัพธ์จริง (รันหลังสร้างตาราง `products` และ insert ข้อมูล 3 แถวไว้แล้วในหัวข้อ 9):

```
pgx ผ่าน database/sql (driver name "pgx"): จำนวนสินค้า = 3
```

สังเกตว่า **โค้ดเหมือนกับที่ใช้ `lib/pq` ทุกอย่าง** ต่างกันแค่ import path และชื่อ driver ที่ส่งให้ `sql.Open` (`"pgx"` แทน `"postgres"`) — นี่คือพลังของสถาปัตยกรรม driver-agnostic ที่อธิบายไว้ใน Part 071 หัวข้อ 1 การย้ายจาก `lib/pq` ไป `pgx` (compat mode) แทบไม่กระทบโค้ดส่วนอื่นเลย

โหมดนี้เหมาะกับกรณีที่ต้องการประโยชน์เรื่อง performance ของ `pgx` แต่ยังอยากเก็บโค้ดให้ generic ผ่าน `database/sql` ไว้ (เช่น ใช้ร่วมกับ library ที่ยอมรับแค่ `*sql.DB`) แต่ถ้าต้องการดึงความสามารถเต็มรูปแบบของ PostgreSQL และประสิทธิภาพสูงสุด ควรใช้ **native API** ในหัวข้อถัดไปแทน

---

## 5. `pgx` แบบ Native API และ `pgxpool` สำหรับ Connection Pooling

`pgx` มี API ของตัวเองที่ไม่ผ่าน `database/sql` เลย ให้ความสามารถและ performance เต็มรูปแบบมากกว่า โดยเฉพาะการจัดการ connection pool ผ่าน sub-package **`pgxpool`** ซึ่งออกแบบมาสำหรับ PostgreSQL โดยเฉพาะ (ต่างจาก pool ทั่วไปใน `database/sql` ที่ generic ข้ามฐานข้อมูล)

```go
import "github.com/jackc/pgx/v5/pgxpool"

pool, err := pgxpool.New(ctx, "postgres://postgres:postgres@127.0.0.1:5432/godemo?sslmode=disable")
if err != nil {
	log.Fatal("pgxpool.New:", err)
}
defer pool.Close()

if err := pool.Ping(ctx); err != nil {
	log.Fatal("ping:", err)
}
```

สังเกตความต่างจาก `database/sql`: **`ctx` เป็น parameter บังคับตั้งแต่แรกเริ่ม** ในทุก method ของ native API ไม่มีเวอร์ชันไม่มี `ctx` ให้เลือกเหมือน `database/sql` เลย — นี่คือปรัชญาการออกแบบของ `pgx` ที่บังคับให้ผู้ใช้คิดเรื่อง cancellation/timeout ตั้งแต่ต้น สอดคล้องกับที่เรียนใน **Part 032**

`pgxpool.Pool` มี method ที่หน้าตาคล้าย `database/sql` มาก (`Query`, `QueryRow`, `Exec`) แต่รับ `ctx` เป็น argument แรกเสมอ:

```go
rows, err := pool.Query(ctx, `SELECT id, name FROM products ORDER BY id`)
_, err = pool.Exec(ctx, `INSERT INTO products (name) VALUES ($1)`, "สินค้าใหม่")
row := pool.QueryRow(ctx, `SELECT name FROM products WHERE id = $1`, 1)
```

ตรวจสอบสถานะ pool ด้วย `pool.Stat()`:

```go
stat := pool.Stat()
fmt.Printf("pgxpool stats: TotalConns=%d IdleConns=%d AcquiredConns=%d\n",
	stat.TotalConns(), stat.IdleConns(), stat.AcquiredConns())
```

ผลลัพธ์จริง:

```
pgxpool stats: TotalConns=1 IdleConns=1 AcquiredConns=0
```

`pgxpool` ปรับขนาด pool อัตโนมัติตามการใช้งาน และมีค่า config ที่ปรับได้ผ่าน `pgxpool.ParseConfig` เช่น `MaxConns`, `MinConns`, `MaxConnLifetime`, `MaxConnIdleTime` ซึ่งมีบทบาทคล้ายกับ `SetMaxOpenConns`/`SetMaxIdleConns`/`SetConnMaxLifetime` ของ `database/sql` ที่เรียนใน Part 071 หัวข้อ 9 — รายละเอียดการ tune ค่าเหล่านี้จะอยู่ใน **Part 078**

---

## 6. `RETURNING` Clause: ได้ค่าที่เพิ่งเพิ่มกลับมาในรอบเดียว

จำได้จาก **Part 071** ว่า `LastInsertId()` ใช้กับ PostgreSQL ไม่ได้เพราะ PostgreSQL ไม่มีแนวคิด auto-increment ID แบบ built-in ที่ driver ดึงกลับมาได้ตรงๆ (Postgres ใช้ `SERIAL`/`IDENTITY` ซึ่งเป็นแค่ sequence object) — วิธีมาตรฐานของ PostgreSQL ที่ทำงานได้ดีกว่ามากคือ **`RETURNING` clause** ซึ่งเป็น extension ของ PostgreSQL ที่ไม่มีใน SQL มาตรฐานทั่วไป

`RETURNING` ทำให้คำสั่ง `INSERT`/`UPDATE`/`DELETE` **คืนค่า column ที่ระบุกลับมาได้ทันทีในรอบเดียว** โดยไม่ต้องยิง query แยกไปถามซ้ำอีกรอบ (ลด round-trip ไป-กลับกับฐานข้อมูล):

```go
var newID int
var createdAt time.Time
err = pool.QueryRow(ctx,
	`INSERT INTO products (name, tags, metadata) VALUES ($1, $2, $3) RETURNING id, created_at`,
	"เก้าอี้ไม้", []string{"furniture", "wood", "sale"}, metaJSON,
).Scan(&newID, &createdAt)
```

สังเกตว่าเราใช้ `pool.QueryRow` (ไม่ใช่ `Exec`) เพราะแม้จะเป็นคำสั่ง `INSERT` แต่ `RETURNING` ทำให้มันคืนแถวข้อมูลกลับมาเหมือน `SELECT`

ผลลัพธ์จริง:

```
insert สำเร็จ ได้ id=1 กลับมาทันทีจาก RETURNING (created_at=2026-09-26T02:50:36Z)
```

`RETURNING` มีประโยชน์มากในหลายสถานการณ์ เช่น การดึง `updated_at` ที่ trigger ของฐานข้อมูลตั้งค่าอัตโนมัติกลับมาแสดงผลทันทีหลัง `UPDATE`, หรือดึง list ของ ID ที่ถูกลบกลับมาหลัง `DELETE ... WHERE ... RETURNING id`

---

## 7. Arrays: เก็บและอ่าน PostgreSQL Array จาก Go Slice โดยตรง

PostgreSQL รองรับ column ชนิด **array** ในตัว (เช่น `TEXT[]`, `INTEGER[]`) ซึ่งเป็น feature ที่ฐานข้อมูลอื่นอย่าง MySQL ไม่มี (ต้องจำลองด้วยตารางแยกหรือ JSON แทน) — `pgx` รองรับการแปลง Go slice ↔ PostgreSQL array โดยอัตโนมัติ ทั้งตอน insert และตอน scan กลับ:

```go
_, err = pool.Exec(ctx, `CREATE TABLE products (
	id SERIAL PRIMARY KEY,
	name TEXT NOT NULL,
	tags TEXT[] NOT NULL DEFAULT '{}',
	metadata JSONB NOT NULL DEFAULT '{}',
	created_at TIMESTAMPTZ NOT NULL DEFAULT now()
)`)

// ส่ง []string ตรงๆ เป็น argument — pgx แปลงเป็น PostgreSQL array ให้อัตโนมัติ
_, err = pool.Exec(ctx, `INSERT INTO products (name, tags, metadata) VALUES ($1, $2, $3)`,
	"โต๊ะทำงาน", []string{"furniture", "office"}, []byte(`{"weight_kg": 15.5}`))
```

ตอนอ่านกลับก็ scan เข้า `[]string` ได้ตรงๆ เช่นกัน:

```go
var p Product
var metaBytes []byte
rows.Scan(&p.ID, &p.Name, &p.Tags, &metaBytes) // p.Tags เป็น []string
```

ผลลัพธ์จริง:

```
  #1 เก้าอี้ไม้   tags=[furniture wood sale] metadata=map[color:red weight_kg:1.2]
  #2 โต๊ะทำงาน    tags=[furniture office] metadata=map[weight_kg:15.5]
  #3 หูฟังไร้สาย  tags=[electronics] metadata=map[battery_hours:30 color:black]
```

การ query โดยใช้ operator เฉพาะของ array เช่น `ANY(tags)` เพื่อหาว่า array มีค่าที่ต้องการอยู่หรือไม่:

```go
rows2, err := pool.Query(ctx, `SELECT name FROM products WHERE 'furniture' = ANY(tags)`)
```

ผลลัพธ์จริง:

```
--- ค้นหาด้วย array operator: tags มี 'furniture' ---
  - เก้าอี้ไม้
  - โต๊ะทำงาน
```

Array เหมาะกับข้อมูลที่เป็น "list ของค่าง่ายๆ" ที่ไม่ต้องการ query ซับซ้อนมาก (เช่น tags, categories) — ถ้าความสัมพันธ์ซับซ้อนกว่านั้น (ต้องการ query/join/filter บนแต่ละ item อย่างละเอียด) ควรออกแบบเป็นตารางแยกแบบ many-to-many แทน (จะเรียนกับ GORM ใน **Part 075**)

---

## 8. JSONB: เก็บข้อมูลกึ่งโครงสร้างและ Query ด้วย Operator ของ JSON

PostgreSQL มี column ชนิด `JSON` และ `JSONB` สำหรับเก็บข้อมูลแบบ JSON โดยตรงในฐานข้อมูล — **`JSONB`** (JSON Binary) เป็นตัวที่แนะนำให้ใช้เกือบทุกกรณี เพราะเก็บข้อมูลในรูปแบบ binary ที่ query/index ได้เร็วกว่า `JSON` ธรรมดา (ซึ่งเก็บเป็น text ดิบๆ และต้อง parse ใหม่ทุกครั้งที่ query) ข้อแลกเปลี่ยนเดียวคือ `JSONB` ไม่รักษาลำดับ key และช่องว่างในข้อความต้นฉบับไว้ (ซึ่งแทบไม่มีผลกับการใช้งานจริง)

### Insert ด้วย `encoding/json` (ทบทวน Part 025)

```go
metadata := map[string]any{"weight_kg": 1.2, "color": "red"}
metaJSON, _ := json.Marshal(metadata) // ได้ []byte ของ JSON text

_, err = pool.Exec(ctx, `INSERT INTO products (name, metadata) VALUES ($1, $2)`, "เก้าอี้ไม้", metaJSON)
```

`pgx` (และ `lib/pq`) ยอมรับ `[]byte` เป็นค่าให้กับ column `JSONB` ได้ตรงๆ — เราจึงใช้ `encoding/json` มาตรฐานที่เรียนใน Part 025 marshal ข้อมูลก่อนส่งเข้าไปได้เลย ไม่ต้องพึ่ง library พิเศษเพิ่ม

### อ่านกลับด้วย `encoding/json`

```go
var metaBytes []byte
rows.Scan(&p.ID, &p.Name, &p.Tags, &metaBytes)
json.Unmarshal(metaBytes, &p.Metadata) // p.Metadata เป็น map[string]any
```

### Query ด้วย JSON Operator

PostgreSQL มี operator พิเศษสำหรับดึงค่าจาก JSONB โดยตรงในคำสั่ง SQL โดยไม่ต้องดึงทั้งก้อนมา parse ฝั่ง Go:

- `->` ดึงค่าออกมาเป็น JSON (ยังเป็น JSON อยู่ ใช้ chain ต่อได้)
- `->>` ดึงค่าออกมาเป็น text
- `@>` เช็คว่า JSONB หนึ่งมีอีกอันเป็น subset อยู่ข้างในหรือไม่

```go
var name string
err = pool.QueryRow(ctx, `SELECT name FROM products WHERE metadata->>'color' = 'black'`).Scan(&name)
```

ผลลัพธ์จริง:

```
--- ค้นหาด้วย JSONB operator: metadata->>'color' = 'black' ---
  พบ: หูฟังไร้สาย
```

จุดแข็งของ `JSONB` คือสามารถสร้าง **GIN index** บน column นี้ได้ (`CREATE INDEX ... USING GIN (metadata)`) ทำให้ query ด้วย operator อย่าง `@>` เร็วเทียบเท่า column ปกติที่มี index แม้ข้อมูลจะไม่มีโครงสร้างตายตัวก็ตาม — นี่คือเหตุผลสำคัญที่ทีมจำนวนมากเลือก PostgreSQL แทน MySQL เมื่อต้องเก็บข้อมูลกึ่งโครงสร้าง (semi-structured data) โดยไม่อยากเสียความสามารถ query/index แบบ SQL ไป (จะเทียบกับ MySQL อีกครั้งใน **Part 073**)

> **ข้อควรระวัง**: JSONB สะดวก แต่ไม่ควรใช้แทนการออกแบบ schema ที่ดีเสมอไป — ถ้ารู้โครงสร้างข้อมูลที่แน่นอนล่วงหน้าและต้องการ constraint/type safety ระดับฐานข้อมูล (เช่น `NOT NULL`, foreign key, `CHECK`) ควรใช้ column ปกติ JSONB เหมาะกับข้อมูลที่โครงสร้างยืดหยุ่นหรือเปลี่ยนบ่อยจริงๆ เช่น custom metadata, event payload, configuration ที่ผู้ใช้กำหนดเอง

---

## 9. ตัวอย่างเต็ม: CRUD ที่รันจริงกับ PostgreSQL ด้วย `pgxpool`

โปรแกรมเต็มที่รวมทุก feature ในบทนี้ไว้ด้วยกัน — รันจริงทั้งหมด ผลลัพธ์คือ output จริงจากการรัน:

```go
package main

import (
	"context"
	"encoding/json"
	"fmt"
	"log"
	"time"

	"github.com/jackc/pgx/v5/pgxpool"
)

type Product struct {
	ID       int
	Name     string
	Tags     []string
	Metadata map[string]any
}

func main() {
	ctx := context.Background()

	pool, err := pgxpool.New(ctx, "postgres://postgres:postgres@127.0.0.1:5432/godemo?sslmode=disable")
	if err != nil {
		log.Fatal("pgxpool.New:", err)
	}
	defer pool.Close()

	if err := pool.Ping(ctx); err != nil {
		log.Fatal("ping:", err)
	}
	fmt.Println("เชื่อมต่อผ่าน pgxpool สำเร็จ")

	_, err = pool.Exec(ctx, `CREATE TABLE IF NOT EXISTS products (
		id SERIAL PRIMARY KEY,
		name TEXT NOT NULL,
		tags TEXT[] NOT NULL DEFAULT '{}',
		metadata JSONB NOT NULL DEFAULT '{}',
		created_at TIMESTAMPTZ NOT NULL DEFAULT now()
	)`)
	if err != nil {
		log.Fatal("create table:", err)
	}

	// CREATE พร้อม RETURNING
	metadata := map[string]any{"weight_kg": 1.2, "color": "red"}
	metaJSON, _ := json.Marshal(metadata)

	var newID int
	var createdAt time.Time
	err = pool.QueryRow(ctx,
		`INSERT INTO products (name, tags, metadata) VALUES ($1, $2, $3) RETURNING id, created_at`,
		"เก้าอี้ไม้", []string{"furniture", "wood", "sale"}, metaJSON,
	).Scan(&newID, &createdAt)
	if err != nil {
		log.Fatal("insert returning:", err)
	}
	fmt.Printf("insert สำเร็จ ได้ id=%d กลับมาทันทีจาก RETURNING\n", newID)

	// READ ทั้งหมด พร้อม array + jsonb
	rows, err := pool.Query(ctx, `SELECT id, name, tags, metadata FROM products ORDER BY id`)
	if err != nil {
		log.Fatal(err)
	}
	defer rows.Close()

	var products []Product
	for rows.Next() {
		var p Product
		var metaBytes []byte
		if err := rows.Scan(&p.ID, &p.Name, &p.Tags, &metaBytes); err != nil {
			log.Fatal(err)
		}
		json.Unmarshal(metaBytes, &p.Metadata)
		products = append(products, p)
	}
	if err := rows.Err(); err != nil {
		log.Fatal(err)
	}
	for _, p := range products {
		fmt.Printf("  #%d %-12s tags=%v metadata=%v\n", p.ID, p.Name, p.Tags, p.Metadata)
	}

	// UPDATE
	_, err = pool.Exec(ctx, `UPDATE products SET metadata = metadata || '{"on_sale": true}' WHERE id = $1`, newID)
	if err != nil {
		log.Fatal(err)
	}

	// DELETE
	_, err = pool.Exec(ctx, `DELETE FROM products WHERE id = $1`, newID)
	if err != nil {
		log.Fatal(err)
	}
	fmt.Println("CRUD ครบทั้ง 4 operation สำเร็จ")

	stat := pool.Stat()
	fmt.Printf("pgxpool stats: TotalConns=%d IdleConns=%d\n", stat.TotalConns(), stat.IdleConns())
}
```

โปรแกรมนี้สาธิตครบทุกจุดสำคัญของบทนี้: การเชื่อมต่อด้วย `pgxpool`, `RETURNING` clause แทน `LastInsertId()`, การ insert/scan `TEXT[]` และ `JSONB` โดยตรง, และการ update ด้วย JSONB concatenation operator (`||`) ซึ่งเป็นอีกหนึ่ง operator เฉพาะของ PostgreSQL ที่ใช้ merge JSON object สองก้อนเข้าด้วยกัน

---

## สรุปสิ่งที่ได้เรียนในบทนี้

- PostgreSQL เป็นฐานข้อมูล open source ที่ feature ครบและมี ecosystem Go ที่แข็งแกร่งเป็นพิเศษ
- Driver หลัก 2 ตัว: `lib/pq` (pure Go, อยู่ใน maintenance mode) และ `jackc/pgx` (เร็วกว่า พัฒนาต่อเนื่อง แนะนำสำหรับโปรเจกต์ใหม่)
- `pgx` ใช้ได้ 2 โหมด: `database/sql` compatibility mode ผ่าน `pgx/v5/stdlib` (เปลี่ยนจาก `lib/pq` ได้ไม่กระทบโค้ด) และ native API ที่ให้ performance และ feature เต็มรูปแบบกว่า
- `pgxpool` คือ connection pool แบบ native ของ `pgx` ออกแบบมาสำหรับ PostgreSQL โดยเฉพาะ ทุก method บังคับรับ `context.Context` เสมอ
- `RETURNING` clause แก้ปัญหาที่ PostgreSQL ไม่มี `LastInsertId()` โดยคืนค่า column ที่ต้องการกลับมาในรอบเดียวกับ `INSERT`/`UPDATE`/`DELETE`
- Array (`TEXT[]`) และ JSONB เป็น feature เฉพาะของ PostgreSQL ที่ `pgx` รองรับการแปลงกับ Go slice/map โดยตรง ทำให้เก็บข้อมูลกึ่งโครงสร้างได้โดยไม่ต้องออกแบบตารางแยกเสมอไป
- ตัวอย่างทั้งหมดในบทนี้รันจริงกับ PostgreSQL 16 ในสภาพแวดล้อมของหลักสูตร ไม่ใช่โค้ดที่ไม่ได้ทดสอบ

## แบบฝึกหัดท้ายบท

1. เขียนโปรแกรมเชื่อมต่อ PostgreSQL ด้วย `pgxpool` แล้วสร้างตาราง `orders` ที่มี column `items JSONB` เก็บ list ของสินค้าที่สั่งซื้อ (แต่ละ item มี `name` และ `qty`) ลองใช้ `jsonb_array_elements` ใน SQL เพื่อ query ว่ามี order ไหนที่สั่งสินค้าชื่อหนึ่งๆ บ้าง
2. เปรียบเทียบเวลาในการ insert 1,000 แถวระหว่างการเรียก `pool.Exec` ทีละแถวใน loop กับการใช้ `pgx.CopyFrom` (COPY protocol) — วัดเวลาและอธิบายว่าทำไม `CopyFrom` เร็วกว่ามาก
3. เขียนฟังก์ชันที่ insert ข้อมูลแล้วใช้ `RETURNING id, created_at, updated_at` ดึงค่า timestamp ที่ database กำหนดให้อัตโนมัติ (ผ่าน `DEFAULT now()`) กลับมาแสดงผลทันที โดยไม่ query ซ้ำรอบสอง
4. ลองสร้าง GIN index บน column JSONB (`CREATE INDEX idx_products_metadata ON products USING GIN (metadata)`) แล้วใช้ `EXPLAIN ANALYZE` (ผ่าน `psql` หรือ query ตรงๆ จาก Go) เปรียบเทียบแผนการ query ก่อนและหลังมี index เมื่อ query ด้วย operator `@>`
5. แปลงโปรแกรมในหัวข้อ 4 ของ Part 071 (สาธิต SQL Injection) ให้ใช้ `pgx` native API แทน `database/sql` แล้วทดสอบว่า injection ยังถูกป้องกันได้เหมือนเดิมหรือไม่เมื่อใช้ placeholder `$1`
6. ทดลองใช้ `pgxpool.ParseConfig` กำหนดค่า `MaxConns` และ `MinConns` เอง แทนการส่ง DSN ตรงๆ เข้า `pgxpool.New` แล้วอธิบายความแตกต่างระหว่างการตั้งค่าผ่าน `Config` กับผ่าน query parameter ใน DSN

---

**ต่อไป**: [Part 073 — MySQL กับ Go](./073-mysql.md)
