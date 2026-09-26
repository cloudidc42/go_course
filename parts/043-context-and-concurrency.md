# Part 043: Context กับ Concurrency: cancellation, timeout

> ภาคที่ 3: การทำงานพร้อมกัน (Concurrency) — ตอนที่ 8 จาก 10 (Part 36–45)

## สารบัญของบทนี้

1. ทบทวน `context.Context` จาก Part 032 ในมุมมองของ concurrency
2. ธรรมเนียม: `ctx` ต้องเป็นพารามิเตอร์ตัวแรกเสมอ
3. Goroutine Leak คืออะไร และทำไมอันตราย
4. สาธิต: โปรแกรมที่ leak goroutine จริง วัดด้วย `runtime.NumGoroutine()`
5. แก้ปัญหาด้วย `select` + `ctx.Done()`
6. ตัวอย่างสมบูรณ์: Fan-out Worker Pool ที่หยุดทันทีเมื่อ ctx ถูกยกเลิก
7. `context.WithTimeout` ครอบทั้ง Pipeline
8. การส่งต่อ (propagate) ctx เดียวผ่าน call tree ทั้งหมด
9. ข้อควรระวังและ anti-pattern ของการใช้ context
10. สรุปสิ่งที่ได้เรียนในบทนี้
11. แบบฝึกหัดท้ายบท

---

## 1. ทบทวน `context.Context` จาก Part 032 ในมุมมองของ concurrency

ใน Part 032 เราเรียนเรื่องแพ็กเกจ `context` ไปแล้วในภาพรวม: `context.Context` คือค่าที่ใช้ส่งต่อ **สัญญาณยกเลิก (cancellation signal)**, **เส้นตาย (deadline/timeout)**, และ **ค่าประกอบ (request-scoped values)** ผ่านฟังก์ชันหลายชั้น เราเห็นการใช้งานพื้นฐานอย่าง `context.WithCancel`, `context.WithTimeout`, `context.WithDeadline`, `context.WithValue` และเมธอด `ctx.Done()`, `ctx.Err()`

บทนี้จะไม่พูดซ้ำเรื่องพื้นฐานเหล่านั้น แต่จะเจาะลึกเฉพาะ **สถานการณ์ concurrency ที่มี goroutine หลายตัวทำงานพร้อมกัน** ซึ่งเป็นจุดที่ `context` แสดงพลังที่แท้จริงออกมา: การสั่งยกเลิก goroutine จำนวนมากพร้อมกันด้วยการเรียก cancel function เพียงครั้งเดียว

หัวใจสำคัญที่ต้องจำ:

> **`ctx.Done()` คืน channel ที่จะถูกปิดเมื่อ context ถูกยกเลิกหรือหมดเวลา** และ **channel ที่ถูกปิดแล้ว จะทำให้ทุกคนที่ `<-ctx.Done()` ได้รับค่าทันที (zero value) ไม่ว่าจะมีกี่ goroutine ที่รออยู่ก็ตาม**

นี่คือกลไกเดียวกับ **done channel** ที่เราเรียนไปใน Part 042 เพียงแต่ `context` เป็นเวอร์ชันที่ครบเครื่องกว่า: มี timeout ในตัว มี `ctx.Err()` บอกเหตุผลที่ถูกยกเลิก และเป็นมาตรฐานที่ library แทบทุกตัวใน ecosystem Go ยอมรับร่วมกัน

---

## 2. ธรรมเนียม: `ctx` ต้องเป็นพารามิเตอร์ตัวแรกเสมอ

ก่อนลงลึกเรื่องเทคนิค ต้องย้ำธรรมเนียมสำคัญที่สุดข้อหนึ่งของโค้ด Go สาย concurrency ให้ชัดเจน:

> **ฟังก์ชันใดก็ตามที่รับ `context.Context` เป็นพารามิเตอร์ ต้องรับไว้เป็นพารามิเตอร์ตัวแรกเสมอ และตั้งชื่อว่า `ctx`**

```go
// ถูกต้องตามธรรมเนียม
func worker(ctx context.Context, id int, jobs <-chan Job) error

// ผิดธรรมเนียม (แม้จะ compile ผ่าน)
func worker(id int, ctx context.Context, jobs <-chan Job) error
```

นี่ไม่ใช่กฎภาษา (compiler ไม่บังคับ) แต่เป็นธรรมเนียมที่เขียนไว้ใน [เอกสารทางการของแพ็กเกจ `context`](https://pkg.go.dev/context) โดยตรง และทีมงาน Go ยึดถืออย่างเคร่งครัดในทุก standard library และ tool ที่รับ `context` เหตุผลคือทำให้คนอ่านโค้ดรู้ทันทีว่าฟังก์ชันนี้ "อยู่ภายใต้การควบคุมของ ctx" โดยไม่ต้องอ่านไปถึงพารามิเตอร์ตัวสุดท้าย และทำให้โค้ดทั้ง ecosystem มีหน้าตาสอดคล้องกัน

ธรรมเนียมนี้สำคัญมากเป็นพิเศษในบทนี้ เพราะเรากำลังจะส่ง `ctx` **ตัวเดียวกัน** ผ่านฟังก์ชันและ goroutine จำนวนมาก — ถ้าตำแหน่งพารามิเตอร์ไม่สอดคล้องกัน โค้ดจะอ่านยากขึ้นทันที

---

## 3. Goroutine Leak คืออะไร และทำไมอันตราย

**Goroutine leak** คือสถานการณ์ที่ goroutine หนึ่งตัวถูกเปิดขึ้นมา แต่ **ไม่มีวันจบการทำงาน** เพราะติดอยู่ในสถานะบล็อก (เช่น รอรับค่าจาก channel ที่ไม่มีใครส่งอีกแล้ว หรือพยายามส่งค่าเข้า channel ที่ไม่มีใครอ่านอีกแล้ว) โดยไม่มีทางออกใดๆ ให้เลย

ปัญหาของ goroutine leak คือ:

- **หน่วยความจำรั่วไหลอย่างช้าๆ**: แต่ละ goroutine ที่ leak กินหน่วยความจำอย่างน้อยเท่ากับขนาด stack (เริ่มต้น 2KB) บวกกับตัวแปรที่มันอ้างอิงถึง (ซึ่ง Garbage Collector เก็บกวาดไม่ได้ เพราะ goroutine ที่ยัง "มีชีวิต" นับเป็น root ที่ GC ต้องรักษาไว้)
- **ตรวจจับยากมาก**: โปรแกรมจะทำงานได้ปกติในตอนแรก แต่หน่วยความจำจะค่อยๆ เพิ่มขึ้นเรื่อยๆ (memory leak แบบ slow burn) กว่าจะสังเกตเห็นอาจต้องรอเป็นชั่วโมงหรือเป็นวันใน production
- **ไม่มี error ให้เห็น**: goroutine ที่ leak ไม่ panic ไม่ crash โปรแกรมยังทำงานต่อไปได้ปกติ (จนกว่าหน่วยความจำจะหมด) ทำให้ตรวจจับด้วยการทดสอบทั่วไปได้ยาก

สาเหตุที่พบบ่อยที่สุดของ goroutine leak ทั้งหมดสรุปได้เป็นข้อเดียว: **goroutine ไม่มี "ทางออก" ที่รับประกันว่าจะถูกกระตุ้นแน่นอน** ซึ่งมักเกิดจาก goroutine ที่ทำ `<-ch` หรือ `ch <- v` ตรงๆ โดยไม่มี `select` คู่กับกลไกยกเลิกใดๆ เลย

---

## 4. สาธิต: โปรแกรมที่ leak goroutine จริง วัดด้วย `runtime.NumGoroutine()`

มาดูตัวอย่าง goroutine leak แบบจับต้องได้ พร้อมวัดผลจริงด้วย `runtime.NumGoroutine()` (ฟังก์ชันจากแพ็กเกจ `runtime` ที่คืนจำนวน goroutine ที่ยังทำงานอยู่ ณ ขณะนั้น — จะเรียนแพ็กเกจนี้เจาะลึกใน Part 055)

```go
package main

import (
	"fmt"
	"runtime"
	"time"
)

// leakyWorker คือตัวอย่าง "ตัวร้าย": รับงานจาก channel โดยไม่มีทางถูกสั่งให้เลิกงานเลย
// ถ้าไม่มีใครส่งงานเข้ามาอีก และไม่มีใครปิด channel นี้ goroutine นี้จะบล็อกอยู่ที่
// "case job := <-jobs" ตลอดไป — นี่คือ goroutine leak ที่พบบ่อยที่สุดในโค้ดจริง
func leakyWorker(jobs <-chan int) {
	for {
		job := <-jobs // ไม่มี select, ไม่มีทางออกฉุกเฉิน ถ้า jobs ไม่มีข้อมูลจะบล็อกตลอดไป
		fmt.Println("leaky worker ได้งาน:", job)
	}
}

func startLeakyWorkers(n int) {
	for i := 0; i < n; i++ {
		jobs := make(chan int) // แต่ละตัวมี channel ของตัวเอง ไม่มีใครส่งงานเข้าอีกเลยหลังจากนี้
		go leakyWorker(jobs)
	}
}

func main() {
	runtime.GC()
	before := runtime.NumGoroutine()
	fmt.Println("จำนวน goroutine ก่อนเริ่ม:", before)

	startLeakyWorkers(1000)

	// ให้เวลา scheduler เริ่ม goroutine ทั้งหมดให้เสร็จ
	time.Sleep(200 * time.Millisecond)

	runtime.GC()
	after := runtime.NumGoroutine()
	fmt.Println("จำนวน goroutine หลังเริ่ม 1000 leaky workers:", after)
	fmt.Println("goroutine ที่เพิ่มขึ้นและไม่มีวันจบ:", after-before)
}
```

ผลลัพธ์เมื่อรัน:

```
จำนวน goroutine ก่อนเริ่ม: 1
จำนวน goroutine หลังเริ่ม 1000 leaky workers: 1001
goroutine ที่เพิ่มขึ้นและไม่มีวันจบ: 1000
```

goroutine ทั้ง 1000 ตัวนี้จะค้างอยู่แบบนี้ **ตลอดไป** จนกว่าโปรแกรมทั้งตัวจะถูกปิด — ไม่มีทางเรียกคืนได้จากภายในโปรแกรมเองเลย เพราะไม่มีใครมีสิทธิ์เข้าถึง `jobs` channel ของแต่ละตัวอีกต่อไป (ตัวแปร `jobs` เป็น local variable ใน loop ที่จบไปแล้ว) นี่คือภาพที่ชัดเจนที่สุดของ goroutine leak: โค้ด **compile ผ่าน รันได้ ไม่ panic** แต่ทิ้งขยะที่กู้คืนไม่ได้ไว้เบื้องหลัง

---

## 5. แก้ปัญหาด้วย `select` + `ctx.Done()`

ทางแก้คือให้ทุก goroutine ที่อาจบล็อกตลอดไปได้ มี **ทางออกฉุกเฉิน** ผ่าน `select` ร่วมกับ `ctx.Done()` เสมอ

```go
// goodWorker รับ ctx เป็นพารามิเตอร์ตัวแรกเสมอ (ธรรมเนียมมาตรฐานของ Go)
// และใช้ select คู่กับ ctx.Done() เพื่อให้มี "ทางออก" เสมอ ไม่ว่า jobs จะมีงานส่งมาหรือไม่
func goodWorker(ctx context.Context, id int, jobs <-chan int) {
	for {
		select {
		case job, ok := <-jobs:
			if !ok {
				return // jobs ถูกปิดแล้ว ทำงานตามปกติจนหมดแล้วออก
			}
			fmt.Printf("worker %d ได้งาน: %d\n", id, job)
		case <-ctx.Done():
			return // ออกจาก loop ทันทีที่ ctx ถูกยกเลิกหรือ timeout — ไม่มี goroutine ค้าง
		}
	}
}
```

มาพิสูจน์ด้วยการวัดผลแบบเดียวกับหัวข้อ 4 อีกครั้ง แต่คราวนี้ใช้ `goodWorker`:

```go
package main

import (
	"context"
	"fmt"
	"runtime"
	"time"
)

func goodWorker(ctx context.Context, id int, jobs <-chan int) {
	for {
		select {
		case job, ok := <-jobs:
			if !ok {
				return
			}
			fmt.Printf("worker %d ได้งาน: %d\n", id, job)
		case <-ctx.Done():
			return
		}
	}
}

func runFanOutPool(ctx context.Context, numWorkers int, jobs <-chan int) {
	for i := 1; i <= numWorkers; i++ {
		go goodWorker(ctx, i, jobs)
	}
}

func main() {
	runtime.GC()
	before := runtime.NumGoroutine()
	fmt.Println("จำนวน goroutine ก่อนเริ่ม:", before)

	// context.WithTimeout ครอบคลุมการรัน pipeline ทั้งหมดไว้ในกำหนดเวลาเดียว
	ctx, cancel := context.WithTimeout(context.Background(), 150*time.Millisecond)
	defer cancel()

	jobs := make(chan int) // จงใจไม่ปิด jobs และไม่ส่งงานเพิ่ม เพื่อจำลองว่า "งานไม่มาแล้ว"
	runFanOutPool(ctx, 1000, jobs)

	// รอจน ctx timeout เอง (เทียบเท่ากับ pipeline ทำงานเกินเวลาที่กำหนด)
	<-ctx.Done()
	fmt.Println("ctx หมดเวลาแล้ว:", ctx.Err())

	// ให้เวลา worker ทุกตัว "สังเกตเห็น" ctx.Done() แล้วออกจาก select
	time.Sleep(200 * time.Millisecond)

	runtime.GC()
	after := runtime.NumGoroutine()
	fmt.Println("จำนวน goroutine หลัง ctx ถูกยกเลิกและรอให้ worker ออก:", after)
	fmt.Println("ต่างจากก่อนเริ่มแค่:", after-before, "(กลับสู่ baseline เพราะทุก worker มีทางออก)")
}
```

ผลลัพธ์ (ทดสอบด้วย `go run -race main.go` เพื่อยืนยันว่าไม่มี data race แถมด้วย):

```
จำนวน goroutine ก่อนเริ่ม: 1
ctx หมดเวลาแล้ว: context deadline exceeded
จำนวน goroutine หลัง ctx ถูกยกเลิกและรอให้ worker ออก: 1
ต่างจากก่อนเริ่มแค่: 0 (กลับสู่ baseline เพราะทุก worker มีทางออก)
```

เทียบกันชัดๆ:

| | leakyWorker (หัวข้อ 4) | goodWorker (หัวข้อ 5) |
|---|---|---|
| goroutine ก่อนเริ่ม | 1 | 1 |
| goroutine หลังเริ่ม 1000 ตัว | 1001 | 1001 (ชั่วคราว) |
| goroutine หลังสั่งยกเลิก + รอ | **ยังเป็น 1001 ตลอดไป** | **กลับเป็น 1 (baseline)** |

นี่คือหลักฐานที่จับต้องได้ว่าการเพิ่ม `select` คู่กับ `ctx.Done()` เพียงบรรทัดเดียว เปลี่ยนโปรแกรมจาก "รั่วอย่างถาวร" เป็น "คืนทรัพยากรได้ครบ 100%"

---

## 6. ตัวอย่างสมบูรณ์: Fan-out Worker Pool ที่หยุดทันทีเมื่อ ctx ถูกยกเลิก

ตัวอย่างในหัวข้อ 5 จงใจไม่ส่งงานเลยเพื่อให้เห็นภาพ leak ชัดๆ ทีนี้มาดูตัวอย่างที่สมจริงกว่า: fan-out worker pool ที่ **กำลังประมวลผลงานอยู่จริง** แต่ต้องหยุดกลางคันเมื่อ timeout มาถึง (สถานการณ์ที่พบบ่อยมากใน production เช่น request มี deadline ตายตัว)

```go
package main

import (
	"context"
	"fmt"
	"time"
)

// simulateWork จำลองงานที่ใช้เวลาสักครู่ แต่เคารพ ctx: ถ้าถูกยกเลิกระหว่างทำงาน
// จะคืนค่า error ทันทีแทนที่จะทำงานให้เสร็จก่อนแล้วค่อยเช็ค
func simulateWork(ctx context.Context, n int) (int, error) {
	select {
	case <-time.After(80 * time.Millisecond):
		return n * n, nil
	case <-ctx.Done():
		return 0, ctx.Err()
	}
}

// worker วนอ่านงานจาก jobs พร้อมเช็ค ctx.Done() ในทุกรอบของ select
// ข้อสังเกต: ctx เป็นพารามิเตอร์ตัวแรกเสมอ ตามธรรมเนียมของ Go
func worker(ctx context.Context, id int, jobs <-chan int, results chan<- int) {
	for {
		select {
		case n, ok := <-jobs:
			if !ok {
				return
			}
			result, err := simulateWork(ctx, n)
			if err != nil {
				return // ctx ถูกยกเลิกระหว่างทำงาน ออกจาก worker ทันที
			}
			select {
			case results <- result:
			case <-ctx.Done():
				return
			}
		case <-ctx.Done():
			fmt.Printf("worker %d: เห็น ctx.Done() แล้วเลิกงาน (%v)\n", id, ctx.Err())
			return
		}
	}
}

func main() {
	// ครอบทั้ง pipeline ด้วย timeout เดียว: ถ้าทำงานไม่เสร็จภายใน 300ms ให้ยกเลิกทั้งหมด
	ctx, cancel := context.WithTimeout(context.Background(), 300*time.Millisecond)
	defer cancel()

	jobs := make(chan int)
	results := make(chan int)

	const numWorkers = 4
	for i := 1; i <= numWorkers; i++ {
		go worker(ctx, i, jobs, results)
	}

	// producer: ป้อนงานเข้า jobs เรื่อยๆ (จำลองว่ามีงานเข้ามาไม่หยุดจากภายนอก เช่น message queue)
	go func() {
		defer close(jobs)
		for n := 1; n <= 100; n++ {
			select {
			case jobs <- n:
			case <-ctx.Done():
				return
			}
		}
	}()

	// consumer: อ่านผลลัพธ์จนกว่า ctx จะถูกยกเลิก
	count := 0
loop:
	for {
		select {
		case r, ok := <-results:
			if !ok {
				break loop
			}
			_ = r
			count++
		case <-ctx.Done():
			fmt.Println("main: ctx ยกเลิกแล้ว หยุดรอผลลัพธ์เพิ่ม:", ctx.Err())
			break loop
		}
	}

	fmt.Printf("ประมวลผลสำเร็จทั้งหมด %d งาน ก่อนถูกยกเลิก\n", count)
}
```

รันด้วย `go run -race main.go` (รันซ้ำ 3 ครั้งเพื่อยืนยันความสม่ำเสมอ):

```
main: ctx ยกเลิกแล้ว หยุดรอผลลัพธ์เพิ่ม: context deadline exceeded
ประมวลผลสำเร็จทั้งหมด 12 งาน ก่อนถูกยกเลิก
```

จาก 100 งานที่ป้อนเข้าไป มีเพียงประมาณ 12 งานที่ประมวลผลทันก่อน timeout 300ms (คำนวณคร่าวๆ: 4 worker × งานละ 80ms × ~4 รอบ = 12-16 งาน ซึ่งตรงกับที่วัดได้จริง) โปรแกรมไม่ค้างรอให้ 100 งานเสร็จครบ และไม่มี goroutine ตกค้าง เพราะทุกจุดที่อาจบล็อก (`jobs <- n`, `results <- result`, การรอ `simulateWork`) ล้วนมี `select` คู่กับ `ctx.Done()` กำกับไว้หมด

จุดที่ควรสังเกตเป็นพิเศษ: **ใน `worker`, เรามี `select` ซ้อนกันสองชั้น** — ชั้นนอกเลือกระหว่างรับงานใหม่กับยกเลิก, ชั้นในเลือกระหว่างส่งผลลัพธ์กับยกเลิก นี่คือรูปแบบที่จำเป็นเมื่อ goroutine หนึ่งตัวมีจุดบล็อกได้มากกว่าหนึ่งจุด (ทั้งรับและส่ง) — **ทุกจุดบล็อกต้องมี `ctx.Done()` กำกับเสมอ ไม่ใช่แค่จุดใดจุดหนึ่ง**

---

## 7. `context.WithTimeout` ครอบทั้ง Pipeline

สังเกตในทุกตัวอย่างข้างต้นว่าเราสร้าง `ctx` เพียง**ครั้งเดียว**ที่จุดเริ่มต้น (`main()`) แล้วส่งต่อ `ctx` ตัวเดียวกันนี้ผ่านทุก goroutine ในระบบ:

```go
ctx, cancel := context.WithTimeout(context.Background(), 300*time.Millisecond)
defer cancel()

go worker(ctx, 1, jobs, results)
go worker(ctx, 2, jobs, results)
go worker(ctx, 3, jobs, results)
go worker(ctx, 4, jobs, results)
go producer(ctx, jobs)
```

นี่คือแนวคิดที่เรียกว่า **"budget เดียวสำหรับทั้งการทำงาน"**: ไม่ว่า pipeline จะมีกี่ stage กี่ worker ก็ตาม ทุกส่วนใช้ deadline เดียวกันร่วมกัน เมื่อ timeout มาถึง `ctx.Done()` จะถูกปิด และ **ทุก** goroutine ที่ถืออ้างอิงถึง `ctx` ตัวนี้จะได้รับสัญญาณพร้อมกันในทันที (ไม่ต้องส่ง signal ไปทีละตัว) — channel ที่ถูกปิดแล้วส่งค่า zero value กลับให้ผู้รับทุกตัวที่ `<-ch` พร้อมกันเสมอ นี่คือคุณสมบัติของ channel ที่ Part 037 อธิบายไว้ และเป็นเหตุผลที่ `context` เลือกใช้ channel เป็นกลไกหลักในการกระจายสัญญาณยกเลิก

ข้อดีของการมี timeout เดียวครอบทั้ง pipeline แทนที่จะตั้ง timeout แยกในแต่ละ stage:

- **คาดการณ์ได้**: รู้แน่ชัดว่าการทำงานทั้งหมด (ตั้งแต่ต้นจนจบ) จะใช้เวลาไม่เกินเท่าไร ไม่ว่า pipeline จะมีกี่ stage
- **ไม่ต้องคำนวณ timeout ย่อยเอง**: ถ้าตั้ง timeout แยกในแต่ละ stage (เช่น stage ละ 100ms, 4 stage = 400ms) จะคำนวณผิดพลาดง่ายและไม่สะท้อนเวลาที่ใช้จริงของ caller
- **ยกเลิกพร้อมกันทุกที่**: ไม่มีความเสี่ยงที่ stage หนึ่งจะยกเลิกไปแล้วแต่อีก stage ยังทำงานต่อ (เพราะลืมเชื่อม cancellation ระหว่างกัน)

---

## 8. การส่งต่อ (propagate) ctx เดียวผ่าน call tree ทั้งหมด

ในระบบจริงที่ซับซ้อนกว่าตัวอย่างในบทนี้ `ctx` มักถูกส่งผ่านหลายชั้นของฟังก์ชัน ไม่ใช่แค่ระดับเดียว ลองนึกภาพ call tree ของ HTTP request หนึ่งตัว (จะเรียนเจาะลึกใน Part 047):

```
HTTP Handler (สร้าง ctx จาก request, มักมี deadline ติดมาแล้ว)
    │
    ├── ctx ──▶ fetchUserFromDB(ctx, userID)
    │               │
    │               └── ctx ──▶ queryRow(ctx, sql)
    │
    ├── ctx ──▶ fetchOrdersFromAPI(ctx, userID)
    │               │
    │               └── ctx ──▶ httpClient.Do(req.WithContext(ctx))
    │
    └── ctx ──▶ runFanOutPool(ctx, numWorkers, jobs)   ← เหมือนตัวอย่างหัวข้อ 6
                    │
                    └── ctx ──▶ worker(ctx, id, jobs, results) × N
```

หลักการสำคัญ: **`ctx` ตัวเดียวกันไหลผ่านทุกชั้นโดยไม่มีการสร้าง `context.Background()` ใหม่กลางทาง** ถ้า handler ด้านบนสุดถูกยกเลิก (เช่น ผู้ใช้ปิด browser, หรือ client ตัดการเชื่อมต่อ) สัญญาณนั้นจะไหลลงไปถึงทุก goroutine ที่ลึกที่สุดในระบบโดยอัตโนมัติ — ไม่ต้องเขียนกลไกแจ้งเตือนเองที่ทุกชั้น

**ข้อผิดพลาดที่ร้ายแรงที่สุด** ที่ทำลายห่วงโซ่นี้คือการสร้าง `context.Background()` ใหม่กลางทางโดยไม่จำเป็น:

```go
// ผิดมาก: ตัด cancellation chain ทิ้งกลางทาง
func fetchOrdersFromAPI(ctx context.Context, userID string) ([]Order, error) {
	// สร้าง ctx ใหม่จาก Background() แทนที่จะใช้ ctx ที่รับมา
	// ผลคือ: ถ้า caller ยกเลิก ctx เดิม ฟังก์ชันนี้จะไม่มีวันรู้เลย
	newCtx := context.Background()
	req, _ := http.NewRequestWithContext(newCtx, "GET", url, nil)
	return httpClient.Do(req)
}
```

```go
// ถูกต้อง: ใช้ ctx ที่รับมาต่อเนื่อง (จะเพิ่ม WithTimeout ทับได้ถ้าต้องการ timeout ย่อยเพิ่มเติม)
func fetchOrdersFromAPI(ctx context.Context, userID string) ([]Order, error) {
	req, _ := http.NewRequestWithContext(ctx, "GET", url, nil)
	return httpClient.Do(req)
}
```

กฎที่จำง่ายที่สุด: **`context.Background()` ควรถูกเรียกแค่จุดเดียวในทั้งโปรแกรม (root ของ call tree เช่นใน `main()` หรือจุดเริ่มต้นของ HTTP handler) ส่วนที่เหลือทั้งหมดควรรับ `ctx` มาจาก parameter แล้วส่งต่อ หรือห่อด้วย `context.With...` เพื่อสร้าง child context เท่านั้น**

---

## 9. ข้อควรระวังและ anti-pattern ของการใช้ context

### อย่าเก็บ `ctx` ไว้ใน struct field

```go
// anti-pattern: เก็บ ctx ไว้เป็น field ของ struct
type Service struct {
	ctx context.Context // ไม่ควรทำแบบนี้
}
```

`context` ถูกออกแบบให้เป็นค่าที่ไหลผ่าน call chain แบบ explicit ผ่าน parameter ไม่ใช่ state ที่เก็บไว้ในโครงสร้างข้อมูลระยะยาว การเก็บ `ctx` ไว้ใน struct ทำให้ไม่ชัดเจนว่า ctx ตัวไหนกำลังถูกใช้งานอยู่ ณ ขณะเรียกแต่ละเมธอด และเสี่ยงต่อการใช้ ctx ที่หมดอายุไปแล้วโดยไม่รู้ตัว (มีข้อยกเว้นน้อยมากที่เอกสารทางการยอมรับ เช่นกรณี struct ที่ implement `http.Handler` ระยะสั้น แต่โดยทั่วไปให้หลีกเลี่ยง)

### อย่าส่ง `nil` แทน context

```go
// ผิด
worker(nil, 1, jobs, results)

// ถูก — ถ้ายังไม่รู้ context จริง ใช้ context.TODO() หรือ context.Background()
worker(context.TODO(), 1, jobs, results)
```

`context.TODO()` มีไว้สื่อความหมายว่า "ยังไม่ชัดเจนว่าควรใช้ context ตัวไหน แต่จะแก้ทีหลัง" ต่างจาก `context.Background()` ที่สื่อว่า "นี่คือจุดเริ่มต้นของ call tree จริงๆ (root)"

### อย่าใช้ `context.Value` เป็นช่องทางส่ง parameter ทั่วไป

`context.Value` ควรใช้เก็บเฉพาะข้อมูลที่ "ผูกกับ request/operation" จริงๆ เช่น request ID, trace ID, ข้อมูล authentication ที่ผ่านมาแล้ว **ไม่ควร**ใช้แทนการส่ง parameter ปกติของฟังก์ชัน เพราะจะทำให้ compiler ตรวจสอบ type ให้ไม่ได้ และโค้ดอ่านยากขึ้นมาก (รายละเอียดเรื่องนี้อยู่ใน Part 032)

### ตรวจสอบ `ctx.Err()` ก่อนเริ่มงานหนัก ไม่ใช่แค่ตอนจบ

```go
func heavyWork(ctx context.Context, data []int) (int, error) {
	if ctx.Err() != nil {
		return 0, ctx.Err() // เช็คก่อนเริ่มเลย ไม่เสียเวลาทำงานที่จะถูกทิ้งอยู่ดี
	}
	// ... ทำงานหนัก ...
}
```

ถ้า `ctx` ถูกยกเลิกไปแล้วตั้งแต่ก่อนเรียกฟังก์ชัน การเช็ค `ctx.Err() != nil` ก่อนเริ่มทำงานหนักช่วยประหยัดทรัพยากรได้ทันที แทนที่จะปล่อยให้ทำงานเสร็จก่อนแล้วค่อยพบว่าผลลัพธ์ไม่มีความหมายแล้ว

---

## สรุปสิ่งที่ได้เรียนในบทนี้

- `context.Context` ในบริบท concurrency คือกลไกกระจายสัญญาณยกเลิกไปยัง goroutine จำนวนมากพร้อมกัน ผ่านคุณสมบัติของ channel ที่ปิดแล้วจะส่งค่าให้ผู้รับทุกตัวทันที
- ธรรมเนียมมาตรฐาน: `ctx` ต้องเป็นพารามิเตอร์ตัวแรกของฟังก์ชันเสมอ
- **Goroutine leak** คือ goroutine ที่บล็อกตลอดไปโดยไม่มีทางออก อันตรายเพราะไม่ panic ไม่ crash แต่กินหน่วยความจำเพิ่มขึ้นเรื่อยๆ อย่างเงียบๆ
- พิสูจน์ได้ด้วย `runtime.NumGoroutine()` ก่อน/หลัง: โค้ดที่ไม่มี `select` + `ctx.Done()` จะทิ้ง goroutine ค้างตลอดไป ส่วนโค้ดที่มีจะกลับสู่ baseline หลังยกเลิก
- ทุกจุดในโค้ดที่อาจบล็อก (`<-ch`, `ch <- v`) ต้องมี `select` คู่กับ `ctx.Done()` กำกับเสมอ ถ้ามีมากกว่าหนึ่งจุดในฟังก์ชันเดียว ต้องกำกับให้ครบทุกจุด
- `context.WithTimeout` ที่ครอบทั้ง pipeline ให้ "budget เวลาเดียว" แก่ทุก stage ดีกว่าการตั้ง timeout แยกในแต่ละ stage
- `ctx` ต้องไหลจาก root (`context.Background()` ที่เดียวในโปรแกรม) ผ่าน call tree ทั้งหมดโดยไม่สร้าง context ใหม่กลางทางโดยไม่จำเป็น
- Anti-pattern ที่ต้องหลีกเลี่ยง: เก็บ `ctx` ใน struct field, ส่ง `nil` แทน context, ใช้ `context.Value` แทน parameter ทั่วไป, ไม่เช็ค `ctx.Err()` ก่อนเริ่มงานหนัก

## แบบฝึกหัดท้ายบท

1. รันตัวอย่างในหัวข้อ 4 (leaky) แล้วใช้ `go tool pprof` หรือเพิ่ม endpoint `net/http/pprof` (จะเรียนละเอียดใน Part 083) เพื่อดู stack trace ของ goroutine ที่ค้างอยู่จริงบนเครื่องของคุณ
2. ดัดแปลงตัวอย่างในหัวข้อ 6 ให้ใช้ `context.WithCancel` แทน `context.WithTimeout` แล้วเรียก `cancel()` เองจาก goroutine อีกตัวหนึ่งหลังผ่านไป 3 งาน (จำลองการยกเลิกโดยเงื่อนไข ไม่ใช่เวลา)
3. เขียนฟังก์ชัน `heavyWork` ที่รับ `ctx` แล้วเช็ค `ctx.Err()` ทั้งก่อนเริ่มทำงานและระหว่างทำงาน (ผ่าน `select`) ทดสอบว่าทำงานถูกต้องทั้งสองกรณี: ctx ถูกยกเลิกก่อนเริ่ม กับ ctx ถูกยกเลิกระหว่างทำงาน
4. ทดลองสร้างสถานการณ์ anti-pattern ในหัวข้อ 9 (สร้าง `context.Background()` ใหม่กลางทาง) แล้วพิสูจน์ด้วยโค้ดว่าเมื่อ ctx ต้นทางถูกยกเลิก ฟังก์ชันที่มี anti-pattern นี้จะไม่ได้รับสัญญาณยกเลิกเลย
5. เขียน table-driven test (ทบทวนจาก Part 034) สำหรับฟังก์ชัน `worker` ในหัวข้อ 6 โดยทดสอบทั้งกรณี ctx ยกเลิกก่อนมีงาน, ctx ยกเลิกระหว่างมีงาน, และงานเสร็จหมดตามปกติโดย ctx ไม่ถูกยกเลิกเลย
6. อธิบายด้วยคำพูดของตัวเองว่าทำไม `context.Context` ถึง "ดีกว่า" done channel มือเปล่าที่เขียนใน Part 042 ในแง่ของ engineering (ความสะดวก, ความปลอดภัย, การยอมรับจาก ecosystem)

---

**ต่อไป**: [Part 044 — Race Condition และ Race Detector](./044-race-conditions.md)
