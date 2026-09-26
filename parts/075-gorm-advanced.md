# Part 075: GORM ขั้นสูง: Relations, Migrations, Hooks

> ภาคที่ 6: ฐานข้อมูล — ตอนที่ 5 จาก 8 (Part 71–78)

## สารบัญของบทนี้

1. Associations: `belongs to`, `has many`, `has one`, `many to many`
2. ปัญหา N+1 Query: สาธิตให้เห็นจริงว่าเกิดอะไรขึ้น
3. แก้ปัญหา N+1 ด้วย `Preload`: Eager Loading
4. Many-to-Many ผ่านตารางกลาง
5. Transactions: `db.Transaction`
6. Hooks: `BeforeCreate`, `AfterUpdate` และการ Hash Password อัตโนมัติ
7. Scopes: Reusable Query Fragments
8. Migration ที่มากกว่า `AutoMigrate`: `golang-migrate` สำหรับ Production
9. ตัวอย่างเต็ม: รวมทุก Feature ที่รันจริงกับ SQLite
10. สรุปสิ่งที่ได้เรียนในบทนี้
11. แบบฝึกหัดท้ายบท

> **หมายเหตุความซื่อสัตย์**: ทุกตัวอย่างในบทนี้ (รวมถึงการนับจำนวน query ในหัวข้อ N+1) **รันจริง** กับ SQLite ผ่าน GORM v1.31 ในสภาพแวดล้อมที่ใช้เขียนหลักสูตรนี้ ตัวเลขจำนวน query ที่แสดงเป็นตัวเลขจริงที่นับได้จากการรันโค้ด ไม่ใช่ค่าประมาณ

---

## 1. Associations: `belongs to`, `has many`, `has one`, `many to many`

GORM รองรับความสัมพันธ์ระหว่างตาราง 4 แบบหลัก ตรงกับแนวคิดมาตรฐานของฐานข้อมูลเชิงสัมพันธ์:

| ความสัมพันธ์ | ความหมาย | ตัวอย่าง |
|---|---|---|
| **Belongs To** | struct นี้ "เป็นของ" อีก struct หนึ่ง (มี foreign key อยู่ในตัวเอง) | `Post` เป็นของ `Author` หนึ่งคน |
| **Has One** | struct นี้มีอีก struct หนึ่งแบบ 1:1 (foreign key อยู่ฝั่งตรงข้าม) | `User` มี `Profile` เดียว |
| **Has Many** | struct นี้มีอีก struct หลายตัว (foreign key อยู่ฝั่งตรงข้าม) | `Author` มี `Post` หลายชิ้น |
| **Many to Many** | ทั้งสองฝั่งมีกันได้หลายตัว ผ่านตารางกลาง (join table) | `Post` มีได้หลาย `Tag`, `Tag` หนึ่งอยู่ใน `Post` ได้หลายอัน |

นิยามด้วย struct ธรรมดา โดย GORM เดา foreign key จาก naming convention (`<TypeName>ID`) ให้อัตโนมัติ:

```go
type Author struct {
	gorm.Model
	Name  string
	Posts []Post // has many: Author หนึ่งคนมีหลาย Post
}

type Post struct {
	gorm.Model
	Title    string
	Body     string
	AuthorID uint  // belongs to: foreign key ชี้กลับไปที่ Author
	Tags     []Tag `gorm:"many2many:post_tags;"` // many-to-many ผ่านตารางกลาง post_tags
}

type Tag struct {
	gorm.Model
	Name string
}
```

GORM มองเห็นว่า `Post.AuthorID` ตรงกับ pattern `<TypeName>ID` จึงรู้เองว่านี่คือ foreign key ที่โยงกับ `Author.ID` โดยไม่ต้องเขียน tag อะไรเพิ่ม (ถ้าต้องการ override ชื่อ foreign key ใช้ tag `gorm:"foreignKey:..."` ได้)

---

## 2. ปัญหา N+1 Query: สาธิตให้เห็นจริงว่าเกิดอะไรขึ้น

**N+1 query problem** คือหนึ่งในกับดักที่พบบ่อยที่สุดเวลาใช้ ORM ใดๆ ก็ตาม ไม่ใช่แค่ GORM — ปัญหาคือ เมื่อดึงข้อมูล parent มา N แถว แล้ว loop ดึงข้อมูล relation ของแต่ละแถวแยกต่างหากทีละครั้ง จะกลายเป็นการยิง query ทั้งหมด **N+1 ครั้ง** (1 ครั้งสำหรับดึง parent, บวกอีก N ครั้งสำหรับดึง relation ของแต่ละแถว) แทนที่จะเป็นแค่ 1-2 ครั้ง

มาดูปัญหานี้เกิดขึ้นจริง — เตรียมข้อมูล author 3 คน คนละ 2 posts:

```go
for i := 1; i <= 3; i++ {
	author := Author{Name: fmt.Sprintf("นักเขียนคนที่ %d", i)}
	db.Create(&author)
	db.Create(&Post{Title: fmt.Sprintf("บทความ %d-1", i), AuthorID: author.ID})
	db.Create(&Post{Title: fmt.Sprintf("บทความ %d-2", i), AuthorID: author.ID})
}
```

จากนั้นลอง**ไม่ใช้** `Preload` เลย และ loop ดึง posts ของแต่ละคนแยกต่างหาก (วิธีที่มือใหม่มักเขียนโดยไม่รู้ตัวว่ากำลังสร้างปัญหา):

```go
var authorsNoPreload []Author
countingDB.Find(&authorsNoPreload) // query #1: ดึง author ทั้งหมด

for _, a := range authorsNoPreload {
	var posts []Post
	countingDB.Where("author_id = ?", a.ID).Find(&posts) // query แยกทุกครั้ง — นี่คือ N+1!
	fmt.Printf("  %s มี %d posts (query แยกต่างหาก)\n", a.Name, len(posts))
}
```

เราวัดจำนวน query จริงด้วยการแทน logger ของ GORM ด้วยตัวนับ (`countingDB` คือ session ของ `db` ที่ผูก logger นับจำนวนครั้งที่มีการ log SQL) ผลลัพธ์จริงที่ได้:

```
=== ไม่ใช้ Preload (เกิด N+1 query) ===
  นักเขียนคนที่ 1 มี 2 posts (query แยกต่างหาก)
  นักเขียนคนที่ 2 มี 2 posts (query แยกต่างหาก)
  นักเขียนคนที่ 3 มี 2 posts (query แยกต่างหาก)
จำนวน query ทั้งหมด: 4 (1 สำหรับ authors + 3 สำหรับ posts ของแต่ละคน)
```

ยืนยันตัวเลขจริง: **4 query** สำหรับ author แค่ 3 คน (1 query ดึง author + 3 query แยกดึง posts ของแต่ละคน) — ลองจินตนาการว่าถ้ามี author 10,000 คน จะกลายเป็น **10,001 query** ทันที ซึ่งในระบบจริงที่มีข้อมูลปริมาณมากๆ ปัญหานี้ทำให้ response time พังยับได้ง่ายมาก โดยที่โค้ดดูเหมือนทำงานถูกต้องตอนทดสอบด้วยข้อมูลน้อยๆ

---

## 3. แก้ปัญหา N+1 ด้วย `Preload`: Eager Loading

วิธีแก้คือใช้ **`Preload`** เพื่อบอก GORM ให้ดึงข้อมูล relation มาพร้อมกันตั้งแต่ต้น (เรียกว่า **eager loading**) แทนที่จะดึงทีหลังแบบ **lazy loading** ทีละแถว:

```go
var authorsPreload []Author
countingDB2.Preload("Posts").Find(&authorsPreload)

for _, a := range authorsPreload {
	fmt.Printf("  %s มี %d posts (มาพร้อมกันจาก Preload)\n", a.Name, len(a.Posts))
}
```

ผลลัพธ์จริง:

```
=== ใช้ Preload (eager load ในคำสั่งเดียว) ===
  นักเขียนคนที่ 1 มี 2 posts (มาพร้อมกันจาก Preload)
  นักเขียนคนที่ 2 มี 2 posts (มาพร้อมกันจาก Preload)
  นักเขียนคนที่ 3 มี 2 posts (มาพร้อมกันจาก Preload)
จำนวน query ทั้งหมด: 2 (query เดียวสำหรับ authors + query เดียวสำหรับ posts ทั้งหมดด้วย WHERE author_id IN (...))
```

ยืนยันตัวเลขจริง: จาก 4 query เหลือแค่ **2 query** เท่านั้น ไม่ว่าจะมี author กี่คนก็ตาม (query แรกดึง author ทั้งหมด, query ที่สองดึง post **ทั้งหมด**ของ author ทุกคนในครั้งเดียวด้วย `WHERE author_id IN (1, 2, 3, ...)` แล้ว GORM จับคู่ผลลัพธ์กลับเข้า field `Posts` ของแต่ละ `Author` ให้อัตโนมัติในหน่วยความจำ) — จาก 4 เป็น 2 อาจดูไม่มาก แต่ถ้าเพิ่ม author เป็น 10,000 คน ตัวเลขจะยังคงเป็น **2 query เท่าเดิม** ต่างจากวิธีแรกที่จะกลายเป็น 10,001 query — นี่คือความแตกต่างที่ทำให้ระบบ scale ได้จริงหรือไม่ในทางปฏิบัติ

> **กฎที่ควรจำ**: ทุกครั้งที่ต้องเข้าถึงข้อมูล relation ของ struct ที่ดึงมาเป็น list (ไม่ว่าจะเป็น `belongs to`, `has many`, `has one`, หรือ `many to many`) ให้ใช้ `Preload("ชื่อField")` เสมอ **ก่อน** `Find`/`First` ไม่ใช่ไป loop ดึงทีหลัง หากมี relation ซ้อนกันหลายชั้น (เช่น `Author` → `Post` → `Comment`) ใช้ dot notation ได้: `Preload("Posts.Comments")`

---

## 4. Many-to-Many ผ่านตารางกลาง

ความสัมพันธ์แบบ many-to-many ใช้ tag `gorm:"many2many:ชื่อตารางกลาง;"` — GORM จะสร้างตารางกลางให้อัตโนมัติตอน `AutoMigrate` โดยไม่ต้องนิยาม struct ของตารางกลางเองเลย (เว้นแต่ต้องการเก็บข้อมูลเพิ่มเติมในความสัมพันธ์ เช่น วันที่ผูก tag):

```go
type Post struct {
	gorm.Model
	Title string
	Tags  []Tag `gorm:"many2many:post_tags;"`
}
```

การผูกความสัมพันธ์ทำผ่าน `Association`:

```go
var firstPost Post
db.First(&firstPost)
db.Model(&firstPost).Association("Tags").Append(&Tag{Name: "golang"}, &Tag{Name: "database"})

var reloaded Post
db.Preload("Tags").First(&reloaded, firstPost.ID)
```

ผลลัพธ์จริง:

```
Post "บทความ 1-1" มี tags: golang database
```

สังเกตว่าการดึง `Tags` กลับมาก็ยังต้องใช้ `Preload("Tags")` เช่นเดียวกับ `has many` เพื่อหลีกเลี่ยง N+1 — หลักการเดียวกันเป๊ะ ไม่ว่าจะเป็นความสัมพันธ์แบบไหน

---

## 5. Transactions: `db.Transaction`

GORM มี wrapper สะดวกสำหรับ transaction ที่ commit/rollback ให้อัตโนมัติตามค่าที่ callback คืนกลับมา:

```go
err = db.Transaction(func(tx *gorm.DB) error {
	newAuthor := Author{Name: "นักเขียนใน transaction"}
	if err := tx.Create(&newAuthor).Error; err != nil {
		return err // return error ใดๆ = rollback อัตโนมัติ
	}
	if err := tx.Create(&Post{Title: "โพสต์ใน transaction", AuthorID: newAuthor.ID}).Error; err != nil {
		return err
	}
	return nil // return nil = commit อัตโนมัติ
})
```

**กฎสำคัญ**: ภายใน callback ต้องใช้ `tx` (ตัวแปรที่ GORM ส่งเข้ามาให้) ในการ query/exec ทุกจุด **ห้ามใช้ `db` ตัวนอก** เพราะ `db` ตัวนอกไม่ได้อยู่ใน transaction เดียวกัน การเผลอใช้ `db` แทน `tx` เป็น bug ที่พบบ่อยมาก — ผลคือคำสั่งนั้นจะ commit ทันทีแยกจาก transaction หลัก ทำให้ atomicity ที่ควรจะได้พังไปเงียบๆ โดยไม่มี error ใดๆ เตือน

ทดสอบว่า rollback ทำงานจริงหรือไม่ ด้วยการจงใจคืน error กลางทาง:

```go
var countBefore int64
db.Model(&Author{}).Count(&countBefore)

err = db.Transaction(func(tx *gorm.DB) error {
	if err := tx.Create(&Author{Name: "จะถูก rollback"}).Error; err != nil {
		return err
	}
	return fmt.Errorf("จำลอง error กลางทาง เพื่อทดสอบ rollback")
})

var countAfter int64
db.Model(&Author{}).Count(&countAfter)
```

ผลลัพธ์จริง:

```
=== Transaction ที่ตั้งใจให้ fail เพื่อดู rollback ===
ผลลัพธ์ transaction: จำลอง error กลางทาง เพื่อทดสอบ rollback
จำนวน author ก่อน=4 หลัง=4 (เท่ากัน = rollback ทำงานถูกต้อง)
```

ยืนยันว่าแม้ `tx.Create(&Author{...})` จะรันสำเร็จไปแล้วในขั้นตอนแรก แต่พอ callback คืน error กลับมา GORM สั่ง `ROLLBACK` ให้อัตโนมัติ ทำให้จำนวน author ก่อนกับหลังเท่ากันเป๊ะ — นี่คือหัวใจของ **atomicity** ใน transaction: ทำสำเร็จทั้งหมด หรือไม่ทำอะไรเลย ไม่มีสถานะครึ่งๆ กลางๆ ค้างอยู่ หัวข้อ transaction แบบเจาะลึก (isolation level, deadlock, nested transaction) จะอยู่ใน **Part 078**

---

## 6. Hooks: `BeforeCreate`, `AfterUpdate` และการ Hash Password อัตโนมัติ

GORM เปิดให้ผูกฟังก์ชันเข้ากับ**จุดต่างๆ ในวงจรชีวิตของการบันทึกข้อมูล** เรียกว่า **hooks** — ทำได้แค่ implement method ตามชื่อที่ GORM กำหนดไว้บน struct นั้นๆ โดยไม่ต้อง config อะไรเพิ่มเติม (คล้ายกับการ implement interface แบบ implicit ที่เรียนใน **Part 013**)

Hooks ที่ใช้บ่อย: `BeforeCreate`, `AfterCreate`, `BeforeUpdate`, `AfterUpdate`, `BeforeSave`, `AfterSave`, `BeforeDelete`, `AfterDelete`

### ตัวอย่างที่ใช้บ่อยที่สุด: Hash Password ก่อนบันทึก

```go
type Account struct {
	gorm.Model
	Username     string `gorm:"uniqueIndex"`
	PasswordHash string
	rawPassword  string `gorm:"-"` // gorm:"-" บอกให้ข้าม field นี้ ไม่ persist ลง DB
}

func (a *Account) SetPassword(pw string) {
	a.rawPassword = pw
}

// BeforeCreate ถูกเรียกอัตโนมัติโดย GORM ก่อน INSERT ทุกครั้ง
func (a *Account) BeforeCreate(tx *gorm.DB) error {
	if a.rawPassword == "" {
		return fmt.Errorf("password is required")
	}
	hash, err := bcrypt.GenerateFromPassword([]byte(a.rawPassword), bcrypt.DefaultCost)
	if err != nil {
		return err
	}
	a.PasswordHash = string(hash)
	return nil
}
```

การใช้งาน:

```go
acc := Account{Username: "somchai"}
acc.SetPassword("supersecret123")
db.Create(&acc) // BeforeCreate ทำงานอัตโนมัติ ก่อนที่ INSERT จริงจะเกิดขึ้น
```

ผลลัพธ์จริง (bcrypt hash เป็นค่าที่ไม่ deterministic ทุกครั้งที่รันจะได้ค่าไม่เหมือนกัน แต่ยืนยันด้วยตัวมันเองผ่าน `CompareHashAndPassword`):

```
สร้าง account สำเร็จ PasswordHash (ตัด 20 ตัวแรก) = $2a$10$V6dD87d97MAiG...
bcrypt.CompareHashAndPassword ตรวจรหัสผ่านถูกต้อง: true
```

สังเกตว่า **ถ้า `BeforeCreate` return error ใดๆ GORM จะยกเลิกการ `INSERT` ทันที** ไม่มีการบันทึกลงฐานข้อมูลเลย — ตัวอย่างนี้ใช้ประโยชน์จากจุดนี้เพื่อบังคับว่าห้ามสร้าง account โดยไม่ตั้งรหัสผ่านก่อน

### `AfterUpdate`: ทำงานหลังอัปเดตสำเร็จ

```go
func (a *Account) AfterUpdate(tx *gorm.DB) error {
	fmt.Printf("[AfterUpdate hook] account %q ถูกอัปเดตแล้ว\n", a.Username)
	return nil
}
```

รันจริงหลังเรียก `db.Save(&acc)` ที่เปลี่ยน `Username`:

```
[AfterUpdate hook] account "somchai_renamed" ถูกอัปเดตแล้ว
```

Hooks มีประโยชน์มากสำหรับ logic ที่ "ต้องเกิดขึ้นทุกครั้ง ไม่ว่าจะเรียก Create/Save จากที่ไหนในโปรเจกต์" เช่น hash password, generate UUID, ส่ง event แจ้งระบบอื่น, validate ข้อมูลขั้นสุดท้ายก่อนบันทึก — ทำให้ logic เหล่านี้อยู่ที่จุดเดียว (ใน model) แทนที่จะต้องจำไปเรียกซ้ำทุกจุดที่มีการ `Create`/`Update` account

---

## 7. Scopes: Reusable Query Fragments

**Scope** คือฟังก์ชันที่รับ `*gorm.DB` แล้วคืน `*gorm.DB` กลับมา ใช้สำหรับห่อ query condition ที่ใช้ซ้ำบ่อยๆ ให้เป็นฟังก์ชันที่ตั้งชื่อได้ อ่านง่ายกว่าการเขียน `.Where(...)` ยาวๆ ซ้ำไปซ้ำมาในหลายจุดของโปรเจกต์:

```go
func ActivePosts(db *gorm.DB) *gorm.DB {
	return db.Where("title <> ''")
}

func TitleLike(kw string) func(db *gorm.DB) *gorm.DB {
	return func(db *gorm.DB) *gorm.DB {
		return db.Where("title LIKE ?", "%"+kw+"%")
	}
}
```

ใช้งานผ่าน `db.Scopes(...)` ใส่ scope ได้หลายตัวพร้อมกัน:

```go
var scopedPosts []Post
db.Scopes(ActivePosts, TitleLike("1-1")).Find(&scopedPosts)
```

ผลลัพธ์จริง:

```
=== Scopes: reusable query fragments ===
Scopes(ActivePosts, TitleLike("1-1")) เจอ 1 แถว: "บทความ 1-1"
```

สังเกตว่า `TitleLike` เป็นฟังก์ชันที่คืนฟังก์ชัน (ทบทวนแนวคิด closure และ higher-order function จาก **Part 009**) เพื่อรับ parameter (`kw`) เข้ามาได้ ในขณะที่ `ActivePosts` เป็น scope แบบไม่มี parameter ใช้ชื่อฟังก์ชันตรงๆ ได้เลย — pattern นี้ช่วยให้เขียน query ที่ซับซ้อนแต่ใช้ซ้ำบ่อยๆ (เช่น "เฉพาะแถวที่ยัง active", "เฉพาะของ tenant นี้ในระบบ multi-tenant") ให้เป็นชื่อความหมายชัดเจนแทนเงื่อนไข SQL ดิบๆ ที่ต้องคัดลอกซ้ำหลายที่

---

## 8. Migration ที่มากกว่า `AutoMigrate`: `golang-migrate` สำหรับ Production

ย้อนกลับไปที่ข้อจำกัดของ `AutoMigrate` ใน **Part 074 หัวข้อ 5** — สำหรับระบบ production ที่ schema เปลี่ยนแปลงอย่างมีการควบคุม (controlled) ต้องการ:

- **Migration history** — บันทึกว่า migration ไหนรันไปแล้วบ้าง เรียงลำดับชัดเจน
- **Rollback ได้** — ถ้า migration ใหม่มีปัญหา ต้องย้อนกลับ schema ไปเวอร์ชันก่อนหน้าได้
- **ควบคุมการเปลี่ยนแปลงที่ซับซ้อน** — เช่น เปลี่ยนชื่อ column ที่มีข้อมูลอยู่แล้วโดยไม่ทำข้อมูลหาย, แยก column เป็นสองตาราง, backfill ข้อมูลระหว่าง migration
- **รันเป็นขั้นตอนที่ตรวจสอบได้ก่อน deploy จริง** — ทีม DevOps ทบทวน SQL ที่จะรันได้ก่อนล่วงหน้า ต่างจาก `AutoMigrate` ที่ทำงานแบบ "เดา" จาก struct โดยอัตโนมัติ

เครื่องมือที่ community Go ใช้กันแพร่หลายที่สุดสำหรับงานนี้คือ **`golang-migrate/migrate`** (`github.com/golang-migrate/migrate`) — ทำงานโดยให้เขียนไฟล์ SQL คู่กันสองไฟล์ต่อหนึ่ง migration (`up` สำหรับเปลี่ยนแปลง, `down` สำหรับย้อนกลับ):

```
migrations/
├── 000001_create_books_table.up.sql
├── 000001_create_books_table.down.sql
├── 000002_add_isbn_column.up.sql
└── 000002_add_isbn_column.down.sql
```

```sql
-- 000002_add_isbn_column.up.sql
ALTER TABLE books ADD COLUMN isbn VARCHAR(20);

-- 000002_add_isbn_column.down.sql
ALTER TABLE books DROP COLUMN isbn;
```

รันผ่าน CLI หรือเรียกจากโค้ด Go โดยตรง:

```bash
migrate -database "postgres://user:pass@localhost/db?sslmode=disable" -path migrations up
migrate -database "postgres://user:pass@localhost/db?sslmode=disable" -path migrations down 1
```

> **หมายเหตุความซื่อสัตย์**: ส่วนนี้เป็น**ข้อมูลอ้างอิง**จาก documentation ของ `golang-migrate/migrate` ไม่ได้ติดตั้งและรันจริงในสภาพแวดล้อมของบทเรียนนี้ เพราะเป็นเครื่องมือ CLI แยกต่างหากที่ไม่ใช่ library ของ GORM โดยตรง (`golang-migrate` ทำงานเป็นอิสระจาก ORM ตัวไหนก็ได้ ใช้ได้แม้กับโปรเจกต์ที่ใช้ raw `database/sql` ล้วนๆ) แนวคิดและ syntax คำสั่งข้างต้นตรงกับ README อย่างเป็นทางการของโปรเจกต์

**แนวทางปฏิบัติที่แนะนำสำหรับทีมจริง**: ใช้ `AutoMigrate` ระหว่างพัฒนา (local development) เพื่อความรวดเร็ว แต่เมื่อ schema เริ่มนิ่งพอจะ deploy จริง ให้เขียน migration file ด้วยเครื่องมืออย่าง `golang-migrate` แล้วรัน migration เหล่านั้นเป็นขั้นตอนหนึ่งใน CI/CD pipeline (จะเรียนเจาะลึกเรื่อง pipeline ใน **ภาคที่ 9: DevOps**) — ทีมจำนวนมากใช้ทั้งสองอย่างคู่กัน: `AutoMigrate` ตอน dev, migration file ตอน deploy production

---

## 9. ตัวอย่างเต็ม: รวมทุก Feature ที่รันจริงกับ SQLite

โปรแกรมเดียวที่รวม Relations + Preload + Many-to-Many + Transaction + Hooks + Scopes ไว้ด้วยกันทั้งหมด (ตัดบางส่วนที่ซ้ำกับหัวข้อก่อนหน้าเพื่อความกระชับ โครงสร้างเต็มเหมือนกับที่ใช้รันจริงตลอดทั้งบทนี้):

```go
package main

import (
	"fmt"
	"log"

	"golang.org/x/crypto/bcrypt"
	"gorm.io/driver/sqlite"
	"gorm.io/gorm"
	"gorm.io/gorm/logger"
)

type Author struct {
	gorm.Model
	Name  string
	Posts []Post
}

type Post struct {
	gorm.Model
	Title    string
	AuthorID uint
	Tags     []Tag `gorm:"many2many:post_tags;"`
}

type Tag struct {
	gorm.Model
	Name string
}

type Account struct {
	gorm.Model
	Username     string `gorm:"uniqueIndex"`
	PasswordHash string
	rawPassword  string `gorm:"-"`
}

func (a *Account) SetPassword(pw string) { a.rawPassword = pw }

func (a *Account) BeforeCreate(tx *gorm.DB) error {
	hash, err := bcrypt.GenerateFromPassword([]byte(a.rawPassword), bcrypt.DefaultCost)
	if err != nil {
		return err
	}
	a.PasswordHash = string(hash)
	return nil
}

func main() {
	db, err := gorm.Open(sqlite.Open("gormadv.db"), &gorm.Config{
		Logger: logger.Default.LogMode(logger.Silent),
	})
	if err != nil {
		log.Fatal(err)
	}
	db.AutoMigrate(&Author{}, &Post{}, &Tag{}, &Account{})

	// สร้างข้อมูลตัวอย่าง
	for i := 1; i <= 3; i++ {
		author := Author{Name: fmt.Sprintf("นักเขียนคนที่ %d", i)}
		db.Create(&author)
		db.Create(&Post{Title: fmt.Sprintf("บทความ %d-1", i), AuthorID: author.ID})
		db.Create(&Post{Title: fmt.Sprintf("บทความ %d-2", i), AuthorID: author.ID})
	}

	// แก้ N+1 ด้วย Preload
	var authors []Author
	db.Preload("Posts").Find(&authors)
	for _, a := range authors {
		fmt.Printf("%s: %d posts\n", a.Name, len(a.Posts))
	}

	// Transaction พร้อม rollback อัตโนมัติเมื่อ error
	err = db.Transaction(func(tx *gorm.DB) error {
		newAuthor := Author{Name: "นักเขียนใหม่"}
		if err := tx.Create(&newAuthor).Error; err != nil {
			return err
		}
		return tx.Create(&Post{Title: "โพสต์แรก", AuthorID: newAuthor.ID}).Error
	})
	fmt.Println("transaction result:", err)

	// Hook: hash password อัตโนมัติ
	acc := Account{Username: "somchai"}
	acc.SetPassword("supersecret123")
	db.Create(&acc)
	fmt.Println("password ถูก hash แล้ว:", acc.PasswordHash != "supersecret123")
}
```

โปรแกรมนี้เป็นตัวอย่างสมบูรณ์ของสิ่งที่ทีมพัฒนาจริงส่วนใหญ่ใช้ GORM ทำในระบบ production: จัดการความสัมพันธ์ระหว่างตารางอย่างมีประสิทธิภาพ (ด้วย `Preload`), รับประกัน atomicity ด้วย transaction, และผูก business logic ที่ต้องเกิดขึ้นเสมอ (hash password) เข้ากับ model โดยตรงผ่าน hooks

---

## สรุปสิ่งที่ได้เรียนในบทนี้

- GORM รองรับความสัมพันธ์ 4 แบบ: belongs-to, has-one, has-many, many-to-many โดยเดา foreign key จาก naming convention อัตโนมัติ
- **N+1 query problem** เกิดเมื่อ loop ดึงข้อมูล relation ทีละแถวแยกกัน — ยืนยันด้วยการรันจริงว่าจาก 3 authors กลายเป็น 4 query โดยไม่ตั้งใจ
- **`Preload`** แก้ปัญหา N+1 ด้วย eager loading ลดจาก 4 query เหลือ 2 query ในตัวอย่างจริง และตัวเลขนี้จะไม่เพิ่มขึ้นตามจำนวนแถวของ parent อีกต่อไป
- **`db.Transaction`** จัดการ commit/rollback อัตโนมัติตามค่าที่ callback คืน — ต้องใช้ `tx` ที่ส่งเข้ามาเท่านั้น ห้ามใช้ `db` ตัวนอกภายใน callback
- **Hooks** (`BeforeCreate`, `AfterUpdate` ฯลฯ) ผูก business logic ที่ต้องเกิดขึ้นเสมอเข้ากับ model โดยตรง ตัวอย่างคลาสสิกคือ hash password ก่อนบันทึก
- **Scopes** ห่อ query condition ที่ใช้ซ้ำบ่อยเป็นฟังก์ชันที่ตั้งชื่อได้ อ่านง่ายกว่าเขียน `.Where(...)` ซ้ำหลายที่
- `AutoMigrate` เหมาะกับ development เท่านั้น สำหรับ production ควรใช้เครื่องมืออย่าง `golang-migrate` ที่รองรับ migration history และ rollback (ส่วนนี้เป็นข้อมูลอ้างอิงจาก documentation ไม่ได้รันจริงในบทเรียน)

## แบบฝึกหัดท้ายบท

1. สร้าง struct `Comment` ที่ `belongs to` ทั้ง `Post` และมี `AuthorName string` แล้วเขียนโปรแกรมที่ดึง `Post` พร้อม `Comments` ทั้งหมดด้วย `Preload` เดียว
2. ทดลอง Preload ความสัมพันธ์ซ้อนสองชั้น เช่น `Preload("Posts.Tags")` จาก `Author` แล้วนับจำนวน query ที่เกิดขึ้นจริงด้วยเทคนิคนับ query แบบในหัวข้อ 2 (ใช้ custom logger)
3. เขียนฟังก์ชันที่ใช้ `db.Transaction` โอนเงินระหว่างบัญชีสองบัญชี (หักจากบัญชีหนึ่ง บวกเข้าอีกบัญชีหนึ่ง) แล้วทดสอบว่าถ้าการบวกเงินบัญชีที่สอง fail (จำลอง error) เงินที่หักจากบัญชีแรกจะถูก rollback กลับมาจริงหรือไม่
4. เพิ่ม hook `BeforeDelete` ให้ `Account` ที่ป้องกันไม่ให้ลบ account ที่มี username เป็น `"admin"` โดย return error จาก hook เพื่อยกเลิกการลบ
5. เขียน scope `Paginate(page, pageSize int) func(*gorm.DB) *gorm.DB` ที่ใช้ `Offset`/`Limit` แล้วนำไปใช้ร่วมกับ scope อื่นผ่าน `db.Scopes(ActivePosts, Paginate(1, 10)).Find(&posts)`
6. ค้นคว้าเพิ่มเติมเกี่ยวกับ `golang-migrate/migrate`: ติดตั้ง CLI จริง แล้วลองสร้างไฟล์ migration คู่ `up`/`down` สำหรับตาราง `books` ที่ตรงกับ struct ใน Part 074 จากนั้นรันคำสั่ง `migrate ... up` กับฐานข้อมูล SQLite หรือ PostgreSQL ที่มีอยู่ เปรียบเทียบผลลัพธ์กับการใช้ `AutoMigrate`

---

**ต่อไป**: [Part 076 — MongoDB กับ Go](./076-mongodb.md)
