# Part 065: Templates ด้วย `html/template`

> ภาคที่ 5: Web Development — ตอนที่ 10 จาก 15 (Part 56–70)

## สารบัญของบทนี้

1. ทำไมต้องมี Template Engine ทั้งที่สร้าง JSON API มาตลอด (Part 056-064)
2. `text/template` vs `html/template`: ต่างกันตรงไหน
3. สาธิตจริง: XSS เมื่อใช้ `text/template` render HTML
4. Contextual Escaping คืออะไร ทำงานอย่างไรเบื้องหลัง
5. Syntax พื้นฐาน: `{{.Field}}`, `{{if}}`, `{{range}}`
6. `{{define}}` และ `{{template}}`: Layout และ Partial
7. โหลด Template จากไฟล์: `ParseFiles` vs `ParseGlob`
8. ส่งข้อมูลเข้า Template: struct, map, และ `Execute`/`ExecuteTemplate`
9. Template Functions: `FuncMap`
10. ตัวอย่างเต็ม: หน้าเว็บ Todo List พร้อม Layout + Partial จาก HTTP Server จริง
11. ข้อควรระวังและแนวทางปฏิบัติที่ดี
12. สรุปสิ่งที่ได้เรียนในบทนี้
13. แบบฝึกหัดท้ายบท

---

## 1. ทำไมต้องมี Template Engine ทั้งที่สร้าง JSON API มาตลอด (Part 056-064)

ตั้งแต่ **Part 056** จนถึง **Part 064** เราสร้าง HTTP server ที่ตอบกลับเป็น **JSON** เกือบทั้งหมด เพราะเป้าหมายคือ REST API ที่ให้ frontend แยกต่างหาก (เช่น React, Vue) หรือ mobile app เรียกใช้ แต่ Go เองก็ใช้สร้างเว็บแอปที่ **render HTML ฝั่งเซิร์ฟเวอร์โดยตรง** (server-side rendering) ได้เช่นกัน โดยไม่ต้องพึ่ง JavaScript framework แยกเลย — แนวทางนี้ยังเป็นที่นิยมมากสำหรับเว็บที่เน้นเนื้อหา (content-heavy), แอปภายในองค์กร (internal tools), หรือ dashboard ที่ไม่ต้องการความซับซ้อนของ SPA (Single Page Application)

การสร้าง HTML string เองด้วยมือผ่าน `fmt.Sprintf` หรือ string concatenation ธรรมดาแบบนี้:

```go
// อย่าทำแบบนี้ในโปรเจกต์จริง!
html := fmt.Sprintf("<h1>สวัสดี %s</h1>", userName)
w.Write([]byte(html))
```

มีปัญหาใหญ่สองข้อ:

1. **อ่านและ maintain ยาก** — พอ HTML ซับซ้อนขึ้น (มี loop, condition, ซ้อนหลายชั้น) โค้ด `fmt.Sprintf` จะกลายเป็นฝันร้ายทันที
2. **อันตรายด้านความปลอดภัย (Cross-Site Scripting / XSS)** — ถ้า `userName` มาจาก input ของผู้ใช้ (เช่น กรอกในฟอร์ม) และมีใครใส่ `<script>...</script>` เข้ามา โค้ดข้างบนจะแทรก script นั้นลงในหน้าเว็บโดยตรงแบบไม่มีการป้องกันใดๆ เลย

Go standard library มี package `text/template` และ `html/template` มาแก้ปัญหาทั้งสองข้อนี้พร้อมกัน — บทนี้จะเจาะลึกว่าทำไม **ต้องใช้ `html/template` เท่านั้นสำหรับ web output** ห้ามใช้ `text/template` แม้ syntax จะเหมือนกันทุกตัวอักษรก็ตาม

---

## 2. `text/template` vs `html/template`: ต่างกันตรงไหน

Go มี template package สองตัวที่ syntax เหมือนกันทุกประการ (`{{.Field}}`, `{{if}}`, `{{range}}` ฯลฯ ใช้ได้เหมือนกันทั้งคู่) แต่มีจุดต่างที่สำคัญมากหนึ่งจุด:

| | `text/template` | `html/template` |
|---|---|---|
| ใช้สำหรับ | สร้าง text ทั่วไป (config file, email แบบ plain text, CLI output) | สร้าง **HTML output** เท่านั้น |
| Escaping | **ไม่มี** — ใส่ข้อมูลลงไปตรงๆ ตามที่เป็น | **มีอัตโนมัติ** — escape ค่าตาม context ที่มันปรากฏอยู่ (HTML body, attribute, URL, JavaScript, CSS) |
| ปลอดภัยจาก XSS หรือไม่ | **ไม่** — ถ้าเอาไป render เป็น HTML คือช่องโหว่ทันที | **ใช่** — escape ให้อัตโนมัติ ป้องกัน XSS โดย default |
| API | เหมือนกันทุกจุด (`Parse`, `Execute`, `FuncMap` ฯลฯ) | เหมือนกันทุกจุด (เป็น superset ของ `text/template`) |

กฎที่ต้องจำจากบทนี้มีประโยคเดียว:

> **ถ้า output จะถูกส่งไปแสดงในเบราว์เซอร์เป็น HTML ต้องใช้ `html/template` เท่านั้น ไม่มีข้อยกเว้น**

`text/template` เหมาะกับกรณีที่ output ไม่ใช่ HTML เลย เช่น generate ไฟล์ `.yaml`, `.env`, หรือ code generator (เราจะเจอการใช้ `text/template` แบบนี้จริงๆ ใน **Part 108: Go Modules Best Practices** ตอนพูดถึงเครื่องมือ generate โค้ด) — แต่สำหรับหัวข้อ **web development** ของภาคนี้ คำตอบคือ `html/template` เสมอ

---

## 3. สาธิตจริง: XSS เมื่อใช้ `text/template` render HTML

มาดูกันชัดๆ ว่าความต่างนี้มีผลจริงแค่ไหน ด้วยโค้ดที่รันได้จริง — สมมติสถานการณ์เว็บที่แสดงความคิดเห็นของผู้ใช้ และมีผู้ใช้ประสงค์ร้าย (`mallory`) พยายามฝัง `<script>` เข้าไปในช่องแสดงความคิดเห็น:

```go
package main

import (
	"fmt"
	"html/template"
	"os"
	texttemplate "text/template"
)

type Comment struct {
	Author string
	Body   string
}

func main() {
	c := Comment{
		Author: "mallory",
		Body:   `<script>alert('stolen cookie: ' + document.cookie)</script>`,
	}

	const tmplSrc = `<p><b>{{.Author}}</b> says: {{.Body}}</p>`

	fmt.Println("=== text/template (ไม่ escape ให้ — อันตรายถ้าใช้ render HTML) ===")
	tt := texttemplate.Must(texttemplate.New("comment").Parse(tmplSrc))
	tt.Execute(os.Stdout, c)
	fmt.Println()
	fmt.Println()

	fmt.Println("=== html/template (escape ให้อัตโนมัติตาม context) ===")
	ht := template.Must(template.New("comment").Parse(tmplSrc))
	ht.Execute(os.Stdout, c)
	fmt.Println()
}
```

สังเกตว่า **template source (`tmplSrc`) เขียนเหมือนกันทุกตัวอักษร** ความต่างอยู่ที่ package ที่ใช้ parse เท่านั้น (`texttemplate.New` vs `template.New` ที่ import มาจาก `html/template`)

รันด้วย `go run main.go` — ผลลัพธ์จริงจากการรัน:

```
=== text/template (ไม่ escape ให้ — อันตรายถ้าใช้ render HTML) ===
<p><b>mallory</b> says: <script>alert('stolen cookie: ' + document.cookie)</script></p>

=== html/template (escape ให้อัตโนมัติตาม context) ===
<p><b>mallory</b> says: &lt;script&gt;alert(&#39;stolen cookie: &#39; &#43; document.cookie)&lt;/script&gt;</p>
```

ผลลัพธ์พูดแทนทุกอย่าง:

- **`text/template`**: HTML ที่ได้มี `<script>...</script>` อยู่ตรงๆ ถ้า string นี้ถูกส่งไปที่เบราว์เซอร์จริง สคริปต์นั้น **จะถูกรันทันที** — ผู้โจมตีสามารถขโมย cookie, session token, หรือทำอะไรก็ได้ในนามของผู้ใช้ที่เปิดหน้านั้น นี่คือ **Cross-Site Scripting (XSS)** ช่องโหว่ด้านความปลอดภัยเว็บที่ร้ายแรงและพบบ่อยที่สุดอันดับต้นๆ ของโลก
- **`html/template`**: `<script>` ถูกแปลงเป็น `&lt;script&gt;` โดยอัตโนมัติ — เมื่อเบราว์เซอร์ render ออกมา ผู้ใช้จะเห็น**ข้อความ** `<script>alert(...)</script>` ตัวโตๆ บนหน้าเว็บเฉยๆ (เป็นแค่ text ธรรมดา) ไม่มีสคริปต์ไหนถูกรันเลย

นี่คือเหตุผลที่แท้จริงว่าทำไม Go ถึงแยก package สองตัวนี้ออกจากกัน แทนที่จะรวมเป็นตัวเดียวแล้วให้ escape เป็น option — **การบังคับให้เลือก package ตั้งแต่ตอน import คือการออกแบบที่ทำให้ความปลอดภัยเป็นค่าเริ่มต้น (secure by default)** ไม่ใช่สิ่งที่ผู้เขียนโค้ดต้อง "จำ" ที่จะเปิดใช้งานเอง

---

## 4. Contextual Escaping คืออะไร ทำงานอย่างไรเบื้องหลัง

จุดที่ทำให้ `html/template` ฉลาดกว่าการ `html.EscapeString` ธรรมดาทั่วไปคือ **มันรู้ context ที่ค่าแต่ละตัวไปปรากฏอยู่ในเอกสาร HTML** และเลือกวิธี escape ให้เหมาะกับ context นั้นโดยอัตโนมัติ — เรียกว่า **contextual autoescaping**

ตัวอย่างที่ทำให้เห็นภาพชัดคือค่าเดียวกัน แต่ escape ต่างกันตาม context ที่มันอยู่:

```go
const tmplSrc = `
<p>{{.Value}}</p>
<a href="/search?q={{.Value}}">ลิงก์</a>
<script>var x = "{{.Value}}";</script>
<div data-info="{{.Value}}"></div>
`
```

ถ้า `.Value` เป็น string `she said "hello" & <bye>` ผลลัพธ์แต่ละบรรทัดจะถูก escape **ต่างกัน** เพราะ engine วิเคราะห์ตอน parse ว่าแต่ละ `{{.Value}}` อยู่ใน context ไหน:

| Context ที่ค่าปรากฏ | วิธี escape ที่ใช้ |
|---|---|
| เนื้อหา HTML ทั่วไป (`<p>{{.Value}}</p>`) | HTML entity escaping (`<` → `&lt;`, `&` → `&amp;`) |
| ใน URL query (`href="...?q={{.Value}}"`) | URL escaping (`&` → `%26`, space → `%20`) |
| ใน JavaScript string (`var x = "{{.Value}}"`) | JavaScript string escaping (escape `"` เป็น `\"` ป้องกันการหลุดออกจาก string literal) |
| ใน HTML attribute (`data-info="{{.Value}}"`) | HTML attribute escaping |

นี่คือสาเหตุที่ `html/template` **ต้อง parse โครงสร้าง HTML ทั้งหมดก่อน** (ไม่ใช่แค่แทนค่าตัวแปรแบบ string ธรรมดาเหมือน `text/template`) เพื่อให้รู้ว่าตำแหน่งที่ `{{.Value}}` ปรากฏอยู่นั้นอยู่ใน tag, attribute, URL, หรือ JavaScript block — ความฉลาดระดับนี้คือสิ่งที่ **ไม่มีทางทำได้ด้วยการ `html.EscapeString` มือเปล่าแบบเดิมๆ ทุกจุด** เพราะกฎการ escape ของแต่ละ context ไม่เหมือนกัน

> **จุดสำคัญที่ต้องจำ**: contextual escaping ทำงานได้ก็ต่อเมื่อ **template ถูก parse โดย `html/template` ทั้งไฟล์** ถ้าไปสร้าง HTML string บางส่วนด้วยวิธีอื่นแล้วค่อยต่อเข้ากับผลลัพธ์ของ template (เช่น ใช้ `template.HTML(untrustedString)` ครอบ string ที่ไม่ผ่านการตรวจสอบ) จะเป็นการ **ปิดกลไก escaping ของจุดนั้นไปเลย** — `template.HTML` คือ type พิเศษที่บอก `html/template` ว่า "เชื่อถือ string นี้ อย่า escape" ใช้ได้เฉพาะกับ string ที่มาจากแหล่งที่เชื่อถือได้จริงๆ เท่านั้น (เช่น HTML ที่ sanitize มาแล้วด้วย library เฉพาะทาง) **ห้ามใช้กับ input จากผู้ใช้เด็ดขาด**

---

## 5. Syntax พื้นฐาน: `{{.Field}}`, `{{if}}`, `{{range}}`

Syntax ของ template action ทั้งหมดอยู่ภายใน `{{ }}` ทั้ง `text/template` และ `html/template` ใช้ syntax ชุดเดียวกัน

### `{{.Field}}` — แสดงค่าจาก field ของ struct หรือ key ของ map

`.` (dot) หมายถึง **ข้อมูลปัจจุบันที่ template กำลังทำงานด้วย** (เรียกว่า "pipeline data") ที่ส่งเข้ามาตอนเรียก `Execute`:

```go
type Todo struct {
	Title string
	Done  bool
}
```

```html
<h1>{{.Title}}</h1>
<p>สถานะ: {{.Done}}</p>
```

ถ้าข้อมูลเป็น `map[string]interface{}` แทน struct ก็ใช้ syntax เดียวกัน (`{{.key}}`)

### `{{if}}...{{else}}...{{end}}` — เงื่อนไข

```html
{{if .Done}}
    <s>{{.Title}}</s> (เสร็จแล้ว)
{{else}}
    {{.Title}}
{{end}}
```

รองรับ `{{else if .SomeCondition}}` ต่อกันได้เหมือน `else if` ในภาษาปกติ ค่าที่ถือว่าเป็น "false" ใน `{{if}}` คือ zero value ของ type นั้น (`""`, `0`, `false`, `nil`, slice/map ว่าง) — กฎเดียวกับที่เราเรียนเรื่อง zero value ไปใน **Part 003**

### `{{range}}...{{end}}` — วนลูป

```go
type PageData struct {
	Todos []Todo
}
```

```html
<ul>
{{range .Todos}}
    <li>{{.Title}}</li>
{{end}}
</ul>
```

ข้างใน `{{range}}` ตัว `.` จะเปลี่ยนความหมายเป็น **แต่ละ element ของ slice** (ไม่ใช่ `PageData` อีกต่อไป) นี่เป็นจุดที่มือใหม่สับสนบ่อยที่สุด — ถ้าต้องการเข้าถึงข้อมูลระดับนอก (เช่น `PageData.Title` ขณะที่อยู่ใน `range .Todos`) ต้องใช้ `$` เพื่ออ้างอิงถึง root data:

```html
{{$pageTitle := .Title}}
{{range .Todos}}
    <li>[{{$pageTitle}}] {{.Title}}</li>
{{end}}
```

`{{$pageTitle := .Title}}` คือการประกาศตัวแปร template ชื่อ `$pageTitle` (คล้าย `:=` ในภาษา Go) เก็บค่าไว้ใช้ต่อแม้ context ของ `.` จะเปลี่ยนไปตอนอยู่ใน `range` แล้วก็ตาม

`{{range}}` ยังรองรับกรณี slice ว่างเปล่าด้วย `{{else}}`:

```html
{{range .Todos}}
    <li>{{.Title}}</li>
{{else}}
    <li>ยังไม่มีรายการ</li>
{{end}}
```

---

## 6. `{{define}}` และ `{{template}}`: Layout และ Partial

เว็บแอปจริงแทบทุกหน้าต้องมี layout ร่วมกัน (เช่น `<head>`, navbar, footer เหมือนกันทุกหน้า) การ copy-paste HTML เดิมซ้ำในทุกไฟล์เป็นวิธีที่แย่มาก — `html/template` แก้ปัญหานี้ด้วย **named template** ผ่าน `{{define "name"}}...{{end}}` แล้วเรียกใช้ด้วย `{{template "name" .}}`

### `{{define "name"}}...{{end}}` — ประกาศ template ย่อยที่มีชื่อ

```html
{{define "navbar"}}
<nav>
    <a href="/">หน้าแรก</a>
</nav>
{{end}}
```

ไฟล์หนึ่งไฟล์มี `{{define}}` ได้หลายอัน และเมื่อ parse หลายไฟล์เข้าด้วยกัน (หัวข้อถัดไป) ทุก `{{define}}` จาก**ทุกไฟล์**จะถูกรวมเข้าเป็น **template set เดียวกัน** เรียกใช้ข้ามไฟล์กันได้อย่างอิสระ

### `{{template "name" .}}` — เรียกใช้ template ที่ประกาศไว้

```html
{{define "layout"}}
<!DOCTYPE html>
<html>
<body>
    {{template "navbar" .}}
    <main>{{template "content" .}}</main>
</body>
</html>
{{end}}
```

จุดสำคัญคือ **`.` ที่ตามหลังชื่อ template** — นี่คือการส่งข้อมูลปัจจุบันต่อไปให้ template ย่อยนั้นใช้งาน ถ้าลืมใส่ `.` (เขียนแค่ `{{template "navbar"}}`) template ย่อยนั้นจะได้ข้อมูลเป็น `nil` ไปแทน ทำให้ `{{.SomeField}}` ข้างในกลายเป็น error ทันที — นี่เป็นจุดที่มือใหม่ลืมบ่อยที่สุดอันดับหนึ่งเวลาทำ layout pattern

### รูปแบบไฟล์ที่นิยมในโปรเจกต์จริง

```
templates/
├── layout.html   ← {{define "layout"}} ... {{template "navbar" .}} ... {{template "content" .}} ... {{end}}
├── navbar.html   ← {{define "navbar"}} ... {{end}}
└── home.html     ← {{define "content"}} ... {{end}} (เนื้อหาเฉพาะหน้า home)
```

แต่ละหน้า (`home.html`, `about.html`, ...) แค่ implement `{{define "content"}}` ของตัวเอง แล้วทุกหน้าจะได้ navbar และโครง HTML เดียวกันจาก `layout.html` โดยอัตโนมัติ — เราจะเห็นรูปแบบนี้ทำงานจริงในตัวอย่างเต็มของหัวข้อ 10

---

## 7. โหลด Template จากไฟล์: `ParseFiles` vs `ParseGlob`

การเขียน template เป็น string ในโค้ด Go (แบบหัวข้อ 3) ใช้ได้ดีสำหรับตัวอย่างสั้นๆ แต่โปรเจกต์จริงจะเก็บ template เป็นไฟล์ `.html` แยกต่างหาก แล้วโหลดด้วยสอง method หลัก:

### `template.ParseFiles(filenames ...string)` — ระบุชื่อไฟล์ตรงๆ

```go
tmpl, err := template.ParseFiles(
	"templates/layout.html",
	"templates/navbar.html",
	"templates/home.html",
)
```

เหมาะกับกรณีที่รู้แน่นอนว่าต้องใช้ไฟล์อะไรบ้าง และต้องการควบคุมลำดับ/รายการไฟล์อย่างชัดเจน

### `template.ParseGlob(pattern string)` — ใช้ wildcard pattern

```go
tmpl, err := template.ParseGlob("templates/*.html")
```

เหมาะกับกรณีที่มีไฟล์ template จำนวนมากในโฟลเดอร์เดียว และไม่อยากแก้โค้ด Go ทุกครั้งที่เพิ่มไฟล์ใหม่ — แค่วางไฟล์ `.html` ใหม่ลงในโฟลเดอร์ `templates/` แล้ว `ParseGlob` จะดึงเข้ามารวมให้อัตโนมัติในรอบถัดไปที่ parse

### รูปแบบที่ใช้บ่อยที่สุด: `New(name).Funcs(funcMap).ParseGlob(pattern)`

ในทางปฏิบัติ เรามักจะ chain method เพื่อกำหนดชื่อ template หลักและ custom function ไปพร้อมกันตั้งแต่ต้น (จะอธิบาย `FuncMap` ในหัวข้อ 9):

```go
tmpl := template.Must(
	template.New("layout").Funcs(funcMap).ParseGlob("templates/*.html"),
)
```

`template.Must(t, err)` เป็น helper ที่ **panic ทันทีถ้า `err != nil`** — ใช้ได้เฉพาะตอน **startup ของโปรแกรม** (ตอนโหลด template ตอน `main()` เริ่มทำงาน) เพราะ error ในการ parse template (เช่น syntax ผิด, เรียก `{{define}}` ซ้ำชื่อกัน) ควรทำให้โปรแกรม**หยุดทำงานทันทีตั้งแต่ startup** ไม่ควรปล่อยให้รันต่อไปแล้วไป error ตอนมี request จริงเข้ามา — หลักการเดียวกับที่เราเรียนเรื่อง `panic`/`recover` ใน **Part 017** ว่าควรใช้ `panic` เฉพาะกับข้อผิดพลาดที่โปรแกรมทำงานต่อไปไม่ได้จริงๆ

**ข้อควรระวัง**: `ParseGlob("templates/*.html")` ดึงเฉพาะไฟล์ที่อยู่ **ตรงชั้นเดียว** ในโฟลเดอร์ที่ระบุเท่านั้น ไม่ลงลึกไปโฟลเดอร์ย่อย ถ้ามี template อยู่หลายโฟลเดอร์ย่อย ต้องเรียก `ParseGlob` หลายครั้ง (ทีละ pattern) หรือใช้ `filepath.WalkDir` (จาก **Part 024**) เดินไล่หาไฟล์เองแล้วส่งรายการเข้า `ParseFiles`

---

## 8. ส่งข้อมูลเข้า Template: struct, map, และ `Execute`/`ExecuteTemplate`

Template ทำงานได้กับข้อมูลได้สองแบบหลัก: **struct** และ **map**

```go
type PageData struct {
	Title string
	Todos []Todo
}

data := PageData{
	Title: "รายการ Todo",
	Todos: []Todo{{Title: "เรียน Go", Done: true}},
}
```

หรือใช้ `map[string]interface{}` ก็ได้เหมือนกัน (สะดวกสำหรับ prototype เร็วๆ แต่เสีย type safety ไปเมื่อเทียบกับ struct):

```go
data := map[string]interface{}{
	"Title": "รายการ Todo",
	"Todos": []Todo{{Title: "เรียน Go", Done: true}},
}
```

ทั้งสองแบบใช้ syntax `{{.Title}}` เหมือนกันทุกประการในฝั่ง template — ต่างกันแค่ฝั่ง Go code เท่านั้น สำหรับโปรเจกต์จริง **แนะนำให้ใช้ struct เสมอ** เพราะได้ compile-time check ว่า field ที่อ้างถึงมีอยู่จริง (ถ้าพิมพ์ field ผิดใน map จะไม่มี error ตอน compile แต่ template จะ render เป็นค่าว่างเงียบๆ ตอนรัน ซึ่ง debug ยากกว่ามาก)

### `Execute` vs `ExecuteTemplate`

```go
// Execute: ใช้ template "หลัก" ที่ตั้งชื่อไว้ตอน New(name) โดยตรง
err := tmpl.Execute(w, data)

// ExecuteTemplate: ระบุชื่อ template ที่ต้องการ render อย่างชัดเจน
err := tmpl.ExecuteTemplate(w, "layout", data)
```

เมื่อไหร่ใช้อันไหน:

- **`Execute`** เหมาะกับกรณีง่ายๆ ที่มี template เดียว ไม่มี `{{define}}` ซ้อนหลายชั้น
- **`ExecuteTemplate`** จำเป็นเมื่อ parse หลายไฟล์รวมกัน (เช่นตอนใช้ `ParseGlob`) เพราะ template set ที่ได้มีหลาย `{{define}}` ปนกันอยู่ ต้องระบุให้ชัดว่าจะ render จากจุดไหน (ปกติคือ `"layout"` ตามรูปแบบในหัวข้อ 6)

`w` (parameter แรก) เป็น `io.Writer` ใดๆ ก็ได้ — ในเว็บเซิร์ฟเวอร์คือ `http.ResponseWriter` (implement `io.Writer` อยู่แล้ว ตามที่เรียนเรื่อง interface composition ใน **Part 048**) แต่จะเป็น `os.Stdout`, `bytes.Buffer`, หรือไฟล์ก็ได้เหมือนกัน (มีประโยชน์มากตอนเขียน test — render ลง `bytes.Buffer` แล้วเทียบ string ที่ได้)

---

## 9. Template Functions: `FuncMap`

Syntax พื้นฐานของ template (หัวข้อ 5) ไม่มี function สำหรับจัดรูปแบบวันที่ ตัวเลข หรือ logic ที่ซับซ้อนกว่านั้นให้ในตัว — ถ้าต้องการ ต้องประกาศเองผ่าน `template.FuncMap`:

```go
funcMap := template.FuncMap{
	"formatDate": func(t time.Time) string {
		return t.Format("2006-01-02 15:04:05")
	},
	"upper": strings.ToUpper,
}

tmpl := template.Must(
	template.New("layout").Funcs(funcMap).ParseGlob("templates/*.html"),
)
```

`FuncMap` คือแค่ `type FuncMap map[string]interface{}` — key คือชื่อ function ที่จะเรียกใช้ในไฟล์ `.html` ส่วน value คือ Go function จริง (ต้อง return ค่าเดียว หรือสองค่าโดยค่าที่สองเป็น `error`)

เรียกใช้ใน template ด้วยชื่อที่ตั้งไว้ (ไม่ใช่ `()` เหมือน Go ปกติ แต่ใช้ space คั่น argument):

```html
<p>อัปเดตล่าสุด: {{formatDate .Now}}</p>
<p>ชื่อผู้ใช้: {{upper .Username}}</p>
```

**ข้อควรระวังสำคัญ**: `Funcs(funcMap)` **ต้องเรียกก่อน** `Parse`/`ParseFiles`/`ParseGlob` เสมอ เพราะตอน parse engine จะตรวจสอบว่าทุก function name ที่ใช้ในไฟล์มีอยู่จริงใน `FuncMap` หรือไม่ทันที ถ้าเรียก `Funcs()` หลัง parse ไปแล้ว จะได้ error ทำนอง `function "formatDate" not defined` แม้ว่าจริงๆ จะประกาศ `funcMap` ไว้ถูกต้องแล้วก็ตาม

Function ยอดนิยมที่โปรเจกต์จริงมักประกาศเพิ่มเอง:

| Function ตัวอย่าง | ประโยชน์ |
|---|---|
| `formatDate` / `formatCurrency` | จัดรูปแบบวันที่/ตัวเลขให้อ่านง่าย (Go ไม่มี built-in currency formatter) |
| `truncate(s string, n int) string` | ตัด text ยาวๆ ให้แสดงแค่บางส่วน (เช่น preview บทความ) |
| `safeHTML(s string) template.HTML` | ครอบ string ที่ sanitize มาแล้วให้ไม่ถูก escape ซ้ำ (ใช้ระวังมากตามที่เตือนไว้ในหัวข้อ 4) |
| `add`, `sub`, `mul` | เลขคณิตพื้นฐาน เพราะ template ไม่มี operator `+`/`-` ในตัว |

---

## 10. ตัวอย่างเต็ม: หน้าเว็บ Todo List พร้อม Layout + Partial จาก HTTP Server จริง

มาประกอบทุกอย่างที่เรียนมาเข้าด้วยกัน เขียน HTTP server ด้วย `net/http` (แบบเดียวกับที่เรียนใน **Part 047** และ **Part 056**) ที่ render หน้าเว็บ HTML จริงด้วย layout + partial

โครงสร้างโปรเจกต์:

```
todo-web/
├── go.mod
├── main.go
└── templates/
    ├── layout.html
    ├── navbar.html
    └── home.html
```

### `templates/layout.html` — โครงหลักของทุกหน้า

```html
{{define "layout"}}
<!DOCTYPE html>
<html lang="th">
<head>
    <meta charset="UTF-8">
    <title>{{.Title}} | Todo App</title>
</head>
<body>
    {{template "navbar" .}}
    <main>
        {{template "content" .}}
    </main>
    <footer>
        <p>อัปเดตล่าสุด: {{formatDate .Now}}</p>
    </footer>
</body>
</html>
{{end}}
```

### `templates/navbar.html` — Partial ที่ใช้ร่วมกันทุกหน้า

```html
{{define "navbar"}}
<nav>
    <a href="/">หน้าแรก</a> |
    <span>ผู้ใช้: {{.User}}</span> |
    <span>รายการทั้งหมด: {{len .Todos}}</span>
</nav>
{{end}}
```

สังเกตการใช้ `{{len .Todos}}` — `len` เป็นหนึ่งใน built-in function ไม่กี่ตัวที่ `html/template` มีให้ใช้ฟรีโดยไม่ต้องประกาศใน `FuncMap` เอง (ทำงานเหมือน `len()` ของภาษา Go ปกติ ใช้ได้กับ slice, map, string)

### `templates/home.html` — เนื้อหาเฉพาะหน้า home

```html
{{define "content"}}
<h1>{{.Title}}</h1>
{{if .Todos}}
<ul>
    {{range .Todos}}
    <li>
        {{if .Done}}<s>{{.Title}}</s> (เสร็จแล้ว){{else}}{{.Title}}{{end}}
    </li>
    {{end}}
</ul>
{{else}}
<p>ยังไม่มีรายการ todo</p>
{{end}}
<p>ข้อความจากผู้ใช้ (สังเกตการ escape อัตโนมัติ): {{.UserComment}}</p>
{{end}}
```

บรรทัดสุดท้าย (`{{.UserComment}}`) ตั้งใจใส่ไว้เพื่อพิสูจน์ contextual escaping จากหัวข้อ 3-4 อีกครั้งในสถานการณ์ HTTP server จริง

### `main.go` — HTTP server ที่โหลด template และ serve หน้าเว็บ

```go
package main

import (
	"html/template"
	"log"
	"net/http"
	"time"
)

type Todo struct {
	Title string
	Done  bool
}

type PageData struct {
	Title       string
	User        string
	Todos       []Todo
	UserComment string
	Now         time.Time
}

var tmpl *template.Template

func formatDate(t time.Time) string {
	return t.Format("2006-01-02 15:04:05")
}

func main() {
	funcMap := template.FuncMap{
		"formatDate": formatDate,
	}

	// ParseGlob โหลดทุกไฟล์ .html ในโฟลเดอร์เดียวกันเข้ามารวมเป็น template set เดียว
	// ทุกไฟล์แชร์ FuncMap และ {{define}} เดียวกันได้หมด
	tmpl = template.Must(
		template.New("layout").Funcs(funcMap).ParseGlob("templates/*.html"),
	)

	http.HandleFunc("/", homeHandler)

	log.Println("server running on :8080")
	log.Fatal(http.ListenAndServe(":8080", nil))
}

func homeHandler(w http.ResponseWriter, r *http.Request) {
	data := PageData{
		Title: "รายการ Todo ของฉัน",
		User:  "phutjirakul",
		Todos: []Todo{
			{Title: "เรียน html/template", Done: true},
			{Title: "เขียน layout + partial", Done: false},
		},
		UserComment: `<script>alert('xss')</script>`,
		Now:         time.Now(),
	}

	w.Header().Set("Content-Type", "text/html; charset=utf-8")
	if err := tmpl.ExecuteTemplate(w, "layout", data); err != nil {
		http.Error(w, err.Error(), http.StatusInternalServerError)
	}
}
```

จุดที่ควรสังเกตในโค้ดนี้ (เชื่อมโยงกับสิ่งที่เรียนใน **Part 047/056**):

- `w.Header().Set("Content-Type", "text/html; charset=utf-8")` **ต้องตั้งเอง** — `net/http` ไม่รู้ล่วงหน้าว่าเราจะเขียน HTML หรือ JSON ออกไป ต่างจากตอนใช้ `c.JSON(...)` ของ Gin/Echo/Fiber ใน Part 061-064 ที่ framework ตั้ง `Content-Type` เป็น `application/json` ให้อัตโนมัติ
- `tmpl.ExecuteTemplate(w, "layout", data)` เขียนผลลัพธ์ตรงลง `http.ResponseWriter` โดยไม่ต้องสร้าง string กลางเลย — เหมาะกับ performance เพราะไม่มีการ allocate buffer เพิ่มโดยไม่จำเป็น
- ถ้า `ExecuteTemplate` คืน error (เช่น field ที่ template อ้างถึงไม่มีจริงตอน runtime) เราต้อง `http.Error(...)` เอง — **ข้อควรระวัง**: ถ้า template render ไปแล้วบางส่วนก่อนเจอ error (เช่น layout ขึ้นไปครึ่งหนึ่งแล้ว) `http.Error` จะเขียนซ้อนทับเข้าไปในสตรีมเดิม ทำให้ HTML ที่ client ได้รับเสียหาย (partial render ปนกับ error message) — ในโปรเจกต์จริงที่ต้องการความทนทานสูง มักจะ render ลง `bytes.Buffer` ก่อน เช็ค error ให้ครบ แล้วค่อย `w.Write(buf.Bytes())` ทีเดียวตอนจบ

### รันและทดสอบจริง

```bash
go run main.go
```

```
2026/09/26 03:21:12 server running on :8080
```

ทดสอบด้วย `curl` (ผลลัพธ์จริงจากการรัน — ตัดพื้นที่ว่างส่วนเกินออกเพื่อความกระชับ):

```bash
curl http://localhost:8080/
```

```html
<!DOCTYPE html>
<html lang="th">
<head>
    <meta charset="UTF-8">
    <title>รายการ Todo ของฉัน | Todo App</title>
</head>
<body>
    
<nav>
    <a href="/">หน้าแรก</a> |
    <span>ผู้ใช้: phutjirakul</span> |
    <span>รายการทั้งหมด: 2</span>
</nav>

    <main>
        
<h1>รายการ Todo ของฉัน</h1>

<ul>
    
    <li>
        <s>เรียน html/template</s> (เสร็จแล้ว)
    </li>
    
    <li>
        เขียน layout &#43; partial
    </li>
    
</ul>

<p>ข้อความจากผู้ใช้ (สังเกตการ escape อัตโนมัติ): &lt;script&gt;alert(&#39;xss&#39;)&lt;/script&gt;</p>

    </main>
    <footer>
        <p>อัปเดตล่าสุด: 2026-09-26 03:21:13</p>
    </footer>
</body>
</html>
```

สังเกตจากผลลัพธ์จริงตรงนี้ครบทุกจุดที่เรียนมาในบทนี้:

1. **navbar (partial)** ถูกดึงมาแสดงถูกต้องผ่าน `{{template "navbar" .}}` พร้อมข้อมูล `.User` และ `len .Todos` ที่ถูกต้อง (2 รายการ)
2. **`{{range .Todos}}`** วนแสดงทั้งสองรายการ และ **`{{if .Done}}`** ตัดสินใจถูกต้องว่ารายการไหนต้องขึ้น `<s>` (ขีดฆ่า) รายการไหนไม่ต้อง
3. **`{{formatDate .Now}}`** (custom function จาก `FuncMap`) แปลง `time.Time` เป็น string รูปแบบที่กำหนดสำเร็จ
4. **`<script>alert('xss')</script>`** ที่จงใจใส่เข้าไปใน `UserComment` ถูก escape เป็น `&lt;script&gt;alert(&#39;xss&#39;)&lt;/script&gt;` โดยอัตโนมัติ — **พิสูจน์ซ้ำอีกครั้งในสถานการณ์ที่ใกล้เคียงกับ production จริง** ว่า `html/template` ป้องกัน XSS ให้เราตลอดเวลาโดยไม่ต้องเขียนโค้ด escape เองเลยสักบรรทัด

---

## 11. ข้อควรระวังและแนวทางปฏิบัติที่ดี

สรุปข้อควรระวังที่สำคัญที่สุดจากทั้งบทไว้ในที่เดียว:

1. **ใช้ `html/template` เสมอสำหรับ web output** — ไม่มีข้อยกเว้น แม้จะดูเหมือนไม่มี user input เข้ามาเกี่ยวข้องเลยก็ตาม เพราะข้อมูลที่ "ตอนนี้ปลอดภัย" อาจกลายเป็นข้อมูลจาก user ในอนาคตเมื่อโค้ดถูกแก้ไขต่อ
2. **ระวัง `template.HTML` และ type พิเศษอื่นๆ** (`template.JS`, `template.CSS`, `template.URL`) — ทุกตัวคือการ "ปิด" การ escape ของ engine สำหรับค่านั้น ใช้ได้เฉพาะกับข้อมูลที่ผ่านการ sanitize มาอย่างน่าเชื่อถือแล้วเท่านั้น
3. **`Funcs()` ต้องเรียกก่อน `Parse`/`ParseFiles`/`ParseGlob` เสมอ** ไม่งั้นจะเจอ error `function ... not defined` แม้ประกาศ `FuncMap` ถูกต้องแล้ว
4. **`{{template "name" .}}` อย่าลืมจุด (`.`) ท้ายชื่อ** ไม่งั้น template ย่อยจะได้ข้อมูลเป็น `nil`
5. **ใช้ `template.Must()` เฉพาะตอน parse ตอน startup เท่านั้น** ไม่ใช่ตอนจัดการ request จริง (error ตอน parse ควรทำให้โปรแกรมไม่ start เลย ไม่ใช่ให้ทุก request พังทีละคน)
6. **`ParseGlob` ไม่ลงโฟลเดอร์ย่อย** — วางไฟล์ template ทั้งหมดไว้ชั้นเดียวกัน หรือเรียก `ParseGlob`/`ParseFiles` หลายครั้งถ้าจำเป็นต้องแยกโฟลเดอร์
7. **Content-Type ต้องตั้งเอง** เมื่อใช้ `net/http` ล้วนๆ (ต่างจาก Gin/Echo/Fiber ที่ `c.JSON()` ตั้งให้อัตโนมัติ) ลืมตั้งจะทำให้บางเบราว์เซอร์เดา content type ผิดได้

---

## สรุปสิ่งที่ได้เรียนในบทนี้

- `text/template` และ `html/template` มี syntax เหมือนกันทุกตัวอักษร แต่ `html/template` เพิ่ม **contextual autoescaping** เข้ามาเพื่อป้องกัน XSS โดยอัตโนมัติ — พิสูจน์ด้วยโค้ดจริงว่า `<script>` ถูก escape เป็น `&lt;script&gt;` เมื่อใช้ `html/template` แต่หลุดออกมาตรงๆ เมื่อใช้ `text/template`
- Contextual escaping รู้ context ที่ค่าปรากฏอยู่ (HTML body, attribute, URL, JavaScript) และเลือกวิธี escape ที่เหมาะสมให้อัตโนมัติ — ทำงานได้เพราะ `html/template` parse โครงสร้าง HTML ทั้งไฟล์ ไม่ใช่แค่แทนค่า string เฉยๆ
- Syntax พื้นฐาน: `{{.Field}}` แสดงค่า, `{{if}}...{{else}}...{{end}}` เงื่อนไข, `{{range}}...{{end}}` วนลูป (พร้อม `{{else}}` กรณี slice ว่าง)
- `{{define "name"}}...{{end}}` ประกาศ template ย่อยที่มีชื่อ เรียกใช้ด้วย `{{template "name" .}}` — ใช้สร้างระบบ layout + partial ที่ใช้ HTML ร่วมกันได้หลายหน้าโดยไม่ต้อง copy-paste
- โหลด template จากไฟล์ด้วย `template.ParseFiles(...)` (ระบุชื่อไฟล์ตรงๆ) หรือ `template.ParseGlob(pattern)` (ใช้ wildcard) — รูปแบบยอดนิยมคือ `template.New(name).Funcs(funcMap).ParseGlob(pattern)`
- ส่งข้อมูลเข้า template ได้ทั้ง struct (แนะนำ เพราะ type-safe กว่า) และ map แล้ว render ด้วย `Execute`/`ExecuteTemplate` ลง `io.Writer` ใดๆ (รวมถึง `http.ResponseWriter`)
- `template.FuncMap` ใช้ประกาศ custom function สำหรับใช้ใน template (เช่น `formatDate`) ต้องเรียก `.Funcs()` ก่อน parse เสมอ
- ตัวอย่างเต็มแสดงให้เห็น HTTP server จริง (ต่อยอดจาก Part 047/056) ที่ render หน้าเว็บด้วย layout + navbar partial + FuncMap ครบวงจร พร้อมพิสูจน์ contextual escaping อีกครั้งในสถานการณ์ใกล้ production

## แบบฝึกหัดท้ายบท

1. รันโค้ดสาธิต XSS ในหัวข้อ 3 ด้วยตัวเอง แล้วลองเปลี่ยน `Body` เป็น string อื่นที่มีอักขระพิเศษ เช่น `<b>bold</b> & "quoted"` สังเกตว่า `html/template` escape อักขระตัวไหนบ้าง
2. เพิ่มไฟล์ `templates/about.html` ที่มี `{{define "content"}}` ของตัวเอง แล้วเพิ่ม route `/about` ใน `main.go` ที่ใช้ `layout.html` และ `navbar.html` เดิม แต่แสดงเนื้อหาต่างออกไป (พิสูจน์ว่า layout ใช้ซ้ำข้ามหน้าได้จริง)
3. เพิ่ม custom function ใหม่ใน `FuncMap` ชื่อ `percentage` ที่รับ `(done, total int) string` แล้วคืนค่าเป็น string เปอร์เซ็นต์ (เช่น `"50%"`) นำไปแสดงในหน้า home ว่า todo เสร็จไปกี่เปอร์เซ็นต์
4. ลองลบจุด (`.`) ท้าย `{{template "navbar" .}}` ออก (ให้เหลือ `{{template "navbar"}}`) แล้วรันดูว่าเกิด error อะไร อธิบายว่าทำไม
5. เขียนฟังก์ชัน `renderToString(tmpl *template.Template, name string, data any) (string, error)` ที่ใช้ `bytes.Buffer` render template ออกมาเป็น string แทนที่จะเขียนตรงลง `http.ResponseWriter` (ใบ้: ใช้ `bytes.Buffer` ที่เรียนใน **Part 050**) แล้วเขียน unit test เช็คว่า string ที่ได้มีคำว่า `&lt;script&gt;` อยู่จริงเมื่อส่ง input ที่มี `<script>` เข้าไป
6. ลองสร้างสถานการณ์ที่ตั้งใจใช้ `template.HTML(userInput)` ครอบ input ที่มาจากผู้ใช้ตรงๆ (ไม่ sanitize) แล้วสังเกตว่า XSS กลับมาเกิดขึ้นได้อีกครั้งหรือไม่ พร้อมอธิบายว่าทำไมจึงไม่ควรทำแบบนี้ในโปรเจกต์จริง

---

**ต่อไป**: [Part 066 — Static Files และ File Upload](./066-static-files-and-upload.md)
