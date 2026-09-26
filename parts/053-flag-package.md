# Part 053: แพ็กเกจ `flag` — CLI Arguments

> ภาคที่ 4: Standard Library เชิงลึก — ตอนที่ 8 จาก 10 (Part 46–55)

## สารบัญของบทนี้

1. `flag` package คืออะไร และทำไมต้องรู้จัก
2. สร้าง flag พื้นฐาน: `flag.String`, `flag.Int`, `flag.Bool`
3. `flag.Parse()` และลำดับการทำงาน
4. รูปแบบ `Var`: `flag.StringVar`, `flag.IntVar`, `flag.BoolVar`
5. Positional Arguments ด้วย `flag.Args()`
6. สร้าง Custom Flag Type ด้วย `flag.Value`
7. Usage/Help Text อัตโนมัติ และการปรับแต่งด้วย `flag.Usage`
8. ตัวอย่างเต็มรูปแบบ: เครื่องมือนับคำ (Word Count Tool)
9. เมื่อไรควรใช้ `flag` เมื่อไรควรใช้ `cobra`/`urfave/cli`
10. สรุปสิ่งที่ได้เรียนในบทนี้
11. แบบฝึกหัดท้ายบท

---

## 1. `flag` package คืออะไร และทำไมต้องรู้จัก

เมื่อเขียนโปรแกรม command-line (CLI) แทบทุกโปรแกรมต้องรับค่า config หรือตัวเลือกจากผู้ใช้ผ่าน argument ตอนรันคำสั่ง เช่น:

```bash
myapp --port 8080 --verbose file.txt
```

Go มี package มาตรฐานชื่อ **`flag`** ที่ทำหน้าที่นี้โดยเฉพาะ ไม่ต้องพึ่ง library ภายนอกเลยสำหรับ CLI แบบง่ายถึงปานกลาง — `flag` จัดการเรื่องการแปลง type (`string` เป็น `int`, `bool` ฯลฯ), การสร้างข้อความ usage/help อัตโนมัติ, และการแยก flag ออกจาก positional argument ให้ทั้งหมด

จุดเด่นของ `flag` คือความเรียบง่ายตามปรัชญาของ Go เอง — API มีขนาดเล็ก เรียนรู้ได้ในเวลาไม่นาน แต่ครอบคลุมความต้องการพื้นฐานเกือบทั้งหมดของ CLI tool ทั่วไป

---

## 2. สร้าง flag พื้นฐาน: `flag.String`, `flag.Int`, `flag.Bool`

`flag` package มีฟังก์ชันสร้าง flag สำหรับ type พื้นฐานที่ใช้บ่อยที่สุด ได้แก่ `String`, `Int`, `Bool`, `Float64`, `Duration` เป็นต้น รูปแบบทั้งหมดคล้ายกัน:

```go
func String(name string, value string, usage string) *string
func Int(name string, value int, usage string) *int
func Bool(name string, value bool, usage string) *bool
```

- `name` คือชื่อ flag (ไม่ต้องใส่ `-` นำหน้า)
- `value` คือค่า default ถ้าผู้ใช้ไม่ระบุ flag นั้นมา
- `usage` คือคำอธิบายที่จะโชว์ใน help text
- ค่าที่คืนกลับมาเป็น **pointer** เสมอ เพราะตอนเรียกฟังก์ชันเหล่านี้ `flag.Parse()` ยังไม่ทำงาน ค่าจริงจะถูกเติมเข้าไปใน pointer นี้ **ภายหลัง** ตอนเรียก `flag.Parse()`

```go
package main

import (
	"flag"
	"fmt"
)

func main() {
	name := flag.String("name", "world", "ชื่อที่จะทักทาย")
	count := flag.Int("count", 1, "จำนวนครั้งที่จะทักทาย")
	shout := flag.Bool("shout", false, "พิมพ์ตัวใหญ่ทั้งหมด")

	flag.Parse() // ต้องเรียกเสมอ ก่อนใช้ค่าใดๆ จาก flag ที่ประกาศไว้

	greeting := fmt.Sprintf("Hello, %s!", *name)
	if *shout {
		greeting = fmt.Sprintf("HELLO, %s!", *name)
	}

	for i := 0; i < *count; i++ {
		fmt.Println(greeting)
	}
}
```

รันด้วย:

```bash
go run main.go -name Alice -count 2 -shout
```

ผลลัพธ์:

```
HELLO, Alice!
HELLO, Alice!
```

### รูปแบบการเขียน flag ที่ยอมรับได้

Go's `flag` package ยืดหยุ่นเรื่อง syntax ค่อนข้างมาก รูปแบบต่อไปนี้ **ใช้แทนกันได้ทั้งหมด**:

```bash
-name Alice
-name=Alice
--name Alice
--name=Alice
```

ต่างจากหลาย library ใน C/Python ที่แยก `-` (short flag) กับ `--` (long flag) อย่างชัดเจน Go ไม่สนใจว่าจะใช้ `-` หรือ `--` นำหน้า — ทั้งสองแบบทำงานเหมือนกันทุกประการ

---

## 3. `flag.Parse()` และลำดับการทำงาน

`flag.Parse()` คือคำสั่งที่ **อ่านค่าจริงจาก `os.Args`** แล้วเติมค่าลงใน pointer ของทุก flag ที่ประกาศไว้ก่อนหน้า กฎสำคัญที่ต้องจำคือ:

> **ต้องประกาศ flag ทั้งหมดให้เสร็จก่อน แล้วจึงเรียก `flag.Parse()` เป็นลำดับสุดท้าย** และห้ามอ่านค่าจาก pointer ที่ได้จาก `flag.String`/`flag.Int`/ฯลฯ ก่อนที่ `flag.Parse()` จะถูกเรียก เพราะค่าจะยังเป็นแค่ default เท่านั้น ยังไม่ถูกอ่านจาก command line จริง

```go
name := flag.String("name", "default", "usage text")
// ณ จุดนี้ *name ยังคงเป็น "default" เสมอ ไม่ว่าผู้ใช้จะพิมพ์ -name อะไรมาก็ตาม

flag.Parse()
// หลังจากนี้เท่านั้น *name จึงจะมีค่าตามที่ผู้ใช้ระบุจริง (หรือยังเป็น default ถ้าไม่ได้ระบุ)
```

Go's `flag` package ยังหยุดการอ่าน flag ทันทีที่เจอ argument ตัวแรกที่ไม่ขึ้นต้นด้วย `-` (หรือเจอ `--` เดี่ยวๆ) — argument ที่เหลือทั้งหมดหลังจากจุดนั้นจะถูกมองเป็น **positional argument** ซึ่งเราจะพูดถึงในหัวข้อที่ 5

---

## 4. รูปแบบ `Var`: `flag.StringVar`, `flag.IntVar`, `flag.BoolVar`

นอกจากรูปแบบที่คืนค่าเป็น pointer ใหม่ (`flag.String(...)`) `flag` package ยังมีรูปแบบ `...Var` ที่ให้เราส่ง**ตัวแปรของเราเอง**เข้าไปให้ `flag` เขียนค่าลงไปโดยตรง แทนที่จะสร้าง pointer ใหม่ให้

```go
func StringVar(p *string, name string, value string, usage string)
func IntVar(p *int, name string, value int, usage string)
func BoolVar(p *bool, name string, value bool, usage string)
```

```go
package main

import (
	"flag"
	"fmt"
)

type config struct {
	Host    string
	Port    int
	Verbose bool
}

func main() {
	var cfg config

	flag.StringVar(&cfg.Host, "host", "localhost", "hostname ที่จะ bind")
	flag.IntVar(&cfg.Port, "port", 8080, "port ที่จะ bind")
	flag.BoolVar(&cfg.Verbose, "verbose", false, "เปิด verbose logging")

	flag.Parse()

	fmt.Printf("config: %+v\n", cfg)
}
```

รูปแบบ `Var` มีประโยชน์มากเมื่อต้องการรวม field ของ flag เข้าไปใน struct เดียว (เช่น struct `config` ในตัวอย่าง) ทำให้ส่งต่อ config ทั้งก้อนไปยังฟังก์ชันอื่นได้สะดวกกว่าการถือ pointer แยกหลายตัว — เป็นรูปแบบที่นิยมมากในโปรเจกต์ระดับ production

---

## 5. Positional Arguments ด้วย `flag.Args()`

หลายครั้ง CLI tool ต้องการรับทั้ง **flag** (ตัวเลือกที่มีชื่อ) และ **positional argument** (ค่าที่ไม่มีชื่อ ระบุตามลำดับตำแหน่ง) พร้อมกัน เช่น:

```bash
mytool -verbose file1.txt file2.txt
```

ในที่นี้ `-verbose` คือ flag ส่วน `file1.txt` และ `file2.txt` คือ positional argument — หลังจากเรียก `flag.Parse()` แล้ว เราดึง positional argument ที่เหลือทั้งหมดได้ผ่าน `flag.Args()` (คืนเป็น `[]string`) หรือ `flag.Arg(i)` (ดึงตัวที่ index `i` ตัวเดียว):

```go
package main

import (
	"flag"
	"fmt"
)

func main() {
	verbose := flag.Bool("verbose", false, "เปิด verbose logging")
	flag.Parse()

	fmt.Println("verbose:", *verbose)
	fmt.Println("จำนวน positional args:", flag.NArg())
	fmt.Println("positional args ทั้งหมด:", flag.Args())

	for i, arg := range flag.Args() {
		fmt.Printf("  arg[%d] = %s\n", i, arg)
	}
}
```

รันด้วย:

```bash
go run main.go -verbose file1.txt file2.txt
```

ผลลัพธ์:

```
verbose: true
จำนวน positional args: 2
positional args ทั้งหมด: [file1.txt file2.txt]
  arg[0] = file1.txt
  arg[1] = file2.txt
```

### ข้อจำกัดที่ต้องรู้: flag ต้องมาก่อน positional argument เสมอ

`flag` package ของ Go standard library **หยุดอ่าน flag ทันทีที่เจอ argument แรกที่ไม่ใช่ flag** ซึ่งหมายความว่ารูปแบบนี้จะ**ไม่ทำงานตามที่คาดหวัง**:

```bash
mytool file1.txt -verbose
```

ในกรณีนี้ `-verbose` จะถูกมองเป็น **positional argument** ไปด้วย (ไม่ถูกตีความเป็น flag) เพราะเจอ `file1.txt` ก่อนแล้ว หยุดอ่าน flag ไปแล้ว — นี่เป็นข้อจำกัดที่ต่างจาก getopt-style parser ในบางภาษา ถ้าต้องการ flag ที่ผสมตำแหน่งกับ positional argument ได้อย่างอิสระ จำเป็นต้องใช้ library ภายนอกอย่าง `cobra` หรือ `pflag` (จะพูดถึงในหัวข้อที่ 9)

---

## 6. สร้าง Custom Flag Type ด้วย `flag.Value`

บางครั้ง type พื้นฐาน (`string`, `int`, `bool`) ไม่เพียงพอ เช่น ต้องการ flag ที่รับได้เฉพาะค่าจากชุดที่กำหนดไว้ล่วงหน้า (`low`, `medium`, `high`) หรือ flag ที่รับ list ของค่าคั่นด้วยจุลภาค — `flag` package รองรับ **custom type** ใดๆ ก็ได้ ตราบใดที่ type นั้น implement interface `flag.Value`:

```go
type Value interface {
	String() string
	Set(string) error
}
```

- `String() string` — ใช้ตอนแสดงค่า default ใน help text
- `Set(string) error` — ถูกเรียกตอน parse flag เพื่อแปลง string ที่ผู้ใช้พิมพ์มาเป็นค่าจริง คืน error ได้ถ้าค่าที่ป้อนมาไม่ถูกต้อง

```go
package main

import (
	"flag"
	"fmt"
)

// levelFlag คือ custom flag type ที่รับได้เฉพาะ "low", "medium", "high"
type levelFlag struct {
	value string
}

func (l *levelFlag) String() string {
	return l.value
}

func (l *levelFlag) Set(s string) error {
	switch s {
	case "low", "medium", "high":
		l.value = s
		return nil
	default:
		return fmt.Errorf("invalid level %q (must be low, medium, or high)", s)
	}
}

func main() {
	var level levelFlag
	level.value = "low" // ตั้งค่า default เอง เพราะ flag.Var ไม่มีพารามิเตอร์ value แยก

	flag.Var(&level, "level", "verbosity level: low, medium, or high")
	flag.Parse()

	fmt.Println("level:", level.String())
}
```

รันด้วยค่าที่ถูกต้อง:

```bash
go run main.go -level=high
# level: high
```

รันด้วยค่าที่ไม่ถูกต้อง:

```bash
go run main.go -level=extreme
# invalid value "extreme" for flag -level: invalid level "extreme" (must be low, medium, or high)
```

สังเกตว่า `flag.Var(&level, "level", "...")` **ไม่มีพารามิเตอร์ value กำหนด default แยก** ต่างจาก `flag.String`/`flag.Int` — เราต้องตั้งค่า default เองก่อนเรียก `flag.Var` (บรรทัด `level.value = "low"` ในตัวอย่าง) เพราะ `flag` ไม่รู้ว่า type ของเราจะแปลงค่า default ยังไง จึงปล่อยให้เราจัดการเอง

Custom flag type เป็นเทคนิคที่ทรงพลังมาก ใช้สร้าง flag ประเภทซับซ้อนได้หลากหลาย เช่น flag ที่รับ list ของค่า (`-tag foo -tag bar` เก็บสะสมเป็น slice), flag ที่ parse เป็น enum, หรือ flag ที่ validate รูปแบบเฉพาะ เช่น IP address หรือ URL

---

## 7. Usage/Help Text อัตโนมัติ และการปรับแต่งด้วย `flag.Usage`

`flag` package สร้างข้อความ help ให้อัตโนมัติจาก `usage` string ที่ระบุตอนประกาศแต่ละ flag เมื่อผู้ใช้พิมพ์ `-h` หรือ `--help` (หรือเมื่อ parse ผิดพลาด) Go จะพิมพ์ help text นี้ออกทาง stderr โดยอัตโนมัติแล้วออกจากโปรแกรมด้วย exit code 2

ลองดูตัวอย่างจาก flag ในหัวข้อก่อนหน้า ถ้าเรียก `go run main.go -h`:

```
Usage of main:
  -level value
    	verbosity level: low, medium, or high (default low)
```

### ปรับแต่ง help text เอง ด้วย `flag.Usage`

ในหลายกรณี เราอยากได้ header, ตัวอย่างการใช้งาน หรือรูปแบบข้อความที่ต่างจาก default — `flag` package เปิดให้เรา override ได้ผ่านตัวแปร `flag.Usage` ซึ่งเป็นแค่ `func()` ธรรมดา:

```go
package main

import (
	"flag"
	"fmt"
	"os"
)

func main() {
	verbose := flag.Bool("verbose", false, "แสดง log แบบละเอียด")
	output := flag.String("output", "-", "ไฟล์ปลายทาง (- คือ stdout)")

	flag.Usage = func() {
		fmt.Fprintf(os.Stderr, "mytool: เครื่องมือประมวลผลไฟล์ตัวอย่าง\n\n")
		fmt.Fprintf(os.Stderr, "การใช้งาน:\n  mytool [flags] <input-file>\n\n")
		fmt.Fprintf(os.Stderr, "ตัวเลือกที่รองรับ:\n")
		flag.PrintDefaults() // พิมพ์รายการ flag ทั้งหมดในรูปแบบมาตรฐานของ flag package
	}

	flag.Parse()
	_ = verbose
	_ = output
}
```

`flag.PrintDefaults()` เป็นฟังก์ชัน helper ที่พิมพ์รายการ flag ทั้งหมดในรูปแบบมาตรฐาน (ชื่อ, type, usage, default) — ใช้ร่วมกับข้อความส่วนหัวที่เขียนเองได้ ทำให้ได้ help text ที่ทั้งเป็นมาตรฐานและมีบริบทเพิ่มเติมตามที่ต้องการ

---

## 8. ตัวอย่างเต็มรูปแบบ: เครื่องมือนับคำ (Word Count Tool)

มารวมทุกอย่างที่เรียนมาเป็นเครื่องมือ CLI ที่ใช้งานได้จริง คล้ายกับคำสั่ง `wc` ของ Unix รองรับ flag `-lines`, `-words`, `-bytes`, custom flag `-level`, และอ่านได้ทั้งจากไฟล์ (positional argument) หรือจาก stdin:

```go
package main

import (
	"bufio"
	"flag"
	"fmt"
	"io"
	"os"
	"strings"
)

// levelFlag คือ custom flag ตัวอย่างที่ใช้ประกอบผลลัพธ์ (แสดงระดับความละเอียดของ log)
type levelFlag struct {
	value string
}

func (l *levelFlag) String() string {
	return l.value
}

func (l *levelFlag) Set(s string) error {
	switch s {
	case "low", "medium", "high":
		l.value = s
		return nil
	default:
		return fmt.Errorf("invalid level %q (must be low, medium, or high)", s)
	}
}

func countStats(r io.Reader) (lines, words, bytesCount int) {
	scanner := bufio.NewScanner(r)
	for scanner.Scan() {
		line := scanner.Text()
		lines++
		bytesCount += len(line) + 1 // +1 สำหรับ newline ที่ Scanner ตัดออกไป
		words += len(strings.Fields(line))
	}
	return
}

func main() {
	showLines := flag.Bool("lines", false, "แสดงจำนวนบรรทัด")
	showWords := flag.Bool("words", false, "แสดงจำนวนคำ")
	showBytes := flag.Bool("bytes", false, "แสดงจำนวนไบต์")

	var level levelFlag
	level.value = "low"
	flag.Var(&level, "level", "ระดับความละเอียดของผลลัพธ์: low, medium, high")

	flag.Usage = func() {
		fmt.Fprintf(os.Stderr, "usage: wc [flags] [file...]\n\n")
		fmt.Fprintf(os.Stderr, "wc counts lines, words, and bytes in files (or stdin).\n\n")
		flag.PrintDefaults()
	}

	flag.Parse()

	// ถ้าไม่ระบุ flag ไหนเลย ให้แสดงทั้งสามค่า (พฤติกรรม default ของ wc จริง)
	if !*showLines && !*showWords && !*showBytes {
		*showLines, *showWords, *showBytes = true, true, true
	}

	files := flag.Args()
	if len(files) == 0 {
		files = []string{"-"} // "-" หมายถึงอ่านจาก stdin
	}

	for _, name := range files {
		var r io.Reader
		if name == "-" {
			r = os.Stdin
		} else {
			f, err := os.Open(name)
			if err != nil {
				fmt.Fprintln(os.Stderr, "error:", err)
				os.Exit(1)
			}
			defer f.Close()
			r = f
		}

		lines, words, bytesCount := countStats(r)
		var parts []string
		if *showLines {
			parts = append(parts, fmt.Sprintf("lines=%d", lines))
		}
		if *showWords {
			parts = append(parts, fmt.Sprintf("words=%d", words))
		}
		if *showBytes {
			parts = append(parts, fmt.Sprintf("bytes=%d", bytesCount))
		}
		fmt.Printf("%s (level=%s): %s\n", name, level.String(), strings.Join(parts, " "))
	}
}
```

ทดสอบรัน:

```bash
$ cat file.txt
line one has three
line two

$ go run main.go file.txt
file.txt (level=low): lines=2 words=6 bytes=28

$ go run main.go -lines -level=high file.txt
file.txt (level=high): lines=2

$ echo "piped input here" | go run main.go -words
- (level=low): words=3

$ go run main.go -badflag
flag provided but not defined: -badflag
usage: wc [flags] [file...]

wc counts lines, words, and bytes in files (or stdin).

  -bytes
    	show byte count
  -level value
    	ระดับความละเอียดของผลลัพธ์: low, medium, high (default low)
  -lines
    	show line count
  -words
    	show word count
```

ตัวอย่างนี้แสดงให้เห็นการผสมผสานครบทุกเทคนิคในบทนี้: flag แบบ `Bool`, custom flag ผ่าน `flag.Value`, positional argument ผ่าน `flag.Args()`, การรองรับ stdin เป็นค่า default, และ usage message ที่ปรับแต่งเอง — เป็นรูปแบบที่ใช้สร้าง CLI tool ระดับ production ได้จริง

---

## 9. เมื่อไรควรใช้ `flag` เมื่อไรควรใช้ `cobra`/`urfave/cli`

`flag` เพียงพอสำหรับ CLI tool ขนาดเล็กถึงปานกลาง แต่เมื่อโปรเจกต์ซับซ้อนขึ้น มักจะเจอข้อจำกัดที่ทำให้ต้องมองหา library ภายนอก ไลบรารีสองตัวที่ได้รับความนิยมที่สุดในระบบนิเวศ Go คือ:

- **[`spf13/cobra`](https://github.com/spf13/cobra)**: ใช้สร้าง CLI แบบมี **subcommand** ซ้อนกันได้หลายระดับ (เช่น `git commit`, `git remote add`, `docker container ls`) เป็น library ที่ `kubectl`, `docker`, `hugo`, `github cli` ใช้จริงใน production
- **[`urfave/cli`](https://github.com/urfave/cli)**: API เรียบง่ายกว่า cobra เล็กน้อย เหมาะกับ CLI ที่ไม่ซับซ้อนมากแต่ต้องการ feature มากกว่า `flag` เช่น subcommand พื้นฐาน, flag ที่ผสมตำแหน่งได้อิสระกว่า

### ตารางเปรียบเทียบเพื่อช่วยตัดสินใจ

| ต้องการ | `flag` (stdlib) เพียงพอไหม |
|---|---|
| CLI ที่มี flag ระดับเดียว ไม่มี subcommand | เพียงพอ |
| ต้องการ subcommand (เช่น `mytool add`, `mytool remove`) | ไม่เพียงพอ — ใช้ `cobra` |
| ต้องการ flag ผสมตำแหน่งกับ positional argument ได้อิสระ | ไม่เพียงพอ (ข้อจำกัดตามหัวข้อ 5) — ใช้ `cobra`/`pflag` |
| ต้องการ auto-completion (bash/zsh completion) | ไม่มีในตัว — ใช้ `cobra` |
| ต้องการ short flag แบบรวมกัน (`-abc` = `-a -b -c`) | ไม่รองรับ — ใช้ `pflag` หรือ `urfave/cli` |
| CLI ขนาดเล็ก สคริปต์ภายใน หรือเครื่องมือ internal | `flag` เพียงพอและไม่ต้องเพิ่ม dependency |

### หลักคิดโดยรวม

> เริ่มต้นด้วย `flag` เสมอสำหรับเครื่องมือขนาดเล็ก เพราะไม่ต้องเพิ่ม dependency ให้โปรเจกต์และครอบคลุมความต้องการพื้นฐานเกือบทั้งหมด เมื่อโปรเจกต์เติบโตจนต้องการ subcommand หลายระดับ หรือ UX ที่ซับซ้อนขึ้น (เช่น CLI tool ที่จะแจกจ่ายให้คนอื่นใช้งานอย่างกว้างขวาง) ค่อยพิจารณาย้ายไปใช้ `cobra` ซึ่งเป็นตัวเลือกที่ได้รับความนิยมมากที่สุดในระบบนิเวศ Go ปัจจุบัน

---

## สรุปสิ่งที่ได้เรียนในบทนี้

- `flag` package เป็นเครื่องมือมาตรฐานของ Go สำหรับ parse command-line argument โดยไม่ต้องพึ่ง library ภายนอก
- `flag.String`/`flag.Int`/`flag.Bool` คืนค่าเป็น pointer ที่จะถูกเติมค่าจริงหลังเรียก `flag.Parse()` เท่านั้น
- รูปแบบ `...Var` (`flag.StringVar` ฯลฯ) ให้เขียนค่าลงตัวแปรที่เรากำหนดเอง เหมาะกับการรวม flag เข้า struct config
- `flag.Args()`/`flag.Arg(i)`/`flag.NArg()` ใช้ดึง positional argument ที่เหลือหลังจาก flag ทั้งหมด — แต่ flag ต้องมาก่อน positional argument เสมอ ผสมกันไม่ได้อย่างอิสระ
- Custom flag type ทำได้โดย implement interface `flag.Value` (มีเมธอด `String()` และ `Set(string) error`) แล้วใช้กับ `flag.Var`
- `flag.Usage` เป็น `func()` ที่ override ได้เพื่อปรับแต่ง help text เอง ร่วมกับ `flag.PrintDefaults()` ที่พิมพ์รายการ flag มาตรฐาน
- `flag` เพียงพอสำหรับ CLI ขนาดเล็ก-กลาง แต่เมื่อต้องการ subcommand หลายระดับหรือ feature ขั้นสูง ควรพิจารณา `cobra` หรือ `urfave/cli`

## แบบฝึกหัดท้ายบท

1. เขียนโปรแกรม CLI ที่รับ flag `-n int` (จำนวนครั้ง, default 1) และ positional argument หนึ่งตัวเป็นข้อความ แล้วพิมพ์ข้อความนั้นซ้ำตามจำนวนที่กำหนด (คล้ายคำสั่ง `yes`)
2. เขียน custom flag type ชื่อ `csvFlag` ที่ implement `flag.Value` เพื่อรับค่า flag แบบคั่นด้วยจุลภาค (เช่น `-tags=go,backend,api`) แล้วแปลงเป็น `[]string` เก็บไว้ใช้งาน
3. ปรับปรุงตัวอย่าง word count tool ในหัวข้อที่ 8 ให้เพิ่ม flag `-chars` สำหรับนับจำนวนตัวอักษร (ไม่ใช่ไบต์ — ให้ใช้ `utf8.RuneCountInString` เพื่อรองรับภาษาไทย/unicode ถูกต้อง)
4. เขียนโปรแกรมที่มี custom `flag.Usage` แสดงตัวอย่างการใช้งานอย่างน้อย 2 ตัวอย่างในข้อความ help พร้อมทดสอบด้วย `-h`
5. ทดลองรัน `go run main.go file.txt -verbose` กับโปรแกรมที่มี flag `-verbose` แล้วสังเกตว่า `-verbose` ไม่ถูกอ่านเป็น flag ตามที่อธิบายในหัวข้อที่ 5 พิสูจน์ด้วยการพิมพ์ค่าที่ได้จาก `flag.Args()` ออกมาดู
6. ค้นคว้าเพิ่มเติมเกี่ยวกับ `flag.NewFlagSet` ที่ใช้สร้างชุด flag แยกต่างหากจากชุด global (มีประโยชน์มากเวลาสร้าง subcommand เองแบบง่ายๆ โดยไม่ต้องพึ่ง cobra) แล้วลองเขียน CLI ที่มี subcommand สองแบบ เช่น `mytool add ...` และ `mytool list ...` โดยแต่ละ subcommand มี flag ของตัวเอง

---

**ต่อไป**: [Part 054 — แพ็กเกจ `log` และแนวทาง Logging ที่ดี](./054-log-package.md)
