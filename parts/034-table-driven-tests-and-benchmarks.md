# Part 034: Table-Driven Tests และ Benchmarks

> ภาคที่ 2: ระดับกลาง (Intermediate) — ตอนที่ 19 จาก 20 (Part 16–35)

## สารบัญของบทนี้

1. Table-Driven Tests คืออะไร และทำไมครองความนิยมในโลก Go
2. เขียน Table-Driven Test ทีละขั้นตอน
3. ข้อควรระวังเรื่อง Loop Variable Capture
4. Golden File Testing: เปรียบเทียบผลลัพธ์กับไฟล์ต้นแบบ
5. Benchmark พื้นฐาน: `func BenchmarkXxx(b *testing.B)` และ `b.N`
6. รัน Benchmark ด้วย `go test -bench` และ `-benchmem`
7. กับดักของ Benchmark: Dead Code Elimination และวิธีป้องกันด้วย `sink`
8. `b.ResetTimer()`: ตัดเวลาการเตรียมข้อมูลออกจากผลวัด
9. ตัวอย่างเต็ม: เทียบประสิทธิภาพสองวิธีต่อ String
10. สรุปสิ่งที่ได้เรียนในบทนี้
11. แบบฝึกหัดท้ายบท

---

## 1. Table-Driven Tests คืออะไร และทำไมครองความนิยมในโลก Go

ใน **Part 033** เราเห็นเทคนิค `t.Run` มาบ้างแล้วตอนเขียน `TestIsEven` ที่วนผ่าน slice ของ struct เพื่อสร้าง subtest หลายตัว — เทคนิคนี้มีชื่อเรียกอย่างเป็นทางการว่า **Table-Driven Test** และเป็น pattern การเขียน test ที่ **พบมากที่สุดในโค้ด Go ทั่วโลก** ไม่ว่าจะเป็นใน standard library เอง หรือโปรเจกต์ open source ขนาดใหญ่แทบทุกโปรเจกต์

แนวคิดหลักคือ: แทนที่จะเขียนฟังก์ชัน `TestXxx` แยกกันสำหรับทุกกรณีทดสอบ (ซึ่งทำให้โค้ดซ้ำซ้อนมากเมื่อ logic การตรวจสอบเหมือนกันหมด ต่างกันแค่ input/output) เรารวบรวมกรณีทดสอบทั้งหมดไว้เป็น **"ตาราง" (table)** ในรูปแบบ slice ของ struct แล้วเขียน logic การตรวจสอบ**เพียงครั้งเดียว** วนลูปผ่านทุกแถวของตาราง

ทำไม pattern นี้ถึงครองความนิยมขนาดนี้:

1. **ลดโค้ดซ้ำอย่างมาก** — logic การตรวจสอบเขียนครั้งเดียว เพิ่มกรณีทดสอบใหม่แค่เพิ่มแถวในตาราง ไม่ต้องเขียนฟังก์ชันใหม่
2. **เห็นภาพรวมกรณีทดสอบทั้งหมดในที่เดียว** — อ่านตารางแล้วรู้ทันทีว่าฟังก์ชันถูกทดสอบกับ input แบบไหนบ้าง เหมือนอ่านเอกสารสเปกของฟังก์ชัน
3. **เพิ่มกรณีทดสอบใหม่ง่ายมาก** — แค่เพิ่มบรรทัดใหม่ในตาราง ไม่ต้อง copy-paste ฟังก์ชันทั้งก้อน
4. **ทำงานร่วมกับ `t.Run` ได้อย่างลงตัว** — แต่ละแถวกลายเป็น subtest ที่รันแยกกัน เห็นผล pass/fail รายกรณีชัดเจน และรันแยกทีละกรณีได้ผ่าน `-run`

ด้วยเหตุผลเหล่านี้ ถ้าเปิดดู source code ของ Go standard library เอง (เช่น package `strings`, `strconv`, `net/http`) จะพบว่าเกือบทุก test file ใช้ pattern นี้เป็นหลัก บทนี้จะพาไปฝึกเขียนให้คล่องแบบเต็มรูปแบบ

---

## 2. เขียน Table-Driven Test ทีละขั้นตอน

มาดูตัวอย่างฟังก์ชัน `Initials` ที่ดึงตัวอักษรตัวแรกของแต่ละคำในประโยคมาต่อกัน (เช่น `"Go Is Fun"` → `"GIF"`):

```go
// strutil.go
package strutil

import "strings"

// Initials คืนตัวอักษรตัวแรกของแต่ละคำ ต่อกันเป็น string เดียว เช่น "Go Is Fun" -> "GIF"
func Initials(sentence string) string {
	words := strings.Fields(sentence)
	var b strings.Builder
	for _, w := range words {
		if len(w) > 0 {
			b.WriteByte(w[0])
		}
	}
	return b.String()
}
```

โครงสร้างมาตรฐานของ table-driven test ประกอบด้วย 3 ส่วนเสมอ:

**ขั้นที่ 1: ประกาศ struct นิรนาม (anonymous struct) เก็บ field ที่จำเป็น** อย่างน้อยต้องมี input กับ output ที่คาดหวัง และควรมี `name` เพื่อให้ subtest มีชื่อที่อ่านเข้าใจง่าย:

```go
tests := []struct {
	name  string
	input string
	want  string
}{
	{"สามคำปกติ", "Go Is Fun", "GIF"},
	{"คำเดียว", "Hello", "H"},
	{"สตริงว่าง", "", ""},
	{"มีช่องว่างซ้อนกัน", "Go   Is   Awesome", "GIA"},
	{"ตัวพิมพ์เล็กทั้งหมด", "go is great", "gig"},
}
```

**ขั้นที่ 2: วนลูปผ่านตาราง สร้าง subtest ด้วย `t.Run` ต่อหนึ่งแถว:**

```go
for _, tt := range tests {
	tt := tt // ป้องกันปัญหา loop variable capture ในเวอร์ชัน Go เก่ากว่า 1.22
	t.Run(tt.name, func(t *testing.T) {
		got := Initials(tt.input)
		if got != tt.want {
			t.Errorf("Initials(%q) = %q, want %q", tt.input, got, tt.want)
		}
	})
}
```

**ขั้นที่ 3: รันดูผลลัพธ์** — รวมทุกอย่างเข้าเป็นฟังก์ชัน test เดียว:

```go
func TestInitials(t *testing.T) {
	tests := []struct {
		name  string
		input string
		want  string
	}{
		{"สามคำปกติ", "Go Is Fun", "GIF"},
		{"คำเดียว", "Hello", "H"},
		{"สตริงว่าง", "", ""},
		{"มีช่องว่างซ้อนกัน", "Go   Is   Awesome", "GIA"},
		{"ตัวพิมพ์เล็กทั้งหมด", "go is great", "gig"},
	}

	for _, tt := range tests {
		tt := tt
		t.Run(tt.name, func(t *testing.T) {
			got := Initials(tt.input)
			if got != tt.want {
				t.Errorf("Initials(%q) = %q, want %q", tt.input, got, tt.want)
			}
		})
	}
}
```

รันด้วย `go test -v`:

```
=== RUN   TestInitials
=== RUN   TestInitials/สามคำปกติ
=== RUN   TestInitials/คำเดียว
=== RUN   TestInitials/สตริงว่าง
=== RUN   TestInitials/มีช่องว่างซ้อนกัน
=== RUN   TestInitials/ตัวพิมพ์เล็กทั้งหมด
--- PASS: TestInitials (0.00s)
    --- PASS: TestInitials/สามคำปกติ (0.00s)
    --- PASS: TestInitials/คำเดียว (0.00s)
    --- PASS: TestInitials/สตริงว่าง (0.00s)
    --- PASS: TestInitials/มีช่องว่างซ้อนกัน (0.00s)
    --- PASS: TestInitials/ตัวพิมพ์เล็กทั้งหมด (0.00s)
PASS
```

ถ้าต้องการเพิ่มกรณีทดสอบใหม่ในอนาคต (เช่น ทดสอบกับข้อความที่มี emoji หรืออักขระพิเศษ) แค่เพิ่มบรรทัดใหม่ในตาราง `tests` เท่านั้น ไม่ต้องแตะโค้ดส่วนวนลูปหรือ logic การตรวจสอบเลยแม้แต่บรรทัดเดียว — นี่คือพลังที่แท้จริงของ pattern นี้

---

## 3. ข้อควรระวังเรื่อง Loop Variable Capture

สังเกตบรรทัด `tt := tt` ในตัวอย่างข้างบนที่ดูเหมือนไม่มีความหมาย (กำหนดตัวแปรให้เท่ากับตัวเอง) — นี่คือ idiom ป้องกันบั๊กคลาสสิกที่เกิดจาก**การจับตัวแปรลูป (loop variable capture)** ใน closure

**ปัญหาคือ**: ก่อน Go 1.22 ตัวแปรที่ประกาศใน `for _, tt := range tests` จะถูก**ใช้ซ้ำตัวเดียวกันทุกรอบของลูป** ไม่ได้สร้างตัวแปรใหม่ในแต่ละรอบ ถ้าใน `t.Run` เรียก `func(t *testing.T) { ... }` ที่อ้างอิงถึง `tt` โดยตรง (closure ทบทวนจาก **Part 009**) แล้ว `t.Run` ทำงานแบบ **asynchronous บางส่วน** (เช่น เรียก `t.Parallel()` ที่จะเจอในหัวข้อขั้นสูงต่อไป) ทุก subtest อาจเห็นค่า `tt` เป็นค่าของ**แถวสุดท้าย**ของตารางเหมือนกันหมด แทนที่จะเป็นค่าของแถวตัวเองตามที่ตั้งใจ

การเขียน `tt := tt` ในลูปคือการสร้าง**ตัวแปรใหม่ในขอบเขตของแต่ละรอบลูป**อย่างชัดเจน (shadow ตัวแปรเดิม) ทำให้ closure แต่ละตัวจับค่าของแถวตัวเองไว้อย่างถูกต้อง ไม่ว่าจะรันแบบขนานหรือไม่ก็ตาม

**ข่าวดี**: ตั้งแต่ **Go 1.22** เป็นต้นไป (ภาษาที่หลักสูตรนี้ใช้อ้างอิงคือ Go 1.22+ ตามที่ระบุใน **Part 001**) พฤติกรรมของตัวแปรลูปถูกเปลี่ยนแล้ว — **ตัวแปรที่ประกาศใน `for` จะถูกสร้างใหม่ทุกรอบของลูปโดยอัตโนมัติ** ทำให้บั๊กแบบนี้**หมดไปโดยสมบูรณ์**แม้จะไม่เขียน `tt := tt` ก็ตาม

ถึงอย่างนั้น **ยังแนะนำให้เขียน `tt := tt` ต่อไปเป็นนิสัยที่ดี** ด้วยเหตุผล 2 ข้อ: (1) โค้ดจำนวนมากในโลกจริงยังต้องรันบน Go เวอร์ชันเก่ากว่า 1.22 หรือทีมงานอาจยังไม่อัปเดต toolchain ทันที และ (2) มันทำให้โค้ดสื่อเจตนาชัดเจนว่า "ตัวแปรนี้ตั้งใจให้เป็นค่าประจำรอบนี้เท่านั้น" อ่านแล้วเข้าใจง่ายขึ้นสำหรับคนที่อาจยังไม่รู้เรื่องการเปลี่ยนแปลงใน Go 1.22

---

## 4. Golden File Testing: เปรียบเทียบผลลัพธ์กับไฟล์ต้นแบบ

เมื่อผลลัพธ์ของฟังก์ชันมีขนาดใหญ่หรือซับซ้อนเกินกว่าจะเขียนค่าคาดหวัง (`want`) ไว้เป็น string literal ในโค้ด test ตรงๆ ได้อย่างสะดวก (เช่น รายงานหลายบรรทัด, HTML output, หรือ JSON ก้อนใหญ่) มีเทคนิคที่เรียกว่า **Golden File Testing** ให้ใช้แทน

แนวคิดคือ: เก็บผลลัพธ์ที่ถูกต้อง ("ต้นแบบ" หรือ golden) ไว้เป็น**ไฟล์แยกต่างหาก** ในโฟลเดอร์ `testdata/` (Go toolchain รู้จักโฟลเดอร์ชื่อนี้เป็นพิเศษ — `go build` จะไม่รวมมันเข้า package แต่ `go test` เข้าถึงได้ตามปกติ) แล้ว test แค่**เปรียบเทียบผลลัพธ์จริงกับเนื้อหาในไฟล์** แทนการเขียนค่าคาดหวังไว้ในโค้ด:

```go
// report.go
package report

import "fmt"

func GenerateReport(name string, score int) string {
	grade := "F"
	switch {
	case score >= 80:
		grade = "A"
	case score >= 70:
		grade = "B"
	case score >= 60:
		grade = "C"
	}
	return fmt.Sprintf("=== รายงานผลการเรียน ===\nชื่อ: %s\nคะแนน: %d\nเกรด: %s\n", name, score, grade)
}
```

```go
// report_test.go
package report

import (
	"flag"
	"os"
	"path/filepath"
	"testing"
)

// update คือ flag พิเศษที่ทำให้ test เขียนทับไฟล์ golden ใหม่แทนการเปรียบเทียบ
// เรียกใช้ตอนตั้งใจอัปเดตค่าคาดหวังใหม่ด้วยคำสั่ง: go test ./... -run TestGenerateReport_Golden -update
var update = flag.Bool("update", false, "อัปเดตไฟล์ golden ใหม่จากผลลัพธ์ปัจจุบัน")

func TestGenerateReport_Golden(t *testing.T) {
	got := GenerateReport("สมชาย", 85)

	goldenPath := filepath.Join("testdata", "report_a_grade.golden")

	if *update {
		if err := os.WriteFile(goldenPath, []byte(got), 0644); err != nil {
			t.Fatalf("เขียนไฟล์ golden ไม่สำเร็จ: %v", err)
		}
	}

	want, err := os.ReadFile(goldenPath)
	if err != nil {
		t.Fatalf("อ่านไฟล์ golden ไม่สำเร็จ: %v", err)
	}

	if got != string(want) {
		t.Errorf("ผลลัพธ์ไม่ตรงกับ golden file\n--- got ---\n%s\n--- want ---\n%s", got, want)
	}
}
```

ครั้งแรกที่สร้างไฟล์ golden ให้รันด้วย flag `-update` เพื่อสร้างไฟล์จากผลลัพธ์ปัจจุบัน (ต้องแน่ใจว่าผลลัพธ์ตอนนั้นถูกต้องแล้ว!):

```bash
go test ./report/... -run TestGenerateReport_Golden -update
```

หลังจากนั้นไฟล์ `testdata/report_a_grade.golden` จะถูกสร้างขึ้นพร้อมเนื้อหา:

```
=== รายงานผลการเรียน ===
ชื่อ: สมชาย
คะแนน: 85
เกรด: A
```

ตั้งแต่นั้นไป รัน test แบบปกติ (ไม่มี `-update`) จะแค่**เปรียบเทียบ**ผลลัพธ์กับไฟล์นี้:

```bash
go test -v ./report/... -run TestGenerateReport_Golden
```

```
=== RUN   TestGenerateReport_Golden
--- PASS: TestGenerateReport_Golden (0.00s)
PASS
```

ข้อดีของ pattern นี้คือ **ไม่ต้องเขียนค่าคาดหวังก้อนใหญ่ปนอยู่ในโค้ด Go** ทำให้โค้ด test อ่านง่ายขึ้นมาก และเมื่อ format ผลลัพธ์เปลี่ยนไปจริงๆ (โดยตั้งใจ) ก็แค่รันด้วย `-update` อีกครั้งเพื่ออัปเดตไฟล์ต้นแบบ แทนที่จะต้องแก้ string literal ยาวๆ ในโค้ดด้วยมือ — pattern นี้ใช้บ่อยมากในการทดสอบ code generator, template rendering, และ CLI tool ที่มี output แบบข้อความยาวๆ

---

## 5. Benchmark พื้นฐาน: `func BenchmarkXxx(b *testing.B)` และ `b.N`

นอกจากการทดสอบว่าโค้ด**ถูกต้อง**หรือไม่ (`testing.T`) package `testing` ยังมีเครื่องมือวัด**ประสิทธิภาพ**ในตัวด้วย เรียกว่า **Benchmark** ผ่านฟังก์ชันที่ตั้งชื่อขึ้นต้นด้วย `Benchmark` และรับ `*testing.B`:

```go
func BenchmarkXxx(b *testing.B) {
	for i := 0; i < b.N; i++ {
		// โค้ดที่ต้องการวัดความเร็ว
	}
}
```

กฎการตั้งชื่อและไฟล์เหมือนกับ `TestXxx` ทุกประการ (อยู่ในไฟล์ `_test.go`, ชื่อขึ้นต้นด้วยตัวใหญ่ตามด้วย `Benchmark`) จุดที่ต่างและสำคัญมากคือ **`b.N`**

**`b.N` คือจำนวนรอบที่ Go testing framework ตัดสินใจเองว่าควรรันฟังก์ชันกี่ครั้ง** เพื่อให้ได้ผลวัดที่แม่นยำและเสถียรทางสถิติ — เราไม่ได้เป็นคนกำหนดค่านี้เอง! กลไกภายในทำงานประมาณนี้: Go จะเริ่มรันด้วย `b.N` น้อยๆ ก่อน วัดเวลาที่ใช้ ถ้าเวลารวมยังน้อยเกินไปที่จะวัดได้แม่นยำ (ค่า default คือน้อยกว่าประมาณ 1 วินาที) framework จะเพิ่ม `b.N` แล้วรันซ้ำอีกครั้ง วนแบบนี้ไปเรื่อยๆ จนกว่าจะได้เวลารวมที่นานพอจะคำนวณ "เวลาเฉลี่ยต่อครั้ง" ได้แม่นยำ

หน้าที่ของเราแค่เขียน loop `for i := 0; i < b.N; i++ { ... }` ครอบโค้ดที่ต้องการวัด แล้วปล่อยให้ framework จัดการเรื่องจำนวนรอบและสถิติให้ทั้งหมด

---

## 6. รัน Benchmark ด้วย `go test -bench` และ `-benchmem`

Benchmark **ไม่รันอัตโนมัติ**พร้อมกับ `go test` ธรรมดา (ต่างจาก `TestXxx` ที่รันเสมอ) ต้องระบุ flag `-bench` เพื่อสั่งให้รันโดยเฉพาะ:

```bash
go test -bench=.
```

`-bench=.` หมายถึง "รัน benchmark ทุกตัวที่ชื่อ match กับ regular expression `.`" (คือ match ทุกตัว) ถ้าต้องการรันเฉพาะบางตัวก็ระบุ pattern แคบลงได้เหมือน `-run` ของ test ธรรมดา

flag ที่มีประโยชน์มากอีกตัวคือ **`-benchmem`** ซึ่งเพิ่มรายงานการใช้ **memory allocation** เข้ามาด้วย (จำนวน bytes และจำนวนครั้งที่ allocate ต่อการรันหนึ่งครั้ง) — สำคัญมากเพราะบางครั้งโค้ดที่เร็วกว่าในแง่เวลา อาจ allocate memory เยอะกว่ามาก ซึ่งส่งผลเสียต่อ garbage collector ในระยะยาว (เรื่อง GC จะเรียนเจาะลึกใน **Part 055**)

```bash
go test -bench=. -benchmem
```

โดยทั่วไปเวลารัน benchmark ควรใช้ `-run=^$` ควบคู่ไปด้วยเพื่อ**ปิดการรัน test ปกติทั้งหมด** (เพราะ `go test` จะรันทั้ง test และ benchmark พร้อมกันถ้าไม่ระบุ) ทำให้ผลลัพธ์ที่ได้มีแต่ตัวเลข benchmark ล้วนๆ ไม่ปนกับ log ของ test:

```bash
go test -bench=. -benchmem -run=^$
```

รูปแบบผลลัพธ์ของ benchmark มีคอลัมน์มาตรฐานที่ต้องอ่านให้เป็น:

```
BenchmarkConcatBuilder-4   436197   2564 ns/op   3320 B/op   9 allocs/op
```

| ส่วน | ความหมาย |
|---|---|
| `BenchmarkConcatBuilder-4` | ชื่อ benchmark ตามด้วย `-4` คือจำนวน CPU core ที่ใช้รัน (`GOMAXPROCS`) |
| `436197` | จำนวนรอบ (`b.N`) ที่ framework ตัดสินใจรันจริง |
| `2564 ns/op` | เวลาเฉลี่ยต่อการรันหนึ่งครั้ง (nanosecond per operation) — ตัวเลขหลักที่ใช้เปรียบเทียบความเร็ว |
| `3320 B/op` | จำนวน bytes ที่ถูก allocate โดยเฉลี่ยต่อการรันหนึ่งครั้ง (ปรากฏเมื่อใช้ `-benchmem`) |
| `9 allocs/op` | จำนวนครั้งที่เกิดการ allocate memory โดยเฉลี่ยต่อการรันหนึ่งครั้ง (ปรากฏเมื่อใช้ `-benchmem`) |

---

## 7. กับดักของ Benchmark: Dead Code Elimination และวิธีป้องกันด้วย `sink`

นี่คือกับดักที่มือใหม่เขียน benchmark เจอบ่อยที่สุดจนได้ตัวเลขที่ผิดพลาดโดยไม่รู้ตัว: **compiler ของ Go ฉลาดพอที่จะสังเกตเห็นว่าผลลัพธ์ของฟังก์ชันที่ถูกเรียกใน benchมark ไม่ได้ถูกใช้งานที่ไหนต่อเลย** แล้วอาจ**ตัดการเรียกฟังก์ชันทั้งหมดทิ้งไปเลย** (เทคนิคนี้เรียกว่า **dead code elimination** เป็นการ optimize มาตรฐานที่ compiler ภาษาต่างๆ ทำกัน) ผลคือ benchmark วัดเวลาของ**ลูปเปล่าๆ ที่แทบไม่ทำอะไรเลย** แทนที่จะวัดฟังก์ชันจริงๆ ทำให้ได้ตัวเลขเร็วผิดปกติที่ไม่มีความหมาย

โค้ด benchmark ที่ **เสี่ยงมีปัญหานี้**:

```go
// อันตราย: ผลลัพธ์ของ ConcatBuilder ไม่ถูกใช้ที่ไหนเลย compiler อาจตัดทิ้งได้
func BenchmarkConcatBuilder_Risky(b *testing.B) {
	words := genWords(200)
	for i := 0; i < b.N; i++ {
		ConcatBuilder(words) // ผลลัพธ์ถูกทิ้งไปเฉยๆ ไม่ถูก assign ที่ไหนเลย
	}
}
```

วิธีป้องกันมาตรฐานที่ใช้กันทั่วไปคือสร้าง **ตัวแปรระดับ package (package-level variable)** ขึ้นมาทำหน้าที่เป็น "ที่รับผลลัพธ์" (เรียกกันติดปากว่า **sink**) แล้ว assign ผลลัพธ์จากทุกรอบเข้าไปที่ตัวแปรนี้เสมอ:

```go
// sink คือตัวแปรระดับ package ใช้ "ดูดซับ" ผลลัพธ์จากฟังก์ชันที่ถูก benchmark
// เพื่อป้องกันไม่ให้ compiler ทำ dead code elimination ตัดการเรียกฟังก์ชันทิ้งไป
// เพราะเห็นว่าไม่มีใครใช้ผลลัพธ์เลย
var sink string

func BenchmarkConcatBuilder(b *testing.B) {
	words := genWords(200)
	var r string
	for i := 0; i < b.N; i++ {
		r = ConcatBuilder(words)
	}
	sink = r // assign เข้าตัวแปรระดับ package หลังจบลูป ป้องกัน compiler ตัดโค้ดทิ้ง
}
```

เหตุผลที่การ assign เข้าตัวแปร**ระดับ package** (ไม่ใช่ตัวแปรใน local scope ธรรมดา) ถึงป้องกันปัญหานี้ได้ เพราะตัวแปร package-level อาจถูกโค้ดส่วนอื่นของโปรแกรมมองเห็นและใช้งานได้เสมอในทางทฤษฎี (แม้ในทางปฏิบัติจะไม่มีใครใช้เลยก็ตาม) compiler จึง**ไม่กล้า**ตัดการคำนวณที่นำไปสู่การ assign ค่านั้นทิ้งไป ต่างจากตัวแปร local ที่ compiler รู้แน่ชัดว่าไม่มีใครใช้ต่อเลยถ้าไม่ได้ `return` หรือส่งออกไปไหน

**กฎทองคำที่ต้องจำ**: ทุกครั้งที่เขียน benchmark ของฟังก์ชันที่คืนค่า ให้ assign ผลลัพธ์เข้าตัวแปร local ก่อน แล้ว assign ตัวแปร local นั้นเข้าตัวแปรระดับ package อีกทีหลังจบลูปเสมอ เพื่อความมั่นใจว่าตัวเลขที่วัดได้เป็นของจริง ไม่ใช่ผลจาก dead code elimination

---

## 8. `b.ResetTimer()`: ตัดเวลาการเตรียมข้อมูลออกจากผลวัด

บางครั้งก่อนเริ่มวัดผล benchmark ต้องมีขั้นตอนเตรียมข้อมูลที่ใช้เวลาพอสมควร (เช่น สร้าง slice ขนาดใหญ่, เปิดไฟล์, สร้าง struct ที่ซับซ้อน) ถ้าปล่อยให้เวลาการเตรียมข้อมูลนี้ปนเข้าไปในผลวัด ตัวเลขที่ได้จะไม่สะท้อนความเร็วของฟังก์ชันที่ต้องการวัดจริงๆ

`b.ResetTimer()` คือคำสั่งที่บอก benchmark framework ว่า **"รีเซ็ตนาฬิกาจับเวลา ไม่ต้องนับเวลาที่ผ่านมาก่อนหน้านี้เลย"** — เรียกทันทีหลังขั้นตอนเตรียมข้อมูลเสร็จ ก่อนเข้าลูป `for i := 0; i < b.N; i++`:

```go
func genWords(n int) []string {
	words := make([]string, n)
	for i := 0; i < n; i++ {
		words[i] = "word" + strconv.Itoa(i)
	}
	return words
}

func BenchmarkConcatBuilder(b *testing.B) {
	words := genWords(200) // ขั้นตอนเตรียมข้อมูล -- ไม่ควรถูกนับเข้าผลวัด
	b.ResetTimer()          // รีเซ็ตนาฬิกา เริ่มนับเวลาใหม่จากตรงนี้
	var r string
	for i := 0; i < b.N; i++ {
		r = ConcatBuilder(words)
	}
	sink = r
}
```

ถ้าไม่เรียก `b.ResetTimer()` เวลาที่ใช้สร้าง `words` ด้วย `genWords(200)` จะถูกนับรวมเข้าไปในรอบแรกของการวัด ทำให้ผลลัพธ์คลาดเคลื่อน (แม้จะเป็นสัดส่วนที่น้อยมากเมื่อ `b.N` มีค่าสูงพอ แต่สำหรับการเตรียมข้อมูลที่หนักจริงๆ เช่น เชื่อมต่อฐานข้อมูลหรืออ่านไฟล์ขนาดใหญ่ ผลกระทบจะเห็นชัดเจนมาก)

---

## 9. ตัวอย่างเต็ม: เทียบประสิทธิภาพสองวิธีต่อ String

มาดูตัวอย่างที่รวมทุกเทคนิคเข้าด้วยกัน — เปรียบเทียบสองวิธีต่อ string ใน Go: การใช้ operator `+` ธรรมดาในลูป เทียบกับการใช้ `strings.Builder` (ทบทวนจาก **Part 019**):

```go
// strutil.go
package strutil

import "strings"

// ConcatPlus ต่อ string ทั้งหมดใน slice ด้วย operator + ธรรมดา
func ConcatPlus(words []string) string {
	result := ""
	for _, w := range words {
		result += w
	}
	return result
}

// ConcatBuilder ต่อ string ทั้งหมดใน slice ด้วย strings.Builder
func ConcatBuilder(words []string) string {
	var b strings.Builder
	for _, w := range words {
		b.WriteString(w)
	}
	return b.String()
}
```

```go
// strutil_bench_test.go
package strutil

import (
	"strconv"
	"testing"
)

var sink string

func genWords(n int) []string {
	words := make([]string, n)
	for i := 0; i < n; i++ {
		words[i] = "word" + strconv.Itoa(i)
	}
	return words
}

func BenchmarkConcatPlus(b *testing.B) {
	words := genWords(200)
	b.ResetTimer()
	var r string
	for i := 0; i < b.N; i++ {
		r = ConcatPlus(words)
	}
	sink = r
}

func BenchmarkConcatBuilder(b *testing.B) {
	words := genWords(200)
	b.ResetTimer()
	var r string
	for i := 0; i < b.N; i++ {
		r = ConcatBuilder(words)
	}
	sink = r
}
```

รันด้วย:

```bash
go test -bench=. -benchmem -run=^$ ./strutil/...
```

ผลลัพธ์จริงจากการทดสอบ (ตัวเลขจะต่างกันไปตามเครื่อง แต่สัดส่วนความต่างจะใกล้เคียงกันเสมอ):

```
goos: linux
goarch: amd64
pkg: strutil
cpu: Intel(R) Xeon(R) Processor @ 2.80GHz
BenchmarkConcatPlus-4      	   14719	     80399 ns/op	  130664 B/op	     199 allocs/op
BenchmarkConcatBuilder-4   	  436197	      2564 ns/op	    3320 B/op	       9 allocs/op
PASS
```

วิเคราะห์ผลลัพธ์:

- **`ConcatBuilder` เร็วกว่า `ConcatPlus` ประมาณ 31 เท่า** (80399 ns/op ÷ 2564 ns/op) สำหรับการต่อคำ 200 คำ
- **`ConcatBuilder` ใช้ memory น้อยกว่ามาก**: 3,320 bytes เทียบกับ 130,664 bytes (น้อยกว่าประมาณ 39 เท่า) และ allocate เพียง 9 ครั้ง เทียบกับ 199 ครั้ง
- เหตุผลเบื้องหลัง: ทุกครั้งที่ใช้ `result += w` กับ string ปกติ Go ต้อง **สร้าง string ใหม่ทั้งก้อนทุกครั้ง** เพราะ string ใน Go เป็น **immutable** (ทบทวนจาก **Part 019**) ทำให้ยิ่งต่อคำเยอะยิ่งเสียเวลา copy ข้อมูลเดิมซ้ำไปเรื่อยๆ (เป็น O(n²) โดยรวม) ในขณะที่ `strings.Builder` ใช้ buffer ภายในที่ขยายขนาดแบบ amortized (คล้ายกับที่ slice ขยายขนาดที่เรียนใน **Part 006**) ทำให้การต่อ string โดยรวมเป็น O(n) เท่านั้น

นี่คือตัวอย่างที่สมบูรณ์แบบของการใช้ benchmark เพื่อ**พิสูจน์ด้วยตัวเลขจริง**ว่าทำไม `strings.Builder` ถึงเป็นวิธีที่แนะนำเมื่อต้องต่อ string จำนวนมากในลูป แทนที่จะเชื่อแค่คำแนะนำลอยๆ โดยไม่มีหลักฐาน — นี่คือคุณค่าที่แท้จริงของการเขียน benchmark ในโค้ด production: ช่วยตัดสินใจเลือก implementation จากข้อมูลจริง ไม่ใช่จากความรู้สึกหรือสมมติฐาน

---

## สรุปสิ่งที่ได้เรียนในบทนี้

- **Table-Driven Test** คือ pattern มาตรฐานของ Go: เก็บกรณีทดสอบเป็น slice ของ struct นิรนาม แล้ววนด้วย `t.Run` ต่อหนึ่งแถว ลดโค้ดซ้ำและเพิ่มกรณีทดสอบใหม่ได้ง่ายมาก
- ตั้งแต่ **Go 1.22** ปัญหา loop variable capture ใน closure ถูกแก้แล้วโดยธรรมชาติของภาษา แต่การเขียน `tt := tt` ยังเป็นนิสัยที่ดีเพื่อความชัดเจนและรองรับโค้ดที่ยังรันบนเวอร์ชันเก่า
- **Golden File Testing** เก็บผลลัพธ์ต้นแบบไว้ในโฟลเดอร์ `testdata/` แล้วเปรียบเทียบ เหมาะกับผลลัพธ์ที่ใหญ่หรือซับซ้อนเกินกว่าจะเขียนเป็น string literal ในโค้ด — ใช้ flag กำหนดเอง (เช่น `-update`) เพื่ออัปเดตไฟล์ต้นแบบเมื่อจำเป็น
- **Benchmark** เขียนผ่าน `func BenchmarkXxx(b *testing.B)` โดย **`b.N`** คือจำนวนรอบที่ framework ปรับให้เองเพื่อความแม่นยำทางสถิติ ไม่ใช่ค่าที่เรากำหนด
- รันด้วย `go test -bench=. -benchmem -run=^$` — `-bench` เปิดใช้งาน benchmark, `-benchmem` แสดงข้อมูล memory allocation, `-run=^$` ปิดการรัน test ปกติเพื่อให้เห็นแต่ผล benchmark
- **กับดักสำคัญที่สุด**: compiler อาจตัดโค้ดที่ผลลัพธ์ไม่ถูกใช้งานทิ้งไป (dead code elimination) ทำให้ตัวเลข benchmark ผิดเพี้ยน — ป้องกันด้วยการ assign ผลลัพธ์เข้าตัวแปรระดับ package (`sink`) เสมอ
- **`b.ResetTimer()`** ใช้ตัดเวลาการเตรียมข้อมูลออกจากผลวัด เรียกทันทีก่อนเข้าลูปหลักของ benchmark
- Benchmark ช่วยเปรียบเทียบ implementation สองแบบด้วยตัวเลขจริง (เช่น `ConcatPlus` vs `ConcatBuilder` ที่ต่างกันถึง 30 เท่า) แทนการเดาหรือเชื่อคำแนะนำลอยๆ

## แบบฝึกหัดท้ายบท

1. เขียน table-driven test สำหรับฟังก์ชัน `Percentage` จากแบบฝึกหัด Part 033 ให้ครอบคลุมอย่างน้อย 5 กรณี (รวมกรณี error) โดยใช้ struct นิรนามที่มี field `wantErr bool` เพิ่มเข้ามาเพื่อระบุว่ากรณีนั้นควร error หรือไม่
2. เขียนฟังก์ชัน `Fibonacci(n int) int` สองแบบ: แบบ recursive ธรรมดา และแบบ iterative (วนลูป) แล้วเขียน benchmark เปรียบเทียบทั้งสองแบบด้วย `n = 30` พร้อมใช้ `-benchmem` วิเคราะห์ว่าทำไมผลต่างกันมากขนาดนั้น
3. ลองลบ `b.ResetTimer()` ออกจาก benchmark ในหัวข้อ 9 แล้วเปลี่ยน `genWords` ให้ใช้ `time.Sleep(10 * time.Millisecond)` จำลองการเตรียมข้อมูลที่ช้ามาก สังเกตว่าผลลัพธ์ของ benchmark เปลี่ยนไปอย่างไรเมื่อเทียบกับตอนมี `b.ResetTimer()`
4. เขียน golden file test สำหรับฟังก์ชันที่สร้าง JSON string จาก struct (ใช้ความรู้ `encoding/json` จาก **Part 025**) แล้วทดลองใช้ flag `-update` เพื่อสร้างและอัปเดตไฟล์ golden
5. ทดลองลบตัวแปร `sink` ออกจาก benchmark ในหัวข้อ 9 แล้วเปลี่ยนให้เรียกฟังก์ชันในลูปโดยไม่ assign ผลลัพธ์ที่ไหนเลย (`ConcatBuilder(words)` เฉยๆ) รันด้วย `-benchmem` เทียบผลลัพธ์ดูว่าตัวเลข `ns/op` เปลี่ยนไปน่าสงสัยหรือไม่ (คำเตือน: ผลลัพธ์อาจแตกต่างกันไปตามเวอร์ชัน compiler ที่ใช้ ให้สังเกตหลักการมากกว่าตัวเลขที่แน่นอน)
6. ค้นคว้าเพิ่มเติม: อ่านเกี่ยวกับ `testing.B.RunParallel` สำหรับเขียน benchmark ที่รันแบบขนานหลาย goroutine พร้อมกัน (จะเข้าใจเต็มที่หลังเรียน **Part 036-039**) แล้วลองเขียนโครงร่างคร่าวๆ ว่าจะนำไปใช้กับฟังก์ชันในบทนี้ตัวไหนได้บ้าง

---

**ต่อไป**: [Part 035 — Mocking และ Test Doubles](./035-mocking-and-test-doubles.md)
