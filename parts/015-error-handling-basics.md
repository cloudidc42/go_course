# Part 015: Error Handling พื้นฐาน

> ภาคที่ 1: พื้นฐานภาษา Go (Fundamentals) — ตอนที่ 15 จาก 15

## สารบัญของบทนี้

1. ปรัชญาการจัดการข้อผิดพลาดของ Go: ทำไมไม่มี Exception
2. `error` Interface: ง่ายที่สุดเท่าที่จะเป็นไปได้
3. สร้าง Error ด้วย `errors.New`
4. สร้าง Error แบบมีรูปแบบด้วย `fmt.Errorf`
5. Pattern มาตรฐาน: `if err != nil`
6. Error Value เป็น Sentinel: เปรียบเทียบด้วย `==`
7. ทำไมการทิ้ง Error ด้วย `_` ถึงอันตราย
8. การส่งต่อ Error ขึ้นไปตาม Call Stack
9. สรุปสิ่งที่ได้เรียนในบทนี้
10. แบบฝึกหัดท้ายบท

---

## 1. ปรัชญาการจัดการข้อผิดพลาดของ Go: ทำไมไม่มี Exception

ตามที่กล่าวไปใน Part 001 หนึ่งในการตัดสินใจออกแบบที่เด่นชัดที่สุดของ Go คือ **ไม่มี exception และ `try`/`catch`** เหมือนภาษาอย่าง Java, Python, JavaScript หรือ C# ภาษาเหล่านั้นจัดการข้อผิดพลาดด้วยการ "โยน" (throw) exception ขึ้นไปแล้วให้โค้ดชั้นบนดัก (catch) เอา:

```java
// สไตล์ Java/Python: exception ลอยขึ้นไปเรื่อยๆ จนกว่าจะมีคน catch
try {
    int result = divide(10, 0);
} catch (ArithmeticException e) {
    System.out.println("Error: " + e.getMessage());
}
```

ปัญหาของแนวทางนี้ (ในมุมมองของผู้ออกแบบ Go) คือ:

- **มองจากจุดที่เรียกฟังก์ชันแล้วไม่รู้เลยว่ามันอาจ throw อะไรได้บ้าง** ต้องไปเปิดอ่าน implementation หรือ documentation เอง signature ของฟังก์ชันไม่ได้บอกอะไรเลย
- **Exception กระโดดข้าม stack หลายชั้นได้** ทำให้ยากต่อการตามรอยว่าจุดไหนเป็นต้นเหตุจริงๆ โดยเฉพาะในโปรแกรมขนาดใหญ่
- เกิด**วัฒนธรรมการเขียนโค้ดที่มองข้าม error case** ได้ง่าย เพราะไม่บังคับให้ต้องจัดการ ณ จุดที่เกิดปัญหาทันที

Go เลือกแนวทางตรงกันข้ามอย่างสิ้นเชิง: **ข้อผิดพลาดคือค่า (error as a value)** เหมือนค่าอื่นๆ ในภาษา ถูก return กลับออกมาจากฟังก์ชันตรงๆ ผ่าน multiple return values (ที่เรียนไปแล้วใน Part 008) และผู้เรียกต้อง **ตรวจสอบมันอย่างชัดเจนทุกครั้ง** ก่อนจะใช้ผลลัพธ์อื่น

```go
result, err := divide(10, 0)
if err != nil {
    fmt.Println("Error:", err)
    return
}
fmt.Println(result)
```

ข้อดีของแนวทางนี้:

- **Signature ของฟังก์ชันบอกตรงๆ ว่ามันอาจล้มเหลวได้** — เห็น return type เป็น `(T, error)` ก็รู้ทันทีว่าต้องเช็ค error
- **Control flow เป็นเส้นตรง อ่านจากบนลงล่างได้เลย** ไม่มีการกระโดดข้ามชั้นแบบ exception ที่มองไม่เห็นในโค้ด
- **บังคับ (โดยธรรมเนียมและ tooling อย่าง `go vet`/`golangci-lint`) ให้จัดการ error ทันทีที่จุดเกิดเหตุ** ลดโอกาสที่ error จะถูกมองข้ามไปเงียบๆ

แน่นอนว่าแนวทางนี้ก็มีข้อเสียคือโค้ดจะมี `if err != nil` ซ้ำๆ เยอะมาก ซึ่งเป็นเรื่องที่ถูกวิพากษ์วิจารณ์บ่อยที่สุดเกี่ยวกับ Go แต่ทีมออกแบบภาษามองว่าความชัดเจนและคาดเดาได้ (explicitness and predictability) สำคัญกว่าความกระชับของโค้ด — สอดคล้องกับปรัชญา "less is more" ที่เน้นความเรียบง่ายและตรงไปตรงมาซึ่งเราเห็นซ้ำๆ ตลอดหลักสูตรนี้

---

## 2. `error` Interface: ง่ายที่สุดเท่าที่จะเป็นไปได้

`error` ใน Go **ไม่ใช่ keyword พิเศษหรือกลไกภาษาเฉพาะ** แต่เป็นแค่ **interface ธรรมดา** ที่ประกาศไว้ใน built-in ของภาษา (ไม่ต้อง import อะไรเพิ่ม) ตามที่เรียนหลักการ interface ไปใน Part 013-014:

```go
type error interface {
	Error() string
}
```

แค่นั้นเลย — interface ที่มี method เดียวชื่อ `Error()` คืนค่าเป็น `string` **ใครก็ตามที่มี method `Error() string` ก็ถือว่าเป็น `error` ได้ทันที** ผ่านกลไก implicit satisfaction ที่เรียนไปแล้ว โดยไม่ต้องประกาศอะไรพิเศษเพิ่มเติม

```go
package main

import "fmt"

// error ใน Go คือ interface มาตรฐานที่มี method เดียว: Error() string
// ใครก็ตามที่มี method Error() string ก็ถือว่าเป็น error ได้ทันที (implicit satisfaction)
type simpleError struct {
	message string
}

func (e simpleError) Error() string {
	return e.message
}

func mustBePositive(n int) error {
	if n <= 0 {
		return simpleError{message: fmt.Sprintf("%d is not positive", n)}
	}
	return nil
}

func main() {
	err := mustBePositive(-3)
	fmt.Println(err)         // เรียก Error() โดยอัตโนมัติผ่าน fmt
	fmt.Println(err.Error()) // เรียกตรงๆ ก็ได้ผลเหมือนกัน

	var e error = err // simpleError implement error interface โดยไม่ต้องประกาศ implements ใดๆ
	fmt.Println(e)
}
```

ผลลัพธ์:

```
-3 is not positive
-3 is not positive
-3 is not positive
```

สังเกตว่า `fmt.Println(err)` พิมพ์ข้อความเดียวกับ `err.Error()` เป๊ะ — นั่นเพราะ `fmt` เช็คว่าค่าที่ได้รับมามี method `Error() string` หรือไม่ (คล้ายกับที่มันเช็ค `String()` จาก `fmt.Stringer` ใน Part 013) ถ้ามี มันจะเรียก `Error()` มาใช้แสดงผลโดยอัตโนมัติ

ความจริงที่ `error` เป็นแค่ interface ธรรมดานี้เอง คือรากฐานสำคัญที่ทำให้ Go ขยาย error ให้มีข้อมูลเพิ่มเติม (custom error type ที่มี field เก็บรายละเอียด) ได้อย่างอิสระในอนาคต — เรื่องนี้จะเรียนแบบเจาะลึกเต็มๆ ใน **Part 016**

---

## 3. สร้าง Error ด้วย `errors.New`

ในทางปฏิบัติ เราแทบไม่ต้องประกาศ struct ของตัวเองแบบ `simpleError` ในหัวข้อก่อนหน้าเพื่อสร้าง error ง่ายๆ เพราะ standard library มี package `errors` ที่มีฟังก์ชันสำเร็จรูปให้ใช้งานอยู่แล้ว: `errors.New`

```go
package main

import (
	"errors"
	"fmt"
)

// divide คืนค่า error แบบ explicit แทนการ throw exception
func divide(a, b float64) (float64, error) {
	if b == 0 {
		return 0, errors.New("division by zero")
	}
	return a / b, nil
}

func main() {
	result, err := divide(10, 2)
	if err != nil {
		fmt.Println("เกิดข้อผิดพลาด:", err)
		return
	}
	fmt.Println("ผลลัพธ์:", result)

	result, err = divide(10, 0)
	if err != nil {
		fmt.Println("เกิดข้อผิดพลาด:", err)
		return
	}
	fmt.Println("ผลลัพธ์:", result)
}
```

ผลลัพธ์:

```
ผลลัพธ์: 5
เกิดข้อผิดพลาด: division by zero
```

`errors.New(message string) error` สร้างค่า error ที่เก็บ string ข้อความไว้ข้างใน แล้วคืนออกมาเป็น `error` interface — ข้างในของ package `errors` เพียงแค่ประกาศ struct ง่ายๆ ที่มี method `Error()` คืนข้อความนั้น (คล้ายกับ `simpleError` ที่เราเขียนเองในหัวข้อก่อนหน้าเป๊ะๆ) ดังนั้นในทางปฏิบัติ **ไม่ควรเขียน struct error เองถ้าแค่ต้องการข้อความ error ธรรมดาโดยไม่มีข้อมูลเพิ่มเติม** ให้ใช้ `errors.New` แทนเสมอ

---

## 4. สร้าง Error แบบมีรูปแบบด้วย `fmt.Errorf`

บ่อยครั้งที่เราต้องการใส่ค่าตัวแปรลงไปในข้อความ error (เช่น ชื่อไฟล์ที่หาไม่เจอ, ค่าที่ผิดพลาด) การเอา `errors.New` มาต่อกับ `fmt.Sprintf` ทำได้แต่ดูเทอะทะ Go จึงมี `fmt.Errorf` ที่ทำสองอย่างนี้พร้อมกันในฟังก์ชันเดียว โดยใช้ format string เหมือนที่เรียนใน Part 021 (`fmt` เจาะลึก) ทุกประการ:

```go
qty := -5
if qty < 0 {
	err := fmt.Errorf("invalid quantity: %d (must be >= 0)", qty)
	fmt.Println(err)
}
```

ผลลัพธ์:

```
invalid quantity: -5 (must be >= 0)
```

`fmt.Errorf(format string, args ...any) error` ทำงานเหมือน `fmt.Sprintf` ทุกอย่าง เพียงแต่คืนค่าเป็น `error` แทนที่จะเป็น `string` ธรรมดา ในโค้ด Go ทั่วไป `fmt.Errorf` ถูกใช้บ่อยกว่า `errors.New` มาก เพราะข้อความ error ส่วนใหญ่มักต้องใส่ context (เช่น ชื่อตัวแปร, ค่าที่ผิด, ชื่อฟังก์ชัน) เข้าไปด้วยเสมอ

> **หมายเหตุ**: `fmt.Errorf` ยังมีความสามารถพิเศษเรื่อง **error wrapping** ด้วย verb `%w` ที่ทำให้ error ใหม่ "ห่อ" error เดิมไว้ข้างในแบบตรวจสอบย้อนกลับได้ — เรื่องนี้เป็นหัวข้อสำคัญที่จะเรียนแบบเจาะลึกใน **Part 016** พร้อมกับ `errors.Is` และ `errors.As` บทนี้ขอเก็บ `fmt.Errorf` ไว้แค่ในฐานะเครื่องมือสร้างข้อความ error ธรรมดาก่อน

---

## 5. Pattern มาตรฐาน: `if err != nil`

รูปแบบการเช็ค error ที่พบได้แทบทุกบรรทัดในโค้ด Go คือ:

```go
result, err := someFunction()
if err != nil {
    // จัดการข้อผิดพลาด แล้ว return/break/continue ออกจาก flow ปกติ
    return err
}
// ใช้งาน result ต่อได้อย่างมั่นใจ เพราะผ่านการเช็คแล้วว่าไม่มี error
```

หลักการสำคัญของ pattern นี้คือ **เช็คทันทีหลังเรียกฟังก์ชันที่อาจล้มเหลว** ไม่ปล่อยให้ error ค้างอยู่เฉยๆ แล้วไปเช็คทีหลัง เพราะถ้ามี error เกิดขึ้นจริง ค่าตัวแรก (`result` ในตัวอย่างนี้) มักจะเป็น **zero value** ที่ใช้งานต่อไม่ได้เลย (ตามที่เคยเรียนเรื่อง multiple return values ใน Part 008 — ธรรมเนียมของ Go คือ return zero value ของ type นั้นคู่กับ error ที่ไม่เป็น nil)

```go
package main

import (
	"fmt"
	"strconv"
)

func main() {
	input := "abc"

	n, err := strconv.Atoi(input) // แปลง string เป็น int อาจล้มเหลวถ้า string ไม่ใช่ตัวเลข
	if err != nil {
		fmt.Println("แปลงค่าไม่สำเร็จ:", err)
		return // ออกจากฟังก์ชันทันที ไม่ใช้ n ต่อเพราะมันเป็น zero value (0) ที่ไม่มีความหมาย
	}

	fmt.Println("แปลงสำเร็จ ได้ค่า:", n)
}
```

ธรรมเนียมการเขียนโค้ด Go ที่ยึดถือกันอย่างกว้างขวางคือ **early return** — เช็ค error แล้วรีบ `return` ออกไปทันทีถ้าเกิดปัญหา แทนที่จะเขียน logic ปกติไว้ใน `else` block ทำให้ระดับการ indent ของโค้ดไม่ลึกเกินไป และ "happy path" (เส้นทางที่ทุกอย่างทำงานถูกต้อง) อ่านไล่ลงมาเป็นเส้นตรงชัดเจน:

```go
// แนวทางที่แนะนำ: early return, happy path อยู่นอก if
n, err := strconv.Atoi(input)
if err != nil {
	return err
}
fmt.Println(n) // happy path อยู่ระดับ indent เดียวกับ err check ไม่ต้องซ้อน else

// แนวทางที่ไม่แนะนำ: ซ้อน else ทำให้ indent ลึกขึ้นเรื่อยๆ เมื่อมีการเช็ค error หลายจุด
n, err := strconv.Atoi(input)
if err != nil {
	return err
} else {
	fmt.Println(n)
}
```

---

## 6. Error Value เป็น Sentinel: เปรียบเทียบด้วย `==`

บางครั้งเราต้องการมากกว่าแค่ "รู้ว่ามี error" — เราต้องการรู้ว่า **error ชนิดไหนกันแน่** เพื่อตัดสินใจทำอย่างอื่นต่อ วิธีที่ง่ายและตรงไปตรงมาที่สุดคือการประกาศค่า error คงที่ระดับ package ไว้ล่วงหน้า เรียกว่า **sentinel error** แล้วให้ผู้เรียกเปรียบเทียบด้วย `==` ตรงๆ

Standard library มีตัวอย่างของแนวคิดนี้อยู่หลายที่ ที่รู้จักกันดีที่สุดคือ:

- **`io.EOF`** — คืนกลับมาจาก `Read()` เมื่ออ่านข้อมูลจนหมดแล้ว (เห็นตัวอย่างการใช้งานจริงไปแล้วใน Part 013 ตอนอ่านค่าจาก `strings.Reader`)
- **`sql.ErrNoRows`** — คืนกลับมาจาก `database/sql` เมื่อ query ไม่เจอแถวไหนเลย (จะเรียนแบบเต็มใน Part 071)

ลองเขียน sentinel error ของตัวเองในสไตล์เดียวกัน:

```go
package main

import (
	"errors"
	"fmt"
)

// ErrNotFound คือ sentinel error: ค่า error คงที่ระดับ package ที่ผู้เรียกใช้เปรียบเทียบได้ตรงๆ ด้วย ==
// รูปแบบเดียวกับ io.EOF หรือ sql.ErrNoRows ใน standard library
var ErrNotFound = errors.New("item not found")

type Store struct {
	items map[string]int
}

func (s *Store) Get(key string) (int, error) {
	v, ok := s.items[key]
	if !ok {
		return 0, ErrNotFound
	}
	return v, nil
}

func main() {
	store := &Store{items: map[string]int{"apple": 10}}

	v, err := store.Get("apple")
	fmt.Println(v, err)

	_, err = store.Get("banana")
	if err == ErrNotFound { // เปรียบเทียบกับ sentinel error ได้ตรงๆ เพราะเป็นค่าเดียวกันเป๊ะ
		fmt.Println("ไม่พบสินค้าตามที่คาด:", err)
	}
}
```

ผลลัพธ์:

```
10 <nil>
ไม่พบสินค้าตามที่คาด: item not found
```

เหตุผลที่เปรียบเทียบด้วย `==` ได้ตรงๆ เพราะ `ErrNotFound` เป็น**ตัวแปรตัวเดียว (ตำแหน่งในหน่วยความจำเดียวกัน)** ที่ถูกสร้างขึ้นครั้งเดียวตอนเริ่มโปรแกรม ทุกที่ที่ `return ErrNotFound` คือ return ค่าเดียวกันเป๊ะ การเทียบ `==` จึงเทียบว่า "เป็นตัวแปรเดียวกันหรือเปล่า" ซึ่งตรงตามเจตนา — ต่างจากการสร้าง `errors.New("item not found")` ขึ้นมาใหม่ทุกครั้งที่ error เกิดขึ้น ซึ่งจะได้ค่าคนละตัวกัน เทียบ `==` แล้วไม่เท่ากันแม้ข้อความจะเหมือนกันทุกตัวอักษรก็ตาม

**ข้อจำกัดของ sentinel error แบบพื้นฐานนี้**: มันบอกได้แค่ "error คืออะไร" (identity) แต่ไม่มีที่เก็บข้อมูลรายละเอียดเพิ่มเติม (เช่น ชื่อ key ที่หาไม่เจอ, error code) และถ้า error ถูกห่อซ้อนหลายชั้น (wrapped) การเทียบด้วย `==` ตรงๆ แบบนี้จะใช้ไม่ได้ผลอีกต่อไป — ปัญหาทั้งสองนี้คือเหตุผลที่ Go 1.13 เพิ่ม **custom error type** และฟังก์ชัน **`errors.Is`/`errors.As`** เข้ามา ซึ่งจะเรียนแบบเจาะลึกเต็มรูปแบบใน **Part 016** บทนี้ขอให้เข้าใจแค่หลักการพื้นฐานของ sentinel error ก่อนเป็นพื้นฐาน

---

## 7. ทำไมการทิ้ง Error ด้วย `_` ถึงอันตราย

Go ใช้ underscore (`_`) เป็น **blank identifier** สำหรับ "ทิ้ง" ค่าที่ไม่ต้องการ (เรียนไปแล้วตอน multiple return values ใน Part 008) และเนื่องจาก error ก็เป็นแค่ค่าคืนตัวหนึ่งเหมือนกัน **Go จึงไม่ได้บังคับให้เราต้องเช็ค error เสมอไปในทางไวยากรณ์** — เราทิ้งมันได้ตามใจชอบด้วย `_`

```go
data, _ := os.ReadFile("does-not-exist.txt") // ทิ้ง error ไปเฉยๆ -- อันตรายมาก!
fmt.Println(len(data))
```

ปัญหาคือ **โค้ดแบบนี้ compile ผ่านสบายๆ ไม่มี warning ใดๆ** จากตัว compiler เอง (แม้ `go vet` และ linter อย่าง `golangci-lint` ที่จะเรียนใน Part 086 จะช่วยเตือนได้ในหลายกรณี) แต่ผลลัพธ์ทาง logic คือความหายนะเงียบๆ ลองดูตัวอย่างเปรียบเทียบเต็มๆ:

```go
package main

import (
	"fmt"
	"os"
)

func main() {
	// ตัวอย่างที่ไม่ควรทำ: ใช้ _ ทิ้ง error ไปเฉยๆ
	data, _ := os.ReadFile("does-not-exist.txt")
	fmt.Println("ความยาวข้อมูลที่อ่านได้:", len(data)) // ได้ 0 เงียบๆ โดยไม่รู้เลยว่าอ่านไฟล์ไม่สำเร็จ
	fmt.Printf("data == nil: %v\n", data == nil)

	// ตัวอย่างที่ถูกต้อง: เช็ค error เสมอ
	data2, err := os.ReadFile("does-not-exist.txt")
	if err != nil {
		fmt.Println("อ่านไฟล์ไม่สำเร็จ (ตามที่คาด):", err)
	} else {
		fmt.Println("อ่านได้:", data2)
	}
}
```

ผลลัพธ์:

```
ความยาวข้อมูลที่อ่านได้: 0
data == nil: true
อ่านไฟล์ไม่สำเร็จ (ตามที่คาด): open does-not-exist.txt: no such file or directory
```

ในตัวอย่างแรก โปรแกรมเดินหน้าต่อไปราวกับว่าอ่านไฟล์สำเร็จ (ได้ `data` ที่เป็น `nil` แบบเงียบๆ) — ถ้าโค้ดส่วนถัดไปเอา `data` ไปประมวลผลต่อโดยคิดว่ามันมีเนื้อหาจริง (เช่น parse เป็น JSON) ก็จะพังในจุดที่ไกลจากต้นเหตุจริงมาก ทำให้ debug ยากขึ้นหลายเท่า ต่างจากตัวอย่างที่สองที่รู้ทันทีตรงจุดเกิดเหตุว่า "อ่านไฟล์ไม่สำเร็จเพราะอะไร"

**กฎที่ควรยึดถือ**: การใช้ `_` ทิ้ง error ควรเกิดขึ้น**เฉพาะกรณีที่คิดมาอย่างรอบคอบแล้วจริงๆ ว่า error นั้นไม่สำคัญและไม่มีผลกระทบใดๆ ถ้าเกิดขึ้น** (ซึ่งพบได้น้อยมากในทางปฏิบัติ) เช่น `defer file.Close()` ในบางสถานการณ์ที่ไม่สนใจผลของการปิดไฟล์จริงๆ แม้กระนั้นก็ยังนิยมเขียนคอมเมนต์กำกับไว้ให้ชัดเจนว่าทำไมถึงทิ้ง error ตรงนั้นไปได้ เพื่อให้คนอ่านโค้ดในอนาคต (รวมถึงตัวเราเองในอนาคต) ไม่งงว่าเป็นความตั้งใจหรือความสะเพร่า

### ธรรมเนียมการเขียนข้อความ Error ที่ดี

Go มีธรรมเนียมการเขียนข้อความ error ที่ชุมชนยึดถือกันอย่างกว้างขวาง (ระบุไว้ใน [Go Code Review Comments](https://go.dev/wiki/CodeReviewComments) อย่างเป็นทางการ) ซึ่งควรทำตามเสมอ:

1. **ขึ้นต้นด้วยตัวพิมพ์เล็ก** (ยกเว้นชื่อเฉพาะหรือ acronym) เพราะข้อความ error มักถูกเอาไปต่อกับข้อความอื่น (ตามที่เห็นในหัวข้อ 8 เรื่อง propagation) การขึ้นต้นด้วยตัวใหญ่จะทำให้ประโยคที่ต่อกันดูแปลกๆ
2. **ไม่ควรลงท้ายด้วยเครื่องหมายวรรคตอน** เช่น `.` หรือ `!` เพราะข้อความอาจถูกนำไปต่อท้ายด้วยข้อความอื่นอีก
3. **บอกบริบทให้ชัดเจน** แต่ไม่ต้องใส่ stack trace หรือรายละเอียดทางเทคนิคที่มากเกินไปจนอ่านยาก

```go
// ไม่แนะนำ: ขึ้นต้นตัวใหญ่ ลงท้ายด้วยจุด
return fmt.Errorf("Invalid quantity: %d.", qty)

// แนะนำ: ตัวพิมพ์เล็ก ไม่มีจุดปิดท้าย
return fmt.Errorf("invalid quantity: %d", qty)
```

ลองดูผลลัพธ์ตอนถูกนำไปต่อกันหลายชั้นตามธรรมเนียมนี้ เทียบกับตอนไม่ทำตาม จะเห็นความต่างชัดเจน:

```
setupServer: readConfig: invalid quantity: -5      // อ่านลื่น เป็นประโยคเดียวกันต่อเนื่อง
setupServer: readConfig: Invalid quantity: -5.     // สะดุด เพราะมีตัวใหญ่และจุดแทรกอยู่กลางประโยค
```

---

## 8. การส่งต่อ Error ขึ้นไปตาม Call Stack

ในโปรแกรมจริง ฟังก์ชันมักถูกเรียกซ้อนกันหลายชั้น (function A เรียก B เรียก C) เมื่อ error เกิดขึ้นที่ชั้นในสุด มันต้อง **"เดินทาง" ขึ้นไปหาชั้นบนสุดที่รู้วิธีจัดการกับมันอย่างเหมาะสม** (เช่น แสดงข้อความให้ผู้ใช้เห็น, log ลงไฟล์, หรือ retry) กระบวนการนี้เรียกว่า **error propagation**

รูปแบบพื้นฐานที่สุดคือ **return error ต่อขึ้นไปเรื่อยๆ** โดยแต่ละชั้นอาจเพิ่มบริบท (context) เข้าไปในข้อความ เพื่อให้ผู้อ่าน log เห็นภาพรวมว่า error เกิดจากจุดไหน ผ่านลำดับการเรียกอะไรมาบ้าง:

```go
package main

import (
	"fmt"
	"strconv"
)

// readConfig จำลองการอ่านค่า config แล้วแปลงเป็นตัวเลข
func readConfig(raw string) (int, error) {
	n, err := strconv.Atoi(raw)
	if err != nil {
		// ห่อ context เพิ่มเติมแล้วส่ง error ต่อขึ้นไปให้ผู้เรียก (propagation)
		return 0, fmt.Errorf("readConfig: cannot parse %q as int: %v", raw, err)
	}
	return n, nil
}

// setupServer เรียก readConfig แล้วส่ง error ต่อขึ้นไปอีกชั้นถ้ามีปัญหา
func setupServer(portRaw string) error {
	port, err := readConfig(portRaw)
	if err != nil {
		return fmt.Errorf("setupServer: %v", err)
	}
	fmt.Println("server starting on port", port)
	return nil
}

func main() {
	if err := setupServer("8080"); err != nil {
		fmt.Println("error:", err)
	}

	if err := setupServer("not-a-number"); err != nil {
		fmt.Println("error:", err)
	}
}
```

ผลลัพธ์:

```
server starting on port 8080
error: setupServer: readConfig: cannot parse "not-a-number" as int: strconv.Atoi: parsing "not-a-number": invalid syntax
```

สังเกตข้อความ error สุดท้ายที่ `main()` ได้รับ — มันเก็บ**ร่องรอยทั้งเส้นทาง** ที่ error เดินทางผ่านมา (`setupServer` → `readConfig` → `strconv.Atoi`) ทำให้ debug ง่ายขึ้นมากโดยไม่ต้องเปิด debugger หรือดู stack trace เลย นี่คือประโยชน์ของการเพิ่ม context ทีละชั้นแทนที่จะ return error ตัวเดิมเปล่าๆ ขึ้นไปตรงๆ

รูปแบบ `fmt.Errorf("functionName: %v", err)` ที่เห็นในตัวอย่างนี้เป็นการห่อ error แบบพื้นฐานที่สุด — มันสร้าง error **ใหม่** ที่มีแค่ข้อความรวมกัน แต่**ไม่ได้เก็บความสัมพันธ์กับ error ต้นตอไว้ในเชิงโปรแกรม** (เช่น เอาไปเทียบว่า "error นี้มาจาก `io.EOF` หรือเปล่า" ด้วย `errors.Is` ไม่ได้) การห่อ error แบบที่ตรวจสอบย้อนกลับได้จริงต้องใช้ verb พิเศษ `%w` แทน `%v` ซึ่งเป็นเนื้อหาหลักของ **Part 016** ที่จะเรียนต่อจากบทนี้ทันที

---

## สรุปสิ่งที่ได้เรียนในบทนี้

- Go เลือกใช้ **ค่า error ที่ return กลับมาตรงๆ** แทน exception/`try`-`catch` เพื่อให้ signature ของฟังก์ชันบอกชัดเจนว่าอาจล้มเหลวได้ และบังคับให้จัดการ error ณ จุดเกิดเหตุทันที
- **`error`** เป็นแค่ interface มาตรฐานที่มี method เดียวคือ `Error() string` — ใครก็ตามที่มี method นี้ก็เป็น `error` ได้ทันทีผ่าน implicit satisfaction
- **`errors.New(msg)`** สร้าง error จากข้อความธรรมดา ส่วน **`fmt.Errorf(format, args...)`** สร้าง error พร้อม format string เหมือน `fmt.Sprintf` และใช้บ่อยกว่าในทางปฏิบัติ
- Pattern มาตรฐานคือ **`if err != nil { return err }`** เช็คทันทีหลังเรียกฟังก์ชันที่อาจล้มเหลว และใช้ **early return** เพื่อให้ happy path อ่านง่ายไม่ซ้อน `else` ลึกเกินไป
- **Sentinel error** คือค่า error คงที่ระดับ package (เช่น `io.EOF`, `sql.ErrNoRows`) ที่เปรียบเทียบได้ตรงๆ ด้วย `==` เพราะเป็นตัวแปรตัวเดียวกันเสมอ
- การ**ทิ้ง error ด้วย `_`** เป็นอันตราย เพราะ compile ผ่านได้โดยไม่มี warning แต่ทำให้โปรแกรมเดินหน้าต่อด้วยสถานะที่ผิดพลาดแบบเงียบๆ ควรทำเฉพาะกรณีที่คิดมาอย่างรอบคอบแล้วเท่านั้น
- **Error propagation** คือการส่ง error ต่อขึ้นไปตาม call stack โดยเพิ่ม context ทีละชั้นด้วย `fmt.Errorf` ทำให้ผู้อ่าน log เห็นเส้นทางที่ error เดินทางผ่านมาได้ทั้งหมด
- เนื้อหาเรื่อง **custom error type, `errors.Is`/`errors.As`, และการห่อ error ด้วย `%w`** เป็นหัวข้อขั้นสูงกว่านี้ ซึ่งจะเรียนแบบเจาะลึกใน **Part 016** ทันที

## แบบฝึกหัดท้ายบท

1. เขียนฟังก์ชัน `validateAge(age int) error` ที่คืน error (ด้วย `fmt.Errorf`) ถ้า `age` น้อยกว่า 0 หรือมากกว่า 150 ทดสอบเรียกด้วยค่าที่ถูกและผิด
2. เขียน sentinel error ของตัวเองชื่อ `ErrInsufficientFunds` แล้วสร้างฟังก์ชัน `Withdraw(balance, amount float64) (float64, error)` ที่คืน sentinel error นี้เมื่อเงินไม่พอ ทดสอบเปรียบเทียบ error ที่ได้ด้วย `==`
3. เขียนโค้ดที่จงใจทิ้ง error ด้วย `_` จากการเรียก `os.Open` กับไฟล์ที่ไม่มีอยู่จริง แล้วลองใช้ค่าที่ได้ต่อ (เช่นเรียก method บนมัน) สังเกตว่าเกิดอะไรขึ้น จากนั้นแก้ให้เช็ค error อย่างถูกต้อง
4. เขียนฟังก์ชัน 3 ชั้นซ้อนกัน (`levelC` เรียกจาก `levelB` เรียกจาก `levelA`) ที่แต่ละชั้นเพิ่ม context เข้าไปในข้อความ error ด้วย `fmt.Errorf` แบบเดียวกับตัวอย่างในหัวข้อ 8 แล้วดูข้อความ error สุดท้ายที่ได้จาก `main()`
5. อธิบายด้วยคำพูดของตัวเองว่าทำไม pattern `if err != nil { return err }` ถึงดีกว่าการปล่อยให้โปรแกรม panic หรือ crash ทันทีเมื่อเจอปัญหา (คำใบ้: นึกถึงความแตกต่างระหว่างข้อผิดพลาดที่คาดเดาได้กับข้อผิดพลาดร้ายแรงที่คาดเดาไม่ได้ ซึ่งจะเรียนต่อใน Part 017)
6. ค้นคว้าเพิ่มเติม: เปิดเอกสารของ `io` package (`go doc io.EOF`) แล้วอธิบายว่าทำไม `io.EOF` ถึงถูกออกแบบให้เป็น sentinel error แทนที่จะเป็นแค่ boolean `hasMore bool` ที่คืนออกมาคู่กับข้อมูล

---

**ต่อไป**: [Part 016 — Error Handling ขั้นสูง: custom error, `errors.Is/As`, wrapping](./016-error-handling-advanced.md)
