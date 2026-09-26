# Part 029: Generics ขั้นสูง: constraints, generic data structures

> ภาคที่ 2: ระดับกลาง (Intermediate) — ตอนที่ 14 จาก 20 (Part 16–35)

## สารบัญของบทนี้

1. ทบทวน: ทำไมต้องนิยาม Constraint เอง
2. นิยาม Constraint แบบ Union of Types: `type Number interface { ... }`
3. `~T`: Approximation Element สำหรับ Named Type
4. Package `golang.org/x/exp/constraints` และ `cmp.Ordered` ใน Standard Library
5. Generic Data Structure: `Set[T comparable]`
6. Generic Data Structure: Linked List
7. ข้อจำกัด: Method รับ Type Parameter ของตัวเองไม่ได้
8. Generics vs Interfaces: เมื่อไรควรใช้อันไหน
9. ข้อพิจารณาด้าน Performance: Monomorphization vs Interface Boxing
10. สรุปสิ่งที่ได้เรียนในบทนี้
11. แบบฝึกหัดท้ายบท

---

## 1. ทบทวน: ทำไมต้องนิยาม Constraint เอง

ใน **Part 028** เราเห็นแล้วว่า constraint สำเร็จรูปอย่าง `any` และ `comparable` ไม่พอสำหรับทุกสถานการณ์ — ถ้าต้องการเขียนฟังก์ชันที่ใช้ operator เปรียบเทียบขนาด (`<`, `>`) หรือ operator คำนวณ (`+`, `-`) ต้องมี constraint ที่รับประกันว่า type parameter รองรับ operator นั้นๆ

Go แก้ปัญหานี้ด้วยการให้เรา**นิยาม constraint interface เอง** โดยใช้ syntax แบบ **union of types** ที่เกริ่นไว้แล้วในบทก่อน บทนี้จะพาไปเจาะลึกเรื่องนี้ พร้อมสร้าง generic data structure ที่ใช้ได้จริง และปิดท้ายด้วยข้อจำกัดสำคัญของ generics ที่ต้องรู้ก่อนใช้งานจริงในระดับ production

---

## 2. นิยาม Constraint แบบ Union of Types: `type Number interface { ... }`

Constraint คือ **interface ชนิดพิเศษ** ที่นอกจากจะระบุ method ได้เหมือน interface ปกติ (ทบทวนจาก **Part 013**) ยังระบุ**รายการ type ที่ยอมรับได้**โดยใช้เครื่องหมาย `|` คั่นระหว่างแต่ละ type:

```go
package main

import "fmt"

// นิยาม constraint เอง: Number คือ union ของ type ตัวเลขที่ยอมให้ใช้ operator +, <, > ได้
type Number interface {
	int | int32 | int64 | float32 | float64
}

func Sum[T Number](values []T) T {
	var total T
	for _, v := range values {
		total += v
	}
	return total
}

func Max[T Number](values []T) T {
	max := values[0]
	for _, v := range values[1:] {
		if v > max {
			max = v
		}
	}
	return max
}

func main() {
	fmt.Println(Sum([]int{1, 2, 3, 4}))
	fmt.Println(Sum([]float64{1.5, 2.5, 3.0}))
	fmt.Println(Max([]int{5, 9, 2, 7}))
}
```

ผลลัพธ์:

```
10
7
9
```

`Number` เป็น interface ที่**ไม่มี method ใดๆ เลย** มีแต่รายการ type คั่นด้วย `|` — เมื่อใช้เป็น constraint (`[T Number]`) หมายความว่า `T` จะต้องเป็น**หนึ่งใน type ที่ระบุไว้เท่านั้น** (`int`, `int32`, `int64`, `float32`, หรือ `float64`) และเพราะทุก type ในรายการรองรับ operator `+` และ `>` ได้ compiler จึงยอมให้ใช้ operator เหล่านี้กับค่าที่มี type เป็น `T` ได้โดยไม่ error

หลักการตั้งชื่อ constraint แบบนี้: **ตั้งชื่อตามความหมายของกลุ่ม type ที่ต้องการสื่อ** เช่น `Number`, `Ordered`, `Signed`, `Unsigned` — เพื่อให้โค้ดอ่านแล้วเข้าใจทันทีว่า type parameter นี้ถูกจำกัดด้วยแนวคิดอะไร

---

## 3. `~T`: Approximation Element สำหรับ Named Type

ลองพิจารณาสถานการณ์นี้: สมมติมี named type ที่สร้างจาก `int` (ทบทวนแนวคิด named type จาก **Part 011**):

```go
type UserID int
```

`UserID` มี **underlying type** เป็น `int` แต่ในสายตาของ type system ของ Go แล้ว `UserID` กับ `int` เป็นคนละ type กัน ถ้า constraint เขียนแค่ `int | int32 | ...` (ไม่มี `~`) แบบในหัวข้อก่อน `UserID` จะ**ไม่ผ่าน constraint นั้น** เพราะ Go เช็คว่า type ตรงกันเป๊ะ (exact match) ไม่ใช่แค่ underlying type ตรงกัน

เครื่องหมาย **`~`** (tilde, เรียกว่า "approximation element") แก้ปัญหานี้ — `~int` หมายถึง **"ทุก type ที่มี underlying type เป็น `int`"** ไม่ใช่แค่ `int` เป๊ะๆ เท่านั้น:

```go
package main

import "fmt"

// ~int หมายถึง "ทุก type ที่มี underlying type เป็น int" ไม่ใช่แค่ int เป๊ะๆ
type Ordered interface {
	~int | ~int32 | ~int64 | ~float32 | ~float64 | ~string
}

func SortAsc[T Ordered](values []T) {
	for i := 0; i < len(values); i++ {
		for j := i + 1; j < len(values); j++ {
			if values[j] < values[i] {
				values[i], values[j] = values[j], values[i]
			}
		}
	}
}

// UserID มี underlying type เป็น int แต่เป็นคนละ type กับ int ตรงๆ
type UserID int

func main() {
	ids := []UserID{5, 2, 9, 1}
	SortAsc(ids) // ใช้ได้เพราะ constraint ใช้ ~int ไม่ใช่ int
	fmt.Println(ids)

	names := []string{"charlie", "alice", "bob"}
	SortAsc(names)
	fmt.Println(names)
}
```

ผลลัพธ์:

```
[1 2 5 9]
[alice bob charlie]
```

ถ้าเปลี่ยน constraint `Ordered` ให้เป็น `int | int32 | ... | string` (ไม่มี `~`) แล้วเรียก `SortAsc(ids)` (ที่ `ids` เป็น `[]UserID`) จะได้ **compile error** ทันที เพราะ `UserID` ไม่ตรงกับ `int` แบบเป๊ะๆ — นี่คือเหตุผลที่ constraint ที่ออกแบบมาให้ใช้งานทั่วไป (general-purpose) มักใส่ `~` นำหน้าเสมอ เพื่อให้รองรับทั้ง type พื้นฐานและ named type ที่ผู้ใช้สร้างขึ้นเองจาก type พื้นฐานนั้น

---

## 4. Package `golang.org/x/exp/constraints` และ `cmp.Ordered` ใน Standard Library

เพราะ constraint แบบ `Ordered` (รวมทุก type ที่เปรียบเทียบขนาดกันได้) เป็นแพทเทิร์นที่ใช้บ่อยมากจนแทบทุกโปรเจกต์ที่ใช้ generics ต้องมี ทีม Go จึงจัดเตรียม constraint สำเร็จรูปไว้ให้แล้วในสองที่:

### `golang.org/x/exp/constraints` (Third-Party/Experimental Package)

เป็น package ทดลอง (experimental) ที่ทีม Go เองดูแล เผยแพร่ก่อนที่ generics จะเข้า standard library เต็มตัว มี constraint สำเร็จรูปเช่น `constraints.Ordered`, `constraints.Integer`, `constraints.Float`, `constraints.Signed`, `constraints.Unsigned` ต้องติดตั้งด้วย `go get` เหมือน third-party package ทั่วไปที่เรียนใน **Part 026**:

```bash
go get golang.org/x/exp/constraints
```

```go
package main

import (
	"fmt"

	"golang.org/x/exp/constraints"
)

func Max[T constraints.Ordered](a, b T) T {
	if a > b {
		return a
	}
	return b
}

func main() {
	fmt.Println(Max(3, 7))
	fmt.Println(Max("go", "rust"))
}
```

### `cmp.Ordered` ใน Standard Library (แนะนำสำหรับโค้ดใหม่)

ตั้งแต่ **Go 1.21** เป็นต้นมา (หลังจากที่ generics ได้รับการพิสูจน์แล้วว่าใช้งานได้ดีในทางปฏิบัติ) ทีม Go ได้เพิ่ม package มาตรฐานชื่อ **`cmp`** ที่มี `cmp.Ordered` เป็น constraint สำเร็จรูปสำหรับ type ที่เปรียบเทียบขนาดกันได้ทั้งหมด (ตัวเลขทุกชนิดและ string) โดย**ไม่ต้อง `go get` อะไรเลย** เพราะอยู่ใน standard library:

```go
package main

import (
	"cmp"
	"fmt"
)

func Max[T cmp.Ordered](a, b T) T {
	if a > b {
		return a
	}
	return b
}

func main() {
	fmt.Println(Max(3, 7))
	fmt.Println(cmp.Compare(3, 7)) // คืนค่า -1, 0, หรือ 1
}
```

ผลลัพธ์:

```
7
-1
```

**คำแนะนำในทางปฏิบัติ**: สำหรับโค้ดใหม่ที่ใช้ Go 1.21 ขึ้นไป ให้ใช้ `cmp.Ordered` จาก standard library แทน `golang.org/x/exp/constraints` เสมอ เพราะไม่ต้องเพิ่ม dependency ภายนอก และ `golang.org/x/exp` เป็น package ที่ระบุไว้ชัดเจนว่าเป็น **experimental** (API อาจเปลี่ยนแปลงได้โดยไม่รักษาความเข้ากันได้แบบ [Go 1 Compatibility Promise](https://go.dev/doc/go1compat) ที่กล่าวถึงใน **Part 001**) — `golang.org/x/exp/constraints` ยังพบเห็นได้ในโค้ดเก่าที่เขียนก่อน Go 1.21 หรือในบางโปรเจกต์ที่ต้องรองรับ constraint พิเศษที่ `cmp` ไม่มี (เช่น `constraints.Signed`, `constraints.Unsigned` แยกเฉพาะ)

---

## 5. Generic Data Structure: `Set[T comparable]`

มาสร้าง data structure ที่ใช้ generics แก้ปัญหาจริงกัน เริ่มด้วย **Set** — โครงสร้างข้อมูลที่เก็บค่าไม่ซ้ำกัน:

```go
package main

import "fmt"

// Set[T] คือ generic data structure ที่ใช้ map[T]struct{} เป็น backing store
// ใช้ struct{} เพราะไม่กินพื้นที่ (zero-size type) ต่างจาก map[T]bool ที่เสียพื้นที่เก็บ bool โดยไม่จำเป็น
type Set[T comparable] struct {
	items map[T]struct{}
}

func NewSet[T comparable](initial ...T) *Set[T] {
	s := &Set[T]{items: make(map[T]struct{})}
	for _, v := range initial {
		s.Add(v)
	}
	return s
}

func (s *Set[T]) Add(v T) {
	s.items[v] = struct{}{}
}

func (s *Set[T]) Remove(v T) {
	delete(s.items, v)
}

func (s *Set[T]) Contains(v T) bool {
	_, ok := s.items[v]
	return ok
}

func (s *Set[T]) Len() int {
	return len(s.items)
}

func (s *Set[T]) Union(other *Set[T]) *Set[T] {
	result := NewSet[T]()
	for v := range s.items {
		result.Add(v)
	}
	for v := range other.items {
		result.Add(v)
	}
	return result
}

func main() {
	a := NewSet(1, 2, 3)
	b := NewSet(3, 4, 5)

	fmt.Println("a contains 2:", a.Contains(2))
	fmt.Println("a contains 9:", a.Contains(9))

	u := a.Union(b)
	fmt.Println("union size:", u.Len())

	strSet := NewSet("go", "rust", "go")
	fmt.Println("string set size:", strSet.Len()) // 2 เพราะ "go" ซ้ำ
}
```

ผลลัพธ์:

```
a contains 2: true
a contains 9: false
union size: 5
string set size: 2
```

จุดที่น่าสนใจ:

- `Set[T comparable]` ใช้ `comparable` เป็น constraint เพราะต้องนำ `T` ไปเป็น **key ของ map** ซึ่ง Go บังคับว่า key ของ map ต้องเป็น type ที่ comparable ได้เท่านั้น (ทบทวนกฎนี้จาก **Part 007**)
- `map[T]struct{}` เป็นแพทเทิร์นมาตรฐานของ Go สำหรับทำ Set เพราะ `struct{}` (empty struct) **ไม่กินพื้นที่หน่วยความจำเลย** (zero-size type) ต่างจาก `map[T]bool` ที่ต้องเก็บค่า `bool` จริงๆ ทุก entry ทั้งที่ค่านั้นไม่มีความหมายอะไรเลยในบริบทของ Set (แค่เช็คว่า "มีอยู่" หรือ "ไม่มี")
- `NewSet[T comparable](initial ...T) *Set[T]` เป็น**constructor function** (ทบทวน pattern นี้จาก **Part 011**) ที่รับ variadic parameter (ทบทวนจาก **Part 009**) และ instantiate `Set[T]` ให้พร้อมใช้งานทันที

---

## 6. Generic Data Structure: Linked List

อีกตัวอย่างคลาสสิกของ generic data structure คือ **linked list** ซึ่งแสดงให้เห็นว่า generic type สามารถอ้างอิงถึงตัวเอง (self-referential) ผ่าน pointer ได้ตามปกติ:

```go
package main

import "fmt"

// node[T] และ LinkedList[T] คือ generic linked list
type node[T any] struct {
	value T
	next  *node[T]
}

type LinkedList[T any] struct {
	head *node[T]
	size int
}

func (l *LinkedList[T]) PushFront(v T) {
	l.head = &node[T]{value: v, next: l.head}
	l.size++
}

func (l *LinkedList[T]) ToSlice() []T {
	result := make([]T, 0, l.size)
	for n := l.head; n != nil; n = n.next {
		result = append(result, n.value)
	}
	return result
}

func main() {
	var list LinkedList[string]
	list.PushFront("c")
	list.PushFront("b")
	list.PushFront("a")
	fmt.Println(list.ToSlice())

	var intList LinkedList[int]
	intList.PushFront(3)
	intList.PushFront(2)
	intList.PushFront(1)
	fmt.Println(intList.ToSlice())
}
```

ผลลัพธ์:

```
[a b c]
[1 2 3]
```

สังเกตว่า `node[T]` เป็น **unexported generic type** (ตัวพิมพ์เล็กตามกฎจาก **Part 001**) ที่ใช้ภายใน package เท่านั้น ส่วน `LinkedList[T]` เป็น type ที่ export ออกไปให้ผู้ใช้เรียกใช้งาน — นี่คือแพทเทิร์นทั่วไปในการออกแบบ data structure: **ซ่อนรายละเอียดการ implement (node) ไว้ข้างใน เปิดเผยแค่ interface การใช้งานที่จำเป็น (LinkedList)**

---

## 7. ข้อจำกัด: Method รับ Type Parameter ของตัวเองไม่ได้

มีข้อจำกัดสำคัญข้อหนึ่งของ generics ใน Go ที่ต้องรู้ก่อนออกแบบ API: **method ไม่สามารถมี type parameter ของตัวเองเพิ่มเติมได้** แม้ว่า type ที่ method นั้นสังกัดอยู่จะเป็น generic type อยู่แล้วก็ตาม

ลองดูตัวอย่างที่**ผิด** (compile ไม่ผ่าน):

```go
type Box[T any] struct {
	value T
}

// ผิด: method ไม่สามารถมี type parameter ของตัวเองเพิ่มได้
func (b Box[T]) Convert[U any]() U {
	var u U
	return u
}
```

ถ้าลอง compile โค้ดข้างบนจะได้ error:

```
./main.go:7:24: syntax error: method must have no type parameters
```

**เหตุผลที่ Go ออกแบบให้เป็นข้อจำกัดนี้**: ระบบ type ของ Go (โดยเฉพาะกลไก **method set** ที่ใช้ตัดสินว่า type ไหน implement interface ไหนบ้าง ทบทวนจาก **Part 013**) ต้องรู้ **method set ของแต่ละ type ให้แน่นอนตายตัวตั้งแต่ compile time** ถ้ายอมให้ method มี type parameter ของตัวเองเพิ่มได้ จะทำให้จำนวน method ที่เป็นไปได้ของ type หนึ่งกลายเป็น**ไม่จำกัด** (เพราะเรียก `Convert[int]()`, `Convert[string]()`, `Convert[bool]()` ก็ล้วนถือเป็น "การมีอยู่" ของ method ที่ต่างกัน) ซึ่งจะทำลายกลไก interface satisfaction ที่เป็นหัวใจสำคัญของภาษา Go ไปโดยสิ้นเชิง

**ทางออกเมื่อต้องการ logic แบบนี้จริงๆ**: เปลี่ยนจาก method เป็น**ฟังก์ชันอิสระ (standalone function)** ที่รับ `Box[T]` เป็น parameter แทน เพราะฟังก์ชันอิสระมี type parameter ของตัวเองได้อย่างอิสระ ไม่ผูกกับ method set ของ type ใดๆ:

```go
package main

import "fmt"

type Box[T any] struct {
	value T
}

// ถูกต้อง: เปลี่ยนจาก method เป็นฟังก์ชันอิสระที่มี type parameter เพิ่มได้อย่างอิสระ
func Convert[T, U any](b Box[T], convert func(T) U) Box[U] {
	return Box[U]{value: convert(b.value)}
}

func main() {
	intBox := Box[int]{value: 42}
	strBox := Convert(intBox, func(n int) string {
		return fmt.Sprintf("value-%d", n)
	})
	fmt.Println(strBox)
}
```

นี่คือแพทเทิร์นมาตรฐานในการเขียนฟังก์ชันแปลง type ข้าม generic type ใน Go — ใช้ฟังก์ชันอิสระเสมอเมื่อ logic นั้นต้องการ type parameter เพิ่มเติมนอกเหนือจากที่ type เดิมมีอยู่แล้ว

---

## 8. Generics vs Interfaces: เมื่อไรควรใช้อันไหน

Generics และ Interface เป็นเครื่องมือสองอย่างที่ดูคล้ายกัน (ทั้งคู่ทำให้โค้ดทำงานกับหลาย type ได้) แต่แก้ปัญหาคนละแบบ และมีสถานการณ์ที่เหมาะสมต่างกันชัดเจน:

| ประเด็น | Interface (Part 013-014) | Generics (Part 028-029) |
|---|---|---|
| หลักการทำงาน | Runtime polymorphism — ตัดสินใจว่าจะเรียก method ไหนตอน**runtime** ผ่าน dynamic dispatch | Compile-time specialization — compiler สร้างโค้ดเฉพาะสำหรับแต่ละ type ตอน**compile time** |
| เหมาะกับ | รวม type ที่**พฤติกรรมต่างกัน**แต่มี "สัญญา" (contract) ร่วมกัน เช่น `io.Reader` ที่ `*os.File`, `bytes.Buffer`, `net.Conn` implement ต่างกันไปแต่ทำสิ่งเดียวกัน | รวม type ที่**พฤติกรรมเหมือนกันทุกตัวอักษร** ต่างกันแค่ type ของข้อมูล เช่น `Stack[int]` กับ `Stack[string]` มี logic เดียวกันเป๊ะ |
| ใส่ค่าหลาย type ใน collection เดียว | ทำได้ตามธรรมชาติ เช่น `[]io.Writer` เก็บได้ทั้ง `*os.File` และ `*bytes.Buffer` ปนกัน | ทำไม่ได้โดยตรง — `Stack[int]` กับ `Stack[string]` เป็นคนละ type กัน ผสมกันใน slice เดียวไม่ได้ |
| Type safety | อาจต้อง type assertion ตอนดึงค่ากลับมาใช้ (เสี่ยง runtime panic ถ้าใช้ผิด) | Type safety เต็มรูปแบบตอน compile time ไม่มี type assertion เลย |
| ตัวอย่างการใช้จริง | Plugin system, dependency injection, mock สำหรับ testing (`Part 035`), `io.Reader`/`io.Writer` (`Part 048`) | Data structure ทั่วไป (`Stack`, `Set`, `LinkedList`), ฟังก์ชัน utility (`Max`, `Filter`, `Map`) |

**กฎง่ายๆ ในการตัดสินใจ**: ถามตัวเองว่า **"ฉันต้องการเก็บ type ที่แตกต่างกันไว้ด้วยกัน แล้วเรียกพฤติกรรมเดียวกันจากแต่ละตัว"** (→ ใช้ interface) หรือ **"ฉันต้องการเขียน logic เดียวที่ใช้ได้กับหลาย type โดยไม่ต้องเขียนซ้ำ"** (→ ใช้ generics) ในโค้ด production จริงมักใช้**ทั้งสองอย่างร่วมกัน** เช่น generic function ที่มี constraint เป็น interface (แบบที่เห็นมาตลอดบทนี้ — `Number`, `Ordered`, `comparable` ล้วนเป็น interface ทั้งสิ้น)

---

## 9. ข้อพิจารณาด้าน Performance: Monomorphization vs Interface Boxing

เพื่อให้เข้าใจว่าทำไม generics ถึงเร็วกว่าการใช้ `interface{}` ในสถานการณ์ที่เทียบเคียงกันได้ ต้องเข้าใจกลไกเบื้องหลังทั้งสองแบบคร่าวๆ:

### Interface Boxing

เมื่อเก็บค่า concrete type (เช่น `int`) ไว้ในตัวแปร type `interface{}`/`any` Go ต้อง **"box" ค่านั้น** คือสร้างโครงสร้างข้อมูลพิเศษที่เก็บทั้ง **type information** (บอกว่าค่าข้างในเป็น type อะไร) และ **pointer ไปยังค่าจริง** กระบวนการนี้มี overhead ทั้งด้านหน่วยความจำ (ต้อง allocate เพิ่ม) และความเร็ว (ต้อง dereference ผ่าน pointer และตรวจสอบ type ตอน runtime เวลาทำ type assertion)

### Monomorphization (แนวทางที่ Go Generics ใช้)

Go compiler ใช้เทคนิคคล้ายกับที่เรียกว่า **monomorphization** (แนวคิดเดียวกับที่ Rust ใช้) — เมื่อเรียก `Stack[int]` และ `Stack[string]` compiler จะสร้างโค้ดเครื่อง (machine code) **เฉพาะทาง** สำหรับแต่ละ type ที่ถูกใช้จริง ทำให้การเข้าถึงค่าภายใน `Stack[int]` ทำงานกับ `int` โดยตรง**เหมือนกับเขียน `IntStack` มาแต่แรกด้วยมือ** ไม่มี boxing และไม่มี type assertion ใดๆ เกิดขึ้นตอน runtime เลย

ในทางปฏิบัติ ทีม Go ใช้เทคนิคผสม (GC shape stenciling) ที่แชร์โค้ดเครื่องระหว่าง type ที่มีขนาดและโครงสร้างหน่วยความจำ (memory layout) เหมือนกัน เพื่อลดขนาด binary ที่ compile ออกมาไม่ให้บวมเกินไปเมื่อใช้ generic type กับหลาย type จำนวนมาก แต่หลักการโดยรวมยังคงเป็นการสร้างโค้ดเฉพาะทางที่หลีกเลี่ยง boxing/unboxing ตอน runtime ได้เกือบทั้งหมด

**ผลลัพธ์ในทางปฏิบัติ**: โค้ดที่ใช้ generics มักมี performance**ใกล้เคียงกับการเขียน type เฉพาะทางด้วยมือ** และเร็วกว่าการใช้ `interface{}` + type assertion อย่างชัดเจนในสถานการณ์ที่ทำงานหนักและเรียกซ้ำจำนวนมาก (เช่น data structure ที่ใช้ในงานประมวลผลข้อมูลปริมาณมาก) อย่างไรก็ตาม ความแตกต่างด้าน performance นี้มักไม่มีนัยสำคัญเลยสำหรับโค้ดทั่วไปที่ไม่ได้อยู่ใน hot path ของโปรแกรม — **หลักการที่ถูกต้องคือเลือกใช้ generics เพื่อความชัดเจนของโค้ดและ type safety เป็นหลัก แล้วค่อยวัดผล performance จริงด้วยเครื่องมือ profiling (จะเรียนใน Part 083) ถ้าสงสัยว่าเป็นคอขวดจริงๆ** ไม่ควรเลือกใช้ generics แทน interface (หรือกลับกัน) จากการคาดเดาเรื่อง performance ล่วงหน้าโดยไม่มีข้อมูลวัดผลจริงรองรับ

---

## สรุปสิ่งที่ได้เรียนในบทนี้

- นิยาม constraint เองด้วย union of types: `type Number interface { int | float64 | ... }` ทำให้ใช้ operator เฉพาะ (`+`, `<`, `>`) กับ type parameter ได้
- `~T` (approximation element) ขยาย constraint ให้ครอบคลุมทุก named type ที่มี `T` เป็น underlying type ไม่ใช่แค่ `T` เป๊ะๆ
- `golang.org/x/exp/constraints` เป็น package ทดลองที่มี constraint สำเร็จรูป ส่วน `cmp.Ordered` ใน standard library (ตั้งแต่ Go 1.21) คือทางเลือกที่แนะนำสำหรับโค้ดใหม่เพราะไม่ต้องพึ่ง dependency ภายนอก
- Generic data structure ที่พบบ่อย: `Set[T comparable]` (ใช้ `map[T]struct{}`) และ linked list ที่ใช้ unexported generic node type ซ่อนรายละเอียด implementation
- **Method ไม่สามารถมี type parameter ของตัวเองเพิ่มได้** เพราะจะทำลายความแน่นอนของ method set ตอน compile time — ต้องเปลี่ยนไปใช้ฟังก์ชันอิสระแทนเมื่อต้องการ logic แบบนี้
- Interface เหมาะกับการรวม type ที่พฤติกรรมต่างกันแต่มี "สัญญา" ร่วมกัน (runtime polymorphism), generics เหมาะกับ logic ที่เหมือนกันทุกตัวอักษรแต่ต่างแค่ type ของข้อมูล (compile-time specialization)
- Generics ใช้แนวทางคล้าย monomorphization สร้างโค้ดเฉพาะทางสำหรับแต่ละ type หลีกเลี่ยง interface boxing ทำให้ performance ใกล้เคียงกับเขียน type เฉพาะทางด้วยมือ — แต่ควรตัดสินใจเลือกใช้จากความชัดเจนของโค้ดเป็นหลัก แล้วค่อย profile จริงถ้าสงสัยเรื่อง performance

## แบบฝึกหัดท้ายบท

1. นิยาม constraint `Signed interface { ~int | ~int8 | ~int16 | ~int32 | ~int64 }` แล้วเขียนฟังก์ชัน `Abs[T Signed](n T) T` ที่คืนค่าสัมบูรณ์ (absolute value) ทดสอบกับทั้ง `int` และ named type ที่สร้างจาก `int32`
2. เขียน generic function `SumOrdered[T cmp.Ordered](s []T) T` โดยใช้ `cmp.Ordered` จาก standard library หาค่ามากที่สุดใน slice แล้วทดสอบกับ `[]string`
3. ขยาย `Set[T comparable]` จากบทเรียนให้มี method `Intersection(other *Set[T]) *Set[T]` (คืนค่าที่มีอยู่ในทั้งสอง set) และ `Difference(other *Set[T]) *Set[T]` (คืนค่าที่มีใน set แรกแต่ไม่มีใน set หลัง)
4. เขียน generic type `Queue[T any]` (คิว แบบ FIFO) ที่มี method `Enqueue`, `Dequeue`, `IsEmpty` โดยใช้แนวคิดคล้าย `Stack[T]` จาก **Part 028**
5. ลองเขียน method ที่มี type parameter ของตัวเองเพิ่มเติมบน generic type ที่สร้างขึ้น แล้วดู error message ที่ compiler แจ้ง จากนั้นแก้ปัญหาด้วยการเปลี่ยนเป็นฟังก์ชันอิสระตามแนวทางในบทเรียน
6. เขียนโปรแกรมเปรียบเทียบ: implement ฟังก์ชันหาค่ามากสุดสองแบบ (แบบ generic ด้วย `cmp.Ordered` และแบบใช้ `interface{}` + type switch) แล้วอธิบายด้วยคำพูดของตัวเองว่าทำไมแบบ generic ถึงปลอดภัยกว่าตอน compile time

---

**ต่อไป**: [Part 030 — Struct Embedding และ Interface Embedding](./030-embedding.md)
