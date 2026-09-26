# Part 082: Test Coverage

> ภาคที่ 7: Testing, Tooling, Performance — ตอนที่ 4 จาก 9 (Part 79–87)

## สารบัญของบทนี้

1. Test Coverage คืออะไร วัดอะไรกันแน่
2. `go test -cover`: ดูเปอร์เซ็นต์แบบเร็ว
3. `-coverprofile`: เก็บรายละเอียดระดับบรรทัดไว้วิเคราะห์ต่อ
4. `go tool cover -func`: ดู Coverage แยกรายฟังก์ชัน
5. `go tool cover -html`: รายงานภาพสี บรรทัดไหนถูกทดสอบ บรรทัดไหนไม่ถูก
6. ทำไม 100% Coverage ไม่ใช่เป้าหมาย: พิสูจน์ด้วยบั๊กจริง
7. `-covermode`: `set`, `count`, `atomic` ต่างกันอย่างไร
8. ตั้ง Coverage Threshold Gate สำหรับ CI
9. รวม Coverage ข้ามหลาย Package ด้วย `-coverpkg=./...`
10. ข้อควรระวังและ Best Practice
11. สรุปสิ่งที่ได้เรียนในบทนี้
12. แบบฝึกหัดท้ายบท

---

## 1. Test Coverage คืออะไร วัดอะไรกันแน่

**Part 033** เกริ่นถึง flag `-cover` ไปสั้นๆ แล้ว และท้ายบทนั้นก็ทิ้งคำเตือนไว้ว่า "coverage สูงไม่ได้แปลว่า test ดีเสมอไป" — บทนี้จะอธิบายเรื่องนั้นให้ครบถ้วน พร้อมพิสูจน์ด้วยโค้ดจริงว่าทำไมถึงเป็นเช่นนั้น

**Test coverage** (หรือ **code coverage**) คือเปอร์เซ็นต์ของ **statement** (คำสั่งในโค้ด) ที่ถูก**ทำงานจริง**อย่างน้อยหนึ่งครั้งระหว่างรัน test suite ทั้งหมด สิ่งสำคัญที่ต้องเข้าใจให้ชัดตั้งแต่ต้นคือ:

> **Coverage วัดแค่ว่า "บรรทัดโค้ดถูกรันหรือไม่" มันไม่ได้วัดว่า "ผลลัพธ์ที่ได้ถูกตรวจสอบอย่างครบถ้วนหรือไม่"**

พูดง่ายๆ คือ ถ้าฟังก์ชันหนึ่งถูกเรียกจาก test แม้แต่ครั้งเดียว โดยที่ test นั้น**ไม่ได้เช็คค่า return เลยด้วยซ้ำ** บรรทัดในฟังก์ชันนั้นก็ยังนับว่า "ถูก cover" อยู่ดี — coverage เป็นเพียง **ตัวชี้วัดว่า "โค้ดส่วนไหนที่ไม่เคยถูกแตะเลย"** (ซึ่งมีประโยชน์มากในการหาจุดบอด) ไม่ใช่ตัวชี้วัดคุณภาพของการตรวจสอบ (assertion) ภายใน test เอง

---

## 2. `go test -cover`: ดูเปอร์เซ็นต์แบบเร็ว

วิธีดู coverage ที่เร็วที่สุดคือเพิ่ม flag `-cover` เข้าไปตอนรัน test ปกติ:

```bash
go test -cover ./...
```

มาดูตัวอย่างสองแพ็กเกจที่จะใช้ตลอดบทนี้ — package แรกชื่อ `mathx` ที่มี 3 ฟังก์ชันแต่ทดสอบไม่ครบ:

```go
// mathx/mathx.go
package mathx

// Add คืนผลบวกของ a และ b
func Add(a, b int) int {
	return a + b
}

// Sub คืนผลลบของ a และ b
func Sub(a, b int) int {
	return a - b
}

// Abs คืนค่าสัมบูรณ์ของ n
func Abs(n int) int {
	if n < 0 {
		return -n
	}
	return n
}
```

```go
// mathx/mathx_test.go
package mathx

import "testing"

// สังเกตว่าไฟล์นี้ทดสอบแค่ Add กับ Abs เท่านั้น ไม่ได้ทดสอบ Sub เลย
// ตั้งใจเว้นไว้เพื่อสาธิตว่า coverage ของ package นี้จะไม่ถึง 100%
func TestAdd(t *testing.T) {
	if got := Add(2, 3); got != 5 {
		t.Errorf("Add(2, 3) = %d, want 5", got)
	}
}

func TestAbs(t *testing.T) {
	tests := []struct {
		name string
		n    int
		want int
	}{
		{"บวก", 5, 5},
		{"ลบ", -5, 5},
		{"ศูนย์", 0, 0},
	}
	for _, tt := range tests {
		t.Run(tt.name, func(t *testing.T) {
			if got := Abs(tt.n); got != tt.want {
				t.Errorf("Abs(%d) = %d, want %d", tt.n, got, tt.want)
			}
		})
	}
}
```

รันด้วย `-cover`:

```bash
go test -cover ./mathx/...
```

```
ok  	part082demo/mathx	0.003s	coverage: 80.0% of statements
```

**80.0%** — เพราะฟังก์ชัน `Sub` (2 บรรทัดจาก 10 statement ทั้งหมดใน package) ไม่เคยถูกเรียกจาก test เลย ตัวเลขนี้คำนวณจากสัดส่วน "statement ที่ถูกรันอย่างน้อยหนึ่งครั้ง / statement ทั้งหมดใน package"

---

## 3. `-coverprofile`: เก็บรายละเอียดระดับบรรทัดไว้วิเคราะห์ต่อ

`-cover` ให้แค่ตัวเลขสรุปตัวเดียว แต่ถ้าอยากรู้ว่า **บรรทัดไหนกันแน่** ที่ยังไม่ถูกทดสอบ ต้องใช้ `-coverprofile` เพื่อบันทึกข้อมูล coverage แบบละเอียดลงไฟล์:

```bash
go test -coverprofile=coverage.out ./mathx/...
```

```
ok  	part082demo/mathx	0.003s	coverage: 80.0% of statements
```

คำสั่งนี้ยังสร้างไฟล์ `coverage.out` ขึ้นมาด้วย ซึ่งเป็นไฟล์ text format พิเศษที่ `go tool cover` เข้าใจ (ไม่ได้ออกแบบมาให้มนุษย์อ่านตรงๆ):

```
mode: set
part082demo/mathx/mathx.go:4.24,6.2 1 1
part082demo/mathx/mathx.go:9.24,11.2 1 0
part082demo/mathx/mathx.go:14.19,16.2 1 1
part082demo/mathx/mathx.go:16.2,18.2 1 1
```

แต่ละบรรทัดหมายถึง "ช่วงบรรทัด:คอลัมน์" ของ statement block หนึ่งก้อน ตามด้วยจำนวน statement ในก้อนนั้น และค่าสุดท้ายคือ **ถูกรันกี่ครั้ง** (`0` = ไม่เคยถูกรันเลย) ไฟล์นี้เป็น**ข้อมูลดิบ**ที่ต้องป้อนต่อให้เครื่องมืออื่นในหัวข้อถัดไปเพื่อแปลงเป็นรายงานที่อ่านง่าย

---

## 4. `go tool cover -func`: ดู Coverage แยกรายฟังก์ชัน

`go tool cover` เป็นเครื่องมือแยกต่างหาก (ไม่ใช่ subcommand ของ `go test`) ที่อ่านไฟล์ profile แล้วสร้างรายงานได้หลายรูปแบบ แบบแรกที่มีประโยชน์มากคือ `-func` ซึ่งแตกรายละเอียด coverage **แยกเป็นรายฟังก์ชัน**:

```bash
go tool cover -func=coverage.out
```

```
part082demo/mathx/mathx.go:4:			Add		100.0%
part082demo/mathx/mathx.go:9:			Sub		0.0%
part082demo/mathx/mathx.go:14:			Abs		100.0%
total:						(statements)	80.0%
```

เห็นชัดเจนทันทีว่า **`Sub` มี coverage 0.0%** — ไม่เคยถูกเรียกจาก test เลยสักครั้ง ในขณะที่ `Add` และ `Abs` ได้ 100.0% การรายงานแบบนี้มีประโยชน์มากตอนต้องการหาว่า "ฟังก์ชันไหนในระบบที่ไม่มีการทดสอบรองรับเลย" อย่างรวดเร็วโดยไม่ต้องไล่อ่านโค้ดทั้งไฟล์

---

## 5. `go tool cover -html`: รายงานภาพสี บรรทัดไหนถูกทดสอบ บรรทัดไหนไม่ถูก

รูปแบบรายงานที่ใช้งานบ่อยที่สุดในทางปฏิบัติคือ **HTML report** ซึ่งแสดงโค้ดต้นฉบับพร้อมไฮไลต์สี: **สีเขียว** คือ statement ที่ถูกทดสอบ, **สีแดง** คือ statement ที่ไม่เคยถูกทดสอบเลย, สีเทาคือโค้ดที่ไม่นับ (เช่น comment หรือ declaration ที่ไม่ใช่ executable statement)

```bash
go tool cover -html=coverage.out -o coverage.html
```

คำสั่งนี้สร้างไฟล์ `coverage.html` ขึ้นมา — ถ้ารันโดยไม่ระบุ `-o` มันจะพยายามเปิดเบราว์เซอร์ให้อัตโนมัติทันที ซึ่งสะดวกมากบนเครื่องพัฒนาที่มี GUI แต่ **ในสภาพแวดล้อมที่ไม่มีเบราว์เซอร์** (เช่น container, remote server ผ่าน SSH, หรือ sandbox แบบที่ใช้เขียนบทเรียนนี้) ต้องระบุ `-o` เพื่อบันทึกเป็นไฟล์แล้วเปิดดูภายหลัง หรือ copy กลับไปเปิดบนเครื่องที่มี GUI

> **ยืนยันแล้วจริง**: คำสั่งข้างต้นรันสำเร็จบนเครื่องที่ใช้เขียนบทเรียนนี้ ได้ไฟล์ `coverage.html` ขนาดประมาณ 4 KB ที่มีโครงสร้าง HTML/CSS/JavaScript ครบถ้วน (ฝัง syntax highlighting และปุ่มสลับดูแต่ละไฟล์ในตัว) พร้อมเปิดดูในเบราว์เซอร์ได้ทันที

โครงสร้างของ HTML report ประกอบด้วย:

- **Dropdown เลือกไฟล์** ที่มุมบนซ้าย (ถ้ามีหลายไฟล์ในรายงาน) สลับดูโค้ดแต่ละไฟล์ได้
- **เนื้อโค้ดต้นฉบับทั้งไฟล์** พร้อมพื้นหลังสีตามที่อธิบายไว้ข้างต้น
- Comment และ declaration ที่ไม่ใช่ executable code จะไม่ถูกไฮไลต์เลย (เป็นสีเทาเข้มปกติ) เพราะ coverage วัดเฉพาะ statement ที่ "รันได้" เท่านั้น

รูปแบบนี้เหมาะมากสำหรับการ **ไล่ดูโค้ดจริงทีละไฟล์** เพื่อหาว่า branch ไหนของ `if`/`switch` ที่ไม่เคยถูกทดสอบ ซึ่งบอกรายละเอียดมากกว่า `-func` ที่ให้แค่ตัวเลขสรุปต่อฟังก์ชัน

---

## 6. ทำไม 100% Coverage ไม่ใช่เป้าหมาย: พิสูจน์ด้วยบั๊กจริง

นี่คือหัวใจสำคัญที่สุดของบทนี้ มาดูตัวอย่างฟังก์ชันที่**ได้ coverage 100%** แต่**มีบั๊กจริงที่ยังไม่ถูกจับได้**:

```go
// membership/membership.go
package membership

// IsValidAge ตรวจสอบว่าอายุอยู่ในเกณฑ์ที่ระบบสมัครสมาชิกยอมรับหรือไม่
// สเปกที่แท้จริง (ตามที่ทีมธุรกิจกำหนด) คือ "รับสมัครเฉพาะอายุ 18-65 ปีเท่านั้น"
// แต่โค้ดนี้มีบั๊ก: ลืมเช็ค upper bound (65) ไปเลย — ตั้งใจใส่บั๊กไว้เพื่อสาธิตบทนี้
func IsValidAge(age int) bool {
	if age >= 18 {
		return true // BUG: ควรเช็คด้วยว่า age <= 65 ด้วย แต่ลืมใส่เงื่อนไขนี้
	}
	return false
}
```

และ test ที่ **ดูเผินๆ เหมือนครอบคลุมดี** เพราะมีทั้งกรณี `true` และ `false`:

```go
// membership/membership_test.go
package membership

import "testing"

// TestIsValidAge_LooksComplete ดูเผินๆ เหมือนครอบคลุมดี: มีทั้งกรณี true และ false
// รันแล้วจะได้ coverage 100% ของฟังก์ชัน IsValidAge เพราะทุกบรรทัด (ทุก branch)
// ถูกทำงานอย่างน้อยหนึ่งครั้ง — แต่ "100% coverage" ในที่นี้ไม่ได้แปลว่าโค้ดถูกต้อง!
func TestIsValidAge_LooksComplete(t *testing.T) {
	tests := []struct {
		name string
		age  int
		want bool
	}{
		{"อายุน้อยกว่า 18", 10, false},
		{"อายุพอดี 18", 18, true},
		{"อายุผู้ใหญ่ทั่วไป", 30, true},
	}
	for _, tt := range tests {
		t.Run(tt.name, func(t *testing.T) {
			if got := IsValidAge(tt.age); got != tt.want {
				t.Errorf("IsValidAge(%d) = %v, want %v", tt.age, got, tt.want)
			}
		})
	}
}
```

รันดู coverage:

```bash
go test -cover ./membership/...
```

```
ok  	part082demo/membership	0.003s	coverage: 100.0% of statements
```

**100.0% เต็ม!** ทั้ง 2 บรรทัดของ `IsValidAge` (บรรทัด `return true` และ `return false`) ถูกทำงานอย่างน้อยหนึ่งครั้งจาก 3 กรณีทดสอบ — ตาม**นิยามของ coverage** นี่คือ coverage ที่สมบูรณ์แบบที่สุดเท่าที่จะเป็นไปได้

**แต่ฟังก์ชันนี้มีบั๊กจริง**: ลองเรียก `IsValidAge(200)` (อายุ 200 ปี ซึ่งไม่ควรผ่านตามสเปกที่กำหนดไว้ 18-65 ปี) จะได้ผลลัพธ์ `true` ทั้งที่ควรเป็น `false` — เพราะโค้ดลืมเช็ค upper bound (`age <= 65`) ไปเลย

**ทำไม coverage ถึงไม่เห็นปัญหานี้?** เพราะฟังก์ชันมีแค่ 2 branch (`age >= 18` เป็นจริง หรือเป็นเท็จ) และ test case ทั้ง 3 ตัวที่มีอยู่ครอบคลุมทั้ง 2 branch นั้นแล้ว **ปัญหาไม่ได้อยู่ที่ "branch ที่ไม่ถูกทดสอบ" แต่อยู่ที่ "branch ที่ควรมีแต่ไม่มีอยู่ในโค้ดเลยตั้งแต่แรก"** — coverage วัดจากโครงสร้างโค้ดที่มีอยู่จริงเท่านั้น มันไม่มีทางรู้ได้เลยว่าโค้ดควรมีเงื่อนไขเพิ่มเติมที่ผู้เขียนลืมใส่ ไม่ว่าจะมี test เพิ่มอีกกี่ร้อยกรณีก็ตาม ถ้าไม่มีสักกรณีที่ทดสอบด้วยอายุมากกว่า 65 ปี บั๊กนี้จะไม่มีวันถูกจับได้เลย

นี่คือบทเรียนสำคัญที่สุด: **coverage บอกได้แค่ "โค้ดที่มีอยู่ถูกรันหรือยัง" มันไม่เคยบอกได้ว่า "test case ครอบคลุมทุกสถานการณ์ทางธุรกิจที่ควรจะเป็นหรือยัง"** การเขียน test ที่ดีต้องอาศัยการวิเคราะห์ **specification/business rule** ควบคู่ไปกับการดู coverage เสมอ — เทคนิคจาก **Part 034** เรื่อง table-driven test ที่ครอบคลุม **edge case** อย่างรอบด้าน (ค่าน้อยสุด, มากสุด, ค่าที่ขอบเขต, ค่านอกขอบเขต) ยังคงเป็นสิ่งที่ต้องคิดด้วยตัวเองเสมอ ไม่ใช่แค่ไล่ให้ตัวเลข coverage ขึ้นถึง 100%

แก้บั๊กให้ถูกต้อง:

```go
func IsValidAge(age int) bool {
	return age >= 18 && age <= 65
}
```

และเพิ่ม test case ที่ครอบคลุม upper bound:

```go
{"อายุเกิน 65", 70, false},
{"อายุพอดี 65", 65, true},
```

> **ข้อสรุปเชิงปฏิบัติ**: ใช้ coverage เป็นเครื่องมือหา **จุดบอดที่ชัดเจน** (โค้ดที่ไม่เคยถูกแตะเลย เช่น `Sub` ในหัวข้อ 2) แต่อย่าใช้ตัวเลข coverage เป็นเป้าหมายสุดท้ายของคุณภาพ test ทีมที่ตั้งเป้า "ต้องได้ 100% coverage เท่านั้น" มักจบลงด้วยการเขียน test ที่ "เรียกฟังก์ชันแต่ไม่ assert อะไรที่มีความหมาย" แค่เพื่อไล่ตัวเลข ซึ่งแย่กว่าไม่มี test เลยด้วยซ้ำ เพราะทำให้ทีมรู้สึกปลอดภัยแบบผิดๆ (false sense of security)

---

## 7. `-covermode`: `set`, `count`, `atomic` ต่างกันอย่างไร

`go test -covermode` กำหนดว่าจะบันทึกข้อมูล coverage แบบไหนลงในไฟล์ profile มี 3 โหมด:

| โหมด | ความหมาย | ใช้เมื่อไหร่ |
|---|---|---|
| **`set`** (default) | บันทึกแค่ "ถูกรันหรือไม่" (0 หรือ 1) | การใช้งานทั่วไป เร็วที่สุด |
| **`count`** | บันทึก **จำนวนครั้ง** ที่แต่ละ statement ถูกรัน | ต้องการรู้ว่า statement ไหนถูกรันบ่อย/น้อยแค่ไหน (hint เชิง performance) |
| **`atomic`** | เหมือน `count` แต่ใช้ atomic operation ในการนับ (ทบทวน `sync/atomic` จาก **Part 040**) | **จำเป็นเมื่อรันคู่กับ `-race`** เพราะการรันแบบ concurrent (multiple goroutine เขียน counter เดียวกัน) ต้องการความถูกต้องแบบ thread-safe ไม่งั้นตัวเลขที่นับได้อาจผิดพลาดจาก race condition ในตัวนับเอง |

```bash
go test -race -covermode=atomic -coverprofile=coverage.out ./...
```

```
ok  	part082demo/mathx	1.013s	coverage: 80.0% of statements
ok  	part082demo/membership	1.017s	coverage: 100.0% of statements
```

> **กฎจำง่าย**: ถ้ารัน test คู่กับ `-race` (แนะนำเสมอตามที่เรียนใน **Part 044**) ให้ใช้ `-covermode=atomic` เสมอ — จริงๆ แล้วถ้าใส่ `-race` ไว้ `go test` จะ**บังคับเปลี่ยนเป็น `atomic` ให้อัตโนมัติ**อยู่แล้วแม้จะไม่ได้ระบุ `-covermode` เอง แต่การเข้าใจว่าทำไมถึงต้องเป็นแบบนั้นสำคัญกว่าการจำ default value เฉยๆ

---

## 8. ตั้ง Coverage Threshold Gate สำหรับ CI

ในหลายทีม การกำหนด **coverage threshold** (เกณฑ์ขั้นต่ำที่ยอมรับได้) เป็นส่วนหนึ่งของ CI pipeline (ปูทางสู่ **Part 098**) เพื่อป้องกันไม่ให้ coverage โดยรวมของโปรเจกต์ลดลงเรื่อยๆ โดยไม่มีใครสังเกตเห็น Go ไม่มี flag สำเร็จรูปสำหรับ "fail ถ้า coverage ต่ำกว่า X%" แต่เขียนสคริปต์สั้นๆ ครอบคำสั่งที่มีอยู่แล้วได้ไม่ยาก:

```bash
#!/usr/bin/env bash
# check_coverage.sh — gate coverage ให้ต่ำกว่า threshold ที่กำหนดไม่ได้ ใช้ใน CI (ดู Part 098)
set -euo pipefail

THRESHOLD="${1:-70}"

go test -coverprofile=coverage.out -coverpkg=./... ./... > /dev/null

# ดึงตัวเลขเปอร์เซ็นต์รวมจากบรรทัดสุดท้ายของ `go tool cover -func`
# รูปแบบบรรทัดนั้นคือ: total:  (statements)  92.3%
COVERAGE=$(go tool cover -func=coverage.out | tail -1 | awk '{print $3}' | tr -d '%')

echo "Total coverage: ${COVERAGE}% (threshold: ${THRESHOLD}%)"

# เปรียบเทียบเป็นทศนิยมด้วย awk (bash เทียบเลขทศนิยมตรงๆ ไม่ได้)
PASS=$(awk -v cov="$COVERAGE" -v th="$THRESHOLD" 'BEGIN { print (cov >= th) ? "1" : "0" }')

if [ "$PASS" != "1" ]; then
    echo "FAIL: coverage ${COVERAGE}% ต่ำกว่า threshold ${THRESHOLD}%"
    exit 1
fi

echo "PASS: coverage ผ่านเกณฑ์"
```

ทดสอบสคริปต์จริงกับ 2 threshold ต่างกัน (coverage รวมของทั้งสอง package คือ 92.3%):

```bash
chmod +x check_coverage.sh
./check_coverage.sh 70
```

```
Total coverage: 92.3% (threshold: 70%)
PASS: coverage ผ่านเกณฑ์
```

```bash
./check_coverage.sh 95
```

```
Total coverage: 92.3% (threshold: 95%)
FAIL: coverage 92.3% ต่ำกว่า threshold 95%
```

exit code เป็น `0` ในกรณีแรกและ `1` ในกรณีที่สอง — CI pipeline (เช่น GitHub Actions ใน **Part 098**) จะอ่าน exit code นี้เพื่อตัดสินว่า job นั้นผ่านหรือ fail:

```yaml
# ตัวอย่างการใช้ใน GitHub Actions step (รายละเอียดเต็มใน Part 098)
- name: Check coverage threshold
  run: ./check_coverage.sh 80
```

**เรื่องที่ต้องระวัง**: threshold แบบ "ตัวเลขรวมทั้งโปรเจกต์" มีข้อจำกัด — โปรเจกต์ที่มี coverage 95% อยู่แล้วอาจเพิ่มโค้ดใหม่ที่ coverage 0% เข้าไป แต่ตัวเลขรวมยังคงสูงพอที่จะผ่าน threshold ได้เพราะโค้ดเก่าที่ coverage สูงกลบไว้ ทีมที่จริงจังเรื่องนี้มักใช้เครื่องมือระดับสูงกว่าที่เช็ค **"coverage เฉพาะส่วนที่เปลี่ยนแปลงใน pull request"** (diff coverage) แทน ซึ่งอยู่นอกเหนือขอบเขตของ `go tool cover` เอง (ต้องพึ่งเครื่องมือเสริม เช่น Codecov หรือ Coveralls)

---

## 9. รวม Coverage ข้ามหลาย Package ด้วย `-coverpkg=./...`

โดย default เมื่อรัน `go test ./...` ทุก package จะรายงาน coverage **ของตัวเองเท่านั้น** — ปัญหาคือถ้า package `A` มี test ที่เรียกใช้ฟังก์ชันจาก package `B` (ผ่านการ import ปกติ) แต่ package `B` ไม่มี test file ของตัวเองเลย ค่า coverage ของ `B` จะไม่ถูกนับรวมอะไรเลยแม้ว่าโค้ดใน `B` จะถูกใช้งานจริงจาก test ของ `A` ก็ตาม (เพราะ `go test ./B/...` ไม่มี test ให้รัน)

Flag **`-coverpkg`** แก้ปัญหานี้โดยบอกให้ `go test` **นับ coverage ของ package ที่ระบุ ไม่ว่า test จะมาจาก package ไหนก็ตาม**:

```bash
go test -coverprofile=coverage_all.out -coverpkg=./... ./...
```

```
ok  	part082demo/mathx	0.002s	coverage: 30.8% of statements in ./...
ok  	part082demo/membership	0.002s	coverage: 61.5% of statements in ./...
```

สังเกตว่าตัวเลขเปลี่ยนไปจากหัวข้อ 2 (`mathx` จาก 80.0% กลายเป็น 30.8%) — เพราะตอนนี้ตัวส่วน (จำนวน statement ทั้งหมด) กลายเป็น **statement ทั้งหมดในทุก package รวมกัน** (`mathx` + `membership`) ไม่ใช่แค่ของ package ตัวเองอีกต่อไป ค่าที่รายงานออกมาต่อบรรทัดจึงสื่อถึง **"สัดส่วนของทั้งโปรเจกต์ที่ package นี้ทดสอบให้"** มากกว่า "สัดส่วนของตัวเองที่ทดสอบตัวเอง"

ดูภาพรวมที่ถูกต้องได้ชัดกว่าด้วย `go tool cover -func` กับไฟล์ที่รวมแล้ว:

```bash
go tool cover -func=coverage_all.out
```

```
part082demo/mathx/mathx.go:4:			Add		100.0%
part082demo/mathx/mathx.go:9:			Sub		0.0%
part082demo/mathx/mathx.go:14:			Abs		100.0%
part082demo/membership/membership.go:6:		IsValidAge	100.0%
part082demo/membership/membership.go:14:	Grade		100.0%
total:						(statements)	92.3%
```

บรรทัดสุดท้าย `total: (statements) 92.3%` คือ **ตัวเลข coverage รวมของทั้งโปรเจกต์ในไฟล์เดียว** — นี่คือค่าที่สคริปต์ `check_coverage.sh` ในหัวข้อ 8 ดึงไปใช้เปรียบเทียบกับ threshold และเป็นตัวเลขที่ทีมส่วนใหญ่ใช้รายงานเป็น "coverage ของโปรเจกต์" ใน README หรือ badge บน CI dashboard

> **ข้อแนะนำ**: เวลารายงาน coverage ของทั้งโปรเจกต์ให้ทีมหรือผู้บริหารดู ควรใช้ `-coverpkg=./...` เสมอ ไม่ใช่แค่รัน `go test -cover ./...` เฉยๆ เพราะแบบหลังจะรายงานเป็นตัวเลขแยกทีละ package ซึ่งไม่มี "ตัวเลขรวมเดียว" ให้ดูโดยตรง ต้องรวมด้วยมือเอง

---

## 10. ข้อควรระวังและ Best Practice

สรุปหลักปฏิบัติที่ควรยึดถือเมื่อทำงานกับ test coverage ในโปรเจกต์จริง:

1. **อย่าตั้งเป้า 100% coverage เป็นกฎตายตัว** — ตามที่พิสูจน์แล้วในหัวข้อ 6 ตัวเลข 100% ไม่ได้การันตีความถูกต้อง และการไล่ให้ถึง 100% เป๊ะมักบังคับให้ต้องเขียน test ทดสอบโค้ดที่ไม่มีความหมาย (เช่น `getter`/`setter` เปล่าๆ หรือ error path ที่แทบเป็นไปไม่ได้ในทางปฏิบัติ เช่น `os.Getwd()` คืน error) เสียเวลาโดยไม่ได้เพิ่มความมั่นใจในคุณภาพจริง ทีมส่วนใหญ่ตั้งเกณฑ์ที่สมเหตุสมผลกว่า เช่น 70-85% แล้วโฟกัสที่ **ความสำคัญทางธุรกิจของโค้ดส่วนนั้น** มากกว่าตัวเลขเปล่าๆ
2. **โค้ดที่สร้างขึ้นอัตโนมัติ (generated code)** เช่นไฟล์ที่ได้จาก `go generate`, protobuf compiler, หรือ mock generator (ทบทวนจาก **Part 035**) มักไม่จำเป็นต้องนับรวมใน coverage เลย เพราะไม่ได้เขียนโดยมนุษย์และมักถูกทดสอบโดยอ้อมผ่านโค้ดที่เรียกใช้มันอยู่แล้ว วิธีที่ใช้กันทั่วไปคือตั้งชื่อไฟล์ให้จบด้วย `_gen.go` หรือ `.pb.go` แล้วกรองไฟล์เหล่านี้ออกก่อนคำนวณตัวเลขรวม (เช่น กรองด้วย `grep -v` ก่อนส่งเข้า `go tool cover -func`)
3. **เกณฑ์ coverage ควรต่างกันตามชั้นของสถาปัตยกรรม** — โค้ด business logic ที่มีกฎซับซ้อน (เช่น การคำนวณราคา, การตรวจสอบสิทธิ์) ควรตั้งเกณฑ์สูงกว่าโค้ดที่เป็นแค่ "ทางผ่าน" ข้อมูล (เช่น handler ที่แค่ decode JSON แล้วส่งต่อให้ service อีกที) เพราะบั๊กในชั้น business logic มักส่งผลกระทบร้ายแรงกว่ามาก
4. **coverage รวมทั้งโปรเจกต์ (`total:` จาก `-coverpkg=./...`) สามารถ "หลอกตา" ได้** ถ้าโค้ดเก่าที่มี coverage สูงอยู่แล้วมีปริมาณมากกว่าโค้ดใหม่ที่เพิ่งเขียนมาก การเพิ่มโค้ดใหม่ 200 บรรทัดที่ coverage 0% อาจทำให้ตัวเลขรวมลดลงแค่ 1-2% เท่านั้น ไม่ทันสังเกต — ทีมที่จริงจังเรื่องนี้จึงมักเสริมด้วยเครื่องมือ **diff coverage** (ดูหัวข้อ 8) ที่โฟกัสเฉพาะโค้ดที่เปลี่ยนแปลงใน pull request แต่ละครั้งแทน
5. **รัน coverage คู่กับ `-race` เสมอเมื่อโค้ดมีการใช้ goroutine** (ทบทวนจาก **Part 044**) และอย่าลืมว่า `-race` จะบังคับใช้ `-covermode=atomic` ให้อัตโนมัติ (หัวข้อ 7) — การรัน `-cover` เฉยๆ โดยไม่มี `-race` กับโค้ด concurrent อาจพลาดบั๊กที่ race detector จับได้
6. **ใช้ coverage เป็นเครื่องมือ "สนทนา" ในการทำ code review ไม่ใช่ "คำตัดสิน" เพียงอย่างเดียว** — เมื่อเห็นว่า pull request ลด coverage ลง คำถามที่ควรถามคือ "โค้ดส่วนที่ไม่มี test ครอบคลุมคือส่วนไหน และความเสี่ยงคุ้มที่จะปล่อยผ่านหรือไม่" มากกว่าการ block ทุกครั้งที่ตัวเลขลดลงแม้แต่ 0.1%

## สรุปสิ่งที่ได้เรียนในบทนี้

- **Test coverage วัดว่า statement ถูกรันหรือไม่** ไม่ได้วัดว่าผลลัพธ์ถูกตรวจสอบครบถ้วนแค่ไหน — เป็นเครื่องมือหา "จุดบอด" ไม่ใช่ตัวชี้วัดคุณภาพ test โดยตรง
- **`go test -cover ./...`** ให้ตัวเลขสรุปเร็วๆ ต่อ package, **`-coverprofile=out.file`** เก็บรายละเอียดระดับบรรทัดไว้วิเคราะห์ต่อด้วย `go tool cover`
- **`go tool cover -func=coverage.out`** แสดง coverage แยกรายฟังก์ชัน มีประโยชน์มากในการหาฟังก์ชันที่ไม่เคยถูกทดสอบเลย (`0.0%`)
- **`go tool cover -html=coverage.out -o coverage.html`** สร้างรายงาน HTML พร้อมไฮไลต์สีเขียว/แดงบนโค้ดต้นฉบับจริง — ใช้ `-o` เมื่อไม่มีเบราว์เซอร์ให้เปิดอัตโนมัติ
- **100% coverage ไม่ได้การันตีความถูกต้อง**: พิสูจน์แล้วว่าฟังก์ชัน `IsValidAge` ได้ 100% coverage เต็มแต่ยังมีบั๊ก (ลืมเช็ค upper bound) เพราะ coverage มองเห็นแค่ branch ที่มีอยู่ในโค้ด ไม่เห็น branch ที่ควรมีแต่ผู้เขียนลืมใส่
- **`-covermode`**: `set` (default), `count` (นับจำนวนครั้ง), `atomic` (thread-safe, บังคับใช้อัตโนมัติเมื่อรันคู่กับ `-race`)
- **Coverage threshold gate** เขียนเป็นสคริปต์สั้นๆ ดึงตัวเลขจาก `go tool cover -func` บรรทัดสุดท้ายมาเทียบกับเกณฑ์ที่กำหนด แล้ว `exit 1` เมื่อไม่ผ่าน เพื่อใช้เป็น step หนึ่งใน CI (Part 098)
- **`-coverpkg=./...`** รวม coverage ข้ามหลาย package เข้าเป็นตัวเลขเดียวของทั้งโปรเจกต์ แก้ปัญหาที่ package ไม่มี test ของตัวเองแต่ถูกใช้งานจาก test ของ package อื่น

## แบบฝึกหัดท้ายบท

1. เขียนฟังก์ชัน `Sub` ทดสอบเพิ่มใน package `mathx` แล้วรัน `go test -cover ./mathx/...` ยืนยันว่า coverage ขึ้นเป็น 100% จริง
2. คิดฟังก์ชันของตัวเองอย่างน้อย 1 ฟังก์ชันที่มี "บั๊กจากการลืมเช็ค edge case" คล้ายกับ `IsValidAge` ในหัวข้อ 6 (เช่น ฟังก์ชันตรวจสอบรหัสผ่านที่ลืมเช็คความยาวสูงสุด หรือฟังก์ชันคำนวณส่วนลดที่ลืม cap ที่ 100%) เขียน test ที่ทำให้ coverage ถึง 100% แต่ยังพลาดบั๊กอยู่ แล้วเพิ่ม test case ที่จับบั๊กนั้นได้จริง
3. รันคำสั่ง `go tool cover -html=coverage.out -o coverage.html` กับ package จากแบบฝึกหัดข้อ 1 แล้วเปิดไฟล์ HTML ดูด้วยเบราว์เซอร์ (หรือดูโครงสร้าง HTML ด้วยตาเปล่าถ้าไม่มีเบราว์เซอร์) สังเกตว่าบรรทัดที่ยังไม่ถูกทดสอบถูกไฮไลต์เป็นสีอะไร
4. เขียนและรัน `check_coverage.sh` (หรือเวอร์ชันปรับปรุงของตัวเอง) กับโปรเจกต์จริงที่เคยเขียนไว้ในแบบฝึกหัดของ part ก่อนหน้า ลองปรับ threshold หลายค่าดูว่า script ทำงานถูกต้องหรือไม่
5. ทดลองรัน `go test -covermode=count -coverprofile=count.out ./...` แล้วเปิดดูเนื้อหาไฟล์ `count.out` ด้วย `cat` เปรียบเทียบกับไฟล์ที่ได้จาก `-covermode=set` (default) สังเกตความแตกต่างของตัวเลขในแต่ละบรรทัด
6. ค้นคว้าเพิ่มเติมเกี่ยวกับเครื่องมือ diff coverage ของบุคคลที่สาม เช่น Codecov หรือ Coveralls ที่มักใช้คู่กับ GitHub Actions อธิบายด้วยคำพูดตัวเองว่าทำไม "coverage เฉพาะส่วนที่เปลี่ยนแปลงใน PR" ถึงมีประโยชน์มากกว่า "coverage รวมทั้งโปรเจกต์" ในบางสถานการณ์

---

**ต่อไป**: [Part 083 — Profiling ด้วย `pprof`](./083-pprof-profiling.md)
