# Part 094: Circuit Breaker และ Retry Pattern

> ภาคที่ 8: Microservices, gRPC, Message Queue — ตอนที่ 7 จาก 7 (Part 88–94)

## สารบัญของบทนี้

1. ทบทวน: ทำไมการเรียก service อื่นถึง "ต้องพังบ้าง" เป็นเรื่องปกติ
2. ปัญหาของ naive infinite retry: Retry Storm และ Cascading Failure
3. Retry Pattern: Exponential Backoff และ Jitter (ทฤษฎี)
4. Hand-rolled Retry: เขียนเองและรันทดสอบจริงด้วย `httptest`
5. Library ทางเลือก: `github.com/avast/retry-go`
6. Circuit Breaker Pattern: แนวคิด Closed / Open / Half-Open
7. ใช้งานจริงด้วย `github.com/sony/gobreaker`: เปิด breaker และทดสอบการฟื้นตัว
8. รวมทุกอย่างเข้าด้วยกัน: Retry + Circuit Breaker + Timeout ในตัวเดียว
9. เลือกใช้ retry อย่างเดียว หรือต้องมี circuit breaker ด้วย
10. ทางเลือกระดับ Infrastructure: Service Mesh (Istio, Linkerd)
11. สรุปสิ่งที่ได้เรียนในบทนี้
12. แบบฝึกหัดท้ายบท

---

## 1. ทบทวน: ทำไมการเรียก service อื่นถึง "ต้องพังบ้าง" เป็นเรื่องปกติ

ตลอดภาคที่ 8 นี้ เราเรียนวิธีให้ service คุยกันหลายรูปแบบ: **gRPC** สำหรับการเรียกแบบ synchronous ที่เร็วและมี type ชัดเจน (Part 089-090), **RabbitMQ** และ **Kafka** สำหรับการสื่อสารแบบ asynchronous ผ่าน message broker (Part 091-092), และ **Service Discovery/API Gateway** สำหรับหา service ปลายทางในสภาพแวดล้อมที่ instance เปลี่ยนที่อยู่ตลอดเวลา (Part 093)

แต่ไม่ว่าจะเลือกวิธีสื่อสารแบบไหน มีความจริงข้อหนึ่งที่ **หลีกเลี่ยงไม่ได้** ใน distributed system: **การเรียกข้าม network จะล้มเหลวเป็นครั้งคราวเสมอ** ไม่ใช่เพราะโค้ดมี bug แต่เพราะธรรมชาติของระบบแบบกระจาย:

- **Network เป็นสิ่งที่ไม่น่าเชื่อถือ (unreliable) โดยธรรมชาติ** — packet หายได้, latency พุ่งสูงชั่วขณะได้, connection ขาดกลางทางได้ (นี่คือข้อที่ 1 ใน "Fallacies of Distributed Computing" ที่ทุกคนที่เขียนระบบกระจายควรรู้จัก)
- **Service ปลายทางอาจกำลัง deploy ใหม่** (rolling update) ทำให้ instance บางตัวไม่ตอบสนองชั่วขณะ
- **Service ปลายทางอาจโหลดสูงชั่วคราว** (เช่น GC pause ยาวผิดปกติ, มี traffic พุ่งจากที่อื่น) ทำให้ตอบช้าหรือปฏิเสธ request ชั่วขณะ
- **Resource มีจำกัดเสมอ** — database connection pool เต็ม (Part 078), thread pool ของ service ปลายทางเต็ม ฯลฯ

จุดสำคัญคือ: ความล้มเหลวเหล่านี้ส่วนใหญ่เป็น **transient failure** (ล้มเหลวชั่วคราว) ไม่ใช่ **permanent failure** (ล้มเหลวถาวร เช่น request ผิดรูปแบบ, ไม่มีสิทธิ์เข้าถึง) — ถ้าเรารู้จักแยกสองอย่างนี้ออกจากกัน เราจะออกแบบระบบที่ **"ลองใหม่อย่างฉลาด"** แทนที่จะยอมแพ้ทันทีหรือลองใหม่แบบไม่มีสติ ซึ่งเป็นหัวใจของทั้งบทนี้

---

## 2. ปัญหาของ naive infinite retry: Retry Storm และ Cascading Failure

สัญชาตญาณแรกของโปรแกรมเมอร์ส่วนใหญ่เมื่อเจอ error จากการเรียก service อื่นคือ "ลองใหม่สิ" ซึ่งถูกต้องในหลักการ แต่การ implement แบบไม่ระวังจะสร้างปัญหาใหม่ที่ร้ายแรงกว่าเดิม

### ตัวอย่างโค้ดที่ดูเหมือนถูกแต่อันตรายมาก

```go
// อย่าเขียนแบบนี้! (naive infinite retry ไม่มี delay)
func callDownstreamNaive(url string) (*http.Response, error) {
	for {
		resp, err := http.Get(url)
		if err == nil && resp.StatusCode == http.StatusOK {
			return resp, nil
		}
		// ลองใหม่ทันที ไม่มีการรอเลย!
	}
}
```

ถ้า service ปลายทางกำลังโหลดสูงอยู่แล้ว (นี่คือสาเหตุที่มันตอบ error ตั้งแต่แรก) การยิง request ใหม่ทันทีแบบไม่มีการหน่วงเวลาจะทำให้:

### Retry Storm (พายุการลองใหม่)

ถ้ามี client หลายพันตัวพร้อมกันเจอ error จาก service เดียวกัน แล้วทุกตัวลองใหม่ทันที — จำนวน request ที่ยิงเข้าไปยัง service ที่กำลังลำบากอยู่แล้วจะยิ่งเพิ่มขึ้นทวีคูณ แทนที่จะลดลง นี่คือสิ่งที่เรียกว่า **retry storm**: การ retry ที่ตั้งใจจะช่วยแก้ปัญหา กลับกลายเป็นตัวขยายปัญหาให้ใหญ่ขึ้นจนบางครั้งทำให้ service ที่แค่ "ช้าชั่วคราว" กลายเป็น "ล่มสนิท" เพราะรับ request รัวๆ ไม่ไหว

```
Service ปลายทางโหลดสูงเล็กน้อย (ตอบ error 5%)
        │
        ▼
Client 1,000 ตัว retry ทันทีไม่มี delay
        │
        ▼
Request เพิ่มขึ้นทวีคูณ (แต่ละ client ยิงซ้ำหลายรอบต่อวินาที)
        │
        ▼
Service ปลายทางล่มสนิท (ตอบ error 100%)
```

### Cascading Failure (ความล้มเหลวลุกลาม)

ในระบบ microservices ที่ service A เรียก B, B เรียก C ต่อ — ถ้า C เริ่มช้าลง แล้ว B รอ C แบบไม่มี timeout พร้อมกับ retry ไม่จำกัดจำนวนครั้ง เธรด/goroutine ของ B ที่ใช้รอ C จะค้างสะสมขึ้นเรื่อยๆ จนกระทั่ง B เองก็ resource หมดและตอบสนอง A ช้าลงตาม แล้ว A ก็ retry ไป B ซ้ำเติมอีกชั้นหนึ่ง — ความล้มเหลวจาก C จุดเดียว **ลุกลาม (cascade)** ไปทำให้ทั้งระบบล่มตามกันเป็นโดมิโน่ ทั้งที่ปัญหาต้นตออาจเป็นแค่ C ช้าชั่วคราวเท่านั้น

### หลักการแก้ปัญหาที่บทนี้จะเรียนทั้งหมด

มีสามเครื่องมือหลักที่ใช้ร่วมกันเพื่อป้องกันทั้งสองปัญหานี้:

1. **Retry ที่ฉลาด** (หัวข้อ 3-5): จำกัดจำนวนครั้ง + หน่วงเวลาแบบเพิ่มขึ้นเรื่อยๆ (exponential backoff) + สุ่มเวลาหน่วง (jitter) เพื่อไม่ให้ client ทั้งหมด retry พร้อมกันเป๊ะ
2. **Circuit Breaker** (หัวข้อ 6-7): หยุดยิง request ไปยัง service ที่กำลังพังอยู่ชั่วคราว แทนที่จะปล่อยให้ retry ซ้ำเติมมันไปเรื่อยๆ
3. **Timeout ด้วย `context`** (ทบทวนจาก **Part 032**): ไม่รอ response ไม่จำกัดเวลา ป้องกัน resource ค้างสะสม

---

## 3. Retry Pattern: Exponential Backoff และ Jitter (ทฤษฎี)

### Exponential Backoff

แนวคิดคือ: แทนที่จะรอเวลาคงที่ระหว่างการลองใหม่แต่ละครั้ง ให้ **เวลารอเพิ่มขึ้นแบบทวีคูณ** ทุกครั้งที่ลองแล้วยังไม่สำเร็จ เช่น รอ 100ms, 200ms, 400ms, 800ms, ... เหตุผลคือถ้าล้มเหลวติดกันหลายครั้ง แปลว่าปัญหาน่าจะรุนแรงกว่าที่คิด การรอนานขึ้นเรื่อยๆ ให้เวลา service ปลายทางฟื้นตัวได้จริง แทนที่จะไปกวนมันถี่ๆ ด้วยอัตราเดิม

```
attempt 1: รอ 0ms    (ลองทันที)
attempt 2: รอ 100ms  (ล้มเหลวครั้งที่ 1)
attempt 3: รอ 200ms  (ล้มเหลวครั้งที่ 2)
attempt 4: รอ 400ms  (ล้มเหลวครั้งที่ 3)
attempt 5: รอ 800ms  (ล้มเหลวครั้งที่ 4)
```

### Jitter (การสุ่มเวลาหน่วง)

ปัญหาของ exponential backoff แบบล้วนๆ (ไม่มี jitter) คือ: ถ้า client หลายพันตัวเจอ error พร้อมกัน (เช่น service ปลายทาง restart ตอน 10:00:00 น. พอดี) ทุกตัวจะคำนวณเวลาหน่วงได้**ค่าเดียวกันเป๊ะ** แล้วทุกตัวก็จะ retry **พร้อมกันอีกครั้ง** ที่จังหวะเดิมเป๊ะ กลายเป็น retry storm แบบมีจังหวะ (thundering herd) แทนที่จะกระจายตัวออก

การแก้คือใส่ **jitter** — สุ่มค่าความคลาดเคลื่อนเล็กน้อย (หรือมาก) เข้าไปในเวลาหน่วง ทำให้ client แต่ละตัว retry ที่เวลาต่างกันเล็กน้อย กระจาย load ออกไปตามธรรมชาติ กลยุทธ์ jitter ที่นิยมที่สุดคือ **Full Jitter**: สุ่มเวลาหน่วงในช่วง `[0, backoff]` แบบเต็มช่วง (แนวคิดนี้มาจาก blog ของทีม AWS Architecture ที่เผยแพร่ไว้อย่างมีอิทธิพลต่อวงการ) บทนี้ใช้สูตรที่ใกล้เคียงคือสุ่มในช่วงครึ่งบนของ backoff เพื่อไม่ให้เวลารอสั้นเกินไปจนไม่ต่างจากไม่มี backoff เลย

### กฎสำคัญ: retry เฉพาะ error ที่ควร retry เท่านั้น

ไม่ใช่ error ทุกชนิดควร retry — ตัวอย่างเช่น HTTP 400 (Bad Request) หรือ 401 (Unauthorized) คือปัญหาถาวรที่ retry กี่ครั้งก็ไม่มีทางสำเร็จ (request มันผิดตั้งแต่ต้น) มีแต่จะเสียเวลาและสร้างภาระเพิ่มโดยเปล่าประโยชน์ ควร retry เฉพาะ error ที่มีโอกาสเป็น transient เท่านั้น เช่น:

| ควร retry | ไม่ควร retry |
|---|---|
| Network timeout, connection refused | HTTP 400 Bad Request |
| HTTP 502/503/504 (server ปลายทางมีปัญหาชั่วคราว) | HTTP 401/403 (auth ผิด) |
| HTTP 429 Too Many Requests (ควรเคารพ `Retry-After` header ด้วย) | HTTP 404 Not Found |
| Database connection error ชั่วคราว | Business logic error (เช่น "ยอดเงินไม่พอ") |

---

## 4. Hand-rolled Retry: เขียนเองและรันทดสอบจริงด้วย `httptest`

มาเขียน exponential backoff + jitter ด้วยมือทีละขั้นตอน แล้วทดสอบกับ HTTP server จำลองที่ **ล้มเหลว 2 ครั้งแรกแล้วสำเร็จในครั้งที่ 3** — ใช้ `net/http/httptest` (ทบทวนจาก **Part 081**) เพื่อให้ตัวอย่างนี้รันได้จริงทั้งหมดโดยไม่ต้องพึ่ง service ภายนอกใดๆ เลย

```go
package main

import (
	"context"
	"fmt"
	"math/rand"
	"time"
)

// callWithRetry ทำ retry แบบ exponential backoff + jitter ด้วยมือ (hand-rolled)
// maxAttempts: จำนวนครั้งสูงสุดที่จะลอง (นับรวมครั้งแรก)
// baseDelay: หน่วยเวลาฐานที่ใช้คำนวณ backoff
// do: ฟังก์ชันที่พยายามทำงานจริง คืน error ถ้าล้มเหลว
func callWithRetry(ctx context.Context, maxAttempts int, baseDelay time.Duration, do func() error) error {
	var lastErr error
	for attempt := 0; attempt < maxAttempts; attempt++ {
		if attempt > 0 {
			// exponential backoff: 1x, 2x, 4x, 8x, ... ของ baseDelay
			backoff := baseDelay * time.Duration(1<<uint(attempt-1))
			// full jitter แบบง่าย: สุ่มในช่วง [backoff/2, backoff]
			jitter := time.Duration(rand.Int63n(int64(backoff) / 2))
			wait := backoff/2 + jitter

			select {
			case <-time.After(wait):
			case <-ctx.Done():
				return ctx.Err() // เคารพ cancellation/timeout จาก context เสมอ (Part 032)
			}
		}

		lastErr = do()
		if lastErr == nil {
			return nil // สำเร็จ ไม่ต้องลองต่อ
		}
		fmt.Printf("  attempt %d failed: %v\n", attempt+1, lastErr)
	}
	return fmt.Errorf("all %d attempts failed, last error: %w", maxAttempts, lastErr)
}
```

จุดที่ควรสังเกต:

- **`ctx context.Context` เป็นพารามิเตอร์แรกเสมอ** (ทบทวนจาก Part 032) — ทำให้ผู้เรียกควบคุมได้ว่าจะยกเลิกการ retry ทั้งหมดเมื่อไหร่ (เช่น ตั้ง `context.WithTimeout` ครอบทั้งการ retry ทั้งชุดไว้อีกชั้นหนึ่ง)
- **`select` ระหว่างรอ backoff กับ `ctx.Done()`** — สำคัญมาก ถ้าใช้ `time.Sleep(wait)` ตรงๆ แทน จะรอจนครบเวลาเสมอแม้ context จะถูกยกเลิกไปแล้วก็ตาม (ทบทวน pattern นี้จาก Part 038 เรื่อง `select`)
- **`do func() error`** — รับฟังก์ชันที่จะพยายามทำงานจริงเป็น parameter ทำให้ `callWithRetry` เป็น **higher-order function** ที่ใช้ซ้ำได้กับงานแบบไหนก็ได้ ไม่ผูกติดกับ HTTP โดยเฉพาะ (จะเรียก database, gRPC, หรืออะไรก็ได้ที่คืน error)

### ทดสอบกับ server จำลองที่ล้มเหลว 2 ครั้งแล้วสำเร็จ

```go
package main

import (
	"context"
	"fmt"
	"io"
	"net/http"
	"net/http/httptest"
	"sync/atomic"
	"testing"
	"time"
)

func TestCallWithRetry_HandRolled(t *testing.T) {
	var calls int32
	// httptest.Server จำลอง service ปลายทางที่ fail 2 ครั้งแรกแล้วสำเร็จในครั้งที่ 3
	srv := httptest.NewServer(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		n := atomic.AddInt32(&calls, 1)
		if n <= 2 {
			w.WriteHeader(http.StatusServiceUnavailable)
			return
		}
		w.WriteHeader(http.StatusOK)
		w.Write([]byte("ok"))
	}))
	defer srv.Close()

	start := time.Now()
	err := callWithRetry(context.Background(), 5, 50*time.Millisecond, func() error {
		resp, err := http.Get(srv.URL)
		if err != nil {
			return err
		}
		defer resp.Body.Close()
		body, _ := io.ReadAll(resp.Body)
		if resp.StatusCode != http.StatusOK {
			return fmt.Errorf("status %d", resp.StatusCode)
		}
		fmt.Println("  response body:", string(body))
		return nil
	})
	elapsed := time.Since(start)

	if err != nil {
		t.Fatalf("expected eventual success, got error: %v", err)
	}
	if calls != 3 {
		t.Fatalf("expected exactly 3 calls, got %d", calls)
	}
	fmt.Printf("succeeded after %d calls, total elapsed=%v\n", calls, elapsed)
}
```

> **หมายเหตุความโปร่งใส**: โค้ดทุกชิ้นในบทนี้ (retry, retry-go, gobreaker, resilient client) **ถูกรันจริง**ด้วย `go test` ในสภาพแวดล้อมที่เขียนบทนี้ ไม่ต้องพึ่ง service ภายนอกใดๆ เลย เพราะใช้ `httptest.Server` จำลองพฤติกรรม flaky/ล่มของ downstream ทั้งหมด ผลลัพธ์ที่แสดงคือผลลัพธ์จากการรันจริง

### ผลลัพธ์จากการรันจริง (`go test -run TestCallWithRetry_HandRolled -v`)

```
=== RUN   TestCallWithRetry_HandRolled
  attempt 1 failed: status 503
  attempt 2 failed: status 503
  response body: ok
succeeded after 3 calls, total elapsed=105.043282ms
--- PASS: TestCallWithRetry_HandRolled (0.11s)
PASS
```

สังเกตว่า `calls` เท่ากับ 3 เป๊ะ (ไม่มากไม่น้อยกว่านั้น) และเวลารวมประมาณ 105ms ซึ่งตรงกับผลรวมของ backoff สองรอบ (รอก่อน attempt 2 และก่อน attempt 3) บวกเวลาที่ HTTP round-trip ใช้จริง

### เมื่อ retry ครบทุกครั้งแล้วยังไม่สำเร็จ

```go
func TestCallWithRetry_AllFail(t *testing.T) {
	var calls int32
	srv := httptest.NewServer(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		atomic.AddInt32(&calls, 1)
		w.WriteHeader(http.StatusServiceUnavailable) // ล้มเหลวตลอดกาล ไม่มีทางสำเร็จ
	}))
	defer srv.Close()

	err := callWithRetry(context.Background(), 3, 20*time.Millisecond, func() error {
		resp, err := http.Get(srv.URL)
		if err != nil {
			return err
		}
		defer resp.Body.Close()
		if resp.StatusCode != http.StatusOK {
			return fmt.Errorf("status %d", resp.StatusCode)
		}
		return nil
	})

	if err == nil {
		t.Fatal("expected error after exhausting all attempts")
	}
	if calls != 3 {
		t.Fatalf("expected exactly 3 calls (maxAttempts), got %d", calls)
	}
	fmt.Println("got expected final error:", err)
}
```

ผลลัพธ์จากการรันจริง:

```
=== RUN   TestCallWithRetry_AllFail
  attempt 1 failed: status 503
  attempt 2 failed: status 503
  attempt 3 failed: status 503
got expected final error: all 3 attempts failed, last error: status 503
--- PASS: TestCallWithRetry_AllFail (0.05s)
```

สังเกตว่าฟังก์ชันหยุดที่ 3 ครั้งพอดี (`maxAttempts=3`) ไม่ลองไม่จำกัดจำนวนครั้งแบบที่หัวข้อ 2 เตือนไว้ — นี่คือหัวใจสำคัญที่แยก retry ที่ฉลาดออกจาก naive infinite retry

---

## 5. Library ทางเลือก: `github.com/avast/retry-go`

การเขียน retry เองอย่างในหัวข้อ 4 ช่วยให้เข้าใจกลไกอย่างถ่องแท้ แต่ในงานจริง โปรดักชันส่วนใหญ่มักใช้ library สำเร็จรูปเพื่อประหยัดเวลาและลดโอกาสเขียนผิดพลาด (เช่น ลืมใส่ jitter, ลืมเช็ค context cancellation) หนึ่งใน library ที่ได้รับความนิยมสูงในวงการ Go คือ `github.com/avast/retry-go`

ติดตั้งด้วย (บทนี้ใช้เวอร์ชัน v4 ที่เป็น major version ปัจจุบัน):

```bash
go get github.com/avast/retry-go/v4@latest
```

```go
package main

import (
	"fmt"
	"net/http"
	"net/http/httptest"
	"sync/atomic"
	"testing"
	"time"

	retry "github.com/avast/retry-go/v4"
)

func TestRetryGoLibrary(t *testing.T) {
	var calls int32
	srv := httptest.NewServer(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		n := atomic.AddInt32(&calls, 1)
		if n <= 2 {
			w.WriteHeader(http.StatusServiceUnavailable)
			return
		}
		w.WriteHeader(http.StatusOK)
	}))
	defer srv.Close()

	err := retry.Do(
		func() error {
			resp, err := http.Get(srv.URL)
			if err != nil {
				return err
			}
			defer resp.Body.Close()
			if resp.StatusCode != http.StatusOK {
				return fmt.Errorf("status %d", resp.StatusCode)
			}
			return nil
		},
		retry.Attempts(5),
		retry.Delay(30*time.Millisecond),
		retry.DelayType(retry.BackOffDelay), // exponential backoff สำเร็จรูป (มี jitter ในตัวถ้าใช้ CombineDelay)
		retry.OnRetry(func(n uint, err error) {
			fmt.Printf("  retry-go attempt %d failed: %v\n", n+1, err)
		}),
	)
	if err != nil {
		t.Fatalf("expected eventual success, got: %v", err)
	}
	fmt.Printf("retry-go succeeded after %d calls\n", calls)
}
```

### ผลลัพธ์จากการรันจริง

```
=== RUN   TestRetryGoLibrary
  retry-go attempt 1 failed: status 503
  retry-go attempt 2 failed: status 503
retry-go succeeded after 3 calls
--- PASS: TestRetryGoLibrary (0.09s)
```

### ทำไมบางทีก็ยังเลือกเขียนเอง

`retry-go` สะดวกมากสำหรับ retry แบบทั่วไป แต่ข้อดีของ hand-rolled version ในหัวข้อ 4 คือ **ควบคุมรายละเอียดได้ทุกจุด** เช่น jitter algorithm เฉพาะทาง, การ log เชิงลึกเฉพาะโดเมน, หรือการรวมเข้ากับ metrics/observability (Part 099) แบบเจาะจง กฎทั่วไปคือ: **เริ่มจาก library อย่าง `retry-go` ก่อนเสมอสำหรับ 90% ของงาน** เพราะทดสอบมาดีแล้วและลดโอกาสเขียนผิด สลับไปเขียนเองเฉพาะเมื่อมีความต้องการเฉพาะทางที่ library สำเร็จรูปทำไม่ได้จริงๆ — นี่คือหลักการเดียวกับที่เราใช้เลือกระหว่าง `kafka-go` (pure Go) กับ `confluent-kafka-go` (cgo) ใน Part 092

---

## 6. Circuit Breaker Pattern: แนวคิด Closed / Open / Half-Open

Retry แก้ปัญหา "ล้มเหลวชั่วครู่แล้วหาย" ได้ดี แต่ถ้า service ปลายทาง**ล่มยาว** (ไม่ใช่แค่ชั่วคราว) การ retry ต่อไปเรื่อยๆ (แม้จะมี backoff) ก็ยังคง**เพิ่มภาระ**ให้ service ที่กำลังพยายามฟื้นตัวอยู่ ยิ่งถ้ามี client จำนวนมากทำแบบเดียวกันพร้อมกัน — นี่คือจุดที่ **Circuit Breaker** เข้ามาเสริม

แนวคิดยืมมาจาก **เบรกเกอร์ไฟฟ้า** ในบ้าน: เมื่อกระแสไฟเกินกำหนด (short circuit) เบรกเกอร์จะ**ตัดวงจรทันที** ป้องกันไม่ให้ไฟไหม้บ้านทั้งหลัง แทนที่จะปล่อยให้กระแสไฟไหลต่อไปจนเกิดความเสียหายรุนแรงกว่าเดิม Circuit Breaker ในซอฟต์แวร์ทำหน้าที่เดียวกัน: เมื่อการเรียก service ปลายทางล้มเหลวถี่เกินเกณฑ์ที่กำหนด ให้ **"ตัดวงจร"** หยุดยิง request ไปยังปลายทางนั้นชั่วคราวทันที โดยไม่ต้องรอให้ request หมดเวลาไปเปล่าๆ ซ้ำแล้วซ้ำเล่า

### สามสถานะของ Circuit Breaker

```
        ล้มเหลวเกินเกณฑ์
   ┌───────────────────────┐
   │                       ▼
┌──────┐              ┌──────┐
│CLOSED│              │ OPEN │
└──────┘              └──────┘
   ▲                       │
   │                       │ ครบเวลา timeout
   │   สำเร็จ (trial)      ▼
   │                  ┌──────────┐
   └──────────────────│HALF-OPEN │
        ล้มเหลว (trial)└──────────┘
        กลับไป OPEN
```

| สถานะ | พฤติกรรม |
|---|---|
| **Closed** (ปกติ) | ปล่อยให้ request ผ่านไปยัง service ปลายทางตามปกติทุกครั้ง พร้อมนับจำนวนความสำเร็จ/ล้มเหลว |
| **Open** (ตัดวงจร) | **บล็อก request ทันทีโดยไม่ยิงไปที่ปลายทางเลย** คืน error ทันที (เร็วมาก ไม่ต้องรอ timeout) เข้าสู่สถานะนี้เมื่อจำนวนความล้มเหลวเกินเกณฑ์ที่ตั้งไว้ในสถานะ Closed |
| **Half-Open** (ทดสอบ) | หลังจากอยู่ในสถานะ Open ครบเวลาที่กำหนด (`Timeout`) breaker จะยอมให้ request **จำนวนจำกัด** ผ่านไปทดสอบ ถ้าสำเร็จ → กลับไป Closed (ปลายทางฟื้นแล้ว) ถ้าล้มเหลวอีก → กลับไป Open ต่อ (ยังไม่ฟื้น รอใหม่) |

จุดสำคัญที่ทำให้ circuit breaker ต่างจาก retry: **retry พยายามทำให้ request ที่ค้างอยู่สำเร็จ** ในขณะที่ **circuit breaker ป้องกันไม่ให้มี request ใหม่ถูกส่งไปเลยตั้งแต่แรก** เมื่อรู้อยู่แล้วว่าน่าจะล้มเหลว — สองอย่างนี้เสริมกันได้อย่างลงตัว (จะเห็นในหัวข้อ 8)

---

## 7. ใช้งานจริงด้วย `github.com/sony/gobreaker`: เปิด breaker และทดสอบการฟื้นตัว

`github.com/sony/gobreaker` เป็น library circuit breaker ที่ได้รับความนิยมสูงมากในวงการ Go เขียนโดยวิศวกรของ Sony ออกแบบตาม pattern ที่ Michael Nygard อธิบายไว้ในหนังสือ *Release It!* (หนังสือคลาสสิกด้าน production resilience) ติดตั้งด้วย:

```bash
go get github.com/sony/gobreaker@latest
```

### API หลัก

```go
cb := gobreaker.NewCircuitBreaker(gobreaker.Settings{
	Name:        "downstream-service",
	MaxRequests: 1,                      // จำนวน request ที่ยอมให้ผ่านตอน half-open
	Timeout:     200 * time.Millisecond, // เวลาที่อยู่ในสถานะ open ก่อนขยับไป half-open
	ReadyToTrip: func(counts gobreaker.Counts) bool {
		return counts.ConsecutiveFailures >= 3 // เงื่อนไขที่ทำให้ trip (เปิดวงจร)
	},
	OnStateChange: func(name string, from, to gobreaker.State) {
		fmt.Printf("%s: %s -> %s\n", name, from, to)
	},
})

result, err := cb.Execute(func() (interface{}, error) {
	// โค้ดที่จะเรียก service ปลายทางจริง
	return callDownstream()
})
```

- **`Execute`** คือจุดเดียวที่ต้องเรียกทุกครั้งที่จะติดต่อ service ปลายทาง — มันจะเช็คสถานะปัจจุบันก่อน (ถ้า Open จะคืน `gobreaker.ErrOpenState` ทันทีโดยไม่เรียกฟังก์ชันข้างในเลย), เรียกฟังก์ชันจริงถ้าสถานะอนุญาต, แล้วอัปเดต counts ตามผลลัพธ์
- **`ReadyToTrip`** คือ callback ที่ตัดสินใจว่าเมื่อไหร่ควร trip จาก Closed ไป Open — ตัวอย่างนี้ใช้เกณฑ์ "ล้มเหลวติดกัน 3 ครั้ง" แต่ปรับเป็นเกณฑ์อื่นได้ เช่น "อัตราความล้มเหลวเกิน 50% จากอย่างน้อย 10 request"
- **`MaxRequests`** ควบคุมว่าตอน Half-Open จะยอมให้กี่ request ผ่านไปทดสอบพร้อมกัน — ตั้งค่าต่ำ (เช่น 1) เพื่อไม่ยิง request จำนวนมากไปทดสอบปลายทางที่เพิ่งจะฟื้น ซึ่งอาจทำให้มันล่มซ้ำอีก

### ตัวอย่างที่รันจริง: ทำให้ breaker เปิด แล้วทดสอบการฟื้นตัวใน half-open

```go
package main

import (
	"fmt"
	"net/http"
	"net/http/httptest"
	"sync/atomic"
	"testing"
	"time"

	"github.com/sony/gobreaker"
)

func TestGobreakerOpenAndRecover(t *testing.T) {
	var failing int32 = 1 // ตอนเริ่มต้น server จะตอบ error เสมอ
	srv := httptest.NewServer(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		if atomic.LoadInt32(&failing) == 1 {
			w.WriteHeader(http.StatusInternalServerError)
			return
		}
		w.WriteHeader(http.StatusOK)
	}))
	defer srv.Close()

	cb := gobreaker.NewCircuitBreaker(gobreaker.Settings{
		Name:        "downstream-service",
		MaxRequests: 1,
		Timeout:     200 * time.Millisecond,
		ReadyToTrip: func(counts gobreaker.Counts) bool {
			return counts.ConsecutiveFailures >= 3
		},
		OnStateChange: func(name string, from, to gobreaker.State) {
			fmt.Printf("  [breaker] %s: %s -> %s\n", name, from, to)
		},
	})

	callDownstream := func() (interface{}, error) {
		resp, err := http.Get(srv.URL)
		if err != nil {
			return nil, err
		}
		defer resp.Body.Close()
		if resp.StatusCode != http.StatusOK {
			return nil, fmt.Errorf("downstream returned %d", resp.StatusCode)
		}
		return "ok", nil
	}

	// เฟส 1: downstream ล้มเหลวต่อเนื่อง -> ยิง 5 ครั้ง คาดว่าจะ trip หลังครั้งที่ 3
	fmt.Println("phase 1: downstream is failing")
	for i := 1; i <= 5; i++ {
		_, err := cb.Execute(callDownstream)
		fmt.Printf("  call %d: err=%v, state=%s\n", i, err, cb.State())
	}
	if cb.State() != gobreaker.StateOpen {
		t.Fatalf("expected breaker to be open after repeated failures, got %s", cb.State())
	}

	// breaker เปิดอยู่ -> เรียกอีกครั้งต้องโดนบล็อกทันที ไม่ยิง HTTP จริงไปที่ server เลย
	_, err := cb.Execute(callDownstream)
	if err != gobreaker.ErrOpenState {
		t.Fatalf("expected ErrOpenState while breaker is open, got %v", err)
	}
	fmt.Println("phase 2: breaker is open, call short-circuited without hitting downstream:", err)

	// เฟส 3: downstream ฟื้นตัวแล้ว รอให้ breaker เข้าสถานะ half-open แล้วลองใหม่
	atomic.StoreInt32(&failing, 0)
	fmt.Println("phase 3: downstream recovered, waiting for breaker Timeout to elapse")
	time.Sleep(250 * time.Millisecond) // เกิน Timeout (200ms) -> breaker เข้าสู่ half-open ในการเรียกถัดไป

	result, err := cb.Execute(callDownstream)
	if err != nil {
		t.Fatalf("expected success in half-open trial, got %v", err)
	}
	fmt.Printf("  half-open trial call: result=%v, state after=%s\n", result, cb.State())

	if cb.State() != gobreaker.StateClosed {
		t.Fatalf("expected breaker to close after successful half-open trial, got %s", cb.State())
	}
	fmt.Println("phase 4: breaker closed again, downstream calls flow normally")
}
```

### ผลลัพธ์จากการรันจริง

```
=== RUN   TestGobreakerOpenAndRecover
phase 1: downstream is failing
  call 1: err=downstream returned 500, state=closed
  call 2: err=downstream returned 500, state=closed
  [breaker] downstream-service: closed -> open
  call 3: err=downstream returned 500, state=open
  call 4: err=circuit breaker is open, state=open
  call 5: err=circuit breaker is open, state=open
phase 2: breaker is open, call short-circuited without hitting downstream: circuit breaker is open
phase 3: downstream recovered, waiting for breaker Timeout to elapse
  [breaker] downstream-service: open -> half-open
  [breaker] downstream-service: half-open -> closed
  half-open trial call: result=ok, state after=closed
phase 4: breaker closed again, downstream calls flow normally
--- PASS: TestGobreakerOpenAndRecover (0.25s)
```

เจาะดูผลลัพธ์ทีละจุด:

- **call 1, 2**: ล้มเหลวแต่ breaker ยังอยู่ใน `closed` (ยังไม่ครบ 3 ครั้งติดกัน)
- **หลัง call 2 เสร็จ**: `ConsecutiveFailures` แตะ 3 พอดี (เพราะ call ที่ 3 กำลังจะเกิด) — สังเกต log `closed -> open` ปรากฏขึ้น**ก่อน** call 3 จะเสร็จผล เพราะ `Execute` ของ call 3 เองก็ยังเป็นตัวที่ทำให้ยอด failure ครบ 3 (breaker ตัดสินใจ trip ทันทีหลังนับผลของ call 3 เสร็จ)
- **call 4, 5**: breaker อยู่ใน `open` แล้ว จึงคืน error `circuit breaker is open` ทันที **โดยไม่มีการยิง HTTP request ไปที่ server เลย** (ประหยัด resource ทั้งฝั่ง client และไม่ไปรบกวน server ที่กำลังลำบากอยู่เพิ่ม)
- **phase 3**: หลังรอเกิน `Timeout` (200ms) breaker เปลี่ยนเป็น `half-open` โดยอัตโนมัติในการเรียกครั้งถัดไป ยอมให้ 1 request (`MaxRequests: 1`) ผ่านไปทดสอบจริง
- **phase 4**: เพราะ downstream ฟื้นแล้วจริง (เราสั่ง `failing = 0` ไปก่อนหน้า) การทดสอบสำเร็จ breaker จึงกลับไป `closed` พร้อม reset ตัวนับทั้งหมด

นี่คือพฤติกรรมทั้งหมดของ circuit breaker ที่พิสูจน์ได้จริงในโค้ดเดียว: **ป้องกันการยิงซ้ำเมื่อรู้อยู่แล้วว่าจะพัง (open) และทดสอบการฟื้นตัวอย่างระมัดระวัง (half-open) ก่อนจะกลับมาทำงานเต็มรูปแบบ (closed)**

---

## 8. รวมทุกอย่างเข้าด้วยกัน: Retry + Circuit Breaker + Timeout ในตัวเดียว

ในโปรดักชันจริง เราแทบไม่เคยใช้แค่อย่างใดอย่างหนึ่งเดี่ยวๆ — resilient HTTP client ที่ดีควรมีทั้งสามชั้นทำงานร่วมกัน:

```
┌─────────────────────────────────────────────┐
│           Circuit Breaker (ชั้นนอกสุด)         │  ← ถ้า open, บล็อกทันทีไม่ต้องลองเลย
│  ┌─────────────────────────────────────────┐ │
│  │        Retry Loop (exponential backoff)  │ │  ← ถ้า closed/half-open ค่อยลองใหม่หลายครั้ง
│  │  ┌─────────────────────────────────────┐ │ │
│  │  │   Timeout ต่อ 1 attempt (context)    │ │ │  ← แต่ละครั้งที่ลอง ไม่รอเกินเวลาที่กำหนด
│  │  │         HTTP call จริง               │ │ │
│  │  └─────────────────────────────────────┘ │ │
│  └─────────────────────────────────────────┘ │
└─────────────────────────────────────────────┘
```

ลำดับที่ถูกต้องคือ **circuit breaker ครอบ retry loop ทั้งชุดไว้อีกที** ไม่ใช่ retry ครอบ circuit breaker — เหตุผลคือถ้าทำกลับกัน (breaker อยู่ข้างในของแต่ละ attempt) breaker จะนับความล้มเหลวเป็นระดับ "attempt ย่อย" แทนที่จะนับเป็นระดับ "การเรียกทั้งชุด" ทำให้ trip ไวเกินจำเป็นหรือช้าเกินไปอย่างไม่ตรงเจตนา ตัวอย่างข้างล่างจึงให้ retry loop อยู่**ข้างใน** ฟังก์ชันที่ `cb.Execute` เรียก

```go
package main

import (
	"context"
	"fmt"
	"io"
	"math/rand"
	"net/http"
	"time"

	"github.com/sony/gobreaker"
)

// ResilientClient รวม timeout (context) + retry (exponential backoff+jitter) + circuit breaker
// เข้าด้วยกันเป็น http client wrapper ตัวเดียวที่ใช้ซ้ำได้ทุกจุดที่เรียก service อื่น
type ResilientClient struct {
	http      *http.Client
	breaker   *gobreaker.CircuitBreaker
	retries   int
	baseDelay time.Duration
}

func NewResilientClient(name string) *ResilientClient {
	return &ResilientClient{
		http: &http.Client{},
		breaker: gobreaker.NewCircuitBreaker(gobreaker.Settings{
			Name:        name,
			MaxRequests: 1,
			Timeout:     500 * time.Millisecond,
			ReadyToTrip: func(c gobreaker.Counts) bool {
				return c.ConsecutiveFailures >= 3
			},
			OnStateChange: func(name string, from, to gobreaker.State) {
				fmt.Printf("  [breaker] %s: %s -> %s\n", name, from, to)
			},
		}),
		retries:   3,
		baseDelay: 20 * time.Millisecond,
	}
}

// Get เรียก GET ไปยัง url โดยทั้งชุด (retry loop) ถูกครอบด้วย circuit breaker อีกชั้นหนึ่ง
// แต่ละ attempt ภายในมี timeout ของตัวเอง ผูกกับ ctx ต้นทางที่ผู้เรียกส่งมา
func (c *ResilientClient) Get(ctx context.Context, url string) ([]byte, error) {
	result, err := c.breaker.Execute(func() (interface{}, error) {
		var lastErr error
		for attempt := 0; attempt < c.retries; attempt++ {
			if attempt > 0 {
				backoff := c.baseDelay * time.Duration(1<<uint(attempt-1))
				jitter := time.Duration(rand.Int63n(int64(backoff) + 1))
				select {
				case <-time.After(backoff/2 + jitter/2):
				case <-ctx.Done():
					return nil, ctx.Err()
				}
			}

			// timeout ต่อ attempt เดียว (ทบทวน context.WithTimeout จาก Part 032)
			reqCtx, cancel := context.WithTimeout(ctx, 100*time.Millisecond)
			req, _ := http.NewRequestWithContext(reqCtx, http.MethodGet, url, nil)
			resp, err := c.http.Do(req)
			cancel()
			if err != nil {
				lastErr = err
				continue
			}
			body, _ := io.ReadAll(resp.Body)
			resp.Body.Close()
			if resp.StatusCode >= 500 {
				lastErr = fmt.Errorf("server error: %d", resp.StatusCode)
				continue
			}
			return body, nil
		}
		return nil, fmt.Errorf("all %d attempts failed: %w", c.retries, lastErr)
	})
	if err != nil {
		return nil, err
	}
	return result.([]byte), nil
}
```

### ทดสอบเคสที่ 1: ล้มเหลว 2 ครั้งแล้วสำเร็จ (retry ช่วยได้ทัน breaker ไม่ทันได้ trip)

```go
func TestResilientClient_RetriesThenSucceeds(t *testing.T) {
	var calls int32
	srv := httptest.NewServer(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		n := atomic.AddInt32(&calls, 1)
		if n <= 2 {
			w.WriteHeader(http.StatusServiceUnavailable)
			return
		}
		w.Write([]byte("pong"))
	}))
	defer srv.Close()

	client := NewResilientClient("pong-service")
	body, err := client.Get(context.Background(), srv.URL)
	if err != nil {
		t.Fatalf("expected success, got %v", err)
	}
	fmt.Println("got body:", string(body), "after", calls, "calls")
}
```

ผลลัพธ์จากการรันจริง:

```
=== RUN   TestResilientClient_RetriesThenSucceeds
got body: pong after 3 calls
--- PASS: TestResilientClient_RetriesThenSucceeds (0.05s)
```

### ทดสอบเคสที่ 2: downstream พังถาวร — breaker ต้อง trip หลังเรียกซ้ำหลายครั้ง

```go
func TestResilientClient_BreakerTripsAfterPersistentFailure(t *testing.T) {
	srv := httptest.NewServer(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		w.WriteHeader(http.StatusInternalServerError) // ล้มเหลวตลอด
	}))
	defer srv.Close()

	client := NewResilientClient("always-down-service")

	// เรียกซ้ำหลายครั้ง (แต่ละครั้งมี retry 3 attempts ข้างในของตัวเอง แต่ breaker
	// นับความล้มเหลวเป็นระดับ "การเรียก Execute หนึ่งชุด" ไม่ใช่ระดับ attempt ย่อยข้างใน)
	var lastErr error
	for i := 1; i <= 3; i++ {
		_, lastErr = client.Get(context.Background(), srv.URL)
		fmt.Printf("  call %d: err=%v, breaker state=%s\n", i, lastErr, client.breaker.State())
	}
	if client.breaker.State() != gobreaker.StateOpen {
		t.Fatalf("expected breaker open after persistent failures, got %s", client.breaker.State())
	}

	// เรียกครั้งถัดไปควรถูกบล็อกทันทีโดย breaker ไม่ยิง HTTP ไปที่ server เลย
	_, err := client.Get(context.Background(), srv.URL)
	if err != gobreaker.ErrOpenState {
		t.Fatalf("expected ErrOpenState, got %v", err)
	}
	fmt.Println("call after trip short-circuited immediately:", err)
}
```

ผลลัพธ์จากการรันจริง:

```
=== RUN   TestResilientClient_BreakerTripsAfterPersistentFailure
  call 1: err=all 3 attempts failed: server error: 500, breaker state=closed
  call 2: err=all 3 attempts failed: server error: 500, breaker state=closed
  [breaker] always-down-service: closed -> open
  call 3: err=all 3 attempts failed: server error: 500, breaker state=open
call after trip short-circuited immediately: circuit breaker is open
--- PASS: TestResilientClient_BreakerTripsAfterPersistentFailure (0.12s)
```

สังเกตว่า: **แต่ละครั้งที่เรียก `client.Get` มี retry 3 attempts ซ่อนอยู่ข้างในของตัวเอง** (จึงเห็นข้อความ `all 3 attempts failed`) แต่ breaker เองนับ**การเรียก `Get` ทั้งชุด**เป็น 1 หน่วยความล้มเหลว ไม่ได้นับทีละ attempt ย่อย — พอเรียก `Get` ล้มเหลวครบ 3 ชุดติดกัน (ไม่ใช่ 3 attempts ย่อย) breaker ก็ trip เป็น `open` ทันที การเรียกครั้งที่ 4 จึงถูกบล็อกทันทีโดยไม่มีการยิง HTTP หรือ retry ใดๆ เกิดขึ้นเลย — นี่คือพฤติกรรมที่ถูกต้องของการวาง breaker ไว้ **ครอบ** retry loop ตามที่อธิบายไว้ก่อนโค้ด

---

## 9. เลือกใช้ retry อย่างเดียว หรือต้องมี circuit breaker ด้วย

| สถานการณ์ | แนวทางที่เหมาะสม |
|---|---|
| เรียก service ที่ปกติเสถียรมาก นานๆ ครั้งถึงจะล้มเหลวแบบสุ่ม (network hiccup) | Retry + backoff เพียงพอ ไม่จำเป็นต้องมี breaker |
| เรียก service ที่อาจล่มยาว (deploy ผิดพลาด, database ล่ม) หรือมี traffic จำนวนมากเรียกซ้ำกัน | ต้องมี Circuit Breaker เพื่อป้องกันไม่ให้ retry ของทุก client ซ้ำเติมปัญหา |
| Service ปลายทางมี rate limit ชัดเจน (เช่น external API ที่คิดเงินต่อ request) | Retry ต้องเคารพ `Retry-After` header เป็นหลัก บวก breaker เพื่อป้องกันการเรียกเกิน quota โดยไม่จำเป็น |
| เรียก service ภายในองค์กรเดียวกันจำนวนมากจุด (microservices เยอะ) | ควรมีทั้งคู่ในทุกจุดที่เรียกข้าม service — และมักย้ายไปทำระดับ infrastructure แทนโค้ดแอป (หัวข้อ 10) |

หลักคิดสั้นๆ: **retry ป้องกัน false negative จากความล้มเหลวชั่วครู่ ส่วน circuit breaker ป้องกันไม่ให้ retry กลายเป็นตัวปัญหาเสียเอง** สองอย่างนี้ควรมองว่าเป็น**คู่หูที่ทำงานร่วมกัน**เสมอในระบบที่มีการเรียกข้าม service จำนวนมาก ไม่ใช่ตัวเลือกที่ต้องเลือกอย่างใดอย่างหนึ่ง

---

## 10. ทางเลือกระดับ Infrastructure: Service Mesh (Istio, Linkerd)

ตัวอย่างในบทนี้ implement resilience pattern ไว้**ในโค้ดแอปพลิเคชันโดยตรง** (application-level) ซึ่งมีข้อดีคือควบคุมได้ละเอียด แต่มีข้อเสียชัดเจนเมื่อระบบมี microservices จำนวนมาก (ย้อนกลับไปดู Part 088 เรื่องความซับซ้อนที่มาพร้อมกับ microservices): **ทุก service ต้อง implement retry/circuit breaker เองซ้ำๆ** ในทุกภาษาที่ทีมต่างๆ ใช้เขียน ทำให้พฤติกรรมไม่สม่ำเสมอกันข้ามทีม และแก้ config (เช่น เปลี่ยนค่า timeout) ต้อง build/deploy โค้ดใหม่ทุกครั้ง

**Service Mesh** คือแนวทางแก้ปัญหานี้ในระดับ infrastructure: แทรก **proxy** (เรียกว่า **sidecar**) เข้าไปคู่กับทุก instance ของทุก service โดยที่ทุก network call ระหว่าง service วิ่งผ่าน sidecar proxy นี้เสมอ (ไม่ใช่วิ่งตรงระหว่างแอปกับแอป) แล้ว retry, circuit breaker, timeout, load balancing, mTLS, observability ทั้งหมด**ถูกจัดการโดย proxy** ไม่ใช่โค้ดแอป

```
ไม่มี Service Mesh:                    มี Service Mesh:

Service A ──────► Service B         Service A ──► [sidecar] ──► [sidecar] ──► Service B
(retry/breaker เขียนในโค้ด A เอง)      (retry/breaker/mTLS จัดการโดย sidecar
                                        ไม่ต้องเขียนโค้ดพิเศษใน A หรือ B เลย)
```

สอง implementation ที่ได้รับความนิยมสูงสุดในระบบนิเวศ Kubernetes:

- **Istio**: Service Mesh ที่ครบเครื่องที่สุด ใช้ Envoy proxy เป็น sidecar เดิมพัฒนาโดย Google/IBM/Lyft feature ครบมากรวมถึง traffic splitting สำหรับ canary deployment แต่แลกมาด้วยความซับซ้อนในการติดตั้ง/ดูแลค่อนข้างสูง
- **Linkerd**: เน้นความเรียบง่ายและเบา (lightweight) เขียน proxy เองด้วย Rust แทนที่จะใช้ Envoy ติดตั้งง่ายกว่า Istio มาก เหมาะกับทีมที่ต้องการ resilience pattern พื้นฐานโดยไม่ต้องแบกความซับซ้อนเต็มรูปแบบ

ข้อควรพิจารณาก่อนเลือกใช้ Service Mesh: มันแก้ปัญหา "retry/breaker กระจัดกระจายไม่สม่ำเสมอข้ามทีม" ได้ดีมาก แต่ก็เพิ่ม **operational complexity** อีกชั้นหนึ่งให้ทีม infrastructure ต้องดูแล (sidecar เพิ่ม latency เล็กน้อยทุก hop, ต้องเรียนรู้ config เฉพาะของ mesh) ทีมขนาดเล็กที่มี microservices ไม่กี่ตัวมักเริ่มจาก resilience pattern แบบ application-level อย่างที่เรียนในบทนี้ก่อน (ผ่าน library เช่น `gobreaker`) แล้วค่อยพิจารณา service mesh เมื่อจำนวน service เยอะขึ้นจนความไม่สม่ำเสมอข้ามทีมกลายเป็นปัญหาจริง — เรื่องการ deploy และ orchestrate ระบบเหล่านี้จะเริ่มเรียนใน **ภาคที่ 9** ที่กำลังจะเริ่มถัดจากนี้ (Docker ใน Part 095, Kubernetes ใน Part 097)

---

## สรุปสิ่งที่ได้เรียนในบทนี้

- การเรียกข้าม network ล้มเหลวเป็นครั้งคราวเป็นเรื่องปกติของ distributed system (transient failure) ต่างจาก permanent failure ที่ retry กี่ครั้งก็ไม่มีทางสำเร็จ
- **Naive infinite retry** (ไม่มี backoff, ไม่จำกัดจำนวนครั้ง) สร้างปัญหาใหม่ที่ร้ายแรงกว่าเดิม: **Retry Storm** (ยิ่ง retry ยิ่งเพิ่มโหลดให้ปลายทางที่กำลังลำบาก) และ **Cascading Failure** (ความล้มเหลวลุกลามเป็นโดมิโน่ทั้งระบบ)
- **Exponential Backoff** เพิ่มเวลารอแบบทวีคูณทุกครั้งที่ล้มเหลว ส่วน **Jitter** สุ่มเวลารอเพื่อไม่ให้ client จำนวนมาก retry พร้อมกันเป๊ะ (thundering herd)
- เขียน retry เองได้ไม่ยาก (hand-rolled) และควรเคารพ `context` เสมอเพื่อรองรับ cancellation — หรือใช้ library สำเร็จรูปอย่าง `github.com/avast/retry-go` เพื่อความสะดวกและลดโอกาสเขียนผิด
- **Circuit Breaker** มี 3 สถานะ: **Closed** (ปกติ), **Open** (ตัดวงจร บล็อก request ทันทีไม่ยิงไปปลายทางเลย), **Half-Open** (ทดสอบด้วย request จำนวนจำกัดว่าปลายทางฟื้นหรือยัง)
- `github.com/sony/gobreaker` implement pattern นี้ให้ครบผ่าน `CircuitBreaker.Execute` — ทดสอบจริงแล้วเห็นว่า breaker เปิดหลังล้มเหลวติดกันครบเกณฑ์ และกลับมาปิดได้เองหลัง half-open trial สำเร็จ
- Resilient HTTP client ในโปรดักชันควรมี **Circuit Breaker ครอบ Retry Loop ครอบ Timeout ต่อ attempt** ทั้งสามชั้นทำงานร่วมกัน ไม่ใช่เลือกใช้แค่อย่างเดียว
- **Service Mesh** (Istio, Linkerd) ย้าย resilience pattern เหล่านี้ไปทำที่ระดับ infrastructure ผ่าน sidecar proxy แทนที่จะเขียนซ้ำในโค้ดทุก service — เหมาะกับระบบที่มี microservices จำนวนมากและหลายทีม/หลายภาษา
- โค้ดทุกชิ้นในบทนี้ (retry แบบ hand-rolled, retry-go, gobreaker, resilient client ที่รวมทุกอย่าง) **รันจริงด้วย `go test` และ `httptest.Server` จำลอง downstream ที่ล้มเหลว** ไม่ต้องพึ่ง infrastructure ภายนอกเลย

นี่คือบทสุดท้ายของ **ภาคที่ 8: Microservices, gRPC, Message Queue** เราเดินทางจากหลักการออกแบบ microservices (Part 088), การสื่อสารแบบ synchronous ด้วย gRPC (Part 089-090), การสื่อสารแบบ asynchronous ผ่าน message broker ทั้ง RabbitMQ และ Kafka (Part 091-092), การค้นหา service และ routing ผ่าน API Gateway (Part 093), จนมาถึงการทำให้ระบบทนทานต่อความล้มเหลวด้วย retry และ circuit breaker ในบทนี้ — ทุกชิ้นส่วนเหล่านี้คือองค์ประกอบพื้นฐานของระบบ microservices ระดับโปรดักชันจริง

ขั้นตอนถัดไปที่ธรรมชาติที่สุดคือ: เมื่อเขียนโค้ดที่ทนทานแล้ว จะ **build, package, และ deploy** มันไปรันจริงอย่างไร นี่คือสิ่งที่ **ภาคที่ 9: DevOps และ Deployment** จะพาไปเรียนต่อ เริ่มจากการห่อหุ้มแอป Go ด้วย **Docker** (Part 095)

---

## แบบฝึกหัดท้ายบท

1. ดัดแปลง `callWithRetry` ในหัวข้อ 4 ให้รับ parameter เพิ่มเป็น `shouldRetry func(error) bool` เพื่อแยกว่า error แบบไหนควร retry แบบไหนไม่ควร (ตามตารางในหัวข้อ 3) แล้วทดสอบด้วย `httptest.Server` ที่ตอบ HTTP 400 (ไม่ควร retry) เทียบกับ 503 (ควร retry)
2. ทดลองปรับ `ReadyToTrip` ในตัวอย่างหัวข้อ 7 จาก "ล้มเหลวติดกัน 3 ครั้ง" เป็น "อัตราความล้มเหลวเกิน 60% จากอย่างน้อย 5 request" (ใช้ field `counts.Requests`, `counts.TotalFailures`) แล้วรันทดสอบดูว่า breaker trip เร็วขึ้นหรือช้าลงเมื่อเทียบกับเกณฑ์เดิม
3. เพิ่ม metric/counter ง่ายๆ เข้าไปใน `ResilientClient` จากหัวข้อ 8 เพื่อนับจำนวนครั้งที่ breaker บล็อก request ทันที (`ErrOpenState`) แยกจากจำนวนครั้งที่ retry ทั้งชุดล้มเหลว แล้วพิมพ์สรุปหลังรันเทส
4. ลองรวม `retry-go` (หัวข้อ 5) เข้ากับ `gobreaker` (หัวข้อ 7) แทนที่จะเขียน retry loop เอง — เปรียบเทียบความยาวและความอ่านง่ายของโค้ดกับเวอร์ชัน hand-rolled ในหัวข้อ 8
5. เขียนสถานการณ์ทดสอบใหม่ที่ downstream "ยังไม่ฟื้นจริง" ตอน half-open trial (คือทดสอบแล้วยังล้มเหลวอยู่) แล้วตรวจสอบว่า breaker กลับไปที่สถานะ `open` อีกครั้งแทนที่จะเป็น `closed` — อธิบายว่าทำไม pattern นี้ถึงปลอดภัยกว่าการปล่อยให้ request ทั้งหมดผ่านไปทดสอบพร้อมกัน
6. (ขั้นสูง) ค้นคว้าเพิ่มเติมเกี่ยวกับ Istio's `DestinationRule` และ `VirtualService` ที่ใช้ตั้งค่า retry/circuit breaker ระดับ infrastructure แล้วเปรียบเทียบว่า config เหล่านั้นสอดคล้องกับแนวคิด `ReadyToTrip`, `MaxRequests`, `Timeout` ที่เรียนในบทนี้อย่างไรบ้าง

---

**ต่อไป**: [Part 095 — Docker กับ Go Application](./095-docker.md)
