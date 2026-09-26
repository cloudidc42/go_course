# Part 037: Channels พื้นฐาน

> ภาคที่ 3: การทำงานพร้อมกัน (Concurrency) — ตอนที่ 2 จาก 10 (Part 36–45)

## สารบัญของบทนี้

1. Channel คืออะไร และทำไมถึงเป็นหัวใจของ concurrency แบบ Go
2. สร้าง channel ด้วย `make`: Unbuffered vs Buffered
3. Syntax ส่งและรับค่า
4. Unbuffered Channel: กลไก "Rendezvous"
5. Buffered Channel: ความจุและจุดที่ block
6. ปิด channel ด้วย `close()`
7. รับค่าจาก channel ที่ปิดแล้ว: zero value และ comma-ok pattern
8. `range` บน channel จนกว่าจะปิด
9. Nil Channel: พฤติกรรมที่ block ตลอดไป
10. Directional Channel Types: `chan<-` และ `<-chan`
11. ตัวอย่างจริง: Producer/Consumer
12. สรุปสิ่งที่ได้เรียนในบทนี้
13. แบบฝึกหัดท้ายบท

---

## 1. Channel คืออะไร และทำไมถึงเป็นหัวใจของ concurrency แบบ Go

ใน Part 036 เราเรียนวิธีสร้าง**หน่วยงานที่ทำงานพร้อมกัน** ด้วย goroutine ไปแล้ว แต่ยังขาดเครื่องมือสำคัญอีกชิ้นหนึ่ง: ทำอย่างไรให้ goroutine เหล่านั้น **สื่อสารกัน** ได้อย่างปลอดภัย?

นี่คือจุดที่ **channel** เข้ามามีบทบาท ทบทวนปรัชญาที่เราเปิดเรื่องไว้ใน Part 036:

> "Do not communicate by sharing memory; share memory by communicating."

**Channel คือท่อสื่อสารที่มี type กำกับ (typed conduit)** ซึ่งใช้ส่งข้อมูลระหว่าง goroutine อย่างปลอดภัย พูดง่ายๆ คือ channel เป็นเหมือน "ท่อ" ที่ goroutine หนึ่งใส่ข้อมูลเข้าไปที่ปลายด้านหนึ่ง แล้วอีก goroutine หนึ่งดึงข้อมูลออกมาจากอีกปลายด้านหนึ่ง โดยที่ Go runtime รับประกันความปลอดภัยของการส่ง-รับนี้ให้เอง (ไม่ต้องใช้ mutex ป้องกันเอง)

จุดเด่นที่ทำให้ channel ต่างจาก queue ธรรมดาทั่วไปคือมันมี **type** กำกับชัดเจน — channel ที่ประกาศไว้สำหรับส่ง `int` จะส่งได้แค่ `int` เท่านั้น ส่ง `string` เข้าไปไม่ได้ ทำให้ compiler ช่วยตรวจสอบความถูกต้องให้ตั้งแต่ก่อนรันโปรแกรมเลย

---

## 2. สร้าง channel ด้วย `make`: Unbuffered vs Buffered

Channel สร้างด้วยฟังก์ชัน built-in `make` เหมือนกับ slice และ map ที่เรียนมาก่อนหน้านี้ (Part 006, Part 007) โดยมี 2 รูปแบบหลัก:

```go
ch1 := make(chan int)      // unbuffered channel (ความจุ 0)
ch2 := make(chan int, 5)   // buffered channel ความจุ 5
```

| รูปแบบ | ความหมาย |
|---|---|
| `make(chan T)` | **Unbuffered channel** — ไม่มีที่เก็บข้อมูลภายใน การส่งค่าต้องมีผู้รับพร้อมรับ ณ ขณะนั้นเท่านั้น |
| `make(chan T, n)` | **Buffered channel** ความจุ `n` — เก็บข้อมูลไว้ในตัวได้สูงสุด `n` ค่า โดยผู้ส่งไม่ต้องรอผู้รับทันที (ตราบใดที่ buffer ยังไม่เต็ม) |

ความแตกต่างระหว่างสองแบบนี้คือหัวใจสำคัญของการทำความเข้าใจ channel — เดี๋ยวจะอธิบายเจาะลึกทีละแบบ

---

## 3. Syntax ส่งและรับค่า

การส่ง (send) และรับ (receive) ค่าใน channel ใช้ operator `<-` (ลูกศรชี้ทิศทางการไหลของข้อมูล):

```go
ch := make(chan int, 1)

ch <- 42        // ส่งค่า 42 เข้า channel (ลูกศรชี้เข้าหา channel)
v := <-ch       // รับค่าออกจาก channel มาเก็บใน v (ลูกศรชี้ออกจาก channel)
```

จำง่ายๆ ว่า `<-` อยู่**ทางซ้ายของชื่อ channel** = รับค่า (receive), อยู่**ทางขวาของชื่อ channel** = ส่งค่า (send) ทิศทางของลูกศรบอกทิศทางการไหลของข้อมูลอย่างตรงไปตรงมา

---

## 4. Unbuffered Channel: กลไก "Rendezvous"

**Unbuffered channel** (`make(chan T)` ไม่ระบุความจุ หรือระบุ `0`) ไม่มีที่เก็บข้อมูลใดๆ ภายในตัวมันเองเลย ผลคือ:

> **การส่งค่าเข้า unbuffered channel จะ block (ค้างรอ) จนกว่าจะมี goroutine อื่นมารับค่านั้นพอดี ณ ขณะเดียวกัน**

พฤติกรรมนี้เรียกว่า **rendezvous** (แปลว่า "การนัดพบ") — ผู้ส่งกับผู้รับต้อง "มาเจอกัน" ในเวลาเดียวกันเท่านั้นการส่งจึงจะสำเร็จ เหมือนการส่งของมือต่อมือที่ต้องมีคนยื่นมือมารับพร้อมกันพอดี

```go
package main

import (
	"fmt"
	"time"
)

func main() {
	ch := make(chan string) // unbuffered channel

	go func() {
		fmt.Println("goroutine: กำลังทำงาน...")
		time.Sleep(200 * time.Millisecond)
		fmt.Println("goroutine: กำลังจะส่งค่า")
		ch <- "งานเสร็จแล้ว" // จะ block ตรงนี้จนกว่า main จะพร้อมรับ
		fmt.Println("goroutine: ส่งค่าสำเร็จ")
	}()

	fmt.Println("main: กำลังรอรับค่าจาก channel...")
	msg := <-ch // จะ block ตรงนี้จนกว่า goroutine จะส่งค่ามา
	fmt.Println("main: ได้รับค่าคือ", msg)
}
```

ผลลัพธ์:

```
main: กำลังรอรับค่าจาก channel...
goroutine: กำลังทำงาน...
goroutine: กำลังจะส่งค่า
goroutine: ส่งค่าสำเร็จ
main: ได้รับค่าคือ งานเสร็จแล้ว
```

สังเกตลำดับเหตุการณ์: `main` ไปถึงบรรทัด `<-ch` ก่อน (แล้ว block รออยู่ตรงนั้น) ส่วน goroutine sleep อยู่ 200ms ก่อนถึงจะส่งค่า พอ goroutine ส่งค่า `ch <- "งานเสร็จแล้ว"` ปุ๊บ ทั้งสองฝั่ง (ผู้ส่งกับผู้รับ) จะ **"synchronize" กันโดยอัตโนมัติ ณ จุดนั้นพอดี** — บรรทัด `fmt.Println("goroutine: ส่งค่าสำเร็จ")` จะทำงานหลังจากการส่งสำเร็จเท่านั้น (คือหลังจาก `main` รับค่าไปแล้วพอดี)

นี่คือคุณสมบัติพิเศษของ unbuffered channel ที่ไม่มีใน queue ทั่วไป: **มันทำหน้าที่เป็นกลไก synchronization ในตัว** ไม่ใช่แค่ที่เก็บข้อมูล — เมื่อการส่งสำเร็จ เรารู้ทันทีว่าฝั่งรับได้รับค่าไปแล้วจริงๆ ณ ขณะนั้น

---

## 5. Buffered Channel: ความจุและจุดที่ block

**Buffered channel** (`make(chan T, n)`) มีที่เก็บข้อมูลภายในขนาด `n` ทำให้ผู้ส่งสามารถส่งค่าเข้าไปได้โดยไม่ต้องรอผู้รับทันที **ตราบใดที่ buffer ยังไม่เต็ม**

```go
package main

import "fmt"

func main() {
	ch := make(chan int, 3) // buffered channel ความจุ 3

	ch <- 1
	fmt.Println("ส่งค่า 1 แล้ว, len =", len(ch), "cap =", cap(ch))
	ch <- 2
	fmt.Println("ส่งค่า 2 แล้ว, len =", len(ch), "cap =", cap(ch))
	ch <- 3
	fmt.Println("ส่งค่า 3 แล้ว, len =", len(ch), "cap =", cap(ch))

	// ถ้าส่งค่าที่ 4 ตอนนี้โดยไม่มีใครรับ จะ block ทันที เพราะ buffer เต็ม
	// ch <- 4 // ลอง uncomment บรรทัดนี้ดู โปรแกรมจะค้าง (deadlock) เพราะไม่มี goroutine อื่นมารับ

	fmt.Println("รับค่า:", <-ch)
	fmt.Println("รับค่า:", <-ch)
	fmt.Println("รับค่า:", <-ch)
}
```

ผลลัพธ์:

```
ส่งค่า 1 แล้ว, len = 1 cap = 3
ส่งค่า 2 แล้ว, len = 2 cap = 3
ส่งค่า 3 แล้ว, len = 3 cap = 3
รับค่า: 1
รับค่า: 2
รับค่า: 3
```

สังเกตว่า `len(ch)` บอกจำนวนค่าที่**ค้างอยู่ในบัฟเฟอร์ตอนนี้** ส่วน `cap(ch)` บอกความจุสูงสุดที่กำหนดไว้ตอนสร้าง (ทั้งสองฟังก์ชันนี้ใช้ได้กับ channel เหมือนที่ใช้กับ slice ตามที่เรียนใน Part 006)

ทดลองส่งค่าเกินความจุโดยไม่มีผู้รับดูว่าเกิดอะไรขึ้น:

```go
package main

func main() {
	ch := make(chan int, 3)
	ch <- 1
	ch <- 2
	ch <- 3
	ch <- 4 // buffer เต็มและไม่มีผู้รับ -> deadlock
}
```

รันแล้วโปรแกรมจะไม่ค้างเงียบๆ แต่ Go runtime ฉลาดพอที่จะตรวจจับ **deadlock** ได้และแจ้ง error ทันที:

```
fatal error: all goroutines are asleep - deadlock!

goroutine 1 [chan send]:
main.main()
	...
exit status 2
```

Go runtime ตรวจจับกรณีนี้ได้เพราะมันรู้ว่า **ไม่มี goroutine ไหนเหลืออยู่ที่กำลังทำงาน** (ทุกตัวกำลัง block รอกันหมด) จึงสรุปได้แน่ชัดว่าโปรแกรมนี้ไม่มีทางไปต่อได้อีกแล้ว นี่คือ safety net ที่มีประโยชน์มากตอนพัฒนา — deadlock ง่ายๆ แบบนี้จะไม่ทำให้โปรแกรมค้างอยู่เฉยๆ โดยไม่รู้สาเหตุ

สรุปกฎการ block ของ channel:

| การกระทำ | Unbuffered (`make(chan T)`) | Buffered (`make(chan T, n)`) |
|---|---|---|
| ส่งค่า (`ch <- v`) | Block จนกว่าจะมีผู้รับพร้อม | Block เฉพาะเมื่อ buffer **เต็ม** (`len == cap`) |
| รับค่า (`<-ch`) | Block จนกว่าจะมีผู้ส่งพร้อม | Block เฉพาะเมื่อ buffer **ว่าง** (`len == 0`) |

---

## 6. ปิด channel ด้วย `close()`

เมื่อผู้ส่งไม่มีข้อมูลจะส่งเข้า channel อีกแล้ว สามารถ **ปิด (close)** channel นั้นเพื่อบอกฝั่งผู้รับได้อย่างชัดเจน:

```go
ch := make(chan int, 3)
ch <- 1
ch <- 2
close(ch) // ปิด channel -- บอกว่าจะไม่มีการส่งค่าเข้ามาอีกแล้ว
```

กฎสำคัญเกี่ยวกับการปิด channel ที่ต้องจำให้แม่น:

1. **มีแค่ผู้ส่ง (sender) เท่านั้นที่ควรปิด channel** ไม่ใช่ผู้รับ — เพราะถ้าผู้รับปิด channel ไปเฉยๆ ผู้ส่งที่ยังพยายามส่งค่าเข้ามาอยู่จะ panic
2. **ห้ามปิด channel ที่ปิดไปแล้วซ้ำอีกครั้ง** — จะเกิด panic ทันที (`panic: close of closed channel`)
3. **ห้ามส่งค่าเข้า channel ที่ปิดไปแล้ว** — จะเกิด panic ทันที (`panic: send on closed channel`)
4. **การปิด channel ไม่ใช่ข้อบังคับเสมอไป** — ถ้าไม่มีใครต้องรอสัญญาณ "หมดข้อมูลแล้ว" ก็ไม่จำเป็นต้องปิด (channel ที่ไม่ได้ถูกปิดจะถูก garbage collect ไปตามปกติเมื่อไม่มีใครอ้างอิงถึงมันแล้ว) แต่ถ้าฝั่งรับใช้ `range` (หัวข้อถัดไป) การปิด channel เป็นสิ่งจำเป็นเพื่อให้ลูปจบได้

---

## 7. รับค่าจาก channel ที่ปิดแล้ว: zero value และ comma-ok pattern

คำถามสำคัญ: ถ้า channel ถูกปิดไปแล้ว แล้วมีคน `<-ch` อีก จะเกิดอะไรขึ้น? คำตอบคือ **ไม่ panic** — การรับค่าจาก channel ที่ปิดแล้วจะสำเร็จเสมอ โดยมีพฤติกรรมสองแบบขึ้นกับว่ายังมีข้อมูลค้างอยู่ใน buffer หรือไม่:

```go
package main

import "fmt"

func main() {
	ch := make(chan int, 2)
	ch <- 100
	ch <- 200
	close(ch) // ปิด channel -- ค่าที่ค้างอยู่ใน buffer ยังรับได้ตามปกติ

	v1, ok1 := <-ch
	fmt.Println(v1, ok1) // 100 true

	v2, ok2 := <-ch
	fmt.Println(v2, ok2) // 200 true

	v3, ok3 := <-ch
	fmt.Println(v3, ok3) // 0 false -- channel ว่างและปิดแล้ว ได้ zero value

	v4, ok4 := <-ch
	fmt.Println(v4, ok4) // 0 false เหมือนกัน อ่านซ้ำได้เรื่อยๆ ไม่ panic
}
```

ผลลัพธ์:

```
100 true
200 true
0 false
0 false
```

รูปแบบ `v, ok := <-ch` นี้เรียกว่า **comma-ok pattern** ซึ่งคุ้นเคยกันมาแล้วจากการทำ type assertion ใน Part 014 และการอ่านค่าจาก map ใน Part 007 — หลักการเดียวกันเป๊ะ:

- `ok == true`: ได้รับค่าจริงจากการส่งของผู้ส่ง (ไม่ว่า channel จะปิดไปแล้วหรือไม่ก็ตาม ตราบใดที่ค่านั้นถูกส่งเข้ามาก่อนปิด)
- `ok == false`: channel **ถูกปิดแล้วและไม่มีค่าเหลืออยู่ใน buffer** — ค่าที่ได้จะเป็น **zero value** ของ type นั้น (เช่น `0` สำหรับ `int`, `""` สำหรับ `string`, `nil` สำหรับ pointer/slice/map)

> **ข้อควรระวัง**: ถ้ารับค่าแบบ `v := <-ch` (ไม่ใช้ comma-ok) จาก channel ที่ปิดแล้วและว่างเปล่า จะได้ `v` เป็น zero value เงียบๆ โดยไม่รู้ว่าเป็นค่าจริงหรือเป็นเพราะ channel ปิดไปแล้ว — ถ้าต้อง**แยกแยะทั้งสองกรณีนี้ให้ออก** จำเป็นต้องใช้ comma-ok pattern เสมอ

---

## 8. `range` บน channel จนกว่าจะปิด

เหมือนที่แอบเกริ่นไว้ใน Part 005 ว่า `for range` ใช้กับ channel ได้เช่นกัน — และนี่คือรูปแบบที่ใช้บ่อยที่สุดในการอ่านค่าจาก channel จนกว่าจะหมด:

```go
package main

import "fmt"

func generate(n int) <-chan int {
	out := make(chan int)
	go func() {
		defer close(out) // สำคัญมาก: ต้อง close เมื่อส่งเสร็จ ไม่งั้น range ฝั่งรับจะค้างตลอดไป
		for i := 1; i <= n; i++ {
			out <- i * i
		}
	}()
	return out
}

func main() {
	for v := range generate(5) {
		fmt.Println("ได้รับ:", v)
	}
	fmt.Println("range จบเพราะ channel ถูกปิดแล้ว")
}
```

ผลลัพธ์:

```
ได้รับ: 1
ได้รับ: 4
ได้รับ: 9
ได้รับ: 16
ได้รับ: 25
range จบเพราะ channel ถูกปิดแล้ว
```

`for v := range ch` จะรับค่าจาก channel ไปเรื่อยๆ ทีละค่า และ**จบลูปโดยอัตโนมัติทันทีที่ channel ถูกปิดและไม่มีค่าเหลืออยู่** (เทียบเท่ากับการเขียน comma-ok pattern วนใน loop เองแบบ manual แต่กระชับกว่ามาก) สังเกตการใช้ `defer close(out)` ในฟังก์ชัน `generate` — เป็น pattern มาตรฐานที่บอกชัดเจนว่า "เมื่อฟังก์ชันที่ผลิตข้อมูล (producer) ทำงานเสร็จ ให้ปิด channel ทันที"

> **ย้ำอีกครั้งเพื่อความชัดเจน**: ถ้าลืม `close(out)` ในตัวอย่างข้างต้น โปรแกรมจะไม่จบ — `for range` ฝั่ง `main` จะติด block รอค่าถัดไปตลอดไป (ทั้งที่ผู้ส่งทำงานเสร็จไปแล้ว) นี่คือสาเหตุการเกิด **goroutine leak** ที่พบบ่อยที่สุดในโค้ด Go ที่เขียนไม่รัดกุม

---

## 9. Nil Channel: พฤติกรรมที่ block ตลอดไป

ตัวแปร channel ที่ประกาศไว้แต่ยังไม่ได้ `make()` จะมีค่าเป็น `nil` (ตามหลักการ zero value ของ reference type ที่เรียนมาตั้งแต่ Part 003 และ Part 010) การส่งหรือรับค่าจาก **nil channel** มีพฤติกรรมพิเศษที่ควรรู้จักไว้:

> **การส่งหรือรับค่าจาก nil channel จะ block ตลอดไปไม่มีวันสิ้นสุด (ไม่ panic แต่ไม่มีทาง ready ด้วย)**

ฟังดูเหมือนไม่มีประโยชน์ แต่จริงๆ แล้วพฤติกรรมนี้ถูกใช้เป็น**เทคนิค**ในการเขียนโค้ด `select` (ซึ่งจะเรียนเจาะลึกใน Part 038) เพื่อ "ปิดการทำงาน" ของบาง case ใน `select` แบบมีเงื่อนไข โดยไม่ต้องเขียน logic แยกซับซ้อน — เดี๋ยวจะได้เห็นการใช้งานจริงในบทหน้า

สาธิตพฤติกรรม block ตลอดไปแบบควบคุมได้ด้วย `time.After` (จะอธิบาย `select` เต็มรูปแบบใน Part 038 แต่ขอยืมมาใช้แสดงผลตรงนี้ก่อน):

```go
package main

import (
	"fmt"
	"time"
)

func main() {
	var ch chan int // nil channel -- ยังไม่ได้ make()

	select {
	case v := <-ch: // การรับจาก nil channel จะ block ตลอดไป ไม่มีวัน ready
		fmt.Println("ไม่มีทางมาถึงตรงนี้:", v)
	case <-time.After(200 * time.Millisecond):
		fmt.Println("timeout: ยืนยันว่า nil channel block ตลอดไปจริง จึงต้องใช้ select ควบคุม")
	}
}
```

ผลลัพธ์:

```
timeout: ยืนยันว่า nil channel block ตลอดไปจริง จึงต้องใช้ select ควบคุม
```

ยืนยันว่า case ที่รับจาก `nil channel` ไม่มีวันพร้อมทำงานเลย โปรแกรมจึงไปเข้า case `time.After` แทนเสมอ — ถ้าไม่มี timeout มาช่วยควบคุม โปรแกรมนี้จะค้างตลอดไปโดยไม่มี deadlock detector มาช่วย (เพราะ Go runtime มองว่า goroutine หลักยังมี "ความเป็นไปได้" ที่จะได้รับค่าอยู่ตามหลักทฤษฎี แม้ในทางปฏิบัติจะไม่มีวันเกิดขึ้นจริงก็ตาม)

---

## 10. Directional Channel Types: `chan<-` และ `<-chan`

Channel ที่สร้างด้วย `make(chan T)` เป็น **bidirectional** (ส่งได้ทั้งสองทิศทาง — ทั้งส่งและรับ) แต่เวลาส่ง channel เป็น parameter ให้ฟังก์ชันอื่น เราสามารถ**จำกัดทิศทาง**การใช้งานได้ด้วย **directional channel type**:

| Syntax | ความหมาย |
|---|---|
| `chan T` | Channel แบบสองทิศทาง (ส่งและรับได้ทั้งคู่) |
| `chan<- T` | **Send-only** channel — ใช้ส่งค่าเข้าไปได้อย่างเดียว รับค่าออกมาไม่ได้ |
| `<-chan T` | **Receive-only** channel — ใช้รับค่าออกมาได้อย่างเดียว ส่งค่าเข้าไปไม่ได้ |

```go
package main

import "fmt"

// sendOnly รับ channel แบบ send-only เท่านั้น -- ป้องกันไม่ให้ฟังก์ชันนี้อ่านค่าจาก channel โดยไม่ได้ตั้งใจ
func sendOnly(ch chan<- int, n int) {
	for i := 0; i < n; i++ {
		ch <- i
	}
	close(ch)
}

// receiveOnly รับ channel แบบ receive-only เท่านั้น -- คอมไพเลอร์จะ error ถ้าพยายามส่งค่าเข้าไปในฟังก์ชันนี้
func receiveOnly(ch <-chan int) int {
	sum := 0
	for v := range ch {
		sum += v
	}
	return sum
}

func main() {
	ch := make(chan int) // bidirectional channel ตอนสร้าง

	go sendOnly(ch, 5) // Go แปลง bidirectional -> directional ให้อัตโนมัติตอนส่งเป็น argument

	total := receiveOnly(ch)
	fmt.Println("ผลรวม:", total)
}
```

ผลลัพธ์:

```
ผลรวม: 10
```

ประโยชน์ของ directional channel type ไม่ใช่แค่เรื่อง performance (แทบไม่มีผลต่างด้าน runtime เลย) แต่เป็นเรื่อง **API ที่ชัดเจนและปลอดภัยกว่า (self-documenting API)**:

1. **สื่อเจตนาชัดเจน**: คนอ่านโค้ดเห็น `func sendOnly(ch chan<- int, ...)` แล้วรู้ทันทีว่าฟังก์ชันนี้มีหน้าที่ "ส่ง" ข้อมูลเข้า channel เท่านั้น ไม่ต้องเปิดดู implementation ข้างในก็เดาได้
2. **Compiler ช่วยตรวจสอบให้**: ถ้าเผลอเขียน `<-ch` ข้างในฟังก์ชัน `sendOnly` (พยายามรับค่าจาก channel ที่ประกาศเป็น send-only) จะเกิด **compile error** ทันที ไม่ต้องรอไปเจอบั๊กตอนรันจริง
3. **แปลงประเภทอัตโนมัติทางเดียว**: Go อนุญาตให้ส่ง bidirectional channel (`chan T`) เข้าไปในพารามิเตอร์ที่รับ directional channel (`chan<- T` หรือ `<-chan T`) ได้โดยอัตโนมัติ (implicit conversion) แต่ทำย้อนกลับไม่ได้ — เมื่อ "จำกัดสิทธิ์" ไปแล้วจะขยายสิทธิ์กลับไม่ได้อีก

รูปแบบนี้พบได้บ่อยมากในโค้ด Go จริง โดยเฉพาะฟังก์ชันที่คืนค่า channel ออกมา (return type มักเป็น `<-chan T` เพื่อบอกผู้เรียกว่า "อ่านค่าจาก channel นี้ได้อย่างเดียว ห้ามส่งค่าเข้าไปเอง")

---

## 11. ตัวอย่างจริง: Producer/Consumer

มาปิดท้ายบทนี้ด้วยตัวอย่างที่รวมทุกอย่างที่เรียนมาเข้าด้วยกัน: แพทเทิร์น **Producer/Consumer** ซึ่งเป็นรากฐานสำคัญที่จะต่อยอดไปสู่ **Worker Pool Pattern** ใน Part 041

```go
package main

import (
	"fmt"
	"sync"
)

// producer ผลิตงาน (job) ส่งเข้า channel jobs แล้ว close เมื่อผลิตครบ
func producer(jobs chan<- int, count int) {
	for i := 1; i <= count; i++ {
		jobs <- i
	}
	close(jobs)
}

// consumer อ่านงานจาก jobs จนกว่าจะปิด แล้วส่งผลลัพธ์เข้า results
func consumer(id int, jobs <-chan int, results chan<- string, wg *sync.WaitGroup) {
	defer wg.Done()
	for j := range jobs {
		results <- fmt.Sprintf("consumer %d ประมวลผล job %d ได้ %d", id, j, j*j)
	}
}

func main() {
	jobs := make(chan int, 5)
	results := make(chan string, 5)
	var wg sync.WaitGroup

	go producer(jobs, 5)

	const numConsumers = 3
	wg.Add(numConsumers)
	for i := 1; i <= numConsumers; i++ {
		go consumer(i, jobs, results, &wg)
	}

	// goroutine ปิด results เมื่อ consumer ทุกตัวทำงานเสร็จ
	go func() {
		wg.Wait()
		close(results)
	}()

	count := 0
	for r := range results {
		_ = r
		count++
	}
	fmt.Println("ประมวลผลครบทั้งหมด:", count, "งาน")
}
```

ผลลัพธ์ (จำนวนรวมจะคงที่เสมอที่ 5 แม้ลำดับการประมวลผลจริงจะสลับกันไปในแต่ละครั้งที่รัน เพราะ 3 consumer แข่งกันดึงงานจาก `jobs`):

```
ประมวลผลครบทั้งหมด: 5 งาน
```

โครงสร้างนี้มีรายละเอียดที่ควรสังเกตหลายจุด:

1. **`producer`** รับ `jobs` เป็น `chan<- int` (send-only) — หน้าที่ของมันคือใส่งานเข้าไปแล้วปิด channel เมื่อผลิตครบ
2. **`consumer`** รับ `jobs` เป็น `<-chan int` (receive-only) — ใช้ `range` วนอ่านงานจนกว่า `jobs` จะถูกปิดและว่างเปล่า จึงจบลูปเองอัตโนมัติ (ตามหลักการหัวข้อ 8) การที่มี consumer 3 ตัวอ่านจาก channel เดียวกันไม่มีปัญหาเรื่อง race เพราะ **channel รับประกันว่าแต่ละค่าที่ส่งเข้าไปจะถูกรับโดย goroutine เพียงตัวเดียวเท่านั้น** (ไม่มีทางที่สอง consumer จะได้ค่าเดียวกันซ้ำ)
3. **การปิด `results`** ต้องรอให้ consumer **ทุกตัว** ทำงานเสร็จก่อน (ไม่ใช่แค่ตัวใดตัวหนึ่ง) จึงต้องใช้ `sync.WaitGroup` (เรียนเจาะลึกใน Part 039) มาช่วยนับ แล้วใช้อีก goroutine แยกต่างหากคอย `wg.Wait()` แล้วค่อย `close(results)` — ถ้า close ไปตรงๆ ใน `main()` ก่อนที่ consumer จะทำงานเสร็จ จะเกิด panic "send on closed channel" ทันทีที่ consumer พยายามส่งผลลัพธ์เข้า channel ที่ปิดไปแล้ว

แพทเทิร์นนี้คือรากฐานของสถาปัตยกรรมการประมวลผลแบบขนานที่ใช้กันแพร่หลายในโค้ด Go จริง — เราจะขยายแนวคิดนี้ให้เป็นระบบที่สมบูรณ์และควบคุมจำนวน worker ได้อย่างมีขอบเขตใน **Part 041: Worker Pool Pattern**

---

## สรุปสิ่งที่ได้เรียนในบทนี้

- **Channel** คือท่อสื่อสารที่มี type กำกับ ใช้ส่งข้อมูลระหว่าง goroutine อย่างปลอดภัย สร้างด้วย `make(chan T)` (unbuffered) หรือ `make(chan T, n)` (buffered)
- **Unbuffered channel** ทำให้เกิด **rendezvous** — การส่งจะสำเร็จก็ต่อเมื่อมีผู้รับพร้อม ณ ขณะนั้นพอดี เป็นกลไก synchronization ในตัว
- **Buffered channel** ให้ผู้ส่งส่งค่าได้โดยไม่ต้องรอผู้รับ ตราบใดที่ `len(ch) < cap(ch)` — เมื่อ buffer เต็มจะ block ทันที
- `close(ch)` ปิด channel — ทำโดยผู้ส่งเท่านั้น ห้ามส่งค่าเข้า channel ที่ปิดแล้ว และห้ามปิดซ้ำ
- รับค่าด้วย **comma-ok pattern** (`v, ok := <-ch`) เพื่อแยกแยะว่าได้ค่าจริงหรือ channel ถูกปิดแล้ว (`ok == false` คู่กับ zero value)
- `for v := range ch` วนรับค่าจนกว่า channel จะถูกปิดและว่างเปล่า จบลูปเองอัตโนมัติ — ต้องมีใครสัก goroutine ปิด channel มิฉะนั้นจะ leak
- **Nil channel** block ตลอดไปทั้งการส่งและรับ — ใช้เป็นเทคนิคใน `select` ได้ (Part 038)
- **Directional channel type** (`chan<- T`, `<-chan T`) ทำให้ API ชัดเจนขึ้นและให้ compiler ช่วยตรวจสอบทิศทางการใช้งานให้
- แพทเทิร์น **Producer/Consumer** คือการนำทุกอย่างมาประกอบกัน และเป็นรากฐานของ Worker Pool Pattern ใน Part 041

## แบบฝึกหัดท้ายบท

1. เขียนโปรแกรมที่สร้าง unbuffered channel แล้วให้ goroutine หนึ่งส่งตัวเลข 1 ถึง 5 เข้า channel ทีละตัว โดย `main` รับค่าและพิมพ์ออกมาทีละตัวด้วย `range` (ต้อง `close` channel ให้ถูกจังหวะ)
2. เขียนโปรแกรมที่สร้าง buffered channel ความจุ 2 แล้วลองส่งค่า 3 ค่าจาก goroutine เดียวโดยไม่มี `main` มารับเลยจนกว่าจะส่งครบ สังเกตว่าเกิดอะไรขึ้น (ค้างหรือ deadlock?) แล้วอธิบายว่าทำไม
3. เขียนฟังก์ชันที่รับ channel แบบ `<-chan int` และคืนค่าผลรวมของทุกค่าที่รับได้จนกว่า channel จะปิด ทดสอบโดยส่ง channel จาก `make(chan int)` เข้าไปตรงๆ (ไม่ต้อง cast เอง) เพื่อยืนยันว่า Go แปลง bidirectional -> directional ให้อัตโนมัติ
4. ทดลองเขียนโค้ดที่พยายามส่งค่าเข้า channel ที่ปิดไปแล้ว (`panic: send on closed channel`) และโค้ดที่พยายามปิด channel ซ้ำสองครั้ง (`panic: close of closed channel`) บันทึก error message ที่ได้แล้วอธิบายด้วยคำพูดตัวเองว่าทำไมถึง panic
5. ดัดแปลงตัวอย่าง Producer/Consumer ในหัวข้อ 11 ให้มี producer 2 ตัวส่งงานเข้า channel เดียวกัน (ยังมี consumer 3 ตัวเหมือนเดิม) ต้องแก้ไขจุดไหนบ้างเพื่อให้ `jobs` channel ปิดถูกจังหวะ (ปิดหลังจาก producer ทั้งสองตัวส่งงานเสร็จแล้วเท่านั้น ไม่ใช่แค่ตัวใดตัวหนึ่ง)
6. ค้นคว้าเพิ่มเติม: เหตุใดการรับค่าจาก nil channel ถึงไม่ทำให้เกิด deadlock error แบบเดียวกับตัวอย่างในหัวข้อ 5 (การส่งเข้า buffered channel ที่เต็ม) ทั้งที่ทั้งสองกรณีต่างก็ "ค้างตลอดไป" เหมือนกัน? (คำใบ้: ลองคิดว่า deadlock detector ของ Go ตรวจสอบอะไรกันแน่)

---

**ต่อไป**: [Part 038 — คำสั่ง `select`](./038-select-statement.md)
