# Part 078: Database Transactions และ Connection Pooling

> ภาคที่ 6: ฐานข้อมูล — ตอนที่ 8 จาก 8 (Part 71–78)

นี่คือบทสุดท้ายของ **ภาคที่ 6: ฐานข้อมูล** — ตลอดภาคนี้เราเรียนมาตั้งแต่กลไกพื้นฐานของ `database/sql` (**Part 071**), เจาะลึก PostgreSQL และ MySQL (**Part 072–073**), ORM อย่าง GORM ทั้งเบื้องต้นและขั้นสูง (**Part 074–075**), ไปจนถึงฐานข้อมูลนอกสาย SQL อย่าง MongoDB และ Redis (**Part 076–077**) ทุก part ที่ผ่านมาแตะเรื่อง transaction และ connection pool เป็นระยะๆ แต่บทนี้จะเป็น**บทเจาะลึกเฉพาะสองเรื่องนี้เต็มรูปแบบ** เพราะทั้งคู่คือสิ่งที่แยกระหว่างโปรแกรมเมอร์ที่ "เขียนโค้ดคุยกับฐานข้อมูลได้" กับ "เขียนระบบที่พึ่งพาได้จริงใน production" — bug เรื่อง transaction ที่ทำ atomicity พัง หรือ connection pool ที่ตั้งค่าไม่ถูกต้อง มักไม่โผล่ในการทดสอบตอน dev เลยสักนิด แต่จะระเบิดตอนมี concurrent user จริงจำนวนมากเท่านั้น

> **หมายเหตุเรื่องความซื่อสัตย์ในการสาธิต**: ทุกตัวอย่างโค้ดและตัวเลขผลลัพธ์ในบทนี้ **ถูกรันจริง** กับ PostgreSQL 16.13 (ผ่าน `service postgresql start`, ฐานข้อมูล `godemo`) ในสภาพแวดล้อมที่ใช้เขียนหลักสูตรนี้ ด้วย Go 1.24.7 และ driver `github.com/lib/pq` v1.12.3 ตัวเลขเวลา, จำนวน connection, ค่า `db.Stats()` ที่แสดงทั้งหมดเป็น output จริงจากการรันโค้ด ไม่ใช่ค่าที่แต่งขึ้น รวมถึงพฤติกรรมภายในของ `database/sql` บางจุด (เช่น รอบเวลาของ background cleaner) ที่อ้างอิงจาก source code จริงของ Go standard library ในเครื่องที่ใช้เขียนบทนี้ (`/usr/local/go/src/database/sql/sql.go`)

## สารบัญของบทนี้

1. ACID คืออะไร: หลักการที่ transaction ทุกตัวต้องยึดถือ
2. `db.Begin()`, `tx.Commit()`, `tx.Rollback()`: Pattern มาตรฐานและ idiom `defer`
3. บั๊กคลาสสิก: ลืม Commit/Rollback แล้ว Connection รั่วออกจาก Pool (สาธิตจริง)
4. Isolation Levels: Read Committed vs Serializable และค่า Default ของ PostgreSQL
5. ตัวอย่างเต็ม: ฟังก์ชันโอนเงินระหว่างบัญชีแบบ Atomic พร้อมกรณี Rollback (สาธิตจริง)
6. Connection Pool เจาะลึก: `SetMaxOpenConns`, `SetMaxIdleConns`, `SetConnMaxLifetime`, `SetConnMaxIdleTime`
7. วินิจฉัยปัญหา Pool Exhaustion แบบที่ 1: Goroutine ต่อคิวรอ Connection เพราะ Pool เล็กเกินไป (สาธิตจริง)
8. วินิจฉัยปัญหา Pool Exhaustion แบบที่ 2: ลืมปิด `rows`/ลืม Commit ทำให้ Connection หายไปจาก Pool ถาวร (สาธิตจริง)
9. เชื่อมทุกอย่างเข้าด้วยกัน: `database/sql`, Driver เฉพาะทาง, และ GORM
10. ภาพรวมภาคที่ 6: ฐานข้อมูล — จากศูนย์ถึง Production
11. สรุปสิ่งที่ได้เรียนในบทนี้
12. แบบฝึกหัดท้ายบท

---

## 1. ACID คืออะไร: หลักการที่ transaction ทุกตัวต้องยึดถือ

**Transaction** คือการรวมชุดคำสั่ง SQL หลายคำสั่งให้ทำงาน "เป็นก้อนเดียว" — ไม่ว่าจะมีกี่คำสั่งอยู่ข้างในก็ตาม ผลลัพธ์สุดท้ายต้องเป็นแค่สองแบบเท่านั้น: **สำเร็จทั้งหมด** หรือ **ไม่มีอะไรเปลี่ยนแปลงเลย** ไม่มีสถานะครึ่งๆ กลางๆ ตกค้างอยู่ ฐานข้อมูลเชิงสัมพันธ์ที่ดีทุกตัว (PostgreSQL, MySQL, SQL Server, Oracle) รับประกันคุณสมบัตินี้ผ่านหลักการที่เรียกว่า **ACID** ซึ่งเป็นตัวย่อของ 4 คุณสมบัติ:

| ตัวอักษร | คุณสมบัติ | ความหมายแบบเข้าใจง่าย |
|---|---|---|
| **A** | **Atomicity** (ความเป็นอะตอม) | ทุกคำสั่งใน transaction เดียวกันทำงานเสมือนเป็นหน่วยเดียวที่แบ่งแยกไม่ได้ ("all or nothing") ถ้าคำสั่งใดคำสั่งหนึ่งล้มเหลว **ทุกคำสั่งก่อนหน้าใน transaction เดียวกันต้องถูกยกเลิกไปด้วย** |
| **C** | **Consistency** (ความสอดคล้อง) | transaction ที่จบสมบูรณ์ต้องพาฐานข้อมูลจากสถานะที่ถูกต้องหนึ่ง ไปสู่อีกสถานะที่ถูกต้องอีกอันหนึ่งเสมอ (เช่น constraint, foreign key, check constraint ต้องไม่ถูกละเมิด) |
| **I** | **Isolation** (ความเป็นอิสระ) | หลาย transaction ที่รันพร้อมกัน ต้องไม่เห็นสถานะ "ระหว่างทาง" ที่ยังไม่ commit ของกันและกัน (รายละเอียดว่า "ไม่เห็นแค่ไหน" คือเรื่อง isolation level ในหัวข้อ 4) |
| **D** | **Durability** (ความคงทน) | เมื่อ transaction commit สำเร็จแล้ว ข้อมูลต้องอยู่ถาวร แม้เครื่อง server จะไฟดับหรือ crash ทันทีหลัง commit ข้อมูลก็ต้องไม่หายไป |

ตัวอย่างที่เห็นภาพชัดที่สุดคือ **การโอนเงินระหว่างบัญชี** (ซึ่งจะเป็นตัวอย่างหลักของหัวข้อ 5): การโอนเงิน 200 บาทจากบัญชี A ไปบัญชี B ประกอบด้วย 2 คำสั่งขั้นต่ำ — หักเงินจาก A และเพิ่มเงินให้ B ถ้าหักเงินจาก A สำเร็จ แต่โปรแกรม crash ก่อนจะเพิ่มเงินให้ B เงิน 200 บาทนั้นจะหายไปจากระบบทันที (ไม่ได้อยู่ที่ A และไม่ได้อยู่ที่ B) นี่คือสิ่งที่ **Atomicity** มีไว้ป้องกันโดยตรง — ถ้าขั้นตอนใดขั้นตอนหนึ่งล้มเหลว ทั้งการหักและการเพิ่มต้องถูกยกเลิกกลับไปเหมือนไม่มีอะไรเกิดขึ้นเลย

ทบทวนจาก **Part 071** และ **Part 075**: ถ้าใช้ `db.Exec` เดี่ยวๆ สองครั้งแยกกันโดยไม่มี transaction ครอบ (หรือใน GORM คือใช้ `db.Create`/`db.Save` ตรงๆ สองครั้งแทนที่จะครอบด้วย `db.Transaction`) แต่ละคำสั่งจะ commit ทันทีเป็นเอกเทศ (auto-commit mode ซึ่งเป็นค่า default ของทั้ง `database/sql` และ GORM เมื่อไม่ได้เปิด transaction เอง) — ไม่มีการรับประกัน atomicity ข้ามคำสั่งใดๆ เลย นี่คือเหตุผลที่ต้องใช้ transaction อย่างชัดเจนทุกครั้งที่มีคำสั่งเขียนข้อมูลมากกว่าหนึ่งคำสั่งที่ **ต้องสำเร็จไปด้วยกันเสมอ**

---

## 2. `db.Begin()`, `tx.Commit()`, `tx.Rollback()`: Pattern มาตรฐานและ idiom `defer`

`database/sql` ให้ transaction ผ่าน type `*sql.Tx` ซึ่งได้มาจากการเรียก `db.Begin()` (หรือ `db.BeginTx(ctx, opts)` เวอร์ชันที่รับ context และ isolation level — จะเจาะลึกใน หัวข้อ 4):

```go
func (db *DB) Begin() (*Tx, error)
func (db *DB) BeginTx(ctx context.Context, opts *TxOptions) (*Tx, error)
```

จุดสำคัญที่ต้องเข้าใจก่อน: **`*sql.Tx` ยึด connection หนึ่งเส้นจาก pool ไว้ตลอดอายุของ transaction** ตั้งแต่ `Begin()` จนกว่าจะ `Commit()` หรือ `Rollback()` สำเร็จ — คำสั่งทุกคำสั่งที่เรียกผ่าน `tx.Exec`/`tx.Query`/`tx.QueryRow` (สังเกตว่าเรียกผ่านตัวแปร `tx` ไม่ใช่ `db`) จะถูกส่งไปที่ connection เส้นเดียวกันนี้เสมอ ทำให้ฐานข้อมูลมองว่าเป็นชุดคำสั่งเดียวกันในเซสชันเดียวกัน — นี่คือเหตุผลเชิงกลไกว่าทำไม atomicity ถึงทำงานได้: connection คือหน่วยที่ฐานข้อมูลใช้ผูก transaction เข้าด้วยกัน

`*sql.Tx` มี method หลักที่ต้องรู้จัก:

```go
func (tx *Tx) Commit() error   // ยืนยันทุกคำสั่งใน transaction ให้ถาวร แล้วคืน connection กลับ pool
func (tx *Tx) Rollback() error // ยกเลิกทุกคำสั่งใน transaction กลับไปเหมือนไม่เคยเกิดขึ้น แล้วคืน connection กลับ pool
```

### Pattern ที่ถูกต้อง: `defer tx.Rollback()` ทันทีหลัง `Begin()` สำเร็จ

Idiom มาตรฐานที่ต้องจำให้ขึ้นใจคือเขียน `defer tx.Rollback()` **ทันที** หลังเช็ค error ของ `Begin()`/`BeginTx()` แล้ว — ก่อนจะเขียนคำสั่งอื่นใดๆ ในฟังก์ชัน:

```go
func safeInsert(db *sql.DB, name string, balance float64) (err error) {
	tx, err := db.Begin()
	if err != nil {
		return fmt.Errorf("begin: %w", err)
	}
	// กฎเหล็ก: defer Rollback() ทันที ก่อนทำอะไรอื่น
	// ถ้า Commit() สำเร็จไปแล้ว Rollback() ที่ตามมาจะเป็น no-op
	// (คืน sql.ErrTxDone เฉยๆ ไม่มีผลข้างเคียงใดๆ กับข้อมูลที่ commit ไปแล้ว)
	defer tx.Rollback()

	if _, err = tx.Exec(`INSERT INTO accounts (name, balance) VALUES ($1, $2)`, name, balance); err != nil {
		return fmt.Errorf("insert: %w", err)
	}

	if err = tx.Commit(); err != nil {
		return fmt.Errorf("commit: %w", err)
	}
	return nil
}
```

ทำไมแบบนี้ถึงปลอดภัยแม้จะดู "ขัดกัน" ที่มีทั้ง `defer tx.Rollback()` และ `tx.Commit()` อยู่ในฟังก์ชันเดียวกัน? เพราะภายใน `*sql.Tx` มี flag บอกสถานะ "จบแล้วหรือยัง" (`done`) กำกับอยู่ — เมื่อ `Commit()` เรียกสำเร็จ transaction จะถูกทำเครื่องหมายว่าจบแล้วและ connection ถูกคืน pool ไปแล้ว พอ `defer tx.Rollback()` ทำงานตามมาทีหลัง (ซึ่งจะทำงานเสมอไม่ว่าฟังก์ชันจะ return แบบไหนก็ตาม) มันจะเจอว่า transaction จบไปแล้ว แล้วคืนค่า `sql.ErrTxDone` เฉยๆ **ไม่มีผลกระทบใดๆ กับข้อมูลที่ commit สำเร็จไปแล้ว** เราทดสอบและยืนยันโค้ดชุดนี้จริง:

```
insert ผ่าน transaction สำเร็จ, commit เรียบร้อย
insert ล้มเหลวตามคาด (balance ติดลบ ผิด CHECK constraint): pq: new row for relation "accounts" violates check constraint "accounts_balance_check" (23514)
จำนวนแถวทั้งหมดหลังทดสอบ (แถวที่ผิด constraint ต้องไม่ถูกนับ): 3
```

Output ที่สองมาจากการทดสอบว่าถ้าคำสั่งใน transaction ล้มเหลวกลางทาง (ในที่นี้คือ insert แถวที่ balance ติดลบ ซึ่งผิด `CHECK` constraint ของตาราง) แถวนั้นจะไม่ถูกนับรวมเลย เพราะ `defer tx.Rollback()` ทำงานให้อัตโนมัติตอนฟังก์ชันจบ (ไม่ว่าจะ return ตรงจุดเช็ค error หรือปล่อยให้ไหลผ่านไปจนจบฟังก์ชันก็ตาม)

> เทียบกับ `defer file.Close()` (**Part 024**) และ `defer rows.Close()` (**Part 071**) — หลักการเดียวกันทุกประการ: ทรัพยากรที่ยึดมา (connection ผ่าน transaction) ต้องมีจุดปล่อยคืนที่รับประกันว่าจะทำงานเสมอ ไม่ว่า flow ของฟังก์ชันจะออกทางไหน

### ทำไมต้อง `defer tx.Rollback()` แทนที่จะเขียน `Rollback()` แค่ตรงจุด error

มือใหม่บางคนอาจคิดว่าเขียนแบบนี้ก็พอ:

```go
// ไม่แนะนำ — เสี่ยงพลาดจุดใดจุดหนึ่ง
tx, _ := db.Begin()
if _, err := tx.Exec(query1); err != nil {
	tx.Rollback() // ต้องจำเขียนแบบนี้ทุกจุดที่ error
	return err
}
if _, err := tx.Exec(query2); err != nil {
	tx.Rollback() // และตรงนี้อีก
	return err
}
tx.Commit()
```

ปัญหาคือฟังก์ชันจริงมักมีจุด `return` error หลายจุด (validation, network error, panic ที่ recover ไว้ ฯลฯ) การพึ่งให้โปรแกรมเมอร์เขียน `tx.Rollback()` ซ้ำทุกจุดเสี่ยงพลาดตกหล่นจุดใดจุดหนึ่งได้ง่ายมาก — พอลืมแม้แค่จุดเดียว transaction จะค้างไม่ commit ไม่ rollback ซึ่งนำไปสู่บั๊กที่ร้ายแรงกว่ามากในหัวข้อถัดไป การเขียน `defer tx.Rollback()` ครั้งเดียวตรงต้นฟังก์ชันแก้ปัญหานี้ทั้งหมดในจุดเดียว ปลอดภัยจากทุก exit path

---

## 3. บั๊กคลาสสิก: ลืม Commit/Rollback แล้ว Connection รั่วออกจาก Pool

นี่คือบั๊กที่พบบ่อยที่สุดอันดับต้นๆ ของโปรแกรม Go ที่ใช้ `database/sql` กับ transaction — และเป็นบั๊กที่ **ไม่มีทาง compile error หรือ panic ทันที** ทำให้ตรวจจับยากมากถ้าไม่รู้ล่วงหน้าว่าต้องระวังจุดนี้

ทบทวนจากหัวข้อ 2 ว่า `*sql.Tx` ยึด connection ไว้จาก pool ตั้งแต่ `Begin()` จนกว่าจะ `Commit()`/`Rollback()` — คำถามคือ **ถ้าโปรแกรมเมอร์ลืมเรียกทั้งสองอย่างเลย จะเกิดอะไรขึ้น?**

```go
db.SetMaxOpenConns(1) // จำกัด pool ให้เหลือ connection เดียว เพื่อเห็นผลชัดที่สุด

// *** นี่คือบั๊กคลาสสิก ***
tx, err := db.Begin()
if err != nil {
	log.Fatal(err)
}
_, err = tx.Exec(`UPDATE accounts SET balance = balance + 1 WHERE name = $1`, "Somchai")
if err != nil {
	log.Fatal(err)
}
fmt.Println("รัน UPDATE ใน transaction สำเร็จ... แต่ไม่เรียก tx.Commit() หรือ tx.Rollback() เลย!")
// เจตนาไม่เรียก tx.Commit()/tx.Rollback() ตรงนี้ เพื่อสาธิตปัญหา
```

เราตั้ง `db.SetMaxOpenConns(1)` ไว้ล่วงหน้า แล้วตรวจสอบ `db.Stats()` ก่อนและหลัง:

```
--- ก่อนเกิดบั๊ก ---
  MaxOpenConnections=1 OpenConnections=0 InUse=0 Idle=0 WaitCount=0 WaitDuration=0s
รัน UPDATE ใน transaction สำเร็จ... แต่ไม่เรียก tx.Commit() หรือ tx.Rollback() เลย!
--- หลังเกิดบั๊ก (connection ยังไม่ถูกคืน pool) ---
  MaxOpenConnections=1 OpenConnections=1 InUse=1 Idle=0 WaitCount=0 WaitDuration=0s
```

สังเกตว่า `InUse=1` ค้างอยู่ — connection เส้นเดียวที่มีอยู่ใน pool ถูกยึดไปแล้วและไม่มีทางถูกคืนกลับมาเลยตราบใดที่ตัวแปร `tx` ยังไม่ถูกเรียก `Commit()`/`Rollback()` จากนั้นเราลอง query อื่นด้วย `context.WithTimeout` 2 วินาที เพื่อไม่ให้โปรแกรมค้างตลอดกาลตอนสาธิต:

```go
ctx, cancel := context.WithTimeout(context.Background(), 2*time.Second)
defer cancel()

var balance float64
err = db.QueryRowContext(ctx, `SELECT balance FROM accounts WHERE name = $1`, "Somchai").Scan(&balance)
```

ผลลัพธ์จริง:

```
Query ถัดไปล้มเหลว หลังรอ 2.001s: context deadline exceeded
สาเหตุ: pool มี connection สูงสุด 1 เส้น และเส้นเดียวที่มีถูก transaction ที่ไม่ปิดยึดไว้ตลอดกาล
query ใหม่จึงต้องรอ connection ว่างจนกว่า context timeout จะตัดจบ
```

Query ที่สองต้อง**รอครบ 2 วินาทีเต็มแล้วโดน timeout ตัดจบ** เพราะไม่มี connection ว่างเหลือให้ใช้เลย — นี่คือรูปย่อของสิ่งที่เกิดขึ้นจริงใน production: ทุกครั้งที่โค้ดพลาดลืม `Commit()`/`Rollback()` ของ transaction สักตัว จะมี connection หนึ่งเส้นหายไปจาก pool อย่างถาวร (ตราบใดที่ตัวแปร `tx` ยังไม่ถูก garbage collect และ process ยังไม่ถูกปิด) ถ้าโค้ด path ที่มีบั๊กนี้ถูกเรียกซ้ำๆ (เช่น endpoint ที่มีคน request เข้ามาเรื่อยๆ) connection จะถูกกินไปเรื่อยๆ ทีละเส้นจนกว่า pool จะหมด แล้วทุก request ในระบบ (ไม่ใช่แค่ endpoint ที่มีบั๊ก) จะเริ่มค้างรอ connection พร้อมกันหมด — อาการนี้เรียกกันทั่วไปว่า **"pool exhaustion"** และมักปรากฏใน production เป็น latency ที่ค่อยๆ แย่ลงเรื่อยๆ จนระบบหยุดตอบสนองไปทั้งระบบ ทั้งที่โค้ดจุดที่มีบั๊กอาจถูกเรียกน้อยมากก็ตาม

> **ทำไมไม่ panic หรือ error ทันที**: `database/sql` ออกแบบมาให้ `*sql.Tx` ที่ถูก `Begin()` ด้วย `context.Background()` (หรือ context ที่ไม่มีวัน cancel) จะไม่มีกลไกอัตโนมัติใดๆ มาปิด transaction ให้ ถ้าไม่เรียก `Commit()`/`Rollback()` เอง — และ standard library เองก็ไม่มี `finalizer` (กลไกที่ผูกกับ garbage collector ให้ทำความสะอาดทรัพยากรอัตโนมัติ) สำหรับ `*sql.Tx` เลย พูดง่ายๆ คือ **connection จะถูกยึดไว้ตลอดไปจนกว่า process จะจบการทำงาน** (ตอนนั้น connection ทั้งหมดจะถูกตัดที่ระดับ TCP และฝั่งฐานข้อมูลจะ rollback transaction ที่ค้างอยู่ให้เองโดยอัตโนมัติ — ซึ่งเป็นเหตุผลที่ตัวอย่างข้างต้นเมื่อตรวจสอบข้อมูลหลังโปรแกรมจบ พบว่า `balance` ของ Somchai ไม่ได้ถูกบวกเพิ่มเลย เพราะ transaction ที่ค้างถูก rollback โดยอัตโนมัติตอนโปรแกรมปิด connection ทิ้ง) แต่ใน**โปรแกรม server ที่รันต่อเนื่องไม่มีวันจบ** เช่น เว็บ API การรั่วแบบนี้จะสะสมไปเรื่อยๆ ไม่มีวันได้ rollback อัตโนมัติจนกว่าจะ restart โปรแกรม

**บทเรียนที่ต้องจำ**: `defer tx.Rollback()` ที่เขียนไว้ในหัวข้อ 2 ไม่ใช่แค่ "style ที่ดี" แต่เป็น**เกราะป้องกันบั๊กระดับ production-breaking** ตัวนี้โดยตรง — ทุกครั้งที่เขียนโค้ดเรียก `db.Begin()` ต้องเขียน `defer tx.Rollback()` เป็นบรรทัดถัดไปทันทีโดยไม่มีข้อยกเว้น

---

## 4. Isolation Levels: Read Committed vs Serializable และค่า Default ของ PostgreSQL

ทบทวนจากหัวข้อ 1 ว่า **Isolation** คือคุณสมบัติที่ควบคุมว่า transaction ที่รันพร้อมกันจะ "เห็น" การเปลี่ยนแปลงของกันและกันมากแค่ไหน มาตรฐาน SQL นิยาม isolation level ไว้ 4 ระดับ เรียงจากหลวมสุดไปแน่นสุด: **Read Uncommitted**, **Read Committed**, **Repeatable Read**, **Serializable** — บทนี้จะโฟกัสที่สองระดับที่ใช้บ่อยที่สุดในทางปฏิบัติ

### Read Committed: ค่า Default ของ PostgreSQL (และ Oracle, SQL Server)

**Read Committed** รับประกันว่า transaction จะไม่มีวันเห็นข้อมูลที่ transaction อื่น**ยังไม่ commit** (ป้องกัน "dirty read") แต่**ไม่รับประกัน**ว่าถ้า query ซ้ำสองครั้งในทรานแซกชันเดียวกัน จะได้ผลลัพธ์เหมือนกัน — เพราะระหว่างสอง query นั้น อาจมี transaction อื่น commit ข้อมูลใหม่เข้ามาแทรกได้ (เรียกว่า "non-repeatable read") นี่คือระดับที่**สมดุลระหว่าง performance กับความปลอดภัยของข้อมูล** จึงถูกเลือกเป็นค่า default ในฐานข้อมูลหลักส่วนใหญ่

### Serializable: เข้มงวดที่สุด

**Serializable** คือระดับที่เข้มงวดที่สุด รับประกันว่าผลลัพธ์ของการรัน transaction หลายตัวพร้อมกัน จะเหมือนกับการรันทีละตัวเรียงกันเสมอ (as if รันแบบ sequential) ไม่มี transaction ใดเห็นผลข้างเคียงจากกันและกันเลยแม้แต่น้อย แลกมาด้วย performance ที่ต่ำกว่า และมีโอกาสที่ transaction จะถูกฐานข้อมูล**ปฏิเสธและบังคับให้ retry** (serialization failure, error code `40001` ใน PostgreSQL) เมื่อตรวจพบว่าลำดับการรันจริงอาจทำให้ผลลัพธ์ต่างจากการรันแบบ sequential

### ตั้งค่า Isolation Level ผ่าน `sql.TxOptions`

`database/sql` ให้ระบุ isolation level ผ่าน `BeginTx` และ `sql.TxOptions`:

```go
type TxOptions struct {
	Isolation IsolationLevel
	ReadOnly  bool
}
```

ค่าคงที่ `sql.IsolationLevel` ที่มีให้ใช้ครบตามมาตรฐาน SQL: `sql.LevelDefault`, `sql.LevelReadUncommitted`, `sql.LevelReadCommitted`, `sql.LevelWriteCommitted`, `sql.LevelRepeatableRead`, `sql.LevelSnapshot`, `sql.LevelSerializable`, `sql.LevelLinearizable` — ฐานข้อมูลแต่ละตัวรองรับไม่เท่ากัน (PostgreSQL ไม่รองรับ Read Uncommitted จริงๆ จะปฏิบัติเหมือน Read Committed แทน) เราทดสอบจริงกับ PostgreSQL ด้วยการอ่านค่า isolation ปัจจุบันผ่านคำสั่ง `SHOW transaction_isolation`:

```go
func showIsolation(db *sql.DB, opts *sql.TxOptions, label string) {
	tx, err := db.BeginTx(context.Background(), opts)
	if err != nil {
		log.Fatal(err)
	}
	defer tx.Rollback()

	var level string
	tx.QueryRow(`SHOW transaction_isolation`).Scan(&level)
	fmt.Printf("%s: transaction_isolation = %s\n", label, level)
}

showIsolation(db, nil, "db.Begin() ปกติ (ไม่ระบุ isolation level)")
showIsolation(db, &sql.TxOptions{Isolation: sql.LevelReadCommitted}, "sql.LevelReadCommitted")
showIsolation(db, &sql.TxOptions{Isolation: sql.LevelSerializable}, "sql.LevelSerializable")
```

ผลลัพธ์จริง:

```
db.Begin() ปกติ (ไม่ระบุ isolation level): transaction_isolation = read committed
sql.LevelReadCommitted: transaction_isolation = read committed
sql.LevelSerializable: transaction_isolation = serializable
```

ยืนยันชัดเจนตามที่อธิบายไว้: **ถ้าไม่ระบุ `TxOptions` เลย (ส่ง `nil`) PostgreSQL จะใช้ค่า default คือ `read committed` เสมอ** และเมื่อระบุ `sql.LevelSerializable` ตรงๆ ฐานข้อมูลก็เปลี่ยนไปใช้ระดับที่เข้มงวดที่สุดตามที่สั่งจริง

### `ReadOnly: true` — บอกฐานข้อมูลว่า transaction นี้จะไม่เขียนอะไรเลย

```go
roTx, _ := db.BeginTx(context.Background(), &sql.TxOptions{ReadOnly: true})
defer roTx.Rollback()
_, err := roTx.Exec(`UPDATE accounts SET balance = balance + 1 WHERE name = 'Somchai'`)
```

ผลลัพธ์จริง:

```
พยายาม UPDATE ใน read-only transaction ล้มเหลวตามคาด: pq: cannot execute UPDATE in a read-only transaction (25006)
```

`ReadOnly: true` มีประโยชน์สองทาง: (1) เป็น**เอกสารในโค้ด** บอกเจตนาชัดเจนว่า transaction นี้ใช้แค่อ่านข้อมูล และ (2) เปิดให้ฐานข้อมูลบางตัว (รวมถึง PostgreSQL) **optimize การทำงานได้ดีขึ้น** เพราะรู้ล่วงหน้าว่าไม่ต้องเตรียมกลไกสำหรับ lock การเขียนเลย — เหมาะมากกับ transaction ที่ต้องอ่านข้อมูลหลายตารางแบบ consistent snapshot (เช่น สร้างรายงานที่ต้องอ่านหลาย query แต่อยากให้เห็นข้อมูล ณ เวลาเดียวกันทั้งหมด)

> **แนวทางปฏิบัติ**: ส่วนใหญ่ไม่จำเป็นต้องเปลี่ยน isolation level จาก default เลย — Read Committed เพียงพอสำหรับ use case ส่วนใหญ่ พิจารณาใช้ Serializable เฉพาะกรณีที่ธุรกิจต้องการความถูกต้องของข้อมูลระดับสูงสุดจริงๆ (เช่น ระบบการเงินที่ต้องป้องกัน race condition ระหว่างการเช็คยอดกับการหักยอดอย่างเข้มงวด) และต้องเตรียมโค้ดฝั่ง Go ให้ **retry transaction อัตโนมัติ**เมื่อเจอ serialization failure เพราะการถูกฐานข้อมูลปฏิเสธและให้ลองใหม่เป็นพฤติกรรมปกติของ Serializable ไม่ใช่ error ที่ควรแจ้งผู้ใช้ตรงๆ

---

## 5. ตัวอย่างเต็ม: ฟังก์ชันโอนเงินระหว่างบัญชีแบบ Atomic พร้อมกรณี Rollback

มาถึงตัวอย่างที่รวมทุกอย่างจากหัวข้อ 1–2 เข้าด้วยกัน: ฟังก์ชันโอนเงินระหว่างบัญชี — ตัวอย่างคลาสสิกที่สุดของความจำเป็นต้องใช้ transaction เพราะประกอบด้วยคำสั่งเขียนข้อมูล 2 คำสั่งที่ **ต้องสำเร็จไปด้วยกันเสมอ หรือไม่สำเร็จเลยทั้งคู่**

ตารางที่ใช้ทดสอบ:

```sql
CREATE TABLE accounts (
    id SERIAL PRIMARY KEY,
    name TEXT NOT NULL,
    balance NUMERIC(12,2) NOT NULL CHECK (balance >= 0)
);
```

สังเกต `CHECK (balance >= 0)` — เราตั้งใจใส่ constraint นี้ไว้ที่ระดับฐานข้อมูล เพื่อให้ฐานข้อมูลเองปฏิเสธการหักเงินที่ทำให้ยอดติดลบโดยอัตโนมัติ (**Consistency** ตัว C ใน ACID ทำงานผ่าน constraint แบบนี้โดยตรง) แทนที่จะพึ่งแค่ logic ฝั่ง Go เพียงอย่างเดียว — เป็นหลักการป้องกันสองชั้น (defense in depth) ที่ดี

```go
var ErrInsufficientFunds = errors.New("insufficient funds")

// Transfer โอนเงินจากบัญชี from ไปยังบัญชี to แบบ atomic
// ต้องสำเร็จทั้งคู่ (หักเงินและเติมเงิน) หรือไม่สำเร็จเลยทั้งคู่
func Transfer(ctx context.Context, db *sql.DB, from, to string, amount float64) (err error) {
	tx, err := db.BeginTx(ctx, nil)
	if err != nil {
		return fmt.Errorf("begin transaction: %w", err)
	}
	// defer tx.Rollback() ทันที: ถ้า Commit() สำเร็จไปแล้ว บรรทัดนี้จะเป็น no-op
	// ถ้าฟังก์ชัน return ก่อนถึง Commit() ไม่ว่าจุดไหน (error ใดๆ, panic, ctx ถูกยกเลิก)
	// Rollback() จะถูกเรียกเสมอ ป้องกัน transaction ค้างและ connection รั่วออกจาก pool
	defer tx.Rollback()

	// หักเงินจากบัญชีต้นทาง — CHECK (balance >= 0) ที่ตารางจะปฏิเสธถ้าเงินไม่พอ
	res, err := tx.ExecContext(ctx,
		`UPDATE accounts SET balance = balance - $1 WHERE name = $2`, amount, from)
	if err != nil {
		return fmt.Errorf("debit %s: %w", from, err)
	}
	if affected, _ := res.RowsAffected(); affected == 0 {
		return fmt.Errorf("debit %s: %w", from, sql.ErrNoRows)
	}

	// เติมเงินให้บัญชีปลายทาง
	res, err = tx.ExecContext(ctx,
		`UPDATE accounts SET balance = balance + $1 WHERE name = $2`, amount, to)
	if err != nil {
		return fmt.Errorf("credit %s: %w", to, err)
	}
	if affected, _ := res.RowsAffected(); affected == 0 {
		return fmt.Errorf("credit %s: %w", to, sql.ErrNoRows)
	}

	if err := tx.Commit(); err != nil {
		return fmt.Errorf("commit: %w", err)
	}
	return nil
}
```

จุดที่ควรสังเกต:

1. **ใช้ `tx.ExecContext` ไม่ใช่ `db.ExecContext`** — ต้องเรียกผ่านตัวแปร `tx` เสมอ เพื่อให้ทั้งสองคำสั่งอยู่ใน transaction/connection เดียวกัน (บั๊กที่พบบ่อยพอๆ กับการลืม `Rollback()` คือการเผลอเรียก `db.Exec` แทน `tx.Exec` กลางฟังก์ชัน ทำให้คำสั่งนั้นหลุดออกไป commit เดี่ยวๆ นอก transaction ทันทีโดยไม่มี error ใดเตือน — เจอปัญหาเดียวกันนี้ใน GORM ด้วยเมื่อเผลอใช้ `db` แทน `tx` ใน callback ของ `db.Transaction` ตามที่เตือนไว้ใน **Part 075 หัวข้อ 5**)
2. **เช็ค `RowsAffected()` เป็น 0** เพื่อจับกรณีชื่อบัญชีไม่มีอยู่จริง — ถ้าไม่เช็คจุดนี้ การ `UPDATE ... WHERE name = 'ไม่มีคนนี้'` จะไม่ error อะไรเลย (แค่ไม่กระทบแถวไหน) ทำให้เข้าใจผิดว่าโอนสำเร็จทั้งที่จริงไม่มีอะไรเกิดขึ้น
3. **`defer tx.Rollback()`** ครอบคลุมทุก error path ให้อัตโนมัติทั้งหมด ไม่ต้องเขียน `tx.Rollback()` ซ้ำในทุกจุดที่ return error

### ทดสอบจริง: กรณีสำเร็จ

```go
printBalances(db, "=== ยอดเงินก่อนโอน ===")
Transfer(ctx, db, "Somchai", "Suda", 200)
printBalances(db, "=== ยอดเงินหลังโอนสำเร็จ ===")
```

ผลลัพธ์จริง:

```
=== ยอดเงินก่อนโอน ===
  Somchai    1000.00
  Suda       500.00
  Anan       750.00

โอนเงิน Somchai -> Suda จำนวน 200 สำเร็จ
=== ยอดเงินหลังโอนสำเร็จ ===
  Somchai    800.00
  Suda       700.00
  Anan       750.00
```

ยอด Somchai ลด 200, ยอด Suda เพิ่ม 200 พอดี — เงินไม่หายไปไหนและไม่งอกขึ้นมาเอง ตรงตามที่ควรจะเป็น

### ทดสอบจริง: กรณีล้มเหลวกลางทาง ต้อง rollback ทั้งสองฝั่ง

จงใจโอนเงินจำนวนมากเกินยอดที่มี เพื่อให้คำสั่งหักเงิน (คำสั่งแรก) ชนกับ `CHECK (balance >= 0)` และล้มเหลว:

```go
err = Transfer(ctx, db, "Suda", "Somchai", 999999)
if err != nil {
	fmt.Println("โอนเงิน Suda -> Somchai จำนวน 999999 ล้มเหลวตามคาด:", err)
}
printBalances(db, "=== ยอดเงินหลังพยายามโอนเกินยอด (ต้องไม่เปลี่ยนแปลงเลย) ===")
```

ผลลัพธ์จริง:

```
โอนเงิน Suda -> Somchai จำนวน 999999 ล้มเหลวตามคาด: debit Suda: pq: new row for relation "accounts" violates check constraint "accounts_balance_check" (23514)
=== ยอดเงินหลังพยายามโอนเกินยอด (ต้องไม่เปลี่ยนแปลงเลย) ===
  Somchai    800.00
  Suda       700.00
  Anan       750.00
```

**ยอดเงินทั้งสองบัญชีไม่เปลี่ยนแปลงเลยแม้แต่สตางค์เดียว** — คำสั่งหักเงินจาก Suda ที่ฐานข้อมูลปฏิเสธไปแล้ว (เพราะ CHECK constraint) ถูก `defer tx.Rollback()` ยกเลิกโดยอัตโนมัติทั้ง transaction ทั้งที่ในทางทฤษฎีคำสั่งที่สอง (เติมเงินให้ Somchai) ไม่เคยถูกรันเลยด้วยซ้ำเพราะคำสั่งแรกล้มเหลวไปก่อน แต่ต่อให้คำสั่งแรกสำเร็จและคำสั่งที่สองล้มเหลวแทน ผลลัพธ์ก็จะเหมือนกันเป๊ะ: **rollback ทั้งคู่** — นี่คือ atomicity ที่ทำงานได้จริงตามที่โฆษณาไว้ใน ACID

---

## 6. Connection Pool เจาะลึก: `SetMaxOpenConns`, `SetMaxIdleConns`, `SetConnMaxLifetime`, `SetConnMaxIdleTime`

**Part 071 หัวข้อ 9** แนะนำภาพรวมของ pool ไปแล้วว่า `*sql.DB` คือ connection pool ไม่ใช่ connection เดี่ยว หัวข้อนี้จะเจาะลึกทั้ง 4 method ที่ควบคุมพฤติกรรมของ pool ทีละตัวว่าแต่ละตัวควบคุมอะไรจริงๆ และควรตั้งค่าเท่าไหร่

### `SetMaxOpenConns(n)`: จำนวน connection สูงสุดที่เปิดพร้อมกันได้

```go
db.SetMaxOpenConns(25)
```

นี่คือเพดานบนของ **connection ทั้งหมด** ที่ `*sql.DB` จะเปิดพร้อมกันได้ ไม่ว่าจะกำลังถูกใช้งานอยู่ (`InUse`) หรือพักเฉยๆ รอถูกใช้ซ้ำ (`Idle`) ก็ตาม เมื่อทุก connection ถูกใช้งานหมดพร้อมกัน (`InUse == MaxOpenConns`) request ถัดไปที่ต้องการ connection จะ**ต่อคิวรอ** จนกว่าจะมี connection ว่างคืนกลับมา — ค่า `0` (default เมื่อไม่ได้ตั้งเลย) หมายถึง **ไม่จำกัด** ซึ่งเป็นค่าที่อันตรายมากใน production ตามที่เตือนไว้ใน Part 071: ถ้า traffic พุ่งสูงพร้อมกันมาก แอปพลิเคชันอาจเปิด connection เข้าฐานข้อมูลนับพันเส้นพร้อมกัน จนฐานข้อมูลปฏิเสธ connection ใหม่ (PostgreSQL มีค่า `max_connections` จำกัดไว้ที่ระดับ server เอง ปกติ default อยู่ที่ 100) หรือแย่กว่านั้นคือฐานข้อมูลใช้หน่วยความจำจนพังทั้งเครื่อง

### `SetMaxIdleConns(n)`: จำนวน connection สูงสุดที่พักไว้เฉยๆ

```go
db.SetMaxIdleConns(25)
```

เมื่อ query จบและ connection ถูกคืนกลับ pool (`Commit`, `Rollback`, `rows.Close()`, หรือ query แบบ `Exec`/`QueryRow` ที่จบไปแล้ว) `database/sql` จะไม่ปิด connection นั้นทิ้งทันที แต่เก็บไว้เป็น "idle" รอถูกหยิบมาใช้ซ้ำในครั้งถัดไป (เพราะการเปิด connection ใหม่มี overhead สูง ทั้ง TCP handshake และ authentication) `SetMaxIdleConns` กำหนดว่าจะเก็บไว้ได้สูงสุดกี่เส้น — ถ้า idle connection เกินจำนวนนี้ตอนถูกคืน connection ส่วนเกินจะถูก**ปิดทิ้งจริงๆ ทันที** ไม่ใช่แค่ปล่อยว่างไว้

**ข้อควรระวังสำคัญ**: ถ้าตั้ง `MaxIdleConns` ต่ำกว่า `MaxOpenConns` มาก (เช่น `MaxOpenConns=100, MaxIdleConns=2` ซึ่งเป็นค่า default ของ Go ก่อนตั้งอะไรเลย `MaxIdleConns` default คือ 2) ในช่วง traffic สูงที่มีการเปิด connection จำนวนมากพร้อมกัน แล้ว traffic ลดลงกะทันหัน connection ส่วนใหญ่จะถูก**ปิดทิ้งเกือบจะในทันที** ทั้งที่เพิ่งเปิดไปไม่กี่วินาทีก่อนหน้า พอ traffic กลับมาสูงอีกครั้งก็ต้องเปิด connection ใหม่วนซ้ำ เกิดการเปิด/ปิด connection ถี่ๆ โดยไม่จำเป็น (thrashing) สิ้นเปลือง CPU/network ทั้งสองฝั่งโดยเปล่าประโยชน์ **แนวทางที่แนะนำ**: ตั้ง `MaxIdleConns` ให้เท่ากับ `MaxOpenConns` เสมอ เพื่อไม่ให้ connection ที่เพิ่งเปิดมาถูกปิดทิ้งเร็วเกินไปโดยไม่จำเป็น

### `SetConnMaxLifetime(d)`: อายุสูงสุดของ connection หนึ่งเส้น

```go
db.SetConnMaxLifetime(30 * time.Minute)
```

บังคับให้ connection ที่เปิดมานานเกิน `d` ถูกปิดและเปิดใหม่ แม้จะยังใช้งานได้ปกติดีอยู่ก็ตาม เหตุผลที่ต้องมีค่านี้ไม่ใช่เพราะ connection "เสื่อม" ตามเวลา แต่เพื่อรองรับสถานการณ์ระดับ infrastructure ที่พบบ่อยในระบบจริง:

- **Load balancer / DNS failover**: ถ้าฐานข้อมูลอยู่หลัง load balancer หรือมีการทำ failover ไปยัง replica ใหม่ connection เก่าที่เปิดค้างไว้นานจะยังคงชี้ไปที่ปลายทางเดิม ไม่รู้ว่ามีการเปลี่ยนแปลงเกิดขึ้น การบังคับ recycle เป็นระยะช่วยให้แอปพลิเคชัน "เห็น" การเปลี่ยนแปลง infrastructure ได้ในที่สุด
- **การ maintenance ฝั่งฐานข้อมูล**: DBA บางทีต้องการ rolling restart หรือเปลี่ยนค่า config ที่ต้องการให้ connection เก่าทยอยหมดอายุแทนที่จะตัดทุกเส้นพร้อมกัน
- **หลีกเลี่ยง resource leak สะสมระยะยาวบางประเภท** ทั้งฝั่ง driver และฝั่งฐานข้อมูลเอง ที่อาจสะสมไปตามอายุการเชื่อมต่อ

ค่า `0` (default) หมายถึง connection ไม่มีวันหมดอายุเพราะอายุเลยเด็ดขาด — สำหรับ production ควรตั้งเป็นค่าระดับนาทีถึงสิบนาที (ตัวอย่างข้างต้นตั้งไว้ 30 นาทีเป็นจุดเริ่มต้นที่สมเหตุสมผล)

### `SetConnMaxIdleTime(d)`: เวลาสูงสุดที่ connection จะพักเฉยๆ ก่อนถูกปิด

```go
db.SetConnMaxIdleTime(5 * time.Minute)
```

ต่างจาก `MaxLifetime` (นับอายุรวมตั้งแต่เปิด ไม่ว่าจะถูกใช้งานหรือพักอยู่ก็ตาม) — `ConnMaxIdleTime` นับเฉพาะ**ช่วงเวลาที่ connection ว่างต่อเนื่อง** ถ้า connection ถูกใช้งานสม่ำเสมอ มันจะไม่มีวันโดนปิดเพราะ idle time เลย แต่ถ้าแอปพลิเคชันมีช่วงที่ traffic ต่ำมากเป็นระยะ (กลางคืน, ช่วง off-peak) connection ที่เปิดค้างไว้เยอะเกินความจำเป็นจะถูกทยอยปิด ประหยัดทรัพยากรทั้งฝั่งแอปและฝั่งฐานข้อมูล ค่า `0` (default) หมายถึงไม่มีการปิดเพราะ idle time เลย

### เจาะกลไกเบื้องหลัง: `connectionCleaner` และรอบตรวจขั้นต่ำ 1 วินาที

เราทดสอบจริงว่า `ConnMaxIdleTime` ทำงานเมื่อไหร่ โดยตั้งค่าไว้สั้นมาก (200ms) เพื่อสาธิต แล้วเปิด connection พร้อมกัน 10 เส้นด้วย goroutine:

```go
db.SetMaxOpenConns(10)
db.SetMaxIdleConns(10)
db.SetConnMaxIdleTime(200 * time.Millisecond)

var wg sync.WaitGroup
for i := 0; i < 10; i++ {
	wg.Add(1)
	go func() {
		defer wg.Done()
		db.Exec(`SELECT 1`)
	}()
}
wg.Wait()
printStats(db) // ทันทีหลัง query เสร็จ

time.Sleep(100 * time.Millisecond)
printStats(db) // ยังไม่ถึง 200ms

time.Sleep(1100 * time.Millisecond)
printStats(db) // รวมเวลาผ่านไปเกิน 1 วินาทีแล้ว
```

ผลลัพธ์จริง:

```
ทันทีหลังยิง 10 query พร้อมกัน (ทุก connection กลับไป idle ใน pool):
  MaxOpenConnections=10 OpenConnections=10 InUse=0 Idle=10 MaxIdleClosed=0 MaxIdleTimeClosed=0 MaxLifetimeClosed=0

รอ 100ms (ยังไม่ถึง ConnMaxIdleTime=200ms):
  MaxOpenConnections=10 OpenConnections=10 InUse=0 Idle=10 MaxIdleClosed=0 MaxIdleTimeClosed=0 MaxLifetimeClosed=0

รอเพิ่มอีก 1.1s (เกิน 1 วินาที ซึ่งเป็นรอบตรวจขั้นต่ำของ connectionCleaner ภายใน database/sql):
  MaxOpenConnections=10 OpenConnections=0 InUse=0 Idle=0 MaxIdleClosed=0 MaxIdleTimeClosed=10 MaxLifetimeClosed=0
```

สังเกตจุดที่น่าสนใจมาก: หลังผ่านไปแค่ 100ms (เกิน `ConnMaxIdleTime=200ms` ไปแล้วครึ่งทาง) connection ทั้ง 10 เส้นยังไม่ถูกปิด แต่พอรอจนรวมเกิน 1 วินาที connection ทั้งหมดถูกปิดพร้อมกันทันที (`MaxIdleTimeClosed=10`) — เหตุผลคือ **`database/sql` มี background goroutine ชื่อ `connectionCleaner` ที่คอยตรวจสอบ idle connection เป็นระยะ แต่ตัวมันเองถูกกำหนดค่าคงที่ไว้ในซอร์สโค้ดว่ารอบตรวจจะไม่ถี่กว่า 1 วินาที (`minInterval = time.Second`) ไม่ว่าเราจะตั้งค่า `ConnMaxIdleTime` สั้นแค่ไหนก็ตาม** พูดง่ายๆ คือ **`ConnMaxIdleTime`/`ConnMaxLifetime` ที่ตั้งไว้ต่ำกว่า 1 วินาที จะไม่มีผลต่างจากตั้งไว้ที่ประมาณ 1 วินาทีเลยในทางปฏิบัติ** เพราะรอบตรวจของ cleaner เองก็ไม่ถี่ไปกว่านั้นอยู่ดี — รายละเอียดนี้ไม่ค่อยมีคนพูดถึงแต่สำคัญมากถ้ากำลัง debug ว่าทำไม pool "ไม่ยอมปิด" connection ตามเวลาที่คาดไว้เป๊ะๆ ในทางปฏิบัติค่าที่ตั้งจริงในระบบ production มักเป็นระดับนาทีอยู่แล้ว จึงไม่ค่อยชนกับข้อจำกัดนี้

### สรุปตารางค่าที่แนะนำสำหรับจุดเริ่มต้น

| Method | ค่า default (ไม่ตั้ง) | ค่าเริ่มต้นที่แนะนำสำหรับ production |
|---|---|---|
| `SetMaxOpenConns` | ไม่จำกัด (`0`) | 25 (ปรับตาม `max_connections` ของฐานข้อมูลหารด้วยจำนวน instance ของแอป) |
| `SetMaxIdleConns` | 2 | เท่ากับ `MaxOpenConns` |
| `SetConnMaxLifetime` | ไม่หมดอายุ (`0`) | 30 นาที |
| `SetConnMaxIdleTime` | ไม่หมดอายุ (`0`) | 5 นาที |

ตัวเลข 25 ไม่ใช่ค่าตายตัว — สูตรคร่าวๆ ที่ใช้กันทั่วไปคือเอา `max_connections` ของฐานข้อมูล (เผื่อพื้นที่ให้ connection สำหรับงาน admin/monitoring ไว้ส่วนหนึ่ง) หารด้วยจำนวน instance ของแอปพลิเคชันที่รันพร้อมกัน (เช่น จำนวน pod ใน Kubernetes) เช่น ฐานข้อมูลตั้ง `max_connections=200` และแอปรัน 8 instance พร้อมกัน ค่าที่สมเหตุสมผลต่อ instance คือประมาณ `(200 - 20 สำรอง) / 8 ≈ 22`

---

## 7. วินิจฉัยปัญหา Pool Exhaustion แบบที่ 1: Goroutine ต่อคิวรอ Connection เพราะ Pool เล็กเกินไป

อาการที่พบบ่อยที่สุดของปัญหา pool ในระบบจริงคือ: **request ทุกตัวเริ่มช้าลงพร้อมกันหมด** ทั้งที่แต่ละ query เดี่ยวๆ เร็วปกติดี — อาการนี้มักเกิดจาก `MaxOpenConns` ตั้งไว้ต่ำเกินไปเมื่อเทียบกับปริมาณงานพร้อมกันจริง เรามาดูให้เห็นภาพชัดด้วยการจำลอง query ที่ใช้เวลานาน (`SELECT pg_sleep(0.5)` — สั่งให้ PostgreSQL หน่วงเวลา 0.5 วินาทีต่อครั้ง จำลอง query ที่ซับซ้อนหรือ report หนักๆ) แล้วยิงพร้อมกันจาก 10 goroutine เทียบสองกรณี:

```go
func run(label string, maxOpen, numWorkers int) {
	db, _ := sql.Open("postgres", dsn)
	defer db.Close()
	db.SetMaxOpenConns(maxOpen)
	db.SetMaxIdleConns(maxOpen)

	var wg sync.WaitGroup
	start := time.Now()
	for i := 0; i < numWorkers; i++ {
		wg.Add(1)
		go func(id int) {
			defer wg.Done()
			db.Exec(`SELECT pg_sleep(0.5)`)
		}(i)
	}
	wg.Wait()
	elapsed := time.Since(start)

	stats := db.Stats()
	fmt.Printf("%s: %d goroutines, MaxOpenConns=%d -> เวลารวม %v (WaitCount=%d, WaitDuration=%v)\n",
		label, numWorkers, maxOpen, elapsed.Round(time.Millisecond), stats.WaitCount, stats.WaitDuration.Round(time.Millisecond))
}

run("pool แคบ", 2, 10)
run("pool กว้างพอ", 10, 10)
```

ผลลัพธ์จริง:

```
pool แคบ: 10 goroutines, MaxOpenConns=2 -> เวลารวม 2.515s (WaitCount=8, WaitDuration=10.093s)
pool กว้างพอ: 10 goroutines, MaxOpenConns=10 -> เวลารวม 522ms (WaitCount=0, WaitDuration=0s)
```

ตัวเลขเหล่านี้เล่าเรื่องได้ชัดมาก:

- **กรณี pool แคบ (`MaxOpenConns=2`)**: งาน 10 ชิ้นที่ใช้เวลา 0.5 วินาทีต่อชิ้น ถูกบีบให้ผ่าน connection แค่ 2 เส้น กลายเป็นต้องทำงานเป็น "รอบ" ประมาณ 5 รอบ (10 งาน ÷ 2 connection) แต่ละรอบใช้เวลา 0.5 วินาที รวมแล้วประมาณ 2.5 วินาที ตรงกับตัวเลขจริงที่วัดได้ (2.515s) เป๊ะ — และค่า `WaitCount=8` ยืนยันว่ามี 8 จาก 10 งานที่ต้อง**เข้าคิวรอ**ก่อนได้ connection (2 งานแรกได้ connection ทันที ที่เหลืออีก 8 งานต้องรอ) ส่วน `WaitDuration=10.093s` คือ**ผลรวมเวลารอของทุกงานที่ต้องรอ** (ไม่ใช่เวลาต่องาน) สะท้อนว่ามี "เวลารอสะสม" มหาศาลซ่อนอยู่เบื้องหลังตัวเลข throughput ที่ดูเหมือนไม่แย่มากนัก
- **กรณี pool กว้างพอ (`MaxOpenConns=10`)**: ทุกงานได้ connection ของตัวเองทันที ทำงานพร้อมกันจริง ใช้เวลารวมแค่ ~522ms (ใกล้เคียงกับเวลาของ query เดี่ยวๆ หนึ่งครั้ง) และ `WaitCount=0` ยืนยันว่าไม่มีใครต้องรอเลย

`db.Stats()` (ที่แนะนำให้ทำความรู้จักตั้งแต่ **Part 071**) จึงเป็นเครื่องมือวินิจฉัยที่ตรงประเด็นที่สุดสำหรับปัญหานี้ — โดยเฉพาะสองฟิลด์ `WaitCount` และ `WaitDuration` ที่บอกตรงๆ ว่า "มีกี่ครั้งที่ต้องรอ connection" และ "รอไปนานรวมเท่าไหร่" ในระบบ production ที่ดี ควรมีการ export ค่าจาก `db.Stats()` ออกไปเป็น metric (เช่นผ่าน Prometheus ซึ่งจะเรียนใน **Part 099**) และตั้ง alert เมื่อ `WaitCount`/`WaitDuration` เพิ่มขึ้นผิดปกติ — เป็นสัญญาณเตือนล่วงหน้าของปัญหา pool exhaustion ก่อนที่ผู้ใช้จริงจะเริ่มบ่นเรื่องความช้า

---

## 8. วินิจฉัยปัญหา Pool Exhaustion แบบที่ 2: ลืมปิด `rows`/ลืม Commit ทำให้ Connection หายไปจาก Pool ถาวร

หัวข้อ 3 สาธิตการลืม `Commit()`/`Rollback()` ของ transaction ไปแล้ว หัวข้อนี้จะสาธิตบั๊กฝาแฝดที่พบบ่อยไม่แพ้กัน: **ลืมปิด `*sql.Rows`** — ทบทวนกฎเหล็กข้อแรกจาก **Part 071 หัวข้อ 6** ว่าต้อง `defer rows.Close()` เสมอ คราวนี้เราจะพิสูจน์ให้เห็นจริงว่าถ้าไม่ทำตามกฎนี้จะเกิดอะไรขึ้น

```go
db.SetMaxOpenConns(2) // pool เล็กมากโดยตั้งใจ เพื่อให้เห็นผลชัดเจนเร็ว

for i := 1; i <= 2; i++ {
	rows, err := db.Query(`SELECT id, name, balance FROM accounts`)
	if err != nil {
		log.Fatal(err)
	}
	rows.Next() // อ่านแค่แถวแรกแล้วหยุด ไม่ loop จนจบ ไม่เรียก rows.Close()
	var id int
	var name string
	var balance float64
	rows.Scan(&id, &name, &balance)
	fmt.Printf("query ครั้งที่ %d: อ่านแถวแรกได้ id=%d name=%s (ไม่ปิด rows!)\n", i, id, name)
	// *** ไม่มี rows.Close() ตรงนี้ — connection ที่ query ครั้งนี้ยืมมาจะไม่ถูกคืน pool เลย ***
}
```

โค้ดชิ้นนี้จำลองบั๊กที่เกิดขึ้นจริงบ่อยมาก: โปรแกรมเมอร์เขียน `db.Query` เพื่อค้นหาแถวแรกที่ตรงเงื่อนไข พอเจอแล้วก็ `return`/`break` ออกจาก loop ทันทีโดยลืมเรียก `rows.Close()` (หรือแย่กว่านั้นคือลืมเขียน `defer rows.Close()` ไว้ตั้งแต่ต้นเลย) ผลลัพธ์จริงที่ได้:

```
--- เริ่มต้น ---
  MaxOpenConnections=2 OpenConnections=0 InUse=0 Idle=0
query ครั้งที่ 1: อ่านแถวแรกได้ id=3 name=Anan (ไม่ปิด rows!)
query ครั้งที่ 2: อ่านแถวแรกได้ id=3 name=Anan (ไม่ปิด rows!)
--- หลังยิง query 2 ครั้งโดยไม่ปิด rows (MaxOpenConns=2 ถูกใช้จนหมด) ---
  MaxOpenConnections=2 OpenConnections=2 InUse=2 Idle=0
```

แค่ 2 query ที่ไม่ปิด `rows` ก็กิน connection ไปจนครบ `MaxOpenConns=2` ทันที (`InUse=2`) — ลองยิง query ที่ 3 ด้วย timeout สั้นๆ:

```go
ctx, cancel := context.WithTimeout(context.Background(), 2*time.Second)
defer cancel()
err = db.QueryRowContext(ctx, `SELECT COUNT(*) FROM accounts`).Scan(&count)
```

```
query ที่ 3 ล้มเหลว หลังรอ 2s: context deadline exceeded
สาเหตุ: connection ทั้ง 2 เส้นถูก rows ที่ไม่ได้ปิดยึดไว้ถาวร ไม่มีเส้นว่างให้ query ใหม่ใช้
```

**ข้อแตกต่างสำคัญจากบั๊กในหัวข้อ 3**: การลืมปิด `rows` **ไม่มีทางถูกแก้ไขเองแม้จะรอไปนานแค่ไหนก็ตาม** และไม่มีกลไก timeout ใดๆ มาช่วยปลดปล่อย connection ให้อัตโนมัติเลย (ต่างจาก transaction ที่อย่างน้อยยังถูก rollback ให้ตอน process ปิดตัว) เพราะ `*sql.Rows` ไม่ได้ผูกกับ context ที่มีวันหมดอายุแบบเดียวกับ `*sql.Tx` — ตราบใดที่ตัวแปร `rows` ยังอยู่ในขอบเขตที่โปรแกรมยังทำงานอยู่ (ไม่ได้ถูก reassign หรือ garbage collect และ `database/sql` เองก็ไม่มี finalizer ให้ทั้งคู่ตามที่อธิบายไว้ในหัวข้อ 3) connection เส้นนั้นจะ**หายไปจาก pool อย่างถาวรตลอดอายุของ process** นี่คือเหตุผลที่กฎ `defer rows.Close()` ใน Part 071 ถูกเน้นว่า **"สำคัญมาก"** และ **"เป็นสาเหตุอันดับ 1 ของ connection pool exhausted ในโปรแกรม Go ที่ใช้ `database/sql`"** — ตอนนี้เราได้เห็นตัวเลขจริงยืนยันคำเตือนนั้นแล้ว

> **แนวทางป้องกันเชิงระบบ**: นอกจากวินัยส่วนตัวในการเขียน `defer rows.Close()` ทุกครั้งไม่มีข้อยกเว้น เครื่องมืออย่าง `go vet` และ linter เช่น `golangci-lint` (จะเรียนใน **Part 086**) มี rule ตรวจจับการเรียก `db.Query`/`rows` โดยไม่มี `Close()` ตามมาในบางกรณีได้ — การเปิด linter เหล่านี้ไว้ใน CI/CD pipeline (**ภาคที่ 9**) ช่วยจับบั๊กประเภทนี้ได้ตั้งแต่ก่อน merge code เข้า production

---

## 9. เชื่อมทุกอย่างเข้าด้วยกัน: `database/sql`, Driver เฉพาะทาง, และ GORM

ตอนนี้เราเรียนมาครบทั้งภาคที่ 6 แล้ว มาสรุปว่าเรื่อง transaction และ connection pool ที่เรียนในบทนี้ เชื่อมโยงกับทุก part ก่อนหน้าอย่างไร:

- **Part 071 (`database/sql` พื้นฐาน)**: บทนี้ต่อยอดโดยตรงจากหัวข้อ 9 ของ Part 071 ที่แนะนำ `SetMaxOpenConns`/`SetMaxIdleConns`/`SetConnMaxLifetime` แบบผิวเผิน — ตอนนี้เราเข้าใจกลไกเบื้องหลังลึกถึงระดับ `connectionCleaner` และรู้วิธีวินิจฉัยปัญหาด้วย `db.Stats()` แล้ว ส่วน `*sql.Tx` ที่เจาะลึกในบทนี้ ก็สร้างอยู่บนพื้นฐาน `*sql.DB` ตัวเดียวกันกับที่ใช้ `Query`/`Exec` มาตลอดทั้งภาค — ไม่มี API ใหม่ต้องเรียนเพิ่มนอกจาก `Begin`/`BeginTx`/`Commit`/`Rollback`
- **Part 072–073 (PostgreSQL, MySQL)**: ทุกอย่างที่เรียนในบทนี้ทำงานเหมือนกันทุกประการไม่ว่าจะใช้ driver ไหน เพราะ transaction และ pool เป็นกลไกระดับ `database/sql` ไม่ใช่ระดับ driver — สิ่งที่ต่างกันมีแค่รายละเอียดปลีกย่อยเฉพาะฐานข้อมูล เช่น isolation level ที่รองรับ, error code ที่ได้ตอน constraint ถูกละเมิด (`23514` ของ PostgreSQL เทียบกับ error code ของ MySQL ที่ต่างออกไปตามที่เรียนใน Part 073), และ placeholder syntax (`$1` เทียบกับ `?`) ถ้าใช้ `pgxpool` (**Part 072 หัวข้อ 5**) แทน `database/sql` ตรงๆ หลักการ pool เดียวกันนี้ยังใช้ได้ทั้งหมด เพียงแค่ชื่อ method อาจต่างไปบ้าง (เช่น `pgxpool.Config.MaxConns` แทน `SetMaxOpenConns`)
- **Part 074–075 (GORM)**: `db.Transaction(func(tx *gorm.DB) error {...})` ที่เรียนใน **Part 075 หัวข้อ 5** เป็นเพียง wrapper ที่สะดวกกว่ารอบๆ `Begin`/`Commit`/`Rollback` ตัวเดียวกันนี้เป๊ะ — กฎ "return error ใดๆ = rollback อัตโนมัติ, return nil = commit อัตโนมัติ" ของ GORM ก็คือการเรียก `tx.Rollback()`/`tx.Commit()` ให้อัตโนมัติตามค่าที่ callback คืนกลับมาเท่านั้นเอง ส่วนเรื่อง connection pool `*gorm.DB` ภายในก็ถือ `*sql.DB` ตัวเดียวกันไว้ (เรียกดูได้ผ่าน `sqlDB, _ := gormDB.DB()` แล้วเรียก `sqlDB.SetMaxOpenConns(...)` ได้ตรงๆ เหมือนกันทุกประการ) หมายความว่าทุกอย่างที่เรียนในหัวข้อ 6–8 ของบทนี้ **ใช้ได้กับโปรเจกต์ที่ใช้ GORM โดยตรง ไม่ต้องแปลความหมายใหม่**
- **Part 076–077 (MongoDB, Redis)**: ฐานข้อมูลทั้งสองตัวนี้ไม่ได้ใช้ `database/sql` (เพราะไม่ใช่ SQL relational database) จึงไม่มี `*sql.Tx`/`*sql.DB` แบบเดียวกัน — แต่แนวคิดเรื่อง **connection pool** ยังคงมีอยู่ในทั้งคู่ (MongoDB driver มี pool ในตัวควบคุมผ่าน `options.Client().SetMaxPoolSize(...)`, Redis driver `go-redis` ก็มี `PoolSize` ใน config เช่นกัน) หลักคิดเรื่อง "จำกัดจำนวน connection ให้เหมาะสมกับ traffic" และ "อย่าลืมปิด/คืน resource ที่ยืมมา" ที่เรียนในบทนี้เป็นหลักการสากลที่ใช้ได้กับฐานข้อมูลทุกประเภท แม้ว่ารายละเอียดการตั้งค่าจะต่างกันไปตาม driver ก็ตาม ส่วนเรื่อง transaction นั้น MongoDB มี multi-document transaction ให้ใช้ได้เช่นกัน (ต้องมี replica set) แต่ Redis โดยธรรมชาติเป็น single-threaded และมีคำสั่ง `MULTI`/`EXEC` ให้ความรู้สึกคล้าย transaction แต่ทำงานต่างจาก ACID transaction ของ SQL อย่างมีนัยสำคัญ (ไม่มี rollback กลางทางถ้าคำสั่งใดคำสั่งหนึ่งล้มเหลวด้วยเหตุผล runtime)

---

## 10. ภาพรวมภาคที่ 6: ฐานข้อมูล — จากศูนย์ถึง Production

ก่อนปิดท้ายบทนี้ (และปิดท้ายทั้งภาคที่ 6) มาย้อนดูเส้นทางทั้งหมดที่เราเดินผ่านมาตั้งแต่ Part 071:

```
Part 071  database/sql พื้นฐาน           → กลไกกลางที่ไม่ผูกกับฐานข้อมูลยี่ห้อใด
Part 072  PostgreSQL (pgx, lib/pq)       → driver เฉพาะทาง + feature เฉพาะของ Postgres
Part 073  MySQL                          → driver เฉพาะทาง + ข้อแตกต่างจาก Postgres
Part 074  GORM เบื้องต้น                  → ORM ลด boilerplate ของ database/sql
Part 075  GORM ขั้นสูง                    → Relations, N+1, Transaction wrapper, Hooks
Part 076  MongoDB                        → ฐานข้อมูลนอกสาย SQL แบบ document
Part 077  Redis                          → ฐานข้อมูลนอกสาย SQL แบบ key-value/cache
Part 078  Transactions & Pooling (บทนี้) → ความถูกต้องของข้อมูล (ACID) + ความเสถียรของระบบ (pool)
```

สังเกตว่าโครงสร้างของทั้งภาคนี้เดินตามลำดับที่มีเหตุผล: เริ่มจาก**กลไกกลาง** ที่ใช้ได้กับทุกฐานข้อมูล SQL (071) → **ลงลึกแต่ละฐานข้อมูล** เพื่อใช้ feature เฉพาะทาง (072–073) → **ยกระดับ productivity** ด้วย ORM (074–075) → **ขยายขอบเขต** ไปนอกสาย SQL (076–077) → และปิดท้ายด้วย**สองเรื่องที่ตัดขวางทุก layer ก่อนหน้าทั้งหมด** (078) เพราะไม่ว่าจะใช้ raw `database/sql`, driver เฉพาะทาง, หรือ GORM ก็ตาม สุดท้ายทุกอย่างวิ่งอยู่บน connection pool เดียวกัน และทุก write operation ที่มีมากกว่าหนึ่งคำสั่งก็ต้องการ transaction แบบเดียวกัน

สิ่งที่ควรพกติดตัวไปจากภาคนี้ทั้งภาค ไม่ใช่แค่วิธีเขียน syntax ของแต่ละฐานข้อมูล แต่คือ**นิสัยการเขียนโค้ดที่ปลอดภัย**สามข้อที่ย้ำซ้ำแล้วซ้ำเล่าตลอดทั้ง 8 part:

1. **ปิดทุกอย่างที่เปิดมาเสมอ** — `rows.Close()`, `stmt.Close()`, `tx.Rollback()` (ผ่าน `defer` ทันทีหลังเปิดสำเร็จ) ไม่มีข้อยกเว้น
2. **ห้ามต่อ string SQL จากค่าที่มาจากผู้ใช้เด็ดขาด** — ใช้ parameterized query เสมอ ไม่ว่าจะผ่าน `database/sql` ตรงๆ หรือผ่าน ORM
3. **ตั้งค่า connection pool อย่างมีสติ** ไม่ปล่อยให้เป็นค่า default ที่ไม่จำกัด และหมั่นตรวจสอบ `db.Stats()` เป็นระยะในระบบ production จริง

---

## สรุปสิ่งที่ได้เรียนในบทนี้

- **ACID** คือ 4 หลักการที่ transaction ต้องยึดถือ: Atomicity (all-or-nothing), Consistency (พาข้อมูลจากสถานะถูกต้องหนึ่งไปอีกสถานะถูกต้องหนึ่ง), Isolation (ไม่เห็นสถานะระหว่างทางของ transaction อื่น), Durability (commit แล้วต้องอยู่ถาวร)
- `db.Begin()`/`db.BeginTx()` คืน `*sql.Tx` ที่ยึด connection หนึ่งเส้นจาก pool ไว้ตลอดอายุ transaction — ต้องเรียกคำสั่งผ่าน `tx` เท่านั้น ไม่ใช่ `db` ตัวนอก
- Idiom มาตรฐาน: **`defer tx.Rollback()` ทันทีหลัง `Begin()` สำเร็จ** — ถ้า `Commit()` สำเร็จไปแล้ว `Rollback()` ที่ตามมาจะเป็น no-op ปลอดภัย 100%
- **บั๊กคลาสสิก**: ลืมเรียกทั้ง `Commit()`/`Rollback()` ทำให้ connection ถูกยึดไว้ถาวร (ยืนยันด้วยการรันจริงว่าโปรแกรมค้างจนโดน context timeout ตัดจบ) — ไม่มี finalizer ใดๆ มาช่วยปลดปล่อยให้อัตโนมัติ
- **Isolation level**: PostgreSQL ใช้ **Read Committed เป็นค่า default** เสมอถ้าไม่ระบุ, ตั้งค่าผ่าน `sql.TxOptions{Isolation: ...}` ได้ผ่าน `BeginTx` — **Serializable** เข้มงวดที่สุดแต่มีโอกาส transaction ถูกปฏิเสธและต้อง retry
- ตัวอย่างโอนเงินระหว่างบัญชี (`Transfer`) ยืนยันด้วยการรันจริงว่า transaction รับประกัน atomicity จริง: กรณีสำเร็จยอดเงินย้ายถูกต้องเป๊ะ, กรณีล้มเหลวกลางทาง (ชนกับ `CHECK` constraint) ยอดเงินทั้งสองฝั่งไม่เปลี่ยนแปลงเลยแม้แต่สตางค์เดียว
- **Connection pool 4 ค่า**: `SetMaxOpenConns` (เพดาน connection รวม), `SetMaxIdleConns` (ควรตั้งเท่ากับ MaxOpenConns), `SetConnMaxLifetime` (บังคับ recycle รองรับ infra เปลี่ยนแปลง), `SetConnMaxIdleTime` (ปิด connection ที่ไม่ได้ใช้นาน) — ทั้งหมดถูก `connectionCleaner` ภายในตรวจสอบ แต่รอบตรวจไม่ถี่กว่า 1 วินาทีเสมอ (ค่าคงที่ในซอร์สโค้ด Go เอง)
- **Pool exhaustion แบบที่ 1**: `MaxOpenConns` ต่ำเกินไปเทียบกับ traffic พร้อมกัน ทำให้ goroutine ต้องต่อคิวรอ — วินิจฉัยด้วย `db.Stats().WaitCount`/`WaitDuration` ที่รันจริงยืนยันตัวเลขชัดเจน (2.5 วินาทีเทียบ 0.5 วินาที)
- **Pool exhaustion แบบที่ 2**: ลืม `rows.Close()` ทำให้ connection หายไปจาก pool **ถาวรตลอดอายุ process** โดยไม่มีทาง timeout ใดๆ มาช่วยปลดปล่อยเลย ต่างจากกรณี transaction ที่อย่างน้อยยังถูก rollback ตอน process ปิด
- Transaction และ connection pool เป็นกลไกของ `database/sql` ที่ **ใช้ร่วมกันได้กับทุก driver** (PostgreSQL, MySQL) และ **GORM ก็สร้างอยู่บนกลไกเดียวกันนี้เป๊ะ** ผ่าน `db.Transaction` wrapper และ `sqlDB.SetMaxOpenConns` ที่เรียกผ่าน `gormDB.DB()` ได้ตรงๆ

## แบบฝึกหัดท้ายบท

1. เขียนฟังก์ชัน `TransferWithRetry` ที่ครอบ `Transfer` จากหัวข้อ 5 ไว้อีกชั้น โดยใช้ `sql.TxOptions{Isolation: sql.LevelSerializable}` แล้วถ้าเจอ error ที่มี PostgreSQL error code `40001` (serialization failure — ตรวจสอบผ่าน type assertion เป็น `*pq.Error` แล้วเช็ค field `Code`) ให้ retry ใหม่สูงสุด 3 ครั้งก่อนจะ return error จริง
2. ทดลองรันฟังก์ชัน `Transfer` จากหัวข้อ 5 พร้อมกันจากหลาย goroutine (โอนเงินไปมาระหว่างบัญชีเดียวกันสองบัญชีพร้อมกัน 20 goroutine) แล้วตรวจสอบยอดเงินรวมของทั้งสองบัญชีก่อนกับหลังว่าเท่ากันหรือไม่ (ควรเท่ากันเสมอถ้า transaction ทำงานถูกต้อง — ถ้าไม่เท่ากันแสดงว่ามีจุดบกพร่องในโค้ด)
3. เขียนโปรแกรมที่จงใจสร้างสถานการณ์ **deadlock**: เปิด 2 transaction พร้อมกันจากคนละ goroutine โดยตัวแรก UPDATE บัญชี A ก่อนแล้วค่อย UPDATE บัญชี B, ตัวที่สอง UPDATE บัญชี B ก่อนแล้วค่อย UPDATE บัญชี A (สลับลำดับกัน) แล้วสังเกต error ที่ PostgreSQL คืนกลับมา (คำใบ้: ค้นคำว่า "deadlock detected" ใน PostgreSQL documentation)
4. ปรับโปรแกรมในหัวข้อ 7 ให้ทดลองค่า `MaxOpenConns` หลายค่า (1, 2, 5, 10, 20) กับจำนวน goroutine คงที่ 20 ตัว แล้วพล็อตความสัมพันธ์ระหว่าง `MaxOpenConns` กับเวลารวมที่ใช้ อธิบายว่าทำไมเพิ่ม `MaxOpenConns` เกินจุดหนึ่งแล้วเวลาไม่ลดลงอีก (คำใบ้: เกี่ยวกับ `max_connections` ของฐานข้อมูลและจำนวน CPU core ของเครื่อง database server)
5. เขียนโปรแกรมที่จำลองบั๊กจากหัวข้อ 8 (ลืม `rows.Close()`) แต่คราวนี้ใส่ `db.SetConnMaxLifetime(3 * time.Second)` เพิ่มเข้าไป แล้วสังเกตว่า connection ที่ถูก `rows` ที่ไม่ได้ปิดยึดไว้ ยังคงถูกนับเป็น `InUse` ต่อไปแม้อายุจะเกิน `ConnMaxLifetime` แล้วหรือไม่ (คำใบ้: `ConnMaxLifetime` มีผลกับ connection ที่ idle เท่านั้น ไม่สามารถบังคับปิด connection ที่กำลัง `InUse` อยู่ได้)
6. ใช้ GORM (ทบทวนจาก Part 074–075) เขียนฟังก์ชันโอนเงินแบบเดียวกับหัวข้อ 5 ของบทนี้ แต่ใช้ `db.Transaction(func(tx *gorm.DB) error {...})` แทน `sql.Tx` ตรงๆ แล้วทดสอบกรณี fail เดียวกัน (โอนเงินเกินยอด) ยืนยันว่า rollback ทำงานถูกต้องเหมือนกันทั้งสองวิธี

---

ยินดีด้วยที่เรียนจบ **ภาคที่ 6: ฐานข้อมูล** ครบทั้ง 8 part แล้ว! ตอนนี้พร้อมแล้วสำหรับภาคถัดไปที่จะยกระดับคุณภาพโค้ดทั้งหมดที่เขียนมา ผ่านการทดสอบ เครื่องมือ และการวัด performance อย่างเป็นระบบ

**ต่อไป**: [Part 079 — Unit Testing ขั้นสูง](./079-unit-testing-advanced.md)
