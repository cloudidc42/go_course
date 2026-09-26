# Part 085: Memory Management และ Optimization

> ภาคที่ 7: Testing, Tooling, Performance — ตอนที่ 7 จาก 9 (Part 79–87)

## สารบัญของบทนี้

1. ทบทวน Stack, Heap และ Escape Analysis จาก Part 010 และ Part 055
2. กลยุทธ์ที่ 1: ส่ง struct ขนาดเล็กแบบ value แทน pointer
3. กลยุทธ์ที่ 2: Pre-allocate slice ด้วย `make([]T, 0, n)` เมื่อรู้ขนาดล่วงหน้า
4. กลยุทธ์ที่ 3: ใช้ซ้ำหน่วยความจำด้วย `sync.Pool` (และกับดักเรื่อง interface boxing)
5. String-building ทบทวนจาก Part 019/050: `strings.Builder` กับ `Grow()`
6. หลีกเลี่ยง Interface Boxing โดยไม่จำเป็น
7. ผลกระทบของ GC Pause ต่อระบบจริง และเมื่อไรควรปรับ `GOGC`/`GOMEMLIMIT`
8. สรุปทุกกลยุทธ์เป็นกรณีศึกษาเดียว: วัดผลก่อน-หลังด้วย `testing.AllocsPerRun` และ `-benchmem`
9. Profile ก่อนเสมอ — อย่าเดา-optimize
10. สรุปสิ่งที่ได้เรียนในบทนี้
11. แบบฝึกหัดท้ายบท

---

## 1. ทบทวน Stack, Heap และ Escape Analysis จาก Part 010 และ Part 055

ก่อนเข้าเนื้อหาใหม่ ขอทบทวนสั้นๆ สิ่งที่เรียนไปแล้วสองจุดสำคัญ เพราะทุกกลยุทธ์ในบทนี้ตั้งอยู่บนความเข้าใจนี้ทั้งหมด:

- **Part 010 (Pointers)**: ตัวแปร local ใน Go ไม่ได้อยู่บน stack เสมอไป — compiler มีขั้นตอน **escape analysis** ที่วิเคราะห์ตอน compile time ว่าตัวแปรใดต้อง "รอด" อยู่นานกว่าฟังก์ชันที่สร้างมัน (เช่น ถูกคืนออกไปเป็น pointer หรือถูกส่งผ่าน interface) ถ้าใช่ ตัวแปรนั้นจะ **escape ไปอยู่บน heap** แทนที่จะเป็น stack
- **Part 055 (runtime และ GC)**: ทุกครั้งที่มีตัวแปร escape ไป heap หมายถึงมีการ **heap allocation** เกิดขึ้น และหน่วยความจำนั้นต้องรอ **Garbage Collector** มาเก็บกวาดในภายหลัง ยิ่ง allocate ถี่ ยิ่งเพิ่มภาระให้ GC (แม้ Go GC จะออกแบบมาให้ pause time ต่ำมากก็ตาม)

บทนี้จะไม่สอนแนวคิดเหล่านี้ซ้ำ แต่จะพาไปสู่ **กลยุทธ์รูปธรรมที่ใช้ลด heap allocation ได้จริงในโค้ดระดับ production** พร้อมพิสูจน์ทุกกลยุทธ์ด้วยตัวเลขจริงจาก `testing.AllocsPerRun` และ `-benchmem` (ตามที่เรียนหลักการอ่านตัวเลขเหล่านี้ไปแล้วใน Part 084) ไม่ใช่แค่พูดลอยๆ ว่า "แบบนี้เร็วกว่า"

### ทำไมต้องสนใจเรื่องนี้เป็นพิเศษ

ตามที่ Part 055 อธิบายไว้ Go เขียนโปรแกรมได้โดยไม่ต้องคิดเรื่อง memory เองในกรณีทั่วไป — **แต่สำหรับโค้ดที่อยู่ใน hot path** (ถูกเรียกความถี่สูงมาก เช่น handler ของ web server ที่รับ request หลายพัน request ต่อวินาที) การลด allocation ที่ไม่จำเป็นสามารถส่งผลจริงต่อทั้ง throughput และ latency ของทั้งระบบ เพราะ allocation ที่น้อยลงหมายถึง GC ทำงานถี่น้อยลง และ CPU cycle ที่เคยเสียไปกับการจัดสรร/เก็บกวาดหน่วยความจำ กลับมาเป็นของ business logic มากขึ้น

---

## 2. กลยุทธ์ที่ 1: ส่ง struct ขนาดเล็กแบบ value แทน pointer

สัญชาตญาณของโปรแกรมเมอร์จากภาษาอื่น (โดยเฉพาะที่มาจาก Java หรือ Python ที่ object ทุกตัวเป็น reference โดยปริยาย) มักจะใช้ pointer กับทุก struct โดยอัตโนมัติ เพราะรู้สึกว่า "pointer เบากว่า เพราะ copy แค่ 8 byte" แต่ในความเป็นจริง สำหรับ **struct ขนาดเล็ก** การใช้ pointer กลับมีต้นทุนซ่อนอยู่: มันบังคับให้ข้อมูลนั้น **escape ไป heap** และต้องมี allocation แยกต่างหากสำหรับทุก instance

มาดูตัวอย่างที่พิสูจน์เรื่องนี้ด้วยตัวเลขจริง:

```go
// points.go
package geometry

// Point คือพิกัด 2 มิติ - struct ขนาดเล็กมาก (16 byte บนระบบ 64-bit: float64 สองตัว)
type Point struct {
	X, Y float64
}

// NewPointsNoCapHint สร้าง slice ของ Point (แบบ value ไม่ใช่ pointer) โดยไม่บอกความจุล่วงหน้า
// ทำให้ runtime ต้องขยาย (grow) backing array หลายครั้งระหว่างทาง -- จะอธิบายในหัวข้อถัดไป
func NewPointsNoCapHint(n int) []Point {
	var pts []Point
	for i := 0; i < n; i++ {
		pts = append(pts, Point{X: float64(i), Y: float64(i)})
	}
	return pts
}

// NewPointsValue สร้าง slice ของ Point แบบ value พร้อมบอกความจุล่วงหน้าด้วย make(..., 0, n)
func NewPointsValue(n int) []Point {
	pts := make([]Point, 0, n)
	for i := 0; i < n; i++ {
		pts = append(pts, Point{X: float64(i), Y: float64(i)})
	}
	return pts
}

// NewPointsPointer ทำงานแบบเดียวกัน แต่เก็บเป็น slice ของ *Point (pointer) แทน
// ทุก Point ที่สร้างต้อง allocate แยกก้อนบน heap ต่างหาก
func NewPointsPointer(n int) []*Point {
	pts := make([]*Point, 0, n)
	for i := 0; i < n; i++ {
		pts = append(pts, &Point{X: float64(i), Y: float64(i)})
	}
	return pts
}
```

เขียน benchmark เปรียบเทียบทั้งสามแบบ:

```go
// points_bench_test.go
package geometry

import "testing"

var sinkPoints []Point
var sinkPointers []*Point

func BenchmarkNewPointsNoCapHint(b *testing.B) {
	b.ReportAllocs()
	for i := 0; i < b.N; i++ {
		sinkPoints = NewPointsNoCapHint(1000)
	}
}

func BenchmarkNewPointsValue(b *testing.B) {
	b.ReportAllocs()
	for i := 0; i < b.N; i++ {
		sinkPoints = NewPointsValue(1000)
	}
}

func BenchmarkNewPointsPointer(b *testing.B) {
	b.ReportAllocs()
	for i := 0; i < b.N; i++ {
		sinkPointers = NewPointsPointer(1000)
	}
}
```

รันด้วยคำสั่งมาตรฐานจาก Part 084:

```bash
go test -bench=. -benchmem -run=^$ ./...
```

ผลลัพธ์จริง:

```
goos: linux
goarch: amd64
pkg: geometry
cpu: Intel(R) Xeon(R) Processor @ 2.10GHz
BenchmarkNewPointsNoCapHint-4   	   87190	     14958 ns/op	   50416 B/op	      12 allocs/op
BenchmarkNewPointsValue-4       	  255741	      4510 ns/op	   16384 B/op	       1 allocs/op
BenchmarkNewPointsPointer-4     	   53287	     23940 ns/op	   24192 B/op	    1001 allocs/op
PASS
ok  	geometry	4.161s
```

### วิเคราะห์ผลลัพธ์ทีละแถว

- **`NewPointsPointer`: 1001 allocs/op** — 1 ครั้งสำหรับ backing array ของ `[]*Point` เอง บวกอีก **1000 ครั้ง** สำหรับ `&Point{...}` แต่ละตัวที่ต้อง allocate แยกก้อนบน heap (เพราะ pointer หลุดออกจาก loop ไปเก็บใน slice ที่มีอายุยืนกว่า escape analysis จึงบังคับให้ทุก `Point` escape) ผลคือ **ช้าที่สุด** (23940 ns/op) ทั้งที่ใช้ byte รวมน้อยกว่า `NewPointsNoCapHint` ด้วยซ้ำ (24192 B/op เทียบกับ 50416 B/op) — นี่คือหลักฐานชัดเจนว่า **จำนวนครั้งของ allocation (allocs/op) ส่งผลต่อความเร็วมากกว่าจำนวน byte รวม (B/op)** เพราะทุก allocation มีต้นทุนคงที่ (การหา slot ว่าง, การอัปเดต metadata ของ allocator) ไม่ใช่แค่ต้นทุนตามขนาดข้อมูล
- **`NewPointsValue`: 1 alloc/op เท่านั้น** — เพราะ `Point` เป็น value ธรรมดา ถูกเก็บ**ติดกันในหน่วยความจำ**ภายใน backing array เดียวของ slice ไม่มี pointer แยกให้ escape เป็นรายตัว การ allocate เกิดขึ้นแค่ครั้งเดียวตอนสร้าง backing array ก้อนใหญ่ก้อนเดียว ทำให้**เร็วที่สุดในสามแบบ** (4510 ns/op) และมี**cache locality ดีที่สุด**ด้วย (ข้อมูลอยู่ติดกัน CPU cache อ่านต่อเนื่องได้มีประสิทธิภาพ)
- **`NewPointsNoCapHint`: 12 allocs/op** — จะอธิบายในหัวข้อถัดไป

### กฎที่ควรจำ

> **ถ้า struct มีขนาดเล็ก (ไม่กี่ field ธรรมดาอย่าง `int`, `float64`, `string` สั้นๆ) และไม่มีเหตุผลที่ต้องแชร์การแก้ไขระหว่างหลายที่ ให้พิจารณาใช้ value แทน pointer** โดยเฉพาะเมื่อเก็บเป็น slice จำนวนมาก เพราะ value ทำให้ทั้ง slice เป็นก้อนเดียวติดกันบน heap (หรือแม้แต่บน stack ถ้า slice นั้นไม่ escape เลย) แทนที่จะเป็นกระจัดกระจายหลายร้อยหลายพันก้อนเหมือน pointer

ข้อควรระวัง: กฎนี้ใช้กับ struct **ขนาดเล็ก** เท่านั้น ถ้า struct มีขนาดใหญ่มาก (หลาย field ขนาดใหญ่ เช่น array คงที่ขนาดใหญ่ฝังอยู่ข้างใน) การ copy value ทุกครั้งที่ส่งผ่านฟังก์ชันอาจแพงกว่าการส่ง pointer — หลักการเดียวกับที่เรียนเรื่อง value receiver กับ pointer receiver ใน Part 012 นั่นเอง: **เล็กใช้ value ใหญ่ใช้ pointer** ไม่มีกฎตายตัวที่ใช้ได้ทุกกรณี ต้องวัดผลจริงเมื่อไม่แน่ใจ

---

## 3. กลยุทธ์ที่ 2: Pre-allocate slice ด้วย `make([]T, 0, n)` เมื่อรู้ขนาดล่วงหน้า

กลับมาดูผลลัพธ์ของ `NewPointsNoCapHint` จากหัวข้อก่อนหน้าอีกครั้ง:

```
BenchmarkNewPointsNoCapHint-4   	   87190	     14958 ns/op	   50416 B/op	      12 allocs/op
BenchmarkNewPointsValue-4       	  255741	      4510 ns/op	   16384 B/op	       1 allocs/op
```

ทั้งสองฟังก์ชันเก็บข้อมูลแบบ value เหมือนกันทุกประการ ต่างกันแค่จุดเดียว: `NewPointsNoCapHint` ประกาศ `var pts []Point` (slice ว่างเปล่า ไม่มีความจุ) ในขณะที่ `NewPointsValue` ใช้ `make([]Point, 0, n)` (จองความจุ `n` ไว้ล่วงหน้า) ผลต่างคือ **12 allocs/op เทียบกับ 1 alloc/op** — ต่างกันถึง 12 เท่า!

### ทำไมถึงต่างกันมากขนาดนี้: กลไกการขยาย slice

ทบทวนจาก Part 006: เมื่อ `append` ทำให้ slice เกินความจุ (`cap`) ปัจจุบัน Go runtime จะ **allocate backing array ก้อนใหม่ที่ใหญ่ขึ้น** (ปกติเพิ่มเป็นประมาณ 2 เท่าสำหรับ slice ขนาดเล็ก และช้าลงเป็นประมาณ 1.25 เท่าเมื่อ slice ใหญ่ขึ้น) แล้ว **copy ข้อมูลเดิมทั้งหมด**ไปยังก้อนใหม่ ก่อนจะทิ้งก้อนเก่าให้ GC เก็บกวาด

เมื่อไม่บอกความจุล่วงหน้า (`var pts []Point`) การเติม 1000 element เข้าไปทีละตัวด้วย `append` ทำให้เกิดการขยายซ้ำหลายรอบ (ประมาณ `log2(1000) ≈ 10` ครั้ง บวกกับ overhead อื่นๆ ที่ทำให้ตัวเลขจริงออกมาเป็น 12) **ทุกครั้งที่ขยาย คือหนึ่ง heap allocation ก้อนใหม่ บวกกับการ copy ข้อมูลทั้งหมดที่มีอยู่ ณ ขณะนั้น** ยิ่ง slice โตขึ้นเรื่อยๆ ยิ่งต้อง copy ข้อมูลมากขึ้นในแต่ละรอบขยาย ทำให้ต้นทุนรวมสูงกว่าการ allocate ครั้งเดียวจบมาก

เมื่อรู้ขนาดสุดท้ายล่วงหน้าด้วย `make([]Point, 0, n)` runtime จอง backing array ขนาด `n` ไว้ตั้งแต่ต้น การ `append` ทุกครั้งหลังจากนั้นแค่เขียนลงในพื้นที่ที่จองไว้แล้ว **ไม่ต้อง allocate หรือ copy อะไรเพิ่มเลยตลอดทั้งลูป**

### กฎการปฏิบัติ

> **ทุกครั้งที่รู้ (หรือประมาณได้อย่างสมเหตุสมผล) ว่า slice หรือ map ที่กำลังจะสร้างมีขนาดสุดท้ายประมาณเท่าไร ให้จองความจุล่วงหน้าเสมอ** ด้วย `make([]T, 0, n)` สำหรับ slice หรือ `make(map[K]V, n)` สำหรับ map (map ก็มีกลไกขยายคล้ายกันเมื่อจำนวน key เกิน load factor ที่กำหนดไว้ภายใน)

ตัวอย่างสถานการณ์จริงที่พบบ่อย:

```go
// แย่: ไม่รู้ขนาดล่วงหน้าทั้งที่จริงๆ รู้อยู่แล้ว
func toUppercaseSlow(items []string) []string {
	var result []string
	for _, s := range items {
		result = append(result, strings.ToUpper(s))
	}
	return result
}

// ดี: รู้อยู่แล้วว่าผลลัพธ์มีขนาดเท่ากับ items เป๊ะ จองความจุไว้ตั้งแต่ต้น
func toUppercaseFast(items []string) []string {
	result := make([]string, 0, len(items))
	for _, s := range items {
		result = append(result, strings.ToUpper(s))
	}
	return result
}
```

กรณีนี้พบได้ทั่วไปมาก โดยเฉพาะเมื่อ transform slice หนึ่งไปเป็นอีก slice หนึ่งแบบ 1-to-1 (ขนาดผลลัพธ์เท่ากับ input เป๊ะ) หรือ query ฐานข้อมูลที่รู้จำนวนแถวล่วงหน้าจาก `COUNT(*)` แล้วค่อย scan ผลลัพธ์เข้า slice ที่จองความจุไว้แล้ว — เป็นการ optimize ที่ **แทบไม่มีต้นทุนด้านความอ่านง่ายเลย** (แค่เปลี่ยนบรรทัดประกาศตัวแปรบรรทัดเดียว) แต่ได้ผลตอบแทนชัดเจน จึงควรทำเป็นนิสัยเมื่อรู้ขนาดล่วงหน้า

---

## 4. กลยุทธ์ที่ 3: ใช้ซ้ำหน่วยความจำด้วย `sync.Pool` (และกับดักเรื่อง interface boxing)

Part 055 แนะนำ `sync.Pool` ไปสั้นๆ แล้วว่าช่วยลด allocation ได้มากในสถานการณ์ที่มีการจัดสรร object ขนาดเดิมซ้ำๆ ปริมาณมาก (เช่น buffer สำหรับอ่าน network connection) บทนี้จะเจาะลึกกว่านั้น รวมถึง **กับดักที่พบบ่อยมากแต่แทบไม่มีใครพูดถึง**: การใช้ `sync.Pool` ผิดวิธีอาจทำให้**ไม่ได้ประโยชน์อะไรเลย**

### สถานการณ์จำลอง

สมมติมีฟังก์ชันที่ต้องใช้ buffer ชั่วคราวขนาด 4096 byte ต่อการประมวลผลหนึ่ง request แล้วเขียนผลลัพธ์ออกไปยัง `io.Writer` ปลายทาง (เช่น network connection จริงในระบบ production):

```go
// bufpool.go
package reqhandler

import (
	"io"
	"sync"
)

// sinkWriter จำลอง io.Writer ปลายทาง (เช่น network connection หรือ response writer จริง)
// การเขียนผ่าน interface นี้ทำให้ compiler มองไม่เห็นว่า buffer จะถูกใช้ต่ออย่างไร (ตามหลักการที่เรียนใน Part 055)
// จึงบังคับให้ buf ต้อง escape ไป heap เสมอ จำลองสถานการณ์จริงที่ต้องส่งข้อมูลออก connection จริง
var sinkWriter io.Writer = io.Discard

// processWithoutPool allocate buffer ใหม่จาก heap ทุกครั้งที่เรียก
func processWithoutPool(data []byte) {
	buf := make([]byte, 4096)
	n := copy(buf, data)
	sinkWriter.Write(buf[:n])
}
```

วัดจำนวน allocation ด้วย `testing.AllocsPerRun` (ทบทวนจาก Part 050):

```go
// bufpool_test.go
package reqhandler

import "testing"

func TestAllocs_WithoutPool(t *testing.T) {
	payload := []byte("hello world")
	allocs := testing.AllocsPerRun(1000, func() {
		processWithoutPool(payload)
	})
	t.Logf("allocs/op WITHOUT pool: %.1f", allocs)
}
```

```
=== RUN   TestAllocs_WithoutPool
    bufpool_test.go:9: allocs/op WITHOUT pool: 1.0
--- PASS: TestAllocs_WithoutPool (0.00s)
```

ตามคาด: 1 allocation ต่อการเรียกหนึ่งครั้ง มาลองแก้ด้วย `sync.Pool` แบบที่คนส่วนใหญ่เขียนกันเป็นครั้งแรก:

```go
// วิธีที่ "ดูเหมือน" ถูกต้อง แต่มีกับดักซ่อนอยู่
var naiveBytePool = sync.Pool{
	New: func() any {
		return make([]byte, 4096)
	},
}

func naiveProcess(data []byte) {
	buf := naiveBytePool.Get().([]byte) // เก็บ []byte ตรงๆ
	defer naiveBytePool.Put(buf)
	n := copy(buf, data)
	sinkWriter.Write(buf[:n])
}
```

```go
func TestAllocs_NaivePoolBoxing(t *testing.T) {
	payload := []byte("hello world")
	allocs := testing.AllocsPerRun(1000, func() {
		naiveProcess(payload)
	})
	t.Logf("allocs/op naive []byte pool (boxing ทุกครั้ง): %.1f", allocs)
}
```

ผลลัพธ์จริงที่น่าตกใจ:

```
=== RUN   TestAllocs_NaivePoolBoxing
    naive_pool_test.go:27: allocs/op naive []byte pool (boxing ทุกครั้ง): 1.0
--- PASS: TestAllocs_NaivePoolBoxing (0.00s)
```

**ยังคง 1 allocation ต่อครั้ง เท่ากับตอนไม่ใช้ `sync.Pool` เลย!** ใช้ `sync.Pool` ไปแล้วแต่ไม่ได้ประโยชน์อะไรเลยสักนิด — นี่คือกับดักที่จะอธิบายในหัวข้อถัดไป

### สาเหตุ: `sync.Pool.Get()`/`Put()` รับ-คืนค่าเป็น `any`

`sync.Pool` มี signature เป็น `Get() any` และ `Put(x any)` — ทุกครั้งที่ `Put(buf)` โดย `buf` เป็น `[]byte` (value type ที่เป็น slice header ขนาด 3 คำ: pointer, len, cap) ค่านั้นต้องถูก **"กล่อง" (box)** ลงใน interface `any` เสียก่อน และการกล่องค่าที่มีขนาดใหญ่กว่า 1 คำ (word) ลงใน interface **ต้อง heap allocate เพิ่มเพื่อเก็บข้อมูลนั้น** (interface ภายในเก็บแค่ type descriptor + pointer ไปยังข้อมูลจริง ถ้าข้อมูลไม่ใช่ pointer อยู่แล้ว ต้อง allocate ที่เก็บแยกต่างหาก) — สรุปคือ **การ boxing เอง กลายเป็น allocation ใหม่ที่มาแทนที่ allocation เดิมของ `make()` พอดี** ทำให้ `sync.Pool` ไม่ได้ช่วยอะไรเลยในกรณีนี้

### วิธีแก้: เก็บ pointer แทนค่าตรงๆ

```go
// bufferPool เก็บ "pointer ไปยัง []byte" (*[]byte) ไม่ใช่ []byte ตรงๆ
var bufferPool = sync.Pool{
	New: func() any {
		buf := make([]byte, 4096)
		return &buf
	},
}

// processWithPool ยืม buffer จาก pool ผ่าน pointer แล้วคืนกลับทันทีที่ใช้เสร็จ
func processWithPool(data []byte) {
	bufPtr := bufferPool.Get().(*[]byte)
	defer bufferPool.Put(bufPtr)
	buf := *bufPtr
	n := copy(buf, data)
	sinkWriter.Write(buf[:n])
}
```

```go
func TestAllocs_BufferPool(t *testing.T) {
	payload := []byte("hello world")
	allocs := testing.AllocsPerRun(1000, func() {
		processWithPool(payload)
	})
	t.Logf("allocs/op WITH pointer-based pool: %.1f", allocs)
}
```

ผลลัพธ์จริง:

```
=== RUN   TestAllocs_BufferPool
    bufpool_test.go:20: allocs/op WITH pointer-based pool: 0.0
--- PASS: TestAllocs_BufferPool (0.01s)
```

**0.0 allocs/op!** เพราะ `*[]byte` เป็น pointer ธรรมดา (1 คำ) ที่บรรจุลง interface data word ได้โดยตรง**โดยไม่ต้อง allocate เพิ่ม** สรุปผลทั้งสามแบบเปรียบเทียบกัน:

| แบบ | allocs/op | หมายเหตุ |
|---|---|---|
| ไม่ใช้ pool เลย | 1.0 | `make([]byte, 4096)` ทุกครั้ง |
| `sync.Pool` เก็บ `[]byte` ตรงๆ | 1.0 | **ไม่ได้ประโยชน์เลย** — allocation ย้ายไปที่ boxing แทน |
| `sync.Pool` เก็บ `*[]byte` | 0.0 | ได้ประโยชน์เต็มที่ตามที่ตั้งใจ |

> **กฎสำคัญ**: เมื่อใช้ `sync.Pool` เก็บ slice, string, หรือ struct ขนาดใหญ่กว่า 1 คำ **ให้เก็บเป็น pointer เสมอ** (`*[]byte`, `*bytes.Buffer`, `*MyStruct`) ไม่ใช่ value ตรงๆ มิฉะนั้นจะเจอกับดักเดียวกับตัวอย่างนี้ — ในทางปฏิบัติ ไลบรารีมาตรฐานอย่าง `sync.Pool` ที่ใช้กับ `bytes.Buffer` ในโค้ด production จริงแทบทั้งหมดจึงเก็บเป็น `*bytes.Buffer` เสมอ ไม่ใช่ `bytes.Buffer` ตรงๆ

---

## 5. String-building ทบทวนจาก Part 019/050: `strings.Builder` กับ `Grow()`

Part 019 สอนไปแล้วว่า `strings.Builder` เร็วกว่าการต่อ string ด้วย `+=` ในลูปถึงหลักพันเท่า เพราะเปลี่ยนความซับซ้อนจาก O(n²) เป็น O(n) แต่ `strings.Builder` เองก็ยังมีจุดที่ optimize เพิ่มได้อีกขั้น: **การบอกความจุล่วงหน้าด้วย `Grow(n)`** เหมือนหลักการเดียวกับ `make([]T, 0, n)` ในหัวข้อ 3 ทุกประการ (เพราะภายใน `strings.Builder` เก็บข้อมูลเป็น `[]byte` ที่ขยายแบบเดียวกับ slice)

```go
// builder.go
package textbuild

import "strings"

// buildNoGrow ต่อ string ด้วย strings.Builder แบบไม่บอกความจุล่วงหน้า
func buildNoGrow(words []string) string {
	var b strings.Builder
	for _, w := range words {
		b.WriteString(w)
		b.WriteString(" ")
	}
	return b.String()
}

// buildWithGrow คำนวณขนาดรวมล่วงหน้า แล้วเรียก Grow() ก่อนเริ่มเขียน
func buildWithGrow(words []string) string {
	var b strings.Builder
	total := 0
	for _, w := range words {
		total += len(w) + 1 // +1 สำหรับช่องว่างที่จะเติมต่อท้าย
	}
	b.Grow(total)
	for _, w := range words {
		b.WriteString(w)
		b.WriteString(" ")
	}
	return b.String()
}
```

Benchmark เปรียบเทียบด้วย slice ของคำ 200 คำ:

```go
// builder_bench_test.go
package textbuild

import "testing"

var sinkStr string

func BenchmarkBuildNoGrow(b *testing.B) {
	words := make([]string, 200)
	for i := range words {
		words[i] = "golang"
	}
	b.ReportAllocs()
	for i := 0; i < b.N; i++ {
		sinkStr = buildNoGrow(words)
	}
}

func BenchmarkBuildWithGrow(b *testing.B) {
	words := make([]string, 200)
	for i := range words {
		words[i] = "golang"
	}
	b.ReportAllocs()
	for i := 0; i < b.N; i++ {
		sinkStr = buildWithGrow(words)
	}
}
```

ผลลัพธ์จริง:

```
goos: linux
goarch: amd64
pkg: textbuild
cpu: Intel(R) Xeon(R) Processor @ 2.10GHz
BenchmarkBuildNoGrow-4     	  572976	      1941 ns/op	    3320 B/op	       9 allocs/op
BenchmarkBuildWithGrow-4   	 1001301	      1319 ns/op	    1408 B/op	       1 allocs/op
PASS
```

**9 allocs/op ลดเหลือ 1 alloc/op** ด้วยการเพิ่มแค่ 3 บรรทัด (คำนวณขนาดรวมแล้วเรียก `Grow`) — เร็วขึ้น ~32% (1941 → 1319 ns/op) และใช้ memory น้อยลงกว่าครึ่ง (3320 → 1408 B/op) ต้นทุนของการเปลี่ยนแปลงนี้แทบไม่มีเลยในแง่ความอ่านง่าย จึงควรทำเป็นนิสัยเมื่อพอจะประมาณขนาดผลลัพธ์สุดท้ายได้ (เช่น ต่อ string จาก field ของ struct ที่รู้ขนาดโดยประมาณ หรือต่อจาก slice ที่รู้จำนวน element)

> **จำหลักการเดียวกันไว้**: ไม่ว่าจะเป็น `make([]T, 0, n)`, `make(map[K]V, n)`, หรือ `strings.Builder{}.Grow(n)` — **หลักการคือเดียวกันทั้งหมด: บอก allocator ล่วงหน้าว่าต้องการพื้นที่เท่าไร เพื่อเลี่ยงการขยายซ้ำหลายรอบ**

---

## 6. หลีกเลี่ยง Interface Boxing โดยไม่จำเป็น

หัวข้อ 4 แนะนำแนวคิด **interface boxing** ไปแล้วในบริบทของ `sync.Pool` แต่จริงๆ แล้วปัญหานี้เกิดขึ้นได้ทั่วไปกว่านั้นมาก **ทุกครั้งที่ค่าที่ไม่ใช่ pointer (และมีขนาดใหญ่กว่า 1 คำ หรือแม้แต่ int ที่มีค่ามากพอ) ถูกส่งผ่าน parameter ชนิด interface (`any`, `error`, หรือ interface ที่กำหนดเอง) จะมีโอกาสสูงที่ compiler ต้อง allocate เพื่อกล่องค่านั้นลง interface**

### ตัวอย่างที่พบบ่อยที่สุด: `fmt.Sprintf` กับ `strconv`

ฟังก์ชันแปลง `int` เป็น `string` เป็นงานที่พบได้แทบทุกที่ในโค้ด Go มาเปรียบเทียบสองวิธีที่ให้ผลลัพธ์เหมือนกัน:

```go
// numformat.go
package numformat

import (
	"fmt"
	"strconv"
)

func FormatSprintf(n int) string {
	return fmt.Sprintf("%d", n)
}

func FormatItoa(n int) string {
	return strconv.Itoa(n)
}
```

วัดด้วย `testing.AllocsPerRun`:

```go
func TestAllocs_InterfaceBoxing(t *testing.T) {
	n := 123456 // ตัวเลขเกิน 100 เพื่อเลี่ยง small-integer string cache ภายใน strconv ที่อาจทำให้ผลลัพธ์เข้าใจผิด

	allocsSprintf := testing.AllocsPerRun(1000, func() {
		sinkStr = fmt.Sprintf("%d", n)
	})
	allocsItoa := testing.AllocsPerRun(1000, func() {
		sinkStr = strconv.Itoa(n)
	})

	t.Logf("allocs/op fmt.Sprintf(\"%%d\", n): %.1f", allocsSprintf)
	t.Logf("allocs/op strconv.Itoa(n):        %.1f", allocsItoa)
}
```

ผลลัพธ์จริง:

```
=== RUN   TestAllocs_InterfaceBoxing
    boxing_test.go:21: allocs/op fmt.Sprintf("%d", n): 2.0
    boxing_test.go:22: allocs/op strconv.Itoa(n):        1.0
--- PASS: TestAllocs_InterfaceBoxing (0.00s)
```

`fmt.Sprintf("%d", n)` allocate **2 ครั้ง**: หนึ่งครั้งสำหรับ**กล่อง `n` (int) ลงใน `any`** เพื่อส่งเป็น variadic argument (`...any` ตาม signature ของ `fmt.Sprintf`) และอีกหนึ่งครั้งสำหรับสร้าง string ผลลัพธ์ ในขณะที่ `strconv.Itoa(n)` allocate แค่ **1 ครั้ง** (เฉพาะ string ผลลัพธ์) เพราะรับ `int` เป็น parameter ชนิดตรงๆ ไม่ผ่าน interface เลย

> **หมายเหตุ**: ถ้าลองใช้ `n` ที่มีค่าน้อยกว่า 256 ผลลัพธ์ของ `strconv.Itoa` อาจแสดง 0 allocation เพราะ Go runtime มี cache ค่า string ของตัวเลขน้อยๆ ไว้ภายในเพื่อลด allocation ในกรณีที่พบบ่อย — เป็นรายละเอียด implementation ที่ไม่ควรอิงพึ่งในการออกแบบโค้ด แต่เป็นตัวอย่างที่ดีว่า standard library เองก็ให้ความสำคัญกับการลด allocation ในจุดที่ถูกเรียกบ่อยมาก

### กฎการปฏิบัติ

> **ในโค้ดที่อยู่ใน hot path จริงๆ (ผ่าน profiling ยืนยันแล้ว) ให้เลือกใช้ฟังก์ชันที่รับ parameter เป็น concrete type ตรงๆ แทนที่จะผ่าน `interface{}`/`any` เมื่อมีตัวเลือกให้ใช้** เช่น `strconv.Itoa`/`strconv.FormatFloat` แทน `fmt.Sprintf` เมื่อแค่ต้องการแปลงเลขเป็น string เดี่ยวๆ, หรือ logging library ที่มี method รับ field แบบ typed (เช่น `zap.Int("count", n)`) แทนการ log ด้วย `fmt.Sprintf` ที่ต้อง box ทุก argument

ยืนยันด้วย escape analysis (ทบทวนจาก Part 055):

```bash
go build -gcflags="-m" numformat.go
```

ผลลัพธ์ (ตัดเฉพาะบรรทัดสำคัญ):

```
./numformat.go:10:21: n escapes to heap
./numformat.go:14:20: ... argument does not escape
```

บรรทัดแรกยืนยันตรงตามที่อธิบาย: `n` (ที่ส่งเข้า `fmt.Sprintf` ผ่าน `...any`) escape ไป heap เพราะถูกกล่องลง interface ในขณะที่ `strconv.Itoa` ไม่มีข้อความ escape สำหรับ `n` เลยเพราะรับ parameter เป็น `int` ตรงๆ

**ข้อควรระวังเรื่องความสมดุล**: กฎนี้ใช้เฉพาะจุดที่พิสูจน์แล้วว่าเป็น hot path เท่านั้น การเขียน log ทั่วไปในโค้ด business logic ธรรมดาที่เรียกไม่บ่อย **ไม่จำเป็นต้องกังวลเรื่องนี้เลย** — `fmt.Sprintf`/`log.Printf` อ่านง่ายกว่ามากและควรเป็นค่าเริ่มต้นเสมอ จนกว่า profiling จะชี้ชัดว่าจุดนั้นเป็นคอขวดจริง (จะย้ำเรื่องนี้อีกครั้งในหัวข้อสุดท้ายของบทนี้)

---

## 7. ผลกระทบของ GC Pause ต่อระบบจริง และเมื่อไรควรปรับ `GOGC`/`GOMEMLIMIT`

Part 055 อธิบายกลไกของ Go GC และตัวแปร `GOGC`/`GOMEMLIMIT` ไปแล้วอย่างละเอียด หัวข้อนี้จะแสดงให้เห็น**ผลกระทบที่วัดได้จริง**ว่าการลด allocation (ตามกลยุทธ์ที่เรียนมาทั้งบท) ส่งผลต่อความถี่ของ GC และประสิทธิภาพโดยรวมของระบบมากแค่ไหน โดยใช้ `GODEBUG=gctrace=1` (ที่ Part 055 ทิ้งไว้เป็นแบบฝึกหัดค้นคว้าเพิ่มเติม) มาพิสูจน์ให้เห็นเป็นตัวเลขจริง

### ทดลอง: allocate หนักๆ เทียบกับใช้ `sync.Pool`

```go
// main.go
package main

import (
	"fmt"
	"io"
	"os"
	"sync"
)

var sink io.Writer = io.Discard

func withoutPool(n int) {
	for i := 0; i < n; i++ {
		buf := make([]byte, 8192)
		sink.Write(buf)
	}
}

var pool = sync.Pool{
	New: func() any {
		buf := make([]byte, 8192)
		return &buf
	},
}

func withPool(n int) {
	for i := 0; i < n; i++ {
		bufPtr := pool.Get().(*[]byte)
		sink.Write(*bufPtr)
		pool.Put(bufPtr)
	}
}

func main() {
	mode := os.Args[1]
	n := 2_000_000
	if mode == "pool" {
		withPool(n)
	} else {
		withoutPool(n)
	}
	fmt.Println("done:", mode)
}
```

รันทั้งสองแบบพร้อมเปิด `GODEBUG=gctrace=1` เพื่อนับจำนวนรอบ GC และวัดเวลารวมด้วย `time`:

```bash
go build -o gcdemo .
time (GODEBUG=gctrace=1 ./gcdemo noop 2>gc_noop.log; wc -l gc_noop.log)
time (GODEBUG=gctrace=1 ./gcdemo pool 2>gc_pool.log; wc -l gc_pool.log)
```

ผลลัพธ์จริง — เวอร์ชันที่ allocate ทุกครั้ง (2 ล้านครั้ง):

```
done: noop
5090 gc_noop.log

real	0m5.005s
user	0m6.130s
sys	0m3.379s
```

ผลลัพธ์จริง — เวอร์ชันที่ใช้ `sync.Pool`:

```
done: pool
0 gc_pool.log

real	0m0.027s
user	0m0.028s
sys	0m0.001s
```

### วิเคราะห์ผลลัพธ์ — ตัวเลขที่พูดเสียงดังกว่าคำอธิบายใดๆ

- เวอร์ชัน**ไม่ใช้ pool**ทำให้เกิด GC ไปทั้งหมด **5,090 รอบ** ในระหว่างการทำงาน (จาก allocation 2 ล้าน buffer ขนาด 8 KB ทำให้ heap โตเร็วมากจน trigger GC ถี่มาก) ใช้เวลารวม **5.0 วินาที**
- เวอร์ชัน**ใช้ `sync.Pool`**ไม่ trigger GC แม้แต่รอบเดียว (**0 รอบ**) เพราะแทบไม่มี heap allocation ใหม่เกิดขึ้นเลยตลอดการทำงาน (buffer ถูกนำกลับมาใช้ซ้ำทั้งหมด) ใช้เวลารวมเพียง **0.027 วินาที**
- **ความต่างคือเกือบ 185 เท่า** (5.005s เทียบกับ 0.027s) — นี่ไม่ใช่แค่ "เร็วขึ้นเล็กน้อย" แต่เป็นความต่างระดับที่เปลี่ยนแปลงความเป็นไปได้ของระบบทั้งหมด (จาก "รับได้ไม่กี่ request ต่อวินาที" เป็น "รับได้เป็นแสนต่อวินาที" ในสถานการณ์จริงที่มี pattern การใช้ buffer แบบนี้)

ตัวเลขนี้แสดงให้เห็นชัดเจนว่า**การลด allocation ไม่ได้แค่ลดภาระ GC แบบผิวเผิน แต่ส่งผลถึงคอขวดของระบบทั้งระบบได้โดยตรง** เมื่อ hot path ถูกออกแบบไม่ดี

### ทดลองต่อ: ปรับ `GOGC` เมื่อยังต้อง allocate หนักอยู่

สมมติสถานการณ์ที่แก้โค้ดให้ไม่ allocate เลยไม่ได้จริงๆ (เช่น library ภายนอกที่ควบคุมไม่ได้) แต่ยังพอปรับ `GOGC` ได้ ลองรันเวอร์ชัน `noop` (allocate หนัก) เทียบกันระหว่าง `GOGC` ค่า default กับค่าที่สูงขึ้น:

```bash
time (GOGC=100 GODEBUG=gctrace=1 ./gcdemo noop 2>gc100.log; wc -l gc100.log)
time (GOGC=400 GODEBUG=gctrace=1 ./gcdemo noop 2>gc400.log; wc -l gc400.log)
```

ผลลัพธ์จริง:

```
=== GOGC=100 (default) ===
5019 gc100.log
real	0m4.778s

=== GOGC=400 ===
1140 gc400.log
real	0m2.202s
```

การเพิ่ม `GOGC` จาก 100 เป็น 400 (อนุญาตให้ heap โตได้มากขึ้นก่อนที่ GC รอบถัดไปจะเริ่มทำงาน ตามสูตรที่เรียนใน Part 055) ลดจำนวนรอบ GC จาก **5,019 เหลือ 1,140 รอบ** (ลดลงกว่า 4 เท่า) และลดเวลารวมจาก **4.78 วินาทีเหลือ 2.2 วินาที** (เร็วขึ้นกว่า 2 เท่า) **แลกมาด้วยการที่โปรแกรมใช้ memory สูงสุดมากขึ้นระหว่างทาง** (heap ได้รับอนุญาตให้โตกว่าเดิมก่อน trigger การเก็บกวาด)

### บทสรุปของหัวข้อนี้: ลำดับความสำคัญที่ถูกต้อง

จากทั้งสองการทดลองข้างต้น ควรเรียงลำดับการแก้ปัญหาแบบนี้เสมอ:

1. **ลด allocation ที่ต้นตอก่อนเสมอ** (กลยุทธ์ทั้งหมดในหัวข้อ 2-6 ของบทนี้) — ให้ผลลัพธ์ที่ดีที่สุดและยั่งยืนที่สุด (185 เท่าในตัวอย่างของเรา) เพราะแก้ที่สาเหตุจริง ไม่ใช่แค่ "ประวิงเวลา" ให้ GC ทำงานน้อยลง
2. **ปรับ `GOGC`/`GOMEMLIMIT` เป็นทางเลือกรองลงมา** เมื่อ (ก) ได้ลด allocation จากโค้ดของตัวเองจนสุดทางแล้ว หรือ (ข) allocation ส่วนใหญ่มาจาก dependency ภายนอกที่ควบคุมไม่ได้ และ (ค) มีข้อมูลจาก production monitoring ยืนยันชัดเจนว่าคุ้มค่าที่จะแลก memory usage ที่สูงขึ้นเพื่อลด CPU overhead จาก GC — ไม่ใช่ทางเลือกแรกที่ควรทำเมื่อยังไม่ได้พยายามลด allocation ที่ต้นตอ

---

## 8. สรุปทุกกลยุทธ์เป็นกรณีศึกษาเดียว: วัดผลก่อน-หลังด้วย `testing.AllocsPerRun` และ `-benchmem`

มารวบยอดทุกกลยุทธ์ที่เรียนมาในบทนี้เป็นตารางเปรียบเทียบเดียว เพื่อเห็นภาพรวมว่าแต่ละกลยุทธ์ให้ผลตอบแทนมากแค่ไหนเมื่อเทียบกัน (ตัวเลขทั้งหมดคือผลจริงที่วัดได้ในหัวข้อก่อนหน้า):

| กลยุทธ์ | Before (allocs/op) | After (allocs/op) | ปรับปรุง |
|---|---|---|---|
| Value แทน pointer (Point x1000) | 1001 | 1 | **1001 เท่า** |
| Pre-allocate slice (`make(..., 0, n)`) | 12 | 1 | **12 เท่า** |
| `sync.Pool` แบบ pointer (แก้ boxing) | 1 | 0 | **allocation หายไปสมบูรณ์** |
| `strings.Builder.Grow()` | 9 | 1 | **9 เท่า** |
| หลีกเลี่ยง interface boxing (`Itoa` แทน `Sprintf`) | 2 | 1 | **2 เท่า** |

ข้อสังเกตสำคัญจากตารางนี้: **กลยุทธ์ที่ให้ผลตอบแทนมากที่สุด (value vs pointer) คือกลยุทธ์ที่มักถูกมองข้ามมากที่สุด** เพราะดูเป็นเรื่องพื้นฐานเกินกว่าจะคิดว่าสำคัญ ในขณะที่กลยุทธ์ที่คนมักตื่นเต้นกันมาก (เช่น `sync.Pool`) ถ้าใช้ผิดวิธีกลับให้ผลเป็นศูนย์ นี่คือเหตุผลที่ **การวัดผลจริงสำคัญกว่าการเชื่อสัญชาตญาณเสมอ** ตรงตามหลักการที่ Part 084 ย้ำไว้: "วัดผลจริง อย่าเดา"

### แนวทางการนำไปใช้จริงในทีม

เมื่อพบว่าโค้ดส่วนใดส่วนหนึ่งเป็นคอขวดด้าน memory (จาก `pprof` ใน Part 083 หรือจาก benchmark เปรียบเทียบ) ให้ไล่ตรวจตามลำดับนี้:

1. มี struct เล็กๆ ที่ถูกส่งเป็น pointer โดยไม่จำเป็นหรือไม่? → เปลี่ยนเป็น value
2. มี slice/map ที่ `append`/เพิ่ม key โดยไม่บอกความจุล่วงหน้า ทั้งที่รู้ขนาดอยู่แล้วหรือไม่? → เพิ่ม `make(..., 0, n)` หรือ `make(map[K]V, n)`
3. มีการจัดสรร buffer ขนาดเดิมซ้ำๆ ความถี่สูงหรือไม่? → พิจารณา `sync.Pool` **และตรวจสอบว่าเก็บเป็น pointer**
4. มีการต่อ string จำนวนมากหรือไม่? → ใช้ `strings.Builder` พร้อม `Grow()` เมื่อประมาณขนาดได้
5. มีการส่งค่าผ่าน `fmt.Sprintf`/`any`/`interface{}` ใน hot path หรือไม่? → พิจารณาใช้ typed function แทน

---

## 9. Profile ก่อนเสมอ — อย่าเดา-optimize

หลังจากเรียนกลยุทธ์ทั้งหมดในบทนี้ อาจเกิดความรู้สึกอยากไล่ optimize ทุกจุดในโค้ดที่มีอยู่ทันที — แต่นี่คือกับดักเดียวกับที่ Part 055 เตือนไว้เรื่อง **premature optimization** และควรย้ำอีกครั้งให้ชัดเจนที่สุดในบทสุดท้ายของเรื่อง memory:

> **ทุกตัวเลขในบทนี้มาจากการวัดผลจริงด้วย `testing.AllocsPerRun` และ `-benchmem` ไม่ใช่การคาดเดา** — นี่คือวิธีการทำงานที่ถูกต้องเสมอ ไม่ว่าจะเป็นการเรียนรู้ (อย่างในบทนี้) หรือการทำงานจริงในทีม

หลักการที่ควรยึดถือเมื่อกลับไปทำงานกับโค้ดจริง:

1. **หาคอขวดจริงด้วย `pprof` ก่อนเสมอ** (Part 083) — memory profile (`heap` profile) จะชี้ตำแหน่งที่ allocate มากที่สุดในโปรแกรมจริงได้ตรงจุดกว่าการเดาด้วยตาเปล่ามาก อย่าไล่ optimize ทุกฟังก์ชันในโปรเจกต์โดยไม่มีข้อมูลว่าฟังก์ชันไหนคือคอขวดจริง
2. **เขียน benchmark เพื่อยืนยันก่อน-หลังเสมอ** (Part 084) — การเปลี่ยนโค้ดโดยเชื่อว่า "น่าจะเร็วขึ้น" โดยไม่วัดผล อาจทำให้โค้ดซับซ้อนขึ้นโดยไม่ได้ประโยชน์จริง หรือแม้กระทั่งช้าลงกว่าเดิม (เหมือนตัวอย่าง `Dedup` ใน Part 084 ที่เร็วขึ้นด้านเวลาแต่ allocate มากขึ้นถึง 9 เท่า — ต้องชั่งน้ำหนักตามบริบทจริงเสมอ)
3. **โค้ดที่อ่านง่ายชนะเสมอในกรณีที่ไม่ใช่ hot path** — กลยุทธ์ทั้งหมดในบทนี้ (โดยเฉพาะเรื่อง pointer boxing และ interface) เพิ่มความซับซ้อนให้โค้ดเล็กน้อย คุ้มค่าเฉพาะจุดที่พิสูจน์แล้วว่าเป็นคอขวดจริง ไม่ใช่สิ่งที่ควรทำแบบเหมารวมทุกฟังก์ชันในโปรเจกต์
4. **Go GC ได้รับการออกแบบมาอย่างดีสำหรับ workload ทั่วไป** — โปรแกรม web API, CLI tool, หรือ microservice ขนาดกลางส่วนใหญ่ไม่จำเป็นต้องใช้กลยุทธ์ในบทนี้เลยด้วยซ้ำ เก็บเทคนิคเหล่านี้ไว้ใช้เมื่อข้อมูลจริง (จาก monitoring หรือ profiling) ชี้ชัดว่าจำเป็น

การเรียนรู้กลยุทธ์เหล่านี้มีค่าไม่ใช่เพื่อเอาไปใช้ทุกที่ แต่เพื่อ**รู้จักหยิบมาใช้ได้ถูกจุดเมื่อข้อมูลบอกว่าจำเป็นจริงๆ** เท่านั้น

---

## สรุปสิ่งที่ได้เรียนในบทนี้

- **Struct ขนาดเล็กควรส่งเป็น value ไม่ใช่ pointer** เมื่อเก็บเป็น slice จำนวนมาก เพราะ value อยู่ติดกันบน heap ก้อนเดียว ในขณะที่ pointer ทำให้ทุก instance ต้อง allocate แยกกัน (พิสูจน์แล้ว: 1 alloc เทียบกับ 1001 allocs สำหรับ 1000 Point)
- **`make([]T, 0, n)` และ `make(map[K]V, n)`** ลดจำนวนรอบขยาย (grow) เมื่อรู้ขนาดสุดท้ายล่วงหน้า ลด allocation ได้หลายเท่า (12 allocs เหลือ 1 alloc ในตัวอย่าง)
- **`sync.Pool` ลด allocation ได้จริงเฉพาะเมื่อเก็บเป็น pointer** (`*[]byte`, `*bytes.Buffer`) — ถ้าเก็บเป็น value ตรงๆ การ boxing ลง `any` ของ `Get()`/`Put()` จะสร้าง allocation ใหม่มาแทนที่ ทำให้ไม่ได้ประโยชน์อะไรเลย
- **`strings.Builder.Grow(n)`** ใช้หลักการเดียวกับ pre-allocate slice ลด allocation จาก 9 เหลือ 1 ในตัวอย่างจริง
- **Interface boxing** (การส่งค่าผ่าน `any`/`interface{}`) มีต้นทุน allocation แฝงอยู่ — `strconv.Itoa` allocate น้อยกว่า `fmt.Sprintf` เพราะไม่ต้องกล่อง argument ลง interface ควรใช้ typed function แทนใน hot path ที่พิสูจน์แล้วว่าเป็นคอขวด
- **การลด allocation ส่งผลโดยตรงต่อความถี่ของ GC และประสิทธิภาพรวมของระบบ** — ตัวอย่างจริงแสดงความต่างถึง ~185 เท่า (5.0 วินาทีเทียบกับ 0.027 วินาที) ระหว่างเวอร์ชันที่ allocate หนักกับเวอร์ชันที่ใช้ `sync.Pool` อย่างถูกต้อง
- **`GOGC`/`GOMEMLIMIT`** (ทบทวนจาก Part 055) เป็นทางเลือกรองจากการลด allocation ที่ต้นตอ ใช้เมื่อลด allocation จากโค้ดตัวเองจนสุดทางแล้วเท่านั้น
- **Profile ก่อนเสมอ (Part 083), วัดผลด้วย benchmark เสมอ (Part 084), และอย่า optimize โดยไม่มีข้อมูล** — โค้ดที่อ่านง่ายชนะเสมอนอกจาก hot path ที่พิสูจน์แล้วจริงๆ

## แบบฝึกหัดท้ายบท

1. เขียน struct `Employee` ที่มี field `Name string`, `Age int`, `Salary float64` แล้วเขียนฟังก์ชันสองแบบที่สร้าง slice ของพนักงาน 5000 คน แบบแรกเป็น `[]Employee` แบบที่สองเป็น `[]*Employee` เขียน benchmark เปรียบเทียบด้วย `-benchmem` แล้วอธิบายผลลัพธ์ที่ได้บนเครื่องของตัวเอง
2. หยิบฟังก์ชันใดก็ได้จาก Part ก่อนหน้าที่มีการ `append` ในลูปโดยไม่บอกความจุล่วงหน้า แล้วแก้ไขให้ใช้ `make([]T, 0, n)` วัดผลก่อน-หลังด้วย `testing.AllocsPerRun`
3. เขียน `sync.Pool` สำหรับ struct ที่กำหนดเอง (ไม่ใช่ `[]byte`) แล้วทดลองทั้งแบบเก็บเป็น value ตรงๆ กับแบบเก็บเป็น pointer วัดผลด้วย `testing.AllocsPerRun` เพื่อพิสูจน์กับดัก boxing ด้วยตัวเอง
4. เขียนฟังก์ชันที่ต่อ string จำนวนมาก (เช่น สร้าง CSV row จาก slice ของ struct) ด้วย `strings.Builder` แบบไม่ใช้ `Grow` แล้วแก้ให้ใช้ `Grow` วัดผลเปรียบเทียบ
5. ใช้ `go build -gcflags="-m"` กับฟังก์ชันของตัวเองที่ส่งค่าผ่าน `fmt.Sprintf`/`log.Printf` แล้วสังเกตข้อความ `escapes to heap` ที่เกิดจาก interface boxing ลองแก้เป็น typed function (เช่น `strconv`) แล้วเปรียบเทียบผลลัพธ์ escape analysis ก่อน-หลัง
6. ทดลองรันโปรแกรมที่ allocate หนักๆ ของตัวเองด้วย `GODEBUG=gctrace=1` แล้วนับจำนวนรอบ GC ที่เกิดขึ้น จากนั้นลองใช้กลยุทธ์ใดก็ได้จากบทนี้ลดจำนวน allocation แล้ววัดจำนวนรอบ GC อีกครั้งเพื่อดูว่าลดลงมากแค่ไหน

---

**ต่อไป**: [Part 086 — Linting ด้วย `golangci-lint` และ Code Quality](./086-linting-and-code-quality.md)
