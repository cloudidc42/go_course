# Part 039: แพ็กเกจ `sync`: Mutex, WaitGroup, Once

> ภาคที่ 3: การทำงานพร้อมกัน (Concurrency) — ตอนที่ 4 จาก 10 (Part 36–45)

## สารบัญของบทนี้

1. เมื่อไรควรใช้ `sync` แทน channel
2. Data Race คืออะไร: สาธิตด้วย Race Detector
3. `sync.Mutex`: การป้องกัน Critical Section
4. ทำไมต้อง `defer mu.Unlock()` ทันทีหลัง `Lock()`
5. `sync.RWMutex`: แยก Read Lock และ Write Lock
6. `sync.WaitGroup`: รอกลุ่ม goroutine ให้ทำงานเสร็จ
7. ข้อผิดพลาดคลาสสิก: เรียก `Add` ข้างในตัว goroutine
8. `sync.Once`: การทำ Lazy Initialization
9. `sync.Map`: เมื่อไรถึงควรใช้จริงๆ
10. สรุปสิ่งที่ได้เรียนในบทนี้
11. แบบฝึกหัดท้ายบท

---

## 1. เมื่อไรควรใช้ `sync` แทน channel

ตลอด Part 036-038 เราเน้นย้ำปรัชญาของ Go ที่ว่า "share memory by communicating" ผ่าน channel มาโดยตลอด แต่ Go **ไม่ได้บังคับ**ให้ใช้ channel เสมอไป — แพ็กเกจมาตรฐาน **`sync`** มอบเครื่องมือสำหรับการซิงโครไนซ์แบบ **shared memory** (แชร์หน่วยความจำร่วมกัน + ป้องกันด้วย lock) ซึ่งเป็นแนวทางดั้งเดิมที่ภาษาโปรแกรมมิ่งส่วนใหญ่ใช้กัน

คำถามที่พบบ่อยคือ "แล้วเมื่อไรควรใช้ channel เมื่อไรควรใช้ `sync.Mutex`?" Rob Pike เคยให้แนวทางคร่าวๆ ไว้ว่า:

> "Channels orchestrate; mutexes serialize." (Channel ใช้จัดลำดับ/ประสานงานการไหลของข้อมูล ส่วน mutex ใช้ทำให้การเข้าถึงข้อมูลเป็นไปทีละคน)

แนวทางปฏิบัติที่ใช้ได้จริงคือ:

- ใช้ **channel** เมื่อต้องการ**ส่งต่อความเป็นเจ้าของข้อมูล**ระหว่าง goroutine หรือประสานจังหวะการทำงาน (เช่น producer/consumer, pipeline, การรอสัญญาณ) — สิ่งที่เราเรียนมาใน Part 037-038
- ใช้ **`sync.Mutex`** เมื่อมี**ข้อมูลสถานะ (state) ชิ้นเดียว** ที่หลาย goroutine ต้อง**อ่าน/เขียนร่วมกัน** และไม่มีเหตุผลต้องส่งต่อความเป็นเจ้าของไปมา (เช่น ตัวนับ, cache ในหน่วยความจำ, ค่า config ที่แก้ไขได้)

ทั้งสองแนวทางถูกต้องและใช้ร่วมกันได้ในโปรแกรมเดียวกัน ไม่มีกฎตายตัวว่าต้องเลือกอย่างใดอย่างหนึ่งเสมอ — Go เพียงแค่**เสนอ**ทางเลือก channel เป็นแนวทางที่มักทำให้โค้ดอ่านง่ายกว่าในหลายสถานการณ์ ไม่ได้ห้ามใช้ mutex

---

## 2. Data Race คืออะไร: สาธิตด้วย Race Detector

ก่อนเรียนวิธีใช้ `sync.Mutex` มาดูปัญหาที่มันแก้ให้เห็นภาพชัดเจนก่อน: **Data Race**

> **Data Race** เกิดขึ้นเมื่อมีอย่างน้อย 2 goroutine เข้าถึงตัวแปรตัวเดียวกัน**พร้อมกัน** โดยที่**อย่างน้อยหนึ่งฝั่งเป็นการเขียน** และไม่มีกลไก synchronization ใดๆ มาควบคุมลำดับการเข้าถึง

```go
package main

import (
	"fmt"
	"sync"
)

func main() {
	counter := 0
	var wg sync.WaitGroup

	for i := 0; i < 1000; i++ {
		wg.Add(1)
		go func() {
			defer wg.Done()
			counter++ // อ่าน-บวก-เขียน: ไม่ใช่ atomic operation! หลาย goroutine เข้าถึงพร้อมกันโดยไม่มีการป้องกัน
		}()
	}

	wg.Wait()
	fmt.Println("counter:", counter) // มักจะได้ค่าน้อยกว่า 1000 เพราะเกิด race
}
```

รันแบบปกติ (ไม่ใช้ race detector) หลายครั้งจะเห็นค่าที่ไม่แน่นอนและมักน้อยกว่า 1000:

```
counter: 961
counter: 991
counter: 980
```

เหตุผลคือ `counter++` ไม่ใช่การกระทำเดียว (atomic) แต่จริงๆ แล้วประกอบด้วย 3 ขั้นตอนภายใน: **(1) อ่านค่าปัจจุบันของ `counter`, (2) บวกเพิ่ม 1, (3) เขียนค่าใหม่กลับไป** ถ้าสอง goroutine ทำสามขั้นตอนนี้สอดแทรกกัน (interleave) เช่น ทั้งคู่อ่านค่า `500` พร้อมกัน แล้วต่างคนต่างบวกเป็น `501` แล้วเขียนทับ ผลคือค่าที่ควรจะเพิ่มขึ้น 2 ครั้งกลับเพิ่มขึ้นแค่ 1 ครั้ง — การเพิ่มค่าบางครั้ง "หายไป" แบบนี้เองที่ทำให้ผลรวมสุดท้ายน้อยกว่า 1000

Go มีเครื่องมือมาตรฐานในการตรวจจับปัญหานี้**โดยอัตโนมัติ** เรียกว่า **Race Detector** เปิดใช้งานง่ายๆ ด้วย flag `-race`:

```bash
go run -race counter.go
```

ผลลัพธ์:

```
==================
WARNING: DATA RACE
Read at 0x00c000114078 by goroutine 8:
  main.main.func1()
      /path/to/counter.go:16 +0x84

Previous write at 0x00c000114078 by goroutine 7:
  main.main.func1()
      /path/to/counter.go:16 +0x96

Goroutine 8 (running) created at:
  main.main()
      /path/to/counter.go:14 +0x78

Goroutine 7 (finished) created at:
  main.main()
      /path/to/counter.go:14 +0x78
==================
```

Race Detector บอกได้ชัดเจนมากว่า **บรรทัดไหน** ที่เกิดการอ่าน/เขียนชนกัน และ **goroutine ไหน** ที่เกี่ยวข้อง — เป็นเครื่องมือที่ทรงพลังมากในการหา bug ประเภทนี้ ซึ่งปกติแล้วยากมากที่จะหาด้วยการอ่านโค้ดเฉยๆ หรือแม้แต่ debug ด้วยตาเปล่า เพราะ data race อาจไม่แสดงอาการทุกครั้งที่รัน (เป็น bug แบบ "flaky" ที่ขึ้นอยู่กับจังหวะเวลาของ scheduler)

> **คำแนะนำสำคัญ**: **ควรรัน test suite ด้วย `go test -race ./...` เป็นประจำ** โดยเฉพาะโปรเจกต์ที่มีการใช้ goroutine เยอะ เพราะ data race ที่ตรวจไม่พบตอน develop อาจไปโผล่เป็นบั๊กประหลาดที่วินิจฉัยยากมากใน production เราจะเจาะลึกเรื่อง race detector และการวิเคราะห์ผลลัพธ์แบบเต็มรูปแบบใน **Part 044: Race Condition และ Race Detector**

---

## 3. `sync.Mutex`: การป้องกัน Critical Section

**`sync.Mutex`** (mutual exclusion) คือกลไกล็อกพื้นฐานที่สุด: มันรับประกันว่า**มีแค่ goroutine เดียวเท่านั้น**ที่สามารถอยู่ใน "ส่วนวิกฤต" (**critical section** — ช่วงโค้ดที่เข้าถึงข้อมูลร่วมกัน) ได้ในขณะใดขณะหนึ่ง

Mutex มี 2 method หลัก: `Lock()` และ `Unlock()` มาแก้ตัวอย่างก่อนหน้าด้วย mutex กัน:

```go
package main

import (
	"fmt"
	"sync"
)

func main() {
	counter := 0
	var mu sync.Mutex
	var wg sync.WaitGroup

	for i := 0; i < 1000; i++ {
		wg.Add(1)
		go func() {
			defer wg.Done()
			mu.Lock()
			defer mu.Unlock() // defer ทันทีหลัง Lock() -- การันตีว่า Unlock ถูกเรียกเสมอแม้เกิด panic ระหว่างทาง
			counter++
		}()
	}

	wg.Wait()
	fmt.Println("counter:", counter) // ได้ 1000 เสมอ เพราะ mutex ป้องกันไม่ให้สอง goroutine เข้า critical section พร้อมกัน
}
```

ผลลัพธ์ (รันด้วย `go run -race` ก็ผ่านสะอาด ไม่มี warning ใดๆ เลย):

```
counter: 1000
counter: 1000
counter: 1000
```

กลไกการทำงาน: `mu.Lock()` จะ**ล็อก** mutex ทันที ถ้ามี goroutine อื่นถือ lock อยู่ก่อนแล้ว การเรียก `Lock()` จะ**block รอ**จนกว่า goroutine นั้นจะ `Unlock()` ก่อน — ผลคือมีเพียง goroutine เดียวเท่านั้นที่ผ่านเข้าไปทำ `counter++` ได้ในแต่ละขณะ ทำให้ปัญหา interleaving ที่เห็นในหัวข้อ 2 หายไปโดยสิ้นเชิง เพราะไม่มีทางที่สอง goroutine จะอ่าน-เขียน `counter` พร้อมกันได้อีกต่อไป

> **หมายเหตุ**: `sync.Mutex` ต้อง**ไม่ถูกคัดลอก (copy)** หลังจากถูกใช้งานแล้ว — ต้องส่งต่อผ่าน pointer เสมอถ้าจำเป็นต้องส่งให้ฟังก์ชันอื่น (เช่น `func f(mu *sync.Mutex)` ไม่ใช่ `func f(mu sync.Mutex)`) เพราะการคัดลอก mutex ที่กำลังถูกใช้งานจะทำให้กลไก lock เพี้ยนไปโดยสิ้นเชิง — `go vet` (ที่เรียนใน Part 001) จะช่วยเตือนเรื่องนี้ให้อัตโนมัติถ้าตรวจพบ

---

## 4. ทำไมต้อง `defer mu.Unlock()` ทันทีหลัง `Lock()`

สังเกตในตัวอย่างข้างต้นว่าเราเขียน `defer mu.Unlock()` **ทันที**ในบรรทัดถัดจาก `mu.Lock()` เสมอ นี่ไม่ใช่แค่สไตล์การเขียนโค้ด แต่เป็นแนวปฏิบัติที่ควรยึดถือเป็นกฎเหล็ก ด้วยเหตุผล:

1. **ป้องกันการลืม Unlock**: ถ้าฟังก์ชันมี logic ซับซ้อนหลายจุด return ได้จากหลายที่ (multiple return path) การเขียน `mu.Unlock()` ตรงๆ ที่ท้ายฟังก์ชันเสี่ยงมากที่จะลืมใส่ในบาง path ทำให้เกิด **deadlock ถาวร** (ล็อกค้างตลอดไป ไม่มีใครปลดล็อกได้อีกเลย) การใช้ `defer` รับประกันว่า `Unlock()` จะถูกเรียกไม่ว่าฟังก์ชันจะ return จากจุดไหนก็ตาม
2. **ปลอดภัยแม้เกิด panic**: ถ้าโค้ดระหว่าง `Lock()` กับ `Unlock()` เกิด panic ขึ้นมา (ตามที่เรียนใน Part 017) การเขียน `Unlock()` ตรงๆ จะไม่ถูกรันเลยเพราะ panic ทำให้ execution กระโดดข้ามไป — แต่ `defer` จะยังถูกเรียกอยู่เสมอแม้เกิด panic (ตราบใดที่ยังไม่ถึงจุดที่ process ตายสนิท) ทำให้ mutex ไม่ค้างล็อกตลอดไป
3. **อ่านง่าย จับคู่ชัดเจน**: การเขียน `Lock()` กับ `defer Unlock()` ติดกันสองบรรทัด ทำให้คนอ่านโค้ดเห็นการจับคู่ Lock/Unlock ได้ทันทีโดยไม่ต้องไล่หาที่ท้ายฟังก์ชัน

ข้อเสียเพียงอย่างเดียวของ `defer mu.Unlock()` คือมี overhead เล็กน้อยเมื่อเทียบกับการเรียก `Unlock()` ตรงๆ (เพราะกลไก `defer` มีต้นทุนของมันเอง) แต่ในเกือบทุกกรณีการใช้งานจริง ความปลอดภัยและความอ่านง่ายที่ได้มาคุ้มค่ากับ overhead ที่แทบวัดไม่ได้นี้เสมอ — ยกเว้นกรณีพิเศษที่ critical section นั้นถูกเรียกในจุดที่ performance สำคัญมากๆ (hot path) ซึ่งจะพูดถึงเพิ่มเติมใน Part 085 (Memory Management และ Optimization)

---

## 5. `sync.RWMutex`: แยก Read Lock และ Write Lock

ในหลายสถานการณ์ ข้อมูลที่แชร์กันถูก**อ่านบ่อยกว่าเขียนมาก** (เช่น ค่า config ที่ตั้งครั้งเดียวแต่อ่านหลายพันครั้ง) การใช้ `sync.Mutex` ธรรมดาจะทำให้แม้แต่การอ่านพร้อมกันหลาย goroutine ก็ต้องเข้าคิวทีละตัว ทั้งที่จริงๆ แล้วการอ่านพร้อมกันโดยไม่มีใครเขียนไม่ก่อให้เกิดปัญหาอะไรเลย

**`sync.RWMutex`** (Read-Write Mutex) แก้ปัญหานี้ด้วยการแยก lock เป็น 2 ประเภท:

- **`RLock()` / `RUnlock()`**: read lock — **หลาย goroutine ถือ read lock พร้อมกันได้** ตราบใดที่ไม่มีใครถือ write lock อยู่
- **`Lock()` / `Unlock()`**: write lock — เหมือน mutex ธรรมดา ต้องรอให้ทุกคน (ทั้ง reader และ writer อื่น) ปล่อยก่อน จึงจะเข้าได้ และระหว่างถือ write lock จะไม่มีใคร (แม้แต่ reader) เข้าได้เลย

```go
package main

import (
	"fmt"
	"sync"
	"time"
)

type SafeConfig struct {
	mu   sync.RWMutex
	data map[string]string
}

func NewSafeConfig() *SafeConfig {
	return &SafeConfig{data: make(map[string]string)}
}

func (c *SafeConfig) Get(key string) string {
	c.mu.RLock() // หลาย goroutine เรียก RLock พร้อมกันได้ ตราบใดที่ไม่มีใครถือ Lock (write lock) อยู่
	defer c.mu.RUnlock()
	time.Sleep(time.Millisecond) // จำลองงานอ่านที่ใช้เวลาเล็กน้อย
	return c.data[key]
}

func (c *SafeConfig) Set(key, value string) {
	c.mu.Lock() // ต้องรอให้ reader ทั้งหมดปล่อยก่อน และกันไม่ให้ reader ใหม่เข้ามาระหว่างเขียน
	defer c.mu.Unlock()
	c.data[key] = value
}

func main() {
	cfg := NewSafeConfig()
	cfg.Set("mode", "production")

	var wg sync.WaitGroup
	// จำลอง reader จำนวนมากอ่านพร้อมกัน
	for i := 0; i < 10; i++ {
		wg.Add(1)
		go func(id int) {
			defer wg.Done()
			_ = cfg.Get("mode")
		}(i)
	}
	wg.Wait()
	fmt.Println("อ่านค่าสำเร็จทั้งหมด ค่าปัจจุบัน:", cfg.Get("mode"))
}
```

ผลลัพธ์:

```
อ่านค่าสำเร็จทั้งหมด ค่าปัจจุบัน: production
```

เนื่องจาก `Get` ใช้ `RLock()` reader ทั้ง 10 goroutine สามารถอ่านค่า `data["mode"]` **พร้อมกัน**ได้จริง (ไม่ต้องเข้าคิวทีละตัวเหมือนใช้ `Mutex` ธรรมดา) ทำให้ throughput สูงขึ้นมากในสถานการณ์ที่อ่านเยอะกว่าเขียนมาก — แต่ถ้ามีการเรียก `Set` (write lock) ขึ้นมาระหว่างนั้น มันจะรอให้ reader ทั้งหมดที่กำลังอ่านอยู่ปล่อยก่อน แล้วกันไม่ให้ reader ใหม่เข้ามาจนกว่าจะเขียนเสร็จ รับประกันว่าจะไม่มีใครอ่านข้อมูลที่กำลังถูกเขียนอยู่ครึ่งๆ กลางๆ

> **กฎการเลือกใช้**: ถ้าอัตราส่วนการอ่านต่อการเขียนสูงมาก (อ่านบ่อยกว่าเขียนมาก) `RWMutex` มักให้ performance ดีกว่า `Mutex` ธรรมดา แต่ถ้าอัตราส่วนใกล้เคียงกันหรือเขียนบ่อยพอๆ กับอ่าน `RWMutex` อาจไม่ได้ให้ประโยชน์อะไรเลย (เพราะมันมี overhead ภายในสูงกว่า `Mutex` ธรรมดาเล็กน้อยจากการต้องนับจำนวน reader) — ควร **benchmark จริง** (ตามที่เรียนใน Part 034) ก่อนตัดสินใจเปลี่ยนจาก `Mutex` เป็น `RWMutex` เสมอ อย่าเปลี่ยนตามความรู้สึกว่า "น่าจะเร็วกว่า"

---

## 6. `sync.WaitGroup`: รอกลุ่ม goroutine ให้ทำงานเสร็จ

กลับมาที่ปัญหาที่เราทิ้งค้างไว้ใน Part 036: จะรอ goroutine หลายตัวให้ทำงานเสร็จ**อย่างแม่นยำ**ได้อย่างไร โดยไม่ใช้ `time.Sleep` ที่เป็นการเดาสุ่ม — คำตอบคือ **`sync.WaitGroup`**

`WaitGroup` ทำงานเหมือน "ตัวนับ" ที่มี 3 method หลัก:

| Method | หน้าที่ |
|---|---|
| `Add(n)` | เพิ่มตัวนับขึ้น `n` — เรียกก่อนที่จะ spawn goroutine เพื่อบอกว่า "จะมีงานเพิ่มอีก n ชิ้นที่ต้องรอ" |
| `Done()` | ลดตัวนับลง 1 — เรียกเมื่อ goroutine ทำงานเสร็จแล้ว (มักเขียนคู่กับ `defer` ที่ต้นฟังก์ชัน) |
| `Wait()` | **block** จนกว่าตัวนับจะลดลงเหลือ 0 |

เราเห็นการใช้งานนี้มาแล้วหลายครั้งในบทก่อนหน้า มาดูรูปแบบมาตรฐานอีกครั้งให้ชัดเจน:

```go
var wg sync.WaitGroup

for i := 0; i < 5; i++ {
	wg.Add(1)          // (1) เพิ่มตัวนับ ก่อน spawn goroutine
	go func() {
		defer wg.Done() // (2) ลดตัวนับ เมื่อ goroutine ทำงานเสร็จ (ใช้ defer เพื่อรับประกันว่าเรียกเสมอแม้ panic)
		// ... ทำงาน ...
	}()
}

wg.Wait() // (3) block รอจนตัวนับเหลือ 0 (คือ goroutine ทุกตัวเรียก Done() ครบแล้ว)
```

`WaitGroup` เป็นวิธีที่**แม่นยำ**ในการรอ ต่างจาก `time.Sleep` ตรงที่ `Wait()` จะปลดล็อกทันทีที่ตัวนับเป็น 0 จริงๆ ไม่ว่า goroutine จะใช้เวลาสั้นหรือยาวแค่ไหนก็ตาม ไม่มีการเดาสุ่มเวลาเลย

---

## 7. ข้อผิดพลาดคลาสสิก: เรียก `Add` ข้างในตัว goroutine

นี่คือข้อผิดพลาดที่พบบ่อยที่สุดอันดับต้นๆ ของการใช้ `WaitGroup`: **เรียก `wg.Add(1)` ข้างในตัว goroutine เอง** แทนที่จะเรียกก่อน spawn

```go
package main

import (
	"fmt"
	"sync"
	"time"
)

func wrongWay() {
	var wg sync.WaitGroup
	counter := 0
	var mu sync.Mutex

	for i := 0; i < 5; i++ {
		go func() {
			wg.Add(1) // ผิด! Add เรียกข้างในตัว goroutine เอง -- race กับ wg.Wait() ที่อาจถูกเรียกไปแล้วก่อนหน้านี้
			defer wg.Done()
			time.Sleep(time.Millisecond)
			mu.Lock()
			counter++
			mu.Unlock()
		}()
	}

	wg.Wait() // อาจ Wait() เสร็จก่อนที่ goroutine บางตัวจะทัน Add(1) ด้วยซ้ำ -> ค่า counter ไม่แน่นอน
	fmt.Println("wrongWay: counter =", counter, "(ค่าอาจไม่ครบ 5 เพราะ Wait() คืนค่าก่อนบาง goroutine เริ่มนับ)")
}

func rightWay() {
	var wg sync.WaitGroup
	counter := 0
	var mu sync.Mutex

	for i := 0; i < 5; i++ {
		wg.Add(1) // ถูกต้อง! Add เรียกก่อน spawn goroutine เสมอ ในฝั่งเดียวกับที่วนลูป
		go func() {
			defer wg.Done()
			time.Sleep(time.Millisecond)
			mu.Lock()
			counter++
			mu.Unlock()
		}()
	}

	wg.Wait()
	fmt.Println("rightWay: counter =", counter, "(ได้ 5 เสมอ)")
}

func main() {
	rightWay()
	wrongWay()
}
```

ผลลัพธ์ (รันหลายครั้งแทบทุกครั้งจะได้แบบนี้ เพราะ `main()` ของ goroutine ยังไม่ทันถูก schedule ก่อนที่ `wg.Wait()` จะถูกเรียก — ตอนนั้นตัวนับยังเป็น 0 อยู่ `Wait()` จึงคืนค่าทันที):

```
rightWay: counter = 5 (ได้ 5 เสมอ)
wrongWay: counter = 0 (ค่าอาจไม่ครบ 5 เพราะ Wait() คืนค่าก่อนบาง goroutine เริ่มนับ)
```

ปัญหาของ `wrongWay` คือ **ลำดับเวลาไม่แน่นอน (race)** ระหว่างการ spawn goroutine (`go func() { wg.Add(1); ... }()`) กับการเรียก `wg.Wait()`:

1. Loop สร้าง goroutine ทั้ง 5 ตัวโดยยังไม่ได้เพิ่มตัวนับเลยสักตัว (เพราะ `Add(1)` อยู่ **ข้างใน** goroutine ซึ่งยังไม่ได้ถูกรันจริง)
2. `wg.Wait()` ถูกเรียกทันทีหลัง loop จบ — ถ้าตอนนั้นตัวนับยังเป็น `0` (เพราะยังไม่มี goroutine ไหนทันได้รัน `Add(1)` เลย) `Wait()` จะคืนค่า**ทันที**โดยไม่รอ
3. `main` เดินหน้าพิมพ์ `counter` ต่อ **ก่อน** ที่ goroutine ตัวไหนจะทันได้ทำงานจริงด้วยซ้ำ

นี่คือเหตุผลที่ Go documentation ของ `sync.WaitGroup` ระบุไว้อย่างชัดเจนว่า:

> **"Note that calls with a positive delta that start when the counter is zero must happen before a Wait."** (การเรียก `Add` ด้วยค่าบวกที่เริ่มต้นตอนตัวนับเป็นศูนย์ ต้องเกิดขึ้น**ก่อน**การเรียก `Wait` เสมอ)

กฎเหล็กที่ต้องจำ: **`wg.Add(n)` ต้องถูกเรียกในฝั่งเดียวกับ loop ที่ spawn goroutine เสมอ (ก่อนคำสั่ง `go`) ไม่ใช่ข้างในตัว goroutine ที่ถูก spawn**

---

## 8. `sync.Once`: การทำ Lazy Initialization

**`sync.Once`** คือเครื่องมือที่รับประกันว่าโค้ดชิ้นหนึ่งจะถูกรัน**เพียงครั้งเดียว**เท่านั้นตลอดอายุโปรแกรม แม้จะถูกเรียกจากหลาย goroutine พร้อมกันก็ตาม — เหมาะมากสำหรับการทำ **lazy initialization** เช่น การสร้าง singleton object ที่ใช้ทรัพยากรมาก (เช่น database connection) โดยสร้างแค่ครั้งแรกที่มีใครต้องการใช้จริงๆ

```go
package main

import (
	"fmt"
	"sync"
)

type Database struct {
	connectionString string
}

var (
	instance *Database
	once     sync.Once
)

// GetDatabase คืนค่า singleton instance -- ไม่ว่าจะเรียกจากกี่ goroutine พร้อมกัน
// โค้ดใน once.Do จะถูกรันแค่ "ครั้งเดียว" เท่านั้นตลอดอายุโปรแกรม
func GetDatabase() *Database {
	once.Do(func() {
		fmt.Println("กำลังสร้าง connection... (ควรเห็นข้อความนี้แค่ครั้งเดียว)")
		instance = &Database{connectionString: "postgres://localhost/mydb"}
	})
	return instance
}

func main() {
	var wg sync.WaitGroup
	for i := 0; i < 10; i++ {
		wg.Add(1)
		go func(id int) {
			defer wg.Done()
			db := GetDatabase()
			_ = db
		}(i)
	}
	wg.Wait()
	fmt.Println("connection string:", GetDatabase().connectionString)
}
```

ผลลัพธ์:

```
กำลังสร้าง connection... (ควรเห็นข้อความนี้แค่ครั้งเดียว)
connection string: postgres://localhost/mydb
```

แม้จะมี 10 goroutine เรียก `GetDatabase()` พร้อมกัน แต่ข้อความ `"กำลังสร้าง connection..."` จะถูกพิมพ์**เพียงครั้งเดียวเท่านั้น** — `sync.Once` รับประกันความปลอดภัยนี้ให้เองโดยอัตโนมัติ ไม่ว่าจะมี race กันมากแค่ไหนก็ตาม (การรับประกันนี้ทำได้เพราะภายใน `Once` ใช้ atomic operation และ mutex ผสมกันเพื่อควบคุมให้โค้ดใน `Do` รันครั้งเดียวเท่านั้น ไม่ว่ากี่ goroutine จะแข่งกันเรียกพร้อมกัน)

ข้อดีเทียบกับการใช้ `if instance == nil { instance = ... }` แบบตรงไปตรงมา: การเช็ค `nil` แบบธรรมดาไม่ปลอดภัยเลยถ้ามีหลาย goroutine เรียกพร้อมกัน (เป็น data race แบบ check-then-act ที่จะกล่าวถึงอีกครั้งใน Part 040) เพราะสอง goroutine อาจเห็น `instance == nil` พร้อมกันแล้วต่างคนต่างสร้าง object ใหม่ซ้อนกันได้ `sync.Once` แก้ปัญหานี้ให้อย่างสมบูรณ์และเขียนโค้ดได้กระชับกว่ามาก

---

## 9. `sync.Map`: เมื่อไรถึงควรใช้จริงๆ

Go มี type พิเศษชื่อ **`sync.Map`** ซึ่งเป็น map ที่ปลอดภัยสำหรับใช้งานพร้อมกันหลาย goroutine โดยไม่ต้องมี mutex ป้องกันเอง:

```go
package main

import (
	"fmt"
	"sync"
)

func main() {
	var m sync.Map // เหมาะกับ: อ่านเยอะ เขียนน้อย และแต่ละ goroutine มักแตะ key คนละตัวกัน (disjoint keys)

	var wg sync.WaitGroup
	for i := 0; i < 20; i++ {
		wg.Add(1)
		go func(id int) {
			defer wg.Done()
			key := fmt.Sprintf("worker-%d", id)
			m.Store(key, id*id) // Store/Load/Delete/Range เป็น method หลักของ sync.Map -- ไม่ต้องใช้ mutex เอง
		}(i)
	}
	wg.Wait()

	count := 0
	m.Range(func(key, value any) bool {
		count++
		return true // return false เพื่อหยุด Range กลางทาง
	})
	fmt.Println("จำนวน entry ทั้งหมด:", count)

	if v, ok := m.Load("worker-5"); ok {
		fmt.Println("worker-5 =>", v)
	}
}
```

ผลลัพธ์:

```
จำนวน entry ทั้งหมด: 20
worker-5 => 25
```

แต่ **`sync.Map` ไม่ใช่ตัวเลือก default ที่ควรใช้ทุกครั้งที่ต้องการ map ที่ปลอดภัยต่อ concurrency** — Go documentation เองก็ระบุไว้ชัดเจนว่า:

> **`sync.Map` เหมาะสมเฉพาะ 2 สถานการณ์เท่านั้น:**
> 1. เมื่อ key ของแต่ละ goroutine ที่เข้าถึงมักจะ**ไม่ซ้ำกัน (disjoint keys)** — แต่ละ goroutine อ่าน/เขียนคนละ key เป็นส่วนใหญ่ ไม่ค่อยมีการแย่งกันเข้าถึง key เดียวกัน
> 2. เมื่อมีการ**อ่านมากกว่าเขียนมาก (read-heavy)** และค่าที่เก็บไว้แทบไม่เปลี่ยนแปลงหลังจากถูกเขียนครั้งแรก

**สำหรับกรณีทั่วไป** ที่ไม่ตรงกับสองเงื่อนไขข้างต้น (เช่น หลาย goroutine แก้ไข key เดียวกันบ่อยๆ หรืออัตราส่วนอ่าน-เขียนใกล้เคียงกัน) **map ธรรมดาคู่กับ `sync.Mutex` (หรือ `sync.RWMutex`) มักให้ประสิทธิภาพดีกว่า `sync.Map`** และยังเข้าใจง่ายกว่าด้วย เพราะ `sync.Map` ถูกออกแบบมาโดยเฉพาะสำหรับ 2 กรณีข้างต้นเท่านั้น (ใช้เทคนิคภายในที่ optimize สำหรับ workload แบบนั้นโดยเฉพาะ) เมื่อ workload ไม่ตรงกับที่มันถูกออกแบบมา ประสิทธิภาพอาจแย่กว่า map+mutex ธรรมดาด้วยซ้ำ

รูปแบบ map+mutex ธรรมดาที่แนะนำสำหรับกรณีทั่วไป:

```go
type SafeCounter struct {
	mu sync.Mutex
	m  map[string]int
}

func (c *SafeCounter) Inc(key string) {
	c.mu.Lock()
	defer c.mu.Unlock()
	c.m[key]++
}
```

> **สรุปกฎการเลือก**: เริ่มต้นด้วย **map ธรรมดา + `sync.Mutex`/`sync.RWMutex`** เป็น default เสมอ แล้วค่อยพิจารณาเปลี่ยนไปใช้ `sync.Map` **เฉพาะเมื่อ benchmark จริง** (Part 034) ยืนยันว่า workload ของคุณตรงกับ 2 เงื่อนไขข้างต้นและ `sync.Map` ให้ผลลัพธ์ดีกว่าจริง — อย่าเลือกใช้ `sync.Map` แค่เพราะชื่อฟังดูเหมือน "ทันสมัยกว่า" หรือ "ปลอดภัยกว่าโดยอัตโนมัติ"

---

## สรุปสิ่งที่ได้เรียนในบทนี้

- แพ็กเกจ `sync` มอบเครื่องมือ synchronization แบบ **shared memory** เป็นทางเลือกคู่กับแนวทาง channel — ทั้งสองแบบใช้ร่วมกันได้ในโปรแกรมเดียวกัน
- **Data Race** เกิดเมื่อหลาย goroutine เข้าถึงตัวแปรเดียวกันพร้อมกันโดยไม่มีการป้องกัน และอย่างน้อยหนึ่งฝั่งเป็นการเขียน — ตรวจจับได้ด้วย flag **`-race`**
- **`sync.Mutex`** (`Lock`/`Unlock`) ป้องกัน critical section ให้มีแค่ goroutine เดียวเข้าถึงได้ในแต่ละขณะ — ควรเขียน **`defer mu.Unlock()`** ทันทีหลัง `Lock()` เสมอ เพื่อป้องกันการลืม unlock และรองรับกรณี panic
- **`sync.RWMutex`** แยก `RLock`/`RUnlock` (อ่านพร้อมกันได้หลาย goroutine) ออกจาก `Lock`/`Unlock` (เขียนได้ทีละคน) เหมาะกับ workload ที่อ่านมากกว่าเขียนมาก
- **`sync.WaitGroup`** (`Add`/`Done`/`Wait`) ใช้รอกลุ่ม goroutine อย่างแม่นยำ — ต้องเรียก **`Add()` ก่อน spawn goroutine เสมอ** ไม่ใช่ข้างในตัว goroutine เอง (ข้อผิดพลาดคลาสสิก)
- **`sync.Once`** รับประกันว่าโค้ดจะรันแค่ครั้งเดียวตลอดอายุโปรแกรม แม้เรียกจากหลาย goroutine พร้อมกัน — เหมาะกับ lazy initialization / singleton
- **`sync.Map`** เหมาะเฉพาะกรณี disjoint keys + read-heavy เท่านั้น — กรณีทั่วไปควรใช้ map ธรรมดา + mutex ก่อนเสมอ แล้วค่อย benchmark เปรียบเทียบ

## แบบฝึกหัดท้ายบท

1. เขียนโปรแกรมที่จงใจสร้าง data race (หลาย goroutine เขียนตัวแปรร่วมกันโดยไม่มี mutex) แล้วรันด้วย `go run -race` ยืนยันว่า Go แจ้ง `WARNING: DATA RACE` จริง จากนั้นแก้ไขด้วย `sync.Mutex` แล้วรันซ้ำเพื่อยืนยันว่าผ่านสะอาด
2. เขียน struct `BankAccount` ที่มี field `balance int` และ method `Deposit(amount int)` กับ `Withdraw(amount int) bool` (คืนค่า `false` ถ้าเงินไม่พอ) โดยป้องกันด้วย `sync.Mutex` ทดสอบด้วยการเรียก `Deposit`/`Withdraw` จากหลาย goroutine พร้อมกันแล้วตรวจสอบว่ายอดเงินสุดท้ายถูกต้อง
3. ดัดแปลงตัวอย่าง `SafeConfig` ในหัวข้อ 5 ให้เพิ่ม method `Delete(key string)` โดยใช้ write lock ที่เหมาะสม แล้วเขียน benchmark เปรียบเทียบระหว่างใช้ `sync.Mutex` ธรรมดา กับ `sync.RWMutex` ในสถานการณ์ที่มี reader 100 ตัวต่อ writer 1 ตัว (จะได้เรียนวิธีเขียน benchmark แบบเต็มรูปแบบใน Part 034 แต่ลองเขียนคร่าวๆ ดูก่อนได้)
4. จงใจเขียนโค้ดที่ทำผิดแบบหัวข้อ 7 (เรียก `Add` ข้างในตัว goroutine) แล้วรันซ้ำ 20 ครั้ง บันทึกว่าได้ผลลัพธ์ที่ผิดพลาด (counter ไม่ครบ) กี่ครั้งจาก 20 ครั้ง แล้วอธิบายว่าทำไมบางครั้งอาจได้ผลลัพธ์ถูกต้องโดยบังเอิญ
5. เขียน struct `LazyResource` ที่มี method `Get() *Resource` ใช้ `sync.Once` เพื่อสร้าง resource แค่ครั้งแรกที่ถูกเรียก ทดสอบเรียกจาก 50 goroutine พร้อมกันและยืนยันว่า resource ถูกสร้างครั้งเดียวจริง (พิมพ์ข้อความตอนสร้างแล้วนับจำนวนครั้งที่พิมพ์)
6. ค้นคว้าเพิ่มเติม: อ่าน Go documentation ของ `sync.Map` อย่างละเอียด (https://pkg.go.dev/sync#Map) แล้วสรุปด้วยคำพูดตัวเองว่าทำไมภายในของมันถึง optimize สำหรับกรณี "อ่านเยอะ เขียนน้อย" ได้ดีกว่า map+mutex ธรรมดา (คำใบ้: ค้นคำว่า "read-only map" กับ "dirty map" ในเอกสารของ Go)

---

**ต่อไป**: [Part 040 — `sync/atomic` และ Lock-Free Programming](./040-sync-atomic.md)
