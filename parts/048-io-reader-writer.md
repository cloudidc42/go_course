# Part 048: `io.Reader` / `io.Writer` และ Interface Composition

> ภาคที่ 4: Standard Library เชิงลึก — ตอนที่ 3 จาก 10 (Part 46–55)

## สารบัญของบทนี้

1. ทบทวน: `io.Reader`/`io.Writer` คือนามธรรมสากลของ Go
2. ทำไมแทบทุกอย่างใน Go ถึง implement interface นี้ได้
3. `io.Copy`: ฟังก์ชันที่ไม่สนใจว่าต้นทาง/ปลายทางเป็นอะไร
4. Interface ประกอบร่าง: `io.ReadWriter`, `io.ReadCloser`, `io.ReadWriteCloser`
5. Decorator Pattern กับ `io.Reader`: เขียน `HashingReader` เอง
6. `io.LimitReader`: จำกัดปริมาณข้อมูลที่อ่านได้
7. `io.MultiReader`: เชื่อมหลาย Reader เป็นตัวเดียว
8. `io.Pipe`: เชื่อม Writer เข้ากับ Reader แบบ in-process
9. สรุปสิ่งที่ได้เรียนในบทนี้
10. แบบฝึกหัดท้ายบท

---

## 1. ทบทวน: `io.Reader`/`io.Writer` คือนามธรรมสากลของ Go

**Part 024** แนะนำ `io.Reader`/`io.Writer` ไปแบบสั้นๆ ในฐานะกลไกเบื้องหลังของ `bufio.Scanner` และ `bufio.Writer` ส่วน **Part 046-047** ก็ใช้ทั้งสอง interface นี้ไปโดยไม่รู้ตัวหลายครั้ง (เช่น `resp.Body`, `req.Body`, `w http.ResponseWriter`) บทนี้จะพาไปเจาะลึกว่าทำไม interface เล็กๆ แค่ method เดียวสองตัวนี้ ถึงกลายเป็นรากฐานที่ยึดทั้งระบบ I/O ของ Go เอาไว้ด้วยกัน

ทบทวนนิยามอีกครั้ง:

```go
type Reader interface {
	Read(p []byte) (n int, err error)
}

type Writer interface {
	Write(p []byte) (n int, err error)
}
```

- `Read(p []byte)` — พยายามอ่านข้อมูลใส่ลงใน slice `p` ที่ผู้เรียกเตรียมมาให้ คืนจำนวน byte ที่อ่านได้จริง (`n`) และ error (ถ้าอ่านจนจบข้อมูลจะได้ `io.EOF`)
- `Write(p []byte)` — เขียนข้อมูลทั้งหมดใน `p` ออกไปยังปลายทาง คืนจำนวน byte ที่เขียนสำเร็จและ error

สังเกตว่า**ทั้งสอง interface ไม่ได้บอกอะไรเลยว่า "ข้อมูลมาจากไหน" หรือ "ข้อมูลไปที่ไหน"** — นี่คือหัวใจของการออกแบบ: มัน**นิยามแค่พฤติกรรม (behavior) ไม่ใช่การ implement (implementation)** สอดคล้องกับปรัชญา "interface เล็กที่สุดเท่าที่จำเป็น" ที่เจอซ้ำแล้วซ้ำเล่าตลอดหลักสูตรนี้ ตั้งแต่ **Part 013** (interface พื้นฐาน) จนถึง **Part 030** (interface embedding)

---

## 2. ทำไมแทบทุกอย่างใน Go ถึง implement interface นี้ได้

เพราะ `io.Reader`/`io.Writer` เรียกร้องแค่ method เดียวที่มี signature เรียบง่ายมาก แทบทุก "แหล่งข้อมูล" หรือ "ปลายทางข้อมูล" ที่มีอยู่จริงในโลกคอมพิวเตอร์จึง implement มันได้อย่างเป็นธรรมชาติ — ไม่ว่าจะเป็นไฟล์บนดิสก์, การเชื่อมต่อเครือข่าย, หน่วยความจำ, หรือแม้แต่ terminal เอง

มาดูให้เห็นภาพชัดๆ ว่าฟังก์ชันเดียวกันเป๊ะ ใช้งานได้กับแหล่งข้อมูลที่หน้าตาต่างกันโดยสิ้นเชิงทั้งหมดนี้:

```go
package main

import (
	"bytes"
	"fmt"
	"io"
	"net/http"
	"net/http/httptest"
	"os"
	"strings"
)

// printAll รับ io.Reader "ชนิดใดก็ได้" แล้วอ่านเนื้อหาทั้งหมดออกมาพิมพ์
// ฟังก์ชันนี้ "ไม่รู้และไม่สนใจ" เลยว่าข้อมูลจริงๆ มาจากไหน - นี่คือพลังของ io.Reader
func printAll(label string, r io.Reader) {
	data, err := io.ReadAll(r)
	if err != nil {
		fmt.Println(label, "error:", err)
		return
	}
	fmt.Printf("%s: %s\n", label, string(data))
}

func main() {
	// 1. strings.Reader - อ่านจาก string ใน memory
	printAll("จาก strings.Reader", strings.NewReader("ข้อความจาก string"))

	// 2. bytes.Buffer - อ่านจาก byte buffer ใน memory (จะเรียนเจาะลึกใน Part 050)
	var buf bytes.Buffer
	buf.WriteString("ข้อความจาก bytes.Buffer")
	printAll("จาก bytes.Buffer", &buf)

	// 3. os.File - อ่านจากไฟล์จริงบนดิสก์ (ทบทวนจาก Part 024)
	tmpFile, _ := os.CreateTemp("", "iodemo-*.txt")
	tmpFile.WriteString("ข้อความจากไฟล์บนดิสก์")
	tmpFile.Close()
	defer os.Remove(tmpFile.Name())
	f, _ := os.Open(tmpFile.Name())
	defer f.Close()
	printAll("จาก os.File", f)

	// 4. http response body - อ่านจาก network connection ผ่าน HTTP (ทบทวนจาก Part 046)
	server := httptest.NewServer(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		fmt.Fprint(w, "ข้อความจาก HTTP response body")
	}))
	defer server.Close()
	resp, err := http.Get(server.URL)
	if err != nil {
		fmt.Println("http get error:", err)
		return
	}
	defer resp.Body.Close()
	printAll("จาก http response body", resp.Body)

	fmt.Println()
	fmt.Println("ทั้งหมดนี้ผ่านฟังก์ชันเดียวกันคือ printAll(label string, r io.Reader)")
	fmt.Println("โดยที่ printAll ไม่มีโค้ดแยกเคสตามชนิดข้อมูลต้นทางเลยแม้แต่บรรทัดเดียว")
}
```

ผลลัพธ์:

```
จาก strings.Reader: ข้อความจาก string
จาก bytes.Buffer: ข้อความจาก bytes.Buffer
จาก os.File: ข้อความจากไฟล์บนดิสก์
จาก http response body: ข้อความจาก HTTP response body

ทั้งหมดนี้ผ่านฟังก์ชันเดียวกันคือ printAll(label string, r io.Reader)
โดยที่ printAll ไม่มีโค้ดแยกเคสตามชนิดข้อมูลต้นทางเลยแม้แต่บรรทัดเดียว
```

### ตารางสรุป: ใครเป็น `io.Reader`/`io.Writer` บ้าง

| Type | เป็น `io.Reader` | เป็น `io.Writer` | เรียนไปแล้วใน |
|---|---|---|---|
| `*os.File` | ใช่ | ใช่ | Part 024 |
| `*strings.Reader` | ใช่ | ไม่ | Part 019 |
| `*bytes.Buffer` | ใช่ | ใช่ | Part 050 |
| `*bytes.Reader` | ใช่ | ไม่ | Part 050 |
| `*strings.Builder` | ไม่ | ใช่ (ผ่าน `WriteString` ไม่ใช่ `Write` ตรงๆ — จริงๆ implement `io.Writer` ด้วยเช่นกัน) | Part 019 |
| `net.Conn` (TCP/network connection) | ใช่ | ใช่ | ภาคที่ 5-6 |
| `http.Request.Body`/`http.Response.Body` | ใช่ (เป็น `io.ReadCloser`) | ไม่ | Part 046-047 |
| `http.ResponseWriter` | ไม่ | ใช่ | Part 047 |
| `os.Stdin` | ใช่ | ไม่ | Part 024 |
| `os.Stdout`/`os.Stderr` | ไม่ | ใช่ | Part 024 |
| `gzip.Reader`/`gzip.Writer` | ใช่/ใช่ | — | — |

**ข้อสังเกตสำคัญ**: การออกแบบฟังก์ชันหรือ package ใดๆ ให้รับพารามิเตอร์เป็น `io.Reader`/`io.Writer` แทนที่จะรับ type รูปธรรม (เช่น `*os.File` ตรงๆ) ทำให้โค้ดนั้น **ใช้ได้กับทุกแถวในตารางข้างบนโดยอัตโนมัติ** โดยที่ผู้เขียนโค้ดนั้นไม่จำเป็นต้องรู้จักหรือ import type เหล่านั้นเลยด้วยซ้ำ — นี่คือเหตุผลที่ Go เขียน parser, encoder, compression algorithm ฯลฯ ให้ทำงานกับ `io.Reader`/`io.Writer` เสมอ แทนที่จะผูกกับไฟล์หรือ network โดยตรง

---

## 3. `io.Copy`: ฟังก์ชันที่ไม่สนใจว่าต้นทาง/ปลายทางเป็นอะไร

`io.Copy(dst io.Writer, src io.Reader) (int64, error)` คือตัวอย่างที่ดีที่สุดของพลัง interface ทั้งสองตัวนี้ทำงานร่วมกัน — มันอ่านจาก `src` แล้วเขียนไปยัง `dst` ทีละก้อนจนกว่า `src` จะให้ `io.EOF` โดยไม่สนใจเลยว่าเบื้องหลังทั้งสองฝั่งเป็นอะไร

```go
package main

import (
	"bytes"
	"fmt"
	"io"
	"os"
	"strings"
)

func main() {
	src := strings.NewReader("ข้อมูลต้นทางเดียวกัน นำไปเขียนได้หลายปลายทาง")

	// io.Copy(dst io.Writer, src io.Reader) (int64, error)
	// คัดลอกข้อมูลจาก src ไปยัง dst ทีละ buffer จนกว่า src จะอ่านหมด (EOF)
	// ทั้ง dst และ src เป็น interface ล้วนๆ - io.Copy ไม่สนใจว่าเบื้องหลังเป็นอะไร

	// 1. คัดลอกไปยัง os.Stdout ตรงๆ
	fmt.Print("ปลายทางที่ 1 (stdout): ")
	n1, err := io.Copy(os.Stdout, src)
	fmt.Println()
	fmt.Println("เขียนไป", n1, "bytes, err:", err)

	// ต้อง reset src กลับไปตำแหน่งเริ่มต้นก่อน เพราะ io.Copy อ่านจนหมดไปแล้วในรอบแรก
	src.Seek(0, io.SeekStart)

	// 2. คัดลอกไปยัง bytes.Buffer ใน memory
	var buf bytes.Buffer
	n2, err := io.Copy(&buf, src)
	fmt.Println("ปลายทางที่ 2 (bytes.Buffer):", buf.String())
	fmt.Println("เขียนไป", n2, "bytes, err:", err)

	src.Seek(0, io.SeekStart)

	// 3. คัดลอกไปยังไฟล์จริงบนดิสก์
	tmpFile, _ := os.CreateTemp("", "iocopy-*.txt")
	defer os.Remove(tmpFile.Name())
	n3, err := io.Copy(tmpFile, src)
	tmpFile.Close()
	fmt.Println("เขียนไปไฟล์", n3, "bytes, err:", err)

	fileContent, _ := os.ReadFile(tmpFile.Name())
	fmt.Println("ปลายทางที่ 3 (ไฟล์):", string(fileContent))

	fmt.Println()
	fmt.Println("สังเกต: โค้ดที่เรียก io.Copy ทั้งสามครั้งเหมือนกันทุกประการในเชิงโครงสร้าง")
	fmt.Println("ต่างกันแค่ argument ตัวแรก (destination) เท่านั้น เพราะทุกปลายทางเป็น io.Writer")
}
```

ผลลัพธ์:

```
ปลายทางที่ 1 (stdout): ข้อมูลต้นทางเดียวกัน นำไปเขียนได้หลายปลายทาง
เขียนไป 130 bytes, err: <nil>
ปลายทางที่ 2 (bytes.Buffer): ข้อมูลต้นทางเดียวกัน นำไปเขียนได้หลายปลายทาง
เขียนไป 130 bytes, err: <nil>
เขียนไปไฟล์ 130 bytes, err: <nil>
ปลายทางที่ 3 (ไฟล์): ข้อมูลต้นทางเดียวกัน นำไปเขียนได้หลายปลายทาง

สังเกต: โค้ดที่เรียก io.Copy ทั้งสามครั้งเหมือนกันทุกประการในเชิงโครงสร้าง
ต่างกันแค่ argument ตัวแรก (destination) เท่านั้น เพราะทุกปลายทางเป็น io.Writer
```

### ทำไม `io.Copy` ถึงสำคัญกว่าที่คิด

`io.Copy` ไม่ได้เป็นแค่ความสะดวก — มันมีข้อดีด้าน**ประสิทธิภาพหน่วยความจำ**ที่สำคัญมาก: มันอ่าน-เขียนข้อมูล**ทีละ buffer เล็กๆ** (ปกติ 32KB) แทนที่จะโหลดข้อมูลทั้งก้อนเข้า memory ก่อนด้วย `io.ReadAll` แล้วค่อยเขียนออกทีเดียว หมายความว่าเวลาคัดลอกไฟล์ขนาดหลาย GB หรือ stream ข้อมูลจาก network ที่ไม่รู้ขนาดล่วงหน้า `io.Copy` ใช้ memory คงที่เสมอไม่ว่าข้อมูลจะใหญ่แค่ไหน — ต่างจากการเขียน `dst.Write(io.ReadAll(src))` ที่ต้องมี memory เพียงพอสำหรับข้อมูลทั้งก้อนพร้อมกัน

ในบางกรณี ถ้าทั้ง `src` และ `dst` implement interface พิเศษที่ชื่อ `io.ReaderFrom` หรือ `io.WriterTo` เพิ่มเติม `io.Copy` ยังฉลาดพอที่จะ**ข้ามการคัดลอกผ่าน buffer กลางไปเลย** (เช่น กรณีคัดลอกระหว่างไฟล์บน Linux ที่ใช้ system call `sendfile` ได้โดยตรง) นี่คือตัวอย่างที่ดีของการที่ Go ออกแบบให้ interface เล็กๆ ประกอบกันเป็น optimization ที่ซับซ้อนกว่าได้โดยผู้ใช้งานทั่วไปไม่ต้องรู้รายละเอียดเหล่านี้เลย

---

## 4. Interface ประกอบร่าง: `io.ReadWriter`, `io.ReadCloser`, `io.ReadWriteCloser`

`Part 030` สอนเรื่อง **interface embedding** ไปแล้ว — package `io` เป็นตัวอย่างที่ใช้เทคนิคนี้อย่างเป็นระบบที่สุดใน standard library ทั้งหมด โดยประกอบ interface เล็กๆ (`Reader`, `Writer`, `Closer`, `Seeker`) เข้าด้วยกันเป็น interface ที่ใหญ่ขึ้นตามต้องการ:

```go
type Closer interface {
	Close() error
}

// ReadWriter ประกอบร่างจาก Reader + Writer
type ReadWriter interface {
	Reader
	Writer
}

// ReadCloser ประกอบร่างจาก Reader + Closer
type ReadCloser interface {
	Reader
	Closer
}

// ReadWriteCloser ประกอบร่างจากทั้งสาม
type ReadWriteCloser interface {
	Reader
	Writer
	Closer
}
```

ประโยชน์ของการแยก interface ย่อยแล้วประกอบร่างแบบนี้คือ **ฟังก์ชันแต่ละตัวขอ (require) ความสามารถแค่เท่าที่ต้องใช้จริง** — ฟังก์ชันที่แค่ต้องการอ่านข้อมูลแล้วปิดเมื่อจบ (pattern ที่พบบ่อยที่สุด) ควรรับ `io.ReadCloser` ไม่ใช่รับ `*os.File` ทั้งก้อน (ที่มี method อื่นอีกมากมายที่ไม่เกี่ยวข้องเลย เช่น `Chmod`, `Stat`) ทำให้ฟังก์ชันนั้นใช้ได้กับทุกอย่างที่ implement แค่สองความสามารถนี้ ไม่ว่าจะเป็นไฟล์จริง หรือ HTTP response body

```go
package main

import (
	"bytes"
	"fmt"
	"io"
	"os"
)

// closeAfterReading รับ io.ReadCloser ใดๆ ก็ได้ อ่านเนื้อหาทั้งหมดแล้วปิดให้เสมอ
// ใช้ได้กับทั้ง *os.File, resp.Body ของ HTTP response, หรือ type อื่นที่ผสาน Read+Close เข้าด้วยกัน
func closeAfterReading(rc io.ReadCloser) (string, error) {
	defer rc.Close()
	data, err := io.ReadAll(rc)
	return string(data), err
}

// echoBuffer รับ io.ReadWriter (อ่านได้และเขียนได้ในตัวเดียว) เขียนข้อความลงไปแล้วอ่านกลับออกมาทันที
func echoBuffer(rw io.ReadWriter, message string) string {
	rw.Write([]byte(message))
	data, _ := io.ReadAll(rw)
	return string(data)
}

func main() {
	// bytes.Buffer implement ทั้ง io.Reader และ io.Writer พร้อมกัน จึงเป็น io.ReadWriter ได้ในตัว
	// (ต่างจาก strings.Builder ที่เขียนได้อย่างเดียว - จะเปรียบเทียบละเอียดใน Part 050)
	var buf bytes.Buffer
	result := echoBuffer(&buf, "ข้อความทดสอบ echo")
	fmt.Println("echoBuffer ผลลัพธ์:", result)

	// io.ReadWriteCloser คือ interface ที่ประกอบร่างทั้งสาม (Read + Write + Close)
	// *os.File เป็นตัวอย่างที่ implement ครบทั้งสาม method จึงเป็นทั้ง Reader, Writer, และ Closer พร้อมกัน
	var _ io.ReadWriteCloser = (*os.File)(nil)
	fmt.Println("*os.File สามารถใช้แทน io.ReadWriteCloser ได้ (compile ผ่านแสดงว่า implement ครบ)")

	// io.ReadCloser: ใช้กับไฟล์
	tmpFile, _ := os.CreateTemp("", "readcloser-*.txt")
	tmpFile.WriteString("เนื้อหาในไฟล์ทดสอบ")
	tmpFile.Close()
	defer os.Remove(tmpFile.Name())

	f, _ := os.Open(tmpFile.Name())
	content, err := closeAfterReading(f) // f (*os.File) implement io.ReadCloser จึงส่งเข้าได้ตรงๆ
	fmt.Println("closeAfterReading จากไฟล์:", content, "err:", err)
}
```

ผลลัพธ์:

```
echoBuffer ผลลัพธ์: ข้อความทดสอบ echo
*os.File สามารถใช้แทน io.ReadWriteCloser ได้ (compile ผ่านแสดงว่า implement ครบ)
closeAfterReading จากไฟล์: เนื้อหาในไฟล์ทดสอบ err: <nil>
```

### เทคนิค `var _ InterfaceType = (*ConcreteType)(nil)`

บรรทัด `var _ io.ReadWriteCloser = (*os.File)(nil)` เป็นสำนวนที่พบบ่อยมากในโค้ด Go เพื่อ**ยืนยันตอน compile time** ว่า type หนึ่งๆ implement interface ที่ต้องการจริง โดยไม่ต้องสร้างตัวแปรใช้งานจริง (ตัวแปร `_` คือ blank identifier ทบทวนจาก **Part 003**) ถ้า `*os.File` ไม่ได้ implement `io.ReadWriteCloser` ครบทุก method บรรทัดนี้จะ**compile ไม่ผ่านทันที** — เป็นวิธี "เขียนเทสต์แบบ compile-time" ที่ประหยัดและได้ผลลัพธ์ทันที

`http.Request.Body` และ `http.Response.Body` (ที่เจอไปแล้วใน **Part 046-047**) มี type เป็น `io.ReadCloser` ตรงๆ — ตอนนี้เข้าใจแล้วว่าทำไม: มันคือการประกาศชัดเจนว่า "รับประกันว่าอ่านได้และปิดได้ ไม่รับประกันอะไรมากกว่านั้น" ซึ่งเพียงพอสำหรับงานส่วนใหญ่ที่เกี่ยวกับ HTTP body แล้ว

---

## 5. Decorator Pattern กับ `io.Reader`: เขียน `HashingReader` เอง

**Part 030 หัวข้อ 8** สอนเทคนิค **decorator pattern** ผ่านการ embed `io.Reader` เข้าไปใน struct แล้วเขียนตัวอย่าง `CountingReader` ที่นับจำนวน byte ที่อ่านผ่านไป — บทนี้จะขยายเทคนิคเดียวกัน แต่เปลี่ยนงานที่ทำระหว่างทางจาก "นับจำนวน" เป็น **"คำนวณ hash"** ซึ่งเป็นประโยชน์ใช้งานจริงที่พบบ่อยมาก เช่น การคำนวณ checksum ของไฟล์ขนาดใหญ่**ระหว่างที่กำลัง stream มันไปที่อื่น** โดยไม่ต้องอ่านซ้ำสองรอบ

```go
package main

import (
	"crypto/sha256"
	"encoding/hex"
	"fmt"
	"hash"
	"io"
	"strings"
)

// HashingReader ห่อ (wrap) io.Reader ตัวใดก็ได้ แล้วคำนวณ hash ของข้อมูลที่ไหลผ่าน
// "ระหว่างทาง" ที่มันถูกอ่าน โดยไม่ต้องอ่านข้อมูลทั้งหมดเข้า memory ก่อนแล้วค่อย hash
// pattern นี้คือ "decorator" เดียวกับ CountingReader ที่เรียนไปแล้วใน Part 030 หัวข้อ 8
// ต่างกันตรงที่ตัวนี้ทำงานเพิ่มเติม (hash) แทนที่จะแค่นับจำนวน byte
type HashingReader struct {
	r io.Reader
	h hash.Hash
}

func NewHashingReader(r io.Reader) *HashingReader {
	return &HashingReader{r: r, h: sha256.New()}
}

// Read implement io.Reader โดยการอ่านจาก reader ข้างในตามปกติ แล้ว "แอบ" ป้อนข้อมูล
// ที่อ่านได้เข้า hash ไปพร้อมกันทุกครั้งก่อนคืนค่ากลับไปให้ผู้เรียก
func (hr *HashingReader) Read(p []byte) (int, error) {
	n, err := hr.r.Read(p)
	if n > 0 {
		hr.h.Write(p[:n]) // hash.Hash เป็น io.Writer ด้วย จึงเขียนข้อมูลเข้าไปแบบนี้ได้ตรงๆ
	}
	return n, err
}

// Sum256Hex คืนค่า hash สะสมทั้งหมด ณ ตอนนี้ในรูปแบบ hex string
func (hr *HashingReader) Sum256Hex() string {
	return hex.EncodeToString(hr.h.Sum(nil))
}

func main() {
	src := strings.NewReader("ข้อมูลที่ต้องการทั้ง stream ผ่านไปยังปลายทาง และคำนวณ hash ไปพร้อมกัน")

	hr := NewHashingReader(src)

	// ฟังก์ชันใดๆ ที่รับ io.Reader ใช้ hr แทนได้ทันที เพราะ HashingReader implement io.Reader
	// (ในที่นี้ใช้ io.Copy คัดลอกไปยัง io.Discard คือ "อ่านทิ้ง" แต่ hash ยังคำนวณไปด้วยระหว่างทาง)
	n, err := io.Copy(io.Discard, hr)
	fmt.Println("อ่านไปทั้งหมด", n, "bytes, err:", err)
	fmt.Println("SHA-256:", hr.Sum256Hex())

	// พิสูจน์ว่า hash ตรงกับการคำนวณตรงๆ ด้วย sha256 package
	expected := sha256.Sum256([]byte("ข้อมูลที่ต้องการทั้ง stream ผ่านไปยังปลายทาง และคำนวณ hash ไปพร้อมกัน"))
	fmt.Println("ตรงกับค่าที่คำนวณตรงๆ หรือไม่:", hr.Sum256Hex() == hex.EncodeToString(expected[:]))

	fmt.Println()
	fmt.Println("=== ทางเลือกอื่น: io.TeeReader (มีให้ใน standard library แล้ว) ===")
	// io.TeeReader(r, w) คืน io.Reader ตัวใหม่ที่ทุกครั้งที่ถูกอ่าน จะ "แอบ" เขียนข้อมูล
	// ชุดเดียวกันไปยัง w ด้วย - แก้ปัญหาเดียวกับ HashingReader แต่ไม่ต้องเขียน struct เอง
	src2 := strings.NewReader("ข้อความสำหรับสาธิต TeeReader")
	h2 := sha256.New()
	tee := io.TeeReader(src2, h2) // อ่านจาก tee = อ่านจาก src2 ไปพร้อมกับ "แตกสำเนา" เข้า h2

	io.Copy(io.Discard, tee)
	fmt.Println("SHA-256 ผ่าน TeeReader:", hex.EncodeToString(h2.Sum(nil)))
}
```

ผลลัพธ์:

```
อ่านไปทั้งหมด 177 bytes, err: <nil>
SHA-256: d2fba7d20bbe9c65bf0e3bdc252109abcad6401dc3bb77491852f7d2055b81f8
ตรงกับค่าที่คำนวณตรงๆ หรือไม่: true

=== ทางเลือกอื่น: io.TeeReader (มีให้ใน standard library แล้ว) ===
SHA-256 ผ่าน TeeReader: cc3735ec617bbd72531b20c53449408d4f7b4bf0a1b81284c724bc976b6f4519
```

### เจาะลึกเทคนิค

- **`hash.Hash`** (จาก package `hash`) คือ interface ที่**ประกอบร่างจาก `io.Writer` เพิ่มเติม** — ทุก hash algorithm ใน Go (`sha256.New()`, `md5.New()`, `crc32.New()` ฯลฯ) implement `io.Writer` เพื่อรับข้อมูลเข้าไปสะสม hash แล้วมี method `Sum([]byte) []byte` สำหรับดึงผลลัพธ์สุดท้ายออกมา (จะเจาะลึก hashing เต็มรูปแบบใน **Part 051**)
- **`HashingReader.Read`** ทำสิ่งเดียวกับที่ `CountingReader.Read` ใน **Part 030** ทำ: เรียก method ของ reader ข้างในตัวจริงก่อน แล้ว "แอบทำงานเพิ่ม" กับข้อมูลที่อ่านได้ ก่อนคืนค่ากลับไปให้ผู้เรียกเหมือนไม่มีอะไรเกิดขึ้น — ผู้เรียกที่ใช้ `io.Copy(io.Discard, hr)` ไม่รู้เลยว่ามีการคำนวณ hash เกิดขึ้น "แอบแฝง" อยู่เบื้องหลัง
- **`io.TeeReader(r, w)`** คือฟังก์ชันสำเร็จรูปใน standard library ที่ทำสิ่งเดียวกับ `HashingReader` แบบทั่วไปกว่า: มันคืน `io.Reader` ที่ทุกครั้งที่ถูกอ่าน จะเขียนข้อมูลชุดเดียวกันไปยัง `io.Writer` ที่ระบุด้วย (ชื่อ "Tee" มาจากสัญลักษณ์ตัว T ที่แยกท่อน้ำออกเป็นสองทาง) — ในทางปฏิบัติ ถ้างานที่ต้องการทำแค่ "แอบบันทึกสำเนาไปยัง Writer อื่นระหว่างอ่าน" `io.TeeReader` สะดวกกว่าเขียน decorator เอง แต่ถ้าต้องการ logic ที่ซับซ้อนกว่านั้น (เช่น นับจำนวน + จำกัดอัตราการอ่าน + validate ข้อมูลไปพร้อมกัน) การเขียน decorator type เองแบบ `HashingReader`/`CountingReader` จะยืดหยุ่นกว่า

**เมื่อไรควรเขียน decorator เอง เมื่อไรควรใช้ตัวช่วยจาก `io` ที่มีให้แล้ว**: ตรวจสอบก่อนเสมอว่า package `io` มีฟังก์ชันสำเร็จรูปที่ตรงกับความต้องการหรือไม่ (`io.TeeReader`, `io.LimitReader`, `io.MultiReader` ที่จะเรียนในหัวข้อถัดไป) — เขียน decorator เองเมื่อ logic ที่ต้องการมีความเฉพาะทางเกินกว่าที่ standard library เตรียมไว้ให้เท่านั้น

---

## 6. `io.LimitReader`: จำกัดปริมาณข้อมูลที่อ่านได้

`io.LimitReader(r io.Reader, n int64) io.Reader` คืน `io.Reader` ตัวใหม่ที่ "ห่อ" ต้นทางเดิมไว้ แต่จะรายงาน `io.EOF` ทันทีเมื่ออ่านครบ `n` byte แล้ว แม้ต้นทางจริงจะยังมีข้อมูลเหลืออยู่อีกมากก็ตาม:

```go
package main

import (
	"fmt"
	"io"
	"strings"
)

func main() {
	fmt.Println("=== io.LimitReader: จำกัดจำนวน byte สูงสุดที่จะอ่านได้ ===")
	src := strings.NewReader("ข้อความยาวมากที่เราต้องการอ่านแค่บางส่วนแรกเท่านั้น ส่วนที่เหลือไม่สนใจเลย")

	// io.LimitReader(r, n) คืน io.Reader ตัวใหม่ที่อ่านได้ "ไม่เกิน n byte" แล้วจะรายงาน EOF ทันที
	// แม้ต้นทางจริง (src) จะยังมีข้อมูลเหลืออยู่อีกมากก็ตาม - มีประโยชน์มากเวลาต้องการอ่านแค่
	// "หัวไฟล์" (เช่น ตรวจสอบ magic byte ของไฟล์) โดยไม่ต้องโหลดข้อมูลทั้งก้อนเข้า memory
	limited := io.LimitReader(src, 30)

	data, err := io.ReadAll(limited)
	fmt.Printf("อ่านได้ %d bytes: %q\n", len(data), string(data))
	fmt.Println("err:", err)

	fmt.Println()
	fmt.Println("=== ประโยชน์จริง: ป้องกัน request body ขนาดใหญ่เกินไปทำร้าย server ===")
	// สถานการณ์จริงที่พบบ่อยมากในการเขียน HTTP server (ทบทวนจาก Part 047): ถ้าไม่จำกัดขนาด
	// r.Body ที่อ่าน ผู้ใช้ที่ไม่หวังดีอาจส่ง body ขนาดหลาย GB มาทำให้ server หมด memory
	hugeInput := strings.NewReader(strings.Repeat("A", 10_000_000)) // จำลอง body ขนาด 10 ล้านตัวอักษร
	const maxBodySize = 1024                                        // อนุญาตแค่ 1024 byte เท่านั้น

	safeReader := io.LimitReader(hugeInput, maxBodySize)
	safeData, _ := io.ReadAll(safeReader)
	fmt.Printf("แม้ input จริงมี 10,000,000 bytes แต่ปลอดภัยเพราะอ่านได้แค่ %d bytes\n", len(safeData))
}
```

ผลลัพธ์:

```
=== io.LimitReader: จำกัดจำนวน byte สูงสุดที่จะอ่านได้ ===
อ่านได้ 30 bytes: "ข้อความยาว"
err: <nil>

=== ประโยชน์จริง: ป้องกัน request body ขนาดใหญ่เกินไปทำร้าย server ===
แม้ input จริงมี 10,000,000 bytes แต่ปลอดภัยเพราะอ่านได้แค่ 1024 bytes
```

### ทำไมสำคัญ: ป้องกัน Denial-of-Service ผ่าน Body ขนาดใหญ่

การใช้ `io.LimitReader` ห่อ `r.Body` ก่อนอ่าน (หรือใช้ `http.MaxBytesReader` ซึ่งเป็นเวอร์ชันเฉพาะทางสำหรับ HTTP server ที่ทำสิ่งเดียวกันแต่คืน error ที่ชัดเจนกว่าเมื่อเกินขนาด) เป็นแนวทางป้องกันความปลอดภัยพื้นฐานที่ **HTTP server ทุกตัวควรมี** — ไม่ว่า client จะพยายามส่ง body ขนาดใหญ่แค่ไหน server จะอ่านได้ไม่เกินขนาดที่กำหนดไว้เสมอ ป้องกันการโจมตีแบบส่ง payload ขนาดมหาศาลมาทำให้ server หมด memory (จะกลับมาใช้เทคนิคนี้จริงจังใน **Part 106: Security Best Practices**)

---

## 7. `io.MultiReader`: เชื่อมหลาย Reader เป็นตัวเดียว

`io.MultiReader(readers ...io.Reader) io.Reader` เชื่อม `io.Reader` หลายตัวเข้าด้วยกันให้ทำงานเหมือนเป็น stream เดียวต่อเนื่องกัน โดยอ่านตัวแรกจนหมด (`EOF`) ก่อนขยับไปอ่านตัวถัดไปโดยอัตโนมัติ:

```go
package main

import (
	"fmt"
	"io"
	"strings"
)

func main() {
	header := strings.NewReader("=== HEADER ===\n")
	body := strings.NewReader("เนื้อหาหลักของเอกสาร\n")
	footer := strings.NewReader("=== FOOTER ===\n")

	// io.MultiReader เชื่อม io.Reader หลายตัวเข้าด้วยกันเป็นตัวเดียว โดยอ่านไล่ตามลำดับ
	// เมื่อตัวแรกอ่านจนหมด (EOF) จะขยับไปอ่านตัวถัดไปอัตโนมัติ จนกว่าจะครบทุกตัว
	// มีประโยชน์มากเวลาต้องการ "เชื่อมข้อมูลหลายส่วน" เข้าด้วยกันโดยไม่ต้องคัดลอกมารวมกันเองก่อน
	combined := io.MultiReader(header, body, footer)

	data, err := io.ReadAll(combined)
	fmt.Print(string(data))
	fmt.Println("err:", err)

	fmt.Println("--- เทียบกับการต่อ string เองแล้วค่อยสร้าง Reader ---")
	fmt.Println("io.MultiReader ประหยัดกว่าเพราะไม่ต้องคัดลอกข้อมูลทั้งสามส่วนมารวมเป็นก้อนเดียวก่อน")
	fmt.Println("โดยเฉพาะเมื่อแต่ละส่วนมีขนาดใหญ่ (เช่น ต่อไฟล์หลายไฟล์เข้าด้วยกันแบบ streaming)")
}
```

ผลลัพธ์:

```
=== HEADER ===
เนื้อหาหลักของเอกสาร
=== FOOTER ===
err: <nil>
--- เทียบกับการต่อ string เองแล้วค่อยสร้าง Reader ---
io.MultiReader ประหยัดกว่าเพราะไม่ต้องคัดลอกข้อมูลทั้งสามส่วนมารวมเป็นก้อนเดียวก่อน
โดยเฉพาะเมื่อแต่ละส่วนมีขนาดใหญ่ (เช่น ต่อไฟล์หลายไฟล์เข้าด้วยกันแบบ streaming)
```

**กรณีใช้งานจริงที่พบบ่อย**: ต่อ HTTP request body เข้ากับข้อมูลที่อ่านไปแล้วบางส่วน (เช่น เผลออ่าน header ไปแล้วบางส่วนตอน validate แล้วต้องการ "คืน" ข้อมูลนั้นกลับไปรวมกับส่วนที่เหลือ), เชื่อมไฟล์หลายไฟล์เข้าด้วยกันเป็น stream เดียวโดยไม่ต้องโหลดทุกไฟล์เข้า memory มารวมกันก่อน, หรือสร้าง mock data สำหรับทดสอบที่ประกอบจากหลายส่วน

---

## 8. `io.Pipe`: เชื่อม Writer เข้ากับ Reader แบบ in-process

`io.Pipe()` คืนค่าคู่ `(*io.PipeReader, *io.PipeWriter)` ที่ผูกกันแบบ **in-memory synchronous pipe**: ข้อมูลที่เขียนเข้า `PipeWriter` จะถูกส่งตรงไปให้ `PipeReader` อ่านได้ทันที **โดยไม่มี buffer กลางเก็บข้อมูลไว้เลย** — คล้าย unbuffered channel ที่เรียนใน **Part 037** แต่ทำงานผ่าน interface `io.Reader`/`io.Writer` แทนที่จะส่งค่าผ่าน channel ตรงๆ

ประโยชน์หลักคือทำให้เขียนโค้ดสองฝั่งที่ "คุยกันแบบ streaming" ได้ โดยฝั่งหนึ่งมองว่ากำลังเขียนไฟล์ปกติ อีกฝั่งมองว่ากำลังอ่านไฟล์ปกติ ทั้งที่จริงๆ ไม่มีไฟล์หรือ buffer ตรงกลางเกิดขึ้นเลย:

```go
package main

import (
	"encoding/json"
	"fmt"
	"io"
)

func main() {
	// io.Pipe() คืน (*io.PipeReader, *io.PipeWriter) ที่ผูกกันแบบ in-memory: ข้อมูลที่เขียนเข้า
	// PipeWriter จะถูกส่งตรงไปให้ PipeReader อ่านได้ทันที "โดยไม่ผ่าน buffer กลาง" ต่างจาก channel
	// ที่ส่งค่าเป็นก้อนๆ - io.Pipe ทำให้ฝั่งเขียนกับฝั่งอ่าน "คุยกันแบบ streaming" ผ่าน io.Reader/io.Writer
	// ธรรมดา ราวกับกำลังเขียน/อ่านไฟล์จริง ทั้งที่จริงๆ ไม่มีไฟล์หรือ buffer ตรงกลางเลย
	pr, pw := io.Pipe()

	// ต้องเขียนใน goroutine แยกเสมอ เพราะ PipeWriter.Write บล็อกจนกว่าจะมีฝั่งอ่านมารับข้อมูลนั้นไป
	// (unbuffered โดยธรรมชาติ คล้าย unbuffered channel ที่เรียนใน Part 037)
	go func() {
		defer pw.Close() // ต้องปิดเสมอเพื่อให้ฝั่งอ่านรู้ว่าจบข้อมูลแล้ว (จะได้ io.EOF)

		// เขียน JSON ตรงลง pw โดยไม่ต้องสร้าง []byte ก้อนใหญ่มาพักไว้ก่อนเลย
		encoder := json.NewEncoder(pw)
		for i := 1; i <= 3; i++ {
			record := map[string]int{"record_id": i, "value": i * 100}
			if err := encoder.Encode(record); err != nil {
				pw.CloseWithError(err) // ส่ง error ไปให้ฝั่งอ่านรับรู้แทนการปิดเฉยๆ
				return
			}
		}
	}()

	// ฝั่งอ่านใช้ pr เป็น io.Reader ธรรมดา - ไม่รู้เลยว่าข้อมูลมาจาก goroutine อื่นแบบ real-time
	decoder := json.NewDecoder(pr)
	for {
		var record map[string]int
		if err := decoder.Decode(&record); err != nil {
			if err == io.EOF {
				break
			}
			fmt.Println("decode error:", err)
			break
		}
		fmt.Printf("อ่านได้: record_id=%d value=%d\n", record["record_id"], record["value"])
	}

	fmt.Println("จบการอ่านทั้งหมดแล้ว (ไม่มี buffer กลาง ไม่มีไฟล์ชั่วคราวเกิดขึ้นเลย)")
}
```

ผลลัพธ์:

```
อ่านได้: record_id=1 value=100
อ่านได้: record_id=2 value=200
อ่านได้: record_id=3 value=300
จบการอ่านทั้งหมดแล้ว (ไม่มี buffer กลาง ไม่มีไฟล์ชั่วคราวเกิดขึ้นเลย)
```

### กฎสำคัญของ `io.Pipe`

1. **ต้องเขียนใน goroutine แยกเสมอ** — `PipeWriter.Write` เป็น **blocking call** ที่จะรอจนกว่าจะมีฝั่งอ่านมาอ่านข้อมูลนั้นไปพอดี (เหมือน unbuffered channel) ถ้าเขียนและอ่านอยู่ใน goroutine เดียวกันแบบลำดับปกติ โปรแกรมจะ **deadlock ทันที** เพราะฝั่งเขียนรอฝั่งอ่าน แต่ฝั่งอ่านก็ยังไม่ถูกเรียกเพราะโค้ดยังไม่ผ่านบรรทัดเขียนไปได้
2. **ต้อง `Close()` ฝั่งเขียนเสมอเมื่อเขียนจบ** — ฝั่งอ่านจะไม่มีทางรู้ว่า "ข้อมูลจบแล้ว" (ได้ `io.EOF`) จนกว่า `PipeWriter.Close()` จะถูกเรียก
3. **ใช้ `CloseWithError(err)` แทน `Close()` เฉยๆ ถ้าต้องการรายงาน error ไปให้ฝั่งอ่านรู้** — ฝั่งอ่านจะได้ `err` ตัวนั้นกลับมาจาก `Read()` แทนที่จะได้ `io.EOF` ธรรมดา

**กรณีใช้งานจริงที่พบบ่อยที่สุด**: เชื่อมฟังก์ชันที่ **เขียนออกทาง `io.Writer`** (เช่น `json.NewEncoder(w).Encode(...)`, `gzip.NewWriter(w)`, หรือ template engine ที่ render ผ่าน `io.Writer`) เข้ากับฟังก์ชันอีกตัวที่ **ต้องการ `io.Reader`** เป็น input (เช่น `http.NewRequest(method, url, body io.Reader)` ตอนอัปโหลดไฟล์แบบ streaming โดยไม่ต้องสร้างไฟล์ชั่วคราวหรือ buffer ทั้งก้อนไว้ใน memory ก่อน) — `io.Pipe` คือ "ตัวเชื่อม" ที่ทำให้ทั้งสองโลกนี้ (เขียนแบบ push, อ่านแบบ pull) มาเจอกันได้อย่างมีประสิทธิภาพ

---

## สรุปสิ่งที่ได้เรียนในบทนี้

- `io.Reader`/`io.Writer` เป็น interface ที่มี method เดียว ทำให้แทบทุก type ที่เกี่ยวข้องกับข้อมูลใน Go implement มันได้ (ไฟล์, network connection, memory buffer, HTTP body) — โค้ดที่รับ interface เหล่านี้แทน type รูปธรรมจึงใช้ได้กับทุกแหล่งข้อมูลโดยไม่ต้องแก้โค้ด
- `io.Copy` คัดลอกข้อมูลระหว่าง `io.Writer`/`io.Reader` ใดๆ ก็ได้ทีละ buffer เล็กๆ ทำให้ใช้ memory คงที่แม้ข้อมูลจะมีขนาดใหญ่มาก
- Interface ประกอบร่าง (`io.ReadWriter`, `io.ReadCloser`, `io.ReadWriteCloser`) ทำตาม pattern เดียวกับที่เรียนใน **Part 030** — ให้ฟังก์ชันขอความสามารถเท่าที่จำเป็นจริงเท่านั้น
- **Decorator pattern** ด้วยการ embed `io.Reader` (จาก **Part 030**) ใช้เขียน wrapper ที่ทำงานเพิ่มเติมระหว่างการอ่านได้ เช่น `HashingReader` ที่คำนวณ hash ไปพร้อมกับอ่านข้อมูลผ่าน — ถ้า standard library มีตัวช่วยสำเร็จรูปอยู่แล้ว (เช่น `io.TeeReader`) ควรใช้ตัวนั้นก่อนเขียนเอง
- `io.LimitReader` จำกัดปริมาณข้อมูลที่อ่านได้ — สำคัญมากสำหรับป้องกัน HTTP server จาก request body ขนาดใหญ่เกินไป
- `io.MultiReader` เชื่อมหลาย `io.Reader` เป็น stream เดียวโดยไม่ต้องคัดลอกข้อมูลมารวมกันก่อน
- `io.Pipe` เชื่อมโค้ดฝั่งเขียน (`io.Writer`) กับฝั่งอ่าน (`io.Reader`) แบบ synchronous ใน process เดียวกัน โดยไม่มี buffer หรือไฟล์ชั่วคราวเกิดขึ้น — ต้องเขียนฝั่ง write ใน goroutine แยกเสมอและ `Close()` ให้ครบเพื่อส่งสัญญาณ EOF

## แบบฝึกหัดท้ายบท

1. เขียน decorator type ชื่อ `ProgressReader` ที่ห่อ `io.Reader` แล้วพิมพ์เปอร์เซ็นต์ความคืบหน้าออกทาง `fmt.Println` ทุกครั้งที่อ่านครบทุก 25% ของขนาดข้อมูลทั้งหมด (ต้องรู้ขนาดรวมล่วงหน้าตอนสร้าง `ProgressReader`)
2. ใช้ `io.MultiReader` เขียนโปรแกรมที่รวมเนื้อหาจากไฟล์ 3 ไฟล์ในโฟลเดอร์เดียวกันให้กลายเป็น stream เดียว แล้วคัดลอกไปเขียนเป็นไฟล์ใหม่ไฟล์เดียวด้วย `io.Copy` โดยไม่ต้องโหลดไฟล์ใดไฟล์หนึ่งเข้า memory ทั้งก้อนก่อน
3. เขียนฟังก์ชันที่รับ `io.Reader` และ `int64` (ขนาดสูงสุดที่อนุญาต) แล้วคืน error ทันทีถ้าข้อมูลมีขนาดเกินกว่าที่กำหนด (ใบ้: ใช้ `io.LimitReader` กับขนาด `n+1` แล้วเช็คว่าอ่านได้เกิน `n` byte จริงหรือไม่)
4. ใช้ `io.Pipe` เชื่อม `gzip.NewWriter` (ฝั่งเขียนไฟล์บีบอัด) เข้ากับ `http.NewRequest` (ฝั่งอ่านเป็น request body) เพื่ออัปโหลดข้อมูลแบบบีบอัดไปยัง `httptest.Server` โดยไม่สร้างไฟล์ชั่วคราวเลยตลอดกระบวนการ
5. เขียน `MultiWriter` เวอร์ชันของตัวเอง (ก่อนจะไปเปิดดู `io.MultiWriter` ที่มีอยู่แล้วใน standard library) ที่รับ `io.Writer` หลายตัว แล้ว `Write` ข้อมูลชุดเดียวกันไปยังทุกตัวพร้อมกัน คืน error ตัวแรกที่เจอถ้ามีตัวใดตัวหนึ่งเขียนไม่สำเร็จ
6. ทดลองลบ `defer pw.Close()` ออกจากตัวอย่างในหัวข้อ 8 แล้วรันดูว่าเกิดอะไรขึ้น (ใบ้: กรณีนี้ไม่ถึงกับค้างตลอดไปแบบเงียบๆ เพราะ goroutine หลักที่กำลังรอ `decoder.Decode` เพียงตัวเดียวไม่มีใครปลุกได้อีก — runtime ของ Go ตรวจจับ "all goroutines are asleep" ได้เองและจบโปรแกรมด้วย `fatal error: all goroutines are asleep - deadlock!` ทันที ต่างจากตัวอย่าง `select {}` ใน **Part 046** ที่มี goroutine อื่นยังทำงานอยู่จึงต้องใช้ `timeout` ช่วยตัดจบจากภายนอก) อธิบายว่าทำไม Go ตรวจจับกรณีนี้ได้ต่างจากกรณีใน Part 046

---

**ต่อไป**: [Part 049 — `bufio` ขั้นสูง](./049-bufio-advanced.md)
