# Part 102: Design Patterns ใน Go

> ภาคที่ 10: มืออาชีพและระดับโลก (Professional & World-Class) — ตอนที่ 3 จาก 11 (Part 100–110)

หนังสือ "Design Patterns: Elements of Reusable Object-Oriented Software" (1994) โดย Gang of Four (Erich Gamma, Richard Helm, Ralph Johnson, John Vlissides — เรียกกันสั้นๆ ว่า **GoF**) คือหนังสือที่นิยาม design pattern คลาสสิก 23 แบบที่วิศวกรซอฟต์แวร์ทั่วโลกใช้ศัพท์ร่วมกันมาเกือบ 30 ปี แต่ทั้ง 23 pattern นั้นถูกออกแบบมาสำหรับภาษาที่มี class-based inheritance แบบ C++ และ Smalltalk

Go **ไม่มี** class, ไม่มี inheritance, ไม่มี constructor overloading, ไม่มี generic ตอนเปิดตัว (ทบทวนปรัชญา "less is more" จาก **Part 001**) — คำถามคือ pattern เหล่านี้ยังจำเป็นไหมใน Go? คำตอบที่บทนี้จะแสดงให้เห็นคือ: **pattern ส่วนใหญ่ยังมีประโยชน์ แต่ implement ได้เรียบง่ายกว่าเดิมมาก** เพราะ first-class function และ implicit interface satisfaction ของ Go ตัดพิธีกรรม (ceremony) ที่ภาษา OOP ดั้งเดิมต้องมีออกไปเกือบหมด และบาง pattern ก็**ไม่จำเป็นอีกต่อไปเลย**เพราะ Go แก้ปัญหาที่ pattern เหล่านั้นแก้ไว้ตั้งแต่ในตัวภาษาแล้ว

## สารบัญของบทนี้

1. ทำไม Pattern ใน Go ถึงใช้ Ceremony น้อยกว่าภาษา OOP ดั้งเดิม
2. Functional Options: ทางเลือกที่ Go นิยมแทน Builder และ Telescoping Constructor
3. Strategy Pattern: Interface หรือแม้แต่ Function Type เดียวก็เพียงพอ
4. Decorator Pattern: ทบทวนและขยายจาก io.Reader ใน Part 048
5. Observer Pattern: Channel คือ Observer ที่มีมาให้ในตัวภาษา
6. Singleton: `sync.Once` และคำเตือนเรื่อง Overuse
7. Pattern ที่ Go ทำให้ไม่จำเป็น: Factory Hierarchy และ Visitor
8. ตารางสรุป: Pattern ไหนควรใช้ Pattern ไหนควรเลี่ยง
9. สรุปสิ่งที่ได้เรียนในบทนี้
10. แบบฝึกหัดท้ายบท

---

## 1. ทำไม Pattern ใน Go ถึงใช้ Ceremony น้อยกว่าภาษา OOP ดั้งเดิม

ก่อนเข้าเนื้อหาแต่ละ pattern มาดูภาพรวมว่าทำไม Go ถึง "โกง" ทำให้ pattern คลาสสิกง่ายขึ้นมาก มี 3 เหตุผลหลัก:

1. **Function เป็น first-class citizen** (ทบทวนจาก **Part 008–009**) — ใน Java/C++ แบบเก่า การจะส่ง "พฤติกรรม" เป็นพารามิเตอร์ต้องสร้าง interface + anonymous class หรือ functor object ทั้งหมด ใน Go เราส่ง `func(...) ...` ตรงๆ ได้เลย ทำให้ pattern จำนวนมาก (Strategy, Command, Observer) ย่อลงเหลือแค่ "ส่งฟังก์ชันเป็นค่า" ไม่ต้องมี type ใหม่เลยด้วยซ้ำในหลายกรณี
2. **Implicit interface satisfaction** (ทบทวนจาก **Part 013**) — ไม่ต้องประกาศ `implements` ทำให้การ "ห่อ" (wrap) หรือ "แทนที่" (substitute) object ทำได้อย่างอิสระโดยไม่ต้องขอความร่วมมือจาก type ต้นทาง
3. **ไม่มี inheritance เลย** — pattern ของ GoF จำนวนมาก (Factory Method, Template Method, Visitor) มีอยู่เพื่อ**หลีกเลี่ยงปัญหาที่เกิดจาก class inheritance โดยเฉพาะ** เมื่อ Go ไม่มี inheritance ตั้งแต่ต้น ปัญหาที่ pattern เหล่านั้นแก้ก็ไม่เกิดขึ้นเลย ทำให้ไม่จำเป็นต้องมี pattern มาแก้ไขด้วยซ้ำ (จะอธิบายเจาะลึกในหัวข้อ 7)

บทนี้จะเดินผ่าน 5 pattern ที่**ยังมีประโยชน์ชัดเจนใน Go** พร้อมโค้ดที่ verify แล้วว่ารันได้จริง แล้วปิดท้ายด้วย pattern ที่ Go ทำให้ไม่จำเป็นอีกต่อไป

---

## 2. Functional Options: ทางเลือกที่ Go นิยมแทน Builder และ Telescoping Constructor

### ปัญหาที่ต้องแก้: Telescoping Constructor

ลองนึกภาพ `Server` struct ที่มี field จำนวนมาก และส่วนใหญ่มีค่า default ที่ดีอยู่แล้ว แต่บางครั้งก็ต้องปรับแต่งเฉพาะบางตัว ถ้าใช้ constructor ตรงๆ (แบบที่ภาษาอื่นทำผ่าน overloading) จะกลายเป็น "telescoping constructor" — ชุด constructor ที่พารามิเตอร์เพิ่มขึ้นเรื่อยๆ:

```go
// ปัญหา: ต้องมี constructor หลายเวอร์ชันไล่ตามจำนวน field ที่อยากปรับแต่ง
func NewServer(host string) *Server { /* ... */ }
func NewServerWithPort(host string, port int) *Server { /* ... */ }
func NewServerWithPortAndTimeout(host string, port int, timeout time.Duration) *Server { /* ... */ }
// ... และจะยิ่งแย่ลงเรื่อยๆ เมื่อมี field ใหม่เพิ่มเข้ามา
```

ปัญหานี้ยิ่งแย่ลงไปอีกเมื่อ Go **ไม่รองรับ function overloading เลย** (สองฟังก์ชันชื่อเดียวกันพารามิเตอร์ต่างกันไม่ได้ — ทบทวนจาก **Part 008**) ทำให้ยิ่งต้องคิดชื่อฟังก์ชันแปลกๆ แบบข้างบนไปเรื่อยๆ

### ทางออก: Functional Options Pattern

**Functional Options** คือ pattern ที่ community ของ Go นิยมใช้แทน Builder pattern แบบ Java (ที่ต้องมี builder class แยกพร้อม method chaining `.setX().setY().build()`) — ใช้ closure (ทบทวนจาก **Part 009**) เป็นหัวใจหลัก:

```go
package main

import (
	"fmt"
	"time"
)

// Server คือตัวอย่างมาตรฐานสำหรับสาธิต Functional Options Pattern —
// มี field จำนวนมากที่ "ควรมีค่า default ที่สมเหตุสมผล" แต่ก็ต้องปรับแต่งได้
// ในบางกรณี ปัญหาคลาสสิกคือถ้าใช้ constructor ตรงๆ (telescoping constructor)
// จะต้องมีหลายเวอร์ชันมากมาย: NewServer(host), NewServer(host, port),
// NewServer(host, port, timeout), NewServer(host, port, timeout, maxConns), ...
type Server struct {
	Host       string
	Port       int
	Timeout    time.Duration
	MaxConns   int
	EnableTLS  bool
	TLSCertPEM string
}

// Option คือ function type ที่แก้ไข *Server — นี่คือการใช้ closure (ทบทวน
// Part 009) เพื่อสร้าง "การตั้งค่าแบบ deferred" ที่รอถูกเรียกใน constructor
type Option func(*Server)

// WithPort, WithTimeout, WithMaxConns, WithTLS คือฟังก์ชันที่คืน Option —
// แต่ละตัวรับผิดชอบแค่ field เดียวหรือกลุ่มเดียวที่เกี่ยวข้องกัน อ่านเข้าใจง่าย
// กว่า struct literal ที่มี field เยอะๆ มาก และเพิ่ม option ใหม่ในอนาคตได้โดย
// ไม่ต้องแก้ signature ของ NewServer เลยแม้แต่นิดเดียว (backward compatible เสมอ)
func WithPort(port int) Option {
	return func(s *Server) { s.Port = port }
}

func WithTimeout(d time.Duration) Option {
	return func(s *Server) { s.Timeout = d }
}

func WithMaxConns(n int) Option {
	return func(s *Server) { s.MaxConns = n }
}

func WithTLS(certPEM string) Option {
	return func(s *Server) {
		s.EnableTLS = true
		s.TLSCertPEM = certPEM
	}
}

// NewServer รับ host ที่จำเป็นเสมอเป็น positional argument ตรงๆ (ค่าที่ไม่มี
// default ที่สมเหตุสมผลควรบังคับให้ผู้เรียกใส่มาตรงๆ ไม่ใช่ผ่าน option)
// ส่วนค่าที่เหลือทั้งหมดมี default ที่ดีอยู่แล้ว และปรับแต่งได้ผ่าน opts ...Option
// ที่ apply ทับ default ทีละตัวตามลำดับที่ผู้เรียกส่งมา
func NewServer(host string, opts ...Option) *Server {
	s := &Server{
		Host:     host,
		Port:     8080,
		Timeout:  30 * time.Second,
		MaxConns: 100,
	}
	for _, opt := range opts {
		opt(s)
	}
	return s
}

func main() {
	// เรียกแบบไม่ใส่ option เลย ได้ค่า default ทั้งหมด
	s1 := NewServer("localhost")
	fmt.Printf("s1: %+v\n", *s1)

	// เรียกแบบใส่แค่บาง option ที่ต้องการ เรียงลำดับอะไรก็ได้ ข้ามตัวไหนก็ได้
	s2 := NewServer("api.example.com",
		WithPort(443),
		WithTLS("-----BEGIN CERTIFICATE-----..."),
		WithTimeout(5*time.Second),
	)
	fmt.Printf("s2: Host=%s Port=%d Timeout=%s EnableTLS=%v\n", s2.Host, s2.Port, s2.Timeout, s2.EnableTLS)

	// เทียบกับแนวทาง Builder pattern แบบภาษา Java/C++ ที่ต้องมี BuilderStruct
	// แยกต่างหาก พร้อม method .SetX() ที่ mutate ตัวเองแล้ว return ตัวเองกลับ
	// (method chaining) — Go เลือกใช้ functional options เพราะไม่ต้องสร้าง type
	// ใหม่เพิ่มขึ้นมาเลย (Option ก็เป็นแค่ function ธรรมดา) และยังคง "immutable
	// เสร็จสมบูรณ์ในการเรียกครั้งเดียว" (ไม่มี intermediate builder state ค้างอยู่)
	s3 := NewServer("cache.internal", WithMaxConns(1000))
	fmt.Printf("s3: %+v\n", *s3)
}
```

ผลลัพธ์จริงจากการรัน `go run .`:

```
s1: {Host:localhost Port:8080 Timeout:30s MaxConns:100 EnableTLS:false TLSCertPEM:}
s2: Host=api.example.com Port=443 Timeout=5s EnableTLS=true
s3: {Host:cache.internal Port:8080 Timeout:30s MaxConns:1000 EnableTLS:false TLSCertPEM:}
```

**ทำไม pattern นี้ถึงเป็นที่นิยมมากในวงการ Go**: library ระดับ production จำนวนมากใช้ pattern นี้ เช่น `grpc.Dial(addr, opts ...grpc.DialOption)`, `http.Client` แบบ custom transport มักถูกห่อด้วย option pattern ในหลาย wrapper library และ `zap` (logging library ยอดนิยม) ก็ใช้ `zap.Option` แบบเดียวกันนี้ทุกประการ — เหตุผลที่มันชนะ Builder pattern แบบดั้งเดิมคือ **ไม่ต้องสร้าง type ใหม่ (Builder struct)** และ **backward compatible เสมอ**: เพิ่ม `WithXxx()` ใหม่ในอนาคตได้โดยไม่ทำให้โค้ดเดิมที่เรียก `NewServer(host, opts...)` พังเลยแม้แต่จุดเดียว

---

## 3. Strategy Pattern: Interface หรือแม้แต่ Function Type เดียวก็เพียงพอ

**Strategy Pattern** ของ GoF คือการ "สลับอัลกอริทึม" ได้แบบ runtime โดยไม่ต้องแก้โค้ดที่ใช้งานมัน ในภาษา OOP ดั้งเดิมต้องมี interface + concrete class แยกกันสำหรับแต่ละกลยุทธ์ พร้อม inject class ที่เลือกเข้าไปใน context — ใน Go เราทำแบบเดียวกันได้ด้วย interface เล็กๆ (ทบทวน **Part 013**) หรือในหลายกรณี **function type ตัวเดียวก็เพียงพอโดยไม่ต้องมี interface เลยด้วยซ้ำ**:

```go
package main

import (
	"fmt"
	"sort"
)

// PricingStrategy คือ interface กลยุทธ์การคำนวณราคาหลังหักส่วนลด
type PricingStrategy interface {
	CalculatePrice(basePrice float64) float64
}

// RegularPricing ไม่มีส่วนลดเลย
type RegularPricing struct{}

func (RegularPricing) CalculatePrice(basePrice float64) float64 { return basePrice }

// PercentageDiscount ลดราคาเป็นเปอร์เซ็นต์
type PercentageDiscount struct {
	Percent float64
}

func (d PercentageDiscount) CalculatePrice(basePrice float64) float64 {
	return basePrice * (1 - d.Percent/100)
}

// FixedAmountDiscount ลดราคาเป็นจำนวนเงินคงที่ (ไม่ติดลบ)
type FixedAmountDiscount struct {
	Amount float64
}

func (d FixedAmountDiscount) CalculatePrice(basePrice float64) float64 {
	result := basePrice - d.Amount
	if result < 0 {
		return 0
	}
	return result
}

// Checkout คือ "context" ที่ใช้ strategy — ไม่รู้จักเลยว่าเบื้องหลังเป็นกลยุทธ์
// แบบไหน สลับ strategy ได้แบบ runtime โดยไม่ต้องแก้โค้ดตรงนี้เลยแม้แต่บรรทัดเดียว
func Checkout(basePrice float64, strategy PricingStrategy) float64 {
	return strategy.CalculatePrice(basePrice)
}

// ตัวอย่างที่สอง: Strategy ผ่าน "function type" ตรงๆ ไม่ต้องมี interface เลยด้วยซ้ำ
// เพราะกลยุทธ์นี้มี "input-output" แบบเดียว ไม่มี state อื่นที่ต้องเก็บ —
// sort.Slice ใน standard library (ทบทวน Part 027) ก็ใช้แนวคิดเดียวกันนี้ทุกประการ
// ผ่าน parameter `less func(i, j int) bool`
type CompareFunc func(a, b int) bool

func SortInts(data []int, less CompareFunc) {
	sort.Slice(data, func(i, j int) bool { return less(data[i], data[j]) })
}

func main() {
	fmt.Println("=== Strategy ผ่าน interface ===")
	strategies := []PricingStrategy{
		RegularPricing{},
		PercentageDiscount{Percent: 10},
		FixedAmountDiscount{Amount: 50},
	}
	for _, s := range strategies {
		fmt.Printf("  %T: ราคาสุดท้าย = %.2f\n", s, Checkout(500, s))
	}

	fmt.Println("\n=== Strategy ผ่าน function type ตรงๆ (ไม่ต้องมี interface) ===")
	data := []int{5, 2, 8, 1, 9}
	ascending := func(a, b int) bool { return a < b }
	descending := func(a, b int) bool { return a > b }

	dataAsc := append([]int(nil), data...)
	SortInts(dataAsc, ascending)
	fmt.Println("  ascending:", dataAsc)

	dataDesc := append([]int(nil), data...)
	SortInts(dataDesc, descending)
	fmt.Println("  descending:", dataDesc)
}
```

ผลลัพธ์จริง:

```
=== Strategy ผ่าน interface ===
  main.RegularPricing: ราคาสุดท้าย = 500.00
  main.PercentageDiscount: ราคาสุดท้าย = 450.00
  main.FixedAmountDiscount: ราคาสุดท้าย = 450.00

=== Strategy ผ่าน function type ตรงๆ (ไม่ต้องมี interface) ===
  ascending: [1 2 5 8 9]
  descending: [9 8 5 2 1]
```

**หลักการเลือกว่าจะใช้ interface หรือ function type ตรงๆ**: ถ้ากลยุทธ์นั้นมีแค่ "input มา output ไป" ครั้งเดียวไม่มี state อื่นที่ต้องเก็บ (เหมือน `CompareFunc`) — ใช้ function type ตรงๆ ได้เลย ประหยัดโค้ดกว่ามาก แต่ถ้ากลยุทธ์นั้นต้องมี configuration หลาย field ประกอบกัน (เหมือน `PercentageDiscount{Percent: 10}` ที่ต้องเก็บค่า `Percent` ไว้) การใช้ interface + struct จะอ่านง่ายและขยายได้ดีกว่า

---

## 4. Decorator Pattern: ทบทวนและขยายจาก io.Reader ใน Part 048

**Decorator Pattern** คือการ "ห่อ" object ตัวหนึ่งด้วย object อีกตัวที่มี interface เดียวกัน เพื่อเพิ่มพฤติกรรมโดยไม่แก้ตัวต้นฉบับ — เราเคยเห็นตัวอย่างนี้มาแล้วจริงๆ ใน **Part 048** กับ `HashingReader` ที่ห่อ `io.Reader` ตัวไหนก็ได้แล้วคำนวณ hash ไปพร้อมกันระหว่างอ่าน มาขยายตัวอย่างนั้นด้วยการห่อซ้อนกันหลายชั้น และโชว์ว่า middleware pattern จาก **Part 057** ก็คือ decorator ตัวเดียวกันในคราบของ `http.Handler`:

```go
package main

import (
	"fmt"
	"io"
	"strings"
)

// CountingReader คือ decorator ตัวแรก: ห่อ io.Reader ตัวไหนก็ได้ แล้วนับจำนวน
// byte ที่อ่านผ่านไปทั้งหมด สังเกตว่า CountingReader เองก็ implement io.Reader
// ด้วย (มี method Read เหมือนกัน) จึงเอาไปห่อซ้อนกันเป็นชั้นๆ ได้ไม่จำกัด
type CountingReader struct {
	r     io.Reader
	Count int
}

func NewCountingReader(r io.Reader) *CountingReader {
	return &CountingReader{r: r}
}

func (c *CountingReader) Read(p []byte) (int, error) {
	n, err := c.r.Read(p)
	c.Count += n
	return n, err
}

// UpperCaseReader คือ decorator ตัวที่สอง: แปลงตัวอักษรเป็นตัวพิมพ์ใหญ่ระหว่าง
// อ่านผ่าน — ไม่รู้จัก CountingReader หรือ reader ต้นฉบับเลย รู้แค่ว่า "มีอะไรก็ตาม
// ที่ implement io.Reader ส่งเข้ามา"
type UpperCaseReader struct {
	r io.Reader
}

func NewUpperCaseReader(r io.Reader) *UpperCaseReader {
	return &UpperCaseReader{r: r}
}

func (u *UpperCaseReader) Read(p []byte) (int, error) {
	n, err := u.r.Read(p)
	for i := 0; i < n; i++ {
		if p[i] >= 'a' && p[i] <= 'z' {
			p[i] -= 'a' - 'A'
		}
	}
	return n, err
}

func main() {
	fmt.Println("=== Decorator ซ้อนกันหลายชั้นบน io.Reader ===")
	base := strings.NewReader("hello, clean architecture and design patterns in go")

	// ห่อ base ด้วย UpperCaseReader ก่อน แล้วห่อผลลัพธ์นั้นอีกทีด้วย CountingReader
	// ลำดับการห่อมีผลต่อพฤติกรรม: ชั้นนอกสุดคือชั้นที่ io.Copy จะเรียกก่อนเสมอ
	upper := NewUpperCaseReader(base)
	counting := NewCountingReader(upper)

	result, err := io.ReadAll(counting)
	if err != nil {
		panic(err)
	}
	fmt.Printf("ผลลัพธ์: %s\n", result)
	fmt.Printf("จำนวน byte ที่อ่านทั้งหมด (นับผ่าน decorator): %d\n\n", counting.Count)

	fmt.Println("=== Decorator กับ http.Handler (middleware คือ decorator ในคราบอื่น) ===")
	demoHandlerDecorator()
}
```

```go
package main

import (
	"fmt"
	"net/http"
	"net/http/httptest"
	"time"
)

// middleware ที่เรียนไปใน Part 057 คือ Decorator Pattern ในคราบของ Go: มันรับ
// http.Handler ตัวหนึ่งเข้ามา แล้วคืน http.Handler ตัวใหม่ที่ "หุ้ม" พฤติกรรมเพิ่ม
// รอบๆ ตัวเดิม โดยที่ handler ต้นฉบับไม่รู้ตัวเลยว่าถูกหุ้มอยู่ — เหมือนกับที่
// CountingReader หุ้ม io.Reader ทุกประการ แค่เปลี่ยน interface จาก io.Reader
// เป็น http.Handler เท่านั้นเอง
func withLogging(next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		start := time.Now()
		next.ServeHTTP(w, r)
		fmt.Printf("  [log] %s %s ใช้เวลา %s\n", r.Method, r.URL.Path, time.Since(start))
	})
}

func withHeader(key, value string) func(http.Handler) http.Handler {
	return func(next http.Handler) http.Handler {
		return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
			w.Header().Set(key, value)
			next.ServeHTTP(w, r)
		})
	}
}

func demoHandlerDecorator() {
	base := http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		w.Write([]byte("hello from base handler"))
	})

	// ห่อ base ด้วย decorator สองชั้น เรียงจากในสุดไปนอกสุด: header -> logging
	decorated := withLogging(withHeader("X-Powered-By", "go-decorators")(base))

	server := httptest.NewServer(decorated)
	defer server.Close()

	resp, err := http.Get(server.URL + "/hello")
	if err != nil {
		panic(err)
	}
	defer resp.Body.Close()
	fmt.Printf("  ได้ header X-Powered-By: %s\n", resp.Header.Get("X-Powered-By"))
}
```

ผลลัพธ์จริง:

```
=== Decorator ซ้อนกันหลายชั้นบน io.Reader ===
ผลลัพธ์: HELLO, CLEAN ARCHITECTURE AND DESIGN PATTERNS IN GO
จำนวน byte ที่อ่านทั้งหมด (นับผ่าน decorator): 51

=== Decorator กับ http.Handler (middleware คือ decorator ในคราบอื่น) ===
  [log] GET /hello ใช้เวลา 5.766µs
  ได้ header X-Powered-By: go-decorators
```

จุดที่ควรสังเกต: **ทั้ง `io.Reader` decorator และ `http.Handler` middleware ใช้หลักการเดียวกันทุกประการ** — รับ interface ตัวหนึ่งเข้ามา คืน implementation ใหม่ที่ interface เดียวกัน ห่อพฤติกรรมเพิ่มรอบๆ การเรียกเดิม นี่คือเหตุผลที่พูดได้ว่า **middleware ก็คือ Decorator Pattern** เพียงแต่ในวงการ web development นิยมเรียกมันว่า "middleware" มากกว่า "decorator" — แต่โครงสร้างทางความคิดเหมือนกันทุกประการ และ Go ทำให้มันเรียบง่ายกว่า GoF ดั้งเดิมมาก เพราะไม่ต้องมี "abstract Decorator base class" ที่ implement interface แล้ว delegate ทุก method ไปที่ wrapped object (ซึ่งเป็นภาระในภาษา OOP ดั้งเดิมที่มี interface ขนาดใหญ่) — ใน Go ที่นิยม small interfaces (ทบทวน **Part 013 หัวข้อ 8**) การห่อ interface ที่มีแค่ 1 method จึงเบากว่ามาก

---

## 5. Observer Pattern: Channel คือ Observer ที่มีมาให้ในตัวภาษา

**Observer Pattern** ของ GoF ต้องมี `Subject` ที่เก็บ list ของ `Observer` แล้ว loop เรียก `.Update()` ของทุกตัวแบบ synchronous ทุกครั้งที่ state เปลี่ยน — ใน Go เรามีเครื่องมือที่เหมาะกับงานนี้ **"โดยกำเนิด"** อยู่แล้ว นั่นคือ **channel** (ทบทวน **Part 037**): ตัว publisher แค่ส่งค่าเข้า channel ส่วนตัว subscriber (goroutine ที่ `range` บน channel ของตัวเอง) จะได้รับการแจ้งเตือนโดยอัตโนมัติ ไม่ต้องเขียน interface `Observer` หรือเรียก `.Update()` เอง:

```go
package main

import (
	"fmt"
	"sync"
)

// PriceUpdate คือ event ที่ subject จะกระจายออกไป
type PriceUpdate struct {
	Symbol string
	Price  float64
}

// PriceTicker คือ "subject" — เก็บ channel ของผู้ subscribe แต่ละคนไว้ แล้ว
// broadcast ค่าที่เปลี่ยนไปให้ทุกคนพร้อมกันเมื่อมีการเรียก Publish
type PriceTicker struct {
	mu          sync.Mutex
	subscribers []chan PriceUpdate
}

func NewPriceTicker() *PriceTicker {
	return &PriceTicker{}
}

// Subscribe คืน channel แบบ receive-only ให้ผู้เรียก — ผู้ subscribe ไม่มีสิทธิ์
// ส่งค่าเข้า channel ของตัวเอง (compiler บังคับผ่าน type <-chan) มีหน้าที่แค่ range
// อ่านค่าเท่านั้น นี่คือการใช้ directional channel (ทบทวน Part 037) เพื่อบังคับ
// ทิศทางการสื่อสารให้ถูกต้องตั้งแต่ระดับ type system
func (t *PriceTicker) Subscribe() <-chan PriceUpdate {
	t.mu.Lock()
	defer t.mu.Unlock()
	ch := make(chan PriceUpdate, 1) // buffer 1 กัน publisher ค้างถ้า subscriber อ่านช้ากว่าเล็กน้อย
	t.subscribers = append(t.subscribers, ch)
	return ch
}

// Publish กระจาย update ไปให้ทุก subscriber พร้อมกัน
func (t *PriceTicker) Publish(update PriceUpdate) {
	t.mu.Lock()
	defer t.mu.Unlock()
	for _, ch := range t.subscribers {
		ch <- update
	}
}

// Close ปิด channel ของทุก subscriber เพื่อบอกว่า "จะไม่มี event เข้ามาอีกแล้ว"
// (ทบทวน Part 037: การปิด channel คือสัญญาณ "จบ" ที่ range รับรู้ได้เอง)
func (t *PriceTicker) Close() {
	t.mu.Lock()
	defer t.mu.Unlock()
	for _, ch := range t.subscribers {
		close(ch)
	}
}

func main() {
	ticker := NewPriceTicker()

	var wg sync.WaitGroup

	// จำลอง observer 2 ตัว: ตัวหนึ่งพิมพ์ทุกราคา อีกตัวหนึ่งพิมพ์เฉพาะตอนราคาเกิน 100
	loggerCh := ticker.Subscribe()
	wg.Add(1)
	go func() {
		defer wg.Done()
		for update := range loggerCh {
			fmt.Printf("  [logger]  %s = %.2f\n", update.Symbol, update.Price)
		}
		fmt.Println("  [logger]  channel ปิดแล้ว, observer จบการทำงาน")
	}()

	alertCh := ticker.Subscribe()
	wg.Add(1)
	go func() {
		defer wg.Done()
		for update := range alertCh {
			if update.Price > 100 {
				fmt.Printf("  [alert]   %s พุ่งเกิน 100! (%.2f)\n", update.Symbol, update.Price)
			}
		}
		fmt.Println("  [alert]   channel ปิดแล้ว, observer จบการทำงาน")
	}()

	fmt.Println("=== Publish ราคาหลายรอบ ===")
	ticker.Publish(PriceUpdate{Symbol: "GOLANG", Price: 85.50})
	ticker.Publish(PriceUpdate{Symbol: "GOLANG", Price: 120.75})
	ticker.Publish(PriceUpdate{Symbol: "GOLANG", Price: 99.00})

	ticker.Close() // ปิด channel ทั้งหมด -> ทุก observer จะออกจาก range loop เอง
	wg.Wait()      // รอให้ observer ทุกตัวประมวลผล event ที่ค้างอยู่และจบให้เรียบร้อยก่อน
}
```

ผลลัพธ์จริง (รันด้วย `go run -race .` เพื่อยืนยันว่าไม่มี data race ด้วย — ทบทวน race detector จาก **Part 044**):

```
=== Publish ราคาหลายรอบ ===
  [logger]  GOLANG = 85.50
  [logger]  GOLANG = 120.75
  [logger]  GOLANG = 99.00
  [logger]  channel ปิดแล้ว, observer จบการทำงาน
  [alert]   GOLANG พุ่งเกิน 100! (120.75)
  [alert]   channel ปิดแล้ว, observer จบการทำงาน
```

เทียบกับ Observer pattern แบบดั้งเดิมที่ต้องมี:
```
interface Observer { void update(Event e); }
interface Subject { void subscribe(Observer o); void unsubscribe(Observer o); void notify(Event e); }
```
โค้ด Go ข้างบนไม่ต้องมี interface `Observer` เลยแม้แต่ตัวเดียว — **channel เป็น "observer" ที่มากับภาษาโดยตรง** และยังได้ความสามารถเพิ่มเติมที่ pattern ดั้งเดิมไม่มีให้ฟรีๆ: **backpressure** (ถ้า subscriber อ่านช้า channel ที่ไม่มี buffer จะทำให้ publisher รอ แทนที่จะยัด event ท่วมจน memory ระเบิด), **การจบแบบสง่างาม** ผ่านการ `close()` channel ที่ทุก subscriber รับรู้ได้พร้อมกันทันที และ **type safety** เต็มรูปแบบผ่าน generic channel type

---

## 6. Singleton: `sync.Once` และคำเตือนเรื่อง Overuse

**Singleton Pattern** รับประกันว่า class หนึ่งมี instance เดียวตลอดอายุโปรแกรม ในภาษา OOP ดั้งเดิมมักทำผ่าน private constructor + static method ที่เช็ค `if instance == null` — ปัญหาคือแบบนี้ **ไม่ปลอดภัยเลยเมื่อมีหลาย thread เรียกพร้อมกัน** (race condition แบบ check-then-act ที่สอง thread อาจเห็น `null` พร้อมกันแล้วสร้างซ้อนกัน)

ใน Go เราใช้ **`sync.Once`** (ทบทวนเจาะลึกจาก **Part 039 หัวข้อ 8**) ซึ่งรับประกันว่าโค้ดข้างในจะรันแค่ครั้งเดียวตลอดอายุโปรแกรม แม้เรียกจากหลาย goroutine พร้อมกันก็ตาม:

```go
package main

import (
	"fmt"
	"sync"
)

// Config จำลอง resource ที่ "แพง" ในการสร้าง (เช่น อ่านไฟล์ config, เชื่อมต่อ
// ฐานข้อมูล, โหลด certificate) ซึ่งควรสร้างแค่ครั้งเดียวแล้วใช้ซ้ำตลอดอายุโปรแกรม
type Config struct {
	APIKey string
}

var (
	configInstance *Config
	configOnce     sync.Once
)

// GetConfig คือจุดเข้าถึง singleton จุดเดียวของทั้งโปรแกรม — ไม่ว่าจะเรียกจาก
// ที่ไหนกี่ครั้งก็ตาม จะได้ pointer ตัวเดียวกันเสมอ และ "กำลังสร้าง config..."
// จะถูกพิมพ์แค่ครั้งเดียวเท่านั้น
func GetConfig() *Config {
	configOnce.Do(func() {
		fmt.Println("  กำลังสร้าง config... (ควรเห็นข้อความนี้แค่ครั้งเดียว)")
		configInstance = &Config{APIKey: "secret-key-12345"}
	})
	return configInstance
}

func main() {
	fmt.Println("=== เรียก GetConfig() จาก 10 goroutine พร้อมกัน ===")
	var wg sync.WaitGroup
	results := make([]*Config, 10)

	for i := 0; i < 10; i++ {
		wg.Add(1)
		go func(idx int) {
			defer wg.Done()
			results[idx] = GetConfig()
		}(i)
	}
	wg.Wait()

	allSame := true
	for _, cfg := range results {
		if cfg != results[0] {
			allSame = false
		}
	}
	fmt.Printf("ทุก goroutine ได้ pointer ตัวเดียวกันหรือไม่: %v\n", allSame)
	fmt.Printf("APIKey: %s\n\n", results[0].APIKey)
}
```

ผลลัพธ์จริง (รันด้วย `go run -race .` — ยืนยันว่าไม่มี data race):

```
=== เรียก GetConfig() จาก 10 goroutine พร้อมกัน ===
  กำลังสร้าง config... (ควรเห็นข้อความนี้แค่ครั้งเดียว)
ทุก goroutine ได้ pointer ตัวเดียวกันหรือไม่: true
APIKey: secret-key-12345
```

ข้อความ `"กำลังสร้าง config..."` ถูกพิมพ์แค่ครั้งเดียวแม้เรียกจาก 10 goroutine พร้อมกัน — `sync.Once` รับประกันเรื่องนี้ให้อัตโนมัติผ่าน atomic operation และ mutex ที่ผสมกันอยู่ภายใน (ทบทวนกลไกเบื้องหลังจาก **Part 039**)

### คำเตือนเรื่อง Overuse: Singleton ทำให้ Test ยากขึ้น

แม้ `sync.Once` จะแก้ปัญหา thread-safety ได้สมบูรณ์ แต่ Singleton Pattern เองก็มี**ข้อเสียเชิงโครงสร้าง**ที่ยังคงอยู่ไม่ว่าจะ implement ด้วยกลไกไหน:

- **Global state ที่ถูกแชร์ข้าม test case ทั้งหมดในโปรเซสเดียวกัน** — ถ้า test หนึ่งเปลี่ยนแปลงพฤติกรรมของ singleton (เช่น mock ค่า config) test ตัวอื่นที่รันหลังจากนั้นอาจได้รับผลกระทบโดยไม่ตั้งใจ ทำให้ test มีลำดับการรันที่มีผลต่อผลลัพธ์ (test order dependency) ซึ่งเป็นกลิ่นของโค้ดที่ไม่ดี
- **ซ่อน dependency ที่แท้จริงของฟังก์ชัน** — ฟังก์ชันที่เรียก `GetConfig()` ข้างในตรงๆ ดูจาก signature ภายนอกแล้วไม่มีทางรู้เลยว่ามันต้องพึ่งพา config ตัวไหน ต่างจากการรับ `*Config` เป็นพารามิเตอร์ที่เห็น dependency ชัดเจนจาก signature ทันที
- **แทนที่ (substitute) เพื่อ test ยาก** — อยากทดสอบพฤติกรรมตอน config ผิดพลาด (เช่น API key หมดอายุ) ต้องหาทางแฮ็ค global state ซึ่งมักไม่สวยงามและเสี่ยงกระทบ test อื่น

แนวทางที่นักพัฒนาอาวุโสจำนวนมากแนะนำในโค้ด production: **ส่ง `*Config` เข้าไปเป็น dependency ผ่าน constructor ตรงๆ** (explicit dependency injection แบบที่ใช้ตลอดใน **Part 100**) แล้วสร้าง instance เดียวใน `main()` แค่ที่เดียว แทนที่จะให้ทุก package เข้าถึง global singleton ได้ตามใจชอบ — `sync.Once` ยังคงมีประโยชน์จริงสำหรับกรณีที่จำกัดมากขึ้น เช่น lazy-loading ค่าคงที่ที่คำนวณแพงจริงๆ (regex ที่ compile ครั้งเดียว, connection pool ที่สร้างครั้งเดียวใน `main()` เอง) แต่ควรหลีกเลี่ยงการใช้เป็น "ทางลัด" แทน dependency injection ที่ถูกต้อง

---

## 7. Pattern ที่ Go ทำให้ไม่จำเป็น: Factory Hierarchy และ Visitor

GoF pattern บางตัวมีอยู่เพื่อ**แก้ปัญหาที่เกิดจากข้อจำกัดของภาษา class-based inheritance โดยเฉพาะ** — เมื่อ Go ไม่มีข้อจำกัดนั้นตั้งแต่ต้น pattern เหล่านั้นก็ไม่จำเป็นอีกต่อไป

### Factory Method / Abstract Factory Hierarchy

ในภาษา Java การจะสร้าง object ที่ "ชนิดขึ้นอยู่กับเงื่อนไข runtime" โดยไม่ผูก client เข้ากับ concrete class ตรงๆ ต้องสร้าง hierarchy ของ Factory class เต็มรูปแบบ (`ShapeFactory`, `AbstractShapeFactory`, `ConcreteShapeFactoryA`, ...) เพราะ Java ไม่มีทางส่ง "constructor" เป็นค่าตรงๆ ได้

ใน Go เพราะ **function เป็น first-class citizen** เราแค่ใช้ `map[string]func() Shape` หรือฟังก์ชันธรรมดาที่ return ตาม type switch ก็เพียงพอแล้ว ไม่ต้องมี factory hierarchy ใดๆ เลย:

```go
type Shape interface{ Area() float64 }

// ไม่ต้องมี "AbstractShapeFactory" — ใช้ map ของ constructor function ตรงๆ
var shapeFactories = map[string]func() Shape{
	"circle":    func() Shape { return &Circle{Radius: 1} },
	"rectangle": func() Shape { return &Rectangle{Width: 1, Height: 1} },
}

func CreateShape(kind string) (Shape, error) {
	factory, ok := shapeFactories[kind]
	if !ok {
		return nil, fmt.Errorf("unknown shape: %s", kind)
	}
	return factory(), nil
}
```

นี่ไม่ใช่ "Factory Pattern แบบง่าย" แต่เป็น**การตัดความจำเป็นของ hierarchy ทั้งชุดออกไปเลย** เพราะปัญหาที่ Factory hierarchy พยายามแก้ (การส่ง "วิธีสร้าง object" เป็นค่าได้โดยไม่ผูกกับ concrete class) ถูกแก้ไปแล้วโดย function-as-value ตั้งแต่ระดับภาษา

### Visitor Pattern

**Visitor Pattern** ของ GoF มีไว้แก้ปัญหา "อยากเพิ่ม operation ใหม่ให้ object hierarchy ที่มีอยู่แล้ว โดยไม่ต้องแก้ไข class เดิมทุกตัว" (double dispatch ผ่าน `Accept(Visitor)` + `Visit(ConcreteType)` คู่กัน) — ปัญหานี้เกิดหนักในภาษาที่ inheritance ทำให้แก้ไข class ฐานกระทบทุก subclass และไม่มีทางเพิ่ม behavior จากภายนอกได้เลยถ้าไม่แก้ source code ต้นทาง

Go แก้ปัญหานี้ได้เรียบง่ายกว่ามากด้วยสองแนวทางที่เรียนมาแล้วในหลักสูตรนี้:

1. **Type switch** (ทบทวนจาก **Part 014**) — ถ้ามี type จำกัดจำนวนไม่มาก การเขียน `switch v := shape.(type) { case *Circle: ...; case *Rectangle: ... }` ตรงไปตรงมากว่า Visitor pattern มาก และเพิ่ม operation ใหม่ทำได้แค่เขียนฟังก์ชันใหม่ที่มี type switch ของตัวเอง ไม่ต้องแก้ type เดิมเลย
2. **Generics + constraints** (ทบทวนจาก **Part 028–029**) — เมื่อ operation ที่ต้องการทำงานได้กับหลาย type ที่มีโครงสร้างคล้ายกัน generic function ที่มี type constraint ที่เหมาะสมมักทดแทน Visitor pattern ได้ตรงไปตรงมากว่า โดยได้ type safety เต็มรูปแบบตอน compile time เหมือนกัน

```go
// แทนที่จะมี Visitor interface + Accept/Visit คู่กันเต็มรูปแบบ
// ใช้ type switch ธรรมดาก็เพียงพอสำหรับเพิ่ม "operation ใหม่" โดยไม่แก้ type เดิม
func Describe(s Shape) string {
	switch v := s.(type) {
	case *Circle:
		return fmt.Sprintf("วงกลมรัศมี %.1f", v.Radius)
	case *Rectangle:
		return fmt.Sprintf("สี่เหลี่ยม %.1fx%.1f", v.Width, v.Height)
	default:
		return "รูปทรงไม่รู้จัก"
	}
}
```

**บทสรุปของหัวข้อนี้**: การที่ Go ไม่มี pattern เหล่านี้ให้ใช้ตรงๆ ไม่ใช่ข้อบกพร่อง — มันคือสัญญาณว่า **ปัญหาที่ pattern เหล่านั้นแก้ไขไม่มีอยู่จริงในภาษา Go ตั้งแต่แรก** นี่คือตัวอย่างที่ดีของปรัชญา "less is more" ที่กล่าวถึงตั้งแต่ **Part 001**: ภาษาที่มี feature น้อยกว่าแต่ compose กันได้ดี บางทีก็ทำให้ต้องมี "เทคนิคพิเศษ" น้อยกว่าภาษาที่มี feature เยอะแต่ต้องแก้ปัญหาที่ feature ตัวเองสร้างขึ้นมา

---

## 8. ตารางสรุป: Pattern ไหนควรใช้ Pattern ไหนควรเลี่ยง

| GoF Pattern | สถานะใน Go | วิธี Implement ที่แนะนำ |
|---|---|---|
| **Functional Options** (ไม่ใช่ GoF ดั้งเดิม แต่เทียบเท่า Builder) | นิยมใช้มาก | `type Option func(*T)` + `opts ...Option` |
| **Strategy** | ยังมีประโยชน์ | interface เล็กๆ หรือ function type ตรงๆ |
| **Decorator** | ยังมีประโยชน์มาก | ห่อ interface (โดยเฉพาะ small interface) ด้วย struct ที่ implement interface เดียวกัน |
| **Observer** | แทนที่ด้วยกลไกภาษา | channel + goroutine (ทบทวน Part 037) |
| **Singleton** | ใช้ได้แต่ระวัง overuse | `sync.Once` — แต่พิจารณา explicit DI ก่อนเสมอ |
| **Factory Method / Abstract Factory** | ไม่จำเป็นในรูปแบบ hierarchy เต็ม | function-as-value, `map[string]func() T` |
| **Visitor** | ไม่จำเป็น | type switch (Part 014) หรือ generics (Part 028–029) |
| **Adapter** | ยังมีประโยชน์ | struct ที่ implement interface เป้าหมาย ห่อ type ที่ไม่ตรง interface |
| **Template Method** | ทำได้ต่างวิธี | struct embedding (Part 030) หรือส่ง function เป็นพารามิเตอร์ |
| **Command** | ยังมีประโยชน์ | function type ตรงๆ (`type Command func() error`) มักเพียงพอโดยไม่ต้องมี interface |

หลักการเลือกใช้ pattern ใน Go ที่สรุปได้จากทั้งบท: **เริ่มจากคำถามว่า "ปัญหานี้แก้ด้วย function ตรงๆ ได้ไหม" ก่อนเสมอ** ถ้าได้ ให้ใช้ function เพราะเรียบง่ายที่สุด ถ้าต้องการ state หรือ configuration ที่ซับซ้อนกว่านั้นค่อยขยับไปใช้ interface + struct และสงวน pattern ที่ซับซ้อนกว่านั้น (เช่น Visitor เต็มรูปแบบ) ไว้เฉพาะกรณีที่ type switch หรือ generics ไม่เพียงพอจริงๆ ซึ่งพบได้น้อยมากในทางปฏิบัติ

---

## สรุปสิ่งที่ได้เรียนในบทนี้

- GoF design pattern ทั้ง 23 แบบถูกออกแบบมาสำหรับภาษา class-based inheritance — Go ที่ไม่มี inheritance, มี first-class function, และมี implicit interface satisfaction ทำให้หลาย pattern เรียบง่ายขึ้นมากหรือไม่จำเป็นอีกต่อไป
- **Functional Options** (`type Option func(*T)`) คือทางเลือกที่ Go นิยมแทน Builder pattern และ telescoping constructor — backward compatible เสมอเมื่อเพิ่ม option ใหม่
- **Strategy Pattern** ทำได้ผ่าน interface เล็กๆ หรือ function type ตรงๆ (เหมือน `sort.Slice`) โดยไม่ต้องมี class hierarchy
- **Decorator Pattern** คือหลักการเดียวกับที่เห็นใน `io.Reader` (Part 048) และ HTTP middleware (Part 057) — ห่อ interface ด้วย struct ที่ implement interface เดียวกัน ซ้อนกันได้หลายชั้น
- **Observer Pattern** ถูกแทนที่ด้วย channel + goroutine อย่างเป็นธรรมชาติ ได้ backpressure, graceful shutdown ผ่าน `close()`, และ type safety มาฟรี
- **Singleton** ทำผ่าน `sync.Once` เพื่อความปลอดภัยจาก race condition แต่ควรระวัง overuse — global state ทำให้ test ยากและซ่อน dependency ที่แท้จริง ควรพิจารณา explicit dependency injection ก่อนเสมอ
- **Factory Hierarchy และ Visitor** ไม่จำเป็นในรูปแบบเต็มของ GoF อีกต่อไป เพราะ function-as-value, type switch (Part 014), และ generics (Part 028–029) แก้ปัญหาเดียวกันได้เรียบง่ายกว่า

## แบบฝึกหัดท้ายบท

1. ทำตามตัวอย่างในบทนี้ทั้ง 5 pattern ให้ครบ แล้วรัน `go run .` (และ `go run -race .` สำหรับตัวอย่าง Observer กับ Singleton) ด้วยตัวเอง ยืนยันว่าได้ output ตรงกับที่แสดงในบทเรียน
2. เพิ่ม `Option` ใหม่ชื่อ `WithLogger(logger func(string))` เข้าไปในตัวอย่าง Functional Options แล้วเรียกใช้ logger นั้นตอนสร้าง `Server` เสร็จ (พิมพ์ข้อความสรุป config ที่ตั้งค่าไว้)
3. เขียน Strategy pattern ของตัวเองสำหรับ "วิธีการจัดส่งสินค้า" (เช่น `StandardShipping`, `ExpressShipping`, `SameDayShipping`) แต่ละแบบคำนวณค่าส่งและเวลาที่ใช้ต่างกัน ทดสอบด้วยฟังก์ชันที่รับ `[]ShippingStrategy` แล้วพิมพ์ตัวเลือกทั้งหมดพร้อมราคา
4. เพิ่ม decorator ตัวที่สามในตัวอย่าง `io.Reader` ชื่อ `ThrottledReader` ที่จำกัดความเร็วการอ่าน (ใช้ `time.Sleep` สั้นๆ ระหว่างการอ่านแต่ละครั้ง) แล้วห่อซ้อนเข้ากับ `CountingReader` และ `UpperCaseReader` เดิม ทดสอบว่ายังทำงานถูกต้องเมื่อห่อ 3 ชั้น
5. ดัดแปลงตัวอย่าง Observer ให้ subscriber แต่ละตัว unsubscribe ตัวเองได้ (เพิ่ม method `Unsubscribe(ch <-chan PriceUpdate)` ให้ `PriceTicker`) โดยยังคง safe จาก data race เมื่อทดสอบด้วย `go run -race`
6. อภิปราย (เตรียมคำตอบไว้): เลือก GoF pattern อีก 3 แบบที่ไม่ได้กล่าวถึงในบทนี้ (เช่น State, Chain of Responsibility, Composite) แล้ววิเคราะห์ว่าถ้าจะ implement ใน Go จะใช้กลไกอะไรของภาษา (interface, function type, channel, generics) และมันเรียบง่ายกว่าเวอร์ชัน Java/C++ ดั้งเดิมอย่างไรบ้าง

---

**ต่อไป**: [Part 103 — โปรเจกต์: E-Commerce REST API แบบเต็มรูปแบบ](./103-project-ecommerce-api.md)
