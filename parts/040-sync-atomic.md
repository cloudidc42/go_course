# Part 040: `sync/atomic` และ Lock-Free Programming

> ภาคที่ 3: การทำงานพร้อมกัน (Concurrency) — ตอนที่ 5 จาก 10 (Part 36–45)

## สารบัญของบทนี้

1. ทำไมต้องมี atomic operation ทั้งที่มี mutex อยู่แล้ว
2. Typed API สมัยใหม่ (Go 1.19+): `atomic.Int64`, `atomic.Bool`, `atomic.Value`
3. `atomic.Pointer[T]`: เปลี่ยนข้อมูลทั้งก้อนแบบ atomic
4. API แบบเก่า (function-based) เทียบกับ typed API
5. Benchmark: เปรียบเทียบ Mutex กับ Atomic
6. เมื่อ atomic ยังไม่พอ: Check-Then-Act ยังคง Race ได้
7. Memory Model เบื้องต้น: Happens-Before
8. สรุปสิ่งที่ได้เรียนในบทนี้
9. แบบฝึกหัดท้ายบท

---

## 1. ทำไมต้องมี atomic operation ทั้งที่มี mutex อยู่แล้ว

ใน Part 039 เราเรียน `sync.Mutex` ไปแล้วว่ามันป้องกัน critical section ได้อย่างสมบูรณ์ คำถามคือ ถ้ามี mutex ที่ทำงานถูกต้องอยู่แล้ว ทำไม Go ยังต้องมีแพ็กเกจ **`sync/atomic`** แยกออกมาอีก?

คำตอบคือเรื่อง **ต้นทุน (overhead)** การใช้ `Mutex` แม้จะเป็นวิธีที่ถูกต้องและใช้งานได้ทั่วไป แต่ก็มีค่าใช้จ่ายแฝงอยู่เสมอ: การ `Lock()` ต้องเช็คว่ามีใครถือ lock อยู่หรือไม่ ถ้ามีต้อง block goroutine ปัจจุบันและอาจต้องให้ scheduler เข้ามาช่วยจัดการ (ถ้ามีการแย่งชิงกันสูง) เมื่อสิ่งที่ต้องป้องกันเป็นแค่ **ตัวแปรเดี่ยวๆ ตัวเดียว** (เช่น ตัวนับ หรือ flag boolean) การเรียก `Lock()`/`Unlock()` ทุกครั้งเพื่อป้องกันแค่การบวกเลขครั้งเดียวถือว่าเป็นการ "ใช้ปืนใหญ่ยิงนก" — เกินความจำเป็นไปมาก

**Atomic operation** คือคำสั่งระดับ CPU ที่รับประกันว่าการอ่าน-แก้ไข-เขียนตัวแปรตัวหนึ่งจะเกิดขึ้นเป็น**หน่วยเดียวที่แบ่งแยกไม่ได้ (indivisible)** โดย**ไม่ต้องพึ่งกลไก lock ระดับ software เลย** — ใช้ความสามารถพิเศษของฮาร์ดแวร์ CPU โดยตรง (เช่นคำสั่ง `LOCK XADD` หรือ `CMPXCHG` บนสถาปัตยกรรม x86) ทำให้เร็วกว่า mutex มากในสถานการณ์ที่เหมาะสม — จะเห็นตัวเลขเปรียบเทียบจริงในหัวข้อ 4

> **หลักการสำคัญที่ต้องจำ**: `sync/atomic` เหมาะสำหรับ**การกระทำเดี่ยวๆ ง่ายๆ** บนตัวแปรตัวเดียว (บวก, อ่าน, เขียน, เปรียบเทียบแล้วสลับ) เท่านั้น — ถ้าต้อง**ป้องกันข้อมูลหลายชิ้นให้สอดคล้องกัน (consistency ข้าม field หลายตัว)** หรือทำ**หลายขั้นตอนที่ต้องเป็นหน่วยเดียวกัน** `sync.Mutex` ยังคงเป็นเครื่องมือที่ถูกต้องกว่าเสมอ (จะเห็นตัวอย่างที่ atomic เพียงอย่างเดียวไม่พอในหัวข้อ 5)

---

## 2. Typed API สมัยใหม่ (Go 1.19+): `atomic.Int64`, `atomic.Bool`, `atomic.Value`

ตั้งแต่ **Go 1.19** เป็นต้นมา แพ็กเกจ `sync/atomic` มี **typed API** ที่ใช้งานง่ายและปลอดภัยกว่าของเดิมมาก โดยมี type สำเร็จรูปให้เลือกใช้ตรงกับชนิดข้อมูลที่ต้องการ:

| Type | ใช้แทน |
|---|---|
| `atomic.Int32` / `atomic.Int64` | จำนวนเต็มมีเครื่องหมาย |
| `atomic.Uint32` / `atomic.Uint64` | จำนวนเต็มไม่มีเครื่องหมาย |
| `atomic.Bool` | ค่า boolean |
| `atomic.Value` | ค่าชนิดใดก็ได้ (`any`) — เก็บและอ่านค่าทั้งก้อนแบบ atomic |
| `atomic.Pointer[T]` | pointer ชนิด generic (Go 1.19+ ใช้ generics ร่วมกับ atomic ได้) |

แต่ละ type มี method หลักที่ใช้บ่อยคล้ายกัน: `Load()` (อ่านค่า), `Store(v)` (เขียนค่า), `Add(delta)` (บวกค่า — เฉพาะ type ตัวเลข), และ `CompareAndSwap(old, new)` (เปลี่ยนค่าเฉพาะเมื่อค่าปัจจุบันตรงกับที่คาดไว้)

```go
package main

import (
	"fmt"
	"sync"
	"sync/atomic"
)

func main() {
	var counter atomic.Int64 // ชนิดข้อมูล atomic ที่มาพร้อม type-safety ตั้งแต่ Go 1.19

	var wg sync.WaitGroup
	for i := 0; i < 1000; i++ {
		wg.Add(1)
		go func() {
			defer wg.Done()
			counter.Add(1) // อะตอมิก: ไม่ต้องใช้ mutex เพราะเป็น operation เดียวที่ CPU รับประกัน atomicity ให้
		}()
	}
	wg.Wait()

	fmt.Println("counter:", counter.Load()) // ได้ 1000 เสมอ

	var flag atomic.Bool
	flag.Store(true)
	fmt.Println("flag:", flag.Load())

	swapped := flag.CompareAndSwap(true, false) // ถ้าค่าปัจจุบันตรงกับที่คาดไว้ (true) ให้เปลี่ยนเป็น false
	fmt.Println("swapped:", swapped, "ค่าใหม่:", flag.Load())

	swapped2 := flag.CompareAndSwap(true, false) // ตอนนี้ค่าจริงคือ false แล้ว ไม่ตรงกับที่คาดไว้ (true) จึงไม่เปลี่ยน
	fmt.Println("swapped2:", swapped2, "ค่ายังเป็น:", flag.Load())
}
```

ผลลัพธ์:

```
counter: 1000
flag: true
swapped: true ค่าใหม่: false
swapped2: false ค่ายังเป็น: false
```

`CompareAndSwap` (มักเรียกย่อว่า **CAS**) เป็นหัวใจของการเขียนโปรแกรมแบบ **lock-free**: มันทำ 2 อย่างพร้อมกันเป็นหน่วยเดียวที่แบ่งแยกไม่ได้ — **"เปรียบเทียบว่าค่าปัจจุบันตรงกับที่คาดไว้หรือไม่ ถ้าตรง ค่อยเปลี่ยนเป็นค่าใหม่"** ถ้าค่าปัจจุบันไม่ตรงกับที่คาดไว้ (เพราะมี goroutine อื่นมาเปลี่ยนไปก่อนแล้ว) การสลับจะ**ล้มเหลว**และคืนค่า `false` โดยไม่มีการเปลี่ยนแปลงใดๆ เกิดขึ้น — เดี๋ยวหัวข้อ 5 จะแสดงประโยชน์ของ CAS ในการแก้ปัญหา race ที่ atomic ธรรมดาแก้ไม่ได้

---

## 3. `atomic.Pointer[T]`: เปลี่ยนข้อมูลทั้งก้อนแบบ atomic

`atomic.Value` ในหัวข้อก่อนหน้าเก็บค่าชนิดใดก็ได้ (`any`) แต่ตั้งแต่ Go 1.19 มี **`atomic.Pointer[T]`** ซึ่งใช้ **generics** (Part 028-029) ร่วมกับ atomic ทำให้ได้ pointer ที่ type-safe เต็มรูปแบบ ไม่ต้องทำ type assertion ตอนอ่านค่ากลับแบบ `atomic.Value` — เหมาะมากสำหรับสถานการณ์ที่ต้องการ "สลับข้อมูลทั้งก้อนใหม่" แบบ atomic โดยไม่ต้องล็อกเพื่ออ่าน

```go
package main

import (
	"fmt"
	"sync"
	"sync/atomic"
)

type Config struct {
	MaxRetries int
	Timeout    int
}

func main() {
	var current atomic.Pointer[Config] // atomic.Pointer[T] เก็บ pointer ไปยัง struct ได้แบบ type-safe (generics + atomic, Go 1.19+)
	current.Store(&Config{MaxRetries: 3, Timeout: 30})

	var wg sync.WaitGroup
	// นักอ่านหลายคนอ่านค่า config พร้อมกัน ขณะที่มีคนหนึ่งกำลังจะเปลี่ยน config ทั้งก้อนใหม่
	for i := 0; i < 5; i++ {
		wg.Add(1)
		go func() {
			defer wg.Done()
			cfg := current.Load() // อ่าน pointer ปัจจุบันแบบ atomic -- ได้ config ทั้งก้อนที่สอดคล้องกันเสมอ ไม่มีทางเห็น field ผสมจากคนละเวอร์ชัน
			_ = cfg.MaxRetries
			_ = cfg.Timeout
		}()
	}

	// เปลี่ยน config ทั้งก้อนในครั้งเดียวแบบ atomic -- ไม่มีใครเห็นสถานะครึ่งๆ กลางๆ ระหว่างเปลี่ยน
	current.Store(&Config{MaxRetries: 5, Timeout: 60})
	wg.Wait()

	final := current.Load()
	fmt.Println("MaxRetries:", final.MaxRetries, "Timeout:", final.Timeout)
}
```

ผลลัพธ์ (ผ่าน `-race` สะอาดด้วย):

```
MaxRetries: 5 Timeout: 60
```

จุดสำคัญของแพทเทิร์นนี้คือ **การเขียน (`Store`) แทนที่ pointer ทั้งก้อนไปเลย แทนที่จะแก้ field ภายใน struct เดิมทีละตัว** — ถ้าเผลอทำ `current.Load().MaxRetries = 5` ตรงๆ (แก้ field ของ struct ที่ pointer ชี้ไปอยู่) จะกลายเป็นการเขียนข้อมูลที่ reader ตัวอื่นอาจกำลังอ่านอยู่พอดี ซึ่งไม่ปลอดภัยและไม่ใช่ atomic operation อีกต่อไป หลักการที่ถูกต้องคือ **สร้าง struct ใหม่ทั้งก้อนเสมอ แล้วค่อย `Store` pointer ของก้อนใหม่เข้าไปแทนที่** วิธีนี้รับประกันว่า reader ที่ `Load()` มาในเวลาใดก็ตาม จะได้ struct ที่ **สมบูรณ์และสอดคล้องกันทุก field เสมอ** ไม่มีทางเห็นสถานะครึ่งๆ กลางๆ ที่ field หนึ่งเป็นค่าเก่าแต่อีก field เป็นค่าใหม่ปะปนกัน — รูปแบบนี้พบได้บ่อยมากสำหรับการทำ **hot-reload configuration** ในระบบที่รันตลอดเวลาและไม่อยากหยุดเพื่อโหลด config ใหม่

---

## 4. API แบบเก่า (function-based) เทียบกับ typed API

ก่อน Go 1.19 แพ็กเกจ `sync/atomic` มีแค่ชุดฟังก์ชันแบบเก่าที่ทำงานบนตัวแปรพื้นฐานโดยตรงผ่าน pointer ยังพบเห็นได้ในโค้ดเก่าจำนวนมาก จึงควรรู้จักไว้แม้จะไม่แนะนำให้ใช้ในโค้ดใหม่แล้ว:

```go
package main

import (
	"fmt"
	"sync"
	"sync/atomic"
)

func main() {
	var counter int64 // ตัวแปร int64 ธรรมดา ไม่มี type-safety พิเศษ

	var wg sync.WaitGroup
	for i := 0; i < 1000; i++ {
		wg.Add(1)
		go func() {
			defer wg.Done()
			atomic.AddInt64(&counter, 1) // ต้องส่ง pointer ไปเอง และเรียก function แยกสำหรับแต่ละ type (AddInt32, AddInt64, ...)
		}()
	}
	wg.Wait()

	fmt.Println("counter (แบบเก่า):", atomic.LoadInt64(&counter))
	// ข้อเสียของ API แบบเก่า: ไม่มีอะไรห้ามไม่ให้ใครเผลอเขียน counter++ ตรงๆ (ไม่ผ่าน atomic) ปะปนกับโค้ดที่ใช้ atomic
	// ส่วน atomic.Int64 ในไฟล์ก่อนหน้าบังคับ (โดย convention) ให้เข้าถึงผ่าน method เท่านั้น อ่านง่ายและปลอดภัยกว่า
}
```

ผลลัพธ์:

```
counter (แบบเก่า): 1000
```

เปรียบเทียบข้อแตกต่างของทั้งสอง API:

| | API แบบเก่า (function-based) | API สมัยใหม่ (typed, Go 1.19+) |
|---|---|---|
| Syntax | `atomic.AddInt64(&counter, 1)` | `counter.Add(1)` |
| ต้องส่ง pointer เอง | ใช่ (`&counter`) เสี่ยงต่อการส่งผิดตัว | ไม่ต้อง (method อยู่ติดกับตัวแปรโดยตรง) |
| ป้องกันการเข้าถึงแบบไม่ atomic โดยไม่ตั้งใจ | ไม่ป้องกัน — ตัวแปรเป็น `int64` ธรรมดา ใครจะเขียน `counter++` ตรงๆ ก็ได้ (compiler ไม่เตือน) | ป้องกันได้ดีกว่า — ฟิลด์ภายในของ `atomic.Int64` ถูกซ่อนไว้ ต้องผ่าน method เท่านั้น |
| อ่านง่าย | ต้องจำชื่อฟังก์ชันแยกตาม type (`AddInt32`, `AddInt64`, `AddUint32`, ...) | ชื่อ method เหมือนกันทุก type (`Add`, `Load`, `Store`, `CompareAndSwap`) |

ด้วยเหตุผลเหล่านี้ **บทเรียนนี้และแนวทางปัจจุบันของ Go เอง แนะนำให้ใช้ typed API (`atomic.Int64`, `atomic.Bool`, ฯลฯ) เป็นค่าเริ่มต้นเสมอสำหรับโค้ดใหม่** — API แบบเก่ายังใช้งานได้และยังไม่ถูก deprecate (Go รักษาความเข้ากันได้ย้อนหลังตาม Go 1 Compatibility Promise ที่เรียนใน Part 001) แต่ควรรู้จักไว้เพื่ออ่านโค้ดเก่าเป็นหลัก ไม่ใช่เพื่อเขียนโค้ดใหม่

---

## 4. Benchmark: เปรียบเทียบ Mutex กับ Atomic

คำกล่าวอ้างที่ว่า "atomic เร็วกว่า mutex" ควรได้รับการพิสูจน์ด้วยตัวเลขจริง ไม่ใช่แค่เชื่อตามคำบอกเล่า — นำหลักการเขียน benchmark จาก **Part 034** มาใช้วัดผลจริง โดยใช้ `b.RunParallel` เพื่อจำลองสถานการณ์ที่หลาย goroutine (เท่ากับจำนวน `GOMAXPROCS`) แข่งกันเข้าถึงตัวนับพร้อมกันจริงๆ:

```go
package main

import (
	"sync"
	"sync/atomic"
	"testing"
)

// ใช้ b.RunParallel เพื่อวัดความเร็วของ mutex กับ atomic ในสถานการณ์ที่หลาย goroutine
// (เท่ากับจำนวน GOMAXPROCS) แข่งกันเข้าถึงตัวนับพร้อมกันจริงๆ โดยไม่ให้ overhead
// ของการสร้าง goroutine ใหม่ทุกครั้งมาบดบังผลต่างของ synchronization primitive เอง

func BenchmarkMutexCounter(b *testing.B) {
	var mu sync.Mutex
	var counter int64

	b.RunParallel(func(pb *testing.PB) {
		for pb.Next() {
			mu.Lock()
			counter++
			mu.Unlock()
		}
	})
}

func BenchmarkAtomicCounter(b *testing.B) {
	var counter atomic.Int64

	b.RunParallel(func(pb *testing.PB) {
		for pb.Next() {
			counter.Add(1)
		}
	})
}
```

รันด้วยคำสั่ง:

```bash
go test -bench=. -run=^$
```

ผลลัพธ์ (ตัวเลขจริงขึ้นกับเครื่องที่รัน ตัวอย่างนี้รันบนเครื่อง 4 core):

```
goos: linux
goarch: amd64
cpu: Intel(R) Xeon(R) Processor @ 2.80GHz
BenchmarkMutexCounter-4    	 2000000	       102.6 ns/op
BenchmarkAtomicCounter-4   	 2000000	        17.95 ns/op
PASS
```

ผลลัพธ์ยืนยันชัดเจน: **`atomic.Int64.Add()` เร็วกว่า `sync.Mutex` ประมาณ 5-6 เท่า** ในสถานการณ์นี้ (17.95 ns/op เทียบกับ 102.6 ns/op) เหตุผลคือ mutex ต้องผ่านกลไกการจัดการ lock ที่ซับซ้อนกว่า (การเช็คสถานะ, การจัดคิว goroutine ที่รออยู่เมื่อเกิดการแย่งชิงสูง) ในขณะที่ atomic operation ใช้คำสั่งฮาร์ดแวร์โดยตรงแค่คำสั่งเดียว

> **ข้อควรระวังในการอ่านผล benchmark**: ตัวเลขนี้เป็นเพียง**ตัวอย่างประกอบการสอน**บนเครื่องหนึ่งเครื่องเท่านั้น ผลจริงจะแตกต่างกันไปตามสถาปัตยกรรม CPU, จำนวน core, ระดับการแย่งชิง (contention) และเวอร์ชัน Go — หลักการสำคัญที่ควรจำไม่ใช่ตัวเลขที่แน่นอน แต่คือ **แนวโน้ม**: atomic มักเร็วกว่า mutex อย่างมีนัยสำคัญสำหรับ operation ง่ายๆ บนตัวแปรเดี่ยว และควร**วัดผลจริงในระบบของคุณเอง**เสมอก่อนตัดสินใจ optimize (ตามหลักการ "premature optimization" ที่ควรหลีกเลี่ยง — เรียนเพิ่มเติมได้ใน Part 083-085)

---

## 5. เมื่อ atomic ยังไม่พอ: Check-Then-Act ยังคง Race ได้

นี่คือกับดักที่แม้แต่โปรแกรมเมอร์ที่มีประสบการณ์ก็พลาดได้: **การใช้ atomic type ไม่ได้แปลว่าโค้ดทั้งหมดที่ใช้ตัวแปรนั้นจะปลอดภัยจาก race โดยอัตโนมัติ** ถ้าโค้ดนั้นทำ **หลาย atomic operation ติดกัน** โดยมี logic คั่นกลาง (เช่น "เช็คค่าก่อน แล้วค่อยตัดสินใจทำอะไรต่อ" หรือที่เรียกว่า **check-then-act**) ก็ยังเกิด race ได้เหมือนเดิม เพราะแม้แต่ละ operation เดี่ยวๆ จะเป็น atomic แต่ **การรวมกันของหลาย operation ไม่ได้เป็น atomic ไปด้วย**

ตัวอย่างคลาสสิก: ระบบขายตั๋วที่มีจำนวนจำกัด

```go
package main

import (
	"fmt"
	"runtime"
	"sync"
	"sync/atomic"
)

const maxTickets = 100

// buyTicketWrong ดูเผินๆ เหมือนปลอดภัยเพราะใช้ atomic.Int64 แต่จริงๆ ยังมี race
// เพราะ "เช็คแล้วค่อยทำ" (check-then-act) ประกอบด้วย 2 operation แยกกัน (Load แล้ว Add)
// ระหว่างกลางอาจมี goroutine อื่นแทรกเข้ามา Add ไปแล้วก็ได้
// (runtime.Gosched() ด้านล่างใส่ไว้เพื่อ "บังคับ" ให้ scheduler สลับไปรัน goroutine อื่นตรงรอยต่อ
// พอดี ซึ่งขยาย race window ให้เห็นผลชัดเจนในการสาธิต -- ในโค้ดจริงบั๊กแบบนี้เกิดแบบสุ่มโดยไม่ต้องมีใครช่วย)
func buyTicketWrong(sold *atomic.Int64) bool {
	if sold.Load() < maxTickets { // operation ที่ 1: อ่านค่า
		runtime.Gosched() // ให้โอกาส goroutine อื่นแทรกคั่นกลางระหว่าง Load กับ Add
		sold.Add(1)       // operation ที่ 2: บวกค่า -- ระหว่างนี้ goroutine อื่นอาจแทรกมาบวกไปแล้วก็ได้
		return true
	}
	return false
}

// buyTicketRight ใช้ CompareAndSwap วนลูปเพื่อทำ "เช็คและทำ" แบบ atomic รวมเป็น operation เดียวจริงๆ
func buyTicketRight(sold *atomic.Int64) bool {
	for {
		current := sold.Load()
		if current >= maxTickets {
			return false
		}
		if sold.CompareAndSwap(current, current+1) {
			// สำเร็จ: ไม่มีใครมาเปลี่ยนค่าระหว่างที่เรา Load กับตอนที่เรา CompareAndSwap
			return true
		}
		// ถ้า CompareAndSwap ล้มเหลว แปลว่ามี goroutine อื่นแทรกมาเปลี่ยนค่าไปแล้ว -- วนกลับไป Load ค่าล่าสุดใหม่แล้วลองอีกครั้ง
	}
}

func run(buy func(*atomic.Int64) bool) int64 {
	var sold atomic.Int64
	var wg sync.WaitGroup
	var successCount atomic.Int64

	for i := 0; i < 500; i++ { // จำลองคน 500 คนแย่งกันซื้อตั๋วที่มีแค่ 100 ใบ
		wg.Add(1)
		go func() {
			defer wg.Done()
			if buy(&sold) {
				successCount.Add(1)
			}
		}()
	}
	wg.Wait()
	return successCount.Load()
}

func main() {
	fmt.Println("--- buyTicketWrong (อาจขายเกิน 100 ใบเพราะ check-then-act race) ---")
	for i := 0; i < 3; i++ {
		fmt.Println("จำนวนตั๋วที่ขายได้จริง:", run(buyTicketWrong))
	}

	fmt.Println("--- buyTicketRight (ใช้ CompareAndSwap loop, ไม่มีวันเกิน 100 ใบ) ---")
	for i := 0; i < 3; i++ {
		fmt.Println("จำนวนตั๋วที่ขายได้จริง:", run(buyTicketRight))
	}
}
```

ผลลัพธ์ (ตัวเลขของ `buyTicketWrong` จะแตกต่างกันไปในแต่ละครั้ง แต่มักเกิน 100 เสมอ):

```
--- buyTicketWrong (อาจขายเกิน 100 ใบเพราะ check-then-act race) ---
จำนวนตั๋วที่ขายได้จริง: 102
จำนวนตั๋วที่ขายได้จริง: 111
จำนวนตั๋วที่ขายได้จริง: 135
--- buyTicketRight (ใช้ CompareAndSwap loop, ไม่มีวันเกิน 100 ใบ) ---
จำนวนตั๋วที่ขายได้จริง: 100
จำนวนตั๋วที่ขายได้จริง: 100
จำนวนตั๋วที่ขายได้จริง: 100
```

จะเห็นว่า `buyTicketWrong` **ขายตั๋วเกิน 100 ใบไปมาก** ทั้งที่ใช้ `atomic.Int64` ตลอดทั้งฟังก์ชัน! สาเหตุคือ `sold.Load()` (เช็คว่ายังขายได้ไหม) กับ `sold.Add(1)` (บวกค่าจริง) เป็นคนละ operation กัน — ระหว่างที่ goroutine ตัวหนึ่งเช็คว่า `sold.Load() < 100` (เช่น เห็นค่า 99) แล้วยังไม่ทัน `Add(1)` อาจมี goroutine อื่นอีกหลายสิบตัวมาเช็คแล้วเห็นค่า 99 เหมือนกันพอดี (เพราะยังไม่มีใคร update ค่าจริงเลย) ทุกตัวจึงคิดว่า "ยังขายได้" แล้วต่างคน**ต่าง `Add(1)` พร้อมกัน** ทำให้ยอดขายทะลุ 100 ไปไกล

ในทางกลับกัน `buyTicketRight` ใช้ **`CompareAndSwap` (CAS) แบบวนลูป** ซึ่งรวม "เช็ค" และ "ทำ" ให้เป็น**หน่วยเดียวที่แบ่งแยกไม่ได้จริงๆ**: อ่านค่าปัจจุบันมาเก็บไว้ (`current`) แล้วพยายามเปลี่ยนเป็น `current+1` **ก็ต่อเมื่อ**ค่าจริงตอนนั้นยังเป็น `current` อยู่เท่านั้น (ไม่มีใครมาแซงคั่นกลาง) — ถ้ามีคนแซงไปแล้ว CAS จะล้มเหลว (`false`) แล้ววนกลับไปอ่านค่าล่าสุดใหม่และลองใหม่จนกว่าจะสำเร็จหรือพบว่าตั๋วหมดแล้วจริง แพทเทิร์น "Load → คำนวณ → CompareAndSwap → วนใหม่ถ้าล้มเหลว" นี้คือรูปแบบมาตรฐานที่สุดของการเขียนโปรแกรมแบบ **lock-free** ด้วย CAS

> **บทเรียนสำคัญที่สุดของหัวข้อนี้**: **atomic type ป้องกันแค่ operation เดี่ยวๆ แต่ละตัวเท่านั้น ไม่ได้ทำให้ "ลำดับของหลาย operation ที่เขียนติดกัน" กลายเป็น atomic ไปด้วย** ถ้า logic ของคุณต้อง "เช็คค่าก่อนแล้วค่อยตัดสินใจทำอะไรต่อ" (compound operation) มีทางเลือก 2 ทาง: **(1)** ใช้ `CompareAndSwap` วนลูปแบบตัวอย่างข้างต้น (เหมาะกับกรณีง่ายๆ ที่ตัวแปรเดียว) หรือ **(2)** กลับไปใช้ `sync.Mutex` ป้องกันทั้งบล็อกของ "เช็ค-แล้ว-ทำ" ให้เป็นหน่วยเดียวกันไปเลย (เหมาะกับ logic ที่ซับซ้อนกว่าหรือเกี่ยวข้องกับหลายตัวแปรพร้อมกัน) — เมื่อ logic ซับซ้อนขึ้น มักจะเขียนและอ่านง่ายกว่าด้วย mutex

---

## 6. Memory Model เบื้องต้น: Happens-Before

หัวข้อสุดท้ายนี้เป็นแนวคิดเชิงลึกที่ควรรู้จักไว้ในระดับพื้นฐาน (รายละเอียดเต็มรูปแบบเกินขอบเขตของหลักสูตรระดับนี้ แต่สำคัญพอที่ควรรู้จักคำศัพท์และแนวคิดหลักไว้)

**Go Memory Model** คือข้อกำหนดที่บอกว่า **เมื่อไรที่การเขียนค่าของตัวแปรโดย goroutine หนึ่ง รับประกันว่าจะถูกมองเห็นโดย goroutine อีกตัวหนึ่งอย่างถูกต้อง** ฟังดูอาจแปลกใจ แต่ในระบบที่มีหลาย CPU core พร้อม cache ของตัวเอง **การเขียนค่าของ goroutine หนึ่งไม่ได้การันตีว่าจะถูกมองเห็นโดย goroutine อื่นทันที** เว้นแต่จะมีกลไก synchronization ที่ถูกต้องมาคั่นกลาง

แนวคิดหลักที่ใช้อธิบายเรื่องนี้เรียกว่า **happens-before relationship**: เหตุการณ์ A "happens-before" เหตุการณ์ B หมายความว่า **ผลของ A รับประกันว่าจะถูกมองเห็นโดย B อย่างสมบูรณ์** ตัวอย่างความสัมพันธ์ happens-before ที่กลไก concurrency ของ Go รับประกันให้ (ที่เราเรียนมาทั้งหมดในภาคนี้ล้วนอาศัยกฎเหล่านี้อยู่เบื้องหลัง):

- การส่งค่าเข้า channel **happens-before** การรับค่านั้นสำเร็จจากอีกฝั่ง (นี่คือเหตุผลที่ channel เป็นทั้งเครื่องมือส่งข้อมูล**และ**เครื่องมือ synchronization ไปพร้อมกัน ตามที่เรียนใน Part 037)
- การปิด channel **happens-before** การรับค่าที่คืนเป็น zero value เพราะ channel ถูกปิด (นี่คือกลไกเบื้องหลังของ done-channel pattern ใน Part 038)
- การปลดล็อก `sync.Mutex` **happens-before** การล็อกครั้งถัดไปที่สำเร็จบน mutex ตัวเดียวกัน (ไม่ว่าจะโดย goroutine ใดก็ตาม)
- การเรียก `wg.Done()` ครั้งสุดท้ายที่ทำให้ตัวนับเป็นศูนย์ **happens-before** การคืนค่าของ `wg.Wait()` ที่สอดคล้องกัน
- Atomic operation ทุกตัวใน `sync/atomic` มีความสัมพันธ์ happens-before ที่ชัดเจนกับ atomic operation อื่นบนตัวแปรเดียวกัน (นี่คือสิ่งที่ทำให้ atomic "ปลอดภัย" จริงๆ ไม่ใช่แค่ "เร็ว" เฉยๆ)

ทำไมเรื่องนี้ถึงสำคัญ? เพราะมันอธิบายว่า **ทำไมโค้ดที่ "ดูเหมือนน่าจะทำงานถูกต้อง" อาจไม่ถูกต้องจริงๆ** ถ้าไม่มีความสัมพันธ์ happens-before ที่ชัดเจนรองรับอยู่ ตัวอย่างเช่น การอ่าน-เขียนตัวแปรธรรมดา (ไม่ใช่ atomic, ไม่มี mutex ป้องกัน, ไม่ผ่าน channel) จากสอง goroutine พร้อมกัน **ไม่มีการรับประกันใดๆ ตาม memory model เลย** ว่าฝั่งหนึ่งจะเห็นค่าล่าสุดที่อีกฝั่งเขียนไป — นี่คือรากฐานทางทฤษฎีของสิ่งที่เราเรียกว่า "data race" ตลอดสองบทที่ผ่านมา

> **สำหรับผู้ที่ต้องการศึกษาเชิงลึก**: หัวข้อนี้เป็นเพียงการแนะนำแนวคิดผิวเผินที่สุดเท่านั้น รายละเอียดทางเทคนิคแบบเต็มรูปแบบ (รวมถึงกฎเฉพาะของแต่ละ synchronization primitive) มีอธิบายไว้อย่างเป็นทางการใน **The Go Memory Model** ที่ https://go.dev/ref/mem ซึ่งเป็นเอกสารที่ทุกคนที่เขียนโค้ด concurrent ระดับ production ควรอ่านอย่างน้อยหนึ่งครั้ง — ไม่จำเป็นต้องเข้าใจทุกรายละเอียดในตอนนี้ แต่ควรรู้ว่าเอกสารนี้มีอยู่และเป็นแหล่งอ้างอิงที่ถูกต้องที่สุดเมื่อเกิดข้อสงสัยเรื่อง memory visibility ระหว่าง goroutine

---

## สรุปสิ่งที่ได้เรียนในบทนี้

- **Atomic operation** คือคำสั่งระดับ CPU ที่รับประกัน indivisibility โดยไม่ต้องพึ่ง lock ระดับ software — เหมาะกับการกระทำเดี่ยวๆ บนตัวแปรตัวเดียว เช่น ตัวนับหรือ flag
- **Typed API สมัยใหม่** (`atomic.Int64`, `atomic.Bool`, `atomic.Value`, `atomic.Pointer[T]` — Go 1.19+) ควรใช้เป็นค่าเริ่มต้นแทน API แบบเก่า (`atomic.AddInt64(&x, 1)`) เพราะปลอดภัยและอ่านง่ายกว่า
- **`Load`/`Store`/`Add`/`CompareAndSwap`** คือ method หลักของ atomic type สมัยใหม่ — `CompareAndSwap` (CAS) เป็นหัวใจของการเขียนโปรแกรมแบบ lock-free
- **Benchmark จริง** ยืนยันว่า atomic เร็วกว่า mutex อย่างมีนัยสำคัญสำหรับ operation ง่ายๆ (ในตัวอย่างนี้ประมาณ 5-6 เท่า) แต่ตัวเลขจริงขึ้นกับระบบและควรวัดเอง
- **Atomic ไม่ได้แก้ปัญหา check-then-act race**: การรวมหลาย atomic operation เข้าด้วยกัน (เช่น Load แล้วค่อย Add) ไม่ได้เป็น atomic โดยรวม — ต้องใช้ `CompareAndSwap` วนลูป หรือกลับไปใช้ `sync.Mutex` ป้องกันทั้งบล็อก
- **Go Memory Model** และแนวคิด **happens-before** อธิบายว่าเมื่อไรการเขียนค่าของ goroutine หนึ่งรับประกันว่าจะถูกมองเห็นโดยอีกตัวหนึ่ง — เป็นรากฐานทางทฤษฎีเบื้องหลังกลไก concurrency ทั้งหมดที่เรียนมาในภาคนี้

## แบบฝึกหัดท้ายบท

1. เขียนโปรแกรมที่ใช้ `atomic.Int32` นับจำนวนครั้งที่ฟังก์ชันหนึ่งถูกเรียกจากหลาย goroutine พร้อมกัน (1000 goroutine เรียกฟังก์ชันเดียวกัน) แล้วยืนยันด้วย `Load()` ว่าค่าสุดท้ายถูกต้องครบ 1000
2. แปลงโค้ดในข้อ 1 ให้ใช้ API แบบเก่า (`atomic.AddInt32`/`atomic.LoadInt32`) แทน แล้วเปรียบเทียบว่าโค้ดแบบไหนอ่านง่ายกว่าและอธิบายเหตุผล
3. เขียน struct `RateLimiter` ที่มี field เป็น `atomic.Int64` เก็บจำนวน request ที่อนุญาตให้ผ่านได้ในหน้าต่างเวลาหนึ่ง (เช่น ไม่เกิน 100 request) ใช้ `CompareAndSwap` แบบวนลูปเพื่อป้องกัน check-then-act race แบบเดียวกับตัวอย่าง `buyTicketRight` ในหัวข้อ 5
4. เขียน benchmark เปรียบเทียบ `atomic.Bool` กับ `sync.Mutex` ที่ป้องกัน field `bool` ธรรมดา ในสถานการณ์ที่มีการ `Store`/`Lock+Unlock` ถี่ๆ พร้อมกันหลาย goroutine ด้วย `b.RunParallel` แล้วรายงานผลตัวเลขที่ได้บนเครื่องของตัวเอง
5. ทดลองรันตัวอย่าง `buyTicketWrong` ในหัวข้อ 5 ซ้ำ 10 ครั้ง บันทึกจำนวนตั๋วที่ขายได้จริงในแต่ละครั้ง แล้วอธิบายว่าทำไมตัวเลขไม่คงที่และทำไมบางครั้งอาจได้ 100 พอดีโดยบังเอิญ
6. ค้นคว้าเพิ่มเติม: เปิดอ่าน The Go Memory Model ที่ https://go.dev/ref/mem หาหัวข้อที่อธิบายความสัมพันธ์ happens-before ของ `sync.Once` โดยเฉพาะ แล้วอธิบายด้วยคำพูดตัวเองว่าทำไมโค้ดใน `once.Do(f)` ถึงรับประกันว่าทุก goroutine ที่เรียก `Get()` ทีหลังจะเห็นผลของ `f` ครบถ้วนเสมอ ไม่มีทางเห็นสถานะที่เขียนไม่เสร็จ

---

**ต่อไป**: [Part 041 — Worker Pool Pattern](./041-worker-pool-pattern.md)
