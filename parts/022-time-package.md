# Part 022: เวลาและแพ็กเกจ `time`

> ภาคที่ 2: ระดับกลาง (Intermediate) — ตอนที่ 7 จาก 20 (Part 16–35)

## สารบัญของบทนี้

1. ทำไมการจัดการเวลาถึงเป็นเรื่องซับซ้อนกว่าที่คิด
2. `time.Time` และ `time.Duration` คืออะไร
3. `time.Now()` และการดึงส่วนประกอบของเวลา
4. Format และ Parse เวลาด้วย Reference Date อันโด่งดัง
5. Time Zone และ `time.LoadLocation`
6. เลขคณิตกับเวลา: `Add`, `Sub`, `Since`, `Until`
7. การเปรียบเทียบเวลา: `Before`, `After`, `Equal`
8. `time.Sleep` และการหน่วงเวลา
9. Timer และ Ticker: ตัวอย่างเบื้องต้น (`time.After`, `time.Tick`, `NewTimer`, `NewTicker`)
10. กับดักสำคัญ: Monotonic Clock vs Wall Clock
11. สรุปสิ่งที่ได้เรียนในบทนี้
12. แบบฝึกหัดท้ายบท

---

## 1. ทำไมการจัดการเวลาถึงเป็นเรื่องซับซ้อนกว่าที่คิด

เวลาเป็นหนึ่งในเรื่องที่ดูเหมือนง่ายแต่จริงๆ แล้วซับซ้อนมากในการเขียนโปรแกรม เพราะต้องรับมือกับ:

- **Time zone** ที่แตกต่างกันทั่วโลก และบางประเทศมี Daylight Saving Time (DST) ที่เปลี่ยนเวลาปีละ 2 ครั้ง
- **รูปแบบการแสดงผล** ที่แตกต่างกันในแต่ละประเทศ (`DD/MM/YYYY` vs `MM/DD/YYYY`)
- **ความแม่นยำ** ที่ต้องการ (วินาที, มิลลิวินาที, นาโนวินาที)
- **การวัดระยะเวลา** ที่ต้องไม่ถูกกระทบจากการที่ผู้ใช้เปลี่ยนนาฬิการะบบ

ภาษาโปรแกรมมิ่งหลายภาษาแยก "เวลา" (point in time) กับ "ระยะเวลา" (duration) ปนกันจนสับสน แต่ Go ออกแบบ package `time` ให้แยกสอง concept นี้อย่างชัดเจนตั้งแต่ระดับ type — และนี่คือสิ่งแรกที่เราต้องเข้าใจให้แม่นก่อน

---

## 2. `time.Time` และ `time.Duration` คืออะไร

### `time.Time` — จุดใดจุดหนึ่งบนเส้นเวลา

`time.Time` คือ struct ที่แทน **จุดเวลาที่แน่นอนจุดหนึ่ง** (a specific instant) เช่น "26 กันยายน 2026 เวลา 14:30:00" มันเก็บทั้งวันที่ เวลา และ time zone ไว้ในตัวเดียวกัน

```go
var t time.Time // zero value คือ 0001-01-01 00:00:00 UTC
```

สังเกตว่า zero value ของ `time.Time` ไม่ใช่ค่าว่างเปล่าแบบ `nil` แต่เป็นวันที่ `0001-01-01 00:00:00 UTC` จริงๆ ซึ่งมีประโยชน์เวลาเช็คว่าตัวแปร `time.Time` "ยังไม่ถูกตั้งค่า" หรือไม่ ด้วย method `IsZero()`

### `time.Duration` — ระยะเวลาที่ผ่านไป

`time.Duration` คือ **ระยะเวลาระหว่างสองจุดเวลา** (elapsed time) โดยภายในเก็บเป็นจำนวน **nanosecond** ในรูปแบบ `int64`:

```go
type Duration int64
```

Go มี constant สำเร็จรูปให้ใช้สร้าง Duration ได้อย่างอ่านง่าย:

```go
d1 := 90 * time.Minute       // 1h30m0s
d2 := 500 * time.Millisecond // 500ms
d3 := 2 * time.Hour          // 2h0m0s
```

ค่าคงที่ที่มีให้ ได้แก่ `time.Nanosecond`, `time.Microsecond`, `time.Millisecond`, `time.Second`, `time.Minute`, `time.Hour` — สังเกตว่าเวลาเขียน Duration แบบนี้จะได้ type safety เต็มรูปแบบ ผิดกับหลายภาษาที่ใช้ตัวเลข `int` เฉยๆ แทนมิลลิวินาทีจนสับสนได้ง่ายว่าหน่วยไหนเป็นหน่วยไหน

```go
package main

import (
	"fmt"
	"time"
)

func main() {
	now := time.Now()
	fmt.Println("ตอนนี้:", now)
	fmt.Println("ปี:", now.Year())
	fmt.Println("เดือน:", now.Month())
	fmt.Println("วัน:", now.Day())
	fmt.Println("ชั่วโมง:นาที:วินาที:", now.Hour(), now.Minute(), now.Second())
	fmt.Println("วันในสัปดาห์:", now.Weekday())

	d := 90 * time.Minute
	fmt.Println("Duration 90 นาที:", d)
	fmt.Println("Duration ชั่วโมง:", d.Hours())
	fmt.Println("Duration เป็นวินาที:", d.Seconds())
}
```

ผลลัพธ์ (เวลาจะต่างกันไปตามตอนที่รันจริง):

```
ตอนนี้: 2026-09-26 01:59:20.706968003 +0000 UTC m=+0.000080903
ปี: 2026
เดือน: September
วัน: 26
ชั่วโมง:นาที:วินาที: 1 59 20
วันในสัปดาห์: Saturday
Duration 90 นาที: 1h30m0s
Duration ชั่วโมง: 1.5
Duration เป็นวินาที: 5400
```

สังเกตส่วนท้าย `m=+0.000080903` ใน output ของ `now` — นี่คือค่า **monotonic clock reading** ที่ Go แนบมาให้อัตโนมัติ เราจะอธิบายความสำคัญของมันในหัวข้อ 10

---

## 3. `time.Now()` และการดึงส่วนประกอบของเวลา

`time.Now()` คือฟังก์ชันที่ใช้บ่อยที่สุดใน package นี้ — คืนค่า `time.Time` ของเวลาปัจจุบัน (ตาม local time zone ของเครื่องที่รัน) `time.Time` มี method สำหรับดึงส่วนประกอบต่างๆ ออกมาครบถ้วน:

| Method | ความหมาย | ตัวอย่างค่าที่ได้ |
|---|---|---|
| `.Year()` | ปี (ค.ศ.) | `2026` |
| `.Month()` | เดือน (type `time.Month`) | `September` |
| `.Day()` | วันที่ (1-31) | `26` |
| `.Hour()` | ชั่วโมง (0-23) | `14` |
| `.Minute()` | นาที (0-59) | `30` |
| `.Second()` | วินาที (0-59) | `0` |
| `.Weekday()` | วันในสัปดาห์ (type `time.Weekday`) | `Saturday` |
| `.Unix()` | Unix timestamp (วินาทีนับจาก 1 มกราคม 1970 UTC) | `1758852000` |
| `.UnixMilli()` / `.UnixNano()` | Unix timestamp หน่วยมิลลิวินาที/นาโนวินาที | — |

ถ้าต้องการสร้าง `time.Time` เอง (ไม่ใช้ `Now()`) ใช้ `time.Date`:

```go
t := time.Date(2024, time.March, 15, 14, 30, 0, 0, time.UTC)
// ปี, เดือน, วัน, ชั่วโมง, นาที, วินาที, nanosecond, location
```

`time.Month` เป็น custom type ที่มี `String()` implement ไว้แล้ว (เทคนิคเดียวกับที่เรียนใน **Part 021**) จึงพิมพ์เป็นชื่อเดือนภาษาอังกฤษ (`September`) แทนตัวเลขได้ทันที

---

## 4. Format และ Parse เวลาด้วย Reference Date อันโด่งดัง

นี่คือจุดที่ทำให้มือใหม่ Go งงที่สุดเมื่อเริ่มทำงานกับเวลา: **Go ไม่ใช้สัญลักษณ์แบบ `YYYY-MM-DD` หรือ `%Y-%m-%d` แบบภาษาอื่น แต่ใช้ "reference date" ที่ตายตัวค่าหนึ่งแทน**

### Reference Date คืออะไร และทำไมต้องเป็นค่านี้

รูปแบบเวลาที่ใช้ทั้งหมดใน Go อ้างอิงจากวันเวลาที่ตายตัวค่าเดียวคือ:

```
Mon Jan 2 15:04:05 MST 2006
```

หรือเขียนเป็นตัวเลขล้วนคือ **01/02 03:04:05PM '06 -0700** ซึ่งถ้าเรียงตามลำดับ **เดือน-วัน-ชั่วโมง-นาที-วินาที-ปี-timezone** จะได้ `1 2 3 4 5 6 7` เรียงกันพอดี — Rob Pike (หนึ่งในผู้สร้าง Go ที่กล่าวถึงใน **Part 001**) เลือกวันที่นี้เพราะมันจำง่ายเป็นตัวเลขไล่ 1-2-3-4-5-6-7 (เดือนที่ 1, วันที่ 2, ชั่วโมงที่ 3 (แบบ 12 ชม.), นาทีที่ 4, วินาทีที่ 5, ปี 06, timezone offset -07:00) แทนที่จะต้องจำสัญลักษณ์พิเศษแบบ `%Y %m %d` เหมือนภาษาอื่น

**หลักการใช้งาน**: เวลาจะเขียน format string ให้ `time.Format`/`time.Parse` เราต้อง**เขียนวันที่ `Mon Jan 2 15:04:05 MST 2006` ในรูปแบบที่เราต้องการให้ผลลัพธ์ออกมา** แล้ว Go จะแทนที่ตำแหน่งต่างๆ ด้วยค่าจริงให้อัตโนมัติ

| ส่วนใน reference date | ความหมาย |
|---|---|
| `2006` | ปี 4 หลัก |
| `06` | ปี 2 หลักท้าย |
| `01` | เดือน 2 หลัก (01-12) |
| `1` | เดือนไม่เติม 0 นำหน้า |
| `Jan` | ชื่อเดือนย่อ |
| `January` | ชื่อเดือนเต็ม |
| `02` | วันที่ 2 หลัก |
| `2` | วันที่ไม่เติม 0 นำหน้า |
| `15` | ชั่วโมงแบบ 24 ชม. |
| `03` | ชั่วโมงแบบ 12 ชม. |
| `04` | นาที |
| `05` | วินาที |
| `PM` | AM/PM |
| `Mon` | วันในสัปดาห์แบบย่อ |
| `Monday` | วันในสัปดาห์เต็ม |
| `MST` | ชื่อย่อ time zone |
| `-0700` | UTC offset |
| `Z07:00` | UTC offset (แสดง `Z` ถ้าเป็น UTC) |

### ตัวอย่างการ Format

```go
package main

import (
	"fmt"
	"time"
)

func main() {
	t := time.Date(2024, time.March, 15, 14, 30, 0, 0, time.UTC)
	fmt.Println(t.Format("2006-01-02 15:04:05"))
	fmt.Println(t.Format("02/01/2006"))
	fmt.Println(t.Format("Mon, 02 Jan 2006"))
	fmt.Println(t.Format(time.RFC3339))
	fmt.Println(t.Format("15:04"))
}
```

ผลลัพธ์:

```
2024-03-15 14:30:00
15/03/2024
Fri, 15 Mar 2024
2024-03-15T14:30:00Z
14:30
```

`time` package มี layout สำเร็จรูปให้ใช้หลายแบบ เช่น `time.RFC3339` (`"2006-01-02T15:04:05Z07:00"`), `time.RFC1123`, `time.Kitchen` (`"3:04PM"`) — layout ที่ใช้บ่อยที่สุดในโปรแกรมจริงคือ `time.RFC3339` เพราะเป็นมาตรฐานสากลที่ใช้แลกเปลี่ยนข้อมูลระหว่างระบบ (เราจะเจอมันอีกครั้งใน **Part 025** ตอนทำ JSON)

### ตัวอย่างการ Parse (แปลง string กลับเป็น `time.Time`)

```go
parsed, err := time.Parse("2006-01-02 15:04:05", "2024-12-25 09:00:00")
if err != nil {
	fmt.Println("parse error:", err)
}
fmt.Println("parsed:", parsed)

_, err2 := time.Parse("2006-01-02", "invalid-date")
fmt.Println("parse error (invalid):", err2)
```

ผลลัพธ์:

```
parsed: 2024-12-25 09:00:00 +0000 UTC
parse error (invalid): parsing time "invalid-date" as "2006-01-02": cannot parse "invalid-date" as "2006"
```

**ข้อควรจำสำคัญ**: layout string ที่ส่งให้ `time.Parse` ต้องตรงกับรูปแบบของ string ที่จะ parse แบบเป๊ะๆ (ยกเว้นจำนวนหลักในบางกรณี) ถ้ารูปแบบไม่ตรงจะได้ `error` กลับมาเสมอ — ตามหลักการ error handling ที่เรียนมาตั้งแต่ **Part 015** เราต้องเช็ค `err` ทุกครั้งที่ `Parse` ไม่ใช่ไว้ใจว่าจะสำเร็จเสมอ โดยเฉพาะเมื่อ string มาจากผู้ใช้หรือระบบภายนอก

---

## 5. Time Zone และ `time.LoadLocation`

`time.Time` ทุกค่าผูกกับ **`*time.Location`** เสมอ (field ภายในที่มองไม่เห็นตรงๆ แต่ส่งผลต่อการแสดงผลและการคำนวณ) มี location พิเศษสองตัวที่มีอยู่แล้วในตัว:

- `time.UTC` — Coordinated Universal Time
- `time.Local` — Local time zone ของเครื่องที่โปรแกรมรันอยู่

สำหรับ time zone อื่นๆ ต้องโหลดจาก **IANA Time Zone Database** ด้วย `time.LoadLocation`:

```go
package main

import (
	"fmt"
	"time"
)

func main() {
	utcNow := time.Now().UTC()
	fmt.Println("UTC:", utcNow.Format("2006-01-02 15:04:05 MST"))

	loc, err := time.LoadLocation("Asia/Bangkok")
	if err != nil {
		fmt.Println("load location error:", err)
	} else {
		bkk := utcNow.In(loc)
		fmt.Println("Bangkok:", bkk.Format("2006-01-02 15:04:05 MST"))
	}

	ny, err := time.LoadLocation("America/New_York")
	if err != nil {
		fmt.Println("load location error:", err)
	} else {
		nyTime := utcNow.In(ny)
		fmt.Println("New York:", nyTime.Format("2006-01-02 15:04:05 MST"))
	}
}
```

ผลลัพธ์ (ตัวอย่าง — เวลาจริงจะเปลี่ยนตามตอนที่รัน):

```
UTC: 2026-09-26 01:59:37 UTC
Bangkok: 2026-09-26 08:59:37 +07
New York: 2026-09-25 21:59:37 EDT
```

สังเกตว่า:

- `.In(loc)` **ไม่เปลี่ยนจุดเวลาจริง** เปลี่ยนแค่ "มุมมอง" การแสดงผลเท่านั้น — เวลาที่แสดงต่างกัน (08:59 ที่กรุงเทพ vs 21:59 ที่นิวยอร์ก) แต่เป็นจุดเวลาเดียวกันเป๊ะๆ บนเส้นเวลาจริง
- New York แสดงเป็น `EDT` (Eastern Daylight Time) ไม่ใช่ `EST` เพราะช่วงเวลานั้นอยู่ใน Daylight Saving Time — `time.LoadLocation` จัดการ DST ให้อัตโนมัติจากฐานข้อมูล IANA โดยไม่ต้องคำนวณเอง

### `time.LoadLocation` ต้องพึ่งฐานข้อมูล timezone ของระบบ

`time.LoadLocation` จะหาไฟล์ timezone database จากระบบปฏิบัติการ (เช่น `/usr/share/zoneinfo` บน Linux) ถ้าเครื่อง/container ไม่มีฐานข้อมูลนี้ (พบบ่อยใน Docker image แบบ minimal เช่น `scratch` หรือ `alpine` ที่ไม่ได้ลง `tzdata`) จะได้ error `unknown time zone` เสมอ วิธีแก้คือติดตั้ง package `tzdata` ในระบบ หรือ import package พิเศษ `time/tzdata` แบบ blank import เพื่อฝังฐานข้อมูลไว้ใน binary โดยตรง:

```go
import _ "time/tzdata"
```

วิธีนี้ทำให้ binary มีขนาดใหญ่ขึ้นเล็กน้อย แต่รับประกันว่า `LoadLocation` จะทำงานได้แม้ deploy ไปยัง environment ที่ไม่มี timezone database ติดตั้งไว้

---

## 6. เลขคณิตกับเวลา: `Add`, `Sub`, `Since`, `Until`

### `Add` — บวก Duration เข้ากับ `time.Time` ได้ `time.Time` ใหม่

```go
t1 := time.Date(2024, 1, 1, 0, 0, 0, 0, time.UTC)
t2 := t1.Add(48 * time.Hour)
fmt.Println("t1 + 48h:", t2) // 2024-01-03 00:00:00 +0000 UTC
```

ถ้าต้องการบวกแบบ "ปี/เดือน/วัน" (ซึ่งความยาวไม่คงที่ ต่างจาก Duration ที่นับ nanosecond ตรงๆ) ให้ใช้ `AddDate(years, months, days int)` แทน:

```go
nextMonth := t1.AddDate(0, 1, 0) // บวกไปอีก 1 เดือน
```

### `Sub` — ลบสอง `time.Time` ได้ `time.Duration`

```go
diff := t2.Sub(t1)
fmt.Println("diff:", diff) // 48h0m0s
```

### `Since` และ `Until` — ทางลัดที่ใช้บ่อยที่สุด

```go
start := time.Now()
// ... ทำงานบางอย่าง ...
elapsed := time.Since(start) // เท่ากับ time.Now().Sub(start)

deadline := time.Now().Add(1 * time.Hour)
remaining := time.Until(deadline) // เท่ากับ deadline.Sub(time.Now())
```

`time.Since(t)` และ `time.Until(t)` เป็นฟังก์ชันที่พบบ่อยมากในโค้ดจริง โดยเฉพาะการวัดเวลาที่ใช้ในการทำงาน (เช่น profiling แบบง่ายๆ ก่อนจะเรียนเรื่อง `pprof` แบบเจาะลึกใน **Part 083**) หรือคำนวณเวลาที่เหลือก่อนหมด deadline (จะเจอบ่อยมากตอนเรียน `context` ใน **Part 032** และ **Part 043**)

```go
package main

import (
	"fmt"
	"time"
)

func main() {
	start := time.Now()
	time.Sleep(10 * time.Millisecond)
	elapsed := time.Since(start)
	fmt.Println("ใช้เวลาไป:", elapsed > 0) // true เสมอ เพราะเวลาผ่านไปแล้วต้องเป็นบวก
}
```

---

## 7. การเปรียบเทียบเวลา: `Before`, `After`, `Equal`

`time.Time` **ห้ามเปรียบเทียบด้วย `==` โดยตรง** (หรือถ้าเปรียบเทียบได้ก็อาจให้ผลลัพธ์ที่ไม่ตรงกับที่คาดหวัง) เพราะ `time.Time` อาจมี location, monotonic reading ที่ต่างกันแม้จะแทนจุดเวลาเดียวกัน จึงต้องใช้ method เฉพาะเสมอ:

```go
t1 := time.Date(2024, 1, 1, 0, 0, 0, 0, time.UTC)
t2 := t1.Add(48 * time.Hour)

fmt.Println("t1 before t2:", t1.Before(t2)) // true
fmt.Println("t1 after t2:", t1.After(t2))   // false
fmt.Println("t1 equal t1:", t1.Equal(t1))   // true
```

- `.Before(u)` — คืน `true` ถ้าเวลาปัจจุบันมาก่อน `u`
- `.After(u)` — คืน `true` ถ้าเวลาปัจจุบันมาหลัง `u`
- `.Equal(u)` — เปรียบเทียบว่าเป็นจุดเวลาเดียวกันจริงหรือไม่ (คำนึงถึง time zone ที่ต่างกันให้ถูกต้อง ต่างจาก `==` ที่เปรียบเทียบ field ภายในแบบตรงๆ)

**กฎทอง**: ใช้ `Before`/`After`/`Equal` เสมอ อย่าใช้ `==`, `<`, `>` กับ `time.Time`

---

## 8. `time.Sleep` และการหน่วงเวลา

`time.Sleep(d Duration)` หยุด goroutine ปัจจุบันไว้ตามระยะเวลาที่กำหนด:

```go
fmt.Println("เริ่ม")
time.Sleep(2 * time.Second)
fmt.Println("ผ่านไป 2 วินาที")
```

`time.Sleep` เหมาะกับสคริปต์ง่ายๆ หรือการหน่วงเวลาระหว่าง retry แต่ **ไม่ควรใช้เป็นกลไกหลักในการควบคุม concurrency** (เช่น ใช้ sleep แทนการรอ goroutine อื่นทำงานเสร็จ) เพราะเป็นการเดาเวลาที่ไม่แม่นยำและสิ้นเปลือง — เราจะเรียนเครื่องมือที่เหมาะสมกว่าอย่าง `sync.WaitGroup` ใน **Part 039** และ channel ใน **Part 037**

---

## 9. Timer และ Ticker: ตัวอย่างเบื้องต้น

Package `time` มีเครื่องมือสำหรับทำงานร่วมกับ **goroutine และ channel** (ที่จะเรียนเจาะลึกใน **ภาคที่ 3: Concurrency**) โดยเฉพาะ `time.After`, `time.Tick`, `time.NewTimer`, และ `time.NewTicker` — บทนี้จะแนะนำแค่ผิวเผินให้รู้จักหน้าตาและวิธีใช้เบื้องต้น ส่วนการใช้งานจริงร่วมกับ `select` และ `context` จะเรียนแบบเจาะลึกใน **Part 043 (Context กับ Concurrency)**

### `time.NewTimer` — จับเวลาครั้งเดียว

```go
timer := time.NewTimer(50 * time.Millisecond)
<-timer.C // block รอจนกว่า timer จะครบกำหนด แล้วส่งค่าเวลาปัจจุบันเข้า channel C
fmt.Println("timer fired")
```

`timer.C` เป็น channel ชนิด `<-chan time.Time` ที่จะได้รับค่าเพียง**ครั้งเดียว**เมื่อครบกำหนดเวลา ถ้าต้องการยกเลิก timer ก่อนครบกำหนด ใช้ `timer.Stop()`

### `time.NewTicker` — ทำงานซ้ำเป็นรอบ

```go
ticker := time.NewTicker(20 * time.Millisecond)
count := 0
for range ticker.C {
	count++
	if count >= 3 {
		ticker.Stop() // สำคัญมาก! ต้อง Stop เสมอไม่งั้น goroutine รั่วไหล (leak)
		break
	}
}
fmt.Println("ticker ticked", count, "times")
```

`ticker.C` จะส่งค่าเวลาเข้ามาซ้ำๆ **ทุกช่วงเวลาที่กำหนด** ไปเรื่อยๆ จนกว่าจะเรียก `Stop()` — เหมาะกับงานที่ต้องทำซ้ำเป็นจังหวะ เช่น ส่ง heartbeat, polling สถานะ, หรือรายงานสถิติเป็นระยะ

### `time.After` — ทางลัดสำหรับใช้ร่วมกับ `select`

```go
select {
case <-time.After(30 * time.Millisecond):
	fmt.Println("time.After fired")
}
```

`time.After(d)` คืน channel ที่จะได้รับค่าเมื่อเวลาผ่านไปครบ `d` — มักใช้คู่กับ `select` เพื่อทำ **timeout pattern** (เช่น "รอ operation ให้เสร็จ ถ้าเกิน 5 วินาทีให้ยกเลิก") ซึ่งเป็นรูปแบบที่ใช้บ่อยมากในโปรแกรม concurrent ระดับ production

รันโค้ดทั้งสามตัวอย่างรวมกันได้ผลลัพธ์:

```
timer fired
ticker ticked 3 times
time.After fired
```

**คำเตือนสำคัญ**: `time.Tick(d)` (ตัวพิมพ์เล็ก ไม่มี `New` นำหน้า) มีพฤติกรรมคล้าย `NewTicker(d).C` แต่ **ไม่มีทางหยุดหรือคืนทรัพยากรได้เลย** เพราะไม่คืนค่า `*Ticker` ให้เรียก `Stop()` เอกสารทางการของ Go แนะนำให้ใช้ `time.Tick` เฉพาะในโปรแกรมที่รันตลอดอายุของโปรแกรม (เช่นทำงานตลอดไปไม่มีวันจบ) เท่านั้น ถ้าอยู่ใน loop ที่สร้างซ้ำๆ ควรใช้ `time.NewTicker` แล้ว `Stop()` ให้เรียบร้อยเสมอเพื่อป้องกัน resource leak

เนื้อหาเรื่อง goroutine leak, `select`, และการผสาน timer/ticker เข้ากับ `context.Context` แบบเต็มรูปแบบจะเรียนอย่างละเอียดใน **Part 038 (select)** และ **Part 043 (Context กับ Concurrency)**

---

## 10. กับดักสำคัญ: Monotonic Clock vs Wall Clock

นี่คือหนึ่งในเรื่องที่แม้แต่โปรแกรมเมอร์ Go ที่มีประสบการณ์ก็ยังพลาดได้ ถ้าไม่เข้าใจกลไกนี้

### ปัญหาของ Wall Clock

**Wall clock** (นาฬิกาผนัง) คือเวลาตามปฏิทิน/นาฬิกาจริงที่แสดงบนหน้าจอ — ปัญหาคือนาฬิการะบบสามารถ **ถูกปรับเปลี่ยนได้ตลอดเวลา** เช่น:

- ผู้ดูแลระบบปรับเวลาด้วยมือ
- ระบบซิงค์เวลาผ่าน NTP (Network Time Protocol) แล้วพบว่านาฬิกาเดินคลาดเคลื่อน จึงปรับ (อาจปรับถอยหลังได้ด้วย!)
- Daylight Saving Time เปลี่ยนเวลาไปมา

ถ้าเราวัดระยะเวลาด้วยการลบ wall clock สองค่า (`t2 - t1`) แล้วนาฬิการะบบถูกปรับระหว่างทาง ผลลัพธ์ที่ได้อาจ**ติดลบ**หรือผิดเพี้ยนไปมากได้ ทั้งที่ในความเป็นจริงเวลาผ่านไปตามปกติ

### ทางแก้: Monotonic Clock

**Monotonic clock** คือนาฬิกาที่เดินไปข้างหน้าเสมอ **ไม่มีวันถูกปรับย้อนกลับ** เหมาะสำหรับวัด "ระยะเวลาที่ผ่านไป" (elapsed time) โดยเฉพาะ

ตั้งแต่ Go 1.9 เป็นต้นมา `time.Now()` จะแนบค่า monotonic clock reading ไปพร้อมกับ wall clock reading **ในตัวแปร `time.Time` เดียวกันโดยอัตโนมัติ** — สังเกตได้จากส่วนท้าย `m=+...` เวลาพิมพ์ `time.Time` ที่ได้จาก `time.Now()` ตรงๆ:

```go
t1 := time.Now()
fmt.Println("t1:", t1) // มี m=+... ต่อท้าย คือ monotonic reading
```

ผลลัพธ์:

```
t1: 2026-09-26 01:59:47.242468188 +0000 UTC m=+0.000087478
```

**กลไกสำคัญที่ต้องรู้**: เมื่อเราเรียก method เปรียบเทียบหรือคำนวณ เช่น `Sub`, `Before`, `After`, `Equal`, `Since` ระหว่าง `time.Time` สองค่า **ที่ทั้งคู่มี monotonic reading ติดมาด้วย** Go จะ**ใช้ monotonic reading ในการคำนวณโดยอัตโนมัติ** แทน wall clock ทำให้ผลลัพธ์แม่นยำและไม่มีทางติดลบจากปัญหานาฬิกาถูกปรับ

### เมื่อไรที่ monotonic reading จะหายไป (และทำไมถึงเป็นปัญหา)

Monotonic reading จะถูก **ตัดทิ้ง** เมื่อ:

- ใช้ method `.Round(0)` (วิธีมาตรฐานที่เอกสาร Go แนะนำสำหรับตัด monotonic ทิ้งโดยตั้งใจ)
- ผ่านการ `Marshal`/`Unmarshal` เป็น JSON, หรือแปลงเป็น string แล้ว parse กลับ (เช่นผ่าน `.Format()` แล้ว `time.Parse()` กลับมา)
- เก็บลงฐานข้อมูลแล้วดึงกลับมา
- ใช้ `.UTC()`, `.Local()`, `.In()` — เมธอดเหล่านี้ก็ตัด monotonic ทิ้งเช่นกัน

```go
t1 := time.Now()
t2 := t1.Round(0) // ตัด monotonic ออก
fmt.Println("t2 (ตัด monotonic):", t2)
fmt.Println("t1 == t2 (Equal):", t1.Equal(t2)) // true — ยังถือว่าเป็นจุดเวลาเดียวกัน
```

ผลลัพธ์:

```
t2 (ตัด monotonic): 2026-09-26 01:59:47.242468188 +0000 UTC
t1 == t2 (Equal): true
```

**ผลกระทบในโค้ดจริง**: ถ้านำ `time.Time` ที่มาจากแหล่งต่างกัน — ตัวหนึ่งได้จาก `time.Now()` ตรงๆ (มี monotonic) อีกตัวได้จากการอ่านฐานข้อมูลหรือ deserialize JSON (ไม่มี monotonic) — มาเปรียบเทียบกัน Go จะ**เปลี่ยนไปใช้ wall clock ในการเทียบแทนทันที** เพราะเทียบ monotonic reading ข้ามแหล่งที่มาที่ต่างกันไม่ได้ (มันเป็นแค่ตัวเลขที่มีความหมายเทียบกันได้เฉพาะภายในโปรเซสเดียวกันเท่านั้น) และในกรณีนั้นก็จะกลับไปเจอปัญหาความไม่แม่นยำแบบ wall clock เหมือนเดิม

**แนวทางปฏิบัติที่ดี**:

1. เมื่อ**วัดระยะเวลาที่ผ่านไป** ให้ใช้ `time.Since(start)` โดยที่ `start` มาจาก `time.Now()` ในโปรเซสเดียวกันเสมอ — จะได้ประโยชน์จาก monotonic clock เต็มที่ ไม่ต้องกังวลเรื่องนาฬิการะบบถูกปรับระหว่างทาง
2. อย่าเก็บ `time.Time` จาก `time.Now()` ไว้เปรียบเทียบข้ามการ restart โปรแกรม หรือข้ามระบบ (เช่นส่งผ่าน network) แล้วคาดหวังความแม่นยำระดับ monotonic — เพราะพอ serialize/deserialize แล้ว monotonic reading จะหายไปเสมอ
3. ถ้าต้องการเก็บ "เวลาที่เกิดเหตุการณ์" ไว้ใช้อ้างอิงในอนาคต (เช่น timestamp ใน log, ใน database) ให้ใช้ wall clock ตามปกติ (ผ่าน `.Format()` หรือ `.Unix()`) เพราะกรณีนี้ไม่ได้ต้องการวัด "ระยะเวลาที่ผ่านไป" แต่ต้องการ "จุดเวลาที่แน่นอน" ซึ่ง wall clock ทำหน้าที่นี้ได้ถูกต้องอยู่แล้ว

---

## สรุปสิ่งที่ได้เรียนในบทนี้

- `time.Time` แทนจุดเวลาหนึ่งจุด ส่วน `time.Duration` แทนระยะเวลา — แยกกันชัดเจนตั้งแต่ระดับ type
- `time.Now()`, `time.Date()` สร้าง `time.Time` ได้ และมี method ดึงส่วนประกอบ (`Year`, `Month`, `Day`, ...) ครบถ้วน
- Format/Parse ใช้ **reference date** `Mon Jan 2 15:04:05 MST 2006` (จำง่ายด้วยลำดับ 1-2-3-4-5-6-7) แทนสัญลักษณ์แบบ `%Y-%m-%d`
- `time.LoadLocation` โหลด time zone จาก IANA database — ต้องมี `tzdata` ในระบบ หรือ blank-import `time/tzdata`
- เลขคณิตเวลาใช้ `Add` (บวก Duration), `AddDate` (บวกปี/เดือน/วัน), `Sub` (ลบสองเวลาได้ Duration), `Since`/`Until` (ทางลัดที่ใช้บ่อยที่สุด)
- เปรียบเทียบเวลาต้องใช้ `Before`/`After`/`Equal` **ห้ามใช้ `==`**
- `time.Sleep` หน่วงเวลาแบบง่าย ส่วน `NewTimer`/`NewTicker`/`time.After` ใช้ร่วมกับ channel และ `select` สำหรับงาน concurrent (เจาะลึกใน Part 038, 043) — อย่าลืม `Stop()` ticker เสมอเพื่อไม่ให้ resource รั่วไหล
- **Monotonic clock** (เดินหน้าเสมอ) ถูกแนบมากับ wall clock อัตโนมัติใน `time.Now()` ตั้งแต่ Go 1.9 ทำให้ `Sub`/`Since`/`Before`/`After` แม่นยำแม้นาฬิการะบบถูกปรับ — แต่ monotonic reading จะหายไปเมื่อผ่าน `Round(0)`, serialize, หรือแปลง time zone

## แบบฝึกหัดท้ายบท

1. เขียนโปรแกรมที่รับวันเกิดของผู้ใช้ในรูปแบบ `"2006-01-02"` แล้วคำนวณว่าอายุปัจจุบันกี่ปี กี่วัน (ใช้ `time.Parse`, `time.Since` หรือ `AddDate` ช่วยคำนวณ)
2. เขียนฟังก์ชันที่รับเวลาสองค่าเป็น `time.Time` แล้วคืนค่าว่าเวลาไหนมาก่อน โดยใช้ `Before`/`After` (ห้ามใช้ `==` หรือ `<`)
3. ทดลองใช้ `time.LoadLocation` โหลด time zone ของ 3 เมืองที่ต่างกัน (เช่น `Asia/Tokyo`, `Europe/London`, `America/Los_Angeles`) แล้วแสดงเวลาปัจจุบันของทั้ง 3 เมืองพร้อมกันในรูปแบบเดียวกัน
4. เขียนโปรแกรมที่ใช้ `time.NewTicker` พิมพ์ "กำลังทำงาน..." ทุก 1 วินาที รวม 5 ครั้ง แล้ว `Stop()` ให้เรียบร้อย
5. อธิบายด้วยคำพูดตัวเอง (เขียนเป็นคอมเมนต์ในโค้ด) ว่าทำไมการเปรียบเทียบ `time.Time` สองค่าด้วย `==` ถึงเป็นความคิดที่ไม่ปลอดภัย ยกตัวอย่างสถานการณ์ที่อาจเกิดปัญหา
6. เขียนฟังก์ชันวัดเวลาที่ใช้ในการรันฟังก์ชันอื่น (higher-order function) โดยรับ `func()` เป็น argument แล้วคืนค่า `time.Duration` ที่ใช้ไป ใช้หลักการจาก `time.Since` ร่วมกับ closure ที่เรียนจาก **Part 009**

---

**ต่อไป**: [Part 023 — Regular Expressions ด้วย `regexp`](./023-regexp.md)
