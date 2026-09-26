# Part 052: `os/exec` — รัน External Command

> ภาคที่ 4: Standard Library เชิงลึก — ตอนที่ 7 จาก 10 (Part 46–55)

## สารบัญของบทนี้

1. `os/exec` คืออะไร และใช้ทำอะไร
2. `exec.Command` พื้นฐาน
3. `Run()` vs `Output()` vs `CombinedOutput()`
4. แยก stdout/stderr ด้วย `cmd.Stdout`/`cmd.Stderr`
5. ส่งข้อมูลเข้า subprocess ผ่าน stdin
6. Streaming Output แบบ Real-time ด้วย `StdoutPipe`, `Start()`, `Wait()`
7. กำหนด Environment Variables และ Working Directory
8. ตรวจสอบ Exit Code ด้วย `*exec.ExitError`
9. ยกเลิก subprocess ที่ค้างด้วย `context.WithTimeout` + `exec.CommandContext`
10. คำเตือนด้านความปลอดภัย: Command Injection
11. สรุปสิ่งที่ได้เรียนในบทนี้
12. แบบฝึกหัดท้ายบท

---

## 1. `os/exec` คืออะไร และใช้ทำอะไร

บางครั้งโปรแกรม Go ของเราต้องการเรียกใช้โปรแกรมภายนอกที่ติดตั้งอยู่ในระบบปฏิบัติการ เช่น เรียก `git` เพื่อ clone repository, เรียก `ffmpeg` เพื่อแปลงไฟล์วิดีโอ, เรียก `docker` เพื่อจัดการ container, หรือแค่เรียกคำสั่ง shell พื้นฐานอย่าง `ls`/`grep` — package **`os/exec`** คือเครื่องมือมาตรฐานของ Go สำหรับงานประเภทนี้

`os/exec` ให้ความสามารถครบถ้วนในการ:

- รันโปรแกรมภายนอกและรอผลลัพธ์
- ควบคุม stdin/stdout/stderr ของ subprocess อย่างละเอียด
- กำหนด environment variable และ working directory ของ subprocess แยกจากโปรเซสหลัก
- ตรวจสอบ exit code เพื่อรู้ว่าโปรแกรมที่เรียกสำเร็จหรือล้มเหลว
- ยกเลิกโปรแกรมที่ทำงานนานเกินไปด้วย `context`

เครื่องมือ DevOps และ CLI ชื่อดังจำนวนมากที่เขียนด้วย Go เช่น Docker CLI, Terraform, kubectl ล้วนใช้ `os/exec` เป็นส่วนประกอบสำคัญในการเรียกใช้เครื่องมือภายนอกหรือแม้แต่เรียกตัวเองแบบ subprocess

---

## 2. `exec.Command` พื้นฐาน

จุดเริ่มต้นของทุกการเรียก subprocess คือฟังก์ชัน `exec.Command`:

```go
func Command(name string, arg ...string) *Cmd
```

- `name` คือชื่อโปรแกรมที่จะรัน (Go จะค้นหาใน `PATH` ให้อัตโนมัติ เหมือนพิมพ์ใน terminal)
- `arg ...string` คือ argument แต่ละตัว **แยกกันเป็น string คนละตัว** — นี่คือจุดสำคัญที่จะพูดถึงซ้ำในหัวข้อความปลอดภัยท้ายบท

```go
package main

import (
	"fmt"
	"os/exec"
)

func main() {
	cmd := exec.Command("echo", "hello from subprocess")
	err := cmd.Run()
	fmt.Println("Run err:", err)
}
```

ผลลัพธ์:

```
Run err: <nil>
```

สังเกตว่าเราไม่เห็นข้อความ `hello from subprocess` เลย เพราะ `Run()` เพียงแค่ **รันคำสั่งแล้วรอจนจบ** โดยไม่ได้ capture หรือ forward stdout ของ subprocess ไปที่ไหน ถ้าอยากเห็น output ต้องเลือกวิธีที่เหมาะสมตามหัวข้อถัดไป

`exec.Command` เพียงแค่ **สร้าง** ค่า `*exec.Cmd` ขึ้นมา ยังไม่ได้รันโปรแกรมจริง ๆ จนกว่าจะเรียก method อย่าง `Run()`, `Output()`, `Start()` เป็นต้น — pattern นี้คล้ายกับ builder pattern ที่เราตั้งค่า field ต่างๆ ของ `*exec.Cmd` ก่อน (`Dir`, `Env`, `Stdin`, `Stdout`, ...) แล้วค่อยสั่งให้ทำงานทีหลัง

---

## 3. `Run()` vs `Output()` vs `CombinedOutput()`

`*exec.Cmd` มี method หลักสามตัวที่ใช้บ่อยที่สุดสำหรับการรันแบบ synchronous (รอจนจบก่อนไปบรรทัดถัดไป) แต่ละตัวเหมาะกับสถานการณ์ต่างกัน:

| Method | Return | เหมาะกับ |
|---|---|---|
| `Run() error` | แค่ error (หรือ nil) | ไม่สนใจ output เลย แค่ต้องการรู้ว่าสำเร็จหรือไม่ |
| `Output() ([]byte, error)` | stdout เท่านั้น | ต้องการอ่านผลลัพธ์ที่โปรแกรม "ตั้งใจ" ส่งออกมา |
| `CombinedOutput() ([]byte, error)` | stdout + stderr รวมกัน | ต้องการ debug หรือ log ทุกอย่างที่โปรแกรมพิมพ์ออกมา |

```go
package main

import (
	"fmt"
	"os/exec"
)

func runOutput() {
	cmd := exec.Command("echo", "captured output")
	out, err := cmd.Output()
	fmt.Println("Output:", string(out), "err:", err)
}

func runCombined() {
	cmd := exec.Command("sh", "-c", "echo to-stdout; echo to-stderr 1>&2")
	out, err := cmd.CombinedOutput()
	fmt.Println("CombinedOutput:", string(out), "err:", err)
}

func main() {
	runOutput()
	runCombined()
}
```

ผลลัพธ์:

```
Output: captured output
 err: <nil>
CombinedOutput: to-stdout
to-stderr
 err: <nil>
```

### ข้อควรระวังของ `Output()`

ถ้า subprocess เขียนอะไรลง **stderr** ระหว่างที่ `cmd.Stderr` ยังไม่ได้ถูกกำหนดไว้ Go จะไม่ capture มันไปไหนเลย (ไม่ error แต่ไม่ได้อะไรกลับมา) แต่ถ้าโปรแกรม **จบด้วย exit code ที่ไม่ใช่ 0** เมธอด `Output()` จะคืน error เป็น `*exec.ExitError` ซึ่งมี field `Stderr` แนบข้อความ stderr มาให้ดูด้วย (แต่จำกัดขนาดไว้ที่ไม่กี่ KB เท่านั้น) — ถ้าต้องการ stderr แบบเต็มเสมอ ควรกำหนด `cmd.Stderr` เองแทนตามหัวข้อถัดไป

---

## 4. แยก stdout/stderr ด้วย `cmd.Stdout`/`cmd.Stderr`

ในสถานการณ์จริงส่วนใหญ่ เราต้องการควบคุมปลายทางของ stdout และ stderr แยกจากกันอย่างละเอียด — `*exec.Cmd` มี field `Stdout` และ `Stderr` ที่รับค่าเป็น **`io.Writer`** ธรรมดา ทำให้เราส่งอะไรก็ได้ที่ implement `io.Writer` เข้าไป ไม่ว่าจะเป็นไฟล์, buffer ในหน่วยความจำ, หรือแม้แต่ `os.Stdout`/`os.Stderr` ของโปรเซสหลักเราเอง (ทำให้ output ของ subprocess โผล่ตรงหน้าจอทันที) — แนวคิด `io.Writer` นี้คือ interface เดียวกับที่เราเรียนเจาะลึกไปในบทที่ 048 (`io.Reader`/`io.Writer` และ Interface Composition)

```go
package main

import (
	"bytes"
	"fmt"
	"os/exec"
)

func main() {
	cmd := exec.Command("sh", "-c", "echo to-stdout; echo to-stderr 1>&2")

	var stdout, stderr bytes.Buffer
	cmd.Stdout = &stdout // bytes.Buffer implement io.Writer จึงใช้เป็นปลายทางได้ตรงๆ
	cmd.Stderr = &stderr

	err := cmd.Run()
	fmt.Printf("stdout=%q stderr=%q err=%v\n", stdout.String(), stderr.String(), err)
}
```

ผลลัพธ์:

```
stdout="to-stdout\n" stderr="to-stderr\n" err=<nil>
```

เพราะ `Stdout`/`Stderr` เป็นแค่ `io.Writer` เฉยๆ เราจึงสามารถ:

- ส่ง `os.Stdout` ตรงๆ เพื่อให้ output ของ subprocess ไหลออกหน้าจอ terminal ทันที (เหมาะกับ CLI tool ที่อยาก forward output ให้ผู้ใช้เห็นแบบ real-time)
- ส่งไฟล์ที่เปิดด้วย `os.Create` เพื่อบันทึก log ของ subprocess ลงไฟล์โดยตรง
- ส่ง `io.MultiWriter(os.Stdout, logFile)` (จากบทที่ 048) เพื่อให้ output ไปทั้งหน้าจอและไฟล์พร้อมกัน

```go
cmd := exec.Command("some-tool")
cmd.Stdout = os.Stdout
cmd.Stderr = os.Stderr
err := cmd.Run() // output ของ subprocess จะไหลออกหน้าจอทันที ไม่ต้องรอจบก่อน
```

---

## 5. ส่งข้อมูลเข้า subprocess ผ่าน stdin

เช่นเดียวกับ `Stdout`/`Stderr` field `Stdin` ก็รับค่าเป็น `io.Reader` ทำให้เราส่งข้อมูลเข้าไปให้ subprocess อ่านผ่าน standard input ได้โดยตรง — มีประโยชน์มากเวลาต้อง "ต่อท่อ" (pipe) ข้อมูลระหว่างโปรแกรม Go กับเครื่องมือ command-line เช่น ส่งข้อความเข้า `grep`, ส่ง JSON เข้า `jq`, หรือส่งเนื้อหาเข้า `cat`

```go
package main

import (
	"fmt"
	"os/exec"
	"strings"
)

func main() {
	cmd := exec.Command("cat")
	cmd.Stdin = strings.NewReader("data piped via stdin\n")

	out, err := cmd.Output()
	fmt.Println("stdin result:", string(out), "err:", err)
}
```

ผลลัพธ์:

```
stdin result: data piped via stdin
 err: <nil>
```

`strings.NewReader` คืนค่า `io.Reader` จาก string ธรรมดา แต่ในทางปฏิบัติ `cmd.Stdin` รับ `io.Reader` อะไรก็ได้เช่นกัน เช่น `os.Stdin` (forward stdin ของโปรเซสหลักไปให้ subprocess โดยตรง) หรือไฟล์ที่เปิดไว้

---

## 6. Streaming Output แบบ Real-time ด้วย `StdoutPipe`, `Start()`, `Wait()`

`Run()`, `Output()`, และ `CombinedOutput()` ทั้งหมดเป็นแบบ **blocking แบบเต็มรูปแบบ** คือรอจน subprocess จบการทำงานก่อน แล้วจึงคืนผลลัพธ์ทั้งหมดกลับมาทีเดียว วิธีนี้ใช้งานง่ายแต่ไม่เหมาะกับกรณีที่ subprocess ใช้เวลานานและเราต้องการเห็น output **ทีละบรรทัดทันทีที่มันเกิดขึ้น** (เช่น แสดง progress ของการ build, การ download หรือ log ของ process ที่รันต่อเนื่อง)

สำหรับกรณีนี้ ต้องแยกขั้นตอนออกเป็น 3 ส่วน: เปิด pipe ด้วย `cmd.StdoutPipe()`, เริ่ม subprocess แบบไม่รอด้วย `cmd.Start()`, แล้วอ่านผลลัพธ์แบบ streaming ก่อนปิดท้ายด้วย `cmd.Wait()`:

```go
package main

import (
	"bufio"
	"fmt"
	"os/exec"
)

func main() {
	cmd := exec.Command("sh", "-c", "for i in 1 2 3; do echo line-$i; sleep 0.05; done")

	// StdoutPipe คืน io.ReadCloser ที่เชื่อมกับ stdout ของ subprocess โดยตรง
	// ต้องเรียกก่อน Start() เสมอ มิฉะนั้นจะไม่มี pipe ให้เชื่อมต่อ
	stdout, err := cmd.StdoutPipe()
	if err != nil {
		panic(err)
	}

	// Start() เริ่ม subprocess แล้ว "คืนการควบคุมทันที" ไม่รอจนจบเหมือน Run()
	if err := cmd.Start(); err != nil {
		panic(err)
	}
	fmt.Println("started with PID:", cmd.Process.Pid)

	// อ่าน output ทีละบรรทัดขณะที่ subprocess กำลังทำงานอยู่จริง
	scanner := bufio.NewScanner(stdout)
	for scanner.Scan() {
		fmt.Println("streamed:", scanner.Text())
	}

	// Wait() รอจน subprocess จบการทำงานจริง และคืนค่า error ถ้าจบด้วย exit code ที่ไม่ใช่ 0
	if err := cmd.Wait(); err != nil {
		fmt.Println("wait error:", err)
	} else {
		fmt.Println("process finished cleanly")
	}
}
```

ผลลัพธ์ (แต่ละบรรทัด `streamed: ...` จะปรากฏขึ้นทันทีตามจังหวะที่ subprocess พิมพ์ออกมาจริง ไม่ใช่รอครบทั้งหมดก่อน):

```
started with PID: 8275
streamed: line-1
streamed: line-2
streamed: line-3
process finished cleanly
```

### จุดสำคัญของรูปแบบ `Start()` + `Wait()`

- **`cmd.Process.Pid`** ให้ process ID ของ subprocess ที่กำลังรันอยู่จริง มีประโยชน์เวลาต้อง log หรือติดตามสถานะ process จากภายนอก
- **ต้องเรียก `cmd.Wait()` เสมอ** หลังใช้ `Start()` แม้จะอ่าน output ผ่าน pipe จนจบแล้วก็ตาม เพราะ `Wait()` เป็นตัวที่คอย "เก็บกวาด" resource ของ subprocess (release file descriptor, รอ process ที่กลายเป็น zombie process ใน Unix) ถ้าไม่เรียก `Wait()` เลย subprocess ที่จบไปแล้วอาจค้างเป็น zombie process อยู่ในระบบ
- รูปแบบนี้เป็นพื้นฐานเดียวกับที่ `Run()` ใช้ภายใน — จริงๆ แล้ว `Run()` ก็คือ `Start()` ตามด้วย `Wait()` ที่ห่อรวมให้สะดวกขึ้นนั่นเอง ส่วน `StdoutPipe()`/`StdinPipe()`/`StderrPipe()` ให้ความยืดหยุ่นเพิ่มเติมสำหรับกรณีที่ต้องการควบคุมจังหวะการอ่าน/เขียนเองอย่างละเอียด

---

## 7. กำหนด Environment Variables และ Working Directory

`*exec.Cmd` มี field `Env` และ `Dir` สำหรับควบคุมสภาพแวดล้อมที่ subprocess จะรันอยู่ **แยกต่างหากจากโปรเซสหลักของเรา**

```go
package main

import (
	"fmt"
	"os"
	"os/exec"
)

func main() {
	cmd := exec.Command("sh", "-c", "echo $MY_VAR; pwd")

	// os.Environ() คืน environment variable ทั้งหมดของโปรเซสปัจจุบันเป็น []string
	// ต้อง append ค่าเพิ่มเข้าไป ไม่งั้น subprocess จะไม่เห็น PATH, HOME ฯลฯ เลย
	cmd.Env = append(os.Environ(), "MY_VAR=custom-value")
	cmd.Dir = "/tmp"

	out, err := cmd.Output()
	fmt.Println("env/dir result:", string(out), "err:", err)
}
```

ผลลัพธ์:

```
env/dir result: custom-value
/tmp
 err: <nil>
```

### จุดสำคัญเรื่อง `Env`

- ถ้า **ไม่กำหนด** `cmd.Env` เลย (ปล่อยเป็น `nil`) subprocess จะได้ environment variable ชุดเดียวกับโปรเซสหลักโดยอัตโนมัติ (พฤติกรรม default)
- แต่ถ้า **กำหนด** `cmd.Env` เป็นค่าอะไรก็ตาม (แม้แค่ค่าเดียว) จะเป็นการ **แทนที่ทั้งชุด** ไม่ใช่การเพิ่มเข้าไป — ดังนั้นถ้าต้องการแค่ "เพิ่ม" ตัวแปรใหม่โดยยังคง environment เดิมไว้ ต้อง `append(os.Environ(), "KEY=VALUE")` เสมอตามตัวอย่างข้างบน มิฉะนั้น subprocess อาจหา `PATH` ไม่เจอจนรันไม่ได้เลย
- `cmd.Dir` กำหนด working directory ของ subprocess โดยไม่กระทบ working directory ของโปรเซส Go หลักเราเองแต่อย่างใด (ต่างจากการ `os.Chdir` ที่เปลี่ยนของทั้งโปรเซส)

---

## 8. ตรวจสอบ Exit Code ด้วย `*exec.ExitError`

เมื่อ subprocess จบการทำงานด้วย **exit code ที่ไม่ใช่ 0** (หมายถึงเกิดข้อผิดพลาดตามธรรมเนียมของ Unix) `Run()`/`Output()`/`CombinedOutput()` จะคืนค่า error ที่มี underlying type เป็น `*exec.ExitError` เราใช้ `errors.As` (จากบทที่ 016 เรื่อง Error Handling ขั้นสูง) เพื่อดึงรายละเอียดออกมา:

```go
package main

import (
	"errors"
	"fmt"
	"os/exec"
)

func main() {
	cmd := exec.Command("sh", "-c", "exit 3")
	err := cmd.Run()

	var exitErr *exec.ExitError
	if errors.As(err, &exitErr) {
		fmt.Println("exit code:", exitErr.ExitCode())
	}
}
```

ผลลัพธ์:

```
exit code: 3
```

### แยกความแตกต่างระหว่าง error สองแบบ

`os/exec` มี error ที่พบบ่อยสองแบบซึ่งความหมายต่างกันโดยสิ้นเชิง:

- **`*exec.Error`**: เกิดตอน **หาโปรแกรมไม่เจอ** เช่น พิมพ์ชื่อคำสั่งผิด หรือคำสั่งนั้นไม่ได้ติดตั้งอยู่ใน `PATH` เลย — โปรแกรมยังไม่ทันได้เริ่มรันด้วยซ้ำ
- **`*exec.ExitError`**: เกิดเมื่อโปรแกรม**รันได้สำเร็จแล้ว** แต่จบด้วย exit code ที่ไม่ใช่ 0 — ตัวโปรแกรมทำงานจริง เพียงแต่รายงานว่าตัวเองล้มเหลว

```go
_, err := exec.Command("this-command-does-not-exist").Output()
// err จะเป็น *exec.Error ("executable file not found in $PATH")

_, err = exec.Command("sh", "-c", "exit 1").Output()
// err จะเป็น *exec.ExitError (โปรแกรมรันได้ แต่ exit code = 1)
```

การแยกสองกรณีนี้ให้ถูกสำคัญมากในงานจริง เช่น เครื่องมือ CI/CD ที่ต้องรายงานให้ต่างกันระหว่าง "เครื่องมือไม่ได้ติดตั้ง" กับ "เครื่องมือรันแล้วแต่ test fail"

---

## 9. ยกเลิก subprocess ที่ค้างด้วย `context.WithTimeout` + `exec.CommandContext`

ปัญหาที่พบบ่อยเวลาเรียก external command คือ **โปรแกรมค้าง (hang) ไม่มีวันจบ** เช่น รอ network response ที่ไม่เคยมาถึง หรือรอ input ที่ไม่มีใครป้อนให้ ถ้าใช้ `exec.Command` ธรรมดา โปรแกรม Go ของเราจะค้างรอไปตลอดไปเช่นกัน

ทางแก้คือใช้ **`exec.CommandContext`** ร่วมกับ **`context.WithTimeout`** (จากบทที่ 032 เรื่อง `context` package) — เมื่อ context ถูกยกเลิกหรือหมดเวลา Go จะส่งสัญญาณ kill ไปยัง subprocess ให้อัตโนมัติ

```go
package main

import (
	"context"
	"fmt"
	"os/exec"
	"time"
)

func main() {
	ctx, cancel := context.WithTimeout(context.Background(), 200*time.Millisecond)
	defer cancel()

	// sleep 5 วินาที แต่ context ตั้ง timeout ไว้แค่ 200ms
	cmd := exec.CommandContext(ctx, "sleep", "5")
	err := cmd.Run()

	fmt.Println("timeout run err:", err)
	fmt.Println("ctx.Err():", ctx.Err())
}
```

ผลลัพธ์:

```
timeout run err: signal: killed
ctx.Err(): context deadline exceeded
```

### หลักการทำงาน

`exec.CommandContext(ctx, name, arg...)` สร้าง `*exec.Cmd` เหมือน `exec.Command` ทุกประการ เพียงแต่ผูก **lifecycle** ของ subprocess เข้ากับ context ที่ส่งเข้าไป เมื่อ context ถูกยกเลิก (ไม่ว่าจะ timeout หรือเรียก `cancel()` เอง) Go จะเรียก `cmd.Process.Kill()` ให้อัตโนมัติทันที ทำให้โปรแกรม Go ของเราไม่มีวันค้างรอ subprocess ที่ไม่ยอมจบเกินเวลาที่กำหนดไว้

Pattern นี้เป็นรากฐานสำคัญมากในเครื่องมือ production ที่เรียก external command เช่น เครื่องมือ CI ที่ต้อง kill test ที่รันนานเกินไป หรือ web server ที่เรียก external tool เพื่อประมวลผล request และต้องไม่ปล่อยให้ request ค้างรอไม่จบสิ้น — เหมือนกับที่เราเคยใช้ `context.WithTimeout` ควบคุม HTTP request ในบทที่ 032

---

## 10. คำเตือนด้านความปลอดภัย: Command Injection

นี่คือหัวข้อที่**สำคัญที่สุดของบทนี้** เพราะการเขียนโค้ดผิดวิธีในเรื่องนี้เป็นสาเหตุของช่องโหว่ความปลอดภัยระดับร้ายแรงที่พบบ่อยมากในโปรแกรมจริง

### รูปแบบที่อันตราย (ห้ามทำ)

```go
// อันตราย! ห้ามเขียนแบบนี้เด็ดขาดเมื่อ userInput มาจากผู้ใช้
func unsafeSearch(userInput string) ([]byte, error) {
	cmd := exec.Command("sh", "-c", "grep "+userInput+" /var/log/app.log")
	return cmd.Output()
}
```

ถ้าผู้ใช้ป้อนค่า `userInput` เป็น:

```
"" ; rm -rf / ;
```

คำสั่งที่ shell มองเห็นจริงๆ จะกลายเป็น:

```bash
sh -c "grep  ; rm -rf / ; /var/log/app.log"
```

เพราะเราสร้าง**สตริงคำสั่งเดียว**แล้วยัดให้ shell (`sh -c`) ตีความเอง shell จะมองเห็นเครื่องหมาย `;`, `|`, `&&`, `` ` `` เป็น**ตัวแบ่งคำสั่ง** ไม่ใช่ข้อมูลธรรมดา ทำให้ผู้โจมตีสามารถ "แทรก" คำสั่งอันตรายใดๆ เข้าไปรันบนเครื่อง server ของเราได้ทันที — นี่คือช่องโหว่ที่เรียกว่า **Command Injection** ซึ่งอยู่ใน OWASP Top 10 มาโดยตลอดและเป็นหนึ่งในช่องโหว่ที่อันตรายที่สุดเพราะให้ผู้โจมตีรันคำสั่งอะไรก็ได้บนเซิร์ฟเวอร์

### รูปแบบที่ปลอดภัย (ทำแบบนี้เสมอ)

```go
// ปลอดภัย: ส่ง argument แยกเป็นแต่ละ string ไม่ผ่าน shell เลย
func safeSearch(userInput string) ([]byte, error) {
	cmd := exec.Command("grep", userInput, "/var/log/app.log")
	return cmd.Output()
}
```

วิธีนี้ปลอดภัยเพราะ `exec.Command("grep", userInput, "/var/log/app.log")` **ไม่ผ่าน shell เลย** — Go เรียก `grep` โดยตรงผ่าน system call (`execve` บน Unix) และส่ง `userInput` เข้าไปเป็น **argument ตัวที่สองแบบดิบๆ** ไม่ว่า `userInput` จะมีอักขระพิเศษอะไรอยู่ข้างในก็ตาม (`;`, `|`, `` ` ``, `$()`) มันจะถูกส่งเข้าไปเป็น**ข้อมูลข้อความล้วนๆ** ให้ `grep` ตีความเป็นคำค้นหาเท่านั้น ไม่มีทางถูกตีความเป็นคำสั่งแยกได้เลย

### หลักการสรุปให้จำง่าย

| รูปแบบ | ปลอดภัยไหม | เหตุผล |
|---|---|---|
| `exec.Command("grep", userInput, file)` | **ปลอดภัย** | argument แต่ละตัวแยกกันชัดเจน ไม่ผ่าน shell interpretation |
| `exec.Command("sh", "-c", "grep "+userInput+" "+file)` | **อันตราย** | สร้างสตริงคำสั่งเดียวให้ shell ตีความ เปิดช่องให้แทรกคำสั่งได้ |
| `exec.Command("sh", "-c", fixedCommand)` (ไม่มี user input ปนอยู่เลย) | ปลอดภัย | คำสั่งคงที่ ไม่มีส่วนที่มาจากภายนอก |

> **กฎเหล็ก**: ถ้า argument ของคำสั่งมาจาก user input (หรือแหล่งข้อมูลที่ไม่น่าเชื่อถือใดๆ เช่น ไฟล์ config ที่แก้ไขได้จากภายนอก, ค่าใน HTTP request) **ห้ามประกอบเป็น shell string เด็ดขาด** ให้ส่งผ่าน `exec.Command(name, arg1, arg2, ...)` แบบแยก argument เสมอ ถ้าจำเป็นต้องใช้ feature ของ shell จริงๆ (เช่น pipe, wildcard) ให้ validate/sanitize input อย่างเข้มงวดมากก่อน หรือพิจารณาเขียน logic นั้นด้วย Go เองแทนการพึ่ง shell ไปเลย

หลักการนี้เชื่อมโยงกับแนวคิดเดียวกับ SQL Injection ที่หลายคนอาจคุ้นเคยมาก่อน — ปัญหาเกิดจากการ**ผสมข้อมูล (data) เข้ากับคำสั่ง (code) โดยไม่แยกจากกันอย่างชัดเจน** ทางแก้คือใช้กลไกที่แยก data ออกจาก code เสมอ ไม่ว่าจะเป็น prepared statement ใน SQL หรือ argument list แบบแยกใน `os/exec`

---

## สรุปสิ่งที่ได้เรียนในบทนี้

- `os/exec` ใช้เรียกโปรแกรมภายนอกจาก Go ผ่าน `exec.Command(name, arg...)`
- `Run()` ไม่คืน output ใดๆ, `Output()` คืนเฉพาะ stdout, `CombinedOutput()` คืนทั้ง stdout และ stderr รวมกัน
- `cmd.Stdout`/`cmd.Stderr` รับค่าเป็น `io.Writer` ทำให้ควบคุมปลายทาง output ได้อย่างอิสระ (buffer, ไฟล์, หน้าจอ) — ต่อยอดจาก interface `io.Writer` ในบทที่ 048
- `cmd.Stdin` รับค่าเป็น `io.Reader` ใช้ป้อนข้อมูลเข้า subprocess ได้
- `cmd.StdoutPipe()` ร่วมกับ `Start()`/`Wait()` ใช้อ่าน output แบบ streaming ทีละบรรทัดขณะ subprocess กำลังทำงานอยู่ ต่างจาก `Run()`/`Output()` ที่รอผลลัพธ์ทั้งหมดก่อนคืนค่าทีเดียว — ต้องเรียก `Wait()` เสมอเพื่อเก็บกวาด resource ของ subprocess
- `cmd.Env` ต้อง `append(os.Environ(), ...)` เพื่อ**เพิ่ม**ตัวแปรใหม่โดยไม่ทำลาย environment เดิม ส่วน `cmd.Dir` กำหนด working directory ของ subprocess แยกจากโปรเซสหลัก
- ใช้ `errors.As(err, &exitErr)` กับ `*exec.ExitError` เพื่ออ่าน exit code และแยกจาก `*exec.Error` (หาโปรแกรมไม่เจอ) ให้ถูกต้อง
- `exec.CommandContext` ร่วมกับ `context.WithTimeout` (จากบทที่ 032) ช่วยยกเลิก subprocess ที่ค้างนานเกินไปโดยอัตโนมัติ
- **ห้ามประกอบ shell command string จาก user input เด็ดขาด** เพราะเปิดช่องให้เกิด Command Injection ให้ส่ง argument แยกผ่าน `exec.Command` เสมอเมื่อข้อมูลมาจากแหล่งที่ไม่น่าเชื่อถือ

## แบบฝึกหัดท้ายบท

1. เขียนโปรแกรมที่เรียก `exec.Command("go", "version")` แล้วพิมพ์ผลลัพธ์ด้วย `Output()` ลองเปลี่ยนเป็นคำสั่งที่ไม่มีอยู่จริง แล้วสังเกตว่า error ที่ได้เป็น `*exec.Error` หรือ `*exec.ExitError` พร้อมอธิบายว่าทำไม
2. เขียนฟังก์ชัน `runWithLog(name string, args ...string) error` ที่รันคำสั่งโดยส่ง `cmd.Stdout`/`cmd.Stderr` ไปที่ไฟล์ log ที่เปิดด้วย `os.Create` แทนการพิมพ์ออกหน้าจอ
3. เขียนโปรแกรมที่ใช้ `cmd.Stdin` ส่งข้อความหลายบรรทัดเข้าไปให้คำสั่ง `sort` แล้วอ่านผลลัพธ์ที่เรียงแล้วกลับมาด้วย `Output()`
4. เขียนฟังก์ชัน `runWithTimeout(timeout time.Duration, name string, args ...string) error` ที่ใช้ `context.WithTimeout` + `exec.CommandContext` ยกเลิกคำสั่งอัตโนมัติถ้าใช้เวลาเกินกำหนด ทดสอบด้วยคำสั่ง `sleep` ที่ตั้งเวลานานกว่า timeout ที่กำหนด
5. เขียนฟังก์ชันสองแบบที่รับ search term จากผู้ใช้แล้วค้นหาในไฟล์ด้วย `grep`: แบบแรกใช้ `exec.Command("sh", "-c", ...)` ประกอบสตริงเอง (ไม่ปลอดภัย) และแบบที่สองใช้ `exec.Command("grep", term, file)` (ปลอดภัย) ทดลองป้อนค่า search term ที่มีเครื่องหมาย `;` ปนอยู่ แล้วสังเกตความแตกต่างของพฤติกรรมทั้งสองแบบ (ทำในสภาพแวดล้อมทดสอบเท่านั้น ห้ามทดลองกับระบบจริง)
6. เขียนโปรแกรมที่ใช้ `cmd.StdoutPipe()` + `cmd.Start()` อ่าน output ของคำสั่งที่พิมพ์ทีละบรรทัดพร้อม `time.Sleep` คั่นระหว่างบรรทัด (เช่นในตัวอย่างหัวข้อที่ 6) แล้วพิมพ์ timestamp กำกับแต่ละบรรทัดที่อ่านได้ด้วย `time.Now()` เพื่อพิสูจน์ว่า output มาถึงแบบ real-time ไม่ใช่รอครบก่อนทั้งหมด ลองเปรียบเทียบกับการใช้ `cmd.Output()` แบบเดิมว่าพฤติกรรมต่างกันอย่างไร

---

**ต่อไป**: [Part 053 — แพ็กเกจ `flag` — CLI Arguments](./053-flag-package.md)
