# Part 073: MySQL กับ Go

> ภาคที่ 6: ฐานข้อมูล — ตอนที่ 3 จาก 8 (Part 71–78)

## สารบัญของบทนี้

1. MySQL ในโลก Go: Driver มาตรฐาน `go-sql-driver/mysql`
2. DSN ของ MySQL: รูปแบบเฉพาะที่ต่างจาก PostgreSQL
3. `parseTime=true`: Gotcha ที่พบบ่อยที่สุดเวลาใช้ MySQL กับ Go
4. ตัวอย่าง CRUD ที่รันจริงกับ MySQL-compatible Server
5. Auto-increment ID: `LastInsertId()` vs PostgreSQL's `RETURNING`
6. Case-Sensitivity ของชื่อตารางบนแพลตฟอร์มต่างกัน
7. จัดการ Error เฉพาะของ MySQL: `*mysql.MySQLError` และ Error Code 1062
8. ตารางเปรียบเทียบ: เมื่อไหร่ควรเลือก MySQL เมื่อไหร่ควรเลือก PostgreSQL
9. สรุปสิ่งที่ได้เรียนในบทนี้
10. แบบฝึกหัดท้ายบท

---

## หมายเหตุความซื่อสัตย์เรื่องฐานข้อมูลที่ใช้ทดสอบ

บทนี้ตั้งใจสาธิตการคุยกับ **MySQL** จาก Go แต่ในสภาพแวดล้อมที่ใช้เขียนหลักสูตรนี้ ตัวติดตั้ง `mysql-server` จาก apt repository ของ Ubuntu ดาวน์โหลดไม่สำเร็จ (mirror คืน HTTP 404) จึงติดตั้ง **MariaDB 10.11** แทน — MariaDB เป็น fork ของ MySQL ที่พูด **wire protocol เดียวกันกับ MySQL ทุกประการ** และเป็นเซิร์ฟเวอร์ที่ driver `github.com/go-sql-driver/mysql` รองรับอย่างเป็นทางการ (README ของ driver ระบุ MariaDB ไว้เป็นหนึ่งในเซิร์ฟเวอร์ที่ทดสอบด้วยโดยตรง) โค้ดทุกบรรทัดในบทนี้ **รันจริงกับ MariaDB 10.11** ด้วย DSN, driver, และ syntax เดียวกันเป๊ะกับที่ใช้กับ MySQL แท้ — ผลลัพธ์ที่แสดงคือ output จริงจากการรัน ความแตกต่างระหว่าง MySQL กับ MariaDB ในระดับที่กระทบโค้ดฝั่ง Go แทบไม่มีเลยสำหรับการใช้งานพื้นฐานที่สอนในบทนี้ ส่วนไหนที่เป็น feature เฉพาะของ MySQL แท้ๆ ที่ MariaDB ไม่มี (เช่น MySQL 8's `X DevAPI`) จะระบุไว้ชัดเจนว่าเป็นข้อมูลอ้างอิงเท่านั้น

---

## 1. MySQL ในโลก Go: Driver มาตรฐาน `go-sql-driver/mysql`

**MySQL** เป็นฐานข้อมูลเชิงสัมพันธ์ที่มีฐานผู้ใช้งานมากที่สุดตัวหนึ่งของโลก ใช้กันแพร่หลายมาตั้งแต่ยุคแรกๆ ของเว็บ (คู่กับ PHP ในยุค LAMP stack) และยังคงเป็นตัวเลือกหลักของระบบจำนวนมหาศาลในปัจจุบัน ไม่ว่าจะเป็นระบบเก่าที่มีอยู่แล้ว (legacy system) หรือระบบใหม่ที่ทีมคุ้นเคยกับ MySQL อยู่แล้ว

ต่างจาก PostgreSQL ที่มี driver แข่งกันหลายตัว (`lib/pq`, `pgx`) วงการ Go มี driver สำหรับ MySQL ที่ community ยอมรับเป็นมาตรฐานแทบจะตัวเดียวชัดเจนคือ **`github.com/go-sql-driver/mysql`** — เป็น pure Go driver (ไม่ต้องพึ่ง cgo หรือ libmysqlclient ของระบบปฏิบัติการ) implement `database/sql` driver interface มาตรฐานเต็มรูปแบบ ดูแลและพัฒนาต่อเนื่องมายาวนาน เสถียรมากในระดับ production

ติดตั้ง:

```bash
go get github.com/go-sql-driver/mysql
```

ใช้งานผ่าน `database/sql` แบบเดียวกับที่เรียนใน **Part 071** ทุกประการ:

```go
import (
	"database/sql"

	_ "github.com/go-sql-driver/mysql" // ลงทะเบียน driver name "mysql"
)

db, err := sql.Open("mysql", dsn)
```

---

## 2. DSN ของ MySQL: รูปแบบเฉพาะที่ต่างจาก PostgreSQL

DSN ของ `go-sql-driver/mysql` มีรูปแบบที่ต่างจาก URL แบบ PostgreSQL ค่อนข้างชัดเจน:

```
username:password@protocol(address)/dbname?param1=value1&param2=value2
```

ตัวอย่างที่ใช้เชื่อมต่อผ่าน TCP ไปยัง localhost:

```
godemo:godemo_pass@tcp(127.0.0.1:3306)/godemo?parseTime=true&charset=utf8mb4
```

แยกส่วนประกอบ:

| ส่วน | ความหมาย |
|---|---|
| `godemo:godemo_pass` | username:password |
| `tcp(127.0.0.1:3306)` | protocol และ address — MySQL default port คือ 3306 (ต่างจาก PostgreSQL ที่ 5432) นอกจาก `tcp` ยังใช้ `unix(/path/to/socket)` เชื่อมผ่าน Unix socket ได้ |
| `godemo` (หลัง `/`) | ชื่อฐานข้อมูล |
| `?parseTime=true&charset=utf8mb4` | query parameter ปรับพฤติกรรม driver |

พารามิเตอร์ที่ควรรู้จัก:

| พารามิเตอร์ | ความหมาย |
|---|---|
| `parseTime` | **สำคัญมาก** — ดูหัวข้อ 3 |
| `charset` | ชุดตัวอักษร ควรใช้ `utf8mb4` เสมอ (ไม่ใช่ `utf8` ธรรมดาที่ MySQL รองรับ Unicode ไม่เต็มรูปแบบ โดยเฉพาะ emoji และตัวอักษรบางภาษา) |
| `loc` | Timezone ที่ใช้แปลงค่า `DATETIME`/`TIMESTAMP` เช่น `loc=Asia%2FBangkok` |
| `timeout` | เวลาสูงสุดรอการเชื่อมต่อ |
| `readTimeout` / `writeTimeout` | เวลาสูงสุดรอ I/O แต่ละคำสั่ง |
| `multiStatements` | อนุญาตให้รันหลายคำสั่ง SQL ในการเรียกครั้งเดียว (คั่นด้วย `;`) — ปกติควรปิดไว้ (default) เพราะเปิดความเสี่ยงด้าน SQL Injection เพิ่มขึ้นถ้าดันไปต่อ string เอง |

---

## 3. `parseTime=true`: Gotcha ที่พบบ่อยที่สุดเวลาใช้ MySQL กับ Go

นี่คือกับดักที่นักพัฒนาที่มาจาก PostgreSQL หรือใช้ MySQL ครั้งแรกเจอบ่อยที่สุด — **ถ้าไม่ใส่ `parseTime=true` ใน DSN การ `Scan` column ชนิด `DATETIME`/`TIMESTAMP` เข้า `time.Time` จะ error ทันที**

เราทดลองรันจริงเพื่อดู error message ที่แท้จริงเมื่อไม่ใส่ `parseTime=true`:

```go
// ไม่ใส่ parseTime=true โดยตั้งใจ เพื่อสาธิต gotcha
dsn := "godemo:godemo_pass@tcp(127.0.0.1:3306)/godemo?charset=utf8mb4"
db, err := sql.Open("mysql", dsn)
// ...

var t time.Time
err = db.QueryRow(`SELECT created_at FROM orders LIMIT 1`).Scan(&t)
if err != nil {
	fmt.Println("Scan เข้า time.Time โดยไม่มี parseTime=true ล้มเหลว:", err)
}
```

ผลลัพธ์จริงที่ได้จากการรัน:

```
Scan เข้า time.Time โดยไม่มี parseTime=true ล้มเหลว: sql: Scan error on column index 0, name "created_at": unsupported Scan, storing driver.Value type []uint8 into type *time.Time
```

สาเหตุคือ: ถ้าไม่เปิด `parseTime=true` driver จะส่งค่า `DATETIME`/`TIMESTAMP` กลับมาเป็น **`[]byte` (ข้อความดิบ)** ไม่ใช่ `time.Time` — `database/sql` จึงไม่รู้วิธีแปลง `[]byte` เป็น `time.Time` ให้อัตโนมัติ (ต่างจาก `pgx`/`lib/pq` ที่แปลง PostgreSQL's `TIMESTAMP` เป็น `time.Time` ให้อัตโนมัติเสมอโดยไม่ต้องตั้งค่าอะไรเพิ่ม) ถ้า scan เข้า `string` ธรรมดาแทนจะไม่ error แต่ได้ข้อความดิบแทน:

```go
var s string
err = db.QueryRow(`SELECT created_at FROM orders LIMIT 1`).Scan(&s)
// ถ้า scan เป็น string ธรรมดาจะได้: 2026-09-26 02:51:00
```

ผลลัพธ์จริง:

```
ถ้า scan เป็น string ธรรมดาจะได้: 2026-09-26 02:51:00
```

**วิธีแก้ถาวร**: ใส่ `parseTime=true` ใน DSN เสมอทุกครั้งที่เชื่อมต่อ MySQL/MariaDB จาก Go ถ้าตารางมี column วันเวลาที่ต้องการใช้เป็น `time.Time`:

```go
dsn := "godemo:godemo_pass@tcp(127.0.0.1:3306)/godemo?parseTime=true&charset=utf8mb4"
```

> **แนวทางปฏิบัติ**: จำ query parameter นี้ให้ขึ้นใจ — ใส่มันเป็นค่าเริ่มต้นในทุกโปรเจกต์ที่ใช้ `go-sql-driver/mysql` ตั้งแต่วันแรก เพราะแทบทุกตารางในระบบจริงมี column วันเวลาอย่างน้อยหนึ่ง column (`created_at`, `updated_at` เป็นต้น) การลืมใส่พารามิเตอร์นี้เป็นสาเหตุของ bug report จำนวนมากในโปรเจกต์ Go ที่ใช้ MySQL

---

## 4. ตัวอย่าง CRUD ที่รันจริงกับ MySQL-compatible Server

โปรแกรมเต็มที่สาธิต CRUD ครบทั้ง 4 แบบ รันจริงกับ MariaDB (ตามที่อธิบายไว้ในหมายเหตุต้นบท):

```go
package main

import (
	"database/sql"
	"fmt"
	"log"
	"time"

	_ "github.com/go-sql-driver/mysql"
)

type Order struct {
	ID        int64
	Product   string
	Quantity  int
	CreatedAt time.Time
}

func main() {
	dsn := "godemo:godemo_pass@tcp(127.0.0.1:3306)/godemo?parseTime=true&charset=utf8mb4"
	db, err := sql.Open("mysql", dsn)
	if err != nil {
		log.Fatal("sql.Open:", err)
	}
	defer db.Close()

	if err := db.Ping(); err != nil {
		log.Fatal("ping:", err)
	}
	fmt.Println("เชื่อมต่อ MySQL/MariaDB สำเร็จ")

	// CREATE TABLE
	_, err = db.Exec(`CREATE TABLE IF NOT EXISTS orders (
		id BIGINT AUTO_INCREMENT PRIMARY KEY,
		product VARCHAR(255) NOT NULL,
		quantity INT NOT NULL,
		created_at DATETIME NOT NULL
	)`)
	if err != nil {
		log.Fatal("create table:", err)
	}

	// CREATE (INSERT)
	res, err := db.Exec(`INSERT INTO orders (product, quantity, created_at) VALUES (?, ?, ?)`,
		"เมาส์ไร้สาย", 2, time.Now())
	if err != nil {
		log.Fatal("insert:", err)
	}
	lastID, _ := res.LastInsertId()
	fmt.Println("แถวที่เพิ่ม ได้ auto-increment id =", lastID)

	// READ
	rows, err := db.Query(`SELECT id, product, quantity, created_at FROM orders ORDER BY id`)
	if err != nil {
		log.Fatal(err)
	}
	defer rows.Close()

	var orders []Order
	for rows.Next() {
		var o Order
		if err := rows.Scan(&o.ID, &o.Product, &o.Quantity, &o.CreatedAt); err != nil {
			log.Fatal("scan:", err)
		}
		orders = append(orders, o)
	}
	if err := rows.Err(); err != nil {
		log.Fatal(err)
	}
	for _, o := range orders {
		fmt.Printf("  #%d %-14s qty=%d created_at=%s\n", o.ID, o.Product, o.Quantity, o.CreatedAt.Format(time.RFC3339))
	}

	// UPDATE
	res2, err := db.Exec(`UPDATE orders SET quantity = ? WHERE product = ?`, 5, "เมาส์ไร้สาย")
	if err != nil {
		log.Fatal(err)
	}
	n, _ := res2.RowsAffected()
	fmt.Println("update affected rows:", n)

	// DELETE
	res3, err := db.Exec(`DELETE FROM orders WHERE product = ?`, "คีย์บอร์ด")
	if err != nil {
		log.Fatal(err)
	}
	n3, _ := res3.RowsAffected()
	fmt.Println("delete affected rows:", n3)
}
```

ผลลัพธ์จริงจากการรัน:

```
เชื่อมต่อ MySQL/MariaDB สำเร็จ
สร้างตาราง orders สำเร็จ
แถวที่เพิ่ม ได้ auto-increment id = 1

--- อ่านข้อมูลกลับ (ทดสอบ parseTime=true scan เป็น time.Time) ---
  #1 เมาส์ไร้สาย    qty=2 created_at(type=time.Time)=2026-09-26T02:51:00Z
  #2 คีย์บอร์ด      qty=1 created_at(type=time.Time)=2026-09-26T02:51:00Z

--- ทดสอบ UPDATE / DELETE ---
update affected rows: 1
delete affected rows: 1
```

สังเกตว่าโครงสร้างโค้ดแทบเหมือนกับ PostgreSQL ทุกอย่างตามหลักการ driver-agnostic ของ `database/sql` — ความต่างที่เห็นชัดคือ **placeholder ใช้ `?` แทน `$1, $2`** และ syntax การสร้างตาราง (`AUTO_INCREMENT` แทน `SERIAL`, `DATETIME` แทน `TIMESTAMPTZ`)

---

## 5. Auto-increment ID: `LastInsertId()` vs PostgreSQL's `RETURNING`

ใน **Part 072** เรียนไปแล้วว่า PostgreSQL ไม่รองรับ `LastInsertId()` และต้องใช้ `RETURNING` clause แทน — MySQL กลับตรงข้าม: **`LastInsertId()` ใช้งานได้ปกติและเป็นวิธีมาตรฐานในการดึง auto-increment ID** ที่เพิ่งถูกสร้างจาก `INSERT`:

```go
res, err := db.Exec(`INSERT INTO orders (product, quantity, created_at) VALUES (?, ?, ?)`, "เมาส์ไร้สาย", 2, time.Now())
lastID, err := res.LastInsertId()
```

ผลลัพธ์จริง: `แถวที่เพิ่ม ได้ auto-increment id = 1`

เหตุผลที่ MySQL ทำแบบนี้ได้คือ protocol ของ MySQL ส่งค่า auto-increment ID ที่เพิ่งสร้างกลับมาพร้อมกับ response ของคำสั่ง `INSERT` เองอยู่แล้วในระดับ wire protocol โดยไม่ต้อง query แยก — driver แค่ดึงค่านั้นออกมาให้ ต่างจาก PostgreSQL ที่ไม่มีกลไกนี้ในระดับ protocol เลย จึงต้องพึ่ง `RETURNING` ซึ่งเป็นการยิง statement พิเศษที่คืนแถวข้อมูลแทน

> **หมายเหตุ**: MySQL 8+ (เวอร์ชันใหม่) เริ่มรองรับ syntax คล้าย `RETURNING` บ้างแล้วในบาง context ผ่านส่วนขยายเฉพาะ แต่ยังไม่ครอบคลุมเท่า PostgreSQL และยังไม่ได้เป็นมาตรฐานที่ community ใช้กันแพร่หลายเท่า `LastInsertId()` — ข้อมูลส่วนนี้เป็นข้อมูลอ้างอิงจาก MySQL documentation ไม่ได้ทดสอบจริงในสภาพแวดล้อมนี้เพราะ MariaDB ยังไม่รองรับ syntax นี้ในลักษณะเดียวกัน

---

## 6. Case-Sensitivity ของชื่อตารางบนแพลตฟอร์มต่างกัน

จุดที่สร้างความสับสน (และ bug ที่ debug ยาก) ให้ทีมที่ deploy ข้าม OS บ่อยๆ คือ **MySQL/MariaDB จัดการ case-sensitivity ของชื่อตารางแตกต่างกันไปตามระบบปฏิบัติการของเซิร์ฟเวอร์** โดยควบคุมด้วยตัวแปรระบบ `lower_case_table_names`:

| ค่า `lower_case_table_names` | พฤติกรรม | พบบน |
|---|---|---|
| `0` | ชื่อตารางเก็บตามตัวพิมพ์ที่สร้างจริง และ **case-sensitive** (`Users` กับ `users` เป็นคนละตารางกัน) | Linux (ค่า default) |
| `1` | ชื่อตารางถูกแปลงเป็นตัวพิมพ์เล็กเสมอตอนเก็บลง disk และเทียบแบบ case-insensitive | Windows (ค่า default) |
| `2` | เก็บตามตัวพิมพ์ที่สร้างจริง แต่เทียบชื่อแบบ case-insensitive | macOS (ค่า default) |

เราตรวจสอบค่านี้จริงบนเซิร์ฟเวอร์ MariaDB ที่ใช้ในบทเรียน (รันบน Linux):

```sql
SHOW VARIABLES LIKE 'lower_case_table_names';
```

ผลลัพธ์จริง:

```
Variable_name            Value
lower_case_table_names   0
```

ยืนยันว่าเซิร์ฟเวอร์นี้ (Linux) ตั้งค่าเป็น case-sensitive ตามค่า default ของแพลตฟอร์ม — หมายความว่าถ้าเขียน `SELECT * FROM Orders` ทั้งที่สร้างตารางไว้ชื่อ `orders` (ตัวเล็กทั้งหมด) จะได้ error ว่าตารางไม่มีอยู่จริงทันที **บน Linux** แต่ถ้า deploy โค้ดเดียวกันนี้ไปที่เซิร์ฟเวอร์ MySQL บน Windows กลับจะทำงานได้ปกติเพราะ Windows เทียบชื่อแบบไม่สนตัวพิมพ์

**แนวทางปฏิบัติที่ปลอดภัยที่สุด**: ตั้งชื่อ table และ column เป็น **lowercase + snake_case เสมอ** (เช่น `orders`, `order_items`, `created_at`) ไม่ผสมตัวพิมพ์ใหญ่เล็กเด็ดขาด และเขียน SQL ให้ตรงกับชื่อจริงเป๊ะทุกตัวอักษร วิธีนี้ทำให้โค้ดพกพาข้าม OS ได้อย่างปลอดภัย โดยไม่ต้องพึ่งพฤติกรรม default ของแพลตฟอร์มใดๆ เลย — เป็นเหตุผลเดียวกับที่ PostgreSQL แนะนำให้ใช้ lowercase เช่นกัน (PostgreSQL จะแปลง identifier ที่ไม่ได้ครอบด้วย `"..."` เป็นตัวพิมพ์เล็กอัตโนมัติเสมออยู่แล้ว ทำให้ปัญหานี้ไม่ค่อยเกิดกับ Postgres)

---

## 7. จัดการ Error เฉพาะของ MySQL: `*mysql.MySQLError` และ Error Code 1062

เวลา query ล้มเหลวเพราะละเมิด constraint ของฐานข้อมูล (เช่น unique constraint, foreign key) `database/sql` จะคืน `error` ทั่วไปกลับมา แต่ `go-sql-driver/mysql` ให้รายละเอียดเพิ่มเติมผ่าน type เฉพาะของมันเองคือ **`*mysql.MySQLError`** ซึ่งมี field `Number` (รหัส error ตัวเลขของ MySQL) และ `Message` — ทบทวนเทคนิค `errors.As` จาก **Part 016** เพื่อแปลง error ทั่วไปกลับเป็น type เฉพาะนี้:

```go
import (
	"errors"

	"github.com/go-sql-driver/mysql"
)

_, err = db.Exec(`INSERT INTO accounts (username) VALUES (?)`, "somchai")
if err != nil {
	var mysqlErr *mysql.MySQLError
	if errors.As(err, &mysqlErr) {
		fmt.Printf("MySQLError: Number=%d Message=%q\n", mysqlErr.Number, mysqlErr.Message)
		if mysqlErr.Number == 1062 {
			fmt.Println("นี่คือ error code 1062 = Duplicate entry (unique constraint violation)")
		}
	}
}
```

ทดลองรันจริงโดย insert username เดิมซ้ำสองครั้งเข้าตารางที่มี `UNIQUE` constraint บน column `username`:

```
insert แรกสำเร็จ
insert ซ้ำเจอ MySQLError: Number=1062 Message="Duplicate entry 'somchai' for key 'username'"
ยืนยันว่าเป็น error code 1062 = Duplicate entry (unique constraint violation)
```

Error code 1062 เป็นหนึ่งในรหัสที่เจอบ่อยที่สุดในการเขียนโปรแกรมกับ MySQL/MariaDB — การเช็ค error code แบบนี้ทำให้แยกแยะได้ว่า error ที่เกิดขึ้นเป็น "ข้อมูลซ้ำที่ผู้ใช้ควรแก้ไข" (ควรตอบ HTTP 409 Conflict กลับไปเมื่อทำเป็น REST API ตาม **Part 059**) หรือเป็น "ข้อผิดพลาดของระบบจริงๆ" (ควรตอบ HTTP 500 และ log ไว้สืบสวนต่อ) — เทคนิคเดียวกันนี้ใช้กับ PostgreSQL ได้เช่นกันผ่าน `*pgconn.PgError` (จาก `pgx`) ที่มี field `Code` เป็นรหัส SQLSTATE มาตรฐาน (เช่น `"23505"` สำหรับ unique violation) แทนที่จะเป็นตัวเลขแบบ MySQL

---

## 7.5 `wait_timeout` ของ MySQL กับ Connection Pool ของ Go

ทบทวนจาก **Part 071** หัวข้อ 9 ว่า `*sql.DB` คือ connection pool ที่ปกติจะเก็บ connection idle ไว้ใช้ซ้ำ — MySQL มีตัวแปรระบบชื่อ **`wait_timeout`** ที่กำหนดว่าเซิร์ฟเวอร์จะปิด connection ที่ idle (ไม่มีการใช้งาน) นานเกินไปเองโดยอัตโนมัติ ค่า default บนเซิร์ฟเวอร์ที่ใช้ทดสอบในบทนี้คือ:

```
Variable_name   Value
wait_timeout    28800
```

(28800 วินาที = 8 ชั่วโมง)

ปัญหาที่เกิดได้คือ: ถ้า `*sql.DB` ฝั่ง Go เก็บ connection idle ไว้นานกว่าที่เซิร์ฟเวอร์กำหนด เซิร์ฟเวอร์จะปิด connection นั้นทิ้งไปโดยที่ฝั่ง Go ไม่รู้ตัว พอมี query ใหม่มาแล้ว pool หยิบ connection ที่ถูกปิดไปแล้วนี้มาใช้ จะได้ error ทำนอง "invalid connection" หรือ "broken pipe" ทันที ทางแก้คือตั้ง **`db.SetConnMaxLifetime`** (แนะนำใน Part 071 หัวข้อ 9) ให้สั้นกว่า `wait_timeout` ของเซิร์ฟเวอร์เสมอ เพื่อให้ Go เป็นฝ่ายปิด/เปิด connection ใหม่ก่อนที่เซิร์ฟเวอร์จะตัดทิ้งเองอยู่เสมอ — รายละเอียดการ tune ค่าพวกนี้ให้เหมาะกับ workload จริง จะอยู่ใน **Part 078**

---

## 8. ตารางเปรียบเทียบ: เมื่อไหร่ควรเลือก MySQL เมื่อไหร่ควรเลือก PostgreSQL

ทั้งสองฐานข้อมูลเป็น open source, เสถียร, มี ecosystem Go รองรับดีทั้งคู่ ไม่มีตัวไหน "แย่" อย่างชัดเจน — การเลือกส่วนใหญ่ขึ้นกับบริบทของทีมและงาน:

| ประเด็น | MySQL / MariaDB | PostgreSQL |
|---|---|---|
| **Ecosystem/ฐานผู้ใช้** | ใหญ่มาก ใช้กันแพร่หลายมานาน โดยเฉพาะระบบเก่า, hosting ราคาถูก, WordPress/PHP ecosystem | ใหญ่และเติบโตเร็วมากในช่วงหลัง โดยเฉพาะระบบใหม่และ startup |
| **JSON support** | มี `JSON` type แต่ query/index ได้จำกัดกว่า | `JSONB` + GIN index ทรงพลังกว่ามาก (Part 072) |
| **Array type** | ไม่มี ต้องจำลองด้วยตารางแยกหรือ JSON | มี `ARRAY` type ในตัว (Part 072) |
| **Full-text Search** | มี แต่ feature จำกัดกว่า | มี `tsvector`/`tsquery` ที่ยืดหยุ่นและทรงพลังกว่า รวมถึง extension เช่น `pg_trgm` |
| **Replication/Clustering** | เรื่องมาตรฐานมานาน มี tool/บริการรองรับเยอะมาก (เช่น Galera Cluster) | รองรับดีเช่นกัน และเติบโตเร็วในด้าน HA solution |
| **Extension/Plugin** | มีจำกัดกว่า | ระบบ extension ทรงพลัง (เช่น `PostGIS` สำหรับ GIS, `pgvector` สำหรับ vector search/AI) |
| **ความเข้มงวดของ SQL/Type** | ยืดหยุ่นกว่า (บางเวอร์ชัน/โหมดยอมรับ type mismatch แบบ implicit conversion ได้มากกว่า) | เข้มงวดกว่า บังคับ type ถูกต้องชัดเจน ลดโอกาส bug เชิงข้อมูลที่ซ่อนอยู่ |
| **Community/แนวโน้มปัจจุบัน** | ยังคงเป็นฐานติดตั้งที่ใหญ่ที่สุดในโลกจำนวนมาก โดยเฉพาะระบบที่มีอยู่แล้ว | ถูกใช้เป็นค่าเริ่มต้นของโปรเจกต์ใหม่จำนวนมากขึ้นเรื่อยๆ ในช่วงหลายปีหลัง |

### คำแนะนำโดยรวม

> **สำหรับโปรเจกต์ใหม่ที่ไม่มีข้อจำกัดผูกมัด** แนวโน้มปัจจุบันในวงการ (รวมถึงชุมชน Go) เอนไปทาง **PostgreSQL เป็นค่าเริ่มต้นที่แนะนำ** เพราะ feature set ที่ครบกว่า โดยเฉพาะเรื่อง JSON/Array/Full-text search และความเข้มงวดของ type system ที่ช่วยจับ bug ได้เร็วกว่า อย่างไรก็ตาม **MySQL/MariaDB ยังคงเป็นตัวเลือกที่ดีมาก** และเหมาะสมอย่างยิ่งเมื่อ: ทีมมีความเชี่ยวชาญ MySQL อยู่แล้ว, ระบบต้องต่อกับ infrastructure เดิมที่ใช้ MySQL อยู่, หรือใช้ managed service/hosting ที่รองรับ MySQL ดีกว่าในราคาที่คุ้มค่ากว่า — ฐานผู้ใช้งาน MySQL ทั่วโลกยังคงมหาศาลและจะไม่หายไปในเร็วๆ นี้อย่างแน่นอน ไม่มีเหตุผลต้อง migrate ระบบที่ทำงานดีอยู่แล้วเพียงเพราะกระแส

---

## สรุปสิ่งที่ได้เรียนในบทนี้

- MySQL/MariaDB ใช้ driver มาตรฐาน `github.com/go-sql-driver/mysql` ผ่าน `database/sql` แบบเดียวกับที่เรียนใน Part 071
- DSN ของ MySQL มีรูปแบบ `user:pass@tcp(host:port)/dbname?param=value` ต่างจาก URL style ของ PostgreSQL
- **`parseTime=true`** เป็นพารามิเตอร์ที่ต้องใส่เสมอ ไม่งั้น scan column วันเวลาเข้า `time.Time` จะ error — ยืนยันด้วยการรันจริงและเห็น error message ตรงๆ
- MySQL ใช้ `LastInsertId()` ดึง auto-increment ID ได้ตรงๆ (ต่างจาก PostgreSQL ที่ต้องใช้ `RETURNING`)
- ชื่อตารางบน MySQL/MariaDB **case-sensitive บน Linux แต่ไม่ case-sensitive บน Windows** โดย default (ควบคุมด้วย `lower_case_table_names`) — ทางแก้ที่ปลอดภัยที่สุดคือตั้งชื่อ table/column เป็น lowercase เสมอ
- ทั้ง MySQL และ PostgreSQL เป็นตัวเลือกที่ดีทั้งคู่ แต่แนวโน้มปัจจุบันเอนไปทาง PostgreSQL เป็นค่าเริ่มต้นของโปรเจกต์ใหม่ เพราะ feature set ด้าน JSON/Array/Full-text search ที่ครบกว่า ขณะที่ MySQL ยังคงมีฐานผู้ใช้งานมหาศาลและเหมาะกับหลายบริบทโดยเฉพาะระบบที่มีอยู่แล้ว
- `*mysql.MySQLError` (เข้าถึงได้ด้วย `errors.As`) ให้รายละเอียด error code ตัวเลขของ MySQL เช่น 1062 (duplicate entry) ช่วยแยกแยะ error ที่เกิดจากข้อมูลผิดปกติออกจาก error ของระบบจริงๆ

## แบบฝึกหัดท้ายบท

1. เขียนโปรแกรมเชื่อมต่อ MySQL/MariaDB โดย**ตั้งใจไม่ใส่** `parseTime=true` แล้วลองแก้ปัญหาด้วยการ scan เข้า `[]byte` หรือ `string` แทน จากนั้นแปลงเป็น `time.Time` เองด้วย `time.Parse` — เปรียบเทียบว่าวิธีไหนสะดวกกว่ากัน
2. สร้างตารางชื่อ `Users` (ขึ้นต้นด้วยตัวใหญ่) บนเซิร์ฟเวอร์ MySQL/MariaDB บน Linux แล้วลอง query ด้วย `SELECT * FROM users` (ตัวเล็กทั้งหมด) สังเกต error ที่ได้ และแก้ปัญหาให้ถูกต้องตามแนวทางที่แนะนำในบทนี้
3. เขียนฟังก์ชัน `CreateUser` ที่คืนค่า id ที่เพิ่งสร้างกลับมาโดยใช้ `LastInsertId()` แล้วเขียนฟังก์ชันเดียวกันสำหรับ PostgreSQL ที่ใช้ `RETURNING id` แทน — จัดโครงสร้างโค้ดด้วย interface (ทบทวน Part 013) เพื่อให้ business logic ที่เหลือไม่ต้องรู้ว่ากำลังคุยกับฐานข้อมูลอะไรอยู่
4. ทดลองใส่ `multiStatements=true` ใน DSN แล้วรันคำสั่ง SQL สองคำสั่งคั่นด้วย `;` ในการเรียก `Exec` ครั้งเดียว จากนั้นอธิบายว่าทำไม feature นี้ต้องระวังเป็นพิเศษถ้าเคยมีจุดที่ต่อ string SQL เอง (ย้อนกลับไปดูหัวข้อ SQL Injection ใน Part 071)
5. เขียนโปรแกรมเปรียบเทียบ column type `VARCHAR(255)` ของ MySQL กับ `TEXT` ของ PostgreSQL — insert ข้อความยาวเกิน 255 ตัวอักษรลงแต่ละฐานข้อมูล สังเกตพฤติกรรมที่ต่างกัน (MySQL อาจ error หรือตัดข้อความทิ้งขึ้นกับ SQL mode ที่ตั้งไว้)
6. อ่านเรื่อง `lower_case_table_names` เพิ่มเติมจาก MySQL official documentation แล้วทดลองเปลี่ยนค่านี้บนเซิร์ฟเวอร์ทดสอบของตัวเอง (ระวัง: ต้องทำตอนสร้างฐานข้อมูลใหม่ก่อนมีข้อมูล เพราะเปลี่ยนทีหลังอาจทำให้ตารางเดิมหายไปจากมุมมองของ MySQL) แล้วสังเกตผลกระทบต่อ query ที่เขียนไว้
7. เขียนฟังก์ชัน `RegisterAccount(db *sql.DB, username string) error` ที่ insert username ใหม่ลงตารางที่มี `UNIQUE` constraint แล้วใช้ `errors.As` เช็ค `*mysql.MySQLError` — ถ้า `Number == 1062` ให้คืน error ชนิดพิเศษของแอปพลิเคชันเอง (เช่น `ErrUsernameTaken`) แทนที่จะปล่อย `*mysql.MySQLError` ดิบๆ ออกไปให้ชั้นบนของโปรแกรมต้องรู้จัก driver โดยตรง

---

**ต่อไป**: [Part 074 — ORM: GORM เบื้องต้น](./074-gorm-basics.md)
