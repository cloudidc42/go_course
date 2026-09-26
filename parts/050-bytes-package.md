# Part 050: แพ็กเกจ `bytes`

> ภาคที่ 4: Standard Library เชิงลึก — ตอนที่ 5 จาก 10 (Part 46–55)

## สารบัญของบทนี้

1. `bytes.Buffer`: Growable Byte Buffer ที่เป็นทั้ง `io.Reader` และ `io.Writer`
2. `bytes.Buffer` vs `strings.Builder`: ต่างกันตรงไหน และทำไม Builder ถึงเร็วกว่า
3. ฟังก์ชันใน `bytes` ที่ทำงานคู่กับ `strings` แบบเป๊ะๆ
4. `bytes.NewReader` vs `bytes.NewBuffer`
5. เมื่อไรควรใช้ `[]byte` แทน `string`: หลีกเลี่ยง Allocation ใน Hot Path
6. `bytes.Equal` vs `==`: ทำไม Slice เทียบกันตรงๆ ไม่ได้
7. สรุปสิ่งที่ได้เรียนในบทนี้
8. แบบฝึกหัดท้ายบท

---

## 1. `bytes.Buffer`: Growable Byte Buffer ที่เป็นทั้ง `io.Reader` และ `io.Writer`

`bytes.Buffer` คือโครงสร้างข้อมูลที่เก็บ `[]byte` ที่ขยายขนาดได้อัตโนมัติ (growable) พร้อมกับ **implement ทั้ง `io.Reader` และ `io.Writer` ในตัวเดียวกัน** (ทบทวน `io.ReadWriter` จาก **Part 048**) ทำให้มันเป็นเครื่องมือที่ยืดหยุ่นที่สุดตัวหนึ่งเมื่อต้องจัดการข้อมูลไบต์ใน memory:

```go
package main

import (
	"bytes"
	"fmt"
	"io"
)

func main() {
	var buf bytes.Buffer

	// ยืนยันด้วยการ compile ว่า *bytes.Buffer implement ทั้ง io.Reader และ io.Writer พร้อมกัน
	var _ io.Reader = &buf
	var _ io.Writer = &buf
	fmt.Println("*bytes.Buffer เป็นทั้ง io.Reader และ io.Writer ในตัวเดียว (ยืนยันด้วย compile-time check)")

	// เขียนข้อมูลหลายรูปแบบเข้า Buffer
	buf.WriteString("สวัสดี ")
	buf.WriteByte('G')
	buf.Write([]byte("o"))
	buf.WriteRune('!')
	fmt.Fprintf(&buf, " ระดับ %d", 1) // Buffer implement io.Writer จึงใช้กับ fmt.Fprintf ได้ตรงๆ

	fmt.Println("เนื้อหาทั้งหมดใน buffer (ผ่าน String()):", buf.String())
	fmt.Println("ความยาวข้อมูลที่ยังไม่ได้อ่าน (Len()):", buf.Len())

	// อ่านข้อมูลกลับออกมาผ่าน io.Reader interface - Buffer มี "ตำแหน่งอ่าน" ภายในของตัวเอง
	// เตรียม slice ให้มีขนาดเท่ากับคำว่า "สวัสดี " พอดี (ภาษาไทยเป็น multi-byte UTF-8 นับเป็น byte ไม่ใช่ตัวอักษร)
	firstWord := make([]byte, len("สวัสดี "))
	n, err := buf.Read(firstWord)
	fmt.Printf("อ่านมา %d bytes: %q, err: %v\n", n, firstWord, err)
	fmt.Println("หลังอ่านไปแล้ว ข้อมูลที่เหลือใน buffer:", buf.String())
	fmt.Println("(สังเกต: ข้อมูลที่อ่านไปแล้วหายไปจาก buffer จริงๆ ต่างจาก strings.Reader ที่แค่เลื่อน cursor)")

	fmt.Println()
	fmt.Println("=== Grow และ Reset ===")
	var buf2 bytes.Buffer
	buf2.Grow(1024) // จองความจุล่วงหน้า ลด allocation ซ้ำเวลาข้อมูลใหญ่ขึ้นเรื่อยๆ (เหมือน strings.Builder.Grow)
	buf2.WriteString("ข้อมูลชุดแรก")
	fmt.Println("ก่อน Reset:", buf2.String())
	buf2.Reset() // ล้างเนื้อหาทิ้งแต่ยังเก็บ capacity เดิมไว้ใช้ซ้ำได้ (ประหยัดกว่าสร้างตัวใหม่)
	buf2.WriteString("ข้อมูลชุดใหม่หลัง Reset")
	fmt.Println("หลัง Reset:", buf2.String())
}
```

ผลลัพธ์:

```
*bytes.Buffer เป็นทั้ง io.Reader และ io.Writer ในตัวเดียว (ยืนยันด้วย compile-time check)
เนื้อหาทั้งหมดใน buffer (ผ่าน String()): สวัสดี Go! ระดับ 1
ความยาวข้อมูลที่ยังไม่ได้อ่าน (Len()): 40
อ่านมา 19 bytes: "สวัสดี ", err: <nil>
หลังอ่านไปแล้ว ข้อมูลที่เหลือใน buffer: Go! ระดับ 1
(สังเกต: ข้อมูลที่อ่านไปแล้วหายไปจาก buffer จริงๆ ต่างจาก strings.Reader ที่แค่เลื่อน cursor)

=== Grow และ Reset ===
ก่อน Reset: ข้อมูลชุดแรก
หลัง Reset: ข้อมูลชุดใหม่หลัง Reset
```

### จุดที่ต้องเข้าใจให้ชัด: `bytes.Buffer` เป็นทั้ง "อ่านได้" และ "เขียนได้" ในตัวเดียว จริงๆ

สังเกตว่าหลังเรียก `buf.Read(firstWord)` ข้อมูลที่อ่านไปแล้ว (`"สวัสดี "`) **หายไปจาก buffer จริงๆ** — `bytes.Buffer` เก็บตำแหน่ง "จุดที่อ่านไปแล้ว" ไว้ภายใน และตัด (`ตัด`) ข้อมูลส่วนหน้าที่อ่านไปแล้วทิ้งเมื่อมีที่ว่างเพียงพอ พฤติกรรมนี้ทำให้ `bytes.Buffer` เหมาะกับสถานการณ์แบบ **FIFO queue ของ byte** (เขียนเข้าไปเรื่อยๆ ด้านหลัง อ่านออกมาเรื่อยๆ จากด้านหน้า) เช่น buffer สำหรับ network protocol ที่ข้อมูลไหลเข้ามาเรื่อยๆ ขณะที่ประมวลผลไปด้วย — ต่างจาก `strings.Reader`/`bytes.Reader` ที่แค่มี "cursor" เลื่อนไปเรื่อยๆ โดยข้อมูลต้นฉบับยังอยู่ครบเสมอ (Seek กลับไปอ่านซ้ำได้)

---

## 2. `bytes.Buffer` vs `strings.Builder`: ต่างกันตรงไหน และทำไม Builder ถึงเร็วกว่า

**Part 019 หัวข้อ 4** สอน `strings.Builder` ไปแล้วในฐานะเครื่องมือต่อ string ที่มีประสิทธิภาพ ตอนนี้เราเรียน `bytes.Buffer` ที่ดูเผินๆ ก็ทำงานคล้ายกันมาก — คำถามที่พบบ่อยคือ **"เลือกใช้ตัวไหนดี"** คำตอบสั้นๆ:

> **`strings.Builder` เขียนได้อย่างเดียว (write-only) และเร็วกว่าเมื่อจุดหมายสุดท้ายคือ string — `bytes.Buffer` อ่านได้ด้วยเขียนได้ด้วย (read+write) และยืดหยุ่นกว่าเมื่อต้องทำงานร่วมกับ `io.Reader`/`io.Writer` อื่นๆ**

มาพิสูจน์ความต่างด้านประสิทธิภาพด้วยตัวเลขจริงผ่าน `testing.AllocsPerRun` (ฟังก์ชันวัดจำนวน heap allocation ต่อการเรียกหนึ่งครั้ง ใช้งานได้แม้ในโปรแกรมทั่วไปที่ไม่ใช่ไฟล์ `_test.go`):

```go
package main

import (
	"bytes"
	"fmt"
	"strings"
	"testing"
)

func buildWithBuffer(n int) string {
	var buf bytes.Buffer
	buf.Grow(n)
	for i := 0; i < n; i++ {
		buf.WriteString("x")
	}
	return buf.String()
}

func buildWithBuilder(n int) string {
	var b strings.Builder
	b.Grow(n)
	for i := 0; i < n; i++ {
		b.WriteString("x")
	}
	return b.String()
}

func main() {
	const n = 100_000

	r1 := buildWithBuffer(n)
	r2 := buildWithBuilder(n)
	fmt.Println("ผลลัพธ์ความยาวเท่ากัน:", len(r1) == len(r2))

	// ใช้ testing.AllocsPerRun เพื่อวัดจำนวนครั้งที่เกิด heap allocation ต่อการเรียกหนึ่งครั้ง
	// (ฟังก์ชันนี้อยู่ใน package testing แต่เรียกใช้ได้จากโปรแกรมทั่วไป ไม่จำเป็นต้องเป็นไฟล์ _test.go)
	allocsBuffer := testing.AllocsPerRun(100, func() {
		_ = buildWithBuffer(n)
	})
	allocsBuilder := testing.AllocsPerRun(100, func() {
		_ = buildWithBuilder(n)
	})

	fmt.Printf("bytes.Buffer:    %.0f allocations ต่อการเรียกหนึ่งครั้ง\n", allocsBuffer)
	fmt.Printf("strings.Builder: %.0f allocations ต่อการเรียกหนึ่งครั้ง\n", allocsBuilder)

	fmt.Println()
	fmt.Println("=== สาเหตุที่ Builder ทำ allocation น้อยกว่า ===")
	fmt.Println("strings.Builder.String() คืนค่าโดย 'ยืม' หน่วยความจำเดิมมาตีความเป็น string ตรงๆ")
	fmt.Println("(ผ่าน unsafe package ภายใน) โดยไม่ copy ข้อมูลเลย เพราะรู้ว่า Builder เป็น write-only")
	fmt.Println("จึงมั่นใจได้ว่าจะไม่มีใครมาแก้ไข buffer เดิมอีกหลังจากเรียก String() ไปแล้ว")
	fmt.Println()
	fmt.Println("bytes.Buffer.String() ต้อง copy ข้อมูลออกมาเป็น string ใหม่ทุกครั้งที่เรียก")
	fmt.Println("เพราะ Buffer ยังเป็น io.Writer ที่แก้ไขเนื้อหาต่อได้ - ถ้าคืน string แบบไม่ copy")
	fmt.Println("แล้วมีคนเขียนข้อมูลเพิ่มลง Buffer อีก string เดิมที่ 'ควรจะ immutable' จะเปลี่ยนค่าไปด้วย")
	fmt.Println("ซึ่งขัดกับสัญญาของภาษา Go ที่ string ต้อง immutable เสมอ (ทบทวนจาก Part 019)")
}
```

ผลลัพธ์:

```
ผลลัพธ์ความยาวเท่ากัน: true
bytes.Buffer:    2 allocations ต่อการเรียกหนึ่งครั้ง
strings.Builder: 1 allocations ต่อการเรียกหนึ่งครั้ง

=== สาเหตุที่ Builder ทำ allocation น้อยกว่า ===
strings.Builder.String() คืนค่าโดย 'ยืม' หน่วยความจำเดิมมาตีความเป็น string ตรงๆ
(ผ่าน unsafe package ภายใน) โดยไม่ copy ข้อมูลเลย เพราะรู้ว่า Builder เป็น write-only
จึงมั่นใจได้ว่าจะไม่มีใครมาแก้ไข buffer เดิมอีกหลังจากเรียก String() ไปแล้ว

bytes.Buffer.String() ต้อง copy ข้อมูลออกมาเป็น string ใหม่ทุกครั้งที่เรียก
เพราะ Buffer ยังเป็น io.Writer ที่แก้ไขเนื้อหาต่อได้ - ถ้าคืน string แบบไม่ copy
แล้วมีคนเขียนข้อมูลเพิ่มลง Buffer อีก string เดิมที่ 'ควรจะ immutable' จะเปลี่ยนค่าไปด้วย
ซึ่งขัดกับสัญญาของภาษา Go ที่ string ต้อง immutable เสมอ (ทบทวนจาก Part 019)
```

ตัวเลขยืนยันชัดเจน: การสร้าง string ขนาดเท่ากันด้วย `bytes.Buffer` ใช้ allocation มากกว่า `strings.Builder` อยู่ **1 ครั้งพอดี** — allocation ที่เกินมานี้คือ**การ copy ข้อมูลตอนเรียก `.String()`** นั่นเอง

### ตารางเปรียบเทียบสรุป

| | `bytes.Buffer` | `strings.Builder` |
|---|---|---|
| อ่านได้ (`io.Reader`) | ได้ | ไม่ได้ |
| เขียนได้ (`io.Writer`) | ได้ | ได้ |
| `.String()` copy ข้อมูลหรือไม่ | copy ทุกครั้งที่เรียก | ไม่ copy (คืนแบบ zero-copy) |
| คัดลอกค่า (`=`) ได้หรือไม่ | ได้ (แต่ต้องระวังเรื่อง buffer ภายใน) | **ห้ามคัดลอกหลังเขียนแล้ว** (มี noCopy guard ป้องกัน `go vet` เตือน) |
| เหมาะกับงาน | ต่อข้อมูลไปพร้อมกับอ่าน/ประมวลผลสลับกันไปมา, ทำงานร่วมกับ `io.Reader`/`io.Writer` อื่น | ต่อ string ล้วนๆ ที่รู้อยู่แล้วว่าจะแค่เขียนแล้วอ่านผลลัพธ์ตอนจบ |

**คำแนะนำเชิงปฏิบัติ**: ถ้าเป้าหมายสุดท้ายคือ "ได้ string ก้อนเดียว" และไม่ต้องอ่านระหว่างทาง ให้ใช้ `strings.Builder` เสมอ (ตามที่ **Part 019** แนะนำไปแล้ว) แต่ถ้าต้องการ**ผสมกับการอ่าน** ระหว่างทาง หรือต้องส่งต่อเป็น `io.Reader`/`io.Writer` ให้ฟังก์ชันอื่น (เช่น เป็น request body ให้ `http.NewRequest` ตามที่เรียนใน **Part 046**) `bytes.Buffer` คือตัวเลือกที่เหมาะสมกว่า

---

## 3. ฟังก์ชันใน `bytes` ที่ทำงานคู่กับ `strings` แบบเป๊ะๆ

package `bytes` ถูกออกแบบให้มีฟังก์ชันที่ **ชื่อเหมือนกันเป๊ะกับ package `strings`** (ที่เรียนไปแล้วใน **Part 019**) เพียงแต่รับ/คืนค่าเป็น `[]byte` แทน `string` ทำให้ความรู้ที่มีอยู่แล้วเกี่ยวกับ `strings` migrate มาใช้กับ `bytes` ได้แทบจะทันที:

```go
package main

import (
	"bytes"
	"fmt"
)

func main() {
	data := []byte("  Go เป็นภาษาที่ยอดเยี่ยม, เรียบง่าย, และเร็ว  ")

	// แพ็กเกจ bytes มีฟังก์ชันเกือบทั้งหมดที่ตรงกับ strings ทุกประการ เพียงแต่ทำงานกับ []byte
	// แทน string - ชื่อฟังก์ชันเหมือนกันเป๊ะ ต่างแค่ argument/return type เท่านั้น

	fmt.Printf("ต้นฉบับ: %q\n", data)

	trimmed := bytes.TrimSpace(data)
	fmt.Printf("bytes.TrimSpace:  %q\n", trimmed)

	fmt.Println("bytes.Contains(ค้นหา \"ยอดเยี่ยม\"):", bytes.Contains(data, []byte("ยอดเยี่ยม")))

	parts := bytes.Split(trimmed, []byte(", "))
	fmt.Println("bytes.Split ด้วย \", \":")
	for i, p := range parts {
		fmt.Printf("  [%d] %q\n", i, p)
	}

	joined := bytes.Join(parts, []byte(" | "))
	fmt.Printf("bytes.Join กลับด้วย \" | \": %q\n", joined)

	fmt.Println("bytes.ToUpper:", string(bytes.ToUpper([]byte("go is awesome"))))
	fmt.Println("bytes.HasPrefix (\"Go\"):", bytes.HasPrefix(trimmed, []byte("Go")))
	fmt.Println("bytes.Count (\"เ\"):", bytes.Count(trimmed, []byte("เ")))
	fmt.Println("bytes.Replace (แทน \",\" ด้วย \";\"):", string(bytes.ReplaceAll(trimmed, []byte(","), []byte(";"))))

	fmt.Println()
	fmt.Println("=== bytes.Equal: วิธีเปรียบเทียบ []byte ที่ถูกต้อง ===")
	a := []byte("สวัสดี")
	b := []byte("สวัสดี")
	fmt.Println("a กับ b มีเนื้อหาเหมือนกันหรือไม่ (bytes.Equal):", bytes.Equal(a, b))
	// a == b เปรียบเทียบตรงๆ ไม่ได้เลย - []byte ไม่ใช่ comparable type (compile error ถ้าลองเขียน a == b)
	fmt.Println("(หมายเหตุ: เขียน `a == b` ตรงๆ ไม่ได้ จะ compile error เพราะ []byte ไม่ใช่ comparable type)")
}
```

ผลลัพธ์:

```
ต้นฉบับ: "  Go เป็นภาษาที่ยอดเยี่ยม, เรียบง่าย, และเร็ว  "
bytes.TrimSpace:  "Go เป็นภาษาที่ยอดเยี่ยม, เรียบง่าย, และเร็ว"
bytes.Contains(ค้นหา "ยอดเยี่ยม"): true
bytes.Split ด้วย ", ":
  [0] "Go เป็นภาษาที่ยอดเยี่ยม"
  [1] "เรียบง่าย"
  [2] "และเร็ว"
bytes.Join กลับด้วย " | ": "Go เป็นภาษาที่ยอดเยี่ยม | เรียบง่าย | และเร็ว"
bytes.ToUpper: GO IS AWESOME
bytes.HasPrefix ("Go"): true
bytes.Count ("เ"): 4
bytes.Replace (แทน "," ด้วย ";"): Go เป็นภาษาที่ยอดเยี่ยม; เรียบง่าย; และเร็ว

=== bytes.Equal: วิธีเปรียบเทียบ []byte ที่ถูกต้อง ===
a กับ b มีเนื้อหาเหมือนกันหรือไม่ (bytes.Equal): true
(หมายเหตุ: เขียน `a == b` ตรงๆ ไม่ได้ จะ compile error เพราะ []byte ไม่ใช่ comparable type)
```

ฟังก์ชันอื่นที่พบบ่อยและมีคู่แฝดใน `strings` ทุกตัว ได้แก่ `bytes.Fields` (แบ่งด้วยช่องว่างติดๆ กันกี่ตัวก็ได้ คล้าย `bufio.ScanWords` ที่เรียนใน **Part 049** แต่ทำงานกับข้อมูลทั้งก้อนในความจำทีเดียวแทนที่จะ stream ทีละ token), `bytes.Runes` (แปลง `[]byte` เป็น `[]rune` สำหรับกรณีที่ต้องเข้าถึงทีละตัวอักษร UTF-8 ทบทวนแนวคิด rune จาก **Part 019**), และ `bytes.Repeat` (ทำซ้ำ `[]byte` เท่ากับที่ `strings.Repeat` ทำกับ `string`)

### ทำไมต้องมีทั้ง `bytes` และ `strings` แยกกัน ทั้งที่หน้าตาเหมือนกันเป๊ะ

คำตอบสั้นๆ คือ **Go ไม่มี generic function overloading** (ฟังก์ชันชื่อเดียวกันที่รับ type ต่างกันได้หลายแบบ) ก่อนยุค generics (**Part 028-029**) การมี type ที่ทำงานคล้ายกันแต่เป็นคนละ type (`string` กับ `[]byte`) จำเป็นต้องมีฟังก์ชันแยกชุดกันอย่างสิ้นเชิง แม้ generics จะเข้ามาใน Go 1.18 แล้ว แต่ standard library ก็ยังคงแยก package ทั้งสองไว้ตามเดิม เพราะการเปลี่ยนแปลงตอนนี้จะกระทบโค้ดทั่วโลกมหาศาล (ผูกกับ **Go 1 Compatibility Promise** ที่เรียนไปตั้งแต่ **Part 001**) — ในทางปฏิบัติ นักพัฒนา Go จึงต้อง "รู้จักสองครั้ง" ทั้ง `strings.Xxx` และ `bytes.Xxx` แต่ข่าวดีคือ**พฤติกรรมและชื่อเหมือนกันแทบทุกฟังก์ชัน** ทำให้เรียนรู้ได้เร็ว

---

## 4. `bytes.NewReader` vs `bytes.NewBuffer`

ทั้งสองฟังก์ชันสร้างค่าที่ implement `io.Reader` ได้จาก `[]byte` ที่มีอยู่แล้ว แต่มีจุดต่างสำคัญ:

```go
package main

import (
	"bytes"
	"fmt"
	"io"
)

func main() {
	data := []byte("ข้อมูลต้นฉบับ")

	fmt.Println("=== bytes.NewReader: อ่านอย่างเดียว รองรับ Seek/ReadAt ===")
	r := bytes.NewReader(data)
	// *bytes.Reader implement io.Reader, io.Seeker, io.ReaderAt, io.WriterTo ครบ - เหมาะกับกรณี
	// ที่มีข้อมูลอยู่แล้วใน []byte และต้องการแค่ "อ่าน" มัน (เช่น ส่งเป็น request body ใน Part 046)
	buf := make([]byte, 6)
	n, _ := r.Read(buf)
	fmt.Printf("อ่านครั้งแรก %d bytes: %q\n", n, buf[:n])

	r.Seek(0, io.SeekStart) // ย้อนกลับไปอ่านใหม่ตั้งแต่ต้นได้ (io.Seeker)
	all, _ := io.ReadAll(r)
	fmt.Printf("หลัง Seek กลับไปจุดเริ่มต้น อ่านใหม่ทั้งหมด: %q\n", all)

	fmt.Println()
	fmt.Println("=== bytes.NewBuffer: เริ่มต้นจากข้อมูลเดิม แต่เขียนต่อได้ ===")
	// bytes.NewBuffer(initial []byte) สร้าง *bytes.Buffer จากข้อมูลที่มีอยู่แล้ว
	// ต่างจาก bytes.NewReader ตรงที่ผลลัพธ์ยังเป็น io.Writer เขียนข้อมูลเพิ่มต่อท้ายได้ด้วย
	b := bytes.NewBuffer([]byte("เริ่มต้นด้วยข้อความนี้ "))
	b.WriteString("แล้วเขียนต่อท้ายได้อีก")
	fmt.Println("bytes.NewBuffer + เขียนต่อ:", b.String())

	fmt.Println()
	fmt.Println("=== เมื่อไรใช้อันไหน ===")
	fmt.Println("bytes.NewReader: มีข้อมูลอยู่แล้ว ต้องการแค่อ่าน (อ่านซ้ำได้ด้วย Seek) - เบากว่า Buffer")
	fmt.Println("bytes.NewBuffer: มีข้อมูลเริ่มต้น และอาจต้องเขียนเพิ่ม/อ่านสลับกันไปมา")
}
```

ผลลัพธ์:

```
=== bytes.NewReader: อ่านอย่างเดียว รองรับ Seek/ReadAt ===
อ่านครั้งแรก 6 bytes: "ข้"
หลัง Seek กลับไปจุดเริ่มต้น อ่านใหม่ทั้งหมด: "ข้อมูลต้นฉบับ"

=== bytes.NewBuffer: เริ่มต้นจากข้อมูลเดิม แต่เขียนต่อได้ ===
bytes.NewBuffer + เขียนต่อ: เริ่มต้นด้วยข้อความนี้ แล้วเขียนต่อท้ายได้อีก

=== เมื่อไรใช้อันไหน ===
bytes.NewReader: มีข้อมูลอยู่แล้ว ต้องการแค่อ่าน (อ่านซ้ำได้ด้วย Seek) - เบากว่า Buffer
bytes.NewBuffer: มีข้อมูลเริ่มต้น และอาจต้องเขียนเพิ่ม/อ่านสลับกันไปมา
```

### ทำไม `bytes.NewReader` ถึง "เบากว่า" `bytes.NewBuffer`

`*bytes.Reader` มี field ภายในน้อยกว่ามาก (แค่ slice ต้นฉบับ + ตำแหน่ง cursor ปัจจุบัน) และไม่ต้องคอย "ตัดข้อมูลที่อ่านไปแล้วทิ้ง" แบบที่ `bytes.Buffer` ทำ (ทบทวนหัวข้อ 1) ทำให้มัน**เหมาะกับการอ่านซ้ำได้หลายรอบ**ผ่าน `Seek` โดยไม่มีค่าใช้จ่ายเพิ่ม — นี่คือเหตุผลที่ **Part 046 หัวข้อ 9** เลือกใช้ `bytes.NewReader(jsonBytes)` (ไม่ใช่ `bytes.NewBuffer`) ตอนส่ง JSON เป็น request body: เราแค่ต้องการ "อ่าน" ข้อมูลที่มีอยู่แล้วส่งออกไป ไม่ได้ต้องการเขียนอะไรเพิ่มอีก

---

## 5. เมื่อไรควรใช้ `[]byte` แทน `string`: หลีกเลี่ยง Allocation ใน Hot Path

**Part 019** อธิบายไว้ว่า string ใน Go เป็น immutable และเก็บเป็น UTF-8 byte sequence ภายใน — สิ่งที่สำคัญมากและมักถูกมองข้ามคือ: **ทุกครั้งที่แปลงระหว่าง `string` กับ `[]byte` (`string(b)` หรือ `[]byte(s)`) Go ต้อง copy ข้อมูลทั้งก้อนเสมอ** ไม่ใช่แค่ "มองข้อมูลเดิมในมุมมองใหม่" เหมือนที่หลายภาษาอื่นทำได้ (เพราะ string ต้อง immutable แต่ `[]byte` mutable — ถ้าไม่ copy แล้วมีใครแก้ไข `[]byte` เดิม string ที่ "ควรจะ immutable" จะเปลี่ยนค่าตามไปด้วย ซึ่งเป็นไปไม่ได้ตามกฎของภาษา)

ในโค้ดที่ถูกเรียกซ้ำบ่อยมาก (**hot path** — คำที่ใช้เรียกโค้ดส่วนที่ทำงานหนักที่สุดของโปรแกรม) การแปลงชนิดที่ดูเหมือนไม่มีอะไรนี้ อาจกลายเป็นคอขวดด้านประสิทธิภาพได้จริง มาพิสูจน์ด้วยตัวเลข:

```go
package main

import (
	"bytes"
	"fmt"
	"strings"
	"testing"
)

var data = []byte("the quick brown fox jumps over the lazy dog the fox runs away")

// countTheWithConversion แปลง []byte เป็น string ทุกครั้งก่อนนับ - การแปลง []byte -> string
// ต้อง "copy" ข้อมูลทั้งหมดไปสร้าง string ใหม่เสมอ (เพราะ string ต้อง immutable ตามที่เรียนใน
// Part 019) ทำให้เกิด allocation ที่ไม่จำเป็นทุกครั้งที่ฟังก์ชันนี้ถูกเรียก
func countTheWithConversion(b []byte) int {
	s := string(b) // <-- จุดที่เกิดการ copy ข้อมูลทั้งก้อน
	return strings.Count(s, "the")
}

// countTheDirect ทำงานกับ []byte โดยตรงตลอด ไม่มีการแปลงชนิดเลยสักครั้ง
// เพราะแพ็กเกจ bytes มีฟังก์ชัน Count ที่ทำงานกับ []byte ได้ตรงๆ อยู่แล้ว
func countTheDirect(b []byte) int {
	return bytes.Count(b, []byte("the"))
}

func main() {
	c1 := countTheWithConversion(data)
	c2 := countTheDirect(data)
	fmt.Println("ผลลัพธ์เท่ากันหรือไม่:", c1 == c2, "(นับได้", c1, "ครั้ง)")

	allocsConversion := testing.AllocsPerRun(1000, func() {
		countTheWithConversion(data)
	})
	allocsDirect := testing.AllocsPerRun(1000, func() {
		countTheDirect(data)
	})

	fmt.Printf("countTheWithConversion: %.0f allocations ต่อครั้ง\n", allocsConversion)
	fmt.Printf("countTheDirect:         %.0f allocations ต่อครั้ง\n", allocsDirect)

	fmt.Println()
	fmt.Println("=== เมื่อไรควรใช้ []byte แทน string ===")
	fmt.Println("1. เมื่อข้อมูลต้นทางเป็น []byte อยู่แล้ว (เช่น จาก io.Reader, network, ไฟล์)")
	fmt.Println("   และงานที่ต้องทำมีฟังก์ชันเทียบเท่าใน package bytes ครบอยู่แล้ว")
	fmt.Println("2. เมื่ออยู่ใน hot path (โค้ดที่ถูกเรียกซ้ำบ่อยมาก) ที่ต้องการเลี่ยง allocation")
	fmt.Println("   ทุก string(b) และ []byte(s) คือการ copy ข้อมูลทั้งก้อนเสมอ ไม่ใช่แค่ 'มองต่างมุม'")
	fmt.Println("3. เมื่อต้องแก้ไขข้อมูล in-place (เช่น เปลี่ยนตัวอักษรบางตำแหน่งโดยไม่สร้าง string ใหม่)")
	fmt.Println("   เพราะ []byte เปลี่ยนแปลงได้ (mutable) ต่างจาก string ที่ immutable เสมอ")
}
```

ผลลัพธ์:

```
ผลลัพธ์เท่ากันหรือไม่: true (นับได้ 3 ครั้ง)
countTheWithConversion: 1 allocations ต่อครั้ง
countTheDirect:         0 allocations ต่อครั้ง
```

ตัวเลขนี้ชัดเจนมาก: ฟังก์ชันที่แปลง `[]byte` เป็น `string` ก่อนใช้งาน มี allocation เกิดขึ้น **1 ครั้งต่อการเรียก** ในขณะที่ฟังก์ชันที่ทำงานกับ `[]byte` โดยตรงตลอด **ไม่มี allocation เกิดขึ้นเลย** (`0`) — ถ้าฟังก์ชันนี้ถูกเรียกหลักล้านครั้งต่อวินาที (เช่น parsing ข้อมูลใน network server ที่มี throughput สูง) ผลต่างนี้ส่งผลโดยตรงต่อทั้งความเร็วและแรงกดดันที่ garbage collector ต้องรับมือ (ทบทวนแนวคิด GC จะเรียนเจาะลึกใน **Part 055**)

**ข้อควรระวัง**: ไม่ใช่ว่าต้องหลีกเลี่ยงการแปลง `string`/`[]byte` ทุกครั้งเสมอไป — โค้ดส่วนใหญ่ในโปรแกรมทั่วไปไม่ได้อยู่ใน hot path และความชัดเจนของโค้ด (เขียนด้วย `string` เพราะอ่านง่ายกว่า) สำคัญกว่าการ optimize ที่ยังไม่ได้พิสูจน์ว่าจำเป็นจริง (ตามหลัก "premature optimization is the root of all evil") — ควรใช้ profiling เครื่องมืออย่าง `pprof` (จะเรียนใน **Part 083**) เพื่อหาจุดที่ "คุ้มค่า" ที่จะ optimize จริงๆ ก่อนเปลี่ยนโค้ดทั้งหมดมาใช้ `[]byte`

---

## 6. `bytes.Equal` vs `==`: ทำไม Slice เทียบกันตรงๆ ไม่ได้

**Part 006 หัวข้อ 11** อธิบายไปแล้วว่า **slice ไม่ใช่ comparable type** ในภาษา Go (ต่างจาก array ที่เทียบด้วย `==` ได้) — `[]byte` ก็คือ slice ชนิดหนึ่ง จึงใช้กฎเดียวกัน: **เขียน `a == b` ตรงๆ ระหว่าง `[]byte` สอง ตัวไม่ได้เลย จะ compile error ทันที**

```go
a := []byte("x")
b := []byte("x")
fmt.Println(a == b) // compile error!
```

Error ที่ได้:

```
./main.go:8:14: invalid operation: a == b (slice can only be compared to nil)
```

ข้อความ error บอกไว้ชัดเจนว่า **slice เทียบได้กับ `nil` เท่านั้น** (เช่น `if a == nil`) เทียบกับ slice อีกตัวไม่ได้เลยไม่ว่ากรณีใด

### วิธีที่ถูกต้อง: `bytes.Equal`

```go
a := []byte("สวัสดี")
b := []byte("สวัสดี")
fmt.Println(bytes.Equal(a, b)) // true - เปรียบเทียบเนื้อหาทีละ byte
```

`bytes.Equal(a, b []byte) bool` เปรียบเทียบเนื้อหาของ slice ทั้งสองตัวทีละ byte (value equality) คืน `true` ถ้าเนื้อหาเหมือนกันทุกประการและมีความยาวเท่ากัน — เป็นวิธีเดียวที่ถูกต้องในการเช็คว่า `[]byte` สอง ตัว "มีค่าเดียวกัน" หรือไม่

**ทางเลือกอื่นที่ควรรู้จักแต่ไม่แนะนำสำหรับกรณีนี้**: `reflect.DeepEqual(a, b)` (ทบทวนแนวคิด reflection จาก **Part 031**) ก็เปรียบเทียบ slice ได้เช่นกัน แต่**ช้ากว่า `bytes.Equal` มาก** เพราะ `reflect.DeepEqual` เป็นฟังก์ชันทั่วไปที่ต้องใช้ reflection ตรวจสอบ type และโครงสร้างข้อมูลใดๆ ก็ได้ ในขณะที่ `bytes.Equal` เป็นฟังก์ชันเฉพาะทางที่ optimize มาสำหรับ `[]byte` โดยตรง (ในหลาย platform ใช้ CPU instruction พิเศษเปรียบเทียบข้อมูลหลาย byte พร้อมกันด้วยซ้ำ) — **กฎทอง**: ใช้ `bytes.Equal` เสมอเมื่อเทียบ `[]byte` สองตัว อย่าใช้ `reflect.DeepEqual` ถ้าไม่จำเป็นจริงๆ

### `bytes.Compare`: เมื่อต้องการรู้ "มากกว่า/น้อยกว่า" ไม่ใช่แค่ "เท่ากันไหม"

บางสถานการณ์ไม่ได้ต้องการแค่เช็คว่าเนื้อหาเหมือนกันหรือไม่ แต่ต้องการ**จัดลำดับ** `[]byte` ด้วย เช่น เรียง key ของฐานข้อมูลแบบ byte-order, หรือใช้เป็น comparator ให้ `sort.Slice` (ทบทวนจาก **Part 027**) — `bytes.Compare(a, b []byte) int` ตอบโจทย์นี้โดยตรง:

```go
fmt.Println(bytes.Compare([]byte("apple"), []byte("banana"))) // -1 (a น้อยกว่า b)
fmt.Println(bytes.Compare([]byte("banana"), []byte("apple"))) // 1  (a มากกว่า b)
fmt.Println(bytes.Compare([]byte("go"), []byte("go")))        // 0  (เท่ากัน)
```

`bytes.Compare` คืน `-1` ถ้า `a < b`, `0` ถ้า `a == b`, และ `1` ถ้า `a > b` เปรียบเทียบแบบ **lexicographic (เรียงตามพจนานุกรม)** ทีละ byte — ใช้ `bytes.Equal(a, b)` เมื่อสนใจแค่ "เท่ากันไหม" (เร็วกว่าเล็กน้อยเพราะหยุดทันทีที่เจอ byte ที่ไม่ตรงกัน โดยไม่ต้องรู้ว่าฝั่งไหนมากกว่า) และใช้ `bytes.Compare` เมื่อต้องการนำผลไปจัดลำดับต่อ

### เชื่อมโยงกลับไปที่ Part 006: ทำไม array เทียบได้แต่ slice เทียบไม่ได้

ทบทวนสั้นๆ จาก **Part 006**: **array** มีขนาดคงที่ที่รู้ตอน compile time และ Go เปรียบเทียบมันแบบ **value type** (เทียบค่าทุกช่องตรงๆ) จึงเป็น comparable และใช้เป็น map key ได้ ส่วน **slice** เป็นแค่ "ตัวชี้ไปยัง underlying array + ความยาว + ความจุ" การเปรียบเทียบ slice แบบ `==` มีความหมายกำกวม (ควรเทียบว่าชี้ไปที่ array เดียวกันหรือควรเทียบเนื้อหาข้างในทีละตัว) ภาษา Go จึงเลือก**ไม่อนุญาตให้เทียบเลย** (ยกเว้นกับ `nil`) เพื่อบังคับให้ผู้เขียนโค้ดต้อง**ตั้งใจเลือก**วิธีเปรียบเทียบที่ถูกต้องเอง (`bytes.Equal` สำหรับเทียบเนื้อหา, หรือเทียบ pointer ของ underlying array โดยตรงถ้าต้องการแบบนั้นจริงๆ) แทนที่จะปล่อยให้เกิดความกำกวมซ่อนอยู่ในโค้ดที่ดูเหมือนใช้งานได้ปกติ

---

## สรุปสิ่งที่ได้เรียนในบทนี้

- `bytes.Buffer` เป็น growable byte buffer ที่ implement ทั้ง `io.Reader` และ `io.Writer` — ข้อมูลที่อ่านไปแล้วจะถูกตัดออกจาก buffer จริง ต่างจาก `strings.Reader`/`bytes.Reader` ที่แค่เลื่อน cursor
- `strings.Builder` เขียนได้อย่างเดียวแต่เร็วกว่า `bytes.Buffer` เมื่อสร้าง string เพราะ `.String()` คืนค่าแบบ zero-copy ในขณะที่ `bytes.Buffer.String()` ต้อง copy ข้อมูลทุกครั้ง (พิสูจน์ด้วย `testing.AllocsPerRun`: 1 allocation ต่างกันพอดี) — เลือก Builder เมื่อแค่ต้องการ string สุดท้าย เลือก Buffer เมื่อต้องอ่าน/เขียนสลับกันหรือทำงานร่วมกับ `io.Reader`/`io.Writer`
- package `bytes` มีฟังก์ชันชื่อเดียวกับ `strings` แทบทุกตัว (`Contains`, `Split`, `Join`, `TrimSpace`, `ToUpper`, ฯลฯ) เพียงแต่ทำงานกับ `[]byte` แทน `string`
- `bytes.NewReader` เบากว่าและเหมาะกับการอ่านอย่างเดียว (รองรับ `Seek`), `bytes.NewBuffer` เหมาะกับข้อมูลที่ต้องเขียนเพิ่มต่อท้ายได้
- แปลงระหว่าง `string` กับ `[]byte` เป็นการ **copy ข้อมูลทั้งก้อนเสมอ** ไม่ใช่แค่มองต่างมุม — ใน hot path ควรทำงานกับ `[]byte` โดยตรงถ้าข้อมูลต้นทางเป็น `[]byte` อยู่แล้ว เพื่อลด allocation (พิสูจน์แล้วว่าต่างกัน 1 vs 0 allocation ต่อการเรียก)
- `[]byte` เป็น slice จึง **ไม่ใช่ comparable type** เทียบด้วย `==` ตรงๆ ไม่ได้ (compile error) ต้องใช้ `bytes.Equal` เสมอ ซึ่งเร็วกว่า `reflect.DeepEqual` มาก — เชื่อมโยงกับกฎ array-vs-slice comparability จาก **Part 006**

## แบบฝึกหัดท้ายบท

1. เขียนฟังก์ชัน `reverseBytes(b []byte) []byte` ที่กลับลำดับของ `[]byte` โดยไม่แปลงเป็น `string` เลยระหว่างทาง แล้วทดสอบว่าทำงานถูกต้องกับข้อความภาษาไทย (ระวัง: การกลับลำดับทีละ byte ตรงๆ จะทำให้ตัวอักษร UTF-8 หลาย byte เสียหาย ลองคิดวิธีแก้ไขที่ถูกต้อง หรือระบุข้อจำกัดนี้ในคำตอบ)
2. ใช้ `testing.AllocsPerRun` เปรียบเทียบ allocation ระหว่างการต่อ `[]byte` ด้วย `append` ตรงๆ ในลูป กับการใช้ `bytes.Buffer.Write` ในลูปเดียวกัน (จำนวนรอบเท่ากัน) แล้วอธิบายผลลัพธ์ที่ได้
3. เขียนโปรแกรมที่อ่านไฟล์ CSV (ทบทวน `os`/`bufio` จาก **Part 024, 049**) แล้วใช้ `bytes.Split`/`bytes.TrimSpace` แยกแต่ละ field ออกจากกันโดยทำงานกับ `[]byte` ตลอดทั้งกระบวนการ ไม่แปลงเป็น `string` จนกว่าจะถึงขั้นตอนสุดท้ายที่ต้องแสดงผล
4. ทดลองเขียน `if a == b` ระหว่างตัวแปร `[]byte` สองตัวเอง แล้วอ่าน error message ที่ได้ให้ละเอียด อธิบายว่าทำไม Go ถึงบอกว่า "slice can only be compared to nil" แทนที่จะบอกแค่ว่า "compare ไม่ได้" เฉยๆ
5. เขียนฟังก์ชัน `containsAny(b []byte, targets ...[]byte) bool` ที่เช็คว่า `b` มี substring ใดๆ จาก `targets` อยู่บ้างหรือไม่ โดยใช้ `bytes.Contains` วนเช็คทีละตัว แล้วเปรียบเทียบวิธีการเขียนกับเวอร์ชันที่ใช้ `strings.ContainsAny` (ทบทวนจาก **Part 019**) ว่าต่างกันอย่างไร
6. วัด allocation ของการใช้ `bytes.NewReader` เทียบกับ `bytes.NewBuffer` เมื่อสร้างจากข้อมูลก้อนเดียวกันแล้วอ่านทั้งหมดออกมาด้วย `io.ReadAll` ทันทีโดยไม่เขียนเพิ่ม อธิบายว่าทำไมถึงมีผลต่างหรือไม่มีผลต่าง

---

**ต่อไป**: [Part 051 — Cryptography พื้นฐาน: hashing, encryption](./051-crypto-basics.md)
