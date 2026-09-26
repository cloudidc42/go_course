# Part 074: ORM: GORM เบื้องต้น

> ภาคที่ 6: ฐานข้อมูล — ตอนที่ 4 จาก 8 (Part 71–78)

## สารบัญของบทนี้

1. ORM คืออะไร และแก้ปัญหาอะไรจาก `database/sql`
2. ข้อแลกเปลี่ยนของ ORM: ที่ได้มาและสิ่งที่เสียไป
3. ติดตั้ง GORM และเลือก Driver
4. นิยาม Model: Struct Tags และ `gorm.Model`
5. `AutoMigrate`: สร้าง/ปรับ Schema จาก Struct อัตโนมัติ
6. Create: เพิ่มข้อมูล
7. Read: `First`, `Find`, และ `Where`
8. Update: `Save` และการอัปเดตแบบเจาะจง field
9. Delete: Soft Delete ผ่าน `DeletedAt`
10. ตัวอย่างเต็ม: CRUD ที่รันจริงกับ SQLite
11. สรุปสิ่งที่ได้เรียนในบทนี้
12. แบบฝึกหัดท้ายบท

---

## 1. ORM คืออะไร และแก้ปัญหาอะไรจาก `database/sql`

**ORM (Object-Relational Mapping)** คือเทคนิค/library ที่แปลงข้อมูลระหว่าง**ตารางในฐานข้อมูลเชิงสัมพันธ์**กับ**object/struct ในภาษาโปรแกรมมิ่ง**ให้อัตโนมัติ แทนที่จะต้องเขียน `rows.Scan(&a, &b, &c, ...)` เองทีละ field แบบที่เรียนใน **Part 071**

ใน Go, ORM ที่ได้รับความนิยมสูงสุดคือ **GORM** (`gorm.io/gorm`) — เป็น full-featured ORM ที่รองรับฐานข้อมูลหลักเกือบทุกตัว (PostgreSQL, MySQL, SQLite, SQL Server) ผ่าน driver แยกที่ใช้ API เดียวกันหมด

เทียบให้เห็นภาพชัดๆ ระหว่างสองวิธีทำสิ่งเดียวกัน — ดึง user ทั้งหมดจากตาราง:

**แบบ `database/sql` (Part 071):**

```go
rows, err := db.Query(`SELECT id, name, email FROM users`)
if err != nil {
	return nil, err
}
defer rows.Close()

var users []User
for rows.Next() {
	var u User
	if err := rows.Scan(&u.ID, &u.Name, &u.Email); err != nil {
		return nil, err
	}
	users = append(users, u)
}
if err := rows.Err(); err != nil {
	return nil, err
}
return users, nil
```

**แบบ GORM:**

```go
var users []User
result := db.Find(&users)
if result.Error != nil {
	return nil, result.Error
}
return users, nil
```

ความแตกต่างชัดเจนมาก: GORM ลด boilerplate การ scan/loop/close ที่ต้องเขียนซ้ำๆ ทุกครั้งลงเหลือแค่บรรทัดเดียว โดย mapping ชื่อ column กับชื่อ field ของ struct ให้อัตโนมัติผ่าน naming convention (หรือ struct tag ถ้าต้องการ override)

---

## 2. ข้อแลกเปลี่ยนของ ORM: ที่ได้มาและสิ่งที่เสียไป

ก่อนตัดสินใจใช้ ORM ในโปรเจกต์จริง ควรเข้าใจข้อแลกเปลี่ยนอย่างตรงไปตรงมา — ORM ไม่ใช่ "ดีกว่า `database/sql` เสมอไป" แต่เป็นเครื่องมือที่เหมาะกับบางสถานการณ์มากกว่าอีกสถานการณ์หนึ่ง

### สิ่งที่ได้มา

- **ลด boilerplate มหาศาล** โดยเฉพาะงาน CRUD พื้นฐานที่ทำซ้ำๆ ทั่วทั้งโปรเจกต์
- **Portable ข้ามฐานข้อมูลง่ายขึ้น** — สลับจาก SQLite ไป PostgreSQL อาจแค่เปลี่ยน driver บรรทัดเดียว (ในกรณีที่ไม่ได้ใช้ SQL เฉพาะทางของฐานข้อมูลใดฐานข้อมูลหนึ่ง)
- **Type safety ระดับหนึ่ง** — คอมไพเลอร์ช่วยจับ error บาง class ที่ raw SQL string จับไม่ได้ตอน compile time (แม้ query ที่ผิด logic ยังคง error ตอนรันอยู่ดี)
- **Feature ระดับสูงมาพร้อมตัว** เช่น associations/relations (**Part 075**), hooks, migration, soft delete — ที่ถ้าเขียนเองด้วย `database/sql` ต้องเขียนกลไกเหล่านี้ขึ้นมาเองทั้งหมด

### สิ่งที่เสียไป

- **ควบคุม SQL ที่ generate ออกมาได้น้อยลง** — บางครั้ง GORM generate query ที่ไม่ efficient เท่าที่เขียนมือ โดยเฉพาะ query ซับซ้อนที่มี join หลายชั้นหรือ subquery ซับซ้อน
- **ความเสี่ยงเรื่อง N+1 query** — ถ้าไม่เข้าใจกลไก eager loading (`Preload`) ดีพอ การ loop ดึงข้อมูลความสัมพันธ์ (relation) ทีละแถวจะยิง query แยกทุกครั้งโดยไม่รู้ตัว กลายเป็น query นับร้อยนับพันครั้งจากที่ควรเป็นแค่ 1-2 ครั้ง — ปัญหานี้สำคัญพอที่จะเป็นหัวข้อหลักของ **Part 075**
- **Debugging ยากขึ้นเล็กน้อย** — เวลา query ผิดพลาดหรือช้า ต้องเปิด log ของ GORM ดู SQL จริงที่ generate ออกมา แทนที่จะเห็น SQL ตรงๆ ในโค้ดเหมือน `database/sql`
- **"Magic" ที่ซ่อนพฤติกรรมบางอย่างไว้** — เช่น soft delete ที่ทำให้ `Delete()` ไม่ได้ลบแถวจริงๆ (ดูหัวข้อ 9) ถ้าไม่รู้กลไกนี้อาจสร้างความสับสนได้

> **แนวทางปฏิบัติที่สมดุล**: ทีมจำนวนมากเลือกใช้ GORM (หรือ ORM/query builder อื่น) สำหรับงาน CRUD พื้นฐาน 80% ของโปรเจกต์ ที่ไม่ซับซ้อนและได้ประโยชน์จากความเร็วในการพัฒนา แต่หลีกเลี่ยงมันสำหรับ query ที่ซับซ้อนหรือ performance-critical จริงๆ โดยใช้ raw SQL ผ่าน `db.Raw()` ของ GORM เอง (ซึ่งยังคงใช้ `*gorm.DB` connection pool เดียวกันได้) หรือหลุดไปใช้ `database/sql` ตรงๆ สำหรับจุดนั้นๆ

---

## 3. ติดตั้ง GORM และเลือก Driver

```bash
go get gorm.io/gorm
go get gorm.io/driver/sqlite    # สำหรับ SQLite
# go get gorm.io/driver/postgres  # สำหรับ PostgreSQL (API เดียวกันทุกประการ)
# go get gorm.io/driver/mysql     # สำหรับ MySQL (API เดียวกันทุกประการ)
```

บทนี้และ **Part 075** จะใช้ **SQLite** เป็นฐานข้อมูลสาธิตหลัก ผ่าน driver `gorm.io/driver/sqlite` (ซึ่งภายในใช้ `github.com/mattn/go-sqlite3` — ไลบรารีที่พึ่ง cgo เพื่อผูกกับ SQLite engine ต้นฉบับที่เขียนด้วยภาษา C) เหตุผลที่เลือก SQLite สำหรับสาธิต GORM คือ **ไม่ต้องมีเซิร์ฟเวอร์ฐานข้อมูลแยกต่างหาก** ฐานข้อมูลทั้งก้อนเป็นแค่ไฟล์เดียวบน disk ทำให้ทุกตัวอย่างในบทนี้**รันได้จริงและตรวจสอบซ้ำได้ทันที**ในทุกสภาพแวดล้อมโดยไม่ต้องพึ่งการติดตั้งเซิร์ฟเวอร์เพิ่ม

สิ่งสำคัญที่ต้องเข้าใจคือ **API ของ GORM เหมือนกันทุกตัวอักษรไม่ว่าจะใช้ driver ไหน** — โค้ดทั้งหมดในบทนี้ (`db.Create`, `db.Find`, `db.Where`, ...) จะทำงานเหมือนเดิมทุกประการถ้าเปลี่ยนบรรทัดเดียวจาก `sqlite.Open(...)` เป็น `postgres.Open(dsn)` หรือ `mysql.Open(dsn)` (สมมติว่าไม่ได้ใช้ SQL เฉพาะทางของฐานข้อมูลใดฐานข้อมูลหนึ่งผ่าน raw SQL) — นี่คือคุณค่าหลักของ ORM ในแง่ portability ที่กล่าวถึงในหัวข้อ 2

```go
import (
	"gorm.io/driver/sqlite"
	"gorm.io/gorm"
)

db, err := gorm.Open(sqlite.Open("app.db"), &gorm.Config{})
```

> **หมายเหตุความซื่อสัตย์**: ทุกตัวอย่างในบทนี้และ Part 075 **รันจริง** กับ SQLite ผ่าน `gorm.io/driver/sqlite` (GORM v1.31, driver v1.6) ในสภาพแวดล้อมที่ใช้เขียนหลักสูตรนี้ ผลลัพธ์ที่แสดงคือ output จริงจากการรันโค้ด API ของ GORM ที่แสดงในบทนี้ทำงานเหมือนกันกับ driver postgres/mysql ตาม documentation ทางการของ GORM แต่การรันข้าม driver จริงไม่ได้ทำซ้ำในบทนี้เพราะ SQLite เพียงพอต่อการสาธิตกลไกของ GORM เอง ซึ่งเป็นสิ่งที่ไม่ขึ้นกับฐานข้อมูลเบื้องหลัง

---

## 4. นิยาม Model: Struct Tags และ `gorm.Model`

GORM แปลง struct ธรรมดาให้เป็นตารางในฐานข้อมูลโดยอาศัย **naming convention** และ **struct tag** เพิ่มเติม:

```go
type Book struct {
	gorm.Model
	Title  string  `gorm:"size:255;not null"`
	Author string  `gorm:"size:120;index"`
	Price  float64
}
```

### `gorm.Model`: struct สำเร็จรูปที่ embed ได้

`gorm.Model` เป็น struct ที่ GORM เตรียมไว้ให้ (ทบทวนเทคนิค struct embedding จาก **Part 030**) ประกอบด้วย 4 field มาตรฐานที่เกือบทุกตารางต้องการ:

```go
type Model struct {
	ID        uint `gorm:"primarykey"`
	CreatedAt time.Time
	UpdatedAt time.Time
	DeletedAt gorm.DeletedAt `gorm:"index"`
}
```

- **`ID`** — primary key แบบ auto-increment ตั้งค่าอัตโนมัติ
- **`CreatedAt`** — GORM set ค่าให้อัตโนมัติตอน `Create` เท่านั้น
- **`UpdatedAt`** — GORM set ค่าใหม่ให้อัตโนมัติทุกครั้งที่ `Save`/`Update`
- **`DeletedAt`** — ใช้สำหรับกลไก **soft delete** (ดูหัวข้อ 9) — ค่าเป็น `NULL` ตราบใดที่แถวยังไม่ถูกลบ

การ embed `gorm.Model` เข้าไปใน struct ของเรา ทำให้ทั้ง 4 field นี้ถูก "ยก" ขึ้นมาเป็น field ของ `Book` โดยตรง (ทบทวนกฎ embedding promotion จาก Part 030) เข้าถึงได้ผ่าน `book.ID`, `book.CreatedAt` ได้ทันทีโดยไม่ต้องเขียน `book.Model.ID`

### Struct Tag ที่ใช้บ่อย

| Tag | ความหมาย |
|---|---|
| `gorm:"primarykey"` | กำหนดให้ field นี้เป็น primary key |
| `gorm:"size:255"` | ความยาวสูงสุดของ column (แปลงเป็น `VARCHAR(255)` เป็นต้น) |
| `gorm:"not null"` | เพิ่ม `NOT NULL` constraint |
| `gorm:"uniqueIndex"` | สร้าง unique index บน column นี้ |
| `gorm:"index"` | สร้าง index ธรรมดา (ไม่ unique) |
| `gorm:"column:custom_name"` | ตั้งชื่อ column ให้ต่างจากชื่อ field |
| `gorm:"-"` | บอก GORM ให้ข้าม field นี้ ไม่ persist ลงฐานข้อมูล |
| `gorm:"default:0"` | ค่า default ของ column ในฐานข้อมูล |

ถ้าไม่ระบุ tag ใดๆ เลย GORM จะแปลงชื่อ struct field เป็น **snake_case** โดยอัตโนมัติเป็นชื่อ column (เช่น `Title` → `title`, `AuthorID` → `author_id`) และแปลงชื่อ struct เป็นพหูพจน์ snake_case เป็นชื่อตาราง (เช่น struct `Book` → ตาราง `books`)

---

## 5. `AutoMigrate`: สร้าง/ปรับ Schema จาก Struct อัตโนมัติ

`db.AutoMigrate` อ่านโครงสร้าง struct แล้วสร้างตารางให้อัตโนมัติถ้ายังไม่มี หรือ**เพิ่ม column ใหม่**ให้ถ้าตารางมีอยู่แล้วแต่ struct มี field ใหม่เพิ่มมา (มันจะไม่ลบ column เก่าที่ไม่มีใน struct แล้วออกให้ — ปลอดภัยจากการทำข้อมูลหายโดยไม่ตั้งใจ):

```go
if err := db.AutoMigrate(&Book{}); err != nil {
	log.Fatal("AutoMigrate:", err)
}
```

รันจริง:

```
AutoMigrate ตาราง books สำเร็จ
```

`AutoMigrate` เหมาะมากสำหรับช่วง**พัฒนา (development)** ที่ schema เปลี่ยนบ่อย แต่**ไม่แนะนำให้ใช้เป็นกลไกหลักในการจัดการ schema ของระบบ production** เพราะมันไม่รองรับการ rollback, ไม่มี migration history ที่ตรวจสอบย้อนหลังได้, และจัดการการเปลี่ยนแปลงที่ซับซ้อน (เช่น เปลี่ยนชื่อ column, เปลี่ยน type ของ column ที่มีข้อมูลอยู่แล้ว) ได้ไม่ดีนัก — สำหรับ production ทีมส่วนใหญ่ใช้เครื่องมือ migration เฉพาะทางแทน ซึ่งจะพูดถึงใน **Part 075 หัวข้อ Migration ขั้นสูง**

---

## 6. Create: เพิ่มข้อมูล

```go
book := Book{Title: "Learning Go", Author: "Jon Bodner", Price: 450}
if err := db.Create(&book).Error; err != nil {
	log.Fatal("create:", err)
}
fmt.Printf("สร้างหนังสือ id=%d CreatedAt=%s\n", book.ID, book.CreatedAt.Format(time.RFC3339))
```

ผลลัพธ์จริง:

```
สร้างหนังสือ id=1 CreatedAt=2026-09-26T02:52:25Z
```

สังเกตว่า **`db.Create(&book)` เขียนค่า `ID` และ `CreatedAt`/`UpdatedAt` กลับเข้าไปใน struct `book` ให้อัตโนมัติ** หลัง insert สำเร็จ — เป็นพฤติกรรมคล้ายกับ `RETURNING` ของ PostgreSQL ที่เรียนใน **Part 072** (GORM ใช้กลไก `RETURNING` จริงๆ เบื้องหลังเมื่อฐานข้อมูลรองรับ เช่น PostgreSQL/SQLite และใช้ `LastInsertId()` เมื่อเป็น MySQL — ผู้ใช้ไม่ต้องสนใจความต่างนี้เลย เพราะ GORM จัดการให้ตาม driver ที่ใช้อยู่)

`db.Create` ทุกครั้งคืนค่าเป็น `*gorm.DB` เสมอ (ไม่ใช่ `error` ตรงๆ) — pattern มาตรฐานของ GORM คือเช็คผ่าน field `.Error`:

```go
result := db.Create(&book)
if result.Error != nil {
	// จัดการ error
}
fmt.Println("จำนวนแถวที่ถูกกระทบ:", result.RowsAffected)
```

---

## 7. Read: `First`, `Find`, และ `Where`

### `First`: ดึงแถวแรก (เรียงตาม primary key)

```go
var first Book
db.First(&first)
fmt.Println("First():", first.Title, "โดย", first.Author)
```

ผลลัพธ์จริง:

```
First(): Learning Go โดย Jon Bodner
```

### `Find`: ดึงหลายแถว

```go
var all []Book
db.Find(&all)
for _, b := range all {
	fmt.Printf("  #%d %-30s %-25s ฿%.2f\n", b.ID, b.Title, b.Author, b.Price)
}
```

ผลลัพธ์จริง (มี 3 เล่มในฐานข้อมูล):

```
Find() ทั้งหมด:
  #1 Learning Go                    Jon Bodner                ฿450.00
  #2 The Go Programming Language    Donovan & Kernighan       ฿390.00
  #3 Concurrency in Go              Katherine Cox-Buday       ฿520.00
```

### `Where`: กรองเงื่อนไข

```go
var cheap []Book
db.Where("price < ?", 400).Find(&cheap)
```

ผลลัพธ์จริง:

```
Where price < 400:
  The Go Programming Language ฿390.00
```

สังเกตว่า `Where` ยังคงใช้ **placeholder `?` แบบ parameterized query** เหมือนที่เรียนใน Part 071 — GORM ไม่ได้ยกเว้นกฎเรื่อง SQL Injection แต่อย่างใด การส่งค่าผ่าน `?` ยังคงจำเป็นเสมอ ไม่ควรต่อ string เงื่อนไขเองแม้จะใช้ ORM อยู่ก็ตาม

---

## 8. Update: `Save` และการอัปเดตแบบเจาะจง field

### `Save`: บันทึกค่าทุก field ของ struct กลับไป

```go
first.Price = 480
if err := db.Save(&first).Error; err != nil {
	log.Fatal("save:", err)
}
fmt.Printf("Update ราคาเล่ม %q เป็น %.2f สำเร็จ (UpdatedAt=%s)\n",
	first.Title, first.Price, first.UpdatedAt.Format(time.RFC3339))
```

ผลลัพธ์จริง:

```
Update ราคาเล่ม "Learning Go" เป็น 480.00 สำเร็จ (UpdatedAt=2026-09-26T02:52:25Z)
```

`Save` จะอัปเดต**ทุก field** ของ struct กลับลงฐานข้อมูล (ถ้า struct มี primary key อยู่แล้วจะเป็นการ `UPDATE`, ถ้ายังไม่มีจะเป็นการ `INSERT`) และตั้งค่า `UpdatedAt` ใหม่ให้อัตโนมัติเสมอ

ถ้าต้องการอัปเดตแค่บาง field โดยไม่กระทบ field อื่น ใช้ `Model(...).Updates(...)` แทน:

```go
db.Model(&Book{}).Where("id = ?", first.ID).Updates(map[string]any{"price": 480})
// หรือใช้ struct (จะข้าม zero value โดยอัตโนมัติ ยกเว้นใช้ Select เจาะจง)
db.Model(&Book{}).Where("id = ?", first.ID).Updates(Book{Price: 480})
```

---

## 9. Delete: Soft Delete ผ่าน `DeletedAt`

นี่คือพฤติกรรม default ของ GORM ที่สำคัญมากและมือใหม่มักไม่ทันสังเกต: **ถ้า struct embed `gorm.Model` (ซึ่งมี field `DeletedAt gorm.DeletedAt`) การเรียก `db.Delete(...)` จะไม่ลบแถวออกจากฐานข้อมูลจริงๆ** แต่จะแค่ตั้งค่า column `deleted_at` เป็นเวลาปัจจุบันแทน — เรียกว่า **soft delete**

```go
if err := db.Delete(&first).Error; err != nil {
	log.Fatal("delete:", err)
}
fmt.Println("ลบเล่มแรกแล้ว (soft delete ผ่าน DeletedAt)")
```

ผลลัพธ์จริง:

```
ลบเล่มแรกแล้ว (soft delete ผ่าน DeletedAt)
```

หลังจากนั้น `Find`/`First` ปกติจะ**ไม่แสดง**แถวที่ soft-delete แล้วโดยอัตโนมัติ (GORM แอบเติม `WHERE deleted_at IS NULL` ให้ทุก query):

```go
var afterDelete []Book
db.Find(&afterDelete)
```

ผลลัพธ์จริง (เหลือแค่ 2 เล่ม จากทั้งหมด 3 เล่ม):

```
Find() หลังลบ (ไม่เห็นแถวที่ soft-delete แล้ว):
  #2 The Go Programming Language
  #3 Concurrency in Go
```

ถ้าต้องการเห็นแม้แถวที่ถูก soft-delete ไปแล้ว (เช่น ทำหน้า "ถังขยะ" กู้คืนข้อมูล) ใช้ `.Unscoped()`:

```go
var withDeleted []Book
db.Unscoped().Find(&withDeleted)
```

ผลลัพธ์จริง (เห็นครบ 3 เล่ม พร้อมระบุว่าเล่มไหนถูกลบแล้ว):

```
Find() ด้วย Unscoped() (เห็นแม้ที่ถูก soft-delete แล้ว):
  #1 Learning Go                    deleted=yes, deleted_at=2026-09-26T02:52:25Z
  #2 The Go Programming Language    deleted=no
  #3 Concurrency in Go              deleted=no
```

การลบแบบถาวรจริงๆ (**hard delete**) ทำได้โดยเรียก `Unscoped().Delete(...)`:

```go
db.Unscoped().Delete(&first) // ลบแถวออกจากฐานข้อมูลจริง ไม่สามารถกู้คืนได้
```

> **ข้อควรระวังสำคัญ**: ถ้า struct **ไม่ได้** embed `gorm.Model` และไม่มี field ชนิด `gorm.DeletedAt` เลย `db.Delete(...)` จะเป็น **hard delete ทันที** ไม่มี soft delete ให้ — พฤติกรรมนี้ขึ้นอยู่กับว่า struct มี field `DeletedAt` หรือไม่เท่านั้น ไม่ใช่ configuration แยกต่างหาก จึงควรตรวจสอบ struct ของตัวเองให้แน่ใจเสมอว่าตั้งใจให้มีหรือไม่มี soft delete

---

## 10. ตัวอย่างเต็ม: CRUD ที่รันจริงกับ SQLite

รวมทุกหัวข้อของบทนี้ไว้ในโปรแกรมเดียว — รันจริงทั้งหมด ผลลัพธ์คือ output จริง:

```go
package main

import (
	"fmt"
	"log"
	"time"

	"gorm.io/driver/sqlite"
	"gorm.io/gorm"
	"gorm.io/gorm/logger"
)

type Book struct {
	gorm.Model
	Title  string `gorm:"size:255;not null"`
	Author string `gorm:"size:120;index"`
	Price  float64
}

func main() {
	db, err := gorm.Open(sqlite.Open("gormdemo.db"), &gorm.Config{
		Logger: logger.Default.LogMode(logger.Silent), // ปิด log SQL ที่ GORM พิมพ์ตามปกติ
	})
	if err != nil {
		log.Fatal("gorm.Open:", err)
	}

	if err := db.AutoMigrate(&Book{}); err != nil {
		log.Fatal("AutoMigrate:", err)
	}

	// CREATE
	book := Book{Title: "Learning Go", Author: "Jon Bodner", Price: 450}
	db.Create(&book)
	db.Create(&Book{Title: "The Go Programming Language", Author: "Donovan & Kernighan", Price: 390})
	db.Create(&Book{Title: "Concurrency in Go", Author: "Katherine Cox-Buday", Price: 520})

	// READ
	var first Book
	db.First(&first)

	var cheap []Book
	db.Where("price < ?", 400).Find(&cheap)

	// UPDATE
	first.Price = 480
	db.Save(&first)

	// DELETE (soft delete)
	db.Delete(&first)

	var remaining []Book
	db.Find(&remaining)
	fmt.Println("หนังสือที่เหลือ (ไม่นับที่ถูกลบ):", len(remaining))

	var all []Book
	db.Unscoped().Find(&all)
	fmt.Println("หนังสือทั้งหมดรวมที่ถูกลบ:", len(all))
}
```

ตัวอย่างนี้ครอบคลุมวงจรชีวิตพื้นฐานของ ORM ครบทั้ง 4 การกระทำ (Create, Read, Update, Delete) พร้อมกลไก soft delete ที่เป็นค่าเริ่มต้นของ GORM — ใน **Part 075** เราจะขยายจากนี้ไปสู่ความสัมพันธ์ระหว่างตาราง (relations), การแก้ปัญหา N+1 query, transaction, hooks, และ scopes

---

## สรุปสิ่งที่ได้เรียนในบทนี้

- ORM อย่าง GORM ช่วยลด boilerplate ของการ scan ข้อมูลเข้า struct ที่ต้องเขียนเองใน `database/sql` แต่แลกมาด้วยการควบคุม SQL ที่ generate ออกมาได้น้อยลง และความเสี่ยงเรื่อง N+1 query ถ้าใช้ไม่ถูกวิธี
- ติดตั้งผ่าน `gorm.io/gorm` + driver แยกตามฐานข้อมูล (`gorm.io/driver/sqlite`, `postgres`, `mysql`) — API เหมือนกันทุกตัวไม่ว่าจะใช้ driver ไหน
- นิยาม model ด้วย struct ธรรมดา + struct tag `gorm:"..."`, ใช้ `gorm.Model` (embed) เพื่อได้ `ID`, `CreatedAt`, `UpdatedAt`, `DeletedAt` มาตรฐานฟรี
- `db.AutoMigrate` สร้าง/ปรับตารางจาก struct อัตโนมัติ เหมาะกับ development แต่ไม่เหมาะเป็นกลไกหลักของ production
- CRUD พื้นฐาน: `Create`, `First`/`Find`/`Where`, `Save`, `Delete` — ทุก method คืน `*gorm.DB` เช็ค error ผ่าน `.Error`
- ถ้า struct มี field `DeletedAt gorm.DeletedAt` (ผ่าน `gorm.Model`) `Delete` จะเป็น **soft delete** โดยอัตโนมัติ ต้องใช้ `Unscoped()` เพื่อเห็นหรือลบแถวที่ถูก soft-delete ไปแล้วอย่างถาวร
- ตัวอย่างทั้งหมดในบทนี้รันจริงกับ SQLite ผ่าน `gorm.io/driver/sqlite` ไม่ใช่โค้ดที่ไม่ได้ทดสอบ

## แบบฝึกหัดท้ายบท

1. เขียน struct `Customer` ที่ embed `gorm.Model` มี field `Name`, `Email` (ที่มี `uniqueIndex`), และ `Phone` แล้วเขียนโปรแกรม `AutoMigrate` + insert ลูกค้า 3 คน
2. ทดลอง insert ลูกค้าที่มี `Email` ซ้ำกับที่มีอยู่แล้ว สังเกต error ที่ได้ (คำใบ้: จะคล้ายกับ error 1062 ของ MySQL ใน Part 073 หรือ unique violation ของ PostgreSQL ใน Part 072 ขึ้นกับ driver ที่ใช้)
3. เขียนฟังก์ชันที่ใช้ `db.Model(&Book{}).Where("id = ?", id).Updates(map[string]any{"price": newPrice})` เปรียบเทียบกับการใช้ `db.Save(&book)` — อธิบายว่ากรณีไหนควรใช้แบบไหน (คำใบ้: ลองนึกถึงกรณีที่ struct มี field ที่ไม่อยากให้ถูกอัปเดตทับโดยไม่ตั้งใจ)
4. เขียนโปรแกรมที่ soft-delete หนังสือไปแล้วครึ่งหนึ่งของทั้งหมด จากนั้นเขียนฟังก์ชัน `RestoreBook(db *gorm.DB, id uint) error` ที่กู้คืนหนังสือที่ถูก soft-delete กลับมา (คำใบ้: ใช้ `Unscoped().Model(...).Update("deleted_at", nil)`)
5. ลองสร้าง struct ที่**ไม่**embed `gorm.Model` แต่กำหนด field `ID uint gorm:"primarykey"` เอง แล้วลอง `Delete` ดู สังเกตว่าเป็น hard delete ทันทีหรือไม่ พร้อมอธิบายเหตุผล
6. เปลี่ยน driver จาก `sqlite.Open("gormdemo.db")` เป็นการเชื่อมต่อ PostgreSQL จริงที่ตั้งค่าไว้ใน Part 072 (`postgres.Open(dsn)` จาก `gorm.io/driver/postgres`) แล้วรันโปรแกรมเดิมในหัวข้อ 10 ซ้ำ ยืนยันว่า API และผลลัพธ์เหมือนกันทุกประการโดยไม่ต้องแก้โค้ดส่วน logic เลย

---

**ต่อไป**: [Part 075 — GORM ขั้นสูง: Relations, Migrations, Hooks](./075-gorm-advanced.md)
