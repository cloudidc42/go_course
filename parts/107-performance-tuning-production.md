# Part 107: Performance Tuning ระดับ Production

> ภาคที่ 10: มืออาชีพและระดับโลก (Professional & World-Class) — ตอนที่ 8 จาก 11 (Part 100–110)

> **หมายเหตุเรื่องความซื่อสัตย์ในการสาธิต**: ทุกตัวเลขในบทนี้ (throughput, latency, ผลลัพธ์ `pprof`, ผลลัพธ์ `benchstat`, พฤติกรรม `GOMAXPROCS` ใน container) **รันจริง** บนเครื่องที่ใช้เขียนหลักสูตรนี้ (4 CPU core, Go 1.24.7, `hey` สำหรับ load testing, Docker 29.3.1 สำหรับสาธิตเรื่อง container) ไม่ใช่ค่าที่แต่งขึ้น ตัวเลข throughput ที่แสดงเป็นของเครื่องนี้โดยเฉพาะ — เครื่องของผู้อ่านจะได้ตัวเลขต่างไปตามจำนวน core และ hardware แต่ **สัดส่วนการเปลี่ยนแปลง** (เช่น เร็วขึ้นกี่เท่าหลังแก้บั๊ก) ควรใกล้เคียงกัน เพราะมาจากธรรมชาติของโค้ดเอง ไม่ใช่จากฮาร์ดแวร์

## สารบัญของบทนี้

1. "Measure, don't guess": หลักการเดียวที่คุมทั้งบทนี้
2. เครื่องมือที่เรียนมาแล้วทั้งหมด กับบทบาทของแต่ละตัวใน production
3. Load Testing ด้วย `hey`: วัด Baseline ก่อนแตะโค้ดแม้แต่บรรทัดเดียว
4. หา Bottleneck ด้วย `pprof` บน Server ที่กำลังรับโหลดจริง
5. อ่านผลลัพธ์ `pprof`: เจอ Hotspot ตัวจริง
6. แก้บั๊ก แล้ววัดผลด้วย `go test -bench` และ `benchstat`
7. วัดผลกระทบจริงอีกครั้งด้วย Load Test: ก่อน vs หลัง
8. `GOMAXPROCS` ในโลกของ Container: กับดักที่ทีมส่วนใหญ่ไม่รู้ตัว
9. ปรับแต่ง `GOGC`/`GOMEMLIMIT` สำหรับ Production (ทบทวน Part 055, 085)
10. Connection Pool Tuning ทบทวนจาก Part 078 ในบริบทของ Production
11. Monitoring คือ Performance Tuning แบบต่อเนื่อง (เชื่อม Part 099)
12. Checklist: Production Performance
13. สรุปสิ่งที่ได้เรียนในบทนี้
14. แบบฝึกหัดท้ายบท

---

## 1. "Measure, don't guess": หลักการเดียวที่คุมทั้งบทนี้

**Part 083** (pprof), **Part 084** (benchmarking), **Part 085** (memory management) และ **Part 099** (monitoring) ต่างสอนเครื่องมือคนละชิ้น แต่ทั้งหมดยืนอยู่บนหลักการเดียวกันที่ **Part 085** ปิดท้ายไว้แล้วว่า **"Profile ก่อนเสมอ — อย่าเดา-optimize"** บทนี้ขยายหลักการนี้จากระดับ "ฟังก์ชันเดียว" ไปสู่ระดับ **"ระบบทั้งระบบใน production"**

เหตุผลที่หลักการนี้สำคัญถึงขนาดต้องย้ำเป็นบทเดี่ยว: สัญชาตญาณของนักพัฒนาเกี่ยวกับ "โค้ดตรงไหนช้า" **ผิดบ่อยกว่าที่คิดมาก** แม้แต่นักพัฒนาที่มีประสบการณ์สูง เพราะ CPU cache, garbage collector, network I/O, และ scheduler ของ Go runtime ล้วนมีพฤติกรรมที่ขัดกับสามัญสำนึกในหลายสถานการณ์ บทนี้จะพาไล่ตาม **workflow การสืบสวนปัญหา performance แบบมีระบบ** ทีละขั้นตอน โดยใช้ตัวอย่างจริงที่รันจริงทุกจุด:

```
วัด Baseline → หา Bottleneck ด้วยข้อมูลจริง → แก้เฉพาะจุดที่ข้อมูลชี้ → วัดผลซ้ำ → ทำซ้ำจนพอ
```

Workflow นี้ไม่มีขั้นตอนไหน "เดา" เลยแม้แต่ขั้นเดียว — นี่คือสิ่งที่แยกวิศวกรที่ tuning ระบบ production ได้จริง ออกจากคนที่แค่ "ลองเปลี่ยนอะไรสักอย่างแล้วดูว่าดีขึ้นไหม"

---

## 2. เครื่องมือที่เรียนมาแล้วทั้งหมด กับบทบาทของแต่ละตัวใน Production

ก่อนลงมือ มาจัดระเบียบว่าเครื่องมือแต่ละตัวที่เรียนมาตอบคำถามอะไร เพื่อเลือกใช้ให้ถูกจังหวะ:

| เครื่องมือ | ตอบคำถามอะไร | ใช้ตอนไหน |
|---|---|---|
| `hey` / load testing | "ระบบรับโหลดได้เท่าไร ก่อนที่ latency จะแย่ลง?" | ก่อนเริ่ม tuning (baseline) และหลัง fix (เทียบผล) |
| `pprof` (**Part 083**) | "เวลา/หน่วยความจำส่วนใหญ่ถูกใช้ไปกับฟังก์ชันไหน?" | เมื่อรู้แล้วว่าช้า แต่ไม่รู้ว่าช้าตรงไหน |
| `go test -bench` + `benchstat` (**Part 084**) | "การแก้ไขนี้เร็วขึ้นจริงหรือแค่รู้สึกว่าเร็วขึ้น?" | หลังแก้โค้ด ก่อน merge |
| `testing.AllocsPerRun`, memory strategies (**Part 085**) | "ทำไม GC ถึงทำงานถี่ ใครสร้าง allocation เยอะ?" | เมื่อ pprof ชี้ว่าปัญหาอยู่ที่ memory ไม่ใช่ CPU |
| Prometheus/Grafana (**Part 099**) | "ระบบกำลังมีปัญหาจริงหรือไม่ ตอนนี้เลย ใน production?" | ตลอดเวลา แบบต่อเนื่อง ไม่ใช่แค่ตอน investigate |

สังเกตว่าเครื่องมือเหล่านี้ **ไม่แข่งกัน แต่ต่อกันเป็น pipeline เดียว**: monitoring บอกว่า "มีปัญหา", load testing สร้างสถานการณ์ที่ทำซ้ำปัญหาได้ในสภาพแวดล้อมควบคุม, pprof ชี้ตำแหน่งที่แท้จริง, benchmark+benchstat ยืนยันว่าวิธีแก้ได้ผลจริง — บทนี้จะเดินตาม pipeline นี้ทั้งหมดกับตัวอย่างเดียวกัน

---

## 3. Load Testing ด้วย `hey`: วัด Baseline ก่อนแตะโค้ดแม้แต่บรรทัดเดียว

สมมติสถานการณ์ที่พบบ่อยที่สุดใน production จริง: มี HTTP endpoint หนึ่งตัวที่ทีมสงสัยว่า "ช้ากว่าที่ควรจะเป็น" ตัวอย่างนี้คือ endpoint `/validate` ที่รับ email แล้วตรวจสอบรูปแบบด้วย regular expression:

```go
package main

import (
	"encoding/json"
	"fmt"
	"log"
	"net/http"
	_ "net/http/pprof" // เปิด pprof endpoint ไว้สำหรับ investigate (หัวข้อ 4)
	"regexp"
)

func validateEmail(email string) bool {
	re := regexp.MustCompile(`^[a-zA-Z0-9._%+\-]+@[a-zA-Z0-9.\-]+\.[a-zA-Z]{2,}$`)
	return re.MatchString(email)
}

type response struct {
	Email string `json:"email"`
	Valid bool   `json:"valid"`
}

func handler(w http.ResponseWriter, r *http.Request) {
	email := r.URL.Query().Get("email")
	if email == "" {
		email = "user@example.com"
	}
	valid := validateEmail(email)
	w.Header().Set("Content-Type", "application/json")
	json.NewEncoder(w).Encode(response{Email: email, Valid: valid})
}

func main() {
	http.HandleFunc("/validate", handler)
	log.Fatal(http.ListenAndServe(":8080", nil))
}
```

ก่อนสงสัยว่าโค้ดตรงไหนมีปัญหา ขั้นแรกที่ต้องทำเสมอคือ **วัด baseline** — ตัวเลขที่เป็นข้อเท็จจริง ไม่ใช่ความรู้สึก เครื่องมือที่ใช้คือ **`hey`** ([github.com/rakyll/hey](https://github.com/rakyll/hey)) load generator ที่เขียนด้วย Go เอง ใช้งานง่าย เหมาะกับการทดสอบเร็วๆ ในเครื่อง dev ก่อนส่งต่อไปทดสอบระดับใหญ่ขึ้นด้วยเครื่องมืออย่าง `vegeta`, `k6`, หรือ `Locust`

ติดตั้ง:

```bash
go install github.com/rakyll/hey@latest
```

รันโหลดจริง 8 วินาที ด้วย 200 concurrent connection:

```bash
hey -z 8s -c 200 "http://localhost:8080/validate?email=alice@example.com"
```

ผลลัพธ์จริงจากเครื่องที่เขียนบทนี้ (4 CPU core):

```
Summary:
  Total:	8.0070 secs
  Slowest:	0.0496 secs
  Fastest:	0.0000 secs
  Average:	0.0048 secs
  Requests/sec:	41687.7367

  Total data:	14353099 bytes
  Size/request:	43 bytes

Response time histogram:
  0.000 [1]	|
  0.005 [210641]	|■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■
  0.010 [82654]	|■■■■■■■■■■■■■■■■
  0.015 [27213]	|■■■■■
  0.020 [9282]	|■■
  0.025 [2763]	|■
  0.030 [830]	|
  0.035 [305]	|
  0.040 [40]	|
  0.045 [60]	|
  0.050 [4]	|

Latency distribution:
  10% in 0.0005 secs
  25% in 0.0013 secs
  50% in 0.0035 secs
  75% in 0.0068 secs
  90% in 0.0108 secs
  95% in 0.0138 secs
  99% in 0.0206 secs

Status code distribution:
  [200]	333793 responses
```

นี่คือ **baseline**: **41,687 requests/sec**, p50 latency **3.5ms**, p99 latency **20.6ms** จำตัวเลขนี้ไว้ให้แม่น — ทุกการเปลี่ยนแปลงโค้ดหลังจากนี้ต้องเทียบกับตัวเลขชุดนี้เสมอ ไม่ใช่เทียบกับความรู้สึก

### อ่านผลลัพธ์ `hey` ให้ถูกจุด

- **Requests/sec** คือ throughput รวม — ตัวเลขเดียวที่หลายคนดูก่อน แต่ **ไม่ควรดูตัวเดียว**
- **Latency distribution (percentile)** สำคัญกว่า average เสมอในระบบ production เพราะ **average ถูกลากด้วยค่าผิดปกติได้ง่าย** ค่าที่ทีม SRE ดูจริงคือ **p95/p99** — ในตัวอย่างนี้ p50 อยู่ที่ 3.5ms แต่ p99 กระโดดไปถึง 20.6ms (เกือบ 6 เท่า) นี่คือสัญญาณว่ามี "หาง" (tail latency) ที่ผู้ใช้บางส่วนเจอประสบการณ์แย่กว่าค่าเฉลี่ยมาก แม้ระบบ "ดูเหมือนโอเค" จากตัวเลข average ก็ตาม

---

## 4. หา Bottleneck ด้วย `pprof` บน Server ที่กำลังรับโหลดจริง

ตอนนี้รู้แล้วว่า throughput อยู่ที่เท่าไร แต่ยังไม่รู้ว่า **อะไรกินเวลาไป** — นี่คือจุดที่ `pprof` (**Part 083**) เข้ามาแทนที่การเดา เพราะ handler มีการ import `_ "net/http/pprof"` ไว้แล้ว จึงเก็บ CPU profile ได้โดยตรงจาก endpoint `/debug/pprof/profile` **ขณะที่ server กำลังรับโหลดจริงจาก `hey` พร้อมกัน**:

```bash
# terminal 1: ยิงโหลดต่อเนื่อง 8 วินาที
hey -z 8s -c 200 "http://localhost:8080/validate?email=alice@example.com" > /dev/null &

# terminal 2 (เริ่มเกือบพร้อมกัน): เก็บ CPU profile 6 วินาทีระหว่างที่มีโหลด
curl -s "http://localhost:8080/debug/pprof/profile?seconds=6" -o cpu.pprof
```

เทคนิคสำคัญตรงนี้คือ **เก็บ profile ระหว่างที่ระบบมีโหลดจริง** ไม่ใช่ตอนว่าง เพราะปัญหา performance หลายอย่างปรากฏเฉพาะภายใต้โหลดสูง (เช่น lock contention, GC pressure) ที่ไม่เห็นเลยตอนทดสอบแบบ request เดียว

---

## 5. อ่านผลลัพธ์ `pprof`: เจอ Hotspot ตัวจริง

เปิดผลลัพธ์ด้วย `go tool pprof` แบบ cumulative (`-cum`) เพื่อดูว่าเวลาส่วนใหญ่ไหลไปที่ฟังก์ชันไหนตามลำดับการเรียก:

```bash
go tool pprof -top -cum -nodecount=15 cpu.pprof
```

ผลลัพธ์จริง (กรองเฉพาะบรรทัดที่เกี่ยวกับโค้ดของเราและ `regexp`):

```
      flat  flat%   sum%        cum   cum%
         0     0%  0.61%      2.18s 26.62%  main.handler
         0     0% 22.71%      1.73s 21.12%  main.validateEmail
         0     0% 22.95%      1.53s 18.68%  regexp.Compile (inline)
         0     0% 22.95%      1.53s 18.68%  regexp.MustCompile
         0     0% 22.95%      1.53s 18.68%  regexp.compile
         0     0% 24.66%      0.63s  7.69%  regexp/syntax.Parse (inline)
     0.01s  0.12% 24.79%      0.63s  7.69%  regexp/syntax.parse
     0.01s  0.12% 25.15%      0.47s  5.74%  regexp.compileOnePass
```

นี่คือหลักฐานที่ชัดเจนที่สุดเท่าที่จะหาได้: **ภายในเวลา CPU ทั้งหมดของ profile นี้ 18.68% ถูกใช้ไปกับ `regexp.Compile` เพียงอย่างเดียว** และเมื่อดูภายใน `main.validateEmail` (21.12% ของเวลาทั้งหมด) จะเห็นว่า **เกือบทั้งหมด (18.68% จาก 21.12%) คือการ compile regex ซ้ำๆ** ไม่ใช่การ match จริง

อ่านโค้ดต้นเหตุอีกครั้ง:

```go
func validateEmail(email string) bool {
	re := regexp.MustCompile(`^[a-zA-Z0-9._%+\-]+@[a-zA-Z0-9.\-]+\.[a-zA-Z]{2,}$`)
	return re.MatchString(email)
}
```

พบบั๊กประสิทธิภาพคลาสสิกที่พบบ่อยมากในโค้ด Go จริง: **`regexp.MustCompile` ถูกเรียกใหม่ทุกครั้งที่ฟังก์ชันทำงาน** แทนที่จะ compile เพียงครั้งเดียวแล้วใช้ซ้ำ การ compile regex เป็นงานหนัก (ต้อง parse ไวยากรณ์, สร้าง state machine) ที่ตั้งใจให้ทำ**ครั้งเดียว**แล้ว `*regexp.Regexp` ที่ได้นำไปใช้ `MatchString` ซ้ำได้อย่างปลอดภัยจากหลาย goroutine พร้อมกัน (`*regexp.Regexp` เป็น thread-safe สำหรับการอ่าน)

---

## 6. แก้บั๊ก แล้ววัดผลด้วย `go test -bench` และ `benchstat`

ก่อนเอาไปรันบน server จริงอีกรอบ ให้วัดผลกระทบระดับฟังก์ชันล้วนๆ ด้วย benchmark ก่อน (ทบทวน **Part 084**) เพราะแยกผลกระทบของ "โค้ดที่แก้" ออกจากปัจจัยอื่น (network, OS scheduler) ได้ชัดเจนกว่า:

```go
package bench

import (
	"regexp"
	"testing"
)

const testEmail = "alice@example.com"

// เวอร์ชันเดิม (มีบั๊ก): compile regex ใหม่ทุกครั้งที่เรียก
func validateSlow(email string) bool {
	re := regexp.MustCompile(`^[a-zA-Z0-9._%+\-]+@[a-zA-Z0-9.\-]+\.[a-zA-Z]{2,}$`)
	return re.MatchString(email)
}

// เวอร์ชันแก้แล้ว: compile ครั้งเดียวระดับ package แล้วใช้ซ้ำ
var fastRe = regexp.MustCompile(`^[a-zA-Z0-9._%+\-]+@[a-zA-Z0-9.\-]+\.[a-zA-Z]{2,}$`)

func validateFast(email string) bool {
	return fastRe.MatchString(email)
}

func BenchmarkValidateSlow(b *testing.B) {
	b.ReportAllocs()
	for i := 0; i < b.N; i++ {
		validateSlow(testEmail)
	}
}

func BenchmarkValidateFast(b *testing.B) {
	b.ReportAllocs()
	for i := 0; i < b.N; i++ {
		validateFast(testEmail)
	}
}
```

รันด้วย `-count=10` เพื่อให้ `benchstat` มีข้อมูลพอวิเคราะห์ทางสถิติ (ทบทวนหลักการนี้จาก **Part 084**):

```bash
go test -bench=. -benchmem -run=^$ -count=10 . > bench_out.txt
benchstat bench_out.txt
```

ผลลัพธ์จริง:

```
goos: linux
goarch: amd64
pkg: perfdemo/bench
cpu: Intel(R) Xeon(R) Processor @ 2.10GHz
               │ bench_out.txt │
               │    sec/op     │
ValidateSlow-4     6.114µ ± 7%
ValidateFast-4     361.1n ± 5%
geomean            1.486µ

               │ bench_out.txt  │
               │      B/op      │
ValidateSlow-4   5.714Ki ± 0%
ValidateFast-4     0.000 ± 0%

               │ bench_out.txt │
               │   allocs/op   │
ValidateSlow-4    66.00 ± 0%
ValidateFast-4    0.000 ± 0%
```

ตัวเลขนี้ชัดเจนเกินกว่าจะตีความผิด:

- **เร็วขึ้นประมาณ 17 เท่า** (6.114µs → 361.1ns ต่อครั้ง)
- **ลด allocation จาก 66 ครั้งเหลือ 0 ครั้งต่อการเรียก** — เชื่อมโยงตรงกับ **Part 085** ที่สอนว่า allocation คือต้นตอของแรงกดดันต่อ GC เมื่อ allocation ต่อ request ลดลงเหลือศูนย์ หมายความว่า path นี้จะไม่สร้างภาระให้ garbage collector เลยแม้จะถูกเรียกถี่แค่ไหนก็ตาม
- **ลด memory ต่อครั้งจาก 5.7 KiB เหลือ 0 ไบต์** — ทุก byte เหล่านั้นคือ heap ที่ regex compiler ต้อง allocate ใหม่ทุกครั้ง (AST, instruction table ของ state machine) แล้วทิ้งทันทีให้ GC เก็บกวาด

ตัวเลข **`± 7%`** และ **`± 5%`** คือ**ค่าความแปรปรวน** ที่ `benchstat` รายงานให้อัตโนมัติ — ค่าต่ำแบบนี้บอกว่าผลการวัดเสถียร ไม่ใช่ noise ทำให้มั่นใจได้ว่าความแตกต่างที่เห็นเป็นผลจากการแก้โค้ดจริง ไม่ใช่ความบังเอิญของการวัด

---

## 7. วัดผลกระทบจริงอีกครั้งด้วย Load Test: ก่อน vs หลัง

Benchmark ระดับฟังก์ชันยืนยันแล้วว่าโค้ดเร็วขึ้นจริง แต่คำถามที่ทีมต้องตอบคือ **"ผลกระทบนี้สะท้อนไปถึงผู้ใช้จริงมากแค่ไหน?"** — ต้องกลับไปที่ `hey` อีกครั้งกับ server เวอร์ชันที่แก้แล้ว (ย้าย `regexp.MustCompile` ออกมาเป็น package-level variable):

```go
var emailRe = regexp.MustCompile(`^[a-zA-Z0-9._%+\-]+@[a-zA-Z0-9.\-]+\.[a-zA-Z]{2,}$`)

func validateEmail(email string) bool {
	return emailRe.MatchString(email)
}
```

รันโหลดเดิมทุกประการ (concurrency 200, ระยะเวลา 8 วินาที) กับ server เวอร์ชันใหม่:

```bash
hey -z 8s -c 200 "http://localhost:8081/validate?email=alice@example.com"
```

ผลลัพธ์จริง:

```
Summary:
  Total:	8.0023 secs
  Slowest:	0.0835 secs
  Fastest:	0.0000 secs
  Average:	0.0037 secs
  Requests/sec:	54589.8590

Latency distribution:
  10% in 0.0005 secs
  25% in 0.0012 secs
  50% in 0.0026 secs
  75% in 0.0050 secs
  90% in 0.0080 secs
  95% in 0.0105 secs
  99% in 0.0162 secs

Status code distribution:
  [200]	436846 responses
```

สรุปผลก่อน-หลังแบบเทียบตรง:

| ตัวชี้วัด | ก่อนแก้ | หลังแก้ | เปลี่ยนแปลง |
|---|---|---|---|
| Requests/sec | 41,687 | 54,590 | **+31%** |
| p50 latency | 3.5ms | 2.6ms | **-26%** |
| p99 latency | 20.6ms | 16.2ms | **-21%** |
| ns/op (function-level) | 6,114ns | 361ns | **-94% (~17x)** |
| Allocations/op | 66 | 0 | **-100%** |

### ทำไมตัวเลขระดับ server (throughput +31%) ต่างจากระดับฟังก์ชัน (~17x)?

จุดนี้สำคัญมากและเป็นบทเรียนสำคัญของบทนี้: **การแก้ไขที่ทำให้ฟังก์ชันเร็วขึ้น 17 เท่า ไม่ได้แปลว่า server ทั้งตัวจะเร็วขึ้น 17 เท่าตาม** เพราะ `main.handler` มีงานอื่นที่กิน CPU ด้วยเช่นกัน (JSON encoding, HTTP header handling, network I/O ที่เห็นเป็น `runtime.syscall` สัดส่วนใหญ่ในผล `pprof` ตอนต้น) — `validateEmail` เป็นแค่**ส่วนหนึ่ง**ของงานทั้งหมดใน request แม้จะเป็นส่วนที่ใหญ่ที่สุดในตอนนั้น (21% ของ CPU time) แต่เมื่อกำจัดคอขวดนี้ไปแล้ว **คอขวดถัดไป** (ในกรณีนี้คือ network I/O ที่เห็นสัดส่วนสูงสุดใน `pprof` ตั้งแต่ต้น) จะกลายเป็นตัวจำกัด throughput แทน

นี่คือเหตุผลที่ workflow "measure → fix → measure" ต้อง**ทำซ้ำเป็นวงจร** ไม่ใช่ทำครั้งเดียวจบ — แก้คอขวดตัวหนึ่งแล้ว ให้กลับไป profile ใหม่เสมอเพื่อดูว่าคอขวดตัวถัดไปคืออะไร แทนที่จะสันนิษฐานว่า "แก้เสร็จแล้ว"

---

## 8. `GOMAXPROCS` ในโลกของ Container: กับดักที่ทีมส่วนใหญ่ไม่รู้ตัว

**Part 055** สอนไปแล้วว่า `runtime.GOMAXPROCS` ควบคุมจำนวน OS thread สูงสุดที่ Go runtime ใช้รัน goroutine พร้อมกัน โดย default จะเท่ากับ `runtime.NumCPU()` — ปัญหาที่ทีมจำนวนมากไม่รู้ตัวจนกว่าจะเจอปัญหาจริงคือ: **`runtime.NumCPU()` อ่านจำนวน CPU ของเครื่อง host ทั้งเครื่อง ไม่ใช่โควตา CPU ที่ container ถูกจำกัดไว้ผ่าน cgroup**

มาสาธิตปัญหานี้ให้เห็นจริงด้วยโปรแกรมเล็กๆ:

```go
package main

import (
	"fmt"
	"os"
	"runtime"
)

func main() {
	fmt.Println("runtime.NumCPU():     ", runtime.NumCPU())
	fmt.Println("runtime.GOMAXPROCS(0):", runtime.GOMAXPROCS(0))
	fmt.Println("GOMAXPROCS env var:   ", os.Getenv("GOMAXPROCS"))
}
```

Build เป็น static binary แล้วรันใน container ที่จำกัด CPU ไว้แค่ 1 core ด้วย Docker (ต่อยอด multi-stage build จาก **Part 095**):

```bash
CGO_ENABLED=0 go build -o cpucheck main.go
docker build -t cpucheck .   # Dockerfile: FROM scratch + COPY cpucheck
```

รันแบบไม่จำกัด CPU เทียบกับจำกัดด้วย `--cpus=1`:

```bash
docker run --rm cpucheck
docker run --rm --cpus=1 cpucheck
```

ผลลัพธ์จริง (เครื่อง host มี 4 core):

```
$ docker run --rm cpucheck
runtime.NumCPU():      4
runtime.GOMAXPROCS(0): 4
GOMAXPROCS env var:

$ docker run --rm --cpus=1 cpucheck
runtime.NumCPU():      4
runtime.GOMAXPROCS(0): 4
GOMAXPROCS env var:
```

**นี่คือกับดักตัวจริง**: แม้สั่ง `docker run --cpus=1` (จำกัด container ให้ใช้ CPU ได้ไม่เกิน 1 core ผ่าน cgroup) โปรแกรม Go ก็ยังรายงาน `GOMAXPROCS(0)` เป็น **4** เหมือนเดิม เพราะมันอ่านจำนวน core ของ host เครื่องจริง ไม่ได้อ่านโควตาของ cgroup ผลที่ตามมาคือ Go runtime จะพยายามสร้าง OS thread และ schedule goroutine ราวกับมี 4 core ให้ใช้งานจริง ทั้งที่ kernel จะ throttle การใช้ CPU ให้เหลือแค่ 1 core เทียบเท่า — เกิด **CPU throttling ที่มองไม่เห็นจากมุมมองของแอปพลิเคชัน**, context switch เกินความจำเป็น, และ latency ที่แปรปรวนสูงกว่าที่ควรเป็น (สังเกตได้จาก metric `container_cpu_cfs_throttled_periods_total` ใน Kubernetes ตาม **Part 099** ถ้าตัวเลขนี้สูงต่อเนื่อง นี่คือสัญญาณของปัญหานี้โดยตรง)

### ทางแก้: ตั้งค่า `GOMAXPROCS` ให้ตรงกับโควตาจริง

วิธีแก้ที่ตรงไปตรงมาที่สุดคือ**ตั้งค่า `GOMAXPROCS` ให้ตรงกับ CPU limit ที่ container ได้รับจริง** ผ่าน environment variable:

```bash
docker run --rm --cpus=1 -e GOMAXPROCS=1 cpucheck
```

ผลลัพธ์จริง:

```
runtime.NumCPU():      4
runtime.GOMAXPROCS(0): 1
GOMAXPROCS env var:    1
```

`GOMAXPROCS(0)` เปลี่ยนเป็น 1 ตามที่ตั้งใจแล้ว — ค่า `runtime.NumCPU()` ยังคงเป็น 4 เสมอ (เพราะมันรายงานจำนวน core ของ host ตามนิยามของมัน ไม่เปลี่ยนตามการตั้งค่า) แต่ `GOMAXPROCS` คือค่าที่ runtime ใช้จริงในการตัดสินใจ schedule งาน

### แนวทางที่แนะนำสำหรับ Production: `uber-go/automaxprocs`

การตั้งค่า `GOMAXPROCS` ด้วยมือใน environment variable ใช้ได้ แต่เปราะบาง — ถ้ามีคนเปลี่ยน CPU limit ใน Kubernetes manifest แล้วลืมอัปเดต `GOMAXPROCS` ให้ตรงกัน ปัญหาก็กลับมาอีก ชุมชน Go จึงมี library ที่ Uber เปิดเป็น open source ชื่อ **[`go.uber.org/automaxprocs`](https://github.com/uber-go/automaxprocs)** ซึ่งเป็นมาตรฐานที่ยอมรับกันกว้างขวางในอุตสาหกรรมสำหรับปัญหานี้โดยเฉพาะ — มันอ่านค่า CPU quota จาก cgroup โดยตรงตอน process เริ่มทำงาน แล้วเรียก `runtime.GOMAXPROCS(...)` ให้ตรงกับโควตาจริงโดยอัตโนมัติ ไม่ต้องพึ่ง environment variable ที่ตั้งด้วยมือ:

```go
import (
	"log"

	_ "go.uber.org/automaxprocs" // side-effect import: ตั้งค่า GOMAXPROCS อัตโนมัติตอน init()
)

func main() {
	// GOMAXPROCS ถูกตั้งให้ตรงกับ cgroup CPU quota แล้วตั้งแต่ก่อนบรรทัดนี้ทำงาน
	log.Println("starting server...")
	// ...
}
```

> **แนวทางปฏิบัติที่แนะนำ**: สำหรับ service ที่รันใน container พร้อม CPU limit (Kubernetes, Docker Compose ตาม **Part 096-097**) ให้ import `go.uber.org/automaxprocs` เป็นมาตรฐานเริ่มต้นของทุกโปรเจกต์ตั้งแต่วันแรก เช่นเดียวกับที่ตั้ง `CGO_ENABLED=0` เป็นมาตรฐานใน **Part 095** — เป็นการลงทุนบรรทัดเดียวที่ตัดปัญหา CPU throttling ที่มองไม่เห็นออกไปได้ทั้งหมด

---

## 9. ปรับแต่ง `GOGC`/`GOMEMLIMIT` สำหรับ Production (ทบทวน Part 055, 085)

**Part 055** อธิบายกลไกของ `GOGC` และ `GOMEMLIMIT` ไว้แล้วในเชิงหลักการ บทนี้เติมมุมมองของ**การตัดสินใจตั้งค่าจริงในสภาพแวดล้อม container ที่มี memory limit ตายตัว** (Kubernetes pod ที่ตั้ง `resources.limits.memory`)

### หลักการตั้งค่าที่แนะนำสำหรับ container

```bash
# ตัวอย่าง: Kubernetes pod ตั้ง memory limit ไว้ที่ 512Mi
# ตั้ง GOMEMLIMIT ให้ต่ำกว่า limit จริงเล็กน้อย เผื่อพื้นที่ให้ non-heap memory
# (goroutine stack, OS thread, C library ที่เชื่อมผ่าน cgo ถ้ามี)
GOMEMLIMIT=450MiB
```

เหตุผลที่ต้องเผื่อ margin (ในตัวอย่างนี้ประมาณ 12%): `GOMEMLIMIT` ควบคุมเฉพาะ **heap memory** ที่ Go runtime จัดการโดยตรง แต่โปรเซสยังใช้ memory ส่วนอื่นที่อยู่นอกเหนือการควบคุมนี้ (เช่น goroutine stack ในสถานการณ์ที่มี goroutine จำนวนมากพร้อมกัน) ถ้าตั้ง `GOMEMLIMIT` ชิด limit ของ container มากเกินไป โปรเซสอาจถูก **OOM-killed** โดย kernel ทั้งที่ heap ยังไม่ถึงเพดานที่ตั้งไว้เลยด้วยซ้ำ

### `GOGC` กับ `GOMEMLIMIT` ทำงานร่วมกันอย่างไรใน Production

แนวทางที่ทีมจำนวนมากใช้จริงและได้ผลดี:

- **ปล่อย `GOGC` ไว้ที่ค่า default (100) หรือปรับสูงขึ้นเล็กน้อย** (เช่น 150-200) เพื่อลดความถี่ของ GC cycle ในสถานการณ์ปกติที่ memory ยังไม่ตึง — ลด CPU overhead จาก GC
- **ตั้ง `GOMEMLIMIT` เป็น "ตาข่ายนิรภัย"** ที่คอยเร่ง GC ให้ทำงานถี่ขึ้นเฉพาะตอนที่ memory ใกล้ชนเพดานจริงๆ (ตามที่ **Part 055** อธิบายกลไกไว้)

ผลลัพธ์คือระบบใช้ CPU น้อยลงในสถานการณ์ปกติ (เพราะ `GOGC` สูงขึ้น) แต่ยังคงปลอดภัยจาก OOM ในสถานการณ์ที่ traffic พุ่งขึ้นกะทันหัน (เพราะ `GOMEMLIMIT` เข้ามาควบคุม)

### อย่าตั้งค่าเหล่านี้โดยไม่มีข้อมูลรองรับ

ย้ำหลักการจาก **Part 055**: `GOGC`/`GOMEMLIMIT` เป็นเครื่องมือสำหรับ**สถานการณ์ที่วัดผลได้จริงแล้วว่าจำเป็น** ก่อนปรับค่าเหล่านี้ใน production ต้องมีข้อมูลจาก **Part 099** (Prometheus metrics เช่น `go_memstats_gc_cpu_fraction`, `go_gc_duration_seconds`) ยืนยันก่อนว่า GC เป็นปัญหาจริงในระบบ ไม่ใช่การตั้งค่าตามความรู้สึกหรือเลียนแบบ config ของโปรเจกต์อื่นที่มี traffic pattern ต่างกันโดยสิ้นเชิง

---

## 10. Connection Pool Tuning ทบทวนจาก Part 078 ในบริบทของ Production

**Part 078** สอน `SetMaxOpenConns`, `SetMaxIdleConns`, `SetConnMaxLifetime`, `SetConnMaxIdleTime` ไว้อย่างละเอียดพร้อมสาธิต pool exhaustion จริง สิ่งที่บทนี้เพิ่มคือ**มุมมองเรื่องการตั้งค่าเหล่านี้ให้สัมพันธ์กับขนาดจริงของ production deployment**

### สูตรคิดเลขคร่าวๆ สำหรับ `SetMaxOpenConns`

กับดักที่พบบ่อยคือตั้ง `SetMaxOpenConns` สูงเกินไปโดยคิดว่า "ยิ่งเยอะยิ่งเร็ว" — ในความเป็นจริง **database มีขีดจำกัดจำนวน connection รวมที่รับได้** (เช่น PostgreSQL default `max_connections` คือ 100) ถ้า service มีหลาย instance (เช่น 10 pod บน Kubernetes) แต่ละ instance ตั้ง `SetMaxOpenConns(100)` เท่ากับพยายามเปิดได้สูงสุดถึง 1,000 connection พร้อมกัน ซึ่งเกินขีดจำกัดของ database ไปมาก และจะทำให้ connection ถูกปฏิเสธในช่วง traffic สูงพอดีที่สุด (จังหวะที่แย่ที่สุดเท่าที่จะเป็นไปได้)

หลักคิดที่ปลอดภัยกว่า:

```
SetMaxOpenConns ต่อ instance ≈ (max_connections ของ database × margin ปลอดภัย ~80%) ÷ จำนวน instance ที่รันพร้อมกัน
```

ตัวอย่าง: database รับได้ 100 connection, มี 10 instance พร้อมกัน, เผื่อ margin ให้เครื่องมืออื่น (migration script, admin tool) ใช้ร่วม → ตั้ง `SetMaxOpenConns(8)` ต่อ instance (100 × 0.8 ÷ 10 = 8) แทนที่จะตั้งค่าสูงๆ แบบเดาสุ่ม

### `SetMaxIdleConns` ควรใกล้เคียงกับ `SetMaxOpenConns`

ถ้า `SetMaxIdleConns` ต่ำกว่า `SetMaxOpenConns` มาก จะเกิดพฤติกรรมที่ connection ถูกปิดแล้วเปิดใหม่บ่อยเกินความจำเป็นในช่วงที่ traffic ขึ้นๆ ลงๆ (แต่ละครั้งของการเปิด connection ใหม่มี cost จาก TCP handshake + TLS handshake ถ้าเชื่อมแบบเข้ารหัส) แนวทางทั่วไปคือตั้งให้ทั้งสองค่าใกล้เคียงกัน เว้นแต่มีเหตุผลเฉพาะที่ต้องการประหยัด connection ตอน idle จริงๆ

### เชื่อมกับ Monitoring: อย่าตั้งแล้วลืม

**Part 078** สอนการอ่านค่า `db.Stats()` ไปแล้ว — ใน production ค่าเหล่านี้ควรถูก export เป็น Prometheus metric ต่อเนื่อง (ผ่านแนวทางเดียวกับ **Part 099**) โดยเฉพาะ `WaitCount` และ `WaitDuration` ถ้าตัวเลขนี้เพิ่มขึ้นต่อเนื่องใน production หมายความว่า pool เล็กเกินไปเมื่อเทียบกับโหลดจริง และควรกลับไปคำนวณสูตรข้างบนใหม่ด้วยตัวเลขที่อัปเดตแล้ว

---

## 11. Monitoring คือ Performance Tuning แบบต่อเนื่อง (เชื่อม Part 099)

Workflow ทั้งหมดในบทนี้ (load test → pprof → benchmark → load test ซ้ำ) เหมาะกับการ**สืบสวนปัญหาที่รู้ตัวแล้ว** แต่คำถามที่สำคัญกว่าคือ **"จะรู้ได้อย่างไรว่าเมื่อไรควรเริ่มสืบสวน?"** — คำตอบคือ **Part 099** (Prometheus, Grafana, OpenTelemetry) ที่สอนการวัดค่าต่อเนื่องแบบ real-time

**RED Method** ที่ **Part 099** แนะนำไว้ (**R**ate, **E**rrors, **D**uration) คือจุดเริ่มต้นที่ดีที่สุดสำหรับตรวจจับปัญหา performance ก่อนที่ผู้ใช้จะร้องเรียน:

- **Rate** ที่ตกลงกะทันหันอาจบ่งชี้ว่า client เริ่ม timeout แล้วเลิกยิง request (สัญญาณของปัญหาที่เกิดขึ้นแล้ว)
- **Duration** (โดยเฉพาะ p99) ที่ค่อยๆ เพิ่มขึ้นตามเวลาคือสัญญาณเตือนล่วงหน้าคลาสสิกของ resource leak (connection ไม่ถูกปล่อย, goroutine leak, memory leak) — ตรงกับสิ่งที่ **Part 078** สาธิต pool exhaustion ไว้

หลักการปิดท้ายของบทนี้คือ **performance tuning ไม่ใช่กิจกรรมที่ทำครั้งเดียวจบตอน launch** แต่เป็น**วงจรต่อเนื่อง**: monitoring คอยเฝ้าระวังตลอดเวลา → เมื่อพบสัญญาณผิดปกติ ดึง workflow ในบทนี้มาสืบสวนอย่างมีระบบ → แก้ไข → deploy → monitoring ยืนยันผลอีกครั้งว่าตัวเลขจริงดีขึ้นตามที่คาดหวัง

---

## 12. Checklist: Production Performance

- [ ] มี **baseline** ที่วัดด้วย load testing tool จริง (`hey`/`vegeta`/`k6`) ก่อนเริ่ม tuning ใดๆ เสมอ ไม่เดาว่า "น่าจะดีขึ้น"
- [ ] ดู **percentile latency (p95/p99)** ควบคู่กับ average เสมอ ไม่ตัดสินใจจาก average อย่างเดียว
- [ ] เปิด `net/http/pprof` ไว้ในทุก service (บน port/network ที่ปลอดภัยตามที่ **Part 106** แนะนำ) พร้อมใช้งานได้ทันทีเมื่อจำเป็นต้อง investigate โดยไม่ต้อง deploy ใหม่
- [ ] ทุก performance fix มี **benchmark + `benchstat`** ยืนยันผลก่อน merge ไม่ใช่แค่ "ลองรันดูเร็วขึ้น"
- [ ] Import `go.uber.org/automaxprocs` (หรือเทียบเท่า) ในทุก service ที่รันใน container ที่มี CPU limit
- [ ] ตั้ง `GOMEMLIMIT` ให้ต่ำกว่า container memory limit เสมอเมื่อรันใน environment ที่มี memory limit ตายตัว
- [ ] คำนวณ `SetMaxOpenConns`/`SetMaxIdleConns` โดยคำนึงถึง**จำนวน instance ที่รันพร้อมกัน** ไม่ใช่แค่ instance เดียว
- [ ] Export connection pool metrics (`db.Stats()`) และ RED metrics ของทุก endpoint เข้า Prometheus ต่อเนื่อง ไม่ใช่แค่ดูตอน investigate
- [ ] ทำ workflow "measure → fix → measure" **เป็นวงจรซ้ำ** จนกว่าตัวเลขจะเข้าเป้าหมายที่ทีมตั้งไว้ (SLO) ไม่หยุดหลังแก้จุดแรกจุดเดียว
- [ ] Load test ด้วย traffic pattern ที่**ใกล้เคียงของจริง** (concurrency, payload size, ratio ของแต่ละ endpoint) ไม่ใช่แค่ยิง endpoint เดียวซ้ำๆ ด้วยข้อมูลเดียวกันตลอด

---

## สรุปสิ่งที่ได้เรียนในบทนี้

- หลักการเดียวที่คุมทั้งบทนี้คือ **"measure, don't guess"** — ทุกการตัดสินใจ tuning ต้องอิงข้อมูลจริงจากเครื่องมือ ไม่ใช่สัญชาตญาณ
- Workflow มาตรฐานคือ **วัด baseline (`hey`) → หา bottleneck (`pprof`) → แก้แล้ววัดผลระดับฟังก์ชัน (`benchstat`) → วัดผลระดับระบบซ้ำ (`hey` อีกครั้ง) → ทำซ้ำเป็นวงจร**
- เราสาธิตจริงกับบั๊ก `regexp.MustCompile` ถูกเรียกซ้ำทุก request: `pprof` ชี้ว่ากิน CPU ไป 18.68%, การแก้ไข (compile ครั้งเดียวระดับ package) ทำให้ฟังก์ชันเร็วขึ้น ~17 เท่าและลด allocation จาก 66 เหลือ 0 ต่อครั้ง ส่งผลให้ throughput ของ server เพิ่มขึ้นจริง 31% และ p99 latency ลดลง 21%
- ตัวเลขระดับฟังก์ชันกับระดับระบบไม่จำเป็นต้องสัมพันธ์กันแบบเชิงเส้นเสมอไป เพราะยังมีคอขวดอื่นรออยู่ — ต้อง profile ซ้ำหลังแก้ทุกครั้ง
- **`GOMAXPROCS`** ไม่ได้ปรับตาม cgroup CPU limit อัตโนมัติ (สาธิตจริงด้วย Docker `--cpus=1`) ต้องตั้งด้วยมือหรือใช้ `go.uber.org/automaxprocs` เป็นมาตรฐาน
- **`GOGC`/`GOMEMLIMIT`** ควรใช้ร่วมกันใน container: `GOGC` สูงขึ้นเล็กน้อยเพื่อลด CPU overhead ปกติ, `GOMEMLIMIT` ต่ำกว่า container limit เล็กน้อยเป็นตาข่ายนิรภัย
- **Connection pool** ต้องคำนวณตามจำนวน instance ที่รันพร้อมกัน ไม่ใช่คิดแยกทีละ instance
- Monitoring (**Part 099**) ไม่ใช่แค่ dashboard สวยๆ แต่เป็นจุดเริ่มต้นที่บอกว่า "เมื่อไรควรเริ่ม" workflow ทั้งหมดในบทนี้ — performance tuning เป็นวงจรต่อเนื่อง ไม่ใช่งานที่ทำครั้งเดียวจบ

## แบบฝึกหัดท้ายบท

1. สร้าง server ตัวอย่างในบทนี้ขึ้นมาเองในเครื่อง แล้วรัน `hey` ด้วยค่า concurrency ต่างๆ (`-c 10`, `-c 50`, `-c 200`, `-c 500`) สังเกตว่า throughput และ p99 latency เปลี่ยนไปอย่างไรเมื่อ concurrency เพิ่มขึ้น หาจุดที่ throughput เริ่ม "อิ่มตัว" (saturation point)
2. ใช้ `go tool pprof -http=:0 cpu.pprof` เปิด Web UI ดู flame graph ของ profile ที่เก็บได้ในหัวข้อ 4 (ถ้าไม่มีเบราว์เซอร์ ใช้ `-svg` เพื่อ export เป็นไฟล์ภาพแทน)
3. เขียนโค้ดที่มีบั๊กประสิทธิภาพแบบอื่น (เช่น `strings.Builder` ที่ไม่ได้เรียก `Grow()` ล่วงหน้าตาม **Part 085**, หรือ `json.Marshal` struct ขนาดใหญ่ซ้ำๆ โดยไม่จำเป็น) แล้วทำ workflow เต็มรูปแบบตามบทนี้: profile → พบ hotspot → แก้ → benchmark ยืนยันด้วย `benchstat`
4. ทดลองสร้าง Docker image ของ server ตัวอย่าง แล้วรันด้วย `docker run --cpus=0.5` (ครึ่ง core) เทียบ throughput จาก `hey` ระหว่างที่ตั้ง `GOMAXPROCS` ให้ตรงกับ 1 (ค่าปัดขึ้นจาก 0.5) กับปล่อยเป็นค่า default ดูว่าความแตกต่างมีนัยสำคัญแค่ไหนในสถานการณ์จำกัด CPU จริง
5. ทบทวนโค้ด transaction/pool จาก **Part 078** แล้วคำนวณค่า `SetMaxOpenConns` ที่เหมาะสมสำหรับสถานการณ์สมมติ: database รับได้ 200 connection, service มี 15 instance รันพร้อมกัน, ต้องการเผื่อ margin 20% ให้เครื่องมือ admin อื่น
6. ค้นคว้าเพิ่มเติมเรื่อง **`k6`** หรือ **`vegeta`** เปรียบเทียบกับ `hey` — ทั้งสองตัวรองรับการเขียน load test scenario ที่ซับซ้อนกว่า (หลาย endpoint, traffic pattern ที่ค่อยๆ เพิ่มขึ้น) ลองเขียน scenario ทดสอบ endpoint หลายตัวพร้อมกันในสัดส่วนที่ใกล้เคียงการใช้งานจริง

---

**ต่อไป**: [Part 108 — Go Modules Best Practices และ Semantic Versioning](./108-modules-versioning-best-practices.md)
