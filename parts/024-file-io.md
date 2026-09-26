# Part 024: File I/O: `os`, `io`, `bufio`

> ภาคที่ 2: ระดับกลาง (Intermediate) — ตอนที่ 9 จาก 20 (Part 16–35)

## สารบัญของบทนี้

1. ภาพรวม: สาม package ที่ทำงานร่วมกันเรื่อง I/O
2. อ่าน/เขียนไฟล์แบบง่ายที่สุด: `os.ReadFile` และ `os.WriteFile`
3. `os.Open`, `os.Create`, `os.OpenFile` และการควบคุมแบบละเอียด
4. `io.Reader` และ `io.Writer`: นามธรรมสากลของ I/O ใน Go
5. อ่านทีละบรรทัดด้วย `bufio.Scanner`
6. เขียนแบบ Buffered ด้วย `bufio.Writer` และความสำคัญของ `Flush()`
7. จัดการ error: แยกแยะไฟล์ไม่พบ vs error ประเภทอื่น
8. ทำงานกับไดเรกทอรี: `os.Mkdir`, `os.MkdirAll`, `os.ReadDir`
9. ปิดไฟล์ให้ถูกต้องเสมอด้วย `defer f.Close()`
10. สรุปสิ่งที่ได้เรียนในบทนี้
11. แบบฝึกหัดท้ายบท

---

## 1. ภาพรวม: สาม package ที่ทำงานร่วมกันเรื่อง I/O

การทำงานกับไฟล์ใน Go เกี่ยวข้องกับ 3 package หลักที่ทำงานประสานกัน:

- **`os`** — จัดการกับระบบปฏิบัติการโดยตรง เช่น เปิด/ปิด/สร้าง/ลบไฟล์และไดเรกทอรี
- **`io`** — กำหนด **interface มาตรฐาน** สำหรับการอ่าน/เขียนข้อมูล (ไม่สนใจว่าปลายทางเป็นไฟล์ เครือข่าย หรือ memory)
- **`bufio`** — เพิ่มชั้น **buffering** ให้กับการอ่าน/เขียน เพื่อประสิทธิภาพที่ดีขึ้นและมี utility สะดวกๆ เช่นอ่านทีละบรรทัด

ความสัมพันธ์ระหว่างสาม package นี้คือ: `os.File` (ค่าที่ได้จากการเปิดไฟล์ด้วย `os`) **implement interface `io.Reader` และ `io.Writer`** โดยอัตโนมัติ และ `bufio` ก็ **ห่อหุ้ม (wrap) รอบ `io.Reader`/`io.Writer` ใดๆ ก็ได้** ไม่จำเป็นต้องเป็นไฟล์เท่านั้น — นี่คือพลังของการออกแบบ Go ด้วย interface ที่เรียนมาตั้งแต่ **Part 013** และ **Part 014**: เขียนโค้ดครั้งเดียว ใช้ได้กับทุกแหล่งข้อมูลที่ implement interface เดียวกัน

---

## 2. อ่าน/เขียนไฟล์แบบง่ายที่สุด: `os.ReadFile` และ `os.WriteFile`

สำหรับงานง่ายๆ ที่ต้องการอ่านหรือเขียนไฟล์ทั้งไฟล์ในคำสั่งเดียว (ไม่ต้องควบคุมแบบละเอียด) `os` มีฟังก์ชันสำเร็จรูปให้ใช้:

```go
package main

import (
	"fmt"
	"os"
)

func main() {
	dir := "filedata"
	os.MkdirAll(dir, 0755)

	// os.WriteFile - เขียนไฟล์ทั้งก้อนในคำสั่งเดียว
	content := []byte("สวัสดี Go\nบรรทัดที่สอง\n")
	err := os.WriteFile(dir+"/greeting.txt", content, 0644)
	if err != nil {
		fmt.Println("write error:", err)
		return
	}
	fmt.Println("เขียนไฟล์สำเร็จ")

	// os.ReadFile - อ่านไฟล์ทั้งก้อนในคำสั่งเดียว
	data, err := os.ReadFile(dir + "/greeting.txt")
	if err != nil {
		fmt.Println("read error:", err)
		return
	}
	fmt.Println("เนื้อหาไฟล์:")
	fmt.Print(string(data))
}
```

ผลลัพธ์:

```
เขียนไฟล์สำเร็จ
เนื้อหาไฟล์:
สวัสดี Go
บรรทัดที่สอง
```

### อธิบายทีละส่วน

- `os.MkdirAll(dir, 0755)` — สร้างไดเรกทอรี (รวมทั้ง parent directory ที่ยังไม่มี) เลข `0755` คือ **file permission แบบ Unix** (owner อ่าน/เขียน/execute ได้, กลุ่มและคนอื่นอ่าน/execute ได้อย่างเดียว) — ถ้าไดเรกทอรีมีอยู่แล้ว `MkdirAll` จะไม่ error
- `os.WriteFile(path, data []byte, perm os.FileMode)` — เขียน `[]byte` ลงไฟล์ทั้งก้อน **สร้างไฟล์ใหม่ถ้ายังไม่มี หรือ overwrite ทั้งหมดถ้ามีอยู่แล้ว** เลข `0644` คือ permission ของไฟล์ (owner อ่าน/เขียนได้ คนอื่นอ่านได้อย่างเดียว)
- `os.ReadFile(path)` — อ่านไฟล์ทั้งไฟล์กลับมาเป็น `[]byte` เดียว คืนค่าเป็น `([]byte, error)` ตามหลัก error handling จาก **Part 015**

**ข้อจำกัดสำคัญ**: `os.ReadFile` โหลด**ทั้งไฟล์เข้า memory ในครั้งเดียว** เหมาะกับไฟล์ขนาดเล็ก-กลาง ถ้าไฟล์มีขนาดใหญ่มาก (เช่นหลาย GB) การใช้ `os.ReadFile` อาจทำให้โปรแกรมใช้ memory เกินความจำเป็นหรือ crash ได้ กรณีนี้ควรใช้การอ่านแบบ stream ผ่าน `bufio.Scanner` หรือ `io.Reader` แทน (จะอธิบายในหัวข้อ 5)

---

## 3. `os.Open`, `os.Create`, `os.OpenFile` และการควบคุมแบบละเอียด

เมื่อไรที่ต้องการควบคุมมากกว่าการอ่าน/เขียนทั้งไฟล์ในครั้งเดียว (เช่น อ่านทีละส่วน, เขียนต่อท้าย, ควบคุม flag การเปิดไฟล์) ต้องใช้ฟังก์ชันกลุ่มนี้แทน ซึ่งทั้งหมดคืนค่าเป็น `*os.File`:

| ฟังก์ชัน | ความหมาย |
|---|---|
| `os.Open(name)` | เปิดไฟล์แบบ **read-only** เท่านั้น (เทียบเท่า `OpenFile(name, O_RDONLY, 0)`) |
| `os.Create(name)` | สร้างไฟล์ใหม่แบบ **read-write** — ถ้ามีไฟล์อยู่แล้วจะถูก **truncate** (ล้างเนื้อหาเดิมทิ้ง) |
| `os.OpenFile(name, flag, perm)` | เปิดไฟล์แบบกำหนด flag เองได้เต็มที่ (เช่น append, create-if-not-exist) |

```go
package main

import (
	"fmt"
	"io"
	"os"
)

func main() {
	dir := "filedata"

	// os.Create - สร้างไฟล์ใหม่ (หรือ truncate ถ้ามีอยู่แล้ว) คืน *os.File
	f, err := os.Create(dir + "/log.txt")
	if err != nil {
		fmt.Println("create error:", err)
		return
	}
	// เขียนผ่าน io.Writer interface (f คือ *os.File ซึ่ง implement io.Writer)
	_, err = f.WriteString("เริ่มต้น log\n")
	if err != nil {
		fmt.Println("write error:", err)
	}
	n, err := f.Write([]byte("บรรทัดที่สอง\n"))
	fmt.Println("เขียนไป", n, "bytes, err:", err)

	if cerr := f.Close(); cerr != nil {
		fmt.Println("close error:", cerr)
	}

	// os.Open - เปิดไฟล์แบบ read-only คืน *os.File ซึ่ง implement io.Reader
	rf, err := os.Open(dir + "/log.txt")
	if err != nil {
		fmt.Println("open error:", err)
		return
	}
	defer func() {
		if cerr := rf.Close(); cerr != nil {
			fmt.Println("close error:", cerr)
		}
	}()

	// อ่านทั้งหมดผ่าน io.ReadAll (ใช้ io.Reader interface ตรงๆ)
	data, err := io.ReadAll(rf)
	if err != nil {
		fmt.Println("read error:", err)
		return
	}
	fmt.Println("อ่านได้:")
	fmt.Print(string(data))
}
```

ผลลัพธ์:

```
เขียนไป 37 bytes, err: <nil>
อ่านได้:
เริ่มต้น log
บรรทัดที่สอง
```

### `os.OpenFile` สำหรับควบคุม flag แบบละเอียด — เช่น การ Append

```go
path := "filedata/append.txt"
os.WriteFile(path, []byte("บรรทัดแรก\n"), 0644)

f, err := os.OpenFile(path, os.O_APPEND|os.O_WRONLY, 0644)
if err != nil {
	fmt.Println("open error:", err)
	return
}
defer f.Close()

f.WriteString("บรรทัดที่เพิ่มเข้ามา\n")

data, _ := os.ReadFile(path)
fmt.Print(string(data))
```

ผลลัพธ์:

```
บรรทัดแรก
บรรทัดที่เพิ่มเข้ามา
```

Flag ที่ใช้บ่อยที่สุดใน `os.OpenFile` (รวมกันได้ด้วย bitwise OR `|` ทบทวนจาก **Part 004**):

| Flag | ความหมาย |
|---|---|
| `os.O_RDONLY` | เปิดแบบอ่านอย่างเดียว |
| `os.O_WRONLY` | เปิดแบบเขียนอย่างเดียว |
| `os.O_RDWR` | เปิดแบบอ่านและเขียน |
| `os.O_APPEND` | เขียนต่อท้ายไฟล์เดิม (ไม่ overwrite) |
| `os.O_CREATE` | สร้างไฟล์ใหม่ถ้ายังไม่มี |
| `os.O_TRUNC` | ล้างเนื้อหาเดิมทิ้งตอนเปิด |
| `os.O_EXCL` | ใช้คู่กับ `O_CREATE` — ถ้าไฟล์มีอยู่แล้วจะ error ทันที (ป้องกันการเขียนทับโดยไม่ตั้งใจ) |

---

## 4. `io.Reader` และ `io.Writer`: นามธรรมสากลของ I/O ใน Go

นี่คือหัวใจสำคัญที่สุดของบทนี้ และเป็นหนึ่งใน interface ที่ทรงพลังที่สุดในภาษา Go ทั้งหมด:

```go
type Reader interface {
	Read(p []byte) (n int, err error)
}

type Writer interface {
	Write(p []byte) (n int, err error)
}
```

สังเกตว่าทั้งสอง interface มี method **เดียว** เท่านั้น (ตามปรัชญา "less is more" ที่เรียนใน **Part 001** — interface เล็กที่สุดเท่าที่จำเป็น) และนี่คือเหตุผลที่ทำให้มัน**นำไปใช้ได้กับแทบทุกอย่างที่เกี่ยวข้องกับข้อมูล**: ไฟล์ (`os.File`), การเชื่อมต่อเครือข่าย (`net.Conn`), ข้อมูลใน memory (`bytes.Buffer`, `strings.Reader`), ข้อมูลบีบอัด (`gzip.Reader`), หรือแม้แต่ stdin/stdout (`os.Stdin`, `os.Stdout`)

`*os.File` **implement ทั้ง `io.Reader` และ `io.Writer`** โดยอัตโนมัติ (เพราะมี method `Read` และ `Write` ที่ตรงกับ signature ของ interface พอดี — ทบทวนหลักการ "implicit interface satisfaction" จาก **Part 013**) นี่คือเหตุผลที่ในตัวอย่างข้างต้นเราส่ง `rf` (ซึ่งเป็น `*os.File`) เข้าไปในฟังก์ชัน `io.ReadAll(r io.Reader)` ได้โดยตรง ทั้งที่ `io.ReadAll` ไม่รู้จัก `os.File` เป็นการเฉพาะเลย — มันรู้จักแค่ว่ามันเป็นอะไรก็ตามที่มี method `Read`

**ทำไมเรื่องนี้สำคัญ**: โค้ดที่เขียนโดยรับ parameter เป็น `io.Reader` หรือ `io.Writer` แทนที่จะรับ `*os.File` ตรงๆ จะสามารถทำงานกับ**แหล่งข้อมูลใดก็ได้**โดยไม่ต้องแก้โค้ดเลย เช่น ฟังก์ชัน parse ข้อมูลที่เขียนให้รับ `io.Reader` จะทำงานได้ทั้งกับไฟล์จริง, ข้อมูลที่ส่งมาทาง HTTP, หรือ string ใน memory (ผ่าน `strings.NewReader`) — เราจะเรียนเรื่องนี้แบบเจาะลึกยิ่งขึ้นใน **Part 048 (`io.Reader`/`io.Writer` และ Interface Composition)** ซึ่งจะพูดถึง interface ประกอบร่างอื่นๆ เช่น `io.ReadWriter`, `io.Closer`, `io.ReadCloser`

---

## 5. อ่านทีละบรรทัดด้วย `bufio.Scanner`

การอ่านไฟล์ทั้งไฟล์ด้วย `os.ReadFile` ใช้ได้กับไฟล์เล็ก แต่ถ้าต้องการ**ประมวลผลทีละบรรทัด** (เช่น อ่าน log file, อ่าน CSV แบบง่าย, อ่าน input จากผู้ใช้) `bufio.Scanner` คือเครื่องมือที่เหมาะสมที่สุด:

```go
package main

import (
	"bufio"
	"fmt"
	"os"
)

func main() {
	dir := "filedata"

	f, err := os.Open(dir + "/greeting.txt")
	if err != nil {
		fmt.Println("open error:", err)
		return
	}
	defer f.Close()

	scanner := bufio.NewScanner(f)
	lineNo := 1
	for scanner.Scan() {
		line := scanner.Text()
		fmt.Printf("บรรทัด %d: %s\n", lineNo, line)
		lineNo++
	}
	if err := scanner.Err(); err != nil {
		fmt.Println("scan error:", err)
	}
}
```

ผลลัพธ์:

```
บรรทัด 1: สวัสดี Go
บรรทัด 2: บรรทัดที่สอง
```

### กลไกการทำงานของ `bufio.Scanner`

- `bufio.NewScanner(r io.Reader)` — สร้าง Scanner จาก `io.Reader` ใดๆ ก็ได้ (ไม่จำกัดแค่ไฟล์! ใช้กับ `os.Stdin` เพื่ออ่าน input จากผู้ใช้ทีละบรรทัดได้เช่นกัน)
- `scanner.Scan()` — คืนค่า `bool` เป็น `true` ถ้าอ่านบรรทัดถัดไปได้สำเร็จ, `false` เมื่อถึงจุดจบไฟล์ (EOF) หรือเกิด error — เขียนเป็น loop `for scanner.Scan() { ... }` ได้อย่างกระชับ
- `scanner.Text()` — คืนเนื้อหาบรรทัดปัจจุบันเป็น `string` (ตัด newline character ท้ายบรรทัดออกให้อัตโนมัติแล้ว)
- `scanner.Err()` — **ต้องเช็คเสมอหลัง loop จบ** เพราะ `Scan()` จะคืน `false` ทั้งกรณี "อ่านจบไฟล์ปกติ" และ "เกิด error ระหว่างอ่าน" — วิธีเดียวที่จะแยกสองกรณีนี้ออกจากกันคือเช็ค `Err()` หลัง loop (ถ้าจบไฟล์ปกติ `Err()` จะคืน `nil`)

`bufio.Scanner` สามารถเปลี่ยนวิธีแบ่งข้อมูลได้ผ่าน `scanner.Split(...)` เช่น `bufio.ScanWords` (แบ่งทีละคำ) หรือ `bufio.ScanRunes` (แบ่งทีละตัวอักษร) แต่ค่า default คือ `bufio.ScanLines` (แบ่งทีละบรรทัด) ซึ่งเป็นที่ใช้บ่อยที่สุด เราจะเรียนการปรับแต่ง `bufio.Scanner` แบบเจาะลึกยิ่งขึ้นใน **Part 049 (`bufio` ขั้นสูง)**

---

## 6. เขียนแบบ Buffered ด้วย `bufio.Writer` และความสำคัญของ `Flush()`

การเขียนไฟล์ทีละเล็กทีละน้อยด้วย `f.Write()` หรือ `f.WriteString()` ตรงๆ หลายครั้งติดกัน (เช่นใน loop) มีต้นทุนสูง เพราะแต่ละครั้งเป็นการเรียก **system call** ไปยังระบบปฏิบัติการ ซึ่งช้ากว่าการทำงานใน memory มาก `bufio.Writer` แก้ปัญหานี้ด้วยการ**สะสมข้อมูลไว้ใน buffer ใน memory ก่อน** แล้วค่อยเขียนลงไฟล์จริงเป็นก้อนใหญ่ในคราวเดียว

```go
package main

import (
	"bufio"
	"fmt"
	"os"
)

func main() {
	dir := "filedata"

	wf, err := os.Create(dir + "/buffered.txt")
	if err != nil {
		fmt.Println("create error:", err)
		return
	}
	writer := bufio.NewWriter(wf)
	for i := 1; i <= 3; i++ {
		fmt.Fprintf(writer, "รายการที่ %d\n", i)
	}
	// ถ้าลืม Flush ข้อมูลอาจยังไม่ถูกเขียนลงไฟล์จริง!
	if err := writer.Flush(); err != nil {
		fmt.Println("flush error:", err)
	}
	wf.Close()

	data, _ := os.ReadFile(dir + "/buffered.txt")
	fmt.Print(string(data))
}
```

ผลลัพธ์:

```
รายการที่ 1
รายการที่ 2
รายการที่ 3
```

### ทำไมต้อง `Flush()` เสมอ

`bufio.Writer` **ไม่รับประกันว่าข้อมูลจะถูกเขียนลงไฟล์จริงทันที** ข้อมูลที่เขียนผ่าน `writer.Write(...)` หรือ `fmt.Fprintf(writer, ...)` จะถูกเก็บไว้ใน buffer ใน memory ก่อน และจะถูก "ระบาย" (flush) ลงไฟล์จริงก็ต่อเมื่อ:

1. Buffer เต็ม (ขนาด default คือ 4096 bytes)
2. เรียก `writer.Flush()` **ด้วยตัวเอง**

**ข้อผิดพลาดที่พบบ่อยที่สุดของมือใหม่**: เขียนข้อมูลผ่าน `bufio.Writer` แล้วลืมเรียก `Flush()` ก่อนปิดโปรแกรมหรือปิดไฟล์ ทำให้ข้อมูลบางส่วน (หรือทั้งหมด ถ้าข้อมูลน้อยกว่า buffer size) **หายไปโดยไม่มี error ใดๆ แจ้งเตือน** เพราะโปรแกรมดูเหมือนทำงานสำเร็จทุกอย่าง เพียงแต่ข้อมูลยังค้างอยู่ใน buffer ตอนโปรแกรมจบการทำงาน

**กฎทอง**: ทุกครั้งที่ใช้ `bufio.Writer` ต้องเรียก `Flush()` ก่อนที่โปรแกรมจะจบการทำงานหรือก่อนปิดไฟล์เสมอ และควรเช็ค error ที่ `Flush()` คืนกลับมาด้วย (เพราะ error จากการเขียนจริงบางครั้งจะไปโผล่ตอน `Flush()` แทนที่จะโผล่ตอน `Write()`) รูปแบบที่ปลอดภัยที่สุดคือใช้ `defer` ร่วมกับการเช็ค error:

```go
writer := bufio.NewWriter(wf)
defer func() {
	if err := writer.Flush(); err != nil {
		fmt.Println("flush error:", err)
	}
}()
```

---

## 7. จัดการ error: แยกแยะไฟล์ไม่พบ vs error ประเภทอื่น

จากหลักการ error handling ที่เรียนใน **Part 015** และ **Part 016** เรารู้ว่าการเช็ค error อย่างเดียวไม่พอ บางครั้งเราต้องรู้ด้วยว่า**เป็น error ประเภทไหน** เพื่อจัดการต่างกัน เช่น "ไฟล์ไม่พบ" ควรจัดการต่างจาก "ไม่มีสิทธิ์เข้าถึง" หรือ "disk เต็ม"

```go
package main

import (
	"errors"
	"fmt"
	"os"
)

func main() {
	_, err := os.Open("filedata/not-exist.txt")
	if err != nil {
		if os.IsNotExist(err) {
			fmt.Println("แบบเก่า (os.IsNotExist): ไม่พบไฟล์ ->", err)
		}
		if errors.Is(err, os.ErrNotExist) {
			fmt.Println("แบบใหม่ (errors.Is): ไม่พบไฟล์ ->", err)
		}
	}

	_, err2 := os.ReadFile("filedata")
	if err2 != nil {
		fmt.Println("error ประเภทอื่น:", err2)
		fmt.Println("เป็น not-exist หรือไม่:", os.IsNotExist(err2))
	}
}
```

ผลลัพธ์:

```
แบบเก่า (os.IsNotExist): ไม่พบไฟล์ -> open filedata/not-exist.txt: no such file or directory
แบบใหม่ (errors.Is): ไม่พบไฟล์ -> open filedata/not-exist.txt: no such file or directory
error ประเภทอื่น: read filedata: is a directory
เป็น not-exist หรือไม่: false
```

### `os.IsNotExist` vs `errors.Is(err, os.ErrNotExist)` — ควรใช้อันไหน

ทั้งสองวิธีให้ผลลัพธ์เหมือนกันในกรณีทั่วไป แต่มีความแตกต่างที่สำคัญ:

- **`os.IsNotExist(err)`** — เป็นฟังก์ชันที่มีมาตั้งแต่ Go เวอร์ชันแรกๆ ทำงานได้ดีกับ error ที่มาจาก package `os` โดยตรง แต่ **มีข้อจำกัดกับ error ที่ถูก wrap หลายชั้น** ด้วย `fmt.Errorf("%w", ...)` (เทคนิคจาก **Part 016**)
- **`errors.Is(err, os.ErrNotExist)`** — เป็นวิธีที่**แนะนำในโค้ดปัจจุบัน** เพราะ `errors.Is` (ที่เรียนใน **Part 016**) จะไล่ตรวจสอบ error chain ทั้งหมดที่ถูก wrap ไว้ ทำให้ตรวจจับได้ถูกต้องแม้ error จะถูกห่อหลายชั้นผ่านฟังก์ชันตัวกลางหลายตัว

**คำแนะนำ**: ในโค้ดใหม่ ให้ใช้ `errors.Is(err, os.ErrNotExist)` เป็นหลัก เพราะทำงานถูกต้องแม่นยำกว่าในทุกสถานการณ์ที่มีการ wrap error ซึ่งเป็นแนวทางที่พบบ่อยมากในโค้ด Go สมัยใหม่

---

## 8. ทำงานกับไดเรกทอรี: `os.Mkdir`, `os.MkdirAll`, `os.ReadDir`

```go
package main

import (
	"fmt"
	"os"
)

func main() {
	// os.Mkdir - สร้างไดเรกทอรีเดียว (parent ต้องมีอยู่แล้ว ไม่งั้น error)
	err := os.Mkdir("filedata/subdir", 0755)
	if err != nil {
		fmt.Println("mkdir error:", err)
	} else {
		fmt.Println("สร้าง subdir สำเร็จ")
	}

	// os.ReadDir - อ่านรายการไฟล์/โฟลเดอร์ในไดเรกทอรี
	entries, err := os.ReadDir("filedata")
	if err != nil {
		fmt.Println("readdir error:", err)
		return
	}
	fmt.Println("รายการในโฟลเดอร์ filedata:")
	for _, e := range entries {
		kind := "ไฟล์"
		if e.IsDir() {
			kind = "โฟลเดอร์"
		}
		info, _ := e.Info()
		fmt.Printf("- %s (%s, %d bytes)\n", e.Name(), kind, info.Size())
	}
}
```

ผลลัพธ์ (รายการไฟล์อาจต่างกันขึ้นอยู่กับไฟล์ที่สร้างไว้ก่อนหน้าในตัวอย่างก่อนๆ):

```
สร้าง subdir สำเร็จ
รายการในโฟลเดอร์ filedata:
- buffered.txt (ไฟล์, 90 bytes)
- greeting.txt (ไฟล์, 59 bytes)
- log.txt (ไฟล์, 66 bytes)
- subdir (โฟลเดอร์, 4096 bytes)
```

### `os.Mkdir` vs `os.MkdirAll`

- **`os.Mkdir(path, perm)`** — สร้างไดเรกทอรีเดียว ถ้า parent directory ของ path นั้นยังไม่มีอยู่จะ **error ทันที**
- **`os.MkdirAll(path, perm)`** — สร้างไดเรกทอรีพร้อม parent directory ที่ขาดหายไปทั้งหมดให้ในคำสั่งเดียว (เหมือนคำสั่ง `mkdir -p` ใน Unix) และ **ไม่ error ถ้าไดเรกทอรีมีอยู่แล้ว**

ในทางปฏิบัติ `os.MkdirAll` ถูกใช้บ่อยกว่ามาก เพราะปลอดภัยกว่า (ไม่ error ถ้ามีอยู่แล้ว) และสะดวกกว่า (สร้าง parent ให้อัตโนมัติ)

### `os.ReadDir` แทนที่ `ioutil.ReadDir` แบบเก่า

`os.ReadDir` คืนค่าเป็น `[]os.DirEntry` ซึ่งแต่ละตัวมี method `Name()`, `IsDir()`, และ `Info()` (คืน `os.FileInfo` ที่มี `.Size()`, `.ModTime()`, `.Mode()` ฯลฯ) — ฟังก์ชันนี้มาแทนที่ `ioutil.ReadDir` ที่เคยใช้ในเวอร์ชันเก่าของ Go (package `io/ioutil` ถูก deprecate ไปแล้วตั้งแต่ Go 1.16 โดยฟังก์ชันทั้งหมดถูกย้ายมาอยู่ใน `os` และ `io` โดยตรง เช่น `ioutil.ReadFile` → `os.ReadFile`, `ioutil.ReadAll` → `io.ReadAll`) หากเจอโค้ดเก่าที่ยังใช้ `io/ioutil` อยู่ ควรปรับมาใช้ `os`/`io` เวอร์ชันปัจจุบันแทน

---

## 9. ปิดไฟล์ให้ถูกต้องเสมอด้วย `defer f.Close()`

ทุกครั้งที่เปิดไฟล์ด้วย `os.Open`, `os.Create`, หรือ `os.OpenFile` **ต้องปิดไฟล์เสมอ** เพื่อคืนทรัพยากรของระบบปฏิบัติการ (file descriptor) กลับคืน — ถ้าเปิดไฟล์จำนวนมากโดยไม่ปิด อาจทำให้โปรแกรม error ด้วยข้อความ "too many open files" ได้

รูปแบบมาตรฐานที่ควรใช้เสมอคือ `defer` ทันทีหลังจากเปิดไฟล์สำเร็จ (ทบทวนหลักการ `defer` จาก **Part 009**):

```go
f, err := os.Open(path)
if err != nil {
	return err
}
defer f.Close()

// ... ใช้งาน f ต่อไป ...
```

การวาง `defer f.Close()` ไว้**ทันที**หลังเช็ค error จากการเปิดไฟล์ (ไม่ใช่ปิดท้ายฟังก์ชัน) ทำให้มั่นใจได้ว่าไฟล์จะถูกปิดเสมอไม่ว่าฟังก์ชันจะ return ตรงไหนก็ตาม (จาก error ระหว่างทาง หรือ return ปกติ) — เป็น pattern ที่ Go แนะนำและใช้กันอย่างแพร่หลาย

### เมื่อไรที่ต้องเช็ค error จาก `Close()` ด้วย

ในกรณีทั่วไป การ `defer f.Close()` แล้วไม่เช็ค error ที่คืนมาก็เพียงพอ เพราะ error จากการปิดไฟล์ที่**เปิดไว้อ่านอย่างเดียว**มักไม่มีนัยสำคัญ แต่ในกรณีที่ **เขียนไฟล์** การปิดไฟล์คือจุดที่ระบบปฏิบัติการอาจ**เขียนข้อมูลที่ค้างอยู่ใน buffer ของระบบลง disk จริง** (flush ระดับ OS) ซึ่งอาจล้มเหลวได้ (เช่น disk เต็ม, disk error) ในกรณีนี้ **ควรเช็ค error จาก `Close()` เสมอ**:

```go
f, err := os.Create(path)
if err != nil {
	return err
}
defer func() {
	if cerr := f.Close(); cerr != nil {
		log.Println("ปิดไฟล์ล้มเหลว:", cerr)
	}
}()

if _, err := f.WriteString(data); err != nil {
	return err
}
```

หรือในกรณีที่ error จากการปิดไฟล์เขียนสำคัญมากถึงขั้นต้อง return กลับไปเป็น error ของฟังก์ชันด้วย สามารถใช้เทคนิค **named return value** ร่วมกับ `defer` (ทบทวนจาก **Part 009**) เพื่อ capture error จาก `Close()` เข้าไปใน return value ของฟังก์ชันได้:

```go
func writeToFile(path, data string) (err error) {
	f, err := os.Create(path)
	if err != nil {
		return err
	}
	defer func() {
		if cerr := f.Close(); cerr != nil && err == nil {
			err = cerr // ถ้ายังไม่มี error อื่นมาก่อน ให้ error จาก Close() เป็นตัวหลัก
		}
	}()

	_, err = f.WriteString(data)
	return err
}
```

**สรุปกฎปฏิบัติ**: ไฟล์ที่เปิดอ่านอย่างเดียว → `defer f.Close()` เฉยๆ ก็พอ ไฟล์ที่เขียนข้อมูลสำคัญ → ควรเช็ค error จาก `Close()` เสมอ เพื่อไม่พลาดปัญหาการเขียนที่ล้มเหลวแบบเงียบๆ

---

## สรุปสิ่งที่ได้เรียนในบทนี้

- Package `os` จัดการไฟล์ระดับระบบปฏิบัติการ, `io` กำหนด interface กลาง (`Reader`/`Writer`), `bufio` เพิ่มชั้น buffering และ utility สะดวกๆ
- งานง่ายๆ ใช้ `os.ReadFile`/`os.WriteFile` ได้เลย แต่ไฟล์ใหญ่หรือต้องการควบคุมละเอียดต้องใช้ `os.Open`/`os.Create`/`os.OpenFile`
- `io.Reader`/`io.Writer` คือ interface ที่มี method เดียว ทำให้ `*os.File` (และแหล่งข้อมูลอื่นๆ อีกมากมาย) ใช้แทนกันได้ในฟังก์ชันเดียวกัน — รายละเอียดเจาะลึกรอที่ **Part 048**
- `bufio.Scanner` อ่านทีละบรรทัดสะดวก ต้องเช็ค `scanner.Err()` หลัง loop เสมอเพื่อแยก EOF ปกติออกจาก error จริง
- `bufio.Writer` เพิ่มประสิทธิภาพการเขียน แต่ **ต้องเรียก `Flush()` เสมอ** ไม่งั้นข้อมูลอาจหายไปแบบเงียบๆ
- ใช้ `errors.Is(err, os.ErrNotExist)` แทน `os.IsNotExist(err)` ในโค้ดใหม่ เพราะรองรับ wrapped error ได้ถูกต้องกว่า
- `os.MkdirAll` ปลอดภัยกว่า `os.Mkdir` เพราะสร้าง parent directory ให้และไม่ error ถ้ามีอยู่แล้ว, `os.ReadDir` อ่านรายการไฟล์/โฟลเดอร์
- ปิดไฟล์เสมอด้วย `defer f.Close()` ทันทีหลังเปิดไฟล์สำเร็จ และเช็ค error จาก `Close()` เมื่อเป็นไฟล์ที่เขียนข้อมูลสำคัญ

## แบบฝึกหัดท้ายบท

1. เขียนโปรแกรมที่รับชื่อไฟล์และข้อความ แล้วเขียนข้อความนั้นลงไฟล์ด้วย `os.WriteFile` จากนั้นอ่านกลับมาแสดงผลด้วย `os.ReadFile`
2. เขียนโปรแกรมที่เปิดไฟล์ข้อความหนึ่งไฟล์ แล้วใช้ `bufio.Scanner` อ่านทีละบรรทัด พร้อมนับจำนวนบรรทัดทั้งหมดและจำนวนคำทั้งหมดในไฟล์
3. เขียนโปรแกรมที่ใช้ `bufio.Writer` เขียนตัวเลข 1-1000 ลงไฟล์ (แต่ละบรรทัด 1 ตัวเลข) แล้วทดลอง**ลบ**บรรทัด `Flush()` ออกดู สังเกตว่าไฟล์ที่ได้มีข้อมูลครบหรือไม่ อธิบายว่าทำไม
4. เขียนฟังก์ชัน `fileExists(path string) bool` ที่เช็คว่าไฟล์มีอยู่จริงหรือไม่ โดยใช้ `os.Stat` ร่วมกับ `errors.Is(err, os.ErrNotExist)`
5. เขียนโปรแกรมที่สร้างโฟลเดอร์ใหม่ด้วย `os.MkdirAll`, สร้างไฟล์ 3 ไฟล์ในโฟลเดอร์นั้น แล้วใช้ `os.ReadDir` แสดงรายชื่อไฟล์ทั้งหมดพร้อมขนาดไฟล์ เรียงจากไฟล์ที่มีขนาดใหญ่ไปเล็ก
6. เขียนฟังก์ชัน `appendLog(path, message string) error` ที่เปิดไฟล์ด้วย `os.OpenFile` แบบ append (สร้างไฟล์ใหม่ถ้ายังไม่มี) แล้วเขียนข้อความพร้อม timestamp ต่อท้ายไฟล์เสมอ (ใช้ความรู้เรื่อง `time` จาก **Part 022** ร่วมด้วย)

---

**ต่อไป**: [Part 025 — JSON ด้วย `encoding/json`](./025-json-encoding.md)
