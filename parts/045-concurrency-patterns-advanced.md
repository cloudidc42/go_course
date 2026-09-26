# Part 045: Concurrency Patterns ขั้นสูง: Pipeline, Pub-Sub

> ภาคที่ 3: การทำงานพร้อมกัน (Concurrency) — ตอนที่ 10 จาก 10 (Part 36–45)

## สารบัญของบทนี้

1. ทำให้ Pipeline เป็นทางการด้วย Generics
2. Pub-Sub Pattern: แนวคิดและความแตกต่างจาก Fan-out
3. ตัวอย่างสมบูรณ์: Pub-Sub Broker ด้วย Channel และ Mutex
4. Rate Limiting ด้วย `time.Ticker`
5. Rate Limiting ด้วย `golang.org/x/time/rate`
6. Semaphore ผ่าน Buffered Channel
7. ตารางสรุป: เลือก Concurrency Pattern ไหนเมื่อไร (Part 36-45)
8. จบภาคที่ 3: สิ่งที่ควรฝึกฝนต่อ
9. สรุปสิ่งที่ได้เรียนในบทนี้
10. แบบฝึกหัดท้ายบท

---

## 1. ทำให้ Pipeline เป็นทางการด้วย Generics

ใน Part 042 เราสร้าง pipeline จาก stage ที่เขียนเฉพาะสำหรับ `int` (`generator`, `square`) ซึ่งใช้งานได้ดีแต่ต้องเขียนโค้ดใหม่ทุกครั้งที่ชนิดข้อมูลเปลี่ยน ตอนนี้เรามีเครื่องมือที่เรียนไปแล้วใน **Part 028-029 (Generics)** มาช่วยทำให้ pipeline stage เป็น "แม่แบบ" ที่ใช้ซ้ำได้กับข้อมูลทุกชนิด

หลักการคือกำหนดรูปแบบ (type) ของ stage หนึ่งขั้นให้ชัดเจน:

```go
// Stage คือ "รูปแบบมาตรฐาน" ของ pipeline stage หนึ่งขั้น: รับ ctx + input channel
// คืนค่าเป็น output channel เดียว ใช้ generics (จาก part 028-029) ทำให้ stage
// หนึ่งตัวนำไปต่อกับ stage ชนิดข้อมูลอื่นได้อย่างปลอดภัยในเชิง type
type Stage[In, Out any] func(ctx context.Context, in <-chan In) <-chan Out
```

จากนั้นเขียนฟังก์ชันสร้าง stage แบบ generic ที่ใช้ได้กับข้อมูลทุกชนิด:

```go
// Generate เป็น stage แรกสุดของทุก pipeline (generator pattern จาก part 042)
func Generate[T any](ctx context.Context, items ...T) <-chan T {
	out := make(chan T)
	go func() {
		defer close(out)
		for _, item := range items {
			select {
			case out <- item:
			case <-ctx.Done():
				return
			}
		}
	}()
	return out
}

// Map สร้าง Stage ทั่วไปจากฟังก์ชันแปลงค่าใดๆ — นำไป compose กับ stage อื่นได้เรื่อยๆ
func Map[In, Out any](f func(In) Out) Stage[In, Out] {
	return func(ctx context.Context, in <-chan In) <-chan Out {
		out := make(chan Out)
		go func() {
			defer close(out)
			for v := range in {
				select {
				case out <- f(v):
				case <-ctx.Done():
					return
				}
			}
		}()
		return out
	}
}

// Filter เป็นอีกหนึ่ง stage มาตรฐาน: ผ่านเฉพาะค่าที่ตรงเงื่อนไข
func Filter[T any](keep func(T) bool) Stage[T, T] {
	return func(ctx context.Context, in <-chan T) <-chan T {
		out := make(chan T)
		go func() {
			defer close(out)
			for v := range in {
				if !keep(v) {
					continue
				}
				select {
				case out <- v:
				case <-ctx.Done():
					return
				}
			}
		}()
		return out
	}
}
```

ตอนนี้เราประกอบ pipeline ได้โดยเลือก stage มาต่อกันเหมือนเลโก้ โดยไม่ต้องเขียนโค้ด goroutine เองซ้ำในทุก stage:

```go
func main() {
	ctx, cancel := context.WithCancel(context.Background())
	defer cancel()

	// ประกอบ pipeline จาก stage สำเร็จรูป: generate -> filter เลขคู่ -> map ยกกำลังสอง
	nums := Generate(ctx, 1, 2, 3, 4, 5, 6, 7, 8, 9, 10)
	evens := Filter(func(n int) bool { return n%2 == 0 })(ctx, nums)
	squared := Map(func(n int) int { return n * n })(ctx, evens)

	for v := range squared {
		fmt.Println("ผลลัพธ์:", v)
	}
}
```

รันด้วย `go run -race main.go` ผลลัพธ์:

```
ผลลัพธ์: 4
ผลลัพธ์: 16
ผลลัพธ์: 36
ผลลัพธ์: 64
ผลลัพธ์: 100
```

(กรองเฉพาะเลขคู่ 2,4,6,8,10 แล้วยกกำลังสอง = 4,16,36,64,100 ตรงตามที่คาดไว้)

สังเกตว่าทุก stage รับ `ctx` เป็นพารามิเตอร์ตัวแรกตามธรรมเนียมที่เรียนไปใน Part 043 และทุก `select` มี `case <-ctx.Done()` กำกับไว้ ทำให้ pipeline ทั้งสายยกเลิกได้ทันทีเมื่อ `cancel()` ถูกเรียก โดยไม่มี goroutine ตกค้าง — นี่คือจุดที่ทุกอย่างที่เรียนมาตลอด Part 041-044 มาบรรจบกัน: **fan-out/fan-in (042) + generics (028-029) + context cancellation (043) + ความมั่นใจว่าไม่มี data race เพราะผ่าน channel ล้วนๆ (044)**

ข้อดีของการทำ pipeline ให้เป็นทางการแบบนี้คือ **นำ stage กลับมาใช้ซ้ำข้ามโปรเจกต์ได้** — เขียน `Map`, `Filter` ครั้งเดียว ใช้ได้กับทุกชนิดข้อมูลตลอดไป ต่างจากการเขียน stage เฉพาะทางแบบ Part 042 ที่ต้อง copy โค้ดใหม่ทุกครั้ง

---

## 2. Pub-Sub Pattern: แนวคิดและความแตกต่างจาก Fan-out

**Publish-Subscribe (Pub-Sub)** เป็น pattern ที่ดูผิวเผินคล้าย fan-out/fan-in แต่มีเป้าหมายต่างกันโดยสิ้นเชิง:

| | Fan-out (Part 042) | Pub-Sub |
|---|---|---|
| เป้าหมาย | กระจาย**งาน**ให้ประมวลผลขนาน | กระจาย**ข้อความเดียวกัน**ให้ผู้รับหลายฝ่าย |
| ข้อความหนึ่งชิ้นถูกใครประมวลผล | **ผู้รับตัวใดตัวหนึ่งเท่านั้น** (แข่งกันแย่งงาน) | **ผู้รับ (subscriber) ทุกตัวที่ subscribe topic นั้น** ได้รับสำเนาเดียวกัน |
| ผู้ส่งรู้จักผู้รับไหม | รู้ (ส่งเข้า channel เดียวกันตรงๆ) | ไม่รู้ (ส่งผ่าน "topic" กลาง ไม่สนว่าใคร subscribe อยู่บ้าง) |
| ตัวอย่างการใช้งานจริง | ประมวลผลรูปภาพจำนวนมาก, Worker Pool | ระบบแจ้งเตือน, event bus ภายในแอป, WebSocket broadcast |

พูดง่ายๆ: fan-out คือ "ใครว่างก็รับงานไป" (งานหนึ่งชิ้นมีเจ้าของเดียว) ส่วน pub-sub คือ "ทุกคนที่สนใจเรื่องนี้ต้องได้รับข่าวเหมือนกันหมด" (ข้อความหนึ่งชิ้นมีผู้รับได้หลายคน)

---

## 3. ตัวอย่างสมบูรณ์: Pub-Sub Broker ด้วย Channel และ Mutex

หัวใจของการ implement pub-sub ด้วย channel คือ **Broker** (ตัวกลาง) ที่เก็บ **registry ของ subscriber channel แยกตาม topic** และป้องกัน registry นี้ด้วย mutex เพราะจะถูกเข้าถึงจากหลาย goroutine พร้อมกัน (คนที่ publish และคนที่ subscribe อาจเป็นคนละ goroutine)

```go
package main

import (
	"fmt"
	"sync"
)

// Broker คือ registry กลางที่เก็บ subscriber channel ทั้งหมดของแต่ละ topic
// ป้องกัน map ด้วย sync.RWMutex เพราะ Subscribe/Publish/Close เกิดขึ้นจากหลาย goroutine พร้อมกันได้
type Broker struct {
	mu   sync.RWMutex
	subs map[string][]chan string
}

func NewBroker() *Broker {
	return &Broker{subs: make(map[string][]chan string)}
}

// Subscribe คืน channel ใหม่สำหรับ topic ที่ระบุ ผู้เรียกมีหน้าที่ range อ่านค่าจาก channel นี้
func (b *Broker) Subscribe(topic string) <-chan string {
	b.mu.Lock()
	defer b.mu.Unlock()
	ch := make(chan string, 4) // buffer เล็กน้อยกันผู้ subscribe ช้าบล็อกผู้ publish ตรงๆ
	b.subs[topic] = append(b.subs[topic], ch)
	return ch
}

// Publish ส่งข้อความไปยังทุก subscriber ของ topic นั้น (ไม่บล็อกถ้า subscriber เต็ม buffer)
func (b *Broker) Publish(topic, msg string) {
	b.mu.RLock()
	defer b.mu.RUnlock()
	for _, ch := range b.subs[topic] {
		select {
		case ch <- msg:
		default:
			// subscriber ตามไม่ทัน (buffer เต็ม) เลือกทิ้งข้อความแทนที่จะบล็อก publisher
			fmt.Println("คำเตือน: subscriber ตามไม่ทัน ข้อความถูกทิ้ง:", msg)
		}
	}
}

// Close ปิด channel ของ subscriber ทุกตัวในทุก topic เพื่อให้ goroutine ที่ range อยู่จบการทำงาน
func (b *Broker) Close() {
	b.mu.Lock()
	defer b.mu.Unlock()
	for _, chans := range b.subs {
		for _, ch := range chans {
			close(ch)
		}
	}
	b.subs = make(map[string][]chan string)
}

func main() {
	broker := NewBroker()

	var wg sync.WaitGroup
	subscribe := func(name, topic string) {
		ch := broker.Subscribe(topic)
		wg.Add(1)
		go func() {
			defer wg.Done()
			for msg := range ch {
				fmt.Printf("[%s] ได้รับจาก topic %q: %s\n", name, topic, msg)
			}
			fmt.Printf("[%s] channel ถูกปิดแล้ว เลิกฟัง\n", name)
		}()
	}

	subscribe("subscriber-A", "news")
	subscribe("subscriber-B", "news")
	subscribe("subscriber-C", "sports")

	broker.Publish("news", "Go 1.24 released")
	broker.Publish("sports", "ทีมชนะไปแล้ว 3-0")
	broker.Publish("news", "แพ็กเกจใหม่ใน stdlib")

	broker.Close()
	wg.Wait()
}
```

รันด้วย `go run -race main.go` ผลลัพธ์ตัวอย่าง (ลำดับอาจต่างกันไปในแต่ละครั้งเพราะ goroutine ทำงานขนาน แต่เนื้อหาถูกต้องเสมอ):

```
[subscriber-C] ได้รับจาก topic "sports": ทีมชนะไปแล้ว 3-0
[subscriber-C] channel ถูกปิดแล้ว เลิกฟัง
[subscriber-B] ได้รับจาก topic "news": Go 1.24 released
[subscriber-A] ได้รับจาก topic "news": Go 1.24 released
[subscriber-B] ได้รับจาก topic "news": แพ็กเกจใหม่ใน stdlib
[subscriber-A] ได้รับจาก topic "news": แพ็กเกจใหม่ใน stdlib
[subscriber-B] channel ถูกปิดแล้ว เลิกฟัง
[subscriber-A] channel ถูกปิดแล้ว เลิกฟัง
```

สังเกตว่า **subscriber-A และ subscriber-B ทั้งคู่ได้รับข้อความ "news" ทั้งสองข้อความเหมือนกันทุกตัวอักษร** นี่คือความแตกต่างสำคัญจาก fan-out: ถ้าเป็น fan-out ข้อความ "Go 1.24 released" จะถูกส่งให้แค่ตัวใดตัวหนึ่งเท่านั้น (แข่งกันแย่ง) แต่ pub-sub ส่ง**สำเนา**ให้ทุก subscriber ของ topic นั้น

### ทำไมต้องใช้ `sync.RWMutex` แทน `sync.Mutex` ธรรมดา

`Publish` ถูกเรียกบ่อยกว่า `Subscribe` มากในระบบจริง (มีข้อความส่งเข้ามาต่อเนื่อง แต่ subscriber ใหม่เพิ่มเข้ามาไม่บ่อยเท่า) `sync.RWMutex` (ทบทวนจาก Part 039) อนุญาตให้หลาย goroutine เรียก `RLock()` เพื่อ**อ่าน** พร้อมกันได้ ตราบใดที่ไม่มีใครเรียก `Lock()` เพื่อ**เขียน**อยู่ ทำให้ `Publish` หลายครั้งพร้อมกันไม่ต้องรอคิวกันเอง (เพราะแค่อ่าน slice ของ subscriber ไม่ได้แก้ไข) ในขณะที่ `Subscribe`/`Close` ซึ่งแก้ไข map ต้องใช้ `Lock()` แบบ exclusive ตามปกติ

### ทำไม `Publish` ใช้ `select` + `default` แทนการส่งตรงๆ

```go
select {
case ch <- msg:
default:
	// ทิ้งข้อความแทนที่จะบล็อก publisher
}
```

ถ้า subscriber ตัวใดตัวหนึ่งอ่านข้อความช้ากว่าที่ publish เข้ามา (buffer ของ channel เต็ม) การส่งแบบ `ch <- msg` ตรงๆ จะ**บล็อก** `Publish` ทั้งฟังก์ชัน ทำให้ subscriber ตัวอื่นที่ทำงานปกติต้องรอไปด้วย (แม้จะไม่เกี่ยวข้องกัน) การใช้ `select` กับ `default` (รูปแบบที่เรียนไปใน Part 038) ทำให้ `Publish` **ไม่มีวันบล็อก** — ถ้า subscriber ตามไม่ทันก็แค่ทิ้งข้อความนั้นไป (best-effort delivery) ซึ่งเป็นทางเลือกที่เหมาะกับระบบแจ้งเตือนที่ยอมรับการทำข้อความหายได้บ้าง (ถ้าต้องการการันตีว่าข้อความไม่หายเลย ต้องออกแบบกลไกอื่นเพิ่ม เช่น per-subscriber queue ที่โตได้ หรือใช้ message queue จริงจังอย่าง Part 091-092)

---

## 4. Rate Limiting ด้วย `time.Ticker`

**Rate limiting** คือการจำกัด "อัตรา" ที่งานจะถูกทำ (เช่น ไม่เกิน N ครั้งต่อวินาที) ต่างจาก Worker Pool ที่จำกัด "จำนวนที่ทำพร้อมกัน" — rate limiting จำกัดที่ **ความถี่ตามเวลา** วิธีที่ง่ายที่สุดใช้ `time.Ticker` (ทบทวนจาก Part 022) ซึ่งส่งสัญญาณเข้า channel ทุกช่วงเวลาที่กำหนดไว้อย่างสม่ำเสมอ

```go
package main

import (
	"fmt"
	"time"
)

func main() {
	// time.Ticker ส่งสัญญาณเข้า channel ทุกช่วงเวลาที่กำหนด — ใช้จำกัดอัตราการทำงานอย่างง่าย
	// ตัวอย่างนี้: อนุญาตให้ทำงานได้ไม่เกิน 1 ครั้งทุก 100ms (~10 requests/sec)
	limiter := time.NewTicker(100 * time.Millisecond)
	defer limiter.Stop()

	requests := []string{"req-1", "req-2", "req-3", "req-4", "req-5"}

	start := time.Now()
	for _, req := range requests {
		<-limiter.C // บล็อกจนกว่าจะถึงรอบถัดไป
		fmt.Printf("[%v] ประมวลผล %s\n", time.Since(start).Round(time.Millisecond), req)
	}
}
```

ผลลัพธ์:

```
[100ms] ประมวลผล req-1
[201ms] ประมวลผล req-2
[300ms] ประมวลผล req-3
[401ms] ประมวลผล req-4
[500ms] ประมวลผล req-5
```

แต่ละ request ห่างกันประมาณ 100ms ตามที่ตั้งไว้ — วิธีนี้ง่ายและใช้ standard library ล้วนๆ แต่มีข้อจำกัด: **ไม่มีแนวคิดเรื่อง "burst"** (การอนุญาตให้ทำงานรวดเดียวหลายครั้งได้ในบางช่วง แล้วค่อยๆ ชดเชยคืน) ถ้าต้องการควบคุมที่ซับซ้อนกว่านี้ ควรใช้ package เฉพาะทาง

---

## 5. Rate Limiting ด้วย `golang.org/x/time/rate`

`golang.org/x/time/rate` เป็น package จากทีม Go เอง (นอก standard library แต่ดูแลอย่างเป็นทางการ) ที่ implement อัลกอริทึม **token bucket** ซึ่งเป็นมาตรฐานอุตสาหกรรมสำหรับ rate limiting

แนวคิด token bucket: มี "ถัง" ที่เติม token เข้าไปเรื่อยๆ ด้วยอัตราคงที่ (r token/วินาที) ถังมีความจุสูงสุด (b, burst size) ทุกครั้งที่จะทำงานหนึ่งครั้งต้องมี token อย่างน้อย 1 อันในถัง (ใช้แล้วหักออก) ถ้าถังว่างต้องรอจนกว่าจะมี token เติมเข้ามาใหม่ — ข้อดีของวิธีนี้คือ **อนุญาตให้ทำงาน "รัว" ได้ในช่วงสั้นๆ ถ้ามี token สะสมไว้ (burst)** โดยไม่ทำลายค่าเฉลี่ยระยะยาว

ติดตั้งด้วย:

```bash
go get golang.org/x/time/rate
```

```go
package main

import (
	"context"
	"fmt"
	"time"

	"golang.org/x/time/rate"
)

func main() {
	// rate.NewLimiter(r, b): r = อัตราเติม token ต่อวินาที, b = ขนาด burst (token สูงสุดที่สะสมได้)
	// ที่นี่: เติม 5 token/วินาที (เทียบเท่า 1 ครั้งทุก 200ms โดยเฉลี่ย) burst ได้สูงสุด 2 ครั้งรวด
	limiter := rate.NewLimiter(rate.Limit(5), 2)

	ctx := context.Background()
	start := time.Now()
	for i := 1; i <= 6; i++ {
		// Wait บล็อกจนกว่าจะมี token พอ หรือคืน error ถ้า ctx ถูกยกเลิกก่อน
		if err := limiter.Wait(ctx); err != nil {
			fmt.Println("ยกเลิก:", err)
			return
		}
		fmt.Printf("[%v] request %d ผ่าน rate limiter แล้ว\n", time.Since(start).Round(time.Millisecond), i)
	}
}
```

ผลลัพธ์:

```
[0s] request 1 ผ่าน rate limiter แล้ว
[0s] request 2 ผ่าน rate limiter แล้ว
[201ms] request 3 ผ่าน rate limiter แล้ว
[400ms] request 4 ผ่าน rate limiter แล้ว
[601ms] request 5 ผ่าน rate limiter แล้ว
[801ms] request 6 ผ่าน rate limiter แล้ว
```

สังเกตว่า **request 1 และ 2 ผ่านทันที** (เพราะถังเริ่มต้นมี token สะสมเต็ม 2 อันจาก burst) แต่ตั้งแต่ request 3 เป็นต้นไป ต้องรอประมาณ 200ms ต่อครั้ง (ตามอัตรา 5 token/วินาที = 1 token ทุก 200ms) นี่คือพฤติกรรมที่ `time.Ticker` แบบพื้นฐานทำไม่ได้ — token bucket ยืดหยุ่นกว่ามากสำหรับสถานการณ์จริง เช่น การเรียก API ภายนอกที่อนุญาตให้ "แหว่ง" ได้บ้างในบางช่วง (burst) ตราบใดที่อัตราเฉลี่ยระยะยาวไม่เกินที่กำหนด

ที่สำคัญ `limiter.Wait(ctx)` รับ `context.Context` เป็นพารามิเตอร์ (ตามธรรมเนียมที่เรียนใน Part 043) ทำให้ยกเลิกการรอได้ทันทีถ้า `ctx` ถูกยกเลิกหรือ timeout ระหว่างรอคิว — ไม่ต้องเขียนกลไก cancellation เอง

`golang.org/x/time/rate` ยังมีเมธอดอื่นที่มีประโยชน์: `Allow()` (เช็คทันทีโดยไม่รอ คืน `true`/`false`), `Reserve()` (จองสิทธิ์ล่วงหน้าแล้วดูว่าต้องรอนานแค่ไหน) ใช้เลือกได้ตามสถานการณ์ที่ต้องการ

---

## 6. Semaphore ผ่าน Buffered Channel

**Semaphore** คือกลไกจำกัดจำนวน "ผู้เข้าถึง resource พร้อมกัน" ไม่เกินค่าที่กำหนด — ฟังดูคล้าย Worker Pool แต่ต่างกันตรงที่ semaphore ไม่ได้ผูกกับรูปแบบ jobs/results channel ตายตัว สามารถใช้ล้อมรอบส่วนใดของโค้ดก็ได้ที่ต้องการจำกัดการเข้าถึงพร้อมกัน (เช่น จำกัดจำนวน HTTP request ที่ยิงออกไปพร้อมกัน, จำกัดจำนวนไฟล์ที่เปิดพร้อมกัน)

Go ไม่มี semaphore ใน standard library โดยตรง (ก่อน generics มี `golang.org/x/sync/semaphore` ให้ใช้) แต่ pattern ที่นิยมมากและใช้แค่ **buffered channel** ล้วนๆ ก็เพียงพอสำหรับกรณีส่วนใหญ่:

```go
package main

import (
	"fmt"
	"sync"
	"time"
)

// semaphore ใช้ buffered channel เป็นตัวจำกัดจำนวน goroutine ที่เข้าถึง resource
// พร้อมกันได้ ไม่เกินขนาด buffer — เป็น pattern ที่ใช้บ่อยมากเมื่อมี resource จำกัด
// เช่น DB connection pool, จำนวน request พร้อมกันไปยัง service ภายนอก
type semaphore chan struct{}

func newSemaphore(n int) semaphore {
	return make(semaphore, n)
}

func (s semaphore) acquire() { s <- struct{}{} }
func (s semaphore) release() { <-s }

func main() {
	const maxConcurrent = 3
	sem := newSemaphore(maxConcurrent)

	var wg sync.WaitGroup
	var mu sync.Mutex
	activeNow := 0
	maxObserved := 0

	for i := 1; i <= 10; i++ {
		wg.Add(1)
		go func(id int) {
			defer wg.Done()

			sem.acquire()
			defer sem.release()

			mu.Lock()
			activeNow++
			if activeNow > maxObserved {
				maxObserved = activeNow
			}
			snapshot := activeNow // อ่านค่าไว้ในตัวแปรท้องถิ่นขณะยัง lock อยู่ ป้องกัน race ตอน print
			mu.Unlock()

			fmt.Printf("goroutine %d: เข้าถึง resource แล้ว (active=%d)\n", id, snapshot)
			time.Sleep(50 * time.Millisecond) // จำลองการใช้ resource

			mu.Lock()
			activeNow--
			mu.Unlock()
		}(i)
	}

	wg.Wait()
	fmt.Println("จำนวน goroutine ที่เข้าถึง resource พร้อมกันสูงสุด:", maxObserved, "(ต้องไม่เกิน", maxConcurrent, ")")
}
```

รันด้วย `go run -race main.go` ผลลัพธ์:

```
จำนวน goroutine ที่เข้าถึง resource พร้อมกันสูงสุด: 3 (ต้องไม่เกิน 3 )
```

**หลักการทำงานของ semaphore แบบ channel**: `acquire()` คือ `s <- struct{}{}` — ถ้า channel ยังไม่เต็ม (มีที่ว่างใน buffer) จะส่งสำเร็จทันที แต่ถ้า buffer เต็มแล้ว (มี goroutine ครองสิทธิ์ครบ `maxConcurrent` ตัวแล้ว) การส่งจะ**บล็อก**จนกว่าจะมีใคร `release()` (ทำ `<-s` เพื่อดึงค่าออกจาก channel คืนพื้นที่ว่าง) นี่คือการใช้ความจุของ buffered channel เป็นตัวนับจำนวนสิทธิ์ที่เหลืออยู่โดยตรง — ไม่ต้องเขียนตัวนับหรือ mutex เพิ่มเติมสำหรับ logic การจำกัดจำนวนเอง (แม้ในตัวอย่างนี้เราจะเพิ่ม `mu`/`activeNow`/`maxObserved` แยกไว้ต่างหากเพียงเพื่อ**วัดผลและพิสูจน์**ว่า semaphore ทำงานถูกต้องเท่านั้น ไม่ใช่ส่วนที่จำเป็นต่อตัว semaphore เอง)

> **บันทึกจากการพัฒนาเนื้อหาบทนี้**: ตอนแรกโค้ดตัวอย่างนี้พิมพ์ `activeNow` ใน `fmt.Printf` โดยอ่านค่า**นอก** critical section (หลัง `mu.Unlock()` ไปแล้ว) ซึ่ง `go run -race` จับได้ทันทีว่าเป็น data race จริง (มี goroutine อื่นเขียน `activeNow` พร้อมกับตอนที่กำลังอ่านค่าไปพิมพ์) วิธีแก้คือก๊อปค่าลงตัวแปร local (`snapshot`) **ขณะที่ยังถือ lock อยู่** แล้วค่อยใช้ `snapshot` ตอน print นี่เป็นตัวอย่างจริงที่ตอกย้ำบทเรียนจาก Part 044: แม้โค้ดจะดู "ป้องกันด้วย mutex แล้ว" แต่การอ่านค่าแม้แต่ครั้งเดียวนอก critical section ก็เพียงพอที่จะทำให้เกิด data race — ต้องรัน `-race` ตรวจสอบเสมอ ไม่ใช่แค่ดูโค้ดด้วยตาแล้วเชื่อว่าปลอดภัย

---

## 7. ตารางสรุป: เลือก Concurrency Pattern ไหนเมื่อไร (Part 36-45)

ตลอดภาคที่ 3 เราเรียน pattern และเครื่องมือมามากมาย ตารางนี้สรุปภาพรวมทั้งหมดเพื่อช่วยตัดสินใจว่าจะหยิบอะไรมาใช้ในสถานการณ์ไหน:

| Pattern / เครื่องมือ | เรียนใน | ใช้เมื่อ | อย่าใช้เมื่อ |
|---|---|---|---|
| **Goroutine เดี่ยว** | Part 036 | งาน background ง่ายๆ ที่ไม่ต้องรอผลลัพธ์หรือ sync กับใคร | ต้องการควบคุมจำนวนหรือรอผลลัพธ์แบบมีโครงสร้าง |
| **Channel (unbuffered/buffered)** | Part 037 | ส่งข้อมูล/สัญญาณระหว่าง goroutine พร้อม sync ในตัว | ต้องการแค่ป้องกันตัวแปรร่วม (ใช้ mutex ตรงกว่า) |
| **select + default/timeout** | Part 038 | ต้องรอหลาย channel พร้อมกัน หรือไม่อยากบล็อกตลอดไป | มี channel เดียวและไม่ต้องมี timeout |
| **sync.Mutex / RWMutex** | Part 039 | ป้องกันตัวแปรหรือ struct ร่วมที่เข้าถึงจากหลาย goroutine | การสื่อสารข้อมูลระหว่าง goroutine (ใช้ channel แทน) |
| **sync.WaitGroup** | Part 039 | รอให้ goroutine กลุ่มหนึ่งทำงานเสร็จครบทุกตัว | ต้องการผลลัพธ์ (error) กลับมาด้วย (ใช้ errgroup แทน) |
| **sync.Once** | Part 039 | รับประกันว่าโค้ดบางส่วนทำงานแค่ครั้งเดียว (เช่น init) | ต้องทำงานซ้ำได้ตามเงื่อนไข |
| **sync/atomic** | Part 040 | นับค่า/flag ง่ายๆ ที่ต้องการความเร็วสูงกว่า mutex | logic ซับซ้อนกว่าการอ่าน/เขียนค่าเดี่ยว |
| **Worker Pool** | Part 041 | จำกัดจำนวน goroutine ที่ประมวลผลงานจำนวนมากพร้อมกัน | งานมีน้อยหรือไม่ต้องการจำกัดความขนาน |
| **errgroup** | Part 041 | Worker Pool ที่ต้องหยุดทันทีเมื่อเจอ error แรกและรวม error/cancellation | ไม่สนใจ error หรือแค่ต้องการรอให้จบ (ใช้ WaitGroup พอ) |
| **Fan-out / Fan-in** | Part 042 | แยกงานเป็นหลาย stage อิสระที่ปรับความขนานแยกกันได้ | งานขั้นตอนเดียวไม่ซับซ้อน หรือลำดับผลลัพธ์สำคัญมาก |
| **Generator pattern** | Part 042 | จุดเริ่มต้นมาตรฐานของทุก pipeline | — (ใช้แทบทุกครั้งที่เริ่ม pipeline) |
| **Done channel / ctx.Done()** | Part 042-043 | ป้องกัน goroutine leak เมื่อ pipeline ถูกยกเลิกกลางคัน | โค้ดที่รับประกันว่าทำงานจบเร็วมากอยู่แล้วโดยธรรมชาติ |
| **context.Context** | Part 032, 043 | ส่งสัญญาณยกเลิก/timeout ผ่าน call tree ทั้งหมด | เก็บ state ระยะยาวหรือส่ง parameter ทั่วไป |
| **go run/test -race** | Part 044 | ตรวจสอบทุกครั้งที่เขียนโค้ด concurrent โดยไม่มีข้อยกเว้น | build binary สำหรับ production (ช้าเกินไป) |
| **Generic Pipeline Stage** | Part 045 | ต้องการ stage ที่ใช้ซ้ำได้ข้ามชนิดข้อมูล/โปรเจกต์ | pipeline ง่ายๆ ที่ใช้ครั้งเดียวไม่คุ้มจะ generalize |
| **Pub-Sub** | Part 045 | กระจายข้อความเดียวกันให้ผู้รับหลายฝ่ายพร้อมกัน | ต้องการให้งานหนึ่งชิ้นมีผู้ประมวลผลแค่คนเดียว (ใช้ fan-out) |
| **Rate Limiting (Ticker / x/time/rate)** | Part 045 | จำกัดความถี่การเรียก resource ภายนอกตามเวลา | จำกัดจำนวนที่ทำงาน "พร้อมกัน" (ใช้ semaphore/Worker Pool) |
| **Semaphore (buffered channel)** | Part 045 | จำกัดจำนวนผู้เข้าถึง resource พร้อมกัน แบบยืดหยุ่นไม่ผูกกับ jobs/results | ต้องการโครงสร้าง job/result ที่ชัดเจน (ใช้ Worker Pool) |

---

## 8. จบภาคที่ 3: สิ่งที่ควรฝึกฝนต่อ

ภาคที่ 3 ครอบคลุมตั้งแต่พื้นฐานที่สุดของ concurrency ใน Go (goroutine, channel) ไปจนถึง pattern ระดับ production (worker pool, pipeline, pub-sub, rate limiting) และเครื่องมือตรวจสอบความถูกต้อง (race detector) ครบถ้วนแล้ว หัวใจที่ควรจำติดตัวไปตลอด:

1. **"Do not communicate by sharing memory; share memory by communicating"** (ปรัชญาจาก Part 001) — เลือกใช้ channel เพื่อสื่อสารข้อมูลเป็นอันดับแรก ใช้ mutex เมื่อแค่ต้องป้องกัน state ร่วมที่จำเป็นต้องเป็น shared memory จริงๆ
2. **ทุก goroutine ต้องมีทางออก** — ไม่ว่าจะผ่าน channel ที่ปิดแน่นอน หรือ `ctx.Done()` ไม่มีข้อยกเว้น
3. **`go run -race` และ `go test -race` คือเพื่อนซี้ตลอดกาล** ของการเขียนโค้ด concurrent ใน Go — ไม่มีความมั่นใจใดจะแทนที่การรันตรวจสอบจริงได้
4. **เลือก pattern ให้เหมาะกับปัญหา** ไม่ใช่เลือกเพราะมันซับซ้อนดูเท่ — ตารางในหัวข้อ 7 คือจุดเริ่มต้นที่ดีในการตัดสินใจ

ในภาคถัดไป **ภาคที่ 4: Standard Library เชิงลึก (Part 46-55)** เราจะออกจากโลกของ concurrency pattern ล้วนๆ ไปสำรวจ standard library ที่ใช้บ่อยที่สุดในงานจริง เริ่มจาก `net/http` ทั้งฝั่ง client และ server ซึ่งจะนำ concurrency pattern ที่เรียนมาทั้งภาคนี้ไปใช้งานจริงอีกครั้ง (เช่น HTTP server ที่ต้อง handle หลาย request พร้อมกัน แต่ละ request มี `context` ของตัวเองที่ผูกกับการยกเลิกเมื่อ client ตัดการเชื่อมต่อ — ตรงกับที่เรียนไปใน Part 043 เป๊ะๆ)

---

## สรุปสิ่งที่ได้เรียนในบทนี้

- Pipeline stage ทำให้เป็นทางการได้ด้วย Go generics (`Stage[In, Out any]`) ทำให้เขียน `Generate`, `Map`, `Filter` ครั้งเดียวใช้ซ้ำได้กับข้อมูลทุกชนิด
- **Pub-Sub** ต่างจาก fan-out ตรงที่ข้อความหนึ่งชิ้นถูกส่งให้ **ทุก subscriber** ของ topic นั้น ไม่ใช่แค่ตัวใดตัวหนึ่ง — implement ได้ด้วย registry ของ channel ที่ป้องกันด้วย `sync.RWMutex`
- `select` + `default` ใน `Publish` ทำให้ระบบ pub-sub ไม่บล็อกแม้ subscriber บางตัวตามไม่ทัน (best-effort delivery)
- **Rate limiting** จำกัดความถี่ตามเวลา ทำได้ง่ายๆ ด้วย `time.Ticker` หรือแบบยืดหยุ่นกว่าด้วย token bucket algorithm ผ่าน `golang.org/x/time/rate` ซึ่งรองรับ burst และรับ `context` เพื่อยกเลิกการรอได้
- **Semaphore** ผ่าน buffered channel ใช้ความจุของ channel เป็นตัวจำกัดจำนวนผู้เข้าถึง resource พร้อมกัน โดยไม่ต้องผูกกับโครงสร้าง job/result แบบ Worker Pool
- ตัวอย่าง semaphore เจอ data race จริงระหว่างพัฒนาเนื้อหา (อ่านค่านอก critical section) และแก้ด้วยการอ่านค่าไว้ในตัวแปร local ขณะยัง lock อยู่ — ยืนยันบทเรียนจาก Part 044 ว่าต้องรัน `-race` ตรวจสอบเสมอ
- ตารางสรุปในหัวข้อ 7 คือแผนที่รวบยอดของทุก pattern ตลอด Part 036-045 ใช้เป็นจุดอ้างอิงเวลาตัดสินใจเลือก pattern ในงานจริง
- ภาคที่ 3 จบแล้ว ก้าวต่อไปคือภาคที่ 4: Standard Library เชิงลึก เริ่มจาก `net/http`

## แบบฝึกหัดท้ายบท

1. เพิ่ม stage `Reduce[In, Out any]` เข้าไปในตัวอย่างหัวข้อ 1 ที่รวมค่าทั้งหมดจาก input channel เป็นค่าเดียว (ไม่ใช่ channel) แล้วใช้ปิดท้าย pipeline แทนการ `range` อ่านเอง
2. เพิ่มเมธอด `Unsubscribe` ให้กับ `Broker` ในหัวข้อ 3 ที่ลบ channel ของ subscriber ออกจาก registry (ต้องจัดการเรื่อง mutex ให้ถูกต้อง และคิดว่าจะปิด channel ของ subscriber ที่ unsubscribe ไปเมื่อไรจึงจะปลอดภัย)
3. เขียนโปรแกรมเปรียบเทียบ `time.Ticker` กับ `golang.org/x/time/rate` โดยจำลองสถานการณ์ที่ต้องการอนุญาต burst 5 request แรกทันที แล้วค่อยจำกัดเหลือ 2 request/วินาที — อธิบายว่าทำไม `time.Ticker` เพียงอย่างเดียวทำสิ่งนี้ไม่ได้
4. ดัดแปลง semaphore ในหัวข้อ 6 ให้รับ `context.Context` เพิ่มเติม เพื่อให้ `acquire()` ยกเลิกการรอได้ถ้า `ctx` ถูกยกเลิกก่อนที่จะมีสิทธิ์ว่าง (ใบ้: ต้องเปลี่ยนจากการส่งตรงๆ เป็น `select` ระหว่างส่งเข้า semaphore channel กับ `ctx.Done()`)
5. เขียนตารางเปรียบเทียบของตัวเอง (นอกเหนือจากหัวข้อ 7) โดยเลือก 3 สถานการณ์จากประสบการณ์การเขียนโปรแกรมที่ผ่านมา (หรือสถานการณ์สมมติ) แล้วระบุว่าจะเลือก pattern ไหนจาก Part 36-45 มาใช้ พร้อมเหตุผล
6. ทบทวนทั้งภาคที่ 3: เลือกตัวอย่างโค้ดจาก Part ใดก็ได้ใน 36-45 ที่คุณคิดว่ายากที่สุด แล้วเขียนอธิบายใหม่ด้วยคำพูดของตัวเอง (รวมถึงวาด diagram ประกอบถ้าช่วยให้เข้าใจง่ายขึ้น) เพื่อทดสอบว่าเข้าใจกลไกเบื้องหลังจริงหรือแค่จำโค้ดได้

---

**ต่อไป**: [Part 046 — `net/http` — HTTP Client เจาะลึก](./046-http-client.md)
