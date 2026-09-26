# Part 044: Race Condition และ Race Detector

> ภาคที่ 3: การทำงานพร้อมกัน (Concurrency) — ตอนที่ 9 จาก 10 (Part 36–45)

## สารบัญของบทนี้

1. Data Race คืออะไรกันแน่
2. ทำไม Data Race ถึงอันตรายกว่าที่คิด: ผลลัพธ์ "ดูเหมือนถูกต้อง" ก็ยังเป็นบั๊ก
3. สาธิต: เขียนโปรแกรมที่มี Data Race จริง แล้ววัดผลลัพธ์ที่ไม่แน่นอน
4. Go Race Detector คืออะไร ทำงานอย่างไร (แนวคิดเบื้องหลัง)
5. วิธีใช้งาน: `go run -race`, `go test -race`, `go build -race`
6. อ่านผลลัพธ์ของ Race Detector ทีละส่วน
7. แก้ไข Race Condition ด้วย Mutex แล้วยืนยันด้วย `-race` อีกครั้ง
8. Race บน map: เมื่อ Go หยุดโปรแกรมให้เองโดยไม่ต้องมี `-race`
9. ตรวจจับ Race ใน Unit Test ด้วย `go test -race`
10. ข้อจำกัดของ Race Detector ที่ต้องรู้
11. รัน `-race` ใน CI ทุกครั้งที่ทดสอบ (ทบทวนก่อน Part 098)
12. สรุปสิ่งที่ได้เรียนในบทนี้
13. แบบฝึกหัดท้ายบท

---

## 1. Data Race คืออะไรกันแน่

นิยามที่แม่นยำที่สุดของ **data race** (บางครั้งเรียกสั้นๆ ว่า race condition แม้สองคำนี้จะไม่ตรงกันเป๊ะในทางทฤษฎี):

> **Data race เกิดขึ้นเมื่อมี goroutine ตั้งแต่ 2 ตัวขึ้นไป เข้าถึงตัวแปร (หน่วยความจำ) เดียวกัน "พร้อมกัน" โดยที่อย่างน้อยหนึ่งในการเข้าถึงนั้นเป็นการ "เขียน" (write) และไม่มีกลไก synchronization ใดๆ (เช่น mutex, channel) มาควบคุมลำดับการเข้าถึงนั้น**

สังเกตเงื่อนไข 3 ข้อที่ต้องครบทั้งหมดถึงจะเป็น data race:

1. มีการเข้าถึงหน่วยความจำเดียวกันจากมากกว่าหนึ่ง goroutine
2. อย่างน้อยหนึ่งในนั้นเป็นการ **เขียน**
3. ไม่มี **happens-before relationship** ระหว่างการเข้าถึงเหล่านั้น (คือไม่มีอะไรรับประกันว่าการเข้าถึงครั้งหนึ่งจะเกิดขึ้น "ก่อน" หรือ "หลัง" อีกครั้งหนึ่งอย่างแน่นอน)

ถ้าทุก goroutine แค่ **อ่าน** ตัวแปรร่วมกันโดยไม่มีใครเขียนเลย จะไม่เป็น data race (อ่านพร้อมกันปลอดภัยเสมอ) และถ้ามีการเขียนแต่ใช้ mutex ป้องกันไว้อย่างถูกต้อง (ทำให้การเข้าถึงแต่ละครั้งมี happens-before ต่อกัน) ก็จะไม่เป็น data race เช่นกัน

---

## 2. ทำไม Data Race ถึงอันตรายกว่าที่คิด: ผลลัพธ์ "ดูเหมือนถูกต้อง" ก็ยังเป็นบั๊ก

จุดที่ทำให้ data race เป็นบั๊กที่อันตรายที่สุดประเภทหนึ่งในโปรแกรมมิ่งคือ: **มันไม่ได้ทำให้โปรแกรม crash หรือให้ผลลัพธ์ผิดเสมอไป** บางครั้งรันสิบครั้งได้ผลลัพธ์ถูกต้องทั้งสิบครั้ง แต่พอเอาไปรันบนเครื่อง production ที่มีจำนวน CPU core ต่างกัน, ภายใต้ load ที่หนักกว่า, หรือคอมไพล์ด้วย Go เวอร์ชันที่ต่างกัน (ซึ่งอาจปรับ scheduler หรือทำ compiler optimization ต่างออกไป) ผลลัพธ์อาจผิดทันที

เหตุผลเชิงเทคนิคที่ลึกกว่านั้นคือ: เมื่อไม่มี synchronization กำกับ **compiler และ CPU มีสิทธิ์เรียงลำดับคำสั่ง (reorder) หรือทำ optimization ใดๆ ก็ได้กับโค้ดที่เข้าถึงตัวแปรนั้น** ตราบใดที่ผลลัพธ์ภายใน goroutine เดียวกันยังถูกต้องตาม sequential semantics นี่หมายความว่า data race ไม่ใช่แค่ "อาจมีปัญหาเรื่อง timing" แต่เป็น **undefined behavior** ในเชิงภาษา — โมเดลหน่วยความจำของ Go (Go Memory Model) ไม่รับประกันผลลัพธ์ใดๆ เลยเมื่อเกิด data race แม้แต่ผลลัพธ์ที่ "ดูสมเหตุสมผล" ก็อาจใช้ไม่ได้ในรันครั้งถัดไป

นี่คือเหตุผลที่ **การทดสอบด้วยตาเปล่า (รันแล้วดูว่าผลลัพธ์ถูกไหม) ไม่เพียงพอที่จะยืนยันว่าโค้ด concurrent ปลอดภัย** ต้องใช้เครื่องมือเฉพาะทางอย่าง race detector มาช่วยเสมอ

---

## 3. สาธิต: เขียนโปรแกรมที่มี Data Race จริง แล้ววัดผลลัพธ์ที่ไม่แน่นอน

มาดูตัวอย่างคลาสสิกที่สุดของ data race: หลาย goroutine เพิ่มค่าตัวแปรร่วมกันโดยไม่มีการป้องกัน

```go
package main

import (
	"fmt"
	"sync"
)

// counter คือตัวแปรธรรมดา ไม่มีการป้องกันใดๆ เลย
var counter int

func increment(wg *sync.WaitGroup) {
	defer wg.Done()
	for i := 0; i < 1000; i++ {
		counter++ // อ่าน-บวก-เขียน 3 ขั้นตอน ไม่ใช่ operation เดียวระดับ hardware
	}
}

func main() {
	var wg sync.WaitGroup
	for i := 0; i < 10; i++ {
		wg.Add(1)
		go increment(&wg)
	}
	wg.Wait()
	fmt.Println("ค่า counter สุดท้าย:", counter)
}
```

ทำไม `counter++` ถึงไม่ปลอดภัย? เพราะในระดับเครื่อง (CPU instruction) คำสั่งนี้ไม่ได้เป็น operation เดียว แต่แยกเป็น 3 ขั้นตอน: **อ่าน**ค่าปัจจุบันของ `counter` เข้า register → **บวก** 1 เข้า register → **เขียน**ค่าใน register กลับไปที่ `counter` ถ้า goroutine สองตัวทำสามขั้นตอนนี้ "สลับกัน" (เช่น ทั้งคู่อ่านค่าเดิมก่อนที่ใครจะเขียนค่าใหม่กลับไป) การเพิ่มค่าครั้งหนึ่งจะหายไปเงียบๆ

ลองรันโปรแกรมนี้แบบปกติ (ไม่ใช้ `-race`) ซ้ำหลายครั้ง:

```bash
go run race_bad.go
```

ผลลัพธ์จากการรันจริงสามครั้งติดกันบนเครื่องเดียวกัน:

```
ค่า counter สุดท้าย: 10000
ค่า counter สุดท้าย: 10000
ค่า counter สุดท้าย: 7206
```

นี่คือหลักฐานที่ชัดเจนที่สุดของอันตรายที่อธิบายไว้ในหัวข้อ 2: **สองครั้งแรกได้ค่าที่ "ดูถูกต้อง" (10 goroutine × 1000 ครั้ง = 10000 พอดี) แต่ครั้งที่สามได้ 7206 ซึ่งน้อยกว่าที่ควรเป็น** เพราะการเพิ่มค่าบางครั้งถูก "เขียนทับ" หายไประหว่างที่ goroutine อื่นแทรกเข้ามา ถ้าทีมพัฒนาทดสอบแค่สองสามครั้งแรกและเห็นค่า 10000 ก็อาจสรุปผิดว่าโค้ดนี้ "ใช้งานได้" ทั้งที่จริงมันมี data race ซ่อนอยู่ ซึ่งจะแสดงอาการออกมาเมื่อไรก็ได้โดยไม่มีการเตือนล่วงหน้า

---

## 4. Go Race Detector คืออะไร ทำงานอย่างไร (แนวคิดเบื้องหลัง)

Go มีเครื่องมือในตัวชื่อ **race detector** ที่ built-in มากับ toolchain ตั้งแต่ Go 1.1 ใช้เทคโนโลยีจาก Google ชื่อ [ThreadSanitizer (TSan)](https://github.com/google/sanitizers/wiki/ThreadSanitizerCppManual) ซึ่งเดิมพัฒนาสำหรับ C/C++ แล้วนำมาปรับใช้กับ Go

หลักการทำงานเบื้องหลัง (ไม่ต้องเข้าใจลึกระดับ implementation แต่ควรรู้แนวคิด):

- เมื่อคอมไพล์ด้วย flag `-race` compiler จะ **แทรกโค้ดตรวจสอบ (instrumentation)** เข้าไปในทุกจุดที่มีการอ่าน/เขียนหน่วยความจำที่ share กันได้ และทุกจุดที่มีการ synchronize กัน (channel operation, mutex lock/unlock, goroutine start ฯลฯ)
- race detector ใช้แนวคิดที่เรียกว่า **vector clock** (หรือใกล้เคียง: happens-before tracking) เพื่อติดตามว่า การเข้าถึงหน่วยความจำแต่ละครั้งเกิดขึ้น "ก่อน" หรือ "หลัง" การเข้าถึงอื่นๆ อย่างแน่นอนหรือไม่ (มี happens-before relationship หรือไม่) โดยแต่ละ goroutine จะมีนาฬิกาลอจิก (logical clock) ของตัวเอง ที่จะถูก "ซิงค์" กันทุกครั้งที่มีการสื่อสารผ่าน channel หรือ mutex
- ถ้าพบการเข้าถึงหน่วยความจำเดียวกันสองครั้งที่ **ไม่มี** happens-before ระหว่างกัน และอย่างน้อยหนึ่งครั้งเป็นการเขียน — นั่นคือ data race และ detector จะรายงานทันทีพร้อม stack trace ของทั้งสองจุดที่ชนกัน

ข้อดีของแนวทางนี้คือ **race detector ตรวจจับ data race ที่เกิดขึ้นจริงระหว่างการรัน (runtime) ไม่ใช่การเดาแบบ static analysis** ทำให้แม่นยำมาก (แทบไม่มี false positive) แต่ก็มีข้อจำกัดที่ต้องรู้เช่นกัน (ดูหัวข้อ 8)

ต้นทุนของการเปิด race detector: โปรแกรมจะทำงาน**ช้าลง 2-20 เท่า** และใช้หน่วยความจำมากขึ้น 5-10 เท่า เพราะต้องแทรกโค้ดตรวจสอบทุกจุด — ด้วยเหตุนี้ `-race` จึงเหมาะกับการใช้ตอน **พัฒนาและทดสอบ** เท่านั้น ไม่ควรใช้ compile binary สำหรับ production

---

## 5. วิธีใช้งาน: `go run -race`, `go test -race`, `go build -race`

Race detector เปิดใช้งานผ่าน flag `-race` ที่ใช้ร่วมกับคำสั่งหลักของ `go` toolchain ได้ทั้งหมด:

```bash
go run -race main.go        # compile + รัน พร้อมเปิด race detector
go build -race -o app main.go  # คอมไพล์เป็น binary ที่มี race detector ฝังอยู่
go test -race ./...         # รัน test ทั้งหมดพร้อมเปิด race detector (ใช้บ่อยที่สุดในทางปฏิบัติ)
```

ข้อกำหนด: `-race` ต้องการ `CGO_ENABLED=1` (ค่า default อยู่แล้วบนเครื่องส่วนใหญ่ที่มี C toolchain ติดตั้งไว้) และรองรับเฉพาะบาง platform เท่านั้น (linux/amd64, linux/arm64, darwin/amd64, darwin/arm64, windows/amd64 เป็นหลัก) — เช็ครายละเอียดล่าสุดได้ที่เอกสารทางการของ Go

ในทางปฏิบัติ **ทีมพัฒนา Go ที่ดีเกือบทุกทีมจะรัน `go test -race ./...` เป็นส่วนหนึ่งของ CI pipeline เสมอ** (เราจะติดตั้งจริงใน Part 098 — CI/CD ด้วย GitHub Actions) เพราะการรัน test แบบปกติ (ไม่มี `-race`) อาจผ่านฉลุยทั้งที่มี data race ซ่อนอยู่ ดังที่เห็นในหัวข้อ 3

---

## 6. อ่านผลลัพธ์ของ Race Detector ทีละส่วน

มารันตัวอย่างในหัวข้อ 3 ด้วย `-race`:

```bash
go run -race race_bad.go
```

ผลลัพธ์จริงที่ได้ (จัดรูปแบบให้อ่านง่าย):

```
==================
WARNING: DATA RACE
Read at 0x0000006006d0 by goroutine 8:
  main.increment()
      /home/user/race-demo/race_bad.go:14 +0x91
  main.main.gowrap1()
      /home/user/race-demo/race_bad.go:22 +0x33

Previous write at 0x0000006006d0 by goroutine 15:
  main.increment()
      /home/user/race-demo/race_bad.go:14 +0xa9
  main.main.gowrap1()
      /home/user/race-demo/race_bad.go:22 +0x33

Goroutine 8 (running) created at:
  main.main()
      /home/user/race-demo/race_bad.go:22 +0x56

Goroutine 15 (finished) created at:
  main.main()
      /home/user/race-demo/race_bad.go:22 +0x56
==================
ค่า counter สุดท้าย: 10000
Found 1 data race(s)
exit status 66
```

มาแกะทีละส่วน:

### `WARNING: DATA RACE`

บรรทัดแรกบอกชัดเจนว่าเจอ data race หนึ่งจุด (แต่ละจุดที่ตรวจพบจะห่อด้วย `==================` คั่นไว้ ถ้ามีหลายจุดจะมีหลายบล็อก)

### `Read at 0x0000006006d0 by goroutine 8:`

- `0x0000006006d0` คือ**ที่อยู่หน่วยความจำ**ของตัวแปรที่เกิดปัญหา (ในที่นี้คือ `counter`)
- **goroutine 8** คือหมายเลขที่ Go runtime ตั้งให้ (ไม่ใช่ ID ที่ผู้เขียนโปรแกรมกำหนดเอง แต่เป็นเลขที่ runtime สร้างขึ้นเพื่อระบุตัวตนของแต่ละ goroutine ในรายงานนี้)
- Stack trace ด้านล่าง (`main.increment()` ที่บรรทัด `race_bad.go:14`) บอกว่า goroutine นี้กำลัง**อ่าน** ค่าจาก `counter` ที่บรรทัด 14 ซึ่งตรงกับ `counter++` พอดี (เพราะการอ่านค่าเดิมคือขั้นตอนแรกของ `++`)

### `Previous write at 0x0000006006d0 by goroutine 15:`

- นี่คือครึ่งที่สองของการชนกัน: **goroutine 15** เขียนค่าลงตำแหน่งความจำเดียวกัน **ก่อนหน้า** (จึงใช้คำว่า "Previous write") ที่บรรทัดเดียวกัน (14) — เพราะขั้นตอนเขียนกลับของ `++` ก็อยู่ที่บรรทัดเดิม
- คำว่า "Previous" สำคัญมาก: มันบอกว่า detector เจอลำดับเหตุการณ์ (อาจจะ) เป็น "goroutine 15 เขียน แล้ว goroutine 8 อ่านค่าที่ตำแหน่งเดียวกัน" แต่ **ไม่มี happens-before relationship ใดๆ รับประกันลำดับนี้** จึงถือเป็น race แม้ในการรันจริงลำดับเวลาจะเป็นแบบนี้ก็ตาม (การรันครั้งถัดไปอาจสลับกัน)

### `Goroutine 8 (running) created at:` / `Goroutine 15 (finished) created at:`

ส่วนสุดท้ายบอกว่าแต่ละ goroutine ที่เกี่ยวข้องถูกสร้าง (ด้วยคำสั่ง `go`) จากที่ไหน — ในที่นี้ทั้งคู่ถูกสร้างที่บรรทัด 22 (`go increment(&wg)`) ซึ่งสมเหตุสมผลเพราะเราเปิด `increment` เป็น goroutine 10 ตัวจากจุดเดียวกันในลูป สถานะ `(running)` กับ `(finished)` บอกว่า ณ ขณะที่ detector รายงาน goroutine ตัวนั้นกำลังทำงานอยู่หรือจบไปแล้ว

### `Found 1 data race(s)` และ `exit status 66`

Go race detector จะทำให้โปรแกรม**จบด้วย exit code 66** โดยอัตโนมัติเมื่อพบ data race อย่างน้อยหนึ่งจุด (ไม่ว่าโปรแกรมจะรันจบตามปกติหรือไม่ก็ตาม) นี่คือเหตุผลที่ `go test -race` ใน CI จะรายงานว่า **test ล้มเหลว** ทันทีที่เจอ data race แม้ assertion ทั้งหมดในเทสต์จะผ่านก็ตาม — exit code ที่ไม่ใช่ 0 ทำให้ CI pipeline หยุดและแจ้งเตือนทีมได้ทันที

---

## 7. แก้ไข Race Condition ด้วย Mutex แล้วยืนยันด้วย `-race` อีกครั้ง

การแก้ไขตรงไปตรงมาที่สุดคือใช้ `sync.Mutex` (ทบทวนจาก Part 039) ล้อมรอบทุกจุดที่เข้าถึงตัวแปรร่วม:

```go
package main

import (
	"fmt"
	"sync"
)

var (
	counter int
	mu      sync.Mutex
)

func increment(wg *sync.WaitGroup) {
	defer wg.Done()
	for i := 0; i < 1000; i++ {
		mu.Lock()
		counter++
		mu.Unlock()
	}
}

func main() {
	var wg sync.WaitGroup
	for i := 0; i < 10; i++ {
		wg.Add(1)
		go increment(&wg)
	}
	wg.Wait()
	fmt.Println("ค่า counter สุดท้าย:", counter)
}
```

รันด้วย `go run -race race_fixed.go` ซ้ำสามครั้งเพื่อยืนยันความสม่ำเสมอ:

```
ค่า counter สุดท้าย: 10000
ค่า counter สุดท้าย: 10000
ค่า counter สุดท้าย: 10000
```

ไม่มี `WARNING: DATA RACE` ปรากฏเลยแม้แต่ครั้งเดียว และค่าที่ได้ถูกต้องตรงกันทุกครั้ง (10000 พอดี) — นี่คือสัญญาณที่แสดงว่าโค้ดปลอดภัยแล้ว

**ทำไม mutex แก้ปัญหาได้**: `mu.Lock()` และ `mu.Unlock()` สร้าง happens-before relationship ระหว่างการเข้าถึงของแต่ละ goroutine อย่างชัดเจน — เมื่อ goroutine หนึ่งเรียก `mu.Unlock()` แล้ว goroutine ถัดไปที่เรียก `mu.Lock()` สำเร็จ รับประกันว่าจะเห็นผลของการเขียนทั้งหมดที่เกิดขึ้นก่อนหน้า `Unlock()` นั้น (ตาม Go Memory Model) การเข้าถึง `counter` ทุกครั้งจึงถูก "เรียงคิว" กันอย่างชัดเจน ไม่มีการอ่าน/เขียนที่ทับซ้อนกันแบบไม่มีลำดับอีกต่อไป

ทางเลือกอื่นที่แก้ปัญหาเดียวกันได้ (ทบทวนจาก Part 040): ใช้ `atomic.Int64` แทน `int` ธรรมดา ซึ่งเหมาะกับกรณีที่มีแค่ operation ง่ายๆ อย่างการนับค่า (เร็วกว่า mutex เล็กน้อยเพราะไม่มี lock/unlock overhead) แต่ mutex ยืดหยุ่นกว่าเมื่อต้องป้องกัน logic ที่ซับซ้อนกว่าการอ่าน/เขียนค่าเดี่ยวๆ

---

## 8. Race บน map: เมื่อ Go หยุดโปรแกรมให้เองโดยไม่ต้องมี `-race`

ตัวอย่างในหัวข้อ 3 เป็น data race บนตัวแปร `int` ธรรมดา ซึ่ง Go runtime **ไม่มีทางรู้เองว่าเกิด race** ถ้าไม่เปิด `-race` โปรแกรมจะรันต่อไปเงียบๆ พร้อมค่าที่อาจผิด แต่ `map` เป็นกรณีพิเศษ: **Go runtime มีกลไกตรวจจับการเขียน map พร้อมกันในตัว** (ทบทวนจาก Part 007 ว่า `map` ไม่ safe สำหรับการเขียนพร้อมกัน) และจะทำให้โปรแกรม **panic ทันทีแบบ fatal error** แม้จะไม่ได้เปิด `-race` เลยก็ตาม

```go
package main

import (
	"fmt"
	"sync"
)

func main() {
	m := make(map[int]int)

	var wg sync.WaitGroup
	for i := 0; i < 20; i++ {
		wg.Add(1)
		go func(n int) {
			defer wg.Done()
			m[n] = n * n // เขียน map เดียวกันจากหลาย goroutine พร้อมกัน โดยไม่มี mutex
		}(i)
	}
	wg.Wait()

	fmt.Println("map มีสมาชิกทั้งหมด:", len(m))
}
```

รันโปรแกรมนี้ซ้ำหลายครั้งแบบธรรมดา (ไม่ใช้ `-race` เลย):

```bash
go run map_race.go
```

ผลลัพธ์จากการรันจริง 4 ครั้งติดกัน:

```
map มีสมาชิกทั้งหมด: 20
```

```
fatal error: concurrent map writes

goroutine 24 [running]:
internal/runtime/maps.fatal({0x4b8163?, 0x0?})
	/usr/local/go1.24.7/src/runtime/panic.go:1058 +0x18
main.main.func1(0x6)
	/home/user/race-demo/map_race.go:16 +0x56
created by main.main in goroutine 1
	/home/user/race-demo/map_race.go:14 +0x45
exit status 2
```

```
fatal error: concurrent map writes
...
exit status 2
```

```
fatal error: concurrent map writes
...
exit status 2
```

จาก 4 ครั้ง มีเพียงครั้งแรกที่ "รอด" (เพราะจังหวะการสลับ goroutine บังเอิญไม่ชนกัน) ส่วนอีก 3 ครั้งโปรแกรม crash ทันทีด้วย **`fatal error: concurrent map writes`** จุดที่ต้องสังเกตให้ดี:

- นี่คือ **fatal error ของ runtime ไม่ใช่ panic ธรรมดา** — สังเกตว่าไม่มีคำว่า `panic:` นำหน้า และที่สำคัญกว่านั้นคือ **`recover()` ดักจับ fatal error แบบนี้ไม่ได้** (ต่างจาก panic ทั่วไปที่ Part 017 สอนให้ใช้ `recover()` ดักจับได้) เพราะ Go runtime มองว่านี่คือสถานะที่เสียหายเกินกว่าจะกู้คืนโปรแกรมต่อได้อย่างปลอดภัย โปรแกรมจึงถูกบังคับให้จบทันทีเสมอ
- **ผลลัพธ์ไม่แน่นอน** (nondeterministic) เหมือนกับตัวอย่างตัวนับใน หัวข้อ 3 เป๊ะๆ — บางครั้งรอด บางครั้ง crash ทั้งที่โค้ดเหมือนกันทุกตัวอักษร นี่คือธรรมชาติของ data race ที่ต้องจำให้ขึ้นใจ
- การที่ Go จับ concurrent map write ได้เองเป็นเพียง **กรณีพิเศษของ `map` เท่านั้น** ไม่ได้แปลว่า Go ตรวจจับ data race ได้ทุกกรณีโดยอัตโนมัติ — ตัวแปรชนิดอื่น (`int`, `struct`, `slice` ฯลฯ) ไม่มีกลไกป้องกันตัวเองแบบนี้เลย นี่คือเหตุผลที่ **ต้องพึ่ง `-race` เสมอ ไม่ใช่หวังพึ่ง fatal error ของ runtime**

การแก้ไขคือใช้ `sync.Mutex` หรือ `sync.RWMutex` ล้อมรอบการเข้าถึง `map` เหมือนตัวอย่าง `Broker` ใน Part 045 หรือใช้ `sync.Map` (โครงสร้างข้อมูลพิเศษจาก package `sync` ที่ออกแบบมาสำหรับใช้งานแบบ concurrent โดยเฉพาะ เหมาะกับกรณีที่ key ส่วนใหญ่ถูกเขียนครั้งเดียวแล้วอ่านซ้ำบ่อยๆ)

---

## 9. ตรวจจับ Race ใน Unit Test ด้วย `go test -race`

ในทางปฏิบัติ เราแทบไม่เขียน `func main()` เพื่อทดสอบ concurrency แบบตรงๆ แต่จะเขียนเป็น unit test (ทบทวนจาก Part 033-034) แล้วรันด้วย `go test -race` ซึ่งเป็นรูปแบบที่ใช้จริงใน CI ทุกวัน

ตัวอย่าง: `Cache` แบบง่ายที่ยังไม่ได้ป้องกัน concurrency

```go
package cache

// Cache คือตัวอย่าง struct ธรรมดาที่ยังไม่ได้ป้องกัน concurrency
type Cache struct {
	data map[string]int
}

func NewCache() *Cache {
	return &Cache{data: make(map[string]int)}
}

func (c *Cache) Set(key string, value int) {
	c.data[key] = value
}

func (c *Cache) Get(key string) int {
	return c.data[key]
}
```

พร้อม test ที่จำลองการใช้งานพร้อมกันจากหลาย goroutine (เช่น หลาย HTTP request เข้ามาพร้อมกันเรียก `Cache` ตัวเดียวกัน):

```go
package cache

import (
	"sync"
	"testing"
)

// TestCacheConcurrent จำลองการใช้งาน Cache จากหลาย goroutine พร้อมกัน (เช่น หลาย request
// เข้ามาพร้อมกันใน HTTP server) ถ้ารันด้วย go test -race จะเจอ data race ทันที
func TestCacheConcurrent(t *testing.T) {
	cache := NewCache()

	var wg sync.WaitGroup
	for i := 0; i < 50; i++ {
		wg.Add(1)
		go func(n int) {
			defer wg.Done()
			cache.Set("key", n)
			_ = cache.Get("key")
		}(i)
	}
	wg.Wait()
}
```

รันด้วย:

```bash
go test -race ./...
```

ผลลัพธ์ (ตัดมาบางส่วนเพราะรายงานหลายจุดชนกัน):

```
==================
WARNING: DATA RACE
Write at 0x00c00011e600 by goroutine 8:
  runtime.mapassign_faststr()
      /usr/local/go/src/internal/runtime/maps/runtime_faststr_swiss.go:263 +0x0
  cache.(*Cache).Set()
      /home/user/race-demo/cache_test.go:18 +0xb4
  cache.TestCacheConcurrent.func1()
      /home/user/race-demo/cache_test.go:35 +0x8f
  cache.TestCacheConcurrent.gowrap1()
      /home/user/race-demo/cache_test.go:37 +0x41

Previous write at 0x00c00011e600 by goroutine 16:
  runtime.mapassign_faststr()
      ...
==================
--- FAIL: TestCacheConcurrent (0.02s)
FAIL
```

สังเกตว่า stack trace คราวนี้มี **สาม** เฟรมแทนที่จะมีสองเฟรมเหมือนตัวอย่างตัวนับ: `runtime.mapassign_faststr()` (ฟังก์ชันภายในของ runtime ที่จัดการเขียนค่าลง map) → `cache.(*Cache).Set()` (เมธอดของเราที่เรียก `c.data[key] = value`) → `cache.TestCacheConcurrent.func1()` (closure ที่เราเปิดเป็น goroutine ใน test) — การอ่าน stack trace แบบไล่จากบนลงล่างแบบนี้ช่วยให้เห็นชัดว่า "ตัวการ" ที่แท้จริงคือเมธอด `Set` ของเราเอง แม้ตัว error จะโผล่มาจากโค้ดภายใน runtime ก็ตาม

หลังแก้ด้วย `sync.RWMutex` (แบบเดียวกับ `Broker` ใน Part 045):

```go
type SafeCache struct {
	mu   sync.RWMutex
	data map[string]int
}

func (c *SafeCache) Set(key string, value int) {
	c.mu.Lock()
	defer c.mu.Unlock()
	c.data[key] = value
}

func (c *SafeCache) Get(key string) int {
	c.mu.RLock()
	defer c.mu.RUnlock()
	return c.data[key]
}
```

รันซ้ำด้วย `go test -race -v ./...` ผลลัพธ์:

```
=== RUN   TestSafeCacheConcurrent
--- PASS: TestSafeCacheConcurrent (0.00s)
PASS
ok  	cache	1.019s
```

`PASS` ที่ได้นี้มีความหมายมากกว่าการที่ assertion ผ่าน — มันหมายความว่า **ตลอดการรัน test ครั้งนี้ ไม่มี goroutine คู่ไหนเข้าถึงหน่วยความจำเดียวกันโดยไม่มี happens-before ระหว่างกันเลย** นี่คือเหตุผลที่ทีมพัฒนาที่ใช้ Go อย่างจริงจังถือว่า `go test -race` เป็นส่วนหนึ่งของคำนิยาม "test ผ่าน" ไม่ใช่แค่ทางเลือกเสริม

---

## 10. ข้อจำกัดของ Race Detector ที่ต้องรู้

แม้ race detector จะทรงพลังมาก แต่ก็มีข้อจำกัดสำคัญที่ต้องเข้าใจ:

- **ตรวจจับได้เฉพาะ race ที่เกิดขึ้นจริงระหว่างการรันนั้นๆ**: ถ้า code path ที่มี race ไม่ถูก execute ในการรันครั้งนั้น (เช่น เงื่อนไขบางอย่างไม่เกิดขึ้น, ไม่มี load เพียงพอที่จะทำให้ goroutine สองตัวชนกันจริง) detector จะไม่รายงานอะไรเลย แม้ race จะยังซ่อนอยู่ในโค้ด — **การรันผ่าน `-race` ไม่ได้แปลว่าโค้ดปลอดจาก race 100% รับประกันได้แค่ว่า "ในการรันที่ทดสอบนี้ ไม่พบ race"**
- **ควรรัน test ให้ครอบคลุม concurrency ทุกเส้นทางที่เป็นไปได้**: เขียน test ที่จงใจสร้างสถานการณ์แข่งขันสูง (เช่น เปิด goroutine จำนวนมาก, ใช้ `-race` ร่วมกับ `go test -count=10` เพื่อรันซ้ำหลายรอบ) เพื่อเพิ่มโอกาสให้ detector เจอ race ที่ซ่อนอยู่
- **ค่าใช้จ่ายด้าน performance สูง**: อย่างที่กล่าวไปในหัวข้อ 4 ห้ามใช้ binary ที่ build ด้วย `-race` ใน production
- **ไม่ตรวจจับ logic race** (การแข่งขันเชิงตรรกะ เช่น ลำดับเหตุการณ์ทางธุรกิจที่ผิดแม้ไม่มีการชนกันของหน่วยความจำ) — race detector ตรวจจับเฉพาะ **data race ระดับหน่วยความจำ** เท่านั้น ไม่ใช่บั๊ก concurrency ทุกประเภท

---

## 11. รัน `-race` ใน CI ทุกครั้งที่ทดสอบ (ทบทวนก่อน Part 098)

จากทุกอย่างที่เรียนมาในบทนี้ สรุปเป็นแนวปฏิบัติที่ควรยึดถือในทุกโปรเจกต์ Go ที่มีโค้ด concurrent:

> **ทุก CI pipeline ที่รัน `go test` ควรรันด้วย flag `-race` เสมอ ไม่มีข้อยกเว้น** (`go test -race ./...`)

เหตุผลสรุปสั้นๆ:

- Data race เป็นบั๊กที่ตรวจจับด้วยตาเปล่าไม่ได้ (หัวข้อ 2-3)
- Race detector ตรวจจับได้แม่นยำและให้ข้อมูลละเอียดพอที่จะแก้ปัญหาได้ทันที (หัวข้อ 6)
- ต้นทุนด้าน performance ระหว่างทดสอบไม่ใช่ปัญหา เพราะไม่ได้เอาไปรันจริงใน production
- ยิ่งเจอ race เร็วเท่าไร (ตั้งแต่ pull request) ยิ่งแก้ง่ายและถูกกว่าการไปเจอตอน production ล่มโดยไม่ทราบสาเหตุ

เราจะตั้งค่า GitHub Actions ให้รัน `go test -race ./...` เป็นขั้นตอนบังคับในทุก pull request อย่างละเอียดใน **Part 098: CI/CD Pipeline ด้วย GitHub Actions** — บทนี้เป็นการปูพื้นฐานความเข้าใจว่า "ทำไม" ขั้นตอนนั้นถึงสำคัญมาก

---

## สรุปสิ่งที่ได้เรียนในบทนี้

- **Data race** คือการเข้าถึงหน่วยความจำเดียวกันจากหลาย goroutine พร้อมกันโดยไม่มี synchronization ซึ่งอย่างน้อยหนึ่งการเข้าถึงเป็นการเขียน
- Data race อันตรายเพราะเป็น undefined behavior — ผลลัพธ์อาจ "ดูถูกต้อง" ในบางการรัน แต่ผิดในการรันอื่น ทำให้ตรวจจับด้วยการทดสอบทั่วไปไม่ได้ ต้องพึ่งเครื่องมือเฉพาะ
- Go มี **race detector** ในตัว (ใช้เทคโนโลยี ThreadSanitizer) เปิดใช้งานผ่าน `-race` กับ `go run`, `go build`, `go test`
- แนวคิดเบื้องหลังคือการติดตาม happens-before relationship ระหว่างการเข้าถึงหน่วยความจำผ่านกลไกคล้าย vector clock
- ผลลัพธ์ของ race detector บอกตำแหน่ง read/write ที่ชนกัน, หมายเลข goroutine, และจุดที่ goroutine แต่ละตัวถูกสร้าง — อ่านแล้วสามารถระบุจุดบั๊กได้ตรงจุดทันที
- แก้ไขด้วย `sync.Mutex` (หรือ `sync/atomic` สำหรับกรณีง่ายๆ) เพื่อสร้าง happens-before relationship ที่ชัดเจน แล้วยืนยันด้วย `-race` ซ้ำจนไม่มี warning
- `map` ธรรมดาของ Go มีกลไกตรวจจับการเขียนพร้อมกันในตัว (fatal error: concurrent map writes ที่ `recover()` ดักไม่ได้) แต่ตัวแปรชนิดอื่นไม่มีกลไกป้องกันแบบนี้ ต้องพึ่ง `-race` เสมอ
- `go test -race` คือรูปแบบที่ใช้จริงในทางปฏิบัติ — เขียน test ที่จำลองการเข้าถึงพร้อมกัน แล้วให้ `-race` เป็นผู้ตัดสินว่า test "ผ่าน" จริงหรือไม่ ไม่ใช่แค่ดู assertion
- Race detector ตรวจจับได้เฉพาะ race ที่เกิดขึ้นจริงในการรันนั้น ไม่ใช่การพิสูจน์ว่าปลอดจาก race 100%
- ควรรัน `go test -race ./...` เป็นส่วนบังคับของทุก CI pipeline เสมอ (รายละเอียดเต็มใน Part 098)

## แบบฝึกหัดท้ายบท

1. รันตัวอย่างในหัวข้อ 3 (`race_bad.go`) แบบไม่ใช้ `-race` สัก 10 ครั้งติดกัน บันทึกค่าที่ได้แต่ละครั้ง แล้วอธิบายว่าทำไมบางครั้งได้ 10000 บางครั้งได้ค่าน้อยกว่า
2. รันตัวอย่างเดียวกันด้วย `go run -race` แล้วอ่านผลลัพธ์ที่ได้ ระบุว่าบรรทัดไหนคือ "Read", บรรทัดไหนคือ "Previous write" และ goroutine หมายเลขใดถูกสร้างจากบรรทัดไหน
3. แก้ไข `race_bad.go` ด้วย `sync/atomic` (`atomic.Int64`) แทนที่จะใช้ `sync.Mutex` แล้วยืนยันด้วย `-race` ว่าไม่มี warning เหลืออยู่ เปรียบเทียบว่าโค้ดเวอร์ชันไหนอ่านง่ายกว่ากัน
4. รันตัวอย่าง `map_race.go` ในหัวข้อ 8 ซ้ำ 10 ครั้ง นับว่า "รอด" กี่ครั้งและ "fatal error" กี่ครั้ง แล้วทดลองใช้ `recover()` ครอบ goroutine ที่เขียน map ดู ยืนยันว่า `recover()` ดักจับ fatal error นี้ไม่ได้จริงตามที่บทเรียนอธิบายไว้
5. เขียน unit test ของตัวเอง (ทบทวนจาก Part 033-034) สำหรับ `Cache`/`SafeCache` ในหัวข้อ 9 เพิ่มเติมอีก 1 เคส ที่ทดสอบการอ่านและเขียนพร้อมกันบน key ที่ต่างกันหลายตัว แล้วรันด้วย `go test -race -count=5 ./...` อธิบายว่า flag `-count=5` มีประโยชน์อย่างไรเมื่อใช้ร่วมกับ `-race`
6. ค้นคว้าเพิ่มเติม: ThreadSanitizer (TSan) ที่ Go race detector ใช้เป็นฐาน ถูกพัฒนาขึ้นมาสำหรับภาษาอะไรเป็นภาษาแรก และใช้แนวคิดอะไรตรวจจับ race ในภาษานั้น (เตรียมคำตอบสั้นๆ ไว้อภิปรายกับเพื่อนร่วมชั้น)

---

**ต่อไป**: [Part 045 — Concurrency Patterns ขั้นสูง: Pipeline, Pub-Sub](./045-concurrency-patterns-advanced.md)
