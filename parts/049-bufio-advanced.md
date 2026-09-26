# Part 049: `bufio` ขั้นสูง

> ภาคที่ 4: Standard Library เชิงลึก — ตอนที่ 4 จาก 10 (Part 46–55)

## สารบัญของบทนี้

1. ทบทวน `bufio.Scanner` และ `SplitFunc`
2. เขียน `SplitFunc` เอง: แบ่งข้อมูลตามกฎที่กำหนดเอง
3. ข้อจำกัดขนาด Token ของ Scanner และการแก้ด้วย `Buffer()`
4. `bufio.Reader`: `ReadString`, `ReadBytes`, `ReadLine`, `Peek` — เมื่อไรควรใช้อันไหน
5. `bufio.Writer` ภายในทำงานอย่างไร และทำไม `Flush()` ถึงสำคัญ
6. ห่อ `os.Stdin`/`os.Stdout` เพื่อ CLI I/O ที่มีประสิทธิภาพ
7. สรุปสิ่งที่ได้เรียนในบทนี้
8. แบบฝึกหัดท้ายบท

---

## 1. ทบทวน `bufio.Scanner` และ `SplitFunc`

**Part 024** แนะนำ `bufio.Scanner` ในฐานะเครื่องมืออ่านไฟล์ทีละบรรทัดที่สะดวกที่สุด ด้วย loop รูปแบบ `for scanner.Scan() { ... }` และเช็ค `scanner.Err()` หลังจบ loop เสมอ — บทนี้จะพาไปดูว่า **"ทีละบรรทัด" เป็นแค่พฤติกรรม default เท่านั้น** เบื้องหลังจริงๆ `Scanner` ทำงานผ่านฟังก์ชันที่เรียกว่า **`bufio.SplitFunc`** ซึ่งกำหนดว่า "อะไรคือ token หนึ่งชิ้น" — และเราสามารถเปลี่ยนมันได้อิสระ

signature ของ `SplitFunc`:

```go
type SplitFunc func(data []byte, atEOF bool) (advance int, token []byte, err error)
```

- **`data`** — ข้อมูลที่ยังไม่ถูกประมวลผลที่มีอยู่ใน buffer ณ ขณะนั้น
- **`atEOF`** — `true` ถ้าไม่มีข้อมูลเหลือให้อ่านเพิ่มอีกแล้ว (ถึงจุดจบ stream)
- **`advance`** — จำนวน byte ที่ให้ `Scanner` เลื่อน cursor ไปข้างหน้า (คือจำนวนที่ "ใช้ไปแล้ว")
- **`token`** — ข้อมูลที่ถือว่าเป็น token หนึ่งชิ้นสมบูรณ์ ที่จะให้ `scanner.Text()`/`scanner.Bytes()` คืนกลับไป
- **`err`** — error ถ้ามี (ปกติคืน `nil`)

`bufio` package มี `SplitFunc` สำเร็จรูปให้ใช้ 4 แบบ: `bufio.ScanLines` (default — แบ่งทีละบรรทัด), `bufio.ScanWords` (แบ่งทีละคำ), `bufio.ScanRunes` (แบ่งทีละ rune รองรับ UTF-8), และ `bufio.ScanBytes` (แบ่งทีละ byte ดิบ)

---

## 2. เขียน `SplitFunc` เอง: แบ่งข้อมูลตามกฎที่กำหนดเอง

เมื่อกฎการแบ่งข้อมูลทั้ง 4 แบบสำเร็จรูปไม่ตรงกับความต้องการ (เช่น ต้องการแบ่งด้วย delimiter พิเศษ) เราเขียน `SplitFunc` เองได้โดยตรง:

```go
package main

import (
	"bufio"
	"fmt"
	"strings"
)

// commaSplitFunc คือ bufio.SplitFunc เอง แบ่ง token ด้วยเครื่องหมายจุลภาค (,) แทนที่จะแบ่งด้วย
// บรรทัดหรือคำแบบมาตรฐาน - เขียนตาม signature ที่ bufio.SplitFunc กำหนดไว้ตรงๆ
//
// SplitFunc รับ (data []byte, atEOF bool) และต้องคืนค่า 3 อย่างเสมอ:
//   - advance: จำนวน byte ที่ "กินไป" แล้วในรอบนี้ (ให้ Scanner เลื่อน cursor ไปเท่านี้)
//   - token:   ข้อมูลที่ถือว่าเป็น "token หนึ่งชิ้น" ที่จะคืนให้ผู้เรียกผ่าน scanner.Text()/Bytes()
//   - err:     error ถ้ามี (ปกติคืน nil)
func commaSplitFunc(data []byte, atEOF bool) (advance int, token []byte, err error) {
	if atEOF && len(data) == 0 {
		return 0, nil, nil // ไม่มีข้อมูลเหลือแล้วและถึงจุดจบ - บอก Scanner ว่าจบการทำงาน
	}
	if i := indexByte(data, ','); i >= 0 {
		// เจอ comma ที่ตำแหน่ง i แล้ว - คืน token ที่ตัดก่อนหน้า comma (ไม่รวม comma เอง)
		// พร้อม advance ไปอีก i+1 byte (ข้าม comma ไปด้วย) เพื่อเตรียมหา token ถัดไป
		return i + 1, data[:i], nil
	}
	if atEOF {
		// ถึงจุดจบไฟล์แล้วแต่ไม่มี comma เหลือ - คืนข้อมูลที่เหลือทั้งหมดเป็น token สุดท้าย
		return len(data), data, nil
	}
	// ยังหา comma ไม่เจอ และยังไม่ถึง EOF - ขอข้อมูลเพิ่มเติมก่อน (คืน advance เป็น 0)
	return 0, nil, nil
}

func indexByte(data []byte, b byte) int {
	for i, c := range data {
		if c == b {
			return i
		}
	}
	return -1
}

func main() {
	fmt.Println("=== ตัวอย่างที่ 1: bufio.ScanWords (มีให้ใน standard library แล้ว) ===")
	scanner1 := bufio.NewScanner(strings.NewReader("Go เป็นภาษาที่เรียบง่าย และรวดเร็ว"))
	scanner1.Split(bufio.ScanWords) // เปลี่ยนจาก default (ScanLines) เป็นแบ่งทีละคำ
	for scanner1.Scan() {
		fmt.Println("คำ:", scanner1.Text())
	}

	fmt.Println()
	fmt.Println("=== ตัวอย่างที่ 2: SplitFunc ที่เขียนเอง (แบ่งด้วย comma) ===")
	scanner2 := bufio.NewScanner(strings.NewReader("แอปเปิ้ล,กล้วย,ส้ม,มะม่วง"))
	scanner2.Split(commaSplitFunc)
	for scanner2.Scan() {
		fmt.Println("รายการ:", scanner2.Text())
	}

	fmt.Println()
	fmt.Println("=== ตัวอย่างที่ 3: bufio.ScanRunes (แบ่งทีละ rune รองรับ UTF-8) ===")
	scanner3 := bufio.NewScanner(strings.NewReader("Go๓"))
	scanner3.Split(bufio.ScanRunes)
	count := 0
	for scanner3.Scan() {
		count++
		fmt.Printf("rune ที่ %d: %q\n", count, scanner3.Text())
	}
}
```

ผลลัพธ์:

```
=== ตัวอย่างที่ 1: bufio.ScanWords (มีให้ใน standard library แล้ว) ===
คำ: Go
คำ: เป็นภาษาที่เรียบง่าย
คำ: และรวดเร็ว

=== ตัวอย่างที่ 2: SplitFunc ที่เขียนเอง (แบ่งด้วย comma) ===
รายการ: แอปเปิ้ล
รายการ: กล้วย
รายการ: ส้ม
รายการ: มะม่วง

=== ตัวอย่างที่ 3: bufio.ScanRunes (แบ่งทีละ rune รองรับ UTF-8) ===
rune ที่ 1: "G"
rune ที่ 2: "o"
rune ที่ 3: "๓"
```

### ข้อควรระวังในการเขียน `SplitFunc` เอง

1. **ต้องจัดการกรณี `atEOF == true` เสมอ** — ถ้าลืมกรณีนี้ token สุดท้ายที่ไม่มี delimiter ต่อท้าย (เช่น ตัวอย่างข้างบนถ้าไม่มี comma ปิดท้าย) จะหายไปเงียบๆ
2. **อย่าคืน `advance` เป็น `0` พร้อมกับ `token` ที่ไม่ใช่ `nil`** — จะทำให้ `Scanner` วนลูปไม่รู้จบ (คืน token เดิมซ้ำตลอดไปโดยไม่ขยับ cursor เลย)
3. **`data` อาจไม่ใช่ข้อมูลทั้งหมด** — ถ้ายังหา delimiter ไม่เจอและยังไม่ถึง `atEOF` ให้คืน `(0, nil, nil)` เพื่อบอก `Scanner` ว่า "ขอข้อมูลเพิ่มก่อน" ห้ามสรุปว่า "ไม่มี delimiter แน่ๆ" ตั้งแต่ข้อมูลยังมาไม่ครบ (นี่คือสาเหตุที่ `commaSplitFunc` เช็ค `atEOF` ก่อนตัดสินใจคืน token ที่เหลือทั้งหมด)

---

## 3. ข้อจำกัดขนาด Token ของ Scanner และการแก้ด้วย `Buffer()`

`bufio.Scanner` มีข้อจำกัดที่สำคัญมากและมักทำให้โปรแกรมพังกลางทางแบบไม่คาดคิด: **ขนาด buffer สูงสุด default คือ `bufio.MaxScanTokenSize` = 64KB (65,536 byte)** ถ้า token หนึ่งชิ้น (เช่น บรรทัดหนึ่งบรรทัดตอนใช้ `ScanLines`) มีขนาดใหญ่กว่านี้ `Scanner.Scan()` จะคืน `false` ทันทีและ `Scanner.Err()` จะรายงาน error ว่า **`bufio.Scanner: token too long`**

สถานการณ์นี้พบได้บ่อยกว่าที่คิดในโลกจริง เช่น อ่าน log file ที่มี stack trace ยาวมากอัดอยู่ในบรรทัดเดียว, อ่านไฟล์ CSV ที่มี field ขนาดใหญ่ (เช่น เก็บ base64 ของรูปภาพ), หรือ JSON Lines ที่แต่ละบรรทัดเป็น object ขนาดใหญ่

มาดูปัญหานี้แบบจับต้องได้ แล้วแก้ไขด้วย `scanner.Buffer(...)`:

```go
package main

import (
	"bufio"
	"fmt"
	"strings"
)

func main() {
	// bufio.MaxScanTokenSize คือขนาด buffer สูงสุด default ของ Scanner = 64 * 1024 = 65536 byte
	// ถ้า token เดียว (เช่น หนึ่งบรรทัด) มีขนาดใหญ่กว่านี้ Scanner จะ error ทันที
	fmt.Println("bufio.MaxScanTokenSize (ค่า default) =", bufio.MaxScanTokenSize, "bytes")

	// จำลองบรรทัดที่ยาวเกิน 64KB (เช่น log line ที่มี stack trace ยาวมาก หรือ CSV field ขนาดใหญ่)
	hugeLine := strings.Repeat("x", 100_000)
	input := "บรรทัดแรกปกติ\n" + hugeLine + "\nบรรทัดสุดท้ายปกติ\n"

	fmt.Println()
	fmt.Println("=== ก่อนแก้: ใช้ Scanner แบบ default buffer ===")
	scanner := bufio.NewScanner(strings.NewReader(input))
	lineNo := 0
	for scanner.Scan() {
		lineNo++
		fmt.Printf("บรรทัด %d: ความยาว %d ตัวอักษร\n", lineNo, len(scanner.Text()))
	}
	if err := scanner.Err(); err != nil {
		fmt.Println("เกิด error:", err)
	}

	fmt.Println()
	fmt.Println("=== หลังแก้: ขยาย buffer ด้วย scanner.Buffer() ===")
	scanner2 := bufio.NewScanner(strings.NewReader(input))
	// scanner.Buffer(buf []byte, max int) กำหนด buffer เริ่มต้นและขนาดสูงสุดที่ยอมให้ขยายไปถึง
	// ในที่นี้ให้ buffer เริ่มต้นขนาด 4096 byte แต่ขยายได้สูงสุดถึง 1MB ถ้าจำเป็น
	buf := make([]byte, 4096)
	scanner2.Buffer(buf, 1024*1024)

	lineNo = 0
	for scanner2.Scan() {
		lineNo++
		fmt.Printf("บรรทัด %d: ความยาว %d ตัวอักษร\n", lineNo, len(scanner2.Text()))
	}
	if err := scanner2.Err(); err != nil {
		fmt.Println("เกิด error:", err)
	} else {
		fmt.Println("อ่านสำเร็จครบทุกบรรทัดโดยไม่มี error")
	}
}
```

ผลลัพธ์:

```
bufio.MaxScanTokenSize (ค่า default) = 65536 bytes

=== ก่อนแก้: ใช้ Scanner แบบ default buffer ===
บรรทัด 1: ความยาว 39 ตัวอักษร
เกิด error: bufio.Scanner: token too long

=== หลังแก้: ขยาย buffer ด้วย scanner.Buffer() ===
บรรทัด 1: ความยาว 39 ตัวอักษร
บรรทัด 2: ความยาว 100000 ตัวอักษร
บรรทัด 3: ความยาว 51 ตัวอักษร
อ่านสำเร็จครบทุกบรรทัดโดยไม่มี error
```

### เจาะลึก `scanner.Buffer(buf []byte, max int)`

- **`buf`** — buffer เริ่มต้นที่ `Scanner` จะใช้ (ถ้าไม่พอจะขยายเองอัตโนมัติ) ใส่ `nil` ได้ถ้าต้องการให้ `Scanner` จัดสรรเอง
- **`max`** — ขนาดสูงสุดที่ยอม**ขยาย**ไปได้ ถ้า token ยังใหญ่กว่านี้อีกถึงจะ error จริง
- **ต้องเรียกก่อน `scanner.Scan()` ครั้งแรกเสมอ** — เรียกหลังจากเริ่ม scan ไปแล้วจะไม่มีผลใดๆ (หรือ panic ในบาง Go version)

**คำแนะนำในทางปฏิบัติ**: ถ้าโปรแกรมต้องประมวลผลข้อมูลจากแหล่งที่ควบคุมขนาดบรรทัดไม่ได้ (เช่น รับ log จากระบบภายนอก, ประมวลผลไฟล์ที่ผู้ใช้อัปโหลดเอง) ควรตั้ง `scanner.Buffer(...)` ด้วยค่า `max` ที่เหมาะสมไว้เสมอตั้งแต่แรก แทนที่จะรอให้เจอ error `token too long` ใน production ก่อนแล้วค่อยแก้ — แต่ก็ต้องระวังไม่ตั้งค่าสูงเกินไปแบบไม่มีขีดจำกัด (เช่น `math.MaxInt`) เพราะจะเปิดช่องให้ข้อมูลที่ประสงค์ร้ายส่งบรรทัดเดียวขนาดมหาศาลมาทำให้โปรแกรมใช้ memory จนล่มได้ (แนวคิดเดียวกับ `io.LimitReader` ที่เรียนใน **Part 048**)

---

## 4. `bufio.Reader`: `ReadString`, `ReadBytes`, `ReadLine`, `Peek` — เมื่อไรควรใช้อันไหน

`bufio.Scanner` สะดวกแต่มีข้อจำกัดหลายอย่าง (ต้องเช็ค token size, ควบคุมยากเมื่อ delimiter มีความซับซ้อน) `bufio.Reader` เป็นเครื่องมือที่ **low-level กว่าแต่ยืดหยุ่นกว่า**:

```go
package main

import (
	"bufio"
	"fmt"
	"io"
	"strings"
)

func main() {
	fmt.Println("=== ReadString: อ่านจนเจอ delimiter ที่ระบุ (รวม delimiter ไว้ในผลลัพธ์ด้วย) ===")
	r1 := bufio.NewReader(strings.NewReader("ชื่อ:สมชาย,อายุ:30,เมือง:กรุงเทพ,"))
	for {
		part, err := r1.ReadString(',')
		if len(part) > 0 {
			fmt.Printf("อ่านได้: %q\n", part)
		}
		if err != nil {
			if err == io.EOF {
				fmt.Println("จบข้อมูลแล้ว (EOF)")
			}
			break
		}
	}

	fmt.Println()
	fmt.Println("=== ReadBytes: เหมือน ReadString แต่คืน []byte แทน string (เลี่ยงการแปลงชนิดถ้าไม่จำเป็น) ===")
	r2 := bufio.NewReader(strings.NewReader("a|b|c"))
	for {
		b, err := r2.ReadBytes('|')
		if len(b) > 0 {
			fmt.Printf("bytes: %v -> %q\n", b, b)
		}
		if err != nil {
			break
		}
	}

	fmt.Println()
	fmt.Println("=== Peek: แอบดูข้อมูลข้างหน้าโดยไม่เลื่อน cursor (อ่านซ้ำได้) ===")
	r3 := bufio.NewReader(strings.NewReader("GIF89a รูปภาพขนาดเล็ก"))
	magic, err := r3.Peek(6) // แอบดู 6 byte แรกโดยไม่ "กิน" ข้อมูลจริง
	if err != nil {
		fmt.Println("peek error:", err)
	} else {
		fmt.Printf("Peek เห็น magic bytes: %q (cursor ยังไม่ขยับ)\n", magic)
	}
	// อ่านจริงหลัง Peek - จะเริ่มจากตำแหน่งเดิม ไม่ใช่ต่อจากที่ Peek ไปแล้ว
	full, _ := io.ReadAll(r3)
	fmt.Printf("อ่านทั้งหมดหลัง Peek: %q (ยืนยันว่า Peek ไม่ทำให้ข้อมูลหายไป)\n", full)

	fmt.Println()
	fmt.Println("=== ReadLine: คืน []byte แบบ low-level ไม่รวม newline (แนะนำให้ใช้ Scanner แทนในกรณีทั่วไป) ===")
	r4 := bufio.NewReader(strings.NewReader("บรรทัดที่ 1\nบรรทัดที่ 2\n"))
	for {
		line, isPrefix, err := r4.ReadLine()
		if err != nil {
			break
		}
		fmt.Printf("บรรทัด: %q (isPrefix=%v ถ้า true แปลว่าบรรทัดยาวเกิน buffer ต้องอ่านต่อ)\n", line, isPrefix)
	}
}
```

ผลลัพธ์:

```
=== ReadString: อ่านจนเจอ delimiter ที่ระบุ (รวม delimiter ไว้ในผลลัพธ์ด้วย) ===
อ่านได้: "ชื่อ:สมชาย,"
อ่านได้: "อายุ:30,"
อ่านได้: "เมือง:กรุงเทพ,"
จบข้อมูลแล้ว (EOF)

=== ReadBytes: เหมือน ReadString แต่คืน []byte แทน string (เลี่ยงการแปลงชนิดถ้าไม่จำเป็น) ===
bytes: [97 124] -> "a|"
bytes: [98 124] -> "b|"
bytes: [99] -> "c"

=== Peek: แอบดูข้อมูลข้างหน้าโดยไม่เลื่อน cursor (อ่านซ้ำได้) ===
Peek เห็น magic bytes: "GIF89a" (cursor ยังไม่ขยับ)
อ่านทั้งหมดหลัง Peek: "GIF89a รูปภาพขนาดเล็ก" (ยืนยันว่า Peek ไม่ทำให้ข้อมูลหายไป)

=== ReadLine: คืน []byte แบบ low-level ไม่รวม newline (แนะนำให้ใช้ Scanner แทนในกรณีทั่วไป) ===
บรรทัด: "บรรทัดที่ 1" (isPrefix=false ถ้า true แปลว่าบรรทัดยาวเกิน buffer ต้องอ่านต่อ)
บรรทัด: "บรรทัดที่ 2" (isPrefix=false ถ้า true แปลว่าบรรทัดยาวเกิน buffer ต้องอ่านต่อ)
```

### ตารางสรุป: เลือกใช้ `Scanner` หรือ `Reader` เมื่อไร

| ต้องการ | ใช้ |
|---|---|
| อ่านทีละบรรทัด/ทีละคำแบบทั่วไป ไม่ต้องควบคุมละเอียด | `bufio.Scanner` (`Scan()`/`Text()`) |
| ต้องการ delimiter ที่ซับซ้อนกว่า `SplitFunc` รองรับสะดวก หรือ delimiter เปลี่ยนไปเรื่อยๆ ระหว่างอ่าน | `bufio.Reader.ReadString`/`ReadBytes` |
| ต้องการ "แอบดู" ข้อมูลข้างหน้าก่อนตัดสินใจว่าจะอ่านต่อยังไง (เช่น ตรวจสอบ magic byte ของไฟล์, protocol parsing) | `bufio.Reader.Peek` |
| ไม่อยากกังวลเรื่อง token size limit เลย (อยากควบคุม memory เอง) | `bufio.Reader` (ไม่มีข้อจำกัดขนาดแบบ `Scanner`) |
| ต้องการโค้ดที่อ่านง่ายที่สุดสำหรับกรณีอ่านทีละบรรทัดปกติ | `bufio.Scanner` ยังคงเป็นตัวเลือกที่ดีที่สุด |

เช่นเดียวกับ `bufio.NewWriterSize` ที่จะเจอในหัวข้อถัดไป `bufio.Reader` ก็มี `bufio.NewReaderSize(r io.Reader, size int)` ให้กำหนดขนาด buffer ภายในเองได้ (ค่า default ก็คือ 4096 byte เท่ากัน) — เพิ่มขนาด buffer ให้ใหญ่ขึ้นช่วยลดจำนวนครั้งที่ต้องอ่านข้อมูลเพิ่มจากต้นทางจริง (เช่น จากดิสก์หรือเครือข่าย) เหมาะกับกรณีที่รู้ล่วงหน้าว่าข้อมูลมีปริมาณมากและจะอ่านต่อเนื่องกันไปเรื่อยๆ

**ข้อสังเกตสำคัญ**: `bufio.Reader` **ไม่มีข้อจำกัดเรื่องขนาด token แบบ `Scanner`** เพราะมันคืนผลลัพธ์ทันทีที่เจอ delimiter (ต่อให้ผลลัพธ์นั้นมีขนาดใหญ่แค่ไหนก็ไม่ error) แต่ก็แปลว่า**ผู้เขียนโค้ดต้องระวังเรื่อง memory เองแทน** — ถ้าข้อมูลระหว่างสอง delimiter มีขนาดใหญ่มาก `ReadString`/`ReadBytes` จะพยายามอ่านมันทั้งหมดเข้า memory โดยไม่มีการเตือนใดๆ นี่คือ trade-off ที่ตรงข้ามกับปัญหาที่ `Scanner` มีในหัวข้อ 3 พอดี

---

## 5. `bufio.Writer` ภายในทำงานอย่างไร และทำไม `Flush()` ถึงสำคัญ

**Part 024 หัวข้อ 6** สอนไปแล้วว่าต้องเรียก `writer.Flush()` เสมอ ไม่งั้นข้อมูลอาจหายไปแบบเงียบๆ — บทนี้จะพิสูจน์ **"ทำไม" ด้วยตัวเลขจริง** ว่า `bufio.Writer` ประหยัดการเขียนจริงไปได้มากแค่ไหน:

```go
package main

import (
	"bufio"
	"fmt"
	"io"
)

// countingWriter ห่อ io.Writer แล้วนับว่า Write() ถูกเรียกจริงกี่ครั้ง (แทนการนับ system call
// จริงๆ ซึ่งวัดยากกว่า - หลักการเดียวกับ HashingReader/CountingReader ที่เรียนใน Part 048/030)
type countingWriter struct {
	w          io.Writer
	writeCalls int
}

func (cw *countingWriter) Write(p []byte) (int, error) {
	cw.writeCalls++
	return cw.w.Write(p)
}

func main() {
	fmt.Println("=== ไม่ใช้ bufio.Writer: เขียนตรงๆ ทุกครั้ง ===")
	cw1 := &countingWriter{w: io.Discard}
	for i := 0; i < 1000; i++ {
		fmt.Fprintf(cw1, "บรรทัดที่ %d\n", i)
	}
	fmt.Println("จำนวนครั้งที่ Write() ถูกเรียกจริง:", cw1.writeCalls, "ครั้ง (เท่ากับจำนวนลูปพอดี)")

	fmt.Println()
	fmt.Println("=== ใช้ bufio.Writer ห่อไว้: ข้อมูลถูกสะสมใน buffer ก่อน ===")
	cw2 := &countingWriter{w: io.Discard}
	bw := bufio.NewWriter(cw2)
	for i := 0; i < 1000; i++ {
		fmt.Fprintf(bw, "บรรทัดที่ %d\n", i)
	}
	fmt.Println("ก่อน Flush: จำนวนครั้งที่ Write() ถูกเรียกจริงบน underlying writer:", cw2.writeCalls, "ครั้ง")
	bw.Flush()
	fmt.Println("หลัง Flush: จำนวนครั้งที่ Write() ถูกเรียกจริงบน underlying writer:", cw2.writeCalls, "ครั้ง")
	fmt.Println("(น้อยกว่า 1000 ครั้งมาก เพราะ bufio.Writer รวมข้อมูลหลายๆ ครั้งเป็น buffer เดียว")
	fmt.Println(" แล้วค่อยเรียก Write() จริงตอน buffer เต็มหรือตอน Flush() เท่านั้น)")

	fmt.Println()
	fmt.Println("=== ขนาด buffer เริ่มต้นและการปรับแต่ง ===")
	fmt.Println("bufio.NewWriter(w) ใช้ buffer ขนาด default 4096 byte")
	bwCustom := bufio.NewWriterSize(cw2, 64*1024) // ปรับขนาด buffer เองด้วย NewWriterSize
	fmt.Println("bufio.NewWriterSize(w, 64*1024) กำหนด buffer เองได้ ขนาดปัจจุบัน:", bwCustom.Available()+bwCustom.Buffered(), "byte")
}
```

ผลลัพธ์:

```
=== ไม่ใช้ bufio.Writer: เขียนตรงๆ ทุกครั้ง ===
จำนวนครั้งที่ Write() ถูกเรียกจริง: 1000 ครั้ง (เท่ากับจำนวนลูปพอดี)

=== ใช้ bufio.Writer ห่อไว้: ข้อมูลถูกสะสมใน buffer ก่อน ===
ก่อน Flush: จำนวนครั้งที่ Write() ถูกเรียกจริงบน underlying writer: 7 ครั้ง
หลัง Flush: จำนวนครั้งที่ Write() ถูกเรียกจริงบน underlying writer: 8 ครั้ง
(น้อยกว่า 1000 ครั้งมาก เพราะ bufio.Writer รวมข้อมูลหลายๆ ครั้งเป็น buffer เดียว
 แล้วค่อยเรียก Write() จริงตอน buffer เต็มหรือตอน Flush() เท่านั้น)

=== ขนาด buffer เริ่มต้นและการปรับแต่ง ===
bufio.NewWriter(w) ใช้ buffer ขนาด default 4096 byte
bufio.NewWriterSize(w, 64*1024) กำหนด buffer เองได้ ขนาดปัจจุบัน: 65536 byte
```

ตัวเลขนี้พูดได้ชัดกว่าคำอธิบายใดๆ: การเขียน 1000 บรรทัดแบบตรงๆ คือการเรียก `Write()` จริง **1000 ครั้ง** แต่พอห่อด้วย `bufio.Writer` เหลือแค่ **7 ครั้งก่อน `Flush()`** (buffer เต็มพอดี 7 รอบจาก 4096 byte) และเพิ่มมาอีกแค่ **1 ครั้งตอน `Flush()`** เพื่อระบายข้อมูลที่เหลือค้าง buffer อยู่ทั้งหมดออกไป

### ทำไมเรื่องนี้สำคัญในทางปฏิบัติ

ทุกครั้งที่ `Write()` ถูกเรียกบน `*os.File` จริง คือการทำ **system call** ไปยังระบบปฏิบัติการ ซึ่งมีค่าใช้จ่ายสูงกว่าการทำงานใน user space ของโปรแกรมมาก (ต้องสลับ context ไปทำงานใน kernel แล้วสลับกลับ) การลดจำนวน system call จาก 1000 เหลือ 8 ครั้ง (ในตัวอย่างข้างต้น) จึงเป็นการเพิ่มประสิทธิภาพที่มีนัยสำคัญจริงในโปรแกรมที่เขียนข้อมูลจำนวนมาก เช่น export CSV ขนาดใหญ่, เขียน log จำนวนมหาศาล, หรือสร้างไฟล์ output ของโปรแกรมประมวลผลข้อมูล

**และนี่คือเหตุผลเดียวกันที่อธิบายว่าทำไมลืม `Flush()` ถึงทำให้ข้อมูลหายไปแบบเงียบๆ**: ถ้าโปรแกรมจบการทำงานก่อนที่ buffer จะถูกระบายออกไป (buffer ยังไม่เต็มพอดีและไม่มีใครเรียก `Flush()`) ข้อมูลที่ "ค้าง" อยู่ใน memory ของ `bufio.Writer` จะหายไปพร้อมกับโปรแกรมทันที เพราะมันไม่เคยถูกเขียนลง `os.File` จริงเลยสักครั้งเดียว

---

## 6. ห่อ `os.Stdin`/`os.Stdout` เพื่อ CLI I/O ที่มีประสิทธิภาพ

`os.Stdin` และ `os.Stdout` เป็น `*os.File` ธรรมดา (เชื่อมกับ standard input/output ของ process) — เหตุผลเดียวกับหัวข้อก่อนหน้าทำให้การอ่าน/เขียนผ่านมันตรงๆ ในโปรแกรม CLI ที่มี I/O จำนวนมากไม่มีประสิทธิภาพเท่าที่ควร รูปแบบมาตรฐานของ CLI tool ที่ดีคือห่อทั้งคู่ด้วย `bufio`:

```go
package main

import (
	"bufio"
	"fmt"
	"os"
	"strings"
)

func main() {
	// ห่อ os.Stdin ด้วย bufio.NewReader และ os.Stdout ด้วย bufio.NewWriter
	// เป็น pattern มาตรฐานสำหรับ CLI tool ที่ต้องอ่าน/เขียนจำนวนมาก เพราะ os.Stdin/os.Stdout
	// เป็น *os.File ธรรมดาที่ไม่มี buffer ในตัว - ทุกครั้งที่ Read/Write ตรงๆ คือ system call จริง
	reader := bufio.NewReader(os.Stdin)
	writer := bufio.NewWriter(os.Stdout)
	defer writer.Flush() // อย่าลืม! ไม่งั้นข้อความค้างใน buffer อาจไม่ถูกพิมพ์ออกมาก่อนโปรแกรมจบ

	fmt.Fprintln(writer, "=== โปรแกรมทักทายและนับจำนวนบรรทัด ===")
	writer.Flush() // flush ทันทีสำหรับข้อความที่ต้องการให้ผู้ใช้เห็นก่อนพิมพ์ input ต่อไป

	lineCount := 0
	for {
		line, err := reader.ReadString('\n')
		trimmed := strings.TrimRight(line, "\n")
		if trimmed != "" {
			lineCount++
			fmt.Fprintf(writer, "บรรทัดที่ %d: คุณพิมพ์ว่า %q\n", lineCount, trimmed)
		}
		if err != nil {
			break // ปกติคือ io.EOF เมื่อ stdin ถูกปิด (เช่น กด Ctrl+D หรือ input หมด)
		}
	}

	fmt.Fprintf(writer, "จบโปรแกรม อ่านไปทั้งหมด %d บรรทัด\n", lineCount)
	// ไม่ต้องเรียก writer.Flush() ตรงนี้ซ้ำ เพราะมี defer ไว้แล้วด้านบน
}
```

ทดสอบโดย pipe ข้อความเข้าไปแทนการพิมพ์สด (จำลอง input จากไฟล์หรือโปรแกรมอื่น):

```bash
printf "สวัสดี\nGo คือภาษาที่ดี\n\nบรรทัดสุดท้าย\n" | go run main.go
```

ผลลัพธ์:

```
=== โปรแกรมทักทายและนับจำนวนบรรทัด ===
บรรทัดที่ 1: คุณพิมพ์ว่า "สวัสดี"
บรรทัดที่ 2: คุณพิมพ์ว่า "Go คือภาษาที่ดี"
บรรทัดที่ 3: คุณพิมพ์ว่า "บรรทัดสุดท้าย"
จบโปรแกรม อ่านไปทั้งหมด 3 บรรทัด
```

### ประเด็นสำคัญของ pattern นี้

- **`defer writer.Flush()` ที่ต้นฟังก์ชัน `main`** — วางไว้ทันทีหลังสร้าง `writer` เพื่อรับประกันว่าไม่ว่าโปรแกรมจะจบทางไหน (return ปกติ, `break` ออกจาก loop, หรือแม้แต่ `panic` ที่ recover ไว้) ข้อมูลที่ค้างอยู่ใน buffer จะถูกระบายออกมาเสมอ
- **`writer.Flush()` เพิ่มเติมกลางโปรแกรม** จำเป็นเมื่อต้องการให้ผู้ใช้ **เห็นข้อความทันที** ก่อนที่โปรแกรมจะไปรอ input ถัดไป (เช่น ข้อความ prompt "กรุณาป้อนชื่อ:") เพราะถ้าไม่ flush ข้อความอาจยังค้างอยู่ใน buffer ไม่ถูกแสดงบนหน้าจอจนกว่าจะมีการ flush ครั้งถัดไปหรือโปรแกรมจบ ทำให้ผู้ใช้เห็นหน้าจอว่างเปล่าทั้งที่โปรแกรมกำลังรอ input อยู่แล้ว
- **`os.Stdin` เป็น `io.Reader` ธรรมดา** ทำให้ทุกเทคนิคที่เรียนมาตลอดบทนี้ (Scanner, SplitFunc, `bufio.Reader` methods) ใช้กับมันได้เหมือนกับแหล่งข้อมูลอื่นๆ ทุกประการ — นี่คือผลจากการที่ Go ออกแบบทุกอย่างบนนามธรรมเดียวกันตามที่เรียนใน **Part 048**

---

## สรุปสิ่งที่ได้เรียนในบทนี้

- `bufio.Scanner` ทำงานผ่าน `SplitFunc` ที่กำหนดว่า "อะไรคือ token หนึ่งชิ้น" — เปลี่ยนพฤติกรรมได้ด้วย `scanner.Split(...)` ใช้ตัวสำเร็จรูป (`ScanWords`, `ScanRunes`, `ScanBytes`) หรือเขียน `SplitFunc` เองก็ได้ตาม signature `func(data []byte, atEOF bool) (advance int, token []byte, err error)`
- `Scanner` มีข้อจำกัดขนาด token สูงสุด default 64KB (`bufio.MaxScanTokenSize`) — ถ้าเจอ error `token too long` ให้แก้ด้วย `scanner.Buffer(buf, max)` ก่อนเริ่ม `Scan()` ครั้งแรก
- `bufio.Reader` (`ReadString`, `ReadBytes`, `ReadLine`, `Peek`) ยืดหยุ่นกว่า `Scanner` และไม่มีข้อจำกัดขนาด token แต่ต้องระวังเรื่อง memory เอง — `Peek` มีประโยชน์มากสำหรับ "แอบดู" ข้อมูลก่อนตัดสินใจโดยไม่เลื่อน cursor
- `bufio.Writer` สะสมข้อมูลใน buffer (default 4096 byte) ก่อนเขียนจริงเป็นก้อนใหญ่ ลดจำนวน system call ได้มาก (พิสูจน์แล้วว่าลดจาก 1000 เหลือ ~8 ครั้งในตัวอย่าง) — **ต้องเรียก `Flush()` เสมอ** ไม่งั้นข้อมูลที่ค้าง buffer จะหายไปแบบเงียบๆ เมื่อโปรแกรมจบ
- ห่อ `os.Stdin`/`os.Stdout` ด้วย `bufio.NewReader`/`bufio.NewWriter` เป็น pattern มาตรฐานของ CLI tool ที่มี I/O จำนวนมาก — ใช้ `defer writer.Flush()` เป็นตาข่ายนิรภัย และ `Flush()` เพิ่มเติมเมื่อต้องการให้ผู้ใช้เห็นข้อความทันที

## แบบฝึกหัดท้ายบท

1. เขียน `SplitFunc` เองที่แบ่งข้อความตามเครื่องหมายวรรคตอน (`.`, `!`, `?`) เพื่อแบ่งข้อความยาวๆ ออกเป็น "ประโยค" แทนที่จะเป็นบรรทัดหรือคำ
2. ทดลองตั้ง `scanner.Buffer(nil, 10)` (จำกัด max ให้เล็กมากแค่ 10 byte) แล้วอ่านไฟล์ที่มีบรรทัดยาวกว่านั้น สังเกต error message ที่ได้ และอธิบายว่าทำไมค่า `max` ที่เล็กเกินไปถึงเป็นปัญหาในโลกจริง
3. เขียนโปรแกรมที่ใช้ `bufio.Reader.Peek` ตรวจสอบ "magic bytes" ของไฟล์ (เช่น PNG เริ่มด้วย `\x89PNG`, PDF เริ่มด้วย `%PDF`) เพื่อบอกชนิดไฟล์เบื้องต้นโดยไม่ต้องอ่านไฟล์ทั้งไฟล์เข้า memory ก่อน
4. ปรับปรุงตัวอย่างในหัวข้อ 5 ให้ทดลองหลายขนาด buffer (`bufio.NewWriterSize` ที่ 512, 4096, 65536 byte) แล้วเปรียบเทียบว่าจำนวนครั้งที่ `Write()` จริงถูกเรียกเปลี่ยนไปอย่างไรเมื่อขนาด buffer เปลี่ยน
5. เขียนโปรแกรม CLI ที่ห่อ `os.Stdin`/`os.Stdout` ด้วย `bufio` ตามหัวข้อ 6 ให้เป็นเครื่องคิดเลขง่ายๆ ที่รับ input รูปแบบ `เลข1 ตัวดำเนินการ เลข2` (เช่น `5 + 3`) ทีละบรรทัดแล้วพิมพ์ผลลัพธ์ จนกว่าจะพิมพ์คำว่า `exit`
6. เปรียบเทียบเวลาที่ใช้ระหว่างการเขียนไฟล์ขนาดใหญ่ (เช่น 1 ล้านบรรทัด) ด้วย `os.File.WriteString` ตรงๆ กับการห่อด้วย `bufio.Writer` แล้ว `Flush()` ท้ายสุด โดยใช้ `time.Since` วัดเวลาแบบเดียวกับที่เรียนใน **Part 019** ตอนเปรียบเทียบ `strings.Builder` กับ `+=`

---

**ต่อไป**: [Part 050 — แพ็กเกจ `bytes`](./050-bytes-package.md)
