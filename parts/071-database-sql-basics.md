# Part 071: `database/sql` พื้นฐาน

> ภาคที่ 6: ฐานข้อมูล — ตอนที่ 1 จาก 8 (Part 71–78)

ยินดีต้อนรับสู่ **ภาคที่ 6: ฐานข้อมูล** — หลังจากภาคที่ 5 เราเรียนรู้การสร้าง Web Server และ REST API ด้วย Go ไปแล้ว แต่ API ที่มีประโยชน์จริงแทบทั้งหมดต้องเก็บและดึงข้อมูลจากฐานข้อมูล ภาคนี้จะพาไปรู้จักวิธีคุยกับฐานข้อมูลจาก Go ตั้งแต่กลไกพื้นฐานที่สุดในตอนนี้ ไปจนถึง PostgreSQL, MySQL, ORM อย่าง GORM, MongoDB, Redis และปิดท้ายด้วย Transaction/Connection Pooling แบบเจาะลึก

> **หมายเหตุเรื่องความซื่อสัตย์ในการสาธิต**: ทุกตัวอย่างโค้ดใน Part 071–075 (บทนี้ถึง GORM ขั้นสูง) **ถูกรันจริง**กับฐานข้อมูลจริงในสภาพแวดล้อมที่ใช้เขียนหลักสูตรนี้ — PostgreSQL 16 (ผ่าน `service postgresql start` แล้วสร้างฐานข้อมูล) และ MariaDB 10.11 (ซึ่งพูดโปรโตคอลเดียวกับ MySQL และใช้ driver `go-sql-driver/mysql` ตัวเดียวกันได้ทุกประการ) ผลลัพธ์ที่แสดงในบทเรียนคือ output จริงจากการรันโค้ด ไม่ใช่ค่าที่แต่งขึ้น ส่วนไหนที่เป็น syntax เฉพาะของฐานข้อมูลใดฐานข้อมูลหนึ่งที่ไม่ได้รันจริงในสภาพแวดล้อมนี้ จะมีป้ายกำกับไว้ชัดเจนว่าเป็น "ข้อมูลอ้างอิง" (verified จาก documentation/source code ของ driver แทน)

## สารบัญของบทนี้

1. `database/sql` คืออะไร และสถาปัตยกรรมแบบ Driver-Agnostic
2. ติดตั้ง Driver และเหตุผลของ Blank Import (`_ "..."`)
3. `sql.Open` ไม่ได้เชื่อมต่อจริง — และทำไมต้อง `db.Ping()`
4. `db.Exec`: คำสั่งที่ไม่คืนแถวข้อมูล (INSERT/UPDATE/DELETE/CREATE)
5. `db.Query` และ `db.QueryRow`: ดึงข้อมูลหลายแถว vs แถวเดียว
6. Scan ข้อมูลเข้า Struct, `defer rows.Close()`, และ `rows.Err()`
7. Prepared Statements และ SQL Injection: เรื่องความปลอดภัยที่พลาดไม่ได้
8. `QueryContext` / `ExecContext` กับ `context.Context`
9. Connection Pool เบื้องต้น: `SetMaxOpenConns`, `SetMaxIdleConns`, `SetConnMaxLifetime`
10. ตัวอย่างเต็ม: CRUD ที่รันจริงกับ PostgreSQL
11. สรุปสิ่งที่ได้เรียนในบทนี้
12. แบบฝึกหัดท้ายบท

---

## 1. `database/sql` คืออะไร และสถาปัตยกรรมแบบ Driver-Agnostic

Go มี package มาตรฐานชื่อ **`database/sql`** สำหรับคุยกับฐานข้อมูลเชิงสัมพันธ์ (relational database) แบบ SQL ทุกยี่ห้อ — PostgreSQL, MySQL, SQLite, SQL Server, Oracle ฯลฯ จุดที่น่าสนใจที่สุดคือ `database/sql` **ไม่ได้รู้จักฐานข้อมูลตัวใดตัวหนึ่งเป็นการเฉพาะเลย** มันเป็นแค่ **interface กลาง** (generic interface) ที่นิยามพฤติกรรมมาตรฐาน เช่น "เปิดการเชื่อมต่อ", "รันคำสั่ง SQL", "ดึงแถวข้อมูลกลับมา" — ส่วนการคุยกับฐานข้อมูลจริงๆ ในระดับ wire protocol (เช่น PostgreSQL wire protocol หรือ MySQL protocol) ถูกผลักไปให้ **driver** แยกต่างหากที่ implement interface เหล่านี้

สถาปัตยกรรมนี้คล้ายกับหลักการ **small interfaces + implementation แยกจากกัน** ที่เราเรียนไปใน **Part 013–014 (Interfaces)** — โค้ดที่เขียนโดยใช้ `database/sql` (`db.Query`, `db.Exec` ฯลฯ) จะเหมือนกันทุกตัวอักษรไม่ว่าจะใช้ฐานข้อมูลอะไรอยู่เบื้องหลัง สิ่งที่เปลี่ยนมีแค่ 2 อย่าง:

1. **Driver ที่ import** (เช่น `github.com/lib/pq` สำหรับ PostgreSQL, `github.com/go-sql-driver/mysql` สำหรับ MySQL)
2. **Connection string / DSN** (Data Source Name) ที่ส่งให้ `sql.Open`

ข้อดีมหาศาลของสถาปัตยกรรมนี้คือ ถ้าวันหนึ่งต้องย้ายจาก MySQL ไป PostgreSQL โค้ดที่เรียก query ส่วนใหญ่ (ยกเว้น SQL syntax เฉพาะทางบางจุด) แทบไม่ต้องแก้เลย เพราะ `*sql.DB`, `*sql.Rows`, `*sql.Row`, `*sql.Stmt` เป็น type เดียวกันหมด

```
Application Code (Go)
        │
        ▼
  database/sql  (interface กลาง — ไม่รู้จัก Postgres/MySQL)
        │
        ▼
  driver.Driver (interface ที่ driver ต้อง implement)
        │
   ┌────┴─────┬──────────────┬───────────────┐
   ▼          ▼              ▼               ▼
 lib/pq   go-sql-driver   modernc.org/    ... driver อื่นๆ
(Postgres)   /mysql        sqlite
```

บทนี้จะโฟกัสที่กลไก `database/sql` ล้วนๆ ที่ใช้ได้กับทุกฐานข้อมูล ส่วน syntax และลูกเล่นเฉพาะของ PostgreSQL/MySQL จะเรียนเจาะลึกใน **Part 072** และ **Part 073**

---

## 2. ติดตั้ง Driver และเหตุผลของ Blank Import (`_ "..."`)

การจะใช้ `database/sql` ได้ ต้อง `go get` driver ของฐานข้อมูลที่จะใช้มาด้วยเสมอ ตัวอย่างการติดตั้ง driver ของ PostgreSQL ที่ใช้สาธิตในบทนี้:

```bash
go get github.com/lib/pq
```

จากนั้น import แบบพิเศษที่เรียกว่า **blank import**:

```go
import (
	"database/sql"

	_ "github.com/lib/pq" // blank import — ใช้เพื่อ side effect เท่านั้น
)
```

ทำไมต้องใช้ `_` นำหน้า? ทบทวนจาก **Part 001** ว่า Go บังคับให้ import ที่ไม่ได้ใช้งานเป็น compile error — แต่โค้ดของเราจะไม่เรียกฟังก์ชันใดๆ จาก package `pq` โดยตรงเลยสักครั้ง (เราเรียกผ่าน `database/sql` ทั้งหมด) ทุก driver ที่ดีจะมีฟังก์ชัน `init()` (ทบทวน **Part 002/018**) ที่ลงทะเบียนตัวเองเข้ากับ `database/sql` โดยอัตโนมัติตอนโปรแกรมเริ่มทำงาน ผ่านฟังก์ชัน `sql.Register`:

```go
// นี่คือสิ่งที่เกิดขึ้นภายใน package lib/pq (โดยประมาณ)
func init() {
	sql.Register("postgres", &Driver{})
}
```

`_ "github.com/lib/pq"` จึงบอก Go compiler ว่า "ฉันตั้งใจ import package นี้เพื่อให้ `init()` ของมันทำงาน แม้จะไม่เรียกใช้ function ใดจากมันตรงๆ เลยก็ตาม" — ถ้าลืม blank import ตัวนี้ โปรแกรมจะ compile ผ่าน แต่จะ panic ตอนรันด้วย error ประมาณ `sql: unknown driver "postgres" (forgotten import?)`

---

## 3. `sql.Open` ไม่ได้เชื่อมต่อจริง — และทำไมต้อง `db.Ping()`

จุดที่มือใหม่เข้าใจผิดบ่อยที่สุดคือ **`sql.Open` ไม่ได้เปิดการเชื่อมต่อไปยังฐานข้อมูลทันที**

```go
func Open(driverName, dataSourceName string) (*DB, error)
```

`sql.Open` แค่ **ตรวจสอบรูปแบบของ DSN ให้ถูกต้อง** และคืนค่า `*sql.DB` ที่เป็นตัวแทนของ **connection pool** (จะเรียกลึกใน หัวข้อ 9 และเต็มรูปแบบใน **Part 078**) — การเชื่อมต่อจริงไปยัง server จะเกิดขึ้น**แบบ lazy** คือรอจนกว่าจะมีการเรียก query/exec ครั้งแรกเท่านั้น พูดอีกแบบคือ `sql.Open` ที่ error แทบไม่มีทางเกิดจากปัญหาเชื่อมต่อฐานข้อมูล (เช่น server ไม่ทำงาน, password ผิด) เลย มันจะ error แค่กรณี DSN ผิดรูปแบบ หรือไม่รู้จัก driver name เท่านั้น

เพราะฉะนั้น การเช็คว่าเชื่อมต่อฐานข้อมูลได้จริงหรือไม่ **ต้องเรียก `db.Ping()` เองเสมอ**:

```go
db, err := sql.Open("postgres", dsn)
if err != nil {
	log.Fatal("sql.Open error:", err) // มักไม่เกิด เว้นแต่ DSN ผิดรูปแบบ
}
defer db.Close()

if err := db.Ping(); err != nil {
	log.Fatal("เชื่อมต่อฐานข้อมูลไม่ได้:", err) // นี่คือจุดที่เจอ error จริงถ้า server ล่ม/password ผิด
}
fmt.Println("เชื่อมต่อฐานข้อมูลสำเร็จ")
```

รันจริงกับ PostgreSQL 16 ที่ localhost (สร้างฐานข้อมูลชื่อ `godemo` ไว้ล่วงหน้า):

```
เชื่อมต่อฐานข้อมูลสำเร็จ (Ping OK)
```

> **แนวทางปฏิบัติ**: เกือบทุกโปรเจกต์จริงควรเรียก `db.Ping()` (หรือ `db.PingContext(ctx)`) ทันทีหลัง `sql.Open` ตอน startup ของแอปพลิเคชัน เพื่อ fail fast — ถ้าฐานข้อมูลเชื่อมต่อไม่ได้ อยากให้โปรแกรม crash ทันทีตอนเริ่มต้น ดีกว่าไปพังตอนมี request เข้ามาแล้วหาสาเหตุยาก

`*sql.DB` ไม่ใช่ connection เดี่ยวๆ แต่เป็น **pool ของ connection หลายเส้น** ที่ปลอดภัยต่อการใช้งานพร้อมกันจากหลาย goroutine (safe for concurrent use) — ปกติทั้งแอปพลิเคชันควรมี `*sql.DB` แค่ตัวเดียว สร้างครั้งเดียวตอน startup แล้วแชร์ใช้ทั่วทั้งโปรแกรม **ไม่ควรเปิด/ปิดใหม่ทุกครั้งที่จะ query**

---

## 4. `db.Exec`: คำสั่งที่ไม่คืนแถวข้อมูล

`db.Exec` ใช้กับคำสั่ง SQL ที่ไม่คาดหวังแถวข้อมูลกลับมา เช่น `CREATE TABLE`, `INSERT`, `UPDATE`, `DELETE`:

```go
func (db *DB) Exec(query string, args ...any) (Result, error)
```

ตัวอย่างสร้างตารางและ insert (รันจริงกับ PostgreSQL):

```go
_, err = db.Exec(`CREATE TABLE users (
	id SERIAL PRIMARY KEY,
	name TEXT NOT NULL,
	email TEXT NOT NULL UNIQUE
)`)
if err != nil {
	log.Fatal("create table:", err)
}

res, err := db.Exec(`INSERT INTO users (name, email) VALUES ($1, $2)`, "Somchai", "somchai@example.com")
if err != nil {
	log.Fatal("insert:", err)
}
affected, _ := res.RowsAffected()
fmt.Println("แถวที่ถูกเพิ่ม:", affected)
```

ผลลัพธ์จริง:

```
สร้างตาราง users สำเร็จ
แถวที่ถูกเพิ่ม: 1
```

สังเกตว่าเราใช้ `$1`, `$2` เป็น placeholder แทนการต่อ string เอง — นี่คือ **parameterized query** ซึ่งเป็นหัวใจของหัวข้อ 7 สังเกตด้วยว่า **รูปแบบ placeholder ต่างกันไปตามแต่ละ driver**: PostgreSQL (`lib/pq`, `pgx`) ใช้ `$1, $2, ...`, MySQL/SQLite ใช้ `?` เฉยๆ — นี่เป็นหนึ่งในจุดที่ syntax ไม่ portable 100% ข้ามฐานข้อมูล แม้ตัว `database/sql` เองจะ generic ก็ตาม

`Result` ที่ `db.Exec` คืนมา มี 2 method หลัก:

```go
type Result interface {
	LastInsertId() (int64, error) // MySQL/SQLite รองรับ, PostgreSQL ไม่รองรับ (ต้องใช้ RETURNING แทน — Part 072)
	RowsAffected() (int64, error) // จำนวนแถวที่ query กระทบ (ใช้ได้ทุกฐานข้อมูล)
}
```

จุดสำคัญ: `LastInsertId()` กับ PostgreSQL driver อย่าง `lib/pq` จะคืน error เสมอเพราะ PostgreSQL ไม่มีแนวคิดนี้ในระดับ protocol — วิธีที่ถูกต้องสำหรับ PostgreSQL คือใช้ `RETURNING` clause ซึ่งจะเรียนใน **Part 072**

---

## 5. `db.Query` และ `db.QueryRow`: ดึงข้อมูลหลายแถว vs แถวเดียว

เมื่อคาดหวังว่าจะได้แถวข้อมูลกลับมา มี 2 method ให้เลือกตามจำนวนแถวที่ต้องการ

### `db.Query`: หลายแถว

```go
func (db *DB) Query(query string, args ...any) (*Rows, error)
```

คืนค่า `*sql.Rows` ที่ใช้ loop อ่านทีละแถวด้วย `rows.Next()`

### `db.QueryRow`: แถวเดียว

```go
func (db *DB) QueryRow(query string, args ...any) *Row
```

`QueryRow` สะดวกกว่าเมื่อรู้อยู่แล้วว่าคาดหวังผลลัพธ์แค่แถวเดียว (เช่น ค้นด้วย primary key) — สังเกตว่า **`QueryRow` ไม่คืนค่า `error` แยกต่างหาก** แต่ผูก error ไว้กับตอนเรียก `.Scan(...)` แทน:

```go
var name string
err = db.QueryRow(`SELECT name FROM users WHERE id = $1`, 1).Scan(&name)
switch {
case err == sql.ErrNoRows:
	fmt.Println("ไม่พบ user")
case err != nil:
	log.Fatal(err)
default:
	fmt.Println("user id=1 ชื่อ:", name)
}
```

ผลลัพธ์จริง:

```
user id=1 ชื่อ: Somchai
```

จุดสำคัญที่ต้องจำ: ถ้าไม่พบแถวใดเลย `QueryRow(...).Scan(...)` จะคืนค่า **`sql.ErrNoRows`** เสมอ — นี่คือค่า sentinel error (ทบทวนแนวคิด sentinel error จาก **Part 016**) ที่ต้องเช็คแยกจาก error ทั่วไป เพราะ "ไม่พบข้อมูล" มักไม่ใช่ความผิดพลาดร้ายแรงของโปรแกรม แต่เป็น flow ปกติทางธุรกิจที่ต้องจัดการต่างจาก error จริงๆ (เช่น การเชื่อมต่อหลุด)

---

## 6. Scan ข้อมูลเข้า Struct, `defer rows.Close()`, และ `rows.Err()`

`database/sql` **ไม่มี ORM ในตัว** — มันไม่รู้จัก struct ของเราเลย การแปลงข้อมูลจากแถวในฐานข้อมูลเข้า struct ต้องทำ **manual scan ทีละ field** ด้วยมือเอง นี่คือความแตกต่างสำคัญจาก GORM ที่จะเรียนใน **Part 074** — และเป็นเหตุผลว่าทำไมหลายทีมเลือกใช้ ORM หรือ query builder เพื่อลด boilerplate ตรงนี้

รูปแบบมาตรฐาน (idiom) ในการ loop อ่านหลายแถว:

```go
type User struct {
	ID    int
	Name  string
	Email string
}

rows, err := db.Query(`SELECT id, name, email FROM users ORDER BY id`)
if err != nil {
	log.Fatal(err)
}
defer rows.Close() // สำคัญมาก: ต้องปิด rows เสมอ ไม่ว่า loop จะจบแบบไหน

var users []User
for rows.Next() {
	var u User
	if err := rows.Scan(&u.ID, &u.Name, &u.Email); err != nil {
		log.Fatal("scan:", err)
	}
	users = append(users, u)
}
if err := rows.Err(); err != nil { // เช็คหลัง loop จบเสมอ
	log.Fatal("rows error:", err)
}
```

ผลลัพธ์จริง:

```
  #1 Somchai <somchai@example.com>
  #2 Suda <suda@example.com>
  #3 Anan <anan@example.com>
```

มี 3 กฎเหล็กที่ต้องจำให้ขึ้นใจทุกครั้งที่ใช้ `db.Query`:

1. **`defer rows.Close()` เสมอ** — `*sql.Rows` ถือ connection จาก pool ไว้อยู่ระหว่างที่ยังไม่ปิด ถ้าลืมปิด connection จะไม่ถูกคืนกลับ pool ทำให้ pool ค่อยๆ หมดและแอปพลิเคชันค้างในที่สุด (นี่คือสาเหตุอันดับ 1 ของ "connection pool exhausted" ในโปรแกรม Go ที่ใช้ `database/sql`) เขียน `defer` ทันทีหลังเช็ค error ของ `Query` ไม่ต้องรอ
2. **`rows.Scan(...)` ต้องส่ง pointer ของทุก column ตามลำดับให้ตรงกับ `SELECT`** — จำนวนและชนิดต้องตรงกันเป๊ะ ถ้าจำนวนไม่ตรง จะได้ error ทันที
3. **เช็ค `rows.Err()` หลัง loop จบเสมอ** — ข้อผิดพลาดที่เกิดระหว่างการดึงข้อมูล (เช่น connection หลุดกลางทาง) จะไม่ถูกโยนออกมาจาก `rows.Next()` (ซึ่งคืนแค่ `bool`) แต่จะถูกเก็บไว้ให้เช็คทีหลังผ่าน `rows.Err()` — ถ้าลืมเช็คตรงนี้ อาจดูเหมือนโปรแกรมทำงานปกติทั้งที่จริงข้อมูลไม่ครบ

> เทียบกับ `defer file.Close()` ใน **Part 024 (File I/O)** — หลักการเดียวกันทุกประการ: ทรัพยากรที่เปิดมาต้องมีคนรับผิดชอบปิดเสมอ

---

## 7. Prepared Statements และ SQL Injection: เรื่องความปลอดภัยที่พลาดไม่ได้

นี่คือหัวข้อที่**สำคัญที่สุดในบทนี้**ในแง่ความปลอดภัย ถ้าจดจำได้แค่เรื่องเดียวจากบทนี้ ขอให้เป็นเรื่องนี้

### วิธีอันตราย: ต่อ string เอง

สมมติต้องการค้นหา user ด้วยชื่อที่ผู้ใช้พิมพ์เข้ามา มือใหม่มักเขียนแบบนี้โดยไม่ทันคิด:

```go
// อันตรายมาก — ห้ามทำแบบนี้เด็ดขาดใน production
dangerousQuery := fmt.Sprintf(`SELECT id, name, email FROM users WHERE name = '%s'`, userInput)
rows, err := db.Query(dangerousQuery)
```

ถ้า `userInput` เป็นข้อความปกติเช่น `"Somchai"` ก็ดูเหมือนทำงานถูกต้อง แต่ถ้าเป็นผู้ใช้ที่ประสงค์ร้ายส่งค่า:

```
hacker' OR '1'='1
```

Query ที่ประกอบขึ้นจะกลายเป็น:

```sql
SELECT id, name, email FROM users WHERE name = 'hacker' OR '1'='1'
```

เงื่อนไข `'1'='1'` เป็นจริงเสมอ ทำให้ `WHERE` ทั้งอันเป็นจริงหมดทุกแถว! เราทดลองรันจริงกับข้อมูลใน PostgreSQL (มี user 3 คน: Somchai, Suda, Anan โดยไม่มีใครชื่อ hacker เลย):

```go
maliciousInput := "hacker' OR '1'='1"
dangerousQuery := fmt.Sprintf(`SELECT id, name, email FROM users WHERE name = '%s'`, maliciousInput)
fmt.Println("Query ที่ต่อ string ตรงๆ:", dangerousQuery)
rows, err := db.Query(dangerousQuery)
```

ผลลัพธ์จริงที่ได้ (น่ากลัวมาก):

```
Query ที่ต่อ string ตรงๆ: SELECT id, name, email FROM users WHERE name = 'hacker' OR '1'='1'
!! query ที่ควรไม่เจอ user ชื่อ hacker กลับคืนแถวทั้งหมด 3 แถว เพราะเงื่อนไข OR '1'='1' เป็นจริงเสมอ
```

Query ที่ควรจะค้นหา "user ชื่อ hacker" (ซึ่งไม่มีจริง) กลับ**คืนข้อมูล user ทุกคนในระบบ** นี่เป็นแค่ตัวอย่างเบสิกที่สุด ในการโจมตีจริง SQL Injection ยังสามารถใช้ทำได้อันตรายกว่านี้มาก เช่น ลบทั้งตาราง (`'; DROP TABLE users; --`), ขโมยข้อมูลจากตารางอื่นด้วย `UNION SELECT`, หรือ bypass ระบบ login ทั้งหมด

### วิธีปลอดภัย: Parameterized Query (Prepared Statement)

การแก้ไขคือ**ไม่ต่อ string เอง**เด็ดขาด แต่ส่งค่าที่มาจากผู้ใช้ผ่าน **placeholder parameter** เสมอ:

```go
safeRows, err := db.Query(`SELECT id, name, email FROM users WHERE name = $1`, maliciousInput)
```

ผลลัพธ์จริงเมื่อรันด้วย input อันตรายตัวเดิม:

```
parameterized query ค้นหาชื่อ 'hacker' OR '1'='1' เจอ 0 แถว (ถูกต้อง เพราะไม่มี user ชื่อนี้จริง)
```

ผลลัพธ์ถูกต้อง: ไม่พบแถวใดเลย เพราะ driver จะส่งค่า `"hacker' OR '1'='1"` ไปเป็น**ค่าข้อมูลดิบ** (data) ไม่ใช่ส่วนหนึ่งของโครงสร้างคำสั่ง SQL — ฐานข้อมูลจะมองว่านี่คือการค้นหาชื่อที่มีเครื่องหมาย `'` และคำว่า `OR '1'='1` อยู่ในตัวสตริงจริงๆ ไม่ได้ตีความเป็นส่วนของ SQL syntax เลย

### ทำไม Parameterized Query ถึงปลอดภัย: กลไกเบื้องหลัง

เวลาส่ง `db.Query(sqlText, args...)` driver จะทำสิ่งที่เรียกว่า **prepared statement** ในระดับ protocol กับฐานข้อมูล คือ:

1. ส่งโครงสร้างคำสั่ง SQL (`SELECT ... WHERE name = $1`) ไปให้ฐานข้อมูล**แยกต่างหาก**ก่อน โดยที่ `$1` เป็นแค่ placeholder ว่างๆ ยังไม่มีค่า
2. ฐานข้อมูล parse และ compile query plan จากโครงสร้างนี้เสร็จเรียบร้อย **ก่อน**ที่จะรู้ค่าจริงของ `$1` เลยด้วยซ้ำ
3. จากนั้นค่อยส่งค่าจริง (`"hacker' OR '1'='1"`) ไปแทนที่ `$1` **ในฐานะข้อมูล (data)** ไม่ใช่ในฐานะโค้ด SQL

เพราะขั้นตอน parse โครงสร้างคำสั่งเกิดขึ้น**ก่อน**ที่ค่าจริงจะเข้ามา จึงเป็นไปไม่ได้เลยที่ค่าข้อมูลจะ "แทรก" ตัวเองเข้าไปเปลี่ยนโครงสร้างคำสั่ง SQL ได้ ต่างจากการต่อ string ที่ค่าข้อมูลกับโครงสร้างคำสั่งถูกผสมรวมเป็นข้อความเดียวกันตั้งแต่แรก ทำให้ฐานข้อมูลแยกไม่ออกว่าอันไหนคือ "คำสั่ง" อันไหนคือ "ข้อมูล"

### กฎทองคำ

> **ห้ามใช้ `fmt.Sprintf`, การต่อ string (`+`), หรือวิธีใดๆ ที่นำค่าจากผู้ใช้มาประกอบเป็นข้อความ SQL โดยตรงเด็ดขาด ให้ใช้ placeholder (`$1`/`?`) พร้อมส่งค่าผ่าน `args ...any` ของ `Query`/`QueryRow`/`Exec` เสมอ — ไม่มีข้อยกเว้น**

ข้อยกเว้นเดียวที่ยอมรับได้คือชื่อ table/column แบบ dynamic (เช่น `ORDER BY <column>` ที่ column มาจาก user) เพราะ placeholder ใช้แทนที่ได้แค่ค่าข้อมูล ไม่ใช่ชื่อ identifier — กรณีนี้ต้องใช้วิธี **whitelist** (เช็คว่าค่าที่รับมาตรงกับรายการชื่อ column ที่อนุญาตไว้ล่วงหน้าเท่านั้น) ไม่ใช่ต่อ string ตรงๆ

### `db.Prepare`: เตรียม statement ไว้ใช้ซ้ำ

เมื่อจะรัน query โครงสร้างเดียวกันหลายรอบด้วยค่าต่างกัน (เช่น insert หลายแถวใน loop) ควรใช้ `db.Prepare` เพื่อให้ฐานข้อมูล parse/compile query plan แค่ครั้งเดียว แล้วนำ statement ที่ได้มาเรียกซ้ำได้ — เร็วกว่าการเรียก `db.Exec` ตรงๆ ทุกรอบ (ซึ่งแต่ละรอบฐานข้อมูลต้อง parse ใหม่ทุกครั้งถ้าไม่มี query cache):

```go
stmt, err := db.Prepare(`INSERT INTO users (name, email) VALUES ($1, $2)`)
if err != nil {
	log.Fatal(err)
}
defer stmt.Close() // Prepared statement ก็เป็นทรัพยากรที่ต้องปิดเหมือนกัน

names := []struct{ name, email string }{
	{"Suda", "suda@example.com"},
	{"Anan", "anan@example.com"},
}
for _, n := range names {
	if _, err := stmt.Exec(n.name, n.email); err != nil {
		log.Fatal(err)
	}
}
```

ผลลัพธ์จริง:

```
เพิ่มข้อมูลผ่าน prepared statement สำเร็จ
```

**ข้อควรระวัง**: อย่าสับสนระหว่าง "Prepared Statement" (คุณสมบัติด้าน performance ที่ทำให้ query plan ถูก cache ไว้ใช้ซ้ำ) กับ "Parameterized Query" (คุณสมบัติด้านความปลอดภัยจากการแยกโครงสร้างคำสั่งออกจากข้อมูล) — ในทางปฏิบัติเวลาเรียก `db.Query(sql, args...)` หรือ `db.Exec(sql, args...)` แบบ `database/sql` มาตรฐาน ภายใต้ผิว driver ส่วนใหญ่จะสร้าง prepared statement แบบ implicit (ใช้ครั้งเดียวแล้วทิ้ง) ให้อัตโนมัติอยู่แล้ว จึงได้ทั้งสองคุณสมบัตินี้พร้อมกันเสมอตราบใดที่ใช้ placeholder อย่างถูกต้อง — ไม่จำเป็นต้องเรียก `db.Prepare` เองทุกครั้งเพื่อความปลอดภัย (ความปลอดภัยมาจากการใช้ placeholder ไม่ใช่จากการเรียก `Prepare`) แต่ `db.Prepare` คุ้มค่าเมื่อต้องการ**ประสิทธิภาพ**จากการรัน query โครงสร้างเดิมซ้ำหลายรอบ

---

## 8. `QueryContext` / `ExecContext` กับ `context.Context`

ทุก method ของ `database/sql` มีคู่แฝดที่รับ `context.Context` เป็น parameter แรกเสมอ — ทบทวนจาก **Part 032** ว่า `context.Context` ใช้ส่งสัญญาณ cancellation และ timeout ไปยังงานที่กำลังทำงานอยู่:

| ไม่มี Context | มี Context |
|---|---|
| `db.Query(query, args...)` | `db.QueryContext(ctx, query, args...)` |
| `db.QueryRow(query, args...)` | `db.QueryRowContext(ctx, query, args...)` |
| `db.Exec(query, args...)` | `db.ExecContext(ctx, query, args...)` |
| `db.Ping()` | `db.PingContext(ctx)` |
| `db.Prepare(query)` | `db.PrepareContext(ctx, query)` |

การใช้เวอร์ชันที่มี `ctx` สำคัญมากในเว็บแอปพลิเคชัน (ที่จะเรียนเจาะลึกใน **Part 047** และประยุกต์กับฐานข้อมูลในโปรเจกต์จริงช่วง **ภาคที่ 10**) เพราะ `http.Request` ทุกตัวมี `context.Context` ติดตัวมาอัตโนมัติ ถ้า client ปิดการเชื่อมต่อกลางทาง หรือตั้ง timeout ไว้ การส่ง `r.Context()` เข้าไปที่ `QueryContext` จะทำให้ query ที่กำลังทำงานอยู่ **ถูกยกเลิกทันที** แทนที่จะปล่อยให้ทำงานต่อไปโดยไม่มีใครรอผลแล้ว (ซึ่งสิ้นเปลือง connection และทรัพยากรฐานข้อมูลโดยเปล่าประโยชน์)

ตัวอย่างรันจริง — ใช้ `context.WithTimeout` จำกัดเวลา query ไว้ 2 วินาที:

```go
ctx, cancel := context.WithTimeout(context.Background(), 2*time.Second)
defer cancel()

rows3, err := db.QueryContext(ctx, `SELECT COUNT(*) FROM users`)
if err != nil {
	log.Fatal(err)
}
var total int
for rows3.Next() {
	rows3.Scan(&total)
}
rows3.Close()
fmt.Println("จำนวน user ทั้งหมด:", total)
```

ผลลัพธ์จริง:

```
จำนวน user ทั้งหมด: 3
```

ถ้า query ใช้เวลานานเกิน 2 วินาที (เช่น table ใหญ่มากและไม่มี index) `ctx` จะถูกยกเลิกอัตโนมัติ และ `QueryContext` จะคืน error ที่ wrap `context.DeadlineExceeded` ไว้ทันที แทนที่จะรอ query ทำงานจนจบ

> **แนวทางปฏิบัติ**: ในโค้ด production โดยเฉพาะฝั่ง server ควรใช้ `...Context` variant เสมอ ไม่ใช้เวอร์ชันไม่มี `ctx` เลย ยกเว้นสคริปต์เล็กๆ ที่ไม่มี context ให้ส่งต่ออยู่แล้ว (เช่น `main()` ตรงๆ ที่ใช้ `context.Background()`)

---

## 9. Connection Pool เบื้องต้น: `SetMaxOpenConns`, `SetMaxIdleConns`, `SetConnMaxLifetime`

อย่างที่กล่าวไปในหัวข้อ 3 ว่า `*sql.DB` คือ connection pool ไม่ใช่ connection เดี่ยว — Go จัดการเปิด/ปิด/ใช้ซ้ำ connection ให้อัตโนมัติ แต่เราสามารถปรับพฤติกรรมของ pool ได้ผ่าน 3 method หลัก (บทนี้แนะนำแค่ภาพรวม รายละเอียดเชิงลึกเรื่อง tuning ค่าเหล่านี้สำหรับ production จะอยู่ใน **Part 078: Database Transactions และ Connection Pooling**):

```go
db.SetMaxOpenConns(25)              // จำนวน connection สูงสุดที่เปิดพร้อมกันได้ (ทั้ง idle + กำลังใช้งาน)
db.SetMaxIdleConns(25)              // จำนวน connection สูงสุดที่พักไว้เฉยๆ (idle) รอถูกใช้ซ้ำ
db.SetConnMaxLifetime(5 * time.Minute) // อายุสูงสุดของ connection หนึ่งเส้น ก่อนถูกปิดแล้วเปิดใหม่
```

รันจริงและตรวจสอบสถิติ pool ด้วย `db.Stats()`:

```go
stats := db.Stats()
fmt.Printf("MaxOpenConnections=%d OpenConnections=%d InUse=%d Idle=%d\n",
	stats.MaxOpenConnections, stats.OpenConnections, stats.InUse, stats.Idle)
```

ผลลัพธ์จริง:

```
MaxOpenConnections=25 OpenConnections=1 InUse=0 Idle=1
```

(ตัวเลข `OpenConnections`/`InUse`/`Idle` ที่แสดงจะแตกต่างกันไปตามจังหวะที่เรียก เพราะขึ้นกับว่ามี query ไหนกำลังทำงานอยู่ ณ ขณะนั้น)

ทำไมค่า default (ไม่ตั้งอะไรเลย = ไม่จำกัดจำนวน connection สูงสุด) ถึงอันตราย: ถ้าแอปพลิเคชันมี traffic สูงพร้อมกันมาก และไม่จำกัด `MaxOpenConns` แอปพลิเคชันอาจเปิด connection เข้าฐานข้อมูลเป็นพันๆ เส้นพร้อมกัน จนฐานข้อมูล**ปฏิเสธการเชื่อมต่อใหม่**หรือ**หน่วยความจำฝั่งฐานข้อมูลพัง**เอาได้ง่ายๆ การตั้งค่า pool ให้เหมาะสมกับขนาดฐานข้อมูลและ traffic ที่คาดหวังจึงเป็นเรื่องสำคัญมากสำหรับระบบ production — เราจะกลับมาเจาะลึกเรื่องการเลือกค่าที่เหมาะสม ผลกระทบต่อ latency และการ monitor pool ใน **Part 078**

---

## 10. ตัวอย่างเต็ม: CRUD ที่รันจริงกับ PostgreSQL

รวมทุกหัวข้อที่เรียนมาไว้ในโปรแกรมเดียว — โปรแกรมนี้ถูกรันจริงทั้งหมด ผลลัพธ์คือ output จริง:

```go
package main

import (
	"context"
	"database/sql"
	"fmt"
	"log"
	"time"

	_ "github.com/lib/pq"
)

type User struct {
	ID    int
	Name  string
	Email string
}

func main() {
	db, err := sql.Open("postgres", "postgres://postgres:postgres@127.0.0.1:5432/godemo?sslmode=disable")
	if err != nil {
		log.Fatal("sql.Open error:", err)
	}
	defer db.Close()

	if err := db.Ping(); err != nil {
		log.Fatal("ping error:", err)
	}
	fmt.Println("เชื่อมต่อฐานข้อมูลสำเร็จ (Ping OK)")

	// CREATE
	_, err = db.Exec(`CREATE TABLE IF NOT EXISTS users (
		id SERIAL PRIMARY KEY,
		name TEXT NOT NULL,
		email TEXT NOT NULL UNIQUE
	)`)
	if err != nil {
		log.Fatal(err)
	}

	// INSERT (parameterized)
	_, err = db.Exec(`INSERT INTO users (name, email) VALUES ($1, $2)`, "Somchai", "somchai@example.com")
	if err != nil {
		log.Fatal(err)
	}

	// READ ทั้งหมด
	rows, err := db.Query(`SELECT id, name, email FROM users ORDER BY id`)
	if err != nil {
		log.Fatal(err)
	}
	defer rows.Close()

	var users []User
	for rows.Next() {
		var u User
		if err := rows.Scan(&u.ID, &u.Name, &u.Email); err != nil {
			log.Fatal(err)
		}
		users = append(users, u)
	}
	if err := rows.Err(); err != nil {
		log.Fatal(err)
	}
	for _, u := range users {
		fmt.Printf("  #%d %s <%s>\n", u.ID, u.Name, u.Email)
	}

	// UPDATE
	ctx, cancel := context.WithTimeout(context.Background(), 3*time.Second)
	defer cancel()
	_, err = db.ExecContext(ctx, `UPDATE users SET email = $1 WHERE name = $2`, "somchai_new@example.com", "Somchai")
	if err != nil {
		log.Fatal(err)
	}

	// DELETE
	_, err = db.ExecContext(ctx, `DELETE FROM users WHERE name = $1`, "Somchai")
	if err != nil {
		log.Fatal(err)
	}
	fmt.Println("CRUD ครบทั้ง 4 operation สำเร็จ")
}
```

โค้ดชุดนี้ครอบคลุมทุกกลไกหลักของ `database/sql`: `sql.Open` + `Ping`, `Exec` สำหรับ DDL/DML, `Query` + `Scan` + `rows.Err()`, และ `...Context` variant พร้อม timeout — และเป็นพื้นฐานที่จะต่อยอดไปเรียน syntax เฉพาะของ PostgreSQL ใน Part ถัดไป

---

## สรุปสิ่งที่ได้เรียนในบทนี้

- `database/sql` เป็น interface กลางที่ไม่ผูกกับฐานข้อมูลยี่ห้อใดยี่ห้อหนึ่ง ต้อง import driver แยกต่างหากเสมอด้วย blank import (`_ "..."`) เพื่อให้ `init()` ของ driver ลงทะเบียนตัวเองผ่าน `sql.Register`
- `sql.Open` ไม่เชื่อมต่อฐานข้อมูลจริงทันที (lazy connection) — ต้องเรียก `db.Ping()` เพื่อตรวจสอบการเชื่อมต่อจริง
- `db.Exec` ใช้กับคำสั่งที่ไม่คืนแถว (INSERT/UPDATE/DELETE/CREATE) คืนค่า `Result` ที่มี `RowsAffected()` และ `LastInsertId()` (ตัวหลังใช้ไม่ได้กับ PostgreSQL)
- `db.Query` คืนหลายแถวผ่าน `*sql.Rows` ต้อง `defer rows.Close()` เสมอและเช็ค `rows.Err()` หลัง loop จบ, `db.QueryRow` คืนแถวเดียวและใช้ `sql.ErrNoRows` บอกกรณีไม่พบข้อมูล
- **ห้ามต่อ string SQL เองจากค่าที่มาจากผู้ใช้เด็ดขาด** — ต้องใช้ parameterized query (`$1`/`?`) เสมอ เพื่อป้องกัน SQL Injection ซึ่งเป็นช่องโหว่ความปลอดภัยที่อันตรายที่สุดอย่างหนึ่งของแอปพลิเคชันที่คุยกับฐานข้อมูล
- `db.Prepare` ช่วยเรื่อง performance เมื่อรัน query โครงสร้างเดิมซ้ำหลายรอบ แต่ความปลอดภัยจาก injection มาจากการใช้ placeholder ไม่ใช่จากการเรียก `Prepare`
- ทุก method มีเวอร์ชัน `...Context` ที่รับ `context.Context` เพื่อรองรับ cancellation/timeout ควรใช้เป็นค่าเริ่มต้นในโค้ด production
- `SetMaxOpenConns`, `SetMaxIdleConns`, `SetConnMaxLifetime` ควบคุมพฤติกรรม connection pool — รายละเอียดเชิงลึกและการ tuning จะอยู่ใน Part 078

## แบบฝึกหัดท้ายบท

1. เขียนโปรแกรมที่เชื่อมต่อฐานข้อมูล SQLite (ใช้ driver `modernc.org/sqlite` ซึ่งเป็น pure Go ไม่ต้องพึ่ง cgo) แล้วสร้างตาราง `products` (id, name, price) และทำ CRUD ครบทั้ง 4 แบบ
2. ทดลองลบ `defer rows.Close()` ออกจากโปรแกรม แล้วเขียน loop ที่เรียก `db.Query` ซ้ำๆ หลายพันรอบโดยไม่ปิด `rows` เลย สังเกตว่าเกิดอะไรขึ้นกับพฤติกรรมของโปรแกรม (คำใบ้: ลอง `db.SetMaxOpenConns(5)` แล้วดูว่าโปรแกรมค้างหรือไม่)
3. เขียนฟังก์ชัน `FindUserByName(db *sql.DB, name string) (*User, error)` ที่คืน `nil, nil` เมื่อไม่พบ user (โดยเช็ค `sql.ErrNoRows` ภายในฟังก์ชัน) แทนที่จะให้ error หลุดออกไปให้ผู้เรียกต้องเช็ค `sql.ErrNoRows` เอง
4. ลองสร้างตารางที่มี column `note TEXT` แล้วเขียนโปรแกรมสองเวอร์ชัน: เวอร์ชันหนึ่งต่อ string ด้วย `fmt.Sprintf` (แบบอันตราย), อีกเวอร์ชันใช้ parameterized query — ทดลองส่งค่า input ที่มีเครื่องหมาย `'` ปนอยู่ (เช่น `O'Brien`) แล้วดูว่าเวอร์ชันไหนพังและเวอร์ชันไหนทำงานถูกต้อง
5. เขียนฟังก์ชันที่รับ `ctx context.Context` เป็น parameter แรกเสมอ แล้วภายในใช้ `QueryContext` ทุกจุด จากนั้นทดลองเรียกด้วย `context.WithTimeout(ctx, 1*time.Nanosecond)` (เวลาน้อยจนไม่มีทางเสร็จทัน) แล้วดู error ที่ได้ ว่า wrap `context.DeadlineExceeded` ไว้จริงหรือไม่ (ใช้ `errors.Is` ทบทวนจาก Part 016)
6. ปรับค่า `SetMaxOpenConns` เป็น 2 แล้วเขียนโปรแกรมที่ spawn 10 goroutine (ทบทวน Part 036) ให้แต่ละตัวรัน query ที่ใช้เวลานาน (เช่น `SELECT pg_sleep(1)` สำหรับ PostgreSQL) พร้อมกัน แล้ววัดเวลารวมที่ใช้ — อธิบายว่าทำไมเวลารวมถึงนานกว่าการรันแบบไม่จำกัด pool

---

**ต่อไป**: [Part 072 — PostgreSQL กับ Go (`pgx`, `lib/pq`)](./072-postgresql.md)
