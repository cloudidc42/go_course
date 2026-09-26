# Part 041: Worker Pool Pattern

> ภาคที่ 3: การทำงานพร้อมกัน (Concurrency) — ตอนที่ 6 จาก 10 (Part 36–45)

## สารบัญของบทนี้

1. ปัญหาที่ Worker Pool แก้: goroutine ไม่ใช่ของฟรี
2. แนวคิดของ Worker Pool Pattern
3. ตัวอย่างสมบูรณ์: Worker Pool ประมวลผลงานจำนวนมาก
4. เจาะลึกทีละขั้นตอน: jobs channel, results channel, WaitGroup
5. การปิด channel อย่างปลอดภัยเมื่อ worker ทำงานเสร็จ
6. การเลือกขนาด Pool: CPU-bound vs I/O-bound
7. Worker Pool ที่ยกเลิกได้ด้วย `errgroup` และ `context`
8. ข้อผิดพลาดที่พบบ่อยเมื่อเขียน Worker Pool
9. สรุปสิ่งที่ได้เรียนในบทนี้
10. แบบฝึกหัดท้ายบท

---

## 1. ปัญหาที่ Worker Pool แก้: goroutine ไม่ใช่ของฟรี

ใน Part 036 เราเรียนไปแล้วว่า goroutine มีค่าใช้จ่ายเริ่มต้นถูกมาก (stack เริ่มต้นแค่ 2KB และขยายได้เอง) จึงเป็นเรื่องปกติที่จะเห็นโปรแกรม Go เปิด goroutine หลักพันหรือหลักหมื่นตัวพร้อมกันโดยไม่มีปัญหา แต่คำถามคือ: **ถ้าต้องประมวลผลงาน 1 ล้านชิ้น ควรเปิด goroutine 1 ล้านตัวเลยไหม?**

คำตอบคือ **ไม่ควร** ด้วยเหตุผลหลายข้อ:

- **หน่วยความจำ**: แม้ stack เริ่มต้นเล็ก แต่ 1 ล้าน goroutine ก็ยังกินหน่วยความจำหลาย GB รวมกับ overhead ของ scheduler
- **Resource ภายนอกมีจำกัด**: ถ้างานแต่ละชิ้นต้องเปิด connection ไปยังฐานข้อมูลหรือเรียก API ภายนอก การเปิด 1 ล้าน connection พร้อมกันจะทำให้ database หรือ API ปลายทางล่มทันที (thundering herd)
- **Context switching overhead**: ยิ่งมี goroutine ที่ "พร้อมทำงาน" (runnable) เยอะเกินจำนวน CPU ที่มีจริง scheduler ก็ยิ่งต้องสลับงานบ่อยขึ้น ซึ่งมีต้นทุน
- **ควบคุมยาก**: ถ้าเปิด goroutine ตามจำนวนงานแบบไม่จำกัด เราจะคุมอัตราการทำงาน (throughput) และจัดการ error/cancellation ได้ยากมาก

ลองดูตัวอย่างปัญหาแบบง่ายๆ: สมมติเรามีงาน 100,000 ชิ้นที่ต้อง "เขียนลงไฟล์" (I/O) การเปิด goroutine 100,000 ตัวพร้อมกันแล้วให้ทุกตัวแข่งกันเขียนไฟล์เดียวกัน หรือเปิด connection ไปยัง service เดียวกันพร้อมกันทั้งหมด จะทำให้ระบบปลายทางรับภาระเกินกำลังทันที

**Worker Pool Pattern** คือคำตอบมาตรฐานของ Go สำหรับปัญหานี้: จำกัดจำนวน goroutine ที่ทำงานพร้อมกันไว้ที่ตัวเลขคงที่ (N ตัว) ไม่ว่าจะมีงานเข้ามากี่ชิ้นก็ตาม

---

## 2. แนวคิดของ Worker Pool Pattern

Worker Pool ประกอบด้วย 3 ส่วนหลัก:

1. **Jobs channel**: channel ที่ใช้ส่งงานเข้าไป worker ทุกตัวจะแข่งกันอ่านงานจาก channel เดียวกันนี้ (ใครว่างก่อนได้งานก่อน — งานกระจายกันเองโดยอัตโนมัติ ไม่ต้องเขียน logic แบ่งงานเอง)
2. **Worker goroutines**: จำนวนคงที่ (N ตัว) แต่ละตัวรันฟังก์ชันเดียวกัน วนอ่านงานจาก jobs channel จนกว่า channel จะถูกปิด
3. **Results channel**: channel ที่ worker แต่ละตัวส่งผลลัพธ์กลับไป ให้ goroutine หลัก (หรือตัวอื่น) นำไปใช้ต่อ

```
                    ┌─────────┐
              ┌────▶│ worker 1 │────┐
              │     └─────────┘    │
   jobs chan  │     ┌─────────┐    │   results chan
  ───────────▶├────▶│ worker 2 │────┼──────────────▶
              │     └─────────┘    │
              │     ┌─────────┐    │
              └────▶│ worker 3 │────┘
                    └─────────┘
```

จุดสำคัญ: **ไม่ว่างานจะมี 12 ชิ้นหรือ 12 ล้านชิ้น จำนวน goroutine ที่ทำงานพร้อมกันจริงจะคงที่อยู่ที่ N ตัวเสมอ** นี่คือสิ่งที่ทำให้ pattern นี้ scale ได้อย่างปลอดภัย

---

## 3. ตัวอย่างสมบูรณ์: Worker Pool ประมวลผลงานจำนวนมาก

มาดูตัวอย่างที่รันได้จริง: จำลองการประมวลผลงาน 12 ชิ้น (คำนวณกำลังสองของตัวเลข พร้อม `time.Sleep` จำลองงานที่ใช้เวลา เช่น เรียก API ภายนอก) ด้วย worker 3 ตัว

```go
package main

import (
	"fmt"
	"math/rand"
	"sync"
	"time"
)

// Job คือหน่วยงานที่ต้องประมวลผล ในตัวอย่างนี้คือ "หมายเลขที่ต้องคำนวณ"
type Job struct {
	ID    int
	Input int
}

// Result คือผลลัพธ์ของงานแต่ละชิ้น
type Result struct {
	JobID  int
	Output int
}

// worker คือ goroutine หนึ่งตัวที่วนอ่านงานจาก jobs channel เรื่อยๆ
// จนกว่า channel จะถูกปิด (closed) แล้วส่งผลลัพธ์เข้า results channel
func worker(id int, jobs <-chan Job, results chan<- Result, wg *sync.WaitGroup) {
	defer wg.Done()
	for job := range jobs {
		// จำลองงานที่ใช้เวลา เช่น เรียก API ภายนอก หรือคำนวณหนัก
		time.Sleep(time.Duration(50+rand.Intn(100)) * time.Millisecond)
		output := job.Input * job.Input
		fmt.Printf("worker %d: ประมวลผล job %d (input=%d) -> %d\n", id, job.ID, job.Input, output)
		results <- Result{JobID: job.ID, Output: output}
	}
}

func main() {
	const numJobs = 12
	const numWorkers = 3

	jobs := make(chan Job, numJobs)
	results := make(chan Result, numJobs)

	var wg sync.WaitGroup
	// เปิด worker จำนวนคงที่ (numWorkers) — ไม่ว่าจะมีงานกี่ชิ้นก็ตาม
	for w := 1; w <= numWorkers; w++ {
		wg.Add(1)
		go worker(w, jobs, results, &wg)
	}

	// ส่งงานทั้งหมดเข้า jobs channel
	for j := 1; j <= numJobs; j++ {
		jobs <- Job{ID: j, Input: j}
	}
	close(jobs) // สำคัญมาก: ปิด jobs เพื่อบอก worker ว่าไม่มีงานเพิ่มแล้ว ทำให้ range จบลูป

	// goroutine แยกต่างหากคอยปิด results เมื่อ worker ทุกตัวทำงานเสร็จ
	go func() {
		wg.Wait()
		close(results)
	}()

	// อ่านผลลัพธ์ทั้งหมดที่ main goroutine (จะบล็อกจนกว่า results จะถูกปิด)
	sum := 0
	count := 0
	for r := range results {
		sum += r.Output
		count++
	}

	fmt.Printf("ประมวลผลครบ %d jobs, ผลรวม = %d\n", count, sum)
}
```

รันด้วย `go run -race main.go` (ตรวจสอบด้วย race detector เสมอเมื่อเขียนโค้ด concurrent — จะเรียนละเอียดใน Part 044) ผลลัพธ์ตัวอย่าง:

```
worker 1: ประมวลผล job 3 (input=3) -> 9
worker 1: ประมวลผล job 4 (input=4) -> 16
worker 3: ประมวลผล job 2 (input=2) -> 4
worker 2: ประมวลผล job 1 (input=1) -> 1
worker 1: ประมวลผล job 5 (input=5) -> 25
worker 3: ประมวลผล job 6 (input=6) -> 36
worker 2: ประมวลผล job 7 (input=7) -> 49
worker 1: ประมวลผล job 8 (input=8) -> 64
worker 3: ประมวลผล job 9 (input=9) -> 81
worker 2: ประมวลผล job 10 (input=10) -> 100
worker 3: ประมวลผล job 12 (input=12) -> 144
worker 1: ประมวลผล job 11 (input=11) -> 121
ประมวลผลครบ 12 jobs, ผลรวม = 650
```

สังเกตว่า **ลำดับที่ worker แต่ละตัวได้งานไม่แน่นอน** (nondeterministic) — เพราะ worker ไหนว่างก่อนก็คว้างานจาก channel ไปทำก่อน นี่คือพฤติกรรมที่ถูกต้องและเป็นธรรมชาติของ pattern นี้ ผลรวมสุดท้าย (650 = 1²+2²+...+12²) จะถูกต้องเสมอไม่ว่าลำดับจะเป็นอย่างไร เพราะเราไม่ได้พึ่งพาลำดับการประมวลผลเลย

---

## 4. เจาะลึกทีละขั้นตอน: jobs channel, results channel, WaitGroup

### ทำไม jobs ต้องเป็น `<-chan Job` และ results ต้องเป็น `chan<- Result`

```go
func worker(id int, jobs <-chan Job, results chan<- Result, wg *sync.WaitGroup)
```

สังเกต type ของ parameter: `jobs` เป็น **receive-only channel** (`<-chan Job`) และ `results` เป็น **send-only channel** (`chan<- Result`) นี่ไม่ใช่แค่สไตล์การเขียนโค้ดสวยๆ แต่เป็นการใช้ **type system ของ Go บังคับทิศทางการไหลของข้อมูล** — ถ้า worker เผลอเขียน `jobs <- something` compiler จะ error ทันที เพราะประกาศไว้ชัดเจนว่า worker มีสิทธิ์แค่ "อ่าน" จาก jobs และ "เขียน" ลง results เท่านั้น นี่เป็นแนวปฏิบัติที่ดีมากเมื่อเขียนฟังก์ชันที่รับ channel เป็นพารามิเตอร์ ควรระบุทิศทางเสมอถ้าเป็นไปได้

### ทำไมต้องใช้ `sync.WaitGroup` (ทบทวนจาก Part 039)

ปัญหาคือ: main goroutine จะรู้ได้อย่างไรว่า worker **ทุกตัว** ทำงานเสร็จแล้ว เพื่อจะได้ปิด `results` channel อย่างปลอดภัย?

ถ้าปิด `results` เร็วเกินไป (ตอนที่ worker บางตัวยังทำงานไม่เสร็จ) worker ตัวนั้นจะพยายามส่งค่าเข้า channel ที่ถูกปิดแล้ว ซึ่งจะทำให้เกิด **panic: send on closed channel** ทันที

`sync.WaitGroup` แก้ปัญหานี้ตรงๆ:

```go
var wg sync.WaitGroup
for w := 1; w <= numWorkers; w++ {
	wg.Add(1)           // เพิ่ม counter ก่อนเปิด goroutine แต่ละตัว
	go worker(w, jobs, results, &wg)
}

go func() {
	wg.Wait()            // บล็อกจนกว่า counter จะเป็น 0 (worker ทุกตัวเรียก Done() ครบ)
	close(results)        // ปิด results อย่างปลอดภัย เพราะรู้แน่ชัดว่าไม่มีใครส่งเข้ามาอีก
}()
```

### รูปแบบ "ปิด channel ด้วย goroutine แยก" คือธรรมเนียมมาตรฐาน

สังเกตว่าเราไม่เรียก `wg.Wait()` ตรงๆ ใน `main()` ก่อนอ่าน `results` เพราะถ้าทำแบบนั้น เราจะติด **deadlock**: `wg.Wait()` จะบล็อก main goroutine รอ worker ทำงานเสร็จ แต่ worker เองก็บล็อกอยู่ที่ `results <- ...` เพราะไม่มีใครมาอ่านจาก `results` เลย (เพราะ main ไปรอที่ `wg.Wait()` ก่อน)

การแก้คือแยก `wg.Wait()` กับ `close(results)` ไปไว้ใน goroutine ต่างหาก ปล่อยให้ main goroutine ไป `range results` ได้เลยทันที ทำให้ทั้งสองฝั่งทำงานพร้อมกันได้จริง — นี่คือรูปแบบที่จะเจอซ้ำแล้วซ้ำอีกตลอดภาคนี้ (fan-in ใน Part 042 ก็ใช้แนวคิดเดียวกัน)

---

## 5. การปิด channel อย่างปลอดภัยเมื่อ worker ทำงานเสร็จ

กฎทองของการปิด channel ใน Go ที่ต้องจำให้ขึ้นใจ (ทบทวนจาก Part 037):

> **ให้ฝั่งที่ "ส่ง" (sender) เป็นคนปิด channel เสมอ ไม่ใช่ฝั่งที่ "รับ" (receiver)** และห้ามปิด channel ที่มีคนอื่นกำลังจะส่งข้อมูลเข้ามาอยู่ (ไม่เช่นนั้นจะ panic)

ในตัวอย่างของเรา:

- `jobs` channel: **main goroutine** เป็นคนส่ง (producer) จึงเป็นคนปิดหลังส่งงานครบ
- `results` channel: **worker ทั้งหมด** เป็นคนส่ง (producer) แต่เพราะมีหลายตัว จึงต้องมีกลไกกลาง (WaitGroup) มารอให้ทุกตัวส่งเสร็จก่อน แล้วให้ "ตัวแทน" (goroutine ที่เรียก `wg.Wait()`) เป็นคนปิดแทน

หากมีหลาย goroutine พยายามปิด channel เดียวกันพร้อมกัน (ไม่ว่ากรณีไหน) จะเกิด panic **close of closed channel** ทันที — จำไว้ว่า channel หนึ่งควรมี "เจ้าของ" ที่รับผิดชอบการปิดเพียงจุดเดียวเสมอ

---

## 6. การเลือกขนาด Pool: CPU-bound vs I/O-bound

คำถามที่พบบ่อยที่สุดของ Worker Pool คือ "ควรเปิด worker กี่ตัวดี?" คำตอบขึ้นอยู่กับ **ลักษณะของงาน**:

### งานแบบ CPU-bound (คำนวณหนัก ไม่รอ I/O)

ถ้างานคือการคำนวณล้วนๆ (เช่น เข้ารหัส, ประมวลผลภาพ, sorting ข้อมูลขนาดใหญ่) การเปิด worker เกินจำนวน CPU จริงจะไม่ช่วยอะไร เพราะ CPU core หนึ่งตัวรันได้ทีละ 1 goroutine ในเวลาหนึ่งขณะอยู่ดี (ไม่นับ hyperthreading) ค่าที่แนะนำคือ:

```go
numWorkers := runtime.NumCPU()
```

`runtime.NumCPU()` คืนจำนวน logical CPU ที่เครื่องมองเห็น (ทดสอบบนเครื่องที่ใช้เขียนบทนี้ได้ค่า 4) การเปิด worker มากกว่านี้จะทำให้เกิด context switching overhead โดยไม่ได้ throughput เพิ่มขึ้นจริง

```go
package main

import (
	"fmt"
	"runtime"
)

func main() {
	fmt.Println("จำนวน logical CPU บนเครื่องนี้:", runtime.NumCPU())
	fmt.Println("จำนวน OS thread สูงสุดที่ Go runtime อนุญาตให้ใช้พร้อมกัน (GOMAXPROCS):", runtime.GOMAXPROCS(0))
}
```

ผลลัพธ์ (บนเครื่อง 4 core):

```
จำนวน logical CPU บนเครื่องนี้: 4
จำนวน OS thread สูงสุดที่ Go runtime อนุญาตให้ใช้พร้อมกัน (GOMAXPROCS): 4
```

`GOMAXPROCS` คือจำนวน OS thread สูงสุดที่ Go runtime scheduler จะใช้รัน goroutine พร้อมกันจริงๆ (บน CPU core) ค่า default จะเท่ากับ `NumCPU()` เสมอตั้งแต่ Go เวอร์ชันปัจจุบัน

### งานแบบ I/O-bound (รอ network, disk, database)

ถ้างานส่วนใหญ่คือ "รอ" (เช่น เรียก HTTP API ภายนอก, query ฐานข้อมูล, อ่านไฟล์ขนาดใหญ่) ระหว่างที่ worker ตัวหนึ่งรอ I/O อยู่ CPU ก็ว่างพอที่จะให้ worker ตัวอื่นทำงานได้ ดังนั้นจำนวน worker ที่เหมาะสมมักจะ **มากกว่า** `NumCPU()` หลายเท่า เพราะ Go runtime จะจัดการสลับ goroutine ที่กำลังรอ I/O (ผ่าน netpoller) ให้ core ว่างไปทำงานอื่นโดยอัตโนมัติ

ตัวเลขที่เหมาะสมสำหรับงาน I/O-bound ไม่มีสูตรตายตัว ขึ้นอยู่กับ:

- ปลายทางรองรับ concurrent connection ได้กี่ connection (เช่น database connection pool มีจำกัด — จะเรียนใน Part 078)
- Latency เฉลี่ยของงานแต่ละชิ้น
- Rate limit ของ API ภายนอก (ถ้ามี — ดู pattern rate limiting ใน Part 045)

แนวทางปฏิบัติจริง: **เริ่มจากค่ากลางๆ แล้ววัดผลจริง** (benchmark ด้วย Part 034 หรือ profiling ด้วย Part 083) แทนที่จะเดาตัวเลข

| ลักษณะงาน | ขนาด Pool ที่แนะนำ | เหตุผล |
|---|---|---|
| CPU-bound (คำนวณหนัก) | `runtime.NumCPU()` | มากกว่านี้ไม่ช่วย เพราะ CPU คือคอขวด |
| I/O-bound (รอ network/disk) | มากกว่า `NumCPU()` หลายเท่า (ทดลองและวัดผล) | ระหว่างรอ I/O core ว่างไปช่วยงานอื่นได้ |
| Mixed (คำนวณ + I/O) | เริ่มจาก `NumCPU() * 2` แล้วปรับตาม benchmark | ต้องวัดผลจริงเสมอ |

---

## 7. Worker Pool ที่ยกเลิกได้ด้วย `errgroup` และ `context`

Worker Pool แบบพื้นฐานที่เห็นไปมีข้อจำกัด: ถ้า worker ตัวใดตัวหนึ่งเจอ error ร้ายแรง (เช่น token หมดอายุ, service ปลายทางล่ม) เรา**ไม่มีกลไกบอก worker ตัวอื่นให้หยุดทำงานทันที** — worker ตัวอื่นจะทำงานต่อไปเรื่อยๆ จนกว่างานจะหมด ซึ่งเปลืองทรัพยากรและอาจทำให้ error เกิดซ้ำซ้อน

นี่คือจุดที่ package `golang.org/x/sync/errgroup` เข้ามาช่วย — เป็น extension official ของทีม Go (อยู่นอก standard library แต่ดูแลโดยทีมเดียวกัน) ที่รวม pattern "รอ goroutine กลุ่มหนึ่งจบ + เก็บ error ตัวแรก + ยกเลิกที่เหลือด้วย context" ไว้ในที่เดียว

ติดตั้งด้วย:

```bash
go get golang.org/x/sync/errgroup
```

ตัวอย่าง Worker Pool ที่ใช้ `errgroup.WithContext`:

```go
package main

import (
	"context"
	"fmt"
	"math/rand"
	"time"

	"golang.org/x/sync/errgroup"
)

// processURL จำลองการดึงข้อมูลจาก URL หนึ่งรายการ
// ถ้า input เป็นเลขที่หารด้วย 7 ลงตัว จะจำลองว่าเกิด error (เช่น server ตอบ 500)
func processURL(ctx context.Context, id int) (int, error) {
	select {
	case <-time.After(time.Duration(20+rand.Intn(60)) * time.Millisecond):
		// ทำงานสำเร็จ (จำลอง)
	case <-ctx.Done():
		return 0, ctx.Err()
	}
	if id == 7 {
		return 0, fmt.Errorf("job %d: ปลายทางตอบ HTTP 500", id)
	}
	return id * 10, nil
}

func main() {
	const numJobs = 15
	const numWorkers = 4

	ctx, cancel := context.WithTimeout(context.Background(), 3*time.Second)
	defer cancel()

	g, ctx := errgroup.WithContext(ctx)

	jobs := make(chan int)
	// ตัว generator: ส่งงานเข้า jobs channel แล้วปิดเมื่อครบ หรือหยุดทันทีถ้า ctx ถูกยกเลิก
	g.Go(func() error {
		defer close(jobs)
		for i := 1; i <= numJobs; i++ {
			select {
			case jobs <- i:
			case <-ctx.Done():
				return ctx.Err()
			}
		}
		return nil
	})

	// เปิด worker จำนวน numWorkers ตัว ทุกตัวใช้ g.Go เดียวกัน
	for w := 1; w <= numWorkers; w++ {
		workerID := w
		g.Go(func() error {
			for {
				select {
				case job, ok := <-jobs:
					if !ok {
						return nil // jobs ถูกปิดแล้ว ไม่มีงานเหลือ ออกจาก worker ได้ปกติ
					}
					result, err := processURL(ctx, job)
					if err != nil {
						// return error ตัวแรกที่เจอ -> errgroup จะยกเลิก ctx ให้ worker อื่นหยุดตาม
						return fmt.Errorf("worker %d: %w", workerID, err)
					}
					fmt.Printf("worker %d: job %d เสร็จ -> %d\n", workerID, job, result)
				case <-ctx.Done():
					return ctx.Err()
				}
			}
		})
	}

	// g.Wait() บล็อกจนกว่าทุก goroutine ที่เปิดด้วย g.Go จะจบ
	// และคืนค่า error ตัวแรกที่เกิดขึ้น (ถ้ามี)
	if err := g.Wait(); err != nil {
		fmt.Println("เกิดข้อผิดพลาด:", err)
		return
	}
	fmt.Println("ทุกงานเสร็จสมบูรณ์โดยไม่มี error")
}
```

รันด้วย `go run -race main.go` (ทดสอบซ้ำหลายรอบเพื่อยืนยันความถูกต้อง) ผลลัพธ์ตัวอย่าง:

```
worker 1: job 4 เสร็จ -> 40
worker 4: job 3 เสร็จ -> 30
worker 2: job 2 เสร็จ -> 20
worker 3: job 1 เสร็จ -> 10
worker 1: job 5 เสร็จ -> 50
เกิดข้อผิดพลาด: worker 2: job 7: ปลายทางตอบ HTTP 500
```

สังเกตว่าโปรแกรมหยุดทันทีที่เจองาน id 7 (ที่จำลองว่า error) โดยไม่รอให้งานที่เหลือ (8-15) ถูกประมวลผลจนครบ — นี่คือสิ่งที่ Worker Pool แบบพื้นฐานทำไม่ได้ถ้าไม่มี `errgroup` มาช่วย

### กลไกเบื้องหลัง `errgroup.WithContext`

- `errgroup.WithContext(ctx)` คืน `*errgroup.Group` และ `context.Context` ตัวใหม่ที่ผูกกับ group นี้
- ทุกครั้งที่เรียก `g.Go(fn)` มันจะเปิด goroutine รัน `fn` ให้ (ไม่ต้องเขียน `go` เอง ไม่ต้องจัดการ `WaitGroup` เอง)
- ถ้า `fn` ตัวใดตัวหนึ่ง return error ที่ไม่ใช่ `nil` — **ctx ที่ได้จาก `errgroup.WithContext` จะถูกยกเลิกทันที** (เรียก cancel function ภายในให้อัตโนมัติ) ทำให้ goroutine อื่นที่กำลัง `select` รอ `ctx.Done()` อยู่หลุดออกมาทำงานต่อได้ทันที
- `g.Wait()` จะรอทุก goroutine จบ (เหมือน `WaitGroup.Wait()`) แล้วคืนค่า **error ตัวแรกที่เกิดขึ้นเท่านั้น** (ไม่ใช่ทุก error) ซึ่งเพียงพอสำหรับกรณีส่วนใหญ่ที่แค่ต้องการรู้ว่า "มีอะไรพังไหม พังเพราะอะไร"

นี่คือเหตุผลที่ `errgroup` กลายเป็นตัวเลือกยอดนิยมสำหรับ Worker Pool ในโค้ด production จริง เพราะรวม 3 อย่างที่ต้องเขียนเองตลอด (goroutine management + error propagation + cancellation) ไว้ใน API เดียว

---

## 8. ข้อผิดพลาดที่พบบ่อยเมื่อเขียน Worker Pool

### ข้อผิดพลาดที่ 1: ปิด jobs channel ก่อนส่งงานครบ

```go
close(jobs)
jobs <- Job{ID: 1} // panic: send on closed channel
```

ต้องปิด **หลัง** ส่งงานครบทุกชิ้นเสมอ

### ข้อผิดพลาดที่ 2: ลืม `wg.Add(1)` ก่อนเปิด goroutine

```go
go func() {
	wg.Add(1) // ผิด! อาจรันหลัง wg.Wait() ไปแล้ว ทำให้ Wait() คืนค่าก่อนที่คาดไว้
	defer wg.Done()
	// ...
}()
```

`wg.Add()` ต้องเรียกจาก goroutine ที่เปิด goroutine ใหม่ (ปกติคือ main goroutine) **ก่อน** สั่ง `go` เสมอ ไม่ใช่เรียกจากข้างในตัว goroutine เอง (ปัญหานี้เป็น race condition ที่ตรวจจับได้ด้วย `-race` เช่นกัน)

### ข้อผิดพลาดที่ 3: capture ตัวแปร loop ผิดใน Go เวอร์ชันเก่า

```go
for w := 1; w <= numWorkers; w++ {
	go worker(w, jobs, results, &wg) // ปลอดภัยตั้งแต่ Go 1.22 เป็นต้นไป
}
```

ก่อน Go 1.22 ตัวแปร loop (`w`) จะถูกใช้ร่วมกันข้าม iteration ทำให้ goroutine ทุกตัวอาจเห็นค่า `w` เป็นค่าสุดท้ายเหมือนกันหมด (ต้อง copy ตัวแปรเข้า local variable ก่อน) Go 1.22 ขึ้นไปแก้ปัญหานี้ให้อัตโนมัติแล้ว (ตัวแปร loop เป็นตัวใหม่ทุก iteration) แต่ถ้าทำงานกับโค้ดเก่าหรือเห็นโค้ดแบบนี้ ให้ระวังเรื่องนี้ไว้เสมอ

### ข้อผิดพลาดที่ 4: เปิด worker มากเกินจำเป็นโดยไม่มีเหตุผล

การใส่ `numWorkers := 10000` "เผื่อไว้" โดยไม่ได้วัดผลจริงคือ anti-pattern — วกกลับไปสู่ปัญหาเดิมที่ Worker Pool ถูกสร้างมาเพื่อแก้ (ดูหัวข้อ 6)

---

## สรุปสิ่งที่ได้เรียนในบทนี้

- Worker Pool Pattern จำกัดจำนวน goroutine ที่ทำงานพร้อมกันไว้ที่ N ตัวคงที่ แม้จะมีงานเข้ามาจำนวนมาก เพื่อป้องกันปัญหาหน่วยความจำและ resource ภายนอกล้น
- โครงสร้างหลักคือ jobs channel (ให้ worker แข่งกันอ่าน) + worker goroutines จำนวนคงที่ + results channel
- `sync.WaitGroup` ใช้รู้ว่า worker ทุกตัวทำงานเสร็จเมื่อไร เพื่อปิด results channel ได้อย่างปลอดภัยผ่าน goroutine แยกต่างหาก
- กฎทอง: ฝั่งที่ "ส่ง" ต้องเป็นคนปิด channel เสมอ และควรมีเจ้าของเพียงจุดเดียวต่อ channel
- ขนาด Pool ที่เหมาะสมขึ้นอยู่กับลักษณะงาน: CPU-bound ใช้ `runtime.NumCPU()`, I/O-bound มักใช้มากกว่านั้นหลายเท่าและต้องวัดผลจริง
- `golang.org/x/sync/errgroup` รวม goroutine management + error propagation + context cancellation ไว้ในที่เดียว เหมาะกับ Worker Pool ที่ต้องหยุดทันทีเมื่อเจอ error แรก
- ข้อผิดพลาดที่พบบ่อย: ปิด channel ผิดจังหวะ, `wg.Add()` ผิดตำแหน่ง, เปิด worker มากเกินจำเป็น

## แบบฝึกหัดท้ายบท

1. ดัดแปลงตัวอย่างในหัวข้อ 3 ให้ประมวลผลงาน 50 ชิ้นแทน 12 ชิ้น แล้วลองปรับ `numWorkers` เป็น 1, 5, 20 ตามลำดับ สังเกตความเร็วรวมที่เปลี่ยนไป (ใช้ `time.Since` วัดเวลา) แล้วอธิบายว่าทำไมเพิ่ม worker เกินจุดหนึ่งแล้วความเร็วไม่ต่างกันมาก
2. เขียน Worker Pool ที่ประมวลผล error ของแต่ละ job แยกจากผลลัพธ์ปกติ (เช่น ให้ `Result` มี field `Err error` แทนที่จะให้ error ทำให้ทั้งโปรแกรมหยุด) — ใช้แนวทางไหนดีกว่ากันระหว่างวิธีนี้กับการใช้ `errgroup`?
3. ทดลองลบ `close(jobs)` ออกจากตัวอย่างในหัวข้อ 3 แล้วรันดู เกิดอะไรขึ้น? อธิบายด้วยคำพูดของตัวเองว่าทำไมโปรแกรมไม่จบ (deadlock)
4. เขียนโปรแกรมวัดว่า `runtime.NumCPU()` บนเครื่องของคุณคืนค่าเท่าไร แล้วลองรัน Worker Pool แบบ CPU-bound จริง (เช่น คำนวณจำนวนเฉพาะ) โดยปรับ `numWorkers` ให้เท่ากับ, น้อยกว่า และมากกว่า `NumCPU()` วัดเวลาที่ใช้แต่ละกรณีเปรียบเทียบกัน
5. ดัดแปลงตัวอย่าง `errgroup` ในหัวข้อ 7 ให้จำลอง error ที่ job หลายตัวพร้อมกัน (เช่น job ที่หารด้วย 3 และ 7 ลงตัวทั้งคู่ error) แล้วสังเกตว่า error ที่ `g.Wait()` คืนกลับมาคือ error ของ job ไหน อธิบายว่าทำไมถึงเป็นแบบนั้น
6. ลองเขียน Worker Pool เวอร์ชันที่ใช้ buffered channel ขนาดพอดีกับจำนวนงาน (เหมือนตัวอย่างหัวข้อ 3) เทียบกับเวอร์ชันที่ใช้ unbuffered channel ทั้ง jobs และ results — มีความแตกต่างด้าน timing หรือพฤติกรรมอย่างไรบ้าง?

---

**ต่อไป**: [Part 042 — Fan-in / Fan-out Pattern](./042-fan-in-fan-out.md)
