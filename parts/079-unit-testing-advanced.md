# Part 079: Unit Testing ขั้นสูง

> ภาคที่ 7: Testing, Tooling, Performance — ตอนที่ 1 จาก 9 (Part 79–87)

## สารบัญของบทนี้

1. ทบทวนสิ่งที่รู้แล้วจาก Part 033–035 และภาพรวมของบทนี้
2. Test Fixtures และ `TestMain` แบบเจาะลึก
3. `t.TempDir()`: ไฟล์และโฟลเดอร์ชั่วคราวที่ลบให้อัตโนมัติ
4. `t.Cleanup()`: การันตี Teardown ไม่ว่า Test จะจบแบบไหน
5. `t.Parallel()` และการรัน Test แบบขนาน
6. กับดัก Loop Variable Capture กับ Parallel Subtests (และทำไม Go 1.22 ยังไม่ได้แก้ทุกกรณี)
7. Fuzzing: ให้ Go หา Input ที่ทำให้โค้ดพังเอง
8. เขียน Fuzz Test ตัวแรก และตามล่าบั๊กจริงด้วย `go test -fuzz`
9. ทดสอบโค้ดที่พึ่งพา Environment Variable ด้วย `t.Setenv`
10. ทดสอบโค้ดที่พึ่งพาเวลาด้วย Dependency Injection ของ `Clock`
11. สรุปสิ่งที่ได้เรียนในบทนี้
12. แบบฝึกหัดท้ายบท

---

## 1. ทบทวนสิ่งที่รู้แล้วจาก Part 033–035 และภาพรวมของบทนี้

ใน **ภาคที่ 2** เราวางรากฐานเรื่อง testing ไว้ครบแล้ว:

- **Part 033** สอนไฟล์ `_test.go`, ฟังก์ชัน `TestXxx(t *testing.T)`, `t.Error`/`t.Fatal`, `t.Helper()`, `t.Run` สำหรับ subtest, และแนะนำ `TestMain` เบื้องต้นสำหรับ setup/teardown ระดับ package
- **Part 034** สอน table-driven tests แบบเต็มรูปแบบ และ benchmark พื้นฐานด้วย `func BenchmarkXxx(b *testing.B)`
- **Part 035** สอนเรื่อง test doubles (dummy, stub, fake, spy, mock) และการออกแบบด้วย consumer-defined interface เพื่อให้เทสง่าย

บทนี้ไม่ได้สอนพื้นฐานซ้ำ แต่จะพาไปลึกกว่านั้นในประเด็นที่โปรเจกต์ระดับโปรดักชันจริงต้องเจอ: การจัดการ fixture ที่ซับซ้อนขึ้น, การทำงานกับไฟล์ชั่วคราวอย่างปลอดภัย, การรัน test คู่ขนานให้เร็วขึ้นโดยไม่เกิด race, การให้ Go **ค้นหาบั๊กเอง** ด้วย fuzzing ซึ่งเป็นความสามารถที่ภาษาโปรแกรมมิ่งกระแสหลักส่วนใหญ่ไม่มีในตัว, และเทคนิคการออกแบบโค้ดให้เทสได้ง่ายเมื่อโค้ดนั้นพึ่งพาสิ่งที่ "ควบคุมไม่ได้" อย่าง environment variable หรือเวลาปัจจุบัน

โค้ดทุกตัวอย่างในบทนี้ถูกรันจริงด้วย `go test` (รวมถึง `-race`, `-fuzz`) บนเครื่องที่ใช้เขียนบทเรียนนี้แล้ว ยืนยันว่าคอมไพล์ผ่านและพฤติกรรมตรงตามที่อธิบายทุกจุด

---

## 2. Test Fixtures และ `TestMain` แบบเจาะลึก

**Fixture** คือ "สภาพแวดล้อมที่เตรียมไว้ล่วงหน้า" สำหรับให้ test ใช้งาน เช่น การเชื่อมต่อฐานข้อมูล, ไฟล์ตัวอย่าง, หรือ object ที่สร้างขึ้นมาครั้งเดียวแล้วใช้ซ้ำในหลาย test เพื่อประหยัดเวลา

Part 033 แนะนำ `TestMain` ไปแล้วในระดับพื้นฐาน มาดูรูปแบบที่ใช้จริงกับ fixture ที่มีต้นทุนสูง (สมมติว่าเป็นการเปิด connection หรือ migrate schema):

```go
package fixture

import "fmt"

// Store จำลอง "การเชื่อมต่อฐานข้อมูล" ที่มีต้นทุนในการสร้างสูง
// เช่น เปิด connection pool, migrate schema ฯลฯ
type Store struct {
	data   map[string]string
	closed bool
}

func OpenStore() (*Store, error) {
	// สมมติว่ามีค่าใช้จ่ายสูง เช่น connect ไป container ทดสอบ
	return &Store{data: make(map[string]string)}, nil
}

func (s *Store) Close() error {
	s.closed = true
	return nil
}

func (s *Store) Put(key, value string) {
	s.data[key] = value
}

func (s *Store) Get(key string) (string, bool) {
	v, ok := s.data[key]
	return v, ok
}
```

และไฟล์ test ที่ใช้ `TestMain` เป็น fixture ระดับ package:

```go
package fixture

import (
	"os"
	"testing"
)

// sharedStore คือ fixture ระดับ package ที่สร้างครั้งเดียวแล้วใช้ร่วมกันทุก test
// ในไฟล์นี้ ผ่าน TestMain
var sharedStore *Store

func TestMain(m *testing.M) {
	// --- Setup: ทำงานก่อน test ทุกตัวในไฟล์นี้เริ่มรัน ---
	s, err := OpenStore()
	if err != nil {
		panic(err)
	}
	sharedStore = s
	s.Put("seed-user", "alice")

	// รัน test ทั้งหมด แล้วเก็บ exit code ไว้
	code := m.Run()

	// --- Teardown: ทำงานหลัง test ทุกตัวรันเสร็จ ไม่ว่าจะผ่านหรือไม่ผ่าน ---
	if err := sharedStore.Close(); err != nil {
		panic(err)
	}

	os.Exit(code)
}

func TestSharedStoreHasSeedData(t *testing.T) {
	v, ok := sharedStore.Get("seed-user")
	if !ok || v != "alice" {
		t.Fatalf("expected seed-user=alice, got %q (ok=%v)", v, ok)
	}
}

func TestSharedStoreCanAddMore(t *testing.T) {
	sharedStore.Put("second-user", "bob")
	v, ok := sharedStore.Get("second-user")
	if !ok || v != "bob" {
		t.Fatalf("expected second-user=bob, got %q (ok=%v)", v, ok)
	}
}
```

รันแล้วได้ผล:

```
=== RUN   TestSharedStoreHasSeedData
--- PASS: TestSharedStoreHasSeedData (0.00s)
=== RUN   TestSharedStoreCanAddMore
--- PASS: TestSharedStoreCanAddMore (0.00s)
PASS
```

### ประเด็นสำคัญที่ต้องระวังกับ `TestMain`

1. **`TestMain` มีได้แค่ 1 ตัวต่อ package** — ถ้าประกาศซ้ำจะ compile error
2. **`m.Run()` ต้องถูกเรียกเสมอ** — ถ้าลืมเรียก test ทั้งหมดในไฟล์จะไม่รันเลยแม้ `go test` จะรายงานว่า "ok" (เพราะไม่มี test ถูก execute จริง)
3. **ต้องเรียก `os.Exit(code)` ด้วย exit code ที่ได้จาก `m.Run()`** ไม่ใช่ `os.Exit(0)` เสมอ ไม่งั้น CI จะไม่รู้ว่า test fail
4. **`TestMain` ทำงานระดับ package ไม่ใช่ระดับไฟล์** — ถ้ามีหลายไฟล์ `_test.go` ใน package เดียวกัน setup/teardown นี้ครอบคลุมทุกไฟล์
5. Fixture ที่ใช้ `TestMain` เหมาะกับสิ่งที่ **สร้างครั้งเดียวแล้วปลอดภัยที่จะใช้ร่วมกัน** เท่านั้น ถ้า test แต่ละตัวต้องการสถานะที่ "สะอาด" แยกจากกัน ควรใช้ fixture ระดับ test แทน (ดูหัวข้อถัดไป)

---

## 3. `t.TempDir()`: ไฟล์และโฟลเดอร์ชั่วคราวที่ลบให้อัตโนมัติ

หลายครั้งโค้ดที่ต้องเทสต้องทำงานกับไฟล์จริงบนดิสก์ — โหลด config, เขียน log, export รายงาน ฯลฯ สมัยก่อนต้องเขียนโค้ดแบบนี้เอง:

```go
dir, err := os.MkdirTemp("", "myapp-test")
if err != nil { t.Fatal(err) }
defer os.RemoveAll(dir) // ต้องจำเรียกเองทุกครั้ง ลืมไม่ได้
```

Go มีวิธีที่สั้นและปลอดภัยกว่าคือ `t.TempDir()` ซึ่งสร้างโฟลเดอร์ชั่วคราวที่**ไม่ซ้ำกัน**สำหรับแต่ละ test และ**ลบทิ้งให้อัตโนมัติ**เมื่อ test (รวมถึง subtest ทั้งหมดของมัน) จบลง ไม่ว่า test จะผ่าน ล้มเหลว หรือ panic ก็ตาม

โค้ดที่จะทดสอบ:

```go
package tempfiles

import (
	"os"
	"path/filepath"
)

// LoadConfig อ่านไฟล์ config แบบง่ายๆ จาก path ที่กำหนด
func LoadConfig(path string) (string, error) {
	data, err := os.ReadFile(path)
	if err != nil {
		return "", err
	}
	return string(data), nil
}

// WriteLogEntry เขียน log ลงไฟล์ใน directory ที่กำหนด (append)
func WriteLogEntry(dir, entry string) error {
	f, err := os.OpenFile(filepath.Join(dir, "app.log"), os.O_APPEND|os.O_CREATE|os.O_WRONLY, 0o644)
	if err != nil {
		return err
	}
	defer f.Close()
	_, err = f.WriteString(entry + "\n")
	return err
}
```

Test ที่ใช้ `t.TempDir()`:

```go
package tempfiles

import (
	"os"
	"path/filepath"
	"testing"
)

func TestLoadConfig(t *testing.T) {
	// t.TempDir() สร้าง directory ชั่วคราวที่ไม่ซ้ำกันสำหรับ test นี้โดยเฉพาะ
	// และ Go จะลบทิ้งให้อัตโนมัติเมื่อ test (รวมถึง subtest ทั้งหมด) จบลง
	// ไม่ต้องเขียน os.RemoveAll เองแบบสมัยก่อน
	dir := t.TempDir()

	cfgPath := filepath.Join(dir, "config.yaml")
	want := "port: 8080\nname: myapp\n"
	if err := os.WriteFile(cfgPath, []byte(want), 0o644); err != nil {
		t.Fatalf("write config: %v", err)
	}

	got, err := LoadConfig(cfgPath)
	if err != nil {
		t.Fatalf("LoadConfig() error = %v", err)
	}
	if got != want {
		t.Errorf("LoadConfig() = %q, want %q", got, want)
	}
}
```

รันแล้วผ่านตามคาด:

```
=== RUN   TestLoadConfig
--- PASS: TestLoadConfig (0.00s)
```

**ข้อดีของ `t.TempDir()` เทียบกับ `os.MkdirTemp` เอง:**

- ลบให้อัตโนมัติเสมอ แม้ test จะ panic กลางทาง (ต่างจาก `defer` ที่ถ้า process ถูกฆ่าแบบรุนแรง หรือลืมเขียน `defer` เองก็จะรั่ว)
- โฟลเดอร์ตั้งชื่อตาม test name อัตโนมัติ ทำให้ debug ง่ายเวลาต้องดูว่าไฟล์ไหนของ test ไหน
- ถ้าเรียกจาก subtest (ผ่าน `t.Run`) แต่ละ subtest จะได้โฟลเดอร์ของตัวเอง ไม่ชนกัน

---

## 4. `t.Cleanup()`: การันตี Teardown ไม่ว่า Test จะจบแบบไหน

`t.Cleanup(func())` ลงทะเบียนฟังก์ชันที่จะถูกเรียก **หลังจาก test (และ subtest ทั้งหมดของมัน) จบการทำงาน** ไม่ว่าจะ pass, fail, หรือ panic ก็ตาม พฤติกรรมนี้คล้าย `defer` แต่มีข้อแตกต่างสำคัญที่ทำให้ `t.Cleanup` มีประโยชน์กว่าในหลายสถานการณ์:

1. **เรียกจาก helper function ใดๆ ก็ได้** ไม่จำเป็นต้องอยู่ในฟังก์ชัน `TestXxx` โดยตรงเหมือน `defer` (ซึ่งผูกกับ scope ของฟังก์ชันที่เรียกมันเท่านั้น)
2. **cleanup หลายตัวถูกเรียกตามลำดับ LIFO** (เข้าทีหลังออกก่อน) เหมือน `defer` ทำให้จัดการ resource ที่ต้องปิดตามลำดับย้อนกลับได้ถูกต้อง
3. **ทำงานร่วมกับ parallel subtest ได้ถูกต้อง** — cleanup จะรอให้ subtest ที่เป็น parallel ทั้งหมดรันจบก่อนเสมอ (จุดนี้สำคัญมาก จะกลับมาอธิบายละเอียดในหัวข้อ 6)

```go
func TestWriteLogEntry_WithCleanup(t *testing.T) {
	dir := t.TempDir()

	// t.Cleanup ลงทะเบียนฟังก์ชันที่จะถูกเรียกตอน test จบ (ไม่ว่าจะ pass, fail, หรือ panic)
	// ต่างจาก defer ตรงที่ t.Cleanup เรียกใช้ได้จาก helper function ใดๆ ก็ได้
	var closed bool
	t.Cleanup(func() {
		closed = true
	})

	if err := WriteLogEntry(dir, "hello world"); err != nil {
		t.Fatalf("WriteLogEntry() error = %v", err)
	}

	data, err := os.ReadFile(filepath.Join(dir, "app.log"))
	if err != nil {
		t.Fatalf("read log: %v", err)
	}
	if string(data) != "hello world\n" {
		t.Errorf("log content = %q", string(data))
	}

	if closed {
		t.Fatal("cleanup ไม่ควรถูกเรียกก่อน test จบ")
	}
}
```

จุดที่มีประโยชน์มากที่สุดคือการใช้ใน **test helper** ที่ทำหน้าที่ setup ให้ test อื่นเรียกใช้ โดยที่ตัว helper เองเป็นผู้รับผิดชอบลงทะเบียน cleanup แทน caller ทำให้ caller ไม่ต้องจำว่าต้อง teardown เอง:

```go
// setupTempStore เป็นตัวอย่าง test helper ที่ลงทะเบียน t.Cleanup แทน caller
// เพื่อให้ caller ไม่ต้องจำว่าต้องเรียก teardown เอง
func setupTempStore(t *testing.T) string {
	t.Helper()
	dir := t.TempDir()
	t.Cleanup(func() {
		// ในสถานการณ์จริงอาจเป็นการปิด connection หรือลบ resource ภายนอก
		t.Log("cleaning up temp store at", dir)
	})
	return dir
}

func TestWriteLogEntry_HelperRegistersCleanup(t *testing.T) {
	dir := setupTempStore(t)
	if err := WriteLogEntry(dir, "from helper"); err != nil {
		t.Fatalf("WriteLogEntry() error = %v", err)
	}
}
```

ผลการรัน (สังเกตว่า log ของ cleanup ปรากฏหลังบรรทัด assert ทั้งหมดใน test):

```
=== RUN   TestWriteLogEntry_HelperRegistersCleanup
    configloader_test.go:66: cleaning up temp store at /tmp/TestWriteLogEntry_HelperRegistersCleanup3589539076/001
--- PASS: TestWriteLogEntry_HelperRegistersCleanup (0.00s)
```

> **หลักการเลือกใช้**: ถ้า cleanup logic เรียบง่ายและอยู่ใน `TestXxx` ตรงๆ ใช้ `defer` ก็เพียงพอและอ่านง่ายกว่า แต่ถ้าเป็น logic ที่ใช้ซ้ำในหลาย test ผ่าน helper function หรือถ้าต้องการ cleanup ที่รอ parallel subtest ให้เสร็จก่อนแน่นอน ให้ใช้ `t.Cleanup()`

---

## 5. `t.Parallel()` และการรัน Test แบบขนาน

เมื่อ test suite มีหลายร้อยหรือหลายพัน test การรันแบบ sequential (ทีละตัว) อาจใช้เวลานานโดยไม่จำเป็น โดยเฉพาะ test ที่มีการรอ I/O (เช่น เรียก HTTP, อ่านไฟล์) ซึ่งส่วนใหญ่ของเวลาคือการ "รอ" ไม่ใช่การประมวลผล CPU

`t.Parallel()` บอกกับ `go test` ว่า "test นี้ปลอดภัยที่จะรันพร้อมกันกับ test อื่นที่ประกาศ `t.Parallel()` เช่นกัน"

```go
package parallel

import (
	"sync/atomic"
	"testing"
	"time"
)

// TestSlowOperations แสดงว่า t.Parallel() ทำให้ subtest หลายตัวรันพร้อมกันจริง
// โดยวัดเวลารวม: ถ้ารันแบบ sequential จะใช้เวลา ~4*sleepDuration
// แต่ถ้ารันแบบ parallel เวลารวมจะใกล้เคียง sleepDuration เดียว (บวก overhead เล็กน้อย)
func TestSlowOperations(t *testing.T) {
	const sleepDuration = 100 * time.Millisecond
	start := time.Now()

	names := []string{"a", "b", "c", "d"}
	for _, name := range names {
		name := name // ใน Go 1.22+ บรรทัดนี้ไม่จำเป็นแล้ว แต่ใส่ไว้เพื่อความชัดเจน/เข้ากันได้กับ go.mod เก่า
		t.Run(name, func(t *testing.T) {
			t.Parallel() // ประกาศว่า subtest นี้ "รันคู่ขนาน" กับ subtest อื่นที่ประกาศ t.Parallel() เช่นกัน
			time.Sleep(sleepDuration)
		})
	}

	t.Cleanup(func() {
		elapsed := time.Since(start)
		if elapsed >= 4*sleepDuration {
			t.Errorf("subtests ดูเหมือนไม่ได้รันคู่ขนานกัน: ใช้เวลา %v", elapsed)
		}
		t.Logf("total elapsed: %v (sequential เดิมจะใช้ ~%v)", elapsed, 4*sleepDuration)
	})
}
```

รันจริงแล้ววัดได้:

```
total elapsed: 101.475004ms (sequential เดิมจะใช้ ~400ms)
--- PASS: TestSlowOperations (0.00s)
    --- PASS: TestSlowOperations/d (0.10s)
    --- PASS: TestSlowOperations/b (0.10s)
    --- PASS: TestSlowOperations/c (0.10s)
    --- PASS: TestSlowOperations/a (0.10s)
```

subtest ทั้ง 4 ตัวใช้เวลารวมแค่ ~100ms แทนที่จะเป็น ~400ms — เห็นผลของการรันคู่ขนานชัดเจน

### พฤติกรรมที่มักเข้าใจผิด: parent test รอ parallel subtest เมื่อไหร่กันแน่

จุดที่มือใหม่ (และมือเก๋าที่เผลอ) มักพลาดคือ: เมื่อ subtest เรียก `t.Parallel()` ตัว subtest นั้นจะ **"หยุดรอ (pause)" ทันที** และคืนการควบคุมกลับให้ loop ของ parent ทำงานต่อไปจนจบฟังก์ชันก่อน subtest ที่ parallel ทั้งหมดจะเริ่ม **"รันจริง"** ก็ต่อเมื่อฟังก์ชัน parent test (บรรทัดสุดท้ายของ `TestXxx`) return แล้วเท่านั้น

```go
// TestParentWaitsForParallelChildren ยืนยันพฤติกรรมสำคัญที่มักเข้าใจผิด:
// เมื่อ subtest เรียก t.Parallel() ตัว subtest นั้นจะ "หยุดรอ (pause)" ทันที
// และคืนการควบคุมกลับให้ loop ของ parent ทำงานต่อไปจนจบฟังก์ชันก่อน
// subtest ที่ parallel ทั้งหมดจะเริ่ม "รันจริง" ก็ต่อเมื่อฟังก์ชัน parent test
// (บรรทัดสุดท้ายของ TestXxx) return แล้วเท่านั้น
func TestParentWaitsForParallelChildren(t *testing.T) {
	var completed int32

	for i := 0; i < 3; i++ {
		t.Run("child", func(t *testing.T) {
			t.Parallel()
			time.Sleep(20 * time.Millisecond)
			atomic.AddInt32(&completed, 1)
		})
	}

	// ผิด! ตรงนี้ completed จะยังเป็น 0 เสมอ เพราะ parallel subtest ยังไม่ได้เริ่มรัน
	// ณ จุดนี้ (พวกมันแค่ "ถูก pause" ไว้เท่านั้น)
	if got := atomic.LoadInt32(&completed); got != 0 {
		t.Errorf("completed ทันทีหลัง loop = %d, want 0 (นี่คือพฤติกรรมที่คาดหวัง ไม่ใช่บั๊ก)", got)
	}

	// ถูกต้อง: ใช้ t.Cleanup เพื่อตรวจสอบ "หลังจาก" parallel subtest ทั้งหมดรันจบจริง
	t.Cleanup(func() {
		got := atomic.LoadInt32(&completed)
		if got != 3 {
			t.Errorf("completed ใน Cleanup = %d, want 3", got)
		}
	})
}
```

รันแล้วผ่านทั้งสอง assertion ยืนยันพฤติกรรมตามที่อธิบาย: `completed` เป็น 0 ทันทีหลัง loop แต่เป็น 3 ใน `t.Cleanup` ที่รันหลัง parallel subtest ทั้งหมดเสร็จแล้ว นี่คือเหตุผลสำคัญอีกข้อที่ `t.Cleanup` มีประโยชน์กว่าการเขียนโค้ดตรงๆ ต่อจาก loop ของ `t.Run`

---

## 6. กับดัก Loop Variable Capture กับ Parallel Subtests

**Part 005** เคยพูดถึงการเปลี่ยนแปลงสำคัญใน **Go 1.22**: ตัวแปรลูป (loop variable) ใน `for` แต่ละรอบจะถูกสร้างเป็น **ตัวแปรใหม่แยกกันในแต่ละ iteration** แทนที่จะเป็นตัวแปรตัวเดียวที่ถูกเขียนทับซ้ำๆ แบบ Go เวอร์ชันก่อนหน้า การเปลี่ยนแปลงนี้แก้ปัญหาคลาสสิกเรื่อง "closure capture ตัวแปรลูปผิดตัว" ได้เกือบทั้งหมด — **เกือบ** เพราะยังมีจุดที่ต้องระวังอยู่ดี

### ทำไมปัญหานี้ถึงสำคัญมากกับ parallel subtests โดยเฉพาะ

การรวม `t.Run` + `t.Parallel()` ใน loop คือจุดที่บั๊กนี้ระเบิดชัดเจนที่สุด เพราะ subtest ที่ `t.Parallel()` จะ **pause แล้วรอ** ก่อนดังที่อธิบายในหัวข้อก่อนหน้า ทำให้ loop วิ่งผ่านไปเขียนทับตัวแปรก่อนที่ subtest จะได้ "อ่าน" ค่าที่ต้องการจริง ๆ (ถ้าใช้ semantics เก่า) — subtest ทั้งหมดจะไปเห็นค่าตัวแปรจาก iteration สุดท้ายพร้อมกันหมด

### พิสูจน์ด้วยการรันจริง: go.mod เก่ายังมีปัญหาอยู่

ประเด็นสำคัญที่หลายคนไม่รู้คือ: **`go` directive ใน `go.mod` เป็นตัวกำหนด semantics ของภาษาที่ compiler ใช้** ไม่ใช่ตัว Go toolchain ที่ติดตั้งอยู่ ต่อให้เครื่องมี Go 1.24 แต่ถ้า `go.mod` เขียนว่า `go 1.21` ตัว compiler จะยังใช้ **semantics แบบเก่า** (ตัวแปรลูปตัวเดียวถูกใช้ซ้ำ) กับไฟล์ในโมดูลนั้นทั้งหมด นี่คือเหตุผลที่ "โค้ดเก่าที่ไม่เคยแตะ `go.mod`" จึงยังคงมีบั๊กนี้อยู่แม้จะรันด้วย toolchain ใหม่ก็ตาม

มาพิสูจน์ด้วยการรันจริง สร้างสอง module ที่มีโค้ดทดสอบเหมือนกันทุกตัวอักษร ต่างกันแค่บรรทัด `go` ใน `go.mod`:

**Module A** (`go.mod` มี `go 1.21`):

```go
package loopcapold

import (
	"sync"
	"testing"
)

func TestParallelLoopCapture(t *testing.T) {
	cases := []struct {
		name string
		n    int
	}{
		{"one", 1}, {"two", 2}, {"three", 3}, {"four", 4}, {"five", 5},
	}

	var mu sync.Mutex
	seen := map[int]int{} // n -> จำนวนครั้งที่ subtest เห็นค่านี้จริง

	for _, tc := range cases {
		t.Run(tc.name, func(t *testing.T) {
			t.Parallel()
			mu.Lock()
			seen[tc.n]++
			mu.Unlock()
		})
	}

	t.Cleanup(func() {
		mu.Lock()
		defer mu.Unlock()
		for _, tc := range cases {
			if seen[tc.n] != 1 {
				t.Logf("BUG: n=%d ถูกเห็น %d ครั้ง (ควรเป็น 1)", tc.n, seen[tc.n])
			}
		}
	})
}
```

รันด้วย `go.mod` ที่มี `go 1.21`:

```
BUG: n=1 ถูกเห็น 0 ครั้ง (ควรเป็น 1)
BUG: n=2 ถูกเห็น 0 ครั้ง (ควรเป็น 1)
BUG: n=3 ถูกเห็น 0 ครั้ง (ควรเป็น 1)
BUG: n=4 ถูกเห็น 0 ครั้ง (ควรเป็น 1)
BUG: n=5 ถูกเห็น 5 ครั้ง (ควรเป็น 1)
--- PASS: TestParallelLoopCapture (0.00s)
```

ทุก subtest เห็นค่า `n=5` (ค่าจาก iteration สุดท้าย) พร้อมกันหมด — เกิดบั๊กจริงตามที่คาด แม้ toolchain ที่รันจะเป็น Go 1.24.7 ก็ตาม เพราะ `go.mod` บอกให้ compile ด้วย semantics แบบ Go 1.21

รันโค้ด**เดียวกันเป๊ะ**อีกครั้ง แต่เปลี่ยน `go.mod` เป็น `go 1.24`:

```
--- PASS: TestParallelLoopCapture (0.00s)
    --- PASS: TestParallelLoopCapture/one (0.00s)
    --- PASS: TestParallelLoopCapture/three (0.00s)
    --- PASS: TestParallelLoopCapture/two (0.00s)
    --- PASS: TestParallelLoopCapture/five (0.00s)
    --- PASS: TestParallelLoopCapture/four (0.00s)
```

คราวนี้ไม่มี log "BUG" เลยสักบรรทัด — แต่ละ subtest เห็นค่า `tc.n` ของตัวเองถูกต้องครบทั้ง 5 ค่า เพราะ Go 1.22+ ให้ตัวแปรลูปใหม่ทุก iteration โดยอัตโนมัติ

### วิธีป้องกันแบบไม่ต้องพึ่ง `go.mod` เวอร์ชัน: shadow ตัวแปรด้วยมือ

ถ้าโค้ดของทีมยังต้องรองรับ `go.mod` เวอร์ชันเก่ากว่า 1.22 (เช่น library ที่ต้องรองรับผู้ใช้หลากหลายเวอร์ชัน) วิธีป้องกันแบบเดิมยังใช้ได้เสมอ คือสร้างตัวแปรใหม่ shadow ตัวแปรลูปก่อนเข้า closure:

```go
for _, tc := range cases {
	tc := tc // shadow ตัวแปรลูปด้วยตัวแปรใหม่ในแต่ละรอบ (จำเป็นสำหรับ go.mod ที่ go < 1.22)
	t.Run(tc.name, func(t *testing.T) {
		t.Parallel()
		mu.Lock()
		seen[tc.n]++
		mu.Unlock()
	})
}
```

รันด้วยโค้ดที่แก้แล้วบน `go.mod` เวอร์ชัน 1.21 เดิม: ผ่านทุก subtest โดยไม่มี `BUG` log ปรากฏเลย เพราะการ shadow `tc := tc` สร้างตัวแปรใหม่ในหน่วยความจำแยกกันทุกรอบ ไม่ว่า compiler จะใช้ semantics แบบเก่าหรือใหม่ก็ตาม

> **ข้อสรุปเชิงปฏิบัติ**: แม้โปรเจกต์จะอัปเดต `go.mod` เป็น 1.22+ แล้ว การเขียน `tc := tc` (หรือตั้งชื่อพารามิเตอร์ให้ชัดเจนในฟังก์ชัน) ก่อน `t.Run` ยังเป็นนิสัยที่ดีที่ควรรักษาไว้ เพราะ (1) ทำให้โค้ดสื่อความหมายชัดเจนไม่ต้องพึ่งความจำเรื่อง Go version, (2) ป้องกันบั๊กทันทีถ้าใครเผลอลด `go` directive ลง หรือโค้ดถูกก็อปปี้ไปใช้ในโมดูลที่ยังไม่ได้อัปเดต, และ (3) แม้ Go 1.22 จะแก้ปัญหานี้กับ `for...range` ได้ แต่ pattern การ capture ตัวแปรที่ share กันในลักษณะอื่น (เช่น pointer หรือ index ที่ถูกแก้ไขภายหลังนอก loop) ยังคงเป็นเรื่องที่ต้องระวังด้วยเหตุผลอื่นอยู่ดี

---

## 7. Fuzzing: ให้ Go หา Input ที่ทำให้โค้ดพังเอง

Table-driven test (Part 034) ทดสอบด้วย input ที่ **เราเป็นคนคิดขึ้นเอง** — ปัญหาคือเรามักคิดไม่ถึงกรณีประหลาดๆ ที่ทำให้โค้ดพัง โดยเฉพาะกับโค้ดที่รับ string, byte slice หรือข้อมูลจากภายนอกที่ควบคุมรูปแบบไม่ได้ทั้งหมด

**Fuzzing** (เพิ่มเข้ามาใน Go standard toolchain ตั้งแต่ **Go 1.18**) คือเทคนิคที่ให้เครื่องมือสุ่มสร้าง input จำนวนมหาศาลป้อนเข้าฟังก์ชัน โดยเริ่มจาก "seed corpus" ที่เราให้ไว้ แล้ว **กลายพันธุ์ (mutate)** ค่าเหล่านั้นไปเรื่อยๆ พร้อมวัด code coverage แบบ real-time เพื่อพยายามสำรวจ code path ใหม่ๆ ที่ยังไม่เคยถูกทดสอบ จนกว่าจะเจอ input ที่ทำให้เกิด panic, error ที่ไม่ควรเกิด, หรือ property บางอย่างที่เราระบุไว้ผิดพลาด

จุดเด่นของ fuzzing ใน Go คือ **เป็นส่วนหนึ่งของ `go test` toolchain โดยตรง** ไม่ต้องติดตั้ง library ภายนอกเพิ่มเลย

---

## 8. เขียน Fuzz Test ตัวแรก และตามล่าบั๊กจริงด้วย `go test -fuzz`

มาดูตัวอย่างจริง: ฟังก์ชัน `Reverse` ที่กลับด้าน string โดย**ตั้งใจใส่บั๊ก**ไว้เพื่อสาธิตว่า fuzzing เจอมันได้เร็วแค่ไหน

```go
package fuzzpkg

import "unicode/utf8"

// Reverse กลับด้าน string โดยพยายามรองรับ UTF-8 (multi-byte rune)
// ฟังก์ชันนี้ตั้งใจใส่บั๊กไว้เพื่อสาธิตว่า fuzzing เจอ input ที่ทำให้พัง
// (บั๊ก: วนลูปทีละ "byte" แทนที่จะเป็นทีละ "rune" ทำให้ multi-byte UTF-8
// ถูกตัดกลางตัวอักษร เกิด invalid UTF-8 sequence)
func Reverse(s string) (string, error) {
	if !utf8.ValidString(s) {
		return "", errInvalidUTF8
	}
	b := []byte(s)
	for i, j := 0, len(b)-1; i < j; i, j = i+1, j-1 {
		b[i], b[j] = b[j], b[i]
	}
	out := string(b)
	if !utf8.ValidString(out) {
		return "", errInvalidUTF8
	}
	return out, nil
}
```

Test ทั้ง unit test ปกติ และ fuzz test:

```go
package fuzzpkg

import (
	"testing"
	"unicode/utf8"
)

// unit test พื้นฐานแบบ table-driven (ทวนความรู้จาก Part 034) — ผ่านหมดถ้าทดสอบแค่ ASCII
func TestReverse_ASCII(t *testing.T) {
	cases := []struct {
		in, want string
	}{
		{"hello", "olleh"},
		{"", ""},
		{"a", "a"},
		{"golang", "gnalog"},
	}
	for _, tc := range cases {
		got, err := Reverse(tc.in)
		if err != nil {
			t.Fatalf("Reverse(%q) unexpected error: %v", tc.in, err)
		}
		if got != tc.want {
			t.Errorf("Reverse(%q) = %q, want %q", tc.in, got, tc.want)
		}
	}
}

// FuzzReverse คือ fuzz test ของ Go (เพิ่มมาตั้งแต่ Go 1.18)
// รูปแบบ: func FuzzXxx(f *testing.F) รับ *testing.F แทน *testing.T
func FuzzReverse(f *testing.F) {
	// f.Add ใส่ "seed corpus" — ตัวอย่าง input เริ่มต้นที่ fuzzer จะใช้เป็นฐาน
	// ในการ mutate สร้าง input ใหม่ๆ ต่อไป
	f.Add("hello")
	f.Add("")
	f.Add("Go is fun")

	f.Fuzz(func(t *testing.T, s string) {
		got, err := Reverse(s)
		if err != nil {
			// ถ้า input เป็น UTF-8 ที่ถูกต้อง แต่ Reverse คืน error แปลว่ามีบั๊ก
			if utf8.ValidString(s) {
				t.Fatalf("Reverse(%q) returned error on valid UTF-8 input: %v", s, err)
			}
			return // input ไม่ valid UTF-8 ตั้งแต่แรก ข้ามไปได้
		}

		// property-based check: กลับด้านสองครั้งต้องได้ string เดิม
		back, err := Reverse(got)
		if err != nil {
			t.Fatalf("Reverse(Reverse(%q)) returned error: %v", s, err)
		}
		if back != s {
			t.Errorf("Reverse(Reverse(%q)) = %q, want %q", s, back, s)
		}
	})
}
```

**สังเกตโครงสร้างของ `FuzzXxx`:**

- รับพารามิเตอร์ `f *testing.F` (ไม่ใช่ `*testing.T`)
- `f.Add(...)` ใส่ seed corpus ได้หลายค่า ต้องมี type และจำนวน parameter ตรงกับที่ `f.Fuzz` รับ
- `f.Fuzz(func(t *testing.T, ...))` รับ closure ที่พารามิเตอร์ตัวแรกเป็น `*testing.T` เสมอ ตามด้วย parameter ที่จะถูกสุ่มค่า (รองรับ `string`, `[]byte`, และชนิดตัวเลขพื้นฐานทั้งหมด)

รัน unit test ปกติก่อน — ผ่านทุก case:

```
=== RUN   TestReverse_ASCII
--- PASS: TestReverse_ASCII (0.00s)
```

ตอนนี้มาสั่งให้ fuzzer ทำงานจริงด้วย flag `-fuzz`:

```bash
go test -fuzz=FuzzReverse -fuzztime=30s
```

ผลลัพธ์จริงจากการรัน (ใช้เวลาไม่ถึง 1 วินาทีก็เจอบั๊กแล้ว):

```
fuzz: elapsed: 0s, gathering baseline coverage: 0/3 completed
fuzz: elapsed: 0s, gathering baseline coverage: 3/3 completed, now fuzzing with 4 workers
fuzz: minimizing 35-byte failing input file
fuzz: elapsed: 0s, minimizing
--- FAIL: FuzzReverse (0.04s)
    --- FAIL: FuzzReverse (0.00s)
        reverse_test.go:43: Reverse("ǰ") returned error on valid UTF-8 input: fuzzpkg: invalid UTF-8 string

Failing input written to testdata/fuzz/FuzzReverse/af6bd977d8baad89
To re-run:
go test -run=FuzzReverse/af6bd977d8baad89
```

Fuzzer เจอว่า string `"ǰ"` (ตัวอักษร U+01F0 ที่เข้ารหัสด้วย 2 byte ใน UTF-8) ทำให้ `Reverse` คืน error ทั้งที่ input เป็น UTF-8 ที่ถูกต้อง — สาเหตุคือฟังก์ชันกลับด้าน**ทีละ byte** แทนที่จะเป็นทีละ **rune** ทำให้ multi-byte character ถูกตัดกลางตัวจนกลายเป็น sequence ที่ไม่ใช่ UTF-8 ที่ถูกต้องอีกต่อไป

สิ่งสำคัญคือ Go **บันทึก failing case ไว้เป็นไฟล์** ที่ `testdata/fuzz/FuzzReverse/af6bd977d8baad89` โดยอัตโนมัติ:

```
go test fuzz v1
string("ǰ")
```

ไฟล์นี้จะถูกรันเป็น **regression test ปกติทุกครั้งที่สั่ง `go test`** ต่อจากนี้ไป (ไม่ต้องสั่ง `-fuzz` อีก) ทำให้มั่นใจได้ว่าบั๊กเดิมจะไม่กลับมาอีกถ้าใครแก้โค้ดพลาดในอนาคต:

```
=== RUN   FuzzReverse
=== RUN   FuzzReverse/seed#0
=== RUN   FuzzReverse/seed#1
=== RUN   FuzzReverse/seed#2
=== RUN   FuzzReverse/af6bd977d8baad89
    reverse_test.go:43: Reverse("ǰ") returned error on valid UTF-8 input: fuzzpkg: invalid UTF-8 string
--- FAIL: FuzzReverse (0.00s)
```

### แก้บั๊กให้ถูกต้อง

วิธีแก้คือแปลง string เป็น `[]rune` ก่อนกลับด้าน แทนที่จะเป็น `[]byte`:

```go
// Reverse กลับด้าน string โดยรองรับ UTF-8 (multi-byte rune) อย่างถูกต้อง
// โดยแปลงเป็น []rune ก่อนกลับด้าน แทนที่จะกลับด้านทีละ byte
func Reverse(s string) (string, error) {
	if !utf8.ValidString(s) {
		return "", errInvalidUTF8
	}
	r := []rune(s)
	for i, j := 0, len(r)-1; i < j; i, j = i+1, j-1 {
		r[i], r[j] = r[j], r[i]
	}
	return string(r), nil
}
```

รัน fuzz test อีกครั้งด้วยเวลาที่นานขึ้น (20 วินาที) เพื่อความมั่นใจ:

```bash
go test -fuzz=FuzzReverse -fuzztime=20s
```

```
fuzz: elapsed: 0s, gathering baseline coverage: 0/7 completed
fuzz: elapsed: 0s, gathering baseline coverage: 7/7 completed, now fuzzing with 4 workers
fuzz: elapsed: 3s, execs: 75418 (25130/sec), new interesting: 18 (total: 25)
fuzz: elapsed: 20s, execs: 1250415 (57757/sec), new interesting: 31 (total: 38)
PASS
ok  	example.com/unitadv/fuzzpkg	20.087s
```

รันไปกว่า **1.25 ล้านครั้ง** ในเวลา 20 วินาที ไม่พบ input ใหม่ที่ทำให้พังอีกเลย — มั่นใจได้มากกว่าการทดสอบด้วย test case ที่เขียนเองไม่กี่กรณีอย่างมาก

> **เมื่อไหร่ควรใช้ fuzzing**: เหมาะกับฟังก์ชันที่ (1) รับ input ประเภท string/bytes/ตัวเลขที่มาจากแหล่งภายนอก (parser, decoder, validator), (2) มี "property" ที่ตรวจสอบได้ชัดเจนโดยไม่ต้องรู้คำตอบล่วงหน้า เช่น "encode แล้ว decode กลับต้องได้ค่าเดิม" (round-trip), "ต้องไม่ panic ไม่ว่า input จะเป็นอะไร" ฯลฯ ไม่เหมาะกับ business logic ที่ผลลัพธ์ถูกต้องขึ้นอยู่กับกฎเฉพาะทางที่ต้องระบุ input/output คู่กันตรงๆ (ใช้ table-driven test แทน)

---

## 9. ทดสอบโค้ดที่พึ่งพา Environment Variable ด้วย `t.Setenv`

โค้ดที่อ่านค่าจาก `os.Getenv` โดยตรงมักเทสยาก เพราะการตั้งค่า environment variable ด้วย `os.Setenv` แล้วลืมคืนค่าเดิม จะทำให้ test อื่นที่รันต่อ (โดยเฉพาะเมื่อรันแบบ parallel) ได้รับผลกระทบโดยไม่ได้ตั้งใจ Go จึงมี `t.Setenv(key, value)` ที่**ตั้งค่า environment variable และคืนค่าเดิมกลับให้อัตโนมัติเมื่อ test จบ** (ทำงานคล้าย `t.Cleanup` ภายใน):

```go
func isFeatureEnabled(envKey string) bool {
	return os.Getenv(envKey) == "true"
}

func TestFeatureFlagFromEnv(t *testing.T) {
	t.Setenv("FEATURE_NEW_UI", "true")

	if !isFeatureEnabled("FEATURE_NEW_UI") {
		t.Error("expected feature to be enabled")
	}
}

func TestFeatureFlagFromEnv_Disabled(t *testing.T) {
	// ไม่ได้ตั้งค่า env ตัวนี้ในทดสอบนี้ ดังนั้นควรเป็น false เสมอ
	// ไม่ว่า test ก่อนหน้าจะตั้งค่าอะไรไว้ก็ตาม เพราะ t.Setenv คืนค่าที่ถูกต้องเสมอ
	if isFeatureEnabled("FEATURE_NEW_UI") {
		t.Error("expected feature to be disabled by default")
	}
}
```

ทั้งสอง test รันผ่านอย่างอิสระจากกัน ไม่ว่าจะรันตามลำดับไหน เพราะ `t.Setenv` จัดการคืนค่าให้เองเสมอ

> **ข้อจำกัดสำคัญ**: `t.Setenv` **ใช้ร่วมกับ `t.Parallel()` ในระดับเดียวกันไม่ได้** — ถ้าเรียก `t.Setenv` ใน test ที่ประกาศ `t.Parallel()` แล้ว Go จะ panic ทันทีตอนรัน เพราะ environment variable เป็น global state ของ process การรัน parallel พร้อมกับเปลี่ยนค่านี้จะทำให้ test อื่นที่รันคู่ขนานเห็นค่าที่ไม่ถูกต้องได้

---

## 10. ทดสอบโค้ดที่พึ่งพาเวลาด้วย Dependency Injection ของ `Clock`

ปัญหาคลาสสิกอีกข้อคือโค้ดที่เรียก `time.Now()` ตรงๆ ภายใน business logic ทำให้ผลลัพธ์ของ test **ไม่ deterministic** — รันตอนนี้ได้ผลหนึ่ง รันพรุ่งนี้ได้อีกผลหนึ่ง หรือแย่กว่านั้นคือ test แบบ "รอ 1 วินาทีแล้วเช็คว่าหมดอายุหรือยัง" ที่ทำให้ test ช้าและ flaky (บางครั้งผ่านบางครั้งไม่ผ่านเพราะ timing คลาดเคลื่อนเล็กน้อย)

ทางแก้มาตรฐานคือทำ **dependency injection**: ห่อการเรียก `time.Now()` ไว้หลัง interface เล็กๆ แล้วส่ง (inject) เข้าไปในโค้ดที่ต้องการ แทนที่จะเรียกฟังก์ชัน global ตรงๆ

```go
package clockpkg

import "time"

// Clock คือ interface เล็กๆ ที่ห่อหุ้มการเรียก time.Now()
// การ inject interface นี้เข้าไปแทนที่จะเรียก time.Now() ตรงๆ ในโค้ด business logic
// ทำให้ unit test สามารถควบคุม "เวลาปัจจุบัน" ได้อย่างแน่นอน (deterministic)
type Clock interface {
	Now() time.Time
}

// RealClock คือ implementation จริงที่ใช้งานตอน production
type RealClock struct{}

func (RealClock) Now() time.Time {
	return time.Now()
}

// FixedClock คือ implementation สำหรับ test ที่คืนเวลาคงที่เสมอ
type FixedClock struct {
	FixedTime time.Time
}

func (c FixedClock) Now() time.Time {
	return c.FixedTime
}

// Session คือตัวอย่าง business logic ที่ต้องพึ่งพา "เวลาปัจจุบัน"
type Session struct {
	clock     Clock
	CreatedAt time.Time
	TTL       time.Duration
}

// NewSession สร้าง session ใหม่โดยรับ Clock เข้ามา (dependency injection)
// แทนที่จะเรียก time.Now() ตรงๆ ภายในฟังก์ชัน
func NewSession(clock Clock, ttl time.Duration) *Session {
	return &Session{
		clock:     clock,
		CreatedAt: clock.Now(),
		TTL:       ttl,
	}
}

// IsExpired ตรวจสอบว่า session หมดอายุหรือยัง โดยเทียบกับเวลาปัจจุบันจาก Clock
func (s *Session) IsExpired() bool {
	return s.clock.Now().After(s.CreatedAt.Add(s.TTL))
}
```

Test ที่ควบคุมเวลาได้แบบ 100% แม่นยำ ไม่ต้อง `time.Sleep` เลยสักบรรทัด:

```go
func TestSession_IsExpired(t *testing.T) {
	base := time.Date(2024, 1, 1, 12, 0, 0, 0, time.UTC)

	tests := []struct {
		name    string
		now     time.Time
		ttl     time.Duration
		expired bool
	}{
		{"ยังไม่หมดอายุ", base.Add(30 * time.Second), time.Minute, false},
		{"หมดอายุพอดี", base.Add(time.Minute), time.Minute, false}, // After() ไม่รวมจุดเท่ากันเป๊ะ
		{"หมดอายุแล้ว", base.Add(2 * time.Minute), time.Minute, true},
	}

	for _, tc := range tests {
		t.Run(tc.name, func(t *testing.T) {
			// สร้าง session ด้วย FixedClock ที่คืนค่า base เสมอตอนสร้าง
			creationClock := FixedClock{FixedTime: base}
			s := NewSession(creationClock, tc.ttl)

			// สลับ clock ภายใน session เป็นเวลาที่ต้องการทดสอบ "ตอนนี้"
			s.clock = FixedClock{FixedTime: tc.now}

			if got := s.IsExpired(); got != tc.expired {
				t.Errorf("IsExpired() = %v, want %v (now=%v, createdAt=%v, ttl=%v)",
					got, tc.expired, tc.now, s.CreatedAt, tc.ttl)
			}
		})
	}
}
```

ทุก subtest ผ่านทันที ไม่มี flaky test จากความคลาดเคลื่อนของเวลาจริงเลยแม้แต่มิลลิวินาทีเดียว เพราะเวลาทั้งหมดถูกกำหนดตายตัวผ่าน `FixedClock`

**หลักการออกแบบที่ควรจำ**: เมื่อไหร่ก็ตามที่โค้ด business logic ต้องพึ่งพา "สิ่งที่ไม่ deterministic" — เวลาปัจจุบัน, ตัวเลขสุ่ม, การเรียก network, หรือ environment variable — ให้พิจารณาห่อมันไว้หลัง interface ขนาดเล็ก (ตามหลักการ consumer-defined interface จาก **Part 035**) แล้ว inject เข้าไป วิธีนี้ทำให้ test ควบคุมได้แม่นยำ รันเร็ว และไม่ flaky โดยไม่ต้องพึ่ง library เสริมใดๆ เลย

---

## สรุปสิ่งที่ได้เรียนในบทนี้

- **`TestMain`** ใช้ทำ fixture ระดับ package ที่มีต้นทุนสูง ต้องเรียก `m.Run()` เสมอและส่ง exit code ต่อให้ `os.Exit`
- **`t.TempDir()`** สร้างโฟลเดอร์ชั่วคราวที่ลบให้อัตโนมัติเมื่อ test จบ ปลอดภัยกว่า `os.MkdirTemp` + `defer os.RemoveAll` เอง
- **`t.Cleanup()`** การันตี teardown ไม่ว่า test จะจบแบบไหน เรียกจาก helper function ได้ และรอ parallel subtest ให้เสร็จก่อนเสมอ
- **`t.Parallel()`** ทำให้ subtest รันคู่ขนานได้จริง (พิสูจน์แล้วว่าลดเวลาได้หลายเท่า) แต่ parent test function จะรันผ่าน subtest ที่ parallel ไปเลยทันที ต้องใช้ `t.Cleanup` ถ้าต้องการเช็คผลลัพธ์หลังจาก subtest ทั้งหมดเสร็จจริง
- **Loop variable capture กับ parallel subtests**: Go 1.22+ แก้ปัญหาระดับ semantics ของ `for...range` ได้ แต่ `go.mod` ที่ยังระบุ `go` เวอร์ชันต่ำกว่า 1.22 จะยังคงมีบั๊กนี้อยู่แม้รันด้วย toolchain ใหม่ — พิสูจน์ได้จริงด้วยการรันโค้ดเดียวกันบนสอง `go.mod` ที่ต่างกัน การเขียน `tc := tc` ก่อน `t.Run` ยังเป็นนิสัยที่ดีที่ควรรักษาไว้
- **Fuzzing** (`func FuzzXxx(f *testing.F)`, `f.Add`, `f.Fuzz`, `go test -fuzz=...`) ให้ Go สุ่มสร้าง input จำนวนมากเพื่อหาบั๊กที่มนุษย์คิดไม่ถึง โดยพิสูจน์จริงว่าเจอบั๊ก UTF-8 ใน `Reverse` ได้ภายในเวลาไม่ถึงวินาที และบันทึก failing case ไว้เป็น regression test อัตโนมัติใน `testdata/fuzz/`
- **`t.Setenv`** ตั้งค่า environment variable แล้วคืนค่าเดิมให้อัตโนมัติ แต่ใช้ร่วมกับ `t.Parallel()` ในระดับเดียวกันไม่ได้
- **Dependency injection ของ `Clock`** แก้ปัญหาโค้ดที่พึ่งพา `time.Now()` โดยตรง ทำให้ test เวลาแม่นยำ 100% ไม่ flaky และไม่ต้อง `time.Sleep` เลย

## แบบฝึกหัดท้ายบท

1. เขียน fixture ด้วย `TestMain` สำหรับ package ที่จำลอง "การเชื่อมต่อ cache" (เช่น struct ที่มี `map[string]string` ข้างใน) โดย setup ใส่ข้อมูลเริ่มต้น 3 รายการ และ teardown พิมพ์ log ว่า "cache closed" ออกมา ทดสอบว่า test หลายตัวเห็นข้อมูลเริ่มต้นร่วมกันได้ถูกต้อง
2. เขียนฟังก์ชัน `CountLines(path string) (int, error)` ที่อ่านไฟล์แล้วนับจำนวนบรรทัด จากนั้นเขียน test โดยใช้ `t.TempDir()` สร้างไฟล์ตัวอย่างขึ้นมาทดสอบ 3 กรณี: ไฟล์มีหลายบรรทัด, ไฟล์ว่าง, ไฟล์ไม่มี newline ท้ายบรรทัดสุดท้าย
3. ทำการทดลองด้วยตัวเอง: สร้างโฟลเดอร์ใหม่พร้อม `go.mod` ที่ระบุ `go 1.20` เขียน test ที่มี `for _, tc := range cases { t.Run(...) { t.Parallel(); ... } }` โดยไม่มีการ shadow ตัวแปร แล้วสังเกตว่าเกิดบั๊ก loop variable capture หรือไม่ จากนั้นลองเปลี่ยน `go.mod` เป็น `go 1.24` แล้วรันโค้ดเดิมอีกครั้งโดยไม่แก้อะไรเลย เปรียบเทียบผลลัพธ์
4. เขียนฟังก์ชัน `ParseCSVLine(line string) ([]string, error)` ง่ายๆ ที่แยก string ด้วยเครื่องหมายจุลภาค (ไม่ต้องรองรับ quoted field ก็ได้) แล้วเขียน `FuzzParseCSVLine` ที่ตรวจสอบว่าฟังก์ชันไม่ panic ไม่ว่า input จะเป็นอะไร รันด้วย `go test -fuzz` อย่างน้อย 15 วินาที แล้วรายงานว่าเจอ input ที่ทำให้ panic หรือไม่ (ถ้าเจอ ให้แก้บั๊กแล้วรันซ้ำเพื่อยืนยัน)
5. ออกแบบ interface `IDGenerator` ที่มี method `NewID() string` แล้วเขียน implementation จริง (สุ่มด้วย UUID หรือ timestamp) และ implementation ปลอมสำหรับ test ที่คืนค่าคงที่ตามลำดับที่กำหนดไว้ล่วงหน้า (เช่น "id-1", "id-2", "id-3", ...) ใช้ทดสอบฟังก์ชันที่สร้าง object ใหม่หลายตัวติดกัน
6. เขียน test ที่ใช้ทั้ง `t.Setenv` และ `t.Parallel()` ในฟังก์ชันเดียวกัน (ตั้งใจทำผิด) แล้วสังเกต error message ที่ Go พิมพ์ออกมาตอนรัน อธิบายด้วยคำพูดตัวเองว่าทำไม Go ถึงห้ามใช้สองอย่างนี้ร่วมกัน

---

**ต่อไป**: [Part 080 — Integration Testing](./080-integration-testing.md)
