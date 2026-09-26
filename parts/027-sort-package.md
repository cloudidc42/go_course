# Part 027: การเรียงลำดับด้วยแพ็กเกจ `sort`

> ภาคที่ 2: ระดับกลาง (Intermediate) — ตอนที่ 12 จาก 20 (Part 16–35)

## สารบัญของบทนี้

1. แพ็กเกจ `sort` คืออะไร และทำไมต้องเรียนแยกบท
2. เรียง Built-in Type: `sort.Ints`, `sort.Strings`, `sort.Float64s`
3. ตรวจสอบว่าเรียงแล้วหรือยัง: `sort.IntsAreSorted` และญาติของมัน
4. `sort.Slice`/`sort.SliceStable`: วิธีสมัยใหม่ที่ใช้บ่อยที่สุด
5. เรียง Struct หลายเงื่อนไข (Multi-Field Sort)
6. `sort.Slice` vs `sort.SliceStable`: ความเสถียรของการเรียงลำดับ
7. วิธีคลาสสิก: Implement `sort.Interface` เอง (Len/Less/Swap)
8. เมื่อไรควรใช้ `sort.Interface` แทน `sort.Slice`
9. `sort.Reverse`: กลับด้านการเรียงโดยไม่ต้องเขียน Less ใหม่
10. `sort.Search`: Binary Search แบบทั่วไป
11. ความซับซ้อนเชิงเวลา และมองไปข้างหน้าสู่ package `slices` (Go 1.21+)
12. สรุปสิ่งที่ได้เรียนในบทนี้
13. แบบฝึกหัดท้ายบท

---

## 1. แพ็กเกจ `sort` คืออะไร และทำไมต้องเรียนแยกบท

การเรียงลำดับข้อมูล (sorting) เป็นหนึ่งในงานพื้นฐานที่สุดในโปรแกรมมิ่ง Go มี package มาตรฐานชื่อ `sort` ที่ให้ algorithm การเรียงลำดับที่มีประสิทธิภาพ (ใช้ [pattern-defeating quicksort](https://arxiv.org/abs/2106.05123) ภายใน ซึ่งรับประกันเวลาทำงานแบบ O(n log n) เสมอ) โดยไม่ต้องเขียน algorithm เรียงลำดับเองเลย

สิ่งที่ทำให้ `sort` น่าสนใจคือการออกแบบ API ของมันสะท้อนวิวัฒนาการของภาษา Go เอง — API ยุคแรก (ก่อน Go 1.8) ใช้วิธี implement interface `sort.Interface` ซึ่งเขียนยาวแต่ยืดหยุ่นสูง ส่วน API ยุคหลัง (`sort.Slice`, Go 1.8 เป็นต้นมา) ใช้ **closure** (ทบทวนจาก **Part 009**) ทำให้เขียนโค้ดสั้นกระชับกว่ามาก บทนี้จะพาไปรู้จักทั้งสองแนวทาง และบอกว่าแนวทางไหนควรใช้เมื่อไร

---

## 2. เรียง Built-in Type: `sort.Ints`, `sort.Strings`, `sort.Float64s`

สำหรับ slice ของ type พื้นฐานที่พบบ่อยที่สุด (`int`, `string`, `float64`) `sort` มีฟังก์ชันสำเร็จรูปให้ใช้ตรงๆ โดยเรียงจากน้อยไปมาก (ascending) เสมอ และแก้ไข slice เดิม**ในที่เดิม (in-place)** โดยไม่คืนค่าใหม่:

```go
package main

import (
	"fmt"
	"sort"
)

func main() {
	nums := []int{5, 2, 8, 1, 9, 3}
	sort.Ints(nums)
	fmt.Println("sorted ints:", nums)

	words := []string{"banana", "apple", "cherry"}
	sort.Strings(words)
	fmt.Println("sorted strings:", words)

	floats := []float64{3.14, 1.41, 2.71}
	sort.Float64s(floats)
	fmt.Println("sorted floats:", floats)
}
```

ผลลัพธ์:

```
sorted ints: [1 2 3 5 8 9]
sorted strings: [apple banana cherry]
sorted floats: [1.41 2.71 3.14]
```

**จุดสำคัญที่ต้องจำ**: ฟังก์ชันเหล่านี้แก้ไข slice ต้นฉบับโดยตรง (เพราะ slice ใน Go อ้างอิงถึง underlying array เดียวกัน ทบทวนจาก **Part 006**) ไม่ได้คืนค่า slice ใหม่ ดังนั้นการเขียน `nums = sort.Ints(nums)` จะ **compile error** เพราะ `sort.Ints` ไม่มีค่าคืนเลย (คืนแค่ `void`)

`sort.Strings` เรียงตาม**ลำดับ byte ของ UTF-8** ไม่ใช่ตามหลักภาษาศาสตร์หรือ locale ใดๆ ซึ่งสำหรับข้อความภาษาไทยอาจไม่ตรงกับลำดับพจนานุกรมที่คนไทยคุ้นเคย (เรื่องนี้เกี่ยวข้องกับ `golang.org/x/text/collate` ซึ่งอยู่นอกขอบเขตหลักสูตรพื้นฐาน)

---

## 3. ตรวจสอบว่าเรียงแล้วหรือยัง: `sort.IntsAreSorted` และญาติของมัน

ก่อนจะเสียเวลาเรียงข้อมูลที่อาจเรียงอยู่แล้ว หรือเพื่อ validate ผลลัพธ์หลังการประมวลผลบางอย่าง `sort` มีฟังก์ชันตรวจสอบให้พร้อมสำหรับ type พื้นฐานทั้งสามคู่กับฟังก์ชันเรียงด้านบน:

```go
package main

import (
	"fmt"
	"sort"
)

func main() {
	nums := []int{1, 2, 3, 5, 8, 9}
	fmt.Println("is sorted:", sort.IntsAreSorted(nums))

	unsorted := []int{3, 1, 4, 1, 5}
	fmt.Println("is sorted:", sort.IntsAreSorted(unsorted))
}
```

ผลลัพธ์:

```
is sorted: true
is sorted: false
```

มี `sort.StringsAreSorted` และ `sort.Float64sAreSorted` คู่กันเช่นกัน สำหรับ slice ที่เรียงด้วยเงื่อนไขกำหนดเอง (จาก `sort.Slice`) จะใช้ `sort.SliceIsSorted` แทน ซึ่งรับ closure เดียวกับที่ใช้ตอนเรียง

---

## 4. `sort.Slice`/`sort.SliceStable`: วิธีสมัยใหม่ที่ใช้บ่อยที่สุด

ในทางปฏิบัติ เรามักต้องเรียง slice ของ struct ที่ไม่ใช่ type พื้นฐาน และต้องการเงื่อนไขการเรียงเอง (เช่น เรียงคนตามอายุ) ตั้งแต่ Go 1.8 เป็นต้นมา วิธีที่นิยมที่สุดคือ `sort.Slice` ซึ่งรับ **closure** เป็น "less function" แทนที่จะต้อง implement interface แบบเต็มรูปแบบ:

```go
package main

import (
	"fmt"
	"sort"
)

type Person struct {
	Name string
	Age  int
}

func main() {
	people := []Person{
		{"สมชาย", 35},
		{"สมหญิง", 28},
		{"วิชัย", 42},
	}

	sort.Slice(people, func(i, j int) bool {
		return people[i].Age < people[j].Age
	})
	fmt.Println("sorted by age:", people)
}
```

ผลลัพธ์:

```
sorted by age: [{สมหญิง 28} {สมชาย 35} {วิชัย 42}]
```

`sort.Slice` รับ argument สองตัว:

1. `slice any` — slice ที่จะเรียง (ต้องเป็น slice เท่านั้น ไม่ใช่ array)
2. `less func(i, j int) bool` — closure ที่รับ index สองตัว และคืนค่า `true` ถ้า element ที่ index `i` ควรมาก่อน index `j`

เพราะ `less` เป็น closure จึงสามารถ**เข้าถึงตัวแปร `people` จาก scope ภายนอกได้โดยตรง** (ทบทวนหลักการ closure จาก **Part 009**) ทำให้เขียนเงื่อนไขการเรียงแบบกำหนดเองได้อย่างยืดหยุ่นโดยไม่ต้องประกาศ type หรือ method ใดๆ เพิ่มเลย — สั้นกระชับกว่าวิธีคลาสสิกใน หัวข้อที่ 7 มาก

---

## 5. เรียง Struct หลายเงื่อนไข (Multi-Field Sort)

ในสถานการณ์จริง มักต้องเรียงตามหลายเงื่อนไขพร้อมกัน เช่น เรียงตามอายุก่อน ถ้าอายุเท่ากันให้เรียงตามชื่อ ทำได้ง่ายๆ ด้วยการเขียน `if` ต่อกันใน closure:

```go
package main

import (
	"fmt"
	"sort"
)

type Person struct {
	Name string
	Age  int
}

func main() {
	people := []Person{
		{"สมชาย", 35},
		{"สมหญิง", 28},
		{"วิชัย", 42},
		{"มานี", 28},
	}

	sort.Slice(people, func(i, j int) bool {
		if people[i].Age != people[j].Age {
			return people[i].Age < people[j].Age
		}
		return people[i].Name < people[j].Name
	})
	fmt.Println("sorted by age then name:", people)
}
```

ผลลัพธ์:

```
sorted by age then name: [{มานี 28} {สมหญิง 28} {สมชาย 35} {วิชัย 42}]
```

หลักการเขียน multi-field less function คือ: **เช็คเงื่อนไขแรกก่อน ถ้าค่าต่างกันให้ return ผลเปรียบเทียบทันที ถ้าเท่ากันค่อยไปเช็คเงื่อนไขถัดไป** ทำแบบนี้ต่อกันไปเรื่อยๆ ได้ตามจำนวนเงื่อนไขที่ต้องการ

---

## 6. `sort.Slice` vs `sort.SliceStable`: ความเสถียรของการเรียงลำดับ

**Stability** ของ sorting algorithm หมายถึง: ถ้า element สองตัวมีค่าที่ใช้เปรียบเทียบ**เท่ากัน** algorithm ที่ stable จะ**รักษาลำดับเดิม**ของสอง element นั้นไว้ ในขณะที่ algorithm ที่ไม่ stable อาจสลับลำดับกันได้

`sort.Slice` **ไม่รับประกันความเสถียร** (ใช้ quicksort-based algorithm ที่เร็วกว่าแต่ไม่ stable) ส่วน `sort.SliceStable` **รับประกันความเสถียร** เสมอ (ใช้ algorithm แบบ stable ซึ่งช้ากว่าเล็กน้อย เพราะต้องทำงานเพิ่มเพื่อรักษาลำดับเดิม)

```go
package main

import (
	"fmt"
	"sort"
)

type Person struct {
	Name string
	Age  int
}

func main() {
	people := []Person{
		{"สมชาย", 35},
		{"สมหญิง", 28},
		{"วิชัย", 42},
		{"มานี", 28},
	}

	sort.SliceStable(people, func(i, j int) bool {
		return people[i].Age < people[j].Age
	})
	fmt.Println("stable sorted by age:", people)
}
```

ผลลัพธ์:

```
stable sorted by age: [{สมหญิง 28} {มานี 28} {สมชาย 35} {วิชัย 42}]
```

สังเกตว่า `สมหญิง` (อายุ 28) ยังคงอยู่**ก่อน** `มานี` (อายุ 28 เหมือนกัน) เพราะ `สมหญิง` มาก่อนใน slice ต้นฉบับ — นี่คือความหมายของความเสถียร

**หลักการเลือกใช้**: ถ้าเรียงตามเงื่อนไขเดียวที่รับประกันว่าไม่มีค่าซ้ำกัน (เช่น ID ที่ unique) ใช้ `sort.Slice` ได้เลยเพราะเร็วกว่า แต่ถ้าเรียงตามเงื่อนไขที่อาจมีค่าซ้ำกันได้ (เช่นอายุ, เกรด) และ**ลำดับเดิมของข้อมูลที่ค่าเท่ากันมีความหมาย** (เช่น ต้องการรักษาลำดับที่ผู้ใช้ป้อนเข้ามา) ให้ใช้ `sort.SliceStable` เสมอ

---

## 7. วิธีคลาสสิก: Implement `sort.Interface` เอง (Len/Less/Swap)

ก่อน Go 1.8 (และยังใช้ได้อยู่ในปัจจุบัน) การเรียง slice ของ type กำหนดเองทำผ่านการ implement interface `sort.Interface` ที่มีสาม method (ทบทวนหลักการ interface satisfaction จาก **Part 013**):

```go
type Interface interface {
	Len() int
	Less(i, j int) bool
	Swap(i, j int)
}
```

```go
package main

import (
	"fmt"
	"sort"
)

type Employee struct {
	Name   string
	Salary int
}

// BySalary implement sort.Interface เพื่อเรียง []Employee ตาม Salary
type BySalary []Employee

func (s BySalary) Len() int           { return len(s) }
func (s BySalary) Less(i, j int) bool { return s[i].Salary < s[j].Salary }
func (s BySalary) Swap(i, j int)      { s[i], s[j] = s[j], s[i] }

func main() {
	employees := []Employee{
		{"สมชาย", 35000},
		{"สมหญิง", 42000},
		{"วิชัย", 28000},
	}

	sort.Sort(BySalary(employees))
	fmt.Println("sorted by salary:", employees)
}
```

ผลลัพธ์:

```
sorted by salary: [{วิชัย 28000} {สมชาย 35000} {สมหญิง 42000}]
```

ขั้นตอนการทำงาน:

1. ประกาศ **named type ใหม่** ที่มี underlying type เป็น `[]Employee` (ทบทวนแนวคิด named type จาก **Part 011**) ชื่อ `BySalary`
2. Implement method `Len`, `Less`, `Swap` บน `BySalary` ตามที่ `sort.Interface` กำหนด
3. แปลง (convert) slice ต้นฉบับให้เป็น type `BySalary` ตอนเรียก `sort.Sort(BySalary(employees))` — การ convert นี้ไม่เสีย cost อะไรเพราะ `BySalary` มี underlying type เดียวกับ `[]Employee`

---

## 8. เมื่อไรควรใช้ `sort.Interface` แทน `sort.Slice`

ในโค้ด production สมัยใหม่ **`sort.Slice` เป็นตัวเลือกแรกเสมอ** เพราะสั้นกว่า อ่านง่ายกว่า และไม่ต้องประกาศ type เพิ่ม แต่ยังมีบางสถานการณ์ที่ `sort.Interface` เหมาะสมกว่า:

| สถานการณ์ | เหตุผลที่เลือก `sort.Interface` |
|---|---|
| ต้องเรียงด้วยเงื่อนไขเดียวกันซ้ำๆ หลายที่ในโปรเจกต์ | ประกาศ `BySalary` เป็น named type ครั้งเดียว ใช้ `sort.Sort(BySalary(x))` ได้ทุกที่ ไม่ต้องเขียน closure ซ้ำ |
| ต้องการให้ type ของเราเอง**เป็น** `sort.Interface` เพื่อส่งต่อให้ฟังก์ชันอื่นที่รับ `sort.Interface` โดยตรง | เช่นฟังก์ชันใน library ภายนอกที่ออกแบบให้รับ `sort.Interface` เป็น parameter |
| ต้องการเอกสาร (documentation) ที่ชัดเจนว่าเงื่อนไขการเรียงคืออะไร ผ่านชื่อ type | ชื่อ `BySalary` สื่อความหมายได้ทันทีเมื่ออ่านโค้ด ต่างจาก closure ที่ต้องอ่าน body ถึงจะรู้เงื่อนไข |
| Go เวอร์ชันเก่ามากที่ยังไม่มี `sort.Slice` (ก่อน 1.8) | ปัจจุบันแทบไม่พบในทางปฏิบัติแล้ว เพราะ Go 1.8 ออกมาตั้งแต่ปี 2017 |

สรุปสั้นๆ: **เขียน closure ครั้งเดียวใช้ที่เดียว → `sort.Slice`, เขียนเงื่อนไขที่นำกลับมาใช้ซ้ำได้หลายที่ หรือมีความหมายที่ควรมีชื่อ type ชัดเจน → `sort.Interface`**

---

## 9. `sort.Reverse`: กลับด้านการเรียงโดยไม่ต้องเขียน Less ใหม่

หากมี `sort.Interface` อยู่แล้ว (ไม่ว่าจะเขียนเองหรือได้จากที่อื่น) การเรียงย้อนกลับ (descending) ทำได้ง่ายๆ ด้วย `sort.Reverse` โดยไม่ต้องเขียน `Less` ใหม่:

```go
sort.Sort(sort.Reverse(BySalary(employees)))
fmt.Println("sorted by salary desc:", employees)
```

ผลลัพธ์:

```
sorted by salary desc: [{สมหญิง 42000} {สมชาย 35000} {วิชัย 28000}]
```

`sort.Reverse` ทำงานโดยห่อ (wrap) `sort.Interface` เดิมไว้ในอีก struct หนึ่งที่**สลับผลลัพธ์ของ `Less`** (คือเรียก `Less(j, i)` แทน `Less(i, j)`) — เป็นตัวอย่างที่ดีของการใช้ **composition** ผ่าน interface ที่จะเรียนเจาะลึกใน **Part 030 (Embedding)**

สำหรับ `sort.Slice` ถ้าต้องการเรียงย้อนกลับ วิธีที่ง่ายที่สุดคือสลับเครื่องหมายเปรียบเทียบใน closure ตรงๆ (`>` แทน `<`) โดยไม่ต้องใช้ `sort.Reverse` เลย เพราะ `sort.Slice` ไม่ได้คืนค่าเป็น `sort.Interface` ที่จะห่อได้โดยตรง

---

## 10. `sort.Search`: Binary Search แบบทั่วไป

เมื่อ slice **เรียงลำดับอยู่แล้ว** การค้นหาค่าด้วย binary search จะเร็วกว่าการวนลูปเช็คทีละตัว (linear search) มาก — จาก O(n) เหลือ O(log n) `sort` มีฟังก์ชันสำเร็จรูปสำหรับ type พื้นฐาน (`sort.SearchInts`, `sort.SearchStrings`, `sort.SearchFloat64s`) และฟังก์ชันทั่วไปที่สุดคือ `sort.Search`:

```go
func Search(n int, f func(int) bool) int
```

`sort.Search` หา**ตำแหน่งแรกสุด** (index น้อยที่สุด) ที่ทำให้ `f(index)` คืนค่า `true` โดยสมมติว่า slice เรียงลำดับในลักษณะที่ `f` ให้ผล `false, false, ..., false, true, true, ..., true` เรียงต่อกัน (เรียกว่า monotonic)

```go
package main

import (
	"fmt"
	"sort"
)

func main() {
	nums := []int{1, 3, 6, 10, 15, 21, 28, 36}

	target := 15
	idx := sort.Search(len(nums), func(i int) bool {
		return nums[i] >= target
	})
	if idx < len(nums) && nums[idx] == target {
		fmt.Printf("พบค่า %d ที่ index %d\n", target, idx)
	} else {
		fmt.Printf("ไม่พบค่า %d (ตำแหน่งที่ควรแทรกคือ %d)\n", target, idx)
	}

	notFound := 16
	idx2 := sort.Search(len(nums), func(i int) bool {
		return nums[i] >= notFound
	})
	fmt.Printf("ตำแหน่งที่ควรแทรก %d คือ index %d\n", notFound, idx2)
}
```

ผลลัพธ์:

```
พบค่า 15 ที่ index 4
ตำแหน่งที่ควรแทรก 16 คือ index 5
```

รูปแบบการใช้งานมาตรฐานของ `sort.Search` คือ: หา index แรกที่ `nums[i] >= target` แล้ว**เช็คซ้ำอีกครั้ง**ว่า `nums[idx] == target` จริงหรือไม่ เพราะ `sort.Search` แค่บอกตำแหน่งที่เงื่อนไขเริ่มเป็นจริง ไม่ได้การันตีว่าเจอค่าที่ตรงกันเป๊ะ ถ้าไม่เจอเลย ค่าที่คืนมาจะเป็น**ตำแหน่งที่ควรแทรกค่านั้นเข้าไป**เพื่อให้ slice ยังคงเรียงลำดับอยู่ (ในตัวอย่างข้างบน `16` ไม่มีใน slice แต่ตำแหน่งที่ควรแทรกคือ index 5 ซึ่งอยู่ระหว่าง `15` กับ `21`)

**ข้อควรระวังสำคัญ**: `sort.Search` (และฟังก์ชัน `sort.SearchXxx` ทั้งหมด) **ทำงานถูกต้องก็ต่อเมื่อ slice เรียงลำดับอยู่แล้วเท่านั้น** ถ้าเรียกกับ slice ที่ยังไม่เรียง ผลลัพธ์จะไม่มีความหมายใดๆ (undefined behavior ในเชิง logic ไม่ใช่ compile error หรือ panic)

---

## 11. ความซับซ้อนเชิงเวลา และมองไปข้างหน้าสู่ package `slices` (Go 1.21+)

algorithm ที่ `sort` ใช้ภายในเป็นแบบ **pdqsort (pattern-defeating quicksort)** ซึ่งรับประกันความซับซ้อนเชิงเวลา (time complexity) แบบ **O(n log n)** ในกรณีเลวร้ายที่สุดเสมอ (ต่างจาก quicksort ดั้งเดิมที่อาจแย่ลงถึง O(n²) ในกรณีเลวร้าย) ส่วน `sort.Search` (binary search) มีความซับซ้อน **O(log n)** ตามหลักการ binary search ทั่วไป — ตัวเลขเหล่านี้ไม่จำเป็นต้องท่องจำ แต่ควรเข้าใจว่า**การเรียงข้อมูลมี cost เสมอ** ดังนั้นถ้าต้อง sort ข้อมูลเดิมซ้ำๆ หลายครั้งโดยข้อมูลไม่เปลี่ยน ควร sort ครั้งเดียวแล้วเก็บผลลัพธ์ไว้ใช้ซ้ำ แทนที่จะเรียก `sort.Slice` ใหม่ทุกครั้งที่ต้องการอ่านข้อมูลแบบเรียงลำดับ

อีกประเด็นที่ควรรู้ไว้สำหรับ Go เวอร์ชันใหม่: ตั้งแต่ **Go 1.21** เป็นต้นมา standard library มี package ใหม่ชื่อ **`slices`** ที่นำแนวคิด **generics** (ซึ่งเราจะเรียนเจาะลึกใน **Part 028** และ **Part 029** ถัดไป) มาเขียนฟังก์ชันจัดการ slice ที่ทำงานคล้ายกับ `sort` แต่สั้นกระชับกว่า เพราะไม่ต้องระบุ type ต่อท้ายชื่อฟังก์ชัน (`sort.Ints` vs `slices.Sort`) และรองรับทุก type ที่ `cmp.Ordered` ครอบคลุมโดยอัตโนมัติ ไม่ต้องมีฟังก์ชันแยกสำหรับแต่ละ type แบบ `sort.Ints`/`sort.Strings`/`sort.Float64s`:

```go
package main

import (
	"fmt"
	"slices"
)

func main() {
	nums := []int{5, 2, 8, 1, 9}
	slices.Sort(nums)
	fmt.Println(nums)
	fmt.Println(slices.IsSorted(nums))
	fmt.Println(slices.Contains(nums, 8))
}
```

ผลลัพธ์:

```
[1 2 5 8 9]
true
true
```

`slices.Sort` ทำงานแบบเดียวกับ `sort.Ints`/`sort.Strings` ทุกประการ (แก้ไข slice ในที่เดิม, เรียงจากน้อยไปมาก) เพียงแต่ใช้ได้กับทุก type ที่ `cmp.Ordered` ครอบคลุมโดยไม่ต้องมีฟังก์ชันแยกต่อ type — **หลักสูตรนี้ยังคงสอน package `sort` เป็นหลักในบทนี้** เพราะ (1) โค้ด production จำนวนมหาศาลที่มีอยู่แล้วก่อน Go 1.21 ยังคงใช้ `sort` และ Go Developer มืออาชีพต้องอ่านโค้ดเก่าเหล่านี้ได้ (2) `sort.Slice`/`sort.SliceStable` ยังคงจำเป็นเสมอเมื่อต้องเรียงด้วยเงื่อนไขกำหนดเอง (custom less function) ซึ่ง `slices` package ก็มี `slices.SortFunc` ให้ใช้ในรูปแบบที่คล้ายกันมาก และ (3) การเข้าใจ `sort.Interface` แบบคลาสสิกช่วยให้เข้าใจหลักการออกแบบ API ของ Go ในภาพรวมได้ลึกซึ้งขึ้น ก่อนจะไปเรียน generics อย่างเป็นทางการใน **Part 028** ถัดไป

---

## สรุปสิ่งที่ได้เรียนในบทนี้

- `sort.Ints`/`sort.Strings`/`sort.Float64s` เรียง slice ของ type พื้นฐานแบบ in-place จากน้อยไปมาก, มีคู่ `sort.XxxAreSorted` ไว้ตรวจสอบ
- `sort.Slice`/`sort.SliceStable` คือวิธีสมัยใหม่ที่ใช้บ่อยที่สุด — รับ closure เป็น less function ทำให้เขียนเงื่อนไขการเรียงกำหนดเอง (รวมถึงหลายเงื่อนไข) ได้อย่างกระชับ
- `sort.Slice` ไม่รับประกันความเสถียร (stability) ส่วน `sort.SliceStable` รับประกันเสมอแต่ช้ากว่าเล็กน้อย — เลือกใช้ตามว่าลำดับเดิมของค่าที่เท่ากันมีความหมายหรือไม่
- วิธีคลาสสิกคือ implement `sort.Interface` (`Len`/`Less`/`Swap`) บน named type แล้วเรียก `sort.Sort(...)` — เหมาะเมื่อต้องใช้เงื่อนไขเดียวกันซ้ำหลายที่ หรือต้องส่ง `sort.Interface` ให้ฟังก์ชันอื่น
- `sort.Reverse` ห่อ `sort.Interface` เดิมเพื่อกลับด้านการเรียงโดยไม่ต้องเขียน `Less` ใหม่
- `sort.Search` ทำ binary search แบบทั่วไปด้วย closure บน slice ที่เรียงลำดับแล้วเท่านั้น ให้ผลลัพธ์เป็น O(log n)

## แบบฝึกหัดท้ายบท

1. เขียนโปรแกรมที่มี slice ของ `int` แบบไม่เรียงลำดับ ใช้ `sort.Ints` เรียง แล้วใช้ `sort.IntsAreSorted` ตรวจสอบผลลัพธ์
2. เขียน struct `Product` ที่มี `Name string` และ `Price float64` สร้าง slice ของ `Product` หลายตัว แล้วใช้ `sort.Slice` เรียงจากราคาน้อยไปมาก และเรียงจากราคามากไปน้อย (สองแบบแยกกัน)
3. เขียน struct `Student` ที่มี `Grade int` และ `Name string` สร้าง slice ที่มีนักเรียนเกรดซ้ำกันหลายคน แล้วเรียงด้วย `sort.SliceStable` ตามเกรดจากมากไปน้อย และสังเกตว่าลำดับของนักเรียนเกรดเท่ากันยังคงเดิมหรือไม่
4. Implement `sort.Interface` (`Len`/`Less`/`Swap`) บน named type `ByName []Student` (เรียงตามชื่อ a-z) แล้วใช้ `sort.Sort` และ `sort.Sort(sort.Reverse(...))` ทดสอบทั้งสองทิศทาง
5. เขียนฟังก์ชันที่รับ slice ของ `int` ที่เรียงลำดับแล้ว กับค่าที่ต้องการค้นหา แล้วใช้ `sort.Search` คืนค่า index ที่พบ หรือ `-1` ถ้าไม่พบ
6. เขียน struct `Task` ที่มี `Priority int` และ `CreatedAt time.Time` (ทบทวนจาก **Part 022**) แล้วเรียงด้วยเงื่อนไขสองชั้น: priority มากไปน้อยก่อน ถ้า priority เท่ากันให้เรียงตามเวลาสร้างจากเก่าไปใหม่

---

**ต่อไป**: [Part 028 — Generics พื้นฐาน (Go 1.18+)](./028-generics-basics.md)
