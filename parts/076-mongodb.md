# Part 076: MongoDB กับ Go

> ภาคที่ 6: ฐานข้อมูล (Database) — ตอนที่ 6 จาก 8 (Part 71–78)

## สารบัญของบทนี้

1. MongoDB คืออะไร และต่างจากฐานข้อมูล relational อย่างไร
2. เมื่อไหร่ควรใช้ document database เมื่อไหร่ควรใช้ relational
3. ติดตั้งและเริ่มต้นใช้งาน official driver
4. เชื่อมต่อ MongoDB ด้วย `mongo.Connect`
5. ออกแบบ struct ด้วย `bson` tag คู่กับ `json` tag
6. Insert เอกสาร: `InsertOne` และ `InsertMany`
7. Query เอกสาร: `Find` และ `FindOne` ด้วย `bson.M`
8. Update เอกสาร: `UpdateOne`, `UpdateMany`
9. Delete เอกสาร: `DeleteOne`, `DeleteMany`
10. `context.Context` กับทุก operation ของ MongoDB driver
11. Index เพื่อ performance ของ query
12. หมายเหตุความซื่อสัตย์เรื่องการรันจริงในบทนี้
13. สรุปสิ่งที่ได้เรียนในบทนี้
14. แบบฝึกหัดท้ายบท

---

## 1. MongoDB คืออะไร และต่างจากฐานข้อมูล relational อย่างไร

ตลอด Part 071–075 เราใช้เวลาอยู่กับฐานข้อมูลแบบ **relational** (PostgreSQL, MySQL) ซึ่งข้อมูลถูกจัดเก็บเป็น **ตาราง (table)** ที่มี **schema ตายตัว** — ทุกแถวต้องมีคอลัมน์ตรงตามที่ประกาศไว้ และความสัมพันธ์ระหว่างตาราง (one-to-many, many-to-many) ใช้ **foreign key** เชื่อมกัน แล้วดึงข้อมูลข้ามตารางด้วย `JOIN`

**MongoDB** เป็นตัวแทนของฐานข้อมูลอีกตระกูลหนึ่งที่เรียกว่า **document database** (บางครั้งเรียกรวมว่า NoSQL) หลักการพื้นฐานต่างออกไปคนละแบบ:

| แนวคิด | Relational (PostgreSQL/MySQL) | MongoDB (Document) |
|---|---|---|
| หน่วยเก็บข้อมูล | แถว (row) ในตาราง (table) | เอกสาร (document) ในคอลเลกชัน (collection) |
| รูปแบบข้อมูล | Schema ตายตัว กำหนดคอลัมน์ล่วงหน้า | Schema ยืดหยุ่น เอกสารในคอลเลกชันเดียวกันมีโครงสร้างต่างกันได้ |
| รูปแบบไฟล์ดิบ | ตาราง 2 มิติ (แถว x คอลัมน์) | JSON-like (จริงๆ เก็บเป็น **BSON** — Binary JSON) |
| ความสัมพันธ์ | Foreign key + JOIN | มักฝัง (embed) ข้อมูลลูกไว้ในเอกสารเดียวกัน หรืออ้างอิงด้วย ID เอง (ไม่มี JOIN ในตัว) |
| Transaction ข้ามหลายแถว/เอกสาร | รองรับเต็มรูปแบบมาแต่ไหนแต่ไร (ACID) | รองรับ multi-document transaction ได้ตั้งแต่ MongoDB 4.0 แต่ไม่ใช่จุดแข็งดั้งเดิม |
| การขยายขนาด (scale) | แนวตั้ง (vertical) เป็นหลัก, sharding ทำได้แต่ซับซ้อนกว่า | ออกแบบมาให้ shard แนวนอน (horizontal) ได้ง่ายตั้งแต่ต้น |

ตัวอย่างที่เห็นภาพชัดที่สุด: สมมติเรามี "โพสต์บล็อก" หนึ่งโพสต์ที่มี comment หลายอัน

**แบบ relational** เราจะแยกเป็น 2 ตาราง `posts` และ `comments` โดย `comments.post_id` เป็น foreign key ไปยัง `posts.id` แล้วเวลาจะอ่านโพสต์พร้อม comment ต้อง `JOIN` สองตารางเข้าด้วยกัน

**แบบ document (MongoDB)** เราสามารถเก็บ comment เป็น array ฝังอยู่ **ในเอกสารเดียวกัน** กับโพสต์ได้เลย:

```json
{
  "_id": "665f1a2b3c4d5e6f7a8b9c0d",
  "title": "เรียน Go ให้เก่งใน 100 วัน",
  "author": "somchai",
  "comments": [
    { "user": "malee", "text": "บทความดีมากครับ", "created_at": "2025-01-10T10:00:00Z" },
    { "user": "wichai", "text": "รอตอนต่อไปครับ", "created_at": "2025-01-11T08:30:00Z" }
  ],
  "tags": ["go", "programming", "tutorial"]
}
```

อ่านครั้งเดียวได้ข้อมูลครบ ไม่ต้อง JOIN เลย — นี่คือจุดแข็งสำคัญของ document database

## 2. เมื่อไหร่ควรใช้ document database เมื่อไหร่ควรใช้ relational

นี่คือคำถามที่สำคัญกว่า "MongoDB ดีกว่า PostgreSQL ไหม" มาก เพราะทั้งสองแบบถูกออกแบบมาให้ **เหมาะกับปัญหาคนละแบบ** ไม่มีใครดีกว่าใครแบบเบ็ดเสร็จ

### MongoDB (document) เหมาะเมื่อ

- **Schema ยืดหยุ่นหรือเปลี่ยนบ่อย**: ข้อมูลแต่ละ record มีโครงสร้างไม่เหมือนกัน เช่น attribute ของสินค้าที่ต่างประเภทกันโดยสิ้นเชิง (เสื้อผ้ามีไซส์/สี, หนังสือมี ISBN/จำนวนหน้า) การยัดทุกอย่างลงตาราง relational เดียวจะมีคอลัมน์ NULL เต็มไปหมด
- **ข้อมูลเป็นเอกสารที่อ่าน/เขียนเป็นก้อนเดียวเกือบตลอด**: เช่น โปรไฟล์ผู้ใช้ที่มี preference ซ้อนกันหลายชั้น, log event, catalog สินค้า e-commerce
- **ต้องการ iterate เร็วในช่วงพัฒนา**: ไม่ต้องเขียน migration ทุกครั้งที่เพิ่ม field ใหม่ (แต่ข้อดีนี้ก็มีข้อเสียแฝงคือ "schema-less" จริงๆ แล้วหมายถึง "schema อยู่ในโค้ดแอปพลิเคชันแทน" — ความรับผิดชอบไม่ได้หายไปไหน แค่ย้ายที่)
- **ต้องการ scale แนวนอนในระดับข้อมูลมหาศาล** (sharding เป็นล้าน document ข้ามหลายเครื่อง)

### Relational (PostgreSQL/MySQL) เหมาะเมื่อ

- **ข้อมูลมีความสัมพันธ์ซับซ้อนและต้อง query ข้าม entity บ่อย** เช่น ระบบบัญชี ระบบธนาคาร ระบบสั่งซื้อที่ต้อง JOIN ตาราง orders, order_items, products, customers, payments เข้าด้วยกันตลอดเวลา
- **ต้องการ strong consistency และ referential integrity** เช่น ห้ามลบ customer ที่ยังมี order ค้างอยู่ — foreign key constraint บังคับกฎนี้ที่ระดับฐานข้อมูลได้เลย ไม่ต้องพึ่งโค้ดแอปพลิเคชัน
- **ต้องการ transaction ที่ซับซ้อนครอบคลุมหลายตาราง** เช่น การโอนเงิน (เราจะเห็นตัวอย่างเต็มรูปแบบใน Part 078) — relational database ออกแบบมาเพื่อสิ่งนี้โดยเฉพาะมาตั้งแต่ต้น
- **ข้อมูลมีโครงสร้างชัดเจนแน่นอนและไม่ค่อยเปลี่ยน** เช่น ข้อมูลพนักงาน, สินค้าคงคลังแบบมาตรฐาน

### ในทางปฏิบัติ

ระบบขนาดใหญ่จำนวนมากใช้ **ทั้งสองแบบร่วมกัน** — เช่น ใช้ PostgreSQL เก็บข้อมูลออเดอร์/บัญชีที่ต้องการ ACID เต็มรูปแบบ และใช้ MongoDB เก็บ log กิจกรรมผู้ใช้หรือ catalog สินค้าที่ schema เปลี่ยนบ่อย นี่คือแนวคิด **polyglot persistence** — เลือกฐานข้อมูลให้เหมาะกับแต่ละส่วนของระบบ ไม่ใช่ยัดทุกอย่างลงฐานข้อมูลเดียว

---

## 3. ติดตั้งและเริ่มต้นใช้งาน official driver

MongoDB มี official driver สำหรับ Go ชื่อ `go.mongodb.org/mongo-driver` ติดตั้งด้วย:

```bash
go get go.mongodb.org/mongo-driver/mongo
```

> **หมายเหตุเวอร์ชัน**: ขณะเขียนบทนี้ `go.mongodb.org/mongo-driver` (v1) เป็นเวอร์ชันที่ใช้กันแพร่หลายที่สุดและเสถียรมานาน แต่ทีม MongoDB ได้ออก **`go.mongodb.org/mongo-driver/v2`** ซึ่งเป็นเวอร์ชันถัดไปที่ปรับ API บางส่วน (เช่นเปลี่ยนวิธี handle `context` ในบาง constructor) โปรเจกต์ใหม่ควรเช็ค [เอกสารทางการ](https://www.mongodb.com/docs/drivers/go/current/) ว่าควรใช้เวอร์ชันไหน — บทนี้สอนด้วย v1 เพราะยังเป็น API ที่พบเจอมากที่สุดในโค้ด production ปัจจุบัน แต่แนวคิดหลัก (Connect, InsertOne, Find, context ทุก call) เหมือนกันทั้งสองเวอร์ชัน

ถ้าต้องการรันตัวอย่างในบทนี้จริง ต้องมี MongoDB server ให้เชื่อมต่อก่อน วิธีที่สะดวกที่สุดในเครื่อง dev คือใช้ Docker:

```bash
docker run -d --name mongo-dev -p 27017:27017 mongo:7
```

หรือติดตั้ง MongoDB Community Server ตรงจาก https://www.mongodb.com/try/download/community

---

## 4. เชื่อมต่อ MongoDB ด้วย `mongo.Connect`

```go
package main

import (
	"context"
	"log"
	"time"

	"go.mongodb.org/mongo-driver/mongo"
	"go.mongodb.org/mongo-driver/mongo/options"
)

func main() {
	// mongo.Connect ต้องการ context เสมอ — ถ้าเชื่อมต่อไม่ทันภายในเวลานี้ จะ error
	ctx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
	defer cancel()

	client, err := mongo.Connect(ctx, options.Client().ApplyURI("mongodb://localhost:27017"))
	if err != nil {
		log.Fatal(err)
	}
	// เชื่อมต่อจริงในเบื้องหลังเป็นแบบ lazy — ต้อง Ping เพื่อยืนยันว่าต่อได้จริง
	defer func() {
		if err := client.Disconnect(context.Background()); err != nil {
			log.Println("disconnect error:", err)
		}
	}()

	if err := client.Ping(ctx, nil); err != nil {
		log.Fatal("ping failed:", err)
	}
	log.Println("connected to MongoDB!")
}
```

จุดสำคัญที่ต้องสังเกต:

1. **`mongo.Connect` ไม่ได้ต่อ connection ทันที** — มันแค่สร้าง client และเริ่ม "topology monitoring" ในเบื้องหลัง การเชื่อมต่อจริงเกิดขึ้นแบบ lazy เมื่อมีการเรียกใช้ operation ครั้งแรก ดังนั้นถ้าอยากรู้ทันทีว่าต่อฐานข้อมูลได้จริงหรือไม่ ต้องเรียก `client.Ping(ctx, nil)` เสมอ
2. **`defer client.Disconnect(...)`** ควรเรียกเมื่อโปรแกรมเลิกใช้ client แล้ว (เช่นตอนปิดแอป) ไม่ใช่ปิดทุกครั้งหลังทำ operation เดียว — `mongo.Client` ถูกออกแบบมาให้สร้างครั้งเดียวแล้วใช้ตลอดอายุโปรแกรม เหมือนกับ `*sql.DB` ใน Part 071 ที่เป็น connection pool ไม่ใช่ connection เดี่ยว
3. Connection string รูปแบบ `mongodb://host:port` เป็นรูปแบบพื้นฐาน ถ้าใช้ MongoDB Atlas (cloud) จะเป็น `mongodb+srv://user:pass@cluster.mongodb.net/`

### รูปแบบที่แนะนำ: เก็บ client ไว้ใน struct แล้วส่งต่อ

เหมือนที่เราทำกับ `*sql.DB` หรือ GORM `*gorm.DB` ใน Part 071–075 เราควรสร้าง `mongo.Client` ครั้งเดียวตอน startup แล้วส่งต่อผ่าน dependency injection แทนที่จะสร้างใหม่ทุกครั้ง:

```go
type ProductRepository struct {
	coll *mongo.Collection
}

func NewProductRepository(client *mongo.Client) *ProductRepository {
	return &ProductRepository{
		coll: client.Database("shop").Collection("products"),
	}
}
```

`client.Database("shop")` ได้ reference ไปยัง database ชื่อ `shop` (ไม่ต้องสร้างล่วงหน้า — MongoDB สร้าง database และ collection ให้อัตโนมัติเมื่อมีการ insert เอกสารแรก) และ `.Collection("products")` ได้ reference ไปยัง collection ชื่อ `products` ภายในนั้น

---

## 5. ออกแบบ struct ด้วย `bson` tag คู่กับ `json` tag

ใน Part 025 เราเรียน `encoding/json` และการใช้ struct tag แบบ `` `json:"name"` `` เพื่อควบคุมว่า field ไหน map กับ key อะไรตอน marshal/unmarshal MongoDB driver ใช้หลักการเดียวกันทุกประการ แต่เปลี่ยนจาก `json` เป็น **`bson`** (Binary JSON — รูปแบบไบนารีที่ MongoDB ใช้เก็บข้อมูลจริงภายใน)

```go
package model

import (
	"time"

	"go.mongodb.org/mongo-driver/bson/primitive"
)

type Product struct {
	// primitive.ObjectID คือชนิดของ MongoDB _id โดย default
	// omitempty ทำให้ตอน insert เอกสารใหม่ ถ้าไม่ได้กำหนด ID มา MongoDB จะ generate ให้เอง
	ID        primitive.ObjectID `bson:"_id,omitempty" json:"id"`
	Name      string             `bson:"name" json:"name"`
	Price     float64            `bson:"price" json:"price"`
	Tags      []string           `bson:"tags,omitempty" json:"tags,omitempty"`
	CreatedAt time.Time          `bson:"created_at" json:"created_at"`
}
```

ข้อสังเกตสำคัญ:

- **`_id` คือ field พิเศษ**: MongoDB ทุกเอกสารต้องมี field ชื่อ `_id` เป็น primary key (คล้าย `id` ในตาราง relational) ถ้าไม่ระบุตอน insert MongoDB จะสร้าง `primitive.ObjectID` ให้อัตโนมัติ — เป็นเลข 12 byte ที่เข้ารหัส timestamp + machine + process + counter ทำให้ unique แทบจะรับประกันได้โดยไม่ต้องขอ server generate แบบ sequence
- **ทำไมต้องมีทั้ง `bson` และ `json` tag**: field เดียวกันมักถูกใช้สองที่พร้อมกันในเว็บแอป — เก็บลง MongoDB (`bson` tag) และส่งออกเป็น JSON response ให้ client (`json` tag) การมีสอง tag คู่กันทำให้ struct เดียวทำหน้าที่ได้ทั้งสองแบบ โดยไม่ต้องเขียน struct ซ้ำสอง copy
- **`omitempty`** ทำงานเหมือนใน `encoding/json` ทุกประการ — ถ้า field เป็น zero value จะไม่ถูกใส่ในเอกสารที่ insert เลย
- ชนิดข้อมูลที่ `bson` รองรับ map ตรงกับ Go type ส่วนใหญ่ได้เลย: `string`, `int`, `int64`, `float64`, `bool`, `time.Time`, `[]T`, `map[string]T`, และ nested struct (จะกลายเป็น embedded document อัตโนมัติ)

### ตัวอย่างเอกสารซ้อนกัน (embedded document)

```go
type Address struct {
	City    string `bson:"city" json:"city"`
	ZipCode string `bson:"zip_code" json:"zip_code"`
}

type Customer struct {
	ID      primitive.ObjectID `bson:"_id,omitempty" json:"id"`
	Name    string             `bson:"name" json:"name"`
	Address Address            `bson:"address" json:"address"`
}
```

ตอน insert เอกสาร `Customer` ค่า `Address` จะถูกเก็บเป็น **nested document** ในเอกสาร MongoDB โดยอัตโนมัติ — นี่คือความแตกต่างสำคัญจากตาราง relational ที่ต้องแยก `addresses` เป็นอีกตารางแล้ว JOIN

---

## 6. Insert เอกสาร: `InsertOne` และ `InsertMany`

```go
func CreateProduct(ctx context.Context, coll *mongo.Collection, p Product) (primitive.ObjectID, error) {
	// ทุก operation ของ driver ต้องมี context.Context เสมอ (บังคับโดย signature)
	insertCtx, cancel := context.WithTimeout(ctx, 5*time.Second)
	defer cancel()

	p.CreatedAt = time.Now()

	res, err := coll.InsertOne(insertCtx, p)
	if err != nil {
		return primitive.NilObjectID, fmt.Errorf("insert product: %w", err)
	}

	// InsertedID เป็น interface{} ต้อง type assert เป็น primitive.ObjectID
	id, ok := res.InsertedID.(primitive.ObjectID)
	if !ok {
		return primitive.NilObjectID, fmt.Errorf("unexpected InsertedID type: %T", res.InsertedID)
	}
	return id, nil
}
```

`InsertMany` ใช้เมื่อต้องการ insert หลายเอกสารพร้อมกันในคำสั่งเดียว (เร็วกว่าเรียก `InsertOne` วนลูปมาก เพราะลด round-trip เครือข่าย):

```go
func CreateProducts(ctx context.Context, coll *mongo.Collection, products []interface{}) error {
	insertCtx, cancel := context.WithTimeout(ctx, 10*time.Second)
	defer cancel()

	_, err := coll.InsertMany(insertCtx, products)
	return err
}
```

> สังเกตว่า `InsertMany` รับ `[]interface{}` ไม่ใช่ `[]Product` โดยตรง เพราะ driver ออกแบบให้ยืดหยุ่น รับเอกสารต่างชนิดกันในคอลเลกชันเดียวได้ (ตามธรรมชาติ schema-less ของ MongoDB) เวลาเรียกใช้จริงต้องแปลง slice ก่อน:
>
> ```go
> docs := make([]interface{}, len(products))
> for i, p := range products {
>     docs[i] = p
> }
> coll.InsertMany(ctx, docs)
> ```

---

## 7. Query เอกสาร: `Find` และ `FindOne` ด้วย `bson.M`

MongoDB ใช้ **query document** แทน `WHERE` clause แบบ SQL — เขียนเป็น Go ด้วย `bson.M` (แทน "map") หรือ `bson.D` (แทน "document" ที่รักษาลำดับ key ซึ่งจำเป็นสำหรับบาง operation เช่น sort)

### `FindOne` — หาเอกสารเดียว

```go
func GetProductByName(ctx context.Context, coll *mongo.Collection, name string) (*Product, error) {
	findCtx, cancel := context.WithTimeout(ctx, 5*time.Second)
	defer cancel()

	var p Product
	err := coll.FindOne(findCtx, bson.M{"name": name}).Decode(&p)
	if err != nil {
		if errors.Is(err, mongo.ErrNoDocuments) {
			// เทียบเท่ากับ sql.ErrNoRows ใน database/sql (Part 071)
			return nil, nil
		}
		return nil, fmt.Errorf("find product: %w", err)
	}
	return &p, nil
}
```

สังเกตว่า `mongo.ErrNoDocuments` คือ error พิเศษที่คู่ขนานกับ `sql.ErrNoRows` ที่เราเจอมาแล้วใน Part 071 — เป็น sentinel error ที่ต้องเช็คด้วย `errors.Is` (ตามหลักการ error handling ใน Part 016) เพื่อแยกกรณี "หาไม่เจอ" ออกจาก "error จริงๆ"

### `Find` — หาหลายเอกสาร พร้อม cursor

```go
func ListExpensiveProducts(ctx context.Context, coll *mongo.Collection, minPrice float64) ([]Product, error) {
	listCtx, cancel := context.WithTimeout(ctx, 5*time.Second)
	defer cancel()

	filter := bson.M{"price": bson.M{"$gte": minPrice}}

	cur, err := coll.Find(listCtx, filter)
	if err != nil {
		return nil, fmt.Errorf("find products: %w", err)
	}
	defer cur.Close(listCtx) // สำคัญมาก: ต้องปิด cursor เสมอ เหมือน rows.Close() ใน database/sql

	var products []Product
	if err := cur.All(listCtx, &products); err != nil {
		return nil, fmt.Errorf("decode products: %w", err)
	}
	return products, nil
}
```

จุดที่ควรสังเกต:

- **`$gte`** คือ MongoDB query operator แปลว่า "greater than or equal" มี operator อื่นๆ ที่ใช้บ่อย: `$gt`, `$lt`, `$lte`, `$ne` (not equal), `$in` (อยู่ใน list), `$exists` (field มีอยู่หรือไม่), `$regex` (จับคู่ pattern)
- **`cur.Close(ctx)` ต้องเรียกเสมอ** — นี่คือแนวคิดเดียวกับ `rows.Close()` ใน `database/sql` (Part 071) ถ้าลืมปิด cursor จะรั่วทรัพยากรที่ server และ connection ในบาง driver mode
- **`cur.All(ctx, &products)`** เป็นวิธีสะดวกในการดึงทุกเอกสารจาก cursor มาใส่ slice เดียว เหมาะกับ result set ที่ไม่ใหญ่มาก ถ้าข้อมูลเยอะมากควรวนลูปด้วย `cur.Next(ctx)` แทนเพื่อประมวลผลทีละเอกสาร ไม่โหลดทั้งหมดเข้า memory พร้อมกัน:

```go
for cur.Next(listCtx) {
	var p Product
	if err := cur.Decode(&p); err != nil {
		return nil, err
	}
	// ประมวลผลทีละเอกสาร
}
if err := cur.Err(); err != nil {
	return nil, err
}
```

### Options: sort, limit, skip

```go
opts := options.Find().
	SetSort(bson.D{{Key: "price", Value: -1}}). // -1 = descending, 1 = ascending
	SetLimit(10).
	SetSkip(0)

cur, err := coll.Find(ctx, bson.M{}, opts)
```

`bson.D{{Key: "price", Value: -1}}` ใช้ `bson.D` แทน `bson.M` เพราะการ sort หลาย field ต้องรักษาลำดับ (map ใน Go ไม่รับประกันลำดับ key ตามที่เรียนใน Part 007)

---

## 8. Update เอกสาร: `UpdateOne`, `UpdateMany`

MongoDB update ต้องระบุ **update operator** เสมอ (ต่างจาก SQL `UPDATE table SET col = val` ที่เขียนตรงไปตรงมา) operator ที่ใช้บ่อยที่สุดคือ `$set`:

```go
func UpdateProductPrice(ctx context.Context, coll *mongo.Collection, id primitive.ObjectID, newPrice float64) error {
	updCtx, cancel := context.WithTimeout(ctx, 5*time.Second)
	defer cancel()

	filter := bson.M{"_id": id}
	update := bson.M{"$set": bson.M{"price": newPrice}}

	res, err := coll.UpdateOne(updCtx, filter, update)
	if err != nil {
		return fmt.Errorf("update product: %w", err)
	}
	if res.MatchedCount == 0 {
		return fmt.Errorf("product %s not found", id.Hex())
	}
	return nil
}
```

`UpdateResult` มี field สำคัญให้เช็ค:

- **`MatchedCount`**: จำนวนเอกสารที่ตรงกับ filter (ถ้าเป็น 0 แปลว่าไม่เจอเอกสารเลย)
- **`ModifiedCount`**: จำนวนเอกสารที่ถูกแก้ไขจริง (อาจน้อยกว่า `MatchedCount` ถ้าค่าที่ set เหมือนค่าเดิมอยู่แล้ว MongoDB จะไม่นับว่า modified)
- **`UpsertedID`**: ถ้าใช้ตัวเลือก upsert แล้วไม่เจอเอกสารเดิม จะสร้างใหม่ และ ID ของเอกสารใหม่จะอยู่ตรงนี้

operator อื่นที่ใช้บ่อย:

| Operator | ความหมาย |
|---|---|
| `$set` | กำหนดค่า field |
| `$unset` | ลบ field ออกจากเอกสาร |
| `$inc` | เพิ่ม/ลดค่าตัวเลข (atomic) |
| `$push` | เพิ่ม element เข้า array |
| `$pull` | ลบ element ออกจาก array ที่ตรงเงื่อนไข |

ตัวอย่าง `$inc` สำหรับนับ view แบบ atomic (ปลอดภัยจาก race condition แม้มีหลาย request พร้อมกัน — เทียบได้กับการใช้ `UPDATE table SET views = views + 1` ใน SQL แทนที่จะอ่านค่าออกมาบวกแล้ว set กลับ):

```go
coll.UpdateOne(ctx,
	bson.M{"_id": id},
	bson.M{"$inc": bson.M{"views": 1}},
)
```

`UpdateMany` ใช้ syntax เหมือนกันทุกประการ แต่แก้ไขทุกเอกสารที่ตรงกับ filter:

```go
res, err := coll.UpdateMany(ctx,
	bson.M{"tags": "discontinued"},
	bson.M{"$set": bson.M{"available": false}},
)
```

---

## 9. Delete เอกสาร: `DeleteOne`, `DeleteMany`

```go
func DeleteProduct(ctx context.Context, coll *mongo.Collection, id primitive.ObjectID) error {
	delCtx, cancel := context.WithTimeout(ctx, 5*time.Second)
	defer cancel()

	res, err := coll.DeleteOne(delCtx, bson.M{"_id": id})
	if err != nil {
		return fmt.Errorf("delete product: %w", err)
	}
	if res.DeletedCount == 0 {
		return fmt.Errorf("product %s not found", id.Hex())
	}
	return nil
}
```

`DeleteMany` ลบทุกเอกสารที่ตรงกับ filter ในคำสั่งเดียว — ใช้ด้วยความระมัดระวัง เพราะเหมือน `DELETE FROM table WHERE ...` ใน SQL ถ้า filter เป็น `bson.M{}` (ว่างเปล่า) จะลบ **ทุกเอกสารในคอลเลกชัน**:

```go
// อันตราย! ลบทุกเอกสารในคอลเลกชัน — ใช้เฉพาะตอนตั้งใจจริงๆ เช่น cleanup ข้อมูลทดสอบ
res, err := coll.DeleteMany(ctx, bson.M{})
```

---

## 10. `context.Context` กับทุก operation ของ MongoDB driver

นี่คือจุดที่ควรเน้นย้ำเป็นพิเศษ เพราะต่างจาก `database/sql` (Part 071) ที่มีทั้งเวอร์ชันมี context (`QueryContext`) และไม่มี context (`Query`) MongoDB driver **บังคับ**ให้ทุก method ที่ทำ I/O รับ `context.Context` เป็น parameter แรกเสมอ ไม่มีเวอร์ชันไม่มี context ให้เลือก

เหตุผลตามที่เรียนใน Part 032 (`context` package): operation ที่คุยกับเครือข่ายทุกตัวควรมี **timeout หรือ deadline** กำกับไว้เสมอ ไม่งั้นถ้า MongoDB server ค้างหรือเครือข่ายมีปัญหา โปรแกรมฝั่ง Go จะค้างรอไม่มีกำหนด (hang) ทีมพัฒนา MongoDB driver จึงออกแบบให้ signature บังคับ context ตั้งแต่ compile time เพื่อไม่ให้นักพัฒนาลืมใส่

### แนวทางปฏิบัติที่แนะนำ

```go
func (r *ProductRepository) FindByID(ctx context.Context, id primitive.ObjectID) (*Product, error) {
	// ใช้ context ที่รับมาจาก caller (เช่นจาก HTTP handler ที่มี request context)
	// ครอบด้วย timeout เฉพาะของ operation นี้ ไม่ให้เกิน 3 วินาที
	opCtx, cancel := context.WithTimeout(ctx, 3*time.Second)
	defer cancel()

	var p Product
	err := r.coll.FindOne(opCtx, bson.M{"_id": id}).Decode(&p)
	if err != nil {
		return nil, err
	}
	return &p, nil
}
```

การรับ `ctx context.Context` จาก caller แล้วครอบด้วย `context.WithTimeout` ก่อนใช้งานจริง เป็นแนวทางเดียวกับที่เราเรียนใน Part 043 (Context กับ Concurrency) — ทำให้ทั้ง **cancellation จาก caller** (เช่น client ยกเลิก HTTP request) และ **timeout เฉพาะจุด** ทำงานร่วมกันได้ ถ้า context ต้นทางถูกยกเลิกก่อน operation จะหยุดทันทีโดยไม่ต้องรอ timeout ของตัวเอง

### สิ่งที่เกิดขึ้นเมื่อ context หมดเวลา

ถ้า deadline หมดก่อนที่ server จะตอบกลับ ทุก method ของ driver จะ return error ที่ wrap `context.DeadlineExceeded` ไว้ — ตรวจสอบได้ด้วย `errors.Is(err, context.DeadlineExceeded)` ตามที่เรียนใน Part 016

---

## 11. Index เพื่อ performance ของ query

เหมือนกับฐานข้อมูล relational MongoDB ก็ scan ทุกเอกสารในคอลเลกชัน (collection scan) ถ้าไม่มี index รองรับ query นั้น ซึ่งช้ามากเมื่อข้อมูลเยอะขึ้น การสร้าง index ช่วยให้ MongoDB หาเอกสารที่ตรงเงื่อนไขได้เร็วโดยไม่ต้องอ่านทุกเอกสาร

```go
func EnsureProductIndexes(ctx context.Context, coll *mongo.Collection) error {
	idxCtx, cancel := context.WithTimeout(ctx, 10*time.Second)
	defer cancel()

	_, err := coll.Indexes().CreateMany(idxCtx, []mongo.IndexModel{
		{
			// index ธรรมดาบน field "name" เพื่อให้ FindOne(bson.M{"name": ...}) เร็วขึ้น
			Keys: bson.D{{Key: "name", Value: 1}},
			Options: options.Index().SetUnique(true), // บังคับว่าชื่อสินค้าห้ามซ้ำ
		},
		{
			// compound index: query ที่กรองด้วยทั้ง price และ tags พร้อมกันจะได้ประโยชน์
			Keys: bson.D{
				{Key: "price", Value: 1},
				{Key: "tags", Value: 1},
			},
		},
	})
	return err
}
```

หลักการเลือก field มาทำ index (แนวคิดเดียวกับ index ในฐานข้อมูล relational ที่จะกล่าวถึงเพิ่มเติมใน Part 078):

- สร้าง index บน field ที่ใช้ **filter บ่อยที่สุด** ใน query (`bson.M{"field": ...}`)
- สร้าง index บน field ที่ใช้ **sort** บ่อย (เช่น `created_at` สำหรับ query "ล่าสุดก่อน")
- **`unique` index** ใช้เมื่อต้องการบังคับว่าค่าห้ามซ้ำ (คล้าย `UNIQUE` constraint ในตาราง SQL)
- **Compound index** (index หลาย field รวมกัน) มีประโยชน์เมื่อ query กรองด้วยหลาย field พร้อมกันเป็นประจำ — ลำดับ field ใน compound index มีผลต่อ query ที่ได้ประโยชน์ ต้องเรียงจาก field ที่ใช้ equality filter ไปยัง field ที่ใช้ range filter
- **อย่าสร้าง index มากเกินจำเป็น** เพราะทุก index ที่มีจะทำให้การ insert/update ช้าลง (ต้องอัปเดต index ทุกตัวที่เกี่ยวข้องด้วยทุกครั้งที่เขียนข้อมูล) — นี่คือ trade-off ระหว่างความเร็วในการอ่านกับความเร็วในการเขียน ที่เป็นจริงในทุกฐานข้อมูล ไม่ใช่เฉพาะ MongoDB

ตรวจสอบว่า query ใช้ index จริงหรือไม่ด้วยคำสั่ง `explain` ผ่าน `mongosh`:

```javascript
db.products.find({ name: "Mechanical Keyboard" }).explain("executionStats")
```

ถ้า `winningPlan.stage` เป็น `"IXSCAN"` แปลว่าใช้ index แล้ว แต่ถ้าเป็น `"COLLSCAN"` แปลว่ากำลัง scan ทุกเอกสาร — สัญญาณว่าต้องเพิ่ม index

---

## 12. หมายเหตุความซื่อสัตย์เรื่องการรันจริงในบทนี้

เพื่อความโปร่งใส: ใน sandbox ที่ใช้เขียนบทนี้ **ไม่มี MongoDB server (`mongod`) ให้เชื่อมต่อจริง** — ไม่มี `mongod` binary ติดตั้งอยู่ ไม่มี Docker daemon ที่รันอยู่ (ตรวจสอบแล้วด้วย `docker ps` ซึ่งขึ้น error ว่าไม่มี daemon) และการดาวน์โหลดตัวติดตั้ง MongoDB จากอินเทอร์เน็ตในสภาพแวดล้อมนี้ก็ถูกบล็อกโดย network policy (ทดสอบแล้วด้วย `curl` ไปยัง `fastdl.mongodb.org` ได้ผลลัพธ์ `403 Forbidden` จาก proxy)

สิ่งที่ทำได้จริงและได้ตรวจสอบแล้วคือ:

1. **ดาวน์โหลด driver ตัวจริง** (`go.mongodb.org/mongo-driver v1.17.10`) ผ่าน `go get` สำเร็จ (Go module proxy ใช้งานได้ปกติ)
2. **เขียนโค้ดตัวอย่างทั้งหมดในบทนี้ (Connect, InsertOne, Find, FindOne, UpdateOne, DeleteOne, Index)** แล้ว **compile ผ่าน `go build` และ `go vet` สำเร็จ** จริงกับ driver เวอร์ชันจริง — ยืนยันว่า signature ของฟังก์ชัน, ชนิดข้อมูล, และการใช้ `bson.M`/`bson.D` ถูกต้องตาม API จริงทุกจุด ไม่ใช่โค้ดที่เขียนจากความจำเฉยๆ
3. **ทดลองรันโปรแกรมจริง** (`go run`) กับ `mongo.Connect` + `client.Ping` แล้วได้ผลลัพธ์ตามที่คาดไว้ทุกประการ:

```
2026/09/26 02:52:09 server selection error: context deadline exceeded, current topology:
{ Type: Unknown, Servers: [{ Addr: localhost:27017, Type: Unknown,
Last error: dial tcp 127.0.0.1:27017: connect: connection refused }, ] }
```

ผลลัพธ์นี้ยืนยันสองอย่างพร้อมกัน: (ก) ไม่มี MongoDB server รันอยู่จริงตามที่คาดไว้ และ (ข) `context.WithTimeout` ที่ครอบ `mongo.Connect`/`Ping` ทำงานถูกต้อง — เมื่อต่อไม่ได้ driver จะพยายามจนหมดเวลาแล้ว return error ที่มี `context deadline exceeded` แทนที่จะค้างไม่มีกำหนด ตรงตามหลักการใน Part 032

**สรุปคือ**: โค้ดทุกตัวอย่างในบทนี้ผ่านการ compile และ type-check จริงกับ driver เวอร์ชันจริง (ไม่ใช่แค่เดาจาก documentation) แต่ผลลัพธ์ของ query/insert/update/delete จริง (เช่นค่า `MatchedCount`, ข้อมูลที่ query ได้) เป็นสิ่งที่**อธิบายจากพฤติกรรมที่ทางการของ MongoDB driver documentation ระบุไว้** ไม่ใช่ผลจากการรันจริงกับ server เพราะไม่มี server ให้รันในสภาพแวดล้อมนี้ หากต้องการทดลองรันเองแนะนำให้ใช้ `docker run -d -p 27017:27017 mongo:7` ตามที่แนะนำไว้ในหัวข้อที่ 3 แล้ว copy โค้ดในบทนี้ไปรันได้ทันที

---

## สรุปสิ่งที่ได้เรียนในบทนี้

- MongoDB เป็น **document database** เก็บข้อมูลเป็นเอกสาร BSON ที่ schema ยืดหยุ่น ต่างจากตาราง relational ที่ schema ตายตัวและใช้ JOIN เชื่อมความสัมพันธ์
- เลือกใช้ MongoDB เมื่อข้อมูลมีโครงสร้างไม่แน่นอน/เปลี่ยนบ่อย หรืออ่าน-เขียนเป็นก้อนเอกสารเดียว และเลือก relational เมื่อต้องการ referential integrity, transaction ซับซ้อน, และ query ข้าม entity บ่อย — ระบบใหญ่จำนวนมากใช้ทั้งสองแบบร่วมกัน (polyglot persistence)
- ติดตั้งและใช้ official driver `go.mongodb.org/mongo-driver/mongo` เชื่อมต่อด้วย `mongo.Connect` แล้วยืนยันด้วย `client.Ping`
- Struct ที่ใช้กับ MongoDB ใส่ทั้ง `bson` tag (สำหรับ MongoDB) และ `json` tag (สำหรับ API response) คู่กันได้ในตัวเดียว
- CRUD หลัก: `InsertOne`/`InsertMany`, `Find`/`FindOne` กับ `bson.M` filter, `UpdateOne`/`UpdateMany` ด้วย operator เช่น `$set`/`$inc`, `DeleteOne`/`DeleteMany`
- ทุก operation ของ driver บังคับรับ `context.Context` เป็น parameter แรกเสมอ ควรครอบด้วย timeout ที่เหมาะสมทุกครั้งตามหลักการจาก Part 032
- Index ช่วยให้ query เร็วขึ้นมากโดยไม่ต้อง scan ทุกเอกสาร แต่มี trade-off คือทำให้ write ช้าลง ต้องเลือกสร้างเฉพาะที่จำเป็น
- ในบทนี้ไม่มี MongoDB server จริงให้เชื่อมต่อในสภาพแวดล้อมที่ใช้เขียน จึงได้ตรวจสอบโค้ดด้วยการ compile/vet กับ driver จริง และรัน `mongo.Connect`/`Ping` จริงเพื่อยืนยันพฤติกรรม timeout — ไม่ได้อ้างว่ารัน CRUD จริงกับ server ที่ไม่มีอยู่

## แบบฝึกหัดท้ายบท

1. ติดตั้ง MongoDB ด้วย Docker (`docker run -d -p 27017:27017 mongo:7`) แล้วรันโค้ดตัวอย่างในบทนี้ทั้งหมดจริงในเครื่องของตัวเอง เทียบผลลัพธ์กับที่อธิบายไว้ในบท
2. เขียน struct `Order` ที่มี field `Items []OrderItem` โดย `OrderItem` เป็น struct ซ้อนอยู่ข้างในที่มี `ProductName string` และ `Quantity int` แล้วลอง `InsertOne` เอกสารที่มี item ซ้อนหลายชิ้น สังเกตว่า MongoDB เก็บ nested struct เป็น embedded document อย่างไร
3. เขียนฟังก์ชัน `SearchProductsByTag(ctx context.Context, coll *mongo.Collection, tag string) ([]Product, error)` ที่ใช้ query operator `$in` เพื่อค้นหาสินค้าที่มี tag ใดๆ ตรงกับ list ของ tag ที่ส่งเข้ามา (เช่น `["electronics", "gaming"]`)
4. ทดลองสร้าง unique index บน field `email` ของ collection `users` แล้วลอง `InsertOne` สอง document ที่มี email ซ้ำกัน ดูว่า driver return error อะไร และใช้ `mongo.IsDuplicateKeyError(err)` ตรวจสอบ error นั้น
5. เปรียบเทียบเวลาในการ query 10,000 เอกสารด้วย filter บน field ที่ **มี index** เทียบกับ field ที่ **ไม่มี index** โดยใช้ `explain("executionStats")` ผ่าน `mongosh` ดูความต่างของ `executionTimeMillis`
6. ลองปรับ `context.WithTimeout` ใน `mongo.Connect` ให้สั้นมาก (เช่น 1 nanosecond) แล้วสังเกต error ที่ได้ อธิบายว่าทำไม driver ถึงบังคับให้ทุก operation ต้องมี context ตามที่อธิบายในหัวข้อที่ 10

---

**ต่อไป**: [Part 077 — Redis กับ Go](./077-redis.md)
