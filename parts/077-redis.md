# Part 077: Redis กับ Go

> ภาคที่ 6: ฐานข้อมูล (Database) — ตอนที่ 7 จาก 8 (Part 71–78)

## สารบัญของบทนี้

1. Redis คืออะไร และใช้ทำอะไรได้บ้าง
2. ติดตั้ง client library `go-redis/v9`
3. เชื่อมต่อ Redis จาก Go
4. คำสั่งพื้นฐาน: `Set`/`Get`/`Del` และการตั้งเวลาหมดอายุ
5. โครงสร้างข้อมูลอื่นของ Redis: Hash, List, Set
6. Cache-Aside Pattern: แคชหน้าฐานข้อมูลจริง
7. Redis Pub/Sub: ทางเลือกเบา ๆ แทน Message Queue
8. `context.Context` กับ go-redis
9. หมายเหตุความซื่อสัตย์เรื่องการรันจริงในบทนี้
10. สรุปสิ่งที่ได้เรียนในบทนี้
11. แบบฝึกหัดท้ายบท

---

## 1. Redis คืออะไร และใช้ทำอะไรได้บ้าง

**Redis** (REmote DIctionary Server) คือฐานข้อมูลแบบ **in-memory key-value store** — เก็บข้อมูลทั้งหมดไว้ใน RAM ทำให้อ่าน/เขียนเร็วกว่าฐานข้อมูลที่เก็บบน disk อย่าง PostgreSQL/MySQL/MongoDB หลายเท่าตัว (มักอยู่ในหลัก microsecond) แลกกับข้อจำกัดว่าข้อมูลทั้งหมดต้องพอดีกับ memory ที่มี (แม้ Redis จะรองรับการเขียนข้อมูลลง disk เพื่อ persistence ได้ก็ตาม)

เพราะความเร็วนี้ Redis จึงไม่ได้ถูกใช้แทนฐานข้อมูลหลัก (primary database) แต่ถูกใช้เป็น **ตัวช่วยเสริม** ในสถาปัตยกรรมระบบ สำหรับงานที่เน้นความเร็วและไม่จำเป็นต้องมี durability เข้มงวดแบบฐานข้อมูลหลัก:

| กรณีใช้งาน | อธิบาย |
|---|---|
| **Caching** | เก็บผลลัพธ์ query ที่คำนวณ/ดึงจากฐานข้อมูลหลักไว้ชั่วคราว ลดภาระฐานข้อมูลหลักและลด latency ของ request ถัดไป (หัวข้อที่ 6 ในบทนี้) |
| **Session storage** | เก็บข้อมูล session ของผู้ใช้ที่ login อยู่ — ตามที่กล่าวถึงใน Part 068 (OAuth2 และ Session) ระบบเว็บที่มีหลาย server instance มักเก็บ session ไว้ที่ Redis กลาง แทนที่จะเก็บใน memory ของแต่ละ server (ซึ่งจะทำให้ user ต้อง login ใหม่ถ้า request ไปตกที่ server คนละตัว) |
| **Rate limiting** | นับจำนวน request ต่อ user/IP ในหน้าต่างเวลาหนึ่ง ๆ โดยใช้ `INCR` + `EXPIRE` แบบ atomic เพื่อจำกัดว่า user เรียก API ได้กี่ครั้งต่อวินาที/นาที |
| **Pub/Sub** | ส่งข้อความแบบ broadcast แบบเรียลไทม์ระหว่าง process/service (หัวข้อที่ 7) |
| **Queue อย่างง่าย** | ใช้ List (`LPUSH`/`RPOP`) เป็นคิวงานพื้นฐาน เหมาะกับงานที่ไม่ต้องการ guarantee การส่งมอบข้อความระดับสูงแบบที่ RabbitMQ/Kafka มี (จะเรียนเจาะลึกใน Part 091–092) |
| **Leaderboard / counting** | ใช้ Sorted Set เก็บคะแนนแล้ว query "10 อันดับแรก" ได้เร็วมาก |

จุดสำคัญที่ต้องเข้าใจ: Redis **ไม่ใช่** ฐานข้อมูลทดแทน PostgreSQL/MongoDB มันเป็นเครื่องมือเสริมที่ทำงานคู่กับฐานข้อมูลหลักเสมอ — ข้อมูลที่ Redis เก็บมักเป็นข้อมูลที่ "สร้างใหม่ได้" ถ้าหายไป (เช่น cache หายก็แค่ไป query ฐานข้อมูลหลักใหม่) ไม่ใช่ข้อมูล source of truth เพียงแหล่งเดียว

---

## 2. ติดตั้ง client library `go-redis/v9`

client library ที่นิยมที่สุดสำหรับ Go คือ [`github.com/redis/go-redis/v9`](https://github.com/redis/go-redis) (เดิมชื่อ `go-redis/redis` แต่ปัจจุบันย้ายมาอยู่ใต้ organization ทางการของ Redis แล้ว):

```bash
go get github.com/redis/go-redis/v9
```

> สังเกต `/v9` ต่อท้าย path — เป็น major version ปัจจุบันของ library (ตามหลักการ semantic versioning ของ Go module ที่ major version ตั้งแต่ v2 ขึ้นไปต้องระบุไว้ใน import path ด้วย)

ในบทนี้ผู้เขียนได้ติดตั้งและรัน Redis server จริงในสภาพแวดล้อมที่ใช้เขียนบทความ (`redis-server` มีอยู่ในเครื่องแล้ว) และรันโค้ดทุกตัวอย่างในบทนี้จริงกับ server ตัวนั้น ผลลัพธ์ที่แสดงในบทความคือผลลัพธ์จริงจากการรัน ไม่ใช่ผลลัพธ์ที่แต่งขึ้น

---

## 3. เชื่อมต่อ Redis จาก Go

```go
package main

import (
	"context"
	"fmt"
	"log"

	"github.com/redis/go-redis/v9"
)

func main() {
	ctx := context.Background()

	rdb := redis.NewClient(&redis.Options{
		Addr:     "localhost:6379", // host:port ของ Redis server
		Password: "",               // ใส่ password ถ้า Redis ตั้งค่า requirepass ไว้
		DB:       0,                // Redis มี logical database 0-15 แยกข้อมูลได้ในตัวเดียวกัน
	})
	defer rdb.Close()

	pong, err := rdb.Ping(ctx).Result()
	if err != nil {
		log.Fatalf("connect to redis failed: %v", err)
	}
	fmt.Println("PING ->", pong)
}
```

**ผลลัพธ์จากการรันจริง**:

```
PING -> PONG
```

จุดสำคัญ:

- **`redis.NewClient` ไม่ได้เชื่อมต่อทันที** เหมือนกับ `mongo.Connect` ใน Part 076 — เป็น lazy connection ที่จะสร้าง TCP connection จริงเมื่อมีคำสั่งแรกถูกเรียก การเรียก `Ping` คือวิธียืนยันว่าต่อ Redis ได้จริง
- **`*redis.Client` เป็น connection pool ในตัว** ไม่ใช่ connection เดี่ยว — ควรสร้างครั้งเดียวตอน startup แล้วใช้ตลอดอายุโปรแกรม เหมือนกับ `*sql.DB` (Part 071) และ `*mongo.Client` (Part 076) ทุกประการ ห้ามสร้าง client ใหม่ทุกครั้งที่จะใช้งาน
- `redis.Options` มี field อื่นที่ควรตั้งค่าใน production เช่น `PoolSize` (จำนวน connection สูงสุดใน pool — คู่ขนานกับ `SetMaxOpenConns` ใน Part 078), `DialTimeout`, `ReadTimeout`, `WriteTimeout`

---

## 4. คำสั่งพื้นฐาน: `Set`/`Get`/`Del` และการตั้งเวลาหมดอายุ

```go
// SET แบบไม่มี expiration (0 = ไม่หมดอายุ)
err := rdb.Set(ctx, "user:1:name", "Somchai", 0).Err()

// GET
name, err := rdb.Get(ctx, "user:1:name").Result()
```

**ผลลัพธ์จากการรันจริง**:

```
GET user:1:name -> Somchai
```

### ตั้งเวลาหมดอายุตอน set: `SetEx` (หรือ `Set` พร้อมระบุ TTL)

ในระบบเว็บที่เก็บ session (ตามที่กล่าวถึงใน Part 068) แทบทุกครั้งที่เก็บ session ลง Redis ต้องกำหนดเวลาหมดอายุด้วยเสมอ ไม่งั้น session ที่ผู้ใช้ไม่ได้ logout เองจะค้างอยู่ใน Redis ตลอดไป:

```go
// SetEx: set ค่าพร้อมกำหนด TTL ในคำสั่งเดียว
err := rdb.SetEx(ctx, "session:abc123", "user:1", 2*time.Second).Err()

// ตรวจสอบเวลาที่เหลือด้วย TTL
ttl, _ := rdb.TTL(ctx, "session:abc123").Result()
```

**ผลลัพธ์จากการรันจริง**:

```
TTL session:abc123 -> 2s
```

> เทียบเท่ากับการเรียก `rdb.Set(ctx, "session:abc123", "user:1", 2*time.Second)` โดยตรง — `go-redis` รวม `SET` + expiration ไว้ใน method `Set` ตัวเดียวผ่าน parameter สุดท้าย (`expiration time.Duration`) อยู่แล้ว ส่วน `SetEx`/`SetNX` เป็น method แยกที่ map ตรงกับคำสั่ง Redis ดั้งเดิม `SETEX`/`SETNX` สำหรับคนที่คุ้นเคยกับชื่อคำสั่งเดิม

### `Expire` — ตั้งเวลาหมดอายุให้ key ที่มีอยู่แล้ว

```go
rdb.Set(ctx, "temp:key", "value", 0) // ตอนแรกไม่มีวันหมดอายุ
rdb.Expire(ctx, "temp:key", 5*time.Second) // มาตั้งทีหลังได้
```

**ผลลัพธ์จากการรันจริง**:

```
TTL temp:key -> 5s
```

### `Del` — ลบ key

```go
n, err := rdb.Del(ctx, "temp:key").Result()
// n คือจำนวน key ที่ลบสำเร็จจริง (ถ้า key ไม่มีอยู่แล้ว n จะเป็น 0)
```

**ผลลัพธ์จากการรันจริง**:

```
DEL temp:key -> 1 <nil>
```

### กรณี key ไม่มีอยู่: `redis.Nil`

เมื่อ `Get` เจอ key ที่ไม่มีอยู่ go-redis จะ return error พิเศษชื่อ `redis.Nil` — เป็น sentinel error เหมือนกับ `sql.ErrNoRows` (Part 071) และ `mongo.ErrNoDocuments` (Part 076) ทุกประการ เช็คด้วย `errors.Is` หรือเทียบตรง ๆ ก็ได้เพราะเป็น package-level variable:

```go
val, err := rdb.Get(ctx, "no:such:key").Result()
if err == redis.Nil {
	// key ไม่มีอยู่ ไม่ใช่ error จริง — เป็นกรณีปกติที่ต้อง handle
} else if err != nil {
	// error จริง เช่น เชื่อมต่อ Redis ไม่ได้
}
```

**ผลลัพธ์จากการรันจริง**:

```
GET missing key err -> redis: nil == redis.Nil: true
```

นี่คือรูปแบบ error ที่ซ้ำไปซ้ำมาในทุกฐานข้อมูลที่เราเรียนในภาคนี้ — **"หาไม่เจอ" ไม่ใช่ error ทางเทคนิค แต่เป็นผลลัพธ์ปกติที่ต้อง handle แยกจาก error จริง** เสมอ ไม่ว่าจะเป็น `database/sql`, MongoDB, หรือ Redis

---

## 5. โครงสร้างข้อมูลอื่นของ Redis: Hash, List, Set

Redis ไม่ได้เก็บได้แค่ string ธรรมดา แต่มีโครงสร้างข้อมูลในตัวหลายแบบที่ทำงานฝั่ง server โดยตรง (ไม่ต้องดึงมาประมวลผลที่ฝั่ง client)

### Hash — เหมาะกับข้อมูลแบบ object/record

Hash คือ key เดียวที่เก็บ field-value ได้หลายคู่ข้างใน เหมาะมากสำหรับเก็บข้อมูลแบบ record เช่นโปรไฟล์ผู้ใช้ (แทนที่จะแปลง struct ทั้งก้อนเป็น JSON string เก็บใน key เดียว การใช้ Hash ทำให้แก้ไข field เดียวได้โดยไม่ต้องอ่าน-เขียนทั้งก้อน):

```go
// HSet: ตั้งหลาย field พร้อมกันในคำสั่งเดียว
err := rdb.HSet(ctx, "user:1", map[string]interface{}{
	"name":  "Somchai",
	"email": "somchai@example.com",
	"age":   "30",
}).Err()

// HGetAll: ดึงทุก field-value ออกมาเป็น map[string]string
userHash, err := rdb.HGetAll(ctx, "user:1").Result()

// HGet: ดึงเฉพาะ field เดียว
email, err := rdb.HGet(ctx, "user:1", "email").Result()
```

**ผลลัพธ์จากการรันจริง**:

```
HGETALL user:1 -> map[age:30 email:somchai@example.com name:Somchai]
HGET user:1 email -> somchai@example.com
```

### List — เหมาะกับข้อมูลที่เรียงลำดับ เช่น timeline, log ล่าสุด, คิวงาน

```go
// LPush: เพิ่ม element เข้าด้านซ้าย (หัว) ของ list
err := rdb.LPush(ctx, "recent:logins", "user:3", "user:2", "user:1").Err()

// LRange: ดึง element ตามช่วง index (0 ถึง -1 = ทั้งหมด)
logins, err := rdb.LRange(ctx, "recent:logins", 0, -1).Result()
```

**ผลลัพธ์จากการรันจริง**:

```
LRANGE recent:logins 0 -1 -> [user:1 user:2 user:3]
```

สังเกตลำดับผลลัพธ์: เพราะ `LPush` ใส่แต่ละค่าไว้ที่ **หัว** ของ list ทีละตัว ค่าสุดท้ายที่ push (`"user:1"`) จะไปอยู่หน้าสุด — เป็นพฤติกรรมมาตรฐานของ Redis `LPUSH` ที่ต้องระวังถ้าคาดหวังลำดับตามที่ใส่เข้าไป

ถ้าต้องการใช้ List เป็น **คิวงานอย่างง่าย**: producer เรียก `LPush` เพื่อใส่งานใหม่ และ worker เรียก `RPop` (หรือ `BRPop` แบบ blocking ที่รอจนกว่าจะมีงานเข้าคิว) เพื่อดึงงานที่เก่าที่สุดออกมาทำ — นี่คือรูปแบบพื้นฐานที่สุดของคิว ก่อนจะไปเรียนระบบคิวที่ทรงพลังกว่าอย่าง RabbitMQ/Kafka ใน Part 091–092

### Set — เหมาะกับข้อมูลที่ไม่ซ้ำกันและไม่สนใจลำดับ

```go
// SAdd: เพิ่ม member เข้า set (ค่าซ้ำจะถูกข้ามอัตโนมัติ)
err := rdb.SAdd(ctx, "tags:post:42", "go", "redis", "backend", "go").Err()

// SMembers: ดึงทุก member ออกมา
members, err := rdb.SMembers(ctx, "tags:post:42").Result()

// SIsMember: เช็คว่ามี member นี้อยู่ใน set หรือไม่ (O(1) เร็วมาก)
isMember, err := rdb.SIsMember(ctx, "tags:post:42", "go").Result()
```

**ผลลัพธ์จากการรันจริง**:

```
SMEMBERS tags:post:42 -> [redis go backend]
SISMEMBER tags:post:42 go -> true
```

สังเกตว่าแม้จะ `SAdd` คำว่า `"go"` ซ้ำสองครั้ง แต่ผลลัพธ์มีแค่ตัวเดียว — เป็นคุณสมบัติพื้นฐานของ Set ที่รับประกันว่าไม่มี duplicate (ลำดับผลลัพธ์ของ `SMembers` ไม่รับประกันว่าตรงกับลำดับที่ใส่เข้าไป เพราะ Redis เก็บ Set แบบไม่มีลำดับภายใน)

---

## 6. Cache-Aside Pattern: แคชหน้าฐานข้อมูลจริง

**Cache-Aside** (บางครั้งเรียก Lazy Loading) คือ pattern การใช้ cache ที่นิยมที่สุด หลักการคือ:

1. เมื่อมี request เข้ามา **เช็ค cache ก่อน**
2. ถ้าเจอ (**cache hit**) — ส่งค่าจาก cache กลับไปเลย ไม่ต้องแตะฐานข้อมูลหลัก
3. ถ้าไม่เจอ (**cache miss**) — ไป query ฐานข้อมูลหลัก (ตามที่เรียนใน Part 071–075) แล้ว **เขียนผลลัพธ์เข้า cache** ก่อนส่งกลับ เพื่อให้ request ถัดไปที่ถามข้อมูลเดียวกัน hit cache ได้

```go
func fakeQueryProductFromDB(id int) (string, error) {
	time.Sleep(50 * time.Millisecond) // จำลอง latency ของการ query ฐานข้อมูลหลัก
	return fmt.Sprintf(`{"id":%d,"name":"Mechanical Keyboard","price":1990}`, id), nil
}

func GetProduct(ctx context.Context, rdb *redis.Client, productID int) (string, error) {
	key := fmt.Sprintf("product:%d", productID)

	// ขั้นตอนที่ 1: เช็ค cache ก่อนเสมอ
	val, err := rdb.Get(ctx, key).Result()
	if err == redis.Nil {
		// ขั้นตอนที่ 2: cache miss -> query ฐานข้อมูลหลัก
		val, err = fakeQueryProductFromDB(productID)
		if err != nil {
			return "", fmt.Errorf("query db: %w", err)
		}
		// ขั้นตอนที่ 3: populate cache ด้วย TTL (อย่าลืม TTL! ไม่งั้นข้อมูลเก่าจะค้างตลอดไปแม้ฐานข้อมูลหลักเปลี่ยนแล้ว)
		if err := rdb.Set(ctx, key, val, 30*time.Second).Err(); err != nil {
			// การเขียน cache ล้มเหลวไม่ควรทำให้ทั้ง request fail — log ไว้แล้วส่งค่ากลับตามปกติ
			log.Println("warning: failed to populate cache:", err)
		}
	} else if err != nil {
		return "", fmt.Errorf("redis get: %w", err)
	}

	return val, nil
}
```

**ผลลัพธ์จากการรันจริง** (เรียกฟังก์ชันเดียวกันสองครั้งติดกันด้วย `productID` เดียวกัน):

```
--- Cache-aside demo ---
cache MISS for product:7 - querying DB...
result: {"id":7,"name":"Mechanical Keyboard","price":1990} (took 50.618347ms)
cache HIT for product:7
result: {"id":7,"name":"Mechanical Keyboard","price":1990} (took 115.62µs)
```

ผลลัพธ์นี้แสดงประเด็นสำคัญที่สุดของ caching ให้เห็นชัดเจน: การเรียกครั้งแรก (cache miss) ใช้เวลา **~50 มิลลิวินาที** เพราะต้องรอ "ฐานข้อมูล" (จำลอง `time.Sleep(50ms)`) แต่การเรียกครั้งที่สอง (cache hit) ใช้เวลาแค่ **~0.1 มิลลิวินาที** — เร็วขึ้นกว่า 400 เท่า เพราะอ่านจาก memory ของ Redis โดยตรง ไม่ต้องแตะฐานข้อมูลหลักเลย

### ข้อควรระวังของ Cache-Aside

- **Cache invalidation**: เมื่อข้อมูลในฐานข้อมูลหลักเปลี่ยน (เช่น update ราคาสินค้า) ต้องลบหรืออัปเดต cache ที่เกี่ยวข้องด้วย ไม่งั้นผู้ใช้จะเห็นข้อมูลเก่า วิธีที่ง่ายที่สุดคือ `rdb.Del(ctx, key)` ทันทีหลัง update ฐานข้อมูลหลักสำเร็จ (invalidate แล้วให้ request ถัดไป miss แล้วโหลดใหม่ แทนที่จะพยายามอัปเดต cache ให้ตรงเป๊ะซึ่งเสี่ยง bug มากกว่า)
- **TTL เป็นตาข่ายนิรภัยสุดท้าย**: แม้จะ invalidate cache ทุกจุดที่ควร แต่การตั้ง TTL ไว้เสมอ (เช่น 30 วินาทีถึงไม่กี่นาทีแล้วแต่ความสด ("freshness") ของข้อมูลที่ต้องการ) ช่วยป้องกันไม่ให้ข้อมูลเก่าค้างอยู่ตลอดไปหากมี bug ที่ลืม invalidate จุดใดจุดหนึ่ง
- **Thundering herd / cache stampede**: ถ้า key ยอดนิยมหมดอายุพร้อมกันตอนมี traffic สูง หลาย request อาจ miss cache พร้อมกันแล้วยิง query ไปที่ฐานข้อมูลหลักพร้อมกันหมด ทำให้ฐานข้อมูลหลักโอเวอร์โหลด วิธีแก้เบื้องต้นคือใช้ lock สั้น ๆ ใน Redis (`SetNX`) ให้มีแค่ request เดียวที่ query ฐานข้อมูลจริง ส่วนที่เหลือรอผลจาก request แรก

---

## 7. Redis Pub/Sub: ทางเลือกเบา ๆ แทน Message Queue

Redis รองรับรูปแบบ **Publish/Subscribe** ในตัว — ผู้ส่ง (publisher) ส่งข้อความไปยัง "channel" หนึ่ง ๆ และผู้รับทุกคนที่ subscribe channel นั้นอยู่จะได้รับข้อความแบบเรียลไทม์

```go
func subscriber(ctx context.Context, rdb *redis.Client) {
	sub := rdb.Subscribe(ctx, "notifications")
	defer sub.Close()

	// รอ confirmation ว่า subscribe สำเร็จก่อนเริ่มรับข้อความ
	if _, err := sub.Receive(ctx); err != nil {
		log.Fatal(err)
	}

	ch := sub.Channel() // ได้ Go channel ปกติที่ส่ง *redis.Message มาเรื่อย ๆ
	for msg := range ch {
		fmt.Printf("received on %s: %s\n", msg.Channel, msg.Payload)
	}
}

func publisher(ctx context.Context, rdb *redis.Client) {
	n, err := rdb.Publish(ctx, "notifications", "order:1001 shipped").Result()
	if err != nil {
		log.Fatal(err)
	}
	fmt.Println("delivered to", n, "subscriber(s)")
}
```

**ผลลัพธ์จากการรันจริง** (subscriber ทำงานใน goroutine แยก แล้ว publish จาก main goroutine):

```
--- Pub/Sub demo ---
PUBLISH delivered to 1 subscriber(s)
received on notifications: order:1001 shipped
```

`sub.Channel()` คืน Go `<-chan *redis.Message` ธรรมดา ทำให้ผสานเข้ากับแนวคิด channel และ `select` ที่เรียนใน Part 037–038 ได้อย่างเป็นธรรมชาติ — ตัวอย่างเช่นรอรับข้อความพร้อมกับ timeout:

```go
select {
case msg := <-ch:
	fmt.Println("got:", msg.Payload)
case <-time.After(5 * time.Second):
	fmt.Println("no message received within 5s")
case <-ctx.Done():
	return ctx.Err()
}
```

### Redis Pub/Sub vs Message Queue (RabbitMQ/Kafka)

ก่อนไปเรียน Message Queue เต็มรูปแบบใน Part 091–092 ควรเข้าใจข้อจำกัดสำคัญของ Redis Pub/Sub เทียบกับ message queue จริงจัง:

| คุณสมบัติ | Redis Pub/Sub | RabbitMQ / Kafka |
|---|---|---|
| ข้อความที่ไม่มีใคร subscribe อยู่ | **หายทันที** ไม่มีการเก็บไว้ | เก็บไว้ในคิว/log รอผู้รับมาอ่านทีหลังได้ |
| Guarantee การส่งมอบ | ไม่มี (fire-and-forget) | มี (at-least-once หรือมากกว่า พร้อม acknowledgment) |
| Persistence | ไม่มี (อยู่ใน memory ชั่วคราวระหว่างส่ง) | มี (เขียนลง disk ได้) |
| เหมาะกับ | แจ้งเตือนแบบเรียลไทม์ที่พลาดได้บ้าง (เช่น "มีคนพิมพ์อยู่" ใน chat, live notification บน dashboard) | งานที่ต้องประมวลผลแน่นอนทุกชิ้น เช่น คำสั่งซื้อ, การชำระเงิน, event ที่ห้ามหาย |

สรุปสั้น ๆ: ถ้าข้อความหายไปบ้างแล้วไม่กระทบระบบ (เช่น notification สด ๆ ที่ refresh หน้าเว็บก็เห็นข้อมูลล่าสุดอยู่ดี) Redis Pub/Sub เพียงพอและเบากว่ามาก แต่ถ้าข้อความทุกชิ้นสำคัญและห้ามหาย ต้องใช้ message queue ตัวเต็มอย่าง RabbitMQ หรือ Kafka

---

## 8. `context.Context` กับ go-redis

เหมือนกับ MongoDB driver ใน Part 076 คำสั่งของ go-redis v9 ทุกตัวรับ `context.Context` เป็น parameter แรกเสมอ (นี่คือการเปลี่ยนแปลงหลักจาก go-redis v8 ที่เพิ่งบังคับ context ในทุก method) หลักการใช้งานเหมือนกันทุกประการกับที่เรียนมาตลอดภาคนี้:

```go
func (s *ProductService) GetPrice(ctx context.Context, productID int) (int, error) {
	opCtx, cancel := context.WithTimeout(ctx, 200*time.Millisecond) // Redis ควรตอบเร็วมาก timeout สั้นได้
	defer cancel()

	val, err := s.rdb.Get(opCtx, fmt.Sprintf("price:%d", productID)).Int()
	if err != nil {
		return 0, err
	}
	return val, nil
}
```

ข้อสังเกตที่ควรจำ: เพราะ Redis เป็น in-memory และปกติตอบกลับเร็วมาก (หลัก microsecond ถึง low millisecond) การตั้ง timeout ให้ Redis command **ควรสั้นกว่า** timeout ที่ตั้งให้ฐานข้อมูลหลักอย่างมาก — ถ้า Redis ตอบช้าเกิน 100-200ms มักแปลว่ามีปัญหาเครือข่ายหรือ Redis เอง โปรแกรมควร fail fast แทนที่จะรอนาน (โดยเฉพาะถ้า Redis ถูกใช้เป็น cache — การ fallback ไปอ่านฐานข้อมูลหลักโดยตรงเมื่อ Redis timeout ยังดีกว่าปล่อยให้ request ทั้งหมดค้าง)

---

## 9. หมายเหตุความซื่อสัตย์เรื่องการรันจริงในบทนี้

ต่างจาก Part 076 (MongoDB) ที่ไม่มี server ให้ต่อจริง **บทนี้มี Redis server ตัวจริงรันอยู่ในสภาพแวดล้อมที่ใช้เขียนบทความ** (`redis-server` ติดตั้งอยู่แล้วในเครื่อง) ผู้เขียนได้:

1. สั่งรัน `redis-server --daemonize yes --port 6379` แล้วตรวจสอบด้วย `redis-cli ping` ได้ผลลัพธ์ `PONG` ยืนยันว่า server พร้อมใช้งานจริง
2. ดาวน์โหลด `github.com/redis/go-redis/v9` เวอร์ชันจริง (v9.22.0) ผ่าน `go get`
3. เขียนโปรแกรม Go ที่ครอบคลุมทุกคำสั่งที่กล่าวถึงในบทนี้ (`Ping`, `Set`/`Get`/`Del`, `SetEx`/`Expire`/`TTL`, `HSet`/`HGetAll`/`HGet`, `LPush`/`LRange`, `SAdd`/`SMembers`/`SIsMember`, cache-aside pattern, `Publish`/`Subscribe`) แล้ว **รันจริงด้วย `go run` กับ Redis server ตัวจริง**
4. ผลลัพธ์ทุกก้อนที่แสดงในบทความ (คำอธิบาย "ผลลัพธ์จากการรันจริง") คือ **output ที่ copy มาจากการรันจริงทุกตัวอักษร** ไม่ได้แต่งขึ้นเอง

ดังนั้นเนื้อหาในบทนี้ทั้งหมด **ได้รับการยืนยันด้วยการรันจริงกับ Redis server จริง** ไม่ใช่แค่จากเอกสารหรือความจำ — ต่างจาก Part 076 ที่ต้องพึ่งการ compile-check เพราะไม่มี MongoDB server ให้ต่อในสภาพแวดล้อมเดียวกัน

---

## สรุปสิ่งที่ได้เรียนในบทนี้

- Redis คือ in-memory key-value store ที่เร็วมาก ใช้เป็นตัวช่วยเสริมคู่กับฐานข้อมูลหลัก ไม่ใช่ตัวแทนฐานข้อมูลหลัก — ใช้ทำ caching, session storage (ต่อเนื่องจาก Part 068), rate limiting, pub/sub, และคิวงานอย่างง่าย
- Client library มาตรฐานคือ `github.com/redis/go-redis/v9` เชื่อมต่อด้วย `redis.NewClient` แล้วยืนยันด้วย `Ping`
- คำสั่งพื้นฐาน `Set`/`Get`/`Del` ทำงานแบบ key-value ตรงไปตรงมา ตั้งเวลาหมดอายุได้ด้วย `SetEx` หรือ `Expire`/`TTL`
- Redis มีโครงสร้างข้อมูลในตัวหลายแบบที่ทำงานฝั่ง server: Hash (`HSet`/`HGetAll`) เหมาะกับข้อมูลแบบ record, List (`LPush`/`LRange`) เหมาะกับข้อมูลเรียงลำดับ/คิวอย่างง่าย, Set (`SAdd`/`SMembers`) เหมาะกับข้อมูลไม่ซ้ำที่ไม่สนลำดับ
- **Cache-Aside Pattern**: เช็ค cache ก่อน ถ้า miss ค่อยไป query ฐานข้อมูลหลัก (Part 071–075) แล้วเขียนผลลัพธ์กลับเข้า cache พร้อม TTL — การทดสอบจริงในบทนี้แสดงให้เห็นว่า cache hit เร็วกว่า cache miss กว่า 400 เท่า
- **Pub/Sub** เป็นทางเลือกเบา ๆ สำหรับการส่งข้อความแบบเรียลไทม์ แต่ไม่มี guarantee การส่งมอบและไม่ persist ข้อความ — งานที่ห้ามข้อความหายต้องใช้ message queue เต็มรูปแบบอย่าง RabbitMQ/Kafka (Part 091–092)
- ทุกคำสั่งของ go-redis v9 บังคับรับ `context.Context` เป็น parameter แรก ควรตั้ง timeout ให้สั้น เพราะ Redis ปกติตอบเร็วมาก
- บทนี้มี Redis server จริงให้ทดสอบในสภาพแวดล้อมที่เขียนบทความ โค้ดทุกตัวอย่างถูกรันจริงและผลลัพธ์ที่แสดงคือ output จริง ไม่ใช่การจำลอง

## แบบฝึกหัดท้ายบท

1. เขียนฟังก์ชัน `IncrementRateLimit(ctx context.Context, rdb *redis.Client, userID string, limit int, window time.Duration) (bool, error)` ที่ใช้ `INCR` เพิ่มตัวนับของ `userID` และตั้ง `EXPIRE` เฉพาะตอนที่เป็นการเพิ่มครั้งแรก (ค่าตัวนับเท่ากับ 1) แล้ว return `false` ถ้าตัวนับเกิน `limit` ภายในหน้าต่างเวลานั้น
2. ปรับ cache-aside pattern ในหัวข้อที่ 6 ให้ทำ **cache invalidation**: เมื่อมีการเรียก `UpdateProductPrice` ให้เรียก `rdb.Del(ctx, key)` เพื่อลบ cache เก่าทันทีหลัง update ฐานข้อมูลหลักสำเร็จ
3. ใช้ `SETNX` (หรือ `SetNX` ใน go-redis) เขียนฟังก์ชัน distributed lock อย่างง่าย `AcquireLock(ctx, rdb, lockKey string, ttl time.Duration) (bool, error)` ที่คืนค่า `true` ถ้าได้ lock สำเร็จ (ป้องกัน cache stampede ตามที่กล่าวถึงในหัวข้อที่ 6)
4. เขียนโปรแกรมที่มี publisher ส่งข้อความเข้า channel `"chat:room:1"` ทุก 1 วินาที และมี subscriber 2 ตัวรับข้อความพร้อมกัน สังเกตว่าทั้งสอง subscriber ได้รับข้อความเดียวกันครบทุกข้อความหรือไม่
5. ใช้ Sorted Set (`ZAdd`/`ZRevRange`) เขียนระบบ leaderboard อย่างง่ายที่เก็บคะแนนผู้เล่นและ query "10 อันดับแรก" ได้ (ค้นเพิ่มเติมเรื่อง Sorted Set จาก Redis documentation เพราะไม่ได้กล่าวถึงในบทนี้)
6. ทดลองปิด Redis server ระหว่างที่โปรแกรมกำลังเรียก `Get` อยู่ (เช่นด้วย `redis-cli shutdown nosave`) แล้วสังเกต error ที่เกิดขึ้น และดูว่า `context.WithTimeout` ที่ครอบไว้ช่วยให้โปรแกรม fail fast แทนที่จะค้างได้อย่างไร

---

**ต่อไป**: [Part 078 — Database Transactions และ Connection Pooling](./078-transactions-and-pooling.md)
