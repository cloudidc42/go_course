# Part 042: Fan-in / Fan-out Pattern

> ภาคที่ 3: การทำงานพร้อมกัน (Concurrency) — ตอนที่ 7 จาก 10 (Part 36–45)

## สารบัญของบทนี้

1. ทบทวน: Worker Pool คือกรณีพิเศษของ Fan-out
2. Generator Pattern: จุดเริ่มต้นมาตรฐานของทุก pipeline
3. Fan-out: กระจายงานจาก channel เดียวไปหลาย goroutine
4. Fan-in: รวมหลาย channel กลับเป็น channel เดียว
5. ตัวอย่างสมบูรณ์: Pipeline แบบ Fan-out แล้ว Fan-in
6. Done Channel: กลไกยกเลิก pipeline กลางคัน
7. เมื่อไรควรใช้ Fan-out/Fan-in และเมื่อไรไม่ควร
8. สรุปสิ่งที่ได้เรียนในบทนี้
9. แบบฝึกหัดท้ายบท

---

## 1. ทบทวน: Worker Pool คือกรณีพิเศษของ Fan-out

ใน Part 041 เราเขียน Worker Pool ที่มี goroutine หลายตัวอ่านงานจาก `jobs` channel เดียวกัน แล้วเขียนผลลัพธ์ทั้งหมดไปที่ `results` channel เดียวกัน แท้จริงแล้วนี่คือรูปแบบทั่วไปสองอย่างรวมกัน ที่ในวงการ Go เรียกกันว่า:

- **Fan-out**: การให้หลาย goroutine (consumer) อ่านข้อมูลจาก **channel เดียวกัน** พร้อมกัน เพื่อกระจายงานออกไปประมวลผลแบบขนาน
- **Fan-in**: การรวมข้อมูลจาก **หลาย channel** เข้าเป็น **channel เดียว** เพื่อให้ผู้บริโภคปลายทางอ่านจากจุดเดียว

Worker Pool ของ Part 041 มี fan-out อยู่แล้ว (worker หลายตัวอ่านจาก `jobs`) แต่ยังไม่มี fan-in ที่ชัดเจน (worker เขียนตรงเข้า `results` channel เดียวกันเลย โดยไม่ผ่าน channel ของตัวเองก่อน) บทนี้จะแยกสองแนวคิดนี้ออกจากกันให้ชัดเจน และแสดงให้เห็นว่าเมื่อนำมา**ต่อกัน**เป็น pipeline หลายขั้นตอน จะยืดหยุ่นและนำกลับมาใช้ซ้ำได้มากกว่าการเขียน Worker Pool แบบขั้นตอนเดียว

แนวคิดทั้งหมดนี้มาจาก talk ชื่อดังของ **Rob Pike** หนึ่งในผู้สร้างภาษา Go เรื่อง **"Go Concurrency Patterns"** (Google I/O 2012) ซึ่งเป็นต้นแบบของคำศัพท์และรูปแบบที่ชุมชน Go ใช้กันมาจนถึงทุกวันนี้

---

## 2. Generator Pattern: จุดเริ่มต้นมาตรฐานของทุก pipeline

ก่อนจะพูดถึง fan-out/fan-in เราต้องมีวิธี "เริ่มต้น" ส่งข้อมูลเข้า pipeline ก่อน วิธีมาตรฐานคือ **generator pattern**: เขียนฟังก์ชันที่ **คืนค่าเป็น channel** โดยมี goroutine ทำงานอยู่เบื้องหลังคอยป้อนข้อมูลเข้า channel นั้นเรื่อยๆ

```go
// generator (generator pattern): ฟังก์ชันที่คืนค่าเป็น channel แล้วมี goroutine
// คอยป้อนข้อมูลเข้า channel นั้นในเบื้องหลัง นี่คือวิธีมาตรฐานในการ "เริ่ม" pipeline stage แรก
func generator(nums ...int) <-chan int {
	out := make(chan int)
	go func() {
		defer close(out)
		for _, n := range nums {
			out <- n
		}
	}()
	return out
}
```

จุดที่ควรสังเกต:

- Return type เป็น `<-chan int` (receive-only) — ผู้เรียกฟังก์ชันนี้มีสิทธิ์แค่ "อ่าน" เท่านั้น สอดคล้องกับหลักการที่เรียนไปใน Part 041
- `defer close(out)` รับประกันว่า channel จะถูกปิดเสมอเมื่อ goroutine ทำงานจนจบ (ไม่ว่าจะจบแบบปกติหรือ panic ก็ตาม) ทำให้ผู้ที่ `range` อ่านจาก channel นี้รู้ว่าเมื่อไรควรหยุด
- ฟังก์ชันคืนค่ากลับทันที (ไม่บล็อก) เพราะการส่งข้อมูลจริงเกิดขึ้นใน goroutine แยกต่างหาก — นี่คือเหตุผลที่เรียกว่า "generator" เหมือนกับ generator ในภาษาอื่นที่ผลิตค่าออกมาทีละตัวแบบ lazy

Generator เป็น **stage แรกสุด** ของทุก pipeline ที่เราจะสร้างต่อจากนี้ ทั้งในบทนี้และ Part 045 (ที่จะทำให้เป็น pattern แบบ generic ที่ใช้ซ้ำได้)

---

## 3. Fan-out: กระจายงานจาก channel เดียวไปหลาย goroutine

Fan-out คือการเปิดหลาย goroutine ที่ **อ่านจาก channel input เดียวกัน** เมื่องานมาถึง ใครว่างก่อนก็ได้งานนั้นไปทำ (การกระจายงานเกิดขึ้นเองโดยกลไกของ channel ไม่ต้องเขียน logic แบ่งงานเอง)

```go
// square คือ pipeline stage: รับ channel เข้า คืน channel ออก
// (stage รูปแบบนี้จะกลายเป็นหัวใจของ part 045 เรื่อง reusable pipeline)
func square(in <-chan int) <-chan int {
	out := make(chan int)
	go func() {
		defer close(out)
		for n := range in {
			out <- n * n
		}
	}()
	return out
}

// fanOut สร้าง worker หลายตัวที่อ่านจาก in channel เดียวกัน (แข่งกันดึงงาน)
// แล้วแต่ละตัวส่งผลลัพธ์เข้า channel ของตัวเอง คืนค่าเป็น slice ของ output channels
func fanOut(in <-chan int, n int) []<-chan int {
	outs := make([]<-chan int, n)
	for i := 0; i < n; i++ {
		outs[i] = square(in)
	}
	return outs
}
```

สังเกตว่า `fanOut` เรียก `square(in)` ซ้ำ `n` ครั้งโดยส่ง channel `in` **ตัวเดียวกัน** ให้ทุกครั้ง — ผลลัพธ์คือ `n` goroutine ที่ทำงานแบบเดียวกัน (square) แต่ทั้งหมดแข่งกันอ่านจาก `in` channel เดียว ทำให้งานถูกกระจายไปยัง goroutine ที่ว่างที่สุด ณ ขณะนั้นโดยอัตโนมัติ — หลักการเดียวกับ Worker Pool ใน Part 041 เป๊ะๆ เพียงแต่เขียนในรูปแบบ "stage" ที่นำไปต่อกับ stage อื่นได้ง่ายกว่า

**ทำไมแต่ละ output จึงต้องเป็น channel ของตัวเอง** แทนที่จะเขียนรวมเข้า channel เดียวกันตรงๆ? เพราะการแยก output ของแต่ละ worker ออกจากกันทำให้เราต้องมีขั้นตอนถัดไปที่ **ตั้งใจ** รวมมันกลับเข้าด้วยกัน (fan-in) ซึ่งทำให้ pipeline แต่ละ stage รับผิดชอบหน้าที่เดียวชัดเจน (single responsibility) — นี่คือข้อดีของการแยก fan-out และ fan-in ออกจากกันเป็นสอง stage แทนที่จะรวมไว้ในฟังก์ชันเดียวแบบ Worker Pool

---

## 4. Fan-in: รวมหลาย channel กลับเป็น channel เดียว

Fan-in คือขั้นตอนตรงข้าม: รับ channel หลายตัว รวมเป็น channel เดียว ให้ผู้บริโภคปลายทางอ่านจากจุดเดียวโดยไม่ต้องสนใจว่าข้อมูลมาจาก channel ไหน

```go
// fanIn รวมหลาย channel เข้าเป็น channel เดียว โดยใช้ WaitGroup
// รอให้ทุก input channel ถูกปิดก่อน แล้วค่อยปิด output channel
func fanIn(channels ...<-chan int) <-chan int {
	out := make(chan int)
	var wg sync.WaitGroup
	wg.Add(len(channels))

	for _, c := range channels {
		go func(c <-chan int) {
			defer wg.Done()
			for n := range c {
				out <- n
			}
		}(c)
	}

	// goroutine แยกต่างหากคอยปิด out เมื่อทุก input channel หมด
	go func() {
		wg.Wait()
		close(out)
	}()

	return out
}
```

โครงสร้างนี้คุ้นเคยมากใช่ไหม? มันคือหลักการเดียวกับที่เราใช้ปิด `results` channel ใน Worker Pool (Part 041 หัวข้อ 4): เปิด goroutine หนึ่งตัวต่อ input channel หนึ่งตัว คอยส่งต่อค่าเข้า `out`, ใช้ `WaitGroup` นับว่า goroutine เหล่านี้ทำงานเสร็จครบทุกตัวหรือยัง แล้วให้ goroutine แยกต่างหากปิด `out` เมื่อ `wg.Wait()` ผ่าน

**ทำไมต้องมี goroutine แยกสำหรับแต่ละ input channel?** เพราะ `out <- n` อาจบล็อก (ถ้าไม่มีใครมาอ่าน `out` ทันที) การรวมทุก input channel ไว้ใน loop เดียวแบบ sequential (`for c := range c1 { ... }; for c := range c2 { ... }`) จะทำให้ต้องอ่าน `c1` ให้หมดก่อนถึงจะไปอ่าน `c2` ได้ ซึ่งไม่ใช่ fan-in ที่แท้จริง (ไม่ concurrent) การเปิด goroutine แยกสำหรับ channel แต่ละตัวทำให้ทุก channel ถูกอ่าน**พร้อมกัน**จริงๆ

---

## 5. ตัวอย่างสมบูรณ์: Pipeline แบบ Fan-out แล้ว Fan-in

มาประกอบทุกชิ้นส่วนเข้าด้วยกันเป็น pipeline สมบูรณ์: **generator → fan-out (square) → fan-in**

```go
package main

import (
	"fmt"
	"sync"
)

func generator(nums ...int) <-chan int {
	out := make(chan int)
	go func() {
		defer close(out)
		for _, n := range nums {
			out <- n
		}
	}()
	return out
}

func square(in <-chan int) <-chan int {
	out := make(chan int)
	go func() {
		defer close(out)
		for n := range in {
			out <- n * n
		}
	}()
	return out
}

func fanOut(in <-chan int, n int) []<-chan int {
	outs := make([]<-chan int, n)
	for i := 0; i < n; i++ {
		outs[i] = square(in)
	}
	return outs
}

func fanIn(channels ...<-chan int) <-chan int {
	out := make(chan int)
	var wg sync.WaitGroup
	wg.Add(len(channels))

	for _, c := range channels {
		go func(c <-chan int) {
			defer wg.Done()
			for n := range c {
				out <- n
			}
		}(c)
	}

	go func() {
		wg.Wait()
		close(out)
	}()

	return out
}

func main() {
	// stage 1: generator สร้าง input channel จากตัวเลข 1..10
	source := generator(1, 2, 3, 4, 5, 6, 7, 8, 9, 10)

	// stage 2: fan-out ให้ worker 3 ตัวช่วยกันคำนวณ square แบบขนาน
	workers := fanOut(source, 3)

	// stage 3: fan-in รวมผลลัพธ์จาก worker ทั้ง 3 กลับเป็น channel เดียว
	merged := fanIn(workers...)

	sum := 0
	count := 0
	for n := range merged {
		sum += n
		count++
	}

	fmt.Printf("ประมวลผล %d ค่า, ผลรวมของ square ทั้งหมด = %d\n", count, sum)
}
```

รันด้วย `go run -race main.go` ผลลัพธ์ (ยืนยันด้วยการรันซ้ำ 3 ครั้งว่าได้ค่าถูกต้องเสมอ):

```
ประมวลผล 10 ค่า, ผลรวมของ square ทั้งหมด = 385
```

(1² + 2² + ... + 10² = 385 — ถูกต้องเสมอไม่ว่า worker ไหนจะประมวลผลตัวเลขไหนก่อน เพราะการบวกผลรวมไม่ขึ้นกับลำดับ)

Diagram ของ pipeline นี้:

```
                              ┌──────────┐
                        ┌────▶│ square 1 │────┐
                        │     └──────────┘    │
 generator(1..10)  ─────┼────▶│ square 2 │────┼──── fanIn ──── sum
   (source chan)        │     └──────────┘    │   (merged chan)
                        └────▶│ square 3 │────┘
                              └──────────┘
                            (fan-out)      (fan-in)
```

สังเกตว่าทุก stage (`generator`, `square`, `fanIn`) มี **รูปแบบเดียวกันหมด**: รับ channel (หรือค่าตั้งต้น) เข้า คืน channel ออก และมี goroutine ทำงานเบื้องหลัง — นี่คือสิ่งที่ทำให้ pipeline สไตล์นี้ต่อกันเป็นสายยาวได้เรื่อยๆ (composability) เราจะทำให้ pattern นี้เป็นทางการมากขึ้นด้วย generics ใน Part 045

---

## 6. Done Channel: กลไกยกเลิก pipeline กลางคัน

ปัญหาของ pipeline ที่เขียนไปข้างต้นคือ: ถ้าผู้บริโภคปลายทาง (main) **เลิกอ่าน** จาก `merged` channel ก่อนที่ทุก stage จะส่งข้อมูลครบ (เช่น เจอค่าที่ต้องการแล้วอยากหยุดทันที) goroutine ทุกตัวใน pipeline (`generator`, ทุก `square`, ทุก goroutine ใน `fanIn`) จะ**ค้างอยู่ตลอดไป** เพราะพยายามส่งข้อมูลเข้า channel ที่ไม่มีใครมาอ่านอีกแล้ว (blocked send) — นี่คือ **goroutine leak** ซึ่งเราจะพูดถึงอย่างละเอียดใน Part 043

Rob Pike เสนอวิธีแก้ใน talk เดียวกัน: ส่ง **done channel** (มักเป็น `chan struct{}`) ผ่านเข้าไปในทุก stage ของ pipeline แล้วให้ทุก stage คอย `select` ระหว่างการส่งข้อมูลปกติ กับการรับสัญญาณจาก `done`

```go
// generator เวอร์ชันที่รับ done channel เพิ่ม เพื่อให้หยุดส่งข้อมูลได้ทันทีที่ถูกสั่งยกเลิก
// รูปแบบนี้เป็นแนวคิดที่ Rob Pike อธิบายไว้ใน talk "Go Concurrency Patterns" (2012):
// ทุก stage ของ pipeline ต้อง "สังเกตเห็น" สัญญาณยกเลิกได้ ไม่เช่นนั้น goroutine จะค้าง (leak) ตลอดไป
func generator(done <-chan struct{}, nums ...int) <-chan int {
	out := make(chan int)
	go func() {
		defer close(out)
		for _, n := range nums {
			select {
			case out <- n:
			case <-done:
				return // เลิกส่งทันทีเมื่อ done ถูกปิด
			}
		}
	}()
	return out
}

func square(done <-chan struct{}, in <-chan int) <-chan int {
	out := make(chan int)
	go func() {
		defer close(out)
		for n := range in {
			select {
			case out <- n * n:
			case <-done:
				return
			}
		}
	}()
	return out
}

func fanIn(done <-chan struct{}, channels ...<-chan int) <-chan int {
	out := make(chan int)
	var wg sync.WaitGroup
	wg.Add(len(channels))

	for _, c := range channels {
		go func(c <-chan int) {
			defer wg.Done()
			for n := range c {
				select {
				case out <- n:
				case <-done:
					return
				}
			}
		}(c)
	}

	go func() {
		wg.Wait()
		close(out)
	}()

	return out
}
```

ตัวอย่างการใช้งาน: สร้างตัวเลข 1 ล้านตัว แต่อ่านแค่ 5 ค่าแรกแล้วเลิกอ่าน

```go
func main() {
	done := make(chan struct{})
	defer close(done) // รับประกันว่าทุก goroutine ใน pipeline จะได้รับสัญญาณเลิกงานเมื่อ main จบ

	// สร้างตัวเลข 1..1000000 แต่เราจะอ่านแค่ 5 ค่าแรกแล้วเลิกอ่านกลางคัน
	nums := make([]int, 1_000_000)
	for i := range nums {
		nums[i] = i + 1
	}

	source := generator(done, nums...)
	squared := square(done, source)

	count := 0
	for n := range squared {
		fmt.Println("ได้ผลลัพธ์:", n)
		count++
		if count == 5 {
			break // ออกจาก loop กลางคัน โดยไม่ต้องรอให้ pipeline ส่งครบทุกค่า
		}
	}

	fmt.Println("อ่านมาแค่", count, "ค่า แล้วเลิก — เพราะ defer close(done) goroutine ใน pipeline จะไม่ leak")
}
```

รันด้วย `go run -race main.go` ผลลัพธ์:

```
ได้ผลลัพธ์: 1
ได้ผลลัพธ์: 4
ได้ผลลัพธ์: 9
ได้ผลลัพธ์: 16
ได้ผลลัพธ์: 25
อ่านมาแค่ 5 ค่า แล้วเลิก — เพราะ defer close(done) goroutine ใน pipeline จะไม่ leak
```

โปรแกรมจบทันทีโดยไม่ค้าง แม้ `nums` จะมีตั้ง 1 ล้านตัว เพราะทันทีที่ `main()` return, `defer close(done)` จะทำงาน ปิด `done` channel ซึ่งทำให้ `select` ใน `generator` และ `square` ทุกตัวหลุดออกจาก blocked send ทันที (ไม่มี goroutine ค้างรอส่งข้อมูลที่ไม่มีใครอ่านอีกต่อไป)

### ทำไมต้องเป็น `chan struct{}` ไม่ใช่ `chan bool`

`struct{}` คือชนิดข้อมูลที่ **ไม่กินพื้นที่หน่วยความจำเลย** (zero-size type) ในเมื่อ done channel ใช้แค่ "สัญญาณ" ว่ามีเหตุการณ์เกิดขึ้นหรือไม่ ไม่ได้ใช้ค่าที่ส่งมาจริงๆ การใช้ `chan struct{}` จึงสื่อความหมายชัดเจนกว่า `chan bool` (ซึ่งจะทำให้คนอ่านโค้ดสงสัยว่า "แล้วถ้าส่ง `false` เข้ามาจะเกิดอะไรขึ้น") และเป็นธรรมเนียมมาตรฐานของโค้ด Go ที่ใช้ channel เป็นสัญญาณล้วนๆ

> **หมายเหตุ**: ในทางปฏิบัติปัจจุบัน โค้ด Go ส่วนใหญ่นิยมใช้ `context.Context` แทน `chan struct{}` แบบมือเปล่า เพราะ `ctx.Done()` ก็คืนค่าเป็น `<-chan struct{}` อยู่แล้ว แถมยังพ่วงความสามารถเรื่อง timeout, deadline และการส่งค่าประกอบ (`context.Value`) มาด้วย เราจะเจาะลึกการใช้ `context` แทน done channel มือเปล่าใน **Part 043**

---

## 7. เมื่อไรควรใช้ Fan-out/Fan-in และเมื่อไรไม่ควร

### ควรใช้เมื่อ

- งานแต่ละชิ้นเป็นอิสระจากกัน (ไม่ต้องพึ่งผลลัพธ์ของงานอื่น) และประมวลผลแบบขนานได้จริง
- ต้องการแยก concern ของแต่ละ stage ออกจากกันชัดเจน (เช่น stage อ่านไฟล์ / stage แปลงข้อมูล / stage เขียนฐานข้อมูล) และอยากให้แต่ละ stage ปรับระดับความขนานได้อิสระ (เช่น stage ที่ I/O-bound ใช้ worker เยอะกว่า stage ที่ CPU-bound)
- ต้องการ pipeline ที่ต่อยอด/แก้ไข/ทดสอบแต่ละ stage แยกกันได้ง่าย

### ไม่ควรใช้เมื่อ

- งานมีขั้นตอนเดียวและไม่ซับซ้อน — Worker Pool ธรรมดา (Part 041) ก็เพียงพอและอ่านง่ายกว่า
- ลำดับผลลัพธ์สำคัญมาก (fan-out/fan-in ทำให้ลำดับผลลัพธ์ไม่แน่นอนโดยธรรมชาติ ถ้าต้องการรักษาลำดับ ต้องเพิ่มกลไก sequencing เช่น ใส่ index กำกับแล้วเรียงลำดับใหม่ตอนท้าย)
- Overhead ของการสร้าง channel และ goroutine เพิ่มเติมในแต่ละ stage มากกว่าประโยชน์ที่ได้ (งานเบามากๆ อาจไม่คุ้มที่จะทำเป็น pipeline หลายขั้น)

---

## สรุปสิ่งที่ได้เรียนในบทนี้

- **Fan-out** คือหลาย goroutine อ่านจาก channel input เดียวกัน เพื่อกระจายงานประมวลผลแบบขนาน (Worker Pool ใน Part 041 คือกรณีพิเศษของ fan-out)
- **Fan-in** คือการรวมหลาย channel กลับเป็น channel เดียว โดยใช้ `sync.WaitGroup` รอให้ input channel ทุกตัวถูกปิดก่อนค่อยปิด output channel
- **Generator pattern** คือวิธีมาตรฐานในการเริ่ม pipeline: ฟังก์ชันคืนค่าเป็น channel ที่มี goroutine ป้อนข้อมูลอยู่เบื้องหลัง
- Pipeline ที่ประกอบจาก generator → fan-out → fan-in ทำให้แต่ละ stage มีหน้าที่ชัดเจนและนำกลับมาใช้ซ้ำ/ปรับขนาดแยกกันได้
- **Done channel** (`chan struct{}`) ที่ส่งผ่านทุก stage คือกลไกป้องกัน goroutine leak เมื่อผู้บริโภคปลายทางเลิกอ่านข้อมูลกลางคัน — แนวคิดนี้มาจาก talk "Go Concurrency Patterns" ของ Rob Pike
- ในทางปฏิบัติปัจจุบันนิยมใช้ `context.Context` แทน done channel มือเปล่า ซึ่งจะเรียนต่อใน Part 043

## แบบฝึกหัดท้ายบท

1. เพิ่ม stage ที่สามเข้าไปในตัวอย่างหัวข้อ 5: หลังจาก fan-in แล้ว ให้ fan-out อีกครั้งเพื่อกรองเฉพาะค่าที่เป็นเลขคู่ ก่อนจะรวมกลับด้วย fan-in อีกครั้ง (pipeline 5 stage รวม)
2. เขียนฟังก์ชัน `merge` (คือ `fanIn`) เวอร์ชันที่รับ channel ชนิด generic ใดๆ ก็ได้ (`<-chan T`) โดยใช้ Go generics ที่เรียนไปใน Part 028-029
3. ทดลองลบ `select`/`done` ออกจากตัวอย่างในหัวข้อ 6 (กลับไปใช้ `out <- n` ตรงๆ แบบหัวข้อ 2) แล้วรันพร้อม `break` กลางทาง สังเกตว่าโปรแกรมค้างหรือไม่ (ใบ้: ลองรันด้วย timeout เช่น `timeout 3 go run main.go` ใน terminal)
4. ปรับ `fanOut` ในหัวข้อ 3 ให้จำนวน worker ปรับได้ตาม `runtime.NumCPU()` แทนที่จะ hardcode เป็น 3
5. เขียน pipeline ที่ generator สร้างชื่อไฟล์จำลอง (string) แทนตัวเลข, stage หนึ่งจำลองการ "อ่านไฟล์" (`time.Sleep`), อีก stage จำลองการนับจำนวนตัวอักษร แล้ว fan-in ผลรวมความยาวทั้งหมด
6. ลองเขียน diagram (วาดมือหรือ ASCII art) ของ pipeline ในแบบฝึกหัดข้อ 1 อธิบายว่าที่จุดไหนของ pipeline เป็น fan-out และจุดไหนเป็น fan-in

---

**ต่อไป**: [Part 043 — Context กับ Concurrency: cancellation, timeout](./043-context-and-concurrency.md)
