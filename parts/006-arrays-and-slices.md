# Part 006: Arrays และ Slices เจาะลึก

> ภาคที่ 1: พื้นฐานภาษา Go (Fundamentals) — ตอนที่ 6 จาก 15

## สารบัญของบทนี้

1. Array คืออะไร: fixed-size value type
2. Array เป็น value type จริงจัง: copy semantics
3. Slice คืออะไร: descriptor เหนือ underlying array
4. สร้าง slice ด้วย `make` และความหมายของ `len`/`cap`
5. `append` และพฤติกรรมการขยาย capacity
6. Slicing Expression แบบเต็ม `a[low:high:max]`
7. Aliasing: กับดักการแชร์ underlying array โดยไม่รู้ตัว
8. `copy` — คัดลอกข้อมูลระหว่าง slice อย่างปลอดภัย
9. Multi-dimensional Slices
10. `nil` slice กับ empty slice: เหมือนแต่ไม่เหมือนกัน
11. Array เปรียบเทียบด้วย `==` ได้ ใช้เป็น map key ได้ (ที่ slice ทำไม่ได้)
12. ตารางเปรียบเทียบ Array vs Slice
13. สรุปสิ่งที่ได้เรียนในบทนี้
14. แบบฝึกหัดท้ายบท

---

## 1. Array คืออะไร: fixed-size value type

**Array** ใน Go คือโครงสร้างข้อมูลที่เก็บค่าหลายตัวชนิดเดียวกันเรียงต่อกัน โดยมี **ขนาดตายตัวที่กำหนดตอน compile time** และเป็นส่วนหนึ่งของ**ชนิดข้อมูล (type)** ของมันเองด้วย

```go
var numbers [5]int          // array ของ int ขนาด 5 ช่อง (zero value = [0 0 0 0 0])
grades := [3]float64{90.5, 85.0, 78.5}
```

จุดสำคัญที่ต้องเข้าใจตั้งแต่แรก: **`[5]int` และ `[3]int` ถือเป็นคนละชนิดข้อมูลกันโดยสิ้นเชิง** เหมือนกับที่ `int` กับ `int64` เป็นคนละชนิดกัน (ตามกฎ type conversion ใน Part 003)

```go
var a [5]int
var b [3]int
// a = b   // ❌ compile error: cannot use b (type [3]int) as type [5]int
```

Go ยังรองรับการให้ compiler นับขนาดให้อัตโนมัติจากจำนวน element ที่ใส่ด้วย `...`:

```go
weekdays := [...]string{"จันทร์", "อังคาร", "พุธ", "พฤหัสบดี", "ศุกร์"}
fmt.Println(len(weekdays)) // 5 — compiler นับให้เองจาก literal
```

การเข้าถึง/แก้ไขค่าทำผ่าน index (เริ่มที่ 0) เหมือนภาษาอื่นทั่วไป:

```go
grades[0] = 95.0
fmt.Println(grades[0])
```

---

## 2. Array เป็น value type จริงจัง: copy semantics

นี่คือความต่างที่สำคัญที่สุดระหว่าง Go กับหลายภาษา (เช่น Java ที่ array เป็น reference type เสมอ): **Array ใน Go เป็น value type แบบเต็มรูปแบบ** เมื่อ assign หรือส่งเป็น argument ให้ฟังก์ชัน **จะเป็นการคัดลอกข้อมูลทั้งก้อนเสมอ ไม่ใช่การอ้างอิง**

### การ assign คัดลอกทั้งก้อน

```go
original := [3]int{1, 2, 3}
copyArr := original      // คัดลอกข้อมูลทั้งหมด ไม่ใช่ pointer มาชี้ที่เดียวกัน
copyArr[0] = 999
fmt.Println("original:", original)
fmt.Println("copyArr:", copyArr)
```

ผลลัพธ์:

```
original: [1 2 3]
copyArr: [999 2 3]
```

การแก้ `copyArr` ไม่กระทบ `original` เลย เพราะมันเป็นข้อมูลคนละก้อนกันตั้งแต่บรรทัด `copyArr := original`

### การส่งเป็น argument ก็คัดลอกเช่นกัน

```go
func modifyArray(a [3]int) {
	a[0] = 100 // ไม่กระทบต้นฉบับ เพราะ a เป็นสำเนา
}

func main() {
	original := [3]int{1, 2, 3}
	modifyArray(original)
	fmt.Println("after modifyArray, original:", original)
}
```

ผลลัพธ์:

```
after modifyArray, original: [1 2 3]
```

ฟังก์ชัน `modifyArray` ได้รับ**สำเนา**ของ `original` ไปทั้งหมด การแก้ไขภายในฟังก์ชันจึงไม่มีผลกับตัวแปรต้นฉบับใน `main` เลย — ถ้าต้องการให้ฟังก์ชันแก้ไข array ต้นฉบับได้จริง ต้องส่ง **pointer** ไปแทน (จะเจาะลึกเรื่อง pointer เต็มรูปแบบใน Part 010):

```go
func modifyArrayPtr(a *[3]int) {
	a[0] = 100 // กระทบต้นฉบับ เพราะส่ง pointer เข้ามา ไม่ใช่สำเนา
}

func main() {
	original := [3]int{1, 2, 3}
	modifyArrayPtr(&original)
	fmt.Println("after modifyArrayPtr, original:", original)
}
```

ผลลัพธ์:

```
after modifyArrayPtr, original: [100 2 3]
```

> **ทำไมเรื่องนี้สำคัญ**: การส่ง array ขนาดใหญ่เป็น value เข้าฟังก์ชันบ่อยๆ เป็นการคัดลอกข้อมูลจำนวนมากซ้ำๆ ซึ่งกินทั้ง memory และเวลา นี่คือเหตุผลหลักที่ **ในทางปฏิบัติแทบไม่มีใครใช้ array ตรงๆ เป็น parameter ของฟังก์ชัน** แต่จะใช้ **slice** แทนแทบทั้งหมด ซึ่งเป็นหัวข้อหลักของบทนี้ที่จะพูดถึงต่อไป

---

## 3. Slice คืออะไร: descriptor เหนือ underlying array

**Slice** เป็นโครงสร้างข้อมูลที่ใช้บ่อยที่สุดใน Go สำหรับเก็บลำดับข้อมูล (sequence) — มันคือ "มุมมอง (view)" ที่ยืดหยุ่นเหนือ array ต้นฉบับตัวหนึ่ง (เรียกว่า **underlying array**)

ภายใน slice ประกอบด้วยข้อมูล 3 ส่วน (descriptor) ซึ่งเป็นแนวคิดที่**สำคัญที่สุดในบทนี้**:

```
slice = { pointer, length, capacity }
```

| ส่วนประกอบ | ความหมาย |
|---|---|
| **pointer** | ตำแหน่งของ element แรกที่ slice นี้ "มองเห็น" ใน underlying array |
| **length** (`len()`) | จำนวน element ที่ slice นี้เข้าถึงได้ในปัจจุบัน |
| **capacity** (`cap()`) | จำนวน element สูงสุดที่ slice นี้ **ขยายได้** โดยไม่ต้องสร้าง underlying array ใหม่ (นับจาก pointer ไปจนสุด underlying array เดิม) |

พูดให้เห็นภาพ: slice **ไม่ได้เก็บข้อมูลเอง** มันแค่ "ชี้ไปที่" ส่วนหนึ่งของ array ที่มีอยู่จริงที่อื่น การประกาศ slice literal ตรงๆ เช่น `[]int{1, 2, 3}` ก็คือให้ Go สร้าง underlying array ให้อัตโนมัติแล้วสร้าง slice มาชี้ทั้งก้อนให้

```go
s := []int{1, 2, 3} // ต่าง [3]int{1,2,3} ตรงที่ไม่มี [3] ระบุขนาด — นี่คือ slice ไม่ใช่ array
```

---

## 4. สร้าง slice ด้วย `make` และความหมายของ `len`/`cap`

วิธีมาตรฐานในการสร้าง slice ที่ยังไม่รู้ค่าเริ่มต้นแน่ชัดคือ builtin function `make`:

```go
s := make([]int, 3, 5)
fmt.Println("s:", s, "len:", len(s), "cap:", cap(s))
```

ผลลัพธ์:

```
s: [0 0 0] len: 3 cap: 5
```

`make([]int, length, capacity)` สร้าง underlying array ขนาด `capacity` (`5` ในที่นี้) และคืน slice ที่มองเห็นแค่ `length` ตัวแรก (`3` ตัว ค่าเป็น zero value คือ `0` ทั้งหมด) — ถ้าละ `capacity` ไม่ใส่ (`make([]int, 3)`) จะได้ `cap` เท่ากับ `len` พอดี

### slice ที่เกิดจากการ slicing array

เมื่อเรา slice ผ่าน array หรือ slice อื่นด้วย `[low:high]` จะได้ slice ใหม่ที่**ชี้ไปยัง underlying array เดียวกัน** ไม่ได้สร้างข้อมูลใหม่:

```go
arr := [5]int{10, 20, 30, 40, 50}
sl := arr[1:3]
fmt.Println("sl:", sl, "len:", len(sl), "cap:", cap(sl))

sl[0] = 999
fmt.Println("after modifying sl, arr:", arr)
```

ผลลัพธ์:

```
sl: [20 30] len: 2 cap: 4
after modifying sl, arr: [10 999 30 40 50]
```

`arr[1:3]` หมายถึง "เอา element ตั้งแต่ index 1 ถึงก่อน index 3" ได้ `[20, 30]` (index 1, 2) ส่วน **capacity เป็น 4** เพราะนับจาก index 1 ไปจนสุด array (index 1, 2, 3, 4 รวม 4 ช่อง) เมื่อแก้ `sl[0] = 999` จริงๆ แล้วคือการแก้ `arr[1]` โดยตรง (เพราะ `sl[0]` กับ `arr[1]` **คือช่องความจำเดียวกัน**) ผลลัพธ์ `arr` จึงกลายเป็น `[10 999 30 40 50]`

นี่คือหัวใจสำคัญของ slice: **การแก้ไขค่าใน slice จะกระทบ underlying array ต้นฉบับเสมอ** (และกระทบ slice อื่นที่ชี้มายัง array เดียวกันในช่วงที่ทับซ้อนกันด้วย) เรื่องนี้จะขยายความในหัวข้อที่ 7

---

## 5. `append` และพฤติกรรมการขยาย capacity

`append` คือฟังก์ชันหลักในการเพิ่มข้อมูลเข้า slice พฤติกรรมของมันขึ้นอยู่กับว่า **capacity ที่เหลืออยู่พอไหม**:

- **ถ้า `len(s) < cap(s)`**: `append` เขียนข้อมูลใหม่ต่อท้ายลงใน underlying array เดิมได้เลย (ไม่ต้องสร้าง array ใหม่) — slice ที่ได้ยังคงชี้ไปที่ array เดิม
- **ถ้า `len(s) == cap(s)` (capacity เต็มแล้ว)**: `append` ต้อง**สร้าง underlying array ใหม่ที่ใหญ่กว่า** คัดลอกข้อมูลเก่าไปทั้งหมด แล้วค่อยเพิ่มข้อมูลใหม่เข้าไป — slice ที่ได้จะชี้ไปที่ array ใหม่ ไม่ใช่ตัวเดิม

ทดลองดูรูปแบบการขยาย capacity จริงของ Go เวอร์ชันปัจจุบัน:

```go
s := make([]int, 0, 2)
prevCap := cap(s)
for i := 0; i < 10; i++ {
	s = append(s, i)
	if cap(s) != prevCap {
		fmt.Printf("len=%d cap changed %d -> %d\n", len(s), prevCap, cap(s))
		prevCap = cap(s)
	}
}
```

ผลลัพธ์:

```
len=3 cap changed 2 -> 4
len=5 cap changed 4 -> 8
len=9 cap changed 8 -> 16
```

จะเห็นว่า capacity ขยายแบบ**เพิ่มขึ้นเป็นเท่าตัวโดยประมาณ** (ไม่ใช่ทีละ 1) เพื่อลดจำนวนครั้งที่ต้องคัดลอก array ใหม่ทั้งก้อน — รายละเอียดตัวเลขที่แน่นอน (double เป๊ะๆ หรือ growth factor เท่าไหร่) **เป็นรายละเอียด internal ของ Go runtime ที่เปลี่ยนไปได้ในแต่ละเวอร์ชัน ห้ามเขียนโค้ดที่พึ่งพาตัวเลขที่แน่นอนนี้** สิ่งที่ควรจำมีแค่หลักการ: **append ขยายแบบ amortized (เฉลี่ยแล้วเร็ว) ไม่ใช่ทุกครั้งที่เรียกจะคัดลอกข้อมูลทั้งหมด**

### ผลกระทบสำคัญ: หลัง append ที่ทำให้เกิด array ใหม่ ความสัมพันธ์กับ slice เดิมจะขาดจากกัน

```go
original := make([]int, 3, 3) // cap เต็มพอดี ไม่เหลือที่ขยาย
original[0], original[1], original[2] = 1, 2, 3

grown := append(original, 4) // cap ไม่พอ (3 == 3) ต้องสร้าง array ใหม่
grown[0] = 999

fmt.Println("original after append-grow:", original)
fmt.Println("grown:", grown)
```

ผลลัพธ์:

```
original after append-grow: [1 2 3]
grown: [999 2 3 4]
```

เพราะ `cap(original)` เท่ากับ `len(original)` พอดี (`3 == 3`) การ `append` จึงต้องสร้าง underlying array ใหม่ให้ `grown` แยกออกจาก `original` โดยสิ้นเชิง การแก้ `grown[0]` จึงไม่กระทบ `original` เลย — **นี่ต่างจากตัวอย่างในหัวข้อที่ 4 ที่ capacity ยังเหลือ** ซึ่งการแก้ไขจะกระทบกันเพราะยังใช้ array เดียวกันอยู่

> **บทเรียนสำคัญที่สุดของหัวข้อนี้**: `append(s, ...)` **อาจจะคืน slice ที่ชี้ไปยัง underlying array เดียวกับ `s` เดิม หรือชี้ไปยัง array ใหม่ทั้งหมดก็ได้ ขึ้นอยู่กับ capacity ที่เหลือ ณ ตอนนั้น** ผู้เขียนโปรแกรมไม่สามารถรู้ล่วงหน้าได้แน่นอนโดยไม่เช็ค `cap()` เอง นี่คือเหตุผลว่าทำไมเราต้อง**เขียน `s = append(s, x)` เก็บผลลัพธ์กลับเข้าตัวแปรเดิมเสมอ** ไม่ใช่แค่เรียก `append(s, x)` เฉยๆ แล้วคาดหวังว่า `s` จะเปลี่ยนไปเอง (เพราะ Go เป็น pass-by-value แม้กับ slice header เอง — ถ้าไม่ assign กลับ ตัวแปร `s` ต้นฉบับจะยังมี length เท่าเดิม)

---

## 6. Slicing Expression แบบเต็ม `a[low:high:max]`

นอกจาก `a[low:high]` แบบพื้นฐาน (ได้ element ตั้งแต่ index `low` ถึงก่อน `high`) Go ยังมี slicing expression แบบ **3 ค่า** ที่ควบคุม capacity ของ slice ผลลัพธ์ได้ด้วย: `a[low:high:max]`

- `low` และ `high` กำหนด length เหมือนเดิม (`high - low`)
- `max` กำหนดขอบเขตบนสุดของ **capacity** (`max - low`) — จำกัดไม่ให้ slice ผลลัพธ์ขยายไปทับ element ที่อยู่เลย `max` ในการ `append` ครั้งถัดไป

```go
base := []int{1, 2, 3, 4, 5, 6, 7, 8}
limited := base[1:3:4]
fmt.Println("limited:", limited, "len:", len(limited), "cap:", cap(limited))
```

ผลลัพธ์:

```
limited: [2 3] len: 2 cap: 3
```

`base[1:3:4]` ได้ length `3-1=2` (element index 1,2 คือ `[2 3]`) และ capacity `4-1=3` (ไม่ใช่ `8-1=7` แบบที่จะได้ถ้าเขียนแค่ `base[1:3]`) ประโยชน์ของรูปแบบนี้คือ**การจำกัด capacity โดยตั้งใจ** เพื่อบังคับให้ `append` ครั้งถัดไปสร้าง underlying array ใหม่ทันทีที่ capacity เต็ม แทนที่จะไปเขียนทับ element ของ `base` ที่อยู่ถัดจาก index 4 โดยไม่ได้ตั้งใจ (ซึ่งเป็นกับดักการ aliasing ที่จะอธิบายในหัวข้อถัดไป) — เทคนิคนี้เรียกกันในวงการว่า **full slice expression** และมักใช้เมื่อต้องส่ง slice ย่อยให้โค้ดส่วนอื่นแล้วต้องการการันตีว่ามันจะไม่ไปกระทบข้อมูลถัดจากขอบเขตที่กำหนด

---

## 7. Aliasing: กับดักการแชร์ underlying array โดยไม่รู้ตัว

เพราะ slice หลายตัวสามารถชี้ไปยัง underlying array เดียวกันได้พร้อมกัน (ตามที่อธิบายในหัวข้อ 3-4) การแก้ไขผ่าน slice ตัวหนึ่งจึงอาจ**กระทบ slice อีกตัวที่ไม่เกี่ยวข้องโดยตรงในสายตาโปรแกรมเมอร์** นี่คือบั๊กที่พบบ่อยที่สุดอันดับต้นๆ เมื่อทำงานกับ slice ใน Go:

```go
base := []int{1, 2, 3, 4, 5}
a := base[0:2] // [1, 2] — ชี้ไปที่ base index 0,1
b := base[1:3] // [2, 3] — ชี้ไปที่ base index 1,2 (ทับซ้อนกับ a ที่ index 1!)

a[1] = 999 // a[1] คือ base[1] ตัวเดียวกับ b[0]
fmt.Println("base:", base)
fmt.Println("a:", a)
fmt.Println("b:", b)
```

ผลลัพธ์:

```
base: [1 999 3 4 5]
a: [1 999]
b: [999 3]
```

สังเกตว่า `a[1] = 999` ทำให้ `b[0]` เปลี่ยนไปด้วยทั้งที่โค้ดไม่ได้แตะ `b` เลยแม้แต่บรรทัดเดียว! เพราะ `a` และ `b` มีช่วงที่**ทับซ้อนกัน** (`base[1]`) ในสายตา underlying array มันคือช่องความจำเดียวกัน

### แนวทางป้องกันปัญหา aliasing

1. **ถ้าต้องการ slice ที่เป็นอิสระจากต้นฉบับจริงๆ** ให้สร้าง slice ใหม่ด้วย `make` แล้ว `copy` ข้อมูลออกมา (ดูหัวข้อถัดไป) แทนการ slice ตรงๆ
2. **ใช้ full slice expression (`a[low:high:max]`)** เพื่อจำกัด capacity ไม่ให้ `append` เผลอไปเขียนทับข้อมูลของ slice อื่นที่ใช้ underlying array เดียวกัน
3. **ระวังเป็นพิเศษเมื่อส่ง slice เป็น argument ให้ฟังก์ชันอื่น** เพราะฟังก์ชันนั้นแก้ไข element ผ่าน index ได้ (กระทบต้นฉบับ) แม้จะแก้ length ของ slice header เองไม่ได้ (เพราะ slice header เป็น value ที่ถูกคัดลอกตอนส่งเป็น argument — คัดลอกแค่ pointer/len/cap 3 ค่านี้ ไม่ใช่คัดลอกข้อมูลใน underlying array)

> เทียบกับ array ใน Part 002-005 ที่เราเน้นย้ำว่าเป็น value type คัดลอกทั้งก้อนเสมอ — slice **ไม่ใช่** value type แบบนั้น แม้ตัว slice header (`pointer, len, cap`) จะถูกคัดลอกเวลา assign/ส่ง argument ก็ตาม แต่**ข้อมูลที่มันชี้ไปนั้นเป็นก้อนเดียวกันกับต้นฉบับเสมอ** นี่คือความแตกต่างเชิงพฤติกรรมที่สำคัญที่สุดระหว่าง array กับ slice ที่ต้องจำให้แม่น

---

## 8. `copy` — คัดลอกข้อมูลระหว่าง slice อย่างปลอดภัย

เมื่อต้องการคัดลอก**ข้อมูลจริง** จาก slice หนึ่งไปอีก slice หนึ่ง (ไม่ใช่แค่แชร์ underlying array) ใช้ builtin function `copy`:

```go
src := []int{1, 2, 3, 4, 5}
dst := make([]int, 3)
n := copy(dst, src)
fmt.Println("copied", n, "elements:", dst)
```

ผลลัพธ์:

```
copied 3 elements: [1 2 3]
```

`copy(dst, src)` คัดลอกข้อมูลจาก `src` ไปยัง `dst` **ทีละ byte จริง** (ไม่ใช่แค่ชี้ pointer ไปที่เดิม) และคืนค่าจำนวน element ที่คัดลอกสำเร็จ ซึ่งเท่ากับ **ค่าที่น้อยกว่าระหว่าง `len(dst)` กับ `len(src)`** (ในตัวอย่างนี้ `dst` มี length แค่ 3 จึงคัดลอกได้แค่ 3 ตัวแรกของ `src` แม้ `src` จะมี 5 ตัวก็ตาม)

หลังจากนี้ `dst` เป็นอิสระจาก `src` อย่างสมบูรณ์ — แก้ไข `dst` จะไม่กระทบ `src` เลย เพราะเป็น underlying array คนละก้อนกัน (สร้างผ่าน `make` แยกต่างหาก) นี่คือวิธีมาตรฐานในการ "ตัดขาด" ความสัมพันธ์แบบ aliasing เมื่อต้องการ slice อิสระจริงๆ

---

## 9. Multi-dimensional Slices

Go ไม่มี array/slice หลายมิติแบบ built-in อย่างแท้จริงเหมือนบางภาษา (เช่น `int[,]` ใน C#) แต่ใช้วิธี **slice ของ slice** (slice ที่แต่ละ element เป็น slice อีกที) แทน ซึ่งยืดหยุ่นกว่าในหลายกรณี (แต่ละแถวมีความยาวต่างกันได้ ไม่ต้องเป็นสี่เหลี่ยมผืนผ้าเป๊ะ):

```go
grid := make([][]int, 3) // slice ของ 3 แถว แต่ละแถวยังไม่ได้สร้าง (nil slice)
for i := range grid {
	grid[i] = make([]int, 3) // สร้างแต่ละแถวแยกกัน ขนาด 3 คอลัมน์
	for j := range grid[i] {
		grid[i][j] = i*3 + j
	}
}

for _, row := range grid {
	fmt.Println(row)
}
```

ผลลัพธ์:

```
[0 1 2]
[3 4 5]
[6 7 8]
```

จุดสำคัญที่ต้องระวัง: `make([][]int, 3)` สร้างแค่ **slice ชั้นนอก** (มี 3 ช่องสำหรับเก็บ slice ย่อย) แต่**แต่ละแถวยังเป็น `nil` slice อยู่** ต้องวนสร้าง (`make([]int, ...)`) ให้แต่ละแถวเองเสมอ ก่อนจะเข้าถึง `grid[i][j]` ได้ — ถ้าลืมขั้นตอนนี้แล้วพยายามเขียน `grid[i][j] = ...` ทั้งที่ `grid[i]` ยัง `nil` อยู่ จะได้ runtime panic ทันที (`index out of range` เพราะ `nil` slice มี length เป็น 0)

รูปแบบ slice ของ slice นี้ยืดหยุ่นกว่า array 2 มิติตรงที่แต่ละแถวไม่จำเป็นต้องยาวเท่ากัน (เรียกว่า **jagged array/slice**) เหมาะกับข้อมูลที่ไม่ได้เป็นตารางสี่เหลี่ยมสมบูรณ์แบบ เช่น ข้อมูล adjacency list ของกราฟ ที่แต่ละ node มีจำนวนเพื่อนบ้านไม่เท่ากัน

---

## 10. `nil` slice กับ empty slice: เหมือนแต่ไม่เหมือนกัน

ตามที่กล่าวไปใน Part 003 ว่า zero value ของ slice คือ `nil` — แต่มีอีกกรณีที่ดูคล้ายกันมากจนสับสนได้ง่าย คือ **empty slice** (slice ที่มี length เป็น 0 แต่ไม่ใช่ `nil`) ทั้งสองแบบใช้งานทั่วไปเหมือนกัน (append ได้ปกติ, `len()` เป็น 0 เหมือนกัน) แต่ **ต่างกันตรงการเทียบกับ `nil` โดยตรง**:

```go
var nilSlice []int      // ประกาศด้วย var เฉยๆ ไม่กำหนดค่า -> nil slice
emptySlice := []int{}   // ประกาศด้วย literal ว่างเปล่า -> empty slice (ไม่ใช่ nil)

fmt.Println("nilSlice == nil:", nilSlice == nil)
fmt.Println("emptySlice == nil:", emptySlice == nil)
fmt.Println("len(nilSlice):", len(nilSlice), "len(emptySlice):", len(emptySlice))

nilSlice = append(nilSlice, 1)
fmt.Println("after append to nilSlice:", nilSlice)
```

ผลลัพธ์:

```
nilSlice == nil: true
emptySlice == nil: false
len(nilSlice): 0 len(emptySlice): 0
after append to nilSlice: [1]
```

ประเด็นสำคัญที่ต้องจำ:

1. **`append` ใช้กับ `nil` slice ได้โดยตรงอย่างปลอดภัย** ไม่จำเป็นต้อง `make` ก่อนเสมอไป — Go จะสร้าง underlying array ใหม่ให้เองตั้งแต่การ `append` ครั้งแรก นี่คือเหตุผลที่มักเห็นโค้ด Go ประกาศ `var results []int` เฉยๆ แล้วค่อย `append` เพิ่มทีหลังโดยไม่ error
2. `len()` และ `cap()` ใช้ได้กับ `nil` slice ปกติ (คืนค่า `0` ทั้งคู่) — **ไม่ panic** ต่างจากการพยายามเข้าถึง element ด้วย index (`nilSlice[0]`) ซึ่ง**จะ panic ทันที** เพราะไม่มี element ให้เข้าถึงเลย
3. ถ้าโค้ดต้อง**เช็คว่า slice "ไม่เคยถูกกำหนดค่าเลย" ต่างจาก "ถูกกำหนดเป็นค่าว่างโดยตั้งใจ"** (เช่น แยกแยะระหว่าง "ยังไม่ได้ query ข้อมูล" กับ "query แล้วไม่เจอผลลัพธ์") การเทียบกับ `nil` ตรงๆ (`if s == nil`) มีประโยชน์มาก แต่ถ้าแค่ต้องการเช็คว่า "มีข้อมูลอยู่หรือไม่" ให้เช็ค `len(s) == 0` แทน เพราะครอบคลุมทั้ง `nil` slice และ empty slice ในคำสั่งเดียว (แนวทางที่ทีม Go core แนะนำในทางปฏิบัติจริง)

---

## 11. Array เปรียบเทียบด้วย `==` ได้ ใช้เป็น map key ได้ (ที่ slice ทำไม่ได้)

ตามที่ระบุไว้ในตารางเปรียบเทียบท้ายบท (หัวข้อถัดไป) ว่า array เปรียบเทียบด้วย `==` ได้แต่ slice ไม่ได้ — คุณสมบัติ **comparable** นี้เองที่ทำให้ **array ใช้เป็น key ของ map ได้ แต่ slice ใช้ไม่ได้** (เรื่อง map เต็มรูปแบบจะเจาะลึกใน Part 007 แต่ประเด็นนี้เชื่อมโยงโดยตรงกับสิ่งที่เรียนในบทนี้จึงควรรู้ไว้ตั้งแต่ตอนนี้)

```go
type Point [2]int // array ขนาด 2 ใช้แทนพิกัด (x, y)

visited := map[Point]bool{}
visited[Point{1, 2}] = true

fmt.Println(visited[Point{1, 2}], visited[Point{3, 4}])
```

ผลลัพธ์:

```
true false
```

`Point{1, 2}` สองตัวที่แยกกันสร้าง (ตอน `visited[Point{1, 2}] = true` กับตอน query `visited[Point{1, 2}]`) ถือเป็น **ค่าเดียวกัน** ในสายตา map เพราะ array เปรียบเทียบกันด้วยการเทียบ**ค่าทุกช่องทีละช่อง** (value equality) — ถ้าลองเปลี่ยน `Point` จาก `[2]int` เป็น `[]int` (slice) โค้ดนี้จะ **compile ไม่ผ่านทันที** เพราะ slice ไม่ใช่ comparable type จึงใช้เป็น map key ไม่ได้ ข้อจำกัดข้อนี้เป็นหนึ่งในเหตุผลหลักที่บางครั้งเราเลือกใช้ array แทน slice ทั้งที่ปกติจะใช้ slice เป็น default (เช่น ใช้ `[2]int`, `[3]float64` แทนพิกัดที่ต้องใช้เป็น key เปรียบเทียบหรือใช้เป็น map key โดยตรง)

---

## 12. ตารางเปรียบเทียบ Array vs Slice

สรุปเปรียบเทียบทุกมิติที่เรียนมาในบทนี้:

| คุณสมบัติ | Array | Slice |
|---|---|---|
| ขนาด | ตายตัว เป็นส่วนหนึ่งของ type (`[5]int` ≠ `[3]int`) | ยืดหยุ่น ปรับขนาดได้ด้วย `append` |
| เก็บข้อมูลที่ไหน | เก็บข้อมูลเองโดยตรง | แค่ descriptor (pointer, len, cap) ชี้ไปยัง underlying array |
| การ assign / ส่งเป็น argument | **คัดลอกข้อมูลทั้งก้อน** (value semantics เต็มรูปแบบ) | คัดลอกแค่ descriptor 3 ค่า แต่**ข้อมูลจริงยังใช้ underlying array ร่วมกัน** |
| Zero value | array ที่ทุกช่องเป็น zero value ของ type นั้น (ใช้งานได้ทันที) | `nil` (length และ capacity เป็น 0 แต่ใช้กับ `append` ได้ปกติ) |
| เปรียบเทียบด้วย `==` | ได้ (ถ้า element type comparable) | **ไม่ได้** (ยกเว้นเทียบกับ `nil`) |
| ใช้บ่อยแค่ไหนในทางปฏิบัติ | น้อย (ใช้เมื่อรู้ขนาดตายตัวแน่นอนจริงๆ เช่น key ของ hash แบบ fixed size) | **บ่อยมาก** — เป็นโครงสร้างข้อมูลเชิงลำดับหลักที่ใช้ในโค้ด Go เกือบทั้งหมด |

> แนวทางปฏิบัติโดยรวม: **ใช้ slice เป็นค่า default เสมอเมื่อทำงานกับลำดับข้อมูล** สงวน array ไว้สำหรับกรณีพิเศษที่รู้ขนาดตายตัวจริงๆ และต้องการ value semantics (เช่น การใช้เป็น key ของ map ตามที่จะเห็นใน Part 007 เพราะ slice ใช้เป็น map key ไม่ได้แต่ array ใช้ได้)

---

## สรุปสิ่งที่ได้เรียนในบทนี้

- **Array** มีขนาดตายตัว เป็นส่วนหนึ่งของ type และเป็น **value type เต็มรูปแบบ** — assign/ส่ง argument คือคัดลอกข้อมูลทั้งก้อนเสมอ ต้องส่ง pointer ถ้าต้องการแก้ไขต้นฉบับจากในฟังก์ชัน
- **Slice** คือ descriptor 3 ส่วน (pointer, length, capacity) ที่ชี้ไปยัง underlying array ไม่ได้เก็บข้อมูลเอง
- `make([]T, len, cap)` สร้าง slice ใหม่พร้อม underlying array ขนาด `cap`; การ slicing (`a[low:high]`) จาก array/slice เดิมจะแชร์ underlying array เดียวกัน
- `append` เขียนข้อมูลต่อท้ายลง underlying array เดิมถ้า capacity ยังพอ แต่ถ้า capacity เต็มจะสร้าง array ใหม่ทั้งก้อน (growth แบบ amortized เพิ่มเป็นเท่าตัวโดยประมาณ) — ต้อง `s = append(s, x)` เก็บผลลัพธ์กลับเสมอ
- Full slice expression `a[low:high:max]` ควบคุม capacity ของ slice ผลลัพธ์ได้ ป้องกันการ append ไปทับข้อมูลของ slice อื่นที่ใช้ underlying array ร่วมกัน
- **Aliasing** คือกับดักสำคัญที่สุดของ slice: slice หลายตัวที่ชี้ไปยัง underlying array เดียวกันจะกระทบกันเมื่อแก้ไขผ่านช่วงที่ทับซ้อนกัน แม้ไม่ได้แก้ตัวแปรนั้นตรงๆ
- `copy(dst, src)` คัดลอกข้อมูลจริง (ไม่ใช่แค่ descriptor) ทำให้ slice ที่ได้เป็นอิสระจากต้นฉบับ คืนค่าจำนวน element ที่คัดลอกสำเร็จ (ค่าน้อยกว่าระหว่าง `len(dst)` กับ `len(src)`)
- Multi-dimensional data ใน Go ทำผ่าน **slice ของ slice** ต้องสร้างแต่ละแถวด้วย `make` แยกกันเอง (สร้างแค่ชั้นนอกไม่พอ)
- `nil` slice กับ empty slice มี `len()` เป็น 0 เหมือนกันและใช้ `append` ได้ปกติทั้งคู่ แต่เทียบกับ `nil` ได้ผลต่างกัน (`nilSlice == nil` เป็น `true`, `emptySlice == nil` เป็น `false`)
- Array เป็น comparable type (เทียบด้วย `==` ได้) และใช้เป็น **map key** ได้ ต่างจาก slice ที่ทำทั้งสองอย่างนี้ไม่ได้เลย
- Slice เป็นโครงสร้างข้อมูลเชิงลำดับหลักที่ใช้ในทางปฏิบัติเกือบทั้งหมด ส่วน array สงวนไว้สำหรับกรณีพิเศษที่ต้องการขนาดตายตัว, value semantics, หรือใช้เป็น map key

## แบบฝึกหัดท้ายบท

1. เขียนโปรแกรมสร้าง array ขนาด 5 กำหนดค่าเริ่มต้น แล้ว assign ให้ตัวแปรใหม่และแก้ไขตัวแปรใหม่นั้น พิสูจน์ด้วยการ print ว่าตัวต้นฉบับไม่เปลี่ยนแปลง (ตอกย้ำ copy semantics ของ array)
2. เขียนฟังก์ชันที่รับ slice ของ int แล้วคูณทุกค่าด้วย 2 **ผ่าน index โดยตรง (ไม่ใช้ append)** แล้วพิสูจน์ว่าการแก้ไขนี้กระทบ slice ต้นฉบับที่ส่งเข้ามาจาก `main`
3. เขียนโปรแกรมสร้าง slice ด้วย `make([]int, 0, 3)` แล้ว `append` ค่าไปเรื่อยๆ พร้อม print `len`/`cap` ทุกรอบ จนกระทั่ง capacity ขยายอย่างน้อย 2 ครั้ง สังเกตรูปแบบการขยายที่เกิดขึ้นจริงบนเครื่องของตัวเอง
4. จำลองปัญหา aliasing: สร้าง slice ต้นฉบับ 1 ตัว แล้ว slice ออกมาเป็น 2 ตัวที่ช่วง index ทับซ้อนกัน แก้ไขค่าผ่าน slice ตัวแรก แล้วพิสูจน์ว่า slice ตัวที่สองเปลี่ยนค่าตามไปด้วยทั้งที่ไม่ได้แตะมันตรงๆ
5. แก้ปัญหาจากข้อ 4 โดยใช้ `copy` เพื่อสร้าง slice ที่เป็นอิสระจากกันอย่างแท้จริง แล้วพิสูจน์ว่าการแก้ไขไม่กระทบกันอีกต่อไป
6. เขียนโปรแกรมสร้างตารางหมากรุก 8x8 ด้วย slice ของ slice (`[][]string`) กำหนดค่าเริ่มต้นเป็น `"."` ทุกช่อง แล้ววางตัวหมาก `"K"` (King) ไว้ที่ตำแหน่ง `[0][4]` จากนั้น print ตารางทั้งหมดออกมาทีละแถว
7. ประกาศ `type Coordinate [2]int` แล้วสร้าง `map[Coordinate]string` เก็บชื่อสถานที่ตามพิกัด ทดลอง query ด้วยพิกัดที่สร้างขึ้นใหม่ (คนละตัวแปรกับตอนใส่ข้อมูล) แล้วพิสูจน์ว่ายังหาเจอ เพราะ array เปรียบเทียบด้วยค่า ไม่ใช่ด้วย reference

---

**ต่อไป**: [Part 007 — Maps เจาะลึก](./007-maps.md)
