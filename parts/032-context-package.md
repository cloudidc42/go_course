# Part 032: แพ็กเกจ `context`

> ภาคที่ 2: ระดับกลาง (Intermediate) — ตอนที่ 17 จาก 20 (Part 16–35)

## สารบัญของบทนี้

1. `context` แก้ปัญหาอะไร
2. `context.Background()` และ `context.TODO()`
3. `context.WithCancel`: ยกเลิกงานด้วยมือ
4. เขียนฟังก์ชันที่เคารพสัญญาณยกเลิก: `ctx.Done()` และ `ctx.Err()`
5. `context.WithTimeout` และ `context.WithDeadline`
6. ทำไมต้อง `defer cancel()` เสมอ — และจะเกิดอะไรถ้าลืม
7. `context.WithValue`: ส่งข้อมูลไปกับ Context อย่างระมัดระวัง
8. Context เป็น Tree: การยกเลิก Context แม่ กระทบ Context ลูกทั้งหมด
9. Convention: `ctx context.Context` เป็น Parameter ตัวแรกเสมอ
10. ตัวอย่างเต็ม: HTTP Request ที่เคารพ Timeout
11. สรุปสิ่งที่ได้เรียนในบทนี้
12. แบบฝึกหัดท้ายบท

---

## 1. `context` แก้ปัญหาอะไร

ลองนึกภาพสถานการณ์นี้: มี HTTP handler ตัวหนึ่งที่รับ request จาก client แล้วต้องเรียก database query, เรียก API ภายนอกอีกตัว, และเขียน log — ทั้งหมดนี้อาจเกิดขึ้นในหลาย goroutine พร้อมกัน (เรื่อง goroutine จะเรียนเจาะลึกใน **ภาคที่ 3** ที่กำลังจะมาถึง แต่ตอนนี้ขอให้เข้าใจแค่ว่างานหนึ่ง request อาจกระจายไปทำงานหลายจุดพร้อมกันได้)

คำถามคือ: ถ้า client ปิดการเชื่อมต่อกลางคัน หรือ request ใช้เวลานานเกินไปจนต้องการยกเลิก **เราจะบอกงานทุกจุดที่กำลังทำงานอยู่ให้หยุดพร้อมกันได้อย่างไร?** และถ้าต้องการส่งข้อมูลบางอย่าง (เช่น request ID สำหรับ logging) ให้ทุกฟังก์ชันที่เกี่ยวข้องกับ request นี้เห็นเหมือนกันหมด โดยไม่ต้องแก้ signature ของทุกฟังก์ชันให้รับ parameter เพิ่ม จะทำอย่างไร?

นี่คือปัญหาที่ package มาตรฐาน **`context`** ถูกออกแบบมาแก้โดยเฉพาะ สรุปเป็น 3 หน้าที่หลัก:

1. **Cancellation signal (สัญญาณยกเลิก)** — กลไกส่งสัญญาณ "หยุดทำงานได้แล้ว" ไปยังทุกจุดที่กำลังทำงานอยู่พร้อมกัน โดยไม่ต้องมี reference ตรงไปยัง goroutine หรือฟังก์ชันนั้นๆ เลย
2. **Deadline/Timeout (เส้นตายเวลา)** — กำหนดเวลาสูงสุดที่งานควรทำเสร็จ ถ้าเกินเวลาจะถูกยกเลิกอัตโนมัติ
3. **Request-scoped values (ค่าที่ผูกกับ request)** — ส่งข้อมูลเล็กๆ น้อยๆ ที่เกี่ยวข้องกับ request หนึ่งๆ (เช่น request ID, ข้อมูล authentication) ให้ไหลผ่านทุกฟังก์ชันที่เกี่ยวข้องได้โดยไม่ต้องแก้ signature ของทุกฟังก์ชัน

ทั้งสามหน้าที่นี้ถูกรวมไว้ใน **`context.Context`** ซึ่งเป็น interface ตัวเดียว (ทบทวนหลักการ small interfaces จาก **Part 013**) ที่ถูกส่งผ่านไปตาม "สาย" การเรียกฟังก์ชัน (call chain) ตั้งแต่จุดเริ่มต้น (เช่น HTTP handler) ไปจนถึงจุดที่ลึกที่สุด (เช่น database query) — เราจะเห็นการใช้งาน `context` แบบเข้มข้นมากขึ้นใน **Part 043 (Context กับ Concurrency)** หลังจากเรียน goroutine และ channel ใน **ภาคที่ 3** และใน **Part 047 (HTTP Server เจาะลึก)** ที่ทุก `http.Request` มี `context.Context` ติดตัวมาโดยอัตโนมัติ

`context.Context` ถูกนิยามไว้ประมาณนี้ (แบบย่อ):

```go
type Context interface {
	Done() <-chan struct{}
	Err() error
	Deadline() (deadline time.Time, ok bool)
	Value(key any) any
}
```

บทนี้จะพาไปทำความเข้าใจ method ทั้ง 4 ตัวนี้ทีละตัว พร้อมฟังก์ชันสร้าง Context ที่ใช้บ่อยที่สุด

---

## 2. `context.Background()` และ `context.TODO()`

`context.Context` เป็น interface จึงต้องมีจุดเริ่มต้นเสมอ — package `context` มีฟังก์ชันสองตัวที่สร้าง **root context** (context ตัวต้นทางที่ไม่มี context แม่) ให้เลือกใช้:

```go
package main

import (
	"context"
	"fmt"
)

func main() {
	ctx1 := context.Background()
	ctx2 := context.TODO()
	fmt.Println(ctx1, ctx2)
	fmt.Println(ctx1.Err(), ctx2.Err())
}
```

ผลลัพธ์:

```
context.Background context.TODO
<nil> <nil>
```

ทั้งสองฟังก์ชันคืน Context เปล่าๆ ที่ **ไม่มีวันถูกยกเลิก ไม่มี deadline และไม่มี value ใดๆ** พฤติกรรมเหมือนกันทุกประการในทางเทคนิค — ความต่างมีแค่ **เจตนาในการสื่อสารกับคนอ่านโค้ด**:

| ฟังก์ชัน | ใช้เมื่อไหร่ |
|---|---|
| `context.Background()` | จุดเริ่มต้นที่ชัดเจนของโปรแกรม เช่น `func main()`, จุดเริ่มต้นของ HTTP server, หรือ context ระดับบนสุดของ test |
| `context.TODO()` | ยังไม่แน่ใจว่าควรใช้ context ตัวไหน หรือกำลัง refactor โค้ดเก่าที่ยังไม่รองรับ context แล้วยังไม่มีเวลาแก้ให้ครบ — เป็นเหมือน "placeholder" ที่บอกตัวเองและทีมว่า "ตรงนี้ต้องกลับมาแก้" |

โดยทั่วไปโค้ด production ส่วนใหญ่จะเริ่มต้นด้วย `context.Background()` เสมอ ส่วน `context.TODO()` มีประโยชน์ตอน refactor โค้ดเก่าให้รองรับ context ทีละส่วน

---

## 3. `context.WithCancel`: ยกเลิกงานด้วยมือ

`context.Background()`/`context.TODO()` เพียงอย่างเดียวยังทำอะไรไม่ได้มาก ความสามารถที่แท้จริงเริ่มต้นเมื่อเราสร้าง **child context** จาก context ตัวอื่นด้วยฟังก์ชันตระกูล `With...` ตัวแรกที่ควรรู้จักคือ `context.WithCancel`:

```go
func WithCancel(parent Context) (ctx Context, cancel CancelFunc)
```

ฟังก์ชันนี้คืนค่าสองตัวเสมอ: **child context ตัวใหม่** และ **ฟังก์ชัน `cancel`** ที่เมื่อเรียกแล้วจะส่งสัญญาณยกเลิกไปยัง context ตัวนั้น (และ context ลูกทั้งหมดของมัน — จะอธิบายเพิ่มในหัวข้อ 8)

---

## 4. เขียนฟังก์ชันที่เคารพสัญญาณยกเลิก: `ctx.Done()` และ `ctx.Err()`

การมี context ที่ยกเลิกได้ยังไม่มีประโยชน์ ถ้าฟังก์ชันที่ทำงานอยู่ **ไม่ยอมเช็ค**สัญญาณนั้นเลย — Go ไม่ได้บังคับให้ฟังก์ชันหยุดทำงานอัตโนมัติเมื่อ context ถูกยกเลิก (ต่างจากภาษาที่มี exception ที่อาจ throw ทันที) แต่**เป็นหน้าที่ของผู้เขียนโค้ดที่ต้องเช็คสัญญาณนี้เองอย่างสม่ำเสมอ** ผ่าน method สองตัว:

- **`ctx.Done()`** — คืนค่าเป็น channel (`<-chan struct{}`) ที่จะถูก "ปิด" (close) ทันทีที่ context นั้นถูกยกเลิกไม่ว่าจะด้วยเหตุผลใดก็ตาม (การอ่านค่าจาก channel ที่ปิดแล้วจะได้ผลลัพธ์ทันทีเสมอ ซึ่งเป็นกลไกมาตรฐานของ channel ที่จะเรียนเจาะลึกใน **Part 037**)
- **`ctx.Err()`** — คืนค่า `error` ที่บอกว่า**ทำไม**ถึงถูกยกเลิก มีค่าที่เป็นไปได้หลักๆ สองแบบ: `context.Canceled` (ถูกยกเลิกด้วยมือผ่าน `cancel()`) และ `context.DeadlineExceeded` (หมดเวลาตาม timeout/deadline) — ถ้า context ยังไม่ถูกยกเลิก `ctx.Err()` จะคืนค่า `nil`

รูปแบบมาตรฐาน (idiom) ที่ใช้เช็คสัญญาณยกเลิกในลูปการทำงาน คือใช้ `select` ร่วมกับ `ctx.Done()` (คำสั่ง `select` จะเรียนเจาะลึกเต็มรูปแบบใน **Part 038** บทนี้ขอใช้แค่รูปแบบพื้นฐานที่สุดพอให้เข้าใจภาพรวม):

```go
package main

import (
	"context"
	"fmt"
	"time"
)

// countUp จำลองงานที่ทำงานเป็นรอบๆ ต้องเช็ค ctx.Done() ทุกรอบเพื่อหยุดตามสัญญาณยกเลิก
func countUp(ctx context.Context) {
	i := 0
	for {
		select {
		case <-ctx.Done():
			fmt.Println("หยุดทำงานเพราะ:", ctx.Err())
			return
		default:
			i++
			fmt.Println("นับ:", i)
			time.Sleep(100 * time.Millisecond)
		}
	}
}

func main() {
	// ตัวอย่างที่ 1: WithCancel - ยกเลิกเองด้วยมือ
	ctx, cancel := context.WithCancel(context.Background())
	go func() {
		time.Sleep(350 * time.Millisecond)
		cancel() // เรียก cancel เพื่อส่งสัญญาณยกเลิก
	}()
	countUp(ctx)

	fmt.Println("---")

	// ตัวอย่างที่ 2: WithTimeout - ยกเลิกอัตโนมัติเมื่อครบเวลา
	ctx2, cancel2 := context.WithTimeout(context.Background(), 250*time.Millisecond)
	defer cancel2() // ต้องเรียกเสมอแม้ timeout จะทำงานเอง เพื่อคืน resource ทันที
	countUp(ctx2)
}
```

ผลลัพธ์ (จำนวนรอบ "นับ" อาจต่างกันเล็กน้อยตามความเร็วเครื่อง แต่รูปแบบจะเหมือนกันเสมอ):

```
นับ: 1
นับ: 2
นับ: 3
นับ: 4
หยุดทำงานเพราะ: context canceled
---
นับ: 1
นับ: 2
นับ: 3
หยุดทำงานเพราะ: context deadline exceeded
```

> **หมายเหตุ**: ตัวอย่างนี้ใช้ `go func() { ... }()` เพื่อรัน `cancel()` แบบขนานกับ `countUp` และใช้ `select` เพื่อเช็ค `ctx.Done()` — ทั้ง goroutine (`go func`) และ `select` เป็นหัวใจของ concurrency ใน Go ที่จะเรียนอย่างละเอียดใน **Part 036 (Goroutines)**, **Part 037 (Channels)** และ **Part 038 (select)** ที่กำลังจะเริ่มในภาคถัดไป ตอนนี้ขอให้เข้าใจแค่ภาพรวมว่า `ctx.Done()` คือ channel ที่ปิดตัวเองเมื่อ context ถูกยกเลิก และ `select` คือวิธีมาตรฐานในการรอฟังหลาย channel พร้อมกัน — รายละเอียดเชิงลึกทั้งหมดจะกลับมาอธิบายอีกครั้งใน **Part 043 (Context กับ Concurrency)**

จุดสำคัญที่ต้องจำจากตัวอย่างนี้คือ **`countUp` ต้องเช็ค `ctx.Done()` เองในทุกรอบของลูป** — ถ้าฟังก์ชันไม่เช็คเลย (เช่น ลืมใส่ `select` และเขียนแค่ `for { ... }` เฉยๆ) context ที่ถูกยกเลิกจะไม่มีผลอะไรกับฟังก์ชันนั้นเลย มันจะทำงานต่อไปเรื่อยๆ จนกว่าจะจบด้วยตัวเอง — **`context` เป็นเพียง "สัญญาณแจ้งเตือน" ไม่ใช่กลไกบังคับหยุดการทำงาน**

---

## 5. `context.WithTimeout` และ `context.WithDeadline`

จากตัวอย่างข้างบนจะเห็นแล้วว่า `context.WithTimeout` ทำงานคล้าย `WithCancel` แต่ยกเลิกอัตโนมัติเมื่อครบเวลาที่กำหนด โดยไม่ต้องเรียก `cancel()` เอง:

```go
func WithTimeout(parent Context, timeout time.Duration) (Context, CancelFunc)
func WithDeadline(parent Context, d time.Time) (Context, CancelFunc)
```

ความต่างระหว่างสองฟังก์ชันนี้มีแค่วิธีระบุเวลา:

- **`WithTimeout`** — ระบุเป็น **ระยะเวลา** นับจากตอนนี้ (เช่น "อีก 5 วินาทีข้างหน้า") เหมาะกับกรณีทั่วไปที่รู้แค่ว่า "งานนี้ไม่ควรใช้เวลาเกิน X"
- **`WithDeadline`** — ระบุเป็น **เวลาที่แน่นอน** (`time.Time`) เหมาะกับกรณีที่ต้องคำนวณเส้นตายจากที่อื่น เช่น deadline ของ request ที่ถูกส่งต่อมาจาก service อื่น หรือ deadline ที่ผูกกับเวลาปิดระบบตอนเที่ยงคืน

จริงๆ แล้ว `WithTimeout` ภายในก็แค่เรียก `WithDeadline(parent, time.Now().Add(timeout))` ให้เท่านั้นเอง — เป็นแค่ทางลัดที่สะดวกกว่าสำหรับกรณีที่พบบ่อยที่สุด

```go
package main

import (
	"context"
	"fmt"
	"time"
)

func main() {
	deadline := time.Now().Add(200 * time.Millisecond)
	ctx, cancel := context.WithDeadline(context.Background(), deadline)
	defer cancel()

	select {
	case <-time.After(500 * time.Millisecond):
		fmt.Println("งานเสร็จตามปกติ")
	case <-ctx.Done():
		fmt.Println("เกิน deadline:", ctx.Err())
	}
}
```

ผลลัพธ์:

```
เกิน deadline: context deadline exceeded
```

ในตัวอย่างนี้งานสมมติใช้เวลา 500ms แต่ deadline กำหนดไว้แค่ 200ms จึงถูกยกเลิกก่อนที่งานจะเสร็จ — สังเกตว่า `ctx.Err()` คืนค่า `context.DeadlineExceeded` ต่างจากตัวอย่างก่อนหน้าที่ยกเลิกด้วยมือซึ่งได้ `context.Canceled`

---

## 6. ทำไมต้อง `defer cancel()` เสมอ — และจะเกิดอะไรถ้าลืม

ทุกฟังก์ชันในตระกูล `With...` (`WithCancel`, `WithTimeout`, `WithDeadline`, และ `WithValue` ที่จะเรียนต่อไป) คืนค่า `cancel` function มาด้วยเสมอ และ**กฎเหล็กที่ต้องจำขึ้นใจคือต้องเรียก `cancel()` เสมอ ไม่ว่า context นั้นจะหมดเวลาไปเองแล้วหรือไม่ก็ตาม** โดยทั่วไปเขียนเป็น idiom มาตรฐาน:

```go
ctx, cancel := context.WithTimeout(parent, 5*time.Second)
defer cancel()
```

**ทำไมต้องเรียกแม้ว่า timeout จะหมดอายุไปเองอยู่แล้ว?** เพราะทุกครั้งที่สร้าง context ด้วย `With...` Go จะจอง resource ภายในไว้เพื่อติดตามสถานะของ context นั้น (เช่น timer ที่คอยนับเวลาถอยหลังสำหรับ `WithTimeout`/`WithDeadline`) resource เหล่านี้จะถูกปล่อยคืนก็ต่อเมื่อ **`cancel()` ถูกเรียก** เท่านั้น ถ้าไม่เคยเรียกเลย:

- Timer ภายในจะยังคงทำงานอยู่ในหน่วยความจำจนกว่าจะถึงเวลา deadline จริงๆ (สำหรับ `WithTimeout`/`WithDeadline`) หรือ **ค้างอยู่ตลอดไปไม่มีวันถูกเก็บกวาด** (สำหรับ `WithCancel` ที่ไม่มี timeout ในตัว)
- ถ้าโปรแกรมสร้าง context แบบนี้ซ้ำๆ จำนวนมาก (เช่น สร้าง context ใหม่ทุกครั้งที่มี request เข้ามา) โดยไม่เรียก `cancel()` จะเกิด **resource leak (การรั่วไหลของทรัพยากร)** สะสมไปเรื่อยๆ ทำให้โปรแกรมใช้หน่วยความจำเพิ่มขึ้นเรื่อยๆ จนอาจ crash ในระยะยาว

เรื่องนี้สำคัญมากจนทีม Go สร้างเครื่องมือตรวจจับบั๊กนี้ไว้ใน **`go vet`** (ที่แนะนำให้รู้จักตั้งแต่ **Part 001**) โดยเฉพาะ — ลองดูตัวอย่างโค้ดที่มีบั๊กนี้:

```go
package main

import (
	"context"
	"fmt"
	"time"
)

func leaky() {
	ctx, _ := context.WithTimeout(context.Background(), time.Second) // ทิ้ง cancel ไปเฉยๆ ด้วย _
	fmt.Println(ctx)
}

func main() {
	leaky()
}
```

รันคำสั่ง `go vet` กับโค้ดนี้:

```bash
go vet .
```

ผลลัพธ์:

```
./main.go:10:7: the cancel function returned by context.WithTimeout should be called, not discarded, to avoid a context leak
```

`go vet` ตรวจจับ pattern นี้ได้อัตโนมัติเมื่อเห็นว่า `cancel` ถูกทิ้งไปด้วย `_` โดยไม่มีการเรียกใช้เลยในทุกเส้นทางของฟังก์ชัน — เป็นตัวอย่างที่ดีมากว่าทำไม `go vet` ถึงเป็นเครื่องมือที่ควรรันเป็นประจำ ไม่ใช่แค่ `go build`/`go run` เฉยๆ

**กฎทองคำที่ต้องจำ**: หลังสร้าง context ด้วยฟังก์ชันตระกูล `With...` ทุกครั้ง ให้เขียน `defer cancel()` เป็นบรรทัดถัดไปทันทีเป็นนิสัย เหมือนกับที่เรียน `defer file.Close()` หลังเปิดไฟล์ใน **Part 024**

---

## 7. `context.WithValue`: ส่งข้อมูลไปกับ Context อย่างระมัดระวัง

นอกจากสัญญาณยกเลิกและ deadline แล้ว `context` ยังมีความสามารถที่สามคือการพก **ค่าที่ผูกกับ request** ไปด้วย ผ่าน:

```go
func WithValue(parent Context, key, val any) Context
```

```go
package main

import (
	"context"
	"fmt"
)

// contextKey เป็น type เฉพาะของ package นี้ ป้องกัน key ชนกับ package อื่นที่ใช้ context.WithValue เหมือนกัน
type contextKey string

const requestIDKey contextKey = "requestID"

// logWithRequestID จำลอง logging function ที่ดึง request ID ที่ฝังไว้ใน context มาแปะทุกบรรทัด log
func logWithRequestID(ctx context.Context, message string) {
	reqID, ok := ctx.Value(requestIDKey).(string)
	if !ok {
		reqID = "unknown"
	}
	fmt.Printf("[request-id=%s] %s\n", reqID, message)
}

func handleRequest(ctx context.Context) {
	logWithRequestID(ctx, "เริ่มประมวลผล request")
	// ...ทำงานต่อ ส่ง ctx เดิมนี้ต่อไปยังฟังก์ชันอื่นๆ...
	logWithRequestID(ctx, "ประมวลผลเสร็จสิ้น")
}

func main() {
	ctx := context.WithValue(context.Background(), requestIDKey, "req-12345")
	handleRequest(ctx)

	// ถ้าไม่มี value ก็ยังทำงานได้ปกติ แค่ได้ "unknown"
	handleRequest(context.Background())
}
```

ผลลัพธ์:

```
[request-id=req-12345] เริ่มประมวลผล request
[request-id=req-12345] ประมวลผลเสร็จสิ้น
[request-id=unknown] เริ่มประมวลผล request
[request-id=unknown] ประมวลผลเสร็จสิ้น
```

จุดสำคัญของโค้ดตัวอย่างนี้:

- **ใช้ type เฉพาะ (`contextKey`) เป็น key แทน string ธรรมดา** — นี่ไม่ใช่แค่สไตล์ แต่เป็นวิธีป้องกันปัญหาจริง: ถ้าสอง package ต่างใช้ string ธรรมดาเป็น key (เช่น `"requestID"`) โดยไม่รู้จักกัน อาจเกิด key ชนกันโดยไม่ตั้งใจ ทำให้ค่าที่ package หนึ่งเซ็ตไว้ถูก package อื่นเขียนทับโดยไม่รู้ตัว การสร้าง type ของตัวเอง (`type contextKey string`) ทำให้ key ของแต่ละ package แยกจากกันทาง type system อย่างสมบูรณ์ แม้ค่า string ข้างในจะเหมือนกันก็ตาม
- `ctx.Value(key)` คืนค่าเป็น `any` เสมอ ต้อง type assertion (ทบทวนจาก **Part 014**) ก่อนใช้งาน และควรใช้ comma-ok idiom เพื่อป้องกัน panic ถ้า key ไม่มีอยู่จริงหรือ value ไม่ตรง type ที่คาดไว้

### ทำไมต้องใช้ `WithValue` อย่างระมัดระวัง

นี่คือความสามารถของ `context` ที่ถูกใช้ผิดบ่อยที่สุด กฎที่ควรยึดถืออย่างเคร่งครัดคือ:

> **`context.WithValue` ควรใช้เฉพาะกับข้อมูล "metadata ที่ผูกกับ request" เท่านั้น เช่น request ID, trace ID สำหรับ distributed tracing, ข้อมูล user ที่ authenticate แล้ว — ไม่ควรใช้เป็นช่องทางส่ง parameter ปกติของฟังก์ชันเด็ดขาด**

เหตุผลที่ต้องระมัดระวัง:

1. **ไม่มี type safety เลย** — `Value(key any) any` รับ-คืนค่าเป็น `any` ทั้งคู่ ทำให้ compiler ตรวจสอบไม่ได้เลยว่าค่าที่ใส่เข้าไปกับค่าที่ดึงออกมาเป็น type เดียวกันจริงหรือไม่ ต่างจาก parameter ปกติของฟังก์ชันที่ compiler เช็ค type ให้เต็มรูปแบบ
2. **ทำให้ dependency ของฟังก์ชันไม่ชัดเจน (hidden dependency)** — ถ้าฟังก์ชันต้องพึ่งค่าจาก context จะไม่มีทางรู้จาก signature ของฟังก์ชันเลยว่ามันต้องการอะไรบ้าง ต้องไปไล่อ่านโค้ดข้างในถึงจะรู้ ต่างจาก parameter ปกติที่เห็นชัดเจนจาก signature
3. **Debug ยาก** — ถ้าค่าที่ควรมีใน context หายไปกลางทาง (เช่น มีคนสร้าง context ใหม่จาก `context.Background()` แทนที่จะสืบทอดจาก context เดิม) จะไม่มี compile error เตือนเลย จะรู้ตัวอีกทีตอน runtime panic หรือได้ค่าผิดที่ไม่คาดคิด

**แนวทางปฏิบัติที่แนะนำ**: ถ้าข้อมูลนั้นเป็น "input ที่จำเป็น" ต่อการทำงานของฟังก์ชัน (เช่น user ID ที่ต้องใช้ query database) ให้ส่งเป็น **parameter ปกติ** ของฟังก์ชันเสมอ ใช้ `context.WithValue` เฉพาะกับข้อมูลที่เป็น **cross-cutting concern** จริงๆ ที่ไหลผ่านหลายชั้นของโค้ดโดยที่แต่ละชั้นไม่จำเป็นต้องรู้จักหรือใช้มันโดยตรง เช่น trace ID สำหรับ logging/monitoring ที่จะเจอเต็มรูปแบบใน **Part 099 (Monitoring และ Observability)**

---

## 8. Context เป็น Tree: การยกเลิก Context แม่ กระทบ Context ลูกทั้งหมด

จุดที่มักไม่ได้พูดถึงตรงๆ แต่สำคัญมากคือ Context ทุกตัวที่สร้างจากฟังก์ชัน `With...` จะเก็บ**อ้างอิงกลับไปยัง context แม่** เสมอ ทำให้ context ทั้งหมดในโปรแกรมหนึ่งๆ ก่อตัวเป็นโครงสร้างแบบ **tree** (ต้นไม้) โดยมี `context.Background()` เป็นราก

กฎสำคัญที่ตามมาจากโครงสร้าง tree นี้คือ:

> **ถ้ายกเลิก context ตัวใดตัวหนึ่ง (เรียก `cancel()` หรือ timeout หมดอายุ) context ลูกทุกตัวที่สร้างต่อจากมัน (ไม่ว่าจะซ้อนกันกี่ชั้น) จะถูกยกเลิกตามไปด้วยโดยอัตโนมัติทันที**

แต่ทิศทางตรงข้ามไม่เป็นจริง: **การยกเลิก context ลูก ไม่มีผลอะไรกับ context แม่หรือ context พี่น้องเลย** เพราะสัญญาณไหล "ลงล่าง" ทางเดียวจากรากไปยังใบเสมอ ไม่ไหลย้อนกลับ

ลักษณะนี้ตรงกับการใช้งานจริงมาก: สมมติ HTTP handler สร้าง context หลักของ request หนึ่งตัว แล้วส่งต่อ (หรือสร้าง child context ที่มี timeout สั้นลงเฉพาะจุด) ไปให้ฟังก์ชันย่อยหลายตัวที่ทำงานพร้อมกัน — ถ้า client ยกเลิก request (context หลักถูกยกเลิก) งานย่อยทุกตัวที่กำลังทำอยู่ (query database, เรียก API ภายนอก, เขียนไฟล์) จะได้รับสัญญาณยกเลิกพร้อมกันทันที โดยที่ handler ไม่ต้องไล่ยกเลิกทีละจุดเอง — นี่คือพลังที่แท้จริงของการออกแบบ `context` แบบ tree และเป็นหัวข้อที่จะเจาะลึกอีกครั้งใน **Part 043**

---

## 9. Convention: `ctx context.Context` เป็น Parameter ตัวแรกเสมอ

Go community มี convention ที่ยึดถือกันอย่างเคร่งครัดมากเรื่องหนึ่ง (แม้จะไม่มีการบังคับจาก compiler เลย) คือ:

> **ฟังก์ชันใดก็ตามที่ต้องการรับ context ควรรับมันเป็น parameter ตัวแรกเสมอ และตั้งชื่อว่า `ctx`**

```go
// ถูกต้องตาม convention
func FetchUser(ctx context.Context, id int) (*User, error) { ... }

// ผิด convention -- ไม่ควรทำ
func FetchUser(id int, ctx context.Context) (*User, error) { ... }
```

Convention นี้แข็งแรงมากถึงขั้นที่ **`go vet` มีการตรวจสอบเรื่องนี้เช่นกัน** (ผ่าน analyzer หนึ่งใน `go vet` แบบขยาย) และ code reviewer ในโปรเจกต์ Go แทบทุกที่จะขอให้แก้ทันทีถ้าเห็น context ไม่ได้อยู่ตำแหน่งแรก เหตุผลคือความสม่ำเสมอ (consistency) ทำให้อ่านโค้ดคนอื่นได้เร็วขึ้นมาก เพราะรู้เสมอว่า parameter ตัวแรกของฟังก์ชันแทบทุกตัวในระบบคือ context

อีกกฎที่มาคู่กันคือ **ห้ามเก็บ `context.Context` ไว้เป็น field ของ struct** (ยกเว้นกรณีพิเศษน้อยมาก เช่น struct ที่ implement `context.Context` เอง) เพราะ context ถูกออกแบบมาให้ "ไหลผ่าน" การเรียกฟังก์ชัน ไม่ใช่ถูก "เก็บไว้" ยาวนาน — ถ้าจำเป็นต้องส่ง context ให้ method ของ struct ให้ส่งเป็น parameter ของ method นั้นแทน ไม่ใช่ผูกไว้กับ struct ตอนสร้าง object

---

## 10. ตัวอย่างเต็ม: HTTP Request ที่เคารพ Timeout

มาดูตัวอย่างที่ใกล้เคียงกับการใช้งานจริงมากที่สุด — ฟังก์ชันที่เรียก HTTP request ออกไปยัง server ภายนอก โดยเคารพ timeout ที่กำหนดผ่าน context (แนวคิดนี้จะเจาะลึกเต็มรูปแบบใน **Part 046: `net/http` — HTTP Client เจาะลึก** และ **Part 047: `net/http` — HTTP Server เจาะลึก** บทนี้ขอสาธิตแค่หลักการที่เกี่ยวกับ context เท่านั้น)

```go
package main

import (
	"context"
	"fmt"
	"net/http"
	"net/http/httptest"
	"time"
)

func fetchWithTimeout(ctx context.Context, url string) error {
	req, err := http.NewRequestWithContext(ctx, http.MethodGet, url, nil)
	if err != nil {
		return fmt.Errorf("สร้าง request ไม่สำเร็จ: %w", err)
	}

	resp, err := http.DefaultClient.Do(req)
	if err != nil {
		return fmt.Errorf("เรียก HTTP ไม่สำเร็จ: %w", err)
	}
	defer resp.Body.Close()

	fmt.Println("สถานะ:", resp.Status)
	return nil
}

func main() {
	// จำลอง server ที่ตอบช้ามาก (600ms) เพื่อทดสอบ timeout
	slowServer := httptest.NewServer(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		time.Sleep(600 * time.Millisecond)
		w.WriteHeader(http.StatusOK)
	}))
	defer slowServer.Close()

	ctx, cancel := context.WithTimeout(context.Background(), 200*time.Millisecond)
	defer cancel()

	if err := fetchWithTimeout(ctx, slowServer.URL); err != nil {
		fmt.Println("เกิดข้อผิดพลาด:", err)
	}
}
```

ผลลัพธ์:

```
เกิดข้อผิดพลาด: เรียก HTTP ไม่สำเร็จ: Get "http://127.0.0.1:xxxxx": context deadline exceeded
```

จุดสำคัญที่ต้องสังเกต:

- `http.NewRequestWithContext(ctx, ...)` คือวิธีมาตรฐานในการผูก context เข้ากับ HTTP request — เมื่อ context ถูกยกเลิกหรือหมดเวลา `http.DefaultClient.Do(req)` จะ**ยกเลิก connection ทันที** และคืน error กลับมาแทนที่จะรอ response จนกว่า server จะตอบ (ในตัวอย่างนี้ server จงใจตอบช้าถึง 600ms แต่ context กำหนด timeout ไว้แค่ 200ms คำขอจึงถูกยกเลิกก่อนได้รับคำตอบ)
- ใช้ `httptest.NewServer` (จาก package `net/http/httptest`) เพื่อจำลอง server จริงในการทดสอบโดยไม่ต้องพึ่ง network ภายนอก — เครื่องมือนี้จะเรียนอย่างละเอียดใน **Part 081 (`httptest` — ทดสอบ HTTP Handler)** บทนี้แค่ใช้มันเป็นเครื่องมือช่วยสาธิตเท่านั้น
- error ที่ได้จาก `http.DefaultClient.Do` จะห่อ (wrap) ข้อความ `context deadline exceeded` เอาไว้ข้างใน — สามารถใช้ `errors.Is(err, context.DeadlineExceeded)` (ทบทวนจาก **Part 016**) เพื่อตรวจสอบสาเหตุที่แท้จริงได้อย่างแม่นยำ แทนการเทียบ error message เป็น string ตรงๆ ซึ่งเป็นวิธีที่ไม่แนะนำ

มาพิสูจน์การใช้ `errors.Is` กับ context error ให้เห็นชัดๆ อีกครั้ง:

```go
package main

import (
	"context"
	"errors"
	"fmt"
	"net/http"
	"net/http/httptest"
	"time"
)

func main() {
	slowServer := httptest.NewServer(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		time.Sleep(500 * time.Millisecond)
	}))
	defer slowServer.Close()

	ctx, cancel := context.WithTimeout(context.Background(), 100*time.Millisecond)
	defer cancel()

	req, _ := http.NewRequestWithContext(ctx, http.MethodGet, slowServer.URL, nil)
	_, err := http.DefaultClient.Do(req)

	if errors.Is(err, context.DeadlineExceeded) {
		fmt.Println("ยืนยันแล้ว: request ถูกยกเลิกเพราะ timeout จริงๆ ไม่ใช่ error อื่น")
	} else {
		fmt.Println("error อื่นที่ไม่ใช่ timeout:", err)
	}
}
```

ผลลัพธ์:

```
ยืนยันแล้ว: request ถูกยกเลิกเพราะ timeout จริงๆ ไม่ใช่ error อื่น
```

แม้ error message ดิบที่ได้จาก `http.DefaultClient.Do` จะมีรายละเอียดอื่นปนมาด้วย (เช่น URL, ชื่อ method) แต่ `errors.Is` สามารถ "มองทะลุ" ข้อความเหล่านั้นไปหา error ต้นตอที่แท้จริง (`context.DeadlineExceeded`) ที่ถูกห่อซ้อนอยู่ข้างในได้อย่างแม่นยำ — นี่คือเหตุผลที่ **Part 016** เน้นย้ำว่าไม่ควรเทียบ error ด้วยการเทียบข้อความ string ตรงๆ เพราะข้อความอาจเปลี่ยนแปลงได้ตลอดเวลาโดยไม่กระทบ logic ที่ถูกต้อง

ตัวอย่างนี้แสดงให้เห็นภาพรวมของทั้งบทอย่างครบถ้วน: สร้าง context ด้วย `WithTimeout`, ส่งต่อเป็น parameter ตัวแรกตาม convention, ผูกเข้ากับการทำงานจริง (HTTP request), และ `defer cancel()` เพื่อไม่ให้ resource รั่วไหล — รูปแบบนี้จะเป็นรากฐานสำคัญเมื่อไปเรียน HTTP server/client อย่างเต็มรูปแบบใน **ภาคที่ 5 (Web Development)**

---

## สรุปสิ่งที่ได้เรียนในบทนี้

- `context.Context` แก้ 3 ปัญหาหลัก: **cancellation signal**, **deadline/timeout**, และ **การส่ง request-scoped value** ข้ามฟังก์ชันและ goroutine
- `context.Background()` ใช้เป็นจุดเริ่มต้นของโปรแกรมจริง ส่วน `context.TODO()` ใช้เป็น placeholder ตอนยังไม่แน่ใจหรือกำลัง refactor
- `context.WithCancel` สร้าง context ที่ยกเลิกได้ด้วยมือผ่านฟังก์ชัน `cancel` ที่คืนมาด้วย
- ฟังก์ชันที่ทำงานยาวต้องเช็ค `ctx.Done()` (channel ที่ปิดเมื่อถูกยกเลิก) เองอย่างสม่ำเสมอ context ไม่ได้บังคับหยุดการทำงานให้อัตโนมัติ และ `ctx.Err()` บอกสาเหตุที่ถูกยกเลิก (`context.Canceled` หรือ `context.DeadlineExceeded`)
- `context.WithTimeout` (ระบุระยะเวลา) และ `context.WithDeadline` (ระบุเวลาที่แน่นอน) ยกเลิก context อัตโนมัติเมื่อครบกำหนด
- **ต้องเรียก `defer cancel()` เสมอ** หลังสร้าง context จากฟังก์ชันตระกูล `With...` ทุกครั้ง ไม่เช่นนั้นจะเกิด resource leak — `go vet` ช่วยตรวจจับบั๊กนี้ได้อัตโนมัติ
- `context.WithValue` ใช้ส่งข้อมูล metadata ที่ผูกกับ request เท่านั้น (เช่น request ID, trace ID) **ไม่ใช่ช่องทางส่ง parameter ปกติของฟังก์ชัน** เพราะไม่มี type safety และทำให้ dependency ของฟังก์ชันไม่ชัดเจน
- Context ก่อตัวเป็น **tree** เสมอ — ยกเลิก context แม่จะยกเลิก context ลูกทั้งหมดตามไปด้วยอัตโนมัติ แต่ทิศทางตรงข้ามไม่เป็นจริง
- Convention ที่ต้องยึดถือเคร่งครัด: **`ctx context.Context` ต้องเป็น parameter ตัวแรกเสมอ** และห้ามเก็บ context ไว้เป็น field ของ struct
- `http.NewRequestWithContext` คือวิธีมาตรฐานในการผูก context กับ HTTP request เพื่อให้ยกเลิก/timeout ได้ตามที่กำหนด — จะเจาะลึกเต็มรูปแบบใน **Part 046-047**

## แบบฝึกหัดท้ายบท

1. เขียนฟังก์ชัน `processItems(ctx context.Context, items []string)` ที่วนประมวลผล item ทีละตัว (จำลองด้วย `time.Sleep` สั้นๆ และ `fmt.Println`) แต่หยุดทันทีถ้า `ctx` ถูกยกเลิกระหว่างทาง ทดสอบด้วย `context.WithTimeout` ที่ตั้งเวลาให้สั้นกว่าเวลาที่ใช้ประมวลผลทั้งหมด
2. เขียนโปรแกรมที่สร้าง context ด้วย `context.WithCancel` แล้วลองไม่เรียก `cancel()` เลย (ทิ้งด้วย `_`) จากนั้นรัน `go vet .` ดูว่า Go เตือนอะไรบ้าง แล้วแก้ไขให้ถูกต้อง
3. เขียนฟังก์ชัน `withRequestID(ctx context.Context, id string) context.Context` และ `getRequestID(ctx context.Context) (string, bool)` โดยใช้ custom key type ตามหลักการในหัวข้อ 7 แล้วทดสอบว่าดึงค่ากลับมาได้ถูกต้อง
4. สร้าง context แม่ด้วย `context.WithCancel` แล้วสร้าง context ลูกอีก 2 ตัวจากมันด้วย `context.WithTimeout` คนละค่า ลองเรียก `cancel()` ของ context แม่แล้วพิสูจน์ว่า context ลูกทั้งสองตัวถูกยกเลิกตามไปด้วยจริง (เช็คด้วย `ctx.Err()`)
5. เขียนฟังก์ชันที่รับ `context.Context` เป็น parameter ตัวแรก แล้วจำลองงานที่ใช้เวลานาน (`time.Sleep`) ลองเรียกด้วย `context.WithTimeout` ที่ตั้งเวลาสั้นกว่าและนานกว่าการทำงาน สังเกตความแตกต่างของผลลัพธ์และค่าที่ได้จาก `ctx.Err()`
6. ค้นคว้าเพิ่มเติม: อ่านเกี่ยวกับ `context.WithCancelCause` (เพิ่มเข้ามาใน Go 1.20+) ที่อนุญาตให้ระบุเหตุผลการยกเลิกแบบกำหนดเองได้ (ผ่าน `context.Cause(ctx)`) แล้วลองเขียนโปรแกรมทดสอบเทียบกับ `context.WithCancel` ธรรมดา

---

**ต่อไป**: [Part 033 — Testing พื้นฐานด้วย `testing`](./033-testing-basics.md)
