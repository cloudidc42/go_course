# Part 059: RESTful API Design หลักการ

> ภาคที่ 5: Web Development — ตอนที่ 4 จาก 15 (Part 56–70)

## สารบัญของบทนี้

1. REST คืออะไรกันแน่ (ไม่ใช่แค่ "API ที่ใช้ JSON")
2. หลักการของ REST: Resource, Representation, Statelessness
3. ความหมายที่แท้จริงของแต่ละ HTTP Method และ Idempotency
4. หลักการตั้งชื่อ Resource (Naming Convention)
5. HTTP Status Code: ใช้ตัวไหนเมื่อไหร่
6. Pagination: Offset-based vs Cursor-based
7. Filtering และ Sorting ผ่าน Query Parameter
8. Versioning: URL Path vs Header
9. ออกแบบ JSON Error Envelope ที่สอดคล้องกันทั้ง API
10. HATEOAS: อุดมคติของ REST ที่ API ส่วนใหญ่เลือกข้าม
11. สรุปสิ่งที่ได้เรียนในบทนี้
12. แบบฝึกหัดท้ายบท

---

## 1. REST คืออะไรกันแน่ (ไม่ใช่แค่ "API ที่ใช้ JSON")

หลายคนเข้าใจว่า "REST API" คือ "API ที่รับส่งข้อมูลเป็น JSON ผ่าน HTTP" ซึ่งเป็นความเข้าใจที่ไม่ผิดทั้งหมดแต่ก็ไม่ครบถ้วน **REST (Representational State Transfer)** เป็นชื่อของ**สถาปัตยกรรม (architectural style)** ที่ Roy Fielding เสนอไว้ในวิทยานิพนธ์ปริญญาเอกปี 2000 โดยมี**ข้อบังคับ (constraints)** ชุดหนึ่งที่ระบบต้องทำตามถึงจะเรียกว่า REST ได้เต็มรูปแบบ

ในทางปฏิบัติ API ส่วนใหญ่ในโลกที่เรียกตัวเองว่า "RESTful" จะทำตามหลักการบางส่วนเท่านั้น (ไม่ใช่ REST แบบสมบูรณ์ตามตำรา) — บทนี้จะพาไปดูหลักการที่**นำไปใช้ได้จริง** และเป็นที่ยอมรับกันในอุตสาหกรรม มากกว่าไล่ตามความ "บริสุทธิ์" ทางทฤษฎีทั้งหมด (ซึ่งหัวข้อ 10 จะพูดถึงส่วนที่ API ส่วนใหญ่ "เลือกข้าม" อย่างตรงไปตรงมา)

---

## 2. หลักการของ REST: Resource, Representation, Statelessness

### Resource (ทรัพยากร)

หัวใจของ REST คือทุกอย่างในระบบถูกมองเป็น **"resource"** — สิ่งของบางอย่างที่มีตัวตนและเข้าถึงได้ผ่าน URL เฉพาะของมัน เช่น "note ตัวที่ 5" คือ resource หนึ่งตัว เข้าถึงได้ที่ `/notes/5` "รายการ note ทั้งหมด" ก็เป็น resource เช่นกัน เข้าถึงได้ที่ `/notes`

ข้อสำคัญ: resource ไม่ใช่ "action" หรือ "verb" — URL ที่ดีจะไม่มีคำกริยาอยู่ใน path เช่น `/getNotes` หรือ `/createNote` (นี่คือรูปแบบของ RPC ไม่ใช่ REST) แต่ใช้ HTTP method (`GET`, `POST`) เป็นตัวบอก action แทน โดยที่ URL คงที่เป็นชื่อ resource เสมอ

### Representation (การนำเสนอ)

resource ตัวเดียวกันสามารถมี "representation" ได้หลายรูปแบบ — note ตัวเดียวกันอาจแสดงเป็น JSON, XML หรือ HTML ก็ได้ขึ้นกับที่ client ร้องขอ (ผ่าน header `Accept`) ในทางปฏิบัติปัจจุบัน API ส่วนใหญ่เลือกใช้ JSON เป็น representation เดียวเพื่อความเรียบง่าย แต่หลักการเรื่อง representation ที่แยกออกจากตัว resource เองยังคงเป็นแนวคิดสำคัญ — resource คือ "แนวคิด" ส่วน representation คือ "รูปแบบข้อมูลที่ส่งออกมาจริง"

### Statelessness (ไม่มีสถานะ)

นี่คือข้อบังคับที่สำคัญที่สุดข้อหนึ่งของ REST: **แต่ละ request ต้องมีข้อมูลครบถ้วนในตัวมันเองที่จะให้ server ประมวลผลได้ โดยไม่ต้องพึ่งข้อมูลที่เก็บไว้จาก request ก่อนหน้า** server จะไม่จำ "session state" ของ client ไว้ระหว่าง request (ต่างจากเว็บแอปแบบเก่าที่ใช้ session เก็บสถานะฝั่ง server)

ผลที่ตามมาของ statelessness:

- ทุก request ที่ต้องยืนยันตัวตนต้องแนบ token หรือ credential มาด้วยทุกครั้ง (เช่น header `Authorization: Bearer <token>` ที่เจอใน Part 057)
- server สามารถเพิ่มจำนวน instance ได้ตามใจ (horizontal scaling) โดยไม่ต้องกังวลเรื่อง "request ต้องไปตกที่ server เดิมที่เก็บ state ไว้" — เพราะไม่มี state เก็บไว้ที่ server เลย
- ถ้าต้องการ state จริงๆ (เช่น sesssion login) ต้องเก็บไว้ฝั่ง client (cookie/token) หรือ shared store ภายนอก (เช่น Redis ที่จะเรียนใน **Part 077**) ไม่ใช่ memory ของ server instance ใดตัวหนึ่ง

---

## 3. ความหมายที่แท้จริงของแต่ละ HTTP Method และ Idempotency

REST ใช้ HTTP method ที่มีอยู่แล้วเพื่อสื่อความหมายของ action แทนที่จะสร้างคำกริยาใหม่ใน URL แต่ละ method มีความหมายเฉพาะตัวที่ควรยึดตามอย่างเคร่งครัด:

| Method | ความหมาย | ควรมี body ไหม | **Safe** (ไม่แก้ข้อมูล) | **Idempotent** (เรียกซ้ำผลเหมือนเดิม) |
|---|---|---|---|---|
| `GET` | อ่านข้อมูล resource | ไม่ควรมี | ใช่ | ใช่ |
| `POST` | สร้าง resource ใหม่ (หรือ action ที่ไม่ idempotent) | มี | ไม่ | ไม่ |
| `PUT` | แทนที่ resource ทั้งตัวด้วยข้อมูลใหม่ | มี | ไม่ | ใช่ |
| `PATCH` | แก้ไขบางส่วนของ resource | มี | ไม่ | ไม่รับประกัน (ขึ้นกับ implementation) |
| `DELETE` | ลบ resource | ไม่ควรมี | ไม่ | ใช่ |

### Idempotency คืออะไร และทำไมสำคัญ

**Idempotent** หมายถึง: ไม่ว่าจะเรียก operation นั้นกี่ครั้งด้วย input เดิม ผลลัพธ์สุดท้ายในระบบจะเหมือนกันเสมอ (ต่างจาก "ไม่มี side effect" — idempotent operation มี side effect ได้ แค่เรียกซ้ำแล้วผลลัพธ์ไม่เปลี่ยนแปลงเพิ่มเติม)

ตัวอย่างที่ทำให้เห็นภาพชัด:

- `DELETE /notes/5` — เรียกครั้งแรก note หายไป (204) เรียกซ้ำครั้งที่สอง note ก็ยังหายอยู่เหมือนเดิม (มักตอบ 404 เพราะไม่มีอยู่แล้ว) แต่ **สถานะสุดท้ายของระบบเหมือนกัน** ไม่ว่าเรียกกี่ครั้ง → idempotent
- `PUT /notes/5` ที่ส่ง `{"title": "Updated"}` — เรียกกี่ครั้งผลลัพธ์คือ note 5 มี title เป็น "Updated" เสมอ → idempotent
- `POST /notes` ที่ส่ง `{"title": "New note"}` — เรียกครั้งแรกได้ note ใหม่ 1 อัน เรียกซ้ำอีกครั้งได้ note ใหม่**อีกอัน** (มี id ต่างกัน) → **ไม่ idempotent** เพราะเรียกซ้ำแล้วสถานะระบบเปลี่ยนไปเรื่อยๆ (มี note เพิ่มขึ้นทุกครั้ง)

ทำไมเรื่องนี้สำคัญในทางปฏิบัติ: เวลาเครือข่ายมีปัญหา (timeout, connection reset) client มักจะ **retry request อัตโนมัติ** ถ้า operation เป็น idempotent การ retry ปลอดภัย 100% แต่ถ้าไม่ idempotent (เช่น `POST` สร้างข้อมูล) การ retry อาจสร้างข้อมูลซ้ำโดยไม่ตั้งใจ — นี่คือเหตุผลที่ระบบที่ต้องการความน่าเชื่อถือสูง (เช่นระบบชำระเงิน) มักมีแนวคิด **idempotency key** เพิ่มเติมสำหรับ `POST` เพื่อให้ retry ปลอดภัยด้วย (จะเจอแนวคิดนี้อีกครั้งใน **Part 094: Circuit Breaker และ Retry Pattern**)

---

## 4. หลักการตั้งชื่อ Resource (Naming Convention)

แนวทางการตั้งชื่อ URL ที่ยอมรับกันอย่างกว้างขวางในอุตสาหกรรม:

### ใช้คำนามพหูพจน์เสมอ

```
✅ GET  /users
✅ GET  /users/42
❌ GET  /user
❌ GET  /getUsers
```

การใช้พหูพจน์เสมอ (ไม่ว่าจะหมายถึง collection หรือ item เดียว) ทำให้ URL คาดเดาได้ง่าย ไม่ต้องจำว่า resource ไหนใช้เอกพจน์ resource ไหนใช้พหูพจน์

### Nested resource สำหรับความสัมพันธ์แบบเป็นเจ้าของ (ownership)

```
GET  /users/42/orders          รายการ order ทั้งหมดของ user คนที่ 42
GET  /users/42/orders/7        order ที่ 7 ของ user คนที่ 42
POST /users/42/orders          สร้าง order ใหม่ให้ user คนที่ 42
```

Nesting ควรมีความลึกไม่เกิน 2-3 ชั้นเพื่อไม่ให้ URL ยาวเกินไปอ่านยาก ถ้าจำเป็นต้องเข้าถึง order โดยตรงโดยไม่ผ่าน user ก็ควรมี endpoint แบบ flat เสริมไว้ด้วย เช่น `GET /orders/7` — เลือกใช้ nested หรือ flat form ขึ้นกับบริบทของการใช้งานจริง

### ใช้ HTTP method บอก action ไม่ใช่คำกริยาใน URL

```
✅ POST   /notes         (สร้าง note)
✅ DELETE /notes/5       (ลบ note)
❌ POST   /notes/create
❌ POST   /notes/5/delete
```

### กรณีพิเศษ: action ที่ไม่ใช่ CRUD ตรงๆ

บางครั้งมี action ที่ไม่ใช่ create/read/update/delete ตรงๆ เช่น "ส่ง email ยืนยัน" หรือ "publish บทความ" กรณีนี้ยอมให้ใช้คำกริยาเป็น sub-resource ได้ (เป็นข้อยกเว้นที่ยอมรับกันทั่วไป):

```
POST /articles/5/publish
POST /users/42/password-reset-emails
```

มองว่า "publish" เป็นเหมือน sub-resource ที่ถูกสร้างขึ้น (create การ publish) มากกว่าเป็นคำกริยาลอยๆ

---

## 5. HTTP Status Code: ใช้ตัวไหนเมื่อไหร่

การเลือก status code ที่ถูกต้องคือสิ่งที่ทำให้ API "พูดภาษาเดียวกัน" กับ client ทุกตัวที่มาเรียกใช้ โดยไม่ต้องอ่าน documentation ทุกครั้งเพื่อรู้ว่าเกิดอะไรขึ้น

| Status | ชื่อ | ใช้เมื่อไหร่ |
|---|---|---|
| **200** | OK | request สำเร็จ และมีข้อมูลส่งกลับ (GET, PUT, PATCH ที่สำเร็จ) |
| **201** | Created | สร้าง resource ใหม่สำเร็จ (POST ที่สร้างของใหม่) ควรมี header `Location` ชี้ไปยัง resource ที่สร้าง |
| **204** | No Content | สำเร็จแต่ไม่มีข้อมูลส่งกลับ (DELETE ที่สำเร็จ, PUT ที่ไม่ต้องการคืนข้อมูล) |
| **400** | Bad Request | request ผิดรูปแบบ (malformed JSON, query param ผิด type) — เป็นปัญหาที่ client ควรแก้ได้จากข้อความ error |
| **401** | Unauthorized | ไม่ได้ยืนยันตัวตน หรือ token ไม่ถูกต้อง/หมดอายุ (ที่จริงชื่อ "Unauthorized" ทำให้เข้าใจผิด — ความหมายจริงคือ "ยังไม่รู้ว่าเป็นใคร") |
| **403** | Forbidden | ยืนยันตัวตนแล้ว แต่ไม่มีสิทธิ์เข้าถึง resource นี้ (รู้ว่าเป็นใคร แต่ไม่อนุญาต) |
| **404** | Not Found | ไม่พบ resource ที่ระบุ (id ไม่มีอยู่จริง หรือ path ไม่มีอยู่เลย) |
| **409** | Conflict | request ขัดแย้งกับสถานะปัจจุบันของ resource (เช่น สร้าง user ด้วย email ที่มีอยู่แล้ว, แก้ไขข้อมูลที่ถูกคนอื่นแก้ไปแล้วแบบ optimistic locking) |
| **422** | Unprocessable Entity | รูปแบบ request ถูกต้อง (เป็น JSON ที่ถูกต้อง) แต่ข้อมูลไม่ผ่าน validation เชิงธุรกิจ (เช่น email format ผิด, ค่าติดลบที่ไม่ควรติดลบ) |
| **429** | Too Many Requests | เกิน rate limit ที่กำหนดไว้ |
| **500** | Internal Server Error | เกิดข้อผิดพลาดฝั่ง server ที่ไม่คาดคิด (bug, panic ที่ recover ได้ตาม Part 057) |

### แยกให้ออก: 400 vs 422

จุดที่คนสับสนบ่อยที่สุดคือ 400 กับ 422 — หลักการแยกที่ใช้ได้จริง:

- **400**: JSON เองผิดรูปแบบ (parse ไม่ได้เลย) เช่น `{"title": ` (ขาดปิดวงเล็บ) หรือ field เป็น type ผิด เช่นส่ง `"title": 123` ทั้งที่ควรเป็น string
- **422**: JSON ถูกต้องสมบูรณ์ decode ผ่านหมด แต่ค่าที่ได้ไม่ผ่านกฎทางธุรกิจ เช่น `title` เป็น string ว่างเปล่าทั้งที่ต้องมีค่า หรือ `age` เป็น `-5` ทั้งที่ต้องเป็นบวก

การแยกสองอย่างนี้ให้ชัดเจนช่วยให้ client เขียนโค้ดจัดการ error ได้ตรงจุด — 400 มักหมายถึงบั๊กในโค้ดฝั่ง client (ส่ง JSON ผิดรูปแบบ) ในขณะที่ 422 หมายถึง input ของ user ไม่ถูกต้องซึ่งควรแสดงข้อความให้ user แก้ไข

---

## 6. Pagination: Offset-based vs Cursor-based

เมื่อ resource มีจำนวนมาก (เช่น note นับล้านตัว) การส่งข้อมูลทั้งหมดกลับมาในครั้งเดียวเป็นไปไม่ได้ในทางปฏิบัติ — ต้องแบ่งส่งเป็นหน้าๆ (pagination) มีสองแนวทางหลักที่ใช้กันในอุตสาหกรรม:

### Offset-based pagination

```
GET /notes?limit=20&offset=40
```

```go
type PageParams struct {
	Limit  int
	Offset int
}

// parseOffsetPagination อ่าน query param ?limit=&offset= พร้อมค่า default
// และเพดานสูงสุด (ป้องกัน client ขอ limit สูงเกินไปจนกิน resource server)
func parseOffsetPagination(q url.Values) PageParams {
	limit, err := strconv.Atoi(q.Get("limit"))
	if err != nil || limit <= 0 {
		limit = 20
	}
	if limit > 100 {
		limit = 100
	}
	offset, err := strconv.Atoi(q.Get("offset"))
	if err != nil || offset < 0 {
		offset = 0
	}
	return PageParams{Limit: limit, Offset: offset}
}
```

ทดสอบจริง:

```go
q1, _ := url.ParseQuery("limit=10&offset=20")
fmt.Printf("%+v\n", parseOffsetPagination(q1))
// {Limit:10 Offset:20}

q2, _ := url.ParseQuery("limit=99999&offset=-5")
fmt.Printf("%+v\n", parseOffsetPagination(q2))
// {Limit:100 Offset:0}  -- limit ถูกจำกัดเพดาน, offset ติดลบถูกปัดเป็น 0
```

**ข้อดี**: เข้าใจง่าย กระโดดไปหน้าไหนก็ได้ตรงๆ (เช่น "ไปหน้า 5") เหมาะกับ UI ที่มีเลขหน้าให้กด

**ข้อเสีย**: performance แย่ลงเมื่อ offset สูงมาก (database ต้อง scan ข้ามแถวที่ offset ไปจนถึงตำแหน่งที่ต้องการ) และถ้ามีข้อมูลถูกเพิ่ม/ลบระหว่างที่ client กำลังเปิดดูหลายหน้า อาจเห็นข้อมูลซ้ำหรือขาดหายไปได้ (เพราะตำแหน่ง "offset ที่ 40" ขยับไปตามจำนวนแถวที่เปลี่ยนแปลง)

### Cursor-based pagination

```
GET /notes?limit=20&cursor=NDI=
```

```go
// encodeCursor/decodeCursor เข้ารหัส "ตำแหน่งล่าสุดที่อ่านถึง" เป็น opaque
// string ที่ client ส่งกลับมาในหน้าถัดไป โดยไม่ต้องรู้ว่าข้างในเก็บอะไร
func encodeCursor(lastID int) string {
	return base64.URLEncoding.EncodeToString([]byte(strconv.Itoa(lastID)))
}

func decodeCursor(cursor string) (int, error) {
	if cursor == "" {
		return 0, nil
	}
	b, err := base64.URLEncoding.DecodeString(cursor)
	if err != nil {
		return 0, fmt.Errorf("invalid cursor")
	}
	return strconv.Atoi(string(b))
}
```

ทดสอบจริง:

```go
c := encodeCursor(42)
fmt.Println("cursor:", c) // cursor: NDI=
id, _ := decodeCursor(c)
fmt.Println("decoded back to lastID:", id) // decoded back to lastID: 42
```

**ข้อดี**: performance คงที่ไม่ว่าจะอยู่หน้าไหน (database ค้นจาก index ของ id/timestamp ล่าสุดโดยตรง ไม่ต้อง scan ข้าม) และปลอดภัยจากปัญหาข้อมูลซ้ำ/หายเมื่อมีการเพิ่ม-ลบข้อมูลระหว่างเปิดดู

**ข้อเสีย**: กระโดดไปหน้าที่ไม่ติดกันโดยตรงไม่ได้ (ต้องไล่ทีละหน้าเท่านั้น) ไม่เหมาะกับ UI แบบมีเลขหน้าให้กดเลือก

### เลือกแบบไหนดี

ใช้ **offset-based** เมื่อ UI ต้องการแสดงเลขหน้าและให้ user กระโดดข้ามหน้าได้ และข้อมูลมีจำนวนไม่มากเกินไป ใช้ **cursor-based** เมื่อข้อมูลมีปริมาณมาก (feed แบบ infinite scroll, log, timeline) และต้องการ performance ที่สม่ำเสมอไม่ว่าจะเลื่อนลึกแค่ไหน — API ขนาดใหญ่อย่าง Twitter/X, Stripe, GitHub ล้วนใช้ cursor-based pagination เป็นหลักในปัจจุบัน

---

## 7. Filtering และ Sorting ผ่าน Query Parameter

รูปแบบที่ใช้กันแพร่หลาย: filter ผ่าน query param ตรงชื่อ field, sort ผ่าน param `sort` โดยใช้ `-` นำหน้าหมายถึงเรียงจากมากไปน้อย:

```
GET /products?tag=book&sort=-price
```

```go
type Product struct {
	ID    int
	Name  string
	Price int
	Tag   string
}

// applyFilterAndSort ใช้กับ query แบบ ?tag=book&sort=-price (นำหน้าด้วย - คือ
// เรียงจากมากไปน้อย) — pattern ที่ REST API จำนวนมากใช้กันจริง
func applyFilterAndSort(products []Product, q url.Values) []Product {
	tag := q.Get("tag")
	result := make([]Product, 0, len(products))
	for _, p := range products {
		if tag != "" && p.Tag != tag {
			continue
		}
		result = append(result, p)
	}

	sortParam := q.Get("sort")
	if sortParam == "" {
		return result
	}
	desc := false
	field := sortParam
	if field[0] == '-' {
		desc = true
		field = field[1:]
	}
	sort.Slice(result, func(i, j int) bool {
		var less bool
		switch field {
		case "price":
			less = result[i].Price < result[j].Price
		case "name":
			less = result[i].Name < result[j].Name
		default:
			less = result[i].ID < result[j].ID
		}
		if desc {
			return !less
		}
		return less
	})
	return result
}
```

ทดสอบจริง:

```go
products := []Product{
	{1, "Go in Action", 350, "book"},
	{2, "Mechanical Keyboard", 2500, "gadget"},
	{3, "The Go Programming Language", 420, "book"},
}
q, _ := url.ParseQuery("tag=book&sort=-price")
for _, p := range applyFilterAndSort(products, q) {
	fmt.Printf("  %+v\n", p)
}
```

ผลลัพธ์:

```
  {ID:3 Name:The Go Programming Language Price:420 Tag:book}
  {ID:1 Name:Go in Action Price:350 Tag:book}
```

เห็นว่าตัวที่ `Tag` ไม่ตรง (`gadget`) ถูกกรองออก และผลลัพธ์ที่เหลือถูกเรียงจากราคามากไปน้อยตามที่ `sort=-price` ระบุ — `sort.Slice` ที่ใช้ตรงนี้ทบทวนได้จาก **Part 027**

---

## 8. Versioning: URL Path vs Header

API จะต้องเปลี่ยนแปลงไปตามเวลา แต่ client เก่าที่ยังพึ่งพา response รูปแบบเดิมต้องยังใช้งานได้ต่อไป — นี่คือเหตุผลที่ต้องมี **versioning** สองแนวทางหลักที่ใช้กัน:

### Versioning ผ่าน URL path

```go
// versionByPath สาธิตการแยกเวอร์ชันด้วย path prefix (/v1/, /v2/) ซึ่งเห็นชัด
// ในตัว URL ทันที เดาง่าย cache ง่าย เป็นแนวทางที่ REST API สาธารณะส่วนใหญ่ใช้
func versionByPath(mux *http.ServeMux) {
	mux.HandleFunc("GET /v1/users/{id}", func(w http.ResponseWriter, r *http.Request) {
		w.Write([]byte("v1 user " + r.PathValue("id")))
	})
	mux.HandleFunc("GET /v2/users/{id}", func(w http.ResponseWriter, r *http.Request) {
		w.Write([]byte("v2 user " + r.PathValue("id") + " (with extra fields)"))
	})
}
```

**ข้อดี**: เห็นเวอร์ชันชัดเจนในตัว URL เดาง่าย เปิดใน browser ตรงๆ ก็รู้ว่าเรียกเวอร์ชันไหน cache ผ่าน CDN ได้ง่าย (เพราะ URL ต่างกันชัดเจน)

**ข้อเสีย**: การแยกเวอร์ชันของทั้ง resource ทำให้ URL "ซ้ำซ้อน" ในทางความหมาย (`/v1/users/5` กับ `/v2/users/5` คือ resource เดียวกันแท้ๆ)

### Versioning ผ่าน Header

```go
// versionByHeader สาธิตการแยกเวอร์ชันด้วย custom header เช่น
// "Accept: application/vnd.myapi.v2+json" — URL สะอาดกว่า แต่ debug ยากกว่า
// เพราะเปิดลิงก์ใน browser ตรงๆ จะได้ version default เสมอ
func versionByHeader(w http.ResponseWriter, r *http.Request) {
	accept := r.Header.Get("Accept")
	switch accept {
	case "application/vnd.myapi.v2+json":
		w.Write([]byte("v2 response"))
	default:
		w.Write([]byte("v1 response (default)"))
	}
}
```

**ข้อดี**: URL เดียวคงที่สำหรับ resource เดียวกันเสมอ (สอดคล้องกับหลักการ REST ที่ URL ควรระบุ resource ไม่ใช่ representation) เหมาะกับ API ระดับ enterprise ที่ควบคุม client ได้ชัดเจน

**ข้อเสีย**: debug ยากกว่า — เปิด URL ใน browser ตรงๆ ไม่เห็นว่ากำลังเรียกเวอร์ชันไหน ต้องตั้งค่า header เองทุกครั้งเวลาทดสอบด้วยเครื่องมืออย่าง `curl` หรือ Postman

**แนวทางที่ใช้บ่อยที่สุดในทางปฏิบัติ**: API สาธารณะขนาดใหญ่ส่วนมาก (GitHub, Stripe บางส่วน) เลือกใช้ **URL path versioning** เพราะความชัดเจนและง่ายต่อการ debug ชนะเรื่อง "ความบริสุทธิ์ทางทฤษฎี" ไปในทางปฏิบัติ — Stripe เองใช้วิธีผสม (header ระบุวันที่ของ API version แทนเลขเวอร์ชัน)

---

## 9. ออกแบบ JSON Error Envelope ที่สอดคล้องกันทั้ง API

หนึ่งในสิ่งที่ทำให้ API ใช้งานยากที่สุดคือ error response ที่มีรูปแบบไม่สม่ำเสมอ — บาง endpoint ตอบ `{"error": "..."}` บาง endpoint ตอบ `{"message": "..."}` บาง endpoint ตอบ string เปล่าๆ แนวทางที่ดีคือออกแบบ **error envelope** เดียวที่ใช้กับทุก endpoint ตั้งแต่แรก:

```go
type ErrorDetail struct {
	Message string            `json:"message"`
	Fields  map[string]string `json:"fields,omitempty"`
}

type ErrorResponse struct {
	Error ErrorDetail `json:"error"`
}
```

ตัวอย่าง response จริงที่ได้จากโครงสร้างนี้:

```json
{
  "error": {
    "message": "validation failed",
    "fields": {
      "title": "field is required",
      "priority": "must be at most 5"
    }
  }
}
```

จุดออกแบบที่สำคัญ:

- **ห่อ error ไว้ใต้ key `"error"` เสมอ** — client ตรวจสอบง่ายๆ ด้วย `if response.error != null` โดยไม่ต้องเดาว่า key ชื่ออะไรในแต่ละ endpoint
- **`message`** เป็นข้อความสรุปสำหรับแสดงหรือ log ทั่วไป ควรเขียนให้อ่านเข้าใจง่าย ไม่ใช่ raw error จาก internal system
- **`fields`** (optional ใช้ `omitempty` เพื่อไม่ต้องมีทุกครั้ง) เก็บ error เฉพาะแต่ละ field เมื่อเป็นปัญหาจาก validation ทำให้ UI ฝั่ง client แสดง error ใต้ input field ที่ถูกต้องได้ตรงตัว
- envelope นี้ใช้ได้กับทุก error code (400, 404, 422, 500) — โครงสร้างเหมือนกันหมด ต่างกันแค่ค่า `message`/`fields` และ HTTP status code เท่านั้น

envelope นี้จะถูกนำไปใช้งานจริงเต็มรูปแบบใน **Part 060** ที่จะสร้าง helper `respondError` ที่ใช้ซ้ำได้ทุก handler

---

## 10. HATEOAS: อุดมคติของ REST ที่ API ส่วนใหญ่เลือกข้าม

**HATEOAS (Hypermedia As The Engine Of Application State)** เป็นหนึ่งในข้อบังคับดั้งเดิมของ REST ตามทฤษฎีของ Roy Fielding ที่บอกว่า **response ควรมีลิงก์บอกว่า "จากตรงนี้ทำอะไรต่อได้บ้าง"** ฝังมาด้วยเสมอ ไม่ใช่ให้ client ต้อง hardcode URL structure ไว้ล่วงหน้า

ตัวอย่าง response แบบ HATEOAS เต็มรูปแบบ:

```json
{
  "id": 5,
  "title": "Buy milk",
  "done": false,
  "_links": {
    "self": { "href": "/notes/5" },
    "delete": { "href": "/notes/5", "method": "DELETE" },
    "update": { "href": "/notes/5", "method": "PATCH" },
    "parent": { "href": "/notes" }
  }
}
```

ในทางทฤษฎี HATEOAS ทำให้ API "ค้นพบตัวเองได้" (self-discoverable) — client ไม่จำเป็นต้องรู้ล่วงหน้าว่า URL ของ action ต่างๆ คืออะไร แค่ตาม link ที่ server ให้มาก็พอ คล้ายกับการเปิดเว็บไซต์แล้วคลิกลิงก์ไปเรื่อยๆ โดยไม่ต้องพิมพ์ URL เอง

**แต่ในทางปฏิบัติ** API ส่วนใหญ่ในโลก — รวมถึง API ระดับใหญ่อย่าง GitHub, Stripe, Twitter — **เลือกที่จะไม่ทำ HATEOAS เต็มรูปแบบ** ด้วยเหตุผลหลักๆ คือ:

1. **ความซับซ้อนที่เพิ่มขึ้นไม่คุ้มกับประโยชน์ที่ได้** ในทางปฏิบัติ client (เช่น mobile app, frontend) แทบทุกตัว hardcode URL structure ไว้ตั้งแต่ตอน develop อยู่แล้ว ไม่ได้ "ค้นพบ" endpoint แบบ dynamic จริงๆ
2. **Documentation (เช่น OpenAPI/Swagger) ทำหน้าที่แทนได้ดีกว่า** — นักพัฒนาอ่าน API docs แล้วเขียนโค้ดเรียก endpoint ตรงๆ ไม่ต้องพึ่งการ "เดินตามลิงก์" แบบ runtime
3. **เพิ่มขนาด response โดยไม่จำเป็น** สำหรับ use case ส่วนใหญ่

**สรุปในทางปฏิบัติ**: ควรรู้จัก HATEOAS ไว้ในฐานะหลักการดั้งเดิมของ REST (และอาจเจอคำถามนี้ในการสัมภาษณ์งาน) แต่ไม่จำเป็นต้องนำมาใช้เต็มรูปแบบในโปรเจกต์จริงส่วนใหญ่ — ยกเว้นกรณีพิเศษเช่น API ที่ต้องรองรับ workflow ที่ซับซ้อนมาก (เช่น payment flow ของ PayPal ที่ใช้ HATEOAS จริงเพื่อบอก client ว่าขั้นตอนถัดไปทำอะไรได้บ้าง)

---

## สรุปสิ่งที่ได้เรียนในบทนี้

- REST คือสถาปัตยกรรมที่มองทุกอย่างเป็น "resource" เข้าถึงผ่าน URL คงที่ ใช้ HTTP method สื่อ action และต้องเป็น stateless (แต่ละ request สมบูรณ์ในตัวเอง ไม่พึ่ง state ที่ server จำไว้)
- `GET`, `PUT`, `DELETE` เป็น idempotent (เรียกซ้ำผลลัพธ์สุดท้ายเหมือนเดิม) ส่วน `POST` ไม่ใช่ — เรื่องนี้สำคัญมากเวลาออกแบบ retry logic
- ตั้งชื่อ resource เป็นคำนามพหูพจน์เสมอ ใช้ HTTP method บอก action ไม่ใช่คำกริยาใน URL
- แยก 400 (JSON ผิดรูปแบบ) กับ 422 (JSON ถูกแต่ข้อมูลไม่ผ่าน validation) ให้ชัดเจน และเลือก status code ที่สื่อความหมายตรงกับสถานการณ์จริงเสมอ
- Offset-based pagination เข้าใจง่ายแต่ performance แย่ลงเมื่อข้อมูลเยอะ, cursor-based pagination performance คงที่แต่กระโดดข้ามหน้าไม่ได้
- Filtering/sorting ผ่าน query param เป็น pattern มาตรฐาน (`?tag=x&sort=-price`)
- Versioning ทำได้ทั้งผ่าน URL path (ชัดเจน debug ง่าย) และ header (URL สะอาดกว่าแต่ debug ยากกว่า) — URL path เป็นที่นิยมมากกว่าในทางปฏิบัติ
- ออกแบบ JSON error envelope เดียวที่ใช้กับทุก endpoint ตั้งแต่แรก ช่วยให้ client จัดการ error ได้สม่ำเสมอ
- HATEOAS เป็นอุดมคติดั้งเดิมของ REST แต่ API ส่วนใหญ่ในโลกจริงเลือกข้ามไป เพราะ documentation ทำหน้าที่แทนได้ดีกว่าในทางปฏิบัติ

## แบบฝึกหัดท้ายบท

1. ออกแบบ URL structure (ไม่ต้องเขียนโค้ด) สำหรับระบบ "ร้านหนังสือออนไลน์" ที่มี resource: users, books, orders, reviews โดยยึดหลักการตั้งชื่อจากหัวข้อ 4 ให้ครบทุกจุด รวมถึง nested resource ที่เหมาะสม
2. เลือก status code ที่ถูกต้องสำหรับสถานการณ์ต่อไปนี้ พร้อมอธิบายเหตุผล: (ก) สมัครสมาชิกด้วย email ที่มีคนใช้แล้ว (ข) ลบ post ที่ตัวเองไม่ได้เป็นเจ้าของ (ค) แก้ไข profile ด้วย field `age` เป็น `-10`
3. เขียนฟังก์ชัน `parseOffsetPagination` เวอร์ชันของตัวเอง แต่เปลี่ยนพารามิเตอร์เป็น `page` และ `per_page` แทน `limit`/`offset` (เช่น `?page=3&per_page=20`) แล้วคำนวณ offset ภายในฟังก์ชันเอง
4. ทดลองสร้าง cursor-based pagination จริงกับ slice ของ struct (ไม่ต้องมีฐานข้อมูล) โดยให้ cursor เก็บ `created_at` แทนที่จะเป็น `id`
5. เขียน middleware ที่เช็ค header `Accept-Version` แล้ว route ไปยัง handler คนละตัวตามเวอร์ชัน จำลอง versioning แบบ header ให้ทำงานได้จริงด้วย `net/http` ล้วนๆ
6. ถกเถียงกับตัวเอง (เตรียมคำตอบไว้): ถ้าต้องออกแบบ API ใหม่ตั้งแต่ต้นสำหรับบริษัท จะเลือก URL versioning หรือ header versioning และทำไม ให้ยกตัวอย่างสถานการณ์ที่แต่ละแบบจะมีปัญหา

---

**ต่อไป**: [Part 060 — Request/Response และ JSON API](./060-request-response-json.md)
