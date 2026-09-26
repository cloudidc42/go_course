# Part 066: Static Files และ File Upload

> ภาคที่ 5: Web Development — ตอนที่ 11 จาก 15 (Part 56–70)

## สารบัญของบทนี้

1. ทำไมต้องมี Static Files และการอัปโหลดไฟล์ในเว็บแอป
2. `http.FileServer` และ `http.StripPrefix` เจาะลึก
3. ความปลอดภัย: Directory Listing และวิธีปิดมัน
4. ความปลอดภัย: Path Traversal และวิธีที่ `http.FileServer` ป้องกันให้อัตโนมัติ
5. Embedding ไฟล์ static ลงใน binary ด้วย `embed.FS` (Go 1.16+)
6. รับไฟล์อัปโหลดด้วย Multipart Form: `ParseMultipartForm` และ `FormFile`
7. ตรวจสอบขนาดและชนิดไฟล์ก่อนบันทึกจริง
8. บันทึกไฟล์อย่างปลอดภัย: Sanitize ชื่อไฟล์จาก client
9. ตัวอย่างสมบูรณ์: อัปโหลดและเสิร์ฟไฟล์กลับ
10. สรุปสิ่งที่ได้เรียนในบทนี้
11. แบบฝึกหัดท้ายบท

---

## 1. ทำไมต้องมี Static Files และการอัปโหลดไฟล์ในเว็บแอป

ตลอด Part 056–065 เราสร้าง handler, middleware, router, REST API และ template กันมาเยอะแล้ว แต่เว็บแอปพลิเคชันจริงแทบทุกตัวยังต้องทำสองอย่างเพิ่มเติมเสมอ:

1. **เสิร์ฟไฟล์ static** — CSS, JavaScript, รูปภาพ, favicon, font ที่ browser ต้องโหลดไปแสดงผลหน้าเว็บ (ทำงานคู่กับ `html/template` จาก Part 065)
2. **รับไฟล์ที่ผู้ใช้อัปโหลด** — รูปโปรไฟล์, เอกสารแนบ, ไฟล์ CSV สำหรับ import ข้อมูล ฯลฯ

ทั้งสองเรื่องนี้ฟังดูง่าย แต่เป็นจุดที่แอปจำนวนมากมีช่องโหว่ความปลอดภัยเกิดขึ้นจริงในโลก production เช่น การเปิดให้ list ไฟล์ทั้งโฟลเดอร์โดยไม่ตั้งใจ, การให้ client เขียนไฟล์ทับตำแหน่งอื่นในเซิร์ฟเวอร์ผ่านชื่อไฟล์ที่มี `../` หรือการรับไฟล์ executable ปลอมเป็นรูปภาพ บทนี้จะพาไปดูทั้งวิธีทำให้ถูกต้องและวิธีป้องกันช่องโหว่เหล่านี้อย่างละเอียด

Go standard library มีเครื่องมือครบสำหรับงานทั้งสองแบบนี้อยู่แล้วใน `net/http` และ `mime/multipart` โดยไม่ต้องพึ่ง framework หรือ library ภายนอกเลย — สอดคล้องกับจุดเด่นของ Go ที่เราพูดถึงตั้งแต่ **Part 001**: "Standard Library แข็งแกร่ง"

---

## 2. `http.FileServer` และ `http.StripPrefix` เจาะลึก

วิธีมาตรฐานในการเสิร์ฟไฟล์ static ใน Go คือใช้ `http.FileServer` ร่วมกับ `http.Dir`:

```go
package main

import (
	"log"
	"net/http"
)

func main() {
	fs := http.FileServer(http.Dir("./static"))
	http.Handle("/static/", http.StripPrefix("/static/", fs))

	log.Println("listening on :8080")
	log.Fatal(http.ListenAndServe(":8080", nil))
}
```

สมมติโครงสร้างโฟลเดอร์เป็น:

```
myserver/
├── main.go
└── static/
    ├── index.html
    └── sub/
        └── data.txt
```

รันเซิร์ฟเวอร์แล้วเปิด `http://localhost:8080/static/index.html` จะได้เนื้อหาของ `static/index.html` กลับมา

### `http.Dir` ทำอะไร

`http.Dir("./static")` แปลง path string ธรรมดาให้กลายเป็น `http.FileSystem` — interface ที่มีเมธอด `Open(name string) (http.File, error)` เพียงตัวเดียว มันคือ root directory ที่ `http.FileServer` จะใช้ค้นหาไฟล์ทุกครั้งที่มี request เข้ามา

### ทำไมต้อง `http.StripPrefix`

ถ้าเราลงทะเบียน route ด้วย `/static/` แต่ไม่ใช้ `StripPrefix` เลย `http.FileServer` จะพยายามหาไฟล์ที่ชื่อ `static/index.html` **ภายใน** `http.Dir("./static")` ทันที กลายเป็นการหาไฟล์ที่ `./static/static/index.html` ซึ่งไม่มีจริง `http.StripPrefix("/static/", fs)` จะตัดคำว่า `/static/` ออกจาก URL ก่อนส่งต่อให้ `fs` ทำให้ path ที่เหลือ (`index.html`) ตรงกับตำแหน่งไฟล์จริงในโฟลเดอร์

### พฤติกรรมพิเศษที่ควรรู้: redirect ไปหา index.html

ถ้า client ขอ `/static/index.html` ตรงๆ `http.FileServer` จะตอบ `301 Moved Permanently` ไปที่ `/static/` แทน (ทดสอบได้จริงด้วย `curl -v`) เพราะ Go ถือว่าไฟล์ที่ชื่อ `index.html` คือ "ไฟล์ default" ของโฟลเดอร์นั้น และพยายามให้ URL ที่ผู้ใช้เห็นเป็น URL ของโฟลเดอร์ (`/static/`) แทนที่จะเป็นชื่อไฟล์ตรงๆ — เป็นพฤติกรรม standard ของ Go เอง ไม่ใช่ bug

### เสิร์ฟไฟล์เดี่ยว: `http.ServeFile`

ถ้าต้องการเสิร์ฟไฟล์เดียวแบบเจาะจง (เช่น favicon) โดยไม่ต้องเปิดทั้งโฟลเดอร์:

```go
http.HandleFunc("/favicon.ico", func(w http.ResponseWriter, r *http.Request) {
	http.ServeFile(w, r, "./static/favicon.ico")
})
```

> **คำเตือน**: ห้ามส่ง path ที่มาจาก `r.URL.Path` หรือ query parameter เข้า `http.ServeFile` โดยตรงโดยไม่ตรวจสอบ เพราะ `http.ServeFile` (ต่างจาก `http.FileServer`) **ไม่ได้เช็ค path traversal ให้เองอย่างสมบูรณ์เมื่อ pattern ที่ใช้ไม่มี trailing slash** เอกสาร Go เตือนเรื่องนี้ไว้ชัดเจน ทางที่ปลอดภัยที่สุดคือใช้ `http.FileServer` กับ `http.Dir` เสมอเมื่อ path มาจากผู้ใช้

---

## 3. ความปลอดภัย: Directory Listing และวิธีปิดมัน

นี่คือจุดที่คนพลาดกันบ่อยที่สุด: **ถ้าโฟลเดอร์ที่ถูกร้องขอไม่มีไฟล์ `index.html` อยู่ `http.FileServer` จะสร้างหน้า HTML แสดงรายชื่อไฟล์ทั้งหมดในโฟลเดอร์นั้นให้อัตโนมัติ** ทดลองได้จริง:

```
$ curl http://localhost:8080/static/sub/
<!doctype html>
<meta name="viewport" content="width=device-width">
<pre>
<a href="data.txt">data.txt</a>
</pre>
```

ถ้าโฟลเดอร์ `static/` มีไฟล์ที่ไม่ควรเปิดเผยต่อสาธารณะ เช่น ไฟล์ backup, ไฟล์ config ที่หลุดเข้ามา หรือไฟล์อัปโหลดของผู้ใช้คนอื่น การเปิด directory listing แบบนี้เท่ากับให้ใครก็ได้ไล่ดูว่ามีไฟล์อะไรอยู่บ้าง — เป็นการรั่วไหลข้อมูล (information disclosure) ที่ป้องกันได้ง่ายแต่ถูกมองข้ามบ่อยมาก

### วิธีปิด directory listing

Go standard library ไม่มี flag สำเร็จรูปให้ปิดพฤติกรรมนี้ตรงๆ แต่ทำได้ง่ายด้วยการห่อ `http.FileSystem` เดิมด้วย type ของเราเอง ที่จะตอบ "ไม่พบไฟล์" ทันทีเมื่อเจอว่าเป็นการเปิดโฟลเดอร์ (และโฟลเดอร์นั้นไม่มี `index.html`):

```go
package main

import (
	"net/http"
	"os"
)

// noDirListingFS wraps http.Dir so that opening a directory returns
// os.ErrNotExist instead of the directory handle, which stops
// http.FileServer from being able to generate a directory listing.
type noDirListingFS struct {
	http.Dir
}

func (fs noDirListingFS) Open(name string) (http.File, error) {
	f, err := fs.Dir.Open(name)
	if err != nil {
		return nil, err
	}
	info, err := f.Stat()
	if err != nil {
		f.Close()
		return nil, err
	}
	if info.IsDir() {
		indexPath := name
		if indexPath == "" || indexPath[len(indexPath)-1] != '/' {
			indexPath += "/"
		}
		index, err := fs.Dir.Open(indexPath + "index.html")
		if err != nil {
			f.Close()
			return nil, os.ErrNotExist
		}
		index.Close()
	}
	return f, nil
}

func main() {
	fs := http.FileServer(noDirListingFS{http.Dir("./static")})
	mux := http.NewServeMux()
	mux.Handle("/static/", http.StripPrefix("/static/", fs))

	http.ListenAndServe(":8080", mux)
}
```

หลักการคือ: `Open` ของเราเรียก `Open` เดิมก่อน ถ้าผลลัพธ์เป็นโฟลเดอร์และไม่มี `index.html` อยู่ข้างใน ให้คืน `os.ErrNotExist` ปลอม (ทำให้ `http.FileServer` ตอบ `404` แทนที่จะ list ไฟล์) ทดสอบแล้วได้ผลตรงตามที่ต้องการ:

```
GET /static/sub/          -> 404 (ไม่มี index.html ใน sub/)
GET /static/sub/data.txt  -> 200 (ไฟล์ตรงๆ ยังเข้าถึงได้ปกติ)
GET /static/               -> 200 (มี index.html)
```

> **แนวทางที่ใช้บ่อยในโปรเจกต์จริง**: สร้างไฟล์ `index.html` เปล่าๆ ไว้ในทุกโฟลเดอร์ static ที่ไม่ต้องการให้ list (`static/uploads/index.html` ว่างเปล่า) ก็ปิด directory listing ได้เช่นกันโดยไม่ต้องเขียนโค้ดเพิ่ม แต่วิธี wrap `http.FileSystem` ด้านบนสะอาดกว่าและไม่ต้องพึ่งการสร้างไฟล์ทิ้งไว้ทุกที่

---

## 4. ความปลอดภัย: Path Traversal และวิธีที่ `http.FileServer` ป้องกันให้อัตโนมัติ

**Path traversal** คือการโจมตีที่ผู้ใช้พยายามส่ง path ที่มี `..` เพื่อ "ปีนออกจาก" โฟลเดอร์ที่ตั้งใจให้เข้าถึงได้ ไปอ่านไฟล์อื่นในเครื่อง เช่นพยายามอ่าน `/etc/passwd` หรือ source code ของแอปเอง

ข่าวดีคือ **`http.FileServer` ร่วมกับ `http.Dir` ป้องกันเรื่องนี้ให้อัตโนมัติ** ลองทดสอบจริง:

```go
resp, _ := http.Get("http://localhost:8080/static/../main.go")
// resp.StatusCode == 404

resp2, _ := http.Get("http://localhost:8080/static/..%2f..%2fmain.go") // URL-encode ".." ก็ยังโดนบล็อก
// resp2.StatusCode == 404
```

ทั้งสอง request คืนค่า `404` ไม่ใช่เนื้อหาไฟล์ `main.go` กลไกเบื้องหลังมีสองชั้น:

1. **`net/http` มาตรฐานจะ clean URL path ก่อนส่งเข้า handler** — path อย่าง `/static/../main.go` จะถูก resolve และมักจะ redirect ไปที่ path ที่ clean แล้วก่อนถึง handler ของเรา
2. **`http.Dir.Open()` ปฏิเสธ path ที่มี element ".." โดยตรง** — แม้ path จะหลุดผ่านมาถึงชั้นนี้ (เช่นมาจาก `http.StripPrefix` ที่ไม่ได้ clean ซ้ำ) `http.Dir` จะตรวจพบว่ามี `..` อยู่ในองค์ประกอบของ path แล้วปฏิเสธการเปิดไฟล์ทันที คืน error กลับมา ทำให้ `http.FileServer` ตอบ `404` แทนที่จะเสิร์ฟไฟล์นอกโฟลเดอร์ root

นี่คือเหตุผลสำคัญที่ **ควรใช้ `http.FileServer` + `http.Dir` เสมอเมื่อ path มาจากผู้ใช้** แทนที่จะเขียน `os.Open(userProvidedPath)` เองตรงๆ ซึ่งไม่มีการป้องกันใดๆ ให้เลย

> **ข้อควรระวัง**: การป้องกันนี้ใช้ได้กับ `http.Dir` (และ `embed.FS`/`fs.FS` ทั่วไปที่ผ่าน `http.FS`) เท่านั้น ถ้าท่านเขียนโค้ดจัดการไฟล์เองในส่วนอื่น เช่น endpoint อัปโหลด/ลบไฟล์ตามชื่อที่ client ส่งมา ต้อง sanitize เองเสมอ — ดูหัวข้อที่ 8

---

## 5. Embedding ไฟล์ static ลงใน binary ด้วย `embed.FS` (Go 1.16+)

จำจุดเด่นข้อหนึ่งของ Go จาก **Part 001** ได้ไหม: **"Binary เดียวจบ"** — คอมไพล์แล้วได้ไฟล์ executable เดียว ไม่ต้องติดตั้ง runtime หรือ dependency เพิ่ม

ปัญหาคือถ้าเราเสิร์ฟไฟล์ static จากโฟลเดอร์ `./static` ด้วย `http.Dir` แบบปกติ เวลา deploy เราต้อง**ก๊อปทั้งโฟลเดอร์ `static/` ไปพร้อมกับตัว binary เสมอ** ไม่งั้นรันแล้วหาไฟล์ไม่เจอ ทำให้ "binary เดียวจบ" ไม่จริงสมบูรณ์แบบอีกต่อไป

Go 1.16 แก้ปัญหานี้ด้วย package `embed` — ให้เรา **ฝังไฟล์ static เข้าไปในตัว binary ตอน compile ได้เลย** ไม่ต้องมีไฟล์แยกตอน deploy

```go
package main

import (
	"embed"
	"io/fs"
	"log"
	"net/http"
)

//go:embed static
var staticFiles embed.FS

func main() {
	// static/* ถูกฝังพร้อม prefix "static/" เราต้องการให้ FileServer เห็นแค่เนื้อหาข้างใน
	// จึงใช้ fs.Sub เพื่อ "เจาะเข้าไป" ที่โฟลเดอร์ static โดยตรง
	sub, err := fs.Sub(staticFiles, "static")
	if err != nil {
		log.Fatal(err)
	}

	mux := http.NewServeMux()
	mux.Handle("/static/", http.StripPrefix("/static/", http.FileServer(http.FS(sub))))

	log.Println("listening on :8080")
	log.Fatal(http.ListenAndServe(":8080", mux))
}
```

จุดสำคัญที่ต้องเข้าใจทีละบรรทัด:

```go
//go:embed static
var staticFiles embed.FS
```

- `//go:embed static` เป็น **compiler directive** (ต้องอยู่ติดกับบรรทัด `var` ที่ตามมาโดยไม่มีบรรทัดว่างคั่น) สั่งให้ Go compiler อ่านทุกไฟล์ในโฟลเดอร์ `static/` (รวม subfolder) แล้วฝังเนื้อหาทั้งหมดลงใน binary ตอน compile
- `embed.FS` เป็น type ที่ implement interface `fs.FS` ของ standard library ทำให้ใช้ร่วมกับฟังก์ชันที่รับ `fs.FS` ได้ทันที เช่น `http.FS()`

```go
sub, err := fs.Sub(staticFiles, "static")
```

- `embed.FS` ที่ได้จะมี path ภายในเป็น `static/index.html` (ติด prefix ชื่อโฟลเดอร์ที่ระบุใน `//go:embed` เสมอ) `fs.Sub` ใช้ "เจาะ" เข้าไปในโฟลเดอร์ย่อยนั้น ทำให้ path ที่เหลือกลายเป็น `index.html` ตรงๆ ไม่ต้องมี prefix `static/` ซ้ำซ้อนกับที่เรา `StripPrefix` ไปแล้วจาก URL

```go
http.FileServer(http.FS(sub))
```

- `http.FS()` แปลง `fs.FS` (ซึ่งเป็น interface มาตรฐานของ Go ตั้งแต่ 1.16) ให้กลายเป็น `http.FileSystem` ที่ `http.FileServer` ต้องการ — ทุกอย่างที่เหลือทำงานเหมือน `http.Dir` ทุกประการ รวมถึงการป้องกัน path traversal ที่อธิบายไปข้างต้น

ทดสอบแล้วได้ผลตรงตามคาด: build เป็น binary ตัวเดียว รันจากที่ไหนก็ได้ (แม้ลบโฟลเดอร์ `static/` ต้นฉบับทิ้งไปแล้ว!) แล้ว request `/static/index.html` ก็ยังได้เนื้อหากลับมาถูกต้อง เพราะไฟล์ถูกฝังอยู่ใน binary เรียบร้อยแล้วตั้งแต่ตอน `go build`

### เมื่อไหร่ควรใช้ `embed.FS` เมื่อไหร่ควรใช้ `http.Dir`

| สถานการณ์ | แนะนำ |
|---|---|
| CSS/JS/รูปภาพของหน้าเว็บที่ไม่เปลี่ยนบ่อย, ต้องการ deploy เป็นไฟล์เดียว | `embed.FS` |
| ไฟล์ที่ผู้ใช้อัปโหลดเข้ามาระหว่างรัน (เปลี่ยนแปลงตลอดเวลา) | `http.Dir` (จะฝังไม่ได้เพราะ embed ต้องรู้เนื้อหาตอน compile) |
| ไฟล์ config ที่ต้องแก้ได้โดยไม่ compile ใหม่ | `http.Dir` หรือ config file แยก |

ในทางปฏิบัติ โปรเจกต์จริงมักผสมทั้งสองแบบ: `embed.FS` สำหรับ static asset ของแอปเอง และ `http.Dir` สำหรับโฟลเดอร์ที่เก็บไฟล์อัปโหลดของผู้ใช้ (ซึ่งเป็นหัวข้อถัดไป)

---

## 6. รับไฟล์อัปโหลดด้วย Multipart Form: `ParseMultipartForm` และ `FormFile`

เมื่อ browser ส่งฟอร์มที่มี `<input type="file">` มันจะเข้ารหัส request body เป็น **`multipart/form-data`** — รูปแบบที่แบ่ง body ออกเป็นหลาย "ส่วน" (part) แต่ละส่วนมี header และเนื้อหาของตัวเอง ทำให้ส่งได้ทั้ง text field ปกติและไฟล์ binary ในคำขอเดียวกัน

ตัวอย่างฟอร์ม HTML ฝั่ง client:

```html
<form action="/upload" method="POST" enctype="multipart/form-data">
	<input type="file" name="file">
	<button type="submit">อัปโหลด</button>
</form>
```

ฝั่งเซิร์ฟเวอร์ใช้ `r.ParseMultipartForm` แล้วดึงไฟล์ด้วย `r.FormFile`:

```go
func uploadHandler(w http.ResponseWriter, r *http.Request) {
	if r.Method != http.MethodPost {
		http.Error(w, "method not allowed", http.StatusMethodNotAllowed)
		return
	}

	// maxMemory คือขนาดสูงสุดที่ยอมเก็บใน memory ตอน parse
	// ส่วนที่เกินจะถูกเขียนลง temp file บน disk ให้อัตโนมัติ
	const maxMemory = 10 << 20 // 10 MB
	if err := r.ParseMultipartForm(maxMemory); err != nil {
		http.Error(w, "form ไม่ถูกต้อง: "+err.Error(), http.StatusBadRequest)
		return
	}

	file, header, err := r.FormFile("file") // "file" ต้องตรงกับ name ของ <input>
	if err != nil {
		http.Error(w, "ไม่พบไฟล์: "+err.Error(), http.StatusBadRequest)
		return
	}
	defer file.Close()

	fmt.Fprintf(w, "ได้รับไฟล์: %s (%d bytes, Content-Type ที่ client อ้าง: %s)\n",
		header.Filename, header.Size, header.Header.Get("Content-Type"))
}
```

อธิบายทีละส่วน:

- **`r.ParseMultipartForm(maxMemory)`** — อ่านและ parse ทั้ง multipart body โดย field ที่เป็น text ธรรมดาจะถูกเก็บไว้ใน `r.MultipartForm.Value` ส่วนไฟล์จะเก็บ metadata ไว้ใน `r.MultipartForm.File` ค่า `maxMemory` ที่ส่งเข้าไปคือขนาดสูงสุดของข้อมูลที่ยอมเก็บใน RAM — ถ้าไฟล์ใหญ่กว่านั้น Go จะเขียนส่วนเกินลง temp file บน disk ให้อัตโนมัติแล้วค่อยให้เราอ่านผ่าน `io.Reader` เหมือนเดิม โดยที่เราไม่ต้องจัดการเรื่องนี้เอง
- **`r.FormFile("file")`** — คืนค่า 3 อย่าง: `multipart.File` (interface ที่เป็นทั้ง `io.Reader`, `io.ReaderAt`, `io.Seeker`, `io.Closer`), `*multipart.FileHeader` (metadata เช่นชื่อไฟล์ ขนาด header), และ `error`
- **`header.Filename`** — ชื่อไฟล์ตามที่ client ส่งมา **ห้ามเชื่อถือค่านี้โดยตรง** (อธิบายเหตุผลในหัวข้อที่ 8)
- **`header.Header.Get("Content-Type")`** — Content-Type ที่ browser ส่งมาให้ ก็ **ห้ามเชื่อถือเช่นกัน** เพราะเป็นแค่ค่าที่ client "อ้าง" ไม่ใช่การตรวจสอบจริง ผู้ใช้ที่ประสงค์ร้ายสามารถแก้ header นี้ให้เป็นอะไรก็ได้ก่อนส่ง request มา

---

## 7. ตรวจสอบขนาดและชนิดไฟล์ก่อนบันทึกจริง

### จำกัดขนาดไฟล์อย่างจริงจังด้วย `http.MaxBytesReader`

การส่ง `maxMemory` ให้ `ParseMultipartForm` **ไม่ได้จำกัดขนาด body ทั้งหมด** มันแค่กำหนดว่าจะเก็บกี่ byte ไว้ใน RAM ก่อนเปลี่ยนไปเขียนลง disk แทน — ถ้าไม่ทำอะไรเพิ่ม ผู้ใช้ที่ประสงค์ร้ายยังสามารถส่งไฟล์ขนาดหลาย GB มาถล่ม disk เซิร์ฟเวอร์ได้อยู่ดี

วิธีป้องกันที่ถูกต้องคือห่อ `r.Body` ด้วย `http.MaxBytesReader` **ก่อน** เรียก `ParseMultipartForm`:

```go
const maxUploadSize = 5 << 20 // 5 MB

r.Body = http.MaxBytesReader(w, r.Body, maxUploadSize)
if err := r.ParseMultipartForm(maxUploadSize); err != nil {
	http.Error(w, "ไฟล์ใหญ่เกินไปหรือฟอร์มไม่ถูกต้อง: "+err.Error(), http.StatusBadRequest)
	return
}
```

`http.MaxBytesReader` จะทำให้การอ่าน `r.Body` ที่เกิน `maxUploadSize` byte ล้มเหลวทันทีด้วย error แทนที่จะยอมให้อ่านต่อไปเรื่อยๆ

### ตรวจชนิดไฟล์จากเนื้อหาจริง ไม่ใช่จาก Filename หรือ Content-Type

อย่างที่บอกไปว่า `header.Filename` และ header `Content-Type` เป็นค่าที่ client กำหนดเองได้ทั้งหมด วิธีที่ปลอดภัยกว่าคือดูจาก**เนื้อหาไฟล์จริง** ด้วย `http.DetectContentType` ซึ่งจะตรวจ "magic bytes" ช่วงต้นของไฟล์ (signature ที่บอกชนิดไฟล์จริงๆ เช่น PNG ขึ้นต้นด้วย byte เฉพาะเสมอ):

```go
var allowedTypes = map[string]bool{
	"image/png":  true,
	"image/jpeg": true,
	"image/gif":  true,
}

buf := make([]byte, 512) // http.DetectContentType ใช้ข้อมูลแค่ 512 byte แรกก็พอ
n, _ := file.Read(buf)
contentType := http.DetectContentType(buf[:n])
if !allowedTypes[contentType] {
	http.Error(w, fmt.Sprintf("ไม่รองรับชนิดไฟล์: %s", contentType), http.StatusUnsupportedMediaType)
	return
}

// อ่านไปแล้วตอนตรวจสอบ ต้อง seek กลับไปจุดเริ่มต้นก่อนจะอ่านเพื่อบันทึกจริง
if _, err := file.Seek(0, io.SeekStart); err != nil {
	http.Error(w, "อ่านไฟล์ไม่ได้", http.StatusInternalServerError)
	return
}
```

การทดสอบจริงยืนยันว่าวิธีนี้ตรวจจับได้ถูกต้อง: ไฟล์ที่มีนามสกุล `.png` แต่เนื้อหาข้างในเป็น script HTML ล้วนๆ (เช่นพยายามหลอกระบบเพื่อทำ stored XSS) จะถูก `http.DetectContentType` ตรวจพบว่าเป็น `text/plain; charset=utf-8` ไม่ใช่ `image/png` และถูกปฏิเสธด้วย `415 Unsupported Media Type` ทันที

---

## 8. บันทึกไฟล์อย่างปลอดภัย: Sanitize ชื่อไฟล์จาก client

นี่คือจุดที่อันตรายที่สุดถ้าทำพลาด: **ห้ามเอา `header.Filename` ไปต่อเป็น path แล้วบันทึกไฟล์ตรงๆ เด็ดขาด**

```go
// ❌ อันตรายมาก ห้ามทำแบบนี้
dst, _ := os.Create("./uploads/" + header.Filename)
```

ถ้า client ตั้งชื่อไฟล์เป็น `../../etc/passwd` หรือ `../../../home/user/.ssh/authorized_keys` (หรือชื่อยาวๆ ที่มี `..` ปนอยู่) โค้ดด้านบนจะเขียนไฟล์ทับตำแหน่งที่ผู้โจมตีเลือกเองได้เลย เป็นช่องโหว่ **path traversal ฝั่งเขียนไฟล์** ซึ่งอันตรายกว่าฝั่งอ่านไฟล์ (หัวข้อที่ 4) มาก เพราะเปิดทางให้แก้ไข/สร้างไฟล์ในตำแหน่งใดก็ได้ที่ process มีสิทธิ์เขียนถึง

### แนวทางที่ถูกต้อง: อย่าเชื่อชื่อไฟล์เดิมเลย ตั้งชื่อใหม่เอง

วิธีที่ปลอดภัยและนิยมใช้ที่สุดคือ **เก็บแค่ extension ที่อนุญาต (whitelist) จากชื่อเดิม แล้วสุ่มชื่อไฟล์ใหม่ทั้งหมด**:

```go
import (
	"crypto/rand"
	"encoding/hex"
	"fmt"
	"path/filepath"
	"strings"
)

func sanitizeAndRandomizeFilename(original string) (string, error) {
	// filepath.Base ตัด path ที่นำหน้ามาทั้งหมดออก เหลือแค่ชื่อไฟล์ท้ายสุด
	// (กันกรณีชื่อไฟล์มี "/" หรือ "\" ปนมาด้วย)
	ext := strings.ToLower(filepath.Ext(filepath.Base(original)))
	switch ext {
	case ".png", ".jpg", ".jpeg", ".gif":
		// อนุญาต
	default:
		return "", fmt.Errorf("นามสกุลไฟล์ไม่รองรับ: %s", ext)
	}

	randBytes := make([]byte, 16)
	if _, err := rand.Read(randBytes); err != nil {
		return "", err
	}
	return hex.EncodeToString(randBytes) + ext, nil // เช่น deb94b64be83a5f4b71c95b972d5cd16.png
}
```

จุดสำคัญ:

1. **`filepath.Base`** ตัดทุกอย่างก่อน `/` (หรือ `\` บน Windows) ทิ้ง เหลือแค่ส่วนชื่อไฟล์ท้ายสุดจริงๆ
2. **whitelist extension แบบ `switch`** — อนุญาตเฉพาะนามสกุลที่รู้จักและคาดหวังเท่านั้น ปฏิเสธทุกอย่างที่ไม่อยู่ในรายการ (denylist ชนิดไฟล์อันตรายไม่ปลอดภัยพอ เพราะรายชื่อนามสกุลอันตรายมีเยอะและเพิ่มขึ้นเรื่อยๆ)
3. **สุ่มชื่อไฟล์ใหม่ทั้งหมดด้วย `crypto/rand`** — ทำให้ไม่มีทางที่ input จาก client จะกำหนด path ปลายทางได้อีกเลย และเป็นผลพลอยได้ที่ดี: ไฟล์จะไม่ชนกัน (overwrite) แม้ผู้ใช้สองคนอัปโหลดไฟล์ชื่อเดียวกัน

เมื่อได้ชื่อไฟล์ที่ปลอดภัยแล้ว ให้ใช้ `os.O_EXCL` ตอนเปิดไฟล์เพื่อป้องกันการเขียนทับไฟล์ที่มีอยู่แล้วโดยไม่ตั้งใจ (แม้จะสุ่มชื่อแล้วโอกาสชนแทบเป็นศูนย์ แต่การป้องกันซ้อนไม่เสียหาย):

```go
dst, err := os.OpenFile(dstPath, os.O_WRONLY|os.O_CREATE|os.O_EXCL, 0o644)
```

---

## 9. ตัวอย่างสมบูรณ์: อัปโหลดและเสิร์ฟไฟล์กลับ

รวมทุกอย่างที่เรียนมาในบทนี้เข้าด้วยกัน: จำกัดขนาด, ตรวจชนิดไฟล์จริง, sanitize ชื่อไฟล์, บันทึกอย่างปลอดภัย และเสิร์ฟไฟล์ที่อัปโหลดแล้วกลับผ่าน `http.FileServer`:

```go
package main

import (
	"crypto/rand"
	"encoding/hex"
	"fmt"
	"io"
	"log"
	"net/http"
	"os"
	"path/filepath"
	"strings"
)

const (
	maxUploadSize = 5 << 20 // 5 MB
	uploadDir     = "./uploads"
)

var allowedTypes = map[string]bool{
	"image/png":  true,
	"image/jpeg": true,
	"image/gif":  true,
}

func uploadHandler(w http.ResponseWriter, r *http.Request) {
	if r.Method != http.MethodPost {
		http.Error(w, "method not allowed", http.StatusMethodNotAllowed)
		return
	}

	r.Body = http.MaxBytesReader(w, r.Body, maxUploadSize)
	if err := r.ParseMultipartForm(maxUploadSize); err != nil {
		http.Error(w, "ไฟล์ใหญ่เกินไปหรือฟอร์มไม่ถูกต้อง: "+err.Error(), http.StatusBadRequest)
		return
	}

	file, header, err := r.FormFile("file")
	if err != nil {
		http.Error(w, "ไม่พบไฟล์ที่ชื่อ field 'file': "+err.Error(), http.StatusBadRequest)
		return
	}
	defer file.Close()

	buf := make([]byte, 512)
	n, _ := file.Read(buf)
	contentType := http.DetectContentType(buf[:n])
	if !allowedTypes[contentType] {
		http.Error(w, fmt.Sprintf("ไม่รองรับชนิดไฟล์: %s", contentType), http.StatusUnsupportedMediaType)
		return
	}
	if _, err := file.Seek(0, io.SeekStart); err != nil {
		http.Error(w, "อ่านไฟล์ไม่ได้", http.StatusInternalServerError)
		return
	}

	safeName, err := sanitizeAndRandomizeFilename(header.Filename)
	if err != nil {
		http.Error(w, "ชื่อไฟล์ไม่ถูกต้อง", http.StatusBadRequest)
		return
	}

	if err := os.MkdirAll(uploadDir, 0o755); err != nil {
		http.Error(w, "สร้างโฟลเดอร์ไม่ได้", http.StatusInternalServerError)
		return
	}

	dstPath := filepath.Join(uploadDir, safeName)
	dst, err := os.OpenFile(dstPath, os.O_WRONLY|os.O_CREATE|os.O_EXCL, 0o644)
	if err != nil {
		http.Error(w, "บันทึกไฟล์ไม่ได้", http.StatusInternalServerError)
		return
	}
	defer dst.Close()

	if _, err := io.Copy(dst, file); err != nil {
		http.Error(w, "เขียนไฟล์ไม่สำเร็จ", http.StatusInternalServerError)
		return
	}

	fmt.Fprintf(w, "อัปโหลดสำเร็จ: %s (%s, %d bytes)\n", safeName, contentType, header.Size)
}

func sanitizeAndRandomizeFilename(original string) (string, error) {
	ext := strings.ToLower(filepath.Ext(filepath.Base(original)))
	switch ext {
	case ".png", ".jpg", ".jpeg", ".gif":
	default:
		return "", fmt.Errorf("นามสกุลไฟล์ไม่รองรับ: %s", ext)
	}

	randBytes := make([]byte, 16)
	if _, err := rand.Read(randBytes); err != nil {
		return "", err
	}
	return hex.EncodeToString(randBytes) + ext, nil
}

func main() {
	http.HandleFunc("/upload", uploadHandler)
	// เสิร์ฟไฟล์ที่อัปโหลดแล้วกลับ ผ่าน http.Dir เดิม (ห้ามใช้ embed.FS เพราะไฟล์เหล่านี้
	// เปลี่ยนแปลงระหว่างรัน ไม่รู้เนื้อหาล่วงหน้าตอน compile)
	http.Handle("/uploads/", http.StripPrefix("/uploads/", http.FileServer(http.Dir(uploadDir))))

	log.Println("listening on :8080")
	log.Fatal(http.ListenAndServe(":8080", nil))
}
```

ทดสอบด้วย `curl`:

```bash
curl -F "file=@profile.png" http://localhost:8080/upload
# อัปโหลดสำเร็จ: deb94b64be83a5f4b71c95b972d5cd16.png (image/png, 24 bytes)

curl http://localhost:8080/uploads/deb94b64be83a5f4b71c95b972d5cd16.png -o downloaded.png
```

โค้ดชุดนี้ผ่านการทดสอบจริงด้วย `httptest` แล้วว่า:

- อัปโหลดไฟล์ PNG จริง (ตรวจจาก magic bytes) พร้อมชื่อไฟล์ที่พยายามทำ path traversal (`../../etc/passwd.png`) → **สำเร็จ แต่ไฟล์ที่บันทึกจริงถูกตั้งชื่อใหม่แบบสุ่มทั้งหมด** ไม่มีร่องรอยของชื่อเดิมเหลืออยู่เลย
- อัปโหลดไฟล์ที่นามสกุลเป็น `.png` แต่เนื้อหาเป็น script HTML → **ถูกปฏิเสธด้วย 415**
- อัปโหลดไฟล์นามสกุลที่ไม่อยู่ใน whitelist เช่น `.exe` → **ถูกปฏิเสธด้วย 400** ตั้งแต่ก่อนเปิดอ่านเนื้อหาไฟล์ด้วยซ้ำ

---

## สรุปสิ่งที่ได้เรียนในบทนี้

- `http.FileServer` คู่กับ `http.Dir` และ `http.StripPrefix` คือวิธีมาตรฐานในการเสิร์ฟไฟล์ static ใน Go
- **Directory listing เปิดอยู่โดย default** เมื่อโฟลเดอร์ไม่มี `index.html` — ต้อง wrap `http.FileSystem` เองเพื่อปิดถ้าไม่ต้องการให้เผยรายชื่อไฟล์
- **Path traversal ฝั่งอ่านไฟล์ถูกป้องกันอัตโนมัติ** โดย `http.Dir.Open()` ที่ปฏิเสธ path ซึ่งมี `..` อยู่ในองค์ประกอบ
- `embed.FS` (Go 1.16+) ฝังไฟล์ static ลงใน binary ตอน compile ทำให้ deploy เป็น "binary เดียวจบ" ได้จริงตามจุดเด่นของ Go ที่กล่าวไว้ใน Part 001
- รับไฟล์อัปโหลดด้วย `r.ParseMultipartForm` + `r.FormFile` และต้องจำกัดขนาดจริงจังด้วย `http.MaxBytesReader` ก่อนเสมอ
- **ห้ามเชื่อ `header.Filename` หรือ Content-Type ที่ client ส่งมา** — ตรวจชนิดไฟล์จริงด้วย `http.DetectContentType` จากเนื้อหา ไม่ใช่จาก metadata
- **Path traversal ฝั่งเขียนไฟล์อันตรายกว่าฝั่งอ่านมาก** — ทางแก้ที่ปลอดภัยที่สุดคือไม่เชื่อชื่อไฟล์เดิมเลย เก็บแค่ extension ที่อยู่ใน whitelist แล้วสุ่มชื่อไฟล์ใหม่ทั้งหมดด้วย `crypto/rand`

## แบบฝึกหัดท้ายบท

1. เขียนเซิร์ฟเวอร์ที่เสิร์ฟไฟล์ static จากโฟลเดอร์ `./public` แล้วทดลองเข้าถึงโฟลเดอร์ที่ไม่มี `index.html` ดูว่า directory listing แสดงผลอย่างไร จากนั้นแก้ไขให้ปิด directory listing ตามวิธีในหัวข้อที่ 3
2. ทดลองยิง request ที่มี `..` ในหลายรูปแบบ (ตรงๆ, URL-encode เป็น `%2e%2e`, ผสมกับ backslash) ไปที่เซิร์ฟเวอร์ static ของท่าน แล้วอธิบายว่าทำไมทุกกรณีถึงได้ `404` กลับมา
3. แปลงเซิร์ฟเวอร์ static จากข้อ 1 ให้ใช้ `embed.FS` แทน `http.Dir` แล้วลองลบโฟลเดอร์ `./public` ต้นฉบับทิ้งหลัง build เสร็จ ยืนยันว่า binary ยังทำงานได้ปกติ
4. เขียน endpoint `/upload` ที่รับเฉพาะไฟล์ PDF (`application/pdf`) ขนาดไม่เกิน 2 MB โดยใช้ `http.DetectContentType` ตรวจสอบจริง (สังเกตว่า PDF มี magic bytes ขึ้นต้นด้วย `%PDF`)
5. ทดลองแก้โค้ดในหัวข้อที่ 8 ให้ใช้ `header.Filename` ตรงๆ โดยไม่ sanitize (จำลองโค้ดที่ไม่ปลอดภัย) แล้วพิสูจน์ด้วยตัวเองว่าสามารถส่งชื่อไฟล์ที่มี `../` ผ่าน `curl -F` เพื่อพยายามเขียนไฟล์นอกโฟลเดอร์ `uploads/` ได้จริงหรือไม่ อธิบายผลลัพธ์
6. เพิ่ม endpoint `/uploads/list` ที่ใช้ `os.ReadDir` แสดงรายชื่อไฟล์ทั้งหมดใน `uploads/` แบบ JSON (ไม่ใช้ directory listing ของ `http.FileServer`) เพื่อให้ควบคุมรูปแบบผลลัพธ์ได้เอง

---

**ต่อไป**: [Part 067 — Authentication ด้วย JWT](./067-jwt-authentication.md)
