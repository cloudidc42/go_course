# Part 087: Debugging ด้วย Delve

> ภาคที่ 7: Testing, Tooling, Performance — ตอนที่ 9 จาก 9 (Part 79–87)

## สารบัญของบทนี้

1. ทำไมต้องมี Debugger ทั้งที่มี `fmt.Println`/`log`
2. ติดตั้ง `dlv` จริงบนเครื่อง (พร้อมปัญหาจริงที่เจอระหว่างติดตั้ง)
3. Debug โปรแกรมด้วย `dlv debug`: breakpoint, next, step, stepout, print, args, locals
4. Debug Goroutines โดยเฉพาะ: `goroutines` และการสลับ context
5. Debug โปรเซสที่กำลังรันอยู่ด้วย `dlv attach`
6. Debug Test ด้วย `dlv test`
7. ผูก Delve เข้ากับ VS Code: `launch.json`
8. เมื่อไรควรใช้ Debugger เทียบกับ Print Debugging หรือเขียน Test
9. สรุปสิ่งที่ได้เรียนในบทนี้
10. แบบฝึกหัดท้ายบท
11. สรุปภาพรวมภาคที่ 7 และก้าวต่อไปสู่ภาคที่ 8

---

## 1. ทำไมต้องมี Debugger ทั้งที่มี `fmt.Println`/`log`

ตลอดหลักสูตรนี้ เราใช้ `fmt.Println` และ `log` (Part 054) เป็นเครื่องมือหลักในการดูว่าโปรแกรมทำงานถูกต้องหรือไม่ — วิธีนี้เรียกว่า **print debugging** และเป็นวิธีที่เร็วที่สุดสำหรับปัญหาส่วนใหญ่ในชีวิตจริง แต่มันมีข้อจำกัดที่ชัดเจนขึ้นเรื่อยๆ เมื่อบั๊กซับซ้อนขึ้น:

- ต้องเดาล่วงหน้าว่าควร print ตัวแปรไหน ณ จุดไหน — ถ้าเดาผิดต้องแก้โค้ด เพิ่ม print ใหม่ แล้วรันใหม่ วนซ้ำไปเรื่อยๆ
- การ print ตัวแปรจำนวนมากทำให้โค้ดรกและต้องจำไว้ลบออกทีหลัง (หรือแย่กว่านั้นคือลืมลบแล้วหลุดไปกับ production code)
- ไม่สามารถ "หยุดโปรแกรม ณ จุดหนึ่งแล้วสำรวจ state ทั้งหมดอย่างละเอียด" ได้ — เห็นแค่สิ่งที่ตั้งใจ print ไว้ล่วงหน้าเท่านั้น
- สำหรับบั๊กที่เกี่ยวข้องกับ **goroutine หลายตัวทำงานพร้อมกัน** (ภาคที่ 3 ทั้งภาค) การ print บอก timing คร่าวๆ ได้ แต่ไม่สามารถ "หยุดทุกอย่างแล้วดูว่า goroutine แต่ละตัวอยู่ตรงไหน กำลังทำอะไรอยู่" ได้เลย

**Debugger** คือเครื่องมือที่แก้ข้อจำกัดเหล่านี้โดยตรง: มันให้เราหยุดโปรแกรม ณ จุดใดก็ได้ที่ต้องการ (โดยไม่ต้องแก้โค้ดหรือ compile ใหม่) แล้วสำรวจค่าตัวแปรทุกตัวที่มองเห็นได้ ณ ขณะนั้น เดินหน้าทีละบรรทัด และตรวจสอบ call stack ทั้งหมด — สำหรับภาษา Go เครื่องมือมาตรฐานของวงการคือ **Delve (`dlv`)** ซึ่งเป็น debugger ที่พัฒนาขึ้นมาเฉพาะสำหรับ Go โดยเข้าใจโครงสร้างภายในของ Go runtime อย่างลึกซึ้ง (goroutine, channel, interface) ต่างจาก debugger ทั่วไปอย่าง GDB ที่ไม่เข้าใจ concept เหล่านี้

ทบทวนจาก **Part 001**: ตอนติดตั้ง VS Code พร้อม Go extension เราเห็น `dlv` เป็นหนึ่งใน tools ที่ extension เสนอให้ติดตั้งอัตโนมัติไปแล้ว บทนี้จะพาไปใช้งานมันอย่างเต็มรูปแบบทั้งจาก command line และจาก VS Code

---

## 2. ติดตั้ง `dlv` จริงบนเครื่อง (พร้อมปัญหาจริงที่เจอระหว่างติดตั้ง)

ติดตั้ง Delve ด้วย `go install` (ทบทวน `GOPATH`/`GOBIN` จาก **Part 001**):

```bash
go install github.com/go-delve/delve/cmd/dlv@latest
```

ผลลัพธ์จริงตอนทดสอบเขียนบทเรียนนี้:

```
go: github.com/go-delve/delve@v1.27.2 requires go >= 1.25.0; switching to go1.26.8
```

เหมือนที่เจอกับ `benchstat` ใน **Part 084** และ `golangci-lint` ใน **Part 086** — `go install` สลับไปใช้ Go toolchain เวอร์ชันใหม่กว่าชั่วคราวเพื่อ build ตัว `dlv` เอง (`GOTOOLCHAIN=auto`) ตรวจสอบว่าติดตั้งสำเร็จ:

```bash
dlv version
```

```
Delve Debugger
Version: 1.27.2
Build: $Id: 360e7b2181d3da54115aaee4f569954bca95cfe3 $
```

### ปัญหาจริงที่เจอ: Delve เวอร์ชันใหม่ปฏิเสธ Go เวอร์ชันเก่ากว่า

หลักสูตรนี้ใช้ **Go 1.24.7** เป็นเวอร์ชันหลักในการ build โปรเจกต์ (ตามที่ตั้งไว้ตั้งแต่ Part 001) แต่พอลองใช้ `dlv` เวอร์ชัน 1.27.2 ที่เพิ่งติดตั้งมา debug โปรแกรมที่ build ด้วย Go 1.24.7 กลับเจอ error ทันที:

```bash
dlv debug .
```

```
Go version go1.24.7 is too old for this version of Delve (minimum supported version 1.25, suppress this error with --check-go-version=false)
```

**นี่คือปัญหาจริงที่ควรรู้ไว้**: Delve แต่ละเวอร์ชันมีข้อกำหนดเรื่อง**เวอร์ชันขั้นต่ำของ Go** ที่มัน debug ได้ (เพราะต้องเข้าใจ DWARF debug information ที่ compiler แต่ละเวอร์ชันสร้างขึ้น ซึ่งเปลี่ยนแปลงไปตามเวอร์ชัน) ถ้า `go install .../dlv@latest` ดึงเวอร์ชันใหม่ล่าสุดมา แต่เครื่องยังใช้ Go เวอร์ชันเก่ากว่าที่มันรองรับ จะเจอปัญหานี้ทันที มีทางแก้สองทาง:

**ทางแก้ที่ 1**: ใช้ flag `--check-go-version=false` เพื่อข้ามการเช็ค (ใช้งานได้จริงแต่ Delve เตือนว่าเป็น "undefined behavior" อาจไม่เสถียร 100%)

```bash
dlv debug --check-go-version=false .
```

**ทางแก้ที่ 2 (แนะนำ)**: ติดตั้ง `dlv` เวอร์ชันที่รองรับ Go เวอร์ชันที่ใช้งานจริงโดยเจาะจงเวอร์ชัน:

```bash
go install github.com/go-delve/delve/cmd/dlv@v1.24.2
dlv version
```

```
Delve Debugger
Version: 1.24.2
Build: $Id: 7e679e5860c17d90514f9c440e055355de831ce2 $
```

เวอร์ชันนี้ทำงานร่วมกับ Go 1.24.7 ได้อย่างสมบูรณ์โดยไม่มี warning ใดๆ เลย — บทเรียนที่เหลือทั้งหมดในบทนี้ใช้ `dlv` เวอร์ชัน 1.24.2 นี้

> **ข้อคิดสำหรับการทำงานจริง**: เวลาตั้งค่าเครื่องมือให้ทีม (โดยเฉพาะใน CI หรือ Docker image) ควรพิจารณา pin เวอร์ชันของ `dlv` ให้ตรงกับเวอร์ชัน Go ที่โปรเจกต์ใช้เสมอ แทนที่จะใช้ `@latest` ตรงไปตรงมา เพื่อเลี่ยงปัญหานี้ในอนาคต — เป็นหลักการเดียวกับที่ Part 018 สอนเรื่องการ pin เวอร์ชันของ dependency ใน `go.mod`

---

## 3. Debug โปรแกรมด้วย `dlv debug`: breakpoint, next, step, stepout, print, args, locals

สร้างโปรแกรมตัวอย่างที่มีบั๊กเบาๆ ไว้สาธิต — ฟังก์ชันคำนวณค่าเฉลี่ยที่ลืมป้องกันกรณี slice ว่างเปล่า:

```go
// main.go
package main

import "fmt"

// sumSlice รวมค่าทั้งหมดใน slice
func sumSlice(nums []int) int {
	total := 0
	for _, n := range nums {
		total += n
	}
	return total
}

// average คำนวณค่าเฉลี่ยของตัวเลขใน slice
// (บั๊กที่ตั้งใจใส่ไว้: ลืมป้องกันกรณี nums ว่าง ทำให้ปัดตัวหารเป็นศูนย์ในบางกรณี)
func average(nums []int) int {
	total := sumSlice(nums)
	avg := total / len(nums)
	return avg
}

func main() {
	scores := []int{80, 90, 100, 70}
	result := average(scores)
	fmt.Println("average score:", result)
}
```

เริ่ม debug session ด้วย `dlv debug` (คอมไพล์และเริ่ม debug ทันทีในคำสั่งเดียว คล้ายกับ `go run` แต่หยุดรอที่ debugger แทนที่จะรันจบเลย):

```bash
dlv debug .
```

จะเข้าสู่ prompt แบบ interactive `(dlv)` จากตรงนี้ไปคือคำสั่งที่พิมพ์ในเซสชันจริง (บรรทัดที่ขึ้นต้นด้วย `(dlv)` คือ prompt ของ Delve ตามด้วยคำสั่งที่พิมพ์เข้าไป):

```
(dlv) break average
Breakpoint 1 set at 0x4b1b8a for main.average() ./main.go:16
(dlv) continue
> [Breakpoint 1] main.average() ./main.go:16 (hits goroutine(1):1 total:1) (PC: 0x4b1b8a)
=>  16:	func average(nums []int) int {
    17:		total := sumSlice(nums)
    18:		avg := total / len(nums)
    19:		return avg
    20:	}
(dlv) args
nums = []int len: 4, cap: 4, [...]
(dlv) next
> main.average() ./main.go:17 (PC: 0x4b1ba6)
=>  17:		total := sumSlice(nums)
    18:		avg := total / len(nums)
(dlv) step
> main.sumSlice() ./main.go:6 (PC: 0x4b1ae4)
=>   6:	func sumSlice(nums []int) int {
     7:		total := 0
     8:		for _, n := range nums {
(dlv) locals
(no locals)
(dlv) next
> main.sumSlice() ./main.go:7 (PC: 0x4b1aff)
=>   7:		total := 0
     8:		for _, n := range nums {
(dlv) next
> main.sumSlice() ./main.go:8 (PC: 0x4b1b08)
     7:		total := 0
=>   8:		for _, n := range nums {
     9:			total += n
(dlv) stepout
> main.average() ./main.go:17 (PC: 0x4b1bab)
Values returned:
	~r0: 340

=>  17:		total := sumSlice(nums)
    18:		avg := total / len(nums)
(dlv) next
> main.average() ./main.go:18 (PC: 0x4b1bb0)
    17:		total := sumSlice(nums)
=>  18:		avg := total / len(nums)
    19:		return avg
(dlv) print total
340
(dlv) print avg
Command failed: could not find symbol value for avg
(dlv) continue
average score: 85
Process 20568 has exited with status 0
(dlv) quit
```

*(ตัดข้อความ source listing ที่ซ้ำซ้อนบางส่วนออกเพื่อความกระชับ — เนื้อหาที่เหลือทั้งหมดคือ output จริงจาก Delve)*

### อ่านคำสั่งทีละตัว

- **`break average`** (ย่อ: `b`) — ตั้ง breakpoint ที่จุดเริ่มต้นของฟังก์ชัน `average` โปรแกรมจะหยุดทันทีที่เรียกฟังก์ชันนี้
- **`continue`** (ย่อ: `c`) — สั่งให้โปรแกรมรันต่อไปจนกว่าจะเจอ breakpoint ถัดไปหรือจบโปรแกรม
- **`args`** — พิมพ์ argument ทั้งหมดของฟังก์ชันปัจจุบัน (เห็น `nums` มีค่า 4 element ตรงกับที่ `main` ส่งเข้ามา)
- **`next`** (ย่อ: `n`) — **step over**: เดินไปบรรทัดถัดไปในฟังก์ชันปัจจุบัน โดย**ไม่ลงไปข้างในฟังก์ชันที่ถูกเรียก** ถ้าบรรทัดนั้นมีการเรียกฟังก์ชันอื่น มันจะรันฟังก์ชันนั้นให้จบเลยแล้วหยุดที่บรรทัดถัดไป
- **`step`** (ย่อ: `s`) — **step into**: ถ้าบรรทัดปัจจุบันมีการเรียกฟังก์ชัน จะ**กระโดดเข้าไปข้างในฟังก์ชันนั้นทันที** สังเกตจากตัวอย่างข้างบน: `next` ที่บรรทัด 16→17 ไม่ลงไปใน `sumSlice` แต่ `step` ที่บรรทัด 17 กลับกระโดดเข้าไปในฟังก์ชัน `sumSlice` ทันที (เห็น prompt เปลี่ยนเป็น `main.sumSlice()`)
- **`locals`** — พิมพ์ตัวแปร local ทั้งหมดที่มองเห็นได้ ณ จุดปัจจุบัน (ตอนเพิ่งเข้าฟังก์ชัน `sumSlice` ยังไม่มีตัวแปร local เลยเพราะยังไม่ถึงบรรทัดที่ประกาศ `total`)
- **`stepout`** (ย่อ: `so`) — รันฟังก์ชันปัจจุบันจนจบแล้วกลับไปยังจุดที่เรียกมัน สังเกตว่า Delve แสดง **"Values returned: ~r0: 340"** ให้เห็นค่าที่ `sumSlice` คืนกลับมาทันที — มีประโยชน์มากเวลาไม่อยากเดินทีละบรรทัดข้างในฟังก์ชันย่อยที่มั่นใจอยู่แล้วว่าไม่มีบั๊ก
- **`print`** (ย่อ: `p`) — ประเมินค่านิพจน์หรือตัวแปร ณ จุดปัจจุบัน สังเกตว่า `print avg` ล้มเหลวด้วยข้อความ **"could not find symbol value for avg"** เพราะบรรทัด `avg := total / len(nums)` ยัง**ไม่ถูกรันจริง** ณ ตอนนั้น (Delve หยุด ณ จุด**ก่อน**บรรทัดปัจจุบันจะทำงาน ไม่ใช่หลัง) ต้อง `next` อีกครั้งก่อนตัวแปรนั้นถึงจะมีค่าให้ดู — เป็นรายละเอียดที่สำคัญมากตอนเริ่มใช้ Delve ใหม่ๆ

### สิ่งที่ยืนยันจากการ debug ครั้งนี้

จากการเดินโปรแกรมทีละขั้น เราเห็นว่า `sumSlice` คืนค่า `340` ถูกต้อง (`80+90+100+70=340`) และ `average` คำนวณ `340/4=85` ถูกต้องเป๊ะ ยืนยันว่าโค้ดทำงานถูกต้องสำหรับ input ชุดนี้ — แต่บั๊กที่แอบซ่อนอยู่ (`len(nums)` เป็น 0 เมื่อ `nums` ว่างเปล่า ทำให้เกิด **division by zero panic**) ยังไม่ถูกกระตุ้นเพราะ input ที่ใช้ทดสอบไม่ใช่ slice ว่าง — นี่คือเหตุผลที่การ debug ต้องทำควบคู่กับการคิดถึง **edge case** เสมอ (จะให้ลองสร้าง breakpoint แบบมีเงื่อนไขเพื่อจับกรณีนี้ไว้เป็นแบบฝึกหัดท้ายบท)

---

## 4. Debug Goroutines โดยเฉพาะ: `goroutines` และการสลับ context

นี่คือจุดที่ Delve เหนือกว่า print debugging อย่างชัดเจนที่สุด — การ debug โค้ด concurrent จากภาคที่ 3 ทั้งภาค สร้างโปรแกรมตัวอย่างที่มี worker หลายตัวทำงานพร้อมกัน:

```go
// main.go
package main

import (
	"fmt"
	"sync"
)

// worker คำนวณผลรวมอย่างง่ายแล้วส่งผลลัพธ์เข้า channel results
func worker(id int, wg *sync.WaitGroup, results chan<- int) {
	defer wg.Done()
	sum := 0
	for i := 0; i < 5; i++ {
		sum += i * id
	}
	results <- sum
}

func main() {
	var wg sync.WaitGroup
	results := make(chan int, 3)

	for i := 1; i <= 3; i++ {
		wg.Add(1)
		go worker(i, &wg, results)
	}

	wg.Wait()
	close(results)

	total := 0
	for r := range results {
		total += r
	}
	fmt.Println("total:", total)
}
```

ตั้ง breakpoint ไว้ข้างใน `worker` แล้ว `continue` — เพราะมี 3 goroutine เรียก `worker` เกือบพร้อมกัน มีโอกาสสูงที่ **breakpoint จะถูกชนโดยหลาย goroutine ในจังหวะใกล้เคียงกัน**:

```
(dlv) break worker
Breakpoint 1 set at 0x4b1eee for main.worker() ./main.go:9
(dlv) continue
> [Breakpoint 1] main.worker() ./main.go:9 (hits goroutine(21):1 total:3) (PC: 0x4b1eee)
> [Breakpoint 1] main.worker() ./main.go:9 (hits goroutine(20):1 total:3) (PC: 0x4b1eee)
> [Breakpoint 1] main.worker() ./main.go:9 (hits goroutine(22):1 total:3) (PC: 0x4b1eee)
=>   9:	func worker(id int, wg *sync.WaitGroup, results chan<- int) {
    10:		defer wg.Done()
    11:		sum := 0
```

สังเกตข้อความ **`total:3`** — ยืนยันว่าทั้ง 3 goroutine ชน breakpoint แล้ว และเมื่อ Delve หยุดโปรแกรม **มันหยุดทุก goroutine พร้อมกันทั้งหมด** (ไม่ใช่แค่ตัวที่ชน breakpoint) ทำให้ปลอดภัยที่จะสำรวจ state ของทุก goroutine ได้พร้อมกันโดยไม่ต้องกังวลว่ามันจะขยับต่อระหว่างที่เรากำลังดู — นี่คือข้อได้เปรียบสำคัญเหนือ print debugging ที่ไม่สามารถ "แช่แข็ง" ทุก goroutine พร้อมกันแบบนี้ได้เลย

### `goroutines`: ดูรายการ goroutine ทั้งหมด

```
(dlv) goroutines
  Goroutine 1 - User: /usr/local/go/src/runtime/sema.go:110 sync.runtime_SemacquireWaitGroup (0x471c85) [GC worker (idle)]
  Goroutine 2 - User: /usr/local/go/src/runtime/proc.go:436 runtime.gopark (0x470b31) [force gc (idle)]
  Goroutine 3 - User: /usr/local/go/src/runtime/proc.go:436 runtime.gopark (0x470b31) [GC sweep wait]
  Goroutine 4 - User: /usr/local/go/src/runtime/proc.go:436 runtime.gopark (0x470b31) [GC scavenge wait]
  Goroutine 17 - User: /usr/local/go/src/runtime/mfinal.go:179 runtime.runfinq (0x41cd80)
  Goroutine 20 - User: ./main.go:9 main.worker (0x4b1eee) (thread 15473)
  Goroutine 21 - User: ./main.go:9 main.worker (0x4b1eee) (thread 15461)
* Goroutine 22 - User: ./main.go:9 main.worker (0x4b1eee) (thread 15479)
[8 goroutines]
```

สังเกตสิ่งสำคัญ:

- **เครื่องหมาย `*`** หน้า `Goroutine 22` บอกว่านี่คือ **goroutine ที่กำลัง active อยู่ในบริบทปัจจุบัน** ของ debugger (คือตัวที่คำสั่งอย่าง `print`/`locals`/`bt` จะอ้างอิงถึง)
- **Goroutine 1-4, 17** คือ goroutine ภายในของ Go runtime เอง (GC worker, finalizer) — ทบทวนจาก **Part 055**: `runtime.NumGoroutine()` นับรวม goroutine เหล่านี้ด้วยเสมอ นี่คือสิ่งที่อยู่เบื้องหลังตัวเลขนั้นจริงๆ
- **Goroutine 20, 21, 22** คือ 3 worker ของเราเอง ทั้งหมดหยุดอยู่ที่ `./main.go:9` (จุด breakpoint) ตรงกัน

### สลับไปดู goroutine อื่นด้วย `goroutine <id>`

```
(dlv) goroutine 20
Switched from 22 to 20 (thread 15473)
(dlv) print id
1
(dlv) goroutine 21
Switched from 20 to 21 (thread 15461)
(dlv) print id
2
(dlv) goroutine 22
Switched from 21 to 22 (thread 15479)
(dlv) bt
0  0x00000000004b1eee in main.worker
   at ./main.go:9
1  0x00000000004b2365 in main.main.gowrap1
   at ./main.go:24
2  0x0000000000476941 in runtime.goexit
   at /usr/local/go/src/runtime/asm_amd64.s:1700
(dlv) continue
> [Breakpoint 1] main.worker() ./main.go:9 (hits goroutine(19):1 total:4) (PC: 0x4b1eee)
(dlv) continue
total: 60
Process 15461 has exited with status 0
```

คำสั่ง **`goroutine <id>`** (ย่อ: `gr`) สลับบริบทปัจจุบันไปยัง goroutine ที่ระบุ — สังเกตว่า `print id` ให้ค่าต่างกันในแต่ละ goroutine (`1`, `2`, ...) ตรงกับ argument `id` ที่ `main` ส่งเข้าไปตอนสร้างแต่ละ worker (`go worker(i, ...)` สำหรับ `i = 1, 2, 3`) — นี่คือการยืนยันโดยตรงว่า **แต่ละ goroutine มี stack และตัวแปร local เป็นของตัวเองอย่างสมบูรณ์** ตามที่เรียนไปใน **Part 036** และ **`bt`** (ย่อของ `stack`) แสดง call stack ของ goroutine ที่ active อยู่ ณ ขณะนั้น เห็นชัดว่า `worker` ถูกเรียกผ่าน `main.main.gowrap1` (ฟังก์ชัน wrapper ภายในที่ compiler สร้างขึ้นสำหรับ `go` statement) ซึ่งถูกเรียกจาก `runtime.goexit`

> **หมายเหตุสำคัญเกี่ยวกับหมายเลข Goroutine ID**: หมายเลข ID ที่เห็นจริงบนเครื่องของคุณ (เช่น 18, 19, 20 หรือ 5, 6, 7) **จะไม่ตรงกับตัวอย่างในบทเรียนนี้เสมอไป** เพราะขึ้นอยู่กับจังหวะที่ goroutine ภายในของ runtime (GC worker, sweep helper) เริ่มทำงาน ซึ่งไม่ deterministic ระหว่างการรันแต่ละครั้ง — **นี่คือธรรมชาติของการทำงานแบบ concurrent เอง** (ทบทวนจาก **Part 044**: ลำดับการทำงานของ goroutine ไม่รับประกันเสมอ) เวลาใช้งานจริง ให้ดูหมายเลขจาก output ของ `goroutines` ในเซสชันของตัวเองเสมอ ไม่ใช่ hardcode ตามตัวอย่าง

### ทำไมนี่คือเครื่องมือที่มีค่าที่สุดสำหรับบั๊ก concurrency

บั๊กประเภท **deadlock** (Part 037-039) หรือ **race condition** (Part 044) มักซับซ้อนเกินกว่าจะเข้าใจได้จากการ print เพียงอย่างเดียว เพราะปัญหาคือ**ลำดับเวลาที่ goroutine หลายตัวโต้ตอบกัน** การหยุดโปรแกรมทั้งหมดแล้วดูว่า **แต่ละ goroutine ค้างอยู่ตรงไหน กำลังรอ channel ตัวไหน หรือถือ lock ตัวไหนอยู่** (ผ่าน `goroutines` + `bt` ของแต่ละตัว) คือวิธีวินิจฉัย deadlock ที่ตรงจุดที่สุด — ยกตัวอย่างเช่น ถ้าโปรแกรม hang ค้างไม่จบ การรัน `dlv attach` (หัวข้อถัดไป) แล้วดู `goroutines` จะบอกได้ทันทีว่ามี goroutine ไหนติดอยู่ที่ `chan send`/`chan receive` ตัวไหนบ้าง ซึ่งมักชี้ตรงไปยังสาเหตุของ deadlock ได้เลยทันที

---

## 5. Debug โปรเซสที่กำลังรันอยู่ด้วย `dlv attach`

บางสถานการณ์ปัญหาเกิดขึ้นกับโปรแกรมที่**กำลังรันอยู่แล้ว** (เช่น service ที่ deploy ไว้และเริ่มมีพฤติกรรมผิดปกติ) ไม่สะดวกที่จะหยุดแล้วเริ่มใหม่ด้วย `dlv debug` — `dlv attach` แก้ปัญหานี้โดยเชื่อมต่อเข้ากับ **process ID (PID)** ที่กำลังรันอยู่โดยตรง

สร้างโปรแกรมจำลอง service ที่ทำงานต่อเนื่อง:

```go
// main.go
package main

import (
	"fmt"
	"time"
)

// tick คำนวณ state ใหม่ทุกครั้งที่ถูกเรียก (จำลอง service ที่ทำงานต่อเนื่องตลอดเวลา)
func tick(n int) int {
	return n*2 + 1
}

func main() {
	state := 0
	for {
		state = tick(state)
		fmt.Println("state:", state)
		time.Sleep(1 * time.Second)
	}
}
```

Build และรันเป็น background process:

```bash
go build -o app .
./app &
echo $!    # เก็บ PID ไว้ใช้ attach
```

### ปัญหาจริงที่เจอ: "debugging optimized function"

ลอง attach เข้าไปทันทีโดยยังไม่ได้ปิด compiler optimization:

```bash
dlv attach 15787
```

```
(dlv) break tick
Breakpoint 1 set at 0x491df4 for main.main() ./main.go:10
(dlv) continue
> [Breakpoint 1] main.main() ./main.go:10 (hits goroutine(1):1 total:1) (PC: 0x491df4)
Warning: debugging optimized function
=>  10:		return n*2 + 1
(dlv) print n
65535
```

สังเกตสองปัญหาที่เกิดพร้อมกัน: **(1)** breakpoint ที่ตั้งไว้ที่ `main.tick()` กลับไปหยุดที่ `main.main()` แทน และ **(2)** `print n` ให้ค่า **`65535`** ซึ่งเป็นค่าขยะ ไม่ใช่ค่าจริงที่ `tick` ได้รับเลย! สาเหตุตรงกับที่เรียนใน **Part 084 และ Part 055**: ฟังก์ชัน `tick` มีขนาดเล็กมาก (`return n*2+1` บรรทัดเดียว) ทำให้ compiler ตัดสินใจ **inline** มันเข้าไปใน `main` โดยตรง (`can inline tick` ถ้าตรวจด้วย `-gcflags="-m"`) เมื่อฟังก์ชันถูก inline ไป **ตัวแปร parameter ของมันก็ไม่มี "ตำแหน่ง" ที่ตรงกับ stack frame ที่ debugger คาดหวังอีกต่อไป** ทำให้ debugger อ่านค่าผิดตำแหน่งไปอ่าน memory หรือ register ที่ไม่เกี่ยวข้อง

### ทางแก้: ปิด optimization และ inlining ตอน build

```bash
go build -gcflags="all=-N -l" -o app .
```

`-N` ปิดการทำ optimization ทั้งหมด และ `-l` ปิดการทำ inlining ทั้งหมด (ทบทวนจาก **Part 084**: `//go:noinline` ปิด inline เฉพาะจุด ส่วน `-gcflags="all=-N -l"` ปิดทั้งโปรแกรมเพื่อการ debug โดยเฉพาะ) รันโปรแกรมใหม่แล้ว attach อีกครั้ง:

```
(dlv) break tick
Breakpoint 1 set at 0x4b1d24 for main.tick() ./main.go:9
(dlv) continue
> [Breakpoint 1] main.tick() ./main.go:9 (hits goroutine(1):1 total:1) (PC: 0x4b1d24)
=>   9:	func tick(n int) int {
    10:		return n*2 + 1
(dlv) print n
127
(dlv) next
> main.tick() ./main.go:10 (PC: 0x4b1d35)
     9:	func tick(n int) int {
=>  10:		return n*2 + 1
(dlv) print n
127
(dlv) continue
> [Breakpoint 1] main.tick() ./main.go:9 (hits goroutine(1):2 total:2) (PC: 0x4b1d24)
```

คราวนี้ breakpoint หยุดตรงจุดที่ตั้งไว้จริง (`main.tick()`) และ `print n` ให้ค่าที่ถูกต้อง (`127` ตรงกับลำดับ `1, 3, 7, 15, 31, 63, 127, ...` ที่ `tick` สร้างขึ้นตามฟังก์ชัน `n*2+1`)

> **กฎสำคัญที่ควรจำ**: เวลา build binary เพื่อ debug ด้วย Delve (ไม่ว่าจะ `dlv debug`, `dlv attach`, หรือ `dlv exec`) **ควร build ด้วย `-gcflags="all=-N -l"` เสมอ** เพื่อปิด optimization และ inlining ที่ทำให้ตัวแปรอ่านค่าผิดเพี้ยนแบบนี้ — `dlv debug` ทำสิ่งนี้ให้อัตโนมัติอยู่แล้วเวลา build เอง แต่ `dlv attach` เข้าไปยัง binary ที่ build ไว้ล่วงหน้า **จึงต้องมั่นใจเองว่า binary นั้นถูก build โดยปิด optimization ไว้แล้ว** มิฉะนั้นจะเจอปัญหานี้ทุกครั้งกับฟังก์ชันขนาดเล็กที่มีโอกาสถูก inline

### Detach โดยไม่ฆ่า process

เมื่อ debug เสร็จแล้ว ต้องการหยุด debugger แต่**ให้ service ทำงานต่อไปตามปกติ** ไม่ใช่ฆ่าทิ้ง:

```
(dlv) exit
Would you like to kill the process? [Y/n] n
```

ตอบ **`n`** เพื่อ **detach** ออกจากโปรเซสโดยไม่ฆ่ามัน — โปรแกรมจะทำงานต่อไปตามปกติราวกับไม่มีอะไรเกิดขึ้น (ตรวจสอบได้จาก log ของโปรแกรมที่ยังคง print `state:` ต่อไปเรื่อยๆ หลัง detach) นี่คือความแตกต่างสำคัญระหว่าง `dlv attach` กับ `dlv debug`: `dlv debug` เริ่มโปรเซสขึ้นมาเอง ดังนั้น `quit` จะฆ่าโปรเซสนั้นไปด้วยเสมอ ในขณะที่ `dlv attach` เข้าไปยังโปรเซสที่มีอยู่แล้วก่อน จึงเลือกได้ว่าจะฆ่าทิ้งหรือปล่อยให้ทำงานต่อ

---

## 6. Debug Test ด้วย `dlv test`

Delve debug unit test ได้โดยตรงเช่นกัน มีประโยชน์มากเมื่อ test ล้มเหลวและอยากรู้ว่าค่ากลางทางเป็นอย่างไรบ้างโดยไม่ต้องเขียน `t.Logf` เพิ่มทีละจุด สมมติมีฟังก์ชันคำนวณส่วนลดที่มีบั๊ก:

```go
// calc.go
package calc

// Discount คำนวณราคาหลังหักส่วนลด (percent เป็นตัวเลข 0-100)
// บั๊กที่ตั้งใจใส่ไว้: หารด้วย 100 ผิดเป็นคูณ ทำให้ผลลัพธ์คลาดเคลื่อน
func Discount(price float64, percent float64) float64 {
	discountAmount := price * percent * 100 // ควรเป็น price * percent / 100
	return price - discountAmount
}
```

```go
// calc_test.go
package calc

import "testing"

func TestDiscount(t *testing.T) {
	got := Discount(200, 10) // ลด 10% จาก 200 ควรได้ 180
	want := 180.0
	if got != want {
		t.Errorf("Discount(200, 10) = %v, want %v", got, want)
	}
}
```

รัน test ปกติก่อนเพื่อยืนยันว่ามันล้มเหลวจริง:

```bash
go test ./...
```

```
--- FAIL: TestDiscount (0.00s)
    calc_test.go:9: Discount(200, 10) = -199800, want 180
FAIL
```

`-199800` เป็นค่าที่ผิดเพี้ยนมากจนดูไม่ออกทันทีว่าปัญหาอยู่ตรงไหนถ้าไม่ไล่คำนวณด้วยมือ — มา debug เข้าไปดูค่ากลางทางจริงด้วย `dlv test`:

```bash
dlv test .
```

```
(dlv) break Discount
Breakpoint 1 set at 0x59e784 for dlvtest.Discount() ./calc.go:5
(dlv) continue
> [Breakpoint 1] dlvtest.Discount() ./calc.go:5 (hits goroutine(18):1 total:1) (PC: 0x59e784)
=>   5:	func Discount(price float64, percent float64) float64 {
     6:		discountAmount := price * percent * 100
     7:		return price - discountAmount
(dlv) args
price = 200
percent = 10
(dlv) next
> dlvtest.Discount() ./calc.go:6 (PC: 0x59e79c)
     5:	func Discount(price float64, percent float64) float64 {
=>   6:		discountAmount := price * percent * 100
     7:		return price - discountAmount
(dlv) next
> dlvtest.Discount() ./calc.go:7 (PC: 0x59e7b2)
     6:		discountAmount := price * percent * 100
=>   7:		return price - discountAmount
(dlv) print discountAmount
200000
(dlv) continue
--- FAIL: TestDiscount (0.01s)
    calc_test.go:9: Discount(200, 10) = -199800, want 180
FAIL
Process 18501 has exited with status 1
```

`args` ยืนยันว่า input ถูกต้อง (`price=200`, `percent=10`) แต่ `print discountAmount` หลัง `next` ผ่านบรรทัดคำนวณไปแล้วเผยให้เห็นค่า **`200000`** ทันที — เห็นชัดเจนว่าตัวเลขนี้ผิดปกติมาก (`200*10*100 = 200000` แทนที่จะเป็น `200*10/100 = 20`) ทำให้หาสาเหตุของบั๊กได้แม่นยำและรวดเร็วกว่าการนั่งไล่สูตรด้วยตาเปล่ามาก **นี่คือจุดแข็งที่สุดของ `dlv test`**: มันให้ debug เข้าไปในโค้ดจริงที่ test เรียกใช้ โดยไม่ต้องเขียนโปรแกรม `main` แยกต่างหากเพื่อจำลองสถานการณ์เดียวกัน — ใช้ test ที่มีอยู่แล้วเป็นจุดเริ่ม debug ได้ทันที

---

## 7. ผูก Delve เข้ากับ VS Code: `launch.json`

การพิมพ์คำสั่ง `dlv` ทีละบรรทัดใน terminal มีประโยชน์มากสำหรับการเรียนรู้และ debug เฉพาะกิจ แต่งานประจำวันจริงส่วนใหญ่สะดวกกว่ามากเมื่อ debug ผ่าน editor โดยตรง (ตั้ง breakpoint ด้วยการคลิกข้างเลขบรรทัด, กด F5 เพื่อรัน) ทบทวนจาก **Part 001**: VS Code พร้อม Go extension ติดตั้ง `dlv` ให้อัตโนมัติอยู่แล้ว เหลือแค่ตั้งค่าไฟล์ **`launch.json`** เพื่อบอกวิธี debug โปรเจกต์

สร้างไฟล์ `.vscode/launch.json`:

```jsonc
{
    "version": "0.2.0",
    "configurations": [
        {
            "name": "Launch Package",
            "type": "go",
            "request": "launch",
            "mode": "auto",
            "program": "${workspaceFolder}"
        },
        {
            "name": "Launch Current File",
            "type": "go",
            "request": "launch",
            "mode": "debug",
            "program": "${file}"
        },
        {
            "name": "Attach to Process",
            "type": "go",
            "request": "attach",
            "mode": "local",
            "processId": "${command:pickProcess}"
        },
        {
            "name": "Debug Test (Current Package)",
            "type": "go",
            "request": "launch",
            "mode": "test",
            "program": "${workspaceFolder}"
        }
    ]
}
```

### ความหมายของแต่ละ configuration

- **`Launch Package`** (`mode: "auto"`) — เทียบเท่ากับ `dlv debug .` ที่เรียนในหัวข้อ 3 VS Code จะ build และเริ่ม debug session ให้อัตโนมัติ เหมาะเป็นค่าเริ่มต้นสำหรับ debug โปรแกรม `main` ทั่วไป
- **`Launch Current File`** — debug เฉพาะไฟล์ `.go` ที่เปิดอยู่ในแท็บปัจจุบัน (`${file}`) มีประโยชน์เมื่อทดลองโค้ดสั้นๆ ในไฟล์เดียว
- **`Attach to Process`** — เทียบเท่ากับ `dlv attach <pid>` ในหัวข้อ 5 แต่สะดวกกว่ามาก เพราะ `${command:pickProcess}` จะเปิดหน้าต่างให้เลือกโปรเซสจากรายการที่กำลังรันอยู่ ไม่ต้องหา PID เองด้วยมือ
- **`Debug Test (Current Package)`** — เทียบเท่ากับ `dlv test .` ในหัวข้อ 6 debug test ทั้งหมดในไฟล์/package ปัจจุบัน

หลังตั้งค่าแล้ว การ debug ทำได้ง่ายมาก: **คลิกซ้ายที่ช่องว่างข้างเลขบรรทัด**ในโค้ดเพื่อตั้ง breakpoint (จุดแดงจะปรากฏ) แล้วกด **F5** (หรือคลิก "Run and Debug" ที่แถบด้านซ้าย) เลือก configuration ที่ต้องการ VS Code จะหยุดโปรแกรมที่ breakpoint พร้อมแสดง **panel ตัวแปร local, call stack, และ watch expression** ทั้งหมดแบบภาพกราฟิก ไม่ต้องพิมพ์คำสั่ง `args`/`locals`/`bt` เองทีละคำสั่งเหมือนใน terminal — เบื้องหลังการทำงานทั้งหมดนี้คือ `dlv` ตัวเดียวกับที่เรียนมาทั้งบท เพียงแต่ VS Code สื่อสารกับมันผ่าน **Debug Adapter Protocol (DAP)** แทนที่จะเป็น text interface ตรงๆ

---

## 8. เมื่อไรควรใช้ Debugger เทียบกับ Print Debugging หรือเขียน Test

หลังเรียนรู้ความสามารถทั้งหมดของ Delve แล้ว คำถามที่สำคัญไม่แพ้กันคือ **เมื่อไรควรใช้มันจริงๆ** เพราะ debugger ไม่ใช่เครื่องมือที่เหมาะกับทุกสถานการณ์เสมอไป

### ใช้ Print/Log Debugging เมื่อ...

- บั๊กเกิดจาก**ค่าที่ผิดพลาดชัดเจน** และรู้อยู่แล้วคร่าวๆ ว่าน่าจะอยู่ตรงไหน — เพิ่ม `fmt.Println`/`log.Printf` 1-2 จุดแล้วรันใหม่เร็วกว่าการเปิด debugger เต็มรูปแบบมาก
- ต้องดูพฤติกรรมของโปรแกรม**ตลอดช่วงเวลาที่ยาวนาน** (เช่น log การทำงานตลอด 24 ชั่วโมงของ service) — debugger ไม่เหมาะกับการสังเกตพฤติกรรมระยะยาวแบบนี้เลย
- โค้ด production ที่รันบน environment ที่**เข้าถึง debugger ไม่ได้โดยตรง** (เช่น serverless function บางแพลตฟอร์ม) — structured logging (Part 054) คือทางเลือกเดียวที่ทำได้จริง

### เขียน Test เมื่อ...

- พบบั๊กแล้วต้องการ**ป้องกันไม่ให้มันกลับมาอีก** — เขียน test case ที่ reproduce บั๊กนั้นก่อนเสมอ (fail ก่อน แล้วค่อยแก้โค้ดให้ test ผ่าน) ตามหลักการ TDD ที่เรียนใน **Part 033-035** วิธีนี้ให้ผลตอบแทนระยะยาวมากกว่าการ debug แล้วแก้แบบครั้งเดียวจบ เพราะ test case จะคอยเตือนถ้าบั๊กเดิมย้อนกลับมาในอนาคต
- บั๊กเกิดจาก**ตรรกะที่ซับซ้อนแต่ isolate ได้ง่าย** (pure function ที่รับ input คืน output ชัดเจน) — เขียน table-driven test (Part 034) ครอบคลุมหลาย edge case มักหาสาเหตุได้เร็วกว่าการเดินโปรแกรมทีละบรรทัดด้วยซ้ำ

### ใช้ Debugger (Delve) เมื่อ...

- บั๊กเกี่ยวข้องกับ **หลาย goroutine ทำงานพร้อมกัน** (deadlock, race condition ที่ reproduce ยาก) — ความสามารถหยุดทุก goroutine พร้อมกันแล้วสำรวจ state ของ `dlv` (หัวข้อ 4) ไม่มีสิ่งใดทดแทนได้ในกรณีนี้
- ต้องการ **สำรวจ state ที่ซับซ้อนและไม่รู้ล่วงหน้าว่าต้องดูตัวแปรไหน** — เช่น debug เข้าไปใน library ของคนอื่นที่ไม่คุ้นเคยกับโค้ดเลย การตั้ง breakpoint แล้วไล่ดู call stack และตัวแปรทีละชั้นเร็วกว่าการเดา print หลายรอบมาก
- บั๊กเกิดขึ้น**เฉพาะบน production หรือ environment ที่ reproduce ในเครื่อง dev ไม่ได้** — `dlv attach` (หัวข้อ 5) ให้เข้าไปสำรวจ process ที่กำลังรันจริงได้โดยไม่ต้องหยุดหรือแก้โค้ดเลย
- ต้องการเข้าใจ**พฤติกรรมภายในของ Go runtime เอง** (เช่น escape analysis ทำงานจริงอย่างไรกับโค้ดของเรา ตามที่เห็นในหัวข้อ 5 เรื่อง inlining) — debugger ให้มุมมองที่ตรงและแม่นยำกว่า

### หลักปฏิบัติที่สมดุล

ในทางปฏิบัติ นักพัฒนา Go มืออาชีพส่วนใหญ่ใช้ **print/log debugging และการเขียน test เป็นเครื่องมือหลักในชีวิตประจำวัน** (เพราะครอบคลุมบั๊กส่วนใหญ่ที่เจอ และเร็วกว่า) และหยิบ **Delve มาใช้เฉพาะเมื่อเจอปัญหาที่สองวิธีแรกไม่พอจริงๆ** โดยเฉพาะบั๊ก concurrency ที่ซับซ้อน — การรู้จักทั้งสามเครื่องมือและรู้ว่าเมื่อไรควรหยิบตัวไหนมาใช้ คือทักษะที่แยกนักพัฒนาที่ debug เก่งออกจากคนที่ยังต้องคลำทางอยู่

---

## สรุปสิ่งที่ได้เรียนในบทนี้

- **Delve (`dlv`)** คือ debugger มาตรฐานของ Go ที่เข้าใจโครงสร้างภายในของ Go runtime (goroutine, channel) ติดตั้งด้วย `go install github.com/go-delve/delve/cmd/dlv@latest` — ควรตรวจสอบว่าเวอร์ชันของ `dlv` รองรับเวอร์ชัน Go ที่ใช้งานจริง มิฉะนั้นจะเจอ error "too old" (แก้ด้วยการ pin เวอร์ชัน `dlv` ให้เหมาะสม)
- **`dlv debug`** compile และเริ่ม debug session ทันที — คำสั่งหลักที่ต้องรู้: `break`, `continue`, `next` (step over), `step` (step into), `stepout`, `print`, `args`, `locals`
- **Debug goroutine โดยเฉพาะ** ด้วย `goroutines` (แสดงทุก goroutine พร้อม `*` บอกตัวที่ active) และ `goroutine <id>` (สลับ context) — เมื่อ breakpoint ถูกชน **ทุก goroutine หยุดพร้อมกันหมด** ทำให้สำรวจ state ของ concurrent bug ได้อย่างปลอดภัย
- **`dlv attach <pid>`** debug โปรเซสที่กำลังรันอยู่แล้ว โดยไม่ต้องหยุดมันก่อน — ต้อง build binary ด้วย `-gcflags="all=-N -l"` เพื่อปิด optimization/inlining มิฉะนั้นจะเจอ "debugging optimized function" และค่าตัวแปรผิดเพี้ยน — `exit` แล้วตอบ `n` เพื่อ detach โดยไม่ฆ่าโปรเซส
- **`dlv test`** debug unit test ได้โดยตรง ใช้ test ที่มีอยู่แล้วเป็นจุดเริ่ม debug แทนการเขียนโปรแกรม `main` แยกต่างหาก
- **VS Code** debug ผ่าน `.vscode/launch.json` ด้วย 4 mode หลัก: launch package, launch current file, attach to process, debug test — ทั้งหมดใช้ `dlv` เดียวกันเบื้องหลังผ่าน Debug Adapter Protocol
- **เลือกเครื่องมือให้เหมาะกับสถานการณ์**: print/log debugging และ test เป็นเครื่องมือหลักในชีวิตประจำวัน ส่วน debugger เหมาะที่สุดกับบั๊ก concurrency ที่ซับซ้อน, การสำรวจโค้ดที่ไม่คุ้นเคย, และปัญหาที่ reproduce ในเครื่อง dev ไม่ได้

## แบบฝึกหัดท้ายบท

1. ติดตั้ง `dlv` บนเครื่องของตัวเอง ตรวจสอบว่าเวอร์ชันรองรับ Go เวอร์ชันที่ใช้งานอยู่หรือไม่ (ทดลองรัน `dlv debug` กับโปรเจกต์ใดก็ได้ ถ้าเจอ error "too old" ให้แก้ตามวิธีในหัวข้อ 2)
2. ใช้โปรแกรม `average`/`sumSlice` จากหัวข้อ 3 แล้วตั้ง **conditional breakpoint** (ค้นคว้าคำสั่ง `condition`) ที่ทำงานเฉพาะเมื่อ `len(nums) == 0` แล้วทดสอบเรียก `average(nil)` เพื่อยืนยันว่า debugger หยุดตรงจุดที่ panic เกิดขึ้นจริง
3. เขียนโปรแกรมที่มี goroutine อย่างน้อย 4 ตัวติด **deadlock** จริง (เช่น รอ channel ที่ไม่มีใครส่งข้อมูลเข้ามา) แล้วใช้ `dlv attach` เข้าไปดู `goroutines` เพื่อหาว่า goroutine ไหนติดอยู่ที่ไหนบ้าง
4. ทดลอง build โปรแกรมที่มีฟังก์ชันเล็กๆ (เหมือน `tick` ในหัวข้อ 5) โดย**ไม่ใส่** `-gcflags="all=-N -l"` แล้ว attach ดู สังเกตความผิดเพี้ยนของค่าตัวแปรด้วยตัวเอง จากนั้นลองใหม่พร้อมใส่ flag แล้วเปรียบเทียบ
5. ตั้งค่า `.vscode/launch.json` ตามหัวข้อ 7 ในโปรเจกต์ทดสอบของตัวเอง ลองตั้ง breakpoint ด้วยการคลิกใน VS Code แล้วกด F5 เปรียบเทียบประสบการณ์กับการพิมพ์คำสั่งใน terminal
6. เลือกบั๊กหรือปัญหาที่เคยเจอจริงในแบบฝึกหัดของ Part ก่อนหน้า (โดยเฉพาะจากภาคที่ 3: Concurrency) แล้วลองแก้ปัญหานั้นใหม่อีกครั้งด้วย Delve แทนการ print debugging เปรียบเทียบว่าวิธีไหนหาสาเหตุได้เร็วกว่าในกรณีนั้น

---

## สรุปภาพรวมภาคที่ 7 และก้าวต่อไปสู่ภาคที่ 8

ภาคที่ 7 (Part 79-87) พาเราเดินทางผ่านเครื่องมือและเทคนิคที่แยก **"เขียนโค้ด Go ได้"** ออกจาก **"เขียนโค้ด Go ระดับมืออาชีพที่พร้อมใช้งานจริง"**: การทดสอบขั้นสูง (unit, integration, `httptest`, coverage), การวัดผลด้วย profiling และ benchmarking, การจัดการหน่วยความจำอย่างมีข้อมูลรองรับ, การรักษาคุณภาพโค้ดด้วย linting อัตโนมัติ, และสุดท้ายคือการ debug ปัญหาที่ซับซ้อนด้วยเครื่องมือระดับมืออาชีพอย่าง Delve

ทักษะทั้งหมดนี้มีจุดร่วมเดียวกัน: **การตัดสินใจบนพื้นฐานของข้อมูลจริง ไม่ใช่การเดา** ไม่ว่าจะเป็น `pprof` บอกคอขวดจริง, `benchstat` บอกว่าความต่างมีนัยสำคัญทางสถิติหรือไม่, `testing.AllocsPerRun` พิสูจน์ว่ากลยุทธ์ optimize ได้ผลจริง, `golangci-lint` จับปัญหาที่มองข้ามได้ตามหลักฐานที่ระบุชัดเจน หรือ Delve ที่ให้เห็น state จริงของโปรแกรม ณ ขณะหนึ่งแทนการคาดเดา

จากนี้ไป **ภาคที่ 8: Microservices, gRPC, Message Queue (Part 88-94)** จะพาไปสู่การออกแบบระบบที่ใหญ่ขึ้น — แตกระบบ monolith เป็น service ย่อยที่สื่อสารกันผ่าน gRPC และ message queue (RabbitMQ, Kafka) พร้อมรูปแบบการออกแบบสำหรับระบบกระจาย (distributed system) เช่น service discovery, API gateway, circuit breaker — ทักษะเรื่อง testing, profiling, memory management, linting และ debugging ที่เพิ่งเรียนจบในภาคนี้จะกลับมาสำคัญยิ่งขึ้นไปอีก เพราะระบบที่กระจายตัวออกเป็นหลาย service ยิ่งวินิจฉัยปัญหายากขึ้นตามธรรมชาติ และยิ่งต้องพึ่งเครื่องมือที่แม่นยำแทนการเดามากกว่าเดิม

---

**ต่อไป**: [Part 088 — หลักการ Microservices Architecture](./088-microservices-architecture.md)
