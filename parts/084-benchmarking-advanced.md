# Part 084: Benchmarking ขั้นสูง

> ภาคที่ 7: Testing, Tooling, Performance — ตอนที่ 6 จาก 9 (Part 79–87)

## สารบัญของบทนี้

1. ทบทวน Benchmark พื้นฐานจาก Part 034
2. Sub-benchmark ด้วย `b.Run`: เบนช์มาร์กแบบพารามิเตอร์
3. `b.ReportAllocs()` และ `-benchmem` เจาะลึก: ทำไม allocation สำคัญกว่า `ns/op`
4. เปรียบเทียบผลก่อน-หลังด้วย `benchstat`
5. `b.RunParallel` เจาะลึก: เบนช์มาร์กโค้ด concurrent
6. กับดักของ Benchmark ที่ต้องระวัง: compiler inlining, CPU frequency scaling, `-count=N`
7. สรุปสิ่งที่ได้เรียนในบทนี้
8. แบบฝึกหัดท้ายบท

---

## 1. ทบทวน Benchmark พื้นฐานจาก Part 034

ใน **Part 034** เราเรียนพื้นฐานของ benchmark ใน Go ไปแล้ว: ฟังก์ชันที่ขึ้นต้นด้วย `Benchmark` รับ `*testing.B`, ตัวแปร `b.N` ที่ framework ปรับให้เองเพื่อความแม่นยำทางสถิติ, การรันด้วย `go test -bench=. -benchmem -run=^$`, กับดัก dead code elimination ที่ป้องกันด้วยตัวแปร `sink` ระดับ package, และ `b.ResetTimer()` สำหรับตัดเวลาการเตรียมข้อมูลออกจากผลวัด

บทนี้จะไม่สอนพื้นฐานเหล่านั้นซ้ำ แต่จะพาไปสู่ระดับที่ทีมวิศวกรมืออาชีพใช้จริงในการตัดสินใจเรื่อง performance: การเขียน benchmark ให้ครอบคลุมหลายขนาด input ในตัวเดียว การอ่านตัวเลข allocation ให้ลึกกว่าผิวเผิน การเปรียบเทียบผลลัพธ์ "ก่อน" กับ "หลัง" แก้โค้ดด้วยเครื่องมือทางสถิติจริง (ไม่ใช่แค่มองตาเทียบตัวเลข) และการเบนช์มาร์กโค้ด concurrent อย่างถูกต้อง

ก่อนเริ่ม ให้ทบทวนโครงสร้างมาตรฐานของ benchmark อีกครั้งสั้นๆ:

```go
func BenchmarkXxx(b *testing.B) {
	// เตรียมข้อมูล (ควรเรียก b.ResetTimer() หลังจากส่วนนี้ถ้าใช้เวลานาน)
	for i := 0; i < b.N; i++ {
		// โค้ดที่ต้องการวัด
	}
}
```

---

## 2. Sub-benchmark ด้วย `b.Run`: เบนช์มาร์กแบบพารามิเตอร์

ปัญหาที่พบบ่อยมากในทางปฏิบัติคือ: ฟังก์ชันตัวหนึ่งอาจมีพฤติกรรมด้าน performance ที่ต่างกันมากขึ้นอยู่กับ**ขนาดของ input** เช่น อัลกอริทึมที่เร็วกับข้อมูลน้อยแต่ช้าลงแบบไม่เป็นเชิงเส้นเมื่อข้อมูลมากขึ้น ถ้าเขียน benchmark แยกฟังก์ชันสำหรับแต่ละขนาด (`BenchmarkFoo10`, `BenchmarkFoo100`, `BenchmarkFoo1000`, ...) จะซ้ำซ้อนมาก — วิธีที่ถูกต้องคือใช้ **`b.Run`** สร้าง **sub-benchmark** เหมือนกับที่ `t.Run` สร้าง subtest ในบทที่ 034 นั่นเอง หลักการเดียวกับ table-driven test ทุกประการ เพียงแต่ใช้กับ benchmark

มาดูตัวอย่าง: เปรียบเทียบการค้นหาค่าใน slice ด้วย **linear search** กับการค้นหาผ่าน **map** ที่ต้องสร้างขึ้นมาก่อน

```go
// search.go
package strutil

// ContainsLinear ค้นหาค่า target ใน slice ด้วยการวนลูปตรงไปตรงมา (linear search)
func ContainsLinear(items []int, target int) bool {
	for _, v := range items {
		if v == target {
			return true
		}
	}
	return false
}

// ContainsMap ค้นหาค่า target โดยสร้าง map ของ items ก่อน แล้วเช็คด้วย O(1) lookup
// (สร้าง map ใหม่ทุกครั้งที่เรียก เพื่อจำลองสถานการณ์ "ค้นหาครั้งเดียว" อย่างยุติธรรม
// เทียบกับ ContainsLinear — ถ้าค้นหาซ้ำหลายครั้งกับ items ชุดเดิม ควรสร้าง map ไว้ล่วงหน้าครั้งเดียว)
func ContainsMap(items []int, target int) bool {
	set := make(map[int]struct{}, len(items))
	for _, v := range items {
		set[v] = struct{}{}
	}
	_, ok := set[target]
	return ok
}

func genInts(n int) []int {
	items := make([]int, n)
	for i := 0; i < n; i++ {
		items[i] = i
	}
	return items
}
```

```go
// search_bench_test.go
package strutil

import (
	"strconv"
	"testing"
)

var sinkBool bool

// BenchmarkContainsLinear ใช้ b.Run สร้าง sub-benchmark สำหรับแต่ละขนาด input
// เพื่อดูว่าเวลาที่ใช้เพิ่มขึ้นอย่างไรเมื่อขนาด slice โตขึ้น (เชิงเส้นตามที่คาดหวังจาก linear search)
func BenchmarkContainsLinear(b *testing.B) {
	sizes := []int{10, 100, 1000, 10000}
	for _, n := range sizes {
		items := genInts(n)
		target := n - 1 // กรณีแย่ที่สุด: ค่าที่ค้นหาอยู่ท้ายสุด บังคับให้วนครบทุกตัว
		b.Run("n="+strconv.Itoa(n), func(b *testing.B) {
			b.ReportAllocs()
			var r bool
			for i := 0; i < b.N; i++ {
				r = ContainsLinear(items, target)
			}
			sinkBool = r
		})
	}
}

// BenchmarkContainsMap ทำแบบเดียวกันแต่ใช้ ContainsMap เพื่อเทียบกันในตารางเดียว
func BenchmarkContainsMap(b *testing.B) {
	sizes := []int{10, 100, 1000, 10000}
	for _, n := range sizes {
		items := genInts(n)
		target := n - 1
		b.Run("n="+strconv.Itoa(n), func(b *testing.B) {
			b.ReportAllocs()
			var r bool
			for i := 0; i < b.N; i++ {
				r = ContainsMap(items, target)
			}
			sinkBool = r
		})
	}
}
```

จุดสำคัญคือ **`b.Run` เรียกก่อนเข้าลูป `b.N`** — การเตรียมข้อมูล (`genInts(n)`) ทำครั้งเดียวนอก sub-benchmark ต่อหนึ่งขนาด แล้วค่อยส่งเข้าไปให้ closure ของ sub-benchmark ใช้ ทำให้เวลาเตรียมข้อมูลไม่ปนเข้าไปในผลวัดของแต่ละขนาด

รันด้วยคำสั่งเดิมที่คุ้นเคยจาก Part 034:

```bash
go test -bench=. -benchmem -run=^$ ./...
```

ผลลัพธ์จริงจากการทดสอบ:

```
goos: linux
goarch: amd64
pkg: strutil
cpu: Intel(R) Xeon(R) Processor @ 2.80GHz
BenchmarkContainsLinear/n=10-4         	158807034	         8.089 ns/op	       0 B/op	       0 allocs/op
BenchmarkContainsLinear/n=100-4        	16884550	        71.88 ns/op	       0 B/op	       0 allocs/op
BenchmarkContainsLinear/n=1000-4       	 1832236	       676.0 ns/op	       0 B/op	       0 allocs/op
BenchmarkContainsLinear/n=10000-4      	  183751	      6947 ns/op	       0 B/op	       0 allocs/op
BenchmarkContainsMap/n=10-4            	 2466181	       453.0 ns/op	     328 B/op	       3 allocs/op
BenchmarkContainsMap/n=100-4           	  449394	      2875 ns/op	    2344 B/op	       3 allocs/op
BenchmarkContainsMap/n=1000-4          	   38199	     35943 ns/op	   36944 B/op	       5 allocs/op
BenchmarkContainsMap/n=10000-4         	    3016	    471639 ns/op	  295553 B/op	      33 allocs/op
PASS
```

### อ่านผลลัพธ์อย่างซื่อสัตย์ — บทเรียนสำคัญกว่าตัวเลข

ผลลัพธ์นี้อาจทำให้แปลกใจ: **`ContainsLinear` เร็วกว่า `ContainsMap` ในทุกขนาดที่ทดสอบ** ทั้งที่ทฤษฎีบอกว่า map lookup เป็น O(1) เร็วกว่า linear search ที่เป็น O(n) เสมอ — เหตุผลคือ **`ContainsMap` ในตัวอย่างนี้นับรวมต้นทุนการสร้าง map ใหม่ทุกครั้งที่เรียก** (ตั้งใจออกแบบให้เป็นแบบนี้เพื่อจำลองสถานการณ์ "ค้นหาครั้งเดียว") ต้นทุนของการสร้าง map และ hash ทุก element ในนั้นแพงกว่าการวนลูปเปรียบเทียบตรงๆ มาก โดยเฉพาะเมื่อค้นหาแค่ครั้งเดียว

นี่คือบทเรียนที่สำคัญกว่าตัวเลขเองเสียอีก: **"map เร็วกว่า array" เป็นความจริงก็ต่อเมื่อสร้าง map ไว้แล้วค้นหาซ้ำหลายครั้ง** ถ้าต้องสร้างใหม่ทุกครั้งที่ค้นหา linear search อาจเร็วกว่าจริงสำหรับข้อมูลขนาดเล็กถึงกลาง — sub-benchmark ที่ครอบคลุมหลายขนาดแบบนี้ช่วยให้เห็น**แนวโน้มที่แท้จริง**แทนที่จะเชื่อสัญชาตญาณหรือ Big-O เพียงอย่างเดียว ซึ่งตรงกับหลักการที่เราย้ำมาตลอดตั้งแต่ Part 034: **วัดผลจริง อย่าเดา**

---

## 3. `b.ReportAllocs()` และ `-benchmem` เจาะลึก: ทำไม allocation สำคัญกว่า `ns/op`

Part 034 แนะนำให้ใช้ `-benchmem` เพื่อดู `B/op` และ `allocs/op` ควบคู่กับ `ns/op` ไปแล้ว บทนี้จะอธิบายว่า**ทำไมตัวเลข allocation ถึงสำคัญยิ่งกว่า `ns/op` ในหลายสถานการณ์** โดยเฉพาะระบบที่รันต่อเนื่องยาวนาน (long-running server)

`b.ReportAllocs()` ที่เห็นในตัวอย่างข้างบนคือการสั่งให้ **sub-benchmark ตัวนั้นโดยเฉพาะ** รายงานข้อมูล allocation เสมอ แม้จะไม่ได้ส่ง flag `-benchmem` ตอนรันก็ตาม — มีประโยชน์เมื่อต้องการบังคับดูข้อมูล allocation ของ benchmark บางตัวโดยไม่ต้องพึ่ง flag ภายนอก (เหมาะกับการเขียน benchmark ที่ตั้งใจให้ทีมอื่นรันโดยไม่ลืมเปิด `-benchmem`)

### ทำไม allocation ถึงสำคัญกว่าที่คิด

ตัวเลข `ns/op` บอกแค่ **"ความเร็วของการรันหนึ่งครั้ง ตอนนี้ ณ ขณะที่วัด"** แต่ไม่ได้บอกอะไรเกี่ยวกับ**ผลกระทบสะสม**ที่มีต่อระบบเมื่อรันซ้ำเป็นล้านๆ ครั้งต่อวินาทีในระบบจริง ทุกครั้งที่มี heap allocation เกิดขึ้น สิ่งที่ตามมาคือ:

1. **ภาระของ Garbage Collector เพิ่มขึ้น** — ยิ่ง allocate เยอะ ยิ่งทำให้ heap โตเร็ว กระตุ้นให้ GC ทำงานถี่ขึ้น (ทบทวนกลไก GC และ `GOGC` จาก **Part 055**) แม้ Go GC จะออกแบบมาให้ pause time สั้นมาก แต่ GC ก็ยังกิน **CPU cycle** ไปจากงานจริงของโปรแกรมอยู่ดี ยิ่ง allocate ถี่ ยิ่งเสีย CPU ไปกับงาน "เก็บกวาด" มากขึ้นตามสัดส่วน
2. **Cache locality แย่ลง** — object ที่กระจายอยู่บน heap มักทำให้ CPU cache miss บ่อยกว่าข้อมูลที่อยู่ติดกันบน stack หรือใน array เดียวกัน
3. **ผลสะสมที่ `ns/op` มองไม่เห็น** — benchmark วัด "การรันแยกกันหนึ่งครั้ง" แต่ในระบบจริงที่มีการเรียกฟังก์ชันซ้ำๆ พร้อมกันหลาย goroutine allocation ที่ดูเหมือนเล็กน้อยต่อครั้งอาจกลายเป็นภาระ GC ระดับ production ที่วัดได้จริงจาก latency ที่เพิ่มขึ้นเป็นระยะ (GC-induced latency spike)

### กฎการอ่านผลลัพธ์ที่ควรยึดถือ

เมื่อเปรียบเทียบสอง implementation ที่ทำงานเหมือนกัน ให้พิจารณาทั้งสามค่าประกอบกันเสมอ ไม่ใช่แค่ `ns/op`:

| สถานการณ์ | ควรเลือกอะไร |
|---|---|
| A เร็วกว่า B มาก และ allocate น้อยกว่าด้วย | เลือก A ไม่ต้องคิดมาก |
| A เร็วกว่า B เล็กน้อย แต่ allocate มากกว่าเยอะ | ต้องพิจารณาบริบท — ถ้าเป็น hot path ที่เรียกบ่อยมาก B (allocate น้อยกว่า) อาจดีกว่าในระยะยาวเพราะกด GC pressure ต่ำกว่า |
| A ช้ากว่า B แต่ allocate น้อยกว่ามาก | ขึ้นกับว่าโค้ดนี้อยู่ใน hot path ที่ถูกเรียกความถี่สูงแค่ไหน — ถ้าใช่ ให้ชั่งน้ำหนักไปทาง allocation ต่ำ |

เราจะเห็นตัวอย่างที่เป็นรูปธรรมของการชั่งน้ำหนักนี้ในหัวข้อถัดไป ที่ใช้ `benchstat` เปรียบเทียบสอง implementation อย่างเป็นระบบ และในบทหน้า (**Part 085: Memory Management**) ที่จะเจาะลึกกลยุทธ์การลด allocation อย่างเต็มรูปแบบ

---

## 4. เปรียบเทียบผลก่อน-หลังด้วย `benchstat`

จนถึงตอนนี้เราเปรียบเทียบผล benchmark ด้วยการ**มองตา**เทียบตัวเลขสองชุด ซึ่งมีปัญหาสำคัญ: **ผลของ benchmark มีความแปรปรวนตามธรรมชาติ** (จะอธิบายละเอียดในหัวข้อ 6) ถ้ารันครั้งเดียวแล้วเห็นตัวเลขต่างกัน 5% เราไม่มีทางรู้เลยว่านั่นคือ**ความต่างจริง**จากการแก้โค้ด หรือเป็นแค่ **สัญญาณรบกวน (noise)** จากการรันคนละครั้ง

เครื่องมือมาตรฐานที่ทีม Go เองใช้แก้ปัญหานี้คือ **`benchstat`** ซึ่งรับผล benchmark หลายๆ รอบ (ผ่าน flag `-count`) มาคำนวณค่าสถิติ (median, confidence interval) แล้วบอกว่าความต่างที่เห็นนั้น **"มีนัยสำคัญทางสถิติจริงหรือไม่"**

### ติดตั้ง benchstat

```bash
go install golang.org/x/perf/cmd/benchstat@latest
```

ผลการติดตั้งจริงบนเครื่องที่ใช้เขียนบทเรียนนี้:

```
go: downloading golang.org/x/perf v0.0.0-20260908200009-22c9c6c9d4da
go: golang.org/x/perf@v0.0.0-20260908200009-22c9c6c9d4da requires go >= 1.26.0; switching to go1.26.8
go: downloading github.com/aclements/go-moremath v0.0.0-20210112150236-f10218a38794
```

> **หมายเหตุ**: `go install` สามารถดาวน์โหลด **Go toolchain เวอร์ชันใหม่กว่าที่ติดตั้งอยู่มาใช้ชั่วคราวโดยอัตโนมัติ** (ฟีเจอร์ `GOTOOLCHAIN=auto` ที่มีมาตั้งแต่ Go 1.21) เมื่อ dependency ของเครื่องมือที่กำลังติดตั้งต้องการเวอร์ชันใหม่กว่า ซึ่งไม่กระทบ Go เวอร์ชันหลักที่ใช้ compile โปรเจกต์ของเราเองเลย — `benchstat` ที่ได้จะถูกวางไว้ที่ `$(go env GOPATH)/bin/benchstat` ตามปกติ (ทบทวน `GOPATH`/`GOBIN` จาก **Part 001**)

ตรวจสอบว่าติดตั้งสำเร็จ:

```bash
benchstat -h
```

### ตัวอย่างจริง: เปรียบเทียบ `Dedup` สองเวอร์ชัน

มาดูสถานการณ์ที่พบบ่อยมากในงานจริง: มีฟังก์ชัน `Dedup` (ตัดค่าซ้ำออกจาก slice) เวอร์ชัน**เก่า**ที่เขียนแบบ naive (`O(n²)`) แล้วเราจะ optimize เป็นเวอร์ชัน**ใหม่**ที่ใช้ map (`O(n)`) — และอยากรู้ว่าการเปลี่ยนแปลงนี้คุ้มค่าจริงหรือไม่ในทุกมิติ (ไม่ใช่แค่ความเร็ว)

**เวอร์ชันเก่า** (`dedup.go`):

```go
package dedup

// Dedup ตัดค่าที่ซ้ำออกจาก slice โดยรักษาลำดับเดิมของค่าที่พบครั้งแรกไว้
// เวอร์ชันนี้คือ "ของเก่า" (naive): เช็คค่าซ้ำด้วยการวนลูปย้อนหลังทุกครั้ง -- เป็น O(n^2)
func Dedup(items []int) []int {
	result := make([]int, 0, len(items))
	for _, v := range items {
		found := false
		for _, r := range result {
			if r == v {
				found = true
				break
			}
		}
		if !found {
			result = append(result, v)
		}
	}
	return result
}
```

```go
// dedup_bench_test.go
package dedup

import (
	"math/rand"
	"testing"
)

var sinkInts []int

func genDupInts(n int) []int {
	items := make([]int, n)
	for i := range items {
		items[i] = rand.Intn(n / 2) // สุ่มค่าให้มีค่าซ้ำเยอะพอสมควร
	}
	return items
}

func BenchmarkDedup(b *testing.B) {
	items := genDupInts(2000)
	b.ReportAllocs()
	b.ResetTimer()
	var r []int
	for i := 0; i < b.N; i++ {
		r = Dedup(items)
	}
	sinkInts = r
}
```

รันเบนช์มาร์กของเวอร์ชันเก่าด้วย **`-count=10`** (รันซ้ำ 10 รอบเต็มเพื่อเก็บสถิติ — เหตุผลเรื่องจำนวนรอบจะอธิบายในหัวข้อ 6) แล้วเก็บผลลงไฟล์:

```bash
go test -bench=. -benchmem -run=^$ -count=10 ./... > old.txt
```

จากนั้นแก้ `Dedup` เป็น**เวอร์ชันใหม่**ที่ใช้ map:

```go
// Dedup ตัดค่าที่ซ้ำออกจาก slice โดยรักษาลำดับเดิมของค่าที่พบครั้งแรกไว้
// เวอร์ชันนี้คือ "ของใหม่": ใช้ map เก็บค่าที่เจอแล้ว ทำให้เช็คค่าซ้ำเป็น O(1) ต่อครั้ง -- รวมเป็น O(n)
func Dedup(items []int) []int {
	seen := make(map[int]struct{}, len(items))
	result := make([]int, 0, len(items))
	for _, v := range items {
		if _, ok := seen[v]; ok {
			continue
		}
		seen[v] = struct{}{}
		result = append(result, v)
	}
	return result
}
```

รันเบนช์มาร์กอีกครั้งด้วยคำสั่งเดิม แล้วเก็บลงไฟล์ใหม่:

```bash
go test -bench=. -benchmem -run=^$ -count=10 ./... > new.txt
```

สุดท้าย ใช้ `benchstat` เปรียบเทียบสองไฟล์:

```bash
benchstat old.txt new.txt
```

ผลลัพธ์จริง:

```
goos: linux
goarch: amd64
pkg: p084bench
cpu: Intel(R) Xeon(R) Processor @ 2.80GHz
        │   old.txt    │               new.txt                │
        │    sec/op    │    sec/op     vs base                │
Dedup-4   349.1µ ± 11%   108.6µ ± 29%  -68.88% (p=0.000 n=10)

        │   old.txt    │                new.txt                │
        │     B/op     │     B/op      vs base                 │
Dedup-4   16.00Ki ± 0%   88.16Ki ± 0%  +450.98% (p=0.000 n=10)

        │  old.txt   │               new.txt                │
        │ allocs/op  │  allocs/op   vs base                 │
Dedup-4   1.000 ± 0%   10.000 ± 0%  +900.00% (p=0.000 n=10)
```

### อ่านผลลัพธ์ของ benchstat

- **`-68.88% (p=0.000 n=10)`** ในตาราง `sec/op`: เวอร์ชันใหม่**เร็วกว่าเวอร์ชันเก่าเกือบ 69%** — และค่า **`p=0.000`** (p-value) บอกว่าความต่างนี้**มีนัยสำคัญทางสถิติจริง** ไม่ใช่ noise (โดยทั่วไปถือว่า `p < 0.05` คือมีนัยสำคัญ ยิ่งค่าน้อยยิ่งมั่นใจได้มาก) ถ้า `benchstat` เห็นว่าความต่างไม่มีนัยสำคัญทางสถิติ มันจะแสดงเครื่องหมาย **`~`** แทนเปอร์เซ็นต์ ให้ตีความว่า **"ยังสรุปไม่ได้ว่าต่างจริงหรือแค่ noise"**
- **`± 11%`** และ **`± 29%`**: นี่คือ**ความแปรปรวน (variability)** ของผลวัดทั้ง 10 รอบ — สังเกตว่าเวอร์ชันใหม่มีความแปรปรวนสูงกว่า (29% เทียบกับ 11%) ซึ่งสมเหตุสมผล เพราะการทำงานกับ map เกี่ยวข้องกับ hashing และ bucket ที่มีพฤติกรรมไม่แน่นอนกว่าการวนลูปธรรมดา
- **ตาราง `B/op` และ `allocs/op` เล่าเรื่องที่ตรงข้ามกันโดยสิ้นเชิง**: เวอร์ชันใหม่ใช้ memory **มากกว่าเดิมถึง 450%** และ allocate **มากกว่าเดิมถึง 900%** (จาก 1 ครั้งเป็น 10 ครั้งต่อการเรียก) เพราะการสร้างและขยาย map ภายในต้องมีการ allocate หลายจุด (bucket array, overflow bucket) ต่างจาก slice เดิมที่ allocate ก้อนเดียวจบ

### บทเรียนที่สำคัญที่สุดของหัวข้อนี้

นี่คือตัวอย่างที่สมบูรณ์แบบของสิ่งที่พูดถึงในหัวข้อ 3: **ตัวเลขทั้งสามด้าน (เวลา, memory, allocation) อาจขัดแย้งกันเอง** เวอร์ชันใหม่เร็วกว่ามากในแง่เวลา แต่แลกมาด้วย allocation ที่สูงกว่ามาก — การตัดสินใจว่าจะใช้เวอร์ชันไหนขึ้นอยู่กับบริบทจริง: ถ้าฟังก์ชันนี้ถูกเรียกไม่บ่อยแต่ทำงานกับข้อมูลจำนวนมาก (ที่ `O(n²)` จะช้ามากจนรับไม่ได้) เวอร์ชันใหม่คือตัวเลือกที่ถูกต้องชัดเจน แต่ถ้าฟังก์ชันนี้ถูกเรียกถี่มากในระบบที่ latency สำคัญกว่าความเร็วดิบ (เช่น ระบบที่ GC pressure เป็นปัญหาอยู่แล้ว) การแลก allocation ที่สูงขึ้น 9 เท่าอาจไม่คุ้มค่า — **`benchstat` ให้ข้อมูลที่ครบถ้วนพอให้ตัดสินใจอย่างมีเหตุผล แทนที่จะเดา**

> **แนวทางปฏิบัติในทีมจริง**: หลายทีมนำ `benchstat` ไปผูกกับ CI pipeline (จะเรียนเรื่อง CI/CD เต็มรูปแบบใน **Part 098**) เพื่อ**ป้องกัน performance regression โดยอัตโนมัติ** — รัน benchmark บน commit ปัจจุบันเทียบกับ baseline ที่เก็บไว้ ถ้าพบว่าช้าลงอย่างมีนัยสำคัญทางสถิติ (ไม่ใช่แค่ `~`) ก็แจ้งเตือนหรือ block การ merge ทันที

---

## 5. `b.RunParallel` เจาะลึก: เบนช์มาร์กโค้ด concurrent

ใน **Part 040** เราใช้ `b.RunParallel` เปรียบเทียบ `sync.Mutex` กับ `atomic.Int64` มาแล้วสั้นๆ สำหรับตัวนับธรรมดา บทนี้จะไปลึกกว่านั้น: เบนช์มาร์ก**สถานการณ์ผสม (mixed workload)** ที่สมจริงกว่ามาก — cache ที่มีทั้ง**อ่านและเขียน**ในสัดส่วนที่พบได้บ่อยในระบบจริง (อ่าน 95% เขียน 5%) เพื่อเปรียบเทียบ `sync.Mutex` กับ `sync.RWMutex`

ทบทวนสั้นๆ ว่า `b.RunParallel` ทำงานอย่างไร: มันสร้าง goroutine จำนวนหนึ่ง (ค่า default เท่ากับ `GOMAXPROCS`) แต่ละตัวรัน closure ที่เราส่งเข้าไปวนซ้ำผ่าน `pb.Next()` จนกว่า `b.N` รอบทั้งหมดจะถูกใช้ครบ — จำลองสถานการณ์ที่หลาย goroutine แข่งกันเข้าถึงทรัพยากรร่วมพร้อมกันจริงๆ ต่างจากการเรียกลูปธรรมดาที่วัดแค่ performance แบบ single-thread

```go
// cache.go
package cache

import "sync"

// MutexCache ป้องกัน map ด้วย sync.Mutex ตัวเดียว -- ทั้ง read และ write ต้องแย่ง lock เดียวกันหมด
type MutexCache struct {
	mu   sync.Mutex
	data map[string]int
}

func NewMutexCache() *MutexCache {
	return &MutexCache{data: map[string]int{"key": 1}}
}

func (c *MutexCache) Get(key string) int {
	c.mu.Lock()
	defer c.mu.Unlock()
	return c.data[key]
}

func (c *MutexCache) Set(key string, val int) {
	c.mu.Lock()
	defer c.mu.Unlock()
	c.data[key] = val
}

// RWMutexCache ใช้ sync.RWMutex แทน -- นักอ่านหลายคน "อ่านพร้อมกันได้" ไม่ต้องรอกัน
// ต้องรอเฉพาะตอนมีนักเขียนเท่านั้น เหมาะกับ workload ที่อ่านเยอะกว่าเขียนมาก
type RWMutexCache struct {
	mu   sync.RWMutex
	data map[string]int
}

func NewRWMutexCache() *RWMutexCache {
	return &RWMutexCache{data: map[string]int{"key": 1}}
}

func (c *RWMutexCache) Get(key string) int {
	c.mu.RLock()
	defer c.mu.RUnlock()
	return c.data[key]
}

func (c *RWMutexCache) Set(key string, val int) {
	c.mu.Lock()
	defer c.mu.Unlock()
	c.data[key] = val
}
```

```go
// cache_bench_test.go
package cache

import "testing"

var sinkVal int

// BenchmarkMutexCache_ReadHeavy จำลอง workload ที่อ่าน 95% เขียน 5% (สัดส่วนที่พบบ่อยในระบบจริง เช่น cache)
// ทุก goroutine ที่ b.RunParallel สร้างขึ้น (ตามจำนวน GOMAXPROCS โดย default) วนเรียก body พร้อมกัน
func BenchmarkMutexCache_ReadHeavy(b *testing.B) {
	c := NewMutexCache()
	b.ReportAllocs()
	b.RunParallel(func(pb *testing.PB) {
		i := 0
		for pb.Next() {
			i++
			if i%20 == 0 { // 1 ใน 20 ครั้ง (5%) เป็นการเขียน ที่เหลือเป็นการอ่าน
				c.Set("key", i)
			} else {
				sinkVal = c.Get("key")
			}
		}
	})
}

// BenchmarkRWMutexCache_ReadHeavy ทำ workload เดียวกันทุกประการ ต่างกันที่ synchronization primitive
func BenchmarkRWMutexCache_ReadHeavy(b *testing.B) {
	c := NewRWMutexCache()
	b.ReportAllocs()
	b.RunParallel(func(pb *testing.PB) {
		i := 0
		for pb.Next() {
			i++
			if i%20 == 0 {
				c.Set("key", i)
			} else {
				sinkVal = c.Get("key")
			}
		}
	})
}

// BenchmarkRWMutexCache_Parallelism4 สาธิต b.SetParallelism: บังคับจำนวน goroutine ต่อ CPU
// ค่า default ของ b.RunParallel คือ GOMAXPROCS goroutine; SetParallelism(p) คูณเข้าไปอีก
// เช่น SetParallelism(4) บนเครื่อง 4 core จะได้ 16 goroutine พร้อมกัน จำลอง contention ที่สูงกว่าปกติ
func BenchmarkRWMutexCache_Parallelism4(b *testing.B) {
	c := NewRWMutexCache()
	b.SetParallelism(4)
	b.ReportAllocs()
	b.RunParallel(func(pb *testing.PB) {
		i := 0
		for pb.Next() {
			i++
			if i%20 == 0 {
				c.Set("key", i)
			} else {
				sinkVal = c.Get("key")
			}
		}
	})
}
```

รันด้วยคำสั่งมาตรฐาน:

```bash
go test -bench=. -benchmem -run=^$ ./...
```

ผลลัพธ์จริง (รันบนเครื่อง 4 core):

```
goos: linux
goarch: amd64
pkg: cache
cpu: Intel(R) Xeon(R) Processor @ 2.80GHz
BenchmarkMutexCache_ReadHeavy-4        	10639588	       110.0 ns/op	       0 B/op	       0 allocs/op
BenchmarkRWMutexCache_ReadHeavy-4      	34308535	        37.06 ns/op	       0 B/op	       0 allocs/op
BenchmarkRWMutexCache_Parallelism4-4   	30303596	        40.86 ns/op	       0 B/op	       0 allocs/op
PASS
```

### วิเคราะห์ผลลัพธ์

**`sync.RWMutex` เร็วกว่า `sync.Mutex` เกือบ 3 เท่า** (37.06 ns/op เทียบกับ 110.0 ns/op) ในสถานการณ์อ่านหนัก 95% — เหตุผลตรงตามทฤษฎีที่เรียนใน **Part 039**: `RWMutex` อนุญาตให้**นักอ่านหลายคนถือ read lock พร้อมกันได้**โดยไม่ต้องรอกัน ต่อเมื่อมีนักเขียนเท่านั้นที่ต้องรอให้นักอ่านทั้งหมดปล่อย lock ก่อน ในขณะที่ `Mutex` ธรรมดาบังคับให้**ทุกการเข้าถึง ไม่ว่าจะอ่านหรือเขียน ต้องเข้าคิวทีละคนเสมอ** — เมื่อสัดส่วนการอ่านสูงมาก ความแตกต่างนี้ยิ่งเห็นชัด

ส่วน `BenchmarkRWMutexCache_Parallelism4` ที่ใช้ `b.SetParallelism(4)` (เพิ่มจำนวน goroutine เป็น 4 เท่าของ `GOMAXPROCS` คือ 16 goroutine บนเครื่อง 4 core) แสดงให้เห็นว่าเมื่อมีการแย่งชิง (contention) สูงขึ้นกว่าจำนวน core จริง ประสิทธิภาพลดลงเล็กน้อย (40.86 ns/op เทียบกับ 37.06 ns/op) — มีประโยชน์มากเวลาต้องการจำลองสถานการณ์ที่ระบบจริงมี concurrent request สูงกว่าจำนวน CPU core ที่มีอยู่จริง (เช่น web server ที่รับ request พร้อมกันหลายพันตัวบนเครื่องที่มีแค่ไม่กี่ core)

> **ทบทวนจาก Part 040**: ถ้าสิ่งที่ต้องป้องกันเป็นแค่ตัวแปรเดี่ยวๆ ตัวเดียว (counter, flag) `sync/atomic` ยังคงเร็วกว่า mutex ทุกแบบเสมอ — `RWMutex` เข้ามาเป็นตัวเลือกที่ดีกว่า `Mutex` ธรรมดาเมื่อต้อง**ป้องกันโครงสร้างข้อมูลที่ซับซ้อนกว่าตัวแปรเดี่ยว** (เช่น map หรือ struct หลาย field) ในสถานการณ์ที่อ่านเยอะกว่าเขียนมาก

---

## 6. กับดักของ Benchmark ที่ต้องระวัง: compiler inlining, CPU frequency scaling, `-count=N`

Part 034 พูดถึงกับดักเรื่อง **dead code elimination** ไปแล้ว หัวข้อนี้จะพาไปดูกับดักอื่นๆ ที่พบบ่อยไม่แพ้กัน โดยเฉพาะเมื่อเริ่มทำ benchmark ในระดับที่ต้องการความแม่นยำสูงสำหรับการตัดสินใจจริงในทีม

### Compiler Inlining: เมื่อ benchmark วัดโค้ดที่ไม่มีอยู่จริง

**Inlining** คือการที่ compiler แทนที่การเรียกฟังก์ชันด้วยเนื้อหาของฟังก์ชันนั้นโดยตรง (ไม่ต้องเสียเวลา call/return จริง) เป็นการ optimize มาตรฐานที่ทำกับฟังก์ชันเล็กๆ ที่เรียกบ่อย ปัญหาคือ **ถ้าฟังก์ชันที่ถูก benchmark ถูก inline เข้าไปในลูปของ benchmark เอง** ตัวเลขที่ได้อาจไม่สะท้อนต้นทุนจริงของการเรียกฟังก์ชันนั้นในโค้ดจริง (ที่อาจไม่ได้ถูก inline เพราะซับซ้อนกว่าหรือถูกเรียกผ่าน interface)

ตรวจสอบว่าฟังก์ชันไหนถูก inline ได้ด้วย escape analysis flag ที่เรียนใน **Part 055**:

```bash
go build -gcflags="-m -m" ./... 2>&1 | grep "can inline"
```

ถ้าเห็นข้อความ `can inline ContainsLinear` แปลว่า compiler มีสิทธิ์ inline ฟังก์ชันนี้เข้าไปในจุดที่เรียกใช้ — สำหรับ benchmark ที่ต้องการวัด**ต้นทุนที่แท้จริงของการเรียกฟังก์ชัน** (ไม่ใช่แค่เนื้อหาข้างในหลัง inline) สามารถปิด inline เฉพาะฟังก์ชันได้ด้วย compiler directive **`//go:noinline`**:

```go
//go:noinline
func ContainsLinear(items []int, target int) bool {
	// ...
}
```

> **ข้อควรระวัง**: อย่าใส่ `//go:noinline` พร่ำเพรื่อ — ควรใช้เฉพาะตอนที่ต้องการวัด "ต้นทุนของการเรียกฟังก์ชันแยกต่างหาก" อย่างจงใจ (เช่น เปรียบเทียบ dispatch cost ของ interface กับ concrete type) ในกรณีทั่วไป การปล่อยให้ compiler ตัดสินใจ inline เองตามปกติจะให้ตัวเลขที่ตรงกับพฤติกรรมจริงของโค้ด production มากกว่า เพราะโค้ด production ก็ถูก compiler optimize แบบเดียวกันอยู่แล้ว

### CPU Frequency Scaling และสัญญาณรบกวนอื่นๆ

CPU สมัยใหม่แทบทุกตัว (โดยเฉพาะบนเครื่อง laptop หรือ cloud VM ที่ใช้ร่วมกับ workload อื่น) มีกลไก **frequency scaling** (เช่น Intel Turbo Boost, cpufreq governor) ที่ปรับความเร็วสัญญาณนาฬิกาขึ้นลงตามอุณหภูมิและภาระงาน ทำให้ **ผลการรัน benchmark เดียวกันซ้ำๆ กันได้ตัวเลขไม่เท่ากันทุกครั้ง** แม้จะไม่ได้แก้โค้ดอะไรเลย — ปรากฏการณ์นี้เห็นได้ชัดจากตัวอย่างข้างล่าง

รัน benchmark เดิม (`ContainsLinear/n=10`) ซ้ำ 5 รอบด้วย `-count=5`:

```bash
go test -bench=ContainsLinear -benchmem -run=^$ -count=5 ./...
```

ผลลัพธ์จริง (เฉพาะขนาด `n=10`):

```
BenchmarkContainsLinear/n=10-4         	159367551	         7.361 ns/op	       0 B/op	       0 allocs/op
BenchmarkContainsLinear/n=10-4         	161811204	         7.437 ns/op	       0 B/op	       0 allocs/op
BenchmarkContainsLinear/n=10-4         	162208182	         7.329 ns/op	       0 B/op	       0 allocs/op
BenchmarkContainsLinear/n=10-4         	163524720	         7.711 ns/op	       0 B/op	       0 allocs/op
BenchmarkContainsLinear/n=10-4         	148233949	         7.445 ns/op	       0 B/op	       0 allocs/op
```

ทั้งที่เป็นการรันฟังก์ชันเดียวกันเป๊ะๆ ไม่มีการแก้โค้ดใดๆ ระหว่างรอบ ตัวเลข `ns/op` ยังแกว่งอยู่ในช่วง 7.33 - 7.71 (ความต่างเกือบ 5%) — นี่คือสัญญาณรบกวนตามธรรมชาติของระบบที่ต้องยอมรับและออกแบบวิธีวัดผลให้รองรับมัน ไม่ใช่ปัญหาของโค้ดหรือ benchmark เอง

### `-count=N` และทำไมต้อง ≥ 6 รอบสำหรับ `benchstat`

วิธีมาตรฐานในการรับมือกับความแปรปรวนนี้คือ**รัน benchmark หลายรอบแล้วดูค่าทางสถิติ** แทนที่จะเชื่อผลจากการรันครั้งเดียว — ลองใช้ `benchstat` วิเคราะห์ผลจาก `-count=5` ข้างต้น:

```bash
go test -bench=ContainsLinear -benchmem -run=^$ -count=5 ./... > count5.txt
benchstat count5.txt
```

ผลลัพธ์จริง:

```
                         │  count5.txt  │
                         │    sec/op    │
ContainsLinear/n=10-4      7.437n ± ∞ ¹
ContainsLinear/n=100-4     71.77n ± ∞ ¹
ContainsLinear/n=1000-4    633.9n ± ∞ ¹
ContainsLinear/n=10000-4   6.282µ ± ∞ ¹
geomean                    214.7n
¹ need >= 6 samples for confidence interval at level 0.95
```

`benchstat` **ปฏิเสธที่จะคำนวณช่วงความเชื่อมั่น (confidence interval)** ตรงๆ และแจ้งเตือนว่า **"need >= 6 samples"** — นี่คือเหตุผลที่ทีม Go เองแนะนำให้ใช้ **`-count=10`** เป็นค่ามาตรฐานเมื่อต้องการผลที่นำไปวิเคราะห์ทางสถิติจริงจัง ลองรันใหม่ด้วย `-count=10`:

```bash
go test -bench=ContainsLinear -benchmem -run=^$ -count=10 ./... > count10.txt
benchstat count10.txt
```

ผลลัพธ์จริง:

```
                         │ count10.txt │
                         │   sec/op    │
ContainsLinear/n=10-4      7.541n ± 3%
ContainsLinear/n=100-4     73.58n ± 5%
ContainsLinear/n=1000-4    632.2n ± 4%
ContainsLinear/n=10000-4   6.452µ ± 5%
geomean                    218.1n
```

คราวนี้ `benchstat` คำนวณช่วงความแปรปรวนได้แล้ว (± 3-5%) — ตัวเลขนี้คือ**ระดับ noise ตามธรรมชาติของเครื่องนี้** ซึ่งมีประโยชน์มาก: เมื่อเปรียบเทียบ "ก่อน-หลัง" แก้โค้ดในหัวข้อ 4 ถ้าเห็นความต่างที่**น้อยกว่าระดับ noise นี้** (เช่น ต่างกันแค่ 2-3%) ก็ไม่ควรสรุปว่าโค้ดที่แก้เร็วขึ้นจริง — นี่คือสิ่งที่ `p-value` ใน `benchstat` คำนวณให้อัตโนมัติอยู่แล้ว แต่การเข้าใจที่มาของมันช่วยให้ตีความผลลัพธ์ได้อย่างมั่นใจมากขึ้น

### สรุปกฎการรัน benchmark ที่น่าเชื่อถือ

1. **ใช้ `-count=10` เป็นค่ามาตรฐาน** เมื่อผลลัพธ์จะถูกนำไปตัดสินใจสำคัญ (ไม่ใช่แค่ดูผ่านๆ ระหว่างพัฒนา)
2. **ปิดโปรแกรมอื่นที่กินทรัพยากร CPU หนักๆ** ระหว่างรัน benchmark (browser, IDE ที่กำลัง index โค้ด ฯลฯ)
3. **หลีกเลี่ยงการรัน benchmark บน cloud VM ที่ใช้ CPU ร่วมกับ tenant อื่น** (shared/burstable instance) ถ้าเป็นไปได้ — ใช้เครื่อง dedicated หรืออย่างน้อย VM แบบ dedicated CPU
4. **ใช้ `benchstat` เปรียบเทียบเสมอแทนการมองตา** โดยเฉพาะเมื่อความต่างที่เห็นไม่ได้มากอย่างชัดเจน (เช่น ต่างกันไม่ถึง 20-30%)
5. **ระวัง dead code elimination** (จาก Part 034) **และ inlining** (จากหัวข้อนี้) ที่อาจทำให้ตัวเลขที่วัดไม่ตรงกับพฤติกรรมจริงของโค้ด

---

## สรุปสิ่งที่ได้เรียนในบทนี้

- **`b.Run`** สร้าง **sub-benchmark** ได้เหมือน `t.Run` สร้าง subtest — เหมาะมากสำหรับเปรียบเทียบพฤติกรรมของฟังก์ชันข้ามหลายขนาด input ในตารางเดียว
- **Allocation สำคัญกว่า `ns/op` ในหลายสถานการณ์** เพราะส่งผลต่อภาระของ Garbage Collector และ cache locality สะสมในระบบที่รันต่อเนื่อง — ต้องพิจารณาทั้ง `ns/op`, `B/op`, และ `allocs/op` ประกอบกันเสมอ ไม่ใช่แค่ตัวเดียว
- **`benchstat`** (ติดตั้งด้วย `go install golang.org/x/perf/cmd/benchstat@latest`) เปรียบเทียบผล benchmark สองไฟล์ (เก่า-ใหม่) พร้อมคำนวณ **p-value** บอกว่าความต่างมีนัยสำคัญทางสถิติจริงหรือไม่ (`~` หมายถึงยังสรุปไม่ได้)
- **`b.RunParallel`** จำลองสถานการณ์ concurrent จริง — `sync.RWMutex` เร็วกว่า `sync.Mutex` อย่างมีนัยสำคัญในสถานการณ์อ่านหนักเขียนเบา ส่วน **`b.SetParallelism(p)`** ใช้เพิ่ม contention ให้จำลองสถานการณ์ที่หนักกว่าจำนวน CPU core จริง
- **Compiler inlining** อาจทำให้ benchmark วัดโค้ดที่ไม่ตรงกับพฤติกรรมจริง — ใช้ `//go:noinline` ปิดเฉพาะจุดเมื่อจำเป็นต้องวัดต้นทุนการเรียกฟังก์ชันแยกต่างหาก
- **CPU frequency scaling และ noise ตามธรรมชาติ** ทำให้ผล benchmark แกว่งได้แม้รันโค้ดเดียวกันซ้ำ — ใช้ **`-count=10`** เป็นมาตรฐาน (ต้องการอย่างน้อย 6 รอบให้ `benchstat` คำนวณช่วงความเชื่อมั่นได้) แล้วเปรียบเทียบด้วย `benchstat` เสมอแทนการมองตา

## แบบฝึกหัดท้ายบท

1. เขียนฟังก์ชัน `Reverse(s string) string` สองแบบ (แบบใช้ `[]rune` กับแบบใช้ `[]byte`) แล้วเขียน benchmark แบบ sub-benchmark ด้วย `b.Run` ทดสอบกับ string ความยาว 10, 100, 1000 ตัวอักษร เปรียบเทียบผลด้วย `-benchmem`
2. ติดตั้ง `benchstat` บนเครื่องของตัวเอง แล้วทำตามตัวอย่าง `Dedup` ในหัวข้อ 4 ให้ครบทุกขั้นตอน (รัน old.txt, แก้โค้ด, รัน new.txt, เปรียบเทียบ) บันทึกผลลัพธ์จริงที่ได้บนเครื่องของตัวเอง
3. เขียน benchmark ด้วย `b.RunParallel` เปรียบเทียบ `map` ธรรมดาที่ป้องกันด้วย `sync.Mutex` กับ `sync.Map` (ค้นคว้าเพิ่มเติมว่า `sync.Map` คืออะไร เหมาะกับสถานการณ์แบบไหน) ในสถานการณ์ read-heavy เดียวกับหัวขัอ 5
4. ทดลองใส่ `//go:noinline` ให้ฟังก์ชัน `ContainsLinear` ในหัวข้อ 2 แล้วรัน benchmark เทียบกับตอนไม่ใส่ สังเกตว่าตัวเลขเปลี่ยนไปมากน้อยแค่ไหน (คำใบ้: ฟังก์ชันนี้อาจเล็กเกินกว่าจะเห็นความต่างชัดเจน ลองทำกับฟังก์ชันที่ซับซ้อนกว่านี้ดูด้วย)
5. รัน benchmark ตัวใดก็ได้จากบทนี้ด้วย `-count=1`, `-count=5`, และ `-count=10` แล้วเปรียบเทียบผลลัพธ์จาก `benchstat` ของแต่ละแบบ อธิบายว่าทำไม `-count=1` ถึงไม่เพียงพอสำหรับการตัดสินใจสำคัญ
6. ค้นคว้าเพิ่มเติมเกี่ยวกับ flag `-cpu` ของ `go test` (เช่น `-cpu=1,2,4`) ที่รัน benchmark เดิมซ้ำด้วยค่า `GOMAXPROCS` ต่างกัน ลองใช้กับ benchmark ในหัวข้อ 5 แล้ววิเคราะห์ว่าประสิทธิภาพเปลี่ยนไปอย่างไรเมื่อจำนวน CPU ที่อนุญาตให้ใช้เปลี่ยนไป

---

**ต่อไป**: [Part 085 — Memory Management และ Optimization](./085-memory-management.md)
