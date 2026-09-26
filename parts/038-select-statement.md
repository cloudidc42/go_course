# Part 038: คำสั่ง `select`

> ภาคที่ 3: การทำงานพร้อมกัน (Concurrency) — ตอนที่ 3 จาก 10 (Part 36–45)

## สารบัญของบทนี้

1. `select` คืออะไร: switch สำหรับ channel operation
2. Syntax พื้นฐานและการทำงาน
3. เมื่อหลาย case พร้อมพร้อมกัน: การเลือกแบบสุ่ม (pseudo-random)
4. `default` case: การทำงานแบบ non-blocking
5. `select` กับ `time.After`: การทำ Timeout
6. `select` กับ `context.Done()`: การทำ Cancellation (เชื่อมโยง Part 032)
7. `for { select { ... } }`: แพทเทิร์น Event Loop
8. Done Channel Pattern: การ broadcast ด้วย `close()`
9. สรุปสิ่งที่ได้เรียนในบทนี้
10. แบบฝึกหัดท้ายบท

---

## 1. `select` คืออะไร: switch สำหรับ channel operation

ใน Part 037 เราเรียนวิธีส่งและรับค่าผ่าน channel แบบตัวต่อตัวไปแล้ว (channel เดียว การกระทำเดียว) แต่ในโปรแกรม concurrent จริง เรามักต้องรอเหตุการณ์จาก**หลาย channel พร้อมกัน** เช่น "รอผลลัพธ์จาก worker หรือรอ timeout อันไหนมาถึงก่อนก็เอาอันนั้น" หรือ "คอยรับ event จากหลายแหล่งพร้อมกันแบบไม่รู้จบ"

นี่คือหน้าที่ของคำสั่ง **`select`** — เปรียบเทียบง่ายๆ คือ `select` ทำหน้าที่คล้าย `switch` (ที่เรียนใน Part 004) แต่แทนที่จะเทียบค่าตัวแปรกับ case ต่างๆ `select` จะ**รอให้ channel operation ใน case ใด case หนึ่งพร้อมทำงาน** แล้วเลือกรัน case นั้น

> **นิยามสั้นๆ**: `select` คือคำสั่งที่ทำให้ goroutine หนึ่งสามารถ **รอเหตุการณ์จากหลาย channel พร้อมกัน** และทำงานทันทีที่ channel ใด channel หนึ่ง "พร้อม" ก่อน

---

## 2. Syntax พื้นฐานและการทำงาน

```go
package main

import (
	"fmt"
	"time"
)

func main() {
	ch1 := make(chan string)
	ch2 := make(chan string)

	go func() {
		time.Sleep(100 * time.Millisecond)
		ch1 <- "จาก ch1"
	}()
	go func() {
		time.Sleep(50 * time.Millisecond)
		ch2 <- "จาก ch2"
	}()

	// select จะรอจนกว่า "อย่างน้อยหนึ่ง" case พร้อมทำงาน แล้วเลือก case นั้น
	for i := 0; i < 2; i++ {
		select {
		case msg1 := <-ch1:
			fmt.Println("ได้รับ:", msg1)
		case msg2 := <-ch2:
			fmt.Println("ได้รับ:", msg2)
		}
	}
}
```

ผลลัพธ์:

```
ได้รับ: จาก ch2
ได้รับ: จาก ch1
```

สังเกตว่า `ch2` มาถึงก่อนเพราะ goroutine ของมัน sleep แค่ 50ms (น้อยกว่า `ch1` ที่ sleep 100ms) ดังนั้นรอบแรกของ `select` จะได้ case ของ `ch2` ก่อน แล้วรอบที่สองถึงจะได้ `ch1`

หลักการทำงานของ `select` สรุปได้ดังนี้:

1. `select` จะประเมิน **ทุก channel operation ในทุก case พร้อมกัน** (ไม่ใช่ไล่ทีละ case จากบนลงล่างแบบ `switch` หรือ `if-else`)
2. ถ้ามี case ใด case หนึ่ง**พร้อมทำงานทันที** (channel มีค่าให้รับ หรือมีที่ว่างให้ส่ง) `select` จะรัน case นั้นทันที
3. ถ้า**ไม่มี case ไหนพร้อมเลย** `select` จะ **block** รอจนกว่าจะมี case ใด case หนึ่งพร้อม (คล้ายการรอ channel เดี่ยวๆ แต่ตอนนี้รอได้หลายทางเลือกพร้อมกัน)
4. ถ้ามี**หลาย case พร้อมพร้อมกันในเวลาเดียวกันพอดี** — นี่คือประเด็นที่น่าสนใจในหัวข้อถัดไป

---

## 3. เมื่อหลาย case พร้อมพร้อมกัน: การเลือกแบบสุ่ม (pseudo-random)

นี่คือพฤติกรรมที่มักสร้างความประหลาดใจให้มือใหม่: ถ้า**หลาย case พร้อมทำงานพร้อมกันในเวลาเดียวกันพอดี** `select` **จะไม่เลือก case แรกที่เขียนไว้เสมอ** (ต่างจาก `switch` ที่ประเมิน case ตามลำดับจากบนลงล่าง) แต่จะ **สุ่มเลือก (pseudo-random) หนึ่ง case จากบรรดา case ที่พร้อมทั้งหมดอย่างเท่าเทียมกัน**

สาธิตด้วยการทำให้ทั้งสอง channel มีค่าพร้อมอยู่เสมอ:

```go
package main

import "fmt"

func main() {
	ch1 := make(chan int, 1)
	ch2 := make(chan int, 1)
	ch1 <- 1
	ch2 <- 2

	countCh1 := 0
	countCh2 := 0

	// ทั้ง ch1 และ ch2 มีค่าพร้อมอยู่แล้วตั้งแต่ต้น (ready พร้อมกัน) ทุกครั้งที่ select ทำงาน
	// Go จะ "สุ่มเลือก" หนึ่ง case จากบรรดา case ที่พร้อมทั้งหมดอย่างเท่าเทียมกัน (ไม่ใช่เรียงจากบนลงล่างแบบ switch)
	// เติมค่ากลับเข้าเฉพาะ channel ที่เพิ่งถูกดึงออกไป เพื่อให้ทั้งคู่ "พร้อม" อยู่เสมอทุกรอบ
	for i := 0; i < 1000; i++ {
		select {
		case <-ch1:
			countCh1++
			ch1 <- 1
		case <-ch2:
			countCh2++
			ch2 <- 2
		}
	}

	fmt.Println("จำนวนครั้งที่เลือก ch1:", countCh1)
	fmt.Println("จำนวนครั้งที่เลือก ch2:", countCh2)
	fmt.Println("(ทั้งสองค่าควรใกล้เคียงกัน ~500 เพราะเลือกแบบสุ่มสม่ำเสมอ ไม่ใช่ ch1 ชนะทุกครั้งเพราะเขียนไว้ก่อน)")
}
```

ผลลัพธ์ (ตัวเลขจริงจะแตกต่างกันเล็กน้อยในแต่ละครั้งที่รัน เพราะเป็นการสุ่ม แต่จะใกล้เคียง 500/500 เสมอ):

```
จำนวนครั้งที่เลือก ch1: 510
จำนวนครั้งที่เลือก ch2: 490
(ทั้งสองค่าควรใกล้เคียงกัน ~500 เพราะเลือกแบบสุ่มสม่ำเสมอ ไม่ใช่ ch1 ชนะทุกครั้งเพราะเขียนไว้ก่อน)
```

ลองรันซ้ำหลายครั้งจะเห็นตัวเลขขยับไปมาเล็กน้อยรอบๆ 500 เสมอ ไม่ใช่ `1000/0` ซึ่งจะเกิดขึ้นถ้า `select` เลือกตามลำดับ case ที่เขียนไว้เหมือน `switch`

> **ทำไม Go ถึงออกแบบให้สุ่ม?** เหตุผลหลักคือเพื่อ**ป้องกัน starvation** — ถ้า `select` เลือก case แรกเสมอเมื่อพร้อมพร้อมกัน case ที่เขียนไว้ทีหลังอาจไม่มีวันถูกเลือกเลยถ้า case แรกพร้อมอยู่ตลอดเวลา (เช่น ในแพทเทิร์น event loop ที่มีหลายแหล่งข้อมูลเข้ามาตลอด) การสุ่มทำให้ทุก case มีโอกาสถูกเลือกอย่างเป็นธรรม (fair) ไม่มี case ไหนถูก "ทิ้ง" ไว้ตลอดกาล

---

## 4. `default` case: การทำงานแบบ non-blocking

ถ้าใส่ `default` case ไว้ใน `select` พฤติกรรมจะเปลี่ยนไปทันที: **ถ้าไม่มี case ไหนพร้อมทำงานเลย ณ ขณะนั้น `select` จะรัน `default` ทันทีโดยไม่ block รอ**

```go
package main

import "fmt"

func main() {
	ch := make(chan int)

	// ไม่มีใครส่งค่าเข้า ch เลย -- ถ้าไม่มี default, select จะ block ตลอดไป
	select {
	case v := <-ch:
		fmt.Println("ได้รับ:", v)
	default:
		fmt.Println("ไม่มีข้อมูลพร้อมใน channel ตอนนี้ -- ทำงานอื่นต่อแทนที่จะรอ")
	}

	ch2 := make(chan int, 1)
	ch2 <- 42
	select {
	case v := <-ch2:
		fmt.Println("ได้รับจาก ch2:", v)
	default:
		fmt.Println("จะไม่ถูกพิมพ์ เพราะ ch2 มีค่าพร้อมอยู่แล้ว")
	}
}
```

ผลลัพธ์:

```
ไม่มีข้อมูลพร้อมใน channel ตอนนี้ -- ทำงานอื่นต่อแทนที่จะรอ
ได้รับจาก ch2: 42
```

`select` ที่มี `default` จึงกลายเป็นเครื่องมือสำหรับทำ **non-blocking channel operation** — ตรวจสอบว่า channel มีข้อมูลพร้อมหรือไม่ **โดยไม่ต้องหยุดรอ** ถ้าไม่พร้อมก็ไปทำอย่างอื่นต่อได้ทันที เป็นประโยชน์มากในสถานการณ์เช่น: เช็คว่ามีงานใหม่เข้ามาไหมระหว่างที่ยังทำงานหลักอยู่ โดยไม่อยากให้ต้องหยุดรอ

> **ข้อควรระวัง**: การใช้ `default` พร่ำเพรื่อเพื่อ "polling" (เช็คซ้ำๆ ในลูปรัวๆ) แทนที่จะรอแบบ block เป็นเรื่องที่ไม่ควรทำโดยไม่จำเป็น เพราะจะทำให้ CPU ทำงานหนักเปล่าประโยชน์ (busy-wait) — ควรใช้เฉพาะเมื่อต้องการเช็คสถานะแบบ "ผ่านๆ" จริงๆ เท่านั้น ถ้าต้องการรอจริงจังให้ปล่อยให้ `select` block ตามปกติ (ไม่ใส่ `default`) จะมีประสิทธิภาพดีกว่ามาก เพราะ goroutine ที่ block อยู่ไม่กินเวลา CPU เลย

---

## 5. `select` กับ `time.After`: การทำ Timeout

หนึ่งในการใช้งาน `select` ที่พบบ่อยที่สุดในโค้ด production คือการทำ **timeout**: รอผลลัพธ์จากงานหนึ่ง แต่ถ้านานเกินไปก็ยกเลิกแล้วไปทำอย่างอื่นแทน

`time.After(d)` คืนค่าเป็น channel ที่จะได้รับค่า (เวลาปัจจุบัน) **หลังจากเวลา `d` ผ่านไป** — ใช้เป็น case คู่กับงานที่เรากำลังรออยู่ได้พอดี:

```go
package main

import (
	"fmt"
	"time"
)

func slowOperation() <-chan string {
	out := make(chan string)
	go func() {
		time.Sleep(300 * time.Millisecond) // จำลองงานที่ใช้เวลานาน
		out <- "ผลลัพธ์จากงานที่ช้า"
	}()
	return out
}

func main() {
	select {
	case result := <-slowOperation():
		fmt.Println("สำเร็จ:", result)
	case <-time.After(100 * time.Millisecond):
		fmt.Println("timeout: งานใช้เวลานานเกินไป (รอเกิน 100ms)")
	}
}
```

ผลลัพธ์:

```
timeout: งานใช้เวลานานเกินไป (รอเกิน 100ms)
```

เพราะ `slowOperation()` ใช้เวลาถึง 300ms แต่เรากำหนด timeout ไว้แค่ 100ms `select` จึงเลือก case `time.After` ก่อนเสมอ (ในตัวอย่างนี้) — ถ้าสลับตัวเลขให้ `slowOperation` เร็วกว่า timeout ผลลัพธ์ก็จะกลับกัน

> **ข้อควรระวังเรื่อง resource**: `time.After` สร้าง `time.Timer` ตัวใหม่ทุกครั้งที่ถูกเรียก และ timer นั้นจะไม่ถูกเก็บกวาด (garbage collect) จนกว่าจะครบเวลาที่กำหนดจริงๆ ถ้าเรียก `time.After` ซ้ำๆ ในลูปที่ทำงานถี่มาก (เช่น ใน event loop ที่จะเห็นในหัวข้อ 7) อาจทำให้มี timer ค้างอยู่ในหน่วยความจำจำนวนมากโดยไม่จำเป็น ในโค้ด production ที่ใช้ `select` วนซ้ำบ่อยๆ ควรพิจารณาใช้ `time.NewTimer` ร่วมกับ `defer timer.Stop()` แทน เพื่อควบคุมการคืนทรัพยากรอย่างชัดเจน

---

## 6. `select` กับ `context.Done()`: การทำ Cancellation (เชื่อมโยง Part 032)

ใน **Part 032** เราเรียนแพ็กเกจ `context` ไปแล้วว่ามันใช้ส่งสัญญาณ **cancellation** และ **deadline** ข้ามผ่านหลายชั้นของฟังก์ชันได้ — ตอนนี้ถึงเวลาเห็นว่ามันทำงานร่วมกับ `select` อย่างไรในทางปฏิบัติ

`context.Context` มี method `Done()` ที่คืนค่าเป็น `<-chan struct{}` — channel นี้จะถูก **ปิด** เมื่อ context ถูกยกเลิก (ไม่ว่าจะเพราะเรียก `cancel()` ตรงๆ, ครบ timeout, หรือครบ deadline) เมื่อ channel ถูกปิด การรับค่าจากมันจะสำเร็จทันที (ตามหลักการหัวข้อ 7 ของ Part 037) ทำให้ `select` ตรวจจับการยกเลิกได้ทันที:

```go
package main

import (
	"context"
	"fmt"
	"time"
)

func worker(ctx context.Context, id int) {
	for {
		select {
		case <-ctx.Done():
			fmt.Printf("worker %d: ถูกยกเลิกแล้ว เหตุผล: %v\n", id, ctx.Err())
			return
		case <-time.After(50 * time.Millisecond):
			fmt.Printf("worker %d: ทำงานอยู่...\n", id)
		}
	}
}

func main() {
	ctx, cancel := context.WithTimeout(context.Background(), 170*time.Millisecond)
	defer cancel()

	worker(ctx, 1)
	fmt.Println("main: worker หยุดทำงานแล้ว โปรแกรมจบ")
}
```

ผลลัพธ์:

```
worker 1: ทำงานอยู่...
worker 1: ทำงานอยู่...
worker 1: ทำงานอยู่...
worker 1: ถูกยกเลิกแล้ว เหตุผล: context deadline exceeded
main: worker หยุดทำงานแล้ว โปรแกรมจบ
```

`worker` ทำงานทุก 50ms แต่ context ถูกกำหนด timeout ไว้ 170ms พอครบเวลา `ctx.Done()` channel จะถูกปิด ทำให้ case `<-ctx.Done()` พร้อมทำงานทันทีในรอบ `select` ถัดไป — `worker` จึงหยุดทำงานอย่างสุภาพ (graceful) แทนที่จะถูกฆ่าทิ้งกลางคัน และ `ctx.Err()` บอกเหตุผลของการยกเลิกให้ทราบด้วย (`context deadline exceeded` ในกรณีนี้ หรือ `context canceled` ถ้าเรียก `cancel()` ตรงๆ)

นี่คือรูปแบบมาตรฐานที่สุดในการเขียน goroutine ที่ **ยกเลิกได้ (cancellable)** ในโค้ด Go — คุณจะเห็นแพทเทิร์น `select { case <-ctx.Done(): ...; case <-otherChan: ... }` ซ้ำแล้วซ้ำเล่าในโค้ด production ระดับ enterprise แทบทุกที่ที่มีการทำงานพร้อมกัน เราจะเจาะลึกการผสาน `context` เข้ากับ concurrency แบบเต็มรูปแบบอีกครั้งใน **Part 043**

---

## 7. `for { select { ... } }`: แพทเทิร์น Event Loop

การรวม `for` แบบไม่มีเงื่อนไข (infinite loop ตามที่เรียนใน Part 005) เข้ากับ `select` จะได้แพทเทิร์นที่ใช้บ่อยมากเรียกว่า **event loop** — goroutine ที่ทำหน้าที่คอยรับและจัดการ "เหตุการณ์" จากหลายแหล่งไม่มีที่สิ้นสุด จนกว่าจะได้รับสัญญาณให้หยุด:

```go
package main

import (
	"fmt"
	"time"
)

func main() {
	events := make(chan string)
	ticker := time.NewTicker(30 * time.Millisecond)
	defer ticker.Stop()
	done := make(chan struct{})

	go func() {
		time.Sleep(20 * time.Millisecond)
		events <- "event: login"
		time.Sleep(30 * time.Millisecond)
		events <- "event: purchase"
		time.Sleep(50 * time.Millisecond)
		close(done)
	}()

	tickCount := 0
	// รูปแบบ "event loop": for { select { ... } } วนรับ event จากหลายแหล่งไม่มีที่สิ้นสุดจนกว่าจะสั่งหยุด
	for {
		select {
		case ev := <-events:
			fmt.Println("จัดการ:", ev)
		case t := <-ticker.C:
			tickCount++
			_ = t
		case <-done:
			fmt.Println("event loop: ได้รับสัญญาณ done, tick ทั้งหมด >=", tickCount >= 1)
			return
		}
	}
}
```

ผลลัพธ์:

```
จัดการ: event: login
จัดการ: event: purchase
event loop: ได้รับสัญญาณ done, tick ทั้งหมด >= true
```

แพทเทิร์น `for { select { ... } }` นี้คือหัวใจของระบบจำนวนมากที่ต้องคอยรับข้อมูลจากหลายแหล่งพร้อมกันตลอดเวลา เช่น server ที่รับ connection ใหม่, event สั่งงาน, และสัญญาณ shutdown พร้อมกัน โครงสร้างนี้มีองค์ประกอบสำคัญ 3 ส่วนเสมอ:

1. **Event source ต่างๆ** — channel ที่ส่งข้อมูลจริงเข้ามา (`events`, `ticker.C` ในตัวอย่างนี้)
2. **สัญญาณหยุด (stop signal)** — มักเป็น `done` channel หรือ `ctx.Done()` ตามหัวข้อ 6 เพื่อให้ loop จบได้เมื่อถึงเวลา
3. **`return` หรือ `break`** ใน case ของสัญญาณหยุด เพื่อออกจาก infinite loop อย่างถูกต้อง (ถ้าลืมใส่ `return`/`break` loop จะวนต่อไปเรื่อยๆ แม้ได้รับสัญญาณหยุดแล้วก็ตาม)

---

## 8. Done Channel Pattern: การ broadcast ด้วย `close()`

หัวข้อสุดท้ายของบทนี้คือเทคนิคที่ทรงพลังมาก และอาศัยพฤติกรรมของ channel ที่ปิดแล้วซึ่งเรียนไปใน Part 037 หัวข้อ 7 โดยตรง: **การใช้ `close()` เพื่อส่งสัญญาณไปยังหลาย goroutine พร้อมกันในครั้งเดียว**

ทบทวนก่อนว่า: การรับค่าจาก channel ที่ปิดแล้วจะ**สำเร็จทันทีเสมอ** (ได้ zero value พร้อม `ok == false`) ไม่ว่าจะมี goroutine กี่ตัวมารับพร้อมกันก็ตาม — คุณสมบัตินี้ต่างจากการ**ส่งค่า**ปกติเข้า channel ซึ่งค่าหนึ่งค่าจะถูกรับโดย goroutine เพียงตัวเดียวเท่านั้น (ตามที่เห็นในตัวอย่าง producer/consumer ของ Part 037) การปิด channel จึงกลายเป็นกลไก **broadcast** ตามธรรมชาติ: ปิดครั้งเดียว แต่ **ทุก** goroutine ที่กำลัง `select` รอ channel นั้นอยู่จะได้รับสัญญาณพร้อมกันทั้งหมด

```go
package main

import (
	"fmt"
	"sync"
	"time"
)

// worker ทำงานซ้ำไปเรื่อยๆ จนกว่า done channel จะถูกปิด
func worker(id int, done <-chan struct{}, wg *sync.WaitGroup) {
	defer wg.Done()
	for {
		select {
		case <-done: // การ close(done) จะทำให้ทุก goroutine ที่รออยู่ "ตื่น" พร้อมกันทั้งหมด
			fmt.Printf("worker %d: ได้รับสัญญาณหยุด\n", id)
			return
		default:
			// ทำงานปกติสั้นๆ แล้ววนมาเช็คสัญญาณใหม่
			time.Sleep(10 * time.Millisecond)
		}
	}
}

func main() {
	done := make(chan struct{})
	var wg sync.WaitGroup

	const numWorkers = 4
	wg.Add(numWorkers)
	for i := 1; i <= numWorkers; i++ {
		go worker(i, done, &wg)
	}

	time.Sleep(50 * time.Millisecond)
	fmt.Println("main: สั่งหยุด worker ทั้งหมดพร้อมกันด้วย close(done)")
	close(done) // broadcast: ปิด channel ครั้งเดียว แต่ทุก goroutine ที่ select บน done จะรับรู้พร้อมกัน

	wg.Wait()
	fmt.Println("main: worker ทุกตัวหยุดแล้ว")
}
```

ผลลัพธ์ (ลำดับ worker ที่หยุดอาจสลับกันไปในแต่ละครั้ง แต่ครบทั้ง 4 ตัวเสมอ):

```
main: สั่งหยุด worker ทั้งหมดพร้อมกันด้วย close(done)
worker 1: ได้รับสัญญาณหยุด
worker 2: ได้รับสัญญาณหยุด
worker 3: ได้รับสัญญาณหยุด
worker 4: ได้รับสัญญาณหยุด
main: worker ทุกตัวหยุดแล้ว
```

จุดสำคัญที่ทำให้แพทเทิร์นนี้ต่างจากการใช้ channel ส่งค่าปกติ: ถ้า `main` พยายามส่งค่าจริงๆ เข้า `done` (เช่น `done <- struct{}{}`) แทนที่จะ `close(done)` จะมีแค่ **worker เดียวเท่านั้น** ที่ได้รับสัญญาณ (เพราะค่าที่ส่งเข้า channel ถูกรับได้แค่ครั้งเดียวโดยผู้รับตัวใดตัวหนึ่ง) ต้องส่งซ้ำ 4 ครั้งพอดีเพื่อให้ worker ทั้ง 4 ตัวได้รับครบ ซึ่งเปราะบางมาก (ถ้าจำนวน worker เปลี่ยนไปโดยไม่รู้ตัวก็ผิดทันที) การ `close()` เพียงครั้งเดียวจึงเป็นวิธีที่ปลอดภัยและ scale ได้ดีกว่ามากในการสั่งหยุด goroutine จำนวนเท่าใดก็ได้พร้อมกัน

> **หมายเหตุ**: การใช้ channel type `chan struct{}` (แทนที่จะเป็น `chan bool` หรือ `chan int`) เป็นธรรมเนียมปฏิบัติมาตรฐานสำหรับ "สัญญาณ" ที่ไม่มีข้อมูลอะไรจะส่งจริงๆ นอกจากตัว "เหตุการณ์" เอง เพราะ `struct{}` เป็น type ที่ **ไม่กินพื้นที่หน่วยความจำเลย** (zero-size type) สื่อเจตนาได้ชัดเจนกว่าการใช้ `bool` ซึ่งอาจตีความผิดว่ามีความหมายของค่า `true`/`false` แฝงอยู่ ทั้งที่จริงๆ แล้วสิ่งที่สำคัญคือ "การมาถึงของสัญญาณ" เท่านั้น ไม่ใช่ค่าที่ส่งมา — และนี่ก็คือกลไกเดียวกันเป๊ะกับที่ `context.Done()` ใช้ภายใน (คืนค่าเป็น `<-chan struct{}`)

---

## สรุปสิ่งที่ได้เรียนในบทนี้

- **`select`** ทำหน้าที่คล้าย `switch` แต่สำหรับ channel operation — รอหลาย channel พร้อมกัน แล้วรัน case ของ channel ที่พร้อมก่อน
- เมื่อ**หลาย case พร้อมพร้อมกัน** Go จะ**สุ่มเลือก**อย่างเท่าเทียม ไม่ใช่เลือกตามลำดับที่เขียนไว้ — ออกแบบมาเพื่อป้องกัน starvation
- ใส่ **`default`** เพื่อทำ `select` แบบ **non-blocking** — ถ้าไม่มี case ไหนพร้อม จะรัน `default` ทันทีแทนที่จะรอ
- **`time.After(d)`** ใช้คู่กับ `select` เพื่อทำ **timeout** ได้อย่างกระชับ
- **`ctx.Done()`** (จาก Part 032) ใช้คู่กับ `select` เพื่อทำ **cancellation** ที่สุภาพ (graceful) — ตรวจสอบเหตุผลด้วย `ctx.Err()`
- **`for { select { ... } }`** คือแพทเทิร์น **event loop** มาตรฐาน สำหรับรับเหตุการณ์จากหลายแหล่งไม่มีที่สิ้นสุดจนกว่าจะได้รับสัญญาณหยุด
- การ **`close()`** channel เป็นกลไก **broadcast** ตามธรรมชาติ — ปิดครั้งเดียว ทุก goroutine ที่ `select` รอ channel นั้นอยู่จะได้รับสัญญาณพร้อมกันทั้งหมด ต่างจากการส่งค่าปกติที่มีแค่ผู้รับตัวเดียวเท่านั้นที่ได้รับ
- ธรรมเนียมปฏิบัติ: ใช้ `chan struct{}` สำหรับ channel ที่ทำหน้าที่เป็น "สัญญาณ" ล้วนๆ ไม่มีข้อมูลจริงจะส่ง

## แบบฝึกหัดท้ายบท

1. เขียนโปรแกรมที่มี 3 channel ให้ goroutine 3 ตัวส่งค่าเข้าแต่ละ channel ด้วยเวลาสุ่มต่างกัน (`time.Sleep` เวลาสุ่มโดยใช้แพ็กเกจ `math/rand`) แล้วใช้ `select` ใน `main` เพื่อรับค่าและพิมพ์ทันทีที่มาถึง จนกว่าจะได้ครบทั้ง 3 ค่า
2. ทดลองรันตัวอย่างในหัวข้อ 3 (pseudo-random) ซ้ำ 5 ครั้ง บันทึกตัวเลขที่ได้แต่ละครั้ง แล้วอธิบายว่าทำไมตัวเลขไม่เท่ากันเป๊ะทุกครั้งแต่ก็ไม่เอียงไปทาง case ใดทางหนึ่งอย่างสม่ำเสมอ
3. เขียนฟังก์ชันที่รับ channel และ timeout duration เป็น parameter แล้วคืนค่า `(value int, ok bool)` โดยใช้ `select` กับ `time.After` — ถ้าได้ค่าก่อน timeout ให้คืน `ok = true`, ถ้า timeout ก่อนให้คืน `ok = false`
4. ดัดแปลงตัวอย่างในหัวข้อ 6 (context.Done) ให้เปลี่ยนจาก `context.WithTimeout` เป็น `context.WithCancel` แล้วให้ `main` เรียก `cancel()` เองหลัง sleep ไประยะหนึ่ง สังเกตว่า `ctx.Err()` เปลี่ยนข้อความไปเป็นอะไร (เทียบกับตอนใช้ `WithTimeout`)
5. เขียน event loop ของตัวเองที่รับ event จาก 2 channel (`orders` กับ `cancellations`) พร้อมกับมี `done` channel สำหรับหยุดการทำงาน ทดสอบว่าเมื่อ `close(done)` แล้ว event loop หยุดทำงานทันทีโดยไม่ค้างรอ event ที่เหลือ
6. ค้นคว้าเพิ่มเติม: หาข้อมูลเรื่อง `time.NewTimer` กับ `timer.Stop()` ว่าทำไมจึงแนะนำให้ใช้แทน `time.After` ในกรณีที่ `select` ถูกเรียกซ้ำในลูปที่ทำงานถี่ (เช่น event loop ในหัวข้อ 7) ลองเขียนโค้ดเปรียบเทียบทั้งสองแบบดูด้วยตัวเอง

---

**ต่อไป**: [Part 039 — แพ็กเกจ `sync`: Mutex, WaitGroup, Once](./039-sync-package.md)
