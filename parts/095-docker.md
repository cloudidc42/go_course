# Part 095: Docker กับ Go Application

> ภาคที่ 9: DevOps และ Deployment — ตอนที่ 1 จาก 5 (Part 95–99)

## สารบัญของบทนี้

1. ทำไม Go ถึงเป็นภาษาที่ "เกิดมาเพื่อ Docker"
2. Docker พื้นฐานที่ต้องรู้ก่อนไปต่อ: Image, Container, Layer
3. เขียน API ตัวอย่าง: ต่อยอดจาก Part 060 ให้พร้อมสำหรับ Container
4. Dockerfile แบบไร้เดียงสา (Naive) และปัญหาที่ตามมา
5. Multi-Stage Build: แยก "ตอนคอมไพล์" ออกจาก "ตอนรัน"
6. `CGO_ENABLED=0`: กุญแจสำคัญของ Static Binary
7. `scratch` vs `distroless`: final stage เล็กที่สุดสองแบบ
8. เทียบขนาด Image จริง: Naive vs Scratch vs Distroless
9. `.dockerignore`: กันไฟล์ที่ไม่ควรหลุดเข้า build context
10. `HEALTHCHECK` เมื่อไม่มี Shell ให้ใช้
11. Build และ Run จริงด้วย Docker บนเครื่องที่เขียนบทความนี้
12. แนวทางเลือก Base Image และข้อผิดพลาดที่พบบ่อย
13. หมายเหตุความซื่อสัตย์เรื่องการรันจริงในบทนี้
14. สรุปสิ่งที่ได้เรียนในบทนี้
15. แบบฝึกหัดท้ายบท

---

## 1. ทำไม Go ถึงเป็นภาษาที่ "เกิดมาเพื่อ Docker"

ย้อนกลับไป **Part 001** เราพูดถึงจุดเด่นของ Go ไว้ข้อหนึ่งว่า **"Binary เดียวจบ — คอมไพล์แล้วได้ไฟล์ executable เดียว ไม่ต้องติดตั้ง runtime หรือ dependency เพิ่ม"** ข้อเท็จจริงนี้ไม่ใช่แค่ความสะดวกเฉยๆ แต่เป็นเหตุผลสำคัญที่ทำให้ Go กลายเป็นภาษาอันดับต้นๆ ที่ใช้เขียนเครื่องมือ infrastructure ระดับโลก — **Docker เองก็เขียนด้วย Go ทั้งหมด และ Kubernetes ก็เช่นกัน** (ตามที่กล่าวถึงใน Part 001 หัวข้อ 3)

เทียบกับภาษาอื่นที่ต้องมี "ตัวแปลภาษา" (runtime) ติดไปกับ container เสมอ:

| ภาษา | ต้องมีอะไรใน container ตอนรัน |
|---|---|
| Python | ต้องมี Python interpreter + pip packages ทั้งหมด |
| Node.js | ต้องมี Node.js runtime + `node_modules` ทั้งหมด |
| Java | ต้องมี JVM (มักหนักหลายร้อย MB) |
| **Go** | **ไม่ต้องมีอะไรเลย** — ได้ native machine code ที่ OS เรียกรันได้ตรงๆ |

เพราะ Go compile เป็น **native machine code** ที่ผูก (link) ทุกอย่างที่โปรแกรมต้องใช้เข้าไว้ในไฟล์เดียวตั้งแต่ตอน `go build` (ต่างจากภาษาที่ต้อง interpret หรือรันบน virtual machine) เราจึงสามารถสร้าง Docker image ที่ **เล็กที่สุดเท่าที่ระบบปฏิบัติการจะอนุญาตให้มี container ได้** — เล็กจนแทบไม่มีอะไรอยู่ในนั้นเลยนอกจากตัวโปรแกรมเอง นี่คือหัวใจของบทนี้

---

## 2. Docker พื้นฐานที่ต้องรู้ก่อนไปต่อ: Image, Container, Layer

สำหรับผู้ที่ยังไม่เคยใช้ Docker มาก่อน สรุปแนวคิดหลัก 3 อย่างที่ต้องเข้าใจก่อน:

- **Image** คือ "แม่แบบ" ที่อ่านอย่างเดียว (read-only) บรรจุทุกอย่างที่โปรแกรมต้องใช้ในการรัน (ไฟล์ binary, library, config) สร้างจากไฟล์คำสั่งที่ชื่อ **`Dockerfile`**
- **Container** คือ "instance ที่กำลังรันอยู่จริง" ของ image หนึ่งตัว เปรียบเทียบได้กับความสัมพันธ์ระหว่าง **class กับ object** ในภาษาเชิงวัตถุ (หรือใน Go คือ struct type กับตัวแปรที่สร้างจาก type นั้น) — image เดียวสร้าง container ได้พร้อมกันหลายตัว
- **Layer** คือ Docker image ถูกสร้างเป็นชั้นๆ ซ้อนกัน แต่ละคำสั่งใน Dockerfile (เช่น `COPY`, `RUN`) สร้าง layer ใหม่ 1 ชั้น Docker **cache แต่ละ layer ไว้** ถ้า layer ไหนไม่เปลี่ยน (เช่น dependency ไม่เปลี่ยน) การ build รอบถัดไปจะข้าม layer นั้นไปเลยโดยไม่ต้องทำซ้ำ — นี่คือเหตุผลที่ **ลำดับคำสั่งใน Dockerfile มีผลต่อความเร็วในการ build อย่างมาก**

คำสั่งพื้นฐานที่ใช้บ่อยที่สุดตลอดบทนี้:

```bash
docker build -t <ชื่อimage>:<tag> .   # สร้าง image จาก Dockerfile ในโฟลเดอร์ปัจจุบัน
docker images                          # ดูรายการ image ทั้งหมดที่มีในเครื่อง พร้อมขนาด
docker run -d -p 8080:8080 <image>     # รัน container จาก image แบบ background (-d)
docker ps                              # ดู container ที่กำลังรันอยู่
docker logs <container>                # ดู log ของ container
docker exec -it <container> sh         # เข้าไปที่ shell ข้างใน container (ถ้ามี shell)
docker stop/rm <container>             # หยุด/ลบ container
```

> **หมายเหตุสภาพแวดล้อม**: บทนี้เขียนและรันคำสั่ง Docker จริงบนเครื่องที่ใช้เขียนหลักสูตร ซึ่งมี **Docker Engine 29.3.1** และ **Docker Compose v5.1.1** ติดตั้งพร้อมใช้งานอยู่แล้ว ผลลัพธ์ทุกอย่างที่แสดงในบทนี้ (ขนาด image, log, สถานะ container) คือผลลัพธ์จริงจากการรันคำสั่งเหล่านี้ ไม่ใช่ค่าที่แต่งขึ้น

---

## 3. เขียน API ตัวอย่าง: ต่อยอดจาก Part 060 ให้พร้อมสำหรับ Container

ก่อนจะพูดเรื่อง Docker เราต้องมี Go application ก่อน บทนี้ใช้ API เล็กๆ สไตล์เดียวกับ Todo API ใน **Part 060** (ใช้ `respondJSON` helper แบบเดียวกัน) แต่เพิ่มสิ่งที่จำเป็นสำหรับการรันใน container: **graceful shutdown** (ทบทวนจาก **Part 047**), **structured logging ด้วย `slog`** (ทบทวนจาก **Part 054**), และ endpoint **`/healthz`** สำหรับ health check

```go
package main

import (
	"context"
	"encoding/json"
	"errors"
	"fmt"
	"log/slog"
	"net/http"
	"os"
	"os/signal"
	"sync"
	"syscall"
	"time"
)

// Task คือ resource ง่ายๆ ของ API ตัวอย่างนี้ (สไตล์เดียวกับ Todo API ใน Part 060)
type Task struct {
	ID   int    `json:"id"`
	Name string `json:"name"`
	Done bool   `json:"done"`
}

type store struct {
	mu     sync.Mutex
	nextID int
	tasks  map[int]Task
}

func newStore() *store {
	return &store{nextID: 1, tasks: make(map[int]Task)}
}

func (s *store) create(name string) Task {
	s.mu.Lock()
	defer s.mu.Unlock()
	t := Task{ID: s.nextID, Name: name}
	s.tasks[t.ID] = t
	s.nextID++
	return t
}

func (s *store) list() []Task {
	s.mu.Lock()
	defer s.mu.Unlock()
	out := make([]Task, 0, len(s.tasks))
	for _, t := range s.tasks {
		out = append(out, t)
	}
	return out
}

func respondJSON(w http.ResponseWriter, status int, v any) {
	w.Header().Set("Content-Type", "application/json; charset=utf-8")
	w.WriteHeader(status)
	if v != nil {
		_ = json.NewEncoder(w).Encode(v)
	}
}

// runHealthcheck คือโหมดพิเศษของ binary ตัวเอง ใช้เป็นคำสั่งของ Docker HEALTHCHECK
// จำเป็นเพราะ image แบบ scratch/distroless ไม่มี curl/wget/shell ให้เรียกจากภายนอก
// จึงให้ binary ตัวเองยิง HTTP request เข้า /healthz แล้วคืน exit code ให้ Docker อ่านแทน
// (อธิบายเหตุผลแบบเต็มในหัวข้อ 10)
func runHealthcheck() {
	port := os.Getenv("PORT")
	if port == "" {
		port = "8080"
	}
	client := http.Client{Timeout: 2 * time.Second}
	resp, err := client.Get("http://127.0.0.1:" + port + "/healthz")
	if err != nil || resp.StatusCode != http.StatusOK {
		os.Exit(1)
	}
	_ = resp.Body.Close()
	os.Exit(0)
}

func main() {
	if len(os.Args) > 1 && os.Args[1] == "-healthcheck" {
		runHealthcheck()
		return
	}

	logger := slog.New(slog.NewJSONHandler(os.Stdout, nil))
	slog.SetDefault(logger)

	s := newStore()
	s.create("เขียนบท Docker")
	s.create("เขียนบท Compose")

	mux := http.NewServeMux()

	// /healthz คือ endpoint สำหรับ Docker HEALTHCHECK และ Kubernetes probe ใน Part 097
	mux.HandleFunc("GET /healthz", func(w http.ResponseWriter, r *http.Request) {
		respondJSON(w, http.StatusOK, map[string]string{"status": "ok"})
	})

	mux.HandleFunc("GET /tasks", func(w http.ResponseWriter, r *http.Request) {
		respondJSON(w, http.StatusOK, s.list())
	})

	mux.HandleFunc("POST /tasks", func(w http.ResponseWriter, r *http.Request) {
		var body struct {
			Name string `json:"name"`
		}
		if err := json.NewDecoder(r.Body).Decode(&body); err != nil || body.Name == "" {
			respondJSON(w, http.StatusBadRequest, map[string]string{"error": "invalid body"})
			return
		}
		t := s.create(body.Name)
		respondJSON(w, http.StatusCreated, t)
	})

	port := os.Getenv("PORT")
	if port == "" {
		port = "8080"
	}

	srv := &http.Server{
		Addr:         ":" + port,
		Handler:      mux,
		ReadTimeout:  5 * time.Second,
		WriteTimeout: 10 * time.Second,
		IdleTimeout:  120 * time.Second,
	}

	serverErr := make(chan error, 1)
	go func() {
		slog.Info("server starting", "port", port)
		serverErr <- srv.ListenAndServe()
	}()

	quit := make(chan os.Signal, 1)
	signal.Notify(quit, syscall.SIGINT, syscall.SIGTERM)

	select {
	case err := <-serverErr:
		if err != nil && !errors.Is(err, http.ErrServerClosed) {
			slog.Error("server error", "error", err)
			os.Exit(1)
		}
	case sig := <-quit:
		slog.Info("shutdown signal received", "signal", sig.String())
		ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
		defer cancel()
		if err := srv.Shutdown(ctx); err != nil {
			slog.Error("graceful shutdown failed", "error", err)
		} else {
			slog.Info("server stopped gracefully")
		}
	}
	fmt.Println("bye")
}
```

**ทำไม `PORT` อ่านจาก environment variable แทนที่จะ hardcode `:8080`**: นี่คือธรรมเนียมมาตรฐานของแอปที่ออกแบบมาให้รันใน container ตาม [12-Factor App](https://12factor.net/) — container orchestrator (Docker, Kubernetes, Cloud Run ฯลฯ) มักกำหนด port ที่แอปต้อง listen ผ่าน environment variable แทนที่จะให้แอป hardcode พอร์ตตายตัว ทำให้ image เดียวกันรันได้ในหลายสภาพแวดล้อมโดยไม่ต้องแก้โค้ดหรือ build ใหม่

รันทดสอบในเครื่องก่อน (ยังไม่ยุ่งกับ Docker เลย) เพื่อยืนยันว่าโค้ดถูกต้อง:

```bash
go build -o server .
./server &
curl -s http://localhost:8080/healthz
curl -s http://localhost:8080/tasks
curl -s -X POST http://localhost:8080/tasks -d '{"name":"ทดสอบ Docker"}'
```

**ผลลัพธ์จริงจากการรัน**:

```
{"time":"2026-09-26T03:39:56.36Z","level":"INFO","msg":"server starting","port":"8080"}
{"status":"ok"}
[{"id":1,"name":"เขียนบท Docker","done":false},{"id":2,"name":"เขียนบท Compose","done":false}]
{"id":3,"name":"ทดสอบ Docker","done":false}
```

---

## 4. Dockerfile แบบไร้เดียงสา (Naive) และปัญหาที่ตามมา

นักพัฒนาที่เพิ่งเริ่มใช้ Docker กับ Go มักเขียน Dockerfile แบบตรงไปตรงมาที่สุดแบบนี้:

```dockerfile
# Dockerfile.naive - วิธีไร้เดียงสา: ใช้ image golang เต็มรูปแบบทั้ง build และ run
FROM golang:1.24-alpine
WORKDIR /app
COPY go.mod ./
RUN go mod download
COPY . .
RUN go build -o /app/server .
EXPOSE 8080
CMD ["/app/server"]
```

โค้ดนี้**ทำงานได้ถูกต้อง** — แต่มีปัญหาใหญ่ที่มองไม่เห็นจนกว่าจะเช็คขนาด image: **image สุดท้ายพก Go toolchain ทั้งชุด (compiler, standard library ทั้งหมด, เครื่องมือ build) ติดไปด้วย** ทั้งที่ตอน**รัน**จริงเราต้องการแค่ไฟล์ binary ที่ compile เสร็จแล้วไฟล์เดียวเท่านั้น เหมือนสั่งเดลิเวอรี่อาหารแล้วให้ครัวทั้งครัวติดมาส่งที่บ้านด้วย

นี่คือปัญหาคลาสสิกที่ **Multi-Stage Build** ถูกออกแบบมาเพื่อแก้โดยเฉพาะ

---

## 5. Multi-Stage Build: แยก "ตอนคอมไพล์" ออกจาก "ตอนรัน"

แนวคิดของ multi-stage build คือ **ใช้ Dockerfile เดียว แต่มีหลาย `FROM` ซ้อนกัน** แต่ละ `FROM` เริ่ม "stage" ใหม่ โดย stage หลังๆ สามารถ **คัดลอกเฉพาะไฟล์ที่ต้องการ** จาก stage ก่อนหน้าด้วยคำสั่ง `COPY --from=<stage>` ได้ ส่วนไฟล์อื่นที่เหลือใน stage ก่อนหน้า (รวมถึง Go toolchain ทั้งชุด) **จะไม่ถูกนำไปใส่ใน image สุดท้ายเลย**

```dockerfile
# Stage 1: "builder" - มี Go toolchain เต็มรูปแบบ ใช้คอมไพล์อย่างเดียว
# stage นี้จะไม่ถูก ship ไปกับ image จริงที่ deploy เลยแม้แต่ byte เดียว
FROM golang:1.24-alpine AS builder
WORKDIR /src
COPY go.mod ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -ldflags="-s -w" -o /out/server .

# Stage 2: "final" - scratch คือ image เปล่าที่สุดเท่าที่มี ไม่มี OS ไม่มี shell ไม่มีอะไรเลย
# นอกจาก binary ที่เราคัดลอกมาจาก stage builder ด้วย COPY --from=builder
FROM scratch
COPY --from=builder /out/server /server
EXPOSE 8080
ENTRYPOINT ["/server"]
```

จุดสำคัญที่ต้องสังเกต:

- **`AS builder`** ตั้งชื่อ stage แรกว่า `builder` เพื่อให้ stage หลังอ้างอิงถึงได้
- **`COPY --from=builder /out/server /server`** คัดลอกเฉพาะไฟล์ binary ที่ compile เสร็จแล้วออกมา — ไม่มีการคัดลอก Go compiler, source code, หรือ cache ของ `go mod download` มาด้วยเลย
- **`FROM scratch`** เริ่ม stage ใหม่จาก image ที่ **ว่างเปล่าโดยสมบูรณ์** (ไม่มี layer ใดๆ อยู่ก่อนเลย) — เป็นไปได้เพราะ Go binary ไม่ต้องพึ่งอะไรจากระบบปฏิบัติการเลยนอกจาก kernel syscall เท่านั้น (เงื่อนไขสำคัญคือต้องเป็น **static binary** ตามหัวข้อถัดไป)
- **`-ldflags="-s -w"`** ตัด debug symbol (`-s`) และ DWARF debugging information (`-w`) ออกจาก binary ทำให้ไฟล์เล็กลงอีก (ไม่มีผลต่อการทำงานปกติ แต่ debug ด้วย `dlv` ตามที่เรียนใน **Part 087** จะทำไม่ได้กับ binary ที่ตัด symbol เหล่านี้ออก — ถ้าต้องการ debug production ให้ตัด flag นี้ออก หรือ build image แยกสำหรับ debug)

---

## 6. `CGO_ENABLED=0`: กุญแจสำคัญของ Static Binary

ย้อนกลับไปดูตาราง environment variable ของ Go ใน **Part 001** จะเห็นแถวนี้:

| ตัวแปร | ความหมาย | ค่า default |
|---|---|---|
| `CGO_ENABLED` | เปิด/ปิดการใช้ C code ผ่าน cgo | `1` |

ค่า default ของ `CGO_ENABLED` คือ `1` (เปิด) ซึ่งมีผลสำคัญที่มือใหม่มักไม่รู้: **แม้โค้ด Go ของเราจะไม่ได้เขียน cgo เองเลยสักบรรทัด แต่ package `net` ในมาตรฐาน (ที่ `net/http` ใช้ข้างใน) จะใช้ตัว resolver ของระบบปฏิบัติการผ่าน cgo โดยอัตโนมัติเมื่อ `CGO_ENABLED=1` และมี C library ให้ link ได้** ผลคือ binary ที่ได้จะกลายเป็น **dynamically linked** คือผูกกับ `libc` ของระบบที่ compile อยู่ ซึ่งจะไม่มีให้ link เลยใน image แบบ `scratch` (ที่ไม่มี libc ติดมาด้วย)

มาพิสูจน์ด้วยคำสั่งจริงบนเครื่องที่เขียนบทความนี้:

```bash
go build -o server_default .              # ใช้ค่า default (CGO_ENABLED=1 บนเครื่องนี้)
CGO_ENABLED=0 go build -o server_static .  # บังคับปิด cgo อย่างชัดเจน
ldd server_default
ldd server_static
```

**ผลลัพธ์จริง**:

```
$ ldd server_default
	linux-vdso.so.1 (0x00007fb776494000)
	libc.so.6 => /lib/x86_64-linux-gnu/libc.so.6 (0x00007fb776200000)
	/lib64/ld-linux-x86-64.so.2 (0x00007fb776496000)

$ ldd server_static
	not a dynamic executable
```

เห็นความแตกต่างชัดเจน: `server_default` **ผูกกับ `libc.so.6` ของระบบ** ส่วน `server_static` (ที่ build ด้วย `CGO_ENABLED=0`) **ไม่ผูกกับ dynamic library ใดเลย** — เป็น binary ที่สมบูรณ์ในตัวเอง (self-contained) 100%

> **กฎที่ต้องจำ**: ถ้า final stage ของ Dockerfile คือ `scratch` (หรือ base image ที่ไม่มี libc เช่น `distroless/static`) **ต้อง `CGO_ENABLED=0` เสมอ** ไม่งั้น container จะรันไม่ขึ้นเลย พร้อม error ประมาณ `exec /server: no such file or directory` (ข้อความ error นี้ทำให้สับสนได้ง่ายมาก เพราะไฟล์มีอยู่จริง แต่ตัว dynamic linker ที่ binary ต้องการกลับไม่มีอยู่ใน image ต่างหาก) ถ้าโปรเจกต์จำเป็นต้องใช้ cgo จริงๆ (เช่น driver บางตัวของฐานข้อมูลที่พึ่ง C library) ต้องเปลี่ยนไปใช้ base image ที่มี libc เช่น `distroless/base` หรือ `alpine` แทน `scratch`

`GOOS=linux` ที่เห็นคู่กันในคำสั่ง build ก็สำคัญไม่แพ้กัน: เวลา build image บนเครื่อง macOS หรือ Windows (ที่ GOOS ปัจจุบันไม่ใช่ `linux`) ต้องระบุ `GOOS=linux` ชัดเจนเสมอ เพราะ container ทุกตัวรันบน Linux kernel ไม่ว่าเครื่อง host จะเป็น OS อะไรก็ตาม — นี่คือการใช้ **cross-compilation** ที่ **Part 001** กล่าวถึงไว้ว่าเป็นจุดเด่นของ Go (คอมไพล์จาก Mac ให้รันบน Linux ได้โดยไม่ต้องมีเครื่อง Linux เลย)

---

## 7. `scratch` vs `distroless`: final stage เล็กที่สุดสองแบบ

`scratch` คือ image ที่เล็กที่สุดเท่าที่เป็นไปได้ (มีขนาด 0 byte) แต่แลกมาด้วยข้อจำกัดที่ต้องรู้:

| สิ่งที่ขาดหายไปใน `scratch` | ผลกระทบ |
|---|---|
| ไม่มี shell (`sh`, `bash`) | `docker exec -it <container> sh` เข้าไปดูข้างในไม่ได้เลย |
| ไม่มี `ca-certificates` | โปรแกรมที่เรียก HTTPS ออกไปหา service ภายนอก (เช่น เรียก API อื่น, ต่อ TLS database) จะ **verify certificate ไม่ผ่าน** ทันที |
| ไม่มี `/etc/passwd` | รันเป็น user ID เท่านั้น ตั้งชื่อ user ไม่ได้ |
| ไม่มี timezone data | `time.LoadLocation("Asia/Bangkok")` จะ error เพราะหาไฟล์ timezone ไม่เจอ |

**Distroless** (จาก Google, `gcr.io/distroless/*`) คือทางเลือกที่ตรงกลาง — ยังคง "ไม่มี package manager, ไม่มี shell, ไม่มี tool อะไรที่ไม่จำเป็น" (ตามชื่อ "distro-less") แต่**เติมสิ่งที่โปรแกรมทั่วไปมักต้องใช้กลับเข้ามา**: `ca-certificates`, timezone data, และ `/etc/passwd` ที่มี user `nonroot` ให้ใช้:

```dockerfile
FROM golang:1.24-alpine AS builder
WORKDIR /src
COPY go.mod ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -ldflags="-s -w" -o /out/server .

# distroless/static-debian12 มี ca-certificates, tzdata, /etc/passwd ให้เล็กน้อย
# เหมาะกับแอปที่ต้องเรียก HTTPS ออกไปหา service อื่น (ต้องใช้ CA cert เพื่อ verify TLS)
FROM gcr.io/distroless/static-debian12
COPY --from=builder /out/server /server
EXPOSE 8080
USER nonroot:nonroot
ENTRYPOINT ["/server"]
```

**`USER nonroot:nonroot`** สำคัญมากด้าน security: ถ้าไม่ระบุ container จะรันเป็น `root` โดย default ซึ่งถ้า attacker หาทาง exploit โปรแกรมได้ ก็จะได้สิทธิ์ root ไปด้วย — การรันเป็น non-root user เป็นแนวทางปฏิบัติมาตรฐาน (security best practice) ที่ควรทำเสมอเมื่อเป็นไปได้ (เราจะกลับมาพูดเรื่อง security ของ container แบบเจาะลึกกว่านี้ใน **Part 106**)

### เมื่อไรควรเลือกอันไหน

> **ใช้ `scratch`** เมื่อโปรแกรมไม่เรียก HTTPS ออกไปข้างนอกเลย (เช่น API ภายในที่คุยกันเองผ่าน internal network แบบไม่เข้ารหัส หรือ service ที่ไม่มี dependency ภายนอกเลย) ต้องการ image เล็กที่สุดเท่าที่เป็นไปได้จริงๆ
>
> **ใช้ `distroless`** เป็นค่าเริ่มต้นที่ปลอดภัยกว่าสำหรับ production ทั่วไป โดยเฉพาะแอปที่เรียก HTTPS API ภายนอก, ต่อฐานข้อมูลผ่าน TLS, หรือจัดการเรื่องเวลาแบบมี timezone — แลกกับขนาดที่ใหญ่กว่า `scratch` เล็กน้อยเท่านั้น

---

## 8. เทียบขนาด Image จริง: Naive vs Scratch vs Distroless

มาถึงส่วนที่น่าตื่นเต้นที่สุด — build image ทั้งสามแบบจริงแล้วเทียบขนาดกัน:

```bash
docker build -f Dockerfile.naive      -t goapi:naive      .
docker build -f Dockerfile.scratch    -t goapi:scratch    .
docker build -f Dockerfile.distroless -t goapi:distroless .
docker images goapi
```

**ผลลัพธ์จริงจากการรันบนเครื่องที่เขียนบทความนี้**:

```
REPOSITORY:TAG      SIZE
goapi:naive         523MB
goapi:distroless    15.4MB
goapi:scratch       9.17MB
```

ตัวเลขพูดแทนคำอธิบายได้ชัดเจนมาก:

| Image | ขนาด | เทียบกับ naive |
|---|---|---|
| `goapi:naive` (golang เต็มรูปแบบ) | 523 MB | 1x (baseline) |
| `goapi:distroless` | 15.4 MB | **เล็กกว่า ~34 เท่า** |
| `goapi:scratch` | 9.17 MB | **เล็กกว่า ~57 เท่า** |

ทำไมขนาดนี้ถึงสำคัญในทางปฏิบัติ ไม่ใช่แค่ตัวเลขสวยๆ:

- **Deploy เร็วขึ้นมาก**: image เล็กใช้เวลา push/pull จาก registry น้อยกว่ามาก — สำคัญมากเวลา scale out (เพิ่ม instance ใหม่ต้อง pull image ก่อนเริ่มรันได้)
- **ประหยัดค่าใช้จ่าย**: registry ส่วนใหญ่คิดเงินตามพื้นที่เก็บและ bandwidth ที่ใช้ push/pull
- **พื้นที่โจมตี (attack surface) เล็กลง**: `scratch`/`distroless` ไม่มี shell, ไม่มี package manager, ไม่มี library ที่ไม่จำเป็น — ไม่มีอะไรให้ attacker ใช้เป็นเครื่องมือต่อยอดได้แม้จะเจาะเข้ามาได้ก็ตาม
- **Attack surface จาก CVE ของ base image**: `golang:1.24-alpine` มี package ของระบบจำนวนมากที่อาจมีช่องโหว่ (CVE) ถูกค้นพบได้ตลอดเวลา ยิ่ง base image มีของน้อย ยิ่งมีโอกาสโดน CVE กระทบน้อยตามไปด้วย

---

## 9. `.dockerignore`: กันไฟล์ที่ไม่ควรหลุดเข้า build context

เมื่อรัน `docker build .` Docker จะส่ง **ทุกไฟล์ในโฟลเดอร์ปัจจุบัน** (เรียกว่า "build context") ไปให้ Docker daemon ก่อนเริ่ม build จริง ถ้าไม่ระวัง ไฟล์ที่ไม่ควรอยู่ใน image เลย (เช่น `.git/`, ไฟล์ `.env` ที่มี secret, ไฟล์ binary เก่าที่ build ค้างไว้) อาจถูกส่งไปโดยไม่ตั้งใจ ทำให้ build ช้าลงและเสี่ยงข้อมูลรั่วไหล

`.dockerignore` ทำงานเหมือน `.gitignore` ทุกประการ — ระบุ pattern ของไฟล์ที่ไม่ต้องการให้ Docker เห็นเลยตั้งแต่ต้น:

```
.git
*.md
Dockerfile*
docker-compose*.yml
.env
.dockerignore
bin/
tmp/
*.log
```

> **ข้อควรระวังเรื่อง secret**: `.dockerignore` ป้องกันไม่ให้ไฟล์เข้าไปอยู่ใน **build context** แต่ถ้า secret หลุดเข้าไปอยู่ใน **layer** ของ image ไปแล้ว (เช่น `COPY .env .` ก่อนจะมาลบทีหลังด้วย `RUN rm .env`) **secret นั้นยังคงอยู่ใน layer เก่าที่ image เก็บไว้** ตรวจสอบด้วย `docker history` ได้เสมอ — วิธีที่ถูกต้องคือ**ไม่ก็อปปี้ secret เข้า image เลยตั้งแต่แรก** ใช้ environment variable หรือ secret management ของ orchestrator แทน (เรื่องนี้จะกลับมาเจาะลึกอีกครั้งใน **Part 097** เรื่อง Kubernetes Secret และ **Part 106** เรื่อง Security)

---

## 10. `HEALTHCHECK` เมื่อไม่มี Shell ให้ใช้

คำสั่ง `HEALTHCHECK` ใน Dockerfile บอก Docker ว่าจะเช็คว่า container "มีชีวิตอยู่จริงและพร้อมให้บริการ" ได้อย่างไร — Docker จะรันคำสั่งนี้เป็นระยะ แล้วอัปเดตสถานะ container เป็น `healthy`/`unhealthy` ให้ดูผ่าน `docker ps` ได้ทันที (แนวคิดเดียวกับ readiness/liveness probe ของ Kubernetes ที่จะเรียนเจาะลึกใน **Part 097**)

ปัญหาคือรูปแบบที่คนส่วนใหญ่คุ้นเคย เช่น `CMD curl -f http://localhost:8080/healthz || exit 1` **ใช้ไม่ได้เลยกับ `scratch`/`distroless`** เพราะไม่มีทั้ง `curl` และไม่มี shell ให้ตีความ `||` ด้วยซ้ำ

ทางแก้ที่สะอาดที่สุดคือ **ให้ binary ของเราเองมีโหมดพิเศษสำหรับ health check** ตามที่เขียนไว้ในหัวข้อ 3 (`runHealthcheck`) แล้วเรียกมันแบบ **exec form** (array syntax) ที่ไม่ต้องพึ่ง shell เลย:

```dockerfile
HEALTHCHECK --interval=10s --timeout=3s --start-period=5s --retries=3 \
  CMD ["/server", "-healthcheck"]
```

- **`--interval=10s`**: เช็คทุก 10 วินาที
- **`--timeout=3s`**: ถ้าคำสั่งเช็คใช้เวลาเกิน 3 วินาทีถือว่าล้มเหลวรอบนั้น
- **`--start-period=5s`**: ให้เวลา 5 วินาทีแรกหลัง container เริ่ม โดยไม่นับผลเช็คที่ล้มเหลวในช่วงนี้เป็น "unhealthy" ทันที (เผื่อเวลา warm-up)
- **`--retries=3`**: ต้องเช็คล้มเหลวติดกัน 3 ครั้งถึงจะเปลี่ยนสถานะเป็น `unhealthy`
- **`CMD ["/server", "-healthcheck"]`**: **ต้องเป็น exec form (array)** เท่านั้น ไม่ใช่ shell form (`CMD curl ...` แบบ string ธรรมดา) เพราะ shell form ต้องพึ่ง `/bin/sh` ตีความคำสั่งซึ่งไม่มีอยู่ใน `scratch`

---

## 11. Build และ Run จริงด้วย Docker บนเครื่องที่เขียนบทความนี้

รวมทุกอย่างเข้าด้วยกันเป็น Dockerfile สุดท้ายที่ใช้จริงในบทนี้:

```dockerfile
# Dockerfile
FROM golang:1.24-alpine AS builder
WORKDIR /src
COPY go.mod ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -ldflags="-s -w" -o /out/server .

FROM scratch
COPY --from=builder /out/server /server
EXPOSE 8080
HEALTHCHECK --interval=10s --timeout=3s --start-period=5s --retries=3 \
  CMD ["/server", "-healthcheck"]
ENTRYPOINT ["/server"]
```

Build และรันจริง:

```bash
docker build -t goapi:latest .
docker run -d --name goapi-demo -p 18080:8080 goapi:latest
curl -s http://localhost:18080/healthz
curl -s http://localhost:18080/tasks
docker logs goapi-demo
```

**ผลลัพธ์จริง**:

```
$ curl -s http://localhost:18080/healthz
{"status":"ok"}
$ curl -s http://localhost:18080/tasks
[{"id":1,"name":"เขียนบท Docker","done":false},{"id":2,"name":"เขียนบท Compose","done":false}]
$ docker logs goapi-demo
{"time":"2026-09-26T03:41:52.05Z","level":"INFO","msg":"server starting","port":"8080"}
```

รอให้ `HEALTHCHECK` ทำงานครบรอบแล้วเช็คสถานะ:

```bash
sleep 12
docker inspect --format='{{json .State.Health}}' goapi-demo
docker ps --filter name=goapi-demo
```

**ผลลัพธ์จริง**:

```
$ docker inspect --format='{{json .State.Health}}' goapi-demo
{"Status":"healthy","FailingStreak":0,"Log":[{"Start":"...","End":"...","ExitCode":0,"Output":""}]}

$ docker ps --filter name=goapi-demo
CONTAINER ID   IMAGE          COMMAND     CREATED          STATUS                    PORTS                     NAMES
f673e5c493df   goapi:latest   "/server"   14 seconds ago   Up 13 seconds (healthy)   0.0.0.0:18080->8080/tcp   goapi-demo
```

สังเกตคอลัมน์ `STATUS` — `docker ps` แสดง **`(healthy)`** ต่อท้ายตรงๆ โดยไม่ต้องเข้าไปดู log หรือเดาเอง นี่คือประโยชน์ของการตั้ง `HEALTHCHECK` ไว้อย่างถูกต้อง

ทดสอบ graceful shutdown ที่เขียนไว้ในโค้ด (ทบทวนจาก **Part 047**) ด้วยการส่ง `SIGTERM` ผ่าน `docker stop` (ซึ่งภายใน Docker จะส่ง `SIGTERM` ให้ก่อนเสมอ แล้วรอ grace period หนึ่ง ก่อนจะบังคับด้วย `SIGKILL`):

```bash
docker stop goapi-demo
docker logs goapi-demo
```

**ผลลัพธ์จริง** (เห็น log การปิดตัวแบบนุ่มนวลที่ตั้งใจเขียนไว้ทำงานถูกต้องแม้อยู่ใน container):

```
{"time":"...","level":"INFO","msg":"shutdown signal received","signal":"terminated"}
{"time":"...","level":"INFO","msg":"server stopped gracefully"}
bye
```

ล้างทิ้งหลังทดสอบเสร็จ:

```bash
docker stop goapi-demo && docker rm goapi-demo
```

---

## 12. แนวทางเลือก Base Image และข้อผิดพลาดที่พบบ่อย

### แนวทางเลือก build stage image

| Base image ของ build stage | เหมาะกับ |
|---|---|
| `golang:1.24-alpine` | ตัวเลือกยอดนิยมที่สุด — Alpine Linux เล็กและเร็ว ใช้ build ได้ปกติ (ไม่กระทบ final image เพราะไม่ได้ถูก ship ไปด้วย) |
| `golang:1.24` (Debian-based) | ใหญ่กว่า alpine แต่บาง C library เข้ากันได้ดีกว่า (Alpine ใช้ `musl libc` แทน `glibc` ซึ่งบาง dependency ที่ผูก cgo อาจมีปัญหา) |

### ข้อผิดพลาดที่พบบ่อยที่สุด

1. **ลืม `CGO_ENABLED=0` แล้วใช้ `scratch`** → container รันไม่ขึ้น error `exec /server: no such file or directory` (สับสนเพราะดูเหมือนไฟล์หาไม่เจอ ทั้งที่จริงคือ dynamic linker หาไม่เจอต่างหาก)
2. **ลืม `GOOS=linux` เวลา build บน macOS/Windows** → ได้ binary ของ macOS/Windows ใส่เข้า Linux container ไม่ได้เลย
3. **`COPY . .` ก่อน `go mod download`** → ทุกครั้งที่โค้ดเปลี่ยนแม้แต่บรรทัดเดียว (แต่ dependency ไม่เปลี่ยนเลย) Docker จะ invalidate cache ของ `go mod download` ทำให้ build ช้าลงมากโดยไม่จำเป็น ควร **`COPY go.mod go.sum` แล้ว `RUN go mod download` ก่อน `COPY . .` เสมอ** เพื่อให้ cache ของ dependency ใช้ซ้ำได้ตราบใดที่ `go.mod`/`go.sum` ไม่เปลี่ยน (ตัวอย่างในบทนี้ที่ยังไม่มี dependency ภายนอกจึงไม่เห็นผลชัด แต่จะสำคัญมากใน **Part 096** ที่เพิ่ม `pgx`/`go-redis` เข้ามา)
4. **ใช้ `latest` tag ของ base image โดยไม่ pin เวอร์ชัน** → build วันนี้กับ build เดือนหน้าอาจได้ผลลัพธ์ต่างกันโดยไม่รู้ตัว ควรระบุเวอร์ชันชัดเจนเสมอ (`golang:1.24-alpine` ไม่ใช่ `golang:alpine`)

---

## 13. หมายเหตุความซื่อสัตย์เรื่องการรันจริงในบทนี้

ทุกคำสั่ง `docker build`, `docker run`, `docker inspect`, `docker stop` และผลลัพธ์ทั้งหมดที่แสดงในบทนี้ (รวมถึงตัวเลขขนาด image ในหัวข้อ 8) **รันจริงบน Docker Engine 29.3.1 ที่ติดตั้งอยู่ในสภาพแวดล้อมที่ใช้เขียนหลักสูตรนี้** — ไม่ใช่ตัวเลขที่คาดเดาหรือแต่งขึ้น เครื่องนี้มี Docker daemon (`dockerd`) รันอยู่จริงและเข้าถึง Docker Hub/`gcr.io` เพื่อ pull base image ได้ตามปกติ

---

## สรุปสิ่งที่ได้เรียนในบทนี้

- Go เหมาะกับ Docker เป็นพิเศษเพราะ compile เป็น native binary เดียวจบ ไม่ต้องมี runtime แถมไปด้วย (ตรงข้ามกับ Python/Node.js/Java)
- **Multi-stage build** แยก "build stage" (มี Go toolchain เต็ม) ออกจาก "final stage" (มีแค่ binary) ด้วย `COPY --from=<stage>`
- **`CGO_ENABLED=0`** บังคับให้ได้ **static binary** ที่ไม่ผูกกับ `libc` ของระบบ — จำเป็นเสมอเมื่อ final stage คือ `scratch`/`distroless` พิสูจน์ได้ด้วย `ldd`
- **`scratch`** คือ image เปล่าที่สุด (0 byte) เหมาะกับแอปที่ไม่เรียก HTTPS ออกไปข้างนอก ส่วน **`distroless`** เติม `ca-certificates`/`tzdata`/`nonroot` user กลับมาเล็กน้อย เหมาะเป็นค่าเริ่มต้นที่ปลอดภัยกว่าสำหรับ production ทั่วไป
- ผลการทดลองจริง: `goapi:naive` (523MB) เทียบกับ `goapi:scratch` (9.17MB) และ `goapi:distroless` (15.4MB) — **เล็กกว่ากันได้ถึง 34-57 เท่า**
- `.dockerignore` กันไฟล์ที่ไม่จำเป็น (และอาจมี secret) ไม่ให้เข้า build context
- `HEALTHCHECK` กับ image ที่ไม่มี shell ต้องให้ binary ตัวเองมีโหมด self-check แล้วเรียกด้วย exec form (`CMD ["/server", "-healthcheck"]`)
- ลำดับ `COPY go.mod/go.sum` → `go mod download` → `COPY . .` สำคัญมากต่อความเร็วในการ build ซ้ำ เพราะ Docker cache แต่ละ layer แยกกัน

## แบบฝึกหัดท้ายบท

1. Build image ทั้งสามแบบ (`Dockerfile.naive`, `Dockerfile.scratch`, `Dockerfile.distroless`) ด้วยตัวเองบนเครื่องของคุณ แล้วเทียบขนาดด้วย `docker images` — ได้ตัวเลขใกล้เคียงกับในบทความหรือไม่ ถ้าต่าง ลองอธิบายว่าทำไม (เวอร์ชัน Go, สถาปัตยกรรม CPU ที่ต่างกันมีผลได้)
2. ลบบรรทัด `CGO_ENABLED=0` ออกจาก Dockerfile ที่ใช้ `scratch` เป็น final stage แล้วลอง build และ run ดู สังเกต error message ที่ได้ แล้วอธิบายว่าทำไมถึงเกิด error นี้ (เชื่อมโยงกับผลลัพธ์ของ `ldd` ในหัวข้อ 6)
3. เพิ่ม endpoint `/panic` ในโค้ดตัวอย่างที่จงใจ `panic(...)` แล้ว build/run เป็น container ทดสอบว่า `HEALTHCHECK` ตรวจจับได้หรือไม่ว่า container "ตายแล้ว" (สังเกตว่าถ้าโปรแกรม crash ทั้ง process container ก็จะหยุดทำงานไปเลย ต่างจากกรณี handler เดียว panic แต่ไม่ recover เทียบกับที่เรียนใน **Part 017**)
4. แก้ `Dockerfile.distroless` ให้ไม่ตั้ง `USER nonroot:nonroot` แล้วใช้ `docker inspect` ดูว่า container รันด้วย user อะไรโดย default เทียบกับตอนตั้ง `USER nonroot:nonroot` แล้ว
5. ลองสลับลำดับใน Dockerfile ให้ `COPY . .` มาก่อน `RUN go mod download` แล้วแก้โค้ดใน `main.go` เพียงเล็กน้อย (เช่น เปลี่ยนข้อความ) แล้ว build ใหม่สองรอบ สังเกตความแตกต่างของเวลา build ระหว่างลำดับนี้กับลำดับที่ถูกต้องในบทความ (ยิ่งเห็นชัดเมื่อโปรเจกต์มี dependency ภายนอกจำนวนมาก)
6. ค้นคว้าเพิ่มเติมเรื่อง multi-platform build ด้วย `docker buildx build --platform linux/amd64,linux/arm64` — ทำไมการ build ให้รองรับหลายสถาปัตยกรรม CPU พร้อมกันถึงสำคัญเมื่อต้อง deploy ทั้งเครื่อง Intel/AMD และเครื่อง ARM (เช่น AWS Graviton หรือ Apple Silicon)

---

**ต่อไป**: [Part 096 — Docker Compose สำหรับ Multi-Service](./096-docker-compose.md)
