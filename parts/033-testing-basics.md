# Part 033: Testing พื้นฐานด้วย `testing`

> ภาคที่ 2: ระดับกลาง (Intermediate) — ตอนที่ 18 จาก 20 (Part 16–35)

## สารบัญของบทนี้

1. ทำไมต้องเขียน Test และปรัชญาการเทสของ Go
2. ไฟล์ `_test.go` และกฎการตั้งชื่อ `TestXxx(t *testing.T)`
3. คำสั่ง `go test` และ Flag สำคัญ: `-v`, `-run`, `-cover`
4. `t.Error`/`t.Errorf` vs `t.Fatal`/`t.Fatalf`
5. Test Helper Functions และ `t.Helper()`
6. Subtests ด้วย `t.Run`
7. `TestMain`: Setup/Teardown ระดับ Package
8. การทดสอบ Error: เช็คทั้งค่าและ Error ให้ครบ
9. ตัวอย่างเต็ม: ทดสอบ Package `mathutil`
10. สรุปสิ่งที่ได้เรียนในบทนี้
11. แบบฝึกหัดท้ายบท

---

## 1. ทำไมต้องเขียน Test และปรัชญาการเทสของ Go

ตลอด 32 part ที่ผ่านมา เราเขียนโปรแกรมแล้วตรวจสอบผลลัพธ์ด้วยตาเปล่าผ่าน `fmt.Println` และรันด้วย `go run` เป็นหลัก วิธีนี้ใช้ได้ดีตอนเรียนรู้ แต่ในโปรเจกต์จริงที่มีโค้ดหลายพันบรรทัดและมีการแก้ไขต่อเนื่องไปเรื่อยๆ **การตรวจสอบด้วยตาเปล่าทุกครั้งที่แก้โค้ดเป็นเรื่องที่ทำไม่ไหวและเสี่ยงต่อการพลาดบั๊กมาก**

**Automated testing (การทดสอบอัตโนมัติ)** คือการเขียนโค้ดอีกชุดหนึ่งที่ทำหน้าที่ตรวจสอบว่าโค้ดหลักทำงานถูกต้องตามที่คาดหวัง โดยรันซ้ำได้ไม่จำกัดจำนวนครั้งด้วยคำสั่งเดียว ประโยชน์หลักๆ คือ:

- **จับบั๊กได้เร็วตั้งแต่ตอนพัฒนา** ก่อนที่จะไปถึงมือผู้ใช้จริง
- **มั่นใจได้เมื่อ refactor โค้ด** — แก้โค้ดเสร็จแล้วรัน test ทั้งหมด ถ้าผ่านหมดแปลว่ายังทำงานถูกต้องเหมือนเดิม
- **เป็นเอกสารประกอบโค้ดในตัว** — test ที่ดีบอกได้ว่าฟังก์ชันนั้นควรมีพฤติกรรมอย่างไรในสถานการณ์ต่างๆ ชัดเจนกว่า comment เสียอีก
- **ทำงานเป็นทีมได้ปลอดภัยขึ้น** — เพื่อนร่วมทีมแก้โค้ดที่เราเขียน แล้วรัน test ของเราเพื่อเช็คว่าไม่ได้ทำอะไรพัง

จุดเด่นที่สำคัญมากของ Go คือ **testing framework เป็นส่วนหนึ่งของ standard library โดยตรง** ผ่าน package `testing` และคำสั่ง `go test` ที่มาพร้อมกับ toolchain ตั้งแต่ติดตั้ง Go (ทบทวนจาก **Part 001** ที่กล่าวถึง `go test` เป็นหนึ่งใน tool หลักที่ต้องรู้จัก) ต่างจากหลายภาษาที่ต้องติดตั้ง testing framework แยกต่างหาก (เช่น JUnit ของ Java, pytest ของ Python) — นี่คือตัวอย่างที่ชัดเจนอีกครั้งของปรัชญา "standard library แข็งแกร่ง" ที่ Go ยึดถือมาตั้งแต่ **Part 001**

บทนี้และอีก 2 บทถัดไป (**Part 034: Table-Driven Tests และ Benchmarks**, **Part 035: Mocking และ Test Doubles**) จะปูพื้นฐานการเขียน test ให้แน่น ก่อนไปเจาะลึกเรื่อง testing ขั้นสูงอีกครั้งใน **ภาคที่ 7 (Part 079-087)** ที่ครอบคลุมทั้ง integration test, `httptest`, test coverage แบบละเอียด และ profiling

---

## 2. ไฟล์ `_test.go` และกฎการตั้งชื่อ `TestXxx(t *testing.T)`

Go มีกฎที่เข้มงวดและเรียบง่ายมากสำหรับการเขียน test ต่างจากภาษาอื่นที่มักต้องมี annotation หรือ config file แยกต่างหาก:

1. **ไฟล์ test ต้องลงท้ายด้วย `_test.go` เสมอ** เช่น `mathutil.go` คู่กับ `mathutil_test.go` — Go toolchain รู้จักไฟล์เหล่านี้เป็นพิเศษ: `go build` จะ**ไม่รวม**ไฟล์ `_test.go` เข้าไปใน binary ที่ deploy จริง แต่ `go test` จะ**รวม**ไฟล์เหล่านี้เข้ามาคอมไพล์และรันด้วยเสมอ
2. **ไฟล์ test มักอยู่ใน package เดียวกันกับโค้ดที่ทดสอบ** (เรียกว่า **internal test** — สามารถเข้าถึง unexported identifier ได้ตรงๆ) หรือจะแยกเป็น package ชื่อ `xxx_test` ก็ได้ (เรียกว่า **external test** — เข้าถึงได้เฉพาะ exported identifier เท่านั้น เหมาะกับการทดสอบว่า public API ของ package ใช้งานได้ตามที่ตั้งใจจากมุมมองของผู้ใช้จริง)
3. **ฟังก์ชัน test ต้องตั้งชื่อขึ้นต้นด้วย `Test` ตามด้วยตัวอักษรตัวใหญ่เสมอ** เช่น `TestAdd`, `TestDivide` (ไม่ใช่ `Testadd` เพราะตัวถัดจาก `Test` ต้องเป็นตัวพิมพ์ใหญ่ ไม่เช่นนั้น Go จะไม่รู้จักว่าเป็น test function) — ทบทวนกฎ capitalization จาก **Part 001** อีกครั้ง
4. **Signature ต้องเป็น `func TestXxx(t *testing.T)` เป๊ะๆ เท่านั้น** รับ parameter ตัวเดียวคือ `*testing.T` และไม่คืนค่าอะไรเลย

ตัวอย่างที่ง่ายที่สุด — สมมติมีฟังก์ชัน `Add` ใน `mathutil.go`:

```go
// mathutil.go
package mathutil

func Add(a, b int) int {
	return a + b
}
```

คู่กับไฟล์ทดสอบ `mathutil_test.go`:

```go
// mathutil_test.go
package mathutil

import "testing"

func TestAdd(t *testing.T) {
	got := Add(2, 3)
	want := 5
	if got != want {
		t.Errorf("Add(2, 3) = %d, want %d", got, want)
	}
}
```

รันด้วยคำสั่ง:

```bash
go test ./...
```

ผลลัพธ์ (ถ้าผ่านทั้งหมด):

```
ok  	mathutil	0.003s
```

สังเกตว่า `testing.T` คือ struct ที่มี method ต่างๆ (`t.Errorf`, `t.Fatal`, ฯลฯ) ไว้ให้เรารายงานผลการทดสอบ — Go **ไม่มีการ assert แบบมี built-in keyword** เหมือนบางภาษา (เช่น `assert` ของ Python) แต่ใช้ `if` ธรรมดาตรวจสอบเงื่อนไข แล้วเรียก method ของ `t` เพื่อรายงานความล้มเหลวเมื่อเงื่อนไขไม่ตรงตามคาด — นี่คือความเรียบง่ายแบบ Go ที่ไม่ต้องมี syntax พิเศษเพิ่ม ใช้แค่ `if` และฟังก์ชันปกติทุกอย่าง

---

## 3. คำสั่ง `go test` และ Flag สำคัญ: `-v`, `-run`, `-cover`

คำสั่ง `go test` เป็น subcommand ของ `go` toolchain (แนะนำครั้งแรกใน **Part 001**) ที่ compile และรัน test ทั้งหมดใน package ที่ระบุ มี flag ที่ใช้บ่อยที่สุด 3 ตัวที่ต้องรู้จัก:

| Flag | ความหมาย |
|---|---|
| `go test ./...` | รัน test ทุก package ในโปรเจกต์ (`./...` หมายถึง "ทุก package ตั้งแต่ directory ปัจจุบันลงไป") |
| `go test -v` | verbose mode — พิมพ์ชื่อ test ทุกตัวที่รัน พร้อมสถานะ PASS/FAIL และเวลาที่ใช้ ไม่ใช่แค่สรุปรวม |
| `go test -run TestAdd` | รันเฉพาะ test ที่ชื่อ**match กับ regular expression** ที่ระบุ (ในที่นี้คือ test ที่ชื่อมีคำว่า `TestAdd`) มีประโยชน์มากตอน debug test ตัวใดตัวหนึ่งโดยไม่ต้องรอ test อื่นทั้งหมด |
| `go test -cover` | รายงาน**เปอร์เซ็นต์ของโค้ดที่ถูกทดสอบ** (test coverage) — จะเจาะลึกเรื่องนี้เต็มรูปแบบใน **Part 082 (Test Coverage)** |

ตัวอย่างการรันด้วย `-v`:

```bash
go test -v ./...
```

```
=== RUN   TestAdd
--- PASS: TestAdd (0.00s)
=== RUN   TestDivide
--- PASS: TestDivide (0.00s)
PASS
ok  	mathutil	0.003s
```

ตัวอย่างการรันเฉพาะบาง test ด้วย `-run`:

```bash
go test -run TestDivide -v ./...
```

```
=== RUN   TestDivide
--- PASS: TestDivide (0.00s)
=== RUN   TestDivide_ByZero
--- PASS: TestDivide_ByZero (0.00s)
PASS
ok  	mathutil	0.003s
```

สังเกตว่า `-run TestDivide` match ทั้ง `TestDivide` และ `TestDivide_ByZero` เพราะมันเป็น regular expression ที่เช็คว่าชื่อ**มีคำนี้อยู่**ไม่ใช่การ match แบบเป๊ะทั้งชื่อ — ถ้าต้องการ match แบบเป๊ะให้ใช้ `^` และ `$` ครอบ เช่น `-run '^TestDivide$'`

ตัวอย่างการรันพร้อมดู coverage:

```bash
go test -cover ./...
```

```
ok  	mathutil	0.004s	coverage: 100.0% of statements
```

flag ทั้งสามตัวนี้มักถูกใช้ร่วมกันในการพัฒนาประจำวัน เช่น `go test -v -run TestDivide -cover ./...` เพื่อโฟกัสที่ test ตัวเดียวพร้อมดูรายละเอียดและ coverage ไปในตัว

---

## 4. `t.Error`/`t.Errorf` vs `t.Fatal`/`t.Fatalf`

`*testing.T` มี method สำหรับรายงานความล้มเหลว 4 ตัวหลัก แบ่งเป็น 2 กลุ่มตามพฤติกรรม:

| Method | พฤติกรรมเมื่อถูกเรียก |
|---|---|
| `t.Error(args...)` / `t.Errorf(format, args...)` | บันทึกว่า test นี้ **fail** แล้ว**ทำงานต่อไปจนจบฟังก์ชัน** (ไม่หยุดทันที) |
| `t.Fatal(args...)` / `t.Fatalf(format, args...)` | บันทึกว่า test นี้ **fail** แล้ว**หยุดฟังก์ชัน test ทันที** (เทียบเท่าเรียก `runtime.Goexit()` ภายใน) |

ความต่างระหว่าง `Error` กับ `Errorf`/`Fatal` กับ `Fatalf` เหมือนกับ `fmt.Print` กับ `fmt.Printf` ที่เรียนใน **Part 021**: ตัวที่ลงท้ายด้วย `f` รับ format string พร้อม verb (`%d`, `%s`, `%v`, ...) ส่วนตัวที่ไม่มี `f` รับ argument แล้วต่อกันแบบ `fmt.Sprintln` ธรรมดา

**คำถามสำคัญคือ: เมื่อไหร่ควรใช้ตัวไหน?** หลักการตัดสินใจคือ:

> **ใช้ `t.Fatal`/`t.Fatalf` เมื่อการทำงานต่อไปของ test ไม่มีความหมายอีกแล้วถ้าเงื่อนไขนี้ไม่ผ่าน (เช่น เกิด error ที่ไม่ควรเกิด แล้วโค้ดถัดไปต้องใช้ผลลัพธ์ที่ควรได้จากตรงนั้น) ใช้ `t.Error`/`t.Errorf` เมื่อต้องการรายงานปัญหาแต่ยังอยากให้ test ตรวจสอบเงื่อนไขอื่นๆ ที่เหลือต่อไปด้วย เพื่อเห็นภาพรวมของปัญหาทั้งหมดในการรันครั้งเดียว**

มาดูตัวอย่างที่แสดงความต่างชัดเจน (ตัวอย่างนี้จงใจเขียนให้ผิดพลาดเพื่อสาธิตพฤติกรรม ไม่ใช่โค้ดที่ควรเขียนจริง):

```go
package demo

import "testing"

func Add(a, b int) int {
	return a + b
}

// TestErrorVsFatal จงใจเขียนให้ผิดพลาดหลายจุด เพื่อสาธิตว่า t.Error ทำให้ test ทำงานต่อ
// ในขณะที่ t.Fatal หยุดฟังก์ชันทดสอบทันที (รันจริงแล้วจะเห็น FAIL -- นี่คือของจงใจ ไม่ใช่บั๊ก)
func TestErrorVsFatal(t *testing.T) {
	t.Log("บรรทัดที่ 1: กำลังเช็คค่าแรก")
	if got := Add(2, 2); got != 5 { // ผิดโดยตั้งใจ (2+2=4 ไม่ใช่ 5)
		t.Errorf("Add(2, 2) = %d, want %d", got, 5)
	}

	t.Log("บรรทัดที่ 2: t.Error ไม่หยุดการทำงาน จึงมาถึงบรรทัดนี้ได้")

	if got := Add(3, 3); got != 7 { // ผิดโดยตั้งใจอีกครั้ง (3+3=6 ไม่ใช่ 7)
		t.Fatalf("Add(3, 3) = %d, want %d", got, 7)
	}

	t.Log("บรรทัดนี้จะไม่ถูกพิมพ์เลย เพราะ t.Fatal ด้านบนหยุดฟังก์ชันไปแล้ว")
}
```

ผลลัพธ์จากการรัน `go test -v`:

```
=== RUN   TestErrorVsFatal
    demo_test.go:8: บรรทัดที่ 1: กำลังเช็คค่าแรก
    demo_test.go:10: Add(2, 2) = 4, want 5
    demo_test.go:13: บรรทัดที่ 2: t.Error ไม่หยุดการทำงาน จึงมาถึงบรรทัดนี้ได้
    demo_test.go:16: Add(3, 3) = 6, want 7
--- FAIL: TestErrorVsFatal (0.00s)
FAIL
```

สังเกตว่า:

- `t.Log` (คล้าย `fmt.Println` แต่พิมพ์เฉพาะตอนรันด้วย `-v` หรือตอน test fail) ทั้งสามบรรทัดแรกถูกพิมพ์ออกมา แสดงว่า test **ทำงานต่อได้** หลังจาก `t.Errorf` บรรทัดแรก
- `t.Log` บรรทัดสุดท้ายไม่ถูกพิมพ์เลย เพราะ `t.Fatalf` ก่อนหน้าทำให้ **ฟังก์ชัน `TestErrorVsFatal` หยุดทำงานทันที** ไม่มีทางไปถึงบรรทัดนั้นได้
- ผลลัพธ์สุดท้ายคือ `FAIL` เพราะมีการเรียก `t.Error*`/`t.Fatal*` เกิดขึ้นอย่างน้อยหนึ่งครั้ง

รูปแบบที่พบบ่อยในโค้ดจริงคือใช้ `t.Fatal` ทันทีหลังเช็ค `error` ที่ไม่ควรเกิด (เพราะถ้า error เกิดขึ้นจริง ค่าที่ได้กลับมามักจะเป็น zero value ที่ไม่มีความหมายให้เช็คต่อ) แล้วใช้ `t.Error`/`t.Errorf` สำหรับเช็คค่าจริงหลังจากนั้น ดังตัวอย่างในหัวข้อ 9

---

## 5. Test Helper Functions และ `t.Helper()`

เมื่อมี logic การตรวจสอบที่ต้องใช้ซ้ำในหลาย test (เช่น การเช็คว่าค่าสองตัวเท่ากัน) การเขียนฟังก์ชันช่วย (helper function) ขึ้นมาแยกต่างหากช่วยลดโค้ดซ้ำได้มาก:

```go
func assertEqual(t *testing.T, got, want int) {
	t.Helper()
	if got != want {
		t.Errorf("got %d, want %d", got, want)
	}
}
```

จุดสำคัญคือการเรียก **`t.Helper()`** เป็นบรรทัดแรกของฟังก์ชัน helper เสมอ — เรียกแล้วมันบอก package `testing` ว่า "ฟังก์ชันนี้เป็นแค่ตัวช่วย ไม่ใช่ต้นตอของปัญหา" ผลลัพธ์คือ **เวลา test fail ข้อความ error จะรายงานเลขบรรทัดของโค้ดที่เรียก `assertEqual` (ฝั่งผู้ใช้) แทนที่จะรายงานเลขบรรทัดภายใน `assertEqual` เอง** ทำให้ debug ได้เร็วขึ้นมาก เพราะรู้ทันทีว่า test ตัวไหนบรรทัดไหนที่มีปัญหาจริงๆ โดยไม่ต้องไล่ stack ย้อนกลับเอง

```go
func TestAddUsingHelper(t *testing.T) {
	assertEqual(t, Add(10, 20), 30)
	assertEqual(t, Add(-5, 5), 0)
}
```

ถ้าลองแก้ `assertEqual(t, Add(10, 20), 30)` เป็นค่าผิดๆ เช่น `assertEqual(t, Add(10, 20), 999)` แล้วรัน จะเห็นว่าข้อความ error ชี้ไปที่บรรทัดของ `TestAddUsingHelper` ที่เรียก `assertEqual` ไม่ใช่บรรทัด `t.Errorf` ที่อยู่ข้างในฟังก์ชัน `assertEqual` เอง — นี่คือประโยชน์ที่แท้จริงของ `t.Helper()`

---

## 6. Subtests ด้วย `t.Run`

**Subtest** คือ test ย่อยที่ซ้อนอยู่ในฟังก์ชัน test หลักตัวหนึ่ง สร้างผ่าน `t.Run(name string, f func(t *testing.T))` ประโยชน์หลักคือช่วยจัดกลุ่มกรณีทดสอบที่เกี่ยวข้องกัน และทำให้เห็นผลลัพธ์แยกเป็นรายกรณีอย่างชัดเจนเวลารันด้วย `-v`:

```go
func TestIsEven(t *testing.T) {
	t.Run("เลขคู่", func(t *testing.T) {
		if !IsEven(4) {
			t.Error("4 ควรเป็นเลขคู่")
		}
	})
	t.Run("เลขคี่", func(t *testing.T) {
		if IsEven(3) {
			t.Error("3 ไม่ควรเป็นเลขคู่")
		}
	})
}
```

ผลลัพธ์เมื่อรันด้วย `-v`:

```
=== RUN   TestIsEven
=== RUN   TestIsEven/เลขคู่
=== RUN   TestIsEven/เลขคี่
--- PASS: TestIsEven (0.00s)
    --- PASS: TestIsEven/เลขคู่ (0.00s)
    --- PASS: TestIsEven/เลขคี่ (0.00s)
PASS
```

ข้อดีที่สำคัญของ subtest นอกจากความอ่านง่ายคือ:

- **รันแยกกรณีเดียวได้ผ่าน `-run`** ด้วยรูปแบบ `go test -run 'TestIsEven/เลขคู่'` (ใช้ `/` คั่นระหว่างชื่อ test หลักกับชื่อ subtest)
- **subtest หนึ่งตัว fail ไม่กระทบ subtest อื่น** — ทุก subtest ทำงานเป็นอิสระจากกัน ถ้า `เลขคู่` fail แต่ `เลขคี่` ยังผ่านอยู่ ผลลัพธ์ก็ยังบอกได้ชัดว่าตัวไหน pass ตัวไหน fail
- **เป็นรากฐานของ table-driven test** ซึ่งเป็น pattern ที่ใช้กันแพร่หลายที่สุดใน Go — จะเจาะลึกเต็มรูปแบบใน **Part 034** บทถัดไปทันที ลองสังเกตตัวอย่างสั้นๆ ก่อนว่า `t.Run` มักถูกเรียกในลูปที่วนผ่าน slice ของ struct ที่เก็บกรณีทดสอบไว้ ดังตัวอย่างในหัวข้อ 9

---

## 7. `TestMain`: Setup/Teardown ระดับ Package

บางครั้งชุดทดสอบทั้ง package ต้องการการเตรียมงานก่อนเริ่ม (setup) และการเก็บกวาดหลังจบทั้งหมด (teardown) เช่น เปิดการเชื่อมต่อฐานข้อมูลทดสอบ, สร้างไฟล์ชั่วคราว, หรือแค่พิมพ์ log บอกจุดเริ่ม/จบ — ทำได้ผ่านฟังก์ชันพิเศษชื่อ **`TestMain(m *testing.M)`**

ถ้า package ไหนมีฟังก์ชันนี้ **`go test` จะเรียก `TestMain` แทนที่จะรัน `TestXxx` ทุกตัวโดยตรง** และเป็นหน้าที่ของเราที่ต้องเรียก `m.Run()` เองภายใน `TestMain` เพื่อสั่งให้ test ทั้งหมดทำงานจริง:

```go
func TestMain(m *testing.M) {
	fmt.Println(">>> เริ่มต้นชุดทดสอบ mathutil (setup)")
	code := m.Run() // รัน test ทั้งหมดในไฟล์นี้ คืนค่า exit code
	fmt.Println(">>> จบชุดทดสอบ mathutil (teardown)")
	os.Exit(code)
}
```

ผลลัพธ์เมื่อรัน:

```
>>> เริ่มต้นชุดทดสอบ mathutil (setup)
=== RUN   TestAdd
--- PASS: TestAdd (0.00s)
...
PASS
>>> จบชุดทดสอบ mathutil (teardown)
ok  	mathutil	0.003s
```

จุดสำคัญที่ต้องจำ:

- **ต้องเรียก `os.Exit(code)` เสมอ** โดยใช้ค่าที่ได้จาก `m.Run()` — ถ้าลืม `os.Exit` โปรแกรมจะจบแบบปกติและรายงานผลผิดพลาดไม่ถูกต้อง (`go test` จะไม่รู้ว่า test fail หรือ pass)
- **`TestMain` มีได้แค่หนึ่งเดียวต่อ package เท่านั้น** ถ้าต้องการ setup/teardown เฉพาะบาง test ให้ใช้วิธีอื่น เช่น helper function ที่เรียกตอนต้น/ท้ายของแต่ละ `TestXxx` หรือ `t.Cleanup()` (ฟังก์ชันที่ลงทะเบียนให้ทำงานอัตโนมัติตอน test นั้นจบ ไม่ว่าจะ pass หรือ fail)
- ใช้ `TestMain` เมื่อ setup/teardown เป็นงานที่ **ทำครั้งเดียวสำหรับทั้ง package** เท่านั้น เช่น เปิด/ปิดการเชื่อมต่อ resource ที่ใช้ร่วมกันในหลาย test — ถ้า setup เฉพาะเจาะจงกับ test ตัวเดียว ให้เขียนไว้ในฟังก์ชัน test นั้นตรงๆ จะชัดเจนกว่า

---

## 8. การทดสอบ Error: เช็คทั้งค่าและ Error ให้ครบ

ฟังก์ชันจำนวนมากใน Go คืนค่าเป็น `(value, error)` ตามหลักการ error handling ที่เรียนใน **Part 015-016** การเขียน test ให้ครอบคลุมฟังก์ชันแบบนี้ต้องตรวจสอบทั้งสองส่วนอย่างระมัดระวัง:

```go
var ErrDivideByZero = errors.New("mathutil: divide by zero")

func Divide(a, b float64) (float64, error) {
	if b == 0 {
		return 0, ErrDivideByZero
	}
	return a / b, nil
}
```

**กรณีที่ 1: ทดสอบ path ที่ไม่ควร error** — ต้องเช็ค `err` ก่อนเสมอด้วย `t.Fatal` (เพราะถ้า error เกิดขึ้นทั้งที่ไม่ควร ค่า `result` ที่ได้กลับมามักไม่มีความหมายให้ตรวจสอบต่อ):

```go
func TestDivide(t *testing.T) {
	result, err := Divide(10, 2)
	if err != nil {
		t.Fatalf("ไม่ควร error แต่ได้: %v", err)
	}
	if result != 5 {
		t.Errorf("Divide(10, 2) = %v, want 5", result)
	}
}
```

**กรณีที่ 2: ทดสอบ path ที่ควร error** — ต้องเช็คว่า `err` ไม่ใช่ `nil` ก่อน แล้วค่อยตรวจสอบว่าเป็น error **ตัวที่ถูกต้องจริงๆ** ด้วย `errors.Is` (ทบทวนจาก **Part 016**) แทนการเทียบข้อความ error เป็น string ตรงๆ ซึ่งเปราะบางกว่ามาก (ถ้ามีคนแก้ข้อความ error ในอนาคต test ที่เทียบ string จะพังทันทีทั้งที่ logic ยังถูกต้องอยู่):

```go
func TestDivide_ByZero(t *testing.T) {
	_, err := Divide(10, 0)
	if err == nil {
		t.Fatal("ควร error แต่ไม่ error")
	}
	if !errors.Is(err, ErrDivideByZero) {
		t.Errorf("err = %v, want ErrDivideByZero", err)
	}
}
```

**หลักการสำคัญที่ต้องจำจากหัวข้อนี้**:

1. เช็ค `err == nil`/`err != nil` ก่อนเสมอด้วย `t.Fatal`/`t.Fatalf` เพื่อป้องกันไม่ให้โค้ดเช็คค่าต่อไปทำงานกับข้อมูลที่ไม่มีความหมาย
2. เมื่อคาดว่าจะได้ error ที่เจาะจง ให้เช็คด้วย `errors.Is`/`errors.As` แทนการเทียบข้อความ string ตรงๆ เพื่อให้ test ทนทานต่อการเปลี่ยนแปลงข้อความในอนาคต
3. ทดสอบทั้ง **happy path** (ทำงานสำเร็จ) และ **error path** (ควร error) เสมอ อย่าทดสอบแค่กรณีที่ทุกอย่างถูกต้องเพียงอย่างเดียว เพราะบั๊กส่วนใหญ่มักซ่อนอยู่ในเส้นทางที่จัดการ error ต่างหาก

---

## 9. ตัวอย่างเต็ม: ทดสอบ Package `mathutil`

มารวบรวมทุกอย่างที่เรียนมาในบทนี้เป็นตัวอย่างที่สมบูรณ์ สมมติมี package `mathutil` ที่มี 3 ฟังก์ชัน:

```go
// mathutil.go
package mathutil

import "errors"

var ErrDivideByZero = errors.New("mathutil: divide by zero")

func Add(a, b int) int {
	return a + b
}

func Divide(a, b float64) (float64, error) {
	if b == 0 {
		return 0, ErrDivideByZero
	}
	return a / b, nil
}

func IsEven(n int) bool {
	return n%2 == 0
}
```

และไฟล์ทดสอบที่ครอบคลุมทุกเทคนิคจากบทนี้:

```go
// mathutil_test.go
package mathutil

import (
	"errors"
	"fmt"
	"os"
	"testing"
)

func TestMain(m *testing.M) {
	fmt.Println(">>> เริ่มต้นชุดทดสอบ mathutil (setup)")
	code := m.Run()
	fmt.Println(">>> จบชุดทดสอบ mathutil (teardown)")
	os.Exit(code)
}

func TestAdd(t *testing.T) {
	got := Add(2, 3)
	want := 5
	if got != want {
		t.Errorf("Add(2, 3) = %d, want %d", got, want)
	}
}

func TestDivide(t *testing.T) {
	result, err := Divide(10, 2)
	if err != nil {
		t.Fatalf("ไม่ควร error แต่ได้: %v", err)
	}
	if result != 5 {
		t.Errorf("Divide(10, 2) = %v, want 5", result)
	}
}

func TestDivide_ByZero(t *testing.T) {
	_, err := Divide(10, 0)
	if err == nil {
		t.Fatal("ควร error แต่ไม่ error")
	}
	if !errors.Is(err, ErrDivideByZero) {
		t.Errorf("err = %v, want ErrDivideByZero", err)
	}
}

func assertEqual(t *testing.T, got, want int) {
	t.Helper()
	if got != want {
		t.Errorf("got %d, want %d", got, want)
	}
}

func TestIsEven(t *testing.T) {
	cases := []struct {
		name  string
		input int
		want  bool
	}{
		{"เลขคู่", 4, true},
		{"เลขคี่", 3, false},
		{"ศูนย์ถือเป็นเลขคู่", 0, true},
		{"เลขลบคู่", -4, true},
	}

	for _, c := range cases {
		t.Run(c.name, func(t *testing.T) {
			got := IsEven(c.input)
			if got != c.want {
				t.Errorf("IsEven(%d) = %v, want %v", c.input, got, c.want)
			}
		})
	}
}

func TestAddUsingHelper(t *testing.T) {
	assertEqual(t, Add(10, 20), 30)
	assertEqual(t, Add(-5, 5), 0)
}
```

รันทดสอบทั้งหมดพร้อมดู coverage:

```bash
go test -v -cover ./...
```

ผลลัพธ์:

```
>>> เริ่มต้นชุดทดสอบ mathutil (setup)
=== RUN   TestAdd
--- PASS: TestAdd (0.00s)
=== RUN   TestDivide
--- PASS: TestDivide (0.00s)
=== RUN   TestDivide_ByZero
--- PASS: TestDivide_ByZero (0.00s)
=== RUN   TestIsEven
=== RUN   TestIsEven/เลขคู่
=== RUN   TestIsEven/เลขคี่
=== RUN   TestIsEven/ศูนย์ถือเป็นเลขคู่
=== RUN   TestIsEven/เลขลบคู่
--- PASS: TestIsEven (0.00s)
    --- PASS: TestIsEven/เลขคู่ (0.00s)
    --- PASS: TestIsEven/เลขคี่ (0.00s)
    --- PASS: TestIsEven/ศูนย์ถือเป็นเลขคู่ (0.00s)
    --- PASS: TestIsEven/เลขลบคู่ (0.00s)
=== RUN   TestAddUsingHelper
--- PASS: TestAddUsingHelper (0.00s)
PASS
>>> จบชุดทดสอบ mathutil (teardown)
coverage: 100.0% of statements
ok  	mathutil	0.003s
```

ตัวอย่างนี้ใช้ครบทุกเทคนิคจากบทนี้: `TestMain` สำหรับ setup/teardown, `t.Fatal` เมื่อ error ไม่คาดคิดทำให้ทดสอบต่อไม่ได้, `t.Error` เมื่อยังตรวจสอบต่อได้, `errors.Is` สำหรับเช็ค error ที่เจาะจง, `t.Helper()` ในฟังก์ชันช่วย และ `t.Run` สำหรับจัดกลุ่มกรณีทดสอบ — coverage 100% แสดงว่าทุกบรรทัดของ `mathutil.go` ถูกทดสอบครบ (แต่ต้องระวังว่า **coverage สูงไม่ได้แปลว่า test ดีเสมอไป** เพราะวัดแค่ว่าบรรทัดโค้ดถูกรันหรือไม่ ไม่ได้วัดว่าตรวจสอบผลลัพธ์ถูกต้องครบถ้วนแค่ไหน — เรื่องนี้จะพูดถึงอย่างละเอียดใน **Part 082**)

---

## สรุปสิ่งที่ได้เรียนในบทนี้

- Go มี testing framework ในตัวผ่าน package `testing` และคำสั่ง `go test` โดยไม่ต้องติดตั้ง library เพิ่ม
- ไฟล์ test ต้องลงท้ายด้วย **`_test.go`** และฟังก์ชัน test ต้องมีชื่อ **`TestXxx(t *testing.T)`** (`Xxx` ขึ้นต้นด้วยตัวพิมพ์ใหญ่เสมอ)
- `go test ./...` รันทุก test, `-v` แสดงรายละเอียดทุก test, `-run <pattern>` รันเฉพาะ test ที่ชื่อ match, `-cover` รายงานเปอร์เซ็นต์โค้ดที่ถูกทดสอบ
- **`t.Error`/`t.Errorf`** รายงาน fail แล้วทำงานต่อ ส่วน **`t.Fatal`/`t.Fatalf`** รายงาน fail แล้วหยุดฟังก์ชันทันที — ใช้ `Fatal` เมื่อทำงานต่อไม่มีความหมาย (เช่น หลังเช็ค error ที่ไม่ควรเกิด)
- Test helper function ควรเรียก **`t.Helper()`** เป็นบรรทัดแรกเสมอ เพื่อให้ error message ชี้ไปที่บรรทัดของผู้เรียกแทนที่จะชี้เข้ามาในตัว helper เอง
- **`t.Run(name, func(t *testing.T){...})`** สร้าง subtest ที่จัดกลุ่มกรณีทดสอบที่เกี่ยวข้องกัน รันแยกได้ผ่าน `-run 'Test/subtest'` และเป็นรากฐานของ table-driven test ที่จะเรียนใน Part 034
- **`TestMain(m *testing.M)`** ใช้ทำ setup/teardown ระดับทั้ง package ต้องเรียก `m.Run()` และ `os.Exit(code)` เสมอ
- การทดสอบฟังก์ชันที่คืนค่า `(value, error)` ต้องเช็ค error ก่อนด้วย `t.Fatal` แล้วใช้ `errors.Is`/`errors.As` (Part 016) เช็ค error ที่เจาะจง แทนการเทียบข้อความ string ตรงๆ
- Test coverage สูงไม่ได้การันตีว่า test มีคุณภาพดี — วัดแค่ว่าบรรทัดถูกรันหรือไม่ ไม่ได้วัดความครบถ้วนของการตรวจสอบผลลัพธ์

## แบบฝึกหัดท้ายบท

1. เขียน package `stringutil` ที่มีฟังก์ชัน `Reverse(s string) string` (กลับด้านสตริง) แล้วเขียนไฟล์ `stringutil_test.go` ทดสอบด้วย `TestXxx` ธรรมดา (ยังไม่ต้องใช้ subtest) อย่างน้อย 3 กรณี
2. เขียนฟังก์ชัน `Percentage(part, total float64) (float64, error)` ที่คืน error ถ้า `total` เป็น 0 แล้วเขียน test ครอบคลุมทั้ง happy path และ error path โดยใช้ `errors.Is` เช็ค error ที่เจาะจง
3. เขียน test helper function `assertNoError(t *testing.T, err error)` และ `assertError(t *testing.T, err error)` ที่เรียก `t.Helper()` แล้วนำไปใช้ลดโค้ดซ้ำในแบบฝึกหัดข้อ 2
4. เขียนฟังก์ชัน `Max(numbers []int) (int, error)` (คืนค่ามากที่สุดใน slice, error ถ้า slice ว่าง) แล้วทดสอบด้วย `t.Run` แยกกรณี: slice ปกติ, slice มีค่าเดียว, slice มีค่าลบ, slice ว่าง
5. ลองรัน `go test -run` ด้วย pattern ต่างๆ กับ test ที่เขียนไว้ในข้อ 1-4 (เช่น รันเฉพาะ package เดียว, รันเฉพาะ subtest เดียว) สังเกตว่าผลลัพธ์เปลี่ยนไปอย่างไร
6. เพิ่ม `TestMain` เข้าไปใน package จากข้อ 4 ที่พิมพ์ข้อความก่อนและหลังรัน test ทั้งหมด แล้วลองลบ `os.Exit(code)` ออกดู สังเกตว่า `go test` ยังรายงานผลถูกต้องหรือไม่ (คำใบ้: ลองทำให้ test บางตัว fail ดูว่า exit code ของคำสั่ง `go test` เปลี่ยนไปหรือเปล่าด้วยคำสั่ง `echo $?` ต่อท้าย)

---

**ต่อไป**: [Part 034 — Table-Driven Tests และ Benchmarks](./034-table-driven-tests-and-benchmarks.md)
