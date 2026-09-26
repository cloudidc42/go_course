# Part 055: แพ็กเกจ `runtime` และ Garbage Collector

> ภาคที่ 4: Standard Library เชิงลึก — ตอนที่ 10 จาก 10 (Part 46–55)

## สารบัญของบทนี้

1. แพ็กเกจ `runtime` คืออะไร
2. ตรวจสอบสภาพแวดล้อมด้วย `NumCPU`, `NumGoroutine`, `GOMAXPROCS`
3. ตรวจสอบหน่วยความจำด้วย `runtime.ReadMemStats`
4. `runtime.GC()`: การสั่ง GC ทำงานด้วยมือ (และทำไมแทบไม่จำเป็น)
5. Go's Garbage Collector ทำงานอย่างไร (ระดับแนวคิด)
6. ทำไม Go GC ถึงเน้น Low Latency มากกว่า Throughput
7. ปรับแต่ง GC ด้วย `GOGC` และ `GOMEMLIMIT`
8. Escape Analysis ทบทวน: `go build -gcflags="-m"` เจาะลึก
9. เมื่อไรไม่ควรสู้กับ GC (Premature Optimization)
10. สรุปสิ่งที่ได้เรียนในบทนี้
11. แบบฝึกหัดท้ายบท

---

## 1. แพ็กเกจ `runtime` คืออะไร

**`runtime`** เป็น package มาตรฐานที่เปิดหน้าต่างเล็กๆ ให้เรามองเข้าไปใน **Go runtime** — ระบบที่ทำงานอยู่เบื้องหลังทุกโปรแกรม Go คอยจัดการ goroutine scheduler, garbage collector, memory allocator และกลไกพื้นฐานอื่นๆ ที่ทำให้ Go เป็นภาษาที่ไม่ต้องจัดการ memory เองแบบ C/C++

เราเคยใช้ `runtime.NumCPU()` และ `runtime.GOMAXPROCS()` ไปแล้วสั้นๆ ในบทที่ 036 ตอนเรียนเรื่อง Goroutines ในบทนี้เราจะกลับมาเจาะลึก package นี้อย่างเต็มรูปแบบ ทั้งการตรวจสอบสถานะของโปรแกรมที่กำลังรันอยู่ และการทำความเข้าใจ **Garbage Collector (GC)** ซึ่งเป็นหนึ่งในกลไกสำคัญที่สุดของ Go runtime — สิ่งที่ทำให้เราเขียนโปรแกรม Go โดยไม่ต้องเรียก `free()` เองแบบ C

### เปรียบเทียบสั้นๆ กับภาษาอื่น

ก่อนเข้าเนื้อหา ควรทำความเข้าใจก่อนว่า Go ไม่ใช่ภาษาแรกที่มี Garbage Collector — แต่ละภาษาก็เลือกออกแบบ GC ให้เหมาะกับจุดประสงค์การใช้งานของตัวเองต่างกันไป:

| ภาษา | แนวทาง GC | จุดเน้น |
|---|---|---|
| C / C++ | ไม่มี GC — ผู้เขียนโปรแกรมจัดการ memory เอง (`malloc`/`free`, RAII) | ควบคุมได้เต็มที่ แต่เสี่ยง memory leak/dangling pointer สูง |
| Java (JVM) | หลายอัลกอริทึมให้เลือก (G1, ZGC, Shenandoah) ปรับแต่งได้ละเอียดมาก | Throughput สูง ปรับแต่งได้หลากหลาย แต่ tuning ซับซ้อน |
| Python | Reference counting + cycle collector เสริม | เรียบง่าย แต่มี overhead ต่อทุก object และ GIL จำกัด concurrency |
| **Go** | Concurrent tri-color mark-and-sweep ตัวเดียว ปรับแต่งน้อยจุดโดยเจตนา | **Low latency** และใช้งานง่าย แทบไม่ต้อง tuning ในกรณีทั่วไป |

จุดที่ทำให้ Go แตกต่างคือ**ความเรียบง่ายโดยเจตนา** (ตามปรัชญาที่กล่าวถึงในบทที่ 001) — แทนที่จะให้ผู้ใช้เลือกอัลกอริทึม GC ได้หลายแบบแบบ Java ทีม Go เลือกดูแล GC ตัวเดียวให้ดีที่สุดสำหรับ use case ส่วนใหญ่ (server/network application) แล้วเปิดจุดปรับแต่งไว้แค่ไม่กี่ตัว (`GOGC`, `GOMEMLIMIT`) ที่ครอบคลุมความต้องการเกือบทั้งหมดโดยไม่ต้องเข้าใจ internals ลึกซึ้ง

---

## 2. ตรวจสอบสภาพแวดล้อมด้วย `NumCPU`, `NumGoroutine`, `GOMAXPROCS`

```go
package main

import (
	"fmt"
	"runtime"
)

func main() {
	fmt.Println("NumCPU:", runtime.NumCPU())
	fmt.Println("GOMAXPROCS:", runtime.GOMAXPROCS(0))
	fmt.Println("NumGoroutine (start):", runtime.NumGoroutine())

	done := make(chan struct{})
	for i := 0; i < 5; i++ {
		go func() {
			<-done
		}()
	}
	fmt.Println("NumGoroutine (after spawning 5):", runtime.NumGoroutine())
	close(done)
}
```

ผลลัพธ์ (ค่า `NumCPU`/`GOMAXPROCS` ขึ้นกับเครื่องที่รัน):

```
NumCPU: 4
GOMAXPROCS: 4
NumGoroutine (start): 1
NumGoroutine (after spawning 5): 6
```

### ทบทวนความหมายของแต่ละค่า

- **`runtime.NumCPU()`**: จำนวน logical CPU core ของเครื่องที่โปรแกรมกำลังรันอยู่ (ค่าคงที่ อ่านจากระบบปฏิบัติการ อ่านได้อย่างเดียว)
- **`runtime.GOMAXPROCS(0)`**: จำนวน OS thread สูงสุดที่ Go runtime จะใช้รันโค้ด Go แบบขนานพร้อมกัน — เรียกด้วย `0` หมายถึงแค่ **query** ค่าปัจจุบัน ไม่เปลี่ยนแปลงอะไร (ถ้าส่งค่าอื่นที่ไม่ใช่ 0 จะเป็นการ**ตั้งค่าใหม่**และคืนค่าเก่ากลับมา) ตามที่เรียนไปในบทที่ 036 ค่า default เท่ากับ `NumCPU()` และแทบไม่จำเป็นต้องปรับเองยกเว้นกรณีพิเศษ เช่น รันใน container ที่ถูกจำกัด CPU quota ต่ำกว่าจำนวน core จริงของเครื่องแม่
- **`runtime.NumGoroutine()`**: จำนวน goroutine ที่กำลังทำงานอยู่ ณ ขณะนั้น (รวม goroutine หลักของ `main()` เองด้วย จึงเริ่มต้นที่ 1 เสมอ) มีประโยชน์มากสำหรับ debug **goroutine leak** (goroutine ที่ไม่มีวันจบตามที่กล่าวถึงในบทที่ 036) — ถ้าเรียกค่านี้เป็นระยะแล้วเห็นตัวเลขค่อยๆ ไต่สูงขึ้นเรื่อยๆ โดยไม่มีทีท่าจะลดลง มักเป็นสัญญาณว่ามี goroutine ค้างอยู่ที่ไหนสักแห่งในระบบ

---

## 3. ตรวจสอบหน่วยความจำด้วย `runtime.ReadMemStats`

`runtime.ReadMemStats` อ่านสถิติการใช้หน่วยความจำของโปรแกรมปัจจุบันมาเก็บใน struct `runtime.MemStats` ซึ่งมี field จำนวนมาก แต่ที่ใช้บ่อยที่สุดมีไม่กี่ตัว:

```go
package main

import (
	"fmt"
	"runtime"
)

func main() {
	var m runtime.MemStats
	runtime.ReadMemStats(&m)
	fmt.Printf("Alloc = %v KB\n", m.Alloc/1024)
	fmt.Printf("TotalAlloc = %v KB\n", m.TotalAlloc/1024)
	fmt.Printf("Sys = %v KB\n", m.Sys/1024)
	fmt.Printf("NumGC = %v\n", m.NumGC)

	// จำลองการจัดสรร memory จำนวนมากเพื่อดูสถิติเปลี่ยนไป
	data := make([][]byte, 0)
	for i := 0; i < 1000; i++ {
		data = append(data, make([]byte, 1024*10))
	}
	_ = data

	runtime.GC() // สั่งให้ GC ทำงานทันที (จะอธิบายในหัวข้อถัดไป)

	var m2 runtime.MemStats
	runtime.ReadMemStats(&m2)
	fmt.Printf("After alloc+GC: Alloc = %v KB, NumGC = %v\n", m2.Alloc/1024, m2.NumGC)
}
```

ผลลัพธ์ตัวอย่าง (ตัวเลขจริงจะต่างกันไปตามเครื่องและเวอร์ชัน Go):

```
Alloc = 78 KB
TotalAlloc = 78 KB
Sys = 5975 KB
NumGC = 0
After alloc+GC: Alloc = 83 KB, NumGC = 3
```

### ความหมายของ field สำคัญใน `MemStats`

| Field | ความหมาย |
|---|---|
| `Alloc` | หน่วยความจำบน heap ที่กำลังถูกใช้งานอยู่ ณ ขณะนี้ (ยังไม่ถูกเก็บกวาด) |
| `TotalAlloc` | ผลรวมสะสมของหน่วยความจำที่เคยจัดสรรบน heap ตั้งแต่โปรแกรมเริ่มรัน (ไม่ลดลงแม้ GC จะเก็บกวาดไปแล้ว) |
| `Sys` | หน่วยความจำทั้งหมดที่ Go runtime ขอมาจากระบบปฏิบัติการ (รวม heap, stack, และ internal structure ต่างๆ) |
| `NumGC` | จำนวนครั้งที่ GC ทำงานไปแล้วนับตั้งแต่โปรแกรมเริ่มรัน |
| `HeapObjects` | จำนวน object ทั้งหมดที่อยู่บน heap ณ ขณะนี้ |
| `HeapAlloc` | เหมือนกับ `Alloc` แต่เจาะจงเฉพาะส่วน heap (ในทางปฏิบัติค่าจะเท่ากับ `Alloc` เสมอ) |
| `HeapIdle` | หน่วยความจำบน heap ที่ runtime จองไว้จาก OS แล้วแต่ยังไม่ได้ใช้งาน ณ ขณะนี้ |
| `HeapReleased` | ส่วนของ `HeapIdle` ที่ runtime คืนกลับให้ OS ไปแล้วจริงๆ (ไม่ได้ถือครองไว้เปล่าๆ) |
| `PauseTotalNs` | ผลรวมเวลาที่โปรแกรมถูกหยุด (stop-the-world) เพื่อ GC สะสมทั้งหมด หน่วยเป็นนาโนวินาที |

Field เหล่านี้มีประโยชน์มากตอนวิเคราะห์ปัญหาเรื่อง memory ในโปรแกรม production เช่น ตรวจสอบว่า memory usage โตขึ้นเรื่อยๆ (สัญญาณของ memory leak) หรือ GC ทำงานถี่เกินไปจนกระทบ performance — การวิเคราะห์เชิงลึกกว่านี้ด้วยเครื่องมือ `pprof` จะเรียนในภาคที่ 7 (Part 083: Profiling)

---

## 4. `runtime.GC()`: การสั่ง GC ทำงานด้วยมือ (และทำไมแทบไม่จำเป็น)

`runtime.GC()` บังคับให้ GC ทำงานหนึ่งรอบทันที (blocking จนกว่าจะเสร็จ) แทนที่จะรอให้ runtime ตัดสินใจเองว่าควรรันตอนไหน

```go
runtime.GC() // บังคับให้ GC ทำงานทันที
```

### ทำไมแทบไม่ต้องเรียกเองในทางปฏิบัติ

Go runtime มีระบบตัดสินใจอัตโนมัติที่ซับซ้อนและได้รับการปรับจูนมาอย่างดีอยู่แล้วว่า **ควรรัน GC เมื่อไร** โดยพิจารณาจากอัตราการจัดสรร memory เทียบกับ memory ที่ยังมีชีวิตอยู่ (คุมด้วยค่า `GOGC` ที่จะพูดถึงในหัวข้อที่ 7) การเรียก `runtime.GC()` เองมักจะ:

- **ไม่ได้ช่วยอะไร** เพราะ runtime มักตัดสินใจได้ดีกว่ามนุษย์อยู่แล้วว่าจังหวะไหนเหมาะสม
- **อาจทำให้ช้าลง** เพราะบังคับให้ GC ทำงานบ่อยกว่าที่จำเป็น สิ้นเปลือง CPU cycle ไปโดยเปล่าประโยชน์

### เมื่อไรถึงจะมีเหตุผลให้เรียก

กรณีที่พบได้บ้าง (แต่ยังคงหายาก) คือในการเขียน **benchmark** หรือ **test** ที่ต้องการวัด memory state ที่ "นิ่ง" แล้ว (ไม่มี garbage ค้างจากรอบก่อนหน้าปนเข้ามารบกวนผลวัด) หรือในโปรแกรมที่รู้ล่วงหน้าแน่ชัดว่ากำลังจะเข้าสู่ช่วงพักที่ไม่ทำงานหนัก (idle period) และต้องการคืน memory ให้ระบบปฏิบัติการโดยเร็ว — แต่ในโค้ด business logic ทั่วไป **แทบไม่มีเหตุผลให้เรียก `runtime.GC()` เองเลย**

---

## 5. Go's Garbage Collector ทำงานอย่างไร (ระดับแนวคิด)

หัวข้อนี้อธิบายแนวคิดของ Go GC ในระดับที่เพียงพอสำหรับความเข้าใจการทำงานและผลกระทบต่อโปรแกรมของเรา **ไม่ได้ลงรายละเอียดระดับ internals ของ runtime source code** (ซึ่งซับซ้อนมากและเปลี่ยนแปลงไปในแต่ละเวอร์ชันของ Go)

### Tri-color Mark-and-Sweep คืออะไร

Go ใช้อัลกอริทึม GC แบบ **tri-color mark-and-sweep** ที่ทำงาน**พร้อมกัน (concurrently)** กับโปรแกรมของเรา แนวคิดหลักแบ่งเป็นสองขั้นตอนใหญ่:

1. **Mark (ทำเครื่องหมาย)**: เริ่มจาก **root** (ตัวแปร global, ตัวแปรบน stack ของทุก goroutine) แล้วไล่ตาม pointer ทั้งหมดเพื่อหาว่า object ใดบ้าง "ยังมีชีวิตอยู่" (ยังมีใครอ้างถึงอยู่)
2. **Sweep (กวาดทิ้ง)**: object ใดๆ ที่ไม่ถูก mark ว่ามีชีวิต (ไม่มี pointer ใดอ้างถึงแล้ว) ถือว่าเป็น **garbage** และหน่วยความจำของมันจะถูกคืนกลับไปให้ allocator นำไปใช้ใหม่ได้

ที่มาของชื่อ "tri-color" คือระหว่างขั้นตอน mark, GC จะแบ่ง object ทั้งหมดออกเป็น 3 สี:

- **สีขาว (White)**: ยังไม่ถูกตรวจสอบ (ค่าเริ่มต้นของทุก object เมื่อเริ่มรอบ GC ใหม่) — ถ้าจบรอบ mark แล้วยังเป็นสีขาวอยู่ แปลว่าไม่มีใครอ้างถึงแล้ว จะถูกกวาดทิ้งในขั้นตอน sweep
- **สีเทา (Gray)**: ถูกพบว่ามีชีวิต (มีคนอ้างถึง) แล้ว แต่ยังไม่ได้ตรวจสอบ pointer ที่ตัวมันเองถืออยู่ต่อ
- **สีดำ (Black)**: มีชีวิตแน่นอน และตรวจสอบ pointer ภายในของมันครบถ้วนแล้ว

กระบวนการ mark ไล่เปลี่ยนสีจากขาว → เทา → ดำ ไปเรื่อยๆ จนไม่เหลือ object สีเทาอีกต่อไป (แปลว่าตรวจสอบครบทุกเส้นทางที่เข้าถึงได้แล้ว) เทคนิคการแบ่งสามสีนี้เองที่ทำให้ GC สามารถทำงาน**สลับกับโปรแกรมจริงได้อย่างปลอดภัย** แม้โปรแกรมจะยังคงสร้างและแก้ไข pointer ต่อไประหว่างที่ GC กำลังทำงานอยู่ก็ตาม (ด้วยกลไกเสริมที่เรียกว่า **write barrier** ซึ่งคอยตรวจจับการเปลี่ยนแปลง pointer ระหว่างการ mark เพื่อรักษาความถูกต้อง)

---

## 6. ทำไม Go GC ถึงเน้น Low Latency มากกว่า Throughput

การออกแบบ GC ในภาษาต่างๆ ต้องเลือกจุดสมดุลระหว่างสองสิ่งที่มักขัดแย้งกัน:

- **Throughput**: ประมวลผลงานได้มากที่สุดต่อหน่วยเวลา (ใช้ CPU cycle ไปกับ business logic ให้มากที่สุด เสีย overhead ให้ GC ให้น้อยที่สุด)
- **Latency**: เวลาที่โปรแกรมต้อง "หยุดชะงัก" (pause) เพื่อให้ GC ทำงาน สั้นที่สุดเท่าที่จะทำได้ในแต่ละครั้ง

Go **เลือกให้น้ำหนักกับ low latency เป็นหลัก** เพราะ Go ถูกออกแบบมาสำหรับงานประเภท network server และ concurrent system ที่ต้องตอบสนอง request ได้อย่างรวดเร็วและสม่ำเสมอ (predictable) — การหยุดโปรแกรมทั้งหมดเป็นเวลานาน (**stop-the-world pause**) แม้เพียงไม่กี่ร้อยมิลลิวินาที ก็อาจทำให้ request ของผู้ใช้ค้างหรือ timeout ได้ในระบบที่ต้องรองรับ concurrent request จำนวนมาก

GC รุ่นเก่าในหลายภาษา (รวมถึง Go เวอร์ชันแรกๆ ก่อนปี 2015) ทำงานแบบ **stop-the-world** เต็มรูปแบบ คือหยุดทุก goroutine ทั้งหมดระหว่าง mark ทั้งกระบวนการ ซึ่งอาจใช้เวลาหลายร้อยมิลลิวินาทีถึงหลักวินาทีในโปรแกรมที่มี heap ขนาดใหญ่ — ทีม Go ทุ่มเทพัฒนา GC มาอย่างต่อเนื่องหลายปี จนปัจจุบัน **ขั้นตอน mark ส่วนใหญ่ทำงานพร้อมกัน (concurrent) กับโปรแกรมจริงได้** เหลือเพียงช่วงสั้นๆ ที่ต้อง stop-the-world จริงๆ (เช่น ตอนเริ่มต้นและสิ้นสุดรอบ mark เพื่อซิงค์สถานะ) ทำให้ pause time ในเวอร์ชันปัจจุบันของ Go มักอยู่ที่ระดับ **ต่ำกว่า 1 มิลลิวินาที** แม้ในโปรแกรมที่มี heap ขนาดหลาย GB — นี่คือหนึ่งในจุดขายสำคัญที่สุดของ Go สำหรับงาน backend/infrastructure ที่ต้องการ latency ต่ำและสม่ำเสมอ

ข้อแลกเปลี่ยนคือ Go GC ใช้ CPU cycle มากกว่า GC แบบเน้น throughput ล้วนๆ ของบางภาษา (เพราะต้องทำงานสลับกับโปรแกรมหลักตลอดเวลา ไม่ใช่หยุดรอบเดียวจบ) แต่นี่เป็นการแลกเปลี่ยนที่เหมาะสมกับ use case ส่วนใหญ่ของ Go

---

## 7. ปรับแต่ง GC ด้วย `GOGC` และ `GOMEMLIMIT`

แม้ GC จะทำงานอัตโนมัติโดยไม่ต้องแตะต้องอะไรเลยในกรณีส่วนใหญ่ Go ยังเปิดช่องให้ปรับจูนพฤติกรรมผ่าน environment variable สองตัวสำหรับกรณีที่จำเป็นจริงๆ

### `GOGC`: ควบคุมความถี่ของ GC

`GOGC` กำหนด**อัตราส่วน**ระหว่างขนาด heap หลังรอบ GC ล่าสุด กับขนาด heap ที่จะกระตุ้นให้ GC รอบถัดไปเริ่มทำงาน ค่า default คือ **100** ซึ่งหมายความว่า:

> GC รอบถัดไปจะเริ่มทำงานเมื่อ heap โตขึ้นเป็น **2 เท่า** ของขนาด heap ที่เหลืออยู่หลังรอบ GC ล่าสุด (เพิ่มขึ้น 100%)

```bash
GOGC=50 ./myapp   # GC ทำงานถี่ขึ้น (heap โต 50% ก็ trigger GC แล้ว) — ใช้ memory น้อยลง แต่เปลือง CPU มากขึ้น
GOGC=200 ./myapp  # GC ทำงานห่างขึ้น (heap ต้องโต 200% ถึงจะ trigger) — ใช้ memory มากขึ้น แต่เปลือง CPU น้อยลง
GOGC=off ./myapp  # ปิด GC โดยสมบูรณ์ — memory จะโตขึ้นเรื่อยๆ ไม่มีการเก็บกวาดเลย (อันตรายมาก ใช้เฉพาะกรณีพิเศษจริงๆ)
```

ปรับได้ทั้งผ่าน environment variable ก่อนรันโปรแกรม หรือปรับจากในโค้ดผ่านฟังก์ชัน `debug.SetGCPercent` ใน package `runtime/debug`:

```go
import "runtime/debug"

debug.SetGCPercent(50) // เทียบเท่ากับตั้ง GOGC=50
```

### `GOMEMLIMIT`: กำหนดเพดานหน่วยความจำ (Go 1.19+)

`GOGC` ควบคุม "อัตราส่วน" แต่ไม่ได้จำกัด "จำนวนสูงสุด" ของ memory ที่โปรแกรมจะใช้ ในสภาพแวดล้อมที่มีข้อจำกัดด้าน memory ชัดเจน เช่น container ที่ถูกจำกัด memory limit ไว้ (Kubernetes pod ที่ตั้ง memory limit) การปล่อยให้ heap โตได้ไม่จำกัดอาจทำให้โปรแกรมถูก **OOM-killed** (ระบบปฏิบัติการฆ่าโปรเซสเพราะใช้ memory เกิน)

**`GOMEMLIMIT`** (เพิ่มเข้ามาใน Go 1.19) กำหนด**เพดานหน่วยความจำรวม (soft memory limit)** ที่ Go runtime จะพยายามไม่ให้เกิน โดย GC จะทำงานถี่ขึ้นโดยอัตโนมัติเมื่อ memory usage ใกล้เพดานที่กำหนด:

```bash
GOMEMLIMIT=512MiB ./myapp   # พยายามไม่ให้ heap รวมเกิน 512 MiB
```

`GOGC` และ `GOMEMLIMIT` ใช้**ร่วมกันได้** — แนวทางที่แนะนำในปัจจุบันคือตั้ง `GOMEMLIMIT` ให้ต่ำกว่า memory limit ของ container เล็กน้อย (เผื่อพื้นที่ให้ non-heap memory เช่น goroutine stack) ในขณะที่ปล่อยให้ `GOGC` ทำงานตามค่า default หรือปรับสูงขึ้นเล็กน้อยเพื่อลดความถี่ของ GC ในสถานการณ์ปกติ แล้วให้ `GOMEMLIMIT` เป็นเหมือน "ตาข่ายนิรภัย" คอยเร่ง GC เฉพาะตอนที่ memory ใกล้ชนเพดานจริงๆ

> **ข้อควรระวัง**: ทั้ง `GOGC` และ `GOMEMLIMIT` เป็นเครื่องมือสำหรับ**สถานการณ์เฉพาะที่วัดผลได้จริงแล้วว่าจำเป็น** เช่น จาก production metrics ที่เห็นปัญหาชัดเจน ไม่ควรตั้งค่าตามความรู้สึกหรือเลียนแบบโปรเจกต์อื่นโดยไม่เข้าใจบริบท เพราะการตั้งค่าไม่เหมาะสม (เช่น `GOGC` ต่ำเกินไป) อาจทำให้ CPU usage สูงขึ้นอย่างมีนัยสำคัญโดยไม่ได้ประโยชน์คุ้มค่า

### ปรับค่าจากในโค้ดด้วย `runtime/debug`

นอกจาก environment variable แล้ว ทั้งสองค่ายังปรับได้จากในโค้ดโดยตรงผ่าน package `runtime/debug` ซึ่งมีประโยชน์เมื่อต้องการปรับค่าแบบ dynamic ตาม logic ของโปรแกรมเอง (เช่น ปรับตาม config ที่โหลดมาตอน runtime แทนที่จะพึ่ง environment variable ที่ตั้งตายตัวตอน deploy):

```go
package main

import (
	"fmt"
	"runtime"
	"runtime/debug"
)

func main() {
	old := debug.SetGCPercent(50) // เทียบเท่า GOGC=50 ตั้งจากในโค้ด
	fmt.Println("previous GOGC:", old)

	oldLimit := debug.SetMemoryLimit(256 << 20) // เทียบเท่า GOMEMLIMIT=256MiB
	fmt.Println("previous memory limit:", oldLimit)

	data := make([][]byte, 0)
	for i := 0; i < 2000; i++ {
		data = append(data, make([]byte, 1024*10))
	}
	_ = data

	var m runtime.MemStats
	runtime.ReadMemStats(&m)
	fmt.Println("NumGC after allocations:", m.NumGC)
}
```

ผลลัพธ์ตัวอย่าง:

```
previous GOGC: 100
previous memory limit: 9223372036854775807
NumGC after allocations: 6
```

สังเกตว่า `debug.SetMemoryLimit` คืนค่าเดิมกลับมาเป็น `9223372036854775807` (`math.MaxInt64`) ซึ่งเป็นค่า default ของ `GOMEMLIMIT` — หมายความว่า**โดย default แล้ว Go ไม่ได้จำกัดเพดาน memory ไว้เลย** (ปิดฟีเจอร์นี้ไว้จนกว่าจะตั้งค่าเอง) ทั้ง `SetGCPercent` และ `SetMemoryLimit` คืนค่าเดิมก่อนเปลี่ยนกลับมาเสมอ ทำให้ปรับค่าไปมาชั่วคราวแล้วคืนค่าเดิมได้ง่าย (pattern คล้ายกับ `runtime.GOMAXPROCS(n)` ที่คืนค่าเดิมเช่นกัน)

### วิวัฒนาการของ Go GC ในภาพรวม

Go GC ไม่ได้เป็นแบบที่เห็นในปัจจุบันตั้งแต่ต้น แต่ผ่านการปรับปรุงครั้งใหญ่มาหลายรอบ:

- **Go 1.0-1.4**: GC แบบ stop-the-world ล้วนๆ เวลา pause อาจสูงถึงหลักร้อยมิลลิวินาทีในโปรแกรมที่มี heap ขนาดใหญ่
- **Go 1.5 (2015)**: เปลี่ยนมาใช้ concurrent mark-and-sweep เป็นครั้งแรก ลด pause time ลงอย่างมาก และเป็นรุ่นเดียวกับที่ compiler ถูกเขียนใหม่ทั้งหมดด้วยภาษา Go เอง (ตามที่กล่าวถึงในบทที่ 001)
- **Go 1.8 (2017)**: ปรับปรุง write barrier ทำให้ pause time ส่วนใหญ่ลดลงเหลือระดับต่ำกว่า 1 มิลลิวินาที
- **ปัจจุบัน**: ทีม Go ยังคงปรับจูน GC อย่างต่อเนื่องในทุกรุ่น เน้นลด CPU overhead ของ GC โดยไม่กระทบ latency ที่ทำได้ดีอยู่แล้ว รวมถึงเพิ่มเครื่องมือควบคุมอย่าง `GOMEMLIMIT` ใน Go 1.19 เพื่อรองรับการรันในสภาพแวดล้อม container ที่มีข้อจำกัด memory ชัดเจนมากขึ้นเรื่อยๆ

---

## 8. Escape Analysis ทบทวน: `go build -gcflags="-m"` เจาะลึก

ในบทที่ 010 (Pointers) เราเรียนไปแล้วว่า **escape analysis** คือขั้นตอนที่ compiler วิเคราะห์ตอน compile time ว่าตัวแปรแต่ละตัวควรอยู่บน **stack** (เร็ว จัดการอัตโนมัติเมื่อฟังก์ชันจบ) หรือต้อง "escape" ไปอยู่บน **heap** (ต้องพึ่ง GC ในการเก็บกวาด) — มาดูตัวอย่างที่ใหญ่ขึ้นเพื่อเห็นภาพชัดเจนกว่าเดิมว่า escape analysis ตัดสินใจอย่างไรในสถานการณ์ต่างๆ

```go
package main

import "fmt"

type User struct {
	Name string
	Age  int
}

// newUserOnHeap: คืน pointer -> escape ไป heap เสมอ
// เพราะตัวแปร u ต้องมีชีวิตอยู่ต่อหลังฟังก์ชันจบ (ผู้เรียกยังใช้ pointer นี้ต่อ)
func newUserOnHeap(name string, age int) *User {
	u := User{Name: name, Age: age}
	return &u
}

// sumOnStack: ไม่มี pointer หลุดออกไปไหนเลย -> อยู่บน stack ได้ ไม่ต้องพึ่ง GC
func sumOnStack(a, b int) int {
	result := a + b
	return result
}

// printUser: รับค่าแบบ value (ไม่ใช่ pointer) แต่ fmt.Println รับพารามิเตอร์เป็น ...any (interface)
// ทำให้ compiler ไม่มั่นใจว่าค่าที่ส่งเข้าไปจะถูกเก็บอ้างอิงไว้นานแค่ไหน จึงตัดสินใจให้ escape ไป heap
func printUser(u User) {
	fmt.Println(u)
}

func main() {
	u := newUserOnHeap("Alice", 30)
	fmt.Println(u)

	s := sumOnStack(3, 4)
	fmt.Println(s)

	local := User{Name: "Bob", Age: 25}
	printUser(local)
}
```

รันคำสั่งนี้เพื่อดูรายงานของ escape analysis:

```bash
go build -gcflags="-m" escape_demo.go
```

ผลลัพธ์ (ตัดเฉพาะบรรทัดที่สำคัญ):

```
./escape_demo.go:11:20: leaking param: name
./escape_demo.go:12:2: moved to heap: u
./escape_demo.go:24:16: leaking param: u
./escape_demo.go:25:14: u escapes to heap
./escape_demo.go:33:14: s escapes to heap
./escape_demo.go:36:11: u escapes to heap
```

### อ่านผลลัพธ์ทีละบรรทัด

- **`moved to heap: u`** (บรรทัด 12 ใน `newUserOnHeap`): ยืนยันสิ่งที่คาดไว้ — เพราะฟังก์ชันคืน `&u` ตัวแปร `u` ต้อง "รอด" อยู่หลังฟังก์ชันจบการทำงาน compiler จึงต้องย้ายไปไว้บน heap เพื่อให้ผู้เรียกใช้ pointer นั้นต่อได้อย่างปลอดภัย (ตามหลักการที่เรียนไปในบทที่ 010)
- **`s escapes to heap`** (บรรทัดใน `main`): แม้ `sumOnStack` เองจะคำนวณ `result` บน stack ล้วนๆ (ไม่มีบรรทัด `moved to heap` ปรากฏสำหรับฟังก์ชันนี้เลย) แต่พอค่า `s` ถูกส่งเข้า `fmt.Println(s)` ใน `main` มันกลับ escape ไป heap — เหตุผลคือ `fmt.Println` รับพารามิเตอร์เป็น `...any` (interface type) ทำให้ compiler **ไม่สามารถวิเคราะห์ต่อได้ว่า `fmt.Println` จะเก็บ reference ของค่านั้นไว้ใช้ต่อหรือไม่** จึงเลือกทางที่ปลอดภัยไว้ก่อนคือย้ายไป heap เสมอ
- **`u escapes to heap`** (ในทั้ง `printUser` และตอนเรียกจาก `main`): เหตุผลเดียวกัน — ทันทีที่ค่าถูกส่งผ่าน interface parameter (`fmt.Println`, หรือ interface อื่นใดก็ตาม) โอกาสสูงมากที่ compiler จะตัดสินใจให้ escape ไป heap เพราะไม่สามารถรับประกันขอบเขตชีวิตของค่าที่ชัดเจนได้อีกต่อไป

### บทเรียนสำคัญจากตัวอย่างนี้

การส่งค่าผ่าน **interface parameter** (ซึ่งพบได้บ่อยมาก เพราะ `fmt.Println`, `log.Println`, และ logging function จำนวนมากรับ `...any`) มักทำให้ตัวแปรนั้น escape ไป heap แม้จะดูเหมือนเป็นค่าธรรมดาที่ควรอยู่บน stack ได้ก็ตาม — นี่คือเหตุผลหนึ่งที่โค้ดที่ log บ่อยมากๆ ในจุดที่ performance สำคัญมาก (hot path) อาจมี allocation มากกว่าที่คาดไว้ ซึ่งเป็นความรู้ที่มีประโยชน์เวลาต้องทำ performance tuning ในภายหลัง

---

## 9. เมื่อไรไม่ควรสู้กับ GC (Premature Optimization)

หลังจากเข้าใจกลไกภายในของ GC และ escape analysis แล้ว อาจเกิดความรู้สึกอยากไป "แก้ไข" ทุกจุดในโค้ดที่ escape ไป heap หรือปรับ `GOGC` ทุกโปรเจกต์ตั้งแต่วันแรก — แต่นี่คือกับดักที่เรียกว่า **premature optimization** (การปรับแต่งประสิทธิภาพก่อนที่จะรู้ว่าจำเป็นจริงหรือไม่)

### หลักการที่ควรยึดถือ

> **"Premature optimization is the root of all evil"** — Donald Knuth

ในบริบทของ Go GC หลักการนี้แปลว่า:

1. **อย่าไล่ไล่แก้ escape analysis ทุกจุดโดยไม่มีข้อมูลสนับสนุน** — โปรแกรม Go ส่วนใหญ่ทำงานได้ดีมากโดยไม่ต้องสนใจเรื่อง heap/stack เลย GC ของ Go ได้รับการออกแบบและปรับจูนมาอย่างดีสำหรับ workload ทั่วไปอยู่แล้ว
2. **วัดผลก่อนปรับแต่ง** — ใช้เครื่องมืออย่าง `pprof` (จะเรียนในภาคที่ 7) เพื่อหาว่า allocation หรือ GC pause จริงๆ แล้วเป็นคอขวด (bottleneck) ของระบบหรือไม่ ก่อนเสียเวลาไปกับการปรับแต่งจุดที่ไม่ได้ส่งผลกระทบต่อ performance โดยรวมเลย
3. **ปรับ `GOGC`/`GOMEMLIMIT` เมื่อมีหลักฐานชัดเจนเท่านั้น** — เช่น เห็นจาก monitoring จริงว่า memory usage สูงเกินคาดหรือ GC ทำงานถี่จนกระทบ latency ไม่ใช่ตั้งค่าตามความรู้สึกหรือเลียนแบบ config ของโปรเจกต์อื่น
4. **โค้ดที่อ่านง่ายมาก่อนเสมอในเบื้องต้น** — การเขียนโค้ดซับซ้อนขึ้นเพื่อบังคับให้ตัวแปรอยู่บน stack (เช่น เปลี่ยนโครงสร้างโค้ดทั้งหมดเพื่อหลีกเลี่ยง interface) มักไม่คุ้มค่ากับความอ่านยากที่เพิ่มขึ้น ยกเว้นในจุดที่ profiling ยืนยันแล้วว่าเป็นคอขวดจริงๆ (hot path ที่ถูกเรียกหลายล้านครั้งต่อวินาที)

### ตัวอย่าง: เมื่อการลด Allocation คุ้มค่าจริง

เพื่อไม่ให้หัวข้อนี้ฟังดูเหมือนบอกว่า "อย่าสนใจเรื่อง allocation เลย" มาดูตัวอย่างที่แสดงให้เห็นว่าการลด allocation **สามารถส่งผลจริงและวัดผลได้** เมื่อทำในจุดที่ถูกต้อง (hot path ที่ถูกเรียกซ้ำจำนวนมาก) โดยใช้ `sync.Pool` (จะเรียนเจาะลึกในภาคที่ 7) เพื่อนำ buffer กลับมาใช้ซ้ำแทนการจัดสรรใหม่ทุกครั้ง:

```go
package main

import (
	"fmt"
	"runtime"
	"sync"
)

var sink []byte

// withoutPool: จัดสรร buffer ใหม่ทุกครั้ง แล้วเก็บ reference ไว้ที่ตัวแปร package-level
// เพื่อบังคับให้ escape ไป heap ทุกครั้ง (จำลองสถานการณ์จริงที่ buffer ถูกใช้ต่อนอกฟังก์ชัน)
func withoutPool(n int) {
	for i := 0; i < n; i++ {
		buf := make([]byte, 4096)
		sink = buf
	}
}

var bufPool = sync.Pool{
	New: func() any {
		return make([]byte, 4096)
	},
}

// withPool: ยืม buffer จาก pool แล้วคืนกลับทันทีที่ใช้เสร็จ แทนการทิ้งให้ GC เก็บกวาด
func withPool(n int) {
	for i := 0; i < n; i++ {
		buf := bufPool.Get().([]byte)
		bufPool.Put(buf)
	}
}

func main() {
	var m1, m2 runtime.MemStats

	runtime.ReadMemStats(&m1)
	withoutPool(200000)
	runtime.ReadMemStats(&m2)
	fmt.Println("without pool, TotalAlloc delta (KB):", (m2.TotalAlloc-m1.TotalAlloc)/1024)

	runtime.ReadMemStats(&m1)
	withPool(200000)
	runtime.ReadMemStats(&m2)
	fmt.Println("with pool, TotalAlloc delta (KB):", (m2.TotalAlloc-m1.TotalAlloc)/1024)
}
```

ผลลัพธ์ตัวอย่าง:

```
without pool, TotalAlloc delta (KB): 800014
with pool, TotalAlloc delta (KB): 4692
```

ความแตกต่างชัดเจนมาก — การใช้ `sync.Pool` ลดปริมาณ heap allocation ลงกว่า 170 เท่าในตัวอย่างนี้ เพราะ buffer ถูกนำกลับมาใช้ซ้ำแทนที่จะขอ memory ใหม่จาก heap ทุกครั้ง ซึ่งหมายถึง GC มีงานน้อยลงมากตามไปด้วย (allocation ที่น้อยลงทำให้ heap โตช้าลง trigger รอบ mark-and-sweep ถี่น้อยลง) เทคนิคนี้เหมาะกับจุดที่มีการจัดสรร object ขนาดเดิมซ้ำๆ ปริมาณมากจริงๆ เช่น buffer สำหรับอ่าน network connection ใน web server ที่รับ request จำนวนมาก **แต่ไม่ใช่สิ่งที่ควรทำแบบเหมารวมทุกจุดในโค้ด** — ควรใช้เมื่อ profiling ยืนยันแล้วว่าจุดนั้นเป็นคอขวดจริง เพราะ `sync.Pool` เองก็มี overhead และความซับซ้อนเพิ่มขึ้น (ต้องระวังเรื่อง state ค้างจากการใช้ครั้งก่อนที่ไม่ได้ล้าง) ไม่คุ้มที่จะใช้ในทุกที่โดยไม่จำเป็น

### เมื่อไรถึงควรใส่ใจเรื่องนี้จริงจัง

การปรับแต่งเรื่อง memory allocation และ GC tuning อย่างจริงจังเหมาะกับสถานการณ์เฉพาะ เช่น ระบบที่ต้องการ latency ต่ำมากระดับ sub-millisecond (เช่น trading system), ระบบที่ประมวลผลข้อมูลปริมาณมหาศาลต่อวินาที, หรือระบบที่รันบน hardware ที่มี memory จำกัดมาก — สำหรับโปรแกรมทั่วไปอย่าง web API, CLI tool, หรือ microservice ขนาดกลาง GC ของ Go ทำงานได้ดีเพียงพอโดยแทบไม่ต้องแตะต้องอะไรเลย

เนื้อหาเรื่องการจัดการ memory เชิงลึกกว่านี้ รวมถึงเทคนิคลด allocation, การใช้ `sync.Pool`, และการอ่านผล `pprof` แบบมืออาชีพ จะเรียนแบบเต็มรูปแบบใน **ภาคที่ 7 (Part 085: Memory Management และ Optimization)**

---

## สรุปสิ่งที่ได้เรียนในบทนี้

- `runtime.NumCPU()`, `runtime.GOMAXPROCS(0)`, `runtime.NumGoroutine()` ใช้ตรวจสอบสภาพแวดล้อมของโปรแกรมที่กำลังรันอยู่ — มีประโยชน์มากสำหรับ debug goroutine leak
- `runtime.ReadMemStats(&m)` อ่านสถิติหน่วยความจำผ่าน struct `runtime.MemStats` (`Alloc`, `TotalAlloc`, `Sys`, `NumGC` ฯลฯ)
- `runtime.GC()` บังคับให้ GC ทำงานทันที แต่แทบไม่จำเป็นในโค้ดจริง เพราะ runtime ตัดสินใจจังหวะที่เหมาะสมได้ดีอยู่แล้ว
- Go GC ใช้อัลกอริทึม **concurrent tri-color mark-and-sweep** — แบ่ง object เป็นสีขาว/เทา/ดำระหว่างขั้นตอน mark เพื่อให้ทำงานสลับกับโปรแกรมจริงได้อย่างปลอดภัยผ่านกลไก write barrier
- Go ออกแบบ GC ให้เน้น **low latency** มากกว่า throughput เพราะเหมาะกับ network server ที่ต้องการ response time สั้นและสม่ำเสมอ ทำให้ pause time ปัจจุบันมักต่ำกว่า 1 มิลลิวินาที
- **`GOGC`** (default 100) ควบคุมความถี่ของ GC ผ่านอัตราส่วนการโตของ heap ส่วน **`GOMEMLIMIT`** (Go 1.19+) กำหนดเพดาน memory รวม ทั้งสองใช้ร่วมกันได้และควรปรับเมื่อมีหลักฐานจาก production เท่านั้น
- `go build -gcflags="-m"` แสดงผลลัพธ์ escape analysis ได้ละเอียด — การส่งค่าผ่าน interface parameter (เช่น `fmt.Println`) มักทำให้ตัวแปร escape ไป heap แม้จะดูเป็นค่าธรรมดา
- **อย่าสู้กับ GC ก่อนมีข้อมูล** — วัดผลด้วย `pprof` ก่อนปรับแต่งเสมอ โค้ดที่อ่านง่ายมาก่อน ยกเว้นในจุดที่พิสูจน์แล้วว่าเป็นคอขวดจริง

## แบบฝึกหัดท้ายบท

1. เขียนโปรแกรมที่พิมพ์ `runtime.NumGoroutine()` ทุก 1 วินาที (ใช้ `time.Sleep` วนลูป) ขณะที่สร้าง goroutine ใหม่ 10 ตัวทุกรอบโดย**ไม่ปิด channel ที่มันรออยู่เลย** สังเกตว่าตัวเลขไต่สูงขึ้นเรื่อยๆ อย่างไร (สาธิต goroutine leak ที่เรียนไปในบทที่ 036)
2. เขียนโปรแกรมที่เรียก `runtime.ReadMemStats` ก่อนและหลังการสร้าง slice ขนาดใหญ่ (เช่น `make([]int, 10_000_000)`) แล้วเปรียบเทียบค่า `Alloc` ก่อน-หลัง อธิบายความแตกต่างที่เห็น
3. ทดลองรันโปรแกรมเดียวกันด้วยค่า `GOGC` ที่ต่างกัน (`GOGC=50`, `GOGC=100` (default), `GOGC=400`) พร้อมพิมพ์ `m.NumGC` หลังทำงานเสร็จ เปรียบเทียบว่าจำนวนรอบ GC เปลี่ยนไปอย่างไร
4. เขียนฟังก์ชันสองแบบที่ทำงานเหมือนกัน (เช่น สร้างและคืนค่า struct) แบบแรกคืนค่าเป็น pointer แบบที่สองคืนค่าเป็น value แล้วรัน `go build -gcflags="-m"` เปรียบเทียบว่า escape analysis ตัดสินใจต่างกันอย่างไร
5. ค้นคว้าและทดลองใช้ `debug.SetMemoryLimit` จาก package `runtime/debug` (เทียบเท่า `GOMEMLIMIT` แต่ตั้งจากในโค้ด) เขียนโปรแกรมที่ตั้งเพดานไว้ต่ำๆ แล้วจัดสรร memory จำนวนมากเพื่อสังเกตว่า GC ทำงานถี่ขึ้นจริงหรือไม่ (สังเกตผ่าน `m.NumGC`)
6. ค้นคว้าเพิ่มเติมเกี่ยวกับ `GODEBUG=gctrace=1` ซึ่งเป็น environment variable ที่ทำให้ Go runtime พิมพ์รายละเอียดของ GC แต่ละรอบออกทาง stderr แบบ real-time (เวลาที่ใช้, ขนาด heap ก่อน-หลัง, จำนวน goroutine ที่ช่วย mark) ลองรันโปรแกรมที่จัดสรร memory เยอะๆ ด้วย flag นี้แล้ววิเคราะห์ผลลัพธ์ที่ได้

---

**ต่อไป**: [Part 056 — HTTP Server ด้วย `net/http`: Routing, Handler](./056-http-server-routing.md)
