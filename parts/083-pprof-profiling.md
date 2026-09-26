# Part 083: Profiling ด้วย `pprof`

> ภาคที่ 7: Testing, Tooling, Performance — ตอนที่ 5 จาก 9 (Part 79–87)

## สารบัญของบทนี้

1. Profiling ตอบคำถามอะไรที่ Testing และ Benchmark ตอบไม่ได้
2. CPU Profiling ด้วย `go test -cpuprofile`
3. อ่านผลลัพธ์ด้วย `go tool pprof`: `top` และ `list`
4. แนวคิด Flame Graph (เมื่อไม่มีเบราว์เซอร์ให้เปิด Web UI)
5. Memory Profiling ด้วย `-memprofile`
6. เปิด Profiling บนเซิร์ฟเวอร์ที่กำลังรันจริงด้วย `net/http/pprof`
7. Profiling โปรแกรมแบบ CLI ด้วย `runtime/pprof` โดยตรง
8. ตัวอย่างเต็ม: หา Hotspot แล้ว Optimize พร้อมวัดผลก่อน-หลัง
9. สรุปสิ่งที่ได้เรียนในบทนี้
10. แบบฝึกหัดท้ายบท

---

## 1. Profiling ตอบคำถามอะไรที่ Testing และ Benchmark ตอบไม่ได้

**Part 033-034** และ **Part 079** สอนวิธีตอบคำถามว่า **"โค้ดทำงานถูกต้องหรือไม่"** (testing) ส่วน **Part 034** ยังสอนวิธีตอบคำถามว่า **"โค้ด A เร็วกว่าโค้ด B แค่ไหน"** (benchmark) — สิ่งที่ทั้งสองเรื่องนี้ยังตอบไม่ได้คือคำถามที่สำคัญไม่แพ้กันเมื่อโปรแกรมช้าจริงในระดับระบบใหญ่:

> **"เวลา (หรือหน่วยความจำ) ที่ใช้ไปทั้งหมดนั้น ไปอยู่ที่ฟังก์ชันไหนกันแน่?"**

Benchmark บอกได้แค่ "ฟังก์ชันทั้งก้อนใช้เวลาเท่าไหร่โดยรวม" แต่ถ้าฟังก์ชันนั้นมีความซับซ้อนภายในหลายสิบบรรทัด เรียกฟังก์ชันย่อยอีกหลายตัว benchmark เพียงอย่างเดียวไม่มีทางบอกได้เลยว่า **80% ของเวลาไปอยู่ที่บรรทัดไหน หรือฟังก์ชันย่อยตัวไหน** — นี่คือช่องว่างที่ **profiling** เข้ามาเติมเต็ม

**Profiling** คือการสุ่มเก็บตัวอย่าง (sampling) ว่าโปรแกรมกำลังทำอะไรอยู่ ณ ขณะนั้นๆ ระหว่างการทำงานจริง แล้วนำข้อมูลตัวอย่างจำนวนมากมารวมกันเป็นภาพรวมว่า **"เวลาส่วนใหญ่ถูกใช้ไปกับ call stack แบบไหนบ่อยที่สุด"** Go มีเครื่องมือ profiling ในตัวผ่าน package `runtime/pprof` และ `net/http/pprof` พร้อมเครื่องมือวิเคราะห์ผล `go tool pprof` — ทั้งหมดนี้เป็นส่วนหนึ่งของ toolchain มาตรฐาน ไม่ต้องติดตั้งอะไรเพิ่มเลย (สอดคล้องกับปรัชญา "tooling ในตัวครบ" จาก **Part 001**)

Go รองรับ profile หลายประเภท ที่ใช้บ่อยที่สุดคือ:

| ประเภท Profile | ตอบคำถามว่า |
|---|---|
| **CPU profile** | เวลา CPU ถูกใช้ไปกับฟังก์ชันไหนมากที่สุด |
| **Memory (heap) profile** | หน่วยความจำถูก allocate ที่จุดไหนในโค้ดมากที่สุด |
| **Goroutine profile** | มี goroutine ค้างอยู่กี่ตัว และค้างอยู่ที่จุดไหน (มีประโยชน์มากตอนหา goroutine leak) |
| **Block profile** | goroutine ไหนรอ (block) อยู่ที่ mutex/channel นานที่สุด |

บทนี้โฟกัสที่ 2 ประเภทแรกซึ่งใช้บ่อยที่สุดในทางปฏิบัติ

---

## 2. CPU Profiling ด้วย `go test -cpuprofile`

วิธีที่ง่ายที่สุดในการเก็บ CPU profile คือผ่าน benchmark ที่เรียนไปแล้วใน **Part 034** เพิ่ม flag `-cpuprofile` เข้าไปตอนรัน `go test -bench`:

```bash
go test -bench=BenchmarkCountWordsSlow -benchtime=3s -run=^$ -cpuprofile=cpu.prof ./wordfreq/...
```

```
goos: linux
goarch: amd64
pkg: part083demo/wordfreq
cpu: Intel(R) Xeon(R) Processor @ 2.10GHz
BenchmarkCountWordsSlow-4   	     193	  17893257 ns/op
PASS
ok  	part083demo/wordfreq	5.528s
```

`-run=^$` (ทบทวนจาก **Part 034**) ปิดการรัน test ปกติเพื่อโฟกัสที่ benchmark เท่านั้น `-benchtime=3s` สั่งให้ benchmark รันต่อเนื่องอย่างน้อย 3 วินาที (นานกว่าค่า default) เพื่อให้ profiler มีเวลาเก็บตัวอย่างได้มากพอที่จะให้ผลลัพธ์ที่แม่นยำทางสถิติ — profiling ที่รันสั้นเกินไป (เช่นต่ำกว่า 1 วินาที) มักได้ตัวอย่างน้อยเกินกว่าจะสรุปอะไรได้ชัดเจน

คำสั่งนี้สร้างไฟล์ `cpu.prof` ขึ้นมา (เป็น binary format ที่ `go tool pprof` เข้าใจ ไม่ใช่ text ที่อ่านตรงๆ ได้) พร้อมกับไฟล์ **binary ของ test เอง** (`wordfreq.test`) ที่ `go tool pprof` ต้องใช้คู่กันเพื่อแปล address กลับเป็นชื่อฟังก์ชันและเลขบรรทัดได้ถูกต้อง (สร้างขึ้นเองอัตโนมัติถ้ายังไม่มี หรือระบุ path ตรงๆ ก็ได้)

**Profiling ผ่าน `go test` ใช้กับโค้ดที่ทดสอบผ่าน benchmark ได้สะดวกที่สุด** ส่วนการ profile โปรแกรมที่กำลังรันเป็นเซิร์ฟเวอร์จริง (ไม่ใช่ benchmark) ต้องใช้วิธีอื่นที่จะพูดถึงในหัวข้อ 6

---

## 3. อ่านผลลัพธ์ด้วย `go tool pprof`: `top` และ `list`

เปิดไฟล์ profile ด้วย `go tool pprof`:

```bash
go tool pprof -top -nodecount=10 ./wordfreq.test cpu.prof
```

`-top` แสดงรายชื่อฟังก์ชันที่ใช้เวลามากที่สุดเรียงจากมากไปน้อย, `-nodecount=10` จำกัดแค่ 10 อันดับแรก (ค่า default คือ 10 อยู่แล้ว แต่ระบุชัดเจนไว้เพื่อความแน่ใจ):

```
File: wordfreq.test
Type: cpu
Time: 2026-09-26 06:02:32 UTC
Duration: 5.52s, Total samples = 5.36s (97.08%)
Showing nodes accounting for 5.18s, 96.64% of 5.36s total
Dropped 65 nodes (cum <= 0.03s)
Showing top 10 nodes out of 19
      flat  flat%   sum%        cum   cum%
     2.24s 41.79% 41.79%      2.24s 41.79%  memeqbody
     2.10s 39.18% 80.97%      5.31s 99.07%  part083demo/wordfreq.CountWordsSlow
     0.79s 14.74% 95.71%      0.79s 14.74%  runtime.memequal
     0.03s  0.56% 96.27%      0.03s  0.56%  internal/runtime/maps.ctrlGroup.matchH2 (inline)
     0.01s  0.19% 96.46%      0.06s  1.12%  internal/runtime/maps.(*table).split
     0.01s  0.19% 96.64%      0.12s  2.24%  runtime.mapassign_faststr
         0     0% 96.64%      0.06s  1.12%  internal/runtime/maps.(*table).rehash
         0     0% 96.64%      0.03s  0.56%  internal/runtime/maps.newTable
         0     0% 96.64%      5.31s 99.07%  part083demo/wordfreq.BenchmarkCountWordsSlow
         0     0% 96.64%      0.03s  0.56%  runtime.gcBgMarkWorker
```

**อ่านคอลัมน์ให้เป็น:**

| คอลัมน์ | ความหมาย |
|---|---|
| **`flat`** | เวลาที่ใช้ **ในฟังก์ชันนี้เอง** โดยตรง ไม่นับเวลาที่ฟังก์ชันย่อยที่มันเรียกใช้ |
| **`flat%`** | `flat` คิดเป็นกี่เปอร์เซ็นต์ของเวลารวมทั้งหมด |
| **`sum%`** | ผลรวมสะสมของ `flat%` จากบนลงมาถึงแถวนี้ |
| **`cum`** (cumulative) | เวลาที่ใช้ **ในฟังก์ชันนี้รวมถึงฟังก์ชันย่อยทั้งหมดที่มันเรียก** |
| **`cum%`** | `cum` คิดเป็นกี่เปอร์เซ็นต์ของเวลารวม |

จากผลลัพธ์นี้อ่านได้ทันทีว่า: **`CountWordsSlow` กิน CPU ไปถึง 99.07% ของเวลาทั้งหมด** (`cum%`) ซึ่งสมเหตุสมผลเพราะ benchmark เรียกแค่ฟังก์ชันนี้ตัวเดียว แต่ที่น่าสนใจกว่าคือ **`memeqbody`** (ฟังก์ชันภายในของ runtime ที่ใช้เปรียบเทียบ string ว่าเท่ากันหรือไม่) และ **`runtime.memequal`** รวมกันกิน CPU ไปเกือบ **56%** (41.79% + 14.74%) — นี่คือสัญญาณชัดเจนว่า **โค้ดของเรากำลังเปรียบเทียบ string กันบ่อยเกินความจำเป็นมาก**

### เจาะลึกด้วย `list`: ดูเป็นรายบรรทัด

`top` บอกแค่ระดับฟังก์ชัน แต่ `list` แสดง **เวลาที่ใช้ในแต่ละบรรทัดของฟังก์ชันที่ระบุ** ซึ่งช่วยชี้เป้าได้แม่นยำกว่ามาก:

```bash
go tool pprof -list=CountWordsSlow ./wordfreq.test cpu.prof
```

```
Total: 5.36s
ROUTINE ======================== part083demo/wordfreq.CountWordsSlow in .../wordfreq.go
     2.10s      5.31s (flat, cum) 99.07% of Total
         .          .     10:func CountWordsSlow(text string) map[string]int {
         .       20ms     11:	words := strings.Fields(text)
         .          .     12:	var seen []string
         .          .     13:	counts := make(map[string]int)
         .          .     14:
         .          .     15:	for _, w := range words {
         .       10ms     16:		w = strings.ToLower(w)
         .          .     17:
         .          .     18:		found := false
     1.21s      1.21s     19:		for _, s := range seen { // จุดที่เป็นปัญหา: linear search แทนที่จะใช้ map lookup
     730ms      3.76s     20:			if s == w {
         .          .     21:				found = true
         .          .     22:				break
         .          .     23:			}
         .          .     24:		}
         .          .     25:		if !found {
     160ms      190ms     26:			seen = append(seen, w)
         .          .     27:		}
         .          .     28:
         .      120ms     29:		counts[w]++
         .          .     30:	}
         .          .     31:	return counts
         .          .     32:}
```

**เห็นชัดเจนทันที**: บรรทัด 19 (`for _, s := range seen`) และบรรทัด 20 (`if s == w`) รวมกันมี `cum` สูงถึง **3.76 วินาที จาก 5.36 วินาทีทั้งหมด (ประมาณ 70%)** — นี่คือ **hotspot** ที่แท้จริงของฟังก์ชันนี้ ตรงกับที่คาดไว้ตอนออกแบบโค้ด: ลูปค้นหาเชิงเส้น (linear search) ที่ควรใช้ map แทน

รูปแบบการทำงานที่แนะนำคือ **เริ่มจาก `top` เพื่อหาว่า "ฟังก์ชันไหน" น่าสงสัยที่สุดก่อน แล้วค่อยใช้ `list=<ชื่อฟังก์ชัน>` เจาะลึกไปที่ "บรรทัดไหน"** ภายในฟังก์ชันนั้นอีกที — ลำดับนี้เร็วกว่าการไล่อ่านโค้ดทั้งไฟล์ด้วยตาเปล่ามาก โดยเฉพาะกับโค้ดที่มีหลายพันบรรทัด

---

## 4. แนวคิด Flame Graph (เมื่อไม่มีเบราว์เซอร์ให้เปิด Web UI)

`go tool pprof` มีโหมด **Web UI** ที่เปิดเบราว์เซอร์แสดงผลแบบกราฟิกได้ทันที:

```bash
go tool pprof -http=:8080 ./wordfreq.test cpu.prof
```

หน้าเว็บที่ได้จะมีมุมมองหลายแบบ ที่มีชื่อเสียงที่สุดคือ **Flame Graph** — แนวคิดของมันคือ:

- **แกนแนวนอน (X)** แทน**สัดส่วนของเวลา** ที่ใช้ไป (ไม่ใช่ลำดับเวลา!) — แท่งที่กว้างกว่าหมายถึงกินเวลามากกว่า ไม่ได้หมายถึงเกิดก่อนหรือหลัง
- **แกนแนวตั้ง (Y)** แทน **ความลึกของ call stack** — ฟังก์ชันที่อยู่ด้านล่างสุดเรียกฟังก์ชันที่อยู่ด้านบนมันขึ้นไปเรื่อยๆ
- แต่ละแท่งสี่เหลี่ยมคือหนึ่งฟังก์ชันใน call stack แท่งที่ **กว้างและอยู่สูงตระหง่านขึ้นไปเป็น "ยอดแหลม" (flame)** ที่ด้านบน คือจุดที่ควรสงสัยเป็นอันดับแรก เพราะแปลว่าฟังก์ชันนั้นถูกเรียกจาก call path เดียวกันซ้ำๆ และกินเวลาไปมาก

Flame graph มีประโยชน์มากเมื่อ call stack ลึกหลายสิบชั้น เพราะมองเห็นภาพรวมทั้งหมดได้ในหน้าจอเดียว ต่างจาก `top`/`list` ที่ต้องไล่ดูทีละฟังก์ชัน

**ในสภาพแวดล้อมที่ไม่มีเบราว์เซอร์ใช้งานได้** (เช่น container, remote server ผ่าน SSH, หรือ sandbox แบบที่ใช้เขียนบทเรียนนี้ที่ไม่มี GUI) `-http` จะไม่มีประโยชน์เลย — นี่คือเหตุผลที่บทนี้เน้นโหมด **text-based** (`-top`, `-list`) เป็นหลัก เพราะทำงานได้ในทุกสภาพแวดล้อมโดยไม่ต้องพึ่งเบราว์เซอร์ ให้ข้อมูลเชิงตัวเลขที่แม่นยำเท่ากัน เพียงแต่ไม่มีภาพกราฟิกให้ดูเท่านั้น — ในทางปฏิบัติเมื่อทำงานบนเครื่องพัฒนาที่มี GUI การใช้ `-http` สะดวกกว่ามาก แต่หลักการอ่านผล (หาฟังก์ชัน/บรรทัดที่กิน `flat`/`cum` เวลาเยอะที่สุด) เหมือนกันทุกประการไม่ว่าจะดูผ่านรูปแบบไหน

คำสั่ง `go tool pprof` ยังมีโหมด interactive (พิมพ์ `go tool pprof cpu.prof` เฉยๆ โดยไม่ระบุ `-top`/`-list`/`-http`) ที่เปิด prompt ให้พิมพ์คำสั่งย่อยๆ ได้ต่อเนื่องในเซสชันเดียว (เช่น `top10`, `list <func>`, `web`) เหมาะกับการสำรวจ profile แบบลึกหลายรอบโดยไม่ต้อง parse ไฟล์ใหม่ทุกครั้ง

---

## 5. Memory Profiling ด้วย `-memprofile`

เพิ่ม `-memprofile` เข้าไปตอนรัน benchmark เพื่อเก็บข้อมูลว่าหน่วยความจำถูก allocate ที่จุดไหนบ้าง (คู่กับ `-benchmem` ที่เรียนใน **Part 034** ซึ่งให้แค่ตัวเลขสรุป ไม่ได้บอกตำแหน่งในโค้ด):

```bash
go test -bench=BenchmarkCountWordsSlow -benchtime=3s -run=^$ -memprofile=mem.prof ./wordfreq/...
```

วิเคราะห์ด้วย `go tool pprof` เหมือนเดิม แต่เพิ่ม `-alloc_space` เพื่อดูจากมุมมอง **"allocate ไปทั้งหมดกี่ byte ตลอดการรัน"** (มีมุมมองอื่นให้เลือกด้วย เช่น `-inuse_space` ที่ดูเฉพาะหน่วยความจำที่ยัง "ค้างอยู่" ณ ขณะที่เก็บ profile ซึ่งมีประโยชน์มากตอนตามหา memory leak):

```bash
go tool pprof -top -nodecount=8 -alloc_space ./wordfreq.test mem.prof
```

```
File: wordfreq.test
Type: alloc_space
Showing nodes accounting for 203.36MB, 98.71% of 206.01MB total
Dropped 26 nodes (cum <= 1.03MB)
Showing top 8 nodes out of 11
      flat  flat%   sum%        cum   cum%
  188.34MB 91.42% 91.42%   201.64MB 97.87%  part083demo/wordfreq.CountWordsSlow
   13.30MB  6.45% 97.87%    13.30MB  6.45%  strings.Fields
    1.72MB  0.84% 98.71%     1.72MB  0.84%  runtime/pprof.StartCPUProfile
         0     0% 98.71%     1.72MB  0.84%  main.main
         0     0% 98.71%   202.16MB 98.13%  part083demo/wordfreq.BenchmarkCountWordsSlow
         0     0% 98.71%     1.72MB  0.84%  runtime.main
         0     0% 98.71%   201.64MB 97.87%  testing.(*B).launch
         0     0% 98.71%   202.66MB 98.37%  testing.(*B).runN
```

`CountWordsSlow` allocate ไปทั้งหมด **188.34 MB** ตลอดการรัน benchmark — ตัวเลขนี้สูงเพราะทุกครั้งที่ `seen = append(seen, w)` ทำงาน (เมื่อ slice เต็มความจุเดิม) Go ต้อง allocate array ใหม่ที่ใหญ่ขึ้นแล้ว copy ข้อมูลเดิมทั้งหมดไป (ทบทวนพฤติกรรมการขยายขนาดของ slice จาก **Part 006**) ยิ่ง `seen` มีขนาดใหญ่ขึ้นเรื่อยๆ การ allocate ซ้ำก็ยิ่งกินหน่วยความจำมากตามไปด้วย

**หลักการอ่าน memory profile**: `flat` สูงหมายถึง **โค้ดตรงจุดนั้นเป็นคน allocate โดยตรง** (เช่น `make`, `append` ที่ต้องขยายขนาด, การสร้าง struct/slice/map ใหม่) — ถ้าเจอฟังก์ชันที่ allocate เยอะผิดปกติ ให้ตรวจสอบว่ามีการสร้างข้อมูลซ้ำโดยไม่จำเป็นหรือ append เข้า slice/map ที่ไม่ได้จอง capacity ไว้ล่วงหน้าหรือไม่ (การใช้ `make([]T, 0, knownSize)` จอง capacity ล่วงหน้าเมื่อรู้ขนาดคร่าวๆ อยู่แล้ว เป็นเทคนิคลดการ allocate ซ้ำที่ใช้บ่อยมาก)

---

## 6. เปิด Profiling บนเซิร์ฟเวอร์ที่กำลังรันจริงด้วย `net/http/pprof`

หัวข้อ 2-5 ใช้ profiling ผ่าน benchmark ซึ่งเหมาะกับการวิเคราะห์ฟังก์ชันเดี่ยวๆ แต่คำถามที่พบบ่อยกว่าในโปรดักชันจริงคือ **"เซิร์ฟเวอร์ที่กำลังรันอยู่ตอนนี้ ใช้ CPU/memory ไปกับอะไร"** — สำหรับกรณีนี้ Go มี package พิเศษชื่อ **`net/http/pprof`** ที่ลงทะเบียน HTTP endpoint สำหรับดึง profile ออกมาจากเซิร์ฟเวอร์ที่กำลังทำงานอยู่จริง โดยแทบไม่ต้องเขียนโค้ดเพิ่มเลย:

```go
// cmd/server/main.go
package main

import (
	"fmt"
	"log"
	"net/http"
	_ "net/http/pprof" // side-effect import: ลงทะเบียน handler ทั้งหมดไว้ที่ /debug/pprof/ ให้อัตโนมัติ
)

func main() {
	http.HandleFunc("/hello", func(w http.ResponseWriter, r *http.Request) {
		fmt.Fprintln(w, "Hello from the demo server!")
	})

	// การ import "net/http/pprof" เฉยๆ (แม้ไม่เรียกใช้ฟังก์ชันใดจากมันตรงๆ) ก็เพียงพอแล้ว
	// เพราะไฟล์ init() ภายใน package นั้นลงทะเบียน handler ต่างๆ ไว้ที่ http.DefaultServeMux
	// โดยอัตโนมัติ: /debug/pprof/, /debug/pprof/profile, /debug/pprof/heap,
	// /debug/pprof/goroutine, /debug/pprof/allocs ฯลฯ
	log.Println("listening on :6060 (try /debug/pprof/ and /hello)")
	log.Fatal(http.ListenAndServe("localhost:6060", nil))
}
```

จุดสำคัญที่สุดของโค้ดนี้คือบรรทัด `import _ "net/http/pprof"` — เครื่องหมาย `_` (blank identifier ทบทวนจาก **Part 002**) หมายถึง "import package นี้เพื่อผลข้างเคียง (side effect) เท่านั้น ไม่ได้เรียกใช้ชื่ออะไรจากมันตรงๆ" ผลข้างเคียงที่ว่าคือฟังก์ชัน `init()` ภายใน package `net/http/pprof` จะถูกเรียกอัตโนมัติตอนโปรแกรมเริ่มทำงาน และมันจะไปลงทะเบียน handler หลายตัวไว้ที่ `http.DefaultServeMux` (ทบทวน mux เริ่มต้นจาก **Part 056**) ให้เอง

รันเซิร์ฟเวอร์แล้วดู endpoint ทั้งหมดที่มีให้:

```bash
go run ./cmd/server
```

```
2026/09/26 06:03:17 listening on :6060 (try /debug/pprof/ and /hello)
```

เปิดดูรายการ endpoint ผ่าน `curl` (หรือเบราว์เซอร์ถ้ามี GUI):

```bash
curl -s http://localhost:6060/debug/pprof/
```

```html
<html>
<head><title>/debug/pprof/</title></head>
<body>
/debug/pprof/
<p>Set debug=1 as a query parameter to export in legacy text format</p>
Types of profiles available:
<table>
<thead><td>Count</td><td>Profile</td></thead>
<tr><td>2</td><td><a href='allocs?debug=1'>allocs</a></td></tr>
<tr><td>0</td><td><a href='block?debug=1'>block</a></td></tr>
...
```

> **ยืนยันแล้วจริง**: เซิร์ฟเวอร์ตัวอย่างนี้รันจริงบนเครื่องที่ใช้เขียนบทเรียนนี้ `curl http://localhost:6060/hello` ตอบกลับข้อความปกติ และ `/debug/pprof/` แสดงรายชื่อ profile ที่มีให้ดึงจริงตามที่อธิบายไว้

ดึง CPU profile จากเซิร์ฟเวอร์ที่กำลังรันสด (จะบันทึกตัวอย่างต่อเนื่อง 30 วินาทีตามที่ระบุ):

```bash
curl -s "http://localhost:6060/debug/pprof/profile?seconds=30" -o server_cpu.prof
```

ระหว่าง 30 วินาทีนั้น **ต้องมีการยิง request จริงเข้าเซิร์ฟเวอร์** (จากผู้ใช้จริง หรือเครื่องมือ load testing) ไม่งั้น profile ที่ได้จะแทบว่างเปล่า เพราะไม่มีงานให้ CPU ทำเลยในช่วงเวลานั้น (ทดสอบแล้วว่าเซิร์ฟเวอร์ที่ไม่มี traffic เข้าเลยจะได้ `Total samples = 0` เมื่อนำไปเปิดด้วย `go tool pprof`) — เมื่อได้ไฟล์แล้ว วิเคราะห์ด้วย `go tool pprof` ได้ในรูปแบบเดียวกับ profile ที่ได้จาก benchmark ทุกประการ:

```bash
go tool pprof -top server_cpu.prof
```

**Endpoint อื่นที่มีประโยชน์ใน `/debug/pprof/`:**

| Endpoint | ใช้ทำอะไร |
|---|---|
| `/debug/pprof/heap` | Memory profile ปัจจุบัน (ดึงได้ทันที ไม่ต้องรอเหมือน CPU) |
| `/debug/pprof/goroutine?debug=1` | รายชื่อ goroutine ทั้งหมดที่กำลังทำงาน พร้อม stack trace — สำคัญมากตอนสงสัยว่ามี goroutine leak (ทบทวนแนวคิด goroutine จาก **Part 036**) |
| `/debug/pprof/profile?seconds=N` | CPU profile ต่อเนื่อง N วินาที |
| `/debug/pprof/allocs` | สรุปการ allocate หน่วยความจำสะสมตั้งแต่โปรแกรมเริ่มทำงาน |

> **คำเตือนด้านความปลอดภัย**: `net/http/pprof` ลงทะเบียน handler ไว้ที่ `http.DefaultServeMux` ซึ่งเป็น mux เดียวกับที่ `http.ListenAndServe(addr, nil)` ใช้เป็นค่าเริ่มต้น **ห้ามเปิด endpoint นี้ให้เข้าถึงได้จากอินเทอร์เน็ตสาธารณะโดยตรงเด็ดขาด** เพราะมันเปิดเผยข้อมูลภายในของโปรแกรม (source path, memory layout, goroutine stack) ที่อาจเป็นประโยชน์ต่อผู้โจมตี ในโปรดักชันจริงมักแยกพอร์ต pprof ออกจากพอร์ต API หลัก (ดังตัวอย่างที่ผูกกับ `localhost:6060` เท่านั้น ไม่ใช่ `0.0.0.0`) หรือเปิดผ่าน mux แยกต่างหากที่ป้องกันด้วย authentication/firewall เสมอ

---

## 7. Profiling โปรแกรมแบบ CLI ด้วย `runtime/pprof` โดยตรง

นอกจาก benchmark (หัวข้อ 2) และเซิร์ฟเวอร์ที่รันต่อเนื่อง (หัวข้อ 6) ยังมีสถานการณ์ที่สามที่พบบ่อยมากในทางปฏิบัติ: **โปรแกรมแบบ CLI หรือ batch job ที่รันครั้งเดียวจบ** (เช่น สคริปต์ประมวลผลไฟล์ขนาดใหญ่, data pipeline, cron job) ซึ่งไม่ใช่ทั้ง benchmark และไม่ใช่ทั้งเซิร์ฟเวอร์ที่เปิดค้างรอ request สำหรับกรณีนี้ package **`runtime/pprof`** (คนละตัวกับ `net/http/pprof` ในหัวข้อ 6 แต่ทำงานร่วมกันได้) ให้ฟังก์ชันสำหรับเริ่ม/หยุดเก็บ CPU profile ได้โดยตรงในโค้ด `main()`:

```go
// cmd/cli/main.go
package main

import (
	"flag"
	"fmt"
	"log"
	"os"
	"runtime/pprof"
	"strconv"
	"strings"

	"part083demo/wordfreq"
)

func main() {
	cpuProfilePath := flag.String("cpuprofile", "", "path ที่จะเขียน CPU profile ลงไป (ว่าง = ไม่เก็บ)")
	flag.Parse()

	if *cpuProfilePath != "" {
		f, err := os.Create(*cpuProfilePath)
		if err != nil {
			log.Fatalf("create cpu profile file: %v", err)
		}
		defer f.Close()

		if err := pprof.StartCPUProfile(f); err != nil {
			log.Fatalf("start cpu profile: %v", err)
		}
		defer pprof.StopCPUProfile() // สำคัญมาก: ต้องเรียกก่อนโปรแกรมจบเสมอ ไม่งั้นไฟล์ profile จะไม่สมบูรณ์
	}

	text := generateSampleText(20000)
	counts := wordfreq.CountWordsSlow(text)
	fmt.Printf("processed %d unique words\n", len(counts))
}

func generateSampleText(n int) string {
	words := make([]string, n)
	for i := 0; i < n; i++ {
		words[i] = "word" + strconv.Itoa(i)
	}
	return strings.Join(words, " ")
}
```

**จุดสำคัญที่ต้องระวังเสมอเมื่อใช้ `runtime/pprof` แบบตรงๆ ในโค้ด:**

1. **`pprof.StartCPUProfile(f)` ต้องเรียกก่อนงานที่ต้องการ profile เริ่มทำงาน** และ **`pprof.StopCPUProfile()` ต้องถูกเรียกก่อนโปรแกรมจบเสมอ** (นิยมใช้ `defer` ไว้ทันทีหลัง `Start` สำเร็จ ตามรูปแบบมาตรฐานของ `defer` ที่เรียนจาก **Part 009**) ถ้าโปรแกรมจบแบบกะทันหันโดยไม่เรียก `Stop` (เช่น เกิด panic ที่ไม่มี `recover`, หรือเรียก `os.Exit` ตรงๆ ก่อนถึง `defer`) ไฟล์ profile ที่ได้จะไม่สมบูรณ์หรือใช้งานไม่ได้เลย
2. **ทำงานได้กับทุกโปรแกรม Go ไม่จำกัดว่าต้องมี HTTP server** ต่างจาก `net/http/pprof` ที่ต้องมีเซิร์ฟเวอร์ HTTP รันอยู่ก่อนถึงจะดึง profile ผ่าน endpoint ได้
3. รูปแบบนี้เหมาะมากกับการเปิด flag `-cpuprofile` แบบมีเงื่อนไข (ตามตัวอย่าง) เพื่อให้โปรแกรมตัวเดียวกันรันได้ทั้งแบบปกติ (ไม่มี overhead ของการเก็บ profile) และแบบเก็บ profile เมื่อต้องการวิเคราะห์ปัญหา

รันและตรวจสอบ:

```bash
go run ./cmd/cli -cpuprofile=cli_cpu.prof
```

```
processed 20000 unique words
```

```bash
go tool pprof -top -nodecount=6 cli_cpu.prof
```

```
File: cli
Type: cpu
Duration: 501.58ms, Total samples = 310ms (61.81%)
Showing nodes accounting for 310ms, 100% of 310ms total
Showing top 6 nodes out of 11
      flat  flat%   sum%        cum   cum%
     170ms 54.84% 54.84%      310ms   100%  part083demo/wordfreq.CountWordsSlow
      90ms 29.03% 83.87%       90ms 29.03%  memeqbody
      40ms 12.90% 96.77%       40ms 12.90%  runtime.memequal
      10ms  3.23%   100%       10ms  3.23%  internal/runtime/maps.(*ctrlGroup).setEmpty (inline)
```

> **ยืนยันแล้วจริง**: รันคำสั่งข้างต้นบนเครื่องที่ใช้เขียนบทเรียนนี้ ได้ผลลัพธ์ตรงตามที่แสดง — เห็น `CountWordsSlow` และฟังก์ชันเปรียบเทียบ string (`memeqbody`, `runtime.memequal`) เป็น hotspot เดียวกันกับที่พบตอนใช้ `go test -cpuprofile` ในหัวข้อ 2-3 ยืนยันว่าไม่ว่าจะเก็บ profile ผ่านช่องทางไหน (benchmark, HTTP endpoint, หรือ `runtime/pprof` ตรงๆ) ผลการวิเคราะห์ที่ได้สอดคล้องกันเสมอ เพราะทั้งหมดใช้กลไก sampling เดียวกันของ Go runtime เบื้องหลัง

สังเกตด้วยว่า `Total samples = 310ms (61.81%)` ในขณะที่ `Duration` ทั้งหมดคือ 501.58ms — ตัวเลขทั้งสองนี้**ไม่เท่ากันเสมอไป**เป็นเรื่องปกติ เพราะ CPU profiler สุ่มเก็บตัวอย่างเป็นช่วงๆ (ค่า default คือประมาณ 100 ครั้งต่อวินาที) ไม่ได้จับภาพทุกนาโนวินาทีที่โปรแกรมทำงาน ส่วนที่เหลืออีก ~38% อาจเป็นช่วงที่ CPU ว่าง (รอ I/O, รอ garbage collector) หรือช่วงเวลาที่สั้นเกินกว่าจะถูกสุ่มตัวอย่างพอดี ยิ่งรันนานขึ้นหรือมีงานให้ CPU ทำมากขึ้น สัดส่วน `Total samples` ต่อ `Duration` จะยิ่งเข้าใกล้ 100% มากขึ้น (ดูตัวอย่างในหัวข้อ 3 ที่ได้ 97.08% เพราะรันด้วย `-benchtime=3s` ที่นานกว่า)

---

## 8. ตัวอย่างเต็ม: หา Hotspot แล้ว Optimize พร้อมวัดผลก่อน-หลัง

มารวมทุกอย่างเข้าด้วยกันเป็นกระบวนการทำงานแบบเต็มรูปแบบ: จากฟังก์ชันที่ไม่มีประสิทธิภาพ (ที่ใช้ในหัวข้อ 2-3) ไปจนถึงการแก้ไขและวัดผลจริง

**ฟังก์ชันเดิม (ช้า)**: `CountWordsSlow` นับความถี่คำในข้อความ แต่ใช้ linear search เช็คว่าเคยเจอคำนี้มาก่อนหรือยัง (โค้ดเต็มอยู่ในหัวข้อ 3) — จากการวิเคราะห์ด้วย `go tool pprof -list` เราพบแล้วว่า hotspot ที่แท้จริงคือบรรทัด `for _, s := range seen` และ `if s == w` ที่กิน 70% ของเวลาทั้งหมด

**ฟังก์ชันที่ optimize แล้ว**: เอา `seen` slice กับ linear search ออกไปทั้งหมด ใช้ประโยชน์จาก map ที่มีอยู่แล้ว (ซึ่งตรวจสอบ key ซ้ำแบบ O(1) อยู่แล้วในตัว โดยไม่ต้องมี logic เพิ่มเติมเลย):

```go
// CountWordsFast นับความถี่ของแต่ละคำ โดยใช้ map ตรงๆ (O(1) ต่อการ lookup/insert หนึ่งครั้ง)
// ไม่ต้องเก็บ seen slice แยกต่างหากเลย เพราะ map ทำหน้าที่นั้นอยู่แล้วในตัว
func CountWordsFast(text string) map[string]int {
	counts := make(map[string]int)
	for _, w := range strings.Fields(text) {
		counts[strings.ToLower(w)]++
	}
	return counts
}
```

ยืนยันก่อนว่าทั้งสองฟังก์ชันให้ผลลัพธ์**เหมือนกันทุกประการ** ด้วย unit test ปกติ (ทบทวนหลักการ "ทดสอบความถูกต้องก่อนวัดประสิทธิภาพเสมอ" — การ optimize โค้ดที่ผิดให้เร็วขึ้นไม่มีประโยชน์อะไรเลย):

```go
func TestCountWords_BothImplementationsAgree(t *testing.T) {
	text := "the Quick brown Fox jumps over the lazy dog THE fox runs"

	slow := CountWordsSlow(text)
	fast := CountWordsFast(text)

	if !reflect.DeepEqual(slow, fast) {
		t.Errorf("CountWordsSlow() = %v, CountWordsFast() = %v; ผลลัพธ์ควรเหมือนกันทุกประการ", slow, fast)
	}
}
```

```
=== RUN   TestCountWords_BothImplementationsAgree
--- PASS: TestCountWords_BothImplementationsAgree (0.00s)
```

ผ่าน — ทั้งสอง implementation ให้ผลลัพธ์เหมือนกันทุกประการ ตอนนี้วัดความเร็วเทียบกันด้วย benchmark (ทบทวนเทคนิค `sink` และ `b.ResetTimer()` จาก **Part 034**):

```go
var sink map[string]int

func genText(n int) string {
	words := make([]string, n)
	for i := 0; i < n; i++ {
		words[i] = fmt.Sprintf("word%d", i)
	}
	return strings.Join(words, " ")
}

func BenchmarkCountWordsSlow(b *testing.B) {
	text := genText(4000)
	b.ResetTimer()
	var r map[string]int
	for i := 0; i < b.N; i++ {
		r = CountWordsSlow(text)
	}
	sink = r
}

func BenchmarkCountWordsFast(b *testing.B) {
	text := genText(4000)
	b.ResetTimer()
	var r map[string]int
	for i := 0; i < b.N; i++ {
		r = CountWordsFast(text)
	}
	sink = r
}
```

รันเทียบกันด้วย `-benchmem`:

```bash
go test -bench=. -benchmem -run=^$ ./wordfreq/...
```

```
goos: linux
goarch: amd64
pkg: part083demo/wordfreq
cpu: Intel(R) Xeon(R) Processor @ 2.10GHz
BenchmarkCountWordsSlow-4   	      66	  18691680 ns/op	  742185 B/op	      64 allocs/op
BenchmarkCountWordsFast-4   	    2450	    453507 ns/op	  502181 B/op	      49 allocs/op
PASS
ok  	part083demo/wordfreq	2.421s
```

**ผลลัพธ์ก่อน-หลังการ optimize (วัดจริงบนข้อความ 4,000 คำที่แทบไม่ซ้ำกันเลย):**

| ตัวชี้วัด | `CountWordsSlow` (ก่อน) | `CountWordsFast` (หลัง) | ดีขึ้น |
|---|---|---|---|
| เวลาต่อครั้ง (`ns/op`) | 18,691,680 ns | 453,507 ns | **เร็วขึ้นประมาณ 41 เท่า** |
| หน่วยความจำต่อครั้ง (`B/op`) | 742,185 B | 502,181 B | ลดลงประมาณ 32% |
| จำนวนครั้งที่ allocate (`allocs/op`) | 64 | 49 | ลดลงเล็กน้อย |

**บทเรียนสำคัญจากตัวอย่างนี้**: ความแตกต่างของความซับซ้อนเชิงเวลา (**time complexity**) — จาก O(n²) ของการค้นหาเชิงเส้นซ้ำๆ ไปเป็น O(n) ของการใช้ map — ส่งผลต่อประสิทธิภาพจริงมากกว่าการ micro-optimize เล็กๆ น้อยๆ มหาศาล และที่สำคัญยิ่งกว่าคือ **เราไม่ได้เดาว่าปัญหาอยู่ตรงไหน** แต่ใช้ `go tool pprof -list` ชี้เป้าไปที่บรรทัด 19-20 อย่างแม่นยำก่อนตัดสินใจแก้ไข ซึ่งเป็นวิธีการทำงานที่ถูกต้องเสมอเมื่อต้อง optimize โค้ดจริง:

> **กระบวนการที่ถูกต้อง**: (1) เขียน test ยืนยันความถูกต้องก่อนเสมอ (2) เขียน benchmark วัด baseline (3) ใช้ profiling หา hotspot จริง อย่าเดา (4) แก้เฉพาะจุดที่ profiling ชี้ (5) รัน test อีกครั้งยืนยันว่ายังถูกต้อง (6) รัน benchmark เทียบผลก่อน-หลังด้วยตัวเลขจริง — ข้ามขั้นตอนไหนไปก็เสี่ยงทั้ง optimize ผิดจุด (แก้จุดที่ไม่ใช่ปัญหาจริง) หรือแก้แล้วโค้ดพัง (ไม่มี test ยืนยัน) ทั้งสิ้น

การเจาะลึกเรื่อง benchmark ขั้นสูงกว่านี้ (เช่น `b.RunParallel`, การเปรียบเทียบ benchmark หลายรุ่นด้วย `benchstat`) จะอยู่ใน **Part 084: Benchmarking ขั้นสูง** ที่ต่อยอดจากบทนี้โดยตรง

---

## สรุปสิ่งที่ได้เรียนในบทนี้

- **Profiling ตอบคำถาม "เวลา/หน่วยความจำไปอยู่ที่ไหน"** ซึ่งต่างจาก testing (ถูกต้องไหม) และ benchmark (เร็วแค่ไหนโดยรวม) — เป็นเครื่องมือชี้เป้าหา **hotspot** ที่แท้จริงในโค้ด
- **CPU profiling** เก็บได้ผ่าน `go test -bench=... -cpuprofile=cpu.prof` (สำหรับ benchmark) หรือผ่าน `net/http/pprof` (สำหรับเซิร์ฟเวอร์ที่กำลังรันจริง)
- **`go tool pprof -top`** แสดงฟังก์ชันที่กิน CPU/memory มากที่สุด อ่านคอลัมน์ `flat` (เวลาในฟังก์ชันเอง) กับ `cum` (เวลารวมฟังก์ชันย่อยด้วย) ให้เป็น
- **`go tool pprof -list=<ฟังก์ชัน>`** เจาะลึกลงไปที่ระดับบรรทัด ชี้เป้าได้แม่นยำกว่า `-top` มาก — พิสูจน์แล้วว่าชี้ไปที่บรรทัด linear search ได้อย่างแม่นยำในตัวอย่างจริง
- **Flame Graph** (ผ่าน `go tool pprof -http`) แสดงผลเชิงกราฟิก แกน X คือสัดส่วนเวลา แกน Y คือความลึกของ call stack — ใช้ text mode (`-top`/`-list`) แทนได้เมื่อไม่มีเบราว์เซอร์
- **Memory profiling** ด้วย `-memprofile` และ `go tool pprof -alloc_space` ชี้จุดที่ allocate หน่วยความจำมากที่สุด — มักเกิดจาก slice/map ที่ขยายขนาดซ้ำๆ โดยไม่ได้จอง capacity ล่วงหน้า
- **`net/http/pprof`** เปิดใช้งานด้วยการ `import _ "net/http/pprof"` เพียงบรรทัดเดียว ลงทะเบียน endpoint `/debug/pprof/*` ให้อัตโนมัติ — ต้องระวังไม่เปิดพอร์ตนี้สู่อินเทอร์เน็ตสาธารณะโดยตรง
- **กระบวนการ optimize ที่ถูกต้อง**: ทดสอบความถูกต้องก่อน → benchmark วัด baseline → profiling หา hotspot จริง → แก้เฉพาะจุดนั้น → ทดสอบซ้ำ → benchmark เทียบผล — พิสูจน์แล้วว่าการเปลี่ยนจาก O(n²) เป็น O(n) ทำให้เร็วขึ้นถึง **41 เท่า** ด้วยตัวเลขจริงจากการรันบนเครื่องจริง

## แบบฝึกหัดท้ายบท

1. เขียนฟังก์ชันของตัวเองที่จงใจไม่มีประสิทธิภาพ (เช่น ฟังก์ชันเช็คว่า slice มีสมาชิกซ้ำหรือไม่ด้วย nested loop O(n²)) เขียน benchmark แล้วเก็บ CPU profile วิเคราะห์ด้วย `go tool pprof -top` และ `-list` ยืนยันว่า hotspot อยู่ตรงจุดที่คาดไว้จริง
2. แก้ฟังก์ชันจากข้อ 1 ให้ใช้ map แทน nested loop แล้ววัดผลก่อน-หลังด้วย benchmark เทียบกับตัวอย่างในบทนี้
3. เขียนเซิร์ฟเวอร์ HTTP เล็กๆ ที่ import `net/http/pprof` แล้วรัน จากนั้นเปิด `/debug/pprof/goroutine?debug=1` ดูรายชื่อ goroutine ที่กำลังทำงานอยู่ ลองสร้าง goroutine ที่ค้างอยู่ตลอดไปโดยตั้งใจ (เช่น `go func() { select {} }()`) แล้วสังเกตว่ามันปรากฏใน endpoint นี้หรือไม่
4. ทดลองใช้ `go tool pprof -alloc_objects` แทน `-alloc_space` กับ memory profile ในหัวข้อ 5 สังเกตว่าผลลัพธ์ต่างกันอย่างไร (คำใบ้: `-alloc_space` วัดเป็น byte, `-alloc_objects` วัดเป็น "จำนวนครั้ง" ของการ allocate)
5. ถ้าเครื่องมี GUI ให้ลองรัน `go tool pprof -http=:8080` กับ profile ที่เก็บไว้ แล้วสำรวจมุมมอง Flame Graph, Graph, และ Source เปรียบเทียบว่าแต่ละมุมมองเหมาะกับการวิเคราะห์แบบไหน
6. ค้นคว้าเพิ่มเติมเกี่ยวกับ **block profile** และ **mutex profile** (`runtime.SetBlockProfileRate`, `runtime.SetMutexProfileFraction`) ที่ใช้หาจุดที่ goroutine รอ (block) กันนานที่สุด อธิบายด้วยคำพูดตัวเองว่าเหมาะกับสถานการณ์แบบไหนที่ CPU/memory profile ตอบไม่ได้ (เชื่อมโยงกับ `sync.Mutex` และ channel ที่เรียนใน **Part 037-039**)

---

**ต่อไป**: [Part 084 — Benchmarking ขั้นสูง](./084-benchmarking-advanced.md)
